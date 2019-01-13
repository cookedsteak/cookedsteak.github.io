---
layout: post
title: golang并发安全试验-2
category: 技术
keywords: golang,go,并发,channel,routine,context
comments: true
toc: true
---

## 场景-抢红包
首先我们来考虑一个场景--红包。
这里有个跨年红包，金额100，我们让阿猫，阿狗同时来抢这个红包，直到这个红包余额变为0。
