# 提纲
[toc]

## TServer单行事务的处理
### TServer接收到写请求的RPC时的处理
当TServer接收到写请求时，依次会执行以下逻辑：
```
TabletServerServiceIf::Handle
    - TabletServiceImpl::Write
        # 查找tablet leader
        - auto tablet = LookupLeaderTabletOrRespond(
            server_->tablet_peer_lookup(), req->tablet_id(), resp, &context)
        - auto operation_state = std::make_unique<WriteOperationState>(tablet.peer->tablet(), req, resp)
        # 设置WriteOperationCompletionCallback，当操作执行完毕之后，
        # WriteOperationCompletionCallback::OperationCompleted将会被调用
        - operation_state->set_completion_callback(...)
        # 如果是system catalog tables，则必须确保该table相关的操作是transactional的
        - AdjustYsqlOperationTransactionality
        # 提交请求给table异步执行
        - tablet.peer->WriteAsync(std::move(operation_state), ...)
            - auto operation = std::make_unique<WriteOperation>(
                std::move(state), term, std::move(preparing_token), deadline, this)
            - tablet_->AcquireLocksAndPerformDocOperations(std::move(operation))
                # 对于YSQL相关的操作会进入这里，对于YCQL/YEDIS等会进入其它逻辑
                - KeyValueBatchFromPgsqlWriteBatch(std::move(operation))
                    - auto status = PreparePgsqlWriteOperations(operation.get())
                        # 获取WriteOperation中的DocOperations句柄，DocOperations是关于
                        # DocOperation的数组
                        - docdb::DocOperations& doc_ops = operation->doc_ops()
                        # 交换operation->request()和batch_request中的信息，交换之后，
                        # batch_request中将包含operation->request()中的所有信息，然后
                        # 在利用batch_request中的信息来设置operation->request()中的必要
                        # 信息
                        - SetupKeyValueBatch(operation->request(), &batch_request)
                        # 获取write batch，其中可能包含多个write操作
                        - auto* pgsql_write_batch = batch_request.mutable_pgsql_write_batch()
                        # 在doc_ops中预留一定的空间，每个write操作(一个RPC中可能包含多个操作)
                        # 占用doc_ops中的一个空间
                        - doc_ops.reserve(pgsql_write_batch->size())
                        - 逐一遍历pgsql_write_batch中的每一个PgsqlWriteRequest，执行如下：
                            # 获取第i个write 操作
                            - PgsqlWriteRequestPB* req = pgsql_write_batch->Mutable(i)
                            # 为第i个write 操作生成一个response结构
                            - PgsqlResponsePB* resp =     
                                operation->response()->add_pgsql_response_batch()
                            # CreateTransactionOperationContext只在第一个write操作执行之前创建一次
                            - if (doc_ops.empty()) {
                                txn_op_ctx = CreateTransactionOperationContext(...);
                              }
                            # 为当前的操作创建一个PgsqlWriteOperation，参数tnx_op_ctx
                            # 会进一步传递给PgsqlWriteOperation::txn_op_context_
                            - auto write_op = std::make_unique<PgsqlWriteOperation>
                              (table_info->schema, *txn_op_ctx)
                              # 设置PgsqlWriteOperation::request_和PgsqlWriteOperation::response_,
                              # 从ybctid中获取DocKey或者从partition columns和range columns生成
                              # DocKey，保存在PgsqlWriteOperation::doc_key_中
                              write_op->Init(req, resp)
                              # 将当前操作对应的PgsqlWriteOperation添加到doc_ops中，实际上是
                              # 添加到了WriteOperation::doc_ops_中
                              doc_ops.emplace_back(std::move(write_op))
                    # 根据接收到的WriteOperation生成DocWriteOperation，StartDocWriteOperation方法中
                    # 第三个参数类型是DocWriteOperationCallback
                    - StartDocWriteOperation(...)
                        # 生成DocWriteOperation
                        # StartDocWriteOperation中第三个类型为DocWriteOperationCallback的参数，会在
                        # 这里传递给DocWriteOperation::callback_，它会在事务执行过程中出错，或者
                        # 事务执行完成的时候被调用
                        - auto doc_write_operation = std::make_shared<DocWriteOperation>()
                        - doc_write_operation->Start()
                            - Tablet::DoStart()
```

#### 创建TransactionOperationContext 
```
Result<TransactionOperationContextOpt> Tablet::CreateTransactionOperationContext(
    const TransactionMetadataPB& transaction_metadata,
    bool is_ysql_catalog_table) const {
  if (!txns_enabled_)
    return boost::none;

  if (transaction_metadata.has_transaction_id()) {
    # 对于分布式事务来说，在WriteRequestPB的KeyValueWriteBatchPB中附带有
    # transaction metadata，其中包含transaction id，可以直接使用
    Result<TransactionId> txn_id = FullyDecodeTransactionId(
        transaction_metadata.transaction_id());
    RETURN_NOT_OK(txn_id);
    return CreateTransactionOperationContext(boost::make_optional(*txn_id), is_ysql_catalog_table);
  } else {
    # 对于单行事务，或者非事务性操作，则没有对应的transaction id
    return CreateTransactionOperationContext(boost::none, is_ysql_catalog_table);
  }
}

Tablet::CreateTransactionOperationContext
    - if (transaction_id.is_initialized()) {
        # 分布式事务会走到这里
        return TransactionOperationContext(transaction_id.get(), transaction_participant());
      } else if (metadata_->schema()->table_properties().is_transactional() ||    
        is_ysql_catalog_table) {
        # 如果操作的table是transactional的，或者是YSQL system catalog table，则走到这里
        # 这里会随机生成一个transaction id
        return TransactionOperationContext(
            TransactionId::GenerateRandom(), transaction_participant());
      } else {
        # 其它
        return boost::none;
      }
```
在创建TransactionOperationContext的时候，无论是单行事务还是分布式事务，都包含一个transaction id，对于分布式事务，这个transaction id直接从transaction metadata中获取，对于单行事务，则随机生成一个transaction id，且创建的TransactionOperationContext中会包含一个TransactionStatusManager，使用的是该tablet对应的TransactionParticipant。

#### 事务执行完成或者事务执行过程中出错情况下的回调
这个回调是在Tablet::KeyValueBatchFromPgsqlWriteBatch -> Tablet::StartDocWriteOperation中被设置的。当事务执行完成时，我们再来看，它到底做了些什么？
```
void Tablet::KeyValueBatchFromPgsqlWriteBatch(std::unique_ptr<WriteOperation> operation) {
  ...

  # StartDocWriteOperation方法中的第3个参数就是在事务执行完成或者事务执行过程中出错情况下的回调
  StartDocWriteOperation(std::move(operation), std::move(scoped_read_operation),
                         [](auto operation, const Status& status) {
    if (!status.ok() || operation->restart_read_ht().is_valid()) {
      WriteOperation::StartSynchronization(std::move(operation), status);
      return;
    }
    
    auto& doc_ops = operation->doc_ops();
    for (size_t i = 0; i < doc_ops.size(); i++) {
      PgsqlWriteOperation* pgsql_write_op = down_cast<PgsqlWriteOperation*>(doc_ops[i].get());
      // We'll need to return the number of rows inserted, updated, or deleted by each operation.
      doc_ops[i].release();
      operation->state()->pgsql_write_ops()
                        ->emplace_back(unique_ptr<PgsqlWriteOperation>(pgsql_write_op));
    }

    WriteOperation::StartSynchronization(std::move(operation), Status::OK());
  });
}
```

### 单行事务的执行
单行事务执行主要包括以下几个阶段：
- 准备阶段
    - 分析当前操作(WriteOperation，包括1个或者多个PgsqlWriteOperation)中所涉及的所有的keys及其前缀所需要加的intent lock锁类型
    - 对当前操作中所涉及的所有的keys及其前缀部分加相应的intent lock
        - 如果不存在锁冲突，则加锁成功，更新SharedLockManager中记录的关于所有keys的锁的状态
        - 如果存在锁冲突，则等待锁的其它持有者释放锁后唤醒
            - 如果在等待过程中，发生了超时，则退出等待
            - 如果等待过程中被唤醒，则重新对当前操作中所涉及的所有的keys及其前缀部分加相应的intent lock
- 检测冲突，解决冲突
    - 检测和临时记录之间是否存在冲突
        - 因为任何一个操作，一旦成功写入了临时记录，就会释放它所持有的锁，所以在准备阶段进行加锁的时候可能不会出现锁冲突，但是临时记录毕竟是临时记录，如果事务还没有commit的话，还是可能会rollback的，所以必须检查和临时记录之间是否存在冲突
    - 如果存在冲突，则解决冲突，解决冲突的基本思想是：
        - a. 移除所有处于COMMITTED或者ABORTED状态的transaction，其中transaction状态可能从status tablet获取，也可能直接从本地获取(如果本地已经获取过transaction状态，且当前记录的transaction状态是可信的)
        - b. 如果移除之后没有其它冲突的transactions，则认为所有冲突已经解决，直接调用冲突解决之后的回调，否则，对所有剩余的存在冲突的transactions，逐一发送AbortTransactionRequestPB请求给status tablet
        - c. 当接收到所有transactions的AbortTransactionRequestPB请求的响应时，移除所有处于COMMITTED或者ABORTED状态的transaction
        - d. 如果移除之后没有其它冲突的transactions，则认为所有冲突已经解决，直接调用冲突解决之后的回调，否则，还存在冲突的transactions，进入步骤a
- 执行阶段

```                           
Tablet::DocWriteOperation::DoStart
    # 对于PGSQL类型的操作，partial_range_key_intents为true，其它类型为false
    # partial_range_key_intents用于决定在解析DocKey的过程中处理range keys部分的时候
    # 是否针对每个range key单独处理
    - const auto partial_range_key_intents = UsePartialRangeKeyIntents(metadata)
    # 为操作所涉及的DocKey及其DocKey的所有前缀加锁
    - prepare_result_ = VERIFY_RESULT(docdb::PrepareDocWriteOperation(operation_->doc_ops(), ...)
        # PrepareDocWriteOperationResult将作为docdb::PrepareDocWriteOperation的返回值，
        # 它当中它当中包括两个成员：所有需要加锁的key到它所对应的intent lock的映射表，
        # 以及是否需要read snapshot的标识
        - PrepareDocWriteOperationResult result
        
        # 决定doc_write_ops和read_pairs中的每个DocOperation对应的DocKey及其DocKey的所有前缀
        # 需要加哪种intent lock，返回的结果determine_keys_to_lock_result，它当中包含两个成员：
        # 所有需要加锁的key到它所对应的intent lock的映射表，以及是否需要read snapshot的标识
        - auto determine_keys_to_lock_result = docdb::DetermineKeysToLock(doc_write_ops, read_pairs...)
        
        # 设置PrepareDocWriteOperationResult::need_read_snapshot
        - result.need_read_snapshot = determine_keys_to_lock_result.need_read_snapshot
        
        # 首先对determine_keys_to_lock_result.lock_batch中的所有的keys按照升序排序，然后
        # 合并相同key的IntentTypeSet，并且在determine_keys_to_lock_result.lock_batch中对
        # 于相同的key只保留一个，这个key对应的IntentTypeSet是所有跟该key相同的key对应的
        # IntentTypeSet的并集
        - FilterKeysToLock(&determine_keys_to_lock_result.lock_batch)
        # 对determine_keys_to_lock_result.lock_batch中所有的LockBatchEntry加锁，如果不存
        # 在锁冲突，则直接加锁，否则会等待，直到加锁成功，或者超时时间到
        - result.lock_batch = LockBatch(
            lock_manager, std::move(determine_keys_to_lock_result.lock_batch), deadline)
        # 如果加锁失败，则返回
        - RETURN_NOT_OK_PREPEND(
            result.lock_batch.status(), Format("Timeout: $0", deadline - ToCoarse(start_time)))
    # 获取tablet相关联的transaction participant
    - auto* transaction_participant = tablet_.transaction_participant()
    - 对于隔离级别为SNAPSHOT_ISOLATION或者SERIALIZABLE_ISOLATION情况的处理，暂略
    # 检查和临时记录之间是否存在冲突，如果存在，则解决冲突
    - docdb::ResolveOperationConflicts(
          operation_->doc_ops(), now, tablet_.doc_db(), partial_range_key_intents,
          transaction_participant,
          # 这是当冲突解决完毕之后的回调
          [self = shared_from_this(), now](const Result<HybridTime>& result) {
            if (!result.ok()) {
              self->InvokeCallback(result.status());
              return;
            }
            self->NonTransactionalConflictsResolved(now, *result);
          });
        # 构建OperationConflictResolverContext                             
        auto context = std::make_unique<OperationConflictResolverContext>(&doc_ops, resolution_ht);
        # 构建ConflictResolver
        auto resolver = std::make_shared<ConflictResolver>(
            doc_db, status_manager, partial_range_key_intents, std::move(context), std::move(callback));
        # 检查并解决冲突
        resolver->Resolve();
            # 在当前上下文中对应的是OperationConflictResolverContext::ReadConflicts
            - auto status = context_->ReadConflicts(this);
            # 如果有冲突，则解决之
            - ResolveConflicts();
}
```

#### 计算DocKey及其前缀要加哪种类型的intent lock - DetermineKeysToLock
DetermineKeysToLock会获取操作中所涉及的所有的keys及其前缀部分需要加哪种类型的锁，结果保存在DetermineKeysToLockResult中，DetermineKeysToLockResult中包括2个成员：LockBatchEntries类型的lock_batch和bool类型的need_read_snapshot，DetermineKeysToLockResult::lock_batch中存放key到key上所加的锁的类型的映射，以及该key在SharedLockManager中的加锁状态信息(LockedBatchEntry)；DetermineKeysToLockResult::need_read_snapshot表示是否需要读取snapshot数据。
```
Result<DetermineKeysToLockResult> DetermineKeysToLock(
    const std::vector<std::unique_ptr<DocOperation>>& doc_write_ops,
    const google::protobuf::RepeatedPtrField<KeyValuePairPB>& read_pairs,
    const IsolationLevel isolation_level,
    const OperationKind operation_kind,
    const RowMarkType row_mark_type,
    bool transactional_table,
    PartialRangeKeyIntents partial_range_key_intents)
    - DetermineKeysToLockResult result
    - 逐一遍历参数doc_write_ops中的每一个DocOperation，对每一个DocOperation执行如下操作：
        # 获取当前DocOperation对应的Doc Path和操作所需要用到的isolation level(不会影响事务
        # 本身的隔离级别)，这里实际调用的是PgsqlWriteOperation::GetDocPaths，因为这里采用
        # 的是GetDocPathsMode::kLock模式，所以获取到的doc_paths中只包含DocKey
        - doc_op->GetDocPaths(GetDocPathsMode::kLock, &doc_paths, &level)
            # 设置临时的isolation level，它只是用于决定获取哪种类型的IntentLock，
            # 而不会影响事务本身的隔离级别，注释是这么说的：当写操作需要读的时候，
            # 它需要read snapshot，因此使用snapshot isolation，而对于纯写操作(pure
            # writes，不需要读)，使用serializable isolation
            #
            # 从RequireReadSnapshot来看，只有PgsqlWriteRequestPB::PGSQL_UPSERT类型
            # 的操作会返回false，其它操作，比如标准的insert/update/delete都返回true，
            # 因为他们需要读取primary key
            #
            # 综上，只有upsert操作会走serializable isolation，其它都是snapshot
            - *level = RequireReadSnapshot() ? IsolationLevel::SNAPSHOT_ISOLATION
                 : IsolationLevel::SERIALIZABLE_ISOLATION
            # 当前上下文中，在调用GetDocPaths的时候，使用的是GetDocPathsMode::kLock
            # 模式，因此会进入到这里，否则对于GetDocPathsMode::kIntents模式，会对每
            # 一列数据都按照DocKey + ColumnId的形式生成Doc Path
            #
            # 对于GetDocPathsMode::kLock模式，直接将encoded_doc_key_添加到doc paths中
            - if (encoded_doc_key_) {
                paths->push_back(encoded_doc_key_);
              }
    # isolation_level是事务本身的隔离级别，如果事物本身的隔离级别不是NON_TRANSACTIONAL，
    # 则直接使用事务本身的隔离级别
    -   if (isolation_level != IsolationLevel::NON_TRANSACTIONAL) {
          level = isolation_level;
        }
    # 获取当前操作使用哪种类型的intent lock，关于GetStrongIntentTypeSet会单独拿出来分析
    - IntentTypeSet strong_intent_types = GetStrongIntentTypeSet(level, 
            operation_kind, row_mark_type);
    # 如果是serializable的write操作，且需要read snapshot的话，则strong_intent_types
    # 被调整为{IntentType::kStrongRead, IntentType::kStrongWrite}
    - if (isolation_level == IsolationLevel::SERIALIZABLE_ISOLATION &&
        operation_kind == OperationKind::kWrite &&
        doc_op->RequireReadSnapshot()) {
        strong_intent_types = IntentTypeSet({IntentType::kStrongRead, IntentType::kStrongWrite});
      }
    # 遍历GetDocPaths中返回的doc_paths中的每一个doc path，对每一个doc path执行如下：
        # 获取DocKey中的每一部分的前缀长度，包括：hash component所对应的前缀的长度，
        # range component中每一个range column所对应的前缀的长度，以及sub keys所对应
        # 的前缀的长度，以这样的一个DocKey为例：(hash_value, h1, h2, r1, r2, s1, s11)，
        # 则会返回以下的值：
        # 所有hash component所对应的前缀的长度
        # encoded_length(hash_value, h1, h2)
        # 
        # 每个range component所对应的前缀的长度
        # encoded_length(hash_value, h1, h2, r1)
        # encoded_length(hash_value, h1, h2, r1, r2)
        # 
        # subkey s1所对应的前缀的长度
        # encoded_length(hash_value, h1, h2, r1, r2, s1)
        #
        # subkey s11所对应的前缀的长度
        # encoded_length(hash_value, h1, h2, r1, r2, s1, s11)
        #
        # 每一部分所对应的前缀的长度存放在key_prefix_lengths中
        - SubDocKey::DecodePrefixLengths(doc_path.as_slice(), &key_prefix_lengths)
        # key_prefix_lengths中最后一个元素表示整个DocKey的前缀的长度，而在整个DocKey上
        # 要加strong lock，所以直接将key_prefix_lengths中最后一个元素移除
        - key_prefix_lengths.pop_back();
        # 这一步是在整个table上加weak锁？
        - auto partial_key = doc_path;
          if (doc_path.size() > 0 && transactional_table) {
            partial_key.Resize(0);
            RETURN_NOT_OK(ApplyIntent(
                partial_key, StrongToWeak(strong_intent_types), &result.lock_batch));
          }
        # 对DocKey中的每一个前缀部分加锁
        - for (size_t prefix_length : key_prefix_lengths) {
            partial_key.Resize(prefix_length);
            # StrongToWeak：
            # ApplyIntent：去除partial_key中的kGroupEnd部分，然后将partial_key和
            # 它对应的intent_types添加到result.lock_batch中
            RETURN_NOT_OK(ApplyIntent(
                partial_key, StrongToWeak(strong_intent_types), &result.lock_batch));
          }
        # 对整个DocKey进行加锁，加strong lock
        - RETURN_NOT_OK(ApplyIntent(doc_path, strong_intent_types, &result.lock_batch))
        # 如果当前DocPath需要read snapshot，则设置need_read_snapshot为true
        - if (doc_op->RequireReadSnapshot()) {
            result.need_read_snapshot = true;
          }
    # 对read_pairs的处理，对read_pairs中的每个KV pair对应的key及其前缀部分分别加锁，
    # 对完整的key添加strong read lock，而对于key的前缀部分添加weak read lock
    - if (!read_pairs.empty()) {
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
    # 最后返回结果
    - return result;
```

#### 合并LockBatchEntries中相同的key - FilterKeysToLock
```
FilterKeysToLock(LockBatchEntries *keys_locked)
    # 升序排序
    - std::sort(keys_locked->begin(), keys_locked->end(),
        [](const auto& lhs, const auto& rhs) {
          return lhs.key < rhs.key;
        })
    - auto w = keys_locked->begin();
      for (auto it = keys_locked->begin(); ++it != keys_locked->end();) {
        if (it->key == w->key) {
          # 合并相同的key的IntentTypeSet
          w->intent_types |= it->intent_types;
        } else {
          # 对于相同的key，只保留一个
          ++w;
          *w = *it;
        }
      }

      # 清除从w开始，到keys_locked->end()之间的所有的元素
      ++w;
      keys_locked->erase(w, keys_locked->end());
```

#### 获取所需要加的strong lock的类型 - GetStrongIntentTypeSet
如果row_mark_type是合法的RowMarkType(只有在使用PgSql行锁的情况下才有？)，则通过它来确定IntentTypeSet，否则，根据isolation level和operation kind来确定IntentTypeSet。
```
IntentTypeSet GetStrongIntentTypeSet(
    IsolationLevel level,
    OperationKind operation_kind,
    RowMarkType row_mark) {
  if (IsValidRowMarkType(row_mark)) {
    // TODO: possibly adjust this when issue #2922 is fixed.
    switch (row_mark) {
      case RowMarkType::ROW_MARK_EXCLUSIVE: FALLTHROUGH_INTENDED;
      case RowMarkType::ROW_MARK_NOKEYEXCLUSIVE:
        return IntentTypeSet({IntentType::kStrongRead, IntentType::kStrongWrite});
        break;
      case RowMarkType::ROW_MARK_SHARE: FALLTHROUGH_INTENDED;
      case RowMarkType::ROW_MARK_KEYSHARE:
        return IntentTypeSet({IntentType::kStrongRead});
        break;
      default:
        // We shouldn't get here because other row lock types are disabled at the postgres level.
        LOG(DFATAL) << "Unsupported row lock of type " << RowMarkType_Name(row_mark);
        break;
    }
  }

  switch (level) {
    case IsolationLevel::SNAPSHOT_ISOLATION:
      return IntentTypeSet({IntentType::kStrongRead, IntentType::kStrongWrite});
    case IsolationLevel::SERIALIZABLE_ISOLATION:
      switch (operation_kind) {
        case OperationKind::kRead:
          return IntentTypeSet({IntentType::kStrongRead});
        case OperationKind::kWrite:
          return IntentTypeSet({IntentType::kStrongWrite});
      }
      FATAL_INVALID_ENUM_VALUE(OperationKind, operation_kind);
    case IsolationLevel::NON_TRANSACTIONAL:
      LOG(DFATAL) << "GetStrongIntentTypeSet invoked for non transactional isolation";
      return IntentTypeSet();
  }
  FATAL_INVALID_ENUM_VALUE(IsolationLevel, level);
}
```

#### 加锁 - LockBatch
在LockBatch的构造方法中直接加锁。
```
LockBatch::LockBatch(SharedLockManager* lock_manager, LockBatchEntries&& key_to_intent_type,
                     CoarseTimePoint deadline)
    : data_(std::move(key_to_intent_type), lock_manager) {
  # 通过SharedLockManager加锁
  if (!empty() && !lock_manager->Lock(&data_.key_to_type, deadline)) {
    data_.shared_lock_manager = nullptr;
    data_.key_to_type.clear();
    data_.status = STATUS_FORMAT(
        TryAgain, "Failed to obtain locks until deadline: $0", deadline);
  }
}

bool SharedLockManager::Lock(LockBatchEntries* key_to_intent_type, CoarseTimePoint deadline) {
  return impl_->Lock(key_to_intent_type, deadline);
}

bool SharedLockManager::Impl::Lock(LockBatchEntries* key_to_intent_type, CoarseTimePoint deadline) {
  # 确保key_to_intent_type中的每个key在SharedLockManager中都有对应的LockedBatchEntry
  Reserve(key_to_intent_type);
  for (auto it = key_to_intent_type->begin(); it != key_to_intent_type->end(); ++it) {
    const auto& key_and_intent_type = *it;
    const auto intent_types = key_and_intent_type.intent_types;
    # 加锁，key_and_intent_type.locked中记录了其它操作关于当前的key已经持有的锁的情况，
    # intent_types则表示当前操作想要加的锁
    #
    # 见LockedBatchEntry::Lock
    if (!key_and_intent_type.locked->Lock(intent_types, deadline)) {
      # 如果加锁失败，则释放本次操作已经成功加的锁
      while (it != key_to_intent_type->begin()) {
        --it;
        it->locked->Unlock(it->intent_types);
      }
      
      Cleanup(*key_to_intent_type);
      return false;
    }
  }

  return true;
}

void SharedLockManager::Impl::Reserve(LockBatchEntries* key_to_intent_type) {
  std::lock_guard<std::mutex> lock(global_mutex_);
  # 逐一处理key_to_intent_type中的每个LockBatchEntry
  for (auto& key_and_intent_type : *key_to_intent_type) {
    # 看SharedLockManager中是否已经有其它操作对该key加了锁
    auto& value = locks_[key_and_intent_type.key];
    
    # 没有其它操作加过锁的话，则位置分配一个LockedBatchEntry
    if (!value) {
      if (!free_lock_entries_.empty()) {
        # 从可用的LockedBatchEntry资源池中获取一个
        value = free_lock_entries_.back();
        free_lock_entries_.pop_back();
      } else {
        # 临时分配一个
        lock_entries_.emplace_back(std::make_unique<LockedBatchEntry>());
        value = lock_entries_.back().get();
      }
    }
    
    # 增加引用计数
    value->ref_count++;
    
    # 将对应的LockedBatchEntry关联到当前的LockBatchEntry，这样在后续如果需要
    # 访问对应的LockedBatchEntry时，就无须再从SharedLockManager中获取了(需要加锁)
    key_and_intent_type.locked = value;
  }
}

bool LockedBatchEntry::Lock(IntentTypeSet lock_type, CoarseTimePoint deadline) {
  size_t type_idx = lock_type.ToUIntPtr();
  # 在当前key上的锁的持有情况(持有的锁的类型，每种锁的持有者数目)
  auto& num_holding = this->num_holding;
  auto old_value = num_holding.load(std::memory_order_acquire);
  
  # 如果可以成功的加lock_type对应的锁，则需要如何去修改key上锁的持有情况
  auto add = kIntentTypeSetAdd[type_idx];
  for (;;) {
    if ((old_value & kIntentTypeSetConflicts[type_idx]) == 0) {
      # 没有冲突，则可以成功加锁
      # 原子性的修改key上锁的持有情况
      auto new_value = old_value + add;
      
      if (num_holding.compare_exchange_weak(old_value, new_value, std::memory_order_acq_rel)) {
        # 更新成功，则直接返回true
        return true;
      }
      
      # 更新失败，则重试，此时old_value也已经被更新为当前看到的最新的值(compare_exchange_weak保证)
      continue;
    }
    
    # 至此，存在冲突，则进入等待状态
    
    # 增加关于当前key的处于等待状态的操作数目
    num_waiters.fetch_add(1, std::memory_order_release);
    
    # 当从该方法返回时，减少关于当前key的处于等待状态的操作数目
    auto se = ScopeExit([this] {
      num_waiters.fetch_sub(1, std::memory_order_release);
    });
    
    # 条件等待，等待被唤醒
    std::unique_lock<std::mutex> lock(mutex);
    # 重新检查是否还存在冲突
    old_value = num_holding.load(std::memory_order_acquire);
    if ((old_value & kIntentTypeSetConflicts[type_idx]) != 0) {
      # 如果依然存在冲突，则等待
      if (deadline != CoarseTimePoint::max()) {
        # 如果在被唤醒之前超时，则返回false
        if (cond_var.wait_until(lock, deadline) == std::cv_status::timeout) {
          return false;
        }
      } else {
        # 条件等待
        cond_var.wait(lock);
      }
    }
  }
}
```

#### 检查和临时记录之间存在哪些冲突 - OperationConflictResolverContext::ReadConflicts
OperationConflictResolverContext::ReadConflicts执行如下：
- 遍历WriteOperation中的每一个PgsqlWriteOperation，对每一个PgsqlWriteOperation执行如下：
    - 获取当前的PgsqlWriteOperation的对应的doc_paths
        - 当前PgsqlWriteOperation所涉及的每个column对应一个doc path，由DocKey + ColumnId构成
    - 遍历doc_paths中的每个doc_path，对每个doc_path执行如下：
        - 在当前的doc_path及其前缀部分分别应用EnumerateIntentsCallback
            - EnumerateIntentsCallback接收2个参数：当前处理的intent_key_prefix和想要加的IntentTypeSet
            - 在临时记录表中查找以intent_key_prefix作为前缀的操作与intent_key_prefix上想要加的IntentTypeSet之间是否存在冲突
                - 如果存在冲突，则将临时记录对应的transaction id添加到冲突的transaction id列表中
                
所以，OperationConflictResolverContext::ReadConflicts执行之后的结果就是设置ConflictResolver::conflicts_，它里面存放的是所有与当前WriteOperation冲突的临时记录对应的transaction id的集合。

```
OperationConflictResolverContext::ReadConflicts
    # 这是一个回调，用于在枚举操作对应的keys及其前缀部分过程中，检查是否有
    # 临时记录与之冲突
    - EnumerateIntentsCallback callback = [&strong_intent_types, resolver]
        (IntentStrength intent_strength, Slice, KeyBytes* encoded_key_buffer, LastKey) {
        return resolver->ReadIntentConflicts(
          intent_strength == IntentStrength::kStrong ? strong_intent_types
                                                     : StrongToWeak(strong_intent_types),
          encoded_key_buffer);
      };
    # 遍历操作中的每个子操作
    - for (const auto& doc_op : doc_ops_) {
        # 获取当前操作对应的所有的doc paths，保存在doc_paths中
        # 这里使用的是GetDocPathsMode::kIntents模式，获取到doc_paths中包括当前子操作
        # 所涉及的每个Column对应的doc path，也就是DocKey + ColumnId，比如当前子操作
        # 操作两个columns：column1和column2，则对应的doc_paths中包括：
        #   DocKey + ValueType::kColumnId + column1's id
        #   DocKey + ValueType::kColumnId + column2's id
        RETURN_NOT_OK(doc_op->GetDocPaths(GetDocPathsMode::kIntents, &doc_paths, &isolation))
        # 获取对应的strong IntentTypeSet
        strong_intent_types = GetStrongIntentTypeSet(isolation, OperationKind::kWrite,
                                                   RowMarkType::ROW_MARK_ABSENT)
        # 逐一处理当前子操作对应的所有的doc_paths中的每个doc_path               
        for (const auto& doc_path : doc_paths) {
          # 在当前的doc_path及其前缀部分分别应用EnumerateIntentsCallback，对于完整的
          # doc_path上设定IntentStrength是kStrong，而对于doc_path的前缀部分设定的
          # IntentStrength是kWeak
          RETURN_NOT_OK(EnumerateIntents(
            doc_path.as_slice(), Slice(), callback, &encoded_key_buffer,
            PartialRangeKeyIntents::kTrue));
        }                                                   
      }
      
# 在EnumerateIntentsCallback中执行的主体
ConflictResolver::ReadIntentConflicts(IntentTypeSet type, KeyBytes* intent_key_prefix)
    # 在intent DB上创建迭代器
    - EnsureIntentIteratorCreated
        # 创建迭代器的时候会指定查找的最大的key @intent_key_upperbound_
        - intent_iter_ = CreateRocksDBIterator(doc_db_.intents, ..., &intent_key_upperbound_)
    # 获取给定IntentTypeSet对应的冲突掩码
    - const auto conflicting_intent_types = kIntentTypeSetConflicts[type.ToUIntPtr()]
    # 设定intent_key_upperbound_，将在intent DB上的迭代器使用
    - KeyBytes upperbound_key(*intent_key_prefix);
      upperbound_key.AppendValueType(ValueType::kMaxByte);
      intent_key_upperbound_ = upperbound_key.AsSlice()
    # 获取intent_key_prefix的原始大小
    - size_t original_size = intent_key_prefix->size
    # 当前过程是在Intent DB中查找和给定的key：intent_key_prefix的给定的锁类型：type
    # 相冲突的intent记录，根据YugaByte中锁冲突理论：当2个操作所申请的锁中至少有一个
    # 是strong lock，且2个操作所申请的锁必须分别是read锁和write锁，这样才能冲突，
    # 既然，这里我们发现type中不包含strong锁，那么如果要和我冲突，就必须是包含strong
    # 锁的intent才可以，所以修改intent_key_prefix，直接在intent_key_prefix后面加上
    # strong intent lock，这样就可以略过那些不包含strong intent lock的intent记录了
    -   intent_key_prefix->AppendValueType(ValueType::kIntentTypeSet);
        if (!HasStrong(type)) {
          char value = 1 << kStrongIntentFlag;
          intent_key_prefix->AppendRawBytes(&value, 1);
        }
    # 获取原始的intent_key_prefix(因为intent_key_prefix可能再上一步在末尾追加了数据)
    - Slice prefix_slice(intent_key_prefix->AsSlice().data(), original_size)
    # 从intent_key_prefix开始查找intents
    - intent_iter_.Seek(intent_key_prefix->AsSlice())
      while (intent_iter_.Valid()) {
        # 当前intent对应的key
        auto existing_key = intent_iter_.key();
        # 当前intent对应的value
        auto existing_value = intent_iter_.value();
        # 去除existing_value头部的第一个字节(存放的是ValueTypeAsChar::kTransactionId)，
        # 去除之后existing_value的头部的第一个字节就是该intent对应的transaction id
        - existing_value.consume_byte()
        # 返回的是ParsedIntent类型，其中ParsedIntent::doc_path记录对应的intent key，
        # ParsedIntent::types记录对应的IntentTypeSet，ParsedIntent::doc_ht记录对应
        # 的DocHybridTime
        - auto existing_intent = VERIFY_RESULT(
            docdb::ParseIntentKey(intent_iter_.key(), existing_value))
            - ParsedIntent result
              result.doc_path = intent_key
              # 获取DocHybridTime部分占用的字节大小
              RETURN_NOT_OK(DocHybridTime::CheckAndGetEncodedSize(result.doc_path, &doc_ht_size));
              # 移除doc path尾部的四部分：ValueType::kIntentType, the actual intent type, 
              # ValueType::kHybridTime, DocHybridTime
              result.doc_path.remove_suffix(doc_ht_size + 3);
            # 获取doc_path中IntentType，保留在result.types中
            - auto intent_type_and_doc_ht = result.doc_path.end()
              result.types = ObsoleteIntentTypeToSet(intent_type_and_doc_ht[1])
            # 获取DocHybridTime，保留在result.doc_ht中
            - result.doc_ht = Slice(result.doc_path.end() + 2, doc_ht_size + 1)
        # 获取当前intent的IntentTypeSet对应的Mask
        - const auto intent_mask = kIntentTypeSetMask[existing_intent.types.ToUIntPtr()]
        # 如果存在冲突
        - if ((conflicting_intent_types & intent_mask) != 0) {
            # 从existing_value中获取对应的transaction id
            auto transaction_id = VERIFY_RESULT(FullyDecodeTransactionId(
                Slice(existing_value.data(), TransactionId::StaticSize())));
            # 对于OperationConflictResolverContext来说，IgnoreConflictsWith始终为false
            if (!context_->IgnoreConflictsWith(transaction_id)) {
              # 将与当前key存在冲突的操作对应的transaction id记录下来
              conflicts_.insert(transaction_id);
            }
          }
        
        # 继续处理下一个intent
        intent_iter_.Next()
      }    
```

#### 解决可能存在的冲突 - ConflictResolver::ResolveConflicts
ConflictResolver::ResolveConflicts主要执行以下操作：
- 检查是否存在冲突
    - 如果不存在冲突，则直接调用冲突解决之后的回调
    - 否则，存在冲突
        - 将所有冲突的事务记录在ConflictResolver::transactions_中
        - [**解决冲突**]
            - 单独拿出来讲解

[**解决冲突**]
- 检查是否所有存在冲突的transaction已经在在本地commit了
    - 是
        - 直接调用冲突解决之后的回调
    - 否
        - 移除从ConflictResolver::transactions_中移除所有已经在本地commit的transaction
        - 获取所有冲突的transaction的status，如果最近曾经获取过某个transaction的status，且该status是可信的，则直接利用本地获取的status，否则发送GetTransactionStatusRequestPB请求来获取transaction status
        - 当接收到关于某个transaction的GetTransactionStatusRequestPB请求的响应的时候，处理如下：
            - 如果成功获取到某个transaction的status时，会在本地更新该transaction的status，如果是COMMITTED状态，则设置transaction对应的commit time
            - 如果返回TryAgain，则设置transaction status为PENDING
            - 如果返回NotFound，则设置transaction status为ABORTED
            - 其它情况，则设置transaction failure(transaction.failure)
        - 当接收到所有transactions的GetTransactionStatusRequestPB请求的响应时：
            - 移除所有处于COMMITTED或者ABORTED状态的transaction
                - 如果移除之后没有其它冲突的transactions，则认为所有冲突已经解决，直接调用冲突解决之后的回调
                - 否则，还存在冲突的transactions
                    - 对所有剩余的存在冲突的transactions，逐一发送AbortTransactionRequestPB请求给status tablet
                    - 当接收到关于某个transaction的AbortTransactionRequestPB请求的响应的时候，处理如下：
                        - 如果成功abort某个transaction时，则设置相应transaction状态为aborted
                        - 其它情况，则设置transaction failure(transaction.failure)
                    - 当接收到所有transactions的AbortTransactionRequestPB请求的响应时：
                        - 移除所有处于COMMITTED或者ABORTED状态的transaction
                            - 如果移除之后没有其它冲突的transactions，则认为所有冲突已经解决，直接调用冲突解决之后的回调
                            - 否则，还存在冲突的transactions
                                - 再次进入[**解决冲突**]的过程


```
ConflictResolver::ResolveConflicts
    # 如果不存在冲突，则直接调用冲突解决之后的回调
    - if (conflicts_.empty()) {
        callback_(context_->GetResolutionHt());
        return;
      }
      
    # 将所有冲突的事务记录在ConflictResolver::transactions_中
    - transactions_.reserve(conflicts_.size());
      for (const auto& transaction_id : conflicts_) {
        transactions_.push_back({ transaction_id });
      }
    # 解决冲突
    - DoResolveConflicts
    
  void DoResolveConflicts() {
    # 检查冲突是否已经解决，如果所有在之前被认定为存在冲突的操作都已经在本地提交了，
    # 则认为冲突已经解决
    if (CheckResolutionDone(CheckLocalCommits())) {
      return;
    }

    # 获取所有冲突的transaction的status
    FetchTransactionStatuses();
  }
  
  Result<bool> ConflictResolver::CheckLocalCommits() {
    auto write_iterator = transactions_.begin();
    for (const auto& transaction : transactions_) {
      # TransactionParticipant::Impl::LocalCommitTime
      # 查看Tablet的TransactionParticipant是否知道当前事务已经在本地commit了，
      # 如果已经commit，则返回commit timestamp，否则返回invalid timestamp
      auto commit_time = status_manager().LocalCommitTime(transaction.id);
      if (!commit_time.is_valid()) {
        # 只在transactions_中保留那些没有commit的事务
        *write_iterator = transaction;
        ++write_iterator;
        continue;
      }
      
      # 检查是否和已经committed的事务之间存在冲突
      # OperationConflictResolverContext::CheckConflictWithCommitted，在其中会检查
      # OperationConflictResolverContext::resolution_ht_是否比当前transaction的commit
      # time小，如果是，则设置OperationConflictResolverContext::resolution_ht_为当前
      # transaction的commit time
      RETURN_NOT_OK(context_->CheckConflictWithCommitted(transaction.id, commit_time));
    }
    
    # 移除所有已经在本地commit的transaction
    transactions_.erase(write_iterator, transactions_.end());

    # 如果transactions_为空，则表明和之前认为存在冲突的临时记录之间不存在冲突了
    return transactions_.empty();
  }  
  
  MUST_USE_RESULT bool ConflictResolver::CheckResolutionDone(const Result<bool>& result) {
    if (!result.ok()) {
      # 出错的情况下的处理
      callback_(result.status());
      return true;
    }

    if (result.get()) {
      # 所有冲突已经解决，则调用冲突解决之后的回调
      callback_(context_->GetResolutionHt());
      return true;
    }

    # 还存在冲突
    return false;
  }  
  
  void ConflictResolver::FetchTransactionStatuses() {
    static const std::string kRequestReason = "conflict resolution"s;
    auto self = shared_from_this();
    # 记录还有多少个transaction没有接收到关于transaction status的请求的响应
    pending_requests_.store(transactions_.size());
    # 逐一处理每一个被认定存在冲突的transaction
    for (auto& i : transactions_) {
      auto& transaction = i;
      # 生成一个StatusRequest
      StatusRequest request = {
        &transaction.id,
        context_->GetResolutionHt(),
        context_->GetResolutionHt(),
        0, // serial no. Could use 0 here, because read_ht == global_limit_ht.
           // So we cannot accept status with time >= read_ht and < global_limit_ht.
        &kRequestReason,
        TransactionLoadFlags{TransactionLoadFlag::kCleanup},
        # 接收到响应之后的回调
        [self, &transaction](Result<TransactionStatusResult> result) {
          if (result.ok()) {
            # 成功获取到transaction status信息
            transaction.ProcessStatus(*result);
          } else if (result.status().IsTryAgain()) {
            # 重试
            // It is safe to suppose that transaction in PENDING state in case of try again error.
            transaction.status = TransactionStatus::PENDING;
          } else if (result.status().IsNotFound()) {
            # 没有找到对应的transaction，则认为该transaction被abort了
            transaction.status = TransactionStatus::ABORTED;
          } else {
            # 失败
            transaction.failure = result.status();
          }
          
          # 减少没有接收到关于transaction status的请求的响应的transaction数目
          if (self->pending_requests_.fetch_sub(1, std::memory_order_acq_rel) == 1) {
            # 所有transaction status请求都已经处理完毕
            self->FetchTransactionStatusesDone();
          }
        }
      };
      
      # 由TransactionParticipant发送请求给Status tablet来获取transaction status
      status_manager().RequestStatusAt(request);
    }
  }  
```

##### TransactionParticipant发送transaction status请求给Status tablet - TransactionParticipant::Impl::RequestStatusAt
```
  void TransactionParticipant::Impl::RequestStatusAt(const StatusRequest& request) {
    auto lock_and_iterator = LockAndFind(*request.id, *request.reason, request.flags);
    if (!lock_and_iterator.found()) {
      request.callback(
          STATUS_FORMAT(NotFound, "Request status of unknown transaction: $0", *request.id));
      return;
    }
    
    # RunningTransaction::RequestStatusAt中会根据request.global_limit_ht和
    # RunningTransaction::last_known_status_hybrid_time_之间的大小关系，结合
    # RunningTransaction::last_known_status_来推断transaction当前的status，
    # 如果不能推断出来，则通过RunningTransaction::SendStatusRequest来向
    # status tablet发送GetTransactionStatusRequestPB请求来获取transaction status
    lock_and_iterator.transaction().RequestStatusAt(request, &lock_and_iterator.lock);
  }
```

##### 成功获取到transaction status信息时的处理 - TransactionData::ProcessStatus
TransactionData::ProcessStatus会根据获取到的transaction status设置本地记录的transaction status，如果transaction status是COMMITTED，则同时在本地设置transaction对应的commit time。
```
struct TransactionData {
  TransactionId id;
  TransactionStatus status;
  HybridTime commit_time;
  uint64_t priority;
  Status failure;

  void ProcessStatus(const TransactionStatusResult& result) {
    # 设置transaction status
    status = result.status;
    if (status == TransactionStatus::COMMITTED) {
      # 如果是COMMITTED状态，则设置transaction对应的commit time
      commit_time = result.status_time;
    }
  }
};
```

##### 获取到所有transaction的transaction status时的处理 -  ConflictResolver::FetchTransactionStatusesDone

```
  void ConflictResolver::FetchTransactionStatusesDone() {
    if (CheckResolutionDone(ContinueResolve())) {
      return;
    }
  }
  
  Result<bool> ConflictResolver::ContinueResolve() {
    # 移除处于COMMITTED或者ABORTED状态的transaction，如果移除之后没有其它
    # 冲突的transaction，则Cleanup()返回true，但是则VERIFY_RESULT会返回true
    if (VERIFY_RESULT(Cleanup())) {
      return true;
    }

    # 至此，还存在与之存在冲突的transactions
    # OperationConflictResolverContext::CheckPriority始终返回Status::OK()，
    # 也就是说单行事务的优先级始终是最高的？
    RETURN_NOT_OK(context_->CheckPriority(this, &transactions_));

    # abort剩余的所有存在冲突的transactions
    AbortTransactions();
    return false;
  }
  
  # 移除ABORTED和COMMITTED状态的transaction，保留其它状态的transaction
  Result<bool> ConflictResolver::Cleanup() {
    auto write_iterator = transactions_.begin();
    for (const auto& transaction : transactions_) {
      RETURN_NOT_OK(transaction.failure);
      auto status = transaction.status;
      if (status == TransactionStatus::COMMITTED) {
        # 处于COMMITTED状态的transaction被移除
        RETURN_NOT_OK(context_->CheckConflictWithCommitted(
            transaction.id, transaction.commit_time));
        continue;
      } else if (status == TransactionStatus::ABORTED) {
        # 处于ABORTED状态的transaction被移除
        auto commit_time = status_manager().LocalCommitTime(transaction.id);
        if (commit_time.is_valid()) {
          RETURN_NOT_OK(context_->CheckConflictWithCommitted(transaction.id, commit_time));
        } else {
          VLOG_WITH_PREFIX(4) << "Aborted: " << transaction.id;
        }
        continue;
      } else {
        DCHECK(TransactionStatus::PENDING == status ||
               TransactionStatus::APPLYING == status)
            << "Actual status: " << TransactionStatus_Name(status);
      }
      
      *write_iterator = transaction;
      ++write_iterator;
    }
    transactions_.erase(write_iterator, transactions_.end());

    return transactions_.empty();
  }
  
  
  
  void ConflictResolver::AbortTransactions() {
    auto self = shared_from_this();
    # 记录关于待abort的transaction的请求的总数目
    pending_requests_.store(transactions_.size());
    for (auto& i : transactions_) {
      auto& transaction = i;
      # TransactionParticipant::Abort -> TransactionParticipant::Impl::Abort
      status_manager().Abort(
          transaction.id,
          # abort waiter，在发送abort请求之前会将其添加到关于abort的等待队列中，
          # 当接收到abort响应的时候会在RunningTransaction::AbortReceived中被调用
          [self, &transaction](Result<TransactionStatusResult> result) {
        if (result.ok()) {
          # 成功abort，设置transaction状态为aborted
          transaction.ProcessStatus(*result);
        } else if (result.status().IsRemoteError()) {
          transaction.failure = result.status();
        } else {
          LOG(INFO) << self->LogPrefix() << "Abort failed, would retry: " << result.status();
        }
        
        # 接收到abort请求的响应，减少关于待abort的transaction的请求的数目
        if (self->pending_requests_.fetch_sub(1, std::memory_order_acq_rel) == 1) {
          # 接收到了所有关于待abort的transaction的请求的响应
          # ConflictResolver::AbortTransactionsDone
          self->AbortTransactionsDone();
        }
      });
    }
  }
  
  void TransactionParticipant::Impl::Abort(const TransactionId& id, TransactionStatusCallback callback) {
    auto lock_and_iterator = LockAndFind(
        id, "abort"s, TransactionLoadFlags{TransactionLoadFlag::kMustExist});
    if (!lock_and_iterator.found()) {
      callback(STATUS_FORMAT(NotFound, "Abort of unknown transaction: $0", id));
      return;
    }
    auto client_result = client();
    if (!client_result.ok()) {
      callback(client_result.status());
      return;
    }
    
    # RunningTransaction::Abort，向status tablet发送abort请求
    lock_and_iterator.transaction().Abort(
        *client_result, std::move(callback), &lock_and_iterator.lock);
  }
  
void RunningTransaction::Abort(client::YBClient* client,
                               TransactionStatusCallback callback,
                               std::unique_lock<std::mutex>* lock) {
  if (last_known_status_ == TransactionStatus::ABORTED ||
      last_known_status_ == TransactionStatus::COMMITTED) {
    # ABORTED或者COMMITTED状态已经是最终的状态了，不要再发送abort请求给status tablet了
    TransactionStatusResult status{last_known_status_, last_known_status_hybrid_time_};
    lock->unlock();
    callback(status);
    return;
  }
  
  bool was_empty = abort_waiters_.empty();
  abort_waiters_.push_back(std::move(callback));
  lock->unlock();
  if (!was_empty) {
    # 已经有其它地方发送过了abort请求，无需再次发送
    return;
  }
  
  # 发送abort请求给status tablet
  tserver::AbortTransactionRequestPB req;
  req.set_tablet_id(metadata_.status_tablet);
  req.set_transaction_id(metadata_.transaction_id.data(), metadata_.transaction_id.size());
  req.set_propagated_hybrid_time(context_.participant_context_.Now().ToUint64());
  context_.rpcs_.RegisterAndStart(
      client::AbortTransaction(
          TransactionRpcDeadline(),
          nullptr /* tablet */,
          client,
          &req,
          # 接收到abort请求的响应时的回调，在其中设置RunningTransaction::last_known_status_
          # 和RunningTransaction::last_known_status_hybrid_time_，并且唤醒所有处于abort
          # 的等待队列中的waiter
          std::bind(&RunningTransaction::AbortReceived, this, _1, _2, shared_from_this())),
      &abort_handle_);
}

  void ConflictResolver::AbortTransactionsDone() {
    # 移除处于COMMITTED或者ABORTED状态的transaction，如果移除之后没有其它
    # 冲突的transaction，则Cleanup()返回true
    if (CheckResolutionDone(Cleanup())) {
      return;
    }

    # 至此，还存在冲突
    # 继续解决冲突
    DoResolveConflicts();
  }
  
  MUST_USE_RESULT bool ConflictResolver::CheckResolutionDone(const Result<bool>& result) {
    if (!result.ok()) {
      # 出错的情况下的处理
      callback_(result.status());
      return true;
    }

    if (result.get()) {
      # 所有冲突已经解决，则调用冲突解决之后的回调
      callback_(context_->GetResolutionHt());
      return true;
    }

    # 还存在冲突
    return false;
  }  
```


#### 冲突解决之后的回调
对于单行事务，冲突解决后的回调时在DocWriteOperation::DoStart -> ResolveOperationConflicts中设置的。
```
  CHECKED_STATUS DocWriteOperation::DoStart() {
    ...

    if (isolation_level_ == IsolationLevel::NON_TRANSACTIONAL) {
      auto now = tablet_.clock()->Now();
      docdb::ResolveOperationConflicts(
          operation_->doc_ops(), now, tablet_.doc_db(), partial_range_key_intents,
          transaction_participant,
          # 冲突解决之后的回调
          [self = shared_from_this(), now](const Result<HybridTime>& result) {
            if (!result.ok()) {
              # 如果发生错误，则执行DocWriteOperationCallback，这是在
              # Tablet::KeyValueBatchFromPgsqlWriteBatch -> Tablet::StartDocWriteOperation
              # 中设置的，暂不关注
              self->InvokeCallback(result.status());
              return;
            }
            
            # DocWriteOperation::NonTransactionalConflictsResolved
            self->NonTransactionalConflictsResolved(now, *result);
          });
      return Status::OK();
    }
  }
  
  void NonTransactionalConflictsResolved(HybridTime now, HybridTime result) {
    # 更新本地时钟
    if (now != result) {
      tablet_.clock()->Update(result);
    }

    # 见下面
    Complete();
  }
  
  void DocWriteOperation::Complete() {
    # 当DoComplete执行之后会调用InvokeCallback，这个callback是在
    # Tablet::KeyValueBatchFromPgsqlWriteBatch -> Tablet::StartDocWriteOperation -> 
    # 创建DocWriteOperation是设置的
    InvokeCallback(DoComplete());
  }
  
  CHECKED_STATUS DocWriteOperation::DoComplete() {
    # prepare_result_.need_read_snapshot是在PrepareDocWriteOperation中设置的
    # 对于PgsqlWriteOperation类型的操作来说，目前只有UPSERT操作不需要read snapshot
    auto read_op = prepare_result_.need_read_snapshot
        ? VERIFY_RESULT(ScopedReadOperation::Create(&tablet_, RequireLease::kTrue, read_time_))
        : ScopedReadOperation();
    // Actual read hybrid time used for read-modify-write operation.
    auto real_read_time = prepare_result_.need_read_snapshot
        ? read_op.read_time()
        // When need_read_snapshot is false, this time is used only to write TTL field of record.
        : ReadHybridTime::SingleTime(tablet_.clock()->Now());

    // We expect all read operations for this transaction to be done in ExecuteDocWriteOperation.
    // Once read_txn goes out of scope, the read point is deregistered.
    HybridTime restart_read_ht;
    bool local_limit_updated = false;

    // This loop may be executed multiple times multiple times only for serializable isolation or
    // when read_time was not yet picked for snapshot isolation.
    // In all other cases it is executed only once.
    InitMarkerBehavior init_marker_behavior = tablet_.table_type() == TableType::REDIS_TABLE_TYPE
        ? InitMarkerBehavior::kRequired
        : InitMarkerBehavior::kOptional;
    for (;;) {
      RETURN_NOT_OK(docdb::ExecuteDocWriteOperation(
          operation_->doc_ops(), operation_->deadline(), real_read_time, tablet_.doc_db(),
          operation_->request()->mutable_write_batch(), init_marker_behavior,
          tablet_.monotonic_counter(), &restart_read_ht,
          tablet_.metadata()->table_name()));

      // For serializable isolation we don't fix read time, so could do read restart locally,
      // instead of failing whole transaction.
      if (!restart_read_ht.is_valid() || !allow_immediate_read_restart()) {
        break;
      }

      real_read_time.read = restart_read_ht;
      if (!local_limit_updated) {
        local_limit_updated = true;
        real_read_time.local_limit =
            std::min(real_read_time.local_limit, tablet_.SafeTime(RequireLease::kTrue));
      }

      restart_read_ht = HybridTime();

      operation_->request()->mutable_write_batch()->clear_write_pairs();

      for (auto& doc_op : operation_->doc_ops()) {
        doc_op->ClearResponse();
      }
    }

    operation_->SetRestartReadHt(restart_read_ht);

    if (allow_immediate_read_restart() &&
        isolation_level_ != IsolationLevel::NON_TRANSACTIONAL &&
        operation_->response()) {
      real_read_time.ToPB(operation_->response()->mutable_used_read_time());
    }

    if (operation_->restart_read_ht().is_valid()) {
      return Status::OK();
    }

    # 将当前操作涉及的所有的锁信息记录在WriteOperationState中，当WriteOperation成功
    # 写入intent DB中的临时记录之后就会释放这些锁(见OperationDriver::ReplicationFinished ->
    # Tablet::ApplyRowOperations -> WriteOperationState::ReplaceDocDBLocks)
    operation_->state()->ReplaceDocDBLocks(std::move(prepare_result_.lock_batch));

    return Status::OK();
  }  

Status ExecuteDocWriteOperation(const vector<unique_ptr<DocOperation>>& doc_write_ops,
                                CoarseTimePoint deadline,
                                const ReadHybridTime& read_time,
                                const DocDB& doc_db,
                                KeyValueWriteBatchPB* write_batch,
                                InitMarkerBehavior init_marker_behavior,
                                std::atomic<int64_t>* monotonic_counter,
                                HybridTime* restart_read_ht,
                                const string& table_name) {
  DCHECK_ONLY_NOTNULL(restart_read_ht);
  DocWriteBatch doc_write_batch(doc_db, init_marker_behavior, monotonic_counter);
  DocOperationApplyData data = {&doc_write_batch, deadline, read_time, restart_read_ht};
  for (const unique_ptr<DocOperation>& doc_op : doc_write_ops) {
    # PgsqlWriteOperation::Apply，咱们先以insert为例进行分析
    # PgsqlWriteOperation::Apply会将当前的DocOperation @doc_op所需要记录的所有的
    # 临时记录都保存到DocWriteBatch类型的doc_write_batch中，具体来说，是保存到
    # DocWriteBatch::cache中
    # 
    # PgsqlWriteOperation::Apply会在PgsqlWriteOperation中单独分析
    Status s = doc_op->Apply(data);
    RETURN_NOT_OK(s);
  }
  
  # 将DocWriteBatch中相关信息转移到KeyValueWriteBatchPB中，实际上是转移到了
  # WriteOperation中的WriteRequestPB中的write_batch_这个KeyValueWriteBatchPB中
  doc_write_batch.MoveToWriteBatchPB(write_batch);
  return Status::OK();
}
```

当DoComplete执行之后会调用InvokeCallback，这个callback是在Tablet::KeyValueBatchFromPgsqlWriteBatch -> Tablet::StartDocWriteOperation -> 创建DocWriteOperation过程中设置的，如下：
```
void Tablet::KeyValueBatchFromPgsqlWriteBatch(std::unique_ptr<WriteOperation> operation) {
  ...
  
  StartDocWriteOperation(std::move(operation), std::move(scoped_read_operation),
                         [](auto operation, const Status& status) {
    if (!status.ok() || operation->restart_read_ht().is_valid()) {
      WriteOperation::StartSynchronization(std::move(operation), status);
      return;
    }
    
    # 所有的DocOperation
    auto& doc_ops = operation->doc_ops();

    for (size_t i = 0; i < doc_ops.size(); i++) {
      PgsqlWriteOperation* pgsql_write_op = down_cast<PgsqlWriteOperation*>(doc_ops[i].get());
      // We'll need to return the number of rows inserted, updated, or deleted by each operation.
      doc_ops[i].release();
      # WriteOperation对应的WriteOperationState::pgsql_write_ops_记录的是当前
      # WriteOperation所涉及的所有的PgsqlWriteOperation
      operation->state()->pgsql_write_ops()
                        ->emplace_back(unique_ptr<PgsqlWriteOperation>(pgsql_write_op));
    }

    # 会进一步调用WriteOperation::DoStartSynchronization
    WriteOperation::StartSynchronization(std::move(operation), Status::OK());
  });
}

void WriteOperation::DoStartSynchronization(const Status& status) {
  std::unique_ptr<WriteOperation> self(this);
  // If a restart read is required, then we return this fact to caller and don't perform the write
  // operation.
  if (restart_read_ht_.is_valid()) {
    auto restart_time = state()->response()->mutable_restart_read_time();
    restart_time->set_read_ht(restart_read_ht_.ToUint64());
    auto local_limit = context_->ReportReadRestart();
    restart_time->set_local_limit_ht(local_limit.ToUint64());
    // Global limit is ignored by caller, so we don't set it.
    state()->CompleteWithStatus(Status::OK());
    return;
  }

  if (!status.ok()) {
    state()->CompleteWithStatus(status);
    return;
  }

  # context_是在WriteOperation的构造方法中设置的TabletPeer，所以这里就是
  # TabletPeer::Submit，从这里会提交给raft执行，会单独拿出来分析
  context_->Submit(std::move(self), term_);
}
```

### 提交给raft执行
```
TabletPeer::Submit
    # 检查当前tablet的运行状态
    - auto status = CheckRunning()
    # 如果是处于running状态，则创建并初始化一个OperationDriver，然后
    - if (status.ok()) {
        auto driver = NewLeaderOperationDriver(&operation, term);
        if (driver.ok()) {
          (**driver).ExecuteAsync();
        } else {
          status = driver.status();
        }
      }
    # 如果tablet不是处于running状态，或者在初始化OperationDriver的过程中出错，则abort当前操作
    - if (!status.ok()) {
        # Operation::Aborted -> WriteOperation::DoAborted -> WriteOperationState::Abort
        operation->Aborted(status);
      }
```

#### 创建并初始化OperationDriver - TabletPeer::NewLeaderOperationDriver
```
TabletPeer::NewLeaderOperationDriver(std::unique_ptr<Operation>* operation, int64_t term)
    - TabletPeer::NewOperationDriver
        - TabletPeer::CreateOperationDriver
            - new OperationDriver(...)
                - 设置replication_state_为NOT_REPLICATING
                # 设置prepare_state_为NOT_PREPARED
        - OperationDriver::Init(operation, term)
            # 将Operation关联到OperationDriver中
            - operation_ = std::move(*operation)
            # 在当前上下文中对应的是WriteOperation::NewReplicateMsg
            - consensus::ReplicateMsgPtr replicate_msg = operation_->NewReplicateMsg()
                - 会分配一个ReplicateMsg，并设置它的操作类型是WRITE_OP，并将WriteOperationState中的WriteRequestPB关联到该ReplicateMsg中
            # 分配一个ConsensusRound并将ReplicateMsg关联到其中，设置该ConsensusRound
            # 被成功复制之后的回调是OperationDriver::ReplicationFinished
            # WriteOperationState::consensus_round_
            - mutable_state()->set_consensus_round(
                consensus_->NewRound(std::move(replicate_msg),
                    std::bind(&OperationDriver::ReplicationFinished, this, _1, _2, _3)));
              # 设置ConsensusRound对应的term，如果term发生变更，则该ConsensusRound不被执行
              mutable_state()->consensus_round()->BindToTerm(term);
              # 设置当ConsensusRound被添加到Consensus的pending队列中(还没有开始raft
              # 复制)时候的回调
              mutable_state()->consensus_round()->SetAppendCallback(this);
            # 添加到OperationTracker中
            - auto result = operation_tracker_->Add(this)
```

#### 提交OperationDriver给Raft - OperationDriver::ExecuteAsync
```
OperationDriver::ExecuteAsync
    - auto s = preparer_->Submit(this)
        - PreparerImpl::Submit
            # 添加到Preparer线程的队列中，最终由PreparerImpl::Run负责执行
            - queue_.Push(operation_driver)
    # 对于WriteOperation来说，什么也没做
    - operation_->SubmittedToPreparer()
    
PreparerImpl::Run
    - 最外层是一个无限循环，在无限循环内执行如下：
        - 逐一处理PreparerImpl::queue_中的每一个OperationDriver，对每一个OperationDriver调用ProcessItem：
            - 如果当前操作需要在leader上执行(replication_state_是NOT_REPLICATING)，则：
                - 为了实现批量提交操作给raft，它会保证尽可能多的操作添加到batch中批量执行，但是这个batch必须满足一些条件：比如batch大小不超过阈值，batch中所有的OperationDriver对应的consensus round的term必须是一致的，另外某些操作可能需要单独执行，如果添加到了batch中且batch大小达到阈值，或者当前操作必须单独执行，则都会立即执行ProcessAndClearLeaderSideBatch
                    - 逐一遍历batch中的每一个OperationDriver，对每个OperationDriver执行如下：
                        - Status s = operation_driver->PrepareAndStart()
                            # 在当前上下文中对应的是WriteOperation::Prepare，什么都没做
                            - operation_->Prepare()
                            - 修改OperationDriver::prepare_state_为PREPARED
                            - 修改OperationDriver::replication_state_为REPLICATING
                    - ReplicateSubBatch(replication_subbatch_begin, replication_subbatch_end)
                        - 将由replication_subbatch_begin, replication_subbatch_end这两个迭代器确定的所有的OperationDriver对应的ConsensusRound添加到PreparerImpl::rounds_to_replicate_中
                        # 将PreparerImpl::rounds_to_replicate_中所有ConsensusRound提交给raft复制
                        - consensus_->ReplicateBatch(&rounds_to_replicate_)
                            - 检查所有的ConsensusRounds对应的term是否跟当前的term一致，如果不一致，则返回错误
                            # 将所有的ConsensusRounds添加到RaftConsensus的队列中
                            - AppendNewRoundsToQueueUnlocked
                                # 分配一个存放ReplicateMsg的数组，用于存放所有ConsensusRound对应的ReplicateMsg
                                - std::vector<ReplicateMsgPtr> replicate_msgs;
                                  replicate_msgs.reserve(rounds.size());
                                - 逐一处理ConsensusRounds中的每个ConsensusRound，对每个ConsensusRound执行如下：
                                    # 借助于ReplicaState为当前的ConsensusRound分配一个OpId，并保存在ReplicateMsg中
                                    - state_->NewIdUnlocked(round->replicate_msg()->mutable_id())
                                    # 获取当前ConsensusRound对应的ReplicateMsg
                                    - ReplicateMsg* const replicate_msg = round->replicate_msg().get()
                                    # 根据ReplicaState中的last committed OpId来设置ReplicateMsg::committed_op_id_
                                    # YugaByte会在每条raft log中包含last committed OpId
                                    - state_->GetCommittedOpIdUnlocked().ToPB(replicate_msg->mutable_committed_op_id())
                                    # 调用ConsensusRound被添加到Consensus的pending队列后的回调(下面会添加到pending队列中)
                                    - auto* const append_cb = round->append_callback()
                                      append_cb->HandleConsensusAppend()
                                      Status s = state_->AddPendingOperation(round)
                                        - 如果当前tablet peer的角色是leader，则检查lease status
                                            - 如果lease status为LeaderLeaseStatus::OLD_LEADER_MAY_HAVE_LEASE，则表示当前leader还没有获取到lease，退出
                                        - 如果是写操作，则将对应的ConsensusRound添加到ReplicaState::retryable_requests_中
                                        - 将对应的ConsensusRound添加到ReplicaState::pending_operations_中
                                    # 添加到ReplicateMsgs中
                                    - replicate_msgs.push_back(round->replicate_msg())
                                # 将replicate_msgs数组中的所有的ReplicateMsg添加到PeerMessageQueue中，
                                # 并写入本地raft log，这些ReplicateMsgs会保存在PeerMessageQueue::log_cache_
                                # 中，当成功添加到本地raft log时，PeerMessageQueue::LocalPeerAppendFinished
                                # 会被调用
                                - Status s = queue_->AppendOperations(
                                replicate_msgs, state_->GetCommittedOpIdUnlocked(), state_->Clock().Now())
                                # 更新ReplicaState中记录的last received OpId
                                - state_->UpdateLastReceivedOpIdUnlocked(replicate_msgs.back()->id())
                            # 向所有的tablet peers发送复制请求
                            - peer_manager_->SignalRequest(RequestTriggerMode::kNonEmptyOnly)
                                - 遍历每一个tablet peer，并调用Peer::SignalRequest
                                    # Peer::SignalRequest会进一步调用Peer::SendNextRequest
                                    - 会从PeerMessageQueue::log_cache_中读取待复制的ReplicateMsgs，填充到ConsensusRequestPB中，然后异步的发送之
            - 否则，当前操作不应该在leader上执行，则：
                # 先处理leader上batch(PreparerImpl::leader_side_batch_)中的操作
                - ProcessAndClearLeaderSideBatch
                # 再处理当前这个操作，调用的是OperationDriver::PrepareAndStartTask
                - item->PrepareAndStartTask()
                    - OperationDriver::PrepareAndStart
        - 当处理完PreparerImpl::queue_中的所有OperationDrivers之后，调用ProcessAndClearLeaderSideBatch来处理
            - 将leader上batch(PreparerImpl::leader_side_batch_)中的操作提交给raft执行
```

#### 当某个ConsensusRound被添加到Consensus的pending队列后的回调
```
OperationDriver::HandleConsensusAppend
    - StartOperation
        - operation_->Start()
            # 在当前上下文中是WriteOperation::DoStart
            - WriteOperation::DoStart
                - state()->tablet()->StartOperation(state())
                    - HybridTime ht = operation_state->hybrid_time_even_if_unset()
                    # ht初始状态为HybridTime::kInvalid
                    - bool was_valid = ht.is_valid();
                      if (!was_valid) {
                        # MvccManager::AddPending，主要是为当前的操作分配一个hybrid time，并且将
                        # 该hybrid time添加到MvccManager中的有序的hybrid time队列中
                        mvcc_.AddPending(&ht);
                        # 设置WriteOperationState::hybrid_time_，作为这个transaction的hybrid time
                        operation_state->set_hybrid_time(ht);
                      }
        - op_id_copy_.store(yb::OpId::FromPB(operation_->state()->op_id()), boost::memory_order_release)
    - auto* const replicate_msg = operation_->state()->consensus_round()->replicate_msg().get();
      replicate_msg->set_hybrid_time(operation_->state()->hybrid_time().ToUint64());
      replicate_msg->set_monotonic_counter(*operation_->state()->tablet()->monotonic_counter())
      
MvccManager::AddPending
    # 如果hybrid time是invalid，则表示是leader，否则是follower
    - const bool is_follower_side = ht->is_valid();
    - 如果是leader，则：
        # 获取当前的hybrid time，并设置给ht
        - *ht = clock_->Now();
    # 添加到关于被tracked操作的hybrid time队列中，该队列中所有hybrid time应该是有序的，
    # 所以在添加到队列之前会进行一系列的检查，这里暂时略过了这些检查
    - queue_.push_back(*ht)
```

#### 当接收到本地或者其它peers的复制响应之后的处理
```
PeerMessageQueue::ResponseFromPeer
    - 暂略
```


#### 当某个ConsensusRound Raft复制完成之后的回调
ConsensusRound成功进行Raft复制之后的回调是OperationDriver::ReplicationFinished，它的调用栈如下：
```
PeerMessageQueue::ResponseFromPeer
PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
RaftConsensus::UpdateMajorityReplicated
ReplicaState::UpdateMajorityReplicatedUnlocked
ReplicaState::AdvanceCommittedOpIdUnlocked
ReplicaState::ApplyPendingOperationsUnlocked
ReplicaState::NotifyReplicationFinishedUnlocked
ConsensusRound::NotifyReplicationFinished
OperationDriver::ReplicationFinished


OperationDriver::ReplicationFinished
    - 根据复制成功与否设置OperationDriver::replication_state_，如果成功，则设置为REPLICATED，否则，设置为REPLICATION_FAILED，并设置OperationDriver::operation_status_为相应的错误状态
    - 读取当前的OperationDriver::prepare_state_，如果当前是PREPARED状态，则执行ApplyOperation
        - OperationDriver::ApplyOperation(leader_term, applied_op_ids)
            - 如果OperationDriver::operation_status_显示为ok，则：
                # 检查操作是否是按顺序执行的
                - order_verifier_->CheckApply(op_id_copy_.load(boost::memory_order_relaxed).index, prepare_physical_hybrid_time_)
                # 将数据写入rocksdb，并释放锁
                - ApplyTask(leader_term, applied_op_ids)
                    # Operation::Replicated
                    - auto status = operation_->Replicated(leader_term)
                        - Status complete_status = Status::OK()
                        # 在当前上下文对应的是WriteOperation::DoReplicated
                        - DoReplicated(leader_term, &complete_status)
                            - *complete_status = state()->tablet()->ApplyRowOperations(state())
                                # 从ReplicateMsg中获取写请求，或者从WriteOperationState中获取写请求，
                                # 获取到的写请求类型为WriteRequestPB
                                - const auto& write_request =
                                  operation_state->consensus_round() && operation_state->consensus_round()->replicate_msg()
                                      // Online case.
                                      ? operation_state->consensus_round()->replicate_msg()->write_request()
                                      // Bootstrap case.
                                      : *operation_state->request()
                                # 从WriteRequestPB中解析出KeyValueWriteBatchPB，它是在DocWriteOperation::DoComplete ->
                                # ExecuteDocWriteOperation中被设置的
                                - const KeyValueWriteBatchPB& put_batch = write_request.write_batch()
                                - return ApplyOperationState(*operation_state, write_request.batch_idx(), put_batch)
                                    # 设置ConsensusFrontiers中的OpId和hybrid time
                                    - docdb::ConsensusFrontiers frontiers;
                                      set_op_id(yb::OpId::FromPB(operation_state.op_id()), &frontiers);
                                      # 对于WriteOperationState来说，WriteHybridTime和hybrid_time可能不同
                                      auto hybrid_time = operation_state.WriteHybridTime();
                                      set_hybrid_time(operation_state.hybrid_time(), &frontiers);
                                        - Tablet::ApplyKeyValueRowOperations(batch_idx, write_batch, &frontiers, hybrid_time)
                            - state()->Commit()
                                # 检查确认MvccManager::queue_中头部第一个hybrid time就是当前这个操作对应的hybrid time，
                                # 并从MvccManager::queue_中移除该hybrid time，同时设置last_replicated hybrid time
                                - tablet()->mvcc_manager()->Replicated(hybrid_time_)
                                    - CHECK_EQ(queue_.front(), ht) << InvariantViolationLogPrefix()
                                    - PopFront(&lock)
                                    - last_replicated_ = ht
                                    # 唤醒因为获取safe time而导致的等待
                                    - cond_.notify_all()
                                - ReleaseDocDbLocks
                                    # docdb_locks_是在WriteOperationState::ReplaceDocDBLocks中设置的
                                    - docdb_locks_.Reset()
                                        # 解除锁
                                        - data_.shared_lock_manager->Unlock(data_.key_to_type)
                                        # 清除key到IntentTypeSet的映射
                                        - data_.key_to_type.clear()
                                - ResetRpcFields
                        - state()->CompleteWithStatus(complete_status)
                            # 如果有completion callback的话，则调用之
                            # 对于WriteOperation来说，对应的是WriteOperationCompletionCallback，
                            # 最终调用的是WriteOperationCompletionCallback::OperationCompleted
                            # 主要是填充响应内容，并返回响应
                            - completion_clbk_->CompleteWithStatus(status)
            - 否则：
                - HandleFailure

Tablet::ApplyKeyValueRowOperations                
    - rocksdb::WriteBatch write_batch
    - 如果是事务性操作
        - PrepareTransactionWriteBatch(batch_idx, put_batch, hybrid_time, &write_batch)
            # 解析出transaction id
            - auto transaction_id = CHECK_RESULT(
                FullyDecodeTransactionId(put_batch.transaction().transaction_id()))
            # 添加一条关于transaction metadata的记录到rocksdb::WriteBatch中，记录
            # 格式为：{{transaction_id}, {TransactionMetadata}}，其中TransactionMetadata
            # 中包括status tablet，isolation level等信息
            - transaction_participant()->Add(put_batch.transaction(), rocksdb_write_batch)
                - TransactionParticipant::Impl::Add
                    - auto metadata = TransactionMetadata::FromPB(data)
                    # 只有在bootstrap过程中才需要wait？
                    - WaitLoaded(metadata->transaction_id);
                    # 先在TransactionParticipant::Impl::transactions_中查找，如果没有，
                    # 则插入一个新的RunningTransaction，如果查找到，则直接返回
                    - std::lock_guard<std::mutex> lock(mutex_);
                      auto it = transactions_.find(metadata->transaction_id);
                      if (it == transactions_.end()) {
                        if (WasTransactionRecentlyRemoved(metadata->transaction_id)) {
                          return false;
                        }
                        
                        # 插入一个新的RunningTransaction
                        transactions_.insert(std::make_shared<RunningTransaction>(
                            *metadata, TransactionalBatchData(), OneWayBitmap(), this));
                        TransactionsModifiedUnlocked(&min_running_notifier);
                      }
                    # 添加一条记录：{{transaction_id}, {TransactionMetadata}}
                    - docdb::KeyBytes key;
                      AppendTransactionKeyPrefix(metadata->transaction_id, &key)
                      auto data_copy = data;
                      data_copy.set_metadata_write_time(GetCurrentTimeMicros());
                      auto value = data.SerializeAsString();
                      write_batch->Put(key.data(), value)
            - boost::container::small_vector<uint8_t, 16> encoded_replicated_batch_idx_set;
              auto prepare_batch_data = transaction_participant()->PrepareBatchData(
                  transaction_id, batch_idx, &encoded_replicated_batch_idx_set)
                - TransactionParticipant::Impl::PrepareBatchData
                    # 查找TransactionParticipant中是否存在该事务，如果存在，则返回指向它的iterator
                    # 和一个锁(TransactionParticipant::Impl中的mutex，实际上是RunningTransactionContext
                    # 中的mutex)，这样这个锁没有被释放，所以共享这个锁的其它操作都必须等待该锁的释放
                    - auto lock_and_iterator = LockAndFind(
                        id, "metadata with write id"s, TransactionLoadFlags{TransactionLoadFlag::kMustExist})
                    # 获取iterator所指向的transaction
                    - auto& transaction = lock_and_iterator.transaction();
                    # 使用batch_idx来更新RunningTransaction::replicated_batches_，并且将更新后的
                    # RunningTransaction::replicated_batches_编码到encoded_replicated_batches中
                    - transaction.AddReplicatedBatch(batch_idx, encoded_replicated_batches
                    # 返回isolation level和transaction中上一次更新的那个batch对应的TransactionalBatchData，
                    # TransactionalBatchData中包含上一次更新的那个batch对应的write_id和hybrid_time，关于
                    # RunningTransaction::last_batch_data_的设置见TransactionParticipant::Impl::BatchReplicated ->
                    # RunningTransaction::BatchReplicated
                    - return std::make_pair(transaction.metadata().isolation, transaction.last_batch_data())
            # 从prepare_batch_data中获取isolation level和last_batch_data(当前transaction上一次
            # 更新的那个batch对应的write_id和hybrid_time)
            - auto isolation_level = prepare_batch_data->first;
              auto& last_batch_data = prepare_batch_data->second;
              yb::docdb::PrepareTransactionWriteBatch(
                  put_batch, hybrid_time, rocksdb_write_batch, transaction_id, isolation_level,
                  UsePartialRangeKeyIntents(*metadata_),
                  Slice(encoded_replicated_batch_idx_set.data(), encoded_replicated_batch_idx_set.size()),
                  # last_batch_data.write_id可能在PrepareTransactionWriteBatch中被更新，它代表的是当前
                  # transaction中上一个strong write intent的编号，初始值为0，transaction中写intent记录
                  # 的时候对于完整的DocKey会记录一条strong intent，而对于DocKey的所有前缀部分会分别记录
                  # 一条weak intent，这个write_id只在记录strong intent的时候被加1.
                  &last_batch_data.write_id);
                  - RowMarkType row_mark = GetRowMarkTypeFromPB(put_batch);
                    # PrepareTransactionWriteBatchHelper用于向rocksdb::WriteBatch中填充数据，
                    # 关于PrepareTransactionWriteBatchHelper，单独讲述
                    PrepareTransactionWriteBatchHelper helper(
                      hybrid_time, rocksdb_write_batch, transaction_id, replicated_batches_state, write_id);
                    # 在这里会设置PrepareTransactionWriteBatchHelper::strong_intent_types_
                    helper.Setup(isolation_level, OperationKind::kWrite, row_mark);
                    # 遍历每一个KeyValuePairPB，对于每一个KV pair中的key及其key的前缀部分分别运用
                    # PrepareTransactionWriteBatchHelper::operator()进行处理，对于key自身使用的是
                    # strong级别的锁，而对于key的前缀部分使用的是weak级别的锁
                    # 
                    # 关于PrepareTransactionWriteBatchHelper，单独讲述
                    CHECK_OK(EnumerateIntents(put_batch.write_pairs(), std::ref(helper),
                             partial_range_key_intents));
                    if (!put_batch.read_pairs().empty()) {
                      # 对于read pairs部分做和write pairs类似的处理
                      helper.Setup(isolation_level, OperationKind::kRead, row_mark);
                      CHECK_OK(EnumerateIntents(put_batch.read_pairs(), std::ref(helper), partial_range_key_intents));
                    }
                    helper.Finish
              # 更新last_batch_data，其中write_id已经在上一步中更新了
              last_batch_data.hybrid_time = hybrid_time;
              # 设置RunningTransaction::last_batch_data_
              transaction_participant()->BatchReplicated(transaction_id, last_batch_data)
                - TransactionParticipant::Impl::BatchReplicated
                    - 在TransactionParticipant::Impl::transactions_中查找对应的transaction，然后调用RunningTransaction::BatchReplicated
                    - auto it = transactions_.find(id);
                      # 查找到之后，执行RunningTransaction::BatchReplicated
                      (**it).BatchReplicated(data);
                        - last_batch_data_ = value
        # 将rocksdb::WriteBatch中的数据写入Intent DB
        - WriteToRocksDB(frontiers, &write_batch, StorageDbType::kIntents)
    - 如果不是事务性操作
        - PrepareNonTransactionWriteBatch(put_batch, hybrid_time, &write_batch);
            - 这里比较简单，直接遍历put_batch中的每个KV pair，并对每个KV pair写入一条记录到rocksdb::WriteBatch中，对于每个KV pair处理如下：
                - hybrid_time = kv_pair.has_external_hybrid_time() ?
                  HybridTime(kv_pair.external_hybrid_time()) : hybrid_time;
                  # 设置写入rocksdb的记录的key，包含2部分：用户的key，hybrid_time + write_id
                  std::array<Slice, 2> key_parts = {{
                    Slice(kv_pair.key()),
                    doc_ht_buffer.EncodeWithValueType(hybrid_time, write_id),
                  }};
                  Slice key_value = kv_pair.value();
                  # 将key_parts作为key，{ &key_value, 1 }作为value，写入rocksdb::WriteBatch
                  rocksdb_write_batch->Put(key_parts, { &key_value, 1 })
        # 将rocksdb::WriteBatch中的数据写入regular DB中
        - WriteToRocksDB(frontiers, &write_batch, StorageDbType::kRegular)    
```

##### PrepareTransactionWriteBatchHelper
PrepareTransactionWriteBatchHelper用于将事务性的操作的临时记录添加到rocksdb::WriteBatch中，它主要执行以下步骤：
- 处理一个WriteOperation对应的WriteOperationState中的WriteRequestPB中的KeyValueWriteBatchPB中的所有的write pairs，对于write pairs中的每个KV pair执行如下：
    - 对于该KV pair中的key，执行：
        - 添加一条关于该key的intent记录：{{DocKey, IntentTypeSet, hybrid_time + write_id} -> {transaction_id, intra_txn_write_id_, value}}
            - 所有上述每个字段中其实都还包含有一个字节的ValueTypeAsChar，比如IntentTypeSet前面包括一个ValueTypeAsChar::kIntentTypeSet，transaction_id前面还包括一个ValueTypeAsChar::kTransactionId
        - 添加一条关于该key的reverse index记录：{{reverse_key_prefix, transaction id, hybrid time} -> {reverse_value_prefix，key}}
            - reverse_value_prefix可能为空
    - 对于该KV pair中的key的所有前缀部分，执行：
        - 记录key的各个前缀部分(姑且称之为key_prefix)分别要加哪种weak类型IntentTypeSet，如果已经存在相同的key_prefix，则更新它的IntentTypeSet为2者的并集
- 处理一个WriteOperation对应的WriteOperationState中的WriteRequestPB中的KeyValueWriteBatchPB中的所有的read pairs，对于read pairs中的每个KV pair执行如下：
    - 和对于write pairs的处理类似
- 对于所有的write pairs和read pairs中的KV pair的key的前缀部分，执行如下：
    - 在前面处理write pairs和read pairs的过程中，对于每个KV pair的key的前缀部分并未处理，只记录了key的前缀部分要加哪种weak类型的IntentTypeSet，这里统一进行处理，因为只有在这里，这些前缀部分需要加哪种weak类型的IntentTypeSet才会被最终确定
    - 对每个前缀部分的key，添加一条记录：{{weak intent key，weak intent IntentTypeSet，hybrid time}， {transaction_id}}

```
  # 构造方法
  PrepareTransactionWriteBatchHelper(HybridTime hybrid_time,
                                     rocksdb::WriteBatch* rocksdb_write_batch,
                                     const TransactionId& transaction_id,
                                     const Slice& replicated_batches_state,
                                     IntraTxnWriteId* intra_txn_write_id)
      : hybrid_time_(hybrid_time),
        rocksdb_write_batch_(rocksdb_write_batch),
        transaction_id_(transaction_id),
        replicated_batches_state_(replicated_batches_state),
        intra_txn_write_id_(intra_txn_write_id) {
  }
  
  void Setup(
      IsolationLevel isolation_level,
      OperationKind kind,
      RowMarkType row_mark) {
    row_mark_ = row_mark;
    # 根据isolation level，row_mark标识和操作类型，确定应该申请哪种类型的Strong IntentTypeSet
    strong_intent_types_ = GetStrongIntentTypeSet(isolation_level, kind, row_mark);
  }
  
  # 参数intent_strength表示要加的锁是strong类型的还是weak类型的，value_slice表示
  # KV中的value部分，key表示KV中的key部分，last_key为true则表示这是所有KV pairs
  # 中的最后一对
  CHECKED_STATUS operator()(IntentStrength intent_strength, Slice value_slice, KeyBytes* key,
                            LastKey last_key) {
    if (intent_strength == IntentStrength::kWeak) {
      # 如果在给定key上加的是weak的锁，则先记录下来在给定的key上所加的锁，然后就直接返回了，
      # 这些key肯定是用户操作的完整key的前缀部分
      weak_intents_[key->data()] |= StrongToWeak(strong_intent_types_);
      return Status::OK();
    }

    const auto transaction_value_type = ValueTypeAsChar::kTransactionId;
    const auto write_id_value_type = ValueTypeAsChar::kWriteId;
    const auto row_lock_value_type = ValueTypeAsChar::kRowLock;
    # 这里用到了intra_txn_write_id_，下面还会用到write_id_，这2者有何区别呢？是这样的：
    # intra_txn_write_id_是在Tablet::PrepareTransactionWriteBatch -> yb::docdb::PrepareTransactionWriteBatch ->
    # PrepareTransactionWriteBatchHelper::PrepareTransactionWriteBatchHelper中通过TransactionalBatchData::write_id
    # 初始化的，用于表示这是当前transaction中第几个strong intent record，它在PrepareTransactionWriteBatchHelper::operator()
    # 中每当遇到strong intent record的时候就会加1，因为这是一个指针，所以实际上修改的是RunningTransaction::last_batch_data_
    # 中的write_id，如果一个RunningTransaction会涉及多个PrepareTransactionWriteBatchHelper，在多个
    # PrepareTransactionWriteBatchHelper之间，它的作用会得到继承
    # 
    # 而write_id_在PrepareTransactionWriteBatchHelper的类定义中被初始化为0，在PrepareTransactionWriteBatchHelper::operator()
    # 中每当遇到strong intent record的时候就会加1，因而它的作用范围只是在PrepareTransactionWriteBatchHelper内部
    IntraTxnWriteId big_endian_write_id = BigEndian::FromHost32(*intra_txn_write_id_);
    
    # value部分，组成为：{ValueTypeAsChar::kTransactionId, transaction id, ValueTypeAsChar::kWriteId, write id, 用户要写的数据}
    std::array<Slice, 5> value = {{
        Slice(&transaction_value_type, 1),
        transaction_id_.AsSlice(),
        Slice(&write_id_value_type, 1),
        Slice(pointer_cast<char*>(&big_endian_write_id), sizeof(big_endian_write_id)),
        value_slice,
    }};
    
    # 对于行锁类型的操作，在rocksdb value中不记录真实的value_slice
    // Store a row lock indicator rather than data (in value_slice) for row lock intents.
    if (IsValidRowMarkType(row_mark_)) {
      value.back() = Slice(&row_lock_value_type, 1);
    }

    ++*intra_txn_write_id_;

    # intent type部分组成：{ValueTypeAsChar::kIntentTypeSet, strong_intent_types_}
    char intent_type[2] = { ValueTypeAsChar::kIntentTypeSet,
                            static_cast<char>(strong_intent_types_.ToUIntPtr()) };

    DocHybridTimeBuffer doc_ht_buffer;

    # 写入rocksdb的key组成为：{用户写入的key，intent_type，hybrid_time + write_id}
    constexpr size_t kNumKeyParts = 3;
    std::array<Slice, kNumKeyParts> key_parts = {{
        key->AsSlice(),
        Slice(intent_type, 2),
        # 将hybrid_time + write_id编码到doc_ht_buffer中
        doc_ht_buffer.EncodeWithValueType(hybrid_time_, write_id_++),
    }};

    Slice reverse_value_prefix;
    if (last_key && FLAGS_enable_transaction_sealing) {
      reverse_value_prefix = replicated_batches_state_;
    }
    
    # 将key -> value和reverse_key -> reverse_value分别添加到rocksdb_write_batch_
    AddIntent<kNumKeyParts>(
        transaction_id_, key_parts, value, rocksdb_write_batch_, reverse_value_prefix);

    return Status::OK();
  }
  
# AddIntent不是PrepareTransactionWriteBatchHelper中的方法
template <int N>
void AddIntent(
    const TransactionId& transaction_id,
    const FixedSliceParts<N>& key,
    const SliceParts& value,
    rocksdb::WriteBatch* rocksdb_write_batch,
    Slice reverse_value_prefix = Slice()) {
  char reverse_key_prefix[1] = { ValueTypeAsChar::kTransactionId };
  size_t doc_ht_buffer[kMaxWordsPerEncodedHybridTimeWithValueType];
  
  # key中的最后一部分是hybrid time
  auto doc_ht_slice = key.parts[N - 1];
  memcpy(doc_ht_buffer, doc_ht_slice.data(), doc_ht_slice.size());
  for (size_t i = 0; i != kMaxWordsPerEncodedHybridTimeWithValueType; ++i) {
    doc_ht_buffer[i] = ~doc_ht_buffer[i];
  }
  doc_ht_slice = Slice(pointer_cast<char*>(doc_ht_buffer), doc_ht_slice.size());

  # reverse key的组成：{reverse_key_prefix, transaction id, hybrid time}
  std::array<Slice, 3> reverse_key = {{
      Slice(reverse_key_prefix, sizeof(reverse_key_prefix)),
      transaction_id.AsSlice(),
      doc_ht_slice,
  }};
  
  # 写入intent记录: key -> value
  rocksdb_write_batch->Put(key, value);
  
  # 写入reverse index：reverse_key -> reverse_value
  if (reverse_value_prefix.empty()) {
    # reverse_value_prefix为空的情况下，reverse value的组成：{key}
    rocksdb_write_batch->Put(reverse_key, key);
  } else {
    # reverse_value_prefix不为空的情况下，reverse value的组成：{reverse_value_prefix，key}
    std::array<Slice, N + 1> reverse_value;
    reverse_value[0] = reverse_value_prefix;
    memcpy(&reverse_value[1], key.parts, sizeof(*key.parts) * N);
    rocksdb_write_batch->Put(reverse_key, reverse_value);
  }
}

  void Finish() {
    char transaction_id_value_type = ValueTypeAsChar::kTransactionId;

    DocHybridTimeBuffer doc_ht_buffer;

    # 对于weak intent，value中记录的是操作对应的transaction Id
    std::array<Slice, 2> value = {{
        Slice(&transaction_id_value_type, 1),
        transaction_id_.AsSlice(),
    }};

    if (PREDICT_TRUE(!FLAGS_docdb_sort_weak_intents_in_tests)) {
      # 遍历所有的weak_intents_，它是在PrepareTransactionWriteBatchHelper::operator()
      # 中被填充的，当遇到某个key上指定的是weak锁的时候，会添加到weak_intents_中，
      # weak_intents_是一个从key到对应的weak IntentTypeSet的映射表
      for (const auto& intent_and_types : weak_intents_) {
        AddWeakIntent(intent_and_types, value, &doc_ht_buffer);
      }
    } else {
      // This is done in tests when deterministic DocDB state is required.
      std::vector<std::pair<std::string, IntentTypeSet>> intents_and_types(
          weak_intents_.begin(), weak_intents_.end());
      sort(intents_and_types.begin(), intents_and_types.end());
      for (const auto& intent_and_types : intents_and_types) {
        AddWeakIntent(intent_and_types, value, &doc_ht_buffer);
      }
    }
  }

  void AddWeakIntent(
      const std::pair<std::string, IntentTypeSet>& intent_and_types,
      const std::array<Slice, 2>& value,
      DocHybridTimeBuffer* doc_ht_buffer) {
    char intent_type[2] = { ValueTypeAsChar::kIntentTypeSet,
                            static_cast<char>(intent_and_types.second.ToUIntPtr()) };
    constexpr size_t kNumKeyParts = 3;
    # weak intent的key的组成：{weak intent key，weak intent IntentTypeSet，hybrid time}
    std::array<Slice, kNumKeyParts> key = {{
        Slice(intent_and_types.first),
        Slice(intent_type, 2),
        doc_ht_buffer->EncodeWithValueType(hybrid_time_, write_id_++),
    }};

    # 将Intent记录添加到rocksdb_write_batch_中
    AddIntent<kNumKeyParts>(transaction_id_, key, value, rocksdb_write_batch_);
  }
```

接下来关注一个transaction中的多个操作，是每个操作单独在raft中进行复制，还是统一在raft中复制。


