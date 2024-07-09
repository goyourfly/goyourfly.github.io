---
title: VSync简介
published: 2017-02-09
tags: [Android]
category: 技术
draft: false 
---

## VSync

VSync是个什么东西呢？
V是Vertical的意思，Sync是同步的意思，所以VSync就是垂直同步的意思，大部分游戏的画面设置里都有这一个开关，开启以后可以避免画面撕裂感。

那么在Android中，VSync是什么呢？

VSync是个CPU终端，每隔16毫秒发送一次，CPU接到这个中断以后，暂停手头其他活，专门负责处理UI，处理完成后再干其他事情，这就保证了UI的流畅度。