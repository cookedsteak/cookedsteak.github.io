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

## 配件