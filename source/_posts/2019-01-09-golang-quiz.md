---
layout: post
title: Golang Adventure
category: 技术
keywords: golang,interface,go,框架,后端
comments: false
---

这里记录些关于 golang 的问题。
有些很基础，有些很有趣。总之，是作为 gopher 都应该掌握的知识。

---

## 实现接口的到底是谁
今天在网上看到一个问题，关于 golang 的 interface 实现的问题。

``` go
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

那如果我们以 `(m Mana)Goo` 以去实现 Goo这个方法，`*Mana` 是不是也实现了 Goo 这个方法呢？
答案是 `Yep`。（面试血的教训）

---

## 反射拿啥？

啥是反射？语言对自己行为的描述和监控。
GRPC 就是通过反射实现的。
golang 的反射实现需要的一个前提是在做变量声明与创建的时候，有一个 pair 的概念。他记录了变量的值以及值类型。
具体代码在 runtime>runtime2.go 中的 `itab` 结构体。

```go
type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```

反射的几个基本操作。
`reflect.ValueOf(T)` 获取反射值 - 比如这个值是1<br/>
`reflect.TypeOf(T)` 获取反射值类型 - 比如这个类型是 int<br/>
`reflect.TypeOf(T).NumField()` 获取反射值（结构体）的属性个数<br/>
`reflect.TypeOf(T).NumMethod()` 获取反射值（结构体）的方法个数<br/>
`reflect.TypeOf(T).Field(I)` 直接通过 index 索引去获取属性<br/>
`reflect.TypeOf(T).Method(I)` 直接通过 index 索引去获取方法<br/>

---

## golang 运行位置

我们可以通过 `runtime.Caller` 来获取当前运行的文件目录。
要注意的是 runtime.Caller 中有一个int参数，注解是这样的：
> The argument skip is the number of stack frames
to ascend, with 0 identifying the caller of Caller.

意思就是你要追溯的堆栈的级数。

返回的数据分别是：
> The return values report the
program counter, file name, and line number within the file of the corresponding
call. The boolean ok is false if it was not possible to recover the information.

程序计数器，文件名，文件对应的行号，以及是否拿到了运行堆信息。

举个例子，最简单的如下代码：

```go
func main() {
    fmt.Println(runtime.Caller(2))
}
```

我们改变参数

- 参数为 0
返回 17371019 /my/file/path/main.go 18 true
- 参数为 1
返回了 16943046 /usr/local/go/src/runtime/proc.go 201 true
- 参数为 2
返回了 17107488 /usr/local/go/src/runtime/asm_amd64.s 1333 true

当我们拿到了自己所要的层级路径后，可以使用 filepath 包去处理。
常用的该包的方法有：

- `filepath.Rel(a, b)` b 相较于 a 的相对路径
- `filepath.Join(a, b)` a 与 b 组成新的路径
- `filepath.Clean(a, b)` 清理路径中的多余字符
- `filepath.Abs(a)` a 的绝对路径
- `filepath.Match(a, b)` 按照 a 的正则比较 b 路径

---

## golang 的自举

什么是自举？就是一种向下兼容的编译策略，借用老版本的 runtime，编译新版本的编译、连接工具。
然后再用新的工具去编译真正的目标代码。

## 什么是 runtime？

一直听说 runtime,runtime，那 runtime 到底是什么呢？
知乎上也有相关的问题，可以看下[到底什么是 runtime](https://www.zhihu.com/question/20607178)。

## slice：我怕 array 太寂寞

slice 是一个十分方便的数据结构。我一直认为 slice 是 array 的升级版本。
的确是这样，slice 中传递的不再是真正的值，他给 array 套了一层扩展骨架，同时变成了引用传递的方式进行骨架的操作。
注意，golang 中不存在指针传递，但是为什么有时候会变得像“引用传递”呢？
其实是因为你传递的本身就是个内存地址。

比如 Slice 的结构：

```go
type SliceHeader struct {
    Data uintptr
    len int
    cap int
}
```

Slice 结构中存放的一直都是一个指针。
也正是因为这个原因，所以或导致下面这种有趣的结果：

```go
func main() {
    s := make([]int, 2)
    mdSlice(s)
    fmt.Println(s)
}

func mdSlice(s []int) {
    s[0] = 1
    s[1] = 2
}
```

输出结构是：[1,2]

```go
func main() {
    s := make([]int, 2)
    mdSlice(s)
    fmt.Println(s)
}

func mdSlice(s []int) {
    s = append(s, 1)
    s = append(s, 2)
}
```

输出结果是：[0,0]

在第二个例子中，append 的操作只会对复制后的参数起效，所以原 slice 还是没有append 的。

---

## 为啥要 make

我们一直知道，golang 中有些数据类型需要先 make，slice，map，chan。但是有没有想过为什么要 make 呢？

其实 make 并不是一个通用的方法，而是会根据 make 数据类型的不同而改变的。
我们就拿 `makeMap` 作为例子看下源码的实现：

首先我们还是要了解 map的数据结构。可以翻看我的另一篇文章<从 Hash Table 到 Go Map>

---

## defer & panic & recover

defer 永远是 FILO 模型，不论 defer 里面是不是有 revocer。
而 recover 必须配合 defer 才能获取。
panic 后的同作用域的函数都不会被运行。

## 线程安全

不要被这四个字吓到了，其实就是公用资源的锁问题。
线程安全这个词中，线程限定了范围，给出了程序运行环境：说明是一个程序里的不同函数。
安全指的是不会出现错误，什么错误？既然是同一个进程里的不同线程，能出的错误无非就是资源共享。

**那么，golang有哪些安全读写共享变量的方法呢?**

### sync.Mutex

这是最典型的方法，通过制造并发锁，来控制特殊程序的堵塞运行。
可以细分到读、写锁。

### channel

由于 channel 是一种协程之间共享资源的通道。他是并发安全的，但是 channel 只存在于内存中，如果你对数据的丢失不敏感，可以直接用 channel 作为资源共享控制的手段。

注意了，channel 是分有缓冲和无缓冲两种的：

```go
var ch = make(chan int) // 无缓冲
var ch = make(chan int, 1) // 有缓冲
```

那有无缓冲有什么区别呢？除了缓冲可以容纳更多的数据外，无缓冲就算是空的，也会死锁，因为他需要有一个协程处理 channel 的方法，
也就是说下面的代码会报错：

```go
package main

import (
    "fmt"
)

func main() {
    i := make(chan int, 1)
    s := make(chan string) // 无缓冲，需要有协程去接这个通道的值，否则就会死锁

    i <- 1
    s <- "name"

    select {
    case value := <-i:
        fmt.Println(value)
    case value := <-s:
        fmt.Println(value)
    }
}
```

只有有了这个方法，无缓冲 chan 才能被使用。所以，无缓冲 channel 可以认为是同步的。
我们除了用 `<-` 去获取 channel 中的数据，还可以用 `range` 阻塞获取。
这样的读取方式会阻塞当前协程，如果在其他协程中调用了close(channel),那么就会跳出for range循环。

另外，如果你在一个协程中使用了无缓冲的channel，那么你无法在同一个协程中获取这个channel的值，原因很简单，因为这个协程被blocking了。除非有其他的协程拿走了channel的值，才能继续执行下去。

channel 也是有状态的！

关闭一个channel只需要调用函数close()即可，
如果channel已经关闭,继续往它发送数据会导致panic: send on closed channel，
channel关闭后，仍然可以从中读取以发送的数据，读取完数据后，将读取到零值，可以多次读取。

- chan 和 <-chan、chan<- 的区别
    在很多函数中，我们可以看到有箭头的 chan 类型。
    记住箭头的作用
    > The optional <- operator specifies the channel direction, send or receive. If no direction is given, the channel is bidirectional. A channel may be constrained only to send or only to receive by conversion or assignment.

    箭头符号永远定义了 channel 的方向，到底是接受还是发送，如果没有给到箭头符号，说明这个 channel 是双向的。

    ``` go
    chan T          // T 可以塞数据，也可以读数据
    chan<- float64  // 只能塞入 float64
    <-chan int      // 只能读取 int 数据
    ```

    这个限制 constraint 是对于函数外部而言的。

---

## Golang的协程机制

golang 主要内置实现了协程，基于 CSP 模型。
后者是一种比较有名的并发模型。

那 golang 的协程管理机制是怎么样的呢？
这个就要说到 GPM 模型了。

### GPM
---

## 调试

调用包：

```shell
runtime/pprof
runtime/trace
net/http/pprof
```

一个 go 程序，用于性能分析指标的概要文件(profiles)有三种：

1. CPU profile
2. Men profile 内存概要
3. Block profile 阻塞概要

注意，概要文件里的信息都是二进制的，需要用go tool pprof查看。
这个二进制流信息，正是通过 protobuf （protocal buffer 数据序列化协议）生成的。

如果我们要开始采样的话，我们需要使用`StartCPUProfile`函数。参数是一个 io.Writer。
结束的话使用 `StopCPUProfile`，下面是一个使用例子：

```go
func main() {
    // 打开文件，准备写入
    filename := "cpuprofile2.out"
    f, err := os.Create(filename)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Create File Error: %v", err)
        return
    }
    defer f.Close()

    // 进行采样
    if err := pprof.StartCPUProfile(f); err != nil {
        fmt.Fprintf(os.Stderr, "CPU profile start error: %v\n", err)
        return
    }
    /* 这里写需要测试的代码
    */
    // 停止采样
    pprof.StopCPUProfile()
}
```

但是上述的方法有时候会失效，更简单的方法是：
献上连接：
> <https://flaviocopes.com/golang-profiling/>

网上找了很多关于 pprof 的使用，不是道是不是因为我太傻，愣是没有一个能输出想要的结果。
而且大家都是抄来抄去，很棒，就是这种氛围和调性。

果然还是找到了一篇国外的使用教程，简单明了。
详见上面的链接，我这里概括下主要的使用步骤。

1. 下载 `github.com/pkg/profile` 包
2. `brew install graphviz` 安装图形化插件
3. 在程序中加入 `defer profile.Start().Stop()`
4. 运行程序，会提示你 pprof 文件所在的位置
5. 找到文件并且运行 `go tool pprof --pdf ~/go/bin/yourbinary /var/path/to/cpu.pprof > file.pdf`
6. 完成，看你的 pdf

就是这么简单几步，得到了我想要的性能分析输出。

<!-- more -->

*内存使用统计

- 将语句换成 `defer profile.Start(profile.MemProfile).Stop()` 即可

---

## iota

有了 iota，在声明常量的时候真的会很方便，
不过你搞得起 iota 各种骚气的写法吗？
比如：

```go
const(
    a = iota
    b
    -
    c = iota
)
```

iota 是从0开始的，所以 a=0，然后会依次递增，b=1，-说明占位，他会让 iota 加1，但是不会赋值给任何常量。
如果你这时候重新 iota 一把，iota 的值会从上一个最后一次赋值开始，所以 c=1。

再有：

```go
const (
    Apple, Banana = iota + 1, iota + 2
    Cherimoya, Durian
    Elderberry, Fig
)
```

猜猜看，各会是什么？

``` shell
// Apple: 1
// Banana: 2
// Cherimoya: 2
// Durian: 3
// Elderberry: 3
// Fig: 4
```

iota 他还是根据行来增加的。

---

## 官方 mysql 库

### 连接池与连接数
一些该有的配置文件

---

## 选择性编译

golang 可以通过在包头部增加注释

```go
// +build python
```

进行选择性编译，如果你要编译时包含这个包，则需要

```sh
go build -tags="python"
```

很神奇，导致 goland 里的关联提示一直显示红色，逼死强迫症。

---

uint/int

golang 的int 可不是默认32位，这个值是根据CPU而变化的。

## *参考
- https://juejin.im/post/5a75a4fb5188257a82110544
- http://blog.51cto.com/steed/2349944