---
title: Surface - Draw基本形状
published: 2016-03-16
tags: [Android,Surface,Canvas]
category: 技术
draft: false
---


### Drawing

Canvas有很多的Drawing方法，每个怎么使用呢，我们先看一个简单的    
    
    canvas.drawRect(100, 100, 200, 200, paint);

其参数分别是：
- 左边位置
- 上边位置
- 右边位置
- 下边位置
- Paint
那根据我们的推断，最后的效果应该是绘制一个100 * 100的矩形，其左上角的位置是(100,100)；


![drawRect效](./image.png)

通过真机测试效果确实如设想的那样。

那么，怎么绘制一个空心的矩形呢？Canvas的所有方法里没有找到可以绘制空心矩形的方法，怎么办呢？答案就在drawRect的最后一个参数Paint中。
Paint有一个方法叫setStyle(Paint.Style)，传入的类型有3种
- FILL：填充
- FILL_AND_STROKE：即填充又空心
- STROKE：空心


    paint.setStyle(Paint.Style.STROKE);

其中FILL_AND_STROKE在绘制矩形是和FILL效果是一样的，它和FILL的差异后面在讲。

那么，如果我想要自定义线条的宽度怎么设置呢？

    paint.setStrokeWidth(10);

如果想将线条改为虚线怎么设置呢？

    paint.setPathEffect()
传入的参数有：
- CornerPathEffect 圆角
- DashPathEffect 虚线
- DiscretePathEffect 虚线（随机）
- PathDashPathEffect 定义一个新的形状(路径)并将其用作原始路径的轮廓标记。
- SumPathEffect 顺序地在一条路径中添加两种效果，这样每一种效果都可以应用到原始路径中，而且两种结果可以结合起来。
- ComposePathEffect 将两种效果组合起来应用，先使用第一种效果，然后在这种效果的基础上应用第二种效果。

其中DashPathEffect就可以我们想要的虚线效果

![drawRect虚线](./image%20copy.png)

如果需要使用到虚线的圆角，则需要使用ComposePathEffect将DashPathEffect和CornerPathEffect组合使用

    paint.setPathEffect(new ComposePathEffect(new DashPathEffect(new float[]{8, 8}, 0), new CornerPathEffect(50)));

![drawRect圆角](./image%20copy%202.png)

> 真正需要用到使用圆角矩形，可以直接使用drawRoundRect

然后我们再看一个基本几何形状，Circle

    canvas.drawCircle(100,100,50,paint);

执行上一段代码，显示效果是这样的：

![Circle基本形状](./image%20copy%203.png)

如果想要空心，同样也是设置Paint的setStyle()方法，用法和Rectangle一样。