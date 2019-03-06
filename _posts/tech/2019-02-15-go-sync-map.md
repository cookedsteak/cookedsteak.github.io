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
其实，原始的思路并没有发生改变，还是用一个 BanList 去盛放哪些暂时无法访问的用户 id。
然后每次访问的时候判断一下这个用户是否在这个 List 中。

## 修改
好，那我们现在需要一个结构体，因为我们会并发读取 map，所以我们直接使用 sync.Map：
```go
type Ban struct {
    M sync.Map
}
```
如果你点进 sync.Map 你会发现他真正存储数据的是一个`atomic.Value`。
一个具有原子特性的 interface{}。

同时Ban这个结构提还会有一个 `IsIn` 的方法用来判断用户 id 是否在Map中。
```go
func (b *Ban) IsIn(user string) bool {
    fmt.Printf("%s 进来了\n", user)
    // Load 方法返回两个值，一个是如果能拿到的 key 的 value
    // 还有一个是否能够拿到这个值的 bool 结果
	v, ok := b.M.Load(user) // sync.Map.Load 去查询对应 key 的值
	if !ok {
        // 如果没有，说明可以访问
        fmt.Printf("名单里没有 %s，可以访问\n", user)
        // 将用户名存入到 Ban List 中
        b.M.Store(ip, time.Now())
		return false
    }
    // 如果有，则判断用户的时间距离现在是否已经超过了 180 秒，也就是3分钟
	if time.Now().Second() - v.(time.Time).Second() > 180 {
        // 超过则可以继续访问
        fmt.Printf("时间为：%d-%d\n", v.(time.Time).Second(), time.Now().Second())
        // 同时重新存入时间
        b.M.Store(ip, time.Now())
		return false
	}
    // 否则不能访问
	fmt.Printf("名单里有 %s，拒绝访问\n", user)
	return true
}
```

下面看看测试的函数：
```go
func main() {
	var success int64 = 0
    ban := new(Ban)
	wg := sync.WaitGroup{} // 保证程序运行完成
	for i := 0; i < 2; i++ { // 我们大循环两次，每个user 连续访问两次
		for j := 0; j < 10; j++ { // 人数预先设定为 10 个人
			wg.Add(1)
			go func(c int) {
				defer wg.Done()
				ip := fmt.Sprintf("%d", c)
				if !ban.IsIn(ip) {
                    // 原子操作增加计数器，用来统计我们人数的
					atomic.AddInt64(&success, 1)
				}
			}(j)
		}
	}
	wg.Wait()
	fmt.Println("此次访问量：", success)
}
```
其实测试的函数并没有做大的改动，只不过，因为我们是并发去运行的，需要增加一个 sync.WaitGroup() 保证程序完整运行完毕后才退出。

我特地把运行数值调小一点，以方便测试。
把`1000`次请求，改为`2`次。`100`人改为`10`人。

所以整个代码应该是这样的：
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
    ...
}

func main() {
    ...
}
```

运行一下...

诶，似乎不太对哦，发现会出现 10~15 次不等的访问量结果。为什么呢？
寻思着，其实因为并发导致的，看到这里了吗？
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
并发发起的 `sync.Map.Load` 其实并没有与 `sync.Map.Store` 连接起来形成原子操作。
所以如果有3个 user 同时进来，程序同时查询，三个返回结果都会是 false（不在名单里）。
所以也就增加了访问的数量。

其实 sync.Map 也已经考虑到了这种情况，所以他会有一个 `LoadOrStore` 的原子方法--
如果 Load 不出，就直接 Store，如果 Load 出来，那啥也不做。

所以我们小改一下 IsIn 的代码：
```go
func (b *Ban) IsIn(user string) bool {
	...
	v, ok := b.M.LoadOrStore(user, time.Now())
	if !ok {
		fmt.Printf("名单里没有 %s，可以访问\n", user)
        // 删除b.M.Store(ip, time.Now())
		return false
	}
	...
}
```
然后我们再运行一下，运行几次。
发觉不会再出现 此次访问量大于 10 的情况了。

## 深究一下
到此为止，这个场景下的代码实现我们算是成功了。
但是真正限制用户访问的场景需求可不能这么玩，一般还是配合内存数据库去实现。

那么，如果你只想了解 sync.Map 的应用，就到这里为止了。
然而好奇心驱使我看看 sync.Map 的实现，我们继续吧。

## 知道 hash table 吗？

好吧，我承认你需要先了解什么是 map，什么是 hash table。
你可以看一下我的另一篇文章，Hash Table。
同时，还希望你能区别一下 哈希表（hash table），字典（可以看做 map），二维数组的区别，可以看[【这里】](https://www.zhihu.com/question/266414962)

## 制造问题
如果硬是要并发读写 map 会怎么样。

先来个主角 A
```go
type A map[string]int
```
我们定义成了自己一个类型 A，他骨子里还是 map。

```go
type A map[string]int

func main() {
	// 初始化一个 A
	m := make(A)
	m["one"] = 1
	m["two"] = 3

	// 读取 one
	go m.ReadMap("one")
	// 设置 two 值为 2
	go m.SetMap("two", 2)

	time.Sleep(1*time.Second)
}

// A 有个读取某个 Key 的方法
func (a A)ReadMap(key string) {
	fmt.Printf("Get Key %s: %d",key, a[key])
}
// A 有个设置某个 Key 的方法
func (a A)SetMap(key string, value int) {
	a[key] = value
	fmt.Printf("Set Key %s: %d",key, a[key])
}
```
诶，看上去不错，我们给 map A 类型定义了 get， set 方法，如果 golang 不允许并发读写 map 的话，应该会报错吧，我们跑一下。

```bash
> Get Key one: 1
> Set Key two: 2
```
喵喵喵???
为什么正常输出了？
说好的并发读写报错呢？
好吧，其实原因是上面的 map 读写，虽然我们设置了协程，但是对于计算机来说还是有时间差的。只要一个微小的先后，就不会造成 map 数据的读写异常，所以我们改一下。
```go
func main() {
	m := make(A)
	m["one"] = 1
	m["two"] = 3

	go func() {
		for {
			m.ReadMap("one")
		}
	}()

	go func(){
		for {
			m.SetMap("two", 2)
		}
	}()

	time.Sleep(1*time.Second)
}
```
为了让读写能够尽可能碰撞，我们增加了循环。
现在我们可以看到了：
```
> fatal error: concurrent map read and map write
```

## 解决问题
我们证实了 map 并发读写的问题，现在我们尝试来解决。

#### 思路
既然是读写造成的冲突，那我们首先考虑的便是读写分离
我们让 一个 map 变成两个。
`read map` 用作读取
`dirty map` 用作写入，因为一直会变嘛，所以我们叫他脏数组





我们进入 sync.Map 的定义，map.go。
```
type Map struct {
    ...
}

type readOnly struct {
    ...
}

type entry struct {
    ...
}
```

Sync.Map 的功能实现，大部分还是依靠了 atomic 


## 参考
- https://www.jianshu.com/p/aa0d4808cbb8
- https://www.zhihu.com/question/266414962
- https://juejin.im/post/5ae01447f265da0ba062d2e8