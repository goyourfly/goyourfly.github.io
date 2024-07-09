---
title: "尾递归优化"
published: 2019-04-11
tags: [Kotlin]
category: 技术
---

- Kotlin 有一个关键字叫 `tailrec` 它的作用是尾递归优化
- 比如写一个阶乘的普通递归是这样的：
- ````
    fun jieCheng(i:Int):Int{
        if(i <= 1)
            return i;
        return jieCheng(i - 1) * i
    }
- 尾递归是这样的
- ````
    tailrec fun tailJieCheng(n:Int,res:Int):Int{
        if(n <= 1) return res
        return tailJieCheng(n - 1,n * res)
    }
- 在方法前面加上 `tailrec` 关键字后，Kotlin 在编译时会自动将尾递归方法转换成一个循环
- 编译后的方法是这样的
- ````
    public final int tailJieCheng(int n, int res) {
      while(n > 1) {
         int var10000 = n - 1;
         res = n * res;
         n = var10000;
      }

      return res;
    }
- 我认为，简单点说，递归和尾递归主要的区别是看返回的是一个方法调用还是运算式

- 以下是吐槽
- Kotlin 有个很奇怪的设计如下：
- ````
    val a = 1
    val b = 2
    // 注意看下面这个
    a == b
- 此处 `a==b` 直接写居然不报错，而且它其实是个运算式，结果是 `false`，
- 有时候想你真的想把 `b` 的值赋给 `a` 的时候，不小心写成这样了，导致的结果就是你可能需要很久的时间找 BUG！
- 类似的问题还有：
- ````
    val a = 20
          + 10
- 上面这段代码其实是两个表达式，分别是 `val a = 20` 和 `+10`，很容易造成错误。
- 这种代码毫无意义，不知道 Kotlin 为什么不直接报错，难道是为了证明它和 Java 不一样？