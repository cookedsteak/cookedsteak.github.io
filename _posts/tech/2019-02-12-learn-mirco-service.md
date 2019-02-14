---
layout: post
title: 玩玩微服务1（持更）
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
- golang 1.11+

### 关于 Docker
大家有时候会把 docker 看做是虚拟机，这个没有问题，但是我想说的是：一个“容器”，实际上是一个由 Linux Namespace、Linux Cgroups 和 rootfs 三种技术构建出来的进程的隔离环境。严格意义上，docker 并不等同于 VM。

### 关于 kubernetes 
在安装 kubernetes 的时候，请使用阿里云的镜像安装。什么？你有 VPN？
没用的朋友，安装过程中会在容器中再行进行镜像的拉取，除非你修改安装脚本设置 http proxy，否则还是会被墙的。

说实话，关于 k8s 的概念完全可以另开一篇文章了。
但是我尽可能用最少的语言白话一下关键的概念点。

首先你要明白，k8s 最终操作的是 docker。为了让好多好多 docker 管理起来更加方便，也易于伸缩。
把 k8s 想象成一个大箱子，里面会有不同大小的篮子，有哪几种篮子，颗粒度分别是怎样的呢？

#### Pod
对于 k8s 来说，他看得见的最小单位就是 pod。

#### Deployment


#### Service


## 试验代码
大家都是Hello World老手了，我们简单过一下:
目录结构如下
```
├── hello
   ├── Dockerfile
   └── main.go
```

main.go 实现一个简单的 http server
```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(r http.ResponseWriter, r2 *http.Request){
		r.Write([]byte("Hello World"))
	})

	fmt.Println("Start listening...")
	http.ListenAndServe(":8080", nil)
}
```

我们还需要 `go mod init` 一下，因为我们没有什么需要依赖的第三方库，所以不会生成 `go.sum`，我们手动创建一个空文件。

## 写Dockerfile

关于 Dockerfile 需要的看[【这里】](http://seanlook.com/2014/11/17/dockerfile-introduction/)。

普通的go程序已经对我们没有吸引力了，我们现在要将他封装成一个 container。
为了保证 container 的精简，我们最后肯定是只在原始镜像中放一个可执行文件，以及运行该程序的最小配置环境。
可别动不动就用上百 MB 的全镜像做基镜像啊，除非你家开硬盘厂的（捂脸）。

感谢[Pierre Prinetti](https://medium.com/@pierreprinetti)的文章，给我们展示了一个完整的基础镜像的构建过程。
来上代码：
```
# 第一阶段的 image ===========
# 设定 Go 的版本
ARG GO_VERSION=1.11

# 创建一个可执行镜像，基础为 alpine, alpine自带包管理
# 并明命名为 builder
FROM golang:${GO_VERSION}-alpine AS builder

# 创建一个没有特权的用户和用户组，这个很重要
# 否则程序会没有权限执行

RUN mkdir /user && \
    echo 'nobody:x:65534:65534:nobody:/:' > /user/passwd && \
    echo 'nobody:x:65534:' > /user/group

# 通过 alpine 的包管理工具 apk
# 添加 ca-certificates 工具，用来做 https 通讯的, 以及 git
RUN apk add --no-cache ca-certificates git

# 设置 go 的命令行环境参数
# CGO_ENABLE=0 使用静态连接的编译
# GOFLAGS=-mod=vendor 强制在 vendor 文件夹中寻找依赖
# ENV CGO_ENABLE=0 GOFLAGS=-mod=vendor

# 设定接下来的 RUN 的执行目录
WORKDIR /src

# 先获取需要的依赖，可以加速之后的构建速度
COPY ./go.mod ./go.sum ./
RUN go mod download

# 复制代码哟
COPY ./ ./

# 静态连接方式编译程序
RUN CGO_ENABLED=0 go build \
    -installsuffix 'static' \
    -o /main .

# 第二阶段的 image ===========
# scratch 比 alpine 更轻
FROM scratch AS final

# 从第一个 image 导入用户和组设置
COPY --from=builder /user/group /user/passwd /etc/

# 将证书文件也拷贝过来
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# 直接将可执行文件也拷贝过来
COPY --from=builder /main /main

# 暴露我们需要的 8080 端口
EXPOSE 8080

# 以 nobody 的名义执行下面的指令
USER nobody:nobody

# main 走起
ENTRYPOINT ["/main"]
```
准备好了 Dockerfile 后，我们需要 build 一下
> go build -t hello-go .

我们将最终的镜像取名为 `hello-go`
完后发现整个过程没有报异常并且 build 成功。
我们使用`docker images`命令查看，会发现多了两个镜像，
一个是没有名称的，也就是我们在第一构建阶段的镜像。还有一个是我们的 hello-go。

我们需要运行这个容器来验证一下：
> docker run --name hello-go-1 -p 8080:8080 -d hello-go

用 `docker ps` 查看已经运行起来了，我们访问 127.0.0.1:8080，发现页面已经输出了 Hello World。
到此，我们构建一个 go应用容器成功。 

## 不过等等

实际情况真有这么简单吗？应用什么也不需要，直接裸跑？
别做梦了！

现实是，一个应用不光会使用很多的第三方代码库，还会和其他服务一起协同，比如数据库啦，缓存啦，队列啦，blablablah...

那好，我们就来改进下我们的场景：
除了输出内容外，我们还需要将接受到的数据存入到 Redis 队列中。
既然我们做成了容器，那就是为了能够更好的伸缩，所以我们的这个新应用会多个容器同时运行。

如下图：



## *参考
- http://seanlook.com/2014/11/17/dockerfile-introduction/
- https://medium.com/@pierreprinetti/the-go-1-11-dockerfile-a3218319d191
- 