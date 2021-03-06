---
layout: post
title: xx的索引·二
category: 技术
keywords: mysql,索引,数据库
comments: true
toc: true
---

## 前言
在《xx的索引·一》中我们简单看到了索引的创建和作用，这篇，我想集中总结下索引的类型，基本原理。
*有时候想想，怎么能够设计一套存储系统，把乱七八糟的数据快速的搜索，修改，删除等，这样想来就觉得数据库是个了不起的东西。也就更想看看数据库背后设计逻辑到底是怎么样的。

推荐一个超棒的基础讲解视频，要翻墙：[阿三老师讲 B-Tree](https://www.youtube.com/watch?v=aZjYr87r1b8)。

我这里简单概括下为什么要用索引：
数据依旧是存储在存储介质上，但是存储的位置是不连续的，散列的。
所以我们需要一个类似目录的东西，当我们快速定位数据的位置。这时候我们有了第一层的 id。
那 b-tree、b+tree，就算是在我们第一层的 id 基础上再加一层所以，比如1对应 id 1~32，2 对应 id 33~64...
在数据量非常大的情况下，我们可以多做几层这样的索引，加速查询。这样一层一层就变成了树。

上面的行为本质上就是用空间换时间的做法。

而索引算法，不单单是单纯的叠加索引的数量，他是一种维护索引结构的机制，创建，删除，更新，我们都要保证索引的结构清晰，查询高效。

MySQL 数据库中的存储引擎其实有好多，但是最常用的是 MyISAM InnoDB。
但截至这篇文章，mysql 最新的版本8 已经逐渐弃用 myISAM，准确的说，留给 myISAM 引擎优势不多了。只剩：

- 数据大小比未压缩过的 InnoDB 小
- count(*)速度比较快
  
这两点怎么说呢...也并不算是很突出的我需要用到 myISAM 的地方吧。8中你要显式设置 engine=MYISAM。

## m-way search tree
正常的节点是一个元素一节点，但是这样放有点浪费，所以我们可以在一个节点存放多个指向内容的 key。同时，多个元素中间都会有一个空档指向“大于上一个，小于下一个”元素的子index。
有点像夹心饼干。

这个就是 m-way search tree

所以一个 m-way search tree 的节点能够有 m-1 个子 index。

## b-tree
但是 mway tree 会有一个问题，就是插入的时候我们没有一个指定的规则去控制插入的算法。
如果我要插入的是一个 10个 key 的顺序排列数据。你最后会发现捣鼓出来了一个 高度为 10 线性树。
B-tree 是一个有规则的 m-way tree。

## 索引类型
- B-Tree 索引
- Hash 索引
- Fulltext 索引
- R-Tree 索引

#### MyISAM
用的大名鼎鼎的 b-tree 改进版本，b+tree。
b+tree 改进在哪里呢？

在 b tree 的基础上，每个根节点都会有一个在子节点的拷贝，同时，数据的指针全部放在叶子节点。
并且叶子节点互相连续，产生了一条链。

MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。
![myisam](/assets/img/trees/myisam.png)
我觉得MyISAM 的优势已经渐行渐远，所以不打算展开讲 MyISAM 的索引结构。
不过我们可以详细看下 InnoDB 的索引。

#### InnoDB
Innodb 的索引还是建立在 btree 上，但是在存储结构上有些不一样。
我们都知道 myisam 的 btree 是 data 和索引分离的。
但是 innodb 的 data 和索引是在一起的，innodb 的索引本身就是记录的主键。
每个 InnoDB 表都有两种索引，分别是：

- 聚簇索引（聚集索引，Clustered Index）
- 二级索引（辅助索引，Secondary Index）
  
一般来讲，聚簇索引就是表的主键。为了加速每张表的查询、插入等数据库操作，我们通常都会定义主键，这个主键就会被当做索引，就算没有主键，我们也最好定义一个自增字段。如果没有主键，innodb 会去找一个非空的 unique 字段，把它当做聚簇索引依据来使用。

如果主键和 unique都没有，那 innodb 也很无奈，只能自己创建一个隐藏索引叫做 GEN_CLUST_INDEX。

#### 聚簇索引
区区一个多出来的字段，这玩意怎么能加速的查询呢？

让我们先下沉一下，看一下 mysql 存储数据的结构（不是索引结构）。
我们在官方文档找到了这个-[File Space Management 文件空间管理](https://dev.mysql.com/doc/refman/8.0/en/innodb-file-space.html)。

我们所看到的表啊，其实是一个逻辑显示，每个表空间是由·页·（pages）组成的，每个页默认16kb（这个数字是可变的，只要你在编译的时候改就行了，但是范围在8KB to 64KB之间，还能设置参数 innodb_page_size），
64页一组=》也就是1mb。
英语单位描述 extents（不知道如何翻译比较准确）。
找到一张不错的图，可以看下：
![存储结构](/assets/img/trees/data-s.jpg)

可以看到我们的数据其实是分散存储在不同的 pages 上的，
所以，如果是搜索一个数据，有索引的条件下，你会通过索引直接指向数据所在的 page，也正是因为这个原因，加速了数据查询。

![innodb](/assets/img/trees/innodb.png)
可以看到叶子节点存放了完整的数据记录，这种索引是聚簇索引。


#### 二级索引
myisam 的二级索引存储的还是指向行的指针。

nnoDB的的二级索引的叶子节点存放的是KEY字段加主键值。因此，通过二级索引查询首先查到是主键值，然后InnoDB再根据查到的主键值通过主键索引找到相应的数据块。

## 联合索引
联合索引就是几个字段加起来的索引类型。
比如一个表中有如下字段：
```
user  phone  name  birth  height
```
我们对 `userid` `phone` `name` 做联合索引：
```sql
create index union_index on tableName(user,phone,name);
```
好，那接下来我们的测试就来了，我们想要看看哪些查询对于联合索引是有效的。
使用 explain 查询。

#### 普通查询
- 根据 userid 查询，联合索引被使用
- 根据 phone 查询，联合索引无效
- 根据 name 查询，联合索引无效

- 根据 userid and phone 查询，联合索引有效
- 根据 userid or phone 查询，联合索引无效
- 根据 phone and name 查询，联合索引无效
- 根据 userid and name 查询，联合索引有效
- 根据 userid and phone and name 查询，联合索引有效

所以你会发现，真正能够使用到联合索引的符合查询，还是看联合索引的第一个字段在不在查询条件里。

## 单列索引

还是用上面的例子，只不过我们把联合索引改为了单列索引：
`userid`,`phone`,`name`

- 根据 userid and name and phone 查询，只用到了第一个 userid
- 根据 name and phone 查询，只用到了那么
- 根据 userid or name 查询，发现两个都用到了

所以我们可以发现，单列索引只会使用查询条件里的第一个查询字段的索引。

## 参考
- https://www.percona.com/blog/2016/10/11/mysql-8-0-end-myisam/
- https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html