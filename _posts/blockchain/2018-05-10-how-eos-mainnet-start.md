---
layout: post
title: EOS主网上线流程
category: 区块链
keywords: 区块链,技术,eos,EOS
comments: false
---

### 等待EOS主网的上线
最近在做一个基于eos的DAPP社区项目，eos的更新非常勤快，转眼已经DAWN-4.0.0了。
苦于不太熟悉c++，没法细致阅读eos的源码，也就没法深入了解eos的机制。
但是好在文档还是比较详细的，让开发并没有太多大坑。
为了更好理解使用eos，我们就说说eos主网上线的流程是怎么样的。

### 先朋友，后对手
在主网发布之前，大家都希望eos能够顺利启动。但是启动之后，由于节点间的互相竞争，大家又变成了对手。
我们假设有50个节点参与竞选（比如引力区啊，Oraclechain啦，EosNation啦）。我们将会有一个列表，这些在列表中的组织节点，
就是参与这场竞争的主角。那我们如何保证进展有序进行，并又让eos主网顺利上线呢？
一位智者给出了一个方案[Bios boot EOS blockchain](https://medium.com/eosio/bios-boot-eosio-blockchain-2b58b8a978a1)
我们分阶段看下。

### 阶段-0
我们称之为预热，在预热阶段，我们用可被证明的随机函数，选择一个候选节点作为boot node，又称创世节点。

---
### 参考
1. https://www.youtube.com/channel/UCoTHlEOVdeN9NOc5b5C1-vQ
2. 