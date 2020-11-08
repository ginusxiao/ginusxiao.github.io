# 提纲
[toc]

# 数据结构


# 实现
## VersionSet
### 恢复
VersionSet::Recover()尚未分析完毕，后续继续。。。
```
Status VersionSet::Recover(
    const std::vector<ColumnFamilyDescriptor>& column_families,
    bool read_only) {
  /*ColumnFamilyOptions是对ColumnFamily name和ColumnFamily options的封装*/
  std::unordered_map<std::string, ColumnFamilyOptions> cf_name_to_options;
  
  /*将ColumnFamilyDescriptor数组转换为ColumnFamily name到ColumnFamily options的映射*/
  for (auto cf : column_families) {
    cf_name_to_options.insert({cf.name, cf.options});
  }
  
  // keeps track of column families in manifest that were not found in
  // column families parameters. if those column families are not dropped
  // by subsequent manifest records, Recover() will return failure status
  /*用于记录那些在manifest文件中但是不在参数column_families数组中的column families*/
  std::unordered_map<int, std::string> column_families_not_found;

  // Read "CURRENT" file, which contains a pointer to the current manifest file
  /*从dbname_目录下的CURRENT文件中读取最新的manifest文件名称manifest_filename*/
  std::string manifest_filename;
  Status s = ReadFileToString(
      env_, CurrentFileName(dbname_), &manifest_filename
  );
  if (!s.ok()) {
    return s;
  }
  
  /*最新的manifest文件名称不正确*/
  if (manifest_filename.empty() ||
      manifest_filename.back() != '\n') {
    return Status::Corruption("CURRENT file does not end with newline");
  }
  // remove the trailing '\n'
  manifest_filename.resize(manifest_filename.size() - 1);
  FileType type;
  /*从manifest文件中解析出manifest文件编号和类型*/
  bool parse_ok =
      ParseFileName(manifest_filename, &manifest_file_number_, &type);
  if (!parse_ok || type != kDescriptorFile) {
    return Status::Corruption("CURRENT file corrupted");
  }

  ROCKS_LOG_INFO(db_options_->info_log, "Recovering from manifest file: %s\n",
                 manifest_filename.c_str());

  /* 构建关于manifest_filename文件的manifest_file_reader（SequentialFileReader类型）
   * 用于顺序读取manifest文件
   */
  manifest_filename = dbname_ + "/" + manifest_filename;
  unique_ptr<SequentialFileReader> manifest_file_reader;
  {
    unique_ptr<SequentialFile> manifest_file;
    s = env_->NewSequentialFile(manifest_filename, &manifest_file,
                                env_options_);
    if (!s.ok()) {
      return s;
    }
    manifest_file_reader.reset(
        new SequentialFileReader(std::move(manifest_file)));
  }
  
  /*获取manifest_filename这一文件的大小*/
  uint64_t current_manifest_file_size;
  s = env_->GetFileSize(manifest_filename, &current_manifest_file_size);
  if (!s.ok()) {
    return s;
  }

  bool have_log_number = false;
  bool have_prev_log_number = false;
  bool have_next_file = false;
  bool have_last_sequence = false;
  uint64_t next_file = 0;
  uint64_t last_sequence = 0;
  uint64_t log_number = 0;
  uint64_t previous_log_number = 0;
  uint32_t max_column_family = 0;
  /*builders中存放ColumnFamily ID到其对应的BaseReferencedVersionBuilder的映射*/
  std::unordered_map<uint32_t, BaseReferencedVersionBuilder*> builders;

  // add default column family
  /*在cf_name_to_options中查找默认的column family，如果找不到，则报错*/
  auto default_cf_iter = cf_name_to_options.find(kDefaultColumnFamilyName);
  if (default_cf_iter == cf_name_to_options.end()) {
    return Status::InvalidArgument("Default column family not specified");
  }
  
  /*初始化关于默认Column Family的VersionEdit，并为默认的Column Family创建ColumnFamilyData*/
  VersionEdit default_cf_edit;
  default_cf_edit.AddColumnFamily(kDefaultColumnFamilyName);
  default_cf_edit.SetColumnFamily(0);
  ColumnFamilyData* default_cfd =
      CreateColumnFamily(default_cf_iter->second, &default_cf_edit);
  /*建立default ColumnFamily到其BaseReferencedVersionBuilder的映射*/
  builders.insert({0, new BaseReferencedVersionBuilder(default_cfd)});

  {
    VersionSet::LogReporter reporter;
    reporter.status = &s;
    /*log::Reader实际上还是通过SequentialFile的接口来读取*/
    log::Reader reader(NULL, std::move(manifest_file_reader), &reporter,
                       true /*checksum*/, 0 /*initial_offset*/, 0);
    Slice record;
    std::string scratch;
    while (reader.ReadRecord(&record, &scratch) && s.ok()) {
      /*从record中解析出VersionEdit*/
      VersionEdit edit;
      s = edit.DecodeFrom(record);
      if (!s.ok()) {
        break;
      }

      // Not found means that user didn't supply that column
      // family option AND we encountered column family add
      // record. Once we encounter column family drop record,
      // we will delete the column family from
      // column_families_not_found.
      bool cf_in_not_found =
          column_families_not_found.find(edit.column_family_) !=
          column_families_not_found.end();
      // in builders means that user supplied that column family
      // option AND that we encountered column family add record
      bool cf_in_builders =
          builders.find(edit.column_family_) != builders.end();

      // they can't both be true
      assert(!(cf_in_not_found && cf_in_builders));

      ColumnFamilyData* cfd = nullptr;

      if (edit.is_column_family_add_) {
        if (cf_in_builders || cf_in_not_found) {
          s = Status::Corruption(
              "Manifest adding the same column family twice");
          break;
        }
        auto cf_options = cf_name_to_options.find(edit.column_family_name_);
        if (cf_options == cf_name_to_options.end()) {
          /*ColumnFamily不在参数column_families中*/
          column_families_not_found.insert(
              {edit.column_family_, edit.column_family_name_});
        } else {
          /* 为该ColumnFamily创建ColumnFamilyData，并为之创建BaseReferencedVersionBuilder，
           * 最后添加到builders映射表中
           */
          cfd = CreateColumnFamily(cf_options->second, &edit);
          builders.insert(
              {edit.column_family_, new BaseReferencedVersionBuilder(cfd)});
        }
      } else if (edit.is_column_family_drop_) {
        if (cf_in_builders) {
          auto builder = builders.find(edit.column_family_);
          assert(builder != builders.end());
          delete builder->second;
          builders.erase(builder);
          cfd = column_family_set_->GetColumnFamily(edit.column_family_);
          if (cfd->Unref()) {
            delete cfd;
            cfd = nullptr;
          } else {
            // who else can have reference to cfd!?
            assert(false);
          }
        } else if (cf_in_not_found) {
          column_families_not_found.erase(edit.column_family_);
        } else {
          s = Status::Corruption(
              "Manifest - dropping non-existing column family");
          break;
        }
      } else if (!cf_in_not_found) {
        if (!cf_in_builders) {
          s = Status::Corruption(
              "Manifest record referencing unknown column family");
          break;
        }

        cfd = column_family_set_->GetColumnFamily(edit.column_family_);
        // this should never happen since cf_in_builders is true
        assert(cfd != nullptr);
        if (edit.max_level_ >= cfd->current()->storage_info()->num_levels()) {
          s = Status::InvalidArgument(
              "db has more levels than options.num_levels");
          break;
        }

        // if it is not column family add or column family drop,
        // then it's a file add/delete, which should be forwarded
        // to builder
        auto builder = builders.find(edit.column_family_);
        assert(builder != builders.end());
        builder->second->version_builder()->Apply(&edit);
      }

      if (cfd != nullptr) {
        if (edit.has_log_number_) {
          if (cfd->GetLogNumber() > edit.log_number_) {
            ROCKS_LOG_WARN(
                db_options_->info_log,
                "MANIFEST corruption detected, but ignored - Log numbers in "
                "records NOT monotonically increasing");
          } else {
            cfd->SetLogNumber(edit.log_number_);
            have_log_number = true;
          }
        }
        if (edit.has_comparator_ &&
            edit.comparator_ != cfd->user_comparator()->Name()) {
          s = Status::InvalidArgument(
              cfd->user_comparator()->Name(),
              "does not match existing comparator " + edit.comparator_);
          break;
        }
      }

      if (edit.has_prev_log_number_) {
        previous_log_number = edit.prev_log_number_;
        have_prev_log_number = true;
      }

      if (edit.has_next_file_number_) {
        next_file = edit.next_file_number_;
        have_next_file = true;
      }

      if (edit.has_max_column_family_) {
        max_column_family = edit.max_column_family_;
      }

      if (edit.has_last_sequence_) {
        last_sequence = edit.last_sequence_;
        have_last_sequence = true;
      }
    }
  }

  if (s.ok()) {
    if (!have_next_file) {
      s = Status::Corruption("no meta-nextfile entry in descriptor");
    } else if (!have_log_number) {
      s = Status::Corruption("no meta-lognumber entry in descriptor");
    } else if (!have_last_sequence) {
      s = Status::Corruption("no last-sequence-number entry in descriptor");
    }

    if (!have_prev_log_number) {
      previous_log_number = 0;
    }

    column_family_set_->UpdateMaxColumnFamily(max_column_family);

    MarkFileNumberUsedDuringRecovery(previous_log_number);
    MarkFileNumberUsedDuringRecovery(log_number);
  }

  // there were some column families in the MANIFEST that weren't specified
  // in the argument. This is OK in read_only mode
  if (read_only == false && !column_families_not_found.empty()) {
    std::string list_of_not_found;
    for (const auto& cf : column_families_not_found) {
      list_of_not_found += ", " + cf.second;
    }
    list_of_not_found = list_of_not_found.substr(2);
    s = Status::InvalidArgument(
        "You have to open all column families. Column families not opened: " +
        list_of_not_found);
  }

  if (s.ok()) {
    for (auto cfd : *column_family_set_) {
      if (cfd->IsDropped()) {
        continue;
      }
      auto builders_iter = builders.find(cfd->GetID());
      assert(builders_iter != builders.end());
      auto* builder = builders_iter->second->version_builder();

      if (GetColumnFamilySet()->get_table_cache()->GetCapacity() ==
          TableCache::kInfiniteCapacity) {
        // unlimited table cache. Pre-load table handle now.
        // Need to do it out of the mutex.
        builder->LoadTableHandlers(
            cfd->internal_stats(), db_options_->max_file_opening_threads,
            false /* prefetch_index_and_filter_in_cache */);
      }

      Version* v = new Version(cfd, this, current_version_number_++);
      builder->SaveTo(v->storage_info());

      // Install recovered version
      v->PrepareApply(*cfd->GetLatestMutableCFOptions(),
          !(db_options_->skip_stats_update_on_db_open));
      AppendVersion(cfd, v);
    }

    manifest_file_size_ = current_manifest_file_size;
    next_file_number_.store(next_file + 1);
    last_sequence_ = last_sequence;
    prev_log_number_ = previous_log_number;

    ROCKS_LOG_INFO(
        db_options_->info_log,
        "Recovered from manifest file:%s succeeded,"
        "manifest_file_number is %lu, next_file_number is %lu, "
        "last_sequence is %lu, log_number is %lu,"
        "prev_log_number is %lu,"
        "max_column_family is %u\n",
        manifest_filename.c_str(), (unsigned long)manifest_file_number_,
        (unsigned long)next_file_number_.load(), (unsigned long)last_sequence_,
        (unsigned long)log_number, (unsigned long)prev_log_number_,
        column_family_set_->GetMaxColumnFamily());

    for (auto cfd : *column_family_set_) {
      if (cfd->IsDropped()) {
        continue;
      }
      ROCKS_LOG_INFO(db_options_->info_log,
                     "Column family [%s] (ID %u), log number is %" PRIu64 "\n",
                     cfd->GetName().c_str(), cfd->GetID(), cfd->GetLogNumber());
    }
  }

  for (auto& builder : builders) {
    delete builder.second;
  }

  return s;
}
```

### 创建ColumnFamilyData
VersionSet::CreateColumnFamily()基于给定的ColumnFamilyOptions和VersionEdit创建Version和ColumnFamilyData，并关联两者（Version引用该ColumnFamilyData，ColumnFamilyData中的关于Version的双向链表中也将包含新创建的Version）：
```
ColumnFamilyData* VersionSet::CreateColumnFamily(
    const ColumnFamilyOptions& cf_options, VersionEdit* edit) {
  assert(edit->is_column_family_add_);

  /*创建dummy_versions作为即将创建的ColumnFamilyData中Version双向链表表头*/
  Version* dummy_versions = new Version(nullptr, this);
  // Ref() dummy version once so that later we can call Unref() to delete it
  // by avoiding calling "delete" explicitly (~Version is private)
  dummy_versions->Ref();
  /*创建ColumnFamilyData，dummy_versions作为其中关于Version的双向链表表头*/
  auto new_cfd = column_family_set_->CreateColumnFamily(
      edit->column_family_name_, edit->column_family_, dummy_versions,
      cf_options);

  /*创建Version*/
  Version* v = new Version(new_cfd, this, current_version_number_++);

  // Fill level target base information.
  v->storage_info()->CalculateBaseBytes(*new_cfd->ioptions(),
                                        *new_cfd->GetLatestMutableCFOptions());
  /*将新创建的Version添加到ColumnFamilyData中关于Version的双向链表中，并设置其为Current Version*/
  AppendVersion(new_cfd, v);
  // GetLatestMutableCFOptions() is safe here without mutex since the
  // cfd is not available to client
  /*为该ColumnFamilyData创建Memtable*/
  new_cfd->CreateNewMemtable(*new_cfd->GetLatestMutableCFOptions(),
                             LastSequence());
  new_cfd->SetLogNumber(edit->log_number_);
  return new_cfd;
}
```

### VersionSet::LogAndApply
VersionSet::LogAndApply()将VersionEdit列表应用到当前最新的Version中，以生成新的最新的Version。
```
Status VersionSet::LogAndApply(ColumnFamilyData* column_family_data,
                               const MutableCFOptions& mutable_cf_options,
                               const autovector<VersionEdit*>& edit_list,
                               InstrumentedMutex* mu, Directory* db_directory,
                               bool new_descriptor_log,
                               const ColumnFamilyOptions* new_cf_options) {
  mu->AssertHeld();
  // num of edits
  auto num_edits = edit_list.size();
  if (num_edits == 0) {
    return Status::OK();
  } else if (num_edits > 1) {
#ifndef NDEBUG
    // no group commits for column family add or drop
    /* 如果edit_list中的VersionEdit数目不为1，则这些VersionEdit一定不是关于创建/删除
     * ColumnFamily相关的VersionEdit
     */
    for (auto& edit : edit_list) {
      assert(!edit->IsColumnFamilyManipulation());
    }
#endif
  }

  // column_family_data can be nullptr only if this is column_family_add.
  // in that case, we also need to specify ColumnFamilyOptions
  /*如果参数中的column_family_data为nullptr，则一定是关于创建ColumnFamily的VersionEdit*/
  if (column_family_data == nullptr) {
    assert(num_edits == 1);
    assert(edit_list[0]->is_column_family_add_);
    assert(new_cf_options != nullptr);
  }

  // queue our request
  /*为当前的提交创建一个ManifestWriter，其中包含需要更新Manifest文件相关的信息*/
  ManifestWriter w(mu, column_family_data, edit_list);
  /*将新创建的ManifestWriter添加到全局的manifest_writers队列中的末尾*/
  manifest_writers_.push_back(&w);
  /*新创建的ManifestWriter需要等待它前面的所有ManifestWriters完成之后才能执行*/
  while (!w.done && &w != manifest_writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status;
  }
  
  if (column_family_data != nullptr && column_family_data->IsDropped()) {
    // if column family is dropped by the time we get here, no need to write
    // anything to the manifest
    /* 如果发现ColumnFamily被drop了，则无需向manifest文件写入任何信息，所以从manifest_writers
     * 中移除队列头的ManifestWriter，就是前面为当前提交创建的ManifestWriter。同时唤醒
     * manifest_writers队列中后续被阻塞的ManifestWriters
     */
    manifest_writers_.pop_front();
    // Notify new head of write queue
    if (!manifest_writers_.empty()) {
      manifest_writers_.front()->cv.Signal();
    }
    // we steal this code to also inform about cf-drop
    return Status::ShutdownInProgress();
  }

  autovector<VersionEdit*> batch_edits;
  Version* v = nullptr;
  std::unique_ptr<BaseReferencedVersionBuilder> builder_guard(nullptr);

  // process all requests in the queue
  ManifestWriter* last_writer = &w;
  assert(!manifest_writers_.empty());
  assert(manifest_writers_.front() == &w);
  /*w.edit_list实际上就是参数中的edit_list，w一定是manifest_writers队列头部的元素*/
  if (w.edit_list.front()->IsColumnFamilyManipulation()) {
    // no group commits for column family add or drop
    /*关于创建/删除ColumnFamily的VersionEdit要单独提交*/
    LogAndApplyCFHelper(w.edit_list.front());
    batch_edits.push_back(w.edit_list.front());
  } else {
    /* 至此，manifest_writers队列头的元素w中的edit_list.front()不是关于ColumnFamily的
     * 创建/删除操作，则尝试合并多个ManifestWriter，合并必须满足如下条件：
     *  A. writer->edit_list.front()不是关于ColumnFamily的创建/删除操作，这样可以保证
     *     writer->edit_list中所有的VersionEdit都不是关于ColumnFamily的创建/删除操作
     *  B. writer->cfd->GetID() != column_family_data->GetID()，即不能跨越不同的ColumnFamily
     *     执行group commit
     */
     
    /*创建新的Version*/
    v = new Version(column_family_data, this, current_version_number_++);
    /*基于column_family_data的当前Version创建VersionBuilder*/
    builder_guard.reset(new BaseReferencedVersionBuilder(column_family_data));
    auto* builder = builder_guard->version_builder();
    for (const auto& writer : manifest_writers_) {
      if (writer->edit_list.front()->IsColumnFamilyManipulation() ||
          writer->cfd->GetID() != column_family_data->GetID()) {
        // no group commits for column family add or drop
        // also, group commits across column families are not supported
        /*不满足合并条件，就停止合并*/
        break;
      }
      
      /* 记录下来最后一个合并的ManifestWriter，并且将合并的ManifestWriter中所有
       * VersionEdit添加到batch_edits中
       */
      last_writer = writer;
      for (const auto& edit : writer->edit_list) {
        /* 利用builder，将edit中的添加/删除的文件集合信息应用到column_family_data的当前
         * Version中（因为builder是基于column_family_data的当前Version创建的）
         */
        LogAndApplyHelper(column_family_data, builder, v, edit, mu);
        batch_edits.push_back(edit);
      }
    }
    
    /* 将column_family_data的当前Version信息（存放在VersionBuilder::rep_::base_vstorage_）和
     * edit中的添加/删除的文件集合信息（已经在LogAndApplyHelper中存放到VersionBuilder::rep_::levels_
     * 中了）进行合并，并存放到v对应的VersionStorageInfo中
     */
    builder->SaveTo(v->storage_info());
  }

  // Initialize new descriptor log file if necessary by creating
  // a temporary file that contains a snapshot of the current version.
  uint64_t new_manifest_file_size = 0;
  Status s;

  assert(pending_manifest_file_number_ == 0);
  /*descriptor_log_是关于manifest文件的log::Writer*/
  /* 如果没有关于manifest文件的log::Writer，或者当前manifest文件的大小超过了设定的
   * db_options_->max_manifest_file_size，则创建新的manifest文件及其对应的log::Writer
   */
  if (!descriptor_log_ ||
      manifest_file_size_ > db_options_->max_manifest_file_size) {
    /* 为新创建的manifest文件分配编号，在NewFileNumber()中会将next_file_number_加1，
     * 返回加1之前的next_file_number_
     */
    pending_manifest_file_number_ = NewFileNumber();
    batch_edits.back()->SetNextFile(next_file_number_.load());
    new_descriptor_log = true;
  } else {
    pending_manifest_file_number_ = manifest_file_number_;
  }

  if (new_descriptor_log) {
    // if we're writing out new snapshot make sure to persist max column family
    if (column_family_set_->GetMaxColumnFamily() > 0) {
      w.edit_list.front()->SetMaxColumnFamily(
          column_family_set_->GetMaxColumnFamily());
    }
  }

  // Unlock during expensive operations. New writes cannot get here
  // because &w is ensuring that all new writes get queued.
  {

    mu->Unlock();

    TEST_SYNC_POINT("VersionSet::LogAndApply:WriteManifest");
    if (!w.edit_list.front()->IsColumnFamilyManipulation() &&
        this->GetColumnFamilySet()->get_table_cache()->GetCapacity() ==
            TableCache::kInfiniteCapacity) {
      // unlimited table cache. Pre-load table handle now.
      // Need to do it out of the mutex.
      builder_guard->version_builder()->LoadTableHandlers(
          column_family_data->internal_stats(),
          column_family_data->ioptions()->optimize_filters_for_hits,
          true /* prefetch_index_and_filter_in_cache */);
    }

    // This is fine because everything inside of this block is serialized --
    // only one thread can be here at the same time
    if (new_descriptor_log) {
      // create manifest file
      /* 创建新的manifest文件（descriptor_file），并为之创建log::Writer（保存在
       * descriptor_log_中），并借助descriptor_log_来保存当前的快照信息，包括所有
       * ColumnFamilies信息，所有层级的文件信息
       */
      ROCKS_LOG_INFO(db_options_->info_log, "Creating manifest %" PRIu64 "\n",
                     pending_manifest_file_number_);
      unique_ptr<WritableFile> descriptor_file;
      EnvOptions opt_env_opts = env_->OptimizeForManifestWrite(env_options_);
      /*创建manifest文件，即descriptor_file*/
      s = NewWritableFile(
          env_, DescriptorFileName(dbname_, pending_manifest_file_number_),
          &descriptor_file, opt_env_opts);
      if (s.ok()) {
        descriptor_file->SetPreallocationBlockSize(
            db_options_->manifest_preallocation_size);

        /*为该manifest文件关联一个log::Writer，即descriptor_log_*/
        unique_ptr<WritableFileWriter> file_writer(
            new WritableFileWriter(std::move(descriptor_file), opt_env_opts));
        descriptor_log_.reset(
            new log::Writer(std::move(file_writer), 0, false));
        
        /* 借助descriptor_log_来保存当前的快照信息（包括所有ColumnFamilies自身信息，所有
         * ColumnFamilies在各层级的文件信息）到新创建的manifest文件中
         */
        s = WriteSnapshot(descriptor_log_.get());
      }
    }

    if (!w.edit_list.front()->IsColumnFamilyManipulation()) {
      // This is cpu-heavy operations, which should be called outside mutex.
      v->PrepareApply(mutable_cf_options, true);
    }

    // Write new record to MANIFEST log
    if (s.ok()) {
      for (auto& e : batch_edits) {
        std::string record;
        if (!e->EncodeTo(&record)) {
          s = Status::Corruption(
              "Unable to Encode VersionEdit:" + e->DebugString(true));
          break;
        }
        TEST_KILL_RANDOM("VersionSet::LogAndApply:BeforeAddRecord",
                         rocksdb_kill_odds * REDUCE_ODDS2);
        s = descriptor_log_->AddRecord(record);
        if (!s.ok()) {
          break;
        }
      }
      if (s.ok()) {
        s = SyncManifest(env_, db_options_, descriptor_log_->file());
      }
      if (!s.ok()) {
        ROCKS_LOG_ERROR(db_options_->info_log, "MANIFEST write: %s\n",
                        s.ToString().c_str());
      }
    }

    // If we just created a new descriptor file, install it by writing a
    // new CURRENT file that points to it.
    if (s.ok() && new_descriptor_log) {
      s = SetCurrentFile(env_, dbname_, pending_manifest_file_number_,
                         db_directory);
    }

    if (s.ok()) {
      // find offset in manifest file where this version is stored.
      new_manifest_file_size = descriptor_log_->file()->GetFileSize();
    }

    if (w.edit_list.front()->is_column_family_drop_) {
      TEST_SYNC_POINT("VersionSet::LogAndApply::ColumnFamilyDrop:0");
      TEST_SYNC_POINT("VersionSet::LogAndApply::ColumnFamilyDrop:1");
      TEST_SYNC_POINT("VersionSet::LogAndApply::ColumnFamilyDrop:2");
    }

    LogFlush(db_options_->info_log);
    TEST_SYNC_POINT("VersionSet::LogAndApply:WriteManifestDone");
    mu->Lock();
  }

  // Append the old mainfest file to the obsolete_manifests_ list to be deleted
  // by PurgeObsoleteFiles later.
  if (s.ok() && new_descriptor_log) {
    obsolete_manifests_.emplace_back(
        DescriptorFileName("", manifest_file_number_));
  }

  // Install the new version
  if (s.ok()) {
    if (w.edit_list.front()->is_column_family_add_) {
      // no group commit on column family add
      assert(batch_edits.size() == 1);
      assert(new_cf_options != nullptr);
      CreateColumnFamily(*new_cf_options, w.edit_list.front());
    } else if (w.edit_list.front()->is_column_family_drop_) {
      assert(batch_edits.size() == 1);
      column_family_data->SetDropped();
      if (column_family_data->Unref()) {
        delete column_family_data;
      }
    } else {
      uint64_t max_log_number_in_batch  = 0;
      for (auto& e : batch_edits) {
        if (e->has_log_number_) {
          max_log_number_in_batch =
              std::max(max_log_number_in_batch, e->log_number_);
        }
      }
      if (max_log_number_in_batch != 0) {
        assert(column_family_data->GetLogNumber() <= max_log_number_in_batch);
        column_family_data->SetLogNumber(max_log_number_in_batch);
      }
      AppendVersion(column_family_data, v);
    }

    manifest_file_number_ = pending_manifest_file_number_;
    manifest_file_size_ = new_manifest_file_size;
    prev_log_number_ = w.edit_list.front()->prev_log_number_;
  } else {
    std::string version_edits;
    for (auto& e : batch_edits) {
      version_edits = version_edits + "\n" + e->DebugString(true);
    }
    ROCKS_LOG_ERROR(
        db_options_->info_log,
        "[%s] Error in committing version edit to MANIFEST: %s",
        column_family_data ? column_family_data->GetName().c_str() : "<null>",
        version_edits.c_str());
    delete v;
    if (new_descriptor_log) {
      ROCKS_LOG_INFO(db_options_->info_log, "Deleting manifest %" PRIu64
                                            " current manifest %" PRIu64 "\n",
                     manifest_file_number_, pending_manifest_file_number_);
      descriptor_log_.reset();
      env_->DeleteFile(
          DescriptorFileName(dbname_, pending_manifest_file_number_));
    }
  }
  pending_manifest_file_number_ = 0;

  // wake up all the waiting writers
  while (true) {
    ManifestWriter* ready = manifest_writers_.front();
    manifest_writers_.pop_front();
    if (ready != &w) {
      ready->status = s;
      ready->done = true;
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }
  // Notify new head of write queue
  if (!manifest_writers_.empty()) {
    manifest_writers_.front()->cv.Signal();
  }
  return s;
}
```


```
void VersionSet::LogAndApplyCFHelper(VersionEdit* edit) {
  assert(edit->IsColumnFamilyManipulation());
  edit->SetNextFile(next_file_number_.load());
  edit->SetLastSequence(last_sequence_);
  if (edit->is_column_family_drop_) {
    // if we drop column family, we have to make sure to save max column family,
    // so that we don't reuse existing ID
    edit->SetMaxColumnFamily(column_family_set_->GetMaxColumnFamily());
  }
}
```


```
void VersionSet::LogAndApplyHelper(ColumnFamilyData* cfd,
                                   VersionBuilder* builder, Version* v,
                                   VersionEdit* edit, InstrumentedMutex* mu) {
  mu->AssertHeld();
  assert(!edit->IsColumnFamilyManipulation());

  if (edit->has_log_number_) {
    assert(edit->log_number_ >= cfd->GetLogNumber());
    assert(edit->log_number_ < next_file_number_.load());
  }

  if (!edit->has_prev_log_number_) {
    edit->SetPrevLogNumber(prev_log_number_);
  }
  edit->SetNextFile(next_file_number_.load());
  edit->SetLastSequence(last_sequence_);

  /* 根据VersionEdit中关于的删除文件集合信息和添加文件集合信息更新VersionBuilder中
   * 维护的关于每一个Level的状态（即添加文件集合和删除文件集合）
   */
  builder->Apply(edit);
}
```


```
Status VersionSet::WriteSnapshot(log::Writer* log) {
  // TODO: Break up into multiple records to reduce memory usage on recovery?

  // WARNING: This method doesn't hold a mutex!!

  // This is done without DB mutex lock held, but only within single-threaded
  // LogAndApply. Column family manipulations can only happen within LogAndApply
  // (the same single thread), so we're safe to iterate.
  
  /* 保存所有ColumnFamilies信息（包括ColumnFamily自身的信息和关于ColumnFamily在
   * 每一层级中的文件信息，被drop的ColumnFamilies除外）
   */
  for (auto cfd : *column_family_set_) {
    if (cfd->IsDropped()) {
      /*略过被drop的ColumnFamilies*/
      continue;
    }
    
    {
      // Store column family info
      /*借助VersionEdit来保存当前的ColumnFamily信息*/
      VersionEdit edit;
      if (cfd->GetID() != 0) {
        // default column family is always there,
        // no need to explicitly write it
        edit.AddColumnFamily(cfd->GetName());
        edit.SetColumnFamily(cfd->GetID());
      }
      edit.SetComparatorName(
          cfd->internal_comparator().user_comparator()->Name());
      std::string record;
      /*将VersionEdit序列化为字符串*/
      if (!edit.EncodeTo(&record)) {
        return Status::Corruption(
            "Unable to Encode VersionEdit:" + edit.DebugString(true));
      }
      
      /*将当前ColumnFamily自身的信息保存到manifest文件中*/
      Status s = log->AddRecord(record);
      if (!s.ok()) {
        return s;
      }
    }

    {
      // Save files
      /*借助VersionEdit来保存当前ColumnFamily在每一层级的文件信息*/
      VersionEdit edit;
      /*首先记录当前的ColumnFamily ID*/
      edit.SetColumnFamily(cfd->GetID());

      /*接着记录该ColumnFamily在各层级的文件信息*/
      for (int level = 0; level < cfd->NumberLevels(); level++) {
        /*将该ColumnFamily在level这一层级的所有文件信息都添加到VersionEdit中*/
        for (const auto& f :
             cfd->current()->storage_info()->LevelFiles(level)) {
          edit.AddFile(level, f->fd.GetNumber(), f->fd.GetPathId(),
                       f->fd.GetFileSize(), f->smallest, f->largest,
                       f->smallest_seqno, f->largest_seqno,
                       f->marked_for_compaction);
        }
      }
      
      /* 在VersionEdit中记录在WAL日志中的包含该ColumnFamily的数据的最早的日志
       * 编号（更早的日志对于该ColumnFamily的数据不再具有价值了）
       */
      edit.SetLogNumber(cfd->GetLogNumber());
      /*将VersionEdit序列化为字符串*/
      std::string record;
      if (!edit.EncodeTo(&record)) {
        return Status::Corruption(
            "Unable to Encode VersionEdit:" + edit.DebugString(true));
      }
      
      /*将当前ColumnFamily在各层的文件信息保存到manifest文件中*/
      Status s = log->AddRecord(record);
      if (!s.ok()) {
        return s;
      }
    }
  }

  return Status::OK();
}
```


# 参考
[【Rocksdb实现分析及优化】FileIndexer](http://kernelmaker.github.io/Rocksdb_file_indexers)


