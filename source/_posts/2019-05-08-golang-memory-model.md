---
layout: post
title: Golang 内存模型
category: 技术
keywords:  golang,go,内存,后端
comments: false
---

## 开头

为什么会看到内存模型呢？
主要是这几天在整理微服务架构的一些知识点，其中一个并发分支中有内存模型这个子分支。
所以就好奇翻看了内存模型和并发性能之间的关系。

内存模型（Memory Model），这是一个涉及高级编程语言，编译器，处理器综合性问题。

下面我们讨论的内容需要考虑并行的环境。
<!--more-->

## 我要程序跑的快

都9012年了，技术突飞猛进，编译优化技术和CPU执行优化技术提高了我们程序指令的执行速度。我们先来看看，如果我要我程序跑得快，这两种技术是怎么起到助推作用的。

### 编译优化

我们知道，我们所写的程序会被编译器编译为计算机的可执行代码，或者是解释器、虚拟机的可执行代码。
我们会在编写程序的时候就尽可能优化我们的代码，但其实我们的优化站在计算机的角度并不完全。所以
编译器会在代码基础上进行重排序（instruction reordering）。
那重排序的依据是什么呢？如何保证重排序后执行结果还是正确的呢？
其中一个依据就是数据依赖：

```c
a = 1; b = a; #1
a = 1; a = 2; #2
a = b; b = 1; #3
```

其实就是运行代码的前后依赖关系，上面的三种赋值操作，只要改变其中一个的前后顺序，得到的结果就会不一样。
所以，编译器会根据数据依赖性，来重排序。

### CPU执行优化

- 乱序执行

基本上现有 CPU 的指令运行都是乱序执行的。
什么是乱序，就是 CPU 判读你的前后两个指令并不连续相关，那么他就会议选择并行执行这两个指令。

这是保证 CPU 高效运行的基础。

- CPU缓存

CPU会有自己的缓存，缓存为了提高CPU运作执行效率。
缓存中的数据来自内存，但是，CPU 缓存与内存之间是存在时间差的，所以可能导致两遍的数据不一致。那CPU在使用缓存数据的时候有一个标准，就是只有内存和缓存中的数据一致，那才会使用缓存中的数据提高执行速度。


## 快了，然后呢

没错，我们通过编译优化和CPU执行优化提高了程序的执行速度，但是准确性呢？

我们上面的字里行间似乎没有提到内存什么事儿，但是程序的执行和数据都是存内存里来的呀。我们也知道，内存到CPU缓存中间还有一条很长的总线呢，导致可能一个进程将数据传输到CPU执行的时候，其实内存中的数据已经被另外的线程改变了，缓存中的数据不准确，数据产生了错误，计算结果也产生错误。这个就是缓存一致性的问题。

而这种错误在大并发模型的场景下更容易发生。
如果我们将缓存一致性抽象出来，可以理解为这是一个数据正确性的问题，而数据的正确性依靠3个方面：

- 原子性-CPU层面的最小操作指令，要么一下子完成，要么就不执行
- 可见性-线程对内存的操作都互相可见
- 有序性-代码按照顺序执行，或者说得到的结果必须是和代码顺序执行得到的结果相同的

所以我们需要有一个进程之间共享数据的通信模式，而进程之间的通信无非是建立在内存（共享式内存模型）或网络+内存（分布式内存模型）的基础上（这些都是计算机资源）。于是演变成了对于内存模型的设计。

## 内存模型（规范/规则）

所以这时候就要让【合理的】内存模型来解决我们遇到的问题了。
我们再来整理下我们遇到的问题：
1. 限制过度的编译优化和直连并行优化
2. 线程之间对内存的操作能够互相知晓，保证计算结果正确

好了，现在我们知道要【合理】，那到底是怎么个合理法？我们需要建立在一定的语言基础上，所以我们以golang为例，看一下golang的内存模型是怎么设计的。

*顺带说一下，比如Java也有自己的内存模型，Java 的内存模型是在java虚拟机上再模拟了一套高速缓存和内存的关系。

强烈建议参读下[官方对与内存模型的说明](https://golang.org/ref/mem#tmp_7)。

### happens-before

我们首先来理解下go原生条件下的读写顺序。

单个goroutine程序可以被编译器和处理器重新排序执行（我们之前说过了，大部分程序都是如此），但是要建立在不破坏原goroutine行为的基础上，这个基础是由该语言的语言规范制约的（就是要保证计算结果对）。

在这个基础上，导致的现象就是一个goroutine真正执行的顺序和其他程序所侦测到的顺序可能不一样。比如赋值，`a=1;b=2`，这个其实是没有明确顺序可言的。

那如果我们一定要强调顺序怎么办呢？我们就定义一个关系：happens-before。
用来描述内存操作的部分顺序。

在单个goroutine中，happens-before 等同于描述【操作声明】的先后关系。

我们这么定义 happens-before（外国佬特喜欢用否定句式去定义）:
> 事件1在事件2发生（happens-before），那么我们也可以说事件2在事件1后发生（happens-after）。
> 事件1没有在事件2之前发生、也没有在事件2之后发生，那么我们可以说事件1和事件2是同时发生的（happen concurrently）

比如：

```go
a := 5 // 1
b := a + 1  // 2
c := b * b // 3
```
我们就说1 happens-before 2。
同时，happens-before 也具有传递性。
1 happens-before 2，2 happens-before 3，所以1 happens-before 3。

但请注意，并不是说 a happens-before b ，a就在b之前执行（execute）。
执行和声明关系是不行相关的。

现在我们明确下读写的先后关系，也就三种
- 之前发生 happens-before
- 同时发生 happen-concurrently
- 之后发生 happens-after


我们假定一个共享变量v，把对变量的读记作 R(v)，写记作 W(v)。
如果满足：

1. R(v)不在W(v)前发生
2. 没有另一个 W'(v)，发生在 W(v)之后，R(v)之前

那么，R(v)肯定能够检测到W(v)的变化。（废话啦）

为了保证我们读取的结果能够正好是写入操作的结果，需要满足：

1. W(v) 在 R(v) 之前发生
2. 其他的 W'(v) 要么位于 W(v) 前，要么位于 R(v) 之后


为了确保 R(v) 一定能够侦测到 W(v)，我们要确定 W(v) 是唯一的对R(v)的可侦测写操作。

上面两种条件，在单协程的情况下，很容易理解（甚至有一点废话），而且他们是等价的。

但是我们的目标是解决高并发的场景，所以单协程对我们没有什么讨论的价值。

单协程顺序读写按照happens-before，多协程的话依靠锁和channel来保证顺序。

### channel之间的通讯

我们刚刚说了，为了保证并发执行的正确性，我们需要有线程之间的通信。
Golang 借鉴了部分 CSP 的概念也是CSP的核心概念，channel。

Golang对channel的happens-before也规定了三种情况：
1. 对一个channel的发送操作 happens-before 相应来自channel的接收操作完成
2. 关闭一个channel happens-before channel接收到最后的0返回值
3. 不带缓冲的channel接收操作 happens-before channel 发送操作完成

这里要明确下，channel的发送，原文是 `send on a channel`，写作 `c <- ?`，所以来自channel的接收是
` <- c `


其实真正使用channel的时候并没有这么复杂，我们需要关注的还是
- 有缓冲channel
- 无缓冲channel

在向无缓冲channel发送一个值的时候，该go routine 会进入blocking状态，直到有其他的routine把这个值取出来。


## 参考

- <http://valleylord.github.io/post/201606-memory-model/>
- <https://www.zhihu.com/question/36293510>
- <https://blog.csdn.net/beiyetengqing/article/details/49580559>
- <https://golang.org/ref/mem>
- <https://tiancaiamao.gitbooks.io/go-internals/content/zh/10.1.html>
- <https://www.hollischuang.com/archives/2550>
- <七天七并发模型>