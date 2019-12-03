# 使用PyInstaller轻松打包Python应用
[原文链接](https://realpython.com/pyinstaller-python/)

<!-- TOC -->

- [使用PyInstaller轻松打包Python应用](#使用pyinstaller轻松打包python应用)
    - [发布时的问题](#发布时的问题)
    - [PyInstaller](#pyinstaller)
    - [准备你的项目](#准备你的项目)
    - [使用PyInstaller](#使用pyinstaller)

<!-- /TOC -->

你是否嫉妒[Go](https://realpython.com/pyinstaller-python/)开发者能轻松的打包自己的程序成可执行文件然后分发给自己的用户？
如果你的用户可以**不需要安装任何额外组件就可以运行你的应用**岂不是很棒？
[PyInstaller](https://pyinstaller.readthedocs.io/en/stable/)能在Python生态中帮你实现这个梦想。

有数不清的教程教你如何[搭建虚拟环境](https://realpython.com/dependency-pitfalls/)，[管理依赖](https://realpython.com/products/managing-python-dependencies/)，[发布到PyPI](https://realpython.com/pypi-publish-python-package/)，当你想制作Python库的时候这些很有帮助。但是很少有文章介绍**building Python应用**，这篇指南是给那些想给可能不是Python开发者的用户发布应用的开发者准备的。

**在这篇指南中，你会学到如下内容：**

* PyInstaller如何简化程序的发布

* 如何在你的项目上使用PyInstaller

* 如何调试PyInstaller错误

* PyInstaller能做什么

[PyInstaller](https://pyinstaller.readthedocs.io/en/stable/)让你有能力生成用户可以直接运行的可执行程序。为了充分了解PyInstaller的功能，先看一下PyInstaller能帮你避免什么发布时的问题

## 发布时的问题

尤其对非开发者来说，设置Python项目环境就很令人沮丧了。通常搭建项目环境从打开一个终端输入一些命令开始，这对大量的用户来说是不可能的。在安装指南深入到虚拟环境、Python版本和大量潜在依赖关系这些复杂细节之前，这个障碍就已经组织了用户。

想一下在一台新机器上搭建Python环境，一般来说要这么做：
* 下载并安装指定版本的Python解释器
* 设置pip
* 设置虚拟环境
* 复制你的代码进来
* 安装依赖

停下来想一下如果你不是开发人员，上面的步骤对你来说是否有意义？可能不会。

如果你的用户幸运的得到了这些安装依赖那一切ok。在过去几年随着wheel的流行这一点已经变好了很多，但是一些依赖仍然需要C/C++甚至FORTRAN编译器（比如金融库ta-lib）。

如果你的目标是让更多的用户使用你的应用，那这个门槛就太高了。正如[Raymond Hettinger](https://mobile.twitter.com/raymondh)在[ excellent talks](https://pyvideo.org/speaker/raymond-hettinger.html)上常说的一样，“There has to be a better way.”

## PyInstaller
[PyInstaller](https://pyinstaller.readthedocs.io/en/stable/)抽象出这些细节，找出应用全部的依赖并集成到一起。你的用户甚至不知道他在运行一个Python项目，因为Python解释器已经集成到可执行程序中了。从此告别复杂的安装文档！

PyInstaller首先会[内省](https://en.wikipedia.org/wiki/Type_introspection)你的代码，侦测你的依赖，然后将他们打包成适当的格式，这取决于你的操作系统。

PyInstaller有很多有趣的细节，但现在你要学的是基本用法。如果你想要更多细节请参考[极好的PyInstaller文档](https://pyinstaller.readthedocs.io/en/stable/operating-mode.html#analysis-finding-the-files-your-program-needs)。

此外，PyInstaller能创建Windows，Linux和macOS的可执行文件。Windows用户会得到一个`.exe`，Linux用户会得到一个常规可执行文件，macOS用户会得到一个`.app`文件。这里有一些说明和警告，详情见[限制]()章节。

## 准备你的项目

PyInstaller需要你的应用满足某个最小结构，也就是你要有一个CLI脚本启动你的应用。通常做法是在项目外创建一个脚本，import你的项目然后运行`main()`。

入口点脚本是一个Python脚本，技术层面来说你可以在入口点脚本中做任何事，但是最好避开[显式相对导入](https://realpython.com/absolute-vs-relative-python-imports/#relative-imports)。如果你喜欢这么做，那仍然可以在应用程序的其余部分中使用相对导入。

* 注意：入口点（entry-point）指的是启动你项目或应用的代码

你可以拿自己的项目试试，也可以使用[Real Python feed reader project](https://github.com/realpython/reader)。[reader project](https://github.com/realpython/reader)的详细信息在这篇教程中可以找到：[如何发布一个包到PyPI](https://realpython.com/pypi-publish-python-package/)。

构建项目的可执行版本的第一步是添加一个入口点脚本。幸运的是，feed reader项目结构良好，因此只需在 *包外* 编写一个简短的脚本并运行它。举个例子，在包外创建一个叫`cli.py`的文件，代码如下：
```python
from reader.__main__ import main

if __name__ == '__main__':
    main()
```

`cli.py`脚本调用`main()`函数启动feed reader项目。

在你自己的项目上创建入口点脚本非常简单，因为你了解你的代码。然而其他人的代码就不是很好找入口点了。在这种情况下，你可以从查看第三方项目中的`setup.py`文件开始。

在项目的`setup.py`中查找对`entry_points`参数的引用。例如，这是reader项目的`setup.py`：
```python
    name="realpython-reader",
    version="1.0.0",
    description="Read the latest Real Python tutorials",
    long_description=README,
    long_description_content_type="text/markdown",
    url="https://github.com/realpython/reader",
    author="Real Python",
    author_email="office@realpython.com",
    license="MIT",
    classifiers=[
        "License :: OSI Approved :: MIT License",
        "Programming Language :: Python",
        "Programming Language :: Python :: 2",
        "Programming Language :: Python :: 3",
    ],
    packages=["reader"],
    include_package_data=True,
    install_requires=[
        "feedparser", "html2text", "importlib_resources", "typing"
    ],
    entry_points={"console_scripts": ["realpython=reader.__main__:main"]},
)
```

可以看到，入口点`cli.py`脚本调用了`entry_points`参数中提到的相同函数。

添加完新的脚本文件之后，reader项目的文件结构如下，假设你将该项目放置在`reader`文件夹下：
```
reader/
|
├── reader/
|   ├── __init__.py
|   ├── __main__.py
|   ├── config.cfg
|   ├── feed.py
|   └── viewer.py
|
├── cli.py
├── LICENSE
├── MANIFEST.in
├── README.md
├── setup.py
└── tests
```

这里reader项目的代码没有变化，只是新建了一个叫`cli.py`的文件。入口点脚本通常是打包你的项目的必要内容。

但你仍需要寻找函数内部的`__import__()`调用或导入，这种在PyInstaller术语中被称为[隐藏导入](https://pyinstaller.readthedocs.io/en/stable/when-things-go-wrong.html?highlight=Hidden#listing-hidden-imports)。

如果你感觉修改应用中的导入有困难，可以手工让PyInstaller强制导入这些隐藏导入的依赖库，教程接下来会介绍怎么做。

一旦你可以在项目外使用Python脚本启动你的程序，就可以尝试创建可执行文件了。

## 使用PyInstaller

首先从[PyPI](https://pypi.org/)安装PyInstaller，使用pip即可：
```shell
$ pip install pyinstaller
```

`pip`会安装PyInstaller的依赖和一个新命令：PyInstaller。你可以在Python代码中导入PyInstaller作为库使用，但你可能只会将它当作CLI工具使用。

如果你想[创建自己的钩子文件](https://pyinstaller.readthedocs.io/en/stable/hooks.html#)，可以使用它的库接口。

如果你的程序依赖为纯Python依赖那会增加PyInstaller打包成功的概率。如果你又很复杂的C/C++依赖那也不要强求能成功。

PyInstaller支持很多流行库，[NumPy](http://www.numpy.org/)、[PyQt](https://pypi.org/project/PyQt5/)和[Matplotlib](https://matplotlib.org/)，你不需要做额外的工作处理这些库的依赖。你可以查看[PyInstaller文档](https://github.com/pyinstaller/pyinstaller/wiki/Supported-Packages)寻找更多官方支持的库。

如果你的一些依赖没出现在官方文档中也不要担心，很多Python包都可以很好地工作。实际上PyInstaller非常流行，很多项目都有关于如何使用PyInstaller的说明。

总之你的项目开箱即用的概率很高。

要尝试创建全部参数为默认的可执行文件，只需给PyInstaller主入口点脚本的名称。

