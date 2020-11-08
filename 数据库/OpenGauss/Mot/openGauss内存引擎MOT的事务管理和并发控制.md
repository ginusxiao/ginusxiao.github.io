# 提纲
[toc]

## 简介
本文以repeatable read隔离级别为例分析openGauss内存引擎插入行，查询行，更新行和删除行相关的逻辑。

对于插入行，更新行和删除行，主要关注以下逻辑：
```
TxnManager::StartTransaction
MOTBeginForeignModify
MOTIterateForeignScan
MOTExecForeignInsert/MOTExecForeignUpdate/MOTExecForeignDelete
MOTEndForeignModify
TxnManager::Commit
TxnManager::EndTransaction
```

对于查询行，主要关注以下逻辑：
```
TxnManager::StartTransaction
MOTIterateForeignScan
TxnManager::Commit
TxnManager::EndTransaction
```

TxnManager::StartTransaction用于启动事务其中会设置transaction id和隔离级别，并且设置transaction state为TxnState::TXN_START。

在TxnManager::Commit用于提交事务，其中则运行基于[SILO论文](https://dl.acm.org/doi/10.1145/2517349.2522713)实现的提交协议(commit protocol)。

TxnManager::EndTransaction用于结束事务，其中执行一些锁释放和资源清理工作。

MOTBeginForeignModify和MOTEndForeignModify是insert/update/delete的通用逻辑，分别用于在insert/update/delete之前执行一些必要的初始化操作和在insert/update/delete之后释放相关资源。

MOTIterateForeignScan是read/update/delete的通用逻辑，用于从外部数据源中获取一行数据，并通过TupleTableSlot返回这一行数据。

关于MOTBeginForeignModify，MOTIterateForeignScan和MOTEndForeignModify的更多介绍，请参考[这里](https://www.postgresql.org/docs/10/fdw-callbacks.html)

## 事务管理和并发控制所用到的主要数据结构
### InstItem - 一个索引插入操作相关的信息
数据成员 | 作用
---|---
Index* m_index |  待插入到哪个索引中
Row* m_row | 这是与哪行数据相关的索引插入操作，代码中主要通过m_row来获取对应的Table信息
MOT::Key* m_key | 对应的key
IndexOrder m_indexOrder | 指示这是一个主键索引还是二级索引

### TxnInsertAction - 管理当前事务中索引插入操作对应的insert请求，也称为Insert Manager
数据成员 | 作用
---|---
InsItem* m_insertSet | InstItem数组，每个索引插入操作对应的insert请求，都依次保存在这里
uint32_t m_insertSetSize | 有多少个insert请求
uint32_t m_insertArraySize | m_insertSet数组的大小
TxnManager* m_manager | 对应的TxnManager，或者说对应的是哪个事务

TxnInsertAction由TxnAccess管理，只在插入操作中使用，用于存放关于主键索引或者二级索引的插入请求。

### Access - 一个DML操作在事务私有空间(或者说本地)的执行
数据成员 | 作用
---|---
Row* m_localInsertRow | 对于insert操作，在m_localInsertRow中保存待插入的数据，如果事务执行成功，则会以m_localInsertRow作为新插入的行
Row* m_localRow | 对于update/delete操作，在m_localRow中分别保存更新后的数据/待删除的行的数据
Row* m_auxRow | 对于同一个事务中先delete后insert的场景下，在m_auxRow中保存待插入的数据
Sentinel* m_origSentinel | 当前access所关联的行(全局的被所有事务共享的行，而不是当前事务本地的行)在索引中对应的sentinel(如果是关于二级索引的access，则对应的是二级索引中的sentinel，如果是主键索引的access，则对应的是主键索引中的sentinel)，通过它可以获取当前access所关联的行(全局的被所有事务共享的行，而不是当前事务本地的行)
BitmapSet m_modifiedColumns | 当前操作更新了哪些列，在记录事务redo日志的时候会用到
TransactionId m_tid | 当前操作发生之前，它所关联的行的最新事务id，用于在OCC并发控制中，运行commit protocol的时候检查是否有其它事务并发的更新了该操作所关联的行
AccessParams<uint8_t> m_params | 当前Access中设置了哪些标识，可以设置的标识包括PRIMARY_SENTINEL, UNIQUE_INDEX, UPGRADE_INSERT, ROW_COMMITTED等
AccessType m_type = AccessType::INV | 这个Access对应的操作类型，比如read, write, delete, insert
uint32_t m_stmtCount = 0 | 当前操作是当前事务中的第几个操作
uint32_t m_localRowSize | 本地更新后row的大小

Access类中还包括几个重要方法：
```
方法原型1：
Sentinel* GetSentinel() const

作用：
获取这个Access所关联的行(全局的被所有事务共享的行，而不是当前事务本地的行)在索引(可能是主键索引，也可能是二级索引)中的sentinel

实现：
直接返回Access::m_origSentinel，通过这个sentinel可以唯一定位到一个行

方法原型2：
Row* GetRowFromHeader() const

作用：
获取这个Access所关联的行(全局的被所有事务共享的行，而不是当前事务本地的行)

实现：
对于insert操作且不是upgrade(从delete操作upgrade到insert)的insert操作，直接返回Access::m_localInsertRow
对于其它操作，通过Access::m_origSentinel获取对应的行，如果Access::m_origSentinel是主键索引对应的sentinel，则返回Access::m_origSentinel::m_status记录的Row的地址，如果Access::m_origSentinel是二级索引对应的sentinel，则首先通过Access::m_origSentinel::m_status找到主键索引对应的sentinel，记为primary_sentinel，然后返回primary_sentinel::m_status中记录的Row的地址

方法原型3：
Row* GetTxnRow() const

作用：
获取这个Access所关联的本地行(当前事务在本地私有空间的行)

实现：
对于非insert类型的操作，返回Access::m_localRow
对于insert类型的操作，如果是upgrade(从delete操作upgrade到insert)的insert操作，则返回Access::m_auxRow，如果不是upgrade的insert操作，则返回m_localInsertRow
```

### TxnAccess - 一个事务中所有Access的集合，也被称为Access Manager
数据成员 | 作用
---|---
Access** m_accessesSetBuff | 关于Access的数组，用于从其中分配Access
uint32_t m_allocatedAc | 已从m_accessesSetBuff数组中分配的Access的数目
uint32_t m_accessSetSize | m_accessesSetBuff数组的大小
TxnManager* m_txnManager | 当前TxnAccess是属于哪个TxnManager的，或者说对应的是哪个事务
TxnInsertAction* m_insertManager | 用于管理insert操作对应的insert请求
Access* m_lastAcc | 当前TxnAccess中所管理的最新一个Access
TxnOrderedSet_t* m_rowsSet | TxnAccess所管理的所有(sentinel, access)2元组，这些2元组在TxnAccess中按照sentinel指针地址排序存放，目的是避免在OCC并发控制算法中出现死锁，保证加锁顺序

### DDLAccess - 一个DDL操作在事务私有空间(或者说本地)的执行
暂略

### TxnDDLAccess - 一个事务相关的所有DDLAccess的集合，也被称为DDLAccess Manager
数据成员 | 作用
---|---
TxnManager* m_txn | 对应的TxnManager，或者说对应的是哪个事务
uint16_t m_size | 当前管理了多少个DDLAccess
TxnDDLAccess::DDLAccess* m_accessList[MAX_DDL_ACCESS_SIZE] | 所有DDLAccess都保存在这里

### TxnManager - 管理一个事务的生命周期
数据成员 | 作用
---|---
uint64_t m_latestEpoch | 事务看到的最新的epoch
uint64_t m_threadId | 执行当前事务的线程ID
uint64_t m_connectionId | 执行当前事务的连接ID
SessionContext* m_sessionContext | 执行当前事务的session
RedoLog m_redoLog | 当前事务所关联的RedoLog相关的信息，用于写事务redo日志
OccTransactionManager m_occManager | 事务并发控制管理器
GcManager* m_gcSession | 每个session独有的Gabage collector
CheckpointPhase m_checkpointPhase | Checkpoint所在的阶段，包括rest，prepare，resolve，capture，complete
bool m_checkpointNABit | 当前是0还是1代表not_available，因为Checkpoint中会发生available和not_available的切换所致
uint64_t m_csn | 事务commit的时候获取到的commit sequence number
uint64_t m_transactionId | 事务启动的时候分配的transaction id
SurrogateKeyGenerator m_surrogateGen | 如果没有指定primary key的情况下，使用SurrogateKey建立主键索引，m_surrogateGen用于生成SurrogateKey
TxnState m_state | 事务当前所处的状态，包括TXN_START, TXN_COMMIT, TXN_ROLLBACK, TXN_END_TRANSACTION等
int m_isolationLevel | 事务隔离级别
MemSessionPtr<TxnAccess> m_accessMgr | 当前事务DML相关的Access Manager，管理当前事务相关的所有的DML Access
TxnDDLAccess* m_txnDdlAccess | 当前事务DDL相关的Access Manager，管理当前事务相关的所有的DDL Access

### Row - 一行数据
数据成员 | 作用
---|---
RowHeader m_rowHeader | 用于OCC并发控制
Table* m_table | 这个row所属的Table
uint64_t m_surrogateKey | 如果没有PrimaryKey的情况下，使用它作为内置的PrimaryKey
Sentinel* m_pSentinel | 主键索引中指向该row的sentinel
uint64_t m_rowId | 该Row的ID，在Checkpoint和RedoLog中都会用到它，另外如果某个索引是非唯一索引，则会在原始的key之后添加m_rowId作为新的key，以变成唯一索引
KeyType m_keyType | key的类型，可取值为SURROGATE_KEY, INTERNAL_KEY等，在MOT中目前没有起到什么作用
bool m_twoPhaseRecoverMode | 是否正在recovery过程中
uint8_t m_data[0] | 数据部分，其中保存每列的数据

### RowHeader - 一行数据的头部部分，用于OCC并发控制
数据成员 | 作用
---|---
volatile uint64_t m_csnWord | 该行记录对应的commit sequence number
> 事实上m_csnWord中不仅存储了csn信息，还存储了status信息。m_csnWord会被划分为2个区域，低位的61个bits被用于存储commit sequence number，高位的3个bits被用于存储status信息，第63 bit是lock bit，用于保护记录不被并发更新，第62 bit是latest version bit，用于标记该记录是否是对应key的最新的记录，第61 bit是absent bit，用于标记该记录没有对应的key，absent bit主要用于insert和remove操作

### Sentinel - 主键索引/二级索引中的value部分
数据成员 | 作用
---|---
volatile uint64_t m_status | 用于记录一些状态信息以及地址信息，对于主键索引，地址信息中记录的是Row在内存中的地址，对于二级索引，地址信息中记录的则是主键索引对应的Sentinel的地址
Index* m_index | 该Sentinel是关于哪个索引的
uint64_t m_stable | 用于记录一些状态信息以及地址信息，主要用于CALC(Checkpoint Asynchronously using Logical Consistency)算法中checkpoint
volatile uint32_t m_refCount | 关于相同key的并发插入数目

### OccTransactionManager - 乐观并发控制管理(参考[SILO原型](https://dl.acm.org/doi/10.1145/2517349.2522713)实现)
数据成员 | 作用
---|---
uint32_t m_txnCounter | 和m_abortsCounter一起，用于计算冲突比例是否高
uint32_t m_abortsCounter | 和m_txnCounter一起，用于计算冲突比例是否高
uint32_t m_writeSetSize | TxnAccess中有多少个操作类型为write/insert/delete的access
uint32_t m_rowsSetSize | 当前事务涉及到多少个行的操作，相同行的操作只算一个
uint32_t m_deleteSetSize | TxnAccess中有多少个操作类型为delete的access
uint32_t m_insertSetSize | TxnAccess中有多少个操作类型为insert的access
uint16_t m_dynamicSleep | 在冲突比例较高的情况下，动态睡眠的时间
bool m_rowsLocked | 是否已经成功在sentinel和row上加锁
bool m_preAbort | 如果设置为true，则在commit-validation阶段会调用quickVersionCheck进行快速检查
bool m_validationNoWait | 如果设置为true(默认设置)，则在commit-validation阶段会调用Sentinel::TryLock，不等待加锁成功，如果设置为false，则在commit-validation阶段会调用Sentinel::Lock，直到加锁成功才退出

## 基于SILO算法的乐观并发控制
根据[SILO算法](https://dl.acm.org/doi/10.1145/2517349.2522713)，事务在执行阶段都是在事务私有空间进行，在提交阶段，运行[commit-protocol](http://note.youdao.com/noteshare?id=926619a23e770e691f6eb061c3271ed9&sub=804CA043DFC14AD382249D99E94F49F1)。

### SILO算法中commit-protocol的伪代码
```
// Phase 1
for w, v in WriteSet {
    Lock(w); // use a lock bit in TID
}

Fence(); // compiler-only on x86，确保获取Global_Epoch的过程是在之前所有的内存访问之后
e = Global_Epoch; // serialization point
Fence(); // compiler-only on x86，确保获取Global_Epoch的过程是在后续所有的内存访问之前

// Phase 2
for r, t in ReadSet {
    Validate(r, t); // abort if fails
}

tid = Generate_TID(ReadSet, WriteSet, e);

// Phase 3
for w, v in WriteSet {
    Write(w, v, tid);
    Unlock(w);
}
```

### SILO算法中commit-protocol的解释
Phase 1：
- 对write-set中的所有的记录(非本地记录，而是table中的全局共享的记录)加锁(将TID中的lock bit设置为1)，为了避免死锁，必须按照一定的顺序进行加锁，SILO采用的是按照记录的地址的顺序进行加锁
- 获取全局的Epoch(必须确保是全局的，而非本地缓存的)

Phase 2：
- 检查read-set中的所有的记录，对于每个记录，都要拿read-set中的本地记录跟table中的全局共享的记录进行比较，检查是否发生以下现象：
    - 两者的TID发生了变更
    - table中的全局共享的记录不是最近的记录(TID中的latest-version bit为0)
    - 其它事务已经在table中的全局共享的记录上加了锁(或者说，TID中的lock bit为1，但是这个记录不在当前事务的write-set中)
- 如果检查结果显示上述现象中有至少一种发生，则释放锁，并且abort当前的事务
- 如果检查结果显示上述现象都没有发生，则使用在Phase 1中获取的Epoch结合TID分配算法来为当前的事务分配一个TID

Phase 3：
- 将所有的更新应用到table中所有的全局共享的记录中
- 更新这些记录中的TID为当前事务的TID
- 释放当前事务在table中的全局共享记录上所加的锁

### MOT中commit-protocol的实现
MOT中与SILO算法中commit-protocol相关的代码如下：
```
- 1. MOTAdaptor::Commit
    - 2. TxnManager::Commit
        - 3. OccTransactionManager::ValidateOcc
            - 4. OccTransactionManager::LockHeaders
                - 5. 对TxnAccess中所有的access执行：
                    - 6. access所关联的全局行所对应的sentinel(即access::m_origSentinel)上加锁
            - 7. OccTransactionManager::ValidateReadSet
                - 8. 对TxnAccess中所有操作类型为read的access上执行：
                    - 9. 检查access所关联的全局行(通过access::GetRowFromHeader方法获取)上是否已经加锁
                    - 10. 检查access所关联的全局行(通过access::GetRowFromHeader方法获取)中记录的commit sequence number和access中记录的commit sequence number是否一致
                    - 11. 如果9或者10的检查结果是“否”，则当前事务abort
            - 12. OccTransactionManager::ValidateWriteSet
                - 13. 对TxnAccess中所有操作类型不是read且关联的sentinel是PrimarySentinel的access上分别执行：
                    - 14. 检查access所关联的全局行(通过access::GetRowFromHeader方法获取)中记录的commit sequence number和access中记录的commit sequence number是否一致
                    - 15. 如果检查结果是“否”，则当前事务abort
        - 16. TxnManager::CommitInternal // 只在步骤3返回成功的情况下执行
            - 17. 设置当前事务的commit sequence number
            - 18. 写redo日志
            - 19. 应用DDL相关的更新
            - 20. OccTransactionManager::WriteChanges
                - 21. OccTransactionManager::LockRows
                    - 22. 对TxnAccess中所有操作类型不是read且关联的sentinel是PrimarySentinel的access上分别执行：
                        - 23. 在access所关联的全局行(通过access::GetRowFromHeader获取)上加锁
                - 24. 对TxnAccess中所有操作类型不是read的access上分别执行：
                    - 25. RowHeader::WriteChangesToRow
                        - 26. 如果操作类型是write
                            - 27. 将access::m_local中保存的本地更新拷贝到access所关联的全局行(通过access::GetRowFromHeader方法获取)上
                            - 28. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的commit sequence number
                            - 29. 在access所关联的全局行(通过access::GetRowFromHeader方法获取)上加锁
                                - 30. 这可以看做是步骤23的延续，因为commit sequence number和LOCK标识都保存在RowHeader::m_csnWord中，步骤28和步骤29是原子操作 
                        - 31. 如果操作类型是delete
                            - 32. 如果access所关联的sentinel是PrimarySentinel，则：
                                - 33. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的commit sequence number
                                - 34. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的LOCK标识
                                    - 35. 这可以看做是步骤23的延续，因为commit sequence number和LOCK标识都保存在RowHeader::m_csnWord中，步骤33和步骤34是原子操作 
                                - 36. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的ABSENT标识
                                    - 步骤36和步骤33、步骤34都是原子操作
                                - 37. 如果access所关联的sentinel中没有stable版本的记录，则还会设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的LATEST_VERSION标识
                            - 38. 在access所关联的全局行所对应的sentinel(即access::m_origSentinel)上设置DIRTY标识
                        - 39. 如果操作类型是insert
                            - 40. 如果access所关联的sentinel是PrimarySentinel，则：
                                - 41. 如果是upgrade insert(也就是在同一个事务中先执行了delete操作，然后又执行了insert操作)
                                    - 42. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的commit sequence number
                                    - 43. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的LOCK标识
                                        - 44. 这可以看做是步骤23的延续，因为commit sequence number和LOCK标识都保存在RowHeader::m_csnWord中，步骤42和步骤43是原子操作 
                                    - 45. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的LATEST_VERSION标识
                                        - 步骤45和步骤42、步骤43都是原子操作
                                - 46. 如果不是upgrade insert，则：
                                    - 47. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的commit sequence number
                                    - 48. 设置access所关联的全局行(通过access::GetRowFromHeader方法获取)中的LOCK标识
                                        - 49. 这可以看做是步骤23的延续，因为commit sequence number和LOCK标识都保存在RowHeader::m_csnWord中，步骤47和步骤48是原子操作 
                - 50. 对TxnAccess中所有操作类型是insert的access上分别执行：
                    - 51. 如果是upgrade insert(也就是在同一个事务中先执行了delete操作，然后又执行了insert操作)，执行；
                        - 52. 如果access所关联的sentinel是PrimarySentinel，则：
                            - 53. 将access当前所关联的全局行(通过access::GetRowFromHeader方法获取)保存到access::m_localInsertRow中
                            - 54. 将access所关联的全局行设置为access::m_auxRow(access::m_origSentinel没有变，但是access::m_origSentinel中记录的行的地址变为access::m_auxRow)
                            - 55. 将access::m_localInsertRow所代表的行添加到Garbage Collector中，等待回收
                        - 56. 否则：
                            - 57. 将access->m_origSentinel中记录的主键索引对应的sentinel修改为access::m_auxRow中记录的关于主键索引的sentinel
                    - 58. 否则，执行：
                        - 59. 如果access所关联的sentinel是PrimarySentinel，则：
                            - 60. 将access所关联的全局行设置为access::m_localInsertRow(access::m_origSentinel没有变，但是access::m_origSentinel中记录的行的地址变为access::m_localInsertRow)
                        - 61. 否则：
                            - 62. 将access->m_origSentinel中记录的主键索引对应的sentinel修改为access::m_localInsertRow中记录的关于主键索引的sentinel
                - 62. 对TxnAccess中所有操作类型是insert的access上分别执行：
                    - 63. 清除access所关联的全局行所对应的sentinel(即access::m_origSentinel)上的DIRTY标识
- 64. MOTAdaptor::EndTransaction
    - 65. TxnManager::EndTransaction
        - 66. OccTransactionManager::ReleaseLocks
            - 67. OccTransactionManager::ReleaseHeaderLocks
                - 68. 对TxnAccess中所有操作类型不是read的access上分别执行：
                    - 69. 释放access所关联的全局行所对应的sentinel(即access::m_origSentinel)上的锁
            - 70. OccTransactionManager::ReleaseRowsLocks
                - 71. 对TxnAccess中所有操作类型不是read的access上分别执行：
                    - 72. 如果access所关联的sentinel是PrimarySentinel，则：
                        - 73. 释放access当前所关联的全局行(通过access::GetRowFromHeader方法获取)上的锁
                        - 74. 如果是upgrade insert类型的access，则：
                            - 75. 清除保存在access::m_localInsertRow中的旧的Row上的锁
                                - 76. 见步骤53中对access::m_localInsertRow的设置
                                - 77. 实际上access::m_localInsertRow已经交给Garbage Collector等待回收了
        - 78. OccTransactionManager::CleanRowsFromIndexes
            - 79. 对TxnAccess中所有操作类型为delete的access上分别执行：
                - 80. TxnManager::RemoveKeyFromIndex
                    - 81. 如果access所关联的全局行(对于delete操作类型的access，保存在access::m_localRow中)中没有stable版本的记录，则执行：
                        - 82. Table::RemoveKeyFromIndex
                            - 83. 调用Sentinel::RefCountUpdate，减少access所关联的全局行所对应的sentinel(即access::m_origSentinel)上的引用计数
                            - 84. 如果步骤83的执行之后，access所关联的全局行所对应的sentinel(即access::m_origSentinel)上的引用计数降为0，则：
                                - 85. 调用Index::IndexRemove，从索引中删除相应的记录
```

上述整个过程就是MOT中实现的commit-protocol。步骤3-15对应phase1和phase2，步骤16-85对应phase3。

## 事务操作相关的逻辑
### 通用逻辑
我们将插入行/更新行/查询行/删除行中都可能用到的逻辑单独拿出来分析，在具体分析插入行/更新行/查询行/删除行的过程中，如果遇到了通用逻辑，则跳转到这里即可。

#### 创建关于某行记录在当前事务的本地记录
Opengauss MOT中采用的是OCC进行并发控制，事务中关于某行记录的操作会在当前事务的私有空间进行。
```
函数原型：
Access* TxnAccess::GetNewRowAccess(const Row* row, AccessType type, RC& rc)

输入：
Row* inputRow - 某行记录
AccessType type - 操作类型

输出：
Access* - 关于某行数据在本地的操作
RC - 状态码

执行逻辑：
1. 为当前事务创建一个本地操作记录，类型为Access，记为access
2. 如果是非insert类型的操作，执行如下：
    2.1 分配一个Row实例，用于在本地保存行数据，并关联到access::m_localRow
    2.2 将inputRow中的记录拷贝access::m_localRow中
        2.2.1 如果row::m_rowHeader中设置了ABSENT标识(则表示该记录不存在)，且当前操作类型是insert之外的其它类型，则直接返回，设置状态码为ABORT
        2.2.2 等到row::m_rowHeader不再变更且没有任何事务在它上加锁的时候，对于操作类型是insert之外的所有操作，将row中的记录拷贝到access::m_localRow中
        2.2.3 获取该记录最后一次更新时的Tid，记为lastTid，设置状态码为OK
    2.3 如果状态码不为OK，则释放在步骤2.1中分配的Row实例，并返回null
    2.4 设置access::m_localRow对应的primary sentinel为inputRow对应的primary sentinel
    2.5 设置access的ROW_COMMITTED标识
    2.6 设置access的操作类型，即access::m_tid = type
    2.7 设置access的对应的事务ID，即access::m_tid = lastTid
3. 如果是insert类型的操作，则执行如下：
    3.1 将inputRow关联到access::m_localInsertRow(用于存放插入的记录)
    3.2 清除access的ROW_COMMITTED标识
    3.3 设置access的操作类型，即access::m_tid = insert
    2.7 设置access的对应的事务ID，即access::m_tid = 0
```

#### 将某行记录添加到本地记录缓存中
```
函数原型：
Row* TxnAccess::MapRowtoLocalTable(const AccessType type, Sentinel* const& originalSentinel, RC& rc)

输入：
AccessType type - 操作类型
Sentinel* const& originalSentinel - 某行记录所关联的sentinel

输出:
Row* - 本地记录
RC - 状态码

执行逻辑：
1. 为originalSentinel对应的行创建本地记录，见“创建关于某行记录在当前事务的本地记录”，返回的本地记录记为access
2. 如果access为null，则直接返回null
3. 根据originalSentinel设置access所关联的primary sentinel(即access::m_origSentinel)
    3.1 如果originalSentinel是primary sentinel，则access所关联的primary sentinel就是originalSentinel
    3.2 否则，根据originalSentinel获取primary sentinel，并把获得的primary sentinel设置为access所关联的primary sentinel
4. 设置access的PRIMARY_SENTINEL标识
5. 将以access::m_origSentinel为key，以access为value的键值对添加到本地记录缓存(TxnAccess::m_rowsSet)中
6. 返回本地记录，即access::m_localRow
```

#### commit阶段逻辑
```
1. 并发控制器进行commit前的验证工作
2. 如果验证通过，则更新table，否则终止事务
```

##### commit前的验证准备工作
```
1. 获取Access Manager中的关于Sentinel到它所关联的Access的映射表，记为orderedSet
2. 遍历orderedSet中的所有的条目，获取每个条目对应的Access，根据它们的操作类型统计各种类型的操作的数目，包括rowSetSize(关联到的Sentinel是primary sentinel的Access)，writeSetSize(操作类型为update/insert/delete)， insertSetSize(操作类型为insert)， deleteSetSize(操作类型为delete)
3. 在所有update/insert/delete类型的Access对应的Sentinel上加锁，根据是否等待加锁成功(实现上，如果不需要等待加锁成功，则使用Sentinel::TryLock，否则使用Sentinel::Lock)分别进行处理，这里以需要等待加锁成功为例进行分析：
    3.1 逐一遍历orderedSet中的每一个条目，并对每一个条目做如下处理：
        3.1.1 获取该条目对应的Access
        3.1.2 如果该Access的操作类型是read，则略过
        3.1.3 在Access::m_origSentinel上加锁，并且等待加锁成功
        3.1.4 检测Access对应的Row，从添加到orderedSet开始到现在为止，是否发生了版本变更
            3.1.4.1 如果是update/delete类型的操作，则检查RowHeader中记录的commit sequence number和在Access中记录的commit sequence number相比是否发生了变更
            3.1.4.2 如果是insert类型的操作，则检查Access::m_origSentinel中是否清除了DIRTY标识，如果清除了该标识，则表示发生了变更
        3.1.5 根据步骤3.1.4中检测结果分情况处理
            3.1.5.1 如果发生了变更，则中止提交，见“中止提交”
            3.1.5.2 否则，继续后续流程
4. 检查orderedSet中所有操作类型为read的条目，并对每一个这样的条目进行如下处理：
    4.1 检查Access对应的Row上是否加了锁，或者RowHeader中记录的commit sequence number和在Access中记录的commit sequence number相比是否发生了变更
    4.2 根据步骤4.1的检查结果分情况处理
        4.2.1 如果发生了变更，则中止提交，见“中止提交”
        4.2.2 否则，继续后续流程
5. 检查orderedSet中所有操作类型为write/insert/delete的条目，并对每一个这样的条目进行如下处理：
    5.1 检查Access对应的Row的RowHeader中记录的commit sequence number和在Access中记录的commit sequence number相比是否发生了变更
    5.2 根据步骤5.1的检查结果分情况处理
        5.2.1 如果发生了变更，则中止提交，见“中止提交”
        5.2.2 否则，继续后续流程
```

##### 中止提交
中止提交过程主要是释放在Sentinel上所加的锁，步骤如下：
```
1. 获取Access Manager中的关于Sentinel到它所关联的Access的映射表，记为orderedSet
2. 逐一遍历orderedSet中的每一个条目，并对每一个条目做如下处理：
    2.1 获取该条目对应的Access
    2.2 如果该Access的操作类型是read，则略过
    2.3 如果该Access对应的Access::m_origSentinel上加了锁，则释放Access::m_origSentinel上所加的锁
```

##### 更新table
```
1. 为当前transaction设置commit sequence number
2. 将当前transaction中所涉及的所有的ddl和dml操作都写入redo log中
3. 应用所有ddl相关的更新
4. 应用所有dml相关的更新
    4.1 在所有待更新的行(全局的table的Row)上加锁(设置RowHeader中的LOCK标识)
        4.1.1 [说明:]不同于“并发控制器进行commit前的验证工作”中在sentinel上所加的锁
    4.2 获取Access Manager中的关于Sentinel到它所关联的Access的映射表，记为orderedSet
    4.3 处理orderedSet中的每一个write/insert/delete类型的条目，对于每个条目处理如下：
        4.3.1 获取该条目对应的Access
        4.3.2 对于write类型的操作：
            4.3.2.1 将Access本地的Row数据拷贝到全局的table的Row中
            4.3.2.2 更新全局的table的Row中记录的commit sequence number
        4.3.3 对于insert类型的操作：
            4.3.3.1 如果是关于主键索引的insert操作，则在待插入的行上设置commit sequence number
        4.3.4 对于delete类型的操作：
            4.3.4.1 设置Access对应的Sentinel的DIRTY标识
            4.3.4.2 如果是关于主键索引的delete操作，则在待插入的行上设置commit sequence number，LOCK标识和ABSENT标识
    4.4 处理orderedSet中的每一个insert类型的条目，对于每个条目处理如下：
        4.4.1 获取该条目对应的Access
        4.4.2 根据insert操作是否是关于主键索引的，分别进行处理：
            4.4.2.1 如果是关于主键索引的insert操作，则将Access中主键索引的sentinel和Access中新插入的Row建立关联
            4.4.2.2 如果是关于二级索引的insert操作，则将Access中二级索引的sentinel和主键索引的sentinel建立关联
        4.4.3 清除Access中对应sentinel的DIRTY标识，即表明该操作已经commit了
```

### 插入行
#### 执行阶段
```
1. 分配新的Row，记为newRow
2. 将待插入的的行的数据组装到newRow中
3. 根据待插入的行的数据组装生成主键primaryKey，并将(newRow，主键索引，primaryKey)三元组添加到Insert Manager中
    3.1 Insert Manager中包含一个insert请求数组，最终会将(newRow，主键索引，primaryKey)三元组添加到Insert Manager中的insert请数组中
    3.2 [说明:]这里只是将关于主键索引的更新请求添加到Insert Manager中，但并没有真正更新主键索引
4. 对于每一个二级索引，执行跟主键索引类似的操作：根据待插入的行的数据组装生成二级索引键secondaryKey，并将(newRow，二级索引，secondaryKey)三元组一并添加到Insert Manager中
    4.1 Insert Manager中包含一个insert请求数组，最终会将(newRow，二级索引，secondaryKey)三元组添加到Insert Manager中的insert请数组中
    4.2 [说明:]这里只是将关于二级索引的更新请求添加到Insert Manager中，但并没有真正更新二级索引
5. 逐一遍历Insert Manager中的每个insert请求，并对每个insert请求(记为curInsert)执行如下逻辑：
    5.1 从curInsert中获取它所对应的索引(主键索引或者二级索引)curIndex，键(主键或者二级索引键)curIndexKey
    5.2 将键curIndexKey插入到索引curIndex中
        5.2.1 如果索引树中不存在关于该键的条目，则创建一个新的Sentinel(构造函数中会设置DIRTY标识)，记为newSentinel，并将curIndexKey作为key，newSentinel作为value，添加到curIndex中，返回newSentinel
            5.2.1.1 [说明:]此时，curIndexKey和newSentinel已经添加到了索引树中，但是sentinel和newRow之间尚未建立关联
        5.2.2 否则，获取已存在的条目对应的Sentinel，记为existedSentinel，并返回existedSentinel
    5.3 如果curInsert是关于主键索引的insert请求，则:
        5.3.1 设置newRow的RowHeader中的ABSENT标识为1
        5.3.2 设置newRow对应的primary Sentinel是5.2中返回的Sentinel
        5.3.3 [说明:]此时，虽然newRow和primary Sentinel之间建立了关系，但是primary Sentinel和newRow之间并未建立关系，也就是说通过primary Sentinel是找不到newRow的
    5.4 如果5.2中返回的Sentinel已经清除了DIRTY标识(说明有其它的关于相同key的事务性插入操作已经commit，这个事务性的插入操作可能共用了当前事务在5.2中生成的newSentinel，也可能是当前事务在5.2中共用了已经存在的existedSentinel)
        5.4.1 检查当前事务的Access Manager中是否已经存在关于该Sentinel的操作
            5.4.1.1 [说明:] Access Manager用于在本地管理事务所有的操作，Access Manager中管理的是一个从Sentinel到对应的Access的映射表，Access代表事务在本地执行的一个操作
        5.4.2 如果当前事务中在本次插入操作之前已经存在关于相同Sentinel的操作：
            5.4.2.1 如果不是delete操作，则当前事务必须abort
            5.4.2.2 如果是delete操作，则将delete操作提升为insert操作，这里为什么不abort？
        5.4.3 如果当前事务中不存在存在关于相同Sentinel的操作，则当前事务必须abort
    5.5 如果在5.2中走的是5.2.1分支，或者5.2中返回的Sentinel尚未清除DIRTY标识，则添加一个insert row access到当前事务的Access Manager中，见“添加一个insert row access到当前事务的Access Manager中”
```

##### 添加一个insert row access到当前事务的Access Manager中
```
函数原型：
Row* TxnAccess::AddInsertToLocalAccess(Sentinel* org_sentinel, Row* org_row, RC& rc, bool isUpgrade=false)

输入参数：
Sentinel* org_sentinel - 某个sentinel
Row* org_row - 待插入的行
RC& rc - 
bool isUpgrade - 是否

执行说明：
Access Manager类型为TxnAccess，它维护了一个从Sentinel到它所关联的Access的映射表(m_rowsSet)，按照Sentinel指针地址排序，Sentinel可以简单理解为索引信息，而Access可以简单理解为事务在本地的操作

执行过程：
1. 在m_rowsSet中查找是否存在关于给定sentinel的条目；
2. 如果不存在，则：
    2.1 分配一个Access，记为curAccess
    2.2 如果isUpgrade为true，则将curAccess::m_localInsertRow关联到org_sentinel中记录的live版本，并将curAccess::m_auxRow关联到org_row
    2.2 如果isUpgrade为false，将curAccess::m_localInsertRow关联到org_row
    2.3 将curAccess::m_origSentinel关联到org_sentinel
    2.4 将org_sentinel到curAccess的映射添加到m_rowsSet中
    2.5 返回curAccess::m_localInsertRow
3. 否则：
    3.1 获取对应的Access，也就是查找到的条目中的value部分，记为curAccess；
    3.2 如果curAccess的操作类型是delete，则：
        3.2.1 如果允许提升：
            3.2.1.1 将curAccess的操作类型直接提升为insert
            3.2.1.2 将curAccess::m_auxRow关联到org_row
            3.2.1.3 返回curAccess::m_auxRow
        3.2.2 否则，提示错误信息
        
！！！在执行过程中，不会对参数中的sentinel做任何修改！！！
```

#### commit阶段
见“通用逻辑” -> “commit阶段逻辑”

### 查询行
```
1. 查找索引，找到给定key对应的Sentinel，记为sentinel
2. 查找当前事务的本地记录缓存中关于该sentinel所对应的Row的记录，返回找到的记录(记为localRow)和状态码(记为rc)，见“查找当前事务的本地记录缓存中关于给定sentinel所对应的Row的记录”
3. 如果成功在本地找到了记录，则获取本地记录的内容，并返回给调用者
```

#### 查找当前事务的本地记录缓存中关于给定sentinel所对应的Row的记录
```
函数原型：
Row* TxnManager::RowLookup(const AccessType type, Sentinel* const& sentinel, RC& rc)

输入：
AccessType type - 当前操作类型type
Sentinel* const& sentinel - 给定sentine

输出：
Row* - 当前事务本地记录缓存中关于给定sentinel所对应的Row的记录，如果不存在，则为null
RC rc - 状态码

执行过程：
1. 根据给定sentinel是否已经commit(清除了DIRTY标识)，分别处理，查找到的记录保存在localAccess中：
    1.1 如果已经commit(则该给定的sentinel一定是其它事务性插入操作创建的，因为当前事务仍在进行中，如果是当前事务创建的sentinel，则不可能已经commit了)：
        1.1.1 在当前事务的本地记录缓存中查找给定的sentinel
        1.1.2 如果没有找到，则进一步根据该sentinel是关于主键索引的还是二级索引的分别处理：
            1.1.2.1 如果该sentinel是关于主键索引的，则设置localAccess为null
            1.1.2.2 否则，该sentinel是关于二级索引的
            1.1.2.2.1 首先找到该sentinel所关联的关于主键索引的sentinel，记为primarySentinel
            1.1.2.2.2 然后在当前事务的本地记录缓存中查找primarySentinel
            1.1.2.2.2.1 如果没找到，则设置localAccess为null
            1.1.2.2.2.2 否则设置localAccess为查找到的记录
        1.1.3 否则，设置localAccess为查找到的记录
    1.2 否则(该给定的sentinel可能是当前事务创建的，比如当前事务中执行了insert操作，也可能是其它事务性插入操作创建的)：
        1.2.1 在当前事务的本地记录缓存中查找给定的sentinel
        1.2.2 如果找到，则设置localAccess为查找到的记录
        1.2.3 否则，设置localAccess为null
2. 根据localAccess是否为null以及localAccess对应的操作类型，相应的设置状态码rc：
    2.1 如果localAccess不为null，且操作类型是read/write/read_for_update/insert，则设置状态码rc为FOUND
    2.2 如果localAccess不为null，且操作类型是delete，则设置状态码rc为DELTED
    2.3 如果localAccess为null，则设置状态码rc为NOT_FOUND
3. 根据localAccess是否为null以及localAccess对应的操作类型，相应的设置localRow：
    3.1 如果localAccess不为null，且操作类型是read/write/read_for_update，则设置localRow为localAccess::m_localRow
    3.2 如果localAccess不为null，且操作类型是insert，则设置localRow为localAccess::m_localInsertRow
    3.3 如果localAccess不为null，且操作类型是delete，则将localAccess设置为null
    3.4 如果localAccess为null，则设置localRow为null
4. 根据rc和localRow分别进行处理：
    4.1 如果rc为DELTED，则返回null，表示没有对应的行(对应的行已经删除了)
    4.2 如果rc为FOUND，则返回localRow
    4.3 如果rc为NOT_FOUND，则根据sentinel是否已经commit分别处理：
        4.3.1 如果sentinel已经commit(清除了DIRTY标识)，则将该sentinel对应的Row的数据拷贝到本地记录缓存中，实际上是在当前事务的Access Manager中添加了一个新的Access，并且设置该Access的类型为read，见“通用逻辑” -> “创建关于某行记录在当前事务的本地记录”，并返回新的Access中m_localRow
        4.3.2 否则，返回null，表示没有对应的行
    4.4 否则，返回null
    
！！！上述步骤4.3.1非常重要，虽然是在当前事务的本地记录缓存中查找给定sentinel所对应的Row的记录，但是如果在本地缓存中没有找到的话，会尝试从sentinel所对应的共享的Row中拷贝数据到本地记录缓存中，实际上是在当前事务的Access Manager中添加了一个新的Access，并且设置该Access的类型为read，按照SILO论文中的说法，其实是将要操作的数据添加到了read-set中。

所有的操作都会调用MOTIterateForeignScan，而MOTIterateForeignScan中会调用TxnManager::RowLookup，所以对于一个事务来说，如果其中包含多个关于相同key的操作，则只在第一次关于该key的操作的时候，会在Access Manager找不到对应的Access，而在该事务中关于该key的后续操作，都会在Access Manager中找到对应的Access。
```


### 更新行
#### 执行阶段
```
1. 查找索引，找到给定key对应的Sentinel，记为sentinel
2. 查找当前事务的本地记录缓存中关于该sentinel所对应的Row的记录，返回找到的记录(记为localRow)和状态码(记为rc)，见“查找当前事务的本地记录缓存中关于给定sentinel所对应的Row的记录”
    [说明:]如果在本地记录缓存中没有找到对应的记录，则会尝试从sentinel所对应的共享的Row中拷贝数据到本地记录缓存中，然后再返回
3. 如果没有找到对应的记录，则直接返回
4. 否则，对于找到的记录，它在当前事务的本地记录缓存中存在一个key为sentinel，value为localAccess的条目，对于该条目执行如下处理：
    4.1 由于localAccess可能是从当前事务本地记录缓存中找到的，它当中记录的操作类型可能是当前事务中当前操作之前的关于相同行的其它操作对应的操作类型，而当前操作的操作类型可能和之前的关于相同行的其它操作对应的操作类型不同，因此必须在这里进行操作类型转换以及必要的处理
        4.1.1 检查是否允许操作类型转换
        4.1.2 如果允许转换，则确定转换后的操作类型是啥，否则报错返回
        4.1.3 检查需要进行哪些必要的处理来完成转换
        4.1.4 4.1.1 - 4.1.3见“其它操作类型转换为write类型”
    4.2 使用新的数据来更新本地缓存中关于该行的记录
```

##### 其它操作类型转换为write类型
转换前状态 | 想要转换的状态 | 转换后状态 | 必要处理
---|---|---|---
invalid | write | write | N/A
read | write | write | N/A
write | write | write | N/A
delete | write | delete | 不允许转换
insert | write | insert | N/A
 
#### commit阶段
见“通用逻辑” -> “commit阶段逻辑”


### 删除行
#### 执行阶段
```
1. 查找索引，找到给定key对应的Sentinel，记为sentinel
2. 查找当前事务的本地记录缓存中关于该sentinel所对应的Row的记录，返回找到的记录(记为localRow)和状态码(记为rc)，见“查找当前事务的本地记录缓存中关于给定sentinel所对应的Row的记录”
    [说明:]如果在本地记录缓存中没有找到对应的记录，则会尝试从sentinel所对应的共享的Row中拷贝数据到本地记录缓存中，然后再返回
3. 如果没有找到对应的记录，则直接返回
4. 否则，对于找到的记录，它在当前事务的本地记录缓存中存在一个key为sentinel，value为localAccess的条目，对于该条目执行如下处理：
    4.1 由于localAccess可能是从当前事务本地记录缓存中找到的，它当中记录的操作类型可能是当前事务中当前操作之前的关于相同行的其它操作对应的操作类型，而当前操作的操作类型可能和之前的关于相同行的其它操作对应的操作类型不同，因此必须在这里进行操作类型转换以及必要的处理
        4.1.1 检查是否允许操作类型转换
        4.1.2 如果允许转换，则确定转换后的操作类型是啥，否则报错返回
        4.1.3 检查需要进行哪些必要的处理来完成转换
        4.1.4 4.1.1 - 4.1.3见“其它操作类型转换为delete类型”
```

##### 其它操作类型转换为delete类型
转换前状态 | 想要转换的状态 | 转换后状态 | 必要处理
---|---|---|---
invalid | delete | delete | 添加关于“删除所有二级索引”的操作到本地记录缓存
read | delete | delete | 添加关于“删除所有二级索引”的操作到本地记录缓存
write | delete | delete | 添加关于“删除所有二级索引”的操作到本地记录缓存
delete | delete | delete | 不允许转换
insert | delete | delete | 从本地记录缓存中删除所有关于索引的操作

#### commit阶段
见“通用逻辑” -> “commit阶段逻辑”



