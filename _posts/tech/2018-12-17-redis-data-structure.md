---
layout: post
title: 回到Redis的数据结构
category: 技术
keywords: redis,数据结构,经验,数据库,后端
comments: true
---

## 前言
Redis 绝对不陌生，用得多了，难免好奇他的一些原理，好奇害死猫，这次我就要做一下这只死猫。
原理的话，如果光是分析源码那我估计可以写一本书了，当然我能力有限，写不出这种大作，所以只能先从奇妙的数据结构开始。
那接下来我们分析一个是一个（狗头）。
Redis 的数据类型常用5种：
- String
- List
- Hash
- Set
- Sorted Set

## LIST
一个 String 类型的双向链表
源码位置：[/src/adlist.h](https://github.com/antirez/redis/blob/fc0c9c8097a5b2bc8728bec9cfee26817a702f09/src/adlist.h)
```
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```
我一般用 list 作为消息队列使用，这边 RPUSH，那边 LPOP，很是欢乐。
当然，list 的用处不止消息队列

## SET

