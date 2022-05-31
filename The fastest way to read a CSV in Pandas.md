# 使用pandas更快的读取csv文件
[原文链接](https://pythonspeed.com/articles/pandas-read-csv-fast/)

你有一个巨大的csv文件要载入到Pandas中，但是每次载入你都要等待很久，这回拖慢你的开发节奏和产品上线速度。

但是有更快的加载数据的方式。

这篇文章涵盖了：

1. Pandas默认的csv读取
2. 更快的，在v1.4引入的并发csv读取
3. 让事情更快的另一种实现方式

## 默认的方式读取csv

通过默认方式读取一个850MB的csv文件到pandas：
```python
import pandas as pd

df = pd.read_csv("large.csv")
```

下面是时间开销，使用`time`命令计算：
```shell
$ time python default.py 

real    0m13.245s
user    0m11.808s
sys     0m1.378s
```

如果你对`time`命令的输出不熟悉，请看下[这篇文章](https://pythonspeed.com/articles/blocking-cpu-or-io/)。基本上`real`就是实际经过的时间，另外两个是按应用程序运行时间(用户)和Linux内核运行时间(sys)划分的CPU耗时。

Pandas的csv读取器有多个后端;这是用c语言写的"c"，如果我们使用"python"后端，它运行得会慢得多，但我就懒得演示了，因为太慢了。

## 使用PyArrow读取csv

在2022年1月发布的Pandas 1.4中，有一个新的csv读取后端，它依赖于Arrow库的csv解析器。它仍然被标记为实验性的，并且它不支持默认解析器的所有特性——但是它更快。

下面是如何使用：
```python
import pandas as pd

df = pd.read_csv("large.csv", engine="pyarrow")
```

当我们计时：
```shell
$ time python arrow.py 

real    0m2.707s
user    0m4.945s
sys     0m1.527s
```

两种实现的对比：

|csv解析器|运行时间|CPU时间(user+sys)|
|---|---|---|
|默认C|13.2秒|13.2秒|
|PyArrow|2.7秒|6.5秒|

我们看下CPU时间，PyArrow的实现减少了一般的耗时，提升很大。

其次，运行时间甚至更快，实际上运行时间比CPU时间要短得多。这是因为它和默认后端不同，它使用了并发，利用了处理器的多个核心。

并行性可能有好处，也可能没有，这取决于你如何运行代码。如果你以前只在单个核心上运行它，那这是一个好的性能提升。但是，如果你已经手动使用了多核心，例如通过并发加载多个CSV文件，那么在这里添加并行性不会提高速度，反而[可能会稍微降低速度](https://pythonspeed.com/articles/parallelism-slower/)。

然而，考虑到PyArrow后端本身也更快，看到总CPU时间减少了一半，它可能会提供有意义的加速，即使你已经实现手动并行。

## 重新思考问题

载入一个csv意味着很多工作：

1. 你需要分成几行
2. 你需要用逗号分割每行
3. 你需要处理字符串引号
4. 你需要猜测(!)列的数据类型，除非你显式地将它们传给Pandas
5. 你需要将字符串转换为整数，日期和其他非字符串类型

所有这些都需要CPU时间。

如果你从第三方获得一个csv文件，你只需要处理一次，你没什么可以优化的。但如果你载入一个csv文件多次？如果你是在数据处理流程中为下一步生成输入文件的人，该怎么办？

**与其读取csv文件，还不如读取处理速度更快的其他文件格式**。我们看个使用Parquet数据类型的例子（Parquet 是 Hadoop 生态圈中主流的列式存储格式，最早是由 Twitter 和 Cloudera 合作开发，2015 年 5 月从 Apache 孵化器里毕业成为 Apache 顶级项目。）。Parquet文件设计成可以被快速读取：你不需要像使用csv那样进行大量的解析。不像csv文件列类型没有被编码到文件里，Parquet文件里存储了每列数据的类型。

首先，我们将csv文件转换为Parquet文件;我们禁用了压缩，这样我们就可以和csv进行更多的对比。当然，如果你是最初生成文件的人，则不需要转换步骤，可以直接将数据写入Parquet。

```python
import pandas as pd

df = pd.read_csv("large.csv")
df.to_parquet("large.parquet", compression=None)
```

我们运行：
```shell
$ time python convert.py

real    0m18.403s
user    0m15.695s
sys     0m2.107s
```

我们读取Parquet文件;在我的电脑上，fastparquet引擎更快，但您也可以尝试pyarrow后端。

```python
import pandas as pd

df = pd.read_parquet("large.parquet", engine="fastparquet")
```

如果我们计时：
```shell
$ time python parquet.py 

real    0m2.441s
user    0m1.990s
sys     0m0.575s
```

对比下：
|解析器|运行时间|CPU时间(user+sys)|
|---|---|---|
|默认csv|13.2秒|13.2秒|
|PyArrow|2.7秒|6.5秒|
|fastparquet|2.4秒|2.6秒|

纯按CPU来衡量，fastparquet是目前最快的。它是否能提高运行时间取决于你是否写了人工并发、你的计算机性能等等。不同的csv文件可能有不同的解析成本;这只是一个例子。但显然，读取Parquet格式要有效率得多。

## 最好的csv是非csv

csv是一种糟糕的格式。除了解析它的效率低下之外，缺乏类型数据意味着解析总是比具有实际列类型的结构化文件格式更容易出错和产生歧义。因此，如果可以的话，避免使用csv，使用更好的格式，例如Parquet。



如果你无法摆脱csv，可以考虑使用Pandas 1.4中新的PyArrow csv解析器;你会得到一个很好的加速，特别是如果你的程序目前没有利用多核心。