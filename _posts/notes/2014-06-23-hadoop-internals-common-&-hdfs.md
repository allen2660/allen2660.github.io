---
layout:	post
title:  《Hadoop技术内幕-Common&HDFS》读书笔记
---

# 第二部分 Common

## 第2章 配置信息

常用的配置管理类：

+ java.util.Properties
+ Apache Jakarta Commons
+ org.apache.hadoop.conf.Configuration

Hadoop自己做了一个Configuration类，用于hdfs，mr的配置文件解析。

Configurable接口：一个类是可配置的。

+ setConf()

## 第3章 序列化&压缩

序列化： 将一个对象编码成一个字节流
反序列化：相反。

序列化的作用：

+ 持久化
+ 通信数据格式
+ 拷贝/克隆

Java内置序列化：

+ 对象实现 Serializable接口，不需要实现任何方法
+ ObjectOutputStream.writeObject(obj);
+ Java的内置序列化，Size太大，里面包含太多类的元信息。

Hadoop序列化：

+ 所有可序列化对象实现这个借口：org.apache.hadoop.io.Writable
+ 实现两个方法：write(DataOutput) ; readFields(DataInput);
+ obj.write(DataOutputStream);

Hadoop序列化特点：紧凑、快速、可扩展、跨语言

典型的Writable：

+ 基本类型的Writable
+ ObjectWritable: 对于不知道类型是什么的value对象，比如SequenceFile的value。

其他序列化框架：

+ Avro
+ Thrift
+ Protocol Buffer

### 压缩

CompressCodec

Compressor/Decompressor

如果要添加一个压缩算法，则需要以下几个元素：

+ SnappyCompressCodec & "org.xxx.SnappyCodec"
+ SnappyCompressor/SnappyDecompressor

## 第四章 Hadoop RPC（远程过程调用）

### RPC

RPC的一般实现中，客户存根(client stub)和服务器骨架(server skeleton)都是自动生成的。

### RMI

Java自带的远程过程调用

### Java 动态代理

java.lang.reflect.Proxy java.lang.reflect.InvocationHandler

Demo:

    public interface PDQueryStatus {
        DPFileStatus getFileStatus(String filename);
    }

    public class DPInvocationHandler implements InvocationHandler {
        private DPQueryStatusImpl dpqs;

        public DPInvocationHandler(DPQueryStatusImpl dpqs) {
            this.dpqs = dpqs;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            Object ret = null;

            // 执行一些附加功能

            ret = method.invoke(dpqs,args);

            return ret;
        }
    }

    public static void main(String[] args) {
        Class<?>[] interfaces = new Class[] {PDQueryStatus};

        DPQueryStatusImpl dpqs = new DPQueryStatusImpl(); //这个类是真正干活的
        DPInvocationHandler handler = new DPInvocationHandler();

        Object proxy = Proxy.newProxyInstance(dpqs.getClass().getCloassLoader(), interfaces, handler);

        DPStatus status = proxy.getFileStatus("/home/usr/123");
    }


### NIO

#### Socket

client:

+ connect()
+ getInputStream() read() 阻塞
+ getOutputStream() write() 阻塞

server:

+ bind()
+ accept() 阻塞
+ getInputStream() read() 阻塞
+ getOutputStream() write() 阻塞

一般使用多线程模型，会有大量的闲置客户端



#### New IO

特点：非阻塞

Buffer：

+ get()/write()
+ read()/put()
+ flip()
+ rewind()
+ clear()
+ compact()

Channel

非阻塞

Selector：可以注册到多个Channel中，然后等待。

### Hadoop IPC

TODO，先不看。

## 第5章 Hadoop文件系统

### Linux文件系统回顾

inode-128字节

VFS 虚拟文件系统

### 分布式文件系统

### Java文件系统

File

URI/URL

InputStream/OutputStream

FileInputDtream/FilterInputStream

DataINputrStream extends FilterInputStream implements DataInput


### Hadoop 抽象文件系统

和Java一样，API分为两个层面：

+ FileSystem
+ Input/OutputStream  FSDataInputStream/FSDataOutputStream



org.apache.hadoop.fs.FileSystem

FileStatus


# 第三部分 HDFS

## 第6章 HDFS概述

Block：

+ 64M
+ 降低Namenode存储压力（i-node数目降低）

NameNode & Secondnamenode

+ NN 产出FSImage & Edit Log
+ SNN定期将其合并，上传并替换NN上的FSImage

DistributedFileSystem extends org.apache.hadoop.fs.FileSystem
DFSDataInputStream extends FSDataInputStream
DFSDataOutputStream extends FSDataOutputStream

HDFS Java API:

    //打开文件
    Path inPath = new Path("hdfs://ip:port/user/liwei/in/hello.txt");
    FileSystem hdfs = FileSystem.get(inPath.toUri(), conf));
    
    //写文件
    FSDataOutputStream foutj = hdfs.create(inPath);
    String data = "testingtesting";
    for(int ii = 0; ii < 256; ii++) {
        fout.write(data.getBytes());
    }
    fout.close();

    //获取状态
    FileStatus stat = hdfs.getFileStatus(inPath);
    System.out.println("Replication number of file "+ inPath + " is " + stat.getReplication());

    //删除
    hdfs.delete(inPath);

HDFS源码都在 org.apache.hadoop.hdfs下。
HDFS源码都在 org.apache.hadoop.hdfs下。

## 第7章 DataNode 

## 第8章 Namenode