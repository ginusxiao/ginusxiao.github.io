# 提纲
[toc]

### 1. DocWriteOperation::DoStart
```
  CHECKED_STATUS DoStart() {
    auto write_batch = operation_->request()->mutable_write_batch();
    isolation_level_ = VERIFY_RESULT(tablet_.GetIsolationLevelFromPB(*write_batch));
    const RowMarkType row_mark_type = GetRowMarkTypeFromPB(*write_batch);
    const auto& metadata = *tablet_.metadata();
    const bool transactional_table = metadata.schema()->table_properties().is_transactional() ||
                                     operation_->force_txn_path();

    const auto partial_range_key_intents = UsePartialRangeKeyIntents(metadata);
    # 见“2. PrepareDocWriteOperation”
    prepare_result_ = VERIFY_RESULT(docdb::PrepareDocWriteOperation(
        operation_->doc_ops(), write_batch->read_pairs(), tablet_.metrics()->write_lock_latency,
        isolation_level_, operation_->state()->kind(), row_mark_type, transactional_table,
        operation_->deadline(), partial_range_key_intents, tablet_.shared_lock_manager()));

    auto* transaction_participant = tablet_.transaction_participant();
    read_time_ = operation_->read_time();

    if (isolation_level_ == IsolationLevel::NON_TRANSACTIONAL) {
      # 对于单行事务，会走到这里
      auto now = tablet_.clock()->Now();
      docdb::ResolveOperationConflicts(
          operation_->doc_ops(), now, tablet_.doc_db(), partial_range_key_intents,
          transaction_participant,
          [self = shared_from_this(), now](const Result<HybridTime>& result) {
            if (!result.ok()) {
              self->InvokeCallback(result.status());
              return;
            }
            self->NonTransactionalConflictsResolved(now, *result);
          });
      return Status::OK();
    }

    if (isolation_level_ == IsolationLevel::SERIALIZABLE_ISOLATION &&
        prepare_result_.need_read_snapshot) {
      boost::container::small_vector<RefCntPrefix, 16> paths;
      for (const auto& doc_op : operation_->doc_ops()) {
        paths.clear();
        IsolationLevel ignored_isolation_level;
        RETURN_NOT_OK(doc_op->GetDocPaths(
            docdb::GetDocPathsMode::kLock, &paths, &ignored_isolation_level));
        for (const auto& path : paths) {
          auto key = path.as_slice();
          auto* pair = write_batch->mutable_read_pairs()->Add();
          pair->set_key(key.data(), key.size());
          // Empty values are disallowed by docdb.
          // https://github.com/YugaByte/yugabyte-db/issues/736
          pair->set_value(std::string(1, docdb::ValueTypeAsChar::kNullLow));
        }
      }
    }

    # 对于snapshot isolation的处理
    # 见“3. ResolveTransactionConflicts”
    docdb::ResolveTransactionConflicts( 
        operation_->doc_ops(), *write_batch, tablet_.clock()->Now(),
        read_time_ ? read_time_.read : HybridTime::kMax,
        tablet_.doc_db(), partial_range_key_intents,
        transaction_participant, tablet_.metrics()->transaction_conflicts.get(),
        [self = shared_from_this()](const Result<HybridTime>& result) {
          if (!result.ok()) {
            self->InvokeCallback(result.status());
            return;
          }
          self->TransactionalConflictsResolved();
        });

    return Status::OK();
  }
```

### 2. PrepareDocWriteOperation
```
Result<PrepareDocWriteOperationResult> PrepareDocWriteOperation(
    const std::vector<std::unique_ptr<DocOperation>>& doc_write_ops,
    const google::protobuf::RepeatedPtrField<KeyValuePairPB>& read_pairs,
    const scoped_refptr<Histogram>& write_lock_latency,
    const IsolationLevel isolation_level,
    const OperationKind operation_kind,
    const RowMarkType row_mark_type,
    bool transactional_table,
    CoarseTimePoint deadline,
    PartialRangeKeyIntents partial_range_key_intents,
    SharedLockManager *lock_manager) {
  PrepareDocWriteOperationResult result;
  # 见“4. DetermineKeysToLock”
  # 
  # 最终的determine_keys_to_lock_result.lock_batch中包含当前操作所涉及的所有的
  # keys和这些keys上对应的intent lock
  auto determine_keys_to_lock_result = VERIFY_RESULT(DetermineKeysToLock(
      doc_write_ops, read_pairs, isolation_level, operation_kind, row_mark_type,
      transactional_table, partial_range_key_intents));
  result.need_read_snapshot = determine_keys_to_lock_result.need_read_snapshot;

  # 对determine_keys_to_lock_result.lock_batch中所有的key进行排序，合并相同的key
  # 的intent lock，对于相同的keys，在determine_keys_to_lock_result.lock_batch中
  # 只保留一个
  FilterKeysToLock(&determine_keys_to_lock_result.lock_batch);
  const MonoTime start_time = (write_lock_latency != nullptr) ? MonoTime::Now() : MonoTime();
  
  # 见“5. LockBatch”
  # 尝试进行加锁，如果加锁失败，则result.lock_batch.data_.status中会包含错误状态信息
  result.lock_batch = LockBatch(
      lock_manager, std::move(determine_keys_to_lock_result.lock_batch), deadline);
  RETURN_NOT_OK_PREPEND(
      result.lock_batch.status(), Format("Timeout: $0", deadline - ToCoarse(start_time)));
  if (write_lock_latency != nullptr) {
    const MonoDelta elapsed_time = MonoTime::Now().GetDeltaSince(start_time);
    write_lock_latency->Increment(elapsed_time.ToMicroseconds());
  }

  return result;
}
```

### 3. ResolveTransactionConflicts
```
void ResolveTransactionConflicts(const DocOperations& doc_ops,
                                 const KeyValueWriteBatchPB& write_batch,
                                 HybridTime hybrid_time,
                                 HybridTime read_time,
                                 const DocDB& doc_db,
                                 PartialRangeKeyIntents partial_range_key_intents,
                                 TransactionStatusManager* status_manager,
                                 Counter* conflicts_metric,
                                 ResolutionCallback callback) {
  DCHECK(hybrid_time.is_valid());
  auto context = std::make_unique<TransactionConflictResolverContext>(
      doc_ops, write_batch, hybrid_time, read_time, conflicts_metric);
  auto resolver = std::make_shared<ConflictResolver>(
      doc_db, status_manager, partial_range_key_intents, std::move(context), std::move(callback));
  # 见“8. Resolve”
  // Resolve takes a self reference to extend lifetime.
  resolver->Resolve();
}
```

### 4. DetermineKeysToLock
```
Result<DetermineKeysToLockResult> DetermineKeysToLock(
    const std::vector<std::unique_ptr<DocOperation>>& doc_write_ops,
    const google::protobuf::RepeatedPtrField<KeyValuePairPB>& read_pairs,
    const IsolationLevel isolation_level,
    const OperationKind operation_kind,
    const RowMarkType row_mark_type,
    bool transactional_table,
    PartialRangeKeyIntents partial_range_key_intents) {
  # DetermineKeysToLockResult中包括：
  # LockBatchEntries：关于LockBatchEntry的数组，每一个LockBatchEntry包括：
  # key，intent type set和LockedBatchEntry
  # need_read_snapshot：是否需要读取snapshot
  DetermineKeysToLockResult result;
  boost::container::small_vector<RefCntPrefix, 8> doc_paths;
  boost::container::small_vector<size_t, 32> key_prefix_lengths;
  result.need_read_snapshot = false;
  
  # 遍历doc_write_ops中的每一个DocOperation
  for (const unique_ptr<DocOperation>& doc_op : doc_write_ops) {
    doc_paths.clear();
    IsolationLevel level;
    # 获取当前doc_op对应的DocKey和isolation level
    RETURN_NOT_OK(doc_op->GetDocPaths(GetDocPathsMode::kLock, &doc_paths, &level));
    if (isolation_level != IsolationLevel::NON_TRANSACTIONAL) {
      level = isolation_level;
    }
    
    # 获取当前操作需要哪种strong lock，有2种strong lock：strong read和strong write
    IntentTypeSet strong_intent_types = GetStrongIntentTypeSet(level, operation_kind,
                                                               row_mark_type);
                                                               
    # 对满足下面条件的操作特殊设置其strong lock类型，在GetStrongIntentTypeSet中对于
    # SERIALIZABLE_ISOLATION且OperationKind::kWrite的操作设置的是strong write lock，
    # 但是如果其需要读取snapshot的话，则也需要支持strong read
    #
    # doc_op->RequireReadSnapshot()的一种可能的用例是：在sql语句中有if <condition>语句
    if (isolation_level == IsolationLevel::SERIALIZABLE_ISOLATION &&
        operation_kind == OperationKind::kWrite &&
        doc_op->RequireReadSnapshot()) {
      strong_intent_types = IntentTypeSet({IntentType::kStrongRead, IntentType::kStrongWrite});
    }

    # 当前上下文中doc_paths中只包含DocKey这一个元素
    for (const auto& doc_path : doc_paths) {
      key_prefix_lengths.clear();
      # 分别获取DocKey中Hash keys(包括hash value和所有的hash keys)部分对应的前缀长度，range keys
      # 中每一个range key对应的前缀长度，sub keys中每一个sub key对应的前缀长度，每一部分的长度
      # 保存在key_prefix_lengths中
      RETURN_NOT_OK(SubDocKey::DecodePrefixLengths(doc_path.as_slice(), &key_prefix_lengths));
      # key_prefix_lengths中最后一个元素表示整个DocKey的长度，对于整个DocKey，需要加strong lock，
      # 可以直接从key_prefix_lengths中移除之
      // We will acquire strong lock on entire doc_path, so remove it from list of weak locks.
      key_prefix_lengths.pop_back();
      # 初始化之后partial_key就是整个DocKey
      auto partial_key = doc_path;
      # empty key是干嘛的？
      // Acquire weak lock on empty key for transactional tables,
      // unless specified key is already empty.
      if (doc_path.size() > 0 && transactional_table) {
        partial_key.Resize(0);
        # 见“6. ApplyIntent”
        RETURN_NOT_OK(ApplyIntent(
            partial_key, StrongToWeak(strong_intent_types), &result.lock_batch));
      }
      
      # 此时key_prefix_lengths中已经排除了整个DocKey了，对于key_prefix_lengths中的剩余的key
      # 都加weak lock
      for (size_t prefix_length : key_prefix_lengths) {
        partial_key.Resize(prefix_length);
        # 见“6. ApplyIntent”
        # 首先去除partial_key中末尾的所有的kGroupEnd标记，然后将当前partial_key和
        # 对应的lock一并添加到result.lock_batch中
        RETURN_NOT_OK(ApplyIntent(
            partial_key, StrongToWeak(strong_intent_types), &result.lock_batch));
      }

      # 对整个DocKey添加strong lock
      RETURN_NOT_OK(ApplyIntent(doc_path, strong_intent_types, &result.lock_batch));
    }

    # 设置是否需要读取snapshot
    if (doc_op->RequireReadSnapshot()) {
      result.need_read_snapshot = true;
    }
  }

  # 如果read_pairs不为空，则对于read_pairs中的每一个KV pair也需要加intent lock
  if (!read_pairs.empty()) {
    RETURN_NOT_OK(EnumerateIntents(
        read_pairs,
        [&result](IntentStrength strength, Slice value, KeyBytes* key, LastKey) {
          RefCntPrefix prefix(key->data());
          auto intent_types = strength == IntentStrength::kStrong
              ? IntentTypeSet({IntentType::kStrongRead})
              : IntentTypeSet({IntentType::kWeakRead});
          return ApplyIntent(prefix, intent_types, &result.lock_batch);
        }, partial_range_key_intents));
  }

  return result;
}
```

### 5. LockBatch
```
LockBatch::LockBatch(SharedLockManager* lock_manager, LockBatchEntries&& key_to_intent_type,
                     CoarseTimePoint deadline)
    : data_(std::move(key_to_intent_type), lock_manager) {
  # 见“7. SharedLockManager::Lock”
  if (!empty() && !lock_manager->Lock(&data_.key_to_type, deadline)) {
    data_.shared_lock_manager = nullptr;
    data_.key_to_type.clear();
    data_.status = STATUS_FORMAT(
        TryAgain, "Failed to obtain locks until deadline: $0", deadline);
  }
}

bool SharedLockManager::Lock(LockBatchEntries* key_to_intent_type, CoarseTimePoint deadline) {
  # SharedLockManager::Impl::Lock
  return impl_->Lock(key_to_intent_type, deadline);
}

bool SharedLockManager::Impl::Lock(LockBatchEntries* key_to_intent_type, CoarseTimePoint deadline) {
  # 为key_to_intent_type中的每个key分配对应的LockedBatchEntry结构
  Reserve(key_to_intent_type);
  for (auto it = key_to_intent_type->begin(); it != key_to_intent_type->end(); ++it) {
    const auto& key_and_intent_type = *it;
    const auto intent_types = key_and_intent_type.intent_types;
    # 在当前的key上加锁，见LockedBatchEntry::Lock
    if (!key_and_intent_type.locked->Lock(intent_types, deadline)) {
      # 如果加锁失败，则解除之前已经添加的锁
      while (it != key_to_intent_type->begin()) {
        --it;
        it->locked->Unlock(it->intent_types);
      }
      
      # 从SharedLockManager::Impl::locks_中移除相应的key
      Cleanup(*key_to_intent_type);
      return false;
    }
  }

  return true;
}

void SharedLockManager::Impl::Reserve(LockBatchEntries* key_to_intent_type) {
  std::lock_guard<std::mutex> lock(global_mutex_);
  # key_to_intent_type中包含每一个key和它对应的intent lock
  for (auto& key_and_intent_type : *key_to_intent_type) {
    # 在locks map中查找对应的key
    auto& value = locks_[key_and_intent_type.key];
    if (!value) {
      # 如果不存在，则为之分配一个LockedBatchEntry
      if (!free_lock_entries_.empty()) {
        # 从可用的LockedBatchEntry资源池中分配
        value = free_lock_entries_.back();
        free_lock_entries_.pop_back();
      } else {
        # 直接从内存中分配
        lock_entries_.emplace_back(std::make_unique<LockedBatchEntry>());
        value = lock_entries_.back().get();
      }
    }
    
    value->ref_count++;
    # 将当前key_and_intent_type所对应的LockedBatchEntry添加到key_and_intent_type.locked
    # 中，后续如果需要对它进行访问的话，无需加global_mutex_这个锁
    key_and_intent_type.locked = value;
  }
}

bool LockedBatchEntry::Lock(IntentTypeSet lock_type, CoarseTimePoint deadline) {
  # 获取lock_type对应的整型值
  size_t type_idx = lock_type.ToUIntPtr();
  # num_holding表示关于当前这个key的intent lock的持有情况(持有哪些intent lock)
  auto& num_holding = this->num_holding;
  auto old_value = num_holding.load(std::memory_order_acquire);

  auto add = kIntentTypeSetAdd[type_idx];
  for (;;) {
    # 持有的这些intent lock和当前要加的intent lock之间不存在冲突
    if ((old_value & kIntentTypeSetConflicts[type_idx]) == 0) {
      # 把当前的intent lock添加到num_holding中
      auto new_value = old_value + add;
      if (num_holding.compare_exchange_weak(old_value, new_value, std::memory_order_acq_rel)) {
        return true;
      }
      continue;
    }
    
    # 走到这里说明存在冲突，则增加num_waiters计数
    num_waiters.fetch_add(1, std::memory_order_release);
    
    # 当从该方法返回的时候，减少num_waiters计数
    auto se = ScopeExit([this] {
      num_waiters.fetch_sub(1, std::memory_order_release);
    });
    
    # 存在冲突的情况下，需要加锁来处理？
    std::unique_lock<std::mutex> lock(mutex);
    # 重新获取num_holding的值，如果存在冲突，则进入条件等待，等待一定的
    # 时间，如果超时了，就会返回false
    old_value = num_holding.load(std::memory_order_acquire);
    if ((old_value & kIntentTypeSetConflicts[type_idx]) != 0) {
      if (deadline != CoarseTimePoint::max()) {
        if (cond_var.wait_until(lock, deadline) == std::cv_status::timeout) {
          return false;
        }
      } else {
        cond_var.wait(lock);
      }
    }
  }
}
```

### 6. ApplyIntents
```
CHECKED_STATUS ApplyIntent(RefCntPrefix key,
                           const IntentTypeSet intent_types,
                           LockBatchEntries *keys_locked) {
  // Have to strip kGroupEnd from end of key, because when only hash key is specified, we will
  // get two kGroupEnd at end of strong intent.
  size_t size = key.size();
  if (size > 0) {
    if (key.data()[0] == ValueTypeAsChar::kGroupEnd) {
      if (size != 1) {
        return STATUS_FORMAT(Corruption, "Key starting with group end: $0",
            key.as_slice().ToDebugHexString());
      }
      size = 0;
    } else {
      # 去除key末尾的所有的kGroupEnd标记
      while (key.data()[size - 1] == ValueTypeAsChar::kGroupEnd) {
        --size;
      }
    }
  }
  
  # Resize之后key就是去除了kGroupEnd标记的了
  key.Resize(size);
  # 将key和它对应的intent_types一并添加到keys_locked中
  keys_locked->push_back({key, intent_types});
  return Status::OK();
}
```

### 7. SharedLockManager::Lock
```
bool SharedLockManager::Lock(LockBatchEntries* key_to_intent_type, CoarseTimePoint deadline) {
  return impl_->Lock(key_to_intent_type, deadline);
}
```

### 8. Resolve
```
  void Resolve() {
    # 调用的是TransactionConflictResolverContext::ReadConflicts
    auto status = context_->ReadConflicts(this);
    if (!status.ok()) {
      callback_(status);
      return;
    }

    ResolveConflicts();
  }

  CHECKED_STATUS ReadConflicts(ConflictResolver* resolver) override {
    # TransactionParticipant::PrepareMetadata
    # 返回transaction metadata，但是对于单行事务，没有transaction metadata？
    metadata_ = VERIFY_RESULT(resolver->PrepareMetadata(write_batch_.transaction()));

    boost::container::small_vector<RefCntPrefix, 8> paths;

    const size_t kKeyBufferInitialSize = 512;
    KeyBytes buffer;
    buffer.Reserve(kKeyBufferInitialSize);
    const auto row_mark = GetRowMarkTypeFromPB(write_batch_);
    IntentTypesContainer container;
    IntentProcessor write_processor(
        &container,
        GetStrongIntentTypeSet(metadata_.isolation, docdb::OperationKind::kWrite, row_mark));
    # 逐一处理DocOperations中的每一个DocOperation
    for (const auto& doc_op : doc_ops_) {
      paths.clear();
      IsolationLevel ignored_isolation_level;
      # 这里在GetDocPaths的过程中，使用的是GetDocPathsMode::kIntents模式，
      # 此时，会对本次操作所涉及的每个column id之前添加DocKey，以此作为
      # 每个Column的DocPath，存放到paths中，paths中还包括DocKey作为最后
      # 一个元素
      RETURN_NOT_OK(doc_op->GetDocPaths(
          GetDocPathsMode::kIntents, &paths, &ignored_isolation_level));

      # 逐一处理每个DocPath，所有的keys和它对应的intent type保存在
      # IntentTypesContainer中，而所有的keys都保存在buffer中
      for (const auto& path : paths) {
        RETURN_NOT_OK(EnumerateIntents(
            path.as_slice(),
            /* intent_value */ Slice(),
            [&write_processor](auto strength, auto, auto intent_key, auto) {
              write_processor.Process(strength, intent_key);
              return Status::OK();
            },
            &buffer,
            resolver->partial_range_key_intents()));
      }
    }
    
    # 针对read pairs的处理，跟上面针对doc_ops_的处理类似
    const auto& pairs = write_batch_.read_pairs();
    if (!pairs.empty()) {
      IntentProcessor read_processor(
          &container,
          GetStrongIntentTypeSet(metadata_.isolation, docdb::OperationKind::kWrite, row_mark));
      RETURN_NOT_OK(EnumerateIntents(
          pairs,
          [&read_processor](auto strength, auto, auto intent_key, auto) {
            read_processor.Process(strength, intent_key);
            return Status::OK();
          },
          resolver->partial_range_key_intents()));
    }

    if (container.empty()) {
      return Status::OK();
    }

    StrongConflictChecker checker(
        *transaction_id_, read_time_, resolver, conflicts_metric_, &buffer);
    // Iterator on intents DB should be created before iterator on regular DB.
    // This is to prevent the case when we create an iterator on the regular DB where a
    // provisional record has not yet been applied, and then create an iterator the intents
    // DB where the provisional record has already been removed.
    resolver->EnsureIntentIteratorCreated();

    for(const auto& i : container) {
      if (read_time_ != HybridTime::kMax && HasStrong(i.second)) {
        RETURN_NOT_OK(checker.Check(i.first));
      }
      buffer.Reset(i.first);
      RETURN_NOT_OK(resolver->ReadIntentConflicts(i.second, &buffer));
    }

    return Status::OK();
  }
  
  void ResolveConflicts() {
    if (conflicts_.empty()) {
      callback_(context_->GetResolutionHt());
      return;
    }

    transactions_.reserve(conflicts_.size());
    for (const auto& transaction_id : conflicts_) {
      transactions_.push_back({ transaction_id });
    }

    DoResolveConflicts();
  }
  
  CHECKED_STATUS Check(const Slice& intent_key) {
    const auto hash = VERIFY_RESULT(FetchDocKeyHash(intent_key));
    if (PREDICT_FALSE(!value_iter_.Initialized() || hash != value_iter_hash_)) {
      # 尚未为当前的intent key设置rocksdb iterator
      value_iter_ = CreateRocksDBIterator(
          resolver_.doc_db().regular,
          resolver_.doc_db().key_bounds,
          BloomFilterMode::USE_BLOOM_FILTER,
          intent_key,
          rocksdb::kDefaultQueryId);
      value_iter_hash_ = hash;
    }
    
    value_iter_.Seek(intent_key);
    # 找出给定intent_key的所有的sub doc keys，直到某个sub dock key的hybrid time
    # 大于read time为止，或者遇到第一个不是intent_key的sub doc key为止，如果
    # 先遇到了一个sub doc key的hybrid time大于read time，则返回错误状态
    // Inspect records whose doc keys are children of the intent's doc key.  If the intent's doc
    // key is empty, it signifies an intent on the whole table.
    while (value_iter_.Valid() &&
           (intent_key.starts_with(ValueTypeAsChar::kGroupEnd) ||
            value_iter_.key().starts_with(intent_key))) {
      # existing_key是intent key的sub dock key
      auto existing_key = value_iter_.key();
      # 获取sub doc key中记录的hybrid time
      auto doc_ht = VERIFY_RESULT(DocHybridTime::DecodeFromEnd(&existing_key));
      if (doc_ht.hybrid_time() >= read_time_) {
        conflicts_metric_.Increment();
        return (STATUS(TryAgain,
                       Format("Value write after transaction start: $0 >= $1",
                              doc_ht.hybrid_time(), read_time_), Slice(),
                       TransactionError(TransactionErrorCode::kConflict)));
      }
      
      buffer_.Reset(existing_key);
      // Already have ValueType::kHybridTime at the end
      buffer_.AppendHybridTime(DocHybridTime::kMin);
      ROCKSDB_SEEK(&value_iter_, buffer_.AsSlice());
    }

    return Status::OK();
  }  
  
  // Reads conflicts for specified intent from DB.
  CHECKED_STATUS ReadIntentConflicts(IntentTypeSet type, KeyBytes* intent_key_prefix) {
    EnsureIntentIteratorCreated();

    const auto conflicting_intent_types = kIntentTypeSetConflicts[type.ToUIntPtr()];

    KeyBytes upperbound_key(*intent_key_prefix);
    upperbound_key.AppendValueType(ValueType::kMaxByte);
    intent_key_upperbound_ = upperbound_key.AsSlice();

    size_t original_size = intent_key_prefix->size();
    intent_key_prefix->AppendValueType(ValueType::kIntentTypeSet);
    // Have only weak intents, so could skip other weak intents.
    if (!HasStrong(type)) {
      char value = 1 << kStrongIntentFlag;
      intent_key_prefix->AppendRawBytes(&value, 1);
    }
    auto se = ScopeExit([this, intent_key_prefix, original_size] {
      intent_key_prefix->Truncate(original_size);
      intent_key_upperbound_.clear();
    });
    Slice prefix_slice(intent_key_prefix->AsSlice().data(), original_size);
    intent_iter_.Seek(intent_key_prefix->AsSlice());
    while (intent_iter_.Valid()) {
      auto existing_key = intent_iter_.key();
      auto existing_value = intent_iter_.value();
      if (!existing_key.starts_with(prefix_slice)) {
        break;
      }
      // Support for obsolete intent type.
      // When looking for intent with specific prefix it should start with this prefix, followed
      // by ValueType::kIntentTypeSet.
      // Previously we were using intent type, so should support its value type also, now it is
      // kObsoleteIntentType.
      // Actual handling of obsolete intent type is done in ParseIntentKey.
      if (existing_key.size() <= prefix_slice.size() ||
          !IntentValueType(existing_key[prefix_slice.size()])) {
        break;
      }
      if (existing_value.empty() || existing_value[0] != ValueTypeAsChar::kTransactionId) {
        return STATUS_FORMAT(Corruption,
            "Transaction prefix expected in intent: $0 => $1",
            existing_key.ToDebugHexString(),
            existing_value.ToDebugHexString());
      }
      existing_value.consume_byte();
      auto existing_intent = VERIFY_RESULT(
          docdb::ParseIntentKey(intent_iter_.key(), existing_value));

      const auto intent_mask = kIntentTypeSetMask[existing_intent.types.ToUIntPtr()];
      if ((conflicting_intent_types & intent_mask) != 0) {
        auto transaction_id = VERIFY_RESULT(FullyDecodeTransactionId(
            Slice(existing_value.data(), TransactionId::StaticSize())));

        if (!context_->IgnoreConflictsWith(transaction_id)) {
          conflicts_.insert(transaction_id);
        }
      }

      intent_iter_.Next();
    }

    return Status::OK();
  }  
```