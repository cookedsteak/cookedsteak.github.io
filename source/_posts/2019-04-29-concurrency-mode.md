---
layout: post
title: 并发模式解决方案（持）
category: 技术
keywords: 设计模式,框架,系统
comments: false
notshow: true
---

参照《7天7并发模式》

并发模式是有分类的，分为 内存共享和消息传递。

最常见的 7 个并发模型分别是：

- 线程与锁
- 函数式编程
- Clojure
- Actor
- CSP
- 数据级并行
- Lambda

每个模型所擅长解决的问题都不一样。
我们需要思考的是：

1. 这种模式是适用于并发还是并行
2. 改模型的容错性如何

## 传统方案

### 线程与锁
这个模型好比一辆老爷车，他是一个简单粗暴的方案，但是也有很多坑。
同时他也是很多复杂模型的基础。

### 监听器模式

### 函数式编程

## 非传统方案

非传统方案就是拒绝内存共享，采用非内存共享模式。
我们这里举两个典型的例子：

### Actor - akka

什么是 Actor 模式 ?
角色模型，每个计算单元都是一个单独的角色。
他们都会有一个叫做信箱的东西（消息缓存），他们会通过接收到的消息，单独做计算。
actor 与 actor 之间没有共享的资源（内存）。

当一个 actor 接受到消息后，会做下面三件事情中的一种：

- Create more actors; 创建其他actors
- Send messages to other actors; 向其他actors发送消息
- Designates what to do with the next message. 指定下一条消息到来的行为

actor 终究是一个概念模型，所以他是一种处理问题方式的抽象。

### CSP - Golang Channel

什么是 CSP 模式
CSP 模型，Communicating Sequential Processes 有序通讯过程。

golang 底层实现了非标准的 CSP 模型。有些激进的程序员甚至认为他实现的是一种反模式。

为什么这么说？
CSP模型是一种高并发计算模型，这种模型本身是试图通过一个通道同步管理发送和接受，但是，如果一旦你引入其他同步机制如互斥锁mutex、semaphore或条件判断变量，你就再也不是纯的CSP了。

那这里我们就要问了，golang 的 channel 背后到底是什么？

Go 中的 channel 使用锁保证连续访问的现成安全性，使用 channel 去同步地访问内存，其实也在使用锁。并且，这个锁在性能上还逊色与直接使用 mutex。

```shell
BenchmarkSimpleSet-8 3000000 391 ns/op
BenchmarkSimpleChannelSet-8 1000000 1699 ns/op
```

## 参考

- <https://www.jianshu.com/p/449850aa8e82>
