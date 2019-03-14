---
layout: post
title: Hash Table
category: 技术
keywords: 后端,计算机原理,数据结构,golang,基础
comments: false
---

go map 的底层实现其实是基于 hash table 的。
hash table 作为一种散列式的存储结构，有着好几种算法。

### hash table
array[a,b,c,d,e]
这样一个数组，如果要找 c，一种简单的方法是 o(n) 顺序搜索。
但是如果我们知道 c 的下标（index）是不是就会很快了，比如 array(2) = c。

如果我们一一对应，万一来一个 z, 是不是中间会有很多中空的位置被浪费了。
（index 只能是依次递增的）

虽然对于我们来说，a,b,c,d,e 是顺序的，但其实他的存储方式在计算机中是散列的，松散的。而我们要尽可能让其平均分布于有限的数组空间内，所以就有了散列函数（散列算法）
我们使用一种 Mod 的计算方式，计算出他们的下标。

> address = key Mod n

其中，Mod 是取余运算，n 是数组的长度，key 就是我们要存的内容。
我们一般先把 key 转换为 ASIIC 码，在做运算。

### Collisions (碰撞)

如果遇到 key 的碰撞怎么办？
我们看下下面的算法。
collisions 有两种大解决方案：
1. Open addressing 开放定址法
2. Closed addressing 
3. Rehash
4. Chaining

#### Linear Probing 算法
不同 key mod 出相同索引，就顺推直到索引位置为空。

- Load Factor 加载因子 = 总存储元素 / array 长度

#### Quadratic Probing




但是取余运算也会有算出来不是唯一的时候。
