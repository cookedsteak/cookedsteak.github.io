---
layout: post
title: Golang 的 routine 和 select
category: 技术
keywords: golang,concurrency,select,routine,channel,后端
comments: true
---

## 前言
go的并行模型一直是非常容易理解的，对于开发者来说难度也降低了很多。
今天在看[nebulas](https://github.com/nebulasio/go-nebulas)源码的时候发现了一些疑惑。其中就是有关于select的，所以也就乘机复习一下。

## Concurrency
中文翻译为并发，相对的还有一个 Parallelism--并行。
说实话我并不喜欢中文的翻译，因为两个“并”把原本字面和意义就不同的单词搞混了。
Rob Pike自己也说了，Golang是concurrent language，旨在用最少的成本实现模拟真实世界的并发处理机制。
Concurrency的意思就是在一段时间内，完成多个事情。类似重复利用闲置资源完成更多的事情。
![conc](/assets/img/concurrency.png)

Parallelism则是在同一时间点开始做多个事情，这个就有点像扩充生产线，解决多个任务的问题。

## GoRoutine
GoRoutine是例程，可以简单理解为协程，通常goroutine会被当做coroutine（协程）的 golang实现，但本质并不是（敲黑板）~
进程，大家不陌生，程序在执行过程中的一个实例。
线程是进程中的一个独立执行流程，是把进程分割。
而多线程则是这些被分割代码的复用。

关键字：内核实现线程-》线程的实现模式-》上下文切换（CPU时间分片）-》模式切换（用户态到内核态）

## Select









---
## 参考
- https://segmentfault.com/a/1190000006815341
- https://talks.golang.org/2012/concurrency.slide#1
- https://blog.heroku.com/concurrency_is_not_parallelism