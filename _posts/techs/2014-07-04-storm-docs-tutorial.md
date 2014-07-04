---
layout: default
title:  storm-tutorial
---

# Storm Tutorial

本文为Storm官方文档[Tutorial](http://storm.incubator.apache.org/documentation/Tutorial.html)的读书笔记

## 前言

## Storm集群的组件

## 拓扑

## 流

Storm的核心抽象就是“流”。流是一个无限tuple序列。Stomr提供了分布式地、可靠地将一个流转化成另一个流的的原语。比如说，你可以讲一个tweets流转化成一个流行话题流。

Storm为流处理提供的基本原语是“spout”和“bolt”。spouts和bolts都提供了接口供你实现来运行应用逻辑。

spout是流的源头。举例来说，spout可以从[Kestrel queue](https://github.com/nathanmarz/storm-kestrel)读取元组，提交为一个流。或者spout也可以调用Twitter API从而提交tweets流。

bolt消费任意的输入流、做处理、需要的话输出下一个流。像“从tweets流中计算热门话题”这样的复杂流转化，需要经过多个bolt多步处理。bolt可以执行运行函数、过滤元组、流聚合、流join、连接数据库等等操作。

spouts和bolts组成的网络被打包成“拓扑”，拓扑是你提交给Storm集群执行的顶层抽象。拓扑就是一个流的转化图，该图的每个节点是一个spout或者bolt。图的边表示bolts订阅了一个流。一个spout/bolt提交的tuple会发送给所有订阅该流的bolt。

![topology](http://storm.incubator.apache.org/documentation/images/topology.png)



## 数据模型

## 一个简单的拓扑

## 以local模式运行Demo拓扑

## Stream grouping

## 使用其他语言定义Bolts

## 保证消息处理

## 事务拓扑

## 分布式RPC

## 结论

本教程给出了开发、测试、部署Strom拓扑的大体介绍。文档的其他部分会更深入的讲解Storm的各个方面。