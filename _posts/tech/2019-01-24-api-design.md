---
layout: post
title: 接口设计的一些总结
category: 技术
keywords: api,interface,后端,接口
comments: false
---

> 接口的种类有好几种，http、api、RPC、RMI、WebService、RESTful
> 这篇主要总结 http 接口和 api 接口

## http接口的命名
> 参考 [RESTful API Design. Best Practices in a Nutshell.](https://blog.philipphauer.de/restful-api-design-best-practices/) 进行总结

## api接口请求与返回
> 参考 [How NOT to design APIs](https://blog.usejournal.com/how-not-to-design-restful-apis-fb4892d9057a) 进行总结

作者的朋友的项目正在使用 [Beds24](http://beds24.com/) 这套系统，这套系统主要就是用来做预定的，连接的是Booking\AirBnB。而这个项目的功能就是从一些订房平台上获取可供预定的房间和日期。
接口连接[在这里](https://www.beds24.com/api/json/getAvailabilities)。
我们暂且就称为接口A。接口 A 通过参数获取可用房间和时间，所有可选参数如下所示：
```
{
    "checkIn": "20151001",
    "lastNight": "20151002",
    "checkOut": "20151003",
    "roomId": "12345",
    "propId": "1234",
    "ownerId": "123",
    "numAdult": "2",
    "numChild": "0",
    "offerId": "1",
    "voucherCode": "",
    "referer": "",
    "agent": "",
    "ignoreAvail": false,
    "propIds": [
        1235,
        1236
    ],
    "roomIds": [
        12347,
        12348,
        12349
    ]
}
```
我们就来细细评鉴一下这个参数中有哪些不合理的地方。
---
### 1.日期

我们可以看到，在请求报文中，`checkIn`，`lastnight`，`checkOut`，都使用了黏在一起的年月日形式，YYYYMMDD，虽然这种形式的可读性也不差，但是对于跨语言解析的便利性上就不好说了。
其实作为时间，用 [ISO8601](https://www.cl.cam.ac.uk/~mgk25/iso-time.html) (YYYY-MM-DD)，这样的通用标准形式会更加便捷。不论是从 encode 还是 decode 来说，比如 javascript 就能用 `Date.parse(YYYY-MM-DD)` 直接解析出所代表的 unix timestamp。

### 2.ID与数字

如果一个数字代表是 ID，即用来描述对象唯一的数字。那么最好是用 string 类型的，比如请求结构体
`roomIds`，`propIds`。

如果一个数字是用来描述数量或者值的，那不要使用 string，否则会有歧义并且不好计算。如上头的`numAdult`，`numChild`。

虽然上面数字与 ID 的区分并不是强制要求，但是注意，不论你使用哪一种形式，保持统一。你可以看到：`roomId`和`roomIds`，明明只是一个是另一个的集合，缺使用了两种表示形式，这不免让人误解。

---
在说完了 Request 部分，我们看下返回的 Response。
我们按照
```
{
    "checkIn": "20190501",
    "checkOut": "20190503",
    "ownerId": "25748",
    "numAdult": "2",
    "numChild": "0"
}
```
作为请求，得到了如下的 Response：
```
{
    "10328": {
        "roomId": "10328",
        "propId": "4478",
        "roomsavail": "0"
    },
    "13219": {
        "roomId": "13219",
        "propId": "5729",
        "roomsavail": "0"
    },
    "14900": {
        "roomId": "14900",
        "propId": "6779",
        "roomsavail": 1
    },
    "checkIn": "20190501",
    "lastNight": "20190502",
    "checkOut": "20190503",
    "ownerId": 25748,
    "numAdult": 2
}
```
---
### 结果结构

我们用心体会一下这个 response 是干嘛的...
返回的内容包括了两块：
1. 客户的房间资产列表
2. 我们请求的参数以及相应的检索返回

如果你是前端的话，肯定心里已经开始 mmp 了。
这个 response 把所有数据都捏在了顶层数据结构中。如果我们只想获取房间列表信息的话，我还必须遍历所有数据，并自己做一个筛选？？这显然十分不合理，我们再来看看房间列表。
```
"14900": {
        "roomId": "14900",
        "propId": "6779",
        "roomsavail": 1
    },
```
他把 `roomId` 作为了map的 key。这看上去有点重复，我们很少会在 json 传输过程中使用把关键结果数据作为key的字典类型（dictionary）。对于数据处理方来讲，只对 value
中的数据进行处理是最方便的。你额外来一个关键信息，还是作为 key，要不是 value 中包含了 `roomId`，真的很 cd。

另外，我们看到`roomsavail`，`roomsavail`在值为0，和值为1的时候的数据类型是不一样的（应保持一致），这个也需要注意。

既然我们吐槽完毕，那就看看推荐的数据结构应该是怎么样的：
```
{
    "properties": [
        {
            "id": 4478,
            "rooms": [
                {
                    "id": 12328,
                    "available": false
                }
            ]
        },
        {
            "id": 5729,
            "rooms": [
                {
                    "id": 13219,
                    "available": false
                }
            ]
        },
        {
            "id": 6779,
            "rooms": [
                {
                    "id": 14900,
                    "available": true
                }
            ]
        }
    ],
    "checkIn": "2019-05-01",
    "lastNight": "2019-05-02",
    "checkOut": "2019-05-03",
    "ownerId": 25748,
    "numAdult": 2
}
```
增加了 `properties` 数据块，很明确区分了数据内容，方便取用。
