# python3面向对象编程
[原文地址](https://realpython.com/python3-object-oriented-programming/#what-is-object-oriented-programming-oop)

<!-- TOC -->

- [python3面向对象编程](#python3面向对象编程)
    - [什么是面向对象编程(Object-Oriented Programming, OOP)？](#什么是面向对象编程object-oriented-programming-oop)
    - [python中的类](#python中的类)
    - [python对象(实例)](#python对象实例)
    - [在python中如何定义一个类](#在python中如何定义一个类)
    - [实例属性](#实例属性)
    - [类属性](#类属性)
    - [实例化对象(Instantiating Objects)](#实例化对象instantiating-objects)
    - [刚才发生了什么？？？](#刚才发生了什么)
    - [回顾练习(#1)](#回顾练习1)
    - [实例方法](#实例方法)
    - [修改属性](#修改属性)
    - [python对象继承(Python Object Inheritance)](#python对象继承python-object-inheritance)
    - [例子：小狗公园](#例子小狗公园)
    - [扩展一个父类的功能](#扩展一个父类的功能)
    - [父类 vs. 子类](#父类-vs-子类)
    - [覆盖/重写父类的功能](#覆盖重写父类的功能)
    - [回顾练习(#2)](#回顾练习2)
    - [结尾](#结尾)

<!-- /TOC -->



在这篇文章中你会轻松地学会python面向对象编程(Object-Oriented Programming)的基本概念：
1. python的类
2. 类的实例
3. 定义和运用方法(Methods)
4. 面向对象编程继承
## 什么是面向对象编程(Object-Oriented Programming, OOP)？
面向对象编程，缩写OOP，是一种将属性和行为绑定到独立的类中然后提供一种结构化程序的[编程范式](http://en.wikipedia.org/wiki/Programming_paradigm)

举例来说，“人”可以表示为一个类，有名字，年龄，住址等这些属性，有走步，交谈，呼吸，跑步等行为。或者“电子邮件”也可以是一个类，有收信人列表，主题，正文等属性，有添加附件和发送等行为。

换一种说法，OOP在是实现具体事物的建模，现实世界的物品比如汽车，还有事物之间的关系，例如公司和雇员，学生和老师等等。OOP将现实中的实体抽象成类，这些类有同样的数据也可以执行特定的功能。

还有一种常见的编程范式是*面向过程编程*，这是以函数和代码块的形式按顺序执行一系列步骤以完成一个任务。

关键点在于类是OOP的中心点，不像面向过程编程那样表示数据，在程序主体结构上也可面向过程完全不一样。

提示：python是多范式编程语言，你可以选择最适合解决特定问题的范式，在一个程序中混合使用不同的范式，随着开发的进展从一个范式过渡到另一个范式
## python中的类
首先关注数据，所有事物或对象(object)都是某个类(class)的实例(instance)。

python中的原始数据类型，例如数字，字符串，列表被设计用来表示一些简单的东西。比如某物的成本，一首诗的名字，你最喜欢的颜色。

如果你想表示一些更复杂的东西呢？

举个例子，如果说你想跟踪不同动物的数字。如果你使用一个list，第一个元素可能是动物的名字第二个元素可能代表年龄。

那么你怎么知道每个元素应该对应哪个呢？如果有100种不同的动物呢？你确定每个动物都有一个名字和年龄么？如果想给动物添加额外的属性怎么办？这么做缺少组织性，应该使用类来做。

使用类可以创建包含任意数据的用户自定义数据类型。在动物的例子中我们可以创建一个Animal()类来跟踪动物的年龄或名字等属性。

很重要的是类只提供结构——它是某物应该怎样被定义的蓝图，它不提供任何真实内容。Animal()类可以指定定义一个动物时名字和年龄是必须的，但是它不会真正提供某个动物的名字或年龄。

这样有助于思考类是一个关于某事物应该如何被定义的方法。
## python对象(实例)
类是蓝图，那么实例就是带有真实数据的类的拷贝，字面来讲一个对象属于一个指定的类。实例不再是事物如何被定义的方法了，它是一个真实的动物，就像一个名字叫Roger的八岁的狗。

另一方面来说，一个类就像是一张表格或者调查问卷。它定义了需要提供哪些信息。然后你填好这个表，你就实例化了一个类，实例包含了你需要的数据。

你可以通过一个类拷贝出很多不同的实例，但是如果没了表格的指导你就会迷失，因为不知道都需要什么信息。因此在你创建实例之前，你必须要先定义好一个类。
## 在python中如何定义一个类
在python中定义一个类很简单：
```python
class Dog:
    pass
```
通过class关键词来创建一个类，然后给类起一个名字(使用[驼峰命名法](https://en.wikipedia.org/wiki/Camel_case)，以大写字母开头)

同样，这里我们用了python关键词pass。它让我们可以不抛出异常正常运行代码。

提示：上述代码在python3中是正确的。在python2.x中要这么定义：
```python
# Python 2.x Class Definition:
class Dog(object):
    pass
```
(object)是你要指定需要继承的父类，python3中默认值为object。
## 实例属性
所有的类会创建对象，所有的对象都包含特性叫做属性(attributes)。使用__init__()方法来初始化一个对象的初始属性。这个方法至少要有一个参数就是self，这个参数被类自己引用(本例中为Dog类)。
```python
class Dog:
    # Initializer / Instance Attributes
    def __init__(self, name, age):
        self.name = name
        self.age = age
```
上述例子中每个狗都有一个一个名字和年龄，在你真的开始创建不同的狗时会起作用。记住：类仅仅是定义了狗这个类，并没有指定名字和年龄来创建狗的实例，我们稍后会讲到。

相似的，self这个变量也是类的一个实例。由于每个实例都有不同的属性值，所以我们这样操作属性 Dog\.name = name,而不是self\.name = name。但是每只狗都有不同的名字，我们要给每个实例赋予不同的值。因此我们需要self这个特殊变量，它可以帮我们跟踪每个类的单独实例。

提示：你不需要手动调用__init__()方法；当你创建“Dog”的实例是会被自动调用。

## 类属性
实例属性是不同的实例拥有的属性，类属性是该类的全部实例都拥有的属性——本例中全部实例为*所有的狗*
```python
class Dog:

    # Class Attribute
    species = 'mammal'

    # Initializer / Instance Attributes
    def __init__(self, name, age):
        self.name = name
        self.age = age
```
所以情况就是，每只狗都有一个独有的名字和年龄，所有的狗都是哺乳动物(species = 'mammal'定义了这个)

让我们开始创建一些狗吧...

## 实例化对象(Instantiating Objects)
实例化意思就是创建一个某个类的新的独特的实例。

举个栗子：
```python
>>> class Dog:
...     pass
...
>>> Dog()
<__main__.Dog object at 0x1004ccc50>
>>> Dog()
<__main__.Dog object at 0x1004ccc90>
>>> a = Dog()
>>> b = Dog()
>>> a == b
False
```
开始我们定义了一个新的Dog()类，然后我们创建了两个新的狗，每个都分配成为了不同的对象。创建一个类的实例只需要使用类名加圆括号。然后我们实例化了两个狗分配到a和b变量中去，然后我们比较这两个实例是否相等，答案是否定的。

你感觉一个实例的type是什么呢？
```python
>>> class Dog:
...     pass
...
>>> a = Dog()
>>> type(a)
<class '__main__.Dog'>
```

再看一个稍微复杂点的例子...
```python
class Dog:

    # Class Attribute
    species = 'mammal'

    # Initializer / Instance Attributes
    def __init__(self, name, age):
        self.name = name
        self.age = age


# Instantiate the Dog object
philo = Dog("Philo", 5)
mikey = Dog("Mikey", 6)

# Access the instance attributes
print("{} is {} and {} is {}.".format(
    philo.name, philo.age, mikey.name, mikey.age))

# Is Philo a mammal?
if philo.species == "mammal":
    print("{0} is a {1}!".format(philo.name, philo.species))
```
提示：注意观察我们如何使用“.”符号来访问每个对象的属性。

将上述代码保存为dog_class.py，然后运行程序你会看到：
```
Philo is 5 and Mikey is 6.
Philo is a mammal!
```
## 刚才发生了什么？？？
我们创建了一个Dog()类的实例，然后将实例赋值给philo。我们实例化类的时候传递了两个参数到括号里，"philo"和5，代表狗的名字和年龄。

这些属性会被传递到__init__方法中，当你创建一个新的实例，将名字和年龄赋值到对象的时候__init__会被调用。你可能比较困惑为什么我们没有传递self这个参数。

这就是Python的魔法，当你实例化的时候，python会自动识别self是什么然后将它传递到__init__方法中。

## 回顾练习(#1)
练习：“年龄最大的狗”
使用Dog类实例化三个不同的狗，每只狗有一个不同的年龄。然后写一个函数叫做get_biggest_number()，这个函数接收任意数量的年龄(使用*args)然后返回最大的一个数值。然后输出年龄最大的狗的年龄：
```
The oldest dog is 7 years old.
```
## 实例方法
实例方法定义在一个类的内部一般获取实例的内容。他还可以执行带有对象属性的各种操作。像__init__方法一样第一个参数总是self：
```python
class Dog:

    # Class Attribute
    species = 'mammal'

    # Initializer / Instance Attributes
    def __init__(self, name, age):
        self.name = name
        self.age = age

    # instance method
    def description(self):
        return "{} is {} years old".format(self.name, self.age)

    # instance method
    def speak(self, sound):
        return "{} says {}".format(self.name, sound)

# Instantiate the Dog object
mikey = Dog("Mikey", 6)

# call our instance methods
print(mikey.description())
print(mikey.speak("Gruff Gruff"))
```
将上述代码存为dog_instance_methods.py然后运行：
```
Mikey is 6 years old
Mikey says Gruff Gruff
```
在speak()方法中我们定义了一个行为，每次调用这个方法，就会返回狗的名字和叫声。你还可以给狗赋予其他的行为吗？回到文章最开始看看其他的例子然后开始尝试吧。

## 修改属性
你可以通过一些行为来更改属性值:
```python
>>> class Email:
...     def __init__(self):
...         self.is_sent = False
...     def send_email(self):
...         self.is_sent = True
...
>>> my_email = Email()
>>> my_email.is_sent
False
>>> my_email.send_email()
>>> my_email.is_sent
True
```
这里我添加了一个发送邮件的方法，这个方法会将is_sent的值更新为True.

## python对象继承(Python Object Inheritance)
继承的意思是A类从B类得到B类中的属性和方法。其中A类被叫做子类(child classes)，被继承的类B叫做父类(parent classes)。

请记住子类是覆盖或者扩展父类的功能的。换句话说，子类继承父类全部的属性和行为但是也可以修改这些行为。类的基础类叫做object， 其他的所有类都继承于它。

当你定义一个新的类时，python3隐式的使用object作为父类。所以一下两种定义是相等的：
```python
class Dog(object):
    pass

# In Python 3, this is the same as:

class Dog:
    pass
```
提示：在python2中类分为[新式类和旧式类](https://wiki.python.org/moin/NewClassVsClassicClass)。 我不会深入细节，但是通常你会选择object作为父类确保你在python2的OOP中定义了一个新式类。

## 例子：小狗公园
假设我们在一个小狗公园里。这里有很多属性不同的狗的对象。一些狗在跑，有些在抻懒腰另一些只是站着不动看其他的狗。此外每只狗都有一个主人给的名字，一个年龄，每只狗都是活着的。

怎么区分其中一只狗和剩余的狗呢？用狗的品种怎么样：
```python
>>> class Dog:
...     def __init__(self, breed):
...         self.breed = breed
...
>>> spencer = Dog("German Shepard")
>>> spencer.breed
'German Shepard'
>>> sara = Dog("Boston Terrier")
>>> sara.breed
'Boston Terrier'
```
不同品种的狗会有不同的行为。我们来为每个不同品种分开创建不同的类，这些类是Dog类的子类。
## 扩展一个父类的功能
创建一个叫dog_inheritance.py的文件：
```python
# 父类
class Dog:

    # 类的属性
    species = 'mammal'

    # 实例化/实例属性
    def __init__(self, name, age):
        self.name = name
        self.age = age

    # 实例方法
    def description(self):
        return "{} is {} years old".format(self.name, self.age)

    # 实例方法
    def speak(self, sound):
        return "{} says {}".format(self.name, sound)


# 子类 (继承于Dog类)
class RussellTerrier(Dog):
    def run(self, speed):
        return "{} runs {}".format(self.name, speed)


# 子类 (继承于Dog类)
class Bulldog(Dog):
    def run(self, speed):
        return "{} runs {}".format(self.name, speed)


# 子类继承父类的属性和方法
jim = Bulldog("Jim", 12)
print(jim.description())

# 子类也有自己的属性和方法
print(jim.run("slowly"))
```
在你敲这段代码的时候阅读以下注释会帮助你理解到底发生了什么，在你运行程序前你可以预测以下输出是什么。

你会看到：
```python
Jim is 12 years old
Jim runs slowly
```
我们没有添加特殊的属性或方法来区分RussellTerrier品种或者Bulldog品种的狗，但是现在他们有了不同的类，我们可以在实例化时给他们不同的属性定义不同的速度。

## 父类 vs. 子类
isinstance()函数用来判断一个实例是否是也是某个父类的实例。

将下面的代码存为dog_isinstance.py：
```python
# Parent class
class Dog:

    # Class attribute
    species = 'mammal'

    # Initializer / Instance attributes
    def __init__(self, name, age):
        self.name = name
        self.age = age

    # instance method
    def description(self):
        return "{} is {} years old".format(self.name, self.age)

    # instance method
    def speak(self, sound):
        return "{} says {}".format(self.name, sound)


# Child class (inherits from Dog() class)
class RussellTerrier(Dog):
    def run(self, speed):
        return "{} runs {}".format(self.name, speed)


# Child class (inherits from Dog() class)
class Bulldog(Dog):
    def run(self, speed):
        return "{} runs {}".format(self.name, speed)


# Child classes inherit attributes and
# behaviors from the parent class
jim = Bulldog("Jim", 12)
print(jim.description())

# Child classes have specific attributes
# and behaviors as well
print(jim.run("slowly"))

# Is jim an instance of Dog()?
print(isinstance(jim, Dog))

# Is julie an instance of Dog()?
julie = Dog("Julie", 100)
print(isinstance(julie, Dog))

# Is johnny walker an instance of Bulldog()
johnnywalker = RussellTerrier("Johnny Walker", 4)
print(isinstance(johnnywalker, Bulldog))

# Is julie and instance of jim?
print(isinstance(julie, jim))
```
输出：
```python
('Jim', 12)
Jim runs slowly
True
True
False
Traceback (most recent call last):
  File "dog_isinstance.py", line 50, in <module>
    print(isinstance(julie, jim))
TypeError: isinstance() arg 2 must be a class, type, or tuple of classes and types
```
输出说得通吗？jim和julie都是Dog()类的实例，johnnywalker不是Bulldog()类的实例。然后作为一个完整性检查，我们测试了julie是否是jim的实例，这是不存在的，因为jim是一个类的实例，而不是类本身，这就是TypeError的原因。

## 覆盖/重写父类的功能
子类可以覆盖父类的属性或行为。举个例子：
```python
>>> class Dog:
...     species = 'mammal'
...
>>> class SomeBreed(Dog):
...     pass
...
>>> class SomeOtherBreed(Dog):
...     species = 'reptile'
...
>>> frank = SomeBreed()
>>> frank.species
'mammal'
>>> beans = SomeOtherBreed()
>>> beans.species
'reptile'
```
SomeBreed()类从父类中继承了species属性，SomeOtherBreed()类重写了species属性将其设置为reptile。
## 回顾练习(#2)
练习：“小狗继承”创建一个容纳小狗实例的Pets类;这个类完全独立于Dog类。换句话说，Dog类不会从Pets类继承。然后将三个dog实例分配给Pets类的一个实例。从下面的代码开始。将文件保存为pets_class.py。你的输出应该如下所示:
```python
I have 3 dogs. 
Tom is 6. 
Fletcher is 7. 
Larry is 9. 
And they're all mammals, of course.
```
开始代码：
```python
# Parent class
class Dog:

    # Class attribute
    species = 'mammal'

    # Initializer / Instance attributes
    def __init__(self, name, age):
        self.name = name
        self.age = age

    # instance method
    def description(self):
        return "{} is {} years old".format(self.name, self.age)

    # instance method
    def speak(self, sound):
        return "{} says {}".format(self.name, sound)

# Child class (inherits from Dog class)
class RussellTerrier(Dog):
    def run(self, speed):
        return "{} runs {}".format(self.name, speed)

# Child class (inherits from Dog class)
class Bulldog(Dog):
    def run(self, speed):
        return "{} runs {}".format(self.name, speed)
```
练习：“饥饿的小狗”使用相同的文件，向Dog类添加一个实例属性is_hungry = True。然后添加一个eat()方法，当调用eat()方法时将is_hungry设置为False。找出喂饱每只狗的最佳方法然后输出"My dogs are hungry"。如果全部的狗都不饿，最后输出如下：
```python
I have 3 dogs. 
Tom is 6. 
Fletcher is 7. 
Larry is 9. 
And they're all mammals, of course. 
My dogs are not hungry.
```
(还有两个练习 见原文链接 不想写了 答案也见原文链接)

## 结尾
现在你应该了解了什么是类，你为什么需要使用它以及如何使用父类和子类让你的程序结构更好。

需要注意的是OOP是一种编程范式而不是python的概念。大对数现代编程语言例如Java，C#，C++都主要使用OOP。所以好消息是学习面向对象编程基础对你来说很有价值，不管你是否使用python工作。
