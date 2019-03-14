---
layout: post
title: 一个简单的以太坊代理交易模式合约
category: 技术
keywords: 区块链,以太坊,dapp,智能合约,eth,ethereum
comments: true
---

## 代理模式
（到了现在，别再说什么没有落地的区块链项目了，好的项目国家都在做，只是普通人不知道罢了。）

阻碍以太坊为大众所用的不是缺少出众的idea，而是高昂的使用成本，除了学习一些区块链的基础知识外，每笔发生在以太坊上的交易都要gas费用，除非用户能够看见大于gas费用的价值输出，否则很难真正将 eth based dapp 纳入日常使用的范围。

是否有一种模式，能够让用户脱离高昂的使用学习成本，又能可信地加入到区块链的世界中来呢？正是这样的需求，代理模式就应运而生了。

让开发者帮你支付gas费用。要知道，gas支付最大的一个问题是无法进行代理支付，以及使用其他erc20支付。
比如：我要用你的app，但是我只充值了你app的erc20代币，那我还要去充值eth。这就很麻烦了，不友好。所以我需要可以直接扣我的erc20代币就完成交易的模式。

- [ERC865](https://github.com/ethereum/EIPs/issues/865)
这个issue下引出了很多类似的解决方案。
- [关于gas支付方式的讨论](https://ethresear.ch/t/pos-and-economic-abstraction-stakers-would-be-able-to-accept-gas-price-in-any-erc20-token/721)

## 原理

### call
call被认为是一种不安全的底层调用方法。
如果address是一个合约的地址，那么call可以这样使用：
```
address.call.gas(100000【gas费】).value(1 ether【转移以太坊个数】)(0x72656769【方法】, 45【参数值】)
```
call是底层方法，通过bytes4后的合约方法名字去寻找函数。
然后按照abi的编码规范去拼接方法参数。

abi规则是：
[ABI](https://github.com/ethereum/wiki/wiki/Ethereum-Contract-ABI)
就是定义好数据结构，好让evm做序列化和反序列化。

还有一种调用方式，我称作“生调用”。
``` 
[solidity]
bytes data = 0x2fbebd3800000000000000000000000000000000000000000000000000000022;
address.call(data【将字节化后的方法和参数值拼起来的最后值】);
```
就是把值给捏在一块儿了，然后打包一股脑传输。
注意的是，这个值必须是32字节的倍数字节（hex换算是*2），不足的用0补足，规则就是上面ABI的规则，请自己参考。
测试的时候请不要直接将abi后的data写死在合约中，据我测试下来，合约不支持动态的字节数call，所以还是从外面传入数据比较靠谱。

### 验证
既然使用了代理模式，那如何验证这条消息就是本人发出的呢？
第一个想到的就是签名的验证。
eth中的信息加密签名算是比较简明的（可以看我的另一篇滑溜溜的椭圆曲线了解更多），我们用web3js可以直接输出结果
```
[js]
const prik = "0xaabbbbccccsssdddd";
let dataHash = web3.utils.soliditySha3(data);
let sig = mySign(dataHash, prik);
```
dataHash就是把你需要签名的data，打包成一个摘要。
然后用私钥prik签名。
在验证的时候，我们只需要知道摘要后的hash和签名，就能验证发送的data来自某个地址。
```
[solidity]
address signer = ECRecovery.recover(keccak256(abi.encodePacked(data)), _sig);
```
ECRecovery我们用的是openzeppelin-solidity的代码。


## 模拟的场景
百闻不如一贱，我们就来实操一下，选个场景
用户想通过一个中间合约来买weed，硬不硬核？

我们有个中间合约
Proxy代理，也就是我们代理执行的合约
这个合约需要有一个delegateBuy的方法
还要有一个verifyData的方法

我们还有一个商店的合约
Shop，这个合约中有记录每个地址购买的weed数量。

```

```

