---
layout: post
title: 实时人流统计
category: 技术
keywords: opencv,python,人流统计,机器视觉
comments: false
---

## 突如其来的需求
A先生：“牛排，能写一个放在店门口的人流统计系统吗？”
我：“不如买现成的，但是我觉得可以试一试，听起来蛮有趣。”
所以....emmmm
[源码位置](https://github.com/cookedsteak/people-counting)

## 准备工作

1. python3.6+ 环境
2. opencv3库
3. 摄像头（笔记本自带也可以）

## 安装python3与opencv3

这个安装过程，有个洋吴克已经做得很好了，我建议参照他的方式去安装
[点击这里](https://www.pyimagesearch.com/opencv-tutorials-resources-guides/)

## 基本的实现思路
- 识别动的东西
- 运动方向

## 源码分析

### 引用库
```python
from collections import deque
from imutils.video import WebcamVideoStream
import imutils
import cv2
import time
import math
```
上面是我们将会用到的包，除了cv2, time, math这些图像计算要用的包外，我们还找到一个封装好的视频处理+机器视觉包--[imutils](https://github.com/jrosebr1/imutils)，这个包作者就是我们说的洋吴克，他帮我们造了很多现成的轮子，比如视频流的异步处理（主要就是这个），图片截取优化等。

### 轨迹
既然是物体的进或出，那必定离不开“如何判断运动轨迹”。
所以我们需要一个数组，来存放物体移动的每个时刻的轨迹坐标。
```python
traceList = []
```


## 示例效果
我们可以对摄像头的区域进行划分, 蓝色和红色之间的才是识别区域。
我们将红色的两个罐子比作人头。
![example](/assets/img/people-counting.gif)
同时有两个物体时候会自动区分，运动方向与进出方向也会区分。

一般这种摄像头的位置都会安装在由上往下的视角，比如：
![camera](/assets/img/people-counting-camera.png)

所以其实人流的计数，完全可以看做“数人头”。
我们这里给人头一个定义：
> 俯视角度的椭圆状的移动块

那这里我们就可以有了基本的画面算法思路：
- 计算第一帧与第二帧的异或，二值化
- 根据一定速度下的运动轨迹判断同一物体

