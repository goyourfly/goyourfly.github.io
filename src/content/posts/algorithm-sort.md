---
title: "算法学习 - 排序算法"
published: 2017-01-20
tags: [算法]
category: 技术
---

## 排序算法

### 插入排序

* 时间复杂度：n^2

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

* 时间复杂度：n^2

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
  
### 中值排序

* 时间复杂度：
* 空间复杂度：

 中值排序的核心实现是：将原有的序列按照中间值分成两个小的序列，其中左边的序列都要小于中间值，右边的序列都要大于中间值，如此递归，直到应用到每个子序列中。
 
* 程序实现

  ```javascript
  function partition(a,left,right,pivotIndex){
	//首先，将中值和最右边数据交换
	var pivotValue = a[pivotIndex];
	a[pivotIndex] = a[right];
	a[right] = pivotValue;
	
	var store = left;
	for(var i = left;i < right; i++){
		if(a[i] <= pivotValue){
            //如果发现第i未知的数值小于中间值，
            //则将该值与从左往右第一个大于中间值的数值交换位置
			var temp = a[store];
			a[store] = a[i];
			a[i] = temp;
			store++;
            console.log("a:" + a);
		}
	}
	
	a[right] = a[store];
	a[store] = pivotValue;
  return store;
 }
 function selectKth(a,k,left,right){
    var idx = left;
    var p = partition(a,left,right,idx);
    if(left + k - 1 == p){
        return p;
    }
    if(left + k - 1 < p){
        return selectKth(a,k,left,p - 1);
    }else {
        return selectKth(a,k - (p - left + 1),p + 1,right);
    }
 }
 function mediansort(a,left,right){
    if(right <= left){
        return;
    }
  
    var mid = parseInt((right - left)/2);
    var p = selectKth(a,mid + 1,left,right);
    mediansort(a,left,left + mid-1);
    mediansort(a,left + mid + 1,right);
 }
 var a = [15,9,8,18,1,4,11,7,12,13,6,5,3,16,2,10,14];
 mediansort(a,0,a.length - 1);
 console.log("a:" + a);
  ```
 
### 快速排序
* 时间复杂度：n log n;
快速排序是指，随机的从待排序列表中取出一个元素，这个元素将列表切分成两个子列表，其中左边的列表元素都小于或等于随机取出的元素，右边数组的元素都大于或者等于随机取出的元素，然后再将子列表递归排序。

* 程序实现

  ```javascript
  function partition(a,left,right,pivotIndex){
	//首先，将中值和最右边数据交换
	var pivotValue = a[pivotIndex];
	a[pivotIndex] = a[right];
	a[right] = pivotValue;
	
	var store = left;
	for(var i = left;i < right; i++){
		if(a[i] <= pivotValue){
            //如果发现第i未知的数值小于中间值，
            //则将该值与从左往右第一个大于中间值的数值交换位置
			var temp = a[store];
			a[store] = a[i];
			a[i] = temp;
			store++;
		}
	}
	a[right] = a[store];
	a[store] = pivotValue;
  return store;
 }
 function insertSort(a,left,right){
  //console.log("a:["+left+","+right+"]" + a );
  for(var i = left; i <= right; i++){
     var temp = a[i];
     var j;
     for(j = i - 1; j >= 0 && a[j] > temp; j--){
          a[j + 1] = a[j];
     }
     a[j + 1] = temp;
  }
  //console.log("a:["+left+","+right+"]" + a );
 }
 function qSort(a,left,right){
    if(right <= left){
      return;
    }
    
    var midIndex = parseInt((left + right) / 2);
    midIndex = partition(a,left,right,midIndex);
    var minSize = 2;
    console.log("left:" + left + ",right:" + right + ",midIndex:" + midIndex);
    
    if(midIndex - 1 - left <= minSize){
    	//使用插入排序，将小于等于2的子列表排序
        insertSort(a,left,midIndex - 1);
    }else{
        qSort(a,left,midIndex - 1);
    }
  
    if(right - midIndex - 1 <= minSize){
        insertSort(a,midIndex + 1,right);
    }else{
        qSort(a,midIndex + 1,right);
    }
 }
 var a = [15,9,8,1,4,11,7,12,13,6,5,3,16,2,10,14];
 qSort(a,0,a.length - 1);
 console.log("A:" + a);
  ```
 
### 堆排序

### 计数排序
