---
layout: post
title: 说说康威定律与微服务
category: 技术
keywords: 架构,后端,技术
comments: true
toc: true
---

## 康威定律
康威定律是半个多世纪前就奠定微服务架构的理论基础。

我们来看看康威定律的核心观点：

> “organizations which design systems are constrained to produce designs which are copies of the communication structures of these organizations.”

翻译过来就是：系统设计的结构必定复制设计该系统的组织的沟通结构。

<!--more-->

想要搭建怎样的系统，就选择怎样的组织架构。
如果你想要的是一个微服务，那么就把大功能拆分开，把多个人组队成一个最小的执行单位。
从而达到小团队的子系统内聚，对外系统边界明确。
这样能够有效降低沟通成本，团队设计的系统也会合理高效。

而大型系统的持续优化，必须是容错和弹性的。
什么是容错和弹性，就是正式发生问题的可能性，并且能够容忍发生的问题。
不要指望一个大而全的系统。都要以合理的方式持续迭代。

如果把一个系统的设计、开发、集成、优化看做一个生态系统的话，生态系统追求的是平衡。
这也是为什么当我们把系统的开发、优化过程中精炼到了一定程度，所多出来的复杂度会慢慢偏向于运维和集成的部分。

从实践经验的角度来看，为了让设计高效，首要保证系统设计者之间的沟通高效。

## 沟通
人月神话中，对于沟通成本有一个公式：
> 沟通成本 = n(n-1)/2

即沟通成本随着人员的增加而指数级增长。所以我们可以说，对于一个团队项目的算法复杂度是 O(n^2)。
- 5个人的项目组，需要沟通的渠道是 5*(5–1)/2 = 10
- 15个人的项目组，需要沟通的渠道是15*(15–1)/2 = 105
- 50个人的项目组，需要沟通的渠道是50*(50–1)/2 = 1,225
- 150个人的项目组，需要沟通的渠道是150*(150–1)/2 = 11,175

从生物学的角度讲，我们的大脑只能维持我们一定数据量范围内的人际关系交流。
超出范围，会带来一系列的沟通问题。
人是复杂的社会动物（是因为我们自己看自己，所以觉得复杂，要是有高阶的生命可能还是会觉得我们是蝼蚁），
一般来说，一个大公司的组织管理问题，会被拆分为多个组织加以解决。

## 微服务

在了解了康威定律后，我们来看看他和微服务的异曲同工之处。
- 分布式服务组成一个系统
- 按照业务划分服务，而不是技术
- 自动化运维
- 容错
- 弹性优化

如果我们将上述几点映射到组织结构中的话：

- 分布式服务组成一个系统
    不同的开发小组组成了一个大的开发团队

- 按照业务划分服务，而不是技术
    每个开发小组肯能都会用到多种相同的技术

- 自动化运维
    小组开发完毕后的交付产品会自动对接到整个系统当中

- 容错
    小组中如果出现了一些争论点，或者异常情况，不会影响到其他小组。“错误” 被物理隔离了

- 弹性优化
    如果某一小组所开发的服务需要进行扩展，或者增加功能，那只需要添加这一小组的成员就可以了，甚至可以从其他地方抽调人员

组织结构，与所开发的系统的架构是互相映射的。


## 参考
- https://yq.aliyun.com/articles/8611
- http://blog.itpub.net/31556438/viewspace-2564683/
- https://www.jianshu.com/p/5fef1ed84f7d