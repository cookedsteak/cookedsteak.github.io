---
layout: post
title: beego 奇怪的问题（持更）
category: 技术
keywords: 后端,beego,golang,原理
comments: true
toc: true
---
今天遇见一个奇怪的 beego 问题，从同事那边 copy 过来的代码只会404，似乎router没有起效。
唯一的区别是我用的 mac，他用的 windows。然后自己本地新建一个 bee new 项目就没有问题。很难受这个问题。
所以这里记一下 beego 的一些源码流程。

从 beego.Run()开始进入，

文件 beego.go
-> initBeforeHTTPRun 初始化一些钩子，啥事钩子？比如你拿到 httpStatus Code 404了，直接勾出个 404 not found page，
你也可以理解为一种回调

-> AddAPPStartHook 会注册若干个 hookfunc 类型的函数，放到全局的内部变量 hooks 中。

1. MIME 类型注册，用来告诉浏览器如何处理文档 *go/mime
2. 默认的错误处理函数 *beego/hooks
3. 注册 session *beego/hooks 
   这里就涉及到到了 beego包中的一个 config 文件，BConfig 结构体必要的。
   拥有 init() 函数
4. 注册视图，模板路径
5. 注册 beeadmin 后台函数
6. Gzip 压缩配置

-> BeeApp.Run() 运行真正的 BeeApp，读配置，注册 Handler，判断 HTTPS 还是 HTTP
handler 中结构：handlers -> ControllerRegister -> routers -> map[string]*Tree (beego/tree) -> prefix + fixrouters -> wildcard -> leaves

注册了自己的 Server函数，然后会被调用，去匹配已经注册的 register。