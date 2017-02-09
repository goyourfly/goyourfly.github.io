---
title: "算法学习 - 查找算法"
date: 2017-02-09
---

### 顺序查找

* 时间复杂度：n

顺序查找是指：给定一个元素和一个集合，按照顺序依次的和元素进行比较，如果相等就停止比较。感觉这个算法太水了，傻子都能明白，但是据说是最实用的查找算法了😅。
 
但是这个算法有三个改进感觉挺有意思的：

1. 成功查找到移动到最前面
2. 成功查找到向前移动一位
3. 成功查找到移动到最后面

>输入法在你打出一个不常用字的时候，等你下次再去打这个字，你就会发现他的位置靠前了，
 
### 二分查找

* 时间复杂度：log n

二分查找是指：将有序集合折半，直到找到待查找元素，注意，前提是有序集合。

* 程序实现

 ```java
 	 //通过递归实现二分查找
     public static int search(int[] a, int from, int to, int b) {
        int index = (from + to) / 2;
        System.out.println("i:" + index);
        int value = a[index];
        if (value == b) {
            return index;
        } else if (value > b) {
            return search(a, from, index, b);
        } else {
            return search(a, index + 1, to, b);
        }
    }
 ```