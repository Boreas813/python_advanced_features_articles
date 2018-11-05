# 使用Reids stream实现多进程任务队列
[原文连接](http://charlesleifer.com/blog/multi-process-task-queue-using-redis-streams/)

在这片文章中我会通过一些代码来展示如何使用[Redis stream](https://redis.io/topics/streams-intro)实现python的多进程任务队列。任务队列经常使用在web应用中，这使得在请求/响应的过程中分离耗时的操作实现异步。举个例子，当某人提交了“联系我”的表单后，web应用将消息放进一个消息队列，因此检查垃圾邮件和发送电子邮件的相对耗时的过程发生在web请求之外的一个单独的工作进程中。

脚本大概有100行代码，提供了一个相似的API：
```
queue = TaskQueue('my-queue')

@queue.task
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, a + b
    return b

# Calculate 100,000th fibonacci number in worker process.
fib100k = fib(100000)

# Block until the result becomes ready, then display last 6 digits.
print('100,000th fibonacci ends with: %s' % str(fib100k())[-6:])
```
当我使用Redis作为消息中间件时总是遵从[LPUSH/BRPOP](https://github.com/coleifer/huey/blob/000be5ca574ddae449571580ac6f98e9f72bbe85/huey/storage.py#L311-L325)(left-push, blocking right-pop)来读写数据。当向list写入时确保消息不会丢失。阻塞的右端pop是自动操作，Redis不在乎你有多少个客户端监听，每条消息都会被分发到一个消费者。

使用list有很多缺点，主要是因为阻塞右端pop是一个消极读取。一旦消息被读取，应用不再知道消息是否被成功处理还是处理失败需要重试。同样的，哪些消费者处理了哪些消息也没有可视化。

Redis5.0引入了一种新的数据类型stream，特性为append-only，持久消息日志。通过一个key定义，支持append，read，delete操作。stream提供了比其他数据类型更有益的特性来构建分布式任务队列，特别是使用[消费者组](https://redis.io/topics/streams-intro#consumer-groups)的时候。

- stream支持消息扇出给全部感兴趣的消费者，或者使用消费者组让消息均匀的分散到各个消费者。
- 消息被持久储存并且保留历史记录，即使消息已经被消费者读取。
- Redis跟踪消息分发的状态来确保消息被成功处理或者失败需要处理（ACK确认机制）。
- 消息的结构是任意数量的键值对，比存储在列表中的不透明blob提供了更多的内部结构。

消费者组提供统一的接口来管理消息分发和查询队列状态。这使得使用Redis消息队列是您的不二之选。

## 构建一个简单任务队列
我们展示一个python多进程任务队列来说明使用Redis stream构建任务队列非常简单。我们的任务队列支持使用@task装饰器的一组worker执行任意python函数。

代码围绕三个简单类展开：

- TaskQueue - 负责向stream写入消息以及协调一组工作进程。同样提供任务结果的读写接口。
- TaskWorker - 负责从stream中读消息然后执行任务。
- TaskResultWrapper - 允许使用装饰器的调用者接收任务结果

仔细读代码，处理Redis调用的都是python顶层方法：
- add()方法用来当任务被调用时向stream写入消息。
- read()方法用消费组从stream中读取一条消息。
- ack()方法用来将任务标记为执行成功。

代码如下：
```
from collections import namedtuple
from functools import wraps
import datetime
import multiprocessing
import pickle
import time

# At the time of writing, the standard redis-py client does not implement
# stream/consumer-group commands. We'll use "walrus", which extends the client
# from redis-py to provide stream support and high-level, Pythonic containers.
# More info: https://github.com/coleifer/walrus
from walrus import Walrus


# Lightweight wrapper for storing exceptions that occurred executing a task.
TaskError = namedtuple('TaskError', ('error',))


class TaskQueue(object):
    def __init__(self, client, stream_key='tasks'):
        self.client = client  # our Redis client.
        self.stream_key = stream_key

        # We'll also create a consumer group (whose name is derived from the
        # stream key). Consumer groups are needed to provide message delivery
        # tracking and to ensure that our messages are distributed among the
        # worker processes.
        self.name = stream_key + '-cg'
        self.consumer_group = self.client.consumer_group(self.name, stream_key)
        self.result_key = stream_key + '.results'  # Store results in a Hash.

        # Obtain a reference to the stream within the context of the
        # consumer group.
        self.stream = getattr(self.consumer_group, stream_key)
        self.signal = multiprocessing.Event()  # Used to signal shutdown.
        self.signal.set()  # Indicate the server is not running.

        # Create the stream and consumer group (if they do not exist).
        self.consumer_group.create()
        self._running = False
        self._tasks = {}  # Lookup table for mapping function name -> impl.

    def task(self, fn):
        self._tasks[fn.__name__] = fn  # Store function in lookup table.

        @wraps(fn)
        def inner(*args, **kwargs):
            # When the decorated function is called, a message is added to the
            # stream and a wrapper class is returned, which provides access to
            # the task result.
            message = self.serialize_message(fn, args, kwargs)

            # Our message format is very simple -- just a "task" key and a blob
            # of pickled data. You could extend this to provide additional
            # data, such as the source of the event, etc, etc.
            task_id = self.stream.add({'task': message})
            return TaskResultWrapper(self, task_id)
        return inner

    def deserialize_message(self, message):
        task_name, args, kwargs = pickle.loads(message)
        if task_name not in self._tasks:
            raise Exception('task "%s" not registered with queue.')
        return self._tasks[task_name], args, kwargs

    def serialize_message(self, task, args=None, kwargs=None):
        return pickle.dumps((task.__name__, args, kwargs))

    def store_result(self, task_id, result):
        # API for storing the return value from a task. This is called by the
        # workers after the execution of a task.
        if result is not None:
            self.client.hset(self.result_key, task_id, pickle.dumps(result))

    def get_result(self, task_id):
        # Obtain the return value of a finished task. This API is used by the
        # TaskResultWrapper class. We'll use a pipeline to ensure that reading
        # and popping the result is an atomic operation.
        pipe = self.client.pipeline()
        pipe.hexists(self.result_key, task_id)
        pipe.hget(self.result_key, task_id)
        pipe.hdel(self.result_key, task_id)
        exists, val, n = pipe.execute()
        return pickle.loads(val) if exists else None

    def run(self, nworkers=1):
        if not self.signal.is_set():
            raise Exception('workers are already running')

        # Start a pool of worker processes.
        self._pool = []
        self.signal.clear()
        for i in range(nworkers):
            worker = TaskWorker(self)
            worker_t = multiprocessing.Process(target=worker.run)
            worker_t.start()
            self._pool.append(worker_t)

    def shutdown(self):
        if self.signal.is_set():
            raise Exception('workers are not running')

        # Send the "shutdown" signal and wait for the worker processes
        # to exit.
        self.signal.set()
        for worker_t in self._pool:
            worker_t.join()


class TaskWorker(object):
    _worker_idx = 0

    def __init__(self, queue):
        self.queue = queue
        self.consumer_group = queue.consumer_group

        # Assign each worker processes a unique name.
        TaskWorker._worker_idx += 1
        worker_name = 'worker-%s' % TaskWorker._worker_idx
        self.worker_name = worker_name

    def run(self):
        while not self.queue.signal.is_set():
            # Read up to one message, blocking for up to 1sec, and identifying
            # ourselves using our "worker name".
            resp = self.consumer_group.read(1, 1000, self.worker_name)
            if resp is not None:
                # Resp is structured as:
                # {stream_key: [(message id, data), ...]}
                for stream_key, message_list in resp:
                    task_id, data = message_list[0]
                    self.execute(task_id.decode('utf-8'), data[b'task'])

    def execute(self, task_id, message):
        # Deserialize the task message, which consists of the task name, args
        # and kwargs. The task function is then looked-up by name and called
        # using the given arguments.
        task, args, kwargs = self.queue.deserialize_message(message)
        try:
            ret = task(*(args or ()), **(kwargs or {}))
        except Exception as exc:
            # On failure, we'll store a special "TaskError" as the result. This
            # will signal to the user that the task failed with an exception.
            self.queue.store_result(task_id, TaskError(str(exc)))
        else:
            # Store the result and acknowledge (ACK) the message.
            self.queue.store_result(task_id, ret)
            self.queue.stream.ack(task_id)


class TaskResultWrapper(object):
    def __init__(self, queue, task_id):
        self.queue = queue
        self.task_id = task_id
        self._result = None

    def __call__(self, block=True, timeout=None):
        if self._result is None:
            # Get the result from the result-store, optionally blocking until
            # the result becomes available.
            if not block:
                result = self.queue.get_result(self.task_id)
            else:
                start = time.time()
                while timeout is None or (start + timeout) > time.time():
                    result = self.queue.get_result(self.task_id)
                    if result is None:
                        time.sleep(0.1)
                    else:
                        break

            if result is not None:
                self._result = result

        if self._result is not None and isinstance(self._result, TaskError):
            raise Exception('task failed: %s' % self._result.error)

        return self._result
```
