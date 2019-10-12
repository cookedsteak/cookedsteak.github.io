---
layout: post
title: Rabbitmq Exchange 比较
category: 技术
keywords: MQ,原理,队列,消息驱动,协议
comments:  true
---

## 场景

正好有个项目需求是视频识别，因为识别的路数较多，所以采用消息队列的方式
解耦视频接收端和识别服务。简单顺序可以理解为

```
视频画面 ---> RMQ ---> 识别服务a/识别服务b/识别服务c
```

我们要比较的就是，使用 Direct Exchange 和 Topic Exchange 的区别。
以及哪个 Exchange 更快。

先给结论：Topic Exchange 快一点

- 速度的影响因素：python的循环赋值

如果是使用 Direct Exchange，我们可能会花一部分时间在数据传输至RMQ的阶段，并且，由于每个识别服务都会需要监听一个单独的队列（非抢占式消费）。所以我们是以拷贝的形式
将每一帧的数组重复推入到 RMQ 中。

## 取舍

虽然 topic 比 direct 的重复分发需求场景下稍快一点。
但是如果我的每条消息中都含有根据不同消费者而制定的消费参数的话，topic就无法适用了。

最直观的一个例子就是：

生产者的消息，对于不同的消费者，处理的工艺是不一样的，

> 虽然是同一块钢板料，但是消费者A可能要激光刻画，消费者B可能要压铸，消费者C可能要切割。只有当A、B、C都轮训了一遍后我才能算是一次总消费。

在这种情况下，Fanout Exchange我们首先排除了，因为fanout 无法指定特定的队列。
除非我们通过exchange去区分.

所以剩下 topic 和 direct。

## 测试

我们使用一张图片（大约在233KB），循环n次发送到通道，以模拟视频流的传输。

### Direct Exchange

首先是 direct exchange，该模式下有两种发送情况：
(一行为一次测试，这里都测试了五次，然后取时间平均值)

1. 一个独立队列，三个消费者

#### 逻辑
```
[]
```
#### 测试结果

![pic1](/assets/img/rmq/compare-1.png)

在生产者生产数据这一段占用了过多的时间，因为是3次重复的publish，也会消耗额外的资源。

#### 问题

向一个队列，publish了三次一样的消息，以防止消费者有没有消费到消息的情况。
但是这三次并没有指向性，所以可能出现：一个消费者消费两次，一个消费者没有消费到的情况。

我去看了ack的使用，希望能够有针对指定 consumer 的 ack 方式，的确，有一个consumer_tag，但是只要发送了 ack，默认消息就是被消费了，并不会有额外的逻辑可以留存。

2. 三个独立队列，三个消费者

#### 测试结果

![pic2](/assets/img/rmq/compare-2.png)

同样在生产者端会有较多的资源消耗，三个单独的队列，消费起来互不影响，我们可以在每个消息队列中插入特定的配置信息，也比较好做弹性化设计。

#### 问题

消息会有多份冗余，也没有过多使用到 RMQ 的特性。

### Topic Exchange

1. 三个队列，三个消费者

#### 测试结果

![pic3](/assets/img/rmq/compare-3.png)

可以看到数据的发送时间大大减少了，因为我们只需要发送一次数据，就能被匹配到三个
消费者队列中去。

#### 问题

由于消费者公用一条信息的拷贝，所以如果要根据消费者做差异化配置的话，可能需要提前将所有的配置放入到消息中。

## 超时

在进行实践的过程中，常常会遇见`104 connection reset by peer`。
避免这个错误的方法就是时刻保证client端和RMQ有心跳连接。

另外在Qos的设置上，不要将 `prefetch_count` 设置为大值，`prefetch_count` 这个设置是干什么的？

prefetch_count 是允许consumer 在接收队列中缓存的消息条数，如果达到了缓存的条数，RMQ 会停止投放，知道ack发出。
那为什么 prefetch 和 104 错误有关呢？
104的错误是连接被关闭，然后重置了，重置是服务端的操作。
如果 consumer 的 buffer 满了，RMQ试图写入consumer其他的数据的时候会被block，然后 RMQ 也会block 这个socket，直到超时，就关闭这个连接了。
如果我们把 prefetch 设置为一个很小的值，就不会造成接收buffer满出来（？），RMQ就能保证持续订阅发送消息。

但是，并不是把 prefetch 设置得越小越好。
关于合理设置 prefetch 的时间，请参考
https://www.rabbitmq.com/blog/2012/05/11/some-queuing-theory-throughput-latency-and-bandwidth/



## 连接方式

### BlockingConnection

保持阻塞除非函数回调（只有一种回调）

### BasicConnection



### SelectConnection

这种连接提供完全异步的情况，支持多种状态的回调。
比如：连接回调、取消回调、失败回调、成功回调等。

