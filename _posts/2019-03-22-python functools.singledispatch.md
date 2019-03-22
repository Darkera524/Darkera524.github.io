---
layout: post
title:  "python functools.singledispatch"
categories: python
tags: python python-method std-module
author: 大可
---

## 单分配泛型函数

### 泛型函数（generic function）
为不同的类型实现相同操作的多个函数所组成的函数，在调用时由调度算法来决定使用哪个实现

### 单分配（single dispatch）
是泛型函数的一种分派形式，其实现是基于单个参数的类型来选择的

## 为什么添加该特性
pythonic角度：在没有泛型函数机制的情况下，如要实现相同效果，则需要实现分派函数，通过判断参数类型来调用不同函数，但是在代码中根据参数类型执行不同动作很不pythonic，同时扩展性也比较差，故pep443决定使用装饰器来实现重载函数，来实现泛型

## 用法
用singledispatch来装饰函数来生成泛型函数，注意分派只根据第一个参数的类型

```python
# 用singledispatch装饰的函数为泛型的base函数，该函数用于在找不到某类型合适的泛型函数时使用，针对object类型
from functools import singledispatch
@singledispatch
def fun(arg, verbose=False):
    if verbose:
        print("Let me just say,", end=" ")
    print(arg)

# 注册泛型函数有三种方式

# 方式一：利用函数参数注解（3.7新增）
@fun.register
def _(arg:int, verbose=False):
    if verbose:
        print("int object")
    print(arg)

# 方式二：在register()方法中指明类型
@fun.register(complex)
def _(arg, verbose=False):
    if verbose:
        print("complex object")
    print(arg.real, arg.imag)

# 方式三：函数式注册
def nothing(arg, verbose=False):
    print("nothing")
fun.register(type(None), nothing)

# register方法的返回结果为原函数，故可以支持一个方法同时支持多种类型
from decimal import Decimal
@fun.register(float)
@fun.register(Decimal)
def _(arg, verbose=False):
    if verbose:
        print("float or Decimal")
    print(arg/2)

if __name__ == "__main__":
    fun(1.3)
    fun(None)
    fun(3)
    fun("Asd")
```

## 实现机理
- 使用字典自由变量来保存注册的函数，键为参数类型，值为函数对象
- 检索时，当参数的类型在字典中，则直接调用，如果不在字典中，则对传入对象的MRO列表进行检索，当注册类型中含有ABC对象时，传入对象的MRO列表也会新增对象关于ABC类相关内容
- 当出现歧义，无先后顺序时，会报错；但如果显式声明多继承，则不会报错，遵循MRO检索规则

## 参考
- https://docs.python.org/zh-cn/3.7/glossary.html
- https://docs.python.org/zh-cn/3.7/library/functools.html
- https://www.python.org/dev/peps/pep-0443/
