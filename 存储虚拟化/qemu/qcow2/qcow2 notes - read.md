# 读镜像文件

```
/*和写流程一样，也从blk_co_preadv开始分析*/
```

```
int coroutine_fn blk_co_preadv(BlockBackend *blk, int64_t offset,
                               unsigned int bytes, QEMUIOVector *qiov,
                               BdrvRequestFlags flags)
{
    int ret;
    BlockDriverState *bs = blk_bs(blk);

    trace_blk_co_preadv(blk, bs, offset, bytes, flags);

    ret = blk_check_byte_request(blk, offset, bytes);
    if (ret < 0) {
        return ret;
    }

    bdrv_inc_in_flight(bs);

    /* throttling disk I/O */
    /*限流相关，在此不关注*/
    if (blk->public.throttle_group_member.throttle_state) {
        throttle_group_co_io_limits_intercept(&blk->public.throttle_group_member,
                bytes, false);
    }

    /*这里传入的第一个参数BDrvChild是@blk->root*/
    ret = bdrv_co_preadv(blk->root, offset, bytes, qiov, flags);
    bdrv_dec_in_flight(bs);
    return ret;
}
```

```
/*
 * Handle a read request in coroutine context
 */
int coroutine_fn bdrv_co_preadv(BdrvChild *child,
    int64_t offset, unsigned int bytes, QEMUIOVector *qiov,
    BdrvRequestFlags flags)
{
    BlockDriverState *bs = child->bs;
    BlockDriver *drv = bs->drv;
    BdrvTrackedRequest req;

    /*关于对齐要求，请参考“qcow2 notes - open”*/
    uint64_t align = bs->bl.request_alignment;
    uint8_t *head_buf = NULL;
    uint8_t *tail_buf = NULL;
    QEMUIOVector local_qiov;
    bool use_local_qiov = false;
    int ret;

    bdrv_inc_in_flight(bs);

    /* @bs->copy_on_read:如果该值不为0，则表示在读取的时候如果数据在基础镜像文件中，
     * 则将基础镜像文件中的数据拷贝到本地镜像文件中
     */
    /* Don't do copy-on-read if we read data before write operation */
    if (atomic_read(&bs->copy_on_read) && !(flags & BDRV_REQ_NO_SERIALISING)) {
        flags |= BDRV_REQ_COPY_ON_READ;
    }

    /* Align read if necessary by padding qiov */
    if (offset & (align - 1)) {
        /*如果@offset不是对齐到@align的，则做对齐处理，分配@head_buf并链接到@qiov中*/
        head_buf = qemu_blockalign(bs, align);
        qemu_iovec_init(&local_qiov, qiov->niov + 2);
        qemu_iovec_add(&local_qiov, head_buf, offset & (align - 1));
        qemu_iovec_concat(&local_qiov, qiov, 0, qiov->size);
        use_local_qiov = true;

        /*因为@offset会向下对其到@align，所以也要相应的扩展@bytes*/
        bytes += offset & (align - 1);
        /*@offset向下对其到@align*/
        offset = offset & ~(align - 1);
    }

    if ((offset + bytes) & (align - 1)) {
        /*如果@offset + bytes不是对齐到@align的，则做对齐处理， 分配@tail_buf并链接到@qiov中*/
        if (!use_local_qiov) {
            qemu_iovec_init(&local_qiov, qiov->niov + 1);
            qemu_iovec_concat(&local_qiov, qiov, 0, qiov->size);
            use_local_qiov = true;
        }
        tail_buf = qemu_blockalign(bs, align);
        qemu_iovec_add(&local_qiov, tail_buf,
                       align - ((offset + bytes) & (align - 1)));

        /*@bytes向上扩展以向上对齐到@align*/
        bytes = ROUND_UP(bytes, align);
    }

    /*将该请求添加到tracked request 链表*/
    tracked_request_begin(&req, bs, offset, bytes, BDRV_TRACKED_READ);
    /*处理对齐的读请求，这里的@offset和@bytes一定是对齐到@align的*/
    ret = bdrv_aligned_preadv(child, &req, offset, bytes, align,
                              use_local_qiov ? &local_qiov : qiov,
                              flags);
    tracked_request_end(&req);
    bdrv_dec_in_flight(bs);

    if (use_local_qiov) {
        qemu_iovec_destroy(&local_qiov);
        qemu_vfree(head_buf);
        qemu_vfree(tail_buf);
    }

    return ret;
}
```

```
/**
 * Add an active request to the tracked requests list
 */
static void tracked_request_begin(BdrvTrackedRequest *req,
                                  BlockDriverState *bs,
                                  int64_t offset,
                                  unsigned int bytes,
                                  enum BdrvTrackedRequestType type)
{
    *req = (BdrvTrackedRequest){
        .bs = bs,
        .offset         = offset,
        .bytes          = bytes,
        .type           = type,
        .co             = qemu_coroutine_self(),
        .serialising    = false,
        .overlap_offset = offset,
        .overlap_bytes  = bytes,
    };

    /*初始化关于该请求的等待队列，用于存放被当前请求阻塞的协程*/
    qemu_co_queue_init(&req->wait_queue);
    /*添加到@bs->tracked_requests链表*/
    qemu_co_mutex_lock(&bs->reqs_lock);
    QLIST_INSERT_HEAD(&bs->tracked_requests, req, list);
    qemu_co_mutex_unlock(&bs->reqs_lock);
}
```

```
/**
 * Remove an active request from the tracked requests list
 *
 * This function should be called when a tracked request is completing.
 */
static void tracked_request_end(BdrvTrackedRequest *req)
{
    if (req->serialising) {
        atomic_dec(&req->bs->serialising_in_flight);
    }

    /*从@bs->tracked_requests链表中删除，并唤醒该完成的请求的等待队列中的协程*/
    qemu_co_mutex_lock(&req->bs->reqs_lock);
    QLIST_REMOVE(req, list);
    /*唤醒所有被@req阻塞的请求*/
    qemu_co_queue_restart_all(&req->wait_queue);
    qemu_co_mutex_unlock(&req->bs->reqs_lock);
}
```

```
/*
 * Forwards an already correctly aligned request to the BlockDriver. This
 * handles copy on read, zeroing after EOF, and fragmentation of large
 * reads; any other features must be implemented by the caller.
 */
static int coroutine_fn bdrv_aligned_preadv(BdrvChild *child,
    BdrvTrackedRequest *req, int64_t offset, unsigned int bytes,
    int64_t align, QEMUIOVector *qiov, int flags)
{
    BlockDriverState *bs = child->bs;
    int64_t total_bytes, max_bytes;
    int ret = 0;
    uint64_t bytes_remaining = bytes;
    int max_transfer;

    assert(is_power_of_2(align));
    assert((offset & (align - 1)) == 0);
    assert((bytes & (align - 1)) == 0);
    assert(!qiov || bytes == qiov->size);
    assert((bs->open_flags & BDRV_O_NO_IO) == 0);
    max_transfer = QEMU_ALIGN_DOWN(MIN_NON_ZERO(bs->bl.max_transfer, INT_MAX),
                                   align);

    /* TODO: We would need a per-BDS .supported_read_flags and
     * potential fallback support, if we ever implement any read flags
     * to pass through to drivers.  For now, there aren't any
     * passthrough flags.  */
    assert(!(flags & ~(BDRV_REQ_NO_SERIALISING | BDRV_REQ_COPY_ON_READ)));

    /* Handle Copy on Read and associated serialisation */
    if (flags & BDRV_REQ_COPY_ON_READ) {
        /* If we touch the same cluster it counts as an overlap.  This
         * guarantees that allocating writes will be serialized and not race
         * with each other for the same cluster.  For example, in copy-on-read
         * it ensures that the CoR read and write operations are atomic and
         * guest writes cannot interleave between them. */
        /* 如果设置了BDRV_REQ_COPY_ON_READ标识，则标记@req需要serialising，
         * 并添加到BlockDriverState::serialising_in_flight链表中
         */
        mark_request_serialising(req, bdrv_get_cluster_size(bs));
    }

    if (!(flags & BDRV_REQ_NO_SERIALISING)) {
        /*等待与之存在交集的请求处理完毕*/
        wait_serialising_requests(req);
    }

    /*至此，当前请求可以执行了*/
    if (flags & BDRV_REQ_COPY_ON_READ) {
        /* TODO: Simplify further once bdrv_is_allocated no longer
         * requires sector alignment */
        int64_t start = QEMU_ALIGN_DOWN(offset, BDRV_SECTOR_SIZE);
        int64_t end = QEMU_ALIGN_UP(offset + bytes, BDRV_SECTOR_SIZE);
        int64_t pnum;

        ret = bdrv_is_allocated(bs, start, end - start, &pnum);
        if (ret < 0) {
            goto out;
        }

        /*需要执行copy on read*/
        if (!ret || pnum != end - start) {
            /* 最终也是调用bdrv_driver_preadv -> qcow2_co_preadv去执行读取，
             * 在qcow2_co_preadv中如果发现cluster是QCOW2_CLUSTER_UNALLOCATED
             * 类型的，则执行qcow2_backing_read1去基础镜像文件中读取
             */
            ret = bdrv_co_do_copy_on_readv(child, offset, bytes, qiov);
            goto out;
        }
    }

    /* Forward the request to the BlockDriver, possibly fragmenting it */
    total_bytes = bdrv_getlength(bs);
    if (total_bytes < 0) {
        ret = total_bytes;
        goto out;
    }

    max_bytes = ROUND_UP(MAX(0, total_bytes - offset), align);
    /*如果不超过@max_transfer，则直接调用bdrv_driver_preadv*/
    if (bytes <= max_bytes && bytes <= max_transfer) {
        ret = bdrv_driver_preadv(bs, offset, bytes, qiov, 0);
        goto out;
    }

    /*否则，循环调用bdrv_driver_preadv，每次至多读取@max_transfer字节*/
    while (bytes_remaining) {
        int num;

        if (max_bytes) {
            QEMUIOVector local_qiov;

            num = MIN(bytes_remaining, MIN(max_bytes, max_transfer));
            assert(num);
            qemu_iovec_init(&local_qiov, qiov->niov);
            qemu_iovec_concat(&local_qiov, qiov, bytes - bytes_remaining, num);

            ret = bdrv_driver_preadv(bs, offset + bytes - bytes_remaining,
                                     num, &local_qiov, 0);
            max_bytes -= num;
            qemu_iovec_destroy(&local_qiov);
        } else {
            num = bytes_remaining;
            ret = qemu_iovec_memset(qiov, bytes - bytes_remaining, 0,
                                    bytes_remaining);
        }
        if (ret < 0) {
            goto out;
        }
        bytes_remaining -= num;
    }

out:
    return ret < 0 ? ret : 0;
}
```

```
static bool coroutine_fn wait_serialising_requests(BdrvTrackedRequest *self)
{
    BlockDriverState *bs = self->bs;
    BdrvTrackedRequest *req;
    bool retry;
    bool waited = false;

    /*没有尚未完成的处于serialising状态的请求，则无需等待*/
    if (!atomic_read(&bs->serialising_in_flight)) {
        return false;
    }

    do {
        retry = false;
        qemu_co_mutex_lock(&bs->reqs_lock);
        /*BlockDriverState::tracked_requests中包含所有尚未完成的请求，逐一处理*/
        QLIST_FOREACH(req, &bs->tracked_requests, list) {
            /* 如果待处理的@req不是@self自身，且@req和@self中至少一个指定了
             * serialising标识，则需要处理该@req
             */
            if (req == self || (!req->serialising && !self->serialising)) {
                continue;
            }
            
            /* 检查[req->overlap_offset, req->overlap_offset + req->overlap_bytes)
             * 所代表的区域和[self->overlap_offset, self->overlap_offset + req->overlap_bytes)
             * 所代表的区域之间是否存在交集
             */
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
                if (!req->waiting_for) {
                    /* 如果@req当前没有等待其它请求，则设置@self等待@req，并添加到@req->wait_queue中，
                     * @self所在的协程（即当前协程）退出，直到@req执行完毕唤醒@req->wait_queue中的协
                     * 程，包括@self所在的协程
                     */
                    self->waiting_for = req;
                    qemu_co_queue_wait(&req->wait_queue, &bs->reqs_lock);
                    /*当前协程被唤醒，继续重试，重新遍历BlockDriverState::tracked_requests*/
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

```
int coroutine_fn bdrv_is_allocated(BlockDriverState *bs, int64_t offset,
                                   int64_t bytes, int64_t *pnum)
{
    BlockDriverState *file;
    /*@sector_num表示其实sector号，@nb_sectors表示总共多少个sector*/
    int64_t sector_num = offset >> BDRV_SECTOR_BITS;
    int nb_sectors = bytes >> BDRV_SECTOR_BITS;
    int64_t ret;
    int psectors;

    assert(QEMU_IS_ALIGNED(offset, BDRV_SECTOR_SIZE));
    assert(QEMU_IS_ALIGNED(bytes, BDRV_SECTOR_SIZE) && bytes < INT_MAX);
    ret = bdrv_get_block_status(bs, sector_num, nb_sectors, &psectors,
                                &file);
    if (ret < 0) {
        return ret;
    }
    if (pnum) {
        *pnum = psectors * BDRV_SECTOR_SIZE;
    }
    return !!(ret & BDRV_BLOCK_ALLOCATED);
}
```

```
int64_t bdrv_get_block_status(BlockDriverState *bs,
                              int64_t sector_num,
                              int nb_sectors, int *pnum,
                              BlockDriverState **file)
{
    /* 这里传递了两个BlockDriverState，一个是@bs，表示本地镜像文件，另一个是
     * @backing_bs(bs)，表示基础镜像文件
     */
    return bdrv_get_block_status_above(bs, backing_bs(bs),
                                       sector_num, nb_sectors, pnum, file);
}
```

```
int64_t bdrv_get_block_status_above(BlockDriverState *bs,
                                    BlockDriverState *base,
                                    int64_t sector_num,
                                    int nb_sectors, int *pnum,
                                    BlockDriverState **file)
{
    Coroutine *co;
    BdrvCoGetBlockStatusData data = {
        /*本地镜像文件*/
        .bs = bs,
        /*基础镜像文件*/
        .base = base,
        .file = file,
        .sector_num = sector_num,
        .nb_sectors = nb_sectors,
        .pnum = pnum,
        .done = false,
    };

    /*无论如何最终都会调用到bdrv_get_block_status_above_co_entry*/
    if (qemu_in_coroutine()) {
        /* Fast-path if already in coroutine context */
        bdrv_get_block_status_above_co_entry(&data);
    } else {
        co = qemu_coroutine_create(bdrv_get_block_status_above_co_entry,
                                   &data);
        bdrv_coroutine_enter(bs, co);
        BDRV_POLL_WHILE(bs, !data.done);
    }
    return data.ret;
}
```

```
static void coroutine_fn bdrv_get_block_status_above_co_entry(void *opaque)
{
    BdrvCoGetBlockStatusData *data = opaque;

    data->ret = bdrv_co_get_block_status_above(data->bs, data->base,
                                               data->sector_num,
                                               data->nb_sectors,
                                               data->pnum,
                                               data->file);
    data->done = true;
}
```

```
static int64_t coroutine_fn bdrv_co_get_block_status_above(BlockDriverState *bs,
        BlockDriverState *base,
        int64_t sector_num,
        int nb_sectors,
        int *pnum,
        BlockDriverState **file)
{
    BlockDriverState *p;
    int64_t ret = 0;
    bool first = true;

    /*@bs表示本地镜像文件，@base表示基础镜像文件*/
    assert(bs != base);
    for (p = bs; p != base; p = backing_bs(p)) {
        ret = bdrv_co_get_block_status(p, sector_num, nb_sectors, pnum, file);
        if (ret < 0) {
            break;
        }
        if (ret & BDRV_BLOCK_ZERO && ret & BDRV_BLOCK_EOF && !first) {
            /*
             * Reading beyond the end of the file continues to read
             * zeroes, but we can only widen the result to the
             * unallocated length we learned from an earlier
             * iteration.
             */
            *pnum = nb_sectors;
        }
        if (ret & (BDRV_BLOCK_ZERO | BDRV_BLOCK_DATA)) {
            break;
        }
        /* [sector_num, pnum] unallocated on this layer, which could be only
         * the first part of [sector_num, nb_sectors].  */
        nb_sectors = MIN(nb_sectors, *pnum);
        first = false;
    }
    return ret;
}
```

```
/*
 * Returns the allocation status of the specified sectors.
 * Drivers not implementing the functionality are assumed to not support
 * backing files, hence all their sectors are reported as allocated.
 *
 * If 'sector_num' is beyond the end of the disk image the return value is
 * BDRV_BLOCK_EOF and 'pnum' is set to 0.
 *
 * 'pnum' is set to the number of sectors (including and immediately following
 * the specified sector) that are known to be in the same
 * allocated/unallocated state.
 *
 * 'nb_sectors' is the max value 'pnum' should be set to.  If nb_sectors goes
 * beyond the end of the disk image it will be clamped; if 'pnum' is set to
 * the end of the image, then the returned value will include BDRV_BLOCK_EOF.
 *
 * If returned value is positive and BDRV_BLOCK_OFFSET_VALID bit is set, 'file'
 * points to the BDS which the sector range is allocated in.
 *
 * 上面这一段注释很重要，代码中会用到各种BDRV_BLOCK_*，具体含义请参考“qemu &&
 * qcow2 notes - 杂七杂八[qemu block allocation status中各标识含义]”
 */
static int64_t coroutine_fn bdrv_co_get_block_status(BlockDriverState *bs,
                                                     int64_t sector_num,
                                                     int nb_sectors, int *pnum,
                                                     BlockDriverState **file)
{
    int64_t total_sectors;
    int64_t n;
    int64_t ret, ret2;

    *file = NULL;
    /*镜像总共有@total_sectors个sector*/
    total_sectors = bdrv_nb_sectors(bs);
    if (total_sectors < 0) {
        return total_sectors;
    }

    /*超出镜像大小*/
    if (sector_num >= total_sectors) {
        *pnum = 0;
        return BDRV_BLOCK_EOF;
    }

    /*从@sector_num开始至多查找@nb_sectors个sector*/
    n = total_sectors - sector_num;
    if (n < nb_sectors) {
        nb_sectors = n;
    }

    /* 如果驱动没有提供bdrv_co_get_block_status接口函数，则认为该驱动不支持基础镜像，
     * 因此认为所有的sector都分配了
     */
    if (!bs->drv->bdrv_co_get_block_status) {
        *pnum = nb_sectors;
        ret = BDRV_BLOCK_DATA | BDRV_BLOCK_ALLOCATED;
        if (sector_num + nb_sectors == total_sectors) {
            ret |= BDRV_BLOCK_EOF;
        }
        if (bs->drv->protocol_name) {
            ret |= BDRV_BLOCK_OFFSET_VALID | (sector_num * BDRV_SECTOR_SIZE);
            *file = bs;
        }
        return ret;
    }

    bdrv_inc_in_flight(bs);
    /*对应于qcow2，.bdrv_co_get_block_status = qcow2_co_get_block_status*/
    ret = bs->drv->bdrv_co_get_block_status(bs, sector_num, nb_sectors, pnum,
                                            file);
    if (ret < 0) {
        *pnum = 0;
        goto out;
    }

    /*BDRV_BLOCK_RAW暂时不关注，qcow2_co_get_block_status不会返回这种类型*/
    if (ret & BDRV_BLOCK_RAW) {
        assert(ret & BDRV_BLOCK_OFFSET_VALID && *file);
        ret = bdrv_co_get_block_status(*file, ret >> BDRV_SECTOR_BITS,
                                       *pnum, pnum, file);
        goto out;
    }

    if (ret & (BDRV_BLOCK_DATA | BDRV_BLOCK_ZERO)) {
        ret |= BDRV_BLOCK_ALLOCATED;
    } else {
        if (bdrv_unallocated_blocks_are_zero(bs)) {
            ret |= BDRV_BLOCK_ZERO;
        } else if (bs->backing) {
            BlockDriverState *bs2 = bs->backing->bs;
            int64_t nb_sectors2 = bdrv_nb_sectors(bs2);
            if (nb_sectors2 >= 0 && sector_num >= nb_sectors2) {
                ret |= BDRV_BLOCK_ZERO;
            }
        }
    }

    /*如果当前layer的查找过程中，设置了protocol layer @file，则进入下一层查找*/
    if (*file && *file != bs &&
        (ret & BDRV_BLOCK_DATA) && !(ret & BDRV_BLOCK_ZERO) &&
        (ret & BDRV_BLOCK_OFFSET_VALID)) {
        BlockDriverState *file2;
        int file_pnum;

        /*这里再次调用bdrv_co_get_block_status，最终会调用到raw_co_get_block_status*/
        ret2 = bdrv_co_get_block_status(*file, ret >> BDRV_SECTOR_BITS,
                                        *pnum, &file_pnum, &file2);
        if (ret2 >= 0) {
            /* Ignore errors.  This is just providing extra information, it
             * is useful but not necessary.
             */
            if (ret2 & BDRV_BLOCK_EOF &&
                (!file_pnum || ret2 & BDRV_BLOCK_ZERO)) {
                /*
                 * It is valid for the format block driver to read
                 * beyond the end of the underlying file's current
                 * size; such areas read as zero.
                 */
                ret |= BDRV_BLOCK_ZERO;
            } else {
                /* Limit request to the range reported by the protocol driver */
                *pnum = file_pnum;
                ret |= (ret2 & BDRV_BLOCK_ZERO);
            }
        }
    }

out:
    bdrv_dec_in_flight(bs);
    if (ret >= 0 && sector_num + *pnum == total_sectors) {
        ret |= BDRV_BLOCK_EOF;
    }
    return ret;
}
```

```
/*
 * Returns the allocation status of the specified sectors.
 *
 * If 'sector_num' is beyond the end of the disk image the return value is 0
 * and 'pnum' is set to 0.
 *
 * 'pnum' is set to the number of sectors (including and immediately following
 * the specified sector) that are known to be in the same
 * allocated/unallocated state.
 *
 * 'nb_sectors' is the max value 'pnum' should be set to.  If nb_sectors goes
 * beyond the end of the disk image it will be clamped.
 */
static int64_t coroutine_fn raw_co_get_block_status(BlockDriverState *bs,
                                                    int64_t sector_num,
                                                    int nb_sectors, int *pnum,
                                                    BlockDriverState **file)
{
    off_t start, data = 0, hole = 0;
    int64_t total_size;
    int ret;

    ret = fd_open(bs);
    if (ret < 0) {
        return ret;
    }

    start = sector_num * BDRV_SECTOR_SIZE;
    total_size = bdrv_getlength(bs);
    if (total_size < 0) {
        return total_size;
    } else if (start >= total_size) {
        *pnum = 0;
        return 0;
    } else if (start + nb_sectors * BDRV_SECTOR_SIZE > total_size) {
        nb_sectors = DIV_ROUND_UP(total_size - start, BDRV_SECTOR_SIZE);
    }

    /*检测分配情况*/
    ret = find_allocation(bs, start, &data, &hole);
    if (ret == -ENXIO) {
        /* Trailing hole */
        *pnum = nb_sectors;
        ret = BDRV_BLOCK_ZERO;
    } else if (ret < 0) {
        /* No info available, so pretend there are no holes */
        *pnum = nb_sectors;
        ret = BDRV_BLOCK_DATA;
    } else if (data == start) {
        /* On a data extent, compute sectors to the end of the extent,
         * possibly including a partial sector at EOF. */
        *pnum = MIN(nb_sectors, DIV_ROUND_UP(hole - start, BDRV_SECTOR_SIZE));
        ret = BDRV_BLOCK_DATA;
    } else {
        /* On a hole, compute sectors to the beginning of the next extent.  */
        assert(hole == start);
        *pnum = MIN(nb_sectors, (data - start) / BDRV_SECTOR_SIZE);
        ret = BDRV_BLOCK_ZERO;
    }
    *file = bs;
    return ret | BDRV_BLOCK_OFFSET_VALID | start;
}
```

```
/*
 * Find allocation range in @bs around offset @start.
 * May change underlying file descriptor's file offset.
 * If @start is not in a hole, store @start in @data, and the
 * beginning of the next hole in @hole, and return 0.
 * If @start is in a non-trailing hole, store @start in @hole and the
 * beginning of the next non-hole in @data, and return 0.
 * If @start is in a trailing hole or beyond EOF, return -ENXIO.
 * If we can't find out, return a negative errno other than -ENXIO.
 *
 * 通过调用lseek系统调用来实现分配情况的检测
 */
static int find_allocation(BlockDriverState *bs, off_t start,
                           off_t *data, off_t *hole)
{
#if defined SEEK_HOLE && defined SEEK_DATA
    BDRVRawState *s = bs->opaque;
    off_t offs;

    /*
     * SEEK_DATA cases:
     * D1. offs == start: start is in data
     * D2. offs > start: start is in a hole, next data at offs
     * D3. offs < 0, errno = ENXIO: either start is in a trailing hole
     *                              or start is beyond EOF
     *     If the latter happens, the file has been truncated behind
     *     our back since we opened it.  All bets are off then.
     *     Treating like a trailing hole is simplest.
     * D4. offs < 0, errno != ENXIO: we learned nothing
     */
    offs = lseek(s->fd, start, SEEK_DATA);
    if (offs < 0) {
        return -errno;          /* D3 or D4 */
    }
    assert(offs >= start);

    if (offs > start) {
        /* D2: in hole, next data at offs */
        *hole = start;
        *data = offs;
        return 0;
    }

    /* D1: in data, end not yet known */

    /*
     * SEEK_HOLE cases:
     * H1. offs == start: start is in a hole
     *     If this happens here, a hole has been dug behind our back
     *     since the previous lseek().
     * H2. offs > start: either start is in data, next hole at offs,
     *                   or start is in trailing hole, EOF at offs
     *     Linux treats trailing holes like any other hole: offs ==
     *     start.  Solaris seeks to EOF instead: offs > start (blech).
     *     If that happens here, a hole has been dug behind our back
     *     since the previous lseek().
     * H3. offs < 0, errno = ENXIO: start is beyond EOF
     *     If this happens, the file has been truncated behind our
     *     back since we opened it.  Treat it like a trailing hole.
     * H4. offs < 0, errno != ENXIO: we learned nothing
     *     Pretend we know nothing at all, i.e. "forget" about D1.
     */
    offs = lseek(s->fd, start, SEEK_HOLE);
    if (offs < 0) {
        return -errno;          /* D1 and (H3 or H4) */
    }
    assert(offs >= start);

    if (offs > start) {
        /*
         * D1 and H2: either in data, next hole at offs, or it was in
         * data but is now in a trailing hole.  In the latter case,
         * all bets are off.  Treating it as if it there was data all
         * the way to EOF is safe, so simply do that.
         */
        *data = start;
        *hole = offs;
        return 0;
    }

    /* D1 and H1 */
    return -EBUSY;
#else
    return -ENOTSUP;
#endif
}
```

```
static int64_t coroutine_fn qcow2_co_get_block_status(BlockDriverState *bs,
        int64_t sector_num, int nb_sectors, int *pnum, BlockDriverState **file)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t cluster_offset;
    int index_in_cluster, ret;
    unsigned int bytes;
    int64_t status = 0;

    bytes = MIN(INT_MAX, nb_sectors * BDRV_SECTOR_SIZE);
    qemu_co_mutex_lock(&s->lock);
    /* 在虚拟磁盘[(sector_num << 9), (sector_num << 9) + bytes)区间内查找最大连续的
     * 和(sector_num << 9)所对应的cluster具有相同类型的cluster，返回时，@cluster_offset
     * 表示(sector_num << 9)对应的cluster在镜像文件中的地址，@bytes表示这些相同类型
     * 的cluster占用的字节数，@ret表示这些cluster的类型，后面会分析该函数
     */
    ret = qcow2_get_cluster_offset(bs, sector_num << 9, &bytes,
                                   &cluster_offset);
    qemu_co_mutex_unlock(&s->lock);
    if (ret < 0) {
        return ret;
    }

    /*将@bytes字节转换为sector数目*/
    *pnum = bytes >> BDRV_SECTOR_BITS;
    if (cluster_offset != 0 && ret != QCOW2_CLUSTER_COMPRESSED &&
        !s->crypto) {
        index_in_cluster = sector_num & (s->cluster_sectors - 1);
        cluster_offset |= (index_in_cluster << BDRV_SECTOR_BITS);
        /*设置@file为protocol layer BlockDriverState*/
        *file = bs->file->bs;
        status |= BDRV_BLOCK_OFFSET_VALID | cluster_offset;
    }
    
    if (ret == QCOW2_CLUSTER_ZERO_PLAIN || ret == QCOW2_CLUSTER_ZERO_ALLOC) {
        status |= BDRV_BLOCK_ZERO;
    } else if (ret != QCOW2_CLUSTER_UNALLOCATED) {
        status |= BDRV_BLOCK_DATA;
    }
    
    return status;
}
```

```
static int coroutine_fn bdrv_driver_preadv(BlockDriverState *bs,
                                           uint64_t offset, uint64_t bytes,
                                           QEMUIOVector *qiov, int flags)
{
    BlockDriver *drv = bs->drv;
    int64_t sector_num;
    unsigned int nb_sectors;

    assert(!(flags & ~BDRV_REQ_MASK));

    /*对于qcow2来说，直接调用qcow2_co_preadv，在bdrv_aligned_preadv中传递的flags为0*/
    if (drv->bdrv_co_preadv) {
        return drv->bdrv_co_preadv(bs, offset, bytes, qiov, flags);
    }

    sector_num = offset >> BDRV_SECTOR_BITS;
    nb_sectors = bytes >> BDRV_SECTOR_BITS;

    assert((offset & (BDRV_SECTOR_SIZE - 1)) == 0);
    assert((bytes & (BDRV_SECTOR_SIZE - 1)) == 0);
    assert((bytes >> BDRV_SECTOR_BITS) <= BDRV_REQUEST_MAX_SECTORS);

    if (drv->bdrv_co_readv) {
        return drv->bdrv_co_readv(bs, sector_num, nb_sectors, qiov);
    } else {
        BlockAIOCB *acb;
        CoroutineIOCompletion co = {
            .coroutine = qemu_coroutine_self(),
        };

        acb = bs->drv->bdrv_aio_readv(bs, sector_num, qiov, nb_sectors,
                                      bdrv_co_io_em_complete, &co);
        if (acb == NULL) {
            return -EIO;
        } else {
            qemu_coroutine_yield();
            return co.ret;
        }
    }
}
```

```
static coroutine_fn int qcow2_co_preadv(BlockDriverState *bs, uint64_t offset,
                                        uint64_t bytes, QEMUIOVector *qiov,
                                        int flags)
{
    BDRVQcow2State *s = bs->opaque;
    int offset_in_cluster, n1;
    int ret;
    unsigned int cur_bytes; /* number of bytes in current iteration */
    uint64_t cluster_offset = 0;
    uint64_t bytes_done = 0;
    QEMUIOVector hd_qiov;
    uint8_t *cluster_data = NULL;

    qemu_iovec_init(&hd_qiov, qiov->niov);
    qemu_co_mutex_lock(&s->lock);
    while (bytes != 0) {
        cur_bytes = MIN(bytes, INT_MAX);
        /*获取虚拟磁盘中偏移@offset在镜像文件中的地址@cluster_offset及其类型@ret*/
        ret = qcow2_get_cluster_offset(bs, offset, &cur_bytes, &cluster_offset);
        offset_in_cluster = offset_into_cluster(s, offset);

        qemu_iovec_reset(&hd_qiov);
        qemu_iovec_concat(&hd_qiov, qiov, bytes_done, cur_bytes);

        switch (ret) {
        case QCOW2_CLUSTER_UNALLOCATED:
            /*如果该cluster尚未分配，且存在基础镜像，则尝试从基础镜像读取*/
            if (bs->backing) {
                n1 = qcow2_backing_read1(bs->backing->bs, &hd_qiov,
                                         offset, cur_bytes);
                if (n1 > 0) {
                    QEMUIOVector local_qiov;

                    qemu_iovec_init(&local_qiov, hd_qiov.niov);
                    qemu_iovec_concat(&local_qiov, &hd_qiov, 0, n1);
                    qemu_co_mutex_unlock(&s->lock);
                    ret = bdrv_co_preadv(bs->backing, offset, n1,
                                         &local_qiov, 0);
                    qemu_co_mutex_lock(&s->lock);
                    qemu_iovec_destroy(&local_qiov);
                }
            } else {
                /* 否则，直接填充0即可 */
                qemu_iovec_memset(&hd_qiov, 0, 0, cur_bytes);
            }
            break;

        case QCOW2_CLUSTER_ZERO_PLAIN:
        case QCOW2_CLUSTER_ZERO_ALLOC:
            /*直接填充0即可*/
            qemu_iovec_memset(&hd_qiov, 0, 0, cur_bytes);
            break;

        case QCOW2_CLUSTER_COMPRESSED:
            /*压缩类型的处理暂时不关注*/
            ......
            break;

        case QCOW2_CLUSTER_NORMAL:
            /*加密相关的处理暂时不关注*/
            ......
            
            qemu_co_mutex_unlock(&s->lock);
            /* 再次调用bdrv_co_preadv，但是是到protocol layer层读取，如果protocol layer
             * 为file的话，调用链则是：bdrv_co_preadv -> bdrv_aligned_preadv -> 
             * bdrv_driver_preadv -> drv->bdrv_co_preadv -> raw_co_preadv
             */
            ret = bdrv_co_preadv(bs->file,
                                 cluster_offset + offset_in_cluster,
                                 cur_bytes, &hd_qiov, 0);
            qemu_co_mutex_lock(&s->lock);
            break;

        default:
            g_assert_not_reached();
            ret = -EIO;
            goto fail;
        }

        bytes -= cur_bytes;
        offset += cur_bytes;
        bytes_done += cur_bytes;
    }
    ret = 0;

fail:
    qemu_co_mutex_unlock(&s->lock);
    qemu_iovec_destroy(&hd_qiov);
    qemu_vfree(cluster_data);

    return ret;
}
```

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

    /*计算guest offset @offset在cluster内部偏移*/
    offset_in_cluster = offset_into_cluster(s, offset);
    bytes_needed = (uint64_t) *bytes + offset_in_cluster;

    /*2 ^ l1_bits 表示一个L1 table entry中可以管理多少个字节，用于计算L1 entry index*/
    l1_bits = s->l2_bits + s->cluster_bits;

    /* compute how many bytes there are between the start of the cluster
     * containing offset and the end of the l1 entry */
    bytes_available = (1ULL << l1_bits) - (offset & ((1ULL << l1_bits) - 1))
                    + offset_in_cluster;

    if (bytes_needed > bytes_available) {
        bytes_needed = bytes_available;
    }

    *cluster_offset = 0;

    /* seek to the l2 offset in the l1 table */
    /*根据@offset计算出@l1_index，即@offset对应的cluster在L1 table中索引*/
    l1_index = offset >> l1_bits;
    if (l1_index >= s->l1_size) {
        type = QCOW2_CLUSTER_UNALLOCATED;
        goto out;
    }

    /*@l2_offset表示@offset对应的L2 table的地址，必须对齐到cluster*/
    l2_offset = s->l1_table[l1_index] & L1E_OFFSET_MASK;
    if (!l2_offset) {
        /*如果@l2_offset为0*/
        type = QCOW2_CLUSTER_UNALLOCATED;
        goto out;
    }

    /*@l2_offset未对齐到cluster*/
    if (offset_into_cluster(s, l2_offset)) {
        qcow2_signal_corruption(bs, true, -1, -1, "L2 table offset %#" PRIx64
                                " unaligned (L1 index: %#" PRIx64 ")",
                                l2_offset, l1_index);
        return -EIO;
    }

    /* load the l2 table in memory */
    /* 加载L2 table，如果L2 table在cache中，就直接使用cache中的，如果不在cache中，
     * 则需要从镜像文件中加载
     */
    ret = l2_load(bs, l2_offset, &l2_table);
    if (ret < 0) {
        return ret;
    }

    /* find the cluster offset for the given disk offset */
    /*根据@offset计算出其在L2 table中的索引号，并查询@offset所在的cluster在镜像文件中的地址*/
    l2_index = offset_to_l2_index(s, offset);
    *cluster_offset = be64_to_cpu(l2_table[l2_index]);

    /*本次读取涉及的cluster数目*/
    nb_clusters = size_to_clusters(s, bytes_needed);
    /* bytes_needed <= *bytes + offset_in_cluster, both of which are unsigned
     * integers; the minimum cluster size is 512, so this assertion is always
     * true */
    assert(nb_clusters <= INT_MAX);

    /*@cluster_offset中除了存放地址外，还存放了相关标识位，比如cluster的类型信息等*/
    type = qcow2_get_cluster_type(*cluster_offset);
    if (s->qcow_version < 3 && (type == QCOW2_CLUSTER_ZERO_PLAIN ||
                                type == QCOW2_CLUSTER_ZERO_ALLOC)) {
        qcow2_signal_corruption(bs, true, -1, -1, "Zero cluster entry found"
                                " in pre-v3 image (L2 offset: %#" PRIx64
                                ", L2 index: %#x)", l2_offset, l2_index);
        ret = -EIO;
        goto fail;
    }
    switch (type) {
    case QCOW2_CLUSTER_COMPRESSED:
        /* Compressed clusters can only be processed one by one */
        c = 1;
        *cluster_offset &= L2E_COMPRESSED_OFFSET_SIZE_MASK;
        break;
    case QCOW2_CLUSTER_ZERO_PLAIN:
    case QCOW2_CLUSTER_UNALLOCATED:
        /* how many empty clusters ? */
        /* 检查从@l2_index开始的cluster中有多少个连续的zero_plain或者unallocated类型的
         * cluster，至多检查@nb_clusters个cluster
         */
        c = count_contiguous_clusters_unallocated(nb_clusters,
                                                  &l2_table[l2_index], type);
        *cluster_offset = 0;
        break;
    case QCOW2_CLUSTER_ZERO_ALLOC:
    case QCOW2_CLUSTER_NORMAL:
        /* how many allocated clusters ? */
        /* 检查从@l2_index开始的cluster中有多少个连续的zero_alloc或者normal类型的cluster，
         * 至多检查@nb_clusters个cluster
         */
        c = count_contiguous_clusters(nb_clusters, s->cluster_size,
                                      &l2_table[l2_index], QCOW_OFLAG_ZERO);
        *cluster_offset &= L2E_OFFSET_MASK;
        if (offset_into_cluster(s, *cluster_offset)) {
            qcow2_signal_corruption(bs, true, -1, -1,
                                    "Cluster allocation offset %#"
                                    PRIx64 " unaligned (L2 offset: %#" PRIx64
                                    ", L2 index: %#x)", *cluster_offset,
                                    l2_offset, l2_index);
            ret = -EIO;
            goto fail;
        }
        break;
    default:
        abort();
    }

    qcow2_cache_put(bs, s->l2_table_cache, (void**) &l2_table);

    /*至多读取@c个连续的同类型的cluster*/
    bytes_available = (int64_t)c * s->cluster_size;

out:
    if (bytes_available > bytes_needed) {
        bytes_available = bytes_needed;
    }

    /* bytes_available <= bytes_needed <= *bytes + offset_in_cluster;
     * subtracting offset_in_cluster will therefore definitely yield something
     * not exceeding UINT_MAX */
    assert(bytes_available - offset_in_cluster <= UINT_MAX);
    /*至多从@offset开始读取@*bytes字节*/
    *bytes = bytes_available - offset_in_cluster;

    return type;

fail:
    qcow2_cache_put(bs, s->l2_table_cache, (void **)&l2_table);
    return ret;
}
```

```
static int coroutine_fn raw_co_preadv(BlockDriverState *bs, uint64_t offset,
                                      uint64_t bytes, QEMUIOVector *qiov,
                                      int flags)
{
    return raw_co_prw(bs, offset, bytes, qiov, QEMU_AIO_READ);
}
```

```
static int coroutine_fn raw_co_prw(BlockDriverState *bs, uint64_t offset,
                                   uint64_t bytes, QEMUIOVector *qiov, int type)
{
    BDRVRawState *s = bs->opaque;

    if (fd_open(bs) < 0)
        return -EIO;

    /*
     * Check if the underlying device requires requests to be aligned,
     * and if the request we are trying to submit is aligned or not.
     * If this is the case tell the low-level driver that it needs
     * to copy the buffer.
     */
    /* raw格式在打开标识中没有设置BDRV_O_NOCACHE（不使用host page cache）的情况下会
     * 设置needs_alignment为true
     */
    if (s->needs_alignment) {
        /*检查@qiov中所有的memory是不是对齐到sector的，linux AIO要求对齐到sector*/
        if (!bdrv_qiov_is_aligned(bs, qiov)) {
            type |= QEMU_AIO_MISALIGNED;
#ifdef CONFIG_LINUX_AIO
        } else if (s->use_linux_aio) {
            /*使用linux AIO，下面以linux AIO进行分析*/
            LinuxAioState *aio = aio_get_linux_aio(bdrv_get_aio_context(bs));
            assert(qiov->size == bytes);
            return laio_co_submit(bs, aio, s->fd, offset, qiov, type);
#endif
        }
    }

    /*使用posix AIO*/
    return paio_submit_co(bs, s->fd, offset, qiov, bytes, type);
}
```

```
int coroutine_fn laio_co_submit(BlockDriverState *bs, LinuxAioState *s, int fd,
                                uint64_t offset, QEMUIOVector *qiov, int type)
{
    int ret;
    struct qemu_laiocb laiocb = {
        .co         = qemu_coroutine_self(),
        .nbytes     = qiov->size,
        .ctx        = s,
        .ret        = -EINPROGRESS,
        .is_read    = (type == QEMU_AIO_READ),
        .qiov       = qiov,
    };

    ret = laio_do_submit(fd, &laiocb, offset, type);
    if (ret < 0) {
        return ret;
    }

    if (laiocb.ret == -EINPROGRESS) {
        qemu_coroutine_yield();
    }
    return laiocb.ret;
}
```

```
static int laio_do_submit(int fd, struct qemu_laiocb *laiocb, off_t offset,
                          int type)
{
    LinuxAioState *s = laiocb->ctx;
    struct iocb *iocbs = &laiocb->iocb;
    QEMUIOVector *qiov = laiocb->qiov;

    /*调用linux AIO的准备函数*/
    switch (type) {
    case QEMU_AIO_WRITE:
        io_prep_pwritev(iocbs, fd, qiov->iov, qiov->niov, offset);
	break;
    case QEMU_AIO_READ:
        io_prep_preadv(iocbs, fd, qiov->iov, qiov->niov, offset);
	break;
    /* Currently Linux kernel does not support other operations */
    default:
        fprintf(stderr, "%s: invalid AIO request type 0x%x.\n",
                        __func__, type);
        return -EIO;
    }
    
    /*关联到特定的事件*/
    io_set_eventfd(&laiocb->iocb, event_notifier_get_fd(&s->e));
    
    /*链入pending队列（即使后面没有提交，也无所谓了，反正已经链入pending队列了）*/
    QSIMPLEQ_INSERT_TAIL(&s->io_q.pending, laiocb, next);
    /*增加队列中请求计数*/
    s->io_q.in_queue++;
    
    /*如果可以提交则，则触发提交下一个请求（不一定是本次的请求）*/
    if (!s->io_q.blocked &&
        (!s->io_q.plugged ||
         s->io_q.in_flight + s->io_q.in_queue >= MAX_EVENTS)) {
        ioq_submit(s);
    }

    return 0;
}
```

```
static void ioq_submit(LinuxAioState *s)
{
    int ret, len;
    struct qemu_laiocb *aiocb;
    struct iocb *iocbs[MAX_EVENTS];
    QSIMPLEQ_HEAD(, qemu_laiocb) completed;

    do {
        /*如果已经提交的请求个数超过最大可处理的事件数目，则不提交*/
        if (s->io_q.in_flight >= MAX_EVENTS) {
            break;
        }
        
        /* 依次遍历pending队列，挑选出至多（MAX_EVENTS - s->io_q.in_flight）个请求存
         * 放到@iocbs中，注意此时并没有将这些请求从pending队列中删除
         */
        len = 0;
        QSIMPLEQ_FOREACH(aiocb, &s->io_q.pending, next) {
            iocbs[len++] = &aiocb->iocb;
            if (s->io_q.in_flight + len >= MAX_EVENTS) {
                break;
            }
        }

        /* 直接调用linux AIO的submit接口提交@len个IO请求，这些IO请求存放在@iocbs中，
         * 返回值ret表示提交成功的请求个数
         */
        ret = io_submit(s->ctx, len, iocbs);
        if (ret == -EAGAIN) {
            break;
        }
        
        if (ret < 0) {
            /* Fail the first request, retry the rest */
            /* 所有请求都提交失败（注意是提交失败，不是执行失败），则将pending队列中的第
             * 一个请求移除并设置该请求的执行结果，继续提交除第一个请求以外的后续请求
             */
            aiocb = QSIMPLEQ_FIRST(&s->io_q.pending);
            QSIMPLEQ_REMOVE_HEAD(&s->io_q.pending, next);
            s->io_q.in_queue--;
            aiocb->ret = ret;
            qemu_laio_process_completion(aiocb);
            continue;
        }

        /*增加已经提交成功的请求的个数，同时减少pending队列中请求的个数*/
        s->io_q.in_flight += ret;
        s->io_q.in_queue  -= ret;
        
        /*将最后一个提交成功的请求之前的请求都添加到@completed链表中*/
        aiocb = container_of(iocbs[ret - 1], struct qemu_laiocb, iocb);
        QSIMPLEQ_SPLIT_AFTER(&s->io_q.pending, aiocb, next, &completed);
        
        /*只要是这批提交全部提交成功且pending队列不为空，则继续提交*/
    } while (ret == len && !QSIMPLEQ_EMPTY(&s->io_q.pending));
    
    /*如果pending队列中还有未提交的请求，则设置其为blocked状态*/
    s->io_q.blocked = (s->io_q.in_queue > 0);

    /*如果提交的请求数目不为0，则尝试处理执行完毕的请求*/
    if (s->io_q.in_flight) {
        /* We can try to complete something just right away if there are
         * still requests in-flight. */
        qemu_laio_process_completions(s);
        /*
         * Even we have completed everything (in_flight == 0), the queue can
         * have still pended requests (in_queue > 0).  We do not attempt to
         * repeat submission to avoid IO hang.  The reason is simple: s->e is
         * still set and completion callback will be called shortly and all
         * pended requests will be submitted from there.
         */
    }
}
```

```
/**
 * qemu_laio_process_completions:
 * @s: AIO state
 *
 * Fetches completed I/O requests and invokes their callbacks.
 *
 * The function is somewhat tricky because it supports nested event loops, for
 * example when a request callback invokes aio_poll().  In order to do this,
 * indices are kept in LinuxAioState.  Function schedules BH completion so it
 * can be called again in a nested event loop.  When there are no events left
 * to complete the BH is being canceled.
 */
static void qemu_laio_process_completions(LinuxAioState *s)
{
    struct io_event *events;

    /* Reschedule so nested event loops see currently pending completions */
    qemu_bh_schedule(s->completion_bh);

    while ((s->event_max = io_getevents_advance_and_peek(s->ctx, &events,
                                                         s->event_idx))) {
        for (s->event_idx = 0; s->event_idx < s->event_max; ) {
            struct iocb *iocb = events[s->event_idx].obj;
            struct qemu_laiocb *laiocb =
                container_of(iocb, struct qemu_laiocb, iocb);

            laiocb->ret = io_event_ret(&events[s->event_idx]);

            /* Change counters one-by-one because we can be nested. */
            s->io_q.in_flight--;
            s->event_idx++;
            qemu_laio_process_completion(laiocb);
        }
    }

    qemu_bh_cancel(s->completion_bh);

    /* If we are nested we have to notify the level above that we are done
     * by setting event_max to zero, upper level will then jump out of it's
     * own `for` loop.  If we are the last all counters droped to zero. */
    s->event_max = 0;
    s->event_idx = 0;
}
```
