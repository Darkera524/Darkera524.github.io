---
layout: post
title:  "grpc package"
categories: golang python grpc
tags: golang python grpc
author: 大可
---


## package的作用
在.proto⽂件中使⽤package声明包名，避免命名冲突。
```protobuf
syntax = "proto3";
package foo.bar;
message Open {...}
```
在其他的消息格式定义中可以使⽤包名+消息名的⽅式来使⽤类型，如：
```protobuf
message Foo {
 ...
 foo.bar.Open open = 1;
 ...
}
```

在不同的语⾔中，包名定义对编译后⽣成的代码的影响不同：

C++ 中：对应C++命名空间，例如Open会在命名空间foo::bar中

Java 中：package会作为Java包名，除⾮指定了option jave_package选项

Python 中：package被忽略

Go 中：默认使⽤package名作为包名，除⾮指定了option go_package选项

JavaNano 中：同Java

C# 中：package会转换为驼峰式命名空间，如Foo.Bar,除⾮指定了option csharp_namespace选项

## Method Not Found 错误
在client调用时出现“Method Not Found”错误可能的情况为
- 服务端确实未曾实现被调用方法
- 服务端与客户端的proto package不同
