# 提纲
[toc]

## 在1个transaction block中2条insert语句
### SQL命令
执行如下的命令：
```
# 创建一个空表
yugabyte=# create table users(id int primary key, name text, age int, address text);
CREATE TABLE
yugabyte=# select * from users;
 id | name | age | address 
----+------+-----+---------
(0 rows)

# 在执行这条语句之前添加gdb断点
yugabyte=# begin;insert into users values('1', 'alice', '20', '027');insert into users values('2', 'bob', '22', '021');end;
BEGIN
INSERT 0 1
INSERT 0 1
COMMIT
yugabyte=#
```

执行后查询(我在执行上述命令之前已经插入了4条记录)：
```
yugabyte=# select * from users;
 id | name  | age | address 
----+-------+-----+---------
  1 | alice |  20 | 027
  2 | bob   |  22 | 021
(2 rows)
```

### 断点设置
设置了以下断点：
```
b StartTransactionCommand
b StartTransaction
b YBStartTransaction
b ProcessUtility
b ProcessQuery
b BeginTransactionBlock
b EndTransactionBlock
b finish_xact_command
b CommitTransactionCommand
b CommandCounterIncrement
b YBCCommitTransactionAndUpdateBlockState
b YBCCommitTransaction
b CommitTransaction
b yb::pggate::PgTxnManager::BeginTransaction
b yb::pggate::PgTxnManager::ResetTxnAndSession
b yb::pggate::PgTxnManager::StartNewSession
b YBCExecWriteStmt
b yb::pggate::PgDocOp::Execute
b yb::pggate::PgDocWriteOp::SendRequestImpl
b yb::pggate::PgSession::RunHelper::Apply
b ExecutorStart 
b ExecutorRun
b ExecutorFinish
b ExecutorEnd
b yb::pggate::PgSession::StartOperationsBuffering
b yb::pggate::PgSession::FlushBufferedOperations
b yb::client::YBSession::Apply
b yb::client::YBSession::FlushAsync
b yb::client::internal::Batcher::ExecuteOperations
b yb::client::YBTransaction::Prepare
b yb::client::YBTransaction::Impl::RequestStatusTablet
b yb::client::TransactionManager::PickStatusTablet
b yb::client::YBTransaction::Impl::StatusTabletPicked
b yb::client::YBTransaction::Impl::LookupStatusTablet
b yb::client::YBTransaction::Impl::LookupTabletDone
b yb::client::YBTransaction::Impl::SendHeartbeat
b yb::client::YBTransaction::Impl::HeartbeatDone
b yb::client::YBTransaction::Impl::NotifyWaiters
b yb::client::internal::Batcher::TransactionReady
b yb::client::internal::Batcher::FlushBuffersIfReady
b yb::client::internal::Batcher::FlushAsync
b yb::client::internal::MetaCache::LookupTabletByKey
b yb::client::internal::Batcher::TabletLookupFinished
b yb::client::internal::Batcher::FlushBuffersIfReady
b yb::client::internal::AsyncRpc::Finished
b yb::client::internal::WriteRpc::ProcessResponseFromTserver
b yb::client::YBTransaction::Flushed
b yb::client::YBTransaction::Impl::Commit
b yb::client::YBTransaction::Impl::DoCommit
b yb::client::YBTransaction::Impl::CommitDone
```


### 执行BEGIN命令时gdb情况
```
Breakpoint 1, StartTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2790
2790	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 2, StartTransaction () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:1876
1876	in ../../../../../../../src/postgres/src/backend/access/transam/xact.c
Continuing.

Breakpoint 3, YBStartTransaction (s=0x10c77c0 <TopTransactionStateData>) at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:1852
1852	in ../../../../../../../src/postgres/src/backend/access/transam/xact.c
Continuing.

Breakpoint 14, yb::pggate::PgTxnManager::BeginTransaction (this=0x3184750) at ../../../../../../src/yb/yql/pggate/pg_txn_manager.cc:107
107	../../../../../../src/yb/yql/pggate/pg_txn_manager.cc: No such file or directory.
Continuing.

Breakpoint 15, yb::pggate::PgTxnManager::ResetTxnAndSession (this=0x3184750) at ../../../../../../src/yb/yql/pggate/pg_txn_manager.cc:290
290	in ../../../../../../src/yb/yql/pggate/pg_txn_manager.cc
Continuing.

Breakpoint 16, yb::pggate::PgTxnManager::StartNewSession (this=0x3184750) at ../../../../../../src/yb/yql/pggate/pg_txn_manager.cc:133
133	in ../../../../../../src/yb/yql/pggate/pg_txn_manager.cc
Continuing.

Breakpoint 4, ProcessUtility (pstmt=0x30febe0, queryString=0x30fda98 "begin;", context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x30feed0, 
    completionTag=0x7fff545dcd50 "") at ../../../../../../src/postgres/src/backend/tcop/utility.c:360
360	../../../../../../src/postgres/src/backend/tcop/utility.c: No such file or directory.
Continuing.

Breakpoint 25, yb::pggate::PgSession::StartOperationsBuffering (this=0x3106ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:753
753	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 6, BeginTransactionBlock () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:3575
3575	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 26, yb::pggate::PgSession::FlushBufferedOperations (this=0x3106ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:767
767	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.

Breakpoint 9, CommitTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2897
2897	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.
```

### 第1次执行insert语句时的gdb情况
```
Breakpoint 1, StartTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2790
2790	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 5, ProcessQuery (plan=0x27a48c8, sourceText=0x21ada98 "insert into users values('1', 'alice', '20', '027');", params=0x0, queryEnv=0x0, dest=0x27a4a40, 
    completionTag=0x7ffe39d30220 "", isSingleRowModifyTxn=false) at ../../../../../../src/postgres/src/backend/tcop/pquery.c:154
154	../../../../../../src/postgres/src/backend/tcop/pquery.c: No such file or directory.
Continuing.

Breakpoint 21, ExecutorStart (queryDesc=0x21b4f28, eflags=0) at ../../../../../../src/postgres/src/backend/executor/execMain.c:145
145	../../../../../../src/postgres/src/backend/executor/execMain.c: No such file or directory.
Continuing.

Breakpoint 25, yb::pggate::PgSession::StartOperationsBuffering (this=0x3106ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:753
753	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 22, ExecutorRun (queryDesc=0x21b4f28, direction=ForwardScanDirection, count=0, execute_once=true)
    at ../../../../../../src/postgres/src/backend/executor/execMain.c:308
308	in ../../../../../../src/postgres/src/backend/executor/execMain.c
Continuing.

Breakpoint 17, YBCExecWriteStmt (ybc_stmt=0x23d82a0, rel=0x2d2fc98, rows_affected_count=0x0) at ../../../../../../src/postgres/src/backend/executor/ybcModifyTable.c:279
279	../../../../../../src/postgres/src/backend/executor/ybcModifyTable.c: No such file or directory.
Continuing.

Breakpoint 18, yb::pggate::PgDocOp::Execute (this=0x25f64b0, force_non_bufferable=false) at ../../../../../../src/yb/yql/pggate/pg_doc_op.cc:134
134	../../../../../../src/yb/yql/pggate/pg_doc_op.cc: No such file or directory.
Continuing.

Breakpoint 19, yb::pggate::PgDocWriteOp::SendRequestImpl (this=0x25f64b0, force_non_bufferable=false) at ../../../../../../src/yb/yql/pggate/pg_doc_op.cc:562
562	in ../../../../../../src/yb/yql/pggate/pg_doc_op.cc
Continuing.

# 因为我们在yb::client::YBSession::Apply处设置了断点，但是yb::pggate::PgSession::
# RunHelper::Apply之后并没有在yb::pggate::PgSession::RunHelper::Apply处break住，
# 所以可以肯定的是在yb::pggate::PgSession::RunHelper::Apply中数据被buffered了
Breakpoint 20, yb::pggate::PgSession::RunHelper::Apply (this=0x7ffe39d2f4e0, op=..., relation_id=..., read_time=0x25f64d0, force_non_bufferable=false)
    at ../../../../../../src/yb/yql/pggate/pg_session.cc:265
265	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 23, ExecutorFinish (queryDesc=0x21b4f28) at ../../../../../../src/postgres/src/backend/executor/execMain.c:408
408	../../../../../../src/postgres/src/backend/executor/execMain.c: No such file or directory.
Continuing.

Breakpoint 25, yb::pggate::PgSession::FlushBufferedOperations (this=0x21b6ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:767
767	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

# 在buffer中的操作真正执行之前，需要先找到status tablet
Breakpoint 30, yb::client::YBTransaction::Impl::RequestStatusTablet (this=0x23d4760, deadline=...) at ../../../../../src/yb/client/transaction.cc:702
702	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.

# 查询status tablet信息，从调用栈来看，在本地meta cache中没有找到，所以要发送rpc
Breakpoint 33, yb::client::YBTransaction::Impl::LookupStatusTablet (this=0x23d4760, tablet_id=..., deadline=..., transaction=...)
    at ../../../../../src/yb/client/transaction.cc:736
736	in ../../../../../src/yb/client/transaction.cc
Continuing.

# 至此，还没有接收到关于查询status tablet的响应，但是rpc响应时异步的，所以不会阻塞其它步骤

# 将操作添加到Batcher中
Breakpoint 26, yb::client::YBSession::Apply (this=0x28db300, yb_op=...) at ../../../../../src/yb/client/session.cc:213
213	../../../../../src/yb/client/session.cc: No such file or directory.
Continuing.

Breakpoint 41, yb::client::internal::Batcher::Add (this=0x3539f00, yb_op=...) at ../../../../../src/yb/client/batcher.cc:282
282	../../../../../src/yb/client/batcher.cc: No such file or directory.
Continuing.

# 查找操作对应的tablet
Breakpoint 41, yb::client::internal::MetaCache::LookupTabletByKey(yb::client::YBTable const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char>
 > const&, std::chrono::time_point<yb::CoarseMonoClock, std::chrono::duration<long, std::ratio<1l, 1000000000l> > >, std::function<void (yb::Result<scoped_refptr<yb::clien
t::internal::RemoteTablet> > const&)>) (this=0x7f89f0010ef0, table=0x28dd550, partition_key=..., deadline=..., callback=...)
    at ../../../../../src/yb/client/meta_cache.cc:1035
1035	../../../../../src/yb/client/meta_cache.cc: No such file or directory.
Continuing.

# 查找到操作对应的tablet之后的回调
Breakpoint 42, yb::client::internal::Batcher::TabletLookupFinished (this=0x23df1b0, op=..., lookup_result=...) at ../../../../../src/yb/client/batcher.cc:380
380	../../../../../src/yb/client/batcher.cc: No such file or directory.
Continuing.

# yb::client::internal::Batcher::TabletLookupFinished会触发此方法，但是此时Batcher的
# 状态不是BatcherState::kResolvingTablets，因为yb::client::internal::Batcher::FlushAsync
# 还没有被调用，只有在yb::client::internal::Batcher::FlushAsync中才会将Batcher的状态
# 设置为BatcherState::kResolvingTablets，所以，这里FlushBuffersIfReady并没有做什么事情
Breakpoint 39, yb::client::internal::Batcher::FlushBuffersIfReady (this=0x23df1b0) at ../../../../../src/yb/client/batcher.cc:470
470	in ../../../../../src/yb/client/batcher.cc
Continuing.

# 这里会结束当前的Batcher
Breakpoint 27, yb::client::YBSession::FlushAsync(boost::function<void (yb::Status const&)>) (this=0x28db300, callback=...) at ../../../../../src/yb/client/session.cc:128
128	../../../../../src/yb/client/session.cc: No such file or directory.
Continuing.

# 这里会将Batcher状态设置为设置为BatcherState::kResolvingTablets
Breakpoint 40, yb::client::internal::Batcher::FlushAsync(boost::function<void (yb::Status const&)>) (this=0x23df1b0, callback=...)
    at ../../../../../src/yb/client/batcher.cc:252
252	../../../../../src/yb/client/batcher.cc: No such file or directory.
Continuing.

# 此时Batcher状态为BatcherState::kResolvingTablets，所以会进入yb::client::internal::
# Batcher::ExecuteOperations
Breakpoint 39, yb::client::internal::Batcher::FlushBuffersIfReady (this=0x23df1b0) at ../../../../../src/yb/client/batcher.cc:470
470	in ../../../../../src/yb/client/batcher.cc
Continuing.

# 执行操作(实际上是将操作生成RPC发送给TServer执行)
Breakpoint 28, yb::client::internal::Batcher::ExecuteOperations (this=0x23df1b0, initial=...) at ../../../../../src/yb/client/batcher.cc:505
505	in ../../../../../src/yb/client/batcher.cc
Continuing.

# 因为，自从上次执行yb::client::YBTransaction::Impl::RequestStatusTablet之后，一直没有接收到
# 响应，所以现在的YBTransaction::Impl::ready为false，也就是说和status tablet之间的联系还没建
# 立好，所以这里会再次执行yb::client::YBTransaction::Impl::RequestStatusTablet，并且在
# yb::client::YBTransaction::Prepare传入的回调Batcher::TransactionReady也会被添加到等待队列中
Breakpoint 29, yb::client::YBTransaction::Prepare(std::vector<std::shared_ptr<yb::client::internal::InFlightOp>, std::allocator<std::shared_ptr<yb::client::internal::InFli
ghtOp> > > const&, yb::StronglyTypedBool<yb::client::ForceConsistentRead_Tag>, std::chrono::time_point<yb::CoarseMonoClock, std::chrono::duration<long, std::ratio<1l, 1000
000000l> > >, yb::StronglyTypedBool<yb::client::Initial_Tag>, boost::function<void (yb::Status const&)>, yb::TransactionMetadata*) (this=0x28db9f0, ops=..., 
    force_consistent_read=..., deadline=..., initial=..., waiter=..., metadata=0x23df2a0) at ../../../../../src/yb/client/transaction.cc:1031
1031	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.

# 因为yb::client::YBTransaction::Prepare中YBTransaction::Impl::ready为false而触发的
Breakpoint 30, yb::client::YBTransaction::Impl::RequestStatusTablet (this=0x23d4760, deadline=...) at ../../../../../src/yb/client/transaction.cc:702
702	in ../../../../../src/yb/client/transaction.cc
Continuing.
[Switching to Thread 0x7f89fb7fe700 (LWP 5103)]

# 接收到了lookup status tablet的回调，终于查询到了status tablet信息
Breakpoint 34, yb::client::YBTransaction::Impl::LookupTabletDone (this=0x23d4760, result=..., transaction=...) at ../../../../../src/yb/client/transaction.cc:742
742	in ../../../../../src/yb/client/transaction.cc
Continuing.

# 等待队列中的回调被执行，会进一步调用yb::client::internal::Batcher::ExecuteOperations
Breakpoint 38, yb::client::internal::Batcher::TransactionReady (this=0x23df1b0, status=..., self=...) at ../../../../../src/yb/client/batcher.cc:456
456	../../../../../src/yb/client/batcher.cc: No such file or directory.
Continuing.


Breakpoint 28, yb::client::internal::Batcher::ExecuteOperations (this=0x23df1b0, initial=...) at ../../../../../src/yb/client/batcher.cc:505
505	in ../../../../../src/yb/client/batcher.cc
Continuing.

# 现在的YBTransaction::Impl::ready为true了，可以生成RPC，然后发送给相应的TServer执行了，
# 发送RPC的过程中没有设置断点，只在接收到RPC响应的地方设置了断点，但是RPC响应是异步的
Breakpoint 29, yb::client::YBTransaction::Prepare(std::vector<std::shared_ptr<yb::client::internal::InFlightOp>, std::allocator<std::shared_ptr<yb::client::internal::InFli
ghtOp> > > const&, yb::StronglyTypedBool<yb::client::ForceConsistentRead_Tag>, std::chrono::time_point<yb::CoarseMonoClock, std::chrono::duration<long, std::ratio<1l, 1000
000000l> > >, yb::StronglyTypedBool<yb::client::Initial_Tag>, boost::function<void (yb::Status const&)>, yb::TransactionMetadata*) (this=0x28db9f0, ops=..., 
    force_consistent_read=..., deadline=..., initial=..., waiter=..., metadata=0x23df2a0) at ../../../../../src/yb/client/transaction.cc:1031
1031	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.

# 发送心跳给status tablet，这是由前面的yb::client::YBTransaction::Impl::LookupTabletDone触发的
Breakpoint 35, yb::client::YBTransaction::Impl::SendHeartbeat (this=0x23d4760, status=yb::PENDING, id=..., weak_transaction=...)
    at ../../../../../src/yb/client/transaction.cc:792
792	in ../../../../../src/yb/client/transaction.cc
Continuing.

# 接收到status tablet的心跳的响应
Breakpoint 36, yb::client::YBTransaction::Impl::HeartbeatDone (this=0x23d4760, status=..., response=..., transaction_status=yb::PENDING, transaction=...)
    at ../../../../../src/yb/client/transaction.cc:853
853	in ../../../../../src/yb/client/transaction.cc
Continuing.

# 接收到TServer端操作完成的响应后，yb::client::internal::AsyncRpc::Finished被调用
Breakpoint 44, yb::client::internal::AsyncRpc::Finished (this=0x7f89f0067280, status=...) at ../../../../../src/yb/client/async_rpc.cc:165
165	../../../../../src/yb/client/async_rpc.cc: No such file or directory.
Continuing.

# 处理响应，主要是将RPC中的响应映射到RPC中包含的每个操作的响应上
Breakpoint 45, yb::client::internal::WriteRpc::ProcessResponseFromTserver (this=0x7f89f0067280, status=...) at ../../../../../src/yb/client/async_rpc.cc:562
562	in ../../../../../src/yb/client/async_rpc.cc
Continuing.

Breakpoint 46, yb::client::YBTransaction::Flushed (this=0x28db9f0, ops=..., used_read_time=..., status=...) at ../../../../../src/yb/client/transaction.cc:1040
1040	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.
[Switching to Thread 0x7f8a30b57b40 (LWP 5095)]

Breakpoint 24, ExecutorEnd (queryDesc=0x21b4f28) at ../../../../../../src/postgres/src/backend/executor/execMain.c:472
472	../../../../../../src/postgres/src/backend/executor/execMain.c: No such file or directory.
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.

Breakpoint 9, CommitTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2897
2897	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

# 增加当前transaction block中命令的数目
Breakpoint 10, CommandCounterIncrement () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:951
951	in ../../../../../../../src/postgres/src/backend/access/transam/xact.c
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.
```

### 第2次执行insert语句时的gdb情况
```
Breakpoint 1, StartTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2790
2790	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 5, ProcessQuery (plan=0x27a4230, sourceText=0x21ada98 "insert into users values('2', 'bob', '22', '021');", params=0x0, queryEnv=0x0, dest=0x27a43a8, 
    completionTag=0x7ffe39d30220 "", isSingleRowModifyTxn=false) at ../../../../../../src/postgres/src/backend/tcop/pquery.c:154
154	../../../../../../src/postgres/src/backend/tcop/pquery.c: No such file or directory.
Continuing.

Breakpoint 21, ExecutorStart (queryDesc=0x21b4f28, eflags=0) at ../../../../../../src/postgres/src/backend/executor/execMain.c:145
145	../../../../../../src/postgres/src/backend/executor/execMain.c: No such file or directory.
Continuing.

Breakpoint 25, yb::pggate::PgSession::StartOperationsBuffering (this=0x3106ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:753
753	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 22, ExecutorRun (queryDesc=0x21b4f28, direction=ForwardScanDirection, count=0, execute_once=true)
    at ../../../../../../src/postgres/src/backend/executor/execMain.c:308
308	in ../../../../../../src/postgres/src/backend/executor/execMain.c
Continuing.

Breakpoint 17, YBCExecWriteStmt (ybc_stmt=0x2a19750, rel=0x2d2fc98, rows_affected_count=0x0) at ../../../../../../src/postgres/src/backend/executor/ybcModifyTable.c:279
279	../../../../../../src/postgres/src/backend/executor/ybcModifyTable.c: No such file or directory.
Continuing.

Breakpoint 18, yb::pggate::PgDocOp::Execute (this=0x2a19520, force_non_bufferable=false) at ../../../../../../src/yb/yql/pggate/pg_doc_op.cc:134
134	../../../../../../src/yb/yql/pggate/pg_doc_op.cc: No such file or directory.
Continuing.

Breakpoint 19, yb::pggate::PgDocWriteOp::SendRequestImpl (this=0x2a19520, force_non_bufferable=false) at ../../../../../../src/yb/yql/pggate/pg_doc_op.cc:562
562	in ../../../../../../src/yb/yql/pggate/pg_doc_op.cc
Continuing.

Breakpoint 20, yb::pggate::PgSession::RunHelper::Apply (this=0x7ffe39d2f4e0, op=..., relation_id=..., read_time=0x2a19540, force_non_bufferable=false)
    at ../../../../../../src/yb/yql/pggate/pg_session.cc:265
265	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 23, ExecutorFinish (queryDesc=0x21b4f28) at ../../../../../../src/postgres/src/backend/executor/execMain.c:408
408	../../../../../../src/postgres/src/backend/executor/execMain.c: No such file or directory.
Continuing.

Breakpoint 25, yb::pggate::PgSession::FlushBufferedOperations (this=0x21b6ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:767
767	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

# 相较于“第1次执行insert语句时的gdb情况”，这里没有yb::client::YBTransaction::Impl::
# RequestStatusTablet了，因为在第1次执行insert语句时，已经和status tablet建立好联系了

# 将操作添加到Batcher中
Breakpoint 26, yb::client::YBSession::Apply (this=0x28db300, yb_op=...) at ../../../../../src/yb/client/session.cc:213
213	../../../../../src/yb/client/session.cc: No such file or directory.
Continuing.

Breakpoint 41, yb::client::internal::Batcher::Add (this=0x3539f00, yb_op=...) at ../../../../../src/yb/client/batcher.cc:282
282	../../../../../src/yb/client/batcher.cc: No such file or directory.
Continuing.

# 查找操作对应的tablet
Breakpoint 41, yb::client::internal::MetaCache::LookupTabletByKey(yb::client::YBTable const*, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char>
 > const&, std::chrono::time_point<yb::CoarseMonoClock, std::chrono::duration<long, std::ratio<1l, 1000000000l> > >, std::function<void (yb::Result<scoped_refptr<yb::clien
t::internal::RemoteTablet> > const&)>) (this=0x7f89f0010ef0, table=0x28dd550, partition_key=..., deadline=..., callback=...)
    at ../../../../../src/yb/client/meta_cache.cc:1035
1035	../../../../../src/yb/client/meta_cache.cc: No such file or directory.
Continuing.

# 查找到操作对应的tablet之后的回调
Breakpoint 42, yb::client::internal::Batcher::TabletLookupFinished (this=0x22be370, op=..., lookup_result=...) at ../../../../../src/yb/client/batcher.cc:380
380	../../../../../src/yb/client/batcher.cc: No such file or directory.
Continuing.

# yb::client::internal::Batcher::TabletLookupFinished会触发此方法，但是此时Batcher的
# 状态不是BatcherState::kResolvingTablets，因为yb::client::internal::Batcher::FlushAsync
# 还没有被调用，只有在yb::client::internal::Batcher::FlushAsync中才会将Batcher的状态
# 设置为BatcherState::kResolvingTablets，所以，这里FlushBuffersIfReady并没有做什么事情
Breakpoint 39, yb::client::internal::Batcher::FlushBuffersIfReady (this=0x22be370) at ../../../../../src/yb/client/batcher.cc:470
470	in ../../../../../src/yb/client/batcher.cc
Continuing.

# 这类会结束当前的Batcher
Breakpoint 27, yb::client::YBSession::FlushAsync(boost::function<void (yb::Status const&)>) (this=0x28db300, callback=...) at ../../../../../src/yb/client/session.cc:128
128	../../../../../src/yb/client/session.cc: No such file or directory.
Continuing.

# 这里会将Batcher状态设置为设置为BatcherState::kResolvingTablets
Breakpoint 40, yb::client::internal::Batcher::FlushAsync(boost::function<void (yb::Status const&)>) (this=0x23df1b0, callback=...)
    at ../../../../../src/yb/client/batcher.cc:252
252	../../../../../src/yb/client/batcher.cc: No such file or directory.
Continuing.

# 此时Batcher状态为BatcherState::kResolvingTablets，所以会进入yb::client::internal::
# Batcher::ExecuteOperations
Breakpoint 39, yb::client::internal::Batcher::FlushBuffersIfReady (this=0x23df1b0) at ../../../../../src/yb/client/batcher.cc:470
470	in ../../../../../src/yb/client/batcher.cc
Continuing.

# 执行操作(实际上是将操作生成RPC发送给TServer执行)
Breakpoint 28, yb::client::internal::Batcher::ExecuteOperations (this=0x23df1b0, initial=...) at ../../../../../src/yb/client/batcher.cc:505
505	in ../../../../../src/yb/client/batcher.cc
Continuing.

# yb::client::YBTransaction::Prepare由yb::client::internal::Batcher::ExecuteOperations
# 触发，这里跟“第1次执行insert语句时的gdb情况”不同，在第1次执行insert之后，这里的
# YBTransaction::Impl::ready已经为true了，所以yb::client::YBTransaction::Prepare
# 会返回true，然后在yb::client::internal::Batcher::ExecuteOperations会将Batcher的状态
# 设置为BatcherState::kTransactionReady，最后会生成RPC并发送给相应的TServer
Breakpoint 29, yb::client::YBTransaction::Prepare(std::vector<std::shared_ptr<yb::client::internal::InFlightOp>, std::allocator<std::shared_ptr<yb::client::internal::InFli
ghtOp> > > const&, yb::StronglyTypedBool<yb::client::ForceConsistentRead_Tag>, std::chrono::time_point<yb::CoarseMonoClock, std::chrono::duration<long, std::ratio<1l, 1000
000000l> > >, yb::StronglyTypedBool<yb::client::Initial_Tag>, boost::function<void (yb::Status const&)>, yb::TransactionMetadata*) (this=0x28db9f0, ops=..., 
    force_consistent_read=..., deadline=..., initial=..., waiter=..., metadata=0x22be460) at ../../../../../src/yb/client/transaction.cc:1031
1031	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.
[Switching to Thread 0x7f89fb7fe700 (LWP 5103)]

# 接收到TServer端操作完成的响应后，yb::client::internal::AsyncRpc::Finished被调用
Breakpoint 44, yb::client::internal::AsyncRpc::Finished (this=0x7f89f0067280, status=...) at ../../../../../src/yb/client/async_rpc.cc:165
165	../../../../../src/yb/client/async_rpc.cc: No such file or directory.
Continuing.

# 处理响应，主要是将RPC中的响应映射到RPC中包含的每个操作的响应上
Breakpoint 45, yb::client::internal::WriteRpc::ProcessResponseFromTserver (this=0x7f89f0067280, status=...) at ../../../../../src/yb/client/async_rpc.cc:562
562	in ../../../../../src/yb/client/async_rpc.cc
Continuing.

Breakpoint 46, yb::client::YBTransaction::Flushed (this=0x28db9f0, ops=..., used_read_time=..., status=...) at ../../../../../src/yb/client/transaction.cc:1040
1040	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.
[Switching to Thread 0x7f8a30b57b40 (LWP 5095)]

Breakpoint 24, ExecutorEnd (queryDesc=0x21b4f28) at ../../../../../../src/postgres/src/backend/executor/execMain.c:472
472	../../../../../../src/postgres/src/backend/executor/execMain.c: No such file or directory.
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.

Breakpoint 9, CommitTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2897
2897	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 10, CommandCounterIncrement () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:951
951	in ../../../../../../../src/postgres/src/backend/access/transam/xact.c
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.
```

### 执行END命令时的gdb情况
```
Breakpoint 1, StartTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2790
2790	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 4, ProcessUtility (pstmt=0x21aebe0, queryString=0x21ada98 "end;", context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x21aeed0, 
    completionTag=0x7ffe39d30220 "") at ../../../../../../src/postgres/src/backend/tcop/utility.c:360
360	../../../../../../src/postgres/src/backend/tcop/utility.c: No such file or directory.
Continuing.

Breakpoint 25, yb::pggate::PgSession::StartOperationsBuffering (this=0x3106ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:753
753	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 7, EndTransactionBlock () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:3695
3695	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

# 提交事务，这是由EndTransactionBlock触发的，但是在YBCCommitTransaction之后没有执行
# CommitTransaction
Breakpoint 12, YBCCommitTransaction () at ../../../../../../../src/postgres/src/backend/utils/misc/pg_yb_utils.c:427
427	../../../../../../../src/postgres/src/backend/utils/misc/pg_yb_utils.c: No such file or directory.
Continuing.

# 由YBCCommitTransaction调用
Breakpoint 47, yb::client::YBTransaction::Impl::Commit(std::chrono::time_point<yb::CoarseMonoClock, std::chrono::duration<long, std::ratio<1l, 1000000000l> > >, yb::Strong
lyTypedBool<yb::client::SealOnly_Tag>, boost::function<void (yb::Status const&)>) (this=0x23d4760, deadline=..., seal_only=..., callback=...)
    at ../../../../../src/yb/client/transaction.cc:338
338	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.

# 由yb::client::YBTransaction::Impl::Commit调用
Breakpoint 48, yb::client::YBTransaction::Impl::DoCommit (this=0x23d4760, deadline=..., seal_only=..., status=..., transaction=...)
    at ../../../../../src/yb/client/transaction.cc:583
583	in ../../../../../src/yb/client/transaction.cc
Continuing.
[Switching to Thread 0x7f89fa7fc700 (LWP 5357)]

# 提交成功后的回调
Breakpoint 49, yb::client::YBTransaction::Impl::CommitDone (this=0x23d4760, status=..., response=..., transaction=...) at ../../../../../src/yb/client/transaction.cc:672
672	in ../../../../../src/yb/client/transaction.cc
Continuing.
[Switching to Thread 0x7f8a30b57b40 (LWP 5095)]

# 成功提交事务之后，重置PgTxnManager中的YBTransaction和YBSession
Breakpoint 15, yb::pggate::PgTxnManager::ResetTxnAndSession (this=0x2234750) at ../../../../../../src/yb/yql/pggate/pg_txn_manager.cc:290
290	../../../../../../src/yb/yql/pggate/pg_txn_manager.cc: No such file or directory.
Continuing.

Breakpoint 25, yb::pggate::PgSession::FlushBufferedOperations (this=0x21b6ff0) at ../../../../../../src/yb/yql/pggate/pg_session.cc:767
767	../../../../../../src/yb/yql/pggate/pg_session.cc: No such file or directory.
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.

Breakpoint 9, CommitTransactionCommand () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2897
2897	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

# 提交事务，这是由CommitTransactionCommand触发的，跟由EndTransactionBlock触发的提交不同，
# YBCCommitTransactionAndUpdateBlockState会分别调用YBCCommitTransaction和
# CommitTransaction，但是EndTransactionBlock只会调用YBCCommitTransaction
Breakpoint 11, YBCCommitTransactionAndUpdateBlockState () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2873
2873	in ../../../../../../../src/postgres/src/backend/access/transam/xact.c
Continuing.

# 此时因为在EndTransactionBlock -> YBCCommitTransaction已经执行过提交了，所以这里没啥可做
Breakpoint 12, YBCCommitTransaction () at ../../../../../../../src/postgres/src/backend/utils/misc/pg_yb_utils.c:427
427	../../../../../../../src/postgres/src/backend/utils/misc/pg_yb_utils.c: No such file or directory.
Continuing.

# 由YBCCommitTransactionAndUpdateBlockState触发
Breakpoint 13, CommitTransaction () at ../../../../../../../src/postgres/src/backend/access/transam/xact.c:2026
2026	../../../../../../../src/postgres/src/backend/access/transam/xact.c: No such file or directory.
Continuing.

Breakpoint 8, finish_xact_command () at ../../../../../../src/postgres/src/backend/tcop/postgres.c:2543
2543	../../../../../../src/postgres/src/backend/tcop/postgres.c: No such file or directory.
Continuing.
[Switching to Thread 0x7f8a11ccc700 (LWP 5099)]

Breakpoint 36, yb::client::YBTransaction::Impl::HeartbeatDone(yb::Status const&, yb::tserver::UpdateTransactionResponsePB const&, yb::TransactionStatus, std::shared_ptr<yb
::client::YBTransaction> const&)::{lambda(yb::Status const&)#1}::operator()(yb::Status const&) const (__closure=0x7f89f0069e88)
    at ../../../../../src/yb/client/transaction.cc:866
866	../../../../../src/yb/client/transaction.cc: No such file or directory.
Continuing.

Breakpoint 35, yb::client::YBTransaction::Impl::SendHeartbeat (this=0x23d4760, status=yb::PENDING, id=..., weak_transaction=...)
    at ../../../../../src/yb/client/transaction.cc:792
792	in ../../../../../src/yb/client/transaction.cc
Continuing.

Breakpoint 36, yb::client::YBTransaction::Impl::HeartbeatDone(yb::Status const&, yb::tserver::UpdateTransactionResponsePB const&, yb::TransactionStatus, std::shared_ptr<yb
::client::YBTransaction> const&)::{lambda(yb::Status const&)#1}::~shared_ptr() (this=0x7f89f0069e88, __in_chrg=<optimized out>)
    at ../../../../../src/yb/client/transaction.cc:865
865	in ../../../../../src/yb/client/transaction.cc
Continuing.
```

### 总结
1. 在所有命令中(包括BEGIN和END命令)都会执行的方法
```
start_xact_command 
StartTransactionCommand
finish_xact_command 
CommitTransactionCommand
yb::pggate::PgSession::StartOperationsBuffering
yb::pggate::PgSession::FlushBufferedOperations
```

2. 只在BEGIN和END命令中执行的方法
```
ProcessUtility
yb::pggate::PgTxnManager::BeginTransaction
yb::pggate::PgTxnManager::ResetTxnAndSession
yb::pggate::PgTxnManager::StartNewSession
```

3. 只在BEGIN命令中执行的方法
```
StartTransaction
YBStartTransaction
BeginTransactionBlock
```

4. 只在END命令中执行的方法
```
EndTransactionBlock
YBCCommitTransactionAndUpdateBlockState 
YBCCommitTransaction 
yb::client::YBTransaction::Impl::Commit
yb::client::YBTransaction::Impl::DoCommit
yb::client::YBTransaction::Impl::CommitDone
```

5. 在transaction block中所有insert命令都会执行的方法：
```
ExecutorStart 
ExecutorRun
ExecutorFinish
ExecutorEnd
ProcessQuery
yb::pggate::PgDocOp::Execute
yb::pggate::PgDocWriteOp::SendRequestImpl
yb::pggate::PgSession::RunHelper::Apply
yb::client::internal::AsyncRpc::Finished
yb::client::internal::WriteRpc::ProcessResponseFromTserver
yb::client::YBTransaction::Flushed
CommandCounterIncrement
```

6. 关于yb::client::YBTransaction::Impl::RequestStatusTablet的说明

    在“第1次执行insert语句时的gdb情况”中，yb::client::YBTransaction::Impl::RequestStatusTablet被调用了2次，第一次调用的栈情况为：
    ```
    ExecutorFinish 
        PgSession::FlushBufferedOperations 
            PgSession::FlushBufferedOperationsImpl
                PgSession::GetSession
                    PgTxnManager::BeginWriteTransactionIfNecessary
                        YBTransaction::Take
                            YBTransaction::Impl::StartHeartbeat
                                # 查询status tablet信息
                                YBTransaction::Impl::RequestStatusTablet
    ```

    第二次调用的栈情况为：
    ```
    ExecutorFinish
        PgSession::FlushBufferedOperations
            PgSession::FlushBufferedOperationsImpl
                YBSession::FlushAsync
                    Batcher::FlushAsync
                        Batcher::FlushBuffersIfReady
                            Batcher::ExecuteOperations
                                yb::client::YBTransaction::Prepare
                                    YBTransaction::Impl::RequestStatusTablet
    ```

    在“第2次执行insert语句时的gdb情况”中，yb::client::YBTransaction::Impl::RequestStatusTablet则没有被调用，因为在第1次执行insert语句的时候，已经跟status tablet建立好联系了。

7. 关于yb::client::YBTransaction::Prepare

    在“第1次执行insert语句时的gdb情况”中，yb::client::YBTransaction::Prepare会被执行2次：第1次执行后，YBTransaction::Impl::ready为false，第2次执行后，YBTransaction::Impl::ready为true，也就是说跟status tablet建立好联系了，可以生成RPC并发送给对应的Tablet(注意：不是status tablet)执行了。

    在“第2次执行insert语句时的gdb情况”中，yb::client::YBTransaction::Prepare只被执行1次，且在执行时，YBTransaction::Impl::ready为true，可以生成RPC并发送给对应的Tablet(注意：不是status tablet)执行了。

8. 在执行END命令过程中提交2次

    第1次是由EndTransactionBlock触发：
    ```
    EndTransactionBlock
        YBCCommitTransaction
            yb::client::YBTransaction::Impl::Commit
                yb::client::YBTransaction::Impl::DoCommit
    ```

    第2次是由CommitTransactionCommand触发：
    ```
    CommitTransactionCommand
        # 因为在前面已经做过1次了，这里YBCCommitTransaction没做什么？
        YBCCommitTransaction
        CommitTransaction
    ```