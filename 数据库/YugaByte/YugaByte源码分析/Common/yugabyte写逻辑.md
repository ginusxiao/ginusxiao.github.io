# 提纲
[toc]
        
# 写请求到达YB-TServer之后的执行路径

```
TabletServiceImpl::Write
    - tablet_peer->SubmitWrite
        - tablet_->AcquireLocksAndPerformDocOperations
            - KeyValueBatchFromQLWriteBatch
                - StartDocWriteOperation
                    - docdb::PrepareDocWriteOperation
                    - 
```

## 锁相关的逻辑                    
```
void PrepareDocWriteOperation(const vector<unique_ptr<DocOperation>>& doc_write_ops,
                              const scoped_refptr<Histogram>& write_lock_latency,
                              IsolationLevel isolation_level,
                              SharedLockManager *lock_manager,
                              LockBatch *keys_locked,
                              bool *need_read_snapshot) {
  KeyToIntentTypeMap key_to_lock_type;
  *need_read_snapshot = false;
  // 对于每一个操作分别执行
  for (const unique_ptr<DocOperation>& doc_op : doc_write_ops) {
    list<DocPath> doc_paths;
    IsolationLevel level;
    
    // 获取每一个操作所涉及的doc_paths
    doc_op->GetDocPathsToLock(&doc_paths, &level);
    if (isolation_level != IsolationLevel::NON_TRANSACTIONAL) {
      level = isolation_level;
    }
    
    // 根据隔离级别获取@intent_types，其中intent_types是一个2元组，
    // 要么是{ docdb::IntentType::kStrongSnapshotWrite, docdb::IntentType::kWeakSnapshotWrite }
    // 要么是{ docdb::IntentType::kStrongSerializableRead, docdb::IntentType::kWeakSerializableWrite }
    const IntentTypePair intent_types = GetWriteIntentsForIsolationLevel(level);

    // 对doc_paths中的每一个path分别执行
    for (const auto& doc_path : doc_paths) {
      KeyBytes current_prefix = doc_path.encoded_doc_key();
      // 设置path中的每一个subkey上应该加的锁类型，并将path对应的key和锁类型添加到key_to_lock_type中
      for (int i = 0; i < doc_path.num_subkeys(); i++) {
        ApplyIntent(current_prefix.AsStringRef(), intent_types.weak, &key_to_lock_type);
        doc_path.subkey(i).AppendToKey(&current_prefix);
      }
      ApplyIntent(current_prefix.AsStringRef(), intent_types.strong, &key_to_lock_type);
    }
    if (doc_op->RequireReadSnapshot()) {
      *need_read_snapshot = true;
    }
  }
  const MonoTime start_time = (write_lock_latency != nullptr) ? MonoTime::Now() : MonoTime();
  // 加锁，从LockBatch的实现来看，会等待所有的key都申请到锁才会返回
  *keys_locked = LockBatch(lock_manager, std::move(key_to_lock_type));
  if (write_lock_latency != nullptr) {
    const MonoDelta elapsed_time = MonoTime::Now().GetDeltaSince(start_time);
    write_lock_latency->Increment(elapsed_time.ToMicroseconds());
  }
}    

LockBatch::LockBatch(SharedLockManager* lock_manager, KeyToIntentTypeMap&& key_to_intent_type)
    : key_to_type_(std::move(key_to_intent_type)),
      shared_lock_manager_(lock_manager) {
  if (!empty()) {
    lock_manager->Lock(key_to_type_);
  }
}

void SharedLockManager::Lock(const KeyToIntentTypeMap& key_to_intent_type) {
  TRACE("Locking a batch of $0 keys", key_to_intent_type.size());
  // 见下面SharedLockManager::Reserve
  std::vector<SharedLockManager::LockEntry*> reserved = Reserve(key_to_intent_type);
  size_t idx = 0;
  for (const auto& key_and_intent_type : key_to_intent_type) {
    const auto intent_type = key_and_intent_type.second;
    VLOG(4) << "Locking " << docdb::ToString(intent_type) << ": "
            << util::FormatBytesAsStr(key_and_intent_type.first);
    // 加锁
    reserved[idx]->Lock(intent_type);
    idx++;
  }
  TRACE("Acquired a lock batch of $0 keys", key_to_intent_type.size());
}

std::vector<SharedLockManager::LockEntry*> SharedLockManager::Reserve(
    const KeyToIntentTypeMap& key_to_intent_type) {
  std::vector<SharedLockManager::LockEntry*> reserved;
  reserved.reserve(key_to_intent_type.size());
  {
    std::lock_guard<std::mutex> lock(global_mutex_);
    for (const auto& key_and_intent_type : key_to_intent_type) {
      auto it = locks_.emplace(key_and_intent_type.first, std::make_unique<LockEntry>()).first;
      it->second->num_using++;
      reserved.push_back(&*it->second);
    }
  }
  return reserved;
}

void SharedLockManager::LockEntry::Lock(IntentType lock_type) {
  // TODO(bojanserafimov): Implement CAS fast path. Only wait when CAS fails.
  int type_idx = static_cast<size_t>(lock_type);
  std::unique_lock<std::mutex> lock(mutex);
  auto& state = this->state;
  // 等待，直到当前的锁状态@state和请求加锁的锁类型@lock_type不冲突
  cond_var.wait(lock, [&state, type_idx]() {
    return (state & kIntentConflicts[type_idx]).none();
  });
  ++num_holding[type_idx];
  // 设置当前的锁状态为持有@lock_type类型的锁
  state.set(type_idx);
}
```

## Tablet leader为写操作选择时间戳
