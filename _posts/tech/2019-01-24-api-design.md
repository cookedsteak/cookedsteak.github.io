---
layout: post
title: 接口设计的一些总结
category: 技术
keywords: api,interface,后端,接口
comments: false
---

> 接口的种类有好几种，http、api、RPC、RMI、WebService、RESTful
> 这篇主要总结 http 接口和 api 接口

## api接口请求与返回
> 参考 [How NOT to design APIs](https://blog.usejournal.com/how-not-to-design-restful-apis-fb4892d9057a) 进行总结

作者的朋友的项目正在使用 [Beds24](http://beds24.com/) 这套系统，这套系统主要就是用来做预定的，连接的是Booking\AirBnB上的房源信息。而这个项目的功能就是从一些订房平台上获取可供预定的房间和日期。

但是这个提供的接口服务存在着很多问题，所以作者就拿他作为反面教材愉快地吐槽了一番。

我们先来看一下一个典型的getAvailabilities接口，接口连接[在这里](https://www.beds24.com/api/json/getAvailabilities)。
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
### 3.数据结构

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

### 4.错误处理
> 万能的 200

如果你知道我在说什么的话，你应该也经历过这样的问题~
我们看看 Beds24 接口的错误返回码：

![errorcode](/assets/img/apipro/errorcode.png)

他们定义了一套自己的错误码和错误返回信息。但是这些都是封装在200返回码的 Response 中的。

这里我想说一下，我不太同意原作者的观点，这也是国内和国外的风格差异。

遵循开发效率优先的原则，我认为错误处理的返回应该
`对外 API，统一返回200，并注明错误在自定义 errorCode 中`。这样的好处是，API对接方可以很容易区分是连接层的问题还是业务层的问题。
而`内部接口竟可能使用 HttpStatusCode`，前端同学可以更方便获取异常。

### 4.调用须知
原文中，作者对接口调用方给到的调用建议有些意见，并不涉及太多技术细节，只是提出了接口设计原则，设计要求和思路。
我们就来简单过一下...

调用须知是接口提供方给到的一些调用意见和规范，我们看下 Beds24的调用须知：

1. Calls should be designed to send and receive only the minimum required data.
2. Only one API call at a time is allowed, You must wait for the first call to complete before starting the next API call.
3. Multiple calls should be spaced with a few seconds delay between each call.
4. API calls should be used sparingly and kept to the minimum required for reasonable business usage.
5. Excessive usage within a 5 minute period will cause your account to be blocked without warning.
6. We reserve the right to disable any access we consider to be making excessive use of the API functions at our complete discretion and without warning.

我们大概分析下，1和4的意见还是挺在理的，尽可能保证请求数据的最小化，合理化。

> 第二条：每次只允许单个 api 的请求，只有当上一个请求结束后才能开始下一个请求。

我觉得应该是文档好久没更新了...在当下并发与并行横行的年代，这样的接口限制有点过时。如果你是在搭建REST API，那记住，这个API是无状态的。这也是 REST API 在云应用中被广泛使用的原因，调用者无需在意是否一次一请求，一旦程序出现问题，或者上一个请求超时、异常，完全可以立即重新请求。要求调用者每次只能请求一次也是一个无理要求。

> 第三条：复合请求每次要排队，每次间隔几秒钟。
> 第五条：五分钟内过高的请求频率会导致账户冻结。
> 第六条：接口提供方保留冻结接口使用者的账户的权利。

有点霸王条约，而且条件说明不清晰，包括冻结条件，同时在接口请求限制上有点严格。

### 5.文档展现
推荐使用接口展示的工具，[Swagger](https://swagger.io/)或者[Apiary](https://apiary.io/)。
Beds24的接口文档可谓是十分原汁原味了，没有主要的 index 引导，很容易让开发者看得云里雾里。

### 6.安全
这里原作者主要吐槽了 Beds24 接口不需要鉴权也能够调用的 BUG。
既然说到了安全，我们就稍微延展一下。

如今常见的 API 调用鉴权方式主要有以下几种：
- Basic Authorization
- Oauth Token
- Session
  
#### Basic Authorization
鉴权的方式相对简单，就是在 Http Header 增加经过base64运算后的 username + password。相信你能够发现，这样的鉴权方式没有过期时间，传输内容安全验证的措施，所以如果你的 api 需要上述两个方面的保证的话，不建议使用这种鉴权方式。

#### Session
通过一个唯一的 key，验证在共享内存中的用户状态。单实例应用常用的解决方案。唯一的问题在于，共享内存中的隔离机制，因为是共享 session 的，所以 session 的读取安全性很重要。

#### Oauth Token
不同与 Session, Oauth Token（下面简称 token）不需要在共享内存中记录用户状态。token 本身就会包含用户的基本信息、权限范围、有效时间，当然这些都是经过加密的。调用者将 token 放在 http header 中进行请求。一般提供 Oauth 验证方式的接口，都会附加一个类似刷新 refresh token 的接口，用来解决 token 过期的问题。

---

提供一个优质、可用、易读的 API 也是每个接口提供方的责任。记得前段时间Ant Design的彩蛋事件，所引出的“开源即责任”，更何况是收费模式下的接口服务呢？
文章末尾，作者好心劝告了大家，如果遇到类似Beds24的接口，请远离，除非这个接口提供的服务是不可替代的，因为这样的接口会让你投入更多的理解纠错成本。当然，我们作为开发者，自己也要谨记在设计 API 的时候不要出现上述的一些问题。

### *相关
[RESTful API Design. Best Practices in a Nutshell.](https://blog.philipphauer.de/restful-api-design-best-practices/)