# 提纲
[toc]

## YugatByte和其它数据库的对比
### YugaByteDB和分布式SQL数据库对比
[YugaByteDB和分布式SQL数据库对比](https://docs.yugabyte.com/latest/comparisons/)

### YugaByteDB和NoSQL数据库对比
[YugaByteDB和NoSQL数据库对比](https://docs.yugabyte.com/latest/comparisons/)

### YugaByte和Ignite的对比
1. YugabyteDB的不同API之间是无法互通的[参考：[YugaByteDB Introduction](https://docs.yugabyte.com/latest/introduction/)]，也就是说，通过YCQL API存储的数据是无法被YSQL API访问的。
2. 

## YugaByteDB Architecture
[Distributed PostgreSQL on a Google Spanner Architecture – Storage Layer](https://blog.yugabyte.com/distributed-postgresql-on-a-google-spanner-architecture-storage-layer/)

[Distributed PostgreSQL on a Google Spanner Architecture – Query Layer](https://blog.yugabyte.com/distributed-postgresql-on-a-google-spanner-architecture-query-layer/)

### YugaByte's internal DocDB
[DocDB data model ](https://docs.yugabyte.com/latest/architecture/docdb/persistence/)

[Enhancing RocksDB for Speed & Scale](https://blog.yugabyte.com/enhancing-rocksdb-for-speed-scale/)

#### YugaByteDB YSQL table和DocDB table的映射关系
[Distributed PostgreSQL on a Google Spanner Architecture – Query Layer](https://blog.yugabyte.com/distributed-postgresql-on-a-google-spanner-architecture-query-layer/)

### YugaByteDB's internal RocksDB
[How We Built a High Performance Document Store on RocksDB?](https://blog.yugabyte.com/how-we-built-a-high-performance-document-store-on-rocksdb/)

### YugaByteDB针对RocksDB的增强
1. Scan Resistant Cache
以阻止长时间运行的scan操作带来的2个负面影响：污染RocksDB的block cache，将有用的热数据冲刷出去；

具体采用的机制是：采用类似于MySql和HBase中的机制，将LRU分为两部分，只有多次访问的数据才会被存放到multi-touch/hot portion缓存中去。

2. Block-based Splitting of Bloom/Index Data
在原始的RocksDB中，bloom filter和index相关的数据要么全部在内存中，要么全部不在内存中，在YugaByteDB中，所做的增强是：将bloom filter和index相关的数据变为多层的面向块的结构，以便这些数据可以跟普通的数据块一样，按需加载到block cache中。

3. Multiple Instances of RocksDB Per Node
在YugaByteDB中，每一个tablet都对应一个RocksDB实例。

4. Server-global Block Cache
DocDB采用跨所有RocksDB实例的共享的block cache。

5. Server-global Memstore Limit
RocksDB允许配置每个memstore的flush size，这并不实用，因为memstore的数目会随着用户创建table，或者tablet迁移而发生变化。而且，memstore flush size的设置也没那么容易（偏大或者偏小都有其问题）。为了避免这些，YugaByteDB采用了全局的memstore limit设置，当（server节点）中的memory store的大小达到预先设定的值，就会将memstore中最老的数据flush到持久存储中。

6. Separate Queues for Large & Small Compactions
RocksDB支持同一个compaction queue被多个compaction线程所访问，但是这可能导致某些大的compactions被调度而小的compactions被饿死的情况，YugaByteDB则支持多个compaction queue，并且基于输入的待合并文件的大小作为优先级，每个compaction queue对应多个compaction线程，而且这些compaction线程中的一部分专门用于小的compactions。

7. Double Journaling Avoidance in the Write Ahead Log (WAL)
DocDB采用raft consensus protocol来进行数据复制，数据更新已经记录在raft log中了，RocksDB中的WAL机制就显得多余了。因此，YugaByteDB禁用了RocksDB中的WAL。

8. Multi-Version Concurrency Control (MVCC)
YugaByteDB不采用RocksDB的sequence id来实现MVCC，取而代之，采用编码在key中的混合时间戳来实现MVCC。

## YugaByteDB transaction
[Yes We Can! Distributed ACID Transactions with High Performance](https://blog.yugabyte.com/yes-we-can-distributed-acid-transactions-with-high-performance/)

### 不同类型的transaction的实现方式
- Single-Row/Single-Shard transaction
采用per-shard consensus

- Distributed transaction
采用2PC

### 关于transaction manager
1. YugaByteDB中有多个transaction manager，每个节点上一个，作为tablet server进程的一部分来运行；
2. YugaByteDB transaction manager是无状态的，所以每个transaction可以被路由到任何一个节点上的transaction manager；
3. 虽然说每个transaction可以被路由到任何一个节点上的transaction manager，但是YugaByte Query Layer会试图将transaction调度到那个具有关于本次事务的最多的数据的节点上；

## YugaByteDB Secondary Index

[YugaByte DB 1.1 New Feature: Speeding Up Queries with Secondary Indexes](https://blog.yugabyte.com/yugabyte-db-1-1-new-feature-speeding-up-queries-with-secondary-indexes/)

[A Quick Guide to Secondary Indexes in YugaByte DB](https://blog.yugabyte.com/a-quick-guide-to-secondary-indexes-in-yugabyte-db-database/)



## YugaByteDB consistency
- Single-Row Linearizability
every operation appears to take place atomically and in some total linear order that is consistent with the real-time ordering of those operations.
- multi-key snapshot isolation(serializable isolation is in the works)


## YugaByteDB生态
### 和Apache Kafka集成
[yb-kafka-connector](https://github.com/yugabyte/yb-kafka-connector)

[5 Reasons Why Apache Kafka Needs a Distributed SQL Database](https://blog.yugabyte.com/5-reasons-why-apache-kafka-needs-a-distributed-sql-database/)

### 和Apache Spark集成

这是借助于spark cassandra connector来实现的。

[Integration with Apache Spark](https://docs.yugabyte.com/latest/develop/ecosystem-integrations/apache-spark/)

### 和JanusGraph集成

[Integration with JanusGraph](https://docs.yugabyte.com/latest/develop/ecosystem-integrations/janusgraph/)

YugaByteDB作为底层的数据库，JanusGraph中的图数据是可以存储在Cassandra中的，所以这里实际上使用的是YugaByteDB的YCQL接口？

### 和KairosDB集成

[Integration with KairosDB](https://docs.yugabyte.com/latest/develop/ecosystem-integrations/kairosdb/)

KairosDB是一个基于Cassandra的时序数据库，所以这里使用的是YugaByteDB的YCQL接口。

### 和Presto集成

[Integration with Presto](https://docs.yugabyte.com/latest/develop/ecosystem-integrations/presto/)

[Presto on YugaByte DB: Interactive OLAP SQL Queries Made Easy](https://blog.yugabyte.com/presto-on-yugabyte-db-interactive-olap-sql-queries-made-easy-facebook/)

### 和Metabase集成

[Integration with Metabase](https://docs.yugabyte.com/latest/develop/ecosystem-integrations/metabase/)

Metabase是一个易用的商业智能（Business Intelligence）工具。

## YugaByteDB与HTAP
[TiDB + TiFlash ： 朝着真 HTAP 平台演进](https://blog.csdn.net/TiDB_PingCAP/article/details/100201889)

TiDB本身是为TP而设计的，为了加强TiDB的AP能力，他们在17年发布了了TiSpark来直接访问TiKV，然而由于TiKV 毕竟是为 TP 场景设计的存储层，对于大批量数据的提取、分析能力有限，所以他们为 TiDB引入了新的 TiFlash 组件，它的使命是进一步增强 TiDB 的 AP 能力，使之成为一款真正意义上的 HTAP 数据库。

## YugaByteDB中提到的一些疑惑
1. The SQL insert statement may end up updating a single row or multiple rows. Although DocDB can handle both cases natively, these two cases are detected and handled differently to improve the performance of YSQL. Single row inserts are routed directly to the tablet leader that owns the primary key of that row. Inserts affecting multiple rows are sent to a global transaction manager which performs a distributed transaction. The single-row insert case is shown below.

那么对于updating multiple rows的情况，具有一个全局的transaction manager？这是否会在具有大量multiple rows updating的事务并发的情况下，存在性能问题？但是在YugaByteDB官方博客的其它地方提到如下：

YugabyteDB has a built-in transaction manager to manage the lifecycle of transactions. The transaction manager runs on every node (as a part of the tablet server process) in the cluster and automatically scales out as nodes are added.

从这里可以看出，YugaByteDB中实际上有多个Transaction Manager，每个节点上一个，作为tablet server进程的一部分来运行。

2. 为什么说YugaByteDB是多模数据库

首先明确一下多模数据库的定义：A Multi-model database is a database that can store, index and query data in more than one model. 

YugaByteDB支持document（通过兼容Cassandra的YCQL API支持JSON），支持KV（通过兼容Redis的YEDIS API），支持SQL（通过兼容PostgreSQL的YSQL API）。

需要说明的是，Cassandra Query Language对JSON的支持仅仅是支持在SELECT语句的输出中返回JSON格式的数据和在INSERT语句的输入中指定JSON格式的数据，但是YCQL则支持的是Native JSON，[YugaByte DB 1.1 New Feature: Document Data Modeling with the JSON Data Type](https://blog.yugabyte.com/yugabyte-db-1-1-new-feature-document-data-modeling-with-json-data-type/)。YugaByteDB借助于JSONB（JSON Better）来解析、存储和查询JSON数据。使用JSONB可以非常容易的在document内部搜索和获取内部属性（通过将所有的JSON keys排序之后存储）。当前，更新JSONB的某个列（属性）的数据依然需要read-modify-write操作，未来可能会支持增量更新。

## 关于内存计算选择YugaByteDB时的考虑
1. YugaByteDB对OLAP的支持相较于OLTP来说，弱很多，而我们的内存计算平台则更侧重于OLAP？
的确可以在OLTP系统上实现OLAP的功能，但是问题是：
- 一种存储格式很难同时兼顾两种访问模式，OLTP侧重于行存，而OLAP侧重于列存；
- 一套系统上两种工作负载相互干扰，一个较大的长查询可能对于OLTP来说是一个灾难；
- HTAP的本质是什么？如果一套系统既支持OLTP，又支持OLAP，但是你要么只能把它当做OLTP使用，要么只能把它当做OLAP使用，则我认为这不是HTAP，因为这样的话，部署两套系统即可？真实的HTAP，是为了解决部署两套系统所带来的的问题（比如部署复杂，运维复杂，数据分析的时效性等）；

2. YugabyteDB的不同API之间是无法互通的，也就是说，通过YCQL API存储的数据是无法被YSQL API访问的。原文的说法是：“The YugabyteDB APIs are isolated and independent from one another today. This means that the data inserted or managed by one API cannot be queried by the other API. Additionally, there is no common way to access the data across the APIs (external frameworks such as Presto can help for simple cases).”
- 因为YugaByteDB的不同API之间无法互通，现在YugaByte官网上所说的和Apache Spark的集成是利用YCQL（兼容Cassandra的API），而你用YSQL写入的数据也是无法直接利用Apache Spark进行分析的；
- TiDB采用Apache Spark来实现OLAP的方法则是TiSpark + TiKV，而不是TiSpark + TiDB；

3. 


