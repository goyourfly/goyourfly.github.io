---
title: Long 和 Float 谁更大？
published: 2019-04-08
tags: [Android,Kotlin]
category: '技术'
draft: false 
---

- 64位整形的最大值（Long.MAX_VALUE） 和 32位浮点型的最大值（Float.MAX_VALUE）谁更大？
- ````
  System.out.println("Max long < Max float ? " + (Long.MAX_VALUE < Float.MAX_VALUE));
- 要回答这个问题先得看看他们在计算机中的存储方式（一般情况）：
- 整形在计算机中存储方式相对简单，以 64 位整形（long）为例：
- `| 1位符号位 | 63位数值 |`
- 而浮点型稍微复杂点，它的存储方式谁这样的，32 位为例
- `| 1位符号位 | 8位指数（包含正负） | 23位尾数 |`
- 比如 100.01 = 1 * 1.0001 * 2^2
- 从上面可以看出，long 的范围：`-2^63 ~ 2^63`
- 而 float 的范围：`-2^(2^7) ~ 2^(2^7) = -2^127 ~ 2^128`，其中有几个保留数字
- 虽然 64 位整数没有 32 位浮点数的取值范围大，但是它的精度更高。