# 提纲
[toc]

# 简介
Rocksdb同时支持乐观并发控制和悲观并发控制。其中悲观并发控制采用锁来实现事务隔离，悲观并发控制中默认写策略是WriteCommitted，该策略下只在事务commit之后才会写入数据库中，比如memtable中。这种策略实现简单，但是限制了事务吞吐以及可以支持的事务隔离级别。所以，Rocksdb也提供了其它的写策略：WritePrepared和WriteUnprepared。

# WriteCommitted
对于WriteCommitted写策略，只有在事务提交之后，数据才被写入memtable。这将极大的简化读取路径，因为任何其它事务读取到的数据一定是提交了的，但是这意味着在提交之前写的数据都必须保存在内存缓冲区中，对于大事务来说，内存压力可想而知。而且这种写策略不能提供那些较弱的隔离级别，比如ReadUncommitted。


# WritePrepared
2PC包括3个阶段：写阶段 - ::Put被调用，准备阶段 - ::Prepare被调用（该阶段之后数据库承诺：如果被请求则提交事务），提交阶段 - ::Commit被调用（此后，事务写对所有的读者可见）。为了解决WriteCommitted写策略的不足，应当在2PC（两阶段提交）中尽可能早的阶段写数据到memtable，即要么在写阶段，要么早准备阶段，如果在写阶段则对应WriteUnprepared写策略，如果在准备阶段则对应WritePrepared写策略。对于WritePrepared和WriteUnprepared这两种写策略，如果其它事务读取数据，该事务需要知道要读取的数据是否已被提交了，如果已经提交了，那么是否在本事务之前提交。

对于WritePrepared，事务依然将写数据存放在内存中的WriteBatch对象中，当::Prepare被调用，它将内存中的WriteBatch写入WAL和memtable中，对于WriteBatch中的所有的键值对采用同一个序列号，即prepare_seq，该序列号也作为事务的标识符，当::Commit被调用，它在WAL中写入一个提交标记，提交也会对应一个序列号，即commit_seq，该序列号将作为事务提交的时间戳。Rocksdb采用一个基于内存的数据结构CommitCache来记录prepare_seq到commit_seq的映射。当某个事务从数据库中读取数据的时候，会携带prepare_seq，Rocksdb在CommitCache中查找对应的commit_seq，并检查该commit_seq是否在它的read snapshot中。

## CommitCache
CommitCache是一个固定大小的无锁数组，它缓存最近的提交记录。

插入某个提交记录：

CommitCache[prepare_seq % array_size] = <prepare_seq, commit_seq>

每次插入操作可能会剔除之前的某个提交记录，CommitCache会记录这些被剔除的提交记录中具有最大序列号的记录：max_evicted_seq。当在CommitCache中查找的时候，如果给定的prepare_seq > max_evicted_seq，且在CommitCache中没有找到，则认为该prepare_seq对应的事务尚未提交。如果在CommitCache中找到了该prepare_seq对应的事务Txn_c的commit_seq，则认为该事务Txn_c已经提交，对于任何其它事务Txn_r的读，假设事务Txn_r的read snapshot的序列号为snap_seq，且满足Txn_c的commit_seq <= Txn_r的snap_seq，则事务Txn_r的读可以读取到Txn_c的数据。

上述插入某个记录的方法，存在一个缺陷：假如某个序列号prepare_seq对应的事务尚未提交，但是max_evicted_seq比prepare_seq大，那么就会认为prepare_seq对应的事务已经提交。为了应对该情况，Rocksdb采用了另一个数据结构：PrepareHeap，它是一个堆，在::Prepare的时候prepare_seq被插入，在::Commit的时候prepare_seq被删除。随着max_evicted_se的增长，如果出现max_evicted_seq大于PrepareHeap中的最小prepare_seq，就将从该最小prepare_seq到max_evicted_seq的序列号pop出来存放到OldPrepared中。在判断某个prepare_seq对应的事务是否已经提交的时候，必须事先检查该OldPrepared。

对于只读事务来说，它可能花费很长时间，比如某个事务正在执行备份数据库工作，以至于它不能假设一份它所依赖的较老的数据还在它的read snapshots中，因为它的read snapshots可能因为太旧而被剔除了。因此，如果存在prepare_seq <= snap_seq < commit_seq，则必须为该snapshot保存它所依赖的从CommitCache中剔除的记录<prepare_seq, commit_seq>。Rocksdb采用_OldCommit_Map来保存这些记录，在_OldCommit_Map中会记录从snap_seq到它所依赖的一系列prepare_seq的映射。在read snapshot被释放的时候，_OldCommit_Map会被回收。


## Rollback

为了终止某个事务，对于每一个写入的键值对，写入另一个键值对来取消之前的写入。悲观并发控制中基于锁的写写冲突检测可以确保同一时刻只有一个写请求，因此只需要去检查当前的写请求所处的状态并确定它应该回滚到哪个状态。如果能够读取到该事务开始前的关于同一个key数据，就将该数据直接写入即可，如果没能读取到该事务开始前的关于同一个key数据，则写入一条删除记录。用于回滚的新的数据将在另一个事务中提交，并且被终止的事务的prepare_seq也要从PrepareHeap中移除。


## Atomic Commit

During a commit, a commit marker is written to the WAL and also a commit entry is added to the CommitCache. These two needs to be done atomically otherwise a reading transaction at one point might miss the update into the CommitCache but later sees that. We achieve that by updating the CommitCache before publishing the sequence number of the commit entry. In this way, if a reading snapshot can see the commit sequence number it is guaranteed that the CommitCache is already updated as well. This is done via a PreReleaseCallback that is added to ::WriteImpl logic for this purpose.

## Flush/Compaction

We provide a IsInSnapshot(prepare_seq, commit_seq) interface on top of the CommitCache and related data structures, which can be used by the reading transactions. Flush/Compaction threads are also considered a reader and use the same API to figure which versions can be safely garbage collected without affecting the live snapshots.

## Optimization
### Lock-free CommitCache

Every read from recent data results into a lookup into CommitCache. It is therefore vital to make CommitCache efficient for reads. Using a fixed array was already a concious design decision to serve this purpose. We however need to further avoid the overhead of synchronization for reading from this array. To achieve this, we make CommitCache an array of std::atomic<uint64_t> and encode <prepare_seq, commit_seq> into the available 64 bits. The reads and writes from the array are done with std::memory_order_acquire and std::memory_order_release respectively. In a x86_64 architecture these operations are translated into simple reads and writes into memory thanks to the guarantees of the hardware cache coherency protocol. We have other designs for this data structure and will explore them in future.

To encode <prepare_seq, commit_seq> into 64 bits we use this algorithm: i) the higher bits of prepare_seq is already implied by the index of the entry in the CommitCache; ii) the lower bits of prepare seq are encoded in the higher bits of the 64-bit entry; iii) the difference between the commit_seq and prepare_seq is encoded into the lower bits of the 64-bit entry.

### Less-frequent updates to max_evicted_seq

Normally max_evicted_seq is expected to be updated upon each eviction from the CommitCache. Although updating max_evicted_seq is not necessarily expensive, the maintenance operations that comes with it are. For example it requires holding a mutex to verify the top in PrepareHeap (although this can be optimized to be done without a mutex). More importantly it involves holding the db mutex for fetching the list of live snapshots from the DB since maintaining OldCommit depends on the list of live snapshots up to max_evicted_seq. To reduce this overhead, upon each update to max_evicted_seq we increase its value further by 1% of the CommitCache size, so that the maintenance is done 100 times before CommitCache array wraps around rather than once per eviction.

### Lock-free Snapshot List

In the above, we mentioned that a few read-only transactions doing backups is expected at each point of time. Each evicted entry, which is expected upon each insert, is therefore needs to be checked against the list of live snapshots (which was taken when max_evicted_seq was last advanced). Since this is done frequently it needs to be done efficiently, without holding any mutex. We therefore design a data structure that lets us perform this check in a lock-free manner. To do so, we store the first S snapshots in an array of std::atomic<uint64_t>, which we know is efficient for reads and write on x86_64 architecture. The single writer updates the array with a list of snapshots sorted in ascending order by starting from index 0, and updates the size in an atomic variable.

We need to guarantee that the concurrent reader will be able to read all the snapshots that are still valid after the update. Both new and old lists are sorted and the new list is a subset of the previous list plus some new items. Thus if a snapshot repeats in both new and old lists, it will appear with a lower index in the new list. So if we simply insert the new snapshots in order, if an overwritten item is still valid in the new list, it is either written to the same place in the array or it is written in a place with a lower index before it gets overwritten by another item. This guarantees a reader that reads the array from the other side will eventually see a snapshot that repeats in the update, either before it gets overwritten by the writer or afterwards.

If the number of snapshots exceed the array size, the remaining updates will be stored in a vector protected by a mutex. This is just to ensure correctness in the corner cases and is not expected to happen in a normal run.

# WriteUnprepared
## 待续...

# 参考
[WritePrepared Transactions](https://github.com/facebook/rocksdb/wiki/WritePrepared-Transactions)
