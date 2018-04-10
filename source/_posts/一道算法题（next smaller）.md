---
title: 一道算法题（next smaller）
date: 2018-04-10 14:34:16
tags: [js, 算法]
categories: 技术
---
挺有意思的一道题，这题的重点是数学理解。
<!-- more -->
---
题目如下：
-----

给定某一个正整数，组成这个数的数字不变，返回下一个比它小的数，如：

```javascript
nextSmaller(21) == 12
nextSmaller(531) == 513
nextSmaller(907) == 790
nextSmaller(51226262651257) == 51226262627551
```
如果没有下一个比它小的数字，或者下一个比它小的数字以0开头，则返回-1，如：

```javascript
nextSmaller(9) == -1
nextSmaller(111) == -1
nextSmaller(135) == -1
nextSmaller(1027) == -1 // 0721以0开头，所以返回-1
```


----------

算法描述（找到下一个比它小的数）：
-----------------
**1.find pivot：**这个数从右往左，一位位地来比较，如果第i位的数字，比第i+1位的数字大，则把第i位的数字置为pivot（标志位）。
**2.swap：**从pivot位向右，找到比pivot小的最大的那个数，并与pivot交换。
**3.sort：**交换后，把pivot位向右（注意是pivot的index，不是pivot的值）的所有数字降序排列。
这样，新得到的数就是下一个比它小的数。


----------

举例分析：
-----

比如我们拿54123来分析。
**1.find pivot：**从最右边的3来看，因为3没有第i+1位数字（3没有右边），所以左移一位到2；2比右边的3小，所以继续左移一位到1；同理，1也比2要小，所以继续左移一位到4；4比1要大了，那么把4置为pivot，这时停止。
**2.swap：**现在4是pivot，那么从4向右，有1，2，3三个数字，并且都比4小，这其中3是最大的，所以把4和3的位置交换，得到53124。
**3.sort：**交换后，pivot位上是3，把3往右的所有数字降序排列，得到53421，这就是下一个比54123小的数。


----------

为什么这样做？
-------

下面是我自己思考的为什么，不一定对。

对于每个数，它最小的排列只有1种情况，就是权位从高到低，数字从小到大排列。
比如123的最小排列就是123。
对于54123来说，从右向左可以分割为3，23，123，这三种情况都是最小排列。
也就是说，如果我们只对3或者23或者123进行重组，是没有变化的。
于是再向左进行分割，得到5和4123，发现4123已经不再是最小排列了，换句话说4123肯定有下一个比它小的数，就把4标记出来，对4123进行重组。
怎么重组呢？最高的权位，放上比4稍小的，也就是3，然后对于3xxx的后三位，放上值最大的情况，就是412的降序排列421，这样就能保证“3421是4123的上一个数”。


----------

总结：
---

这个算法一句话总结：对于某个数，从右向左找它的子列，如果子列是最小排列，那子列没法重组，继续向右；如果子列不是最小排列，就对子列进行重组。

感觉算法里，“分治”加“pivot（标志位）”的解法很常用啊，把复杂情况分成最简单的子情况，如果符合规律，那么子情况扩充，如果不符合规律，那么置上标志位，再进行考虑。


----------

下面是写得不怎么样的代码：
-------------

```javascript
function nextSmaller(num) {
  var arr = [];
  (num + "").split("").forEach(function (val) {
    arr.push(parseInt(val));
  });
  // 1st: find the pivot, if digit is greater than its right digit, it becomes a pivot
  var pivot = -1;
  for(var i = arr.length - 2; i >= 0;i--) {
    if (arr[i] > arr[i + 1]) {
      pivot = i;
      break;
    }
    if (i == 0) {
      return -1;
    }
  }
  // 2nd: find the least less than the pivot, and swap them
  var swap = -1;
  for (i = pivot + 1;i < arr.length;i++) {
    if (swap < 0) {
      if (arr[i] < arr[pivot]) {
        swap = i;
      }
    } else {
      if (arr[i] < arr[pivot] && arr[i] > arr[swap]) {
        swap = i
      }
    }
  }
  var _mem;
  _mem = arr[pivot];
  arr[pivot] = arr[swap];
  arr[swap] = _mem;
  if (arr[0] == 0) {
    return -1;
  }
  // 3rd: sort the right of pivot, decreasingly
  var firstArray = arr.slice(0, pivot + 1);
  var secondArray = arr.slice(pivot + 1);
  secondArray.sort(function (a, b) {
    return b -a;
  });
  // 4th: return val
  var result = (firstArray.concat(secondArray)).join("");
  if (result == num){
    return - 1
  }
  return parseInt(result);
}
```