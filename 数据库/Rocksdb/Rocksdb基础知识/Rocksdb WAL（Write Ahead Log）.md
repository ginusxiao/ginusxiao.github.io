# 提纲
[toc]

# WAL基础
## 概览

Rocksdb的每一次写都写入到两个位置：1）基于内存的MemTable（MemTable中的数据会被刷到SST files中）；2）基于硬盘的WAL（write ahead log）。在故障情况下，可以利用WAL来恢复MemTable中的数据，进而恢复数据库。默认配置下，Rocksdb会在每一次写入WAL后调用fflush()来确保即使在进程重启的情况下，数据一致性依然可以得到保证。

## WAL生命周期

通过一个例子来看WAL的生命周期。在调用DB::Open打开一个数据库的时候，Rocksdb会在硬盘上创建一个新的WAL文件用于持久化写操作：

```
    DB* db;
    std::vector<ColumnFamilyDescriptor> column_families;
    column_families.push_back(ColumnFamilyDescriptor(
        kDefaultColumnFamilyName, ColumnFamilyOptions()));
    column_families.push_back(ColumnFamilyDescriptor(
        "new_cf", ColumnFamilyOptions()));
    std::vector<ColumnFamilyHandle*> handles;
    s = DB::Open(DBOptions(), kDBPath, column_families, &handles, &db);
```

分别向两个column families中添加一些键值对：

```
    db->Put(WriteOptions(), handles[1], Slice("key1"), Slice("value1"));
    db->Put(WriteOptions(), handles[0], Slice("key2"), Slice("value2"));
    db->Put(WriteOptions(), handles[1], Slice("key3"), Slice("value3"));
    db->Put(WriteOptions(), handles[0], Slice("key4"), Slice("value4"));
```

至此，WAL中记录了所有的写操作，WAL会记录后续的写操作直到它的大小达到DBOptions::max_total_wal_size。

如果用户要flush “new_cf”这一column family相关的数据，则：1）该column family的数据（key1和key3）被刷到SST file中；2）创建一个新的WAL文件，后续关于任何colum_family的写操作都写入到该WAL中；3）旧的WAL文件将不会写入任何新的写操作，但是该旧的WAL文件也不会被立即删除。
    
```
    db->Flush(FlushOptions(), handles[1]);
    // key5 and key6 will appear in a new WAL
    db->Put(WriteOptions(), handles[1], Slice("key5"), Slice("value5"));
    db->Put(WriteOptions(), handles[0], Slice("key6"), Slice("value6"));
```

至此，Rocksdb中将会有两个WAL文件，较旧的WAL中包含key1到key4的写操作，较新的WAL中包含key5和key6.因为较旧的WAL文件中包含“default”这一column family的数据，所以它不能被删除，只有当用户决定将“default”这一column family的数据刷到SST file中之后，该较旧的WAL文件才能被删除。

```
    db->Flush(FlushOptions(), handles[0]);
    // The older WAL will be archived and purged separetely
```

总结起来说，WAL文件在以下时机被创建：1）打开一个新的数据库；2）某个column family被刷到SST files中。WAL文件只有当它所包含的所有column families在该文件中的最大序列号对应的数据被刷到SST files之后才能被删除，也就是说该WAL文件中所有数据都持久化之后，该WAL文件才能被删除（为了副本的缘故，比如在Transaction log iterator中，WAL文件可能会被归档，并在稍晚的时候被删除）。

## WAL配置

在options.h中有以下关于WAL的配置选项：

**DBOptions::wal_dir**

设置Rocksdb存储WAL文件的目录，允许WALs和数据分别存放在不同的位置。

**DBOptions::WAL_ttl_seconds, DBOptions::WAL_size_limit_MB**

这两个选项影响归档的WAL文件何时被删除。

**DBOptions::max_total_wal_size**

限制WAL文件的总大小，当WAL文件总大小超过该值，Rocksdb将会强制将某些colum families的数据刷到SST files中，以便删除那些较旧的WAL。

**DBOptions::avoid_flush_during_recovery**

在recovery过程中不进行flush。

**DBOptions::manual_wal_flush**

该选项决定WAL在每次写之后自动执行flush，还是通过调用FlushWAL手动执行。

**DBOptions::wal_filter**

通过该选项，用户可以提供一个过滤器对象，被恢复期间的WAL文件处理所调用。

**WriteOptions::disableWAL**

如果用户依赖于其它的logging机制或者不关心数据丢失，则可以禁用WAL。

## WAL日志格式
### WAL日志文件格式
WAL日志文件由一系列变长的日志记录组成，日志记录按照kBlockSize大小(32KB)进行分组。如果当前分组(Block)中的剩余空间不足以容纳某条日志记录，则该分组中剩余空间采用空数据填充。Reader和Writer均以kBlockSize作为读写单位。

           +-----+-------------+--+----+----------+------+-- ... ----+
     File  | r0  |        r1   |P | r2 |    r3    |  r4  |           |
           +-----+-------------+--+----+----------+------+-- ... ----+
           <--- kBlockSize ------>|<-- kBlockSize ------>|
    
      rn = variable size records
      P = Padding
      
### 日志记录格式
每条日志记录的格式如下：

    +---------+-----------+-----------+--- ... ---+
    |CRC (4B) | Size (2B) | Type (1B) | Payload   |
    +---------+-----------+-----------+--- ... ---+
    
    CRC = 在Payload上运行CRC算法计算出来的32 bits的值
    Size = Payload的长度(只占用2Byte，最大只能描述64KB，是因为一个记录是一定不会跨不同Block的，
           而一个Block的大小为32KB，所以一个Record的Payload长度一定不大于32KB - 7B的，所以
           Size部分只占用2Byte足够)
    Type = 记录的类型，主要用于标记哪些日志记录是率属于一条日志的(一条日志可能被拆分为多个日志记        录)，包括以下记录类型：
           Kzerotype：为预配置文件的保留类型
           kFullType：表示日志记录中包含完整的用户数据
           kFirstType：表示被拆分存储的第一个日志记录
           kMiddleType：表示被拆分存储的中间部分日志记录
           kLastType：表示被拆分存储的最后一个日志记录
           kRecyclableFullType：跟kFullType类似，但是当前的日志文件是回收利用的旧的日志文件
           kRecyclableFirstType：跟kFirstType类似，但是当前的日志文件是回收利用的旧的日志文件
           kRecyclableMiddleType：跟kMiddleType类似，但是当前的日志文件是回收利用的旧的日志文件
           kRecyclableLastType：跟kLastType类似，但是当前的日志文件是回收利用的旧的日志文件
    Payload = 字节流
      
每一个block的最后6个字节都不可能作为某个日志记录的起点，因为每条日志记录的（checksum + length + type）这三部分就至少需要7个字节。如果当前block中恰好剩余7个字节，且当前被添加的日志记录的data部分不为空，那么Writer必须写一个FIRST类型的日志记录（但是该FIRST记录中不包含任何用户数据），以填充当前block的末尾部分，然后在后续block中写入用户数据。

为了更好的理解kFullType, kFirstType, kMiddleType和kLastType，这里举个例子。假设有这样一系列用户数据，且block大小为32KB：
```
   A: length 1000
   B: length 97270
   C: length 8000
   
A将以FULL类型的日志记录存放于第一个block中。

B将被拆分成3个片段：第一个片段存放于第一个block中剩余空间中，第二个片段占用第二个block的全部空间，第三个片段则占用第三个block的前面一部分空间。最后第三个block中将剩余6个字节，这6个字节空间作为第三个block的尾部，将被保留为空。

C将以FULL类型的日志记录存放于第四个block中。
```

### Reader/Writer
WAL manager为读写这些WAL文件提供了一层抽象，在WAL内部通过Reader和Writer来进行WAL文件读写。Writer借助WriteableFile接口向WAL文件中添加日志记录，Reader借助SequentialFile接口从WAL文件中读取日志记录。WriteableFile接口和SequentialFile接口均用于提供介质相关的实现逻辑的抽象。



# WAL源码分析


