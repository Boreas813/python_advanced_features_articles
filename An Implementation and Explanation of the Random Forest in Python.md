# Python中随机森林的实现和详解

[原文链接](https://towardsdatascience.com/an-implementation-and-explanation-of-the-random-forest-in-python-77bf308a9b76)


随着[Scikit-Learn](http://scikit-learn.org/)等库的出现，已经很简单在Python中实现[几百种机器学习算法](http://scikit-learn.org/stable/supervised_learning.html)。我们也不怎么清楚算法的底层是如何实现的，清楚所有的细节并不重要，但对了解机器学习的工作原理是有帮助的。这可以让我们在模型表现不佳时做诊断或解释它是如何决策的。如果我们想说服别人相信我们的模型，这一点很重要。

这篇文章中，我们会看到如何在Python中构建随机森林。除了代码之外，我们尝试理解模型的原理。因为随机森林是很多决策树组成的，我们先了解单个决策树是怎么对简单问题进行分类的，然后我们使用随机森林解决一个实际世界中的数据科学问题，本文用到的完整代码[你可以在这里找到](https://github.com/WillKoehrsen/Machine-Learning-Projects/blob/master/Random%20Forest%20Tutorial.ipynb)

**Note：** 这篇文章[原始出现](https://enlight.nyc/random-forest)在[enlight](https://enlight.nyc/)，一个社区驱动的开源平台，为那些想学机器学习的人提供教程。

## 理解决策树

[决策树](http://scikit-learn.org/stable/modules/tree.html)是随机森林的基本组成元素。我们可以理解为决策树是向我们的数据询问一系列是/否问题，最终结果指向一个预测类（在回归情景中是连续值）。这是一个可解释的模型，因为它做分类的方式和我们很像：我们在拥有的数据上做一系列的查询知道我们得出结论（在理想环境中）。

[决策树的技术细节](https://machinelearningmastery.com/classification-and-regression-trees-for-machine-learning/)是关于如何把对数据的问题组织起来。在[CART算法](https://www.stat.wisc.edu/~loh/treeprogs/guide/wires11.pdf)中决策树是通过确定问题（术语为splits of nodes）来构建的，当回答问题时让[基尼不纯度](https://en.wikipedia.org/wiki/Decision_tree_learning)最大下降。意思就是决策树尝试在特征中寻找值来形成来自单个类的高比例样本结点，将数据清晰地划分为类。

过会我们讨论基尼不纯度的底层细节，首先我们先构建一个决策树以便让我们在高层次理解它。

## 简单问题的决策树

我们从一个简单二进制分类问题开始：
![](https://miro.medium.com/max/731/0*dvVMJdNRzlUqOl2Z)
目标是将数据点划分为它们各自的类

数据只有两个特性（预测变量），`x1`和`x2`有六个数据点样本，分为两个不同标签。这个问题虽然简单，但也不是线性可分的，我们不能画一条直线将数据分类。

但是我们可以画一系列直线把数据按区域分开，这个称之为节点（node）。这就是决策树在训练过程中做的事。决策树实际上是通过构造许多线性边界而建立的[非线性模型](https://datascience.stackexchange.com/questions/6787/is-decision-tree-algorithm-a-linear-or-nonlinear-algorithm)。

我们用Scikit-Learn创建并训练决策树。

```python
from sklearn.tree import DecisionTreeClassifier

# Make a decision tree and train
tree = DecisionTreeClassifier(random_state=RSEED)
tree.fit(X, y)
```

训练期间我们向模型输入特征和标签，然后它开始学习基于特征进行分类（在这个简单的例子我们没使用测试集，在实际测试时我们只给模型输入特征，让它对标签进行预测）。

如下代码在训练集上测试模型准确率：
```python
print(f'Model Accuracy: {tree.score(X, y)}')

Model Accuracy: 1.0
```

上面代码得到了100%准确率，这正是我们想要的，因为训练中我们没限制树的深度。因为这样可能导致*过拟合（overfitting）*，这种完全学习的能力可能是决策树的缺点，我们一会再讨论。

## 可视化决策树

所以当我们训练决策树的时候发生了什么？一个更好理解的方法是将其可视化，我们用Scikit-Learn的一个函数实现（具体细节看这篇[文章](https://medium.com/p/38ad2d75f21c?source=user_profile---------4------------------)）

![](https://miro.medium.com/max/875/0*QwJ2oZssAQ2_cchJ)

除了叶节点（带颜色的终端结点）以外的其他节点都有五个部分：

1. 关于特征值的问题。每个问题都用是或否来分割结点，然后数据点基于答案向树下方移动。

2. `gini`：节点的基尼不纯度。随着我们向树下方移动，加权平均的基尼不纯度在减小。

3. `samples`：节点中的观测次数。

4. `value`：每个分类中的样本数量。举个例子，顶级节点有两个样本在class0中，4个样本在class1中。

5. `class`：节点中点的主要分类。在本例的叶节点中是对节点中全部样本的预测。

叶节点不再有问题，因为这是做的最终预测。对一个新数据点进行分类，只需沿着树向下移动，使用数据点特征来回答问题，直到到达一个叶节点，其中的`class`就是预测。

从另一个角度看树，我们在原始数据上画出决策树给出的分割。
![](https://miro.medium.com/max/875/1*MCQ6yUvb3i2HTCEh-Cuz2Q.png)

每个分割都是单独的线，根据特征的值将数据点划分到各个节点。对于这个简单问题且没有最大深度的限制，决策树按照每个节点只有一个类将数据划分到节点中（再次提醒，稍后我们会看到这样的划分不是我们想要的，它会导致过拟合）。

## 基尼不纯度
这时我们来深入[基尼不纯度](https://en.wikipedia.org/wiki/Decision_tree_learning#Gini_impurity)的概念（数学并不是令人胆怯的！）。节点的基尼不纯度是：有一个节点，它是由节点内样本分布来标记的，在这个节点随机选取样本被不正确标注的可能性。举个例子，在顶级节点（根节点），有44.4%的概率根据节点中的样本标签对随机选择的数据点进行错误分类。我们用下面的方程得到这个值：
![](https://miro.medium.com/max/875/1*mcHzG8OjhQ2ryiBH7MBPUA.png)

节点`n`的基尼不纯度为1减去每个类别中样本的分数`p_i`的平方对所有类别`J`（对于二元分类任务这里J是2）的总和。这个定义可能让人有些困惑，我们看下根节点的基尼不纯度是怎么算的。

![](https://miro.medium.com/max/1250/1*uAGS042OxMJ4Ic3k4s313Q.png)


在每个节点，决策树在特征中搜索要分割的值，以最大限度让基尼不纯度下降（[分离节点的一种方法是使用信息增益](https://datascience.stackexchange.com/questions/10228/gini-impurity-vs-entropy)，一个相关概念）。

之后它在贪婪地递归中重复这个分割过程直到到达最大深度或者每个节点只包含一个类型的样本。各层次树的甲醛总基尼不纯度必须下降。在树的第二层，基尼不纯度为0.333：
![](https://miro.medium.com/max/1250/1*gdMrk7yEPJLio0d0Sixtkg.png)

（每个节点的基尼不纯度由该节点中来自父节点的点数的分数加权。）你可以继续解出每个节点的基尼不纯度。通过一些基本的数学运算，一个强大的模型诞生了！

最终最后一层的加权总基尼不纯度变为0意味着每个节点都完全纯净没有随机选择的数据会被误分类。虽然这看起来是积极的，但这意味着模型可能存在过拟合，因为节点仅使用训练集来构建。

## 过拟合：或许为什么森林好过只有一颗树

你可能会问为什么不知用决策树就完了？它看起来是从不出错的完美分类器！记住一个关键点就是决策树**在训练集上**从不出错。机器学习模型的目标是很好地概括它从未见过的**新数据**。

当我们有一个[非常灵活的模型](http://qr.ae/TUNozZ)（模型容量很大），且它本质上是通过密切地**拟合**来记忆训练数据时就会发生过拟合。这个问题是，这个模型不仅学习训练数据中的实际关系，还学习任何存在的噪声。一个灵活的模型会有很高的*方差（variance）*，因为学习到的参数会随着数据集而有很大的变化。

########该段关于过拟合问题的讨论######

## 随机森林

[随机森林](https://www.stat.berkeley.edu/~breiman/RandomForests/cc_home.htm)是很多决策树组成的。比起决策树简单平均预测，随机森林使用[两个随机的概念](https://www.stat.berkeley.edu/~breiman/randomforest2001.pdf)

1. 构建决策树时对训练集进行随机抽样
2. 分割节点时考虑特征的随机子集

### 随机抽样

每棵树训练时使用样本中的随机数值，使用[drawn with replacement](https://en.wikipedia.org/wiki/Bootstrapping_(statistics))来绘制样本，被称为*bootstrapping*，这意味着一些样本将在单个树中多次使用。这个想法是通过不同的样本训练每棵树，尽管每棵树可能对特定的训练集有很高的方差（variance ），但总体上整个森林的方差会更低，且不会以增加偏差（bias）为代价。

在测试时，通过平均每个决策树的预测做最终预测。这种在不同引导数据子集上训练每个个体学习者，然后对预测进行平均的过程被称为bagging，[bootstrap aggregating](https://machinelearningmastery.com/bagging-and-random-forest-ensemble-algorithms-for-machine-learning/)的简称。

### 分割节点的随机特征子集

随机森林的另一个主要概念是只考虑所有特征的子集来分割每个决策树的每个节点。为了进行分类，通常设置为sqrt(n_features)，意思是如果有16个特征，在每棵树的每个节点上只考虑4个随即特征来分割节点(随机森林也可以像回归中常见的那样，考虑每个节点上的所有特征来训练。这些选项可以在Scikit-Learn随机森林实现中控制)。

如果你能理解单个决策树，bagging的理念，特征的随机子集，那你就很好地理解了随机森林的工作原理：

*随机森林将数百或数千颗决策树组合在一起，根据不同的训练集训练每棵树，考虑有限数量的特征对每棵树中的节点进行分割，随机森林的最终预测是通过对每棵树的预测求平均值实现的。*

为了理解为什么随机森林比单个决策树好，想象一下这个情景：你要来预测特斯拉的股票是否会上涨，你可以接触到十几个事先对特斯拉一无所知的分析师。每个分析师的偏差（bias）都很低，因为他们不带有任何主观假设，且都可以从股票报告中来学习。

这可能是个理想情况，但问题是除了真实信号之外，报告可能包含噪声。由于分析师的预测完全基于数据，他们可能被无关的信息左右。分析师可能会从同一数据集得出不同的预测。此外，每个分析师都有很高的方差，如果给出不同的训练报告，他们会得出完全不同的预测。

所以解决方案是不依赖任何一个人，而是把每个分析师的投票做成池。此外就像随机森林中一样，只允许每个分析师访问报告的一部分，希望这样做能抵消噪声的影响。现实中我们依赖于多个信息源，因此不光决策树是靠直觉的，在随机森林中组合他们也是。

## 随机森林实践

接下来我们用python的Scikit-Learn构建一个随机森林。这次我们用现实世界的数据集分割成训练和测试集。我们使用测试集来评估模型在新数据上的表现，这也可以让我们确定模型有多少过拟合。

### 数据集

我们要解决的问题是预测个人健康为目标的二元分类任务。特征是个体的社会经济和生活方式等等，标签0表示不健康，1表示健康。该数据集由美国疾病控制与预防中心收集，可以在[这里](https://www.kaggle.com/cdc/behavioral-risk-factor-surveillance-system)获得。

![](https://miro.medium.com/max/875/1*-yHB8RZnWA0rRCkQ1i3z_Q.png)

通常数据科学项目80%的时间都花在清洗、探索和从数据中提取特征上。然而对于本文我们会注重于建模（其他步骤请看[这篇文章](https://medium.com/p/c62152f39420?source=your_stories_page---------------------------)）

