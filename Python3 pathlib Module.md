# Python3 pathlib模块：驯服文件系统

[原文链接](https://realpython.com/python-pathlib/)

## 目录
* [python文件路径处理的问题](#1)
* [创建路径](#2)
* [读写文件](#3)
* [分析路径的成分](#4)
* [移动和删除文件](#5)
* [示例](#6)
* [Path作为特有对象](#7)
* [总结](#8)

你是否在处理文件路径时遇到过麻烦？在python3.4及以上，不会再有这种麻烦了！你不再需要这样的代码：
```python
>>> path.rsplit('\\', maxsplit=1)[0]
```
或者这样的代码：
```python
>>> os.path.isfile(os.path.join(os.path.expanduser('~'), 'realpython.txt'))
```
在这篇指南中你可以学到如何在python中处理文件夹和文件路径。你将会学到读写文件的新姿势，操作路径以及基础文件系统，还可以看到展示和遍历文件的几个例子。使用`pathlib`模块，上面的两个例子可以重写为如下优雅的，可读的，pythonic的代码：
```python
>>> path.parent
>>> (pathlib.Path.home() / 'realpython.txt').is_file()
```

<h2 id='1'>python文件路径处理的问题</h2>

与文件系统打交道很重要。最基本的是读写一个文件，有时会有更复杂的任务。你可能要按照指定格式展示某目录的全部文件，找出给定文件的父目录，或者创建一个指定的文件。

传统上，python使用正则字符串来表示文件路径，使用[`os.path`](https://docs.python.org/3/library/os.path.html)标准库来支持，这已经足够了，但是有一点小麻烦（接下来的例子你会看到）。然而自从[路径不是字符串](https://snarky.ca/why-pathlib-path-doesn-t-inherit-from-str/)之后，这些重要的函数被拆分到各个标准库当中，例如[`os`](https://docs.python.org/3/library/os.html)，[`glob`](https://docs.python.org/3/library/glob.html)以及[`shutil`](https://docs.python.org/3/library/shutil.html)。下面的例子需要import三个库才能把全部的txt文件移动到指定目录：
```python
import glob
import os
import shutil

for file_name in glob.glob('*.txt'):
    new_path = os.path.join('archive', file_name)
    shutil.move(file_name, new_path)
```
你硬是要用正则做的话也是可行的，但通常来说是个bad hack。举个例子，你应该用`os.path.join()`代替使用+来拼接字符串，使用这个函数可以正确的处理不同操作系统中的路径问题。win下路径用\而mac和linux用/分隔路径。这个不同可能会导致难以调试的错误，正如第一个例子只能在win下使用。

从python3.4([PEP 428](https://www.python.org/dev/peps/pep-0428/))开始，python引入pathlib这个库来处理这些问题。它将必要的功能集中起来，通过在path对象上使用属性和方法来实现。

在这之前其他包仍然使用字符串的文件路径，但是到了python3.6之后，pathlib被加入标准库还添加了[文件系统路径协议](https://www.python.org/dev/peps/pep-0519/)。如果你在遗留项目上卡住了，这里还有一个[支持python2的版本](https://github.com/mcmtroffaes/pathlib2)。

Time for action：让我们看看pathlib怎么在实践中应用。

<h2 id='2'>创建路径</h2>

你需要知道的全部就是`pathlib.Path`这个类。有很多种方法来创建路径。首先这里有两个[类方法](https://realpython.com/instance-class-and-static-methods-demystified/)`.cwd()`（当前工作路径）和`.home()`（你的用户home路径）：
```python
>>> import pathlib
>>> pathlib.Path.cwd()
PosixPath('/home/gahjelle/realpython/')
```
注意：在这个指南中，我们假设已经`import pathlib`库，之后的介绍中不会再写`import`语句。如果你主要使用`Path`类，你也可以`from pathlib import Path`来代替`pathlib.Path`。

可以通过字符串来创建一个路径：
```python
>>> pathlib.Path(r'C:\Users\gahjelle\realpython\file.txt')
WindowsPath('C:/Users/gahjelle/realpython/file.txt')
```

处理win路径的小提示：在win中，路径被反斜杠\分隔。然而在很多环境里反斜杠\同时也是转义字符串。为了避免这个问题，请使用原生字符串(raw string)来表示路径。这需要在字符串之前加上r，在原生字符串中\就是代表\：`r'C:\Users'`。

另一种构建路径的方式是使用特殊操作符/把每部分连起来。这个操作符独立于操作系统提供的分隔路径操作符：
```python
>>> pathlib.Path.home() / 'python' / 'scripts' / 'test.py'
PosixPath('/home/gahjelle/python/scripts/test.py')
```

如果你不喜欢/操作符，你也可以使用`.joinpath()`方法：
```python
>>> pathlib.Path.home().joinpath('python', 'scripts', 'test.py')
PosixPath('/home/gahjelle/python/scripts/test.py')
```

注意在之前的例子中，`pathlib.Path`表示为`WindowsPath`或者`PosixPath`。这个类的表现取决于目前的操作系统。（`WindowsPath`的例子运行在win上，`PosixPath`的例子运行在mac或linux上。）更多信息请参考[操作系统区别](https://realpython.com/python-pathlib/#operating-system-differences)这一章。

<h2 id='3'>读写文件</h2>

传统上python读写文件使用内置函数`open()`。`open()`函数可以直接传入一个`Path`对象进去。下面的例子在一个Markdown文件中找到所有headers并打印他们：
```python
path = pathlib.Path.cwd() / 'test.md'
with open(path, mode='r') as fid:
    headers = [line.strip() for line in fid if line.startswith('#')]
print('\n'.join(headers))
```
一个等效替代，在Path对象上调用`.open()`：
```
with path.open(mode='r') as fid:
    ...
```
实际上，`Path.open()`在幕后调用内置函数`open()`，选择哪一种取决于你的品位。

pathlib库提供了很多方便的方法用来简单地读写文件：

* `.read_Text()`：用文本模式打开路径然后返回一个字符串。
* `.read_bytes()`：用二进制模式打开路径然后返回一个二进制字符串。
* `.wirte_text()`：打开路径并写入字符串数据。
* `.write_bytes()`：打开路径并写入二进制数据。

每种方法都处理了开关文件的琐碎细节：
```python
>>> path = pathlib.Path.cwd() / 'test.md'
>>> path.read_text()
<the contents of the test.md-file>
```
也可以通过文件名来指定路径，这个例子中直接选取的当前工作目录。下面的例子和上一个例子等效：
```python
>>> pathlib.Path('test.md').read_text()
<the contents of the test.md-file>
```
.resolve()方法会打印全路径。下面我们确认一下当前工作目录：
```python
>>> path = pathlib.Path('test.md')
>>> path.resolve()
PosixPath('/home/gahjelle/realpython/test.md')
>>> path.resolve().parent == pathlib.Path.cwd()
True
```
注意当比较路径时，比较的是它们的表示形式。在上面的例子中`path.parent`不等于`pathlib.Path.cwd()`，因为`path.parent`是' . '，`pathlib.Path.cwd()`是'/home/gahjelle/realpython/'。

<h2 id='4'>分析路径的成分</h2>

路径的不同部分可以被属性轻松获取。基础的例子包括：

* `.name`：不包含任何路径的文件名
* `.parent`：文件所在的文件夹路径，如果路径是文件夹则是文件夹的父路径
* `.stem`：去掉后缀的文件名
* `.suffix`：文件的扩展名
* `.anchor`：路径的盘符

下面是一些实例：
```python
>>> path
PosixPath('/home/gahjelle/realpython/test.md')
>>> path.name
'test.md'
>>> path.stem
'test'
>>> path.suffix
'.md'
>>> path.parent
PosixPath('/home/gahjelle/realpython')
>>> path.parent.parent
PosixPath('/home/gahjelle')
>>> path.anchor
'/'
```
注意`.parent`返回了一个新的`Path`对象，其他属性返回的是字符串。这意味着`.parent`可以被链接或者使用/来创建更复杂的新路径：
```python
>>> path.parent.parent / ('new' + path.suffix)
PosixPath('/home/gahjelle/new.md')
```
这个[Pathlib备忘录](https://github.com/chris1610/pbpython/blob/master/extras/Pathlib-Cheatsheet.pdf)记载了其他的属性和方法的用法。

<h2 id='5'>移动和删除文件</h2>

`pathlib`也可以让你进行基础文件操作，例如移动，更新和删除文件。这些的重点是，这些方法不会提供一个警告来等待你确认这些操作，一个不慎操作就可能`rm -rf`了。请谨慎使用这些方法。

使用`.replace()`方法移动文件。注意如果目标路径存在同名文件，`.replace()`方法会覆盖那个同名文件。不幸的是`pathlib`不支持安全的移动文件。为了避免文件覆盖，最简单的方法是在移动之前测试一下目标是否存在同名文件：
```python
if not destination.exists():
    source.replace(destination)
```
然而这样做会引发一个潜在的竞争条件。其他的进程可能会在执行`if`陈述和`.replace()`方法之间新增一个文件到目标路径。如果这是个问题的话，下面是一种安全的操作方式，[独占创建](https://docs.python.org/3/library/functions.html#open)并显式地复制源数据：
```python
with destination.open(mode='xb') as fid:
    fid.write(source.read_bytes())
```
如果目标路径已经存在，上述代码会抛出一个`FileExistsError`异常。技术上来说这个操作复制了一个文件。为了表现为移动文件，只要在复制之后把源文件删掉（见下文）。确保这个过程中不会抛出任何异常。

当你重命名文件时，较好的方法是`.with_name()`和`.with_suffix()`。两种方法都会返回改过名字或后缀的源路径。
比如：
```python
>>> path
PosixPath('/home/gahjelle/realpython/test001.txt')
>>> path.with_suffix('.py')
PosixPath('/home/gahjelle/realpython/test001.py')
>>> path.replace(path.with_suffix('.py'))
```
目录和文件可以使用`.rmdir()`和`.unlink()`方法删除。（再次强调，谨慎操作！）

<h2 id='6'>示例</h2>

在这章中你会看到使用`pathlib`应对一些简单的需求。

<h3>计算文件数量</h3>

有很多种方法来列出文件，最简单的是`.iterdir()`方法，它迭代给定目录的全部文件。下面的例子结合了`.iterdir()`和`collections.Counter`类来计算当前目录下各种类型的文件数量：
```python
>>> import collections
>>> collections.Counter(p.suffix for p in pathlib.Path.cwd().iterdir())
Counter({'.md': 2, '.txt': 4, '.pdf': 2, '.py': 1})
```
可以使用`.glob()`和`.rglob()`（递归版的`.glob()`）方法来更灵活的列出文件。举个例子，`pathlib.Path.cwd().glob('*.txt')`会返回当前目录下后缀为`.txt`的全部文件。下面的例子只计算文件类型以`p`开始的文件数量：
```python
>>> import collections
>>> collections.Counter(p.suffix for p in pathlib.Path.cwd().glob('*.p*'))
Counter({'.pdf': 2, '.py': 1})
```

<h3>列出文件树</h3>

下面的例子我们定了一个叫`tree()`的函数，它会打印一个在给定目录下文件层次结构的可视化树结构。这里我们也要列出子目录，所以我们使用`.rglob()`方法。
```python
def tree(directory):
    print(f'+ {directory}')
    for path in sorted(directory.rglob('*')):
        depth = len(path.relative_to(directory).parts)
        spacer = '    ' * depth
        print(f'{spacer}+ {path.name}')
```
注意我们需要知道当前到根路径的距离。为了实现这个，我们使用`.relative_to()`表示现在路径到根路径的相对距离。然后我们计算中间有几层目录（使用`.parts`属性）。运行它，这个函数会打印一个树结构：
```python
>>> tree(pathlib.Path.cwd())
+ /home/gahjelle/realpython
    + directory_1
        + file_a.md
    + directory_2
        + file_a.md
        + file_b.pdf
        + file_c.py
    + file_1.txt
    + file_2.txt
```
注意：[f-strings](https://realpython.com/python-f-strings/)只能用在Python3.6或更高版本。在老版本的Python中这个表达式`f'{spacer}+ {path.name}'`可以写为`'{0}+ {1}'.format(spacer, path.name)`。

<h3>找出最后修改的文件</h3>

`.iterdir()`，`.glob()`和`.rglob()`方法非常适合生成器表达式和列表推导。为了找出一个目录下最后修改的文件，你能使用`.stat()`方法来获取文件的更多信息。举个例子，`.stat().st_mtime`给出文件的最后修改时间：
```python
>>> from datetime import datetime
>>> time, file_path = max((f.stat().st_mtime, f) for f in directory.iterdir())
>>> print(datetime.fromtimestamp(time), file_path)
2018-03-23 19:23:56.977817 /home/gahjelle/realpython/test001.txt
```
你可以通过类似的表达式读取最后修改的文件内容：
```python
>>> max((f.stat().st_mtime, f) for f in directory.iterdir())[1].read_text()
<the contents of the last modified file in directory>
```
`.stat().st_mtime`属性提供的是从1970年1月1日开始到现在的秒时间截。你需要用`datetime.fromtimestamp`，`time.localtime`或`time.ctime`转换为对你有用的东西。

<h3>创建唯一的文件名</h3>

最后一个例子我们来演示基于模板构造唯一的文件名。首先指定文件名的参数，在这里是数字。第二步连接目录和文件名判断这个路径是否存在。如果已经存在，计数就加1然后继续尝试：
```python
def unique_path(directory, name_pattern):
    counter = 0
    while True:
        counter += 1
        path = directory / name_pattern.format(counter)
        if not path.exists():
            return path

path = unique_path(pathlib.Path.cwd(), 'test{:03d}.txt')
```
如果目录中已经存在`test001.txt`和`test00.txt`，上述代码会设置`path`为`test003.txt`。

<h3>操作系统的区别</h3>

之前我们注意到当我们实例化`pathlib.Path`时，会返回`WindowsPath`或`PosixPath`对象。这个对象取决于你的操作系统。这个特性使得编写跨平台的代码更简单。你也可以直接使用`WindowsPath`或`PosixPath`，但是这样你会把你的代码局限于某个平台，并没有任何好处。同时和你的操作系统冲突的类不允许被使用：
```python
>>> pathlib.WindowsPath('test.md')
NotImplementedError: cannot instantiate 'WindowsPath' on your system
```
有时你需要表示一个路径但是并不需要访问底层文件系统（这种情况下在一个非win系统上表示一个win路径是有意义的，反之亦然）。这个操作可以通过`PurePath`对象完成。这个对象支持我们在[分析路径的成分](#4)中讨论的操作但是没有方法来访问文件系统：
```python
>>> path = pathlib.PureWindowsPath(r'C:\Users\gahjelle\realpython\file.txt')
>>> path.name
'file.txt'
>>> path.parent
PureWindowsPath('C:/Users/gahjelle/realpython')
>>> path.exists()
AttributeError: 'PureWindowsPath' object has no attribute 'exists'
```
你可以在任何系统上直接实例化`PureWindowsPath`或`PurePosixPath`。实例化`PurePath`会返回一个基于你操作系统的对象。

<h2 id='7'>Path作为特有对象</h2>

在[介绍](#1)中简短的提到path不是字符串，`pathlib`背后的动机之一就是使用特有对象来表示文件系统。事实上`pathlib`[官方文档](https://docs.python.org/3/library/pathlib.html)的标题已经指明了 *pathlib* ——面向对象文件系统路径。[面向对象的实现](https://realpython.com/python3-object-oriented-programming/)在上面的例子中已经很明显了（尤其是你把它和`os.path`作比较的时候）。不过现在让我来给你点其他花絮。

与你使用的操作系统无关，路径以Posix格式表示，并使用正斜杠作为路径分隔符。在win上，你会看到这样的东西:
```python
>>> pathlib.Path(r'C:\Users\gahjelle\realpython\file.txt')
WindowsPath('C:/Users/gahjelle/realpython/file.txt')
```

但是当路径转换为字符串时，它会使用原生平台格式，在win上会使用反斜杠：
```python
>>> str(pathlib.Path(r'C:\Users\gahjelle\realpython\file.txt'))
'C:\\Users\\gahjelle\\realpython\\file.txt'
```
这尤其有用，当你使用一个库且这个库不知道如何处理`pathlib.Path`对象的时候。这是一个在Python3.6版本以下时的大问题。举个例子，在Python3.5中，[`configparser`标准库](https://docs.python.org/3/library/configparser.html)只能使用字符串路径来读取文件。处理这种情况需要将`path`显式转为字符串：
```python
>>> from configparser import ConfigParser
>>> path = pathlib.Path('config.txt')
>>> cfg = ConfigParser()
>>> cfg.read(path)                     # Error on Python < 3.6
TypeError: 'PosixPath' object is not iterable
>>> cfg.read(str(path))                # Works on Python >= 3.4
['config.txt']
```
在Python3.6以上推荐使用`os.fspath()`代替`str()`如果你需要将`path`转为字符串。这么做更安全一些，如果你尝试转换一个[不像路径](https://docs.python.org/3/library/os.html#os.PathLike)的对象，它会抛出一个异常。

可能`pathlib`库最不寻常的部分时使用`/`操作符。我们往底层挖一挖，看看这是如何实现的。这是一个操作符重载的例子：操作符的行为会根据上下文来改变。你之前肯定见过，想一想`+`对于字符串或者数字表现出怎样的特性。Python通过使用双下划线方法实现操作符重载。

`/`操作符通过`.__truediv__()`方法定义。如果你看一下[`pathlib`的源代码](https://github.com/python/cpython/blob/master/Lib/pathlib.py)，你会看到这样的东西：
```python
class PurePath(object):

    def __truediv__(self, key):
        return self._make_child((key,))
```

<h2 id='8'>总结</h2>

自从Python3.4开始`pathlib`被引入标准库。通过`pathlib`库，文件路径可以被表示为特有的`Path`对象代替之前直接使用字符串。这些对象使得处理文件路径的代码：

* 更加易读，尤其因为使用`/`连接路径
* 更加强大，很多必要的方法和属性可以直接作用在对象上
* 跨操作系统一致性，不同操作系统的特性被隐藏在`Path`对象之下

通过这篇指南，你已经看到如何创建`Path`对象，读写文件，操作路径和底层文件系统，同时还有迭代多文件路径的例子。