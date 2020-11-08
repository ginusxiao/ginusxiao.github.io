# 提纲
[toc]

# 接口

```
  // Set the database entry for "key" to "value".
  // If "key" already exists, it will be overwritten.
  // Returns OK on success, and a non-OK status on error.
  // Note: consider setting options.sync = true.
  virtual Status Put(const WriteOptions& options,
                     ColumnFamilyHandle* column_family, const Slice& key,
                     const Slice& value) = 0;
  virtual Status Put(const WriteOptions& options, const Slice& key,
                     const Slice& value) {
    return Put(options, DefaultColumnFamily(), key, value);
  }
  
  // Apply the specified updates to the database.
  // If `updates` contains no update, WAL will still be synced if
  // options.sync=true.
  // Returns OK on success, non-OK on failure.
  // Note: consider setting options.sync = true.
  virtual Status Write(const WriteOptions& options, WriteBatch* updates) = 0;
```

# 实现
Rocksdb中提供了两种写实现：DBImpl::WriteImpl和DBImpl::PipelinedWriteImpl。

PipelineWriteImpl is an alternative approach to WriteImpl. In WriteImpl, only one thread is allow to write at the same time. This thread will do both WAL and memtable writes for all write threads in the write group. Pending writers wait in queue until the current writer finishes. In the pipeline write approach, two queue is maintained: one WAL writer queue and one memtable writer queue. All writers (regardless of whether they need to write WAL) will still need to first join the WAL writer queue, and after the house keeping work and WAL writing, they will need to join memtable writer queue if needed. The benefit of this approach is that
1. Writers without memtable writes (e.g. the prepare phase of two phase commit) can exit write thread once WAL write is finish. They don't need to wait for memtable writes in case of group commit.
2. Pending writers only need to wait for previous WAL writer finish to be able to join the write thread, instead of wait also for previous memtable writes.


## DBImpl::WriteImpl
![image](http://note.youdao.com/noteshare?id=bc0591dda896d73831acb2c91e70c150&sub=BE3C12A23CD84AC88389E68D8409AE85)


## DBImpl::PipelinedWriteImpl
![image](http://note.youdao.com/noteshare?id=6644068ceb8e9ace005dcc25be031e06&sub=A1E8DE23E7F446F385A4244593934F7D)


# 参考
[New WriteImpl to pipeline WAL/memtable write](https://gitlab.com/barrel-db/rocksdb/commit/07bdcb91fe1fb6fcbed885580dcded8f5eac36f5)


