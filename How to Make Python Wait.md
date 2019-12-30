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
