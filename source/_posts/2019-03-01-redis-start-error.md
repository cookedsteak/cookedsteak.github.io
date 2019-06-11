---
layout: post
title: redis小坑
category: 技术
keywords: redis,缓存,后端
comments: false
---

## 情况
正常启动 redis 没有问题，
但是指定了 redis 配置文件后，启动不起来，
去 /var/log/redis 查看日志，发现 `Cannot assign requested address`
应该是配置的问题，所以进入看配置，发现 bind 127.0.0.1，
然后注释 bind。就可以了。
那这个 bind 是什么意思呢？
> redis.conf中的bind功能并不是想象中的限制IP访问，而是绑定本机IP:端口
bind设置了监听哪个IP，与apache中的listen一样的功能
所以，client 与 server不在同一内网的话，无法通过bind配置项来实现安全连接

## LPOP 占用 CPU 过高
没错，在 go-redis 中 LPOP 会导致 CPU 的高占用，原因是：

> Using LPOP causes the goroutine to busy wait (constantly sampling the socket to see if there's a change) whereas using the blocking call causes the goroutine to enter a waiting state thus relinquishing its resources till the socket gets a reply from redis. It's similar to the difference between a spin lock and a regular mutex.

也就是说，一般 LPOP 会随着一个 for 循环的 goroutine 出现，而这样的代码会导致 goroutine 一直在忙等待（不断对套接字进行采样以查看是否有更改）。是不是可以理解为不断读取 socket，阻塞后放弃现有资源，redis 有反应了重新获取资源。

解决办法有两种，一种是 BLPOP，还有一种是手动怎家一个 time Sleep。