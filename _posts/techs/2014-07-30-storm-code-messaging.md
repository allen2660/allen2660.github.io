---
layout: post
title:  Storm源码-backtype.storm.messaging 消息传输
---

本篇描述了Storm中消息传输的代码

## ZeroMQ

Storm 0.9以前使用ZeroMQ来做消息传输。0.9以后使用基于Netty的传出层，这给安装带来了很大的方便。

## 基于Netty的消息传输机制

### 接口

消息传输机制由两个接口来定义：IContext和IConnection。

#### IContext

该接口负责client和server端连接的建立，实现了这个接口且提供默认构造函数的类就是Message Plugin：

    public interface IContext {
        /**
         * This method is invoked at the startup of messaging plugin
         * @param storm_conf storm configuration
         */
        public void prepare(Map storm_conf);
        
        /**
         * This method is invoked when a worker is unload a messaging plugin
         */
        public void term(); 

        /**
         * This method establishes a server side connection 
         * @param storm_id topology ID
         * @param port port #
         * @return server side connection
         */
        public IConnection bind(String storm_id, int port);
        
        /**
         * This method establish a client side connection to a remote server
         * @param storm_id topology ID
         * @param host remote host
         * @param port remote port
         * @return client side connection
         */
        public IConnection connect(String storm_id, String host, int port);
    };

#### IConnection

该接口描述了在连接上发送接收数据的接口：

    public Iterator<TaskMessage> recv(int flags, int clientId);
    public void send(int taskId,  byte[] payload);
    public void send(Iterator<TaskMessage> msgs);
    public void close();

TaskMessage类就是一个简单的封装，封装了taskid(int)和message(byte[])。

### 实现

#### Context implements IContex

#### Client implements IConnection

#### Server implements IConnection