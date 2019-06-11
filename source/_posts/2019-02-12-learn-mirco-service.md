---
layout: post
title: 从微服务说起1（持更）
category: 技术
keywords: golang,k8s,docker,微服务,后端
comments: false
---

## 什么是微服务

我觉得老生常谈了，随便 google 一下就能得到很多解答。
微服务其实也是一种很早以前就提出的架构策略。

在几年前，我接触到的应用都非常小，所以对微服务的概念也不准确。
我以为，就是把后端的一体化应用拆分就行了，满足单个模块的弹性扩展，持续集成，业务。
但其实这个是不准确的。微服务应该是更加抽象的应用架构模式。不单单大业务的分割。

## 阿里巴巴的微服务中台

如果微服务在思路上是正确的，那我们是不是可以无脑微服务化呢？
答案肯定是否定的！
服务本身就存在这区别，所以微服务在整个系统的架构中，也会有区别。
比如阿里巴巴的微服务策略是，重中台，轻前台。

这是什么意思呢？

这里的前台，并不是前端，而是包含了前端的，面向用户的或者接入方的服务（可能包含了前端和后端）。
中台，其实就是脱离了业务，而是吧业务所涉及的功能抽象成了服务。
比如用户登录，下单，消息等...依托于阿里巴巴的大生态，这些服务被抽象到中台后，想要开辟一个新的业务场景变得十分简单，就像是搭积木一样。唯一要做的就是填补靠前台的胶水代码。

## 微服务与分布式

由于我是玩 Golang 的，所可能对 go 生态下的微服务架构比较熟悉。
微服务和分布式有什么关系呢？
有些人会认为，微服务不就是分布式吗？emmmm...我觉得这是错的。

分布式是一种系统架构，微服务是服务提供形势。
你可以认为，*分布式是微服务的一种展现形式*。

知乎上有个很经典的总结：

- 微服务：分散能力
- 分布式：分散压力

这样思考的话其实就很清楚了，两个原本没啥关系，但是这两个在一起能够达到 1+1>2 的效果，所以
一起出现的情况会很高。

## 关于 Docker

如果说微服务与分布式是一个发展纲领的话，那 CI&CD 就是对纲领的实践。
既然是实践，我们就需要高效。服务容器化自然而然就被照搬过来了。

大家有时候会把 docker 看做是虚拟机，这个没有问题，但是我想说的是：一个“容器”，实际上是一个由 Linux Namespace（资源隔离）、Linux Cgroups（资源限制） 和 rootfs（基本文件系统） 三种技术构建出来的进程的隔离环境。严格意义上，docker 并不等同于 VM。

## 关于 kubernetes

有了容器，跑不掉容器管理的。外面基本上都用 k8s。
首先你要明白，k8s 最终操作的是 docker。为了让好多好多 docker 管理起来更加方便，也易于伸缩。
把 k8s 想象成一个大箱子，里面会有不同大小的篮子，有哪几种篮子，颗粒度分别是怎样的呢？

接下来就沿着 k8s 做一下知识点的记录，我尽可能用最少的语言白话一下关键的概念点：

### 安装

我的安装策略是先安装 k8s 的基础组件：kubelet, kubeadm, kubectl。
然后通过 kubeadm 去阿里云拉取 k8s 的其他组件。

- kube-apiserver
- kube-scheduler
- kube-controller-manager
- etcd

可惜的 gcr.io 我们访问不了，所以只能曲线救国。
我们可以在安装完三个二进制组件后，使用

```sh
kubeadm init  —image-repository registry.aliyuncs.com/google_containers  —pod-network-cidr=10.244.0.0/16 —kubernetes-version v1.14.1 —ignore-preflight-errors=Swap
```

来从阿里云上的镜像仓库添加 k8s 容器。
注意后面的 —ignore-preflight-errors=Swap 选项，我们必须关闭 swap，才能顺利安装。

所以在执行上面的 `kubeadm init` 之前，我们还要改一下 k8s 的配置文件。

修改 `/etc/systemd/system/kubelet.service.d/` conf 文件

增加：

```sh
Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"
```

然后重启 kubelet

```sh
systemctl daemon-reload
systemctl restart kubelet
```

#### Pod

对于 k8s 来说，他看得见的最小单位就是 pod。
一个 pod 里可以放一个单独的容器，也可以放多个进程资源共享的容器。
总之，应该是提供服务的最小单位。

有了 pod 后，我们就可以通过水平扩展 pod，来增加服务性能输出。
那我怎么水平扩展？这个就是 Devlopment 的作用。

我扩展了以后怎么做 pod 的负载均衡，以及统一的外部 url 地址？
这个就是 Service 的作用。

#### Service

被打上标签的 pods，是 service 区分的依据。
一个 Service，启动获的外网地址后，便可以对外开始服务。
所以我们要将 pods 再抽象一层成为服务。

#### Deployment

Deployment资源可以自动迁移应用程序版本，实现零停机，并且可以在失败时快速回滚到前一版本
你可以看做是一个带有版本管理控制分发的 Service。

## 试验代码

在对 k8s 有了初步的概念后，就要用普通的代码试验一下了。

大家都是Hello World老手了，我们简单过一下:
目录结构如下

```sh
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
# 第二阶段一般不支持 用 RUN，因为镜像很小没有需要的一些命令
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
千万不要忘记了 最后的 `.`。

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

现实是，一个应用不光会使用很多的第三方代码库，还会和其他服务一起协同，比如数据库啦，缓存啦，日志啦，其他服务啦，blablablah...

那好，我们就来改进下我们的场景：
> 除了输出内容外，我们还需要将接受到的数据存入到 Redis 队列中。
既然我们做成了容器，那就是为了能够更好的伸缩，所以我们的这个新应用会多个容器同时运行。

所以问题就聚焦在了如何灵活使用配置。这里有两个方案：

1. 设置在 Dockerfile 中，添加 ENV xxx xxx
2. 在容器运行的时候，添加 -e 参数

我选择了后者，因为更灵活。
假设我们的 redis 服务运行在宿主机上，那么我们需要容器活期宿主机的地址，可以用这个变量 `-e xxx=host.docker.internal`

同时，我们的开发机，打包机，和生产机是要做完全区分的。
而一般的开发发布流程应该是：

1. 开发机提交代码到 git 仓库
2. 触发打包机 pull 代码并进行 在线的 build
3. 打包机生成镜像并 push 到私有的 docker 仓库
4. 生产机 pull 最新的镜像并 run 一个容器

其中，私有的 docker 仓库可以使用阿里云，免费的。
再发布的时候我们是需要打 tag的，docker tag 的作用就是区分镜像的版本。

## *参考

- <http://seanlook.com/2014/11/17/dockerfile-introduction/>
- <https://medium.com/@pierreprinetti/the-go-1-11-dockerfile-a3218319d191>
- <http://dockone.io/article/5132>