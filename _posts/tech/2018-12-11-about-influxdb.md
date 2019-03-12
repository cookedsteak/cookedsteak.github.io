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

但是，我们仔细看到存入的 value 类型。其实对于时序数据而言，string 和数组（其实也是 string）的数据对于观察数据变动情况并没有卵用。所以，最好的方式是把一个你要检测的不能再分割的数值插入到 influxdb 中。
像是这样：
![correct](/assets/img/influxdb/correctdata.png)

## 读取
其实在 influx 提供的 client 端读取数据是十分方便的，因为语法和传统的 sql 没有太大区别。同时，influxdb 本身还提供很多实用的函数。

不过我们还是要用 golang 去做取数据的交互的，接下来就看看我是怎么写的：
```
// cmd就是我们的原生查询语句
// client 是官方提供的 influx client
func QueryDB(cli client.Client, cmd string) (res []client.Result, err error) {
	q := client.Query{
		Command:  cmd,
		Database: "monitor",
	}
	if response, err := cli.Query(q); err == nil {
		if response.Error() != nil {
			return res, response.Error()
		}
		res = response.Results
	} else {
		return res, err
	}
	return res, nil
}
```
由于我们使用的是 influxdb 1.7 的版本，所以 client 包在[【这里】](https://github.com/influxdata/influxdb1-client)

可以看到，其实拿数据的方法还是挺蠢的，就是直接 raw sql。所以按照官方提示，我写了一个`QueryDB`的方法，用来执行我们的查询语句。

这里需要注意的，由于我们拿到的是一个 Result 数组，在判断数组有没有结果的时候会有点特别：
```
if len(res) > 0 && len(res[0].Series) > 0 {...}
```
我们判断了结果的长度，以及结果第一个元素的 Series的元素。


## 展现
只要我们有了时间维度和数据维度的两个值，什么图标插件，尽管来。不过我看很多运维会使用到 Grafana 来显示。

## 问题
在运行了一段时间程序后，你会发现一个问题，服务器上 influxdb 所监听的 8086端口，很出现很多 TIME_WAIT 状态的连接，尽管我程序中已经主动 close也没有用。我在网上查了下：
[InfluxDB TCP 连接数过多问题](https://www.jianshu.com/p/84015f4ecc1d)

需要知道，主动关闭的一方才是会拥有 TIME_WAIT 状态的一方。
所以我们可以知道是 influxdb 主动关闭的连接。

并且，influxdb 在设计的时候，主动要求请求方在请求完毕后关闭连接。因为 go 会复用连接，这会导致一些问题。

另外，TIME_WAIT状态其实在过了 2MSL 后的时间也会自动释放，所以我们还要在服务器上设置这个 2MSL 到一个合理的数值。
查看 2MSL
> cat /proc/sys/net/ipv4/tcp_fin_timeout

配置文件位置在：`/etc/sysctl.conf`
```
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_fin_timeout = 30
```
可以修改上面的内容，没有的话就新增

- net.ipv4.tcp_syncookies = 1 表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；
- net.ipv4.tcp_tw_reuse = 1 表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；
- net.ipv4.tcp_tw_recycle = 1 表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。
- net.ipv4.tcp_fin_timeout 修改系統默认的 TIMEOUT 时间

然后执行 /sbin/sysctl -p 生效。

## 参考
- https://jasper-zhang1.gitbooks.io/influxdb/content/Introduction/getting_start.html
- https://www.lsproc.com/post/close-timewait-connection