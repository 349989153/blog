---
title: 阅读redux源码--compose.js
date: 2018-04-09 16:52:16
tags:
---
compose.js
----------
compose.js的源码相当简单：

```javascript
export default function compose(...funcs) {
  if (funcs.length === 0) {
    return arg => arg

  if (funcs.length === 1) {
    return funcs[0]

  return funcs.reduce((a, b) => (...args) => a(b(...args)))
}
```

**1.参数里面的...funcs起到了什么作用。**
**2.整个compose起到了什么作用。**


----------
...funcs起到的作用，就相当于把arguments转化为了一个数组。
但是细节可能有些不一样。我本来以为相当于：

```javascript
function compose(){
  console.log(...arguments);
}
```
 但是实际上是

```javascript
function compose(){
  console.log([...arguments]);
}
```
三个点`...`的语法，叫做`扩展语法`, 而`扩展语法`在函数中的应用，叫`剩余参数`。关于这点可以去看mdn的[扩展语法-mdn][1],这里不做过多展开。


----------
**关于compose做了什么，用中文解读一下源码好了。**
如果没有参数，返回一个函数，它接受什么就直接返回什么，不做任何处理。
如果有1个参数，那么返回这一个参数（这个参数应该是个函数）。
如果有2个以上的参数，那么对这些参数进行reduce操作，最后会返回一个函数，这个函数接收若干参数，并且执行最右边的函数，然后再用返回值执行之前组合好的函数。

举个例子吧，假设给`compose`传入的参数是`a,b,c`三个函数，那么`...funcs`就是`[a, b, c]`.我们用`cp`来表示之前的组合函数。
这样一来，执行了`compose`后，返回的函数是：

```jsx
(...args) => cp1(c(...args))
```
而cp1则是：

```javascript
(...args) => cp2(b(...args))
```
而cp2则是：
```javascript
(...args) => a(...args)
```
所以把cp2代入上面，再把cp1代入上上面就得到：
```javascript
(...args) => a(b(c(...args)))
```

----------
所以我们理解了，compose函数，其实是一个非常常用的抽象。它把传入其中的函数，按照从右到左的顺序按序执行，除了最右边的可以接收多个参数，剩下的函数都只能接收单个参数。**也就是说compose把参数转化成一个连续的函数变换。**

只是感觉执行顺序有待商榷，我觉得compose从左到右的顺序可能会好些，毕竟先写的函数先执行，逻辑上可能较为通顺点。
```javascript
return funcs.reverse().reduce((a, b) => (...args) => a(b(...args)))
```

----------
## 应用场景 ##
一个典型的应用场景是用echarts.js绘制图表。
比如我们经常会抽象出一个函数来生成echarts的option：

```javascript
export const stackLine = ({
  dataset,
  title,
  yAxisName,
  stackName,
  xAxisData
}) => {
  const option = {
    ...someOptionHere
  };
  return option;
};
```

但是写完后又会发现，这个图表的样式和那个有一点差异，怎么办呢？再写一个函数，然后把函数内把某些设置改一改？这样做的话，重复的代码太多，并且某些改动的通用型太低，对于颜色的改动，一个饼图和一个折线图可能要写2份代码。

一个较好的解决方法是，构造几个函数，它们接收一个option为参数，最后返回一个新的option。比如：

```javascript
// 修改颜色
const changeColor = option => {
    const opt = { ...option };
    opt.color = ['#000', '#fff'];
    return opt;
}
// 修改y轴样式
const changeYAxisStyle = option => {
    const opt = { ...option };
    opt.yAxis.labelStyle = {};
    return opt;
}
// 再修改label的啥啥啥
const changeLabel = option => {
    const opt = { ...option };
    opt.label.something = foo;
    return opt;
}
```
于是使用compose加上之前的stackLine，我们可以拿出一个按需修改的版本：

```javascript
const myStackLine = compose(changeColor, changeYAxisStyle, changeLabel ,stackLine)
```

这样代码的灵活度也很高，哪个修改不需要，直接删掉，或者需要加上哪个修改，直接往上加。另外对于折线图的修改，饼图也能适用，真是非常不错。


----------
## 后记 ##
不读源码不知道，原来隐藏着这么一个强大而被忽略掉的地方。另外react的组件其实也是一个object，那么其实也适用这种组合变换的模式。
  [1]: https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax#%E8%AF%AD%E6%B3%95

