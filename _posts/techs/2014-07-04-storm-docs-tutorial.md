---
layout: post
title:  storm-tutorial
---


本文为Storm官方文档[Tutorial](http://storm.incubator.apache.org/documentation/Tutorial.html)的读书笔记

在本教程中，你可以学习到如何创建Storm拓扑以及部署到Storm集群上。Java是主要使用的语言，有些例子使用Python来展示Storm的多语言能力。

## 前言

本教程使用的例子来自于[storm-start](http://github.com/nathanmarz/storm-starter)项目。推荐你clone这个项目，并运行里面的例子。查看[Setting up a development environment](http://storm.incubator.apache.org/documentation/Setting-up-development-environment.html)和[Creating a new Storm project](http://storm.incubator.apache.org/documentation/Creating-a-new-Storm-project.html)来设置你的机器。

## Storm集群的组件

Storm集群很像Hadoop集群。在Hadoop上你运行“MapReduce 任务”，在Storm上你运行“拓扑”。“任务”和“拓扑”本身是不一样的。一个主要区别是M-R任务最终会完成，而拓扑一直运行。（除非被杀掉）

在Storm集群中有两种节点：master和worker。master节点运行“Nimbus”daemon，类似于Hadoop的JobTracker，Nimbus负责集群上的代码分发、非配任务给机器、监控失败情况。

每个worker节点运行有一个“Supervisor”daemon。Supervisor监控分配给该机器的工作，按照Nimbus的分配启停worker进程。每个worker进程执行拓扑的子集，一个运行的拓扑由分布在很多机器上的work进程组成。

![](http://storm.incubator.apache.org/documentation/images/storm-cluster.png)

Nimbus和Supervisor之间的协调工作都是基于[Zookeeper](http://zookeeper.apache.org/)完成。另外，Numbus daemon和Supervisor daemons 是fail-fast并且无状态的，所有的状态都寄存在Zookeeper或者本地磁盘上。这意味着你可以kill -9 Nimbus或者Supervisor，他们会自动重启，就像什么都没有发生一样。这个设计使得Storm集群异常的稳定。

## 拓扑

想要在Storm上做实时计算，你需要创建一个叫“拓扑”的东西。拓扑是一个计算的图，其中每一个几点包含一个处理逻辑，节点间的链接代表着数据的流动。

运行一个拓扑是简单直接的。首先打包代码和依赖进一个jar包，其次运行如下的一个命令：

    storm jar all-my-code.jar backtype.storm.MyTopology arg1 arg2

这个命令会启动 backtype.storm.MyTopology类，参数为arg1和arg2。该类的主函数定义了拓扑，并且提交这个拓扑到Nimbus上去。`storm jar`负责连接Nimbus、上传jar包。

由于拓扑的定义是一个Thrift结构体，Nimbus其实是一个Thrift服务器，你可以使用任意语言创建、提交拓扑。上面的例子是JVM-based language的最简单做法。关于开始和停止一个拓扑，查看[Running topologies on a production cluster](http://storm.incubator.apache.org/documentation/Running-topologies-on-a-production-cluster.html)获得更多信息。

## 流

Storm的核心抽象就是“流”。流是一个无限tuple序列。Stomr提供了分布式地、可靠地将一个流转化成另一个流的的原语。比如说，你可以讲一个tweets流转化成一个流行话题流。

Storm为流处理提供的基本原语是“spout”和“bolt”。spouts和bolts都提供了接口供你实现来运行应用逻辑。

spout是流的源头。举例来说，spout可以从[Kestrel queue](https://github.com/nathanmarz/storm-kestrel)读取元组，提交为一个流。或者spout也可以调用Twitter API从而提交tweets流。

bolt消费任意的输入流、做处理、需要的话输出下一个流。像“从tweets流中计算热门话题”这样的复杂流转化，需要经过多个bolt多步处理。bolt可以执行运行函数、过滤元组、流聚合、流join、连接数据库等等操作。

spouts和bolts组成的网络被打包成“拓扑”，拓扑是你提交给Storm集群执行的顶层抽象。拓扑就是一个流的转化图，该图的每个节点是一个spout或者bolt。图的边表示bolts订阅了一个流。一个spout/bolt提交的tuple会发送给所有订阅该流的bolt。

![topology](http://storm.incubator.apache.org/documentation/images/topology.png)

你的拓扑中节点之间的连接描述了元组发送的方向。举例来说，如果Spout A -> BoltB，Spout A -> Bolt C，Bolt B -> Bolt C，每次Spout A提交一个元组，该元组会被发送到Bolt B和Bolt C，同时Bolt B的输出也会发送给Bolt C。

Storm 拓扑中的每个节点都是并行执行的。在你的拓扑中，可以指定每个节点的并行度，Storm会根据这个并行数目在集群中分配线程来执行。

拓扑是永远运行的，除非被杀死。Storm会自动的重新分配失败的任务。并且Storm保证没有数据丢失，即使机器挂掉、信息丢包。


## 数据模型

Storm使用元组(tuple)作为数据模型。元组是一个命名的value列表，其中的一个field可以是任意类型或者对象。创造性地，Storm元组的field值支持所有的基础类型、字符串、字符数组。想要使用其他类型的对象，你只需要实现一个该类型的[序列化器](http://storm.incubator.apache.org/documentation/Serialization.html)。

拓扑中的每个节点必须声明它提交的元组的fields。举例来说，下面的bolt提交了二元的元组，fields为"double"和"triple"。

    public class DoubleAndTripleBolt extends BaseRichBolt { 
        private OutputCollectorBase _collector;

        @Override
        public void prepare(Map conf, TopologyContext context, OutputCollectorBase collector) {
            _collector = collector;
        }       

        @Override
        public void execute(Tuple input) {
            int val = input.getInteger(0);        
            _collector.emit(input, new Values(val*2, val*3));
            _collector.ack(input);
        }       

        @Override
        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("double", "triple"));
        }     
    }

declareOutputFields这个函数声明了输出字段为["double","triple"]，关于bolt的其他代码接下来的章节解释。

## 一个简单的拓扑

让我们通过一个简单的拓扑来深入了解一下其概念和代码。storm-start中的ExclamationTopology：

    TopologyBuilder builder = new TopologyBuilder(); 
    builder.setSpout("words", new TestWordSpout(), 10); 
    builder.setBolt("exclaim1", new ExclamationBolt(), 3) .shuffleGrouping("words"); 
    builder.setBolt("exclaim2", new ExclamationBolt(), 2) .shuffleGrouping("exclaim1");

该拓扑包含一个spout、两个bolt。spout输出words，每个bolt在输入后面附加“!!!”。节点是线性排列的：spout提交给第一个bolt，第一个bolt提交给第二个bolt。如果spout输出两个touple ["bob"]和["john"]，第二个bolt就会输出["bob!!!!!!"]和["john!!!!!!"]。

上面的代码使用setSpout和setBolt方法来定义节点。这两个方法接受以下参数：用户定义id、包含处理逻辑的对象、节点的并行数目。在上面的例子中，spout的id是“words”，bolt的id分别为“exclaim1”和“exclaim2”。

其中，第二个参数对象通过实现IRichSpout接口或者IRichBolt接口来包含处理逻辑。

最后一个参数用于控制节点的并发数目，是可选参数。它指定了执行这个组件的并发线程数。如果忽略，Storm默认启一个。

setBolt返回InputDeclarer对象，用于定义Bolt的输入。在这个例子中，“exclaim1”组件声明了其会读取所有“words”输出的元组，通过shuffle 分组的形式，“exclaim2”也使用shuffle的方式读取所有“exclaim1”输出的元组。“shuffle 分组”表示元组会被随机从上游分发到bolt任务。有很多方式来决定组件间数据流的分组方式，下面会详细解释。

如果你想让“exclaim2”读来自“words”和“exclaim1”二者的输出，应该这样写“exclaim2”的定义：

    builder.setBolt("exclaim2", new ExclamationBolt(), 5).shuffleGrouping("words") .shuffleGrouping("exclaim1");

可以看到，可以通过链式连接多个输入源来声明多个输入源。

让我们继续深入看看这个拓扑中spout和bolt的实现。Spouts负责向拓扑中提交消息。该拓扑中的TestWordSpout每100ms从列表["nathan","mike","jackson","golda","bertels"]中随机提交一些一元元组。TestWordSpout中的nextTuple()方法实现如下：

    public void nextTuple() { 
        Utils.sleep(100); 
        final String[] words = new String[] {"nathan", "mike", "jackson", "golda", "bertels"}; 
        final Random rand = new Random(); 
        final String word = words[rand.nextInt(words.length)]; 
        _collector.emit(new Values(word)); 
    }    

可以看到，实现很简单直接。

ExclamationBolt 在输入字符串后附加“!!!”。下面是ExclamationBolt的全部实现：

    public static class ExclamationBolt implements IRichBolt { 
        OutputCollector _collector;

        public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
            _collector = collector;
        }       

        public void execute(Tuple tuple) {
            _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
            _collector.ack(tuple);
        }       

        public void cleanup() {
        }       

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }       

        public Map getComponentConfiguration() {
            return null;
        }
    }

prepare方法提供给bolt一个OutputCollector用于输出元组。元组可以在bolt中处理的任何时候被提交：prepare、execute、cleanup、甚至在异步的另一个线程。上述Demo中的prepare实现简单地保存了OutputCollector以在后面execute中使用。

execute方法接受一个bolt输入的元组。ExclamationBolt提取元组的第一个字段，在后面附加“!!!”。如果你实现一个订阅多个不同源的bolt，可以通过Tuple#getSourceComponent方法获得输入数据来自哪个组件。

execute方法中还有其他一些值得说的，比如说输入tuple是emit方法的第一个参数并且输入tuple被ack了。这些是Storm的可用性API，用以保证没有数据丢失。

cleanup方法在Bolt退出的时候被调用，用来关闭一些打开的资源。不能保证该方法在集群中被调用。比如说某个任务运行所在的机器挂掉了，就没有办法调用该方法。cleanup方法被用[local模式](http://storm.incubator.apache.org/documentation/Local-mode.html)运行拓扑，你想在不泄露资源的情况下启停很多拓扑的时候。

declareOutputFields方法声明ExclamationBolt提交一元元组，field叫"word"。

getComponentConfiguration方法允许你配置该组件运行的很多方面属性。该主题会在[Configuration](http://storm.incubator.apache.org/documentation/Configuration.html)详细解释。

cleanup、getComponentConfiguration这样的方法经常不需要被bolt实现。你可以通过继承一个基本实现来更简洁的实现一个Bolt。ExclamationBolt可以通过继承BaseRichBolt来更简洁的实现：
    
    public static class ExclamationBolt extends BaseRichBolt { 
        OutputCollector _collector;

        public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
            _collector = collector;
        }       

        public void execute(Tuple tuple) {
            _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
            _collector.ack(tuple);
        }       

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }    
    }

## 以local模式运行Excalmation拓扑

Storm有两个运行模式：本地模式和分布式模式。在本地模式中，Storm以一个进程的形式工作，使用线程模拟工作节点。本地模式在测试、开发拓扑的时候很有用。当你跑storm-starter中的拓扑时，它们会以本地模式运行,且你可以看到每个组件提交的消息。在[Local Mode](http://storm.incubator.apache.org/documentation/Local-mode.html)可以读到更多关于在本地模式运行拓扑的信息。

在分布式模式中，Storm以集群的方式运行。提交一个拓扑到master的同时，你也提交了运行拓扑必须的所有代码。 master会负责分发代码、分配worker运行拓扑。如果worker挂掉，master会重新分配给其他worker。你可以参考[Running topologies on a production cluster](http://storm.incubator.apache.org/documentation/Running-topologies-on-a-production-cluster.html)获得更多在集群上运行拓扑的信息。

下面是本地模式运行ExclamationTopology的代码：

    Config conf = new Config(); 
    conf.setDebug(true); 
    conf.setNumWorkers(2);
    LocalCluster cluster = new LocalCluster(); 
    cluster.submitTopology(“test”, conf, builder.createTopology()); 
    Utils.sleep(10000); 
    cluster.killTopology(“test”); 
    cluster.shutdown();

首先，代码使用LocalCluster对象来定义一个进程内集群。提交拓扑到这个本地集群和提交到分布式集群上是一样的。使用submitTopology方法来提交拓扑，使用以下几个参数：name、conf、Topology。

name用来指定一个topo方便杀掉。不杀的话，拓扑会一直运行下去。

配置是用来调节运行拓扑的很多方面的。下面两个配置属性很常见：

+ TOPOLOGY_WORKERS(setNumWorkers) 指定分配多少进程来承担worker的功能。拓扑的组件(spout,bolt)以线程的方式运行。组件的线程数目被setBolt和setSpout方法控制。这些组件的线程在worker进程中运行。举例来说，你可以指定50个工作进程，300个线程的组件。这样的话每个进程有6个线程，每个线程都可以属于不同的组件。你通过调节每个组件的并发度和worker进程的个数来调节Storm拓扑的性能。
+ TOPOLOGY_DEBUG(setDebug) 设为true时，告诉Storm记录每个组件提交的每个消息。这在本地模式测试拓扑的时候很有用，但是在集群中运行的时候最好关掉。

关于拓扑还有很多可配置的。详见[the Javadoc for Config](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html)。

关于设置开发环境、本地模式运行拓扑（eclipse）,参见[Creating a new Storm project](http://storm.incubator.apache.org/documentation/Creating-a-new-Storm-project.html)。

## Stream grouping

流分组告诉拓扑如何在两个组件之间发送元组。记住，每个spout和bolt是以task为单位在集群中并行执行的。如果在task层面看拓扑运行情况的话，应该如下图：

![streaming-grouping](http://storm.incubator.apache.org/documentation/images/topology-tasks.png)

当Bolt A的一个task向Bolt B提交元组的时候，应该发送给Bolt B的哪个task呢？

“流分组”通过告诉Storm如何在tasks集合之间发送元组解决了上面的问题。在我们深入探讨不同种类的流分组之前，让我们看下[storm-starter](http://github.com/nathanmarz/storm-starter)中的另一个拓扑。这个[WordCountTopology](https://github.com/nathanmarz/storm-starter/blob/master/src/jvm/storm/starter/WordCountTopology.java) 读取spout的语句，经过WordCountBolt计算出单词出现的次数：

    TopologyBuilder builder = new TopologyBuilder();
    builder.setSpout(“sentences”, new RandomSentenceSpout(), 5); 
    builder.setBolt(“split”, new SplitSentence(), 8) .shuffleGrouping(“sentences”); 
    builder.setBolt(“count”, new WordCount(), 12) .fieldsGrouping(“split”, new Fields(“word”));

SplitSentence 提交其收到的每一句话中的每一个单词，WordCount在内存中保存了word-count的map。每次WordCount收到一个单词，它更新它的状态、提交新的单词数目。

有几种不同的流分组。

最简单的分组叫做“shuffle grouping”，其将元组发送到随机的下游task。shuffle grouping在上面的拓扑中用于从RandomSentenceSpout发送到SplitSentence bolt。它的作用是随机的将处理元组的任务均匀的地发送出去。

更有趣的分组是“fileds grouping”。fileds grouping用于SplitSentence bolt发送给WordCount的时候。它对于WordCount bolt的正确性很关键：同样的单词总是被发送到同样的task。不然的话，一个单词会被不同的task接收，这些task会提交错误的结果因为他们分别拥有了一部分信息。fields grouping使得你可以通过元组的fields子集来分发，从而fields子集的值一样的元组发送到同一个task。WordCount使用fields grouping来订阅SplitSentence的输出，同样的词被发往同一个task，从而保证结果的正确。

fields grouping是实现流join和流聚合以及其他使用场景的基础。在底层，fields grouping使用取模哈希来实现。

还有其他的一些流分组，在[Concepts](http://storm.incubator.apache.org/documentation/Concepts.html)可以看到更多。


## 使用其他语言定义Bolts

Bolts可以使用任意语言定义。使用其他语言编写的bolt以子进程的方式运行，Storm使用stdI/O的JSON方式来实现进程间通信。通信协议只需要~100行的适配层库，Storm已经为Ruby、Python和Fancy（？Fancy是什么？）。

下面是SplitSentence bolt：

    public static class SplitSentence extends ShellBolt implements IRichBolt { 
        public SplitSentence() { 
            super(“python”, “splitsentence.py”); 
        }

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        } 
    }

SplitSentence继承ShellBolt，且通过第二个参数声明使用splitsentence.py来运行。下面是splitsentence.py的实现：

    import storm

    class SplitSentenceBolt(storm.BasicBolt): 
        def process(self, tup): 
            words = tup.values[0].split(“ “) 
            for word in words: 
                storm.emit([word])

    SplitSentenceBolt().run()

关于如何使用其他语言编写spouts和bolts，以及如何通过其他语言编写拓扑（不使用JVM），参见[Using non-JVM languages with Storm](http://storm.incubator.apache.org/documentation/Using-non-JVM-languages-with-Storm.html)。


## 保证消息处理

在本教程的早些时候，我们跳过了元组是如何被提交的相关讨论。这些方面是Storm的 可靠性API的一部分：Storm如何保证spout输出的每一个消息都被处理。参看[Guaranteeing message processing](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)获得更多信息以及作为Storm用户如何利用Storm的可靠性API。

## 事务拓扑

Storm保证每个消息都被拓扑至少处理一次。一个常见的问题是“如何利用Storm计数？不会多数么？”。Storm有一个特性叫做“事务拓扑”，可以让大叔分计算都能得到“精确处理一次语义”。查看[这里](http://storm.incubator.apache.org/documentation/Transactional-topologies.html)了解更多。

## 分布式RPC

本教程展示了如何利用Storm做基本的流处理。使用Storm原语还可以做很多其他事情。Storm最有趣的应用之一就是分布式RPC，可以让你并行化处理远程紧张的函数调用。查看[这里](http://storm.incubator.apache.org/documentation/Distributed-RPC.html)了解更多。

## 结论

本教程给出了开发、测试、部署Strom拓扑的大体介绍。文档的其他部分会更深入的讲解Storm的各个方面。