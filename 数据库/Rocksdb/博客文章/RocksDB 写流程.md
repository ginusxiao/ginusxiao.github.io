# 提纲
[toc]

# 说明
下面的分析是基于YugaByte所用到的RocksDB 4.5.0。

# 写流程
## 写操作相关接口
DB::Put(...)是一个写操作简单封装，最终会打包一个WriteBatch对象，调用rocksdb::DBImpl::WriteImpl来完成写。
```
Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
               const Slice& key, const Slice& value) {
  // Pre-allocate size of write batch conservatively.
  // 8 bytes are taken by header, 4 bytes for count, 1 byte for type,
  // and we allocate 11 extra bytes for key length, as well as value length.
  WriteBatch batch(key.size() + value.size() + 24);
  batch.Put(column_family, key, value);
  return Write(opt, &batch);
}
```

当然也可以手工构造一个WriteBatch，放入多个key/value操作，然后调用DB::Write(...)。
```
virtual Status Write(const WriteOptions& options, WriteBatch* updates) = 0;

Status DBImpl::Write(const WriteOptions& write_options, WriteBatch* my_batch) {
  return WriteImpl(write_options, my_batch, nullptr);
}
```

## WriteBatch的结构
一个WriteBatch中可能会包含多条记录，可以通过WriteBatch.put(), WriteBatch.Delete(), WriteBatch.SingleDelete(), WriteBatch.Merge()等将操作添加到WriteBatch中。

WriteBatch的格式如下：
```
    WriteBatch::rep_ :=
        sequence: fixed64，batch sequence的起始值
        count: fixed32，操作记录个数
        data: record[count]，操作记录，一共count个记录
```

每个操作记录格式如下：
```
    record :=
        kTypeValue varstring varstring，或者
        kTypeDeletion varstring，或者
        kTypeSingleDeletion varstring，或者
        kTypeMerge varstring varstring，或者
        kTypeColumnFamilyValue varint32 varstring varstring，或者
        kTypeColumnFamilyDeletion varint32 varstring varstring，或者
        kTypeColumnFamilySingleDeletion varint32 varstring varstring，或者
        kTypeColumnFamilyMerge varint32 varstring varstring
        
     varstring :=
        len: varint32
        data: uint8[len]
```


## DBImpl::WriteImpl逻辑

1. 新建一个WriteThread::Writer对象，关联到传入的WriteBatch
2. 调用JoinBatchGroup，将Writer添加到WriteBatchGroup中，如果当前Writer作为当前WriteBatchGroup的leader，则JoinBatchGroup会立刻返回，如果当前Writer作为当前WriteBatchGroup的follower，则必须等待当前Writer的状态变为STATE_GROUP_LEADER | STATE_PARALLEL_FOLLOWER | STATE_COMPLETED才会从JoinBatchGroup返回
3. 检查是否需要执行MemTable Flush，如果需要，则触发或者调度执行MemTable Flush
	- 在非单ColumnFamily模式下，如果WAL log的总大小超过了阈值且当前没有触发MemTable flush操作以释放WAL log，则触发MemTable flush
		- 获取最早的被保留的WAL log file number，记为flush_column_family_if_log_file
		- 遍历所有的ColumnFamily，对每个ColumnFamily处理如下：
			- 获取每个ColumnFamily对应的最早的具有该ColumnFamily的数据的WAL log file number，记为column_family_earliest_log_file
			- 如果column_family_earliest_log_file <= flush_column_family_if_log_file，则触发关于该ColumnFamily的Flush操作，以求能够释放一些WAL log
	- 否则，如果WriteBuffer中已用空间超过了buffer size，则从所有ColumnFamily中找出占用空间最大的MemTable对应的ColumnFamily，并触发关于该MemTable的flush操作
4. 检查DBImpl::flush_scheduler_中是否存在待flush的ColumnFamily，如果有，则调度关于这些ColumnFamily的MemTable flush操作，但是在ScheduleFlushes中只会执行SwitchMemtable，将当前的Mutable MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
	- MemTable的写入操作是由MemTableInserter完成的，MemTableInserter在每次写入MemTable之后，会检查是否需要调度关于该MemTable的flush(MemTable::flush_state_为FlushState::kRequested状态，则表明需要调度)，如果需要调度关于该MemTable的flush，则会将对应的ColumnFamily添加到DBImpl::flush_scheduler_中
5. 检查write_controller_是否需要write stall，如果需要，则会延迟写入
6. 调用EnterAsBatchGroupLeader，以确定当前提交的WriteBatchGroup中应该包含哪些WriteBatch
7. 检查是否leader Writer和follower Writer可以并发执行
8. 将当前WriteBatchGroup中所有的WriteBatch合并成一个WriteBatch，并将合并后的WriteBatch写入WAL log
9. 如果WriteOptions中设置了sync标识，则sync所有的WAL log
10. 将当前WriteBatchGroup中的数据写MemTable
	- 如果不能并发写入
		- 由leader writer所在的线程来负责当前WriteBatchGroup中所有WriteBatch的写入，遍历WriteBatchGroup中的每个WriteBatch，并通过MemTableInserter来完成每个WriteBatch中的每个记录的写入
	- 如果可以并发写入
		- 由leader Writer所在的线程进行并发写之前的准备工作
			- 设置当前WriteBatchGroup中每个WriteBatch对应的Writer对应的Sequence Number
			- 设置当前WriteBatchGroup中所有follower Writer的state为STATE_PARALLEL_FOLLOWER，之后当前WriteBatchGroup中所有的follower Writer所在的线程就可以将它对应的WriteBatch写入MemTable
				- 当follower Writer所在的线程完成MemTable write之后，会调用CompleteParallelWorker来判断是否由它来完成当前WriteBatchGroup的收尾工作，如果是，则调用EarlyExitParallelGroup来完成当前WriteBatchGroup的收尾工作
		- 将leader Writer对应的WriteBatch写入MemTable，当leader Writer所在的线程完成MemTable write之后，会调用CompleteParallelWorker来判断是否由它来完成当前WriteBatchGroup的收尾工作，如果是，则调用ExitAsBatchGroupLeader来完成当前WriteBatchGroup的收尾工作
		
上述逻辑中，除“leader Writer和follower Writer可以并发写入的情况下，follower Writer将它对应的WriteBatch写入MemTable，以及可能的由follower Writer完成当前WriteBatchGroup的收尾工作”之外，所有的逻辑都是由leader Writer所在的线程完成。

```
Status DBImpl::WriteImpl(const WriteOptions& write_options,
                         WriteBatch* my_batch, WriteCallback* callback) {
  Status status;

  # 新建一个WriteThread::Writer对象，关联到传入的WriteBatch
  WriteThread::Writer w;
  w.batch = my_batch;
  w.sync = write_options.sync;
  w.disableWAL = write_options.disableWAL;
  w.in_batch_group = false;
  # 在DB::Put()调用上下文中，callback为null
  w.callback = callback;

  StopWatch write_sw(env_, db_options_.statistics.get(), DB_WRITE);

  # JoinBatchGroup调用LinkOne(w, &linked_as_leader)将当前的writer连接到链表中，
  # 其中write_thread_.newest_writer_是链表的头，是最新加入的follower，而第一个加
  # 入链表的将作为当前group的leader(link_older=nullptr)
  # 
  # 如果当前的writer是作为leader，则可以直接从JoinBatchGroup返回，否则，必须等待
  # 当前的writer.state变为STATE_GROUP_LEADER | STATE_PARALLEL_FOLLOWER | STATE_COMPLETED
  # 才会从JoinBatchGroup返回
  write_thread_.JoinBatchGroup(&w);
  if (w.state == WriteThread::STATE_PARALLEL_FOLLOWER) {
    # 这里进入Follower的处理逻辑，当且仅当leader Writer调用了WriteThread::LaunchParallelFollowers
    # 时，才会将当前WriteBatchGroup中的所有follower设置为 WriteThread::STATE_PARALLEL_FOLLOWER状态，
    # 而leader writer调用WriteThread::LaunchParallelFollowers时，一定是处于并发执行的情况下，也就是
    # 下文代码中的parallel为true的情况，此时leader writer和所有follower write都在它自身调用
    # DBImpl::WriteImpl的那个线程中执行
    
    if (!w.CallbackFailed()) {
      ColumnFamilyMemTablesImpl column_family_memtables(
          versions_->GetColumnFamilySet());
      WriteBatchInternal::SetSequence(w.batch, w.sequence);
      InsertFlags insert_flags{InsertFlag::kConcurrentMemtableWrites};
      
      # 处理当前Writer对应的WriteBatch，会进一步调用WriteBatch::Iterate来处理WriteBatch中的每一个
      # 记录，针对每个记录的处理是由MemTableInserter完成的，关于MemTableInserter的分析见下文
      w.status = WriteBatchInternal::InsertInto(
          w.batch, &column_family_memtables, &flush_scheduler_,
          write_options.ignore_missing_column_families, 0 /*log_number*/, this, insert_flags);
    }

    # 在当前上下文中(当前线程是follower Writer所在的线程)，对于CompleteParallelWorker方法：
    # 1. 如果当前线程(follower Writer所在的线程)不是最后一个完成MemTable write的，则必须
    # 由leader Writer所在的线程完成当前WriteBatchGroup的收尾工作
    # 2. 如果当前线程(follower Writer所在的线程)是最后一个完成MemTable write的，且
    # early_exit_allowed被设置为true，则由当前线程完成当前WriteBatchGroup的收尾工作
    # 3. 如果当前线程(follower Writer所在的线程)是最后一个完成MemTable write的，但
    # early_exit_allowed被设置为false，则将leader Writer设置为STATE_COMPLETED状态，且
    # 等待自身变为STATE_COMPLETED状态，leader将在WriteThread::ExitAsBatchGroupLeader
    # 中将所有follower Writer设置为STATE_COMPLETED状态
    if (write_thread_.CompleteParallelWorker(&w)) {
      # 如果WriteBatchGroup中所有的Writer都已经完成，且允许early exit，则由当前的follower Writer
      # 负责结束当前的WriteBatchGroup，并且更新VersionSet::last_sequence_
      // we're responsible for early exit
      auto last_sequence = w.parallel_group->last_sequence;
      SetTickerCount(stats_, SEQUENCE_NUMBER, last_sequence);
      versions_->SetLastSequence(last_sequence);
      write_thread_.EarlyExitParallelGroup(&w);
    }
    assert(w.state == WriteThread::STATE_COMPLETED);
    // STATE_COMPLETED conditional below handles exit

    status = w.FinalStatus();
  }
  
  # 如果当前的Writer已经完成，则直接返回其状态
  if (w.state == WriteThread::STATE_COMPLETED) {
    // write is complete and leader has updated sequence
    return w.FinalStatus();
  }
  
  # 否则，当前的Writer是当前WriteBatchGroup的leader
  // else we are the leader of the write batch group
  assert(w.state == WriteThread::STATE_GROUP_LEADER);

  # 下面的逻辑都是leader的逻辑

  WriteContext context;
  mutex_.Lock();

  if (!write_options.disableWAL) {
    default_cf_internal_stats_->AddDBStats(InternalDBStatsType::WRITE_WITH_WAL, 1);
  }

  // Once reaches this point, the current writer "w" will try to do its write
  // job.  It may also pick up some of the remaining writers in the "writers_"
  // when it finds suitable, and finish them in the same write batch.
  // This is how a write job could be done by the other writer.
  assert(!single_column_family_mode_ ||
         versions_->GetColumnFamilySet()->NumberOfColumnFamilies() == 1);

  # 获取最大的wal log容量
  uint64_t max_total_wal_size = (db_options_.max_total_wal_size == 0)
                                    ? 4 * max_total_in_memory_state_
                                    : db_options_.max_total_wal_size;
  # 在非单ColumnFamily模式下，如果WAL log的总大小超过了阈值且当前没有触发MemTable
  # flush操作以释放WAL log，则触发MemTable flush
  if (UNLIKELY(!single_column_family_mode_ &&
               alive_log_files_.begin()->getting_flushed == false &&
               total_log_size() > max_total_wal_size)) {
    # 最早的被保留的WAL log file number
    uint64_t flush_column_family_if_log_file = alive_log_files_.begin()->number;
    alive_log_files_.begin()->getting_flushed = true;
    for (auto cfd : *versions_->GetColumnFamilySet()) {
      if (cfd->IsDropped()) {
        continue;
      }
      
      # cfd->GetLogNumber()返回最早的具有该ColumnFamily的数据的WAL log，如果该WAL log
      # file number不大于flush_column_family_if_log_file，则触发关于该ColumnFamily的
      # Flush操作，以求能够释放一些WAL log
      if (cfd->GetLogNumber() <= flush_column_family_if_log_file) {
        status = SwitchMemtable(cfd, &context);
        if (!status.ok()) {
          break;
        }
        cfd->imm()->FlushRequested();
        SchedulePendingFlush(cfd);
      }
    }
    MaybeScheduleFlushOrCompaction();
  } else if (UNLIKELY(write_buffer_.ShouldFlush())) {
    # 如果WriteBuffer中已用空间超过了buffer size，则从所有ColumnFamily中找出占用空间
    # 最大的MemTable对应的ColumnFamily，并触发关于该MemTable的flush操作
    ColumnFamilyData* largest_cfd = nullptr;
    size_t largest_cfd_size = 0;

    for (auto cfd : *versions_->GetColumnFamilySet()) {
      if (cfd->IsDropped()) {
        continue;
      }
      if (!cfd->mem()->IsEmpty()) {
        // We only consider active mem table, hoping immutable memtable is
        // already in the process of flushing.
        size_t cfd_size = cfd->mem()->ApproximateMemoryUsage();
        if (largest_cfd == nullptr || cfd_size > largest_cfd_size) {
          largest_cfd = cfd;
          largest_cfd_size = cfd_size;
        }
      }
    }
    if (largest_cfd != nullptr) {
      status = SwitchMemtable(largest_cfd, &context);
      if (status.ok()) {
        largest_cfd->imm()->FlushRequested();
        SchedulePendingFlush(largest_cfd);
        MaybeScheduleFlushOrCompaction();
      }
    }
  }

  if (UNLIKELY(status.ok() && !bg_error_.ok())) {
    status = bg_error_;
  }

  # MemTable的写入操作是由MemTableInserter完成的，MemTableInserter在每次写入
  # MemTable之后，会检查是否需要调度关于该MemTable的flush(MemTable::flush_state_为
  # FlushState::kRequested状态，则表明需要调度)，如果需要调度关于该MemTable的flush，
  # 则会将对应的ColumnFamily添加到DBImpl::flush_scheduler_中
  #
  # 如果DBImpl::flush_scheduler_中存在待flush的ColumnFamily，则调度关于这些ColumnFamily
  # 的MemTable flush操作，但是在ScheduleFlushes中只会执行SwitchMemtable，将当前的Mutable
  # MemTable转换为Immutable MemTable，并创建新的Mutable MemTable
  if (UNLIKELY(status.ok() && !flush_scheduler_.Empty())) {
    status = ScheduleFlushes(&context);
  }

  # write_controller_用于控制write stall
  if (UNLIKELY(status.ok() && (write_controller_.IsStopped() ||
                               write_controller_.NeedsDelay()))) {
    # 如果需要write stall，则延迟写入
    status = DelayWrite(last_batch_group_size_);
  }

  # 当前最大的Sequence Number，Sequence Number用于控制数据可见性
  uint64_t last_sequence = versions_->LastSequence();
  WriteThread::Writer* last_writer = &w;
  autovector<WriteThread::Writer*> write_group;
  bool need_log_sync = !write_options.disableWAL && write_options.sync;
  bool need_log_dir_sync = need_log_sync && !log_dir_synced_;

  if (status.ok()) {
    if (need_log_sync) {
      while (logs_.front().getting_synced) {
        log_sync_cv_.Wait();
      }
      for (auto& log : logs_) {
        assert(!log.getting_synced);
        log.getting_synced = true;
      }
    }

    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into memtables
  }

  mutex_.Unlock();

  // At this point the mutex is unlocked

  bool exit_completed_early = false;
  # EnterAsBatchGroupLeader用于确定当前提交的WriteBatchGroup中应该包含哪些WriteBatch
  # EnterAsBatchGroupLeader的实现见下文的分析
  # 
  # w表示当前WriteBatchGroup的leader Writer
  # last_writer表示该WriteBatchGroup中的最后一个Writer
  # write_group表示当前的WriteBatchGroup
  last_batch_group_size_ =
      write_thread_.EnterAsBatchGroupLeader(&w, &last_writer, &write_group);

  if (status.ok()) {
    // Rules for when we can update the memtable concurrently
    // 1. supported by memtable
    // 2. Puts are not okay if inplace_update_support
    // 3. Deletes or SingleDeletes are not okay if filtering deletes
    //    (controlled by both batch and memtable setting)
    // 4. Merges are not okay
    // 5. YugaByte-specific user-specified sequence numbers are currently not compatible with
    //    parallel memtable writes.
    //
    // Rules 1..3 are enforced by checking the options
    // during startup (CheckConcurrentWritesSupported), so if
    // options.allow_concurrent_memtable_write is true then they can be
    // assumed to be true.  Rule 4 is checked for each batch.  We could
    // relax rules 2 and 3 if we could prevent write batches from referring
    // more than once to a particular key.
    
    # 检查是否leader Writer和follower Writer可以并发执行，同时，计算这个WriteBatchGroup中
    # 有多少个操作(total_count)以及总的记录的大小(total_byte_size)
    #
    # 变量parallel用于表示leader Writer和follower Writer可以并发执行，并发的前提条件是：
    # 1. memtable支持
    # 2. allow_concurrent_memtable_write设置
    # 3. write_group有多个batch
    # 4. batch中没有merge操作
    bool parallel =
        db_options_.allow_concurrent_memtable_write && write_group.size() > 1;
    size_t total_count = 0;
    uint64_t total_byte_size = 0;
    for (auto writer : write_group) {
      if (writer->CheckCallback(this)) {
        total_count += WriteBatchInternal::Count(writer->batch);
        total_byte_size = WriteBatchInternal::AppendedByteSize(
            total_byte_size, WriteBatchInternal::ByteSize(writer->batch));
        parallel = parallel && !writer->batch->HasMerge();
      }
    }

    # current_sequence将作为当前WriteBatchGroup中的第一个Sequence
    const SequenceNumber current_sequence = last_sequence + 1;

    # 当当前WriteBatchGroup中所有WriteBatch都执行成功的话，系统最新的sequence number
    # 就是last_sequence
    last_sequence += total_count;

    # 接下来，先写WAL log，由leader Writer所在的线程执行WAL log写
    uint64_t log_size = 0;
    if (!write_options.disableWAL) {
      # 将当前WriteBatchGroup中所有的WriteBatch合并成一个WriteBatch，保存在merged_batch中
      WriteBatch* merged_batch = nullptr;
      if (write_group.size() == 1 && !write_group[0]->CallbackFailed()) {
        merged_batch = write_group[0]->batch;
      } else {
        // WAL needs all of the batches flattened into a single batch.
        // We could avoid copying here with an iov-like AddRecord
        // interface
        merged_batch = &tmp_batch_;
        for (auto writer : write_group) {
          if (!writer->CallbackFailed()) {
            WriteBatchInternal::Append(merged_batch, writer->batch);
          }
        }
      }
      
      WriteBatchInternal::SetSequence(merged_batch, current_sequence);

      CHECK_EQ(WriteBatchInternal::Count(merged_batch), total_count);

      # merged_batch中的内容就是要写入WAL log中的数据
      Slice log_entry = WriteBatchInternal::Contents(merged_batch);
      log::Writer* log_writer;
      LogFileNumberSize* last_alive_log_file;
      {
        InstrumentedMutexLock l(&mutex_);
        # 获取最新的log writer和log file
        log_writer = logs_.back().writer;
        last_alive_log_file = &alive_log_files_.back();
      }
      
      # 写入WAL log
      status = log_writer->AddRecord(log_entry);
      total_log_size_.fetch_add(static_cast<int64_t>(log_entry.size()));
      last_alive_log_file->AddSize(log_entry.size());
      log_empty_ = false;
      log_size = log_entry.size();
      RecordTick(stats_, WAL_FILE_BYTES, log_size);
      if (status.ok() && need_log_sync) {
        StopWatch sw(env_, stats_, WAL_FILE_SYNC_MICROS);
        // It's safe to access logs_ with unlocked mutex_ here because:
        //  - we've set getting_synced=true for all logs,
        //    so other threads won't pop from logs_ while we're here,
        //  - only writer thread can push to logs_, and we're in
        //    writer thread, so no one will push to logs_,
        //  - as long as other threads don't modify it, it's safe to read
        //    from std::deque from multiple threads concurrently.
        
        # sync所有的WAL log
        for (auto& log : logs_) {
          status = log.writer->file()->Sync(db_options_.use_fsync);
          if (!status.ok()) {
            break;
          }
        }
        
        # sync log directory
        if (status.ok() && need_log_dir_sync) {
          // We only sync WAL directory the first time WAL syncing is
          // requested, so that in case users never turn on WAL sync,
          // we can avoid the disk I/O in the write code path.
          status = directories_.GetWalDir()->Fsync();
        }
      }

      if (merged_batch == &tmp_batch_) {
        tmp_batch_.Clear();
      }
    }
    
    if (status.ok()) {
      PERF_TIMER_GUARD(write_memtable_time);

      {
        # 更新stats信息，略
        ...
      }

      if (!parallel) {
        # 如果leader Writer和follower Writer不能并发写，则由leader writer来负责当前
        # WriteBatchGroup中所有WriteBatch的写入，遍历WriteBatchGroup中的每个WriteBatch，
        # 并通过MemTableInserter来完成每个WriteBatch中的每个记录的写入
        InsertFlags insert_flags{InsertFlag::kFilterDeletes};
        status = WriteBatchInternal::InsertInto(
            write_group, current_sequence, column_family_memtables_.get(),
            &flush_scheduler_, write_options.ignore_missing_column_families,
            0 /*log_number*/, this, insert_flags);

        if (status.ok()) {
          status = w.FinalStatus();
        }
      } else {
        # leader Writer和follower Writer可以并发执行，则由当前线程(leader Writer所在的线程)
        # 进行并发写之前的准备工作
        WriteThread::ParallelGroup pg;
        pg.leader = &w;
        pg.last_writer = last_writer;
        pg.last_sequence = last_sequence;
        # 如果need_log_sync为false，则early_exit_allowed为true
        pg.early_exit_allowed = !need_log_sync;
        pg.running.store(static_cast<uint32_t>(write_group.size()),
                         std::memory_order_relaxed);
        # 设置当前WriteBatchGroup中每个WriteBatch对应的Writer对应的Sequence Number，
        # WriteBatchGroup以及state，所有follower Writer的state都将被设置为STATE_PARALLEL_FOLLOWER
        write_thread_.LaunchParallelFollowers(&pg, current_sequence);

        if (!w.CallbackFailed()) {
          # leader Writer写它对应的WriteBatch
          ColumnFamilyMemTablesImpl column_family_memtables(
              versions_->GetColumnFamilySet());
          assert(w.sequence == current_sequence);
          # 设置WriteBatch对应的Sequence Number
          WriteBatchInternal::SetSequence(w.batch, w.sequence);
          InsertFlags insert_flags{InsertFlag::kConcurrentMemtableWrites};
          # 将leader Writer对应的WriteBatch写入MemTable
          w.status = WriteBatchInternal::InsertInto(
              w.batch, &column_family_memtables, &flush_scheduler_,
              write_options.ignore_missing_column_families, 0 /*log_number*/,
              this, insert_flags);
        }

        // CompleteParallelWorker returns true if this thread should
        // handle exit, false means somebody else did
        # 检查是否由当前线程完成WriteBatchGroup最后的收尾工作
        #
        # 在当前上下文中(当前线程是leader Writer所在的线程)，对于CompleteParallelWorker方法：
        # 1. 如果当前线程(leader Writer所在的线程)不是最后一个完成MemTable write的，则必须
        # 等待leader writer.state变为STATE_COMPLETED之后才能返回，在返回的时候，如果
        # early_exit_allowed被设置为false，则由当前线程完成当前WriteBatchGroup的收尾工作，
        # leader writer.state是由最后一个完成MemTable write的follower writer设置为STATE_COMPLETED
        # 2. 如果当前线程(leader Writer所在的线程)是最后一个完成MemTable write的，则由
        # 当前线程完成当前WriteBatchGroup的收尾工作
        exit_completed_early = !write_thread_.CompleteParallelWorker(&w);
        status = w.FinalStatus();
      }

      # 由当前线程完成当前WriteBatchGroup的收尾工作
      if (!exit_completed_early && w.status.ok()) {
        SetTickerCount(stats_, SEQUENCE_NUMBER, last_sequence);
        # 设置最新的Sequence Number
        versions_->SetLastSequence(last_sequence);
        if (!need_log_sync) {
          # need_log_sync为false，表明允许early exit，由leader Writer执行当前WriteBatchGroup的
          # 收尾工作，否则需要在最后才调用ExitAsBatchGroupLeader执行收尾工作
          write_thread_.ExitAsBatchGroupLeader(&w, last_writer, w.status);
          exit_completed_early = true;
        }
      }

      if (!status.ok() && bg_error_.ok() && !w.CallbackFailed()) {
        bg_error_ = status;
      }
    }
  }

  if (db_options_.paranoid_checks && !status.ok() && !w.CallbackFailed() && !status.IsBusy()) {
    mutex_.Lock();
    if (bg_error_.ok()) {
      bg_error_ = status;  // stop compaction & fail any further writes
    }
    mutex_.Unlock();
  }

  if (need_log_sync) {
    mutex_.Lock();
    MarkLogsSynced(logfile_number_, need_log_dir_sync, status);
    mutex_.Unlock();
  }

  if (!exit_completed_early) {
    write_thread_.ExitAsBatchGroupLeader(&w, last_writer, w.status);
  }

  return status;
}
```

## WriteThread
### WriteThread::JoinBatchGroup
JoinBatchGroup调用LinkOne(w, &linked_as_leader)将所有的writer连接成一个单向链表(借助于指针link_older)，其中write_thread_.newest_writer_是链表的头，代表最新加入的writer，如果链表中在最新加入的writer之前没有其它writer，则最新加入的writer将作为leader writer。

如果当前的Writer成为了leader，则从WriteThread::JoinBatchGroup立即返回，然后做剩下的写入逻辑，如果当前已经有了leader，则当前的Writer会作为follower writer，follower writer需要等待Writer.state变为STATE_GROUP_LEADER | STATE_PARALLEL_FOLLOWER | STATE_COMPLETED之后才会从WriteThread::JoinBatchGroup返回。

如果leader writer和follower writer不能够并发执行，则leader writer和follower writer的MemTable写操作都是由leader writer所在的线程完成，leader writer所在的线程会逐一处理WriteBatchGroup中的每一个writer对应的WriteBatch，当WriteBatchGroup中所有WriteBatch都完成时，leader writer所在的线程会调用WriteThread::ExitAsBatchGroupLeader来完成WriteBatchGroup的收尾工作，其中会将WriteBatchGroup中每一个Writer的状态设置为STATE_COMPLETED。

如果leader writer和follower writer能够并发执行，则leader writer和follower writer的MemTable写操作分别由各自的writer所在的线程完成，当WriteBatchGroup中所有WriteBatch都完成时，可能由leader writer所在的线程调用WriteThread::ExitAsBatchGroupLeader来完成WriteBatchGroup的收尾工作，也可能由follower writer所在的线程调用WriteThread::EarlyExitParallelGroup来完成WriteBatchGroup的收尾工作，最终都会将WriteBatchGroup中每一个Writer的状态设置为STATE_COMPLETED。

```
void WriteThread::JoinBatchGroup(Writer* w) {
  static AdaptationContext ctx("JoinBatchGroup");

  assert(w->batch != nullptr);
  bool linked_as_leader;
  # 如果w所代表的Writer成为了leader，则linked_as_leader被设置为true
  LinkOne(w, &linked_as_leader);

  if (!linked_as_leader) {
    # 对于follower Writer，需要等待Writer.state变为STATE_GROUP_LEADER | STATE_PARALLEL_FOLLOWER | 
    # STATE_COMPLETED之后才会从WriteThread::JoinBatchGroup返回
    AwaitState(w,
               STATE_GROUP_LEADER | STATE_PARALLEL_FOLLOWER | STATE_COMPLETED,
               &ctx);
    TEST_SYNC_POINT_CALLBACK("WriteThread::JoinBatchGroup:DoneWaiting", w);
  }
}

void WriteThread::LinkOne(Writer* w, bool* linked_as_leader) {
  assert(w->state == STATE_INIT);

  # 原子性的获取最新的writer
  Writer* writers = newest_writer_.load(std::memory_order_relaxed);
  while (true) {
    w->link_older = writers;
    # 尝试将最新的writer更新为w，如果更新失败，则会在writers中返回最新的writer
    if (newest_writer_.compare_exchange_strong(writers, w)) {
      if (writers == nullptr) {
        # w是当前链表中的第一个writer(所有在w之前的writer都已经完成了)
        w->state.store(STATE_GROUP_LEADER, std::memory_order_relaxed);
      }
      *linked_as_leader = (writers == nullptr);
      return;
    }
  }
}
```

### WriteThread::EnterAsBatchGroupLeader
WriteThread::EnterAsBatchGroupLeader用于确定当前WriteBatchGroup中应该包含哪些数据。主要逻辑如下：
- 计算当前可以批量提交的最大长度max_size，如果leader writer对应的WriteBatch size小于128KB，则max_size等于leader writer对应的WriteBatch size加上128KB，否则，max_size=1MB
- 调用CreateMissingNewerLinks(newest_writer)，将整个链表的反向链接建立起来(借助于指针link_newer)，形成一个双向链表，目的是为了从leader开始遍历所有应当被包含在当前WriteBatchGroup中的writer
- 从leader开始反向遍历(沿着link_newer指针)，一直到newest_writer，确定WriteBatchGroup中应当包含哪些writer：
    - 检查当前的writer是否应当被添加到以leader为首的WriteBatchGroup中，每当遇到以下条件就停止查找
        - leader writer没有指定sync标识，但是当前writer指定了sync标识
        - leader writer指定了disableWAL标识，但是当前writer没有指定disableWAL标识
        - 当前writer关联的WriteBatch为空
        - 当前writer不想被以batch模式执行
        - 如果加上当前writer之后，WriteBatchGroup超过了max_size
            - 对于允许添加到当前WriteBatchGroup中的writer，会计算它们总的大小

```
size_t WriteThread::EnterAsBatchGroupLeader(
    Writer* leader, WriteThread::Writer** last_writer,
    autovector<WriteThread::Writer*>* write_batch_group) {
  assert(leader->link_older == nullptr);
  assert(leader->batch != nullptr);

  size_t size = WriteBatchInternal::ByteSize(leader->batch);
  write_batch_group->push_back(leader);

  // Allow the group to grow up to a maximum size, but if the
  // original write is small, limit the growth so we do not slow
  // down the small write too much.
  size_t max_size = 1 << 20;
  if (size <= (128 << 10)) {
    max_size = size + (128 << 10);
  }

  *last_writer = leader;

  Writer* newest_writer = newest_writer_.load(std::memory_order_acquire);

  // This is safe regardless of any db mutex status of the caller. Previous
  // calls to ExitAsGroupLeader either didn't call CreateMissingNewerLinks
  // (they emptied the list and then we added ourself as leader) or had to
  // explicitly wake us up (the list was non-empty when we added ourself,
  // so we have already received our MarkJoined).
  CreateMissingNewerLinks(newest_writer);

  // Tricky. Iteration start (leader) is exclusive and finish
  // (newest_writer) is inclusive. Iteration goes from old to new.
  Writer* w = leader;
  while (w != newest_writer) {
    w = w->link_newer;

    if (w->sync && !leader->sync) {
      // Do not include a sync write into a batch handled by a non-sync write.
      break;
    }

    if (!w->disableWAL && leader->disableWAL) {
      // Do not include a write that needs WAL into a batch that has
      // WAL disabled.
      break;
    }

    if (w->batch == nullptr) {
      // Do not include those writes with nullptr batch. Those are not writes,
      // those are something else. They want to be alone
      break;
    }

    if (w->callback != nullptr && !w->callback->AllowWriteBatching()) {
      // dont batch writes that don't want to be batched
      break;
    }

    auto batch_size = WriteBatchInternal::ByteSize(w->batch);
    if (size + batch_size > max_size) {
      // Do not make batch too big
      break;
    }

    size += batch_size;
    write_batch_group->push_back(w);
    w->in_batch_group = true;
    *last_writer = w;
  }
  return size;
}
```

### WriteThread::CompleteParallelWorker
WriteThread::CompleteParallelWorker用于检查是否由当前线程完成WriteBatchGroup最后的收尾工作。

如果当前线程是follower Writer所在的线程，则：
1. 如果当前线程不是最后一个完成MemTable write的，则必须由leader Writer所在的线程完成当前WriteBatchGroup的收尾工作
2. 如果当前线程是最后一个完成MemTable write的，且early_exit_allowed被设置为true，则由当前线程完成当前WriteBatchGroup的收尾工作
3. 如果当前线程是最后一个完成MemTable write的，但early_exit_allowed被设置为false，则将leader Writer设置为STATE_COMPLETED状态，且等待自身变为STATE_COMPLETED状态，leader将在WriteThread::ExitAsBatchGroupLeader中将所有follower Writer设置为STATE_COMPLETED状态
    
如果当前线程是leader Writer所在的线程，则：
1. 如果当前线程不是最后一个完成MemTable write的，则必须等待leader writer.state变为STATE_COMPLETED之后才能返回，在返回的时候，如果 early_exit_allowed被设置为false，则由当前线程完成当前WriteBatchGroup的收尾工作，leader writer.state是由最后一个完成MemTable write的follower writer设置为STATE_COMPLETED
2. 如果当前线程是最后一个完成MemTable write的，则由当前线程完成当前WriteBatchGroup的收尾工作

```
bool WriteThread::CompleteParallelWorker(Writer* w) {
  static AdaptationContext ctx("CompleteParallelWorker");

  auto* pg = w->parallel_group;
  if (!w->status.ok()) {
    std::lock_guard<std::mutex> guard(w->StateMutex());
    pg->status = w->status;
  }

  auto leader = pg->leader;
  auto early_exit_allowed = pg->early_exit_allowed;

  if (pg->running.load(std::memory_order_acquire) > 1 && pg->running-- > 1) {
    // we're not the last one
    AwaitState(w, STATE_COMPLETED, &ctx);

    // Caller only needs to perform exit duties if early exit doesn't
    // apply and this is the leader.  Can't touch pg here.  Whoever set
    // our state to STATE_COMPLETED copied pg->status to w.status for us.
    return w == leader && !(early_exit_allowed && w->status.ok());
  }
  // else we're the last parallel worker

  if (w == leader || (early_exit_allowed && pg->status.ok())) {
    // this thread should perform exit duties
    w->status = pg->status;
    return true;
  } else {
    // We're the last parallel follower but early commit is not
    // applicable.  Wake up the leader and then wait for it to exit.
    assert(w->state == STATE_PARALLEL_FOLLOWER);
    SetState(leader, STATE_COMPLETED);
    AwaitState(w, STATE_COMPLETED, &ctx);
    return false;
  }
}
```

### WriteThread::ExitAsBatchGroupLeader
WriteThread::ExitAsBatchGroupLeader主要执行如下：
- 在上一次调用WriteThread::EnterAsBatchGroupLeader和当前调用WriteThread::ExitAsBatchGroupLeader之间可能writer链表中又加入了新的writer，调用CreateMissingNewerLinks建立反向链接
- 选出新的leader writer
- 遍历当前WriteBatchGroup中所有的writer, 设置其state为STATE_COMPLETED，这样任何正在调用CompleteParallelWorker的follower writer所在的线程就会退出

```
void WriteThread::ExitAsBatchGroupLeader(Writer* leader, Writer* last_writer,
                                         Status status) {
  assert(leader->link_older == nullptr);

  Writer* head = newest_writer_.load(std::memory_order_acquire);
  if (head != last_writer ||
      !newest_writer_.compare_exchange_strong(head, nullptr)) {
    // Either w wasn't the head during the load(), or it was the head
    // during the load() but somebody else pushed onto the list before
    // we did the compare_exchange_strong (causing it to fail).  In the
    // latter case compare_exchange_strong has the effect of re-reading
    // its first param (head).  No need to retry a failing CAS, because
    // only a departing leader (which we are at the moment) can remove
    // nodes from the list.
    assert(head != last_writer);

    // After walking link_older starting from head (if not already done)
    // we will be able to traverse w->link_newer below. This function
    // can only be called from an active leader, only a leader can
    // clear newest_writer_, we didn't, and only a clear newest_writer_
    // could cause the next leader to start their work without a call
    // to MarkJoined, so we can definitely conclude that no other leader
    // work is going on here (with or without db mutex).
    CreateMissingNewerLinks(head);
    assert(last_writer->link_newer->link_older == last_writer);
    last_writer->link_newer->link_older = nullptr;

    // Next leader didn't self-identify, because newest_writer_ wasn't
    // nullptr when they enqueued (we were definitely enqueued before them
    // and are still in the list).  That means leader handoff occurs when
    // we call MarkJoined
    SetState(last_writer->link_newer, STATE_GROUP_LEADER);
  }
  // else nobody else was waiting, although there might already be a new
  // leader now

  while (last_writer != leader) {
    last_writer->status = status;
    // we need to read link_older before calling SetState, because as soon
    // as it is marked committed the other thread's Await may return and
    // deallocate the Writer.
    auto next = last_writer->link_older;
    SetState(last_writer, STATE_COMPLETED);

    last_writer = next;
  }
}
```
    
### WriteThread::EarlyExitParallelGroup
WriteThread::EarlyExitParallelGroup当且仅当某个follower writer所在的线程最后一个完成MemTable write且early_exit_allowed为true的情况下被调用。它会进一步调用WriteThread::ExitAsBatchGroupLeader来完成收尾工作。

```
void WriteThread::EarlyExitParallelGroup(Writer* w) {
  auto* pg = w->parallel_group;

  assert(w->state == STATE_PARALLEL_FOLLOWER);
  assert(pg->status.ok());
  # 选举新的leader writer，设置所有follower writer的状态为STATE_COMPLETED
  ExitAsBatchGroupLeader(pg->leader, pg->last_writer, pg->status);
  assert(w->status.ok());
  assert(w->state == STATE_COMPLETED);
  # 设置leader writer的状态为STATE_COMPLETED，此时leader writer所在的线程正在
  # WriteThread::CompleteParallelWorker中等待自身状态改变为STATE_COMPLETED
  SetState(pg->leader, STATE_COMPLETED);
}
```

是否可能出现“follower writer所在的线程调用了WriteThread::EarlyExitParallelGroup，同时leader writer所在的线程调用WriteThread::ExitAsBatchGroupLeader”的情况呢？答案是不可能。分析如下：
- 如果leader和follower不能并发执行，则一定是由leader writer所在的线程调用WriteThread::ExitAsBatchGroupLeader
- 如果leader和follower可以并发执行，则：
    - 如果follower writer所在的线程调用了WriteThread::EarlyExitParallelGroup，那么一定满足“follower线程最后一个执行完MemTable write，且early_exit_allowed被设置为true”，那么当leader writer所在的线程一定不是最后一个执行完MemTable write，当它调用WriteThread::CompleteParallelWorker的时候，一定会返回false
    - 如果leader writer所在的线程调用WriteThread::ExitAsBatchGroupLeader，那么要么满足“leader writer所在的线程不是最后一个执行完MemTable write，且early_exit_allowed被设置为false”，要么满足“leader write所在的线程是最后一个完成MemTable write的”，对于前面一种情况，因为early_exit_allowed被设置为false，follower writer所在的线程在调用WriteThread::CompleteParallelWorker的时候，一定会返回false，对于后面一种情况，因为follower writer所在的线程不是最后一个完成MemTable write，当它调用WriteThread::CompleteParallelWorker的时候，一定会返回false


