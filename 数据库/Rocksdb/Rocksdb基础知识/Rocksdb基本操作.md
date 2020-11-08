# 提纲
[toc]

# 基本操作

Rocksdb提供了一个持久化的key-value存储，Keys和Values都是以字节数组存放，Keys根据用户自定义的比较函数（Comparator function）有序存储。

## 打开数据库（Opening A Database）

每一个Rocksdb数据库都拥有一个名称，对应到文件系统的一个目录，数据库的所有内容都存储在该目录下，下面展示了如何打开一个数据库，并且在需要的情况下创建之：

```
  #include <cassert>
  #include "rocksdb/db.h"

  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  assert(status.ok());
  ...
```

如果你想在数据库不存在的情况下提示错误，在rocksdb::DB::Open调用之前添加如下语句即可：

```
  options.error_if_exists = true;
```

如果你的代码是从Leveldb移植过来的，可以通过rocksdb::LevelDBOptions将leveldb::Options对象转换为rocksdb::Options对象。

```
  #include "rocksdb/utilities/leveldb_options.h"

  rocksdb::LevelDBOptions leveldb_options;
  leveldb_options.option1 = value1;
  leveldb_options.option2 = value2;
  ...
  rocksdb::Options options = rocksdb::ConvertOptions(leveldb_options);
```

## RocksDB配置选项（Rocksdb Options）

用户可以显示设置Rocksdb的配置项，也可以通过Option String或者Option Map来设置配置项，参考[Option String and Option Map](https://github.com/facebook/rocksdb/wiki/Option-String-and-Option-Map)

某些配置项甚至可以在Rocksdb运行过程中动态设置. 如下:

```
rocksdb::Status s;
s = db->SetOptions({{"write_buffer_size", "131072"}});
assert(s.ok());
s = db->SetDBOptions({{"max_background_flushes", "2"}});
assert(s.ok());
```

Rocksdb自动在数据库目录下将数据库中用到的配置项以 OPTIONS-xxxxx文件的形式保存，用户可以通过将某些配置项提取到Rocksdb Options File中来实现在数据库重启的情况下依然保留这些配置项的设置。参考[RocksDB Options File](https://github.com/facebook/rocksdb/wiki/RocksDB-Options-File).

## 数据库操作状态（Status）

Rocksdb中大部分可能会出现error的函数都会返回rocksdb::Status类型，你可以通过该返回值检查操作是否成功，也可以打印相关的错误信息：

```
   rocksdb::Status s = ...;
   if (!s.ok()) cerr << s.ToString() << endl;
```

## 关闭数据库（Closing A Database）

当关于数据库的操作完成的时候，直接删除数据库对象即可：

```
  ... open the db as described above ...
  ... do something with db ...
  delete db;
```

## 数据库读写（Reads and Writes）

主要提供了Put, Delete, 和Get方法来更新/查询数据库：

```
  std::string value;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
  if (s.ok()) s = db->Put(rocksdb::WriteOptions(), key2, value);
  if (s.ok()) s = db->Delete(rocksdb::WriteOptions(), key1);
```

Rocksdb也提供了Single Delete方法，该方法在某些特定场景下非常有用。关于Single Delete，请参考[Single Delete](https://github.com/facebook/rocksdb/wiki/Single-Delete)。

每一个Get操作都会产生至少一次memcpy，如果拷贝源在block cache中，则可以通过采用PinnableSlice避免额外的拷贝。关于PinnableSlice，请参考[Pinnable Slice](http://rocksdb.org/blog/2017/08/24/pinnableslice.html)，关于PinnableSlice使用的更多示例参考[Pinnable Slice example](https://github.com/facebook/rocksdb/blob/master/examples/simple_example.cc)。

```
  PinnableSlice pinnable_val;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &pinnable_val);
```

一旦pinnable_val被析构或者在pinnable_val上调用了Reset方法，数据源将被释放。

## 原子更新（Atomic Updates）

可以通过WriteBatch类来实现原子地有序地应用一系列更新：

```
  #include "rocksdb/write_batch.h"
  ...
  std::string value;
  rocksdb::Status s = db->Get(rocksdb::ReadOptions(), key1, &value);
  if (s.ok()) {
    rocksdb::WriteBatch batch;
    batch.Delete(key1);
    batch.Put(key2, value);
    s = db->Write(rocksdb::WriteOptions(), &batch);
  }
```

## 同步写（Synchronous Write）

默认地，leveldb中的写操作是异步的：当将数据写入操作系统的时候写操作就返回，从操作系统内存写入到后端持久存储则是异步的。可以通过设置sync标识，确保直到数据写入持久存储之后写操作才返回。（在Posix系统中，通过fsync(...) 或者 fdatasync(...) 或者 msync(..., MS_SYNC)来实现）。

```
  rocksdb::WriteOptions write_options;
  write_options.sync = true;
  db->Put(write_options, ...);
```

## 异步写（Asynchronous Write）

异步写通常比同步写快上千倍，但是异步写存在一个缺陷：机器宕机可能会导致最后的几个更新操作丢失。注意的是，如果仅仅是写进程宕机则不会产生任何数据丢失。
Asynchronous writes are often more than a thousand times as fast as synchronous writes. The downside of asynchronous writes is that a crash of the machine may cause the last few updates to be lost. Note that a crash of just the writing process (i.e., not a reboot) will not cause any loss since even when sync is false, an update is pushed from the process memory into the operating system before it is considered done.

异步写通常是安全的，比如，在加载大量数据到数据库的过程中机器宕机，可以从头开始重新加载。也可以采用混合模式，即每N个更新被同步一次，如果发生了机器宕机，则从最后一个同步写开始重新加载即可。

WriteBatch为异步写提供了另一个选择，多个更新可以在同一个WriteBatch中，然后一起通过同步写来应用到持久存储中。同步写的开销则被多个写操作分担。

Rocksdb中同样提供了一种针对某个写操作完全禁用Write Ahead Log的方法。通过设置 write_option.disableWAL为true，写操作将不会写入Write Ahead Log这一层，在机器宕机的情况下可能存在数据丢失的风险。

Rocksdb默认地采用较快的fdatasync()来同步文件，如果想使用fsync()，则可以设置Options::use_fsync为true。对于像ext3这样在机器重启时可能丢失文件的文件系统来说，应当设置该标识。

## 并发（Concurrency）

在同一时间一个数据库只能被一个进程打开，Rocksdb通过向操作系统申请一个锁来实现。在同一个进程内部，同一个rocksdb::DB对象可以被多个并发的线程安全共享。比如不同的线程可能同时向数据库中写入数据，或者获取迭代器，或者读取数据，而无需任何外部同步机制（leveldb内部实现中已经自动执行了需要的同步）。但是其它对象（向迭代器Iterator或者WriteBatch）则需要外部同步机制，如果两个线程共享这些类型的对象，它们必须通过自己的锁协议来实现访问保护。更多的细节，请参考公共头文件中关于这些对象的使用说明。

## 合并操作符（Merge Operators）

Merge Operators为read-modify-write操作提供了更有效地支持。更多的关于Merge Operators的接口和实现请参考这里：
[Merge Operator](https://github.com/facebook/rocksdb/wiki/Merge-Operator)

[Merge Operator Implementation](https://github.com/facebook/rocksdb/wiki/Merge-Operator-Implementation)

[Rocksdb Merge Operator实现](http://note.youdao.com/noteshare?id=e18a86d44f59a44ff74ce970e50fe443&sub=85A5632AB10843B7A70110C6AFCD330C)

## 迭代器（Iterator）

下面的示例展示了如何打印数据库中所有的键值对：

```
  rocksdb::Iterator* it = db->NewIterator(rocksdb::ReadOptions());
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    cout << it->key().ToString() << ": " << it->value().ToString() << endl;
  }
  assert(it->status().ok()); // Check for any errors found during the scan
  delete it;
```

下面的示例则展示了如何处理[start, limit)范围内的键：

```
  for (it->Seek(start);
       it->Valid() && it->key().ToString() < limit;
       it->Next()) {
    ...
  }
  assert(it->status().ok()); // Check for any errors found during the scan
```

也可以逆序处理数据库中的每一个键（逆序处理可能比顺序处理要慢）：  

```
  for (it->SeekToLast(); it->Valid(); it->Prev()) {
    ...
  }
  assert(it->status().ok()); // Check for any errors found during the scan
```


这里展示了如何逆序处理(limit, start]范围内的键：

```
  for (it->SeekForPrev(start);
       it->Valid() && it->key().ToString() > limit;
       it->Prev()) {
    ...
  }
  assert(it->status().ok()); // Check for any errors found during the scan
```

关于SeekForPrev，请参考[SeekForPrev](https://github.com/facebook/rocksdb/wiki/SeekForPrev).

关于迭代器的错误处理，迭代器配置选项和最佳实践，请参考[Iterator](https://github.com/facebook/rocksdb/wiki/Iterator).

关于迭代器的实现细节，请参考：

[Iterator's Implementation](https://github.com/facebook/rocksdb/wiki/Iterator-Implementation).

[Rocksdb迭代器实现](http://note.youdao.com/noteshare?id=7ccb569aa578b9cf5c714c676e4eba1c&sub=D81A764A19AA445994A97B44A2746D67)

## 快照（Snapshot）

快照提供了关于键值存储状态的一致性只读视图。如果ReadOptions::snapshot不为NULL，则表明本次读操作将从某个特定版本的数据库视图中读取。如果ReadOptions::snapshot为NULL，则本次读操作将从当前隐示的快照上读取。

通过DB::GetSnapshot()方法创建快照：

```
  rocksdb::ReadOptions options;
  options.snapshot = db->GetSnapshot();
  ... apply some updates to db ...
  rocksdb::Iterator* iter = db->NewIterator(options);
  ... read using iter to view the state when the snapshot was created ...
  delete iter;
  db->ReleaseSnapshot(options.snapshot);
```

如果某个快照不再被使用，则可以使用DB::ReleaseSnapshot接口来释放它。

## Slice

在上面的代码中it->key()和it->value()调用的返回值都是rocksdb::Slice类型的实例。Slice是一个简单的结构体，它包含一个长度信息和一个外部的字节数组的指针。相对于返回std::string，返回一个Slice是一个相对轻量级的操作，因为它无需拷贝可能较大的键或者值。另外Rocksdb方法绝不会返回C语言类型的空字符结束的字符串（null-terminated C-style strings），因为Rocksdb中允许键或者值中包含'\0' 字符。

C++字符串或者以空字符结束的C语言类型的字符串都可以很容易地转换为Slice：

```
   rocksdb::Slice s1 = "hello";

   std::string str("world");
   rocksdb::Slice s2 = str;
```

Slice也可以轻松转换为C++字符串：

```
   std::string str = s1.ToString();
   assert(str == std::string("hello"));
```

在使用Slice的时候一定要小心，因为需要调用者来确保在使用Slice的过程中，Slice中所指向的外部字节数组依然存在。下面的示例就是一个错误的示例：

```
   rocksdb::Slice slice;
   if (...) {
     std::string str = ...;
     slice = str;
   }
   Use(slice);
```

当if语句结束它的作用域的时候，str将会被销毁，slice中所指向的外部字节数组，即str，将消失。

## 事务（Transactions）

Rocksdb支持multi-operation transactions。

关于事务，请参考:

[Transactions](https://github.com/facebook/rocksdb/wiki/Transactions)

[Rocksdb事务](http://note.youdao.com/noteshare?id=064a45ccab5853b952cbb1a6d29d69e7&sub=337C69BF9A5B4BD28E2EACE16A9281E7)

## 比较器（Comparators）

之前的示例中采用的是默认的键排序函数，即按照字典序排序。当然也可以在打开数据库的时候指定自定义的比较器。比如，假设数据库中的每一个键都由两个数字组成，我们想优先按照第一个数字的大小排序，如果两个键的第一个数字相等，则按照第二个数字的大小进行排序。那么需要定义一个继承自rocksdb::Comparator的子类来实现该规则：

```
  class TwoPartComparator : public rocksdb::Comparator {
   public:
    // Three-way comparison function:
    // if a < b: negative result
    // if a > b: positive result
    // else: zero result
    int Compare(const rocksdb::Slice& a, const rocksdb::Slice& b) const {
      int a1, a2, b1, b2;
      ParseKey(a, &a1, &a2);
      ParseKey(b, &b1, &b2);
      if (a1 < b1) return -1;
      if (a1 > b1) return +1;
      if (a2 < b2) return -1;
      if (a2 > b2) return +1;
      return 0;
    }

    // Ignore the following methods for now:
    const char* Name() const { return "TwoPartComparator"; }
    void FindShortestSeparator(std::string*, const rocksdb::Slice&) const { }
    void FindShortSuccessor(std::string*) const { }
  };
  
  TwoPartComparator cmp;
  rocksdb::DB* db;
  rocksdb::Options options;
  options.create_if_missing = true;
  options.comparator = &cmp;
  rocksdb::Status status = rocksdb::DB::Open(options, "/tmp/testdb", &db);
  ...
```

## 列簇（Column Families）
Colunm Families提供了一种逻辑上划分数据库的方式。

关于Column Families，请参考[Rocksdb Column Families](http://note.youdao.com/noteshare?id=67f0fde53640d8a581b32c569039e78b&sub=64E31B410ABB4AFB8E4F7387E07BEF59).


## 批量加载（Bulk Load）
可以创建和注入SST files来批量加载大量数据到数据库中，与此同时对业务流量产生较小影响。

关于创建和注入SST files，请参考[Creating and Ingesting SST files](https://github.com/facebook/rocksdb/wiki/Creating-and-Ingesting-SST-files).

## 备份和检查点（Backup and Checkpoint）

用户可以通过备份来周期性地将增量数据备份到其它文件系统（比如HDFS或者S3），并且在需要的时候恢复它们。

关于备份，请参考[Backup](https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F).

检查点则支持在单独的目录中对正在运行中的数据库执行快照。如果可能的话，检查点对应的目录中的文件都是硬链接，而不是真正的拷贝，因此它是一个相对轻量级的操作。

关于检查点，请参考[Checkpoint](https://github.com/facebook/rocksdb/wiki/Checkpoints).

## I/O

默认地，Rocksdb的IO都会经过操作系统的page cache，用户可以选择旁路page cache，使用Direct IO。通过设置IO限速，可以限制写文件的速度，以便为读IO预留一部分带宽。

关于IO，请参考[IO](https://github.com/facebook/rocksdb/wiki/IO).

关于IO限速，请参考[Rate limiter](https://github.com/facebook/rocksdb/wiki/Rate-Limiter).

## MemTable and Table factories

默认地，Rocksdb在内存中以skiplist memtable的形式保存数据，在硬盘中则以特定的table保存数据。关于table格式，请参考[RocksDB Table Format](https://github.com/facebook/rocksdb/wiki/Rocksdb-Table-Format).

因为Rocksdb的目标之一就是支持各个组件的插拔，它支持各种不同的memtable和table格式。可以通过设置Options::memtable_factory提供自己的memtable，或者通过设置Options::table_factory提供自己的table。对于可用的memtable集合，请参考rocksdb/memtablerep.h，对于可用的table集合，请参考rocksdb/table.h。

Since one of the goals of RocksDB is to have different parts of the system easily pluggable, we support different implementations of both memtable and table format. You can supply your own memtable factory by setting Options::memtable_factory and your own table factory by setting Options::table_factory. For available memtable factories, please refer to rocksdb/memtablerep.h and for table factores to rocksdb/table.h. These features are both in active development and please be wary of any API changes that might break your application going forward.

更多关于memtable，可以参考[这里](https://github.com/facebook/rocksdb/wiki/RocksDB-Basics#memtables)或者[这里](https://github.com/facebook/rocksdb/wiki/MemTable).

## 块大小（Block size）

Rocksdb将连续的keys归集在同一个block中，并且以这些block作为向持久存储写入数据或者从持久存储读取数据的单元。可以通过设置Options::block_size选项来更改块大小。

## 写缓冲（Write buffer）
Options::write_buffer_size 指定了在将写缓冲中的数据转换为SST files之前写缓冲区中可以存放的最大数据集大小。

Options::max_write_buffer_number 指定了内存中可以容纳的最大写缓冲区的个数。

Options::min_write_buffer_number_to_merge 指定了在将写缓冲中的数据写入SST files之前至少要合并的写缓冲的个数。如果被设置为1，则所有的写缓冲都以单独的文件的形式被刷新到L0中，这将带来读放大，因为一个读请求需要检查所有这些文件。并且如果各写缓冲中存在重复的记录的话，写缓冲区合并将减少写入SST files中的数据总量。默认值是1.

## 压缩（Compression）

每一个块在写入到持久存储之前都被单独压缩，默认地，压缩是开启的，因为默认的压缩方式非常高效，并且对于无法压缩的数据会自动禁用压缩功能。很少有情况需要禁用压缩，但是如果测试表明禁用压缩可以提升性能的话，就需要禁用压缩：

```
  rocksdb::Options options;
  options.compression = rocksdb::kNoCompression;
  ... rocksdb::DB::Open(options, name, ...) ....
```

## 缓存（Cache）

数据库内容是以一系列文件存储的，每一个文件存放连续的压缩的数据块。如果options.block_cache不为NULL，则Rocksdb采用该Block Cache来缓存经常被使用的非压缩数据块。而对于压缩的数据，Rocksdb采用操作系统的file cache来缓存。

```
  #include "rocksdb/cache.h"
  rocksdb::BlockBasedTableOptions table_options;
  table_options.block_cache = rocksdb::NewLRUCache(100 * 1048576); // 100MB uncompressed cache

  rocksdb::Options options;
  options.table_factory.reset(rocksdb::NewBlockBasedTableFactory(table_options));
  rocksdb::DB* db;
  rocksdb::DB::Open(options, name, &db);
  ... use the db ...
  delete db
```

在执行批量读取的时候，应用可能希望禁用cache功能，以免cache中充满那些不应该被缓存的数据。可以采用基于iterator的配置项来达到该目的：

```
  rocksdb::ReadOptions options;
  options.fill_cache = false;
  rocksdb::Iterator* it = db->NewIterator(options);
  for (it->SeekToFirst(); it->Valid(); it->Next()) {
    ...
  }
```

也可以通过设置options.no_block_cache为true来禁用Block Cache。

关于Block Cache，请参考:

[Block Cache](https://github.com/facebook/rocksdb/wiki/Block-Cache).

[Rocksdb Block Cache](http://note.youdao.com/noteshare?id=eb4857e72de717ec9ff35314724ec617&sub=F8FEBE1E85C24C7CAD9D44F56F03766C).

## 键布局（Key Layout）

内存缓存和硬盘之间数据传输的基本单元是block，相邻的keys通常被存放于同一个block中。因此应用可以通过将那些经常一同访问的keys放在一起，将那些不经常访问的keys放在另一个单独的地方。

比如，通过Rocksdb实现一个简单的文件系统，在Rocksdb中存放两种类型的键值对：

```
   //metadata
   filename -> permission-bits, length, list of file_block_ids
   //data
   file_block_id -> data
```

可以将filename这个键以一个字符（比如'/'）作为前缀，将file_block_id这个键以另一个字符（比如'0'）作为前缀，以便在只需要扫描metadata的情况下无需读取或者缓存较大的data部分。

## 过滤器（Filters）

因为Rocksdb数据组织方式的缘故，一个Get()调用可能会涉及硬盘上多次读取，可以采用可选的FilterPolicy机制来相当程度的减少硬盘读取次数。

```
   rocksdb::Options options;
   //10 bits per key bloom filter
   options.filter_policy = NewBloomFilter(10, false /* use block based filter */);
   rocksdb::DB* db;
   rocksdb::DB::Open(options, "/tmp/testdb", &db);
   ... use the database ...
   delete db;
   delete options.filter_policy;
```

如果采用自定义的comparator，则必须确保采用的filter policy和comparator兼容。

## 校验和（Checksums）

Rocksdb中所有存放于文件系统中的数据都有一个校验和，有两个单独的选项用于控制数据校验的紧迫程度：

ReadOptions::verify_checksums 对于所有从文件系统读取的数据，强制检查校验和。默认开启。

Options::paranoid_checks 在打开数据库的时候可以设置该选项为true，这样在数据库运行过程中如果检测到任何的内部不一致，就提示错误。默认关闭该选项。

如果出现了任何不一致（即使在数据库打开阶段），将会调用Rocksdb::RepairDB方法来尽可能恢复数据。

## 合并（Compaction）

通过合并，Rocksdb可以清理旧版本的数据，并且确保以较优的方式读取数据。

关于Compaction，请参考:

[Compaction](https://github.com/facebook/rocksdb/wiki/Compaction).

[Rocksdb Compaction](http://note.youdao.com/noteshare?id=b45e45b4625697c5615b60e7886b3bda&sub=21652369655C4F9CA578B18D44BC8DC8).

## key范围占用空间近似大小（Approximate Sizes）

GetApproximateSizes方法可以用于获取一个或者多个key范围内的数据占用的文件系统大小。

```
   rocksdb::Range ranges[2];
   ranges[0] = rocksdb::Range("a", "c");
   ranges[1] = rocksdb::Range("x", "z");
   uint64_t sizes[2];
   rocksdb::Status s = db->GetApproximateSizes(ranges, 2, sizes);
```

## 环境（Environment）

Rocksdb中所有的文件操作（和其它操作系统调用）都通过rocksdb::Env对象进行路由。专用的客户可能希望采用自己的Env来实现更好的控制。比如，应用可能希望在IO路径中引入人为的延迟来限制Rocksdb对系统中其它活动的影响。

```
class SlowEnv : public rocksdb::Env {
    .. implementation of the Env interface ...
  };

  SlowEnv env;
  rocksdb::Options options;
  options.env = &env;
  Status s = rocksdb::DB::Open(options, ...);
```

## 可管理性（Manageability）

为了更好的调优应用，统计信息非常有用。可以通过设置Options::table_properties_collectors或者Options::statistics选项来收集相关统计信息。关于统计([Statistics](https://github.com/facebook/rocksdb/wiki/Statistics))相关，请参考rocksdb/table_properties.h and rocksdb/statistics.h。

也可以采用[Perf Context and IO Stats Contex](https://github.com/facebook/rocksdb/wiki/Perf-Context-and-IO-Stats-Context)来跟踪单个请求。

对于一些内部事件的回调，用户也可以注册自己的[EventListener](https://github.com/facebook/rocksdb/wiki/EventListener)。

## 清理WAL文件（Purging WAL files）

默认地，在应用不再使用它们的时候，旧的WAL文件（Write-ahead logs）将被自动删除。Rocksdb提供了配置项来归档这些WAL文件并且延迟删除，要么根据TTL（Tiem To Live），要么根据大小限制。

这些配置项分别是Options::WAL_ttl_seconds and Options::WAL_size_limit_MB。下面描述了如何使用它们：

如果两者都被设置为0，则WAL文件将被尽早删除，并且不会被归档。

如果WAL_ttl_seconds是0，而WAL_size_limit_MB不是0，则每隔10分钟WAL文件被检查一次，如果WAL文件总大小超过了WAL_size_limit_MB，则WAL文件将从最早的开始被删除，直到总大小小于WAL_size_limit_MB。所有空文件将被删除。

如果WAL_ttl_seconds不是0，而WAL_size_limit_MB是0，则每隔(WAL_ttl_seconds / 2)的时间WAL文件被检查一次，对于那些早于WAL_ttl_seconds的文件将被删除。

如果两者都不为0，则每隔10分钟WAL文件被检查一次，并且同时执行两个检查，但是优先检查TTL。

## Other Information

配置Rocksdb:

[Set Up Options](https://github.com/facebook/rocksdb/wiki/Set-Up-Options)

[Some detailed Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide)

关于Rocksdb的实现细节可以参考以下文档:

[RocksDB Overview and Architecture](https://github.com/facebook/rocksdb/wiki/RocksDB-Basics)

[Rocksdb Table Format](https://github.com/facebook/rocksdb/wiki/Rocksdb-Table-Format)

[Rocksdb SST file格式](http://note.youdao.com/noteshare?id=7f049a3df9fded01db83a003a4e4111e&sub=9EE3830978A2414A89D524FDB4A4F143)

