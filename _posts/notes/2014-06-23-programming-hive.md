---
layout: default
title:  《Hive编程指南》阅读笔记
---

# 第1章 基础知识

Hive 不支持记录级别的更新、插入、删除，不支持事务。

SQL: 一个有效地、合理地且直观地组织和使用数据的模型。

如果需要对大规模数据使用OLAP功能，可以选择NoSQL数据库：HBase、Cassandra、BynamoDB。

Hive的组成模块：

+ CLI HWI ThriftServer JDBC ODBC
+ Driver MetaStore
+ Mapper Reducer 执行XML Job Plan

HQL Demo(Word Count)：

    CREATE TABLE docs (line STRING);

    LOAD DATA INPATH 'docs' OVERWRITE INTO TABLE docs;

    CREATE TABLE word_counts AS
    SELECT word, count(1) AS count FROM
        (SELECT explode(split(line, '\s')) AS word FROM docs) w
    GROUP BY word
    ORDER BY word;


# 第2章 基础操作

Hive三种元数据模式：

+ 用户CWD下建立derby db。
+ MySQL做中央存储
+ MetaServer，'hive.metastore.urls'

Hive中变量和属性命名空间：

+ hivevar 用户自定义变量
+ hiveconf 
+ system：Java定义的配置属性
+ env：Shell环境定义的环境变量
    + 调度器系统可以用这套机制来替换

hiverc：

+ HOME/.hiverc
+ -i 参数指定

hiverc Demo:

    ADD JAR /path/to/custom_hive_extension.jar;
    set hive.cli.print.current.db=true;
    set hive.exec.mode.local.auto=true;
    set hive.cli.print.header=true;

# 第3章 数据类型和文件格式

集合数据类型：

+ struct
+ map
+ array

Hive中没有键的概念，不过可以建立索引

# 第4章 HQL：Data Definition

文件编码

DDL Demo：

    CREATE TABLE employees (
        name STRING,
        salary  FLOAT,
        subordinates ARRAY<STRING>,
        deductions MAP<STRING,STRING>,
        address STRUCT<street:STRING, city:STRING, state:STRING, zip:INT>
    )
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY '\001'
    COLLECTION ITEMS TERMINATED BY '\002'
    MAP KEYS TERMINATED BY '\003'
    LINES TERMINATED BY '\n'
    STORED AS TEXTFILE;

创建DB:

    CREATE DATABASE financials
    LOCATION '/my/preferred/directory' // 不指定就默认${hive.metastore.warehouse.dir}/dbname.db
    WITH DBPROPERTIED ('creator' = 'Mark Moneybags', 'date' = '2012-01-02');

创建表：

    DESCRIBE EXTENDED mydb.employees;

    DESCRIBE FORMATTED mydb.employees;

Partition:

    CREATE TABLE employees (
        name STRING,
        salary FLOAT
    )
    PARTITIONED BY (country STRING, state STRING);

    set hive.mapred.mode=strict; //禁止在查询分区表的时候不指定partition字段

    SHOW PARTITIONS employees;

    SHOW PARTITIONS employees PARTITION(country='US');
    SHOW PARTITIONS employees PARTITION(country='US', state='AK');

    LOAD DATA INPATH '' INTO TABLE employees
    PARTITION (country = 'US', state = 'CA');

外部分区表：

    ALTER TABLE log_message ADD PARTITION(year = 2012, month = 1, day = 2)
    LOCATION 'hdfs://master_server/data/log_messages/2012/01/02';

    ALTER TABLE log_message PARTITION(year = 2012, month =1, day = 2)
    SET LOCATION 's3n://ourbucket/logs/2012/01/02';

    DESCRIBE EXTENDED log_messages PARTITION (year = 2012, month = 1, day = 2);

Hive数据存取相关：

+ InputFormat 将数据流分割成记录
+ OutputFormat 将记录格式化为输出流
+ Ser 序列化器 将记录解析成列
+ De 反序列化器 将列编码成记录

AvroHive Serde

    CREATE TABLE kst
    PARTITIONED BY (ds string)
    ROW FORMAT SERDE 'com.linkedin.haivvreo.AcroSerDe'
    WITH SERDEPROPERTIES ('schema.url'='http://schema_provider/kst.avsc')
    STORED AS
    INPUTFORMAT 'com.linkedin.haivvreo.AvroContainerInputFormat'
    OUTPUTFORMAT 'com.linkedin.haivvreo.AvroContainerOutputFormat';

修改表：

    ALTER TABLE log_messages RENAME TO logmsgs;

    ALTER TABLE log_messages
    PARTITION(year = 2012, month = 1, day = 1)
    SET FILEFORMAT SEQUENCEFILE;

    ALTER TABLE log_messages
    SET SERDE 'com.example.JSONSerDe'
    WITH SERDEPROPERTIES (
        'prop1' = 'value1',
        'prop2' = 'value2');

    ALTER TABLE stocks
    CLUSTERED BY (exchange, symbol)
    SORTED BY (symbol)
    INTO 48 BUCKETS;

    ALTER TABLE log_messages
    PARTITION(year = 2012, month = 1, day = 1) ENABLE NO_DROP;

    ALTER TABLE log_messages
    PARTITION(year = 2012, month = 1, day = 1) ENABLE OFFLINE;

# 第5章 HQL: 数据操作

# 第6章 HQL: 查询

Hive内置函数：P88前后，太多了

+ 数学函数 sin()
+ 聚合函数 sum()
+ 表生成函数 explode()
+ 其他内置函数 regexp_replace()


LEFT SEMI JOIN：实现IN 子查询

ORDER BY 全排序

SORT BY reducer内部排序

DISTRIBUTED BY SORT BY 全局排序



# 第7章 HQL: 视图

# 第8章 HQL: 索引

# 第9章 模式设计

# 第10章 调优

##JOIN优化

大表写后面，或者使用/*streamtable(table)*/ 指定大表。

## 本地模式

## reducer数目

首先可以手工指定reducer数目。

    mapred.reduce.tasks// Hive默认reducer个数是3
Hive是按照输入的数据量大小来确定reducer个数的。

    hive.exec.reduces.bytes.per.reducer
    hive.exec.reducers.max

# 第11章 其他文件格式和压缩算法

# 第12章 开发

# 第13章 函数

# 第14章 Streaming

使用TRANSFORM 操作结合命令行管道来实现。

GenericMR

# 第15章 自定义文件和记录格式

我们describe extended tableName的时候：

    Detailed Table Infomation
    Table
    lastAccessTime:0 ， retention:0
    sd:StorageDescriptor(
        cols:,
        location:,
        inputFormat:,
        outputFormat:,
        compressed:false,
        numBuckets:-1,
        serdeInfo:SerDeInfo(
            name:null,
            serializationLib:org.apache.hasoop.hive.serde2.lazy.lazySimpleSerDe,
            parameters:{serialization.format=1}
        ),   
        bucketCols:[], sortCols:[], parameters:{}, partitionKeys:[],
        parameters:{},
        viewOriginalText:null, viewExpandedText:null, tableType:MANAGED_TABLE
    ),

以上就是一个Table的几乎所有静态元信息。

## InputFormat

TextFile

SequenceFile

RCFile

+ 列式存储
+ bin/hive --service rcfilecat xxx/xxx

实现一个Inputfomat

    DualInputFormat implements InputFormat {
        InputSplit[] getSplits(JobConf, int)
        RecordReader<Text,Text> getRecordReader(InputSplit, JobConf)
    }

    DualInputSplit implements InputSplit {
        getLength()
        getLocations()
        write()
        readFields()
    }

    DualRecordReader implements RecordReader<Text,Text> {
        getPos()
        getProcess()
        createKey()
        createValue();
        next(Text k, Text v)
    }

##Serde

Hive使用InputFormat来读取一行行数据记录。这行记录传给Serde.deserialize()方法进行处理。

JSONSerDe

ArvoSerDe

ProtoBufSerDe

# 第16章 Hive的Thrift服务

CLI是一个胖客户端，使用起来还是有诸多问题的。比如说环境问题（Hadoop&Hive Clients，配置文件，Java环境等）。Thrift的方式，可以让多种语言的瘦客户端（java，cpp，php，python等）来通过写thrift rpc使用。

    cd $HIVE_HOME
    bin/hive --service hiveserver &

生产环境下使用Hive

clients ... HAProxy ... Servers ... metaStoreServer

## ThriftMetastore

    cd ~
    bin/hive --service metastore &

CLI中如此配置

    hive.metastore.uris


# 第17章 存储处理程序和NoSQL

StorageHandler:一个结合InputFormat、OutputFormat、SerDe和Hive需要使用的特定的代码，来将外部实体作为标准的Hive表进行处理的整体。这样可以屏蔽外部存储介质。（Hadoop、HBase、Cassandra、DynamoDB）

# 第18章 安全

Hadoop(0.20.205) 使用Kerberos安全认证。


# 第19章 锁

使用Zookeeper来做高可用的分布式协调功能。

打开锁：

    hive.zookeeper.quorum
    hive.support.concurrency

使用：
    
    SHOW LOCKS tablename EXTENDED;

    LOCK TABLE people EXCLUSIVE;
    UNLOCK TABLE people;

# 第20章 Hive和Oozie

Oozie是一列“动作”的有向无环图（DAG）。

Oozie提供的Action：

+ MapReduce
+ Shell
+ Java
+ Pig
+ Hive
+ DistCp
+ HiveServiceBAction

workflow.xml

# 第21章 Hive和AWS

EMR(弹性MapReduce) 是AWS的一部分。使用EMR可以按需主键一个由节点组成的集群。这些集群用Hadoop和Hive的安装和配置。用户可以运行Hive query，然后在完成所有任务后终止这个集群，然后用户付费。
# 第22章 HCatalog

用Hive的元数据，但是写MR来操作数据。UDW就是这么玩的。

# 第23章 案例研究
