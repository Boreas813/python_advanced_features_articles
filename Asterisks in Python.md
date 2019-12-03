# Pyhton中的*号：他们是什么以及如何使用它们
[原文链接](https://treyhunner.com/2018/10/asterisks-in-python-what-they-are-and-how-to-use-them/)

在python中很多地方可以看到*和**。这两个操作符可能会显得比较神秘，接下来讨论这些操作符和他们的使用方法。

操作符\*和\*\*在过去几年发展很快，我只讨论现在你可以在目前python版本中使用的方式。如果你学过python2中的\*和\**对的功能我也推荐你略读一下这篇文章，因为python3中添加了很多新功能。

如果你是python新手，并且不熟悉关键字参数(keyword arguments)，我推荐你先阅读我的文章[python中的关键字参数](https://treyhunner.com/2018/04/keyword-arguments-in-python/)

<!-- TOC -->

- [Pyhton中的*号：他们是什么以及如何使用它们](#pyhton中的号他们是什么以及如何使用它们)
    - [我们不是在谈论这个](#我们不是在谈论这个)
    - [所以我们到底在谈论什么](#所以我们到底在谈论什么)
    - [*用来进行函数调用的拆包](#用来进行函数调用的拆包)
    - [*用来打包传递到函数的参数](#用来打包传递到函数的参数)
    - [位置参数和强制关键字参数](#位置参数和强制关键字参数)
    - [没有位置型参数的强制关键性参数](#没有位置型参数的强制关键性参数)
    - [\*在元组拆包当中](#\在元组拆包当中)
    - [\*在list literals中](#\在list-literals中)
    - [\*\*操作符在字典literals中](#\\操作符在字典literals中)

<!-- /TOC -->

## 我们不是在谈论这个

当我在这篇文章中提到\*和\*\*时，我指的是前缀操作符而不是中缀操作符。

所以我不是在说乘法和指数操作：

```python
>>> 2 * 5
>>> 2 ** 5
```

## 所以我们到底在谈论什么

我们在谈论当\*作为前缀操作符时，在一个变量之前使用\*和\*\*。举个例子：
```python
>>> numbers = [2, 1, 3, 4, 7]
>>> more_numbers = [*numbers, 11, 18]
>>> print(*more_numbers, sep=', ')
2, 1, 3, 4, 7, 11, 18
```

代码展示了\*的两种用法，\*\*的用法暂没有提到。

这些能力包含：
1. 使用\*和\*\*向函数传递参数
2. 使用\*和\*\*捕获函数接收的参数
3. 使用\*接收强制关键性参数(keyword-only arguments)
4. 使用\*在元组拆包(tuple unpacking)时取值
5. 使用\*将可迭代对象拆包为元组/列表
6. 使用\*\*将字典拆包进另一个字典

及时你已经很熟悉\*和\*\*的打开方式，我还是建议你看一下下文中每个代码块确保他们真的是你熟悉的东西。python核心开发者在过去几年给\*和\*\*添加了很多新新功能。

## *用来进行函数调用的拆包
当函数调用时(calling a function)， *操作符能被用来将可迭代对象拆包成参数传进函数：
```python
>>> fruits = ['lemon', 'pear', 'watermelon', 'tomato']
>>> print(fruits[0], fruits[1], fruits[2], fruits[3])
lemon pear watermelon tomato
>>> print(*fruits)
lemon pear watermelon tomato
```
print(*fruits)那行将list中的全部项当做是分开的参数传递到了print函数中，我们甚至不需要知道有多少个参数被传了进去(译者：可以方便传递不稳定的参数？)。

这里的\*操作符不仅仅是语法糖。如果不用\*的话，将可迭代对象作为分开的参数传递是不可能的，除非list是一个固定的长度。

这有另一个例子：
```python
def transpose_list(list_of_lists):
    return [
        list(row)
        for row in zip(*list_of_lists)
    ]
```
这里我们接收一个列表然后返回一个“转换后”的list的list。
```python
>>> transpose_list([1, 4, 7], [2, 5, 8], [3, 6, 9])
[[1, 2, 3], [4, 5, 6], [7, 8, 9]]
```
\*\*操作符做相同的事情，只不过使用的是关键字参数。\*\*操作符允许我们在函数调用时将一个字典拆包成关键字参数传递。
```python
>>> date_info = {'year': "2020", 'month': "01", 'day': "01"}
>>> filename = "{year}-{month}-{day}.txt".format(**date_info)
>>> filename
'2020-01-01.txt'
```
就我的经验来说，使用\*\*拆包关键字参数并不是很常见。我见过最常用的地方是继承的时候：调用super()时总会包括\*和\*\*。

\*和\*\*在函数调用时都可以被多次使用，在python3.5之后。

有时多次使用\*会很顺手：
```python
>>> fruits = ['lemon', 'pear', 'watermelon', 'tomato']
>>> numbers = [2, 1, 3, 4, 7]
>>> print(*numbers, *fruits)
2 1 3 4 7 lemon pear watermelon tomato
```

多次使用\*\*看起来差不多：
```python
>>> date_info = {'year': "2020", 'month': "01", 'day': "01"}
>>> track_info = {'artist': "Beethoven", 'title': 'Symphony No 5'}
>>> filename = "{year}-{month}-{day}-{artist}-{title}.txt".format(
...     **date_info,
...     **track_info,
... )
>>> filename
'2020-01-01-Beethoven-Symphony No 5.txt'
```
当你多次使用\*\*的时候要小心一点。python的函数不允许同名参数多次使用，所以字典里的key必须要不同，否则会抛出异常。
## *用来打包传递到函数的参数
当定义一个函数时， \*操作符可以捕获到数量不限的传进来的位置参数(positional arguments)。这些参数被捉到一个元组里。
```python
from random import randint

def roll(*dice):
    return sum(randint(1, die) for die in dice)
```
这个函数接收任意数量参数：
```python
>>> roll(20)
18
>>> roll(6, 6)
9
>>> roll(6, 6, 6)
8
```
python的print和zip函数接收任意数量的位置参数。用\*打包任意位置参数可以让我们的函数也像print和zip一样接收任意数量的参数。

\*\*操作符同样：我们可以在定义函数时用\*\*捕捉关键字参数到一个字典：
```python
def tag(tag_name, **attributes):
    attribute_list = [
        f'{name}="{value}"'
        for name, value in attributes.items()
    ]
    return f"<{tag_name} {' '.join(attribute_list)}>"
```
\*\*会捕获到我们传进函数的全部关键字参数，转为一个字典然后被attributes这个参数引用。
```python
>>> tag('a', href="http://treyhunner.com")
'<a href="http://treyhunner.com">'
>>> tag('img', height=20, width=40, src="face.jpg")
'<img height="20" width="40" src="face.jpg">'
```
## 位置参数和强制关键字参数

在python3中我们有一个特殊的语法在函数中接收强制关键字参数。强制关键字参数只能使用关键字语法(keyword syntax)，意味着不可以用位置指定。

为了接收强制关键字参数， 我们可以把\*放在其参数名前面：
```python
def get_multiple(*keys, dictionary, default=None):
    return [
        dictionary.get(key, default)
        for key in keys
    ]
```
上面的函数可以这样使用：
```python
>>> fruits = {'lemon': 'yellow', 'orange': 'orange', 'tomato': 'red'}
>>> get_multiple('lemon', 'tomato', 'squash', dictionary=fruits, default='unknown')
['yellow', 'red', 'unknown']
```
参数dictionary和default在*keys之后，这意味着他们必须被当做关键字参数指定。如果你尝试使用位置参数会得到一个error：
```python
>>> fruits = {'lemon': 'yellow', 'orange': 'orange', 'tomato': 'red'}
>>> get_multiple('lemon', 'tomato', 'squash', fruits, 'unknown')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: get_multiple() missing 1 required keyword-only argument: 'dictionary'
```
该行为在python的PEP3102当中介绍了
## 没有位置型参数的强制关键性参数
强制关键字参数这个特性确实很酷，但是如果你想在没有捕捉不确定数量的位置型参数时请求强制关键词参数应该怎么做？

python允许这样一种奇怪的写法：
```python
def with_previous(iterable, *, fillvalue=None):
    """Yield each iterable item along with the item before it."""
    previous = fillvalue
    for item in iterable:
        yield previous, item
        previous = item
```
该函数接收一个可迭代的参数，这个参数可以是位置型参数，或者通过它的名字和一个强制关键词参数fillvalue。这意味着我们可以像这样调用with_previous：
```python
>>> list(with_previous([2, 1, 3], fillvalue=0))
[(0, 2), (2, 1), (1, 3)]
```
但不要这样：
```python
>>> list(with_previous([2, 1, 3], 0))
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: with_previous() takes 1 positional argument but 2 were given
```
该函数接收两个参数，其中fillvalue必须要作为关键词参数。

我经常使用关键词参数同时捕获任意数量的位置型参数，但是有时我用\*强迫一个参数必须使用位置型指定。

python的内置函数sorted就是这样实现的。如果你看一下sorted的帮助：
```python
>>> help(sorted)
Help on built-in function sorted in module builtins:

sorted(iterable, /, *, key=None, reverse=False)
    Return a new list containing all items from the iterable in ascending order.

    A custom key function can be supplied to customize the sort order, and the
    reverse flag can be set to request the result in descending order.
```
在说明文档里就有\*的奇怪语法(\*-on-its-own)。
## \*在元组拆包当中
python3同样加入了一些和\*定义函数和\*调用函数有关联的新用法。

\*操作符现在可以用于元组拆包：
```python
>>> fruits = ['lemon', 'pear', 'watermelon', 'tomato']
>>> first, second, *remaining = fruits
>>> remaining
['watermelon', 'tomato']
>>> first, *remaining = fruits
>>> remaining
['pear', 'watermelon', 'tomato']
>>> first, *middle, last = fruits
>>> middle
['pear', 'watermelon']
```
如果你对“在我的代码里怎么用这些东西”比较困惑，去看一眼我的文章[python元组拆包](https://treyhunner.com/2018/03/tuple-unpacking-improves-python-code-readability/)。文章中我展示了如何使用\*作为序列切片的替代。

通常当我教\*的时候，我告诉你只能在一个单一的多个赋值调用中使用一个\*表达式。这在技术上是不正确的因为在嵌套拆包(nested unpacking)中可以使用两个(我在元组拆包的文章当中讲过嵌套拆包)：
```python
>>> fruits = ['lemon', 'pear', 'watermelon', 'tomato']
>>> ((first_letter, *remaining), *other_fruits) = fruits
>>> remaining
['e', 'm', 'o', 'n']
>>> other_fruits
['pear', 'watermelon', 'tomato']
```
我从没见过好的使用方式并且我也不推荐你使用，因为看起来含义模糊不清。
PEP在python3.0添加的这个，在PEP3132可以找到，不是很长。
## \*在list literals中
python3.5通过[PEP 448](https://www.python.org/dev/peps/pep-0448/)介绍了成吨的关于\*的新特性。其中之一最大的新特性就是使用\*将可迭代对象转储为一个新的list。

你有一个下面的函数，接收一个序列然后返回一个列表包含序列和反转之后的序列：
```python
def palindromify(sequence):
    return list(sequence) + list(reversed(sequence))
```
这个函数需要多次将序列转为list以便后面的连接操作。在python3.5中，我们可以这样代替：
```python
def palindromify(sequence):
    return [*sequence, *reversed(sequence)]
```
这行代码移除了一些无用的强制list转换所以我们的代码更高效更加可读。

这有另一个例子：
```python
def rotate_first_item(sequence):
    return [*sequence[1:], sequence[0]]
```
这个函数返回一个新的list，新list当中将给定list的第一项移动到新list末尾。

使用\*来链接不同类型的可迭代对象是非常“Great”的。\*操作符可以操作任何可迭代对象，用+号的话只能讲同类型的序列连起来。

这个操作不仅仅可以创建list，也可以将可跌倒对象转储为元组或set：
```python
>>> fruits = ['lemon', 'pear', 'watermelon', 'tomato']
>>> (*fruits[1:], fruits[0])
('pear', 'watermelon', 'tomato', 'lemon')
>>> uppercase_fruits = (f.upper() for f in fruits)
>>> {*fruits, *uppercase_fruits}
{'lemon', 'watermelon', 'TOMATO', 'LEMON', 'PEAR', 'WATERMELON', 'tomato', 'pear'}
```
注意一下上面最后一行，将一个list和一个迭代器转储为一个新的set。在使用\*操作符之前这个没有很简单的方法可以一行完成。倒是有一个方法，但是不太容易记忆或查询：
```python
>>> set().union(fruits, uppercase_fruits)
{'lemon', 'watermelon', 'TOMATO', 'LEMON', 'PEAR', 'WATERMELON', 'tomato', 'pear'}
```
## \*\*操作符在字典literals中
PEP448同样扩展了\*\*操作符将一个键值对从一个字典转储进另一个字典的能力：
```python
>>> date_info = {'year': "2020", 'month': "01", 'day': "01"}
>>> track_info = {'artist': "Beethoven", 'title': 'Symphony No 5'}
>>> all_info = {**date_info, **track_info}
>>> all_info
{'year': '2020', 'month': '01', 'day': '01', 'artist': 'Beethoven', 'title': 'Symphony No 5'}
```
我写了另一篇文章[python中合并字典的常用方法](https://treyhunner.com/2016/02/how-to-merge-dictionaries-in-python/)

现在的方法可以合并超过两个以上的字典。

举个例子，我们复制一个字典同时添加一个新值进去：
```python
>>> date_info = {'year': '2020', 'month': '01', 'day': '7'}
>>> event_info = {**date_info, 'group': "Python Meetup"}
>>> event_info
{'year': '2020', 'month': '01', 'day': '7', 'group': 'Python Meetup'}

```
或者合并字典的同时重写某个特定的值：
```python
>>> event_info = {'year': '2020', 'month': '01', 'day': '7', 'group': 'Python Meetup'}
>>> new_info = {**event_info, 'day': "14"}
>>> new_info
{'year': '2020', 'month': '01', 'day': '14', 'group': 'Python Meetup'}
```
