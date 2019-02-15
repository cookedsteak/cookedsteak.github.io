---
layout: post
title: Go sync.Map 的使用
category: 技术
keywords: golang,sync,后端,并发安全
comments: false
---

偶然看见这么篇文章：[一道并发和锁的golang面试题](https://blog.csdn.net/qq_28163175/article/details/75287877)。
虽然年代久远，但我也稍有兴趣。

正好最近也看到了 sync.Map，所以想试试能不能用 sync.Map 去实现上述的功能。

我还在 gayhub上找到了其他人用 sync.Mutex 的实现方式，[【这里】](https://github.com/AbelLai/What-I-Will-Say/issues/32)。

## 归结一下

需求是这样的：
> 在一个高并发的web服务器中，要限制IP的频繁访问。现模拟100个IP同时并发访问服务器，每个IP要重复访问1000次。每个IP三分钟之内只能访问一次。修改以下代码完成该过程，要求能成功输出 success: 100。

并且给出了原始代码：
```go
package main

import (
	"fmt"
	"time"
)

type Ban struct {
	visitIPs map[string]time.Time
}

func NewBan() *Ban {
	return &Ban{visitIPs: make(map[string]time.Time)}
}

func (o *Ban) visit(ip string) bool {
	if _, ok := o.visitIPs[ip]; ok {
		return true
	}
	o.visitIPs[ip] = time.Now()
	return false
}

func main() {
	success := 0
	ban := NewBan()
	for i := 0; i < 1000; i++ {
		for j := 0; j < 100; j++ {
			go func() {
				ip := fmt.Sprintf("192.168.1.%d", j)
				if !ban.visit(ip) {
					success++
				}
			}()
		}
	}
	fmt.Println("success: ", success)
}
```
哦吼，看到源代码我想说，我能只留个`package main`其他都重新写吗？（捂脸）

聪明的你已经发现，这个问题关键就是想让你给 Ban 加一个读写锁罢了。
而且条件中的三分钟根本无伤大雅，因为这程序压根就活不到那天。

## 思路


## 修改
```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"time"
)

type Ban struct {
	M sync.Map
}

func (b *Ban) IsIn(user string) bool {
	fmt.Printf("%s 进来了\n", user)
	v, ok := b.M.Load(user)
	if !ok {
		fmt.Printf("名单里没有 %s，可以访问\n", user)
        b.M.Store(ip, time.Now())
		return false
	}
	if time.Now().Second() - v.(time.Time).Second() > 180 {
		fmt.Printf("时间为：%d-%d\n", v.(time.Time).Second(), time.Now().Second())
		b.M.Delete(user)
		return false
	}

	fmt.Printf("名单里有 %s，拒绝访问\n", user)
	return true
}

func NewBanList() *Ban {
	return &Ban{}
}

func main() {
	var success int64 = 0
	ban := NewBanList()
	wg := sync.WaitGroup{}
	for i := 0; i < 2; i++ {
		for j := 0; j < 10; j++ {
			wg.Add(1)
			go func(c int) {
				defer wg.Done()
				ip := fmt.Sprintf("%d", c)
				if !ban.IsIn(ip) {
					atomic.AddInt64(&success, 1)
				}
			}(j)
		}
	}
	wg.Wait()
	fmt.Println("此次访问量：", success)
}
```
我们把运行数值调小一点，以方便测试。
把`1000`次请求，改为`2`次。`100`人改为`10`人。

运行一下...诶，似乎不太对哦，会出现 10~15 次不等的访问量结果。
寻思着，还是因为并发导致的，看到这里了吗？
```go
func (b *Ban) IsIn(user string) bool {
	...
	v, ok := b.M.Load(user)
	if !ok {
		fmt.Printf("名单里没有 %s，可以访问\n", user)
        b.M.Store(ip, time.Now())
		return false
	}
	...
}
```
并发发起的 sync.Map.Load 其实并没有与 sync.Map.Store 连接起来形成原子操作。
所以如果有3个 user 同时进来，程序同时查询，三个返回结果都会是 false（不在名单里）。