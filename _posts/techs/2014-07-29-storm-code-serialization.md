---
layout: post
title:  Storm源码-backtype.storm.serialization包
---

## Serialization

[serialization](http://storm.incubator.apache.org/documentation/Serialization.html)

元组由任意的类型组成。Storm是一个分布式系统，所以上面任务之间传递数据就需要用到序列化/反序列化。Storm使用[Kryo](http://code.google.com/p/kryo/)来做序列化。Kryo是一个弹性的、快速的序列化库，生成的二进制格式很小。

默认的，Storm可以序列化基本类型、字符串、字符数组、ArrayList、HashMap、HashSet，以及Clojure集合类型。如果在元组里想使用自己的类型，你需要注册自定义的序列化器。

### 动态类型

在Tuple中没有类型声明。你在field中放对象，Storm动态找出其序列化。在我们讨论serialization接口之前，让我们花一些时间理解下为什么Storm的元组是动态类型的。

在元组中加入静态类型会大大增加Storm API的复杂度。


### 自定义序列化