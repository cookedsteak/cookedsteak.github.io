---
layout: post
title: xx的索引
category: 技术
keywords: mysql,索引,数据库
comments: true
toc: true
---

## 一个头疼的问题

我也不知道为什么，面试官就特别喜欢靠索引的东西，
常年站在巨人的肩膀上，让我已经忘了索引的种种性质。
当我用一系列工具乐此不疲写着orm结构操作的时候，数据库优化的点点滴滴，也渐渐变得陌生。
这种状况并不好，所以今天就要扭转这种状况，来谈谈索引。

## 以一个例子开始

```

CREATE TABLE `names` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `first_name` varchar(50) CHARACTER SET latin1 DEFAULT NULL,
  `last_name` varchar(50) CHARACTER SET latin1 DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

```

我们往表里添加一些内容
![Name Table](/assets/img/table_data.png)

通常建立完表，主键就是第一个索引

我们搜一个特别的名字，然后用 EXPLAIN 语句看下执行过程：

```
MariaDB [fun]> EXPLAIN SELECT * FROM users WHERE last_name = 'Harahap';
+------+-------------+-------+------+---------------+------+---------+------+------+-------------+
| id   | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+------+-------------+-------+------+---------------+------+---------+------+------+-------------+
|    1 | SIMPLE      | users | ALL  | NULL          | NULL | NULL    | NULL |   10 | Using where |
+------+-------------+-------+------+---------------+------+---------+------+------+-------------+
1 row in set (0.00 sec)
```

你会看到，这次搜索一共检索了10条记录（就是全表了），也没有用到索引（废话）。
那我们现在给`last_name`增加一个索引优化下。

```
MariaDB [fun]> CREATE INDEX last_names ON users (last_name);
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

在用 EXPLAIN 跑一次：

```
MariaDB [fun]> EXPLAIN SELECT * FROM users WHERE last_name = 'Harahap';
+------+-------------+-------+------+---------------+------------+---------+-------+------+-----------------------+
| id   | select_type | table | type | possible_keys | key        | key_len | ref   | rows | Extra                 |
+------+-------------+-------+------+---------------+------------+---------+-------+------+-----------------------+
|    1 | SIMPLE      | users | ref  | last_names    | last_names | 53      | const |    2 | Using index condition |
+------+-------------+-------+------+---------------+------------+---------+-------+------+-----------------------+
```

发觉好像有变化了，我们发现搜索范围从全表缩小到了两行，emm...这似乎说明了我们刚刚设置的索引有了效果。
为毛呢，索引是什么原理呢？可以看下[该死的索引·二](/)。

接下去的问题：我们如何才能知道我们的索引对于查询是有效果的呢？首先我们需要明白Mysql中的一个概念：

- Cardinality值

我简称为c值，
> 他是一个数据库自行预估的衡量字段选择性的指标。

什么是选择性？就是这个字段可能出现的数据值的个数、种类。比如性别字段，可能出现的值就三种“男，女，双。这种就是低选择性（Low-cardinality）。相对的，一个uuid字段可能出现的就千千万了，这称为（High-cardinality）。

如何使用查看这个值呢？
> SHOW INDEX FROM [table-name]

可以看到Cardinality字段。
回到刚才的例子：
由于我们这个表是innodb的，innodb对于cardinality的计算规则近似 SELECT COUNT(DISTINCT)... 所以
```
MariaDB [fun]> SELECT COUNT(DISTINCT last_name) AS cardinality FROM users;
+-------------+
| cardinality |
+-------------+
|           9 |
+-------------+
```
我们看到了结果是9，我们和数据总数比一比。
9/10 = 90% 接近1哦，所以说数据重复率很低，适合建立一个索引。

但是需要注意，cardinality终究只是一个预估值，本身就具有不确定性。同时，对于cardinality的维护消耗也是一个问题，不同的存储引擎有自己的维护规则。[参考这里](https://www.percona.com/blog/2008/09/03/analyze-myisam-vs-innodb/)
和[这里](http://www.ywnds.com/?p=8378#comments)


-----
参考链接：
- https://stackoverflow.com/questions/2566211/what-is-cardinality-in-mysql
- https://medium.com/prismapp/indexing-mysql-with-examples-f426332ea3ec
- http://www.ywnds.com/?p=8378#comments
