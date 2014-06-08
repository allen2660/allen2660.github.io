---
layout:	default
title:  Protobuf 初探
---

# Protobuf 初探

标签（空格分隔）： protobuf c++

---

## 安装

下载： [链接](https://code.google.com/p/protobuf/downloads/list)


编译安装：

     ./configure --prefix=/home/liwei12/local/
     make
     make check
     make install

运行example中的例子
    
    cd examples
    export PKG_CONFIG_PATH=/home/liwei12/local/lib/pkgconfig/
    make cpp

## 语法

下面所有的内容，基本都可以在[官方文档](https://developers.google.com/protocol-buffers/docs/overview)里找到。

### 字段类型

### proto文件定义

1. required 每个标准消息必须要有一个
2. optional 消息格式中该字段可以有0个或1个。
3. repeated 可以有0-n个


## 编解码

有如下几类编解码方式：

编码 | 含义 | 适用类型
-----|------|----
0 | Variant  | int32, int64, uint32, uint64, sint32, sint64, bool, enum
1 | 64-bit  | fixed64, sfixed64, double
2 | Length-delimited | string, bytes, embedded messages, packed repeated fields
5 | 32-bit | fixed32, sfixed32, float

### Key的表示

    (id << 3) | wire_type
    
### Variantb 编码

该变长编码是基于128bit的变长编码。就是将正常二进制表示从低到高切分为7个一组，每一组第八位置0或1，1表示后面还有，0表示后面没有了。

整数 300 使用变长编码 ->1010 1100 0000 0010 现在分析一下二进制串,首先要了解变 长整数对于每个字节的编码都有一个最高有效位,如果为 1,表示后面还有字节; 如果为 0,表示后面没有字节。 这样把每个字节的第一位去掉,变成 010 1100 000 0010,变长整数采用的是小端编码,所以倒转一下字符串变成 000 0010 010 1100=300。


### sint32 sint64表示

对于 sint32 来说,采用 (n << 1)^(n >> 31)此方式编码，
对于 sint64 来说,采用 (n << 1)^(n >> 63)此方式编码

### length-delimited编码方法

顾名思义，就是先写入值的长度，再写值。有点类似于sequenceFile的处理方式。

## 其他



