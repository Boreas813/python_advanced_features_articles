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

如果你的程序依赖为纯Python依赖那会增加PyInstaller打包成功的概率。如果你有很复杂的C/C++依赖那也不能强求成功。

PyInstaller支持很多流行库，[NumPy](http://www.numpy.org/)、[PyQt](https://pypi.org/project/PyQt5/)和[Matplotlib](https://matplotlib.org/)，你不需要做额外的工作处理这些库的依赖。你可以查看[PyInstaller文档](https://github.com/pyinstaller/pyinstaller/wiki/Supported-Packages)寻找更多官方支持的库。

如果你的依赖没出现在官方文档中也不要担心，很多Python包都可以很好地工作。实际上PyInstaller非常流行，很多项目都有关于如何使用PyInstaller的说明。

总之你的项目开箱即用的概率很高。

要尝试创建全部参数为默认的可执行文件，只需给PyInstaller主入口点脚本的名称。

首先`cd`到你的程序入口点目录然后向`pyinstaller`命令传递你的入口点文件，这个命令是安装PyInstaller时自动安装的。

例如，使用`reader`项目，`cd`到项目顶层目录后输入：
```shell
$ pyinstaller cli.py
```

不要被构建可执行文件时的大量输出吓到，PyInstaller的输出是冗长的，在debug时还可以将详细程度大大提高，稍后你会看到这一点。

## 深入研究PyInstaller套件

PyInstaller的底层运作复杂并会产生大量输出。所以要清楚什么是首要目标，就是分发给你的用户的可执行文件和潜在的调试信息。默认时，`pyinstaller`命令会创建如下文件：

* 一个`*.spec`文件
* 一个`build/`文件夹
* 一个`dist/`文件夹

### Spec文件
默认情况下，spec文件以CLI脚本命名。在之前的例子中你会看到文件名为`cli.spec`。下面是运行完后spec文件的样子：
```python
# -*- mode: python -*-

block_cipher = None

a = Analysis(['cli.py'],
             pathex=['/Users/realpython/pyinstaller/reader'],
             binaries=[],
             datas=[],
             hiddenimports=[],
             hookspath=[],
             runtime_hooks=[],
             excludes=[],
             win_no_prefer_redirects=False,
             win_private_assemblies=False,
             cipher=block_cipher,
             noarchive=False)
pyz = PYZ(a.pure, a.zipped_data,
             cipher=block_cipher)
exe = EXE(pyz,
          a.scripts,
          [],
          exclude_binaries=True,
          name='cli',
          debug=False,
          bootloader_ignore_signals=False,
          strip=False,
          upx=True,
          console=True )
coll = COLLECT(exe,
               a.binaries,
               a.zipfiles,
               a.datas,
               strip=False,
               upx=True,
               name='cli')
```
这个文件会被`pyinstaller`命令自动生成。你自己的版本会有不同的路径名，但是大体是相同的

别担心，你不用理解上述代码就可以有效使用PyInstaller！

这个文件可以被修改并在后续重复使用。通过提供spec文件而不是入口点脚本，你可以使之后的构建速度加快。

这里有一些[特别的spec案例文件](https://pyinstaller.readthedocs.io/en/stable/spec-files.html#using-spec-files)。但如果是简单项目，你不需要担心这些细节，除非你想客制化自己的项目被如何构建。

### Build文件夹
`build/`文件夹下包含PyInstaller用来构建可执行文件的元数据和内部流水。默认情况下文件夹是这样：
```
build/
|
└── cli/
    ├── Analysis-00.toc
    ├── base_library.zip
    ├── COLLECT-00.toc
    ├── EXE-00.toc
    ├── PKG-00.pkg
    ├── PKG-00.toc
    ├── PYZ-00.pyz
    ├── PYZ-00.toc
    ├── warn-cli.txt
    └── xref-cli.html
```
该文件夹对debug非常有用，除非你遇到了问题，要不然这个文件夹可以被忽略。在该文章的后面你会学到更多debug的知识。

### Dist文件夹
在构建完成后，你会得到`dist/`文件夹：
```
dist/
|
└── cli/
    └── cli
```
`dist/`文件夹包含你要发给用户的可执行文件。可执行文件会在`dist/cli/cli`，或者`dist/cli/cli.exe`如果你用的Windows。

你可能也会找到大量带`.so`、`.pyd`、`.dll`的文件，这些是PyInstaller创建和收集的项目依赖项的共享库。

* 注意：如果你使用`git`工作流，可以将`*.spec`，`build/`，`dist/`添加至`.gitignore`文件来保证`git status`整洁。默认的[python项目github忽略文件](https://github.com/github/gitignore/blob/master/Python.gitignore)已经帮你做完了这些。

此时，如果你执行完上述示例，可以尝试运行`dist/cli/cli`可执行文件。

你会注意到，运行可执行文件会产生`version.txt`的报错。这是因为该项目依赖一些PyInstaller不知道的额外数据文件。为了修复这个问题，你要告诉PyInstaller需要`version.txt`，所以你会在下面[测试你的可执行文件](https://realpython.com/pyinstaller-python/#testing-your-new-executable)的章节学到这些。

## 定制化你的构建
你可以向PyInstaller提供大量参数来定制化构建过程，下面是几个常用而且有帮助的参数。

`--name`

`更改可执行文件名`
这是一种避免使用入口点脚本的方法，如果你像我一样习惯将入口点脚本命名为`cli.py`之类的东西，那么`--name`就很有用

将可执行文件命名为`realpython`的命令如下：
```shell
$ pyinstaller cli.py --name realpython
```

`--onefile`

`将你的整个应用打包成单一可执行文件`

默认设置会创建一个文件夹里面包含依赖和可执行文件，`--onefile`不产生dll等依赖，只生成单一可执行文件。

想打包你的项目到单一可执行文件，你可以使用如下命令：
```shell
$ pyinstaller cli.py --onefile
```

通过上述命令，你的`dist/`目录下只会生成一个可执行文件。

`--hidden-import`

`列出PyInstaller无法自动检测的top-level导入`

该选项用来解决你在函数内调用`import`和`__import__()`无法被自动检测的问题。你可以在命令行中多次添加该选项。

该选项需要你import的包名。比如你的项目中有函数导入了`requests`库，PyInstaller不会自动添加`requests`到你的可执行文件。你需要如下命令：
```shell
$ pyinstaller cli.py --hiddenimport=requests
```

你可以在构建命令行中使用多次，每次指定一个库。

`--add-data`和`--add-binary`
`指示PyInstaller插入额外数据二进制文件`

当你想引入配置文件、示例或其他非代码数据时很有用，在之后的章节你会看到例子。

`--exclude-module`

`将某些库排除在可执行文件之外`

这对排除只针对开发人员的需求(如测试框架)非常有用。这是一个让打包后文件尽可能小的好方法。如果你使用[pytest](https://realpython.com/pytest-python-testing/)，你可能会用下面命令排除它：
```shell
$ pyinstaller cli.py --exclude-module=pytest
```

`-w`

`避免自动打开一个标准输出的控制台`

如果你构建的是GUI应用，这个选项非常有用。这帮你不让用户看到一个终端和里面的输出细节。

和`--onefile`一样，`-w`不需要参数
```shell
$ pyinstaller cli.py -w
```

`.spec`文件
如前所述，你可以重用自动生成的`.spec`文件来进一步定制可执行文件。`.spec`文件是一个常规的Python脚本，它隐式地使用PyInstaller库的API。

你可以在[这里](https://pyinstaller.readthedocs.io/en/stable/spec-files.html)找到关于API的更多细节

## 测试你的可执行文件

测试新可执行文件的最佳方法是在新机器上测试。新机器应该和构建机器的操作系统相同。理想情况下，这台机器应该与用户使用的机器尽可能相似。这并不总是可行的，所以最好的办法是在你自己的机器上进行测试。

测试的关键是在没有安装任何开发环境时运行可执行文件。意思是运行在没有`virtualenv`，`conda`或任何可以访问你Python解释器的环境里。PyInstaller创建可执行文件的一个主要目标就是用户无需再安装任何依赖。

重新回到之前的feed reader例子，你会发现运行`dist/cli`目录下的`cli`可执行文件报错：
```shell
FileNotFoundError: 'version.txt' resource not found in 'importlib_resources'
[15110] Failed to execute script cli
```

`importlib_resources`包需要`version.txt`文件。你可以添加`--add-data`选项进行构建。代码如下：
```shell
$ pyinstaller cli.py \
    --add-data venv/reader/lib/python3.6/site-packages/importlib_resources/version.txt:importlib_resources
```

该命令告诉PyInstaller将包含`version.txt`的文件夹`importlib_resources`引入到你的构建中

* 注意：`pyinstaller`命令使用`\`字符使命令更容易阅读。当你自己运行命令时，可以省略`\`或复制并粘贴下面的命令，如果您使用相同的路径。

再次运行新打包的可执行文件，这次会抛出一个新的关于`config.cfg`的错误。

这个文件也是项目的依赖，所以你也应该把他添加进你的构建命令中：
```shell
$ pyinstaller cli.py \
    --add-data venv/reader/lib/python3.6/site-packages/importlib_resources/version.txt:importlib_resources \
    --add-data reader/config.cfg:reader
```
同样，根据你自己电脑来调整文件路径。

至此，你应该拥有一个可以直接打开的可执行文件了！

## Debugging PyInstaller Executables

如上所述，在运行可执行文件时可能会遇到问题。根据项目的复杂性，修复可以简单引入数据文件，但是有时也需要更多的调试技术。

以下是一些常见的策略，排名不分先后。通常情况下，这些策略中的一种或组合将在艰难的调试过程中带来突破。

### 使用终端

首先，尝试从终端运行可执行文件，这样可以看到所有的输出。

请记住删除`-w`选项，以便在控制台窗口中查看所有标准输出。通常，如果缺少依赖，会看到`ImportError`异常。

### Debug文件

查看`build/cli/warn-cli.txt`文件寻找问题。PyInstaller在打包过程中产生大量输出日志，查看这些日志是好的开始

### 单目录构建

使用`--onedir`构建一个发布目录而不是单个可执行文件。该选项是默认使用的，它给你检查目录内依赖的情况。

`--onedir`适合debug，但是`--onefile`对用户更友好。先debug，然后加入`--onefile`选项进行打包。

### 额外的CLI选项

PyInstaller有CLI选项让构建时输出大量信息。使用`--log-level=DEBUG`选项重新构建试一试。

该选项会产生大量日志，将它定向到一个文件比较好：
```shell
$ pyinstaller --log-level=DEBUG cli.py 2> build.txt
```

上述命令会生成一个`build.txt`的文件，当中包含大量debug信息。

* 注意：普通重定向`>`是不起作用的，PyInstaller打印到`stderr`而不是`stdout`。意思是你需要重定向标准错误到文件，需要加一个`2`

```
67 INFO: PyInstaller: 3.4
67 INFO: Python: 3.6.6
73 INFO: Platform: Darwin-18.2.0-x86_64-i386-64bit
74 INFO: wrote /Users/realpython/pyinstaller/reader/cli.spec
74 DEBUG: Testing for UPX ...
77 INFO: UPX is not available.
78 DEBUG: script: /Users/realptyhon/pyinstaller/reader/cli.py
78 INFO: Extending PYTHONPATH with paths
['/Users/realpython/pyinstaller/reader',
 '/Users/realpython/pyinstaller/reader']
```

### PyInstaller文档

[PyInstaller GitHub Wiki](https://github.com/pyinstaller/pyinstaller/wiki)有大量有用的链接可debug技巧。最有名的是[确保所有东西都正确打包](https://github.com/pyinstaller/pyinstaller/wiki/How-to-Report-Bugs#make-sure-everything-is-packaged-correctly)和[如果它搞砸了](https://github.com/pyinstaller/pyinstaller/wiki/If-Things-Go-Wrong)。

### 协助依赖检查

你看到`ImportError`那多半是PyInstaller不能正确检测你的依赖。上文提到过，你可能使用了` __import__()`，在函数内import或其它类型的[隐藏导入](https://pyinstaller.readthedocs.io/en/stable/when-things-go-wrong.html?highlight=Hidden#listing-hidden-imports)

大部分问题可以通过`--hidden-import`选项解决。

另一个解决办法是[钩子文件](https://pyinstaller.readthedocs.io/en/stable/hooks.html)。这些文件包含帮助PyInstaller打包的额外信息。你可以编写你自己的hook然后使用选项`--additional-hooks-dir`。

钩子文件是PyInstaller本身在内部工作的方式，你可以在[PyInstaller源码](https://github.com/pyinstaller/pyinstaller/tree/develop/PyInstaller/hooks)找到更多例子。

## 局限

PyInstaller非常强大，但它也有一些限制。前面已经讨论了一些限制:入口点脚本中的隐藏导入和相对导入。

PyInstaller支持为Windows、Linux和macOS打包可执行文件，但它不能[交叉编译](https://en.wikipedia.org/wiki/Cross_compiler)。因此，不能从另一个操作系统构建针对这个操作系统的可执行文件。因此，要为多种类型的操作系统分发可执行文件，需要为每种受支持的操作系统配置一台构建机器。

与交叉编译限制相关的是，要知道PyInstaller在技术上并不能绝对绑定应用程序运行所需的所有内容。您的可执行文件仍然依赖于用户的[glibc](https://en.wikipedia.org/wiki/GNU_C_Library)。通常来说，可以通过在每个OS的最老版本上构建来绕过glibc限制。

## 总结

PyInstaller可以使复杂的安装文档变得不必要。相反，你的用户可以简单地运行你的可执行文件，尽可能快地启动。PyInstaller工作流程可以总结出来：

1. 创建入口点脚本用来启动你的主程序
2. 安装PyInstaller
3. 在入口点运行PyInstaller
4. 测试你的可执行文件
5. 把`dist/`目录分发给你的用户
