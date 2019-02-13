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
- k8s
- docker
- golang
- grpc

## 开始
最简单的场景：
我们想象有三个服务，