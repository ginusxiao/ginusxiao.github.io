# 参考资料
[qemu Features/Snapshots](https://wiki.qemu.org/Features/Snapshots)

[KVM-QEMU, QCOW2, QEMU-IMG and Snapshots](http://www.azertech.net/content/kvm-qemu-qcow2-qemu-img-and-snapshots)

[The QCOW2 Image Format](http://www.burtonsys.com/qcow-image-format.html)

[*Backing Chain management in QEMU and libvirt by Eric Blake](http://www.linux-kvm.org/images/5/5f/03x02-Eric_Blake-Backing_Chain_Management_in_QEMU_and_libvirt.pdf)

```
该文章中以图示的形式非常直观的讲述了qcow2格式写操作和internal snapshot操作、
external snapshot等操作过程中qcow2元数据的变化过程，很值得一看。
```

# 相关命令

```
qemu-img snapshot -c <snapshot-name> <imagename> 
Create a snapshot and name it snapshot-name. This snapshot is a simple picture
of the VM image state at the time the snapshot is created. 

qemu-img snapshot -l <imagename> 
List all the snapshots in the specified imagename file. 
 
qemu-img snapshot -a <snapshot-name> <imagename> 
Apply a snapshot named snapshot-name. This function simply restores the clusters
that were saved when the snapshot, snapshot-name, was created. It has the effect
of returning the VM image to the state it was in at that time. 

qemu-img snapshot -d <snapshot-name> <imagename> 
Delete the snapshot named snapshot-name from the specified image file, imagename.
Snapshots can gobble-up a significant amount of disk space. The delete command 
does not actually release any disk space allocated to the image file but it does
release the associated clusters - effectively making them available to the VM for
future storage. 
```



# 源码分析
## qemu-img snapshot入口函数

```
static int img_snapshot(int argc, char **argv)
{
    ......

    /* Open the image */
    blk = img_open(image_opts, filename, NULL, bdrv_oflags, false, quiet,
                   force_share);
    bs = blk_bs(blk);

    /* Perform the requested action */
    switch(action) {
    case SNAPSHOT_LIST:
        dump_snapshots(bs);
        break;

    case SNAPSHOT_CREATE:
        memset(&sn, 0, sizeof(sn));
        pstrcpy(sn.name, sizeof(sn.name), snapshot_name);

        qemu_gettimeofday(&tv);
        sn.date_sec = tv.tv_sec;
        sn.date_nsec = tv.tv_usec * 1000;

        ret = bdrv_snapshot_create(bs, &sn);
        break;

    case SNAPSHOT_APPLY:
        ret = bdrv_snapshot_goto(bs, snapshot_name);
        break;

    case SNAPSHOT_DELETE:
        bdrv_snapshot_delete_by_id_or_name(bs, snapshot_name, &err);
        break;
    }
    
    ......
    return 0;
}
```

## 创建snapshot

```
int bdrv_snapshot_create(BlockDriverState *bs,
                         QEMUSnapshotInfo *sn_info)
{
    BlockDriver *drv = bs->drv;
    if (!drv) {
        return -ENOMEDIUM;
    }
    
    /*对于qcow2来说，直接调用qcow2_snapshot_create*/
    if (drv->bdrv_snapshot_create) {
        return drv->bdrv_snapshot_create(bs, sn_info);
    }
    if (bs->file) {
        return bdrv_snapshot_create(bs->file->bs, sn_info);
    }
    return -ENOTSUP;
}
```

```
/* if no id is provided, a new one is constructed */
int qcow2_snapshot_create(BlockDriverState *bs, QEMUSnapshotInfo *sn_info)
{
    BDRVQcow2State *s = bs->opaque;
    QCowSnapshot *new_snapshot_list = NULL;
    QCowSnapshot *old_snapshot_list = NULL;
    QCowSnapshot sn1, *sn = &sn1;
    int i, ret;
    uint64_t *l1_table = NULL;
    int64_t l1_table_offset;

    /*至多创建QCOW_MAX_SNAPSHOTS个snapshot*/
    if (s->nb_snapshots >= QCOW_MAX_SNAPSHOTS) {
        return -EFBIG;
    }

    memset(sn, 0, sizeof(*sn));

    /* Generate an ID */
    /*生成新的snapshot id，在此不表*/
    find_new_snapshot_id(bs, sn_info->id_str, sizeof(sn_info->id_str));

    /* Check that the ID is unique */
    /* 在BDRVQcow2State::snapshots数组中查找是否存在给定@sn_info->id_str的snapshot，
     * 理应不存在，因为这个是新创建的，还没添加进去
     */
    if (find_snapshot_by_id_and_name(bs, sn_info->id_str, NULL) >= 0) {
        return -EEXIST;
    }

    /* Populate sn with passed data */
    sn->id_str = g_strdup(sn_info->id_str);
    sn->name = g_strdup(sn_info->name);

    /*设置sn->disk_size*/
    sn->disk_size = bs->total_sectors * BDRV_SECTOR_SIZE;
    sn->vm_state_size = sn_info->vm_state_size;
    sn->date_sec = sn_info->date_sec;
    sn->date_nsec = sn_info->date_nsec;
    sn->vm_clock_nsec = sn_info->vm_clock_nsec;

    /* Allocate the L1 table of the snapshot and copy the current one there. */
    /* 将snapshot之前的L1 table拷贝一份，专供当前snapshot使用，所以要先为之分配一个
     * cluster，用于存放该L1 table
     */
    l1_table_offset = qcow2_alloc_clusters(bs, s->l1_size * sizeof(uint64_t));

    /*在QCowSnapshot中记录L1 table的地址及其中元素的个数*/
    sn->l1_table_offset = l1_table_offset;
    sn->l1_size = s->l1_size;

    /*为L1 table分配内存*/
    l1_table = g_try_new(uint64_t, s->l1_size);

    /*将现有L1 table中的内容拷贝到新分配的L1 talbe中*/
    for(i = 0; i < s->l1_size; i++) {
        l1_table[i] = cpu_to_be64(s->l1_table[i]);
    }

    /*将该新的L1 table写入镜像文件*/
    ret = bdrv_pwrite(bs->file, sn->l1_table_offset, l1_table,
                      s->l1_size * sizeof(uint64_t));

    g_free(l1_table);
    l1_table = NULL;

    /*
     * Increase the refcounts of all clusters and make sure everything is
     * stable on disk before updating the snapshot table to contain a pointer
     * to the new L1 table.
     */
    /* 增加所有cluster的引用计数，并且确保在更新snapshot table之前关于引用计数的
     * 更新都已经持久化（实际上不是所有的cluster，主要是L2 table和data cluster）
     */
    ret = qcow2_update_snapshot_refcount(bs, s->l1_table_offset, s->l1_size, 1);
    if (ret < 0) {
        goto fail;
    }

    /* Append the new snapshot to the snapshot list */
    /* 分配一个新的snapshot list，将就得snapshot list拷贝过来，并且将当前的snapshot
     * 也添加到新的snapshot list中，后面会将之写入到镜像文件的snapshot table中
     */
    new_snapshot_list = g_new(QCowSnapshot, s->nb_snapshots + 1);
    if (s->snapshots) {
        /*拷贝当前就得snapshot*/
        memcpy(new_snapshot_list, s->snapshots,
               s->nb_snapshots * sizeof(QCowSnapshot));
        old_snapshot_list = s->snapshots;
    }
    s->snapshots = new_snapshot_list;
    /*将当前的snapshot添加进来*/
    s->snapshots[s->nb_snapshots++] = *sn;

    /*将新的snapshot table写入镜像文件中*/
    ret = qcow2_write_snapshots(bs);
    if (ret < 0) {
        g_free(s->snapshots);
        s->snapshots = old_snapshot_list;
        s->nb_snapshots--;
        goto fail;
    }

    g_free(old_snapshot_list);

    /* The VM state isn't needed any more in the active L1 table; in fact, it
     * hurts by causing expensive COW for the next snapshot. */
    /*暂时还不明确vm_state是干嘛的，待以后分析*/
    qcow2_cluster_discard(bs, qcow2_vm_state_offset(s),
                          align_offset(sn->vm_state_size, s->cluster_size),
                          QCOW2_DISCARD_NEVER, false);

#ifdef DEBUG_ALLOC
    {
      BdrvCheckResult result = {0};
      qcow2_check_refcounts(bs, &result, 0);
    }
#endif
    return 0;

fail:
    g_free(sn->id_str);
    g_free(sn->name);
    g_free(l1_table);

    return ret;
}
```

```
/* update the refcounts of snapshots and the copied flag */
int qcow2_update_snapshot_refcount(BlockDriverState *bs,
    int64_t l1_table_offset, int l1_size, int addend)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t *l1_table, *l2_table, l2_offset, entry, l1_size2, refcount;
    bool l1_allocated = false;
    int64_t old_entry, old_l2_offset;
    int i, j, l1_modified = 0, nb_csectors;
    int ret;

    assert(addend >= -1 && addend <= 1);

    l2_table = NULL;
    l1_table = NULL;
    /*传递进来的@l1_size表示L1 table中L1 table entry的数目*/
    l1_size2 = l1_size * sizeof(uint64_t);

    s->cache_discards = true;

    /* WARNING: qcow2_snapshot_goto relies on this function not using the
     * l1_table_offset when it is the current s->l1_table_offset! Be careful
     * when changing this! */
    if (l1_table_offset != s->l1_table_offset) {
        /*分配一片内存，用于存放L1 table结构*/
        l1_table = g_try_malloc0(align_offset(l1_size2, 512));
        if (l1_size2 && l1_table == NULL) {
            ret = -ENOMEM;
            goto fail;
        }
        l1_allocated = true;

        /*从镜像文件中读取L1 table*/
        ret = bdrv_pread(bs->file, l1_table_offset, l1_table, l1_size2);
        if (ret < 0) {
            goto fail;
        }

        /*将L1 table中的内容转换为CPU字节序*/
        for (i = 0; i < l1_size; i++) {
            be64_to_cpus(&l1_table[i]);
        }
    } else {
        assert(l1_size == s->l1_size);
        l1_table = s->l1_table;
        l1_allocated = false;
    }

    /*依次遍历L1 table中的每一个L1 table entry*/
    for (i = 0; i < l1_size; i++) {
        l2_offset = l1_table[i];
        /*如果@l2_offset不为0，表示该L1 table entry对应的L2 table存在*/
        if (l2_offset) {
            old_l2_offset = l2_offset;
            l2_offset &= L1E_OFFSET_MASK;

            /* 如果L2 table cache中有关于@l2_offset所在的cluster的cache，则直接获取之，
             * 否则先从镜像文件中读取，并添加到L2 table cache中
             */
            ret = qcow2_cache_get(bs, s->l2_table_cache, l2_offset,
                (void**) &l2_table);

            /*依次遍历L2 table中的每一个entry*/
            for (j = 0; j < s->l2_size; j++) {
                uint64_t cluster_index;
                uint64_t offset;

                entry = be64_to_cpu(l2_table[j]);
                old_entry = entry;
                /*取消其QCOW_OFLAG_COPIED标识，因为它的引用计数肯定不为1*/
                entry &= ~QCOW_OFLAG_COPIED;
                offset = entry & L2E_OFFSET_MASK;

                switch (qcow2_get_cluster_type(entry)) {
                case QCOW2_CLUSTER_COMPRESSED:
                    /*压缩类型的cluster暂时不关注*/
                    ......
                    break;

                case QCOW2_CLUSTER_NORMAL:
                case QCOW2_CLUSTER_ZERO_ALLOC:
                    /*通过cluster所在的地址@offset获取cluster的索引号*/
                    cluster_index = offset >> s->cluster_bits;
                    assert(cluster_index);
                    if (addend != 0) {
                        /*更新索引号为@cluster_index的cluster的引用计数*/
                        ret = qcow2_update_cluster_refcount(bs,
                                    cluster_index, abs(addend), addend < 0,
                                    QCOW2_DISCARD_SNAPSHOT);
                    }

                    /*获取索引号为@cluster_index的cluster的引用计数*/
                    ret = qcow2_get_refcount(bs, cluster_index, &refcount);
                    break;

                case QCOW2_CLUSTER_ZERO_PLAIN:
                case QCOW2_CLUSTER_UNALLOCATED:
                    refcount = 0;
                    break;

                default:
                    abort();
                }

                /*如果索引号为@cluster_index的cluster引用计数为1，则设置QCOW_OFLAG_COPIED*/
                if (refcount == 1) {
                    entry |= QCOW_OFLAG_COPIED;
                }
                /* 如果索引号为@cluster_index的cluster在L2 table entry中记录信息发生了改变（
                 * L2 table entry中记录的是关于该cluster的地址信息和标识信息，更新引用计数过
                 * 程中，很可能导致QCOW_OFLAG_COPIED的设置或者取消，所以可能会改变），则在
                 * L2 table cache中设置该cluster的信息为脏
                 */
                if (entry != old_entry) {
                    if (addend > 0) {
                        qcow2_cache_set_dependency(bs, s->l2_table_cache,
                            s->refcount_block_cache);
                    }
                    l2_table[j] = cpu_to_be64(entry);
                    qcow2_cache_entry_mark_dirty(bs, s->l2_table_cache,
                                                 l2_table);
                }
            }

            /*更新L2 table cache*/
            qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

            /* 同样要更新@l2_offset所对应的L2 table的引用计数（l2_offset >> s->cluster_bits
             * 就表示L2 table所在的cluster）
             */
            if (addend != 0) {
                ret = qcow2_update_cluster_refcount(bs, l2_offset >>
                                                        s->cluster_bits,
                                                    abs(addend), addend < 0,
                                                    QCOW2_DISCARD_SNAPSHOT);
            }
            
            /*获取@l2_offset所对应的L2 table的引用计数*/
            ret = qcow2_get_refcount(bs, l2_offset >> s->cluster_bits,
                                     &refcount);
            if (ret < 0) {
                goto fail;
            } else if (refcount == 1) {
                /*如果@l2_offset所对应的L2 table的引用计数为1，也要为其设置QCOW_OFLAG_COPIED标识*/
                l2_offset |= QCOW_OFLAG_COPIED;
            }
            
            /*如果@l2_offset跟当初从L1 table中读取出来的不一样，则标记要更新L1 table*/
            if (l2_offset != old_l2_offset) {
                l1_table[i] = l2_offset;
                l1_modified = 1;
            }
        }
    }

    /*确保L2 table cache和refcount table cache flush之后才去更新L1 table*/
    ret = bdrv_flush(bs);
fail:

    /*将最后一个可能尚未添加到L2 table cache中L2 table添加进去（只在失败的情况下会出现）*/
    if (l2_table) {
        qcow2_cache_put(bs, s->l2_table_cache, (void**) &l2_table);
    }

    s->cache_discards = false;
    qcow2_process_discards(bs, ret);

    /* Update L1 only if it isn't deleted anyway (addend = -1) */
    /*如果L1 table发生了更新，则将L1 table的内容写入镜像文件中（确保持久化）*/
    if (ret == 0 && addend >= 0 && l1_modified) {
        for (i = 0; i < l1_size; i++) {
            cpu_to_be64s(&l1_table[i]);
        }

        /*写后会flush*/
        ret = bdrv_pwrite_sync(bs->file, l1_table_offset,
                               l1_table, l1_size2);

        for (i = 0; i < l1_size; i++) {
            be64_to_cpus(&l1_table[i]);
        }
    }
    if (l1_allocated)
        g_free(l1_table);
        
    return ret;
}
```

```
/*
 * Increases or decreases the refcount of a given cluster.
 *
 * @addend is the absolute value of the addend; if @decrease is set, @addend
 * will be subtracted from the current refcount, otherwise it will be added.
 *
 * On success 0 is returned; on failure -errno is returned.
 */
int qcow2_update_cluster_refcount(BlockDriverState *bs,
                                  int64_t cluster_index,
                                  uint64_t addend, bool decrease,
                                  enum qcow2_discard_type type)
{
    BDRVQcow2State *s = bs->opaque;
    int ret;

    ret = update_refcount(bs, cluster_index << s->cluster_bits, 1, addend,
                          decrease, type);
    if (ret < 0) {
        return ret;
    }

    return 0;
}
```

```
static int QEMU_WARN_UNUSED_RESULT update_refcount(BlockDriverState *bs,
                                                   int64_t offset,
                                                   int64_t length,
                                                   uint64_t addend,
                                                   bool decrease,
                                                   enum qcow2_discard_type type)
{
    BDRVQcow2State *s = bs->opaque;
    int64_t start, last, cluster_offset;
    void *refcount_block = NULL;
    int64_t old_table_index = -1;
    int ret;

    ......
    
    start = start_of_cluster(s, offset);
    last = start_of_cluster(s, offset + length - 1);
    for(cluster_offset = start; cluster_offset <= last;
        cluster_offset += s->cluster_size)
    {
        int block_index;
        uint64_t refcount;
        /*cluster索引号*/
        int64_t cluster_index = cluster_offset >> s->cluster_bits;
        /*当前cluster属于哪个refcount block管理，也就是其在refcount table中的索引号是多少*/
        int64_t table_index = cluster_index >> s->refcount_block_bits;

        /* Load the refcount block and allocate it if needed */
        if (table_index != old_table_index) {
            /*发生了refcount block的切换，则将上一个refcount block存放到refcount block cache中*/
            if (refcount_block) {
                qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
            }
            
            /* 如果该refcount block存在，则首先尝试从refcount block cache中加载，如果refcount
             * block cache中没有，则尝试从镜像文件中读取，如果该refcount block不存在，则尝试
             * 分配新的refcount block，并添加到refcount table中管理
             */
            ret = alloc_refcount_block(bs, cluster_index, &refcount_block);
            if (ret < 0) {
                goto fail;
            }
        }
        
        old_table_index = table_index;
        /*在refcount block cache中标记当前正在处理的refcount block是脏的*/
        qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache,
                                     refcount_block);

        /*@block_index表示@cluster_index所代表的cluster在@refcount_block中的索引号*/
        block_index = cluster_index & (s->refcount_block_size - 1);
        /*获取当前refcount block中记录的索引@block_index所指示的cluster的引用计数*/
        refcount = s->get_refcount(refcount_block, block_index);
        /*更新引用计数*/        
        if (decrease) {
            refcount -= addend;
        } else {
            refcount += addend;
        }
        
        /*如果引用计数降为0，则尝试更新free_cluster_index*/
        if (refcount == 0 && cluster_index < s->free_cluster_index) {
            s->free_cluster_index = cluster_index;
        }
        
        /*更新@refcount block这一内存结构中记录的@block_index所指示的cluster的引用计数*/
        s->set_refcount(refcount_block, block_index, refcount);
        /* 如果引用计数为0，则根据@type决定是否discard相应的cluster所占用的空间。
         * 这里的@type为QCOW2_DISCARD_SNAPSHOT，默认的discard_passthrough为true。
         */
        if (refcount == 0 && s->discard_passthrough[type]) {
            update_refcount_discard(bs, cluster_offset, s->cluster_size);
        }
    }

    ret = 0;
fail:
    if (!s->cache_discards) {
        qcow2_process_discards(bs, ret);
    }

    /* Write last changed block to disk */
    /*更新最后一个refcount block到refcount block cache中*/
    if (refcount_block) {
        qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
    }

    /*
     * Try do undo any updates if an error is returned (This may succeed in
     * some cases like ENOSPC for allocating a new refcount block)
     */
    /*如果上述操作失败，则进行回滚*/
    if (ret < 0) {
        int dummy;
        dummy = update_refcount(bs, offset, cluster_offset - offset, addend,
                                !decrease, QCOW2_DISCARD_NEVER);
        (void)dummy;
    }

    return ret;
}
```

```
static void update_refcount_discard(BlockDriverState *bs,
                                    uint64_t offset, uint64_t length)
{
    BDRVQcow2State *s = bs->opaque;
    Qcow2DiscardRegion *d, *p, *next;

    QTAILQ_FOREACH(d, &s->discards, next) {
        uint64_t new_start = MIN(offset, d->offset);
        uint64_t new_end = MAX(offset + length, d->offset + d->bytes);

        if (new_end - new_start <= length + d->bytes) {
            /* There can't be any overlap, areas ending up here have no
             * references any more and therefore shouldn't get freed another
             * time. */
            /* [offset, offset + length)和(d->offset, d->offset + d->bytes)之间有交集，
             * 那么只可能是两者是相邻的discard请求，将[offset, offset + length)合并到
             * discard区域@d中，扩大@d即可
             */
            assert(d->bytes + length == new_end - new_start);
            d->offset = new_start;
            d->bytes = new_end - new_start;
            goto found;
        }
    }

    /*否则，分配新的discard区域并添加到@s->discards中*/
    d = g_malloc(sizeof(*d));
    *d = (Qcow2DiscardRegion) {
        .bs     = bs,
        .offset = offset,
        .bytes  = length,
    };
    QTAILQ_INSERT_TAIL(&s->discards, d, next);

found:
    /* Merge discard requests if they are adjacent now */
    /* 如果扩大后的@d能够和其它discard请求进行合并，则合并之，在合并过程中@d会不断扩大
     * 并参与到后续的合并过程
     */
    QTAILQ_FOREACH_SAFE(p, &s->discards, next, next) {
        if (p == d
            || p->offset > d->offset + d->bytes
            || d->offset > p->offset + p->bytes)
        {
            continue;
        }

        /* Still no overlap possible */
        assert(p->offset == d->offset + d->bytes
            || d->offset == p->offset + p->bytes);

        QTAILQ_REMOVE(&s->discards, p, next);
        d->offset = MIN(d->offset, p->offset);
        d->bytes += p->bytes;
        g_free(p);
    }
}
```

```
void qcow2_process_discards(BlockDriverState *bs, int ret)
{
    BDRVQcow2State *s = bs->opaque;
    Qcow2DiscardRegion *d, *next;

    QTAILQ_FOREACH_SAFE(d, &s->discards, next, next) {
        QTAILQ_REMOVE(&s->discards, d, next);

        /* 调用bdrv_discard进行discard操作，这里使用的是@bs->file->bs，所以
         * 调用的是raw_aio_pdiscard，最终调用handle_aiocb_discard
         */
        if (ret >= 0) {
            bdrv_pdiscard(bs->file->bs, d->offset, d->bytes);
        }

        g_free(d);
    }
}
```

```
static ssize_t handle_aiocb_discard(RawPosixAIOData *aiocb)
{
    int ret = -EOPNOTSUPP;
    BDRVRawState *s = aiocb->bs->opaque;

    if (!s->has_discard) {
        return -ENOTSUP;
    }

    if (aiocb->aio_type & QEMU_AIO_BLKDEV) {
#ifdef BLKDISCARD
        do {
            /*如果支持BLKDISCARD，则通过ioctl实现*/
            uint64_t range[2] = { aiocb->aio_offset, aiocb->aio_nbytes };
            if (ioctl(aiocb->aio_fildes, BLKDISCARD, range) == 0) {
                return 0;
            }
        } while (errno == EINTR);

        ret = -errno;
#endif
    } else {
#ifdef CONFIG_XFS
        if (s->is_xfs) {
            /*如果支持xfs，则采用xfs_discard实现*/
            return xfs_discard(s, aiocb->aio_offset, aiocb->aio_nbytes);
        }
#endif

#ifdef CONFIG_FALLOCATE_PUNCH_HOLE
        /*通过fallocate实现*/
        ret = do_fallocate(s->fd, FALLOC_FL_PUNCH_HOLE | FALLOC_FL_KEEP_SIZE,
                           aiocb->aio_offset, aiocb->aio_nbytes);
#endif
    }

    ret = translate_err(ret);
    if (ret == -ENOTSUP) {
        s->has_discard = false;
    }
    return ret;
}
```

```
/* add at the end of the file a new list of snapshots */
static int qcow2_write_snapshots(BlockDriverState *bs)
{
    BDRVQcow2State *s = bs->opaque;
    QCowSnapshot *sn;
    QCowSnapshotHeader h;
    QCowSnapshotExtraData extra;
    int i, name_size, id_str_size, snapshots_size;
    struct {
        uint32_t nb_snapshots;
        uint64_t snapshots_offset;
    } QEMU_PACKED header_data;
    int64_t offset, snapshots_offset = 0;
    int ret;

    /* compute the size of the snapshots */
    /*计算新的snapshot table要占用的空间大小*/
    offset = 0;
    for(i = 0; i < s->nb_snapshots; i++) {
        sn = s->snapshots + i;
        offset = align_offset(offset, 8);
        offset += sizeof(h);
        offset += sizeof(extra);
        offset += strlen(sn->id_str);
        offset += strlen(sn->name);

        if (offset > QCOW_MAX_SNAPSHOTS_SIZE) {
            ret = -EFBIG;
            goto fail;
        }
    }

    assert(offset <= INT_MAX);
    snapshots_size = offset;

    /* Allocate space for the new snapshot list */
    /*为新的snapshot table 分配空间*/
    snapshots_offset = qcow2_alloc_clusters(bs, snapshots_size);
    offset = snapshots_offset;
    ret = bdrv_flush(bs);

    /* Write all snapshots to the new list */
    for(i = 0; i < s->nb_snapshots; i++) {
        /* 通过snapshot table中记录的每一个QCowSnapshot信息初始化QCowSnapshotHeader和
         * QCowSnapshotExtraData，每一个snapshot都有其自己的QCowSnapshotHeader和
         * QCowSnapshotExtraData，其中记录该snapshot相关的信息
         */
        sn = s->snapshots + i;
        memset(&h, 0, sizeof(h));
        /*该snapshot的QCowSnapshotHeader中与L1 table相关的信息*/
        h.l1_table_offset = cpu_to_be64(sn->l1_table_offset);
        h.l1_size = cpu_to_be32(sn->l1_size);
        /* If it doesn't fit in 32 bit, older implementations should treat it
         * as a disk-only snapshot rather than truncate the VM state */
        if (sn->vm_state_size <= 0xffffffff) {
            h.vm_state_size = cpu_to_be32(sn->vm_state_size);
        }
        h.date_sec = cpu_to_be32(sn->date_sec);
        h.date_nsec = cpu_to_be32(sn->date_nsec);
        h.vm_clock_nsec = cpu_to_be64(sn->vm_clock_nsec);
        h.extra_data_size = cpu_to_be32(sizeof(extra));

        memset(&extra, 0, sizeof(extra));
        extra.vm_state_size_large = cpu_to_be64(sn->vm_state_size);
        extra.disk_size = cpu_to_be64(sn->disk_size);

        id_str_size = strlen(sn->id_str);
        name_size = strlen(sn->name);
        assert(id_str_size <= UINT16_MAX && name_size <= UINT16_MAX);
        h.id_str_size = cpu_to_be16(id_str_size);
        h.name_size = cpu_to_be16(name_size);
        /*确保任何一个snapshot的信息都是从8字节对齐的位置写入的*/
        offset = align_offset(offset, 8);

        /*写入该snapshot的QCowSnapshotHeader*/
        ret = bdrv_pwrite(bs->file, offset, &h, sizeof(h));
        if (ret < 0) {
            goto fail;
        }
        offset += sizeof(h);

        /*写入该snapshot的QCowSnapshotExtraData*/
        ret = bdrv_pwrite(bs->file, offset, &extra, sizeof(extra));
        if (ret < 0) {
            goto fail;
        }
        offset += sizeof(extra);

        /*写入该snapshot的id信息*/
        ret = bdrv_pwrite(bs->file, offset, sn->id_str, id_str_size);
        if (ret < 0) {
            goto fail;
        }
        offset += id_str_size;

        /*写入该snapshot的name信息*/
        ret = bdrv_pwrite(bs->file, offset, sn->name, name_size);
        if (ret < 0) {
            goto fail;
        }
        offset += name_size;
    }

    /*
     * Update the header to point to the new snapshot table. This requires the
     * new table and its refcounts to be stable on disk.
     */
    /*在更新qcow2 的header之前要确保这些信息持久化*/
    ret = bdrv_flush(bs);
    if (ret < 0) {
        goto fail;
    }

    QEMU_BUILD_BUG_ON(offsetof(QCowHeader, snapshots_offset) !=
        offsetof(QCowHeader, nb_snapshots) + sizeof(header_data.nb_snapshots));

    /*@header_data中的信息将会用于更新qcow2 的header*/
    header_data.nb_snapshots        = cpu_to_be32(s->nb_snapshots);
    header_data.snapshots_offset    = cpu_to_be64(snapshots_offset);

    /*采用@header_data中的信息更新qcow2的header*/
    ret = bdrv_pwrite_sync(bs->file, offsetof(QCowHeader, nb_snapshots),
                           &header_data, sizeof(header_data));
    if (ret < 0) {
        goto fail;
    }

    /* free the old snapshot table */
    /*释放旧的snapshot table所占用的空间，因为所有snapshot列表信息都在新的snapshot table中了*/
    qcow2_free_clusters(bs, s->snapshots_offset, s->snapshots_size,
                        QCOW2_DISCARD_SNAPSHOT);
    s->snapshots_offset = snapshots_offset;
    s->snapshots_size = snapshots_size;
    return 0;

fail:
    if (snapshots_offset > 0) {
        qcow2_free_clusters(bs, snapshots_offset, snapshots_size,
                            QCOW2_DISCARD_ALWAYS);
    }
    return ret;
}
```

## 获取snapshot列表

```
static void dump_snapshots(BlockDriverState *bs)
{
    QEMUSnapshotInfo *sn_tab, *sn;
    int nb_sns, i;

    /*获取snapshot table信息@sn_tab*/
    nb_sns = bdrv_snapshot_list(bs, &sn_tab);
    if (nb_sns <= 0)
        return;
    printf("Snapshot list:\n");
    bdrv_snapshot_dump(fprintf, stdout, NULL);
    printf("\n");
    /*打印snapshot table信息*/
    for(i = 0; i < nb_sns; i++) {
        sn = &sn_tab[i];
        bdrv_snapshot_dump(fprintf, stdout, sn);
        printf("\n");
    }
    g_free(sn_tab);
}
```

```
int bdrv_snapshot_list(BlockDriverState *bs,
                       QEMUSnapshotInfo **psn_info)
{
    BlockDriver *drv = bs->drv;
    if (!drv) {
        return -ENOMEDIUM;
    }
    /*对于qcow2来说，直接调用qcow2_snapshot_list*/
    if (drv->bdrv_snapshot_list) {
        return drv->bdrv_snapshot_list(bs, psn_info);
    }
    if (bs->file) {
        return bdrv_snapshot_list(bs->file->bs, psn_info);
    }
    return -ENOTSUP;
}
```

```
int qcow2_snapshot_list(BlockDriverState *bs, QEMUSnapshotInfo **psn_tab)
{
    BDRVQcow2State *s = bs->opaque;
    QEMUSnapshotInfo *sn_tab, *sn_info;
    QCowSnapshot *sn;
    int i;

    if (!s->nb_snapshots) {
        *psn_tab = NULL;
        return s->nb_snapshots;
    }

    /*直接从BlockDriverState中读取snapshot table信息，赋值给@psn_tab即可*/
    sn_tab = g_new0(QEMUSnapshotInfo, s->nb_snapshots);
    for(i = 0; i < s->nb_snapshots; i++) {
        sn_info = sn_tab + i;
        sn = s->snapshots + i;
        pstrcpy(sn_info->id_str, sizeof(sn_info->id_str),
                sn->id_str);
        pstrcpy(sn_info->name, sizeof(sn_info->name),
                sn->name);
        sn_info->vm_state_size = sn->vm_state_size;
        sn_info->date_sec = sn->date_sec;
        sn_info->date_nsec = sn->date_nsec;
        sn_info->vm_clock_nsec = sn->vm_clock_nsec;
    }
    *psn_tab = sn_tab;
    return s->nb_snapshots;
}
```

## 应用指定snapshot

```
int bdrv_snapshot_goto(BlockDriverState *bs,
                       const char *snapshot_id)
{
    BlockDriver *drv = bs->drv;
    int ret, open_ret;

    if (!drv) {
        return -ENOMEDIUM;
    }
    
    /*直接调用qcow2_snapshot_goto*/
    if (drv->bdrv_snapshot_goto) {
        return drv->bdrv_snapshot_goto(bs, snapshot_id);
    }

    ......
}
```

```
/* copy the snapshot 'snapshot_name' into the current disk image */
/*注释写的很清楚，不再赘述*/
int qcow2_snapshot_goto(BlockDriverState *bs, const char *snapshot_id)
{
    BDRVQcow2State *s = bs->opaque;
    QCowSnapshot *sn;
    int i, snapshot_index;
    int cur_l1_bytes, sn_l1_bytes;
    int ret;
    uint64_t *sn_l1_table = NULL;

    /* Search the snapshot */
    snapshot_index = find_snapshot_by_id_or_name(bs, snapshot_id);
    if (snapshot_index < 0) {
        return -ENOENT;
    }
    sn = &s->snapshots[snapshot_index];

    if (sn->disk_size != bs->total_sectors * BDRV_SECTOR_SIZE) {
        error_report("qcow2: Loading snapshots with different disk "
            "size is not implemented");
        ret = -ENOTSUP;
        goto fail;
    }

    /*
     * Make sure that the current L1 table is big enough to contain the whole
     * L1 table of the snapshot. If the snapshot L1 table is smaller, the
     * current one must be padded with zeros.
     */
    ret = qcow2_grow_l1_table(bs, sn->l1_size, true);
    if (ret < 0) {
        goto fail;
    }

    cur_l1_bytes = s->l1_size * sizeof(uint64_t);
    sn_l1_bytes = sn->l1_size * sizeof(uint64_t);

    /*
     * Copy the snapshot L1 table to the current L1 table.
     *
     * Before overwriting the old current L1 table on disk, make sure to
     * increase all refcounts for the clusters referenced by the new one.
     * Decrease the refcount referenced by the old one only when the L1
     * table is overwritten.
     */
    sn_l1_table = g_try_malloc0(cur_l1_bytes);
    if (cur_l1_bytes && sn_l1_table == NULL) {
        ret = -ENOMEM;
        goto fail;
    }

    ret = bdrv_pread(bs->file, sn->l1_table_offset,
                     sn_l1_table, sn_l1_bytes);
    if (ret < 0) {
        goto fail;
    }

    ret = qcow2_update_snapshot_refcount(bs, sn->l1_table_offset,
                                         sn->l1_size, 1);
    if (ret < 0) {
        goto fail;
    }

    ret = qcow2_pre_write_overlap_check(bs, QCOW2_OL_ACTIVE_L1,
                                        s->l1_table_offset, cur_l1_bytes);
    if (ret < 0) {
        goto fail;
    }

    ret = bdrv_pwrite_sync(bs->file, s->l1_table_offset, sn_l1_table,
                           cur_l1_bytes);
    if (ret < 0) {
        goto fail;
    }

    /*
     * Decrease refcount of clusters of current L1 table.
     *
     * At this point, the in-memory s->l1_table points to the old L1 table,
     * whereas on disk we already have the new one.
     *
     * qcow2_update_snapshot_refcount special cases the current L1 table to use
     * the in-memory data instead of really using the offset to load a new one,
     * which is why this works.
     */
    ret = qcow2_update_snapshot_refcount(bs, s->l1_table_offset,
                                         s->l1_size, -1);

    /*
     * Now update the in-memory L1 table to be in sync with the on-disk one. We
     * need to do this even if updating refcounts failed.
     */
    for(i = 0;i < s->l1_size; i++) {
        s->l1_table[i] = be64_to_cpu(sn_l1_table[i]);
    }

    if (ret < 0) {
        goto fail;
    }

    g_free(sn_l1_table);
    sn_l1_table = NULL;

    /*
     * Update QCOW_OFLAG_COPIED in the active L1 table (it may have changed
     * when we decreased the refcount of the old snapshot.
     */
    ret = qcow2_update_snapshot_refcount(bs, s->l1_table_offset, s->l1_size, 0);
    if (ret < 0) {
        goto fail;
    }

#ifdef DEBUG_ALLOC
    {
        BdrvCheckResult result = {0};
        qcow2_check_refcounts(bs, &result, 0);
    }
#endif
    return 0;

fail:
    g_free(sn_l1_table);
    return ret;
}
```


## 删除snapshot

```
int bdrv_snapshot_delete_by_id_or_name(BlockDriverState *bs,
                                       const char *id_or_name,
                                       Error **errp)
{
    int ret;
    Error *local_err = NULL;

    /*先尝试以id删除之*/
    ret = bdrv_snapshot_delete(bs, id_or_name, NULL, &local_err);
    if (ret == -ENOENT || ret == -EINVAL) {
        error_free(local_err);
        local_err = NULL;
        /*再尝试以name删除之*/
        ret = bdrv_snapshot_delete(bs, NULL, id_or_name, &local_err);
    }

    if (ret < 0) {
        error_propagate(errp, local_err);
    }
    return ret;
}
```

```
int bdrv_snapshot_delete(BlockDriverState *bs,
                         const char *snapshot_id,
                         const char *name,
                         Error **errp)
{
    BlockDriver *drv = bs->drv;
    int ret;

    if (!drv) {
        error_setg(errp, QERR_DEVICE_HAS_NO_MEDIUM, bdrv_get_device_name(bs));
        return -ENOMEDIUM;
    }
    if (!snapshot_id && !name) {
        error_setg(errp, "snapshot_id and name are both NULL");
        return -EINVAL;
    }

    /* drain all pending i/o before deleting snapshot */
    bdrv_drained_begin(bs);

    if (drv->bdrv_snapshot_delete) {
        ret = drv->bdrv_snapshot_delete(bs, snapshot_id, name, errp);
    } else if (bs->file) {
        ret = bdrv_snapshot_delete(bs->file->bs, snapshot_id, name, errp);
    } else {
        error_setg(errp, "Block format '%s' used by device '%s' "
                   "does not support internal snapshot deletion",
                   drv->format_name, bdrv_get_device_name(bs));
        ret = -ENOTSUP;
    }

    bdrv_drained_end(bs);
    return ret;
}
```

```
/*和qcow2_snapshot_create相反的过程，注释的很清楚*/
int qcow2_snapshot_delete(BlockDriverState *bs,
                          const char *snapshot_id,
                          const char *name,
                          Error **errp)
{
    BDRVQcow2State *s = bs->opaque;
    QCowSnapshot sn;
    int snapshot_index, ret;

    /* Search the snapshot */
    snapshot_index = find_snapshot_by_id_and_name(bs, snapshot_id, name);
    if (snapshot_index < 0) {
        error_setg(errp, "Can't find the snapshot");
        return -ENOENT;
    }
    sn = s->snapshots[snapshot_index];

    /* Remove it from the snapshot list */
    memmove(s->snapshots + snapshot_index,
            s->snapshots + snapshot_index + 1,
            (s->nb_snapshots - snapshot_index - 1) * sizeof(sn));
    s->nb_snapshots--;
    ret = qcow2_write_snapshots(bs);

    /*
     * The snapshot is now unused, clean up. If we fail after this point, we
     * won't recover but just leak clusters.
     */
    g_free(sn.id_str);
    g_free(sn.name);

    /*
     * Now decrease the refcounts of clusters referenced by the snapshot and
     * free the L1 table.
     */
    ret = qcow2_update_snapshot_refcount(bs, sn.l1_table_offset,
                                         sn.l1_size, -1);
    qcow2_free_clusters(bs, sn.l1_table_offset, sn.l1_size * sizeof(uint64_t),
                        QCOW2_DISCARD_SNAPSHOT);

    /* must update the copied flag on the current cluster offsets */
    ret = qcow2_update_snapshot_refcount(bs, s->l1_table_offset, s->l1_size, 0);

    return 0;
}
```
