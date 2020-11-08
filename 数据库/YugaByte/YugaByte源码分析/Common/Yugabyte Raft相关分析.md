# 提纲
[toc]

## RaftConsensus和Tablet如何关联起来的
1. 在创建Tablet过程中，会为Tablet的多个副本分别创建TabletPeer结构，在TabletPeer创建过程中会创建RaftConsensus。
```
CatalogManagerBgTasks::Run()
    - CatalogManager::ProcessPendingAssignments
        - CatalogManager::HandleAssignCreatingTablet
            - CatalogManager::SendCreateTabletRequests
                - 向当前Tablet的副本所在的多个Raft peer分别发送AsyncCreateReplica任务，任务运行的主要逻辑是调用AsyncCreateReplica::SendRequest，进一步会调用TabletServerAdminServiceProxy::CreateTabletAsync，该RPC请求会在TServer端被处理，处理逻辑见TabletServerAdminServiceIf::Handle。

TabletServerAdminServiceIf::Handle
    - TabletServiceAdminImpl::CreateTablet
        - TSTabletManager::CreateNewTablet
            - TSTabletManager::OpenTablet
                - TabletPeer::InitTabletPeer
                    # 初始化TabletPeer::consensus_为RaftConsensus
                    - consensus_ = RaftConsensus::Create(...)
                    # 创建Preparer线程
                    - prepare_thread_ = std::make_unique<Preparer>(consensus_.get(), tablet_prepare_pool)
                    # 启动Preparer线程
                    - prepare_thread_->Start()
```

2. 通过在RaftConsensus::Create中可知，RaftConsensus中关联了PeerMessageQueue和PeerManager。
```
shared_ptr<RaftConsensus> RaftConsensus::Create
    # PeerMessageQueue
    - auto queue = std::make_unique<PeerMessageQueue>(...)
    # PeerManager
    - auto peer_manager = std::make_unique<PeerManager>(...)
    # 创建RaftConsensus
    - std::make_shared<RaftConsensus>(..., std::move(queue), std::move(peer_manager), ...)
```

## PeerManager中Peers列表的维护
3. PeerManager中的peers列表的更新
```
RaftConsensus::Start
    - RaftConsensus::ReportFailureDetectedTask
        - RaftConsensus::StartElection
            - RaftConsensus::DoStartElection
                - RaftConsensus::CreateElectionUnlocked
                    - RaftConsensus::ElectionCallback
                        - RaftConsensus::DoElectionCallback
                            - RaftConsensus::BecomeLeaderUnlocked
                                - RaftConsensus::RefreshConsensusQueueAndPeersUnlocked
                                    # 从ReplicaState中获取Raft peer相关的配置信息
                                    - const RaftConfigPB& active_config = state_->GetActiveConfigUnlocked()
                                    # 关闭那些不在active_config中的raft peer相关的连接
                                    - peer_manager_->ClosePeersNotInConfig(active_config)
                                    # 为active_config中的raft peer建立peer结构
                                    - peer_manager_->UpdateRaftConfig(active_config)
                                        # 逐一遍历active_config中的每个Raft peer，如果Raft peer已经存在于PeerManager中，则忽略当前的Raft peer，如果Raft peer就是当前的peer，则忽略当前的raft peer，最后为当前的Raft peer建立一个Peer结构，并保存在PeerManager::peers_中

关于PeerManager::UpdateRaftConfig的代码如下：
void PeerManager::UpdateRaftConfig(const RaftConfigPB& config) {
  VLOG(1) << "Updating peers from new config: " << config.ShortDebugString();

  std::lock_guard<simple_spinlock> lock(lock_);
  // Create new peers.
  for (const RaftPeerPB& peer_pb : config.peers()) {
    # 如果peer_pb对应的raft peer已经在PeerManager::peers_中，则忽略
    if (peers_.find(peer_pb.permanent_uuid()) != peers_.end()) {
      continue;
    }
    
    # 如果peer_pb对应的raft peer就是当前的raft peer，则忽略
    if (peer_pb.permanent_uuid() == local_uuid_) {
      continue;
    }

    # 为peer_pb创建对应的raft peer
    auto remote_peer = Peer::NewRemotePeer(
        peer_pb, tablet_id_, local_uuid_, peer_proxy_factory_->NewProxy(peer_pb), queue_,
        raft_pool_token_, consensus_, peer_proxy_factory_->messenger());

    # 将创建的raft peer添加到PeerManager::peers_中
    peers_[peer_pb.permanent_uuid()] = std::move(*remote_peer);
  }
}                                        
```

通过以上分析可知，raft leader对应的PeerManager::peers_中只包含除leader以外的远端的raft peers，所以在PeerManager::SignalRequest不会在leader自身上执行SignalRequest。

4. 与RaftConsensus::BecomeLeaderUnlocked相对应的是RaftConsensus::BecomeReplicaUnlocked，它用于Leader选举之后，Raft peer成为Follower或者Learner的情况。在RaftConsensus::BecomeReplicaUnlocked中会清除PeerManager::peers_中的所有的raft peer。

## Raft工作流程
下面假设Raft Consensus已经成功启动，且Raft Consensus group中的每个replica（Leader/Follower/Learner）的ReplicaState都是处于Running状态（也就是说Leader可以接收来自应用的请求，Follower和Learner可以接收来自Leader的请求）。

在“YugaByte TServer中写流程分析”中讲到：在Tablet::UpdateQLIndexes中最终会生成OperationDriver并添加到PreparerImpl::queue_中并等待PreparerImpl::Run()方法来处理PreparerImpl::queue_中的OperationDriver。处理逻辑如下：
```
void PreparerImpl::Run() {
  VLOG(2) << "Starting prepare task:" << this;
  for (;;) {
    # 遍历PreparerImpl::queue_中的所有OperationDriver，并对每一个
    # OperationDriver调用ProcessItem
    while (OperationDriver *item = queue_.Pop()) {
      active_tasks_.fetch_sub(1, std::memory_order_release);
      ProcessItem(item);
    }
    
    # 当处理完PreparerImpl::queue_中的所有OperationDriver之后，调用ProcessAndClearLeaderSideBatch
    ProcessAndClearLeaderSideBatch();
    
    ...
    
    return;
  }
}
```

在PreparerImpl::ProcessItem中主要操作就是将给定的OperationDriver添加到PreparerImpl::leader_side_batch_中，如果PreparerImpl::leader_side_batch_中存放的OperationDriver数目不小于FLAGS_max_group_replicate_batch_size，或者当前的OperationDriver具有不同的bound term，或者如果OperationType是kChangeMetadata或者kEmpty，则调用ProcessAndClearLeaderSideBatch进行批量处理当前在PreparerImpl::leader_side_batch_中的所有的OperationDriver，并在处理之后清空PreparerImpl::leader_side_batch_，结合PreparerImpl::Run()可知，当处理完PreparerImpl::queue_中所有的OperationDriver之后，再次调用ProcessAndClearLeaderSideBatch批量处理PreparerImpl::leader_side_batch_中存放的所有的OperationDriver，并在处理之后清空PreparerImpl::leader_side_batch_。

在PreparerImpl::ProcessAndClearLeaderSideBatch中会尝试将PreparerImpl::leader_side_batch_中所有的OperationDriver批量提交给Raft Consensus进行replicate。在遍历PreparerImpl::leader_side_batch_过程中，会对每个OperationDriver执行PrepareAndStart()，如果该方法执行成功则将当前的OperationDriver添加到当前的一批中，否则，结束当前的一批，并处理关于当前OperationDriver的failure，从当前OperationDriver的下一个OperationDriver开始作为新的一批，直到遇到某个OperationDriver在执行PrepareAndStart()的时候失败，或者遍历完毕reparerImpl::leader_side_batch_中所有的OperationDriver。
```
void PreparerImpl::ProcessAndClearLeaderSideBatch() {
  if (leader_side_batch_.empty()) {
    return;
  }

  auto iter = leader_side_batch_.begin();
  auto replication_subbatch_begin = iter;
  auto replication_subbatch_end = iter;

  // PrepareAndStart does not call Consensus::Replicate anymore as of 07/07/2017, and it is our
  // responsibility to do so in case of success. We call Consensus::ReplicateBatch for batches
  // of consecutive successfully prepared operations.

  # 遍历leader_side_batch_中的每个OperationDriver
  while (iter != leader_side_batch_.end()) {
    auto* operation_driver = *iter;

    # 在当前OperationDriver上执行
    Status s = operation_driver->PrepareAndStart();

    if (PREDICT_TRUE(s.ok())) {
      # PrepareAndStart成功，则将当前的OperationDriver添加到当前的一批中
      replication_subbatch_end = ++iter;
    } else {
      # PrepareAndStart失败，则结束当前的一批，并对这一批执行replicate
      ReplicateSubBatch(replication_subbatch_begin, replication_subbatch_end);

      // Handle failure for this operation itself.
      # 处理关于当前OperationDriver的failure
      operation_driver->HandleFailure(s);

      // Now we'll start accumulating a new batch.
      # 下一个OperationDriver开始作为新的一批
      replication_subbatch_begin = replication_subbatch_end = ++iter;
    }
  }

  // Replicate the remaining batch. No-op for an empty batch.
  ReplicateSubBatch(replication_subbatch_begin, replication_subbatch_end);

  # 清空leader_side_batch_
  leader_side_batch_.clear();
}
```

在OperationDriver::PrepareAndStart中，对于leader来说，实际上没做什么工作。

在PreparerImpl::ReplicateSubBatch中，会将这一批OperationDrivers中的每一个OperationDriver所关联的ConsensusRound添加到PreparerImpl::rounds_to_replicate_中，然后调用RaftConsensus::ReplicateBatch，最后清空PreparerImpl::rounds_to_replicate_。

每一个OperationDriver对应的ConsensusRound是在哪里创建的呢？原来是在OperationDriver::Init里面：
```
TabletPeer::Submit
    - TabletPeer::NewLeaderOperationDriver
        - TabletPeer::NewOperationDriver
            - TabletPeer::CreateOperationDriver()
            - OperationDriver::Init
            
Status OperationDriver::Init(std::unique_ptr<Operation>* operation, int64_t term) {
  if (operation) {
    operation_ = std::move(*operation);
  }

  if (term == OpId::kUnknownTerm) {
    ...
  } else {
    if (consensus_) {  // sometimes NULL in tests
      # 创建Operation对应的ReplicateMsg
      consensus::ReplicateMsgPtr replicate_msg = operation_->NewReplicateMsg();
      # 创建一个新的ConsensusRound并添加到OperationDriver.Operation.OperationState中
      mutable_state()->set_consensus_round(
        consensus_->NewRound(std::move(replicate_msg),
                             std::bind(&OperationDriver::ReplicationFinished, this, _1, _2, _3)));
      mutable_state()->consensus_round()->BindToTerm(term);
      mutable_state()->consensus_round()->SetAppendCallback(this);
    }
  }

  auto result = operation_tracker_->Add(this);
  if (!result.ok() && operation) {
    *operation = std::move(operation_);
  }

  return result;
}            
```

## RaftConsensus::ReplicateBatch
在RaftConsensus::ReplicateBatch中，会依次遍历这一批的ConsensusRound（保存在ConsensusRounds中，实际上是一个ConsensusRound数组），将他们添加到PeerMessageQueue中，最后调用PeerManager::SignalRequest来发送给当前Leader的所有的远端的peers。
```
Status RaftConsensus::ReplicateBatch(ConsensusRounds* rounds) {
  RETURN_NOT_OK(ExecuteHook(PRE_REPLICATE));
  {
    ReplicaState::UniqueLock lock;
    
    ...
    
    # 将ConsensusRounds（实际上是ConsensusRound数组）中的所有的
    # ConsensusRound添加到PeerMessageQueue中    
    RETURN_NOT_OK(AppendNewRoundsToQueueUnlocked(*rounds));
  }

  # 借助PeerManager来通知当前Leader的所有的远端的peers
  peer_manager_->SignalRequest(RequestTriggerMode::kNonEmptyOnly);
  RETURN_NOT_OK(ExecuteHook(POST_REPLICATE));
  return Status::OK();
}
```

所以下面的分析将先分析RaftConsensus::AppendNewRoundsToQueueUnlocked，然后分析PeerManager::SignalRequest。

### RaftConsensus::AppendNewRoundsToQueueUnlocked
在RaftConsensus::AppendNewRoundsToQueueUnlocked中，会首先遍历ConsensusRounds中所有的ConsensusRound，对于每一个ConsensusRound执行：为对应的ReplicateMsg设置OpId，并添加last committed id信息，将当前ConsensusRound添添加到ReplicateState的retryable_requests_集合和pending_operations_队列中，将当前ConsensusRound对应的ReplicateMsg添加到replicate_msgs集合中；当遍历完所有的ConsensusRound之后，将replicate_msgs集合添加到PeerMessageQueue中，然后更新ReplicateState中记录的last_received_op_id_。
```
Status RaftConsensus::AppendNewRoundsToQueueUnlocked(
    const std::vector<scoped_refptr<ConsensusRound>>& rounds) {
  # 每一个ConsensusRound对应一个ReplicateMsg
  std::vector<ReplicateMsgPtr> replicate_msgs;
  replicate_msgs.reserve(rounds.size());

  for (auto iter = rounds.begin(); iter != rounds.end(); ++iter) {
    const ConsensusRoundPtr& round = *iter;
    # 设置ReplicateMsg对应的OpId（这里state_类型为ReplicateState，OpId是由它负责分配）
    state_->NewIdUnlocked(round->replicate_msg()->mutable_id());

    ReplicateMsg* const replicate_msg = round->replicate_msg().get();

    // In YB tables we include the last committed id into every REPLICATE log record so we can
    // perform local bootstrap more efficiently.
    # 在ReplicateMsg中设置last committed id信息
    state_->GetCommittedOpIdUnlocked().ToPB(replicate_msg->mutable_committed_op_id());

    ...
    
    # 添加到ReplicateState的retryable_requests_集合和pending_operations_队列中
    Status s = state_->AddPendingOperation(round);

    # 将当前ConsensusRound对应的ReplicateMsg添加到replicate_msgs集合中
    replicate_msgs.push_back(round->replicate_msg());
  }

  if (replicate_msgs.empty()) {
    return Status::OK();
  }

  # 将replicate_msgs集合添加到PeerMessageQueue中
  Status s = queue_->AppendOperations(
      replicate_msgs, state_->GetCommittedOpIdUnlocked(), state_->Clock().Now());

  ...

  # 更新ReplicateState中记录的last_received_op_id_
  state_->UpdateLastReceivedOpIdUnlocked(replicate_msgs.back()->id());
  return Status::OK();
}
```

PeerMessageQueue::AppendOperation用于将ReplicateMsg集合添加到PeerMessageQueue中。
```
Status PeerMessageQueue::AppendOperations(const ReplicateMsgs& msgs,
                                          const yb::OpId& committed_op_id,
                                          RestartSafeCoarseTimePoint batch_mono_time) {
  OpId last_id;
  if (!msgs.empty()) {
    std::unique_lock<simple_spinlock> lock(queue_lock_);

    # 获取这一批消息中最后一个消息的OpId
    last_id = msgs.back()->id();

    if (last_id.term() > queue_state_.current_term) {
      queue_state_.current_term = last_id.term();
    }
  } else {
    std::unique_lock<simple_spinlock> lock(queue_lock_);
    last_id = queue_state_.last_appended;
  }

  # 将ReplicateMsgs添加到log和cache中，当ReplicateMsgs成功写入到log中之后会触发回调：
  # PeerMessageQueue::LocalPeerAppendFinished
  RETURN_NOT_OK(log_cache_.AppendOperations(
      msgs, committed_op_id, batch_mono_time,
      Bind(&PeerMessageQueue::LocalPeerAppendFinished, Unretained(this), last_id)));

  if (!msgs.empty()) {
    std::unique_lock<simple_spinlock> lock(queue_lock_);
    # 更新PeerMessageQueue的QueueState中记录的last_appended
    queue_state_.last_appended = last_id;
    UpdateMetrics();
  }

  return Status::OK();
}
```

在LogCache::AppendOperations中
```
Status LogCache::AppendOperations(const ReplicateMsgs& msgs, const yb::OpId& committed_op_id,
                                  RestartSafeCoarseTimePoint batch_mono_time,
                                  const StatusCallback& callback) {
  PrepareAppendResult prepare_result;
  if (!msgs.empty()) {
    # 做一些必要的检查，然后将ReplicateMsgs中的所有的ReplicateMsg添加到LogCache中
    # 的MessageCache中，MessageCache是一个关于log index -> ReplicateMsg的映射表，
    # 最后会更新next_sequential_op_index_为ReplicateMsgs的最后一个ReplicateMsg
    # 对应的log index
    prepare_result = VERIFY_RESULT(PrepareAppendOperations(msgs));
  }

  # 见“Log::AsyncAppendReplicates”
  Status log_status = log_->AsyncAppendReplicates(
    msgs, committed_op_id, batch_mono_time,
    Bind(&LogCache::LogCallback, Unretained(this), prepare_result.last_idx_in_batch, callback));

  if (!log_status.ok()) {
    LOG_WITH_PREFIX_UNLOCKED(WARNING) << "Couldn't append to log: " << log_status;
    return log_status;
  }

  metrics_.size->IncrementBy(prepare_result.mem_required);
  metrics_.num_ops->IncrementBy(msgs.size());

  return Status::OK();
}

Status Log::AsyncAppendReplicates(const ReplicateMsgs& msgs, const yb::OpId& committed_op_id,
                                  RestartSafeCoarseTimePoint batch_mono_time,
                                  const StatusCallback& callback) {
  # 根据ReplicateMsgs中的所有的ReplicateMsg生成一个LogEntryBatchPB
  auto batch = CreateBatchFromAllocatedOperations(msgs);
  if (committed_op_id) {
    # 设置last commited id
    committed_op_id.ToPB(batch.mutable_committed_op_id());
  }
  
  // Set batch mono time if it was specified.
  if (batch_mono_time != RestartSafeCoarseTimePoint()) {
    batch.set_mono_time(batch_mono_time.ToUInt64());
  }

  # 将LogEntryBatchPB转换为LogEntryBatch
  LogEntryBatch* reserved_entry_batch;
  RETURN_NOT_OK(Reserve(REPLICATE, &batch, &reserved_entry_batch));

  # 设置LogEntryBatch对应的ReplicateMsgs
  reserved_entry_batch->SetReplicates(msgs);

  # 将LogEntryBatch提交给Log::Appender，在Log::Appender中该LogEntryBatch会被异步处理
  RETURN_NOT_OK(AsyncAppend(reserved_entry_batch, callback));
  return Status::OK();
}

Status Log::AsyncAppend(LogEntryBatch* entry_batch, const StatusCallback& callback) {
  {
    SharedLock<rw_spinlock> read_lock(state_lock_.get_lock());
    CHECK_EQ(kLogWriting, log_state_);
  }

  entry_batch->set_callback(callback);
  entry_batch->MarkReady();

  # 将LogEntryBatch提交给Log::Appender，在Log::Appender中该LogEntryBatch会被异步处理
  if (PREDICT_FALSE(!appender_->Submit(entry_batch).ok())) {
    delete entry_batch;
    return kLogShutdownStatus;
  }

  return Status::OK();
}
```

至此，RaftConsensus::AppendNewRoundsToQueueUnlocked就返回了，LogEntryBatch被提交给了Log::Appender，但是在Log::Appender中该LogEntryBatch会被异步处理，结合RaftConsensus::ReplicateBatch的代码可知，接下来就会通过PeerManager来向Leader的所有的Raft peers发送Replicate请求。**从这里可以看出，在Leader向他的所有的Raft peers发送Replicate请求的时候，本地的Log并不一定写入到磁盘中。**

#### 对提交给Log::Appender的LogEntryBatch的处理
在Log::AsyncAppend中会进一步调用Log::Appender::Submit -> TaskStream<T>::Submit -> TaskStreamImpl<T>::Submit -> 添加到TaskStreamImpl<T>的queue（TaskStreamImpl.queue_）中，同时提交一个任务到TaskStreamImpl<T>的taskstream_pool_token_中，最终由TaskStreamImpl::Run进行处理；
```
Log::Appender::Submit(LogEntryBatch* item) {
  # 实际调用的是TaskStreamImpl<T>::Submit
  return task_stream_->Submit(item);
}

template <typename T> Status TaskStreamImpl<T>::Submit(T *task) {
  # 将LogEntryBatch添加到BlockingQueue中
  if (!queue_.BlockingPut(task)) {
    return STATUS_FORMAT(ServiceUnavailable,
                         "TaskStream queue is full (max capacity $0)",
                         queue_.max_size());
  }

  # 如果running为1则表明已经有一个Task在运行，否则表明需要创建一个新的Task
  int expected = 0;
  if (!running_.compare_exchange_strong(expected, 1, std::memory_order_acq_rel)) {
    // running_ was not 0, so we are not creating a task to process operations.
    # running不为0，表明Task已经存在
    return Status::OK();
  }
  
  # 至此，需要创建一个新的Task，通过ThreadPoolToken来提交一个任务到线程池中，
  # 任务的运行主体是TaskStreamImpl::Run
  // We flipped running_ from 0 to 1. The previously running thread could go back to doing another
  // iteration, but in that case since we are submitting to a token of a thread pool, only one
  // such thread will be running, the other will be in the queue.
  return taskstream_pool_token_->SubmitFunc(std::bind(&TaskStreamImpl::Run, this));
}
```

在TaskStreamImpl::Run()中会采用TaskStreamImpl<T>.process_item_来逐一处理TaskStreamImpl.queue_中的每一个LogEntryBatch，当TaskStreamImpl.queue_中的所有的LogEntryBatch都处理完毕之后，会调用TaskStreamImpl<T>.process_item_来处理一个空的LogEntryBatch（目的是为了执行WAL log sync），TaskStreamImpl<T>.process_item_是在Log::Appender::Appender的构造方法中初始化Log::Appender.task_stream_实例时设置的Log::Appender::ProcessBatch。
```
template <typename T> void TaskStreamImpl<T>::Run() {
  VLOG(1) << "Starting taskstream task:" << this;
  for (;;) {
    MonoTime wait_timeout_deadline = MonoTime::Now() + queue_max_wait_;
    std::vector<T *> group;
    # 从队列中将所有待处理的LogEntryBatch取出来存放在@group中
    queue_.BlockingDrainTo(&group, wait_timeout_deadline);
    
    if (!group.empty()) {
      # 逐一遍历@group中的每个LogEntryBatch，并调用ProcessItem处理之，
      # 实际上调用的是Log::Appender::ProcessBatch
      for (T* item : group) {
        ProcessItem(item);
      }
      
      # 最后处理一个空的LogEntryBatch
      ProcessItem(nullptr);
      group.clear();
      continue;
    }
    
    ...
  }
}
```

在Log::Appender::ProcessBatch中，如果当前的LogEntryBatch是空的，则执行Log::Appender::GroupWork()并返回，否则执行Log::Appender::ProcessBatch -> Log::DoAppend写入WAL log。
```
void Log::Appender::ProcessBatch(LogEntryBatch* entry_batch) {
  // A callback function to TaskStream is expected to process the accumulated batch of entries.
  # 通过一个空的LogEntryBatch来确保写入log中的数据被sync到磁盘上
  if (entry_batch == nullptr) {
    // Here, we do sync and call callbacks.
    # 执行Log::Sync()，确保WAL log持久化，然后调用Log.sync_batch_中的每一个LogEntryBatch的回调
    GroupWork();
    return;
  }

  # 将当前的LogEntryBatch写入log，见“Log::DoAppend”
  Status s = log_->DoAppend(entry_batch);
  
  if (!log_->sync_disabled_) {
    bool expected = false;
    if (log_->periodic_sync_needed_.compare_exchange_strong(expected, true,
                                                            std::memory_order_acq_rel)) {
      log_->periodic_sync_earliest_unsync_entry_time_ = MonoTime::Now();
    }
    log_->periodic_sync_unsynced_bytes_ += entry_batch->total_size_bytes();
  }
  
  # 将当前LogEntryBatch添加到Log.sync_batch_中，当执行了Log::Sync()之后，
  # 会依次调用Log.sync_batch_中的每一个LogEntryBatch的回调
  sync_batch_.emplace_back(entry_batch);
}
```

在Log::DoAppend中有3个参数，但是在Log::Appender::ProcessBatch调用的时候只传递了1个参数，是因为有2个默认参数，其中@caller_owns_operation默认为true，@skip_wal_write默认为false，**根据注释说明，如果@skip_wal_write被设置为true，则只会更新consensus metadata和log index，而不会写wal log。**
```
Status Log::DoAppend(LogEntryBatch* entry_batch,
                     bool caller_owns_operation,
                     bool skip_wal_write) {
  if (!skip_wal_write) {
    # 在skip_wal_write为false的情况下，也就是默认情况下，会进入下面的逻辑
    
    # 将LogEntryBatch序列化，序列化后的数据存放在LogEntryBatch::buffer_中
    RETURN_NOT_OK(entry_batch->Serialize());
    
    # 获取LogEntryBatch::buffer_中数据
    Slice entry_batch_data = entry_batch->data();

    # 当前LogEntryBatch所占用的内存空间大小
    uint32_t entry_batch_bytes = entry_batch->total_size_bytes();
    // If there is no data to write return OK.
    if (PREDICT_FALSE(entry_batch_bytes == 0)) {
      return Status::OK();
    }

    // if the size of this entry overflows the current segment, get a new one
    # 下面的判断逻辑目的是：检查当前的LogSegment是否可以容纳当前的LogEntryBatch，
    # 如果无法容纳，则分配一个新的LogSegment，当成功分配新的LogSegment之后，会
    # 调用RollOver切换到该新的LogSegment上，如果当前LogSegment可以容纳，则直接
    # 进入后面的逻辑
    if (allocation_state() == kAllocationNotStarted) {
      if ((active_segment_->Size() + entry_batch_bytes + 4) > cur_max_segment_size_) {
        LOG_WITH_PREFIX(INFO) << "Max segment size " << cur_max_segment_size_ << " reached. "
                              << "Starting new segment allocation. ";
        RETURN_NOT_OK(AsyncAllocateSegment());
        if (!options_.async_preallocate_segments) {
          LOG_SLOW_EXECUTION(WARNING, 50, "Log roll took a long time") {
            RETURN_NOT_OK(RollOver());
          }
        }
      }
    } else if (allocation_state() == kAllocationFinished) {
      LOG_SLOW_EXECUTION(WARNING, 50, "Log roll took a long time") {
        # RollOver用于从当前的LogSegment切换到新分配的LogSegment上，在RollOver中主要
        # 包括以下步骤：
        # 1. 确保当前LogSegment中写入的数据被flush到磁盘上
        # 2. 向当前LogSegment中写入Footer信息，然后关闭当前LogSegment
        # 3. 最后切换到新分配的LogSegment，该新分配的LogSegment变为active_segment
        RETURN_NOT_OK(RollOver());
      }
    } else {
      VLOG_WITH_PREFIX(1) << "Segment allocation already in progress...";
    }

    int64_t start_offset = active_segment_->written_offset();

    # 将当前的LogEntryBatch写入到当前的LogSegment中
    LOG_SLOW_EXECUTION(WARNING, 50, "Append to log took a long time") {
      SCOPED_LATENCY_METRIC(metrics_, append_latency);
      SCOPED_WATCH_STACK(FLAGS_consensus_log_scoped_watch_delay_append_threshold_ms);

      # 将当前的LogEntryBatch写入到当前的LogSegment中
      RETURN_NOT_OK(active_segment_->WriteEntryBatch(entry_batch_data));

      ...
    }

    // Populate the offset and sequence number for the entry batch if we did a WAL write.
    # 在当前LogEntryBatch中记录它所对应的LogSegment的sequence number，及它在该LogSegment
    # 中的偏移
    entry_batch->offset_ = start_offset;
    entry_batch->active_segment_sequence_number_ = active_segment_sequence_number_;
  }

  # 更新LogEntryBatch中每一个LogEntry所对应的LogSegment的sequence number，及它在该LogSegment
  # 中的偏移，LogEntry所对应的LogSegment的sequence number实际上跟LogEntryBatch的相同，LogEntry
  # 在该LogSegment中的偏移实际上跟LogEntryBatch的相同，每一个LogEntry所对应的这两个信息保存
  # 在LogIndexEntry中，最后会将每个LogEntry对应的LogIndexEntry添加到Log::log_index_中，
  # Log::log_index_类型为LogIndex
  CHECK_OK(UpdateIndexForBatch(*entry_batch));
  
  # 根据LogEntryBatch中每一个LogEntry的log index信息来更新LogSegmentFooterPB中的最小和最大
  # log index
  UpdateFooterForBatch(entry_batch);

  ...

  return Status::OK();
}
```



当执行了Log::Sync()之后，会依次调用Log.sync_batch_中的每一个LogEntryBatch的回调，这个回调方法是PeerMessageQueue::LocalPeerAppendFinished -> PeerMessageQueue::ResponseFromPeer。


### PeerManager::SignalRequest



## 答疑
### 1. RaftConsensus::ReplicateBatch中在执行完RaftConsensus::AppendNewRoundsToQueueUnlocked，之后会执行PeerManager::SignalRequest，RaftConsensus::AppendNewRoundsToQueueUnlocked返回的时候，本地WAL已经sync了？

根据前面的分析，执行完RaftConsensus::AppendNewRoundsToQueueUnlocked之后，只是将对应的ReplicateMsgs添加到LogCache中，同时根据ReplicateMsgs生成LogEntryBatch并通过Log::Appender -> TaskStream -> TaskStreamImpl -> taskstream_pool_token_添加到线程池中进行处理。其中taskstream_pool_token_是一个ThreadPoolToken，它对应一个线程池，这个线程池是Log::Appender的构造方法中的append_thread_pool参数对应的线程池，进一步的，这个线程池对应于TSTabletManager中的append_pool_。具体的跟踪过程如下：
```
TSTabletManager::OpenTablet
    # 这里的append_pool()对应于TSTabletManager中的append_pool_
    - tablet::BootstrapTabletData data = {..., append_pool() , ...}
    - BootstrapTablet(data, &tablet, &log, &bootstrap_info)
        - enterprise::TabletBootstrap bootstrap(data)
            - yb::tablet::TabletBootstrap::TabletBootstrap
                # 设置TabletBootstrap.append_pool_为data.append_pool
                - append_pool_(data.append_pool)
                
TabletBootstrap::OpenNewLog
    - Log::Open(..., append_pool_, ...)
        - new Log(..., append_pool_)
            # 设置Log::appender_
            - appender_(new Appender(this, append_thread_pool))
                - TaskStream<LogEntryBatch>(..., append_thread_pool, ...))
                    - TaskStreamImpl(..., thread_pool, ...)
                        - taskstream_pool_token_(thread_pool->NewToken(ThreadPool::ExecutionMode::SERIAL))
                            
RaftConsensus::AppendNewRoundsToQueueUnlocked
    PeerMessageQueue::AppendOperations
        LogCache::AppendOperations
            # 添加到LogCache.MessageCache中
            LogCache::PrepareAppendOperations
            # 异步写入WAL log中
            Log::AsyncAppendReplicates
                Log::AsyncAppend
                    Log::Appender::Submit 
                        TaskStream<T>::Submit
                            TaskStreamImpl<T>::Submit
                                # 将LogEntryBatch提交到TaskStreamImpl的队列中
                                queue_.BlockingPut(task)
                                # 提交一个任务到由taskstream_pool_token_确定的一个线程池中，任务
                                # 的运行逻辑是TaskStreamImpl::Run，它会执行TaskStreamImpl的队列中
                                # 的LogEntryBatch
                                taskstream_pool_token_->SubmitFunc(std::bind(&TaskStreamImpl::Run, this))
```

### 2. 在RaftConsensus::AppendNewRoundsToQueueUnlocked中会将所有的ConsensusRound添加到ReplicaState.PendingOperations队列中，该队列中的ConsensusRound何时被进一步处理？


```
PeerMessageQueue::ResponseFromPeer
    PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
        RaftConsensus::UpdateMajorityReplicated
            ReplicaState::UpdateMajorityReplicatedUnlocked
                ReplicaState::AdvanceCommittedOpIdUnlocked
                    ReplicaState::ApplyPendingOperationsUnlocked    
                        ReplicaState::NotifyReplicationFinishedUnlocked   
                            OperationDriver::ReplicationFinished
```


### 3. 在RaftConsensus::AppendNewRoundsToQueueUnlocked中，如果ConsensusRound对应的ReplicateMsg对应的OperationType如果是WRITE_OP，则会将该ConsensusRound添加到retryable_requests_中，那么retryable_requests_中的请求又是何时被进一步处理的呢？


```
PeerMessageQueue::ResponseFromPeer
    PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
        RaftConsensus::UpdateMajorityReplicated
            ReplicaState::UpdateMajorityReplicatedUnlocked
                ReplicaState::AdvanceCommittedOpIdUnlocked
                    ReplicaState::ApplyPendingOperationsUnlocked    
                        ReplicaState::NotifyReplicationFinishedUnlocked     
                            RetryableRequests::Impl::ReplicationFinished
```

### 4. 本地WAL sync的时机？
本地WAL sync是通过Log::Sync()进行的，它的调用时机主要有如下2个：
```
RaftConsensus::ReplicateBatch
    RaftConsensus::AppendNewRoundsToQueueUnlocked
        PeerMessageQueue::AppendOperations
            LogCache::AppendOperations
                # 添加到LogCache.MessageCache中
                LogCache::PrepareAppendOperations
                # 异步写入WAL log中
                Log::AsyncAppendReplicates
                    Log::AsyncAppend
                        Log::Appender::Submit
                            ...
                                TaskStreamImpl::Run
                                    TaskStreamImpl<T>::ProcessItem
                                        Log::Appender::ProcessBatch
                                            Log::Appender::GroupWork
                                                # 调用时机1
                                                Log::Sync
                                            Log::DoAppend    
                                                # 如果发生了LogSegment切换
                                                Log::RollOver
                                                    # 调用时机2
                                                    Log::Sync  
```

Log::Sync的主要逻辑如下：
(1). 在sync_disabled_为false的情况下，也就是开启wal log sync的情况下，才会真正执行sync操作：
    - 如果durable_wal_write_为false(则表明每次写WAL log都不是采用O_DIRECT方式)， 且periodic_sync_needed_为true，则检查自earliest unsync time以来的时间是否超过了interval_durable_wal_write_的时间，或者unsynced bytes是否超过了bytes_durable_wal_write_mb_，并据此设置是否需要触发sync操作，也就是代码中是否设置timed_or_data_limit_sync为true
    - 如果durable_wal_write_为true或者timed_or_data_limit_sync为true，则对当前的LogSegment执行Sync操作
(2). 更新LogReader可以读取的最新的LogSegment中的偏移
(3). 更新Log中记录的last_synced_entry_op_id_
```
Status Log::Sync() {
  TRACE_EVENT0("log", "Sync");
  SCOPED_LATENCY_METRIC(metrics_, sync_latency);

  if (!sync_disabled_) {
    if (PREDICT_FALSE(GetAtomicFlag(&FLAGS_log_inject_latency))) {
      Random r(GetCurrentTimeMicros());
      int sleep_ms = r.Normal(GetAtomicFlag(&FLAGS_log_inject_latency_ms_mean),
                              GetAtomicFlag(&FLAGS_log_inject_latency_ms_stddev));
      if (sleep_ms > 0) {
        LOG_WITH_PREFIX(INFO) << "Injecting " << sleep_ms << "ms of latency in Log::Sync()";
        SleepFor(MonoDelta::FromMilliseconds(sleep_ms));
      }
    }

    bool timed_or_data_limit_sync = false;
    if (!durable_wal_write_ && periodic_sync_needed_.load()) {
      if (interval_durable_wal_write_) {
        if (MonoTime::Now() > periodic_sync_earliest_unsync_entry_time_
            + interval_durable_wal_write_) {
          timed_or_data_limit_sync = true;
        }
      }
      if (bytes_durable_wal_write_mb_ > 0) {
        if (periodic_sync_unsynced_bytes_ >= bytes_durable_wal_write_mb_ * 1_MB) {
          timed_or_data_limit_sync = true;
        }
      }
    }

    if (durable_wal_write_ || timed_or_data_limit_sync) {
      periodic_sync_needed_.store(false);
      periodic_sync_unsynced_bytes_ = 0;
      LOG_SLOW_EXECUTION(WARNING, 50, "Fsync log took a long time") {
        RETURN_NOT_OK(active_segment_->Sync());
      }
    }
  }

  // Update the reader on how far it can read the active segment.
  reader_->UpdateLastSegmentOffset(active_segment_->written_offset());

  {
    std::lock_guard<std::mutex> write_lock(last_synced_entry_op_id_mutex_);
    last_synced_entry_op_id_.store(last_appended_entry_op_id_, boost::memory_order_release);
    last_synced_entry_op_id_cond_.notify_all();
  }

  return Status::OK();
}
```

### 5. 远端的raft peer接收到复制请求后的处理逻辑。
待续。。。

### 6. Leader端PeerMessageQueue::ResponseFromPeer的处理逻辑。
当leader本地的raft成功写了wal log之后，会调用PeerMessageQueue::LocalPeerAppendFinished -> PeerMessageQueue::ResponseFromPeer；当接收到来自于远程的raft group peer的响应的时候会调用Peer::ProcessResponse -> PeerMessageQueue::ResponseFromPeer，也就是说leader接收到raft log复制的响应之后，一定会进入PeerMessageQueue::ResponseFromPeer。
```
void PeerMessageQueue::ResponseFromPeer(const std::string& peer_uuid,
                                        const ConsensusResponsePB& response,
                                        bool* more_pending) {
  MajorityReplicatedData majority_replicated;
  Mode mode_copy;
  {
    LockGuard scoped_lock(queue_lock_);
    DCHECK_NE(State::kQueueConstructed, queue_state_.state);

    # 找到对应的peer
    TrackedPeer* peer = FindPtrOrNull(peers_map_, peer_uuid);
    
    # 做一些检查
    ...

    const ConsensusStatusPB& status = response.status();

    // Take a snapshot of the current peer status.
    TrackedPeer previous = *peer;

    // Update the peer status based on the response.
    peer->is_new = false;
    peer->last_known_committed_idx = status.last_committed_idx();
    peer->last_successful_communication_time = MonoTime::Now();

    # 下面的逻辑用于设置peer中记录的last_received OpId信息和next_index信息
    bool peer_has_prefix_of_log = IsOpInLog(yb::OpId::FromPB(status.last_received()));
    if (peer_has_prefix_of_log) {
      // If the latest thing in their log is in our log, we are in sync.
      peer->last_received = status.last_received();
      peer->next_index = peer->last_received.index() + 1;
    } else if (!OpIdEquals(status.last_received_current_leader(), MinimumOpId())) {
      // Their log may have diverged from ours, however we are in the process of replicating our ops
      // to them, so continue doing so. Eventually, we will cause the divergent entry in their log
      // to be overwritten.
      peer->last_received = status.last_received_current_leader();
      peer->next_index = peer->last_received.index() + 1;
    } else {
      // The peer is divergent and they have not (successfully) received anything from us yet. Start
      // sending from their last committed index.  This logic differs from the Raft spec slightly
      // because instead of stepping back one-by-one from the end until we no longer have an LMP
      // error, we jump back to the last committed op indicated by the peer with the hope that doing
      // so will result in a faster catch-up process.
      DCHECK_GE(peer->last_known_committed_idx, 0);
      peer->next_index = peer->last_known_committed_idx + 1;
    }

    # response中存在error情况的处理
    ...

    peer->is_last_exchange_successful = true;
    peer->num_sst_files = response.num_sst_files();

    # 关于response中的term信息的检查
    if (response.has_responder_term()) {
      // The peer must have responded with a term that is greater than or equal to the last known
      // term for that peer.
      peer->CheckMonotonicTerms(response.responder_term());

      // If the responder didn't send an error back that must mean that it has a term that is the
      // same or lower than ours.
      CHECK_LE(response.responder_term(), queue_state_.current_term);
    }

    // If our log has the next request for the peer or if the peer's committed index is lower than
    // our own, set 'more_pending' to true.
    *more_pending = log_cache_.HasOpBeenWritten(peer->next_index) ||
        (peer->last_known_committed_idx < queue_state_.committed_index.index());

    mode_copy = queue_state_.mode;
    if (mode_copy == Mode::LEADER) {
      # OpIdWatermark() -> GetWatermark<Policy>()，其中会获取每个peer的last_received中
      # 记录的OpId（根据注释，last_received中记录的是已经发送给该peer且已经ack的最后
      # 一个OpId），并存放在一个叫做Watermarks的数组中，然后从Watermarks数组中找出
      # raft group中满足majority(PeerMessageQueue::QueueState.majority_size_)要求的
      # last_received信息，举例说明如下：
      # 假设Raft group中有5个peer，Leader接收到了包括自身在内的4个Raft peer的响应，
      # Watermarks数组中存放的这4个last_received OpId信息为{10, 7, 9, 8, 6}，则会找出
      # 已经满足majority要求的OpId为8。
      auto new_majority_replicated_opid = OpIdWatermark();
      
      # 更新PeerMessageQueue::QueueState中记录的majority_replicated_opid信息
      if (!OpIdEquals(new_majority_replicated_opid, MinimumOpId())) {
        if (new_majority_replicated_opid.index() == MaximumOpId().index()) {
          queue_state_.majority_replicated_opid = local_peer_->last_received;
        } else {
          queue_state_.majority_replicated_opid = new_majority_replicated_opid;
        }
      }
      
      # 设置majority_replicated信息
      majority_replicated.op_id = queue_state_.majority_replicated_opid;
      peer->last_leader_lease_expiration_received_by_follower =
          peer->last_leader_lease_expiration_sent_to_follower;
      peer->last_ht_lease_expiration_received_by_follower =
          peer->last_ht_lease_expiration_sent_to_follower;
      majority_replicated.leader_lease_expiration = LeaderLeaseExpirationWatermark();
      majority_replicated.ht_lease_expiration = HybridTimeLeaseExpirationWatermark();
      
      # NumSSTFilesWatermark()跟OpIdWatermark()的计算逻辑是一样的，但是这里处理的
      # 是peer中记录的num_sst_files，这个用处何在？
      majority_replicated.num_sst_files = NumSSTFilesWatermark();
    }

    # 更新所有raft peer中最小的last_received到PeerMessageQueue::QueueState.all_replicated_opid中
    UpdateAllReplicatedOpId(&queue_state_.all_replicated_opid);

    # 检查LogCache中哪些OpId对应的CacheEntry可以被删除，并删除之
    auto evict_op = std::min(
        queue_state_.all_replicated_opid.index(), GetCDCConsumerOpIdToEvict().index);
    log_cache_.EvictThroughOp(evict_op);

    UpdateMetrics();
  }

  if (mode_copy == Mode::LEADER) {
    NotifyObserversOfMajorityReplOpChange(majority_replicated);
  }
}
```
从上面的代码来看，在Leader执行PeerMessageQueue::ResponseFromPeer的时候，UpdateAllReplicatedOpId和NotifyObserversOfMajorityReplOpChange一定会执行，且每当执行PeerMessageQueue::ResponseFromPeer的时候，都会计算哪些OpId已经是majority replicated（见OpIdWatermark()），哪些则是all replicated（见UpdateAllReplicatedOpId(...)）。

在PeerMessageQueue::NotifyObserversOfMajorityReplOpChange中，会通过raft_pool_observers_token_向对应的线程池（TSTabletManager::raft_pool_）中提交一个任务，任务的运行主体是PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask。

其中raft_pool_observers_token_对应的线程池是TSTabletManager::raft_pool_，其确认流程如下：
```
TSTabletManager::OpenTablet
    # 使用的是TSTabletManager::raft_pool_
    TabletPeer::InitTabletPeer(..., raft_pool(), ...)
        RaftConsensus::Create(..., raft_pool, ...)
            PeerMessageQueue::PeerMessageQueue(..., raft_pool->NewToken(ThreadPool::ExecutionMode::SERIAL))
                # 初始化PeerMessageQueue::raft_pool_observers_token_
                raft_pool_observers_token_(std::move(raft_pool_token))
```

PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask中，会遍历PeerMessageQueue::observers_中的每一个PeerMessageQueueObserver，并调用PeerMessageQueueObserver::UpdateMajorityReplicated，根据传递进来的MajorityReplicatedData信息进行更新，并设置最新committed的OpId @new_committed_index，最后更新PeerMessageQueue的QueueState中记录的最新已被commit的OpId信息。
```
void PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask(
    const MajorityReplicatedData& majority_replicated_data) {
  std::vector<PeerMessageQueueObserver*> copy;
  {
    LockGuard lock(queue_lock_);
    # observers_中的PeerMessageQueueObserver是通过PeerMessageQueue::RegisterObserver注册的，
    # 目前在代码中只看到RaftConsensus::BecomeLeaderUnlocked()会调用RegisterObserver，
    # 且注册的Observer是Leader所在的RaftConsensus
    copy = observers_;
  }

  // TODO move commit index advancement here so that the queue is not dependent on consensus at all,
  // but that requires a bit more work.
  # 依次遍历每一个PeerMessageQueueObserver，并根据传递进来的
  # MajorityReplicatedData信息进行更新，返回最新committed的OpId
  OpId new_committed_index;
  for (PeerMessageQueueObserver* observer : copy) {
    observer->UpdateMajorityReplicated(majority_replicated_data, &new_committed_index);
  }

  # 设置最新committed的OpId
  {
    LockGuard lock(queue_lock_);
    if (new_committed_index.IsInitialized() &&
        new_committed_index.index() > queue_state_.committed_index.index()) {
      queue_state_.committed_index.CopyFrom(new_committed_index);
    }
  }
}
```

因为PeerMessageQueue::observers_只包含Leader对应的RaftConsensus，所以PeerMessageQueueObserver::UpdateMajorityReplicated实际上调用的是RaftConsensus::UpdateMajorityReplicated。RaftConsensus::UpdateMajorityReplicated的主要逻辑如下：
```
RaftConsensus::UpdateMajorityReplicated
    # ReplicaState::UpdateMajorityReplicatedUnlocked
    - state_->UpdateMajorityReplicatedUnlocked
        - 
    - state_->context()->MajorityReplicated()
    # 
    - if (committed_index_changed &&
        state_->GetActiveRoleUnlocked() == RaftPeerPB::LEADER) {
        if (yb::OpId::FromPB(*committed_op_id) == state_->GetLastReceivedOpIdUnlocked()) {
          queue_->AppendOperations(
          {}, yb::OpId::FromPB(*committed_op_id), state_->Clock().Now());
        }
        
        peer_manager_->SignalRequest(RequestTriggerMode::kNonEmptyOnly);
      }
    
Status ReplicaState::UpdateMajorityReplicatedUnlocked(const OpId& majority_replicated,
                                                      OpId* committed_op_id,
                                                      bool* committed_op_id_changed) {
    # 如果之前记录的last_committed_op_id_对应的term就是当前的term，则采用
    # majority_replicated中的OpId更新last_committed_op_id_，见
    # ReplicaState::AdvanceCommittedOpIdUnlocked
    - if (last_committed_op_id_.term == GetCurrentTermUnlocked()) {
        *committed_op_id_changed = AdvanceCommittedOpIdUnlocked(
        yb::OpId::FromPB(majority_replicated), CouldStop::kFalse);
        last_committed_op_id_.ToPB(committed_op_id);
        return Status::OK();
    }  
    
    - if (majority_replicated.term() == GetCurrentTermUnlocked()) {
        auto previous = last_committed_op_id_;
        *committed_op_id_changed = VERIFY_RESULT(AdvanceCommittedOpIdUnlocked(
            yb::OpId::FromPB(majority_replicated), CouldStop::kFalse));
        last_committed_op_id_.ToPB(committed_op_id);
        LOG_WITH_PREFIX(INFO)
            << "Advanced the committed_op_id across terms."
            << " Last committed operation was: " << previous
            << " New committed index is: " << last_committed_op_id_;
        return Status::OK();
      }   
      
     - last_committed_op_id_.ToPB(committed_op_id);
}

# @committed_op_id表示本次将要commit的OpId
ReplicaState::AdvanceCommittedOpIdUnlocked(
    const yb::OpId& committed_op_id, CouldStop could_stop)
    # 如果本次将要commit的OpId已经被commit，则直接返回false，表示没有更新last committed信息
    -   if (last_committed_op_id_.index >= committed_op_id.index) {
            return false;
        }
    # 如果pending_operations_为空，则表明没有需要commit的OpId
    -   if (pending_operations_.empty()) {
            return false;
        }
    # pending_operations_中的第一个OpId对应的index一定比last_committed_op_id_.index大1
    - CHECK_EQ(pending_operations_.front()->id().index(), last_committed_op_id_.index + 1)
    # 保留之前记录的last_committed_op_id_信息
    - auto old_index = last_committed_op_id_.index
    # 应用ReplicaState中的pending operations，直到last_committed_op_id_对应的operation为止，
    # ReplicaState中的pending operations是在RaftConsensus::AppendNewRoundsToQueueUnlocked
    # 中添加的
    #
    # 具体的实现见ReplicaState::ApplyPendingOperationsUnlocked
    - ApplyPendingOperationsUnlocked(committed_op_id, could_stop)
    # 返回last_committed_op_id_信息是否发生变化
    - return last_committed_op_id_.index != old_index;
    
Status ReplicaState::ApplyPendingOperationsUnlocked(
    const yb::OpId& committed_op_id, CouldStop could_stop) {
  auto prev_id = last_committed_op_id_;
  yb::OpId max_allowed_op_id;
  auto leader_term = GetLeaderStateUnlocked().term;

  OpIds applied_op_ids;
  applied_op_ids.reserve(committed_op_id.index - prev_id.index);

  # 逐一遍历pending_operations_中的每一个pending operation（类型为ConsensusRound），
  while (!pending_operations_.empty()) {
    auto round = pending_operations_.front();
    auto current_id = yb::OpId::FromPB(round->id());
    
    # apply pending operations，直到committed_op_id对应的operation
    if (current_id.index > committed_op_id.index) {
      break;
    }

    # 检查当前operation对应的OpId @current_id的term大于prev_id对应的term，
    # 且@current_id的index比prev_id的index大1
    if (PREDICT_TRUE(prev_id)) {
      CHECK_OK(CheckOpInSequence(prev_id, current_id));
    }

    # 从pending_operations_中移除当前的operation
    pending_operations_.pop_front();
    
    // Set committed configuration.
    # 对类型为CHANGE_CONFIG_OP的operation的处理
    if (PREDICT_FALSE(type == OperationType::CHANGE_CONFIG_OP)) {
      ApplyConfigChangeUnlocked(round);
    }

    # 更新prev_id
    prev_id = current_id;
    
    # 见ReplicaState::NotifyReplicationFinishedUnlocked
    NotifyReplicationFinishedUnlocked(round, Status::OK(), leader_term, &applied_op_ids);
  }

  SetLastCommittedIndexUnlocked(prev_id);

  applied_ops_tracker_(applied_op_ids);

  return Status::OK();
}    

ReplicaState::NotifyReplicationFinishedUnlocked
    # ConsensusRound::NotifyReplicationFinished，调用ConsensusRound的
    # ConsensusReplicatedCallback，在ConsensusRound的构造方法中，该callback被设置为
    # OperationDriver::ReplicationFinished
    - round->NotifyReplicationFinished(status, leader_term, applied_op_ids)
    # 见RetryableRequests::Impl::ReplicationFinished
    - retryable_requests_.ReplicationFinished(*round->replicate_msg(), status, leader_term)
```


### 7. majority replicated的判断逻辑？
在Leader执行PeerMessageQueue::ResponseFromPeer的时候，会计算哪些OpId已经是majority replicated（见OpIdWatermark()）。

### 8. 写入到db中的数据何时可以被读取？
待续。。。


1. 本地rpc从发出到响应的整个过程，包括线程情况等。
2. Operation::Replicated里面CompleteWithStatus，是如何响应RPC的。
3. OperationTracker




