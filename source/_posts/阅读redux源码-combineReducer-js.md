---
title: 阅读redux源码--combineReducer.js
date: 2018-04-10 14:08:26
tags: [js, redux, 源码]
categories: 技术
---
combineReducer是redux的2大核心之一，它为reducer提供了code split的功能，避免我们的reducer文件变得太长。
<!-- more -->
---
## combineReducer.js ##
combineReducer基本上是我们工程中最常用到的功能。


----------
源码的自然语言描述：
接收一个参数reducers。用Object.keys获取reducers的key，然后循环。在循环里，如果这个key的值是一个function，那么就把这个key赋给finalReducer。然后搞出一个finalReducerKeys。

然后是一个try语句，执行`assertReducerShape(finalReducers)`，如果有报错，把错误存到shapeAssertionError里面去。

最后返回一个函数combination，就是一个大的reducer，如果有shapeAssertionError，抛出错误；如果是在开发模式，调用getUnexpectedStateShapeWarningMessage来发警告（注意，不是错误，不会停止代码的运行）。
然后前置处理结束，开始真正的处理逻辑。设hasChanged为false，nextState为{}，对finalReducerKeys进行循环，对于每个key，获得reducer，然后取出previousStateForKey，然后使用reducer生成nextStateForKey，并且把nextStateForKey赋到nextState上面去。然后修改hasChanged的值。最后，如果hasChanged为true，返回nextState，否则返回state。

----------
