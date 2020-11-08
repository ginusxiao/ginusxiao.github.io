# 提纲
[toc]

# 数据结构
## VersionBuilder
VersionBuilder用于将VersionEdit应用到某个特定的Version中，它提供以下接口：

- 利用VersionEdit来更新每一个层级的删除文件集合信息和添加文件集合信息；

- 整合每一个层级的文件（按序合并已经存在的文件和新添加的文件，删除所有待删除的文件），并将整合后的文件信息保存到VersionStorageInfo中；

```
class VersionBuilder {
 public:
  VersionBuilder(const EnvOptions& env_options, TableCache* table_cache,
                 VersionStorageInfo* base_vstorage, Logger* info_log = nullptr);
  ~VersionBuilder();
  void CheckConsistency(VersionStorageInfo* vstorage);
  void CheckConsistencyForDeletes(VersionEdit* edit, uint64_t number,
                                  int level);
  /*利用VersionEdit来更新每一个Level的删除文件集合信息和添加文件集合信息；*/                                  
  void Apply(VersionEdit* edit);
  /* 整合每一个层级的文件（按序合并已经存在的文件和新添加的文件，删除所有待
   * 删除的文件），并将整合后的文件信息保存到VersionStorageInfo中；
   */
  void SaveTo(VersionStorageInfo* vstorage);
  void LoadTableHandlers(InternalStats* internal_stats, int max_threads,
                         bool prefetch_index_and_filter_in_cache);
  void MaybeAddFile(VersionStorageInfo* vstorage, int level, FileMetaData* f);

 private:
  class Rep;
  Rep* rep_;
};
```

但是VersionBuilder的方法都是藉由VersionBuilder::Rep中相关方法来实现的。

## VersionBuilder::Rep
VersionBuilder::Rep中记录了如下信息：

- SST文件缓存；

- 基础的Version对应的VersionStorageInfo信息；

- 关于每一个Level的状态的数组（通过应用VersionEdit转换而来）；

- 每一个Level的文件排序器；

```
class VersionBuilder::Rep {
 private:
  // Helper to sort files_ in v
  // kLevel0 -- NewestFirstBySeqNo
  // kLevelNon0 -- BySmallestKey
  /* FileComparator用于对每一个level的文件进行排序，L0采用NewestFirstBySeqno，
   * 其它Level则采用BySmallestKey进行排序
   */
  struct FileComparator {
    /*文件排序方法*/
    enum SortMethod { kLevel0 = 0, kLevelNon0 = 1, } sort_method;
    /*除Level-0以外的层使用的关于InternalKey的排序器*/
    const InternalKeyComparator* internal_comparator;

    bool operator()(FileMetaData* f1, FileMetaData* f2) const {
      switch (sort_method) {
        case kLevel0:
          return NewestFirstBySeqNo(f1, f2);
        case kLevelNon0:
          return BySmallestKey(f1, f2, internal_comparator);
      }
      assert(false);
      return false;
    }
  };

  /*记录某一层中删除的文件和添加的文件*/
  struct LevelState {
    std::unordered_set<uint64_t> deleted_files;
    // Map from file number to file meta data.
    std::unordered_map<uint64_t, FileMetaData*> added_files;
  };

  const EnvOptions& env_options_;
  Logger* info_log_;
  TableCache* table_cache_;
  VersionStorageInfo* base_vstorage_;
  /*关于每一个Level的状态的数组*/
  LevelState* levels_;
  /*Level-0的文件排序器，对于Level-0采用NewestFirstBySeqNo来排序*/
  FileComparator level_zero_cmp_;
  /*除Level-0以外的其它层的文件排序器，对于除Level-0以外的Level采用BySmallestKey来排序*/
  FileComparator level_nonzero_cmp_;
};
```

## BaseReferencedVersionBuilder
BaseReferencedVersionBuilder封装了关于某个ColumnFamily的当前Version，和以当前Version为基础的VersionBuilder。实际上BaseReferencedVersionBuilder的作用就是将关于某个ColumnFamily的VersionEdit应用到该ColumnFamily当前最新的Version上。BaseReferencedVersionBuilder并没有提供自己的方法，可以直接使用VersionBuilder提供的方法。
```
class BaseReferencedVersionBuilder {
 public:
  explicit BaseReferencedVersionBuilder(ColumnFamilyData* cfd)
      : version_builder_(new VersionBuilder(
            cfd->current()->version_set()->env_options(), cfd->table_cache(),
            cfd->current()->storage_info(), cfd->ioptions()->info_log)),
        version_(cfd->current()) {
    /*增加当前Version的引用计数*/
    version_->Ref();
  }
  ~BaseReferencedVersionBuilder() {
    delete version_builder_;
    /*减少引用计数*/
    version_->Unref();
  }
  VersionBuilder* version_builder() { return version_builder_; }

 private:
  /*以关于某个ColumnFamily的当前Version(cfd->current())为基础的VersionBuilder*/
  VersionBuilder* version_builder_;
  /*关于某个ColumnFamily的当前Version(cfd->current())*/
  Version* version_;
};
```


# 实现
## VersionBuilder
### VersionBuilder构造函数

```
VersionBuilder::VersionBuilder(const EnvOptions& env_options,
                               TableCache* table_cache,
                               VersionStorageInfo* base_vstorage,
                               Logger* info_log)
    : rep_(new Rep(env_options, info_log, table_cache, base_vstorage)) {}
```


### VersionBuilder::Rep构造函数

```
  VersionBuilder::Rep::Rep(const EnvOptions& env_options, Logger* info_log, TableCache* table_cache,
      VersionStorageInfo* base_vstorage)
      : env_options_(env_options),
        info_log_(info_log),
        table_cache_(table_cache),
        base_vstorage_(base_vstorage) {
    /*所有Level的状态数组，每一个Level对应数组中一个entry*/
    levels_ = new LevelState[base_vstorage_->num_levels()];
    /*结合FileComparator中Operator()方法可知，Level-0采用NewestFirstBySeqNo来排序*/
    level_zero_cmp_.sort_method = FileComparator::kLevel0;
    /*结合FileComparator中Operator()方法可知，对于除Level-0以外的Level采用BySmallestKey来排序*/
    level_nonzero_cmp_.sort_method = FileComparator::kLevelNon0;
    level_nonzero_cmp_.internal_comparator =
        base_vstorage_->InternalComparator();
  }
```

### Level-0文件排序函数
NewestFirstBySeqNo()按照largest_seqno优先，smallest_seqno次之，fd.GetNumber()最后的方式进行排序。
```
bool NewestFirstBySeqNo(FileMetaData* a, FileMetaData* b) {
  if (a->largest_seqno != b->largest_seqno) {
    return a->largest_seqno > b->largest_seqno;
  }
  if (a->smallest_seqno != b->smallest_seqno) {
    return a->smallest_seqno > b->smallest_seqno;
  }
  // Break ties by file number
  return a->fd.GetNumber() > b->fd.GetNumber();
}
```


### 除Level-0以外的Level的文件排序函数
BySmallestKey()按照smallest key的顺序升序排序，如果smallest key相等，则通过fd.GetNumber()来进行排序。
```
bool BySmallestKey(FileMetaData* a, FileMetaData* b,
                   const InternalKeyComparator* cmp) {
  int r = cmp->Compare(a->smallest, b->smallest);
  if (r != 0) {
    return (r < 0);
  }
  // Break ties by file number
  return (a->fd.GetNumber() < b->fd.GetNumber());
}
```

### 运用VersionEdit中的关于添加的文件集合和删除的文件集合来更新每一个层级的状态（VersionBuilder::Apply）
VersionBuilder::Apply()根据VersionEdit中关于的删除文件集合信息和添加文件集合信息更新每一个Level的状态，实际上藉由VersionBuilder::Rep::Apply()来实现。
```
void VersionBuilder::Apply(VersionEdit* edit) { rep_->Apply(edit); }
```

VersionBuilder::Rep::Apply()根据VersionEdit中记录的删除文件集合信息和添加文件集合信息更新每一个Level的LevelState（包括添加文件集合和删除文件集合）。
```
  void VersionBuilder::Rep::Apply(VersionEdit* edit) {
    /*首先检查每一个Level的文件是否正确排序*/
    CheckConsistency(base_vstorage_);

    // Delete files
    /*处理删除文件，从VersionEdit中获取所有被删除的文件信息*/
    const VersionEdit::DeletedFileSet& del = edit->GetDeletedFiles();
    /*依次遍历这些被删除的文件，更新到相应的LevelState中*/
    for (const auto& del_file : del) {
      /*VersionEdit中记录了每一个被删除文件所在的Level和编号*/
      const auto level = del_file.first;
      const auto number = del_file.second;
      /*将删除文件添加到删除文件集合中*/
      levels_[level].deleted_files.insert(number);
      /*检查被删除的文件是否存在*/
      CheckConsistencyForDeletes(edit, number, level);

      /*从每一个Level的添加文件中移除被删除的文件*/
      auto exising = levels_[level].added_files.find(number);
      if (exising != levels_[level].added_files.end()) {
        UnrefFile(exising->second);
        levels_[level].added_files.erase(number);
      }
    }

    // Add new files
    /*处理添加文件，遍历VersionEdit中所有新添加的文件*/
    for (const auto& new_file : edit->GetNewFiles()) {
      /*为新添加的文件创建FileMetaData结构*/
      const int level = new_file.first;
      FileMetaData* f = new FileMetaData(new_file.second);
      f->refs = 1;

      /*从每一个Level的删除文件集合中移除新添加的文件，并将新添加的文件添加到添加文件集合中*/
      assert(levels_[level].added_files.find(f->fd.GetNumber()) ==
             levels_[level].added_files.end());
      levels_[level].deleted_files.erase(f->fd.GetNumber());
      levels_[level].added_files[f->fd.GetNumber()] = f;
    }
  }
```

### 检查每一个层级的文件是否按照规定排序的（VersionBuilder::CheckConsistency）
VersionBuilder::CheckConsistency()用于检查每一个Level的文件是不是都是按照规定排序的，它藉由VersionBuilder::Rep::CheckConsistency()来实现。
```
void VersionBuilder::CheckConsistency(VersionStorageInfo* vstorage) {
  rep_->CheckConsistency(vstorage);
}
```

```
  void VersionBuilder::Rep::CheckConsistency(VersionStorageInfo* vstorage) {
#ifdef NDEBUG
    if (!vstorage->force_consistency_checks()) {
      // Dont run consistency checks in release mode except if
      // explicitly asked to
      return;
    }
#endif
    // make sure the files are sorted correctly
    /*遍历每一个Level*/
    for (int level = 0; level < vstorage->num_levels(); level++) {
      /*比较当前Level中的所有相邻的文件*/
      auto& level_files = vstorage->LevelFiles(level);
      for (size_t i = 1; i < level_files.size(); i++) {
        auto f1 = level_files[i - 1];
        auto f2 = level_files[i];
        if (level == 0) {
          /*Level-0跟其他Level的排序函数不同，Level-0有其特殊的排序函数*/
          
          /*相邻两文件排序不正确*/
          if (!level_zero_cmp_(f1, f2)) {
            fprintf(stderr, "L0 files are not sorted properly");
            abort();
          }

          if (f2->smallest_seqno == f2->largest_seqno) {
            // This is an external file that we ingested
            SequenceNumber external_file_seqno = f2->smallest_seqno;
            if (!(external_file_seqno < f1->largest_seqno ||
                  external_file_seqno == 0)) {
              fprintf(stderr, "L0 file with seqno %" PRIu64 " %" PRIu64
                              " vs. file with global_seqno %" PRIu64 "\n",
                      f1->smallest_seqno, f1->largest_seqno,
                      external_file_seqno);
              abort();
            }
          } else if (f1->smallest_seqno <= f2->smallest_seqno) {
            fprintf(stderr, "L0 files seqno %" PRIu64 " %" PRIu64
                            " vs. %" PRIu64 " %" PRIu64 "\n",
                    f1->smallest_seqno, f1->largest_seqno, f2->smallest_seqno,
                    f2->largest_seqno);
            abort();
          }
        } else {
          /*对于其它Level的处理*/
          
          /*相邻两文件排序不正确*/
          if (!level_nonzero_cmp_(f1, f2)) {
            fprintf(stderr, "L%d files are not sorted properly", level);
            abort();
          }

          // Make sure there is no overlap in levels > 0
          /*对于除Level-0以外的Level，相邻两个文件之间不能存在交叠*/
          if (vstorage->InternalComparator()->Compare(f1->largest,
                                                      f2->smallest) >= 0) {
            fprintf(stderr, "L%d have overlapping ranges %s vs. %s\n", level,
                    (f1->largest).DebugString(true).c_str(),
                    (f2->smallest).DebugString(true).c_str());
            abort();
          }
        }
      }
    }
  }
```

### 检查VersionEdit中指定层的特定编号的待删除文件是否存在（VersionBuilder::CheckConsistencyForDeletes）
VersionBuilder::CheckConsistencyForDeletes()检查指定VersionEdit中，level所在的层中待删除的编号为number的文件是否存在，它实际是藉由VersionBuilder::Rep::CheckConsistencyForDeletes()来实现的。
```
void VersionBuilder::CheckConsistencyForDeletes(VersionEdit* edit,
                                                uint64_t number, int level) {
  rep_->CheckConsistencyForDeletes(edit, number, level);
}
```

```
  void VersionBuilder::Rep::CheckConsistencyForDeletes(VersionEdit* edit, uint64_t number,
                                  int level) {
#ifdef NDEBUG
    if (!base_vstorage_->force_consistency_checks()) {
      // Dont run consistency checks in release mode except if
      // explicitly asked to
      return;
    }
#endif
    // a file to be deleted better exist in the previous version
    /*依次遍历每一个Level，查找是否存在待删除的文件*/
    bool found = false;
    for (int l = 0; !found && l < base_vstorage_->num_levels(); l++) {
      /*该层中所有文件集合*/
      const std::vector<FileMetaData*>& base_files =
          base_vstorage_->LevelFiles(l);
      for (size_t i = 0; i < base_files.size(); i++) {
        FileMetaData* f = base_files[i];
        /*在该层中找到了待删除的文件*/
        if (f->fd.GetNumber() == number) {
          found = true;
          break;
        }
      }
    }
    
    // if the file did not exist in the previous version, then it
    // is possibly moved from lower level to higher level in current
    // version
    /* 如果不在前一个Version中，则检查它是否从level层移动到了较高的层级，从level+1层
     * 开始查找每一个Level的添加文件集合（added_files）
     */
    for (int l = level + 1; !found && l < base_vstorage_->num_levels(); l++) {
      auto& level_added = levels_[l].added_files;
      auto got = level_added.find(number);
      if (got != level_added.end()) {
        found = true;
        break;
      }
    }

    // maybe this file was added in a previous edit that was Applied
    /*如果也不在较高层级中，那么检查它是否在level这一层的添加文件集合中*/
    if (!found) {
      auto& level_added = levels_[level].added_files;
      auto got = level_added.find(number);
      if (got != level_added.end()) {
        found = true;
      }
    }
    
    if (!found) {
      /*没有找到，则assert*/
      fprintf(stderr, "not found %" PRIu64 "\n", number);
      abort();
    }
  }
```

### 将新添加的文件按序添加到已有的文件集合中（VersionBuilder::SaveTo）
VersionBuilder::SaveTo()用于将base_vstorage_中已经存在的文件和VersionBuilder::Rep::levels_中新添加的文件进行排序合并，它实际藉由VersionBuilder::Rep::SaveTo()实现。
```
void VersionBuilder::SaveTo(VersionStorageInfo* vstorage) {
  rep_->SaveTo(vstorage);
}
```

```
  void VersionBuilder::Rep::SaveTo(VersionStorageInfo* vstorage) {
    /*分别检查base_vstorage_和vstorage中每一层的文件是否是排序正确的*/
    CheckConsistency(base_vstorage_);
    CheckConsistency(vstorage);

    /* 依次遍历每一层，将base_vstorage_和VersionBuilder::Rep::levels_中相同层
     * 的文件按序合并，存放到vstorage中
     */
    for (int level = 0; level < base_vstorage_->num_levels(); level++) {
      /*根据层级，设定排序器*/
      const auto& cmp = (level == 0) ? level_zero_cmp_ : level_nonzero_cmp_;
      // Merge the set of added files with the set of pre-existing files.
      // Drop any deleted files.  Store the result in *v.
      /*base_vstorage中在指定层级的文件集合*/
      const auto& base_files = base_vstorage_->LevelFiles(level);
      /*base_vstorage_中关于指定层级的文件迭代器*/
      auto base_iter = base_files.begin();
      auto base_end = base_files.end();
      /*VersionBuilder::Rep::levels_中关于level这一层级待添加的文件集合*/
      const auto& unordered_added_files = levels_[level].added_files;
      /*扩大vstorage中关于level这一层级的文件个数*/
      vstorage->Reserve(level,
                        base_files.size() + unordered_added_files.size());

      // Sort added files for the level.
      /*将VersionBuilder::Rep::levels_中关于level这一层级待添加的文件排好序*/
      std::vector<FileMetaData*> added_files;
      added_files.reserve(unordered_added_files.size());
      for (const auto& pair : unordered_added_files) {
        added_files.push_back(pair.second);
      }
      std::sort(added_files.begin(), added_files.end(), cmp);

#ifndef NDEBUG
      FileMetaData* prev_file = nullptr;
#endif

      /*对于level这一层级已经排好序的待添加的文件，和base_vstorage_中level这一层级的文件进行合并*/
      for (const auto& added : added_files) {
#ifndef NDEBUG
        if (level > 0 && prev_file != nullptr) {
          assert(base_vstorage_->InternalComparator()->Compare(
                     prev_file->smallest, added->smallest) <= 0);
        }
        prev_file = added;
#endif

        // Add all smaller files listed in base_
        /* 查找当前待添加的文件added在[base_iter, base_end)文件集合中应当插入的位置bpos，
         * 则依次将[base_iter, bpos)范围内的文件添加到vstorage中
         */
        for (auto bpos = std::upper_bound(base_iter, base_end, added, cmp);
             base_iter != bpos; ++base_iter) {
          /*关于MaybeAddFile，请参考后续分析*/
          MaybeAddFile(vstorage, level, *base_iter);
        }

        /*然后将待添加的added这个文件添加到vstorage中*/
        MaybeAddFile(vstorage, level, added);
      }

      // Add remaining base files
      /*所有处于added_files中的待添加的文件已经添加完毕，则将base_vstorage_中剩余的文件添加到vstorage中*/
      for (; base_iter != base_end; ++base_iter) {
        MaybeAddFile(vstorage, level, *base_iter);
      }
    }

    /*检查合并后的vstorage中的文件是否排序正确*/
    CheckConsistency(vstorage);
  }
```

### VersionBuilder::LoadTableHandlers
VersionBuilder::LoadTableHandlers()
```
  void LoadTableHandlers(InternalStats* internal_stats, int max_threads,
                         bool prefetch_index_and_filter_in_cache) {
    assert(table_cache_ != nullptr);
    // <file metadata, level>
    /*将所有新添加的文件添加到files_meta数组中*/
    std::vector<std::pair<FileMetaData*, int>> files_meta;
    for (int level = 0; level < base_vstorage_->num_levels(); level++) {
      for (auto& file_meta_pair : levels_[level].added_files) {
        auto* file_meta = file_meta_pair.second;
        assert(!file_meta->table_reader_handle);
        files_meta.emplace_back(file_meta, level);
      }
    }

    /* 定义了一个函数（通过lambda表达式实现），作为线程（后面会创建之）的执行函数，
     * 所以接下来的这个代码块并不是LoadTableHandlers中的逻辑
     */
    std::atomic<size_t> next_file_meta_idx(0);
    std::function<void()> load_handlers_func = [&]() {
      while (true) {
        /*因为存在多个线程并发争用"next_file_meta_idx"这一资源，所以需要原子的获取之*/
        size_t file_idx = next_file_meta_idx.fetch_add(1);
        if (file_idx >= files_meta.size()) {
          /*至多有files_meta.size()个线程*/
          break;
        }

        /*当前线程处理file_idx对应的文件*/
        auto* file_meta = files_meta[file_idx].first;
        int level = files_meta[file_idx].second;
        /*在table cache中找到指定文件的table_reader_handle*/
        table_cache_->FindTable(env_options_,
                                *(base_vstorage_->InternalComparator()),
                                file_meta->fd, &file_meta->table_reader_handle,
                                false /*no_io */, true /* record_read_stats */,
                                internal_stats->GetFileReadHist(level), false,
                                level, prefetch_index_and_filter_in_cache);
        if (file_meta->table_reader_handle != nullptr) {
          // Load table_reader
          /*在table cache中通过table_reader_handle获取table_reader*/
          file_meta->fd.table_reader = table_cache_->GetTableReaderFromHandle(
              file_meta->table_reader_handle);
        }
      }
    };

    if (max_threads <= 1) {
      /*如果不多于一个线程，直接在本线程执行*/
      load_handlers_func();
    } else {
      /* 否则启动max_threads个线程，由这些线程来共同完成所有处于files_meta中的文件
       * 的table reader的加载工作（每个线程负责files_meta中的至少一个文件，每个文件
       * 仅且只有一个线程对其负责）
       */
      std::vector<port::Thread> threads;
      for (int i = 0; i < max_threads; i++) {
        threads.emplace_back(load_handlers_func);
      }

      /*等待所有线程执行完毕*/
      for (auto& t : threads) {
        t.join();
      }
    }
  }
```

在VersionBuilder::LoadTableHandlers()中每个线程加载每个文件的table reader的时候涉及到table cache，那么table cache是什么呢？

通过跟踪VersionBuilder::Rep::table_cache_，可以看到table cache来自于ColumnFamilyData::table_cache_：

```
BaseReferencedVersionBuilder::BaseReferencedVersionBuilder(ColumnFamilyData* cfd)
  : version_builder_(new VersionBuilder(
        cfd->current()->version_set()->env_options(), cfd->table_cache(),
        cfd->current()->storage_info(), cfd->ioptions()->info_log)),
    version_(cfd->current()) {
    version_->Ref();
}
  
VersionBuilder::VersionBuilder(const EnvOptions& env_options,
                               TableCache* table_cache,
                               VersionStorageInfo* base_vstorage,
                               Logger* info_log)
    : rep_(new Rep(env_options, info_log, table_cache, base_vstorage)) {}
    
VersionBuilder::Rep(const EnvOptions& env_options, Logger* info_log, TableCache* table_cache,
  VersionStorageInfo* base_vstorage)
  : env_options_(env_options),
    info_log_(info_log),
    table_cache_(table_cache),
    base_vstorage_(base_vstorage) {
......
}
```

继续跟踪ColumnFamilyData::table_cache_：

```
ColumnFamilyData::ColumnFamilyData(
    uint32_t id, const std::string& name, Version* _dummy_versions,
    Cache* _table_cache, WriteBufferManager* write_buffer_manager,
    const ColumnFamilyOptions& cf_options, const ImmutableDBOptions& db_options,
    const EnvOptions& env_options, ColumnFamilySet* column_family_set)
    : id_(id),
      ...... {
  
  ......

  // if _dummy_versions is nullptr, then this is a dummy column family.
  if (_dummy_versions != nullptr) {
    ......
    
    table_cache_.reset(new TableCache(ioptions_, env_options, _table_cache));
    
    ......
  }

  ......
}
```

可以看到，在ColumnFamilyData的构造函数中会为该ColumnFamily创建一个TableCache，关于TableCache请参考“Rocksdb之TableCache”。



### 根据文件是否在待删除文件集合中来决定添加或者删除文件（VersionBuilder::MaybeAddFile）
VersionBuilder::MaybeAddFile()根据文件是否在待删除文件集合中来决定f所代表的文件是被添加到vstorage中，还是从vstorage中删除，它实际上藉由VersionBuilder::Rep::MaybeAddFile()来实现。
```
void VersionBuilder::MaybeAddFile(VersionStorageInfo* vstorage, int level,
                                  FileMetaData* f) {
  rep_->MaybeAddFile(vstorage, level, f);
}
```

```
  void VersionBuilder::Rep::MaybeAddFile(VersionStorageInfo* vstorage, int level, FileMetaData* f) {
    if (levels_[level].deleted_files.count(f->fd.GetNumber()) > 0) {
      // f is to-be-delected table file
      /*如果该文件在待删除文件集合中，则从vstorage中删除之*/
      vstorage->RemoveCurrentStats(f);
    } else {
      /*添加到vstorage中*/
      vstorage->AddFile(level, f, info_log_);
    }
  }
```
