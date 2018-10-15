---
layout: post
title: React备忘
category: 技术
keywords: react,前端,nodejs,javascript
comments: true
---

## 概述
让前端写起来飘飘然，我想这应该就是react开发时候的感觉了吧。
[React](https://reactjs.org)是个啥？可以去官网看看，说得很清楚。
我选择React进行学习的一个原因是预备知识相对少，可以不用熟悉过多的dom属性（类vue中的），只要知道js，html，css的知识就够了。

一个纯粹的react-app是没有后端的任何服务、接口或者逻辑的。
官方会推荐给你一些工具链，来解决你实际开发中遇到的问题，比如：
- 文件个组件越来越多
- npm的第三方包
- 一些代码错误的预处理
- 开发环境中实时编译修改后的文件（js/css）
- 优化生产环境下的代码输出

上面这些问题还是挺常见的，特别是我很久很久以前徒手管理js，css...
所以呢，会有一些推荐：
- 如果你是学习react-app，或者是做单页应用。可以直接用[Create React App](https://reactjs.org/docs/create-a-new-react-app.html#create-react-app)
- 如果你要开发一个带有后端服务的app，建议[Next.js](https://reactjs.org/docs/create-a-new-react-app.html#nextjs)
- 如果你是开发一个静态的内容导向的app，使用[Gatsby](https://reactjs.org/docs/create-a-new-react-app.html#gatsby)

我们就先从最简单的 create-react-app开始。

## 入口
直接npm安装 create-react-app。
一个biu准的react-app，会出现以下的文件结构
```
node_modules
public
 |- index.html
src
 |- index.css
 |- index.js
package.json
yarn.lock
README.md
```

入口文件为 index.html--引用-->index.js
需要在js文件中引用两个基本的模块：React, ReactDOM
启动单页应用只需要 npm start

## 组件与函数，足矣
React是将dom元素填充显示成最终的html页面的。
例如：
```
render() {
        return (
            <div>
                <h1>Hello, world!</h1>
                <h2>现在是 {this.state.date.toLocaleTimeString()}.</h2>
            </div>
        );
    }
```
实现render方法，就会将return的内容作为dom填充。

将一个页面的元素，分割成各个组件，这是对功能的抽象。
而对于React而言，有了组件，函数，就足以构成一个页面。

举个例子，我们要一个hello world的页面。
我们先用函数实现下：
```
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```
好，我们换成类来实现：
```
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```
上面两种实现方法某方面来说是等价的。
那我们怎么才能用他们呢？那就需要ReactDOM出场了。

填充页面的总方法：
```
ReactDOM.render(
    [元素、方法], document.getElementById('需要填充到的标签')
);
```

所以总的一个页面就很简单的三大块：
```
// 引用头
import React from "react";
import ReactDOM from "react-dom";

// 组件主体
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}

// 本页填充
ReactDOM.render(
  <Welcome name="Goo" />, // 注意这里的组件使用语法（JSX）
  document.getElementById('root')
);
```

> <组件、函数名称 />
就是这样规定的写法，这样能够被react解析。
render方法中的第一个参数可以是上面形式的标签，也可以是函数，
但是一般采用标签，因为能够控制标签的输入属性。