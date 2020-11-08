# 提纲
[toc]

# 接口

```
  // Flush all mem-table data.
  virtual Status Flush(const FlushOptions& options,
                       ColumnFamilyHandle* column_family) = 0;
  virtual Status Flush(const FlushOptions& options) {
    return Flush(options, DefaultColumnFamily());
  }
```
