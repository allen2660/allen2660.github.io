---
layout: default
title:  Storm如何保证消息不丢失
---

本文为Storm官方文档[GuaranteeMessageProcessing](http://storm.incubator.apache.org/documentation/Guaranteeing-message-processing.html)的读书笔记

Storm保证从spout发出的每个tuple都会被完整处理。这篇文章描述了Storm如何做到的以及作为用户你如何利用Storm可靠性特点。

## 一个消息被“完整处理”是什么意思？

从一个spout发出的tuple可能会引起成千上万的tuple被产生。举例来说，看下面的word count 拓扑：

    TopologyBuilder builder = new TopologyBuilder(); 
    builder.setSpout("sentences", new KestrelSpout("kestrel.backtype.com", 22133, "sentence_queue", new StringScheme())); builder.setBolt("split", new SplitSentence(), 10) 
        .shuffleGrouping("sentences"); 
    builder.setBolt("count", new WordCount(), 20) 
        .fieldsGrouping("split", new Fields("word"));

该拓扑从Kestrel队列中读一句话，将这句话分割成单词，最后返回每个单词已经出现的次数。从spout发出的一个tuple出发了基于这个元组的无数元组。消息的结构如下：

![](http://storm.incubator.apache.org/documentation/images/tuple_tree.png)

上图中，一个tuple树被遍历到，且树上的每个消息都被处理后，Storm才认为一个tuple被“完整处理”了。在一定时间内某个tuple还没有被完整处理，这个tuple就会被认为失败了。这个时间可以用[Config.TOPOLOGY_MESSAGE_TIMEOUT_SECS](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html#TOPOLOGY_MESSAGE_TIMEOUT_SECS)来设定，默认是30秒。

## 消息处理成功或失败了会发生什么？

下面是spout要实现的接口：

    public interface ISpout extends Serializable { 
        void open(Map conf, TopologyContext context, SpoutOutputCollector collector); 
        void close(); 
        void nextTuple(); 
        void ack(Object msgId); 
        void fail(Object msgId); 
    }

首先，Storm调用storm的nextTuple方法，spout使用open方法中提供的SpoutOutputCollector来提交tuple。当提交tuple时，Spout会提供一个“message id”，用于后面指定这个tuple。举例，KestrelSpout从kestrel队列中读一个消息，将kestrel提供的描述消息的messageid提交了：
    
    _collector.emit(new Values("field1", "field2", 3) , msgId);

接下来，元组被发送到下游bolt，Storm负责跟踪消息树。如果Storm发现元组被完全处理了，会调用原始Spout的ack方法，参数为那个messageid。同样的，faild的话也会调用对应的方法。注意到，一个元组会被创建它的spout task ack/fail。所以如果spout以多个实例的方式在集群中运行，元组不是被不同的task ack/fail的。

我们再以KestrelSpout为例来看看spout需要做些什么才能保证“一个消息始终被完全处理”, 当KestrelSpout从Kestrel里面读出一条消息， 首先它“打开”这条消息， 这意味着这条消息还在kestrel队列里面， 不过这条消息会被标示成“处理中”直到ack或者fail被调用。处于“处理中“状态的消息不会被发给其他消息处理者了；并且如果这个spout“断线”了， 那么所有处于“处理中”状态的消息会被重新标示成“等待处理”。


## Storm的可靠性API

作为Storm的使用者，有两件事情可以用来利用storm的可用性。首先，在tuple tree中创建新节点的时候要告诉Storm。其次，处理完一个tuple的时候要告诉storm。做了这两件事，Storm可以再tuple tree完全处理后检测到，且可以恰当的ack/fail一个spout发出的tuple。Storm的API提供了做这两件事的简明的api。

在tuple treee中指定一个连接关系叫做“anchoring”。anchoring在你提交一个tuple的时候做。使用下面的bolt做例子：

    public class SplitSentence extends BaseRichBolt { OutputCollector _collector;

        public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
            _collector = collector;
        }

        public void execute(Tuple tuple) {
            String sentence = tuple.getString(0);
            for(String word: sentence.split(" ")) {
                _collector.emit(tuple, new Values(word));
            }
            _collector.ack(tuple);
        }

        public void declareOutputFields(OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }        
    }

每个新tuple都被anchored到输入tuple上，通过emit方法指定的两个参数。因为新产生的tuple被anchored了，tuple tree根节点上的spout tuple会被重放，如果这个新产生的tuple处理失败的话。相反，让我们看看如果像 下面这样提交会怎么样：

    _collector.emit(new Values(word));

这样提交word tuple不会做anchoring操作。如果这个新tuple后面处理失败了，根tuple不会被重放。根据你的拓扑中的容错策略，这样也是合理的有时。

一个输出tuple可以被anchoring到多个输入tuple。这种方式在stream合并或者stream聚合的时候很有用。一个多入口tuple处理失败的话，那么它对应的所有输入tuple都要重新执行。看看下面演示怎么指定多个输入tuple:

    List<Tuple> anchors = new ArrayList<Tuple>(); 
    anchors.add(tuple1); 
    anchors.add(tuple2); 
    _collector.emit(anchors, new Values(1, 2, 3));

多入口tuple把这个新tuple加到了多个tuple树里面去了。

![](http://storm.incubator.apache.org/documentation/images/tuple-dag.png)

Storm的实现既能处理tree，也能处理上面的DAG的情况。

我们通过anchoring来构造这个tuple树，最后一件要做的事情是在你处理完当个tuple的时候告诉storm,  通过OutputCollector类的ack和fail方法来做，如果你回过头来看看SplitSentence的例子， 你可以看到“句子tuple”在所有“单词tuple”被发出之后调用了ack。

你可以调用OutputCollector 的fail方法去立即将从消息源头发出的那个tuple标记为fail， 比如你查询了数据库，发现一个错误，你可以马上fail那个输入tuple， 这样可以让这个tuple被快速的重新处理， 因为你不需要等那个timeout时间来让它自动fail。

每个你处理的tuple， 必须被ack或者fail。因为storm追踪每个tuple要占用内存。所以如果你不ack/fail每一个tuple， 那么最终你会看到OutOfMemory错误。

大多数Bolt遵循这样的规律：读取一个tuple；发射一些新的tuple；在execute的结束的时候ack这个tuple。这些Bolt往往是一些过滤器或者简单函数。Storm为这类规律封装了一个BasicBolt类。如果用BasicBolt来做， 上面那个SplitSentence可以改写成这样：

    public class SplitSentence implements IBasicBolt {
        public void prepare(Map conf,
                            TopologyContext context) {
        }
 
        public void execute(Tuple tuple,
                            BasicOutputCollector collector) {
            String sentence = tuple.getString(0);
            for(String word: sentence.split(" ")) {
                collector.emit(new Values(word));
            }
        }
 
        public void cleanup() {
        }
 
        public void declareOutputFields(
                        OutputFieldsDeclarer declarer) {
            declarer.declare(new Fields("word"));
        }
    }


这个实现比之前的实现简单多了， 但是功能上是一样的。发送到BasicOutputCollector的tuple会自动和输入tuple相关联，而在execute方法结束的时候那个输入tuple会被自动ack的。

作为对比，处理聚合和合并的bolt往往要处理一大堆的tuple之后才能被ack， 而这类tuple通常都是多输入的tuple， 所以这个已经不是IBasicBolt可以罩得住的了。

## 如果tuple会被重放，如何保证应用程序正确性？

就像软件设计中常见的答案，“看情况”。Storm0.7.0引入了“事务拓扑”的特性，让你可以实现exactly-once的容错。不过这个接口已经过期了，现在请使用Trident。

## Storm如何高效实现可靠性？

storm里面有一类特殊的task称为：acker， 他们负责跟踪spout发出的每一个tuple的tuple树。当acker发现一个tuple树已经处理完成了。它会发送一个消息给产生这个tuple的那个task。你可以通过Config.TOPOLOGY_ACKERS来设置一个topology里面的acker的数量， 默认值是一。 如果你的topology里面的tuple比较多的话， 那么把acker的数量设置多一点，效率会高一点。

理解storm的可靠性的最好的方法是来看看tuple和tuple树的生命周期， 当一个tuple被创建， 不管是spout还是bolt创建的， 它会被赋予一个64位的id，而acker就是利用这个id去跟踪所有的tuple的。每个tuple知道它的祖宗的id(从spout发出来的那个tuple的id), 每当你新发射一个tuple， 它的祖宗id都会传给这个新的tuple。所以当一个tuple被ack的时候，它会发一个消息给acker，告诉它这个tuple树发生了怎么样的变化。具体来说就是：它告诉acker： 我呢已经完成了， 我有这些儿子tuple, 你跟踪一下他们吧。下面这个图演示了C被ack了之后，这个tuple树所发生的变化。

![](http://storm.incubator.apache.org/documentation/images/ack_tree.png)

关于storm怎么跟踪tuple还有一些细节， 前面已经提到过了， 你可以自己设定你的topology里面有多少个acker。而这又给我们带来一个问题， 当一个tuple需要ack的时候，它到底选择哪个acker来发送这个信息呢？

storm使用一致性哈希来把一个spout-tuple-id对应到acker， 因为每一个tuple知道它所有的祖宗的tuple-id， 所以它自然可以算出要通知哪个acker来ack。（这里所有的祖宗是指这个tuple所对应的所有的根tuple。这里注意因为一个tuple可能存在于多个tuple树，所以才有所有一说）。

storm的另一个细节是acker是怎么知道每一个spout tuple应该交给哪个task来处理。当一个spout发射一个新的tuple， 它会简单的发一个消息给一个合适的acker，并且告诉acker它自己的id(taskid)， 这样storm就有了taskid-tupleid的对应关系。 当acker发现一个树完成处理了， 它知道给哪个task发送成功的消息。

acker task并不显式的跟踪tuple树。对于那些有成千上万个节点的tuple树，把这么多的tuple信息都跟踪起来会耗费太多的内存。相反， acker用了一种不同的方式， 使得对于每个spout tuple所需要的内存量是恒定的（20 bytes) .  这个跟踪算法是storm如何工作的关键，并且也是它的主要突破。

一个acker task存储了一个spout-tuple-id到一对值的一个mapping。这个对子的第一个值是创建这个tuple的taskid， 这个是用来在完成处理tuple的时候发送消息用的。 第二个值是一个64位的数字称作：”ack val”, ack val是整个tuple树的状态的一个表示，不管这棵树多大。它只是简单地把这棵树上的所有创建的tupleid/ack的tupleid一起异或(XOR)。

当一个acker task 发现一个 ack val变成0了， 它知道这棵树已经处理完成了。 因为tupleid是随机的64位数字， 所以， ack val碰巧变成0(而不是因为所有创建的tuple都完成了)的几率极小。算一下就知道了， 就算每秒发生10000个ack， 那么需要50000000万年才可能碰到一个错误。而且就算碰到了一个错误， 也只有在这个tuple失败的时候才会造成数据丢失。 

[这篇文章](http://www.searchtb.com/2012/09/introduction-to-storm.html)中详细图解了这个XOR ack的过程。

既然你已经理解了storm的可靠性算法， 让我们一起过一遍所有可能的失败场景，并看看storm在每种情况下是怎么避免数据丢失的。

1. 由于对应的task挂掉了，一个tuple没有被ack： storm的超时机制在超时之后会把这个tuple标记为失败，从而可以重新处理。

2. Acker挂掉了： 这种情况下由这个acker所跟踪的所有spout tuple都会超时，也就会被重新处理。

3. Spout挂掉了： 在这种情况下给spout发送消息的消息源负责重新发送这些消息。比如Kestrel和RabbitMQ在一个客户端断开之后会把所有”处理中“的消息放回队列。

就像你看到的那样， storm的可靠性机制是完全分布式的， 可伸缩的并且是高度容错的。

## 调整可靠性

Acker task是轻量级的，你不需要在一个拓扑中使用太多。你可以再Storm UI中查看他们的性能表现。如果吞吐率看起来不正常，你可能需要多加点acker。

如果可靠性对你不重要-你不在乎丢失一些数据-那么你可以通过不跟踪tuple树来提高性能。不跟踪消息的话使得系统中消息的传递减少了一般，因为通常每个tuple都需要发送ack消息。除此以外，减少了下游tuple 64bit-id的记录，减少了网络带宽。

有三种方法降低可靠性。

1. Config.TOPOLOGY_ACKERS设置为0.这样spout提交了一个tuple后Storm会立即调用ack，tuple tree不会被跟踪。

2. 在tuple层面去掉。emit的时候不指定messageid。

3. 在bolt emit的时候，不要anchor上下游tuple。这样下游tuple是否成功不会影响上游。