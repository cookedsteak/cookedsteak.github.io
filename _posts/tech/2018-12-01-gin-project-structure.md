---
layout: post
title: gin项目的经验
category: 技术
keywords: golang,编程,gin,经验,框架,后端
comments: true
---

## About GIN
gin 是一个 golang 的 web application/service framework。
如果你知道 beego，那也应该对 gin 有所耳闻。
和 beego不一样的是，gin 主打的是轻，意味着很多组件 gin 自己并不具备。但是也正是因为这样，才让 gin 框架下的 application 变化多端。

## 目录结构
实际上 gin 的项目并没有一个比较官方的目录结构，你觉得怎么方便怎么来，或者说，目录结构更遵循一般 golang 项目的规范，结合 golang 本身的一些特点。
```
- config  // 配置文件
- controllers // 控制器
- lib     // 通用函数库
- middlewares // 中间件
- models  // orm 模型层
- routes  // 路由
- test    // 测试
- main.go // 主入口
```

## 配件

### OS.GetEnv
.env 作为配置文件
不论是服务还是应用都会有配置需求，之前我会选择其他第三方配置库，但是我最近觉得 os 包自带的 Getenv已经够用了，所有的配置都在 .env 文件中。

### XORM
最常用的当然就是 orm 组件啦。这里我比较喜欢 xorm，不论是从运行效率还是编写手势上。
*其实在编程过程中，最长遇到也是稍微绕脑子的是三表关系查询：
A，B，A_B_rel
这种情况直接使用 sql 会比较方便，然后将结果 decode 到 model 中。


### 日志
运行过程中的错误，我一般使用 sirupsen/logrus 这个库。
他在 golang 自己的 log 上做了封装，还没研究透。

缓存-redis，redis是我自己在 go-redis 基础上做了个封装，支持连接池，现在普通的操作足够用了。