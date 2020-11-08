# 提纲
[toc]

# 综述
YugaByte Consensus WAL Log中涉及到Log，Log::Appender，TaskStream，TaskStreamImpl，WritableLogSegment，ReadableLogSegment，LogCache，LogReader，LogIndex等对象。这些对象的作用分别如下：
- Log是WAL log的总管，它管理着Log::Appender，WritableLogSegment，LogReader，LogIndex等，是WAL log对外的入口。
- Log::Appender用于异步地向WAL log中写入记录，它会将要写入的记录提交给TaskStream。
- TaskStream用于任务调度，它不拥有自己的线程，但是拥有一个特定线程池的token，同时拥有一个任务队列，对于接收到的任务(对于WAL log来说，一个待写入的记录就是一个任务)，它会添加到任务队列中，并唤醒特定线程池中的线程来处理任务队列中的任务，任一时刻，至多只有一个线程在处理任务队列中的任务。
- TaskStreamImpl是TaskStream的具体实现。
- WritableLogSegment表示一个可以写入的log segment。WAL log是一个统称，它实际上是由一系列的log segment组成，只有当前正在被写入的log segment是一个WritableLogSegment，其它的log segment不是WritableLogSegment。WritableLogSegment底层封装了一个WritableFile。
- ReadableLogSegment表示一个可以读取的log segment。它主要用于recovery和raft follower catch up。所有log segment，只有没有被回收，都是一个ReadableLogSegment。除当前正在写入的log segment以外的其它所有ReadableLogSegment整个log segment文件都是可以读取的，但是对于当前正在写入的log segment，只有当前已经sync的数据是可以读取的。
- LogIndex用于记录给定的OpId index所关联的ReplicateMsg在哪个log segment中，以及在log segment中的哪个偏移。在从log segment中读取记录时候被用到。
- LogReader用于从所有ReadableLogSegment中读取记录。它借助于LogIndex来获取一个给定的OpId index所在的log segment，以及在该log segment中的偏移，然后从该log segment中读取记录。

# Log && LogSegment
## 向WAL log中写入记录
```
TabletPeer::Submit
# 省略中间冗长的调用链
...
# 在RaftConsensus::ReplicateBatch中加了一把大锁，确保RaftConsensus::ReplicateBatch
# 的并发调用最终是串行执行的
RaftConsensus::ReplicateBatch
RaftConsensus::AppendNewRoundsToQueueUnlocked
PeerMessageQueue::AppendOperations
LogCache::AppendOperations
Log::AsyncAppendReplicates

Status Log::AsyncAppendReplicates(const ReplicateMsgs& msgs, const yb::OpId& committed_op_id,
                                  RestartSafeCoarseTimePoint batch_mono_time,
                                  const StatusCallback& callback) {
  # 创建一个LogEntryBatchPB，LogEntryBatchPB中包含一个LogEntryPB数组，参数@msgs
  # 类型是ReplicateMsgs，它是一个类型为ReplicateMsg的数组，执行该操作之后，@msgs
  # 中的每一个ReplicateMsg都会和LogEntryBatchPB中的一个LogEntryPB一一对应
  auto batch = CreateBatchFromAllocatedOperations(msgs);
  if (committed_op_id) {
    committed_op_id.ToPB(batch.mutable_committed_op_id());
  }
  
  // Set batch mono time if it was specified.
  if (batch_mono_time != RestartSafeCoarseTimePoint()) {
    batch.set_mono_time(batch_mono_time.ToUInt64());
  }

  # 分配一个LogEntryBatch
  LogEntryBatch* reserved_entry_batch;
  RETURN_NOT_OK(Reserve(REPLICATE, &batch, &reserved_entry_batch));

  // If we're able to reserve, set the vector of replicate shared pointers in the LogEntryBatch.
  // This will make sure there's a reference for each replicate while we're appending.
  reserved_entry_batch->SetReplicates(msgs);

  RETURN_NOT_OK(AsyncAppend(reserved_entry_batch, callback));
  return Status::OK();
}

Status Log::Reserve(LogEntryTypePB type,
                    LogEntryBatchPB* entry_batch,
                    LogEntryBatch** reserved_entry) {
  TRACE_EVENT0("log", "Log::Reserve");
  DCHECK(reserved_entry != nullptr);
  {
    SharedLock<rw_spinlock> read_lock(state_lock_.get_lock());
    CHECK_EQ(kLogWriting, log_state_);
  }

  # 根据LogEntryBatchPB创建一个LogEntryBatch
  auto new_entry_batch = std::make_unique<LogEntryBatch>(type, std::move(*entry_batch));
  new_entry_batch->MarkReserved();

  *reserved_entry = new_entry_batch.release();
  return Status::OK();
}

Status Log::AsyncAppend(LogEntryBatch* entry_batch, const StatusCallback& callback) {
  {
    SharedLock<rw_spinlock> read_lock(state_lock_.get_lock());
    CHECK_EQ(kLogWriting, log_state_);
  }

  entry_batch->set_callback(callback);
  entry_batch->MarkReady();

  # Log::Appender::Submit
  if (PREDICT_FALSE(!appender_->Submit(entry_batch).ok())) {
    delete entry_batch;
    return kLogShutdownStatus;
  }

  return Status::OK();
}

class Log::Appender {
  CHECKED_STATUS Submit(LogEntryBatch* item) {
    # TaskStream<T>::Submit，进一步的调用TaskStreamImpl<T>::Submit
    return task_stream_->Submit(item);
  }
}

template <typename T> Status TaskStreamImpl<T>::Submit(T *task) {
  # 检查是否处于stop状态
  if (stop_requested_.load(std::memory_order_acquire)) {
    return STATUS(IllegalState, "Tablet is shutting down");
  }
  
  # 添加到TaskStreamImpl<T>::queue_中
  if (!queue_.BlockingPut(task)) {
    return STATUS_FORMAT(ServiceUnavailable,
                         "TaskStream queue is full (max capacity $0)",
                         queue_.max_size());
  }

  int expected = 0;
  if (!running_.compare_exchange_strong(expected, 1, std::memory_order_acq_rel)) {
    # 已经有线程正在处理TaskStreamImpl<T>::queue_过程中
    return Status::OK();
  }
  
  # 向线程池中提交一个新的任务：TaskStreamImpl::Run，用于处理TaskStreamImpl<T>::queue_中任务
  return taskstream_pool_token_->SubmitFunc(std::bind(&TaskStreamImpl::Run, this));
}

template <typename T> void TaskStreamImpl<T>::Run() {
  for (;;) {
    MonoTime wait_timeout_deadline = MonoTime::Now() + queue_max_wait_;
    # 将TaskStreamImpl<T>::queue_中所有任务都转移到数组@group中
    std::vector<T *> group;
    queue_.BlockingDrainTo(&group, wait_timeout_deadline);
    if (!group.empty()) {
      # 逐一处理数组group中的任务
      for (T* item : group) {
        # 在当前上下文中，ProcessItem最终调用的是在Log::Appender构造函数中初始化
        # Log::Appender::task_stream_时设置的Log::Appender::ProcessBatch
        ProcessItem(item);
      }
      
      # 用nullptr充当一个特殊的task，在当前上下文中主要用于执行sync，以及调用回调
      ProcessItem(nullptr);
      group.clear();
      
      # 继续后续处理
      continue;
    }
    
    // Not processing and queue empty, return from task.
    std::unique_lock<std::mutex> stop_lock(stop_mtx_);
    running_--;
    if (!queue_.empty()) {
      // Got more operations, stay in the loop.
      running_++;
      continue;
    }
    
    if (stop_requested_.load(std::memory_order_acquire)) {
      VLOG(1) << "TaskStream task's Run() function is returning because stop is requested.";
      stop_cond_.notify_all();
      return;
    }
    
    VLOG(1) << "Returning from TaskStream task after inactivity:" << this;
    return;
  }
}

void Log::Appender::ProcessBatch(LogEntryBatch* entry_batch) {
  # entry_batch为nullptr时，执行sync，并调用回调
  if (entry_batch == nullptr) {
    // Here, we do sync and call callbacks.
    GroupWork();
    return;
  }

  # Log::Appender::sync_batch_初始化为空，当执行了Log::Appender::GroupWork之后也为空
  if (sync_batch_.empty()) { // Start of batch.
    auto sleep_duration = log_->sleep_duration_.load(std::memory_order_acquire);
    if (sleep_duration.count() > 0) {
      std::this_thread::sleep_for(sleep_duration);
    }
    time_started_ = MonoTime::Now();
  }
  
  # append到log中
  Status s = log_->DoAppend(entry_batch);

  if (PREDICT_FALSE(!s.ok())) {
    # 失败情况下的处理，略
    ...
    
    return;
  }
  
  if (!log_->sync_disabled_) {
    bool expected = false;
    
    # Log::periodic_sync_needed_表示是否有需要sync的LogEntryBatch，
    # Log::periodic_sync_needed_将在Log::Sync中被使用
    if (log_->periodic_sync_needed_.compare_exchange_strong(expected, true,
                                                            std::memory_order_acq_rel)) {
      log_->periodic_sync_earliest_unsync_entry_time_ = MonoTime::Now();
    }
    
    # 更新“尚未synced字节数目”
    log_->periodic_sync_unsynced_bytes_ += entry_batch->total_size_bytes();
  }
  
  # 将当前的LogEntryBatch添加到sync_batch_中
  sync_batch_.emplace_back(entry_batch);
}

void Log::Appender::GroupWork() {
  if (sync_batch_.empty()) {
    # 如果Log::Appender::sync_batch_为空，则只需要对log执行sync即可
    Status s = log_->Sync();
    return;
  }
  
  # 否则，除了需要对log执行sync以外，还需要调用Log::Appender::sync_batch_中
  # 每一个LogEntryBatch的回调
  
  # ScopeExit中是一个lambda语句，它无论是在正常退出还是异常退出的情况下都会执行
  auto se = ScopeExit([this] {
    if (log_->metrics_) {
      MonoTime time_now = MonoTime::Now();
      log_->metrics_->group_commit_latency->Increment(
          time_now.GetDeltaSince(time_started_).ToMicroseconds());
    }
    
    # 清空Log::Appender::sync_batch_
    sync_batch_.clear();
  });

  # 对log执行sync
  Status s = log_->Sync();
  if (PREDICT_FALSE(!s.ok())) {
    # sync出错情况下，调用每一个LogEntryBatch的回调
    for (std::unique_ptr<LogEntryBatch>& entry_batch : sync_batch_) {
      if (!entry_batch->callback().is_null()) {
        entry_batch->callback().Run(s);
      }
    }
  } else {
    TRACE_EVENT0("log", "Callbacks");
    # 调用每一个LogEntryBatch的回调
    for (std::unique_ptr<LogEntryBatch>& entry_batch : sync_batch_) {
      if (PREDICT_TRUE(!entry_batch->failed_to_append() && !entry_batch->callback().is_null())) {
        entry_batch->callback().Run(Status::OK());
      }
      
      entry_batch.reset();
    }
    
    sync_batch_.clear();
  }
}

Status Log::DoAppend(LogEntryBatch* entry_batch,
                     bool caller_owns_operation,
                     bool skip_wal_write) {
  if (!skip_wal_write) {
    # 将LogEntryBatch序列化之后保存在LogEntryBatch::buffer_中
    RETURN_NOT_OK(entry_batch->Serialize());
    # 返回一个Slice，引用LogEntryBatch::buffer_
    Slice entry_batch_data = entry_batch->data();

    uint32_t entry_batch_bytes = entry_batch->total_size_bytes();
    // If there is no data to write return OK.
    if (PREDICT_FALSE(entry_batch_bytes == 0)) {
      return Status::OK();
    }

    # Log::allocation_state_包括3种状态，分别是：kAllocationNotStarted，kAllocationInProgress
    # 和kAllocationFinished，其中kAllocationNotStarted表示没有请求log segment分配，
    # kAllocationInProgress表示有一个log segment分配请求正在进行中，kAllocationFinished则
    # 表示log segment分配已经完成，关于这3种状态的变迁，见“预分配一个新的log segment”
    
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
        # 已经分配了新的log segment，则切换到新的log segment上
        RETURN_NOT_OK(RollOver());
      }
    } else {
      VLOG_WITH_PREFIX(1) << "Segment allocation already in progress...";
    }

    int64_t start_offset = active_segment_->written_offset();

    LOG_SLOW_EXECUTION(WARNING, 50, "Append to log took a long time") {
      # 将LogEntryBatch对应的数据写入当前的log segment中
      RETURN_NOT_OK(active_segment_->WriteEntryBatch(entry_batch_data));
    }

    # 设置LogEntryBatch对应的log segment序列号，及其在该log segment中的偏移
    entry_batch->offset_ = start_offset;
    entry_batch->active_segment_sequence_number_ = active_segment_sequence_number_;
  }

  # 记录最后一个写入的OpId
  if (entry_batch->count()) {
    last_appended_entry_op_id_ = yb::OpId::FromPB(entry_batch->MaxReplicateOpId());
  }

  CHECK_OK(UpdateIndexForBatch(*entry_batch));
  # 使用@entry_batch中的相关信息更新当前log segment对应的index boundary，
  # 包括Log::footer_builder_::min_replicate_index_，
  # Log::footer_builder_::max_replicate_index_和Log::min_replicate_index_
  UpdateFooterForBatch(entry_batch);

  ...

  return Status::OK();
}

Status WritableLogSegment::WriteEntryBatch(const Slice& data) {
  # 首先将data的length和data的crc写入header，然后再将header的crc写入到header中，
  # 最后将header写入当前log segment对应的文件中
  uint8_t header_buf[kEntryHeaderSize];

  // First encode the length of the message.
  uint32_t len = data.size();
  InlineEncodeFixed32(&header_buf[0], len);

  // Then the CRC of the message.
  uint32_t msg_crc = crc::Crc32c(data.data(), data.size());
  InlineEncodeFixed32(&header_buf[4], msg_crc);

  // Then the CRC of the header
  uint32_t header_crc = crc::Crc32c(&header_buf, 8);
  InlineEncodeFixed32(&header_buf[8], header_crc);

  // Write the header to the file, followed by the batch data itself.
  RETURN_NOT_OK(writable_file_->Append(Slice(header_buf, sizeof(header_buf))));
  written_offset_ += sizeof(header_buf);

  RETURN_NOT_OK(writable_file_->Append(data));
  written_offset_ += data.size();

  return Status::OK();
}
```

## 预分配一个新的log segment
### 预分配主体逻辑
```
Status Log::AsyncAllocateSegment() {
  std::lock_guard<decltype(allocation_mutex_)> lock_guard(allocation_mutex_);
  CHECK_EQ(allocation_state_, kAllocationNotStarted);
  # Log::allocation_status_中保存log segment的分配状态信息
  allocation_status_.Reset();
  # Log::allocation_state_被设置为kAllocationInProgress
  allocation_state_ = kAllocationInProgress;
  
  # 向线程池中提交一个任务Log::SegmentAllocationTask
  return allocation_pool_->SubmitClosure(Bind(&Log::SegmentAllocationTask, Unretained(this)));
}

void Log::SegmentAllocationTask() {
  # PreAllocateNewSegment执行log segment分配，分配结果保存在Log::allocation_status_中
  allocation_status_.Set(PreAllocateNewSegment());
}

Status Log::PreAllocateNewSegment() {
  CHECK_EQ(allocation_state(), kAllocationInProgress);

  WritableFileOptions opts;
  // We always want to sync on close: https://github.com/yugabyte/yugabyte-db/issues/3490
  opts.sync_on_close = true;
  opts.o_direct = durable_wal_write_;
  RETURN_NOT_OK(CreatePlaceholderSegment(opts, &next_segment_path_, &next_segment_file_));

  if (options_.preallocate_segments) {
    # 预先执行fallocate
    uint64_t next_segment_size = NextSegmentDesiredSize();
    RETURN_NOT_OK(next_segment_file_->PreAllocate(next_segment_size));
  }

  {
    # 修改Log::allocation_state_为kAllocationFinished
    std::lock_guard<boost::shared_mutex> lock_guard(allocation_mutex_);
    allocation_state_ = kAllocationFinished;
  }
  
  return Status::OK();
}
```

### 为Tablet预分配第一个log segment
在初始化Tablet的Log的过程中，如果Log::create_new_segment_at_start_为true，则会通过Log::EnsureInitialNewSegmentAllocated预分配一个新的log segment。
```
TSTabletManager::OpenTablet
BootstrapTablet
BootstrapTabletImpl
TabletBootstrap::Bootstrap
TabletBootstrap::OpenNewLog
Log::Open
Log::Init
Log::EnsureInitialNewSegmentAllocated

Status Log::Init() {
  std::lock_guard<percpu_rwlock> write_lock(state_lock_);
  CHECK_EQ(kLogInitialized, log_state_);
  
  ...
  
  if (create_new_segment_at_start_) {
    RETURN_NOT_OK(EnsureInitialNewSegmentAllocated());
  }
  return Status::OK();
}

Status Log::EnsureInitialNewSegmentAllocated() {
  # 在Log::Init中已经持有了Log::state_lock_锁，所以这里没有再次加锁，
  # 在Log::Init中，log_state_是LogState::kLogInitialized，而LogState::kLogWriting
  # 只有在Log::EnsureInitialNewSegmentAllocated中会被设置，所以在当前上下文中，
  # if语句不可能被满足？
  if (log_state_ == LogState::kLogWriting) {
    // New segment already created.
    return Status::OK();
  }
  
  # 当前log_state_一定处于LogState::kLogInitialized状态
  if (log_state_ != LogState::kLogInitialized) {
    return STATUS_FORMAT(
        IllegalState, "Unexpected log state in EnsureInitialNewSegmentAllocated: $0", log_state_);
  }
  
  # 异步分配log segment，分配状态将保存在allocation_status_中，
  # allocation_status_是YugaByte自定义的Promise<Status>类型，根c++的Promise不同
  RETURN_NOT_OK(AsyncAllocateSegment());
  
  # 等待分配结果，只有当分配结束(成功或者失败)之后allocation_status_才会被设置，
  # allocation_status_.Get()才会返回
  RETURN_NOT_OK(allocation_status_.Get());
  
  # 切换到新分配的log segment上(简化版的RollOver，因为当前是在Log::Init过程中，
  # 无需执行RollOver中的其它工作)
  RETURN_NOT_OK(SwitchToAllocatedSegment());

  # 初始化Log::Appender，但实际上什么也没做
  RETURN_NOT_OK(appender_->Init());
  
  # 修改log_state_为LogState::kLogWriting
  log_state_ = LogState::kLogWriting;
  return Status::OK();
}
```

### 在写log的过程中因为当前log segment空间不足而分配新的log segment
```
Status Log::DoAppend(LogEntryBatch* entry_batch,
                     bool caller_owns_operation,
                     bool skip_wal_write) {
  if (!skip_wal_write) {
    ...

    if (allocation_state() == kAllocationNotStarted) {
      # 如果当前的log segment(active_segment_)不足以容纳当前的LogEntryBatch，且当前的
      # Log::allocation_state_为kAllocationNotStarted，即当前没有已经分配好的log segment，
      # 也没有正在分配过程中的log segment，则请求异步分配一个新的log segment
      if ((active_segment_->Size() + entry_batch_bytes + 4) > cur_max_segment_size_) {
        # 请求异步分配一个新的log segment
        RETURN_NOT_OK(AsyncAllocateSegment());
        
        # 如果设置了“同步等待分配结果”，则等待
        if (!options_.async_preallocate_segments) {
          LOG_SLOW_EXECUTION(WARNING, 50, "Log roll took a long time") {
            # RollOver中会同步等待分配结果
            RETURN_NOT_OK(RollOver());
          }
        }
      }
    } else if (allocation_state() == kAllocationFinished) {
      # log segment分配完成，则切换到该log segment上
      LOG_SLOW_EXECUTION(WARNING, 50, "Log roll took a long time") {
        RETURN_NOT_OK(RollOver());
      }
    } else {
      VLOG_WITH_PREFIX(1) << "Segment allocation already in progress...";
    }
    
    ...
  }
  
  ...
}
```

### Log::log_state_的状态变迁
Log::log_state_类型为LogState，包括3种状态，分别是：kLogInitialized，kLogWriting和kLogClosed。状态变迁如下：
- Log::log_state_的初始状态就是kLogInitialized
- 一旦成功分配了该Tablet相关的第一个Log之后，就进入kLogWriting状态
- 在关闭Log的时候，进入kLogClosed状态。

### Log::allocation_state_的状态变迁
Log::allocation_state_类型是SegmentAllocationState，包括3种状态，分别是：kAllocationNotStarted，kAllocationInProgress和kAllocationFinished。状态变迁如下：
- Log::allocation_state_的初始状态就是kAllocationNotStarted
- 一旦有log segment异步分配请求，就会将Log::allocation_state_的状态切换为kAllocationInProgress
- 一旦成功分配了新的log segment，就将Log::allocation_state_的状态切换为kAllocationFinished
- 当成功切换到新的log segment上之后，Log::allocation_state_的状态又被重置为kAllocationNotStarted

## log segment切换
在向log写入的过程中，如果当前的log segment空间不足，则会创建一个新的log segment，并且切换到该新的log segment上。切换操作是由Log::RollOver完成的，主要执行以下逻辑：
- 等待新的log segmetn分配结果
- 如果新的log segment分配失败，则返回
- 对当前log segment执行sync操作
- 向当前的log segment中写入footer信息，并关闭当前的log segment
- 切换到新的log segment
```
Status Log::RollOver() {
  # 等待log segment分配结果
  RETURN_NOT_OK(allocation_status_.Get());

  # 至此，一定分配成功了，Log::allocation_state_一定为kAllocationFinished
  DCHECK_EQ(allocation_state(), kAllocationFinished);

  # 在当前的log segment(active_segment_)上执行sync，见“log segment sync”
  RETURN_NOT_OK(Sync());
  
  # 在当前的log segment(active_segment_)中写入footer信息
  RETURN_NOT_OK(CloseCurrentSegment());

  # 切换到新分配的log segment
  RETURN_NOT_OK(SwitchToAllocatedSegment());

  return Status::OK();
}
```

### log segment footer信息的维护
#### log segment footer的定义
Log::footer_builder_类型是LogSegmentFooterPB，定义如下：
```
class LogSegmentFooterPB : public ::google::protobuf::Message {
  ::google::protobuf::int64 num_entries_;
  ::google::protobuf::int64 close_timestamp_micros_;
  ::google::protobuf::int64 min_replicate_index_;
  ::google::protobuf::int64 max_replicate_index_;
}
```

#### 重置内存中的log segment footer
每当执行Log::SwitchToAllocatedSegment的时候，都会将Log::footer_builder_重置：
```
Status Log::SwitchToAllocatedSegment() {
  ...
  
  footer_builder_.Clear();
  footer_builder_.set_num_entries(0);

  ...
}
```

#### 在写log的过程中更新内存中的log segment footer
在写Log的过程中，会根据写入的LogEntryBatch的相关信息更新当前log segment对应的index boundary，包括Log::footer_builder_::min_replicate_index_，Log::footer_builder_::max_replicate_index_和Log::min_replicate_index_。
```
Status Log::DoAppend(LogEntryBatch* entry_batch,
                     bool caller_owns_operation,
                     bool skip_wal_write) {
  ...
  
  CHECK_OK(UpdateIndexForBatch(*entry_batch));
  UpdateFooterForBatch(entry_batch);

  ...
}

Status Log::UpdateIndexForBatch(const LogEntryBatch& batch) {
  if (batch.type_ != REPLICATE) {
    return Status::OK();
  }

  # 设置LogEntryBatch中的每一个LogEntryPB对应的LogIndexEntry，并将这些
  # LogIndexEntry添加到Log::log_index_中，每一个LogIndexEntry中记录的是
  # 一个OpId到它所对应的log segment sequence number及在该log segment中
  # 的偏移
  for (const LogEntryPB& entry_pb : batch.entry_batch_pb_.entry()) {
    LogIndexEntry index_entry;

    index_entry.op_id = yb::OpId::FromPB(entry_pb.replicate().id());
    index_entry.segment_sequence_number = batch.active_segment_sequence_number_;
    index_entry.offset_in_segment = batch.offset_;
    RETURN_NOT_OK(log_index_->AddEntry(index_entry));
  }
  
  return Status::OK();
}

void Log::UpdateFooterForBatch(LogEntryBatch* batch) {
  footer_builder_.set_num_entries(footer_builder_.num_entries() + batch->count());

  # 更新当前log segment对应的index boundary，包括Log::footer_builder_::min_replicate_index_，
  # Log::footer_builder_::max_replicate_index_和Log::min_replicate_index_
  for (const LogEntryPB& entry_pb : batch->entry_batch_pb_.entry()) {
    int64_t index = entry_pb.replicate().id().index();
    if (!footer_builder_.has_min_replicate_index() ||
        index < footer_builder_.min_replicate_index()) {
      footer_builder_.set_min_replicate_index(index);
      min_replicate_index_.store(index, std::memory_order_release);
    }
    
    if (!footer_builder_.has_max_replicate_index() ||
        index > footer_builder_.max_replicate_index()) {
      footer_builder_.set_max_replicate_index(index);
    }
  }
}
```

### 在关闭当前log segment之前将内存中的log segment footer信息写入log segment对应的文件
```
Status Log::CloseCurrentSegment() {
  # footer_builder_中保存的是内存中的log segment footer信息
  if (!footer_builder_.has_min_replicate_index()) {
    # footer_builder_中没有设置min_replicate_index_信息，则表明footer_builder_没有被设置
    VLOG_WITH_PREFIX(1) << "Writing a segment without any REPLICATE message. Segment: "
                        << active_segment_->path();
  }

  # 设置该log segment关闭的时间戳
  footer_builder_.set_close_timestamp_micros(GetCurrentTimeMicros());
  
  # 写入，然后关闭
  return active_segment_->WriteFooterAndClose(footer_builder_);
}

Status WritableLogSegment::WriteFooterAndClose(const LogSegmentFooterPB& footer) {
  DCHECK(IsHeaderWritten());
  DCHECK(!IsFooterWritten());
  DCHECK(footer.IsInitialized()) << footer.InitializationErrorString();

  # log segment的尾部写入：序列化后的LogSegmentFooterPB + kLogSegmentFooterMagicString + 
  # LogSegmentFooterPB's size
  
  faststring buf;
  if (!pb_util::AppendToString(footer, &buf)) {
    return STATUS(Corruption, "unable to encode header");
  }

  buf.append(kLogSegmentFooterMagicString);
  PutFixed32(&buf, footer.ByteSize());

  # 写入当前log segment对应的文件中
  RETURN_NOT_OK_PREPEND(writable_file()->Append(Slice(buf)), "Could not write the footer");

  footer_.CopyFrom(footer);
  is_footer_written_ = true;

  # 关闭当前log segment对应的文件
  RETURN_NOT_OK(writable_file_->Close());

  written_offset_ += buf.size();

  return Status::OK();
}
```

### 切换到新分配的log segment
```
Status Log::SwitchToAllocatedSegment() {
  CHECK_EQ(allocation_state(), kAllocationFinished);

  # 新的log segment对应的sequence number
  active_segment_sequence_number_++;
  const string new_segment_path =
      FsManager::GetWalSegmentFilePath(wal_dir_, active_segment_sequence_number_);

  # next_segment_path_是在异步分配log segment的过程中临时设置的文件名
  RETURN_NOT_OK(get_env()->RenameFile(next_segment_path_, new_segment_path));
  RETURN_NOT_OK(get_env()->SyncDir(wal_dir_));

  std::unique_ptr<WritableLogSegment> new_segment(
      new WritableLogSegment(new_segment_path, next_segment_file_));

  # 设置log segment header信息
  LogSegmentHeaderPB header;
  header.set_major_version(kLogMajorVersion);
  header.set_minor_version(kLogMinorVersion);
  header.set_sequence_number(active_segment_sequence_number_);
  header.set_tablet_id(tablet_id_);

  # 重置log segment footer信息
  footer_builder_.Clear();
  footer_builder_.set_num_entries(0);

  # 在log segment header中记录当前log对应的Tablet的schema信息
  {
    SharedLock<decltype(schema_lock_)> l(schema_lock_);
    SchemaToPB(schema_, header.mutable_schema());
    header.set_schema_version(schema_version_);
  }

  # 写入log segment header信息
  RETURN_NOT_OK(new_segment->WriteHeaderAndOpen(header));
  
  # 将当前的active log segment(还没有切换到新的log segment之前)转换为ReadableLogSegment，
  # 然后添加给LogReader管理，LogReader主要用于replay log segments for other raft peers
  # 关于LogReader，将会单独讲解
  {
    if (active_segment_.get() != nullptr) {
      std::lock_guard<decltype(state_lock_)> l(state_lock_);
      CHECK_OK(ReplaceSegmentInReaderUnlocked());
    }
  }

  # 基于新的log segment生成ReadableLogSegment，并添加给LogReader管理
  std::unique_ptr<RandomAccessFile> readable_file;
  RETURN_NOT_OK(get_env()->NewRandomAccessFile(new_segment_path, &readable_file));

  scoped_refptr<ReadableLogSegment> readable_segment(
    new ReadableLogSegment(new_segment_path,
                           shared_ptr<RandomAccessFile>(readable_file.release())));
  RETURN_NOT_OK(readable_segment->Init(header, new_segment->first_entry_offset()));
  RETURN_NOT_OK(reader_->AppendEmptySegment(readable_segment));

  # 将active_segment_设置为新分配的log segment
  active_segment_.reset(new_segment.release());
  cur_max_segment_size_ = NextSegmentDesiredSize();

  # 重置Log::allocation_state_为kAllocationNotStarted
  {
    std::lock_guard<decltype(allocation_mutex_)> lock_guard(allocation_mutex_);
    allocation_state_ = kAllocationNotStarted;
  }

  return Status::OK();
}

Status WritableLogSegment::WriteHeaderAndOpen(const LogSegmentHeaderPB& new_header) {
  # log segment头部信息：kLogSegmentHeaderMagicString + LogSegmentHeaderPB’s size +
  # 序列化后的LogSegmentHeaderPB
  faststring buf;
  // First the magic.
  buf.append(kLogSegmentHeaderMagicString);
  // Then Length-prefixed header.
  PutFixed32(&buf, new_header.ByteSize());
  // Then Serialize the PB.
  if (!pb_util::AppendToString(new_header, &buf)) {
    return STATUS(Corruption, "unable to encode header");
  }
  RETURN_NOT_OK(writable_file()->Append(Slice(buf)));

  header_.CopyFrom(new_header);
  first_entry_offset_ = buf.size();
  written_offset_ = first_entry_offset_;
  is_header_written_ = true;

  return Status::OK();
}
```

## log segment sync
### log segment sync的时机
log segment sync在以下时机发生：
- 同步写入log segment，在每次写入log之后，执行一次log segment sync操作
```
Log::Append
Log::Sync
```

- 异步写入log segment的过程中，会将LogEntryBatch提交给TaskStreamImpl<T>::queue_，每当执行线程被调度执行的时候，会批量处理TaskStreamImpl<T>::queue_中所有的LogEntryBatch，当这一批LogEntryBatch都被处理之后，会执行一次log segment sync
```
Log::AsyncAppend(LogEntryBatch* entry, ...)
Log::Appender::Submit
TaskStream<T>::Submit
# 将LogEntryBatch提交给TaskStreamImpl<T>::queue_
TaskStreamImpl<T>::Submit

# 当执行线程被调度的时候，执行TaskStreamImpl<T>::Run
template <typename T> void TaskStreamImpl<T>::Run() {
  for (;;) {
    # 获取TaskStreamImpl<T>::queue_中所有的LogEntryBatch，保存在@group中
    std::vector<T *> group;
    queue_.BlockingDrainTo(&group, wait_timeout_deadline);
    
    if (!group.empty()) {
      # 逐一处理group中的每一个LogEntryBatch，ProcessItem实际是Log::Appender::ProcessBatch
      for (T* item : group) {
        ProcessItem(item);
      }
      
      # 最后提交一个空的LogEntryBatch，作为一个特殊的信号，在Log::Appender::ProcessBatch中
      # 对空LogEntryBatch的处理逻辑是：执行log segment sync + 调用这一批处理的所有的
      # LogEntryBatch的回调
      ProcessItem(nullptr);
      group.clear();
      continue;
    }
    
    ...
  }
}

void Log::Appender::ProcessBatch(LogEntryBatch* entry_batch) {
  # 处理空的LogEntryBatch
  if (entry_batch == nullptr) {
    // Here, we do sync and call callbacks.
    GroupWork();
    return;
  }
  
  ...
}

Log::Appender::GroupWork
Log::Sync
```

- 在Tserver shutdown的过程中关闭Tablet的WAL log的时候会在当前活跃的log segment执行sync
```
TabletServer::Shutdown
TSTabletManager::CompleteShutdown
TabletPeer::CompleteShutdown
Log::Close
# 只有在Log::log_state_为kLogWriting状态才会执行
Log::Sync
```

- 在切换log segment之前对当前活跃的log segment执行sync
```
Log::RollOver
Log::Sync
```

### log segment sync的主要逻辑
```
Status Log::Sync() {
  if (!sync_disabled_) {
    # timed_or_data_limit_sync表示自从上次sync以来的时间是否超过了interval_durable_wal_write_，
    # 或者自从上次sync以来未sync的数据总量是否超过了阈值
    bool timed_or_data_limit_sync = false;
    
    # durable_wal_write_表示是否在每次写log的时候都执行sync，
    # periodic_sync_needed_表示log中是否存在需要被sync的entries
    if (!durable_wal_write_ && periodic_sync_needed_.load()) {
      if (interval_durable_wal_write_) {
        if (MonoTime::Now() > periodic_sync_earliest_unsync_entry_time_
            + interval_durable_wal_write_) {
          # 自从上次sync以来的时间超过了interval_durable_wal_write_
          timed_or_data_limit_sync = true;
        }
      }
      
      if (bytes_durable_wal_write_mb_ > 0) {
        if (periodic_sync_unsynced_bytes_ >= bytes_durable_wal_write_mb_ * 1_MB) {
          # 自从上次sync以来未sync的数据总量超过了阈值
          timed_or_data_limit_sync = true;
        }
      }
    }

    if (durable_wal_write_ || timed_or_data_limit_sync) {
      # 重置periodic_sync_needed_和periodic_sync_unsynced_bytes_
      periodic_sync_needed_.store(false);
      periodic_sync_unsynced_bytes_ = 0;
      LOG_SLOW_EXECUTION(WARNING, 50, "Fsync log took a long time") {
        # 对当前活跃的log segment执行sync，调用WritableLogSegment::Sync
        RETURN_NOT_OK(active_segment_->Sync());
      }
    }
  }

  # 更新LogReader，使LogReader可以读取到active log segment中已经sync的最新数据
  reader_->UpdateLastSegmentOffset(active_segment_->written_offset());

  {
    # 使用last_appended_entry_op_id_更新last_synced_entry_op_id_
    std::lock_guard<std::mutex> write_lock(last_synced_entry_op_id_mutex_);
    last_synced_entry_op_id_.store(last_appended_entry_op_id_, boost::memory_order_release);
    last_synced_entry_op_id_cond_.notify_all();
  }

  return Status::OK();
}

WritableLogSegment::Sync() {
  return writable_file_->Sync();
}
```

## WritableLogSegment
WritableLogSegment表示可写的log segment，只有当前正在写入的log segment是可写入的。从WritableLogSegment的定义来解读WritableLogSegment：
```
// A writable log segment where state data is stored.
class WritableLogSegment {
 private:
  # log segment对应的文件名
  const std::string path_;

  # log segment对应的WritableFile
  const std::shared_ptr<WritableFile> writable_file_;

  # 是否已经写了header部分
  bool is_header_written_;

  # 是否已经写了footer部分
  bool is_footer_written_;

  # log segment的header，在第一次向WritableLogSegment写入记录之前，会先写入header
  LogSegmentHeaderPB header_;

  # log segment的footer，在向WritableLogSegment写入记录的过程中，会不断更新footer，
  # 在关闭WritableLogSegment之前会写入footer
  LogSegmentFooterPB footer_;

  # 第一条记录的偏移(log segment中header之后的第一个字节的偏移)
  int64_t first_entry_offset_;

  # 下一次写入位置
  int64_t written_offset_;
}
```

## ReadableLogSegment
所有未被GC的log segment，包括当前正在被写入的log segment，都是ReadableLogSegment。从ReadableLogSegment的定义来解读ReadableLogSegment：
```
class ReadableLogSegment : public RefCountedThreadSafe<ReadableLogSegment> {
  # 文件路径
  const std::string path_;

  # 对于正在被写入的log segment来说，它对应的ReadableLogSegment的file_size_每当sync
  # 之后就会更新，更新为当前已经sync的offset，对于其它的log segment来说，它对应的
  # ReadableLogSegment的file_size_就是文件的总大小
  AtomicInt<int64_t> file_size_;

  # 该ReadableLogSegment中，截止到哪个偏移的数据都是可以读取的，对于正在被写入的log
  # segment来说，它对应的ReadableLogSegment的readable_to_offset_在每次sync之后更新，
  # 更新为当前已经sync的offset，对于其它的log segment来说，它对应的ReadableLogSegment
  # 的readable_to_offset_就是文件的总大小
  AtomicInt<int64_t> readable_to_offset_;

  # 对应的RandomAccessFile
  const std::shared_ptr<RandomAccessFile> readable_file_;

  # 
  std::shared_ptr<RandomAccessFile> readable_file_checkpoint_;

  bool is_initialized_;

  # header部分，对应于log segment的header部分
  LogSegmentHeaderPB header_;

  # footer部分，对应于log segment的footer部分，如果这个log segmet正在写入过程中，
  # 则footer_没有被设置
  LogSegmentFooterPB footer_;

  # footer信息是否是重建而来的(对于正在写入log segment过程中crash的情况，在恢复的
  # 时候，需要根据log segment中的记录重建footer)
  bool footer_was_rebuilt_;

  # 第一条记录的偏移(log segment中header之后的第一个字节的偏移)
  int64_t first_entry_offset_;
}
```

# LogCache
## 综述
LogCache是一个Write-through cache。它主要用于记录一个ReplicateMsg对应的OpId index到该ReplicateMsg的映射。在将Replicate写入log segment之前，会先添加到LogCache中。

## 数据结构
LogCache是一个从ReplicateMsg对应的OpId index到该ReplicateMsg的映射，对于映射的value部分，它基于ReplicateMsg封装了一个CacheEntry。
```
class LogCache {
  # 对ReplicateMsg的封装
  struct CacheEntry {
    # 缓存的ReplicateMsg
    ReplicateMsgPtr msg;
    # 占用的内存空间
    int64_t mem_usage;
    bool tracked = false;
  };
  
  # 所关联的Log
  scoped_refptr<log::Log> const log_;

  // The UUID of the local peer.
  const std::string local_uuid_;

  # 所关联的Tablet id
  const std::string tablet_id_;

  mutable simple_spinlock lock_;

  # ReplicateMsg相关的buffer cache，是一个关于ReplicateMsg OpId index到ReplicateMsg的映射表
  typedef std::map<uint64_t, CacheEntry> MessageCache;
  MessageCache cache_;

  # 期望的下一个ReplicateMsg对应的OpId index，下一个被添加到LogCache中的ReplicateMsg
  # 的OpId index不能大于next_sequential_op_index_，如果等于next_sequential_op_index_，
  # 则恰好如期望，如果小于next_sequential_op_index_，则表明要替换掉LogCache中的某些
  # ReplicateMsg
  int64_t next_sequential_op_index_;

  # LogCache中所有>= min_pinned_op_index_的CacheEntry都不能够被evicted
  int64_t min_pinned_op_index_;
}
```

## 主要逻辑
**LogCache::AppendOperations**
> 先将给定的ReplicateMsgs添加到LogCache中，然后写入log segment文件中。
```
Status LogCache::AppendOperations(const ReplicateMsgs& msgs, const yb::OpId& committed_op_id,
                                  RestartSafeCoarseTimePoint batch_mono_time,
                                  const StatusCallback& callback) {
  PrepareAppendResult prepare_result;
  if (!msgs.empty()) {
    # 在追加ReplicateMsgs到WAL log和LogCache中之前做一些准备工作，
    # 主要是对ReplicateMsgs进行一些检查，并将ReplicateMsgs添加到LogCache中
    prepare_result = VERIFY_RESULT(PrepareAppendOperations(msgs));
  }

  # 向log segment中写入这些ReplicateMsgs
  # 因为是异步写入，需要一个回调LogCache::LogCallback来通知写入结果，当写入失败
  # 或者这一批ReplicateMsgs被成功写入log segment中且已经sync的情况下，该回调会被
  # 调用
  Status log_status = log_->AsyncAppendReplicates(
    msgs, committed_op_id, batch_mono_time,
    Bind(&LogCache::LogCallback, Unretained(this), prepare_result.last_idx_in_batch, callback));

  return Status::OK();
}

Result<LogCache::PrepareAppendResult> LogCache::PrepareAppendOperations(const ReplicateMsgs& msgs) {
  // SpaceUsed is relatively expensive, so do calculations outside the lock
  PrepareAppendResult result;
  std::vector<CacheEntry> entries_to_insert;
  entries_to_insert.reserve(msgs.size());
  
  # 遍历ReplicateMsgs中的每一个ReplicateMsg，并为之生成一个CacheEntry，用于添加到LogCache中
  for (const auto& msg : msgs) {
    CacheEntry e = { msg, static_cast<int64_t>(msg->SpaceUsedLong()) };
    result.mem_required += e.mem_usage;
    entries_to_insert.emplace_back(std::move(e));
  }

  # ReplicateMsgs中最小的OpId index
  int64_t first_idx_in_batch = msgs.front()->id().index();
  result.last_idx_in_batch = msgs.back()->id().index();

  std::unique_lock<simple_spinlock> lock(lock_);
  # 期待的下一个被添加的ReplicateMsg对应的OpId index是next_sequential_op_index_，
  # 但是这一批ReplicateMsgs中最小的OpId index不是next_sequential_op_index_，则
  # 很有可能是想要替换某些已经存在于LogCache中的ReplicateMsg
  if (first_idx_in_batch != next_sequential_op_index_) {
    # 断言：first_idx_in_batch必须比next_sequential_op_index_小
    CHECK_LE(first_idx_in_batch, next_sequential_op_index_);

    # 移除LogCache中从first_idx_in_batch开始的所有的ReplicateMsg
    for (int64_t i = first_idx_in_batch; i < next_sequential_op_index_; ++i) {
      auto it = cache_.find(i);
      if (it != cache_.end()) {
        AccountForMessageRemovalUnlocked(it->second);
        cache_.erase(it);
      }
    }
  }

  # 将新的ReplicateMsgs添加到LogCache中，并更新next_sequential_op_index_
  for (auto& e : entries_to_insert) {
    auto index = e.msg->id().index();
    EmplaceOrDie(&cache_, index, std::move(e));
    next_sequential_op_index_ = index + 1;
  }

  return result;
}
```

**LogCache::LogCallback**
> 通知LogCache::AppendOperations中写入的一批ReplicateMsgs写入结果(写入失败，或者写入成功且已经完成sync)
```
void LogCache::LogCallback(int64_t last_idx_in_batch,
                           const StatusCallback& user_callback,
                           const Status& log_status) {
  if (log_status.ok()) {
    # 在写入成功且已经sync的情况下，更新min_pinned_op_index_，表示小于
    # min_pinned_op_index_的任何ReplicateMsg都可以从LogCache中剔除了
    std::lock_guard<simple_spinlock> l(lock_);
    if (min_pinned_op_index_ <= last_idx_in_batch) {
      min_pinned_op_index_ = last_idx_in_batch + 1;
    }
  }
  
  # 然后调用用户自定义的回调
  user_callback.Run(log_status);
}
```

**LogCache::LookupOpId**
> 给定一个OpId index，查找对应的OpId
```
Result<yb::OpId> LogCache::LookupOpId(int64_t op_index) const {
  
  {
    # 首先从LogCache中查找
    std::lock_guard<simple_spinlock> l(lock_);

    auto iter = cache_.find(op_index);
    if (iter != cache_.end()) {
      return yb::OpId::FromPB(iter->second.msg->id());
    }
  }

  # 如果没有从LogCache中找到，则从log segment中查找
  return log_->GetLogReader()->LookupOpId(op_index);
}
```

**LogCache::ReadOps**
> 从LogCache或者log segment中读取所有OpId index介于(after_op_index, to_op_index]范围内的所有的ReplicateMsg
> 读取到的所有ReplicateMsg的总大小不超过max_size_bytes
```
Result<ReadOpsResult> LogCache::ReadOps(int64_t after_op_index,
                                        int64_t to_op_index,
                                        int max_size_bytes) {
  ReadOpsResult result;
  result.preceding_op = VERIFY_RESULT(LookupOpId(after_op_index));

  std::unique_lock<simple_spinlock> l(lock_);
  # 确定查找范围：从next_index开始，到to_index为止
  int64_t next_index = after_op_index + 1;
  int64_t to_index = to_op_index > 0
      ? std::min(to_op_index + 1, next_sequential_op_index_)
      : next_sequential_op_index_;

  # 至多返回max_size_bytes大小的内容
  int64_t remaining_space = max_size_bytes;
  
  while (remaining_space > 0 && next_index < to_index) {
    # 先从LogCache中查找第一个不小于next_index的CacheEntry
    MessageCache::const_iterator iter = cache_.lower_bound(next_index);
    if (iter == cache_.end() || iter->first != next_index) {
      int64_t up_to;
      if (iter == cache_.end()) {
        # 不存在不小于next_index的CacheEntry，也就是说LogCache中的CacheEntry
        # 都是小于next_index的，所以要去log segment中读取
        up_to = to_index - 1;
      } else {
        # iter->first > next_index的情况下，需要去log segment中读取从next_index
        # 开始，到up_to截止的所有的index
        up_to = std::min(iter->first - 1, static_cast<uint64_t>(to_index - 1));
      }

      l.unlock();

      # 从log segment中读取，读取到的ReplicateMsg保存在raw_replicate_ptrs
      ReplicateMsgs raw_replicate_ptrs;
      RETURN_NOT_OK_PREPEND(
        log_->GetLogReader()->ReadReplicatesInRange(
            next_index, up_to, remaining_space, &raw_replicate_ptrs),
        Substitute("Failed to read ops $0..$1", next_index, up_to));
        
      l.lock();

      # 逐一处理raw_replicate_ptrs中的每一个ReplicateMsg，添加到result.messages中
      for (auto& msg : raw_replicate_ptrs) {
        CHECK_EQ(next_index, msg->id().index());

        auto current_message_size = TotalByteSizeForMessage(*msg);
        remaining_space -= current_message_size;
        if (remaining_space >= 0 || result.messages.empty()) {
          result.messages.push_back(msg);
          result.read_from_disk_size += current_message_size;
          next_index++;
        } else {
          result.have_more_messages = true;
        }
      }
    } else {
      # 在LogCache中找到了next_index对应的CacheEntry，则从LogCache中读取连续的
      # ReplicateMsg，并保存在result.messages中
      for (; iter != cache_.end(); ++iter) {
        if (to_op_index > 0 && next_index > to_op_index) {
          break;
        }
        const ReplicateMsgPtr& msg = iter->second.msg;
        int64_t index = msg->id().index();
        if (index != next_index) {
          continue;
        }

        auto current_message_size = TotalByteSizeForMessage(*msg);
        remaining_space -= current_message_size;
        if (remaining_space < 0 && !result.messages.empty()) {
          result.have_more_messages = true;
          break;
        }

        result.messages.push_back(msg);
        next_index++;
      }
    }
  }

  return result;
}
```

**LogCache::EvictThroughOp**
> 
```
size_t LogCache::EvictThroughOp(int64_t index, int64_t bytes_to_evict) {
  std::lock_guard<simple_spinlock> lock(lock_);
  return EvictSomeUnlocked(index, bytes_to_evict);
}

size_t LogCache::EvictSomeUnlocked(int64_t stop_after_index, int64_t bytes_to_evict) {
  DCHECK(lock_.is_locked());
  int64_t bytes_evicted = 0;
  for (auto iter = cache_.begin(); iter != cache_.end();) {
    const CacheEntry& entry = iter->second;
    const ReplicateMsgPtr& msg = entry.msg;
    VLOG_WITH_PREFIX_UNLOCKED(2) << "considering for eviction: " << msg->id();
    int64_t msg_index = msg->id().index();
    if (msg_index == 0) {
      // Always keep our special '0' op.
      ++iter;
      continue;
    }

    # 当msg_index大于stop_after_index时停止evict更多的CacheEntry，同时不能evict
    # 不小于min_pinned_op_index_的任何index对应的CacheEntry
    if (msg_index > stop_after_index || msg_index >= min_pinned_op_index_) {
      break;
    }

    bytes_evicted += entry.mem_usage;
    
    # 从LogCache中移除iter所指向的CacheEntry
    cache_.erase(iter++);

    # 最多evict bytes_evicted这么多字节的数据
    if (bytes_evicted >= bytes_to_evict) {
      break;
    }
  }

  return bytes_evicted;
}
```

# LogReader
## 综述
LogReader主要用于从log segment中读取已经写入的ReplicateMsg。比如在向Raft follower发送ReplicateMsg的时候，这些ReplicateMsg如果没有在LogCache中命中，就需要借助于LogReader去log segment中读取。

## LogReader定义
```
class LogReader {
  # 对不同平台上文件相关的操作的封装
  Env *env_;

  # 索引信息，用于快速定位一个ReplicateMsg对应的OpId index在哪个log segment中，
  # 以及在log segment中的哪个偏移的位置
  const scoped_refptr<LogIndex> log_index_;
  
  # 这个LogReader是关于哪个Tablet的
  const std::string tablet_id_;
  const std::string log_prefix_;

  # 当前所有可读的log segment是的集合，SegmentSequence是ReadableLogSegment类型的数组
  SegmentSequence segments_;

  # 用于保护关于segments_的并发操作
  mutable simple_spinlock lock_;

  # LogReader的状态，
  State state_;
}
```

## 主要函数
**LogReader::Init**
> 根据tablet WAL目录下的所有的wal log文件重建LogReader::segments_。
```
Log::Init
LogReader::Open
LogReader::Init

Status LogReader::Init(const string& tablet_wal_path) {
  ...
  
  # 读取给定的WAL目录下的所有的文件，保存在files_from_log_directory中
  std::vector<string> files_from_log_directory;
  RETURN_NOT_OK_PREPEND(env_->GetChildren(tablet_wal_path, &files_from_log_directory),
                        "Unable to read children from path");

  SegmentSequence read_segments;

  # 逐一处理files_from_log_directory中的所有的wal log文件，忽略不是wal log的文件
  for (const string &potential_log_file : files_from_log_directory) {
    if (!IsLogFileName(potential_log_file)) {
      # 忽略不是wal log文件的情况
      continue;
    }

    # 获取当前的wal log文件的全路径名
    string fqp = JoinPathSegments(tablet_wal_path, potential_log_file);
    
    # 基于当前的wal log，建立ReadableLogSegment
    scoped_refptr<ReadableLogSegment> segment;
    RETURN_NOT_OK_PREPEND(ReadableLogSegment::Open(env_, fqp, &segment),
                          Format("Unable to open readable log segment: $0", fqp));
    DCHECK(segment);
    CHECK(segment->IsInitialized()) << "Uninitialized segment at: " << segment->path();

    if (!segment->HasFooter()) {
      # 当前wal log文件没有footer信息，表明这个wal log文件正在写入过程中系统发生了crash，
      # 通过扫描该wal log文件来重建footer信息
      LOG_WITH_PREFIX(WARNING)
          << "Log segment " << fqp << " was likely left in-progress "
             "after a previous crash. Will try to rebuild footer by scanning data.";
      RETURN_NOT_OK(segment->RebuildFooterByScanning());
    }

    # 添加到read_segments集合
    read_segments.push_back(segment);
  }

  # 对read_segments集合中的所有的ReadableLogSegment按照segment sequence number排序
  std::sort(read_segments.begin(), read_segments.end(), LogSegmentSeqnoComparator());

  {
    std::lock_guard<simple_spinlock> lock(lock_);

    string previous_seg_path;
    int64_t previous_seg_seqno = -1;
    
    # 逐一处理read_segments中的每个ReadableLogSegment
    for (const SegmentSequence::value_type& entry : read_segments) {
      VLOG_WITH_PREFIX(1) << " Log Reader Indexed: " << entry->footer().ShortDebugString();
      // Check that the log segments are in sequence.
      if (previous_seg_seqno != -1 && entry->header().sequence_number() != previous_seg_seqno + 1) {
        # WAL log segment sequence number应该是连续的，不连续的情况下提示错误，并返回
        return STATUS(Corruption, Substitute("Segment sequence numbers are not consecutive. "
            "Previous segment: seqno $0, path $1; Current segment: seqno $2, path $3",
            previous_seg_seqno, previous_seg_path,
            entry->header().sequence_number(), entry->path()));
        previous_seg_seqno++;
      } else {
        previous_seg_seqno = entry->header().sequence_number();
      }
      previous_seg_path = entry->path();
      
      # 将当前的ReadableLogSegment添加到LogReader::segments_中
      RETURN_NOT_OK(AppendSegmentUnlocked(entry));
    }

    state_ = kLogReaderReading;
  }
  
  return Status::OK();
}
```

**LogReader::GetSegmentPrefixNotIncluding**
> 获取给定index之前所有可以被回收的log segments，这些log segment必须满足：不是当前正在被写入的最新的log segment；且log segment中所有的OpId index比给定的index小。
```
Log::GC
Log::GetSegmentsToGCUnlocked
LogReader::GetSegmentPrefixNotIncluding

Status LogReader::GetSegmentPrefixNotIncluding(int64_t index, int64_t cdc_max_replicated_index,
                                               SegmentSequence* segments) const {
  DCHECK_GE(index, 0);
  DCHECK(segments);
  segments->clear();

  std::lock_guard<simple_spinlock> lock(lock_);
  CHECK_EQ(state_, kLogReaderReading);

  # 遍历LogReader::segments_中的所有的ReadableLogSegment，获取可以被回收的ReadableLogSegment集合
  int64_t reclaimed_space = 0;
  for (const scoped_refptr<ReadableLogSegment>& segment : segments_) {
    # 如果当前ReadableLogSegment没有footer，则该ReadableLogSegment一定是当前正在写入的
    # log segment，它不可以被回收
    if (!segment->HasFooter()) {
      break;
    }

    # 只回收所有OpId index比给定index小的log segment(如果log segment中最大的OpId index
    # 比给定的index小，则该log segment中所有的OpId index一定比给定的index小)
    if (segment->footer().max_replicate_index() >= index) {
      break;
    }

    # 主要用于cdc场景，暂不考虑
    if (FLAGS_enable_log_retention_by_op_idx &&
        segment->footer().max_replicate_index() >= cdc_max_replicated_index) {
        ...
    }

    # 将当前的log segment添加到待回收的segments集合中
    segments->push_back(segment);
  }

  return Status::OK();
}
```

**LogReader::GetMinReplicateIndex**
> 获取所有已经结束写入的log segment中最小的OpId index。
```
int64_t LogReader::GetMinReplicateIndex() const {
  std::lock_guard<simple_spinlock> lock(lock_);
  int64_t min_remaining_op_idx = -1;

  for (const scoped_refptr<ReadableLogSegment>& segment : segments_) {
    if (!segment->HasFooter()) continue;
    if (!segment->footer().has_min_replicate_index()) continue;
    if (min_remaining_op_idx == -1 ||
        segment->footer().min_replicate_index() < min_remaining_op_idx) {
      min_remaining_op_idx = segment->footer().min_replicate_index();
    }
  }
  return min_remaining_op_idx;
}
```

**LogReader::GetMaxIndexesToSegmentSizeMap**
> 获取每个log segment中最大log index到该log segment文件大小的映射
> 返回的映射表中的entry对应的log segment必须满足：该log segment中最大的log index不小于参数min_op_idx，且该log segment的关闭时间不大于参数max_close_time_us
> 返回的映射表中至多包含参数segments_count个元素
> 返回的映射表保存在参数max_idx_to_segment_size中
```
void LogReader::GetMaxIndexesToSegmentSizeMap(int64_t min_op_idx, int32_t segments_count,
                                              int64_t max_close_time_us,
                                              std::map<int64_t, int64_t>*
                                              max_idx_to_segment_size) const {
  std::lock_guard<simple_spinlock> lock(lock_);
  DCHECK_GE(segments_count, 0);
  for (const scoped_refptr<ReadableLogSegment>& segment : segments_) {
    # 已经达到segments_count个元素
    if (max_idx_to_segment_size->size() == segments_count) {
      break;
    }
    
    # 当前log segment的最大的log index必须不小于参数min_op_idx
    DCHECK(segment->HasFooter());
    if (segment->footer().max_replicate_index() < min_op_idx) {
      // This means we found a log we can GC. Adjust the expected number of logs.
      segments_count--;
      continue;
    }

    # 当前的log segment的关闭时间必须大于参数max_close_time_us
    if (max_close_time_us < segment->footer().close_timestamp_micros()) {
      break;
    }
    
    # 添加到max_idx_to_segment_size中
    (*max_idx_to_segment_size)[segment->footer().max_replicate_index()] = segment->file_size();
  }
}
```

**LogReader::GetSegmentBySequenceNumber**
> 获取给定的log segment sequence number 对应的log segment
```
scoped_refptr<ReadableLogSegment> LogReader::GetSegmentBySequenceNumber(int64_t seq) const {
  std::lock_guard<simple_spinlock> lock(lock_);
  if (segments_.empty()) {
    return nullptr;
  }

  // We always have a contiguous set of log segments, so we can find the requested
  // segment in our vector by calculating its offset vs the first element.
  int64_t first_seqno = segments_[0]->header().sequence_number();
  int64_t relative = seq - first_seqno;
  if (relative < 0 || relative >= segments_.size()) {
    return nullptr;
  }

  DCHECK_EQ(segments_[relative]->header().sequence_number(), seq);
  return segments_[relative];
}
```

**LogReader::ReadBatchUsingIndexEntry**
> 读取由给定LogIndexEntry指示的LogEntryBatchPB。
```
Status LogReader::ReadBatchUsingIndexEntry(const LogIndexEntry& index_entry,
                                           faststring* tmp_buf,
                                           LogEntryBatchPB* batch) const {
  const int64_t index = index_entry.op_id.index;

  # 获取index_entry.segment_sequence_number对应的log segment
  scoped_refptr<ReadableLogSegment> segment = GetSegmentBySequenceNumber(
    index_entry.segment_sequence_number);

  # 从log segment中index_entry.offset_in_segment所指示的偏移处读取，读取的数据
  # 保存在LogEntryBatchPB中
  CHECK_GT(index_entry.offset_in_segment, 0);
  int64_t offset = index_entry.offset_in_segment;
  ScopedLatencyMetric scoped(read_batch_latency_.get());
  RETURN_NOT_OK_PREPEND(segment->ReadEntryHeaderAndBatch(&offset, tmp_buf, batch),
                        Substitute("Failed to read LogEntry for index $0 from log segment "
                                   "$1 offset $2",
                                   index,
                                   index_entry.segment_sequence_number,
                                   index_entry.offset_in_segment));

  return Status::OK();
}
```

**LogReader::ReadReplicatesInRange**
> 读取OpId index从starting_at开始到up_to结束的所有的ReplicateMsg，并保存在replicates中
> 读取到的ReplicateMsg的总的字节数不超过参数max_bytes_to_read
```
Status LogReader::ReadReplicatesInRange(
    const int64_t starting_at,
    const int64_t up_to,
    int64_t max_bytes_to_read,
    ReplicateMsgs* replicates) const {
  ReplicateMsgs replicates_tmp;
  LogIndexEntry prev_index_entry;
  prev_index_entry.segment_sequence_number = -1;
  prev_index_entry.offset_in_segment = -1;

  int64_t total_size = 0;
  bool limit_exceeded = false;
  faststring tmp_buf;
  LogEntryBatchPB batch;
  
  for (int64_t index = starting_at; index <= up_to && !limit_exceeded; index++) {
    LogIndexEntry index_entry;
    # 从LogIndex中查找index所对应的索引项信息，保存在LogIndexEntry中
    RETURN_NOT_OK_PREPEND(log_index_->GetEntry(index, &index_entry),
                          Substitute("Failed to read log index for op $0", index));

    # 一个LogEntryBatchPB中可能会包含多个ReplicateMsg，每个ReplicateMsg都具有不同的OpId
    # index，所以可能存在多个相邻的index对应同一个LogEntryBatchPB的情况
    if (index == starting_at ||
        index_entry.segment_sequence_number != prev_index_entry.segment_sequence_number ||
        index_entry.offset_in_segment != prev_index_entry.offset_in_segment) {
      # 读取由index_entry所指示的LogEntryBatchPB
      RETURN_NOT_OK(ReadBatchUsingIndexEntry(index_entry, &tmp_buf, &batch));
    }

    bool found = false;
    # 处理LogEntryBatchPB中当前index对应的LogEntryPB
    for (int i = 0; i < batch.entry_size(); ++i) {
      LogEntryPB* entry = batch.mutable_entry(i);
      if (!entry->has_replicate()) {
        continue;
      }

      if (entry->replicate().id().index() != index) {
        continue;
      }

      # 检查是否超过了max_bytes_to_read的限制
      int64_t space_required = entry->replicate().SpaceUsed();
      if (replicates_tmp.empty() ||
          max_bytes_to_read <= 0 ||
          total_size + space_required < max_bytes_to_read) {
        total_size += space_required;
        # 将当前index对应的ReplicateMsg添加到replicates_tmp中
        replicates_tmp.emplace_back(entry->release_replicate());
      } else {
        limit_exceeded = true;
      }
      
      # 一旦找到了当前index对应的LogEntryPB，就退出当前的for循环，处理下一个index
      found = true;
      break;
    }

    prev_index_entry = index_entry;
  }

  replicates->swap(replicates_tmp);
  return Status::OK();
}
```

**LogReader::LookupOpId**
> 从LogIndex中获取给定OpId index对应的OpId(其中包括OpId index和OpId term)
```
Result<yb::OpId> LogReader::LookupOpId(int64_t op_index) const {
  LogIndexEntry index_entry;
  RETURN_NOT_OK_PREPEND(log_index_->GetEntry(op_index, &index_entry),
                        strings::Substitute("Failed to read log index for op $0", op_index));
  return index_entry.op_id;
}
```

**LogReader::UpdateLastSegmentOffset**
> 更新LogReader::segments_中最后一个log segment(最后一个log segment在没有关闭之前，仍然处于可写状态)中可读的offset信息。
```
void LogReader::UpdateLastSegmentOffset(int64_t readable_to_offset) {
  std::lock_guard<simple_spinlock> lock(lock_);
  CHECK_EQ(state_, kLogReaderReading);
  DCHECK(!segments_.empty());
  // Get the last segment
  ReadableLogSegment* segment = segments_.back().get();
  DCHECK(!segment->HasFooter());
  segment->UpdateReadableToOffset(readable_to_offset);
}
```

**LogReader::ReplaceLastSegment**
> 当LogReader::segments_中最后一个log segment被正常关闭之后，替换之前记录在LogReader::segments_中的最后一个log segment。这替换之前的最后一个log segment和替换之后的最后一个log segment其实是同一个log segment，只不过替换之前的最后一个log segment处于可写状态，且没有写入footer信息，替换之后的最后一个log segment已经关闭了，不可再写了，且写入了footer信息。
```
Status LogReader::ReplaceLastSegment(const scoped_refptr<ReadableLogSegment>& segment) {
  // This is used to replace the last segment once we close it properly so it must
  // have a footer.
  DCHECK(segment->HasFooter());

  std::lock_guard<simple_spinlock> lock(lock_);
  CHECK_EQ(state_, kLogReaderReading);
  // Make sure the segment we're replacing has the same sequence number
  CHECK(!segments_.empty());
  CHECK_EQ(segment->header().sequence_number(), segments_.back()->header().sequence_number());
  segments_[segments_.size() - 1] = segment;

  return Status::OK();
}
```

# LogIndex
## 综述
LogIndex主要用于从一个ReplicateMsg对应的OpId index(ReplicateMsg::id_::index_)找到它在WAL log中的位置(保存在哪个log segment中，以及log segment中的偏移)。

LogIndex又进一步分为一系列的IndexChunk，每个IndexChunk都对应一个mmap文件，其中存放了固定数目(kEntriesPerIndexChunk)个索引项。每个索引项在内存中的结构是LogIndexEntry，而在IndexChunk中保存的硬盘结构是PhysicalEntry，一个LogIndexEntry对应一个PhysicalEntry。

## 主要数据结构
**LogIndexEntry**
> LogIndexEntry表示一个索引项在内存中的结构。
```
struct LogIndexEntry {
  # ReplicateMsg对应的OpId，OpId中包含term(int64)和index(int64)两个成员
  yb::OpId op_id;
  
  # 当前索引项对应的ReplicateMsg保存在哪个log segment中
  int64_t segment_sequence_number;

  # 当前索引项对应的ReplicateMsg在log segment中的偏移
  int64_t offset_in_segment;
};
```

**PhysicalEntry**
> PhysicalEntry表示一个索引项在mmap文件中保存的内容，它和LogIndexEntry一一对应。
```
struct PhysicalEntry {
  # 对应于LogIndexEntry::op_id::term
  int64_t term;
  
  # 对应于LogIndexEntry::segment_sequence_number
  uint64_t segment_sequence_number;
  
   # 对应于LogIndexEntry::offset_in_segment
  uint64_t offset_in_segment;
} PACKED;
```

**IndexChunk**
> IndexChunk表示LogIndex中的一部分，它对应一个mmap文件，其中存放固定数目的indexes。
```
class LogIndex::IndexChunk : public RefCountedThreadSafe<LogIndex::IndexChunk> {
 private:
  # IndexChunk对应的文件名
  const string path_;
  
  # IndexChunk对应的文件句柄
  int fd_;
  
  # IndexChunk对应的文件映射的内存
  uint8_t* mapping_;
};
```

**LogIndex**
> LogIndex用于保存一个ReplicateMsg对应的OpId index(ReplicateMsg::id_::index_)到它在WAL log中的位置的映射关系。
```
class LogIndex : public RefCountedThreadSafe<LogIndex> {
 private:
  # 保存IndexChunk文件的目录
  const std::string base_dir_;

  # 保护IndexChunk的共享访问
  simple_spinlock open_chunks_lock_;

  // Protected by open_chunks_lock_
  typedef std::map<int64_t, scoped_refptr<IndexChunk> > ChunkMap;
  
  # IndexChunk index到IndexChunk的映射
  ChunkMap open_chunks_;
};
```

## IndexChunk相关的主要函数
**LogIndex::IndexChunk::Open**
> 创建IndexChunk对应的文件，然后将该文件truncate为指定大小，最后建立文件和内存的mmap映射。
```
Status LogIndex::IndexChunk::Open() {
  # 打开IndexChunk对应的文件，返回文件句柄
  RETRY_ON_EINTR(fd_, open(path_.c_str(), O_CLOEXEC | O_CREAT | O_RDWR, 0666));
  RETURN_NOT_OK(CheckError(fd_, "open"));

  int err;
  # 将IndexChunk对应的文件truncate为指定大小(kChunkFileSize)
  RETRY_ON_EINTR(err, ftruncate(fd_, kChunkFileSize));
  RETURN_NOT_OK(CheckError(fd_, "truncate"));

  # 建立文件和内存的mmap映射，返回映射到的内存地址
  mapping_ = static_cast<uint8_t*>(mmap(nullptr, kChunkFileSize, PROT_READ | PROT_WRITE,
                                        MAP_SHARED, fd_, 0));
  if (mapping_ == nullptr) {
    return STATUS(IOError, "Unable to mmap()", Errno(err));
  }

  return Status::OK();
}
```

**LogIndex::IndexChunk::GetEntry**
> 在IndexChunk中读取给定entry_index对应的索引信息，保存在PhysicalEntry中。
```
void LogIndex::IndexChunk::GetEntry(int entry_index, PhysicalEntry* ret) {
  DCHECK_GE(fd_, 0) << "Must Open() first";
  DCHECK_LT(entry_index, kEntriesPerIndexChunk);

  memcpy(ret, mapping_ + sizeof(PhysicalEntry) * entry_index, sizeof(PhysicalEntry));
}
```

**LogIndex::IndexChunk::SetEntry**
> 将给定entry_index对应的索引信息写入IndexChunk中。
```
void LogIndex::IndexChunk::SetEntry(int entry_index, const PhysicalEntry& phys) {
  DCHECK_GE(fd_, 0) << "Must Open() first";
  DCHECK_LT(entry_index, kEntriesPerIndexChunk);

  memcpy(mapping_ + sizeof(PhysicalEntry) * entry_index, &phys, sizeof(PhysicalEntry));
}
```

**LogIndex::IndexChunk::Flush**
> 将内存中保存的内容flush到文件中，使二者同步。
```
Status LogIndex::IndexChunk::Flush() {
  if (mapping_ != nullptr) {
    auto result = msync(mapping_, kChunkFileSize, MS_SYNC);
    return CheckError(result, "msync");
  }
  return Status::OK();
}
```

### LogIndex相关的主要函数
**LogIndex::GetChunkPath**
> 通过给定的Chunk index，获取对应的IndexChunk的文件名。
```
string LogIndex::GetChunkPath(int64_t chunk_idx) {
  return StringPrintf("%s/index.%09" PRId64, base_dir_.c_str(), chunk_idx);
}
```

**LogIndex::OpenChunk**
> 打开一个新的IndexChunk，包括创建对应的文件，将文件truncate为指定大小，建立文件和内存的mmap映射等。
```
Status LogIndex::OpenChunk(int64_t chunk_idx, scoped_refptr<IndexChunk>* chunk) {
  # 获取给定chunk_idx对应的文件名
  string path = GetChunkPath(chunk_idx);

  # 创建一个新的IndexChunk
  scoped_refptr<IndexChunk> new_chunk(new IndexChunk(path));
  
  # 打开新的IndexChunk
  RETURN_NOT_OK(new_chunk->Open());
  
  chunk->swap(new_chunk);
  return Status::OK();
}
```

**LogIndex::GetChunkForIndex**
> 给定一个log index，查找它对应的索引项保存在哪个IndexChunk中。因为每个IndexChunk中保存固定数目(kEntriesPerIndexChunk)的索引项，通过给定的log index对kEntriesPerIndexChunk取模即可得到IndexChunk对应的chunk index，然后根据该chunk index去LogIndex::open_chunks_中查找对应的IndexChunk即可。
```
Status LogIndex::GetChunkForIndex(int64_t log_index, bool create,
                                  scoped_refptr<IndexChunk>* chunk) {
  # 计算给定log_index对应的IndexChunk的chunk index
  int64_t chunk_idx = log_index / kEntriesPerIndexChunk;

  {
    # 在LogIndex::open_chunks_中查找给定chunk_idx的IndexChunk，如果找到，则IndexChunk
    # 被保存在chunk中
    std::lock_guard<simple_spinlock> l(open_chunks_lock_);
    if (FindCopy(open_chunks_, chunk_idx, chunk)) {
      return Status::OK();
    }
  }

  # 如果没有找到IndexChunk，且不创建的情况下，直接返回错误提示
  if (!create) {
    return STATUS(NotFound, "chunk not found");
  }

  # 没有找到IndexChunk的情况下，创建一个新的IndexChunk
  RETURN_NOT_OK_PREPEND(OpenChunk(chunk_idx, chunk),
                        "Couldn't open index chunk");
  {
    # 在将新创建的IndexChunk插入LogIndex::open_chunks_之前，先查找LogIndex::open_chunks_
    # 中是否已经存在相同chunk index的IndexChunk，如果存在，则不插入，否则插入
    
    std::lock_guard<simple_spinlock> l(open_chunks_lock_);
    if (PREDICT_FALSE(ContainsKey(open_chunks_, chunk_idx))) {
      *chunk = FindOrDie(open_chunks_, chunk_idx);
      return Status::OK();
    }

    InsertOrDie(&open_chunks_, chunk_idx, *chunk);
  }

  return Status::OK();
}
```

**LogIndex::AddEntry**
> 添加一个由LogIndexEntry指示的索引项到LogIndex中。先通过LogIndexEntry::op_id::index找到对应的IndexChunk，然后将LogIndexEntry转换为PhysicalEntry，最后写入查找到的IndexChunk中。
```
Status LogIndex::AddEntry(const LogIndexEntry& entry) {
  scoped_refptr<IndexChunk> chunk;
  RETURN_NOT_OK(GetChunkForIndex(entry.op_id.index,
                                 true /* create if not found */,
                                 &chunk));
  int index_in_chunk = entry.op_id.index % kEntriesPerIndexChunk;

  PhysicalEntry phys;
  phys.term = entry.op_id.term;
  phys.segment_sequence_number = entry.segment_sequence_number;
  phys.offset_in_segment = entry.offset_in_segment;

  chunk->SetEntry(index_in_chunk, phys);
  VLOG(3) << "Added log index entry " << entry.ToString();

  return Status::OK();
}
```

**LogIndex::GetEntry**
> 从LogIndex中读取给定log index对应的索引项，保存在LogIndexEntry中。
```
Status LogIndex::GetEntry(int64_t index, LogIndexEntry* entry) {
  scoped_refptr<IndexChunk> chunk;
  # 查找对应的IndexChunk
  RETURN_NOT_OK(GetChunkForIndex(index, false /* do not create */, &chunk));
  
  # 计算给定log index在查找到的IndexChunk中的偏移
  int index_in_chunk = index % kEntriesPerIndexChunk;
  
  # 从查找到的IndexChunk中读取给定log index对应的PhysicalEntry
  PhysicalEntry phys;
  chunk->GetEntry(index_in_chunk, &phys);

  # 将PhysicalEntry转换为LogIndexEntry
  entry->op_id = yb::OpId(phys.term, index);
  entry->segment_sequence_number = phys.segment_sequence_number;
  entry->offset_in_segment = phys.offset_in_segment;

  return Status::OK();
}
```

**LogIndex::GC**
> 回收min_index_to_retain代表的log index所在的IndexChunk之前的所有的IndexChunk中保存的索引项，实现LogIndex的回收。
```
void LogIndex::GC(int64_t min_index_to_retain) {
  # 计算出min_index_to_retain所在的IndexChunk对应的chunk index，保存在min_chunk_to_retain中
  int min_chunk_to_retain = min_index_to_retain / kEntriesPerIndexChunk;

  # 查找LogIndex::open_chunks_中所有小于min_index_to_retain的IndexChunk，添加到
  # chunks_to_delete中
  vector<int64_t> chunks_to_delete;
  {
    std::lock_guard<simple_spinlock> l(open_chunks_lock_);
    # open_chunks_.lower_bound(min_chunk_to_retain)返回第一个chunk index不小于
    # min_chunk_to_retain的IndexChunk的chunk index
    for (auto it = open_chunks_.begin();
         it != open_chunks_.lower_bound(min_chunk_to_retain); ++it) {
      chunks_to_delete.push_back(it->first);
    }
  }

  # 对chunks_to_delete中的所有的IndexChunk执行删除
  for (int64_t chunk_idx : chunks_to_delete) {
    # 获取IndexChunk对应的文件名
    string path = GetChunkPath(chunk_idx);
    # 删除该文件
    int rc = unlink(path.c_str());
    {
      std::lock_guard<simple_spinlock> l(open_chunks_lock_);
      # 从LogIndex::open_chunks_中移除，当成功从LogIndex::open_chunks_中移除
      # 之后，IndexChunk将被析构，在析构函数中会调用munmap解除内存映射，并调
      # 用close关闭文件句柄
      open_chunks_.erase(chunk_idx);
    }
  }
}
```

**LogIndex::Flush**
> 将LogIndex::open_chunks_中所有的IndexChunk进行Flush，使得IndexChunk在内存中的内容和文件中的内容一致。
```
Status LogIndex::Flush() {
  std::vector<scoped_refptr<IndexChunk>> chunks_to_flush;

  {
    std::lock_guard<simple_spinlock> l(open_chunks_lock_);
    chunks_to_flush.reserve(open_chunks_.size());
    for (auto& it : open_chunks_) {
      chunks_to_flush.push_back(it.second);
    }
  }

  # 逐一遍历每个IndexChunk，并调用IndexChunk::Flush完成flush工作
  for (auto& chunk : chunks_to_flush) {
    RETURN_NOT_OK(chunk->Flush());
  }

  return Status::OK();
}
```


