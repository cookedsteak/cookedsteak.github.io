---
layout: post
title: 玩一玩Github的API接口
category: 技术
keywords: github,api,git,graphql
comments: false
---

## 概述
新的github接口使用了GraphQL（以下简称g），之前没有学习过，正好遇见了一个需求：
> 获取一段时间内的项目commit数量

所以我们今按照官方的文档来看看如何用新的GraphQL解决实际问题吧。

## GraphQL
首先肯定要知道 [GraphQL](http://graphql.org/) 这是个什么玩意吧。没错他就是一个数据传输结构约定。
Github提供的g查询接口有一个同意的入口：
> https://api.github.com/graphql

## 身份验证
利用g来做身份认证我们首先需要一个github的accesstoken。
我们可以在[这里](https://help.github.com/articles/creating-an-access-token-for-command-line-use/)设置。
这个token会有不同的权限，选择我们需要的权限。

下面是通过token去进行login的操作。
```
curl -H "Authorization: bearer [accessToken]" -X POST -d " \
 { \
   \"query\": \"query { viewer { login }}\" \
 } \
" https://api.github.com/graphql
```
结果返回的应该是你的用户名。
我一般就直接用POSTMAN进行调试操作，只需要在Authorization中选择Bearer验证就可以了。
然后在选择POST, BODY使用raw json。

![1](/assets/img/github-graphql/1.png)

## 示例讲解
我们来看一下一段示例代码
```
query {
  repository(owner:"octocat", name:"Hello-World") {
    issues(last:20, states:CLOSED) {
      edges {
        node {
          title
          url
          labels(first:5) {
            edges {
              node {
                name
              }
            }
          }
        }
      }
    }
  }
}
```
[Run in Explorer](https://developer.github.com/v4/explorer/?variables=%7B%7D&query=query%20%7B%0A%20%20repository%28owner%3A%22octocat%22%2C%20name%3A%22Hello-World%22%29%20%7B%0A%20%20%20%20issues%28last%3A20%2C%20states%3ACLOSED%29%20%7B%0A%20%20%20%20%20%20edges%20%7B%0A%20%20%20%20%20%20%20%20node%20%7B%0A%20%20%20%20%20%20%20%20%20%20title%0A%20%20%20%20%20%20%20%20%20%20url%0A%20%20%20%20%20%20%20%20%20%20labels%28first%3A5%29%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20edges%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20node%20%7B%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20name%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%20%20%7D%0A%7D)

