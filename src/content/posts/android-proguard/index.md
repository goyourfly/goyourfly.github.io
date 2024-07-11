---
title: "混淆的问题"
published: 2021-07-30
tags: [Android]
category: 技术
---

- 经过 R8 编译混淆过后的代码，为了压缩体积行号表丢掉了，但是这导致根据报错的堆栈信息无法定位到错误的具体行号，提升了修 bug 的难度
- ![img.png](img.png)
- 可以在 proguard 文件指定让它保留 LineNumberTable
- ![img_1.png](img_1.png)
- 保留之后 retrace 堆栈信息就能定位到具体的行号，方便准确的定位报错位置
- ![img_3.png](img_3.png)
- 如果不保留 retrace 就是这样的
- ![img_2.png](img_2.png)
- 但这样打出的堆栈信息里会有抛异常的文件名，暴露代码所在文件
- ![img_4.png](img_4.png)
- 所以需要通过 renamesourcefileattribute SourceFile 将抛出堆栈是的原文件名字改成 SourceFile
- ![img_5.png](img_5.png)