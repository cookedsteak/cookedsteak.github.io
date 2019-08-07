---
layout: post
title: 戏说 AMQP 协议
category: 技术
keywords:  MQ,原理,队列,消息驱动,协议
comments: false
---

## 起源

AMQP 是实现 Message Queue 消息队列技术的一个重要协议。

MQ 这玩意，早在 80 年代就有了，需求呢来源于当时的金融界。
最早的一款 MQ 软件叫做 the information bus (TIB)。
TIB 之后也被电信和通讯公司采用。

之后是 IBM 开发的 MQSeries，微软开发的 MSMQ。反正这种商业闭源、收费又贵的 MQ 软件越来越多，既然 MQ 变多了，那不如就定个标准协议吧？
摩根大通和 iMatrix 就在 2004 年起草 AMQP。并且在 2006 发布了 AMQP 规范。

然后Rabbit技术公司基于AMQP标准开发的RabbitMQ 1.0，目前为止，RabbitMq 应该是AMQP最好的实现了，当然了，RabbitMq 也实现了 STOMP 和 MQTT 等协议。

那既然说道消息，我们再多说几句...

<!--more-->

### 关于消息队列

消息队列、消息调用是一种通信方式。从宏观上看，它提供了一种异步传输的通信解决方案。
相对的，同步通信是常常拿来与之比较的另一种通信方案。

所谓的同步通信和异步通信是有预先定义的，比如：

- 在计算机处理数据层面，其实都是异步的，因为CPU无法直接处理传输的数据，只能通过层层的缓冲：网卡，内存，缓存。之后才会直接处理数据。而在数据通信层面，虽然数据包都是是一帧一帧进行传输的，但是同步的通信的定义是一端的异常会导致数据包传输的堵塞等待。*所以，对同步异步的判断要看问题域的范围*。

消息队列的使用场景一般有：

- 异步处理
    通过消息中间件，提高消息处理效率，使得支持高并发的场景
- 流量控制
    通过消息中间件，履行负载均衡的功能

    在有突发或者异常访问流量的情况下，担当流量削峰、背压的安全职能
- 错误隔离
    通过消息进行应用的解耦，隔离应用错误的影响范围

好了，那接下去我们看下 AMQP 的设计架构。

## 架构

Advance Message Queue Protocol 这是 AMQP 的全称。
翻译成：高级消息队列协议。

首先，AMQP 实现了技术解耦，客户端和服务端可以实现消息的无缝通讯。

AMQP 协议约定了三个重要规范：

1. 网络协议
2. 数据消息封装
3. 代理服务的定义

![pic2](/assets/img/amqp/capabilities.png)

基本传输协议还是基于 TCP/IP，

我们这里要明白 AMQP 中的主要实体：

- 应用
- 代理
- 交换机
- 队列

应用可以理解为我们消息的两端，代理包含了交换机和队列，是中间件主体，应用定义了 AMQP 的实体代理和路由。消息通过交换机被路由到对应的队列中。

## 重要元素

我们可以看到 AMQP 重要组成元素：

![pic-1](/assets/img/amqp/amqp-1.png)

简单介绍下，流程就是 
Producer -> Broker -> Consumer

- Broker（总称）
  接受与分发消息的主体，就是一个应用，比如 rabbitmq server

- Virtual Host（抽象域）
  划分不同的 namespace 个使用同一个 rabbitmq server 的不同用户。
  用来做权限控制再好不过。

- Connection
  publisher & consumer 与 consumer 之间的 TCP 连接。断开连接的操作只会在 client 端，broker 正常情况下不会主动断开连接。

- Channel
  为了减少每次建立 TCP 连接的开销（TCP协议->TCP 协议源码->字节序->大小端->高低位）。
  其实这也是一种连接资源的复用，一般每个线程会复用同一个 channel。

- Exchange（接收消息的实体）
  message到达broker的第一站，根据分发规则，匹配查询表中的routing key，分发消息到queue中去。所以 Exchange 是一种分发规则。

- Queue（存放消息的实体）
  最终消息都在这里等待被 consumer 领走。一个 message 可以被拷贝到多个 queue 中。

- Binding（一种关系）
  Exchange 和 queue 之间的虚拟连接，binding 中包含routing key。
  Binding 信息会被 Exchange 存储到自己的表中，用来作为分发依据。

### Exchange 的几种类型

#### Direct

最直接的方式，routing key 对上了，咱就往 queue 去呗。
工作原理：

- 将一个队列绑定到某个交换机上，同时赋予该绑定一个 routing key。
- 将一个携带routing key 为 R的消息发送给 Direct Exchange，交换鸡直接路由消息给相同绑定值的队列。

直连模式可以将任务分配给多个 worker，而这个过程中负载均衡发生在 consumer 之间，而不是队列。

#### Fanout

有点像广播，所有发到 Exchange 上的 message 都会被发到所有的 queue 上去。

扇形交换机，所有消息队列不管你是啥 routing key，只要帮上他，你就会收到发送给这个交换机的所有消息，是一个典型的广播路由。

下面给出一些使用场景：

- 大规模多用户在线（MMO）游戏可以使用它来处理排行榜更新等全局事件
- 体育新闻网站可以用它来近乎实时地将比分更新分发给移动客户端
- 分发系统使用它来广播各种状态和配置更新

#### Topic

Topic 是一个更加细化的 Direct，不光是 routing key，还会有一个通配规则，比如 my.news\ your.news...，如果 topic 规则是 `#.news` 的话，就会都被发送到 news 的 queue 中。

这里我们比较下，Topic Exchange 和 Direct Exchange 有啥区别。直连交换机的消息只能被路由一次到对应的 routing key的队列中，而话题交换机中的消息按照匹配规则被路由到多个队列中。

*规则依据是 routing key and routing pattern ，我觉得这样就更容易理解了。比如我指定 routing key = "#.news" Exchange 就会根据其类型投递到对应的 queue。

#### Header

头交换机，是直连交换机的另一种形式。
不同点在于，头交换机的交换规则表现在头属性值。


## RabbitMQ

RabbitMQ 是比较常用的，实现了AMQP协议的消息队列软件。
在说 rabbitmq 之前，我想确保大家都知道，微服务通讯的两种方式。

1. P2P 同步通讯
2. Pub/Sub 异步通讯

[一个好玩的 RabbitMQ 模拟网站](http://tryrabbitmq.com/)

为什么需要 rabbitmq？因为他支持的协议很多呀。

- AMQP
- HTTP
- STOMP
- MQTT

其中 MQTT 更是物联网中最 prefer 的协议。

## 横向比较

在上面四个选项中除去 HTTP，我想将剩余的 3 个做一些和横向比较。

### 与MQTT比较

MQTT: Message Queuing Telemetry Transport，消息队列遥测传输

AMQP: Advanced Message Queuing Protocol，高级消息队列协议

所以一般在工业场景下，如果是作为数据采集端，会使用 MQTT 多一点。有时候保证只传输一次对于工业数据处理是个
十分必要的条件。

## 消息驱动

### 基于消息的分布式架构

由于消息持有双方服务规定的业务数据，在一定程度上违背了封装的要义。换言之，生产与消费消息的双方都紧耦合于消息。消息的变化会直接影响到各个服务接口的实现类。然而，为了尽可能保证接口的抽象性，我们所要处理的消息都不是强类型的，这就使得我们在编译期间很难发现因为消息内容发生变更产生的错误。

## 类型

#### 类型系统

在序列化和反序列化的过程中，
AMQP 中所支持的类型有：

```
null
bool
ubyte
ushort
uint
ulong
byte
short
int
long
float
double
decimal32/64/128
char
timestamp
uuid
binary
string
symbol
list
map
array
```

这些基本类型，差不多够从来描述大部分编程语言的数据类型，也称为【原始类型】。原文为 primitive types。
但是，一些专业领域会有一些自定义的数据类型，
用来描述本领域的专业的东西。比如在消息应用领域，
就需要在原始数据类型上进行扩展用来进行消息的传输。

那消息也是用来传递的，别人有别人的数据类型，我们有我们的数据类型，别人如果要与我们通话，
那就必须互相兼容啊，所以我们要个翻译官，AMQP 称之为描述符 discriptor。

AMQP 原始类型 + discriptor = described type (描述类型)
我们也可以认为这是 AMQP 的自定义类型系统。

一个描述类型由两种不重复的信息组成。

``` ？？？？？？？？？？？
symbolic descriptors
<domain>:<name>
numeric descriptors
(domain-id << 32) | descriptor-id
```

#### 类型编码

AMQP编码数据流由带有嵌入式构造函数的无类型字节组成，由嵌入式构造函数决定怎么翻译无类型字节。
所以，一段 AMQP 编码数据流都会以一个构造函数开头。

AMQP编码数据流把数据类型包装成了一个构造函数编码，数据流开头就先说明：“用这个构造函数去解析数据流”。

我们先不用在意传输数据流由几个大部分组成（header\payload...），但记住-我们的数据流永远是以【构造函数编码】开头的。

这里我们跳跃着看下，我们先看看 AMQP 定义的码表：

![encodelist](/assets/img/amqp/encode_list.png)
![encodelist2](/assets/img/amqp/encode_list2.png)

Code 一列就是定义的8位【原始类型】的【构造函数编码】。

```
... 0xA1 0x1E "Hello Glorious Messaging World" ...
```
这是一段简单的字符串数据流，0xA1是用来描述原始数据类型的【构造函数编码】。
`0x1E`是记录分割符，在后面就是我们的内容了。

下面是一个复杂的【构造函数编码】，他是由自描述编码组成。
```
    +-----constructor------+
    |                      |
... 0x00 0xA1 0x03 "URL" 0xA1 0x1E "http://example.org/hello-world" ...
           |_________|
                |
            descriptor
```
这个字节流传输的是一个 URL。
其中 0xA1 到 “URL” 是我们前面说的 descriptor，
一个descriptor定义了如何从原始数据转化为特定领域的数据类型。这里URL就是特定领域。

关于descriptor 这里还有一个概念是巴斯克范式（如果你知道请略过）。简称BNF。
其实BNF就是一个递归的定义说明表。举个例子，我们看一下C语言声明语句的BNF描述：
```
<声明语句> ::= <类型><标识符>; | <类型><标识符>[<数字>]; 
声明语句是由类型和标识符组成的，或者类型、标识符和数字组成的。
<类型> ::= <简单类型> | <指针类型> | <自定义类型> 
<指针类型> ::= <简单类型> * | <自定义类型> * 
<简单类型> ::= int|char|double|float|long|short|void 
<自定义类型> ::= enum<标识符>|struct<标识符>|union<标识符>|<标识符>
```
所有`<xxx>`表示这玩意还能被定义，一直到最后没有`<xxx>`为止才算完成。
就有点像逛维基百科，每个专有名词都会有另一个百科去解释，不断递归...

我们看下 constructor 的BNF：

![cbnf](/assets/img/amqp/cbnf.png)

这些范式最终终止在了AMQP 定义的码表所表示的16进制。

上图就是说，我们的构造函数编码，哪些字节在前，表示什么意思，都是有规则可寻的，
而解析规则就是上面的bnf表。

所以一旦你拿到了一段amqp数据流，不要慌，只要按照bnf一点一点对号入座，就能知道是什么类型得了。

---
> 所以到这里，我们的思维地图应该是

AMQP的数据流由两部分组成，【数据构造器】+【无类型字节】。
而【数据构造器】会根据传输数据的不同分为【原始数据类型】、【自定义数据类型】。
第一个很简单，就是原始数据类型的编码。第二个需要按照BNF去构造。
---


根据我们看到的 BNF，一个【数据构造器】会有4中不同的编码类型，分别是：
1. 定长
2. 边长
3. 复合
4. 数组


#### 定长

#### 变长

#### 复合

#### 数组


#### 复杂类型

虽然原文是 Composite Type，但是觉得翻译为 复杂类型更加贴切
为什么会有复杂类型，因为我们要定义更复杂的数据形式，比如字典。
正式情况下，AMQP里的复合类型全部通过 xml 去定义。比如：

```xml
<type class="composite" name="book" label="example composite type">
        <doc>
          <p>An example composite type.</p>
        </doc>
        <descriptor name="example:book:list" code="0x00000003:0x00000002"/>
        <field name="title" type="string" mandatory="true" label="title of the book"/>
        <field name="authors" type="string" multiple="true"/>
        <field name="isbn" type="string" label="the ISBN code for the book"/>
</type>
```

复杂类型的值表述方式是field。
field是严格按照顺序排序的，每个field都会有诸如 name，type，label，等属性。

这里有两个属性需要说一下，一个是 mandatory：

mandatory 表示一个属性是不是可以为null。

还有一个是 multiple：

multiple 用来表示该元素是不是数组。


我们现在有了复杂类型，那复合类型在数据流中又是怎么被编码的呢？
复合类型一般被编码成一个 描述列表（described list）。描述列表大概长这个亚子：

![clist](/assets/img/amqp/clist.png)


## 传输

通过AMQP组成的网络中，数据是怎么传输的呢？
我们需要定义AMQP网络中的元素，节点和连接（Nodes & Links）。

节点用来存储和发送消息，一个连接是节点之间的一个双向通道。
Links直接连接到 Nodes 的终端（Terminus）。
Terminus 的作用是监控跟踪进出数据留的状态。

那到底啥是Node?
其实 Node 也只是一个抽象的概念，他被放置在一个容器中（不是docker）。
一个容器可以放很多的 Nodes。

诸如Producers，Consumers，Queues 这些都是节点。
Producers和Consumers是在一个客户端中的概念。
Queues是在一个Broker代理中的一个概念。

那像客户端和Broker就是我们所说的容器。

![transnode](/assets/img/amqp/transnode.png)

容器与容器之间才会出现【连接】，连接中不停的传送数据（帧）。
注意，数据帧的传送是双工的，而且是强顺序的。

> 怎么个强顺序法？
如果你现在通过容器接受到了第N帧的数据，那么，必定已经收到了N-1帧的数据。

就算有很多特别的情况导致中间的传输中断，或者帧数缺失。但是AMQP依旧有保证传输可靠性的方式，之后我们再来看有哪些方法。

一个AMQP连接被有计划地分割成几个独立的无方向通道（Channels）进行通信。
在其中流转的帧会被标注好通道编号，有时候一个通道会把帧标号，放到一个子通道中继续传输。
最终呢，这些帧会被完全按照顺序合成一个单独通道的帧顺序。

容器与容器之间出现了连接，除了连接，还有【会话】（Session）。
> 一个连接可以有多个会话，那这个会话是干什么的呢？

会话保持的是通道的状态。他为消息源和消息目的地提供了一个上下文。

刚刚说到，只有容器与容器之间才会有【连接】Connection，那节点与节点之间就不能有吗？
嗯，节点与节点之间也会有，只不过那叫【联系】Link。

> 好了，现在你一定满是疑惑，连接？联系？会话？都是什么关系

其实 Connection Session Link 如同我们网络的OSI七层模型一样

![transosi](/assets/img/amqp/transosi.png)
看上图，他们都是一层套一层的。先有了连接，然后在连接上有了会话，在会话基础上有了联系。

而 Frame（帧），作为我们传输数据的最小单位，会包含一系列的操作信息。
而每个数据的操作，都是由不同的【层】去处理的。如下图：

![transosi](/assets/img/amqp/transosi2.png)

H代表处理，I代表拦截。

传输的大概我们说完了，那我们现在在深入一下，一个一个看传输的子问题。

#### 版本

在发送任何帧之前，连接端都要先发送一个协议头：协议头包含了要在现有的Connection上使用什么协议版本。

这个协议头由5部分组成：

```
__4_____1______1_______1________1____
AMQP | %d0 | major | minor | revision
```

一共是 4 + 1 + 1 + 1 + 1 = 8个字节组成。

AMQP是固定的 ASCII 编码。之后是一个0加上 主版本、服版本、修正版本号。

AMQP 中有客户端和服务端两种角色。客户端是主动打开Connection的那一端。
客户端在必须在TCP建立Session的同时发送传出协议的版本。
AMQP服务端必须应该（MAY）选择等待，直到收到了传入协议头，然后再发送传出协议头。

注意了，一个连接的两端，必须在协议头上的版本号是相同的，才能互传，否则是传不出去的。
各自憋在那里不停地收消息。

我们整理下两个AMQP节点在确认协议阶段的规则：

1. client 建立一个 socket 连接，【必须】发送一个协议头包含客户端想要的协议版本。
1. 如果请求的协议版本支持，server 端【必须】发送自己协议头回去到socket。然后按照协议规定进行后续操作。
1. 如果协议版本不支持，服务端应该发送一个支持的协议版本。然后关闭socket。
1. 当要选择一个协议进行回复，服务端应该选择请求协议的最新版本（≤）。
1. 如果服务端不能解析协议头，则服务端【必须】通过支持的协议版本发送一个合法协议头，并关闭连接。

下面是一张示意图可以参考：

![protsend](/assets/img/amqp/protsend.png)

#### 帧

在我们知道了协议版本如何发送之后，我们看下帧是如何发送的。

帧的构成：
帧由3各部分构成：
1. 一个固定的帧头部
2. 一个可变的扩展头
3. 一个可变的帧报文体

```
    必须            选择                必须
--------------------------------------------
frame header | extended header | frame body
--------------------------------------------
```




## 参考

- <https://www.cnblogs.com/frankyou/p/5283539.html>
- <http://tryrabbitmq.com/>
- <http://www.amqp.org/sites/amqp.org/files/amqp.pdf>
- <https://blog.csdn.net/u014287775/article/details/56014778>