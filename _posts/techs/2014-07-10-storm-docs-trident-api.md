---
layout: post
title:  trident api
---

本文为Storm官方文档[TridentAPI](http://storm.incubator.apache.org/documentation/Trident-API-Overview.html)的读书笔记


# Trident API 综述

## 本地Partition 操作

本地Partition操作不涉及网络传输，只是在每个batch上处理。

### Function

### Filter

### partitionAggregate

### stateQuery & partitionPersist

参见[trident state doc](http://storm.incubator.apache.org/documentation/Trident-state.html)

### projection

## repartition 操作

+ shuffle
+ broadcast
+ partitionBy
+ global
+ batchGlobal
+ partition

## 聚合操作

+ aggregate 单独的在流中的每个batch中聚合。
+ persistentAggregate 将流中的所有batch聚合在一起，存储在一个state中。

## 流分组操作

流上的groupBy操作在特定列上做partitionBy来实现repartition。如果在分组的流上做聚合，聚合会在每个group上做而不是在整个batch上做。persistentAggregate也可以运行在分组流上，结果会保存在MapState中，key是分组字段。

像常规流一样，分组流上的聚合也可以链接起来。

## Merge & Join

Merge较简单

Join比较麻烦：

    topology.join(
        stream1, new Fields("key"), 
        stream2, new Fields("x"), 
        new Fields("key", "a", "b", "c")
    );

当Join在多个spout上发生时，这些spouts的发送batch的状态会被同步。也就是说，一次batch join会包括每一边的batch。

如果想时间“时间窗口 Join”，你需要使用partitionPersist和stateQuery。一个spout过去一个小时的元组被保存在state中，以join key为key。然后stateQuery可以使用join key查询来实现join。
