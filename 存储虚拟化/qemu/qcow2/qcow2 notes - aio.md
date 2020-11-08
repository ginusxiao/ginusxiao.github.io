# 简介

```
本文打算从raw_co_prw开始，以native libaio为例分析qemu中是如何使用aio的。
```

# 源码分析

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
    if (s->needs_alignment) {
        if (!bdrv_qiov_is_aligned(bs, qiov)) {
            type |= QEMU_AIO_MISALIGNED;
#ifdef CONFIG_LINUX_AIO
        } else if (s->use_linux_aio) {
            /*假设我们使用linux native AIO，下面用到了3个接口，分别是：
              bdrv_get_aio_context(),
              aio_get_linux_aio(),
              laio_co_submit()
              后面逐一分析。
            */
            LinuxAioState *aio = aio_get_linux_aio(bdrv_get_aio_context(bs));
            assert(qiov->size == bytes);
            return laio_co_submit(bs, aio, s->fd, offset, qiov, type);
#endif
        }
    }

    return paio_submit_co(bs, s->fd, offset, qiov, bytes, type);
}
```

## AioContext建立
```
AioContext *bdrv_get_aio_context(BlockDriverState *bs)
{
    /* 这里直接返回BlockDriverState::aio_context，这又是在哪里初始化的呢，
     * 下面以virtio block data plane分析
     */
    return bs->aio_context;
}

void virtio_blk_data_plane_create(VirtIODevice *vdev, VirtIOBlkConf *conf,
                                  VirtIOBlockDataPlane **dataplane,
                                  Error **errp)
{
    VirtIOBlockDataPlane *s;
    
    ......
    
    s = g_new0(VirtIOBlockDataPlane, 1);
    s->vdev = vdev;
    s->conf = conf;

    if (conf->iothread) {
        s->iothread = conf->iothread;
        object_ref(OBJECT(s->iothread));
        /*直接返回IOThread::AioContext*/
        s->ctx = iothread_get_aio_context(s->iothread);
    } else {
        s->ctx = qemu_get_aio_context();
    }
    
    ......
    
    *dataplane = s;
}

AioContext *iothread_get_aio_context(IOThread *iothread)
{
    return iothread->ctx;
}

/* Context: QEMU global mutex held */
int virtio_blk_data_plane_start(VirtIODevice *vdev)
{
    VirtIOBlock *vblk = VIRTIO_BLK(vdev);
    VirtIOBlockDataPlane *s = vblk->dataplane;

    ......
    
    blk_set_aio_context(s->conf->conf.blk, s->ctx);

    ......
    
    return 0;
}

void blk_set_aio_context(BlockBackend *blk, AioContext *new_context)
{
    BlockDriverState *bs = blk_bs(blk);
    ThrottleGroupMember *tgm = &blk->public.throttle_group_member;

    if (bs) {
        if (tgm->throttle_state) {
            throttle_group_detach_aio_context(tgm);
            throttle_group_attach_aio_context(tgm, new_context);
        }
        
        bdrv_set_aio_context(bs, new_context);
    }
}

void bdrv_set_aio_context(BlockDriverState *bs, AioContext *new_context)
{
    AioContext *ctx = bdrv_get_aio_context(bs);

    ......
    
    bdrv_attach_aio_context(bs, new_context);
    
    ......
}

void bdrv_attach_aio_context(BlockDriverState *bs,
                             AioContext *new_context)
{
    ......

    bs->aio_context = new_context;

    ......
}

从上面分析来看，bs->aio_context来自于data plane创建的线程。
```

## Libaio初始化
```
#ifdef CONFIG_LINUX_AIO
LinuxAioState *aio_get_linux_aio(AioContext *ctx)
{
    if (!ctx->linux_aio) {
        ctx->linux_aio = laio_init();
        laio_attach_aio_context(ctx->linux_aio, ctx);
    }
    return ctx->linux_aio;
}
#endif

LinuxAioState *laio_init(void)
{
    LinuxAioState *s;

    /*分配LinuxAioState，LinuxAioState数据结构为：
    struct LinuxAioState {
        //qemu中与AIO相关的上下文
        AioContext *aio_context;
    
        //libaio內建上下文
        io_context_t ctx;
        
        //event notifier
        EventNotifier e;
    
        //请求队列
        LaioQueue io_q;
    
        // I/O completion processing. Only runs in I/O thread.
        QEMUBH *completion_bh;
        int event_idx;
        int event_max;
    };
    */
    s = g_malloc0(sizeof(*s));
    /*准备event notifier，可能通过eventfd实现，也可能通过pipe实现*/
    if (event_notifier_init(&s->e, false) < 0) {
        goto out_free_state;
    }

    /*libaio内建函数，创建AIO上下文*/
    if (io_setup(MAX_EVENTS, &s->ctx) != 0) {
        goto out_close_efd;
    }

    /*初始化请求队列*/
    ioq_init(&s->io_q);

    return s;

out_close_efd:
    event_notifier_cleanup(&s->e);
out_free_state:
    g_free(s);
    return NULL;
}

int event_notifier_init(EventNotifier *e, int active)
{
    int fds[2];
    int ret;

#ifdef CONFIG_EVENTFD
    ret = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
#else
    ret = -1;
    errno = ENOSYS;
#endif
    if (ret >= 0) {
        e->rfd = e->wfd = ret;
    } else {
        if (errno != ENOSYS) {
            return -errno;
        }
        if (qemu_pipe(fds) < 0) {
            return -errno;
        }
        ret = fcntl_setfl(fds[0], O_NONBLOCK);
        if (ret < 0) {
            ret = -errno;
            goto fail;
        }
        ret = fcntl_setfl(fds[1], O_NONBLOCK);
        if (ret < 0) {
            ret = -errno;
            goto fail;
        }
        e->rfd = fds[0];
        e->wfd = fds[1];
    }
    
    /*是否触发event notifier*/
    if (active) {
        event_notifier_set(e);
    }
    return 0;

fail:
    close(fds[0]);
    close(fds[1]);
    return ret;
}

int event_notifier_set(EventNotifier *e)
{
    static const uint64_t value = 1;
    ssize_t ret;

    do {
        ret = write(e->wfd, &value, sizeof(value));
    } while (ret < 0 && errno == EINTR);

    /* EAGAIN is fine, a read must be pending.  */
    if (ret < 0 && errno != EAGAIN) {
        return -errno;
    }
    
    return 0;
}

static void ioq_init(LaioQueue *io_q)
{
    QSIMPLEQ_INIT(&io_q->pending);
    io_q->plugged = 0;
    io_q->in_queue = 0;
    io_q->in_flight = 0;
    io_q->blocked = false;
}

void laio_attach_aio_context(LinuxAioState *s, AioContext *new_context)
{
    s->aio_context = new_context;
    
    /*创建新的bottom half，并添加到@new_context的bottom halves链表中，
    新建的bottom half相关联的AioContext是@new_context，回调函数是
    @qemu_laio_completion_bh，回调函数的参数是@s*/
    s->completion_bh = aio_bh_new(new_context, qemu_laio_completion_bh, s);
    
    /* 注册@s->e关联的fd到@new_context->aio_handlers中，并指定相关的读事件处理
     * 函数为qemu_laio_completion_cb，poll函数为qemu_laio_poll_cb，那么这里的poll
     * 是在哪里被用到的呢？参考[qemu thread](http://note.youdao.com/noteshare?
     * id=21db404e3c91d086752cd91b3182eebe&sub=479CA8B341544FE4AFC3AC20FA5CDB55)，
     * 在iothread_run -> aio_poll中所有注册在@new_context->aio_handlers中的AIO handler，
     * 如果其指定了poll函数，则会执行poll检查是否有其感兴趣的事件到达(对于@s->e来说，
     * poll函数为qemu_laio_poll_cb)，如果有则调用相应事件的回调函数(对于@s->e来说，
     * 注册的读事件回调函数qemu_laio_completion_cb)，在“libaio运转”中会有关于
     * qemu_laio_poll_cb和qemu_laio_completion_cb的分析
     */
    aio_set_event_notifier(new_context, &s->e, false,
                           qemu_laio_completion_cb,
                           qemu_laio_poll_cb);
}

QEMUBH *aio_bh_new(AioContext *ctx, QEMUBHFunc *cb, void *opaque)
{
    QEMUBH *bh;
    
    /*创建bottom half*/
    bh = g_new(QEMUBH, 1);
    *bh = (QEMUBH){
        .ctx = ctx, //关联的AioContext
        .cb = cb, //关联的回调函数
        .opaque = opaque, //关联的回调相关的参数
    };
    
    /*在@ctx->list_lock的保护下，将新创建的@bh链入@ctx->first_bh所在的链表中*/
    qemu_lockcnt_lock(&ctx->list_lock);
    bh->next = ctx->first_bh;
    /* Make sure that the members are ready before putting bh into list */
    smp_wmb();
    ctx->first_bh = bh;
    qemu_lockcnt_unlock(&ctx->list_lock);
    
    return bh;
}

void aio_set_event_notifier(AioContext *ctx,
                            EventNotifier *notifier,
                            bool is_external,
                            EventNotifierHandler *io_read,
                            AioPollFn *io_poll)
{
    /*event_notifier_get_fd(notifier): 获取@notifier关联的fd*/
    aio_set_fd_handler(ctx, event_notifier_get_fd(notifier), is_external,
                       (IOHandler *)io_read, NULL, io_poll, notifier);
}

void aio_set_fd_handler(AioContext *ctx,
                        int fd,
                        bool is_external,
                        IOHandler *io_read,
                        IOHandler *io_write,
                        AioPollFn *io_poll,
                        void *opaque)
{
    AioHandler *node;
    bool is_new = false;
    bool deleted = false;

    qemu_lockcnt_lock(&ctx->list_lock);
    /*查找关联到@fs的注册到@ctx->aio_handlers中的AIO handler*/
    node = find_aio_handler(ctx, fd);

    /* Are we deleting the fd handler? */
    /*如果同时不指定@io_read, @io_write, @io_poll，则表明要删除指定的AIO handler*/
    if (!io_read && !io_write && !io_poll) {
        if (node == NULL) {
            /*在@ctx->aio_handlers中没找到该AIO handler，则直接返回*/
            qemu_lockcnt_unlock(&ctx->list_lock);
            return;
        }

        /*Removes a file descriptor from the set of file descriptors polled for this source*/
        g_source_remove_poll(&ctx->source, &node->pfd);

        /* If the lock is held, just mark the node as deleted */
        if (qemu_lockcnt_count(&ctx->list_lock)) {
            node->deleted = 1;
            node->pfd.revents = 0;
        } else {
            /* Otherwise, delete it for real.  We can't just mark it as
             * deleted because deleted nodes are only cleaned up while
             * no one is walking the handlers list.
             */
            QLIST_REMOVE(node, node);
            deleted = true;
        }

        if (!node->io_poll) {
            ctx->poll_disable_cnt--;
        }
    } else {
        if (node == NULL) {
            /* Alloc and insert if it's not already there */
            /*不存在，则分配新的AIO handler并添加到@ctx->aio_handlers中*/
            node = g_new0(AioHandler, 1);
            /*关联到@fd*/
            node->pfd.fd = fd;
            QLIST_INSERT_HEAD_RCU(&ctx->aio_handlers, node, node);

            /*Adds a file descriptor to the set of file descriptors polled for this source*/
            g_source_add_poll(&ctx->source, &node->pfd);
            is_new = true;

            /*如果该AIO handler没有指定@io_poll，则增加@ctx->poll_disable_cnt*/
            ctx->poll_disable_cnt += !io_poll;
        } else {
            ctx->poll_disable_cnt += !io_poll - !node->io_poll;
        }

        /* Update handler with latest information */
        node->io_read = io_read;
        node->io_write = io_write;
        node->io_poll = io_poll;
        node->opaque = opaque;
        node->is_external = is_external;

        /* 设置关注的事件类型，因为不确定后面是否走epoll，所以这里采用的是G_IO_IN,
         * G_IO_OUT,G_IO_HUP,G_IO_ERR作为关注的事件类型，如果后面用epoll，则会转换
         * 为相应的epoll事件类型
         */
        node->pfd.events = (io_read ? G_IO_IN | G_IO_HUP | G_IO_ERR : 0);
        node->pfd.events |= (io_write ? G_IO_OUT | G_IO_ERR : 0);
    }

    /* 如果build with CONFIG_EPOLL，则注册关联到@node->pfd.fd的事件到@ctx->epollfd
     * 对应的epoll实例中
     */
    aio_epoll_update(ctx, node, is_new);
    qemu_lockcnt_unlock(&ctx->list_lock);
    aio_notify(ctx);

    if (deleted) {
        g_free(node);
    }
}

static AioHandler *find_aio_handler(AioContext *ctx, int fd)
{
    AioHandler *node;

    /*遍历所有注册的AIO handlers，查找是否存在一个关联到@fd的AIO handler*/
    QLIST_FOREACH(node, &ctx->aio_handlers, node) {
        if (node->pfd.fd == fd)
            if (!node->deleted)
                return node;
    }

    return NULL;
}

static void aio_epoll_update(AioContext *ctx, AioHandler *node, bool is_new)
{
    struct epoll_event event;
    int r;
    int ctl;

    if (!ctx->epoll_enabled) {
        return;
    }
    if (!node->pfd.events) {
        ctl = EPOLL_CTL_DEL;
    } else {
        event.data.ptr = node;
        /*
         * 将@node->pfd.events转换为@event.events.
         * @node->pfd.events是在aio_set_fd_handler中设置的，是G_IO_IN,G_IO_OUT,G_IO_HUP,G_IO_ERR等bit的组合.
         * @event.events则是下列epoll事件类型的组合:EPOLLIN,EPOLLOUT,EPOLLRDHUP,EPOLLPRI,EPOLLERR,EPOLLHUP,
         * EPOLLET,EPOLLONESHOT.
         */
        event.events = epoll_events_from_pfd(node->pfd.events);
        ctl = is_new ? EPOLL_CTL_ADD : EPOLL_CTL_MOD;
    }

    /*
     * EPOLL_CTL_ADD: Register the target file descriptor @node->pfd.fd on the epoll instance referred
     * to by the file descriptor @ctx->epollfd and associate the event @event with the internal file 
     * linked to @node->pfd.fd.
     *
     * EPOLL_CTL_MOD: change the event @event with the internal file linked to @node->pfd.fd.
     *
     * EPOLL_CTL_DEL: Remove (deregister) the target file descriptor @node->pfd.fd from the epoll
     * instance referred to by @ctx->epollfd. The event is ignored and can be NULL
     */
    r = epoll_ctl(ctx->epollfd, ctl, node->pfd.fd, &event);
    if (r) {
        /*epoll_ctl不成功，则设置暂时不采用epoll*/
        aio_epoll_disable(ctx);
    }
}

void aio_notify(AioContext *ctx)
{
    /* Write e.g. bh->scheduled before reading ctx->notify_me.  Pairs
     * with atomic_or in aio_ctx_prepare or atomic_add in aio_poll.
     */
    smp_mb();
    if (ctx->notify_me) {
        event_notifier_set(&ctx->notifier);
        atomic_mb_set(&ctx->notified, true);
    }
}
```

## Libaio运转
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

    /*如果正在执行中，则退出协程，等待被再次调度*/
    if (laiocb.ret == -EINPROGRESS) {
        qemu_coroutine_yield();
    }
    
    return laiocb.ret;
}

static int laio_do_submit(int fd, struct qemu_laiocb *laiocb, off_t offset,
                          int type)
{
    LinuxAioState *s = laiocb->ctx;
    struct iocb *iocbs = &laiocb->iocb;
    QEMUIOVector *qiov = laiocb->qiov;

    /*根据类型，分别采用io_prep_pwritev、io_prep_preadv进行准备*/
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
    
    /*设置AIO请求关联的fd*/
    io_set_eventfd(&laiocb->iocb, event_notifier_get_fd(&s->e));
    /*假如pending队列*/
    QSIMPLEQ_INSERT_TAIL(&s->io_q.pending, laiocb, next);
    /*增加队列中请求个数*/
    s->io_q.in_queue++;
    /* 如果可以提交，则尝试从pending队列中提交AIO请求
     * s->io_q.blocked: 在调用ioq_submit之后都会检查pending队列中的元素个数，
     * 如果大于0则设置为true，否则设置为false
     * s->io_q.plugged: 在每次调用BlockDriver::bdrv_io_plug的时候都会增加该计数，
     * 在每次调用BlockDriver::bdrv_io_unplug时候都会减少该计数
     * s->io_q.in_flight: 已经成功调用io_submit的请求个数
     * s->io_q.in_queue: 处于pending队列中的AIO请求个数
     */
    if (!s->io_q.blocked &&
        (!s->io_q.plugged ||
         s->io_q.in_flight + s->io_q.in_queue >= MAX_EVENTS)) {
        ioq_submit(s);
    }

    return 0;
}

static void ioq_submit(LinuxAioState *s)
{
    int ret, len;
    struct qemu_laiocb *aiocb;
    struct iocb *iocbs[MAX_EVENTS];
    QSIMPLEQ_HEAD(, qemu_laiocb) completed;

    do {
        /*超过最大数目*/
        if (s->io_q.in_flight >= MAX_EVENTS) {
            break;
        }
        
        /*从pending队列中选择AIO请求提交，直到已提交尚未完成的请求数目多于MAX_EVENTS为止*/
        len = 0;
        QSIMPLEQ_FOREACH(aiocb, &s->io_q.pending, next) {
            iocbs[len++] = &aiocb->iocb;
            if (s->io_q.in_flight + len >= MAX_EVENTS) {
                break;
            }
        }

        /*提交这一批请求*/
        ret = io_submit(s->ctx, len, iocbs);
        if (ret == -EAGAIN) {
            break;
        }
        
        if (ret < 0) {
            /* Fail the first request, retry the rest */
            /*出错，则将第一个AIO请求标记为出错，重新尝试提交后续请求*/
            aiocb = QSIMPLEQ_FIRST(&s->io_q.pending);
            QSIMPLEQ_REMOVE_HEAD(&s->io_q.pending, next);
            s->io_q.in_queue--;
            aiocb->ret = ret;
            qemu_laio_process_completion(aiocb);
            continue;
        }

        /*更新已提交请求数目和队列中请求数目*/
        s->io_q.in_flight += ret;
        s->io_q.in_queue  -= ret;
        aiocb = container_of(iocbs[ret - 1], struct qemu_laiocb, iocb);
        /*从pending队列中删除已提交的请求*/
        QSIMPLEQ_SPLIT_AFTER(&s->io_q.pending, aiocb, next, &completed);
    } while (ret == len && !QSIMPLEQ_EMPTY(&s->io_q.pending));
    s->io_q.blocked = (s->io_q.in_queue > 0);

    if (s->io_q.in_flight) {
        /* We can try to complete something just right away if there are
         * still requests in-flight. */
        /*获取那些已经完成的请求并调用它们的回调函数*/
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

在laio_attach_aio_context中，有如下代码：
    s->aio_context = new_context;
    s->completion_bh = aio_bh_new(new_context, qemu_laio_completion_bh, s);
    aio_set_event_notifier(new_context, &s->e, false,
                           qemu_laio_completion_cb,
                           qemu_laio_poll_cb);
这里在@new_context中注册了@s->completion_bh到@new_context->first_bh链表中，也注册了
@s->e到@new_context->aio_handlers链表中，在iothread_run -> aio_poll中会依次执行
AIO handler poll callback、bottom half callback、AIO handler read/write event callback
和timer callback。对应到这里就是依次执行qemu_laio_poll_cb、qemu_laio_completion_bh和
qemu_laio_completion_cb，下面分别分析。

static bool qemu_laio_poll_cb(void *opaque)
{
    EventNotifier *e = opaque;
    LinuxAioState *s = container_of(e, LinuxAioState, e);
    struct io_event *events;

    /* 检查是否存在已完成的事件，有则返回已完成事件的数目，同时返回已完成事件列表存
     * 放于@events中
     */
    if (!io_getevents_peek(s->ctx, &events)) {
        return false;
    }

    qemu_laio_process_completions_and_submit(s);
    return true;
}

/**
 * io_getevents_peek:
 * @ctx: AIO context
 * @events: pointer on events array, output value

 * Returns the number of completed events and sets a pointer
 * on events array.  This function does not update the internal
 * ring buffer, only reads head and tail.  When @events has been
 * processed io_getevents_commit() must be called.
 */
static inline unsigned int io_getevents_peek(io_context_t ctx,
                                             struct io_event **events)
{
    struct aio_ring *ring = (struct aio_ring *)ctx;
    unsigned int head = ring->head, tail = ring->tail;
    unsigned int nr;

    /*@head和@tail之间的是完成的事件*/
    nr = tail >= head ? tail - head : ring->nr - head;
    *events = ring->io_events + head;
    /* To avoid speculative loads of s->events[i] before observing tail.
       Paired with smp_wmb() inside linux/fs/aio.c: aio_complete(). */
    smp_rmb();

    return nr;
}

static void qemu_laio_process_completions_and_submit(LinuxAioState *s)
{
    /*处理已完成的事件*/
    qemu_laio_process_completions(s);

    /*继续提交新的AIO请求*/
    aio_context_acquire(s->aio_context);
    if (!s->io_q.plugged && !QSIMPLEQ_EMPTY(&s->io_q.pending)) {
        ioq_submit(s);
    }
    aio_context_release(s->aio_context);
}

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
        /*依次处理已完成的事件*/
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

void qemu_bh_schedule(QEMUBH *bh)
{
    AioContext *ctx;

    ctx = bh->ctx;
    bh->idle = 0;
    /* The memory barrier implicit in atomic_xchg makes sure that:
     * 1. idle & any writes needed by the callback are done before the
     *    locations are read in the aio_bh_poll.
     * 2. ctx is loaded before scheduled is set and the callback has a chance
     *    to execute.
     */
    /*atomic_xchg：返回旧值，存入新值。如果旧值是0，则调用aio_notify*/
    if (atomic_xchg(&bh->scheduled, 1) == 0) {
        aio_notify(ctx);
    }
}

void aio_notify(AioContext *ctx)
{
    /* Write e.g. bh->scheduled before reading ctx->notify_me.  Pairs
     * with atomic_or in aio_ctx_prepare or atomic_add in aio_poll.
     */
    smp_mb();
    if (ctx->notify_me) {
        event_notifier_set(&ctx->notifier);
        atomic_mb_set(&ctx->notified, true);
    }
}

/**
 * io_getevents_advance_and_peek:
 * @ctx: AIO context
 * @events: pointer on events array, output value
 * @nr: the number of events on which head should be advanced
 *
 * Advances head of a ring buffer and returns number of elements left.
 */
static inline unsigned int
io_getevents_advance_and_peek(io_context_t ctx,
                              struct io_event **events,
                              unsigned int nr)
{
    /*@aio_ring::head前进@nr个元素*/
    io_getevents_commit(ctx, nr);
    /*返回剩余的已完成AIO事件个数，并返回这些事件列表@events*/
    return io_getevents_peek(ctx, events);
}

/**
 * io_getevents_commit:
 * @ctx: AIO context
 * @nr: the number of events on which head should be advanced
 *
 * Advances head of a ring buffer.
 */
static inline void io_getevents_commit(io_context_t ctx, unsigned int nr)
{
    struct aio_ring *ring = (struct aio_ring *)ctx;

    if (nr) {
        ring->head = (ring->head + nr) % ring->nr;
    }
}

/*
 * Completes an AIO request (calls the callback and frees the ACB).
 */
static void qemu_laio_process_completion(struct qemu_laiocb *laiocb)
{
    int ret;

    ret = laiocb->ret;
    if (ret != -ECANCELED) {
        if (ret == laiocb->nbytes) {
            ret = 0;
        } else if (ret >= 0) {
            /* Short reads mean EOF, pad with zeros. */
            if (laiocb->is_read) {
                qemu_iovec_memset(laiocb->qiov, ret, 0,
                    laiocb->qiov->size - ret);
            } else {
                ret = -ENOSPC;
            }
        }
    }

    laiocb->ret = ret;
    if (laiocb->co) {
        /* If the coroutine is already entered it must be in ioq_submit() and
         * will notice laio->ret has been filled in when it eventually runs
         * later.  Coroutines cannot be entered recursively so avoid doing
         * that!
         */
        if (!qemu_coroutine_entered(laiocb->co)) {
            aio_co_wake(laiocb->co);
        }
    } else {
        laiocb->common.cb(laiocb->common.opaque, ret);
        qemu_aio_unref(laiocb);
    }
}

static void qemu_laio_completion_bh(void *opaque)
{
    LinuxAioState *s = opaque;

    qemu_laio_process_completions_and_submit(s);
}

static void qemu_laio_completion_cb(EventNotifier *e)
{
    LinuxAioState *s = container_of(e, LinuxAioState, e);

    if (event_notifier_test_and_clear(&s->e)) {
        qemu_laio_process_completions_and_submit(s);
    }
}
```