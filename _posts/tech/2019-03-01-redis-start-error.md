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