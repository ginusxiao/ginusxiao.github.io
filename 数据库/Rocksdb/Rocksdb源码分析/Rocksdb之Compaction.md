# 提纲
[toc]

# 接口

```
  // Compact the underlying storage for the key range [*begin,*end].
  // The actual compaction interval might be superset of [*begin, *end].
  // In particular, deleted and overwritten versions are discarded,
  // and the data is rearranged to reduce the cost of operations
  // needed to access the data.  This operation should typically only
  // be invoked by users who understand the underlying implementation.
  //
  // begin==nullptr is treated as a key before all keys in the database.
  // end==nullptr is treated as a key after all keys in the database.
  // Therefore the following call will compact the entire database:
  //    db->CompactRange(options, nullptr, nullptr);
  // Note that after the entire database is compacted, all data are pushed
  // down to the last level containing any data. If the total data size after
  // compaction is reduced, that level might not be appropriate for hosting all
  // the files. In this case, client could set options.change_level to true, to
  // move the files back to the minimum level capable of holding the data set
  // or a given level (specified by non-negative options.target_level).
  virtual Status CompactRange(const CompactRangeOptions& options,
                              ColumnFamilyHandle* column_family,
                              const Slice* begin, const Slice* end) = 0;
  virtual Status CompactRange(const CompactRangeOptions& options,
                              const Slice* begin, const Slice* end) {
    return CompactRange(options, DefaultColumnFamily(), begin, end);
  }

  ROCKSDB_DEPRECATED_FUNC virtual Status CompactRange(
      ColumnFamilyHandle* column_family, const Slice* begin, const Slice* end,
      bool change_level = false, int target_level = -1,
      uint32_t target_path_id = 0) {
    CompactRangeOptions options;
    options.change_level = change_level;
    options.target_level = target_level;
    options.target_path_id = target_path_id;
    return CompactRange(options, column_family, begin, end);
  }

  ROCKSDB_DEPRECATED_FUNC virtual Status CompactRange(
      const Slice* begin, const Slice* end, bool change_level = false,
      int target_level = -1, uint32_t target_path_id = 0) {
    CompactRangeOptions options;
    options.change_level = change_level;
    options.target_level = target_level;
    options.target_path_id = target_path_id;
    return CompactRange(options, DefaultColumnFamily(), begin, end);
  }
  
  // CompactFiles() inputs a list of files specified by file numbers and
  // compacts them to the specified level. Note that the behavior is different
  // from CompactRange() in that CompactFiles() performs the compaction job
  // using the CURRENT thread.
  //
  // @see GetDataBaseMetaData
  // @see GetColumnFamilyMetaData
  virtual Status CompactFiles(
      const CompactionOptions& compact_options,
      ColumnFamilyHandle* column_family,
      const std::vector<std::string>& input_file_names,
      const int output_level, const int output_path_id = -1) = 0;

  virtual Status CompactFiles(
      const CompactionOptions& compact_options,
      const std::vector<std::string>& input_file_names,
      const int output_level, const int output_path_id = -1) {
    return CompactFiles(compact_options, DefaultColumnFamily(),
                        input_file_names, output_level, output_path_id);
  }  
```
