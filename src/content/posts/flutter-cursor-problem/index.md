---
title: "Flutter 文本编辑光标位置错误"
published: 2019-05-14
tags: [Flutter]
category: 技术
---

Flutter 输入中文或者 Emoji 时光标总是对不齐，向上偏移，如下图：

![image](./img.png)

最新发布的 1.5.0 在 Android 上已经修复了，但是 iOS 上这个问题还存在，中文感觉比原来强一点了，但是 Emoji 看着还是很明显

新版本 Emoji 和 中文：
![image](./img_1.png)
![image](./img_2.png)

解决办法是自己找个中文字体替换一下，不过随便一个中文字体都是十几兆～