# 提纲
[toc]

# 说明
下文中会使用“access所关联的全局的row”指代所有事务都可以共享访问的row，以和当前事务本地更新的row进行区分。

# 非并发插入操作的逻辑
下面的分析基于以下假设：
- 事务中只包含一个插入操作，没有任何其它操作
- 插入之前不存在与待插入的key相同的行
- 插入过程中没有任何其它事务与之并发
- 只有主键索引，没有任何二级索引
- 隔离级别是REPEATABLE_READ
```
MOTExecForeignInsert
    - MOTAdaptor::InsertRow
    
MOTAdaptor::InsertRow
    # 分配一个新的Row
    - MOT::Row* row = table->CreateNewRow()
    # 将要插入的数据打包到新分配的Row中
    - newRowData = const_cast<uint8_t*>(row->GetData());
      PackRow(slot, table, fdwState->m_attrsUsed, newRowData)
    - 生成主键primaryKey，并将(row, 主键索引, primaryKey)三元组添加到insert manager中
    - 获取insert manager中的每一个三元组(row, index, key)，当前上下文中insert manager中实际上只有一个三元组，并对该三元组处理如下：
        - 创建一个sentinel，设置DIRTY标识，设置live版本为null，设置stable版本为null
        - 以key为键，以sentine为value添加到index中
        - 设置row的ABSENT标识，设置row::m_pSentinel为该sentinel
    - 分配一个access，并将(sentinel, access)2元组添加到access manager中：
        - 分配access
        - 清除access::m_params中的RowCommitted标识
        - 设置access::m_localInsertRow为row
        - 设置access::m_type为insert
        - 设置access::m_tid为0
        - 设置access::m_origSentinel为sentinel
        - 设置access::m_params中的PrimarySentinel和UniqueIndex标识
        - 将(sentinel, access)2元组添加到access manager中
    
MOTAdaptor::Commit
    - OccTransactionManager::ValidateOcc
        - OccTransactionManager::LockHeaders
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，对这个2元组处理如下：
                - 在access::m_origSentinel上加锁，也就是设置LOCK标识
            - OccTransactionManager::QuickHeaderValidation
                - 在当前上下文中，没什么可做的
        - OccTransactionManager::ValidateReadSet
            - 在当前上下文中只有insert操作，所以什么都不做
        - OccTransactionManager::ValidateWriteSet
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，且当前上下文中这个2元组的操作类型为insert，且不是upgrade insert，处理如下：
                - 检查access::m_localInsertRow::m_rowHeader中记录的commit sequence number和access::m_tid是否相同，如果不同，就abort当前的事务
                    - 在当前上下文中显然不会不同，所以不会abort当前的事务
    - TxnManager::CommitInternal
        - 分配commit sequence number
        - 记录redo日志
        - TxnManager::WriteDDLChanges
            - 当前上下文中没有ddl相关的变更
        - OccTransactionManager::WriteChanges
            - OccTransactionManager::LockRows
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，且当前上下文中这个2元组的操作类型为insert，且不是upgrade insert，处理如下：
                    - 在access::m_localInsertRow::m_rowHeader上加锁(设置LOCK标识)
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，且当前上下文中这个2元组的操作类型为insert，且不是upgrade insert，处理如下：
                - 设置access::m_localInsertRow::m_rowHeader::m_csnWord为前面分配的commit sequence number，并且设置LOCK标识
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，且当前上下文中这个2元组的操作类型为insert，且不是upgrade insert，处理如下：
                - 设置access::m_origSentinel中live版本为access::m_localInsertRow
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，且当前上下文中这个2元组的操作类型为insert，且不是upgrade insert，处理如下：
                - 清除access::m_origSentinel中DIRTY标识

MOTAdaptor::EndTransaction
    - TxnManager::EndTransaction
        - OccTransactionManager::ReleaseLocks
            - OccTransactionManager::ReleaseHeaderLocks
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，且当前上下文中这个2元组的操作类型为insert，且不是upgrade insert，处理如下：
                    - 清除access::m_origSentinel上的LOCK标识
            - OccTransactionManager::ReleaseRowsLocks
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，且当前上下文中这个2元组的操作类型为insert，且不是upgrade insert，处理如下：
                    - 清除access::m_localInsertRow::m_rowHeader上的LOCK标识
        - OccTransactionManager::CleanRowsFromIndexes
            - 当前上下文中，access manager中没有delete操作类型的access，所以什么也不做
```

关于上述过程的关键逻辑总结：
1. 在MOTAdaptor::InsertRow中会分配一个新的row，并将待插入的数据拷贝到row中；
2. 在MOTAdaptor::InsertRow中会创建一个新的sentinel并插入主键索引中，并且对sentinel和row进行如下设置：
    - 2.1 sentine设置DIRTY标识，也就是说该sentinel是没有commit的
    - 2.2 sentinel的live版本和stable版本均设置为null
    - 2.3 row设置ABSENT标识，同时设置row::m_pSentinel为sentinel，也就是将row和sentinel建立了关联，从row可以找到sentinel，但是sentinel中并没有关联row，所以从sentinel还找不到row
3. 在MOTAdaptor::InsertRow中会分配一个access，并将(sentinel, access)2元组添加到access manager中，并且对access进行如下设置：
    - 3.1 清除access::m_params中的RowCommitted标识
    - 3.2 设置access::m_localInsertRow为row
    - 3.3 设置access::m_type为insert
    - 3.4 设置access::m_tid为0
    - 3.5 设置access::m_origSentinel为sentinel
4. 在MOTAdaptor::Commit -> OccTransactionManager::ValidateOcc -> OccTransactionManager::LockHeaders中会在access::m_origSentinel上设置LOCK标识；
5. 在MOTAdaptor::Commit -> TxnManager::CommitInternal中分配commit sequence number；
6. 在MOTAdaptor::Commit -> TxnManager::CommitInternal中记录redo log；
7. 在MOTAdaptor::Commit -> TxnManager::CommitInternal -> OccTransactionManager::WriteChanges ->  OccTransactionManager::LockRows中会在access::m_localInsertRow::m_rowHeader上设置LOCK标识；
8. 在MOTAdaptor::Commit -> TxnManager::CommitInternal -> OccTransactionManager::WriteChanges中使用步骤5中分配的commit sequence number设置access::m_localInsertRow::m_rowHeader::m_csnWord，并且在access::m_localInsertRow::m_rowHeader::m_csnWord上加LOCK标识；
9. 在MOTAdaptor::Commit -> TxnManager::CommitInternal -> OccTransactionManager::WriteChanges中设置access::m_origSentinel中live版本为access::m_localInsertRow；
10. 在MOTAdaptor::Commit -> TxnManager::CommitInternal -> OccTransactionManager::WriteChanges中清除access::m_origSentinel中DIRTY标识，也就是说该sentinel成功commit了；
11. 在MOTAdaptor::EndTransaction -> TxnManager::EndTransaction -> OccTransactionManager::ReleaseLocks -> OccTransactionManager::ReleaseHeaderLocks中清除access::m_origSentinel上的LOCK标识；
12. 在MOTAdaptor::EndTransaction -> TxnManager::EndTransaction -> OccTransactionManager::ReleaseLocks ->OccTransactionManager::ReleaseRowsLocks中清除access::m_localInsertRow::m_rowHeader上的LOCK标识；

说明：
- 步骤2.1中在sentine上设置DIRTY标识，在步骤10中被清除sentinel中的DIRTY标识；
- 步骤2.2中sentinel的live版本被设置为null，在步骤9中sentinel的live版本被设置为access::m_localInsertRow；
- **步骤2.3中row上设置的ABSENT标识，没有被清除？**
- 步骤2.3中通过将row::m_pSentinel设置为sentinel，将row和sentinel建立了关联，然后在步骤9中将sentinel的live版本设置为access::m_localInsertRow(其实就是步骤1中分配的row)，将sentinel和row建立了关联；
- 步骤3.4中将access::m_tid设置为0，而在步骤8中，将access::m_localInsertRow::m_rowHeader::m_csnWord设置为步骤5中分配的commit sequence number；
- 步骤4中在access::m_origSentinel上设置LOCK标识，在步骤11中清除access::m_origSentinel上的LOCK标识；
- 步骤7中在access::m_localInsertRow::m_rowHeader上设置LOCK标识，在步骤12中清除access::m_localInsertRow::m_rowHeader上的LOCK标识；


# 非并发删除操作逻辑
下面的分析基于以下假设：
- 事务中只包含一个删除操作(删除一个已经存在的行)，没有任何其它操作
- 删除过程中没有任何其它事务与之并发
- 除了主键索引之外，还有二级索引
- 隔离级别是REPEATABLE_READ
```
MOTExecForeignDelete
    - TxnManager::RowLookup
        - TxnManager::AccessLookup
            - 在当前的access manager中没有找到关于待删除的行的操作
        - 创建关于待删除行的access，并添加到access manager中
            - 获取待删除的行对应的sentinel和row
            - 创建关于待删除行的access
            - 创建一个新的row，并设置给access::m_localRow
            - 将待删除的行的数据拷贝到access::m_localRow中
                - 只拷贝了待删除的行中的m_data部分
            - 设置access::m_localRow::m_pSentinel为待删除行的m_pSentinel
            - 【注意：这里不是将access::m_type设置为delete】设置access::m_type为read
            - 设置access::m_tid为待删除行的commit sequence number
            - 设置access::m_origSentinel为待删除行对应的sentinel
            - 将(access::m_origSentinel, access)2元组添加到access manager中
    - MOTAdaptor::DeleteRow
        - TxnManager::DeleteLastRow
            - TxnAccess::UpdateRowState，将access的操作类型从read修改为delete
                - 【注意：这里将access::m_type设置为delete】设置access::m_type为delete
                - TxnAccess::GenerateDeletes
                    - 获取access::m_localRow所关联的table
                    - 遍历table中的每个二级索引，并对每个二级索引(记为secondaryIndex)处理如下：
                        - 根据access::m_localRow和table信息，生成secondaryIndex对应的key
                        - 在secondaryIndex中查找key对应的sentinel，记为secondarySentinel
                        - 为该二级索引生成一个关于“删除二级索引”的secondaryAccess
                            - 创建关于待删除行的secondaryAccess
                            - 创建一个新的row，并设置给secondaryAccess::m_localRow
                            - 将待删除的行的数据拷贝到secondaryAccess::m_localRow中
                                - 只拷贝了待删除的行中的m_data部分
                            - 设置secondaryAccess::m_localRow::m_pSentinel为待删除行的m_pSentinel
                            - 设置secondaryAccess::m_type为delete
                            - 设置secondaryAccess::m_tid为待删除行的commit sequence number
                        - 设置secondaryAccess::m_origSentinel为secondarySentinel
                        - 清除secondaryAccess::m_params中的PrimarySentinel标识，表示这是一个关于二级索引的操作
                        - 将(secondarySentinel, secondaryAccess)添加到access manager中
    
MOTAdaptor::Commit
    - OccTransactionManager::ValidateOcc
        - OccTransactionManager::LockHeaders
            - 遍历access manager中的每个2元组(sentinel, access)，对每个2元组处理如下：
                - 在access::m_origSentinel上加锁，也就是设置LOCK标识
            - OccTransactionManager::QuickHeaderValidation
                - 检查access所关联的全局row中的commit sequence number跟access::m_tid一样，如果不一样，则当说明其它事务已经修改了这个row，并且已经成功的commit了该更新，所以当前事务必须abort
        - OccTransactionManager::ValidateReadSet
            - 在当前上下文中只有insert操作，所以什么都不做
        - OccTransactionManager::ValidateWriteSet
            - 遍历access manager中的每个2元组(sentinel, access)，对每个2元组处理如下：
                - 如果是read类型的access，或者是没有PrimarySentinel标识的access，则略过
                - 检查access所关联的全局row中记录的commit sequence number和access::m_tid是否相同，如果不同，就abort当前的事务
                    - 在当前上下文中显然不会不同，所以不会abort当前的事务
    - TxnManager::CommitInternal
        - 分配commit sequence number
        - 记录redo日志
        - TxnManager::WriteDDLChanges
            - 当前上下文中没有ddl相关的变更
        - OccTransactionManager::WriteChanges
            - OccTransactionManager::LockRows
                - 遍历access manager中的每个2元组(sentinel, access)，对每个2元组处理如下：
                    - 如果是read类型的access，或者是没有PrimarySentinel标识的access，则略过
                    - 在access所关联的全局row(删除操作之前的那个row)上加锁(设置LOCK标识)
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中每个元组中的access类型都是delete，对每个2元组处理如下：
                - 设置access::m_origSentinel中的DIRTY标识
                - 如果access中设置了PrimarySentinel标识：
                    - 将access所关联的全局row中的commit sequence number设置为当前事务的commit sequence number
                    - 在access所关联的全局row上设置LOCK标识(和更新commit sequence number一起原子性的发生)
                    - 在access所关联的全局row上设置ABSENT标识(和更新commit sequence number一起原子性的发生)
                    - 如果access关联的全局row没有stable版本，则还会在access所关联的全局row上设置LATEST_VERSION标识(和更新commit sequence number一起原子性的发生)
                
MOTAdaptor::EndTransaction
    - TxnManager::EndTransaction
        - OccTransactionManager::ReleaseLocks
            - OccTransactionManager::ReleaseHeaderLocks
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中每个元组中的access类型都是delete，对每个2元组处理如下：
                    - 清除access::m_origSentinel上的LOCK标识
            - OccTransactionManager::ReleaseRowsLocks
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中每个元组中的access类型都是delete，对每个2元组处理如下：
                    - 如果该access中设置了PrimarySentinel标识：
                        - 清除access所关联的全局row上的LOCK标识
        - OccTransactionManager::CleanRowsFromIndexes
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中每个元组中的access类型都是delete，对每个2元组处理如下：
                - TxnManager::RemoveKeyFromIndex
                    - 如果access所关联的全局row中没有stable版本，则调用Table::RemoveKeyFromIndex进一步处理
                        - 减少sentinel的引用计数
                        - 如果引用计数变为0，则删除相关的索引
                            - Index::IndexRemove执行删除
                            - 通知gc回收被删除的row
                    - 否则，不删除索引，因为stable版本的记录需要使用该索引
                        - 那么这个索引在何时被删除呢？
```

将该过程整理如下(暂忽略二级索引)：
1. 在MOTExecForeignDelete -> TxnManager::RowLookup中为待删除的行row创建一个access，并添加到access manager中
	- 1.1 在access::m_localRow中将row的数据拷贝过来
	- 1.2 设置access::m_localRow::m_pSentinel = row::m_pSentinel
	- 1.3 设置access对应的操作类型是read
	- 1.4 设置access::m_tid = row对应的commit sequence nunber
	- 1.5 设置access::m_origSentinel = row::m_pSentinel
	- 1.6 将(access::m_origSentinel, access)2元组添加到access manager中
2. 在MOTExecForeignDelete -> MOTAdaptor::DeleteRow中将access的操作类型从read修改为delete
3. 在MOTAdaptor::Commit -> OccTransactionManager::ValidateOcc -> OccTransactionManager::LockHeaders中：
	- 3.1 在access::m_origSentinel上设置LOCK标识
	- 3.2 在access所关联的全局row中的commit sequence number和access::m_tid相比是否发生了变化，如果变化了，则当前事务必须abort
4. 在MOTAdaptor::Commit -> OccTransactionManager::ValidateOcc -> OccTransactionManager::ValidateWriteSet中：
	- 4.1 在access所关联的全局row中的commit sequence number和access::m_tid相比是否发生了变化，如果变化了，则当前事务必须abort
5. 在MOTAdaptor::Commit -> TxnManager::CommitInternal中：
	- 5.1 为当前事务分配commit sequence number
	- 5.2 记录redo日志
	- 5.3 在OccTransactionManager::WriteChanges中：
		- 5.3.1 在OccTransactionManager::LockRows中，在access所关联的全局row上设置LOCK标识
		- 5.3.2 设置access::m_origSentinel中的DIRTY标识
		- 5.3.3 将access所关联的全局row中的commit sequence number设置为5.1中所分配的commit sequence number
		- 5.3.4 在access所关联的全局row上设置LOCK标识(和更新commit sequence number一起原子性的发生)
		- 5.3.5 在access所关联的全局row上设置ABSENT标识(和更新commit sequence number一起原子性的发生)
		- 5.3.6 如果access关联的全局row没有stable版本，则还会在access所关联的全局row上设置LATEST_VERSION标识(和更新commit sequence number一起原子性的发生)
6. 在MOTAdaptor::EndTransaction -> TxnManager::EndTransaction -> OccTransactionManager::ReleaseLocks中：
	- 6.1 在OccTransactionManager::ReleaseHeaderLocks中，清除access::m_origSentinel上的LOCK标识
	- 6.2 在OccTransactionManager::ReleaseRowsLocks中，清除access所关联的全局row上的LOCK标识
7. 在MOTAdaptor::EndTransaction -> TxnManager::EndTransaction -> OccTransactionManager::CleanRowsFromIndexes中：
	- 7.1 如果access所关联的全局row中没有stable版本：
		- 7.1.1 减少access::m_origSentinel中的引用计数
		- 7.1.2 如果引用计数变为0，则删除相关的索引
			- 7.1.2.1 Index::IndexRemove执行删除
			- 7.1.2.2 通知gc回收被删除的row
	- 7.2 否则，不删除索引，因为stable版本的记录需要使用该索引
		- 那么这个索引在何时被删除呢？
		

说明：
- 在步骤1.1中拷贝到access::m_localRow中的数据没有起到什么作用
- 在步骤1.3中设置access的操作类型是read，而在步骤2中将之修改为delete，是为了处理二级索引的删除，因为在1.3中只添加了“删除主键索引”相关的access，而没有添加“删除二级索引”相关的access；
- 在步骤3中会在access::m_origSentinel上设置LOCK标识，在步骤6.1中会清除access::m_origSentinel上的LOCK标识
- 在步骤5.3.1中会在access所关联的全局row上设置LOCK标识，在步骤6.2中会清除access所关联的全局row上的LOCK标识
- 在步骤5.3.2中会设置access::m_origSentinel中的DIRTY标识，在后续没有被清除，原因是，如果删除成功，则无需清除，而一旦进入5.3就认为事务一定会成功？
- 在步骤5.3.4中会在access所关联的全局row上设置LOCK标识，在步骤5.3.1中也会在access所关联的row上设置LOCK标识，这两者的关系是这样的，在5.3.1和5.3.4之间对access所关联的row的commit sequence number进行了更新(更新为当前事务的commit sequence number)，5.3.1中是在旧的commit sequence number上加锁，而5.3.4是在新的commit sequence number上加锁，又因为更新commit sequence number和加锁操作是原子的，所以5.3.4中的锁可以认为是5.3.1中的锁的延续
- 在步骤5.3.5中在access所关联的全局row上设置的ABSENT标识，在后续没有被清除，原因是，如果删除成功，则无需清除，而一旦进入5.3就认为事务一定会成功？
- 在步骤5.3.6中在access所关联的全局row上设置的LATEST_VERSION标识，在后续没有被清除，原因是，如果删除成功，则无需清除，而一旦进入5.3就认为事务一定会成功？
- 在步骤7中，只有access所关联的全局row没有stable版本的情况下，才会删除，那么有stable版本的情况下何时被删除呢？理论上stable版本不再有效的时候就可以删除了？
- 在步骤7.1.1中，只有access::m_origSentinel中的引用计数降为0的时候才会删除
- 在步骤7.1.2中，删除过程是：先删除索引，然后通知gc回收被删除的row


# 并发插入情况下的逻辑1
下面的分析基于以下假设：
- 事务中只包含一个插入操作，没有任何其它操作
- 插入之前已经有另一个事务2已经插入了一个相同的行，并且成功commit
- 只有主键索引，没有任何二级索引
- 隔离级别是REPEATABLE_READ

```
MOTExecForeignInsert
    - MOTAdaptor::InsertRow
    
MOTAdaptor::InsertRow
    # 分配一个新的Row
    - MOT::Row* row = table->CreateNewRow()
    # 将要插入的数据打包到新分配的Row中
    - newRowData = const_cast<uint8_t*>(row->GetData());
      PackRow(slot, table, fdwState->m_attrsUsed, newRowData)
    - 生成主键primaryKey，并将(row, 主键索引, primaryKey)三元组添加到insert manager中
    - 获取insert manager中的每一个三元组(row, index, key)，当前上下文中insert manager中实际上只有一个三元组，并对该三元组处理如下：
        - 创建一个sentinel，设置DIRTY标识，设置live版本为null，设置stable版本为null
        - 尝试将(primaryKey, sentinel)插入到index中，但是因为index中已经存在相同的primaryKey，所以插入失败，会返回已经存在的sentinel，记为existedSentinel，它是由事务2创建并插入的
            - 增加existedSentinel的引用计数
            - 释放前面创建的sentinel，后续使用的是existedSentinel
        - 设置row的ABSENT标识，设置row::m_pSentinel为existedSentinel
        - 因为事务2已经成功提交，所以existedSentinel::isCommitted()为true
            - 去access manager中查找是否存在关于existedSentinel的操作
                - 因为当前事务中只有插入操作，不包含其它操作，所以返回NOT_FOUND
                - 销毁当前三元组对应的row
                - 减少existedSentinel的引用计数
        - 当前事务被abort
```    

可见，在当前上下文中，事务2的插入操作已经commit的情况下，当前事务的插入操作会abort。


# 并发插入情况下的逻辑2
下面的分析基于以下假设：
- 事务中只包含一个插入操作，没有任何其它操作
- 插入之前已经有另一个事务2正在执行插入过程中，但还没有commit
    - 在当前事务执行MOTAdaptor::InsertRow过程中事务2没有commit
    - 在当前事务执行MOTAdaptor::Commit的时候事务2已经在sentinel上加了锁
- 只有主键索引，没有任何二级索引
- 隔离级别是REPEATABLE_READ

```
MOTExecForeignInsert
    - MOTAdaptor::InsertRow
    
MOTAdaptor::InsertRow
    # 分配一个新的Row
    - MOT::Row* row = table->CreateNewRow()
    # 将要插入的数据打包到新分配的Row中
    - newRowData = const_cast<uint8_t*>(row->GetData());
      PackRow(slot, table, fdwState->m_attrsUsed, newRowData)
    - 生成主键primaryKey，并将(row, 主键索引, primaryKey)三元组添加到insert manager中
    - 获取insert manager中的每一个三元组(row, index, key)，当前上下文中insert manager中实际上只有一个三元组，并对该三元组处理如下：
        - 创建一个sentinel，设置DIRTY标识，设置live版本为null，设置stable版本为null
        - 尝试将(primaryKey, sentinel)插入到index中，但是因为index中已经存在相同的primaryKey，所以插入失败，会返回已经存在的sentinel，记为existedSentinel，它是由事务2创建并插入的
            - 增加existedSentinel的引用计数
            - 释放前面创建的sentinel，后续使用的是existedSentinel
        - 设置row的ABSENT标识，设置row::m_pSentinel为existedSentinel
        - 假设现在事务2还没有提交成功，则existedSentinel::isCommitted()为false
            - 分配一个access，并将(existedSentinel, access)2元组添加到access manager中：
                - 分配access
                - 清除access::m_params中的RowCommitted标识
                - 设置access::m_localInsertRow为row
                - 设置access::m_type为insert
                - 设置access::m_tid为0
                - 设置access::m_origSentinel为existedSentinel
                - 设置access::m_params中的PrimarySentinel和UniqueIndex标识
                - 将(sentinel, access)2元组添加到access manager中

MOTAdaptor::Commit
    - OccTransactionManager::ValidateOcc
        - OccTransactionManager::LockHeaders
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中实际上只有1个这样的2元组，对这个2元组处理如下：
                - 在access::m_origSentinel上加锁，也就是设置LOCK标识
                    - 因为在当前事务执行MOTAdaptor::Commit的时候事务2已经在access::m_origSentinel上加了锁，所以当前事务只能等待事务2释放access::m_origSentinel上的锁
                        - 当事务2成功提交之后，就会释放access::m_origSentinel上的锁
            - OccTransactionManager::QuickHeaderValidation
                - 发现access::m_origSentinel上的DIRTY标识已经被清除了，也就是事务2已经成功commit
                    - 当前事务abort
```  

可见，在当前上下文中，事务2先向index中添加了一个sentinel，当前事务在插入的时候会共用该sentinel，在当前事务执行commit的时候，会尝试在sentinel上加锁，但是发现事务2已经加锁了，必须等待事务2释放锁，而事务2释放锁的时候，要么是事务2已经成功提交了，要么是事务2 abort了，如果事务2成功提交了，则当前事务必须abort。

# 非并发先删除后插入操作的逻辑
下面的分析基于以下假设：
- 事务中先执行删除操作(删除一个已经存在的行)，后执行插入操作(重新插入被删除的行)，除此之外，没有其它操作
- 事务过程中没有任何其它事务与之并发
- 只有主键索引，没有任何二级索引
- 隔离级别是REPEATABLE_READ

```
MOTExecForeignDelete
    - TxnManager::RowLookup
        - TxnManager::AccessLookup
            - 在当前的access manager中没有找到关于待删除的行的操作
        - 创建关于待删除行的access，并添加到access manager中
            - 获取待删除的行对应的sentinel和row
            - 创建关于待删除行的access
            - 创建一个新的row，并设置给access::m_localRow
            - 将待删除的行的数据拷贝到access::m_localRow中
                - 只拷贝了待删除的行中的m_data部分
            - 设置access::m_localRow::m_pSentinel为待删除行的m_pSentinel
            - 【注意：这里不是将access::m_type设置为delete】设置access::m_type为read
            - 设置access::m_tid为待删除行的commit sequence number
            - 设置access::m_origSentinel为待删除行对应的sentinel
            - 将(access::m_origSentinel, access)2元组添加到access manager中
    - MOTAdaptor::DeleteRow
        - TxnManager::DeleteLastRow
            - TxnAccess::UpdateRowState，将access的操作类型从read修改为delete
                - 【注意：这里将access::m_type设置为delete】设置access::m_type为delete
                - TxnAccess::GenerateDeletes
                    - 生成关于“删除二级索引”的access
                    - 在当前上下文中没有二级索引，所以不会执行

MOTExecForeignInsert
    - MOTAdaptor::InsertRow
        # 分配一个新的Row
        - MOT::Row* row = table->CreateNewRow()
        # 将要插入的数据打包到新分配的Row中
        - newRowData = const_cast<uint8_t*>(row->GetData());
          PackRow(slot, table, fdwState->m_attrsUsed, newRowData)
        - 生成主键primaryKey，并将(row, 主键索引, primaryKey)三元组添加到insert manager中
        - 获取insert manager中的每一个三元组(row, index, key)，当前上下文中insert manager中实际上只有一个三元组，并对该三元组处理如下：
            - 尝试为当前的插入操作分配一个sentinel并添加到主键索引中
                - 但是因为待插入的key已经存在于主键索引中，所以直接返回主键索引中已经存在的sentinel，记为existedSentinel
            - 设置row的ABSENT标识
            - 设置row::m_pSentinel为existedSentinel
            - 在当前事务的access manager中查找是否存在已existedSentinel为key的access
                - 因为当前事务已经在access manager中为插入操作之前的删除操作添加了一个access，所以会找到
            - 因为找到的access是关于删除操作的，说明在插入操作之前发生了删除操作，于是将删除操作提升为插入操作(promote delete to insert)
                - 将查找到的access的操作类型从delete修改为insert
                - 在access::m_params中设置UPGRADE_INSERT标识
                - 【注意：此时access::m_localRow中已经在执行删除操作过程中将待删除的行的数据拷贝进去了】设置access::m_auxRow为之前为insert新分配的row，其中已经保存了待插入的数据
                    - 实际上是将删除操作和插入操作合并为插入操作了，在当前上下文中，access manager中只有一个操作类型为insert的access了

MOTAdaptor::Commit
    - OccTransactionManager::ValidateOcc
        - OccTransactionManager::LockHeaders
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
                - 在access::m_origSentinel上加锁，也就是设置LOCK标识
                - 在access::m_auxRow::m_rowHeader上加锁，也就是设置LOCK标识
            - OccTransactionManager::QuickHeaderValidation
                - 检查access所关联的全局row中的commit sequence number跟access::m_tid是否一样，如果不一样，则当说明其它事务已经修改了这个row，并且已经成功的commit了，所以当前事务必须abort
        - OccTransactionManager::ValidateReadSet
            - 在当前上下文中只有insert操作，所以什么都不做
        - OccTransactionManager::ValidateWriteSet
            - 遍历access manager中的每个2元组(sentinel, access)，对每个2元组处理如下：
                - 如果是read类型的access，或者是没有PrimarySentinel标识的access，则略过
                - 检查access所关联的全局row中记录的commit sequence number和access::m_tid是否相同，如果不同，就abort当前的事务
                    - 在当前上下文中显然不会不同，所以不会abort当前的事务
    - TxnManager::CommitInternal
        - 分配commit sequence number
        - 记录redo日志
        - TxnManager::WriteDDLChanges
            - 当前上下文中没有ddl相关的变更
        - OccTransactionManager::WriteChanges
            - OccTransactionManager::LockRows
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
                    - 如果是read类型的access，或者是没有PrimarySentinel标识的access，则略过
                    - 在access所关联的全局row(access::m_origSentinel对应的row)上加锁(设置LOCK标识)
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
                - 如果access中设置了PrimarySentinel标识：
                    - 将access所关联的全局row中的commit sequence number设置为当前事务的commit sequence number
                    - 在access所关联的全局row上设置LOCK标识(和更新commit sequence number一起原子性的发生)
                    - 在access所关联的全局row上设置LATEST_VERSION标识(和更新commit sequence number一起原子性的发生)
                    - 在access::m_auxRow上设置commit sequence number
                    - 在access::m_auxRow上清除ABSENT标识
        - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
            - 将access所关联的全局row保存到access::m_localInsertedRow中
            - 设置access所关联的全局row为access::m_auxRow
            - 通知gc来回收access::m_localInsertedRow中记录的row(也就是access之前关联的那个旧的row)
        - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
            - 清除access::m_origSentinel中的DIRTY标识
                
MOTAdaptor::EndTransaction
    - TxnManager::EndTransaction
        - OccTransactionManager::ReleaseLocks
            - OccTransactionManager::ReleaseHeaderLocks
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
                    - 清除access::m_origSentinel上的LOCK标识
            - OccTransactionManager::ReleaseRowsLocks
                - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
                    - 如果该access中设置了PrimarySentinel标识：
                        - 清除access所关联的全局row上的LOCK标识
                        - 清除access::m_localInsertedRow中记录的row(也就是access之前关联的那个旧的row)上的LOCK标识
        - OccTransactionManager::CleanRowsFromIndexes
            - 遍历access manager中的每个2元组(sentinel, access)，在当前上下文中只有一个操作类型为insert的access，对每个2元组处理如下：
                - 只对删除类型的access进行处理，当前上下文中没有要做的事情
```     

将该过程整理如下(暂忽略二级索引)：
1. 在MOTExecForeignDelete -> TxnManager::RowLookup为删除操作创建access，并添加到access manager中
	- 1.1 会为该access创建一个新的row，并设置给access::m_localRow，并将待删除的行的数据拷贝到access::m_localRow之前
	- 1.2 将access::m_localRow::m_pSentinel为待删除行的m_pSentinel
	- 1.3 将access的操作类型设定为read
	- 1.4 设置access::m_tid为待删除行的commit sequence number
	- 1.5 设置access::m_origSentinel为待删除行对应的sentinel
	- 1.6 将(access::m_origSentinel, access)2元组添加到access manager中
2. 在MOTExecForeignDelete -> MOTAdaptor::DeleteRow中，将access的的操作类型从read修改为delete
3. 在MOTExecForeignInsert -> MOTAdaptor::InsertRow中：
	- 3.1 分配一个新的Row，并将要插入的数据打包到新分配的Row中
	- 3.2 尝试为当前的插入操作分配一个sentinel并添加到主键索引中，但是因为待插入的key已经存在于主键索引中，所以直接返回主键索引中已经存在的sentinel，这个sentinel跟步骤1.2中的access::m_localRow::m_pSentinel，以及步骤1.5中的access::m_origSentinel都是同一个sentinel
	- 3.3 设置row的ABSENT标识 
	- 3.4 设置row::m_pSentinel为步骤3.2中的sentinel
	- 3.5 将access manager中的与删除操作相关的access从删除操作提升为插入操作
		- 3.5.1 将access的操作类型从delete修改为insert
		- 3.5.2 在access::m_params中设置UPGRADE_INSERT标识(表示这个access是从delete操作提升而来的insert操作)
		- 3.5.3 设置access::m_auxRow为步骤3.1中新分配的row
4. 在MOTAdaptor::Commit -> OccTransactionManager::ValidateOcc中：
	- 4.1 在OccTransactionManager::LockHeaders中对access manager中那个唯一的access进行处理：
		- 4.1.1 在access::m_origSentinel上加锁，也就是设置LOCK标识
		- 4.1.2 在access::m_auxRow::m_rowHeader(也就是插入之后新的row)上加锁，也就是设置LOCK标识
		- 4.1.3 检查access所关联的全局row中的commit sequence number跟access::m_tid是否一样，如果不一样，则当说明其它事务已经修改了这个row，并且已经成功的commit了，当前事务必须abort
	- 4.2 在OccTransactionManager::ValidateWriteSet中对access manager中那个唯一的access进行处理：
		- 4.2.1 检查access所关联的全局row中的commit sequence number跟access::m_tid是否一样，如果不一样，则当说明其它事务已经修改了这个row，并且已经成功的commit了，当前事务必须abort
5. 在MOTAdaptor::Commit -> TxnManager::CommitInternal中：
	- 5.1 分配commit sequence number
	- 5.2 记录redo日志
	- 5.3 在OccTransactionManager::WriteChanges中：
		- 5.3.1 在OccTransactionManager::LockRows中：
			- 5.3.1.1 对access manager中那个唯一的access所关联的全局row(删除操作之前的那个row)上加锁，也就是设置LOCK标识
		- 5.3.2 对access manager中那个唯一的access进行处理：
			- 5.3.2.1 将access所关联的全局row(删除操作之前的那个row)中的commit sequence number设置为当步骤5.1中分配的commit sequence number
			- 5.3.2.2 在access所关联的全局row(删除操作之前的那个row)上设置LOCK标识(和更新commit sequence number一起原子性的发生)
			- 5.3.2.3 在access所关联的全局row(删除操作之前的那个row)上设置LATEST_VERSION标识(和更新commit sequence number一起原子性的发生)
			- 5.3.2.4 在access::m_auxRow上设置commit sequence number
			- 5.3.2.5 在access::m_auxRow上清除ABSENT标识
	- 5.4 对access manager中那个唯一的access进行处理：
		- 5.4.1 将access所关联的全局row(也就是删除操作之前的那个row)保存到access::m_localInsertedRow中
		- 5.4.2 设置access所关联的全局row为access::m_auxRow(也就是插入操作中创建那个row)
		- 5.4.3 通知gc来回收access::m_localInsertedRow中记录的row(也就是删除操作之前的那个row)
	- 5.5 对access manager中那个唯一的access进行处理：
		- 5.5.1 清除access::m_origSentinel上的DIRTY标识
6. 在MOTAdaptor::EndTransaction -> TxnManager::EndTransaction中：
	- 6.1 在OccTransactionManager::ReleaseLocks中：	
		- 6.1.1 在OccTransactionManager::ReleaseHeaderLocks中：
			- 6.1.1.1 对access manager中那个唯一的access进行处理，清除access::m_origSentinel上的LOCK标识
		- 6.1.2 在OccTransactionManager::ReleaseRowsLocks中：
			- 6.1.2.1 对access manager中那个唯一的access进行处理：
				- 6.1.2.1.1 清除access所关联的全局row上的LOCK标识
				- 6.1.2.1.2 清除access::m_localInsertedRow中记录的row(也就是删除操作之前的那个row)上的LOCK标识
				
说明：
- 在步骤1中会为删除操作创建access，并设置access的操作类型为read，但是在步骤2中会将操作类型修改为delete，这样做的目的是为了处理二级索引(我们在当前上下文中没有二级索引，所以没有看到相关的处理)；
- 在步骤1.4中设置access::m_tid为删除操作之前的row中记录的commit sequence number，在步骤4.1.3和4.2.1中会检查access所关联的全局row(也就是删除操作之前的那个row)中的commit sequence number和access::m_tid是否一样，如果不一样，就会终止事务
- 在步骤1.4中设置access::m_tid为删除操作之前的row中记录的commit sequence number，在步骤5.3.2.1中则将将access所关联的全局row中的commit sequence number设置为当前事务的commit sequence number
- 在步骤3.1中会为插入操作分配一个新的row，并执行：
	- 在步骤3.1中将要插入的数据打包到新分配的row中
	- 在步骤3.3中设置该row的ABSENT标识，在步骤5.3.2.5中清除该row上的ABSENT标识
	- 在步骤3.4中将该row::m_pSentinel设置为删除之前的行对应的sentinel，也就是步骤1.5中的那个access::m_origSentinel
- 在步骤3.5中不会为插入操作创建新的access，而是将步骤1中为删除操作创建的的access从删除操作提升为插入操作的access
	- 在步骤3.5.1中将access的操作类型从delete修改为insert
	- 在步骤3.5.2中将access::m_params设置UPGRADE_INSERT标识(表示这个access是从delete操作提升而来的insert操作)
	- 在步骤3.5.3中设置access::m_auxRow为步骤3.1中为插入操作新分配的row
		- 在步骤5.4中会将access::m_auxRow设置为access所关联的新的row，并回收删除操作之前的那个旧的row
- 在步骤4.1.1中对access::m_origSentinel上设置LOCK标识，在步骤6.1.1.1中会清除access::m_origSentinel上的LOCK标识
- 在步骤4.1.2中对access::m_auxRow::m_rowHeader上设置LOCK标识，在步骤6.1.2.1.1中清除access所关联的全局row上的LOCK标识
- 在步骤5.3.1.1中对access所关联的旧的row设置LOCK标识
	- 不让其它操作并对改行进行修改
- 在步骤5.3.2中对access所关联的旧的row进行处理：
	- 设置commit sequence number
	- 设置LOCK标识
	- 设置LATEST_VERSION标识
- 在步骤5.3.2.4中对access::m_auxRow设置commit sequence number
- 在步骤5.3.2.5中对access::m_auxRow清除ABSENT标识，对应于在步骤3.3中设置该row的ABSENT标识
- 在步骤5.5.1中清除access::m_origSentinel上的DIRTY标识，但是在当前上下文中并没有设置access::m_origSentinel上的DIRTY标识？？？
- 在步骤6.1.1.1中清除access::m_origSentinel上的LOCK标识，对应于步骤4.1.1中对access::m_origSentinel上设置LOCK标识
- 在步骤6.1.2.1.1中清除access所关联的新的row上的LOCK标识，对应于步骤4.1.2中对access::m_auxRow::m_rowHeader上设置LOCK标识
- 重要标识的设置和清除：
	- 在步骤3.3设置插入操作之后的row上的ABSENT标识
	- 在步骤4.1.1在access::m_origSentinel上设置LOCK标识
	- 在步骤4.1.2中对access::m_auxRow::m_rowHeader(也就是插入之后新的row)上设置LOCK标识
	- 在步骤5.3.1中在access所关联的全局row(删除操作之前的那个row)上设置LOCK标识
	- 在步骤5.3.2中在access所关联的全局row(删除操作之前的那个row)上设置LOCK标识
		- 步骤5.3.2可以看做是步骤5.3.1的延续，因为5.3.2中还需设置新的commit sequence number，并且LOCK标识是在设置commit sequence number一起原子性设置的
	- 在步骤5.3.2.5中access::m_auxRow(代表插入操作之后的row)上清除ABSENT标识
	- 在步骤6.1.1.1中清除access::m_origSentinel上的LOCK标识
	- 在步骤6.1.2.1.1中清除access所关联的新的row(插入操作之后的row)上的LOCK标识




