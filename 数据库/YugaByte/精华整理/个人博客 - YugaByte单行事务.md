# 提纲
[toc]

# 简介
事务和一致性是RDBMS的基本要求，YugaByte作为兼容PostgreSQL的NewSQL也不例外，它支持只涉及单一行的单行事务，也支持跨行、跨表、跨节点的分布式事务。

本文主要讲述YugaByte中单行事务的实现。

# 单行事务的认定
当操作满足以下条件时，该操作被视为单行事务：
- 不在显式的事务语句块(BEGIN TRANSACTION; ...; END)中；
- 这是一个修改类型的操作，比如insert，update和delete等；
- 只涉及一个table的修改；
- 被修改的table上没有触发器；
- 被修改的table上没有二级索引；
- Insert语句中values子句只包含单一行的value；

# 单行事务相关的SQL语句
YugaByte单行事务是指那些只更新某一行数据的操作，以下SQL语句对应的都是单行事务：
```
# 插入某一行
INSERT INTO table (columns) VALUES (values);

# 某一行上的upsert操作
INSERT INTO table (columns) VALUES (values) ON CONFLICT DO UPDATE SET <values>;

# 更新某一行，同时指定了所有primary keys
UPDATE table SET column = <new value> WHERE <all primary key values are specified>;

# 指定了所有primary keys的删除操作
DELETE FROM table WHERE <all primary key values are specified>;
```

# 单行事务执行逻辑
![单行事务执行逻辑](https://note.youdao.com/yws/public/resource/f772280878921027a29ed6760b0a3ad7/xmlnote/A7720A07C8DA49EEA9D7060EB248E5C6/111789)
单行事务的执行逻辑如上图所示，主要包括2部分，一部分是SQL Layer，主要复用PostgreSQL的代码，负责对client端的查询请求进行解析、生成查询树、生成执行计划等；另一部分是Storage Layer，负责数据的存储和访问。本文主要关注Storage Layer中单行事务的执行逻辑，包括以下步骤：
1) Tablet leader接收到来自于SQL layer的请求；
2) 向Lock Manager申请锁；
3) 借助Conflict Resolver进行冲突检测和冲突处理；
4) 如果有必要(比如，读-写-更新类型的操作，条件更新操作等)，则从RocksDB中读取数据；
5) 将单行事务中所有数据更新提交给raft；
6) 在raft组内部进行raft复制；
7) Raft复制成功之后，将单行事务中所有数据更新写入rocksdb；
8) 向Lock Manager申请释放锁；
9) 发送响应给SQL layer；

# 并发控制
并发控制用于确保多个事务并发进行的同时不违背数据的完整性(比如隔离级别)约束。事务并发控制协议包括2PL(Two Phase Locking)，TO(Timestamp Ordering)，OCC(Optimistic Concurrency Control)和MVCC(Multi Version Concurrency Control)等。其中2PL还进一步进化出了S2PL(Strict Two Phase Locking)和SS2PL(Strong Strict Two Phase Locking)。关于MVCC并发控制协议，还必须指出的是，它必须和其它并发控制协议合作，才能提供完整的并发控制能力，比如MV-2PL，MV-TO和MV-OCC。

YugaByte采用了MVCC-SS2PL来实现并发控制。

## MVCC
YugaByte使用RocksDB作为它的底层存储，对于一行数据，会按照列拆分为多个KVs分别保存在RocksDB中，每一个列对应一个KV。同时，YugaByte在每个列上实现了MVCC。为了实现列级别的MVCC，YugaByte将保存在RockDB中的key定义为RowKey + ColumnKey + HybridTimestamp，其中RowKey表示列所属的行的主键，ColumnKey表示列的ID，HybridTimestamp表示列的版本信息，其中HybridTimestamp是基于一种综合了本地时钟(local clock)和兰伯特时钟(Lamport clock)的优点的分布式混合时间戳算法生成的。

下面举个例子来说明YugaByte的多版本在RocksDB中是如何表示的。

创建一个表，并且向其中插入一行：
```
CREATE TABLE msgs (user_id text,
                   msg_id int,
                   msg text,
                   read boolean,
                   PRIMARY KEY ((user_id), msg_id));

T1: INSERT INTO msgs (user_id, msg_id, msg) VALUES ('user1', 10, 'msg1', true);
```

此时，RocksDB中的条目将如下所示：
```
(hash1, 'user1', 10), msg_column_id, T1 -> 'msg1'
(hash1, 'user1', 10), read_column_id, T1 -> 'true'
```

更新某一列：
```
T2: UPDATE msgs
       SET msg = 'MSG1'
     WHERE user_id = 'user1', msg_id = 10
```

此时，RocksDB中的条目将如下所示：
```
(hash1, 'user1', 10), msg_column_id, T1 -> 'msg1'
(hash1, 'user1', 10), read_column_id, T1 -> 'true'
(hash1, 'user1', 10), msg_column_id, T2 -> 'MSG1'
```

通过MVCC，每个事务的更新操作都可以更新自己的版本，而每个事务的读取操作也可以读取事务开始时的特定版本的数据，从而确保了读不阻塞写，写不阻塞读。

## SS2PL(Strong Strict Two Phase Locking)
这里首先以2个并发事务的一种调度方案介绍一下SS2PL协议：
![SS2PL](https://note.youdao.com/yws/public/resource/f772280878921027a29ed6760b0a3ad7/xmlnote/F2B571CA810843EEA478E75981C3BFBA/112321)

YugaByte借助于SS2PL协议来解决写写冲突问题。

但是YugaByte在锁这一块进行了改进，不再是独占锁和共享锁两种类型的锁，而是提供了4种类型的锁，分别是WeakRead lock，WeakWrite lock，StrongRead lock和StrongWrite lock。这些锁对应的锁冲突矩阵如下(N表示不冲突，Y表示冲突)：
Lock Type|WeakRead lock|WeakWrite lock|StrongRead lock|StrongWrite lock
---|---|---|---|---
WeakRead lock| N | N | N | Y
WeakWrite lock| N | N | Y | N
StrongRead lock| N | Y | N | Y
StrongWrite lock| Y | N | Y | N

事实上，YugaByte中对锁的使用，可能是这4种锁中的一种，也可能是这4种锁的任意组合，对于由2种或者2种以上的锁组合而成的锁，被称之为组合锁，后面组合锁也会被简称为锁，所以也可以认为YugaByte中有15种锁。

看了这个锁冲突矩阵之后，大家可能会有疑问，怎么StrongWrite lock和StrongWrite lock之间不存在冲突啊，原因在后面会讲到，这里只简单介绍下，YugaByte中申请锁的时候，大多数情况下都申请到的是组合锁。

YugaByte会在2个地方管理这些锁，一个是在Tablet leader的内存中通过LockManager进行管理，另一个是在临时记录(分布式事务的更新在没有commit之前都保存在临时记录中)中保存，它会随着临时记录被复制到raft group中。

## Lock Manager
YugaByte通过LockManager来管理每个分区上都有哪些对象正在被事务性访问，以及在这些对象上都分别持有哪些类型的锁。

YugaByte在事务执行之初，会首先进入加锁阶段，执行如下：
- 计算当前事务需要在哪些对象上加锁
- 检查在需要加锁的对象上分别加哪种类型的锁
- 尝试向LockManager申请加锁，对于当前事务的每一个加锁请求处理如下：
    - 如果其它事务已经持有相关对象的锁，且当前事务申请的关于该对象的锁和其它事务已持有的锁冲突，则当前事务必须等待
        - 如果等待超时，则当前事务申请锁失败，事务重试
        - 如果成功申请到锁，则：
            - 向Lock Manager中添加当前事务成功申请到该锁的记录
            - 等待，直到当前事务成功申请到所有的锁，当前事务继续后续步骤
    - 如果没有其它事务持有相关对象的锁，或者当前事务申请的关于该对象的锁和其它事务已持有的锁不冲突，则：
        - 向Lock Manager中添加当前事务成功申请到该锁的记录
        - 等待，直到当前事务成功申请到所有的锁，当前事务继续后续步骤
    
下面我们将逐一分析加锁过程。

### 计算当前事务需要在哪些对象上加锁
当前YugaByte会直接在RowKey对象上加锁。前面讲到YugaByte会将一行数据按列拆分，每个列以单独的KV存储在RocksDB中，一行中的某列在RocksDB中的key是由RowKey + ColumnKey + HybridTimestamp，RowKey就代表列所在的行，所以是对整个行加锁，也就是说YugaByte单行事务中每个操作会加一个行级别的锁。

### 检查在需要加锁的对象上分别加哪种类型的锁
前面已经确定了要在RowKey上加锁，接下来就是确定要加哪种类型的锁了：
- 如果是insert/update/delete操作，则加StrongRead lock + StrongWrite lock组合锁
    - 之所以加StrongRead lock，是因为insert/update/delete操作需要读取/检查主键索引
- 如果是upsert操作，则加StrongWrite lock

### 向LockManager申请加锁
前面已经确定了要在RowKey上加锁，且要加的锁类型也确定了，接下来就去LockManager申请锁。

LockManager中记录了以下信息：
- 哪些对象上已经加了锁
- 每个对象上都加了哪种类型的锁/锁组合
- 每个对象上所加的每种类型的锁都有多少持有者持有它
- 每个对象上有多少事务因为申请不到锁而被阻塞

假设要在对象obj上申请加类型为type的锁，则流程如下：
- 检查LockManager中是否存在关于obj的锁记录
- 如果不存在，则直接将对象obj和锁type添加到LockManager中，并返回加锁成功
- 如果存在，则：
    - 检查对象obj上已经持有的锁的类型和待申请的锁的类型之间是否存在冲突
        - 如果不冲突，则：
            - 将待申请的锁类型添加到对象上所加的锁集合中
            - 增加该对象上该类型的锁的持有者数目
            - 返回加锁成功
        - 否则：
            - 增加该对象上因为申请不到锁而被阻塞的事务的数目
            - 等待直到超时，或者因为该对象上关于该待申请的锁的持有者数目降为0而被唤醒

## Conflict Resolver
当前事务成功申请到锁之后，就会进入冲突检测阶段，如果检测到冲突，则还需要进行冲突处理。冲突检测和冲突处理的工作是由Conflict Resolver负责的。

冲突检测主要是检测当前事务和YugaByte中尚未提交的临时记录之间是否存在冲突。YugaByte中的临时记录是指分布式事务中尚未提交的记录，因为分布事务使用2PC(Two Phase Commit)协议，所以在prepare阶段数据会以临时记录的形式写入RocksDB中，然后再commit阶段再将临时记录转化为正式记录，正式记录就是我们前面提到的RowKey + ColumnKey + HybridTimestamp到value的映射。临时记录和正式记录都是持久化保存在RocksDB中的，两者的格式有所差异，临时记录的key在正式记录的key的基础上多了锁类型字段，临时记录的value在正式记录的value的基础上多了TransactionID字段，也就是说临时记录是RowKey + ColumnKey + LockType + HybridTimestamp到TransactionID + value的映射。单行事务不存在临时记录，但是单行事务和分布式事务是可能并发的，所以必须检查它们之间是否存在冲突。

Conflict Resolver的主要逻辑如下：
- 计算当前事务需要在哪些对象上加锁
- 计算在需要加锁的对象上分别加哪种类型的锁
- 检测当前事务在每个对象上需要加的锁和临时记录中已经持有的锁之间是否存在冲突，对于不存在冲突的事务直接略过，而对于冲突的事务，则会添加到当前事务的冲突事务列表中
- 如果当前事务的冲突事务列表为空，则直接返回，否则进行冲突处理

注意，在冲突检测阶段，单行事务是不会真正在对象上加锁的(这跟前面在LockManager中申请锁是不同的)。

### 冲突检测
#### 计算当前事务需要在哪些对象上加锁
在冲突检测阶段YugaByte会在整个表，RowKey和RowKey + ColumnKey这3种对象上分别加锁，在整个表上加锁主要是为了确保事务执行期间表不被删除。

#### 检查在需要加锁的对象上分别加哪种类型的锁
对于RowKey + ColumnKey这一对象上加锁情况如下：
- 如果是insert/update/delete操作，则加StrongRead lock + StrongWrite lock组合锁
    - 之所以加StrongRead lock，是因为insert/update/delete操作需要读取/检查主键索引
- 如果是upsert操作，则加StrongWrite lock

对于整个表和RowKey这两个对象上加锁情况如下：
- 如果是insert/update/delete操作，则加WeakRead lock + WeakWrite lock组合锁
    - 之所以加StrongRead lock，是因为insert/update/delete操作需要读取/检查主键索引
- 如果是upsert操作，则加WeakWrite lock

#### 检测当前事务在每个对象上需要加的锁和临时记录中已经持有的锁之间是否存在冲突
假设在对象(记为obj，表对象除外，可能是RowKey，也可能是RowKey + ColumnKey)上加锁(锁类型记为type)，则冲突检测过程如下：
- 获取所有以obj为前缀，且小于obj + “\xff”的临时记录，添加到集合records中
- 对records中的每个临时记录分别处理：
    - 如果记录已经持有的锁的类型和当前对象要加的锁之间不存在冲突，则略过当前记录
    - 否则，从临时记录中获取对应的TransactionID，并添加到当前事务的冲突事务列表中

### 冲突处理
至此，所有与当前事务冲突的事务已经保存在当前事务的冲突事务列表中，接下来就与冲突事务列表中的每个事务逐一处理冲突问题：
- 从冲突事务列表中移除所有已经在本地将临时记录转换为正式记录的事务
- 获取冲突事务列表中剩余的所有事务的状态，事务可能处于COMMITTED、PENDING或者ABORTED状态
- 从冲突事务列表中移除所有处于COMMITTED或者ABORTED状态的事务
- 检查冲突事务列表中是否还存在冲突的事务，如果不存在，则结束冲突处理
- 对冲突事务列表中剩余的所有的事务，abort之

# 参考
https://zhuanlan.zhihu.com/p/127274032

https://zhuanlan.zhihu.com/p/42007051
