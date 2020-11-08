# 提纲
[toc]

# Rocksdb LRUCache类关系图示

![image](http://note.youdao.com/noteshare?id=2f95ac1002638729a72455b7a7966aae&sub=630FCC1CAA4E4673BBD985D6623C5A0E)

## Cache
Cache是接口类，声明了以下接口：
- Cache entry操作相关接口

    向Cache中插入一个Cache entry；
    
    从Cache中查找某个Cache entry；
    
    从Cache中删除某个Cache entry；
    
    增加/减少Cache entry的引用计数；
    
    在所有Cache entries上执行某个回调；
    
    删除所有未被外部引用的Cache entries；

- Cache相关信息设置/获取

    设置Cache 容量；
    
    获取Cache 已使用容量；
    
    获取某个Cache entry的使用容量；
    
    获取被pin住的Cache entries的总的使用容量；

```
class Cache {
 public:
  /* Cache中的每一个entry都有其优先级：HIGH或者LOW，低优先级的entry优先于高优先级
   * 的entry被剔除Cache
   */
  enum class Priority { HIGH, LOW };

  Cache() {}
  virtual ~Cache() {}

  /* Handle用于唯一标识一个Cache entry（就跟File Handle中的Handle概念上是一样的），
   * 可以参考LRUHandle来对其有更好的理解
   */
  struct Handle {};

  /*Cache类型*/
  virtual const char* Name() const = 0;

  // Insert a mapping from key->value into the cache and assign it
  // the specified charge against the total cache capacity.
  // If strict_capacity_limit is true and cache reaches its full capacity,
  // return Status::Incomplete.
  //
  // If handle is not nullptr, returns a handle that corresponds to the
  // mapping. The caller must call this->Release(handle) when the returned
  // mapping is no longer needed. In case of error caller is responsible to
  // cleanup the value (i.e. calling "deleter").
  //
  // If handle is nullptr, it is as if Release is called immediately after
  // insert. In case of error value will be cleanup.
  //
  // When the inserted entry is no longer needed, the key and
  // value will be passed to "deleter".
  virtual Status Insert(const Slice& key, void* value, size_t charge,
                        void (*deleter)(const Slice& key, void* value),
                        Handle** handle = nullptr,
                        Priority priority = Priority::LOW) = 0;

  // If the cache has no mapping for "key", returns nullptr.
  //
  // Else return a handle that corresponds to the mapping.  The caller
  // must call this->Release(handle) when the returned mapping is no
  // longer needed.
  // If stats is not nullptr, relative tickers could be used inside the
  // function.
  virtual Handle* Lookup(const Slice& key, Statistics* stats = nullptr) = 0;

  // Increments the reference count for the handle if it refers to an entry in
  // the cache. Returns true if refcount was incremented; otherwise, returns
  // false.
  // REQUIRES: handle must have been returned by a method on *this.
  /*增加handle对应的Cache entry的引用计数*/
  virtual bool Ref(Handle* handle) = 0;

  /**
   * Release a mapping returned by a previous Lookup(). A released entry might
   * still  remain in cache in case it is later looked up by others. If
   * force_erase is set then it also erase it from the cache if there is no
   * other reference to  it. Erasing it should call the deleter function that
   * was provided when the
   * entry was inserted.
   *
   * Returns true if the entry was also erased.
   */
  // REQUIRES: handle must not have been released yet.
  // REQUIRES: handle must have been returned by a method on *this.
  /* 减少handle对应的Cache entry的引用计数，如果没有其它外部引用且设置了force_erase，
   * 则从Cache中删除该Cache entry
   */
  virtual bool Release(Handle* handle, bool force_erase = false) = 0;

  // Return the value encapsulated in a handle returned by a
  // successful Lookup().
  // REQUIRES: handle must not have been released yet.
  // REQUIRES: handle must have been returned by a method on *this.
  /*返回handle对应的Cache entry中的记录的内容*/
  virtual void* Value(Handle* handle) = 0;

  // If the cache contains entry for key, erase it.  Note that the
  // underlying entry will be kept around until all existing handles
  // to it have been released.
  /*删除某个Cache entry*/
  virtual void Erase(const Slice& key) = 0;
  
  // Return a new numeric id.  May be used by multiple clients who are
  // sharding the same cache to partition the key space.  Typically the
  // client will allocate a new id at startup and prepend the id to
  // its cache keys.
  /* 可能用于多client访问分区Cache的时候，每个client都在启动的时候分配一个
   * ID并将该ID添加到它的所有的keys的前面
   */
  virtual uint64_t NewId() = 0;

  // sets the maximum configured capacity of the cache. When the new
  // capacity is less than the old capacity and the existing usage is
  // greater than new capacity, the implementation will do its best job to
  // purge the released entries from the cache in order to lower the usage
  /* 设置Cache最大容量，如果当前使用容量大于设置后的容量，则会尽可能回收那些
   * 可以被释放的Cache entries
   */
  virtual void SetCapacity(size_t capacity) = 0;

  // Set whether to return error on insertion when cache reaches its full
  // capacity.
  /*如果设置了strict capacity limit，则在cache达到容量上限的情况下的插入会报错*/
  virtual void SetStrictCapacityLimit(bool strict_capacity_limit) = 0;

  // Get the flag whether to return error on insertion when cache reaches its
  // full capacity.
  /*是否设置了strict cache limit*/
  virtual bool HasStrictCapacityLimit() const = 0;

  // returns the maximum configured capacity of the cache
  /*返回设置的最大容量*/
  virtual size_t GetCapacity() const = 0;

  // returns the memory size for the entries residing in the cache.
  /*返回所有Cache entries占用的总容量*/
  virtual size_t GetUsage() const = 0;

  // returns the memory size for a specific entry in the cache.
  /*返回某个Cache entry占用的容量*/
  virtual size_t GetUsage(Handle* handle) const = 0;

  // returns the memory size for the entries in use by the system
  /*返回被pin住的Cache entries占用的总容量*/
  virtual size_t GetPinnedUsage() const = 0;

  // Call this on shutdown if you want to speed it up. Cache will disown
  // any underlying data and will not free it on delete. This call will leak
  // memory - call this only if you're shutting down the process.
  // Any attempts of using cache after this call will fail terribly.
  // Always delete the DB object before calling this method!
  virtual void DisownData(){
      // default implementation is noop
  };

  // Apply callback to all entries in the cache
  // If thread_safe is true, it will also lock the accesses. Otherwise, it will
  // access the cache without the lock held
  /*在所有Cache entries上运行某个callback*/
  virtual void ApplyToAllCacheEntries(void (*callback)(void*, size_t),
                                      bool thread_safe) = 0;

  // Remove all entries.
  // Prerequisite: no entry is referenced.
  /*删除那些未被外部引用的Cache entries*/
  virtual void EraseUnRefEntries() = 0;

  virtual std::string GetPrintableOptions() const { return ""; }

  // Mark the last inserted object as being a raw data block. This will be used
  // in tests. The default implementation does nothing.
  virtual void TEST_mark_as_data_block(const Slice& key, size_t charge) {}

 private:
  // No copying allowed
  Cache(const Cache&);
  Cache& operator=(const Cache&);
};
```

## CacheShard
CacheShard也是接口类，包含单个Cache分区相关的接口。
```
class CacheShard {
 public:
  CacheShard() = default;
  virtual ~CacheShard() = default;

  /*接口跟Cache中差不多*/
  virtual Status Insert(const Slice& key, uint32_t hash, void* value,
                        size_t charge,
                        void (*deleter)(const Slice& key, void* value),
                        Cache::Handle** handle, Cache::Priority priority) = 0;
  virtual Cache::Handle* Lookup(const Slice& key, uint32_t hash) = 0;
  virtual bool Ref(Cache::Handle* handle) = 0;
  virtual bool Release(Cache::Handle* handle, bool force_erase = false) = 0;
  virtual void Erase(const Slice& key, uint32_t hash) = 0;
  virtual void SetCapacity(size_t capacity) = 0;
  virtual void SetStrictCapacityLimit(bool strict_capacity_limit) = 0;
  virtual size_t GetUsage() const = 0;
  virtual size_t GetPinnedUsage() const = 0;
  virtual void ApplyToAllCacheEntries(void (*callback)(void*, size_t),
                                      bool thread_safe) = 0;
  virtual void EraseUnRefEntries() = 0;
  virtual std::string GetPrintableOptions() const { return ""; }
};
```

## ShardedCache
ShardedCache类也是一个接口类，它将Cache进行分区，分区数量为2^num_shard_bits，取key的hash值的高num_shard_bits位作为分区依据。
```
class ShardedCache : public Cache {
 public:
  ShardedCache(size_t capacity, int num_shard_bits, bool strict_capacity_limit);
  virtual ~ShardedCache() = default;
  virtual const char* Name() const override = 0;
  /*获取编号为shard的分区*/
  virtual CacheShard* GetShard(int shard) = 0;
  virtual const CacheShard* GetShard(int shard) const = 0;
  /*根据Cache entry的Handle获取Cache entry的value，charge和hash值*/  
  virtual void* Value(Handle* handle) override = 0;
  virtual size_t GetCharge(Handle* handle) const = 0;
  virtual uint32_t GetHash(Handle* handle) const = 0;
  virtual void DisownData() override = 0;

  /*下面这些方法都继承自Cache类*/
  virtual void SetCapacity(size_t capacity) override;
  virtual void SetStrictCapacityLimit(bool strict_capacity_limit) override;

  virtual Status Insert(const Slice& key, void* value, size_t charge,
                        void (*deleter)(const Slice& key, void* value),
                        Handle** handle, Priority priority) override;
  virtual Handle* Lookup(const Slice& key, Statistics* stats) override;
  virtual bool Ref(Handle* handle) override;
  virtual bool Release(Handle* handle, bool force_erase = false) override;
  virtual void Erase(const Slice& key) override;
  virtual uint64_t NewId() override;
  virtual size_t GetCapacity() const override;
  virtual bool HasStrictCapacityLimit() const override;
  virtual size_t GetUsage() const override;
  virtual size_t GetUsage(Handle* handle) const override;
  virtual size_t GetPinnedUsage() const override;
  virtual void ApplyToAllCacheEntries(void (*callback)(void*, size_t),
                                      bool thread_safe) override;
  virtual void EraseUnRefEntries() override;
  virtual std::string GetPrintableOptions() const override;

  /*获取@num_shard_bits_*/
  int GetNumShardBits() const { return num_shard_bits_; }

 private:
  /*获取Slice对应的hash值*/
  static inline uint32_t HashSlice(const Slice& s) {
    return Hash(s.data(), s.size(), 0);
  }

  /*根据hash值计算所在分区*/
  uint32_t Shard(uint32_t hash) {
    // Note, hash >> 32 yields hash in gcc, not the zero we expect!
    return (num_shard_bits_ > 0) ? (hash >> (32 - num_shard_bits_)) : 0;
  }

  /*2^num_shard_bits_表示分区个数*/
  int num_shard_bits_;
  mutable port::Mutex capacity_mutex_;
  size_t capacity_;
  bool strict_capacity_limit_;
  std::atomic<uint64_t> last_id_;
};
```

## LRUHandle
每一个Cache entry都会对应一个Handle，通过该Handle可以唯一引用一个Cache entry。对于LRUCache来说，其中的Cache entry对应的Handle被称为LRUHandle。在LRUCache中每一个Cache entry对应的Handle可能在HashTable中，所以每一个LRUHandle中都有next_hash这个成员，同时也可能会在LRU链表中，所以每一个LRUHandle中有next和prev这两个成员。

LRUHandle可能处于下列三种状态：

在HashTable中且被外部引用，这种情况下满足：(refs > 1 && in_cache == true)，Cache entry不在LRU中；

在HashTable中但不被外部引用，这种情况下满足：(refs == 1 && in_cache == true)，Cache entry在LRU中，可以被释放；

不在HashTable中但被外部引用，这种情况下满足：(refs >= 1 && in_cache == false)，Cache entry不在LRU中；

```
struct LRUHandle {
  /*Cache entry中的内容*/
  void* value;
  void (*deleter)(const Slice&, void* value);
  /*通过next_hash链接到Hash桶的链表中*/
  LRUHandle* next_hash;
  /*通过next和prev形成链表，主要用于LRU链表*/
  LRUHandle* next;
  LRUHandle* prev;
  size_t charge;  // TODO(opt): Only allow uint32_t?
  size_t key_length;
  /*引用计数*/
  uint32_t refs;     // a number of refs to this entry
                     // cache itself is counted as 1

  // Include the following flags:
  //   in_cache:    whether this entry is referenced by the hash table.
  //   is_high_pri: whether this entry is high priority entry.
  //   in_high_pro_pool: whether this entry is in high-pri pool.
  /* 包含如下标识：
   * in_cache： 是否在HashTable中
   * is_high_pri: 是否是高优先级的Cache entry
   * in_high_pro_pool： 是否在高优先级的pool中
   */
  char flags;

  /*hash值*/
  uint32_t hash;     // Hash of key(); used for fast sharding and comparisons
 
  /*Cache entry中的key*/
  char key_data[1];  // Beginning of key

  Slice key() const {
    // For cheaper lookups, we allow a temporary Handle object
    // to store a pointer to a key in "value".
    if (next == this) {
      return *(reinterpret_cast<Slice*>(value));
    } else {
      return Slice(key_data, key_length);
    }
  }

  /*Cache entry标识相关的检查*/
  bool InCache() { return flags & 1; }
  bool IsHighPri() { return flags & 2; }
  bool InHighPriPool() { return flags & 4; }

  /*Cache entry标识相关的设置*/
  void SetInCache(bool in_cache) {
    if (in_cache) {
      flags |= 1;
    } else {
      flags &= ~1;
    }
  }

  void SetPriority(Cache::Priority priority) {
    if (priority == Cache::Priority::HIGH) {
      flags |= 2;
    } else {
      flags &= ~2;
    }
  }

  void SetInHighPriPool(bool in_high_pri_pool) {
    if (in_high_pri_pool) {
      flags |= 4;
    } else {
      flags &= ~4;
    }
  }

  /*释放这个Cache entry*/
  void Free() {
    assert((refs == 1 && InCache()) || (refs == 0 && !InCache()));
    if (deleter) {
      (*deleter)(key(), value);
    }
    delete[] reinterpret_cast<char*>(this);
  }
};
```

## LRUHandleTable
LRUHandleTable实现了简单的关于LRUHandle的HashTable。
```
class LRUHandleTable {
 public:
  LRUHandleTable();
  ~LRUHandleTable();

  /*在HashTable中查找、插入或者删除*/
  LRUHandle* Lookup(const Slice& key, uint32_t hash);
  LRUHandle* Insert(LRUHandle* h);
  LRUHandle* Remove(const Slice& key, uint32_t hash);

  /*在所有Cache entry上执行特定函数*/
  template <typename T>
  void ApplyToAllCacheEntries(T func) {
    /*依次处理每一个hash桶*/
    for (uint32_t i = 0; i < length_; i++) {
      LRUHandle* h = list_[i];
      /*处理hash桶所在的链表中的每一个元素（hash桶中的元素通过LRUHandle::next_hash链接起来）*/
      while (h != nullptr) {
        auto n = h->next_hash;
        assert(h->InCache());
        func(h);
        h = n;
      }
    }
  }

 private:
  /* 查找指定key所在的Cache entry，如果查找到，则返回指向该Cache entry对应的
   * LRUHandle的指针，否则返回相应hash桶的链表的末尾
   */
  LRUHandle** FindPointer(const Slice& key, uint32_t hash);

  /*调整HashTable的大小*/
  void Resize();

  /*HashTable中桶的个数，即list_数组中元素个数*/
  uint32_t length_;
  /*HashTable中总的Cache entry的个数*/
  uint32_t elems_;
  /*HashTable，其中包含length_个桶，每个桶都是一个LRUHandle类型的链表*/
  LRUHandle** list_;
};
```

LRUHandleTable::Insert()向HashTable中插入一个Cache entry，会首先尝试查找，如果已经存在具有相同key和hash值的entry，则替换之。如果不存在
```
LRUHandle* LRUHandleTable::Insert(LRUHandle* h) {
  LRUHandle** ptr = FindPointer(h->key(), h->hash);
  LRUHandle* old = *ptr;
  h->next_hash = (old == nullptr ? nullptr : old->next_hash);
  *ptr = h;
  
  /* HashTable中不存在具有相同key和hash值的entry，则增加元素个数，如果元素个数
   * 超过了HashTable中桶的个数，则调整HashTable中桶的个数，并且重新分布HashTable
   * 中的元素（目标是每一个hash桶中链表长度不大于1）
   */
  if (old == nullptr) {
    ++elems_;
    if (elems_ > length_) {
      // Since each cache entry is fairly large, we aim for a small
      // average linked list length (<= 1).
      Resize();
    }
  }
  return old;
}
```

LRUHandleTable::Resize()根据当前HashTable中元素个数重新计算HashTable中Hash桶的数目，并且对当前HashTable中的元素进行重新分布。
```
void LRUHandleTable::Resize() {
  uint32_t new_length = 16;
  /*从16开始不断尝试增加HashTable中桶的数目*/
  while (new_length < elems_ * 1.5) {
    new_length *= 2;
  }
  
  /*创建新的HashTable*/
  LRUHandle** new_list = new LRUHandle*[new_length];
  memset(new_list, 0, sizeof(new_list[0]) * new_length);
  
  /*依次遍历旧的HashTable中的每一个桶，并将其中的元素在新的HashTable中重新分布*/
  uint32_t count = 0;
  for (uint32_t i = 0; i < length_; i++) {
    LRUHandle* h = list_[i];
    while (h != nullptr) {
      /*记住hash桶中下一个元素*/
      LRUHandle* next = h->next_hash;
      /*根据hash值计算当前元素在新的HashTable的哪个桶中*/
      uint32_t hash = h->hash;
      LRUHandle** ptr = &new_list[hash & (new_length - 1)];
      h->next_hash = *ptr;
      *ptr = h;
      h = next;
      count++;
    }
  }
  
  assert(elems_ == count);
  /*删除旧的HashTable，启用新的HashTable*/
  delete[] list_;
  list_ = new_list;
  length_ = new_length;
}
```

LRUHandleTable::FindPointer()在HashTable中查找指定key所在的Cache entry，如果查找到，则返回指向该Cache entry对应的LRUHandle的指针，否则返回相应hash桶的链表的末尾。
```
LRUHandle** LRUHandleTable::FindPointer(const Slice& key, uint32_t hash) {
  LRUHandle** ptr = &list_[hash & (length_ - 1)];
  while (*ptr != nullptr && ((*ptr)->hash != hash || key != (*ptr)->key())) {
    ptr = &(*ptr)->next_hash;
  }
  return ptr;
}
```

LRUHandleTable::Lookup()在HashTable中查找具有指定key和hash值的Cache entry。
```
LRUHandle* LRUHandleTable::Lookup(const Slice& key, uint32_t hash) {
  return *FindPointer(key, hash);
}
```

LRUHandleTable::Remove()从HashTable中删除具有指定key和hash值的Cache entry。
```
LRUHandle* LRUHandleTable::Remove(const Slice& key, uint32_t hash) {
  LRUHandle** ptr = FindPointer(key, hash);
  LRUHandle* result = *ptr;
  if (result != nullptr) {
    /*通过设置*ptr = result->next_hash来实现移除result所对应的元素*/
    *ptr = result->next_hash;
    --elems_;
  }
  
  return result;
}
```

## LRUCacheShard
LRUCacheShard继承自CacheShard，表示LRUCache的一个分区。LRUCacheShard中包含一个LRU链表：lru_和一个HashTable：table_。LRU链表通过LRUHandle类型的lru_来表示，因为LRUHandle中有prev和next两个指针，最老的元素在lru_.next中，而最新的元素在lru_.prev中，lru_本身则不是链表中的元素，lru_自身就是一个链表头而已。如果调用了LRUCacheShard::SetHighPriorityPoolRatio设置高优先级pool所占的比例，则会将LRU进一步划分为高优先级部分和低优先级部分（在代码中被称为high priority pool和low priority pool），其中高优先级部分的链表头就是lru_，低优先级部分的链表头通过另一个指针lru_low_pri_来指示。
```
// A single shard of sharded cache.
class LRUCacheShard : public CacheShard {
 public:
  LRUCacheShard();
  virtual ~LRUCacheShard();

  // Separate from constructor so caller can easily make an array of LRUCache
  // if current usage is more than new capacity, the function will attempt to
  // free the needed space
  virtual void SetCapacity(size_t capacity) override;

  // Set the flag to reject insertion if cache if full.
  virtual void SetStrictCapacityLimit(bool strict_capacity_limit) override;

  // Set percentage of capacity reserved for high-pri cache entries.
  void SetHighPriorityPoolRatio(double high_pri_pool_ratio);

  // Like Cache methods, but with an extra "hash" parameter.
  virtual Status Insert(const Slice& key, uint32_t hash, void* value,
                        size_t charge,
                        void (*deleter)(const Slice& key, void* value),
                        Cache::Handle** handle,
                        Cache::Priority priority) override;
  virtual Cache::Handle* Lookup(const Slice& key, uint32_t hash) override;
  virtual bool Ref(Cache::Handle* handle) override;
  virtual bool Release(Cache::Handle* handle,
                       bool force_erase = false) override;
  virtual void Erase(const Slice& key, uint32_t hash) override;

  // Although in some platforms the update of size_t is atomic, to make sure
  // GetUsage() and GetPinnedUsage() work correctly under any platform, we'll
  // protect them with mutex_.

  virtual size_t GetUsage() const override;
  virtual size_t GetPinnedUsage() const override;

  virtual void ApplyToAllCacheEntries(void (*callback)(void*, size_t),
                                      bool thread_safe) override;

  virtual void EraseUnRefEntries() override;

  virtual std::string GetPrintableOptions() const override;

  void TEST_GetLRUList(LRUHandle** lru, LRUHandle** lru_low_pri);

 private:
  void LRU_Remove(LRUHandle* e);
  void LRU_Insert(LRUHandle* e);

  // Overflow the last entry in high-pri pool to low-pri pool until size of
  // high-pri pool is no larger than the size specify by high_pri_pool_pct.
  void MaintainPoolSize();

  // Just reduce the reference count by 1.
  // Return true if last reference
  bool Unref(LRUHandle* e);

  // Free some space following strict LRU policy until enough space
  // to hold (usage_ + charge) is freed or the lru list is empty
  // This function is not thread safe - it needs to be executed while
  // holding the mutex_
  void EvictFromLRU(size_t charge, autovector<LRUHandle*>* deleted);

  /*该Cache分区的容量*/
  size_t capacity_;

  /*该Cache分区当前已经使用的容量*/
  size_t usage_;

  /*只在LRU链表中的Cache entries使用的容量*/
  size_t lru_usage_;

  /*高优先级的Cache entries使用的容量*/
  size_t high_pri_pool_usage_;

  /*如果Cache达到Cache容量上限，是否拒绝插入新的Cache entry*/
  bool strict_capacity_limit_;

  /*为高优先级的Cache entries预留的容量所占的比例*/
  double high_pri_pool_ratio_;

  /* 高优先级的Cache entries能够使用的容量，等价于capacity * high_pri_pool_ratio，
   * 这里是为了避免重复计算
   */
  double high_pri_pool_capacity_;

  // mutex_ protects the following state.
  // We don't count mutex_ as the cache's internal state so semantically we
  // don't mind mutex_ invoking the non-const actions.
  mutable port::Mutex mutex_;

  /* LRU链表（lru.prev是最新的entry, lru.next是最旧的entry）
   * LRU链表中保存的是那些可以被剔除的Cache entry
   */
  LRUHandle lru_;

  /*LRU链表中低优先级的Cache entries部分的头*/
  LRUHandle* lru_low_pri_;

  /*HashTable*/
  LRUHandleTable table_;
};
```

LRUCacheShard在构造函数中会初始化LRU链表头lru_和LRU链表的低优先级部分的头部lru_low_pri。
```
LRUCacheShard::LRUCacheShard()
    : usage_(0), lru_usage_(0), high_pri_pool_usage_(0) {
  // Make empty circular linked list
  /*空链表*/
  lru_.next = &lru_;
  lru_.prev = &lru_;
  lru_low_pri_ = &lru_;
}
```

设置单个分区的容量，如果已经使用的容量超过当前设置的容量，则尝试（从LRU中）回收那些未被使用的Cache entries：
```
void LRUCacheShard::SetCapacity(size_t capacity) {
  autovector<LRUHandle*> last_reference_list;
  {
    MutexLock l(&mutex_);
    capacity_ = capacity;
    high_pri_pool_capacity_ = capacity_ * high_pri_pool_ratio_;
    /*从LRU中找出那些不再被使用的Cache entries*/
    EvictFromLRU(0, &last_reference_list);
  }
  
  // we free the entries here outside of mutex for
  // performance reasons
  /*释放这些Cache entries*/
  for (auto entry : last_reference_list) {
    entry->Free();
  }
}

/* EvictFromLRU中有一个charge参数，是干嘛用的呢？因为在Insert的时候可能Cache剩余
 * 空间不足以容纳待插入的新的Cache entry需要的容量，所以需要去执行清理，直到可用
 * 容量足以满足带插入entry需要的容量，即charge
 */
void LRUCacheShard::EvictFromLRU(size_t charge,
                                 autovector<LRUHandle*>* deleted) {
  /* 删除LRU中那些Cache entries，直到使用容量usage_不超过设定容量capacity_
   * 或者LRU链表为空，被删除的元素同时从LRU链表和HashTable中删除，并且添加到
   * deleted集合中
   */
  while (usage_ + charge > capacity_ && lru_.next != &lru_) {
    LRUHandle* old = lru_.next;
    assert(old->InCache());
    assert(old->refs == 1);  // LRU list contains elements which may be evicted
    LRU_Remove(old);
    table_.Remove(old->key(), old->hash);
    old->SetInCache(false);
    Unref(old);
    usage_ -= old->charge;
    deleted->push_back(old);
  }
}
```

LRUCacheShard::SetHighPriorityPoolRatio()设置高优先级的pool的容量，
```
void LRUCacheShard::SetHighPriorityPoolRatio(double high_pri_pool_ratio) {
  MutexLock l(&mutex_);
  high_pri_pool_ratio_ = high_pri_pool_ratio;
  high_pri_pool_capacity_ = capacity_ * high_pri_pool_ratio_;
  MaintainPoolSize();
}

void LRUCacheShard::MaintainPoolSize() {
  /* 高优先级pool使用容量超过设定容量，则将那些处于高优先级pool中的entry标记
   * 为处于低优先级的pool中，同时减少高优先级pool使用的容量，直到高优先级pool
   * 使用容量低于设定容量
   */
  while (high_pri_pool_usage_ > high_pri_pool_capacity_) {
    // Overflow last entry in high-pri pool to low-pri pool.
    lru_low_pri_ = lru_low_pri_->next;
    assert(lru_low_pri_ != &lru_);
    lru_low_pri_->SetInHighPriPool(false);
    high_pri_pool_usage_ -= lru_low_pri_->charge;
  }
}
```

LRUCacheShard::Insert()插入一个Cache entry，charge表示该entry需要占用的容量，handle如果不为空则表示有外部引用希望引用这个新插入的Cache entry。插入过程分以下步骤：
- 首先分配一个新的LRUHandle，用于唯一标识该带插入的Cache entry，并采用传递的参数来初始化之；

- 检查当前Cache中可用容量是否足以容纳本次插入的entry所需要的容量charge，如果Cache当前可用容量（capacity_ - usage_）不足以满足当前插入所需的charge大小的容量，则尝试剔除一些Cache entries（LRU中的Cache entries就是那些可以被剔除的），直到Cache中有足够的容量容纳本次插入或者LRU链表为空（不能再剔除任何元素了）；

- 根据Cache容量使用情况来决定是否插入：
    
    如果不能被剔除的Cache entries占用的总容量（usage_ - lru_usage）加上本次插入需要的容量charge超过设定的容量，且{设置了严格限制Cache容量（strict capacity limit）或者本次插入没有外部引用}，则不执行插入；

    否则，插入到HashTable中，如果具有相同key和hash值的Cache entry已经存在，则替换这个已存在的entry，并减少该已存在的Cache entry的引用计数；如果已存在的Cache entry的引用计数降为0，则销毁之；如果插入成功，但是没有外部引用，则添加到LRU链表中；

```
Status LRUCacheShard::Insert(const Slice& key, uint32_t hash, void* value,
                             size_t charge,
                             void (*deleter)(const Slice& key, void* value),
                             Cache::Handle** handle, Cache::Priority priority) {
  // Allocate the memory here outside of the mutex
  // If the cache is full, we'll have to release it
  // It shouldn't happen very often though.
  LRUHandle* e = reinterpret_cast<LRUHandle*>(
      new char[sizeof(LRUHandle) - 1 + key.size()]);
  Status s;
  autovector<LRUHandle*> last_reference_list;

  e->value = value;
  e->deleter = deleter;
  e->charge = charge;
  e->key_length = key.size();
  e->hash = hash;
  /* 初始化引用计数，如果参数中指定了handle不为空，则表明有外部引用需要引用该Cache
   * entry，所以其引用计数为2，如果handle为nullptr，则只有HashTable引用该Cache
   * entry，所以其引用计数为1
   */
  e->refs = (handle == nullptr
                 ? 1
                 : 2);  // One from LRUCache, one for the returned handle
  e->next = e->prev = nullptr;
  /*设置其在HashTable中*/
  e->SetInCache(true);
  /*设置其优先级（要么为高优先级，要么为低优先级）*/
  e->SetPriority(priority);
  memcpy(e->key_data, key.data(), key.size());

  {
    MutexLock l(&mutex_);

    // Free the space following strict LRU policy until enough space
    // is freed or the lru list is empty
    /* 当前插入需要消耗charge大小的容量，如果Cache当前可用容量（capacity_ - usage_）
     * 不足以满足当前插入所需的charge大小的容量，则尝试剔除一些Cache entries（LRU
     * 中的Cache entries就是那些可以被剔除的），直到Cache中有足够的容量容纳本次插入
     * 或者LRU链表为空（不能再剔除任何元素了），关于EvictFromLRU见前面分析
     */
    EvictFromLRU(charge, &last_reference_list);

    /* 如果不能被剔除的Cache entries占用的总容量（usage_ - lru_usage）加上本次插入
     * 需要的容量charge超过设定的容量，且{设置了严格限制Cache容量（strict capacity
     * limit）或者本次插入没有外部引用}，则不执行插入
     */
    if (usage_ - lru_usage_ + charge > capacity_ &&
        (strict_capacity_limit_ || handle == nullptr)) {
      if (handle == nullptr) {
        // Don't insert the entry but still return ok, as if the entry inserted
        // into cache and get evicted immediately.
        /* 如果没有外部引用，则仍将当前元素添加到last_reference_list集合中
         *（就像是执行了插入，但是马上删除一样）
         */
        last_reference_list.push_back(e);
      } else {
        delete[] reinterpret_cast<char*>(e);
        *handle = nullptr;
        s = Status::Incomplete("Insert failed due to LRU cache being full.");
      }
    } else {
      // insert into the cache
      // note that the cache might get larger than its capacity if not enough
      // space was freed
      /* 插入到HashTable中，如果具有相同key和hash值的Cache entry已经存在，则替换
       * 这个已存在的entry，并减少该已存在的Cache entry的引用计数
       */
      LRUHandle* old = table_.Insert(e);
      /*更新使用容量*/
      usage_ += e->charge;
      /*发生了替换，则设置被替换的entry不在Cache中，且减少其引用计数*/
      if (old != nullptr) {
        old->SetInCache(false);
        if (Unref(old)) {
          usage_ -= old->charge;
          // old is on LRU because it's in cache and its reference count
          // was just 1 (Unref returned 0)
          /*减少引用计数后其引用计数降为0，则表明其之前在LRU链表中，从LRU删除之*/
          LRU_Remove(old);
          last_reference_list.push_back(old);
        }
      }
      
      if (handle == nullptr) {
        /*插入成功了，但是没有外部引用，所以添加到LRU链表中*/
        LRU_Insert(e);
      } else {
        /*返回handle给外部引用*/
        *handle = reinterpret_cast<Cache::Handle*>(e);
      }
      s = Status::OK();
    }
  }

  // we free the entries here outside of mutex for performance reasons
  /*删除所有处于last_reference_list中的entry*/
  for (auto entry : last_reference_list) {
    entry->Free();
  }

  return s;
}
```

LRUCacheShard::Lookup()在HashTable中查找指定key和hash值的Cache entry，如果找到了且引用计数为1，则该entry一定在LRU链表中，从LRU链表中删除之，并增加其引用计数，因为外面需要引用之。
```
Cache::Handle* LRUCacheShard::Lookup(const Slice& key, uint32_t hash) {
  MutexLock l(&mutex_);
  LRUHandle* e = table_.Lookup(key, hash);
  if (e != nullptr) {
    assert(e->InCache());
    if (e->refs == 1) {
      LRU_Remove(e);
    }
    e->refs++;
  }
  return reinterpret_cast<Cache::Handle*>(e);
}
```

LRUCacheShard::Ref()增加h所对应的Cache entry的引用计数，如果在LRU中（在HashTable中且引用计数为1），则从LRU中删除。
```
bool LRUCacheShard::Ref(Cache::Handle* h) {
  LRUHandle* handle = reinterpret_cast<LRUHandle*>(h);
  MutexLock l(&mutex_);
  if (handle->InCache() && handle->refs == 1) {
    LRU_Remove(handle);
  }
  handle->refs++;
  return true;
}
```

LRUCacheShard::Unref()减少e所对应的Cache entry的引用计数，如果引用计数降为0，则返回true。
```
bool LRUCacheShard::Unref(LRUHandle* e) {
  assert(e->refs > 0);
  e->refs--;
  return e->refs == 0;
}
```

LRUCacheShard::Release()减少handle对应的Cache entry的引用计数，分2种情况：
- 如果引用计数降为0，则从Cache中删除该entry；

- 如果引用计数降为1且在HashTable中，则根据Cache容量使用情况或者force_erase标识来处理：

    如果Cache使用容量超过了设定的容量，则直接从Cache中删除之；
    
    如果指定了force_erase，则也直接从Cache中删除之；
    
    否则将该元素添加到LRU中；
    
```
bool LRUCacheShard::Release(Cache::Handle* handle, bool force_erase) {
  if (handle == nullptr) {
    return false;
  }
  LRUHandle* e = reinterpret_cast<LRUHandle*>(handle);
  bool last_reference = false;
  {
    MutexLock l(&mutex_);
    /*减少引用计数*/
    last_reference = Unref(e);
    /*如果该元素引用计数降为0，则该元素即将被删除，减少Cache中使用的容量*/
    if (last_reference) {
      usage_ -= e->charge;
    }
    
    if (e->refs == 1 && e->InCache()) {
      // The item is still in cache, and nobody else holds a reference to it
      /*在HashTable中且元素引用计数为1（没有任何其它外部引用）*/
      if (usage_ > capacity_ || force_erase) {
        // the cache is full
        // The LRU list must be empty since the cache is full
        /* 如果Cache的使用容量超过设定容量，或者设置了force_erase，则删除该元素，
         * 因为该元素没有任何外部引用
         */
        assert(!(usage_ > capacity_) || lru_.next == &lru_);
        // take this opportunity and remove the item
        table_.Remove(e->key(), e->hash);
        e->SetInCache(false);
        Unref(e);
        usage_ -= e->charge;
        last_reference = true;
      } else {
        // put the item on the list to be potentially freed
        /*否则将该元素添加到LRU中*/
        LRU_Insert(e);
      }
    }
  }

  // free outside of mutex
  if (last_reference) {
    e->Free();
  }
  return last_reference;
}
```

LRUCacheShard::Erase()从Cache中删除指定key和hash值的entry
```
void LRUCacheShard::Erase(const Slice& key, uint32_t hash) {
  LRUHandle* e;
  bool last_reference = false;
  {
    MutexLock l(&mutex_);
    /*从HashTable中删除*/
    e = table_.Remove(key, hash);
    if (e != nullptr) {
      /*减少引用计数*/
      last_reference = Unref(e);
      
      /*如果引用计数降为0，则减少Cache中使用容量*/
      if (last_reference) {
        usage_ -= e->charge;
      }
      
      /*如果引用计数降为0，则从LRU中删除*/
      if (last_reference && e->InCache()) {
        LRU_Remove(e);
      }
      e->SetInCache(false);
    }
  }

  // mutex not held here
  // last_reference will only be true if e != nullptr
  if (last_reference) {
    e->Free();
  }
}
```

LRUCacheShard::GetUsage()返回Cache使用容量（包括在LRU中的entries的容量）。
```
size_t LRUCacheShard::GetUsage() const {
  MutexLock l(&mutex_);
  return usage_;
}
```

LRUCacheShard::GetPinnedUsage()返回除LRU中的entries以外Cache entries使用的容量。
```
size_t LRUCacheShard::GetPinnedUsage() const {
  MutexLock l(&mutex_);
  assert(usage_ >= lru_usage_);
  return usage_ - lru_usage_;
}
```

LRUCacheShard::ApplyToAllCacheEntries()在所有的Cache entries上运用callback，如果thread_safe为true，则需要在处理前加锁。
```
void LRUCacheShard::ApplyToAllCacheEntries(void (*callback)(void*, size_t),
                                           bool thread_safe) {
  if (thread_safe) {
    mutex_.Lock();
  }
  
  table_.ApplyToAllCacheEntries(
      [callback](LRUHandle* h) { callback(h->value, h->charge); });
  if (thread_safe) {
    mutex_.Unlock();
  }
}
```

LRUCacheShard::EraseUnRefEntries()从LRUCache分区（LRUCacheShard）中删除那些不被外部引用的Cache entries（这里说明一下什么是外部引用：外部引用就是说有人在使用它，当然HashTable对Cache entry的引用不算做外部引用）：
```
void LRUCacheShard::EraseUnRefEntries() {
  autovector<LRUHandle*> last_reference_list;
  {
    MutexLock l(&mutex_);
    while (lru_.next != &lru_) {
      LRUHandle* old = lru_.next;
      assert(old->InCache());
      assert(old->refs ==
             1);  // LRU list contains elements which may be evicted
      LRU_Remove(old);
      table_.Remove(old->key(), old->hash);
      old->SetInCache(false);
      Unref(old);
      usage_ -= old->charge;
      last_reference_list.push_back(old);
    }
  }

  for (auto entry : last_reference_list) {
    entry->Free();
  }
}
```

LRUCacheShard::LRU_Insert()会将e所指示的元素插入到LRU链表中，根据元素的优先级设定，将其插入到LRU链表的高优先级部分或者低优先级部分：如果high_pri_pool_ratio_不为0，即LRU中存在高优先级部分，且该entry为高优先级元素，则将其插入到LRU中高优先级部分；否则将其插入到LRU中低优先级部分；
    
```
void LRUCacheShard::LRU_Insert(LRUHandle* e) {
  assert(e->next == nullptr);
  assert(e->prev == nullptr);
  if (high_pri_pool_ratio_ > 0 && e->IsHighPri()) {
    // Inset "e" to head of LRU list.
    /*是高优先级entry，则添加到LRU链表头部*/
    e->next = &lru_;
    e->prev = lru_.prev;
    e->prev->next = e;
    e->next->prev = e;
    /*设置其在高优先级pool中*/
    e->SetInHighPriPool(true);
    /*增加高优先级部分使用的容量*/
    high_pri_pool_usage_ += e->charge;
    /*如果高优先级部分超过容量，则调整部分高优先级的Cache entry为低优先级的*/
    MaintainPoolSize();
  } else {
    // Insert "e" to the head of low-pri pool. Note that when
    // high_pri_pool_ratio is 0, head of low-pri pool is also head of LRU list.
    /*否则，添加到低优先级部分的头部*/
    e->next = lru_low_pri_->next;
    e->prev = lru_low_pri_;
    e->prev->next = e;
    e->next->prev = e;
    /*设置其不在高优先级pool中*/
    e->SetInHighPriPool(false);
    /*更新低优先级pool的头部*/
    lru_low_pri_ = e;
  }
  
  /*增加LRU中Cache entries使用的容量*/
  lru_usage_ += e->charge;
}
```

LRUCacheShard::LRU_Remove()从LRU链表中删除某个Cache entry。
```
void LRUCacheShard::LRU_Remove(LRUHandle* e) {
  assert(e->next != nullptr);
  assert(e->prev != nullptr);
  
  /*如果该元素就是LRU中低优先级部分的头，则更新lru中低优先级部分的头*/
  if (lru_low_pri_ == e) {
    lru_low_pri_ = e->prev;
  }
  e->next->prev = e->prev;
  e->prev->next = e->next;
  e->prev = e->next = nullptr;
  /*减少LRU使用容量*/
  lru_usage_ -= e->charge;
  
  if (e->InHighPriPool()) {
    /*如果在高优先级pool中，则减少高优先级pool的使用容量*/
    assert(high_pri_pool_usage_ >= e->charge);
    high_pri_pool_usage_ -= e->charge;
  }
}
```

## LRUCache
LRUCache继承自ShardedCache，并重载了ShardedCache中的相关方法，以支持分区操作，同时LRUCache中的每一个分区类型为LRUCacheShard。从LRUCache的方法中可以看出，它并没有提供插入/查找/删除等封装，而是提供了获取分区的方法，或者获取Cache entry的内容/占用容量/hash值等的方法，对于插入/查找/删除等可以通过GetShard()获取相应的分区，并调用分区相关的方法即可。
```
class LRUCache : public ShardedCache {
 public:
  LRUCache(size_t capacity, int num_shard_bits, bool strict_capacity_limit,
           double high_pri_pool_ratio);
  virtual ~LRUCache();
  
  virtual const char* Name() const override { return "LRUCache"; }
  virtual CacheShard* GetShard(int shard) override;
  virtual const CacheShard* GetShard(int shard) const override;
  virtual void* Value(Handle* handle) override;
  virtual size_t GetCharge(Handle* handle) const override;
  virtual uint32_t GetHash(Handle* handle) const override;
  virtual void DisownData() override;

 private:
  /*分区数组*/
  LRUCacheShard* shards_;
};
```

在LRUCache的构造函数中会分配2^num_shard_bits^个分区，同时为每一个分区设置LRU链表中高优先级部分所占的比例。
```
LRUCache::LRUCache(size_t capacity, int num_shard_bits,
                   bool strict_capacity_limit, double high_pri_pool_ratio)
    : ShardedCache(capacity, num_shard_bits, strict_capacity_limit) {
  /*分区个数为2^num_shard_bits*/
  int num_shards = 1 << num_shard_bits;
  /*分配分区数组*/
  shards_ = new LRUCacheShard[num_shards];
  SetCapacity(capacity);
  SetStrictCapacityLimit(strict_capacity_limit);
  /*为每一个分区设置LRU链表中高优先级pool所占的比例*/
  for (int i = 0; i < num_shards; i++) {
    shards_[i].SetHighPriorityPoolRatio(high_pri_pool_ratio);
  }
}
```

LRUCache的其它方法都比较简单：

```
CacheShard* LRUCache::GetShard(int shard) {
  return reinterpret_cast<CacheShard*>(&shards_[shard]);
}

const CacheShard* LRUCache::GetShard(int shard) const {
  return reinterpret_cast<CacheShard*>(&shards_[shard]);
}

void* LRUCache::Value(Handle* handle) {
  return reinterpret_cast<const LRUHandle*>(handle)->value;
}

size_t LRUCache::GetCharge(Handle* handle) const {
  return reinterpret_cast<const LRUHandle*>(handle)->charge;
}

uint32_t LRUCache::GetHash(Handle* handle) const {
  return reinterpret_cast<const LRUHandle*>(handle)->hash;
}
```

## 创建LRUCache的专用接口
NewLRUCache中根据参数capacity和num_shard_bits来确定分区个数，然后调用LRUCache的构造函数来创建LRUCache。
```
std::shared_ptr<Cache> NewLRUCache(size_t capacity, int num_shard_bits,
                                   bool strict_capacity_limit,
                                   double high_pri_pool_ratio) {
  /*分区个数不能太多*/
  if (num_shard_bits >= 20) {
    return nullptr;  // the cache cannot be sharded into too many fine pieces
  }
  
  /*LRU链表中高优先级pool所占的比例必须介于[0.0, 1.0]之间*/
  if (high_pri_pool_ratio < 0.0 || high_pri_pool_ratio > 1.0) {
    // invalid high_pri_pool_ratio
    return nullptr;
  }
  
  if (num_shard_bits < 0) {
    /* 设置默认的分区个数：先按照每个分区大小为512KB计算可以划分出多少个分区，
     * 如果分区个数超过6个，则设置分区个数为6个
     */
    num_shard_bits = GetDefaultCacheShardBits(capacity);
  }
  
  /*调用LRUCache的构造函数*/
  return std::make_shared<LRUCache>(capacity, num_shard_bits,
                                    strict_capacity_limit, high_pri_pool_ratio);
}
```
