---
layout: post
title: 微信小程序的种种
category: 技术
keywords: 微信,小程序,经验,js,javascript,前端
comments: true
---

## 授权登录
切勿图方便而不使用微信推荐的登录授权方式。

## 原生挺好
这里我指的原生是指小程序的原始写法，现在有些集成小程序的前端框架，比如最近用的 mpvue，可能对于用过 vue 的同学来说会减少学习小程序的成本（其实并没有），同时还能生成 h5版本的页面。
但其实在我看来是得不偿失的。而且框架下很多方法和问题并没有妥善的解决方案，偶尔会有冲突，并且小程序的接口也是在不断更新的，框架是否也跟得上这样的填坑速度呢？

## 样式插件
这个可以有，有很多封装好的视图模板，组件，能够加快开发速度和程序的美观程度。

## 坑
#### 原生组件
原生组件层级最上！这个有点坑哦，意味着很多第三方的 UI 组件遇见原生组件就是“西哈一则”了。
具体的情况可以点击[【这里】](https://developers.weixin.qq.com/miniprogram/dev/component/native-component.html)