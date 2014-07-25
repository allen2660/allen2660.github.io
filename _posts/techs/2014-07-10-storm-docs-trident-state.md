---
layout: post
title:  trident api
---

本文为Storm官方文档[TridentState](http://storm.incubator.apache.org/documentation/Trident-state.html)的读书笔记。

## Trident中的State

Trident提供一级抽象用于读写有状态的数据源。状态可以是拓扑内部的(in-memory/HDFS)或者存储在外部db(Memcached/Cassandra)。这两者在Trident API中是没有区别的。

Trident以容错的方式管理状态，这样状态更新就是幂等的，即使在重试/失败的情况下。这样确保Trident拓扑的消息有且只处理一次。

在状态更新的时候有不同的容错急切。

Trident提供了以下语义来保证一次处理：

1. 元组以小batch的方式处理（参见[tutorial](http://storm.incubator.apache.org/documentation/Trident-tutorial.html)）
2. 每个元组batch给定一个唯一id “transaction id”(txid)。如果batch被回放，txid不变。
3. 状态更新以batch排序。batch2没有成功update，是不会update batch3的。

有三个级别的容错Spout，对应三个级别的容错。

### Transactional spouts

事务spout有以下属性：

1. 一个给定txid的batch是相同的。batch的回放提交的元组是一样的
2. 一个元组只会在一个batch中
3. 每个元组都属于一个batch

这类spout很好理解，流被切成了定长的batch。storm-contrib中有一个[事务spout for kafka](https://github.com/nathanmarz/storm-contrib/blob/master/storm-kafka/src/jvm/storm/kafka/trident/TransactionalTridentKafkaSpout.java)。

处理的时候需要两个字段：
    
    man => [count=3, txid=1] dog => [count=4, txid=3] apple => [count=10, txid=2]

### Opaque transactional spouts

不透明事务spout：不能保证对于一个txid的batch是不变的。

[OpaqueTridentKafkaSpout](https://github.com/nathanmarz/storm-contrib/blob/master/storm-kafka/src/jvm/storm/kafka/trident/OpaqueTridentKafkaSpout.java)就是这样一个spout，且可以容忍kafka丢节点。每次它emitbath的时候，它提交上一个batch提交截止开始的时候的元组。这保证没有元组漏掉或者被多个batch处理。

由于同样的txid对于的batch内容可能不一样，所以不能跳过相同txid的方式来做State update。

处理的时候，就需要三个字段：

    { value = 3, prevValue = 1, txid = 2 }

### Non-transactional spouts

这种spout没有事务保证，可能是“至少一次”处理，也可能是“最多一次”处理。

### spout和state类型的总结

![](http://storm.incubator.apache.org/documentation/images/spout-vs-state.png)

Opaque 事务状态有最强的容错性，但这需要在db中存储txid和两个值。事务状态可以少存一个值，但必须要事务spout才行。

spout和state的取舍是“容错”和“存储消耗”二者的tradeoff。最终你的应用需求决定了使用哪种组合。

## State APIs

你已经看到了如何实现“只处理一次”的语义。好消息是这些Trident都已经在State中封装了容错逻辑-你不需要处理比较txid、存储多个值在db中这些事情，你可以这么写代码：

    TridentTopology topology = new TridentTopology(); 
    TridentState wordCounts = topology.newStream("spout1", spout) 
                                .each(new Fields("sentence"), new Split(), new Fields("word")) 
                                .groupBy(new Fields("word")) 
                                .persistentAggregate(MemcachedState.opaque(serverLocations), new Count(), new Fields("count")) 
                                .parallelismHint(6);

所有的和opaque事务state相关的管理都已经在MemcachedState.opaque这个调用中保证了。另外，更新自动被批量化了以减少与db交互次数。

State接口只有两个方法：

    public interface State { 
        void beginCommit(Long txid); // can be null for things like partitionPersist occurring off a DRPC stream 
        void commit(Long txid); 
    }

    public class LocationDB implements State { 
        public void beginCommit(Long txid) { }
        public void commit(Long txid) {    
        }   

        public void setLocation(long userId, String location) {
            // code to access database and set location
        }   

        public String getLocation(long userId) {
            // code to get location from database
        } 
    }

    public class LocationDBFactory implements StateFactory { 
        public State makeState(Map conf, int partitionIndex, int numPartitions) { 
            return new LocationDB(); 
        } 
    }

Trident提供了QueryFunction接口来查询一个State源，以及StateUpdater接口来更新State。下面的拓扑以一个userids流作为输入。

    TridentTopology topology = new TridentTopology(); 
    TridentState locations = topology.newStaticState(new LocationDBFactory()); 
    topology.newStream("myspout", spout) 
        .stateQuery(locations, new Fields("userid"), new QueryLocation(), new Fields("location"));
    //这个拓扑提交到DRPC就是一个根据userid得到location的DRPC

QueryLocation的实现：

    public class QueryLocation extends BaseQueryFunction<LocationDB, String> { 
        public List batchRetrieve(LocationDB state, List inputs) { 
            List ret = new ArrayList(); 
            for(TridentTuple input: inputs) { 
                ret.add(state.getLocation(input.getLong(0))); 
            } 
            return ret;
        }

        public void execute(TridentTuple tuple, String location, TridentCollector collector) {
            collector.emit(new Values(location));
        } 
    }


要想更新state，需要StateUpdater接口：

    public class LocationUpdater extends BaseStateUpdater<LocationDB> { 
        public void updateState(LocationDB state, List<TridentTuple> tuples, TridentCollector collector) { 
            List<Long> ids = new ArrayList<Long>(); 
            List<String> locations = new ArrayList<String>(); 
            for(TridentTuple t: tuples) { 
                ids.add(t.getLong(0)); 
                locations.add(t.getString(1)); 
            } 
            state.setLocationsBulk(ids, locations); 
        } 
    }

下面是在Trident拓扑中如何使用这个updater：

    TridentTopology topology = new TridentTopology(); 
    TridentState locations = topology.newStream("locations", locationsSpout) 
                                .partitionPersist(new LocationDBFactory(), 
                                                    new Fields("userid", "location"), 
                                                    new LocationUpdater()
                                                    );
    //这个拓扑从外部locationsSpout更新数据到State中，亦即DB

partitionPersist操作更新state来源。StateUpdater接受State和一批元组以及对于State的更新。这段代码从输入元组中拿到userids和locations，同意更新到State中。

partitionPersist返回一个TridentState对象，其代表着Trident拓扑更新的location db。你可以使用这个在拓扑的其他地方的stateQuery操作中使用这个对象。


## persistentAggregate

Trident还有一个更新State的方法叫做persistentAggregate。你已经见过下面的流式wc的例子：

    TridentTopology topology = new TridentTopology(); 
    TridentState wordCounts = topology.newStream("spout1", spout) 
                                .each(new Fields("sentence"), new Split(), new Fields("word")) 
                                .groupBy(new Fields("word")) 
                                .persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))；

persistentAggregate是简历在partitionPersist上的抽象，且其知道如何如何利用Trident聚合来更新state源。在这个例子中，因为这是一个分组流，Trident希望你实现“MapState”接口的实现。分组字段会成为state的key，聚合结果会是value。“MapState”的接口如下：

    public interface MapState<T> extends State { 
        List<T> multiGet(List<List<Object>> keys); 
        List<T> multiUpdate(List<List<Object>> keys, List<ValueUpdater> updaters); 
        void multiPut(List<List<Object>> keys, List<T> vals); 
    }

当在非分组流上做聚合的时候（global分组），Trident希望State对象时实现“Snapshottable”接口：

    public interface Snapshottable<T> extends State { 
        T get(); 
        T update(ValueUpdater updater); 
        void set(T o); 
    }

[MemoryMapState](https://github.com/apache/incubator-storm/blob/master/storm-core/src/jvm/storm/trident/testing/MemoryMapState.java)和[MemcachedState](https://github.com/nathanmarz/trident-memcached/blob/master/src/jvm/trident/memcached/MemcachedState.java)分别实现了上面两个接口。


## Implementing Map States

Trident让实现MapState变得很简单，它帮你做了大部分工作。OpaqueMap、TransactionalMap以及NonTransactionMap实现了所有的容错逻辑。你只需要给这些类提供一个IBackingMap接口，该接口负责multiGets和multiPuts相关的kv。

    public interface IBackingMap<T> { 
        List<T> multiGet(List<List<Object>> keys); 
        void multiPut(List<List<Object>> keys, List<T> vals); 
    }


可以参考[MemcachedState](https://github.com/nathanmarz/trident-memcached/blob/master/src/jvm/trident/memcached/MemcachedState.java)的实现来看看这些组件是如何结合在一起实现一个高性能的MapState的。MemcachedState允许你选择三种事务容错级别中的一个。