---
layout: post
title: 那该死的索引
category: 技术
keywords: mysql,索引,数据库
comments: false
---

### 一个头疼的问题

我也不知道为什么，面试官就特别喜欢靠索引的东西，
常年站在巨人的肩膀上，让我已经忘了索引的种种性质。
当我用一系列工具乐此不疲写着orm结构操作的时候，数据库优化的点点滴滴，也渐渐变得陌生。
这种状况并不好，所以今天就要扭转这种状况，来谈谈索引。

### 以一个例子开始

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

发觉好像有变化了，