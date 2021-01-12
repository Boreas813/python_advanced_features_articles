# 如何让Python等待

[原文链接](https://blog.miguelgrinberg.com/post/how-to-make-python-wait)

许多情况下我们要暂停程序直到某些条件成立。可能要等待另一个线程执行完，或某个路径下出现一个新文件。

很多情况下你都需要让脚本去等待，如果你想正确的实现这个功能其实并不容易。本篇指南中我会展示几种不同的等待方法。全部例子我都用Python实现，但是实现思路可以应用于任何编程语言。

## 一个应用等待的例子

如下示例：
```python
from random import random
import threading
import time

result = None

def background_calculation():
    # 这里做一些长时间计算
    time.sleep(random() * 5 * 60)

    # 计算完成后将结果存到一个全局变量中
    global result
    result = 42

def main():
    thread = threading.Thread(target=background_calculation)
    thread.start()

    # TODO: 在这里等待result有结果之后再执行下面的代码

    print('The result is', result)

if __name__ == '__main__':
    main()
```
在这个应用中，`background_calculation()`函数进行一些很耗时的计算。为了让例子简单，我用`time.sleep()`并在里面加入一个随机数来代替。当计算结束后全局变量`result`被设置为[42](https://en.wikipedia.org/wiki/Phrases_from_The_Hitchhiker%27s_Guide_to_the_Galaxy#Answer_to_the_Ultimate_Question_of_Life,_the_Universe,_and_Everything_(42))。

应用主函数在一个单独的线程中启动后台计算然后等待线程结束打印`result`全局变量。上面版本中还没有等待后台计算完成的部分，你可以看到一个`TODO`部分，这是我们需要完成的。在下面的部分我会展示不同的等待的方法，先从最糟的开始，努力做到最好。

## 最捞的：忙等待（Busy Waiting）

凭直觉最简单的方式是把等待写入一个while循环里：
```python
    # wait here for the result to be available before continuing
    while result is None:
        pass
```

如果你想试试，这里有完整代码：
```python
from random import random
import threading
import time

result = None

def background_calculation():
    # here goes some long calculation
    time.sleep(random() * 5 * 60)

    # when the calculation is done, the result is stored in a global variable
    global result
    result = 42

def main():
    thread = threading.Thread(target=background_calculation)
    thread.start()

    # wait here for the result to be available before continuing
    while result is None:
        pass

    print('The result is', result)

if __name__ == '__main__':
    main()
```
这个实现像屎一样，你能看出来原因么？

你可以在你的系统上运行该脚本。运行的时候打开你的任务管理器查看CPU使用情况，并注意看它是如何一飞冲天的。

这个while循环看起来是一个空循环，实际上大部分时间都是空循环，唯一的作用就是一遍一遍检查确定什么时候退出循环。这样做相当于让Python不断检查`result`是否为`None`，这消耗了大量CPU周期并让运行在这颗核心上的其他事务变得非常慢。

这种类型的等待循环叫做*忙等待（busy wait）*。永远不要这样做。

## 较捞的：带有sleep的忙等待

有趣的是，在前一节的忙等待示例中你可能以为空循环应该减少CPU的工作，但事实恰恰相反。因此想要改进就在while循环中添加一些东西，让CPU判断一次就停一会。

所以我们可以这样做：
```python
    # wait here for the result to be available before continuing
    while result is None:
        time.sleep(15)
```

完整脚本：
```python
from random import random
import threading
import time

result = None

def background_calculation():
    # here goes some long calculation
    time.sleep(random() * 5 * 60)

    # when the calculation is done, the result is stored in a global variable
    global result
    result = 42

def main():
    thread = threading.Thread(target=background_calculation)
    thread.start()

    # wait here for the result to be available before continuing
    while result is None:
        time.sleep(15)

    print('The result is', result)

if __name__ == '__main__':
    main()
```

引入`time.sleep()`让程序每隔一段时间做一次判断，这样处理器在程序sleep期间可以自由的处理其他事务。

如果你尝试这个版本的等待方式，会发现CPU不会过载，因此你可能以为这就是一个完美的解决方案。然而还能做到更好。

虽然这个解决方案比前一个好很多，但是仍有两个问题使它不够理想。首先这个循环依旧是忙等待，它使用的CPU比前一个少很多，但我们仍占用一个CPU来实现等待，我们只是降低了使用频率让它变得可以接受。

第二个问题更令人担心，假设后台计算需要61秒完成。如果等待循环和它同时启动，它会在0，15，30，45，60，75秒检查`result`的值。在60秒的时候仍然返回`False`，所以程序会在75秒的时候退出循环。后台任务执行61秒，等待循环执行75秒，多用了14秒来获取结果。

这种类型的等待很常见，它存在“分析度”问题，即等待的长度是你循环中单次sleep时间的倍数。如果sleep较短那等待时间将更精确，但CPU使用率会上升，如果sleep过多，那CPU使用率会更少，花的时间更长。

## 好的 #1：在线程中使用join（Joining the Thread）

现在我们让等待尽可能高效。我们想要等待直到计算线程输出结果的那一瞬间，我们该怎么做？

为了能够有效地等待，我们需要来自操作系统的外部帮助，它可以在某些事件发生时有效地通知我们的应用程序。特别是它可以告诉我们一个线程何时退出，这个操作称为*连接一个线程（joining a thread）*。

Python标准库的`threading.Trhead`类有一个`join()`方法会在线程退出的时刻返回：
```python
    # wait here for the result to be available before continuing
    thread.join()
```
完整脚本：
```python
from random import random
import threading
import time

result = None

def background_calculation():
    # here goes some long calculation
    time.sleep(random() * 5 * 60)

    # when the calculation is done, the result is stored in a global variable
    global result
    result = 42

def main():
    thread = threading.Thread(target=background_calculation)
    thread.start()

    # wait here for the result to be available before continuing
    thread.join()

    print('The result is', result)

if __name__ == '__main__':
    main()
```

`join()`方法阻塞的机制和`time.sleep()`一样，但是它不阻塞固定的时间，而是在后台线程运行时阻塞。在线程结束的时刻`join()`函数返回，程序继续运行，操作系统层面使其变得简单高效。

## 好的 #2：等待一个事件

如果你需要等待线程完成，用上一节的模式就好。但是许多其他情况下你可能要等待线程以外的东西，那么如何等待某种普通事件而不是线程或其他操作系统资源呢？

为了演示怎么做，我把后台线程的例子改的更复杂一些。现在的线程仍然生成结果，但是生成后不会立刻退出而是进行运行做其他计算：
```python
from random import random
import threading
import time

result = None

def background_calculation():
    # here goes some long calculation
    time.sleep(random() * 5 * 60)

    # when the calculation is done, the result is stored in a global variable
    global result
    result = 42

    # do some more work before exiting the thread
    time.sleep(10)

def main():
    thread = threading.Thread(target=background_calculation)
    thread.start()

    # wait here for the result to be available before continuing
    thread.join()

    print('The result is', result)

if __name__ == '__main__':
    main()
```

如果你运行上述代码结果获取就会滞后十秒钟，因为线程在生成结果后继续运行。但是我们想要立刻得到结果，应对这种情况你需要程序满足一个特定的条件，我们可以使用`threading`模块的`Event`类来实现。下面是如何创建一个事件：
```python
result_available = threading.Event()
```
事件有一个`wait()`方法用来开始我们的等待：
```python
    # wait here for the result to be available before continuing
    result_available.wait()
```

`event.wait()`和`thread.join()`的区别在于后者是预编好的用来等待特定的事件，即线程的结束。前者是一个通用事件，可以等待任何事情。所以如果这个事件类可以等待任何情况，我们如何告知它条件满足了然后结束等待？我们用事件类的`set()`方法来做这个，在后台线程设置`result`全局变量后，它可以立即设置事件，导致任何等待它的代码解除阻塞：
```python
    # when the calculation is done, the result is stored in a global variable
    global result
    result = 42
    result_available.set()
```

完整代码如下：
```python
from random import random
import threading
import time

result = None
result_available = threading.Event()

def background_calculation():
    # here goes some long calculation
    time.sleep(random() * 5 * 60)

    # when the calculation is done, the result is stored in a global variable
    global result
    result = 42
    result_available.set()

    # do some more work before exiting the thread
    time.sleep(10)

def main():
    thread = threading.Thread(target=background_calculation)
    thread.start()

    # wait here for the result to be available before continuing
    result_available.wait()

    print('The result is', result)

if __name__ == '__main__':
    main()
```

这里你可以看到后台线程和主线程是如何围绕这个`Event`类进行 *同步（synchronized ）* 的。

## 好的 #3：显示百分比进度的等待

event类的有个优点是通用性，如果你施展一些创造力，会发现很多场景他们都很有用。举个例子，在编写后台运行线程时的这种常见模式：
```python
exit_thread = False

def background_thread():
    while not exit_thread:
        # do some work
        time.sleep(10)
```

这里我们尝试写一个线程，这个线程当全局变量`exit_thread`设置为`True`时退出。这是一个很常见的模式，但现在你可能知道为什么这不是一个很好的解决方案了对吧？从设置`exit_thread`变量到线程实际退出最长可能需要十秒，这还不包含线程到达sleep语句之前可能经过的额外时间。

我们可以把上述代码用`Event`类写成更高效的形式，向`Event.wait()`方法中加入`timeout`参数：
```python
exit_thread = threading.Event()

def background_thread():
    while True:
        # do some work
        if exit_thread.wait(timeout=10):
            break
```
这个实现中我们把固定sleep替换成事件对象。我们还是每次循环睡眠十秒，但是当线程执行到`exit_thread.wait(timeout=10)`，同时事件的`set()`方法被从别处调用，然后`wait()`调用会立刻返回`True`之后线程退出。如果wait超时会返回`False`就和`time.sleep(10)`效果一样。

如果线程在进行一些工作时`exit_thread.set()`被从别处调用，线程会继续执行工作直到代码执行到`exit_thread.wait()`然后退出。如果不想等待就杀掉线程，就确保事件实例的检查频繁一些。

现在我使用`timeout`参数做一个更复杂的例子，我会扩展上面的代码给它添加一个等待百分比。

首先向后台线程添加进度报告。在最初的版本中，我的睡眠时间是随机的，最多300秒，也就是5分钟。为了在这段时间内报告任务进度，我将用运行100次迭代的循环，每次迭代睡眠一点来取代单一的休眠，这将给我机会在每次迭代中报告进度百分比。因为大睡眠持续了300秒，现在我要做100次，每次3秒。总的来说，这个任务将花费相同数量的随机时间，但是将工作划分为100份可以很容易地报告完成百分比。

这是后台线程中不一样的地方，一个`progress`的全局变量：
```
progress = 0

def background_calculation():
    # here goes some long calculation
    global progress
    for i in range(100):
        time.sleep(random() * 3)
        progress = i + 1

    # ...
```

现在的等待每五秒报告一次百分比，更加智能了：
```
    # wait here for the result to be available before continuing
    while not result_available.wait(timeout=5):
        print('\r{}% done...'.format(progress), end='', flush=True)
    print('\r{}% done...'.format(progress))
```

新的循环每五秒用`result_available`事件当作退出条件。如果过程中无事发生就在循环中打印`progress`。我使用了`\r`和`end='', flush=True`避免`print()`函数跳到另一行。这个技巧能让你在终端同一行进行print。

完整代码：
```
from random import random
import threading
import time

progress = 0
result = None
result_available = threading.Event()

def background_calculation():
    # here goes some long calculation
    global progress
    for i in range(100):
        time.sleep(random() * 3)
        progress = i + 1

    # when the calculation is done, the result is stored in a global variable
    global result
    result = 42
    result_available.set()

    # do some more work before exiting the thread
    time.sleep(10)

def main():
    thread = threading.Thread(target=background_calculation)
    thread.start()

    # wait here for the result to be available before continuing
    while not result_available.wait(timeout=5):
        print('\r{}% done...'.format(progress), end='', flush=True)
    print('\r{}% done...'.format(progress))

    print('The result is', result)

if __name__ == '__main__':
    main()
```

## 等待的更多方式

事件对象并不是你应用中唯一的方法，有些方法比事件更合适，取决于你在等待什么。

如果你需要监视目录中的文件，并在文件被删除或修改时做一些操作那事件就不行了，因为设置事件的条件位于应用的外部。这种情况下你需要使用操作系统提供的工具来监视文件系统。在Python中你可以使用[watchdog](https://pypi.org/project/watchdog/)

如果你需要等待一个子进程结束，[subprocess](https://pypi.org/project/watchdog/)包提供了一些启动和等待进程的功能。

如果你需要从socket上读取数据，socket的默认配置时阻塞读取直到数据到达，使用[select](https://docs.python.org/3/library/select.html?highlight=select#module-select)提供的闭包将其转换为高效的事件等待。

如果你想编写生产者/消费者类型的应用可以使用[Queue](https://docs.python.org/3/library/queue.html)对象。生产者将数据添加到队列，消费者有效地等待项目从队列中取出。

正如这些，大多数情况下操作系统提供了有效的等待机制，你需要做的就是找到如何从Python中访问他们。

## 异步中的等待
pass