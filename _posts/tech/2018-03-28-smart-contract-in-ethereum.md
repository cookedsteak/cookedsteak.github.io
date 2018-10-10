---
layout: post
title: 以太坊智能合约入门
category: 技术
keywords: 以太坊,智能合约,技术,区块链
comments: false
---

## 一、开头
啊，实在抱歉，我再定义一下区块链：

参照了这篇
[How does ethereum work anyway](https://medium.com/@preethikasireddy/how-does-ethereum-work-anyway-22d1df506369\)
区块链网络是--一个由具有共享状态（shared-state）的密码性安全交易的单机构成的网络。

**什么是共享状态？**
就是大家永远保持更新的数据库，没有单一的库会有数据脱节。这里的状态可以和最新公认数据划上约等号。

**什么是密码性安全交易？**
我的理解是不论是交易的发起，还是状态的记录，都是建立在密码学的基础上保证安全性，保证共享状态的数据库没有这么容易被攻破或篡改。

**单机？网络？**
想象一下：阿猫阿狗阿狸，每个人手上都有一台安卓机，每个人的安卓机上都装着一个APP，APP本地存着共享状态。这样的安卓机就是单机（区块链节点），单机连接在一起构成了区块链网络。
以太坊是区块链的一个实现，他扮演着安卓机操作系统的角色。

## 二、智能合约不是直肠子
我将“转账金额”和“转账对象”打包成一个交易请求扔给以太坊网路去处理，处理结果也就是“转账对象”收到了我要求的“转账金额”。

我称这个过程为等价输入输出，吃啥拉啥的直肠子。但其实这种处理方式并不能完全满足现实生活中复杂的交易场景，有时候你需要在转账的同时做一些状态的记录，或者说将转账的金额分发给第三方等等。

这时候智能合约也就出现了，它允许你按照自己的逻辑去制定对输入的处理。吃的东西可以被充分吸收转化一番，然后再出来。

你可以将智能合约看做一套业务逻辑代码，只不过运行的平台从相对连接的单机平台变成了需要绝对连接的网络平台。智能合约被存储在链上（以太坊虚拟机），如果你是个全节点，你就会有所有的智能合约可执行代码。

## 三、智能合约也是个人（大概）
先让大家对交易有个大致概念。
看下灵魂画师之作：
![什么是交易](https://diycode.b0.upaiyun.com/photo/2018/421d4ca1bc53b45358fd990da989d802.png)
交易，是连接以太坊网络与外部世界的一个桥梁，只要是来自外部的、想要改变以太坊内部状态的请求，我们都看做是一笔交易，或者说会**导致以太坊网络内部状态发生变化的行为**都是交易。

继续回到智能合约，
以太坊做了一个非常有趣的抽象，他把智能合约也抽象成了一个账户。就像是一个人工智能的账户（这也解释了为什么叫智能合约）。试想一下，如果你想用一个合约完成分账，是不是类似于一个人在手工帮你计算金额并分别打给不同的人？

于是，以太坊网络中只存在了两种账户：
**外部账户（Externally owned account）**
外部账户他......就是个账户，除了用私钥控制账户的交易之外没有别的特殊功能。
**合约账户（Contract account）**
合约账户只能被合约代码控制，触发合约其实也就是向合约账户发送一笔交易，使合约账户执行合约代码。
![两种账户](https://diycode.b0.upaiyun.com/photo/2018/b8c43a74b5d411e30385260d4023dd1a.png)
我们一般常说的部署一个智能合约上链，就是在创建一个智能合约账户。而我们需要调用智能合约，就是在和合约账户做交易。

一个合约账户与外界互动的方式只有这两个，**被部署**、**被调用**。

所以理论上，你可以用外部账户创建合约（合约账户），合约里又可以执行代码创建合约，合约造合约，无穷无尽....

## 四、四大金刚
刚刚说了人工智能的账户，如果从这个角度出发，就不难理解接下去要说的，账户中会塞些什么东西了。

账户里要塞的东西，或者说与账户关联的数据，由四个状态（内容）组成：
#### 1. nonce
一个账户的所有交易计数，这个计数是个自增的，只和账户相关的值。
#### 2. balance
一个账户拥有的以太坊数量，单位为wei。1 Ether = 10^18 Wei
#### 3. storageRoot
这个账户所存储的内容的hash根值。这个状态的存储结构是Merkle树的结构。
#### 4. codeHash 
这个状态是只有合约账户才会有值。因为合约账户有代码，是把代码（code,code,code）hash后作为codeHash保存。
![账户里的四大金刚](https://diycode.b0.upaiyun.com/photo/2018/99b8a5e448501d737a7ebf4d98b7af3f.png)

说说storageRoot：

Hash（KECCAK-256）函数，把一长串数据用密码散列函数计算出一个唯一的定长（256位）摘要，就像你的指纹就是你整个人的摘要。

这个摘要算法有什么用呢？
简化数据认证过程：比如我下载了一个小电影，我如何知道我没有下载错呢？普通方式就是重头到尾看一遍，那太累人了。有个更快捷的方法，摘要函数，我只要计算一下小电影的hash码是不是和下载前提供的hash码一致就可以了，因为摘要是唯一的。

上述只能说明用hash摘要的必要性，但是为啥是默克尔树的结构呢？

加快数据认证速度：试想一下，如果有人在你下载的小电影里加了《澳门真人在线赌场》的广告，会不会很扫兴？我们用摘要算法完全可以检测这种状况，但是每次全部检测出后都要重新下载正确的片子，如果我们下载的是一篇长达24小时的高清巨作怎么办？吃不消吃不消。那我们把片子切割呢？切割成一小块一小块，可能其中某一个块的被篡改了，我们只要检测到一小部分的修改，并重新下载没有广告那一小部分就可以了。顺着这个思路，看下v神自己解释 [Merkling in Ethereum](https://blog.ethereum.org/2015/11/15/merkling-in-ethereum/)。以太坊中的数据结构不单单只有默克尔树一种，还有Merkel Patricia Tree。

默克尔树结构：
![默克尔树](https://diycode.b0.upaiyun.com/photo/2018/c1c8727228ad5af5bea08e4dfeb76910.png)

最底下的那一行就是我们切割后的数据，例子中是我们切割后的小电影，而在实际合约账户中，存储的则是不同的数据的状态。他们两两配对用来计算第一次hash，然后所有第一次的hash再两两配对，计算第二次的hash....直到只剩最后一个根hash（storageRoot）。

## 五、灵魂
大概知道了合约账户是怎么回事儿，那我们就要进入下一步，赋予智能合约灵魂--写代码。
#### Solidity  [səˈlɪdətɪ] 
一个高级语言，能够实现智能合约的语言之一，也是现在最为常用的智能合约编写语言。
这里有官方的[介绍和语法文档](http://solidity.readthedocs.io/)。
要写好的代码需要一个好的编辑器，现在有好多在线的合约编辑器，我推荐两个：
- [Remix](https://ethereum.github.io/browser-solidity/) 需要科学上网（推荐）
- [ethfiddle](https://ethfiddle.com/) 不需要科学上网，但是功能相对较弱


> 欢迎加入 [DAOONE](http://daoone.org) 分布式协作社区，我们这里有互相学习成长协作的区块链爱好者

