# 提纲
[toc]

## 分布式架构
### 集群组件
MemSQL将集群分为两层，一层被称作是aggregators，另一层被称作是leaves，如果要对集群进行横向扩展的话，直接添加leaves到集群即可。

其中aggregators节点负责:元数据处理，查询路由，聚合结果。集群中可以有1个或者多个aggregators，其中一个叫做master aggregator，专门负责元数据和集群监控相关处理。

Leaves节点负责：存储数据，执行由aggregator分发而来的SQL查询。

用户查询通常都首先由aggregators处理，而且任何一个aggregator都可以处理任何一个查询。

### 数据分布
在创建数据库的时候，MemSQL会将数据库切分为多个分区，分区数目可以通过PARTITIONS=x进行设定，默认的，每个数据库的分区数目是leaves节点数目的8倍。

分区在这些leaves节点间均匀分布。

MemSQL根据每一行数据的primary key对应的hash值，计算他所属的分区。

MemSQL在每个分区内部维护二级索引，Secondary indexes are managed within each partition and currently MemSQL enforces that unique indexes are prefixed by the primary key. 这一句话中and后面的部分是啥意思？

### Aggregator
Aggregator主要负责：
- querying the leaves
- aggregating the results
- returning the results to the client

The Master Aggregator is a specialized aggregator responsible for cluster monitoring and failover.

## Code Generation
Code Generation技术对于MemSQL查询性能至关重要。MemSQL中内嵌了一个编译器以产生高效的机器码。传统数据库是采用的基于翻译的执行模型(interpreter-based execution model)，而MemSQL中默认的，在第一次执行某个查询的时候，会首先翻译，然后在异步的在后台进行编译以方便后续使用。

**但是还是不太明白Code Generation技术和其它数据库中的PreparedStatement的区别是什么？**但是根据下面的一句话：
```
In many other databases, server-side prepared statements provide performance advantages, but MemSQL already compiles and caches query plans internally, so MemSQL does not need server-side prepared statements to get most of those performance advantages.
```
也就是说，其它数据库中都是用PreparedStatement来提升查询性能，但是MemSQL已经采用了Code Generation技术，所以不再需要PreparedStatement了，**但是不知道这两种技术孰优孰劣**?

## Indexes
### Skip List Indexes
[The Story Behind MemSQL’s Skiplist Indexes](https://www.memsql.com/blog/what-is-skiplist-why-skiplist-index-for-memsql/)

### Columnstore Indexes
Columnstore Indexes用于高效的从硬盘获取大量的数据。目前，如果在某个table上指定了ColumnStore Indexes，则该table就会使用列存。也就是说，无法同时支持ColumnStore Indexes和RowStore Indexes。

### Hash Table Indexes
Hash Indexes主要用于快速而精确的匹配的场景。

可以使用如下命令创建一个Hash Index：
```
CREATE TABLE t(a int, b int, INDEX(a, b) USING HASH);
```

### Indexes choosing
MemSQL在执行查询的时候，可以动态的决定选择哪种索引(注意：这里不是选择是采用rowstore的索引还是columnstore的索引)进行查询。主要原理如下：
```
Instead of collecting statistics and building histograms, MemSQL can compute these statistics for a given query on-demand by inspecting its indexes. If a query can match more than one index, MemSQL compiles an execution plan for each choice, along with the necessary expression logic to cheaply analyze and evaluate which plan to choose at runtime. This process eliminates the need to manually recompute statistics on indexes.
```

### Index Hints
MemSQL支持索引暗示，相关语法如下：
```
    tbl_name [index_hint]

    index_hint:
        USE {INDEX | KEY} (index_list)
      | IGNORE {INDEX | KEY} (index_list)
      | FORCE {INDEX | KEY} (index_list)

    index_list:
        index_name [, index_name] ...
```
USE和FORCE都是强制使用给定的索引进行查询，MemSQL中USE和FORCE没有区别。

IGNORE禁用某个索引进行查询。

## RowStore
RowStore是MemSQL中的默认的存储格式，采用如下方式创建RowStore：
```
CREATE TABLE products (
     ProductId INT,
     Color VARCHAR(10),
     Price INT,
     dt DATETIME,
     KEY (Price),
     KEY (Color),
     PRIMARY KEY(ProductId),
     SHARD KEY (ProductId)
);
```

Primary keys必须包括shard keys中指定的所有的列。如果没有指定shard keys，则自动基于primary key进行分区。

### Rowstore persistence
RowStore的持久化是通过WAL log加上周期性的对内存中的数据执行snapshot实现的。


## Columnstore
### 创建一个ColumnStore table
如果在创建table的时候，指定了Clustered ColumnStore key，则这个表就会在物理上使用ColumnStore。如下：
```
CREATE TABLE products (
     ProductId INT,
     Color VARCHAR(10),
     Price INT,
     Qty INT,
     KEY (`Price`) USING CLUSTERED COLUMNSTORE,
     SHARD KEY (`ProductId`)
);
```

目前，对于MemSQL的一个table，只能指定一个Clustered ColumnStore key，如果不想指定Clustered ColumnStore key，则可以采用如下方式指定一个空的key：KEY() USING CLUSTERED COLUMNSTORE。

通过指定SHARD KEY，则可以控制数据分区，如果不关心数据分布，则可以不指定SHARD KEY(使用SHARD KEY())。

### 到底是选择ColumnStore还是RowStore
In-Memory Rowstore|Flash, SSD, or Disk-based Columnstore
---|---
Operational/transactional workloads|Analytical workloads
Fast inserts and updates over a small or large number of rows|Fast inserts over a small or large number of rows
Random seek performance|Fast aggregations and table scans
Updates/deletes are frequent|Updates/deletes are rare
N/A|Compression

### ColomunStore中的一些概念
- ColumnStore key column(s): 在创建table的时候通过KEY (`XXX`) USING CLUSTERED COLUMNSTORE设置的就是ColumnStore key，ColumnStore中的数据就是按照它来有序存放的。

- Row Segment: Row Segment代表ColumnStore中逻辑上的一系列的rows，MemSQL为每一个Row Segment在内存中维护它的元数据，比如该Row Segment中的行数和一个用于跟踪哪些行已经被删除的bitmask。

- Column Segment：每一个Row Segment中包含table中的每一个列对应的Column Segment。Column Segment是ColumnStore中的存储单元，它当中包含某个Row Segment中关于某个列的所有的数据。MemSQL为每一个Column Segment在内存中维护它的元数据，比如该Column Segment中的最小值和最大值等。

- Sorted row segment group：表示一系列的Row Segments，这些Row Segments按照ColumnStore key column(s)进行排序。

- 关于这些概念的示例，见[这里](https://docs.memsql.com/v7.0/concepts/columnstore/)。

### 写数据
MemSQL ColumnStore支持非常快速的小批量的写(比如单行插入)，其实实现上，它是通过先将新写的行数据保存到面向行的跳表中，然后再flush到ColumnStore中，数据一旦保存到面向行的跳表中，就对后续的读可见了(原文是这么说的：MemSQL supports very fast, small-batch writes (such as single row inserts) directly into columnstore tables. This is implemented by storing newly written rows in a row-oriented skiplist before flushing them to the column-oriented format.)。

对于Insert，Delete和Update等操作如下：
- insert：插入操作有可能直接写入Column Store中，也有可能先写入面向行的跳表中。到底写到哪里是由引擎基于插入大小和ColumnStore当前的状态采用启发式算法自动决定的。对于较大的插入操作，则直接写入ColumnStore中，这可能需要先从ColumnStore中加载一些数据，然后和新的数据进行合并排序，最后写入到一个新的Row Segment。默认的，如果INSERT或者LOAD DATA语句在某个PARTITION中写入超过16MB的数据，就会这样做。
- Delete：删除一行，则会更新Segment元数据，而Row Segment中的数据在保持不变。如果一个Segment中所有的行都被删除了，则该Segment被移除。
- Update：在ColumnStore中update操作被当做一个事务性的delete + insert。

### 管理Segments
对于基于ColumnStore的table来说，如果table中的所有的行都是有序的，则对于查询性能是最友好的，但是事实上在一个持续写的场景下，维护这种顺序是很难的。

MemSQL采用了一种高级算法来尽可能的接近于排好序的，它会启动一个称作是background merger的进程来持续的在后台运行来确保尽可能接近于排好序的。

### 预取
如果一个查询涉及多个segments，则可能会利用预取机制来在处理当前segment的数据的过程中事先获取下一个segment的数据。

### Persistent Computed Columns
一个Computed Column是一个由查询语句产生的新的Column，MemSQL支持用户创建一个持久化的computed columns。如：
```
memsql> CREATE TABLE t (a INT PRIMARY KEY, b AS a + 1 PERSISTED INT);
```

## Replication and Durability
MemSQL在如下情况会执行replicate：
- 以high availability, redundancy-2 mode运行MemSQL集群
    - 可以在CREATE DATABASE, RESTORE DATABASE和ALTER DATABASE命令中指定master到slave的replication是synchronous replication还是asynchronous replication，默认使用的是synchronous replication
- 使用REPLICATE DATABASE命令
    - 只能使用asynchronous replication
    
### Synchronous and Asynchronous Replication
- 相较于asynchronous replication，有约10%到20%的性能损失
- 但是提供更强的数据一致性
- 如果使用synchronous replication的情况下，slave partition未能在5s之内返回响应，则slave partition切换到asynchronous replication

### Synchronous and Asynchronous Durability
- 在使用 CREATE DATABASE, RESTORE DATABASE, and REPLICATE DATABASE命令时，可以指定是使用synchronous durability还是asynchronous durability
- Synchronous durability将确保在事务成功提交之前成功写log的硬盘

### Using Synchronous Replication and Synchronous Durability Together
Synchronous Replication和Synchronous Durability可以一起使用，事务只在执行完如下步骤之后才被认为成功提交：
- 在Master的内存中更新
- 复制更新到slave
- master上的更新被写入log
- slave上的更新被写入log

### Using Asynchronous Replication with Synchronous Durability is not Allowed
Asynchronous Replication和Synchronous Durability不允许同时使用。


