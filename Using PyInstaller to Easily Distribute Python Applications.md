# 使用PyInstaller轻松打包Python应用
[原文链接](https://realpython.com/pyinstaller-python/)

[TOC]

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
