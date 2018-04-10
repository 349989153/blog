---
title: 《js函数式编程》读书笔记
date: 2018-04-10 16:01:44
tags: [js, 编程范式, 读书, 函数式]
categories: 技术
---

阅读《Javascript函数式编程》时做的笔记。

<!-- more -->
---

# 胡乱说话

什么是高阶函数，什么是阶？

函数可以返回函数，这就是高阶函数；被返回的函数里又可以返回函数，每返回一次就叫做一阶。



为什么需要高阶函数？

每一阶执行的时候，我们都可以往里填上参数，使其更细节；即使不填入参数，我们也能写一些语句，使得下一阶函数的细节更具体。



比如我们理解`redux-thunk`中的`index.js`:

```javascript
function createThunkMiddleware(extraArgument) {
  return ({ dispatch, getState }) => next => action => {
    if (typeof action === 'function') {
      return action(dispatch, getState, extraArgument);
    }

    return next(action);
  };
}
```

`createThunkMiddleware`返回一个函数，这个函数又返回一个函数，又返回一个函数，又返回一个函数……噢天呐，别说理解，看明白这个都要花上好半天。

但是我们按照之前阶的概念来理解，每返回一个函数，就是一阶，并且这一阶一定做了什么不一样的事情，那么我们就可以列出来这样一个表：

- 第一阶：有或没有额外参数
- 第二阶：传入`store`的`dispatch`和`getState`
- 第三阶：传入之前所做的其他变换（next）
- 第四阶：传入action

**高阶的本质：为下一阶的执行提供上下文。而上下文，是解释某个概念的含义和用途。**

所以上面那段代码段，用这个思路去解释的话，就是：

> 如果`action`是函数，那么调用`action`给他传入`dispatch`，`getState`，和`extraArgument`；否则，直接返回`next`对`action`的执行结果。

`extraArgument`是什么？第一个函数执行后搞定。`dispatch`和`getState`是什么？第二个函数执行后搞定。`next`是什么？第三个函数执行后搞定，`action`是什么？第四个函数执行后搞定。

**把要搞定的东西说成一句话，这叫需求；把自然语言的这一句话，转化成代码，这叫编程；把自然语言的这一句话，里面需要的各个元素，用函数慢慢地一个个注入进去，这叫函数式编程。**

------

# 用函数式思维重构代码

“可变的地方，我们用函数把参数注入进去”，是上一节最后一句话的缩略版，我打算用这个思路把之前的代码重构一下。在我们app的代码里，有如下的代码：

> 获取今天的日期，然后输出“2017年9月”等
>

```javascript
const getCurrentYearMonth = () => {
  const date = new Date();
  const year = date.getFullYear();
  const month = date.getMonth() + 1;
  return `${year}年${month}月`;
};
```

> 获取今天的日期，然后返回数组[年，月，日]，其中月、日小于10的话就补0

```javascript
const getTodayStringArray = () => {
  const time = new Date();
  const year = time.getFullYear();
  const month = time.getMonth() + 1;
  const date = time.getDate();
  return [
    year,
    month < 10 ? '0' + month : month,
    date < 10 ? '0' + date : date
  ];
};
```

> 返回今天加或减num天的日期数组。

```javascript
const getDayStringArray = num => {
  if (!num || typeof num !== 'number') {
    num = 0;
  }
  const time = new Date();
  time.setDate(time.getDate() + num); //设置日期
  const year = time.getFullYear();
  const month = time.getMonth() + 1;
  const date = time.getDate();
  return [
    year,
    month < 10 ? '0' + month : month,
    date < 10 ? '0' + date : date
  ];
};
```

> 返回今天的字符串，如"20170201"

```javascript
const getTodayString = () => getTodayStringArray().join('');
```

> 给8位日期字符加上短横线。

```javascript
const joinDayStringWithHyphen = dayString => {
  const year = dayString.slice(0, 4);
  const month = dayString.slice(4, 6);
  const day = dayString.slice(6, 8);
  let result = '';
  result += year;
  if (month) {
    result += '-' + month;
  }
  if (day) {
    result += '-' + day;
  }
  return result;
};
```

## 思考时间

看着头有点大，因为这几条代码粗看好像相同，细看却又不一样，总而言之就是四个字，有点恶心。现在我们梳理一下，我们对于日期的需求：

1. **获取日期对象**，这个日期对象可能是今天，也可能是今天加减num天。
2. **把日期对象划分成一个数组**，[年，月，日，时，分，秒]
3. **对于数组里面的各个元素进行处理**，比如除了“年“之外，其他小于0的要补前置0。
4. **划分输出范围**，比如有时候要输出年月，有时候要输出年月日，有时候只需要输出时分秒。
5. **加上连接词**，比如有的要“年”和“月”，有的要短横线，有的什么都不要

除了第2条之外，其他的需求都是比较有操作空间的，所以来写代码吧！这次我们要写的可不是什么垃圾，而是集数种日期需求于一身的“要你命三千的时间处理”，命名为timeKing马马虎虎啦。

## 编码时间

```javascript
const timeKing = dateObject => {
  // 输入必须为date对象。
  if (Object.prototype.toString.call(dateObject) !== '[object Date]') {
    throw new Error('parameter must be a date Object')
  }

  const year = dateObject.getFullYear();
  const month = dateObject.getMonth() + 1;
  const date = dateObject.getDate();
  const hour = dateObject.getHours();
  const min = dateObject.getMinutes();
  const sec = dateObject.getSeconds();
  const array = [year, month, date, hour, min, sec];
};

```

然后针对需求3写代码：

```javascript
const timeKing = dateObject => {
  // 输入必须为date对象。
  if (Object.prototype.toString.call(dateObject) !== '[object Date]') {
    throw new Error('parameter must be a date Object')
  }

  const year = dateObject.getFullYear();
  const month = dateObject.getMonth() + 1;
  const date = dateObject.getDate();
  const hour = dateObject.getHours();
  const min = dateObject.getMinutes();
  const sec = dateObject.getSeconds();
  const array = [year, month, date, hour, min, sec];

  return handlerForArrayItem => {// 对于数组里面的各个元素进行处理
    if (handlerForArrayItem === undefined) {
      handlerForArrayItem = arg => arg;
    }
    const handledArray = handlerForArrayItem(array);
  }
};
```

然后针对需求4写代码：

```javascript
const timeKing = dateObject => {
  // 输入必须为date对象。
  if (Object.prototype.toString.call(dateObject) !== '[object Date]') {
    throw new Error('parameter must be a date Object')
  }

  const year = dateObject.getFullYear();
  const month = dateObject.getMonth() + 1;
  const date = dateObject.getDate();
  const hour = dateObject.getHours();
  const min = dateObject.getMinutes();
  const sec = dateObject.getSeconds();
  const array = [year, month, date, hour, min, sec];

  return handlerForArrayItem => {// 对于数组里面的各个元素进行处理
    if (handlerForArrayItem === undefined) {
      handlerForArrayItem = arg => arg;
    }
    const handledArray = handlerForArrayItem(array);

    return outputRange => {// 划分输出范围
      if (outputRange === undefined) {
        outputRange = arg => arg
      }

      const rangedArray = outputRange(handledArray);
    }
  }
};
```

最后针对需求5写代码：

```javascript
const timeKing = dateObject => {
  // 输入必须为date对象。
  if (Object.prototype.toString.call(dateObject) !== '[object Date]') {
    throw new Error('parameter must be a date Object')
  }

  const year = dateObject.getFullYear();
  const month = dateObject.getMonth() + 1;
  const date = dateObject.getDate();
  const hour = dateObject.getHours();
  const min = dateObject.getMinutes();
  const sec = dateObject.getSeconds();
  const array = [year, month, date, hour, min, sec];

  return handlerForArrayItem => {// 对于数组里面的各个元素进行处理
    if (handlerForArrayItem === undefined) {
      handlerForArrayItem = arg => arg;
    }
    const handledArray = handlerForArrayItem(array);

    return outputRange => {// 划分输出范围
      if (outputRange === undefined) {
        outputRange = arg => arg
      }

      const rangedArray = outputRange(handledArray);

      return conjunction => {// 加上连接词。
        if (conjunction === undefined) {
          conjunction = arg => arg.join('')
        }
        
        return conjunction(rangedArray);
      }
    }
  }
};
```

这个...时间处理王中王怎么用呢，emmmm:

```javascript
timeKing(/* 时间对象 */)(/* 对每个元素做处理 */)(/* 划分输出的范围 */)(/* 加上连接词 */);
```

## 测试时间

还是看看用这个王中王怎么实现我们之前的需求吧。

> 获取今天的日期，然后输出“2017年9月”等

```javascript
const addFrontZero = number => +number < 10 ? '0' + number : '' + number;

const addFrontZeroHandler = array => array.map((item, idx) => item >= 1970 ? item : addFrontZero(item));

const CNConjunction = array => {
  return array.map((item, idx) => {
    switch (idx) {
      case 0:
        return item + '年';
      case 1:
        return item + '月';
      case 2:
        return item + '日';
      case 3:
        return item + '时';
      case 4:
        return item + '分';
      case 5:
        return item + '秒';
      default:
        return item;
    }
  }).join('')
};

const yearMonthRange = array => array.slice(0, 2);

console.log(timeKing(new Date())(addFrontZeroHandler)(yearMonthRange)(CNConjunction));
```

> 获取今天的日期，然后返回数组[年，月，日]，其中月、日小于10的话就补0

```javascript
const yearMonthDayRange = array => array.slice(0, 3);
console.log(timeKing(new Date())(addFrontZeroHandler)(yearMonthDayRange)(array => array));
```

> 返回今天加或减num天的日期数组。

```javascript
const todayPlusDaysObject = days => {
  const date = new Date();
  date.setDate(date.getDate() + days);
  return date;
};
console.log(timeKing(todayPlusDaysObject(-10))(addFrontZeroHandler)(yearMonthDayRange)(array => array));
```

> 返回今天的字符串，如"20170201"

```javascript
console.log(timeKing(new Date())(addFrontZeroHandler)(yearMonthDayRange)());
```



## 关于时间王中王的总结

优点：

写起来思路似乎稍微清晰了一点。

每个阶段的处理函数只需要写一次。

缺点：

对使用者很不友好。

```javascript
timeKing()()()();// 这4个坑你怎么知道哪个坑放啥
```

要好好用王中王，基本上你必须和作者一样了解它，了解它有几个坑位，了解它每个坑位的输入和期待输出都是什么，了解我写的变换函数应该放在哪个坑位……emmmmmm

也许是我对函数式理解不够，才会有这种情况发生，所以下一阶段目标就是优化这个想法。



