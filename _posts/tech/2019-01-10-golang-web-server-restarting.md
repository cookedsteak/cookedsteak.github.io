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

## 原理
问题的根源在于，我们无法同时让两个服务，占用同一个端口。
所以我们就从这里入手，发现两个方案：
- 设置 SO_REUSERPORT 参数
- 基于原 socket，复制一个相同的子进程，并在完成切换后关闭原子进程

每个进程都会有两部分，一个是监听，通过监听来判断是不是需要把 listener 文件给到新进程，同时关闭自己。
另一个就是 server 主要的业务处理逻辑。
