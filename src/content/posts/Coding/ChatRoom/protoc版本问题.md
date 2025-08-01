---
title: protoc版本问题
published: 2025-07-09T20:13:52Z
description: ''
image: ''
tags: [ChatRoom, C++, 网络编程, protobuf]
category: 'C++'
draft: false
---

# protobuf版本问题

## 问题描述：

在使用gRPC时，需要使用到protobuf，由于笔者安装过protobuf，版本为`3.21.9`，所以直接使用环境变量默认的`protoc.exe`编译了message文件以及生成对应的gRPC文件。结果报错如下：

```
This file was generated by a newer version of protoc which is incompatible with your Protocol Buffer headers. Please update your headers.
```

## 排查过程：

这个问题由生成的`message.pb.h`头文件的`#if`预编译指令检查输出，经检查为：

```
#if PROTOBUF_VERSION < 3021000
#error This file was generated by a newer version of protoc which is
#error incompatible with your Protocol Buffer headers. Please update
#error your headers.
```

由此可以知道，当前文件中定义的PROTOBUF_VERSION小于`3021000`，而笔者的protobuf版本为`3.21.9`，对应的`PROTOBUF_VERSION`为`3021001`，所以报错。

由于`PROTOBUF_VERSION`是指当前使用的`libprotobufd`库的版本号，而不是`protoc`编译器的版本号，所以需要检查当前使用的`libprotobufd`库的版本。

经过检查`设置->Linker->include`发现，在当前项目的配置中，使用的是`gRPC\third_party\protobuf\Debug\libprotobufd.lib`，对应的源码在`D:\dev\gRPC\third_party\protobuf\src\google\protobuf\stubs\common.h`中，查看源码发现这个库的版本为`3.13.0`，与`protoc`编译器的版本不一致。

对应的源码如下:
```cpp
// The current version, represented as a single integer to make comparison
// easier:  major * 10^6 + minor * 10^3 + micro
#define GOOGLE_PROTOBUF_VERSION 3013000

// A suffix string for alpha, beta or rc releases. Empty for stable releases.
#define GOOGLE_PROTOBUF_VERSION_SUFFIX ""
```

## 解决方法

+ **选择gRPC生成的protoc编译器**：使用`gRPC\third_party\protobuf\Debug\`目录下找到对应版本`protoc.exe`程序，编译生成`.pb.h`文件和`gRPC.pb.h`文件。
