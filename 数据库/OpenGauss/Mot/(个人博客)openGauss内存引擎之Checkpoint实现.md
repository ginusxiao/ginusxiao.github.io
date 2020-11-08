# 提纲
[toc]

# 简介
openGauss是华为基于PostgreSQL自研的数据库，它支持传统的基于硬盘的存储引擎，也支持基于内存的存储引擎，本文分析openGauss内存引擎(MOT，Memory Optimized Table)是如何实现全量Checkpoint的。下文会以MOT指代openGauss内存引擎。

MOT实现Checkpoint的原型是CALC算法，但是MOT对CALC算法进行了一些修改，下文中会用原始的CALC算法指代[CALC算法](https://dl.acm.org/doi/10.1145/2882903.2915966)论文中的实现，而用MOT的CALC算法指代MOT修改后的CALC算法。

# openGauss Checkpoint概览
我们先看一下openGauss数据更新的过程：
- 当更新某行记录时，检查相应的数据是否在内存buffer page中，如果在，则直接更新buffer page中的数据，如果不在，则将所需要的数据加载到内存buffer page中
- 将更新后的buffer page标记为dirty
- 在commit的时候，向WAL中写入变更记录
- 在后续某个时间，dirty pages被写入持久化存储

最后一个步骤实际上就是Checkpoint。openGauss会在以下时机触发Checkpoint：
- 用户手动执行Checkpoint命令
- 每隔checkpoint_timeout(默认300ms)时间周期性执行
- WAL中数据总量超过了max_wal_size(默认1G)
- 数据库backup
- 数据库shutdown
- 数据库recovery完毕

与Checkpoint相关的参数：
- checkpoint_timeout
周期性执行checkpoint的最大时间间隔，默认值是5分钟
- max_wal_size
两次Checkpoint之间最大的WAL数据总量，默认值是1GB
- checkpoint_completion_target
该参数表示checkpoint的完成时间占两次checkpoint之间时间间隔的比例，系统默认值是0.5，也就是说每个checkpoint需要在checkpoints间隔时间的50%内完成。
- checkpoint_flush_after
在Checkpoint的过程中，如果已经向Checkpoint文件中写入但是尚未被flush的数据超过checkpoint_flush_after，则强制执行flush

# MOT之Checkpoint实现
MOT之Checkpoint的实现主要包括2个方面，一是借用openGauss通用的Checkpoint框架，注册Checkpoint相关的事件回调，二是参考[CALC算法](https://dl.acm.org/doi/10.1145/2882903.2915966)实现Checkpoint。

## MOT之Checkpoint实现总览
![image](https://note.youdao.com/yws/public/resource/e6edfab05e5fc6afcd19b8f7e4006df6/xmlnote/851E927006BA4704AF5F03AE378E8B14/116380)

其中：
- Checkpoint Event表示openGauss通用Checkpoint框架中与Checkpoint相关的事件，主要包括EVENT_CHECKPOINT_CREATE_SNAPSHOT，EVENT_CHECKPOINT_SNAPSHOT_READY和EVENT_CHECKPOINT_BEGIN_CHECKPOINT
- rest，prepare，resolve，capture和complete表示CALC算法中Checkpoint的不同阶段
- switch condition表示CALC算法中Checkpoint的不同阶段之间的转换条件
- Action表示MOT的一些操作

MOT Checkpoint的阶段/状态变迁如下：
- 在rest阶段：
    - MOT等待EVENT_CHECKPOINT_CREATE_SNAPSHOT事件；
    - 当接收到EVENT_CHECKPOINT_CREATE_SNAPSHOT事件时：
        - 在DDL相关的锁上加锁，以阻止Vacuum table相关的操作发生
        - 等待所有在上一次Checkpoint的complete阶段开始的事务完成
        - 切换到prepare阶段
- 在prepare阶段：
    - MOT等待所有在rest阶段开始的事务完成
    - 切换到resolve阶段
- 在resolve阶段：
    - MOT不接收任何在该阶段开始的事务，事务线程会被阻塞，直到进入其它阶段
    - MOT等待所有在prepare阶段开始的事务完成
    - 切换到capture阶段
- 在capture阶段：
    - 对MOT RedoLogHandler加锁，以阻止MOT提交任何Redo log
    - MOT等待EVENT_CHECKPOINT_SNAPSHOT_READY事件
    - 当接收到EVENT_CHECKPOINT_SNAPSHOT_READY事件时：
        - 设置本次Checkpoint的consistency point，在consistency point之前的所有的更新都会被Checkpoint，consistency point之后的所有更新则不会被Checkpoint
    - 解除MOT RedoLogHandler上的锁
    - 等待EVENT_CHECKPOINT_BEGIN_CHECKPOINT事件
    - 当接收到EVENT_CHECKPOINT_BEGIN_CHECKPOINT事件时：
        - 对所有记录执行Checkpoint IO
    - 切换到complete阶段
- 在complete阶段：
    - 结束Checkpoint之前的相关工作：更新控制文件，创建map文件等
    - 交换Available和NotAvailable的值
    - 切换到rest阶段
    

## openGauss通用Checkpoint实现
当Checkpoint被触发时，openGauss会调用CreateCheckpoint去执行整个Checkpoint过程，这里我们展示全量Checkpoint的主要处理逻辑：
- 在CheckpointLock上加独占锁，确保任意时刻至多只有一个Checkpoint正在运行
- 调用EVENT_CHECKPOINT_CREATE_SNAPSHOT事件的回调，通知包括MOT在内的注册了Checkpoint相关事件回调的模块“Checkpoint被触发了”，这些模块接收到通知后，会进入Checkpoint前的准备阶段
- 获取本次Checkpoint对应的Consistency point，即本次Checkpoint对应的Redo LSN，在consistency point之前的所有的更新都会被Checkpoint，consistency point之后的所有更新则不会被Checkpoint，然后调用EVENT_CHECKPOINT_SNAPSHOT_READY事件的回调，通知包括MOT在内的注册了Checkpoint相关事件回调的模块“consistency point确定了”
    - 在WALInsertLock上加独占锁，确保在获取consistency point的过程中不会有Redo log写入WAL
    - 获取consistency poit，即本次Checkpoint对应的Redo LSN，它对应的就是当前已经为事务预留的最大的LSN
    - 调用EVENT_CHECKPOINT_SNAPSHOT_READY事件的回调，通知包括MOT在内的注册了Checkpoint相关事件回调的模块“consistency point确定了”，同时通知它们已经确定的consistency point是多少
    - 释放WALInsertLock上的独占锁，后续事务可以继续写Redo log到WAL了
- 刷出所有共享内存中的数据到磁盘并做文件同步，共享内存中的数据包括clog、subtrans、multixact、predicate、relationmap、buffer和2PC等相关数据
- 调用EVENT_CHECKPOINT_BEGIN_CHECKPOINT事件的回调，通知包括MOT在内的注册了Checkpoint相关事件回调的模块“可以执行Checkpoint了”，这些模块接收到通知之后，就开始执行Checkpoint
- 向WAL中添加关于本次Checkpoint的日志记录
- 释放CheckpointLock上的独占锁
```
void CreateCheckPoint(int flags)
{
    ...
    
    # 在CheckpointLock上加独占锁，确保任意时刻至多只有一个Checkpoint正在运行
    LWLockAcquire(CheckpointLock, LW_EXCLUSIVE);
    
    # 通知包括MOT在内的注册了Checkpoint相关事件回调的模块“Checkpoint被触发了”
    if (doFullCheckpoint) {
        CallCheckpointCallback(EVENT_CHECKPOINT_CREATE_SNAPSHOT, 0);
    }

    # 在WALInsertLock上加独占锁，需要等待其它已经持有WALInsertLock的锁的事务
    # 释放锁，然后才能加锁成功，一旦加锁成功，后续事务无法写Redo log到WAL中
    WALInsertLockAcquireExclusive();
    # 获取WAL中为事务预留的最大的LSN，在真正执行Checkpoint之前，这些事务的Redo
    # log会确保被sync到WAL文件中(而不是在WAL buffer中，见CheckPointGuts)
    curInsert = XLogBytePosToRecPtr(Insert->CurrBytePos);
    checkPoint.redo = curInsert;
    # 通知包括MOT在内的注册了Checkpoint相关事件回调的模块“consistency point确定了”
    if (doFullCheckpoint) {
        CallCheckpointCallback(EVENT_CHECKPOINT_SNAPSHOT_READY, checkPoint.redo);
    }
    
    # 释放WALInsertLock上的独占锁，后续事务可以继续写Redo log到WAL了
    WALInsertLockRelease();
    
    ...
    
    # 刷出所有共享内存中的数据到磁盘并做文件同步
    CheckPointGuts(checkPoint.redo, flags, doFullCheckpoint);
    
    # 通知包括MOT在内的注册了Checkpoint相关事件回调的模块“可以执行Checkpoint了”
    if (doFullCheckpoint) {
        CallCheckpointCallback(EVENT_CHECKPOINT_BEGIN_CHECKPOINT, 0);
    }    
    
    ...
    
    # 向WAL中添加关于本次Checkpoint的日志记录
    XLogRegisterData((char*)(&checkPointNew), sizeof(checkPointNew));
    recptr = XLogInsert(RM_XLOG_ID, shutdown ? XLOG_CHECKPOINT_SHUTDOWN : XLOG_CHECKPOINT_ONLINE);
    XLogFlush(recptr);
    
    ...
    
    # Checkpoint完成，释放锁
    LWLockRelease(CheckpointLock);
}
```

在上述逻辑中，openGauss为包括MOT在内的其它模块处理Checkpoint事件搭好了框架，这些模块只需要注册相应的事件回调即可，可以通过RegisterCheckpointCallback来注册回调。

openGauss为Checkpoint定义了4种事件，每种事件的作用如下：
```
typedef enum {
    # Checkpoint被触发
    EVENT_CHECKPOINT_CREATE_SNAPSHOT,
    # 已经确定了consistency point，在consistency point之前的所有的更新都会被Checkpoint，
    # consistency point之后的所有更新则不会被Checkpoint
    EVENT_CHECKPOINT_SNAPSHOT_READY,
    # 开始执行Checkpoint IO
    EVENT_CHECKPOINT_BEGIN_CHECKPOINT,
    # 终止Checkpoint
    EVENT_CHECKPOINT_ABORT
} CheckpointEvent;
```

## MOT注册Checkpoint相关的事件回调
MOT在初始化的时候注册了Checkpoint相关的事件回调MOTCheckpointCallback，在MOTCheckpointCallback中会根据Checkpoint相关的事件类型分别处理，最终都交由MOT中的CheckpointManager来处理：
- EVENT_CHECKPOINT_CREATE_SNAPSHOT最终交由CheckpointManager::CreateSnapShot处理
- EVENT_CHECKPOINT_SNAPSHOT_READY最终交由CheckpointManager::SnapshotReady处理
- EVENT_CHECKPOINT_BEGIN_CHECKPOINT最终交由CheckpointManager::BeginCheckpoint处理
- EVENT_CHECKPOINT_ABORT最终交由CheckpointManager::AbortCheckpoint处理
```
static void InitMOTHandler()
{
    ...

    if (MOT::GetGlobalConfiguration().m_enableIncrementalCheckpoint == false) {
        if (!MOTAdaptor::m_callbacks_initialized) {
            ...

            if (MOT::GetGlobalConfiguration().m_enableCheckpoint) {
                # 注册Checkpoint相关的事件回调
                RegisterCheckpointCallback(MOTCheckpointCallback, NULL);
            }

            ...
        }

        ...
    }
}
```

## MOT对EVENT_CHECKPOINT_CREATE_SNAPSHOT事件的处理
当接收到EVENT_CHECKPOINT_CREATE_SNAPSHOT事件时，MOT会交给CheckpointManager::CreateSnapShot进行处理，主要逻辑如下：
- 创建一个唯一的Checkpoint ID
- 检查当前是否处于rest阶段
    - 如果不是，则提示错误并退出
    > 如果当前不处于rest阶段，则表明一定有Checkpoint正在运行，而在Checkpoint运行过程中，是不可能有其它Checkpoint被触发的
- 加DDL相关的锁，避免Checkpoint过程中发生Vacuum table(即回收垃圾数据)相关的操作
- 等待在前一次Checkpoint的complete阶段开始的事务完成
- 切换到prepare阶段
- 等待，直到进入capture阶段为止，也就是直到可以确定consistency point为止

```
bool CheckpointManager::CreateSnapShot()
{
    # 创建Checkpoint ID
    if (!CheckpointManager::CreateCheckpointId(m_id)) {
        ...
    }

    # EVENT_CHECKPOINT_CREATE_SNAPSHOT事件发生时，只应该处于rest阶段
    if (m_phase != CheckpointPhase::REST) {
        return false;
    }

    MOT::MOTEngine* engine = MOT::MOTEngine::GetInstance();
    # 避免Checkpoint过程中发生Vacuum table(即回收垃圾数据)相关的操作
    engine->LockDDLForCheckpoint();
    ResetFlags();

    # 等待在前一次Checkpoint的complete阶段开始的事务完成
    WaitPrevPhaseCommittedTxnComplete();

    # 切换到prepare阶段
    m_lock.WrLock();
    MoveToNextPhase();
    m_lock.WrUnlock();

    # 等待，直到进入capture阶段
    while (m_phase != CheckpointPhase::CAPTURE) {
        usleep(50000L);
    }

    return !m_errorSet;
}
```

## MOT对EVENT_CHECKPOINT_SNAPSHOT_READY事件的处理
当接收到EVENT_CHECKPOINT_SNAPSHOT_READY事件时，MOT会交给CheckpointManager::SnapshotReady进行处理，主要执行：
- 设置本次Checkpoint的consistency point，即Redo LSN
- 解除MOT RedoLogHandler上的锁(这个锁是在进入capture阶段的时候加的)

```
bool CheckpointManager::SnapshotReady(uint64_t lsn)
{
    if (m_phase != CheckpointPhase::CAPTURE) {
        m_errorSet = true;
    } else {
        # 设置Redo LSN
        SetLsn(lsn);
        # 解除MOT RedoLogHandler上的锁
        if (m_redoLogHandler != nullptr)
            m_redoLogHandler->WrUnlock();
    }
    return !m_errorSet;
}
```

## MOT对EVENT_CHECKPOINT_BEGIN_CHECKPOINT事件的处理
当接收到EVENT_CHECKPOINT_BEGIN_CHECKPOINT事件时，MOT会交给CheckpointManager::BeginCheckpoint进行处理，主要执行：
- 对每个Table中的每条记录执行Checkpoint IO(关于Checkpoint IO的具体执行将在后面单独讲解)
- 切换到complete阶段
- 更新Checkpoint控制文件
- 等待所有在capture阶段开始的事务完成
- 交换Available和NotAvailable的值
- 切换到rest阶段
- 解除DDL上的锁

```
bool CheckpointManager::BeginCheckpoint()
{
    # 对每个Table中的每条记录执行Checkpoint IO
    Capture();
    
    # 因为Checkpoint IO是由其它线程执行的，所以Capture()方法退出时并不一定完成了
    # 所有Table的所有记录的Checkpoint IO，只有当m_checkpointEnded被设置时，才表明
    # 所有Table的所有记录的Checkpoint IO都完成了
    while (!m_checkpointEnded) {
        usleep(100000L);
    }

    # 切换到complete阶段，这里在切换到complete阶段之前，没有等待再resolve阶段开始
    # 的事务执行完毕，是因为在MOT中不存在在resolve阶段开始的事务
    m_lock.WrLock();
    MoveToNextPhase();
    m_lock.WrUnlock();

    if (!m_errorSet) {
        # 更新Checkpoint控制文件
        CompleteCheckpoint(GetId());
    }

    # 等待所有在capture阶段开始的事务完成
    WaitPrevPhaseCommittedTxnComplete();

    # 切换到rest阶段
    m_lock.WrLock();
    MoveToNextPhase();
    m_lock.WrUnlock();

    MOT::MOTEngine* engine = MOT::MOTEngine::GetInstance();
    # 解除DDL上的锁
    engine->UnlockDDLForCheckpoint();
    return !m_errorSet;
}
```

## 事务Commit
在CALC算法中，对事务的处理与事务在哪个阶段开始以及事务在哪个阶段结束是息息相关的，比如在CALC原始的算法中，在prepare阶段开始的事务，如果在prepare阶段结束，则会被包含在本次Checkpoint中，而在prepare阶段开始的事务，如果在resolve阶段结束，则不会被包含在本次Checkpoint中。反过来事务的处理又会影响CALC算法中阶段的变迁，比如，从rest阶段切换到prepare阶段需要等待所有在上一次Checkpoint中complete阶段开始的事务都完成，从prepare阶段切换到resolve阶段需要等待所有在rest阶段开始的事务都完成，等等。

在MOT中所谓的事务开始和事务完成都是指的是事务commit开始和commit完成。因此，需要通过事务commit逻辑来理解MOT实现的CALC算法。MOT中事务commit的主要逻辑如下：
```
    # 1. 通知CheckpointManager事务commit开始
    CheckpointManager::BeginTransaction
    
    # 2. 向WAL中添加关于事务的redo log
    RedoLog::Commit
    
    # 3. 应用事务相关的更新
    # 3.1 应用DDL相关的更新
    TxnManager::WriteDDLChanges
    # 3.2 应用DML相关的更新
    RowHeader::WriteChangesToRow
    # 3.3 根据当前所处的CALC算法的阶段相应地设置stable版本
    CheckpointManager::ApplyWrite
    
    # 4. 通知CheckpointManager事务commit完成
    CheckpointManager::CommitTransaction
    CheckpointManager::TransactionCompleted
```

### 通知CheckpointManager事务commit开始
CALC算法需要记录一个事务是在哪个阶段开始的，这是由CheckpointManager::BeginTransaction完成的。通常在设置一个事务是在哪个阶段开始的时候，都会将其设置为CALC算法当前所在的阶段，但是MOT对标准的CALC算法进行了改进，不能有任何事务在resolve阶段开始，以确保在获取consistency point之前，除已经在prepare阶段开始的事务之外，不会有新的事务写WAL，否则如果允许事务在resolve阶段开始，则在进入capture阶段获取consistency point的时候可能有在resolve阶段开始的事务还没有执行完毕，需要等待一段时间才能获得consistency point。所以如果某个事务在resolve阶段调用CheckpointManager::BeginTransaction的话，该事务会一直阻塞直到进入capture阶段。

另外，MOT实现的CALC算法中，除了从capture阶段切换为complete阶段的前提条件不是所有在resolve阶段开始的事务都执行完毕以外(因为MOT不允许事务在resolve阶段开始)，其它所有的从A阶段切换为B阶段的前提条件都是所有在A阶段之前的那个阶段开始的事务都执行完毕。比如，从rest阶段切换到prepare阶段需要等待所有在上一次Checkpoint中complete阶段开始的事务都完成，从prepare阶段切换到resolve阶段需要等待所有在rest阶段开始的事务都完成，从resolve阶段切换到capture阶段需要等待所有在prepare阶段开始的事务都完成，从complete切换到rest阶段需要等待所有在capture阶段开始的事务都完成。为了记录某个阶段的事务是否完成，需要对事务进行计数，当事务开始的时候增加计数，当事务完成的时候减少计数，这就是CheckpointManager中m_counters的作用。从A阶段切换为B阶段的前提条件都是所有在A阶段之前的那个阶段开始的事务都执行完毕，还隐式的说明在任何一个阶段A，只可能存在在在阶段A之前的一个阶段开始的事务和在阶段A开始的事务。比如，在rest阶段，只可能存在在上一次Checkpoint的complete阶段开始的事务和在本次Checkpoint的rest阶段开始的事务，在prepare阶段，只可能存在在rest阶段开始的事务和在prepare阶段开始的事务。在CheckpointManager中借助一个bool变量m_cntBit来索引在当前阶段开始的事务的计数，!m_cntBit来索引在当前阶段之前的一个阶段开始的事务的计数。

```
void CheckpointManager::BeginTransaction(TxnManager* txn)
{
    m_lock.RdLock();
    # 事务不能在resolve阶段开始
    while (m_phase == CheckpointPhase::RESOLVE) {
        m_lock.RdUnlock();
        usleep(5000);
        m_lock.RdLock();
    }
    
    # 设置事务在哪个阶段开始
    txn->m_checkpointPhase = m_phase;
    txn->m_checkpointNABit = !m_availableBit;
    # 增加当前阶段开始的事务的计数
    m_counters[m_cntBit].fetch_add(1);
    m_lock.RdUnlock();
}
```

### 向WAL中添加关于事务的redo log
请参考我的另一篇博文[openGauss内存引擎WAL实现](http://note.youdao.com/noteshare?id=01b74bae146cc4d9d17a83dd53d669dd&sub=0B26CF2439F44736968E3E85AC239C83)

### 应用DML相关的更新
    
    
### 根据当前所处的CALC算法的阶段相应地设置stable版本
MOT中一个索引关联到一个Sentinel，一个Sentinel则可以关联到一个行的两个版本的记录，分别保存在Sentinel::m_status和Sentinel::m_stable中。其中Sentinel::m_status中保存的是live版本，而Sentinel::m_stable中保存的是stable版本。更新live版本是通过Sentinel::SetNextPtr方法实现的，而更新stable版本则是通过Sentinel::SetStable方法实现的。在Sentinel::m_stable中还预留了一个bit位用于记录CALC算法中用到的名为StableStatus的bit数组，可以通过Sentinel::SetStableStatus(bool val)来设置或者清除之，通过Sentinel::SetStableStatus()获取之。当然Sentinel::m_status中也预留了一些bit位用于记录live版本的记录的状态，但是这些状态主要是在事务并发控制中被用到。

先抛开对insert操作处理，对于update操作：
- 如果事务是在rest阶段开始，则该事务最迟在prepare阶段完成，一定会被包含在本次Checkpoint中，live版本可以直接作为stable版本，无需额外记录stable版本
- 如果事务是在prepare阶段开始，则该事务最迟在resolve阶段完成，一定会被包含在本次Checkpoint中，live版本可以直接作为stable版本，无需额外记录stable版本
    - MOT在发现StableStatus为!m_availableBit的时候，还是会设置stable版本的记录，但是在CheckpointManager::CommitTransaction的时候会删除之，好像没这个必要？
- 不会有事务在resolve阶段开始
- 如果事务是在capture阶段开始，则该事务不会被包含在本次Checkpoint中，如果事务不包含stable版本的记录，则必须记录stable版本，并且设置StableStatus
- 如果事务是在complete阶段开始，则该事务不会被包含在本次Checkpoint中，且本次Checkpoint中所有Table的所有记录的都执行了Checkpoint，则live版本可以直接作为stable版本，无需额外记录stable版本
```
bool CheckpointManager::ApplyWrite(TxnManager* txnMan, Row* origRow, AccessType type)
{
    CheckpointPhase startPhase = txnMan->m_checkpointPhase;
    Sentinel* s = origRow->GetPrimarySentinel();
    MOT_ASSERT(s);
    if (s == nullptr) {
        MOT_LOG_ERROR("No sentinel on row!");
        return false;
    }

    bool statusBit = s->GetStableStatus();
    switch (startPhase) {
        case REST:
            if (type == INS)
                s->SetStableStatus(!m_availableBit);
            break;
        case PREPARE:
            if (type == INS)
                s->SetStableStatus(!m_availableBit);
            else if (statusBit == !m_availableBit) {
                if (!CheckpointUtils::SetStableRow(origRow))
                    return false;
            }
            break;
        case RESOLVE:
        case CAPTURE:
            if (type == INS)
                s->SetStableStatus(m_availableBit);
            else {
                if (statusBit == !m_availableBit) {
                    if (!CheckpointUtils::SetStableRow(origRow))
                        return false;
                    s->SetStableStatus(m_availableBit);
                }
            }
            break;
        case COMPLETE:
            if (type == INS)
                s->SetStableStatus(!txnMan->m_checkpointNABit);
            break;
        default:
            MOT_LOG_ERROR("Unknown transaction start phase: %s", CheckpointManager::PhaseToString(startPhase));
    }

    return true;
}
```

### 通知CheckpointManager事务commit完成
CheckpointManager::CommitTransaction主要是处理在prepare阶段开始的事务，在标准的CALC算法中，prepare阶段开始的事务可能在prepare阶段完成，也可能在resolve阶段完成，但是MOT对标准的CALC算法进行了一些改进，即所有在prepare阶段开始的事务都被包含在本次Checkpoint中，无论这些事务是在哪个阶段完成的。
```
void CheckpointManager::CommitTransaction(TxnManager* txn, int writeSetSize)
{
    # 
    TxnOrderedSet_t& orderedSet = txn->m_accessMgr->GetOrderedRowSet();
    if (txn->m_checkpointPhase == PREPARE) {
        const Access* access = nullptr;
        for (const auto& ra_pair : orderedSet) {
            access = ra_pair.second;
            if (access->m_type == RD) {
                continue;
            }
            if (access->m_params.IsPrimarySentinel()) {
                MOT_ASSERT(access->GetRowFromHeader()->GetPrimarySentinel());
                CheckpointUtils::DestroyStableRow(access->GetRowFromHeader()->GetStable());
                access->GetRowFromHeader()->GetPrimarySentinel()->SetStable(nullptr);
                writeSetSize--;
            }
            if (!writeSetSize) {
                break;
            }
        }
    }
}
```

CheckpointManager::TransactionCompleted







在Checkpoint phase从complete切换到rest之前，会将available和not_available进行交互，这样原来StableStatus为available的就变为not_available了，而原来StableStatus为not_available的可能变为available了。我们看一种情况：一个事务在rest阶段开始，他插入了一行row，后来再也没有更新过这一行，那么StableStatus[row] = 0，也就是not_available，在第一次Checkpoint的complete阶段转换为rest阶段之前，会将not_available和available进行交换，这么一来它就变为available的了，但是并没有stable版本的记录。所以系统中存在StableStatus[row] = available，但是并没有stable版本的记录的情况。



MOT对CALC的改进：
1. 不存在在resolve阶段开始的事务，如果某个事务在resolve阶段开始，则它会等待，直到resolve阶段结束进入其它阶段；

```
void CheckpointManager::BeginTransaction(TxnManager* txn)
{
    m_lock.RdLock();
    # 如果当前是resolve阶段，则等待，直到resolve阶段结束，进入其它阶段
    while (m_phase == CheckpointPhase::RESOLVE) {
        m_lock.RdUnlock();
        usleep(5000);
        m_lock.RdLock();
    }
    
    # 设置事务在m_phase所在的当前阶段开始
    txn->m_checkpointPhase = m_phase;
    txn->m_checkpointNABit = !m_availableBit;
    # 增加在当前阶段开始的事务计数
    m_counters[m_cntBit].fetch_add(1);
    m_lock.RdUnlock();
}
```

2. 所有在prepare阶段开始的事务都被包含在本次Checkpoint中，无论这些事务是在哪个阶段完成的。