---
title: Array.prototype.slice及其他Array方法
date: 2018-04-10 14:19:27
tags: [js]
categories: 技术
---
用Array的方法，对不是Array的东西做一些事情...
<!-- more -->
---
call方法真是一个有意思的东西，它可以改变函数调用时this的值。而我们知道，在函数里，this指向了调用这个函数的环境对象，比如一道经典面试题：

```javascript
var num = 2;
var obj = {
  num: 1,
  show: function () {
    console.log(this.num)
  }
};
var foo = obj.show;
obj.show();/* 显示1；show是被obj调用的，所以this指向obj */
foo();/* 显示2；相当于global.foo(),所以this指向global，如果在浏览器里global就是window */
```
**换句话说，如果一个对象obj上有方法foo，你可以通过`obj.foo()`调用；如果没有obj上没有方法foo，`obj.foo()`是会报错的，但是，使用`foo.call(obj)`，可以强行达到`obj.foo()`的效果，比如：**
```javascript
function foo(){
    console.log(this.num);
}
var obj = {
    num: 1
}
foo.call(obj);// 1
```

`Array.prototype.slice.call`的用处就是这样，可以在array-like（类数组，就是长得像数组，但不是数组）的对象上强行使用slice方法，比如：`Array.prototype.slice.call(arguments)`就是把`arguments`对象转化为数组。当然，除了`arguments`，我们还能在`HTMLCollection`或`NodeList`身上使用。**那么到底什么算是类数组呢？**

**有length属性的对象。**

比如：

```javascript
var obj1 = {
  0: 'Tom',
  1: 'Jack',
  2: 'Jason',
  length: 3
}
var arr = [].slice.call(obj1);
console.log('arr: ', arr);/* [ 'Tom', 'Jack', 'Jason' ] */
```
那如果没有length呢？

```javascript
var obj1 = {
  0: 'Tom',
  1: 'Jack',
  2: 'Jason'
}
var arr = [].slice.call(obj1);//* [] */
```
原来没有length属性的对象也会被转为数组，只不过认为它length=0而已。

那如果对象的属性没有按照0-n顺序乖乖排好呢？

```javascript
var obj1 = {
  1: 'Tom',
  3: 'Jack',
  5: 'Jason',
  7: 'Dave',
  foo: 'bar',
  length: 6
}
var arr = [].slice.call(obj1);/* [ , 'Tom', , 'Jack', , 'Jason' ] */
```
原来转化的时候，会以`length`为基础，生成一个长度为`length`的数组，`obj`的属性是数组的有效`index`的话，就会把对应值填入到对应位置，其他的位置找不到值，就会填入`undefined`。

所以前面的说法其实不对，所有的对象都可以被视为类数组，有`length`的视为长度为`length`的数组，没有的，视为长度为0的数组。

**以`length`属性为基础**

这句话很重要。

另外，`call`方法的参数如果是`原始值类型`，会传入它的`自动包装对象`：
```javascript
var arr = [].slice.call('hello');
```

等价于：
```javascript
var arr = [].slice.call(new String('hello'));/* [ 'h', 'e', 'l', 'l', 'o' ] */

因为new String('hello')就是
{
    0: "h",
    1: "e",
    2: "l",
    3: "l",
    4: "o",
    length: 5
}
```

以上就是`Array.prototype.slice.call`的一些细节，那么除了`slice`之外，`Array`对象还有很多其他的方法，这些方法是不是也能用到对象身上呢？
## Array.prototype.join ##
join方法是把数组转化为字符串的方法，具体表现不再赘述，看两个例子：
```javascript
var obj1 = {
  0: 'Tom',
  1: 'Jack',
  2: 'Jason',
  length: 6
}
var arr = [].join.call(obj1, '-');// Tom-Jack-Jason
var obj1 = {
  0: 'Tom',
  1: 'Jack',
  2: 'Jason',
}
var arr = [].join.call(obj1, '-'); // ''
```
还是那句话，**以`length`为基础**,没有`length`属性的，视为长度为0的数组。
## Array.prototype.push ##
这个方法比较好玩：

```javascript
var obj1 = {
  0: 'Tom',
  1: 'Jack',
  2: 'Jason',
  length: 6
}
var arr = [].push.call(obj1, 'Dave');
console.log('arr: ', arr);// 7，因为push方法返回的是push之后array的操作数
console.log('obj: ', obj1);// { '0': 'Tom', '1': 'Jack', '2': 'Jason', '6': 'Dave', length: 7 }
```
可以看到`obj1`里新增属性`6`，值为`'Dave'`，并且`length`也更新为`7`，这说明调用`push`时会对原有对象进行修改。
我们可以利用这个特性，比如当我们需要一个`obj1`的类数组副本时：

```javascript
var obj = {
  foo: 'foo',
  bar: 'bar',
  cei: 'cei'
};
var copy = {};
for (var i in obj) {
  [].push.call(copy, obj[i])
}
console.log(copy);// { '0': 'foo', '1': 'bar', '2': 'cei', length: 3 }
```
如果，没有传入`length`呢？

```javascript
var obj1 = {
  0: 'Tom',
  1: 'Jack',
  2: 'Jason'
}
var arr = [].push.call(obj1, 'Dave');
console.log('arr: ', arr);// 1
console.log('obj: ', obj1);// { '0': 'Dave', '1': 'Jack', '2': 'Jason', length: 1 }
```
这里的行为有些诡异，不过也更好地解释了**以length为基础**这句话：
没有`length`的时候，认为数组长度为`0`，并且会对`obj`进行修改，把属性0的值改为`Dave`.

那么，会举一反三的话，对于`pop`, `shift`和`unshift`这三个方法的行为应该能想象得出来，就不再赘述了。

## Array.prototype.reverse ##

```javascript
var obj1 = {
  0: 'Tom',
  1: 'Jack',
  2: 'Jason',
  length: 6
}
var arr = [].reverse.call(obj1);
console.log('arr: ', arr);// { '3': 'Jason', '4': 'Jack', '5': 'Tom', length: 6 }
console.log('obj: ', obj1);// { '3': 'Jason', '4': 'Jack', '5': 'Tom', length: 6 }
```
`reverse`的话，`arr === obj1`
## Array.prototype.sort ##

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].sort.call(obj1);
console.log('arr: ', arr);// { '0': 'a', '1': 'b', '2': 'c', length: 6 }
console.log('obj: ', obj1);// { '0': 'a', '1': 'b', '2': 'c', length: 6 }
```
`sort`也一样，`arr === obj1`
## Array.prototype.concat ##
`concat`的表现就不是我们意料之中的了：

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}

var add = {
  foo: 'foo',
  bar: 'bar'
}
var arr = [].concat.call(obj1, add);
console.log('arr: ', arr);// [ { '0': 'c', '1': 'b', '2': 'a', length: 6 }, 'foo', 'bar' ]
console.log('obj: ', obj1);// { '0': 'c', '1': 'b', '2': 'a', length: 6 }
```

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].concat.call(obj1, 'foo', 'bar');
console.log('arr: ', arr);// [ { '0': 'c', '1': 'b', '2': 'a', length: 6 }, 'foo', 'bar' ]
console.log('obj: ', obj1);// { '0': 'c', '1': 'b', '2': 'a', length: 6 }
```

可以看到`obj1`并不会改变，不会像`push`一样会接着形成一个类数组的对象.
## Array.prototype.splice ##

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].splice.call(obj1, 0, 1);
console.log('arr: ', arr);// [ 'c' ]
console.log('obj: ', obj1);// { '0': 'b', '1': 'a', length: 5 }
```

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].splice.call(obj1, 1, 0, 'foo','bar');
console.log('arr: ', arr);// []
console.log('obj: ', obj1);// { '0': 'c', '1': 'foo', '2': 'bar', '3': 'b', '4': 'a', length: 8 }
```

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].splice.call(obj1, 1, 1, 'foo','bar');
console.log('arr: ', arr);// [ 'b' ]
console.log('obj: ', obj1);// { '0': 'c', '1': 'foo', '2': 'bar', '3': 'a', length: 7 }
```
`splice`的行为回归了，它现在对`obj1`产生影响，并且是我们预计的样子
## Array.prototype.every ##

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].every.call(obj1, function (val) {
  return val === 'a' || val === 'c'
});
console.log('arr: ', arr);// false
console.log('obj: ', obj1);// { '0': 'c', '1': 'b', '2': 'a', length: 6 }
```
## Array.prototype.filter ##

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].filter.call(obj1, function (val) {
  return val === 'a' || val === 'c'
});
console.log('arr: ', arr);// [ 'c', 'a' ]
console.log('obj: ', obj1);// { '0': 'c', '1': 'b', '2': 'a', length: 6 }
```
## Array.prototype.forEach ##

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].forEach.call(obj1, function (val) {
  return val + ' add';
});
console.log('arr: ', arr);// undefined
console.log('obj: ', obj1);// { '0': 'c', '1': 'b', '2': 'a', length: 6 }
```
## Array.prototype.map ##

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].map.call(obj1, function (val) {
  return val + ' add';
});
console.log('arr: ', arr);// [ 'c add', 'b add', 'a add', , ,  ]
console.log('obj: ', obj1);// { '0': 'c', '1': 'b', '2': 'a', length: 6 }
```
## Array.prototype.reduce ##

```javascript
var obj1 = {
  0: 'c',
  1: 'b',
  2: 'a',
  length: 6
}
var arr = [].reduce.call(obj1, function (pre, cur) {
  return pre + ' ' + cur
});
console.log('arr: ', arr);// 'c b a'
console.log('obj: ', obj1);// { '0': 'c', '1': 'b', '2': 'a', length: 6 }
```
