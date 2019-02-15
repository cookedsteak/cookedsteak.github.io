---
layout: post
title: Golang Quiz
category: 技术
keywords: golang,interface,go,框架,后端
comments: false
---

这里记录些关于 golang 的有趣问题

## 实现接口的到底是谁
今天在网上看到一个问题，关于 golang 的 interface 实现的问题。

```
package main

import "fmt"

type Gulu interface{
    Goo() string
}

type Mana struct{}

func (m *Mana)Goo() string {
    return "Goo~~"
}

func main() {
    var a Gulu = Mana{}
    fmt.Println(a.Goo())
}
```
问题来了，上面这段代码正确吗？

既然这么问了，那肯定有问题啊。
大家对于 golang 接口的实现并不陌生，非侵入式，只要实现了接口的方法就会认为继承自该接口。
那实现接口方法的对象有没有要求呢？这个就是上面代码无法通过编译的原因了。

`var a Gulu = Mana{}`这句话意味着，
Mana结构体是实现了接口 Gulu。

但其实，真正实现这个接口的不是 Mana 而是 *Mana。
所以，如果我们改成`var a Gulu = &Mana{}`就对了。

## slice：我怕 array 太寂寞
slice 是一个十分方便的数据结构。我一直认为 slice 是 array 的升级版本。
的确是这样，slice 中传递的不再是真正的值，他给 array 套了一层扩展骨架，同时变成了引用传递的方式进行骨架的操作。


## 为啥要 make
我们一直知道，golang 中有些数据类型需要先 make，slice，map，chan。但是有没有想过为什么要 make 呢？

我们循着代码的踪迹找去，发现了 runtime 里的


## 线程安全
不要被这四个字吓到了，其实就是公用资源的锁问题。
线程安全这个词中，线程限定了范围，给出了程序运行环境：说明是一个程序里的不同函数。
安全指的是不会出现错误，什么错误？既然是同一个进程里的不同线程，能出的错误无非就是资源共享。

<b>那么，golang有哪些安全读写共享变量的方法呢？</b>

### sync.Mutex
这是最典型的方法，通过制造并发锁，来控制特殊程序的堵塞运行。
可以细分到读、写锁。

### channel
由于 channel 是一种协程之间共享资源的通道。他是并发安全的，但是 channel 只存在于内存中，如果你对数据的丢失不敏感，可以直接用 channel 作为资源共享控制的手段。

注意了，channel 是分有缓冲和无缓冲两种的：
```
var ch = make(chan int) // 无缓冲
var ch = make(chan int, 1) // 有缓冲
```
那有无缓冲有什么区别呢？除了缓冲可以容纳更多的数据外，无缓冲就算是空的，也会死锁，因为他需要有一个协程处理 channel 的方法，
只有有了这个方法，无缓冲 chan 才能被使用。所以，无缓冲 channel 可以认为是同步的。

## 并发模型
并发模型有哪些？需要先知道在并发模型里都有哪些角色。
### 1. 用户线程

### 2. 内核线程

有了上面的角色定义后，模型的不同就在于两个角色的配对比例：
1. 1:1
   
2. N:1
   
3. M:N
   

- work stealing 算法

- G-P-M

- 
