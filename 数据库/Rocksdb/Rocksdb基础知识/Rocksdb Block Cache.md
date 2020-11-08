# 提纲
[toc]

# 简介
Rocksdb采用Block Cache来作为内存读缓存。同一个Block Cache对象可以被同一进程中多个Rocksdb实例共享，每个Rocksdb实例使用特定大小的容量。Block Cache中存放非压缩的数据块（block），当然（可选的）用户也可以设置另一个Block Cache来存放压缩的数据块。读请求将首先尝试从非压缩的Block Cache中读取，然后是压缩的Block cache中读取，在使用DirectIO的情况下，压缩的BlockCache可以替代操作系统缓存。

关于Block Cache，Rocksdb提供两种实现：LRUCache和ClockCache。两种实现都会被分片，以减轻锁冲突，容量平均分配给这些分片。默认的，每个Block Cache至多切分成64个分片，每一个分片的容量不少于512kB。

# Block Cache使用

默认的，Rocksdb使用8MB大小的LRUCache作为Block Cache。可以调用NewLRUCache()或者NewClockCache()创建Block Cache对象，并设定自定义的容量。用户也可以通过实现Block Cache相关的接口来实现自己的Block Cache。

```
    std::shared_ptr<Cache> cache = NewLRUCache(capacity);
    BlockBasedTableOptions table_options;
    table_options.block_cache = cache;
    Options options;
    options.table_factory.reset(new BlockBasedTableFactory(table_options));
```

或者通过如下方式设定压缩的Block Cache:

```
table_options.block_cache_compressed = another_cache;
```
    
如果BlockBasedTableOptions::block_cache被设置为nullptr，则Rocksdb将会创建默认的Block Cache。如果要禁用BlockCache则：

```
    table_options.no_block_cache = true;
```

# 基于LRU的Block Cache

Out of box, RocksDB will use LRU-based block cache implementation with 8MB capacity. Each shard of the cache maintains its own LRU list and its own hash table for lookup. Synchronization is done via a per-shard mutex. Both lookup and insert to the cache would require a locking mutex of the shard. User can create a LRU cache by calling NewLRUCache(). The function provides several useful options to set to the cache:

默认地，Rocksdb使用容量为8MB的LRUCache作为Block Cache，每一个分片维护它们自己的LRU list和hash tale。每一个分片都有其自己的mutex锁来进行同步控制，插入和查找都在mutex锁的保护下进行。用户可以通过调用NewLRUCache()来创建LRUCache，该方法提供了以下设定选项：

- capacity: Block Cache总容量
 
- num_shard_bits: Block Cache的key中用作分区id的位数，Block Cache将被划分为2^num_shard_bits个分区
 
- strict_capacity_limit: 存在某些情况下，Block Cache实际使用的容量超过设定的容量，比如某些正在进行的读操作或者迭代器将某些数据块pin在Block Cache中，并且被pin在Block Cache中的数据块的总容量超过Block Cache预设定的容量。在实际使用容量超过预设定容量的情况下，如果想进一步插入数据块到该Block Cache中，就取决于strict_capacity_limit这个选项了。如果strict_capacity_limit = false（默认设置为false），则插入会成功，如果放任这种情况，则可能发生非预期的OOM，进而导致数据块crash等。如果strict_capacity_limit = true，则拒绝插入。该选项针对每个分片单独设置

- high_pri_pool_ratio: 为高优先级数据块预留的容量所占的比例

# 基于Clock算法的Block Cache

ClockCache实现了CLOCK算法，每一个分片都维护一个循环链表，一个时钟指针不断地扫描该循环链表，剔除那些unpinned的数据块，与此同时，如果自从上次扫描以来某个数据块被再次使用，则增加该数据块保留在Cache中的机会。查找则采用tbb::concurrent_hash_map。

相较于BlockCache，BlockCache拥有更细的锁粒度，在LRUCache中，在每一个分片查找时，必须持有该分片的mutex锁，因为它可能同时在更新该分片的LRU list。但是从ClockCache中查找时，无需持有该分片的mutex锁，而只需要在concurrent hash map中查找即可，锁粒度更细，只有在插入的时候，需要持有该分片的mutex锁。

用户可以通过调用NewClockCache()来创建ClockCache，想要ClockCache可用，Rocksdb需要链接到Intel的TBB库。该方法提供了以下设定选项：

- capacity: 同LRUCache
 
- num_shard_bits: 同LRUCache
 
- strict_capacity_limit: 同LRUCache
 
# 将Index Blocks和Filter Blocks存放在Block Cache中

默认情况下，index blocks和filter blocks都不被Block Cache管理，而是存放在其它Cache中，这样用户就无法控制使用多少内存来缓存index blocks和filter blocks了。但是Rocksdb允许用户设定将index blocks和filter blocks存放在Block Cache中：

```
    BlockBasedTableOptions table_options;
    table_options.cache_index_and_filter_blocks = true;
```

将index blocks和filter blocks也存放在Block Cache中，这些blocks将会和data blocks形成竞争。虽然index blocks和filter blocks比data blocks访问更频繁，但是Block Cache算法还是会在某些情况下将index blocks和filter blocks给踢出去。而通常情况下，在Block Cache中保留这些index blocks和filter blocks会更具价值，因此Rocksdb也提供了以下选项来解决该问题：

- 设置index blocks和filter blocks具有较高的缓存优先级。目前只在LRUCache中有效，且必须和high_pri_pool_ratio结合使用。如果设置了该选项，则LRUCache中的LRU list将被拆分成两部分，一部分存放高优先级数据块，另外一部分存放低优先级数据块。如果高优先级部分使用容量超过了(BlockCache capacity * high_pri_pool_ratio)，则高优先级部分的LRU list尾部的index blocks或者filter blocks将被移动到低优先级部分的LRU list的头部，一旦index blocks或者filter blocks被移入到低优先级部分的LRU list中，它就要和data blocks竞争（以停留在Block Cache中）了。剔除则从低优先级部分的尾部开始。

- pin_l0_filter_and_index_blocks_in_cache: 将L0的文件中的index blocks和filter blocks pin在Block Cache中，避免它们被剔除。因为L0中的index blocks和filter blocks通常访问比较频繁，且它们可能相对较小，不会消耗太多Block Cache容量。

# Cache模拟器 - SimCache

SimCache用于在Block Cache容量发生改变或者分区数目发生改变的情况下预测缓存命中率，它封装了Rocksdb正在使用的Block Cache对象，同时运行一个模拟指定容量和分区个数的影子LRUCache，并在该影子LRUCache中计算缓存命中率。如下代码用于创建SinCache对象：


```
    // This cache is the actual cache use by the DB.
    std::shared_ptr<Cache> cache = NewLRUCache(capacity);
    // This is the simulated cache.
    std::shared_ptr<Cache> sim_cache = NewSimCache(cache, sim_capacity, sim_num_shard_bits);
    BlockBasedTableOptions table_options;
    table_options.block_cache = sim_cache;
```

SimCache的额外开销不超过设定的sim_capacity的2%。

# 统计

一系列关于Block Cache的统计信息可以通过Options.statistics访问：

    
```
    // total block cache misses
    // REQUIRES: BLOCK_CACHE_MISS == BLOCK_CACHE_INDEX_MISS +
    //                               BLOCK_CACHE_FILTER_MISS +
    //                               BLOCK_CACHE_DATA_MISS;
    BLOCK_CACHE_MISS = 0,
    // total block cache hit
    // REQUIRES: BLOCK_CACHE_HIT == BLOCK_CACHE_INDEX_HIT +
    //                              BLOCK_CACHE_FILTER_HIT +
    //                              BLOCK_CACHE_DATA_HIT;
    BLOCK_CACHE_HIT,
    // # of blocks added to block cache.
    BLOCK_CACHE_ADD,
    // # of failures when adding blocks to block cache.
    BLOCK_CACHE_ADD_FAILURES,
    // # of times cache miss when accessing index block from block cache.
    BLOCK_CACHE_INDEX_MISS,
    // # of times cache hit when accessing index block from block cache.
    BLOCK_CACHE_INDEX_HIT,
    // # of times cache miss when accessing filter block from block cache.
    BLOCK_CACHE_FILTER_MISS,
    // # of times cache hit when accessing filter block from block cache.
    BLOCK_CACHE_FILTER_HIT,
    // # of times cache miss when accessing data block from block cache.
    BLOCK_CACHE_DATA_MISS,
    // # of times cache hit when accessing data block from block cache.
    BLOCK_CACHE_DATA_HIT,
    // # of bytes read from cache.
    BLOCK_CACHE_BYTES_READ,
    // # of bytes written into cache.
    BLOCK_CACHE_BYTES_WRITE,
```
