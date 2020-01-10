---
layout: post
title: 关于鉴权
category: 技术
keywords: 鉴权,架构,网关,微服务,协议
comments: true
notshow: true
---

## 场景
脱离场景讨论一个技术实现方案是耍流氓。
鉴权作为一个非常常用的功能，渗透在各种架构，各种场景之中。

### 单体应用


### 分布式系统

### 微服务
一个大型的系统，有许多微服务组成。
微服务的鉴权方式会相对复杂，因为本身就是一个复杂系统。

#### 微服务之间是否需要鉴权？


## 方式

### 使用 Session
Session 使用一种简单的字符标识，结合客户端的Cookie，进行带有用户缓存状态的鉴权。

### 遵循 Oauth
Oauth是一种协议与约定，提供一种鉴权思路，使用第三方授权的 Token 信息进行鉴权。

### 使用 JWT
Jwt实现了以Token作为载体的客户端鉴权。
一致性的签名方式是：HMACSHA256。防止客户端对数据进行篡改。

在发送请求的时候，jwt被安置在 key为 Authorization 的 Header中，并在 `<token>` 
前怎家 `Bearer` + 空格 标识。

```
Authorization: Bearer <token>
```


### Http Basic Authorization
是一种原始简单的http鉴权方式，因为存在安全缺陷，所以不被推荐。
但是因为简单易用，所以可以用在对安全性要求较低的http请求中。


## 辅助

### IP白名单

