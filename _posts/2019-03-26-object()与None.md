---
layout: post
title:  "python object()与None"
categories: python
tags: python python-object
author: 大可
---

## 问题

### None为什么有问题
- 当None作为本地变量未初始化时的值或者关键字参数的默认值时，可能会出现程序漏洞，原因在于None可能是一个合理输入，并且None是任何人都能输入的值，虽然是unique的，但却对所有人可见，在这种情形下，程序无法判断初始化为None的本地变量没被改变过还是被赋值后的结果为None，无法判断None究竟是关键字参数的默认值还是调用者传入的，此时，一些判断无法进行，从而出现问题
- 本质在于：None是个单例值

### 什么时候有问题
仅当其他人能够使用None作为合理输入时，问题出现

### None和object()的异同
None和object()均不与其他任何对象相等，但是与None不同，None可被任何调用者所使用，而object()无法被任何人随意得到

### 如何解决
使用object()生成一个新unique的对象，将其作为关键字参数的默认值和用来表示局部变量未初始化的状态，这样，在该对象的作用域之外，调用者无法传入相同的object对象，使得使用None会造成的问题得到解决

### 为什么是object(),而不是[],也不是MY_CLASS()
诚然，这三者均能解决上述问题，生成外界不可见的唯一对象，但是：
- 当我们看到列表时，我们一般期待其能够完成作为列表的职责
- 当我们看到我们自己创建的类对象，我们一般期待他能够做些什么，有额外的作用
- 而当我们看到object(),我们不能期待它能够做任何其他的事情，我们只能够关心其唯一性，故不会造成阅读者的迷惑

### 什么时候使用object()
- unique initial values
- unique stop values
- unique skip values

## 补充

### initial value vs sentinel value
- sentinel value一般来说是指如果该值出现在一个算法中，那么此时算法应当结束，通常出现在循环和递归算法中
- 如上我们的描述显然不是sentinel value，故我们称之为initial value用以表示未进行真正赋值时的value


## 参考
- https://treyhunner.com/2019/03/unique-and-sentinel-values-in-python/
- https://en.wikipedia.org/wiki/Sentinel_value