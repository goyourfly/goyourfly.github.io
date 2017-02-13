---
title: "Mat - OpenCV中的图片容器"
date: 2017-02-13
---

### 图片是如何存储的
当我们把现实世界中的图像存储到数字设备的时候，我们记录的本质其实是一些按照一定规则排列的数字，如下图:
![](http://docs.opencv.org/master/MatBasicImageForComputer.jpg)

其中车的镜子在计算机中是一个包含亮度的像素矩阵

### Mat
Mat类包含两部分：

* 矩阵头(matrix header)：包含矩阵的大小，存储的方法，存储在哪里等等；
* 一个指针，指向包含像素信息的矩阵。

矩阵头是常量，但是矩阵的大小是根据图片的大小动态变化。

OpenCV是一个图像处理库，它包含一大堆处理图像的方法，为了解决一个问题，通常需要同时调用很多个方法，这就导致一张图片可能在多个方法之间传递，别忘了我们是在做图像处理算法，这都是很重的操作，所以最后你需要做的事情是减慢你的编程速度，尽量避免不必要的图像拷贝。

为了解决这一问题，OpenCV引入了依赖计数系统，其核心思想是：每个Mat有它自己的Header，但是，矩阵可能会被两个实例共享，此外拷贝操作只拷贝头部和它指向的矩阵，而不是数据本身。

```c++
Mat A, C;                          // creates just the header parts
A = imread(argv[1], IMREAD_COLOR); // here we'll know the method used (allocate matrix)
Mat B(A);                                 // Use the copy constructor
C = A;                                    // Assignment operator
```

以上代码中，最终的点都是一个矩阵，他们的头是不同的，但是，如果改变其中一个的点，其他的也会跟着变化，有趣的是，你甚至可以只关联到所有数据的一小节，如下：

```c++
Mat D (A, Rect(10, 10, 100, 100) ); // using a rectangle
Mat E = A(Range::all(), Range(1,3)); // using row and column boundaries
```

现在你可能会问，如果一个矩阵关联很多歌Mat，那么谁对这个矩阵负责？简单的答案是，最后一个对象，有时候，你可能也想拷贝矩阵，OpenCV提高如下方法拷贝：

```c++
Mat F = A.clone();
Mat G;
A.copyTo(G);
```

现在如果你改变G是不会影响到A的。

总结：

* 图片的回收是自动的
* 在使用C++的方法是，你无须关系内存回收
* 赋值的操作只会拷贝header
* 如果想连同矩阵拷贝，需要用`cv::Mat::clone()`和`cv::Mat::copyTo`

### 存储方式(Storing methods)