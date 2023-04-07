# Python3.11：你需要尝试的新特性

[原文链接](https://realpython.com/python311-new-features/)

Python3.11于2022年10月24日推出，这个新的版本优化了执行速度同时更用户友好。在经过17个月的开发后终于供用户使用。

和每个版本一样，Python3.11进行了大量提升和改进。你可以该[列表](https://docs.python.org/3.11/whatsnew/3.11.html)中查看全部。下面我们看下当中最酷、最有效的新特性。

**在本篇指南中你将了解如下新特性：**

* 有更多信息丰富的回溯的**报错提示**
* **更快的代码执行**由于在Faster CPython项目中付出的努力
* 异步代码中的**任务和异常组**被简化
* Python**静态类型支持**引入几个新类型特性
* 操作配置文件时的原生**TOML支持**

除了了解更多关于该语言的新特性之外，您还将获得一些关于升级到新版本之前应该考虑什么问题的建议。点击下面的链接，下载演示Python3.11新功能的代码示例：https://realpython.com/bonus/python-311-examples/

## 富含更多信息的错误回溯

Python通常被认为是一种很好的初学编程的语言，其语法[可读](https://realpython.com/python-pep8/)，[数据结构](https://realpython.com/python-data-structures/)强大。对于所有人，尤其是那些刚接触Python的人来说，如何解释Python遇到错误时显示的[回溯信息](https://realpython.com/python-traceback/)是一个挑战。

在Python3.10中，Python的错误消息得到了极大的改进。同样,Python3.11最令人期待的特性之一也将提升您的开发体验。装饰性注释被添加到回溯中，可以帮助您更快地解释错误消息。

要查看增强的回溯的快速示例，将以下代码添加到名为`reverse .py`的文件中:

```python
# inverse.py

def inverse(number):
    return 1 / number

print(inverse(0))
```

你可以使用`inverse()`来计算一个数字的[倒数](https://en.wikipedia.org/wiki/Multiplicative_inverse)。但是0没有倒数，所以你的代码会在运行时引发一个错误：
```shell
$ python inverse.py
Traceback (most recent call last):
  File "/home/realpython/inverse.py", line 6, in <module>
    print(inverse(0))
          ^^^^^^^^^^
  File "/home/realpython/inverse.py", line 4, in inverse
    return 1 / number
           ~~^~~~~~~~
ZeroDivisionError: division by zero
```

注意嵌入在回溯中的^和~符号。它们用于引导你注意导致错误的代码。与通常的回溯一样，你应该从底部开始，然后逐步向上。在这个例子中，`ZeroDivisionError`是由除法`1 / 数字`引起的。真正的罪魁祸首是调用逆`inverse(0)`，因为0没有倒数。

在发现错误时得到额外的帮助是有用的。但是，如果您的代码更复杂，带注释的回溯功能会更强大。它们能够传递您以前无法从跟踪本身获得的信息。

为了欣赏改进后回溯的强大功能，您将构建一个关于几个程序员的信息的小型解析器。假设您有一个名为`programmers.json`的文件，内容如下:
```json
[
    {"name": {"first": "Uncle Barry"}},
    {
        "name": {"first": "Ada", "last": "Lovelace"},
        "birth": {"year": 1815},
        "death": {"month": 11, "day": 27}
    },
    {
        "name": {"first": "Grace", "last": "Hopper"},
        "birth": {"year": 1906, "month": 12, "day": 9},
        "death": {"year": 1992, "month": 1, "day": 1}
    },
    {
        "name": {"first": "Ole-Johan", "last": "Dahl"},
        "birth": {"year": 1931, "month": 10, "day": 12},
        "death": {"year": 2002, "month": 6, "day": 29}
    },
    {
        "name": {"first": "Guido", "last": "Van Rossum"},
        "birth": {"year": 1956, "month": 1, "day": 31},
        "death": null
    }
]
```

注意，关于程序员的信息是不完整的。虽然关于Grace Hopper和 Ole-Johan Dahl的信息是完整的，但你却错过了Ada Lovelace的出生日期和月份，以及她的死亡年份。当然，你只有Guido van Rossum的出生信息。最糟糕的是，你只录下了Uncle Barry的名字。

您将创建一个可以包装此信息的类。首先从JSON文件中读取信息：
```python
# programmers.py

import json
import pathlib

programmers = json.loads(
    pathlib.Path("programmers.json").read_text(encoding="utf-8")
)
```

你使用[pathlib](https://realpython.com/python-pathlib/)库读取json文件然后把json信息解析成Python字典。

接下来，你将使用一个[数据类](https://realpython.com/python-data-classes/)来封装关于每个程序员的信息：
```python
# programmers.py

from dataclasses import dataclass

# ...

@dataclass
class Person:
    name: str
    life_span: tuple[int, int]

    @classmethod
    def from_dict(cls, info):
        return cls(
            name=f"{info['name']['first']} {info['name']['last']}",
            life_span=(info["birth"]["year"], info["death"]["year"]),
        )
```

每个`Person`都有一个`name`和一个`life_span`属性。此外，还添加了一个方便的[构造函数](https://realpython.com/python-multiple-constructors/)，可以根据JSON文件中的信息和结构初始化Person。

你还将添加一个函数，可以一次初始化两个`Person`对象：
```python
# programmers.py

# ...

def convert_pair(first, second):
    return Person.from_dict(first), Person.from_dict(second)
```

`convert_pair()`函数使用`.from_dict()`构造函数两次将两个程序员从JSON结构转换为`Person`对象。

现在是时候研究你的代码，特别是查看一些回溯。使用`-i`参数运行程序，打开Python的交互式REPL，其中包含所有可用的变量、类和函数：
```shell
$ python -i programmers.py
>>> Person.from_dict(programmers[2])
Person(name='Grace Hopper', life_span=(1906, 1992))
```
Grace的信息是完整的，因此你可以将她封装到一个`Person`对象中，其中包含关于她的全名和生命周期的信息。

要查看新的回溯操作，请尝试转换`Uncle Barry`：
```python
>>> programmers[0]
{'name': {'first': 'Uncle Barry'}}

>>> Person.from_dict(programmers[0])
Traceback (most recent call last):
  File "/home/realpython/programmers.py", line 17, in from_dict
    name=f"{info['name']['first']} {info['name']['last']}",
                                    ~~~~~~~~~~~~^^^^^^^^
KeyError: 'last'
```
你得到一个`KeyError`，因为`last`字段缺失了。虽然你可能记得`last`是`name`中的子字段，但注释立即为你指出了这一点。

类似地，回忆一下关于Ada的寿命信息是不完整的。你不能为她创建一个`Person`对象：
```python
>>> programmers[1]
{
    'name': {'first': 'Ada', 'last': 'Lovelace'},
    'birth': {'year': 1815},
    'death': {'month': 11, 'day': 27}
}

>>> Person.from_dict(programmers[1])
Traceback (most recent call last):
  File "/home/realpython/programmers.py", line 18, in from_dict
    life_span=(info["birth"]["year"], info["death"]["year"]),
                                      ~~~~~~~~~~~~~^^^^^^^^
KeyError: 'year'
```

你将得到另一个`KeyError`，这一次是因为缺少年份。在这种情况下，回溯甚至比前面的示例更有用。年份有两个子字段，一个是出生日期，一个是死亡日期。回溯注释立即显示你缺少一个死亡日期。

输入Guido会怎么样？你只有关于他出生的信息：
```python
>>> programmers[4]
{
    'name': {'first': 'Guido', 'last': 'Van Rossum'},
    'birth': {'year': 1956, 'month': 1, 'day': 31},
    'death': None
}

>>> Person.from_dict(programmers[4])
Traceback (most recent call last):
  File "/home/realpython/programmers.py", line 18, in from_dict
    life_span=(info["birth"]["year"], info["death"]["year"]),
                                      ~~~~~~~~~~~~~^^^^^^^^
TypeError: 'NoneType' object is not subscriptable
```

在这种情况下，将引发`TypeError`。你以前可能见过这种类型的`NoneType`类型错误。它们非常难以调试，因为不清楚哪个对象意外地为`None`。但是，从注释中可以看到，在本例中info["death"]为None。

在最后一个示例中，你将探索使用嵌套函数调用会发生什么。记住`convert_pair()`调用`Person.from_dict()`两次。现在，试着使用Ada和Ole-Johan:
```python
>>> convert_pair(programmers[3], programmers[1])
Traceback (most recent call last):
  File "/home/realpython/programmers.py", line 24, in convert_pair
    return Person.from_dict(first), Person.from_dict(second)
                                    ^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/realpython/programmers.py", line 18, in from_dict
    life_span=(info["birth"]["year"], info["death"]["year"]),
                                      ~~~~~~~~~~~~~^^^^^^^^
KeyError: 'year'
```
试图封装Ada会引发与前面相同的`KeyError`。但是，请注意`convert_pair()`中的回溯。由于该函数调用`.from_dict()`两次，通常需要一些工作来确定在`第一次`调用或`第二次`调用时引发了错误。在最新版本的Python中，您可以立即看到这些问题是由`第二次`引起的。

这些回溯使得在Python3.11中进行调试比在早期版本中更容易。在Python 3.11预览教程[《更好的错误消息》](https://realpython.com/python311-error-messages/)中，你可以看到更多示例，关于如何实现回溯的更多信息，以及在调试中可以使用的其他工具。有关更多技术细节，请参阅[PEP 657](https://peps.python.org/pep-0657/)。

作为Python开发人员，带注释的回溯将极大地提高你的工作效率。另一个令人兴奋的发展是Python3.11是迄今为止最快的Python版本。

## 更快的代码执行

Python被认为是一种缓慢的语言。例如，Python中的常规循环要比C中的类似循环慢几个数量级。这个缺点可以通过几种方式加以弥补。通常程序员的工作效率比代码执行时间更重要。

Python还非常能够包装用更快的语言编写的库。例如，在NumPy中进行的计算比在纯Python中进行的类似计算快得多。再加上易于开发代码，这使得Python成为数据科学领域的有力竞争者。

不过，一直在有动力在使Python语言变得更快。在2020年秋天，Mark Shannon提出了几个可以在Python中实现的性能改进。这个被称为香农计划(Shannon Plan)的提议非常雄心勃勃，希望在几个版本中使Python速度提高五倍。

微软已经加入进来，目前正在支持一组开发人员(包括Mark Shannon和Python的创建者Guido van rossum)致力于现在所知的Faster CPython项目。基于Faster CPython项目的Python 3.11有很多改进。在本节中，你将了解**专门的自适应解释器**。在后面的小节中，您还将了解[更快的启动时间](https://realpython.com/python311-new-features/#faster-startup)和[零成本异常](https://realpython.com/python311-new-features/#zero-cost-exceptions)。

[PEP 659](https://peps.python.org/pep-0659/)描述了一个专门化的自适应解释器。其主要思想是通过优化经常执行的操作来加速代码的运行。这类似于即时(JIT)编译，只是它不影响编译。相反，Python的字节码是动态调整或更改的。

Python代码在运行之前被编译为字节码。字节码由比常规Python代码更基本的指令组成，因此Python的每一行都转换为几个字节码语句。

你可以使用`dis`来查看Python的字节码。例如，考虑一个可以将英尺转换为米的函数：
```python
>>> def feet_to_meters(feet):
...     return 0.3048 * feet
...
```
你可以通过调用`dis.dis()`将这个函数分解成字节码：
```python
>>> import dis
>>> dis.dis(feet_to_meters)
  1           0 RESUME                   0

  2           2 LOAD_CONST               1 (0.3048)
              4 LOAD_FAST                0 (feet)
              6 BINARY_OP                5 (*)
             10 RETURN_VALUE
```

每行显示关于一个字节码指令的信息。这五列分别是行号、字节地址、操作代码名称、操作参数和括号中参数的解释。

一般来说，编写Python代码不需要了解字节码。不过，它可以帮助你理解Python内部是如何工作的。

在字节码生成中添加了一个称为加速的新步骤。这将使用在运行时可以优化的指令，并将它们替换为自适应指令。每个这样的指令都将查看它是如何使用的，并可能相应地专门化自己。

一旦函数被调用了一定次数，加速就开始了。在CPython3.11中，这发生在八次调用之后。你可以通过调用`dis()`并设置自适应参数来观察解释器如何适应字节码。首先定义一个函数，并以浮点数作为参数调用它7次：
```python
>>> def feet_to_meters(feet):
...     return 0.3048 * feet
...

>>> feet_to_meters(1.1)
0.33528
>>> feet_to_meters(2.2)
0.67056
>>> feet_to_meters(3.3)
1.00584
>>> feet_to_meters(4.4)
1.34112
>>> feet_to_meters(5.5)
1.6764000000000001
>>> feet_to_meters(6.6)
2.01168
>>> feet_to_meters(7.7)
2.34696
```

接下来，看一下`foot_to_meters()`的字节码：
```python
>>> import dis
>>> dis.dis(feet_to_meters, adaptive=True)
  1           0 RESUME                   0

  2           2 LOAD_CONST               1 (0.3048)
              4 LOAD_FAST                0 (feet)
              6 BINARY_OP                5 (*)
             10 RETURN_VALUE
```

你还不会观察到任何特别的东西。此版本的字节码仍然与非自适应字节码相同。当你第八次调用`feet_to_meters()`时，情况发生了变化：
```python
>>> feet_to_meters(8.8)
2.68224

>>> dis.dis(feet_to_meters, adaptive=True)
  1           0 RESUME_QUICK                 0

  2           2 LOAD_CONST__LOAD_FAST        1 (0.3048)
              4 LOAD_FAST                    0 (feet)
              6 BINARY_OP_MULTIPLY_FLOAT     5 (*)
             10 RETURN_VALUE
```

现在，原来的一些指令已经被专门的指令所取代。例如，`BINARY_OP`已经专门用于`BINARY_OP_MULTIPLY_FLOAT`，它在两个浮点数相乘时速度更快。

即使`footet_to_meters()`已经针对`feet`是浮点形参的情况进行了优化，它仍然可以通过返回到原始字节码指令来正常工作于其他类型的形参。内部操作已经更改，但代码的行为将与以前完全相同。

专门化指令仍然是自适应的。再调用你的函数52次，但是现在使用一个整数参数：
```python
>>> for feet in range(52):
...     feet_to_meters(feet)
...

>>> dis.dis(feet_to_meters, adaptive=True)
  1           0 RESUME_QUICK                 0

  2           2 LOAD_CONST__LOAD_FAST        1 (0.3048)
              4 LOAD_FAST                    0 (feet)
              6 BINARY_OP_MULTIPLY_FLOAT     5 (*)
             10 RETURN_VALUE
```

Python解释器仍然希望能够将两个浮点数相乘。当你用一个整数再次调用`feet_to_meters()`时，它会退出并转换回一个非专门化的自适应指令：
```python
>>> feet_to_meters(52)
15.8496

>>> dis.dis(feet_to_meters, adaptive=True)
  1           0 RESUME_QUICK              0

  2           2 LOAD_CONST__LOAD_FAST     1 (0.3048)
              4 LOAD_FAST                 0 (feet)
              6 BINARY_OP_ADAPTIVE        5 (*)
             10 RETURN_VALUE
```

在这种情况下，字节码指令被更改为`BINARY_OP_ADAPTIVE`而不是`BINARY_OP_MULTIPLY_INT`，因为其中一个操作符0.3048始终是一个浮点数。

整数和浮点数之间的乘法比相同类型的数之间的乘法更难优化。至少到目前为止，还没有专门的指令来在`float`和`int`之间进行乘法运算。

这个示例旨在让你了解自适应专门化解释器是如何工作的。一般来说，你不应该担心更改现有代码来利用它。你的大部分代码会运行得更快。

也就是说，在一些情况下，你可以重构代码，从而使其更有效地专门化。Brandt Bucher的[specialist](https://pypi.org/project/specialist/)是一个工具，它可以可视化解释器如何处理你的代码。[本教程](https://github.com/brandtbucher/specialist#tutorial)展示了一个手动改进代码的示例。你可以在[Talk Python to Me](https://talkpython.fm/episodes/show/381/python-perf-specializing-adaptive-interpreter)播客中了解更多。

对于Faster CPython项目，有几个重要的指导方针:
* 该项目不会对Python引入任何破坏性的更改。
* 大多数代码的性能应该得到改进。

在基准测试中，CPython3.11平均比CPython3.10快25%。然而，你应该更感兴趣的是Python3.11在代码上的表现，而不是它在基准测试上的表现。展开下面的方框，了解如何衡量自己代码的性能：

一般来说，有三种方法可以用来衡量代码性能:

* 对程序中重要的小段代码进行基准测试。
* 分析您的程序以找到可以改进的瓶颈。
* 监视整个程序的性能。

通常情况下，你想做全部这些事。在开发代码时，基准测试可以帮助你在不同的实现之间进行选择。Python内置了`timeit`模块对微基准测试的支持。第三方[richbench](https://pypi.org/project/richbench/)工具非常适合对函数进行基准测试。此外，[pyperformance](https://pyperformance.readthedocs.io/)是Faster CPython项目用来衡量改进的基准套件。

如果您需要加快程序的速度，并想要找出应该关注代码的哪一部分，那么代码分析器是很有用的。Python的标准库提供了[cProfile](https://docs.python.org/3.10/library/profile.html#module-cProfile)和[pstats](https://docs.python.org/3.10/library/profile.html#the-stats-class), cProfile可以用来收集程序的统计信息，pstats可以用来查看这些统计信息。

第三种方法是监视程序的运行时，对于所有运行时间超过几秒钟的程序都应该这样做。最简单的方法是在日志消息中添加计时器。第三方库[codetiming](https://pypi.org/project/codetiming/)允许你这样做，例如通过向主函数添加装饰器。

让Python变得更快的一种可行且重要的方法是共享举例说明用例的基准测试。特别是如果你在Python 3.11中没有注意到太多的加速，如果你能够分享你的代码，这将对核心开发人员有帮助。有关更多信息，请参阅Mark Shannon的闪电演讲，[如何帮助加速Python](https://www.youtube.com/watch?v=xQ0-aSmn9ZA&t=1189s)。

Faster CPython项目正在进行中，已经有几个优化计划于2023年10月与Python 3.12一起发布。你可以在GitHub上关注这个项目。要了解更多信息，您还可以查看以下讨论和演示：

* Talk Python to Me: [Making Python Faster With Guido and Mark](https://talkpython.fm/episodes/show/339/making-python-faster-with-guido-and-mark)
* EuroPython 2022: [How We Are Making Python 3.11 Faster](https://www.youtube.com/watch?v=4ytEjwywUMM&t=9748s) by Mark Shannon
* Talk Python to Me: [Python Perf: Specializing, Adaptive Interpreter](https://talkpython.fm/episodes/show/381/python-perf-specializing-adaptive-interpreter) with Brandt Bucher
* PyCon ES: [Faster CPython Project: Como estamos haciendo Python 3.11 más rápido](https://charlas.2022.es.pycon.org/pycones2022/talk/D9ABNU/?linkId=182586474) by Pablo Galindo Salgado, in Spanish

Faster CPython是一个庞大的项目，涉及Python的所有部分。自适应专门化解释器是这项工作的一部分。在本教程的后面部分，你将了解另外两种优化：[更快的启动时间](https://realpython.com/python311-new-features/#faster-startup)和[零成本异常](https://realpython.com/python311-new-features/#zero-cost-exceptions)。

## 异步任务的更好语法

Python对异步编程的支持有相当长的进化周期。Python2时代通过添加[生成器](https://peps.python.org/pep-0255/)奠定了基础。[asyncio](https://peps.python.org/pep-3156/)库最初是在Python3.4中添加的，[async和await](https://peps.python.org/pep-0492/)关键字也在Python3.5中添加。

在后来的版本中继续进行开发，在Python的异步功能中添加了许多小的改进。在Python3.11中，你可以使用任务组，它为运行和监视异步任务提供了更清晰的语法。

如果你还不熟悉Python中的异步编程，那么你可以查看以下资源来开始：
* [Speed Up Your Python Program With Concurrency](https://realpython.com/python-concurrency/)
* [Getting Started With Async Features in Python](https://realpython.com/python-async-features/)
* [Async IO in Python: A Complete Walkthrough](https://realpython.com/async-io-python/)

`asyncio`库是Python标准库的一部分。然而，这并不是异步工作的唯一方式。有几个流行的第三方库提供相同的功能，包括[Trio](https://trio.readthedocs.io/)和[Curio](https://curio.readthedocs.io/)。此外，像[uvloop](https://uvloop.readthedocs.io/)、[AnyIO](https://anyio.readthedocs.io/)和[Quattro](https://pypi.org/project/quattro/)这样的库通过更好的性能和更多的特性增强了`asyncio`。

使用`asyncio`运行多个异步任务的传统方法是使用`create_task()`创建任务，然后使用`gather()`进行await。这可以完成任务，但使用起来有点麻烦。

为了组织子任务，Curio引入了[任务组](https://curio.readthedocs.io/en/latest/tutorial.html#task-groups)，Trio引入了[托儿所](https://trio.readthedocs.io/en/stable/reference-core.html#tasks)作为替代方案。新的异步任务组在很大程度上受到了这些元素的启发。

当你用`gather()`组织你的异步任务时，你的部分代码通常看起来像这样：
```python
tasks = [asyncio.create_task(run_some_task(param)) for param in params]
await asyncio.gather(*tasks)
```

在将任务传递给`gather()`之前，您可以手动跟踪列表中的所有任务。通过等待`gather()`，可以确保每个任务在继续之前都已完成。

任务组的等效代码更加直接。不是使用`gather()`，而是使用上下文管理器来定义何时等待任务：
```python
async with asyncio.TaskGroup() as tg:
    for param in params:
        tg.create_task(run_some_task(param))
```

创建一个任务组对象，在本例中名为`tg`，并使用它的`.create_task()`方法创建新任务。

来看一个完整的示例，请考虑下载几个文件的任务。您希望下载一些历史PEP文档的文本，这些文档展示了Python的异步特性是如何开发的。为了提高效率，您将使用第三方库[aiohttp](https://pypi.org/project/aiohttp/)异步下载文件。

首先导入必要的库，并注意存储每个PEP文本的存储库的URL：
```python
# download_peps_gather.py

import asyncio
import aiohttp

PEP_URL = (
    "https://raw.githubusercontent.com/python/peps/master/pep-{pep:04d}.txt"
)

async def main(peps):
    async with aiohttp.ClientSession() as session:
        await download_peps(session, peps)
```

你可以添加一个`main()`函数来初始化`aiohttp`会话，以管理可以复用的连接池。现在，调用一个名为`download_pep()`的函数，该函数还没有编写。这个函数将为每个需要下载的PEP创建一个任务：
```python
# download_peps_gather.py

# ...

async def download_peps(session, peps):
    tasks = [asyncio.create_task(download_pep(session, pep)) for pep in peps]
    await asyncio.gather(*tasks)
```

这遵循了前面看到的模式。每个任务由运行`download_pep()`组成，你将在下一步中定义它。设置完所有任务后，将它们传递给`gather()`。

每个任务下载一个PEP。这里添加一些`print()调用`，这样你就可以看到发生了什么：
```python
# download_peps_gather.py

# ...

async def download_pep(session, pep):
    print(f"Downloading PEP {pep}")
    url = PEP_URL.format(pep=pep)
    async with session.get(url, params={}) as response:
        pep_text = await response.text()

    title = pep_text.split("\n")[1].removeprefix("Title:").strip()
    print(f"Downloaded PEP {pep}: {title}")
```

对于每个PEP，你可以找到其单独的URL并调用`session.get()`下载。得到PEP的文本后，就可以找到PEP的标题并将其输出到控制台。

最后，异步运行`main()`：
```python
# download_peps_gather.py

# ...

asyncio.run(main([492, 525, 530, 3148, 3156]))
```

使用PEP编号列表调用代码，这些编号都是与Python中的异步有关的特性。运行脚本看看它是如何工作的：
```shell
$ python download_peps_gather.py
Downloading PEP 492
Downloading PEP 525
Downloading PEP 530
Downloading PEP 3148
Downloading PEP 3156
Downloaded PEP 3148: futures - execute computations asynchronously
Downloaded PEP 492: Coroutines with async and await syntax
Downloaded PEP 530: Asynchronous Comprehensions
Downloaded PEP 3156: Asynchronous IO Support Rebooted: the "asyncio" Module
Downloaded PEP 525: Asynchronous Generators
```

可以看到所有的下载都是同时进行的，因为所有任务都打印出它们在任何任务报告完成之前开始下载PEP。另外，请注意任务是按照预先定义的顺序开始的，PEP编号是按从小打到排列的。

相反，任务完成的顺序似乎是随机的。调用`gather()`确保在完成所有异步任务后再执行主代码。

你可以更新代码以使用任务组而不是`gather()`。首先，将`download_peps_gather.py`复制到名为`download_peps_taskgroup.py`的新文件中。这些文件将非常相似，你只需要编辑`download_pep()`函数：
```python
# download_peps_taskgroup.py

# ...

async def download_peps(session, peps):
    async with asyncio.TaskGroup() as tg:
        for pep in peps:
            tg.create_task(download_pep(session, pep))
# ...
```

请注意，代码遵循示例之前概述的通用模式。首先在[上下文管理器](https://realpython.com/python-with-statement/)中设置一个任务组，然后使用该任务组创建子任务：每个任务下载一个PEP文件。运行更新后的代码，观察它的行为是否与早期版本相同。

当你处理多个异步任务时，一个挑战是它们中的任何一个都可能在任何时候引发错误。理论上，两个或多个任务甚至可以同时产生错误。

像[Trio](https://trio.readthedocs.io/en/stable/reference-core.html#working-with-multierrors)和[Curio](https://trio.readthedocs.io/en/stable/reference-core.html#working-with-multierrors)这样的库已经用一种特殊的多错误对象处理了这个问题。这是解决办法有效但有点麻烦，因为Python没有提供太多的内置支持。

为了在任务组中正确地支持错误处理，Python3.11引入了异常组，用于跟踪多个并发错误。在本教程的后面，你将了解更多有关它们的信息。

任务组使用异常组来提供比旧方法更好的错误处理支持。有关任务组的更深入讨论，请参见Python 3.11预览：[任务和异常组](https://realpython.com/python311-exception-groups/#asynchronous-task-groups-in-python-311)。你可以在Guido van Rossum的[Reasoning about asyncio.Semaphore](https://neopythonic.blogspot.com/2022/10/reasoning-about-asynciosemaphore.html)中了解更多关于`asyncio.Semaphore`的基本原则。

## 改进的Type类型

Python是一种动态类型语言，但它通过[类型提](https://realpython.com/python-type-checking/)示支持静态类型。Python静态类型系统的基础在2015年的[PEP 484中](https://peps.python.org/pep-0484)定义。自Python3.5以来，每个Python版本都引入了几个与类型相关的新提议。

Python3.11宣布了五个与类型相关的pep——创历史新高：
* [PEP 646](https://peps.python.org/pep-0646)：可变泛型
* [PEP 655](https://peps.python.org/pep-0655)：根据需要或可能丢失的情况标记单个TypedDict项
* [PEP 673](https://peps.python.org/pep-0673)：Self类型
* [PEP 675](https://peps.python.org/pep-0675)：任意文字字符串类型
* [PEP 681](https://peps.python.org/pep-0681)：数据类转换

在本节中，将重点关注其中的两个:可变泛型和Self类型。有关更多信息，请查看PEP文档以及[Python 3.11预览](https://realpython.com/python311-tomllib/#other-new-features)。

* 注意：对类型特性的支持取决于你的类型检查器以及你的Python版本。例如，在Python3.11发布时，[mypy](https://mypy.readthedocs.io/)[不支持](https://github.com/python/mypy/issues/12840)一些新特性。

[类型变量](https://realpython.com/python-type-checking/#type-variables)从一开始就是Python静态类型系统的一部分。可以使用它们参数化泛型。换句话说，如果你有一个列表，那么你可以使用一个类型变量来检查列表中项目的类型：
```python
from typing import Sequence, TypeVar

T = TypeVar("T")

def first(sequence: Sequence[T]) -> T:
    return sequence[0]
```
`first()`函数从序列类型(如列表)中挑选出第一个元素。该代码的工作原理与序列元素的类型无关。不过，你仍然需要跟踪元素类型，以便知道`first()`的返回类型。

type变量正是这样做的。例如，如果您将一个整数列表传递给first()，那么在类型检查时T将被设置为int。因此，类型检查器可以推断first()的调用将返回int。在本例中，列表被称为**泛型**，因为它可以被其他类型参数化。

一种随着时间的推移而发展起来的模式试图解决引用当前类的类型提示的问题。回想一下前面的Person类：
```python
# programmers.py

from dataclasses import dataclass

# ...

@dataclass
class Person:
    name: str
    life_span: tuple[int, int]

    @classmethod
    def from_dict(cls, info):
        return cls(
            name=f"{info['name']['first']} {info['name']['last']}",
            life_span=(info["birth"]["year"], info["death"]["year"]),
        )
```

`.from_dict()`构造函数返回一个`Person`对象。但是，不允许使用 -> Person作为`.from_dict()`返回值的类型提示，因为此时在代码中`Person`类还没有完全定义。

此外，如果允许使用 -> Person，那么这将不能很好地用于继承。如果你创建了`Person`的子类，那么`.from_dict()`将返回该子类而不是`Person`对象。

这个问题的一个解决方案是使用类型变量绑定到你的类：
```python
# programmers.py

# ...

from typing import Any, Type, TypeVar

TPerson = TypeVar("TPerson", bound="Person")

@dataclass
class Person:
    name: str
    life_span: tuple[int, int]

    @classmethod
    def from_dict(cls: Type[TPerson], info: dict[str, Any]) -> TPerson:
        return cls(
            name=f"{info['name']['first']} {info['name']['last']}",
            life_span=(info["birth"]["year"], info["death"]["year"]),
        )
```

你可以指定`bound`以确保`TPerson`永远只能是`Person`或它的一个子类。这个模式可行但是可读性不是特别强。它还强迫注释`self`或`cls`，这通常是不必要的。

现在可以使用新的[Self](https://docs.python.org/3.11/library/typing.html#typing.Self)类型。它总是引用封装类，因此你不必手动定义类型变量。下面的代码等价于前面的例子：
```python
# programmers.py

# ...

from typing import Any, Self

@dataclass
class Person:
    name: str
    life_span: tuple[int, int]

    @classmethod
    def from_dict(cls, info: dict[str, Any]) -> Self:
        return cls(
            name=f"{info['name']['first']} {info['name']['last']}",
            life_span=(info["birth"]["year"], info["death"]["year"]),
        )
```

你可以从`typing`中导入`Self`，你不需要创建类型变量或注解`cls`。相反，该方法返回`Self`，它将引用`Person`。

请参阅[Python 3.11预览](https://realpython.com/python311-tomllib/#self-type)，当中有如何使用Self的另一个示例。你还可以查看[PEP 673](https://peps.python.org/pep-0673/)以获得更多详细信息。

类型变量的一个限制是它们一次只能代表一种类型。假设你有一个函数，它颠倒了一个二元元组的顺序：
```python
# pair_order.py

def flip(pair):
    first, second = pair
    return (second, first)
```

假设pair是一个有两个元素的元组。元素可以是不同的类型，所以你需要两个类型变量来注解你的函数：
```python
# pair_order.py

from typing import TypeVar

T0 = TypeVar("T0")
T1 = TypeVar("T1")

def flip(pair: tuple[T0, T1]) -> tuple[T1, T0]:
    first, second = pair
    return (second, first)
```

写起来有点麻烦，但也还凑合。注解是显式可读的。如果你想注解你的代码的一个变体，它适用于任意数量的元组，那么挑战就来了：
```python
# tuple_order.py

def cycle(elements):
    first, *rest = elements
    return (*rest, first)
```

使用`cycle()`可以将第一个元素移动到具有任意数量元素的元组的末尾。如果传入一对元素，则这相当于`flip()`。

考虑一下如何注解`cycle()`。如果`elements`是一个包含n个元素的元组，则需要n个类型变量。但是元素的数量可以是任意的，所以你不知道你需要多少类型变量。

[PEP 646](https://peps.python.org/pep-0646/)引入[TypeVarTuple](https://docs.python.org/3.11/library/typing.html#typing.TypeVarTuple)来处理这个例子。`TypeVarTuple`可以代表任意数量的类型。因此，你可以使用它来注解泛型。

可以按如下进行操作：
```python
# tuple_order.py

from typing import TypeVar, TypeVarTuple

T0 = TypeVar("T0")
Ts = TypeVarTuple("Ts")

def cycle(elements: tuple[T0, *Ts]) -> tuple[*Ts, T0]:
    first, *rest = elements
    return (*rest, first)
```

`TypeVarTuple`将替换任意数量的类型，因此此注解将适用于包含1,3,11或任何其他数量元素的元组。

注意，Ts前面的星号(*)是语法的必要部分。它类似于你已经在代码中使用的解包语法，它提醒你Ts代表任意数量的类型。

为了结束关于类型注解这一节，请回忆一下静态类型结合了两种不同的工具：Python语言和类型检查器。要使用新的类型特性，Python版本必须支持它们。此外，它们还需要类型检查器的支持。

许多类型特性，包括`Self`和`TypeVarTuple`，在`typing_extensions`包中被反向移植到旧版本的Python中。在Python3.10上，你可以使用pip在你的虚拟环境中安装类型扩展，然后实现最后一个例子，如下所示：
```python
# tuple_order.py

from typing_extensions import TypeVar, TypeVarTuple, Unpack

T0 = TypeVar("T0")
Ts = TypeVarTuple("Ts")

def cycle(elements: tuple[T0, Unpack[Ts]]) -> tuple[Unpack[Ts], T0]:
    first, *rest = elements
    return (*rest, first)
```

*Ts语法只在Python3.11中被允许。一个适用于旧版Python的等效替代方法是Unpack[Ts]。即使你的代码可以在你的Python版本上运行，也不是所有的类型检查器都支持`TypeVarTuple`。

## 支持TOML配置解析

TOML是**Tom's Obvious Minimal Language**的缩写。

这是一种配置文件格式，在过去十年中变得流行起来。在为包和项目[指定元数据](https://peps.python.org/pep-0518/)时，Python社区已经将[TOML](https://toml.io/en/)作为首选格式。

TOML的设计使人类易于阅读，计算机也易于解析。您可以在[Python and TOML: New Best Friends](https://realpython.com/python-toml/)中了解配置文件格式本身。

虽然TOML已被许多不同的工具使用多年，但Python还没有内置的TOML支持。直到在Python 3.11中，[tomllib](https://docs.python.org/3.11/library/tomllib.html)被添加到标准库。这个新模块构建在流行的第三方库[tomli](https://pypi.org/project/tomli/)之上，允许你解析TOML文件。

下面是一个名为`units.toml`的TOML文件示例：
```TOML
# units.toml

[second]
label   = { singular = "second", plural = "seconds" }
aliases = ["s", "sec", "seconds"]

[minute]
label      = { singular = "minute", plural = "minutes" }
aliases    = ["min", "minutes"]
multiplier = 60
to_unit    = "second"

[hour]
label      = { singular = "hour", plural = "hours" }
aliases    = ["h", "hr", "hours"]
multiplier = 60
to_unit    = "minute"

[day]
label      = { singular = "day", plural = "days" }
aliases    = ["d", "days"]
multiplier = 24
to_unit    = "hour"

[year]
label      = { singular = "year", plural = "years" }
aliases    = ["y", "yr", "years", "julian_year", "julian years"]
multiplier = 365.25
to_unit    = "day"
```

该文件包含几部分，标题用方括号括起来。每个这样的部分在TOML中称为**table**，其标题称为**key**。tables包含**键值对**。可以嵌套表，使值成为新表。在上面的示例中，可以看到每个表(除了第二个表)都具有相同的结构，有四个键：label，aliases，multiplier，and to_unit。

值可以有不同的类型。在这个例子中，你可以看到四种数据类型：
1. label 是一个**内联表**，类似于Python的字典。
2. aliases 是一个**数组**，类似于Python的列表。
3. multiplier 是一个**数字**，可以是整数，也可以是浮点数。
4. to_unit 是一个**字符串**。

TOML还支持其他数据类型，包括布尔和日期。查看[Python and TOML: New Best Friends](https://realpython.com/python-toml/#get-to-know-toml-key-value-pairs)更深入的了解其语法。

你可以用`tomlib`读取一个TOML文件：
```python
>>> import tomllib
>>> with open("units.toml", mode="rb") as file:
...     units = tomllib.load(file)
...
>>> units
{'second': {'label': {'singular': 'second', 'plural': 'seconds'}, ... }}
```

当使用`tomllib.load()`你通过指定mode="rb"传入一个*二进制模式*打开的文件对象。或者，你可以使用`tomllib.loads()`解析字符串：
```python
>>> import tomllib
>>> import pathlib
>>> units = tomllib.loads(
...     pathlib.Path("units.toml").read_text(encoding="utf-8")
... )
>>> units
{'second': {'label': {'singular': 'second', 'plural': 'seconds'}, ... }}
```

在本例中，首先使用[pathlib](https://realpython.com/python-pathlib/)读取`units.toml`为字符串。TOML文档应该以UTF-8编码存储。你应该显式地指定编码，以确保代码在所有平台上运行相同。

接下来，将注意力转向调用`load()`或`loads()`的结果。在上面的例子中，可以看到`units`是一个嵌套的字典。情况总是这样：`tomllib`将TOML文档解析到Python字典中。

在本节的其余部分中，你将练习在Python中使用TOML数据。你将创建一个小的单位转换器，用于解析TOML文件并使用生成的字典。

注意：如果你真的在做单位转换，那么你应该看看[Pint](http://pint.readthedocs.org/)。这个库可以在[数百个单位](https://github.com/hgrecco/pint/blob/master/pint/default_en.txt)之间转换，并且可以很好地集成到[NumPy](https://realpython.com/numpy-tutorial/)等其他包中。

将代码添加到一个名为`units.py`的文件中：
```python
# units.py

import pathlib
import tomllib

# Read units from file
with pathlib.Path("units.toml").open(mode="rb") as file:
    base_units = tomllib.load(file)
```
你希望能够通过名称或别名查找每个单位。你可以通过复制单位信息来实现这一点，这样每个别名都可以用作字典键：
```python
# units.py

# ...

units = {}
for unit, unit_info in base_units.items():
    units[unit] = unit_info
    for alias in unit_info["aliases"]:
        units[alias] = unit_info
```

例如，你的单位字典现在有了键`second`以及它的别名`s`、`sec`和`seconds`，它们都指向`second`表。

接下来，你将定义`to_baseunit()`，它可以将TOML文件中的任何单位转换为相应的基本单位。在本例中，基本单位始终是`second`。但是，你可以扩展表，例如，包含以米为基本单位的长度单位。

在你的文件中添加`to_baseunit()`：
```python
# units.py

# ...

def to_baseunit(value, from_unit):
    from_info = units[from_unit]
    if "multiplier" not in from_info:
        return (
            value,
            from_info["label"]["singular" if value == 1 else "plural"],
        )

    return to_baseunit(value * from_info["multiplier"], from_info["to_unit"])
```

将`to_baseunit()`实现为[递归函数](https://realpython.com/python-recursion/)。如果`from_unit`对应的表不包含`multiplier `字段，则将该单位视为基本单位，并返回其值和名称。另一方面，如果有一`multiplier`字段，则转换到链中的下一个单位并再次调用`to_baseunit()`。

启动你的REPL。然后，导入`units`并转换几个数字：
```python
>>> import units
>>> units.to_baseunit(7, "s")
(7, 'seconds')

>>> units.to_baseunit(3.11, "minutes")
(186.6, 'seconds')
```

在第一个例子中，“s”被解释为`second`，因为它是一个别名。因为这是基本单位，所以7被原封不动地返回。在第二个例子中，`minutes`使函数在`minute`表中查找。它发现它可以通过乘以`60`来转换为`second`。

转换链可以更长：
```python
>>> units.to_baseunit(14, "days")
(1209600, 'seconds')

>>> units.to_baseunit(1 / 12, "yr")
(2629800.0, 'seconds')
```

为了将“天”转换为基本单位，函数首先将日转换为小时，然后将小时转换为分钟，最后将分钟转换为秒。你会发现14天大约有120万秒，1/12年大约有260万秒。

注意:在本例中，你使用TOML文件存储有关单位转换器支持的单位的信息。你也可以将`base_units`定义为字面字典，从而将信息直接放入代码中。

然而，使用配置文件带来了与代码和数据分离相关的几个优点：
* 逻辑与数据分离。
* 非开发人员可以在不接触(甚至不了解)Python代码的情况下为单位转换器做出贡献。
* 只需付出最小的努力，就可以支持额外的单位配置文件。用户可以添加这些，以便在转换器中包括自定义单位。

你应该考虑为您使用的任何项目设置一个配置文件。

如前所述，`tomllib`是基于`tomli`的。如果你想在需要支持旧Python版本的代码中解析TOML文档，那么你可以安装`tomli`并将其用作`tomllib`的后端口，如下所示：
```python
try:
    import tomllib
except ModuleNotFoundError:
    import tomli as tomllib
```

在Python3.11上，它像往常一样导入`tomllib`。在早期版本的Python中，导入会引发ModuleNotFoundError。在这里，你捕获错误并导入`tomli`，同时将其别名为`tomllib`，以便其余代码能够正常工作。

你可以在[Python 3.11 Preview: TOML and tomllib](https://realpython.com/python311-tomllib/)学到更多使用技巧。此外，[PEP 680](https://peps.python.org/pep-0680/)概述了导致`tomllib`被添加到Python的讨论。

## 其他非常酷的特性

到目前为止，你已经了解了Python3.11中最大的变化和改进。然而，还有更多的特性需要探索。在本节中，你将了解一些可能隐藏在标题之下的新功能。它们包括更多的加速，对异常的更多改动，以及对字符串格式的小改进。

### 更快的启动

Faster CPython项目的另一个令人兴奋的成果是更快的启动时间。运行Python脚本时，解释器初始化时会发生几件事。这导致即使是最简单的程序也需要几毫秒的时间来运行：
```shell
$ time python -c "pass"
real    0m0,020s
user    0m0,012s
sys     0m0,008s
```

使用-c直接在命令行上传入程序。在这种情况下，整个程序由一个pass语句组成，它什么也不做。

在许多情况下，启动程序所需的时间与运行代码所需的时间相比可以忽略不计。然而，在运行时间较短的脚本(如典型的命令行应用程序)中，启动时间可能会显著影响程序的性能。

作为一个具体的例子，考虑以下脚本——灵感来自经典的[cowsay](https://en.wikipedia.org/wiki/Cowsay)程序：
```python
# snakesay.py
import sys

message = " ".join(sys.argv[1:])
bubble_length = len(message) + 2
print(
    rf"""
       {"_" * bubble_length}
      ( {message} )
       {"‾" * bubble_length}
        \
         \    __
          \  [oo]
             (__)\
               λ \\
                 _\\__
                (_____)_
               (________)Oo°"""
)
```

在snakesay.py中，从命令行读取消息。然后，你将信息打印在一个带有可爱的蛇的语音气泡中。现在，你可以让蛇说任何话：
```shell
$ python snakesay.py Faster startup!
       _________________
      ( Faster startup! )
       ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
        \
         \    __
          \  [oo]
             (__)\
               λ \\
                 _\\__
                (_____)_
               (________)Oo°
```

这是命令行应用程序的一个基本示例。与许多其他命令行应用程序一样，它运行很快。不过，运行起来还是需要几毫秒。这种开销的很大一部分发生在Python导入模块时，甚至包括一些你自己没有显式导入的模块。

你可以使用[-X import time](https://realpython.com/python37-new-features/#developer-tricks)来显示导入模块所花费的时间概述：
```shell
$ python -X importtime -S snakesay.py Imports are faster!
import time: self [us] | cumulative | imported package
import time:       283 |        283 |   _io
import time:        56 |         56 |   marshal
import time:       647 |        647 |   posix
import time:       587 |       1573 | _frozen_importlib_external
import time:       167 |        167 |   time
import time:       191 |        358 | zipimport
import time:        90 |         90 |     _codecs
import time:       561 |        651 |   codecs
import time:       825 |        825 |   encodings.aliases
import time:      1136 |       2611 | encodings
import time:       417 |        417 | encodings.utf_8
import time:       174 |        174 | _signal
import time:        56 |         56 |     _abc
import time:       251 |        306 |   abc
import time:       310 |        616 | io
       _____________________
      ( Imports are faster! )
       ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
        \
         \    __
          \  [oo]
             (__)\
               λ \\
                 _\\__
                (_____)_
               (________)Oo°
```

表中的数字以微秒为单位。注意最后一列中模块名称的格式。树结构表明存在一些顶级模块，这些模块导入其他模块。例如，`io`是一个顶级导入，而`abc`是由`io`导入的。

该示例在Python3.11上运行。下表将这些数字(以微秒为单位)与在Python3.10中运行相同命令进行了比较：
|Module|Python 3.11|Python 3.10|Speed-up|
|---|---|---|---|
|_frozen_importlib_external|1573|2255|1.43x|
|zipimport|358|558|1.56x|
|encodings|2611|3009|1.15x|
|encodings.utf_8|417|409|0.98x|
|_signal|174|173|0.99x|
|io|616|1216|1.97x|
|总计|5749|7620|1.33x|

你自己的数字会有所不同，但你应该看到相同的模式。Python3.11上的导入速度更快，这有助于Python程序更快地启动。

加速的一个重要原因是缓存字节码的存储和读取方式。正如你所了解的，Python将源代码编译为由解释器运行的字节码。很长一段时间以来，Python一直将编译后的字节码存储在一个名为[__pycache__](https://docs.python.org/3/tutorial/modules.html#compiled-python-files)的目录中，以避免不必要的重新编译。

但在最新版本的Python中，许多模块被冻结并以一种可以更快地将它们加载到内存中的方式存储。你可以在[文档](https://docs.python.org/3.11/whatsnew/3.11.html#faster-startup)中阅读更多关于快速启动的内容。

### 零成本异常
在Python3.11中，异常的内部表示不同。异常对象变得更加轻量级，异常处理也发生了变化，因此只要没有触发`except`子句，`try…except`语句的开销就很小。

所谓的[零成本异常](https://github.com/python/cpython/issues/84403)是受到C++和Java等其他语言的启发。我们的目标是，当没有异常时，这条快乐之路实际上应该是自由的（try...except没有开销）。处理异常仍然需要一些时间。

零成本异常是通过在源代码编译为字节码时让编译器创建跳转表来实现的。如果引发异常，将查询这些表。如果没有异常，则`try`块中的代码没有运行时开销。

回想一下前面倒数的例子。你添加了一些错误处理：
```python
>>> def inverse(number):
...     try:
...         return 1 / number
...     except ZeroDivisionError:
...         print("0 has no inverse")
...
```

如果试图计算0的倒数，则会引发`ZeroDivisionError`。在新的实现中，你捕捉这些错误并打印描述性消息。和前面一样，你使用`dis`来查看底层的字节码：
```python
>>> import dis
>>> dis.dis(inverse)
  1           0 RESUME                   0

  2           2 NOP

  3           4 LOAD_CONST               1 (1)
              6 LOAD_FAST                0 (number)
              8 BINARY_OP               11 (/)
             12 RETURN_VALUE
        >>   14 PUSH_EXC_INFO

  4          16 LOAD_GLOBAL              0 (ZeroDivisionError)
             28 CHECK_EXC_MATCH
             30 POP_JUMP_FORWARD_IF_FALSE    19 (to 70)
             32 POP_TOP

  5          34 LOAD_GLOBAL              3 (NULL + print)
             46 LOAD_CONST               2 ('0 has no inverse')
             48 PRECALL                  1
             52 CALL                     1
             62 POP_TOP
             64 POP_EXCEPT
             66 LOAD_CONST               0 (None)
             68 RETURN_VALUE

  4     >>   70 RERAISE                  0
        >>   72 COPY                     3
             74 POP_EXCEPT
             76 RERAISE                  1
ExceptionTable:
  4 to 10 -> 14 [0]
  14 to 62 -> 72 [1] lasti
  70 to 70 -> 72 [1] lasti
```

你不需要了解字节码的细节。但是，你可以将最左边的列中的数字与源代码中的行号进行比较。请注意，第2行`try:`被转换为一条NOP指令。这是一个**无操作**，什么都不做。更有趣的是，在分解的最后是一个异常表。这是解释器在需要处理异常时使用的跳转表。

在Python3.10及更早的版本中，在运行时有一些异常处理。例如，`try`语句被编译为`SETUP_FINALLY`指令，该指令包含指向第一个异常块的指针。当没有引发异常时，将其替换为跳转表可以加快`try`区块的速度。

零成本异常非常适合[请求原谅比请求许可更容易的代码风格](https://realpython.com/python-lbyl-vs-eafp/)，这种风格经常使用大量的`try … except`代码块。

### 异常组