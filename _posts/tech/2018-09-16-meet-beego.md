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
MVC很基本的结构，另外还有一个 config 文件夹存放配置。
一个 routers 文件夹定义路由。


## 作为一个服务
舍弃了 views，会多一个 swagger 的文件夹，因为 beego 内建用 swagger 作为 api 的说明文档工具。

## 控制器
真的是能够从 beego 中嗅到弄弄的 webapplication 气味。controller 一般都会自己定义一个 BaseController，然后在里面塞入一些公用的方法。包括 Prepare 机制，很好解决了代码冗余问题。

## Session
本次使用的项目我放弃了 jwt 的身份认证方式，改为了传统的 session cookie。另外惊讶的是，beego 对于session 的处理很方便。就是要根据 session 存储方式每次 `import _` 对应的驱动包有点麻烦。

## BUG
在编写过程中发现一个 15年到现在的 bug，也是一个用一个不优雅的方式规避的 issue，[这里](https://github.com/astaxie/beego/issues/1110)。
所以我也按照提示在init函数中手动gob.Register了。