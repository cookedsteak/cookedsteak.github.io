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
是一个无序的String类型数据的集合，其中不会有重复的数据。
Set 的实现用到了 hashmap，真正的值其实是存储在 hashmap 的 key 中的。

set 类型的数据可以有 交并操作，所以很适合用来存一些标签。

## HASH
就是我们常用的 map，key 为 string类型。

## STRING
普通的key/value存储结构

## SORTSET
sortset 和SkipList有点关系，我们看下[源码](https://github.com/antirez/redis/blob/129f2d2746ca80451d8c84b223b568298020b125/src/server.h)：

```
typedef struct zskiplistNode {
    sds ele;
    double score;
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;

```
曾经我拿来用作排行榜的数据，感觉挺好用。