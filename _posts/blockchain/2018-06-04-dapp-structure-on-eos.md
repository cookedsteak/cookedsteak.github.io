---
layout: post
title: 基于EOS的DAPP乱想
category: 区块链
keywords: 区块链,技术,eos,EOS,dapp
comments: false
---

### EOS的交易发送

EOS的交易通讯步骤可以分为以下7步：

1. 准备交易Actions的内容
2. 获取最新的不可逆块的Id，有效时间（get_info）
3. 获取所有可用的公钥（wallet_plugin:get_public_keys）
4. 带着所有公钥和交易信息去获取交易需要的公钥（chain_api_plugin:get_required_keys）
5. 拿着返回的公钥将交易发送给钱包寻求签名（wallet_plugin:sign_transaction）
6. 拿回签名好的交易，直接推送给链（chain:push_transaction）
7. 完成


### 作为DAPP的两种模式
核心的步骤就是wallet的签名，基于上面一个EOS交易的发送流程，可以有如下两种模式对交易签名：

1. 离线模式
所有的交易签名都在用户客户端完成。钱包客户端在用户本机上，所有需要签名的交易都由本机的钱包完成。
一般使用eosjs，或者eos的sdk完成。
1. 托管模式
交易密钥托管在信任的服务器上，只能由用户的安全口令才能申请钱包签名，并返回给用户，然后发送交易上链。
这种方式用户不需要本地钱包客户端，只需要连接运行keosd的服务端就可以了。

### 同步节点数据
在eos release/1.1后默认开启了 mongodb plugin，想必也是为了方便读取交易记录，而不用直接请求eos接口，同时也节省内存的使用空间。
所以一般都会在新版本的nodeos中开启mongodb的数据同步。

### 节点安全架构设计
不论使用哪种模式去实现DAPP，拥有一个自己的同步节点（群）看起来十分有必要，一来是可以加速与链的交易速度，二来能够定制化得查询链上的数据。
当然也有一些团队会在节点与交易发送端之间增加自己的监听服务。
那如何布置或者设计自己的节点（群）呢？
我们可以参照慢雾科技在github上的[分享](https://github.com/slowmist/eos-bp-nodes-security-checklist)
这个架构建议虽然是给BP看的，但是依旧能从中吸取不少。

