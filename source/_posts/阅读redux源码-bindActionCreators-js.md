---
title: 阅读redux源码--bindActionCreators.js
date: 2018-04-10 13:55:46
tags: [js, redux, 源码]
categories: 技术
---
bindActionCreators是把action和dispatch绑定，这样本来2步走————创建action，然后dispatch————的代码，就变成了1步走————创建action之后马上dispatch。
<!-- more -->
---
## bindActionCreators.js ##
接收2个参数：actionCreators和dispatch。actionCreators应该是个function或者一个object，dispatch应该是个function。


----------
**当actionCreators是个function时：**
bindActionCreators得到的结果是一个function，这个function把参数传给actionCreators之后，直接dispatch了。所以是把**actionCreators**升级成**会直接dispatch的actionCreators**


----------
**当actionCreators是个object时：**
返回boundActionCreators，这是一个object。对actionCreators进行遍历，这时候问题转化成了**当actionCreators是个function时**，于是和上面一样的处理，把当前的actionCreator升级成会直接dispatch的actionCreator，然后赋给boundActionCreators。


----------

总结一句话：
------
之前的actionCreator创建了action之后，还要用dispatch来分发掉这个action，所以代码里可能会有非常多的dispatch。而bindActionCreators，就是把actionCreator和dispatch绑定在一起，升级成会直接dispatch的