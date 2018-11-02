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
