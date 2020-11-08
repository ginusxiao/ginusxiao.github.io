# Outline
[toc]

# Introduction

rocksdb从3.0开始支持ColumnFamily的概念。每一个键值对唯一关联到一个Column Family，如果没有为该键值对指定Column Family，则该键值对关联到名为“default”的Column Family。Column Families提供了一种逻辑上对数据库进行分区的方式，它具有以下属性：

- 支持跨Column Families的原子更新，即支持原子的执行Write({cf1, key1, value1}, {cf2, key2, value2})。

- 提供跨Column Families的数据库一致性视图。

- 支持单独配置每个Column Families。

- 支持在线添加或者删除Column Families。

# API
## Example sample

Refer to [column_families_example.cc](https://github.com/facebook/rocksdb/blob/master/examples/column_families_example.cc).

```
// Copyright (c) 2011-present, Facebook, Inc.  All rights reserved.
//  This source code is licensed under both the GPLv2 (found in the
//  COPYING file in the root directory) and Apache 2.0 License
//  (found in the LICENSE.Apache file in the root directory).
#include <cstdio>
#include <string>
#include <vector>

#include "rocksdb/db.h"
#include "rocksdb/slice.h"
#include "rocksdb/options.h"

using namespace rocksdb;

std::string kDBPath = "/tmp/rocksdb_column_families_example";

int main() {
  // open DB
  Options options;
  options.create_if_missing = true;
  DB* db;
  Status s = DB::Open(options, kDBPath, &db);
  assert(s.ok());

  // create column family
  ColumnFamilyHandle* cf;
  s = db->CreateColumnFamily(ColumnFamilyOptions(), "new_cf", &cf);
  assert(s.ok());

  // close DB
  delete cf;
  delete db;

  // open DB with two column families
  std::vector<ColumnFamilyDescriptor> column_families;
  // have to open default column family
  column_families.push_back(ColumnFamilyDescriptor(
      kDefaultColumnFamilyName, ColumnFamilyOptions()));
  // open the new one, too
  column_families.push_back(ColumnFamilyDescriptor(
      "new_cf", ColumnFamilyOptions()));
  std::vector<ColumnFamilyHandle*> handles;
  s = DB::Open(DBOptions(), kDBPath, column_families, &handles, &db);
  assert(s.ok());

  // put and get from non-default column family
  s = db->Put(WriteOptions(), handles[1], Slice("key"), Slice("value"));
  assert(s.ok());
  std::string value;
  s = db->Get(ReadOptions(), handles[1], Slice("key"), &value);
  assert(s.ok());

  // atomic write
  WriteBatch batch;
  batch.Put(handles[0], Slice("key2"), Slice("value2"));
  batch.Put(handles[1], Slice("key3"), Slice("value3"));
  batch.Delete(handles[0], Slice("key"));
  s = db->Write(WriteOptions(), &batch);
  assert(s.ok());

  // drop column family
  s = db->DropColumnFamily(handles[1]);
  assert(s.ok());

  // close db
  for (auto handle : handles) {
    delete handle;
  }
  delete db;

  return 0;
}
```

## Reference

    Options, ColumnFamilyOptions, DBOptions
    
Options structures define how RocksDB behaves and performs. Before, every option was defined in a single Options struct. Going forward, options specific to a single Column Family will be defined in ColumnFamilyOptions and options specific to the whole RocksDB instance will be defined in DBOptions. Options struct is inheriting both ColumnFamilyOptions and DBOptions, which means you can still use it to define all the options for a DB instance with a single (default) column family.

    ColumnFamilyHandle
    
Column Families are handled and referenced with a ColumnFamilyHandle. Think of it as a open file descriptor. You need to delete all ColumnFamilyHandles before you delete your DB pointer. One interesting thing: Even if ColumnFamilyHandle is pointing to a dropped Column Family, you can continue using it. The data is actually deleted only after you delete all outstanding ColumnFamilyHandles.

    DB::Open(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr);

**When opening a DB in a read-write mode, you need to specify all Column Families that currently exist in a DB. If that's not the case, DB::Open call will return Status::InvalidArgument()**. You specify Column Families with a vector of ColumnFamilyDescriptors. ColumnFamilyDescriptor is just a struct with a Column Family name and ColumnFamilyOptions. Open call will return a Status and also a vector of pointers to ColumnFamilyHandles, which you can then use to reference Column Families. Make sure to delete all ColumnFamilyHandles before you delete the DB pointer.

    DB::OpenForReadOnly(const DBOptions& db_options, const std::string& name, const std::vector<ColumnFamilyDescriptor>& column_families, std::vector<ColumnFamilyHandle*>* handles, DB** dbptr, bool error_if_log_file_exist = false)
    
The behavior is similar to DB::Open, except that it opens DB in read-only mode. **One big difference is that when opening the DB as read-only, you don't need to specify all Column Families -- you can only open a subset of Column Families**.

    DB::ListColumnFamilies(const DBOptions& db_options, const std::string& name, std::vector<std::string>* column_families)
    
ListColumnFamilies is a static function that returns the list of all column families currently present in the DB.

    DB::CreateColumnFamily(const ColumnFamilyOptions& options, const std::string& column_family_name, ColumnFamilyHandle** handle)
    
Creates a Column Family specified with option and a name and returns ColumnFamilyHandle through an argument.

    DropColumnFamily(ColumnFamilyHandle* column_family)
    
Drop the column family specified by ColumnFamilyHandle. **Note that the actual data is not deleted until the client calls delete column_family;. You can still continue using the column family if you have outstanding ColumnFamilyHandle pointer**.

    DB::NewIterators(const ReadOptions& options, const std::vector<ColumnFamilyHandle*>& column_families, std::vector<Iterator*>* iterators)
    
This is the new call, which enables you to create iterators on multiple Column Families that have consistent view of the database.

### WriteBatch

To execute multiple writes atomically, you need to build a WriteBatch. All WriteBatch API calls now also take ColumnFamilyHandle* to specify the Column Family you want to write to.

### All other API calls

All other API calls have a new argument ColumnFamilyHandle*, through which you can specify the Column Family.

# Implementation

The main idea behind Column Families is that they share the write-ahead log and don't share memtables and table files. By sharing write-ahead logs we get awesome benefit of atomic writes. By separating memtables and table files, we are able to configure column families independently and delete them quickly.

**Every time a single Column Family is flushed, we create a new WAL (write-ahead log)[注释：flush即将memtable中的数据写入Level-0中的sstfiles中，flush的时候Active memtable会切换为Immutable memtable，同时WAL log也会发生切换，创建新的WAL log可理解为WAL log切换，见下图]**. All new writes to all Column Families go to the new WAL. However, we still can't delete the old WAL since it contains live data from other Column Families. We can delete the old WAL only when all Column Families have been flushed and all data contained in that WAL persisted in table files. This created some interesting implementation details and will create interesting tuning requirements. Make sure to tune your RocksDB such that all column families are regularly flushed. Also, take a look at Options::max_total_wal_size, which can be configured such that stale column families are automatically flushed.

为了更好的理解“Every time a single Column Family is flushed, we create a new WAL (write-ahead log)”，请看下面两个图示：

![image](http://note.youdao.com/noteshare?id=941a207aacc9bb9718ddf9867cc2ac09&sub=199997DA3BEA4C2FBAE939D81DCBF183)

![image](http://note.youdao.com/noteshare?id=1e2638f50086611bbb5f5c8a2f252ba4&sub=9378DEF7B94244FAAF65EFE5E8F0D02C)

## AliSQL数据库内核月报中关于Column Family的介绍
[RocksDB Column Family介绍](http://mysql.taobao.org/monthly/2018/06/09/)

