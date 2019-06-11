---
layout: post
title: protoc 命令
category: 技术
keywords: 后端,golang,protobuf,rpc
comments: false
---

如果你准备选用 gRPC 作为服务调用方式的话，你可能会需要根据一个 proto 文件来生成一个 go 模板文件。

按照官方的安装指导，你会用到 protoc 这个命令。
> protoc --go_out=plugins=grpc:. /path/to/your/file

除了基本的 grpc 生成方式以外，我们还能用经过优化的生成方式：
比如我们 选用 gogofast
> protoc --gofast_out=plugins=grpc:. /path/to/your/file

为了让我们能够更方便使用微服务下的 grpc 调用，还有一个插件叫做 go-micro，
他的生成方式是：
> protoc --go_out=plugins=micro:. /path/to/your/file

这种生成方式能够更简单调用微服务，而不是使用 ip 地址。