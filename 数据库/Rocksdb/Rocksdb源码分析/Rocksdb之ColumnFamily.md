# 提纲
[toc]

# ColumnFamilyHandle
ColumnFamilyHandle是接口类，定义了从ColumnFamilyHandle获取ColumnFamily name，ColumnFamily ID和ColumnFamily descriptor的接口。
```
class ColumnFamilyHandle {
 public:
  virtual ~ColumnFamilyHandle() {}
  // Returns the name of the column family associated with the current handle.
  virtual const std::string& GetName() const = 0;
  // Returns the ID of the column family associated with the current handle.
  virtual uint32_t GetID() const = 0;
  // Fills "*desc" with the up-to-date descriptor of the column family
  // associated with this handle. Since it fills "*desc" with the up-to-date
  // information, this call might internally lock and release DB mutex to
  // access the up-to-date CF options.  In addition, all the pointer-typed
  // options cannot be referenced any longer than the original options exist.
  //
  // Note that this function is not supported in RocksDBLite.
  virtual Status GetDescriptor(ColumnFamilyDescriptor* desc) = 0;
  // Returns the comparator of the column family associated with the
  // current handle.
  virtual const Comparator* GetComparator() const = 0;
};
```

# ColumnFamilyHandleImpl
ColumnFamilyHandleImpl是ColumnFamilyHandle的实现类，用户通过ColumnFamilyHandleImpl来访问不同的ColumnFamilies
```
class ColumnFamilyHandleImpl : public ColumnFamilyHandle {
 public:
  // create while holding the mutex
  ColumnFamilyHandleImpl(
      ColumnFamilyData* cfd, DBImpl* db, InstrumentedMutex* mutex);
  // destroy without mutex
  virtual ~ColumnFamilyHandleImpl();
  virtual ColumnFamilyData* cfd() const { return cfd_; }

  /*以下4个方法都是借助于ColumnFamilyData来实现，不再赘述*/
  virtual uint32_t GetID() const override;
  virtual const std::string& GetName() const override;
  virtual Status GetDescriptor(ColumnFamilyDescriptor* desc) override;
  virtual const Comparator* GetComparator() const override;

 private:
  ColumnFamilyData* cfd_;
  DBImpl* db_;
  InstrumentedMutex* mutex_;
};
```

# ColumnFamilyData
ColumnFamilyData中保存了ColumnFamily相关的所有信息
```
class ColumnFamilyData {
 public:
  ~ColumnFamilyData();

  // thread-safe
  uint32_t GetID() const { return id_; }
  // thread-safe
  const std::string& GetName() const { return name_; }

  // Ref() can only be called from a context where the caller can guarantee
  // that ColumnFamilyData is alive (while holding a non-zero ref already,
  // holding a DB mutex, or as the leader in a write batch group).
  /*增加ColumnFamilyData的引用计数*/
  void Ref() { refs_.fetch_add(1, std::memory_order_relaxed); }

  // Unref decreases the reference count, but does not handle deletion
  // when the count goes to 0.  If this method returns true then the
  // caller should delete the instance immediately, or later, by calling
  // FreeDeadColumnFamilies().  Unref() can only be called while holding
  // a DB mutex, or during single-threaded recovery.
  /* 减少ColumnFamilyData的引用计数，如果引用计数降为0，则返回true，且外部调用
   * 者应当立即删除该对象
   */
  bool Unref() {
    int old_refs = refs_.fetch_sub(1, std::memory_order_relaxed);
    assert(old_refs > 0);
    return old_refs == 1;
  }

  // SetDropped() can only be called under following conditions:
  // 1) Holding a DB mutex,
  // 2) from single-threaded write thread, AND
  // 3) from single-threaded VersionSet::LogAndApply()
  // After dropping column family no other operation on that column family
  // will be executed. All the files and memory will be, however, kept around
  // until client drops the column family handle. That way, client can still
  // access data from dropped column family.
  // Column family can be dropped and still alive. In that state:
  // *) Compaction and flush is not executed on the dropped column family.
  // *) Client can continue reading from column family. Writes will fail unless
  // WriteOptions::ignore_missing_column_families is true
  // When the dropped column family is unreferenced, then we:
  // *) Remove column family from the linked list maintained by ColumnFamilySet
  // *) delete all memory associated with that column family
  // *) delete all the files associated with that column family
  void SetDropped();
  bool IsDropped() const { return dropped_; }

  // thread-safe
  int NumberLevels() const { return ioptions_.num_levels; }

  void SetLogNumber(uint64_t log_number) { log_number_ = log_number; }
  uint64_t GetLogNumber() const { return log_number_; }

  // thread-safe
  const EnvOptions* soptions() const;
  const ImmutableCFOptions* ioptions() const { return &ioptions_; }
  // REQUIRES: DB mutex held
  // This returns the MutableCFOptions used by current SuperVersion
  // You should use this API to reference MutableCFOptions most of the time.
  const MutableCFOptions* GetCurrentMutableCFOptions() const {
    return &(super_version_->mutable_cf_options);
  }
  // REQUIRES: DB mutex held
  // This returns the latest MutableCFOptions, which may be not in effect yet.
  const MutableCFOptions* GetLatestMutableCFOptions() const {
    return &mutable_cf_options_;
  }

  // REQUIRES: DB mutex held
  // Build ColumnFamiliesOptions with immutable options and latest mutable
  // options.
  ColumnFamilyOptions GetLatestCFOptions() const;

  bool is_delete_range_supported() { return is_delete_range_supported_; }

#ifndef ROCKSDB_LITE
  // REQUIRES: DB mutex held
  Status SetOptions(
      const std::unordered_map<std::string, std::string>& options_map);
#endif  // ROCKSDB_LITE

  InternalStats* internal_stats() { return internal_stats_.get(); }

  MemTableList* imm() { return &imm_; }
  MemTable* mem() { return mem_; }
  Version* current() { return current_; }
  Version* dummy_versions() { return dummy_versions_; }
  void SetCurrent(Version* _current);
  uint64_t GetNumLiveVersions() const;  // REQUIRE: DB mutex held
  uint64_t GetTotalSstFilesSize() const;  // REQUIRE: DB mutex held
  void SetMemtable(MemTable* new_mem) { mem_ = new_mem; }

  // calculate the oldest log needed for the durability of this column family
  uint64_t OldestLogToKeep();

  // See Memtable constructor for explanation of earliest_seq param.
  MemTable* ConstructNewMemtable(const MutableCFOptions& mutable_cf_options,
                                 SequenceNumber earliest_seq);
  void CreateNewMemtable(const MutableCFOptions& mutable_cf_options,
                         SequenceNumber earliest_seq);

  TableCache* table_cache() const { return table_cache_.get(); }

  // See documentation in compaction_picker.h
  // REQUIRES: DB mutex held
  bool NeedsCompaction() const;
  // REQUIRES: DB mutex held
  Compaction* PickCompaction(const MutableCFOptions& mutable_options,
                             LogBuffer* log_buffer);

  // Check if the passed range overlap with any running compactions.
  // REQUIRES: DB mutex held
  bool RangeOverlapWithCompaction(const Slice& smallest_user_key,
                                  const Slice& largest_user_key,
                                  int level) const;

  // A flag to tell a manual compaction is to compact all levels together
  // instad of for specific level.
  static const int kCompactAllLevels;
  // A flag to tell a manual compaction's output is base level.
  static const int kCompactToBaseLevel;
  // REQUIRES: DB mutex held
  Compaction* CompactRange(const MutableCFOptions& mutable_cf_options,
                           int input_level, int output_level,
                           uint32_t output_path_id, const InternalKey* begin,
                           const InternalKey* end, InternalKey** compaction_end,
                           bool* manual_conflict);

  CompactionPicker* compaction_picker() { return compaction_picker_.get(); }
  // thread-safe
  const Comparator* user_comparator() const {
    return internal_comparator_.user_comparator();
  }
  // thread-safe
  const InternalKeyComparator& internal_comparator() const {
    return internal_comparator_;
  }

  const std::vector<std::unique_ptr<IntTblPropCollectorFactory>>*
  int_tbl_prop_collector_factories() const {
    return &int_tbl_prop_collector_factories_;
  }

  SuperVersion* GetSuperVersion() { return super_version_; }
  // thread-safe
  // Return a already referenced SuperVersion to be used safely.
  SuperVersion* GetReferencedSuperVersion(InstrumentedMutex* db_mutex);
  // thread-safe
  // Get SuperVersion stored in thread local storage. If it does not exist,
  // get a reference from a current SuperVersion.
  SuperVersion* GetThreadLocalSuperVersion(InstrumentedMutex* db_mutex);
  // Try to return SuperVersion back to thread local storage. Retrun true on
  // success and false on failure. It fails when the thread local storage
  // contains anything other than SuperVersion::kSVInUse flag.
  bool ReturnThreadLocalSuperVersion(SuperVersion* sv);
  // thread-safe
  uint64_t GetSuperVersionNumber() const {
    return super_version_number_.load();
  }
  // will return a pointer to SuperVersion* if previous SuperVersion
  // if its reference count is zero and needs deletion or nullptr if not
  // As argument takes a pointer to allocated SuperVersion to enable
  // the clients to allocate SuperVersion outside of mutex.
  // IMPORTANT: Only call this from DBImpl::InstallSuperVersion()
  SuperVersion* InstallSuperVersion(SuperVersion* new_superversion,
                                    InstrumentedMutex* db_mutex,
                                    const MutableCFOptions& mutable_cf_options);
  SuperVersion* InstallSuperVersion(SuperVersion* new_superversion,
                                    InstrumentedMutex* db_mutex);

  void ResetThreadLocalSuperVersions();

  // Protected by DB mutex
  void set_pending_flush(bool value) { pending_flush_ = value; }
  void set_pending_compaction(bool value) { pending_compaction_ = value; }
  bool pending_flush() { return pending_flush_; }
  bool pending_compaction() { return pending_compaction_; }

  // Recalculate some small conditions, which are changed only during
  // compaction, adding new memtable and/or
  // recalculation of compaction score. These values are used in
  // DBImpl::MakeRoomForWrite function to decide, if it need to make
  // a write stall
  void RecalculateWriteStallConditions(
      const MutableCFOptions& mutable_cf_options);

 private:
  friend class ColumnFamilySet;
  ColumnFamilyData(uint32_t id, const std::string& name,
                   Version* dummy_versions, Cache* table_cache,
                   WriteBufferManager* write_buffer_manager,
                   const ColumnFamilyOptions& options,
                   const ImmutableDBOptions& db_options,
                   const EnvOptions& env_options,
                   ColumnFamilySet* column_family_set);

  uint32_t id_;
  const std::string name_;
  Version* dummy_versions_;  // Head of circular doubly-linked list of versions.
  Version* current_;         // == dummy_versions->prev_

  std::atomic<int> refs_;      // outstanding references to ColumnFamilyData
  bool dropped_;               // true if client dropped it

  const InternalKeyComparator internal_comparator_;
  std::vector<std::unique_ptr<IntTblPropCollectorFactory>>
      int_tbl_prop_collector_factories_;

  const ColumnFamilyOptions initial_cf_options_;
  const ImmutableCFOptions ioptions_;
  MutableCFOptions mutable_cf_options_;

  const bool is_delete_range_supported_;

  std::unique_ptr<TableCache> table_cache_;

  std::unique_ptr<InternalStats> internal_stats_;

  WriteBufferManager* write_buffer_manager_;

  MemTable* mem_;
  MemTableList imm_;
  SuperVersion* super_version_;

  // An ordinal representing the current SuperVersion. Updated by
  // InstallSuperVersion(), i.e. incremented every time super_version_
  // changes.
  std::atomic<uint64_t> super_version_number_;

  // Thread's local copy of SuperVersion pointer
  // This needs to be destructed before mutex_
  std::unique_ptr<ThreadLocalPtr> local_sv_;

  // pointers for a circular linked list. we use it to support iterations over
  // all column families that are alive (note: dropped column families can also
  // be alive as long as client holds a reference)
  ColumnFamilyData* next_;
  ColumnFamilyData* prev_;

  // This is the earliest log file number that contains data from this
  // Column Family. All earlier log files must be ignored and not
  // recovered from
  uint64_t log_number_;

  // An object that keeps all the compaction stats
  // and picks the next compaction
  std::unique_ptr<CompactionPicker> compaction_picker_;

  ColumnFamilySet* column_family_set_;

  std::unique_ptr<WriteControllerToken> write_controller_token_;

  // If true --> this ColumnFamily is currently present in DBImpl::flush_queue_
  bool pending_flush_;

  // If true --> this ColumnFamily is currently present in
  // DBImpl::compaction_queue_
  bool pending_compaction_;

  uint64_t prev_compaction_needed_bytes_;

  // if the database was opened with 2pc enabled
  bool allow_2pc_;
};
```

ColumnFamilyData::ColumnFamilyData()构造函数：
```
ColumnFamilyData::ColumnFamilyData(
    uint32_t id, const std::string& name, Version* _dummy_versions,
    Cache* _table_cache, WriteBufferManager* write_buffer_manager,
    const ColumnFamilyOptions& cf_options, const ImmutableDBOptions& db_options,
    const EnvOptions& env_options, ColumnFamilySet* column_family_set)
    : id_(id),
      name_(name),
      dummy_versions_(_dummy_versions),
      current_(nullptr),
      refs_(0),
      dropped_(false),
      internal_comparator_(cf_options.comparator),
      initial_cf_options_(SanitizeOptions(db_options, cf_options)),
      ioptions_(db_options, initial_cf_options_),
      mutable_cf_options_(initial_cf_options_),
      is_delete_range_supported_(strcmp(cf_options.table_factory->Name(),
                                        BlockBasedTableFactory().Name()) == 0),
      write_buffer_manager_(write_buffer_manager),
      mem_(nullptr),
      imm_(ioptions_.min_write_buffer_number_to_merge,
           ioptions_.max_write_buffer_number_to_maintain),
      super_version_(nullptr),
      super_version_number_(0),
      local_sv_(new ThreadLocalPtr(&SuperVersionUnrefHandle)),
      next_(nullptr),
      prev_(nullptr),
      log_number_(0),
      column_family_set_(column_family_set),
      pending_flush_(false),
      pending_compaction_(false),
      prev_compaction_needed_bytes_(0),
      allow_2pc_(db_options.allow_2pc) {
  /*增加引用计数*/
  Ref();

  // Convert user defined table properties collector factories to internal ones.
  GetIntTblPropCollectorFactory(ioptions_, &int_tbl_prop_collector_factories_);

  // if _dummy_versions is nullptr, then this is a dummy column family.
  if (_dummy_versions != nullptr) {
    /*初始化internal_stats*/
    internal_stats_.reset(
        new InternalStats(ioptions_.num_levels, db_options.env, this));
    /*初始化table_cache*/
    table_cache_.reset(new TableCache(ioptions_, env_options, _table_cache));
    /*根据设置的Compaction style来初始化compaction_picker_*/
    if (ioptions_.compaction_style == kCompactionStyleLevel) {
      compaction_picker_.reset(
          new LevelCompactionPicker(ioptions_, &internal_comparator_));
#ifndef ROCKSDB_LITE
    } else if (ioptions_.compaction_style == kCompactionStyleUniversal) {
      compaction_picker_.reset(
          new UniversalCompactionPicker(ioptions_, &internal_comparator_));
    } else if (ioptions_.compaction_style == kCompactionStyleFIFO) {
      compaction_picker_.reset(
          new FIFOCompactionPicker(ioptions_, &internal_comparator_));
    } else if (ioptions_.compaction_style == kCompactionStyleNone) {
      compaction_picker_.reset(new NullCompactionPicker(
          ioptions_, &internal_comparator_));
      ROCKS_LOG_WARN(ioptions_.info_log,
                     "Column family %s does not use any background compaction. "
                     "Compactions can only be done via CompactFiles\n",
                     GetName().c_str());
#endif  // !ROCKSDB_LITE
    } else {
      ROCKS_LOG_ERROR(ioptions_.info_log,
                      "Unable to recognize the specified compaction style %d. "
                      "Column family %s will use kCompactionStyleLevel.\n",
                      ioptions_.compaction_style, GetName().c_str());
      compaction_picker_.reset(
          new LevelCompactionPicker(ioptions_, &internal_comparator_));
    }

    if (column_family_set_->NumberOfColumnFamilies() < 10) {
      ROCKS_LOG_INFO(ioptions_.info_log,
                     "--------------- Options for column family [%s]:\n",
                     name.c_str());
      initial_cf_options_.Dump(ioptions_.info_log);
    } else {
      ROCKS_LOG_INFO(ioptions_.info_log, "\t(skipping printing options)\n");
    }
  }

  RecalculateWriteStallConditions(mutable_cf_options_);
}
```

ColumnFamilyData::RecalculateWriteStallConditions()用于计算Write stall条件：某些条件下需要暂停写，某些条件下则需要在每次写之后休眠一段时间，某些条件下则需要增加Compaction threads的数目。关于写速率控制，参考WriteController。
```
void ColumnFamilyData::RecalculateWriteStallConditions(
      const MutableCFOptions& mutable_cf_options) {
  /*current_表示当前最新的Version*/
  if (current_ != nullptr) {
    auto* vstorage = current_->storage_info();
    auto write_controller = column_family_set_->write_controller_;
    /*预估的为了将所有level的大小降到阈值以下所需要合并的大小*/
    uint64_t compaction_needed_bytes =
        vstorage->estimated_compaction_needed_bytes();

    bool was_stopped = write_controller->IsStopped();
    bool needed_delay = write_controller->NeedsDelay();

    if (imm()->NumNotFlushed() >= mutable_cf_options.max_write_buffer_number) {
      /* 待Flush的ImmutableMemtable数目超过mutable_cf_options.max_write_buffer_number，
       * 则需要暂停写
       */
      write_controller_token_ = write_controller->GetStopToken();
      internal_stats_->AddCFStats(InternalStats::MEMTABLE_COMPACTION, 1);
    } else if (!mutable_cf_options.disable_auto_compactions &&
               vstorage->l0_delay_trigger_count() >=
                   mutable_cf_options.level0_stop_writes_trigger) {
      /*在禁止自动Compaction的情况下，Level0中文件数目超过阈值，则需要暂停写*/
      write_controller_token_ = write_controller->GetStopToken();
      internal_stats_->AddCFStats(InternalStats::LEVEL0_NUM_FILES_TOTAL, 1);
      if (compaction_picker_->IsLevel0CompactionInProgress()) {
        internal_stats_->AddCFStats(
            InternalStats::LEVEL0_NUM_FILES_WITH_COMPACTION, 1);
      }
    } else if (!mutable_cf_options.disable_auto_compactions &&
               mutable_cf_options.hard_pending_compaction_bytes_limit > 0 &&
               compaction_needed_bytes >=
                   mutable_cf_options.hard_pending_compaction_bytes_limit) {
      /*在禁止自动Compaction的情况下，预估的待Compaction的数据大小超过阈值，则需要暂停写*/            
      write_controller_token_ = write_controller->GetStopToken();
      internal_stats_->AddCFStats(
          InternalStats::HARD_PENDING_COMPACTION_BYTES_LIMIT, 1);
    } else if (mutable_cf_options.max_write_buffer_number > 3 &&
               imm()->NumNotFlushed() >=
                   mutable_cf_options.max_write_buffer_number - 1) {
      write_controller_token_ =
          SetupDelay(write_controller, compaction_needed_bytes,
                     prev_compaction_needed_bytes_, was_stopped,
                     mutable_cf_options.disable_auto_compactions);
      internal_stats_->AddCFStats(InternalStats::MEMTABLE_SLOWDOWN, 1);
      ROCKS_LOG_WARN(
          ioptions_.info_log,
          "[%s] Stalling writes because we have %d immutable memtables "
          "(waiting for flush), max_write_buffer_number is set to %d "
          "rate %" PRIu64,
          name_.c_str(), imm()->NumNotFlushed(),
          mutable_cf_options.max_write_buffer_number,
          write_controller->delayed_write_rate());
    } else if (!mutable_cf_options.disable_auto_compactions &&
               mutable_cf_options.level0_slowdown_writes_trigger >= 0 &&
               vstorage->l0_delay_trigger_count() >=
                   mutable_cf_options.level0_slowdown_writes_trigger) {
      // L0 is the last two files from stopping.
      bool near_stop = vstorage->l0_delay_trigger_count() >=
                       mutable_cf_options.level0_stop_writes_trigger - 2;
      write_controller_token_ =
          SetupDelay(write_controller, compaction_needed_bytes,
                     prev_compaction_needed_bytes_, was_stopped || near_stop,
                     mutable_cf_options.disable_auto_compactions);
      internal_stats_->AddCFStats(InternalStats::LEVEL0_SLOWDOWN_TOTAL, 1);
      if (compaction_picker_->IsLevel0CompactionInProgress()) {
        internal_stats_->AddCFStats(
            InternalStats::LEVEL0_SLOWDOWN_WITH_COMPACTION, 1);
      }
      ROCKS_LOG_WARN(ioptions_.info_log,
                     "[%s] Stalling writes because we have %d level-0 files "
                     "rate %" PRIu64,
                     name_.c_str(), vstorage->l0_delay_trigger_count(),
                     write_controller->delayed_write_rate());
    } else if (!mutable_cf_options.disable_auto_compactions &&
               mutable_cf_options.soft_pending_compaction_bytes_limit > 0 &&
               vstorage->estimated_compaction_needed_bytes() >=
                   mutable_cf_options.soft_pending_compaction_bytes_limit) {
      // If the distance to hard limit is less than 1/4 of the gap between soft
      // and
      // hard bytes limit, we think it is near stop and speed up the slowdown.
      bool near_stop =
          mutable_cf_options.hard_pending_compaction_bytes_limit > 0 &&
          (compaction_needed_bytes -
           mutable_cf_options.soft_pending_compaction_bytes_limit) >
              3 * (mutable_cf_options.hard_pending_compaction_bytes_limit -
                   mutable_cf_options.soft_pending_compaction_bytes_limit) /
                  4;

      write_controller_token_ =
          SetupDelay(write_controller, compaction_needed_bytes,
                     prev_compaction_needed_bytes_, was_stopped || near_stop,
                     mutable_cf_options.disable_auto_compactions);
      internal_stats_->AddCFStats(
          InternalStats::SOFT_PENDING_COMPACTION_BYTES_LIMIT, 1);
      ROCKS_LOG_WARN(
          ioptions_.info_log,
          "[%s] Stalling writes because of estimated pending compaction "
          "bytes %" PRIu64 " rate %" PRIu64,
          name_.c_str(), vstorage->estimated_compaction_needed_bytes(),
          write_controller->delayed_write_rate());
    } else {
      if (vstorage->l0_delay_trigger_count() >=
          GetL0ThresholdSpeedupCompaction(
              mutable_cf_options.level0_file_num_compaction_trigger,
              mutable_cf_options.level0_slowdown_writes_trigger)) {
        write_controller_token_ =
            write_controller->GetCompactionPressureToken();
        ROCKS_LOG_WARN(
            ioptions_.info_log,
            "[%s] Increasing compaction threads because we have %d level-0 "
            "files ",
            name_.c_str(), vstorage->l0_delay_trigger_count());
      } else if (vstorage->estimated_compaction_needed_bytes() >=
                 mutable_cf_options.soft_pending_compaction_bytes_limit / 4) {
        // Increase compaction threads if bytes needed for compaction exceeds
        // 1/4 of threshold for slowing down.
        // If soft pending compaction byte limit is not set, always speed up
        // compaction.
        write_controller_token_ =
            write_controller->GetCompactionPressureToken();
        if (mutable_cf_options.soft_pending_compaction_bytes_limit > 0) {
          ROCKS_LOG_WARN(
              ioptions_.info_log,
              "[%s] Increasing compaction threads because of estimated pending "
              "compaction "
              "bytes %" PRIu64,
              name_.c_str(), vstorage->estimated_compaction_needed_bytes());
        }
      } else {
        write_controller_token_.reset();
      }
      // If the DB recovers from delay conditions, we reward with reducing
      // double the slowdown ratio. This is to balance the long term slowdown
      // increase signal.
      if (needed_delay) {
        uint64_t write_rate = write_controller->delayed_write_rate();
        write_controller->set_delayed_write_rate(static_cast<uint64_t>(
            static_cast<double>(write_rate) * kDelayRecoverSlowdownRatio));
      }
    }
    prev_compaction_needed_bytes_ = compaction_needed_bytes;
  }
}
```


ColumnFamilyData::SetDropped()设置该ColumnFamily被drop，并且从它所属的ColumnFamilySet中移除之。
```
void ColumnFamilyData::SetDropped() {
  // can't drop default CF
  assert(id_ != 0);
  dropped_ = true;
  write_controller_token_.reset();

  // remove from column_family_set
  column_family_set_->RemoveColumnFamily(this);
}
```

ColumnFamilyData::OldestLogToKeep()用于计算为保持该ColumnFamily的持久性所需要保留的最旧的Log。
```
uint64_t ColumnFamilyData::OldestLogToKeep() {
  auto current_log = GetLogNumber();

  if (allow_2pc_) {
    /*分别在Immutable MemTable和Mutable MemTable中查找需要保留的最旧的Log*/
    auto imm_prep_log = imm()->GetMinLogContainingPrepSection();
    auto mem_prep_log = mem()->GetMinLogContainingPrepSection();

    if (imm_prep_log > 0 && imm_prep_log < current_log) {
      current_log = imm_prep_log;
    }

    if (mem_prep_log > 0 && mem_prep_log < current_log) {
      current_log = mem_prep_log;
    }
  }

  return current_log;
}
```

ColumnFamilyData::ConstructNewMemtable()用于创建新的MemTable。
```
MemTable* ColumnFamilyData::ConstructNewMemtable(
    const MutableCFOptions& mutable_cf_options, SequenceNumber earliest_seq) {
  return new MemTable(internal_comparator_, ioptions_, mutable_cf_options,
                      write_buffer_manager_, earliest_seq);
}
```

ColumnFamilyData::CreateNewMemtable()用于为当前ColumnFamilyData设置新的Mutable MemTable。
```
void ColumnFamilyData::CreateNewMemtable(
    const MutableCFOptions& mutable_cf_options, SequenceNumber earliest_seq) {
  if (mem_ != nullptr) {
    /* 如果Mutable MemTable已经存在，则减少其引用计数，如果引用计数降为0，则删除之。
     * MemTable::Unref()减少引用计数，如果引用计数降为0，则返回MemTable自身的指针，
     * 否则返回nullptr。
     */
    delete mem_->Unref();
  }
  SetMemtable(ConstructNewMemtable(mutable_cf_options, earliest_seq));
  mem_->Ref();
}
```

ColumnFamilyData中Compaction相关的操作都是借助于CompactionPicker来实现。
```
bool ColumnFamilyData::NeedsCompaction() const {
  return compaction_picker_->NeedsCompaction(current_->storage_info());
}

Compaction* ColumnFamilyData::PickCompaction(
    const MutableCFOptions& mutable_options, LogBuffer* log_buffer) {
  auto* result = compaction_picker_->PickCompaction(
      GetName(), mutable_options, current_->storage_info(), log_buffer);
  if (result != nullptr) {
    result->SetInputVersion(current_);
  }
  return result;
}

bool ColumnFamilyData::RangeOverlapWithCompaction(
    const Slice& smallest_user_key, const Slice& largest_user_key,
    int level) const {
  return compaction_picker_->RangeOverlapWithCompaction(
      smallest_user_key, largest_user_key, level);
}

Compaction* ColumnFamilyData::CompactRange(
    const MutableCFOptions& mutable_cf_options, int input_level,
    int output_level, uint32_t output_path_id, const InternalKey* begin,
    const InternalKey* end, InternalKey** compaction_end, bool* conflict) {
  auto* result = compaction_picker_->CompactRange(
      GetName(), mutable_cf_options, current_->storage_info(), input_level,
      output_level, output_path_id, begin, end, compaction_end, conflict);
  if (result != nullptr) {
    result->SetInputVersion(current_);
  }
  return result;
}
```

ColumnFamilyData相关的方法将在用到的时候在讲解。。。。


# ColumnFamilySet

```
class ColumnFamilySet {
 public:
  // ColumnFamilySet supports iteration
  class iterator {
   public:
    explicit iterator(ColumnFamilyData* cfd)
        : current_(cfd) {}
    iterator& operator++() {
      // dropped column families might still be included in this iteration
      // (we're only removing them when client drops the last reference to the
      // column family).
      // dummy is never dead, so this will never be infinite
      do {
        current_ = current_->next_;
      } while (current_->refs_.load(std::memory_order_relaxed) == 0);
      return *this;
    }
    bool operator!=(const iterator& other) {
      return this->current_ != other.current_;
    }
    ColumnFamilyData* operator*() { return current_; }

   private:
    ColumnFamilyData* current_;
  };

  ColumnFamilySet(const std::string& dbname,
                  const ImmutableDBOptions* db_options,
                  const EnvOptions& env_options, Cache* table_cache,
                  WriteBufferManager* write_buffer_manager,
                  WriteController* write_controller);
  ~ColumnFamilySet();

  ColumnFamilyData* GetDefault() const;
  // GetColumnFamily() calls return nullptr if column family is not found
  ColumnFamilyData* GetColumnFamily(uint32_t id) const;
  ColumnFamilyData* GetColumnFamily(const std::string& name) const;
  // this call will return the next available column family ID. it guarantees
  // that there is no column family with id greater than or equal to the
  // returned value in the current running instance or anytime in RocksDB
  // instance history.
  uint32_t GetNextColumnFamilyID();
  uint32_t GetMaxColumnFamily();
  void UpdateMaxColumnFamily(uint32_t new_max_column_family);
  size_t NumberOfColumnFamilies() const;

  ColumnFamilyData* CreateColumnFamily(const std::string& name, uint32_t id,
                                       Version* dummy_version,
                                       const ColumnFamilyOptions& options);

  iterator begin() { return iterator(dummy_cfd_->next_); }
  iterator end() { return iterator(dummy_cfd_); }

  // REQUIRES: DB mutex held
  // Don't call while iterating over ColumnFamilySet
  void FreeDeadColumnFamilies();

  Cache* get_table_cache() { return table_cache_; }

 private:
  friend class ColumnFamilyData;
  // helper function that gets called from cfd destructor
  // REQUIRES: DB mutex held
  void RemoveColumnFamily(ColumnFamilyData* cfd);

  // column_families_ and column_family_data_ need to be protected:
  // * when mutating both conditions have to be satisfied:
  // 1. DB mutex locked
  // 2. thread currently in single-threaded write thread
  // * when reading, at least one condition needs to be satisfied:
  // 1. DB mutex locked
  // 2. accessed from a single-threaded write thread
  std::unordered_map<std::string, uint32_t> column_families_;
  std::unordered_map<uint32_t, ColumnFamilyData*> column_family_data_;

  uint32_t max_column_family_;
  /*Head of circular doubly-linked list of ColumnFamilyData*/
  ColumnFamilyData* dummy_cfd_;
  // We don't hold the refcount here, since default column family always exists
  // We are also not responsible for cleaning up default_cfd_cache_. This is
  // just a cache that makes common case (accessing default column family)
  // faster
  ColumnFamilyData* default_cfd_cache_;

  const std::string db_name_;
  const ImmutableDBOptions* const db_options_;
  const EnvOptions env_options_;
  Cache* table_cache_;
  WriteBufferManager* write_buffer_manager_;
  WriteController* write_controller_;
};
```

ColumnFamilySet::CreateColumnFamily()创建具有指定ID和name的ColumnFamilyData并添加到ColumnFamilySet中：
```
ColumnFamilyData* ColumnFamilySet::CreateColumnFamily(
    const std::string& name, uint32_t id, Version* dummy_versions,
    const ColumnFamilyOptions& options) {
  assert(column_families_.find(name) == column_families_.end());
  /*为指定ID和name的ColumnFamily创建ColumnFamilyData*/
  ColumnFamilyData* new_cfd = new ColumnFamilyData(
      id, name, dummy_versions, table_cache_, write_buffer_manager_, options,
      *db_options_, env_options_, this);
  /*将新创建的ColumnFamily信息插入到column_families中*/
  column_families_.insert({name, id});
  /*将新创建的ColumnFamilyData插入到column_family_data_中*/
  column_family_data_.insert({id, new_cfd});
  /*更新当前最大ColumnFamily ID*/
  max_column_family_ = std::max(max_column_family_, id);
  // add to linked list
  /*将新创建的ColumnFamilyData插入到以dummy_cfd为头的双向链表中*/
  new_cfd->next_ = dummy_cfd_;
  auto prev = dummy_cfd_->prev_;
  new_cfd->prev_ = prev;
  prev->next_ = new_cfd;
  dummy_cfd_->prev_ = new_cfd;
  /*如果新创建的ColumnFamily的ID是0，则是默认ColumnFamily*/
  if (id == 0) {
    default_cfd_cache_ = new_cfd;
  }
  return new_cfd;
}
                                       
```


