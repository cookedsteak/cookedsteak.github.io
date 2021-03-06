---
layout: post
title: 日志事件系统-1
category: 技术
keywords: 日志,kafka,elk,elasticsearch
comments: true
toc: true
---

## 概述
---
顾名思义，日志事件统计，这其实是分开来的，日志统计已经完成了设计并投入使用，但是还需要改进。
事件的统计缺少触发业务条件，触发机制可以实现。
诶嘿，我就是一眼相中了[Elastic Stack](https://www.elastic.co/)和[Kafka](https://kafka.apache.org/)美妙结合。<br>
![对](/assets/img/serious.png)

[官网指导](https://www.elastic.co/blog/just-enough-kafka-for-the-elastic-stack-part1)

## 须知
---
**来自的官网的推荐使用姿势**
> When To Use Kafka With The Elastic Stack?

> Scenario 1: Event Spikes

+ 场景1：事件钉子


日志数据和事件驱动数据几乎没有一个固定的，可预见的流量指标。想象一下这个场景：你在周五晚上升级了你的诶呸呸（有脑子的程序员一般不会这么干:)）。你的程序喜闻乐见地出了bug，然后报了一大堆的错误日志信息，然后搞瘫了你的日志系统。这种日志灾难在一般的多租户技术场景（可以理解为云服务场景）下也很常见的，比如游戏或者电商业务。像Kafka这样的存在，就是为了在上面的场景下缓解logstash和elasticsearch的压力。


> ![pic](/assets/img/kafka.png)

**来来来，我们看图：**
上面的传输过程，我们分成了两个阶段 “Logstash Shipper”阶段 & “Logstash Indexer”阶段。Shitter哦不...Shipper阶段对于logstash来说就是不断地从各个数据源灌数据到Kafka的topic。因此，Shipper是作为生产者的角色。
那在另一端担任consumer的logstash，有条不紊地按照自己的节奏处理着Kafka中的数据，很多时候会是像gork，dns查询，建立es索引这种复杂的转换工作。

![pic2](/assets/img/kbeats.png)


既然Shipper中的logstash担任的是传输者的身份，我们大可不必杀鸡用牛刀，Beats系列的服务完全可以胜任data source 到 Kafka topic的任务。
比如我们可以将nginx中的日志用filebeat推送到Kafka topic中，完成Shipper阶段。

> Scenario 2: Elasticsearch not reachable

+ 场景2：ES无法直接对接


另一个场景：有时候你会想要给多借点的es集群做一下升级，不得不重启所有节点。或者es挂了，有老长一段时间没有接收到日志。这两种情况，如果有Kafka的话就好办多了，不你会因为任何es本身的问题而丢失数据源推送过来的数据。因为他们都在未被消费掉的topic里头！同样的，你可以暂时挂起logstash对日志的分析，增加logstash节点，然后再开。方便的水平扩展功能也是es的特色之一。

**什么时候你不该用Kafka**

> Anti-pattern: When not to use Kafka with Elastic Stack

每用一样其他的东西都是会有成本的，Kafka也一样，如果用在生产环境中，你需要额外的监控，应对与告警。
在做集中的日志管理时，往往会有要实时传输日志到节点的需求，也有一部分对日志的实时性要求并不高的，那其实完全没有必要使用Kafka，filebeat够用了，只不过你在传输日志的时候需要做一下文件的拆分。其实在这种高延迟容许的情况下，我们就是把文件缓存当做了一个迷你的Kafka。

## 又来
时隔两年，重新开始 elk 之旅。

### 一个简单的方案
没有太多高配的机器，我们只需要跑一个 es，一个 kibana，一个 filebeat。
多个应用按照一定的格式生成文件日志，filebeat 不停采集，输出给 es。

但是注意，filebeat 的实时性我试下来不高。
所以如果对实时性有要求的可以程序直推 elasticsearch。

这里举一个我自己的例子：
监控端选择 efk，以 json 形式 RawLog 到文件，然后 filebeat 监听文件并推送至 elasticsearch。
注意 filebeat 不会做存量更新。
同时需要做 filebeat json 设置
```
  paths:
    # - /var/log/*.log
    - /Users/steak/Projects/go/src/bitbucket.org/gym3/mreceiver/logs/receive_log_*.log
    #- c:\programdata\elasticsearch\logs\*

  json.keys_under_root: true
  json.overwrite_keys: true
```


### 稍微复杂的场景
随着日志的量级不断上升，es可能会有一些处理瓶颈。
这时候引入队列是一种很好的削峰方式。我们可以选择 kafka。
（你也可以用其他，比如 redis，但是 kafka 是数据溢出的情况下是将数据持久化在硬盘的，redis会丢弃）