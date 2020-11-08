# 参考文档
[qcow2-cache.txt](https://github.com/qemu/qemu/blob/master/docs/qcow2-cache.txt)


# qcow2 cache
## 基础数据结构

```
typedef struct Qcow2CachedTable {
    /*L2 table/refcount table 所在的cluster地址*/
    int64_t  offset;
    /*在L2 table cache/refcount table cache的lru中的位置，用于缓存替换*/
    uint64_t lru_counter;
    /*使用还有调用正在使用该cache entry，如果有，则该cache entry不能被替换*/
    int      ref;
    /*相应的cache entry是否是脏的，如果为脏，则需要flush*/
    bool     dirty;
} Qcow2CachedTable;

struct Qcow2Cache {
    /*元信息数组*/
    Qcow2CachedTable       *entries;
    /*当前cache所依赖的其它cache*/
    struct Qcow2Cache      *depends;
    /*cache的大小（以cluster为单位）*/
    int                     size;
    /*在flush之前是否需要先flush已经写入到镜像文件中的数据*/
    bool                    depends_on_flush;
    /*cache 内存区*/
    void                   *table_array;
    /*当前的lru计数，用于更新cache entry的引用计数*/
    uint64_t                lru_counter;
    /* 上一次执行cache清理的时候看到的lru_counter，用于在下一次清理的时候
     * 清理那些自从上一次清理以来未曾被访问过的cache entry
     */
    uint64_t                cache_clean_lru_counter;
};

在qcow2中无论是refcount block、L2 table等都是统一利用Qcow2Cache来管理其缓存的，
refcount block对应的Qcow2Cache被称为refcount block cache，L2 table对应的
Qcow2Cache被称为L2 table cache。

以L2 table cache为例，假设L2 table占用l2_table_size个cluster，L2 table cache中
包含一片大小为l2_table_cache_size个cluster大小的内存区域（就是Qcow2Cache
中的table_array这个域），l2_table_size和l2_table_cache_size不一定相等，正式因为
l2_table_size和l2_table_cache_size不一定相等，L2 table和L2 table cache之间并非
一一映射，这样就需要有一个元信息去指示L2 table cache中的某个cluster是对应于L2
table中的哪个cluster，这个元信息就对应于Qcow2Cache中的entries这个域。

               ________ ________ ________ ____ ________ ________
entries       |________|________|________|____ ________|________|
                  |                               |
               ___V____ ________ ________ ____ ___V____ ________
table_array   |        |        |        |             |        |
              |        |        |        |             |        |
              |________|________|________|____ ________|________|
                           ^        ^                      ^
                           |________.______________________^_________
                                    .                      ^         |
                   ..................                      ^         |
               ____|___ ________ ________ ____ ________ ___^____ ____|___ ________
img           |        |        |        |             |        |        |        |
              |        |        |L2 table|             |        |        |        |
              |________|________|________|_____ _______|________|________|________|

                   
```

## qcow2中cache创建

```
qcow2_open
    |- qcow2_do_open
        |- qcow2_update_options
            |- qcow2_update_options_prepare
```

```
static int qcow2_update_options_prepare(BlockDriverState *bs,
                                        Qcow2ReopenState *r,
                                        QDict *options, int flags,
                                        Error **errp)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t l2_cache_size, refcount_cache_size;
    int ret;

    qdict_extract_subqdict(options, &encryptopts, "encrypt.");
    encryptfmt = qdict_get_try_str(encryptopts, "format");
    opts = qemu_opts_create(&qcow2_runtime_opts, NULL, 0, &error_abort);
    qemu_opts_absorb_qdict(opts, options, &local_err);

    /* get L2 table/refcount block cache size from command line options */
    /*从命令行选项中获取L2 table cache和refcount block cache的大小*/
    read_cache_sizes(bs, opts, &l2_cache_size, &refcount_cache_size,
                     &local_err);

    /*将以字节为单位的L2 table cache和refcount block cache的大小转换为以cluster个数为单位*/
    l2_cache_size /= s->cluster_size;
    if (l2_cache_size < MIN_L2_CACHE_SIZE) {
        l2_cache_size = MIN_L2_CACHE_SIZE;
    }
    
    refcount_cache_size /= s->cluster_size;
    if (refcount_cache_size < MIN_REFCOUNT_CACHE_SIZE) {
        refcount_cache_size = MIN_REFCOUNT_CACHE_SIZE;
    }
    
    /* alloc new L2 table/refcount block cache, flush old one */
    /*分配新的L2 table/refcount block cache之前，先flush旧的L2 table/refcount block cache*/
    if (s->l2_table_cache) {
        ret = qcow2_cache_flush(bs, s->l2_table_cache);
    }

    if (s->refcount_block_cache) {
        ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    }

    r->l2_table_cache = qcow2_cache_create(bs, l2_cache_size);
    r->refcount_block_cache = qcow2_cache_create(bs, refcount_cache_size);

    ......
    
    return ret;
}
```

```
Qcow2Cache *qcow2_cache_create(BlockDriverState *bs, int num_tables)
{
    BDRVQcow2State *s = bs->opaque;
    Qcow2Cache *c;

    /*分配新的Qcow2Cache管理结构*/
    c = g_new0(Qcow2Cache, 1);
    /*Qcow2Cache大小（以cluster为单位）*/
    c->size = num_tables;
    /*Qcow2Cache元数据区，包含@num_tables个Qcow2CachedTable结构*/
    c->entries = g_try_new0(Qcow2CachedTable, num_tables);
    /*Qcow2Cache缓存区，包含@num_tables个cluster大小的内存区域*/
    c->table_array = qemu_try_blockalign(bs->file->bs,
                                     (size_t) num_tables * s->cluster_size);
                                     
    if (!c->entries || !c->table_array) {
        qemu_vfree(c->table_array);
        g_free(c->entries);
        g_free(c);
        c = NULL;
    }

    return c;
}
```

## qcow2中cache使用接口
当前qcow2 cache使用方式如下：
获取信息：
```
    /* 从refcount block cache中获取@refcount_block_offset所在的cluster在cache中的内容，
     * 存放于@refcount_block中
     */
    ret = qcow2_cache_get(bs, s->refcount_block_cache, refcount_block_offset,
                          &refcount_block);
    block_index = cluster_index & (s->refcount_block_size - 1);
    /*从@refcount_block中读取@block_index所代表的cluster的引用计数*/
    *refcount = s->get_refcount(refcount_block, block_index);
    /*不再使用refcount block cache了，调用qcow2_cache_put接口，减少其引用计数*/
    qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
```

更新信息：
```
    /* 从L2 table cache中获取@l2_offset所在的cluster在cache中的内容，存放于@l2_table中*/
    ret = qcow2_cache_get(bs, s->l2_table_cache, l2_offset, (void**) &l2_table);
    /*读取l2 table中第@j个entry所代表的cluster的地址信息*/
    entry = be64_to_cpu(l2_table[j]);
    /*取消地址信息中的QCOW_OFLAG_COPIED标识*/
    entry &= ~QCOW_OFLAG_COPIED;
    /*将更新后的@entry更新到l2_table中，此时实际上就更新了L2 table cache的内容了*/
    l2_table[j] = cpu_to_be64(entry);
    /*标记L2 table cache中相应的entry为脏*/
    qcow2_cache_entry_mark_dirty(bs, s->l2_table_cache, l2_table);
    /*不再使用refcount block cache了，调用qcow2_cache_put接口，减少其引用计数*/
    qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);
```

```
int qcow2_cache_get(BlockDriverState *bs, Qcow2Cache *c, uint64_t offset,
    void **table)
{
    return qcow2_cache_do_get(bs, c, offset, table, true);
}
```

```
static int qcow2_cache_do_get(BlockDriverState *bs, Qcow2Cache *c,
    uint64_t offset, void **table, bool read_from_disk)
{
    BDRVQcow2State *s = bs->opaque;
    int i;
    int ret;
    int lookup_index;
    uint64_t min_lru_counter = UINT64_MAX;
    int min_lru_index = -1;

    /* Check if the table is already cached */
    /*默认的@lookup_index这里开始查找*/
    i = lookup_index = (offset / s->cluster_size * 4) % c->size;
    do {
        const Qcow2CachedTable *t = &c->entries[i];
        if (t->offset == offset) {
            /*如果缓存中记录的第@i个cluster是关于@offset所在的cluster的缓存*/
            goto found;
        }
        
        /* 如果没有找到，且当前查找的entry引用计数为0，则当前的entry是可以被替换
         * 的，但是这里并不急于替换，一方面要尝试查找其它的entry，看是否能够找到
         *，另一方面，要找到那个lru_counter最小的entry，即那个最近最少访问的entry
         */
        if (t->ref == 0 && t->lru_counter < min_lru_counter) {
            min_lru_counter = t->lru_counter;
            min_lru_index = i;
        }
        
        /*如果找到了最后一个位置，则回到第一个位置继续*/
        if (++i == c->size) {
            i = 0;
        }
    } while (i != lookup_index);

    /*没有引用计数为0的，在当前不可能出现*/
    if (min_lru_index == -1) {
        /* This can't happen in current synchronous code, but leave the check
         * here as a reminder for whoever starts using AIO with the cache */
        abort();
    }

    /* Cache miss: write a table back and replace it */
    /* 在缓存中未命中，则替换@i所在的cluster，用于缓存@offset所在的cluster的数据，
     * 但是在替换之前，要确保旧的cluster的数据被flush
     */
    i = min_lru_index;
    ret = qcow2_cache_entry_flush(bs, c, i);
    if (ret < 0) {
        return ret;
    }

    c->entries[i].offset = 0;
    if (read_from_disk) {
        /* 从镜像文件中读取@offset所在的cluster的数据，存放到Qcow2Cache中相应的地址
         * qcow2_cache_get_table_addr(bs, c, i)用于获取缓存中第@i个cluster的地址，
         * 当bdrv_pread返回的时候，需要缓存的数据已经存在于Qcow2Cache中了
         */
        ret = bdrv_pread(bs->file, offset,
                         qcow2_cache_get_table_addr(bs, c, i),
                         s->cluster_size);
        if (ret < 0) {
            return ret;
        }
    }

    /*更新元信息，指示第@i个cluster中缓存的是镜像文件中该@offset的cluster的数据*/
    c->entries[i].offset = offset;

found:
    /*增加该entry的引用计数，表示还有使用者正在使用之，不能被替换*/
    c->entries[i].ref++;
    /*返回缓存地址*/
    *table = qcow2_cache_get_table_addr(bs, c, i);

    trace_qcow2_cache_get_done(qemu_coroutine_self(),
                               c == s->l2_table_cache, i);

    return 0;
}
```

```
void qcow2_cache_put(BlockDriverState *bs, Qcow2Cache *c, void **table)
{
    /*获取@*table在Qcow2Cache @c中的索引（以cluster为单位）*/
    int i = qcow2_cache_get_table_idx(bs, c, *table);

    /* 与qcow2_cache_get -> qcow2_cache_do_get中相对应，减少其引用计数，
     * 每当调用qcow2_cache_get获取缓存之后，在不用该缓存的时候都必须调用
     * qcow2_cache_put减少其引用计数，以便在缓存替换的时候可以找到被替换
     * 的entry
     */
    c->entries[i].ref--;
    *table = NULL;

    if (c->entries[i].ref == 0) {
        c->entries[i].lru_counter = ++c->lru_counter;
    }

    assert(c->entries[i].ref >= 0);
}
```

```
/*获取@table在Qcow2Cache中的索引（以cluster为单位）*/
static inline int qcow2_cache_get_table_idx(BlockDriverState *bs,
                  Qcow2Cache *c, void *table)
{
    BDRVQcow2State *s = bs->opaque;
    ptrdiff_t table_offset = (uint8_t *) table - (uint8_t *) c->table_array;
    int idx = table_offset / s->cluster_size;
    assert(idx >= 0 && idx < c->size && table_offset % s->cluster_size == 0);
    return idx;
}
```

```
void qcow2_cache_entry_mark_dirty(BlockDriverState *bs, Qcow2Cache *c,
     void *table)
{
    /*获取@table对应的索引编号*/
    int i = qcow2_cache_get_table_idx(bs, c, table);
    assert(c->entries[i].offset != 0);
    /*设置dirty标识位true*/
    c->entries[i].dirty = true;
}
```


## qcow2中cache读取

```
int qcow2_cache_get(BlockDriverState *bs, Qcow2Cache *c, uint64_t offset,
    void **table)
{
    return qcow2_cache_do_get(bs, c, offset, table, true);
}
```

```
static int qcow2_cache_do_get(BlockDriverState *bs, Qcow2Cache *c,
    uint64_t offset, void **table, bool read_from_disk)
{
    BDRVQcow2State *s = bs->opaque;
    int i;
    int ret;
    int lookup_index;
    uint64_t min_lru_counter = UINT64_MAX;
    int min_lru_index = -1;

    /* Check if the table is already cached */
    /*默认的@lookup_index这里开始查找*/
    i = lookup_index = (offset / s->cluster_size * 4) % c->size;
    do {
        const Qcow2CachedTable *t = &c->entries[i];
        if (t->offset == offset) {
            /*如果缓存中记录的第@i个cluster是关于@offset所在的cluster的缓存*/
            goto found;
        }
        
        /* 如果没有找到，且当前查找的entry引用计数为0，则当前的entry是可以被替换
         * 的，但是这里并不急于替换，一方面要尝试查找其它的entry，看是否能够找到
         *，另一方面，要找到那个lru_counter最小的entry，即那个最近最少访问的entry
         */
        if (t->ref == 0 && t->lru_counter < min_lru_counter) {
            min_lru_counter = t->lru_counter;
            min_lru_index = i;
        }
        
        /*如果找到了最后一个位置，则回到第一个位置继续*/
        if (++i == c->size) {
            i = 0;
        }
    } while (i != lookup_index);

    /*没有引用计数为0的，在当前不可能出现*/
    if (min_lru_index == -1) {
        /* This can't happen in current synchronous code, but leave the check
         * here as a reminder for whoever starts using AIO with the cache */
        abort();
    }

    /* Cache miss: write a table back and replace it */
    /* 在缓存中未命中，则替换@i所在的cluster，用于缓存@offset所在的cluster的数据，
     * 但是在替换之前，要确保旧的cluster的数据被flush
     */
    i = min_lru_index;
    ret = qcow2_cache_entry_flush(bs, c, i);
    if (ret < 0) {
        return ret;
    }

    c->entries[i].offset = 0;
    if (read_from_disk) {
        /* 从镜像文件中读取@offset所在的cluster的数据，存放到Qcow2Cache中相应的地址
         * qcow2_cache_get_table_addr(bs, c, i)用于获取缓存中第@i个cluster的地址，
         * 当bdrv_pread返回的时候，需要缓存的数据已经存在于Qcow2Cache中了
         */
        ret = bdrv_pread(bs->file, offset,
                         qcow2_cache_get_table_addr(bs, c, i),
                         s->cluster_size);
        if (ret < 0) {
            return ret;
        }
    }

    /*更新元信息，指示第@i个cluster中缓存的是镜像文件中该@offset的cluster的数据*/
    c->entries[i].offset = offset;

found:
    /*增加该entry的引用计数，表示还有使用者正在使用之，不能被替换*/
    c->entries[i].ref++;
    /*返回缓存地址*/
    *table = qcow2_cache_get_table_addr(bs, c, i);

    trace_qcow2_cache_get_done(qemu_coroutine_self(),
                               c == s->l2_table_cache, i);

    return 0;
}
```

## qcow2中cache落盘

```
int qcow2_cache_flush(BlockDriverState *bs, Qcow2Cache *c)
{
    /*将qcow2 cache中的内容写入镜像文件中*/
    int result = qcow2_cache_write(bs, c);

    if (result == 0) {
        /*确保cache中的内容持久化，注意这里的参数为@bs->file->bs*/
        int ret = bdrv_flush(bs->file->bs);
        if (ret < 0) {
            result = ret;
        }
    }

    return result;
}
```

```
int qcow2_cache_write(BlockDriverState *bs, Qcow2Cache *c)
{
    BDRVQcow2State *s = bs->opaque;
    int result = 0;
    int ret;
    int i;

    for (i = 0; i < c->size; i++) {
        /*依次针对每一个cache entry调用flush*/
        ret = qcow2_cache_entry_flush(bs, c, i);
        if (ret < 0 && result != -ENOSPC) {
            result = ret;
        }
    }

    return result;
}
```

```
static int qcow2_cache_entry_flush(BlockDriverState *bs, Qcow2Cache *c, int i)
{
    BDRVQcow2State *s = bs->opaque;
    int ret = 0;

    /*如果该cache entry不为脏，或者该cache entry无效，则直接返回*/
    if (!c->entries[i].dirty || !c->entries[i].offset) {
        return 0;
    }

    if (c->depends) {
        /*如果当前准备执行flush的cache依赖于其它cache，则先确保它所依赖的cache flush*/
        ret = qcow2_cache_flush_dependency(bs, c);
    } else if (c->depends_on_flush) {
        /*如果在flush当前的cache entry之前必须要确保之前写入文件的数据持久化的话*/
        ret = bdrv_flush(bs->file->bs);
        if (ret >= 0) {
            c->depends_on_flush = false;
        }
    }

    ......
    
    /*将第@i个cache entry的内容写入镜像文件中*/
    ret = bdrv_pwrite(bs->file, c->entries[i].offset,
                      qcow2_cache_get_table_addr(bs, c, i), s->cluster_size);
    if (ret < 0) {
        return ret;
    }

    /*标记该cache entry和镜像文件已经同步了*/
    c->entries[i].dirty = false;

    return 0;
}
```

```
static int qcow2_cache_flush_dependency(BlockDriverState *bs, Qcow2Cache *c)
{
    int ret;

    /*先确保@c所依赖的cache flush，如果c->depends所代表的cache有其自己的依赖的话，将会递归调用*/
    ret = qcow2_cache_flush(bs, c->depends);
    if (ret < 0) {
        return ret;
    }
    
    /*@c的依赖已经刷新，设置其依赖为空*/
    c->depends = NULL;
    /*因为调用qcow2_cache_flush时候，会调用bdrv_flush，所以在flush @c之前无需再flush了*/
    c->depends_on_flush = false;

    return 0;
}
```

```
int bdrv_flush(BlockDriverState *bs)
{
    Coroutine *co;
    FlushCo flush_co = {
        .bs = bs,
        .ret = NOT_DONE,
    };

    /*总之，最后都会在协程中调用bdrv_flush_co_entry*/
    if (qemu_in_coroutine()) {
        /* Fast-path if already in coroutine context */
        bdrv_flush_co_entry(&flush_co);
    } else {
        co = qemu_coroutine_create(bdrv_flush_co_entry, &flush_co);
        bdrv_coroutine_enter(bs, co);
        BDRV_POLL_WHILE(bs, flush_co.ret == NOT_DONE);
    }

    return flush_co.ret;
}
```

```
static void coroutine_fn bdrv_flush_co_entry(void *opaque)
{
    FlushCo *rwco = opaque;

    rwco->ret = bdrv_co_flush(rwco->bs);
}
```

```
int coroutine_fn bdrv_co_flush(BlockDriverState *bs)
{
    int current_gen;
    int ret = 0;

    bdrv_inc_in_flight(bs);
    qemu_co_mutex_lock(&bs->reqs_lock);
    /*当前最新写版本号*/
    current_gen = atomic_read(&bs->write_gen);

    /* Wait until any previous flushes are completed */
    /*等待正在进行中的flush请求完成*/
    while (bs->active_flush_req) {
        qemu_co_queue_wait(&bs->flush_queue, &bs->reqs_lock);
    }

    /* Flushes reach this point in nondecreasing current_gen order.  */
    /*设置正在进行flush*/
    bs->active_flush_req = true;
    qemu_co_mutex_unlock(&bs->reqs_lock);

    /* Write back all layers by calling one driver function */
    /* 在qcow2_cache_flush -> bdrv_flush -> bdrv_flush_co_entry -> bdrv_co_flush调用上
     * 下文中，这里的bs->drv是@bdrv_file，它没有提供bdrv_co_flush接口函数
     */
    if (bs->drv->bdrv_co_flush) {
        ret = bs->drv->bdrv_co_flush(bs);
        goto out;
    }

    /* Write back cached data to the OS even with cache=unsafe */
    /* 在qcow2_cache_flush -> bdrv_flush -> bdrv_flush_co_entry -> bdrv_co_flush调用上
     * 下文中，这里的bs->drv是@bdrv_file，它也没有提供bdrv_co_flush_to_os接口函数
     */
    BLKDBG_EVENT(bs->file, BLKDBG_FLUSH_TO_OS);
    if (bs->drv->bdrv_co_flush_to_os) {
        ret = bs->drv->bdrv_co_flush_to_os(bs);
        if (ret < 0) {
            goto out;
        }
    }

    /* But don't actually force it to the disk with cache=unsafe */
    /*对于cache=unsafe模式，就不执行flush了*/
    if (bs->open_flags & BDRV_O_NO_FLUSH) {
        goto flush_parent;
    }

    /* Check if we really need to flush anything */
    /*如果已经flush到最新的写版本号，则不执行flush*/
    if (bs->flushed_gen == current_gen) {
        goto flush_parent;
    }

    /* 在qcow2_cache_flush -> bdrv_flush -> bdrv_flush_co_entry -> bdrv_co_flush调用上
     * 下文中，这里的bs->drv是@bdrv_file，它提供了.bdrv_aio_flush = raw_aio_flush
     */
    if (bs->drv->bdrv_co_flush_to_disk) {
        ret = bs->drv->bdrv_co_flush_to_disk(bs);
    } else if (bs->drv->bdrv_aio_flush) {
        BlockAIOCB *acb;
        CoroutineIOCompletion co = {
            .coroutine = qemu_coroutine_self(),
        };

        acb = bs->drv->bdrv_aio_flush(bs, bdrv_co_io_em_complete, &co);
        if (acb == NULL) {
            ret = -EIO;
        } else {
            qemu_coroutine_yield();
            ret = co.ret;
        }
    } else {
        /*
         * Some block drivers always operate in either writethrough or unsafe
         * mode and don't support bdrv_flush therefore. Usually qemu doesn't
         * know how the server works (because the behaviour is hardcoded or
         * depends on server-side configuration), so we can't ensure that
         * everything is safe on disk. Returning an error doesn't work because
         * that would break guests even if the server operates in writethrough
         * mode.
         *
         * Let's hope the user knows what he's doing.
         */
        ret = 0;
    }

    if (ret < 0) {
        goto out;
    }

    /* Now flush the underlying protocol.  It will also have BDRV_O_NO_FLUSH
     * in the case of cache=unsafe, so there are no useless flushes.
     */
flush_parent:
    /* 在qcow2_cache_flush -> bdrv_flush -> bdrv_flush_co_entry -> bdrv_co_flush调用上
     * 下文中，这里的bs->file为空
     */
    ret = bs->file ? bdrv_co_flush(bs->file->bs) : 0;
    
out:
    /* Notify any pending flushes that we have completed */
    if (ret == 0) {
        /*更新flush版本号*/
        bs->flushed_gen = current_gen;
    }

    qemu_co_mutex_lock(&bs->reqs_lock);
    bs->active_flush_req = false;
    /*唤醒处于等待队列中的下一个flush相关的协程*/
    qemu_co_queue_next(&bs->flush_queue);
    qemu_co_mutex_unlock(&bs->reqs_lock);

early_exit:
    bdrv_dec_in_flight(bs);
    return ret;
}
```

```
static BlockAIOCB *raw_aio_flush(BlockDriverState *bs, BlockCompletionFunc *cb, void *opaque)
{
    /*本函数用于将镜像文件中的数据flush到物理介质中*/
    BDRVRawState *s = bs->opaque;

    if (fd_open(bs) < 0)
        return NULL;

    /*注意这里传递的最后一个参数QEMU_AIO_FLUSH，表明的是IO类型*/
    return paio_submit(bs, s->fd, 0, NULL, 0, cb, opaque, QEMU_AIO_FLUSH);
}
```

```
static BlockAIOCB *paio_submit(BlockDriverState *bs, int fd,
        int64_t offset, QEMUIOVector *qiov, int bytes,
        BlockCompletionFunc *cb, void *opaque, int type)
{
    RawPosixAIOData *acb = g_new(RawPosixAIOData, 1);
    ThreadPool *pool;

    acb->bs = bs;
    /*IO 类型保存与此，在当前上下文中是QEMU_AIO_FLUSH*/
    acb->aio_type = type;
    acb->aio_fildes = fd;

    acb->aio_nbytes = bytes;
    acb->aio_offset = offset;

    if (qiov) {
        acb->aio_iov = qiov->iov;
        acb->aio_niov = qiov->niov;
        assert(qiov->size == acb->aio_nbytes);
    }

    trace_paio_submit(acb, opaque, offset, bytes, type);
    pool = aio_get_thread_pool(bdrv_get_aio_context(bs));
    /*最终执行函数是aio_worker*/
    return thread_pool_submit_aio(pool, aio_worker, acb, cb, opaque);
}
```

```
static int aio_worker(void *arg)
{
    RawPosixAIOData *aiocb = arg;
    ssize_t ret = 0;

    switch (aiocb->aio_type & QEMU_AIO_TYPE_MASK) {
    ......
    
    case QEMU_AIO_FLUSH:
        ret = handle_aiocb_flush(aiocb);
        break;
        
    ......
    
    }
}
```

```
static ssize_t handle_aiocb_flush(RawPosixAIOData *aiocb)
{
    BDRVRawState *s = aiocb->bs->opaque;
    int ret;

    /*如果page_cache_inconsistent为true，则可能当前的cache有问题，所以不执行flush，直接返回EIO*/
    if (s->page_cache_inconsistent) {
        return -EIO;
    }

    ret = qemu_fdatasync(aiocb->aio_fildes);
    if (ret == -1) {
        /* There is no clear definition of the semantics of a failing fsync(),
         * so we may have to assume the worst. The sad truth is that this
         * assumption is correct for Linux. Some pages are now probably marked
         * clean in the page cache even though they are inconsistent with the
         * on-disk contents. The next fdatasync() call would succeed, but no
         * further writeback attempt will be made. We can't get back to a state
         * in which we know what is on disk (we would have to rewrite
         * everything that was touched since the last fdatasync() at least), so
         * make bdrv_flush() fail permanently. Given that the behaviour isn't
         * really defined, I have little hope that other OSes are doing better.
         *
         * Obviously, this doesn't affect O_DIRECT, which bypasses the page
         * cache. */
         
        /*在qemu_fdatasync失败的情况下，设置page_cache_inconsistent为true*/
        if ((s->open_flags & O_DIRECT) == 0) {
            s->page_cache_inconsistent = true;
        }
        return -errno;
    }
    return 0;
}

int qemu_fdatasync(int fd)
{
#ifdef CONFIG_FDATASYNC
    return fdatasync(fd);
#else
    return fsync(fd);
#endif
}
```

## qcow2中cache清理
qcow2_open
    |- qcow2_do_open
        |- qcow2_update_options
            |- qcow2_update_options_commit


```
static void qcow2_update_options_commit(BlockDriverState *bs,
                                        Qcow2ReopenState *r)
{
    BDRVQcow2State *s = bs->opaque;

    ......
    
    /*如果更新了cache清理的间隔时间，则重新初始化cache清理定时器*/
    if (s->cache_clean_interval != r->cache_clean_interval) {
        cache_clean_timer_del(bs);
        s->cache_clean_interval = r->cache_clean_interval;
        cache_clean_timer_init(bs, bdrv_get_aio_context(bs));
    }

    ......
}
```

```
static void cache_clean_timer_init(BlockDriverState *bs, AioContext *context)
{
    BDRVQcow2State *s = bs->opaque;
    if (s->cache_clean_interval > 0) {
        /*创建一个cache清理定时器，定时时间到，则调用cache_clean_timer_cb执行cache清理*/
        s->cache_clean_timer = aio_timer_new(context, QEMU_CLOCK_VIRTUAL,
                                             SCALE_MS, cache_clean_timer_cb,
                                             bs);
        /*设置cache清理的时间间隔*/
        timer_mod(s->cache_clean_timer, qemu_clock_get_ms(QEMU_CLOCK_VIRTUAL) +
                  (int64_t) s->cache_clean_interval * 1000);
    }
}
```

```
static void cache_clean_timer_cb(void *opaque)
{
    BlockDriverState *bs = opaque;
    BDRVQcow2State *s = bs->opaque;
    /*分别清理L2 table cache和refcount block cache*/
    qcow2_cache_clean_unused(bs, s->l2_table_cache);
    qcow2_cache_clean_unused(bs, s->refcount_block_cache);
    /*设置下一次cache清理时间点*/
    timer_mod(s->cache_clean_timer, qemu_clock_get_ms(QEMU_CLOCK_VIRTUAL) +
              (int64_t) s->cache_clean_interval * 1000);
}
```

```
void qcow2_cache_clean_unused(BlockDriverState *bs, Qcow2Cache *c)
{
    int i = 0;
    while (i < c->size) {
        int to_clean = 0;

        /* Skip the entries that we don't need to clean */
        /*略过那些不能被清理的连续的cache entry（比如正在被使用，或者为脏等）*/
        while (i < c->size && !can_clean_entry(c, i)) {
            i++;
        }

        /* And count how many we can clean in a row */
        /*找到那些连续的可以被清理的cache entry*/
        while (i < c->size && can_clean_entry(c, i)) {
            c->entries[i].offset = 0;
            c->entries[i].lru_counter = 0;
            i++;
            to_clean++;
        }

        if (to_clean > 0) {
            /*清理从@(i - to_clean)开始的@to_clean个cache entry*/
            qcow2_cache_table_release(bs, c, i - to_clean, to_clean);
        }
    }

    /* 标记，执行cache清理的时候，Qcow2Cache中当前最大的lru号是多少，用于下次清理
     * 过程中识别出那些在两次清理之间未曾被使用过的cache entry
     */
    c->cache_clean_lru_counter = c->lru_counter;
}
```

```
static inline bool can_clean_entry(Qcow2Cache *c, int i)
{
    /* 如果某个cache entry没有任何调用在使用它，且它不为脏，且它当中包含的是有效数据，
     * 且它自从上次调用qcow2_cache_clean_unused清理cache以来未曾被访问过，则可以清理
     */
    Qcow2CachedTable *t = &c->entries[i];
    return t->ref == 0 && !t->dirty && t->offset != 0 &&
        t->lru_counter <= c->cache_clean_lru_counter;
}
```

```
static void qcow2_cache_table_release(BlockDriverState *bs, Qcow2Cache *c,
                                      int i, int num_tables)
{
/* Using MADV_DONTNEED to discard memory is a Linux-specific feature */
/* 使用madvise配合MADV_DONTNEED来释放内存(关于madvise和MADV_DONTNEED，请参考
 * http://www.man7.org/linux/man-pages/man2/madvise.2.html)*/
#ifdef CONFIG_LINUX
    BDRVQcow2State *s = bs->opaque;
    /*获取即将清理的cache entry的内存地址*/
    void *t = qcow2_cache_get_table_addr(bs, c, i);
    int align = getpagesize();
    size_t mem_size = (size_t) s->cluster_size * num_tables;
    size_t offset = QEMU_ALIGN_UP((uintptr_t) t, align) - (uintptr_t) t;
    size_t length = QEMU_ALIGN_DOWN(mem_size - offset, align);
    if (length > 0) {
        madvise((uint8_t *) t + offset, length, MADV_DONTNEED);
    }
#endif
}
```

## qcow2中cache依赖
在Qcow2Cache定义中有depends和depends_on_flush两个域，这里我们看看这两个域的作用
及其使用，如下：
```
struct Qcow2Cache {
    Qcow2CachedTable       *entries;
    struct Qcow2Cache      *depends;
    int                     size;
    bool                    depends_on_flush;
    void                   *table_array;
    uint64_t                lru_counter;
    uint64_t                cache_clean_lru_counter;
};
```
查看depends的使用，在如下三个函数中被使用到：

```
static int qcow2_cache_entry_flush(BlockDriverState *bs, Qcow2Cache *c, int i)
{
    /*如果即将flush的cache有依赖，要先确保flush它所依赖的cache*/
    if (c->depends) {
        ret = qcow2_cache_flush_dependency(bs, c);
    }
    
    ......
}

static int qcow2_cache_flush_dependency(BlockDriverState *bs, Qcow2Cache *c)
{
    int ret;

    /*flush cache @c的依赖*/
    ret = qcow2_cache_flush(bs, c->depends);
    if (ret < 0) {
        return ret;
    }

    /*一旦它所依赖的cahe被flush了，就不再依赖任何cache了*/
    c->depends = NULL;
    c->depends_on_flush = false;

    return 0;
}

int qcow2_cache_set_dependency(BlockDriverState *bs, Qcow2Cache *c,
    Qcow2Cache *dependency)
{
    int ret;

    /* 如果cache @c的依赖@dependency有其自己的依赖，则要先确保这个依赖被flush，
     * 这是为了避免循环依赖？
     */
    if (dependency->depends) {
        ret = qcow2_cache_flush_dependency(bs, dependency);
        if (ret < 0) {
            return ret;
        }
    }

    /* 如果cache @c已经有依赖，且跟即将设置的依赖@dependency不一样，则先将其
     * 当前的依赖flush
     */
    if (c->depends && (c->depends != dependency)) {
        ret = qcow2_cache_flush_dependency(bs, c);
        if (ret < 0) {
            return ret;
        }
    }

    /*设置其依赖为@dependency*/
    c->depends = dependency;
    return 0;
}
```
根据上面可以看出，设置依赖就是为了控制flush的顺序，假设cache A依赖于cache B，cache
B 依赖于cache C，则在flush过程中一定会按照cache C -> cache B -> cache A的顺序进行。
那么在qcow2中又是如何使用qcow2_cache_set_dependency的呢？其调用如下：
```
int qcow2_alloc_cluster_link_l2(BlockDriverState *bs, QCowL2Meta *m)
{
    ......
    
    /* 在qcow2_co_pwritev -> qcow2_alloc_cluster_link_l2之前已经经历了qcow2_co_pwritev
     * -> qcow2_alloc_cluster_offset -> handle_alloc -> do_alloc_cluster_offset
     * -> qcow2_alloc_clusters | qcow2_alloc_clusters_at -> update_refcount对于新分配
     * 的cluster在refcount block cache中更新了其引用计数，而在qcow2_alloc_cluster_link_l2
     * 中会更新L2 table信息到L2 table cache，这里通过调用qcow2_cache_set_dependency确保
     * 在flush L2 table  cache之前需要flush refcount block cache。因为refcount中记录的是
     * 某个cluster的引用计数，进而反应其分配情况，L2 table中记录的则是guest cluster到host
     * cluster的映射关系，只有在镜像文件中分配了host cluster（refcount block中记录了其引用
     * 计数不为0），才应该出现在L2 table中。
     */
    if (qcow2_need_accurate_refcounts(s)) {
        qcow2_cache_set_dependency(bs, s->l2_table_cache,
                                   s->refcount_block_cache);
    }
    
    ......
}

static int QEMU_WARN_UNUSED_RESULT update_refcount(BlockDriverState *bs,
                                       int64_t offset,
                                       int64_t length,
                                       uint64_t addend,
                                       bool decrease,
                                       enum qcow2_discard_type type)
{
    ......
    
    /* 在分配cluster/释放cluster/创建快照/删除快照的过程中都可能会更新某些cluster
     * 的引用计数，但是只有在释放cluster/删除快照的过程中才会减少引用计数，即将
     * @decrease设置为true，在这里只有当减少引用计数的情况下才会设置refcount
     * block cache依赖于L2 table cache。这是因为，在减少引用计数的情况下，某些
     * cluster可能引用计数会降为0，那么这些cluster就不应该存在于L2 table中了，
     * 在将这些cluster的引用计数设置为0之前，必须先将其从L2 table中删除，否则可能
     * 会读取到错误的数据？
     */
    if (decrease) {
        qcow2_cache_set_dependency(bs, s->refcount_block_cache,
            s->l2_table_cache);
    }
    
    ......
}

int64_t qcow2_alloc_bytes(BlockDriverState *bs, int size)
{
    ......
    
    /* The cluster refcount was incremented; refcount blocks must be flushed
     * before the caller's L2 table updates. 
     */
    qcow2_cache_set_dependency(bs, s->l2_table_cache, s->refcount_block_cache);
    
    ......
    
}

int qcow2_update_snapshot_refcount(BlockDriverState *bs,
    int64_t l1_table_offset, int l1_size, int addend)
{
    for (i = 0; i < l1_size; i++) {
        l2_offset = l1_table[i];
        if (l2_offset) {
            ......
            
            for (j = 0; j < s->l2_size; j++) {
                entry = be64_to_cpu(l2_table[j]);
                old_entry = entry;
                entry &= ~QCOW_OFLAG_COPIED;
                offset = entry & L2E_OFFSET_MASK;

                switch (qcow2_get_cluster_type(entry)) {
                    ......
                }

                if (refcount == 1) {
                    entry |= QCOW_OFLAG_COPIED;
                }
                
                if (entry != old_entry) {
                    /* 可能是因为引用计数发生改变导致L2 table entry中记录的信息（包括地址和
                     * 相关标识）发生改变，必须确保先flush refcount block，再flush L2 table
                     */
                    if (addend > 0) {
                        qcow2_cache_set_dependency(bs, s->l2_table_cache,
                            s->refcount_block_cache);
                    }
                    
                    l2_table[j] = cpu_to_be64(entry);
                    qcow2_cache_entry_mark_dirty(bs, s->l2_table_cache,
                                                 l2_table);
                }
            }

            ......
        }
    } 
}
```