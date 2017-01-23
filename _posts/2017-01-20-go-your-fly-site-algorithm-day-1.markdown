---
layout: post
title: "算法学习之路 Day 1"
date: 2017-01-20
---

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
  
  选择排序的核心思想是：从一个列表中取出一个最小的值，放在左边的位置，每次取一个，直到取完为止，如：[2,5,3,1]，第一次取1，放在数组最左边[1,5,3,2]，依次，接着取2，放在1的右边...直到变为[1,2,3,5]。
  
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
