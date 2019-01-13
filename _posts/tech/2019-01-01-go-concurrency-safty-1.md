---
layout: post
title: golang并发安全试验-1
category: 技术
keywords: golang,go,并发,channel,routine,context
comments: true
toc: true
---

之前有聊过 golang 的协程，我发觉似乎还很理论，特别是在并发安全上，所以特结合网上的一些例子，来试验下go routine中 的 channel, select, context 的妙用。

## 场景-微服务调用
我们用 [gin](https://github.com/gin-gonic/gin)（一个web框架） 作为处理请求的工具，需求是这样的：
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

#### 要点2--超时信号
还没结束，说好的超时处理呢？
为了实现超时处理，我们需要引入一个东西，就是 context，[什么是 context ?](https://gocn.vip/article/373)
我们这里只使用 context 的一个特性，超时通知（其实这个特性完全可以用 channel 来替代）。

可以看在定义 calHandler 的时候我们已经将 c *gin.Context 作为参数传了进来，那我们就不用自己在声明了。
gin.Context 简单理解为贯穿整个 gin 声明周期的上下文容器，有点像是分身，亦或是量子纠缠的感觉。

有了这个 gin.Context， 我们就能在一个地方对 context 做出操作，而其他正在使用 context 的函数或方法，也会感受到 context 做出的变化。

```
ctx, _ := context.WithTimeout(c, 3*time.Second) //定义一个超时的 context
```

只要时间到了，我们就能用 ctx.Done() 获取到一个超时的 channel(通知)，然后其他用到这个 ctx 的地方也会停掉，并释放 ctx。
一般来说，ctx.Done() 是结合 select 使用的。
所以我们又需要一个循环来监听 ctx.Done()
```
for {
    select {
    case <- ctx.Done():
        // 返回结果
}
```
现在我们有两个 for 了，是不是能够合并下？
```
for {
    select {
    case resContainer = <-resChan:
        sum += resContainer
        fmt.Println("add", resContainer)
    case <- ctx.Done():
        fmt.Println("result:", sum)
        return
    }
}
```
诶嘿，看上去不错。
不过我们怎么在正常完成微服务调用的时候输出结果呢？
看来我们还需要一个 flag
```
var count int
for {
    select {
    case resContainer = <-resChan:
        sum += resContainer
        count ++
        fmt.Println("add", resContainer)
        if count > 2 {
            fmt.Println("result:", sum)
            return
        }
    case <- ctx.Done():
        fmt.Println("timeout result:", sum)
        return
    }
}
```
我们加入一个计数器，因为我们只是调用3次微服务，所以当 count 大于2的时候，我们就应该结束并输出结果了。

#### 要点3--并发中的等待
上面的计时器是一种偷懒的方法，因为我们知道了调用微服务的次数，如果我们并不知道，或者之后还要添加呢？
手动每次改 count 的判断阈值会不会太沙雕了？这时候我们就要加入 sync 包了。
我们将会使用的 sync 的一个特性是 WaitGroup。它的作用是等待一组协程运行完毕后，执行接下去的步骤。

我们来改下之前微服务调用的代码块：
```
var success = make(chan int, 1) // 成功的通道标识
wg := sync.WaitGroup{} // 创建一个 waitGroup 组
wg.Add(3) // 我们往组里加3个标识，因为我们要运行3个任务
go func() {
    resChan <- microService1()
    wg.Done() // 完成一个，Done()一个
}()

go func() {
    resChan <- microService2()
    wg.Done()
}()

go func() {
    resChan <- microService3()
    wg.Done()
}()
wg.Wait() // 直到我们前面三个标识都被 Done 了，否则程序一直会阻塞在这里
success <- 1 // 我们发送一个成功信号到通道中
```

既然我们有了 success 这个信号，那么再把它加入到监控 for 循环中，并做些修改，删除原来 count 判断的部分。
```
go func() {
    for {
        select {
        case resContainer = <-resChan:
            sum += resContainer
            fmt.Println("add", resContainer)
        case <- success:
            fmt.Println("result:", sum)
            return
        case <- ctx.Done():
            fmt.Println("result:", sum)
            return
        }
    }
}()
```
三个 case，分工明确，一个用来拿服务输出的结果并计算，一个用来做最终的完成输出，一个是超时输出。
同时我们将这个循环监听，也作为协程运行。

至此，所有的主要代码都完成了。下面是完全版
```
package main

import (
	"context"
	"fmt"
	"net/http"
	"sync"
	"time"

	"github.com/gin-gonic/gin"
)

// 一个请求会触发调用三个服务，每个服务输出一个 int,
// 请求要求结果为三个服务输出 int 之和
// 请求返回时间不超过3秒，大于3秒只输出已经获得的 int 之和
func calHandler(c *gin.Context) {
	var resContainer, sum int
	var success, resChan = make(chan int), make(chan int, 3)
	ctx, _ := context.WithTimeout(c, 3*time.Second)

	go func() {
		for {
			select {
			case resContainer = <-resChan:
				sum += resContainer
				fmt.Println("add", resContainer)
			case <- success:
				fmt.Println("result:", sum)
				return
			case <- ctx.Done():
				fmt.Println("result:", sum)
				return
			}
		}
	}()

	wg := sync.WaitGroup{}
	wg.Add(3)
	go func() {
		resChan <- microService1()
		wg.Done()
	}()

	go func() {
		resChan <- microService2()
		wg.Done()
	}()

	go func() {
		resChan <- microService3()
		wg.Done()
	}()
	wg.Wait()
	success <- 1

	return
}

func main() {
	r := gin.New()
	r.GET("/calculate", calHandler)
    http.ListenAndServe(":8008", r)
}

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
上面的程序只是简单描述了一个调用其他微服务超时的处理场景。
实际过程中还需要加很多很多调料，才能保证接口的对外完整性。