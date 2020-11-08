# 提纲
[toc]

# 前言
RocksDB使用SST(Sorted Sequence Table)文件来存储用户写入的数据，文件中的数据是按照key排序的。BlockBasedTable是SST文件的默认格式，此外，RocksDB还支持PlainTable，CuckooTable等。本文主要分析BlockBasedTable。

# RocksDB Flush
```
BGWorkFlush
DBImpl::BackgroundCallFlush
DBImpl::BackgroundFlush
DBImpl::FlushMemTableToOutputFile
FlushJob::Run
FlushJob::WriteLevel0Table

Result<FileNumbersHolder> FlushJob::WriteLevel0Table(
    const autovector<MemTable*>& mems, VersionEdit* edit, FileMetaData* meta) {
  ...
    
  auto file_number_holder = file_numbers_provider_->NewFileNumber();
  auto file_number = file_number_holder.Last();
  meta->fd = FileDescriptor(file_number, 0, 0, 0);    

  {
    # 用于保存所有MemTable对应的MemTableIterator
    std::vector<InternalIterator*> memtables;
    Arena arena;
    
    for (MemTable* m : mems) {
      # 基于每个MemTable创建MemTableIterator，添加到memtables中
      memtables.push_back(m->NewIterator(ro, &arena));
      
      # 使用MemTable对应的UserFrontiers更新FileMetaData::smallest.user_frontier
      # 和FileMetaData::largest.user_frontier
      const auto* range = m->Frontiers();
      if (range) {
        UserFrontier::Update(
            &range->Smallest(), UpdateUserValueType::kSmallest, &meta->smallest.user_frontier);
        UserFrontier::Update(
            &range->Largest(), UpdateUserValueType::kLargest, &meta->largest.user_frontier);
      }
    }    
  }
  
  ...
}

```



```
<beginning_of_file>
[data block 1]
[data block 2]
...
[data block N]
[meta block 1: filter block]                  (see section: "filter" Meta Block)
[meta block 2: index block]
[meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
[meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
[meta block 5: stats block]                   (see section: "properties" Meta Block)
...
[meta block K: future extended block]  (we may add more meta blocks in the future)
[metaindex block]
[Footer]                               (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```



压缩后的block格式：
```
compressed data
compression type                        (1 Byte)
```

解压缩之后的数据部分的格式:
```

restarts num                            (4 Bytes)
```


Footer格式：
```
checksum type                           (1 Byte)
meta index BlockHandle                  (20 Bytes)
data index BlockHandle                  (20 Bytes)
version                                 (4 Bytes)
magic low                               (4 Bytes)
magic high                              (4 Bytes)
```



# 参考
[RocksDB block-based SST 文件详解](https://www.jianshu.com/p/d6ce3593a69e)
[Rocksdb Compaction 源码详解（一）：SST文件详细格式源码解析](https://blog.csdn.net/Z_Stand/article/details/106959058)
[RocksDB. BlockBasedTable源码分析](https://www.jianshu.com/p/9b5ade5212ab)
[RocksDB Bloom Filter](https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter)
[浅析RocksDB的SSTable格式](https://zhuanlan.zhihu.com/p/37633790)