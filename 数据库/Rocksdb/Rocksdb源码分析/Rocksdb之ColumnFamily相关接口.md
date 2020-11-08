# 提纲
[toc]

# 接口
Rocksdb提供了创建、删除和列举ColumnFamily的方法如下：
```
  // ListColumnFamilies will open the DB specified by argument name
  // and return the list of all column families in that DB
  // through column_families argument. The ordering of
  // column families in column_families is unspecified.
  /*返回指定名称的数据库中ColumnFamilies列表*/
  static Status ListColumnFamilies(const DBOptions& db_options,
                                   const std::string& name,
                                   std::vector<std::string>* column_families);

  // Create a column_family and return the handle of column family
  // through the argument handle.
  /*创建一个ColumnFamily并在handle中返回该ColumnFamily对应的ColumnFamilyHandle*/
  virtual Status CreateColumnFamily(const ColumnFamilyOptions& options,
                                    const std::string& column_family_name,
                                    ColumnFamilyHandle** handle);

  // Bulk create column families with the same column family options.
  // Return the handles of the column families through the argument handles.
  // In case of error, the request may succeed partially, and handles will
  // contain column family handles that it managed to create, and have size
  // equal to the number of created column families.
  /* 批量创建ColumnFamilies并在handles中返回所有ColumnFamilies对应的ColumnFamilyHandle，
   * 该方法可能成功创建部分ColumnFamilies，handles中只包含那些创建成功的ColumnFamily
   * 对应的ColumnFamilyHandle
   */
  virtual Status CreateColumnFamilies(
      const ColumnFamilyOptions& options,
      const std::vector<std::string>& column_family_names,
      std::vector<ColumnFamilyHandle*>* handles);

  // Bulk create column families.
  // Return the handles of the column families through the argument handles.
  // In case of error, the request may succeed partially, and handles will
  // contain column family handles that it managed to create, and have size
  // equal to the number of created column families.
  /* 批量创建ColumnFamilies并在handles中返回所有ColumnFamilies对应的ColumnFamilyHandle，
   * 该方法可能成功创建部分ColumnFamilies，handles中只包含那些创建成功的ColumnFamily
   * 对应的ColumnFamilyHandle
   */
  virtual Status CreateColumnFamilies(
      const std::vector<ColumnFamilyDescriptor>& column_families,
      std::vector<ColumnFamilyHandle*>* handles);

  // Drop a column family specified by column_family handle. This call
  // only records a drop record in the manifest and prevents the column
  // family from flushing and compacting.
  /*删除指定ColumnFamilyHandle对应的ColumnFamily*/
  virtual Status DropColumnFamily(ColumnFamilyHandle* column_family);

  // Bulk drop column families. This call only records drop records in the
  // manifest and prevents the column families from flushing and compacting.
  // In case of error, the request may succeed partially. User may call
  // ListColumnFamilies to check the result.
  /* 批量删除由ColumnFamilyHandle数组指示的所有ColumnFamilies，该方法只会在manifest
   * 文件中记录这些被删除的ColumnFamilies并且阻止关于这些ColumnFamilies的flush和compact，
   * 该方法可能只成功删除部分ColumnFamilies，可以通过ListColumnFamilies获取删除结果
   */
  virtual Status DropColumnFamilies(
      const std::vector<ColumnFamilyHandle*>& column_families);

  // Close a column family specified by column_family handle and destroy
  // the column family handle specified to avoid double deletion. This call
  // deletes the column family handle by default. Use this method to
  // close column family instead of deleting column family handle directly
  /*关闭指定ColumnFamilyHandle对应的ColumnFamily，同时销毁该ColumnFamilyHandle*/
  virtual Status DestroyColumnFamilyHandle(ColumnFamilyHandle* column_family);
```

# 实现
在DB类中提供了所有CreateColumnFamily/CreateColumnFamilies和所有DropColumnFamily/DropColumnFamilies操作的默认实现，但是都直接返回Status::NotSupported("")，如：
```
Status DB::CreateColumnFamily(const ColumnFamilyOptions& cf_options,
                              const std::string& column_family_name,
                              ColumnFamilyHandle** handle) {
  return Status::NotSupported("");
}
```

在DBImpl中重载了所有CreateColumnFamily/CreateColumnFamilies和所有DropColumnFamily/DropColumnFamilies方法：
```
class DBImpl : public DB {
  virtual Status CreateColumnFamily(const ColumnFamilyOptions& cf_options,
                                    const std::string& column_family,
                                    ColumnFamilyHandle** handle) override;
  virtual Status CreateColumnFamilies(
      const ColumnFamilyOptions& cf_options,
      const std::vector<std::string>& column_family_names,
      std::vector<ColumnFamilyHandle*>* handles) override;
  virtual Status CreateColumnFamilies(
      const std::vector<ColumnFamilyDescriptor>& column_families,
      std::vector<ColumnFamilyHandle*>* handles) override;
  virtual Status DropColumnFamily(ColumnFamilyHandle* column_family) override;
  virtual Status DropColumnFamilies(
      const std::vector<ColumnFamilyHandle*>& column_families) override;
};
```

在DB类中提供了ListColumnFamilies和DestroyColumnFamilyHandle的默认实现：
```
Status DB::ListColumnFamilies(const DBOptions& db_options,
                              const std::string& name,
                              std::vector<std::string>* column_families) {
  return VersionSet::ListColumnFamilies(column_families, name, db_options.env);
}

Status DB::ListColumnFamilies(const DBOptions& db_options,
                              const std::string& name,
                              std::vector<std::string>* column_families) {
  return VersionSet::ListColumnFamilies(column_families, name, db_options.env);
}
```

在DBImple类中则没有重载这些实现。

## DBImpl::CreateColumnFamily

```
Status DBImpl::CreateColumnFamily(const ColumnFamilyOptions& cf_options,
                                  const std::string& column_family,
                                  ColumnFamilyHandle** handle) {
  assert(handle != nullptr);
  Status s = CreateColumnFamilyImpl(cf_options, column_family, handle);
  if (s.ok()) {
    s = WriteOptionsFile(true /*need_mutex_lock*/,
                         true /*need_enter_write_thread*/);
  }
  return s;
}
```


```
Status DBImpl::CreateColumnFamilyImpl(const ColumnFamilyOptions& cf_options,
                                      const std::string& column_family_name,
                                      ColumnFamilyHandle** handle) {
  Status s;
  Status persist_options_status;
  *handle = nullptr;

  /*关于压缩和并发写的检查*/
  s = CheckCompressionSupported(cf_options);
  if (s.ok() && immutable_db_options_.allow_concurrent_memtable_write) {
    s = CheckConcurrentWritesSupported(cf_options);
  }
  if (!s.ok()) {
    return s;
  }

  {
    InstrumentedMutexLock l(&mutex_);

    /*如果该ColumnFamily已经存在，则返回*/
    if (versions_->GetColumnFamilySet()->GetColumnFamily(column_family_name) !=
        nullptr) {
      return Status::InvalidArgument("Column family already exists");
    }
    
    /*为被创建的ColumnFamily关联一个VersionEdit*/
    VersionEdit edit;
    edit.AddColumnFamily(column_family_name);
    uint32_t new_id = versions_->GetColumnFamilySet()->GetNextColumnFamilyID();
    edit.SetColumnFamily(new_id);
    edit.SetLogNumber(logfile_number_);
    edit.SetComparatorName(cf_options.comparator->Name());

    // LogAndApply will both write the creation in MANIFEST and create
    // ColumnFamilyData object
    /*创建ColumnFamily相关的Writer不会执行group commit，而是单独提交*/
    {  // write thread
      WriteThread::Writer w;
      /*等待之前的所有的Writer都结束才开始新的关于创建ColumnFamily的Writer*/
      write_thread_.EnterUnbatched(&w, &mutex_);
      // LogAndApply will both write the creation in MANIFEST and create
      // ColumnFamilyData object
      s = versions_->LogAndApply(nullptr, MutableCFOptions(cf_options), &edit,
                                 &mutex_, directories_.GetDbDir(), false,
                                 &cf_options);
      /*结束关于创建ColumnFamily的Writer，并唤醒后续被挂起的（pending）Writer*/                           
      write_thread_.ExitUnbatched(&w);
    }
    
    if (s.ok()) {
      single_column_family_mode_ = false;
      auto* cfd =
          versions_->GetColumnFamilySet()->GetColumnFamily(column_family_name);
      assert(cfd != nullptr);
      delete InstallSuperVersionAndScheduleWork(
          cfd, nullptr, *cfd->GetLatestMutableCFOptions());

      if (!cfd->mem()->IsSnapshotSupported()) {
        is_snapshot_supported_ = false;
      }

      *handle = new ColumnFamilyHandleImpl(cfd, this, &mutex_);
      ROCKS_LOG_INFO(immutable_db_options_.info_log,
                     "Created column family [%s] (ID %u)",
                     column_family_name.c_str(), (unsigned)cfd->GetID());
    } else {
      ROCKS_LOG_ERROR(immutable_db_options_.info_log,
                      "Creating column family [%s] FAILED -- %s",
                      column_family_name.c_str(), s.ToString().c_str());
    }
  }  // InstrumentedMutexLock l(&mutex_)

  // this is outside the mutex
  if (s.ok()) {
    NewThreadStatusCfInfo(
        reinterpret_cast<ColumnFamilyHandleImpl*>(*handle)->cfd());
  }
  return s;
}
```


## DBImpl::CreateColumnFamilies

```
Status DBImpl::CreateColumnFamilies(
    const ColumnFamilyOptions& cf_options,
    const std::vector<std::string>& column_family_names,
    std::vector<ColumnFamilyHandle*>* handles) {
  assert(handles != nullptr);
  handles->clear();
  size_t num_cf = column_family_names.size();
  Status s;
  bool success_once = false;
  for (size_t i = 0; i < num_cf; i++) {
    ColumnFamilyHandle* handle;
    s = CreateColumnFamilyImpl(cf_options, column_family_names[i], &handle);
    if (!s.ok()) {
      break;
    }
    handles->push_back(handle);
    success_once = true;
  }
  if (success_once) {
    Status persist_options_status = WriteOptionsFile(
        true /*need_mutex_lock*/, true /*need_enter_write_thread*/);
    if (s.ok() && !persist_options_status.ok()) {
      s = persist_options_status;
    }
  }
  return s;
}
```

## DBImpl::CreateColumnFamilies

```
Status DBImpl::CreateColumnFamilies(
    const std::vector<ColumnFamilyDescriptor>& column_families,
    std::vector<ColumnFamilyHandle*>* handles) {
  assert(handles != nullptr);
  handles->clear();
  size_t num_cf = column_families.size();
  Status s;
  bool success_once = false;
  for (size_t i = 0; i < num_cf; i++) {
    ColumnFamilyHandle* handle;
    s = CreateColumnFamilyImpl(column_families[i].options,
                               column_families[i].name, &handle);
    if (!s.ok()) {
      break;
    }
    handles->push_back(handle);
    success_once = true;
  }
  if (success_once) {
    Status persist_options_status = WriteOptionsFile(
        true /*need_mutex_lock*/, true /*need_enter_write_thread*/);
    if (s.ok() && !persist_options_status.ok()) {
      s = persist_options_status;
    }
  }
  return s;
}
```

## DBImpl::DropColumnFamily

```
Status DBImpl::DropColumnFamily(ColumnFamilyHandle* column_family) {
  assert(column_family != nullptr);
  Status s = DropColumnFamilyImpl(column_family);
  if (s.ok()) {
    s = WriteOptionsFile(true /*need_mutex_lock*/,
                         true /*need_enter_write_thread*/);
  }
  return s;
}
```

## DBImpl::DropColumnFamilies

```
Status DBImpl::DropColumnFamilies(
    const std::vector<ColumnFamilyHandle*>& column_families) {
  Status s;
  bool success_once = false;
  for (auto* handle : column_families) {
    s = DropColumnFamilyImpl(handle);
    if (!s.ok()) {
      break;
    }
    success_once = true;
  }
  if (success_once) {
    Status persist_options_status = WriteOptionsFile(
        true /*need_mutex_lock*/, true /*need_enter_write_thread*/);
    if (s.ok() && !persist_options_status.ok()) {
      s = persist_options_status;
    }
  }
  return s;
}
```

## DB::ListColumnFamilies
```
Status DB::ListColumnFamilies(const DBOptions& db_options,
                              const std::string& name,
                              std::vector<std::string>* column_families) {
  return VersionSet::ListColumnFamilies(column_families, name, db_options.env);
}
```

## DB::DestroyColumnFamilyHandle
```
Status DB::DestroyColumnFamilyHandle(ColumnFamilyHandle* column_family) {
  delete column_family;
  return Status::OK();
}
```
