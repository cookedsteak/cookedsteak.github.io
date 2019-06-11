---
layout: post
title: RPC ? gRPC? Protobuf ?
category: 技术
keywords: 微服务,框架,rpc,远程调用
comments: false
---

其实说起 grpc 和 rpc，我至今还是会有知识边界上的模糊。

首先明确一点，RPC 是远程过程调用。

rpc 只是一种思路，你可以实现你自己的 rpc 框架，而且 rpc 并没有限制你在那一层去实现它，不过一般都在传输层以上。

而 grpc 呢，就是 google 自己建立在 http2 协议之上写的一套 rpc 框架。

虽说 grpc 的序列化可以不基于protobuf，但是市面上大家用的都是配合着 protobuf 出现的。

<!--more-->

## Protobuf 做了什么？

