# Redis Stream 介绍
[官网原文链接](https://redis.io/topics/streams-intro)

stream是Redis5.0新加入的数据类型，它定义了一种更抽象的*日志数据类型*，然而日志的本质还是完整的：就像一个日志文件一样，总是以append only的方式操作。Redis的stream就是一种append only的数据类型。至少从概念上来说，因为Redis的stream是基于内存的一种抽象数据类型，它可以突破基于硬盘的日志文件限制，实现更强大的操作。

是什么让stream成为了Redis最复杂的数据类型？尽管这种数据结构本身很简单，它实现了可选择的，无强制性的特性：一套阻塞操作允许消费者等待生产者新添加到stream的数据，除此之外引入了一个新的概念叫做**消费者组(Consumer Groups)**

消费者组最开始是在一个非常受欢迎的消息系统Kafka(TM)中被引入的。Redis用一个完全不同的实现重现了这个概念：使得一组不同的客户端可以操作同样一个数据流的不同部分。

## Stream基础

为了更好地理解Redis的stream的概念和使用，我们先忽略全部的高级特性然后专注于数据结构本身，我们按照一系列命令和操作来熟悉他。基本上，大部分性质和Redis其他数据类型一致，如List, Sets, Sorted Sets等等。然而list也有一套更加复杂的阻塞API，通过BLPOP或其他命令实现。所以stream在这方面和list没有太大区别，仅仅是这套阻塞API更加强大和复杂。

因为stream是append only的数据结构，最基本的写入命令叫做**XADD**，向指定stream中添加一个新条目(entry)。一个stream条目不仅仅是字符串，而是一个或多个键值对构成。通过这种方式，stream中每一个条目都是结构化的，就像CSV格式的追加文件一样，每行中都有多个分割开的字段。(译者：右键csv用文本文档打开，可以开到csv每行的值是通过逗号分隔的。)
```
> XADD mystream * sensor-id 1234 temperature 19.8
1518951480106-0
```
上述代码调用了XADD命令向key值为mystream的stram中添加了一个条目 sensor-id 1234 temperature 19.8，使用了一个自动生成的ID，这是命令返回的值，在这里为1518951480106-0。上述代码得到第一个参数为key的名字mystream，第二个参数为条目ID，这个ID是在每个stream内部定义的。在这个例子里我们传递\*因为我们想让服务器为我们生成一个新的ID。每个新的ID都会单调递增，所以新添加的条目的ID肯定会比之前的条目大。服务器自动生成ID是你经常要用到的，需要指定ID的情况非常少见，我们之后再谈。总的来说，每一个stream条目都有一个ID，这和日志文件很像。回到**XADD**的例子，在key的名字和ID之后，下一个参数是我们要写入的键值对。
通过**XLEN**命令来得到一个stream中的条目总数：
```
> XLEN mystream
(integer) 1
```
## 条目ID
**XADD**命令返回条目ID，该ID可以精准识别stream中的每一个条目，它由两部分组成：
```
<millisecondsTime>-<sequenceNumber>
```
毫秒(millisecondsTime)部分是Redis节点本地的时钟时间，然而如果现在的毫秒时间小于之前的条目时间，那么会使用之前的条目时间代替，所以如果时钟向前回调的话，单调递增的ID仍然会生效。序列号用于在相同毫秒内创建的条目。由于序列号是64位，所以在实际应用中，在相同毫秒内生成的条目没有数量限制。

刚开始看你可能会感觉这种ID格式很奇怪，你也可能会为为什么时间是ID的一部分而感到困惑。（译者：当你熟练掌握后你就会感觉这种数据结构强的一比！（破音））原因是Redis stream支持通过ID进行范围查询。因为ID是可以和条目生成的时间相关联，这就让stream有了基于时间范围查询的能力。我们过会会看到通过**XRANGE**命令。

出于某些原因，用户不想使用基于时间的ID也是可以的。**XADD**命令可以使用明确的ID来代替\*，就像下面的例子一样：
```
> XADD somestream 0-1 field value
0-1
> XADD somestream 0-2 foo bar
0-2
```
在这个例子中，最小的ID为0-1并且这个命令不会接受小于或等于之前ID的值：
```
> XADD somestream 0-1 foo bar
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
```

## 从Stream中获取数据
现在我们可以通过**XADD**向stream中添加数据了。向stream中添加数据看起来很清晰很明确，然而stream提取数据的方式看起来并不是很清晰了。如果我们继续类比日志文件的话，比较清晰的表达方式是我们模仿Unix命令tail -f，这样我们可以开始监听新添加到stream中的信息。注意和Redis的list阻塞操作不同，list阻塞操作只允许一个客户端获取到一个元素，通过阻塞的*pop类型*操作例如**BLPOP**，在stream中全部的消费者都可以看到新添加进stream中的信息，就像很多个tail -f进程一起看日志里写入了什么一样。用专业术语来说我们想让stream可以*扇出(fan out)*到多个客户端。

然而这只是一种潜在的访问模式。我们还可以这样看待stream：不作为一个消息系统，而作为*时间序列存储*。这样可能有助于获取到新append进来的信息，另一种更自然的查询模式是通过时间序列查询信息，或者说使用一个游标来迭代全部历史信息。这是另一种有效的访问模式。

最后，如果我们以消费者的视角来看stream，我们可能想通过另外一种方式访问stream，作为消息流可以将其划分给处理此消息的多个消费者，所以各个组的消费者只能看到消息流中的一个子集。这样就可以将消息划分给不同的消费者，不再需要单个消费者处理全部的消息：每个消费者仅仅得到需要处理的信息。这就是Kafka(TM)使用消费者组实现的。通过消费者组读取消息是从Redis stream中读取信息的一种有趣的模式。

Redis stream使用不同命令来支持上述三种查询模式。下一章会全部展示这三种方法，让我们从最简单最直接的方式开始：范围查询。(range queries)

## 通过范围查询：XRANGE和XREVRANGE
通过范围查询stream我们需要指定两个ID，*起始ID和结束ID*。返回会包含起始ID和结束ID之间的信息。两个特殊的ID-和+代表最小和最大的ID。
```
> XRANGE mystream - +
1) 1) 1518951480106-0
   2) 1) "sensor-id"
      2) "1234"
      3) "temperature"
      4) "19.8"
2) 1) 1518951482479-0
   2) 1) "sensor-id"
      2) "9999"
      3) "temperature"
      4) "18.2"
```
返回的每个条目是包含两项的数组：ID和键值对的list。我们已经知道ID是和时间关联的，因为左边的部分是stream条目创建时的本地节点的Unix毫秒时间，在那一刻条目被创建（注意stream通过XADD命令被复制，所以集群的slaves对于master有可识别的ID）。这意味着我可以使用**XRANGE**来查询一个时间段。为了这样做，我想省略掉ID的序列部分：如果这个字段被省略，range的起始部分假设序列为0，在末尾会假设序列ID为可以取的最大值。使用两个时间来进行查询，我们能得到在这个时间段内生成的条目。举个例子，我想查询两个时间点之间写入的值：
```
> XRANGE mystream 1518951480106 1518951480107
1) 1) 1518951480106-0
   2) 1) "sensor-id"
      2) "1234"
      3) "temperature"
      4) "19.8"
```
这里我只有一条数据，然而在真实数据集中我可以查询几个小时的范围内写入的值，这个数据体量会很大。因为数据量可能会很大，所以**XRANGE**支持一个可选的参数**COUNT**。通过制定一个count，你可以得到前N条数据。如果我想获取更多，我可以取出上一次返回值的最后的ID然后将其作为起始时间继续查询。我们看下面的例子来实现这个功能。我们先使用**XADD**命令添加十条数据（这步自己做，mystream这个stream确保有十条数据）。现在启动迭代，每次命令取出两条数据，我先从全部范围开始查询，但是count设为2。
```
> XRANGE mystream - + COUNT 2
1) 1) 1519073278252-0
   2) 1) "foo"
      2) "value_1"
2) 1) 1519073279157-0
   2) 1) "foo"
      2) "value_2"
```
**XRANGE**的复杂度是O(log(N))，然后O(M)返回M个元素，当count的值较小时这个命令的时间复杂度是对数，这意味着每一步迭代都非常快。所以**XRANGE**实际上就是stream的迭代器并不需要**XSCAN**命令。

**XREVRANGE**功能等同于**XRANGE**但是返回的元素是倒序的，所以经常使用**XREVRANGE**来查看stream中最后的元素是什么：
```
> XREVRANGE mystream + - COUNT 1
1) 1) 1519073287312-0
   2) 1) "foo"
      2) "value_10"
```
请注意**XREVRANGE**命令的起始和结束位参数也要是倒序的。

## 使用**XREAD**命令来监听新的数据

当我们不想通过范围来访问stream中的数据时，通常来说我们*订阅*到达stream中的新数据。这个概念与Redis Pub/Sub有关联，当你订阅了一个channel或者Redis阻塞list，你等待一个key来获取新元素，但是你消费stream时会有几个本质的区别：

1. stream可以有多个等待数据的客户端（消费者）。每次新的数据达到，默认情况下会被分发到这个stream中等待消息的*全部消费者*。该行为和阻塞list不同，阻塞list的每个消费者会收到不同的数据。然鹅*扇出*到多个消费者与Pub/Sub很像。
2. 就像Pub/Sub中消息的fire和forget和永不存储一样，使用阻塞list时当消息被客户端poped接收后会从list中删除，stream的工作方式完全不同。所有的消息被无限地添加进steam中（除非用户明确要删除数据）：不同的消费者通过记住最后收到的消息ID来识别那些新的消息。
3. stream消费者组提供了一个Pub/Sub或者阻塞list无法触及的控制水平，在同一stream中不同的消费组，显式地确认已经处理的数据，检查待处理的数据的能力，声明不处理的消息，以及每个客户端拥有一致的历史可见性，只能查看自己私有的消息历史。

**XREAD**命令提供监听到达stream的新消息的能力。它比**XRANGE**复杂一点，所以我们先从简单的形式开始，之后会介绍整个命令的格式。
```
> XREAD COUNT 2 STREAMS mystream 0
1) 1) "mystream"
   2) 1) 1) 1519073278252-0
         2) 1) "foo"
            2) "value_1"
      2) 1) 1519073279157-0
         2) 1) "foo"
            2) "value_2"
```
上述命令是**XREAD**的非阻塞格式。**COUNT**是可选的，唯一强制参数是STREAMS，他指定一个键列表，以及一个消费者在steam中的ID以便客户端只接收那些拥有更大ID的消息。

上述命令我们写入STREAMS mystream 0所以我们想要在mystream中ID比0-0大的信息。上述命令返回了key的名字，这是因为实际上可以调用这个命令同时从超过一个stream中读取数据。举个例子，可以这样写：STREAMS mystream otherstream 0 0。需要注意的是在STREAMS选项后我们要提供key的名字和ID。所以STREAMS选项必须永远在命令的末尾。

我们可以指定末尾ID来获取新的消息，同时**XREAD**可以同时访问多个stream，除此之外看起来和**XRANGE**命令没什么不同。但是有趣的部分是我们可以轻松将**XREAD**转换为*阻塞命令*，通过指定**BLOCK**参数：
```
> XREAD BLOCK 0 STREAMS mystream $
```
在上述例子中，我们删掉了**COUNT**同时制定了一个新参数**BLOCK**将超时参数设为0.除此之外，我传递了一个特殊ID $ 到正常ID mystream的后面。这个特殊ID意思是**XREAD**应该使用现在stream中存储的最大ID作为起始ID，所以我们这回只会收到从监听开始后新的消息了。这和Unix命令 tail -f 很相似。

当我们使用**BLOCK**命令时不非得使用特殊ID $，我们可以使用任何有效ID。当命令可以马上处理我们的请求时它不会阻塞，否则就会阻塞。通常来讲如果我们想消费新产生的数据时使用$，然后我们记下这个ID用来再次调用。

阻塞**XREAD**也可以监听多个stream。如果至少一个steam中的元素大于你指定的ID，这个命令就会同步返回结果。要不然这个命令会阻塞到至少从一个stream中获取到数据。

和阻塞list相似的是stream也是FIFO风格的。最先阻塞并监听steam的客户端会最先得到新的数据解除阻塞。

**XREAD**只有**COUNT**和**BLOCK**两个参数，这是一个非常基本的命令。可以使用消费者组API来使用更多更强大的特性，但是使用消费者组是基于**XREADGROUP**命令的，下一章会讲到。

## 消费者组(Consumer groups)
当手头的工作是让不同的客户端消费同一个stream时，**XREAD**已经提供了将消息*扇出*到N个客户端的方法，可能你会用到slave提高可扩展性。但是目前的问题是我们不想将一个stream中相同的消息分发给多客户端，而是将stream中消息的子集分发给指定的客户端。

现在假设我们有三个消费者C1,C2,C3，有一个stream包含消息1,2,3,4,5,6,7，然后我们想按下面的图表处理这些消息：
```
1 -> C1
2 -> C2
3 -> C3
4 -> C1
5 -> C2
6 -> C3
7 -> C1
```
Redis使用*消费者组*的概念来实现这个功能。

一个消费者组像一个*pseudo consumer*一样从stream中获取数据，实际上服务于多个消费者，提供如下保证：

1. 每条信息会被不同的消费者消费，绝对不会将相同数据分发到多个消费者。
2. 消费者通过一个必须指定的区分大小写的名字划分进消费者组。这意味着即使断开连接后，消费者组也会保持自身状态，直到客户端再次声明成为消费者。这也意味着这是每个客户端的唯一标识符。
3. 每个消费者组有一个*first ID never consumed*的概念，当消费者请求新的信息时，它可以提供之前从未分发过得消息。
4. 使用消息需要特定的命令显式确认：此消息已正确处理，因此可以从消费者组中驱逐(evicted)。
5. 消费者组跟踪所有目前分发的消息，这个意思是消息被分发给一些组中的消费者，但是还没有被确认为已处理。感谢这个特性，当访问stream的消息历史时，每个消费者*只会看到分发到它的消息*。

某种意义上来说，一个消费者组可以被认为是stream的某种状态：
```
+----------------------------------------+
| consumer_group_name: mygroup           |
| consumer_group_stream: somekey         |
| last_delivered_id: 1292309234234-92    |
|                                        |
| consumers:                             |
|    "consumer-1" with pending messages  |
|       1292309234234-4                  |
|       1292309234232-8                  |
|    "consumer-42" with pending messages |
|       ... (and so forth)               |
+----------------------------------------+
```
如果你看到这个从这个角度来看,很容易理解消费者组可以做什么,它是如何能够为消费者提供他们的历史等待消息,以及消费者如何要求新消息只会配上消息id大于last_delivered_id。与此同时，如果您将使用者组看作是Redis流的辅助数据结构，那么很明显，单个流可以有多个使用者组，它们有不同的使用者组。实际上，同一个流甚至可以让客户端不通过XREAD读取组，而让客户端在不同的消费组中通过XREADGROUP读取。（机翻 看不懂）

基本的消费者组命令如下：
- **XGROUP** 用来创建、销毁和管理消费者组
- **XREADGROUP** 通过消费者组从stream中读取
- **XACK** 允许消费者将接收到的消息标记为已处理

## 创建一个消费者组

假设现在有一个key为mystream的stream，如下命令创建一个消费者组：
```
> XGROUP CREATE mystream mygroup $
OK
```
注意：目前不支持为不存在的stream创建消费者组，然而不久后我们会为**XGROUP**命令添加可选的参数来创建一个空的stream。

上述命令我们必须指定一个ID来创建一个消费者组，这里使用的$。这是因为在消费者组创建的时候，它必须知道第一个消费者连接上时要serve什么内容。如果我们使用$的话，那么只有现在开始新产生的消息将会被推给组里的消费者。如果我们使用()的话消费者组会消费掉这个stream历史中全部的消息。当然，你也可以指定其他有效的ID。你需要明白的是消费者组会根据你选择的ID来分发比该ID更大的消息。因为$代表当前stream中最大的ID，选择$的效果是消费新的消息。

现在消费者组创建好了我们可以马上通过消费者组来读取消息，使用**XREADGROUP**命令。我们通过两个叫做Alice和Bob的消费者来看看系统如何返回不同的信息。

**XREADGROUP**和**XREAD**很相似都提供**BLOCK**操作，同时这也是一个同步命令。这里有一个必须要指定的强制选项叫做**GROUP**，它有两个参数：消费者组的名字和想要读取的消费者名字。**COUNT**选项也是支持的，用法和**XREAD**相同。

在开始读取之前，我们先放一点数据进去：
```
> XADD mystream * message apple
1526569495631-0
> XADD mystream * message orange
1526569498055-0
> XADD mystream * message strawberry
1526569506935-0
> XADD mystream * message apricot
1526569535168-0
> XADD mystream * message banana
1526569544280-0
```
注意：这里message是键，各种水果是键值，记住stream的每条数据就像一个小型字典。

是时候尝试使用消费者组读取一些东西了：
```
> XREADGROUP GROUP mygroup Alice COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) 1526569495631-0
         2) 1) "message"
            2) "apple"
```
**XREADGROUP**的回显和**XREAD**很像。注意GROUP \<group-name> \<consumer-name> 这个陈述，它指定了我用mygroup消费组并且我是消费者Alice来从stream中读取数据。当某个消费组内的消费者要执行某些操作时，它必须在组内指定唯一的名字用来标识消费者。

上述命令还有一个非常重要的细节，在强制性参数**STREAMS**后键mystream需求的ID是特殊ID>。这个特殊ID只在消费组上下文中有效，它的意思是：**目前为止还没有分发给其他消费者的消息。**

上述操作方式一般是你常用到的，同样你也可以指定真实的ID比如()或者其他有效ID，在这个例子里我们请求**XREADGROUP**提供**挂起消息的历史记录**，在上述例子中不会看到新的消息。所以基于我们选择的ID，**XREADGROUP**会有如下不同的行为：

- 如果ID为特殊ID > 那么命令会返回从没有被分发过得新的消息然后更新消费组的最后使用ID
- 如果ID是任何数值的话，命令会访问*挂起消息的历史记录(history of pending messages)*。这是那些已经分发给指定消费者，但是还没有被**XACK**确认的消息。

我们可以将ID设为0来确认这个行为，去掉**COUNT**参数：我们只看挂起消息，这里有一条apple的信息：
```
> XREADGROUP GROUP mygroup Alice STREAMS mystream 0
1) 1) "mystream"
   2) 1) 1) 1526569495631-0
         2) 1) "message"
            2) "apple"
```

如果我们确认这条消息被处理，那么它就不会再出现在挂起消息历史中，所以系统再也不会报告任何东西了：
```
> XACK mystream mygroup 1526569495631-0
(integer) 1
> XREADGROUP GROUP mygroup Alice STREAMS mystream 0
1) 1) "mystream"
   2) (empty list or set)
```

如果你不知道**XACK**是如何运作的也不用慌，这个概念仅仅是处理过的信息不再是我们可以访问的历史信息。

现在轮到Bob来读取一些东西了：
```
> XREADGROUP GROUP mygroup Bob COUNT 2 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) 1526569498055-0
         2) 1) "message"
            2) "orange"
      2) 1) 1526569506935-0
         2) 1) "message"
            2) "strawberry"
```
Bob通过同样的消费组mygroup请求最多两条消息。这里Redis就是发送新的消息。正如你看到的，'apple'是不会被分发的，因为它已经被分发给Alice了，所以Bob得到了orange和strawberry。

这样Alice，Bob和其他同组的消费者就可以从同一个stream中读取不同的消息，读取他们还没有处理的消息历史或者将其标记为已处理。这样就可以创建不同的拓扑和语义来使用stream了。

记住这样几件事：

- 消费者在第一次提到他们的时候自动创建，不需要单独创建。
- 你也可以用**XREADGROUP**从多个key上读取数据，要实现这个目的你需要在每个stream中创建同名的消费组。这个一般不常用到，但值得一提的是在技术上是可行的。
- **XREADGROUP**是一个*写入命令*尽管它从stream读取数据，作为读取数据的附带效果消费组被修改了，所以它只能在主实例中被调用。

下面展示一段用Ruby写的使用消费组的例子。Ruby代码的编写方式几乎可以让任何有经验的使用其他语言进行编程的程序员都能读懂:
```
require 'redis'

if ARGV.length == 0
    puts "Please specify a consumer name"
    exit 1
end

ConsumerName = ARGV[0]
GroupName = "mygroup"
r = Redis.new

def process_message(id,msg)
    puts "[#{ConsumerName}] #{id} = #{msg.inspect}"
end

$lastid = '0-0'

puts "Consumer #{ConsumerName} starting..."
check_backlog = true
while true
    # Pick the ID based on the iteration: the first time we want to
    # read our pending messages, in case we crashed and are recovering.
    # Once we consumer our history, we can start getting new messages.
    if check_backlog
        myid = $lastid
    else
        myid = '>'
    end

    items = r.xreadgroup('GROUP',GroupName,ConsumerName,'BLOCK','2000','COUNT','10','STREAMS',:my_stream_key,myid)

    if items == nil
        puts "Timeout!"
        next
    end

    # If we receive an empty reply, it means we were consuming our history
    # and that the history is now empty. Let's start to consume new messages.
    check_backlog = false if items[0][1].length == 0

    items[0][1].each{|i|
        id,fields = i

        # Process the message
        process_message(id,fields)

        # Acknowledge the message as processed
        r.xack(:my_stream_key,GroupName,id)

        $lastid = id
    }
end
```
正如你看到的一样这里开始消费历史信息，就是我们的挂起信息。因为消费者可能之前崩溃过，这么做就很有用，所以在重启之后我们先重新读一遍分发给我们但是我们没有进行处理确认的消息。这样我们可以一次或多次处理相同信息。（至少在消费者失效的情况下是这样的，但是也可能是Redis持久化和应答的问题，请查阅关于该主题的特定章节）。

一旦我们消费完历史消息，我们就得到一个空的消息list，然后我们设置ID为>开始消费新的消息。

## 从永久性故障中恢复

上面的示例允许我们编写同消费组的消费者，处理每个消息的子集，在故障后重新读取分发来的消息。然而真实环境下消费者可能会永久性故障然后不可恢复了。如果消费者的挂起消息因为某些原因停止并且再也无法恢复，会发生什么情况？

Redis消费组适用于这种情况下的特性，声明一个指定的消费者然后将该消费者挂起的信息转移到其他消费者名下。

这个过程的第一步使用**XPENDING**命令来可视化消费组中的挂起实例。这是一条只读命令，可以在任何时候被安全调用，不会改变任何消息的所有者。简单格式的话，这条命令需要两个参数stream的名字和消费组的名字。
```
> XPENDING mystream mygroup
1) (integer) 2
2) 1526569498055-0
3) 1526569506935-0
4) 1) 1) "Bob"
      2) "2"
```
这条命令输出在该消息组中的挂起消息总数，这里是2，最小和最大的挂起消息的ID，最后是一个消费者的列表以及他们持有的挂起消息数。这里只有Bob有两条挂起消息因为Alice的消息都已经用**XACK**确认。

我们可以给定**XPENDING**更多的参数来获取更多的信息，完整命令格式如下：
```
XPENDING <key> <groupname> [<start-id> <end-id> <count> [<conusmer-name>]]
```
通过起始和结束ID参数以及count参数，我们就可以更好地操作挂起信息。最后的可选参数消费组名字是用来限制命令只返回某个消费组的挂起信息，下面的例子我们不会使用这个特性。
```
> XPENDING mystream mygroup - + 10
1) 1) 1526569498055-0
   2) "Bob"
   3) (integer) 74170458
   4) (integer) 1
2) 1) 1526569506935-0
   2) "Bob"
   3) (integer) 74170458
   4) (integer) 1
```
现在我们得到了消息的具体内容：ID，消费者名字，毫秒格式的空闲时间，这个意思是把消息分发给消费者之后经过的时间，最后一个数字是消息被分发的次数。我们有两条Bob的消息，他们空闲了74170458毫秒，差不多20个小时。

使用**XRANGE**命令，没人可以阻止我们查看第一个消息的内容。
```
> XRANGE mystream 1526569498055-0 1526569498055-0
1) 1) 1526569498055-0
   2) 1) "message"
      2) "orange"
```
我们只是在参数中重复了两次同一个ID。现在有这样一个想法，Alice决定在20小时内如果Bob没有处理任何消息，可能Bob已经不能及时恢复了，所以是时候*声明(claim)*这些消息并且代替Bob处理了。为了这么做，我们使用**XCLAIM**命令。

这个命令有非常多的参数十分复杂，这里我们只使用我们通常用到的参数。在这个例子里进行如下简单调用：
```
XCLAIM <key> <group> <consumer> <min-idle-time> <ID-1> <ID-2> ... <ID-N>
```
基本来说，我们指定一个stream的key和一个消费组，我想让指定ID的消息变更所有者到指定的\<consumer>名下。然而我们也提供一个最小空闲时间，这使这个命令只会生效于消息实际空闲时间大于指定值的消息条目。因为可能会有两个客户端同时尝试对同一消息的声明：
```
Client 1: XCLAIM mystream mygroup Alice 3600000 1526569498055-0
Clinet 2: XCLAIM mystream mygroup Lora 3600000 1526569498055-0
```
声明一条消息的同时会重置它的空闲时间。并且会增加分发计数，所以第二个客户端就会声明失败。这样就避免了消息的重复处理。
这是命令执行的结果：
```
> XCLAIM mystream mygroup Alice 3600000 1526569498055-0
1) 1) 1526569498055-0
   2) 1) "message"
      2) "orange"
```
消息成功被Alice声明，现在可以正常处理和确认它了，一切正常运作即使原始消费者再也不能恢复运作了。

当**XCLAIM**命令成功声明的同时也会返回这条数据的内容。使用**JUSTID**参数可以让命令只返回成功的ID。这对降低服务器和客户端的带宽非常有用，而且你也不会对生命的消息内容感兴趣，因为之后消费者会重新扫描挂起历史来处理这些消息。

声明机制也可以由分离的进程实现：一个进程仅仅检查挂起消息列表，然后将空闲消息分配给活跃的消费者。活跃消费者可以通过Reids stream的可观察特性来获取消息。这是下一章的主题。

## 声明和分发计数器(Claiming and the delivery counter)
你在**XPENDING**命令的输出里观察到的计数值是每个消息的传递次数。该计数通过两种方式增加：当消息成功被**XCLAIM**命令声明或者被**XREADGROUP**调用访问挂起消息的历史时。

当上文的故障发生时，消息可能会被多次分发，但最终都会被处理。然而当处理特定的消息时代码因为bug发生错误崩溃就会产生一个问题。这种情况下其他的消费者会声明这条消息然后也会处理失败。因为我们有一个分发计数器，我们可以使用它来侦测这种情况。一旦在分发计数器中找到一个比你设定的值大的数，就把这条消息放进其他stream然后向系统管理员发一个通知。这是Redis*死信队列(dead letter)*的基本实现概念。

## Stream可观测性(Streams observabilty)
你很难和缺乏可观测性的消息系统工作。不知道谁在消费消息，那些消息被挂起，哪些stream中的哪些消费组在工作，全部事物缺乏透明度。出于这个原因，Redis stream和消费组使用了不同的方式进行可视化。我们已经介绍了**XPENDING**，它允许我们检查消息队列，显示空闲时间和分发次数。

现在我们想更进一步，**XINFO**命令是一个可视化接口，可以用来显示stream和消费组的信息。

这个命令使用一些子命令来显示stream和消费组的不同状态信息。用**XINFO STREAM**显示stream自身信息。
```
> XINFO STREAM mystream
 1) length
 2) (integer) 13
 3) radix-tree-keys
 4) (integer) 1
 5) radix-tree-nodes
 6) (integer) 2
 7) groups
 8) (integer) 2
 9) first-entry
10) 1) 1524494395530-0
    2) 1) "a"
       2) "1"
       3) "b"
       4) "2"
11) last-entry
12) 1) 1526569544280-0
    2) 1) "message"
       2) "banana"
```
输出显示了stream内部编码信息，第一个和最后一个消息。另一个信息是与当前stream关联的消费组数量。我们可以接着挖关于消费组的更多信息。
```
> XINFO GROUPS mystream
1) 1) name
   2) "mygroup"
   3) consumers
   4) (integer) 2
   5) pending
   6) (integer) 2
2) 1) name
   2) "some-other-group"
   3) consumers
   4) (integer) 1
   5) pending
   6) (integer) 0
```
正如你看到的，**XINFO**输出了一系列键值对。在今后我们会在保证旧客户端的兼容性的情况下输出更多的信息。当然其他的命令会更有效的理由带宽，**XPENDING**命令仅仅回显不带键的信息。

上述例子中使用了**GROUPS**子命令，我们可以输入消费组名来检查消费者的详细状态。
```
> XINFO CONSUMERS mystream mygroup
1) 1) name
   2) "Alice"
   3) pending
   4) (integer) 1
   5) idle
   6) (integer) 9104628
2) 1) name
   2) "Bob"
   3) pending
   4) (integer) 1
   5) idle
   6) (integer) 83841983
```
如果你忘了这个命令的语法，使用下面的帮助命令查看：
```
> XINFO HELP
1) XINFO <subcommand> arg arg ... arg. Subcommands are:
2) CONSUMERS <key> <groupname>  -- Show consumer groups of group <groupname>.
3) GROUPS <key>                 -- Show the stream consumer groups.
4) STREAM <key>                 -- Show information about the stream.
5) HELP                         -- Print this help.
```
## 与Kafka(TM)的区别
（我不知道Kafka是什么）

## Capped Streams
很多应用不想往stream中一直存数据。这时给stream内设置一个最大容量值就很有用，它可以将数据从内存转移到硬盘存储，虽然读取速度会下降但是很适合需要保存的历史数据。Redis stream使用**XADD**命令的**MAXLEN**参数来实现这个功能：
```
> XADD mystream MAXLEN 2 * value 1
1526654998691-0
> XADD mystream MAXLEN 2 * value 2
1526654999635-0
> XADD mystream MAXLEN 2 * value 3
1526655000369-0
> XLEN mystream
(integer) 2
> XRANGE mystream - +
1) 1) 1526654999635-0
   2) 1) "value"
      2) "2"
2) 1) 1526655000369-0
   2) 1) "value"
      2) "3"
```
使用**MAXLEN**当数据量达到指定的长度后最旧的数据会被驱逐，所以steam中的数据一致保持一个常量。目前我们没有指定命令的参数，只告诉stream只保留不超过给定数量的值，为了持续运行，这样的命令可能会阻塞很长一段时间，以驱逐旧的消息。例如，假设有一个插入尖峰，然后是一个长暂停，然后是另一个插入，所有这些都是相同的最大时间。stream将阻塞以驱逐暂停期间变得太旧的数据。因此，由用户来做一些计划，并了解所需的最大stream容量是多少。此外,stream的容量与内存使用成正比,调整时间不太简单的控制和预测:这取决于插入率通常是一个变量改变随着时间的推移,(当它不改变,那么就倾大小并不重要)。（部分机翻 讲的是MAXLEN可能会阻塞很久的机制）

然而，使用MAXLEN进行微调代价会很大:stream由宏节点表示为基数树，以便非常高效地使用内存。改变由几十个元素组成的单个宏节点并不是最佳选择。因此，可以用以下特殊形式的命令:
```
XADD mystream MAXLEN ~ 1000 * ... entry fields here ...
```
在**MAXLEN**和实数中的~意思是我不需要最大容量等于1000。它可以是1000或者1010或者1030，只是确保最少存1000个数据。有了这个参数，只有当我们删除整个节点时，才会执行微调。这使它更有效率，这通常是你需要的。

还有一个可用的XTRIM命令，它执行的操作与上面的MAXLEN选项非常类似，但是这个命令不需要添加任何内容，可以在任何单独运行的stream上执行。
```
> XTRIM mystream MAXLEN 10
```
或者：
```
> XTRIM mystream MAXLEN ~ 10
```
然而，XTRIM的设计是为了接受不同的微调策略，即使目前只实现MAXLEN。假设这是一个显式命令，那么将来它可能允许按时间微调，因为以独立方式调用该命令的用户应该知道自己在做什么。

XTRIM应该使用的一个有用的回收策略能够删除一系列id。目前该功能还没有实现的，但是将来可能会实现，以便更容易地使用XRANGE和XTRIM来将数据从Redis移动到其他存储系统(如果需要的话)。
## 持久化，复制和消息安全
stream和其他Redis数据结构一样，同步复制到slave持久化到AOF或者RDB文件。不那么明显的是消费组的完整状态也会被传递到AOF,RDB和slave中，所以在master中挂起的消息在slave中也会存在。同样的，在重启之后AOF会重新储存消费组状态。

记住Redis stream和消费组使用Reids默认复制规则，所以：
- 如果持久化的数据对你的应用很重要，那么AOF必须使用强同步策略
- 默认情况下同步复制不会保证**XADD**命令或消费组状态被复制：故障转移后，根据slave从主服务器接收数据的能力，可能会丢失一些内容。
- 可以使用**WAIT**命令将变更传播给一组slave。虽然这使数据丢失的可能性很小，但在某些特定情况下会丢失数据。

所以当使用stream和消费组来设计应用时，确保你的应用理解这些语义以及可以处理故障，评估应用处理数据是否足够安全。

## 从stream中删除单条数据

stream有一个命令通过指定ID从stream“中间”删除数据。通常来说一个append only的数据结构提供这个看起来很奇怪，但是对应用来讲用处很大，比如隐私条例。命令为**XDEL**，需要提供stream的名字和要删除的ID：
```
> XRANGE mystream - + COUNT 2
1) 1) 1526654999635-0
   2) 1) "value"
      2) "2"
2) 1) 1526655000369-0
   2) 1) "value"
      2) "3"
> XDEL mystream 1526654999635-0
(integer) 1
> XRANGE mystream - + COUNT 2
1) 1) 1526655000369-0
   2) 1) "value"
      2) "3"
```
但是在当前的实现中，直到一个宏节点完全为空，内存才真正被回收，所以您不应该滥用这个特性。

## 空stream
stream和其他Redis数据结构的区别在于，当其他数据结构不再具有元素时，作为调用删除元素的命令的副效果，键本身将被删除。例如,一组排序将被完全移除时调用**ZREM**将删除最后一个元素排序集。而stream元素为零时被允许存在,可能由于使用**MAXLEN**将容量设为为0(**XADD**和**XTRIM**命令),或者调用了**XDEL**。

之所以存在这种不对称，是因为stream可能和消费组关联，我们不希望仅仅因为stream中不再有数据而失去消费组的状态。目前，即使没有关联的使用者组，stream也不会被删除，但这在将来可能会改变。
