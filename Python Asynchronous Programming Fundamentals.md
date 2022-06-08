# Python异步编程基础
[原文链接](https://jwodder.github.io/kbits/posts/pyasync-fundam/#alternative-async-implementations)

Python在2014年的3.4版本中引入异步编程能力，随后再每个小版本中都有更新。然而对许多Python开发者而言，这个领域依然非常神秘和陌生。这篇文章会介绍异步编程的基本概念，让你踏出步入神秘的第一步。

这篇文章不会有太多示例代码，但是会比[官方文档](https://docs.python.org/3/library/asyncio.html)更容易理解。

[toc]

## 顶层实现概览
异步编程多函数同时交错执行（[协程](https://jwodder.github.io/kbits/posts/pyasync-fundam/#coroutines)）。在任何一个时点，只有一个函数正在执行，其他函数等待例如阻塞I/O的执行（或者如果I/O已经执行完毕，它们等待当前协程被挂起以得到重新运行的权力）。


注意：在Python标准库和我所提到的第三方库中，这些顶层实现用来挂起I/O相关的操作，如果你想添加CPU密集型代码，你更需要的是多进程而不是协程。

具体来说，当当前正在运行的协程执行到`await`语句时，该协程可能会被挂起，而另一个先前挂起的协程可能会继续执行，如果挂起的协程I/O结束返回了一个值。挂起也可能发生在`async for`语句向异步迭代器请求下一个值时，或者当进入或退出`async with`语句块时，因为这些操作在底层使用`await`。

请注意，虽然可以一次处理多个协程，有效地以任何顺序完成，但单个协程中的操作将继续按照它们的写入顺序执行。例如，给定以下代码：
```python
async def f():
    ...

async def g():
    ...

async def amain():
    await f()
    await g()

asyncio.run(amain())
```

当到达`await f()`语句时，`amain()`会被挂起直到`f()`结束，只有这行结束之后，才会继续执行到下一行，开始执行协程`g()`。你如果相同时执行`f()`和`g()`，你需要使用[`asyncio.create_task()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)，[`asyncio.gather()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather)或类似方法。查看[这篇文章](https://hynek.me/articles/waiting-in-asyncio/)获取更多细节。

通常，协程只能被其他协程调用或调度。要从同步代码中运行“顶层”协程(即在非协程函数内部或在模块级别)，最简单且首选的方式是使用Python 3.7中引入的`asyncio.run()`函数。更老的版本使用底层的[`loop.run_until_complete()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.loop.run_until_complete)。

## 定义

*异步编程* 的特性是同时执行多个任务，其中一个任务正在运行，同时等待其他任务完成。Python中的异步编程是 *协作多任务处理* 的一个例子[wiki](https://en.wikipedia.org/wiki/Cooperative_multitasking)，因为它要求正在运行的协程相互协作，各自交出控制权；如果当前的协程很长一段时间没有调用`await`，那么执行将一直保持在该协程上，而线程中的所有其他协程将保持挂起状态。这与多线程和多进程程序的抢占式多任务处理形成了对比[wiki](https://en.wikipedia.org/wiki/Preemption_(computing))，Python解释器或操作系统调度器自己决定何时在运行的上下文之间切换。通常情况下，切换发生的时间点是很难预测的。

一个 *协程函数* 通过`async def`进行定义。(在异步编程的上下文中，只用`def`定义的函数被称为 *同步函数* 。)只有异步函数可以包含`await`，`async for`，`async with`语句，从Python3.10开始不能使用`yield from`。

* 注意，仅仅调用一个协程函数并不会让它开始运行；你需要使用`asyncio.create_task()`或类似的方法安排它的并发执行，或者直接`await`它。

*协程对象* 是调用协程函数的结果。函数不会`return`或`raise`任何东西。相反，它是一个未决的计算，可以在协程函数使用`await`、`async for`或`async with`的任何点挂起或恢复。实际的返回值或异常是通过从另一个协程函数中等待协程对象(可能包装在任务和/或类似`asyncio.gather()`的东西中)或通过在同步代码中使用`asyncio.run()`将其作为“顶层”入口点运行来获得的。
* 异步生成器——使用`yield`的协程函数是一个例外。你不`await`函数的结果，而是使用`async for ... in ...`或者`await anext(...)`。
* 令人困惑的是，协程函数和协程对象都可以被称为“协程”。

协程执行的实际调度是由 *事件循环* 管理的。事件循环通过`asyncio.run()`或`asyncio.new_event_loop()`创建，处理一个或多个协程或同步回调，然后要么永远运行，要么直到完成一个“顶层”协程。事件循环的任务是执行当前的协程，直到它`await`挂起，之后它会查看是否任何挂起的协程已经准备好恢复，并选择一个恢复，如果没有准备好，则等待直到有一个准备好。

*可等待对象* 是一个可以应用`await`关键字的任何值；协程对象、future、future-like或任务（往下看）。等待一个可等待对象会导致当前协程被挂起，直到该可等待对象准备好提供返回值或引发异常。

future(`asyncio.Future`类)是一个计算结果(值或异常)的底层容器，它开始时为空，随后被赋值或抛出异常。等待一个future将暂停当前的协程，直到其他程序在未来存储结果或取消它。

* 您可能已经熟悉了[`concurrent.futures`](https://docs.python.org/3/library/concurrent.futures.html)库中`Future`类形式的futures，它提供对在其他线程或进程中计算的操作结果的访问。`asyncio.Future`在设计上类似，但有一套自己的API。

future-like是一个带有`__await__()`方法的对象，它必须返回一个迭代器。`await`一个future-like会导致当前协程被挂起，直到迭代器耗尽，此时`StopIteration`异常会作为`await`表达式的结果返回。

任务(`asyncio.Task`类)表示正在运行的协程。使用`asyncio.create_task()`从协程对象创建任务将导致协程被调度，与其他正在运行的协程并发执行。稍后可以`await`任务实例来暂停当前的协程，直到包装的协程执行完毕并返回结果。

正在执行的任务可以通过调用`Task.cancel()`方法取消。这将导致底层协程在下一次`await`时收到一个`asyncio。CancelledError`异常，结束任务的执行。

## 语法

Python中的异步编程发生在协程的内部，使用`async def`定义函数。在协程中，`await`关键字可以应用于任何可等待的表达式(例如对另一个协程的调用)，以暂停协程的执行，直到可等待的值或异常准备就绪，此时协程恢复，`await`表达式返回该值或引发该异常。

以下是你在教程中看到的一个基本例子：
```python
import asyncio

async def waiter():
    print("Before sleep")
    await asyncio.sleep(5)
    print("After sleep")

asyncio.run(waiter())
```
上述代码先打印一条消息，sleep五秒然后打印另一条消息。正如所写的那样，这毫无意义；我们一次只运行一个协程，因此与使用带有`time.sleep()`的同步函数相比没有任何优势。下面是一些稍微复杂一点的事情:
```python
import asyncio

async def operate(time, result):
    print(f"Spending {time} seconds doing operations ...")
    await asyncio.sleep(time)
    print(f"Operations done after {time} seconds!")
    return result

async def amain():
    x, y = await asyncio.gather(operate(5, 42), operate(2, 23))
    print(f"Got {x=}, {y=}")
    assert x == 42
    assert y == 23

asyncio.run(amain())
```
这段代码模拟了并行地执行两个阻塞操作。如果你运行脚本(需要Python 3.8+)并计时，你将看到它总共只花费了大约5秒，并且2秒的任务比5秒的任务提前3秒完成。完成这两个任务后，程序将“Got x=42, y=23”。

除了`await`，还有两种特定用于协程的语法结构：`async for... in ...:`(用于对异步可迭代对象进行迭代)和`async with...:`(用于进入和退出异步上下文管理器)。这些对象的工作方式与非异步对象相同，只是相关的可迭代对象和上下文管理器需要支持异步使用；例如，`async for`不能遍历列表，而`async with`不能操作普通文件句柄。类似地，通常的`for`不能应用于异步迭代器，通常的`with`也不能应用于`asyncio.Lock`。

谈到异步迭代，它的工作原理与你期待的差不多：通过在协程中使用`yield`，它变成了可以使用`async for`和`await anext(...)`的异步迭代器。请注意，与非生成器协程相比，你不需要对异步生成器应用`await`。举个例子，给出下面的函数：
```python
async def aiterator():
    for i in range(5):
        await asyncio.sleep(i)
        yield i
```
你这么使用它：
```python
async for x in aiterator():
    print(x)
```
不需要在`async for`中写`await`。

请注意，没有办法在不`await`的情况下从协程中获取值(直接或通过类似`asyncio.gather()`的方法)；如果一个协程永远不会被等待，也永远不会被转换为任务，那么`asyncio`将报错未等待的任务被垃圾收集。还有，`await`（`async for`和`async with`）不能在协程外使用；如果想启动一个“最外层”协程，你需要使用`asynico.run()`。

## 类型注释

为异步代码添加类型注释的方式和同步代码相同。如果一个异步`func()`接受一个整数`x`并返回一个字符串，你可以这么写注释`async def func(x: int) -> str`。然而，如果传递一个未等待的协程对象(这并不总是好主意)，你可以将其注释为`Awaitable[T]`，其中`T`是协程的返回类型。

异步可调用对象没有自己的类型；它们被代替注释为`Callable[..., Awaitable[T]]`，其中`T`是协程函数的返回类型。

异步迭代器，可迭代对象，上下文管理器都有自己的类型：`AsyncIterator`，`AsyncIterable`，`typing.AsyncContextManager`/`contextlib.AbstractAsyncContextManager`。

## 魔术方法

### `__aiter__()`和`__anext__()`
这些方法用于实现异步迭代器和可迭代类，作为编写异步生成器函数的替代方案；类似于用`__iter__()`和`__next__()`方法定义类实现同步迭代器，作为编写生成器函数的替代方案。

### `__aenter__()`和`__aexit__()`
这些方法用来实现异步上下文管理器。类似于使用`__enter__()`和`__exit__()`定义同步的上下文管理器，除了异步的版本必须是协程。

### `__await__()`

该方法用来创建一个可以直接被`await`的future-like类。通常很少需要实现它，但是为了文档完整起见，我们在这里包含它。

## 历史注释：基于生成器的协程

当`asyncio`在Python 3.4中首次引入时，`async`和`await`关键字还没有出现。相反，协程函数是通过应用`@asyncio.coroutine`协程装饰器到普通生成器函数，并使用`yield from`完成等待。在3.4中也没有异步迭代器或异步上下文管理器。即使在Python 3.5中引入`async`和`await`之后，旧的基于生成器的协程也不能使用它们。

这种编写协程的风格在Python 3.8中已弃用，并在Python 3.11中完全删除。

## 同时运行多个协程

现在到了你期待已久的部分，也是让异步编程值得一试的部分：实际同时运行多个函数。

最简单的方式是将协程对象传递到`asyncio.create_task()`，这步会安排协程的执行但不会真的执行直到当前协程调用`await`。`asyncio.create_task()`返回一个`asyncio.Task`对象，可以用来查询协程状态或取消它。对任务进行`await`会挂起当前协程直到该任务完成，此时将返回任务底层协程的返回值。

如果你创建了多个任务然后一个接一个`await`它们，给定的`await`在相关任务完成之前不会返回结果；如果任务B完成运行，而你还在等待任务A，那么正在等待的协程将继续被挂起，直到A完成，当它稍后等待B时，它将立即返回B的返回值，因为B已经完成了。举个例子：
```python
import asyncio
from time import strftime

def hms():
    return strftime("%H:%M:%S")

async def operate(time, result):
    print(f"{hms()}: Spending {time} seconds doing operations ...")
    await asyncio.sleep(time)
    print(f"{hms()}: Operations done after {time} seconds!")
    return result

async def amain():
    task1 = asyncio.create_task(operate(5, 42))
    task2 = asyncio.create_task(operate(2, 23))
    r1 = await task1
    print(f"{hms()}: task1 returned {r1}")
    r2 = await task2
    print(f"{hms()}: task2 returned {r2}")

asyncio.run(amain())
```
输出如下：
```shell
17:12:56: Spending 5 seconds doing operations ...
17:12:56: Spending 2 seconds doing operations ...
17:12:58: Operations done after 2 seconds!
17:13:01: Operations done after 5 seconds!
17:13:01: task1 returned 42
17:13:01: task2 returned 23
```
如果你需要等待多个协程但不关心完成顺序，那你可以使用`asyncio.gather()`、`asyncio.as_completed()`或`asyncio.wait()`；[这篇文章](https://hynek.me/articles/waiting-in-asyncio/)解释了这些函数的区别。

如果你不需要一个任务的返回值或者不需要确保它会完成，你就不必等待它，但如果这样的“后台任务”引发了未捕获的异常，`asynico`会报错。解决这个问题的一种方法是使用`task .add_done_callback()`向任务附加一个同步回调函数，该函数使用`task .exception()`检索任何未捕获的异常，就像这样：
```python
import asyncio
from time import strftime

def hms():
    return strftime("%H:%M:%S")

async def bg_task():
    print(f"{hms()}: In the background")
    raise RuntimeError("Ouch")

def done_callback(task):
    try:
        if e := task.exception():
            print(f"{hms()}: Task <{task.get_name()}> raised an error: {e}")
        else:
            print(f"{hms()}: Task <{task.get_name()}> finished successfully")
    except asyncio.CancelledError:
        print(f"{hms()}: Task <{task.get_name()}> was cancelled!")

async def fg_task():
    task = asyncio.create_task(bg_task(), name="bg_task")
    task.add_done_callback(done_callback)
    print(f"{hms()}: Now we sleep and let bg_task do its thing")
    await asyncio.sleep(2)
    print(f"{hms()}: I'm awake!")

asyncio.run(fg_task())
```
上述代码输出：
```shell
17:16:33: Now we sleep and let bg_task do its thing
17:16:33: In the background
17:16:33: Task <bg_task> raised an error: Ouch
17:16:35: I'm awake!
```

## 异常处理

无论何时在协程中发生异常，它都会向上传播到等待它的对象；如果未处理，它将通过`asyncio.run()`调用一路传播出去，这时所有正在运行的任务会全部取消。如果没有指向“顶层”协程的`await`链(例如你执行了`asyncio.create_task()`，然后没有等待结果，让它在后台运行)，当协程最终被垃圾回收时，`asyncio`将报错。看上面的例子，使用`Task.add_done_callback()`进行异常处理

如果触发了`KeyboardInterrupt`，不管主线程正在运行什么协程都会引发异常。

## 示例代码

[gist](https://gist.github.com/jwodder/c0ad1a5a0b6fda18c15dbdb405e1e549)提供了异步编程来同步下载GitHub项目的优秀示例。让我们试试这样：
```shell
python download-assets.py --download-dir jq stedolan/jq jq-1.5 jq-1.6
```
脚本需要Python3.8+和ghrepo和httpx库。

## 异步编程VS线程

异步编程没有使用线程，默认情况所有协程都运行在被称为`asyncio.run()`的线程中。当使用`asyncio.to_thread()`或`loop.run_in_executor()`在单独的线程中运行同步函数时(或者使用后一个函数，甚至是单独的进程)会出现例外情况，返回一个可等待的对象以接收函数的结果。

如果多个线程分别调用`asyncio.run()`，每个线程会持有它自己的事件循环和协程收集。

注意，Python进程中的每个线程每次最多有一个事件循环，而且一个事件循环只能属于一个线程。一个重要的事实是，如果你有一个同步函数`foo()`，它在某个协程中调用了`asyncio.run()`，那么`foo()`不能再被其他协程调用，因为这会导致一个线程中存在两个事件循环，这不会正常工作。

与线程相比，异步编程具有以下优点：
* 在异步编程中，只有当当前协程使用`await`或类似的方法时，正在执行的协程才能改变。这允许程序员确信在同一个协程中，操作不会被其他协程干扰，数据也不会被其他协程修改。

* 另一方面，当使用线程时，正在运行的线程几乎可以在解释器选择的任何时间点进行更改，这就需要仔细编程并大量使用锁，以确保变量不会被另一个使用它们的线程背地里修改。

* 如果你对线程做过认真的工作，你可能会遇到这样的事实：你不能在线程执行的过程中“杀死”线程，除非“可杀死”线程被故意编程为允许这样做，例如，定期检查一些标志如果为ture就退出，。另一方面，异步编程可以通过`asyncio.Task.cancel()`方法取消正在运行的协程；一旦一个协程被取消，当它在`await`或类似的情况下挂起时，下一次事件循环检查它时，协程将被恢复，但不是接收它所等待的值，而是一个`asyncio.CancelledError`将在`await`表达式中引发，可能会结束协程的执行。

回想一下，由于Python的全局解释器锁(GIL)，无论Python程序使用多少线程或机器有多少核，在任何时候都只有一个线程在执行Python字节码。

## 异步版本历史

以下是需要注意的各Python版本中异步变更历史

### Python3.4

实施[PEP 3156](http://www.python.org/dev/peps/pep-3156)，`asyncio`模块被添加进标准库。允许通过`@asyncio.coroutine`装饰器创建协程；协程内部使用`yield from`表现await语句。（通过这种方式创建协程在Python3.8中被废弃，在Python3.11中移除。）

大多数功能在这个版本中被添加进来，整合为`asyncio`的“底层”实现。

### Python3.5

实施[PEP 492](http://www.python.org/dev/peps/pep-0492)

* 可以通过关键字`async def`和`await`定义协程。在Python3.5中不可以在`async def`的协程中使用`yield`。

* 通过`async for`实现异步迭代

* 通过`async with`实现异步上下文管理器

* `__await__()`，`__aiter__()`，`__anext__()`，`__aenter__()`，`__aexit__()`魔术方法被引入。

* 最初，`__aiter__()`方法被期望为解析为异步迭代器的协程(或任何返回可等待对象的方法)。这在3.5.2中被更改为`__aiter__()`，而不是直接返回异步迭代器。3.5.2开始从`__aiter__()`返回一个可等待对象会产生`PendingDeprecationWarning`，从3.6开始产生`DeprecationWarning`，3.7开始产生`RuntimeError`。

### Python3.6

* 可以在`async def`协程函数中使用`yield`，从而启用异步生成器（[PEP 525](http://www.python.org/dev/peps/pep-0525)）。(但`yield from`还是禁止的)

* `async def`可以被用于列表，集合，字典推导和生成器表达式中。

* `await`表达式可被应用于任何推导中。

* 使用`async`或`await`作为标识符会生成`DeprecationWarning`

### Python3.7

* `async`和`await`成为保留字

* 添加`asyncio.run()`

* 添加`asyncio.create_task()`

### Python3.8

* 运行`python -m asyncio`会启动一个异步交互式解释器

* `@asyncio.coroutine()`被废弃

* 对于`asyncio`的大部分高级API，传递`loop`参数被废弃

* `asyncio.CancelledError`现在直接从`BaseException`继承，而不是`Exception`

### Python3.9

* 添加`asyncio.to_thread()`

### Python3.10

* 添加`aiter()`和`anext()`

* `loop`参数（在Python3.8中废弃），现在被从大部分`asyncio`高级API中移除

### Python3.11

* `@asyncio.coroutine`（废弃于Python3.8）被移除

## 异步的替代实现