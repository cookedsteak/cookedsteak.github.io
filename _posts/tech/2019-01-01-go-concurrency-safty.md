---
layout: post
title: golang并发安全试验
category: 技术
keywords: golang,go,并发,channel,routine
comments: true
toc: true
---

之前有聊过 golang 的协程，我发觉似乎还很理论，特别是在并发安全上，所以特结合网上的一些例子，来试验下go routine中 的 channel, select, context 的妙用。

## 场景1-抢红包
首先我们来考虑一个场景--红包。
这里有个跨年红包，金额100，我们让阿猫，阿狗同时来抢这个红包，知道这个红包余额变为0。


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
非常简单，普通的请求接受和 handler 定义。