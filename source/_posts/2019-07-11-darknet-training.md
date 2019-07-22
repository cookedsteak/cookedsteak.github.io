---
layout: post
title: 如何训练自己的识别模型
category: 技术
keywords: 机器学习,yolo,python,darknet,机器视觉,darknet
comments:  true
---

## 前言

最近接到一个需求，就是要通过视频分析某些特征场景，并记录下来。
我寻思着，这个应该就是一个简单的机器学习模型应用场景。

通过标注图片，放入到学习框架中，然后得出模型，调用识别视频流中的每一帧。

嗯，听上去还是挺简单的（确实是）。

所以这篇文章，我就记录一下整个需求实现过程中的要点。

*有需要详细的过程的，可以看下 `参考` 中的链接。

## 准备

### 识别算法

我们这个需求的范围应该划归到 Object Detection 物体识别的范畴。
所以我们需要选择一个合适的物体识别算法（模型），这里我选择了 [Yolo](https://pjreddie.com/darknet/yolo/)。详细介绍请看官网。


### 硬件设备

说服需求方租了一个月的云GPU实例，用的Tesla P4。

关于识别端，比较吃U，使用普通的双核实例（不依赖内存），对于一般清晰度（720p）的视频源支撑2路应该是没有问题的。

#### 驱动的安装

驱动主要体现在学习机上，对应的CUDA、CUDNN和TF框架都是有对应版本的，这个需要到TF官网，或者Nvidia官网上去找。

然后是识别机的 opencv 依赖，由于我用的是 docker，所以没有这个顾虑。

### 数据集

具有明显特征的图片，我觉得至少500张是必须的，否则训练出来的效果不好。
图片一会儿我们还需要用工具进行标注。

测试过一次：VOC2007 数据集，10000张，20类，50000次的迭代学习。大概花了48个小时。
这个时间算是偏慢的，因为在训练参数上我调小了某个阈值，为了防止显存不够而报错，算是牺牲了性能求稳定。

在500张左右的数据集，2类的条件下，我选择了6000次迭代学习。

## 训练

支持 yolo 算法的学习框架比较常用的两个。一个是 `tensorflow` 对 yolo 的支持 -- [Darkflow](https://github.com/thtrieu/darkflow#training-on-your-own-dataset)。还有就是亲生的 [darknet](https://github.com/pjreddie/darknet)。

两个我都试了下，df(darkflow) 的操作会相对简单，但是出一些莫名其妙的bug概率比 dn(darknet)高。
另外df是基于 tensorflow 的，所以要求安装的依赖比较多，同时 tf 和 cuda 的版本也需要对应起来。
dn就没有这个问题，只要能够探测到 GPU，就能够使用 GPU 训练，没有额外的依赖。
截至目前df不支持 yoloV3 的算法，这就有点小蛋疼了。

所以最终我选择了 dn。

### git clone

不用多说，克隆官方仓库。然后下载预训练的模型。

### 准备文件

#### xxx.data
要让程序知道你的文件位置，所以我们需要一个 `xxx.data` 文件。大概长这样：

```data
classes= 1 
train  = /home/guoruili/c/darknet/train/voc/train.txt 
valid  = /home/guoruili/c/darknet/train/voc/2007_test.txt
names = /home/guoruili/c/darknet/train/voc/voc.names
backup = backup
```

- classes
    你要识别的有几种物体
- train
    训练目录的位置
- valid
    测试目录的位置
- names
    识别的物体叫啥的文件
- backup
    阶段训练过程中 weights 存放位置

这里的目录位置文件train、valid，需要该文件中有训练图片的相对或者绝对路径。

#### xxx.names

存放你将要识别的物体类别的文件，一行一个类别，比如：
```
boat
bike
car
...
```

#### cfg文件

这个文件是训练的算法依据，一般是复制现有的 cfg 文件，比如 `yolov3.cfg`，`yolov3-tiny.cfg`，`yolov3-voc.cfg`

复制完毕后根据自己的训练环境，和训练类别调整参数。

#### 预训练模型

通过[这里](https://pjreddie.com/media/files/darknet53.conv.74)下载干净的预训练模型。

#### 必要的 txt 文件

我们一共要几个txt？

`4 + 图片张数`

4是什么？

我们可以把 4 理解为 2 + 2，就是训练文件2个，验证文件2个。
不论是训练还是验证，都会再要求2个txt，一个是标注所有图片名称的txt。还有一个是标注所有图片绝对路径（推荐）的txt。

那图片张数又是什么呢？

darknet 在学习的时候要求有一个 labels 文件夹。里面放的是每张图片对应的标注位置，只不过我们需要把xml转换成txt。

#### backup文件夹

这个文件夹请务必提前创建好，提前创建好，提前创建好。
一般建立在你训练数据集的根目录下。

如果没有这个目录，可能你训练了几天几夜的模型因为找不到文件夹而直接退出了，没有任何备份！

仔细观察所需要的文件，你就能够发现其实是可以用自动化的方式去自动生成的，代价就是将文件名进行预先约定。所以我抽空写了一个自动生成的[【脚本】](https://github.com/cookedsteak/darktrain)，这样的话我们所需要准备的素材就只有4样了。

### 进行训练

训练就是一行命令的事情，记住各个参数格式，我们首先要cd到根目录。

```bash
./darknet detector train 【data文件路径】 【cfg文件路径】 【预训练模型路径】
```

## 使用模型

最终在我们的识别程序中（python + opencv），需要的为3个文件。
1. cfg 文件
2. 训练完毕的权重文件
3. names 文件

将这三个文件放入同一个文件夹，然后供 opencv 调用。

## 注意

执行 darknet 命令时候，注意所在的目录位置，有些默认的文件程序会自行搜索当前目录下面，如果你没有根据需要改动，那训练出来模型可能会是错的。

## 参考
---
- <https://www.pyimagesearch.com/2018/11/12/yolo-object-detection-with-opencv/>
- <https://github.com/AlexeyAB/darknet>
- <https://pjreddie.com/darknet/yolo/>