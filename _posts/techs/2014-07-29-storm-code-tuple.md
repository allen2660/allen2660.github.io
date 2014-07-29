---
layout: post
title:  Storm源码-backtype.storm.tuple包
---

本篇讲解storm的tuple包的主要类及其实现。主要有以下几个类/接口：

+ interface Tuple
+ class TupleImpl
+ class Values
+ class MessageId
+ class Fields

## Tuple

元组是Storm的主要数据结构。一个元组是一个有名字的值列表，其中每个值都可能是任意类型。元组是动态类型的--用户不需要事先声明值的类型。元组提供了像`getInteger`和`getString`这样的帮助方法来得到field的值。

这个接口除了一些getXXX，contains，FieldIndex等接口外，还提供了和Storm实现相关的几个接口：

    /**
     * Returns the global stream id (component + stream) of this tuple.
     */
    public GlobalStreamId getSourceGlobalStreamid();
    
    /**
     * Gets the id of the component that created this tuple.
     */
    public String getSourceComponent();
    
    /**
     * Gets the id of the task that created this tuple.
     */
    public int getSourceTask();
    
    /**
     * Gets the id of the stream that this tuple was emitted to.
     */
    public String getSourceStreamId();
    
    /**
     * Gets the message id that associated with this tuple.
     */
    public MessageId getMessageId();    


## Values

    /**
     * A convenience class for making tuple values using new Values("field1", 2, 3)
     * syntax.
     */
    public class Values extends ArrayList<Object>{
        public Values() {
            
        }
        
        public Values(Object... vals) {
            super(vals.length);
            for(Object o: vals) {
                add(o);
            }
        }
    }

## Fields

Fields就是一个String列表的封装。

    public class Fields implements Iterable<String>, Serializable {
        private List<String> _fields;
        private Map<String, Integer> _index = new HashMap<String, Integer>();
    
        //...
    }

## MessageId

封装了Map<Long, Long>，表示 anchor 到id的映射。同时还实现了序列化反序列化方法，顺便展示了Kryo的用法：

    import com.esotericsoftware.kryo.io.Input;
    import com.esotericsoftware.kryo.io.Output;

    public class MessageId {
        private Map<Long, Long> _anchorsToIds;

        public void serialize(Output out) throws IOException {
            out.writeInt(_anchorsToIds.size(), true);
            for(Entry<Long, Long> anchorToId: _anchorsToIds.entrySet()) {
                out.writeLong(anchorToId.getKey());
                out.writeLong(anchorToId.getValue());
            }
        }   

        public static MessageId deserialize(Input in) throws IOException {
            int numAnchors = in.readInt(true);
            Map<Long, Long> anchorsToIds = new HashMap<Long, Long>();
            for(int i=0; i<numAnchors; i++) {
                anchorsToIds.put(in.readLong(), in.readLong());
            }
            return new MessageId(anchorsToIds);
        }
    }


## TupleImpl

    public class TupleImpl extends IndifferentAccessMap implements Seqable, Indexed, IMeta, Tuple {
        private List<Object> values;
        private int taskId;
        private String streamId;
        private GeneralTopologyContext context;
        private MessageId id;
        private IPersistentMap _meta = null;

        //构造函数
        public TupleImpl(GeneralTopologyContext context, List<Object> values, int taskId, String streamId, MessageId id) {
            this.values = values;
            this.taskId = taskId;
            this.streamId = streamId;
            this.id = id;
            this.context = context;
            
            String componentId = context.getComponentId(taskId);
            Fields schema = context.getComponentOutputFields(componentId, streamId);
            if(values.size()!=schema.size()) {
                throw new IllegalArgumentException(
                        "Tuple created with wrong number of fields. " +
                        "Expected " + schema.size() + " fields but got " +
                        values.size() + " fields");
            }
        }

        //...
    }

该类为了在Clojure中使用，实现了Seqable，Indexed，IMeta，这块内容后面看到Clojure的时候再细细看。