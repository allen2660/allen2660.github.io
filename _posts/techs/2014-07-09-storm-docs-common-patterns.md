---
layout: default
title:  common patterns
---

本文为Storm官方文档[CommonPatterns](http://storm.incubator.apache.org/documentation/Common-patterns.html)的读书笔记

本页描述了一些常见的Storm拓扑。

1. 流join
2. 批处理
3. BasicBolt
4. 内存缓存 + 域分组 二者结合
5. 流式统计 top N
6. TimeCacheMap，高效地保存最近被更新的东西的缓存。
7. CoordinatedBolt and KeyedFairBolt for 分布式 RPC

## Joins

流join将两个或多个数据流通过一个共同的域join在一起。然而通常的数据库join都是基于有限的输入和清楚的语义，而流式join输入是无限的，join的语义也是不明确的。

每个应用的join类型都是不同的。有的应用将两个流的有限时间窗口中的所有元组join在一起，而其他的应用期望join的每一边的每个域只有一个元组。有的应用的join又完全不同。这些join的共同点是多个流的分组是类似的。这在Storm中很容易实现，只要在多个输入流的同一个域上使用域分组。举例：
    
    builder.setBolt("join", new MyJoiner(), parallelism) 
        .fieldsGrouping("1", new Fields("joinfield1", "joinfield2")) 
        .fieldsGrouping("2", new Fields("joinfield1", "joinfield2")) 
        .fieldsGrouping("3", new Fields("joinfield1", "joinfield2"));

当然不同流不一定要求是同一个域名。

## 批处理

有时为了效率或者其他原因，你会批量处理一组组的元组，而不是单独的处理。举例来说，你会将更新db或者流聚合的操作批处理化。

如果想要数据处理的可靠性，正确的做法是在bolt等待做批处理的时候给一组元组一个实例变量。一旦你做批处理的时候，可以ack这组元组。

如果bolt提交元组，你可能想使用multi-anchoring来保证可靠性。查看[Guaranteeing message processing](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)获得更多可用性信息。

## BasicBolt

很多bolt都遵循一个简单的模式：读进一个元组、提交0或多个元组、然后立即确认输入元组。此类Bolts一般是函数或者过滤器。这种类型太普遍以至于Storm暴露了一个叫做IBasicBolt接口来自动化这种类型的bolt。查看[Guaranteeing message processing](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)获得更多信息。

## 内存缓存 + 域分组

在Bolt中保存内存缓存是常见的。结合了域分组的缓存尤其有用。举例来说，假设你有一个扩展短URL的bolt。你可以通过保存短链的LRU缓存来避免不断的做相同的HTTP请求。假设“urls”组件提交短URL，“expand”组件扩展这些短链到长链接并且缓存下来。考虑下面两段代码的不同：

    builder.setBolt("expand", new ExpandUrl(), parallelism) .shuffleGrouping(1);


    builder.setBolt("expand", new ExpandUrl(), parallelism) .fieldsGrouping("urls", new Fields("url"));

第二种方法拥有更有效地缓存，因为同样的URL永远都会发送到同一个task。这避免了task之间缓存的重复，提升了短链的cache命中率。

## 流式的top N

基于Storm的一种常见“持续计算”就是“流式top N”。假设你有一个bolt提交形如["value","count"]的元组，你想要一个bolt提交count为top N的元组。最简单的做法是对这个流做一个global grouping后进入一个bolt，再让这个bolt在内存中保存top N的元素的列表。

这个方法对于流的扩容来说显得不足，因为整个流要流入同一个task。一个更好的方法是并行的对流做分片、分别做top N计算，然后将这些topN merge起来。形如下面：
    
    builder.setBolt("rank", new RankObjects(), parallellism) 
        .fieldsGrouping("objects", new Fields("value")); 
    builder.setBolt("merge", new MergeObjects()) 
        .globalGrouping("rank");

这个方法行得通是因为第一个bolt的域分组使得数据被分区从而保证语义正确。你可以在storm-starter中看到[这个例子](https://github.com/nathanmarz/storm-starter/blob/master/src/jvm/storm/starter/RollingTopWords.java)。

## TimeCacheMap，高效地保存最近被更新的东西的缓存。

有的时候你想在内存中保存最近被“激活”的元素，而让一段时间没有被“激活”的元素过期。[TimeCacheMap](http://storm.incubator.apache.org/apidocs/backtype/storm/utils/TimeCacheMap.html)是做这件事情的一个有效地数据结构。并且其还提供一个item过期的回调。

## CoordinatedBolt and KeyedFairBolt for 分布式 RPC

当基于Storm做分布式RPC的时候，通常需要两个常见的模式。他们被封装为[CoordinatedBolt](http://storm.incubator.apache.org/apidocs/backtype/storm/task/CoordinatedBolt.html) 和 [KeyedFairBolt](http://storm.incubator.apache.org/apidocs/backtype/storm/task/KeyedFairBolt.html)，他们都是Storm发行标准库的一部分。

`CoordinatedBolt`封装了包含了你的逻辑的bolt，并且知道何时你的bolt接收到了任何给定请求的所有元组。直接使用流来做的话很麻烦。

`KeyedFairBolt`也封装了你的逻辑，同时保证你的拓扑同时处理多个DRPC调用，而不是一个个的调用。

查看[Distributed RPC](http://storm.incubator.apache.org/documentation/Distributed-RPC.html)以了解更多信息。