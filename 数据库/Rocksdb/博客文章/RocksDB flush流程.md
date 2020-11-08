# 提纲
[toc]

# 概述
RocksDB中数据是直接写入到MemTable中，然后从MemTable flush到SSTable中，本文将分析RocksDB flush的流程。以下代码是基于RocksDB 6.13进行分析。

# MemTable Flush被触发时的通用执行逻辑
通常在RocksDB中，当触发了MemTable Flush的时候，会执行如下逻辑：
- 获取所有待flush的Immutable MemTable对应的ColumnFamily集合(该步骤根据上下文的不同，实现也不尽相同)；
- 获取每个ColumnFamily中最大的Immutable MemTable ID，并将ColumnFamily到它所对应的最大的Immutable MemTable ID的映射添加到FlushRequest中；
- 将FlushRequest添加到待flush请求队列中；
- 调度flush线程，以执行flush操作；

关于上述逻辑的典型实现如下：
```
Status DBImpl::XXXXXX(...) {
  ...
  autovector<ColumnFamilyData*> cfds;
  
  # 获取所有待flush的ColumnFamily，添加到cfds集合中
  ...
  
  if (status.ok()) {
    if (immutable_db_options_.atomic_flush) {
      AssignAtomicFlushSeq(cfds);
    }
    
    for (auto cfd : cfds) {
      cfd->imm()->FlushRequested();
    }
    
    FlushRequest flush_req;
    # 获取每个ColumnFamily中最大的Immutable MemTable ID，并将ColumnFamily到它所对应
    # 的最大的Immutable MemTable ID的映射添加到FlushRequest中
    GenerateFlushRequest(cfds, &flush_req);
    
    # 将FlushRequest添加到待flush的请求队列中
    SchedulePendingFlush(flush_req, FlushReason::kWriteBufferManager);
    
    # 调度flush线程，以执行flush操作
    MaybeScheduleFlushOrCompaction();
  }
  
  ...
}  
```

# MemTable flush的触发时机
从“MemTable Flush被触发时的通用执行逻辑”中可知，触发MemTable flush的时候，一定会生成FlushRequest并添加到flush请求队列中，也就是一定会调用DBImpl::SchedulePendingFlush，沿着DBImpl::SchedulePendingFlush的调用逻辑发现，主要在以下几个地方：
- DBImpl::FlushMemTable
    - 强制flush给定的ColumnFamily中的MemTables，比如当用户显式调用DBImpl::Flush时会调用之
- DBImpl::HandleWriteBufferFull
    - 如果WriteBuffer(关于WriteBuffer请参考[这里](https://github.com/facebook/rocksdb/wiki/Write-Buffer-Manager)，简单来说，WriteBuffer由Write buffer manager管理，用于控制所有Column Family，甚至所有的DB实例所使用的总的内存大小)使用量超过阈值的情况下，触发MemTable flush，以释放一些WriteBuffer
    - 在DBImpl::WriteImpl -> DBImpl::PreprocessWrite中被调用，DBImpl::PreprocessWrite主要用于在写WriteBatch之前进行一些预处理的工作：如果WAL log总大小超过阈值，或者WriterBuffer满，则触发MemTable flush，如果需要执行write stall，则延迟写入等
- DBImpl::SwitchWAL
    - 如果总的WAL log的大小超过了阈值，则触发MemTable flush，以释放一些WAL log
    - 主要在DBImpl::WriteImpl -> DBImpl::PreprocessWrite中被调用，DBImpl::PreprocessWrite主要用于在写WriteBatch之前进行一些预处理的工作：如果WAL log总大小超过阈值，或者WriterBuffer满，则触发MemTable flush，如果需要执行write stall，则延迟写入等
- DBImpl::ScheduleFlushes
    - 如果DBImpl::flush_scheduler_中存在待flush的ColumnFamily，则调度关于这些ColumnFamily的MemTable flush操作 
        - MemTable的写入操作是由MemTableInserter完成的，MemTableInserter在每次写入MemTable之后，会检查是否需要调度关于该MemTable的flush，如果MemTable::flush_state_为FlushState::kRequested状态，则表明需要调度，则会将对应的ColumnFamily添加到DBImpl::flush_scheduler_中
        - 在MemTable::Add -> MemTable::UpdateFlushState中在满足特定条件的情况下，flush_state_会被设置为FLUSH_REQUESTED，关于这些特殊的条件，请参考[这里](https://code.aliyun.com/ckeyer/rocksdb/commit/a5fafd4f46248e8b89d5ec7c63d3f25aaa347fd1)
    - 主要在DBImpl::WriteImpl -> DBImpl::PreprocessWrite中被调用，DBImpl::PreprocessWrite主要用于在写WriteBatch之前进行一些预处理的工作：如果WAL log总大小超过阈值，或者WriterBuffer满，则触发MemTable flush，如果需要执行write stall，则延迟写入等

下面我们逐一分析上述5个触发时机。

## DBImpl::FlushMemTable
FlushMemTable中主要执行以下步骤：
- 如果flush_options.allow_write_stall为false，则必须确保flush执行之前不存在可能导致write stall的因素
- 等待所有正在写MemTable的操作完成
- 如果Mutable MemTable不为空或者cached_recoverable_state_不为空(这两种情况下，Mutable MemTable中都包含数据)，则将Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
- 生成关于给定ColumnFamily的FlushRequest，其中包含ColumnFamily信息和待Flush的MemTable ID信息
- 将FlushRequest添加到Flush请求队列中
- 尝试调度执行Flush或者Compaction
- 如果flush_options.wait为true，则等待关于给定ColumnFamily的MemTables Flush结束

```
Status DBImpl::FlushMemTable(ColumnFamilyData* cfd,
                             const FlushOptions& flush_options,
                             FlushReason flush_reason, bool writes_stopped) {
  Status s;
  uint64_t flush_memtable_id = 0;
  
  # 如果flush_options.allow_write_stall被设置为true，则flush操作将立即执行，即使
  # flush会导致write stall发生，如果flush_options.allow_write_stall被设置为false，
  # 则flush操作必须等待，直到不会发生write stall或者flush操作已经由其它线程执行了
  # 
  # 默认flush_options.allow_write_stall被设置为false
  if (!flush_options.allow_write_stall) {
    bool flush_needed = true;
    
    # 等待，直到没有任何可能导致write stall的因素存在或者flush操作已经由其它线程执行了
    s = WaitUntilFlushWouldNotStallWrites(cfd, &flush_needed);
    
    # 如果出错，或者flush操作已经由其它线程执行了(则当前线程无需再执行了)，则直接返回
    if (!s.ok() || !flush_needed) {
      return s;
    }
  }
  
  FlushRequest flush_req;
  {
    WriteContext context;
    InstrumentedMutexLock guard_lock(&mutex_);

    WriteThread::Writer w;
    WriteThread::Writer nonmem_w;
    if (!writes_stopped) {
      # 如果没有停止写操作，则此时可能有正在进行的写操作，需要等待这些正在进行的写
      # 操作完成
      write_thread_.EnterUnbatched(&w, &mutex_);
    }
    
    # 等待所有正在写MemTable，或者已经写了WAL但是尚未写MemTable的请求完成MemTable写入，
    # 只在unordered_write = true的情况下发生，如果unordered_write = true，则不保证快照
    # 不变性，暂不关注unordered_write = true的情况
    WaitForPendingWrites();

    # 如果MemTable不为空或者cached_recoverable_state_不为空，则将Mutable MemTable转换为
    # Immutable MemTable(这两种情况下，Mutable MemTable中都包含数据)，并创建新的
    # Mutable MemTable
    if (!cfd->mem()->IsEmpty() || !cached_recoverable_state_empty_.load()) {
      s = SwitchMemtable(cfd, &context);
    }
    
    if (s.ok()) {
      if (cfd->imm()->NumNotFlushed() != 0 || !cfd->mem()->IsEmpty() ||
          !cached_recoverable_state_empty_.load()) {
        # 将最老的Immutable MemTable添加到FlushRequest中
        flush_memtable_id = cfd->imm()->GetLatestMemTableID();
        flush_req.emplace_back(cfd, flush_memtable_id);
      }
      
      if (immutable_db_options_.persist_stats_to_disk) {
        # 略
        ...
      }
    }

    if (s.ok() && !flush_req.empty()) {
      for (auto& elem : flush_req) {
        ColumnFamilyData* loop_cfd = elem.first;
        loop_cfd->imm()->FlushRequested();
      }
      
      # 如果调用者想要等待本次flush结束的话，则表明调用者期望该ColumnFamilyData
      # 不被其它并发执行的线程给free了，所以增加ColumnFamilyData的引用计数
      if (flush_options.wait) {
        for (auto& elem : flush_req) {
          ColumnFamilyData* loop_cfd = elem.first;
          loop_cfd->Ref();
        }
      }
      
      # 将FlushRequest添加到Flush请求队列中
      SchedulePendingFlush(flush_req, flush_reason);
      # 尝试调度执行Flush或者Compaction
      MaybeScheduleFlushOrCompaction();
    }

    if (!writes_stopped) {
      write_thread_.ExitUnbatched(&w);
    }
  }
  
  if (s.ok() && flush_options.wait) {
    autovector<ColumnFamilyData*> cfds;
    autovector<const uint64_t*> flush_memtable_ids;
    for (auto& iter : flush_req) {
      cfds.push_back(iter.first);
      flush_memtable_ids.push_back(&(iter.second));
    }
    
    # 等待多个ColumnFamilyData及其对应的MemTables Flush结束，
    # 见“等待多个ColumnFamilyData及其对应的MemTables Flush结束”
    s = WaitForFlushMemTables(cfds, flush_memtable_ids,
                              (flush_reason == FlushReason::kErrorRecovery));
    InstrumentedMutexLock lock_guard(&mutex_);
    
    # 主动释放所有相关ColumnFamilyData的引用计数
    for (auto* tmp_cfd : cfds) {
      tmp_cfd->UnrefAndTryDelete();
    }
  }
  
  return s;
}
```

### flush之前确保所有可能导致write stall的因素消失
在执行flush之前，RocksDB会检查当前是否存在可能导致write stall的因素存在，如果存在，则会等待这些因素消失，然后再去执行flush，主要逻辑如下：
- 如果待flush的ColumnFamily已经被drop了，无需执行关于该ColumnFaimily的flush了，则直接退出
- 如果DB正在shuting down过程中，则直接退出
- 如果待flush的MemTable已经由其它线程flush了，则直接退出
- 如果当前尚未达到auto flush或者auto compaction的条件，则直接退出
- 检查当前是否需要write stall(包括stall或者stop)，
    - 如果无需write stall，则直接退出
    - 如果需要write stall，则进入条件等待，直到被唤醒
- 当被唤醒之后，重复上述过程
```
Status DBImpl::WaitUntilFlushWouldNotStallWrites(ColumnFamilyData* cfd,
                                                 bool* flush_needed) {
  {
    *flush_needed = true;
    InstrumentedMutexLock l(&mutex_);
    # 获取当前的Mutable MemTable ID
    uint64_t orig_Mutable_memtable_id = cfd->mem()->GetID();
    WriteStallCondition write_stall_condition = WriteStallCondition::kNormal;

    do {
      # write_stall_condition可取的值为：
      # WriteStallCondition::kNormal - 可以执行flush了
      # WriteStallCondition::kDelayed - write操作需要stall(暂停)一段时间
      # WriteStallCondition::kStopped - write操作需要stop(停止)一段时间
      if (write_stall_condition != WriteStallCondition::kNormal) {
        # 条件等待，直到被唤醒，bg_cv_被唤醒的时机，见“bg_cv_的唤醒时机”
        bg_cv_.Wait();
      }
      
      if (cfd->IsDropped()) {
        # 如果给定的ColumnFamily被drop了，则退出
        return Status::ColumnFamilyDropped();
      }
      
      if (shutting_down_.load(std::memory_order_acquire)) {
        # 如果正在shuting down过程中，则退出
        return Status::ShutdownInProgress();
      }

      # 获取当前最小的MemTable ID
      uint64_t earliest_memtable_id =
          std::min(cfd->mem()->GetID(), cfd->imm()->GetEarliestMemTableID());
          
      if (earliest_memtable_id > orig_Mutable_memtable_id) {
        # 如果当前最小的MemTable ID比最初获取的Mutable MemTable ID还要大，则表明
        # 最初的Mutable MemTable也已经被flush了，无需再等待了
        *flush_needed = false;
        return Status::OK();
      }

      const auto& mutable_cf_options = *cfd->GetLatestMutableCFOptions();
      const auto* vstorage = cfd->current()->storage_info();

      // Skip stalling check if we're below auto-flush and auto-compaction
      // triggers. If it stalled in these conditions, that'd mean the stall
      // triggers are so low that stalling is needed for any background work. In
      // that case we shouldn't wait since background work won't be scheduled.
      if (cfd->imm()->NumNotFlushed() <
              cfd->ioptions()->min_write_buffer_number_to_merge &&
          vstorage->l0_delay_trigger_count() <
              mutable_cf_options.level0_file_num_compaction_trigger) {
        # 如果尚未flush的Immutable MemTable数目小于min_write_buffer_number_to_merge，
        # 或者Level-0中文件数目小于level0_file_num_compaction_trigger，则退出
        break;
      }

      # 检查是否需要write stall，以及需要执行哪种类型的write stall，包括kNormal，
      # kDelayed和kStopped
      #
      # 主要检查以下3方面：
      # 1. 是否有太多的尚未flush的Immutable MemTables
      # 2. 是否有太多的Level-0 SST文件
      # 3. 是否有太多的pending compaction bytes
      #
      # 详细分析，见“产生write stall的原因”
      write_stall_condition =
          ColumnFamilyData::GetWriteStallConditionAndCause(
              cfd->imm()->NumNotFlushed() + 1,
              vstorage->l0_delay_trigger_count() + 1,
              vstorage->estimated_compaction_needed_bytes(), mutable_cf_options)
              .first;
    } while (write_stall_condition != WriteStallCondition::kNormal);
  }
  return Status::OK();
}
```

### 产生write stall的原因
write stall包括3种：kNormal，kDelayed和kStopped。kNormal表示write操作无需stall，kDelayed表示write操作需要stall一段时间，kStopped表示write操作需要stop直到特定事件发生。

当前flush操作之前，是否需要write stall，如果需要write stall的话，该执行哪种类型的write stall，是由ColumnFamilyData::GetWriteStallConditionAndCause控制的。它主要检查3个方面：
- 是否有太多的尚未flush的Immutable MemTables
    - 如果尚未flush的MemTable(包括Immutable MemTable和Mutable MemTable)数目不小于max_write_buffer_number，则write操作需要stop直到flush完成
    - 否则
        - 如果max_write_buffer_number大于3且尚未flush的MemTable(包括Immutable MemTable和Mutable MemTable)数目不小于max_write_buffer_number - 1，则示write操作需要stall一段时间
- 是否Level-0 有太多的SST文件
    - 如果Level-0中SST文件的数目达到level0_stop_writes_trigger，则write操作需要stop直到Level-0和Level-1的compaction减少Level-0中SST文件数目
    - 否则
        - 如果Level-0中SST文件的数目达到level0_slowdown_writes_trigger，则write操作需要stall一段时间
- 是否有太多的待compaction的数据
    - 如果预估的待compaction的数据总量达到hard_pending_compaction_bytes，则write操作需要stop以等待compaction
    - 否则
        - 如果预估的待compaction的数据总量达到soft_pending_compaction_bytes，则write操作需要stall一段时间

```
std::pair<WriteStallCondition, ColumnFamilyData::WriteStallCause>
ColumnFamilyData::GetWriteStallConditionAndCause(
    int num_unflushed_memtables, int num_l0_files,
    uint64_t num_compaction_needed_bytes,
    const MutableCFOptions& mutable_cf_options) {
  if (num_unflushed_memtables >= mutable_cf_options.max_write_buffer_number) {
    return {WriteStallCondition::kStopped, WriteStallCause::kMemtableLimit};
  } else if (!mutable_cf_options.disable_auto_compactions &&
             num_l0_files >= mutable_cf_options.level0_stop_writes_trigger) {
    return {WriteStallCondition::kStopped, WriteStallCause::kL0FileCountLimit};
  } else if (!mutable_cf_options.disable_auto_compactions &&
             mutable_cf_options.hard_pending_compaction_bytes_limit > 0 &&
             num_compaction_needed_bytes >=
                 mutable_cf_options.hard_pending_compaction_bytes_limit) {
    return {WriteStallCondition::kStopped,
            WriteStallCause::kPendingCompactionBytes};
  } else if (mutable_cf_options.max_write_buffer_number > 3 &&
             num_unflushed_memtables >=
                 mutable_cf_options.max_write_buffer_number - 1) {
    return {WriteStallCondition::kDelayed, WriteStallCause::kMemtableLimit};
  } else if (!mutable_cf_options.disable_auto_compactions &&
             mutable_cf_options.level0_slowdown_writes_trigger >= 0 &&
             num_l0_files >=
                 mutable_cf_options.level0_slowdown_writes_trigger) {
    return {WriteStallCondition::kDelayed, WriteStallCause::kL0FileCountLimit};
  } else if (!mutable_cf_options.disable_auto_compactions &&
             mutable_cf_options.soft_pending_compaction_bytes_limit > 0 &&
             num_compaction_needed_bytes >=
                 mutable_cf_options.soft_pending_compaction_bytes_limit) {
    return {WriteStallCondition::kDelayed,
            WriteStallCause::kPendingCompactionBytes};
  }
  
  return {WriteStallCondition::kNormal, WriteStallCause::kNone};
}
```

### Mutable MemTable转换为Immutable MemTable
将Mutable MemTable转换为Immutable MemTable的过程如下：
- 如果存在关于应用程序的cached_recoverable_state_，则将其写入到MemTable中
- 根据当前最新的WAL log segment文件中是否存在数据，确定是否需要创建新的WAL log segment文件
    - 如果存在数据，则创建新的WAL log segment文件
        - 为新的WAL log segment文件分配一个log file number，用于生成新的WAL log segment文件名
        - 如果可以复用之前的已回收的WAL log segment，则直接通过该WAL log segment文件构建新的WAL log segment文件，否则创建一个新的WAL log segment
    - 如果不存在数据，则可以直接使用这个空的WAL log segment文件，无需创建新的WAL log segment文件
- 构建一个新的MemTable
- 如果创建了新的WAL log segment文件，则：
    - 将上一个WAL log segment在buffer中的数据flush到OS cache或者storage(direct io模式下)中
    - 设置最新的WAL log segment number
    - 标记新的WAL log segment中尚无任何数据
    - 将当前最新的WAL log segment添加到logs_集合中
    - 将当前最新的WAL log segment对应的log file number添加到alive_log_files_中
- 将当前的Mutable MemTable转变为Immutable MemTable，并添加到ColumnFamily的Immutable MemTable集合中
- 设置前面步骤中创建的MemTable为新的Mutable MemTable
- 更新SuperVersion，并触发Compaction

```
Status DBImpl::SwitchMemtable(ColumnFamilyData* cfd, WriteContext* context) {
  mutex_.AssertHeld();
  WriteThread::Writer nonmem_w;
  std::unique_ptr<WritableFile> lfile;
  log::Writer* new_log = nullptr;
  MemTable* new_mem = nullptr;
  IOStatus io_s;

  # 如果cached_recoverable_state_中有数据的话，则将其写入MemTable，因为
  # cached_recoverable_state_中的数据只写了WAL，当切换了MemTable之后WAL文件有可
  # 能被删除了，因此在切换MemTable之前，必须将其写入MemTable中
  # 
  # cached_recoverable_state_是一个WriteBatch，它是应用层(比如flink)主动产生的用于
  # 保存最新状态的数据，主要用于恢复
  Status s = WriteRecoverableState();
  if (!s.ok()) {
    return s;
  }

  # 如果开启了two_write_queues_的情况下，访问DBImpl::log_empty_需要log_write_mutex_的保护
  assert(versions_->prev_log_number() == 0);
  if (two_write_queues_) {
    log_write_mutex_.Lock();
  }
  
  # log_empty_表示当前的WAL log segment中是否包含数据，如不不包含数据，则直接使用当前
  # 的WAL log segment，否则新建一个WAL log segment
  bool creating_new_log = !log_empty_;
  if (two_write_queues_) {
    log_write_mutex_.Unlock();
  }
  
  # 在需要新建一个WAL log segment的情况下，检查是否可以复用之前的已回收的WAL log segment，
  # 并将可以复用的WAL log segment的文件编号保存在recycle_log_number中
  uint64_t recycle_log_number = 0;
  if (creating_new_log && immutable_db_options_.recycle_log_file_num &&
      !log_recycle_files_.empty()) {
    recycle_log_number = log_recycle_files_.front();
  }
  
  # 如果需要创建新的WAL log segment的情况下，分配一个新的log segment number，
  # 否则无需创建新的WAL log segment，则直接使用上一个log segment number，即
  # logfile_number_所代表的WAL log segment
  uint64_t new_log_number =
      creating_new_log ? versions_->NewFileNumber() : logfile_number_;
  const MutableCFOptions mutable_cf_options = *cfd->GetLatestMutableCFOptions();

  // Log this later after lock release. It may be outdated, e.g., if background
  // flush happens before logging, but that should be ok.
  int num_imm_unflushed = cfd->imm()->NumNotFlushed();
  
  # 计算WAL log segment 每次预分配的大小
  const auto preallocate_block_size =
      GetWalPreallocateBlockSize(mutable_cf_options.write_buffer_size);
  mutex_.Unlock();
  
  if (creating_new_log) {
    # 创建WAL log segment文件，如果recycle_log_number有效，则复用recycle_log_number
    # 所代表的已回收的WAL log segment，否则创建新的WAL log segment文件，无论是复用
    # 还是新建，最终得到的新的WAL log segment的log segment number都是new_log_number
    io_s = CreateWAL(new_log_number, recycle_log_number, preallocate_block_size, &new_log);
    if (s.ok()) {
      s = io_s;
    }
  }
  
  if (s.ok()) {
    # 当前最大的SequenceNumber将被记录在新创建的MemTable中，这个SequenceNumber一定比
    # MemTable中插入的任何的SequenceNumber都要小
    SequenceNumber seq = versions_->LastSequence();
    new_mem = cfd->ConstructNewMemtable(mutable_cf_options, seq);
    context->superversion_context.NewSuperVersion();
  }
  
  mutex_.Lock();
  
  # 从log_recycle_files_中移除被复用的那个log segment number
  if (recycle_log_number != 0) {
    assert(log_recycle_files_.front() == recycle_log_number);
    log_recycle_files_.pop_front();
  }
  
  if (s.ok() && creating_new_log) {
    log_write_mutex_.Lock();
    assert(new_log != nullptr);
    # 将最后一个WAL log segment在buffer中的数据flush到OS cache或者storage(direct io模式下)中
    if (!logs_.empty()) {
      // Alway flush the buffer of the last log before switching to a new one
      log::Writer* cur_log_writer = logs_.back().writer;
      io_s = cur_log_writer->WriteBuffer();
      if (s.ok()) {
        s = io_s;
      }
    }
    
    if (s.ok()) {
      # 设置新的WAL log segment number
      logfile_number_ = new_log_number;
      # 新的WAL log segment中尚无任何数据
      log_empty_ = true;
      log_dir_synced_ = false;
      # 将当前最新的WAL log segment添加到logs_集合中
      logs_.emplace_back(logfile_number_, new_log);
      # 将当前最新的WAL log segment对应的log file number添加到alive_log_files_中
      alive_log_files_.push_back(LogFileNumberSize(logfile_number_));
    }
    log_write_mutex_.Unlock();
  }

  if (!s.ok()) {
    # 错误处理
    ...
  }

  for (auto loop_cfd : *versions_->GetColumnFamilySet()) {
    // all this is just optimization to delete logs that
    // are no longer needed -- if CF is empty, that means it
    // doesn't need that particular log to stay alive, so we just
    // advance the log number. no need to persist this in the manifest
    if (loop_cfd->mem()->GetFirstSequenceNumber() == 0 &&
        loop_cfd->imm()->NumNotFlushed() == 0) {
      # 设置loop_cfd所对应的ColumnFamily的数据只存在于logfile_number_所代表的的WAL
      # log segment及其之后的WAL log segment中
      if (creating_new_log) {
        loop_cfd->SetLogNumber(logfile_number_);
      }
      
      # 更新loop_cfd所对应的ColumnFamily对应的MemTable在创建的时刻系统中最大的Sequence
      # Number，这个值在MemTable被创建的时候第一次设置，后续每次在SwitchMemTable的时候，
      # 如果MemTable中没有写入任何数据，则会使用系统当前最大的SequenceNumber更新之
      loop_cfd->mem()->SetCreationSeq(versions_->LastSequence());
    }
  }

  cfd->mem()->SetNextLogNumber(logfile_number_);
  
  # 将当前的Mutable MemTable转变为Immutable MemTable，并添加到ColumnFamily的Immutable
  # MemTable集合中
  cfd->imm()->Add(cfd->mem(), &context->memtables_to_free_);
  new_mem->Ref();
  # 设置new_mem为新的Mutable MemTable
  cfd->SetMemtable(new_mem);
  # 更新SuperVersion，并触发Compaction
  InstallSuperVersionAndScheduleWork(cfd, &context->superversion_context,
                                     mutable_cf_options);
  return s;
}
```

### 等待关于多个ColumnFamily的MemTables Flush结束
DBImpl::WaitForFlushMemTables会等待给定的多个ColumnFamily的MemTables Flush结束。参数cfds中保存待Flush的ColumnFamily集合，flush_memtable_ids中保存每个ColumnFamily中待Flush的MemTable ID，cfds[i]和flush_memtable_ids[i]一一对应。

对于flush_memtable_ids[i]：
- 如果flush_memtable_ids[i]是null，则cfds[i]中所有的MemTables都需要被flush
- 如果flush_memtable_ids[i]不为null，则cfds[i]中所有小于flush_memtable_ids[i]的MemTables都需要被flush

实现上：
- DBImpl::WaitForFlushMemTables会不断的统计被drop的ColumnFamily数目num_dropped和统计已经完成Flush的ColumnFamily数目num_finished
- 如果num_dropped与num_finished之和等于cfds.size()，则可以退出等待
- 否则会进入条件等待，直到被Flush或者Compaction相关的信号唤醒，然后再次执行前面的2个步骤
```
Status DBImpl::WaitForFlushMemTables(
    const autovector<ColumnFamilyData*>& cfds,
    const autovector<const uint64_t*>& flush_memtable_ids,
    bool resuming_from_bg_err) {
  int num = static_cast<int>(cfds.size());
  # Flush或者Compaction都会持有这个锁
  # 等待Flush或者Compaction结束
  InstrumentedMutexLock l(&mutex_);
  
  // If the caller is trying to resume from bg error, then
  // error_handler_.IsDBStopped() is true.
  while (resuming_from_bg_err || !error_handler_.IsDBStopped()) {
    if (shutting_down_.load(std::memory_order_acquire)) {
      return Status::ShutdownInProgress();
    }
    // If an error has occurred during resumption, then no need to wait.
    if (!error_handler_.GetRecoveryError().ok()) {
      break;
    }
    
    // Number of column families that have been dropped.
    int num_dropped = 0;
    // Number of column families that have finished flush.
    int num_finished = 0;
    for (int i = 0; i < num; ++i) {
      if (cfds[i]->IsDropped()) {
        # 统计被drop的ColumnFamily数目
        ++num_dropped;
      } else if (cfds[i]->imm()->NumNotFlushed() == 0 ||
                 (flush_memtable_ids[i] != nullptr &&
                  cfds[i]->imm()->GetEarliestMemTableID() >
                      *flush_memtable_ids[i])) {
        # 如果关于当前ColumnFamily的所有的MemTables都被Flush了，或者所有小于
        # *flush_memtable_ids[i]的MemTables都被Flush了，则认为关于该ColumnFamily
        # 的Flush已经结束
        #
        # 统计Flush完成的ColumnFamily数目
        ++num_finished;
      }
    }
    
    if (1 == num_dropped && 1 == num) {
      return Status::ColumnFamilyDropped();
    }
    
    if (num_dropped + num_finished == num) {
      # 如果关于当前FlushRequest的所有的ColumnFamily，要么ColumnFamily已经被drop了，
      # 要么关于ColumnFamily的Flush请求已经完成了，则停止等待，退出while循环
      break;
    }
    
    # 等待Flush或者Compaction相关的信号唤醒
    bg_cv_.Wait();
  }
  
  Status s;
  if (!resuming_from_bg_err && error_handler_.IsDBStopped()) {
    s = error_handler_.GetBGError();
  }
  return s;
}
```

### 尝试调度执行Flush或者Compaction
DBImpl::MaybeScheduleFlushOrCompaction执行逻辑如下：
- 检查当前是否可以调度Flush或者Compaction，如果不能调度(如DB正在关闭过程中，等)，则直接返回
- 先尝试调度执行Flush job，根据高优先级的Background线程池是否为空分别处理：
    - 如果不为空，则在高优先级的Background线程池(主要用于Flush Job)中调度执行Flush job，每个Flush请求都会对应一个Flush job(但是这里并没有将Flush job和Flush请求关联)，直到所有待Flush的请求都被调度执行或者允许并发执行的Flush job的数目已经达到上限为止
    - 否则，在低优先级的Background线程池(主要用于Compaction Job)中调度执行Flush job，直到所有Flush请求都被调度或者低优先的Background线程池满为止
- 检查当前是否可以调度执行Compaction，如果不能调度，则直接返回
- 然后尝试调度执行Compaction job，直到总的被调度的Compaction job数目达到上限或者没有更多的等待调度的Compaction请求为止

当被成功调度时，Flush线程的执行主体是DBImpl::BGWorkFlush，Compaction线程的执行主体是DBImpl::BGWorkCompaction。

```
void DBImpl::MaybeScheduleFlushOrCompaction() {
  mutex_.AssertHeld();
  
  # 检查当前是否可以调度Flush或者Compaction，如果不能调度，则直接返回
  if (!opened_successfully_) {
    // Compaction may introduce data race to DB open
    return;
  }
  
  if (bg_work_paused_ > 0) {
    // we paused the background work
    return;
  } else if (error_handler_.IsBGWorkStopped() &&
             !error_handler_.IsRecoveryInProgress()) {
    // There has been a hard error and this call is not part of the recovery
    // sequence. Bail out here so we don't get into an endless loop of
    // scheduling BG work which will again call this function
    return;
  } else if (shutting_down_.load(std::memory_order_acquire)) {
    // DB is being deleted; no more background compactions
    return;
  }
  
  # 最大允许的Background Flush或者Compaction的数目，返回的bg_job_limits类型为
  # BGJobLimits，其中包含最大允许的Flush job数目和最大允许的Compaction job数目
  auto bg_job_limits = GetBGJobLimits();
  
  # 高优先级的线程池(主要用于Flush)是否为空
  bool is_flush_pool_empty =
      env_->GetBackgroundThreads(Env::Priority::HIGH) == 0;
      
  # 在高优先级线程池不为空的情况下，如果有待Flush的请求(unscheduled_flushes_数目大于0)，
  # 且被调度执行的Flush job数目(bg_flush_scheduled_)小于最大允许的Flush job数目，则调度
  # 多个Flush job，直到没有待Flush的请求或者允许并发执行的Flush job的数目已经达到上限为止
  #
  # 需要指出的是，这里只是为每个Flush job在高优先级线程池中启动了一个线程，并没有分配
  # 具体执行哪个Flush请求
  while (!is_flush_pool_empty && unscheduled_flushes_ > 0 &&
         bg_flush_scheduled_ < bg_job_limits.max_flushes) {
    bg_flush_scheduled_++;
    FlushThreadArg* fta = new FlushThreadArg;
    fta->db_ = this;
    fta->thread_pri_ = Env::Priority::HIGH;
    env_->Schedule(&DBImpl::BGWorkFlush, fta, Env::Priority::HIGH, this,
                   &DBImpl::UnscheduleFlushCallback);
    --unscheduled_flushes_;
  }

  # 如果高优先级线程池(主要用于Flush)为空，则在低优先级线程池(主要用于Compaction)
  # 中调度执行Flush job，直到所有Flush请求都被调度或者低优先的Background线程池满为止
  if (is_flush_pool_empty) {
    while (unscheduled_flushes_ > 0 &&
           bg_flush_scheduled_ + bg_compaction_scheduled_ <
               bg_job_limits.max_flushes) {
      bg_flush_scheduled_++;
      FlushThreadArg* fta = new FlushThreadArg;
      fta->db_ = this;
      fta->thread_pri_ = Env::Priority::LOW;
      env_->Schedule(&DBImpl::BGWorkFlush, fta, Env::Priority::LOW, this,
                     &DBImpl::UnscheduleFlushCallback);
      --unscheduled_flushes_;
    }
  }

  # 检查当前是否可以调度执行Compaction
  if (bg_compaction_paused_ > 0) {
    // we paused the background compaction
    return;
  } else if (error_handler_.IsBGWorkStopped()) {
    return;
  }

  # 调度Compaction job，直到总的被调度的Compaction job数目(bg_compaction_scheduled_)
  # 达到上限(bg_job_limits.max_compactions)或者没有更多的等待调度的Compaction请求(
  # unscheduled_compactions_ = 0)为止
  while (bg_compaction_scheduled_ < bg_job_limits.max_compactions &&
         unscheduled_compactions_ > 0) {
    CompactionArg* ca = new CompactionArg;
    ca->db = this;
    ca->prepicked_compaction = nullptr;
    bg_compaction_scheduled_++;
    unscheduled_compactions_--;
    env_->Schedule(&DBImpl::BGWorkCompaction, ca, Env::Priority::LOW, this,
                   &DBImpl::UnscheduleCompactionCallback);
  }
}
```

## DBImpl::HandleWriteBufferFull
DBImpl::HandleWriteBufferFull的主要逻辑如下：
- 从所有的ColumnFamily中选择具有最小Creation SequenceNumber的Mutable MemTale对应的ColumnFamily作为待Flush的ColumnFamily
    - Mutable MemTable Creation Sequence表示在创建Mutable MemTable的时刻，整个DB实例中最新的Sequence Number(VersionSet::LastSequence())
- 将待Flush的ColumnFamily对应的Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
- 生成关于给定ColumnFamily的FlushRequest，其中包含ColumnFamily信息和待Flush的MemTable ID信息
- 将FlushRequest添加到Flush请求队列中
- 尝试调度执行Flush或者Compaction
- 如果flush_options.wait为true，则等待关于给定ColumnFamily的MemTables Flush结束
```
Status DBImpl::HandleWriteBufferFull(WriteContext* write_context) {
  mutex_.AssertHeld();
  assert(write_context != nullptr);
  Status status;

  # 获取需要被Flush的ColumnFamily集合，保存在cfds集合中
  autovector<ColumnFamilyData*> cfds;
  if (immutable_db_options_.atomic_flush) {
    # 暂不关注atomic flush
    SelectColumnFamiliesForAtomicFlush(&cfds);
  } else {
    # 从所有的ColumnFamily中选择具有最小Creation SequenceNumber的Mutable MemTale对应
    # 的ColumnFamily作为待Flush的ColumnFamily
    ColumnFamilyData* cfd_picked = nullptr;
    SequenceNumber seq_num_for_cf_picked = kMaxSequenceNumber;

    for (auto cfd : *versions_->GetColumnFamilySet()) {
      if (cfd->IsDropped()) {
        continue;
      }
      if (!cfd->mem()->IsEmpty()) {
        // We only consider active mem table, hoping immutable memtable is
        // already in the process of flushing.
        uint64_t seq = cfd->mem()->GetCreationSeq();
        if (cfd_picked == nullptr || seq < seq_num_for_cf_picked) {
          cfd_picked = cfd;
          seq_num_for_cf_picked = seq;
        }
      }
    }
    if (cfd_picked != nullptr) {
      cfds.push_back(cfd_picked);
    }
    MaybeFlushStatsCF(&cfds);
  }

  # 逐一处理cfds集合中的每一个ColumnFamily(在没有设置atomic flush的情况下，实际上
  # cfds集合中只有一个ColumnFamily)
  for (const auto cfd : cfds) {
    if (cfd->mem()->IsEmpty()) {
      continue;
    }
    
    # 增加当前ColumnFamily的引用计数
    cfd->Ref();
    # 将Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
    status = SwitchMemtable(cfd, write_context);
    cfd->UnrefAndTryDelete();
    
    if (!status.ok()) {
      break;
    }
  }

  # 下面的逻辑，跟“MemTable Flush被触发时的通用执行逻辑”中一样，不再赘述
  if (status.ok()) {
    if (immutable_db_options_.atomic_flush) {
      AssignAtomicFlushSeq(cfds);
    }
    for (const auto cfd : cfds) {
      cfd->imm()->FlushRequested();
    }
    FlushRequest flush_req;
    GenerateFlushRequest(cfds, &flush_req);
    SchedulePendingFlush(flush_req, FlushReason::kWriteBufferFull);
    MaybeScheduleFlushOrCompaction();
  }
  
  return status;
}
```

## DBImpl::ScheduleFlushes
DBImpl::ScheduleFlushes的主要逻辑如下：
- 将flush_scheduler_(类型为FlushScheduler)中的所有的ColumnFamily都作为待Flush的ColumnFamily
    - 在MemTable::Add -> MemTable::UpdateFlushState中，如果MemTable::flush_state_为FLUSH_NOT_REQUESTED状态且当前MemTable已经写满了，则会设置MemTable::flush_state_为FLUSH_REQUESTED状态
    - 在MemTableInserter::DeleteImpl/MemTableInserter::MergeCF/MemTableInserter::PutCFImpl -> MemTableInserter::CheckMemtableFull中如果发现MemTable::flush_state_为FLUSH_REQUESTED状态，则会将MemTable::flush_state_设置为FLUSH_SCHEDULED状态，并且调用FlushScheduler::ScheduleWork将当前的ColumnFamily添加到FlushScheduler中
    - 在DBImpl::Write -> DBImpl::WriteImpl -> DBImpl::PreprocessWrite中，会将FlushScheduler管理的所有ColumnFamily添加到待Flush的ColumnFamily集合cfds中
- 逐一遍历cfds集合中所有的待Flush的ColumnFamily，将所有待Flush的ColumnFamilys对应的Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
- 生成关于给定ColumnFamily的FlushRequest，其中包含ColumnFamily信息和待Flush的MemTable ID信息
- 将FlushRequest添加到Flush请求队列中
- 尝试调度执行Flush或者Compaction
- 如果flush_options.wait为true，则等待关于给定ColumnFamily的MemTables Flush结束

```
Status DBImpl::ScheduleFlushes(WriteContext* context) {
  autovector<ColumnFamilyData*> cfds;
  if (immutable_db_options_.atomic_flush) {
    # 暂不关注atomic flush
    SelectColumnFamiliesForAtomicFlush(&cfds);
    for (auto cfd : cfds) {
      cfd->Ref();
    }
    flush_scheduler_.Clear();
  } else {
    # 从FlushScheduler中获取所有ColumnFamiliy，这些ColumnFamily将被添加到待
    # Flush的ColumnFamily集合cfds中
    ColumnFamilyData* tmp_cfd;
    while ((tmp_cfd = flush_scheduler_.TakeNextColumnFamily()) != nullptr) {
      cfds.push_back(tmp_cfd);
    }
    MaybeFlushStatsCF(&cfds);
  }
  
  Status status;
  
  for (auto& cfd : cfds) {
    if (!cfd->mem()->IsEmpty()) {
      # 将Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
      status = SwitchMemtable(cfd, context);
    }
    if (cfd->UnrefAndTryDelete()) {
      cfd = nullptr;
    }
    if (!status.ok()) {
      break;
    }
  }

  # 下面的逻辑，跟“MemTable Flush被触发时的通用执行逻辑”中一样，不再赘述
  if (status.ok()) {
    if (immutable_db_options_.atomic_flush) {
      AssignAtomicFlushSeq(cfds);
    }
    FlushRequest flush_req;
    GenerateFlushRequest(cfds, &flush_req);
    SchedulePendingFlush(flush_req, FlushReason::kWriteBufferFull);
    MaybeScheduleFlushOrCompaction();
  }
  return status;
}
```

## DBImpl::SwitchWAL
DBImpl::SwitchWAL的主要逻辑如下：
- 逐一处理VersionSet中记录的所有的ColumnFamily，如果ColumnFamily没有被drop，且该ColumnFamily对应的最小应当被保留的WAL log file number不大于最旧的尚未flush的WAL log file number，则该ColumnFamily将被添加到待Flush的ColumnFamily集合cfds中
- 逐一遍历cfds集合中所有的待Flush的ColumnFamily，将所有待Flush的ColumnFamilys对应的Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
- 生成关于给定ColumnFamily的FlushRequest，其中包含ColumnFamily信息和待Flush的MemTable ID信息
- 将FlushRequest添加到Flush请求队列中
- 尝试调度执行Flush或者Compaction
- 如果flush_options.wait为true，则等待关于给定ColumnFamily的MemTables Flush结束
```
Status DBImpl::SwitchWAL(WriteContext* write_context) {
  mutex_.AssertHeld();
  assert(write_context != nullptr);
  Status status;

  # 正在flush过程中
  if (alive_log_files_.begin()->getting_flushed) {
    return status;
  }

  # 获取最旧的尚未flush的WAL log file number
  auto oldest_alive_log = alive_log_files_.begin()->number;
  bool flush_wont_release_oldest_log = false;
  if (allow_2pc()) {
    ...
  }
  
  if (!flush_wont_release_oldest_log) {
    // we only mark this log as getting flushed if we have successfully
    // flushed all data in this log. If this log contains outstanding prepared
    // transactions then we cannot flush this log until those transactions are
    // commited.
    unable_to_release_oldest_log_ = false;
    alive_log_files_.begin()->getting_flushed = true;
  }

  autovector<ColumnFamilyData*> cfds;
  if (immutable_db_options_.atomic_flush) {
    # 暂不关注atomic flush
    SelectColumnFamiliesForAtomicFlush(&cfds);
  } else {
    # 获取VersionSet中记录的所有的ColumnFamily，如果该ColumnFamily没有被drop，且
    # 该ColumnFamily对应的最小应当保留的WAL log file number比最旧的尚未flush的
    # WAL log file number大，则该ColumnFamily将被添加到待Flush的ColumnFamily集合
    # cfds中
    for (auto cfd : *versions_->GetColumnFamilySet()) {
      if (cfd->IsDropped()) {
        continue;
      }
      if (cfd->OldestLogToKeep() <= oldest_alive_log) {
        cfds.push_back(cfd);
      }
    }
    MaybeFlushStatsCF(&cfds);
  }

  for (const auto cfd : cfds) {
    cfd->Ref();
    # 将Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
    status = SwitchMemtable(cfd, write_context);
    cfd->UnrefAndTryDelete();
    if (!status.ok()) {
      break;
    }
  }

  # 下面的逻辑，跟“MemTable Flush被触发时的通用执行逻辑”中一样，不再赘述
  if (status.ok()) {
    if (immutable_db_options_.atomic_flush) {
      AssignAtomicFlushSeq(cfds);
    }
    for (auto cfd : cfds) {
      cfd->imm()->FlushRequested();
    }
    FlushRequest flush_req;
    GenerateFlushRequest(cfds, &flush_req);
    SchedulePendingFlush(flush_req, FlushReason::kWriteBufferManager);
    MaybeScheduleFlushOrCompaction();
  }
  return status;
}
```

# Flush线程的执行主体DBImpl::BGWorkFlush
DBImpl::BGWorkFlush会直接调用DBImpl::BackgroundCallFlush。
```
void DBImpl::BGWorkFlush(void* arg) {
  FlushThreadArg fta = *(reinterpret_cast<FlushThreadArg*>(arg));
  delete reinterpret_cast<FlushThreadArg*>(arg);
  static_cast_with_check<DBImpl>(fta.db_)->BackgroundCallFlush(fta.thread_pri_);
}
```

DBImpl::BackgroundCallFlush会进一步调用DBImpl::BackgroundFlush从Flush请求队列中选择Flush请求并执行之，当当前的Flush job执行完毕之后，会尝试调度执行更多的Flush和Compaction，并且在最后唤醒等待Flush结束事件的线程。
```
void DBImpl::BackgroundCallFlush(Env::Priority thread_pri) {
  bool made_progress = false;
  JobContext job_context(next_job_id_.fetch_add(1), true);

  # info级别的日志相关的log buffer
  LogBuffer log_buffer(InfoLogLevel::INFO_LEVEL,
                       immutable_db_options_.info_log.get());
  {
    InstrumentedMutexLock l(&mutex_);
    assert(bg_flush_scheduled_);
    num_running_flushes_++;

    std::unique_ptr<std::list<uint64_t>::iterator>
        pending_outputs_inserted_elem(new std::list<uint64_t>::iterator(
            CaptureCurrentFileNumberInPendingOutputs()));
    FlushReason reason;

    # 从Flush请求队列中选择Flush请求并执行之
    Status s = BackgroundFlush(&made_progress, &job_context, &log_buffer,
                               &reason, thread_pri);
                               
    # 出错情况下的处理
    if (!s.ok() && !s.IsShutdownInProgress() && !s.IsColumnFamilyDropped() &&
        reason != FlushReason::kErrorRecovery) {
      ...
    }

    ReleaseFileNumberFromPendingOutputs(pending_outputs_inserted_elem);

    ...

    assert(num_running_flushes_ > 0);
    num_running_flushes_--;
    bg_flush_scheduled_--;
    
    # 尝试调度执行更多的Flush和Compaction
    MaybeScheduleFlushOrCompaction();
    atomic_flush_install_cv_.SignalAll();
    
    # 当前的Flush执行完毕，唤醒其它在bg_cv_上条件等待的线程
    bg_cv_.SignalAll();
  }
}
```

DBImpl::BackgroundFlush会从Flush请求队列中获取一个有效的Flush请求，即满足该Flush请求中至少存在一个ColumnFamily需要Flush，则根据该Flush请求生成BGFlushArg数组，传递给DBImpl::FlushMemTablesToOutputFiles进行处理。
```
Status DBImpl::BackgroundFlush(bool* made_progress, JobContext* job_context,
                               LogBuffer* log_buffer, FlushReason* reason,
                               Env::Priority thread_pri) {
  mutex_.AssertHeld();

  Status status;
  *reason = FlushReason::kOthers;
  
  ...

  autovector<BGFlushArg> bg_flush_args;
  std::vector<SuperVersionContext>& superversion_contexts =
      job_context->superversion_contexts;
  autovector<ColumnFamilyData*> column_families_not_to_flush;
  
  # 从Flush请求队列中获取一个有效的Flush请求(请求中至少有一个ColumnFamily满足：没有被drop
  # 或者该ColumnFamily中至少有一个MemTable没有被Flush)，并将该Flush请求转换为BGFlushArg数
  # 组
  while (!flush_queue_.empty()) {
    // This cfd is already referenced
    const FlushRequest& flush_req = PopFirstFromFlushQueue();
    superversion_contexts.clear();
    superversion_contexts.reserve(flush_req.size());

    for (const auto& iter : flush_req) {
      ColumnFamilyData* cfd = iter.first;
      # 无需在当前的ColumnFamily上执行Flush，则处理Flush请求中的下一个ColumnFamily
      if (cfd->IsDropped() || !cfd->imm()->IsFlushPending()) {
        // can't flush this CF, try next one
        column_families_not_to_flush.push_back(cfd);
        continue;
      }
      
      superversion_contexts.emplace_back(SuperVersionContext(true));
      
      # 根据Flush请求中当前ColumnFamily的相关信息(主要是ColumnFamilyData和MemTable Id)
      # 生成BGFlushArg，并添加到bg_flush_args中
      bg_flush_args.emplace_back(cfd, iter.second,
                                 &(superversion_contexts.back()));
    }
    
    # 如果当前的Flush请求中至少有一个ColumnFamily需要执行Flush，则停止查找
    if (!bg_flush_args.empty()) {
      break;
    }
  }

  if (!bg_flush_args.empty()) {
    auto bg_job_limits = GetBGJobLimits();
    status = FlushMemTablesToOutputFiles(bg_flush_args, made_progress,
                                         job_context, log_buffer, thread_pri);
    *reason = bg_flush_args[0].cfd_->GetFlushReason();
    # Flush结束，减少相应ColumnFamily的引用计数
    for (auto& arg : bg_flush_args) {
      ColumnFamilyData* cfd = arg.cfd_;
      if (cfd->UnrefAndTryDelete()) {
        arg.cfd_ = nullptr;
      }
    }
  }
  
  # 减少那些无需执行Flush的ColumnFamily上的引用计数
  for (auto cfd : column_families_not_to_flush) {
    cfd->UnrefAndTryDelete();
  }
  return status;
}
```

DBImpl::FlushMemTablesToOutputFiles中针对传递过来的BGFlushArg数组中的每个BGFlushArg分别处理，每个BGFlushArg都代表一个ColumnFamily的Flush相关的信息，并针对每一个BGFlushArg进一步调用DBImpl::FlushMemTableToOutputFile进行处理。
```
Status DBImpl::FlushMemTablesToOutputFiles(
    const autovector<BGFlushArg>& bg_flush_args, bool* made_progress,
    JobContext* job_context, LogBuffer* log_buffer, Env::Priority thread_pri) {
  ...
  
  for (auto& arg : bg_flush_args) {
    ColumnFamilyData* cfd = arg.cfd_;
    MutableCFOptions mutable_cf_options = *cfd->GetLatestMutableCFOptions();
    SuperVersionContext* superversion_context = arg.superversion_context_;
    
    Status s = FlushMemTableToOutputFile(
        cfd, mutable_cf_options, made_progress, job_context,
        superversion_context, snapshot_seqs, earliest_write_conflict_snapshot,
        snapshot_checker, log_buffer, thread_pri);
    if (!s.ok()) {
      ...
    }
  }
  return status;
}
```

在DBImpl::FlushMemTableToOutputFile中主要逻辑如下：
- 生成一个FlushJob
- 选择满足条件的所有MemTables作为当前Flush job的Flush对象，如果FlushJob::max_memtable_id_为null，则挑选出所有尚未执行Flush的MemTables，否则，挑选出所有不大于*FlushJob::max_memtable_id_且尚未执行Flush的MemTables，挑选结果保存在FlushJob::mems_中
- 执行flush逻辑
- 更新SuperVersion，并且触发新的Compaction
- 将刚刚Flush的SST文件添加到SstFileManager中
```
Status DBImpl::FlushMemTableToOutputFile(
    ColumnFamilyData* cfd, const MutableCFOptions& mutable_cf_options,
    bool* made_progress, JobContext* job_context,
    SuperVersionContext* superversion_context,
    std::vector<SequenceNumber>& snapshot_seqs,
    SequenceNumber earliest_write_conflict_snapshot,
    SnapshotChecker* snapshot_checker, LogBuffer* log_buffer,
    Env::Priority thread_pri) {
  mutex_.AssertHeld();
  assert(cfd->imm()->NumNotFlushed() != 0);
  assert(cfd->imm()->IsFlushPending());

  # 生成一个FlushJob
  FlushJob flush_job(
      dbname_, cfd, immutable_db_options_, mutable_cf_options,
      nullptr /* memtable_id */, file_options_for_compaction_, versions_.get(),
      &mutex_, &shutting_down_, snapshot_seqs, earliest_write_conflict_snapshot,
      snapshot_checker, job_context, log_buffer, directories_.GetDbDir(),
      GetDataDir(cfd, 0U),
      GetCompressionFlush(*cfd->ioptions(), mutable_cf_options), stats_,
      &event_logger_, mutable_cf_options.report_bg_io_stats,
      true /* sync_output_directory */, true /* write_manifest */, thread_pri,
      io_tracer_, db_id_, db_session_id_);
  FileMetaData file_meta;

  # 选择满足条件的所有MemTables作为当前Flush job的Flush对象，如果
  # FlushJob::max_memtable_id_为null，则挑选出所有尚未执行Flush的MemTables，
  # 否则，挑选出所有不大于*FlushJob::max_memtable_id_且尚未执行Flush的MemTables，
  # 挑选结果保存在FlushJob::mems_中
  flush_job.PickMemTable();

  Status s;
  IOStatus io_s = IOStatus::OK();
  if (logfile_number_ > 0 &&
      versions_->GetColumnFamilySet()->NumberOfColumnFamilies() > 1) {
    # 如果有不止一个ColumnFamily，则必须确保除最近的WAL log file之外的其它所有的
    # WAL log files都被sync，否则如果在MemTable Flush之后，而在WAL log file sync之
    # 前节点crash了，则可能会出现某些ColumnFamily的数据存在，而另外一些ColumnFamily
    # 的数据丢失的情况(因为MemTable/SSTable是各ColumnFamily独有的，而WAL log file
    # 是所有ColumnFamily所共享的)
    #
    # 借助于SyncClosedLogs来确保所有尚未sync的WAL log file都sync
    // If there are more than one column families, we need to make sure that
    // all the log files except the most recent one are synced. Otherwise if
    // the host crashes after flushing and before WAL is persistent, the
    // flushed SST may contain data from write batches whose updates to
    // other column families are missing.
    // SyncClosedLogs() may unlock and re-lock the db_mutex.
    io_s = SyncClosedLogs(job_context);
    s = io_s;
  }

  if (s.ok()) {
    # 执行flush逻辑
    s = flush_job.Run(&logs_with_prep_tracker_, &file_meta);
  } else {
    flush_job.Cancel();
  }
  
  if (io_s.ok()) {
    io_s = flush_job.io_status();
  }

  if (s.ok()) {
    # 更新SuperVersion，并且触发新的Compaction
    InstallSuperVersionAndScheduleWork(cfd, superversion_context,
                                       mutable_cf_options);
    if (made_progress) {
      *made_progress = true;
    }
  }

  if (s.ok()) {
#ifndef ROCKSDB_LITE
    // may temporarily unlock and lock the mutex.
    NotifyOnFlushCompleted(cfd, mutable_cf_options,
                           flush_job.GetCommittedFlushJobsInfo());
    # 将刚刚Flush的SST文件添加到SstFileManager中
    auto sfm = static_cast<SstFileManagerImpl*>(
        immutable_db_options_.sst_file_manager.get());
    if (sfm) {
      // Notify sst_file_manager that a new file was added
      std::string file_path = MakeTableFileName(
          cfd->ioptions()->cf_paths[0].path, file_meta.fd.GetNumber());
      sfm->OnAddFile(file_path);
    }
#endif  // ROCKSDB_LITE
  }
  return s;
}
```

在FlushJob::Run中会将FlushJob::mems_中记录的所有的MemTables Flush到Level-0的SSTable中，如果Flush失败，则重置FlushJob::mems_中记录的所有的MemTables的状态，以便在下次Flush的时候这些MemTable可以被再次选中，如果Flush成功，则更新manifest文件中的记录。
```
Status FlushJob::Run(LogsWithPrepTracker* prep_tracker,
                     FileMetaData* file_meta) {
  db_mutex_->AssertHeld();
  assert(pick_memtable_called);
  AutoThreadOperationStageUpdater stage_run(
      ThreadStatus::STAGE_FLUSH_RUN);

  # 将FlushJob::mems_中记录的所有的MemTables Flush到Level-0的SSTable中
  Status s = WriteLevel0Table();

  if (!s.ok()) {
    # 如果Flush失败，则重置FlushJob::mems_中记录的所有的MemTables的状态，以便
    # 在下次Flush的时候这些MemTable可以被再次选中
    cfd_->imm()->RollbackMemtableFlush(mems_, meta_.fd.GetNumber());
  } else if (write_manifest_) {
    # 如果Flush成功，则在manifest文件中记录
    IOStatus tmp_io_s;
    s = cfd_->imm()->TryInstallMemtableFlushResults(
        cfd_, mutable_cf_options_, mems_, prep_tracker, versions_, db_mutex_,
        meta_.fd.GetNumber(), &job_context_->memtables_to_free, db_directory_,
        log_buffer_, &committed_flush_jobs_info_, &tmp_io_s);
    if (!tmp_io_s.ok()) {
      io_status_ = tmp_io_s;
    }
  }

  if (s.ok() && file_meta != nullptr) {
    *file_meta = meta_;
  }

  return s;
}
```

FlushJob::WriteLevel0Table中将FlushJob中挑选出来的所有Memtable进行Merge然后构造成sstable并写到L0，主要逻辑如下：
- 遍历所有的memtable，并获取每个memtable的InternalIterator(用于遍历待flush的数据)以及FragmentedRangeTombstoneIterator(用于遍历待删除的数据)
- 基于各MemTable的Iterator构建归并迭代器MergingIterator，基于最小堆实现多路归并
- 调用BuildTable将数据写入SSTable文件中
- 如果output_file_directory不为空则同步该目录
- 将Flush生成的SSTable文件添加到L0

```
Status FlushJob::WriteLevel0Table() {
  AutoThreadOperationStageUpdater stage_updater(
      ThreadStatus::STAGE_FLUSH_WRITE_L0);
  db_mutex_->AssertHeld();
  const uint64_t start_micros = db_options_.env->NowMicros();
  const uint64_t start_cpu_micros = db_options_.env->NowCPUNanos() / 1000;
  Status s;
  {
    auto write_hint = cfd_->CalculateSSTWriteHint(0);
    db_mutex_->Unlock();
    if (log_buffer_) {
      log_buffer_->FlushBufferToLog();
    }
    
    // memtables and range_del_iters store internal iterators over each data
    // memtable and its associated range deletion memtable, respectively, at
    // corresponding indexes.
    std::vector<InternalIterator*> memtables;
    std::vector<std::unique_ptr<FragmentedRangeTombstoneIterator>>
        range_del_iters;
    ReadOptions ro;
    ro.total_order_seek = true;
    Arena arena;
    uint64_t total_num_entries = 0, total_num_deletes = 0;
    uint64_t total_data_size = 0;
    size_t total_memory_usage = 0;
    
    # 遍历所有的memtable，并获取每个memtable的iterator以及FragmentedRangeTombstoneIterator
    for (MemTable* m : mems_) {
      memtables.push_back(m->NewIterator(ro, &arena));
      auto* range_del_iter =
          m->NewRangeTombstoneIterator(ro, kMaxSequenceNumber);
      if (range_del_iter != nullptr) {
        range_del_iters.emplace_back(range_del_iter);
      }
      
      total_num_entries += m->num_entries();
      total_num_deletes += m->num_deletes();
      total_data_size += m->get_data_size();
      total_memory_usage += m->ApproximateMemoryUsage();
    }

    {
      # 基于各MemTable的Iterator构建MergingIterator
      ScopedArenaIterator iter(
          NewMergingIterator(&cfd_->internal_comparator(), &memtables[0],
                             static_cast<int>(memtables.size()), &arena));

      ...

      # 根据Merging Iterator构造SSTable file      
      IOStatus io_s;
      s = BuildTable(
          dbname_, db_options_.env, db_options_.fs.get(), *cfd_->ioptions(),
          mutable_cf_options_, file_options_, cfd_->table_cache(), iter.get(),
          std::move(range_del_iters), &meta_, cfd_->internal_comparator(),
          cfd_->int_tbl_prop_collector_factories(), cfd_->GetID(),
          cfd_->GetName(), existing_snapshots_,
          earliest_write_conflict_snapshot_, snapshot_checker_,
          output_compression_, mutable_cf_options_.sample_for_compression,
          mutable_cf_options_.compression_opts,
          mutable_cf_options_.paranoid_file_checks, cfd_->internal_stats(),
          TableFileCreationReason::kFlush, &io_s, io_tracer_, event_logger_,
          job_context_->job_id, Env::IO_HIGH, &table_properties_, 0 /* level */,
          creation_time, oldest_key_time, write_hint, current_time, db_id_,
          db_session_id_);
    }

    if (s.ok() && output_file_directory_ != nullptr && sync_output_directory_) {
      # 如果指定了sync_output_directory，则同步新产生的SSTable文件所在的目录
      s = output_file_directory_->Fsync(IOOptions(), nullptr);
    }
    
    db_mutex_->Lock();
  }
  
  base_->Unref();

  if (s.ok() && meta_.fd.GetFileSize() > 0) {
    # 将新产生的SSTable添加到Level-0
    edit_->AddFile(0 /* level */, meta_.fd.GetNumber(), meta_.fd.GetPathId(),
                   meta_.fd.GetFileSize(), meta_.smallest, meta_.largest,
                   meta_.fd.smallest_seqno, meta_.fd.largest_seqno,
                   meta_.marked_for_compaction, meta_.oldest_blob_file_number,
                   meta_.oldest_ancester_time, meta_.file_creation_time,
                   meta_.file_checksum, meta_.file_checksum_func_name);
  }
#ifndef ROCKSDB_LITE
  // Piggyback FlushJobInfo on the first first flushed memtable.
  mems_[0]->SetFlushJobInfo(GetFlushJobInfo());
#endif  // !ROCKSDB_LITE

  return s;
}
```



# 参考
[Write Stall](https://github.com/facebook/rocksdb/wiki/Write-Stalls)

[Pipelined Write](https://github.com/facebook/rocksdb/wiki/Pipelined-Write)

