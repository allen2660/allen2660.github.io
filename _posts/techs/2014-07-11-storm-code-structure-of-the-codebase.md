---
layout: post
title:  Storm源码-源码结构
---

本文为Storm官方文档[代码结构](http://storm.incubator.apache.org/documentation/Structure-of-the-codebase.html)的读书笔记


Storm源码有三个不同的层次。

首先，Storm一开始是被设计成兼容多语言的。Nimbus是一个Thrift服务、拓扑也被定义成Thrift结构体。Thrift的使用允许Storm兼容多语言。

其次，所有的Storm接口都是Java接口。所以即使在Storm的实现中有很多Clojure代码，使用Storm全部都是Java接口。

第三，Storm的实现大部分基于Clojure。从行数上看，Storm中一半是Java，一半是Clojure。但是Clojure表现力更强，所以实际上大部分代码都是Clojure。

接下来详细解释每个部分。

## storm.thrift

要想了解Storm的代码，首先要看的是[storm.thrift](https://github.com/apache/incubator-storm/blob/master/storm-core/src/storm.thrift)。

Storm使用Thrift的['storm分支'](https://github.com/nathanmarz/thrift/tree/storm)来生成代码。这个分支实际上是thrift 7，java package重命名为org.apache.thrift7。其他方面，它就是Thrift 7。之所以单独出这样一个Thrift版本一是考虑到Thrift缺少向后兼容，二是为了避免包名冲突以满足一些用户在他们自己的topologies中用到其他版本的thrift。

拓扑中的每个spout/bolt都会有一个用户指定的标识符“component id”。[StormTopology](https://github.com/apache/incubator-storm/blob/master/storm-core/src/storm.thrift#L91)包含了一个map用于component id到component的映射。

Spout和bolt有着相同的thrift定义。所以我们只是看下[bolt的thrift定义](https://github.com/apache/incubator-storm/blob/master/storm-core/src/storm.thrift#L79)。它包含一个`ComponentObject`和一个`ComponentCommon`结构体。

`ComponentObject`定义了bolt的实现。它可能是三个类型之一：

1. 序列化的Java对象(实现了[IBolt](https://github.com/apache/incubator-storm/blob/master/storm-core/src/jvm/backtype/storm/task/IBolt.java))
2. ShellComponent对象，标示了实现是其他语言。这样标识一个bolt会让Storm使用[ShellBolt](https://github.com/apache/incubator-storm/blob/master/storm-core/src/jvm/backtype/storm/task/ShellBolt.java)来处理JVM-based work进程和非JVM实现之间的通信。
3. JavaObject结构体，告诉Storm类型名和构造参数用于初始化bolt。这在你想用非jvm语言定义拓扑的时候有用。

`ComponentCommon`定义了component的其他信息，包含：

1. 提交到哪个流，每个流的元数据（是否direct stream，fields declaration）
2. 消费哪个流，(map component_id:stream_id->stream grouping)
3. 并行度
4. 本组件的[配置](https://github.com/apache/incubator-storm/wiki/Configuration)

注意到spout也有ComponentCommon字段，所以spout也可以有其他输入流的声明。但这个不是给用户指定的。Storm在拓扑中添加隐式流，从acker bolt到每个spout。这样acker发送的ack,fail消息就经过这些流发给spout。将用于拓扑转化成实时拓扑的代码在[这里](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/common.clj#L279)

## Java interfaces

Storm主要是Java接口。主要接口是：

1. [IRichBolt](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IRichBolt.html)
2. [IRichSpout](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/IRichSpout.html)
3. [TopologyBuilder](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/TopologyBuilder.html)

大部分接口的策略：

1. 使用Java接口声明
2. 提供默认实现基类。

可以参见[BaseRichSpout](http://storm.incubator.apache.org/apidocs/backtype/storm/topology/base/BaseRichSpout.html)了解这个策略。

Spout和bolt序列化成上面讨论的Thrift定义。

值得一提的一个细节是，IBolt、ISpout与IRichBolt、IRichSpout这两对接口是有区别的。它们主要区别是在"Rich"版本里增加了"declareOutputFields"方法。这样设计的原因是所有的输出stream的输出field声明都必须是在Thrift结构里的（这样就可以做到使用任何编程语言来声明了），但是用户又希望能够在自己的class中来声明stream输出field信息。为解决这个问题，"TopologyBuilder"在构造Thrift结构时就是通过调用"declareOutputFields"方法来得到输出field的声明，然后将其转换纳入Thrift结构。这个转换操作可以从"TopologyBuilder"代码中的[这一段](https://github.com/nathanmarz/storm/blob/master/storm-core/src/jvm/backtype/storm/topology/TopologyBuilder.java#L205)里看到。



## Implementation


应该说，Storm主要是由Clojure语言实现的。尽管从代码行数上看一半是Java一半是Clojure，但其实里面绝大多数的逻辑实现都是Clojure。有两个值得一提的例外就是[DRPC](https://github.com/apache/incubator-storm/wiki/Distributed-RPC)和[支持事务的topology](https://github.com/apache/incubator-storm/wiki/Transactional-topologies)，它们二者都纯Java实现的。这样做的主要目的是来展示如何基于Storm，实现Storm之上更高层次的抽象。DRPC和支持事务的topology的实现分别位于[backtype.storm.coordination](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/coordination)、[backtype.storm.drpc](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/drpc)和[backtype.storm.transactional](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/transactional)包里。

这里总结了一份主要的Java包和Clojure命名空间的内容列表：

### Java包

[backtype.storm.coordination](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/coordination): 实现了DRPC和事务性topology里用到的基于Storm的批处理功能。这个包里最重要的类是CoordinatedBolt

[backtype.storm.drpc](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/drpc): DRPC的更高层次抽象的具体实现

[backtype.storm.generated](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/generated): 自动生成的Thrift代码（利用[这里fork出来的Thrift版本](https://github.com/nathanmarz/thrift)生成的，主要是把org.apache.thrift包重命名成org.apache.thrift7来避免与其他Thrift版本的冲突）

[backtype.storm.grouping](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/grouping): 包含了用户实现自定义stream分组类时需要用到的接口

[backtype.storm.hooks](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/hooks): 定义了处理storm各种事件的钩子接口，例如当task发射tuple时、当tuple被ack时。关于钩子的手册详见[这里](https://github.com/apache/incubator-storm/wiki/Hooks)

[backtype.storm.serialization](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/serialization): storm序列化/反序列化tuple的实现。在[Kryo](http://code.google.com/p/kryo/)之上构建。

[backtype.storm.spout](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/task): spout及相关接口的定义（例如"SpoutOutputCollector"）。也包括了"ShellSpout"来实现非JVM语言定义spout的协议。

[backtype.storm.task](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/task): bolt及相关接口的定义（例如"OutputCollector"）。也包括了"ShellBolt"来实现非JVM语言定义bolt的协议。最后，"TopologyContext"也是在这里定义的，用来在运行时供spout和bolt使用以获取topology的执行信息。

[backtype.storm.testing](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/testing): 包括了storm单元测试中用到的各种测试bolt及工具。

[backtype.storm.topology](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/topology): 在Thrift结构之上的Java层，用以提供一个纯Java API来使用Storm（用户不需要了解Thrift的细节）。"TopologyBuilder"及不同spout和bolt的基类们也在这里定义。稍高一层次的接口"IBasicBolt"也在这里定义，它会使得创建某些特定类型的bolt会更加简洁。

[backtype.storm.transactional](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/transactional): 包括了事务性topology的实现。

[backtype.storm.tuple](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/tuple): 包括Storm中tuple数据模型的实现。

[backtype.storm.utils](https://github.com/apache/incubator-storm/tree/master/storm-core/src/jvm/backtype/storm/tuple): 包含了Storm源码中用到的数据结构及各种工具类。




### Clojure 命名空间


[backtype.storm.bootstrap](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/bootstrap.clj): 包括了1个有用的宏来引入源码中用到的所有类及命名空间。

[backtype.storm.clojure](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/clojure.clj): 包括了利用Clojure为Storm定义的特定领域语言(DSL)。

[backtype.storm.cluster](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/cluster.clj): Storm守护进程中用到的Zookeeper逻辑都封装在这个文件中。这部分代码提供了API来将整个集群的运行状态映射到Zookeeper的"文件系统"上（例如哪里运行着怎样的task，每个task运行的是哪个spout/bolt）。

[backtype.storm.command.*](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/command): 这些命名空间包括了各种"storm xxx"开头的客户端命令行的命令实现。这些实现都很简短。

[backtype.storm.config](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/config.clj): Clojure中config的读取/解析实现。同时也包括了工具函数来告诉nimbus、supervisor等守护进程在各种情况下应该使用哪些本地目录。例如："master-inbox"函数会返回本地目录告诉Nimbus应该将上传给它的jar包保存到哪里。

[backtype.storm.daemon.acker](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/acker.clj): "acker" bolt的实现。这是Storm确保数据被完全处理的关键组成部分。

[backtype.storm.daemon.common](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/common.clj): Storm守护进程用到的公共函数，例如根据topology的名字获取其id，将1个用户定义的topology映射到真正运行的topology（真正运行的topology是在用户定义的topology基础上添加了ack stream及acker bolt，参见system-topology!函数），同时包括了各种心跳及Storm中其他数据结构的定义。

[backtype.storm.daemon.drpc](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/drpc.clj): 包括了DRPC服务器的实现，用来与DRPC topology一起使用。

[backtype.storm.daemon.nimbus](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/nimbus.clj): 包括了Nimbus的实现。

[backtype.storm.daemon.supervisor](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/supervisor.clj): 包括了Supervisor的实现。

[backtype.storm.daemon.task](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/task.clj): 包括了spout或bolt的task实例实现。包括了处理消息路由、序列化、为UI提供的统计集合及spout、bolt执行动作的实现。

[backtype.storm.daemon.worker](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/daemon/worker.clj): 包括了worker进程（1个worker包含很多的task）的实现。包括了消息传输和task启动的实现。

[backtype.storm.event](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/event.clj): 包括了1个简单的异步函数的执行器。Nimbus和Supervisor很多场合都用到了异步函数执行器来避免资源竞争。

[backtype.storm.log](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/log.clj): 定义了用来输出log信息给log4j的函数。

[backtype.storm.messaging.*](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/messaging): 定义了1个高一层次的接口来实现点对点的消息通讯。工作在本地模式时Storm会使用内存中的Java队列来模拟消息传递。工作在集群模式时，消息传递使用的是ZeroMQ。通用的接口在protocol.clj中定义。

[backtype.storm.stats](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/stats.clj): 实现了向Zookeeper中写入UI使用的统计信息时如何进行汇总。实现了不同粒度的聚合。

[backtype.storm.testing](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/testing.clj): 包括了测试Storm topology的工具。包括时间仿真，运行一组固定数量的tuple然后获得输出快照的"complete-topology"，"tracker topology"可以在集群"空闲"时做更细粒度的控制操作，以及其他工具。

[backtype.storm.thrift](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/thrift.clj): 包括了自动生成的Thrift API的Clojure封装以使得使用Thrift结构更加便利。

[backtype.storm.timer](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/timer.clj): 实现了1个后台定时器来延迟执行函数或者定时轮询执行。Storm不能使用Java里的Timer类，因为为了单测Nimbus和Supervisor，必须要与时间仿真集成起来使用。

[backtype.storm.ui.*](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/ui): Storm UI的实现。完全独立于其他的代码，通过Nimbus的Thrift API来获取需要的数据。

[backtype.storm.util](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/util.clj): 包括了Storm代码中用到的通用工具函数。

[backtype.storm.zookeeper](https://github.com/apache/incubator-storm/blob/master/storm-core/src/clj/backtype/storm/zookeeper.clj): 包括了Clojure对Zookeeper API的封装，同时也提供了一些高一层次的操作例如："mkdirs"、"delete-recursive
"
