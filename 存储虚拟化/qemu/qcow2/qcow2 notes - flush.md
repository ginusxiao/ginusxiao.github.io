# 源码分析
## qemu block层触发的flush

```
blk_co_pwritev
    |- bdrv_co_pwritev
        |- bdrv_aligned_pwritev
            |- bdrv_driver_pwritev [根据“qemu && qcow2 notes - write”中的分析，
            bdrv_aligned_pwritev 中如果需要分片传输，则会多次调用bdrv_driver_pwritev，
            但是不管调用多少次，至多会在最后一次调用之后会调用bdrv_co_flush，至于是
            否会被调用在取决于是否设置了BDRV_REQ_FUA标识]
                |- bdrv_co_flush [如果写请求执行成功，且设置了BDRV_REQ_FUA标识]
static int coroutine_fn bdrv_driver_pwritev(BlockDriverState *bs,
                                            uint64_t offset, uint64_t bytes,
                                            QEMUIOVector *qiov, int flags)
{
    ......
    
emulate_flags:
    if (ret == 0 && (flags & BDRV_REQ_FUA)) {
        ret = bdrv_co_flush(bs);
    }

    return ret;
}

int coroutine_fn bdrv_co_flush(BlockDriverState *bs)
{
    int current_gen;
    int ret = 0;

    /* Write back all layers by calling one driver function */
    /*bdrv_qcow2和bdrv_file都没有提供该接口*/
    if (bs->drv->bdrv_co_flush) {
        ret = bs->drv->bdrv_co_flush(bs);
        goto out;
    }

    /* Write back cached data to the OS even with cache=unsafe */
    /* bdrv_qcow2提供了该接口.bdrv_co_flush_to_os = qcow2_co_flush_to_os,
     * bdrv_file则没有提供该接口 
     */
    BLKDBG_EVENT(bs->file, BLKDBG_FLUSH_TO_OS);
    if (bs->drv->bdrv_co_flush_to_os) {
        ret = bs->drv->bdrv_co_flush_to_os(bs);
        if (ret < 0) {
            goto out;
        }
    }

    /* But don't actually force it to the disk with cache=unsafe */
    /*对于cache=unsafe模式，才会设置BDRV_O_NO_FLUSH*/
    if (bs->open_flags & BDRV_O_NO_FLUSH) {
        goto flush_parent;
    }

    /* Check if we really need to flush anything */
    /*如果flush的版本号已经和最新的write版本号相同，则无事可做了*/
    if (bs->flushed_gen == current_gen) {
        goto flush_parent;
    }

    /* 对于bdrv_qcow2既没有提供bdrv_co_flush_to_disk接口也没有提供
     * .bdrv_aio_flush接口，bdrv_file没有提供bdrv_co_flush_to_disk
     * 接口，但是提供了.bdrv_aio_flush = raw_aio_flush接口
     */
    BLKDBG_EVENT(bs->file, BLKDBG_FLUSH_TO_DISK);
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
        ret = 0;
    }

    if (ret < 0) {
        goto out;
    }

    /* Now flush the underlying protocol.  It will also have BDRV_O_NO_FLUSH
     * in the case of cache=unsafe, so there are no useless flushes.
     */
flush_parent:
    /* 如果其存在protocol layer，则继续在protocol layer上执行bdrv_co_flush*/
    ret = bs->file ? bdrv_co_flush(bs->file->bs) : 0;
    
out:
    qemu_co_mutex_lock(&bs->reqs_lock);
    bs->active_flush_req = false;
    /* Return value is ignored - it's ok if wait queue is empty */
    qemu_co_queue_next(&bs->flush_queue);
    qemu_co_mutex_unlock(&bs->reqs_lock);

early_exit:
    bdrv_dec_in_flight(bs);
    return ret;
}
```

```
static coroutine_fn int qcow2_co_flush_to_os(BlockDriverState *bs)
{
    BDRVQcow2State *s = bs->opaque;
    int ret;

    qemu_co_mutex_lock(&s->lock);
    /*将L2 table cache写入镜像文件中（可能存在于host os page cache中）*/
    ret = qcow2_cache_write(bs, s->l2_table_cache);
    if (ret < 0) {
        qemu_co_mutex_unlock(&s->lock);
        return ret;
    }

    if (qcow2_need_accurate_refcounts(s)) {
        /* 如果需要精确引用计数，则将refcount block cache写入镜像文件中
         *（可能存在于host os page cache中）
         */
        ret = qcow2_cache_write(bs, s->refcount_block_cache);
        if (ret < 0) {
            qemu_co_mutex_unlock(&s->lock);
            return ret;
        }
    }
    qemu_co_mutex_unlock(&s->lock);

    return 0;
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

static ssize_t handle_aiocb_flush(RawPosixAIOData *aiocb)
{
    BDRVRawState *s = aiocb->bs->opaque;
    int ret;

    /* 如果page_cache_inconsistent为true，则可能当前的cache有问题，所以不执行flush，
     * 直接返回EIO
     */
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

```
而BDRV_REQ_FUA标识是在blk_co_pwritev中设置的，如下：
int coroutine_fn blk_co_pwritev(BlockBackend *blk, int64_t offset,
                                unsigned int bytes, QEMUIOVector *qiov,
                                BdrvRequestFlags flags)
{
    int ret;
    BlockDriverState *bs = blk_bs(blk);

    ......
    
    /* 如果设置了enable_write_cache，则设置BDRV_REQ_FUA标识，而根据“qemu && qcow2
     * notes - block cache 模式参数解析”一文中关于各种cache模式的分析，可知在设置
     * 了cache.write_back的情况下会设置enable_write_cache为true，也就是没有设置
     * cache.write_back的情况下会设置BDRV_REQ_FUA标识
     */
    if (!blk->enable_write_cache) {
        flags |= BDRV_REQ_FUA;
    }

    ret = bdrv_co_pwritev(blk->root, offset, bytes, qiov, flags);
    
    return ret;
}
```

```
综上，如果没有设置cache.write_back，则qemu block层会在每次写IO请求执行完毕的时候，主
动调用一次bdrv_co_flush。对于protocol layer为file的qcow2来说，qemu block layer触发
的flush分以下两种：
1. cache是unsafe模式：
    调用qcow2_co_flush_to_os，然后返回；
    
2. cache不是unsafe模式：
    依次调用qcow2_co_flush_to_os和raw_aio_flush之后返回；
    
qcow2_co_flush_to_os会确保将qcow2中的L2 table cache刷到镜像文件中（但是数据可能在
host os page cache中），同时在需要精确的引用计数的情况下，将refcount block cache
刷到镜像文件中（但是数据可能在host os page cache中）。

raw_aio_flush则通过fdatasync或者fsync确保镜像文件在硬盘中的数据（包括L2 table、
refcount block和虚拟磁盘的数据）都是最新的。
```


## qcow2内部触发的flush
qcow2 L1 table相关
```
qcow2_grow_l1_table |
qcow2_write_l1_entry
    |- qcow2_cache_flush
    |- bdrv_pwrite_sync
            |- bdrv_flush
因为L1 table没有cache，所有关于它的更新都是同步更新，所以写了镜像文件之后会调用bdrv_flush。
    
/*该函数用于扩大L1 table，需要分配新的L1 table*/        
int qcow2_grow_l1_table(BlockDriverState *bs, uint64_t min_size,
                        bool exact_size)
{
    ......
    
    /*分配新的L1 table*/
    new_l1_table_offset = qcow2_alloc_clusters(bs, new_l1_size2);
    
    /* 在分配新的L1 table的过程中，会更新refcount block cache，因为后面要同步
     * 更新L1 table，所以这里先flush refcount block cache
     */
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    if (ret < 0) {
        goto fail;
    }

    /*同步更新L1 table*/
    ret = bdrv_pwrite_sync(bs->file, new_l1_table_offset,
                           new_l1_table, new_l1_size2);
                           
    /*同步更新Qcow2 header*/
    ret = bdrv_pwrite_sync(bs->file, offsetof(QCowHeader, l1_size),
                           data, sizeof(data));
    ......
    
    return ret;
}

/*该函数用于更新L1 table中的某个entry*/
int qcow2_write_l1_entry(BlockDriverState *bs, int l1_index)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t buf[L1_ENTRIES_PER_SECTOR] = { 0 };
    int l1_start_index;
    int i, ret;

    /* 为了避免read-modify-write，在更新@l1_index所指示的entry的内容到L1 table的时候
     * ，会将@l1_index所在的一整个sector都更新下去，找到@l1_index所指示的entry所在的
     * sector，并将这个sector的内容读取到buf中
     */
    l1_start_index = l1_index & ~(L1_ENTRIES_PER_SECTOR - 1);
    for (i = 0; i < L1_ENTRIES_PER_SECTOR && l1_start_index + i < s->l1_size;
         i++)
    {
        buf[i] = cpu_to_be64(s->l1_table[l1_start_index + i]);
    }

    /*同步更新*/
    ret = bdrv_pwrite_sync(bs->file,
                           s->l1_table_offset + 8 * l1_start_index,
                           buf, sizeof(buf));
    return 0;
}
```

qcow2 refcount block相关
```
alloc_refcount_block |
rebuild_refcount_structure |
qcow2_refcount_area
    |- bdrv_pwrite_sync
            |- bdrv_flush
            
static int alloc_refcount_block(BlockDriverState *bs,
                    int64_t cluster_index, void **refcount_block)
{
    /*计算@cluster_index率属于哪个refcount table*/
    refcount_table_index = cluster_index >> s->refcount_block_bits;
    /* 检查@refcount_table_index所代表的refcount block是否存在，如果存在则
     * 直接从refcount block cache中返回该refcount block或者从镜像文件中加载
     * 该refcount block
     */
    if (refcount_table_index < s->refcount_table_size) {
        uint64_t refcount_block_offset =
            s->refcount_table[refcount_table_index] & REFT_OFFSET_MASK;

        /* If it's already there, we're done */
        if (refcount_block_offset) {
             /*从refcount block cache中返回该refcount block或者从镜像文件中加载*/
             return load_refcount_block(bs, refcount_block_offset,
                                        refcount_block);
        }
    } 
    
    /*否则该refcount block尚不存在，则需要新分配*/
    /* 因为新分配refcount block会更新refcount table，而refcount table没有cache，
     * refcount table只能执行同步更新，在更新之前先确保L2 table cache被flush
     */
    ret = qcow2_cache_flush(bs, s->l2_table_cache);
    
    /*分配新的refcount block*/
    int64_t new_block = alloc_clusters_noref(bs, s->cluster_size);
    
    /* 增加该新分配的refcount block的自身的引用计数，这里将在管理该新分配的refcount
     * block cluster的refcount block（姑且称之为parent refcount block）中更新相应的
     * 计数
     */
    ret = update_refcount(bs, new_block, s->cluster_size, 1, false,
                          QCOW2_DISCARD_NEVER);
    
    /*因为前面在refcount block cache中更新了parent refcount block*/
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    
    /*在refcount block cache中标记该新分配的refcount block是脏的*/
    qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache, *refcount_block);
    /*flush该新分配的refcount block，因为后面马上要同步更新refcount table*/
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    
    /*同步更新refcount table*/
    ret = bdrv_pwrite_sync(bs->file,
        s->refcount_table_offset + refcount_table_index * sizeof(uint64_t),
        &data64, sizeof(data64));
}

rebuild_refcount_structure 和qcow2_refcount_area这里就不分析了。
```

qcow2 snapshot相关：
```
qcow2_update_snapshot_refcount |
qcow2_write_snapshots
    |- bdrv_flush
    
qcow2_update_snapshot_refcount |
qcow2_write_snapshots |
qcow2_snapshot_goto
    |- bdrv_pwrite_sync
            |- bdrv_flush

/*该函数用于在创建或者删除快照之后，更新引用计数*/
int qcow2_update_snapshot_refcount(BlockDriverState *bs,
    int64_t l1_table_offset, int l1_size, int addend)
{
    if (l1_table_offset != s->l1_table_offset) {
        /*为该snapshot分配自身的L1 table*/
        l1_table = g_try_malloc0(align_offset(l1_size2, 512));
        if (l1_size2 && l1_table == NULL) {
            ret = -ENOMEM;
            goto fail;
        }
        l1_allocated = true;

        /*从镜像文件的L1 table中读取内容到snapshot自身的L1 table中*/
        ret = bdrv_pread(bs->file, l1_table_offset, l1_table, l1_size2);
        if (ret < 0) {
            goto fail;
        }

        for (i = 0; i < l1_size; i++) {
            be64_to_cpus(&l1_table[i]);
        }
    } else {
        assert(l1_size == s->l1_size);
        l1_table = s->l1_table;
        l1_allocated = false;
    }

    for (i = 0; i < l1_size; i++) {
        /*L2 table 地址*/
        l2_offset = l1_table[i];
        if (l2_offset) {
            /*获取L2 table内容*/
            ret = qcow2_cache_get(bs, s->l2_table_cache, l2_offset,
                (void**) &l2_table);

            /*遍历L2 table，更新L2 table中管理的每一个cluster的引用计数*/
            for (j = 0; j < s->l2_size; j++) {
                entry = be64_to_cpu(l2_table[j]);
                old_entry = entry;
                entry &= ~QCOW_OFLAG_COPIED;
                offset = entry & L2E_OFFSET_MASK;

                switch (qcow2_get_cluster_type(entry)) {
                ......

                case QCOW2_CLUSTER_NORMAL:
                case QCOW2_CLUSTER_ZERO_ALLOC:
                    cluster_index = offset >> s->cluster_bits;
                    if (addend != 0) {
                        /*更新cluster的引用计数*/                        
                        ret = qcow2_update_cluster_refcount(bs,
                                    cluster_index, abs(addend), addend < 0,
                                    QCOW2_DISCARD_SNAPSHOT);
                    }

                    ret = qcow2_get_refcount(bs, cluster_index, &refcount);
                    break;

                ......
                }

                if (refcount == 1) {
                    entry |= QCOW_OFLAG_COPIED;
                }
                
                if (entry != old_entry) {
                    if (addend > 0) {
                        qcow2_cache_set_dependency(bs, s->l2_table_cache,
                            s->refcount_block_cache);
                    }
                    l2_table[j] = cpu_to_be64(entry);
                    /* 如果更新引用计数前后L2 table entry中记录的内容有变，则
                     * 标记该entry为脏
                     */
                    qcow2_cache_entry_mark_dirty(bs, s->l2_table_cache,
                                                 l2_table);
                }
            }

            qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

            if (addend != 0) {
                /*更新L2 table所在的cluster的引用计数*/
                ret = qcow2_update_cluster_refcount(bs, l2_offset >>
                                                        s->cluster_bits,
                                                    abs(addend), addend < 0,
                                                    QCOW2_DISCARD_SNAPSHOT);
            }
            ret = qcow2_get_refcount(bs, l2_offset >> s->cluster_bits,
                                     &refcount);
            if (ret < 0) {
                goto fail;
            } else if (refcount == 1) {
                l2_offset |= QCOW_OFLAG_COPIED;
            }
            
            /*如果L1 table entry中记录的内容有变，则标记L1 table需要更新*/
            if (l2_offset != old_l2_offset) {
                l1_table[i] = l2_offset;
                l1_modified = 1;
            }
        }
    }

    /* 因为后面可能需要更新L1 table，所以这里调用bdrv_flush确保前面更新的refcount 
     * block cache和L2 table cache都被flush
     */
    ret = bdrv_flush(bs);
fail:
    if (l2_table) {
        qcow2_cache_put(bs, s->l2_table_cache, (void**) &l2_table);
    }

    ......
    
    /* Update L1 only if it isn't deleted anyway (addend = -1) */
    /*如果需要更新L1 table，则同步更新之*/
    if (ret == 0 && addend >= 0 && l1_modified) {
        for (i = 0; i < l1_size; i++) {
            cpu_to_be64s(&l1_table[i]);
        }

        ret = bdrv_pwrite_sync(bs->file, l1_table_offset,
                               l1_table, l1_size2);
    }
    
    ......
    
    return ret;
}

/*该函数用于将snapshot table写入镜像文件中*/
static int qcow2_write_snapshots(BlockDriverState *bs)
{
    /* Allocate space for the new snapshot list */
    /*为snapshot table（即snapshot list）分配空间*/
    snapshots_offset = qcow2_alloc_clusters(bs, snapshots_size);
    offset = snapshots_offset;
    
    /*因为写入snapshot table之后需要flush，所以在此先确保任何cache中的脏数据先被flush*/
    ret = bdrv_flush(bs);

    /*依次写入snapshot table中的每一个snapshot信息*/
    for(i = 0; i < s->nb_snapshots; i++) {
        ......

        /*写入snapshot header信息*/
        ret = bdrv_pwrite(bs->file, offset, &h, sizeof(h));
        offset += sizeof(h);

        /*写入snapshot extradata信息*/
        ret = bdrv_pwrite(bs->file, offset, &extra, sizeof(extra));
        offset += sizeof(extra);

        /*写入snapshot id信息*/
        ret = bdrv_pwrite(bs->file, offset, sn->id_str, id_str_size);
        offset += id_str_size;
        
        /*写入snapshot name信息*/
        ret = bdrv_pwrite(bs->file, offset, sn->name, name_size);
        offset += name_size;
    }

    /*
     * Update the header to point to the new snapshot table. This requires the
     * new table and its refcounts to be stable on disk.
     */
    /*因为后面要更新Qcow2 Header，所以这里要先确保写入的snapshot table信息都持久化*/
    ret = bdrv_flush(bs);

    header_data.nb_snapshots        = cpu_to_be32(s->nb_snapshots);
    header_data.snapshots_offset    = cpu_to_be64(snapshots_offset);

    /*更新Qcow2 header，记录snapshot table的位置和大小等信息*/
    ret = bdrv_pwrite_sync(bs->file, offsetof(QCowHeader, nb_snapshots),
                           &header_data, sizeof(header_data));
    if (ret < 0) {
        goto fail;
    }

    ......
    
    return 0;
}
```

qcow2 cache相关：
```
qcow2_cache_entry_flush |
qcow2_cache_flush
    |- bdrv_flush

qcow2 cache相关的更新都是被动的更新，也就是不是cache自身触发的更新，
而是由L1 table更新、refcount block更新、snapshot更新等触发的被动的更新。
```

qcow2 header更新相关：
```
qcow2_mark_dirty
qcow2_mark_clean
qcow2_mark_consistent
    |- bdrv_flush
因为这3个函数都会更新Qcow2 header信息，更新之后都必须同步，所以要调用bdrv_flush。    
```

qcow2 其它相关：
```
qcow2_truncate |
    |- bdrv_flush
    |- bdrv_pwrite_sync
            |- bdrv_flush
因为qcow2_truncate（现在qcow2 truncate支持extend，不支持shrink）在preallocate模式不为
PREALLOC_MODE_OFF的情况下会预分配cluster，因此会更新refcount block cache和L2 table
cache等，而qcow2_truncate还会更新Qcow2 header中的size（镜像文件大小）这个域，所以
在更新Qcow2 header之前，需要先调用bdrv_flush确保任何cache中的数据都持久化，然后再
调用bdrv_pwrite_sync同步更新header信息。
            
qcow2_reopen_prepare
    |- bdrv_flush
qcow2_reopen_prepare在打开镜像文件的时候，如果设置了以read-only模式打开，则需要先将
缓存中的数据flush。
```

qcow2 bitmap相关：

```
update_header_sync |
update_ext_header_and_dir
    |- bdrv_flush

略...
```

qcow2内部的flush的时候做了些什么？

```
在qcow2内部都是通过调用bdrv_flush来进行flush的。
bdrv_flush
    |- bdrv_flush_co_entry
        |- bdrv_co_flush
可以看到bdrv_flush最终也是调用bdrv_co_flush来实现flush逻辑的，关于
bdrv_co_flush在本文中“qemu block层触发的flush”中已经分析过了。

当然，在qcow2内部调用bdrv_flush的时候，可能会同时走到bdrv_qcow2和bdrv_file
这两层driver，也可能只经过bdrv_file这一层driver，具体该走两层还是一层取决于
传递的BlockDriverState是什么。
```

## qcow2 flush总结
```
综上，qcow2 flush包括两种触发机制，一种是qemu block layer在每次写请求执行完毕之后触发
的，是否触发取决于cache.write_back模式是否设置，如果没有设置cache.write_back则qemu
block层会在每次写IO请求执行完毕的时候，主动调用一次bdrv_co_flush。对于protocol layer
为file的qcow2来说，qemu block layer触发的flush分以下两种：
1. cache是unsafe模式：
    调用qcow2_co_flush_to_os，然后返回；
    
2. cache不是unsafe模式：
    依次调用qcow2_co_flush_to_os和raw_aio_flush之后返回；
    
qcow2_co_flush_to_os会确保将qcow2中的L2 table cache刷到镜像文件中（但是数据可能在
host os page cache中），同时在需要精确的引用计数的情况下，将refcount block cache
刷到镜像文件中（但是数据可能在host os page cache中）。

raw_aio_flush则通过fdatasync或者fsync确保镜像文件在硬盘中的数据（包括L2 table、
refcount block和虚拟磁盘的数据）都是最新的。

另一种是qcow2内部触发的flush，这种flush主要是与qcow2内部的cache、header和snapshot table
等有关系，但是无论是L1 table相关、refcount block相关、snapshot相关、header相关等等，
其本质上都是由于以下两点：
1. 同步更新：
    Qcow2 header必须是同步更新；
    L1 table必须是同步更新；
    refcount table必须是同步更新；
    snapshot table必须是同步更新；
2. flush依赖
    在任何同步更新之前，都必须确保cache中的数据被flush；
    cache被flush之前，必须确保它所依赖的cache被flush；
```

