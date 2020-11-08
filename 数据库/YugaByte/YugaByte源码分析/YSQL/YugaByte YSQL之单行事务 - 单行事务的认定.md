# 提纲
[toc]

## 什么情况下一个事务会被认定是单行事务
对于insert语句，如果EState::es_yb_is_single_row_modify_txn为true，则被认为是单行事务。那么，EState中记录的es_yb_is_single_row_modify_txn是如何被设置的呢？
```
ProcessQuery
    # 如果满足以下条件，则es_yb_is_single_row_modify_txn被设置为true：
    # 1. isSingleRowModifyTxn为true，它在以下条件下为true：
    #    这是一个modify操作，也就是说，是insert，update或者delete操作
    #    只修改一个target table
    #    没有ON CONFLICT和WITH语句
    #    源数据来自于VALUES语句，且在VALUES中只设置一个value(这里一个value指的是一行，
    #    可以设置一行中的一个或者多个列)
    #    VALUES语句中的值是常量或者bind markers(这是啥？)
    # 2. 不在transaction block中
    # 3. 只修改一个table
    # 4. 被修改的table上没有触发器
    # 5. 被修改的table上没有索引二级索引
    - queryDesc->estate->es_yb_is_single_row_modify_txn =
		isSingleRowModifyTxn && queryDesc->estate->es_num_result_relations == 1 &&
		YBCIsSingleRowTxnCapableRel(&queryDesc->estate->es_result_relations[0])
```

## EState::es_yb_is_single_row_modify_txn最终在哪里生效
EState::es_yb_is_single_row_modify_txn是如何被使用的呢？
```
Oid YBCHeapInsert(TupleTableSlot *slot,
				  HeapTuple tuple,
				  EState *estate)
{
	ResultRelInfo *resultRelInfo = estate->es_result_relation_info;
	Relation resultRelationDesc = resultRelInfo->ri_RelationDesc;

	if (estate->es_yb_is_single_row_modify_txn)
	{
	    # 如果是单行事务，则会进入YBCExecuteNonTxnInsert
		return YBCExecuteNonTxnInsert(resultRelationDesc, slot->tts_tupleDescriptor, tuple);
	}
	else
	{
	    # 否则，进入YBCExecuteInsert
		return YBCExecuteInsert(resultRelationDesc, slot->tts_tupleDescriptor, tuple);
	}
}
```

可见，如果EState::es_yb_is_single_row_modify_txn为true，则会执行单行事务，调用YBCExecuteNonTxnInsert，否则执行分布式事务，调用YBCExecuteInsert。但是通过对YBCExecuteNonTxnInsert和YBCExecuteInsert的调用分析，发现它们最终都会调用YBCExecuteInsertInternal，但是在传递给YBCExecuteInsertInternal的最后一个参数上有所区别，如下：
```
YBCExecuteNonTxnInsert
    - YBCExecuteInsertInternal(..., true /* is_single_row_txn */)
    
YBCExecuteInsert    
    - YBCExecuteInsertInternal(..., false /* is_single_row_txn */)
```

那么YBCExecuteInsertInternal中的最后一个参数是如何影响单行事务和分布式事务的后续执行的呢，且看YBCExecuteInsertInternal中的最后一个参数is_single_row_txn在哪里起作用：
```
YBCExecuteInsertInternal(Relation rel,
                        TupleDesc tupleDesc,
                        HeapTuple tuple,
                        bool is_single_row_txn)
    # 进一步传递到这里
    - YBCPgNewInsert(dboid, relid, is_single_row_txn, &insert_stmt)  
        - PgApiImpl::NewInsert(..., const bool is_single_row_txn, ...)
            - auto stmt = make_scoped_refptr<PgInsert>(..., is_single_row_txn)
                # PgInsert继承自PgDmlWrite，在PgInsert的构造方法中，会将参数is_single_row_txn
                # 进一步传递给PgDmlWrite
                - PgDmlWrite(std::move(pg_session), table_id, is_single_row_txn)
                    # 设置PgDmlWrite::is_single_row_txn_
                    - is_single_row_txn_ = is_single_row_txn
```

PgDmlWrite::is_single_row_txn_则进一步传递给YBPgsqlWriteOp::is_single_row_txn_:
```
PgDmlWrite::AllocWriteRequest
    - auto wop = AllocWriteOperation()
    # 设置YBPgsqlWriteOp::is_single_row_txn_
    - wop->set_is_single_row_txn(is_single_row_txn_)
        - is_single_row_txn_ = is_single_row_txn
```

is_single_row_txn_ = is_single_row_txn则进一步在YBPgsqlWriteOp::IsTransactional()中被使用：
```
bool YBPgsqlWriteOp::IsTransactional() const {
  # 只有在不是单行事务，且table是transactional的，才会走事务逻辑
  return !is_single_row_txn_ && table_->schema().table_properties().is_transactional();
}
```

YBPgsqlWriteOp::IsTransactional()则进一步会在PgSession::RunAsync中被用于构建PgSession::RunHelper：
```
  template<class Op>
  Result<PgSessionAsyncRunResult> PgSession::RunAsync(const std::shared_ptr<Op>* op,
                       size_t ops_count,
                       const PgObjectId& relation_id,
                       uint64_t* read_time,
                       bool force_non_bufferable) {
    # 构建PgSession::RunHelper，在PgSession::ShouldHandleTransactionally中会用到
    # YBPgsqlWriteOp::IsTransactional()作为判断当前操作是否应该当做分布式事务来
    # 处理的条件之一
    RunHelper runner(this, ShouldHandleTransactionally(**op));
    for (auto end = op + ops_count; op != end; ++op) {
      RETURN_NOT_OK(runner.Apply(*op, relation_id, read_time, force_non_bufferable));
    }
    return runner.Flush();
  }
  
  # PgSession::RunHelper的构造方法，参数transactional被2个地方使用，1个地方是设置
  # PgSession::RunHelper::transactional_，另1个地方是用于设置到底使用PgSession::
  # buffered_ops_来作为buffered_ops_还是使用PgSession::buffered_txn_ops_，如果
  # transactional为true，则使用PgSession::buffered_txn_ops_，否则使用PgSession::
  # buffered_ops_
  PgSession::RunHelper::RunHelper(PgSession* pg_session, bool transactional)
    :  pg_session_(*pg_session),
       transactional_(transactional),
       buffered_ops_(transactional_ ? pg_session_.buffered_txn_ops_
                                    : pg_session_.buffered_ops_) {
    if (!transactional_) {
      pg_session_.InvalidateForeignKeyReferenceCache();
    }
  }
```

那么PgSession::RunHelper::transactional_又在哪里被用到呢，经分析，会在PgSession::RunHelper::Apply中被用到，PgSession::RunHelper::Apply中操作可能会被buffered，也可能不被buffered，我们暂假设当前操作满足被buffered的条件，那么在PgSession::RunHelper::Apply中，操作会被添加到PgSession::RunHelper::buffered_ops_中，从PgSession::RunHelper::RunHelper的构造方法可知，对于分布式事务操作PgSession::RunHelper::buffered_ops_对应的就是PgSession::buffered_txn_ops_，而对于单行事务操作PgSession::RunHelper::buffered_ops_对应的就是PgSession::buffered_ops_：
```
PgSession::RunHelper::Apply
    # 从PgSession::RunHelper::RunHelper的构造方法可知，对于分布式事务操作
    # PgSession::RunHelper::buffered_ops_对应的就是PgSession::buffered_txn_ops_，
    # 而对于单行事务操作PgSession::RunHelper::buffered_ops_对应的就是
    # PgSession::buffered_ops_
    - buffered_ops_.push_back({std::move(op), relation_id})
```

对于被添加到PgSession::buffered_txn_ops_或者PgSession::buffered_ops_中的操作会在PgSession::FlushBufferedOperationsImpl中被flush：
```
exec_simple_query
    - ...
        - ProcessQuery
            - ...
                - ExecutorFinish
                    - YBEndOperationsBuffering
                        - ...
                            - PgSession::FlushBufferedOperationsImpl
                                - auto ops = std::move(buffered_ops_);
                                  auto txn_ops = std::move(buffered_txn_ops_)
                                # 被buffered的单行事务的操作在这里flush
                                - FlushBufferedOperationsImpl(ops, false /* transactional */)
                                # 被buffered的分布式事务的操作在这里flush
                                - FlushBufferedOperationsImpl(txn_ops, true /* transactional */)
```

从PgSession::FlushBufferedOperationsImpl的实现来看，无论是被buffered的单行事务还是分布式事务，都是在PgSession::FlushBufferedOperationsImpl中被flush的，只不过第二个参数有所区别：
```
# 如果参数transactional为true，则表示是分布式事务操作，否则是单行事务操作
PgSession::FlushBufferedOperationsImpl(const PgsqlOpBuffer& ops, bool transactional)
    # PgSession::GetSession会单独拿出来分析
    # 从PgSession::GetSession实现来看，如果是分布式事务，则从PgTxnManager中获取YBSession，
    # 它是当前事务的专属YBSession，如果是单行事务，则直接使用PgSession中的YBSession，它是
    # 所有单行事务通用的YBSession
    - auto session = VERIFY_RESULT(GetSession(transactional, false /* read_only_op */))
    # 逐一处理被buffered的操作
    - for (auto buffered_op : ops) {
        const auto& op = buffered_op.operation;
        # 添加到Batcher等待执行
        session->Apply(op);
            - Status s = Batcher().Add(yb_op)
                # 添加到inflight op列表中
                - AddInFlightOp
                # 查找操作对应的tablet，当接收到查找响应的时候Batcher::TabletLookupFinished
                # 会被调用
                - client_->data_->meta_cache_->LookupTabletByKey(
                    in_flight_op->yb_op->table(), ...,
                    std::bind(&Batcher::TabletLookupFinished, BatcherPtr(this), in_flight_op, _1))
      }
    - const auto status = session->FlushFuture().get()
        # 结束当前的Batcher，并对当前的Batcher执行Batcher::FlushAsync
        - YBSession::FlushAsync
            - 结束当前的Batcher
            - CheckForFinishedFlush
            - FlushBuffersIfReady
    
Result<YBSession*> PgSession::GetSession(bool transactional,
                                         bool read_only_op,
                                         bool needs_pessimistic_locking) {
  if (transactional) {
    # 如果transactional为true，则从PgTxnManager中获取YBSession，PgTxnManager中的YBSession
    # 是在PgTxnManager::BeginTransaction -> PgTxnManager::StartNewSession创建的，所以是
    # 当前事务的专属YBSession
    YBSession* txn_session = VERIFY_RESULT(pg_txn_manager_->GetTransactionalSession());
    # 在PgTxnManager::BeginWriteTransactionIfNecessary中会创建YBTransaction，并且
    # 将YBTransaction关联到YBSession
    RETURN_NOT_OK(pg_txn_manager_->BeginWriteTransactionIfNecessary(read_only_op, needs_pessimistic_locking);
    return txn_session;
  }
  
  # 否则transactional为false，则直接使用PgSession中的YBSession，所以属于所有单行事务
  # 通用的YBSession
  return session_.get();
}    
```

下面看看FlushBuffersIfReady：
```
Batcher::FlushBuffersIfReady
    # 设置Batcher状态
    - state_ = BatcherState::kTransactionPrepare
    - 对所有即将flush到对应的tablet上的操作进行排序(相同tablet的相同operation group的放在一起)
    - ExecuteOperations(Initial::kTrue)
        # 如果是分布式事务，则Batcher::tranBatcher::transaction()才不为null，否则为null
        - auto transaction = this->transaction()
        - 如果是分布式事务，才会执行YBTransaction::Prepare，因为咱们分析的是单行事务，所以不会进入到这里
        - if (transaction) {
            if (!transaction->Prepare(...)) {
              return;
            }
          }
        # 更新Batcher的状态
        - state_ = BatcherState::kTransactionReady
        - 生成rpc，相同tablet且相同operation group的操作放在同一个rpc中，所有生成的rpc都保存在@rpcs中
        # 逐一发送生成的rpc
        - for (const auto& rpc : rpcs) {
            rpc->SendRpc();
          }
```

在上面生成RPC并发送RPC的流程看似对于单行事务和分布式事务来说没什么两样，但是当深入代码，会发现在生成RPC的时候还是有所区别的：
```
Batcher::CreateRpc
    - AsyncRpcData data{this, tablet, allow_local_calls_in_curr_thread, 
        need_consistent_read, hybrid_time_for_write_, std::move(ops)}
    # 对于写操作，生成WriteRpc
    - case OpGroup::kWrite:
        return std::make_shared<WriteRpc>(&data)
            # WriteRpc的构造方法中会调用它的父类AsyncRpcBase的构造方法
            - WriteRpc::WriteRpc
                # 在AsyncRpcBase的构造方法中会调用它的父类AsyncRpc的构造方法
                - AsyncRpcBase(data, YBConsistencyLevel::STRONG)
                    # 参数data类型为AsyncRpcData，它是在Batcher::CreateRpc中生成的
                    - AsyncRpc(data, consistency_level)
                        - batcher_ = data->batcher
                        - ops_ = std::move(data->ops)
                    # 对于分布式事务来说，在Batcher::ExecuteOperations -> YBTransaction::Prepare
                    # 中会设置Batcher::transaction_metadata_，而对于单行事务来说，
                    # Batcher::transaction_metadata_则为null，所以对于分布式事务来说，
                    # 在创建WriteRpc的时候，会设置KeyValueWriteBatchPB::transaction_，
                    # 在TServer处理write操作的时候，会用到这个值
                    - auto& transaction_metadata = batcher_->transaction_metadata()
                    - if (!transaction_metadata.transaction_id.IsNil()) {
                        SetTransactionMetadata(transaction_metadata, &req_);
                            - auto& write_batch = *req->mutable_write_batch()
                            - metadata.ToPB(write_batch.mutable_transaction())
                                # 这里的dest就是KeyValueWriteBatchPB::transaction_
                                - dest->set_transaction_id(transaction_id.data(), transaction_id.size())
                      }
```

综上可知，EState::es_yb_is_single_row_modify_txn的影响如下：
- 如果EState::es_yb_is_single_row_modify_txn为true，则：
    - 会执行单行事务
    - 如果操作被buffered，则操作被添加到PgSession::buffered_ops_
    - 在flush PgSession::buffered_ops_中的操作的时候，会直接从PgSession中获取YBSession，获取到的是所有非分布式事务共用的YBSession
    - 在flush PgSession::buffered_txn_ops_中的操作的时候，不会创建YBTransaction
    - Batcher::transaction_metadata_为null
    - 在将操作生成RPC发送给tablet执行之前，无需执行YBTransaction::Prepare操作(事实上，单行事务没有对应的YBTransaction)
    - 在生成WriteRpc的时候，KeyValueWriteBatchPB::transaction_为null
    - 在整个执行过程中不会涉及到和Status tablet的交互
- 如果EState::es_yb_is_single_row_modify_txn为false，则：
    - 会执行分布式事务
    - 如果操作被buffered，则操作被添加到PgSession::buffered_txn_ops_
    - 在flush PgSession::buffered_txn_ops_中的操作的时候，会从从PgTxnManager中获取YBSession，获取到的是当前分布式事务的专属的YBSession
    - 在flush PgSession::buffered_txn_ops_中的操作的时候，会创建YBTransaction
    - Batcher::transaction_metadata_不为null
    - 在将操作生成RPC发送给tablet执行之前，需要执行YBTransaction::Prepare操作，在YBTransaction::Prepare中会调用YBSession::RequestStatusTablet来获取status tablet的信息
    - 在生成WriteRpc的时候，KeyValueWriteBatchPB::transaction_被设置
    - 在整个执行过程中会频繁和Status tablet的交互
    
