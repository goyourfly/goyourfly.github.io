---
title: OpenCV - 简介
published: 2017-02-13
tags: [OpenCV]
category: 技术
draft: false
---

### 简介
OpenCV是一个开源的，基于`BSD-licensed`集成百上千个视觉算法的库。

OpenCV是模块化设计的，所以有几个共享或者静态库：

* Core functionality - 基础库
* Image processing - 线性和非线性图片滤镜，几何转换，色彩域转换，直方图
* video - 动作预测，物体捕捉算法
* calib3d - 多视图几何算法，单个或者多个拍照校准，物体姿势估计等等
* features2d - 突出特征检测，特征匹配
* objdetect - 预设物体检测
* highgui - 简单的UI能力
* Video I/O - 简单的视频编解码
* gpu - 不同模块的GPU加速算法
* ...

### CV命名空间
所有的OpenCV类和方法都放在cv命名空间之下，所以使用如下：

```c++
cv::Mat m = ...;
//或者如下
using namespace cv;
Mat m = ...;
```

### 自动垃圾回收
类似于std::vector,Mat和一些其他的OpenCV用到的基础数据结构都支持自动垃圾回收的，如果不是基础数据结构，而是用户自定义的，比如`T* t = new T(...);`，我们可以通过下面方法使其支持自动回收：`Ptr<T> ptr(new T(...));`或者`Ptr<T> ptr = makePtr<T>(...);`

### 自动回收Output Data
...

### 饱和算法（Saturation Arithmetics）
作为一个视觉库，OpenCV需要处理大量的图像像素，他们通常会以8-16位的紧凑方式编入每个通道，形成具有最大值和最小值的区间范围，刺猬还有一些比如色彩域转换，亮度，对比度调整，锐化，复杂的插值等，这将可能导致超出有效值的数值，为了避免这种情况，饱和算法被用与此，它始终保证一个8位的数值在0..255的范围内的最靠近的数值：

`I(x,y) = min(max(round(r),0),255)`

在OpenCV中的用法如下:

`I.at<uchar>(y,x) = saturate_cast<uchar>(r);`

其中uchar是一个8位的无符号整形
> 饱和算法不适用于32位整形

### 固定的像素类型，限制使用模板
OpenCV不允许使用模板
OpenCV支持的元组数据类型有：

* 8-bit unsigned integer(uchar)
* 8-bit signed integer(schar)
* 16-bit unsigned integer(ushort)
* 16-bit signed integer(short)
* 32-bit signed integer(int)
* 32-bit floating-point number(float)
* 64-bit floating-point number(double)
* 一个元组中的多个元素里的所有的元素具有相同的类型，一个队列中的所有元素可以是元组，相对单通道而言被称作多通道队列，他们的元素是数量值。

### InputArray 和 OutputArray
OpenCV的方法处理二维数组或者多维数据通常会以cppMat为参数，但是一些情况下使用std::vector或者Matx<>更方便，为了避免Api的重复，“代理”类被引用，InputArray用于在方法输入上传只读数组，OutputArray继承自InputArray，被用于指定一个输出队列。

### 异常处理
OpenCV使用Exception提示严重错误，一般会返回错误码的方式通知。

Exception通常以CV_Error(errcode,description)的方式抛出。

CV_Assert(condition)用于检测特定情况，如果不满足抛出异常

CV_DbgAssert(condition)只有在Debug模式时起作用

### 多线程重入
OpenCV完全支持多线程重入，类似cv::Mat可以使用在不同线程中，因为引用技术使用的是原子操作。
