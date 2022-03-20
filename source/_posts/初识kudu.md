---
title: 初识kudu
date: 2021-10-13 16:46:55
keywords: 'kudu'
tags:
- kudu
categories:
- 大数据组件
- kudu
---
##  1.1、kudu是什么
简单来说:dudu是一个与hbase类似的列式存储分布式数据库。
官方给kudu的定位是:在更新更及时的基础上实现更快的数据分析

##  1.2、为什么需要kudu
###  1.2.1、hdfs与hbase数据存储的缺点

目前数据存储有了HDFS与hbase，为什么还要额外的弄一个kudu呢。
HDFS:使用列式存储格式Apache Parquet，Apache ORC，适合离线分析，不支持单条纪录级别的update操作，随机读写性能差。
HBASE:可以进行高效随机读写，却并不适用于基于SQL的数据分析方向，大批量数据获取时的性能较差。
正因为HDFS与HBASE有上面这些缺点，KUDU较好的解决了HDFS与HBASE的这些缺点，它不及HDFS批处理快，也不及HBase随机读写能力强，但是反过来它比HBase批处理快（适用于OLAP的分析场景），而且比HDFS随机读写能力强（适用于实时写入或者更新的场景），这就是它能解决的问题。

#  2、架构介绍
##  2.1、基本架构

<img src="https://img-blog.csdnimg.cn/20190422091319780.png" referrerpolicy="no-referrer">

##  2.1.1、概念

 Table（表）：一张table是数据存储在kudu的位置。Table具有schema和全局有序的primary key(主键)。Table被分为很多段，也就是tablets.
 Tablet (段)：一个tablet是一张table连续的segment，与其他数据存储引擎或关系型数据的partition相似。Tablet存在副本机制，其中一个副本为leader tablet。任何副本都可以对读取进行服务，并且写入时需要在所有副本对应的tablet server之间达成一致性。
 Tablet server：存储tablet和为tablet向client提供服务。对于给定的tablet，一个tablet server充当leader，其他tablet server充当该tablet的follower副本。只有leader服务写请求，leader与follower为每个服务提供读请求。
 Master：主要用来管理元数据(元数据存储在只有一个tablet的catalog table中)，即tablet与表的基本信息，监听tserver的状态
 Catalog Table: 元数据表，用来存储table(schema、locations、states)与tablet（现有的tablet列表，每个tablet及其副本所处tserver，tablet当前状态以及开始和结束键）的信息。

#  3、存储机制
##  3.1 存储结构全景图

<img src="https://img-blog.csdnimg.cn/20190422091359222.png" referrerpolicy="no-referrer">

##  3.2 存储结构解析

 一个Table包含多个Tablet，其中Tablet的数量是根据hash或者range进行设置
 一个Tablet中包含MetaData信息和多个RowSet信息
 一个Rowset中包含一个MemRowSet与0个或多个DiskRowset，其中MemRowSet存储insert的数据，一旦MemRowSet写满会flush到磁盘生成一个或多个DiskRowSet，此时MemRowSet清空。MemRowSet默认写满1G或者120s flush一次
(注意:memRowSet是行式存储，DiskRowSet是列式存储，MemRowSet基于primary key有序)。每隔tablet中会定期对一些diskrowset做compaction操作，目的是对多个diskRowSet进行重新排序，以此来使其更有序并减少diskRowSet的数量，同时在compaction的过程中慧慧resolve掉deltaStores当中的delete记录
 一个DiskRowSet包含baseData与DeltaStores两部分，其中baseData存储的数据看起来不可改变，DeltaStores中存储的是改变的数据
 DeltaStores包含一个DeltaMemStores和多个DeltaFile,其中DeltaMemStores放在内存，用来存储update与delete数据，一旦DeltaMemStores写满，会flush成DeltaFile。
当DeltaFile过多会影响查询性能，所以KUDU每隔一段时间会执行compaction操作，将其合并到baseData中，主要是resolve掉update数据。

#  4、kudu的工作机制
##  4.1 概述

1、kudu主要角色分为master与tserver
2、master主要负责:管理元数据信息，监听server，当server宕机后负责tablet的重分配
3、tserver主要负责tablet的存储与和数据的增删改查

##  4.2 内部实现原理图
## <img src="https://img-blog.csdnimg.cn/20190422091414844.png" referrerpolicy="no-referrer">

##  4.3 读流程
###  4.3.1 概述

客户端将要读取的数据信息发送给master，master对其进行一定的校验，比如表是否存在，字段是否存在。Master返回元数据信息给client，然后client与tserver建立连接，通过metaData找到数据所在的RowSet，首先加载内存里面的数据(MemRowSet与DeltMemStore),然后加载磁盘里面的数据，最后返回最终数据给client.

###  4.3.2 详细步骤图

 <img src="https://img-blog.csdnimg.cn/20190422091428943.png" referrerpolicy="no-referrer">

###  4.3.3 详细步骤解析

1、客户端master请求查询表指定数据
2、master对请求进行校验，校验表是否存在，schema中是否存在指定查询的字段，主键是否存在
3、master通过查询catalog Table返回表，将tablet对应的tserver信息、tserver状态等元数据信息返回给client
4、client与tserver建立连接，通过metaData找到primary key对应的RowSet。
5、首先加载RowSet内存中MemRowSet与DeltMemStore中的数据
6、然后加载磁盘中的数据，也就是DiskRowSet中的BaseData与DeltFile中的数据
7、返回数据给Client
8、继续4-7步骤，直到拿到所有数据返回给client



##  4.4、插入流程

###  4.4.1 概述

Client首先连接master，获取元数据信息。然后连接tserver，查找MemRowSet与DeltMemStore中是否存在相同primary key，如果存在，则报错;如果不存在，则将待插入的数据写入WAL日志，然后将数据写入MemRowSet。

###  4.4.2 详细步骤图

<img src="https://img-blog.csdnimg.cn/20190422091442719.png" referrerpolicy="no-referrer"> 

###  4.4.3 详细步骤解析

1、client向master请求预写表的元数据信息
2、master会进行一定的校验，表是否存在，字段是否存在等
3、如果master校验通过，则返回表的分区、tablet与其对应的tserver给client；如果校验失败则报错给client。
4、client根据master返回的元数据信息，将请求发送给tablet对应的tserver.
5、tserver首先会查询内存中MemRowSet与DeltMemStore中是否存在与待插入数据主键相同的数据，如果存在则报错
6、tserver会讲写请求预写到WAL日志，用来server宕机后的恢复操作
7、将数据写入内存中的MemRowSet中，一旦MemRowSet的大小达到1G或120s后，MemRowSet会flush成一个或DiskRowSet,用来将数据持久化
8、返回client数据处理完毕

##  4.5、数据更新流程

###  4.5.1 概述

Client首先向master请求元数据，然后根据元数据提供的tablet信息，连接tserver，根据数据所处位置的不同，有不同的操作:在内存MemRowSet中的数据，会将更新信息写入数据所在行的mutation链表中；在磁盘中的数据，会将更新信息写入DeltMemStore中。

###  4.5.2、详细步骤图

 <img src="https://img-blog.csdnimg.cn/20190422091450166.png" referrerpolicy="no-referrer">

###  4.5.3 详细步骤解析

1、client向master请求预更新表的元数据，首先master会校验表是否存在，字段是否存在，如果校验通过则会返回给client表的分区、tablet、tablet所在tserver信息
2、client向tserver发起更新请求
3、将更新操作预写如WAL日志，用来在server宕机后的数据恢复
4、根据tserver中待更新的数据所处位置的不同，有不同的处理方式:
如果数据在内存中，则从MemRowSet中找到数据所处的行，然后在改行的mutation链表中写入更新信息，在MemRowSet flush的时候，将更新合并到baseData中
如果数据在DiskRowSet中，则将更新信息写入DeltMemStore中，DeltMemStore达到一定大小后会flush成DeltFile。
5、更新完毕后返回消息给client。

