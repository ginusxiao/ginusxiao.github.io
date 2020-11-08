# 提纲
[toc]

## TServer中处理write RPC请求的入口
CQLServer通过RPC将写请求发送给TServer进行处理，当TServer接收到该请求之后的处理入口为TabletServiceImpl::Write:
```
void TabletServiceImpl::Write(const WriteRequestPB* req,
                              WriteResponsePB* resp,
                              rpc::RpcContext context) {
  # 如果FLAGS_tserver_noop_read_write为true，则直接返回
  if (FLAGS_tserver_noop_read_write) {
    for (int i = 0; i < req->ql_write_batch_size(); ++i) {
      resp->add_ql_response_batch();
    }
    context.RespondSuccess();
    return;
  }
  
  # 更新混合时间戳，用于维护“happen-before”关系
  UpdateClock(*req, server_->Clock());

  # 查找Tablet leader，如果leader不存在，则返回
  auto tablet = LookupLeaderTabletOrRespond(
      server_->tablet_peer_lookup(), req->tablet_id(), resp, &context);
  if (!tablet ||
      !CheckWriteThrottlingOrRespond(
          req->rejection_score(), tablet.peer.get(), resp, &context)) {
    return;
  }

  ...

  auto operation_state = std::make_unique<WriteOperationState>(tablet.peer->tablet(), req, resp);
  auto context_ptr = std::make_shared<RpcContext>(std::move(context));
  if (RandomActWithProbability(GetAtomicFlag(&FLAGS_respond_write_failed_probability))) {
    # 为测试用
    ...
  } else {
    # 操作完成的时候的回调
    operation_state->set_completion_callback(
        std::make_unique<WriteOperationCompletionCallback>(
            tablet.peer, context_ptr, resp, operation_state.get(), server_->Clock(),
            req->include_trace()));
  }
  
  # TabletPeer::WriteAsync
  tablet.peer->WriteAsync(
      std::move(operation_state), tablet.leader_term, context_ptr->GetClientDeadline());
}
```

TabletPeer::WriteAsync的调用栈如下：
```
TabletPeer::WriteAsync
    - Tablet::AcquireLocksAndPerformDocOperations
        - Tablet::KeyValueBatchFromQLWriteBatch
```

Tablet::KeyValueBatchFromQLWriteBatch的主要逻辑如下：
```
void Tablet::KeyValueBatchFromQLWriteBatch(std::unique_ptr<WriteOperation> operation) {
  # 从WriteOperation中获取docdb::DocOperations，这是一个关于docdb::DocOperation的数组，将在后面被填充
  docdb::DocOperations& doc_ops = operation->doc_ops();
  WriteRequestPB batch_request;
  SetupKeyValueBatch(operation->request(), &batch_request);
  auto* ql_write_batch = batch_request.mutable_ql_write_batch();

  # 设置doc_ops中docdb::DocOperation的数目
  doc_ops.reserve(ql_write_batch->size());

  # 创建TransactionOperationContextOpt，如果操作的table设置了是事务性的，则创建，否则不创建
  Result<TransactionOperationContextOpt> txn_op_ctx =
      CreateTransactionOperationContext(operation->request()->write_batch().transaction());
      
  # 从ql_write_batch中获取每个写请求对应的protobuf @req，同时为每个写请求创建一个响应对应的
  # protobuf @resp，然后为每个写请求创建一个QLWriteOperation，并采用该@req和@resp初始化该
  # QLWriteOperation，然后将这个QLWriteOperation添加到doc_ops中
  for (size_t i = 0; i < ql_write_batch->size(); i++) {
    QLWriteRequestPB* req = ql_write_batch->Mutable(i);
    QLResponsePB* resp = operation->response()->add_ql_response_batch();
    if (metadata_->schema_version() != req->schema_version()) {
      resp->set_status(QLResponsePB::YQL_STATUS_SCHEMA_VERSION_MISMATCH);
    } else {
      auto write_op = std::make_unique<QLWriteOperation>(
          metadata_->schema(), metadata_->index_map(), unique_index_key_schema_.get_ptr(),
          *txn_op_ctx);
      auto status = write_op->Init(req, resp);
      if (!status.ok()) {
        WriteOperation::StartSynchronization(std::move(operation), status);
        return;
      }
      
      # 在doc_ops，也就是operation->doc_ops()中存放的都是类型为QLWriteOperation的元素，
      # QLWriteOperation的类继承关系为：QLWriteOperation -> DocOperationBase -> DocOperation
      doc_ops.emplace_back(std::move(write_op));
    }
  }

  # 经过调试发现，这里并没有写Rocksdb，主要是将WriteOperation中的所有涉及的DocOperation
  # 转换成一个个小的面向Rocksdb的key-value对，并存放在operation->request()->
  # mutable_write_batch()中
  auto status = StartDocWriteOperation(operation.get());
  
  # 调试发现，这里并没有走到
  if (operation->restart_read_ht().is_valid()) {
    WriteOperation::StartSynchronization(std::move(operation), Status::OK());
    return;
  }

  if (status.ok()) {
    UpdateQLIndexes(std::move(operation));
  } else {
    CompleteQLWriteBatch(std::move(operation), status);
  }
}
```

## Tablet::StartDocWriteOperation
Tablet::StartDocWriteOperation中针对开启了事务的table的逻辑会有些不同，下面的分析是基于没有开启事务的table进行，且写操作中不涉及读，也不涉及If语句等，也就是说只分析最基础的写过程，且这个写操作是Insert操作。
```
Tablet::StartDocWriteOperation
    # 确定需要lock的keys，是否需要read snapshot，然后在所有需要lock的keys上加锁
    - docdb::PrepareDocWriteOperation
    - docdb::ExecuteDocWriteOperation
        # 创建一个DocWriteBatch
        - DocWriteBatch doc_write_batch(doc_db, init_marker_behavior, monotonic_counter);
        # 遍历每一个DocOperation，并在每一个DocOperation上执行Apply，在当前上下文中，
        # 实际上调用的是QLWriteOperation::Apply，执行的结果就是将每一个列的数据转换为
        # 一个sub-document或者一系列sub-documents，并将这些sub-document或者sub-documents
        # 添加到前面创建的doc_write_batch中
        - QLWriteOperation::Apply
            - 对于写请求中涉及的每个Column Value，执行ApplyForRegularColumns
                - ApplyForRegularColumns
                    - DocWriteBatch::InsertSubDocument
                        - DocWriteBatch::ExtendSubDocument
                            - DocWriteBatch::SetPrimitive
                                - DocWriteBatch::SetPrimitiveInternal
            # ？
            - 如果需要更新索引，则调用UpdateIndexes
                - UpdateIndexes
        # 将doc_write_batch中的相关信息添加到write_batch中
        - doc_write_batch.MoveToWriteBatchPB(write_batch)
    - 
```

## Tablet::UpdateQLIndexes
1. 在最外层的for循环中，对于if (write_op->index_requests()->empty())这个条件语句，判断都是返回true，所以最外层的for循环不会执行任何操作；

2. 调用Tablet::CompleteQLWriteBatch；
- 在Tablet::CompleteQLWriteBatch中的for循环中的if-else-语句不会得到执行；
- 直接进入WriteOperation::StartSynchronization() -> WriteOperation::DoStartSynchronization -> context_->Submit，这里的context_类型为WriteOperationContext，追溯可以发现WriteOperation是在TabletPeer::WriteAsync中构建的，对应的WriteOperationContext其实就是TabletPeer。所以最终调用的是TabletPeer::Submit；
- 在TabletPeer::Submit中会创建并初始化一个OperationDriver，最后执行OperationDriver::ExecuteAsync -> Preparer::Submit -> PreparerImpl::Submit -> 将OperationDriver添加到PreparerImpl::queue_中，并提交一个任务给PreparerImpl::tablet_prepare_pool_token_，该任务的执行逻辑是PreparerImpl::Run，这可能是OperationDriver::ExecuteAsync中Async的意义所在？
```
    - 在TabletPeer::Submit中通过调用TabletPeer::NewLeaderOperationDriver -> TabletPeer::NewOperationDriver，进一步的调用，调用OperationDriver的构造方法，其中会设置replication_state_为NOT_REPLICATING，prepare_state_为NOT_PREPARED
        - TabletPeer::CreateOperationDriver()
        - operation_driver->Init(operation, term)
```
- 在PreparerImpl::Run()中，会逐一处理PreparerImpl::queue_中的每一个OperationDriver，处理方法见PreparerImpl::ProcessItem()，当处理完PreparerImpl::queue_中的所有的OperationDriver之后，会调用PreparerImpl::ProcessAndClearLeaderSideBatch()；
    - 插播一下PreparerImpl::ProcessAndClearLeaderSideBatch()的调用栈：
    ```
    PreparerImpl::ProcessAndClearLeaderSideBatch
        - PreparerImpl::ReplicateSubBatch
            - RaftConsensus::ReplicateBatch
                - PeerManager::SignalRequest
                    - Peer::SignalRequest    
                        - Peer::SendNextRequest中会指定一个关于接收到响应的回调Peer::ProcessResponse
                            - Peer::ProcessResponse
                                - PeerMessageQueue::ResponseFromPeer
    ```
- 在PreparerImpl::ProcessItem()中会根据OperationDriver::replication_state_来区分是在leader上执行还是follower上执行：
    - 如果是NOT_REPLICATING状态，则就表明是leader，主要操作就是将给定的OperationDriver添加到PreparerImpl::leader_side_batch_中；
    - 否则表明是follower，暂略；
- 在PreparerImpl::ProcessAndClearLeaderSideBatch()中，首先会逐一处理PreparerImpl::leader_side_batch_中的每一个OperationDriver，对每一个OperationDriver调用OperationDriver::PrepareAndStart()，然后会调用PreparerImpl::ReplicateSubBatch()，最后会清空PreparerImpl::leader_side_batch_。
- 在OperationDriver::PrepareAndStart()中，首先执行WriteOperation::Prepare()，然后将replication_state_设置为REPLICATING；
    - 在WriteOperation::Prepare()中，什么都没做就返回了；
- 在PreparerImpl::ReplicateSubBatch()中，会将每一个OperationDriver对应的ConsensusRound添加到PreparerImpl::rounds_to_replicate_中，然后执行RaftConsensus::ReplicateBatch()，并将PreparerImpl::rounds_to_replicate_作为参数；
- 在RaftConsensus::ReplicateBatch()中，会进一步调用RaftConsensus::AppendNewRoundsToQueueUnlocked()，且仍以PreparerImpl::rounds_to_replicate_作为参数；然后调用PeerManager::SignalRequest来通知Raft group peers；

2.1 这里首先分析RaftConsensus::AppendNewRoundsToQueueUnlocked()，主要的作用就是在本地写raft wal log：
- 在RaftConsensus::AppendNewRoundsToQueueUnlocked()中，会首先针对每一个ConsensusRound分别执行：将当前ConsensusRound添加到RaftConsensus.state_(类型为ReplicateState)等待committed的队列中（ReplicaState::pending_operations_），并将每个ConsensusRound对应的replicate_msg添加到replicate_msgs集合中；然后借助PeerMessageQueue::AppendOperations()方法将replicate_msgs集合添加到PeerMessageQueue中；
- 在PeerMessageQueue::AppendOperations()中会进一步调用LogCache::AppendOperations()，并传递了一个回调PeerMessageQueue::LocalPeerAppendFinished()；
- 在LogCache::AppendOperations()中会进一步调用Log::AsyncAppendReplicates()，并进一步将LogCache::AppendOperations()中的回调PeerMessageQueue::LocalPeerAppendFinished()传递进来；
- 在Log::AsyncAppendReplicates中会将传递进来的ReplicateMsgs转换为LogEntryBatch，同时将回调方法设置到LogEntryBatch中，并进一步调用Log::AsyncAppend()；
- 在Log::AsyncAppend中会进一步调用Log::Appender::Submit -> TaskStream<T>::Submit -> TaskStreamImpl<T>::Submit -> 添加到TaskStreamImpl<T>的queue（TaskStreamImpl.queue_）中，同时提交一个任务到TaskStreamImpl<T>的taskstream_pool_token_中，最终由TaskStreamImpl::Run进行处理；
- 在TaskStreamImpl::Run()中会采用TaskStreamImpl<T>.process_item_来逐一处理TaskStreamImpl.queue_中的每一个LogEntryBatch，当TaskStreamImpl.queue_中的所有的LogEntryBatch都处理完毕之后，会调用TaskStreamImpl<T>.process_item_来处理一个空的LogEntryBatch（目的是为了执行WAL log sync），TaskStreamImpl<T>.process_item_是在Log::Appender::Appender的构造方法中初始化Log::Appender.task_stream_实例时设置的Log::Appender::ProcessBatch。
    - 如果当前的LogEntryBatch是空的，则执行Log::Appender::GroupWork()并返回，否则执行Log::Appender::ProcessBatch -> Log::DoAppend写入WAL log；
        - 在Log::Appender::GroupWork()中会首先执行Log::Sync()，确保WAL log持久化，然后调用Log.sync_batch_中的每一个LogEntryBatch的回调；
    - 将当前的LogEntryBatch添加到Log.sync_batch_中，当WAL log sync了之后，会调用LogEntryBatch的回调；
- 当执行了Log::Sync()之后，会依次调用Log.sync_batch_中的每一个回调，这个回调方法是PeerMessageQueue::LocalPeerAppendFinished -> PeerMessageQueue::ResponseFromPeer。

2.2 在PeerManager::SignalRequest -> 对PeerManager中的每一个Consensus peer调用Peer::SignalRequest -> 向Peer::raft_pool_token_(类型为ThreadPoolToken)提交一个任务，任务的运行主体是Peer::SendNextRequest，当该任务被调度执行后Peer::SendNextRequest主要执行以下逻辑：
```
# 组装一个发往给定peer的request，存放在request_中（protobuf形式）和msgs_holder中
    #（以ReplicateMsgs结构的形成），这些消息都是从LogCache中读取的
    - PeerMessageQueue::RequestForPeer(..., &request_, &msgs_holder, ...)
    # 发送要复制的消息给raft group peer，当收到来自于raft group peer的响应的时候，
    # 设置的回调Peer::ProcessResponse会被调用
    - proxy_->UpdateAsync(&request_, trigger_mode, &response_, &controller_,
      std::bind(&Peer::ProcessResponse, retain_self))
        # 追溯发现这里的proxy_是由RaftConsensus::Create中使用到的RpcPeerProxyFactory
        # 调用NewProxy创建的，最终创建的是RpcPeerProxy，但是在RpcPeerProxy里面封装了
        # ConsensusServiceProxy
        - RpcPeerProxy::UpdateAsync
            # 这里的consensus_proxy_就是ConsensusServiceProxy
            - consensus_proxy_->UpdateConsensusAsync
                - ConsensusServiceProxy::UpdateConsensusAsync
                    - proxy_->AsyncRequest
                        - Proxy::DoAsyncRequest
```

总之在PeerManager::SignalRequest中会将要复制的消息发送给raft group peer，当收到来自于raft group peer的响应的时候，设置的回调Peer::ProcessResponse会被调用。

2.3 通过2.1和2.2的分析可知，当leader本地的raft成功写了wal log之后，会调用PeerMessageQueue::LocalPeerAppendFinished -> PeerMessageQueue::ResponseFromPeer；当接收到来自于远程的raft group peer的响应的时候会调用Peer::ProcessResponse -> PeerMessageQueue::ResponseFromPeer，也就是说leader接收到raft log复制的响应之后，一定会进入PeerMessageQueue::ResponseFromPeer。
```
PeerMessageQueue::ResponseFromPeer
    - 根据接收到的响应（ConsensusResponsePB）和本地关于Consensus的状态来设置majority_replicated信息（类型为MajorityReplicatedData）
    # 更新已经在raft group 中的每个peer中成功更新的OpId（PeerMessageQueue.queue_state_.all_replicated_opid）
    - PeerMessageQueue::UpdateAllReplicatedOpId；
    - PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
        - 向PeerMessageQueue.raft_pool_observers_token_(类型为ThreadPoolToken)中提交一个任务，任务的执行主体是PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask，它会通知PeerMessageQueue所关联的所有PeerMessageQueueObservers(PeerMessageQueue::observers_，这些PeerMessageQueueObserver是通过PeerMessageQueue::RegisterObserver注册的，见RaftConsensus::DoElectionCallback -> RaftConsensus::BecomeLeaderUnlocked -> PeerMessageQueue::RegisterObserver，从当前代码来看，只注册了Leader自己？注册的PeerMessageQueueObserver的真实类型是RaftConsensus)关于majority_replicated的信息
            # PeerMessageQueueObserver的真实类型是RaftConsensus
            - PeerMessageQueueObserver::UpdateMajorityReplicated
                - RaftConsensus::UpdateMajorityReplicated
                    # 加锁
                    - ReplicaState::LockForMajorityReplicatedIndexUpdate
                    # 标记
                    - ReplicaState::UpdateMajorityReplicatedUnlocked
                        - AdvanceCommittedOpIdUnlocked
                            - ReplicaState::ApplyPendingOperationsUnlocked
                                # 遍历ReplicaState::pending_operations_中所有的ConsensusRound，并对所有的ConsensusRound调用ReplicaState::NotifyReplicationFinishedUnlocked
                                - ReplicaState::NotifyReplicationFinishedUnlocked
                                    - ConsensusRound::NotifyReplicationFinished
                                        # 经过追溯，在OperationDriver::Init创建ConsensusRound时候
                                        # 会指定为OperationDriver::ReplicationFinished
                                        - replicated_cb_ (实际上是OperationDriver::ReplicationFinished)
                                            - OperationDriver::ReplicationFinished
                                                - 修改OperationDriver::replication_state_为REPLICATED；
                                                - OperationDriver::ApplyOperation
                                                    - OperationDriver::ApplyTask
                                                        - Operation::Replicated
                                                            - WriteOperation::DoReplicated
                                                                - Tablet::ApplyRowOperations
                                                                - WriteOperationState::Commit()
                                                            - OperationState::CompleteWithStatus
                                                                - WriteOperationCompletionCallback::CompleteWithStatus
                                    - RetryableRequests::ReplicationFinished
                                # 最后更新ReplicaState::last_committed_op_id_
                                - ReplicaState::SetLastCommittedIndexUnlocked
```



        

## 一些小知识点
docdb实际上是一个逻辑的概念，它的数据结构定义在docdb::DocDB中，主要包含一个regular Rocksdb实例和一个intents Rocksdb实例。

