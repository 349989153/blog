---
title: 阅读redux源码--createStore.js
date: 2018-04-10 14:04:00
tags: [js, redux, 源码]
categories: 技术
---
createStore是redux的2大核心之一，它创建了一个Redux store，并把state隐藏在闭包函数之中。
<!-- more -->
---
## createStore.js ##
接收3个参数：reducer，preloadedState，enhancer，返回一个object，概念上是Redux store，有dispatch，subscribe，getState，replaceReducer


----------


首先是各种检查，保证代码能够顺利运行，比如：

 - 如果第二个参数preloadedState是function，并且第三个参数enhancer不存在的话，那么挪一下，把第二个参数的function赋给enhancer，然后把preloadedState置为undefined。这是为了函数能够接收断续的参数，而不必把不需要的参数填上undefined。
 - 如果有enhancer，那么enhancer必须为function，并且return enhancer(createStore)(reducer, preloadedState)。这说明2件事：1.enhancer加强的是createStore。2.enhancer接收一个createStore函数，返回一个和createStore差不多的函数。
 - 如果reducer不是function，报错.


----------
然后声明一些变量：

 - `currentReducer`：入参reducer
 - `currentState`：入参preloadedState
 - `currentListeners`： 一个空数组
 - `nextListeners`：currentListeners
 - `isDispatching`： false


----------
然后是function getState：如果不是isDispatching，返回currentState。

----------
然后是function subscribe，接受一个入参listener。listener必须为function，所以先判断报错。
置isSubscribed为true，并且保证currentListeners和nextListeners不是同一个引用，再往nextListeners里push(listener).
返回一个函数，这个函数执行后，能unsubscribe这个listener。
**一个有意思的地方是，数组的indexOf方法可以对函数使用。** [].indexOf(listener)能够拿到listener在数组里的index。

然后是function subscribe，接受一个入参listener。subscribe就做了一件事情，就是把入参listener push到nextListeners里面，并且返回一个unsubscribe函数。

----------
然后是function dispatch，接受一个入参action，action必须是普通object，并且必须有type属性，并且当前不能是isDispatching。
接着，把isDispatching置为true，以currentState和action为参数，用currentReducer生成新的state，再更新到currentState。
然后把nextListeners赋给currentListeners，每个listener都执行一下。
最后把action返回来。


----------
然后是function replaceReducer，接收一个入参nextReducer，把currentReducer用nextReducer替换了，然后dispatch一个内置的action，只有type。这个dispatch会用nextReducer生成新的state，并且把currentListener都执行一遍。


----------
然后是function observable，没有入参，返回一个object。有2个属性，一个subscribe，一个$$observable。subscribe接收一个参数observer，它是个object，并且有next属性，用来对当前state进行处理。调用subscribe会执行一次observer方法，然后把observer用前面的subscribe方法注册到currentListener上面，这样在每次dispatch的时候，都会对currentState执行observer.next

