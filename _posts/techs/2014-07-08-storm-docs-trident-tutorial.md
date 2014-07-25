---
layout: post
title:  trident tutorial
---

本文为Storm官方文档[TridentTutorial](http://storm.incubator.apache.org/documentation/Trident-tutorial.html)的读书笔记



Trident是一个基于Strom做实时计算的高度抽象。它让你可以无缝的将高吞吐有状态的流处理和低延迟的分布式查询结合起来。如果你熟悉高级批处理工具像Pig或者Cascading，Trident的概念是类似的。Trident有join、聚合、grouping、函数、过滤器等功能。除此以外，Trident还添加了基于db/永久存储做有状态的、增量处理的原语。Trident有持久的、精确一次语义，所以很容易开发Trident拓扑。

## 展示例子

该例子做两件事情：

1. 从一个输入句子流中计算流式word count
2. 实现得到一组word的数目的查询。

下面是输入源：

    FixedBatchSpout spout = new FixedBatchSpout(
        new Fields("sentence"), 
        3, 
        new Values("the cow jumped over the moon"), 
        new Values("the man went to the store and bought some candy"), 
        new Values("four score and seven years ago"), 
        new Values("how many apples can you eat")
    ); 
    spout.setCycle(true);

    TridentTopology topology = new TridentTopology(); 
    TridentState wordCounts = topology.newStream("spout1", spout).each(
            new Fields("sentence"), 
            new Split(), 
            new Fields("word")
        ).groupBy(new Fields("word")).persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count")).parallelismHint(6);

接下来的拓扑部分实现了一个低延迟的分布式查询。查询的输入是空格分隔的words，输入这些words的中数目。这些查询就像普通的RPC一样执行，除了他们在后台是分布式的。下面是调用query的例子：

    DRPCClient client = new DRPCClient("drpc.server.location", 3772); 
    System.out.println(client.execute("words", "cat dog the man"); // prints the JSON-encoded result, e.g.: "[[5078]]"

可以看到，上面的程序和普通RPC没有区别，除了他们在后台是在Storm集群中分布式执行的。这样规模的小查询延迟大概在10ms。更复杂的DRPC耗时可能更长些，不过延迟主要取决于分配的资源。

上面拓扑的分布式查询部分的实现如下：

    topology.newDRPCStream("words")
                .each(new Fields("args"), new Split(), new Fields("word"))
                .groupBy(new Fields("word"))
                .stateQuery(wordCounts, new Fields("word"), new MapGet(), new Fields("count")) //查询第一部分生成的word count
                .each(new Fields("count"), new FilterNull())
                .aggregate(new Fields("count"), new Sum(), new Fields("sum"));

使用同样的TridentTopology对象来创建DRPC流，且函数名称叫“words”。这个名称和DRPCClient调用的第一个参数对应。

每个DRPC调用被当做一个小的批处理job来对待，这个job将请求的内容当做一个单元组。该元组包含一个域叫“args”，值就是client提供的参数。上面的例子中，参数就是空格分隔的一组单词。

首先，使用Split将参数分割成连续的单词。这个流使用“word”分组，然后使用stateQuery查询第一部分生成的TridentState对象。stateQuery将一种state-本例中是拓扑的其他部分计算生成的词频-作为输入，然后查询这个state。本例中，MapGet方法被调用，用以拿到每个单词的词频。由于DPRC流和TridentState一样都是按照“word”分组，每个单词的查询被分发到对应的负责更新该word词频工作TridentState分区。

然后，没有词频的单词背FilterNull过滤器过滤掉，再通过Sum 聚合器得到总数。然后Trident自动将结果发送到等待的客户端。

Trident会智能的执行一个拓扑以获得最大化性能。在上面这个拓扑中有两个有趣的事情：

1. 对于state的读取或写入自动的以批处理的方式操作。这样如果对当前的批处理，有20个更新db的操作，Trident会自动的做一次读或写。（并且在很多场景中，你可以在State视线中使用cache来减少读请求）。所以你得到了两方面的便利-针对每个元组表达计算-以及性能。
2. Trident聚合得到了很重的优化。相比于将一组元组发送到同一个机器再聚合，Trident会尽可能的先做部分聚合然后再传出。举例说，Count 聚合器在每个parttiion计算count，发送分组数据到机器上，进一步汇总。这个技术和MapReduce中的combiner类似。

让我们再看看Trident的另一个例子。

## Reach

Reach是Twitter中一个URL的曝光度。

这个拓扑汇总两个state源中读取数据，一个db提供URL到tweet它的人列表的映射。另一个db提供一个列到他的followers的映射。拓扑定义如下：

    TridentState urlToTweeters = topology.newStaticState(getUrlToTweetersState()); 
    TridentState tweetersToFollowers = topology.newStaticState(getTweeterToFollowersState());

    topology
        .newDRPCStream(“reach”)
        .stateQuery(urlToTweeters, new Fields(“args”), new MapGet(), new Fields(“tweeters”)) 
        .each(new Fields(“tweeters”), new ExpandList(), new Fields(“tweeter”)) 
        .shuffle() 
        .stateQuery(tweetersToFollowers, new Fields(“tweeter”), new MapGet(), new Fields(“followers”)) 
        .parallelismHint(200) 
        .each(new Fields(“followers”), new ExpandList(), new Fields(“follower”)) 
        .groupBy(new Fields(“follower”)) 
        .aggregate(new One(), new Fields(“one”)) 
        .parallelismHint(20) 
        .aggregate(new Count(), new Fields(“reach”)); 

这个拓扑使用newStaticState方法创建了TridentState对象来代表每个外部db。这些对象可以在拓扑中被查询。像所有state源一样，到这些db的查询会被自动的批处理以获得最大化效率。

拓扑的定义是直接的-它就是一个简单的批处理任务。首先，查询urlToTweeters db获得tweet这个URL的所有人，返回一个人列表，然后使用ExpandList函数为每个tweeter创建一个元组。

然后，每个tweeter的follower需要被渠道。这一步的并行化很重要，所以使用了shuffle来平均的分配到每个worker。然后，针对每个tweeter查询db获取followers。你可以看到拓扑的这个部分并发设的很高（200），因为这一部是最耗紫原敦额计算。

接下来，followers去重计数。分为两步来做。一，针对“follower”做group by，每个group运行“One”聚合。“One”聚合简单的提交一个包含1的元组。二，这些“1”被加到一起得到去重的followers总数。下面是“One”聚合的定义：

     public class One implements CombinerAggregator { 
        public Integer init(TridentTuple tuple) { return 1; }

        public Integer combine(Integer val1, Integer val2) { return 1; }

        public Integer zero() { return 1; } 
    }

这是一个“combiner aggregator”，知道如何做部分聚合再传输。Sum同样是一个“combiner aggregator”，所以拓扑最后的全局汇总会非常快。

下面让我们更详细的看看Trident。

## Fields and tuples

## State

Trident做了两件事来实现“each message only processed only once”：

1. 每个batch有一个“transaction id”，如果一个batch重试会有一样的trasaction id。
2. State的batches之间的更新是有序的。也就是说，batch2的更新不完成，是不会做batch3的。

## Trident 拓扑的执行

Trident的拓扑会编译成有效地Storm拓扑。元组只有在重新分片数据的时候才会在网络上发送，比如说groupBy或者Shuffle，所以下面的Trident Topology：

![](http://storm.incubator.apache.org/documentation/images/trident-to-storm1.png)

会被编译成下面的Storm spout/bolt

![](http://storm.incubator.apache.org/documentation/images/trident-to-storm2.png)

## 结论

Trident让实时计算变得优雅。你已经看到，通过Trident API，如果使高吞吐流处理、状态操作、低延迟查询无缝结合起来。Trident让你可以在保证最大性能的基础上更自然的表达实时计算。