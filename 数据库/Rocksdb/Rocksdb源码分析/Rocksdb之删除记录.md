# 提纲
[toc]

# 接口
```
  // Remove the database entry (if any) for "key".  Returns OK on
  // success, and a non-OK status on error.  It is not an error if "key"
  // did not exist in the database.
  // Note: consider setting options.sync = true.
  virtual Status Delete(const WriteOptions& options,
                        ColumnFamilyHandle* column_family,
                        const Slice& key) = 0;
  virtual Status Delete(const WriteOptions& options, const Slice& key) {
    return Delete(options, DefaultColumnFamily(), key);
  }
  
  // Remove the database entry for "key". Requires that the key exists
  // and was not overwritten. Returns OK on success, and a non-OK status
  // on error.  It is not an error if "key" did not exist in the database.
  //
  // If a key is overwritten (by calling Put() multiple times), then the result
  // of calling SingleDelete() on this key is undefined.  SingleDelete() only
  // behaves correctly if there has been only one Put() for this key since the
  // previous call to SingleDelete() for this key.
  //
  // This feature is currently an experimental performance optimization
  // for a very specific workload.  It is up to the caller to ensure that
  // SingleDelete is only used for a key that is not deleted using Delete() or
  // written using Merge().  Mixing SingleDelete operations with Deletes and
  // Merges can result in undefined behavior.
  //
  // Note: consider setting options.sync = true.
  virtual Status SingleDelete(const WriteOptions& options,
                              ColumnFamilyHandle* column_family,
                              const Slice& key) = 0;
  virtual Status SingleDelete(const WriteOptions& options, const Slice& key) {
    return SingleDelete(options, DefaultColumnFamily(), key);
  }

  // Removes the database entries in the range ["begin_key", "end_key"), i.e.,
  // including "begin_key" and excluding "end_key". Returns OK on success, and
  // a non-OK status on error. It is not an error if no keys exist in the range
  // ["begin_key", "end_key").
  //
  // This feature is currently an experimental performance optimization for
  // deleting very large ranges of contiguous keys. Invoking it many times or on
  // small ranges may severely degrade read performance; in particular, the
  // resulting performance can be worse than calling Delete() for each key in
  // the range. Note also the degraded read performance affects keys outside the
  // deleted ranges, and affects database operations involving scans, like flush
  // and compaction.
  //
  // Consider setting ReadOptions::ignore_range_deletions = true to speed
  // up reads for key(s) that are known to be unaffected by range deletions.
  virtual Status DeleteRange(const WriteOptions& options,
                             ColumnFamilyHandle* column_family,
                             const Slice& begin_key, const Slice& end_key);
```

