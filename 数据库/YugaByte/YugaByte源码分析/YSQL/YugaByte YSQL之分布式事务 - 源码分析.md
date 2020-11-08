# 提纲
[toc]

## PostgreSQL中事务相关的逻辑
下面以PostgresMain处理接收到的命令为例来分析事务：
```
PostgresMain
    # PostgresMain中的无限循环，每一轮都执行如下操作
    
    # 读取下一条command
    - ReadCommand
    # 处理当前读取到的command
    - yb_exec_simple_query_attempting_to_restart_read
        - exec_simple_query
            # 对每一个command都会执行，无论它是transaction 
            # block中的哪一条语句(包括BEGIN和COMMIT这种语句)
            - start_xact_command
                # 执行条件：xact_started为false，其中xact_started是一个全局静态bool变量，
                # 用于记录当前是否已经启动一个transaction
                - StartTransactionCommand
                    - 若全局的静态变量CurrentTransactionState中记录的TBlockState是TBLOCK_DEFAULT，则：
                        - StartTransaction
                            # 忽略PostgreSQL自身的一系列逻辑
                            - ...
                            - YBStartTransaction
                                - YBCPgTxnManager_BeginTransaction
                                    # 这里正是进入YQL逻辑
                                    # 获取当前的TransactionManager，并启动transaction
                                    - PgTxnManager::BeginTransaction(YBCGetPgTxnManager())
                                        - ResetTxnAndSession
                                        - txn_in_progress_ = true
                                        - StartNewSession
                                            # 创建YBSession，在YBSession中会关联到一个YBClient(通过
                                            # async_client_init_->client()获得)，以及一个
                                            # ConsistentReadPoint
                                            - session_ = std::make_shared<YBSession>(async_client_init_->client(), clock_)
                                            - session_->SetReadPoint(client::Restart::kFalse)
                                            - session_->SetForceConsistentRead(client::ForceConsistentRead::kTrue)
                                - YBCPgTxnManager_SetIsolationLevel
                                - YBCPgTxnManager_SetReadOnly
                                - YBCPgTxnManager_SetDeferrable
                        - 将全局的静态变量CurrentTransactionState中记录的TBlockState设置为TBLOCK_STARTED
                    - 若全局的静态变量CurrentTransactionState中记录的TBlockState是TBLOCK_INPROGRESS/TBLOCK_IMPLICIT_INPROGRESS/TBLOCK_SUBINPROGRESS，则保持当前状态，什么也不做
                    - 若全局的静态变量CurrentTransactionState中记录的TBlockState是其它状态，则打印日志提示信息
                # 设置全局静态bool变量xact_started为true，表示已经启动了transaction
                - xact_started = true
            - PortalRun
                - PortalRunMulti
                    # 如果YugaByte为PostgreSQL提供存储，则检查是否是单行事务
                    - is_single_row_modify_txn = YBCIsSingleRowModify(pstmt)
                    - 逐一处理portal->stmts中的每一条语句stmtlist_item，每一条语句的处理如下：
                        # 获取对应的PlannedStmt
                        - PlannedStmt *pstmt = lfirst_node(PlannedStmt, stmtlist_item)
                        - 如果是utility statement(如事务相关的begin，start，commit，rollback，savepoint相关的，create table，drop table，prepare，execute等)，则：
                            - PortalRunUtility
                                - ProcessUtility
                                    - standard_ProcessUtility
                                        - 如果是事务相关的语句
                                            - 如果是begin语句
                                                - BeginTransactionBlock
                                                    - 如果全局的静态变量CurrentTransactionState中记录的TBlockState是TBLOCK_STARTED，则将之修改为TBLOCK_BEGIN
                                                    - 否则，如果全局的静态变量CurrentTransactionState中记录的TBlockState是TBLOCK_IMPLICIT_INPROGRESS，则将之修改为TBLOCK_BEGIN
                                                    - 否则，其它状态则打印日志，给出提示信息
                                            - 如果是commit语句
                                                - EndTransactionBlock
                                                    - 如果全局的静态变量CurrentTransactionState中记录的TBlockState是TBLOCK_INPROGRESS，则将之修改为TBLOCK_END
                                                        # 如果YugaByte为PostgreSQL提供存储，则调用YugaByte的相关方法
                                                        - YBCCommitTransaction
                                                            - YBCPgTxnManager_CommitTransaction_Status
                                                                - CommitTransaction
                                                                    - txn_->CommitFuture().get()
                                                                    - ResetTxnAndSession
                                                    - 否则，如果全局的静态变量CurrentTransactionState中记录的TBlockState是TBLOCK_IMPLICIT_INPROGRESS，则将之修改为TBLOCK_END
                                                    - 否则，如果全局的静态变量CurrentTransactionState中记录的TBlockState是TBLOCK_ABORT，则将之修改为TBLOCK_ABORT_END
                                                    # 否则，如果是其它状态，则也有相应的处理，暂略
                                                    - ...
                                            - 如果是rollback语句
                                            - 如果是savepoint语句
                                            - 如果是release savepoint语句
                                            - 如果是rollback to savepoint语句
                                        # 忽略其它语句的处理
                                        - ...
                        - 否则不是utility statement，则：
                            - ProcessQuery
                                -  ExecutorRun
                                    - ExecutePlan
                                        - ExecProcNode
                                            - ExecModifyTable
                                                - ExecInsert
                                                - ExecUpdate
                                                - ExecDelete
                        # 如果当前的stmtlist_item不是portal->stmts中的最后一个，则增加command counter
                        - CommandCounterIncrement
            - finish_xact_command   
                - CommitTransactionCommand
                    - 如果全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState是TBLOCK_BEGIN，则：
                        - 将全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState修改为TBLOCK_INPROGRESS
                    - 如果全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState是TBLOCK_STARTED或者TBLOCK_END，则：
                        - 如果YugaByte作为PostgreSQL的存储，则：
                            - YBCCommitTransactionAndUpdateBlockState
                        - 否则：
                            - CommitTransaction
                                # 忽略PostgreSQL自身的一系列逻辑
                                - ...
                            -  将全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState修改为TBLOCK_DEFAULT
                # 设置全局静态bool变量xact_started为false
                - xact_started = false
```
从上面的分析可以看出：
- PostgreSQL在每次执行一条语句之前和之后，分别会执行start_xact_command(执行之后，全局的静态bool变量xact_started为true)和finish_xact_command(执行之后，全局的静态bool变量xact_started为false)。

- 在start_xact_command中，如果全局的静态bool变量xact_started为false，则会进一步调用StartTransactionCommand，因为每执行一条语句之后，都会调用finish_xact_command，所以xact_started一定为false，所以每次执行start_xact_command一定会进入到StartTransactionCommand。

- 在StartTransactionCommand中会判断全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState，并作相应处理：
    - 如果是TBLOCK_DEFAULT，则会进一步调用StartTransaction，并且将CurrentTransactionState中记录的TBlockState修改为TBLOCK_STARTED。
        - 全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState初始状态就是TBLOCK_DEFAULT，因此，执行第一条命令的时候，一定会走start_xact_command -> StartTransactionCommand -> StartTransaction。
    - 如果是TBLOCK_INPROGRESS/TBLOCK_IMPLICIT_INPROGRESS/TBLOCK_SUBINPROGRESS，则保持当前状态，什么也不做
        - 一旦遇到BEGIN命令，则：
            - 全局的静态变量CurrentTransactionState中记录的TBlockState被修改为TBLOCK_BEGIN
            - 在执行关于BEGIN命令的CommitTransactionCommand时会将全局的静态变量CurrentTransactionState中记录的TBlockState被修改为TBLOCK_INPROGRESS
    - 如果是其它状态，则打印日志提示信息
    
- 若当前被执行的是BEGIN命令，则会进入PortalRun -> PortalRunMulti -> PortalRunUtility -> ProcessUtility -> standard_ProcessUtility -> BeginTransactionBlock，会将全局的静态变量CurrentTransactionState中记录的TBlockState修改为TBLOCK_BEGIN。

- 若当前被执行的是插入/更新/删除/查询命令，则会进入PortalRun -> PortalRunMulti -> ProcessQuery，处理相应的statement。

- 若当前被执行的是COMMIT命令，则会进入PortalRun -> PortalRunMulti -> PortalRunUtility -> ProcessUtility -> standard_ProcessUtility -> EndTransactionBlock -> YBCCommitTransaction，会将全局的静态变量CurrentTransactionState中记录的TBlockState修改为TBLOCK_END。

- 在finish_xact_command中，如果全局的静态bool变量xact_started为true，则会进一步调用CommitTransactionCommand，因为每执行一条语句之后，都会调用start_xact_command，所以xact_started一定为true，所以每次执行finish_xact_command一定会进入到CommitTransactionCommand。

- 在CommitTransactionCommand中会判断全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState，并做相应处理：
    - 如果是TBLOCK_BEGIN，则修改为TBLOCK_INPROGRESS
    - 如果是TBLOCK_INPROGRESS/TBLOCK_IMPLICIT_INPROGRESS/TBLOCK_SUBINPROGRESS，则调用CommandCounterIncrement()来增加command counter
    - 如果是TBLOCK_STARTED/TBLOCK_END，则：
        - 如果YugaByte作为PostgreSQL的存储，则：
            - YBCCommitTransactionAndUpdateBlockState
                - YBCCommitTransaction
                    - YBCPgTxnManager_CommitTransaction_Status
                - CommitTransaction
                - 将全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState修改为TBLOCK_DEFAULT
        - 否则：
            - CommitTransaction
                - ...
            - 将全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState修改为TBLOCK_DEFAULT

### 举例说明
假设执行如下的SQL语句：
```
1) BEGIN
2) SELECT * FROM foo
3) INSERT INTO foo VALUES (...)
4) COMMIT
```

那么那么相应的方法调用及全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState状态如下:
```
方法调用                            相应的命令              TBlockState状态                     xact_started值
start_xact_command                                                                              false -> true
    StartTransactionCommand                                 TBLOCK_DEFAULT
        StartTransaction                                    TBLOCK_STARTED
            YBStartTransaction
ProcessUtility;                     BEGIN                   
    BeginTransactionBlock                                   TBLOCK_BEGIN
finish_xact_command
    CommitTransactionCommand                                TBLOCK_INPROGRESS                   true -> false

start_xact_command                                                                              false -> true
    StartTransactionCommand                                 TBLOCK_INPROGRESS
ProcessQuery                        SELECT
finish_xact_command
    CommitTransactionCommand                                TBLOCK_INPROGRESS                   true -> false
CommandCounterIncrement    

start_xact_command                                                                              false -> true
    StartTransactionCommand                                 TBLOCK_INPROGRESS
ProcessQuery                        INSERT
finish_xact_command
    CommitTransactionCommand                                TBLOCK_INPROGRESS                   true -> false
CommandCounterIncrement

start_xact_command                                                                              false -> true
    StartTransactionCommand                                 TBLOCK_INPROGRESS
ProcessUtility                      COMMIT
    EndTransactionBlock                                     TBLOCK_INPROGRESS -> TBLOCK_END
        YBCCommitTransaction
finish_xact_command
    CommitTransactionCommand                                                                    true -> false                         
        YBCCommitTransactionAndUpdateBlockState             TBLOCK_END -> TBLOCK_DEFAULT
            YBCCommitTransaction
            CommitTransaction
```
可以看出：
- PostgreSQL在每个命令执行前后会分别执行start_xact_command和finish_xact_command
- PostgreSQL在执行BEGIN命令时会执行BeginTransactionBlock
- PostgreSQL在执行COMMIT命令时会执行EndTransactionBlock
- YugaByte在BEGIN命令对应的start_xact_command -> StartTransactionCommand -> StartTransaction中加入了自己的逻辑
- YugaByte在END命令对应的finish_xact_command -> CommitTransactionCommand中加入了自己的逻辑

## YugaByte中事务相关的逻辑
通过前面的分析，我们知道YugaByte在PostgreSQL处理BEGIN命令和COMMIT命令时，添加了自己的逻辑，分别是StartTransaction -> YBStartTransaction和CommitTransactionCommand -> YBCCommitTransactionAndUpdateBlockState，下面分别分析。

### YBStartTransaction(事务控制部分 - 启动事务)
```
StartTransaction
    - 将当前事务对应的TransactionState中的TransState修改为TRANS_START
    - 设置全局变量XactReadOnly
        - 如果当前处于recovery过程中，则设置为true
        - 否则，使用默认值DefaultXactIsoLevel(false)
    - 设置全局变量XactDeferrable为默认值DefaultXactDeferrable(false)
    - 设置全局变量XactIsoLevel为默认值DefaultXactIsoLevel(XACT_READ_COMMITTED)
    - 为当前的事务设置virtual transaction id
    - 将当前事务对应的TransactionState中的TransState修改为TRANS_INPROGRESS
    # 将当前事务对应的TransactionState作为参数传递进去
    - YBStartTransaction
        - YBCPgTxnManager_BeginTransaction(YBCGetPgTxnManager())
            # 这里正是进入YQL逻辑
            # 获取当前的TransactionManager，TransactionManager是PgApiImpl的成员，每一个PostgresMain
            # 进程对应唯一一个TransactionManager
            - PgTxnManager::BeginTransaction
                - ResetTxnAndSession
                - txn_in_progress_ = true
                - StartNewSession
                    # 创建YBSession，在YBSession中会关联到一个YBClient(通过
                    # async_client_init_->client()获得)，以及一个混合时钟clock_
                    - session_ = std::make_shared<YBSession>(async_client_init_->client(), clock_)
                    # 设置ConsistentReadPoint
                    - session_->SetReadPoint(client::Restart::kFalse)
                    - session_->SetForceConsistentRead(client::ForceConsistentRead::kTrue)
        # XactIsoLevel是全局变量，表示事务隔离级别
        - YBCPgTxnManager_SetIsolationLevel(YBCGetPgTxnManager(), XactIsoLevel)
        # XactReadOnly是全局变量，表示事务是否是readonly的
        - YBCPgTxnManager_SetReadOnly(YBCGetPgTxnManager(), XactReadOnly)
        # XactDeferrable是全局变量，表示什么？
        - YBCPgTxnManager_SetDeferrable(YBCGetPgTxnManager(), XactDeferrable)   
        
```
从上面的分析来看，YBStartTransaction主要执行如下：
- 借助TransactionManager为当前的事务启动一个YBSession，并为当前事务设置一个Consistent ReadPoint(对于从多个不同的tablet读取的情况，必须确保读取到的数据是关于整个数据库的最近的一致性快照)。
- 在TransactionManager中记录当前事务设定的隔离级别
- 在TransactionManager中记录当前事务是否是readonly的
- 在TransactionManager中记录当前事务是否是deferrable的


### YBCCommitTransactionAndUpdateBlockState(事务控制部分 - 提交事务)
```
CommitTransactionCommand
    - 如果是TBLOCK_STARTED或者TBLOCK_END状态，则：
        - YBCCommitTransactionAndUpdateBlockState
            - YBCCommitTransaction
                - YBCPgTxnManager_CommitTransaction_Status(YBCGetPgTxnManager())
                    - PgTxnManager::CommitTransaction
                        # txn_是YBTransaction类型，表示YugaByte中的事务
                        - txn_->CommitFuture().get()
                            - YBTransaction::Impl::Commit
                                # 加锁
                                - std::unique_lock<std::mutex> lock(mutex_)
                                # 检查是否可以commit
                                - auto status = CheckCouldCommit(&lock)
                                    # 检查当前transaction处于TransactionState::kRunning状态，
                                    # 否则不能commit
                                    - CheckRunning(lock)
                                    - 其它检查(是否是child transaction，是否是restart required，running_requests_是否不为0)，只有这些检查都为false的情况下，才可以commit
                                - 修改transaction的状态为TransactionState::kCommitted
                                # 设置commit callback
                                - commit_callback_ = std::move(callback)
                                # 因为commit涉及到和status tablet交互，所以如果还没有找到status
                                # tablet，则将Impl::DoCommit添加到等待队列中，等待LookupTabletDone
                                # 被调用之后再执行，并且主动发送一个RequestStatusTablet请求，去
                                # 尝试获取status tablet
                                - if (!ready_) {
                                    waiters_.emplace_back(std::bind(&Impl::DoCommit, this, deadline, _1, transaction));
                                    lock.unlock();
                                    RequestStatusTablet(deadline);
                                    return;
                                  }
                                # 否则，将当前transaction在status tablet中的的状态修改为TransactionStatus::COMMITTED
                                - Impl::DoCommit(deadline, Status::OK(), transaction)
                                    # 初始化UpdateTransactionRequestPB，用于向status tablet发送commit rpc
                                    - tserver::UpdateTransactionRequestPB req
                                      req.set_tablet_id(status_tablet_->tablet_id())
                                      auto& state = *req.mutable_state()
                                      state.set_transaction_id(metadata_.transaction_id.begin(), metadata_.transaction_id.size());
                                      state.set_status(TransactionStatus::COMMITTED)
                                    # 将所有参与本次事务的tablet id也添加到rpc请求中
                                    - state.mutable_tablets()->Reserve(tablets_with_metadata_.size());
                                      for (const auto& tablet : tablets_with_metadata_) {
                                        state.add_tablets(tablet);
                                      }
                                    # 注册一个rpc call及其对应的Handle，会借助于这个Handle来发送rpc call
                                    # 当接收到commit响应的时候，Impl::CommitDone会被调用，进一步commit_callback_
                                    # 会被调用
                                    - manager_->rpcs().RegisterAndStart(
                                        UpdateTransaction(
                                            deadline,
                                            status_tablet_.get(),
                                            manager_->client(),
                                            &req,
                                            std::bind(&Impl::CommitDone, this, _1, _2, transaction)),
                                        &commit_handle_)
                        - ResetTxnAndSession
                            # 修改txn_in_progress_为false
                            - txn_in_progress_ = false
                            # 重置YBSession
                            - session_ = nullptr
                            - txn_ = nullptr
            - CommitTransaction
            - 将全局的静态TransactionState类型的变量CurrentTransactionState中记录的TBlockState修改为TBLOCK_DEFAULT
```

### transaction block中的一条SQL语句的执行(IO操作部分)
首先咱们看看在transaction block中的单条语句是怎么执行的(下面分析的代码可能跟“YugaByte之pggate”中分析的代码可能不同，这里是YugaByteDB 2.1的代码)：
```
YBCExecuteInsertInternal/YBCExecuteUpdate/YBCExecuteDelete
    YBCExecWriteStmt
        - YBCPgDmlExecWriteOp
            - PgApiImpl::DmlExecWriteOp
                - PgDmlWrite::Exec

PgDmlWrite::Exec
    # 更新protobuf信息
    - UpdateBindPBs
      UpdateAssignPBs
    # 设置protobuf中的column reference信息
    - ColumnRefsToPB(write_req_->mutable_column_refs())  
    # 执行该操作，如果成功添加到Batcher中执行，则返回RequestSent::kTrue，否则该操作
    # 被buffer了(添加到PgSession::RunHelper::buffered_ops_中，实际上就是PgSession::
    # buffered_txn_ops_或者PgSession::buffered_ops_，到底是哪个取决于当前操作是一个
    # 事务操作与否)，则返回RequestSent::kFalse
    - doc_op_->Execute(force_non_bufferable)
        - SendRequest(force_non_bufferable)
            # 只有当前不在等待上一个操作的response的过程中才能执行当前操作
            - DCHECK(!response_.InProgress())
            - exec_status_ = SendRequestImpl(force_non_bufferable)
                - PgDocWriteOp::SendRequestImpl
                    - response_ = pg_session_->RunAsync(write_op_, relation_id_, &read_time_, force_non_bufferable)
                        - PgSession::RunAsync(const std::shared_ptr<Op>* op, ...)
                            # ShouldHandleTransactionally用于判断是使用trasactional session，
                            # 还是non-transactional session，并且RunHelper中会有一个
                            # buffered_ops_，用于存放被buffered的数据，如果当前操作是一个
                            # 事务性操作，则buffered_ops_被设置为PgSession::buffered_txn_ops_，
                            # 否则buffered_ops_被设置为PgSession::buffered_ops_
                            - RunHelper runner(this, ShouldHandleTransactionally(**op))
                            # 遍历@op中的每一个操作，对于每一个操作@cur_op执行：
                                # 如果满足buffer条件的话，则将当前操作添加到buffer中(
                                # PgSession::RunHelper::buffered_ops_)
                                # 如果不满足buffer条件的话，则将当前操作添加到Batcher中，
                                # runner.Apply会在下面单独拿出来讲解
                                - runner.Apply(*cur_op, relation_id, read_time, force_non_bufferable)
                            # PgSession::RunHelper::只对Batcher中的操作起作用，将Batcher中
                            # 的操作flush到相应的tablet上真正执行，对于如果在PgSession::
                            # RunHelper::Apply中添加到buffered_ops_中的操作，则在
                            # PgSession::FlushBufferedOperationsImpl中被flush
                            - runner.Flush()
                                # 结束当前的Batcher，并将当前Batcher中的操作flush
                                # 到tablet上执行
                                - YBSession::FlushAsync
        # 如果正在等待响应，则返回RequestSent::kTrue，否则返回RequestSent::kFalse
        - return RequestSent(response_.InProgress())
    # 如果操作没有被buffered，则获取响应结果，并可能会进一步发送请求去获取更多的数据
    # 直到end_of_data_被设置
    - doc_op_->GetResult(&rowsets_)
```

#### PgSession::RunHelper::Apply
```
Status PgSession::RunHelper::Apply(std::shared_ptr<client::YBPgsqlOp> op,
                                   const PgObjectId& relation_id,
                                   uint64_t* read_time,
                                   bool force_non_bufferable) {
  # 获取当前所有被buffered操作的keys集合                              
  auto& buffered_keys = pg_session_.buffered_keys_;
  # 如果满足buffered条件，则将当前的操作添加到buffer中
  # buffered条件为：
  # 当前pg_session_开启了buffering && force_non_bufferable为false && 当前操作是写类型的操作
  if (pg_session_.buffering_enabled_ && !force_non_bufferable &&
      op->type() == YBOperation::Type::PGSQL_WRITE) {
    const auto& wop = *down_cast<client::YBPgsqlWriteOp*>(op.get());
    # 将当前的操作添加到buffered_keys集合中
    if (PREDICT_FALSE(!buffered_keys.insert(RowIdentifier(wop)).second)) {
      # 如果不能插入，则表明buffer中已经有关于相同行的操作了，那么需要在插入
      # 当前操作到buffer之前，需要先将buffer中已有的操作都flush
      RETURN_NOT_OK(pg_session_.FlushBufferedOperationsImpl());
      # 然后再插入当前操作的key到buffered_keys中
      buffered_keys.insert(RowIdentifier(wop));
    }
    
    # 将当前的操作插入到buffer中
    buffered_ops_.push_back({std::move(op), relation_id});
    
    # 如果buffer中的操作数目超过了阈值，则flush之
    return PREDICT_TRUE(buffered_keys.size() < FLAGS_ysql_session_max_batch_size)
        ? Status::OK()
        : pg_session_.FlushBufferedOperationsImpl();
  }

  # 至此，当前操作不能放到buffer中
  if (!buffered_keys.empty()) {
    # 如果buffer中已经有操作，则必须先确保这些操作被flush，也就是说在执行
    # non-buffered操作之前，必须确保所有buffered操作都被flush
    RETURN_NOT_OK(pg_session_.FlushBufferedOperationsImpl());
  }
  
  bool needs_pessimistic_locking = false;
  bool read_only = op->read_only();
  
  # 对于读类型的操作，需要检查它是否是只读的，是否需要使用悲观锁
  if (op->type() == YBOperation::Type::PGSQL_READ) {
    const PgsqlReadRequestPB &read_req = down_cast<client::YBPgsqlReadOp *>(op.get())->request();
    auto row_mark_type = GetRowMarkTypeFromPB(read_req);
    read_only = read_only && !IsValidRowMarkType(row_mark_type);
    needs_pessimistic_locking = RowMarkNeedsPessimisticLock(row_mark_type);
  }

  # 获取一个YBSession，如果是transactional的，则返回PgTxnManager中的YBSession，
  # 否则直接返回PgSession中的YBSession
  auto session = VERIFY_RESULT(pg_session_.GetSession(transactional_,
                              read_only,
                              needs_pessimistic_locking));
                              
  # 如果没有设置PgSession::RunHelper::yb_session，则使用前面获取到的YBSession来设置之
  if (!yb_session_) {
    yb_session_ = session->shared_from_this();
    if (transactional_ && read_time) {
      if (!*read_time) {
        *read_time = pg_session_.clock_->Now().ToUint64();
      }
      yb_session_->SetInTxnLimit(HybridTime(*read_time));
    }
  } else {
    // Session must not be changed as all operations belong to single session
    // (transactional or non-transactional)
    DCHECK_EQ(yb_session_.get(), session);
  }
  
  # YBSession::Apply
  return yb_session_->Apply(std::move(op));
}

YBSession::Apply
    # 这里就进入了我们熟悉的逻辑了
    # 将当前操作添加到in-flight ops中，且执行Tablet lookup，查找操作的目标tablet，
    # 找到tablet之后的回调是Batcher::TabletLookupFinished
    - Batcher().Add(yb_op)
```

##### PgSession中的buffering_enabled_的维护
设置：
```
exec_simple_query
    PortalRun
        PortalRunMulti
            ProcessQuery
                ExecutorStart
                    standard_ExecutorStart
                        # 如果YugaByte作为PostgreSQL的存储
                        YBBeginOperationsBuffering
                            YBCPgStartOperationsBuffering
                                PgApiImpl::StartOperationsBuffering
                                    PgSession::StartOperationsBuffering
                                        - buffering_enabled_ = true
```

清除：
```
# StartTransactionCommand在执行每一条命令的时候都会执行
StartTransactionCommand
    # 如果YugaByte作为PostgreSQL的存储，则在StartTransactionCommand中的
    # 第一件事就是将buffering_enabled_重置
    YBResetOperationsBuffering
        YBCPgResetOperationsBuffering
            PgApiImpl::ResetOperationsBuffering
                PgSession::ResetOperationsBuffering
                    - buffering_enabled_ = false
```

使用：
```
exec_simple_query
    PortalRun
        PortalRunMulti
            ProcessQuery
                ExecutorFinish
                    standard_ExecutorFinish
                    
standard_ExecutorFinish
    # 如果YugaByte作为PostgreSQL的存储，则执行
    YBEndOperationsBuffering
        YBCPgFlushBufferedOperations
            PgApiImpl::FlushBufferedOperations
                PgSession::FlushBufferedOperations
                    - buffering_enabled_ = false
                    - FlushBufferedOperationsImpl
                        - FlushBufferedOperationsImpl(ops, false /* transactional */)
                        - FlushBufferedOperationsImpl(txn_ops, true /* transactional */)
```

上面的设置，使用和清除过程都是在exec_simple_query中，那么exec_simple_query中ExecutorStart，StartTransactionCommand和ExecutorFinish，甚至还有一个ExecutorEnd之间的先后顺序是怎样的呢？
```
exec_simple_query
    start_xact_command
        StartTransactionCommand     # 在这里设置buffering_enabled_为false
    PortalRun
        PortalRunMulti     
            ProcessQuery
                - ExecutorStart     # 在这里设置buffering_enabled_为true
                - ExecutorRun       # 在这里如果buffering_enabled_为true，则会将操作添加到buffer中
                - ExecutorFinish    # 在这里flush buffer中的数据，并将buffering_enabled_设置为false
                - ExecutorEnd
```

##### FlushBufferedOperationsImpl的调用时机
1. 主动flush buffer中的操作
```
exec_simple_query
    PortalRun
        PortalRunMulti
            ProcessQuery
                ExecutorFinish
                    standard_ExecutorFinish
                    
standard_ExecutorFinish
    # 如果YugaByte作为PostgreSQL的存储，则执行
    YBEndOperationsBuffering
        YBCPgFlushBufferedOperations
            PgApiImpl::FlushBufferedOperations
                PgSession::FlushBufferedOperations
                    - buffering_enabled_ = false
                    - FlushBufferedOperationsImpl
                        - FlushBufferedOperationsImpl(ops, false /* transactional */)
                        - FlushBufferedOperationsImpl(txn_ops, true /* transactional */)
```

2. 被动flush buffer中的操作
```
Status PgSession::RunHelper::Apply(std::shared_ptr<client::YBPgsqlOp> op,
                                   const PgObjectId& relation_id,
                                   uint64_t* read_time,
                                   bool force_non_bufferable) {
                                     
  if (pg_session_.buffering_enabled_ && !force_non_bufferable &&
      op->type() == YBOperation::Type::PGSQL_WRITE) {
    # 当前操作需要被buffer
    const auto& wop = *down_cast<client::YBPgsqlWriteOp*>(op.get());
    # 将当前的操作对应的key添加到buffered_keys集合中，
    if (PREDICT_FALSE(!buffered_keys.insert(RowIdentifier(wop)).second)) {
      # 如果不能插入，则表明buffer中已经有关于相同行的操作了，那么需要在
      # 插入当前操作到buffer之前，需要先将buffer中已有的操作都flush
      RETURN_NOT_OK(pg_session_.FlushBufferedOperationsImpl());
      # 然后再插入当前操作的key到buffered_keys中
      buffered_keys.insert(RowIdentifier(wop));
    }
    
    # 将当前的操作插入到buffer中
    buffered_ops_.push_back({std::move(op), relation_id});
    
    # 如果buffer中的操作数目超过了阈值，则buffer中所有的操作立即被flush
    return PREDICT_TRUE(buffered_keys.size() < FLAGS_ysql_session_max_batch_size)
        ? Status::OK()
        : pg_session_.FlushBufferedOperationsImpl();
  }

  # 至此，当前操作不能放到buffer中
  if (!buffered_keys.empty()) {
    # 如果buffer中已经有操作，则必须先确保这些操作被flush，也就是说在执行
    # non-buffered操作之前，必须确保所有buffered操作都被flush
    RETURN_NOT_OK(pg_session_.FlushBufferedOperationsImpl());
  }
  
  ...
}
```

##### FlushBufferedOperationsImpl中都做了些什么
```
PgSession::FlushBufferedOperationsImpl
    # 将buffered_ops_和buffered_txn_ops_分别赋值给临时遍历ops和txn_ops
    - auto ops = std::move(buffered_ops_);
      auto txn_ops = std::move(buffered_txn_ops_);
    # 清空buffered_keys_，buffered_ops_和buffered_txn_ops_
      buffered_keys_.clear();
      buffered_ops_.clear();
      buffered_txn_ops_.clear();
    # flush non-transactional operations
    - FlushBufferedOperationsImpl(ops, false /* transactional */)
    # flush transactional operations
    - FlushBufferedOperationsImpl(txn_ops, true /* transactional */)

PgSession::FlushBufferedOperationsImpl(const PgsqlOpBuffer& ops, bool transactional)
    # 如果是transactional的，则获取PgTxnManager中的YBSession，否则直接获取PgSession中
    # 的YBSession
    - auto session = VERIFY_RESULT(GetSession(transactional, false /* read_only_op */))
    # 逐一执行每一个buffered operation
    - for (auto buffered_op : ops) {
        const auto& op = buffered_op.operation
        # YBSession::Apply
        session->Apply(op)
            # 这里就进入了我们熟悉的逻辑了
            # 将当前操作添加到in-flight ops中，且执行Tablet lookup，查找操作的目标tablet，
            # 并且增加outstanding_lookups_数目，表示正在查找tablet的操作的数目，当找到
            # tablet之后的回调是Batcher::TabletLookupFinished，当该回调被调用的时候，它
            # 会减少outstanding_lookups_数目，如果outstanding_lookups_降为0，则它会尝试
            # 执行Batcher::FlushBuffersIfReady，在Batcher::FlushBuffersIfReady中只有Batcher
            # 状态为BatcherState::kResolvingTablets状态，才会真正尝试去将操作flush到相应
            # 的tablet上执行，但是如果Batcher::FlushAsync还未被调用，则Batcher的状态仍然为
            # BatcherState::kGatheringOps，那么就不会flush了
            - Batcher().Add(yb_op)
      }
    - const auto status = session->FlushFuture().get()
        # 在这里会执行以下：
        # 1. 结束当前的Batcher
        # 2. 将Batcher的状态从BatcherState::kGatheringOps切换为BatcherState::kResolvingTablets
        # 3. FlushBuffersIfReady() - 将这些操作flush到相应的tablet执行，会发送rpc，
        #    rpc完成时的回调都是AsyncRpc::Finished，在AsyncRpc::Finished中会进一步调用
        #    Batcher::CheckForFinishedFlush，以执行flush完成后的回调
        # 
        # 详细分析见PgSession::RunHelper::Flush中对YBSession::FlushAsync的分析
        - YBSession::FlushAsync
```


#### PgSession::RunHelper::Flush
```
PgSession::RunHelper::Flush
    YBSession::FlushAsync
        - 结束当前的Batcher
        # 将当前的Batcher中的所有操作flush到相应tablet上执行
        - Batcher::FlushAsync
            # 将Batcher的状态从BatcherState::kGatheringOps切换为
            # BatcherState::kResolvingTablets
            - CHECK_EQ(state_, BatcherState::kGatheringOps);
              state_ = BatcherState::kResolvingTablets;
            # 告诉事务包含这么多的操作(用于确认事务何时算是完成了的？)
            - operations_count = ops_.size();
              auto transaction = this->transaction()
              transaction->ExpectOperations(operations_count)
                - YBTransaction::Impl::ExpectOperations
                    # 只是增加了running_requests的计数，其它什么都没做
                    - running_requests_ += count
            - CheckForFinishedFlush()
            # 如果Batcher中的所有的操作都找到了它对应的tablet，则将这些操作
            # 发送给tablet执行
            - FlushBuffersIfReady()
                # 如果当前至少还有一个操作还没有接收到tabletlookup的响应，
                # 或者当前Batcher的状态不是BatcherState::kResolvingTablets，
                # 则直接返回
                -   if (outstanding_lookups_ != 0) {
                      return;
                    }
                
                    if (state_ != BatcherState::kResolvingTablets) {
                      return;
                    }
                # 否则，将Batcher的状态调整为BatcherState::kTransactionPrepare
                - state_ = BatcherState::kTransactionPrepare
                - 对所有的操作按照tablet和operation group进行分组排序，率属于
                  同一个tablet的相同operation group的放在一起
                # 执行操作，发送RPC给tablets
                - ExecuteOperations(Initial::kTrue)
                    # prepare
                    - transaction->Prepare(ops_queue_,
                          force_consistent_read_,
                          deadline_,
                          initial,
                          # 如果当前还没有获取到status tablet信息，则该回调会被添加到
                          # 等待队列中，在获取到status tablet信息之后被调用
                          std::bind(&Batcher::TransactionReady, this, _1, BatcherPtr(this)),
                          &transaction_metadata_)
                        - YBTransaction::Impl::Prepare
                            - 检查是否存在某些tablets，没有元数据信息，这块现在还没搞懂！！！
                            # 如果当前还没有获取到status tablet信息，则将回调
                            # Batcher::TransactionReady添加到等待队列中，并发送请求获取
                            # status tablet信息，然后直接返回false，表示还没有真正prepare
                            - waiters_.push_back(std::move(waiter))
                              RequestStatusTablet(deadline)
                              return false
                            # 否则，如果需要设置consistent readpoint，则设置之
                            - SetReadTimeIfNeeded(num_tablets > 1 || force_consistent_read)
                    # 修改Batcher状态为BatcherState::kTransactionReady
                    - state_ = BatcherState::kTransactionReady
                    - 对Batcher中所有操作按照tablet和operation group进行分组，tablet相同且operation group相同的操作放在一起共享一个rpc，生成的所有的rpcs都放在rpcs数组中
                    # 逐一发送生成的所有的rpcs
                    - for (const auto& rpc : rpcs) {
                        # 发送rpc，对于读或者写类型的操作，rpc完成时的回调都是AsyncRpc::Finished
                        rpc->SendRpc()
                      }
```

##### 当接收到flush operation to tablet的rpc回调时的处理
```
AsyncRpc::Finished
    # 以写类型的请求为例，实际调用的是WriteRpc::ProcessResponseFromTserver
    - ProcessResponseFromTserver(new_status)
        - WriteRpc::ProcessResponseFromTserver
            - Batcher::ProcessWriteResponse
            # 从RPC响应中分离出来该RPC中每个操作对应的响应
            - WriteRpc::SwapRequestsAndResponses
      # @ops_是当前这个AsyncRpc中的所有操作集合，不是Batcher中的所有操作的集合
      batcher_->RemoveInFlightOpsAfterFlushing(ops_, new_status, MakeFlushExtraResult())
        - YBTransaction::Flushed
            - YBTransaction::Impl::Flushed
                # 减少running_requests_数目
                - running_requests_ -= ops.size()
                # 如果是snapshot isolation，且在TServer端执行的时候使用了新的read time，则更新read point
                # 其中，used_read_time表示TServer端执行的时候使用到的read time
                - read_point_.SetReadTime(used_read_time, ConsistentReadPoint::HybridTimeMap())
                # 遍历RPC中的每个操作，如果操作执行成功，则设置对应的Tablet的TabletState::has_metadata
                # 为true，这里prev_tablet_id是为了避免重复添加相同的Tablet信息到YBTransaction::Impl::tablets_
                # 中，之所以能够直接通过prev_tablet_id来避免重复，是因为这里操作是排好序的，但是根据前面
                # 的分析，同一个RPC中的请求不是率属于同一个Tablet吗，所以这里的判断是不是多余的
                -   for (const auto& op : ops) {
                      if (op->yb_op->applied() && op->yb_op->should_add_intents(metadata_.isolation)) {
                        const std::string& tablet_id = op->tablet->tablet_id();
                        if (prev_tablet_id == nullptr || tablet_id != *prev_tablet_id) {
                          prev_tablet_id = &tablet_id;
                          tablets_[tablet_id].has_metadata = true;
                        }
                      }
                    }
        # 更新read point
        - if (status.ok() && read_point_) {
            read_point_->UpdateClock(flush_extra_result.propagated_hybrid_time);
          }
        # 从Batcher::ops_中移除该RPC中相应的操作
        - for (auto& op : ops) {
            ops_.erase(op))
          }
      # 检查是否Batcher中的所有操作都flush成功了(接收到flush响应)，如果是，则调用
      # 相应的callback
      batcher_->CheckForFinishedFlush()
```

##### 当Batcher中的所有操作都接收到flush响应时的处理
```
Batcher::CheckForFinishedFlush
    - 检查Batcher::ops_中是不是所有的操作都接收到了flush的响应(AsyncRpc::Finished被调用)，
      如果不是，则直接返回
    - 状态检查，如果当前处于BatcherState::kResolvingTablets状态(对于非事务性操作)，或者
      处于BatcherState::kTransactionReady状态(对于事务性操作)，则继续，否则返回
    # 将Batcher状态设置为BatcherState::kComplete
    - state_ = BatcherState::kComplete
    # 从YBSession::flushed_batchers_中移除当前Batcher，与之对应的是在YBSession::FlushAsync
    # 中会将Batcher添加到YBSession::flushed_batchers_中
    - if (session) {
        session->FlushFinished(this);
      }
    # 执行flush callback
    - RunCallback(s)
```

### gdb调试情况说明
gdb调试结果请参考[这里](http://note.youdao.com/noteshare?id=2050b5a0c589d7ee1901f7d4d2b80d45&sub=77523A128A82420E8168A1EBCFA50239)。

## TransactionManager和StatusTablet的交互
### 查找StatusTablet并获取它的相关信息
```
YBTransaction::Impl::Prepare/YBTransaction::Impl::Commit/YBTransaction::Impl::Abort
    YBTransaction::Impl::RequestStatusTablet
        # 如果当前还没有设置status tablet，则挑选一个，当挑选到StatusTablet之后，
        # YBTransaction::Impl::StatusTabletPicked会被调用，Impl::StatusTabletPicked
        # 会进一步调用YBTransaction::Impl::LookupStatusTablet去获取StatusTablet的
        # 相关信息
        - manager_->PickStatusTablet(
          std::bind(&Impl::StatusTabletPicked, this, _1, deadline, transaction))
        # 否则已经设置了status tablet，则查询status tablet的相关信息
        - LookupStatusTablet(metadata_.status_tablet, deadline, transaction)
```

当成功获取到StatusTablet的信息的时候，YBTransaction::Impl::LookupTabletDone会被调用
```
YBTransaction::Impl::LookupStatusTablet
    - manager_->client()->LookupTabletById(
        tablet_id,
        deadline,
        std::bind(&Impl::LookupTabletDone, this, _1, transaction),
        client::UseCache::kTrue)
```

在YBTransaction::Impl::LookupTabletDone中会将查询到的关于StatusTablet的信息记录到YBTransaction::Impl::status_tablet_中，并且向StatusTablet发送心跳信息：
```
  YBTransaction::Impl::LookupTabletDone(const Result<client::internal::RemoteTabletPtr>& result,
                        const YBTransactionPtr& transaction) {
    bool precreated;
    std::vector<Waiter> waiters;
    {
      std::lock_guard<std::mutex> lock(mutex_);
      # 用获取到的StatusTablet的信息设置YBTransaction::Impl::status_tablet_
      status_tablet_ = std::move(*result);
      if (metadata_.status_tablet.empty()) {
        # 将获取到的StatusTablet的id记录到metadata中
        metadata_.status_tablet = status_tablet_->tablet_id();
        precreated = false;
      } else {
        precreated = true;
        ready_ = true;
        waiters_.swap(waiters);
      }
    }
    
    if (precreated) {
      for (const auto& waiter : waiters) {
        waiter(Status::OK());
      }
    }
    
    # 向StatusTablet发送心跳，在心跳信息中会附带上TransactionStatus，
    # 如果precreated为false，则需要先将事务添加到StatusTablet的管理，
    # 所以设置状态为TransactionStatus::CREATED，否则设置状态为
    # TransactionStatus::PENDING，StatusTablet将会根据状态进行处理
    SendHeartbeat(precreated ? TransactionStatus::PENDING : TransactionStatus::CREATED,
                  metadata_.transaction_id, transaction_->shared_from_this());
  }
  
  void SendHeartbeat(TransactionStatus status,
                     const TransactionId& id,
                     const std::weak_ptr<YBTransaction>& weak_transaction) {
    ...
    
    tserver::UpdateTransactionRequestPB req;
    req.set_tablet_id(status_tablet_->tablet_id());
    req.set_propagated_hybrid_time(manager_->Now().ToUint64());
    auto& state = *req.mutable_state();
    state.set_transaction_id(metadata_.transaction_id.begin(), metadata_.transaction_id.size());
    # 设置当前transaction的status
    state.set_status(status);
    
    # 向StatusTablet发送心跳，当接收到心跳响应之后，Impl::HeartbeatDone会被调用，它会进一步发送心跳
    manager_->rpcs().RegisterAndStart(
        UpdateTransaction(
            TransactionRpcDeadline(),
            status_tablet_.get(),
            manager_->client(),
            &req,
            # 回调中会用到status参数
            std::bind(&Impl::HeartbeatDone, this, _1, _2, status, transaction)),
        &heartbeat_handle_);
  }  
```

如果运行逻辑是YBTransaction::Impl::RequestStatusTablet -> YBTransaction::Impl::StatusTabletPicked -> YBTransaction::Impl::LookupStatusTablet -> YBTransaction::Impl::LookupTabletDone，则此时并不会将YBTransaction::Impl::ready_设置为true，那么什么时候才会将YBTransaction::Impl::ready_设置为true呢？我们知道在YBTransaction::Impl::LookupTabletDone -> YBTransaction::Impl::SendHeartbeat中会向StatusTablet发送心跳信息，当接收到心跳的响应的时候YBTransaction::Impl::HeartbeatDone会被调用：
```
  void HeartbeatDone(const Status& status,
                     HybridTime propagated_hybrid_time,
                     TransactionStatus transaction_status,
                     const YBTransactionPtr& transaction) {
    if (status.ok()) {
      # 在YBTransaction::Impl::LookupTabletDone中，可以看出，在YBTransaction::Impl::ready_
      # 没有被设置为true的情况下，TransactionStatus被设置为TransactionStatus::CREATED，所以
      # 这里的if语句会进入
      if (transaction_status == TransactionStatus::CREATED) {
        # 唤醒所有的waiters
        NotifyWaiters(Status::OK());
      }
      
      std::weak_ptr<YBTransaction> weak_transaction(transaction);
      manager_->client()->messenger()->scheduler().Schedule(
          [this, weak_transaction](const Status&) {
              SendHeartbeat(TransactionStatus::PENDING, metadata_.transaction_id, weak_transaction);
          },
          std::chrono::microseconds(FLAGS_transaction_heartbeat_usec));
    }
  }
    
  void NotifyWaiters(const Status& status) {
    std::vector<Waiter> waiters;
    {
      std::lock_guard<std::mutex> lock(mutex_);
      if (status.ok()) {
        # 将YBTransaction::Impl::ready_设置为true
        DCHECK(!ready_);
        ready_ = true;
      } else {
        SetError(status, &lock);
      }
      waiters_.swap(waiters);
    }
    
    # 唤醒所有的waiters
    for (const auto& waiter : waiters) {
      waiter(status);
    }
  }
```

从上面的分析来看，在YBTransaction::Impl::RequestStatusTablet -> YBTransaction::Impl::StatusTabletPicked -> YBTransaction::Impl::LookupStatusTablet -> YBTransaction::Impl::LookupTabletDone中，在YBTransaction::Impl::LookupTabletDone(也就是获取到了StatusTablet的相关信息)中不会将YBTransaction::Impl::ready_设置为true，而是在YBTransaction::Impl::RequestStatusTablet -> YBTransaction::Impl::StatusTabletPicked -> YBTransaction::Impl::LookupStatusTablet -> YBTransaction::Impl::LookupTabletDone -> YBTransaction::Impl::SendHeartbeat -> YBTransaction::Impl::HeartbeatDone中，当YBTransaction::Impl::HeartbeatDone被调用的时候将YBTransaction::Impl::ready_设置为true。我想，这可能与YBTransaction::Impl::LookupStatusTablet可能在meta cache中缓存有关，需要发送一个心跳进行确认了之后才能真正将YBTransaction::Impl::ready_设置为true。


### 和StatusTablet之间的心跳
在YBTransaction::Impl::SendHeartbeat中会向StatusTablet发送心跳信息。YBTransaction::Impl::SendHeartbeat定义如下：
```
YBTransaction::Impl::SendHeartbeat(TransactionStatus status,
             const TransactionId& id,
             const std::weak_ptr<YBTransaction>& weak_transaction) {
    ...
    
    # RPC request
    tserver::UpdateTransactionRequestPB req;
    # 发送给StatusTablet
    req.set_tablet_id(status_tablet_->tablet_id());
    req.set_propagated_hybrid_time(manager_->Now().ToUint64());
    auto& state = *req.mutable_state();
    state.set_transaction_id(metadata_.transaction_id.begin(), metadata_.transaction_id.size());
    # 设置当前TransactionState
    state.set_status(status);
    # 发送RPC
    manager_->rpcs().RegisterAndStart(
        UpdateTransaction(
            TransactionRpcDeadline(),
            status_tablet_.get(),
            manager_->client(),
            &req,
            # 接收到心跳响应之后的回调
            std::bind(&Impl::HeartbeatDone, this, _1, _2, status, transaction)),
        &heartbeat_handle_);
 }
```

YBTransaction::Impl::SendHeartbeat的调用关系如下：
```
# 在查找到StatusTablet的信息之后，发送第一个心跳    
YBTransaction::Impl::LookupTabletDone
    - YBTransaction::Impl::SendHeartbeat
    
YBTransaction::Impl::HeartbeatDone
    # 接收到心跳响应之后，过一段时间之后，再次发送心跳
    - YBTransaction::Impl::SendHeartbeat
```

### 提交事务
#### status tablet接收到commit请求时的处理
对于commit请求，status tablet接收到的实际上是UpdateTransactionRequestPB，其中附带了一个TransactionStatus为TransactionStatus::Committed，在status tablet端的处理入口是：TabletServiceImpl::UpdateTransaction。
```
TabletServiceImpl::UpdateTransaction
    # 查找status tablet的leader 
    - tablet = LookupLeaderTabletOrRespond(...)
    # 生成一个UpdateTxnOperationState，req->state()类型为TransactionStatePB，其中包括
    # transaction id，TransactionStatus，参与的tablets等信息，这些信息将进一步传递给
    # UpdateTxnOperationState继承自OperationState的request_
    - auto state = std::make_unique<tablet::UpdateTxnOperationState>(tablet.peer->tablet(),
                   &req->state());
      # 设置transaction completion callback，transaction completion并不一定是成功提交了，
      # 也可能是发生了错误，该callback会发送响应给client
      state->set_completion_callback(MakeRpcOperationCompletionCallback(
        std::move(context), resp, server_->Clock()));
    # 因为当前TransactionStatus是COMMITTED，所以交给transaction coordinator来处理
    # transactio update请求
    - tablet.peer->tablet()->transaction_coordinator()->Handle(std::move(state), tablet.leader_term)
        - TransactionCoordinator::Impl::Handle
            # TransactionStatePB，其中包括transaction id，TransactionStatus，参与的tablets等
            - auto& state = *request->request()
            # 获取transaction id
            - auto id = FullyDecodeTransactionId(state.transaction_id())
            # 到managed_transactions_中查找对应的transaction是否存在，如果不存在，则表明
            # transaction已经expired或者由于冲突而aborted了，所以直接调用transaction 
            # completion callback
            - auto status = STATUS(Expired, "Transaction expired or aborted by a conflict",
                               PgsqlError(YBPgErrorCode::YB_PG_T_R_SERIALIZATION_FAILURE));
              status = status.CloneAndAddErrorCode(TransactionError(TransactionErrorCode::kAborted));
              request->CompleteWithStatus(status);
            # 否则，在managed_transactions_中找到了，则处理之，参数@it就是在managed_transactions_
            # 中查找时返回的迭代器
            - managed_transactions_.modify(it, [&request](TransactionState& state) {
                # TransactionState::Handle
                state.Handle(std::move(request));
              });

TransactionState::Handle
    - TransactionState::DoHandle
        - TransactionState::HandleCommit
            - 检查是否过期了，如果过期了，则abort之，否则返回Status::OK()
        # commit请求即将在raft中复制
        - replicating_ = request.get();
          # 这里的context_类型是TransactionCoordinator::Impl
          auto submitted = context_.SubmitUpdateTransaction(std::move(request))  
            - TransactionCoordinator::Impl::SubmitUpdateTransaction
                - 添加到PostponedLeaderActions.updates数组中，PostponedLeaderActions.updates中存放所有被延期的与Update transaction相关的操作，对于添加到PostponedLeaderActions.updates数组中的操作会被异步处理，真正处理是在TransactionCoordinator::Impl::ExecutePostponedLeaderActions
```

#### PostponedLeaderActions中的被延期的操作的执行
在ExecutePostponedLeaderActions中执行那些被延期的操作:
```
TransactionCoordinator::Impl::ExecutePostponedLeaderActions
    - 遍历PostponedLeaderActions::complete_with_status，处理该数组中的操作
        - for (const auto& p : actions->complete_with_status) {
            p.request->CompleteWithStatus(p.status);
          }
    - 遍历PostponedLeaderActions::notify_applying数组中的每一个action(notify_applying数组中存放的是所有将要发送给tablet的Apply操作)，并执行如下：
        # 初始化UpdateTransactionRequestPB
        -   tserver::UpdateTransactionRequestPB req;
            #设置目标tablet
            req.set_tablet_id(action.tablet);
            auto& state = *req.mutable_state();
            state.set_transaction_id(action.transaction.data(), action.transaction.size());
            # 设置APPLYING标识
            state.set_status(TransactionStatus::APPLYING);
            state.add_tablets(context_.tablet_id());
            # 设置commit timestamp
            state.set_commit_hybrid_time(action.commit_time.ToUint64());
            state.set_sealed(action.sealed);
        # 获取一个RPC，设置接收到RPC响应之后的回调，然后发送给对应的tablet
        -   auto handle = rpcs_.Prepare();
            if (handle != rpcs_.InvalidHandle()) {
              *handle = UpdateTransaction(
                  deadline,
                  nullptr /* remote_tablet */,
                  context_.client_future().get(),
                  &req,
                  # 接收到rpc响应之后的回调
                  [this, handle, txn_id = action.transaction, tablet = action.tablet]
                      (const Status& status, const tserver::UpdateTransactionResponsePB& resp) {
                    client::UpdateClock(resp, &context_);
                    rpcs_.Unregister(handle);
                    ...
                  });
              # 发送rpc
              (**handle).SendRpc();
            }
    # 遍历PostponedLeaderActions中updates数组(该数组中存放所有与Update Transaction相关的操作)中的所有的操作
    - for (auto& update : actions->updates) {
        context_.SubmitUpdateTransaction(std::move(update), actions->leader_term);
      }
```

##### PostponedLeaderActions中被延期的Update Transaction相关的操作的处理
对于添加到PostponedLeaderActions.updates数组中的操作会被异步处理，真正处理是在TransactionCoordinator::Impl::ExecutePostponedLeaderActions
中，它会进一步调用TabletPeer::SubmitUpdateTransaction来处理。
```
- TabletPeer::SubmitUpdateTransaction
    # 生成一个tablet::UpdateTxnOperation，并提交给raft
    - auto operation = std::make_unique<tablet::UpdateTxnOperation>(std::move(state))
      Submit(std::move(operation), term)
        # 在创建OperationDriver的过程中，会设置当复制完成时的回调
        # OperationDriver::ReplicationFinished
        - auto driver = NewLeaderOperationDriver(&operation, term)
          (**driver).ExecuteAsync()
            - Preparer::Submit
```

当PostponedLeaderActions中被延期的Update Transaction相关的操作在Raft中复制成功之后，相应的回调OperationDriver::ReplicationFinished会被调用：  
```
OperationDriver::ReplicationFinished
    - OperationDriver::ApplyOperation
        - OperationDriver::ApplyTask
            - Operation::Replicated
                - Status complete_status = Status::OK()
                  DoReplicated(leader_term, &complete_status)
                    - UpdateTxnOperation::DoReplicated
                        - TransactionCoordinator::ProcessReplicated
                            - TransactionCoordinator::Impl::ProcessReplicated
                                - id = FullyDecodeTransactionId(data.state.transaction_id())
                                # 在managed_transactions_中查找相应的transaction，并处理之，
                                # TransactionState::ProcessReplicated将在后面分析
                                - auto it = GetTransaction(*id, data.state.status(),         data.hybrid_time);
                                  managed_transactions_.modify(it, [&result, &data](TransactionState& state) {
                                    result = state.ProcessReplicated(data);
                                  });
                                  # 检查当前请求是否结束了，如果处于TransactionStatus::ABORTED或者
                                  # TransactionStatus::APPLIED_IN_ALL_INVOLVED_TABLETS状态，则认为是结束了，
                                  # 如果结束了则做进一步的处理
                                  CheckCompleted(it);
                  # 调用completion callback发送响应
                  state()->CompleteWithStatus(complete_status)
                  
TransactionState::ProcessReplicated
    # 当前操作的raft复制已经完成了，将replicating_设置为null，表示当前没有正在执行复制的操作
    - replicating_ = nullptr
    - TransactionState::DoProcessReplicated
        # 根据transaction当前所处的状态进行分别处理，在当前上下文中处于Committed状态
        # 所以会进入CommittedReplicationFinished中
        - CommittedReplicationFinished
            # 设置所有参与当前事务的tablets状态
            -   involved_tablets_.reserve(data.state.tablets().size());
                for (const auto& tablet : data.state.tablets()) {
                  InvolvedTabletState state = {
                    .required_replicated_batches = 0,
                    # 关于该tablet的所有的batches都已经复制
                    .all_batches_replicated = true,
                    # 关于该tablet的所有的临时记录还没有转换为正式记录
                    .all_intents_applied = false
                  };
                  
                  # 添加到involved_tablets_中
                  involved_tablets_.emplace(tablet, state);
                }
    
            # 将status tablet上维护的transaction status设置为TransactionStatus::COMMITTED，
            # 在前面的过程中使用的都是RPC request中的TransactionStatus
            - status_ = TransactionStatus::COMMITTED
            # 通知所有的tablets，将其中的临时记录转换为正式记录
            - TransactionState::StartApply
                - for (const auto& tablet : involved_tablets_) {
                    # 实际上调用的是TransactionCoordinator::Impl::NotifyApplying,
                    # 这里会将需要apply的tablet相关的信息添加到PostponedLeaderActions的
                    # notify_applying数组中，在TransactionCoordinator执行过程中的很多地
                    # 方都会调用TransactionCoordinator::Impl::ExecutePostponedLeaderActions
                    # 来处理PostponedLeaderActions中被延期的操作，PostponedLeaderActions
                    # 中存放的都是那些需要等待TransactionCoordinator中的锁释放之后才能
                    # 执行的操作
                    context_.NotifyApplying({
                        .tablet = tablet.first,
                        .transaction = id_,
                        .commit_time = commit_time_,
                        .sealed = status_ == TransactionStatus::SEALED});
                        - postponed_leader_actions_.notify_applying.push_back(std::move(data))
                  }
    # 当前操作被处理过程中可能有新的update transaction请求到达，这些请求都
    # 在TransactionState的request_queue_中等待，TransactionState::ProcessQueue
    # 用于处理这些处于等待队列的请求，过程跟处理当前请求类似
    - TransactionState::ProcessQueue
```

##### PostponedLeaderActions中被延期的notify_applying数组中操作(通知tablet将临时记录转换为正式记录)的处理
对于添加到PostponedLeaderActions.notify_applying数组中的操作会被异步处理，真正处理是在TransactionCoordinator::Impl::ExecutePostponedLeaderActions中：
```
TransactionCoordinator::Impl::ExecutePostponedLeaderActions
    - ...
    - 遍历PostponedLeaderActions::notify_applying数组中的每一个action(notify_applying数组中存放的是所有将要发送给tablet的Apply操作)，并执行如下：
        # 初始化UpdateTransactionRequestPB
        -   tserver::UpdateTransactionRequestPB req;
            #设置目标tablet
            req.set_tablet_id(action.tablet);
            auto& state = *req.mutable_state();
            state.set_transaction_id(action.transaction.data(), action.transaction.size());
            # 设置APPLYING标识
            state.set_status(TransactionStatus::APPLYING);
            state.add_tablets(context_.tablet_id());
            # 设置commit timestamp
            state.set_commit_hybrid_time(action.commit_time.ToUint64());
            state.set_sealed(action.sealed);
        # 获取一个RPC，然后发送给对应的tablet，这里发送的是UpdateTransactionRequestPB类型的请求，
        # 在tablet接收到该请求时，相应的处理为TabletServiceImpl::UpdateTransaction，详细分析见
        # “tablet处理来自于status tablet的Apply请求”
        -   auto handle = rpcs_.Prepare();
            if (handle != rpcs_.InvalidHandle()) {
              *handle = UpdateTransaction(..., &req, ...);
              # 发送rpc
              (**handle).SendRpc();
            }
    - ...    
}
```

#### tablet处理来自于status tablet的Apply请求
```
TabletServiceImpl::UpdateTransaction
    # 查找status tablet的leader 
    - tablet = LookupLeaderTabletOrRespond(...)
    # 生成一个UpdateTxnOperationState，req->state()类型为TransactionStatePB，其中包括
    # transaction id，TransactionStatus，参与的tablets等信息，这些信息将进一步传递给
    # UpdateTxnOperationState继承自OperationState的request_
    - auto state = std::make_unique<tablet::UpdateTxnOperationState>(tablet.peer->tablet(),
                   &req->state());
      # 设置transaction completion callback，transaction completion并不一定是成功提交了，
      # 也可能是发生了错误，该callback会发送响应给client
      state->set_completion_callback(MakeRpcOperationCompletionCallback(
        std::move(context), resp, server_->Clock()));
    # 因为当前TransactionStatus是APPLYING，所以交给transaction participant来处理
    # transactio update请求
    - tablet.peer->tablet()->transaction_participant()->Handle(std::move(state), tablet.leader_term)
        - TransactionParticipant::Impl::Handle
            # 因为当前的状态是APPLYING
            - HandleApplying(std::move(state), term)
                - participant_context_.SubmitUpdateTransaction(std::move(state), term)
                    # 生成一个tablet::UpdateTxnOperation，并提交给raft
                    - auto operation = std::make_unique<tablet::UpdateTxnOperation>(std::move(state))
                      Submit(std::move(operation), term)
                        # 在创建OperationDriver的过程中，会设置当复制完成时的回调
                        # OperationDriver::ReplicationFinished
                        - auto driver = NewLeaderOperationDriver(&operation, term)
                          (**driver).ExecuteAsync()
                            - Preparer::Submit
```

当在tablet上成功地复制了来自于status tablet的Apply请求之后的回调为OperationDriver::ReplicationFinished:
```
OperationDriver::ReplicationFinished
    - OperationDriver::ApplyOperation
        - OperationDriver::ApplyTask
            - Operation::Replicated
                - Status complete_status = Status::OK()
                  DoReplicated(leader_term, &complete_status)
                    - UpdateTxnOperation::DoReplicated
                        - TransactionCoordinator::ProcessReplicated
                            - TransactionCoordinator::Impl::ProcessReplicated
                                # 获取当前的transaction id
                                - id = FullyDecodeTransactionId(data.state.transaction_id())
                                # 因为当前状态为TransactionStatus::APPLYING
                                - ReplicatedApplying(*id, data)
                                    - ProcessApply(apply_data)
                                        # 首先查找对应的transaction
                                        - auto lock_and_iterator = LockAndFind(data.transaction_id, ...)
                                        # 设置transaction的commit timestamp
                                        - lock_and_iterator.transaction().SetLocalCommitTime(data.commit_ht)
                                        - applier_.ApplyIntents(data)
                                            - Tablet::ApplyIntents
                                                # 通过临时记录获取正式记录
                                                - docdb::PrepareApplyIntentsBatch(..., &regular_write_batch, ...)
                                                # 将正式记录写入rocksdb
                                                - WriteToRocksDB(&frontiers, &regular_write_batch, StorageDbType::kRegular)
                                        # 删除对应的transaction
                                        - RemoveUnlocked(lock_and_iterator.iterator, ...)
                                        # 通知对应的status tablet，当前tablet上已经成功的apply
                                        - NotifyApplied(data)
                                            # 初始化UpdateTransactionRequestPB
                                            - tserver::UpdateTransactionRequestPB req;
                                              # 设置发送的目标是status tablet
                                              req.set_tablet_id(data.status_tablet);
                                              auto& state = *req.mutable_state();
                                              state.set_transaction_id(data.transaction_id.data(), data.transaction_id.size());
                                              # 设置状态为APPLIED_IN_ONE_OF_INVOLVED_TABLETS(表示在当前tablet上成功将临时记录转换为正式记录)
                                              state.set_status(TransactionStatus::APPLIED_IN_ONE_OF_INVOLVED_TABLETS);
                                              state.add_tablets(participant_context_.tablet_id());
                                            # 准备一个rpc并发送出去
                                            - auto handle = rpcs_.Prepare()
                                              *handle = UpdateTransaction(..., &req, ...);
                                              (**handle).SendRpc()
```

#### status tablet接收到来自于某个tablet的成功apply的响应时的处理
对于来自于某个tablet的apply响应，status tablet接收到的实际上是UpdateTransactionRequestPB，其中附带了一个TransactionStatus为TransactionStatus::APPLIED_IN_ONE_OF_INVOLVED_TABLETS，在status tablet端的处理入口是：TabletServiceImpl::UpdateTransaction。
```
TabletServiceImpl::UpdateTransaction
    # 查找status tablet的leader 
    - tablet = LookupLeaderTabletOrRespond(...)
    # 生成一个UpdateTxnOperationState，req->state()类型为TransactionStatePB，其中包括
    # transaction id，TransactionStatus，参与的tablets等信息，这些信息将进一步传递给
    # UpdateTxnOperationState继承自OperationState的request_
    - auto state = std::make_unique<tablet::UpdateTxnOperationState>(tablet.peer->tablet(),
                   &req->state());
      # 设置transaction completion callback，transaction completion并不一定是成功提交了，
      # 也可能是发生了错误，该callback会发送响应给client
      state->set_completion_callback(MakeRpcOperationCompletionCallback(
        std::move(context), resp, server_->Clock()));
    # 因为当前TransactionStatus是APPLIED_IN_ONE_OF_INVOLVED_TABLETS，所以交给transaction coordinator来处理
    - tablet.peer->tablet()->transaction_coordinator()->Handle(std::move(state), tablet.leader_term)
        - TransactionCoordinator::Impl::Handle
            # TransactionStatePB，其中包括transaction id，TransactionStatus，参与的tablets等
            - auto& state = *request->request()
            # 获取transaction id
            - auto id = FullyDecodeTransactionId(state.transaction_id())
            # 到managed_transactions_中查找对应的transaction是否存在，如果不存在，则表明
            # transaction已经expired或者由于冲突而aborted了，所以直接调用transaction 
            # completion callback
            - auto status = STATUS(Expired, "Transaction expired or aborted by a conflict",
                               PgsqlError(YBPgErrorCode::YB_PG_T_R_SERIALIZATION_FAILURE));
              status = status.CloneAndAddErrorCode(TransactionError(TransactionErrorCode::kAborted));
              request->CompleteWithStatus(status);
            # 否则，在managed_transactions_中找到了，则处理之，参数@it就是在managed_transactions_
            # 中查找时返回的迭代器
            - managed_transactions_.modify(it, [&request](TransactionState& state) {
                # TransactionState::Handle
                state.Handle(std::move(request));
              });

TransactionState::Handle
    # 当前TransactionStatus是TransactionStatus::APPLIED_IN_ONE_OF_INVOLVED_TABLETS
    - auto status = AppliedInOneOfInvolvedTablets(state);
        # 减少TransactionCoordinator::Impl::tablets_with_not_applied_intents_的计数，如果该计数降为0
        # 则提交一个新的UpdateTxnOperationState请求以将TransactionStatus修改为TransactionStatus::
        # APPLIED_IN_ALL_INVOLVED_TABLETS
        -   if (!it->second.all_intents_applied) {
              --tablets_with_not_applied_intents_;
              it->second.all_intents_applied = true;
              # 当前事务相关的所有的tablets都已经成功的Apply
              if (tablets_with_not_applied_intents_ == 0) {
                SubmitUpdateStatus(TransactionStatus::APPLIED_IN_ALL_INVOLVED_TABLETS);
                    # 初始化TransactionStatePB
                    -   tserver::TransactionStatePB state;
                        state.set_transaction_id(id_.data(), id_.size());
                        state.set_status(status)
                    # 创建UpdateTxnOperationState请求
                    - auto request = context_.coordinator_context().CreateUpdateTransactionState(&state)
                    # 如果正在replicate过程中，则将新生成的UpdateTxnOperationState添加到request_queue_中，
                    # 否则，将当前的UpdateTxnOperationState请求提交给raft进行复制
                    -   if (replicating_) {
                          request_queue_.push_back(std::move(request));
                        } else {
                          replicating_ = request.get();
                          # 将当前的UpdateTxnOperationState请求提交给raft进行复制，这里对应的是
                          # TabletPeer::SubmitUpdateTransaction
                          if (!context_.SubmitUpdateTransaction(std::move(request))) {
                            // Was not able to submit update transaction, for instance we are not leader.
                            // So we are not replicating.
                            replicating_ = nullptr;
                          }
                        }
              }
            }
    - context_.CompleteWithStatus(std::move(request), status)
    
TabletPeer::SubmitUpdateTransaction
    # 生成一个UpdateTxnOperation
    - auto operation = std::make_unique<tablet::UpdateTxnOperation>(std::move(state))
    # 提交给raft进行复制
    - TabletPeer::Submit(std::move(operation), term)
        # 在创建OperationDriver的过程中，会设置当复制完成时的回调OperationDriver::ReplicationFinished
        - auto driver = NewLeaderOperationDriver(&operation, term)
          (**driver).ExecuteAsync()
            # 提交给raft
            - Preparer::Submit
```

当接收到复制成功的响应之后，OperationDriver::ReplicationFinished会被调用：
```
OperationDriver::ReplicationFinished
    - OperationDriver::ApplyOperation
        - OperationDriver::ApplyTask
            - Operation::Replicated
                - UpdateTxnOperation::DoReplicated
                    - TransactionCoordinator::ProcessReplicated
                        - TransactionCoordinator::Impl::ProcessReplicated
                            - TransactionCoordinator::Impl::DoProcessReplicated
                                - TransactionState::ProcessReplicated
                                    # 当前的TransactionStatus是TransactionStatus::APPLIED_IN_ALL_INVOLVED_TABLETS
                                    - ClearRequests
                            - TransactionCoordinator::Impl::CheckCompleted
```

## FAQ
### PgTxnManager中的txn_(类型为YBTransaction)的生命周期
#### 分配
```
PgDocOp::Execute
    PgDocWriteOp::SendRequestUnlocked
        PgSession::PgApplyAsync
            PgSession::GetSession
                PgTxnManager::BeginWriteTransactionIfNecessary
                    - 如果当前没有可用的txn_，则分配一个
                        - 如果PostgresMain和TServer之间设置了共享内存，则尝试从TServer的transaction pool中拿一个，这个拿到的transaction可能已经关联到了某个status tablet，也可能还没有关联到
                        - 否则，直接创建一个
```

#### 重置
- 将txn_设置为null
    - 通过PgTxnManager::ResetTxnAndSession
    ```
    PgTxnManager::ResetTxnAndSession
        - txn_ = nullptr
    ```

在以下地方会调用PgTxnManager::ResetTxnAndSession：
```
start_xact_command
    StartTransactionCommand
        # 只有在当前全局静态变量CurrentTransactionState中的TBlockState为
        # TBLOCK_DEFAULT的情况下才会执行StartTransaction
        StartTransaction
            YBStartTransaction
                YBCPgBeginTransaction
                    PgTxnManager::BeginTransaction
                        # 在YugaByte开始事务之前，重置txn_
                        PgTxnManager::ResetTxnAndSession
    
PgTxnManager::AbortTransaction
    -PgTxnManager::ResetTxnAndSession

finish_xact_command
    CommitTransactionCommand     
        # 只有当前全局静态变量CurrentTransactionState中的TBlockState为
        # TBLOCK_END的情况下才会执行YBCCommitTransactionAndUpdateBlockState
        YBCCommitTransactionAndUpdateBlockState
            YBCCommitTransaction    
                PgTxnManager::CommitTransaction
                    PgTxnManager::ResetTxnAndSession
```

#### 总结
对于以BEGIN开始，以END结束的transaction block，在处理BEGIN命令的时候会分配txn_，在处理END命令的时候会重置txn_。

### PgTxnManager中的txn_in_progress_的维护
1. txn_in_progress_在定义的时候被初始化为false。

2. 在PgTxnManager::BeginTransaction中被设置为true。

3. 在PgTxnManager::ResetTxnAndSession中被设置为false。
   在PgTxnManager::BeginTransaction中也会调用PgTxnManager::ResetTxnAndSession，将之设置为false，但是在调用PgTxnManager::ResetTxnAndSession之后会将txn_in_progress_设置为true，代码如下：
    ```
    PgTxnManager::BeginTransaction
        - ResetTxnAndSession()
        - txn_in_progress_ = true
    ```

4. 除了PgTxnManager::BeginTransaction会调用PgTxnManager::ResetTxnAndSession之外，以下地方也会调研PgTxnManager::ResetTxnAndSession：
```
PgTxnManager::AbortTransaction/PgTxnManager::CommitTransaction
    - PgTxnManager::ResetTxnAndSession
        - txn_in_progress_ = false
```


结合“PgTxnManager中的txn_(类型为YBTransaction)的生命周期”来看，txn_in_progress_在txn_分配的时候被设置为true，则txn_重置的时候被设置为false。

### PgTxnManager中的session_(类型为YBSession)的使用
1. 在PgTxnManager::BeginTransaction中会首先将session_设置为null，然后创建一个新的YBSession，并用它来初始化session_。
```
PgTxnManager::BeginTransaction
    - ResetTxnAndSession
        - session_ = nullptr
    - StartNewSession
        - session_ = std::make_shared<YBSession>(async_client_init_->client(), clock_)
```

2. 在PgTxnManager::AbortTransaction或者PgTxnManager::CommitTransaction中都会调用PgTxnManager::ResetTxnAndSession来将它重置。
```
PgTxnManager::AbortTransaction/PgTxnManager::CommitTransaction
    - PgTxnManager::ResetTxnAndSession
        - session_ = nullptr 
```

3. 在执行事务性操作的时候，需要用到PgTxnManager中的session_。
```
PgSession::PgApplyAsync
    - PgSession::GetSessionForOp
        - GetSession(op->IsTransactional(), op->read_only() && !has_row_mark_)
            - if (transactional) {
                # 如果是事务性操作，则使用PgTxnManager中的YBSession
                YBSession* txn_session = VERIFY_RESULT(pg_txn_manager_->GetTransactionalSession());
                return txn_session
              } else {
                # 直接使用PgSession中的YBSession
                return session_.get();
              }

Result<client::YBSession*> PgTxnManager::GetTransactionalSession() {
  # 如果现在不在事务过程中，则启动一个事务，在PgTxnManager::BeginTransaction中会
  # 调用PgTxnManager::StartNewSession来初始化PgTxnManager中的YBSession
  if (!txn_in_progress_) {
    RETURN_NOT_OK(BeginTransaction());
  }
  
  # 否则，直接返回PgTxnManager中的YBSession
  return session_.get();
}
```

结合“PgTxnManager中的txn_(类型为YBTransaction)的生命周期”来看，PgTxnManager中的session_和PgTxnManager中的txn_具有相同的生命周期，或者说，在PgTxnManager中，每个YBTransaction都有自己的YBSession，在创建YBTransaction的时候，会创建YBSession，在销毁YBTransaction的时候，也会销毁YBSession。

### YBTransaction的TransactionState状态变迁
#### TransactionState::kRunning
YBTransaction的TransactionState实际上是在YBTransaction::Impl中定义，初始化为TransactionState::kRunning，也就是说分配了YBTransaction之后，它的状态就是TransactionState::kRunning。

#### TransactionState::kAborted
在YBTransaction::Impl::Abort中被设置为TransactionState::kAborted。YBTransaction::Impl::Abort的调用关系如下：
```
AbortCurrentTransaction [postgres/src/backend/access/transam/xact.c]
    AbortTransaction
        PgTxnManager::AbortTransaction
            YBTransaction::Abort
                YBTransaction::Impl::Abort
```

#### TransactionState::kCommitted
在YBTransaction::Impl::Commit中被设置为TransactionState::kCommitted。YBTransaction::Impl::Commit的调用关系如下：
```
finish_xact_command
    CommitTransactionCommand     
        # 只有当前全局静态变量CurrentTransactionState中的TBlockState为
        # TBLOCK_END的情况下才会执行YBCCommitTransactionAndUpdateBlockState
        YBCCommitTransactionAndUpdateBlockState
            YBCCommitTransaction  
                PgTxnManager::CommitTransaction
                    YBTransaction::CommitFuture
                        YBTransaction::Impl::Commit
```

#### TransactionState::kReleased
在YBTransaction::Impl::Release中被设置为TransactionState::kReleased。YBTransaction::Impl::Release的调用关系如下：
```
PgDocOp::Execute
    PgDocWriteOp::SendRequestUnlocked
        PgSession::PgApplyAsync
            PgSession::GetSession
                PgTxnManager::BeginWriteTransactionIfNecessary
                    ... // rpc
                        TabletServerServiceIf::Handle  
                            TabletServiceImpl::TakeTransaction
                                YBTransaction::Release
                                    YBTransaction::Impl::Release
```
**上面的调用关系发生在从TabletServer获取一个YBTransaction的时候，具体用途现还不清楚**。

### YBTransaction中的running_requests_的维护
在YBTransaction中running_requests_的初始值为0。

在YBTransaction::Impl::ExpectOperations中会增加它的计数(加上待执行的操作的数目)。YBTransaction::Impl::ExpectOperations的调用关系如下：
```
YBSession::FlushAsync
    Batcher::FlushAsync
        YBTransaction::ExpectOperations
            YBTransaction::Impl::ExpectOperations
```

在YBTransaction::Impl::Flushed中会减少它的计数(减去已经flush的操作的数目)。YBTransaction::Impl::Flushed的调用关系如下：
```
AsyncRpc::Finished
    Batcher::RemoveInFlightOpsAfterFlushing
        YBTransaction::Flushed
            YBTransaction::Impl::Flushed
```

### 对于以BEGIN开始，以END结束的transaction block在PostgreSQL中TBlockState和TransState的变迁
PostgreSQL中维护了一个全局的静态变量CurrentTransactionState(类型为TransactionState)，它当中有2个域：TBlockState和TransState。

假设执行如下的SQL语句：
```
1) BEGIN
2) SELECT * FROM foo
3) INSERT INTO foo VALUES (...)
4) COMMIT
```

假设CurrentTransactionState当前是初始状态，则它当中的TBlockState为TBLOCK_DEFAULT，TransState为TRANS_DEFAULT。
```
方法调用                            相应的命令              TBlockState                                 TransState值
start_xact_command                                          TBLOCK_DEFAULT                              TRANS_DEFAULT  
    StartTransactionCommand                                 
        StartTransaction                                    TBLOCK_DEFAULT -> TBLOCK_STARTED            TRANS_DEFAULT -> TRANS_START -> TRANS_INPROGRESS
            YBStartTransaction
ProcessUtility;                     BEGIN                   
    BeginTransactionBlock                                   TBLOCK_STARTED -> TBLOCK_BEGIN
finish_xact_command
    CommitTransactionCommand                                TBLOCK_BEGIN -> TBLOCK_INPROGRESS           TRANS_INPROGRESS      

start_xact_command                                                                          
    StartTransactionCommand                                 TBLOCK_INPROGRESS                           TRANS_INPROGRESS
ProcessQuery                        SELECT
finish_xact_command
    CommitTransactionCommand                                TBLOCK_INPROGRESS                           TRANS_INPROGRESS
CommandCounterIncrement    

start_xact_command                                                                          
    StartTransactionCommand                                 TBLOCK_INPROGRESS                           TRANS_INPROGRESS
ProcessQuery                        INSERT
finish_xact_command
    CommitTransactionCommand                                TBLOCK_INPROGRESS                           TRANS_INPROGRESS
CommandCounterIncrement

start_xact_command                                                                          
    StartTransactionCommand                                 TBLOCK_INPROGRESS                           TRANS_INPROGRESS
ProcessUtility                      COMMIT
    EndTransactionBlock                                     TBLOCK_INPROGRESS -> TBLOCK_END
        YBCCommitTransaction
finish_xact_command
    CommitTransactionCommand                                                                                         
        YBCCommitTransactionAndUpdateBlockState             TBLOCK_END -> TBLOCK_DEFAULT                
            YBCCommitTransaction                                                                        
                CommitTransaction                                                                       TRANS_INPROGRESS -> TRANS_COMMIT -> TRANS_DEFAULT
```
