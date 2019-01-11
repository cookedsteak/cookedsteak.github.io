---
layout: post
title: Golang Interface
category: 技术
keywords: golang,interface,go,框架,后端
comments: false
---

这里记录些关于 golang interface 的有趣问题

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
