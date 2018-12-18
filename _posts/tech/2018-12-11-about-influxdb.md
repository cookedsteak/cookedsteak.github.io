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

我们选择第二个方法来存储数据。
influxdb中的数据推送需要两个初始化实例，
- Client
- BatchPoint
Client就是一个保持连接的客户端实例。
BatchPoint是需要推送数据的封装实例。

首先我们新建一个Client实例：
```
var Influx client.Client
// 一个很基本的单例模式
func GetInflux() client.Client {
	if Influx == nil {
		c, err := client.NewHTTPClient(client.HTTPConfig{
			Addr:     os.Getenv("INFLUXDB_HOST"),
			//Username: username,
			//Password: password,
		})
		if err != nil {
			log.Fatalln("Error: ", err)
		}
		Influx = c
	}

	return Influx
}
```
再来一个数据推送实例：
```
func NewInfluxBp(db string) client.BatchPoints {
	bp, err := client.NewBatchPoints(client.BatchPointsConfig{
		Database:  db, // 库名，很明显
		Precision: "s", // 表明了任何返回的时间戳的格式和精度，采用 rfc3339标准
	})
	if err != nil {
		Dolog(err.Error(), "influxdb_error")
		panic("influxdb_err")
	}

	return bp
}
```
#### 推送数据
有了推送条件后，我们就要用这两个实例开始推送数据了，我们也单独抽象一个方法：
```
func PushDataToInflux(deviceTag string, data map[string]interface{}) {
	tags := map[string]string{"device": deviceTag}
	c := GetInflux()
	bp := NewInfluxBp("monitor") // 创建了名为 monitor 的 database

	for k, v := range data {
		fields := map[string]interface{}{"value": v}
        // 一个k表示一个 measurement, tags 就是设备号
		pt, err := client.NewPoint(k, tags, fields, time.Now())
		if err != nil {
			Dolog(err.Error(), "influxdb_error")
		}
		bp.AddPoint(pt) // 增加 batchpoint 记录
	}
	// 用 client 实例把 bp 写入数据库
	if err := c.Write(bp); err != nil {
		Dolog(err.Error(), "influxdb_error")
	}

	// 断开数据库连接
	if err := c.Close(); err != nil {
		Dolog(err.Error(), "influxdb_error")
	}
}
```

在大概推送了几条数据后，我们来看看 influxdb 中存储的数据结构：
这些是我们按照一个指标一存的表
![m](/assets/img/influxdb/measurements.png)

我们select ALERT_CODE表看下
![t](/assets/img/influxdb/table.png)

时间+ 设备 id + 存储的值，结构还是很清晰的。

## 展现


## 参考
---
- https://jasper-zhang1.gitbooks.io/influxdb/content/Introduction/getting_start.html