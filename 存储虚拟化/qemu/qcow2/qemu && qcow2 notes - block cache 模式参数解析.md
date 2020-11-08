[KVM cache modes overview](http://www.ilsistemista.net/index.php/virtualization/23-kvm-storage-performance-and-cache-settings-on-red-hat-enterprise-linux-62.html?start=2) 

```
这篇文章中以图示加文字的方式描述了qemu中各cache模式(nocache, writeback,
writethrough)等工作方式，其中还谈到了支持barrier和不支持barrier情况下，
各cache模式工作方式的不同，当然这篇文章中讲到的qemu可能是比较早期的版
本(rhel6.2)。
```

[QEMU version 2.10.92 User Documentation](https://qemu.weilnetz.de/doc/qemu-doc.html#Block-device-options)
```
这是关于Qemu V2.10.92的使用手册，其中谈到了关于-drive的选项，其中关于cache的设置：
cache=cache
cache is "none", "writeback", "unsafe", "directsync" or "writethrough" and controls how the host cache is used to access block data. This is a shortcut that sets the cache.direct and cache.no-flush options (as in -blockdev), and additionally cache.writeback, which provides a default for the write-cache option of block guest devices (as in -device). The modes correspond to the following settings:
             │ cache.writeback   cache.direct   cache.no-flush
─────────────┼─────────────────────────────────────────────────
writeback    │ on                off            off
none         │ on                on             off
writethrough │ off               off            off
directsync   │ off               on             off
unsafe       │ on                off            on
The default mode is cache=writeback
```

```
这里只是给了一个结论，那么代码中是怎样体现的呢？如下：
DriveInfo *drive_new(QemuOpts *all_opts, BlockInterfaceType block_default_type)
{
    ......
    
    /*获取cache选项设置*/
    value = qemu_opt_get(all_opts, "cache");
    
    /* 从cache选项设置中解析出@flags和@writethrough相关的配置，为后面关于cache.direct、
     * cache.no-flush和cache.writeback的设置做准备，关于bdrv_parse_cache_mode的分析见
     * 下面
     */
    if (bdrv_parse_cache_mode(value, &flags, &writethrough) != 0) {
        ......
    }
    
    /* 如果@writethrough没有被设置，就设置cache.writeback为true，否则设置为false，
     * 结合bdrv_parse_cache_mode，可知对于off/none/writeback/unsafe模式都会设置
     * cache.writeback=true，对于writethrough/directsync则设置cache.writeback=false
     */
    if (!qemu_opt_get(all_opts, BDRV_OPT_CACHE_WB)) {
        qemu_opt_set_bool(all_opts, BDRV_OPT_CACHE_WB,
                          !writethrough, &error_abort);
    }
    
    /* 如果@flags中设置了BDRV_NO_CACHE，则设置cache.direct为true，否则设置为false，
     * 结合bdrv_parse_cache_mode，可知对于off/none/directsync模式会设置cache.direct
     * =true，对于writeback/writethrough/unsafe则设置cache.direct=false
     */
    if (!qemu_opt_get(all_opts, BDRV_OPT_CACHE_DIRECT)) {
        qemu_opt_set_bool(all_opts, BDRV_OPT_CACHE_DIRECT,
                          !!(flags & BDRV_O_NOCACHE), &error_abort);
    }
    
    /* 如果@flags中设置了BDRV_O_NO_FLUSH，则设置cache.no_flush为true，否则设置为false，
     * 结合bdrv_parse_cache_mode，可知对于unsafe模式会设置cache.no_flush=true，
     * 对于off/none/directsyncwriteback/writethrough则设置cache.flush=false
     */
    if (!qemu_opt_get(all_opts, BDRV_OPT_CACHE_NO_FLUSH)) {
        qemu_opt_set_bool(all_opts, BDRV_OPT_CACHE_NO_FLUSH,
                          !!(flags & BDRV_O_NO_FLUSH), &error_abort);
    }
    
    ......
    
    /*见下面*/
    blk = blockdev_init(filename, bs_opts, &local_err);
    
    ......
}

int bdrv_parse_cache_mode(const char *mode, int *flags, bool *writethrough)
{
    *flags &= ~BDRV_O_CACHE_MASK;

    if (!strcmp(mode, "off") || !strcmp(mode, "none")) {
        *writethrough = false;
        *flags |= BDRV_O_NOCACHE;
    } else if (!strcmp(mode, "directsync")) {
        *writethrough = true;
        *flags |= BDRV_O_NOCACHE;
    } else if (!strcmp(mode, "writeback")) {
        *writethrough = false;
    } else if (!strcmp(mode, "unsafe")) {
        *writethrough = false;
        *flags |= BDRV_O_NO_FLUSH;
    } else if (!strcmp(mode, "writethrough")) {
        *writethrough = true;
    } else {
        return -1;
    }

    return 0;
}

static BlockBackend *blockdev_init(const char *file, QDict *bs_opts, Error **errp)
{
    QemuOpts *opts;
    
    /*创建@opts，采用@qemu_common_drive_opts中指定的通用参数*/
    opts = qemu_opts_create(&qemu_common_drive_opts, id, 1, &error);
    /*将@bs_opts纳入到@opts中*/
    qemu_opts_absorb_qdict(opts, bs_opts, &error);
    
    /* 从@opts中解析出@writethrough，如果cache.writeback=true，
     * 则writethrough=false，否则writethrough=true
     */
    writethrough = !qemu_opt_get_bool(opts, BDRV_OPT_CACHE_WB, true);
    
    ......
    
    /*注意，这里使用的是@bs_opts*/
    blk = blk_new_open(file, NULL, bs_opts, bdrv_flags, errp);
    
    ......
    
    /*设置BlockBackend::enable_write_cache标识*/
    blk_set_enable_write_cache(blk, !writethrough);
}

那么BlockBackend::enable_write_cache在哪里使用呢？查看其被以下函数调用：
blk_co_pwritev [写IO相关，重点关注]
blk_enable_write_cache [查询是否设置了enable_write_cache]
blk_set_enable_write_cache [设置enable_write_cache]
blk_save_vmstate [保存vmstate相关，暂时不关注]

int coroutine_fn blk_co_pwritev(BlockBackend *blk, int64_t offset,
                                unsigned int bytes, QEMUIOVector *qiov,
                                BdrvRequestFlags flags)
{
    int ret;
    BlockDriverState *bs = blk_bs(blk);

    ......
    
    /*如果没有设置enable_write_cache，则设置上BDRV_REQ_FUA标识*/
    if (!blk->enable_write_cache) {
        flags |= BDRV_REQ_FUA;
    }

    /*该函数中要关注@flags*/
    ret = bdrv_co_pwritev(blk->root, offset, bytes, qiov, flags);
    bdrv_dec_in_flight(bs);
    return ret;
}

int coroutine_fn bdrv_co_pwritev(BdrvChild *child,
    int64_t offset, unsigned int bytes, QEMUIOVector *qiov,
    BdrvRequestFlags flags)
{
    ......
    
    /*@flags被原封不动传递下来*/
    ret = bdrv_aligned_pwritev(child, &req, offset, bytes, align,
                           use_local_qiov ? &local_qiov : qiov,
                           flags);

    ......                           
}

static int coroutine_fn bdrv_aligned_pwritev(BdrvChild *child,
    BdrvTrackedRequest *req, int64_t offset, unsigned int bytes,
    int64_t align, QEMUIOVector *qiov, int flags)
{
    BlockDriverState *bs = child->bs;
    BlockDriver *drv = bs->drv;

    /*detect zero相关的处理*/
    if (!ret && bs->detect_zeroes != BLOCKDEV_DETECT_ZEROES_OPTIONS_OFF &&
        !(flags & BDRV_REQ_ZERO_WRITE) && drv->bdrv_co_pwrite_zeroes &&
        qemu_iovec_is_zero(qiov)) {
        flags |= BDRV_REQ_ZERO_WRITE;
        if (bs->detect_zeroes == BLOCKDEV_DETECT_ZEROES_OPTIONS_UNMAP) {
            flags |= BDRV_REQ_MAY_UNMAP;
        }
    }

    if (ret < 0) {
        /* Do nothing, write notifier decided to fail this request */
    } else if (flags & BDRV_REQ_ZERO_WRITE) {
        /*全0写入处理*/
        bdrv_debug_event(bs, BLKDBG_PWRITEV_ZERO);
        ret = bdrv_co_do_pwrite_zeroes(bs, offset, bytes, flags);
    } else if (flags & BDRV_REQ_WRITE_COMPRESSED) {
        /*压缩写处理*/
        ret = bdrv_driver_pwritev_compressed(bs, offset, bytes, qiov);
    } else if (bytes <= max_transfer) {
        /* 要写入的数据总量@bytes小于@max_transfer则直接写入，下面会分析之，
         * 如果cache.write_back未被设置，则@flags中会被设置BDRV_REQ_FUA标识
         */
        bdrv_debug_event(bs, BLKDBG_PWRITEV);
        ret = bdrv_driver_pwritev(bs, offset, bytes, qiov, flags);
    } else {
        /*分片写入*/
        bdrv_debug_event(bs, BLKDBG_PWRITEV);
        while (bytes_remaining) {
            int num = MIN(bytes_remaining, max_transfer);
            QEMUIOVector local_qiov;
            /*@local_flags直接用@flags初始化，将被传递给bdrv_driver_pwritev*/
            int local_flags = flags;

            /* 结合上面的“local_flags = flags”赋值，和这里的if语句可知：当且仅当是
             * 最后一个分片且@flags中设置了BDRV_REQ_FUA标识的情况下@local_flags中才
             * 会设置BDRV_REQ_FUA标识，而根据前面的分析没有cache.write_back未被设置
             * 的情况下@flags中就会被设置BDRV_REQ_FUA标识，所以可以这么说，当
             * cache.write_back未被设置的情况下，在分片写逻辑中只有最后一个分片的写
             * 需要模拟FUA(结合bdrv_driver_pwritev的实现可知，模拟FUA就是执行flush)
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

    ......

    return ret;
}

static int coroutine_fn bdrv_driver_pwritev(BlockDriverState *bs,
                                            uint64_t offset, uint64_t bytes,
                                            QEMUIOVector *qiov, int flags)
{
    BlockDriver *drv = bs->drv;
    int64_t sector_num;
    unsigned int nb_sectors;
    int ret;

    assert(!(flags & ~BDRV_REQ_MASK));

    /* 对于qcow2来说，.bdrv_co_pwritev = qcow2_co_pwritev，但是在qcow2_co_pwritev，
     * 传递进去的(flags & bs->supported_write_flags)未被使用，所以是否设置了
     * BDRV_REQ_FUA不影响其逻辑
     */
    if (drv->bdrv_co_pwritev) {
        ret = drv->bdrv_co_pwritev(bs, offset, bytes, qiov,
                                   flags & bs->supported_write_flags);
        /* 对于qcow2来说，bs->supported_write_flags未被设置，所以@flags跟传递进来的
         * 时候一样
         */
        flags &= ~bs->supported_write_flags;
        goto emulate_flags;
    }

    ......

emulate_flags:
    /*如果@flags中设置了BDRV_REQ_FUA，则执行flush*/
    if (ret == 0 && (flags & BDRV_REQ_FUA)) {
        ret = bdrv_co_flush(bs);
    }

    return ret;
}

截至目前，我们已经知道：
cache.write_back的作用：用于控制BlockBackend中enable_write_cache的设置，如果cache.write_back设置为true，则BlockBackend::enable_write_cache设置为true，否则BlockBackend::enable_write_cache设置为false；进而通过BlockBackend::enable_write_cache是否被设置来控制是否设置BDRV_REQ_FUA标识，如果Backend::enable_write_cache为true则不设置BDRV_REQ_FUA标识，否则设置BDRV_REQ_FUA标识；进一步在执行写请求过程中，在blk_co_pwritev -> bdrv_aligned_pwritev -> bdrv_driver_pwritev中通过BDRV_REQ_FUA标识控制在写请求执行完毕是否执行flush。最终可以得出这样的结论，如果cache.write_back被设置为false，则在写请求执行完毕之后执行flush，如果写请求需要分片写入则只在最后一个分片执行完毕之后执行flush。

下面接着看看cache.direct起到什么作用呢？
BlockBackend *blk_new_open(const char *filename, const char *reference,
                           QDict *options, int flags, Error **errp)
    - bdrv_open(const char *filename, const char *reference,
                            QDict *options, int flags, Error **errp)
        - bdrv_open_inherit(const char *filename, const char *reference,
                            QDict *options, int flags,
                            BlockDriverState *parent,
                            const BdrvChildRole *child_role,
                            Error **errp)
            - bs->open_flags = flags [这一句很重要，所以我专门提取出来放在这里]
            - bdrv_open_common(BlockDriverState *bs, BlockBackend *file,
                                QDict *options, Error **errp)

static int bdrv_open_common(BlockDriverState *bs, BlockBackend *file,
                            QDict *options, Error **errp)
{
    QemuOpts *opts;

    /*创建@opts，采用@bdrv_runtime_opts中指定的参数*/
    opts = qemu_opts_create(&bdrv_runtime_opts, NULL, 0, &error_abort);
    /*将@options中的参数纳入到@opts中*/
    qemu_opts_absorb_qdict(opts, options, &local_err);

    /*设置BlockDriverState::open_flags标识*/
    update_flags_from_options(&bs->open_flags, opts);
    
    ......
    
    /*将@bs->open_flags转换为@open_flags*/
    open_flags = bdrv_open_flags(bs, bs->open_flags);
    /*见bdrv_open_driver，这里的@options和@open_flags中都有关于cache相关的配置*/
    ret = bdrv_open_driver(bs, drv, node_name, options, open_flags, errp);
}

static void update_flags_from_options(int *flags, QemuOpts *opts)
{
    *flags &= ~BDRV_O_CACHE_MASK;

    /*如果@opts中指定了cache.no_flush，则在@flags中设置BDRV_O_NO_FLUSH标识*/
    assert(qemu_opt_find(opts, BDRV_OPT_CACHE_NO_FLUSH));
    if (qemu_opt_get_bool(opts, BDRV_OPT_CACHE_NO_FLUSH, false)) {
        *flags |= BDRV_O_NO_FLUSH;
    }

    /*如果@opts中指定了cache.direct，则在@flags中设置BDRV_O_NOCACHE标识*/
    assert(qemu_opt_find(opts, BDRV_OPT_CACHE_DIRECT));
    if (qemu_opt_get_bool(opts, BDRV_OPT_CACHE_DIRECT, false)) {
        *flags |= BDRV_O_NOCACHE;
    }

    /*如果@opts中没有指定readonly，则在@flags中设置BDRV_O_RDWR标识*/
    *flags &= ~BDRV_O_RDWR;
    assert(qemu_opt_find(opts, BDRV_OPT_READ_ONLY));
    if (!qemu_opt_get_bool(opts, BDRV_OPT_READ_ONLY, false)) {
        *flags |= BDRV_O_RDWR;
    }
}

static int bdrv_open_driver(BlockDriverState *bs, BlockDriver *drv,
                            const char *node_name, QDict *options,
                            int open_flags, Error **errp)
{
    bs->drv = drv;
    bs->read_only = !(bs->open_flags & BDRV_O_RDWR);
    bs->opaque = g_malloc0(drv->instance_size);

    /* 对于qcow2来说，提供了.bdrv_open = qcow2_open，将传递进来的@options和
     * @open_flags进一步传递给了qcow2_open
     */
    if (drv->bdrv_file_open) {
        assert(!drv->bdrv_needs_filename || bs->filename[0]);
        ret = drv->bdrv_file_open(bs, options, open_flags, &local_err);
    } else if (drv->bdrv_open) {
        ret = drv->bdrv_open(bs, options, open_flags, &local_err);
    } else {
        ret = 0;
    }

    ......
}

static int qcow2_open(BlockDriverState *bs, QDict *options, int flags, Error **errp)
{
    /* file protocol打开镜像文件，因为这里才是真正的打开文件的逻辑，所以cache相关
     * 的配置通过@options传递进来，并在打开的时候体现，@flags在这里没用到
     */
    bs->file = bdrv_open_child(NULL, options, "file", bs, &child_file, false, errp);
    if (!bs->file) {
        return -EINVAL;
    }

    /*qcow2自身与打开相关的逻辑，这里用到了传递进来的@options和@flags，但是与cache关系不大*/
    return qcow2_do_open(bs, options, flags, errp);
}

bdrv_open_child(const char *filename,
               QDict *options, const char *bdref_key,
               BlockDriverState *parent,
               const BdrvChildRole *child_role,
               bool allow_none, Error **errp)
    - bdrv_open_child_bs(const char *filename, QDict *options, const char *bdref_key,
                       BlockDriverState *parent, const BdrvChildRole *child_role,
                       bool allow_none, Error **errp)
        - bdrv_open_inherit(const char *filename,
                           const char *reference,
                           QDict *options, int flags,
                           BlockDriverState *parent,
                           const BdrvChildRole *child_role,
                           Error **errp)
            - bdrv_open_common(BlockDriverState *bs, BlockBackend *file,
                                QDict *options, Error **errp)

再次到达bdrv_open_common，熟悉的面孔，前面已经分析过了，其中会通过转换@options到@open_flags并传递给bdrv_open_driver。
static BlockDriverState *bdrv_open_inherit(const char *filename,
                                           const char *reference,
                                           QDict *options, int flags,
                                           BlockDriverState *parent,
                                           const BdrvChildRole *child_role,
                                           Error **errp)
{
    /*child BlockDriverState*/
    BlockDriverState *bs;
    
    /*创建child BlockDriverState*/
    bs = bdrv_new();
    
    /*继承@parent的@options和@flags*/
    if (child_role) {
        bs->inherits_from = parent;
        child_role->inherit_options(&flags, options,
                                    parent->open_flags, parent->options);
    }
    
    /*将@filename和@flags添加到options中*/
    ret = bdrv_fill_options(&options, filename, &flags, &local_err);
    
    /*设置child BlockDriverState的open_flags和options*/
    bs->open_flags = flags;
    bs->options = options;
    
    /*打开child Block Driver，注意这里传递的@options是继承自@parent的*/
    ret = bdrv_open_common(bs, file, options, &local_err);
    
    ......
}

bdrv_open_common(BlockDriverState *bs, BlockBackend *file,
                QDict *options, Error **errp)
    - bdrv_open_driver(BlockDriverState *bs, BlockDriver *drv,
                        const char *node_name, QDict *options,
                        int open_flags, Error **errp)    
                        
如果创建qcow2文件的时候采用本地文件系统作为protocol layer，则这里对应的是file protocol，所以@drv实际上是bdrv_file，它提供了.bdrv_file_open = raw_open，那么进入raw_open看看打开的时候发生了什么？记住，传递的参数@options和@flags中都有关于cache相关的配置信息哦。
static int raw_open(BlockDriverState *bs, QDict *options, int flags, Error **errp)
{
    BDRVRawState *s = bs->opaque;

    s->type = FTYPE_FILE;
    return raw_open_common(bs, options, flags, 0, errp);
}

static int raw_open_common(BlockDriverState *bs, QDict *options,
                           int bdrv_flags, int open_flags, Error **errp)
{
    BDRVRawState *s = bs->opaque;
    QemuOpts *opts;
    Error *local_err = NULL;
    const char *filename = NULL;
    BlockdevAioOptions aio, aio_default;
    int fd, ret;
    struct stat st;
    OnOffAuto locking;

    /*创建@opts，采用raw_runtime_opts初始化之*/
    opts = qemu_opts_create(&raw_runtime_opts, NULL, 0, &error_abort);
    /*将@options中的配置纳入到@opts中*/
    qemu_opts_absorb_qdict(opts, options, &local_err);

    filename = qemu_opt_get(opts, "filename");

    /*文件名*/
    ret = raw_normalize_devicepath(&filename);

    /*AIO相关设置*/
    aio_default = (bdrv_flags & BDRV_O_NATIVE_AIO)
                  ? BLOCKDEV_AIO_OPTIONS_NATIVE
                  : BLOCKDEV_AIO_OPTIONS_THREADS;
    aio = qapi_enum_parse(&BlockdevAioOptions_lookup,
                          qemu_opt_get(opts, "aio"),
                          aio_default, &local_err);
    s->use_linux_aio = (aio == BLOCKDEV_AIO_OPTIONS_NATIVE);

    /*锁相关设置，略过*/

    /* 根据@bdrv_flags设置打开标识，如果@bdrv_flags中指定了BDRV_O_NOCACHE，
     * 则打开标识中设置O_DIRECT
     */
    s->open_flags = open_flags;
    raw_parse_flags(bdrv_flags, &s->open_flags);

    /*打开*/
    s->fd = -1;
    fd = qemu_open(filename, s->open_flags, 0644);
    s->fd = fd;

    ......
}

static void raw_parse_flags(int bdrv_flags, int *open_flags)
{
    assert(open_flags != NULL);

    *open_flags |= O_BINARY;
    *open_flags &= ~O_ACCMODE;
    if (bdrv_flags & BDRV_O_RDWR) {
        *open_flags |= O_RDWR;
    } else {
        *open_flags |= O_RDONLY;
    }

    /*千呼万唤始出来啊，这里终于看到BDRV_O_NOCACHE的使用了*/
    /* Use O_DSYNC for write-through caching, no flags for write-back caching,
     * and O_DIRECT for no caching. */
    if ((bdrv_flags & BDRV_O_NOCACHE)) {
        *open_flags |= O_DIRECT;
    }
}

int qemu_open(const char *name, int flags, ...)
{
    int ret;
    int mode = 0;

    ......

    /*打开，直接调用open*/
#ifdef O_CLOEXEC
    ret = open(name, flags | O_CLOEXEC, mode);
#else
    ret = open(name, flags, mode);
    if (ret >= 0) {
        qemu_set_cloexec(ret);
    }
#endif

#ifdef O_DIRECT
    if (ret == -1 && errno == EINVAL && (flags & O_DIRECT)) {
        error_report("file system may not support O_DIRECT");
        errno = EINVAL; /* in case it was clobbered */
    }
#endif /* O_DIRECT */

    return ret;
}

截至目前，我们看到了：
cache.write_back的作用：用于控制BlockBackend中enable_write_cache的设置，如果cache.write_back设置为true，则BlockBackend::enable_write_cache设置为true，否则BlockBackend::enable_write_cache设置为false；进而通过BlockBackend::enable_write_cache是否被设置来控制是否设置BDRV_REQ_FUA标识，如果Backend::enable_write_cache为true则不设置BDRV_REQ_FUA标识，否则设置BDRV_REQ_FUA标识；进一步在执行写请求过程中，在blk_co_pwritev -> bdrv_aligned_pwritev -> bdrv_driver_pwritev中通过BDRV_REQ_FUA标识控制在写请求执行完毕是否执行flush。最终可以得出这样的结论，如果cache.write_back被设置为false，则在写请求执行完毕之后执行flush，如果写请求需要分片写入则只在最后一个分片执行完毕之后执行flush。
cache.direct的作用：用于控制打开镜像文件的时候，是否采用O_DIRECT标识，如果cache.direct设置为true，则采用O_DIRECT标识，否则不采用。

但是，但是cache.no_flush还没看到，且慢，下面为您道来。
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

对于protocol layer为file的qcow2来说，在bdrv_driver_pwritev -> bdrv_co_flush中首先调用的是bdrv_qcow2的driver层的flush相关的接口，然后递归调用bdrv_co_flush，会再次调用protocol layer的flush相关的接口，完成flush逻辑。总之在bdrv_driver_pwritev -> bdrv_co_flush逻辑中会依次调用qcow2_co_flush_to_os和raw_aio_flush。

static coroutine_fn int qcow2_co_flush_to_os(BlockDriverState *bs)
{
    /*本函数用于将qcow2 cache中的数据flush到host os（的镜像文件）中*/
    BDRVQcow2State *s = bs->opaque;
    int ret;

    qemu_co_mutex_lock(&s->lock);
    /*flush L2 table cache到镜像文件中*/
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

至此，对于cache.write_back、cache.direct、cache.no_flush作用有以下认知：
cache.write_back的作用：用于控制BlockBackend中enable_write_cache的设置，如果
cache.write_back设置为true，则BlockBackend::enable_write_cache设置为true，否
则BlockBackend::enable_write_cache设置为false；进而通过BlockBackend::enable_write_cache
是否被设置来控制是否设置BDRV_REQ_FUA标识，如果Backend::enable_write_cache为
true则不设置BDRV_REQ_FUA标识，否则设置BDRV_REQ_FUA标识；进一步在执行写请求过
程中，在blk_co_pwritev -> bdrv_aligned_pwritev -> bdrv_driver_pwritev
中通过BDRV_REQ_FUA标识控制在写请求执行完毕是否执行flush。最终可以得出这样的
结论，如果cache.write_back被设置为false，则在写请求执行完毕之后执行flush，
如果写请求需要分片写入则只在最后一个分片执行完毕之后执行flush。

cache.direct的作用：用于控制打开镜像文件的时候，是否采用O_DIRECT标识，如果
cache.direct设置为true，则采用O_DIRECT标识，否则不采用。

cache.no_flush的作用：设置BDRV_O_NO_FLUSH标识，如果由于cache.write_back未被设
置而导致调用了bdrv_co_flush，在bdrv_co_flush中可以通过是否设置了BDRV_O_NO_FLUSH
标识来控制是否执行从host os到物理硬盘的flush（但是如果driver提供了.bdrv_co_flush
接口函数，那么会flush任何一层的cache，直到数据写入物理硬盘，即使指定了
BDRV_O_NO_FLUSH标识，也无能为力，否则driver会尝试先flush到host os，然后再从
host os flush到物理硬盘，如果指定了BDRV_O_NO_FLUSH标识，它可以控制从host os 
flush到物理硬盘这一逻辑是否执行）。
```