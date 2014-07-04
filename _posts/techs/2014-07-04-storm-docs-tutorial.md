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

你的拓扑中节点之间的连接描述了元组发送的方向。举例来说，如果Spout A -> BoltB，Spout -> Bolt C，Bolt B -> Bolt C，每次Spout A提交一个元组，该元组会被发送到Bolt B和Bolt C，同时Bolt B的输出也会发送给Bolt C。

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

该拓扑包含一个spout、两个bolt。spout输出words，每个bolt在输入后面附加“!!!”。节点是线性排列的：spout提交给第一个bolt，第一个bolt提交给第二个bolt。如果spout输出两个touple ["bob"]和["john"]，第二个bolt就会输出["bob!!!"]和["john!!!"]。

上面的代码使用setSpout和setBolt方法来定义节点。这两个方法接受一下参数：用户定义id、包含处理逻辑的对象、节点的并行数目。在上面的例子中，spout的id是“words”，bolt的id分别为“exclaim1”和“exclaim2”。

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

## 以local模式运行Demo拓扑

## Stream grouping

## 使用其他语言定义Bolts

## 保证消息处理

## 事务拓扑

## 分布式RPC

## 结论

本教程给出了开发、测试、部署Strom拓扑的大体介绍。文档的其他部分会更深入的讲解Storm的各个方面。