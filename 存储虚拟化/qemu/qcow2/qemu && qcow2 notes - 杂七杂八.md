# qemu
## qemu相关参考资料
[QEMU version 2.10.92 User Documentation](https://qemu.weilnetz.de/doc/qemu-doc.html#Block-device-options)

## 怎么理解qemu block driver中的BDRV_REQ_FUA？
暂时没有找到相关资料直接讲述BDRV_REQ_FUA，但是在linux内核中有REQ_FUA的概念，
估计是qemu中借鉴之的，内核中的REQ_FUA是为了实现barrier request（可以简单理解
为防止请求乱序执行）。

关于REQ_FUA可以参考[这里](https://hipporoll.net/blog/2014/05/08/how-to-achieve-block-io-ordering-in-qemu/)或者[这里](http://www.zhimengzhe.com/linux/158617.html)。

在qemu中，BDRV_REQ_FUA标识被用于控制是否flush。qemu block driver中的BDRV_REQ_FUA是在哪里被设置的？
以virtio blk为例：
```
virtio_blk_handle_request
    - virtio_blk_submit_multireq
        - submit_requests
            - blk_aio_pwritev [blk_aio_pwritev的第4个参数@flags类型为BdrvRequestFlags，当前设置为0]
                - blk_aio_prwv(blk, offset, qiov->size, qiov, blk_aio_write_entry, flags, cb, opaque) [这里会设置BlkRwCo::flags为blk_aio_pwritev中指定的@flags，类型也为BdrvRequestFlags]

blk_aio_write_entry
    - blk_co_pwritev(rwco->blk, rwco->offset, acb->bytes, rwco->qiov, rwco->flags) [这里的rwco->flags就是在blk_aio_prwv中设置的]

int coroutine_fn blk_co_pwritev(BlockBackend *blk, int64_t offset,
                                unsigned int bytes, QEMUIOVector *qiov,
                                BdrvRequestFlags flags)
{
    int ret;
    BlockDriverState *bs = blk_bs(blk);

    ......
    
    /* 如果该BlockBackend没有enable write cache，则设置BDRV_REQ_FUA标识，
     * 关于BlockBackend::enable_write_cache的设置参考“BlockBackend cache
     * 相关的设置”
     */
    if (!blk->enable_write_cache) {
        flags |= BDRV_REQ_FUA;
    }

    ret = bdrv_co_pwritev(blk->root, offset, bytes, qiov, flags);
    
    ......
    
    return ret;
}
```

在qemu中BDRV_REQ_FUA又起到什么作用？BDRV_REQ_FUA在bdrv_co_flush中用到：

```
int coroutine_fn bdrv_co_flush(BlockDriverState *bs)
{
    int current_gen;
    int ret = 0;

    bdrv_inc_in_flight(bs);

    if (!bdrv_is_inserted(bs) || bdrv_is_read_only(bs) ||
        bdrv_is_sg(bs)) {
        goto early_exit;
    }

    qemu_co_mutex_lock(&bs->reqs_lock);
    /*获取当前最新的write版本号*/
    current_gen = atomic_read(&bs->write_gen);

    /* BlockDriverState::active_flush_req为true标识当前还有正在进行中的flush请求
     * 尚未完成，将当前的协程添加到@BlockDriverState::flush_queue中
     */
    /* Wait until any previous flushes are completed */
    while (bs->active_flush_req) {
        qemu_co_queue_wait(&bs->flush_queue, &bs->reqs_lock);
    }

    /* Flushes reach this point in nondecreasing current_gen order.  */
    /*设置当前正在处理flush请求*/
    bs->active_flush_req = true;
    qemu_co_mutex_unlock(&bs->reqs_lock);

    /* Write back all layers by calling one driver function */
    /*如果提供了.bdrv_co_flush接口函数，则会确保flush任何一层的数据，直到数据到达物理硬盘*/
    /*对于qcow2来说，没有提供bdrv_co_flush接口函数*/
    /*对于file来说，也没有提供bdrv_co_flush接口函数*/
    if (bs->drv->bdrv_co_flush) {
        ret = bs->drv->bdrv_co_flush(bs);
        goto out;
    }

    /* Write back cached data to the OS even with cache=unsafe */
    /*如果没有提供.bdrv_co_flush接口函数，则尝试先flush到host os*/
    /*写入到host os，对于qcow2来说.bdrv_co_flush_to_os = qcow2_co_flush_to_os*/
    /*对于file来说，没有提供bdrv_co_flush_to_os接口函数*/
    BLKDBG_EVENT(bs->file, BLKDBG_FLUSH_TO_OS);
    if (bs->drv->bdrv_co_flush_to_os) {
        ret = bs->drv->bdrv_co_flush_to_os(bs);
        if (ret < 0) {
            goto out;
        }
    }

    /* But don't actually force it to the disk with cache=unsafe */
    /* 如果设置了BDRV_O_NO_FLUSH标识，则对于处于host os中的数据不再执行flush，
     * 否则跳转到flush_parent处
     */
    if (bs->open_flags & BDRV_O_NO_FLUSH) {
        goto flush_parent;
    }

    /* Check if we really need to flush anything */
    /*如果当前flush的版本号跟write的版本号相同，则表明无需执行flush了，跳转到flush_parent处*/
    if (bs->flushed_gen == current_gen) {
        goto flush_parent;
    }

    /*在尝试从host os flush到disk*/
    /*对于qcow2来说，既没有提供.bdrv_co_flush_to_disk，也没有提供.bdrv_aio_flush*/
    /*对于file来说，提供了.bdrv_aio_flush = raw_aio_flush*/
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
    /*如果存在@bs->file，则针对@bs->file执行bdrv_co_flush*/
    /*对于qcow2来说，存在@bs->file，对于file来说则不存在*/
    ret = bs->file ? bdrv_co_flush(bs->file->bs) : 0;
    
out:
    /* Notify any pending flushes that we have completed */
    if (ret == 0) {
        /*更新flush版本号到当前的write版本号*/
        bs->flushed_gen = current_gen;
    }

    qemu_co_mutex_lock(&bs->reqs_lock);
    /*标记处理flush请求完毕*/
    bs->active_flush_req = false;
    /* Return value is ignored - it's ok if wait queue is empty */
    /*调度等待在@BlockDriverState::flush_queue中的下一个协程*/
    qemu_co_queue_next(&bs->flush_queue);
    qemu_co_mutex_unlock(&bs->reqs_lock);

early_exit:
    bdrv_dec_in_flight(bs);
    return ret;
}
```

## qemu block allocation status中各标识含义

```
/*
 * Allocation status flags for bdrv_get_block_status() and friends.
 *
 * Public flags:
 * BDRV_BLOCK_DATA: allocation for data at offset is tied to this layer
 * BDRV_BLOCK_ZERO: offset reads as zero
 * BDRV_BLOCK_OFFSET_VALID: an associated offset exists for accessing raw data
 * BDRV_BLOCK_ALLOCATED: the content of the block is determined by this
 *                       layer (short for DATA || ZERO), set by block layer
 * BDRV_BLOCK_EOF: the returned pnum covers through end of file for this layer
 *
 * Internal flag:
 * BDRV_BLOCK_RAW: for use by passthrough drivers, such as raw, to request
 *                 that the block layer recompute the answer from the returned
 *                 BDS; must be accompanied by just BDRV_BLOCK_OFFSET_VALID.
 *
 * If BDRV_BLOCK_OFFSET_VALID is set, bits 9-62 (BDRV_BLOCK_OFFSET_MASK)
 * represent the offset in the returned BDS that is allocated for the
 * corresponding raw data; however, whether that offset actually contains
 * data also depends on BDRV_BLOCK_DATA and BDRV_BLOCK_ZERO, as follows:
 *
 * DATA ZERO OFFSET_VALID
 *  t    t        t       sectors read as zero, returned file is zero at offset
 *  t    f        t       sectors read as valid from file at offset
 *  f    t        t       sectors preallocated, read as zero, returned file not
 *                        necessarily zero at offset
 *  f    f        t       sectors preallocated but read from backing_hd,
 *                        returned file contains garbage at offset
 *  t    t        f       sectors preallocated, read as zero, unknown offset
 *  t    f        f       sectors read from unknown file or offset
 *  f    t        f       not allocated or unknown offset, read as zero
 *  f    f        f       not allocated or unknown offset, read from backing_hd
 */
#define BDRV_BLOCK_DATA         0x01
#define BDRV_BLOCK_ZERO         0x02
#define BDRV_BLOCK_OFFSET_VALID 0x04
#define BDRV_BLOCK_RAW          0x08
#define BDRV_BLOCK_ALLOCATED    0x10
#define BDRV_BLOCK_EOF          0x20
#define BDRV_BLOCK_OFFSET_MASK  BDRV_SECTOR_MASK
```

## qemu中copy on read的理解
[block: generic copy-on-read](https://lists.gnu.org/archive/html/qemu-devel/2011-11/msg02842.html)


## 如何理解qemu block中的format和protocol
[block: format vs. protocol, and how they stack](https://lists.gnu.org/archive/html/qemu-devel/2010-06/msg02602.html)


## qemu中读写是如何保证读写相关和写写相关的？
qemu中通过serialising机制确保读写相关和写写相关，主要借助于mark_request_serialising
和wait_serialising_requests这两个函数，从代码角度来说，如下：
对于写操作：

```
int coroutine_fn bdrv_co_pwritev(BdrvChild *child,
    int64_t offset, unsigned int bytes, QEMUIOVector *qiov,
    BdrvRequestFlags flags)
{
    BlockDriverState *bs = child->bs;
    BdrvTrackedRequest req;
    /*写操作过程中用到的对齐是bs->bl.request_alignment，跟copy on read中的对齐不一样*/
    uint64_t align = bs->bl.request_alignment;
    uint8_t *head_buf = NULL;
    uint8_t *tail_buf = NULL;
    QEMUIOVector local_qiov;
    bool use_local_qiov = false;
    int ret;

    bdrv_inc_in_flight(bs);
    /*将请求添加到BlockDriverState::tracked_requests链表，该链表存放正在处理的请求*/
    tracked_request_begin(&req, bs, offset, bytes, BDRV_TRACKED_WRITE);

    /*头部对齐处理*/
    if (offset & (align - 1)) {
        QEMUIOVector head_qiov;
        struct iovec head_iov;

        /* 标记该请求需要serialising，并设置serialising区间为[req->overlap_offset,
         * req->overlap_offset + req->overlap_bytes)
         */
        mark_request_serialising(&req, align);
        /* 等待所有存在于BlockDriverState::tracked_requests链表中，与该请求存在交叠的
         * 先于它进入于BlockDriverState::tracked_requests链表的请求执行完毕，当前协程
         * 才会继续执行，所以从wait_serialising_requests返回的时候一定没有先于它的且
         * 与之存在数据相关的请求还在BlockDriverState::tracked_requests链表中了，但是
         * BlockDriverState::tracked_requests链表中可能存在与它存在数据相关性的后于它
         * 到达的请求
         */
        wait_serialising_requests(&req);

        head_buf = qemu_blockalign(bs, align);
        head_iov = (struct iovec) {
            .iov_base   = head_buf,
            .iov_len    = align,
        };
        qemu_iovec_init_external(&head_qiov, &head_iov, 1);

        /*在bdrv_aligned_preadv中会再次调用wait_serialising_requests*/
        ret = bdrv_aligned_preadv(child, &req, offset & ~(align - 1), align,
                                  align, &head_qiov, 0);
        if (ret < 0) {
            goto fail;
        }
        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_AFTER_HEAD);

        qemu_iovec_init(&local_qiov, qiov->niov + 2);
        qemu_iovec_add(&local_qiov, head_buf, offset & (align - 1));
        qemu_iovec_concat(&local_qiov, qiov, 0, qiov->size);
        use_local_qiov = true;

        bytes += offset & (align - 1);
        offset = offset & ~(align - 1);

        /* We have read the tail already if the request is smaller
         * than one aligned block.
         */
        if (bytes < align) {
            qemu_iovec_add(&local_qiov, head_buf + bytes, align - bytes);
            bytes = align;
        }
    }

    if ((offset + bytes) & (align - 1)) {
        QEMUIOVector tail_qiov;
        struct iovec tail_iov;
        size_t tail_bytes;
        bool waited;

        mark_request_serialising(&req, align);
        waited = wait_serialising_requests(&req);
        assert(!waited || !use_local_qiov);

        tail_buf = qemu_blockalign(bs, align);
        tail_iov = (struct iovec) {
            .iov_base   = tail_buf,
            .iov_len    = align,
        };
        qemu_iovec_init_external(&tail_qiov, &tail_iov, 1);

        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_TAIL);
        ret = bdrv_aligned_preadv(child, &req, (offset + bytes) & ~(align - 1),
                                  align, align, &tail_qiov, 0);
        if (ret < 0) {
            goto fail;
        }
        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_AFTER_TAIL);

        if (!use_local_qiov) {
            qemu_iovec_init(&local_qiov, qiov->niov + 1);
            qemu_iovec_concat(&local_qiov, qiov, 0, qiov->size);
            use_local_qiov = true;
        }

        tail_bytes = (offset + bytes) & (align - 1);
        qemu_iovec_add(&local_qiov, tail_buf + tail_bytes, align - tail_bytes);

        bytes = ROUND_UP(bytes, align);
    }

    ret = bdrv_aligned_pwritev(child, &req, offset, bytes, align,
                               use_local_qiov ? &local_qiov : qiov,
                               flags);

fail:

    if (use_local_qiov) {
        qemu_iovec_destroy(&local_qiov);
    }
    qemu_vfree(head_buf);
    qemu_vfree(tail_buf);
out:
    tracked_request_end(&req);
    bdrv_dec_in_flight(bs);
    return ret;
}

static void mark_request_serialising(BdrvTrackedRequest *req, uint64_t align)
{
    int64_t overlap_offset = req->offset & ~(align - 1);
    unsigned int overlap_bytes = ROUND_UP(req->offset + req->bytes, align)
                               - overlap_offset;

    /*设置serialising标识，增加serialising_in_flight计数*/
    if (!req->serialising) {
        atomic_inc(&req->bs->serialising_in_flight);
        req->serialising = true;
    }

    /*设置overlap检查区间*/
    req->overlap_offset = MIN(req->overlap_offset, overlap_offset);
    req->overlap_bytes = MAX(req->overlap_bytes, overlap_bytes);
}

static bool coroutine_fn wait_serialising_requests(BdrvTrackedRequest *self)
{
    BlockDriverState *bs = self->bs;
    BdrvTrackedRequest *req;
    bool retry;
    bool waited = false;

    /*当前没有处于serialising状态的请求，则无需等待*/
    if (!atomic_read(&bs->serialising_in_flight)) {
        return false;
    }

    do {
        retry = false;
        qemu_co_mutex_lock(&bs->reqs_lock);
        /*遍历BlockDriverState::tracked_requests链表，检查相关性，确保serialisation*/
        QLIST_FOREACH(req, &bs->tracked_requests, list) {
            /*@req和@self至少一个设置了serialising标识，才会执行相关性检查*/
            if (req == self || (!req->serialising && !self->serialising)) {
                continue;
            }
            
            /*存在交叠，即存在相关性*/
            if (tracked_request_overlaps(req, self->overlap_offset,
                                         self->overlap_bytes))
            {
                /* Hitting this means there was a reentrant request, for
                 * example, a block driver issuing nested requests.  This must
                 * never happen since it means deadlock.
                 */
                assert(qemu_coroutine_self() != req->co);

                /* If the request is already (indirectly) waiting for us, or
                 * will wait for us as soon as it wakes up, then just go on
                 * (instead of producing a deadlock in the former case). */
                /* 如果@req当前正在等待其它请求（req->waiting_for不为空），那么它所在的
                 * 协程当前一定处于挂起状态，等待它所等待的请求所在的协程唤醒之，如果再
                 * 让当前的请求@self去等待@req，那么当@req所在的协程被唤醒的时候，就会
                 * 重新遍历BlockDriverState::tracked_requests链表，检查与该链表中的请求
                 * 的相关性，在遇到@self这个请求的时候，@req又会等待在@self上，这样就会
                 * 产生死锁。
                 * 
                 * 为了避免死锁的这种情况的发生，当@self和@req检测到相关性的时候，只有当
                 * @req没有等待其它请求，才会设置@self->waiting_for = @req，否则@self
                 * 请求略过@req请求，检查BlockDriverState::tracked_requests链表中后续
                 * 请求的相关性，这么做如何能确保@self和@req的相关性呢？假如@req正在等待
                 * 的请求是@blocker，即@req->waiting_for = @blocker，那么可以断言@blocker
                 * 一定尚未完成，因为如果它完成了，则@req所在的协程会被唤醒，@req->waiting_for
                 * 会被设置为NULL。
                 *
                 * 假设@blocker没有等待其它请求，那么@self和@blocker之间可能存在相关性，
                 * 也可能不存在相关性：
                 * 假如存在相关性，则@self会等待在@blocker上，添加到@blocker的等待队列中，
                 * 因为@slef所在的协程是后于@req所在的协程添加到@blocker的等待队列的，在
                 * @blocker执行完毕唤醒的时候，会先唤醒@req对应的协程，后唤醒@self对应的
                 * 协程，从而保证@req在@self之前完成；
                 * 假如@self和@blocker之间不存在相关性，且@slef和其它请求之间也不存在相关
                 * 性，那么@self就会通过wait_serialising_requests的检查，那么岂不是@slef
                 * 在@req之前执行？？？？
                 * 
                 * ！！！！！！关于数据相关性的分析，在这里卡壳了，后续继续！！！！！！
                 */
                if (!req->waiting_for) {
                    self->waiting_for = req;
                    qemu_co_queue_wait(&req->wait_queue, &bs->reqs_lock);
                    self->waiting_for = NULL;
                    retry = true;
                    waited = true;
                    break;
                }
            }
        }
        qemu_co_mutex_unlock(&bs->reqs_lock);
    } while (retry);

    return waited;
}

```

# qcow2
## qcow2相关参考资料
[The QCOW2 Image Format](https://people.gnome.org/~markmc/qcow-image-format.html)
[Qcow3 features](https://wiki.qemu.org/Features/Qcow3)

## qcow2 QCow2ClusterType详解
先看下qcow2.h中关于QCow2ClusterType的定义：
```
typedef enum QCow2ClusterType {
    QCOW2_CLUSTER_UNALLOCATED,
    QCOW2_CLUSTER_ZERO_PLAIN,
    QCOW2_CLUSTER_ZERO_ALLOC,
    QCOW2_CLUSTER_NORMAL,
    QCOW2_CLUSTER_COMPRESSED,
} QCow2ClusterType;
```
qcow2_get_cluster_offset用于获取虚拟磁盘中给定offset在镜像文件中的偏移和cluster的类型，
从该函数的实现中可以看出各cluster type的含义：
```
/*
 * get_cluster_offset
 *
 * For a given offset of the virtual disk, find the cluster type and offset in
 * the qcow2 file. The offset is stored in *cluster_offset.
 *
 * On entry, *bytes is the maximum number of contiguous bytes starting at
 * offset that we are interested in.
 *
 * On exit, *bytes is the number of bytes starting at offset that have the same
 * cluster type and (if applicable) are stored contiguously in the image file.
 * Compressed clusters are always returned one by one.
 *
 * Returns the cluster type (QCOW2_CLUSTER_*) on success, -errno in error
 * cases.
 *
 * 关于qcow2_get_cluster_offset的参数和作用，上述注释中已经说得很明确了，不再赘述。
 */
int qcow2_get_cluster_offset(BlockDriverState *bs, uint64_t offset,
                             unsigned int *bytes, uint64_t *cluster_offset)
{
    BDRVQcow2State *s = bs->opaque;
    unsigned int l2_index;
    uint64_t l1_index, l2_offset, *l2_table;
    int l1_bits, c;
    unsigned int offset_in_cluster;
    uint64_t bytes_available, bytes_needed, nb_clusters;
    QCow2ClusterType type;
    int ret;

    /*用于计算至多查找多少个跟@offset对应的cluster具有相同类型的cluster*/
    offset_in_cluster = offset_into_cluster(s, offset);
    bytes_needed = (uint64_t) *bytes + offset_in_cluster;
    l1_bits = s->l2_bits + s->cluster_bits;
    /* compute how many bytes there are between the start of the cluster
     * containing offset and the end of the l1 entry */
    bytes_available = (1ULL << l1_bits) - (offset & ((1ULL << l1_bits) - 1))
                    + offset_in_cluster;

    if (bytes_needed > bytes_available) {
        bytes_needed = bytes_available;
    }

    /*计算@offset对应的cluster在L1 table中的索引号*/
    l1_index = offset >> l1_bits;
    /*如果超过了L1 table的大小，表明尚未分配*/
    if (l1_index >= s->l1_size) {
        type = QCOW2_CLUSTER_UNALLOCATED;
        goto out;
    }

    /*@offset对应的cluster在L2 table 地址*/
    l2_offset = s->l1_table[l1_index] & L1E_OFFSET_MASK;
    /*L2 table地址为0，表明该cluster尚未分配*/
    if (!l2_offset) {
        type = QCOW2_CLUSTER_UNALLOCATED;
        goto out;
    }

    /*加载@offset对应的cluster对应的L2 table*/
    ret = l2_load(bs, l2_offset, &l2_table);

    /*计算@offset对应的cluster在L2 table中的索引号*/
    l2_index = offset_to_l2_index(s, offset);
    /*计算@offset对应的cluster在镜像文件中的地址*/
    *cluster_offset = be64_to_cpu(l2_table[l2_index]);

    /*至多从@offset对应的cluster开始查找@nb_clusters个cluster*/
    nb_clusters = size_to_clusters(s, bytes_needed);

    /*获取@offset所在的cluster(在镜像文件中地址为@*cluster_offset)的类型*/
    type = qcow2_get_cluster_type(*cluster_offset);
    switch (type) {
    case QCOW2_CLUSTER_COMPRESSED:
        /*对于压缩类型的cluster，一次只处理一个*/
        c = 1;
        *cluster_offset &= L2E_COMPRESSED_OFFSET_SIZE_MASK;
        break;
    case QCOW2_CLUSTER_ZERO_PLAIN:
    case QCOW2_CLUSTER_UNALLOCATED:
        /* how many empty clusters ? */
        c = count_contiguous_clusters_unallocated(nb_clusters,
                                                  &l2_table[l2_index], type);
        /* 从这里来看，QCOW2_CLUSTER_ZERO_PLAIN和QCOW2_CLUSTER_UNALLOCATED类型的cluster
         * 都是未真实分配空间的(因为返回的@*cluster_offset = 0)
         */                                                 
        *cluster_offset = 0;
        break;
    case QCOW2_CLUSTER_ZERO_ALLOC:
    case QCOW2_CLUSTER_NORMAL:
        /* how many allocated clusters ? */
        c = count_contiguous_clusters(nb_clusters, s->cluster_size,
                                      &l2_table[l2_index], QCOW_OFLAG_ZERO);
        /* 从这里来看，QCOW2_CLUSTER_ZERO_ALLOC和QCOW2_CLUSTER_NORMAL类型的cluster
         * 都是实实在在分配空间了的
         */
        *cluster_offset &= L2E_OFFSET_MASK;
        break;
    default:
        abort();
    }

    qcow2_cache_put(bs, s->l2_table_cache, (void**) &l2_table);
    bytes_available = (int64_t)c * s->cluster_size;

out:
    if (bytes_available > bytes_needed) {
        bytes_available = bytes_needed;
    }

    /* bytes_available <= bytes_needed <= *bytes + offset_in_cluster;
     * subtracting offset_in_cluster will therefore definitely yield something
     * not exceeding UINT_MAX */
    assert(bytes_available - offset_in_cluster <= UINT_MAX);
    *bytes = bytes_available - offset_in_cluster;

    return type;

fail:
    qcow2_cache_put(bs, s->l2_table_cache, (void **)&l2_table);
    return ret;
}

static inline QCow2ClusterType qcow2_get_cluster_type(uint64_t l2_entry)
{
    if (l2_entry & QCOW_OFLAG_COMPRESSED) {
        /*压缩类型的cluster*/
        return QCOW2_CLUSTER_COMPRESSED;
    } else if (l2_entry & QCOW_OFLAG_ZERO) {
        /*在读取设置了QCOW_OFLAG_ZERO标识的cluster时，全部填充0*/
        if (l2_entry & L2E_OFFSET_MASK) {
            /*如果该cluster真实分配了空间，则类型为QCOW2_CLUSTER_ZERO_ALLOC*/
            return QCOW2_CLUSTER_ZERO_ALLOC;
        }
        
        /*该cluster未分配空间，那么类型就是QCOW2_CLUSTER_ZERO_PLAIN*/
        return QCOW2_CLUSTER_ZERO_PLAIN;
    } else if (!(l2_entry & L2E_OFFSET_MASK)) {
        /*为真实分配空间，类型就是QCOW2_CLUSTER_UNALLOCATED*/
        return QCOW2_CLUSTER_UNALLOCATED;
    } else {
        /*否则就是QCOW2_CLUSTER_NORMAL*/
        return QCOW2_CLUSTER_NORMAL;
    }
}
```
从上述代码分析中可以看到：

**QCOW2_CLUSTER_UNALLOCATED：**

尚未在镜像文件中为之分配空间，在L1、L2 table中找不到对应的cluster，
或者对应的cluster在L2 table中的地址信息为0

**QCOW2_CLUSTER_ZERO_PLAIN：**

该cluster存在（或者说其中存在有效数据），但是没有为之分配空间（因为该
cluster中的数据在读取的时候直接返回全0），在L2 table中可以找到该cluster
的信息，且该cluster的地址信息中设置了QCOW_OFLAG_ZERO标识

**QCOW2_CLUSTER_ZERO_ALLOC：**

该cluster存在，且为之分配了空间，在L2 table中可以找到该cluster的信息，
且该cluster的地址信息中设置了QCOW_OFLAG_ZERO标识，表示在读取的时候直接
返回全0

**QCOW2_CLUSTER_NORMAL：**

该cluster存在，且为之分配了空间，但cluster的地址中没有设置QCOW_OFLAG_ZERO
或者QCOW_OFLAG_COMPRESSED标识

**QCOW2_CLUSTER_COMPRESSED：**

该cluster存在，且为之分配了空间，且cluster的地址中设置了QCOW_OFLAG_COMPRESSED标识


## qcow2 QCOW_OFLAG_* 标识详解
**QCOW_OFLAG_COMPRESSED:**

该cluster是压缩类型的cluster

**QCOW_OFLAG_ZERO:**

该cluster在读取的时候返回全0

**QCOW_OFLAG_COPIED:**

该cluster的引用计数是1，可以直接写入，无需执行COW
跟踪代码发现，在所有新分配的cluster添加到L1/L2 table中的时候，都会在其地址信息中
设置QCOW_OFLAG_COPIED标识，标识其引用计数为1，只有在qcow2_update_snapshot_refcount
中有可能会取消该标识。


