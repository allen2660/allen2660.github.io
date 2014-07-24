---
layout: post
title:  storm-concepts
---

本文为Storm官方文档[Concepts](http://storm.incubator.apache.org/documentation/Concepts.html)的读书笔记


本页面列出了Storm的基本概念，和相关的链接。它们是：

1. 拓扑
2. 流
3. Spouts
4. Bolts
5. 流分组
6. 可靠性
7. 任务
8. Worker

# 拓扑

实时应用程序逻辑被打包成了Storm 拓扑。Storm 拓扑同MR任务类似。二者的一个关键区别就是MR job会跑完，而拓扑一直在运行。拓扑是一个由spouts和bolts组成的图，这些元素之间用流分组连接。

## 相关资源：

+ [TopologyBuilder](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/TopologyBuilder.html)：Java语言中用于构造拓扑。
+ [Running topologies on a production cluster](http://storm.incubator.apache.org/documentation/Running-topologies-on-a-production-cluster.html)
+ [Local mode](http://storm.incubator.apache.org/documentation/Local-mode.html)：阅读这篇来学习如何在本地模式下开发和测试拓扑。

# 流

流式Storm的核心抽象。流是一个分布式系统中，被并行处理和创建的无界元组序列。流由描述元组的域的名字组成的schema定义。默认的，元组可以包含int、long、short、bytes、string、double、float、boolean和字节数组。你也可以定义自己的序列化器来自定义元组域类型。

每个流在被定义的时候都会被赋予一个id。由于单流spouts/bolts很普遍，[OutputFieldsDeclarer](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html)有很方便的不需要指定id声明一个单流的方法。在这种情况中，流被给予了一个默认id“default”。

## 相关资源：

+ Tuple：流由元组构成
+ OutputFieldsDeclarer：用于声明流和schema
+ Serialization：Storm的动态元组类型以及声明自定义序列化器的信息。
+ ISerialization：自定义序列化器需要实现的接口。
+ CONFIG.TOPOLOGY_SERIALIZATIONS：自定义序列化器都已通过这个配置注册。

# Spouts

spout是拓扑中流的来源。通常来说，spouts会从外部数据源读元组，并将它们提交到拓扑中。（比如说Kestrel队列或者Twitter API）Spout可以使可靠地或者不可靠的。可靠地spout可以在处理失败时重放元组数据，而不可靠的spout提交了元组后就不管了。

spout可以提交不止一个流。为了达到这个目的，使用OutputFieldsDeclarer的declareStream方法声明多个流，并在提交的时候使用SpoutOutputCollector的emit方法指定特定的流。

spout的主要方法是`nextTuple`。nextTuple提交元组到拓扑中，如果没有新元组可提交就简单的返回。spout的实现中，nextTuple方法保证不会阻塞是很有比较的，因为Storm在同一线程中调用所有的spout方法。

spout的其他主要方法有`ack`和`fail`。Storm发现一个被spout提交的元组成功处理完成后，会调用ack，否则调用fail。ack和fail只有在可靠spout时才调用。参考[Javadoc](http://storm.incubator.apache.org/apidocs/backtype/storm/spout/ISpout.html)以获取更多信息。

## 相关资源：

+ [IRichSpout](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IRichSpout.html)：spout必须实现的接口
+ [Guaranteeing message processing](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)

# Bolts

拓扑中所有的处理都是在bolt中完成的。bolt可以做包括过滤、函数、聚合、join、与db交互等等事情。

Bolt可以做简单地流转换。复杂的流转换通常要求多步多个bolts。距离说来，转换一个tweet流为一个热门图流需要至少两步：一个bolt列出每个图片的rt次数，另外一个或几个bolt算出top X的图片。（使用三个而不是两个bolt可以让这个流转换更加可扩容）

Bolts可以提交不止一个流。为了达到这个目的，使用OutputFieldsDeclarer的declareStream方法声明多个流，并在提交的时候使用OutputCollector的emit方法指定特定的流。

当你声明了bolt的输入流，你通常是订阅另外一个组件的输出流。如果想订阅其他组建的所有流，需要每个单独订阅。InputDeclarer有给订阅默认流id的语法糖。declarer.shuffleGrouping("1") 订阅stream “1”的默认流，与 declarer.shuffleGrouping("1", DEFAULT_STREAM_ID)等价。

bolt的主要方式是execute，它以元组作为输入。bolt使用OutputCollector提交元组。bolt必须为它处理的每个元组调用ack方法，这样Storm在一个元组被完全处理后才会知道（并且最终ack输入的soput元组）。为了描述通常的处理流程：输入tuple，提交0个或多个元组，然后ack这个元组，Storm提供了[IBasicBolt](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IBasicBolt.html)接口来自动化完成上述步骤。

在bolt中启动新建成来做异步处理也是可以的，OutputCollector是线程安全的，可以被多次调用。

## 相关资源：

+ [IRichBolt](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IRichBolt.html)：bolt基本接口
+ [IBasicBolt](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IBasicBolt.html)：方面的接口，为了简单的bolt
+ [OutputCollector](http://storm.incubator.apache.org/apidocs/backtype/storm/task/OutputCollector.html)：bolt使用这个类来提交元组到流
+ [Guaranteeing message processing](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)

# 流分组

# 可靠性

# 任务

每个spout/bolt以很多tasks的形式在集群中执行。每个任务对应一个执行线程，而流分组定义了如何从一个任务组发送元组到另一个任务组。你可以通过TopologyBuilder的setSpout/setBolt方法来设置每个spout/bolt的并发数目（亦即task数目）。

# Worker

拓扑在一个或多个worker进程上运行。每个worker进程是一个物理的JVM，切执行一个拓扑的所有tasks的一个子集。举例说，拓扑的总体并发是300，分配了50个worker，那么每个worker会执行6个任务（以worker线程的方式）。Storm会尝试在所有worker之间平均文佩这些tasks。

## 相关资源：

+ Config.TOPOLOGY_WORKERS：这个配置设置为了执行一个拓扑分配的worker的数目。