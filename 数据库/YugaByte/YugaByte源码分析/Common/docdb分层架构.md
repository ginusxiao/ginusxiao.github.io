# 提纲
[toc]

## DocDB Sharding Layer
### Hash & range sharding
#### Hash sharding
![image](https://docs.yugabyte.com/images/architecture/tablet_hash_1.png)

YugaByte table的Hash sharding使用2字节的hash空间，从0x0000到0xFFFF，因此每一个table至多可以有64k个tablets。如果某个table只有16个tablets，则整个hash空间被划分为16个子的区间，每一个tablet对应一个子区间：[0x0000, 0x1000), [0x1000, 0x2000), … , [0xF000, 0xFFFF]。

![image](https://docs.yugabyte.com/images/architecture/tablet_hash.png)

#### Range sharding
![image](https://docs.yugabyte.com/images/architecture/tablet_range_1.png)

### Tablet splitting
#### 使用场景
- Range scans
    - 某些场景下可能不太好事先选择一个非常好的分割点，于是需要在后期进行调整
- Low-cardinality primary keys
    - 如果使用hash sharding，就可能无法充分利用整个集群中的节点资源
- Small tables that become very large
    - 起初的时候，table比较小，因此可能只使用了较少的shards，但是当table变得比较大的时候，可能会添加新的节点到集群，这样shard数目可能会少于集群中节点数目，于是就需要tablet split来均衡使用集群中的节点

#### tablet splitting的方案
DocDB支持以下机制来进行tablet split：
- Pre-splitting：在创建table的时候，可以指定将table拆分为一定数目的tablets
- Manual splitting：用户可以在集群运行过程中对tablet进行手动拆分
- Dynamic splitting：集群在运行过程中可以自动的根据一定的策略对tablet进行拆分
    - 现在还在开发过程中

### Colocated tables
对于应用中有很多小的table的情况下，如果每个table都拆分为至少一个tablet，则YugaByte中将会有非常多的tablets，因为每个tablet都需要CPU，disk和network开销，所以这将是非常没必要的。为了解决这种问题，YugaByte提供了Colocated tables支持，也就是说多个table共用一个tablet(colocation tablet)。


## DocDB Storage Layer
### 存储模型
![image](https://docs.yugabyte.com/images/architecture/cql_row_encoding.png)

在YugaByte的文档中提到，DocKey部分由以下3部分组成：16 bit hash，一个或者多个hash columns，0个或者多个range columns，那么如果一个table是基于range进行分区，**那么这里的16 bit hash和hash columns的作用是什么呢？难道是YugaByte为了支持类似Cassandra这样会同时指定Hash索引和范围索引的情况才这么做的吗？**

另外，**文档中关于DocKey的介绍和YugaByteDB代码中的注释不一致**，代码中关于DocKey的注释如下：
```
// ------------------------------------------------------------------------------------------------
// DocKey
// ------------------------------------------------------------------------------------------------

// A key that allows us to locate a document. This is the prefix of all RocksDB keys of records
// inside this document. A document key contains:
//   - An optional fixed-width hash prefix.
//   - A group of primitive values representing "hashed" components (this is what the hash is
//     computed based on, so this group is present/absent together with the hash).
//   - A group of "range" components suitable for doing ordered scans.
//
// The encoded representation of the key is as follows:
//   - Optional fixed-width hash prefix, followed by hashed components:
//     * The byte ValueType::kUInt16Hash, followed by two bytes of the hash prefix.
//     * Hashed components:
//       1. Each hash component consists of a type byte (ValueType) followed by the encoded
//          representation of the respective type (see PrimitiveValue's key encoding).
//       2. ValueType::kGroupEnd terminates the sequence.
//   - Range components are stored similarly to the hashed components:
//     1. Each range component consists of a type byte (ValueType) followed by the encoded
//        representation of the respective type (see PrimitiveValue's key encoding).
//     2. ValueType::kGroupEnd terminates the sequence.
```


## DocDB Replication Layer
### 集群内部复制
对于每一个Tablet，通过tablet-peers借助于raft consensus来实现复制。基于Raft的复制协议可以实现强一致性(strongly consistent)。

默认的，读写请求都由tablet leader来处理。但是YugaByte也支持从raft follower读取，这破坏了强一致性读，但是提供了时间轴一致性读(timeline consistency)。

### 跨集群复制
跨集群复制允许在不同的集群之间进行异步复制。

借助于raft consensus算法，YugaByte提供了跨多个(3个或者更多)数据中心的不同集群之间的同步复制，以此来实现更高的可用性和性能。但是许多用例可能不需要同步复制或者管理多个数据中心所带来的复杂性和运营成本，对于这些用例，YugaByte支持跨2个数据中心的部署方式，基于DocDB的CDC功能来实现异步复制。

#### 跨数据中心部署方式之Active-Passive模式
集群之间的复制是单方向的，从一个source cluster(也称为master cluster)到一个或者多个sink clusters(也称为slave clusters)，对于sink clusters，它是passive的，也就是说它不接受写请求。

这种模式多用于借助于sink cluster来实现低延迟的读，也用于disaster recovery。

#### 跨数据中心部署方式之Active-Active模式
集群之间的复制是双向的，所有集群都可以接收读请求或者写请求。针对某个集群的写请求会异步的复制给另一个集群，在复制过程中同时携带上更新的timestamp信息。如果在同一个时间窗内(time window)在2个集群上更新相同的key，那么将导致具有较大时间戳的写请求作为最终的结果，也就是提供的是最终一致性。

这种模式也被称为multi-master模式，从实现上来说，multi-master模式是基于2个单向的mster-slave模式构建的，因此要特别注意确保分配的时间戳能够保证last writer wins语义，同时要确保从其它集群复制过来的数据不再复制回去。 

#### 特性
- 更新满足时间轴一致性(timeline consistent)
- 支持单个key的事务，也支持多个key的事务
- 支持Active-Active模式
- sink cluster可以有多个，source cluster也可以有多个

#### 对应用开发的影响
因为跨2个数据中心的复制是通过复制WAL来实现的异步复制，应用开发应当遵循如下模式：
- 在Active-Active模式下，避免UNIQUE indexes / constraints
    - 因为是通过WAL来实现异步复制的，无法检查unique constraints
- 在Active-Active模式下，避免serial columns in primary key
    - 避免采用自增的主键，因为2个cluster可能会产生相同的主键
- 避免触发器
    - 因为是通过WAL来实现异步复制的，bypass了Query layer，因此触发器将不会被触发

### Read Replicas
DocDB扩展了Raft来支持read replicas(也称为observer nodes)，它不参与写过程，但是可以异步的获取到时间轴一致的数据副本。

Read replica cluster可以有不同于primary data cluster的Replication factor，甚至允许Read replica cluster的replication factor是偶数。

#### 写Read replicas
应用可以向read replica cluster发送写请求，但是这些写请求会被重定向给primary data cluster。

#### schema变更
无需特别考虑

#### Read replicas vs eventual consistency
Read replica提供的是timeline consistency，优于eventual consistency。

### Change data capture (CDC)
CDC确保数据的任何变更都被识别，捕获并且自动应用到其它的data repository或者被其它的应用消费。

#### 使用场景
- 面向微服务的架构
- 异步复制给远端系统
- 多数据中心

#### 架构
![image](https://docs.yugabyte.com/images/architecture/cdc-2dc/process-architecture.png)

## DocDB Transaction Layer
### Transactions overview
#### Time synchronization
使用的是Hybrid Logical Clocks。

#### MVCC
在MVCC这一节中特意提到了Explicit Locking：跟PostgreSQL一样，YugaByte提供了各种锁来控制并发访问，这些锁主要用于那些MVCC可能无法提供期望的行为的场景，应用就可以使用这些锁来实现并发控制。
```
Just as with PostgreSQL, YugabyteDB provides various lock modes to control concurrent access to data in tables. These modes can be used for application-controlled locking in situations where MVCC does not give the desired behavior.
```

### Explicit locking
#### Provisional records

YugaByte中会使用Provisional records，目的是为了冲突处理。Provisional records都关联了优先级，如果2个事务冲突，则低优先级的Provisional records将会被abort。

#### Optimistic concurrency control

乐观锁：在事务中的操作执行完之后才会检查该事务是否满足隔离性和完整性约束。如果检查发现2个事务冲突，则其中一个事务被abort，被abort的事务可能会立即重新执行，或者报错给用户。

在简单事务场景下，YugaByte采用乐观锁。

乐观锁的实现：为每一个事务分配一个随机的优先级，在事务冲突的情况下，低优先级的事务被abort。

#### Pessimistic concurrency control

悲观锁：如果事务中的任何操作违背了隔离性和完整性约束，则阻塞该事务，直到不再违反这些约束为止。

在用户显式的使用row-locks的情况下，YugaByte采用悲观锁。

悲观锁的实现：为当前正在悲观并发控制下执行的事务分配一个非常高的优先级，这样以来，其它的与之存在冲突的事务都被被abort。

#### Row-level locks

Row-level locks不影响查询，它们只是阻塞写那些已经被行锁锁住的行。

Row-level locks包括以下类型：
- FOR UPDATE
    - 会导致由SELECT语句检索到的行被锁定，就好像它们要被UPDATE一样，这可以 阻止它们被其他事务锁定、修改或者删除，一直到当前事务结束
    - 所有的DELETE语句和某些特定列上的UPDATE语句也会获取FOR UPDATE行锁
    - 它会阻塞其它事务在被锁定的行上执行UPDATE, DELETE, SELECT FOR UPDATE, SELECT FOR NO KEY UPDATE, SELECT FOR SHARE或者SELECT FOR KEY SHARE
- FOR NO KEY UPDATE
    - 行为与FOR UPDATE类似，不过获得的锁较弱：这种锁将不会阻塞尝试在相同行上获得SELECT FOR KEY SHARE行级锁
    - 任何不获取FOR UPDATE锁的UPDATE也会获得这种锁
- FOR SHARE
    - 行为与FOR NO KEY UPDATE类似，不过它在每个检索到的行上获得一个共享锁而不是排他锁
    - 它会阻塞其它事务在被锁定的行上执行UPDATE, DELETE, SELECT FOR UPDATE或者SELECT FOR NO KEY UPDATE
    - 它不会阻塞其它事务在被锁定的行上执行SELECT FOR SHARE或者SELECT FOR KEY SHARE
- FOR KEY SHARE
    - 行为与FOR SHARE类似，但是获得锁较弱：SELECT FOR NO KEY UPDATE则不被阻塞
    - 它会阻塞其它事务在被锁定的行上执行DELETE或者更改key的值的UPDATE操作(这里描述的可能不准确，原文是这么说的：blocks other transactions from performing DELETE or any UPDATE that changes the key values)，其它类型的UPDATE操作则不被阻塞
    - 它不会阻塞SELECT FOR NO KEY UPDATE, SELECT FOR SHARE, 或者SELECT FOR KEY SHARE

### Transaction isolation levels
 SQL-92标准定义了4中隔离级别：
 - SERIALIZABLE
 - REPEATABLE READ
 - READ COMMITTED
 - READ UNCOMMITTED

YugaByte支持2种隔离级别：
 - SNAPSHOT(也就是SQL中的REPEATABLE READ)
    - 事务中针对同一行的多次读取都获得一致的结果(snapshot)
    - 事务只有在如下情况下才会被成功提交：当前事务的更新和在当前事务的snapshot之后提交的其它事务之间不存在冲突
 - SERIALIZABLE
    - 确保事务执行结果就跟顺序调度执行的结果一样

YSQL同时支持SERIALIZABLE和SNAPSHOT隔离级别，而YCQL则只支持SNAPSHOT隔离级别。

为了支持SERIALIZABLE和SNAPSHOT这2种隔离级别，YugaByte内部提供下面3种锁类型：
- Snapshot isolation write lock
    - snapshot isolation的事务在write的时候使用
- Serializable read lock
    - serializable isolation的事务在read-modify-write使用
- Serializable write lock
    - serializable isolation的事务在write的时候使用
    - snapshot isolation的事务在pure-write的时候使用
    
YugaByte会进一步区分是在Document上还是SubDocument上获取锁，或者说是在Row上还是在Column上获取锁。如果是在Row上获取锁，则称之为weak lock，而在Column上获取锁，则称之为strong lock。因此Snapshot isolation write lock又会进一步分为Strong Snapshot isolation write lock和Weak Snapshot isolation write lock，Serializable read lock又会进一步分为Strong Serializable read lock和Weak Serializable read lock，Serializable write lock又会进一步分为Strong Serializable write lock和Weak Serializable write lock。

### Single-row ACID transactions
#### Reading the latest data from a recently elected leader
当发生了leader切换的情况下，新的leader会向raft log中追加一个no-op entry，在这些no-op entry被多数节点复制之前，该tablet是不可用的(无法读取最新的数据，也无法执行read-modify-write操作)，因为在no-op entry被多数节点复制之前，该新的leader无法确认所有之前已经被raft committed的entries已经被应用到Rocksdb中。

#### Leader leases: reading the latest data in case of a network partition
Leader lease被用于处理类似下面的场景：
![image](https://docs.yugabyte.com/images/architecture/txn/leader_leases_network_partition.svg)

在上图中：
- 发生network partition之后，leader和follower分别在不同的网络分区
- 2个follower所在的网络分区选择出新的leader
- client发送写请求，新的leader所在的网络分区接收了该请求，并复制到2个tablet peer中
- client此时从旧的leader上读取到过期的数据

leader lease则避免了上述情况，它的工作流程如下：
- 每一次leader向follower发送消息的时候，leader都会在消息中附带一个leader lease interval(比如2s)。
- 当follower接收到消息的时候，它会读取它的本地时间戳(local monitonic time)，加上消息中附带的leader lease interval，作为当前的leader的lease，如果该follower成为了新的leader，则在旧的leader的lease过期之前，它不能处理读写请求。
- 当leader接收到follower的响应之后，它将已经被多数节点复制的leader lease interval作为它的lease，该lease将会被用于决定leader是否可以继续处理读写请求。
- 为了确保新的leader知晓旧的leader的lease过期时间，每一个tablet peer都会记录它所知道的关于旧的leader的lease过期时间。在tablet peer响应RequestForVote请求的时候，它会将旧的leader的lease的剩余时间附带在响应中，当选举出新的leader之后，新的leader会将从所有tablet peer收到的关于旧的leader的lease的剩余时间中找出那个最大的值，等待这个时间过去之后才能处理读写请求。

#### Safe timestamp assignment for a read request
每个读请求都会分配一个特殊的MVCC时间戳ht_read，并且允许其它写请求并发的更新相同的keys。比较关键的一点是，这些并发的写操作的MVCC时间戳不能小于ht_read。

为了正确设置ht_read，YugaByte引入了ht_lease_exp，顾名思义，就是基于hybrid timestamp的leader lease，跟前面的讲的leader lease类似，只不过前面讲的leader lease是基于本地时间戳的，而这里的ht_lease_exp是基于混合时间戳的。

假设当前已经被多数节点复制的ht_lease_exp是replicated_ht_lease_exp，则ht_read为如下几个值的最大值：
- 最后一个提交的raft entry的混合时间戳
- 下面2者2选1：
    - 如果raft log中有未提交的raft entry：min{第一个未提交的raft entry的混合时间戳 - Epson(Epson是混合时间戳中最小可能的时间偏移)，replicated_ht_lease_exp}
    - 如果raft log中没有未提交的raft entry：min{当前的混合时间戳，replicated_ht_lease_exp}

对于从单个tablet读取的情况，无需等待计算得到的ht_read成为安全的，因为它的计算过程已经保证了这一点。但是，对于跨多个tablet读取的情况下，需要等待ht_read在每个参与的tablets上都变成safe的，当然可能在某些tablet上，因为选择的ht_read太小，导致在read事务启动之前写入的数据的时间戳比ht_read要大，这样就需要重新启动read事务，并为之重新选择一个较大的ht_read，关于这一点，在“Read path”中有更详细的讲解。


### Distributed ACID transactions
#### Provisional records
为什么需要Provisional records：对于Single-shard事务来说，它会直接写DocDB，因为只涉及一个tablet，但是对于分布式事务(涉及多个tablet)来说，如果也直接写入DocDB的话，则从不同的tablet读取的时候可能会看到部分应用的的事务(的数据)，破坏了原子性，因此YugaByte向分布式事务中所涉及的所有的tablet写provisional record，之所以称为provisional record，是因为这些record在事务未被commit之前，是对用户不可见的。

与provisional records相对应的是regular records，provisional records和regular records共用同一个Rocksdb实例(但是代码中，是2个Rocksd实例?)，但是通过在key之前添加一个prefix，所有provisional records都会在regular records之前。

对于provisional records，包括3中Rocksdb key-value paris：
- Primary provisional records
    - 格式：DocumentKey, SubKey1, ..., SubKeyN, LockType, ProvisionalRecordHybridTime -> TxnId, Value
- Transaction metadata
    - 格式：TxnId -> StatusTabletId, IsolationLevel, Priority
- Reverse index
    - 格式：TxnId, HybridTime -> primary provisional record key
    - 通过Reverse index，可以查找到关于某个事物的所有的provisional records，这在清理committed或者aborted的事务的时候会被用到

#### Transaction status tracking
事务状态通过transaction status table来维护，对该transaction status table的更新也是一个single-shard事务。

transaction status包括如下的字段：
- Status
    - 包括pending，committed和aborted
- Commit hybrid timestamp
    - 当transaction被commit之后才被设置
    - 设置为transaction status tablet在向raft log追加“transaction committed”日志的那一时刻的混合时间戳
- List of IDs of participating tablets
    - 分布式事务的transaction manager所在的tablet server向transaction status tablet发送commit信息的时候会附带上这个列表，transaction status talet会通过这个列表来通知分布式事务中所有涉及的所有的tablets关于committed状态

### Transactional IO path
#### Write path
![image](https://docs.yugabyte.com/images/architecture/txn/distributed_txn_write_path.svg)

写流程如下：
1. Client requests transaction
    接收到该请求的tablet server成为transaction manager。

2. Create a transaction record
    分配一个transaction id，并为之选择一个transaction status table来保存transaction status record。transaction status table和transaction manager并不一定在同一个tablet server上。

3. Write provisional records
    向该事务所涉及的每个tablet分别写入provisional records，provisional records中会包括provisional hybrid timestamp，这个provisional hybrid timestamp并不是最终的commit timestamp，并且同一个事务中的不同tablet上的provisional hybrid timestamp也可能不同，但是最终的commit timestamp对同一个事务中的不同tablet来说都是一样的。

    在写provisional records的过程中，可能会跟其它事务发生冲突，这种情况下，事务将会abort然后重新执行，这对用户来说是透明的。
    
4. Commit the transaction
    当transaction manager已经在该事务所涉及的所有的tablets上都写完了provisional records，它就会发送一个commit消息给transaction status table。

    只有在当前事务没有因为冲突而abort的情况下，commit才会成功，commit操作的atomicity和durability是通过transaction status tablet上的raft group来保证的。
    
    一旦commit成功，所有的provisional records对于用户来说就是可见的了。

    transaction manager在向transaction status tablet发送commit消息时，会附带上当前事务所涉及的所有的tablet的ID列表。
    
5. Send the response back to client    
    YQL向client发送响应。

6. Asynchronously applying and cleaning up provisional records
    这一步由transaction status tablet负责。

    transaction status tablet知道所有参与该事务的tablets，它会向这些tablets逐一发送cleanup请求。所有的参与该事务的tablets在raft log中记录一个特殊的apply操作，其中包含transaction id和commit timestamp信息，一旦该apply操作被raft group中的多数节点接受，则tablet会移除provisional records，同时写入regular records。
    
    一旦所有参与该事务的tablets都成功的执行了apply操作，则transaction status就可以从transaction status table中删除了。删除操作是通过在transaction status tablet的raft中记录一个"applied everywhere"日志，该事务相关的raft日志则是在正常的raft log gc过程中被移除。

#### Read path
读操作无需加锁，它依赖于MVCC timestamp来读取关于一个一致性快照(consistent snapshot)的数据。

对于从多个不同的tablet读取的情况，必须确保读取到的数据是关于整个数据库(database)的**最近的一致性快照(recent consistent snapshot of the database)**。下面进一步阐述recent snapshot和consistent snapshot：
- Consistent snapshot. 该快照要么包含一个事务的全部数据，要不不包含该事务的任何数据，但是不能出现只包含某个事务的部分数据的情况，YugaByte通过为读分配一个特定的混合时间戳(ht_read)，并忽略任何拥有比ht_read大的混合时间戳的记录。
- Recent snapshot. 该快照中必须包含最新的可见的数据。YugaByte通过如下方式来确保读取到的是最新的数据：如果发现选择的ht_read太小了，也就是说，存在某些数据是在read之前写入的，但是这些数据具有比ht_read更大的混合时间戳，则重新执行该read操作。

![image](https://docs.yugabyte.com/images/architecture/txn/distributed_txn_read_path.svg)


读流程如下：
1. Client's request handling and read transaction initialization
    YQL接收到来自于client的请求，并且发现该请求会涉及到从多个tablet读取的情况，则启动一个只读事务。并为之选择一个ht_read，通常选择transaction manager的当前混合时间戳或者其中一个参与此次事务的tablet的safe time(关于safe time，请参考Safe timestamp assignment for a read request)。同时，还会选择一个称为global_limit的时间戳，它是通过physical_time + max_clock_skew(max_clock_skew是一个全局配置的关于YugaByte集群中各节点之间clock skew的最大值)计算得来，用于判断一个记录是否**一定**是在read之后写的。

2. Read from all tablets at the chosen hybrid time
    Transaction manager向事务所涉及的所有的tablets发送读请求，每个tablet都等待ht_read在它之上成为safe time之后，从它本地的DocDB中读取数据。

    当tablet读取到一条相关的记录，对应的时间戳是ht_record，则执行如下判断：
    - 如果ht_record <= ht_read，则该记录可以包含在读取结果中
    - 如果ht_record > definitely_future_ht，则该记录不能包含在读取结果中，definitely_future_ht被定义为这样一个时间戳：所有具有比它更大的时间戳的记录一定是在本次的read事务启动之后写入的，关于definitely_future_ht的计算后面会讲解
    - 如果ht_read < ht_record ≤ definitely_future_ht，则无法确定该记录是在本次的read事务启动之前写入的还是之后写入的，因此，需要重新启动本次read，并且重新设置ht_read为ht_record
    
    当参与该事务的tablet发送响应给transaction manager的时候，会向transaction manager发送该tablet的safe time，记做local_limit_of_tablet。definitely_future_ht是这样计算的：definitely_future_ht = min(global_limit, local_limit_of_tablet)，其中global_limit是通过physical_time + max_clock_skew(max_clock_skew是一个全局配置的关于YugaByte集群中各节点之间clock skew的最大值)计算得来。
        
3. Tablets query the transaction status
    在每个tablet读取过程中，可能会遇到provisional records，这时候无法确定transaction status，也无法确定commit timestamp，所以需要去查询transaction status tablet，如果事务已经committed，则这个provisional records将被当做regular records，commit timestamp就是transaction status tablet中记录的关于该事务的commit timestamp。    

4. Tablets respond to YQL
    发送响应给transaction manager所在的tablet server的YQL，响应中包含以下信息：
    - 是否需要restart
    - local_limit_of_tablet
    - 从该tablet读取到的值

5. YQL sends the response to the client
    一旦参与当前事务的所有的tablets都成功读取，则发送响应给client。
