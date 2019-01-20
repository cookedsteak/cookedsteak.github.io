---
layout: post
title: 优雅重启 Golang Web Server
category: 技术
keywords: golang,interface,go,框架,后端
comments: false
---

本文参考 [GRACEFULLY RESTARTING A GOLANG WEB SERVER](https://tomaz.lovrec.eu/posts/graceful-server-restart/)
进行归纳和说明。

## 问题
因为 golang 是编译型的，所以当我们修改一个用 go 写的服务的配置后，需要重启该服务，有的甚至还需要重新编译，再发布。如果在重启的过程中有大量的请求涌入，能做的无非是分流，或者堵塞请求。不论哪一种，都不优雅~，所以slax0r以及他的团队，就试图探寻一种更加平滑的，便捷的重启方式。

原文章中除了排版比较帅外，文字内容和说明还是比较少的，所以我希望自己补充一些说明。

## 原理
上述问题的根源在于，我们无法同时让两个服务，监听同一个端口。
解决方案就是复制当前的 listen 文件，然后在新老进程之间通过 socket 直接传输参数和环境变量。
新的开启，老的关掉，就这么简单。

#### 防看不懂须知 
(Unix domain socket)[https://en.wikipedia.org/wiki/Unix_domain_socket]
(一切皆文件)[]


## 代码思路
因为 http server 的运行需要一个监听对象，该对象包含了我们需要监听的
```
func main() {
    主函数，初始化配置
    调用serve()
}

func serve() {
    核心运行函数
    getListener()   // 1. 获取监听 listener
    start()         // 2. 用获取到的 listener 开启 server 服务
    waitForSignal() // 3. 监听外部信号，用来控制程序 fork 还是 shutdown
}

func getListener() {
    获取正在监听的端口对象
    （第一次运行新建）
}

func start() {
    运行 http server
}

func waitForSignal() {
    for {
        等待外部信号
        1. fork子进程
        2. 关闭进程
    }
}
```
上面是代码思路的说明，基本上我们就围绕这个大纲填充完善代码。

## 定义结构体
我们抽象出两个结构体，描述程序中公用的数据结构
```
var cfg *srvCfg
type listener struct {
	// Listener address
	Addr string `json:"addr"`
	// Listener file descriptor
	FD int `json:"fd"`
	// Listener file name
	Filename string `json:"filename"`
}

type srvCfg struct {
	sockFile string
	addr string
	ln net.Listener
	shutDownTimeout time.Duration
	childTimeout time.Duration
}
```
listener 是我们的监听者，他包含了监听地址，文件描述符，文件名。
文件描述符其实就是进程所需要打开的文件的一个索引，非负整数。
我们平时创建一个进程时候，linux都会默认打开三个文件，标准输入stdin,标准输出stdout,标准错误stderr，
这三个文件各自占用了 0，1，2 三个文件描述符。所以之后你进程还要打开文件的话，就得从 3 开始了。
这个listener，就是我们进程之间所要传输的数据了。

srvCfg 是我们的全局环境配置，包含 socket file 路径，服务监听地址，监听者对象，父进程超时时间，子进程超时时间。
因为是全局用的配置数据，我们先 var 一下。

## 入口
看看我们的 main 长什么样子
```
func main() {
	serve(srvCfg{
		sockFile: "/tmp/api.sock",
		addr:     ":8000",
		shutDownTimeout: 5*time.Second,
		childTimeout: 5*time.Second,
	}, http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte(`Hello, world!`))
	}))
}

func serve(config srvCfg, handler http.Handler) {
	cfg = &config
	var err error
	// get tcp listener
	cfg.ln, err = getListener()
	if err != nil {
		panic(err)
	}

	// return an http Server
	srv := start(handler)

	// create a wait routine
	err = waitForSignals(srv)
	if err != nil {
		panic(err)
	}
}
```
很简单，我们把配置都准备好了，然后还注册了一个 handler--输出 Hello, world!

serve 函数的内容就和我们之前的思路一样，只不过多了些错误判断。

接下去，我们一个一个看里面的函数...

## 获取 listener
也就是我们的 getListener() 函数
```
func getListener() (net.Listener, error) {
    // 第一次执行不会 importListener
	ln, err := importListener()
	if err == nil {
		fmt.Printf("imported listener file descriptor for addr: %s\n", cfg.addr)
		return ln, nil
	}
    // 第一次执行会 createListener
	ln, err = createListener()
	if err != nil {
		return nil, err
	}

	return ln, err
}

func importListener() (net.Listener, error) {
    ...
}

func createListener() (net.Listener, error) {
	fmt.Println("首次创建 listener", cfg.addr)
	ln, err := net.Listen("tcp", cfg.addr)
	if err != nil {
		return nil, err
	}

	return ln, err
}
```
因为第一次不会执行 importListener， 所以我们暂时不需要知道 importListener 里是怎么实现的。
只肖明白 createListener 返回了一个监听对象。

而后就是我们的 start 函数
```
func start(handler http.Handler) *http.Server {
	srv := &http.Server{
		Addr: cfg.addr,
		Handler: handler,
	}
	// start to serve
	go srv.Serve(cfg.ln)
	fmt.Println("server 启动完成，配置信息为：",cfg.ln)
	return srv
}
```
很明显，start 传入一个 handler，然后协程运行一个 http server。

## 监听信号
监听信号应该是我们这篇里面重头戏的入口，我们首先来看下代码：
```
func waitForSignals(srv *http.Server) error {
	sig := make(chan os.Signal, 1024)
	signal.Notify(sig, syscall.SIGTERM, syscall.SIGINT, syscall.SIGHUP)
	for {
		select {
		case s := <-sig:
			switch s {
			case syscall.SIGHUP:
				err := handleHangup() // 关闭
				if err == nil {
					// no error occured - child spawned and started
					return shutdown(srv)
				}
			case syscall.SIGTERM, syscall.SIGINT:
				return shutdown(srv)
			}
		}
	}
}
```
首先建立了一个通道，这个通道用来接收系统发送到程序的命令，比如`kill -9 myprog`，
这个 `9` 就是传到通道里的。我们用 Notify 来限制会产生响应的信号，这里有：
- SIGTERM 
- SIGINT
- SIGHUP
(关于信号)[https://unix.stackexchange.com/questions/251195/difference-between-less-violent-kill-signal-hup-1-int-2-and-term-15]

如果实在搞不清这三个信号的区别，只要明白我们通过区分信号，留给了进程自己判断处理的余地。

然后我们开启了一个循环监听，显而易见地，监听的就是系统信号。
当信号为 `syscall.SIGHUP` ，我们就要重启进程了。
而当信号为 `syscall.SIGTERM, syscall.SIGINT`时，我们直接关闭进程。

于是乎，我们就要看看，`handleHangup`里面到底做了什么。

## 父子间的对话