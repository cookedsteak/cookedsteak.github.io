---
layout: post
title: 玩玩微服务
category: 技术
keywords: golang,k8s,docker,微服务,后端
comments: false
---

## 忍不住
一直看人家面试要求 K8s + docker，然而这个经验对于一般的公司来说是十分宝贵的。
（能有大场景的公司能有几个啊？捂脸）
但是好奇心还是趋势我去实现一个最最最最简单的 k8s + docker 提供服务的场景。
所以这篇文章就要从 0 开始搭建这么一套服务。

## 准备
- docker
- k8s
- golang
- grpc

预备概念：
大家有时候会把 docker 看做是虚拟机，这个没有问题，但是我想说的是：一个“容器”，实际上是一个由 Linux Namespace、Linux Cgroups 和 rootfs 三种技术构建出来的进程的隔离环境。严格意义上，docker 并不等同于 VM。

在安装 kubernetes 的时候，请使用阿里云的镜像安装。什么？你有 VPN？
没用的朋友，安装过程中会在 容器中再行进行镜像的拉取，除非你修改安装脚本 设置 http proxy，否则还是会被墙的。

grpc 一般结合 protobuf 更加美味哦。当然 grpc 还是建立在 http 基础上的通讯框架。

## 开始
最简单的场景：
我们想象有三个服务，