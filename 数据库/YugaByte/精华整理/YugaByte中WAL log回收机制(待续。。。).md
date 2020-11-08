# 提纲
[toc]

## MaintenanceManager
### TabletPeer向MaintenanceManager注册类型为LogGCOp的Maintenance操作
当创建或者打开Tablet的时候，会向MaintenanceManager注册类型为LogGCOp的Maintenance操作。
```
TSTabletManager::OpenTablet
TabletPeer::RegisterMaintenanceOps

void TabletPeer::RegisterMaintenanceOps(MaintenanceManager* maint_mgr) {
  std::lock_guard<simple_spinlock> l(state_change_lock_);

  DCHECK(maintenance_ops_.empty());
  
  # 创建一个新的LogGCOp
  gscoped_ptr<MaintenanceOp> log_gc(new LogGCOp(this));
  
  # 将LogGCOp注册到MaintenanceManager中
  maint_mgr->RegisterOp(log_gc.get());
  maintenance_ops_.push_back(log_gc.release());
}

void MaintenanceManager::RegisterOp(MaintenanceOp* op) {
  std::lock_guard<Mutex> guard(lock_);
  # 将注册的MaintenanceOp添加到MaintenanceManager::ops_中
  pair<OpMapTy::iterator, bool> val
    (ops_.insert(OpMapTy::value_type(op, MaintenanceOpStats())));
  # 设置MaintenanceOp对应的MaintenanceManager
  op->manager_ = shared_from_this();
  op->cond_.reset(new ConditionVariable(&lock_));
}
```

### TabletPeer从MaintenanceManager中解除注册类型为LogGCOp的Maintenance操作
当关闭TabletServer的时候，会从MaintenanceManager中解除所有已经注册的Maintenance操作。
```
TabletServer::Shutdown
TSTabletManager::StartShutdown
TabletPeer::StartShutdown
TabletPeer::UnregisterMaintenanceOps
MaintenanceOp::Unregister
MaintenanceManager::UnregisterOp

void MaintenanceManager::UnregisterOp(MaintenanceOp* op) {
  {
    std::lock_guard<Mutex> guard(lock_);
    auto iter = ops_.find(op);
    
    # 如果当前的MaintenanceOp正在运行过程中，则等待其完成
    while (iter->first->running_ > 0) {
      op->cond_->Wait();
      iter = ops_.find(op);
    }
    
    # 从MaintenanceManager::ops_中移除
    ops_.erase(iter);
  }
  
  ...
}
```

### MaintenanceManager对注册的操作的处理
MaintenanceManager不断地从MaintenanceManager::ops_中找出当前最值得执行的MaintenanceOp，并提交给特定的线程池执行之。
```
TabletServer::Start
MaintenanceManager::Init
MaintenanceManager::RunSchedulerThread
MaintenanceManager::LaunchOp
# 对于LogGCOp来说，对应的是LogGCOp::Perform
MaintenanceOp::Perform

void MaintenanceManager::RunSchedulerThread() {
  std::unique_lock<Mutex> guard(lock_);
  while (true) {
    // Loop until we are shutting down or it is time to run another op.
    cond_.TimedWait(polling_interval);
    if (shutdown_) {
      VLOG_AND_TRACE("maintenance", 1) << "Shutting down maintenance manager.";
      return;
    }

    # 查找当前最适合执行的MaintenanceOp
    MaintenanceOp* op = FindBestOp();
    
    // Prepare the maintenance operation.
    op->running_++;
    running_ops_++;
    guard.unlock();
    bool ready = op->Prepare();
    guard.lock();
    
    # 提交给线程池执行
    Status s = thread_pool_->SubmitFunc(std::bind(&MaintenanceManager::LaunchOp, this, op));
    CHECK(s.ok());
  }
}
```

## LogGCOp的执行逻辑
```
LogGCOp::Perform
TabletPeer::RunLogGC

Status TabletPeer::RunLogGC() {
  # 获取应当被保留的最小的Log Index
  int64_t min_log_index = VERIFY_RESULT(GetEarliestNeededLogIndex());
  int32_t num_gced = 0;
  # 执行WAL log GC
  return log_->GC(min_log_index, &num_gced);
}
```

### 获取应当被保留的最小的Log Index
在TabletPeer::GetEarliestNeededLogIndex中会借助下来对象来共同计算应当被保留的最小的Log Index：
- TabletPeer::log_(类型为log::Log)
- TabletPeer::log_anchor_registry_(类型为log::LogAnchorRegistry)
- TabletPeer::operation_tracker_(类型为OperationTracker)
- TabletPeer::consensus_(类型为consensus::RaftConsensus)，更具体的会用到：
    - RaftConsensus::state_::retryable_requests_(类型为consensus::RetryableRequests)
    - RaftConsensus::state_::last_committed_op_id_(类型为OpId)
- TabletPeer::tablet_(类型为TabletPtr)
    - Tablet::MaxPersistentOpId


后面我们会对这些对象一探究竟。
```
Result<int64_t> TabletPeer::GetEarliestNeededLogIndex(std::string* details) const {
  auto latest_log_entry_op_id = log_->GetLatestEntryOpId();
  int64_t min_index = latest_log_entry_op_id.index;

  // If we never have written to the log, no need to proceed.
  if (min_index == 0) {
    return min_index;
  }

  // Next, we interrogate the anchor registry.
  // Returns OK if minimum known, NotFound if no anchors are registered.
  {
    int64_t min_anchor_index;
    Status s = log_anchor_registry_->GetEarliestRegisteredLogIndex(&min_anchor_index);
    if (PREDICT_FALSE(!s.ok())) {
      DCHECK(s.IsNotFound()) << "Unexpected error calling LogAnchorRegistry: " << s.ToString();
    } else {
      min_index = std::min(min_index, min_anchor_index);
    }
  }

  // Next, interrogate the OperationTracker.
  int64_t min_pending_op_index = std::numeric_limits<int64_t>::max();
  for (const auto& driver : operation_tracker_.GetPendingOperations()) {
    auto tx_op_id = driver->GetOpId();
    // A operation which doesn't have an opid hasn't been submitted for replication yet and
    // thus has no need to anchor the log.
    if (tx_op_id != yb::OpId::Invalid()) {
      min_pending_op_index = std::min(min_pending_op_index, tx_op_id.index);
    }
  }

  min_index = std::min(min_index, min_pending_op_index);

  auto min_retryable_request_op_id = consensus_->MinRetryableRequestOpId();
  min_index = std::min(min_index, min_retryable_request_op_id.index);

  auto* transaction_coordinator = tablet()->transaction_coordinator();
  if (transaction_coordinator) {
    auto transaction_coordinator_min_op_index = transaction_coordinator->PrepareGC(details);
    min_index = std::min(min_index, transaction_coordinator_min_op_index);
  }

  auto last_committed_op_id = consensus()->GetLastCommittedOpId();
  min_index = std::min(min_index, last_committed_op_id.index);

  if (tablet_->table_type() != TableType::TRANSACTION_STATUS_TABLE_TYPE) {
    tablet_->FlushIntentsDbIfNecessary(latest_log_entry_op_id);
    auto max_persistent_op_id = VERIFY_RESULT(
        tablet_->MaxPersistentOpId(true /* invalid_if_no_new_data */));
        
    if (max_persistent_op_id.regular.valid()) {
      min_index = std::min(min_index, max_persistent_op_id.regular.index);
    }
    
    if (max_persistent_op_id.intents.valid()) {
      min_index = std::min(min_index, max_persistent_op_id.intents.index);
    }
  }

  {
    // We should prevent Raft log GC from deleting SPLIT_OP designated for this tablet, because
    // it is used during bootstrap to initialize ReplicaState::split_op_id_ which in its turn
    // is used to prevent already split tablet from serving new ops.
    auto split_op_id = consensus()->GetSplitOpId();
    if (split_op_id) {
      min_index = std::min(min_index, split_op_id.index);
    }
  }

  return min_index;
}
```

### 执行WAL log GC(待续。。。)
```
Status Log::GC(int64_t min_op_idx, int32_t* num_gced) {
  CHECK_GE(min_op_idx, 0);

  LOG_WITH_PREFIX(INFO) << "Running Log GC on " << wal_dir_ << ": retaining ops >= " << min_op_idx
                        << ", log segment size = " << options_.segment_size_bytes;
  VLOG_TIMING(1, "Log GC") {
    SegmentSequence segments_to_delete;

    {
      std::lock_guard<percpu_rwlock> l(state_lock_);
      CHECK_EQ(kLogWriting, log_state_);

      RETURN_NOT_OK(GetSegmentsToGCUnlocked(min_op_idx, &segments_to_delete));

      if (segments_to_delete.size() == 0) {
        VLOG_WITH_PREFIX(1) << "No segments to delete.";
        *num_gced = 0;
        return Status::OK();
      }
      // Trim the prefix of segments from the reader so that they are no longer referenced by the
      // log.
      RETURN_NOT_OK(reader_->TrimSegmentsUpToAndIncluding(
          segments_to_delete[segments_to_delete.size() - 1]->header().sequence_number()));
    }

    // Now that they are no longer referenced by the Log, delete the files.
    *num_gced = 0;
    for (const scoped_refptr<ReadableLogSegment>& segment : segments_to_delete) {
      LOG_WITH_PREFIX(INFO) << "Deleting log segment in path: " << segment->path()
                            << " (GCed ops < " << min_op_idx << ")";
      RETURN_NOT_OK(get_env()->DeleteFile(segment->path()));
      (*num_gced)++;
    }

    // Determine the minimum remaining replicate index in order to properly GC the index chunks.
    int64_t min_remaining_op_idx = reader_->GetMinReplicateIndex();
    if (min_remaining_op_idx > 0) {
      log_index_->GC(min_remaining_op_idx);
    }
  }
  return Status::OK();
}
```

## TabletPeer::log_(log::Log)
请参考[YugaByte Consensus WAL Log](http://note.youdao.com/noteshare?id=2e24e269d3ceadcc5fdcee58826bd25d&sub=425FE1EBD012497BBEAE0558CCCFA6CC)

## TabletPeer::log_anchor_registry_(log::LogAnchorRegistry)
LogAnchorRegistry用于注册感兴趣的log index(OpId index)，主要用于阻止那些引用了被注册到LogAnchorRegistry中的log index的log segment被GC回收。

LogAnchorRegistry中定义了一个LogAnchor结构，基于注册到LogAnchorRegistry中的log index封装：
```
struct LogAnchor {
 # 是否已经注册
  bool is_registered;

  # 注册或者更新的时间
  MonoTime when_registered;

  # 注册的log index
  int64_t log_index;

  # 哪个子系统注册的
  std::string owner;
};
```

LogAnchorRegistry提供的主要接口包括：
```
# 在LogAnchorRegistry中注册某个log index
void Register(int64_t log_index, const std::string& owner, LogAnchor* anchor)

# 原子性的将已注册的LogAnchor更新到一个新的log index
CHECKED_STATUS UpdateRegistration(int64_t log_index, LogAnchor* anchor)

# 解除注册
CHECKED_STATUS UnregisterIfAnchored(LogAnchor* anchor)

# 在LogAnchorRegistry中查找被注册的最小的log index
CHECKED_STATUS GetEarliestRegisteredLogIndex(int64_t* op_id)
```

LogAnchorRegistry的主要使用场景：
```
# raft leader处理来自于learner的关于RemoteBootstrap 开始的请求
RemoteBootstrapServiceImpl::BeginRemoteBootstrapSession
RemoteBootstrapSession::Init
LogAnchorRegistry::Register

# raft leader处理来自于learner的关于RemoteBootstrap 获取数据的请求
RemoteBootstrapServiceImpl::FetchData
RemoteBootstrapSession::GetDataPiece
RemoteBootstrapSession::GetLogSegmentPiece
RemoteBootstrapSession::OpenLogSegment

# raft leader处理来自于learner的关于RemoteBootstrap 结束的请求
RemoteBootstrapServiceImpl::EndRemoteBootstrapSession
RemoteBootstrapSession::~RemoteBootstrapSession
RemoteBootstrapSession::UnregisterAnchorIfNeededUnlocked
LogAnchorRegistry::UnregisterIfAnchored
```

RemoteBootstrapSession主要用于raft learner向raft leader获取所需要的superblock，data blocks和log segments等的过程中。比如因为节点故障，导致tablet副本数目小于replication factor的时候，就会在新的节点上启动新的tablet副本，此时就需要建立RemoteBootstrapSession。相关代码见tserver/remote_bootstrap_client.cc:
```
# RemoteBootstrapClient主要用于从其它节点上拷贝一个tablet
class RemoteBootstrapClient {
  # 和remote bootstrap peer之间建立一个remote bootstrap session
  CHECKED_STATUS Start(const std::string& bootstrap_peer_uuid,
                       rpc::ProxyCache* proxy_cache,
                       const HostPort& bootstrap_peer_addr,
                       scoped_refptr<tablet::RaftGroupMetadata>* metadata,
                       TSTabletManager* ts_manager = nullptr);
  
  # 从remote bootstrap peer上获取相关数据
  CHECKED_STATUS FetchAll(tablet::TabletStatusListener* status_listener);

  # 结束和remote bootstrap peer之间的remote bootstrap session
  CHECKED_STATUS EndRemoteSession();
}
```

## TabletPeer::operation_trakcer_(OperationTracker)
每一个TabletPeer都有一个OperationTracker，用于记录处于pending状态的operations。

在OperationDriver的构造函数中会为之设置OperationTracker，并且设置的是TabletPeer::operation_tracker_：
```
TabletPeer::NewLeaderOperationDriver/TabletPeer::NewReplicaOperationDriver
TabletPeer::NewOperationDriver
TabletPeer::CreateOperationDriver

scoped_refptr<OperationDriver> TabletPeer::CreateOperationDriver() {
  return scoped_refptr<OperationDriver>(new OperationDriver(
      # 指定TabletPeer::operation_tracker_作为其参数
      &operation_tracker_, ...));
}

OperationDriver::OperationDriver(OperationTracker *operation_tracker, ...)
    : operation_tracker_(operation_tracker), ... {
        
}
```

在初始化OperationDriver的时候会将OperationDriver添加到OperationTracker中：
```
Status OperationDriver::Init(std::unique_ptr<Operation>* operation, int64_t term) {
  ...
  
  auto result = operation_tracker_->Add(this);
  
  ...
}
```

当Operation完成Raft复制，且写入了RocksDB之后，就从OperationTracker中移除该OperationDriver。
```
OperationDriver::ReplicationFinished
OperationDriver::ApplyOperation
OperationDriver::ApplyTask

void OperationDriver::ApplyTask(int64_t leader_term, OpIds* applied_op_ids) {
  {
    auto status = operation_->Replicated(leader_term);
    LOG_IF_WITH_PREFIX(FATAL, !status.ok()) << "Apply failed: " << status;
    operation_tracker_->Release(this, applied_op_ids);
  }
}
```

## TabletPeer::consensus_(consensus::RaftConsensus)
### RaftConsensus::state_::retryable_requests_(consensus::RetryableRequests)
关于RetryableRequests，它的实现类是RetryableRequests::Impl，已经在[YugaByte RPC请求超时处理机制](http://note.youdao.com/noteshare?id=0881f04b39a1ab36579a3037ac166831&sub=5EBC531CCD034A3A8D5A6D0817257718)一文中“Server端的RPC处理逻辑”中讲解了，在此不再赘述。

但是在本文中，需要进一步对RetryableRequests::Impl::CleanExpiredReplicatedAndGetMinOpId进行分析：
```
class RetryableRequests::Impl {

  # 获取所有client对应的ClientRetryableRequests中尚未过期的最小的OpId
  yb::OpId CleanExpiredReplicatedAndGetMinOpId() {
    yb::OpId result(std::numeric_limits<int64_t>::max(), std::numeric_limits<int64_t>::max());
    auto now = clock_.Now();
    
    # FLAGS_retryable_request_timeout_secs定义了ClientRetryableRequests::replicated中
    # 所有请求的最大保留时间，计算出的clean_start表示ClientRetryableRequests::replicated中
    # 保留的最小时间戳，所有小于该时间戳的请求都被认为过期了，可以被删除
    auto clean_start =
        now - std::chrono::seconds(GetAtomicFlag(&FLAGS_retryable_request_timeout_secs));
        
    # 逐一遍历每个client对应的ClientRetryableRequests
    for (auto ci = clients_.begin(); ci != clients_.end();) {
      ClientRetryableRequests& client_retryable_requests = ci->second;
      auto& op_id_index = client_retryable_requests.replicated.get<OpIdIndex>();
      auto it = op_id_index.begin();
      int64_t count = 0;
      
      # 遍历ClientRetryableRequests::replicated中的所有请求，查找第一个不小于clean_start的请求
      while (it != op_id_index.end() && it->max_time < clean_start) {
        ++it;
        ++count;
      }
      
      if (it != op_id_index.end()) {
        # 移除it之前的所有的请求
        result = std::min(result, it->min_op_id);
        op_id_index.erase(op_id_index.begin(), it);
      } else {
        # ClientRetryableRequests::replicated中所有请求都过期了，移除之
        op_id_index.clear();
      }
      
      ++ci;
    }

    return result;
  }
}
```

### RaftConsensus::state_::last_committed_op_id_(OpId)
每当接收到raft peer的响应的时候，会获取最新的majority replicated OpId，然后使用该majority replicated OpId更新ReplicaState::last_committed_op_id_，但是最终的ReplicaState::last_committed_op_id_不一定等于majority replicated OpId。
```
PeerMessageQueue::ResponseFromPeer
PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
RaftConsensus::UpdateMajorityReplicated
ReplicaState::UpdateMajorityReplicatedUnlocked
ReplicaState::AdvanceCommittedOpIdUnlocked
ReplicaState::ApplyPendingOperationsUnlocked
ReplicaState::SetLastCommittedIndexUnlocked

Status ReplicaState::ApplyPendingOperationsUnlocked(
    const yb::OpId& committed_op_id, CouldStop could_stop) {
  DCHECK(IsLocked());

  auto prev_id = last_committed_op_id_;
  yb::OpId max_allowed_op_id;
  if (!safe_op_id_waiter_) {
    max_allowed_op_id.index = std::numeric_limits<int64_t>::max();
  }
  auto leader_term = GetLeaderStateUnlocked().term;

  OpIds applied_op_ids;
  applied_op_ids.reserve(committed_op_id.index - prev_id.index);

  # 处理pending_operations_中的每一个请求
  while (!pending_operations_.empty()) {
    auto round = pending_operations_.front();
    auto current_id = yb::OpId::FromPB(round->id());
    
    # 比committed_op_id.index更大的还没有被commit
    if (current_id.index > committed_op_id.index) {
      break;
    }

    # 检查OpId index的连续性(prev_id.term <= current_id.term && 
    # prev_id.index == current_id.index + 1)
    if (PREDICT_TRUE(prev_id)) {
      CHECK_OK(CheckOpInSequence(prev_id, current_id));
    }

    auto type = round->replicate_msg()->op_type();

    // For write operations we block rocksdb flush, until appropriate records are written to the
    // log file. So we could apply them before adding to log.
    if (type == OperationType::WRITE_OP) {
      # 对于WRITE_OP类型的操作的处理
      if (could_stop && !context_->ShouldApplyWrite()) {
        YB_LOG_EVERY_N_SECS(WARNING, 5) << LogPrefix()
            << "Stop apply pending operations, because of write delay required, last applied: "
            << prev_id << " of " << committed_op_id;
        break;
      }
    } else if (current_id.index > max_allowed_op_id.index ||
               current_id.term > max_allowed_op_id.term) {
      # 除WRITE_OP以外的其它操作类型，比如NO_OP，CHANGE_CONFIG_OP等的处理，暂略
      ...
    }

    pending_operations_.pop_front();
    // Set committed configuration.
    if (PREDICT_FALSE(type == OperationType::CHANGE_CONFIG_OP)) {
      ApplyConfigChangeUnlocked(round);
    }

    # 记录上一个commit的OpId
    prev_id = current_id;
    NotifyReplicationFinishedUnlocked(round, Status::OK(), leader_term, &applied_op_ids);
  }

  # 更新ReplicaState::last_committed_op_id_
  SetLastCommittedIndexUnlocked(prev_id);

  applied_ops_tracker_(applied_op_ids);

  return Status::OK();
}

void ReplicaState::SetLastCommittedIndexUnlocked(const yb::OpId& committed_op_id) {
  DCHECK(IsLocked());
  CHECK_GE(last_received_op_id_.index, committed_op_id.index);
  last_committed_op_id_ = committed_op_id;
}
```

## TabletPeer::tablet_(TabletPtr)
### Tablet::MaxPersistentOpId
Tablet::MaxPersistentOpId分别返回regular db和intent db中最大的已经persistent的OpId，保存在DocDbOpIds中。如果参数invalid_if_no_new_data为true，则如果相应的db中没有新的数据，则返回invalid OpId。
```
Result<DocDbOpIds> Tablet::MaxPersistentOpId(bool invalid_if_no_new_data) const {
  ScopedRWOperation scoped_read_operation(&pending_op_counter_);
  RETURN_NOT_OK(scoped_read_operation);

  return DocDbOpIds{
      # 获取regular db中最大的已经persistent的OpId
      MaxPersistentOpIdForDb(regular_db_.get(), invalid_if_no_new_data),
      # 获取intent db中最大的已经persistent的OpId
      MaxPersistentOpIdForDb(intents_db_.get(), invalid_if_no_new_data)
  };
}

yb::OpId MaxPersistentOpIdForDb(rocksdb::DB* db, bool invalid_if_no_new_data) {
  if (db == nullptr ||
      (invalid_if_no_new_data &&
       db->GetFlushAbility() == rocksdb::FlushAbility::kNoNewData)) {
    return yb::OpId::Invalid();
  }

  rocksdb::UserFrontierPtr frontier = db->GetFlushedFrontier();
  if (!frontier) {
    return yb::OpId();
  }

  return down_cast<docdb::ConsensusFrontier*>(frontier.get())->op_id();
}

UserFrontierPtr DBImpl::GetFlushedFrontier() {
  InstrumentedMutexLock l(&mutex_);
  auto result = versions_->FlushedFrontier();
  if (result) {
    return result->Clone();
  }
  std::vector<LiveFileMetaData> files;
  versions_->GetLiveFilesMetaData(&files);
  UserFrontierPtr accumulated;
  for (const auto& file : files) {
    if (!file.imported) {
      UserFrontier::Update(
          file.largest.user_frontier.get(), UpdateUserValueType::kLargest, &accumulated);
    }
  }
  return accumulated;
}
```

在理解上面的代码之前需要先理解Rocksdb中的VersionSet。


