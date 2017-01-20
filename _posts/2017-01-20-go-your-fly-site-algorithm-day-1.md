---
layout: post
title: "算法学习之路 Day 1"
date: 2017-01-20
---
参加工作3年了，做了很多的App，对Android这一套Api的使用也滚瓜烂熟，感觉没什么可以扩展的，但是只是会一个简单的使用别人写好的Api，那我跟搬砖头有什么区别呢？那怎么能提高自己的能力，有能力去做一些创造性的东西，而不是做一个辛苦的码农呢？经过别人指点和自己探究，我觉得应该从程序员最根本的东西 -- **算法**入手


## 排序算法

### 插入排序

* 时间复杂度：n2
* 空间复杂度：1
  
  插入排序的核心思想是：将一个列表的分为左边有序区和右边无序区域，然后，从无序区去一个值A，和有序区的值B从右往左比较，如果发现A < B，则将B往右移动一个位置，腾出B原来的位置，然后，A继续和B左边的值C比较，如果A < C ？ 重复以上步骤，否则把A放在C的右边（既B原来的位置），以此类推直到无序区域的数值都移动到有序区为止。

* 程序实现

  ```javascript
  var a = [9,6,7,8,5,2,3,1,4];
  console.log("a not order:" + a);
  for(var i = 0; i < a.length; i++){
     var temp = a[i];
      var j;
     for(j = i - 1; j >= 0 && a[j] > temp; j--){
          a[j + 1] = a[j];
      }
     a[j + 1] = temp;
  }
  console.log("a order:" + a);
  ```
  
### 选择排序

* 时间复杂度：n2
* 空间复杂度：1

  选择排序的核心思想是：从一个列表中取出一个最小的值，填入一个空的列表中，每次取一个，直到取完为止，如：[2,5,3,1]，第一次取1，放在空数组中[1]，依次...直到左边为[]，右边为[1,2,3,5]。
  
* 程序实现

  ```javascript
  var a = [9,6,7,8,5,2,3,1,4];
  var length = a.length;
  for(var i = 0; i < length; i++){
      var minIndex = i;
     for(var j = i + 1; j < length; j++){
          if(a[minIndex] > a[j]){
              minIndex = j;
          }
      }
      var temp = a[i];
      a[i] = a[minIndex];
      a[minIndex] = temp;
  }
  console.log("a:" + a);
  ```
