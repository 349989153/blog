---
title: 阅读redux源码--applyMiddleware.js
date: 2018-04-10 13:49:07
tags: [js, redux, 源码]
categories: 技术
---
applyMiddleware.js就是升级dispatch方法，使之更强大。
<!-- more -->
---
applyMiddleware.js
------------------
接收若干个函数，统称为`...middlewares`,返回一个函数a（代码里是匿名函数，我这里为了表达方便加个名字）。这个函数a接收参数createStore，返回一个Redux Store的对象。因为applyMiddleware.js是当作enhancer用在createStore里面的，所以返回的这个函数a，接收一个createStore为参数。

然后呢创建了一个`store`，声明一个dispatch函数，声明一个chain数组。创建middlewareAPI这个对象，它的getState就是store的getState，dispatch是一个函数。

然后遍历middlewares，对其中的每个middleware都用middlewareAPI执行一下，返回值都是接收一个参数的函数，返回值塞到chain里。举个简单的例子，比如`chain = [b, c, d]`。

然后用compose把chain和store.dispatch组合起来，变成一系列变换，再赋给新的dispatch：

```javascript
dispatch = b(c(d(store.dispatch)))
```

最后返回一个redux store，这个store除了dispatch之外都是原装的，和createStore.js返回的一样。dispatch则是经过middlewares的一系列变换后生成的dispatch。
## 总结 ##
applyMiddleware的目的就是**加强dispatch**，让dispatch在原有的功能上，进行函数变换使之具有新的功能。

它接收若干个middleware作为参数。每个middleware能够接收到一个object作为参数，这个参数有getState和新的dispatch；然后返回一个只接收单个参数的函数。这个函数是要被compose使用的。

然后使用compose制造一个新的dispatch，新的dispatch是按照middlewares从右往左的顺序执行的。

middleware接收一个object作为参数，返回一个函数，这个函数接收一个next作为参数，返回一个函数，这个函数接收一个action作为参数。