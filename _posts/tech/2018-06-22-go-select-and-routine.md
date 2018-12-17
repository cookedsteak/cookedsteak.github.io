---
layout: post
title: Golang 的 routine
category: 技术
keywords: golang,concurrency,routine,channel,后端,并发,线程,进程
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

进程，大家不陌生，经典定义：程序在执行过程中的一个实例。
我们运行的每个程序都运行在某个进程中，进程中又有上下文（程序运行的状态记录，包括代码、数据、栈、通用目的寄存器、计数器、环境变量等）。
两个进程提供给程序的的抽象以及给到我们的假象：
1. 一个独立的逻辑控制流，让我们觉得我们在独占 CPU
2. 一个私有的地址空间，让我们觉得程序独占内存

线程是进程中的一个独立执行流程，是把进程分割。
而多线程则是这些被分割代码的复用。

#### 线程模型
用户线程模型，内核线程模型，混合模型
用户线程模型采用的是用户模式，内核线程模型就是内核模式。
这两个有啥区别？
用户模式无法执行特权指令，比如停止处理器，改变模式位，发起I/O操作。

听起来高大上的东西，我们白话理解下。（*深入理解计算机系统 P510）
首先我们要有一个系统进程，系统进程也是进程，他可以 fork
很多自己，同时每个进程又可以有多个线程，我喜欢这么去理解内核线程。

用户线程模型：就是某个用户进程下的用户线程，只会与系统进程中的某一个线程进行合作调用。这样做，线程并发用户程序自己控制（并非真正的并发），反正最后你用的也只是一个内核线程，系统他 doesn't care ~

内核线程模型：用户的某个线程直接对应系统的某个线程，1：1合作，这样用户就不用自己做异常控制流处理了，而且做到了系统级并发。但代价是开销增大，毕竟你用的是系统的线程调度资源。

混合线程模型：结合上述两个模型的有点，变化多端的合作调用方式。诶嘿，go的 runtime调度器用的就是这种模式，我们称之为 GPM模型

*关键字：内核线程-》线程的实现模式-》上下文切换（CPU时间分片）-》模式切换（用户态到内核态）-》ECF（异常控制流）

#### G-P-M 模型调度


## Goroutine Pool ?
golang 对于自己的并发还是相当自信的，你可以看到标准库中各种 go func()，
但是往往我们要结合实际业务场景，以及资源安全问题。所以一般还需要自己封装一个
goroutine pool 去处理。







---
## 参考
- http://blog.taohuawu.club/article/42
- http://guojing.me/linux-kernel-architecture/posts/progress-model/
- https://segmentfault.com/a/1190000006815341
- https://talks.golang.org/2012/concurrency.slide#1
- https://blog.heroku.com/concurrency_is_not_parallelism