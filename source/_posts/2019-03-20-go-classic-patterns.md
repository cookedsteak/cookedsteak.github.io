---
layout: post
title: Go 设计模式
category: 技术
keywords: 后端,golang,设计模式
comments: true
---

虽说是 go 的设计模式，但是其是设计模式这种是通用的。
我们今天就来看看几个常用的设计模式在 golang 中的实现。

## 单例 Singleton design pattern
保证你只有一个想要的对象。并且不会又重复。
比如像是 ssh connection，或者 mysql\redis connection。
我们可以用单例模式来保证连接资源不被浪费。
比如：
```go
func GetInstance() *singleton {
    if instance == nil {
        mu.Lock()

        if instance == nil {
            instance = &singleton{}
        }
        mu.Unlock()
    }
    return instance
}
```
当然，除了手动判断，还能够利用 sync.Once 这个包
```go
var once = sync.Once{} // 声明一个一次性执行的容器

func GetInstance() *singleton {
    once.Do(func(){
        instance = &singleton{}
    })

    return instance
}
```
结果一样，也只执行了一次。
这里还需要考虑下，如果是多线程并发执行这个 GetInstance，如果没锁的话会造成 instance 被赋值多次。
<!-- more -->
## 生成器 Builder design pattern

## 工厂 Factory method
工厂不单单是功能的抽象，他其实还包含了任务自动化，一个参数变多个参数，
调用接口化。

工厂模式其实也是有组成规律的，一般需要

1. 抽象的产品-对象接口
2. 具体的产品
3. 抽象的工厂-工厂接口
4. 具体的工厂
5. 根据不同参数而生成的不同工厂（switch）

没错，就这么简单，但是随着工厂条件的复杂程度，会有1层甚至多层的抽象，这个就要具体情况具体分析了。
下面是一个简单工厂的例子，生产罐头的：
```go
package main

import (
	"fmt"
)
// 产品结构体
type Cans struct {
	Material string
	Quantity int
}
// 抽象工厂
type ICanFactory interface {
	MakeCan() Cans
}
// 工厂基类，用来生成不同工厂，也可以用作 default 厂
type CanFactory struct {}

func (c *CanFactory) MakeCan() Cans {
	can := Cans{
		Material: "nothing",
		Quantity: 10,
	}

	return can
}

// 生成不同工厂
func (c *CanFactory) NewCanFactory(name string) ICanFactory {
	switch name{
	case "cat":
		return new(CatCanFactory)
	case "dog":
		return new(DogCanFactory)
	default:
		return c
	}
}
// 猫厂
type CatCanFactory struct {}
// 狗厂
type DogCanFactory struct {}
// 实现工厂接口
func (cf *CatCanFactory) MakeCan() Cans {
	can := Cans{
		Material: "fish",
		Quantity: 10,
	}

	return can
}
// 实现工厂接口
func (df *DogCanFactory) MakeCan() Cans {
	can := Cans{
		Material: "meat",
		Quantity: 10,
	}

	return can
}

func main() {
	f := new(CanFactory)
	can := f.NewCanFactory("cat").MakeCan()

	fmt.Println(can)
}
```

### 抽象工厂 Abstract factory
抽象工厂，在简单工厂的基础上，将一套完整的流程用抽象的方式定义出来，并且相互依赖。
最后只需要给定具体的参数，就能自动完成具体类的实现，方法操作。


## 原型 Prototype design pattern

