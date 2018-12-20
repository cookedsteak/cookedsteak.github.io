---
layout: post
title: xx的索引·二
category: 技术
keywords: mysql,索引,数据库
comments: true
toc: true
---

## 前言
在xx的索引·一中我们简单看到了索引的创建和作用，这篇，我想集中总结下索引的类型，基本原理。

mysql 数据库中的存储引擎分为 myISAM InnoDB。
但截至这篇文章，mysql 最新的版本8 已经逐渐弃用 myISAM，准确的说，留给 myISAM 引擎优势不多了。只剩：

- 数据大小比未压缩过的 InnoDB 小
- count(*)速度比较快
  
这两点怎么说呢...也并不算是很突出的我需要用到 myISAM 的地方吧。8中你要显式设置 engine=MYISAM。

不过作为数据结构的一种基本知识，我们还是要去了解一下，两个存储引擎的不同索引结构。

## 索引类型
- B-Tree 索引
- Hash 索引
- Fulltext 索引
- R-Tree 索引

#### MyISAM
大名鼎鼎的 b-tree 改进版本，b+tree。
MyISAM索引文件和数据文件是分离的，索引文件仅保存数据记录的地址。
![myisam](/assets/img/trees/myisam.png)


#### InnoDB
InnoDB也使用B+Tree作为索引结构，但具体实现方式却与MyISAM截然不同。
索引最终的 data 本身也是个索引
![innodb](/assets/img/trees/innodb.png)

## 参考
- https://www.percona.com/blog/2016/10/11/mysql-8-0-end-myisam/