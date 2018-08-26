---
layout: post
title: EOS私链搭建以及多BP设置
category: 区块链
keywords: 区块链,技术,eos,EOS,dapp
comments: false
---

### 前言
一直在调试使用本地单节点的eos环境，好奇多bp的环境是如何配置的，因此这篇就简单介绍下如何搭建一个多bp的eos本地环境，基于eos release 1.1。

网上到处都有的：[EOS单节点手动安装启动教程](https://developers.eos.io/eosio-nodeos/docs/)
其实官方文档真的写得很清楚了，大家多看看。


### 启动一个创世节点
安装调试完毕一般的单节点后，我们热身一下。
```
nodeos -e -p eosio --plugin eosio::chain_api_plugin --plugin eosio::mongo_db_plugin --plugin eosio::sql_db_plugin --plugin eosio::history_plugin --plugin eosio::history_api_plugin  --plugin eosio::wallet_plugin --plugin eosio::wallet_api_plugin --wallet-dir /Users/steak/eosio-wallet/ --chain-state-db-size-mb 1024 --abi-serializer-max-time-ms=5000 --max-transaction-time=5000 --mongodb-uri=mongodb://localhost:27017 --sql_db-uri="mysql://db=EOS user=root host=127.0.0.1 password=root port=3306"
```

我们使用了两个数据库的插件，
一个是mongodb，还有一个是mysql。
这两个插件都是用来同步/异步记录块的历史日志的。
启用这两个插件可以让你更容易查询到你要找的记录，而不用设置filter-on，或者辛辛苦苦调用eos的api。

注意：
- * mongodb在build的时候自己会安装driver
- * 关于sql_db_plugin的安装，需要在安装完c++ 数据库库soci后再进行eos的build操作

### 创建关键账户，部署合约
我们试着启动了一下节点啊，这时候使用的是 -p eosio，也就是eosio作为一个producer。
这时候我们应该也在默认目录（一般在用户的local文件夹，可查询官方文档）生成了一个配置文件和block文件。
那其实真正eos按照治理规则使用起来是需要一些官方提供的账户的，有以下这些：
```
eosio.bpay
eosio.msig
eosio.names
eosio.ram
eosio.ramfee
eosio.saving
eosio.stake
eosio.token
eosio.vpay
```
创建命令：
> cleos create account eosio eosio.bpay

将eosio.token合约部署到同名账户上：
> cleos set contract eosio.token ~/path/to/eosio.token/contracts/

将eosio.msig合约部署到同名账户上：
> cleos set contract eosio.msig ~/path/to/eosio.token/contracts/

### 发币系统代币
创建代币：
> cleos push action eosio.token create '["eosio","1000000000.0000 SYS"]' -p eosio.token
自己发给自己：
> cleos push action eosio.token issue '["eosio","1000000000.0000 SYS","memo"]' -p eosio 

注意，eosio.system 合约中将SYS 设置成了系统代币，你可以更换自己的代币作为系统代币，位置在core_symbol文件中（每个版本都会变:(），更换完成后再重新编译才能生效。

### 设置eosio.system合约
这个合约可是eos治理合约中的主角
如果需要用自己的代币，请修改eosio.system合约
> cleos set contract eosio ~/Projects/c/github.com/EOSIO/eos/eos/build/contracts/eosio.system/

### 创建抵押账户
现在eosio成了大金主，我们就可以用eosio来创建可以真实使用的eos账户了。

> cleos system newaccount eosio --transfer daoone EOS6zCzMRYHyxY1viMqrBHmN1zSUZ8q4SGLFoEHs51VQntZsjZTD4 --stake-net "100000.0000 SYS" --stake-cpu "100000.0000 SYS" --buy-ram "100.0000 SYS"
这里我们创建了一个daoone的账号，这个账号将会是我们社区的主账号。