---
title: 《js函数式编程》第2章
date: 2018-04-11 09:59:11
tags: [js, 编程范式, 读书, 函数式]
categories: 草稿
---

第2章讲了。

<!-- more -->
---

什么是函数式编程？

静态类型、模式匹配、不变性、纯度等。

但是本文的主要观点，是“促进”和“一等函数”。

一等是描述语言里使用方式最多的等级，比如字符。

它可以用在数组、可以用变量储存、可以是对象属性的值、可以作为参数、可以被函数返回。

一等函数，表示函数也能以上述的方式使用。

# 函数式和其他范式的区别

## 函数式和命令式的区别

命令式就是把我们自然语言的描述，逐句地转化成程序语言。

函数式则是区分我们自然语言里的某一些概念，用函数把这些概念表达出来，再用函数的组合来表达概念的组合。



所以我觉得，这其实是自然语言表达时候的区别，也就是说自然语言也有命令式和函数式表达的区别，而不仅仅是函数式。



我们通常用的表达是命令式的，这也使我们熟悉命令式。

比如书里举的例子：

**命令式**：

```
x=99，输出“墙上有X瓶啤酒，拿一个下来，分给大家，墙上还有x-1瓶啤酒”，x--
```

**函数式：**

```
重复x次，每次做f(x)
```

## 函数式和面向对象的区别

## 函数式和元编程的区别

# Applicative编程

Applicative编程就是把函数A作为参数传递给函数B。

## 集合中心编程

对于集合，我们要有统一的处理方式。比如map，就是对集合里所有元素遍历；reduce，就是遍历和暂存；filter，就是遍历筛选出符合条件的。

# 数据思考

这里主要就是讲了underscore.js的各种函数的处理结果：基本都是数组。但是你还可以用一个普通object来作为一种简单的关联性数据存储。

表状数据中，一行就是一个object，一格就是一个键值对。但是用underscore.js处理出来的数据基本都是数组，怎么办呢？于是作者用underscore.js的一些方法，造出了3个与数据库语句类似的方法：select，as，where。用这三个方法操作对象数组，就像用sql语句处理数据库表一样。

