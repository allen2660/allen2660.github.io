---
layout: post
title:  度量
---

本文为Storm官方文档[度量](http://storm.incubator.apache.org/documentation/Metrics.html)的读书笔记。

Storm暴露了一个度量接口来报告整个拓扑的汇总统计。通常用于跟踪你在Nimbus UI控制台上看到的数字：execute和ack数目、平均处理延迟、worker 堆占用等等。

## 度量类型

度量只需要实现一个方法，`getValueAndReset` - 做一些剩余工作以找到汇总值，并重置成初始值。举例来说，MeanReducer用总的运行时间除以运行数目得到平均值，然后将这两个值都初始化为0。

Storm提供了下列度量类型：

+ AssignableMetric
+ CombinedMetric
+ CountMetric
    + MultiCountMetric
+ ReduceMetric
    + MeanReducer
    + MultiReducedMetric


## 度量消费者

## 建立自己的度量

## 内置度量

[内置度量](https://github.com/apache/incubator-storm/blob/46c3ba7/storm-core/src/clj/backtype/storm/daemon/builtin_metrics.clj)度量Strom本身。