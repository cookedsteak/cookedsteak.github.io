---
layout: post
title: BIP和HD钱包
category: 技术
keywords: 区块链,BIP,钱包,加密
comments: false
---

## 什么是BIP
Bitcoin improvement proposals
比特币改进协议，是比特币社区提供的一种规范。
之后的EIP也一样，都是为了让整个生态更加可持续发展。
当然，BIP的各种提案都会有编号，具体可以看这里 [BIP](https://github.com/bitcoin/bips)。

BIP茫茫多，我们今天就要挑选两个和钱包有关的，BIP-32/BIP-44。
这两个标准是和HD钱包相关的标准，他可以生成一个更加简单的钱包。

## 那HD钱包呢
Hierarchical Deterministic Wallet
分层确定性钱包，比较通俗的理解就是，HD钱包根据“种子”（种子密钥）按顺序导出未来的地址，每个新地址相当于“种子+计数器”。通过这种方式，你只用一个种子就可以恢复所有的私钥和地址。


## 神奇的钱包种子
依托于大佬的智慧，我们可以将一长串（多个）的密钥，按照一定的规则简化为十几个【助记词】，只需要记住助记词，以后如果导入到使用同样标准的钱包dapp中，就能直接恢复你的所有账户和资产。