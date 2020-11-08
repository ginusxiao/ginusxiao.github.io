# 提纲
[toc]

## 非并发更新操作基础逻辑
该分析基于以下假设：
- 事务中只包含一个更新操作，没有任何其它操作
- 更新之前行已经存在
- 插入过程中没有任何其它事务与之并发
- 只有主键索引，没有任何二级索引
- 隔离级别是REPEATABLE_READ

1. MOTIterateForeignScan
    - 1.1 从索引中查找待更新的行对应的sentinel
    - 1.2 检查当前事务的access manager中是否已经存在关于1.1中查找到的sentinel的access
        - 1.2.1 在当前上下文中，因为事务中只包含一个更新操作，所以肯定查找不到
        - 1.2.2 为当前事务的更新操作添加一个access到access manager中，并设置access的类型是read
            - 1.2.2.1 创建一个新的access
            - 1.2.2.2 为这个access分配一个新的row，并设置给access::m_localRow
            - 1.2.2.3 将更新前的数据拷贝到access::m_localRow中
            - 1.2.2.4 设置access::m_localRow::m_pSentinel为1.1中的sentinel
            - 1.2.2.5 设置access::m_type为read类型
            - 1.2.2.6 设置access::m_tid为待更新的行当前的tid
            - 1.2.2.7 设置access::m_origSentinel为1.1中的sentinel
            - 1.2.2.8 将(access::m_origSentinel, access)2元组添加到access manager中
        - 1.2.3 将待更新的数据拷贝到access::m_localRow中
2. MOTExecForeignUpdate
    - 2.1 TxnManager::RowLookup
        - 2.1.1 检查当前事务的access manager中是否已经存在关于1.1中查找到的sentinel的access
            - 2.1.1.1 在当前上下文中，因为在步骤1.2.2.8中已经添加了一个关于1.1中查找到的sentinel的access，所以会找到步骤1.2.2.1中创建的access
    - 2.2 MOTAdaptor::UpdateRow
        - 2.2.1 将步骤2.1.1中查找到的access的操作类型从read(在步骤1.2.2.5中设置)修改为write
        - 2.2.2 采用待更新的数据来更新access::m_localRow(在步骤1.2.2.3中已经将更新前的行的数据拷贝到其中)
        - 2.2.3 TxnManager::OverwriteRow
            - 2.2.3.1 设置access::m_modifiedColumns，表示哪些列被更新
3. MOTAdaptor::Commit -> TxnManager::Commit 
    - 3.1 OccTransactionManager::ValidateOcc
        - 3.1.1 OccTransactionManager::LockHeaders
            - 3.1.1.1 遍历access manager中每一个2元组(sentinel, access)，当前上下文中只有一个2元组，并对每个2元组处理如下：
                - 3.1.1.1.1 如果是access::m_type是read类型，则略过当前这个2元组，直接处理下一个
                - 3.1.1.1.2 在access::m_origSentinel上加锁，也就是设置LOCK标识
                - 3.1.1.1.3 检查access所关联的全局的行(不是当前事务本地的行，通过access::m_origSentinel::GetData()获得)中记录的tid和access::m_tid是否一样，如果不一样，说明access所关联的全局行已经被其它事务更新了，当前事务必须abort
        - 3.1.2 OccTransactionManager::ValidateReadSet
            - 3.1.2.1 检查access manager中所有操作类型为read的access，access::m_tid和access所关联的全局的行(不是当前事务本地的行，通过access::m_origSentinel::GetData()获得)是否一致，如果不一样，说明access所关联的全局行已经被其它事务更新了，当前事务必须abort
        - 3.1.3 OccTransactionManager::ValidateWriteSet
            - 3.1.3.1 检查access manager中所有操作类型不为read且对应的sentinel是PrimarySentinel的access，access::m_tid和access所关联的全局的行(不是当前事务本地的行，通过access::m_origSentinel::GetData()获得)是否一致，如果不一样，说明access所关联的全局行已经被其它事务更新了，当前事务必须abort
    - 3.2 TxnManager::CommitInternal
        - 3.2.1 为当前事务分配commit sequence number
        - 3.2.2 记录redo日志
        - 3.2.3 TxnManager::WriteDDLChanges
            - 3.2.3.1 应用ddl相关的更新，因为当前事务中不包括ddl操作，所以暂略
        - 3.2.4 OccTransactionManager::WriteChanges
            - 3.2.4.1 OccTransactionManager::LockRow
                - 3.2.4.1.1 遍历access manager中所有(sentinel, access)2元组，当前上下文中只有一个2元组，处理如下：
                    - 3.2.4.1.1.1 如果操作类型是read，或者sentinel不是PrimarySentinel类型，则忽略，直接处理下一个2元组
                    - 3.2.4.1.1.2 在access所关联的全局的行(不是当前事务本地的行，通过access::m_origSentinel::GetData()获得)上加锁，也就是设置LOCK标识
            - 3.2.4.2 遍历access manager中所有(sentinel, access)2元组，当前上下文中只有一个2元组，处理如下：
                - 3.2.4.2.1 获取access所关联的全局行(不是当前事务本地的行，通过access::m_origSentinel::GetData()获得)row
                - 3.2.4.2.2 将access::m_localRow的数据拷贝到row中
                - 3.2.4.2.3 设置row中记录的commit sequence number为步骤3.2.1中分配的commit sequence number
                - 3.2.4.2.4 在row上加LOCK标识(和步骤3.2.4.2.3中设置commit sequence number一起原子性的设置)
                    - 3.2.4.2.4.1 步骤3.2.4.2.4可以看做是步骤3.2.4.1.1.2的延续，因为步骤3.2.4.2.3和步骤3.2.4.2.4是原子性的完成的，所以在步骤3.2.4.1.1.2和3.2.4.2.4之间并没有丢失锁
4. MOTAdaptor::EndTransaction -> TxnManager::EndTransaction
    - 4.1 OccTransactionManager::ReleaseLocks
        - 4.1.1 OccTransactionManager::ReleaseHeaderLocks
            - 4.1.1.1 遍历access manager中所有(sentinel, access)2元组，当前上下文中只有一个2元组，处理如下：
                - 4.1.1.1.1 对于操作类型为read的access，直接略过，继续处理后续的2元组
                - 4.1.1.1.2 释放access::m_origSentinel上所加的锁
                    - 4.1.1.1.2.1 和步骤3.1.1.1.2中的加锁是对应的
        - 4.1.2 OccTransactionManager::ReleaseRowsLocks
            - 4.1.2.1 遍历access manager中所有(sentinel, access)2元组，当前上下文中只有一个2元组，处理如下：
                - 4.1.2.1.1 如果操作类型是read，或者sentinel不是PrimarySentinel类型，则忽略，直接处理下一个2元组
                - 4.1.2.1.2 释放access所关联的全局行(不是当前事务本地的行，通过access::m_origSentinel::GetData()获得)上的锁，即清除LOCK标识
                    - 4.1.2.1.2.1  跟步骤3.2.4.1.1.2和3.2.4.2.4中的加锁是对应的
    - 4.2 OccTransactionManager::CleanRowsFromIndexes
        - 4.2.1 该操作只对access manager中delete类型的操作起作用，在当前上下文中没有delete类型的操作
