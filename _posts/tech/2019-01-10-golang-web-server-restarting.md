---
layout: post
title: 优雅重启 Golang Web Server
category: 技术
keywords: golang,interface,go,框架,后端
comments: false
---

本文参考 [GRACEFULLY RESTARTING A GOLANG WEB SERVER](https://tomaz.lovrec.eu/posts/graceful-server-restart/)
进行整理和归纳。

## 问题
因为 golang 是编译型的，所以当我们修改一个用 go 写的服务的配置后，需要重启该服务，有的甚至还需要重新编译，再发布。如果在重启的过程中有大量的请求涌入，能做的无非是分流，或者堵塞请求。不论哪一种，都不优雅~，所以slax0r以及他的团队，就试图探寻一种更加平滑的，便捷的重启方式。

原文章中除了排版比较帅外，文字内容和说明还是比较少的，所以我希望自己补充一些说明。

## 原理
上述问题的根源在于，我们无法同时让两个服务，监听同一个端口。
解决方案就是复制当前的 listen 文件，然后在新老进程之间通过 socket 直接传输参数和环境变量。
新的开启，老的关掉，就这么简单。


## 代码大概
我们可以将这个进程划分成为3部分
- 初始化 listen file（监听某个）
- 拿着 listen file 开启 http service
- 监听外部信号

可以看到 serve 是我们的核心函数
```
func serve(config srvCfg, handler http.Handler) {
	// 省略代码...
	cfg.ln, err = getListener() // 初始化 listen file
    // 省略代码...
	srv := start(handler) // 拿着 listen file 开启 http service
    // 省略代码...
	err = waitForSignals(srv) // 监听外部信号
    // 省略代码...
}
```

## 获取 listen


## 监听信号


## 父子间的对话