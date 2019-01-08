# Python3 pathlib模块：驯服文件系统

[原文链接](https://realpython.com/python-pathlib/)

## 目录
* [python文件路径处理的问题](#1)
* [创建路径](#2)
* [读写文件](#3)
* [分析路径的成分](#4)
* [移动和删除文件](#5)


你是否在处理文件路径时遇到过麻烦？在python3.4及以上，不会再有这种麻烦了！你不再需要这样的代码：
```python
>>> path.rsplit('\\', maxsplit=1)[0]
```
或者这样的代码：
```python
>>> os.path.isfile(os.path.join(os.path.expanduser('~'), 'realpython.txt'))
```
在这篇指南中你可以学到如何在python中处理文件夹和文件路径。你将会学到读写文件的新姿势，操作路径以及基础文件系统，还可以看到展示和遍历文件的几个例子。使用pathlib模块，上面的两个例子可以重写为如下优雅的，可读的，pythonic的代码：
```python
>>> path.parent
>>> (pathlib.Path.home() / 'realpython.txt').is_file()
```

<h2 id='1'>python文件路径处理的问题</h2>

与文件系统打交道很重要。最基本的是读写一个文件，有时会有更复杂的任务。你可能要按照指定格式展示某目录的全部文件，找出给定文件的父目录，或者创建一个指定的文件。

传统上，python使用正则字符串来表示文件路径，使用[os.path](https://docs.python.org/3/library/os.path.html)标准库来支持，这已经足够了，但是有一点小麻烦（接下来的例子你会看到）。然而自从[路径不是字符串](https://snarky.ca/why-pathlib-path-doesn-t-inherit-from-str/)之后，这些重要的函数被拆分到各个标准库当中，例如[os](https://docs.python.org/3/library/os.html)，[glob](https://docs.python.org/3/library/glob.html)以及[shutil](https://docs.python.org/3/library/shutil.html)。下面的例子需要import三个库才能把全部的txt文件移动到指定目录：
```python
import glob
import os
import shutil

for file_name in glob.glob('*.txt'):
    new_path = os.path.join('archive', file_name)
    shutil.move(file_name, new_path)
```
你硬是要用正则做的话也是可行的，但通常来说是个bad hack。举个例子，你应该用os.path.join()代替使用+来拼接字符串，使用这个函数可以正确的处理不同操作系统中的路径问题。win下路径用\而mac和linux用/分隔路径。这个不同可能会导致难以调试的错误，正如第一个例子只能在win下使用。

从python3.4([PEP 428](https://www.python.org/dev/peps/pep-0428/))开始，python引入pathlib这个库来处理这些问题。它将必要的功能集中起来，通过在path对象上使用属性和方法来实现。

在这之前其他包仍然使用字符串的文件路径，但是到了python3.6之后，pathlib被加入标准库还添加了[文件系统路径协议](https://www.python.org/dev/peps/pep-0519/)。如果你在遗留项目上卡住了，这里还有一个[支持python2的版本](https://github.com/mcmtroffaes/pathlib2)。

Time for action：让我们看看pathlib怎么在实践中应用。

<h2 id='2'>创建路径</h2>

你需要知道的全部就是pathlib.Path这个类。有很多种方法来创建路径。首先这里有两个类方法.cwd()（当前工作路径）和.home()（你的用户home路径）：
```python
>>> import pathlib
>>> pathlib.Path.cwd()
PosixPath('/home/gahjelle/realpython/')
```
注意：在这个指南中，我们假设已经import了pathlib库，不会在代码中再写import语句。如果你主要使用Path类，你也可以from pathlib import Path来代替pathlib.Path。

可以通过字符串来创建一个路径：
```python
>>> pathlib.Path(r'C:\Users\gahjelle\realpython\file.txt')
WindowsPath('C:/Users/gahjelle/realpython/file.txt')
```

处理win路径的小提示：在win中，路径被反斜杠\分隔。然而在很多环境里反斜杠\同时也是转义字符串。为了避免这个问题，请使用原生字符串(raw string)来表示路径。这需要在字符串之前加上r，在原生字符串中\就是代表\：r'C:\Users'。

另一种构建路径的方式是使用特殊操作符/把每部分连起来。这个操作符独立于操作系统提供的分隔路径操作符：
```python
>>> pathlib.Path.home() / 'python' / 'scripts' / 'test.py'
PosixPath('/home/gahjelle/python/scripts/test.py')
```

如果你不喜欢/操作符，你也可以使用.joinpath()方法：
```python
>>> pathlib.Path.home().joinpath('python', 'scripts', 'test.py')
PosixPath('/home/gahjelle/python/scripts/test.py')
```

注意在之前的例子中，pathlib.Path表示为WindowsPath或者PosixPath。这个类的表现取决于目前的操作系统。（WindowsPath的例子运行在win上，PosixPath的例子运行在mac或linux上。）更多信息请参考[操作系统区别](https://realpython.com/python-pathlib/#operating-system-differences)这一章。

<h2 id='3'>读写文件</h2>

传统上python读写文件使用内置函数open()。open()函数可以直接传入一个Path对象进去。下面的例子在一个Markdown文件中找到所有headers并打印他们：
```python
path = pathlib.Path.cwd() / 'test.md'
with open(path, mode='r') as fid:
    headers = [line.strip() for line in fid if line.startswith('#')]
print('\n'.join(headers))
```
一个等效替代，在Path对象上调用.open()：
```
with path.open(mode='r') as fid:
    ...
```
实际上，Path.open()在幕后调用内置函数open()，选择哪一种取决于你的品位。

pathlib库提供了很多方便的方法用来简单地读写文件：

* .read_Text()：用文本模式打开路径然后返回一个字符串。
* .read_bytes()：用二进制模式打开路径然后返回一个二进制字符串。
* .wirte_text()：打开路径并写入字符串数据。
* .write_bytes()：打开路径并写入二进制数据。

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
注意当比较路径时，比较的是它们的表示形式。在上面的例子中path.parent不等于pathlib.Path.cwd()，因为path.parent是' . '，pathlib.Path.cwd()是'/home/gahjelle/realpython/'。

<h2 id='4'>分析路径的成分</h2>

路径的不同部分可以被属性轻松获取。基础的例子包括：

* .name：不包含任何路径的文件名
* .parent：文件所在的文件夹路径，如果路径是文件夹则是文件夹的父路径
* .stem：去掉后缀的文件名
* .suffix：文件的扩展名
* .anchor：路径的盘符

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
注意.parent返回了一个新的Path对象，其他属性返回的是字符串。这意味着.parent可以被链接或者使用/来创建更复杂的新路径：
```python
>>> path.parent.parent / ('new' + path.suffix)
PosixPath('/home/gahjelle/new.md')
```
这个[Pathlib备忘录](https://github.com/chris1610/pbpython/blob/master/extras/Pathlib-Cheatsheet.pdf)记载了其他的属性和方法的用法。

<h2 id='5'>启动和删除文件</h2>

