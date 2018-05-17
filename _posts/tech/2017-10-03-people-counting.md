---
layout: post
title: 实时人流统计
category: 技术
keywords: opencv,python,人流统计,机器视觉
comments: false
---

### 突如其来的需求
A先生：“牛排，能写一个放在店门口的人流统计系统吗？”
我：“不如买现成的，但是我觉得可以试一试，听起来蛮有趣。”
所以....emmmm
[源码位置](https://github.com/cookedsteak/people-counting)

### 准备工作

1. python3.6+ 环境
2. opencv3库
3. 摄像头（笔记本自带也可以）

### 安装python3与opencv3

这个安装过程，有个洋吴克已经做得很好了，我建议参照他的方式去安装
[点击这里](https://www.pyimagesearch.com/opencv-tutorials-resources-guides/)

### 源码分析


### 示例效果
我们可以对摄像头的区域进行划分, 蓝色和红色之间的才是识别区域。
我们将红色的两个罐子比作人头。
![example](/assets/img/people-counting.gif)

一般这种摄像头的位置都会安装在由上往下的视角，比如：
![camera](/assets/img/people-counting-camera.png)



