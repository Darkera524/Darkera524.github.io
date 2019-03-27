---
layout: post
title:  "python __getattr__场景"
categories: python
tags: python python-object
author: 大可
---

## 自定义属性访问

### 诸方法
- object.__getattr__(self, name)
    - 仅当访问__getattribute__()或者__get__()访问raise AttributeError的时候调用
- object.__getattribute__(self, name)
    - 当访问实例变量时被无条件的调用，为防止该方法被递归调用，应当在方法内仅使用基本类方法。
    - 实例变量实质上存储在object.__dict__中
- object.__get__(self, instance, owner)
    - 当访问类实例的类变量时被调用

### 一些实践例子

#### 像访问属性一样访问dict的键值对

```python
class ObjectDict(dict):
    def __init__(self, *args, **kwargs):
        super(ObjectDict, self).__init__(*args, **kwargs)

    def __getattr__(self, name):
        value =  self[name]
        ## 递归访问字典内容
        if isinstance(value, dict):
            value = ObjectDict(value)
        return value

if __name__ == '__main__':
    od = ObjectDict(asf={'a': 1}, d=True)
    print od.asf, od.asf.a     # {'a': 1} 1
    print od.d                 # True
```

#### 属性延迟初始化

```python
class WidgetShowLazyLoad(object):
    def fetch_complex_attr(self, attrname):
        '''可能是比较耗时的操作， 比如从文件读取'''
        return attrname

    def __getattr__(self, name):
        if name not in self.__dict__:
             self.__dict__[name] = self.fetch_complex_attr(name) 
        return self.__dict__[name]

if __name__ == '__main__':
    w = WidgetShowLazyLoad()
    print 'before', w.__dict__
    w.lazy_loaded_attr
    print 'after', w.__dict__
```

#### adapter wrapper模式
- 初始化Wrapper时，x为可迭代对象，其中的对象拥有相同接口的对象，当调用Wrapper的实例对象的A方法时，由于Wrapper中并未定义A方法，将触发调用__getattr__寻找相应对象，根据方法定义将分别调用所有初始化时被传入的对象的A方法
- 极好的贯彻了组合优于继承的观念，封装了多个实例对象

```python
class Wrapper: 
    def __init__(self, x): 
        self.x = x 
    def __getattr__(self, name): 
        def f(*args, **kwargs): 
            for y in self.x: 
                getattr(y, name)(*args, **kwargs) 
        return f
```

## 参考
- https://www.cnblogs.com/xybaby/p/6280313.html
- https://docs.python.org/3/reference/datamodel.html#object.__getattr__
- https://stackoverflow.com/questions/55211193/wrapping-homogeneous-python-objects#