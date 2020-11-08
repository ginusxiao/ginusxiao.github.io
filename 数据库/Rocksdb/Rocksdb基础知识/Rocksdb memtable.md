# 提纲
[toc]

# Memtable基础
MemTable是一个用于在flush数据到SST files之前保存数据的内存结构，写请求总是现将数据写入MemTable中，读请求也总是先在MemTable中查找。一旦MemTable写满了，它就切换为一个Immutable MemTable，并且被另一个新的MemTable代替。后台线程负责将MemTable中的数据刷到SST files中，一旦该MemTable中的数据全部刷到SST files中了，该MemTable就可以销毁了。

关于MemTable的最重要的属性如下：

- memtable_factory: MemTable工厂对象，藉此用户可以改变MemTable的底层实现，并且提供实现相关的选项。

- write_buffer_size: 每一个MemTable的大小。 

- db_write_buffer_size: MemTable可以使用的总内存大小。

- write_buffer_manager: 除了通过db_write_buffer_size指定MemTable总大小外，用户也可以藉此选项来控制MemTable的总大小。它会覆盖db_write_buffer_size。

- max_write_buffer_number: 内存中可以存在的尚未Flush的MemTable最大数目。

默认的MemTable实现是基于调表（skiplist），用户也可以使用其它类型的MemTable实现，比如HashLinkList，HashSkipList或者Vector。

## Skiplist MemTable

基于skiplist的MemTable对于读写，随机访问和顺序扫描都能提供较好的性能。另外，它能提供其它MemTable实现尚不支持的功能，比如并发插入和带暗示的插入（Concurrent Insert and Insert with Hint）。

## HashSkiplist和HashLinkList MemTable

HashSkipList是一个hash table，hash table中的每一个桶（bucket）都是一个skiplist。

HashLinkList也是一个hash table，hash table中的每一个桶（bucket）都是一个linklist。

这两种MemTable，都是为了减少查询过程中比较的次数。

在插入或者查找某个key的时候，通过Options.prefix_extractor获取key的前缀，然后通过该前缀来找到hash bucket。在hash bucket内部，比较则是基于整个key，跟基于SkipList的MemTable一样。

基于Hash的MemTable，包括HashSkipList和HashLinkList MemTable的最大缺陷在于执行跨多个前缀的扫描的时候需要拷贝和排序，这将消耗大量内存，同时非常慢。

## Flush

有三种情形会触发MemTable Flush：

- 在写入之后MemTable大小超过write_buffer_size。

- 总的MemTable占用的空间超过db_write_buffer_size，或者write_buffer_manager发送了flush信号。在这种情况下，占用空间最大的MemTable将被flush。

- 总的WAL文件大小超过max_total_wal_size，这种情况下，具有最老数据的MemTable将被flush，以便WAL可以回收这些数据占用的空间。

从上述MemTable Flush的触发条件可以看到，一个MemTable可能在它写满之前被flush，这是产生的SST file可能比MemTable小的原因之一，另一个原因是压缩，因为MemTable中的数据是未压缩的。

## Concurrent Insert

Without support of concurrent insert to memtables, concurrent writes to RocksDB from multiple threads will apply to memtable sequentially. Concurrent memtable insert is enabled by default and can be turn off via allow_concurrent_memtable_write option, although only skiplist-based memtable supports the feature.
如果MemTable不支持并发插入，则Rocksdb的并发写在MemTable上只能是串行的。虽然只有SkipList MemTable支持并发插入，MemTable的并发插入默认是开启的，可以通过设置allow_concurrent_memtable_write选项来关闭之。

## Insert with Hint

## In-place Update

## Comparison

MemTable类型 | SkipList | HashSkipList | HashLinkList | Vector
---|---|---|---|---
Optimized Use Case | General | Range query within a specific key prefix | Range query within a specific key prefix and there are only a small number of rows for each prefix | Random write heavy workload
Index type | binary search | hash + binary search | hash + linear search | linear search
Support totally ordered full db scan? | naturally | very costly (copy and sort to create a temporary totally-ordered view) | very costly (copy and sort to create a temporary totally-ordered view) | very costly (copy and sort to create a emporary totally-ordered view)
Memory Overhead | Average (multiple pointers per entry) | High (Hash Buckets + Skip List Metadata for non-empty buckets + multiple pointers per entry) | Lower (Hash buckets + pointer per entry) | Low (pre-allocated space at the end of vector)
MemTable Flush | Fast with constant extra memory | Slow with high temporary memory usage | Slow with high temporary memory usage | Slow with constant extra memory
Concurrent Insert | Support | Not support | Not support | Not support
Insert with Hint | Support (in case there are no concurrent insert) | Not support | Not support | Not support

# MemTable源码分析
## 待续...
## 待续...
