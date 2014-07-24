---
layout: post
title:  运行的拓扑解析--worker，executor，task
---

本文为Storm官方文档[What makes a running topology](https://storm.incubator.apache.org/documentation/Understanding-the-parallelism-of-a-Storm-topology.html)的读书笔记

# What makes a running topology: worker processes, executors and tasks

为了在Storm集群中运行一个拓扑，Storm中有下面三个不同的实体：

1. Worker processes
2. Executors(threads)
3. Tasks

![](https://storm.incubator.apache.org/documentation/images/relationships-worker-processes-executors-tasks.png)


## Configuring the parallelism of a topology

### worker processes数目

### executors（threads）数目

### tasks数目


## Example of a running topology

![](https://storm.incubator.apache.org/documentation/images/example-of-a-running-topology.png)

## How to change the parallelism of a running topology

1. 使用Storm web UI来rebalance
2. 使用CLI来rebalance

## 参考

+ Concepts
+ Configuration
+ Running topologies on a production cluster]
+ Local mode
+ Tutorial
+ Storm API documentation, most notably the class Config
