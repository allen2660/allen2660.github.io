---
layout: post
title:  set up a storm cluster
---

本文为Storm官方文档[SettingUpStormCluster](http://storm.incubator.apache.org/documentation/Setting-up-a-Storm-cluster.html)的读书笔记

搭建Storm集群的步骤总结：

1. 搭建Zookeeper 集群
2. 在Nimbus和Worker机器上安装依赖
3. 下载解压Storm release到Nimbus和worker机器上
4. 将强制性的配置文件写入storm.yaml
5. 启daemon

我的环境：

Zookeeper机器：
     liwei12@cq01-dt-udwtest02.cq01.baidu.com
     /home/liwei12/zookeeper

Nimbus & UI：
     liwei12@cq01-dt-udwtest02.cq01.baidu.com
     /home/liwei12/storm/nimbus

Supervisors：
     liwei12@cq01-rdqa-dev040.cq01.baidu.com
     /home/user/liwei12/storm/supervisor
     work@tc-dt-logictest01.tc.baidu.com
     /home/work/liwei12/storm/supervisor

Storm Client：
    liwei12@cq01-dt-udwtest02.cq01.baidu.com
    /home/liwei12/CVS/liwei12/storm/

# 搭建Zookeeper集群

我使用的是单机模式的Zookeeper，照着[这里](http://zookeeper.apache.org/doc/r3.3.3/zookeeperStarted.html#sc_InstallingSingleMode)搭就行了，注意配置文件：

    tickTime=2000
    dataDir=/home/liwei12/zookeeper/data
    logDir=/home/liwei12/zookeeper/log
    clientPort=2181

# 在Nimbus和Worker机器上安装依赖

要求：

     Java6 
     Python 2.6.6+


# 下载解压Storm release到Nimbus和worker机器上

我下载的是0.9.2版本，[这里](http://apache.fayea.com/apache-mirror/incubator/storm/apache-storm-0.9.2-incubating/apache-storm-0.9.2-incubating.tar.gz)

# 配置

Nimbus和Supervisor 的storm.yaml配置：

    storm.zookeeper.servers:
     - "cq01-dt-udwtest02.cq01.baidu.com"
    storm.local.dir: "/home/liwei12/storm/nimbus/data"
    nimbus.host: "cq01-dt-udwtest02.cq01.baidu.com"
    ui.port: 8081
    drpc.servers:
     - "cq01-dt-udwtest02.cq01.baidu.com"
    
# 启动Daemon

在Nimbus机器上启动Nimbus
    
    bin/storm nimbus

在supervisor上启动supervisor

    bin/storm supervisor

在Nimbus上启动ui

    bin/storm ui

在Nimbus上同时启动drpc

    bin/storm drpc

以上的启动，最好使用supervise工具启动，以避免异常退出，我是在screen的每个session中挨个启动的，异常退出会有感知。