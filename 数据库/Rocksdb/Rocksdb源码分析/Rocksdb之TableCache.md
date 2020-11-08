# 提纲
[toc]

# 基础数据结构

```
class TableCache {
 public:
  TableCache(const ImmutableCFOptions& ioptions,
             const EnvOptions& storage_options, Cache* cache);
  ~TableCache();

  // Return an iterator for the specified file number (the corresponding
  // file length must be exactly "file_size" bytes).  If "tableptr" is
  // non-nullptr, also sets "*tableptr" to point to the Table object
  // underlying the returned iterator, or nullptr if no Table object underlies
  // the returned iterator.  The returned "*tableptr" object is owned by
  // the cache and should not be deleted, and is valid for as long as the
  // returned iterator is live.
  // @param range_del_agg If non-nullptr, adds range deletions to the
  //    aggregator. If an error occurs, returns it in a NewErrorInternalIterator
  // @param skip_filters Disables loading/accessing the filter block
  // @param level The level this table is at, -1 for "not set / don't know"
  InternalIterator* NewIterator(
      const ReadOptions& options, const EnvOptions& toptions,
      const InternalKeyComparator& internal_comparator,
      const FileDescriptor& file_fd, RangeDelAggregator* range_del_agg,
      TableReader** table_reader_ptr = nullptr,
      HistogramImpl* file_read_hist = nullptr, bool for_compaction = false,
      Arena* arena = nullptr, bool skip_filters = false, int level = -1);

  // If a seek to internal key "k" in specified file finds an entry,
  // call (*handle_result)(arg, found_key, found_value) repeatedly until
  // it returns false.
  // @param get_context State for get operation. If its range_del_agg() returns
  //    non-nullptr, adds range deletions to the aggregator. If an error occurs,
  //    returns non-ok status.
  // @param skip_filters Disables loading/accessing the filter block
  // @param level The level this table is at, -1 for "not set / don't know"
  Status Get(const ReadOptions& options,
             const InternalKeyComparator& internal_comparator,
             const FileDescriptor& file_fd, const Slice& k,
             GetContext* get_context, HistogramImpl* file_read_hist = nullptr,
             bool skip_filters = false, int level = -1);

  // Evict any entry for the specified file number
  static void Evict(Cache* cache, uint64_t file_number);

  // Clean table handle and erase it from the table cache
  // Used in DB close, or the file is not live anymore.
  void EraseHandle(const FileDescriptor& fd, Cache::Handle* handle);

  // Find table reader
  // @param skip_filters Disables loading/accessing the filter block
  // @param level == -1 means not specified
  Status FindTable(const EnvOptions& toptions,
                   const InternalKeyComparator& internal_comparator,
                   const FileDescriptor& file_fd, Cache::Handle**,
                   const bool no_io = false, bool record_read_stats = true,
                   HistogramImpl* file_read_hist = nullptr,
                   bool skip_filters = false, int level = -1,
                   bool prefetch_index_and_filter_in_cache = true);

  // Get TableReader from a cache handle.
  TableReader* GetTableReaderFromHandle(Cache::Handle* handle);

  // Get the table properties of a given table.
  // @no_io: indicates if we should load table to the cache if it is not present
  //         in table cache yet.
  // @returns: `properties` will be reset on success. Please note that we will
  //            return Status::Incomplete() if table is not present in cache and
  //            we set `no_io` to be true.
  Status GetTableProperties(const EnvOptions& toptions,
                            const InternalKeyComparator& internal_comparator,
                            const FileDescriptor& file_meta,
                            std::shared_ptr<const TableProperties>* properties,
                            bool no_io = false);

  // Return total memory usage of the table reader of the file.
  // 0 if table reader of the file is not loaded.
  size_t GetMemoryUsageByTableReader(
      const EnvOptions& toptions,
      const InternalKeyComparator& internal_comparator,
      const FileDescriptor& fd);

  // Release the handle from a cache
  void ReleaseHandle(Cache::Handle* handle);

  // Capacity of the backing Cache that indicates inifinite TableCache capacity.
  // For example when max_open_files is -1 we set the backing Cache to this.
  static const int kInfiniteCapacity = 0x400000;

 private:
  // Build a table reader
  Status GetTableReader(const EnvOptions& env_options,
                        const InternalKeyComparator& internal_comparator,
                        const FileDescriptor& fd, bool sequential_mode,
                        size_t readahead, bool record_read_stats,
                        HistogramImpl* file_read_hist,
                        unique_ptr<TableReader>* table_reader,
                        bool skip_filters = false, int level = -1,
                        bool prefetch_index_and_filter_in_cache = true);

  const ImmutableCFOptions& ioptions_;
  const EnvOptions& env_options_;
  Cache* const cache_;
  std::string row_cache_id_;
};
```


# 实现
## TableCache实际上是一个LRUCache

```
DBImpl::DBImpl(const DBOptions& options, const std::string& dbname) :
    ...... {
  ......
  
  table_cache_ = NewLRUCache(table_cache_size,
                             immutable_db_options_.table_cache_numshardbits);

  versions_.reset(new VersionSet(dbname_, &immutable_db_options_, env_options_,
                                 table_cache_.get(), write_buffer_manager_,
                                 &write_controller_));
                                 
  ......
}
    
VersionSet::VersionSet(const std::string& dbname,
                       const ImmutableDBOptions* db_options,
                       const EnvOptions& storage_options, Cache* table_cache,
                       WriteBufferManager* write_buffer_manager,
                       WriteController* write_controller)
    : column_family_set_(
          new ColumnFamilySet(dbname, db_options, storage_options, table_cache,
                              write_buffer_manager, write_controller)),
      ...... {}    
      
ColumnFamilySet::ColumnFamilySet(const std::string& dbname,
                                 const ImmutableDBOptions* db_options,
                                 const EnvOptions& env_options,
                                 Cache* table_cache,
                                 WriteBufferManager* write_buffer_manager,
                                 WriteController* write_controller)
    : ......
      table_cache_(table_cache),
      ...... {
  ......
}      

ColumnFamilyData* ColumnFamilySet::CreateColumnFamily(
    const std::string& name, uint32_t id, Version* dummy_versions,
    const ColumnFamilyOptions& options) {
  assert(column_families_.find(name) == column_families_.end());
  ColumnFamilyData* new_cfd = new ColumnFamilyData(
      id, name, dummy_versions, table_cache_, write_buffer_manager_, options,
      *db_options_, env_options_, this);
      
  ......
}

ColumnFamilyData::ColumnFamilyData(
    uint32_t id, const std::string& name, Version* _dummy_versions,
    Cache* _table_cache, WriteBufferManager* write_buffer_manager,
    const ColumnFamilyOptions& cf_options, const ImmutableDBOptions& db_options,
    const EnvOptions& env_options, ColumnFamilySet* column_family_set)
    : ...... {
  ......

  // if _dummy_versions is nullptr, then this is a dummy column family.
  if (_dummy_versions != nullptr) {
    internal_stats_.reset(
        new InternalStats(ioptions_.num_levels, db_options.env, this));
    table_cache_.reset(new TableCache(ioptions_, env_options, _table_cache));
    
    ......
  }
  
  RecalculateWriteStallConditions(mutable_cf_options_);
}

TableCache::TableCache(const ImmutableCFOptions& ioptions,
                       const EnvOptions& env_options, Cache* const cache)
    : ioptions_(ioptions), env_options_(env_options), cache_(cache) {
  if (ioptions_.row_cache) {
    // If the same cache is shared by multiple instances, we need to
    // disambiguate its entries.
    PutVarint64(&row_cache_id_, ioptions_.row_cache->NewId());
  }
}
```

通过上面的调用关系可以看到，TableCache::cache_最终通过DBImpl::DBImpl中的NewLRUCache创建而来的，而在“[Rocksdb BlockCache之LRUCache](http://note.youdao.com/noteshare?id=1685d718cd24f92047cdb7aef78b8dd2&sub=5DFD899BE7F240DC925F0C9A1749C28D)”中已经讲述过NewLRUCache这个方法了。也就是说TableCache其实使用的是LRUCache。


## TableCache中缓存的是Table（即SST文件）

```
Status TableCache::FindTable(const EnvOptions& env_options,
                             const InternalKeyComparator& internal_comparator,
                             const FileDescriptor& fd, Cache::Handle** handle,
                             const bool no_io, bool record_read_stats,
                             HistogramImpl* file_read_hist, bool skip_filters,
                             int level,
                             bool prefetch_index_and_filter_in_cache) {
  PERF_TIMER_GUARD(find_table_nanos);
  Status s;
  uint64_t number = fd.GetNumber();
  /* 通过FileNumber产生key，并在TableCache中查找之，返回的handle不是key对应的value，
   * 而是唯一代表该key-value pair的一个句柄，从handle中可以得到value
   */
  Slice key = GetSliceForFileNumber(&number);
  *handle = cache_->Lookup(key);

  /*没找到的情况下，需要插入相应的entry到TableCache中*/
  if (*handle == nullptr) {
    if (no_io) {  // Don't do IO and return a not-found status
      return Status::Incomplete("Table not found in table_cache, no_io is set");
    }
    
    /*为指定的fd创建一个TableReader，并以此作为value*/
    unique_ptr<TableReader> table_reader;
    s = GetTableReader(env_options, internal_comparator, fd,
                       false /* sequential mode */, 0 /* readahead */,
                       record_read_stats, file_read_hist, &table_reader,
                       skip_filters, level, prefetch_index_and_filter_in_cache);
    if (!s.ok()) {
      /*出错的情况*/
      assert(table_reader == nullptr);
      RecordTick(ioptions_.statistics, NO_FILE_ERRORS);
      // We do not cache error results so that if the error is transient,
      // or somebody repairs the file, we recover automatically.
    } else {
      /* 插入key-value pair，key是通过FileNumber得到，value就是前面创建的TableReader，
       * 插入成功则返回一个handle，唯一标识本次插入的key-value pair
       */
      s = cache_->Insert(key, table_reader.get(), 1, &DeleteEntry<TableReader>,
                         handle);
      if (s.ok()) {
        // Release ownership of table reader.
        table_reader.release();
      }
    }
  }
  
  return s;
}

void TableCache::Evict(Cache* cache, uint64_t file_number) {
  /*GetSliceForFileNumber(&file_number)产生key，从TableCache中删除指定的key对应的SST文件*/
  cache->Erase(GetSliceForFileNumber(&file_number));
}
```
通过上面的cache_->Insert和cache_->Erase调用可知，TableCache中存放的是SST文件，其中key是通过SST文件的FileNumber产生的，value则是关于该SST文件的TableReader。

## 相关接口分析
### 辅助函数，获取关于指定FileDescriptor的TableReader

```
Status TableCache::GetTableReader(
    const EnvOptions& env_options,
    const InternalKeyComparator& internal_comparator, const FileDescriptor& fd,
    bool sequential_mode, size_t readahead, bool record_read_stats,
    HistogramImpl* file_read_hist, unique_ptr<TableReader>* table_reader,
    bool skip_filters, int level, bool prefetch_index_and_filter_in_cache) {
  /*获取FileName*/
  std::string fname =
      TableFileName(ioptions_.db_paths, fd.GetNumber(), fd.GetPathId());
  /*创建关于该指定文件的RandomAccessFile*/
  unique_ptr<RandomAccessFile> file;
  Status s = ioptions_.env->NewRandomAccessFile(fname, &file, env_options);

  RecordTick(ioptions_.statistics, NO_FILE_OPENS);
  if (s.ok()) {
    if (readahead > 0) {
      /*如果设置了预读，则创建ReadAheadRandomAccessFile*/
      file = NewReadaheadRandomAccessFile(std::move(file), readahead);
    }
    
    if (!sequential_mode && ioptions_.advise_random_on_open) {
      file->Hint(RandomAccessFile::RANDOM);
    }
    StopWatch sw(ioptions_.env, ioptions_.statistics, TABLE_OPEN_IO_MICROS);
    
    /*为该RandomAccessFile创建RandomAccessFileReader*/
    std::unique_ptr<RandomAccessFileReader> file_reader(
        new RandomAccessFileReader(std::move(file), ioptions_.env,
                                   ioptions_.statistics, record_read_stats,
                                   file_read_hist));
                                   
    /*基于该RandomAccessFileReader创建一个TableReader*/                                 
    s = ioptions_.table_factory->NewTableReader(
        TableReaderOptions(ioptions_, env_options, internal_comparator,
                           skip_filters, level),
        std::move(file_reader), fd.GetFileSize(), table_reader,
        prefetch_index_and_filter_in_cache);
    TEST_SYNC_POINT("TableCache::GetTableReader:0");
  }
  
  return s;
}
```

在构建TableReader对象的时候，用到的是r->ioptions.table_factory->NewTableReader()，这里的Options::table_factory可能是BlockBasedTableFactory，或者CuckooTableFactory，或者PlainTableFactory。默认的是BlockBasedTableFactory：

```
ColumnFamilyOptions::ColumnFamilyOptions()
    : compression(Snappy_Supported() ? kSnappyCompression : kNoCompression),
      table_factory(
          std::shared_ptr<TableFactory>(new BlockBasedTableFactory())) {}
```

但是Rocksdb提供了修改TableFactory的接口：

```
/*设置为BlockBasedTableFactory*/
void rocksdb_options_set_block_based_table_factory(
    rocksdb_options_t *opt,
    rocksdb_block_based_table_options_t* table_options) {
  if (table_options) {
    opt->rep.table_factory.reset(
        rocksdb::NewBlockBasedTableFactory(table_options->rep));
  }
}

/*设置为CuckooTableFactory*/
void rocksdb_options_set_cuckoo_table_factory(
    rocksdb_options_t *opt,
    rocksdb_cuckoo_table_options_t* table_options) {
  if (table_options) {
    opt->rep.table_factory.reset(
        rocksdb::NewCuckooTableFactory(table_options->rep));
  }
}

/*设置为PlainTableFactory*/
void rocksdb_options_set_plain_table_factory(
    rocksdb_options_t *opt, uint32_t user_key_len, int bloom_bits_per_key,
    double hash_table_ratio, size_t index_sparseness) {
  rocksdb::PlainTableOptions options;
  options.user_key_len = user_key_len;
  options.bloom_bits_per_key = bloom_bits_per_key;
  options.hash_table_ratio = hash_table_ratio;
  options.index_sparseness = index_sparseness;

  rocksdb::TableFactory* factory = rocksdb::NewPlainTableFactory(options);
  opt->rep.table_factory.reset(factory);
}
```

那么BlockBasedTableFactory是如何创建TableReader的呢？

```
  // Returns a Table object table that can fetch data from file specified
  // in parameter file. It's the caller's responsibility to make sure
  // file is in the correct format.
  //
  // NewTableReader() is called in three places:
  // (1) TableCache::FindTable() calls the function when table cache miss
  //     and cache the table object returned.
  // (2) SstFileReader (for SST Dump) opens the table and dump the table
  //     contents using the iterator of the table.
  // (3) DBImpl::AddFile() calls this function to read the contents of
  //     the sst file it's attempting to add
  //
  // table_reader_options is a TableReaderOptions which contain all the
  //    needed parameters and configuration to open the table.
  // file is a file handler to handle the file for the table.
  // file_size is the physical file size of the file.
  // table_reader is the output table reader.
Status BlockBasedTableFactory::NewTableReader(
    const TableReaderOptions& table_reader_options,
    unique_ptr<RandomAccessFileReader>&& file, uint64_t file_size,
    unique_ptr<TableReader>* table_reader,
    bool prefetch_index_and_filter_in_cache) const {
  return BlockBasedTable::Open(
      table_reader_options.ioptions, table_reader_options.env_options,
      table_options_, table_reader_options.internal_comparator, std::move(file),
      file_size, table_reader, prefetch_index_and_filter_in_cache,
      table_reader_options.skip_filters, table_reader_options.level);
}
```

通过调用BlockBasedTable::Open来实现。关于BlockBasedTable::Open()请参考“[Rocksdb之BlockBasedTable格式](http://note.youdao.com/noteshare?id=5675fd00f2d247a9e17abe145626328f&sub=58A1188283A04BB1B6C125A57D5D9EF1)”








