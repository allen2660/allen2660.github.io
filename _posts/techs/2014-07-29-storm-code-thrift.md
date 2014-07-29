---
layout: post
title:  Storm源码-storm.thrift
---

## storm.thrift

storm.thrift位于`storm-core/src/storm.thrift`，生成的代码在`storm-core/src/jvm/generated/`，生成的逻辑也很简单：

    rm -rf gen-javabean gen-py py
    rm -rf jvm/backtype/storm/generated
    thrift7 --gen java:beans,hashcode,nocamel --gen py:utf8strings storm.thrift
    mv gen-javabean/backtype/storm/generated jvm/backtype/storm/generated
    mv gen-py py
    rm -rf gen-javabean

storm.thrift中的结构体是Nimbus/Supervisor/Acker/DRPC-Server之间交互的数据描述。所以他们很重要。thrift和proto很像，只不过其除了提供序列化反序列化能力外，还可以提供service的桩代码。

## 服务

有下面三个服务：

    service DistributedRPC {
        string execute(1: string functionName, 2: string funcArgs) throws (1: DRPCExecutionException e);
    }

    service DistributedRPCInvocations {
        void result(1: string id, 2: string result);
        DRPCRequest fetchRequest(1: string functionName);
        void failRequest(1: string id);  
    }

    service Nimbus {
      void submitTopology(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology) throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite);
      void submitTopologyWithOpts(1: string name, 2: string uploadedJarLocation, 3: string jsonConf, 4: StormTopology topology, 5: SubmitOptions options) throws (1: AlreadyAliveException e, 2: InvalidTopologyException ite);
      void killTopology(1: string name) throws (1: NotAliveException e);
      void killTopologyWithOpts(1: string name, 2: KillOptions options) throws (1: NotAliveException e);
      void activate(1: string name) throws (1: NotAliveException e);
      void deactivate(1: string name) throws (1: NotAliveException e);
      void rebalance(1: string name, 2: RebalanceOptions options) throws (1: NotAliveException e, 2: InvalidTopologyException ite); 

      // need to add functions for asking about status of storms, what nodes they're running on, looking at task logs   

      string beginFileUpload();
      void uploadChunk(1: string location, 2: binary chunk);
      void finishFileUpload(1: string location);
      
      string beginFileDownload(1: string file);
      //can stop downloading chunks when receive 0-length byte array back
      binary downloadChunk(1: string id);   

      // returns json
      string getNimbusConf();
      // stats functions
      ClusterSummary getClusterInfo();
      TopologyInfo getTopologyInfo(1: string id) throws (1: NotAliveException e);
      //returns json
      string getTopologyConf(1: string id) throws (1: NotAliveException e);
      StormTopology getTopology(1: string id) throws (1: NotAliveException e);
      StormTopology getUserTopology(1: string id) throws (1: NotAliveException e);
    }

很明了了，前两个用于实现DRPC。后一个是Nimbus提供给storm-client的thrift service。