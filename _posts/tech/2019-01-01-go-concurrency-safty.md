---
layout: post
title: golang并发安全试验
category: 技术
keywords: golang,go,并发,channel,routine,context
comments: true
toc: true
---

之前有聊过 golang 的协程，我发觉似乎还很理论，特别是在并发安全上，所以特结合网上的一些例子，来试验下go routine中 的 channel, select, context 的妙用。

## 场景1-抢红包
首先我们来考虑一个场景--红包。
这里有个跨年红包，金额100，我们让阿猫，阿狗同时来抢这个红包，直到这个红包余额变为0。


## 场景2-微服务调用
我们用 gin 作为处理请求的入口，需求是这样的：
一个请求 X 会去并行调用 A, B, C 三个方法，并把三个方法返回的结果加起来作为 X 请求的 Response。
但是我们这个 Response 是有时间要求的（不能超过3秒的响应时间），

可能 A, B, C 中任意一个或两个，处理逻辑十分复杂，或者数据量超大，导致处理时间超出预期，
那么我们就马上切断，并返回已经拿到的任意个返回结果之和。

我们先来定义主函数：
```
func main() {
	r := gin.New()
	r.GET("/calculate", calHandler)
	http.ListenAndServe(":8008", r)
}
```
非常简单，普通的请求接受和 handler 定义。其中 calHandler 是我们用来处理请求的函数。

分别定义三个假的微服务，其中第三个将会是我们超时的哪位~
```
func microService1() int {
	time.Sleep(1*time.Second)
	return 1
}

func microService2() int {
	time.Sleep(2*time.Second)
	return 2
}

func microService3() int {
	time.Sleep(10*time.Second)
	return 3
}
```

接下来，我们看看 calHandler 里到底是什么
```
func calHandler(c *gin.Context) {
    ...
}
```
#### 要点1--并发调用

直接用 go 就好了嘛~
所以一开始我们可能就这么写：
```
go microService1()
go microService2()
go microService3()
```
很简单有没有，但是等等，说好的返回值我怎么接呢？
为了能够并行地接受处理结果，我们很容易想到用 channel 去接。
所以我们把调用服务改成这样：
```
var resChan = make(chan int, 3) // 因为有3个结果，所以我们创建一个可以容纳3个值的 int channel。
go func() {
    resChan <- microService1()
}()

go func() {
    resChan <- microService2()
}()

go func() {
    resChan <- microService3()
}()
```
有东西接，那也要有方法去算，所以我们加一个一直循环拿 resChan 中结果并计算的方法：
```
var resContainer, sum int
for {
    resContainer = <-resChan
    sum += resContainer
}
```
这样一来我们就有一个 sum 来计算每次从 resChan 中拿出的结果了。

还没结束，说好的超时处理呢？
为了实现超时处理，我们需要引入一个东西，就是 context，[什么是 context ?](https://gocn.vip/article/373)
我们这里只使用 context 的一个特性，超时通知（其实这个特性完全可以用 channel 来替代）。

可以看在定义 calHandler 的时候我们已经将 c *gin.Context 作为参数传了进来，那我们就不用自己在声明了。
gin.Context 简单理解为贯穿整个 gin 声明周期的上下文容器，有点像是分身，亦或是量子纠缠的感觉。

有了这个 gin.Context， 我们就能在一个地方对 context 做出操作，而其他正在使用 context 的函数或方法，也会感受到 context 做出的变化。

```
ctx, cancel := context.WithTimeout(c, 3*time.Second) //定义一个超时的 context
defer cancel() // 在超时前完成操作，同样也要释放资源
```

只要时间到了，我们就能用 ctx.Done() 获取到一个超时的 channel(通知)，然后其他用到这个 ctx 的地方也会停掉，并释放 ctx。
一般来说，ctx.Done() 是结合 select 使用的。

```

```

