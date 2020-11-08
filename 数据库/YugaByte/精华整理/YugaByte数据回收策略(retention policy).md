# 提纲
[toc]

## 前言
YugaByte将数据保存在RocksDB中，它借助于RocksDB的Compaction机制实现数据回收。它通过TabletRetentionPolicy和RocksDB compaction filter机制共同实现数据回收。

## TabletRetentionPolicy数据结构
```
class TabletRetentionPolicy : public docdb::HistoryRetentionPolicy {
  ...
  
  mutable std::mutex mutex_;
  # 保存所有活跃的操作对应的HybridTime类型的读时间戳，只有比该集合中第一个元素
  # 小的时间戳对应的数据才可以被回收
  std::multiset<HybridTime> active_readers_ GUARDED_BY(mutex_);
  
  # 在FLAGS_enable_history_cutoff_propagation为true的情况下，在
  # TabletRetentionPolicy::UpdateCommittedHistoryCutoff中被更新，
  # 在TabletRetentionPolicy::RegisterReaderTimestamp和
  # TabletRetentionPolicy::GetRetentionDirective中被使用，其中，
  # 在TabletRetentionPolicy::RegisterReaderTimestamp中用于判断是
  # 否将当前的时间戳注册到active_readers_中，在
  # TabletRetentionPolicy::GetRetentionDirective中用于判断哪个时间
  # 戳之前的数据可以被回收了
  # 
  # 在FLAGS_enable_history_cutoff_propagation为false的情况下，只在
  # TabletRetentionPolicy::GetRetentionDirective中被更新，在
  # TabletRetentionPolicy::RegisterReaderTimestamp中被使用，用于决定
  # 是否将当前的时间戳注册到active_readers_中
  HybridTime committed_history_cutoff_ GUARDED_BY(mutex_) = HybridTime::kMin;
  
  # 在FLAGS_enable_history_cutoff_propagation为true的情况下，下一次向follower
  # propagate history cutoff的时间
  CoarseTimePoint next_history_cutoff_propagation_ GUARDED_BY(mutex_) = CoarseTimePoint::min();
};
```

## TabletRetentionPolicy::active_readers_的维护
### 注册读写操作对应的read time到TabletRetentionPolicy::active_readers_中
在写操作冲突解决之后，如果该写操作需要读取snapshot，则会通过ScopedReadOperation::Create将对应的read_time注册到TabletRetentionPolicy::active_readers_中。

在TabletService接收到读操作的时候，会通过ScopedReadOperation::Create将对应的read_time注册到TabletRetentionPolicy::active_readers_中。
```
# write操作中通过ScopedReadOperation::Create创建ScopedReadOperation
DocWriteOperation::TransactionalConflictsResolved
DocWriteOperation::Complete
DocWriteOperation::DoComplete

class DocWriteOperation : public std::enable_shared_from_this<DocWriteOperation> {
  CHECKED_STATUS DoComplete() {
    # prepare_result_.need_read_snapshot
    auto read_op = prepare_result_.need_read_snapshot
        ? VERIFY_RESULT(ScopedReadOperation::Create(&tablet_, RequireLease::kTrue, read_time_))
        : ScopedReadOperation();
 
    ...
  }       
}

# read操作中通过ScopedReadOperation::Create创建ScopedReadOperation
Result<ReadHybridTime> TabletServiceImpl::DoRead(ReadContext* read_context) {
  auto read_tx = VERIFY_RESULT(
      tablet::ScopedReadOperation::Create(
          read_context->tablet.get(), read_context->require_lease, read_context->read_time));
  ...
}

Result<ScopedReadOperation> ScopedReadOperation::Create(
    AbstractTablet* tablet,
    RequireLease require_lease,
    ReadHybridTime read_time) {
  if (!read_time) {
    read_time = ReadHybridTime::SingleTime(tablet->SafeTime(require_lease));
  }
  
  auto* retention_policy = tablet->RetentionPolicy();
  if (retention_policy) {
    # 将当前操作的read time注册到TabletRetentionPolicy::active_readers_中
    RETURN_NOT_OK(retention_policy->RegisterReaderTimestamp(read_time.read));
  }
  
  # 返回ScopedReadOperation，并将read_time记录在ScopedReadOperation::read_time_中，
  # ScopedReadOperation::read_time_将用于在析构ScopedReadOperation的时候，从
  # TabletRetentionPolicy::active_readers_移除注册的关于当前操作的read time
  return ScopedReadOperation(tablet, read_time);
}
```

### 从TabletRetentionPolicy::active_readers_中移除注册的read time
当从DocWriteOperation::DoComplete()和TabletServiceImpl::DoRead退出的时候，函数内部的ScopedReadOperation类型的局部变量都会被析构。在ScopedReadOperation的析构函数中则会从TabletRetentionPolicy::active_readers_中移除注册的read time。

```
class DocWriteOperation : public std::enable_shared_from_this<DocWriteOperation> {
  CHECKED_STATUS DoComplete() {
    auto read_op = prepare_result_.need_read_snapshot
        ? VERIFY_RESULT(ScopedReadOperation::Create(&tablet_, RequireLease::kTrue, read_time_))
        : ScopedReadOperation();
 
    ...
    
    # 当从这里退出的时候，ScopedReadOperation类型的read_op将会被析构
  }       
}

# read操作中通过ScopedReadOperation::Create创建ScopedReadOperation
Result<ReadHybridTime> TabletServiceImpl::DoRead(ReadContext* read_context) {
  auto read_tx = VERIFY_RESULT(
      tablet::ScopedReadOperation::Create(
          read_context->tablet.get(), read_context->require_lease, read_context->read_time));
  ...
  
  # 当从这里退出的时候，ScopedReadOperation类型的read_op将会被析构
}

# ScopedReadOperation的析构函数
ScopedReadOperation::~ScopedReadOperation() {
  Reset();
}

void ScopedReadOperation::Reset() {
  if (tablet_) {
    # 获取Tablet对应的TabletRetentionPolicy
    auto* retention_policy = tablet_->RetentionPolicy();
    
    # 从TabletRetentionPolicy::read_time_中移除注册的read time(记录在
    # ScopedReadOperation::read_time_中)
    if (retention_policy) {
      retention_policy->UnregisterReaderTimestamp(read_time_.read);
    }
    
    tablet_ = nullptr;
  }
}

void TabletRetentionPolicy::UnregisterReaderTimestamp(HybridTime timestamp) {
  std::lock_guard<std::mutex> lock(mutex_);
  # 从TabletRetentionPolicy::read_time_中移除注册的read time
  active_readers_.erase(timestamp);
}
```

## TabletRetentionPolicy::committed_history_cutoff_的维护
### leader向follower propagate history cutoff信息
只有在FLAGS_enable_history_cutoff_propagation的情况下，leader才会向follower propagate history cutoff信息，但是默认情况下，FLAGS_enable_history_cutoff_propagation为false，所以leader不会向follower propagate history cutoff信息。

如果用户将FLAGS_enable_history_cutoff_propagation调整为true，则leader会主动生成一个HistoryCutoffOperation操作，并通过将该操作提交给raft复制，达到将leader本地的history cutoff propagate给follower的目的。
```
Peer::SendNextRequest
PeerMessageQueue::RequestForPeer
TabletPeer::PreparePeerRequest
TabletRetentionPolicy::HistoryCutoffToPropagate

HybridTime TabletPeer::PreparePeerRequest() {
  auto leader_term = consensus_->GetLeaderState(/* allow_stale= */ true).term;
  if (leader_term >= 0) {
    # 获取last replicated HybridTime
    auto last_write_ht = tablet_->mvcc_manager()->LastReplicatedHybridTime();
    
    # 计算出来的propagated_history_cutoff将会propagate给follower
    auto propagated_history_cutoff =
        tablet_->RetentionPolicy()->HistoryCutoffToPropagate(last_write_ht);

    # 如果propagated_history_cutoff是kInvalidHybridTimeValue，则不会进入下面的if语句，
    # 也就是不会propagate history cutoff到follower上
    if (propagated_history_cutoff) {
      auto state = std::make_unique<HistoryCutoffOperationState>(tablet_.get());
      # 分配待发送给follower的request
      auto request = state->AllocateRequest();
      # 在HistoryCutoffPB request中设置history cutoff信息
      request->set_history_cutoff(propagated_history_cutoff.ToUint64());

      # 生成一个HistoryCutoffOperation操作并提交给raft进行复制(实际上是先提交给本地
      # 的raft，然后在下一次调用Peer::SendNextRequest的时候，再发送给其它的raft peer)
      auto operation = std::make_unique<tablet::HistoryCutoffOperation>(std::move(state));
      Submit(std::move(operation), leader_term);
    }
  }

  ...
}

HybridTime TabletRetentionPolicy::HistoryCutoffToPropagate(HybridTime last_write_ht) {
  std::lock_guard<std::mutex> lock(mutex_);

  auto now = CoarseMonoClock::now();

  # 默认情况下，FLAGS_enable_history_cutoff_propagation为false，所以会直接返回
  # kInvalidHybridTimeValue(HybridTime的默认构造函数会返回kInvalidHybridTimeValue)
  #
  # 如果FLAGS_enable_history_cutoff_propagation为true，但是当前时间(now)并没有到达
  # 下一次history cutoff propagation的时间(next_history_cutoff_propagation_)，或者
  # 最后一个replicated操作对应的时间(last_write_ht)仍然小于committed_history_cutoff_,
  # 则返回kInvalidHybridTimeValue，表示当前无需propagate history cutoff到follower上
  if (!FLAGS_enable_history_cutoff_propagation || now < next_history_cutoff_propagation_ ||
      last_write_ht <= committed_history_cutoff_) {
    return HybridTime();
  }

  # 至此，需要propagate history cutoff到follower
  
  # 更新next_history_cutoff_propagation_
  next_history_cutoff_propagation_ = now + FLAGS_history_cutoff_propagation_interval_ms * 1ms;
  
  # 返回current time减去retention interval所代表的的HybridTime作为“提议的”history cutoff，
  # 然后再结合TabletRetentionPolicy::active_readers_，共同计算出最终的history cutoff
  return EffectiveHistoryCutoff();
}
```

### follower接收到来自leader的history cutoff信息时的处理
```
ConsensusServiceIf::Handle
ConsensusServiceImpl::UpdateConsensus
RaftConsensus::Update
RaftConsensus::UpdateReplica
RaftConsensus::EnqueuePreparesUnlocked
RaftConsensus::StartReplicaOperationUnlocked
TabletPeer::StartReplicaOperation

Status TabletPeer::StartReplicaOperation(
    const scoped_refptr<ConsensusRound>& round, HybridTime propagated_safe_time) {
  ...
  
  auto operation = CreateOperation(replicate_msg);

  ...
  
  OperationDriverPtr driver = VERIFY_RESULT(NewReplicaOperationDriver(&operation));

  ...

  driver->ExecuteAsync();
  return Status::OK();
}

std::unique_ptr<Operation> TabletPeer::CreateOperation(consensus::ReplicateMsg* replicate_msg) {
  switch (replicate_msg->op_type()) {
    ...
    
    case consensus::HISTORY_CUTOFF_OP:
      return std::make_unique<HistoryCutoffOperation>(
          std::make_unique<HistoryCutoffOperationState>(tablet()));
          
    ...
  }
}
```

### HistoryCutoffOperation成功完成raft复制后的处理
在HistoryCutoffOperation成功完成raft复制之后，就会将
```
OperationDriver::ApplyOperation
OperationDriver::ApplyTask
Operation::Replicated
HistoryCutoffOperation::DoReplicated
HistoryCutoffOperationState::Replicated
TabletRetentionPolicy::UpdateCommittedHistoryCutoff

Status HistoryCutoffOperationState::Replicated(int64_t leader_term) {
  # 从HistoryCutoffOperationState对应的HistoryCutoffPB request中获取history cutoff
  HybridTime history_cutoff(request()->history_cutoff());

  # 以获取到的history_cutoff更新TabletRetentionPolicy::committed_history_cutoff_
  history_cutoff = tablet()->RetentionPolicy()->UpdateCommittedHistoryCutoff(history_cutoff);
  
  # 根据history_cutoff生成docdb::ConsensusFrontiers，并更新到RocksDB中
  auto regular_db = tablet()->doc_db().regular;
  if (regular_db) {
    rocksdb::WriteBatch batch;
    docdb::ConsensusFrontiers frontiers;
    frontiers.Largest().set_history_cutoff(history_cutoff);
    # 生成batch，然后写入RocksDB
    batch.SetFrontiers(&frontiers);
    rocksdb::WriteOptions options;
    RETURN_NOT_OK(regular_db->Write(options, &batch));
  }
  
  return Status::OK();
}

HybridTime TabletRetentionPolicy::UpdateCommittedHistoryCutoff(HybridTime value) {
  std::lock_guard<std::mutex> lock(mutex_);
  if (!value) {
    return committed_history_cutoff_;
  }

  # 更新committed_history_cutoff_
  committed_history_cutoff_ = std::max(committed_history_cutoff_, value);
  return committed_history_cutoff_;
}
```

## YugaByte自定义的Compaction Filter - DocDBCompactionFilter
RocksDB通过ImmutableCFOptions支持用户通过自定义的CompactionFilter或者CompactionFilterFactory来实现自定义的CompactionFilter。YugaByte实现了DocDBCompactionFilterFactory，提供自定义CompactionFilter的功能。
```
Compaction::CreateCompactionFilter
DocDBCompactionFilterFactory::CreateCompactionFilter
TabletRetentionPolicy::GetRetentionDirective


std::unique_ptr<CompactionFilter> Compaction::CreateCompactionFilter() const {
  ...
  
  # 调用用户自定义的CompactionFilterFactory来创建自定义的CompactionFilter
  return cfd_->ioptions()->compaction_filter_factory->CreateCompactionFilter(context);
}

unique_ptr<CompactionFilter> DocDBCompactionFilterFactory::CreateCompactionFilter(
    const CompactionFilter::Context& context) {
  # 创建名为DocDBCompactionFilter的自定义CompactionFilter
  return std::make_unique<DocDBCompactionFilter>(
      retention_policy_->GetRetentionDirective(),
      IsMajorCompaction(context.is_full_compaction),
      key_bounds_);
}

# HistoryRetentionDirective用于在compaction过程中决定记录是否应该被保留
HistoryRetentionDirective TabletRetentionPolicy::GetRetentionDirective() {
  HybridTime history_cutoff;
  {
    std::lock_guard<std::mutex> lock(mutex_);
    
    # FLAGS_enable_history_cutoff_propagation默认为false，表示不从leader propagate
    # history cutoff到follower
    if (FLAGS_enable_history_cutoff_propagation) {
      # 如果FLAGS_enable_history_cutoff_propagation被用户调整为true，则要同时结合
      # committed_history_cutoff_和active_readers_来决定将要propagate给follower的
      # history cutoff
      history_cutoff = SanitizeHistoryCutoff(committed_history_cutoff_);
    } else {
      # 否则不会从leader propagate history cutoff到follower，只在leader本地计算
      # 自身的history cutoff(current time减去retention interval_)，然后再结合
      # TabletRetentionPolicy::active_readers_一同计算history cutoff
      history_cutoff = EffectiveHistoryCutoff();
      committed_history_cutoff_ = std::max(history_cutoff, committed_history_cutoff_);
    }
  }

  # 被delete的column信息，将被添加到deleted_before_history_cutoff中
  std::shared_ptr<ColumnIds> deleted_before_history_cutoff = std::make_shared<ColumnIds>();
  for (const auto& deleted_col : *metadata_.deleted_cols()) {
    if (deleted_col.ht < history_cutoff) {
      deleted_before_history_cutoff->insert(deleted_col.id);
    }
  }

  # 生成HistoryRetentionDirective，用以指导Compaction过程
  return {history_cutoff, std::move(deleted_before_history_cutoff),
          TableTTL(*metadata_.schema()),
          docdb::ShouldRetainDeleteMarkersInMajorCompaction(
              ShouldRetainDeleteMarkersInMajorCompaction())};
}

# 综合考虑参数@proposed_cutoff和TabletRetentionPolicy::active_readers_来决定最终的history cutoff
HybridTime TabletRetentionPolicy::SanitizeHistoryCutoff(HybridTime proposed_cutoff) {
  HybridTime allowed_cutoff;
  if (active_readers_.empty()) {
    # 可以回收proposed_cutoff之前的任何记录
    allowed_cutoff = proposed_cutoff;
  } else {
    # 不能回收任何仍在active_readers_中的记录
    allowed_cutoff = std::min(proposed_cutoff, *active_readers_.begin());
  }

  return allowed_cutoff;
}

HybridTime TabletRetentionPolicy::EffectiveHistoryCutoff() {
  auto retention_delta = -FLAGS_timestamp_history_retention_interval_sec * 1s;
  # 返回current time减去retention interval所代表的的HybridTime作为history cutoff，
  # 但是这样计算出来的history cutoff，并不能作为最终的history cutoff，还必须将
  # TabletRetentionPolicy::active_readers_纳入考虑
  return SanitizeHistoryCutoff(clock_->Now().AddDelta(retention_delta));
}
```

## DocDBCompactionFilter是如何工作的
下面一段摘自RocksDB wiki：
Each time a (sub)compaction sees a new key from its input and when the value is a normal value, it invokes the compaction filter. Based on the result of the compaction filter:
- If it decide to keep the key, nothing will change.
- If it request to filter the key, the value is replaced by a deletion marker. Note that if the output level of the compaction is the bottom level, no deletion marker will need to be output.
- If it request to change the value, the value is replaced by the changed value.
- If it request to remove a range of keys by returning kRemoveAndSkipUntil, the compaction will skip over to skip_until (means skip_until will be the next possible key output by the compaction). This one is tricky because in this case the compaction does not insert deletion marker for the keys it skips. This means older version of the keys may reappears as a result. On the other hand, it is more efficient to simply dropping the keys, if the application know there aren't older versions of the keys, or reappearance of the older versions is fine.
- If there are multiple versions of the same key from the input of the compaction, compaction filter will only be invoked once for the newest version. If the newest version is a deletion marker, compaction filter will not be invoked. However, it is possible the compaction filter is invoked on a deleted key, if the deletion marker isn't included in the input of the compaction.

When merge is being used, compaction filter is invoked per merge operand. The result of compaction filter is applied to the merge operand before merge operator is invoked.

Before release 6.0, if there is a snapshot taken later than the key/value pair, RocksDB always try to prevent the key/value pair from being filtered by compaction filter so that users can preserve the same view from a snapshot, unless the compaction filter returns IgnoreSnapshots() = true. However, this feature is deleted since 6.0, after realized that the feature has a bug which can't be easily fixed. Since release 6.0, with compaction filter enabled, RocksDB always invoke filtering for any key, even if it knows it will make a snapshot not repeatable.