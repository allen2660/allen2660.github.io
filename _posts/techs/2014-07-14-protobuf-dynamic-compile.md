---
layout: default
title:  protobuf的动态编译
---

# 动态编译

## 使用场景

通常情况下，使用Protobuf的人会首先写好.proto文件，再使用protoc编译器编译成想要的目标语言文件。然后将这些自动生成的代码和应用程序一起编译。

在某些场景下，人们无法或者说不想预先知道.proto文件（比如Hive的Protobuf Serde）。这就需要动态编译.proto文件。

### C++

通过前面分析protobuf源码知道，Protobuf提供了compiler包来编译.proto文件，protoc就是这么用的。

### Java

Java目前没有找到直接编译.proto文件的方法，[这篇文章](http://blog.csdn.net/lufeng20/article/details/8736584)中讲解可以使用desc文件来build，但是生成desc文件还是要使用protoc二进制。这个待研究。

## Demo

# 反射