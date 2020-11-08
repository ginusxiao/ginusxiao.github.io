# 提纲
[toc]

# 数据结构及其类结构
## 类结构
![image](http://note.youdao.com/noteshare?id=9f22b72e62e56af2b5f7717fc95d3527&sub=AC2EE183CE234EEC9091DBC29616D382)

*注：图片引用自[这里](https://www.jianshu.com/p/9b5ade5212ab)。*

Block:
数据块或者元数据块，SST文件中的一段范围内的数据。Rocksdb中并没有关于Block的数据结构定义，它只是一个概念。
关于Block，可以参考[这里](http://note.youdao.com/noteshare?id=7f049a3df9fded01db83a003a4e4111e&sub=9EE3830978A2414A89D524FDB4A4F143)。

BlockHandle:
用于指示数据块或者元数据块在SST文件中的位置。

BlockBuilder:
用于构造数据块（data block）, 可复用。

IndexBuilder:
用于构建索引块（index block），可复用。

TableBuilder:
接口类，用于构建SST文件（也就是所谓的table）。

BlockBasedTableBuilder:
实现了TableBuilder接口，用于构建BlockBasedTable格式的SST文件。

TableReader:
接口类，用于读取SST文件的抽象类。

BlockBasedTable:
实现了TableReader接口，用于读取BlockBasedTable格式的SST文件。

TableFactory:
工厂类接口, 用来创建指定的TableReader和TableBuilder对象。

BlockBasedTableFactory:
创建BlockBasedTable和BlockBasedTableBuilder。


SstFileWriter:
用于创建SST文件。


## 数据结构
### BlockHandle

```
// BlockHandle is a pointer to the extent of a file that stores a data
// block or a meta block.
class BlockHandle {
 public:
  BlockHandle();
  BlockHandle(uint64_t offset, uint64_t size);

  // The offset of the block in the file.
  uint64_t offset() const { return offset_; }
  void set_offset(uint64_t _offset) { offset_ = _offset; }

  // The size of the stored block
  uint64_t size() const { return size_; }
  void set_size(uint64_t _size) { size_ = _size; }

  void EncodeTo(std::string* dst) const;
  Status DecodeFrom(Slice* input);

  // Return a string that contains the copy of handle.
  std::string ToString(bool hex = true) const;

  // if the block handle's offset and size are both "0", we will view it
  // as a null block handle that points to no where.
  bool IsNull() const {
    return offset_ == 0 && size_ == 0;
  }

  static const BlockHandle& NullBlockHandle() {
    return kNullBlockHandle;
  }

  // Maximum encoding length of a BlockHandle
  enum { kMaxEncodedLength = 10 + 10 };

 private:
  uint64_t offset_;
  uint64_t size_;

  static const BlockHandle kNullBlockHandle;
};
```

### IndexBuilder
IndexBuilder是一个接口类，提供了索引块（index block，是一种元数据块）操作相关的接口。
```
class IndexBuilder {
 public:
  /* 根据指定的index_type创建相应类型的IndexBuilder（比如ShortenedIndexBuilder、
   * HashIndexBuilder和PartitionedIndexBuilder）
   */
  static IndexBuilder* CreateIndexBuilder(
      BlockBasedTableOptions::IndexType index_type,
      const rocksdb::InternalKeyComparator* comparator,
      const InternalKeySliceTransform* int_key_slice_transform,
      const BlockBasedTableOptions& table_opt);

  // Index builder will construct a set of blocks which contain:
  //  1. One primary index block.
  //  2. (Optional) a set of metablocks that contains the metadata of the
  //     primary index.
  /* IndexBuilder可能会创建多个blocks，其中一个primary index block，和可选的多个
   * 包含primary index block的metablocks（即primary index block的元数据块，primary
   * index block也是元数据块）
   */
  struct IndexBlocks {
    Slice index_block_contents;
    std::unordered_map<std::string, Slice> meta_blocks;
  };
  
  explicit IndexBuilder(const InternalKeyComparator* comparator)
      : comparator_(comparator) {}

  virtual ~IndexBuilder() {}

  // Add a new index entry to index block.
  // To allow further optimization, we provide `last_key_in_current_block` and
  // `first_key_in_next_block`, based on which the specific implementation can
  // determine the best index key to be used for the index block.
  // @last_key_in_current_block: this parameter maybe overridden with the value
  //                             "substitute key".
  // @first_key_in_next_block: it will be nullptr if the entry being added is
  //                           the last one in the table
  //
  // REQUIRES: Finish() has not yet been called.
  /*在index block中添加一个索引项（BlockHandle中存放的是该索引项的内容）*/
  virtual void AddIndexEntry(std::string* last_key_in_current_block,
                             const Slice* first_key_in_next_block,
                             const BlockHandle& block_handle) = 0;

  // This method will be called whenever a key is added. The subclasses may
  // override OnKeyAdded() if they need to collect additional information.
  virtual void OnKeyAdded(const Slice& key) {}

  // Inform the index builder that all entries has been written. Block builder
  // may therefore perform any operation required for block finalization.
  //
  // REQUIRES: Finish() has not yet been called.
  inline Status Finish(IndexBlocks* index_blocks) {
    // Throw away the changes to last_partition_block_handle. It has no effect
    // on the first call to Finish anyway.
    BlockHandle last_partition_block_handle;
    return Finish(index_blocks, last_partition_block_handle);
  }

  // This override of Finish can be utilized to build the 2nd level index in
  // PartitionIndexBuilder.
  //
  // index_blocks will be filled with the resulting index data. If the return
  // value is Status::InComplete() then it means that the index is partitioned
  // and the callee should keep calling Finish until Status::OK() is returned.
  // In that case, last_partition_block_handle is pointer to the block written
  // with the result of the last call to Finish. This can be utilized to build
  // the second level index pointing to each block of partitioned indexes. The
  // last call to Finish() that returns Status::OK() populates index_blocks with
  // the 2nd level index content.
  virtual Status Finish(IndexBlocks* index_blocks,
                        const BlockHandle& last_partition_block_handle) = 0;

  // Get the estimated size for index block.
  virtual size_t EstimatedSize() const = 0;

 protected:
  const InternalKeyComparator* comparator_;
};
```

Rocksdb支持kBinarySearch、kHashSearch和kTwoLevelIndexSearch 3种index类型（见BlockBasedTableOptions::IndexType定义），相应的提供了3种IndexBuilder，均继承自IndexBuilder。

#### ShortenedIndexBuilder
ShortenedIndexBuilder用于构建空间节约型的基于二分查找的索引块，对应的BlockBasedTableOptions::IndexType是KBinarySearch。
```
// This index builder builds space-efficient index block.
//
// Optimizations:
//  1. Made block's `block_restart_interval` to be 1, which will avoid linear
//     search when doing index lookup (can be disabled by setting
//     index_block_restart_interval).
//  2. Shorten the key length for index block. Other than honestly using the
//     last key in the data block as the index key, we instead find a shortest
//     substitute key that serves the same function.
class ShortenedIndexBuilder : public IndexBuilder {
 private:
  /*向Index Block中插入也需要借助BlockBuilder*/
  BlockBuilder index_block_builder_;
};
```

#### HashIndexBuilder
HashIndexBuilder用于构建基于hash查找的索引块。
```
// HashIndexBuilder contains a binary-searchable primary index and the
// metadata for secondary hash index construction.
// The metadata for hash index consists two parts:
//  - a metablock that compactly contains a sequence of prefixes. All prefixes
//    are stored consectively without any metadata (like, prefix sizes) being
//    stored, which is kept in the other metablock.
//  - a metablock contains the metadata of the prefixes, including prefix size,
//    restart index and number of block it spans. The format looks like:
//
// +-----------------+---------------------------+---------------------+
// <=prefix 1
// | length: 4 bytes | restart interval: 4 bytes | num-blocks: 4 bytes |
// +-----------------+---------------------------+---------------------+
// <=prefix 2
// | length: 4 bytes | restart interval: 4 bytes | num-blocks: 4 bytes |
// +-----------------+---------------------------+---------------------+
// |                                                                   |
// | ....                                                              |
// |                                                                   |
// +-----------------+---------------------------+---------------------+
// <=prefix n
// | length: 4 bytes | restart interval: 4 bytes | num-blocks: 4 bytes |
// +-----------------+---------------------------+---------------------+
//
// The reason of separating these two metablocks is to enable the efficiently
// reuse the first metablock during hash index construction without unnecessary
// data copy or small heap allocations for prefixes.
class HashIndexBuilder : public IndexBuilder {
 private:
  ShortenedIndexBuilder primary_index_builder_;
  const SliceTransform* hash_key_extractor_;

  // stores a sequence of prefixes
  std::string prefix_block_;
  // stores the metadata of prefixes
  std::string prefix_meta_block_;

  // The following 3 variables keeps unflushed prefix and its metadata.
  // The details of block_num and entry_index can be found in
  // "block_hash_index.{h,cc}"
  uint32_t pending_block_num_ = 0;
  uint32_t pending_entry_index_ = 0;
  std::string pending_entry_prefix_;

  uint64_t current_restart_index_ = 0;
};
```

#### PartitionedIndexBuilder
PartitionedIndexBuilder用于构建两级索引。
```
/**
 * IndexBuilder for two-level indexing. Internally it creates a new index for
 * each partition and Finish then in order when Finish is called on it
 * continiously until Status::OK() is returned.
 *
 * The format on the disk would be I I I I I I IP where I is block containing a
 * partition of indexes built using ShortenedIndexBuilder and IP is a block
 * containing a secondary index on the partitions, built using
 * ShortenedIndexBuilder.
 */
class PartitionedIndexBuilder : public IndexBuilder {
 private:
  struct Entry {
    std::string key;
    std::unique_ptr<ShortenedIndexBuilder> value;
  };
  std::list<Entry> entries_;  // list of partitioned indexes and their keys
  BlockBuilder index_block_builder_;  // top-level index builder
  // the active partition index builder
  ShortenedIndexBuilder* sub_index_builder_;
  // the last key in the active partition index builder
  std::string sub_index_last_key_;
  std::unique_ptr<FlushBlockPolicy> flush_policy_;
  // true if Finish is called once but not complete yet.
  bool finishing_indexes = false;
  const BlockBasedTableOptions& table_opt_;
  // true if it should cut the next filter partition block
  bool cut_filter_block = false;
};
```

### TableBuilder

```
class TableBuilder {
 public:
  // REQUIRES: Either Finish() or Abandon() has been called.
  virtual ~TableBuilder() {}

  // Add key,value to the table being constructed.
  // REQUIRES: key is after any previously added key according to comparator.
  // REQUIRES: Finish(), Abandon() have not been called
  virtual void Add(const Slice& key, const Slice& value) = 0;

  // Return non-ok iff some error has been detected.
  virtual Status status() const = 0;

  // Finish building the table.
  // REQUIRES: Finish(), Abandon() have not been called
  virtual Status Finish() = 0;

  // Indicate that the contents of this builder should be abandoned.
  // If the caller is not going to call Finish(), it must call Abandon()
  // before destroying this builder.
  // REQUIRES: Finish(), Abandon() have not been called
  virtual void Abandon() = 0;

  // Number of calls to Add() so far.
  virtual uint64_t NumEntries() const = 0;

  // Size of the file generated so far.  If invoked after a successful
  // Finish() call, returns the size of the final generated file.
  virtual uint64_t FileSize() const = 0;

  // If the user defined table properties collector suggest the file to
  // be further compacted.
  virtual bool NeedCompact() const { return false; }

  // Returns table properties
  virtual TableProperties GetTableProperties() const = 0;
};
```

### BlockBasedTableBuilder
BlockBasedTableBuilder继承自TableBuilder。
```
class BlockBasedTableBuilder : public TableBuilder {
 private:
  struct Rep;
  class BlockBasedTablePropertiesCollectorFactory;
  class BlockBasedTablePropertiesCollector;
  Rep* rep_;
};
```

BlockBasedTableBuilder相关逻辑实际上藉由BlockBasedTableBuilder::Rep来实现，BlockBasedTableBuilder::Rep通过BlockBasedTableBuilder::Rep::file关联到SST文件。
```
struct BlockBasedTableBuilder::Rep {
  const ImmutableCFOptions ioptions;
  const BlockBasedTableOptions table_options;
  const InternalKeyComparator& internal_comparator;
  /*关联的SST文件*/
  WritableFileWriter* file;
  /*下一个Block在SST文件的写入位置*/
  uint64_t offset = 0;
  Status status;
  /*用于构造待写入的数据块（可以简单理解为内存缓冲区，其中可能包含多个key-value pairs）*/
  BlockBuilder data_block;
  BlockBuilder range_del_block;

  InternalKeySliceTransform internal_prefix_transform;
  /*用于构建索引*/
  std::unique_ptr<IndexBuilder> index_builder;

  /*写入的上一个key-value pair中的user key*/
  std::string last_key;
  const CompressionType compression_type;
  const CompressionOptions compression_opts;
  // Data for presetting the compression library's dictionary, or nullptr.
  const std::string* compression_dict;
  TableProperties props;

  bool closed = false;  // Either Finish() or Abandon() has been called.
  std::unique_ptr<FilterBlockBuilder> filter_builder;
  char compressed_cache_key_prefix[BlockBasedTable::kMaxCacheKeyPrefixSize];
  size_t compressed_cache_key_prefix_size;

  BlockHandle pending_handle;  // Handle to add to index block

  /*压缩后的数据存放在这里*/
  std::string compressed_output;
  std::unique_ptr<FlushBlockPolicy> flush_block_policy;
  /*所关联的ColumnFamily ID和name*/
  uint32_t column_family_id;
  const std::string& column_family_name;

  std::vector<std::unique_ptr<IntTblPropCollector>> table_properties_collectors;
};
```


### TableReader
```
class TableReader {
 public:
  virtual ~TableReader() {}

  // Returns a new iterator over the table contents.
  // The result of NewIterator() is initially invalid (caller must
  // call one of the Seek methods on the iterator before using it).
  // arena: If not null, the arena needs to be used to allocate the Iterator.
  //        When destroying the iterator, the caller will not call "delete"
  //        but Iterator::~Iterator() directly. The destructor needs to destroy
  //        all the states but those allocated in arena.
  // skip_filters: disables checking the bloom filters even if they exist. This
  //               option is effective only for block-based table format.
  virtual InternalIterator* NewIterator(const ReadOptions&,
                                        Arena* arena = nullptr,
                                        const InternalKeyComparator* = nullptr,
                                        bool skip_filters = false) = 0;

  virtual InternalIterator* NewRangeTombstoneIterator(
      const ReadOptions& read_options) {
    return nullptr;
  }

  // Given a key, return an approximate byte offset in the file where
  // the data for that key begins (or would begin if the key were
  // present in the file).  The returned value is in terms of file
  // bytes, and so includes effects like compression of the underlying data.
  // E.g., the approximate offset of the last key in the table will
  // be close to the file length.
  virtual uint64_t ApproximateOffsetOf(const Slice& key) = 0;

  // Set up the table for Compaction. Might change some parameters with
  // posix_fadvise
  virtual void SetupForCompaction() = 0;

  virtual std::shared_ptr<const TableProperties> GetTableProperties() const = 0;

  // Prepare work that can be done before the real Get()
  virtual void Prepare(const Slice& target) {}

  // Report an approximation of how much memory has been used.
  virtual size_t ApproximateMemoryUsage() const = 0;

  // Calls get_context->SaveValue() repeatedly, starting with
  // the entry found after a call to Seek(key), until it returns false.
  // May not make such a call if filter policy says that key is not present.
  //
  // get_context->MarkKeyMayExist needs to be called when it is configured to be
  // memory only and the key is not found in the block cache.
  //
  // readOptions is the options for the read
  // key is the key to search for
  // skip_filters: disables checking the bloom filters even if they exist. This
  //               option is effective only for block-based table format.
  virtual Status Get(const ReadOptions& readOptions, const Slice& key,
                     GetContext* get_context, bool skip_filters = false) = 0;

  // Prefetch data corresponding to a give range of keys
  // Typically this functionality is required for table implementations that
  // persists the data on a non volatile storage medium like disk/SSD
  virtual Status Prefetch(const Slice* begin = nullptr,
                          const Slice* end = nullptr) {
    (void) begin;
    (void) end;
    // Default implementation is NOOP.
    // The child class should implement functionality when applicable
    return Status::OK();
  }

  // convert db file to a human readable form
  virtual Status DumpTable(WritableFile* out_file) {
    return Status::NotSupported("DumpTable() not supported");
  }

  virtual void Close() {}
};
```

### BlockBasedTable
```
class BlockBasedTable : public TableReader {
 public:
  static const std::string kFilterBlockPrefix;
  static const std::string kFullFilterBlockPrefix;
  static const std::string kPartitionedFilterBlockPrefix;
  // The longest prefix of the cache key used to identify blocks.
  // For Posix files the unique ID is three varints.
  static const size_t kMaxCacheKeyPrefixSize = kMaxVarint64Length * 3 + 1;

  // IndexReader is the interface that provide the functionality for index
  // access.
  class IndexReader {
   protected:
    const InternalKeyComparator* icomparator_;

   private:
    Statistics* statistics_;
  };

 protected:
  template <class TValue>
  struct CachableEntry;
  struct Rep;
  Rep* rep_;
};
```

BlockBasedTable实际上借助于BlockBasedTable::Rep来实现自己的逻辑。
```
struct BlockBasedTable::Rep {
  const ImmutableCFOptions& ioptions;
  const EnvOptions& env_options;
  const BlockBasedTableOptions& table_options;
  const FilterPolicy* const filter_policy;
  const InternalKeyComparator& internal_comparator;
  Status status;
  unique_ptr<RandomAccessFileReader> file;
  char cache_key_prefix[kMaxCacheKeyPrefixSize];
  size_t cache_key_prefix_size = 0;
  char persistent_cache_key_prefix[kMaxCacheKeyPrefixSize];
  size_t persistent_cache_key_prefix_size = 0;
  char compressed_cache_key_prefix[kMaxCacheKeyPrefixSize];
  size_t compressed_cache_key_prefix_size = 0;
  uint64_t dummy_index_reader_offset =
      0;  // ID that is unique for the block cache.
  PersistentCacheOptions persistent_cache_options;

  // Footer contains the fixed table information
  Footer footer;
  // index_reader and filter will be populated and used only when
  // options.block_cache is nullptr; otherwise we will get the index block via
  // the block cache.
  unique_ptr<IndexReader> index_reader;
  unique_ptr<FilterBlockReader> filter;

  enum class FilterType {
    kNoFilter,
    kFullFilter,
    kBlockFilter,
    kPartitionedFilter,
  };
  FilterType filter_type;
  BlockHandle filter_handle;

  std::shared_ptr<const TableProperties> table_properties;
  // Block containing the data for the compression dictionary. We take ownership
  // for the entire block struct, even though we only use its Slice member. This
  // is easier because the Slice member depends on the continued existence of
  // another member ("allocation").
  std::unique_ptr<const BlockContents> compression_dict_block;
  BlockBasedTableOptions::IndexType index_type;
  bool hash_index_allow_collision;
  bool whole_key_filtering;
  bool prefix_filtering;
  // TODO(kailiu) It is very ugly to use internal key in table, since table
  // module should not be relying on db module. However to make things easier
  // and compatible with existing code, we introduce a wrapper that allows
  // block to extract prefix without knowing if a key is internal or not.
  unique_ptr<SliceTransform> internal_prefix_transform;

  // only used in level 0 files:
  // when pin_l0_filter_and_index_blocks_in_cache is true, we do use the
  // LRU cache, but we always keep the filter & idndex block's handle checked
  // out here (=we don't call Release()), plus the parsed out objects
  // the LRU cache will never push flush them out, hence they're pinned
  CachableEntry<FilterBlockReader> filter_entry;
  CachableEntry<IndexReader> index_entry;
  // range deletion meta-block is pinned through reader's lifetime when LRU
  // cache is enabled.
  CachableEntry<Block> range_del_entry;
  BlockHandle range_del_handle;

  // If global_seqno is used, all Keys in this file will have the same
  // seqno with value `global_seqno`.
  //
  // A value of kDisableGlobalSequenceNumber means that this feature is disabled
  // and every key have it's own seqno.
  SequenceNumber global_seqno;
};
```

### TableFactory

```
class TableFactory {
 public:
  virtual ~TableFactory() {}

  // The type of the table.
  //
  // The client of this package should switch to a new name whenever
  // the table format implementation changes.
  //
  // Names starting with "rocksdb." are reserved and should not be used
  // by any clients of this package.
  virtual const char* Name() const = 0;

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
  virtual Status NewTableReader(
      const TableReaderOptions& table_reader_options,
      unique_ptr<RandomAccessFileReader>&& file, uint64_t file_size,
      unique_ptr<TableReader>* table_reader,
      bool prefetch_index_and_filter_in_cache = true) const = 0;

  // Return a table builder to write to a file for this table type.
  //
  // It is called in several places:
  // (1) When flushing memtable to a level-0 output file, it creates a table
  //     builder (In DBImpl::WriteLevel0Table(), by calling BuildTable())
  // (2) During compaction, it gets the builder for writing compaction output
  //     files in DBImpl::OpenCompactionOutputFile().
  // (3) When recovering from transaction logs, it creates a table builder to
  //     write to a level-0 output file (In DBImpl::WriteLevel0TableForRecovery,
  //     by calling BuildTable())
  // (4) When running Repairer, it creates a table builder to convert logs to
  //     SST files (In Repairer::ConvertLogToTable() by calling BuildTable())
  //
  // Multiple configured can be accessed from there, including and not limited
  // to compression options. file is a handle of a writable file.
  // It is the caller's responsibility to keep the file open and close the file
  // after closing the table builder. compression_type is the compression type
  // to use in this table.
  virtual TableBuilder* NewTableBuilder(
      const TableBuilderOptions& table_builder_options,
      uint32_t column_family_id, WritableFileWriter* file) const = 0;

  // Sanitizes the specified DB Options and ColumnFamilyOptions.
  //
  // If the function cannot find a way to sanitize the input DB Options,
  // a non-ok Status will be returned.
  virtual Status SanitizeOptions(
      const DBOptions& db_opts,
      const ColumnFamilyOptions& cf_opts) const = 0;

  // Return a string that contains printable format of table configurations.
  // RocksDB prints configurations at DB Open().
  virtual std::string GetPrintableTableOptions() const = 0;

  // Returns the raw pointer of the table options that is used by this
  // TableFactory, or nullptr if this function is not supported.
  // Since the return value is a raw pointer, the TableFactory owns the
  // pointer and the caller should not delete the pointer.
  //
  // In certain case, it is desirable to alter the underlying options when the
  // TableFactory is not used by any open DB by casting the returned pointer
  // to the right class.   For instance, if BlockBasedTableFactory is used,
  // then the pointer can be casted to BlockBasedTableOptions.
  //
  // Note that changing the underlying TableFactory options while the
  // TableFactory is currently used by any open DB is undefined behavior.
  // Developers should use DB::SetOption() instead to dynamically change
  // options while the DB is open.
  virtual void* GetOptions() { return nullptr; }
};
```

### BlockBasedTableFactory

```
class BlockBasedTableFactory : public TableFactory {
 public:
  explicit BlockBasedTableFactory(
      const BlockBasedTableOptions& table_options = BlockBasedTableOptions());

  ~BlockBasedTableFactory() {}

  const char* Name() const override { return "BlockBasedTable"; }

  Status NewTableReader(
      const TableReaderOptions& table_reader_options,
      unique_ptr<RandomAccessFileReader>&& file, uint64_t file_size,
      unique_ptr<TableReader>* table_reader,
      bool prefetch_index_and_filter_in_cache = true) const override;

  TableBuilder* NewTableBuilder(
      const TableBuilderOptions& table_builder_options,
      uint32_t column_family_id, WritableFileWriter* file) const override;

  // Sanitizes the specified DB Options.
  Status SanitizeOptions(const DBOptions& db_opts,
                         const ColumnFamilyOptions& cf_opts) const override;

  std::string GetPrintableTableOptions() const override;

  const BlockBasedTableOptions& table_options() const;

  void* GetOptions() override { return &table_options_; }

 private:
  BlockBasedTableOptions table_options_;
};
```


### BlockBuilder

```
class BlockBuilder {
 private:
  const int          block_restart_interval_;
  const bool         use_delta_encoding_;

  std::string           buffer_;    // Destination buffer
  std::vector<uint32_t> restarts_;  // Restart points
  size_t                estimate_;
  int                   counter_;   // Number of entries emitted since restart
  bool                  finished_;  // Has Finish() been called?
  std::string           last_key_;
};
```

### SstFileWriter
```
class SstFileWriter {
 public:
  // User can pass `column_family` to specify that the the generated file will
  // be ingested into this column_family, note that passing nullptr means that
  // the column_family is unknown.
  // If invalidate_page_cache is set to true, SstFileWriter will give the OS a
  // hint that this file pages is not needed everytime we write 1MB to the file
  SstFileWriter(const EnvOptions& env_options, const Options& options,
                ColumnFamilyHandle* column_family = nullptr,
                bool invalidate_page_cache = true)
      : SstFileWriter(env_options, options, options.comparator, column_family,
                      invalidate_page_cache) {}

  // Deprecated API
  SstFileWriter(const EnvOptions& env_options, const Options& options,
                const Comparator* user_comparator,
                ColumnFamilyHandle* column_family = nullptr,
                bool invalidate_page_cache = true);
                
  ~SstFileWriter();

  // Prepare SstFileWriter to write into file located at "file_path".
  Status Open(const std::string& file_path);

  // Add key, value to currently opened file
  // REQUIRES: key is after any previously added key according to comparator.
  Status Add(const Slice& user_key, const Slice& value);

  // Finalize writing to sst file and close file.
  //
  // An optional ExternalSstFileInfo pointer can be passed to the function
  // which will be populated with information about the created sst file
  Status Finish(ExternalSstFileInfo* file_info = nullptr);

  // Return the current file size.
  uint64_t FileSize();

 private:
  void InvalidatePageCache(bool closing);

  struct Rep;
  Rep* rep_;
};
```

SstFileWriter借助于SstFileWriter::Rep来实现其逻辑。
```
struct SstFileWriter::Rep {
  /*初始化除file_writer和builder以外的域*/
  Rep(const EnvOptions& _env_options, const Options& options,
      const Comparator* _user_comparator, ColumnFamilyHandle* _cfh,
      bool _invalidate_page_cache)
      : env_options(_env_options),
        ioptions(options),
        mutable_cf_options(options),
        internal_comparator(_user_comparator),
        cfh(_cfh),
        invalidate_page_cache(_invalidate_page_cache),
        last_fadvise_size(0) {}

  std::unique_ptr<WritableFileWriter> file_writer;
  std::unique_ptr<TableBuilder> builder;
  EnvOptions env_options;
  ImmutableCFOptions ioptions;
  MutableCFOptions mutable_cf_options;
  InternalKeyComparator internal_comparator;
  ExternalSstFileInfo file_info;
  InternalKey ikey;
  std::string column_family_name;
  ColumnFamilyHandle* cfh;
  // If true, We will give the OS a hint that this file pages is not needed
  // everytime we write 1MB to the file
  bool invalidate_page_cache;
  // the size of the file during the last time we called Fadvise to remove
  // cached pages from page cache.
  uint64_t last_fadvise_size;
};
```

# 实现
## SstFileWriter
### SstFileWriter构造函数（SstFileWriter::SstFileWriter）
在SstFileWriter构造函数很简单。
```
SstFileWriter::SstFileWriter(const EnvOptions& env_options,
                             const Options& options,
                             const Comparator* user_comparator,
                             ColumnFamilyHandle* column_family,
                             bool invalidate_page_cache)
    : rep_(new Rep(env_options, options, user_comparator, column_family,
                   invalidate_page_cache)) {
  rep_->file_info.file_size = 0;
}
```

### 关联SstFileWriter到指定SST文件（SstFileWriter::Open）
SstFileWriter::Open()关联SstFileWriter到指定的SST文件。分为以下步骤：

- 基于指定的文件名创建一个WritableFile；

- 基于创建的WritableFile构建WritableFileWriter对象，并设置给rep_->file_writer；

- 构建TableBuilder对象，并设置给rep_->builder；

- 初始化rep_->file_info；

```
Status SstFileWriter::Open(const std::string& file_path) {
  Rep* r = rep_;
  Status s;
  std::unique_ptr<WritableFile> sst_file;
  s = r->ioptions.env->NewWritableFile(file_path, &sst_file, r->env_options);
  if (!s.ok()) {
    return s;
  }

  CompressionType compression_type;
  if (r->ioptions.bottommost_compression != kDisableCompressionOption) {
    compression_type = r->ioptions.bottommost_compression;
  } else if (!r->ioptions.compression_per_level.empty()) {
    // Use the compression of the last level if we have per level compression
    compression_type = *(r->ioptions.compression_per_level.rbegin());
  } else {
    compression_type = r->mutable_cf_options.compression;
  }

  std::vector<std::unique_ptr<IntTblPropCollectorFactory>>
      int_tbl_prop_collector_factories;

  // SstFileWriter properties collector to add SstFileWriter version.
  int_tbl_prop_collector_factories.emplace_back(
      new SstFileWriterPropertiesCollectorFactory(2 /* version */,
                                                  0 /* global_seqno*/));

  // User collector factories
  auto user_collector_factories =
      r->ioptions.table_properties_collector_factories;
  for (size_t i = 0; i < user_collector_factories.size(); i++) {
    int_tbl_prop_collector_factories.emplace_back(
        new UserKeyTablePropertiesCollectorFactory(
            user_collector_factories[i]));
  }
  int unknown_level = -1;
  uint32_t cf_id;

  if (r->cfh != nullptr) {
    // user explicitly specified that this file will be ingested into cfh,
    // we can persist this information in the file.
    cf_id = r->cfh->GetID();
    r->column_family_name = r->cfh->GetName();
  } else {
    r->column_family_name = "";
    cf_id = TablePropertiesCollectorFactory::Context::kUnknownColumnFamily;
  }

  TableBuilderOptions table_builder_options(
      r->ioptions, r->internal_comparator, &int_tbl_prop_collector_factories,
      compression_type, r->ioptions.compression_opts,
      nullptr /* compression_dict */, false /* skip_filters */,
      r->column_family_name, unknown_level);
  r->file_writer.reset(
      new WritableFileWriter(std::move(sst_file), r->env_options));

  // TODO(tec) : If table_factory is using compressed block cache, we will
  // be adding the external sst file blocks into it, which is wasteful.
  r->builder.reset(r->ioptions.table_factory->NewTableBuilder(
      table_builder_options, cf_id, r->file_writer.get()));

  r->file_info.file_path = file_path;
  r->file_info.file_size = 0;
  r->file_info.num_entries = 0;
  r->file_info.sequence_number = 0;
  r->file_info.version = 2;
  return s;
}
```

在构建TableBuilder对象的时候，用到的是r->ioptions.table_factory->NewTableBuilder()，这里的Options::table_factory可能是BlockBasedTableFactory，或者CuckooTableFactory，或者PlainTableFactory。默认的是BlockBasedTableFactory：

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

当SstFileWriter关联到指定SST文件之后，SstFileWriter就可以向这个SST文件写入了。

### 向SstFileWriter关联的SST文件中添加记录（SstFileWriter::Add）
```
Status SstFileWriter::Add(const Slice& user_key, const Slice& value) {
  Rep* r = rep_;
  if (!r->builder) {
    return Status::InvalidArgument("File is not opened");
  }

  if (r->file_info.num_entries == 0) {
    /*关于该SstFileWriter的第一次添加记录，设置smallest key*/
    r->file_info.smallest_key.assign(user_key.data(), user_key.size());
  } else {
    /*必须确保keys按序添加*/
    if (r->internal_comparator.user_comparator()->Compare(
            user_key, r->file_info.largest_key) <= 0) {
      // Make sure that keys are added in order
      return Status::InvalidArgument("Keys must be added in order");
    }
  }

  // TODO(tec) : For external SST files we could omit the seqno and type.
  /*根据user_key生成InternalKey（其中包含user key，sequence number，value type）*/
  r->ikey.Set(user_key, 0 /* Sequence Number */,
              ValueType::kTypeValue /* Put */);
  /*借助TableBuilder来添加记录到SST文件中*/
  r->builder->Add(r->ikey.Encode(), value);

  // update file info
  /*更新SST文件信息*/
  r->file_info.num_entries++;
  r->file_info.largest_key.assign(user_key.data(), user_key.size());
  r->file_info.file_size = r->builder->FileSize();

  InvalidatePageCache(false /* closing */);

  return Status::OK();
}
```

SstFileWriter::InvalidatePageCache()用于告诉OS无需缓存该SST文件的数据。
```
void SstFileWriter::InvalidatePageCache(bool closing) {
  Rep* r = rep_;
  if (r->invalidate_page_cache == false) {
    // Fadvise disabled
    return;
  }

  /* 如果自从上次fadvise以来，写入的数据总量超过kFadviseTrigger，或者即将关闭
   * 该SST文件，则告诉OS无需缓存该SST文件的数据
   */
  uint64_t bytes_since_last_fadvise =
      r->builder->FileSize() - r->last_fadvise_size;
  if (bytes_since_last_fadvise > kFadviseTrigger || closing) {
    TEST_SYNC_POINT_CALLBACK("SstFileWriter::InvalidatePageCache",
                             &(bytes_since_last_fadvise));
    // Tell the OS that we dont need this file in page cache
    r->file_writer->InvalidateCache(0, 0);
    r->last_fadvise_size = r->builder->FileSize();
  }
}
```

### 停止向SstFileWriter关联的当前的SST文件中写入并关闭当前的SST文件（SstFileWriter::Finish）
```
Status SstFileWriter::Finish(ExternalSstFileInfo* file_info) {
  Rep* r = rep_;
  if (!r->builder) {
    return Status::InvalidArgument("File is not opened");
  }
  if (r->file_info.num_entries == 0) {
    return Status::InvalidArgument("Cannot create sst file with no entries");
  }

  Status s = r->builder->Finish();
  r->file_info.file_size = r->builder->FileSize();

  if (s.ok()) {
    s = r->file_writer->Sync(r->ioptions.use_fsync);
    /*即将关闭该SST文件，告诉OS无需缓存该文件的数据*/
    InvalidatePageCache(true /* closing */);
    if (s.ok()) {
      /*关闭该SST文件*/
      s = r->file_writer->Close();
    }
  }
  if (!s.ok()) {
    r->ioptions.env->DeleteFile(r->file_info.file_path);
  }

  if (file_info != nullptr) {
    *file_info = r->file_info;
  }

  /* 重置TableBuilder，因为SstFileWriter会被复用，在复用的时候会指定新的
   * TableBuilder（见SstFileWriter::Open）
   */
  r->builder.reset();
  return s;
}
```

### 获取SstFileWriter关联的当前SST文件大小（SstFileWriter::FileSize）
```
uint64_t SstFileWriter::FileSize() {
  return rep_->file_info.file_size;
}
```

## BlockBasedTableBuilder
### BlockBasedTableBuilder构造函数
```
BlockBasedTableBuilder::BlockBasedTableBuilder(
    const ImmutableCFOptions& ioptions,
    const BlockBasedTableOptions& table_options,
    const InternalKeyComparator& internal_comparator,
    const std::vector<std::unique_ptr<IntTblPropCollectorFactory>>*
        int_tbl_prop_collector_factories,
    uint32_t column_family_id, WritableFileWriter* file,
    const CompressionType compression_type,
    const CompressionOptions& compression_opts,
    const std::string* compression_dict, const bool skip_filters,
    const std::string& column_family_name) {
  BlockBasedTableOptions sanitized_table_options(table_options);
  if (sanitized_table_options.format_version == 0 &&
      sanitized_table_options.checksum != kCRC32c) {
    ROCKS_LOG_WARN(
        ioptions.info_log,
        "Silently converting format_version to 1 because checksum is "
        "non-default");
    // silently convert format_version to 1 to keep consistent with current
    // behavior
    sanitized_table_options.format_version = 1;
  }

  /*创建BlockBasedTableBuilder::Rep对象*/
  rep_ = new Rep(ioptions, sanitized_table_options, internal_comparator,
                 int_tbl_prop_collector_factories, column_family_id, file,
                 compression_type, compression_opts, compression_dict,
                 skip_filters, column_family_name);

  if (rep_->filter_builder != nullptr) {
    rep_->filter_builder->StartBlock(0);
  }
  if (table_options.block_cache_compressed.get() != nullptr) {
    BlockBasedTable::GenerateCachePrefix(
        table_options.block_cache_compressed.get(), file->writable_file(),
        &rep_->compressed_cache_key_prefix[0],
        &rep_->compressed_cache_key_prefix_size);
  }
}
```

在BlockBasedTableBuilder::Rep构造函数中会关联SST文件，创建data Block相关的Block builder，创建范围删除相关的Block builder，创建filter Block相关的Block builder，创建index Block相关的Block builder。
```
  BlockBasedTableBuilder::Rep(const ImmutableCFOptions& _ioptions,
      const BlockBasedTableOptions& table_opt,
      const InternalKeyComparator& icomparator,
      const std::vector<std::unique_ptr<IntTblPropCollectorFactory>>*
          int_tbl_prop_collector_factories,
      uint32_t _column_family_id, WritableFileWriter* f,
      const CompressionType _compression_type,
      const CompressionOptions& _compression_opts,
      const std::string* _compression_dict, const bool skip_filters,
      const std::string& _column_family_name)
      : ioptions(_ioptions),
        table_options(table_opt),
        internal_comparator(icomparator),
        /*关联的SST文件*/
        file(f),
        /*data Block 相关的Block builder*/
        data_block(table_options.block_restart_interval,
                   table_options.use_delta_encoding),
        /*范围删除相关的Block builder*/
        range_del_block(1),  // TODO(andrewkr): restart_interval unnecessary
        internal_prefix_transform(_ioptions.prefix_extractor),
        compression_type(_compression_type),
        compression_opts(_compression_opts),
        compression_dict(_compression_dict),
        flush_block_policy(
            table_options.flush_block_policy_factory->NewFlushBlockPolicy(
                table_options, data_block)),
        column_family_id(_column_family_id),
        column_family_name(_column_family_name) {
    PartitionedIndexBuilder* p_index_builder = nullptr;
    /*根据index_type来设定相应的index Block相关的Block builder*/
    if (table_options.index_type ==
        BlockBasedTableOptions::kTwoLevelIndexSearch) {
      p_index_builder = PartitionedIndexBuilder::CreateIndexBuilder(
          &internal_comparator, table_options);
      index_builder.reset(p_index_builder);
    } else {
      index_builder.reset(IndexBuilder::CreateIndexBuilder(
          table_options.index_type, &internal_comparator,
          &this->internal_prefix_transform, table_options));
    }
    
    /*设定Filter Block相关的Block builder*/
    if (skip_filters) {
      filter_builder = nullptr;
    } else {
      filter_builder.reset(
          CreateFilterBlockBuilder(_ioptions, table_options, p_index_builder));
    }

    for (auto& collector_factories : *int_tbl_prop_collector_factories) {
      table_properties_collectors.emplace_back(
          collector_factories->CreateIntTblPropCollector(column_family_id));
    }
    table_properties_collectors.emplace_back(
        new BlockBasedTablePropertiesCollector(
            table_options.index_type, table_options.whole_key_filtering,
            _ioptions.prefix_extractor != nullptr));
  }
```


### 添加一个key-value pair到BlockBasedTable中（BlockBasedTableBuilder::Add）
```
void BlockBasedTableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  ValueType value_type = ExtractValueType(key);
  if (IsValueType(value_type)) {
    if (r->props.num_entries > 0) {
      /*确保是有序添加，r->last_key中记录的是上一个添加的user key*/
      assert(r->internal_comparator.Compare(key, Slice(r->last_key)) > 0);
    }

    /*检查是否需要Flush当前的Block*/
    auto should_flush = r->flush_block_policy->Update(key, value);
    if (should_flush) {
      assert(!r->data_block.empty());
      /*将r->data_block(类型为BlockBuilder)中缓冲的数据写入SST文件*/
      Flush();

      // Add item to index block.
      // We do not emit the index entry for a block until we have seen the
      // first key for the next data block.  This allows us to use shorter
      // keys in the index block.  For example, consider a block boundary
      // between the keys "the quick brown fox" and "the who".  We can use
      // "the r" as the key for the index block entry since it is >= all
      // entries in the first block and < all entries in subsequent
      // blocks.
      if (ok()) {
        /* 在IndexBlock中插入一个关于当前Block（r->pending_handle所代表）的新的
         * IndexEntry，r->last_key表示当前Block的最后一个key，key表示当前带插入
         * 的索引项的key，AddIndexEntry需要在[r->last_key, key)之间找一个key作为
         * 当前Block的索引项的key，r->pending_handle则作为当前Block的索引项的内容
         */
        r->index_builder->AddIndexEntry(&r->last_key, &key, r->pending_handle);
      }
    }

    // Note: PartitionedFilterBlockBuilder requires key being added to filter
    // builder after being added to index builder.
    if (r->filter_builder != nullptr) {
      r->filter_builder->Add(ExtractUserKey(key));
    }

    /*更新r->last_key*/
    r->last_key.assign(key.data(), key.size());
    /*插入当前的key-value pair*/
    r->data_block.Add(key, value);
    r->props.num_entries++;
    r->props.raw_key_size += key.size();
    r->props.raw_value_size += value.size();

    /*通知IndexBuilder key已经成功加入，对于ShortenedIndexBuilder来说什么都不做*/
    r->index_builder->OnKeyAdded(key);
    NotifyCollectTableCollectorsOnAdd(key, value, r->offset,
                                      r->table_properties_collectors,
                                      r->ioptions.info_log);

  } else if (value_type == kTypeRangeDeletion) {
    // TODO(wanning&andrewkr) add num_tomestone to table properties
    /*区间删除*/
    r->range_del_block.Add(key, value);
    ++r->props.num_entries;
    r->props.raw_key_size += key.size();
    r->props.raw_value_size += value.size();
    NotifyCollectTableCollectorsOnAdd(key, value, r->offset,
                                      r->table_properties_collectors,
                                      r->ioptions.info_log);
  } else {
    assert(false);
  }
}
```

BlockBasedTableBuilder::Flush()将当前data Block（BlockBasedTableBuilder::Rep::data_block，作为待写入文件中的数据的缓冲区）中的内容写入文件。
```
void BlockBasedTableBuilder::Flush() {
  Rep* r = rep_;
  assert(!r->closed);
  if (!ok()) return;
  if (r->data_block.empty()) return;
  /* 将r->data_block中缓冲的数据写入SST文件中，r->pending_handle中记录的是关于当前
   * 被写入的Block的BlockHandle，r->pending_handle将被用于更新index Block
   */
  WriteBlock(&r->data_block, &r->pending_handle, true /* is_data_block */);
  if (r->filter_builder != nullptr) {
    r->filter_builder->StartBlock(r->offset);
  }
  r->props.data_size = r->offset;
  ++r->props.num_data_blocks;
}
```

BlockBasedTableBuilder::WriteBlock()将停止BlockBuilder向关联的当前Block中写入数据，并将BlockBuilder中关联的当前Block中的数据写入SST文件，最后重置当前Block（实现上是通过重置BlockBuilder）。
```
void BlockBasedTableBuilder::WriteBlock(BlockBuilder* block,
                                        BlockHandle* handle,
                                        bool is_data_block) {
  /*BlockBuilder停止向当前Block中写入，并将该Block中的数据写入SST文件*/                                        
  WriteBlock(block->Finish(), handle, is_data_block);
  /*重置该BlockBuilder，以便为下一个Block复用*/
  block->Reset();
}
```

BlockBasedTableBuilder::WriteBlock()中首先尝试对将要写入SST文件的数据进行压缩，然后再写入SST文件。
```
void BlockBasedTableBuilder::WriteBlock(const Slice& raw_block_contents,
                                        BlockHandle* handle,
                                        bool is_data_block) {
  // File format contains a sequence of blocks where each block has:
  //    block_data: uint8[n]
  //    type: uint8
  //    crc: uint32
  assert(ok());
  Rep* r = rep_;

  auto type = r->compression_type;
  Slice block_contents;
  bool abort_compression = false;

  StopWatchNano timer(r->ioptions.env,
    ShouldReportDetailedTime(r->ioptions.env, r->ioptions.statistics));

  if (raw_block_contents.size() < kCompressionSizeLimit) {
    /*对raw_block_contents中的数据执行压缩*/
    Slice compression_dict;
    if (is_data_block && r->compression_dict && r->compression_dict->size()) {
      compression_dict = *r->compression_dict;
    }

    /* 根据指定的压缩选项，压缩类型，format version（不同于SST文件格式），压缩字典
     * 将raw_block_contents压缩成r->compressed_output
     *
     * 关于format version，请参考BlockBasedTableOptions::format_version的注释
     */
    block_contents = CompressBlock(raw_block_contents, r->compression_opts,
                                   &type, r->table_options.format_version,
                                   compression_dict, &r->compressed_output);

    // Some of the compression algorithms are known to be unreliable. If
    // the verify_compression flag is set then try to de-compress the
    // compressed data and compare to the input.
    /* 某些压缩算法并不可靠，如果设置了verify_compression选项，则尝试解压缩数据并和
     * 压缩前的输入数据进行比较，以验证压缩的正确性，在此不表
     */
    if (type != kNoCompression && r->table_options.verify_compression) {
      // Retrieve the uncompressed contents into a new buffer
      BlockContents contents;
      Status stat = UncompressBlockContentsForCompressionType(
          block_contents.data(), block_contents.size(), &contents,
          r->table_options.format_version, compression_dict, type,
          r->ioptions);

      if (stat.ok()) {
        bool compressed_ok = contents.data.compare(raw_block_contents) == 0;
        if (!compressed_ok) {
          // The result of the compression was invalid. abort.
          abort_compression = true;
          ROCKS_LOG_ERROR(r->ioptions.info_log,
                          "Decompressed block did not match raw block");
          r->status =
              Status::Corruption("Decompressed block did not match raw block");
        }
      } else {
        // Decompression reported an error. abort.
        r->status = Status::Corruption("Could not decompress");
        abort_compression = true;
      }
    }
  } else {
    // Block is too big to be compressed.
    abort_compression = true;
  }

  // Abort compression if the block is too big, or did not pass
  // verification.
  if (abort_compression) {
    /*处理未压缩或者压缩失败的情况*/
    RecordTick(r->ioptions.statistics, NUMBER_BLOCK_NOT_COMPRESSED);
    type = kNoCompression;
    block_contents = raw_block_contents;
  } else if (type != kNoCompression &&
             ShouldReportDetailedTime(r->ioptions.env,
                                      r->ioptions.statistics)) {
    MeasureTime(r->ioptions.statistics, COMPRESSION_TIMES_NANOS,
                timer.ElapsedNanos());
    MeasureTime(r->ioptions.statistics, BYTES_COMPRESSED,
                raw_block_contents.size());
    RecordTick(r->ioptions.statistics, NUMBER_BLOCK_COMPRESSED);
  }

  /*真正将数据（可能被压缩了）写入SST文件*/
  WriteRawBlock(block_contents, type, handle);
  /*清空压缩数据输出缓冲区（压缩后的数据存放在这里）*/
  r->compressed_output.clear();
}
```

BlockBasedTableBuilder::WriteRawBlock()依次向SST文件中写入Block data、type和crc信息，并记录当前Block在SST文件中的位置信息（记录到BlockHandle中），最后将Block data插入到压缩的Block Cache中，并更新SST文件的写入偏移。
```
void BlockBasedTableBuilder::WriteRawBlock(const Slice& block_contents,
                                           CompressionType type,
                                           BlockHandle* handle) {
  Rep* r = rep_;
  StopWatch sw(r->ioptions.env, r->ioptions.statistics, WRITE_RAW_BLOCK_MICROS);
  /*设置本次写入的Block的BlockHandle（指向Block存放的位置信息）*/
  handle->set_offset(r->offset);
  handle->set_size(block_contents.size());
  /*在SST文件中追加写Block的数据部分*/
  r->status = r->file->Append(block_contents);
  if (r->status.ok()) {
    /*紧接着数据部分写入trailer（包括type和校验和）*/
    char trailer[kBlockTrailerSize];
    /*trailer中第一个字节存放type信息*/
    trailer[0] = type;
    /*根据指定的校验和的类型，计算校验和并编码到trailer中*/
    char* trailer_without_type = trailer + 1;
    switch (r->table_options.checksum) {
      case kNoChecksum:
        // we don't support no checksum yet
        assert(false);
        // intentional fallthrough
      case kCRC32c: {
        auto crc = crc32c::Value(block_contents.data(), block_contents.size());
        crc = crc32c::Extend(crc, trailer, 1);  // Extend to cover block type
        EncodeFixed32(trailer_without_type, crc32c::Mask(crc));
        break;
      }
      case kxxHash: {
        void* xxh = XXH32_init(0);
        XXH32_update(xxh, block_contents.data(),
                     static_cast<uint32_t>(block_contents.size()));
        XXH32_update(xxh, trailer, 1);  // Extend  to cover block type
        EncodeFixed32(trailer_without_type, XXH32_digest(xxh));
        break;
      }
    }

    /*追加写入trailer*/
    r->status = r->file->Append(Slice(trailer, kBlockTrailerSize));
    if (r->status.ok()) {
      /*插入Block内容到压缩的Block Cache中*/
      r->status = InsertBlockInCache(block_contents, type, handle);
    }
    if (r->status.ok()) {
      /*更新SST文件的写入偏移（下一个写入位置）*/
      r->offset += block_contents.size() + kBlockTrailerSize;
    }
  }
}
```

### 放弃当前TableBuilder构建的SST文件（BlockBasedTableBuilder::Abandon）
BlockBasedTableBuilder::Abandon放弃当前TableBuilder相关的数据（包括当前Block中尚未Flush到SST文件中的数据和已经Flush到SST文件中的数据）
```
void BlockBasedTableBuilder::Abandon() {
  Rep* r = rep_;
  assert(!r->closed);
  r->closed = true;
}
```

### 完成构建关联的SST文件（BlockBasedTableBuilder::Finish）
BlockBasedTableBuilder::Finish()停止向SST文件中写入并完成对当前SST文件的构建，包括如下步骤：
- 如果当前有尚未Flush的Block，则将其写入SST文件中，并且为之构建索引项，添加到index Block中；

- 通知IndexBuilder结束构建index Block，构建的index Block包含两部分：primary index block和primary index block的元数据；

- 写入[meta block: filter]到SST文件中；

- 写入[meta block: metadata of primary index block]，即index Block中primary index block的元数据部分；

- 写入[meta block: property]到SST文件中；

- 写入[meta block: compression dictionary]到SST文件中；

- 写入[meta block: range deletion]到SST文件中；

- 依次按照如下顺序构建metaindex Block：
	
	- 在metaindex Block中添加关于[meta block: metadata of primary index block]的索引项；
	
	- 在metaindex Block中添加关于[meta block: filter]的索引项，key为<filter_block_prefix>.Name，value为[meta block: filter]的位置；
	
	- 在metaindex Block中添加关于[meta block: property]的索引项，key为kPropertiesBlock，value为[meta block: property]的位置；
	
	- 在metaindex Block中添加关于[meta block: compression dictionary]的索引项，key为kCompressionDictBlock，value为[meta block: compression dictionary]的位置；
	
	- 在metaindex Block中添加关于[meta block: range deletion]的索引项，key为kRangeDelBlock，value为[meta block: range deletion]的位置；

- 将metaindex Block写入到SST文件中；

- 将index Block（index_block.index_block_contents，准确说是dataindex Block）写入到SST文件中；

- 写入footer信息到SST文件中；

```
Status BlockBasedTableBuilder::Finish() {
  Rep* r = rep_;
  bool empty_data_block = r->data_block.empty();
  /*将当前尚未Flush的Block写入SST文件中*/
  Flush();
  assert(!r->closed);
  r->closed = true;

  // To make sure properties block is able to keep the accurate size of index
  // block, we will finish writing all index entries here and flush them
  // to storage after metaindex block is written.
  /* 为刚刚Flush的data Block构建索引项（添加索引到index Block中），其中r->pending_handle
   * 就表示刚刚Flush的data Block对应的BlockHandle，它会作为索引项的value部分
   */
  if (ok() && !empty_data_block) {
    r->index_builder->AddIndexEntry(
        &r->last_key, nullptr /* no next data block */, r->pending_handle);
  }

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle,
      compression_dict_block_handle, range_del_block_handle;
      
  // Write meta blocks and metaindex block with the following order.
  //    1. [meta block: filter]
  //    2. [meta block: properties]
  //    3. [meta block: compression dictionary]
  //    4. [meta block: range deletion tombstone]
  //    5. [metaindex block]
  
  // Write filter block
  /*写入[meta block: filter]到SST文件中*/
  if (ok() && r->filter_builder != nullptr) {
    Status s = Status::Incomplete();
    while (s.IsIncomplete()) {
      Slice filter_content = r->filter_builder->Finish(filter_block_handle, &s);
      assert(s.ok() || s.IsIncomplete());
      r->props.filter_size += filter_content.size();
      WriteRawBlock(filter_content, kNoCompression, &filter_block_handle);
    }
  }

  /* 通知IndexBuilder结束当前的index Block，index Block相关的内容保存在index_blocks中，
   * 当前只有PartitionedIndexBuilder可能返回InComplete status
   *
   * index_blocks::index_block_contents：data blocks相关的index block（也称为primary index block）
   * index_blocks::meta_blocks：包含primary index block的元数据的元数据块
   */
  IndexBuilder::IndexBlocks index_blocks;
  auto index_builder_status = r->index_builder->Finish(&index_blocks);
  if (index_builder_status.IsIncomplete()) {
    // We we have more than one index partition then meta_blocks are not
    // supported for the index. Currently meta_blocks are used only by
    // HashIndexBuilder which is not multi-partition.
    assert(index_blocks.meta_blocks.empty());
  } else if (!index_builder_status.ok()) {
    return index_builder_status;
  }


  // write meta blocks
  /* 写入index_blocks::meta_blocks，并在metaindex Block中添加关于index_blocks::meta_blocks
   * 的索引信息（借助于meta_index_builder.Add来添加）
   */
  MetaIndexBuilder meta_index_builder;
  for (const auto& item : index_blocks.meta_blocks) {
    BlockHandle block_handle;
    /*写入iter.second所对应的meta block到SST文件中，在block_handle中返回写入的位置信息*/
    WriteBlock(item.second, &block_handle, false /* is_data_block */);
    /*在metaindex Block中为iter.second所对应的meta block添加一个索引项*/
    meta_index_builder.Add(item.first, block_handle);
  }

  if (ok()) {
    if (r->filter_builder != nullptr) {
      // Add mapping from "<filter_block_prefix>.Name" to location
      // of filter data.
      /* 在metaindex Block中添加关于[meta block: filter]的索引项，key为<filter_block_prefix>.Name，
       * value为[meta block: filter]的位置
       */
      std::string key;
      if (r->filter_builder->IsBlockBased()) {
        key = BlockBasedTable::kFilterBlockPrefix;
      } else {
        key = r->table_options.partition_filters
                  ? BlockBasedTable::kPartitionedFilterBlockPrefix
                  : BlockBasedTable::kFullFilterBlockPrefix;
      }
      key.append(r->table_options.filter_policy->Name());
      meta_index_builder.Add(key, filter_block_handle);
    }

    // Write properties and compression dictionary blocks.
    {
      PropertyBlockBuilder property_block_builder;
      r->props.column_family_id = r->column_family_id;
      r->props.column_family_name = r->column_family_name;
      r->props.filter_policy_name = r->table_options.filter_policy != nullptr ?
          r->table_options.filter_policy->Name() : "";
      r->props.index_size =
          r->index_builder->EstimatedSize() + kBlockTrailerSize;
      r->props.comparator_name = r->ioptions.user_comparator != nullptr
                                     ? r->ioptions.user_comparator->Name()
                                     : "nullptr";
      r->props.merge_operator_name = r->ioptions.merge_operator != nullptr
                                         ? r->ioptions.merge_operator->Name()
                                         : "nullptr";
      r->props.compression_name = CompressionTypeToString(r->compression_type);
      r->props.prefix_extractor_name =
          r->ioptions.prefix_extractor != nullptr
              ? r->ioptions.prefix_extractor->Name()
              : "nullptr";

      std::string property_collectors_names = "[";
      property_collectors_names = "[";
      for (size_t i = 0;
           i < r->ioptions.table_properties_collector_factories.size(); ++i) {
        if (i != 0) {
          property_collectors_names += ",";
        }
        property_collectors_names +=
            r->ioptions.table_properties_collector_factories[i]->Name();
      }
      property_collectors_names += "]";
      r->props.property_collectors_names = property_collectors_names;

      // Add basic properties
      /*property Block builder构建属性块（[meta block: property]）*/
      property_block_builder.AddTableProperty(r->props);

      // Add use collected properties
      NotifyCollectTableCollectorsOnFinish(r->table_properties_collectors,
                                           r->ioptions.info_log,
                                           &property_block_builder);

      BlockHandle properties_block_handle;
      /*写入[meta block: property]到SST文件中*/
      WriteRawBlock(
          property_block_builder.Finish(),
          kNoCompression,
          &properties_block_handle
      );
      /* 在metaindex Block中添加关于[meta block: property]的索引项，key为kPropertiesBlock，
       * value为[meta block: property]的位置
       */
      meta_index_builder.Add(kPropertiesBlock, properties_block_handle);

      // Write compression dictionary block
      if (r->compression_dict && r->compression_dict->size()) {
        /*写入[meta block: compression dictionary]到SST文件中*/
        WriteRawBlock(*r->compression_dict, kNoCompression,
                      &compression_dict_block_handle);
        /* 在metaindex Block中添加关于[meta block: compression dictionary]的索引项，
         * key为kCompressionDictBlock，value为[meta block: compression dictionary]的位置
         */                        
        meta_index_builder.Add(kCompressionDictBlock,
                               compression_dict_block_handle);
      }
    }  // end of properties/compression dictionary block writing

    if (ok() && !r->range_del_block.empty()) {
      /*写入[meta block: range deletion]到SST文件中*/
      WriteRawBlock(r->range_del_block.Finish(), kNoCompression,
                    &range_del_block_handle);
      /* 在metaindex Block中添加关于[meta block: range deletion]的索引项，
       * key为kRangeDelBlock，value为[meta block: range deletion]的位置
       */                     
      meta_index_builder.Add(kRangeDelBlock, range_del_block_handle);
    }  // range deletion tombstone meta block
  }    // meta blocks

  // Write index block
  if (ok()) {
    // flush the meta index block
    /*将metaindex Block写入到SST文件中*/
    WriteRawBlock(meta_index_builder.Finish(), kNoCompression,
                  &metaindex_block_handle);

    const bool is_data_block = true;
    /*将index Block（index_block.index_block_contents，准确说是dataindex Block）写入到SST文件中*/
    WriteBlock(index_blocks.index_block_contents, &index_block_handle,
               !is_data_block);
    // If there are more index partitions, finish them and write them out
    Status& s = index_builder_status;
    while (s.IsIncomplete()) {
      s = r->index_builder->Finish(&index_blocks, index_block_handle);
      if (!s.ok() && !s.IsIncomplete()) {
        return s;
      }
      WriteBlock(index_blocks.index_block_contents, &index_block_handle,
                 !is_data_block);
      // The last index_block_handle will be for the partition index block
    }
  }

  // Write footer
  /*写入footer信息到SST文件中*/
  if (ok()) {
    // No need to write out new footer if we're using default checksum.
    // We're writing legacy magic number because we want old versions of RocksDB
    // be able to read files generated with new release (just in case if
    // somebody wants to roll back after an upgrade)
    // TODO(icanadi) at some point in the future, when we're absolutely sure
    // nobody will roll back to RocksDB 2.x versions, retire the legacy magic
    // number and always write new table files with new magic number
    bool legacy = (r->table_options.format_version == 0);
    // this is guaranteed by BlockBasedTableBuilder's constructor
    assert(r->table_options.checksum == kCRC32c ||
           r->table_options.format_version != 0);
    /* 在Footer中记录magic number，format version，metaindex BlockHandle，
     * dataindex BlockHandle和checksum信息
     */
    Footer footer(legacy ? kLegacyBlockBasedTableMagicNumber
                         : kBlockBasedTableMagicNumber,
                  r->table_options.format_version);    
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    footer.set_checksum(r->table_options.checksum);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    /*最后写入Footer信息到SST文件中*/
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }

  return r->status;
}
```

### 总结
通过上述关于BlockBasedTableBuilder的代码分析，可以看出BlockBasedTable的格式如下：
```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: metadata of index block] (这里的metadata不要简单理解为index)
[meta block 2: filter block]
[meta block 3: property block]
[meta block 4: compression dictionary block]
[meta block 5: range deletion]
...
[meta block K: future extended block]  (用于未来扩展使用)
[metaindex block]
[index block]
[Footer]                               (固定大小，起始于file_size - sizeof(Footer)这个位置)
<end_of_file>
```

## BlockBasedTable
### BlockBasedTable构造函数

```
explicit BlockBasedTable(Rep* rep) : rep_(rep) {}

BlockBasedTable::Rep(const ImmutableCFOptions& _ioptions, const EnvOptions& _env_options,
  const BlockBasedTableOptions& _table_opt,
  const InternalKeyComparator& _internal_comparator, bool skip_filters)
  : ioptions(_ioptions),
    env_options(_env_options),
    table_options(_table_opt),
    filter_policy(skip_filters ? nullptr : _table_opt.filter_policy.get()),
    internal_comparator(_internal_comparator),
    filter_type(FilterType::kNoFilter),
    whole_key_filtering(_table_opt.whole_key_filtering),
    prefix_filtering(true),
    range_del_handle(BlockHandle::NullBlockHandle()),
    global_seqno(kDisableGlobalSequenceNumber) {}
```

### 辅助函数 - 读取指定的Block并检查其CRC（ReadBlock）
ReadBlock从指定的文件（通过file参数指定）中读取指定的Block（通过handle参数指定），并将读取的数据存放在contents中
```
// Read a block and check its CRC
// contents is the result of reading.
// According to the implementation of file->Read, contents may not point to buf
Status ReadBlock(RandomAccessFileReader* file, const Footer& footer,
                 const ReadOptions& options, const BlockHandle& handle,
                 Slice* contents, /* result of reading */ char* buf) {
  size_t n = static_cast<size_t>(handle.size());
  Status s;

  {
    PERF_TIMER_GUARD(block_read_time);
    /*从PosixRandomAccessFile的实现来看，contents一定是buf的一个子集*/
    s = file->Read(handle.offset(), n + kBlockTrailerSize, contents, buf);
  }

  PERF_COUNTER_ADD(block_read_count, 1);
  PERF_COUNTER_ADD(block_read_byte, n + kBlockTrailerSize);

  if (!s.ok()) {
    return s;
  }
  if (contents->size() != n + kBlockTrailerSize) {
    return Status::Corruption("truncated block read");
  }

  // Check the crc of the type and the block contents
  /*计算contents中真实数据和压缩类型这两部分的CRC，并检查计算得到的CRC是否和存储的CRC一致*/
  const char* data = contents->data();  // Pointer to where Read put the data
  if (options.verify_checksums) {
    PERF_TIMER_GUARD(block_checksum_time);
    /*存储的CRC*/
    uint32_t value = DecodeFixed32(data + n + 1);
    /*计算的新的CRC*/
    uint32_t actual = 0;
    switch (footer.checksum()) {
      case kCRC32c:
        value = crc32c::Unmask(value);
        actual = crc32c::Value(data, n + 1);
        break;
      case kxxHash:
        actual = XXH32(data, static_cast<int>(n) + 1, 0);
        break;
      default:
        s = Status::Corruption("unknown checksum type");
    }
    
    if (s.ok() && actual != value) {
      s = Status::Corruption("block checksum mismatch");
    }
    if (!s.ok()) {
      return s;
    }
  }
  return s;
}
```

### 辅助函数 - 读取指定的Block并根据需要解压缩，或者添加到压缩的/非压缩的持久化cache中（ReadBlockContents）
ReadBlockContents从指定文件中读取指定的Block，主要步骤如下：
1. 尝试在非压缩的/压缩的cache中中查找：
    1.1. 如果在非压缩的持久化cache中找到，则直接返回数据；
    1.2. 如果在压缩的持久化cache中找到，则跳到步骤3；
2. 尝试从硬盘文件中读取：
    2.1. 如果在硬盘文件中成功读取，且设置了压缩的持久化cache，则添加到该压缩的持久化cache中；
3. 如果数据是压缩的，且调用参数明确要求需要解压缩，则解压缩该数据；
    3.1. 如果设置了非压缩的持久化cache，则添加到该非压缩的持久化cache中；
4. 否则，如果数据是从压缩的持久化cache中获取的，则直接返回该数据；
5. 否则，如果数据是从硬盘文件中读取的：
    5.1. 如果设置了非压缩的持久化cache，则添加到该非压缩的持久化cache中；

```
Status ReadBlockContents(RandomAccessFileReader* file, const Footer& footer,
                         const ReadOptions& read_options,
                         const BlockHandle& handle, BlockContents* contents,
                         const ImmutableCFOptions &ioptions,
                         bool decompression_requested,
                         const Slice& compression_dict,
                         const PersistentCacheOptions& cache_options) {
  Status status;
  Slice slice;
  size_t n = static_cast<size_t>(handle.size());
  std::unique_ptr<char[]> heap_buf;
  char stack_buf[DefaultStackBufferSize];
  char* used_buf = nullptr;
  rocksdb::CompressionType compression_type;

  /*设置了持久化的cache，且cache中存放的是非压缩的数据*/
  if (cache_options.persistent_cache &&
      !cache_options.persistent_cache->IsCompressed()) {
    /*在非压缩的持久化cache中查找*/
    status = PersistentCacheHelper::LookupUncompressedPage(cache_options,
                                                           handle, contents);
    if (status.ok()) {
      // uncompressed page is found for the block handle
      /*成功读取到非压缩的数据*/
      return status;
    } else {
      // uncompressed page is not found
      /*在非压缩的持久化cache中未找到关于该Block的数据*/
      if (ioptions.info_log && !status.IsNotFound()) {
        assert(!status.ok());
        ROCKS_LOG_INFO(ioptions.info_log,
                       "Error reading from persistent cache. %s",
                       status.ToString().c_str());
      }
    }
  }

  /*设置了持久化的cache，且cache中存放的是压缩的数据*/
  if (cache_options.persistent_cache &&
      cache_options.persistent_cache->IsCompressed()) {
    // lookup uncompressed cache mode p-cache
    /*在压缩的持久化cache中查找，读取的压缩的数据存放在heap_buf中*/
    status = PersistentCacheHelper::LookupRawPage(
        cache_options, handle, &heap_buf, n + kBlockTrailerSize);
  } else {
    status = Status::NotFound();
  }

  if (status.ok()) {
    // cache hit
    /*在压缩的持久化cache中找到*/
    used_buf = heap_buf.get();
    slice = Slice(heap_buf.get(), n);
  } else {
    if (ioptions.info_log && !status.IsNotFound()) {
      assert(!status.ok());
      ROCKS_LOG_INFO(ioptions.info_log,
                     "Error reading from persistent cache. %s",
                     status.ToString().c_str());
    }
    // cache miss read from device
    /*在cache中未命中，需要到设备中读取*/
    /*首先分配足够大的buffer，用于存放读取的数据*/
    if (decompression_requested &&
        n + kBlockTrailerSize < DefaultStackBufferSize) {
      // If we've got a small enough hunk of data, read it in to the
      // trivially allocated stack buffer instead of needing a full malloc()
      used_buf = &stack_buf[0];
    } else {
      heap_buf = std::unique_ptr<char[]>(new char[n + kBlockTrailerSize]);
      used_buf = heap_buf.get();
    }

    /*尝试从硬盘文件中读取*/
    status = ReadBlock(file, footer, read_options, handle, &slice, used_buf);
    /* 如果读取成功 && 设置了将读取的数据添加到cache && 设置了持久化cache &&
     * 持久化cache是压缩的cache，则将该数据添加到该压缩的持久化cache中
     */
    if (status.ok() && read_options.fill_cache &&
        cache_options.persistent_cache &&
        cache_options.persistent_cache->IsCompressed()) {
      // insert to raw cache
      PersistentCacheHelper::InsertRawPage(cache_options, handle, used_buf,
                                           n + kBlockTrailerSize);
    }
  }

  if (!status.ok()) {
    return status;
  }

  PERF_TIMER_GUARD(block_decompress_time);

  /*从slice.data中获取压缩类型（存放在第n个字节中）*/
  compression_type = static_cast<rocksdb::CompressionType>(slice.data()[n]);
  
  if (decompression_requested && compression_type != kNoCompression) {
    // compressed page, uncompress, update cache
    /*解压缩数据，存放到contents中*/
    status = UncompressBlockContents(slice.data(), n, contents,
                                     footer.version(), compression_dict,
                                     ioptions);
  } else if (slice.data() != used_buf) {
    // the slice content is not the buffer provided
    /*数据是从压缩的持久化cache中读取到的，该数据无需再次添加到cache中（因为其已经在cache中了）*/
    *contents = BlockContents(Slice(slice.data(), n), false, compression_type);
  } else {
    // page is uncompressed, the buffer either stack or heap provided
    /* 至此，数据一定是从硬盘文件中读取的，如果数据存放在基于stack的buffer中，
     * 则要先将之拷贝到基于heap的buffer中
     */
    if (used_buf == &stack_buf[0]) {
      heap_buf = std::unique_ptr<char[]>(new char[n]);
      memcpy(heap_buf.get(), stack_buf, n);
    }
    *contents = BlockContents(std::move(heap_buf), n, true, compression_type);
  }

  if (status.ok() && read_options.fill_cache &&
      cache_options.persistent_cache &&
      !cache_options.persistent_cache->IsCompressed()) {
    // insert to uncompressed cache
    /*插入非压缩的持久化cache中（如果contents是压缩的，或者已经在cache中，则不被插入）*/
    PersistentCacheHelper::InsertUncompressedPage(cache_options, handle,
                                                  *contents);
  }

  return status;
}
```

### 辅助函数 - 从指定文件中读取指定的Block，并返回Block对象（ReadBlockFromFile）
ReadBlockFromFile从指定的文件中读取指定的Block，读取到的内容存放在result中。
```
// Read the block identified by "handle" from "file".
// The only relevant option is options.verify_checksums for now.
// On failure return non-OK.
// On success fill *result and return OK - caller owns *result
// @param compression_dict Data for presetting the compression library's
//    dictionary.
Status ReadBlockFromFile(RandomAccessFileReader* file, const Footer& footer,
                         const ReadOptions& options, const BlockHandle& handle,
                         std::unique_ptr<Block>* result,
                         const ImmutableCFOptions& ioptions, bool do_uncompress,
                         const Slice& compression_dict,
                         const PersistentCacheOptions& cache_options,
                         SequenceNumber global_seqno,
                         size_t read_amp_bytes_per_bit) {
  BlockContents contents;
  /*从指定文件中读取*/
  Status s = ReadBlockContents(file, footer, options, handle, &contents, ioptions,
                               do_uncompress, compression_dict, cache_options);
  if (s.ok()) {
    /*返回Block对象*/
    result->reset(new Block(std::move(contents), global_seqno,
                            read_amp_bytes_per_bit, ioptions.statistics));
  }

  return s;
}
```

### 读取Footer部分（BlockBasedTable::ReadFooterFromFile）
```
ReadFooterFromFile比较简单，不再赘述。
Status BlockBasedTable::ReadFooterFromFile(RandomAccessFileReader* file, uint64_t file_size,
                          Footer* footer, uint64_t enforce_table_magic_number) {
  if (file_size < Footer::kMinEncodedLength) {
    return Status::Corruption("file is too short to be an sstable");
  }

  char footer_space[Footer::kMaxEncodedLength];
  Slice footer_input;
  size_t read_offset =
      (file_size > Footer::kMaxEncodedLength)
          ? static_cast<size_t>(file_size - Footer::kMaxEncodedLength)
          : 0;
  Status s = file->Read(read_offset, Footer::kMaxEncodedLength, &footer_input,
                        footer_space);
  if (!s.ok()) return s;

  // Check that we actually read the whole footer from the file. It may be
  // that size isn't correct.
  if (footer_input.size() < Footer::kMinEncodedLength) {
    return Status::Corruption("file is too short to be an sstable");
  }

  s = footer->DecodeFrom(&footer_input);
  if (!s.ok()) {
    return s;
  }
  if (enforce_table_magic_number != 0 &&
      enforce_table_magic_number != footer->table_magic_number()) {
    return Status::Corruption("Bad table magic number");
  }
  
  return Status::OK();
}
```

### 读取MetaBlock（BlockBasedTable::ReadMetaBlock）
```
Status BlockBasedTable::ReadMetaBlock(Rep* rep,
                                      std::unique_ptr<Block>* meta_block,
                                      std::unique_ptr<InternalIterator>* iter) {
  // TODO(sanjay): Skip this if footer.metaindex_handle() size indicates
  // it is an empty block.
  //  TODO: we never really verify check sum for meta index block
  std::unique_ptr<Block> meta;
  Status s = ReadBlockFromFile(
      rep->file.get(), rep->footer, ReadOptions(),
      rep->footer.metaindex_handle(), &meta, rep->ioptions,
      true /* decompress */, Slice() /*compression dict*/,
      rep->persistent_cache_options, kDisableGlobalSequenceNumber,
      0 /* read_amp_bytes_per_bit */);

  if (!s.ok()) {
    ROCKS_LOG_ERROR(rep->ioptions.info_log,
                    "Encountered error while reading data from properties"
                    " block %s",
                    s.ToString().c_str());
    return s;
  }

  *meta_block = std::move(meta);
  // meta block uses bytewise comparator.
  iter->reset(meta_block->get()->NewIterator(BytewiseComparator()));
  return Status::OK();
}
```

### BlockBasedTable::Open
```
Status BlockBasedTable::Open(const ImmutableCFOptions& ioptions,
                             const EnvOptions& env_options,
                             const BlockBasedTableOptions& table_options,
                             const InternalKeyComparator& internal_comparator,
                             unique_ptr<RandomAccessFileReader>&& file,
                             uint64_t file_size,
                             unique_ptr<TableReader>* table_reader,
                             const bool prefetch_index_and_filter_in_cache,
                             const bool skip_filters, const int level) {
  table_reader->reset();

  Footer footer;

  // Before read footer, readahead backwards to prefetch data
  /*预读最后512KB*/
  Status s =
      file->Prefetch((file_size < 512 * 1024 ? 0 : file_size - 512 * 1024),
                     512 * 1024 /* 512 KB prefetching */);
  /*读取Footer部分*/                    
  s = ReadFooterFromFile(file.get(), file_size, &footer,
                              kBlockBasedTableMagicNumber);
  if (!s.ok()) {
    return s;
  }
  
  /*Footer中的Version信息不匹配*/
  if (!BlockBasedTableSupportedVersion(footer.version())) {
    return Status::Corruption(
        "Unknown Footer version. Maybe this file was created with newer "
        "version of RocksDB?");
  }

  // We've successfully read the footer. We are ready to serve requests.
  // Better not mutate rep_ after the creation. eg. internal_prefix_transform
  // raw pointer will be used to create HashIndexReader, whose reset may
  // access a dangling pointer.
  /*创建BlockBasedTableReader，首先创建BlockBasedTable::Rep，然后创建BlockBasedTable*/
  Rep* rep = new BlockBasedTable::Rep(ioptions, env_options, table_options,
                                      internal_comparator, skip_filters);
  rep->file = std::move(file);
  rep->footer = footer;
  rep->index_type = table_options.index_type;
  rep->hash_index_allow_collision = table_options.hash_index_allow_collision;
  // We need to wrap data with internal_prefix_transform to make sure it can
  // handle prefix correctly.
  rep->internal_prefix_transform.reset(
      new InternalKeySliceTransform(rep->ioptions.prefix_extractor));
  SetupCacheKeyPrefix(rep, file_size);
  unique_ptr<BlockBasedTable> new_table(new BlockBasedTable(rep));

  // page cache options
  rep->persistent_cache_options =
      PersistentCacheOptions(rep->table_options.persistent_cache,
                             std::string(rep->persistent_cache_key_prefix,
                                         rep->persistent_cache_key_prefix_size),
                                         rep->ioptions.statistics);

  // Read meta index
  std::unique_ptr<Block> meta;
  std::unique_ptr<InternalIterator> meta_iter;
  s = ReadMetaBlock(rep, &meta, &meta_iter);
  if (!s.ok()) {
    return s;
  }

  // Find filter handle and filter type
  if (rep->filter_policy) {
    for (auto filter_type :
         {Rep::FilterType::kFullFilter, Rep::FilterType::kPartitionedFilter,
          Rep::FilterType::kBlockFilter}) {
      std::string prefix;
      switch (filter_type) {
        case Rep::FilterType::kFullFilter:
          prefix = kFullFilterBlockPrefix;
          break;
        case Rep::FilterType::kPartitionedFilter:
          prefix = kPartitionedFilterBlockPrefix;
          break;
        case Rep::FilterType::kBlockFilter:
          prefix = kFilterBlockPrefix;
          break;
        default:
          assert(0);
      }
      std::string filter_block_key = prefix;
      filter_block_key.append(rep->filter_policy->Name());
      if (FindMetaBlock(meta_iter.get(), filter_block_key, &rep->filter_handle)
              .ok()) {
        rep->filter_type = filter_type;
        break;
      }
    }
  }

  // Read the properties
  bool found_properties_block = true;
  s = SeekToPropertiesBlock(meta_iter.get(), &found_properties_block);

  if (!s.ok()) {
    ROCKS_LOG_WARN(rep->ioptions.info_log,
                   "Error when seeking to properties block from file: %s",
                   s.ToString().c_str());
  } else if (found_properties_block) {
    s = meta_iter->status();
    TableProperties* table_properties = nullptr;
    if (s.ok()) {
      s = ReadProperties(meta_iter->value(), rep->file.get(), rep->footer,
                         rep->ioptions, &table_properties);
    }

    if (!s.ok()) {
      ROCKS_LOG_WARN(rep->ioptions.info_log,
                     "Encountered error while reading data from properties "
                     "block %s",
                     s.ToString().c_str());
    } else {
      rep->table_properties.reset(table_properties);
    }
  } else {
    ROCKS_LOG_ERROR(rep->ioptions.info_log,
                    "Cannot find Properties block from file.");
  }

  // Read the compression dictionary meta block
  bool found_compression_dict;
  s = SeekToCompressionDictBlock(meta_iter.get(), &found_compression_dict);
  if (!s.ok()) {
    ROCKS_LOG_WARN(
        rep->ioptions.info_log,
        "Error when seeking to compression dictionary block from file: %s",
        s.ToString().c_str());
  } else if (found_compression_dict) {
    // TODO(andrewkr): Add to block cache if cache_index_and_filter_blocks is
    // true.
    unique_ptr<BlockContents> compression_dict_block{new BlockContents()};
    // TODO(andrewkr): ReadMetaBlock repeats SeekToCompressionDictBlock().
    // maybe decode a handle from meta_iter
    // and do ReadBlockContents(handle) instead
    s = rocksdb::ReadMetaBlock(rep->file.get(), file_size,
                               kBlockBasedTableMagicNumber, rep->ioptions,
                               rocksdb::kCompressionDictBlock,
                               compression_dict_block.get());
    if (!s.ok()) {
      ROCKS_LOG_WARN(
          rep->ioptions.info_log,
          "Encountered error while reading data from compression dictionary "
          "block %s",
          s.ToString().c_str());
    } else {
      rep->compression_dict_block = std::move(compression_dict_block);
    }
  }

  // Read the range del meta block
  bool found_range_del_block;
  s = SeekToRangeDelBlock(meta_iter.get(), &found_range_del_block,
                          &rep->range_del_handle);
  if (!s.ok()) {
    ROCKS_LOG_WARN(
        rep->ioptions.info_log,
        "Error when seeking to range delete tombstones block from file: %s",
        s.ToString().c_str());
  } else {
    if (found_range_del_block && !rep->range_del_handle.IsNull()) {
      ReadOptions read_options;
      s = MaybeLoadDataBlockToCache(rep, read_options, rep->range_del_handle,
                                    Slice() /* compression_dict */,
                                    &rep->range_del_entry);
      if (!s.ok()) {
        ROCKS_LOG_WARN(
            rep->ioptions.info_log,
            "Encountered error while reading data from range del block %s",
            s.ToString().c_str());
      }
    }
  }

  // Determine whether whole key filtering is supported.
  if (rep->table_properties) {
    rep->whole_key_filtering &=
        IsFeatureSupported(*(rep->table_properties),
                           BlockBasedTablePropertyNames::kWholeKeyFiltering,
                           rep->ioptions.info_log);
    rep->prefix_filtering &= IsFeatureSupported(
        *(rep->table_properties),
        BlockBasedTablePropertyNames::kPrefixFiltering, rep->ioptions.info_log);

    rep->global_seqno = GetGlobalSequenceNumber(*(rep->table_properties),
                                                rep->ioptions.info_log);
  }

    // pre-fetching of blocks is turned on
  // Will use block cache for index/filter blocks access
  // Always prefetch index and filter for level 0
  if (table_options.cache_index_and_filter_blocks) {
    if (prefetch_index_and_filter_in_cache || level == 0) {
      assert(table_options.block_cache != nullptr);
      // Hack: Call NewIndexIterator() to implicitly add index to the
      // block_cache

      // if pin_l0_filter_and_index_blocks_in_cache is true and this is
      // a level0 file, then we will pass in this pointer to rep->index
      // to NewIndexIterator(), which will save the index block in there
      // else it's a nullptr and nothing special happens
      CachableEntry<IndexReader>* index_entry = nullptr;
      if (rep->table_options.pin_l0_filter_and_index_blocks_in_cache &&
          level == 0) {
        index_entry = &rep->index_entry;
      }
      unique_ptr<InternalIterator> iter(
          new_table->NewIndexIterator(ReadOptions(), nullptr, index_entry));
      s = iter->status();

      if (s.ok()) {
        // Hack: Call GetFilter() to implicitly add filter to the block_cache
        auto filter_entry = new_table->GetFilter();
        // if pin_l0_filter_and_index_blocks_in_cache is true, and this is
        // a level0 file, then save it in rep_->filter_entry; it will be
        // released in the destructor only, hence it will be pinned in the
        // cache while this reader is alive
        if (rep->table_options.pin_l0_filter_and_index_blocks_in_cache &&
            level == 0) {
          rep->filter_entry = filter_entry;
          if (rep->filter_entry.value != nullptr) {
            rep->filter_entry.value->SetLevel(level);
          }
        } else {
          filter_entry.Release(table_options.block_cache.get());
        }
      }
    }
  } else {
    // If we don't use block cache for index/filter blocks access, we'll
    // pre-load these blocks, which will kept in member variables in Rep
    // and with a same life-time as this table object.
    IndexReader* index_reader = nullptr;
    s = new_table->CreateIndexReader(&index_reader, meta_iter.get(), level);

    if (s.ok()) {
      rep->index_reader.reset(index_reader);

      // Set filter block
      if (rep->filter_policy) {
        const bool is_a_filter_partition = true;
        rep->filter.reset(
            new_table->ReadFilter(rep->filter_handle, !is_a_filter_partition));
        if (rep->filter.get()) {
          rep->filter->SetLevel(level);
        }
      }
    } else {
      delete index_reader;
    }
  }

  if (s.ok()) {
    *table_reader = std::move(new_table);
  }

  return s;
}
```



## IndexBuilder
### 根据Index类型创建相应的IndexBuilder（IndexBuilder::CreateIndexBuilder）
```
IndexBuilder* IndexBuilder::CreateIndexBuilder(
    BlockBasedTableOptions::IndexType index_type,
    const InternalKeyComparator* comparator,
    const InternalKeySliceTransform* int_key_slice_transform,
    const BlockBasedTableOptions& table_opt) {
  /*根据index类型创建相应的IndexBuilder*/
  switch (index_type) {
    case BlockBasedTableOptions::kBinarySearch: {
      return new ShortenedIndexBuilder(comparator,
                                       table_opt.index_block_restart_interval);
    }
    case BlockBasedTableOptions::kHashSearch: {
      return new HashIndexBuilder(comparator, int_key_slice_transform,
                                  table_opt.index_block_restart_interval);
    }
    case BlockBasedTableOptions::kTwoLevelIndexSearch: {
      return PartitionedIndexBuilder::CreateIndexBuilder(comparator, table_opt);
    }
    default: {
      assert(!"Do not recognize the index type ");
      return nullptr;
    }
  }
  // impossible.
  assert(false);
  return nullptr;
}
```

## ShortenedIndexBuilder
### ShortenedIndexBuilder构造函数

```
  explicit ShortenedIndexBuilder::ShortenedIndexBuilder(const InternalKeyComparator* comparator,
                                 int index_block_restart_interval)
      : IndexBuilder(comparator),
        /*向Index Block中插入也需要借助BlockBuilder*/
        index_block_builder_(index_block_restart_interval) {}
```

### 向当前的index Block中添加一个index entry（ShortenedIndexBuilder::AddIndexEntry）
ShortenedIndexBuilder::AddIndexEntry()向当前的index Block中添加一个index entry，首先为本次插入确定一个key，接着对要插入的内容进行编码作为value，最后插入key-value pair。其中block_handle对应当前Block的索引内容，last_key_in_current_block对应当前的Block的最后一个key，first_key_in_next_block对应下一个Block的第一个key，需要查找一个介于[last_key_in_current_block的user key, fist_key_in_next_block的user key)之间的key作为当前Block的索引的key。

```
  virtual void ShortenedIndexBuilder::AddIndexEntry(std::string* last_key_in_current_block,
                             const Slice* first_key_in_next_block,
                             const BlockHandle& block_handle) override {
    if (first_key_in_next_block != nullptr) {
      /* 如果last_key_in_current_block中的user key部分小于first_key_in_next_block中的user
       * key部分，则在[last_key_in_current_block的user key, fist_key_in_next_block的user key)
       * 中查找一个物理上更短（从空间占用上来说）且逻辑上比last_key_in_current_block的user key
       * 更大（比如字典序排序）的字符串作为本次插入的index entry的key
       */
      comparator_->FindShortestSeparator(last_key_in_current_block,
                                         *first_key_in_next_block);
    } else {
      /* 查找一个逻辑上大于（比如字典序排序）last_key_in_current_block的user key部分且
       * 较短的key作为本次插入的index entry的key
       */
      comparator_->FindShortSuccessor(last_key_in_current_block);
    }

    std::string handle_encoding;
    /*对要插入的index entry进行编码*/
    block_handle.EncodeTo(&handle_encoding);
    /*将index entry添加到index block中（*last_key_in_current_block作为key，handle_encoding作为value）*/
    index_block_builder_.Add(*last_key_in_current_block, handle_encoding);
  }
```

ShortenedIndexBuilder::AddIndexEntry()的调用关系为：

```
void BlockBasedTableBuilder::Add(const Slice& key, const Slice& value) {
  Rep* r = rep_;
  ......
  
  if (IsValueType(value_type)) {
    ......
    
    if (should_flush) {
      assert(!r->data_block.empty());
      Flush();

      // Add item to index block.
      if (ok()) {
        r->index_builder->AddIndexEntry(&r->last_key, &key, r->pending_handle);
      }
    }
    
    ......
}
    
Status BlockBasedTableBuilder::Finish() {
  ......
  
  Flush();
  
  ......
  
  if (ok() && !empty_data_block) {
    r->index_builder->AddIndexEntry(
        &r->last_key, nullptr /* no next data block */, r->pending_handle);
  }
  
  ......
}
```

从调用关系来看，当TableBuilder检测到当前的Block需要被Flush到SST文件的时候，或者TableBuilder主动调用Finish停止向当前的SST文件中写入数据的时候，都会先调用TableBuilder::Flush将当前Block中的数据写入到SST文件中，然后调用IndexBuilder::AddIndexEntry在index Block中添加一项关于刚刚被Flush的Block的索引。

### 结束构建当前的index Block（ShortenedIndexBuilder::Finish）
ShortenedIndexBuilder::Finish()停止向当前的index Block中写入数据。
```
  virtual Status ShortenedIndexBuilder::Finish(
      IndexBlocks* index_blocks,
      const BlockHandle& last_partition_block_handle) override {
    /* 直接调用关于该index Block的BlockBuilder的Finish()，停止向当前的index Block中写入数据，
     * 返回指向该index Block的内容的Slice
     */
    index_blocks->index_block_contents = index_block_builder_.Finish();
    return Status::OK();
  }
```

### 预估当前index Block的大小（ShortenedIndexBuilder::EstimatedSize）
ShortenedIndexBuilder::EstimatedSize()返回当前的index Block的预估大小。
```
  virtual size_t ShortenedIndexBuilder::EstimatedSize() const override {
    return index_block_builder_.CurrentSizeEstimate();
  }
```

## BlockBuilder
### BlockBuilder构造函数
```
BlockBuilder::BlockBuilder(int block_restart_interval, bool use_delta_encoding)
    : block_restart_interval_(block_restart_interval),
      use_delta_encoding_(use_delta_encoding),
      restarts_(),
      counter_(0),
      finished_(false) {
  assert(block_restart_interval_ >= 1);
  restarts_.push_back(0);       // First restart point is at offset 0
  estimate_ = sizeof(uint32_t) + sizeof(uint32_t);
}
```

### 将指定的key-value pair添加到Block中（BlockBuilder::Add）
BlockBuilder::Add()将指定的key-value pair添加到Block中，分为以下步骤：

- 如果当前Block中添加的key-value pair的数目不小于block_restart_interval_，则结束当前的压缩块（包含一个或者多个key-value pairs，一个Block中可能包含一个或者多个压缩块，每个压缩块的第一个key是不压缩的，每个压缩块中除第一个key以外的后续key是采用只存储非公共前缀部分的方式进行压缩）并启动一个新的压缩块，新添加的key作为该新的压缩块的第一个key；如果当前Block中添加的key-value pair的数目小于block_restart_interval_，则计算当前key和前一个key的公共前缀；

- 计算当前key和前一个key之间非公共前缀部分的长度，对于新开启的压缩块的情况(counter_ >= block_restart_interval_)，非公共前缀部分的长度就是当前添加的key的长度，对于直接使用当前压缩块的情况(counter_ < block_restart_interval_)，非公共前缀部分的长度就是当前添加的key的长度减去公共前缀部分的长度；

- 在当前Block的buffer_中记录<公共前缀部分长度><非公共前缀部分长度><value大小>这3个信息；

- 在当前Block的buffer_中记录当前key的增量部分（即非公共前缀部分）；

- 在当前Block的buffer_中记录value；

- 更新当前压缩块中添加的key-value pair的数目；


```
void BlockBuilder::Add(const Slice& key, const Slice& value) {
  assert(!finished_);
  assert(counter_ <= block_restart_interval_);
  /*和前一个key共享的前缀的字节数*/
  size_t shared = 0;  // number of bytes shared with prev key

  if (counter_ >= block_restart_interval_) {
    /* 当前Block中添加了不少于block_restart_interval_个key-value pair，
     * 则结束当前的压缩块（只存放当前key相对于前一个key的非公共前缀部分，
     * 可以这么理解，一个Block中可能包含多个压缩块）， 开启新的压缩，
     * 并记录新的压缩的起始位置
     */
    // Restart compression
    /*restarts_中记录每一个压缩块的大小*/
    restarts_.push_back(static_cast<uint32_t>(buffer_.size()));
    estimate_ += sizeof(uint32_t);
    /*重置counter_计数*/
    counter_ = 0;

    if (use_delta_encoding_) {
      // Update state
      /* 对key采用基于增量编码的压缩，则会计算当前的key和前一个key的公共前缀，
       * 所以需要记录前一个key，这里因为开启了新的压缩块，所以当前key会作为
       * 该新的压缩块的第一个key（第一个key不会被压缩）
       */
      last_key_.assign(key.data(), key.size());
    }
  } else if (use_delta_encoding_) {
    /*计算当前的key和前一个key之间的公共前缀*/
    Slice last_key_piece(last_key_);
    // See how much sharing to do with previous string
    shared = key.difference_offset(last_key_piece);

    // Update state
    // We used to just copy the changed data here, but it appears to be
    // faster to just copy the whole thing.
    /*记录前一个key*/
    last_key_.assign(key.data(), key.size());
  }

  /*当前key和前一个key之间非公共部分的长度*/
  const size_t non_shared = key.size() - shared;
  const size_t curr_size = buffer_.size();

  // Add "<shared><non_shared><value_size>" to buffer_
  /*在当前Block的buffer_中记录<shared><non_shared><value_size>这3个信息*/
  PutVarint32Varint32Varint32(&buffer_, static_cast<uint32_t>(shared),
                              static_cast<uint32_t>(non_shared),
                              static_cast<uint32_t>(value.size()));

  // Add string delta to buffer_ followed by value
  /*在当前Block的buffer_中记录当前key的delta部分（即非公共前缀部分）*/
  buffer_.append(key.data() + shared, non_shared);
  /*在当前Block的buffer_中记录value*/
  buffer_.append(value.data(), value.size());

  counter_++;
  estimate_ += buffer_.size() - curr_size;
}
```

### 结束构建当前的Block并且返回指向当前Block内容的slice（BlockBuilder::Finish）
BlockBuilder::Finish()结束构建当前的Block并且返回指向当前Block内容的slice，在调用Finish()的时候实际上是将restarts_数组写入到Block中。
```
Slice BlockBuilder::Finish() {
  // Append restart array
  for (size_t i = 0; i < restarts_.size(); i++) {
    /*将restarts_数组中的元素逐一添加到Block中*/
    PutFixed32(&buffer_, restarts_[i]);
  }
  
  /*将restarts_数组中元素的个数添加到Block中*/
  PutFixed32(&buffer_, static_cast<uint32_t>(restarts_.size()));
  finished_ = true;
  return Slice(buffer_);
}
```

### 重置BlockBuilder关联的当前Block，为复用做准备（BlockBuilder::Reset）
```
void BlockBuilder::Reset() {
  /*清空buffer中的内容*/
  buffer_.clear();
  /*清空restarts_中的记录*/
  restarts_.clear();
  restarts_.push_back(0);       // First restart point is at offset 0
  estimate_ = sizeof(uint32_t) + sizeof(uint32_t);
  counter_ = 0;
  finished_ = false;
  /*重置last_key_*/
  last_key_.clear();
}
```

### 预估添加了key-value pair之后的Block大小（BlockBuilder::EstimateSizeAfterKV）
```
size_t BlockBuilder::EstimateSizeAfterKV(const Slice& key, const Slice& value)
  const {
  size_t estimate = CurrentSizeEstimate();
  estimate += key.size() + value.size();
  if (counter_ >= block_restart_interval_) {
    estimate += sizeof(uint32_t); // a new restart entry.
  }

  estimate += sizeof(int32_t); // varint for shared prefix length.
  estimate += VarintLength(key.size()); // varint for key length.
  estimate += VarintLength(value.size()); // varint for value length.

  return estimate;
}
```




