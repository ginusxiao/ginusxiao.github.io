# 创建img

```
static int img_create(int argc, char **argv)
{
    int c;
    uint64_t img_size = -1;
    const char *fmt = "raw";
    const char *base_fmt = NULL;
    const char *filename;
    const char *base_filename = NULL;
    char *options = NULL;
    Error *local_err = NULL;
    bool quiet = false;
    int flags = 0;

    /*
     * 参数解析，获取如下参数：
     * filename: 即将创建的img的文件名
     * fmt: 即将创建的img的格式（如raw，qcow2等）
     * base_filename: 基础img的文件名（即将创建的img基于此img，以此img作为基础镜像）
     * base_fmt: 基础img的格式
     * img_size: img大小
     * flags: open相关标识
     * quiet: 
     */
    ......

    /*详见bdrv_img_create*/
    bdrv_img_create(filename, fmt, base_filename, base_fmt,
                    options, img_size, flags, quiet, &local_err);
    if (local_err) {
        error_reportf_err(local_err, "%s: ", filename);
        goto fail;
    }

    g_free(options);
    return 0;

fail:
    g_free(options);
    return 1;
}
```


```
void bdrv_img_create(const char *filename, const char *fmt,
                     const char *base_filename, const char *base_fmt,
                     char *options, uint64_t img_size, int flags, bool quiet,
                     Error **errp)
{
    QemuOptsList *create_opts = NULL;
    QemuOpts *opts = NULL;
    const char *backing_fmt, *backing_file;
    int64_t size;
    BlockDriver *drv, *proto_drv;
    Error *local_err = NULL;
    int ret = 0;

    /* 根据@fmt找到相应的block driver，各种不同的格式，如qcow、qcow2、raw、rbd等都有其自己的block driver，通过bdrv_register注册，bdrv_find_format查找所有注册的block driver，找到匹配@fmt的那个*/
    drv = bdrv_find_format(fmt);
    if (!drv) {
        error_setg(errp, "Unknown file format '%s'", fmt);
        return;
    }

    /*某些block driver中还会定义protocol name，如nfs、rbd、nbd等，这里从@filename中解析出protocol，然后根据protocol查找相应的block driver，从后面对@proto_drv的使用来看，proto_drv可以提供它自己的创建img的选项*/
    proto_drv = bdrv_find_protocol(filename, true, errp);
    if (!proto_drv) {
        return;
    }

    if (!drv->create_opts) {
        error_setg(errp, "Format driver '%s' does not support image creation",
                   drv->format_name);
        return;
    }

    if (!proto_drv->create_opts) {
        error_setg(errp, "Protocol driver '%s' does not support image creation",
                   proto_drv->format_name);
        return;
    }

    /*合并@drv和proto_drv中提供的创建选项到@create_opts中*/
    create_opts = qemu_opts_append(create_opts, drv->create_opts);
    create_opts = qemu_opts_append(create_opts, proto_drv->create_opts);

    /* Create parameter list with default values(因为@create_opts中的选项来自于@drv和@proto_drv中的默认值)，对于qcow2来说，其默认的创建选项请参考qcow2_create_opts。
    */
    opts = qemu_opts_create(create_opts, NULL, 0, &error_abort);
    /*在@opts中设置img size*/
    qemu_opt_set_number(opts, BLOCK_OPT_SIZE, img_size, &error_abort);

    /* Parse -o options */
    if (options) {
        qemu_opts_do_parse(opts, options, NULL, &local_err);
        if (local_err) {
            error_report_err(local_err);
            local_err = NULL;
            error_setg(errp, "Invalid options for file format '%s'", fmt);
            goto out;
        }
    }

    /*在@opts中设置backfile*/
    if (base_filename) {
        qemu_opt_set(opts, BLOCK_OPT_BACKING_FILE, base_filename, &local_err);
        if (local_err) {
            error_setg(errp, "Backing file not supported for file format '%s'",
                       fmt);
            goto out;
        }
    }

    /*在@opts中设置backfile format*/
    if (base_fmt) {
        qemu_opt_set(opts, BLOCK_OPT_BACKING_FMT, base_fmt, &local_err);
        if (local_err) {
            error_setg(errp, "Backing file format not supported for file "
                             "format '%s'", fmt);
            goto out;
        }
    }

    backing_file = qemu_opt_get(opts, BLOCK_OPT_BACKING_FILE);
    if (backing_file) {
        if (!strcmp(filename, backing_file)) {
            error_setg(errp, "Error: Trying to create an image with the "
                             "same filename as the backing file");
            goto out;
        }
    }

    backing_fmt = qemu_opt_get(opts, BLOCK_OPT_BACKING_FMT);

    /* The size for the image must always be specified, unless we have a backing
     * file and we have not been forbidden from opening it(BDRV_O_NO_BACKING). */
    size = qemu_opt_get_size(opts, BLOCK_OPT_SIZE, 0);
    if (backing_file && !(flags & BDRV_O_NO_BACKING)) {
        BlockDriverState *bs;
        char *full_backing = g_new0(char, PATH_MAX);
        int back_flags;
        QDict *backing_options = NULL;

        bdrv_get_full_backing_filename_from_filename(filename, backing_file,
                                                     full_backing, PATH_MAX,
                                                     &local_err);
        if (local_err) {
            g_free(full_backing);
            goto out;
        }

        /* backing files always opened read-only */
        back_flags = flags;
        back_flags &= ~(BDRV_O_RDWR | BDRV_O_SNAPSHOT | BDRV_O_NO_BACKING);

        if (backing_fmt) {
            backing_options = qdict_new();
            qdict_put_str(backing_options, "driver", backing_fmt);
        }

        bs = bdrv_open(full_backing, NULL, backing_options, back_flags,
                       &local_err);
        g_free(full_backing);
        if (!bs && size != -1) {
            /* Couldn't open BS, but we have a size, so it's nonfatal */
            warn_reportf_err(local_err,
                            "Could not verify backing image. "
                            "This may become an error in future versions.\n");
            local_err = NULL;
        } else if (!bs) {
            /* Couldn't open bs, do not have size (不能打开backing file，且img的创建参数中没有指定size，则出错)*/
            error_append_hint(&local_err,
                              "Could not open backing image to determine size.\n");
            goto out;
        } else {
            /*可以打开backing file，且img的创建参数中没有指定size，则读取backing file的大小，作为创建的img的大小，并在@opts中设置img size*/
            if (size == -1) {
                /* Opened BS, have no size */
                size = bdrv_getlength(bs);
                if (size < 0) {
                    error_setg_errno(errp, -size, "Could not get size of '%s'",
                                     backing_file);
                    bdrv_unref(bs);
                    goto out;
                }
                
                qemu_opt_set_number(opts, BLOCK_OPT_SIZE, size, &error_abort);
            }
            bdrv_unref(bs);
        }
    } /* (backing_file && !(flags & BDRV_O_NO_BACKING)) */

    if (size == -1) {
        error_setg(errp, "Image creation needs a size parameter");
        goto out;
    }

    if (!quiet) {
        printf("Formatting '%s', fmt=%s ", filename, fmt);
        qemu_opts_print(opts, " ");
        puts("");
    }

    /*详见bdrv_create*/
    ret = bdrv_create(drv, filename, opts, &local_err);

    if (ret == -EFBIG) {
        /* This is generally a better message than whatever the driver would
         * deliver (especially because of the cluster_size_hint), since that
         * is most probably not much different from "image too large". */
        const char *cluster_size_hint = "";
        if (qemu_opt_get_size(opts, BLOCK_OPT_CLUSTER_SIZE, 0)) {
            cluster_size_hint = " (try using a larger cluster size)";
        }
        error_setg(errp, "The image size is too large for file format '%s'"
                   "%s", fmt, cluster_size_hint);
        error_free(local_err);
        local_err = NULL;
    }

out:
    qemu_opts_del(opts);
    qemu_opts_free(create_opts);
    error_propagate(errp, local_err);
}
```

```
int bdrv_create(BlockDriver *drv, const char* filename,
                QemuOpts *opts, Error **errp)
{
    int ret;

    Coroutine *co;
    CreateCo cco = {
        .drv = drv,
        .filename = g_strdup(filename),
        .opts = opts,
        .ret = NOT_DONE,
        .err = NULL,
    };

    if (!drv->bdrv_create) {
        error_setg(errp, "Driver '%s' does not support image creation", drv->format_name);
        ret = -ENOTSUP;
        goto out;
    }

    /*无论是在新创建的协程还是已有的协程中执行，真正的执行函数是bdrv_create_co_entry*/
    if (qemu_in_coroutine()) {
        /* Fast-path if already in coroutine context */
        bdrv_create_co_entry(&cco);
    } else {
        co = qemu_coroutine_create(bdrv_create_co_entry, &cco);
        qemu_coroutine_enter(co);
        while (cco.ret == NOT_DONE) {
            aio_poll(qemu_get_aio_context(), true);
        }
    }

    ret = cco.ret;
    if (ret < 0) {
        if (cco.err) {
            error_propagate(errp, cco.err);
        } else {
            error_setg_errno(errp, -ret, "Could not create image");
        }
    }

out:
    g_free(cco.filename);
    return ret;
}
```

```
static void coroutine_fn bdrv_create_co_entry(void *opaque)
{
    Error *local_err = NULL;
    int ret;

    CreateCo *cco = opaque;
    assert(cco->drv);

    /*调用相应driver的create接口，比如bdrv_qcow2::qcow2_create*/
    ret = cco->drv->bdrv_create(cco->filename, cco->opts, &local_err);
    error_propagate(&cco->err, local_err);
    cco->ret = ret;
}
```

```
static int qcow2_create(const char *filename, QemuOpts *opts, Error **errp)
{
    char *backing_file = NULL;
    char *backing_fmt = NULL;
    char *buf = NULL;
    uint64_t size = 0;
    int flags = 0;
    size_t cluster_size = DEFAULT_CLUSTER_SIZE;
    PreallocMode prealloc;
    int version;
    uint64_t refcount_bits;
    int refcount_order;
    char *encryptfmt = NULL;
    Error *local_err = NULL;
    int ret;

    /* Read out options */
    /*获取创建的镜像文件大小信息（向上对齐到sector大小），基础镜像文件名称，基础镜像格式，加密格式等信息*/
    size = ROUND_UP(qemu_opt_get_size_del(opts, BLOCK_OPT_SIZE, 0),
                    BDRV_SECTOR_SIZE);
    backing_file = qemu_opt_get_del(opts, BLOCK_OPT_BACKING_FILE);
    backing_fmt = qemu_opt_get_del(opts, BLOCK_OPT_BACKING_FMT);
    encryptfmt = qemu_opt_get_del(opts, BLOCK_OPT_ENCRYPT_FORMAT);
    /*如果已经指定了加密格式，就不应该指定是否加密这一选项；如果没有指定加密格式，但是指定了加密这一选项，则默认采用AES加密格式*/
    if (encryptfmt) {
        if (qemu_opt_get(opts, BLOCK_OPT_ENCRYPT)) {
            error_setg(errp, "Options " BLOCK_OPT_ENCRYPT " and "
                       BLOCK_OPT_ENCRYPT_FORMAT " are mutually exclusive");
            ret = -EINVAL;
            goto finish;
        }
    } else if (qemu_opt_get_bool_del(opts, BLOCK_OPT_ENCRYPT, false)) {
        encryptfmt = g_strdup("aes");
    }
    
    /*获取镜像文件的cluster_size信息，cluster_size必须介于512B - 2MB之间，且必须是2^n次方，如果没有设置则采用默认值：65536B（即64KB）*/
    cluster_size = qcow2_opt_get_cluster_size_del(opts, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        ret = -EINVAL;
        goto finish;
    }
    
    /*获取preallocate选项，如果没有设置或者设置的是非法的值，则设置为PREALLOC_MODE_OFF，即不支持preallocate*/
    buf = qemu_opt_get_del(opts, BLOCK_OPT_PREALLOC);
    prealloc = qapi_enum_parse(&PreallocMode_lookup, buf,
                               PREALLOC_MODE_OFF, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        ret = -EINVAL;
        goto finish;
    }

    /*获取版本信息，如果指定0.10则版本号为2，如果指定为1.1则版本号为3，如果没有指定则默认版本号为3，如果指定为其他值，则非法*/
    version = qcow2_opt_get_version_del(opts, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        ret = -EINVAL;
        goto finish;
    }

    /*如果设置了lazy refcount，则设置flags*/
    if (qemu_opt_get_bool_del(opts, BLOCK_OPT_LAZY_REFCOUNTS, false)) {
        flags |= BLOCK_FLAG_LAZY_REFCOUNTS;
    }

    /*如果指定了基础镜像，则不能设置preallocate标识*/
    if (backing_file && prealloc != PREALLOC_MODE_OFF) {
        error_setg(errp, "Backing file and preallocation cannot be used at "
                   "the same time");
        ret = -EINVAL;
        goto finish;
    }

    if (version < 3 && (flags & BLOCK_FLAG_LAZY_REFCOUNTS)) {
        error_setg(errp, "Lazy refcounts only supported with compatibility "
                   "level 1.1 and above (use compat=1.1 or greater)");
        ret = -EINVAL;
        goto finish;
    }

    /*获取refcount所占用的bits（即refcount width），必须是2^n次方，且不能超过64bit*/
    refcount_bits = qcow2_opt_get_refcount_bits_del(opts, version, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        ret = -EINVAL;
        goto finish;
    }

    /*根据refcount_bits计算refcount_order，2^refcount_order = refcount_bits*/
    refcount_order = ctz32(refcount_bits);

    ret = qcow2_create2(filename, size, backing_file, backing_fmt, flags,
                        cluster_size, prealloc, opts, version, refcount_order,
                        encryptfmt, &local_err);
    error_propagate(errp, local_err);

finish:
    g_free(backing_file);
    g_free(backing_fmt);
    g_free(encryptfmt);
    g_free(buf);
    return ret;
}
```

```
static int qcow2_create2(const char *filename, int64_t total_size,
                         const char *backing_file, const char *backing_format,
                         int flags, size_t cluster_size, PreallocMode prealloc,
                         QemuOpts *opts, int version, int refcount_order,
                         const char *encryptfmt, Error **errp)
{
    QDict *options;

    /*
     * Open the image file and write a minimal qcow2 header.
     *
     * We keep things simple and start with a zero-sized image. We also
     * do without refcount blocks or a L1 table for now. We'll fix the
     * inconsistency later.
     *
     * We do need a refcount table because growing the refcount table means
     * allocating two new refcount blocks - the seconds of which would be at
     * 2 GB for 64k clusters, and we don't want to have a 2 GB initial file
     * size for any qcow2 image.
     */
    BlockBackend *blk;
    QCowHeader *header;
    uint64_t* refcount_table;
    Error *local_err = NULL;
    int ret;

    if (prealloc == PREALLOC_MODE_FULL || prealloc == PREALLOC_MODE_FALLOC) {
        /*计算完全分配情况下，镜像所占用的总的字节数，计算方式参考qcow2_calc_prealloc_size*/
        int64_t prealloc_size =
            qcow2_calc_prealloc_size(total_size, cluster_size, refcount_order);
        /*重新调整镜像大小参数为计算得到的@prealloc_size*/
        qemu_opt_set_number(opts, BLOCK_OPT_SIZE, prealloc_size, &error_abort);
        /*设置preallocation mode参数，在创建虚拟机的参数中可能尚未指定该参数，所以使用的是默认参数，但是从这以后，就能够从@opts中获取preallocation mode这一参数了*/
        qemu_opt_set(opts, BLOCK_OPT_PREALLOC, PreallocMode_str(prealloc),
                     &error_abort);
    }

    /*从filename中解析出protocol，并调用相应的protocol driver来创建文件*/
    ret = bdrv_create_file(filename, opts, &local_err);
    if (ret < 0) {
        error_propagate(errp, local_err);
        return ret;
    }

    /*创建BlockBackend，打开BlockDriverState并且关联这两者，参考blk_new_open*/
    blk = blk_new_open(filename, NULL, NULL,
                       BDRV_O_RDWR | BDRV_O_RESIZE | BDRV_O_PROTOCOL,
                       &local_err);
    if (blk == NULL) {
        error_propagate(errp, local_err);
        return -EIO;
    }

    /*设置允许在eof之后写入数据*/
    blk_set_allow_write_beyond_eof(blk, true);

    /* Write the header，填充header内存部分 */
    QEMU_BUILD_BUG_ON((1 << MIN_CLUSTER_BITS) < sizeof(*header));
    header = g_malloc0(cluster_size);
    *header = (QCowHeader) {
        .magic                      = cpu_to_be32(QCOW_MAGIC),
        .version                    = cpu_to_be32(version),
        .cluster_bits               = cpu_to_be32(ctz32(cluster_size)),
        .size                       = cpu_to_be64(0),
        .l1_table_offset            = cpu_to_be64(0),
        .l1_size                    = cpu_to_be32(0),
        .refcount_table_offset      = cpu_to_be64(cluster_size),
        .refcount_table_clusters    = cpu_to_be32(1),
        .refcount_order             = cpu_to_be32(refcount_order),
        .header_length              = cpu_to_be32(sizeof(*header)),
    };

    /* We'll update this to correct value later（初始设置为不加密，后面会将之设置为正确的值）*/
    header->crypt_method = cpu_to_be32(QCOW_CRYPT_NONE);

    /*lazy refcount相关的设置*/
    if (flags & BLOCK_FLAG_LAZY_REFCOUNTS) {
        header->compatible_features |=
            cpu_to_be64(QCOW2_COMPAT_LAZY_REFCOUNTS);
    }

    /*同步写入header部分（阻塞直到写入成功）*/
    ret = blk_pwrite(blk, 0, header, cluster_size, 0);
    g_free(header);
    if (ret < 0) {
        error_setg_errno(errp, -ret, "Could not write qcow2 header");
        goto out;
    }

    /*紧接着header部分的是一个refcount table，初始大小为2*cluster_size（假如cluster_size采用默认值64kB，refcount_bits=3，即每个refcount占用8Byte，那么2个cluster大小的refcount_table可以管理的容量多达(((2 * cluster_size) / 8) * (cluster_size / 8)) * cluster_size = 8TB），但实际上refcount table只占用一个cluster，见下面一条注释*/
    /* Write a refcount table with one refcount block */
    refcount_table = g_malloc0(2 * cluster_size);
    /*refcount_table[0]表示refcount_table中的第一个refcount table entry，占用8Byte，表示第一个refcount block在镜像文件中的偏移，但是这里设置第一个refcount block的起始偏移为2 * cluster_size，说明refcount_table只占用了一个cluster，虽然refcount_table在分配的时候分配了2 * cluster_size大小，但是这可能仅仅是为了避免两次分配内存吧*/
    refcount_table[0] = cpu_to_be64(2 * cluster_size);
    /*将refcount table和第一个refcount block写入镜像文件*/
    ret = blk_pwrite(blk, cluster_size, refcount_table, 2 * cluster_size, 0);
    g_free(refcount_table);

    if (ret < 0) {
        error_setg_errno(errp, -ret, "Could not write refcount table");
        goto out;
    }

    blk_unref(blk);
    blk = NULL;

    /*
     * And now open the image and make it consistent first (i.e. increase the
     * refcount of the cluster that is occupied by the header and the refcount
     * table)
     */
    /*调用blk_new分配BlockBackend，同时调用bdrv_open打开镜像文件，bdrv_open会返回BlockDriverState，最后将BlockBackend和BlockDriverState关联起来；bdrv_open -> bdrv_open_inherit -> bdrv_open_common -> bdrv_open_driver -> BlockDriver::bdrv_file_open | BlockDriver::bdrv_open，对于qcow2来说，实现了BlockDriverState::bdrv_open = qcow2_open，详见qcow2_open*/
    options = qdict_new();
    qdict_put_str(options, "driver", "qcow2");
    blk = blk_new_open(filename, NULL, options,
                       BDRV_O_RDWR | BDRV_O_RESIZE | BDRV_O_NO_FLUSH,
                       &local_err);
    if (blk == NULL) {
        error_propagate(errp, local_err);
        ret = -EIO;
        goto out;
    }

    /*为qcow2的header和refcount table部分分配3个cluster，并且在refcount table中更新这些cluster的引用计数，见qcow2_alloc_clusters*/
    ret = qcow2_alloc_clusters(blk_bs(blk), 3 * cluster_size);
    if (ret < 0) {
        error_setg_errno(errp, -ret, "Could not allocate clusters for qcow2 "
                         "header and refcount table");
        goto out;
    } else if (ret != 0) {
        error_report("Huh, first cluster in empty image is already in use?");
        abort();
    }

    /* Create a full header (including things like feature table) */
    ret = qcow2_update_header(blk_bs(blk));
    if (ret < 0) {
        error_setg_errno(errp, -ret, "Could not update qcow2 header");
        goto out;
    }

    /* Okay, now that we have a valid image, let's give it the right size，将之truncate（以PREALLOC_MODE_OFF模式）到@total_size大小*/
    ret = blk_truncate(blk, total_size, PREALLOC_MODE_OFF, errp);
    if (ret < 0) {
        error_prepend(errp, "Could not resize image: ");
        goto out;
    }

    /* Want a backing file? There you go.*/
    if (backing_file) {
        /*更新backing file信息，对于qcow2来说，最终调用qcow2_change_backing_file完成更新，见qcow2_change_backing_file*/
        ret = bdrv_change_backing_file(blk_bs(blk), backing_file, backing_format);
        if (ret < 0) {
            error_setg_errno(errp, -ret, "Could not assign backing file '%s' "
                             "with format '%s'", backing_file, backing_format);
            goto out;
        }
    }

    /* Want encryption? There you go. */
    if (encryptfmt) {
        /*如果设置了加密格式，则建立加密相关驱动，同时更新镜像文件的header部分中关于加密相关的信息*/
        ret = qcow2_set_up_encryption(blk_bs(blk), encryptfmt, opts, errp);
        if (ret < 0) {
            goto out;
        }
    }

    /* And if we're supposed to preallocate metadata, do that now */
    /*如果设置了preallocation，则预先分配元数据部分，见preallocate*/
    if (prealloc != PREALLOC_MODE_OFF) {
        ret = preallocate(blk_bs(blk), 0, total_size);
        if (ret < 0) {
            error_setg_errno(errp, -ret, "Could not preallocate metadata");
            goto out;
        }
    }

    blk_unref(blk);
    blk = NULL;

    /* Reopen the image without BDRV_O_NO_FLUSH to flush it before returning.
     * Using BDRV_O_NO_IO, since encryption is now setup we don't want to
     * have to setup decryption context. We're not doing any I/O on the top
     * level BlockDriverState, only lower layers, where BDRV_O_NO_IO does
     * not have effect.
     */
    options = qdict_new();
    qdict_put_str(options, "driver", "qcow2");
    blk = blk_new_open(filename, NULL, options,
                       BDRV_O_RDWR | BDRV_O_NO_BACKING | BDRV_O_NO_IO,
                       &local_err);
    if (blk == NULL) {
        error_propagate(errp, local_err);
        ret = -EIO;
        goto out;
    }

    ret = 0;
out:
    if (blk) {
        blk_unref(blk);
    }
    return ret;
}
```

```
static int64_t qcow2_calc_prealloc_size(int64_t total_size,
                                        size_t cluster_size,
                                        int refcount_order)
{
    /*计算完全分配情况下镜像所需要的字节数，计算方式如下：
    (1). total_size标识数据部分占用的字节数，需要向上对齐到cluster_size，结果为aligned_total_size;
    (2). 元数据之header部分占用cluster_size大小;
    (3). 元数据之L2 table部分占用大小：每一个数据cluster在L2 table中占用一个8字节的entry，数据部分总共有aligned_total_size / cluster_size个cluster，并且L2 table部分要向上对齐到cluster_size;
    (4). 元数据之L1 table部分占用大小：每一个L2 table entry都在L1 table中占用一个8字节的entry，并且L1 table部分要向上对齐到cluster_size;
    (5). 元数据之refcount table和refcount block部分占用大小,参考qcow2_refcount_metadata_size;
    */
        
    int64_t meta_size = 0;
    uint64_t nl1e, nl2e;
    int64_t aligned_total_size = align_offset(total_size, cluster_size);

    /* header: 1 cluster */
    meta_size += cluster_size;

    /* total size of L2 tables */
    nl2e = aligned_total_size / cluster_size;
    nl2e = align_offset(nl2e, cluster_size / sizeof(uint64_t));
    meta_size += nl2e * sizeof(uint64_t);

    /* total size of L1 tables */
    nl1e = nl2e * sizeof(uint64_t) / cluster_size;
    nl1e = align_offset(nl1e, cluster_size / sizeof(uint64_t));
    meta_size += nl1e * sizeof(uint64_t);

    /* total size of refcount table and blocks */
    meta_size += qcow2_refcount_metadata_size(
            (meta_size + aligned_total_size) / cluster_size,
            cluster_size, refcount_order, false, NULL);

    return meta_size + aligned_total_size;
}
```

```
int64_t qcow2_refcount_metadata_size(int64_t clusters, size_t cluster_size,
                                     int refcount_order, bool generous_increase,
                                     uint64_t *refblock_count)
{
    /*
     * Every host cluster is reference-counted, including metadata (even
     * refcount metadata is recursively included).
     *
     * An accurate formula for the size of refcount metadata size is difficult
     * to derive.  An easier method of calculation is finding the fixed point
     * where no further refcount blocks or table clusters are required to
     * reference count every cluster.
     */
    /*每一个cluster(包括数据部分的cluster和元数据部分的cluster，甚至是refcount table和refcount block这俩元数据)都会被引用计数*/
    
    /*每一个refcount table(占用一个cluster大小)中可以管理@blocks_per_table_cluster个refcount block*/
    int64_t blocks_per_table_cluster = cluster_size / sizeof(uint64_t);
    /*每一个refcount block(占用一个cluster大小)中可以管理@refcounts_per_block个引用计数信息*/
    int64_t refcounts_per_block = cluster_size * 8 / (1 << refcount_order);
    int64_t table = 0;  /* number of refcount table clusters */
    int64_t blocks = 0; /* number of refcount block clusters */
    int64_t last;
    int64_t n = 0;

    do {
        last = n;
        /*计算所有元数据和数据部分所占用的cluster的数目需要多少个refcount block(每一个都占用一个cluster)管理其引用计数*/
        blocks = DIV_ROUND_UP(clusters + table + blocks, refcounts_per_block);
        /*计算管理所有refcount block所需要的refcount table cluster数目*/
        table = DIV_ROUND_UP(blocks, blocks_per_table_cluster);
        n = clusters + blocks + table;

        /*如果@generous_increase被设置，则还会额外分配0.5倍的refcount table空间*/
        if (n == last && generous_increase) {
            clusters += DIV_ROUND_UP(table, 2);
            n = 0; /* force another loop */
            generous_increase = false;
        }
    } while (n != last);

    if (refblock_count) {
        *refblock_count = blocks;
    }

    return (blocks + table) * cluster_size;
}
```

```
int bdrv_create_file(const char *filename, QemuOpts *opts, Error **errp)
{
    BlockDriver *drv;
    Error *local_err = NULL;
    int ret;

    /*查找相应的protocol*/
    drv = bdrv_find_protocol(filename, true, errp);
    if (drv == NULL) {
        return -ENOENT;
    }

    /*调用相应的protocol driver来创建文件，在创建镜像的时候如果没有显示指定protocol，则直接采用的是bdrv_file（file protocol driver），该函数在bdrv_img_create中讲过，但是那里调用的是format driver，对于qcow2来说，就是bdrv_qcow2，这里则是protocol driver，通常来说就是bdrv_file*/
    ret = bdrv_create(drv, filename, opts, &local_err);
    error_propagate(errp, local_err);
    return ret;
}
```

```
BlockDriver *bdrv_find_protocol(const char *filename,
                                bool allow_protocol_prefix,
                                Error **errp)
{
    BlockDriver *drv1;
    char protocol[128];
    int len;
    const char *p;
    int i;

    /* TODO Drivers without bdrv_file_open must be specified explicitly */

    /*
     * XXX(hch): we really should not let host device detection
     * override an explicit protocol specification, but moving this
     * later breaks access to device names with colons in them.
     * Thanks to the brain-dead persistent naming schemes on udev-
     * based Linux systems those actually are quite common.
     */
    /*如果是host device，则查找对应的driver*/
    drv1 = find_hdev_driver(filename);
    if (drv1) {
        return drv1;
    }

    /*如果filename中不包含protocol信息，或者filename中不允许protocol作为前缀，则直接作为文件处理，直接采用@bdrv_file*/
    if (!path_has_protocol(filename) || !allow_protocol_prefix) {
        return &bdrv_file;
    }

    /*否则，解析出相应的protocol，并查找相应的protocol driver*/
    p = strchr(filename, ':');
    assert(p != NULL);
    len = p - filename;
    if (len > sizeof(protocol) - 1)
        len = sizeof(protocol) - 1;
    memcpy(protocol, filename, len);
    protocol[len] = '\0';

    drv1 = bdrv_do_find_protocol(protocol);
    if (drv1) {
        return drv1;
    }

    for (i = 0; i < (int)ARRAY_SIZE(block_driver_modules); ++i) {
        if (block_driver_modules[i].protocol_name &&
            !strcmp(block_driver_modules[i].protocol_name, protocol)) {
            block_module_load_one(block_driver_modules[i].library_name);
            break;
        }
    }

    drv1 = bdrv_do_find_protocol(protocol);
    if (!drv1) {
        error_setg(errp, "Unknown protocol '%s'", protocol);
    }
    return drv1;
}
```

```
int bdrv_create(BlockDriver *drv, const char* filename,
                QemuOpts *opts, Error **errp)
{
    int ret;

    Coroutine *co;
    CreateCo cco = {
        .drv = drv,
        .filename = g_strdup(filename),
        .opts = opts,
        .ret = NOT_DONE,
        .err = NULL,
    };

    if (!drv->bdrv_create) {
        error_setg(errp, "Driver '%s' does not support image creation", drv->format_name);
        ret = -ENOTSUP;
        goto out;
    }

    /*无论是采用已有的协程还是创建新的协程，最终执行的都是bdrv_create_co_entry，见bdrv_create_co_entry*/
    if (qemu_in_coroutine()) {
        /* Fast-path if already in coroutine context */
        bdrv_create_co_entry(&cco);
    } else {
        co = qemu_coroutine_create(bdrv_create_co_entry, &cco);
        qemu_coroutine_enter(co);
        while (cco.ret == NOT_DONE) {
            aio_poll(qemu_get_aio_context(), true);
        }
    }

    ret = cco.ret;
    if (ret < 0) {
        if (cco.err) {
            error_propagate(errp, cco.err);
        } else {
            error_setg_errno(errp, -ret, "Could not create image");
        }
    }

out:
    g_free(cco.filename);
    return ret;
}
```

```
static void coroutine_fn bdrv_create_co_entry(void *opaque)
{
    Error *local_err = NULL;
    int ret;

    CreateCo *cco = opaque;
    assert(cco->drv);

    /*这里调用的是bdrv_file::bdrv_create，即raw_create*/
    ret = cco->drv->bdrv_create(cco->filename, cco->opts, &local_err);
    error_propagate(&cco->err, local_err);
    cco->ret = ret;
}

```

```
static int raw_create(const char *filename, QemuOpts *opts, Error **errp)
{
    int fd;
    int result = 0;
    int64_t total_size = 0;
    bool nocow = false;
    PreallocMode prealloc;
    char *buf = NULL;
    Error *local_err = NULL;

    /*如果@filename是以file protocol开头，则去除"file:"部分*/
    strstart(filename, "file:", &filename);

    /* Read out options */
    total_size = ROUND_UP(qemu_opt_get_size_del(opts, BLOCK_OPT_SIZE, 0),
                          BDRV_SECTOR_SIZE);
    nocow = qemu_opt_get_bool(opts, BLOCK_OPT_NOCOW, false);
    buf = qemu_opt_get_del(opts, BLOCK_OPT_PREALLOC);
    prealloc = qapi_enum_parse(&PreallocMode_lookup, buf,
                               PREALLOC_MODE_OFF, &local_err);
    g_free(buf);
    if (local_err) {
        error_propagate(errp, local_err);
        result = -EINVAL;
        goto out;
    }

    /*创建文件，实际上是对各种不同系统(win32/linux)下open的封装*/
    fd = qemu_open(filename, O_RDWR | O_CREAT | O_TRUNC | O_BINARY,
                   0644);
    if (fd < 0) {
        result = -errno;
        error_setg_errno(errp, -result, "Could not create file");
        goto out;
    }

    /*no copy-on-write*/
    if (nocow) {
#ifdef __linux__
        /* Set NOCOW flag to solve performance issue on fs like btrfs.
         * This is an optimisation. The FS_IOC_SETFLAGS ioctl return value
         * will be ignored since any failure of this operation should not
         * block the left work.
         */
        int attr;
        if (ioctl(fd, FS_IOC_GETFLAGS, &attr) == 0) {
            attr |= FS_NOCOW_FL;
            ioctl(fd, FS_IOC_SETFLAGS, &attr);
        }
#endif
    }

    /*将文件truncate到指定大小，如果将文件truncate后size变大，则根据@prealloc指定的preallocation mode做相应的处理*/
    result = raw_regular_truncate(fd, total_size, prealloc, errp);
    if (result < 0) {
        goto out_close;
    }

out_close:
    /*关闭之前打开的文件*/
    if (qemu_close(fd) != 0 && result == 0) {
        result = -errno;
        error_setg_errno(errp, -result, "Could not close the new file");
    }
out:
    return result;
}
```

```
static int raw_regular_truncate(int fd, int64_t offset, PreallocMode prealloc,
                                Error **errp)
{
    int result = 0;
    int64_t current_length = 0;
    char *buf = NULL;
    struct stat st;

    if (fstat(fd, &st) < 0) {
        result = -errno;
        error_setg_errno(errp, -result, "Could not stat file");
        return result;
    }

    current_length = st.st_size;
    if (current_length > offset && prealloc != PREALLOC_MODE_OFF) {
        error_setg(errp, "Cannot use preallocation for shrinking files");
        return -ENOTSUP;
    }

    /*根据@prealloc中指定的preallocation mode做相应处理*/
    switch (prealloc) {
#ifdef CONFIG_POSIX_FALLOCATE
    case PREALLOC_MODE_FALLOC:
        /*直接采用posix_fallocate来分配空间，成功返回0，出错则返回非0*/
        /*
         * Truncating before posix_fallocate() makes it about twice slower on
         * file systems that do not support fallocate(), trying to check if a
         * block is allocated before allocating it, so don't do that here.
         */
        result = -posix_fallocate(fd, current_length, offset - current_length);
        if (result != 0) {
            /* posix_fallocate() doesn't set errno. */
            error_setg_errno(errp, -result,
                             "Could not preallocate new data");
        }
        goto out;
#endif
    case PREALLOC_MODE_FULL:
    {
        /*直接采用ftruncate + write来填充全部空间*/
        int64_t num = 0, left = offset - current_length;

        /*
         * Knowing the final size from the beginning could allow the file
         * system driver to do less allocations and possibly avoid
         * fragmentation of the file.
         */
        if (ftruncate(fd, offset) != 0) {
            result = -errno;
            error_setg_errno(errp, -result, "Could not resize file");
            goto out;
        }

        buf = g_malloc0(65536);

        result = lseek(fd, current_length, SEEK_SET);
        if (result < 0) {
            result = -errno;
            error_setg_errno(errp, -result,
                             "Failed to seek to the old end of file");
            goto out;
        }

        while (left > 0) {
            num = MIN(left, 65536);
            result = write(fd, buf, num);
            if (result < 0) {
                result = -errno;
                error_setg_errno(errp, -result,
                                 "Could not write zeros for preallocation");
                goto out;
            }
            left -= result;
        }
        if (result >= 0) {
            result = fsync(fd);
            if (result < 0) {
                result = -errno;
                error_setg_errno(errp, -result,
                                 "Could not flush file to disk");
                goto out;
            }
        }
        goto out;
    }
    case PREALLOC_MODE_OFF:
        /*只采用ftruncate*/
        if (ftruncate(fd, offset) != 0) {
            result = -errno;
            error_setg_errno(errp, -result, "Could not resize file");
        }
        return result;
    default:
        result = -ENOTSUP;
        error_setg(errp, "Unsupported preallocation mode: %s",
                   PreallocMode_str(prealloc));
        return result;
    }

out:
    if (result < 0) {
        if (ftruncate(fd, current_length) < 0) {
            error_report("Failed to restore old file length: %s",
                         strerror(errno));
        }
    }

    g_free(buf);
    return result;
}
```

```
BlockBackend *blk_new_open(const char *filename, const char *reference,
                           QDict *options, int flags, Error **errp)
{
    BlockBackend *blk;
    BlockDriverState *bs;
    uint64_t perm;

    /* blk_new_open() is mainly used in .bdrv_create implementations and the
     * tools where sharing isn't a concern because the BDS stays private, so we
     * just request permission according to the flags.
     *
     * The exceptions are xen_disk and blockdev_init(); in these cases, the
     * caller of blk_new_open() doesn't make use of the permissions, but they
     * shouldn't hurt either. We can still share everything here because the
     * guest devices will add their own blockers if they can't share. */
    perm = BLK_PERM_CONSISTENT_READ;
    if (flags & BDRV_O_RDWR) {
        perm |= BLK_PERM_WRITE;
    }
    if (flags & BDRV_O_RESIZE) {
        perm |= BLK_PERM_RESIZE;
    }

    /*创建BlockBackend并添加到全局的@block_backends链表中*/
    blk = blk_new(perm, BLK_PERM_ALL);
    /*打开镜像文件*/
    bs = bdrv_open(filename, reference, options, flags, errp);
    if (!bs) {
        blk_unref(blk);
        return NULL;
    }

    blk->root = bdrv_root_attach_child(bs, "root", &child_root,
                                       perm, BLK_PERM_ALL, blk, errp);
    if (!blk->root) {
        bdrv_unref(bs);
        blk_unref(blk);
        return NULL;
    }

    return blk;
}
```

```
static int qcow2_open(BlockDriverState *bs, QDict *options, int flags,
                      Error **errp)
{
    /*bdrv_open_child暂时不关注*/
    bs->file = bdrv_open_child(NULL, options, "file", bs, &child_file,
                               false, errp);
    if (!bs->file) {
        return -EINVAL;
    }

    /*详见qcow2_do_open*/
    return qcow2_do_open(bs, options, flags, errp);
}
```

```
static int qcow2_do_open(BlockDriverState *bs, QDict *options, int flags,
                         Error **errp)
{
    BDRVQcow2State *s = bs->opaque;
    unsigned int len, i;
    int ret = 0;
    QCowHeader header;
    Error *local_err = NULL;
    uint64_t ext_end;
    uint64_t l1_vm_state_index;
    bool update_header = false;

    /*从镜像文件中读取header信息*/
    ret = bdrv_pread(bs->file, 0, &header, sizeof(header));
    if (ret < 0) {
        error_setg_errno(errp, -ret, "Could not read qcow2 header");
        goto fail;
    }
    be32_to_cpus(&header.magic);
    be32_to_cpus(&header.version);
    be64_to_cpus(&header.backing_file_offset);
    be32_to_cpus(&header.backing_file_size);
    be64_to_cpus(&header.size);
    be32_to_cpus(&header.cluster_bits);
    be32_to_cpus(&header.crypt_method);
    be64_to_cpus(&header.l1_table_offset);
    be32_to_cpus(&header.l1_size);
    be64_to_cpus(&header.refcount_table_offset);
    be32_to_cpus(&header.refcount_table_clusters);
    be64_to_cpus(&header.snapshots_offset);
    be32_to_cpus(&header.nb_snapshots);

    if (header.magic != QCOW_MAGIC) {
        error_setg(errp, "Image is not in qcow2 format");
        ret = -EINVAL;
        goto fail;
    }
    if (header.version < 2 || header.version > 3) {
        error_setg(errp, "Unsupported qcow2 version %" PRIu32, header.version);
        ret = -ENOTSUP;
        goto fail;
    }

    s->qcow_version = header.version;

    /* Initialise cluster size */
    if (header.cluster_bits < MIN_CLUSTER_BITS ||
        header.cluster_bits > MAX_CLUSTER_BITS) {
        error_setg(errp, "Unsupported cluster size: 2^%" PRIu32,
                   header.cluster_bits);
        ret = -EINVAL;
        goto fail;
    }

    /*2^(s->cluster_bits) =  s->cluster_size*/
    s->cluster_bits = header.cluster_bits;
    s->cluster_size = 1 << s->cluster_bits;
    s->cluster_sectors = 1 << (s->cluster_bits - 9);

    /* Initialise version 3 header fields */
    if (header.version == 2) {
        header.incompatible_features    = 0;
        header.compatible_features      = 0;
        header.autoclear_features       = 0;
        header.refcount_order           = 4;
        header.header_length            = 72;
    } else {
        /*对于新创建的镜像文件，除了header.compatible_features中可能设置了QCOW2_COMPAT_LAZY_REFCOUNTS标识以外，其它features都是0*/
        be64_to_cpus(&header.incompatible_features);
        be64_to_cpus(&header.compatible_features);
        be64_to_cpus(&header.autoclear_features);
        be32_to_cpus(&header.refcount_order);
        be32_to_cpus(&header.header_length);

        if (header.header_length < 104) {
            error_setg(errp, "qcow2 header too short");
            ret = -EINVAL;
            goto fail;
        }
    }

    if (header.header_length > s->cluster_size) {
        error_setg(errp, "qcow2 header exceeds cluster size");
        ret = -EINVAL;
        goto fail;
    }

    if (header.header_length > sizeof(header)) {
        s->unknown_header_fields_size = header.header_length - sizeof(header);
        s->unknown_header_fields = g_malloc(s->unknown_header_fields_size);
        ret = bdrv_pread(bs->file, sizeof(header), s->unknown_header_fields,
                         s->unknown_header_fields_size);
        if (ret < 0) {
            error_setg_errno(errp, -ret, "Could not read unknown qcow2 header "
                             "fields");
            goto fail;
        }
    }

    if (header.backing_file_offset > s->cluster_size) {
        error_setg(errp, "Invalid backing file offset");
        ret = -EINVAL;
        goto fail;
    }

    /*对于新创建的镜像文件，header.backing_file_offset为0*/
    if (header.backing_file_offset) {
        ext_end = header.backing_file_offset;
    } else {
        ext_end = 1 << header.cluster_bits;
    }

    /* Handle feature bits */
    s->incompatible_features    = header.incompatible_features;
    s->compatible_features      = header.compatible_features;
    s->autoclear_features       = header.autoclear_features;

    /*如果指定了除了QCOW2_INCOMPAT_MASK中定义的incompatible feature以外的，则出错*/
    if (s->incompatible_features & ~QCOW2_INCOMPAT_MASK) {
        void *feature_table = NULL;
        qcow2_read_extensions(bs, header.header_length, ext_end,
                              &feature_table, flags, NULL, NULL);
        report_unsupported_feature(errp, feature_table,
                                   s->incompatible_features &
                                   ~QCOW2_INCOMPAT_MASK);
        ret = -ENOTSUP;
        g_free(feature_table);
        goto fail;
    }

    /*如果incompatible_features中指定了QCOW2_INCOMPAT_CORRUPT，表示image处于corrupt状态*/
    if (s->incompatible_features & QCOW2_INCOMPAT_CORRUPT) {
        /* Corrupt images may not be written to unless they are being repaired
         */
        if ((flags & BDRV_O_RDWR) && !(flags & BDRV_O_CHECK)) {
            error_setg(errp, "qcow2: Image is corrupt; cannot be opened "
                       "read/write");
            ret = -EACCES;
            goto fail;
        }
    }

    /* Check support for various header values */
    if (header.refcount_order > 6) {
        error_setg(errp, "Reference count entry width too large; may not "
                   "exceed 64 bits");
        ret = -EINVAL;
        goto fail;
    }
    
    /*s->refcount_order 是以bit为单位，如s->refcount_order是4，就表示使用2B表示refcount*/
    s->refcount_order = header.refcount_order;
    /*s->refcount_bits表示使用多少个bit表示一个refcount*/
    s->refcount_bits = 1 << s->refcount_order;
    s->refcount_max = UINT64_C(1) << (s->refcount_bits - 1);
    s->refcount_max += s->refcount_max - 1;

    s->crypt_method_header = header.crypt_method;
    if (s->crypt_method_header) {
        if (bdrv_uses_whitelist() &&
            s->crypt_method_header == QCOW_CRYPT_AES) {
            error_setg(errp,
                       "Use of AES-CBC encrypted qcow2 images is no longer "
                       "supported in system emulators");
            error_append_hint(errp,
                              "You can use 'qemu-img convert' to convert your "
                              "image to an alternative supported format, such "
                              "as unencrypted qcow2, or raw with the LUKS "
                              "format instead.\n");
            ret = -ENOSYS;
            goto fail;
        }

        if (s->crypt_method_header == QCOW_CRYPT_AES) {
            s->crypt_physical_offset = false;
        } else {
            /* Assuming LUKS and any future crypt methods we
             * add will all use physical offsets, due to the
             * fact that the alternative is insecure...  */
            s->crypt_physical_offset = true;
        }

        bs->encrypted = true;
    }

    s->l2_bits = s->cluster_bits - 3; /* L2 is always one cluster */
    s->l2_size = 1 << s->l2_bits;
    /* 2^(s->refcount_order - 3) is the refcount width in bytes */
    s->refcount_block_bits = s->cluster_bits - (s->refcount_order - 3);
    s->refcount_block_size = 1 << s->refcount_block_bits;
    bs->total_sectors = header.size / 512;
    s->csize_shift = (62 - (s->cluster_bits - 8));
    s->csize_mask = (1 << (s->cluster_bits - 8)) - 1;
    s->cluster_offset_mask = (1LL << s->csize_shift) - 1;

    s->refcount_table_offset = header.refcount_table_offset;
    s->refcount_table_size =
        header.refcount_table_clusters << (s->cluster_bits - 3);

    if (header.refcount_table_clusters > qcow2_max_refcount_clusters(s)) {
        error_setg(errp, "Reference count table too large");
        ret = -EINVAL;
        goto fail;
    }

    ret = validate_table_offset(bs, s->refcount_table_offset,
                                s->refcount_table_size, sizeof(uint64_t));
    if (ret < 0) {
        error_setg(errp, "Invalid reference count table offset");
        goto fail;
    }

    /* Snapshot table offset/length */
    if (header.nb_snapshots > QCOW_MAX_SNAPSHOTS) {
        error_setg(errp, "Too many snapshots");
        ret = -EINVAL;
        goto fail;
    }

    ret = validate_table_offset(bs, header.snapshots_offset,
                                header.nb_snapshots,
                                sizeof(QCowSnapshotHeader));
    if (ret < 0) {
        error_setg(errp, "Invalid snapshot table offset");
        goto fail;
    }

    /* read the level 1 table */
    if (header.l1_size > QCOW_MAX_L1_SIZE / sizeof(uint64_t)) {
        error_setg(errp, "Active L1 table too large");
        ret = -EFBIG;
        goto fail;
    }
    s->l1_size = header.l1_size;

    l1_vm_state_index = size_to_l1(s, header.size);
    if (l1_vm_state_index > INT_MAX) {
        error_setg(errp, "Image is too big");
        ret = -EFBIG;
        goto fail;
    }
    s->l1_vm_state_index = l1_vm_state_index;

    /* the L1 table must contain at least enough entries to put
       header.size bytes */
    if (s->l1_size < s->l1_vm_state_index) {
        error_setg(errp, "L1 table is too small");
        ret = -EINVAL;
        goto fail;
    }

    ret = validate_table_offset(bs, header.l1_table_offset,
                                header.l1_size, sizeof(uint64_t));
    if (ret < 0) {
        error_setg(errp, "Invalid L1 table offset");
        goto fail;
    }
    s->l1_table_offset = header.l1_table_offset;


    if (s->l1_size > 0) {
        s->l1_table = qemu_try_blockalign(bs->file->bs,
            align_offset(s->l1_size * sizeof(uint64_t), 512));
        if (s->l1_table == NULL) {
            error_setg(errp, "Could not allocate L1 table");
            ret = -ENOMEM;
            goto fail;
        }
        ret = bdrv_pread(bs->file, s->l1_table_offset, s->l1_table,
                         s->l1_size * sizeof(uint64_t));
        if (ret < 0) {
            error_setg_errno(errp, -ret, "Could not read L1 table");
            goto fail;
        }
        for(i = 0;i < s->l1_size; i++) {
            be64_to_cpus(&s->l1_table[i]);
        }
    }

    /* Parse driver-specific options */
    ret = qcow2_update_options(bs, options, flags, errp);
    if (ret < 0) {
        goto fail;
    }

    s->cluster_cache_offset = -1;
    s->flags = flags;

    ret = qcow2_refcount_init(bs);
    if (ret != 0) {
        error_setg_errno(errp, -ret, "Could not initialize refcount handling");
        goto fail;
    }

    QLIST_INIT(&s->cluster_allocs);
    QTAILQ_INIT(&s->discards);

    /* read qcow2 extensions */
    if (qcow2_read_extensions(bs, header.header_length, ext_end, NULL,
                              flags, &update_header, &local_err)) {
        error_propagate(errp, local_err);
        ret = -EINVAL;
        goto fail;
    }

    /* qcow2_read_extension may have set up the crypto context
     * if the crypt method needs a header region, some methods
     * don't need header extensions, so must check here
     */
    if (s->crypt_method_header && !s->crypto) {
        if (s->crypt_method_header == QCOW_CRYPT_AES) {
            unsigned int cflags = 0;
            if (flags & BDRV_O_NO_IO) {
                cflags |= QCRYPTO_BLOCK_OPEN_NO_IO;
            }
            s->crypto = qcrypto_block_open(s->crypto_opts, "encrypt.",
                                           NULL, NULL, cflags, errp);
            if (!s->crypto) {
                ret = -EINVAL;
                goto fail;
            }
        } else if (!(flags & BDRV_O_NO_IO)) {
            error_setg(errp, "Missing CRYPTO header for crypt method %d",
                       s->crypt_method_header);
            ret = -EINVAL;
            goto fail;
        }
    }

    /* read the backing file name */
    if (header.backing_file_offset != 0) {
        len = header.backing_file_size;
        if (len > MIN(1023, s->cluster_size - header.backing_file_offset) ||
            len >= sizeof(bs->backing_file)) {
            error_setg(errp, "Backing file name too long");
            ret = -EINVAL;
            goto fail;
        }
        ret = bdrv_pread(bs->file, header.backing_file_offset,
                         bs->backing_file, len);
        if (ret < 0) {
            error_setg_errno(errp, -ret, "Could not read backing file name");
            goto fail;
        }
        bs->backing_file[len] = '\0';
        s->image_backing_file = g_strdup(bs->backing_file);
    }

    /* Internal snapshots */
    s->snapshots_offset = header.snapshots_offset;
    s->nb_snapshots = header.nb_snapshots;

    ret = qcow2_read_snapshots(bs);
    if (ret < 0) {
        error_setg_errno(errp, -ret, "Could not read snapshots");
        goto fail;
    }

    /* Clear unknown autoclear feature bits */
    update_header |= s->autoclear_features & ~QCOW2_AUTOCLEAR_MASK;
    update_header =
        update_header && !bs->read_only && !(flags & BDRV_O_INACTIVE);
    if (update_header) {
        s->autoclear_features &= QCOW2_AUTOCLEAR_MASK;
    }

    if (qcow2_load_autoloading_dirty_bitmaps(bs, &local_err)) {
        update_header = false;
    }
    if (local_err != NULL) {
        error_propagate(errp, local_err);
        ret = -EINVAL;
        goto fail;
    }

    if (update_header) {
        ret = qcow2_update_header(bs);
        if (ret < 0) {
            error_setg_errno(errp, -ret, "Could not update qcow2 header");
            goto fail;
        }
    }

    /* Initialise locks */
    qemu_co_mutex_init(&s->lock);
    bs->supported_zero_flags = BDRV_REQ_MAY_UNMAP;

    /* Repair image if dirty */
    if (!(flags & (BDRV_O_CHECK | BDRV_O_INACTIVE)) && !bs->read_only &&
        (s->incompatible_features & QCOW2_INCOMPAT_DIRTY)) {
        BdrvCheckResult result = {0};

        ret = qcow2_check(bs, &result, BDRV_FIX_ERRORS | BDRV_FIX_LEAKS);
        if (ret < 0) {
            error_setg_errno(errp, -ret, "Could not repair dirty image");
            goto fail;
        }
    }

#ifdef DEBUG_ALLOC
    {
        BdrvCheckResult result = {0};
        qcow2_check_refcounts(bs, &result, 0);
    }
#endif
    return ret;

 fail:
    g_free(s->unknown_header_fields);
    cleanup_unknown_header_ext(bs);
    qcow2_free_snapshots(bs);
    qcow2_refcount_close(bs);
    qemu_vfree(s->l1_table);
    /* else pre-write overlap checks in cache_destroy may crash */
    s->l1_table = NULL;
    cache_clean_timer_del(bs);
    if (s->l2_table_cache) {
        qcow2_cache_destroy(bs, s->l2_table_cache);
    }
    if (s->refcount_block_cache) {
        qcow2_cache_destroy(bs, s->refcount_block_cache);
    }
    qcrypto_block_free(s->crypto);
    qapi_free_QCryptoBlockOpenOptions(s->crypto_opts);
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
        /*分配连续的且引用计数为0的clusters，这些连续的clusters要足以容纳size大小的数据，返回这些连续的clusters的起始地址*/
        offset = alloc_clusters_noref(bs, size);
        if (offset < 0) {
            return offset;
        }

        /*更新这些新分配的cluster的refcount*/
        ret = update_refcount(bs, offset, size, 1, false, QCOW2_DISCARD_NEVER);
    } while (ret == -EAGAIN);

    if (ret < 0) {
        return ret;
    }

    return offset;
}
```

```
/* return < 0 if error */
static int64_t alloc_clusters_noref(BlockDriverState *bs, uint64_t size)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t i, nb_clusters, refcount;
    int ret;

    /* We can't allocate clusters if they may still be queued for discard. */
    if (s->cache_discards) {
        qcow2_process_discards(bs, 0);
    }

    /*将@size向上对齐到cluster，需要占用的cluster数目记录在@nb_cluster中*/
    nb_clusters = size_to_clusters(s, size);
retry:
    for(i = 0; i < nb_clusters; i++) {
        uint64_t next_cluster_index = s->free_cluster_index++;
        ret = qcow2_get_refcount(bs, next_cluster_index, &refcount);
        /*如果ret  < 0, 则出错；否则，如果refcount不为0，则重试，继续查找后续的cluster，因为该函数的目标是查找连续的refcount为0的clusters*/
        if (ret < 0) {
            return ret;
        } else if (refcount != 0) {
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
    /*返回这一批连续的refcount为0的clusters的起始地址*/
    return (s->free_cluster_index - nb_clusters) << s->cluster_bits;
}
```

```
int qcow2_get_refcount(BlockDriverState *bs, int64_t cluster_index,
                       uint64_t *refcount)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t refcount_table_index, block_index;
    int64_t refcount_block_offset;
    int ret;
    void *refcount_block;

    /*计算@cluster_index所表示的cluster所在的refcount table index --- 即refcount table entry在refcount table中的索引，1 << s->refcount_block_bits表示一个refcount block中可以管理多少个cluster的refcount*/
    refcount_table_index = cluster_index >> s->refcount_block_bits;
    if (refcount_table_index >= s->refcount_table_size) {
        *refcount = 0;
        return 0;
    }
    
    /*找到refcount block所在的cluster地址*/
    refcount_block_offset =
        s->refcount_table[refcount_table_index] & REFT_OFFSET_MASK;
    /*refcount_block_offset为0，表示该refcount block尚未分配？引用计数当然就是0了*/
    if (!refcount_block_offset) {
        *refcount = 0;
        return 0;
    }

    /*计算refcount_block_offset代表的地址是不是对齐到cluster_size的，如果不是则出错了*/
    if (offset_into_cluster(s, refcount_block_offset)) {
        qcow2_signal_corruption(bs, true, -1, -1, "Refblock offset %#" PRIx64
                                " unaligned (reftable index: %#" PRIx64 ")",
                                refcount_block_offset, refcount_table_index);
        return -EIO;
    }

    /*从cache中获取refcount_block_offset所代表的refcount block信息，保存在@refcount_block中*/
    ret = qcow2_cache_get(bs, s->refcount_block_cache, refcount_block_offset,
                          &refcount_block);
    if (ret < 0) {
        return ret;
    }

    /*计算@cluster_index所代表的cluster在@refcount_block中的索引*/
    block_index = cluster_index & (s->refcount_block_size - 1);
    /*读取@cluster_index的引用计数信息*/
    *refcount = s->get_refcount(refcount_block, block_index);

    /*这是为了维护lru?*/
    qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);

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

    if (length < 0) {
        return -EINVAL;
    } else if (length == 0) {
        return 0;
    }

    if (decrease) {
        qcow2_cache_set_dependency(bs, s->refcount_block_cache,
            s->l2_table_cache);
    }

    /*@start: offset所在的cluster的起始偏移， @last：offset + length - 1 所在的cluster的起始偏移*/
    start = start_of_cluster(s, offset);
    last = start_of_cluster(s, offset + length - 1);
    for(cluster_offset = start; cluster_offset <= last;
        cluster_offset += s->cluster_size)
    {
        int block_index;
        uint64_t refcount;
        /*计算当前cluster的索引及cluster在refcount table中对应的refcount table entry的索引*/
        int64_t cluster_index = cluster_offset >> s->cluster_bits;
        int64_t table_index = cluster_index >> s->refcount_block_bits;

        /* Load the refcount block and allocate it if needed */
        if (table_index != old_table_index) {
            /*切换了refcount block，在cache中更新上一个refcount_block*/
            if (refcount_block) {
                qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
            }
            
            /*首先尝试查找refcount block，如果找不到，则尝试分配一个新的*/
            ret = alloc_refcount_block(bs, cluster_index, &refcount_block);
            if (ret < 0) {
                goto fail;
            }
        }
        old_table_index = table_index;

        qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache,
                                     refcount_block);

        /*获取refcount并更新之*/
        /* we can update the count and save it */
        block_index = cluster_index & (s->refcount_block_size - 1);
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
        
        /*如果refcount降为0，且cluster_index小于当前记录的最小的free的cluster index，则更新s->free_cluster_index，s->free_cluster_index中记录的当前最小的处于free状态的cluster，比它大的cluster不一定是free状态*/
        if (refcount == 0 && cluster_index < s->free_cluster_index) {
            s->free_cluster_index = cluster_index;
        }
        
        /*更新refcount*/
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

    /* Write last changed block to disk ，（更新最后一个更新的refcount_block到cache）*/
    if (refcount_block) {
        qcow2_cache_put(bs, s->refcount_block_cache, &refcount_block);
    }

    /*
     * Try do undo any updates if an error is returned (This may succeed in
     * some cases like ENOSPC for allocating a new refcount block)
     */
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
static int alloc_refcount_block(BlockDriverState *bs,
                                int64_t cluster_index, void **refcount_block)
{
    BDRVQcow2State *s = bs->opaque;
    unsigned int refcount_table_index;
    int64_t ret;

    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC);

    /* Find the refcount block for the given cluster，先计算该@cluster_index在refcount_table的哪个refcount block中，@refcount_table_index即该refcount block的索引*/
    refcount_table_index = cluster_index >> s->refcount_block_bits;

    /*表明refcount_table_index@所代表的refcount block已经分配*/
    if (refcount_table_index < s->refcount_table_size) {

        uint64_t refcount_block_offset =
            s->refcount_table[refcount_table_index] & REFT_OFFSET_MASK;

        /* If it's already there, we're done */
        if (refcount_block_offset) {
            /*@refcount_block_offset必须是cluster_size对齐的*/
            if (offset_into_cluster(s, refcount_block_offset)) {
                qcow2_signal_corruption(bs, true, -1, -1, "Refblock offset %#"
                                        PRIx64 " unaligned (reftable index: "
                                        "%#x)", refcount_block_offset,
                                        refcount_table_index);
                return -EIO;
            }

            /*直接去cache中查找该refcount_block*/
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

    /*走到这里，我们就需要分配一个新的refcount block，也有可能需要分配一个refcount table(如果现有的refcount table太小了的话)*/
    *refcount_block = NULL;

    /* We write to the refcount table, so we might depend on L2 tables */
    ret = qcow2_cache_flush(bs, s->l2_table_cache);
    if (ret < 0) {
        return ret;
    }

    /*分配一个新的refcount block，它占用cluster_size大小的空间*/
    /* Allocate the refcount block itself and mark it as used */
    int64_t new_block = alloc_clusters_noref(bs, s->cluster_size);
    if (new_block < 0) {
        return new_block;
    }

#ifdef DEBUG_ALLOC2
    fprintf(stderr, "qcow2: Allocate refcount block %d for %" PRIx64
        " at %" PRIx64 "\n",
        refcount_table_index, cluster_index << s->cluster_bits, new_block);
#endif

    /*如果cluster_index所对应的refcount block就是这个新分配的，则将该refcount block中的refcount信息全置为0，然后在更新引用计数*/
    if (in_same_refcount_block(s, new_block, cluster_index << s->cluster_bits)) {
        /* Zero the new refcount block before updating it */
        ret = qcow2_cache_get_empty(bs, s->refcount_block_cache, new_block,
                                    refcount_block);
        if (ret < 0) {
            goto fail;
        }

        memset(*refcount_block, 0, s->cluster_size);

        /* The block describes itself, need to update the cache */
        int block_index = (new_block >> s->cluster_bits) &
            (s->refcount_block_size - 1);
        s->set_refcount(*refcount_block, block_index, 1);
    } else {
        /* Described somewhere else. This can recurse at most twice before we
         * arrive at a block that describes itself. */
        /*更新@new_block所在的cluster的引用计数*/
        ret = update_refcount(bs, new_block, s->cluster_size, 1, false,
                              QCOW2_DISCARD_NEVER);
        if (ret < 0) {
            goto fail;
        }

        ret = qcow2_cache_flush(bs, s->refcount_block_cache);
        if (ret < 0) {
            goto fail;
        }

        /*初始化@new_block这个refcount block中所维护的引用计数，全部置为0，在cache中也为@new_block分配一个entry*/
        /* Initialize the new refcount block only after updating its refcount,
         * update_refcount uses the refcount cache itself */
        ret = qcow2_cache_get_empty(bs, s->refcount_block_cache, new_block,
                                    refcount_block);
        if (ret < 0) {
            goto fail;
        }

        memset(*refcount_block, 0, s->cluster_size);
    }

    /* Now the new refcount block needs to be written to disk */
    BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC_WRITE);
    /*将refcount block cache中@refcount_block所在的entry置为脏，在qcow2_cache_flush中会将那些置为脏的entry刷入镜像文件中*/
    qcow2_cache_entry_mark_dirty(bs, s->refcount_block_cache, *refcount_block);
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    if (ret < 0) {
        goto fail;
    }

    /* If the refcount table is big enough, just hook the block up there */
    if (refcount_table_index < s->refcount_table_size) {
        /*更新refcount table*/
        uint64_t data64 = cpu_to_be64(new_block);
        BLKDBG_EVENT(bs->file, BLKDBG_REFBLOCK_ALLOC_HOOKUP);
        ret = bdrv_pwrite_sync(bs->file,
            s->refcount_table_offset + refcount_table_index * sizeof(uint64_t),
            &data64, sizeof(data64));
        if (ret < 0) {
            goto fail;
        }

        /*更新BDRVQcow2State*/
        s->refcount_table[refcount_table_index] = new_block;
        /* If there's a hole in s->refcount_table then it can happen
         * that refcount_table_index < s->max_refcount_table_index */
        s->max_refcount_table_index =
            MAX(s->max_refcount_table_index, refcount_table_index);

        /* The new refcount block may be where the caller intended to put its
         * data, so let it restart the search. */
        return -EAGAIN;
    }

    /*将@refcount_block更新到refcount block cache*/
    qcow2_cache_put(bs, s->refcount_block_cache, refcount_block);

    /*如果走到这里，就表示refcount table空间不足，需要分配新的refcount table，暂时不考虑这种情况，所以后面就不分析了*/
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
/*
 * Updates the qcow2 header, including the variable length parts of it, i.e.
 * the backing file name and all extensions. qcow2 was not designed to allow
 * such changes, so if we run out of space (we can only use the first cluster)
 * this function may fail.
 *
 * Returns 0 on success, -errno in error cases.
 */
int qcow2_update_header(BlockDriverState *bs)
{
    BDRVQcow2State *s = bs->opaque;
    QCowHeader *header;
    char *buf;
    size_t buflen = s->cluster_size;
    int ret;
    uint64_t total_size;
    uint32_t refcount_table_clusters;
    size_t header_length;
    Qcow2UnknownHeaderExtension *uext;

    buf = qemu_blockalign(bs, buflen);

    /* Header structure */
    header = (QCowHeader*) buf;

    if (buflen < sizeof(*header)) {
        ret = -ENOSPC;
        goto fail;
    }

    header_length = sizeof(*header) + s->unknown_header_fields_size;
    total_size = bs->total_sectors * BDRV_SECTOR_SIZE;
    refcount_table_clusters = s->refcount_table_size >> (s->cluster_bits - 3);

    *header = (QCowHeader) {
        /* Version 2 fields */
        .magic                  = cpu_to_be32(QCOW_MAGIC),
        .version                = cpu_to_be32(s->qcow_version),
        .backing_file_offset    = 0,
        .backing_file_size      = 0,
        .cluster_bits           = cpu_to_be32(s->cluster_bits),
        .size                   = cpu_to_be64(total_size),
        .crypt_method           = cpu_to_be32(s->crypt_method_header),
        .l1_size                = cpu_to_be32(s->l1_size),
        .l1_table_offset        = cpu_to_be64(s->l1_table_offset),
        .refcount_table_offset  = cpu_to_be64(s->refcount_table_offset),
        .refcount_table_clusters = cpu_to_be32(refcount_table_clusters),
        .nb_snapshots           = cpu_to_be32(s->nb_snapshots),
        .snapshots_offset       = cpu_to_be64(s->snapshots_offset),

        /* Version 3 fields */
        .incompatible_features  = cpu_to_be64(s->incompatible_features),
        .compatible_features    = cpu_to_be64(s->compatible_features),
        .autoclear_features     = cpu_to_be64(s->autoclear_features),
        .refcount_order         = cpu_to_be32(s->refcount_order),
        .header_length          = cpu_to_be32(header_length),
    };

    /* For older versions, write a shorter header */
    switch (s->qcow_version) {
    case 2:
        ret = offsetof(QCowHeader, incompatible_features);
        break;
    case 3:
        ret = sizeof(*header);
        break;
    default:
        ret = -EINVAL;
        goto fail;
    }

    buf += ret;
    buflen -= ret;
    memset(buf, 0, buflen);

    /* Preserve any unknown field in the header */
    if (s->unknown_header_fields_size) {
        if (buflen < s->unknown_header_fields_size) {
            ret = -ENOSPC;
            goto fail;
        }

        memcpy(buf, s->unknown_header_fields, s->unknown_header_fields_size);
        buf += s->unknown_header_fields_size;
        buflen -= s->unknown_header_fields_size;
    }

    /* Backing file format header extension */
    if (s->image_backing_format) {
        ret = header_ext_add(buf, QCOW2_EXT_MAGIC_BACKING_FORMAT,
                             s->image_backing_format,
                             strlen(s->image_backing_format),
                             buflen);
        if (ret < 0) {
            goto fail;
        }

        buf += ret;
        buflen -= ret;
    }

    /* Full disk encryption header pointer extension */
    if (s->crypto_header.offset != 0) {
        cpu_to_be64s(&s->crypto_header.offset);
        cpu_to_be64s(&s->crypto_header.length);
        ret = header_ext_add(buf, QCOW2_EXT_MAGIC_CRYPTO_HEADER,
                             &s->crypto_header, sizeof(s->crypto_header),
                             buflen);
        be64_to_cpus(&s->crypto_header.offset);
        be64_to_cpus(&s->crypto_header.length);
        if (ret < 0) {
            goto fail;
        }
        buf += ret;
        buflen -= ret;
    }

    /* Feature table（在这里设置feature table）*/
    if (s->qcow_version >= 3) {
        Qcow2Feature features[] = {
            {
                .type = QCOW2_FEAT_TYPE_INCOMPATIBLE,
                .bit  = QCOW2_INCOMPAT_DIRTY_BITNR,
                .name = "dirty bit",
            },
            {
                .type = QCOW2_FEAT_TYPE_INCOMPATIBLE,
                .bit  = QCOW2_INCOMPAT_CORRUPT_BITNR,
                .name = "corrupt bit",
            },
            {
                .type = QCOW2_FEAT_TYPE_COMPATIBLE,
                .bit  = QCOW2_COMPAT_LAZY_REFCOUNTS_BITNR,
                .name = "lazy refcounts",
            },
        };

        ret = header_ext_add(buf, QCOW2_EXT_MAGIC_FEATURE_TABLE,
                             features, sizeof(features), buflen);
        if (ret < 0) {
            goto fail;
        }
        buf += ret;
        buflen -= ret;
    }

    /* Bitmap extension */
    if (s->nb_bitmaps > 0) {
        Qcow2BitmapHeaderExt bitmaps_header = {
            .nb_bitmaps = cpu_to_be32(s->nb_bitmaps),
            .bitmap_directory_size =
                    cpu_to_be64(s->bitmap_directory_size),
            .bitmap_directory_offset =
                    cpu_to_be64(s->bitmap_directory_offset)
        };
        ret = header_ext_add(buf, QCOW2_EXT_MAGIC_BITMAPS,
                             &bitmaps_header, sizeof(bitmaps_header),
                             buflen);
        if (ret < 0) {
            goto fail;
        }
        buf += ret;
        buflen -= ret;
    }

    /* Keep unknown header extensions */
    QLIST_FOREACH(uext, &s->unknown_header_ext, next) {
        ret = header_ext_add(buf, uext->magic, uext->data, uext->len, buflen);
        if (ret < 0) {
            goto fail;
        }

        buf += ret;
        buflen -= ret;
    }

    /* End of header extensions */
    ret = header_ext_add(buf, QCOW2_EXT_MAGIC_END, NULL, 0, buflen);
    if (ret < 0) {
        goto fail;
    }

    buf += ret;
    buflen -= ret;

    /* Backing file name */
    if (s->image_backing_file) {
        size_t backing_file_len = strlen(s->image_backing_file);

        if (buflen < backing_file_len) {
            ret = -ENOSPC;
            goto fail;
        }

        /* Using strncpy is ok here, since buf is not NUL-terminated. */
        strncpy(buf, s->image_backing_file, buflen);

        /*因为@header记录的buf在分配的时候的起始地址（buf在分配之后随着各种extension的填充不断在改变地址），所以header->backing_file_offset中记录的是backing file相关信息所在的起始地址*/
        header->backing_file_offset = cpu_to_be64(buf - ((char*) header));
        header->backing_file_size   = cpu_to_be32(backing_file_len);
    }

    /* Write the new header，将新的header更新到镜像文件*/
    ret = bdrv_pwrite(bs->file, 0, header, s->cluster_size);
    if (ret < 0) {
        goto fail;
    }

    ret = 0;
fail:
    qemu_vfree(header);
    return ret;
}
```

```
static int qcow2_change_backing_file(BlockDriverState *bs,
    const char *backing_file, const char *backing_fmt)
{
    BDRVQcow2State *s = bs->opaque;

    if (backing_file && strlen(backing_file) > 1023) {
        return -EINVAL;
    }

    /*设置@bs->backing_file和@bs->backing_format*/
    pstrcpy(bs->backing_file, sizeof(bs->backing_file), backing_file ?: "");
    pstrcpy(bs->backing_format, sizeof(bs->backing_format), backing_fmt ?: "");

    g_free(s->image_backing_file);
    g_free(s->image_backing_format);

    /*设置@s->backing_file和@s->backing_format*/
    s->image_backing_file = backing_file ? g_strdup(bs->backing_file) : NULL;
    s->image_backing_format = backing_fmt ? g_strdup(bs->backing_format) : NULL;

    /*再次调用qcow2_update_header来更新镜像文件的header部分，这次主要是更新backing file相关的信息*/
    return qcow2_update_header(bs);
}
```

```
/**
 * Preallocates metadata structures for data clusters between @offset (in the
 * guest disk) and @new_length (which is thus generally the new guest disk
 * size，从注释可以看出，@new_length不是从@offset算起的长度，通常是guest disk
 * 的新的大小，这里@offset就相当于平时我们是用的start_offset，而@new_length就
 * 相当于我们平时使用的end_offset).
 *
 * Returns: 0 on success, -errno on failure.
 */
static int preallocate(BlockDriverState *bs,
                       uint64_t offset, uint64_t new_length)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t bytes;
    uint64_t host_offset = 0;
    unsigned int cur_bytes;
    int ret;
    QCowL2Meta *meta;

    if (qemu_in_coroutine()) {
        qemu_co_mutex_lock(&s->lock);
    }

    /*这里传入的@offset是起始偏移，@new_length是新的文件大小*/
    assert(offset <= new_length);
    bytes = new_length - offset;

    while (bytes) {
        /*当前分配至多分配@cur_bytes字节*/
        cur_bytes = MIN(bytes, INT_MAX);
        /* @offset：在guest中看到的偏移
         * @cur_bytes：当前至多分配这么大，当该函数返回的时候返回实际分配的大小
         * @host_offset：新分配的cluster在host中的偏移
         * @meta：与L2 table更新相关的元数据
         */
        ret = qcow2_alloc_cluster_offset(bs, offset, &cur_bytes,
                                         &host_offset, &meta);
        if (ret < 0) {
            goto done;
        }

        while (meta) {
            QCowL2Meta *next = meta->next;

            ret = qcow2_alloc_cluster_link_l2(bs, meta);
            if (ret < 0) {
                qcow2_free_any_clusters(bs, meta->alloc_offset,
                                        meta->nb_clusters, QCOW2_DISCARD_NEVER);
                goto done;
            }

            /* There are no dependent requests, but we need to remove our
             * request from the list of in-flight requests */
            QLIST_REMOVE(meta, next_in_flight);

            g_free(meta);
            meta = next;
        }

        /* TODO Preallocate data if requested */
        bytes -= cur_bytes;
        offset += cur_bytes;
    }

    /*
     * It is expected that the image file is large enough to actually contain
     * all of the allocated clusters (otherwise we get failing reads after
     * EOF). Extend the image to the last allocated sector.
     * 在@host_offset表示最后一次分配的起始偏移，@cur_bytes表示最后一次分配的
     * 大小，只有在host_offset + cur_bytes -1 的位置真实的写了至少一个字节的
     * 数据，镜像文件才会真正分配这些cluster（因为preallocate函数在此之前的
     * 代码中可能没有写host_offset + cur_bytes -1这个位置，那么host上可能就没有
     * 真正分配这些位置）
     */
    if (host_offset != 0) {
        uint8_t data = 0;
        ret = bdrv_pwrite(bs->file, (host_offset + cur_bytes) - 1,
                          &data, 1);
        if (ret < 0) {
            goto done;
        }
    }

    ret = 0;

done:
    if (qemu_in_coroutine()) {
        qemu_co_mutex_unlock(&s->lock);
    }
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

        /*
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
         */
        ret = handle_dependencies(bs, start, &cur_bytes, m);
        if (ret == -EAGAIN) {
            /* Currently handle_dependencies() doesn't yield if we already had
             * an allocation. If it did, we would have to clean up the L2Meta
             * structs before starting over. 如果在返回EAGAIN的同时，本次分配请
             * 求未曾有哪怕一个成功的分配（*m为NULL），则重新分配。当然，在当前
             * 的实现中，如果有哪怕一个成功的分配，handle_dependencies()都不会返
             * 回EAGAIN，见handle_dependency的实现*/
            assert(*m == NULL);
            goto again;
        } else if (ret < 0) {
            return ret;
        } else if (cur_bytes == 0) {
            /*@cur_bytes为0，则直接退出*/
            break;
        } else {
            /* handle_dependencies() may have decreased cur_bytes (shortened
             * the allocations below) so that the next dependency is processed
             * correctly during the next loop iteration. */
        }

        /*
         * 2. Count contiguous COPIED clusters. 
         * 当handle_copied返回的时候，@cluster_offset可能已经更新了，表示镜像文件内部偏移，
         * 当handle_copied返回的时候，@cur_bytes也可能被更新了，表示可以分配的连续字节数
         */
        ret = handle_copied(bs, start, &cluster_offset, &cur_bytes, m);
        if (ret < 0) {
            return ret;
        } else if (ret) {
            continue;
        } else if (cur_bytes == 0) {
            break;
        }

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
 */
static int handle_dependencies(BlockDriverState *bs, uint64_t guest_offset,
    uint64_t *cur_bytes, QCowL2Meta **m)
{
    BDRVQcow2State *s = bs->opaque;
    QCowL2Meta *old_alloc;
    uint64_t bytes = *cur_bytes;

    /*s->cluster_allocs中存放的都是那些正在写的尚未更新L2 table的请求（或者部分请求），依次遍历s->cluster_allocs中的  那些请求，检查当前的分配请求和这些请求之间是否存在依赖关系*/
    QLIST_FOREACH(old_alloc, &s->cluster_allocs, next_in_flight) {
        uint64_t start = guest_offset;
        uint64_t end = start + bytes;
        uint64_t old_start = l2meta_cow_start(old_alloc);
        uint64_t old_end = l2meta_cow_end(old_alloc);

        if (end <= old_start || start >= old_end) {
            /* No intersection，和当前的@old_alloc不存在依赖关系*/
        } else {
            if (start < old_start) {
                /* Stop at the start of a running allocation，当前分配请求至多分配@bytes = old_start - start这么多*/
                bytes = old_start - start;
            } else {
                bytes = 0;
            }

            /* Stop if already an l2meta exists. After yielding, it wouldn't
             * be valid any more, so we'd have to clean up the old L2Metas
             * and deal with requests depending on them before starting to
             * gather new ones. Not worth the trouble. */
            if (bytes == 0 && *m) {
                *cur_bytes = 0;
                return 0;
            }

            if (bytes == 0) {
                /* Wait for the dependency to complete. We need to recheck
                 * the free/allocated clusters when we continue. 将当前分配
                 * 请求挂到old_alloc->dependent_requests上，以等待old_alloc
                 * 请求完成之后再继续，，该情况下@cur_bytes的值是什么无所谓*/
                qemu_co_queue_wait(&old_alloc->dependent_requests, &s->lock);
                return -EAGAIN;
            }
        }
    }

    /* Make sure that existing clusters and new allocations are only used up to
     * the next dependency if we shortened the request above， 当前分配至多只能
     * 分配bytes大小，通过@cur_bytes返回调用者*/
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

    /*要么@host_offset为0，要么@guest_offset和@host_offset在cluster中的偏移一样*/
    assert(*host_offset == 0 ||    offset_into_cluster(s, guest_offset)
                                == offset_into_cluster(s, *host_offset));

    /*
     * Calculate the number of clusters to look for. We stop at L2 table
     * boundaries to keep things simple. 计算以@guest_offset为起始地址，以
     * @bytes为大小的区域涉及多少个cluster
     */
    nb_clusters =
        size_to_clusters(s, offset_into_cluster(s, guest_offset) + *bytes);

    /*offset_to_l2_index: 计算指定的offset在l2 table中的索引，其中会用到s->l2_size表示每一个l1 table entry中的l2 table entry的数目*/
    l2_index = offset_to_l2_index(s, guest_offset);
    nb_clusters = MIN(nb_clusters, s->l2_size - l2_index);
    assert(nb_clusters <= INT_MAX);

    /* Find L2 entry for the first involved cluster，返回@l2_table和l2_index*/
    ret = get_cluster_table(bs, guest_offset, &l2_table, &l2_index);
    if (ret < 0) {
        return ret;
    }

    cluster_offset = be64_to_cpu(l2_table[l2_index]);

    /* Check how many clusters are already allocated and don't need COW */
    if (qcow2_get_cluster_type(cluster_offset) == QCOW2_CLUSTER_NORMAL
        && (cluster_offset & QCOW_OFLAG_COPIED))
    {
        /* If a specific host_offset is required, check it */
        bool offset_matches =
            (cluster_offset & L2E_OFFSET_MASK) == *host_offset;

        /*@cluster_offset & L2E_OFFSET_MASK必须对齐到cluster*/
        if (offset_into_cluster(s, cluster_offset & L2E_OFFSET_MASK)) {
            qcow2_signal_corruption(bs, true, -1, -1, "Data cluster offset "
                                    "%#llx unaligned (guest offset: %#" PRIx64
                                    ")", cluster_offset & L2E_OFFSET_MASK,
                                    guest_offset);
            ret = -EIO;
            goto out;
        }

        if (*host_offset != 0 && !offset_matches) {
            *bytes = 0;
            ret = 0;
            goto out;
        }

        /* We keep all QCOW_OFLAG_COPIED clusters，计算有多少个cluster设置了QCOW_OFLAG_COPIED*/
        keep_clusters =
            count_contiguous_clusters(nb_clusters, s->cluster_size,
                                      &l2_table[l2_index],
                                      QCOW_OFLAG_COPIED | QCOW_OFLAG_ZERO);
        assert(keep_clusters <= nb_clusters);

        /*更新@bytes，将设置了QCOW_OFLAG_COPIED标识的cluster排除在外*/
        *bytes = MIN(*bytes,
                 keep_clusters * s->cluster_size
                 - offset_into_cluster(s, guest_offset));

        ret = 1;
    } else {
        ret = 0;
    }

    /* Cleanup */
out:
    qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

    /* Only return a host offset if we actually made progress. Otherwise we
     * would make requirements for handle_alloc() that it can't fulfill 
     * 更新@host_offset*/
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

    /* seek to the l2 offset in the l1 table，计算指定的@offset所在的l1 table中的索引，如果@l1_index大于当前记录的s->l1_size（记录当前l1 table中entry的数目）,则扩大l1 table*/
    l1_index = offset >> (s->l2_bits + s->cluster_bits);
    if (l1_index >= s->l1_size) {
        ret = qcow2_grow_l1_table(bs, l1_index + 1, false);
        if (ret < 0) {
            return ret;
        }
    }

    /*l2 offset表示l2 table的起始偏移，必须是cluster对齐的*/
    assert(l1_index < s->l1_size);
    l2_offset = s->l1_table[l1_index] & L1E_OFFSET_MASK;
    if (offset_into_cluster(s, l2_offset)) {
        qcow2_signal_corruption(bs, true, -1, -1, "L2 table offset %#" PRIx64
                                " unaligned (L1 index: %#" PRIx64 ")",
                                l2_offset, l1_index);
        return -EIO;
    }

    /*如果设置了QCOW_OFLAG_COPIED标识，表示在l2 table cache中???*/
    /* seek the l2 table of the given l2 offset */
    if (s->l1_table[l1_index] & QCOW_OFLAG_COPIED) {
        /* load the l2 table in memory, 从l2 table cache中加载之*/
        ret = l2_load(bs, l2_offset, &l2_table);
        if (ret < 0) {
            return ret;
        }
    } else {
        /* First allocate a new L2 table (and do COW if needed)，分配新的l2 table*/
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

    if (exact_size) {
        /*只用扩大到@min_size大小即可*/
        new_l1_size = min_size;
    } else {
        /* Bump size up to reduce the number of times we have to grow，扩大到原来的3/2倍*/
        new_l1_size = s->l1_size;
        if (new_l1_size == 0) {
            new_l1_size = 1;
        }
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

    /*重新分配l1 table，分配的大小要对齐到512字节*/
    new_l1_size2 = sizeof(uint64_t) * new_l1_size;
    new_l1_table = qemu_try_blockalign(bs->file->bs,
                                       align_offset(new_l1_size2, 512));
    if (new_l1_table == NULL) {
        return -ENOMEM;
    }
    memset(new_l1_table, 0, align_offset(new_l1_size2, 512));

    if (s->l1_size) {
        /*拷贝旧l1 table中的内容到新的l1 table中*/
        memcpy(new_l1_table, s->l1_table, s->l1_size * sizeof(uint64_t));
    }

    /* write new table (align to cluster) ，为l1 table分配新的cluster中*/
    BLKDBG_EVENT(bs->file, BLKDBG_L1_GROW_ALLOC_TABLE);
    new_l1_table_offset = qcow2_alloc_clusters(bs, new_l1_size2);
    if (new_l1_table_offset < 0) {
        qemu_vfree(new_l1_table);
        return new_l1_table_offset;
    }

    /*flush @s->refcount_block_cache，因为为l1 table分配新的cluster的时候会更新refcount block*/
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    if (ret < 0) {
        goto fail;
    }

    /* the L1 position has not yet been updated, so these clusters must
     * indeed be completely free ，通过qcow2_pre_write_overlap_check ->
     * qcow2_check_metadata_overlap检查@new_l1_table_offset是否和重要的
     * 元数据区域存在交集，如果存在交集，则存在corrupt metadata，报错*/
    ret = qcow2_pre_write_overlap_check(bs, 0, new_l1_table_offset,
                                        new_l1_size2);
    if (ret < 0) {
        goto fail;
    }

    /*将新的l1 table写入镜像文件*/
    BLKDBG_EVENT(bs->file, BLKDBG_L1_GROW_WRITE_TABLE);
    for(i = 0; i < s->l1_size; i++)
        new_l1_table[i] = cpu_to_be64(new_l1_table[i]);
    ret = bdrv_pwrite_sync(bs->file, new_l1_table_offset,
                           new_l1_table, new_l1_size2);
    if (ret < 0)
        goto fail;
    for(i = 0; i < s->l1_size; i++)
        new_l1_table[i] = be64_to_cpu(new_l1_table[i]);

    /* set new table， 更新镜像文件的header部分*/
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
    /*释放旧的l1 table区域，会更新refcount block，表示这些cluster被释放了*/
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
/*
 * l2_allocate
 *
 * Allocate a new l2 entry in the file. If l1_index points to an already
 * used entry in the L2 table (i.e. we are doing a copy on write for the L2
 * table) copy the contents of the old L2 table into the newly allocated one.
 * Otherwise the new table is initialized with zeros.
 *
 */

static int l2_allocate(BlockDriverState *bs, int l1_index, uint64_t **table)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t old_l2_offset;
    uint64_t *l2_table = NULL;
    int64_t l2_offset;
    int ret;

    old_l2_offset = s->l1_table[l1_index];
    trace_qcow2_l2_allocate(bs, l1_index);

    /* allocate a new l2 entry，分配新的l2 table*/
    l2_offset = qcow2_alloc_clusters(bs, s->l2_size * sizeof(uint64_t));
    if (l2_offset < 0) {
        ret = l2_offset;
        goto fail;
    }

    /*flush refcount table cache*/
    ret = qcow2_cache_flush(bs, s->refcount_block_cache);
    if (ret < 0) {
        goto fail;
    }

    /* allocate a new entry in the l2 cache */
    trace_qcow2_l2_allocate_get_empty(bs, l1_index);
    ret = qcow2_cache_get_empty(bs, s->l2_table_cache, l2_offset, (void**) table);
    if (ret < 0) {
        goto fail;
    }

    l2_table = *table;
    if ((old_l2_offset & L1E_OFFSET_MASK) == 0) {
        /*表明这是一个全新的l2 table*/
        /* if there was no old l2 table, clear the new table */
        memset(l2_table, 0, s->l2_size * sizeof(uint64_t));
    } else {
        /*表明这是一个旧的l2 table*/
        uint64_t* old_table;

        /* if there was an old l2 table, read it from the disk */
        BLKDBG_EVENT(bs->file, BLKDBG_L2_ALLOC_COW_READ);
        ret = qcow2_cache_get(bs, s->l2_table_cache,
            old_l2_offset & L1E_OFFSET_MASK,
            (void**) &old_table);
        if (ret < 0) {
            goto fail;
        }

        memcpy(l2_table, old_table, s->cluster_size);
        qcow2_cache_put(bs, s->l2_table_cache, (void **) &old_table);
    }

    /* write the l2 table to the file，将l2 table写入镜像文件*/
    BLKDBG_EVENT(bs->file, BLKDBG_L2_ALLOC_WRITE);
    trace_qcow2_l2_allocate_write_l2(bs, l1_index);
    qcow2_cache_entry_mark_dirty(bs, s->l2_table_cache, l2_table);
    ret = qcow2_cache_flush(bs, s->l2_table_cache);
    if (ret < 0) {
        goto fail;
    }

    /* update the L1 entry，更新l1 table entry，并通过QCOW_OFLAG_COPIED标记
     * l1_index对应的l2 table已经存在于l2 table cache中了*/
    trace_qcow2_l2_allocate_write_l1(bs, l1_index);
    s->l1_table[l1_index] = l2_offset | QCOW_OFLAG_COPIED;
    ret = qcow2_write_l1_entry(bs, l1_index);
    if (ret < 0) {
        goto fail;
    }

    *table = l2_table;
    trace_qcow2_l2_allocate_done(bs, l1_index, 0);
    return 0;

fail:
    trace_qcow2_l2_allocate_done(bs, l1_index, ret);
    if (l2_table != NULL) {
        qcow2_cache_put(bs, s->l2_table_cache, (void**) table);
    }
    s->l1_table[l1_index] = old_l2_offset;
    if (l2_offset > 0) {
        qcow2_free_clusters(bs, l2_offset, s->l2_size * sizeof(uint64_t),
                            QCOW2_DISCARD_ALWAYS);
    }
    return ret;
}
```

```
/*
 * Writes one sector of the L1 table to the disk (can't update single entries
 * and we really don't want bdrv_pread to perform a read-modify-write)，只更新
 * @l1_index 对应的L1 table entry所在的sector到镜像文件中
 */
#define L1_ENTRIES_PER_SECTOR (512 / 8)
int qcow2_write_l1_entry(BlockDriverState *bs, int l1_index)
{
    BDRVQcow2State *s = bs->opaque;
    uint64_t buf[L1_ENTRIES_PER_SECTOR] = { 0 };
    int l1_start_index;
    int i, ret;

    /*@l1_start_index表示@l1_index对齐到sector之后，它所在的sector中的第一个l1 table entry的索引*/
    l1_start_index = l1_index & ~(L1_ENTRIES_PER_SECTOR - 1);
    /*依次遍历该sector中的每一个l1 table entry*/
    for (i = 0; i < L1_ENTRIES_PER_SECTOR && l1_start_index + i < s->l1_size;
         i++)
    {
        buf[i] = cpu_to_be64(s->l1_table[l1_start_index + i]);
    }

    /*更新l1 table entry之前，先检查和元数据所在的区域是否存在冲突*/
    ret = qcow2_pre_write_overlap_check(bs, QCOW2_OL_ACTIVE_L1,
            s->l1_table_offset + 8 * l1_start_index, sizeof(buf));
    if (ret < 0) {
        return ret;
    }

    /*更新l1 table entry到镜像文件*/
    BLKDBG_EVENT(bs->file, BLKDBG_L1_UPDATE);
    ret = bdrv_pwrite_sync(bs->file,
                           s->l1_table_offset + 8 * l1_start_index,
                           buf, sizeof(buf));
    if (ret < 0) {
        return ret;
    }

    return 0;
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
        nb_clusters = count_cow_clusters(s, nb_clusters, l2_table, l2_index);
    }

    /* This function is only called when there were no non-COW clusters, so if
     * we can't find any unallocated or COW clusters either, something is
     * wrong with our code. */
    assert(nb_clusters > 0);

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

        nb_clusters = preallocated_nb_clusters;
        alloc_cluster_offset = entry & L2E_OFFSET_MASK;

        /* We want to reuse these clusters, so qcow2_alloc_cluster_link_l2()
         * should not free them. */
        keep_old_clusters = true;
    }

    qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

    if (!alloc_cluster_offset) {
        /* Allocate, if necessary at a given offset in the image file */
        alloc_cluster_offset = start_of_cluster(s, *host_offset);
        ret = do_alloc_cluster_offset(bs, guest_offset, &alloc_cluster_offset,
                                      &nb_clusters);
        if (ret < 0) {
            goto fail;
        }

        /* Can't extend contiguous allocation， 分配不了*/
        if (nb_clusters == 0) {
            *bytes = 0;
            return 0;
        }

        /* !*host_offset would overwrite the image header and is reserved for
         * "no host offset preferred". If 0 was a valid host offset, it'd
         * trigger the following overlap check; do that now to avoid having an
         * invalid value in *host_offset. 如果@alloc_cluster_offset为0，则需要
         * 检查是否和元数据区域有重叠*/
        if (!alloc_cluster_offset) {
            ret = qcow2_pre_write_overlap_check(bs, 0, alloc_cluster_offset,
                                                nb_clusters * s->cluster_size);
            assert(ret < 0);
            goto fail;
        }
    }

    /*
     * 保留相关元数据信息（主要是QCowL2Meta），用于后续更新元数据。
     * 
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
    uint64_t requested_bytes = *bytes + offset_into_cluster(s, guest_offset);
    int avail_bytes = MIN(INT_MAX, nb_clusters << s->cluster_bits);
    int nb_bytes = MIN(requested_bytes, avail_bytes);
    QCowL2Meta *old_m = *m;

    /*分配并初始化新的QCowL2Meta*/
    *m = g_malloc0(sizeof(**m));
    **m = (QCowL2Meta) {
        .next           = old_m,

        .alloc_offset   = alloc_cluster_offset,
        .offset         = start_of_cluster(s, guest_offset),
        .nb_clusters    = nb_clusters,

        .keep_old_clusters  = keep_old_clusters,

        .cow_start = {
            .offset     = 0,
            .nb_bytes   = offset_into_cluster(s, guest_offset),
        },
        .cow_end = {
            .offset     = nb_bytes,
            .nb_bytes   = avail_bytes - nb_bytes,
        },
    };
    
    /*将新分配的QCowL2Meta连接到@s->cluster_allocs中*/
    qemu_co_queue_init(&(*m)->dependent_requests);
    QLIST_INSERT_HEAD(&s->cluster_allocs, *m, next_in_flight);

    /*更新@host_offset和@bytes*/
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
 *
 * 如果@*host_offset不为0，则必须在@*host_offset所在的位置分配cluster，
 * 否则可以在任意位置分配，并且在该函数返回的时候设置@*host_offset为实际分配的位置
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
        int64_t cluster_offset =
            qcow2_alloc_clusters(bs, *nb_clusters * s->cluster_size);
        if (cluster_offset < 0) {
            return cluster_offset;
        }
        *host_offset = cluster_offset;
        return 0;
    } else {
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
int qcow2_alloc_cluster_link_l2(BlockDriverState *bs, QCowL2Meta *m)
{
    BDRVQcow2State *s = bs->opaque;
    int i, j = 0, l2_index, ret;
    uint64_t *old_cluster, *l2_table;
    uint64_t cluster_offset = m->alloc_offset;

    trace_qcow2_cluster_link_l2(qemu_coroutine_self(), m->nb_clusters);
    assert(m->nb_clusters > 0);

    old_cluster = g_try_new(uint64_t, m->nb_clusters);
    if (old_cluster == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    /* copy content of unmodified sectors，执行copy on write*/
    ret = perform_cow(bs, m);
    if (ret < 0) {
        goto err;
    }

    /* Update L2 table. */
    if (s->use_lazy_refcounts) {
        /* 更新镜像文件的header中的incompatible bits，设置QCOW2_INCOMPAT_DIRTY标识，
         * 表示尚未更新refcount(这就是所谓的lazy refcount)，先在header中设置
         * QCOW2_INCOMPAT_DIRTY，并flush到镜像文件中，然后再在内存中设置该标识，
         * 见qcow2_mark_dirty*/
        qcow2_mark_dirty(bs);
    }
    
    /* 看是否需要精确的refcount，如果没有设置QCOW2_INCOMPAT_DIRTY标识就是需要精确
     * refcount，设置l2_table_cache依赖于refcount_table_cache*/
    if (qcow2_need_accurate_refcounts(s)) {
        qcow2_cache_set_dependency(bs, s->l2_table_cache,
                                   s->refcount_block_cache);
    }

    ret = get_cluster_table(bs, m->offset, &l2_table, &l2_index);
    if (ret < 0) {
        goto err;
    }
    
    /*标记@l2_table为脏*/
    qcow2_cache_entry_mark_dirty(bs, s->l2_table_cache, l2_table);
    assert(l2_index + m->nb_clusters <= s->l2_size);
    for (i = 0; i < m->nb_clusters; i++) {
        /* if two concurrent writes happen to the same unallocated cluster
         * each write allocates separate cluster and writes data concurrently.
         * The first one to complete updates l2 table with pointer to its
         * cluster the second one has to do RMW (which is done above by
         * perform_cow()), update l2 table with its cluster pointer and free
         * old cluster. This is what this loop does */
        if (l2_table[l2_index + i] != 0) {
            /*l2_table[l2_index + i]如果不为0，则记录的是旧地址，表明需要执行COW*/
            old_cluster[j++] = l2_table[l2_index + i];
        }

        /*更新l2_table*/
        l2_table[l2_index + i] = cpu_to_be64((cluster_offset +
                    (i << s->cluster_bits)) | QCOW_OFLAG_COPIED);
     }

    qcow2_cache_put(bs, s->l2_table_cache, (void **) &l2_table);

    /*
     * If this was a COW, we need to decrease the refcount of the old cluster.
     *
     * Don't discard clusters that reach a refcount of 0 (e.g. compressed
     * clusters), the next write will reuse them anyway.
     * 减少这些需要COW的cluster的引用计数
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
static int perform_cow(BlockDriverState *bs, QCowL2Meta *m)
{
    BDRVQcow2State *s = bs->opaque;
    Qcow2COWRegion *start = &m->cow_start;
    Qcow2COWRegion *end = &m->cow_end;
    unsigned buffer_size;
    /*@data_bytes: end COW和start COW之间的区域大小*/
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

    /* If we have to read both the start and end COW regions and the
     * middle region is not too large then perform just one read
     * operation */
    merge_reads = start->nb_bytes && end->nb_bytes && data_bytes <= 16384;
    if (merge_reads) {
        /*一次性读取start COW、region between start COW and end COW、end COW*/
        buffer_size = start->nb_bytes + data_bytes + end->nb_bytes;
    } else {
        /* If we have to do two reads, add some padding in the middle
         * if necessary to make sure that the end region is optimally
         * aligned. 分两次读取start COW、region between start COW and end COW、
         * end COW*/
        size_t align = bdrv_opt_mem_align(bs);
        assert(align > 0 && align <= UINT_MAX);
        assert(QEMU_ALIGN_UP(start->nb_bytes, align) <=
               UINT_MAX - end->nb_bytes);
        buffer_size = QEMU_ALIGN_UP(start->nb_bytes, align) + end->nb_bytes;
    }

    /* Reserve a buffer large enough to store all the data that we're
     * going to read */
    start_buffer = qemu_try_blockalign(bs, buffer_size);
    if (start_buffer == NULL) {
        return -ENOMEM;
    }
    
    /* The part of the buffer where the end region is located */
    end_buffer = start_buffer + buffer_size - end->nb_bytes;
    qemu_iovec_init(&qiov, 2 + (m->data_qiov ? m->data_qiov->niov : 0));

    qemu_co_mutex_unlock(&s->lock);
    /* First we read the existing data from both COW regions. We
     * either read the whole region in one go, or the start and end
     * regions separately. 
     * 下面通过do_perform_cow_read分一次或者两次读取start COW和end COW
     * ，其中@m->offset表示guest看到的offset，在do_perform_cow_read中
     * 通过driver的bdrv_co_preadv来读取*/
    if (merge_reads) {
        qemu_iovec_add(&qiov, start_buffer, buffer_size);
        ret = do_perform_cow_read(bs, m->offset, start->offset, &qiov);
    } else {
        qemu_iovec_add(&qiov, start_buffer, start->nb_bytes);
        ret = do_perform_cow_read(bs, m->offset, start->offset, &qiov);
        if (ret < 0) {
            goto fail;
        }

        qemu_iovec_reset(&qiov);
        qemu_iovec_add(&qiov, end_buffer, end->nb_bytes);
        ret = do_perform_cow_read(bs, m->offset, end->offset, &qiov);
    }
    if (ret < 0) {
        goto fail;
    }

    /* Encrypt the data if necessary before writing it，加密即将写入的数据
     *，@m->alloc_offset表示host看到的offset，也是即将写入的位置*/
    if (bs->encrypted) {
        if (!do_perform_cow_encrypt(bs, m->offset, m->alloc_offset,
                                    start->offset, start_buffer,
                                    start->nb_bytes) ||
            !do_perform_cow_encrypt(bs, m->offset, m->alloc_offset,
                                    end->offset, end_buffer, end->nb_bytes)) {
            ret = -EIO;
            goto fail;
        }
    }

    /* And now we can write everything. If we have the guest data we
     * can write everything in one single operation，@m->data_qiov不为空，
     * 则其中存放的是gutst data，则同时写入start COW、guest data、end COW*/
    if (m->data_qiov) {
        /*重置@qiov*/
        qemu_iovec_reset(&qiov);
        /*先将@start_buffer接入@qiov中*/
        if (start->nb_bytes) {
            qemu_iovec_add(&qiov, start_buffer, start->nb_bytes);
        }
        
        /*将guest data @m->data_qiov连到@qiov后面*/
        qemu_iovec_concat(&qiov, m->data_qiov, 0, data_bytes);
        /*再将@end_buffer接入@qiov中*/
        if (end->nb_bytes) {
            qemu_iovec_add(&qiov, end_buffer, end->nb_bytes);
        }
        /* NOTE: we have a write_aio blkdebug event here followed by
         * a cow_write one in do_perform_cow_write(), but there's only
         * one single I/O operation */
        BLKDBG_EVENT(bs->file, BLKDBG_WRITE_AIO);
        ret = do_perform_cow_write(bs, m->alloc_offset, start->offset, &qiov);
    } else {
        /* If there's no guest data then write both COW regions separately，
         * 没有guest data，则分别写入start COW和end COW*/
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

    if (qiov->size == 0) {
        return 0;
    }

    BLKDBG_EVENT(bs->file, BLKDBG_COW_READ);

    if (!bs->drv) {
        return -ENOMEDIUM;
    }

    /* Call .bdrv_co_readv() directly instead of using the public block-layer
     * interface.  This avoids double I/O throttling and request tracking,
     * which can lead to deadlock when block layer copy-on-read is enabled.
     */
    ret = bs->drv->bdrv_co_preadv(bs, src_cluster_offset + offset_in_cluster,
                                  qiov->size, qiov, 0);
    if (ret < 0) {
        return ret;
    }

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

    if (qiov->size == 0) {
        return 0;
    }

    ret = qcow2_pre_write_overlap_check(bs, 0,
            cluster_offset + offset_in_cluster, qiov->size);
    if (ret < 0) {
        return ret;
    }

    /*写入，参考bdrv_co_pwritev*/
    BLKDBG_EVENT(bs->file, BLKDBG_COW_WRITE);
    ret = bdrv_co_pwritev(bs->file, cluster_offset + offset_in_cluster,
                          qiov->size, qiov, 0);
    if (ret < 0) {
        return ret;
    }

    return 0;
}
```

```
int qcow2_cache_set_dependency(BlockDriverState *bs, Qcow2Cache *c,
    Qcow2Cache *dependency)
{
    int ret;

    if (dependency->depends) {
        ret = qcow2_cache_flush_dependency(bs, dependency);
        if (ret < 0) {
            return ret;
        }
    }

    if (c->depends && (c->depends != dependency)) {
        ret = qcow2_cache_flush_dependency(bs, c);
        if (ret < 0) {
            return ret;
        }
    }

    c->depends = dependency;
    return 0;
}
```

```
/*
 * Free a cluster using its L2 entry (handles clusters of all types, e.g.
 * normal cluster, compressed cluster, etc.)
 * 实际上主要调用qcow2_free_clusters，以减少cluster的引用计数
 */
void qcow2_free_any_clusters(BlockDriverState *bs, uint64_t l2_entry,
                             int nb_clusters, enum qcow2_discard_type type)
{
    BDRVQcow2State *s = bs->opaque;

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
    ret = update_refcount(bs, offset, size, 1, true, type);
    if (ret < 0) {
        fprintf(stderr, "qcow2_free_clusters failed: %s\n", strerror(-ret));
        /* TODO Remember the clusters to free them later and avoid leaking */
    }
}
```

