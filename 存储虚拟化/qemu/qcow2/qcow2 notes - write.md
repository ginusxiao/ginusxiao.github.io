# 写镜像文件
## 参考资料
[virtio-blk后端处理-请求接收、解析、提交](http://blog.csdn.net/LPSTC123/article/details/45171515)

## 源码分析

```
/*我们从bdrv_co_pwritev来开始*/
```

```
/*
 * Handle a write request in coroutine context
 * 对于qcow2来说，如果没有开启加密，则alignment为1，否则为512
 */
int coroutine_fn bdrv_co_pwritev(BdrvChild *child,
    int64_t offset, unsigned int bytes, QEMUIOVector *qiov,
    BdrvRequestFlags flags)
{
    BlockDriverState *bs = child->bs;
    BdrvTrackedRequest req;
    uint64_t align = bs->bl.request_alignment;
    uint8_t *head_buf = NULL;
    uint8_t *tail_buf = NULL;
    QEMUIOVector local_qiov;
    bool use_local_qiov = false;
    int ret;

    trace_bdrv_co_pwritev(child->bs, offset, bytes, flags);

    if (!bs->drv) {
        return -ENOMEDIUM;
    }
    if (bs->read_only) {
        return -EPERM;
    }
    assert(!(bs->open_flags & BDRV_O_INACTIVE));

    ret = bdrv_check_byte_request(bs, offset, bytes);
    if (ret < 0) {
        return ret;
    }

    /*增加BlockDriverState::in_flight计数*/
    bdrv_inc_in_flight(bs);
    /*
     * Align write if necessary by performing a read-modify-write cycle.
     * Pad qiov with the read parts and be sure to have a tracked request not
     * only for bdrv_aligned_pwritev, but also for the reads of the RMW cycle.
     */
     
    /* 每一个BlockDriverState结构都包含一个tracked_requests链表，顾名思义用于跟
     * 踪所有的读写请求，在读写请求执行之前调用tracked_request_begin()将请求添
     * 加到该链表中，在读写请求执行之后则调用tracked_request_end()将请求从该链
     * 表中移除
     */
    tracked_request_begin(&req, bs, offset, bytes, BDRV_TRACKED_WRITE);

    if (!qiov) {
        /*如果qiov为空，则说明对于[offset, offset + bytes)的区域写0*/
        ret = bdrv_co_do_zero_pwritev(child, offset, bytes, flags, &req);
        goto out;
    }

    /*如果起始偏移是非对齐的，则执行read-modify-write，进行对齐处理*/
    if (offset & (align - 1)) {
        QEMUIOVector head_qiov;
        struct iovec head_iov;

        /*标记@req需要serialising，并添加到BlockDriverState::serialising_in_flight链表中*/
        mark_request_serialising(&req, align);
        /*等待与之存在交集的请求处理完毕*/
        wait_serialising_requests(&req);

        /*分配@head_buf用于初始化@head_iov，进一步初始化@head_qiov*/
        head_buf = qemu_blockalign(bs, align);
        head_iov = (struct iovec) {
            .iov_base   = head_buf,
            .iov_len    = align,
        };
        qemu_iovec_init_external(&head_qiov, &head_iov, 1);

        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_HEAD);
        /*读取起始于offset & ~(align - 1)，长度为align字节的数据到buf中*/
        ret = bdrv_aligned_preadv(child, &req, offset & ~(align - 1), align,
                                  align, &head_qiov, 0);
        if (ret < 0) {
            goto fail;
        }
        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_AFTER_HEAD);

        /* 分配@local_qiov，大小为@qiov->niov + 2，因为@local_qiov中要同时存放@qiov
         * 中所有的iovec、头部未对齐部分相关的@head_iov以及尾部未对齐部分相关的@tail_iov
         */
        qemu_iovec_init(&local_qiov, qiov->niov + 2);
        /*先把@head_iov相关的@head_buf添加到@local_qiov中*/
        qemu_iovec_add(&local_qiov, head_buf, offset & (align - 1));
        /*再把@qiov添加到@local_qiov中*/
        qemu_iovec_concat(&local_qiov, qiov, 0, qiov->size);
        use_local_qiov = true;

        bytes += offset & (align - 1);
        offset = offset & ~(align - 1);

        /* We have read the tail already if the request is smaller
         * than one aligned block.
         */
        if (bytes < align) {
            /* 如果@bytes小于@align，表明当前请求请求的数据小于一个aligned block，
             * 为了对齐写，需要将当前aligned block的最后一部分也添加到@local_qiov
             */
            qemu_iovec_add(&local_qiov, head_buf + bytes, align - bytes);
            bytes = align;
        }
    }

    /* 如果结束偏移是非对齐的，也执行read-modify-write，进行对齐处理，处理逻辑跟
     * 头不对齐的处理是类似的，不再赘述
     */
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

    /*执行对齐后的写，注意这里的@offset和@bytes分别是向下对齐和向上对齐后的*/
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
    /*跟前面的tracked_request_begin()相对应*/
    tracked_request_end(&req);
    /*跟前面的bdrv_inc_in_flight()相对应*/
    bdrv_dec_in_flight(bs);
    return ret;
}
```

```
static void tracked_request_begin(BdrvTrackedRequest *req,
                                  BlockDriverState *bs,
                                  int64_t offset,
                                  unsigned int bytes,
                                  enum BdrvTrackedRequestType type)
{
    /*设置BdrvTrackedRequest结构*/
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

    /*初始化BdrvTrackedRequest::wait_queue，该队列中用于存放所有被该请求阻塞的协程*/
    qemu_co_queue_init(&req->wait_queue);

    /*将当前请求@req添加到BlockDriverState::tracked_requests链表中*/
    qemu_co_mutex_lock(&bs->reqs_lock);
    QLIST_INSERT_HEAD(&bs->tracked_requests, req, list);
    qemu_co_mutex_unlock(&bs->reqs_lock);
}
```

```
static void tracked_request_end(BdrvTrackedRequest *req)
{
    /*如果@req设置了serialising标识，则减少BlockDriverState::serialising_in_flight计数*/
    if (req->serialising) {
        atomic_dec(&req->bs->serialising_in_flight);
    }

    qemu_co_mutex_lock(&req->bs->reqs_lock);
    /*从BlockDriverState::tracked_requests链表中移除该请求*/
    QLIST_REMOVE(req, list);
    /*唤醒所有等待该请求的协程*/
    qemu_co_queue_restart_all(&req->wait_queue);
    qemu_co_mutex_unlock(&req->bs->reqs_lock);
}
```

```
static int coroutine_fn bdrv_co_do_zero_pwritev(BdrvChild *child,
                                                int64_t offset,
                                                unsigned int bytes,
                                                BdrvRequestFlags flags,
                                                BdrvTrackedRequest *req)
{
    BlockDriverState *bs = child->bs;
    uint8_t *buf = NULL;
    QEMUIOVector local_qiov;
    struct iovec iov;
    uint64_t align = bs->bl.request_alignment;
    unsigned int head_padding_bytes, tail_padding_bytes;
    int ret = 0;

    /*@head_padding_bytes和@tail_padding_bytes分别表示头部和尾部对齐处理需要填充的字节数*/
    head_padding_bytes = offset & (align - 1);
    tail_padding_bytes = (align - (offset + bytes)) & (align - 1);

    assert(flags & BDRV_REQ_ZERO_WRITE);
    if (head_padding_bytes || tail_padding_bytes) {
        /*为@iov分配buffer并采用@iov来初始化@local_qiov*/
        buf = qemu_blockalign(bs, align);
        iov = (struct iovec) {
            .iov_base   = buf,
            .iov_len    = align,
        };
        qemu_iovec_init_external(&local_qiov, &iov, 1);
    }
    
    /*处理头部的需要对其处理的部分*/
    if (head_padding_bytes) {
        uint64_t zero_bytes = MIN(bytes, align - head_padding_bytes);

        /* RMW the unaligned part before head. */
        mark_request_serialising(req, align);
        /*等待与@req存在交集的请求处理完毕*/
        wait_serialising_requests(req);
        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_HEAD);
        /* 读取head部分的数据，起始偏移为offset & ~(align - 1)，长度为align，
         * 读取的数据存放于前面申请的@buf中，见bdrv_aligne_preadv
         * 在bdrv_aligned_preadv中传递的最后一个参数是0，在读取之前还会再次调用
         * wait_serialising_requests来确保与之存在交叠的且设置了serialising标识
         * 的请求执行完毕
         */
        ret = bdrv_aligned_preadv(child, req, offset & ~(align - 1), align,
                                  align, &local_qiov, 0);
        if (ret < 0) {
            goto fail;
        }
        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_AFTER_HEAD);

        /*@buf中起始偏移为@head_padding_bytes起的@zero_bytes字节设置为0*/
        memset(buf + head_padding_bytes, 0, zero_bytes);
        /*将@buf中的内容写入@起始偏移为@offset & ~(align - 1)，长度为align的区间中*/
        ret = bdrv_aligned_pwritev(child, req, offset & ~(align - 1), align,
                                   align, &local_qiov,
                                   flags & ~BDRV_REQ_ZERO_WRITE);
        if (ret < 0) {
            goto fail;
        }
        offset += zero_bytes;
        bytes -= zero_bytes;
    }

    assert(!bytes || (offset & (align - 1)) == 0);
    /*处理中间部分无需对齐处理的部分*/
    if (bytes >= align) {
        /* Write the aligned part in the middle. */
        uint64_t aligned_bytes = bytes & ~(align - 1);
        ret = bdrv_aligned_pwritev(child, req, offset, aligned_bytes, align,
                                   NULL, flags);
        if (ret < 0) {
            goto fail;
        }
        bytes -= aligned_bytes;
        offset += aligned_bytes;
    }

    assert(!bytes || (offset & (align - 1)) == 0);
    /*处理尾部需要对齐处理的部分，处理逻辑跟头部未对齐部分一样*/
    if (bytes) {
        assert(align == tail_padding_bytes + bytes);
        /* RMW the unaligned part after tail. */
        mark_request_serialising(req, align);
        wait_serialising_requests(req);
        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_TAIL);
        /*见bdrv_aligne_preadv*/
        ret = bdrv_aligned_preadv(child, req, offset, align,
                                  align, &local_qiov, 0);
        if (ret < 0) {
            goto fail;
        }
        bdrv_debug_event(bs, BLKDBG_PWRITEV_RMW_AFTER_TAIL);

        memset(buf, 0, bytes);
        ret = bdrv_aligned_pwritev(child, req, offset, align, align,
                                   &local_qiov, flags & ~BDRV_REQ_ZERO_WRITE);
    }
fail:
    qemu_vfree(buf);
    return ret;

}
```

```
static void mark_request_serialising(BdrvTrackedRequest *req, uint64_t align)
{
    /*@overlap_offset: 将@req->offset向下对齐到@align*/
    int64_t overlap_offset = req->offset & ~(align - 1);
    /* @overlap_bytes: 将(req->offset + req->bytes)向上对齐到@align之后，
     * 从@overlap_offset到它的字节数
     */
    unsigned int overlap_bytes = ROUND_UP(req->offset + req->bytes, align) - overlap_offset;

    /* 设置BdrvTrackedRequest::serialising标识，并添加到BlockDriverState::serialising_in_flight
     * 链表中
     */
    if (!req->serialising) {
        atomic_inc(&req->bs->serialising_in_flight);
        req->serialising = true;
    }

    req->overlap_offset = MIN(req->overlap_offset, overlap_offset);
    req->overlap_bytes = MAX(req->overlap_bytes, overlap_bytes);
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
/*
 * Forwards an already correctly aligned request to the BlockDriver. This
 * handles copy on read, zeroing after EOF, and fragmentation of large
 * reads; any other features must be implemented by the caller.
 *
 * 从注释可知，该函数主要执行一下 工作：
 * 处理copy on read
 * 读取偏移超过EOF的情况下，填充0的处理
 * 对于大块读（超过max_transfer）分片处理
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

    /* 断言@flags中至多包含BDRV_REQ_NO_SERIALISING和BDRV_REQ_COPY_ON_READ这两个标识，
     * 事实上，从bdrv_co_pwritev -> bdrv_aligned_preadv传递下来的时候，@flags中传递
     * 的是0，所以该断言一定成立
     */
    assert(!(flags & ~(BDRV_REQ_NO_SERIALISING | BDRV_REQ_COPY_ON_READ)));

    /* Handle Copy on Read and associated serialisation */
    /*在bdrv_co_pwritev -> bdrv_aligned_preadv调用链中@flags为0，所以不会进入该逻辑*/
    if (flags & BDRV_REQ_COPY_ON_READ) {
        /* If we touch the same cluster it counts as an overlap.  This
         * guarantees that allocating writes will be serialized and not race
         * with each other for the same cluster.  For example, in copy-on-read
         * it ensures that the CoR read and write operations are atomic and
         * guest writes cannot interleave between them. */
        mark_request_serialising(req, bdrv_get_cluster_size(bs));
    }

    /*在bdrv_co_pwritev -> bdrv_aligned_preadv调用链中@flags为0，所以会进入该逻辑*/
    if (!(flags & BDRV_REQ_NO_SERIALISING)) {
        /* 等待与当前请求存在相关性的正在进行中的其它请求（其它请求可能是读请求也可能
         * 是写请求，如果当前请求设置了serialising标识，或者其它请求设置了serialising
         * 标识，且当前请求和其它请求之间存在交叠，则认为二者存在相关性，当前请求需要
         * 等待其它请求执行完毕之后才继续执行）处理完毕
         */
        wait_serialising_requests(req);
    }

    /*在bdrv_co_pwritev -> bdrv_aligned_preadv调用链中@flags为0，所以不会进入该逻辑*/
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

        if (!ret || pnum != end - start) {
            ret = bdrv_co_do_copy_on_readv(child, offset, bytes, qiov);
            goto out;
        }
    }

    /* Forward the request to the BlockDriver, possibly fragmenting it */
    /*获取镜像的大小*/
    total_bytes = bdrv_getlength(bs);
    if (total_bytes < 0) {
        ret = total_bytes;
        goto out;
    }

    /*当前可以读取对齐到@align的最大字节数*/
    max_bytes = ROUND_UP(MAX(0, total_bytes - offset), align);
    /*无需分片读取的情况*/
    if (bytes <= max_bytes && bytes <= max_transfer) {
        ret = bdrv_driver_preadv(bs, offset, bytes, qiov, 0);
        goto out;
    }

    /* 分片读取的情况，@bytes_remaining表示总共需要读取的字节数，@max_bytes表示
     * 在到达文件末尾之前可以读取的最大字节数
     */
    while (bytes_remaining) {
        int num;

        if (max_bytes) {
            QEMUIOVector local_qiov;

            num = MIN(bytes_remaining, MIN(max_bytes, max_transfer));
            assert(num);
            qemu_iovec_init(&local_qiov, qiov->niov);
            qemu_iovec_concat(&local_qiov, qiov, bytes - bytes_remaining, num);
            /*关于bdrv_driver_preadv请参考“qemu && qcow2 notes - read”*/
            ret = bdrv_driver_preadv(bs, offset + bytes - bytes_remaining,
                                     num, &local_qiov, 0);
            max_bytes -= num;
            qemu_iovec_destroy(&local_qiov);
        } else {
            /*超出文件末尾的部分，直接填充0*/
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
/*
 * Forwards an already correctly aligned write request to the BlockDriver,
 * after possibly fragmenting it.
 */
static int coroutine_fn bdrv_aligned_pwritev(BdrvChild *child,
    BdrvTrackedRequest *req, int64_t offset, unsigned int bytes,
    int64_t align, QEMUIOVector *qiov, int flags)
{
    BlockDriverState *bs = child->bs;
    BlockDriver *drv = bs->drv;
    bool waited;
    int ret;

    int64_t start_sector = offset >> BDRV_SECTOR_BITS;
    int64_t end_sector = DIV_ROUND_UP(offset + bytes, BDRV_SECTOR_SIZE);
    uint64_t bytes_remaining = bytes;
    int max_transfer;

    /* BlockDriverState::dirty_bitmaps中如果任何一个bitmap设置了readonly标识，
     * 则不能执行写请求了
     */
    if (bdrv_has_readonly_bitmaps(bs)) {
        return -EPERM;
    }

    assert(is_power_of_2(align));
    assert((offset & (align - 1)) == 0);
    assert((bytes & (align - 1)) == 0);
    assert(!qiov || bytes == qiov->size);
    assert((bs->open_flags & BDRV_O_NO_IO) == 0);
    assert(!(flags & ~BDRV_REQ_MASK));
    max_transfer = QEMU_ALIGN_DOWN(MIN_NON_ZERO(bs->bl.max_transfer, INT_MAX),
                                   align);

    /* 这一句代码非常重要，虽然到这里的请求一定是对齐的，但是也必须要等待与
     * [req->overlap_offset, req->overlap_offset + req->overlap_bytes]存在
     * 交叠的设置了serialising标识的请求执行完毕，方可执行@req
     */
    waited = wait_serialising_requests(req);
    assert(!waited || !req->serialising);
    assert(req->overlap_offset <= offset);
    assert(offset + bytes <= req->overlap_offset + req->overlap_bytes);
    assert(child->perm & BLK_PERM_WRITE);
    assert(end_sector <= bs->total_sectors || child->perm & BLK_PERM_RESIZE);

    /*依次通知@BlockDriverState::before_write_notifiers中的notifier*/
    ret = notifier_with_return_list_notify(&bs->before_write_notifiers, req);
    /* 所有notifier都成功被notify，则检查是否支持全0检测，如果支持全0检测，
     * 且检测到@qiov中的数据都是0，则设置BDRV_REQ_ZERO_WRITE或者BDRV_REQ_MAY_UNMAP
     * 标识，其中BDRV_REQ_ZERO_WRITE表示写入的是0，BDRV_REQ_MAY_UNMAP表示允许通过
     * unmap/discard相应的数据块来实现写入全0这一操作
     */
    if (!ret && bs->detect_zeroes != BLOCKDEV_DETECT_ZEROES_OPTIONS_OFF &&
        !(flags & BDRV_REQ_ZERO_WRITE) && drv->bdrv_co_pwrite_zeroes &&
        qemu_iovec_is_zero(qiov)) {
        flags |= BDRV_REQ_ZERO_WRITE;
        if (bs->detect_zeroes == BLOCKDEV_DETECT_ZEROES_OPTIONS_UNMAP) {
            /* 当且仅当打开设备的时候指定了BDRV_O_UNMAP标识，且设置了
             * BLOCKDEV_DETECT_ZEROES_OPTIONS_UNMAP选项，才会设置
             * BDRV_REQ_MAY_UNMAP
             */
            flags |= BDRV_REQ_MAY_UNMAP;
        }
    }

    if (ret < 0) {
        /* Do nothing, write notifier decided to fail this request */
    } else if (flags & BDRV_REQ_ZERO_WRITE) {
        /*如果支持全0检测的情况下调用bdrv_co_do_pwrite_zeroes，暂不关注*/
        bdrv_debug_event(bs, BLKDBG_PWRITEV_ZERO);
        ret = bdrv_co_do_pwrite_zeroes(bs, offset, bytes, flags);
    } else if (flags & BDRV_REQ_WRITE_COMPRESSED) {
        /*如果支持压缩的情况下调用bdrv_driver_pwritev_compressed，暂不关注*/
        ret = bdrv_driver_pwritev_compressed(bs, offset, bytes, qiov);
    } else if (bytes <= max_transfer) {
        /* 要写入的数据总量@bytes小于@max_transfer则直接写入，下面会分析之，
         * 如果cache.write_back未被设置，则@flags中会被设置BDRV_REQ_FUA标识
         */
        bdrv_debug_event(bs, BLKDBG_PWRITEV);
        ret = bdrv_driver_pwritev(bs, offset, bytes, qiov, flags);
    } else {
        /*否则要写的数据大小大于设定的传输大小，则分片调用bdrv_driver_pwritev*/
        bdrv_debug_event(bs, BLKDBG_PWRITEV);
        while (bytes_remaining) {
            int num = MIN(bytes_remaining, max_transfer);
            QEMUIOVector local_qiov;
            
            /*@local_flags直接用@flags初始化，将被传递给bdrv_driver_pwritev*/
            int local_flags = flags;

            /* 结合上面的“local_flags = flags”赋值和这里的if语句可知：当且仅当是最
             * 后一个分片且@flags中设置了BDRV_REQ_FUA标识的情况下@local_flags中才会
             * 设置BDRV_REQ_FUA标识，而根据前面的分析没有cache.write_back未被设置的
             * 情况下@flags中就会被设置BDRV_REQ_FUA标识，所以可以这么说，当cache.write_back
             * 未被设置的情况下，在分片写逻辑中只有最后一个分片的写需要模拟FUA(结合
             * bdrv_driver_pwritev的实现可知，模拟FUA就是执行flush)
             */
            assert(num);
            if (num < bytes_remaining && (flags & BDRV_REQ_FUA) &&
                !(bs->supported_write_flags & BDRV_REQ_FUA)) {
                /* If FUA is going to be emulated by flush, we only
                 * need to flush on the last iteration */
                local_flags &= ~BDRV_REQ_FUA;
            }
            qemu_iovec_init(&local_qiov, qiov->niov);
            qemu_iovec_concat(&local_qiov, qiov, bytes - bytes_remaining, num);

            ret = bdrv_driver_pwritev(bs, offset + bytes - bytes_remaining,
                                      num, &local_qiov, local_flags);
            qemu_iovec_destroy(&local_qiov);
            if (ret < 0) {
                break;
            }
            bytes_remaining -= num;
        }
    }
    bdrv_debug_event(bs, BLKDBG_PWRITEV_DONE);

    /*增加当前的generation*/
    atomic_inc(&bs->write_gen);
    /*设置dirty bitmap*/
    bdrv_set_dirty(bs, start_sector, end_sector - start_sector);
    /*设置当前写入的最大的offset*/
    stat64_max(&bs->wr_highest_offset, offset + bytes);

    if (ret >= 0) {
        bs->total_sectors = MAX(bs->total_sectors, end_sector);
        ret = 0;
    }

    return ret;
}
```

```
void bdrv_set_dirty(BlockDriverState *bs, int64_t cur_sector,
                    int64_t nr_sectors)
{
    BdrvDirtyBitmap *bitmap;

    /* BlockDriverState::dirty_bitmaps中包含各种各样的dirty bitmap，
     * 如果该链表为空，则表明没有创建任何的dirty bitmap，就没什么可设置的了。
     * 关于dirty bitmap可以通过bdrv_create_dirty_bitmap创建dirty bitmap并添加
     * 到BlockDriverState::dirty_bitmaps中
     */
    if (QLIST_EMPTY(&bs->dirty_bitmaps)) {
        return;
    }

    bdrv_dirty_bitmaps_lock(bs);
    /*逐一遍历BlockDriverState::dirty_bitmaps，*/
    QLIST_FOREACH(bitmap, &bs->dirty_bitmaps, list) {
        if (!bdrv_dirty_bitmap_enabled(bitmap)) {
            continue;
        }
        assert(!bdrv_dirty_bitmap_readonly(bitmap));
        /*设置想相应的bit位*/
        hbitmap_set(bitmap->bitmap, cur_sector, nr_sectors);
    }
    bdrv_dirty_bitmaps_unlock(bs);
}
```

```
static int coroutine_fn bdrv_driver_pwritev(BlockDriverState *bs,
                                            uint64_t offset, uint64_t bytes,
                                            QEMUIOVector *qiov, int flags)
{
    BlockDriver *drv = bs->drv;
    int64_t sector_num;
    unsigned int nb_sectors;
    int ret;

    assert(!(flags & ~BDRV_REQ_MASK));

    /* 对于qemu来说当前bs->supported_write_flags只支持BDRV_REQ_FUA(见BlockDriverState
     * 中关于supported_write_flags的注释),关于BDRV_REQ_FUA的作用、BDRV_REQ_FUA在哪里
     * 被设置的请参考“qemu && qcow2 notes - block cache 模式参数解析”
     */
    /* 对于qcow2来说，.bdrv_co_pwritev = qcow2_co_pwritev，但是在qcow2_co_pwritev，
     * 传递进去的(flags & bs->supported_write_flags)未被使用，所以是否设置了
     * BDRV_REQ_FUA不影响其逻辑
     */
    if (drv->bdrv_co_pwritev) {
        ret = drv->bdrv_co_pwritev(bs, offset, bytes, qiov,
                                   flags & bs->supported_write_flags);
        /* 对于qcow2来说，bs->supported_write_flags未被设置，所以@flags跟传递进来的时
         * 候一样，如果@flags中设置了BDRV_REQ_FUA，则会跳转到emulate_flags处执行flush
         * 操作
         */
        flags &= ~bs->supported_write_flags;
        goto emulate_flags;
    }

    sector_num = offset >> BDRV_SECTOR_BITS;
    nb_sectors = bytes >> BDRV_SECTOR_BITS;

    assert((offset & (BDRV_SECTOR_SIZE - 1)) == 0);
    assert((bytes & (BDRV_SECTOR_SIZE - 1)) == 0);
    assert((bytes >> BDRV_SECTOR_BITS) <= BDRV_REQUEST_MAX_SECTORS);

    if (drv->bdrv_co_writev_flags) {
        ret = drv->bdrv_co_writev_flags(bs, sector_num, nb_sectors, qiov,
                                        flags & bs->supported_write_flags);
        flags &= ~bs->supported_write_flags;
    } else if (drv->bdrv_co_writev) {
        assert(!bs->supported_write_flags);
        ret = drv->bdrv_co_writev(bs, sector_num, nb_sectors, qiov);
    } else {
        BlockAIOCB *acb;
        CoroutineIOCompletion co = {
            .coroutine = qemu_coroutine_self(),
        };

        acb = bs->drv->bdrv_aio_writev(bs, sector_num, qiov, nb_sectors,
                                       bdrv_co_io_em_complete, &co);
        if (acb == NULL) {
            ret = -EIO;
        } else {
            qemu_coroutine_yield();
            ret = co.ret;
        }
    }

emulate_flags:
    /*如果执行成功且设置了BDRV_REQ_FUA标识，则执行bdrv_co_flush*/
    if (ret == 0 && (flags & BDRV_REQ_FUA)) {
        ret = bdrv_co_flush(bs);
    }

    return ret;
}
```

```
static coroutine_fn int qcow2_co_pwritev(BlockDriverState *bs, uint64_t offset,
                                         uint64_t bytes, QEMUIOVector *qiov,
                                         int flags)
{
    BDRVQcow2State *s = bs->opaque;
    int offset_in_cluster;
    int ret;
    unsigned int cur_bytes; /* number of sectors in current iteration */
    uint64_t cluster_offset;
    QEMUIOVector hd_qiov;
    uint64_t bytes_done = 0;
    uint8_t *cluster_data = NULL;
    QCowL2Meta *l2meta = NULL;

    trace_qcow2_writev_start_req(qemu_coroutine_self(), offset, bytes);

    qemu_iovec_init(&hd_qiov, qiov->niov);

    s->cluster_cache_offset = -1; /* disable compressed cache */

    qemu_co_mutex_lock(&s->lock);

    while (bytes != 0) {

        l2meta = NULL;

        trace_qcow2_writev_start_part(qemu_coroutine_self());
        offset_in_cluster = offset_into_cluster(s, offset);
        cur_bytes = MIN(bytes, INT_MAX);
        if (bs->encrypted) {
            cur_bytes = MIN(cur_bytes,
                            QCOW_MAX_CRYPT_CLUSTERS * s->cluster_size
                            - offset_in_cluster);
        }

        /* @offset: offset in guest virtul disk 
         * @cur_bytes: number of bytes to allocated
         * @cluster_offset: offset allocated(or found) in host qcow2 file
         * @l2meta: meta info regarding l2 table update
         * 
         * if clusters specified by @offset and @cur_bytes already allocated,
         * then l2meta->nb_clusters will be set to 0.
         *
         * if any of the cluster specified by @offset and @cur_bytes is newly
         * allocated, then l2meta->nb_clusters will be set to the number of 
         * contiguous clusters that have been allocated
         *
         * if the request conflicts with another write request in flight, the
         * coroutine is queued and will be reentered when the dependency has
         * completed.
         *
         * 该函数虽然在“qcow2 notes - create”中分析过，但是为了更好理解，这里
         * 再次分析，见qcow2_alloc_cluster_offset，当该函数返回时[@offset,
         * @offset + @cur_bytes)所在的区间是可以直接写入数据的，@cluster_offset
         * 表示guest offset @offset在镜像文件中的偏移
         */
        ret = qcow2_alloc_cluster_offset(bs, offset, &cur_bytes,
                                         &cluster_offset, &l2meta);
        if (ret < 0) {
            goto fail;
        }

        assert((cluster_offset & 511) == 0);

        qemu_iovec_reset(&hd_qiov);
        /* 将@qiov中的部分iovecs(起始于@bytes_done，总共@cur_bytes字节)链接到
         * @hd_qiov中
         */
        qemu_iovec_concat(&hd_qiov, qiov, bytes_done, cur_bytes);

        /*加密情况特殊处理，暂不关注*/
        if (bs->encrypted) {
            assert(s->crypto);
            if (!cluster_data) {
                cluster_data = qemu_try_blockalign(bs->file->bs,
                                                   QCOW_MAX_CRYPT_CLUSTERS
                                                   * s->cluster_size);
                if (cluster_data == NULL) {
                    ret = -ENOMEM;
                    goto fail;
                }
            }

            assert(hd_qiov.size <=
                   QCOW_MAX_CRYPT_CLUSTERS * s->cluster_size);
            qemu_iovec_to_buf(&hd_qiov, 0, cluster_data, hd_qiov.size);

            if (qcrypto_block_encrypt(s->crypto,
                                      (s->crypt_physical_offset ?
                                       cluster_offset + offset_in_cluster :
                                       offset) >> BDRV_SECTOR_BITS,
                                      cluster_data,
                                      cur_bytes, NULL) < 0) {
                ret = -EIO;
                goto fail;
            }

            qemu_iovec_reset(&hd_qiov);
            qemu_iovec_add(&hd_qiov, cluster_data, cur_bytes);
        }

        /* 检查起始于cluster_offset + offset_in_cluster，长度为cur_bytes的区间
         * 是否和元数据区存在交叠，如果存在交叠则返回小于0
         */
        ret = qcow2_pre_write_overlap_check(bs, 0,
                cluster_offset + offset_in_cluster, cur_bytes);
        if (ret < 0) {
            goto fail;
        }

        /* If we need to do COW, check if it's possible to merge the
         * writing of the guest data together with that of the COW regions.
         * If it's not possible (or not necessary) then write the
         * guest data now. 
         * 检查是否可以将当前写请求和@l2meta中记录的COW区域内的数据进行合并，如果可以合并，
         * 则会将@hd_qiov添加到@l2meta中，由@l2meta接管要写入的数据
         */
        if (!merge_cow(offset, cur_bytes, &hd_qiov, l2meta)) {
            /*如果不能合并（或者无需执行COW，所以无需合并），则直接写入*/
            qemu_co_mutex_unlock(&s->lock);
            BLKDBG_EVENT(bs->file, BLKDBG_WRITE_AIO);
            trace_qcow2_writev_data(qemu_coroutine_self(),
                                    cluster_offset + offset_in_cluster);
            /* 这里再次调用bdrv_co_pwritev，我们就是从block_co_pwritev -> bdrv_co_pwritev
             * 分析下来的啊，怎么又绕回去了呢？不是绕回去了，两次调用bdrv_co_pwritev的地方
             * 第一个参数是不一样的，在block_co_pwritev -> bdrv_co_pwritev中如下：
             *      ret = bdrv_co_pwritev(blk->root, offset, bytes, qiov, flags);
             * 而在qcow2_co_pwritev -> bdrv_co_pwritev中如下：
             *      ret = bdrv_co_pwritev(bs->file,
             *                       cluster_offset + offset_in_cluster,
             *                       cur_bytes, &hd_qiov, 0);
             * 前者使用的是blk->root，后者使用的是bs->file，blk->root和bs->file分别是怎么初
             * 始化的呢？这在open逻辑中实现的，如下：
             * 在img_open -> img_open_file -> blk_new_open：
             *     BlockBackend *blk;
             *     BlockDriverState *bs;
             *     blk = blk_new(perm, BLK_PERM_ALL);
             *     //创建BlockDriverState，并设置其BlockDriver，对于qcow2来说就是@bdrv_qcow2
             *     bs = bdrv_open(filename, reference, options, flags, errp);
             *     //创建BDrvChild，并设置其BlockDriverState为@bs
             *     blk->root = bdrv_root_attach_child(bs, "root", &child_root, perm,
             *                  BLK_PERM_ALL, blk, errp);
             *
             * 在blk_new_open -> bdrv_open -> bdrv_open_inherit -> bdrv_open_common ->
             * bdrv_open_driver -> drv->bdrv_open -> qcow2_open：
             *     //创建子BlockDriverState并关联到新的BDrvChild（bdrv_open_child返回值），
             *     //然后将该新的BDrvChild赋值给@bs->file，其中@bs是父BlockDriverState，在
             *     //内部实现中会为该新创建的BDrivChild设置其BlockDriver为@bdrv_file
             *     bs->file = bdrv_open_child(NULL, options, "file", bs, &child_file, false, errp);
             * 综上，@blk->root关联到父BlockDriverState，同时关联到的BlockDriver为@bdrv_qcow2
             *（对于qcow2格式来说），@bs->file则关联到子BlockDriverState，同时关联到的BlockDriver
             * 为@bdrv_file（对于protocol layer为file的情况下）
             * 
             * bdrv_co_pwritev -> bdrv_aligned_pwritev 在bdrv_aligned_pwritev中会通过BdrvChild
             * 找到关联的BlockDriverState，bdrv_aligned_pwritev ->bdrv_driver_pwritev中则会进
             * 进一步通过BlockDriverState找到关联BlockDriver，所以这里最终会找的@bdrv_file，并
             * 最终调用@bdrv_file::bdrv_co_pwritev = raw_co_pwritev，所以接下来分析的是
             * raw_co_pwritev
             */
            ret = bdrv_co_pwritev(bs->file,
                                  cluster_offset + offset_in_cluster,
                                  cur_bytes, &hd_qiov, 0);
            qemu_co_mutex_lock(&s->lock);
            if (ret < 0) {
                goto fail;
            }
        }

        /*依次根据@l2meta中记录的元信息执行COW，并更新L2 table*/
        while (l2meta != NULL) {
            QCowL2Meta *next;

            /*详见qcow2_alloc_cluster_link_l2*/
            ret = qcow2_alloc_cluster_link_l2(bs, l2meta);
            if (ret < 0) {
                goto fail;
            }

            /* Take the request off the list of running requests */
            /*将当前@l2meta移除链表*/
            if (l2meta->nb_clusters != 0) {
                QLIST_REMOVE(l2meta, next_in_flight);
            }

            /*唤醒所有依赖于@l2meta的写请求所在的协程*/
            qemu_co_queue_restart_all(&l2meta->dependent_requests);
            /*继续处理下一个@l2meta*/
            next = l2meta->next;
            g_free(l2meta);
            l2meta = next;
        }

        /*当前处理了@cur_bytes字节，更新接下来要处理的@offset和剩余的待处理的字节数@bytes*/
        bytes -= cur_bytes;
        offset += cur_bytes;
        bytes_done += cur_bytes;
        trace_qcow2_writev_done_part(qemu_coroutine_self(), cur_bytes);
    }
    ret = 0;

fail:
    while (l2meta != NULL) {
        QCowL2Meta *next;

        if (l2meta->nb_clusters != 0) {
            QLIST_REMOVE(l2meta, next_in_flight);
        }
        qemu_co_queue_restart_all(&l2meta->dependent_requests);

        next = l2meta->next;
        g_free(l2meta);
        l2meta = next;
    }

    qemu_co_mutex_unlock(&s->lock);

    qemu_iovec_destroy(&hd_qiov);
    qemu_vfree(cluster_data);
    trace_qcow2_writev_done_req(qemu_coroutine_self(), ret);

    return ret;
}
```

```
/*
 * alloc_cluster_offset
 *
 * For a given offset on the virtual disk, find the cluster offset in qcow2
 * file. If the offset is not found, allocate a new cluster.
 *
 * If the cluster was already allocated, m->nb_clusters is set to 0 and
 * other fields in m are meaningless.
 *
 * If the cluster is newly allocated, m->nb_clusters is set to the number of
 * contiguous clusters that have been allocated. In this case, the other
 * fields of m are valid and contain information about the first allocated
 * cluster.
 *
 * If the request conflicts with another write request in flight, the coroutine
 * is queued and will be reentered when the dependency has completed.
 *
 * Return 0 on success and -errno in error cases
 */
int qcow2_alloc_cluster_offset(BlockDriverState *bs, uint64_t offset,
                               unsigned int *bytes, uint64_t *host_offset,
                               QCowL2Meta **m)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t start, remaining;
    uint64_t cluster_offset;
    uint64_t cur_bytes;
    int ret;

    trace_qcow2_alloc_clusters_offset(qemu_coroutine_self(), offset, *bytes);

again:
    start = offset;
    remaining = *bytes;
    cluster_offset = 0;
    *host_offset = 0;
    cur_bytes = 0;
    *m = NULL;

    while (true) {
        /* 因为只有这一个地方设置@host_offset，从start_of_cluster实现来看，用于
         * 获取@cluster_offset所在的cluster的起始地址，host_offset一定是对齐到
         * cluster的
         */
        if (!*host_offset) {
            *host_offset = start_of_cluster(s, cluster_offset);
        }

        assert(remaining >= cur_bytes);

        start           += cur_bytes;
        remaining       -= cur_bytes;
        cluster_offset  += cur_bytes;

        if (remaining == 0) {
            break;
        }

        cur_bytes = remaining;

        /**
         * Now start gathering as many contiguous clusters as possible:
         *
         * 1. Check for overlaps with in-flight allocations
         *
         *      a) Overlap not in the first cluster -> shorten this request and
         *         let the caller handle the rest in its next loop iteration.
         *
         *      b) Real overlaps of two requests. Yield and restart the search
         *         for contiguous clusters (the situation could have changed
         *         while we were sleeping)
         *
         *      c) TODO: Request starts in the same cluster as the in-flight
         *         allocation ends. Shorten the COW of the in-fight allocation,
         *         set cluster_offset to write to the same cluster and set up
         *         the right synchronisation between the in-flight request and
         *         the new one.
         *
         * 虽然在bdrv_co_pwritev -> bdrv_aligned_pwritev -> bdrv_driver_pwritev
         * -> qcow2_co_pwritev调用链中已经多次采用mark_request_serialising和
         * wait_serialising_requests来保证请求之间存在交叠情况下的读写相关和
         * 写写相关，但是在wait_serialising_requests中是按照对齐到
         * bs->bl.request_alignment后的请求区间是否存在交叠来判断是否存在相关性
         * 的，而handle_dependencies是用来处理两个不同的请求要求为同一个guest地
         * 址分配空间的情况的
         *
         * 当成功返回的时候，@cur_bytes是从@start开始的不依赖于任何正在进行中的
         * 写请求的最大的连续的字节数
         */
        ret = handle_dependencies(bs, start, &cur_bytes, m);
        if (ret == -EAGAIN) {
            /* Currently handle_dependencies() doesn't yield if we already had
             * an allocation. If it did, we would have to clean up the L2Meta
             * structs before starting over. */
            assert(*m == NULL);
            goto again;
        } else if (ret < 0) {
            return ret;
        } else if (cur_bytes == 0) {
            break;
        } else {
            /* handle_dependencies() may have decreased cur_bytes (shortened
             * the allocations below) so that the next dependency is processed
             * correctly during the next loop iteration. */
        }

        /* 继续处理当前和其它正在进行的写请求不存在依赖关系的[start, start + cur_bytes)
         * 区间，该区间涉及到的clusters可能尚未分配，也可能已经分配了，但是当前正在进行
         * 的写请求中未涉及到这些clusters
         *
         * handle_copied返回时，如果ret > 0，则@cur_bytes表示起始于@start的长度为@cur_bytes
         * 的区间无需执行COW
         */
        /*
         * 2. Count contiguous COPIED clusters.
         */
        ret = handle_copied(bs, start, &cluster_offset, &cur_bytes, m);
        if (ret < 0) {
            return ret;
        } else if (ret) {
            /* ret > 0 表示handle_copied有所进展，[start, start + cur_bytes)所在的区间无需
             * 执行COW，继续处理请求中从start + cur_bytes开始的剩余区间
             */
            continue;
        } else if (cur_bytes == 0) {
            /*@cluster_offset不正确*/
            break;
        }

        /* 至此，handle_copied返回0，且@cur_bytes > 0，表明@guest_offset对应的cluster尚未分
         * 配或者分配了但是需要执行COW，无论是尚未分配还是需要执行COW，两种情况都需要去分配
         * 新的cluster，如果需要执行COW，则COW相关的信息存放于@m中*/
        /*
         * 3. If the request still hasn't completed, allocate new clusters,
         *    considering any cluster_offset of steps 1c or 2.
         */
        ret = handle_alloc(bs, start, &cluster_offset, &cur_bytes, m);
        if (ret < 0) {
            return ret;
        } else if (ret) {
            continue;
        } else {
            assert(cur_bytes == 0);
            break;
        }
    }

    *bytes -= remaining;
    assert(*bytes > 0);
    assert(*host_offset != 0);

    return 0;
}
```

```
/*
 * Check if there already is an AIO write request in flight which allocates
 * the same cluster. In this case we need to wait until the previous
 * request has completed and updated the L2 table accordingly.
 *
 * Returns:
 *   0       if there was no dependency. *cur_bytes indicates the number of
 *           bytes from guest_offset that can be read before the next
 *           dependency must be processed (or the request is complete)
 *
 *   -EAGAIN if we had to wait for another request, previously gathered
 *           information on cluster allocation may be invalid now. The caller
 *           must start over anyway, so consider *cur_bytes undefined.
 * 
 * 关于handle_dependencies的实现，结合qcow2_alloc_cluster_offset中关于该函数
 * 的注释看会更直观.
 *
 * 该函数返回时：
 * 返回0：
 * 如果*cur_bytes > 0，则[guest_offset, guest_offset + *cur_bytes)的区间和当
 * 前正在进行中的写请求不存在依赖关系
 * 如果*cur_bytes为0，则表示当前写请求和其它正在进行的写请求存在依赖关系
 * 
 * 返回EAGAIN：
 * 需要等待另外一个写请求完成，当前协程被添加到阻塞它的写请求的等待队列中，当
 * 阻塞它的写请求的协程执行完毕之后会唤醒处于等待队列中的写请求.
 */
static int handle_dependencies(BlockDriverState *bs, uint64_t guest_offset,
    uint64_t *cur_bytes, QCowL2Meta **m)
{
    BDRVQcow2State *s = bs->opaque;
    QCowL2Meta *old_alloc;
    uint64_t bytes = *cur_bytes;

    /*@s->cluster_allocs中存放的是所有正在进行中的且尚未更新L2 table的写请求*/
    QLIST_FOREACH(old_alloc, &s->cluster_allocs, next_in_flight) {
        uint64_t start = guest_offset;
        uint64_t end = start + bytes;
        /* 虽然qcow2中写对齐是由bs->bl.request_alignment控制的，不一定是cluster对齐的，
         * 但是因为写操作之后需要更新L2 table（cluster相关），所以这里需要对写的区间扩
         * 展到cluster对齐，以防止其它的后到来的与当前写请求存在交集的写请求先去更新
         * L2 table，[old_start, old_end]所在的区间就是扩展到cluster对齐后的区间，后续
         * 需要判断给定的[start, end]和[old_start, old_end]之间是否存在交叠以及交叠的
         * 情况，以确定对给定的[start, end]区间的裁减情况
         */
        uint64_t old_start = l2meta_cow_start(old_alloc);
        uint64_t old_end = l2meta_cow_end(old_alloc);

        /* 当前分配请求和@old_alloc之间没有交叠，则表明和@old_alloc之间没有依赖关
         * 系，继续检查[start, end]区间是否和下一个@old_alloc之间存在交叠
         */
        if (end <= old_start || start >= old_end) {
            /* No intersection */
        } else {
            if (start < old_start) {
                /* Stop at the start of a running allocation，当前分配请求中区间
                 * [start, old_start)和正在进行的写请求之间不存在依赖关系
                 */
                bytes = old_start - start;
            } else {
                /*当前分配请求的起始地址依赖于正在进行中的写请求*/
                bytes = 0;
            }

            /* Stop if already an l2meta exists. After yielding, it wouldn't
             * be valid any more, so we'd have to clean up the old L2Metas
             * and deal with requests depending on them before starting to
             * gather new ones. Not worth the trouble. 
             * 如果当前分配请求的起始地址依赖于正在进行中的其它写请求，且
             * @m不为空，则直接退出，交由上层调用处理，理解这里为什么这样做，
             * 需要结合qcow2_co_pwritev -> qcow2_alloc_cluster_offset ->
             * handle_dependencies调用链，以及这里的注释（如果@bytes = 0，即
             * 当前的写请求请求区间全部或者部分和正在进行的其它写请求之间存在
             * 依赖关系，当前写请求无法进行了，虽然@*m不为空，即已经分配了一些
             * cluster，但是由于当前请求无法进行了，直接退出，清理这些L2Metas）
             */
            if (bytes == 0 && *m) {
                *cur_bytes = 0;
                return 0;
            }

            if (bytes == 0) {
                /* Wait for the dependency to complete. We need to recheck
                 * the free/allocated clusters when we continue. 如果依赖于
                 * @old_alloc，则将之添加到@old_alloc->dependent_requests中，
                 * 等待@old_alloc相关的写请求完成后再被调度
                 */
                qemu_co_queue_wait(&old_alloc->dependent_requests, &s->lock);
                /* 直到当前请求所依赖的请求执行完毕唤醒当前协程的时候才会返回上层
                 * EAGAIN，重新来过
                 */
                return -EAGAIN;
            }
        }
    }

    /* Make sure that existing clusters and new allocations are only used up to
     * the next dependency if we shortened the request above */
    *cur_bytes = bytes;

    return 0;
}

```

```
/*
 * Checks how many already allocated clusters that don't require a copy on
 * write there are at the given guest_offset (up to *bytes). If
 * *host_offset is not zero, only physically contiguous clusters beginning at
 * this host offset are counted.
 *
 * Note that guest_offset may not be cluster aligned. In this case, the
 * returned *host_offset points to exact byte referenced by guest_offset and
 * therefore isn't cluster aligned as well.
 *
 * Returns:
 *   0:     if no allocated clusters are available at the given offset.
 *          *bytes is normally unchanged. It is set to 0 if the cluster
 *          is allocated and doesn't need COW, but doesn't have the right
 *          physical offset.
 *
 *   1:     if allocated clusters that don't require a COW are available at
 *          the requested offset. *bytes may have decreased and describes
 *          the length of the area that can be written to.
 *
 *  -errno: in error cases
 *
 * 该函数返回：
 * 0：如果@guest_offset对应的cluster不是QCOW2_CLUSTER_NORMAL类型的；
 *    或者不存在起始于@guest_offset的无需执行COW的cluster；
 *    或者存在起始于@guest_offset且无需执行COW的cluster，但是
 *    @host_offset不匹配
 *
 * 1：如果@guest_offset对应的cluster已经分配，且是QCOW2_CLUSTER_NORMAL
 *    类型，且无需执行COW，且存在至少一个起始于@guest_offset的cluster，
 *
 * -errno：发生错误
 */
static int handle_copied(BlockDriverState *bs, uint64_t guest_offset,
    uint64_t *host_offset, uint64_t *bytes, QCowL2Meta **m)
{
    BDRVQcow2State *s = bs->opaque;
    int l2_index;
    uint64_t cluster_offset;
    uint64_t *l2_table;
    uint64_t nb_clusters;
    unsigned int keep_clusters;
    int ret;

    trace_qcow2_handle_copied(qemu_coroutine_self(), guest_offset, *host_offset,
                              *bytes);

    /* 要么@*host_offset为0，要么@guest_offset和@*host_offset在cluster内部具有
     * 相同的偏移*/
    assert(*host_offset == 0 ||    offset_into_cluster(s, guest_offset)
                                == offset_into_cluster(s, *host_offset));

    /*
     * Calculate the number of clusters to look for. We stop at L2 table
     * boundaries to keep things simple.
     * 计算[guest_offset, guest_offset + *bytes)的区域涉及几个cluster
     */
    nb_clusters =
        size_to_clusters(s, offset_into_cluster(s, guest_offset) + *bytes);

    /*@l2_index表示@guest_offset所在的cluster在L2 table中的索引号*/
    l2_index = offset_to_l2_index(s, guest_offset);
    nb_clusters = MIN(nb_clusters, s->l2_size - l2_index);
    assert(nb_clusters <= INT_MAX);

    /* Find L2 entry for the first involved cluster 
     * 找到@guest_offset所在的cluster对应的@l2_table和@l2_index，其中@l2_table
     * 可能是新分配的（如果不存在的话），见get_cluster_table
     */
    ret = get_cluster_table(bs, guest_offset, &l2_table, &l2_index);
    if (ret < 0) {
        return ret;
    }

    /*L2 table中记录的就是@guest_offset对应的@cluster_offset*/
    cluster_offset = be64_to_cpu(l2_table[l2_index]);

    /* Check how many clusters are already allocated and don't need COW */
    /* 如果@cluster_offset所在的cluster是QCOW2_CLUSTER_NORMAL类型的，表明其已经分配，
     * 如果@cluster_offset中设置了QCOW_OFLAG_COPIED标识，表明其引用计数为1，没有任何
     * 的snapshot在引用它，可以直接写入，无需执行COW
     */
    if (qcow2_get_cluster_type(cluster_offset) == QCOW2_CLUSTER_NORMAL
        && (cluster_offset & QCOW_OFLAG_COPIED))
    {
        /* If a specific host_offset is required, check it */
        /* @cluster_offset一定是对齐到cluster的，(cluster_offset & L2E_OFFSET_MASK)
         * 结果一定为0，所以@offset_matches只有在@*host_offset为0的情况下为true
         */
        bool offset_matches =
            (cluster_offset & L2E_OFFSET_MASK) == *host_offset;

        /*检查@cluster_offset一定是对齐到cluster的*/
        if (offset_into_cluster(s, cluster_offset & L2E_OFFSET_MASK)) {
            qcow2_signal_corruption(bs, true, -1, -1, "Data cluster offset "
                                    "%#llx unaligned (guest offset: %#" PRIx64
                                    ")", cluster_offset & L2E_OFFSET_MASK,
                                    guest_offset);
            ret = -EIO;
            goto out;
        }

        /*无需COW，但是*host_offset不正确，则设置@*bytes为0并退出*/
        if (*host_offset != 0 && !offset_matches) {
            *bytes = 0;
            ret = 0;
            goto out;
        }

        /* We keep all QCOW_OFLAG_COPIED clusters */
        /* 在@l2_table中从@l2_index开始查找至多@nb_clusters个最大连续cluster
         * （这些cluster必须满足其在@l2_table中有关于它的记录，且@l2_table中
         * 记录的它的地址中的标志位和@l2_index所代表的cluster在在@l2_table中
         * 记录的地址中的标志位一模一样），这里是找所有设置了QCOW_OFLAG_COPIED
         * 标识的连续cluster，返回连续个数
         */
        keep_clusters =
            count_contiguous_clusters(nb_clusters, s->cluster_size,
                                      &l2_table[l2_index],
                                      QCOW_OFLAG_COPIED | QCOW_OFLAG_ZERO);
        assert(keep_clusters <= nb_clusters);

        *bytes = MIN(*bytes,
                 keep_clusters * s->cluster_size
                 - offset_into_cluster(s, guest_offset));

        ret = 1;
    } else {
        /*@l2_index对应的cluster尚未分配，或者分配了但是需要执行COW*/
        ret = 0;
    }

    /* Cleanup */
out:
    qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

    /* Only return a host offset if we actually made progress. Otherwise we
     * would make requirements for handle_alloc() that it can't fulfill */
    /*本次handle_copied有所进展，则更新@*host_offset*/
    if (ret > 0) {
        *host_offset = (cluster_offset & L2E_OFFSET_MASK)
                     + offset_into_cluster(s, guest_offset);
    }

    return ret;
}
```

```
/*
 * get_cluster_table
 *
 * for a given disk offset, load (and allocate if needed)
 * the l2 table.
 *
 * the l2 table offset in the qcow2 file and the cluster index
 * in the l2 table are given to the caller.
 *
 * Returns 0 on success, -errno in failure case
 */
static int get_cluster_table(BlockDriverState *bs, uint64_t offset,
                             uint64_t **new_l2_table,
                             int *new_l2_index)
{
    BDRVQcow2State *s = bs->opaque;
    unsigned int l2_index;
    uint64_t l1_index, l2_offset;
    uint64_t *l2_table = NULL;
    int ret;

    /* seek to the l2 offset in the l1 table 
     * @s->cluster_bits表示用多少个bit位来表示一个cluster的大小
     * @s->l2_bits表示一个L2 table中有多少个entry
     * @l1_index 表示@offset所在的cluster在L1 table中的索引号
     * 如果l1 index超过了L1 table当前的大小，则增长L1 table*/
    l1_index = offset >> (s->l2_bits + s->cluster_bits);
    if (l1_index >= s->l1_size) {
        /*至少增大到@l1_index + 1 这么大*/
        ret = qcow2_grow_l1_table(bs, l1_index + 1, false);
        if (ret < 0) {
            return ret;
        }
    }

    assert(l1_index < s->l1_size);
    /*获取L1 table中记录的关于该L2 table的地址，改地址必须是对齐到cluster的*/
    l2_offset = s->l1_table[l1_index] & L1E_OFFSET_MASK;
    if (offset_into_cluster(s, l2_offset)) {
        qcow2_signal_corruption(bs, true, -1, -1, "L2 table offset %#" PRIx64
                                " unaligned (L1 index: %#" PRIx64 ")",
                                l2_offset, l1_index);
        return -EIO;
    }

    /* seek the l2 table of the given l2 offset */

    if (s->l1_table[l1_index] & QCOW_OFLAG_COPIED) {
        /* load the l2 table in memory */
        /*直接load L2 table到内存中，要么从cache中加载，要么从镜像文件中加载*/
        ret = l2_load(bs, l2_offset, &l2_table);
        if (ret < 0) {
            return ret;
        }
    } else {
        /* First allocate a new L2 table (and do COW if needed) */
        /* 分配一个新的L2 table，如果@l1_index指向一片已经使用的L2 table，则将
         * 该旧的L2 table中的内容拷贝到新的L2 table中
         */
        ret = l2_allocate(bs, l1_index, &l2_table);
        if (ret < 0) {
            return ret;
        }

        /* Then decrease the refcount of the old table */
        if (l2_offset) {
            qcow2_free_clusters(bs, l2_offset, s->l2_size * sizeof(uint64_t),
                                QCOW2_DISCARD_OTHER);
        }
    }

    /* find the cluster offset for the given disk offset */
    l2_index = offset_to_l2_index(s, offset);

    *new_l2_table = l2_table;
    *new_l2_index = l2_index;

    return 0;
}
```

```
int qcow2_grow_l1_table(BlockDriverState *bs, uint64_t min_size,
                        bool exact_size)
{
    BDRVQcow2State *s = bs->opaque;
    int new_l1_size2, ret, i;
    uint64_t *new_l1_table;
    int64_t old_l1_table_offset, old_l1_size;
    int64_t new_l1_table_offset, new_l1_size;
    uint8_t data[12];

    if (min_size <= s->l1_size)
        return 0;

    /* Do a sanity check on min_size before trying to calculate new_l1_size
     * (this prevents overflows during the while loop for the calculation of
     * new_l1_size) */
    if (min_size > INT_MAX / sizeof(uint64_t)) {
        return -EFBIG;
    }

    /* 在get_cluster_table -> qcow2_grow_l1_table调用中，@exact_size设置为false
     * 表示不是必须增大到指定的大小*/
    if (exact_size) {
        new_l1_size = min_size;
    } else {
        /* Bump size up to reduce the number of times we have to grow 
         * @s->l1_size表示当前l1 table中可以容纳的entry的数目
         */
        new_l1_size = s->l1_size;
        if (new_l1_size == 0) {
            new_l1_size = 1;
        }
        /*在没有增大到@min_size之前，每次增大1.5倍*/
        while (min_size > new_l1_size) {
            new_l1_size = DIV_ROUND_UP(new_l1_size * 3, 2);
        }
    }

    QEMU_BUILD_BUG_ON(QCOW_MAX_L1_SIZE > INT_MAX);
    if (new_l1_size > QCOW_MAX_L1_SIZE / sizeof(uint64_t)) {
        return -EFBIG;
    }

#ifdef DEBUG_ALLOC2
    fprintf(stderr, "grow l1_table from %d to %" PRId64 "\n",
            s->l1_size, new_l1_size);
#endif

    /* 在内存中分配大小为@new_l1_size2的区域，用于拷贝当前L1 table中
     * 的数据（为将L1 table写入新的位置做准备，分配的内存是512字节
     * 对齐的）*/
    new_l1_size2 = sizeof(uint64_t) * new_l1_size;
    new_l1_table = qemu_try_blockalign(bs->file->bs,
                                       align_offset(new_l1_size2, 512));
    if (new_l1_table == NULL) {
        return -ENOMEM;
    }
    memset(new_l1_table, 0, align_offset(new_l1_size2, 512));

    if (s->l1_size) {
        /*从旧的L1 table中拷贝到新的L1 table中*/
        memcpy(new_l1_table, s->l1_table, s->l1_size * sizeof(uint64_t));
    }

    /* write new table (align to cluster) */
    BLKDBG_EVENT(bs->file, BLKDBG_L1_GROW_ALLOC_TABLE);
    /*在镜像文件中分配新的L1 table*/
    new_l1_table_offset = qcow2_alloc_clusters(bs, new_l1_size2);
    if (new_l1_table_offset < 0) {
        qemu_vfree(new_l1_table);
        return new_l1_table_offset;
    }

    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    if (ret < 0) {
        goto fail;
    }

    /* the L1 position has not yet been updated, so these clusters must
     * indeed be completely free */
    ret = qcow2_pre_write_overlap_check(bs, 0, new_l1_table_offset,
                                        new_l1_size2);
    if (ret < 0) {
        goto fail;
    }

    BLKDBG_EVENT(bs->file, BLKDBG_L1_GROW_WRITE_TABLE);
    for(i = 0; i < s->l1_size; i++)
        new_l1_table[i] = cpu_to_be64(new_l1_table[i]);
    ret = bdrv_pwrite_sync(bs->file, new_l1_table_offset,
                           new_l1_table, new_l1_size2);
    if (ret < 0)
        goto fail;
    for(i = 0; i < s->l1_size; i++)
        new_l1_table[i] = be64_to_cpu(new_l1_table[i]);

    /* set new table */
    BLKDBG_EVENT(bs->file, BLKDBG_L1_GROW_ACTIVATE_TABLE);
    stl_be_p(data, new_l1_size);
    stq_be_p(data + 4, new_l1_table_offset);
    ret = bdrv_pwrite_sync(bs->file, offsetof(QCowHeader, l1_size),
                           data, sizeof(data));
    if (ret < 0) {
        goto fail;
    }
    qemu_vfree(s->l1_table);
    old_l1_table_offset = s->l1_table_offset;
    s->l1_table_offset = new_l1_table_offset;
    s->l1_table = new_l1_table;
    old_l1_size = s->l1_size;
    s->l1_size = new_l1_size;
    qcow2_free_clusters(bs, old_l1_table_offset, old_l1_size * sizeof(uint64_t),
                        QCOW2_DISCARD_OTHER);
    return 0;
 fail:
    qemu_vfree(new_l1_table);
    qcow2_free_clusters(bs, new_l1_table_offset, new_l1_size2,
                        QCOW2_DISCARD_OTHER);
    return ret;
}
```

```
int64_t qcow2_alloc_clusters(BlockDriverState *bs, uint64_t size)
{
    int64_t offset;
    int ret;

    BLKDBG_EVENT(bs->file, BLKDBG_CLUSTER_ALLOC);
    do {
        /* 最终分配的是将@size向上对齐到cluster大小之后的大小（整数个cluster），
         * 返回分配到的连续cluster中的起始cluster偏移
         */
        offset = alloc_clusters_noref(bs, size);
        if (offset < 0) {
            return offset;
        }

        ret = update_refcount(bs, offset, size, 1, false, QCOW2_DISCARD_NEVER);
    } while (ret == -EAGAIN);

    if (ret < 0) {
        return ret;
    }

    return offset;
}
```

```
static int64_t alloc_clusters_noref(BlockDriverState *bs, uint64_t size)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t i, nb_clusters, refcount;
    int ret;

    /* We can't allocate clusters if they may still be queued for discard. 
       discard相关，暂时不关注*/
    if (s->cache_discards) {
        qcow2_process_discards(bs, 0);
    }

    /*向上对齐@size到cluster边界，计算该@size涉及到多少cluster*/
    nb_clusters = size_to_clusters(s, size);
    
retry:
    for(i = 0; i < nb_clusters; i++) {
        **/*注意这里每次都会更新@s->free_cluster_index*/**
        uint64_t next_cluster_index = s->free_cluster_index++;
        ret = qcow2_get_refcount(bs, next_cluster_index, &refcount);

        if (ret < 0) {
            /*出错，返回*/
            return ret;
        } else if (refcount != 0) {
            /*refcount 不为0，则不能被分配，进入@retry 重试*/
            goto retry;
        }
    }

    /* Make sure that all offsets in the "allocated" range are representable
     * in an int64_t */
    if (s->free_cluster_index > 0 &&
        s->free_cluster_index - 1 > (INT64_MAX >> s->cluster_bits))
    {
        return -EFBIG;
    }

#ifdef DEBUG_ALLOC2
    fprintf(stderr, "alloc_clusters: size=%" PRId64 " -> %" PRId64 "\n",
            size,
            (s->free_cluster_index - nb_clusters) << s->cluster_bits);
#endif

    /*返回分配的起始cluster的偏移*/
    return (s->free_cluster_index - nb_clusters) << s->cluster_bits;
}
```

```
/*
 * Retrieves the refcount of the cluster given by its index and stores it in
 * *refcount. Returns 0 on success and -errno on failure.
 */
int qcow2_get_refcount(BlockDriverState *bs, int64_t cluster_index,
                       uint64_t *refcount)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t refcount_table_index, block_index;
    int64_t refcount_block_offset;
    int ret;
    void *refcount_block;

    /*@cluster_index对应的cluster在refcount table中的索引号*/
    refcount_table_index = cluster_index >> s->refcount_block_bits;
    if (refcount_table_index >= s->refcount_table_size) {
        /*@refcount_table_index 超出了当前refcount table的总大小*/
        *refcount = 0;
        return 0;
    }
    
    /*@cluster_index对应的cluster相关的refcount block的起始地址*/
    refcount_block_offset =
        s->refcount_table[refcount_table_index] & REFT_OFFSET_MASK;
    
    /*@cluster_index所代表的cluster的refcount block地址为0*/
    if (!refcount_block_offset) {
        *refcount = 0;
        return 0;
    }

    /*@refcount_block_offset必须是对齐到cluster的*/
    if (offset_into_cluster(s, refcount_block_offset)) {
        qcow2_signal_corruption(bs, true, -1, -1, "Refblock offset %#" PRIx64
                                " unaligned (reftable index: %#" PRIx64 ")",
                                refcount_block_offset, refcount_table_index);
        return -EIO;
    }

    /*从@refcount_block_cache查询@refcount_block_offset所在的@refcount_block*/
    ret = qcow2_cache_get(bs, s->refcount_block_cache, refcount_block_offset,
                          &refcount_block);
    if (ret < 0) {
        return ret;
    }

    /*计算@cluster_index所代表的cluster在refcount block中的索引号，并获取引用计数*/
    block_index = cluster_index & (s->refcount_block_size - 1);
    *refcount = s->get_refcount(refcount_block, block_index);

    qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);

    return 0;
}
```

```
/* XXX: cache several refcount block clusters ? */
/* @addend is the absolute value of the addend; if @decrease is set, @addend
 * will be subtracted from the current refcount, otherwise it will be added */
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

#ifdef DEBUG_ALLOC2
    fprintf(stderr, "update_refcount: offset=%" PRId64 " size=%" PRId64
            " addend=%s%" PRIu64 "\n", offset, length, decrease ? "-" : "",
            addend);
#endif
    if (length < 0) {
        return -EINVAL;
    } else if (length == 0) {
        return 0;
    }

    if (decrease) {
        qcow2_cache_set_dependency(bs, s->refcount_block_cache,
            s->l2_table_cache);
    }

    /*依次处理[start, last]范围内的每一个cluster*/
    start = start_of_cluster(s, offset);
    last = start_of_cluster(s, offset + length - 1);
    for(cluster_offset = start; cluster_offset <= last;
        cluster_offset += s->cluster_size)
    {
        int block_index;
        uint64_t refcount;
        int64_t cluster_index = cluster_offset >> s->cluster_bits;
        int64_t table_index = cluster_index >> s->refcount_block_bits;

        /* Load the refcount block and allocate it if needed */
        if (table_index != old_table_index) {
            /* 如果当前cluster和上一个cluster不是率属于同一个refcount table，
             * 则加载关于@table_index的新的@refcount_block（当然事先要对
             * @old_table_index相关的@refcount_block执行qcow2_cache_put，
             * 因为在alloc_refcount_block -> load_refcount_block调用链中会
             * 调用qcow2_cache_get）
             */
            if (refcount_block) {
                qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
            }
            
            /* 这里名称为alloc_refcount_block，实际上并不一定会执行alloc，
             * 也有可能只是load而已，见alloc_refcount_block
             */
            ret = alloc_refcount_block(bs, cluster_index, &refcount_block);
            if (ret < 0) {
                goto fail;
            }
        }
        old_table_index = table_index;

        /*标记@refcount_block为脏*/
        qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache,
                                     refcount_block);

        /* we can update the count and save it */
        block_index = cluster_index & (s->refcount_block_size - 1);

        /*更新引用计数*/
        refcount = s->get_refcount(refcount_block, block_index);
        if (decrease ? (refcount - addend > refcount)
                     : (refcount + addend < refcount ||
                        refcount + addend > s->refcount_max))
        {
            ret = -EINVAL;
            goto fail;
        }
        if (decrease) {
            refcount -= addend;
        } else {
            refcount += addend;
        }
        if (refcount == 0 && cluster_index < s->free_cluster_index) {
            s->free_cluster_index = cluster_index;
        }
        s->set_refcount(refcount_block, block_index, refcount);

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
    if (refcount_block) {
        qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
    }

    /*
     * Try do undo any updates if an error is returned (This may succeed in
     * some cases like ENOSPC for allocating a new refcount block)
     */
    /*失败的情况下撤销引用计数更新*/
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
/*
 * Loads a refcount block. If it doesn't exist yet, it is allocated first
 * (including growing the refcount table if needed).
 *
 * Returns 0 on success or -errno in error case
 */
static int alloc_refcount_block(BlockDriverState *bs,
                                int64_t cluster_index, void **refcount_block)
{
    BDRVQcow2State *s = bs->opaque;
    unsigned int refcount_table_index;
    int64_t ret;

    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC);

    /* Find the refcount block for the given cluster */
    refcount_table_index = cluster_index >> s->refcount_block_bits;

    if (refcount_table_index < s->refcount_table_size) {

        uint64_t refcount_block_offset =
            s->refcount_table[refcount_table_index] & REFT_OFFSET_MASK;

        /* If it's already there, we're done*/
        /*refcount block所在的地址不为0，表明@cluster_index相关的refcount block已经存在*/
        if (refcount_block_offset) {
            /*refcount block的起始偏移必须是对齐到cluster的，所以offset_into_cluster必须是0*/
            if (offset_into_cluster(s, refcount_block_offset)) {
                qcow2_signal_corruption(bs, true, -1, -1, "Refblock offset %#"
                                        PRIx64 " unaligned (reftable index: "
                                        "%#x)", refcount_block_offset,
                                        refcount_table_index);
                return -EIO;
            }

            /*直接从refcount block cache中加载*/
            return load_refcount_block(bs, refcount_block_offset,
                                        refcount_block);
        }
    }

    /*
     * If we came here, we need to allocate something. Something is at least
     * a cluster for the new refcount block. It may also include a new refcount
     * table if the old refcount table is too small.
     *
     * Note that allocating clusters here needs some special care:
     *
     * - We can't use the normal qcow2_alloc_clusters(), it would try to
     *   increase the refcount and very likely we would end up with an endless
     *   recursion. Instead we must place the refcount blocks in a way that
     *   they can describe them themselves.
     *
     * - We need to consider that at this point we are inside update_refcounts
     *   and potentially doing an initial refcount increase. This means that
     *   some clusters have already been allocated by the caller, but their
     *   refcount isn't accurate yet. If we allocate clusters for metadata, we
     *   need to return -EAGAIN to signal the caller that it needs to restart
     *   the search for free clusters.
     *
     * - alloc_clusters_noref and qcow2_free_clusters may load a different
     *   refcount block into the cache
     */

    /*至此，我们至少需要分配一个新的refcount block，甚至还需要分配一个新的refcount table*/
    *refcount_block = NULL;

    /* We write to the refcount table, so we might depend on L2 tables */
    ret = qcow2_cache_flush(bs, s->l2_table_cache);
    if (ret < 0) {
        return ret;
    }

    /* Allocate the refcount block itself and mark it as used， 分配一个新的
     * refcount block，返回该refcount block的偏移
     */
    int64_t new_block = alloc_clusters_noref(bs, s->cluster_size);
    if (new_block < 0) {
        return new_block;
    }

#ifdef DEBUG_ALLOC2
    fprintf(stderr, "qcow2: Allocate refcount block %d for %" PRIx64
        " at %" PRIx64 "\n",
        refcount_table_index, cluster_index << s->cluster_bits, new_block);
#endif

    if (in_same_refcount_block(s, new_block, cluster_index << s->cluster_bits)) {
        /* Zero the new refcount block before updating it */
        /* @cluster_index代表那个需要更新引用计数的cluster，而@new_block代表
         * 的是为@cluster_index分配的refcount block，两者指向同一个cluster，
         * 则说明该cluster描述它自身的引用计数，从refcount block cache中分配
         * 一个新的@refcount_block用于缓存该refcount block
         */
        ret = qcow2_cache_get_empty(bs, s->refcount_block_cache, new_block,
                                    refcount_block);
        if (ret < 0) {
            goto fail;
        }

        /*将该refcount block置为0*/
        memset(*refcount_block, 0, s->cluster_size);

        /* The block describes itself, need to update the cache */
        /*将该refcount block中关于它自身的引用计数设置为1*/
        int block_index = (new_block >> s->cluster_bits) &
            (s->refcount_block_size - 1);
        s->set_refcount(*refcount_block, block_index, 1);
    } else {
        /* Described somewhere else. This can recurse at most twice before we
         * arrive at a block that describes itself. */
        /* 更新新分配的refcount block（通过区间[new_block, new_block + s->cluster_size)
         * 表示）的引用计数，这可能导致级联的分配新的refcount block
         */
        ret = update_refcount(bs, new_block, s->cluster_size, 1, false,
                              QCOW2_DISCARD_NEVER);
        if (ret < 0) {
            goto fail;
        }

        ret = qcow2_cache_flush(bs, s->refcount_block_cache);
        if (ret < 0) {
            goto fail;
        }

        /*为该新分配的refcount block在refcount block cache中分配一块空间，并初始化为全0*/
        /* Initialize the new refcount block only after updating its refcount,
         * update_refcount uses the refcount cache itself */
        ret = qcow2_cache_get_empty(bs, s->refcount_block_cache, new_block,
                                    refcount_block);
        if (ret < 0) {
            goto fail;
        }

        memset(*refcount_block, 0, s->cluster_size);
    }

    /*标记该@refcount_block为脏，写入并flush到镜像文件*/
    /* Now the new refcount block needs to be written to disk */
    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC_WRITE);
    qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache, *refcount_block);
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    if (ret < 0) {
        goto fail;
    }

    /* @refcount_table_index并未超出当前的refcount table范围，则将该新分配的
     * refcount block的地址(@new_block所表示)写入refcount table中
     */
    /* If the refcount table is big enough, just hook the block up there */
    if (refcount_table_index < s->refcount_table_size) {
        uint64_t data64 = cpu_to_be64(new_block);
        BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC_HOOKUP);
        ret = bdrv_pwrite_sync(bs->file,
            s->refcount_table_offset + refcount_table_index * sizeof(uint64_t),
            &data64, sizeof(data64));
        if (ret < 0) {
            goto fail;
        }

        s->refcount_table[refcount_table_index] = new_block;
        /* If there's a hole in s->refcount_table then it can happen
         * that refcount_table_index < s->max_refcount_table_index */
        /*更新@s->max_refcount_table_index*/
        s->max_refcount_table_index =
            MAX(s->max_refcount_table_index, refcount_table_index);

        /* The new refcount block may be where the caller intended to put its
         * data, so let it restart the search. */
        /* 这里返回EAGAIN，详见前面的注释“we need to consider that at this point
         * we  are inside update_refcounts and potentially ......”
         */
        return -EAGAIN;
    }

    qcow2_cache_put(bs, s->refcount_block_cache, refcount_block);
    
    /*
     * If we come here, we need to grow the refcount table. Again, a new
     * refcount table needs some space and we can't simply allocate to avoid
     * endless recursion.
     *
     * Therefore let's grab new refcount blocks at the end of the image, which
     * will describe themselves and the new refcount table. This way we can
     * reference them only in the new table and do the switch to the new
     * refcount table at once without producing an inconsistent state in
     * between.
     */
    BLKDBG_EVENT(bs->file, BLKDBG_REFTABLE_GROW);

    /* Calculate the number of refcount blocks needed so far; this will be the
     * basis for calculating the index of the first cluster used for the
     * self-describing refcount structures which we are about to create.
     *
     * Because we reached this point, there cannot be any refcount entries for
     * cluster_index or higher indices yet. However, because new_block has been
     * allocated to describe that cluster (and it will assume this role later
     * on), we cannot use that index; also, new_block may actually have a higher
     * cluster index than cluster_index, so it needs to be taken into account
     * here (and 1 needs to be added to its value because that cluster is used).
     */
     
    /* 至此，需要扩大refcount table并分配新的refcount table，首先计算出当前已经使
     * 用的最大的cluster数目，然后，在该最大的cluster之后创建新的refcount table和
     * refcount block，并将就得refcount table和refcount block中的内容拷贝到新的
     * refcount table和refcount block中
     */
    uint64_t blocks_used = DIV_ROUND_UP(MAX(cluster_index + 1,
                                            (new_block >> s->cluster_bits) + 1),
                                        s->refcount_block_size);

    /* Create the new refcount table and blocks */
    uint64_t meta_offset = (blocks_used * s->refcount_block_size) *
        s->cluster_size;

    ret = qcow2_refcount_area(bs, meta_offset, 0, false,
                              refcount_table_index, new_block);
    if (ret < 0) {
        return ret;
    }

    /*加载关于@cluster_index的refcount block到@refcount_block中*/
    ret = load_refcount_block(bs, new_block, refcount_block);
    if (ret < 0) {
        return ret;
    }

    /* If we were trying to do the initial refcount update for some cluster
     * allocation, we might have used the same clusters to store newly
     * allocated metadata. Make the caller search some new space. */
    return -EAGAIN;

fail:
    if (*refcount_block != NULL) {
        qcow2_cache_put(bs, s->refcount_block_cache, refcount_block);
    }
    return ret;
}
```

```
static int load_refcount_block(BlockDriverState *bs,
                               int64_t refcount_block_offset,
                               void **refcount_block)
{
    BDRVQcow2State *s = bs->opaque;

    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_LOAD);
    return qcow2_cache_get(bs, s->refcount_block_cache, refcount_block_offset,
                           refcount_block);
}
```

```
/*
 * Starting at @start_offset, this function creates new self-covering refcount
 * structures: A new refcount table and refcount blocks which cover all of
 * themselves, and a number of @additional_clusters beyond their end.
 * @start_offset must be at the end of the image file, that is, there must be
 * only empty space beyond it.
 * If @exact_size is false, the refcount table will have 50 % more entries than
 * necessary so it will not need to grow again soon.
 * If @new_refblock_offset is not zero, it contains the offset of a refcount
 * block that should be entered into the new refcount table at index
 * @new_refblock_index.
 *
 * Returns: The offset after the new refcount structures (i.e. where the
 *          @additional_clusters may be placed) on success, -errno on error.
 */
int64_t qcow2_refcount_area(BlockDriverState *bs, uint64_t start_offset,
                            uint64_t additional_clusters, bool exact_size,
                            int new_refblock_index,
                            uint64_t new_refblock_offset)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t total_refblock_count_u64, additional_refblock_count;
    int total_refblock_count, table_size, area_reftable_index, table_clusters;
    int i;
    uint64_t table_offset, block_offset, end_offset;
    int ret;
    uint64_t *new_table;

    assert(!(start_offset % s->cluster_size));

    /*计算refcount table和refcount block所需的字节数，以及总的refcount block数目*/
    qcow2_refcount_metadata_size(start_offset / s->cluster_size +
                                 additional_clusters,
                                 s->cluster_size, s->refcount_order,
                                 !exact_size, &total_refblock_count_u64);
    if (total_refblock_count_u64 > QCOW_MAX_REFTABLE_SIZE) {
        return -EFBIG;
    }
    
    total_refblock_count = total_refblock_count_u64;

    /* Index in the refcount table of the first refcount block to cover the area
     * of refcount structures we are about to create; we know that
     * @total_refblock_count can cover @start_offset, so this will definitely
     * fit into an int. */
    area_reftable_index = (start_offset / s->cluster_size) /
                          s->refcount_block_size;

    if (exact_size) {
        table_size = total_refblock_count;
    } else {
        table_size = total_refblock_count +
                     DIV_ROUND_UP(total_refblock_count, 2);
    }
    /* The qcow2 file can only store the reftable size in number of clusters */
    table_size = ROUND_UP(table_size, s->cluster_size / sizeof(uint64_t));
    table_clusters = (table_size * sizeof(uint64_t)) / s->cluster_size;

    if (table_size > QCOW_MAX_REFTABLE_SIZE) {
        return -EFBIG;
    }

    new_table = g_try_new0(uint64_t, table_size);

    assert(table_size > 0);
    if (new_table == NULL) {
        ret = -ENOMEM;
        goto fail;
    }

    /* Fill the new refcount table */
    if (table_size > s->max_refcount_table_index) {
        /* We're actually growing the reftable */
        /*扩大refcount table之后将就得refcount table中的内容拷贝过来*/
        memcpy(new_table, s->refcount_table,
               (s->max_refcount_table_index + 1) * sizeof(uint64_t));
    } else {
        /* Improbable case: We're shrinking the reftable. However, the caller
         * has assured us that there is only empty space beyond @start_offset,
         * so we can simply drop all of the refblocks that won't fit into the
         * new reftable. */
        memcpy(new_table, s->refcount_table, table_size * sizeof(uint64_t));
    }

    if (new_refblock_offset) {
        assert(new_refblock_index < total_refblock_count);
        new_table[new_refblock_index] = new_refblock_offset;
    }

    /* Count how many new refblocks we have to create */
    additional_refblock_count = 0;
    for (i = area_reftable_index; i < total_refblock_count; i++) {
        if (!new_table[i]) {
            additional_refblock_count++;
        }
    }

    table_offset = start_offset + additional_refblock_count * s->cluster_size;
    end_offset = table_offset + table_clusters * s->cluster_size;

    /* Fill the refcount blocks, and create new ones, if necessary */
    block_offset = start_offset;
    for (i = area_reftable_index; i < total_refblock_count; i++) {
        void *refblock_data;
        uint64_t first_offset_covered;

        /* Reuse an existing refblock if possible, create a new one otherwise */
        if (new_table[i]) {
            ret = qcow2_cache_get(bs, s->refcount_block_cache, new_table[i],
                                  &refblock_data);
            if (ret < 0) {
                goto fail;
            }
        } else {
            ret = qcow2_cache_get_empty(bs, s->refcount_block_cache,
                                        block_offset, &refblock_data);
            if (ret < 0) {
                goto fail;
            }
            memset(refblock_data, 0, s->cluster_size);
            qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache,
                                         refblock_data);

            new_table[i] = block_offset;
            block_offset += s->cluster_size;
        }

        /* First host offset covered by this refblock */
        first_offset_covered = (uint64_t)i * s->refcount_block_size *
                               s->cluster_size;
        if (first_offset_covered < end_offset) {
            int j, end_index;

            /* Set the refcount of all of the new refcount structures to 1 */

            if (first_offset_covered < start_offset) {
                assert(i == area_reftable_index);
                j = (start_offset - first_offset_covered) / s->cluster_size;
                assert(j < s->refcount_block_size);
            } else {
                j = 0;
            }

            end_index = MIN((end_offset - first_offset_covered) /
                            s->cluster_size,
                            s->refcount_block_size);

            for (; j < end_index; j++) {
                /* The caller guaranteed us this space would be empty */
                assert(s->get_refcount(refblock_data, j) == 0);
                s->set_refcount(refblock_data, j, 1);
            }

            qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache,
                                         refblock_data);
        }

        qcow2_cache_put(bs, s->refcount_block_cache, &refblock_data);
    }

    assert(block_offset == table_offset);

    /* Write refcount blocks to disk */
    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC_WRITE_BLOCKS);
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    if (ret < 0) {
        goto fail;
    }

    /* Write refcount table to disk */
    for (i = 0; i < total_refblock_count; i++) {
        cpu_to_be64s(&new_table[i]);
    }

    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC_WRITE_TABLE);
    ret = bdrv_pwrite_sync(bs->file, table_offset, new_table,
        table_size * sizeof(uint64_t));
    if (ret < 0) {
        goto fail;
    }

    for (i = 0; i < total_refblock_count; i++) {
        be64_to_cpus(&new_table[i]);
    }

    /* Hook up the new refcount table in the qcow2 header */
    struct QEMU_PACKED {
        uint64_t d64;
        uint32_t d32;
    } data;
    data.d64 = cpu_to_be64(table_offset);
    data.d32 = cpu_to_be32(table_clusters);
    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC_SWITCH_TABLE);
    ret = bdrv_pwrite_sync(bs->file,
                           offsetof(QCowHeader, refcount_table_offset),
                           &data, sizeof(data));
    if (ret < 0) {
        goto fail;
    }

    /* And switch it in memory */
    uint64_t old_table_offset = s->refcount_table_offset;
    uint64_t old_table_size = s->refcount_table_size;

    g_free(s->refcount_table);
    s->refcount_table = new_table;
    s->refcount_table_size = table_size;
    s->refcount_table_offset = table_offset;
    /*更新@s->max_refcount_table_index*/
    update_max_refcount_table_index(s);

    /* Free old table. */
    qcow2_free_clusters(bs, old_table_offset, old_table_size * sizeof(uint64_t),
                        QCOW2_DISCARD_OTHER);

    return end_offset;

fail:
    g_free(new_table);
    return ret;
}
```

```
static void update_max_refcount_table_index(BDRVQcow2State *s)
{
    unsigned i = s->refcount_table_size - 1;
    while (i > 0 && (s->refcount_table[i] & REFT_OFFSET_MASK) == 0) {
        i--;
    }
    /* Set s->max_refcount_table_index to the index of the last used entry */
    s->max_refcount_table_index = i;
}
```

```
/*
 * Allocates new clusters for an area that either is yet unallocated or needs a
 * copy on write. If *host_offset is non-zero, clusters are only allocated if
 * the new allocation can match the specified host offset.
 *
 * Note that guest_offset may not be cluster aligned. In this case, the
 * returned *host_offset points to exact byte referenced by guest_offset and
 * therefore isn't cluster aligned as well.
 *
 * Returns:
 *   0:     if no clusters could be allocated. *bytes is set to 0,
 *          *host_offset is left unchanged.
 *
 *   1:     if new clusters were allocated. *bytes may be decreased if the
 *          new allocation doesn't cover all of the requested area.
 *          *host_offset is updated to contain the host offset of the first
 *          newly allocated cluster.
 *
 *  -errno: in error cases
 */
static int handle_alloc(BlockDriverState *bs, uint64_t guest_offset,
    uint64_t *host_offset, uint64_t *bytes, QCowL2Meta **m)
{
    BDRVQcow2State *s = bs->opaque;
    int l2_index;
    uint64_t *l2_table;
    uint64_t entry;
    uint64_t nb_clusters;
    int ret;
    bool keep_old_clusters = false;

    uint64_t alloc_cluster_offset = 0;

    trace_qcow2_handle_alloc(qemu_coroutine_self(), guest_offset, *host_offset,
                             *bytes);
    assert(*bytes > 0);

    /*
     * Calculate the number of clusters to look for. We stop at L2 table
     * boundaries to keep things simple.
     */
    nb_clusters =
        size_to_clusters(s, offset_into_cluster(s, guest_offset) + *bytes);

    l2_index = offset_to_l2_index(s, guest_offset);
    nb_clusters = MIN(nb_clusters, s->l2_size - l2_index);
    assert(nb_clusters <= INT_MAX);

    /* Find L2 entry for the first involved cluster */
    ret = get_cluster_table(bs, guest_offset, &l2_table, &l2_index);
    if (ret < 0) {
        return ret;
    }

    entry = be64_to_cpu(l2_table[l2_index]);

    /* For the moment, overwrite compressed clusters one by one */
    if (entry & QCOW_OFLAG_COMPRESSED) {
        nb_clusters = 1;
    } else {
        /* 找出从@l2_index开始的至多@nb_clusters个cluster中连续的需要执行COW的cluster，
         * 这些cluster要么已经分配，但是需要执行COW，要么尚未分配，但是需要从backing
         * file中拷贝数据
         */
        nb_clusters = count_cow_clusters(s, nb_clusters, l2_table, l2_index);
    }

    /* This function is only called when there were no non-COW clusters, so if
     * we can't find any unallocated or COW clusters either, something is
     * wrong with our code. */
    assert(nb_clusters > 0);

    /* 如果@guest_offset所在的cluster类型为QCOW2_CLUSTER_ZERO_ALLOC，且可以直接写入（设
     * 置了QCOW_OFLAG_COPIED标识），且@guest_offset对应的cluster就是@*host_offset所在的
     * cluster，则尝试复用这些可以直接写入的zero clusters（设置了QCOW_OFLAG_COPIED标识）
     */
    if (qcow2_get_cluster_type(entry) == QCOW2_CLUSTER_ZERO_ALLOC &&
        (entry & QCOW_OFLAG_COPIED) &&
        (!*host_offset ||
         start_of_cluster(s, *host_offset) == (entry & L2E_OFFSET_MASK)))
    {
        /* Try to reuse preallocated zero clusters; contiguous normal clusters
         * would be fine, too, but count_cow_clusters() above has limited
         * nb_clusters already to a range of COW clusters */
        int preallocated_nb_clusters =
            count_contiguous_clusters(nb_clusters, s->cluster_size,
                                      &l2_table[l2_index], QCOW_OFLAG_COPIED);
        assert(preallocated_nb_clusters > 0);

        /*@nb_clusters表示可以直接写入的cluster的数目*/
        nb_clusters = preallocated_nb_clusters;
        alloc_cluster_offset = entry & L2E_OFFSET_MASK;

        /* We want to reuse these clusters, so qcow2_alloc_cluster_link_l2()
         * should not free them. */
        /* 这些cluster虽然是其它的写请求分配的，但是当前写请求要复用它们，设置
         * keep_old_clusters为true，以通知他们在qcow2_alloc_cluster_link_l2()
         * 中不要释放它们
         */
        keep_old_clusters = true;
    }

    qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

    if (!alloc_cluster_offset) {
        /*@alloc_cluster_offset为0，则表明不存在复用zero clusters的情况，只能自己分配了*/
        /* Allocate, if necessary at a given offset in the image file */
        alloc_cluster_offset = start_of_cluster(s, *host_offset);
        /* 为@guest_offset分配cluster，如果@alloc_cluster_offset不为0，则必须在镜像文件中
         * @alloc_cluster_offset所在的位置开始分配，否则可以在镜像文件中任何可用空间分配，
         * 至多分配@nb_clusters个连续的cluster，如果分配成功@alloc_cluster_offset中存放分
         * 配到的cluster在镜像文件中的起始地址
         */
        ret = do_alloc_cluster_offset(bs, guest_offset, &alloc_cluster_offset,
                                      &nb_clusters);
        if (ret < 0) {
            goto fail;
        }

        /* Can't extend contiguous allocation */
        /*当前分配请求无法满足*/
        if (nb_clusters == 0) {
            *bytes = 0;
            return 0;
        }

        /* !*host_offset would overwrite the image header and is reserved for
         * "no host offset preferred". If 0 was a valid host offset, it'd
         * trigger the following overlap check; do that now to avoid having an
         * invalid value in *host_offset. */
        /*检查分配到的起始地址是否会覆盖写元数据区域*/
        if (!alloc_cluster_offset) {
            ret = qcow2_pre_write_overlap_check(bs, 0, alloc_cluster_offset,
                                                nb_clusters * s->cluster_size);
            assert(ret < 0);
            goto fail;
        }
    }

    /* 因为分配cluster成功，在写完成之后，需要相应的在L2 table中更新这些cluster信息，
     * 先将这些L2 table更新相关的元信息保留在QCowL2Meta中，在qcow2_co_pwritev ->
     * qcow2_alloc_cluster_link_l2中会用到这些元数据来更新
     */
    /*
     * Save info needed for meta data update.
     *
     * requested_bytes: Number of bytes from the start of the first
     * newly allocated cluster to the end of the (possibly shortened
     * before) write request.
     *
     * avail_bytes: Number of bytes from the start of the first
     * newly allocated to the end of the last newly allocated cluster.
     *
     * nb_bytes: The number of bytes from the start of the first
     * newly allocated cluster to the end of the area that the write
     * request actually writes to (excluding COW at the end)
     */
    /*@requested_bytes、@avail_bytes和@nb_bytes的意义见上面的注释*/
    uint64_t requested_bytes = *bytes + offset_into_cluster(s, guest_offset);
    int avail_bytes = MIN(INT_MAX, nb_clusters << s->cluster_bits);
    int nb_bytes = MIN(requested_bytes, avail_bytes);
    QCowL2Meta *old_m = *m;

    /*新分配一个QCowL2Meta结构，用于存放L2 table元信息更新相关的信息*/
    *m = g_malloc0(sizeof(**m));

    /*设置QCowL2Meta*/
    **m = (QCowL2Meta) {
        /*和已有的QCowL2Meta链起来*/
        .next           = old_m,
        /*新分配的clusters的起始地址*/        
        .alloc_offset   = alloc_cluster_offset,
        /*新分配的第一个cluster对应的guest_offset*/
        .offset         = start_of_cluster(s, guest_offset),
        /*分配的所有clusters*/
        .nb_clusters    = nb_clusters,
        /*是否需要保留那些复用的cluster，只有在复用的情况下才为true*/
        .keep_old_clusters  = keep_old_clusters,
        /*新分配的第一个cluster中写请求不会涉及的区域*/
        .cow_start = {
            .offset     = 0,
            .nb_bytes   = offset_into_cluster(s, guest_offset),
        },
        /*新分配的最后一个cluster中写请求不会涉及的区域*/
        .cow_end = {
            .offset     = nb_bytes,
            .nb_bytes   = avail_bytes - nb_bytes,
        },
    };
    /*初始化dependent_requests链表，其中所有存放依赖于该写请求的其它写请求所在的协程*/
    qemu_co_queue_init(&(*m)->dependent_requests);
    /* 将@*m添加到@s->cluster_allocs中，即添加到尚未完成的写请求链表中，用于后续写请
     * 求依赖关系的判断
     */
    QLIST_INSERT_HEAD(&s->cluster_allocs, *m, next_in_flight);

    /*更新@*host_offset、@*bytes*/
    *host_offset = alloc_cluster_offset + offset_into_cluster(s, guest_offset);
    *bytes = MIN(*bytes, nb_bytes - offset_into_cluster(s, guest_offset));
    assert(*bytes != 0);

    return 1;

fail:
    if (*m && (*m)->nb_clusters > 0) {
        QLIST_REMOVE(*m, next_in_flight);
    }
    
    return ret;
}
```

```
/*
 * Returns the number of contiguous clusters that can be used for an allocating
 * write, but require COW to be performed (this includes yet unallocated space,
 * which must copy from the backing file)
 */
static int count_cow_clusters(BDRVQcow2State *s, int nb_clusters,
    uint64_t *l2_table, int l2_index)
{
    int i;

    for (i = 0; i < nb_clusters; i++) {
        uint64_t l2_entry = be64_to_cpu(l2_table[l2_index + i]);
        /*获取cluster的类型*/
        QCow2ClusterType cluster_type = qcow2_get_cluster_type(l2_entry);

        switch(cluster_type) {
        case QCOW2_CLUSTER_NORMAL:
            /*设置了QCOW_OFLAG_COPIED标识的cluster是无需执行COW的*/
            if (l2_entry & QCOW_OFLAG_COPIED) {
                goto out;
            }
            break;
        /*对于下列类型，则都认为需要执行COW*/
        case QCOW2_CLUSTER_UNALLOCATED:
        case QCOW2_CLUSTER_COMPRESSED:
        case QCOW2_CLUSTER_ZERO_PLAIN:
        case QCOW2_CLUSTER_ZERO_ALLOC:
            break;
        default:
            abort();
        }
    }

out:
    assert(i <= nb_clusters);
    return i;
}
```

```
/*
 * Allocates new clusters for the given guest_offset.
 *
 * At most *nb_clusters are allocated, and on return *nb_clusters is updated to
 * contain the number of clusters that have been allocated and are contiguous
 * in the image file.
 *
 * If *host_offset is non-zero, it specifies the offset in the image file at
 * which the new clusters must start. *nb_clusters can be 0 on return in this
 * case if the cluster at host_offset is already in use. If *host_offset is
 * zero, the clusters can be allocated anywhere in the image file.
 *
 * *host_offset is updated to contain the offset into the image file at which
 * the first allocated cluster starts.
 *
 * Return 0 on success and -errno in error cases. -EAGAIN means that the
 * function has been waiting for another request and the allocation must be
 * restarted, but the whole request should not be failed.
 */
static int do_alloc_cluster_offset(BlockDriverState *bs, uint64_t guest_offset,
                                   uint64_t *host_offset, uint64_t *nb_clusters)
{
    BDRVQcow2State *s = bs->opaque;

    trace_qcow2_do_alloc_clusters_offset(qemu_coroutine_self(), guest_offset,
                                         *host_offset, *nb_clusters);

    /* Allocate new clusters */
    trace_qcow2_cluster_alloc_phys(qemu_coroutine_self());
    if (*host_offset == 0) {
        /*@*host_offset == 0，表示可以在镜像文件中的任意可用空间分配，返回分配的起始地址*/
        int64_t cluster_offset =
            qcow2_alloc_clusters(bs, *nb_clusters * s->cluster_size);
        if (cluster_offset < 0) {
            return cluster_offset;
        }
        
        /*更新@*host_offset为分配的起始地址*/
        *host_offset = cluster_offset;
        return 0;
    } else {
        /* @*host_offset != 0，表示分配的clusters中第一个cluster的起始地址必须在镜像
         * 文件中的@*host_offset的位置，
         */
        int64_t ret = qcow2_alloc_clusters_at(bs, *host_offset, *nb_clusters);
        if (ret < 0) {
            return ret;
        }
        *nb_clusters = ret;
        return 0;
    }
}
```

```
int64_t qcow2_alloc_clusters_at(BlockDriverState *bs, uint64_t offset,
                                int64_t nb_clusters)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t cluster_index, refcount;
    uint64_t i;
    int ret;

    assert(nb_clusters >= 0);
    if (nb_clusters == 0) {
        return 0;
    }

    do {
        /*查找从@offset开始的至多@nb_clusters个连续的引用计数为0的cluster*/
        cluster_index = offset >> s->cluster_bits;
        for(i = 0; i < nb_clusters; i++) {
            /*读取@cluster_index所代表的cluster的引用计数，如果该cluster尚未分配，其引用计数视为0*/
            ret = qcow2_get_refcount(bs, cluster_index++, &refcount);
            if (ret < 0) {
                return ret;
            } else if (refcount != 0) {
                /*直到遇到refcount不为0的，则停止查找*/
                break;
            }
        }

        /*更新这些找到的cluster的引用计数*/
        ret = update_refcount(bs, offset, i << s->cluster_bits, 1, false,
                              QCOW2_DISCARD_NEVER);
    } while (ret == -EAGAIN);

    if (ret < 0) {
        return ret;
    }

    return i;
}
```

```
/*
 * Retrieves the refcount of the cluster given by its index and stores it in
 * *refcount. Returns 0 on success and -errno on failure.
 */
int qcow2_get_refcount(BlockDriverState *bs, int64_t cluster_index,
                       uint64_t *refcount)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t refcount_table_index, block_index;
    int64_t refcount_block_offset;
    int ret;
    void *refcount_block;

    /*计算@cluster_index对应的cluster所在的refcount table*/
    refcount_table_index = cluster_index >> s->refcount_block_bits;
    /*不在refcount table中，则视其引用计数为0*/
    if (refcount_table_index >= s->refcount_table_size) {
        *refcount = 0;
        return 0;
    }
    
    /*对应的refcount block不存在，则视其引用计数为0*/
    refcount_block_offset =
        s->refcount_table[refcount_table_index] & REFT_OFFSET_MASK;
    if (!refcount_block_offset) {
        *refcount = 0;
        return 0;
    }

    /*尝试从refcount block cache中读取其所在的@refcount_block*/
    ret = qcow2_cache_get(bs, s->refcount_block_cache, refcount_block_offset,
                          &refcount_block);
    /* 计算@cluster_index所在的cluster对应的@block_index，并在@refcount_block
     * 中查找对应的引用计数
     */
    block_index = cluster_index & (s->refcount_block_size - 1);
    *refcount = s->get_refcount(refcount_block, block_index);
    /*更新该@refcount_block到refcount block cache中*/
    qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);

    return 0;
}
```

```
/* Check if it's possible to merge a write request with the writing of
 * the data from the COW regions 
 * merge_cow的注释写的很清楚，不再赘述，从实现来看，只要是在@l2meta中
 * 找到一个可以合并@hd_qiov的QCowL2Meta即返回true，并设置该QCowL2Meta
 * ::data_qiov = hd_qiov
 */
static bool merge_cow(uint64_t offset, unsigned bytes,
                      QEMUIOVector *hd_qiov, QCowL2Meta *l2meta)
{
    QCowL2Meta *m;

    for (m = l2meta; m != NULL; m = m->next) {
        /* If both COW regions are empty then there's nothing to merge */
        if (m->cow_start.nb_bytes == 0 && m->cow_end.nb_bytes == 0) {
            continue;
        }

        /* The data (middle) region must be immediately after the
         * start region */
        if (l2meta_cow_start(m) + m->cow_start.nb_bytes != offset) {
            continue;
        }

        /* The end region must be immediately after the data (middle)
         * region */
        if (m->offset + m->cow_end.offset != offset + bytes) {
            continue;
        }

        /* Make sure that adding both COW regions to the QEMUIOVector
         * does not exceed IOV_MAX */
        /*没有足够的iovec用于merge了，则*/
        if (hd_qiov->niov > IOV_MAX - 2) {
            continue;
        }

        /* QCowL2Meta::data_qiov中记录的是来自guest的写数据，如果不为空，
         * 该iovc中的数据将会和start COW和end COW区域中的数据合并
         */
        m->data_qiov = hd_qiov;
        return true;
    }

    return false;
}
```

```
int qcow2_alloc_cluster_link_l2(BlockDriverState *bs, QCowL2Meta *m)
{
    BDRVQcow2State *s = bs->opaque;
    int i, j = 0, l2_index, ret;
    uint64_t *old_cluster, *l2_table;
    uint64_t cluster_offset = m->alloc_offset;

    trace_qcow2_cluster_link_l2(qemu_coroutine_self(), m->nb_clusters);
    assert(m->nb_clusters > 0);

    /* 新分配了@m->nb_clusters个cluster，这些新分配的cluster对应的旧cluster有可能需要
     * 被释放，old_cluster用于存放这些旧cluster地址
     */
    old_cluster = g_try_new(uint64_t, m->nb_clusters);
    if (old_cluster == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    /* copy content of unmodified sectors */
    /*执行COW，最终会写入COW区域数据和写请求的数据*/
    ret = perform_cow(bs, m);
    if (ret < 0) {
        goto err;
    }

    if (s->use_lazy_refcounts) {
        /* 如果采用lazy refcounts，则设置incompatible_features中的dirty bit，表明现在的
         * 镜像文件中refcount可能是不正确的，参考“qcow2布局.docx”中关于additional header
         * 部分incompatible_features的描述
         */
        qcow2_mark_dirty(bs);
    }
    
    /* qcow2_need_accurate_refcounts()用于检查是否成功将内存中的s->incompatible_features的
     * dirty bit位设置为1，虽然在采用lazy refcount的情况下qcow2_mark_dirty()会尝试去设置，
     * 但是可能由于更新header中incompatible_features中的dirty bit位失败等原因，内存中的
     * s->incompatible_features的dirty bit位可能未被设置，所以即使采用lazy refcount，也必须
     * 做此检查
     */
    if (qcow2_need_accurate_refcounts(s)) {
        /* 需要精确的refcount，则设置L2 table cache依赖于refcount block cache，只有当refcount
         * block cache被成功flush之后，才能flush L2 table cache
         */
        qcow2_cache_set_dependency(bs, s->l2_table_cache,
                                   s->refcount_block_cache);
    }

    /* 获取@m->offset所在的cluster在L2 table中的entry（其在L2 table中的索引号为@l2_index），
     * 并在L2 table cache中标记其为dirty
     */
    ret = get_cluster_table(bs, m->offset, &l2_table, &l2_index);
    if (ret < 0) {
        goto err;
    }
    qcow2_cache_entry_mark_dirty(bs, s->l2_table_cache, l2_table);

    assert(l2_index + m->nb_clusters <= s->l2_size);
    /*这@m->nb_clusters个cluster要么是新分配的cluster，要么是要执行COW的cluster*/
    for (i = 0; i < m->nb_clusters; i++) {
        /* if two concurrent writes happen to the same unallocated cluster
         * each write allocates separate cluster and writes data concurrently.
         * The first one to complete updates l2 table with pointer to its
         * cluster the second one has to do RMW (which is done above by
         * perform_cow()), update l2 table with its cluster pointer and free
         * old cluster. This is what this loop does */
        /* 上面这段注释有问题。
         * 参考https://lists.gnu.org/archive/html/qemu-devel/2017-06/msg00223.html。
         * 正确的注释如下：
         * handle_dependencies() protects from normal cluster allocation
         * collision; still L2 entry might be !0 in case of zero or compressed
         * cluster reusage or writing over the snapshot
         * 
         * 记录下来那些新分配的cluster对应的旧cluster（如果存在的话），并在后面确定
         * 是否需要释放这些旧的clusters
         */
        if (l2_table[l2_index + i] != 0) {
            old_cluster[j++] = l2_table[l2_index + i];
        }

        /*更新内存中的L2 table，这里会设置QCOW_OFLAG_COPIED标识，标识其引用计数为1*/
        l2_table[l2_index + i] = cpu_to_be64((cluster_offset +
                    (i << s->cluster_bits)) | QCOW_OFLAG_COPIED);
     }

    /*更新L2 table cache*/
    qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

    /*
     * If this was a COW, we need to decrease the refcount of the old cluster.
     *
     * Don't discard clusters that reach a refcount of 0 (e.g. compressed
     * clusters), the next write will reuse them anyway.
     */
    /* @m->keep_old_clusters表示是否需要保存旧cluster，见handle_alloc，如果无需保存
     * 旧cluster，且旧cluster数目不为0，则尝试释放之
     */
    if (!m->keep_old_clusters && j != 0) {
        for (i = 0; i < j; i++) {
            qcow2_free_any_clusters(bs, be64_to_cpu(old_cluster[i]), 1,
                                    QCOW2_DISCARD_NEVER);
        }
    }

    ret = 0;
err:
    g_free(old_cluster);
    return ret;
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


    if (decrease) {
        qcow2_cache_set_dependency(bs, s->refcount_block_cache,
            s->l2_table_cache);
    }

    /*计算出要遍历的起始cluster和最后一个cluster，依次遍历之*/
    start = start_of_cluster(s, offset);
    last = start_of_cluster(s, offset + length - 1);
    for(cluster_offset = start; cluster_offset <= last;
        cluster_offset += s->cluster_size)
    {
        int block_index;
        uint64_t refcount;
        int64_t cluster_index = cluster_offset >> s->cluster_bits;
        int64_t table_index = cluster_index >> s->refcount_block_bits;

        /* Load the refcount block and allocate it if needed */
        if (table_index != old_table_index) {
            /* 每处理一个cluster，都对应有一个refcount table，如果在处理当前cluster
             * 的时候，发生了refcount table切换，则肯定也发生了refcount block切换，
             * 将上一个已经更新过的refcount block更新到refcount block cache中
             */
            if (refcount_block) {
                qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
            }
            
            /* 加载当前处理的cluster对应的refcount block，该函数名称是alloc_refcount_block，
             * 字面上有些误导人，其实它的处理逻辑为：先尝试加载，如果没有再分配
             */
            ret = alloc_refcount_block(bs, cluster_index, &refcount_block);
            if (ret < 0) {
                goto fail;
            }
        }
        /*记录下来当前处理的cluster所在的refcount table*/
        old_table_index = table_index;

        /*在refcount block cache中标记当前cluster对应的@refcount block为脏*/
        qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache, refcount_block);

        /*获取当前cluster在@refcount_block中的索引*/
        block_index = cluster_index & (s->refcount_block_size - 1);

        /*获取当前cluster的引用计数*/
        refcount = s->get_refcount(refcount_block, block_index);
        /*更新引用计数*/
        if (decrease ? (refcount - addend > refcount)
                     : (refcount + addend < refcount ||
                        refcount + addend > s->refcount_max))
        {
            ret = -EINVAL;
            goto fail;
        }
        if (decrease) {
            refcount -= addend;
        } else {
            refcount += addend;
        }
        
        /*如果引用计数降为0，则该cluster是处于free状态，更新@s->free_cluster_index*/
        if (refcount == 0 && cluster_index < s->free_cluster_index) {
            s->free_cluster_index = cluster_index;
        }
        /*更新引用计数到@refcount_block中*/
        s->set_refcount(refcount_block, block_index, refcount);

        /*discard(调用底层的discard去真正释放空间？)*/
        if (refcount == 0 && s->discard_passthrough[type]) {
            update_refcount_discard(bs, cluster_offset, s->cluster_size);
        }
    }

    ret = 0;
fail:
    if (!s->cache_discards) {
        qcow2_process_discards(bs, ret);
    }

    /*更新最后一个cluster对应的@refcount_block到refcount block cache中*/
    if (refcount_block) {
        qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
    }

    /*
     * Try do undo any updates if an error is returned (This may succeed in
     * some cases like ENOSPC for allocating a new refcount block)
     */
    /*如果前面的操作发生错误，则undo之*/
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
/*
 * Free a cluster using its L2 entry (handles clusters of all types, e.g.
 * normal cluster, compressed cluster, etc.)
 */
void qcow2_free_any_clusters(BlockDriverState *bs, uint64_t l2_entry,
                             int nb_clusters, enum qcow2_discard_type type)
{
    BDRVQcow2State *s = bs->opaque;

    /* 对于QCOW2_CLUSTER_COMPRESSED、QCOW2_CLUSTER_NORMAL或QCOW2_CLUSTER_ZERO_ALLOC
     * 类型的cluster直接调用qcow2_free_clusters释放之，对于其它类型，什么都不做，因
     * 为其它类型的cluster没有实际执行分配
     */
    switch (qcow2_get_cluster_type(l2_entry)) {
    case QCOW2_CLUSTER_COMPRESSED:
        {
            int nb_csectors;
            nb_csectors = ((l2_entry >> s->csize_shift) &
                           s->csize_mask) + 1;
            qcow2_free_clusters(bs,
                (l2_entry & s->cluster_offset_mask) & ~511,
                nb_csectors * 512, type);
        }
        break;
    case QCOW2_CLUSTER_NORMAL:
    case QCOW2_CLUSTER_ZERO_ALLOC:
        if (offset_into_cluster(s, l2_entry & L2E_OFFSET_MASK)) {
            qcow2_signal_corruption(bs, false, -1, -1,
                                    "Cannot free unaligned cluster %#llx",
                                    l2_entry & L2E_OFFSET_MASK);
        } else {
            qcow2_free_clusters(bs, l2_entry & L2E_OFFSET_MASK,
                                nb_clusters << s->cluster_bits, type);
        }
        break;
    case QCOW2_CLUSTER_ZERO_PLAIN:
    case QCOW2_CLUSTER_UNALLOCATED:
        break;
    default:
        abort();
    }
}
```

```
void qcow2_free_clusters(BlockDriverState *bs,
                          int64_t offset, int64_t size,
                          enum qcow2_discard_type type)
{
    int ret;

    BLKDBG_EVENT(bs->file, BLKDBG_CLUSTER_FREE);
    /*减少[offset, offset + size)区间所涉及的cluster的引用计数*/
    ret = update_refcount(bs, offset, size, 1, true, type);
    if (ret < 0) {
        fprintf(stderr, "qcow2_free_clusters failed: %s\n", strerror(-ret));
        /* TODO Remember the clusters to free them later and avoid leaking */
    }
}
```

```
/*
 * Sets the dirty bit and flushes afterwards if necessary.
 *
 * The incompatible_features bit is only set if the image file header was
 * updated successfully.  Therefore it is not required to check the return
 * value of this function.
 */
int qcow2_mark_dirty(BlockDriverState *bs)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t val;
    int ret;

    assert(s->qcow_version >= 3);
    /*如果已经在incompatible_features中设置了dirty bit，则直接返回*/
    if (s->incompatible_features & QCOW2_INCOMPAT_DIRTY) {
        return 0; /* already dirty */
    }

    /* 设置incompatible_features中dirty bit，并写入到header中，并执行flush，
     * 确保写入到镜像文件中
     */
    val = cpu_to_be64(s->incompatible_features | QCOW2_INCOMPAT_DIRTY);
    ret = bdrv_pwrite(bs->file, offsetof(QCowHeader, incompatible_features),
                      &val, sizeof(val));
    ret = bdrv_flush(bs->file->bs);

    /* Only treat image as dirty if the header was updated successfully */
    /* 只有当header中incompatible_features的dirty bit位成功设置才将内存中
     * incompatible_features的dirty bit位
     */
    s->incompatible_features |= QCOW2_INCOMPAT_DIRTY;
    return 0;
}
```

```
static int perform_cow(BlockDriverState *bs, QCowL2Meta *m)
{
    BDRVQcow2State *s = bs->opaque;
    /*start COW和end COW区域*/
    Qcow2COWRegion *start = &m->cow_start;
    Qcow2COWRegion *end = &m->cow_end;
    unsigned buffer_size;
    unsigned data_bytes = end->offset - (start->offset + start->nb_bytes);
    bool merge_reads;
    uint8_t *start_buffer, *end_buffer;
    QEMUIOVector qiov;
    int ret;

    assert(start->nb_bytes <= UINT_MAX - end->nb_bytes);
    assert(start->nb_bytes + end->nb_bytes <= UINT_MAX - data_bytes);
    assert(start->offset + start->nb_bytes <= end->offset);
    assert(!m->data_qiov || m->data_qiov->size == data_bytes);

    if (start->nb_bytes == 0 && end->nb_bytes == 0) {
        return 0;
    }

    /* 如果start COW、end COW以及要写入的数据总共的字节数不大于16383字节，
     * 则同时读取这三个区域，否则分别读取start COW和end COW区域，根据如何
     * 读取设置相应的读取buffer的大小
     */
    /* If we have to read both the start and end COW regions and the
     * middle region is not too large then perform just one read
     * operation */
    merge_reads = start->nb_bytes && end->nb_bytes && data_bytes <= 16384;
    if (merge_reads) {
        buffer_size = start->nb_bytes + data_bytes + end->nb_bytes;
    } else {
        /* If we have to do two reads, add some padding in the middle
         * if necessary to make sure that the end region is optimally
         * aligned. */
        size_t align = bdrv_opt_mem_align(bs);
        assert(align > 0 && align <= UINT_MAX);
        assert(QEMU_ALIGN_UP(start->nb_bytes, align) <=
               UINT_MAX - end->nb_bytes);
        buffer_size = QEMU_ALIGN_UP(start->nb_bytes, align) + end->nb_bytes;
    }

    /* Reserve a buffer large enough to store all the data that we're
     * going to read */
    /* 分配buffer，用于存放读取的数据*/
    /* @start_buffer为start COW region的buffer起始地址*/
    start_buffer = qemu_try_blockalign(bs, buffer_size);
    if (start_buffer == NULL) {
        return -ENOMEM;
    }
    
    /* The part of the buffer where the end region is located */
    /* @end_buffer 为end COW region的buffer起始地址*/
    end_buffer = start_buffer + buffer_size - end->nb_bytes;

    qemu_iovec_init(&qiov, 2 + (m->data_qiov ? m->data_qiov->niov : 0));

    qemu_co_mutex_unlock(&s->lock);
    /* First we read the existing data from both COW regions. We
     * either read the whole region in one go, or the start and end
     * regions separately. */
    if (merge_reads) {
        /*同时读取start COW和end COW region*/
        qemu_iovec_add(&qiov, start_buffer, buffer_size);
        ret = do_perform_cow_read(bs, m->offset, start->offset, &qiov);
    } else {
        /*分别读取start COW和end COW region*/
        qemu_iovec_add(&qiov, start_buffer, start->nb_bytes);
        ret = do_perform_cow_read(bs, m->offset, start->offset, &qiov);
        if (ret < 0) {
            goto fail;
        }

        qemu_iovec_reset(&qiov);
        qemu_iovec_add(&qiov, end_buffer, end->nb_bytes);
        ret = do_perform_cow_read(bs, m->offset, end->offset, &qiov);
    }

    /*加密相关，暂时不予关注*/
    
    /* And now we can write everything. If we have the guest data we
     * can write everything in one single operation */
    if (m->data_qiov) {
        /* @m->data_qiov不为空，则在merge_cow()中一定有过成功的合并，写请求相关
         * 的数据就存放在m->data_qiov中
         */
        
        /*重置@qiov，并依次将@start_buffer、@m->data_qiov、@end_buffer添加到@qiov中*/
        qemu_iovec_reset(&qiov);
        if (start->nb_bytes) {
            qemu_iovec_add(&qiov, start_buffer, start->nb_bytes);
        }
        qemu_iovec_concat(&qiov, m->data_qiov, 0, data_bytes);
        if (end->nb_bytes) {
            qemu_iovec_add(&qiov, end_buffer, end->nb_bytes);
        }
        
        /* NOTE: we have a write_aio blkdebug event here followed by
         * a cow_write one in do_perform_cow_write(), but there's only
         * one single I/O operation */
        BLKDBG_EVENT(bs->file, BLKDBG_WRITE_AIO);
        /*写入*/
        ret = do_perform_cow_write(bs, m->alloc_offset, start->offset, &qiov);
    } else {
        /* @m->data_qiov为空，写请求相关的数据可能已经写入，见qcow2_co_pwritev，
         * 只需要依次写入start COW区域的数据和end COW区域的数据
         */
        /* If there's no guest data then write both COW regions separately */
        qemu_iovec_reset(&qiov);
        qemu_iovec_add(&qiov, start_buffer, start->nb_bytes);
        ret = do_perform_cow_write(bs, m->alloc_offset, start->offset, &qiov);
        if (ret < 0) {
            goto fail;
        }

        qemu_iovec_reset(&qiov);
        qemu_iovec_add(&qiov, end_buffer, end->nb_bytes);
        ret = do_perform_cow_write(bs, m->alloc_offset, end->offset, &qiov);
    }

fail:
    qemu_co_mutex_lock(&s->lock);

    /*
     * Before we update the L2 table to actually point to the new cluster, we
     * need to be sure that the refcounts have been increased and COW was
     * handled.
     */
    if (ret == 0) {
        qcow2_cache_depends_on_flush(s->l2_table_cache);
    }

    qemu_vfree(start_buffer);
    qemu_iovec_destroy(&qiov);
    return ret;
}
```

```
static int coroutine_fn do_perform_cow_read(BlockDriverState *bs,
                                            uint64_t src_cluster_offset,
                                            unsigned offset_in_cluster,
                                            QEMUIOVector *qiov)
{
    int ret;

    ......
    
    /*对于qcow2来说，对应的是qcow2_co_preadv，请参考“qemu && qcow2 notes - read”*/
    ret = bs->drv->bdrv_co_preadv(bs, src_cluster_offset + offset_in_cluster,
                                  qiov->size, qiov, 0);

    return 0;
}
```

```
static int coroutine_fn do_perform_cow_write(BlockDriverState *bs,
                                             uint64_t cluster_offset,
                                             unsigned offset_in_cluster,
                                             QEMUIOVector *qiov)
{
    int ret;

    ......
    
    /* 注意这里第一个参数是@bs->file，再次调用bdrv_co_pwritev，在前面已经讲过该函数，
     * 在bdrv_co_pwritev中先做对齐处理，然后调用bdrv_aligned_pwritev -> 
     * bdrv_driver_pwritev -> drv->bdrv_co_pwritev | drv->bdrv_co_writev_flags |
     * drv->bdrv_co_writev | drv->bdrv_aio_writev，对于protocol layer为file的情况下，
     * 最终调用的是raw_co_pwritev
     */
    ret = bdrv_co_pwritev(bs->file, cluster_offset + offset_in_cluster,
                          qiov->size, qiov, 0);

    return 0;
}
```

```
static int coroutine_fn raw_co_pwritev(BlockDriverState *bs, uint64_t offset,
                                       uint64_t bytes, QEMUIOVector *qiov,
                                       int flags)
{
    assert(flags == 0);
    return raw_co_prw(bs, offset, bytes, qiov, QEMU_AIO_WRITE);
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
            /* 所有请求都提交失败（注意是提交失败，不是执行失败），则将pending队列中的
             * 第一个请求移除并设置该请求的执行结果，继续提交除第一个请求以外的后续请求
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

    /* BlockDriverState::active_flush_req为true标识当前还有正在进行中的flush
     * 请求尚未完成，将当前的协程添加到@BlockDriverState::flush_queue中
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
    /* 如果提供了.bdrv_co_flush接口函数，则会确保flush任何一层的数据，直到数据
     * 到达物理硬盘
     */
    /*对于qcow2来说，没有提供bdrv_co_flush接口函数*/
    /*对于file来说，也没有提供bdrv_co_flush接口函数*/
    if (bs->drv->bdrv_co_flush) {
        ret = bs->drv->bdrv_co_flush(bs);
        goto out;
    }

    /* Write back cached data to the OS even with cache=unsafe */
    /*如果没有提供.bdrv_co_flush接口函数，则尝试先flush到host os*/
    /*对于qcow2来说.bdrv_co_flush_to_os = qcow2_co_flush_to_os*/
    /*对于file来说，没有提供bdrv_co_flush_to_os接口函数*/
    BLKDBG_EVENT(bs->file, BLKDBG_FLUSH_TO_OS);
    if (bs->drv->bdrv_co_flush_to_os) {
        ret = bs->drv->bdrv_co_flush_to_os(bs);
        if (ret < 0) {
            goto out;
        }
    }

    /* But don't actually force it to the disk with cache=unsafe */
    /* 如果设置了BDRV_O_NO_FLUSH标识，则对于处于host os中的数据不再执行
     * flush，否则跳转到flush_parent处
     */
    if (bs->open_flags & BDRV_O_NO_FLUSH) {
        goto flush_parent;
    }

    /* Check if we really need to flush anything */
    /* 如果当前flush的版本号跟write的版本号相同，则表明无需执行flush了，
     * 跳转到flush_parent处
     */
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

关于BlockDriver中提供的不同的flush函数的说明：
struct BlockDriver{
    /*
     * Flushes all data for all layers by calling bdrv_co_flush for underlying
     * layers, if needed. This function is needed for deterministic
     * synchronization of the flush finishing callback.
     */
    int coroutine_fn (*bdrv_co_flush)(BlockDriverState *bs);
    
    /*
     * Flushes all data that was already written to the OS all the way down to
     * the disk (for example file-posix.c calls fsync()).
     */
    int coroutine_fn (*bdrv_co_flush_to_disk)(BlockDriverState *bs);
    
    /*
     * Flushes all internal caches to the OS. The data may still sit in a
     * writeback cache of the host OS, but it will survive a crash of the qemu
     * process.
     */
    int coroutine_fn (*bdrv_co_flush_to_os)(BlockDriverState *bs);
};

对于protocol layer为file的qcow2来说，在bdrv_driver_pwritev -> bdrv_co_flush
中首先调用的是bdrv_qcow2的driver层的flush相关的接口，然后递归调用bdrv_co_flush，
会再次调用protocol layer的flush相关的接口，完成flush逻辑。
总之在bdrv_driver_pwritev -> bdrv_co_flush逻辑中会依次调用qcow2_co_flush_to_os
和raw_aio_flush。

static coroutine_fn int qcow2_co_flush_to_os(BlockDriverState *bs)
{
    /*本函数用于将qcow2 cache中的数据flush到host os（的镜像文件）中*/
    BDRVQcow2State *s = bs->opaque;
    int ret;

    qemu_co_mutex_lock(&s->lock);
    /*flush L2 table cache到镜像文件中，关于qcow2 cache请参考“qcow2 notes - cache”*/
    ret = qcow2_cache_write(bs, s->l2_table_cache);
    if (ret < 0) {
        qemu_co_mutex_unlock(&s->lock);
        return ret;
    }

    if (qcow2_need_accurate_refcounts(s)) {
        /*flush refcount block cache到镜像文件中*/
        ret = qcow2_cache_write(bs, s->refcount_block_cache);
        if (ret < 0) {
            qemu_co_mutex_unlock(&s->lock);
            return ret;
        }
    }
    qemu_co_mutex_unlock(&s->lock);

    return 0;
}

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

## Q&A
1. 如果qcow2最终走到了bdrv_file，且最终调用AIO接口，那么是如何保证最终
提交的AIO请求中buffer、offset和bytes都是满足direct IO字节对齐要求的呢？
```
详见“qcow2 notes - open: bdrv_refresh_limits”。
```


