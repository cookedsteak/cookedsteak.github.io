---
layout: post
title: gin项目的经验
category: 技术
keywords: golang,编程,gin,经验,框架,后端
comments: true
---

## About A GIN work
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
不论是服务还是应用都会有配置需求，之前我会选择其他第三方配置库，但是我最近觉得 os 包自带的 Getenv已经够用了。
不过偶尔也有些小坑，比如处理.env 文件中的特殊字符，一般我会使用双引号括起来。



### XORM
最常用的当然就是 orm 组件啦。这里我比较喜欢 xorm，不论是从运行效率还是编写手势上。
*其实在编程过程中，最长遇到也是稍微绕脑子的是三表关系查询：
A，B，A_B_rel
这种情况直接使用 sql 会比较方便，然后将结果 decode 到 model 中。
另外，巧用数据库视图，视图有时候能够避免让你写很多麻烦的链式调用，而且更加可读。

在使用 model 的过程中，千万不要用一表一 model 的思路，而是一个数据对象一个 model。有时候你会看见有些 model 他的字段设置并不遵循数据库的字段声明。诶嘿，这样才够 orm 嘛~


### 运行日志
运行过程中的错误，我一般使用 sirupsen/logrus 这个库。
他在 golang 自己的 log 上做了封装，还没研究透。

### 业务数据日志
- influxDB
  
  时序数据库， made by golang。influxdb 对于数据的压缩率可谓是非常大的。influxDB + Grafana也是一种常用搭配（Grafana本身就很万金油）

- ES 套件
  
  elasticsearch + logstash + filebeat + kibana 是很多日志系统的标配。一来是包装好的功能十分强大


### 缓存
缓存-redis，redis是我自己在 go-redis 基础上做了个封装，支持连接池。但是 redis 库的版本比较旧，新版本写法会有区别。