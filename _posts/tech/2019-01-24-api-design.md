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
我们暂且就称接口为 接口 A 吧。接口 A 通过参数获取可用房间和时间，请求报文如下：
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