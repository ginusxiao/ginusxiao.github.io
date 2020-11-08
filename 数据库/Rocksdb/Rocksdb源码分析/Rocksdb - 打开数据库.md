# 提纲
[toc]

# 接口
Rocksdb为打开数据库提供了下列接口，每个接口都有详细介绍，在此不再赘述，直接看注释。
```
  // Open the database with the specified "name".
  // Stores a pointer to a heap-allocated database in *dbptr and returns
  // OK on success.
  // Stores nullptr in *dbptr and returns a non-OK status on error.
  // Caller should delete *dbptr when it is no longer needed.
  static Status Open(const Options& options,
                     const std::string& name,
                     DB** dbptr);

  // Open the database for read only. All DB interfaces
  // that modify data, like put/delete, will return error.
  // If the db is opened in read only mode, then no compactions
  // will happen.
  //
  // Not supported in ROCKSDB_LITE, in which case the function will
  // return Status::NotSupported.
  static Status OpenForReadOnly(const Options& options,
      const std::string& name, DB** dbptr,
      bool error_if_log_file_exist = false);

  // Open the database for read only with column families. When opening DB with
  // read only, you can specify only a subset of column families in the
  // database that should be opened. However, you always need to specify default
  // column family. The default column family name is 'default' and it's stored
  // in rocksdb::kDefaultColumnFamilyName
  //
  // Not supported in ROCKSDB_LITE, in which case the function will
  // return Status::NotSupported.
  static Status OpenForReadOnly(
      const DBOptions& db_options, const std::string& name,
      const std::vector<ColumnFamilyDescriptor>& column_families,
      std::vector<ColumnFamilyHandle*>* handles, DB** dbptr,
      bool error_if_log_file_exist = false);

  // Open DB with column families.
  // db_options specify database specific options
  // column_families is the vector of all column families in the database,
  // containing column family name and options. You need to open ALL column
  // families in the database. To get the list of column families, you can use
  // ListColumnFamilies(). Also, you can open only a subset of column families
  // for read-only access.
  // The default column family name is 'default' and it's stored
  // in rocksdb::kDefaultColumnFamilyName.
  // If everything is OK, handles will on return be the same size
  // as column_families --- handles[i] will be a handle that you
  // will use to operate on column family column_family[i].
  // Before delete DB, you have to close All column families by calling
  // DestroyColumnFamilyHandle() with all the handles.
  static Status Open(const DBOptions& db_options, const std::string& name,
                     const std::vector<ColumnFamilyDescriptor>& column_families,
                     std::vector<ColumnFamilyHandle*>* handles, DB** dbptr);
```
实际上，上面四个接口之间存在如下调用关系：
```
static Status Open(const Options& options, const std::string& name, DB** dbptr)
    - static Status Open(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr)

static Status OpenForReadOnly(const Options& options, const std::string& name, DB** dbptr, bool error_if_log_file_exist = false)
    - static Status OpenForReadOnly(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr, bool error_if_log_file_exist = false)
```

所以在后面分析的时候，只会分析后面两个接口:

```
static Status Open(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr)

static Status OpenForReadOnly(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr, bool error_if_log_file_exist = false)
```


# 以可读写方式打开数据库
## 入口函数

**DB::Open()尚未分析完毕。。。**

```
Status DB::Open(const DBOptions& db_options, const std::string& dbname,
                const std::vector<ColumnFamilyDescriptor>& column_families,
                std::vector<ColumnFamilyHandle*>* handles, DB** dbptr) {
  /*检查选项设置，并进行必要的调整*/
  Status s = SanitizeOptionsByTable(db_options, column_families);
  if (!s.ok()) {
    return s;
  }

  /*检查选项设置之间是否存在冲突*/
  s = ValidateOptions(db_options, column_families);
  if (!s.ok()) {
    return s;
  }

  *dbptr = nullptr;
  handles->clear();

  /*以各column family中设置的最大write buffer size作为即将打开的Rocksdb实例的最大write buffer size*/
  size_t max_write_buffer_size = 0;
  for (auto cf : column_families) {
    max_write_buffer_size =
        std::max(max_write_buffer_size, cf.options.write_buffer_size);
  }

  /*创建DBImpl类型实例，DBImpl继承自DB类*/
  DBImpl* impl = new DBImpl(db_options, dbname);
  /*创建WAL目录(如果不存在的话)*/
  s = impl->env_->CreateDirIfMissing(impl->immutable_db_options_.wal_dir);
  if (s.ok()) {
    /* 创建指定的用于存放SST文件的一系列目录（如果不存在的话），关于db_path的说明
     * 请参考"include/rocksdb/options.h"中关于Options的定义
     */
    for (auto db_path : impl->immutable_db_options_.db_paths) {
      s = impl->env_->CreateDirIfMissing(db_path.path);
      if (!s.ok()) {
        break;
      }
    }
  }

  if (!s.ok()) {
    delete impl;
    return s;
  }

  /*如果需要，则在WAL目录下为WAL创建archive的目录*/
  s = impl->CreateArchivalDirectory();
  if (!s.ok()) {
    delete impl;
    return s;
  }
  impl->mutex_.Lock();
  // Handles create_if_missing, error_if_exists
  s = impl->Recover(column_families);
  if (s.ok()) {
    uint64_t new_log_number = impl->versions_->NewFileNumber();
    unique_ptr<WritableFile> lfile;
    EnvOptions soptions(db_options);
    EnvOptions opt_env_options =
        impl->immutable_db_options_.env->OptimizeForLogWrite(
            soptions, BuildDBOptions(impl->immutable_db_options_,
                                     impl->mutable_db_options_));
    s = NewWritableFile(
        impl->immutable_db_options_.env,
        LogFileName(impl->immutable_db_options_.wal_dir, new_log_number),
        &lfile, opt_env_options);
    if (s.ok()) {
      lfile->SetPreallocationBlockSize(
          impl->GetWalPreallocateBlockSize(max_write_buffer_size));
      impl->logfile_number_ = new_log_number;
      unique_ptr<WritableFileWriter> file_writer(
          new WritableFileWriter(std::move(lfile), opt_env_options));
      impl->logs_.emplace_back(
          new_log_number,
          new log::Writer(
              std::move(file_writer), new_log_number,
              impl->immutable_db_options_.recycle_log_file_num > 0));

      // set column family handles
      for (auto cf : column_families) {
        auto cfd =
            impl->versions_->GetColumnFamilySet()->GetColumnFamily(cf.name);
        if (cfd != nullptr) {
          handles->push_back(
              new ColumnFamilyHandleImpl(cfd, impl, &impl->mutex_));
          impl->NewThreadStatusCfInfo(cfd);
        } else {
          if (db_options.create_missing_column_families) {
            // missing column family, create it
            ColumnFamilyHandle* handle;
            impl->mutex_.Unlock();
            s = impl->CreateColumnFamily(cf.options, cf.name, &handle);
            impl->mutex_.Lock();
            if (s.ok()) {
              handles->push_back(handle);
            } else {
              break;
            }
          } else {
            s = Status::InvalidArgument("Column family not found: ", cf.name);
            break;
          }
        }
      }
    }
    if (s.ok()) {
      for (auto cfd : *impl->versions_->GetColumnFamilySet()) {
        delete impl->InstallSuperVersionAndScheduleWork(
            cfd, nullptr, *cfd->GetLatestMutableCFOptions());
      }
      impl->alive_log_files_.push_back(
          DBImpl::LogFileNumberSize(impl->logfile_number_));
      impl->DeleteObsoleteFiles();
      s = impl->directories_.GetDbDir()->Fsync();
    }
  }

  if (s.ok()) {
    for (auto cfd : *impl->versions_->GetColumnFamilySet()) {
      if (cfd->ioptions()->compaction_style == kCompactionStyleFIFO) {
        auto* vstorage = cfd->current()->storage_info();
        for (int i = 1; i < vstorage->num_levels(); ++i) {
          int num_files = vstorage->NumLevelFiles(i);
          if (num_files > 0) {
            s = Status::InvalidArgument(
                "Not all files are at level 0. Cannot "
                "open with FIFO compaction style.");
            break;
          }
        }
      }
      if (!cfd->mem()->IsSnapshotSupported()) {
        impl->is_snapshot_supported_ = false;
      }
      if (cfd->ioptions()->merge_operator != nullptr &&
          !cfd->mem()->IsMergeOperatorSupported()) {
        s = Status::InvalidArgument(
            "The memtable of column family %s does not support merge operator "
            "its options.merge_operator is non-null", cfd->GetName().c_str());
      }
      if (!s.ok()) {
        break;
      }
    }
  }
  TEST_SYNC_POINT("DBImpl::Open:Opened");
  Status persist_options_status;
  if (s.ok()) {
    // Persist RocksDB Options before scheduling the compaction.
    // The WriteOptionsFile() will release and lock the mutex internally.
    persist_options_status = impl->WriteOptionsFile(
        false /*need_mutex_lock*/, false /*need_enter_write_thread*/);

    *dbptr = impl;
    impl->opened_successfully_ = true;
    impl->MaybeScheduleFlushOrCompaction();
  }
  impl->mutex_.Unlock();

#ifndef ROCKSDB_LITE
  auto sfm = static_cast<SstFileManagerImpl*>(
      impl->immutable_db_options_.sst_file_manager.get());
  if (s.ok() && sfm) {
    // Notify SstFileManager about all sst files that already exist in
    // db_paths[0] when the DB is opened.
    auto& db_path = impl->immutable_db_options_.db_paths[0];
    std::vector<std::string> existing_files;
    impl->immutable_db_options_.env->GetChildren(db_path.path, &existing_files);
    for (auto& file_name : existing_files) {
      uint64_t file_number;
      FileType file_type;
      std::string file_path = db_path.path + "/" + file_name;
      if (ParseFileName(file_name, &file_number, &file_type) &&
          file_type == kTableFile) {
        sfm->OnAddFile(file_path);
      }
    }
  }
#endif  // !ROCKSDB_LITE

  if (s.ok()) {
    ROCKS_LOG_INFO(impl->immutable_db_options_.info_log, "DB pointer %p", impl);
    LogFlush(impl->immutable_db_options_.info_log);
    if (!persist_options_status.ok()) {
      s = Status::IOError(
          "DB::Open() failed --- Unable to persist Options file",
          persist_options_status.ToString());
    }
  }
  if (!s.ok()) {
    for (auto* h : *handles) {
      delete h;
    }
    handles->clear();
    delete impl;
    *dbptr = nullptr;
  }
  return s;
}
```

## 创建DBImpl类型实例

```
DBImpl::DBImpl(const DBOptions& options, const std::string& dbname) :
      /*直接引用options.env，在DBOptions中默认使用的是Env::Default()，在Linux下Env::Default()采用的是PosixEnv*/
      env_(options.env),
      dbname_(dbname),
      /*通过传递的options设置initial_db_options_*/
      initial_db_options_(SanitizeOptions(dbname, options)),
      /*通过initial_db_options_设置不可改变的options*/
      immutable_db_options_(initial_db_options_),
      /*通过initial_db_options_设置可动态改变的options*/
      mutable_db_options_(initial_db_options_),
      /*统计信息，直接引用immutable_db_options_.statistics*/
      stats_(immutable_db_options_.statistics.get()),
      db_lock_(nullptr),
      mutex_(stats_, env_, DB_MUTEX_WAIT_MICROS,
             immutable_db_options_.use_adaptive_mutex),
      shutting_down_(false), 
      bg_cv_(&mutex_), // background purge, flush or compaction相关的条件变量
      logfile_number_(0),
      log_dir_synced_(false),
      log_empty_(true),
      default_cf_handle_(nullptr), //默认ColumnFamilyHandle
      log_sync_cv_(&mutex_), //日志同步相关的条件变量
      total_log_size_(0),
      max_total_in_memory_state_(0),
      is_snapshot_supported_(true),
      write_buffer_manager_(immutable_db_options_.write_buffer_manager.get()), // MemTable内存管理器
      write_thread_(immutable_db_options_), // 写线程
      write_controller_(mutable_db_options_.delayed_write_rate), // 写控制器（控制写暂停）
      last_batch_group_size_(0),
      unscheduled_flushes_(0),
      unscheduled_compactions_(0),
      bg_compaction_scheduled_(0),
      num_running_compactions_(0),
      bg_flush_scheduled_(0),
      num_running_flushes_(0),
      bg_purge_scheduled_(0),
      disable_delete_obsolete_files_(0),
      delete_obsolete_files_last_run_(env_->NowMicros()),
      last_stats_dump_time_microsec_(0),
      next_job_id_(1),
      has_unpersisted_data_(false),
      unable_to_flush_oldest_log_(false),
      /*通过immutable_db_options_和mutable_db_options_构建DBOptions，最后构建EnvOptions*/
      env_options_(BuildDBOptions(immutable_db_options_, mutable_db_options_)),
      num_running_ingest_file_(0),
#ifndef ROCKSDB_LITE
      wal_manager_(immutable_db_options_, env_options_), 
#endif  // ROCKSDB_LITE
      event_logger_(immutable_db_options_.info_log.get()),
      bg_work_paused_(0),
      bg_compaction_paused_(0),
      refitting_level_(false),
      opened_successfully_(false) {
  /* 从dbname中解析出数据库目录的绝对路径，存放于db_absolute_path_中，
   * 调用PosixEnv::GetAbsolutePath
   */
  env_->GetAbsolutePath(dbname, &db_absolute_path_);

  // Reserve ten files or so for other uses and give the rest to TableCache.
  // Give a large number for setting of "infinite" open files.
  const int table_cache_size = (mutable_db_options_.max_open_files == -1)
                                   ? TableCache::kInfiniteCapacity
                                   : mutable_db_options_.max_open_files - 10;
  /*构建基于LRU的BlockCache*/                                   
  table_cache_ = NewLRUCache(table_cache_size,
                             immutable_db_options_.table_cache_numshardbits);

  versions_.reset(new VersionSet(dbname_, &immutable_db_options_, env_options_,
                                 table_cache_.get(), write_buffer_manager_,
                                 &write_controller_));
  column_family_memtables_.reset(
      new ColumnFamilyMemTablesImpl(versions_->GetColumnFamilySet()));

  DumpRocksDBBuildVersion(immutable_db_options_.info_log.get());
  DumpDBFileSummary(immutable_db_options_, dbname_);
  immutable_db_options_.Dump(immutable_db_options_.info_log.get());
  mutable_db_options_.Dump(immutable_db_options_.info_log.get());
  DumpSupportInfo(immutable_db_options_.info_log.get());
}
```

## 恢复数据库
恢复数据库包含以下步骤：
1. 如果不是只读模式，则检查dbname_目录下的CURRENT文件是否存在
    
    如果该文件不存在但设置了create_if_missing，则创建之

    如果该文件不存在且未设置create_if_missing，则报错
    
    如果该文件存在但设置了error_if_exists，则报错
    
    如果检查过程中发生了IO错误，则报错

2. 如果不是只读模式，则检查db_name_这个目录下的IDENTITY文件是否存在，如果不存在则创建之
3. 

**DBImpl::Recover()逻辑尚未分析完毕。。。**

```
Status DBImpl::Recover(const std::vector<ColumnFamilyDescriptor>& column_families,
                 bool read_only = false, bool error_if_log_file_exist = false,
                 bool error_if_data_exists_in_logs = false) {
  mutex_.AssertHeld();

  bool is_new_db = false;
  assert(db_lock_ == nullptr);
  if (!read_only) {
    /*directories_用于管理除主目录以外的所有目录（包括db directory，data directory，wal directory）*/
    Status s = directories_.SetDirectories(env_, dbname_,
                                           immutable_db_options_.wal_dir,
                                           immutable_db_options_.db_paths);
    if (!s.ok()) {
      return s;
    }

    /*获取锁db_lock_并锁住该数据库，以防其它进程访问之*/
    s = env_->LockFile(LockFileName(dbname_), &db_lock_);
    if (!s.ok()) {
      return s;
    }

    /*检查db_name_这个目录下的CURRENT文件是否存在（以此作为判断数据库是否存在的依据?）*/
    s = env_->FileExists(CurrentFileName(dbname_));
    if (s.IsNotFound()) {
      if (immutable_db_options_.create_if_missing) {
        /*数据库不存在但设置了create_if_missing，则创建之*/
        s = NewDB();
        is_new_db = true;
        if (!s.ok()) {
          return s;
        }
      } else {
        /*数据库不存在且未设置create_if_missing，则报错*/
        return Status::InvalidArgument(
            dbname_, "does not exist (create_if_missing is false)");
      }
    } else if (s.ok()) {
      if (immutable_db_options_.error_if_exists) {
        /*数据库存在但是设置了error_if_exists，则报错*/
        return Status::InvalidArgument(
            dbname_, "exists (error_if_exists is true)");
      }
    } else {
      // Unexpected error reading file
      /*读取文件过程中发生了IO错误*/
      assert(s.IsIOError());
      return s;
    }
    
    // Check for the IDENTITY file and create it if not there
    /*检查db_name_这个目录下的IDENTITY文件是否存在，如果不存在则创建之*/
    s = env_->FileExists(IdentityFileName(dbname_));
    if (s.IsNotFound()) {
      /*不存在，则创建之*/
      s = SetIdentityFile(env_, dbname_);
      if (!s.ok()) {
        return s;
      }
    } else if (!s.ok()) {
      /*IO错误*/
      assert(s.IsIOError());
      return s;
    }
  }

  /*恢复版本信息*/
  Status s = versions_->Recover(column_families, read_only);
  if (immutable_db_options_.paranoid_checks && s.ok()) {
    s = CheckConsistency();
  }
  
  if (s.ok()) {
    SequenceNumber next_sequence(kMaxSequenceNumber);
    default_cf_handle_ = new ColumnFamilyHandleImpl(
        versions_->GetColumnFamilySet()->GetDefault(), this, &mutex_);
    default_cf_internal_stats_ = default_cf_handle_->cfd()->internal_stats();
    single_column_family_mode_ =
        versions_->GetColumnFamilySet()->NumberOfColumnFamilies() == 1;

    // Recover from all newer log files than the ones named in the
    // descriptor (new log files may have been added by the previous
    // incarnation without registering them in the descriptor).
    //
    // Note that prev_log_number() is no longer used, but we pay
    // attention to it in case we are recovering a database
    // produced by an older version of rocksdb.
    std::vector<std::string> filenames;
    s = env_->GetChildren(immutable_db_options_.wal_dir, &filenames);
    if (!s.ok()) {
      return s;
    }

    std::vector<uint64_t> logs;
    for (size_t i = 0; i < filenames.size(); i++) {
      uint64_t number;
      FileType type;
      if (ParseFileName(filenames[i], &number, &type) && type == kLogFile) {
        if (is_new_db) {
          return Status::Corruption(
              "While creating a new Db, wal_dir contains "
              "existing log file: ",
              filenames[i]);
        } else {
          logs.push_back(number);
        }
      }
    }

    if (logs.size() > 0) {
      if (error_if_log_file_exist) {
        return Status::Corruption(
            "The db was opened in readonly mode with error_if_log_file_exist"
            "flag but a log file already exists");
      } else if (error_if_data_exists_in_logs) {
        for (auto& log : logs) {
          std::string fname = LogFileName(immutable_db_options_.wal_dir, log);
          uint64_t bytes;
          s = env_->GetFileSize(fname, &bytes);
          if (s.ok()) {
            if (bytes > 0) {
              return Status::Corruption(
                  "error_if_data_exists_in_logs is set but there are data "
                  " in log files.");
            }
          }
        }
      }
    }

    if (!logs.empty()) {
      // Recover in the order in which the logs were generated
      std::sort(logs.begin(), logs.end());
      s = RecoverLogFiles(logs, &next_sequence, read_only);
      if (!s.ok()) {
        // Clear memtables if recovery failed
        for (auto cfd : *versions_->GetColumnFamilySet()) {
          cfd->CreateNewMemtable(*cfd->GetLatestMutableCFOptions(),
                                 kMaxSequenceNumber);
        }
      }
    }
  }

  // Initial value
  max_total_in_memory_state_ = 0;
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    auto* mutable_cf_options = cfd->GetLatestMutableCFOptions();
    max_total_in_memory_state_ += mutable_cf_options->write_buffer_size *
                                  mutable_cf_options->max_write_buffer_number;
  }

  return s;
}
```



# 以只读方式打开数据库
## 入口函数


# 参考
[RocksDB引擎加载 ](http://blog.itpub.net/30088583/viewspace-2136675/)

