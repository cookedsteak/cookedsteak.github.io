---
layout: post
title: 接触Beego
category: 技术
keywords: golang,beego,go,框架,后端
comments: false
---

## Beego
一直听闻Beego，自己在以前也尝试用过，可能因为感觉太重，所以后来也没有再去碰他，转而使用gin。
最近看见好多国内大厂都比较认可beego作为web开发框架，所以勾起了兴趣，重新体验一下。

## 安装
beego的文档还是十分详细的。
[beego文档](https://beego.me/docs/install/bee.md)
go 1.11 
需要安装的两个一个bee工具，另一个就是beego的源码了。

直接在bee文件夹下`make install`就可以了

## 项目结构
MVC很基本的结构都有 models, controllers, views(前后分离的话没有)，另外还有一个 config 文件夹存放配置。
一个 routers 文件夹定义路由。


## 作为一个服务
舍弃了 views，会多一个 swagger 的文件夹，因为 beego 内建用 swagger 作为 api 的说明文档工具。

## 路由
我用的最多的还是 beego.Router，方便好用，也好管理维护。
还有个需要注意的地方，在使用 RESTFUL 命名规则的时候，获取地址栏中参数（"my.com/user/:id"）的方式有两种：
1. controller.GetInt/GetString...(":id")
2. controller.ctx.Input.Param(":id")
注意两个获取都需要加上`:`号。

## 控制器
真的是能够从 beego 中嗅到弄弄的 webapplication 气味。controller 一般都会自己定义一个 BaseController，然后在里面塞入一些公用的方法。包括 Prepare 机制，很好解决了代码冗余问题。

## Session
本次使用的项目我放弃了 jwt 的身份认证方式，改为了传统的 session cookie。另外惊讶的是，beego 对于session 的处理很方便。就是要根据 session 存储方式每次 `import _` 对应的驱动包有点麻烦。

## ORM
其实我自己一直处于中立，也就是徘徊于使用与不使用 orm 之间。beego 的 orm 也算够好用，当然对于复杂的查询我更倾向于raw sql直接上。其实 orm 的难点永远不是你怎么用，而是你怎么设计 model，model 并不是单纯的表的抽象，而是你要用到的数据块的划分。

## BUG
在编写过程中发现一个 15年到现在的 bug，也是一个用一个不优雅的方式规避的 issue，[这里](https://github.com/astaxie/beego/issues/1110)。
所以我也按照提示在init函数中手动gob.Register了。

之所以会遇到这个 BUG，是因为我在 BaseController 中定义了一个存放当前用户的变量。

## *业务与抽象的取舍
其实很多时候会碰到这样的选择题：一个重复的业务逻辑，但是分隔开比较清晰，抽象的话虽然代码看上去优雅，但是容易耦合。这种情况，记住代码是为业务服务的。代码的优雅是一种个人的精神追求（太烂的代码可不是）。牺牲空间和一些工匠洁癖，来换取可维护性，我觉得这样才是可行的。