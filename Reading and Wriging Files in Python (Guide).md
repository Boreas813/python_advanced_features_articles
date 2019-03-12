# 在python中读写文件（指南）
[原文链接](https://realpython.com/read-write-files-python/#what-is-a-file)

[TOC]

使用python读写文件是一个常见的任务。不管是写一个文本文档，读取一个复杂的服务器日志亦或是分析原始二进制文件，这些情况都需要读写文件。

**在这篇教程中你会学到：**

* 什么构成了文件以及为什么这在python中很重要
* python读写文件基础
* 读写文件的一些基础场景

这篇教程主要面向想要进阶的新手玩家，但是这里也有一些高级开发者可以用到的小tips。

## 文件是什么？

在我们开始使用python操作文件之前，理解一个文件到底是什么以及现代操作系统如何处理它们非常重要。

本质上讲，一个文件是[用来存储数据](https://en.wikipedia.org/wiki/Computer_file)的连续的一组字节。这些数据使用一种特定的格式组织起来，不管是简单的文本文档还是复杂的可执行程序。最后这些字节文件为了更便于电脑处理，转化为二进制 `1` 或 `0`。

大多数现代文件系统上的文件有三个主要组成部分：

1. **Header**:文件的元数据（文件名，大小，类型等等）
2. **Data**:创建者或编辑者写入的文件主体内容
3. **End of file(EOF)**:表明文件结束的特定字符

![](https://files.realpython.com/media/FileFormat.02335d06829d.png)

这些数据的组织格式一般使用扩展名来表示。举个例子，一个有`.gif`扩展名的文件很可能是按照[Graphics Interchange Format](https://en.wikipedia.org/wiki/GIF)格式组织的。[这里](https://en.wikipedia.org/wiki/List_of_filename_extensions)有成百上千的扩展名。对于该教程，你只会涉及处理`.txt`和`.csv`文件。

### 文件路径

当你在操作系统上访问一个文件时，你需要一个文件路径。文件路径是表示文件所在位置的一段字符串。它可以被分解成三个主要部分：

1. **Folder Path**:文件夹所在位置，Unix中使用`/`分割，Windows中使用`\`分割
2. **File Name**:文件的名字
3. **Extension**:文件的扩展名，使用`.`分割

举个例子，假设你有如下的一个文件结构：
```
/
│
├── path/
|   │
│   ├── to/
│   │   └── cats.gif
│   │
│   └── dog_breeds.txt
|
└── animals.csv
```
假设说你想访问`cats.gif`，你当前的目录在 `/`，你需要穿过`path`和`to`文件夹，最终到达`cats.gif`文件。文件夹路径是`path/to/`。文件名字是`cats`。文件扩展名是`.gif`。所以完整路径是`path/to/cats.gif`。

现在假设你当前所在位置是`to`文件夹。你可以只是用文件名+扩展名`cats.gif`来代替绝对路径`path/to/cats.gif`。
```
/
│
├── path/
|   │
|   ├── to/  ← 你当前工作路径(cwd)
|   │   └── cats.gif  ← 访问这个文件
|   │
|   └── dog_breeds.txt
|
└── animals.csv
```
但要是想访问`dog_breeds.txt`呢？如何不使用绝对路径访问呢？你可以使用特殊字符 `..`来移动到上一级目录。所以你可以使用`../dog_breeds.txt`在`to`目录中访问`dog_breeds.txt`：
```
/
│
├── path/  ← Referencing this parent folder
|   │
|   ├── to/  ← Current working directory (cwd)
|   │   └── cats.gif
|   │
|   └── dog_breeds.txt  ← Accessing this file
|
└── animals.csv
```
`..`可以链式使用来返回多级目录之上。举个例子，在`to`文件夹中访问`animals.csv`，你可以使用`../../animals.csv`。

### 行尾结束符号

读写文件时总出现的一个问题是如何表示新一行内容开始或者一行内容的结束。换行符的历史可以追溯到摩斯密码时代，[在传输结束或一行结束之后发送一个特定的符号。](https://en.wikipedia.org/wiki/Prosigns_for_Morse_code#Official_International_Morse_code_procedure_signs)

然后，两个国际组织ISO和ASA推出了[标准化电传打字机。](https://en.wikipedia.org/wiki/Newline#History)ASA标准规定应该使用回车(`CR`或`\r`)和换行(`LF`或`\n`)连起来(`CR+LF`或`\r\n`)。然而ISO标准既允许使用`CR+LF`也允许只使用`LF`换行。

[Windows使用CR+LF](https://unix.stackexchange.com/a/411830)换行，Unix和比较新的Mac只使用`LF`字符。当你在一个和文件源不一样的平台上操作这个文件时就会有一些小问题。这有一个例子。我们假设文件`dog_breedx.txt`是创建在Win平台上：
```
Pug\r\n
Jack Russel Terrier\r\n
English Springer Spaniel\r\n
German Shepherd\r\n
Staffordshire Bull Terrier\r\n
Cavalier King Charles Spaniel\r\n
Golden Retriever\r\n
West Highland White Terrier\r\n
Boxer\r\n
Border Terrier\r\n
```
在Unix设备上的输出会有所不同：
```
Pug\r
\n
Jack Russel Terrier\r
\n
English Springer Spaniel\r
\n
German Shepherd\r
\n
Staffordshire Bull Terrier\r
\n
Cavalier King Charles Spaniel\r
\n
Golden Retriever\r
\n
West Highland White Terrier\r
\n
Boxer\r
\n
Border Terrier\r
\n
```
这会使遍历每行有问题，你应该把这样的情况考虑进去。

### 字符编码

另一个你可能遇到的问题是字节编码。一种编码是从字节数据到人类可读字符的一种转换。这通常是指定一个数值来表示一个字符。两种常见的编码方式是[ASCII](https://www.ascii-code.com/)和[UNICODE](https://unicode.org/)。[ASCII只可以存储128个字符](https://en.wikipedia.org/wiki/ASCII)，[Unicode可容纳1,114,112个字符]。

ASCII其实是Unicode（UTF-8）的一个子集，这意味着ASCII和Unicode共享相同字符的数值。需要注意的是使用错误的字符编码解析文件会导致严重错误。举个例子，加入一个文件使用UTF-8编码创建，然后你尝试使用ASCII解析它，如果文件中有128个值之外的字符，就会抛出一个错误。

## 使用python打开和关闭文件

当你想要操作文件时第一步是打开文件。这个使用[内置函数open()](https://docs.python.org/3/library/functions.html#open)完成。`open()`需要一个叫文件路径的参数。`open()`函数有一个返回值，[文件对象](https://docs.python.org/3/glossary.html#term-file-object)：
```python
file = open('dog_breeds.txt')
```
在你打开一个文件之后，你要学着如何关闭它。
```
警告：你必须永远确保打开的文件在合适的时机关闭。
```
记住！关闭文件永远是你的重大职责！大多数情况下，中止应用或脚本后你的文件最终会被关闭。然而不能保证这个情况何时会发生。这可能会导致未预期的行为比如资源泄露。同时这也是Pythonic的最佳实践，确保你的代码以定义良好的方式运行并减少任何未预期的行为。

当你操作文件时，有两种方法可以确保即使抛出了异常的情况下文件也会被恰当的关闭。第一种方法使用`try-finally`代码块：
```python
reader = open('dog_breeds.txt')
try:
    # Further file processing goes here
finally:
    reader.close()
```
第二种方法是使用`with`陈述：
```python
with open('dog_breeds.txt') as reader:
    # Further file processing goes here
```
`with`语句可以当程序流离开`with`代码块时自动关闭文件，即使代码块中出现了错误。我本人强烈建议你尽可能多的使用`with`，它让代码更干净并且你可以轻松的处理任何意外的错误。

同时你会用到第二个位置型参数，`mode`。这个参数是一个包含多个字符的字符串来表示你希望怎样打开这个文件。默认和最常用的是`r`，这代表通过只读方式打开这个文件：
```python
with open('dog_breeds.txt', 'r') as reader:
    # Further file processing goes here
```
其他的模式参考[在线文档](https://docs.python.org/3/library/functions.html#open)，但是最常用的几个选项如下：
```
Character	Meaning
'r'	        只读模式打开（默认）
'w'	        写入模式打开，覆盖之前的内容
'rb' or 'wb'	二进制模式打开（使用字节读/写）
```
现在我们回头看看文件对象是什么。一个文件对象是：

“an object exposing a file-oriented API (with methods such as read() or write()) to an underlying resource.” ([Source](https://docs.python.org/3/glossary.html#term-file-object))

暴露出面向文件API（使用`read()`或`write()`方法）来访问底层资源的一个对象。

文件对象有三种不同类别：

* Text files            文本文件
* Buffered binary files 缓冲二进制文件
* Raw binary files      原始二进制文件

每个文件类型都在`io`模块中被定义好了。下面是一个关于该部分内容的大纲。

### 文本文件类型

文本文件是经常要接触到的类型。这又几个如何打开文件的例子：
```python
open('abc.txt')

open('abc.txt', 'r')

open('abc.txt', 'w')
```
`open()`会返回一个`TextIOWrapper`文件对象：
```python
>>> file = open('dog_breeds.txt')
>>> type(file)
<class '_io.TextIOWrapper'>
```
这是`open()`返回的默认对象。

### 缓存二进制文件

缓存二进制文件类型用来读写二进制文件。下面是打开这类文件的例子：
```python
open('abc.txt', 'rb')

open('abc.txt', 'wb')
```
这时`open()`返回`BufferReader`或`BufferWriter`文件对象：
```python
>>> file = open('dog_breeds.txt', 'rb')
>>> type(file)
<class '_io.BufferedReader'>
>>> file = open('dog_breeds.txt', 'wb')
>>> type(file)
<class '_io.BufferedWriter'>
```