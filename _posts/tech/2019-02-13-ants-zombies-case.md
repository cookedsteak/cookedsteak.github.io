---
layout: post
title: Ants 的僵尸案例 :)
category: 技术
keywords: golang,协程,协程池,后端
comments: false
---

偶尔看见一个不错的协程池实现，觉得不错，但是官方例子有些冗余，所以自己写了个超简单的应用场景。
如果想要了解实现原理的可以点击[【这里】](https://zhuanlan.zhihu.com/p/37754274)，原作者已经写得很清楚了。

## 栗子
场景是--我们要处决僵尸，但是只有3把电椅。
当然，最主要的还是要有僵尸给我们盘，所以我们有两个方法：
```go
var tunnel = make(chan string, 1) // 僵尸隧道

// 处决僵尸
func ExecuteZombie(i interface{}) {
	fmt.Printf("正在处决僵尸 %s 号，还有5秒钟....\n", i.(string))
	time.Sleep(5*time.Second)
	fmt.Printf(":) %s 玩完了，下一个\n-----------------\n", i.(string))
}

// 欢迎僵尸
func IncomingZombie() {
	for i := 0; i < 10; i++ {
		tunnel <- strconv.Itoa(i)
	}
}
```
这一波大概会来10个左右的僵尸。
处理一个僵尸大概5秒钟。我们准备了三把电椅：
```go
func main() {
	go IncomingZombie()

	chairPool, _ := ants.NewPoolWithFunc(3, ExecuteZombie) // 电椅就位
	defer chairPool.Release()

	for {
		select {
		case a := <-tunnel:
			go chairPool.Serve(a) // 来一个电一个
		}
	}
}
```
合起来就是
```go
package main

import (
	"fmt"
	"strconv"
	"time"

	"github.com/panjf2000/ants"
)

var tunnel = make(chan string, 1)

func main() {
	go IncomingZombie()

	chairPool, _ := ants.NewPoolWithFunc(3, ExecuteZombie) // 声明有几把电椅
	defer chairPool.Release()

	for {
		select {
		case a := <-tunnel:
			go chairPool.Serve(a)
		}
	}
}

func ExecuteZombie(i interface{}) {
	fmt.Printf("正在处决僵尸 %s 号，还有5秒钟....\n", i.(string))
	time.Sleep(5*time.Second)
	fmt.Printf(":) %s 玩完了，下一个\n-----------------\n", i.(string))
}


func IncomingZombie() {
	for i := 0; i < 10; i++ {
		tunnel <- strconv.Itoa(i)
	}
}
```
执行看看...

你会发现，一次只会同步处决3枚僵尸。