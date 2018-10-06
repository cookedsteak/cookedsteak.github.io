---
layout: post
title: React备忘
category: 技术
keywords: react,前端,nodejs,javascript
comments: true
---

### 概述
让前端写起来飘飘然，我想这应该就是react开发时候的感觉了吧。
[React](https://reactjs.org)是个啥？可以去官网看看，说得很清楚。
我选择React进行学习的一个原因是预备知识相对少，可以不用熟悉过多的dom属性（类vue中的），只要知道js，html，css的知识就够了。

一个纯粹的react-app是没有后端的任何服务、接口或者逻辑的。
官方会推荐给你一些工具链，来解决你实际开发中遇到的问题，比如：
- 文件个组件越来越多
- npm的第三方包
- 一些代码错误的预处理
- 开发环境中实时编译修改后的文件（js/css）
- 优化生产环境下的代码输出

上面这些问题还是挺常见的，特别是我很久很久以前徒手管理js，css...
所以呢，会有一些推荐：
- 如果你是学习react-app，或者是做单页应用。可以直接用[Create React App](https://reactjs.org/docs/create-a-new-react-app.html#create-react-app)
- 如果你要开发一个带有后端服务的app，建议[Next.js](https://reactjs.org/docs/create-a-new-react-app.html#nextjs)
- 如果你是开发一个静态的内容导向的app，使用[Gatsby](https://reactjs.org/docs/create-a-new-react-app.html#gatsby)

我们就先从最简单的 create-react-app开始。

### 入口
直接npm安装 create-react-app
一个biu准的react-app，会出现以下的文件结构

入口 index.html, index.js

需要在js文件中引用两个基本的模块：React, ReactDOM

### 组件与函数，足矣
