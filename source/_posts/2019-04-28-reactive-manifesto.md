---
layout: post
title: 我签署了反应式宣言
category: 技术
keywords: 设计模式,框架,系统
comments: false
---

## 反应式系统设计宣言

在老大的指引下，去看了下反应式宣言，然后就毫不犹豫签署了这个宣言，尽管宣言的最终版本定格在 2014 年，但仍然有很多技术人员在不断地参与进来。

那，究竟什么是反应式宣言呢？
是大家约定，用反应式方式去构建反应式系统的约定。

<!--more-->

反应式系统有如下特质：
1. 即时响应性
    不论什么响应都要快，正常响应要快，错误响应也要快。

2. 回弹性（错误边界）
    出现错误后不会导致系统崩溃，并且会将错误隔离到最小边界。
    对于出现错误的任务，还要有一定的记录机制，以防止任务丢失。

3. 弹性
    弹性是基于资源可伸缩基础之上的。
    弹性意味着当资源根据需求按比例地减少或者增加时， 系统的吞吐量将自动地向下或者向上缩放， 从而满足不同的需求。

4. 消息驱动
    充分利用 MQ，实现原本需要通过 RPC 才能实现的功能。保证松耦合，隔离，高可用，回压，智能负载等。在牺牲少许性能的同时，增加了其他方面的特性。

名词解释：

- 回压
    在某个服务组件达到处理极限的时候，有一定的机制告诉上游的组件，并降低负载，防止组件在负载下崩溃。

- 可伸缩性
    一个系统通过利用更多的计算资源来提升其性能的能力， 是通过`系统吞吐量的提升`比上`资源所增加的比值`来衡量的。一个完美的可伸缩性系统的特点是这两个数字是成正比的。 所分配的资源加倍也将使得吞吐量翻倍。
    不过这个比值函数线也会遇到一个零界点：参见 Amdahl 定律以及 Gunther 的通用可伸缩模型。

- Amdahl 定律
    阿姆达定律，这个定律是在 1967 年由 Gene Amdal 提出的，主要描述的是 串行系统并行化后的加速比计算公式和理论上限。
    > 加速比 = 优化前的系统耗时/优化后的系统耗时
    
    `SpeedUp≤1 / ( F + (1−F) / N)` 其中
    - SpeedUp：加速比 
    - F：系统内必须串行化的程序比重 
    - N：CPU处理器数量

- 消息驱动和事件驱动的区别
    消息是指发送到特定目的地的一组特定数据， 事件是组件在达到了某个给定状态时所发出的信号。
    消息驱动中，接受者等待消息的到来，否则只是休眠。接收者接收消息。
    事件驱动中，通知监听器被附加到了事件源。监听器监听事件

---
## 参考
- https://www.reactivemanifesto.org/zh-CN
- https://www.reactivemanifesto.org/zh-CN/glossary