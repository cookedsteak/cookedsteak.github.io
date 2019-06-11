---
layout: post
title: 谈谈 istio 和 k8s
category: 技术
keywords: 微服务,框架,服务网格,servicemesh,istio,k8s
comments: false
---

说到 istio，就要说到 Service Mesh 服务网格。
说到服务网格，就要谈谈 Sidecar 模式。

Sidercar 是一种单节点，多容器的应用设计形式。

Kubernetes 支持微服务架构，使开发者能够抽象出一组 Pod 的功能，并且通过明确的 API 来曝露服务给其他开发者。Kubernetes 支持四层负载均衡，但是不能解决更高层的问题，例如，七层指标、流量分流、限速和断路。

<!-- more -->

**这里的四层负载和七层负载是什么？**

- 二层负载
  虚拟 mac 地址接受请求

- 三层负载
  虚拟 ip，接收请求后分发

- 四层负载
  ip + port 的方式接受请求，再转发到对应的机器。

- 七层
  为什么么有五六，因为五六在 osi 中没啥我们能操作的空间。
  应用层负载，http根据虚拟的 url在处理响应的请求。

```
|          | 四层负载均衡（layer 4） | 七层负载均衡（layer 7）                          |
+----------+-------------------------+--------------------------------------------------+
| 基于     | 基于IP+Port的           | 基于虚拟的URL或主机IP等。                        |
+----------+-------------------------+--------------------------------------------------+
| 类似于   | 路由器                  | 代理服务器                                       |
+----------+-------------------------+--------------------------------------------------+
| 握手次数 | 1 次                    | 2 次                                             |
+----------+-------------------------+--------------------------------------------------+
| 复杂度   | 低                      | 高                                               |
+----------+-------------------------+--------------------------------------------------+
| 性能     | 高；无需解析内容        | 中；需要算法识别 URL，Cookie 和 HTTP head 等信息 |
+----------+-------------------------+--------------------------------------------------+
| 安全性   | 低，无法识别 DDoS等攻击 | 高， 可以防御SYN cookie以SYN flood等             |
+----------+-------------------------+--------------------------------------------------+
| 额外功能 | 无                      | 会话保持，图片压缩，防盗链等                     |
```

其实，为什么 istio 能够做到四层以上呢？因为他基于了 envoy 这个通讯总线。

## 什么是 Envoy ?

Envoy 是专为大型现代 SOA（面向服务架构）架构设计的 L7 代理和通信总线。
可以理解为，他为每个应用服务创建一个独立的进程，这个进程负责通讯服务，就像每个人都有了一个手机一样。

原本我是想在基于 consul 的服务发现与注册上，采用一套 APIGATEWAY，用来做统一的服务入口，并与后端的 K8s 做对接。

虽然 go-micro 有自己的 micro API，但是性能一直是一个问题。