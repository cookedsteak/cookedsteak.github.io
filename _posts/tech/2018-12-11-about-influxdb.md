---
layout: post
title: influxDB 初探
category: 技术
keywords: golang,influx,influxDB,经验,数据库,后端
comments: true
---

## influxDB

之前在 [gin项目的经验](./_posts/tech/2018-12-01-gin-project-structure.md)中，也稍微提到了 influxDB 作为一个业务数据记录的时序数据库。那这篇就展开谈一下 influxDB 的概念和具体项目使用。
关于influxDB的安装，可以去官网，写的很清楚。
因为我是用 influxDB 官方的 client 直连的，所以需要go get 一下。过程中遇到 bzr 安装的报错，
```
bzr: ERROR: Not a branch: "/Users/steak/Projects/go/pkg/mod/cache/vcs/ca61c737a32b1e09a0919e15375f9c2b6aa09860cc097f1333b3c3d29e040ea8/.bzr/branch/": location is a repository.
```
删除ca61c737a32b1e09a0919e15375f9c2b6aa09860cc097f1333b3c3d29e040ea8 这个 cache,重新 get 就好了。(路径因机而异)

## 栗子

我现在有一台工业设备 device1，我想要实时收集他的各个指标，比如：
- 设备状态，string 类型
- 当前坐标，float array 类型
- 设备轴温度，float 数据类型

我们该如何存储在 influxDB 中呢？
作为数据库，influxDB同样有 Database 的概念,
在每个 database 中，又有了 measurement，意味记录的测量对象。你可以用表的概念去代入理解，虽然两者使用略有差异。

### 1. 一个设备一个measurement
这个似乎是最常见的思路，一个设备一个作为一个测量对象，不论从语义上还是理解上都十分通顺。

### 2. 一个指标一个measurement
这样的存储结构，其实对于 influxDB 才是恰到好处的。
一来设备可以用 tag 作为索引，二来设备的指标还能随时增减，并且不影响之前数据的展示。