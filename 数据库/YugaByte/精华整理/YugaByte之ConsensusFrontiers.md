# 提纲
[toc]

## ConsensusFrontiers的定义
```
typedef rocksdb::UserFrontiersBase<ConsensusFrontier> ConsensusFrontiers;

template<class Frontier>
class UserFrontiersBase : public rocksdb::UserFrontiers {
 private:
  Frontier smallest_;
  Frontier largest_;
};

class ConsensusFrontier : public rocksdb::UserFrontier {
 private:
  OpId op_id_;
  HybridTime ht_;

  // We use this to keep track of the maximum history cutoff hybrid time used in any compaction, and
  // refuse to perform reads at a hybrid time at which we don't have a valid snapshot anymore. Only
  // the largest frontier of this parameter is being used.
  HybridTime history_cutoff_;

  HybridTime hybrid_time_filter_;
};
```
从上面的定义可以看出，ConsensusFrontiers中记录了关于ConsensusFrontier::op_id_, ConsensusFrontier::ht_, ConsensusFrontier::history_cutoff_和ConsensusFrontier::hybrid_time_filter_的最小值和最大值。

## ConsensusFrontier::ht_
### ConsensusFrontier::ht_的设置
在每次将Write操作写入RocksDB的过程中，会使用该Write操作对应的hybrid_time_来更新docdb::ConsensusFrontiers，并将docdb::ConsensusFrontiers写入对应的MemTable::frontiers_中。
```
OperationDriver::ApplyTask
Operation::Replicated
WriteOperation::DoReplicated
Tablet::ApplyRowOperations
Tablet::ApplyOperationState

Status Tablet::ApplyOperationState(
    const OperationState& operation_state, int64_t batch_idx,
    const docdb::KeyValueWriteBatchPB& write_batch) {
  docdb::ConsensusFrontiers frontiers;
  # 采用operation_state.op_id()设置docdb::ConsensusFrontiers::op_id_
  set_op_id(yb::OpId::FromPB(operation_state.op_id()), &frontiers);
  
  # 采用operation_state.hybrid_time()设置docdb::ConsensusFrontiers::ht_
  auto hybrid_time = operation_state.WriteHybridTime();
  set_hybrid_time(operation_state.hybrid_time(), &frontiers);
  
  # 写RocksDB
  return ApplyKeyValueRowOperations(
      batch_idx, write_batch, &frontiers, hybrid_time);
}

Status Tablet::ApplyKeyValueRowOperations(int64_t batch_idx,
                                          const KeyValueWriteBatchPB& put_batch,
                                          const rocksdb::UserFrontiers* frontiers,
                                          const HybridTime hybrid_time) {
  rocksdb::WriteBatch write_batch;
  if (put_batch.has_transaction()) {
    # 准备好write_batch
    RequestScope request_scope(transaction_participant_.get());
    RETURN_NOT_OK(PrepareTransactionWriteBatch(batch_idx, put_batch, hybrid_time, &write_batch));
    # 写RocksDB
    WriteToRocksDB(frontiers, &write_batch, StorageDbType::kIntents);
  } else {
    # 准备好write_batch
    PrepareNonTransactionWriteBatch(put_batch, hybrid_time, &write_batch);
    # 写RocksDB 
    WriteToRocksDB(frontiers, &write_batch, StorageDbType::kRegular);
  }

  return Status::OK();
}

void Tablet::WriteToRocksDB(
    const rocksdb::UserFrontiers* frontiers,
    rocksdb::WriteBatch* write_batch,
    docdb::StorageDbType storage_db_type) {
  rocksdb::DB* dest_db = nullptr;
  switch (storage_db_type) {
    case StorageDbType::kRegular: dest_db = regular_db_.get(); break;
    case StorageDbType::kIntents: dest_db = intents_db_.get(); break;
  }
  
  # 在WriteBatch中设置Frontiers信息
  write_batch->SetFrontiers(frontiers);

  // We are using Raft replication index for the RocksDB sequence number for
  // all members of this write batch.
  rocksdb::WriteOptions write_options;
  InitRocksDBWriteOptions(&write_options);

  # RocksDB对WriteBatch中Frontiers的处理，见“RocksDB对WriteBatch::frontiers_的处理”
  auto rocksdb_write_status = dest_db->Write(write_options, write_batch);
}
```

### ConsensusFrontier::ht_的使用
当TSTabletManager执行Background flushing task的过程中，会从TSTabletManager所管理的所有的tablets中查找具有最旧的写操作的Mutable MemTable对应的tablet，并对该tablet进行flush。
```
TSTabletManager::TSTabletManager {
  ...
  
  if (should_count_memory) {
    # 启动一个Background flushing task
    background_task_.reset(new BackgroundTask(
      std::function<void()>([this](){ MaybeFlushTablet(); }),
      "tablet manager",
      "flush scheduler bgtask",
      std::chrono::milliseconds(FLAGS_flush_background_task_interval_msec)));
    tablet_options_.memory_monitor = std::make_shared<rocksdb::MemoryMonitor>(
        memstore_size_bytes,
        std::function<void()>([this](){
                                YB_WARN_NOT_OK(background_task_->Wake(), "Wakeup error"); }));
  }    
}

TSTabletManager::MaybeFlushTablet() {
  int iteration = 0;
  while (memory_monitor()->Exceeded() ||
         (iteration++ == 0 && FLAGS_TEST_pretend_memory_exceeded_enforce_flush)) {
    # 返回memstore中最旧的写操作对应的tablet，如果memstore是空或者即将flush，则返回nullptr
    TabletPeerPtr tablet_to_flush = TabletToFlush();
    if (tablet_to_flush) {
      WARN_NOT_OK(
          # 调用Tablet::Flush对tablet进行flush
          tablet_to_flush->tablet()->Flush(
              tablet::FlushMode::kAsync, tablet::FlushFlags::kAll, flush_tick),
          Substitute("Flush failed on $0", tablet_to_flush->tablet_id()));
    }
  }
}

TabletPeerPtr TSTabletManager::TabletToFlush() {
  SharedLock<RWMutex> lock(mutex_); // For using the tablet map
  HybridTime oldest_write_in_memstores = HybridTime::kMax;
  TabletPeerPtr tablet_to_flush;
  # 逐一遍历每一个tablet
  for (const TabletMap::value_type& entry : tablet_map_) {
    const auto tablet = entry.second->shared_tablet();
    if (tablet) {
      # 获取该tablet对应的mutable memtable中最旧的写对应的HybridTime
      const auto ht = tablet->OldestMutableMemtableWriteHybridTime();
      if (ht.ok()) {
        # oldest_write_in_memstores维护当前最旧的写操作对应的时间，
        # tablet_to_flush中记录着当前最旧的写操作所涉及的tablet
        if (*ht < oldest_write_in_memstores) {
          oldest_write_in_memstores = *ht;
          tablet_to_flush = entry.second;
        }
      }
    }
  }
  
  return tablet_to_flush;
}

Result<HybridTime> Tablet::OldestMutableMemtableWriteHybridTime() const {
  HybridTime result = HybridTime::kMax;
  
  # 获取regular db和intent db中最旧的mutable memtable write时间
  for (auto* db : { regular_db_.get(), intents_db_.get() }) {
    if (db) {
      auto mem_frontier = db->GetMutableMemTableFrontier(rocksdb::UpdateUserValueType::kSmallest);
      if (mem_frontier) {
        const auto hybrid_time =
            static_cast<const docdb::ConsensusFrontier&>(*mem_frontier).hybrid_time();
        result = std::min(result, hybrid_time);
      }
    }
  }
  
  return result;
}

UserFrontierPtr DBImpl::GetMutableMemTableFrontier(UpdateUserValueType type) {
  InstrumentedMutexLock l(&mutex_);
  UserFrontierPtr accumulated;
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    if (cfd) {
      const auto* mem = cfd->mem();
      if (mem) {
        if (!cfd->IsDropped() && cfd->imm()->NumNotFlushed() == 0 && !mem->IsEmpty()) {
          auto frontier = mem->GetFrontier(type);
          if (frontier) {
            UserFrontier::Update(frontier.get(), type, &accumulated);
          } else {
            ...
          }
        }
      } else {
        ...
      }
    } else {
      ...
    }
  }
  
  return accumulated;
}

UserFrontierPtr MemTable::GetFrontier(UpdateUserValueType type) const {
  std::lock_guard<SpinMutex> l(frontiers_mutex_);
  if (!frontiers_) {
    return nullptr;
  }

  switch (type) {
    case UpdateUserValueType::kSmallest:
      return frontiers_->Smallest().Clone();
    case UpdateUserValueType::kLargest:
      return frontiers_->Largest().Clone();
  }

  FATAL_INVALID_ENUM_VALUE(UpdateUserValueType, type);
}

void UserFrontier::Update(const UserFrontier* rhs, UpdateUserValueType type, UserFrontierPtr* out) {
  if (!rhs) {
    return;
  }
  
  if (*out) {
    # 在当前上下文调用的是ConsensusFrontier::Update
    (**out).Update(*rhs, type);
  } else {
    *out = rhs->Clone();
  }
}

void ConsensusFrontier::Update(
    const rocksdb::UserFrontier& pre_rhs, rocksdb::UpdateUserValueType update_type) {
  const ConsensusFrontier& rhs = down_cast<const ConsensusFrontier&>(pre_rhs);
  UpdateField(&op_id_, rhs.op_id_, update_type);
  UpdateField(&ht_, rhs.ht_, update_type);
  UpdateField(&history_cutoff_, rhs.history_cutoff_, update_type);
  // Reset filter after compaction.
  hybrid_time_filter_ = HybridTime();
}

template<typename T>
void UpdateField(
    T* this_value, const T& new_value, rocksdb::UpdateUserValueType update_type) {
  switch (update_type) {
    case rocksdb::UpdateUserValueType::kLargest:
      # 取this_value和new_value中的较大值
      this_value->MakeAtLeast(new_value);
      return;
    case rocksdb::UpdateUserValueType::kSmallest:
      # 取this_value和new_value中的较小值
      this_value->MakeAtMost(new_value);
      return;
  }
  
  FATAL_INVALID_ENUM_VALUE(rocksdb::UpdateUserValueType, update_type);
}
```

flush的流程如下：
```
# 在当前上下文中，参数@mode被设置为tablet::FlushMode::kAsync，@flags被设置为tablet::FlushFlags::kAll
Status Tablet::Flush(FlushMode mode, FlushFlags flags, int64_t ignore_if_flushed_after_tick) {
  rocksdb::FlushOptions options;
  options.ignore_if_flushed_after_tick = ignore_if_flushed_after_tick;
  bool flush_intents = intents_db_ && HasFlags(flags, FlushFlags::kIntents);
  if (flush_intents) {
    options.wait = false;
    WARN_NOT_OK(intents_db_->Flush(options), "Flush intents DB");
  }

  if (HasFlags(flags, FlushFlags::kRegular) && regular_db_) {
    options.wait = mode == FlushMode::kSync;
    WARN_NOT_OK(regular_db_->Flush(options), "Flush regular DB");
  }

  if (flush_intents && mode == FlushMode::kSync) {
    RETURN_NOT_OK(intents_db_->WaitForFlush());
  }

  return Status::OK();
}

Status DBImpl::Flush(const FlushOptions& flush_options,
                     ColumnFamilyHandle* column_family) {
  auto cfh = down_cast<ColumnFamilyHandleImpl*>(column_family);
  return FlushMemTable(cfh->cfd(), flush_options);
}

Status DBImpl::FlushMemTable(ColumnFamilyData* cfd,
                             const FlushOptions& flush_options) {
  Status s;
  {
    WriteContext context;
    InstrumentedMutexLock guard_lock(&mutex_);

    if (last_flush_at_tick_ > flush_options.ignore_if_flushed_after_tick) {
      return STATUS(AlreadyPresent, "Mem table already flushed");
    }

    # Immutable Memtable中没有需要flush的数据，且Mutable MemTable为空
    if (cfd->imm()->NumNotFlushed() == 0 && cfd->mem()->IsEmpty()) {
      // Nothing to flush
      return Status::OK();
    }

    last_flush_at_tick_ = FlushTick();

    WriteThread::Writer w;
    # 等待，直到
    write_thread_.EnterUnbatched(&w, &mutex_);

    # 将当前的Mutable MemTable转换为Immutable Memtable，同时创建一个新的Mutable MemTable，
    # 新的Mutable MemTable中的frontiers_变为nullptr
    s = SwitchMemtable(cfd, &context);
    write_thread_.ExitUnbatched(&w);

    cfd->imm()->FlushRequested();

    // schedule flush
    SchedulePendingFlush(cfd);
    MaybeScheduleFlushOrCompaction();
  }

  if (s.ok() && flush_options.wait) {
    // Wait until the compaction completes
    s = WaitForFlushMemTable(cfd);
  }
  return s;
}
```

## ConsensusFrontier::op_id_
### ConsensusFrontier::op_id_的设置
在每次将Write操作写入RocksDB的过程中，会使用该Write操作对应的op_id_来更新docdb::ConsensusFrontiers，并将docdb::ConsensusFrontiers写入对应的MemTable::frontiers_中。
```
OperationDriver::ApplyTask
Operation::Replicated
WriteOperation::DoReplicated
Tablet::ApplyRowOperations
Tablet::ApplyOperationState

Status Tablet::ApplyOperationState(
    const OperationState& operation_state, int64_t batch_idx,
    const docdb::KeyValueWriteBatchPB& write_batch) {
  docdb::ConsensusFrontiers frontiers;
  # 采用operation_state.op_id()设置docdb::ConsensusFrontiers
  set_op_id(yb::OpId::FromPB(operation_state.op_id()), &frontiers);
  
  # 采用operation_state.hybrid_time()设置docdb::ConsensusFrontiers
  auto hybrid_time = operation_state.WriteHybridTime();
  set_hybrid_time(operation_state.hybrid_time(), &frontiers);
  
  # 写RocksDB
  return ApplyKeyValueRowOperations(
      batch_idx, write_batch, &frontiers, hybrid_time);
}

Status Tablet::ApplyKeyValueRowOperations(int64_t batch_idx,
                                          const KeyValueWriteBatchPB& put_batch,
                                          const rocksdb::UserFrontiers* frontiers,
                                          const HybridTime hybrid_time) {
  rocksdb::WriteBatch write_batch;
  if (put_batch.has_transaction()) {
    # 准备好write_batch
    RequestScope request_scope(transaction_participant_.get());
    RETURN_NOT_OK(PrepareTransactionWriteBatch(batch_idx, put_batch, hybrid_time, &write_batch));
    # 写RocksDB
    WriteToRocksDB(frontiers, &write_batch, StorageDbType::kIntents);
  } else {
    # 准备好write_batch
    PrepareNonTransactionWriteBatch(put_batch, hybrid_time, &write_batch);
    # 写RocksDB 
    WriteToRocksDB(frontiers, &write_batch, StorageDbType::kRegular);
  }

  return Status::OK();
}

void Tablet::WriteToRocksDB(
    const rocksdb::UserFrontiers* frontiers,
    rocksdb::WriteBatch* write_batch,
    docdb::StorageDbType storage_db_type) {
  rocksdb::DB* dest_db = nullptr;
  switch (storage_db_type) {
    case StorageDbType::kRegular: dest_db = regular_db_.get(); break;
    case StorageDbType::kIntents: dest_db = intents_db_.get(); break;
  }
  
  # 在WriteBatch中设置Frontiers信息
  write_batch->SetFrontiers(frontiers);

  // We are using Raft replication index for the RocksDB sequence number for
  // all members of this write batch.
  rocksdb::WriteOptions write_options;
  InitRocksDBWriteOptions(&write_options);

  # RocksDB对WriteBatch中Frontiers的处理，见“RocksDB对WriteBatch::frontiers_的处理”
  auto rocksdb_write_status = dest_db->Write(write_options, write_batch);
}
```

### ConsensusFrontier::op_id_的使用
ConsensusFrontier::op_id_主要用于在GC过程中，确定需要被保留的最小opId。
```
LogGCOp::Perform
TabletPeer::RunLogGC
TabletPeer::GetEarliestNeededLogIndex
Tablet::MaxPersistentOpId
MaxPersistentOpIdForDb

Status TabletPeer::RunLogGC() {
  int64_t min_log_index = VERIFY_RESULT(GetEarliestNeededLogIndex());
  int32_t num_gced = 0;
  return log_->GC(min_log_index, &num_gced);
}

Result<int64_t> TabletPeer::GetEarliestNeededLogIndex(std::string* details) const {
  # 获取已经写入raft log并sync的最后一个OpId
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
      if (details) {
        *details += Format("Min anchor index: $0\n", min_anchor_index);
      }
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
  if (details && min_pending_op_index != std::numeric_limits<int64_t>::max()) {
    *details += Format("Min pending op id index: $0\n", min_pending_op_index);
  }

  auto min_retryable_request_op_id = consensus_->MinRetryableRequestOpId();
  min_index = std::min(min_index, min_retryable_request_op_id.index);
  if (details) {
    *details += Format("Min retryable request op id: $0\n", min_retryable_request_op_id);
  }

  auto* transaction_coordinator = tablet()->transaction_coordinator();
  if (transaction_coordinator) {
    auto transaction_coordinator_min_op_index = transaction_coordinator->PrepareGC(details);
    min_index = std::min(min_index, transaction_coordinator_min_op_index);
  }

  // We keep at least one committed operation in the log so that we can always recover safe time
  // during bootstrap.
  // Last committed op id should be read before MaxPersistentOpId to avoid race condition
  // described in MaxPersistentOpIdForDb.
  //
  // If we read last committed op id AFTER reading last persistent op id (INCORRECT):
  // - We read max persistent op id and find there is no new data, so we ignore it.
  // - New data gets written and Raft-committed, but not yet flushed to an SSTable.
  // - We read the last committed op id, which is greater than what max persistent op id would have
  //   returned.
  // - We garbage-collect the Raft log entries corresponding to the new data.
  // - Power is lost and the server reboots, losing committed data.
  //
  // If we read last committed op id BEFORE reading last persistent op id (CORRECT):
  // - We read the last committed op id.
  // - We read max persistent op id and find there is no new data, so we ignore it.
  // - New data gets written and Raft-committed, but not yet flushed to an SSTable.
  // - We still don't garbage-collect the logs containing the committed but unflushed data,
  //   because the earlier value of the last committed op id that we read prevents us from doing so.
  auto last_committed_op_id = consensus()->GetLastCommittedOpId();
  min_index = std::min(min_index, last_committed_op_id.index);
  if (details) {
    *details += Format("Last committed op id: $0\n", last_committed_op_id);
  }

  if (tablet_->table_type() != TableType::TRANSACTION_STATUS_TABLE_TYPE) {
    tablet_->FlushIntentsDbIfNecessary(latest_log_entry_op_id);
    auto max_persistent_op_id = VERIFY_RESULT(
        tablet_->MaxPersistentOpId(true /* invalid_if_no_new_data */));
    if (max_persistent_op_id.regular.valid()) {
      min_index = std::min(min_index, max_persistent_op_id.regular.index);
      if (details) {
        *details += Format("Max persistent regular op id: $0\n", max_persistent_op_id.regular);
      }
    }
    if (max_persistent_op_id.intents.valid()) {
      min_index = std::min(min_index, max_persistent_op_id.intents.index);
      if (details) {
        *details += Format("Max persistent intents op id: $0\n", max_persistent_op_id.intents);
      }
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

  if (details) {
    *details += Format("Earliest needed log index: $0\n", min_index);
  }

  return min_index;
}

Result<DocDbOpIds> Tablet::MaxPersistentOpId(bool invalid_if_no_new_data) const {
  ScopedRWOperation scoped_read_operation(&pending_op_counter_);
  RETURN_NOT_OK(scoped_read_operation);

  return DocDbOpIds{
      MaxPersistentOpIdForDb(regular_db_.get(), invalid_if_no_new_data),
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
  
  # 如果VersionSet中保存了有效的UserFrontier，则直接返回之
  auto result = versions_->FlushedFrontier();
  if (result) {
    return result->Clone();
  }
  
  # 否则，从SST files中获取最大的已经flush的UserFrontier
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

class VersionSet {
  UserFrontier* FlushedFrontier() const {
    return flushed_frontier_.get();
  }
}

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

## ConsensusFrontier::history_cutoff_
### ConsensusFrontier::history_cutoff_的设置
#### 在HistoryCutoffOperation成功执行raft复制之后进行设置
```
OperationDriver::ApplyOperation
OperationDriver::ApplyTask
Operation::Replicated
HistoryCutoffOperation::DoReplicated
Status HistoryCutoffOperationState::Replicated

Status HistoryCutoffOperationState::Replicated(int64_t leader_term) {
  HybridTime history_cutoff(request()->history_cutoff());
  history_cutoff = tablet()->RetentionPolicy()->UpdateCommittedHistoryCutoff(history_cutoff);
  auto regular_db = tablet()->doc_db().regular;
  if (regular_db) {
    rocksdb::WriteBatch batch;
    docdb::ConsensusFrontiers frontiers;
    frontiers.Largest().set_history_cutoff(history_cutoff);
    batch.SetFrontiers(&frontiers);
    rocksdb::WriteOptions options;
    RETURN_NOT_OK(regular_db->Write(options, &batch));
  }
  return Status::OK();
}
```

#### 在compaction开始之前使用HistoryRetentionDirective::history_cutoff设置CompactionJob::largest_user_frontier_
```
CompactionJob::Run
CompactionJob::ProcessKeyValueCompaction
DocDBCompactionFilter::GetLargestUserFrontier

void CompactionJob::ProcessKeyValueCompaction(
    FileNumbersHolder* holder, SubcompactionState* sub_compact) {
  ...
  
  if (compaction_filter) {
    // This is used to persist the history cutoff hybrid time chosen for the DocDB compaction
    // filter.
    largest_user_frontier_ = compaction_filter->GetLargestUserFrontier();
  }
  
  ...
}

rocksdb::UserFrontierPtr DocDBCompactionFilter::GetLargestUserFrontier() const {
  auto* consensus_frontier = new ConsensusFrontier();
  consensus_frontier->set_history_cutoff(retention_.history_cutoff);
  return rocksdb::UserFrontierPtr(consensus_frontier);
}
```

### ConsensusFrontier::history_cutoff_的使用
#### 在打开tablet的过程中使用
```
TSTabletManager::OpenTablet
BootstrapTablet
BootstrapTabletImpl
TabletBootstrap::Bootstrap
TabletBootstrap::OpenTablet
Tablet::Open
Tablet::OpenKeyValueTablet

Status Tablet::OpenKeyValueTablet() {
  ...
  
  auto regular_flushed_frontier = regular_db_->GetFlushedFrontier();
  if (regular_flushed_frontier) {
    retention_policy_->UpdateCommittedHistoryCutoff(
        static_cast<const docdb::ConsensusFrontier&>(*regular_flushed_frontier).history_cutoff());
  }
  
  ...
}
```

#### 在Restore snapshot过程中使用
```
OperationDriver::ApplyOperation
OperationDriver::ApplyTask
Operation::Replicated
SnapshotOperation::DoReplicated
SnapshotOperationState::Apply
TabletSnapshots::Restore
TabletSnapshots::RestoreCheckpoint

Status TabletSnapshots::RestoreCheckpoint(
    const std::string& dir, const docdb::ConsensusFrontier& frontier) {
  ...
  
  if (checkpoint_flushed_frontier) {
    final_frontier.set_history_cutoff(
        down_cast<docdb::ConsensusFrontier&>(*checkpoint_flushed_frontier).history_cutoff());
  }    
  
  ...
}
```

#### 在compaction结束之前使用CompactionJob::largest_user_frontier_更新Compaction::edit_
```
DBImpl::CompactFilesImpl
CompactionJob::Install
CompactionJob::InstallCompactionResults

Status CompactionJob::InstallCompactionResults(
    const MutableCFOptions& mutable_cf_options) {
  ...
  
  if (largest_user_frontier_) {
    compaction->edit()->UpdateFlushedFrontier(largest_user_frontier_);
  }
  
  ...
}
```


## RocksDB对WriteBatch::frontiers_的处理
```
DB::Put
DBImpl::Write
DBImpl::WriteImpl
WriteBatchInternal::InsertInto

Status WriteBatchInternal::InsertInto(const WriteBatch* batch,
                                      ColumnFamilyMemTables* memtables,
                                      FlushScheduler* flush_scheduler,
                                      bool ignore_missing_column_families,
                                      uint64_t log_number, DB* db,
                                      InsertFlags insert_flags) {
  MemTableInserter inserter(WriteBatchInternal::Sequence(batch), memtables,
                            flush_scheduler, ignore_missing_column_families,
                            log_number, db, insert_flags);
  return batch->Iterate(&inserter);
}

Status WriteBatch::Iterate(Handler* handler) const {
  Slice input(rep_);
  if (input.size() < kHeader) {
    return STATUS(Corruption, "malformed WriteBatch (too small)");
  }

  input.remove_prefix(kHeader);
  Slice key, value, blob;
  size_t found = 0;
  Status s;

  # 如果WriteBatch中包含UserFrontiers信息，则更新MemTable::frontiers_
  if (frontiers_) {
    s = handler->Frontiers(*frontiers_);
  }

  ...  
}

class MemTableInserter : public WriteBatch::Handler {
  ...
  
  CHECKED_STATUS Frontiers(const UserFrontiers& frontiers) override {
    Status seek_status;
    if (!SeekToColumnFamily(0, &seek_status)) {
      return seek_status;
    }
    cf_mems_->GetMemTable()->UpdateFrontiers(frontiers);
    return Status::OK();
  }
  
  ...
}

class MemTable {
  ...
  
  void UpdateFrontiers(const UserFrontiers& value) {
    std::lock_guard<SpinMutex> l(frontiers_mutex_);
    if (frontiers_) {
      frontiers_->MergeFrontiers(value);
    } else {
      frontiers_ = value.Clone();
    }
  }
 
  ... 
}

template<class Frontier>
class UserFrontiersBase : public rocksdb::UserFrontiers {
 public:
  ...
  
  void MergeFrontiers(const UserFrontiers& pre_rhs) override {
    const auto& rhs = down_cast<const UserFrontiersBase&>(pre_rhs);
    # 在当前上下文中，smallest_和largest_的类型都是ConsensusFrontier
    smallest_.Update(rhs.smallest_, rocksdb::UpdateUserValueType::kSmallest);
    largest_.Update(rhs.largest_, rocksdb::UpdateUserValueType::kLargest);
  }

 private:
  Frontier smallest_;
  Frontier largest_;
};

void ConsensusFrontier::Update(
    const rocksdb::UserFrontier& pre_rhs, rocksdb::UpdateUserValueType update_type) {
  const ConsensusFrontier& rhs = down_cast<const ConsensusFrontier&>(pre_rhs);
  UpdateField(&op_id_, rhs.op_id_, update_type);
  UpdateField(&ht_, rhs.ht_, update_type);
  UpdateField(&history_cutoff_, rhs.history_cutoff_, update_type);
  // Reset filter after compaction.
  hybrid_time_filter_ = HybridTime();
}

template<typename T>
void UpdateField(
    T* this_value, const T& new_value, rocksdb::UpdateUserValueType update_type) {
  switch (update_type) {
    case rocksdb::UpdateUserValueType::kLargest:
      this_value->MakeAtLeast(new_value);
      return;
    case rocksdb::UpdateUserValueType::kSmallest:
      this_value->MakeAtMost(new_value);
      return;
  }
  FATAL_INVALID_ENUM_VALUE(rocksdb::UpdateUserValueType, update_type);
}
```