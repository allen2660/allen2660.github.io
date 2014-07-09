---
layout: default
title:  分布式 rpc
---

本文为Storm官方文档[Distributed RPC](http://storm.incubator.apache.org/documentation/Distributed-RPC.html)的读书笔记

分布式RPC的概念是将耗时的查询在Storm上并行处理。Storm拓扑将函数参数作为输入流，然后将每个函数调用的结果提交给客户端。

DRPC 不算是Storm的功能。因为它更多的是一个基于Storm原语（流、spout、bolt、topology）的使用场景。DRPC本可以以独立库的方式发行，但由于它太有用了所以还是和Storm一起发行。

# 高层总览

DRPC被一个叫做“DRPC Server”的服务协调（Storm自带这个server的实现）。DRPC server负责接收一个rpc请求，发送请求到Storm拓扑，接收拓扑的结果，发送回给客户端。从一个客户端的角度来看，DRPC调用看起来就像一个普通的RPC调用。举例来说，下面是一个客户端调用“reach”函数的例子，其中参数是“http://twitter.com”:

    DRPCClient client = new DRPCClient("111.222.333.444", 3772); 
    String result = client.execute("reach", "http://twitter.com");

DRPC的工作流看起来就像下面这样：

![](http://storm.incubator.apache.org/documentation/images/drpc-workflow.png)

客户端向DRPC服务器发送函数的名字和函数的参数。拓扑使用DRPCSpout来实现来自DRPC server的函数调用流。然后拓扑计算结果，最后有一个叫`ReturnResults`的bolt连接到DRPC server，发送对应函数调用id的结果。DRPC server查找对应id的client，解除阻塞，发送结果。

# LinearDRPCTopologyBuilder

Storm提供了一个叫做[LinearDRPCTopologyBuilder](http://storm.incubator.apache.org/apidocs/backtype/storm/drpc/LinearDRPCTopologyBuilder.html)的类可以让DRPC的大部分工作自动化。其包括：

1. 设置spout
2. 返回DRPC server结果
3. 提供给bolt做有限的元组的聚合的功能。

让我们看一个简单的例子。这里是一个DRPC拓扑的实现，它在输入后面加上“!”后缀：

    public static class ExclaimBolt extends BaseBasicBolt { 
        public void execute(Tuple tuple, BasicOutputCollector collector) { 
            String input = tuple.getString(1); 
            collector.emit(new Values(tuple.getValue(0), input + “!”)); 
        }

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("id", "result"));
        }   
    }


    public static void main(String[] args) throws Exception { 
        LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder(“exclamation”); 
        builder.addBolt(new ExclaimBolt(), 3); 
        // … 
    }

你可以看到，只需要做很少的事情。当创建`LinearDRPCTopologyBuilder`的时候，你需要告诉它DRPC函数的名字。一个DRPC server可以负责很多函数，函数名字用于区别多个函数。你声明的第一个bolt输入二元元组做参数，第一个域是请求id，第二个是请求的参数。`LinearDRPCTopologyBuilder`期望最后提交的bolt提交一个二元元组流，形如["id","result"]。最后，所有的中间爱你元组都在第一个域中包含请求id。

在这个例子中，`ExclaimBolt`简单地在元组的第二个域后面附加了“!”。`LinearDRPCTopologyBuilder`负责其他的工作，比如连接DRPC server以及发送结果给client。

# 本地模式 DRPC

DRPC可以再本地模式运行，上面的例子以本地模式运行如下：

    LocalDRPC drpc = new LocalDRPC(); 
    LocalCluster cluster = new LocalCluster();
    cluster.submitTopology(“drpc-demo”, conf, builder.createLocalTopology(drpc));
    System.out.println(“Results for ‘hello’:” + drpc.execute(“exclamation”, “hello”));
    cluster.shutdown(); 
    drpc.shutdown();

# 远程模式 DRPC

在真实集群上运行DRPC的三步骤：

1. 启动DRPC server(s)  bin/storm drpc
2. 在Nimbus、Supervisor上配置DRPC服务器 yaml drpc.servers
3. 提交DRPC拓扑到Storm上 

第三步提交DRPC拓扑的过程和提交普通拓扑很像：

    StormSubmitter.submitTopology("exclamation-drpc", conf, builder.createRemoteTopology());


# 一个更复杂的例子

上面的Exclamation DRPC只是一个玩具。下面是一个复杂的例子。在Trident Tutorial中也有过Trident版本的实现，“reach” RPC。

reach topology在storm-starter中有定义：

    LinearDRPCTopologyBuilder builder = new LinearDRPCTopologyBuilder("reach"); 
    builder.addBolt(new GetTweeters(), 3); 
    builder.addBolt(new GetFollowers(), 12) 
        .shuffleGrouping(); 
    builder.addBolt(new PartialUniquer(), 6) 
        .fieldsGrouping(new Fields("id", "follower")); 
    builder.addBolt(new CountAggregator(), 2) 
        .fieldsGrouping(new Fields("id"));

这个拓扑有四个执行步骤：

1. GetTwwwters 拿到发送url的人。[id,url] -> [id,tweeter]
2. GetFollowers。[id,tweeter] -> [id,follower]
3. PartialUniquer。[id,follower] -> [id, count]
4. CountAggregator。[id,count] -> [id, count]

让我们看看`PartialUniquer` bolt：

    public class PartialUniquer extends BaseBatchBolt { 
        BatchOutputCollector _collector; 
        Object _id; 
        Set _followers = new HashSet();

        @Override
        public void prepare(Map conf, TopologyContext context, BatchOutputCollector collector, Object id) {
            _collector = collector;
            _id = id;
        }       

        @Override
        public void execute(Tuple tuple) {
            _followers.add(tuple.getString(1));
        }       

        @Override
        public void finishBatch() {
            _collector.emit(new Values(_id, _followers.size()));
        }       

        @Override
        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("id", "partial-count"));
        } 
    }    

PartialUniquer继承BaseBatchBolt。batch bolt提供一级API批量处理一批元组，每个请求id，都会创建一个batch bolt实例。Storm会负责清理这些实例。

PartialUniquer在execute方法中接收到follower元组的时候，它将其加到内部HashSet中。

Batch bolt提供了finishBatch方法，在这个batch所有的元组都被接受后调用。在这个回调中，PartialUniquer提交了这个id对应的所有followers的数目。

在底层，`CoordinatedBolt`被用于检测一个bolt是否收到了一个请求id对应的所有元组。`CoordinatedBolt`使用'direct stream'来实现这种协调。（direct stream使用direct grouping）

拓扑的其他部分都是直接是的。你可以看见，reach 计算的每个步骤都是并行计算的，定义DRPC拓扑极其简单。

# 非线性 DRPC 拓扑

`LinearDRPCTopologyBuilde`只能处理线性的DRPC拓扑，这种拓扑中计算是一序列步骤。不难相信会有函数需要bolt的分叉以及合并。目前你需要自己使用CoordinateBolt在底层实现。可以等待更多的DRPC拓扑抽象被支持。

# LinearDRPCTopologyBuilder 如何工作

+ DRPCSpout提交[args, return-info]。return-info是DRPC主机和端口加id。
+ 构造包含以下元素的拓扑：
    + DRPCSpout
    + PrepareRequest（生成请求id，创建 return-info以及args的流）
    + CoordinatedBolt wrappers and direct groupings
    + JoinResult
    + ReturnResult
+ LinearDRPCTopologyBuilder是基于Storm原语的高级抽象的一个很好的例子


# 高级

+ `KeyedFairBolt`：在同一时刻多个请求的处理编织在一起。
+ 如何直接使用`CoordinatedBolt`。

# 补充 VS Trident DRPC

Trident topology的API更简单：

    topology.newDRPCStream("reach")....