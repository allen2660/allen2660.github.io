---
layout: post
title:  Storm源码-backtype.storm.serialization包
---

## Serialization背景

[serialization](http://storm.incubator.apache.org/documentation/Serialization.html)

元组由任意的类型组成。Storm是一个分布式系统，所以上面任务之间传递数据就需要用到序列化/反序列化。Storm使用[Kryo](http://code.google.com/p/kryo/)来做序列化。Kryo是一个弹性的、快速的序列化库，生成的二进制格式很小。

默认的，Storm可以序列化基本类型、字符串、字符数组、ArrayList、HashMap、HashSet，以及Clojure集合类型。如果在元组里想使用自己的类型，你需要注册自定义的序列化器。

### 动态类型

在Tuple中没有类型声明。你在field中放对象，Storm动态找出其序列化。在我们讨论serialization接口之前，让我们花一些时间理解下为什么Storm的元组是动态类型的。

在元组中加入静态类型会大大增加Storm API的复杂度。举例来说，Hadoop的key，value都是静态类型但是需要在客户端有大量的注解。Hadoop的API使用起来就是一种负担，且“类型安全”不值得这样。动态类型使用起来更简单。

除此以外，静态化Storm拓扑的类型是不可能的。假设Bolt订阅了多个流。这些流传递过来的元组可能拥有不同的filed类型。当Bolt在execute函数中接收到一个元组，该元组可能来自任意流且可能拥有任意类型的组合。可能可以使用反射来为每一个订阅的流声明不同的方法，但是Storm使用了更简单、直接的动态类型。

最后，另一个使用动态类型的原因是这样的话Storm就可以更直接的被动态语言使用：Clojure、Ruby。


### 自定义序列化

Storm使用Kryo做序列化。为了实现自定义序列化器，你需要向Kryo注册序列化器。强烈推荐你阅读[Kryo's home page](http://code.google.com/p/kryo/)来理解如何处理自定义序列化。

添加自定义序列化器通过“topology.kryo.register”属性来添加。这需要一系列注册，每个都是下面的两种形式之一：

1. 类名。这种情况下，Storm使用Kryo的FieldsSerializer来序列化该类。
2. 从类名到[com.esotericsoftware.kryo.Serializer](http://code.google.com/p/kryo/source/browse/trunk/src/com/esotericsoftware/kryo/Serializer.java)的实现的映射。

让我们看一个例子：

    topology.kryo.register: 
        - com.mycompany.CustomType1 
        - com.mycompany.CustomType2: com.mycompany.serializer.CustomType2Serializer 
        - com.mycompany.CustomType3

第一个和第三个使用FieldsSerializer，第二个使用自己的CustomType2Serializer类来做序列化。

Storm提供了在拓扑配置中注册序列化器的帮助函数。[Config](http://storm.incubator.apache.org/apidocs/backtype/storm/Config.html)类有一个方法叫做`registerSerialization`方法。

有一个高级配置叫做`Config.TOPOLOGY_SKIP_MISSING_KRYO_REGISTRATIONS`。如果为true，Storm会忽略没有找到的序列化类。不然，Storm会抛异常。当你在一个集群上运行很多拓扑，每个都有自己的序列化类，而你想在storm.yaml中把他们都指定的时候，这个配置很有用。

### Java 序列化

如果Storm需要一个没有注册的类需要序列化，它会使用Java serialization。如果该类无法使用Java序列化，Storm报错。

Java序列化非常耗费资源，不管是CPU还是空间占用。强烈建议注册自定义的序列化器。当你需要快速开发原型的时候，Java序列化才是可以用的。

你可以使用`Config.TOPOLOGY_FALL_BACK_ON_JAVA_SERIALIZATION`关闭Java序列化的fall back。



## Serialization


### ITupleSerializer/ITupleDeserializer

这两个接口定义了序列化器和反序列化器的基本接口：

    public interface ITupleDeserializer {
        Tuple deserialize(byte[] ser);        
    }   

    public interface ITupleSerializer {
        byte[] serialize(Tuple tuple);
    }

### KryoTupleSerializer/KryoTupleDeserializer

这两个类提供了序列化和反序列化Tuple的实现。使用Kryo的Output类和Input类，MessageId交给它自己的序列化方法，Values的序列化交给KryoValuesXXSerializer来做：

    public class KryoTupleSerializer implements ITupleSerializer {
        KryoValuesSerializer _kryo;
        SerializationFactory.IdDictionary _ids;   
        Output _kryoOut;
        
        public KryoTupleSerializer(final Map conf, final GeneralTopologyContext context) {
            _kryo = new KryoValuesSerializer(conf);
            _kryoOut = new Output(2000, 2000000000);
            _ids = new SerializationFactory.IdDictionary(context.getRawTopology());
        }   

        public byte[] serialize(Tuple tuple) {
            try {
                
                _kryoOut.clear();
                _kryoOut.writeInt(tuple.getSourceTask(), true);
                _kryoOut.writeInt(_ids.getStreamId(tuple.getSourceComponent(), tuple.getSourceStreamId()), true);
                tuple.getMessageId().serialize(_kryoOut);
                _kryo.serializeInto(tuple.getValues(), _kryoOut);
                return _kryoOut.toBytes();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }   

    public class KryoTupleDeserializer implements ITupleDeserializer {
        GeneralTopologyContext _context;
        KryoValuesDeserializer _kryo;
        SerializationFactory.IdDictionary _ids;
        Input _kryoInput;
        
        public KryoTupleDeserializer(final Map conf, final GeneralTopologyContext context) {
            _kryo = new KryoValuesDeserializer(conf);
            _context = context;
            _ids = new SerializationFactory.IdDictionary(context.getRawTopology());
            _kryoInput = new Input(1);
        }           

        public Tuple deserialize(byte[] ser) {
            try {
                _kryoInput.setBuffer(ser);
                int taskId = _kryoInput.readInt(true);
                int streamId = _kryoInput.readInt(true);
                String componentName = _context.getComponentId(taskId);
                String streamName = _ids.getStreamName(componentName, streamId);
                MessageId id = MessageId.deserialize(_kryoInput);
                List<Object> values = _kryo.deserializeFrom(_kryoInput);
                return new TupleImpl(_context, values, taskId, streamName, id);
            } catch(IOException e) {
                throw new RuntimeException(e);
            }
        }
    }


### SerializationFactory.IdDictionary

这个类记录了对于一个拓扑的每个Component：

    componentName-> Map<streamName,i>
    componentName-> Map<i,streamName>

这样在序列化一个Tuple的时候，只要序列化streamId，反序列化的时候，再通过streamId在字典里查streamName。


### KryoValuesSerializer/KryoValuesDeserializer

这两个类用于序列化/反序列化Values对象。

这里使用ListDelegate来封装List<Object>，这样保证不论是Java集合还是Clojure集合都可以以同样方式写入。

    public class KryoValuesSerializer {
        Kryo _kryo;
        ListDelegate _delegate;
        Output _kryoOut;
        
        public KryoValuesSerializer(Map conf) {
            _kryo = SerializationFactory.getKryo(conf);
            _delegate = new ListDelegate();
            _kryoOut = new Output(2000, 2000000000);
        }
        
        public void serializeInto(List<Object> values, Output out) throws IOException {
            // this ensures that list of values is always written the same way, regardless
            // of whether it's a java collection or one of clojure's persistent collections 
            // (which have different serializers)
            // Doing this lets us deserialize as ArrayList and avoid writing the class here
            _delegate.setDelegate(values);
            _kryo.writeObject(out, _delegate); 
        }
        
        public byte[] serialize(List<Object> values) throws IOException {
            _kryoOut.clear();
            serializeInto(values, _kryoOut);
            return _kryoOut.toBytes();
        }
        
        public byte[] serializeObject(Object obj) {
            _kryoOut.clear();
            _kryo.writeClassAndObject(_kryoOut, obj);
            return _kryoOut.toBytes();
        }
    }

### IKryoFactory/DefaultKryoFactory

这两个类描述了KryoFactory的接口，就是说，到时候需要通过这个Factory的getKryo()方法得到Kryo对象。当然还有几个hook。

说一下Kryo和Input的关系：

    Input.readInt();
    Object obj = kryo.readObject(input, ListDelegate.class);
    kryo.writeObject(out, obj);

在读写复杂类型的时候，就需要Kryo类的read/write方法了。


这里还用到了SerializableSerializer，这个类覆盖了Serializer的方法read/write，在写入前先写一个length，读的时候亦然。

### SerializationFactory

这个类用于在Storm运行时得到已经注册过各种序列化器的Kryo。

这里用到了下面几个序列化器：

+ types/ArrayListSerializer
+ types/HashMapSerializer
+ types/HashSetSerializer
+ types/ListDelegateSerializer

主要方法就是 static getKryo()方法。其中会调用Kryo.register()方法注册默认带的几个以及用户自定义的序列化器。