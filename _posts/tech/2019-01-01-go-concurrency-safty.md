---
layout: post
title: golang并发安全试验
category: 技术
keywords: golang,go,并发,channel,routine
comments: true
toc: true
---

之前有聊过 golang 的协程，我发觉似乎还很理论，特别是在并发安全上，所以特结合网上的一些例子，来试验下go routine中 的 channel 与 select 的妙用。

首先我们来考虑一个场景--红包。
这里有个跨年红包，金额100，我们让阿猫，阿狗同时来抢这个红包，知道这个红包余额变为0。

