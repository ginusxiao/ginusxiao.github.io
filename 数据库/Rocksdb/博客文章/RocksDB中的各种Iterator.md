# 提纲
[toc]

# RocksDB Iterator架构
Rocksdb是一种LSM-tree的工业实现，其提供了一种可以遍历整个数据库的接口Iterator。利用如下的调用就可以很方便的遍历整个数据库。
```
rocksdb::Iterator* it = db->NewIterator(readOptions);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // Do something with it->key() and it->value().
}
```

利用Rocksdb 提供的Iterator，我们可以把Rocksdb当作是全局有序数据来进行操作。然而真正的数据是不可能按照全局排序来组织的，Rocksdb利用了类似std::Iterator的概念，封装了底层的数据存储方式，使用户在遍历时可以屏蔽真正底层的数据存储方式。

![rocksdb iterator](http://note.youdao.com/yws/public/resource/6ab15a691bf5fc740adfeeef70ff76dc/xmlnote/4ACBA1ABAB0641A98E774149D9391E7F/126728)

从上图中可以看出，RocksDB中包括InternalIterator，DBIter，MergingIterator，MemtableIterator，BlockIter，TwoLevelIterator和LevelFileNumIterator等Iterator。下面对这些Iterator逐一讲解。

# Basic Rocksdb Iterator
Rocksdb Iterator 的概念继承自Leveldb，对于每一种数据结构提供统一的查找和遍历接口。其基类函数如下。
```
class InternalIterator {
  bool Valid();
  void SeekToFirst();
  void SeekToLast();
  void Seek(const Slice& target);
  void Next();
  void Prev();
  // current key() and value()
  Slice key();
  Slice value();
}
```

# DBIter
MemTable和SSTable中记录的形式都是(userkey, seq, type) => uservalue。DBIter是对InternalIterator(通常是MergingIterator)的封装，它的作用是在考虑sequence number，deletion marker和merge等的情况下，将相同userkey的多条记录合并成一条记录。

假如InternalIterator呈现的记录如下：
```
InternalKey(user_key="Key1", seqno=10, Type=Put)    | Value = "KEY1_VAL2"
InternalKey(user_key="Key1", seqno=9,  Type=Put)    | Value = "KEY1_VAL1"
InternalKey(user_key="Key2", seqno=16, Type=Put)    | Value = "KEY2_VAL2"
InternalKey(user_key="Key2", seqno=15, Type=Delete) | Value = "KEY2_VAL1"
InternalKey(user_key="Key3", seqno=7,  Type=Delete) | Value = "KEY3_VAL1"
InternalKey(user_key="Key4", seqno=5,  Type=Put)    | Value = "KEY4_VAL1"
```

那么DBIter将呈现给用户的记录如下：
```
Key="Key1"  | Value = "KEY1_VAL2"
Key="Key2"  | Value = "KEY2_VAL2"
Key="Key4"  | Value = "KEY4_VAL1"
```

# MergingIterator
MergingIterator是关于很多child iterators的堆，这些child iterators包括Mutable MemTable，Immutable MemTable，每一个Level的SSTable等对应的Iterator。

利用MergingIterator可以提供全局有序的遍历。

假设它底层的child iterators呈现的记录如下：
```
= Child Iterator 1 =
InternalKey(user_key="Key1", seqno=10, Type=Put)    | Value = "KEY1_VAL2"

= Child Iterator 2 =
InternalKey(user_key="Key1", seqno=9,  Type=Put)    | Value = "KEY1_VAL1"
InternalKey(user_key="Key2", seqno=15, Type=Delete) | Value = "KEY2_VAL1"
InternalKey(user_key="Key4", seqno=5,  Type=Put)    | Value = "KEY4_VAL1"

= Child Iterator 3 =
InternalKey(user_key="Key2", seqno=16, Type=Put)    | Value = "KEY2_VAL2"
InternalKey(user_key="Key3", seqno=7,  Type=Delete) | Value = "KEY3_VAL1"
```

经过MergingIterator的处理之后，所有这些记录以有序流的形式呈现：
```
InternalKey(user_key="Key1", seqno=10, Type=Put)    | Value = "KEY1_VAL2"
InternalKey(user_key="Key1", seqno=9,  Type=Put)    | Value = "KEY1_VAL1"
InternalKey(user_key="Key2", seqno=16, Type=Put)    | Value = "KEY2_VAL2"
InternalKey(user_key="Key2", seqno=15, Type=Delete) | Value = "KEY2_VAL1"
InternalKey(user_key="Key3", seqno=7,  Type=Delete) | Value = "KEY3_VAL1"
InternalKey(user_key="Key4", seqno=5,  Type=Put)    | Value = "KEY4_VAL1"
```

## MergingIterator的定义
```
class MergingIterator : public InternalIterator {
 public:
  // InternalIterator virtual func implementation
  # 实现InternalIterator相关的接口，比如Seek(), Next(), Prev()等
  ...
  
 private:
  ...
  
  # 当前管理的所有Child Iterator的集合
  autovector<IteratorWrapper, kNumIterReserve> children_;
  
  # 缓存当前MergingIterator遍历结果
  IteratorWrapper* current_;
  
  enum Direction {
    kForward,
    kReverse
  };
  
  # 遍历方向，是正向遍历还是反向遍历
  Direction direction_;
  
  # 用于正向遍历的小顶堆
  MergerMinIterHeap minHeap_;
  
  # 用于后向遍历的大顶堆，它只在使用的时候才会初始化之
  std::unique_ptr<MergerMaxIterHeap> maxHeap_;  
}
```

## MergingIterator创建
RocksDB中通过DBImpl::NewIterator()来获取rocksdb::Iterator，然后进一步通过Iterator来查询指定ColumnFamily的数据。在DBImpl::NewIterator()中会创建MergingIterator。
```
Iterator* DBImpl::NewIterator(const ReadOptions& read_options,
                              ColumnFamilyHandle* column_family) {
    ...
  
    // Try to generate a DB iterator tree in continuous memory area to be
    // cache friendly. Here is an example of result:
    // +-------------------------------+
    // |                               |
    // | ArenaWrappedDBIter            |
    // |  +                            |
    // |  +---> Inner Iterator   ------------+
    // |  |                            |     |
    // |  |    +-- -- -- -- -- -- -- --+     |
    // |  +--- | Arena                 |     |
    // |       |                       |     |
    // |          Allocated Memory:    |     |
    // |       |   +-------------------+     |
    // |       |   | DBIter            | <---+
    // |           |  +                |
    // |       |   |  +-> iter_  ------------+
    // |       |   |                   |     |
    // |       |   +-------------------+     |
    // |       |   | MergingIterator   | <---+
    // |           |  +                |
    // |       |   |  +->child iter1  ------------+
    // |       |   |  |                |          |
    // |           |  +->child iter2  ----------+ |
    // |       |   |  |                |        | |
    // |       |   |  +->child iter3  --------+ | |
    // |           |                   |      | | |
    // |       |   +-------------------+      | | |
    // |       |   | Iterator1         | <--------+
    // |       |   +-------------------+      | |
    // |       |   | Iterator2         | <------+
    // |       |   +-------------------+      |
    // |       |   | Iterator3         | <----+
    // |       |   +-------------------+
    // |       |                       |
    // +-------+-----------------------+
    //
    // ArenaWrappedDBIter inlines an arena area where all the iterators in
    // the iterator tree are allocated in the order of being accessed when
    // querying.
    // Laying out the iterators in the order of being accessed makes it more
    // likely that any iterator pointer is close to the iterator it points to so
    // that they are likely to be in the same cache line and/or page.
    
    # 创建一个ArenaWrappedDBIter，其中包含2个成员：DBIter和Arena
    ArenaWrappedDBIter* db_iter = NewArenaWrappedDbIterator(
        env_, *cfd->ioptions(), cfd->user_comparator(), snapshot,
        sv->mutable_cf_options.max_sequential_skip_in_iterations,
        sv->version_number, read_options.iterate_upper_bound,
        read_options.prefix_same_as_start, read_options.pin_data);

    # 创建一个MergingIterator
    InternalIterator* internal_iter =
        NewInternalIterator(read_options, cfd, sv, db_iter->GetArena());
    
    # 设置ArenaWrappedDBIter::DBIter::InternalIterator
    db_iter->SetIterUnderDBIter(internal_iter);

    return db_iter;
}

InternalIterator* DBImpl::NewInternalIterator(const ReadOptions& read_options,
                                              ColumnFamilyData* cfd,
                                              SuperVersion* super_version,
                                              Arena* arena) {
  InternalIterator* internal_iter;
  assert(arena != nullptr);
  
  # MergeIteratorBuilder用于创建MergingIterator，MergeIteratorBuilder::AddIterator
  # 用于向MergingIterator中逐一添加Iterator，MergeIteratorBuilder::Finish用于返回
  # 最终的MergingIterator
  MergeIteratorBuilder merge_iter_builder(cfd->internal_comparator().get(), arena);
  
  # 向MergingIterator中添加关于Mutable MemTable的Iterator，添加的是MemTableIterator
  merge_iter_builder.AddIterator(
      super_version->mem->NewIterator(read_options, arena));
      
  # 向MergingIterator中添加关于Immutable MemTable的Iterator，实际上针对Immutable
  # MemTable中的每个MemTable调用MergeIteratorBuilder::AddIterator，最终添加的是
  # MemTableIterator
  super_version->imm->AddIterators(read_options, &merge_iter_builder);
  
  # 向MergingIterator中添加当前Version中关于L0-Ln层SST文件的Iterator，
  # super_version->current对应的是最新的Version，所以调用的是Version::AddIterators
  # 最终添加的是关于L0层的TwoLevelIterator和L1-Ln层的TwoLevelIterator
  super_version->current->AddIterators(read_options, env_options_,
                                       &merge_iter_builder);
     
  # 从MergeIteratorBuilder中获取最终的MergingIterator
  internal_iter = merge_iter_builder.Finish();
  
  # 注册关于MergingIterator的Cleanup function
  IterState* cleanup = new IterState(this, &mutex_, super_version);
  internal_iter->RegisterCleanup(CleanupIteratorState, cleanup, nullptr);

  return internal_iter;
}

void Version::AddIterators(const ReadOptions& read_options,
                           const EnvOptions& soptions,
                           MergeIteratorBuilder* merge_iter_builder) {
  # 断言当前的Version已经构建完成
  assert(storage_info_.finalized_);

  if (storage_info_.num_non_empty_levels() == 0) {
    // No file in the Version.
    return;
  }

  auto* arena = merge_iter_builder->GetArena();

  # L0层的SST文件单独处理，因为L0层的文件之间可能存在交叠，所以需要对这些文件进行合并，
  # 最终创建的是一个TwoLevelIterator
  for (size_t i = 0; i < storage_info_.LevelFilesBrief(0).num_files; i++) {
    const auto& file = storage_info_.LevelFilesBrief(0).files[i];
    if (!read_options.file_filter || read_options.file_filter->Filter(file)) {
      InternalIterator *file_iter;
      TableCache::TableReaderWithHandle trwh;
      Status s = cfd_->table_cache()->GetTableReaderForIterator(read_options, soptions,
          cfd_->internal_comparator(), file.fd, &trwh, cfd_->internal_stats()->GetFileReadHist(0),
          false);
      if (s.ok()) {
        if (!read_options.table_aware_file_filter ||
            read_options.table_aware_file_filter->Filter(trwh.table_reader)) {
          file_iter = cfd_->table_cache()->NewIterator(
              read_options, &trwh, storage_info_.LevelFiles(0)[i]->UserFilter(), false, arena);
        } else {
          file_iter = nullptr;
        }
      } else {
        file_iter = NewErrorInternalIterator(s, arena);
      }
      if (file_iter) {
        merge_iter_builder->AddIterator(file_iter);
      }
    }
  }

  # 对于L1-Ln层的SST文件，每一层内部的SST文件之间不存在交叠，对于每一层来说，可以通过
  # 将该层内部的所有文件的Iterator连接起来，以实现顺序遍历
  # 
  # 对于每一层来说，最终创建的都是一个TwoLevelIterator
  for (int level = 1; level < storage_info_.num_non_empty_levels(); level++) {
    if (storage_info_.LevelFilesBrief(level).num_files != 0) {
      auto* mem = arena->AllocateAligned(sizeof(LevelFileIteratorState));
      auto* state = new (mem)
          LevelFileIteratorState(cfd_->table_cache(), read_options, soptions,
                                 cfd_->internal_comparator(),
                                 cfd_->internal_stats()->GetFileReadHist(level),
                                 false /* for_compaction */,
                                 cfd_->ioptions()->prefix_extractor != nullptr,
                                 IsFilterSkipped(level));
      mem = arena->AllocateAligned(sizeof(LevelFileNumIterator));
      auto* first_level_iter = new (mem) LevelFileNumIterator(
          *cfd_->internal_comparator(), &storage_info_.LevelFilesBrief(level));
      merge_iter_builder->AddIterator(NewTwoLevelIterator(state, first_level_iter, arena, false));
    }
  }
}
```

在Version::AddIterators中，为L0层和L1-Ln层创建的TwoLevelIterator分别是什么样的呢？将会在后面关于TwoLevelIterator的分析中讲解。

## MergingIterator遍历
MergingIterator的遍历的主要流程是将各个数据结构的iterator指向下一个查找的位置，然后通过比较每一种数据结构当前指向key的大小，最终反馈给上层一个全局的搜索或者遍历结果。

例如：

MergingIterator目前管理四个child iterator分别是memtable iterator，level0的第一个Iterator(level0-1 Iterator)和level0的第2个Iterator(level0-2 Iterator)。其中，单个memtable iterator只能操作一个memtable，level0-1和level0-2也类似只能操作一个sst文件，level1的iterator可以操作整个level1所有的数据。

**SeekToFirst()**

Memtable **(1,1)**, (2,2), (3,10), (10,1), (100,1)

Level0-1 **(3,9)**, (3,8), (3,7), (11,1)

Level0-2 **(2,1)**, (3,6), (3,5), (123,1)

Level1 **(3,4)**, (3,3), (3,2), (3,1)

current_ = (1,1) from Memtable

**粗体**部分表示当前child iterator所在的位置，也是下一次需要搜索的位置。

(3,1)表示key3的第一个版本，(3,10)表示key3的第10个版本。(3,10)数据比(3,1)更新，所在位置比(3,1)更上层，正向查找时也应该优先搜索到。

current表示当前搜索的结果，它指向包含当前搜索结果的那个Child Iterator。调用SeekToFirst后，所有child iteraor被指到了起始位置，通过比较每一个child iterator指向的key值，最终对上层的返回值应该是(1,1)，并存储在current内部。

**Next()**

Memtable (1,1), **(2,2)**, (3,10), (10,1), (100,1)

Level0-1 **(3,9)**, (3,8), (3,7), (11,1)

Level0-2 **(2,1)**, (3,6), (3,5), (123,1)

Level1 **(3,4)**, (3,3), (3,2), (3,1)

current_ = (2,2) from Memtable


**Next()**

Memtable (1,1), (2,2), **(3,10)**, (10,1), (100,1)

Level0-1 **(3,9)**, (3,8), (3,7), (11,1)

Level0-2 **(2,1)**, (3,6), (3,5), (123,1)

Level1 **(3,4)**, (3,3), (3,2), (3,1)

current_ = (2,1) from Level0-2


**Next()**

Memtable	(1,1), (2,2), **(3,10)**, (10,1), (100,1)

Level0-1 **(3,9)**, (3,8), (3,7), (11,1)

Level0-2 (2,1), **(3,6)**, (3,5), (123,1)

Level1 **(3,4)**, (3,3), (3,2), (3,1)

current_ = (3,10) from Memtable


**Next()**

Memtable (1,1), (2,2), (3,10), **(10,1)**, (100,1)

Level0-1 **(3,9)**, (3,8), (3,7), (11,1)

Level0-2 (2,1), **(3,6)**, (3,5), (123,1)

Level1 **(3,4)**, (3,3), (3,2), (3,1)

current_ = (3,9) from Level0-1


**Next()**

Memtable (1,1), (2,2), (3,10), **(10,1)**, (100,1)

Level0-1 (3,9), **(3,8)**, (3,7), (11,1)

Level0-2 (2,1), **(3,6)**, (3,5), (123,1)

Level1 **(3,4)**, (3,3), (3,2), (3,1)

current_ = (3,8) from Level0-1

目前遍历的结果如下:
(1,1), (2,2), (2,1), (3,10), (3,9), (3,8)


下面看看反向遍历。

**Prev()**

direction_ = kReverse

Memtable (1,1), (2,2), **(3,10)**, (10,1), (100,1)

Level0-1 **(3,9)**, (3,8), (3,7), (11,1)

Level0-2 **(2,1)**, (3,6), (3,5), (123,1)

Level1 **""**（3,4), (3,3), (3,2), (3,1)

current_ = (3,9) from Level0-1

反向遍历中每个Child Iterator应该指向它内部current的prev位置，原因如下：所有Child Iterator内部的current位置中只有一个是对应于Table的current，在调用prev()方法的时候，这个Child Iterator内部的current肯定是要指向它内部的current的prev位置，而其它Child Iterator内部的current在调用next()的过程中一定还没有被遍历到，所以它们内部的current也一定要指向它内部的current的prev位置。

Memtable内部的current位置应该是(10,1)，再进行prev 位置为(3,10)，Level0-1，Level0-2，Level1做同样的seek & prev操作之后，新的current位置分别为(3,9), (2,1), ""，对各个Child Iterator内部的current进行比较，得到具有最大的key且相同key中具有最小的版本号的current (3,9) from level0-1。


**Prev()**

Memtable (1,1), (2,2), **(3,10)**, (10,1), (100,1)

Level0-1 **""** (3,9), (3,8), (3,7), (11,1)

Level0-2 **(2,1)**, (3,6), (3,5), (123,1)

Level1 **""**（3,4）, (3,3), (3,2), (3,1)

current_ = (3,10) from Memtable


**Next()**

Memtable (1,1), (2,2), (3,10), **(10,1)**, (100,1)

Level0-1 **(3,9)**, (3,8), (3,7), (11,1)

Level0-2 (2,1), **(3,6)**, (3,5), (123,1)

Level1 **(3,4)**, (3,3), (3,2), (3,1)

current_ = (3,9) from level0-1

正向遍历的时候每个Child Iterator应该指向它内部的current的next位置，对MemTable，Level0-1，Level0-2和Level1来说分别对应的是(10,1), (3,9), (3,6), (3,4)，然后对各个Child Iterator的current进行比较，得到具有最小key且相同key中具有最大版本的current (3,9) from level0-1。


## MergingIterator优化
由于MergingIterator管理的child iterator集合随着rocksdb体积的膨胀会逐渐增加，对于上述所说的比较每一个child iterator指向的key值的这种比较方式显然存在性能问题。为解决这个问题，rocksdb在leveldb的基础上，引入了MergerMinIterHeap和MergerMaxIterHeap的结构。正向查找时，需要返回当前iterator集合的最小值，对应的current指向MergerMinIterHeap的top。反向查找时，需要返回当前iterator的最大值，对应的current指向MergerMaxIterHeap的top。由于用户正向查找的概率远大于反向查找，在初始化时只初始化MergerMinIterHeap，只有用户调用反向查找的接口的时候才创建MergerMaxIterHeap。

## MergingIterator的实现
**AddIterator**

向MergingIterator中添加新的Iterator，主要包括2个步骤：先将Iterator添加到MergingIterator::children_中，然后如果这个新添加的Iterator指向的是一个合法的位置的话，则将该Iterator添加到小顶堆MergingIterator::minHeap中。
```
virtual void AddIterator(InternalIterator* iter) {
    assert(direction_ == kForward);
    children_.emplace_back(iter);
    if (data_pinned_) {
      Status s = iter->PinData();
      assert(s.ok());
    }
    auto new_wrapper = children_.back();
    if (new_wrapper.Valid()) {
      minHeap_.push(&new_wrapper);
      current_ = CurrentForward();
    }
  }
```

**Valid**

检查MergingIterator当前是否指向一个合法的位置  
```
bool Valid() const override { return (current_ != nullptr); }
```


**SeekToFirst**

找到MergingIterator的起始位置，分为以下步骤：
- 清除当前的小顶堆和大顶堆(如果有的话)
- 遍历每一个Child Iterator，并对每个Child Iterator处理如下：
    - 找到该Child Iterator的起始位置
    - 如果该Child Iterator的起始位置指向一个合法的位置，则将它添加到小顶堆中
- 获取小顶堆的堆顶元素，作为MergingIterator的current_位置
```
void SeekToFirst() override {
    ClearHeaps();
    for (auto& child : children_) {
      child.SeekToFirst();
      if (child.Valid()) {
        minHeap_.push(&child);
      }
    }
    direction_ = kForward;
    current_ = CurrentForward();
  }
```


**SeekToLast**

找到MergingIterator的末尾位置，此时会用到大顶堆，步骤如下：
- 清除当前的小顶堆和大顶堆(如果有的话)
- 初始化大顶堆
- 遍历每一个Child Iterator，并对每个Child Iterator处理如下：
    - 找到该Child Iterator的末尾位置
    - 如果该Child Iterator的末尾位置指向一个合法的位置，则将它添加到大顶堆中
- 获取大顶堆的堆顶元素，作为MergingIterator的current_位置
```
void SeekToLast() override {
    ClearHeaps();
    InitMaxHeap();
    for (auto& child : children_) {
      child.SeekToLast();
      if (child.Valid()) {
        maxHeap_->push(&child);
      }
    }
    direction_ = kReverse;
    current_ = CurrentReverse();
  }
```


**Seek**

查找MergingIterator中给定key，并将current_指向第一个不小于给定key的记录所在的Child Iterator。
```
void Seek(const Slice& target) override {
    if (direction_ == kForward && current_ && current_->Valid()) {
      # 首先处理从当前的key前向查找的情况
      
      # 比较当前key和target之间的关系
      int key_vs_target = comparator_->Compare(current_->key(), target);
      if (key_vs_target == 0) {
        // We're already at the right key.
        return;
      }
      
      if (key_vs_target < 0) {
        // This is a "seek forward" operation, and the current key is less than the target. Keep
        // doing a seek on the top iterator and re-adding it to the min heap, until the top iterator
        // gives is a key >= target.
        # 当前的key是小于target的，可以执行前向查找
        while (key_vs_target < 0) {
          DCHECK_EQ(current_, CurrentForward());
          # current_指向包含当前key的Child Iterator，在current_所指向的Child Iterator
          # 中查找不小于target的第一个位置
          current_->Seek(target);
          # 将current_所指向的Child Iterator添加到小顶堆中，这可能导致小顶堆的调整，调整之后
          # 会更新current_为小顶堆的堆顶元素(是一个Child Iterator)
          #
          # 当然，有一种特殊情况，即current_不再指向一个合法的位置，也就是说在current_所对应
          # 的Child Iterator中没有找到不小于target的记录，而当前的current一定是在堆顶，所以
          # 要移除堆顶的current_，查找target的过程中不会再用到该Child Iterator
          UpdateHeapAfterCurrentAdvancement();
          if (current_ == nullptr || !current_->Valid())
            return;  // Reached the end.
          key_vs_target = comparator_->Compare(current_->key(), target);
        }

        // The current key is >= target, this is what we're looking for.
        return;
      }

      // The current key is already greater than the target, so this is not a forward seek.
      // Fall back to a full rebuild of the heap.
    }

    # 至此，无法通过前向查找的方式找到第一个不小于target的记录的位置
    
    # 首先清除当前的小顶堆和大顶堆(如果有的话)
    ClearHeaps();
    
    # 遍历每个Child Iterator，并对其执行Seek，查找第一个不小于target的记录的位置，
    # 并将该Child Iterator添加到小顶堆中
    for (auto& child : children_) {
      {
        PERF_TIMER_GUARD(seek_child_seek_time);
        child.Seek(target);
      }
      PERF_COUNTER_ADD(seek_child_seek_count, 1);

      if (child.Valid()) {
        PERF_TIMER_GUARD(seek_min_heap_time);
        minHeap_.push(&child);
      }
    }
    
    # 获取当前堆顶的元素，即为不小于target的第一个元素
    direction_ = kForward;
    {
      PERF_TIMER_GUARD(seek_min_heap_time);
      current_ = CurrentForward();
    }
  }
  
  void UpdateHeapAfterCurrentAdvancement() {
    if (current_->Valid()) {
      # 将current_放在堆顶，小顶堆会重新调整
      minHeap_.replace_top(current_);
    } else {
      # current_已经不再指向一个合法的位置，所以要将current_从堆顶移除(current_
      # 一定是当前的堆顶元素)
      minHeap_.pop();
    }
    
    # 获取新的堆顶元素作为current_
    current_ = CurrentForward();
  }
```

**Next**

获取当前记录的下一个记录。
```
void Next() override {
    assert(Valid());

    // Ensure that all children are positioned after key().
    // If we are moving in the forward direction, it is already
    // true for all of the non-current children since current_ is
    // the smallest child and key() == current_->key().
    # 首先确保所有的Child Iterator都指向它内部不小于当前的key()的第一个记录
    if (direction_ != kForward) {
      # 清除当前的小顶堆和大顶堆(如果有的话)
      ClearHeaps();
      
      # 将所有Child Iterator中除current_所对应的Child Iterator之外的Iterator都
      # seek到它内部不小于当前的key()的第一个记录，并将这些Iterator添加到小顶堆中
      for (auto& child : children_) {
        if (&child != current_) {
          child.Seek(key());
          if (child.Valid() && comparator_->Equal(key(), child.key())) {
            child.Next();
          }
        }
        
        if (child.Valid()) {
          minHeap_.push(&child);
        }
      }
      
      direction_ = kForward;
      # 因为current_所对应的Child Iterator指向的记录就是当前的key()，而其它的
      # Child Iterator都已经确保不小于key()了，所以current_一定是当前小顶堆中
      # 最小的那个
      assert(current_ == CurrentForward());
    }

    // For the heap modifications below to be correct, current_ must be the current top of the heap.
    assert(current_ == CurrentForward());

    # 然后将current_所对应的Child Iterator向后移动一个位置
    current_->Next();
    
    # 将新的current替换小顶堆的堆顶元素（堆顶元素应该为之前尚未向后移动的current），
    # 然后minHeap会找到当前的最小节点，交换到小顶堆的堆顶，如果current iterator
    # 已经遍历完，那么就从小顶堆中移除当前iterator，之后小顶堆同样会将堆中的最小
    # 节点交换到最顶端
    UpdateHeapAfterCurrentAdvancement();
  }
```


**Prev**

获取当前记录的前一条记录。
```
void Prev() override {
    assert(Valid());
    // Ensure that all children are positioned before key().
    // If we are moving in the reverse direction, it is already
    // true for all of the non-current children since current_ is
    // the largest child and key() == current_->key().
    if (direction_ != kReverse) {
      // Otherwise, retreat the non-current children.  We retreat current_
      // just after the if-block.
      ClearHeaps();
      InitMaxHeap();
      
      # 将所有Child Iterator中除current_所对应的Child Iterator之外的Iterator都
      # seek到它内部小于当前的key()的第一个记录，并将这些Iterator添加到大顶堆中
      for (auto& child : children_) {
        if (&child != current_) {
          child.Seek(key());
          if (child.Valid()) {
            // Child is at first entry >= key().  Step back one to be < key()
            child.Prev();
          } else {
            // Child has no entries >= key().  Position at last entry.
            TEST_SYNC_POINT("MergeIterator::Prev:BeforeSeekToLast");
            child.SeekToLast();
          }
        }
        
        if (child.Valid()) {
          maxHeap_->push(&child);
        }
      }
      
      direction_ = kReverse;
      // Note that we don't do assert(current_ == CurrentReverse()) here
      // because it is possible to have some keys larger than the seek-key
      // inserted between Seek() and SeekToLast(), which makes current_ not
      // equal to CurrentReverse().
      # current_当前一定是大顶堆中堆顶元素
      current_ = CurrentReverse();
    }

    // For the heap modifications below to be correct, current_ must be the
    // current top of the heap.
    assert(current_ == CurrentReverse());

    # 然后将current_所对应的Child Iterator向前移动一个位置
    current_->Prev();
    
    # 如果current_指向一个合法的位置，则将其添加到大顶堆中，否则表明current_所
    # 对应的Child Iterator中不存在小于key()的记录，从大顶堆中移除current_所对应
    # 的Iterator
    if (current_->Valid()) {
      // current is still valid after the Prev() call above.  Call
      // replace_top() to restore the heap property.  When the same child
      // iterator yields a sequence of keys, this is cheap.
      maxHeap_->replace_top(current_);
    } else {
      // current stopped being valid, remove it from the heap.
      maxHeap_->pop();
    }
    
    current_ = CurrentReverse();
  }
```


**key**

获取MergingIterator中当前的key。
```
Slice key() const override {
    assert(Valid());
    return current_->key();
  }
```


**value**

获取MergingIterator中当前的key所对应的value。
```
Slice value() const override {
    assert(Valid());
    return current_->value();
  }
```

# MemtableIterator
MemtableIterator继承自InternalIterator，是对MemTableRep::Iterator的封装。
```
class MemTableIterator : public InternalIterator {
 public:
  MemTableIterator(
      const MemTable& mem, const ReadOptions& read_options, Arena* arena)
      : bloom_(nullptr),
        prefix_extractor_(mem.prefix_extractor_),
        valid_(false),
        arena_mode_(arena != nullptr) {
    if (prefix_extractor_ != nullptr && !read_options.total_order_seek) {
      bloom_ = mem.prefix_bloom_.get();
      iter_ = mem.table_->GetDynamicPrefixIterator(arena);
    } else {
      # 调用具体的MemTable实现类来获取对应的MemTableRep::Iterator
      iter_ = mem.table_->GetIterator(arena);
    }
  }

 private:
  ...
  
  # 内部封装了MemTableRep::Iterator
  MemTableRep::Iterator* iter_;
};
```

每种MemTable实现(比如基于skiplist的MemTable实现)都需要实现MemTableRep::Iterator中的相关接口。比如在基于skiplist的MemTable实现SkipListRep中就实现了MemTableRep::Iterator接口：
```
template <class SkipListImpl>
class SkipListRep : public MemTableRep {
  # 封装了SkipList的具体实现
  SkipListImpl skip_list_;
  
  ...
  
  class Iterator : public MemTableRep::Iterator {
    typename SkipListImpl::Iterator iter_;

   public:
    explicit Iterator(const SkipListImpl* list)
        : iter_(list) {}
        
    # 实现MemTableRep::Iterator中定义的一系列接口，如next(), prev()等
    void Next() override {
      # 借助于SkipListImpl::Iterator实现的
      iter_.Next();
    }

    void Prev() override {
      # 借助于SkipListImpl::Iterator实现的
      iter_.Prev();
    }
    
    ...
    
  }
  
  # 获取MemTableRep::Iterator
  MemTableRep::Iterator* GetIterator(Arena* arena = nullptr) override {
    if (lookahead_ > 0) {
      # 暂不考虑lookahead的情况
      void *mem =
        arena ? arena->AllocateAligned(sizeof(SkipListRep::LookaheadIterator))
              : operator new(sizeof(SkipListRep::LookaheadIterator));
      return new (mem) SkipListRep::LookaheadIterator(*this);
    } else {
      void *mem =
        arena ? arena->AllocateAligned(sizeof(SkipListRep::Iterator))
              : operator new(sizeof(SkipListRep::Iterator));
      return new (mem) SkipListRep::Iterator(&skip_list_);
    }
  }
}
```

可见SkipListRep::Iterator的实现进一步依赖于SkipListImpl::Iterator的实现。从SkipListFactory::CreateMemTableRep实现可知，在RocksDB中，SkipListImpl有2种，分别是支持并发写的InlineSkipList和支持单写的SingleWriterInlineSkipList。

关于InlineSkipList，请参考[RocksDB InlineSkipList](http://note.youdao.com/noteshare?id=59132e3d76b91746a18740b555ba288a&sub=36F86FF840704B219FA884BB2B4E94C2)。下面我们以InlineSkipList::Iterator为例分析基于skiplist的MemTable对应的MemTableIterator在底层是如何实现的。

## InlineSkipList::Iterator
如果对InlineSkipList有所理解的话，下面的代码看起来就非常简单。关于InlineSkipList，请参考[RocksDB InlineSkipList](http://note.youdao.com/noteshare?id=59132e3d76b91746a18740b555ba288a&sub=36F86FF840704B219FA884BB2B4E94C2)。
```
template <class Comparator>
class InlineSkipList {
  class Iterator {
   public:
    # 构造方法
    template <class Comparator>
    inline InlineSkipList<Comparator>::Iterator::Iterator(
        const InlineSkipList* list) {
      # 初始化Iterator对应的InlineSkipList
      SetList(list);
    }
    
    template <class Comparator>
    inline void InlineSkipList<Comparator>::Iterator::SetList(
        const InlineSkipList* list) {
      # 设置Iterator对应的InlineSkipList
      list_ = list;
      # 迭代器当前所指向的节点是无效的
      node_ = nullptr;
    }
    
    # 迭代器当前是否有效
    template <class Comparator>
    inline bool InlineSkipList<Comparator>::Iterator::Valid() const {
      return node_ != nullptr;
    }
    
    # 获取当前迭代器对应的key
    template <class Comparator>
    inline const char* InlineSkipList<Comparator>::Iterator::key() const {
      assert(Valid());
      return node_->Key();
    }
    
    # 找到第一个不小于给定key的节点
    template <class Comparator>
    inline void InlineSkipList<Comparator>::Iterator::Seek(const char* target) {
      node_ = list_->FindGreaterOrEqual(target);
    }
    
    # 找到第一个节点
    template <class Comparator>
    inline void InlineSkipList<Comparator>::Iterator::SeekToFirst() {
      node_ = list_->head_->Next(0);
    }
    
    # 找到最后一个节点，借助于FindLast来实现，实际上跟普通的查找过程是一样的
    template <class Comparator>
    inline void InlineSkipList<Comparator>::Iterator::SeekToLast() {
      node_ = list_->FindLast();
      if (node_ == list_->head_) {
        node_ = nullptr;
      }
    }
    
    # 迭代器的下一个位置
    template <class Comparator>
    inline void InlineSkipList<Comparator>::Iterator::Next() {
      assert(Valid());
      node_ = node_->Next(0);
    }
    
    # 迭代器的前一个位置，借助于FindLessThan来找到最后一个小于当前迭代器对应的key的记录
    template <class Comparator>
    inline void InlineSkipList<Comparator>::Iterator::Prev() {
      // Instead of using explicit "prev" links, we just search for the
      // last node that falls before key.
      assert(Valid());
      node_ = list_->FindLessThan(node_->Key());
      if (node_ == list_->head_) {
        node_ = nullptr;
      }
    }
    
   private:
    # 迭代器对应的InlineSkipList
    const InlineSkipList* list_;
    # 迭代器当前所指向的记录
    Node* node_;
  }
}
```

# BlockIter



# TwoLevelIterator
## L0层的TwoLevelIterator
先看L0层的TwoLevelIterator，为L0层创建TwoLevelIterator的代码如下：
```
void Version::AddIterators(const ReadOptions& read_options,
                           const EnvOptions& soptions,
                           MergeIteratorBuilder* merge_iter_builder) {
  ...
  
  // Merge all level zero files together since they may overlap
  # 注意，这里的Merge并不是把L0中的所有的文件都合并，然后再生成一个Iterator，而是为
  # L0中的所有的文件都生成一个Iterator，然后添加到MergingIterator中，在MergingIterator
  # 遍历过程中会自然而然的对所有这些Iterator进行合并
  for (size_t i = 0; i < storage_info_.LevelFilesBrief(0).num_files; i++) {
    const auto& file = storage_info_.LevelFilesBrief(0).files[i];
    if (!read_options.file_filter || read_options.file_filter->Filter(file)) {
      InternalIterator *file_iter;
      TableCache::TableReaderWithHandle trwh;
      Status s = cfd_->table_cache()->GetTableReaderForIterator(read_options, soptions,
          cfd_->internal_comparator(), file.fd, &trwh, cfd_->internal_stats()->GetFileReadHist(0),
          false);
      if (s.ok()) {
        if (!read_options.table_aware_file_filter ||
            read_options.table_aware_file_filter->Filter(trwh.table_reader)) {
          file_iter = cfd_->table_cache()->NewIterator(
              read_options, &trwh, storage_info_.LevelFiles(0)[i]->UserFilter(), false, arena);
        } else {
          file_iter = nullptr;
        }
      } else {
        file_iter = NewErrorInternalIterator(s, arena);
      }
      if (file_iter) {
        merge_iter_builder->AddIterator(file_iter);
      }
    }
  }
  
  ...
}
```

在创建L0层文件对应的Iterator的过程中，有2个关键方法：TableCache::GetTableReaderForIterator和TableCache::NewIterator，它们的调用栈如下：
```
Version::AddIterators
    # 获取TableReader
    - TableCache::GetTableReaderForIterator
        - TableCache::DoGetTableReaderForIterator
            - TableCache::DoGetTableReader
                # ioptions_.table_factory类型为TableFactory，可以设置为BlockBasedTableFactory，
                # PlainTableFactory和AdaptiveTableFactory，其中BlockBasedTableFactory被运用于
                # Block-based Table format，PlainTableFactory被运用于Plain Table format，而
                # AdaptiveTableFactory会根据SST文件的footer信息自动选择使用BlockBasedTableFactory
                # 来创建TableReader还是使用PlainTableFactory来创建TableReader
                #
                # 对于BlockBasedTableFactory来说，创建的TableReader类型为BlockBasedTable，
                # 对于PlainTableFactory来说，创建的TableReader类型为PlainTableReader
                - ioptions_.table_factory->NewTableReader(..., table_reader)
    - TableCache::NewIterator
        - TableCache::DoNewIterator
            # 借助于在前面获取到的TableReader(可能是BlockBasedTable类型，也可能是PlainTableReader
            # 类型)创建相应的Iterator
            - TableReader::NewIterator
```

下面以BlockBasedTable类型的TableReader为例，分析它所创建的Iterator：
```
InternalIterator* BlockBasedTable::NewIterator(const ReadOptions& read_options,
                                               Arena* arena,
                                               bool skip_filters) {
  # BlockEntryIteratorState会调用BlockBasedTable相关的方法来检查前缀是否匹配(PrefixMayMatch)，
  # 以及创建second level iterator(NewSecondaryIterator)                                  
  auto state = std::make_unique<BlockEntryIteratorState>(
      this, read_options, skip_filters, BlockType::kData);
      
  # 第2个参数NewIndexIterator(read_options)的结果是关于index block的BlockIter，
  # 第1个参数state将被用于TwoLevelIterator进行seek的过程中，创建second level iterator
  #
  # NewTwoLevelIterator最终会调用TwoLevelIterator的构造方法来构造TwoLevelIterator
  return NewTwoLevelIterator(
      state.release(), NewIndexIterator(read_options), arena, true /* need_free_iter_and_state */
  );
}

InternalIterator* BlockBasedTable::NewIndexIterator(
    const ReadOptions& read_options, BlockIter* input_iter) {
  # BlockBasedTable::GetIndexReader实现如下：如果在Block Cache中找到了对应的IndexReader，则
  # 直接返回该IndexReader，否则，调用BlockBasedTable::CreateDataBlockIndexReader来创建一个
  # 新的IndexReader，在BlockBasedTable::CreateDataBlockIndexReader中会根据index的类型来创建
  # 相应的IndexReader，对于IndexType::kBinarySearch类型，创建的是BinarySearchIndexReader，
  # 对于IndexType::kHashSearch类型，创建的是HashIndexReader，对于
  # IndexType::kMultiLevelBinarySearch，则创建的是MultiLevelIndexReader
  const auto index_reader_result = GetIndexReader(read_options);

  # index_reader_result无论是上面3种IndexReader(BinarySearchIndexReader, HashIndexReader和
  # MultiLevelIndexReader)中的哪一种，最终都会调用Block::NewIterator来创建关于index block的
  # BlockIter
  auto* new_iter = index_reader_result->value->NewIterator(
      input_iter, rep_->data_index_iterator_state.get(), read_options.total_order_seek);

  return new_iter;
}
```

TwoLevelIterator会在Seek, SeekToFirst, SeekToLast等方法中调用TwoLevelIterator::InitDataBlock来创建second level iterator。下面以TwoLevelIterator::Seek为例分析创建的second level iterator是啥？
```
class TwoLevelIterator : public InternalIterator {
  TwoLevelIterator(TwoLevelIteratorState* state,
                                   InternalIterator* first_level_iter,
                                   bool need_free_iter_and_state)
    : state_(state),
      first_level_iter_(first_level_iter),
      need_free_iter_and_state_(need_free_iter_and_state) {}
 
 private:   
  # 在当前上下文中对应的是BlockEntryIteratorState
  TwoLevelIteratorState* state_;
  IteratorWrapper first_level_iter_;
  IteratorWrapper second_level_iter_;  // May be nullptr
  bool need_free_iter_and_state_;
  Status status_;
  // If second_level_iter is non-nullptr, then "data_block_handle_" holds the
  // "index_value" passed to block_function_ to create the second_level_iter.
  std::string data_block_handle_;
};

void TwoLevelIterator::Seek(const Slice& target) {
  if (state_->check_prefix_may_match &&
      !state_->PrefixMayMatch(target)) {
    # 如果前缀不匹配，则直接设置二级iterator为nullptr
    SetSecondLevelIterator(nullptr);
    return;
  }
 
  # 先将一级iterator定位到相应的位置
  first_level_iter_.Seek(target);

  # 初始化二级iterator
  InitDataBlock();
  
  # 在二级iterator上执行查找
  if (second_level_iter_.iter() != nullptr) {
    second_level_iter_.Seek(target);
  }
  
  SkipEmptyDataBlocksForward();
}

void TwoLevelIterator::InitDataBlock() {
  if (!first_level_iter_.Valid()) {
    # 如果一级iterator无效，则直接设置二级iterator为nullptr，在调用TwoLevelIterator::InitDataBlock
    # 之前，一定已经在一级iterator上执行了seek操作
    SetSecondLevelIterator(nullptr);
  } else {
    Slice handle = first_level_iter_.value();
    if (second_level_iter_.iter() != nullptr &&
        !second_level_iter_.status().IsIncomplete() &&
        handle.compare(data_block_handle_) == 0) {
      // second_level_iter is already constructed with this iterator, so
      // no need to change anything
    } else {
      # 二级iterator不存在，则创建一个新的，在当前上下文中state_类型为BlockEntryIteratorState，
      # 这里的handle是一级iterator对应的value，BlockEntryIteratorState::NewSecondaryIterator
      # 会基于一级iterator对应的value创建二级iterator
      InternalIterator* iter = state_->NewSecondaryIterator(handle);
      data_block_handle_.assign(handle.cdata(), handle.size());
      SetSecondLevelIterator(iter);
    }
  }
}

class BlockBasedTable::BlockEntryIteratorState : public TwoLevelIteratorState {
  ...
  
  InternalIterator* NewSecondaryIterator(const Slice& index_value) override {
    # table_类型为BlockBasedTable，它其实是一个TableReader，调用的是BlockBasedTable::NewDataBlockIterator
    return table_->NewDataBlockIterator(read_options_, index_value, block_type_);
  }
};

InternalIterator* BlockBasedTable::NewDataBlockIterator(const ReadOptions& ro,
    const Slice& index_value, BlockType block_type, BlockIter* input_iter) {
  # 获取block cache/compressed block cache
  Cache* block_cache = rep_->table_options.block_cache.get();
  Cache* block_cache_compressed =
      rep_->table_options.block_cache_compressed.get();
  CachableEntry<Block> block;

  BlockHandle handle;
  Slice input = index_value;
  Status s = handle.DecodeFrom(&input);

  FileReaderWithCachePrefix* reader = GetBlockReader(block_type);

  # 先从block cache/compressed block cache中查找
  if (block_cache != nullptr || block_cache_compressed != nullptr) {
    Statistics* statistics = rep_->ioptions.statistics;
    char cache_key[block_based_table::kCacheKeyBufferSize];
    char compressed_cache_key[block_based_table::kCacheKeyBufferSize];
    Slice key, /* key to the block cache */
        ckey /* key to the compressed block cache */;

    # 为block cache创建key
    if (block_cache != nullptr) {
      key = GetCacheKey(reader->cache_key_prefix, handle, cache_key);
    }

    # 为compressed block cache创建ckey
    if (block_cache_compressed != nullptr) {
      ckey = GetCacheKey(reader->compressed_cache_key_prefix, handle, compressed_cache_key);
    }

    # 从block cache/compressed block cache中查找对应的block
    s = GetDataBlockFromCache(
        key, ckey, block_cache, block_cache_compressed, statistics, ro, &block,
        rep_->table_options.format_version, block_type, rep_->mem_tracker);

    # 如果在block cache/compressed block cache中没有查找到，且没有设置no_io(表示可以执行IO)，
    # 并且设置了fill_cache(表示要将读取到的结果缓存在block cache)，则从文件中读取，
    # 读取到的数据保存在raw_block中，然后保存到block cache/compressed block cache中
    if (block.value == nullptr && !no_io && ro.fill_cache) {
      std::unique_ptr<Block> raw_block;
      {
        StopWatch sw(rep_->ioptions.env, statistics, READ_BLOCK_GET_MICROS);
        s = block_based_table::ReadBlockFromFile(
            reader->reader.get(), rep_->footer, ro, handle, &raw_block, rep_->ioptions.env,
            rep_->mem_tracker, block_cache_compressed == nullptr);
      }

      if (s.ok()) {
        # 保存到block cache/compressed block cache
        s = PutDataBlockToCache(key, ckey, block_cache, block_cache_compressed,
                                ro, statistics, &block, raw_block.release(),
                                rep_->table_options.format_version, rep_->mem_tracker);
      }
    }
  }
  
  # 如果没有得到block cache
  if (s.ok() && block.value == nullptr) {
    if (no_io) {
      # 也不能执行IO，则直接返回NoIOError
      if (input_iter != nullptr) {
        input_iter->SetStatus(ReturnNoIOError());
        return input_iter;
      } else {
        return NewErrorInternalIterator(ReturnNoIOError());
      }
    }
    
    # 否则，从SSTable文件中读取
    std::unique_ptr<Block> block_value;
    s = block_based_table::ReadBlockFromFile(
        reader->reader.get(), rep_->footer, ro, handle, &block_value, rep_->ioptions.env,
        rep_->mem_tracker);
    if (s.ok()) {
      block.value = block_value.release();
    }
  }
  
  InternalIterator* iter;
  if (s.ok() && block.value != nullptr) {
    # 创建Data block上的迭代器
    iter = block.value->NewIterator(rep_->comparator.get(), input_iter);
    
    ...
  } else {
    ...
  }
}
```

可见，在TwoLevelIterator::InitDataBlock -> BlockBasedTable::BlockEntryIteratorState::NewSecondaryIterator -> BlockBasedTable::NewDataBlockIterator中会基于一级Iterator(index block iterator)对应的value去读取相应的data block(可能在block cache/compressed block cache中查找到，也可能从相应的SSTable文件中读取)，并在读取到的data block上建立二级Iterator(data block iterator)。

## L1-Ln层的TwoLevelIterator
再看L1-Ln层的TwoLevelIterator，为L1-Ln层创建TwoLevelIterator的代码如下：
```
void Version::AddIterators(const ReadOptions& read_options,
                           const EnvOptions& soptions,
                           MergeIteratorBuilder* merge_iter_builder) {
  ...
  
  for (int level = 1; level < storage_info_.num_non_empty_levels(); level++) {
    if (storage_info_.LevelFilesBrief(level).num_files != 0) {
      auto* mem = arena->AllocateAligned(sizeof(LevelFileIteratorState));
      auto* state = new (mem)
          LevelFileIteratorState(cfd_->table_cache(), read_options, soptions,
                                 cfd_->internal_comparator(),
                                 cfd_->internal_stats()->GetFileReadHist(level),
                                 false /* for_compaction */,
                                 cfd_->ioptions()->prefix_extractor != nullptr,
                                 IsFilterSkipped(level));
      mem = arena->AllocateAligned(sizeof(LevelFileNumIterator));
      # 一级Iterator是LevelFileNumIterator
      auto* first_level_iter = new (mem) LevelFileNumIterator(
          *cfd_->internal_comparator(), &storage_info_.LevelFilesBrief(level));
          
      # 基于一级Iterator创建TwoLevelIterator
      merge_iter_builder->AddIterator(NewTwoLevelIterator(state, first_level_iter, arena, false));
    }
  }  
  
  ...
}
```

LevelFileNumIterator用于访问Lm层(1 <= m <= n)中的文件。
```
class LevelFileNumIterator : public InternalIterator {
 public:
  LevelFileNumIterator(const InternalKeyComparator& icmp,
                       const LevelFilesBrief* flevel)
      : icmp_(icmp),
        flevel_(flevel),
        index_(static_cast<uint32_t>(flevel->num_files)),
        current_value_(0, 0, 0, 0) {  // Marks as invalid
  }

  bool Valid() const override { return index_ < flevel_->num_files; }

  # 寻找target所代表的key所在的文件
  void Seek(const Slice& target) override {
    index_ = FindFile(icmp_, *flevel_, target);
  }

  # 定位到第一个文件
  void SeekToFirst() override { index_ = 0; }

  # 定位到最后一个文件
  void SeekToLast() override {
    index_ = (flevel_->num_files == 0)
                 ? 0
                 : static_cast<uint32_t>(flevel_->num_files) - 1;
  }

  # 获取下一个文件
  void Next() override {
    assert(Valid());
    index_++;
  }

  # 获取上一个文件
  void Prev() override {
    assert(Valid());
    if (index_ == 0) {
      index_ = static_cast<uint32_t>(flevel_->num_files);  // Marks as invalid
    } else {
      index_--;
    }
  }

  # 获取当前迭代器所指向的文件中最大的key
  Slice key() const override {
    assert(Valid());
    return flevel_->files[index_].largest.key;
  }

  # 获取当前迭代器所指向的文件对应的FileDescriptor
  Slice value() const override {
    assert(Valid());

    auto file_meta = flevel_->files[index_];
    current_value_ = file_meta.fd;
    return Slice(reinterpret_cast<const char*>(&current_value_),
                 sizeof(FileDescriptor));
  }

  Status status() const override { return Status::OK(); }

 private:
  const InternalKeyComparator icmp_;
  # LevelFilesBrief保存这一层中文件数目及每个文件的range信息
  const LevelFilesBrief* flevel_;
  # 当前迭代器指向哪个文件
  uint32_t index_;
  # 当前迭代器所指向的文件对应的FileDescriptor
  mutable FileDescriptor current_value_;
};
```




# LevelFileNumIterator
见“L1-Ln层的TwoLevelIterator”中关于LevelFileNumIterator的讲解。
