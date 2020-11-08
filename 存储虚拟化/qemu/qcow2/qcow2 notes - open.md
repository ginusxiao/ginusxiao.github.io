# 打开镜像文件


```
static BlockBackend *img_open(bool image_opts,
                              const char *filename,
                              const char *fmt, int flags, bool writethrough,
                              bool quiet, bool force_share)
{
    BlockBackend *blk;
    if (image_opts) {
        QemuOpts *opts;
        if (fmt) {
            error_report("--image-opts and --format are mutually exclusive");
            return NULL;
        }
        opts = qemu_opts_parse_noisily(qemu_find_opts("source"),
                                       filename, true);
        if (!opts) {
            return NULL;
        }
        blk = img_open_opts(filename, opts, flags, writethrough, quiet,
                            force_share);
    } else {
        /*这里options设置为NULL*/
        blk = img_open_file(filename, NULL, fmt, flags, writethrough, quiet,
                            force_share);
    }
    
    return blk;
}
```

```
static BlockBackend *img_open_file(const char *filename,
                                   QDict *options,
                                   const char *fmt, int flags,
                                   bool writethrough, bool quiet,
                                   bool force_share)
{
    BlockBackend *blk;
    Error *local_err = NULL;

    /*分配新的@options*/
    if (!options) {
        options = qdict_new();
    }
    
    /*在@options中设置driver选项*/
    if (fmt) {
        qdict_put_str(options, "driver", fmt);
    }

    if (force_share) {
        qdict_put_bool(options, BDRV_OPT_FORCE_SHARE, true);
    }
    
    /*创建新的BlockBackend和BlockDriverState并关联两者*/
    blk = blk_new_open(filename, NULL, options, flags, &local_err);
    if (!blk) {
        error_reportf_err(local_err, "Could not open '%s': ", filename);
        return NULL;
    }
    
    /*如果@writethrough为false，则设置有write cache*/
    blk_set_enable_write_cache(blk, !writethrough);

    return blk;
}
```


```
/*
 * Creates a new BlockBackend, opens a new BlockDriverState, and connects both.
 *
 * Just as with bdrv_open(), after having called this function the reference to
 * @options belongs to the block layer (even on failure).
 *
 * TODO: Remove @filename and @flags; it should be possible to specify a whole
 * BDS tree just by specifying the @options QDict (or @reference,
 * alternatively). At the time of adding this function, this is not possible,
 * though, so callers of this function have to be able to specify @filename and
 * @flags.
 */
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

    blk = blk_new(perm, BLK_PERM_ALL);
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
/*
 * Create a new BlockBackend with a reference count of one.
 *
 * @perm is a bitmasks of BLK_PERM_* constants which describes the permissions
 * to request for a block driver node that is attached to this BlockBackend.
 * @shared_perm is a bitmask which describes which permissions may be granted
 * to other users of the attached node.
 * Both sets of permissions can be changed later using blk_set_perm().
 *
 * Return the new BlockBackend on success, null on failure.
 */
BlockBackend *blk_new(uint64_t perm, uint64_t shared_perm)
{
    BlockBackend *blk;

    blk = g_new0(BlockBackend, 1);
    blk->refcnt = 1;
    blk->perm = perm;
    blk->shared_perm = shared_perm;
    blk_set_enable_write_cache(blk, true);

    block_acct_init(&blk->stats);

    notifier_list_init(&blk->remove_bs_notifiers);
    notifier_list_init(&blk->insert_bs_notifiers);

    QTAILQ_INSERT_TAIL(&block_backends, blk, link);
    return blk;
}
```

```
BlockDriverState *bdrv_open(const char *filename, const char *reference,
                            QDict *options, int flags, Error **errp)
{
    return bdrv_open_inherit(filename, reference, options, flags, NULL,
                             NULL, errp);
}
```

```
bdrv_open_inherit(const char *filename,
                                           const char *reference,
                                           QDict *options, int flags,
                                           BlockDriverState *parent,
                                           const BdrvChildRole *child_role,
                                           Error **errp)
{
    int ret;
    BlockBackend *file = NULL;
    BlockDriverState *bs;
    BlockDriver *drv = NULL;
    const char *drvname;
    const char *backing;
    Error *local_err = NULL;
    QDict *snapshot_options = NULL;
    int snapshot_flags = 0;

    assert(!child_role || !flags);
    assert(!child_role == !parent);

    if (reference) {
        bool options_non_empty = options ? qdict_size(options) : false;
        QDECREF(options);

        if (filename || options_non_empty) {
            error_setg(errp, "Cannot reference an existing block device with "
                       "additional options or a new filename");
            return NULL;
        }

        bs = bdrv_lookup_bs(reference, reference, errp);
        if (!bs) {
            return NULL;
        }

        bdrv_ref(bs);
        return bs;
    }

    /*分配新的BlockDriverState*/
    bs = bdrv_new();

    /* NULL means an empty set of options */
    if (options == NULL) {
        options = qdict_new();
    }

    /* json: syntax counts as explicit options, as if in the QDict */
    parse_json_protocol(options, &filename, &local_err);
    if (local_err) {
        goto fail;
    }

    bs->explicit_options = qdict_clone_shallow(options);

    if (child_role) {
        bs->inherits_from = parent;
        child_role->inherit_options(&flags, options,
                                    parent->open_flags, parent->options);
    }

    /*设置@options中默认选项，并将传统的filename和flags都转化为QDict entry存放于@options中*/
    ret = bdrv_fill_options(&options, filename, &flags, &local_err);
    if (local_err) {
        goto fail;
    }

    /*
     * Set the BDRV_O_RDWR and BDRV_O_ALLOW_RDWR flags.
     * Caution: getting a boolean member of @options requires care.
     * When @options come from -blockdev or blockdev_add, members are
     * typed according to the QAPI schema, but when they come from
     * -drive, they're all QString.
     */
    if (g_strcmp0(qdict_get_try_str(options, BDRV_OPT_READ_ONLY), "on") &&
        !qdict_get_try_bool(options, BDRV_OPT_READ_ONLY, false)) {
        flags |= (BDRV_O_RDWR | BDRV_O_ALLOW_RDWR);
    } else {
        flags &= ~BDRV_O_RDWR;
    }

    if (flags & BDRV_O_SNAPSHOT) {
        snapshot_options = qdict_new();
        bdrv_temp_snapshot_options(&snapshot_flags, snapshot_options,
                                   flags, options);
        /* Let bdrv_backing_options() override "read-only" */
        qdict_del(options, BDRV_OPT_READ_ONLY);
        bdrv_backing_options(&flags, options, flags, options);
    }

    bs->open_flags = flags;
    bs->options = options;
    options = qdict_clone_shallow(options);

    /* Find the right image format driver */
    /* See cautionary note on accessing @options above */
    /* 找到对应的format driver*/
    drvname = qdict_get_try_str(options, "driver");
    if (drvname) {
        drv = bdrv_find_format(drvname);
        if (!drv) {
            error_setg(errp, "Unknown driver: '%s'", drvname);
            goto fail;
        }
    }

    assert(drvname || !(flags & BDRV_O_PROTOCOL));

    /* See cautionary note on accessing @options above */
    backing = qdict_get_try_str(options, "backing");
    if (backing && *backing == '\0') {
        flags |= BDRV_O_NO_BACKING;
        qdict_del(options, "backing");
    }

    /* Open image file without format layer. This BlockBackend is only used for
     * probing, the block drivers will do their own bdrv_open_child() for the
     * same BDS, which is why we put the node name back into options. 
     * 如果没有指定BDRV_O_PROTOCOL标识，表示无需format driver来打开该镜像文件(
     * 只在probing的时候发生)，暂时不关注*/
    if ((flags & BDRV_O_PROTOCOL) == 0) {
        BlockDriverState *file_bs;

        file_bs = bdrv_open_child_bs(filename, options, "file", bs,
                                     &child_file, true, &local_err);
        if (local_err) {
            goto fail;
        }
        if (file_bs != NULL) {
            file = blk_new(BLK_PERM_CONSISTENT_READ, BLK_PERM_ALL);
            blk_insert_bs(file, file_bs, &local_err);
            bdrv_unref(file_bs);
            if (local_err) {
                goto fail;
            }

            qdict_put_str(options, "file", bdrv_get_node_name(file_bs));
        }
    }

    /* Image format probing，如果到此@drv为空，则是probing???*/
    bs->probed = !drv;
    if (!drv && file) {
        ret = find_image_format(file, filename, &drv, &local_err);
        if (ret < 0) {
            goto fail;
        }
        /*
         * This option update would logically belong in bdrv_fill_options(),
         * but we first need to open bs->file for the probing to work, while
         * opening bs->file already requires the (mostly) final set of options
         * so that cache mode etc. can be inherited.
         *
         * Adding the driver later is somewhat ugly, but it's not an option
         * that would ever be inherited, so it's correct. We just need to make
         * sure to update both bs->options (which has the full effective
         * options for bs) and options (which has file.* already removed).
         */
        qdict_put_str(bs->options, "driver", drv->format_name);
        qdict_put_str(options, "driver", drv->format_name);
    } else if (!drv) {
        error_setg(errp, "Must specify either driver or file");
        goto fail;
    }

    /* BDRV_O_PROTOCOL must be set iff a protocol BDS is about to be created */
    assert(!!(flags & BDRV_O_PROTOCOL) == !!drv->bdrv_file_open);
    /* file must be NULL if a protocol BDS is about to be created
     * (the inverse results in an error message from bdrv_open_common()) */
    assert(!(flags & BDRV_O_PROTOCOL) || !file);

    /* Open the image */
    ret = bdrv_open_common(bs, file, options, &local_err);
    if (ret < 0) {
        goto fail;
    }

    if (file) {
        blk_unref(file);
        file = NULL;
    }

    /* If there is a backing file, use it */
    if ((flags & BDRV_O_NO_BACKING) == 0) {
        ret = bdrv_open_backing_file(bs, options, "backing", &local_err);
        if (ret < 0) {
            goto close_and_fail;
        }
    }

    bdrv_refresh_filename(bs);

    /* Check if any unknown options were used */
    if (qdict_size(options) != 0) {
        const QDictEntry *entry = qdict_first(options);
        if (flags & BDRV_O_PROTOCOL) {
            error_setg(errp, "Block protocol '%s' doesn't support the option "
                       "'%s'", drv->format_name, entry->key);
        } else {
            error_setg(errp,
                       "Block format '%s' does not support the option '%s'",
                       drv->format_name, entry->key);
        }

        goto close_and_fail;
    }

    bdrv_parent_cb_change_media(bs, true);

    QDECREF(options);

    /* For snapshot=on, create a temporary qcow2 overlay. bs points to the
     * temporary snapshot afterwards. */
    if (snapshot_flags) {
        BlockDriverState *snapshot_bs;
        snapshot_bs = bdrv_append_temp_snapshot(bs, snapshot_flags,
                                                snapshot_options, &local_err);
        snapshot_options = NULL;
        if (local_err) {
            goto close_and_fail;
        }
        /* We are not going to return bs but the overlay on top of it
         * (snapshot_bs); thus, we have to drop the strong reference to bs
         * (which we obtained by calling bdrv_new()). bs will not be deleted,
         * though, because the overlay still has a reference to it. */
        bdrv_unref(bs);
        bs = snapshot_bs;
    }

    return bs;

fail:
    blk_unref(file);
    QDECREF(snapshot_options);
    QDECREF(bs->explicit_options);
    QDECREF(bs->options);
    QDECREF(options);
    bs->options = NULL;
    bs->explicit_options = NULL;
    bdrv_unref(bs);
    error_propagate(errp, local_err);
    return NULL;

close_and_fail:
    bdrv_unref(bs);
    QDECREF(snapshot_options);
    QDECREF(options);
    error_propagate(errp, local_err);
    return NULL;
}
```

```
BlockDriverState *bdrv_new(void)
{
    BlockDriverState *bs;
    int i;

    bs = g_new0(BlockDriverState, 1);
    QLIST_INIT(&bs->dirty_bitmaps);
    for (i = 0; i < BLOCK_OP_TYPE_MAX; i++) {
        QLIST_INIT(&bs->op_blockers[i]);
    }
    notifier_with_return_list_init(&bs->before_write_notifiers);
    qemu_co_mutex_init(&bs->reqs_lock);
    qemu_mutex_init(&bs->dirty_bitmap_mutex);
    bs->refcnt = 1;
    bs->aio_context = qemu_get_aio_context();

    qemu_co_queue_init(&bs->flush_queue);

    QTAILQ_INSERT_TAIL(&all_bdrv_states, bs, bs_list);

    return bs;
}
```

```
/*
 * Fills in default options for opening images and converts the legacy
 * filename/flags pair to option QDict entries.
 * The BDRV_O_PROTOCOL flag in *flags will be set or cleared accordingly if a
 * block driver has been specified explicitly.
 */
static int bdrv_fill_options(QDict **options, const char *filename,
                             int *flags, Error **errp)
{
    const char *drvname;
    bool protocol = *flags & BDRV_O_PROTOCOL;
    bool parse_filename = false;
    BlockDriver *drv = NULL;
    Error *local_err = NULL;

    /*
     * Caution: while qdict_get_try_str() is fine, getting non-string
     * types would require more care.  When @options come from
     * -blockdev or blockdev_add, its members are typed according to
     * the QAPI schema, but when they come from -drive, they're all
     * QString.
     */
    /*在img_open_file中设置@options中的“driver”选项为创建的镜像文件的format*/
    drvname = qdict_get_try_str(*options, "driver");
    if (drvname) {
        drv = bdrv_find_format(drvname);
        if (!drv) {
            error_setg(errp, "Unknown driver '%s'", drvname);
            return -ENOENT;
        }
        /* If the user has explicitly specified the driver, this choice should
         * override the BDRV_O_PROTOCOL flag */
        protocol = drv->bdrv_file_open;
    }

    if (protocol) {
        *flags |= BDRV_O_PROTOCOL;
    } else {
        *flags &= ~BDRV_O_PROTOCOL;
    }

    /* Translate cache options from flags into options */
    update_options_from_flags(*options, *flags);

    /* Fetch the file name from the options QDict if necessary */
    if (protocol && filename) {
        if (!qdict_haskey(*options, "filename")) {
            qdict_put_str(*options, "filename", filename);
            parse_filename = true;
        } else {
            error_setg(errp, "Can't specify 'file' and 'filename' options at "
                             "the same time");
            return -EINVAL;
        }
    }

    /* Find the right block driver */
    /* See cautionary note on accessing @options above */
    filename = qdict_get_try_str(*options, "filename");

    /*如果@drivername为空，即没有指定format，但是设置了BDRV_O_PROTOCOL标识（表示如果没有显示指定block driver，
     * 就为其选择一个protocol driver）*/
    if (!drvname && protocol) {
        if (filename) {
            /*从文件名中解析出protocol driver*/
            drv = bdrv_find_protocol(filename, parse_filename, errp);
            if (!drv) {
                return -EINVAL;
            }

            /*采用protocol driver默认的format*/
            drvname = drv->format_name;
            qdict_put_str(*options, "driver", drvname);
        } else {
            error_setg(errp, "Must specify either driver or file");
            return -EINVAL;
        }
    }

    assert(drv || !protocol);

    /* Driver-specific filename parsing，如果driver有其自己的文件名解析函数，则调用之*/
    if (drv && drv->bdrv_parse_filename && parse_filename) {
        drv->bdrv_parse_filename(filename, *options, &local_err);
        if (local_err) {
            error_propagate(errp, local_err);
            return -EINVAL;
        }

        if (!drv->bdrv_needs_filename) {
            qdict_del(*options, "filename");
        }
    }

    return 0;
}
```

```
/*
 * Common part for opening disk images and files
 *
 * Removes all processed options from *options.
 */
static int bdrv_open_common(BlockDriverState *bs, BlockBackend *file,
                            QDict *options, Error **errp)
{
    int ret, open_flags;
    const char *filename;
    const char *driver_name = NULL;
    const char *node_name = NULL;
    const char *discard;
    const char *detect_zeroes;
    QemuOpts *opts;
    BlockDriver *drv;
    Error *local_err = NULL;

    assert(bs->file == NULL);
    assert(options != NULL && bs->options != options);

    /*创建一个新的QemuOpts结构@opts并添加到全局的@brev_runtime_opts中*/
    opts = qemu_opts_create(&bdrv_runtime_opts, NULL, 0, &error_abort);
    /* 将@options中的相关选项添加到@opts中并从@options中移除之，对于那些
     * 不能添加到@opts中的选项仍然保留在@options中*/
    qemu_opts_absorb_qdict(opts, options, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        ret = -EINVAL;
        goto fail_opts;
    }

    /* 从@opts中解析出open flag：
     * BDRV_O_NOCACHE：
     *      不使用host page cache，通过cache.direct选项来设置，默认是false
     * BDRV_O_NO_FLUSH：
     *      禁用flush，通过cache.no-flush选项来设置，默认是false
     * BDRV_OPT_READ_ONLY：
     *      只读，通过read-only选项来设置，默认是false
     */
    update_flags_from_options(&bs->open_flags, opts);

    /*获取driver名称并找到driver*/
    driver_name = qemu_opt_get(opts, "driver");
    drv = bdrv_find_format(driver_name);
    assert(drv != NULL);

    bs->force_share = qemu_opt_get_bool(opts, BDRV_OPT_FORCE_SHARE, false);

    if (bs->force_share && (bs->open_flags & BDRV_O_RDWR)) {
        error_setg(errp,
                   BDRV_OPT_FORCE_SHARE
                   "=on can only be used with read-only images");
        ret = -EINVAL;
        goto fail_opts;
    }

    if (file != NULL) {
        filename = blk_bs(file)->filename;
    } else {
        /*
         * Caution: while qdict_get_try_str() is fine, getting
         * non-string types would require more care.  When @options
         * come from -blockdev or blockdev_add, its members are typed
         * according to the QAPI schema, but when they come from
         * -drive, they're all QString.
         */
        filename = qdict_get_try_str(options, "filename");
    }

    if (drv->bdrv_needs_filename && (!filename || !filename[0])) {
        error_setg(errp, "The '%s' block driver requires a file name",
                   drv->format_name);
        ret = -EINVAL;
        goto fail_opts;
    }

    trace_bdrv_open_common(bs, filename ?: "", bs->open_flags,
                           drv->format_name);

    bs->read_only = !(bs->open_flags & BDRV_O_RDWR);

    if (use_bdrv_whitelist && !bdrv_is_whitelisted(drv, bs->read_only)) {
        error_setg(errp,
                   !bs->read_only && bdrv_is_whitelisted(drv, true)
                        ? "Driver '%s' can only be used for read-only devices"
                        : "Driver '%s' is not whitelisted",
                   drv->format_name);
        ret = -ENOTSUP;
        goto fail_opts;
    }

    /* bdrv_new() and bdrv_close() make it so */
    assert(atomic_read(&bs->copy_on_read) == 0);
    /*如果指定了BDRV_O_COPY_ON_READ标识，则表示从backing image中拷贝读取的数据到当前镜像中*/
    if (bs->open_flags & BDRV_O_COPY_ON_READ) {
        if (!bs->read_only) {
            bdrv_enable_copy_on_read(bs);
        } else {
            error_setg(errp, "Can't use copy-on-read on read-only device");
            ret = -EINVAL;
            goto fail_opts;
        }
    }

    /* 是否支持guest触发的discard（trim/unmap）命令，如果支持，则需要在
     * @bs->open_flags中设置BDRV_O_UNMAP标识，表示支持discard*/
    discard = qemu_opt_get(opts, "discard");
    if (discard != NULL) {
        if (bdrv_parse_discard_flags(discard, &bs->open_flags) != 0) {
            error_setg(errp, "Invalid discard option");
            ret = -EINVAL;
            goto fail_opts;
        }
    }

    /*是否支持detect-zeroes*/
    detect_zeroes = qemu_opt_get(opts, "detect-zeroes");
    if (detect_zeroes) {
        BlockdevDetectZeroesOptions value =
            qapi_enum_parse(&BlockdevDetectZeroesOptions_lookup,
                            detect_zeroes,
                            BLOCKDEV_DETECT_ZEROES_OPTIONS_OFF,
                            &local_err);
        if (local_err) {
            error_propagate(errp, local_err);
            ret = -EINVAL;
            goto fail_opts;
        }

        if (value == BLOCKDEV_DETECT_ZEROES_OPTIONS_UNMAP &&
            !(bs->open_flags & BDRV_O_UNMAP))
        {
            error_setg(errp, "setting detect-zeroes to unmap is not allowed "
                             "without setting discard operation to unmap");
            ret = -EINVAL;
            goto fail_opts;
        }

        bs->detect_zeroes = value;
    }

    if (filename != NULL) {
        pstrcpy(bs->filename, sizeof(bs->filename), filename);
    } else {
        bs->filename[0] = '\0';
    }
    pstrcpy(bs->exact_filename, sizeof(bs->exact_filename), bs->filename);

    /* Open the image, either directly or using a protocol */
    open_flags = bdrv_open_flags(bs, bs->open_flags);
    node_name = qemu_opt_get(opts, "node-name");

    assert(!drv->bdrv_file_open || file == NULL);
    ret = bdrv_open_driver(bs, drv, node_name, options, open_flags, errp);
    if (ret < 0) {
        goto fail_opts;
    }

    qemu_opts_del(opts);
    return 0;

fail_opts:
    qemu_opts_del(opts);
    return ret;
}
```

```
static int bdrv_open_driver(BlockDriverState *bs, BlockDriver *drv,
                            const char *node_name, QDict *options,
                            int open_flags, Error **errp)
{
    Error *local_err = NULL;
    int ret;

    bdrv_assign_node_name(bs, node_name, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        return -EINVAL;
    }

    bs->drv = drv;
    bs->read_only = !(bs->open_flags & BDRV_O_RDWR);
    bs->opaque = g_malloc0(drv->instance_size);

    /*对于qcow2来说，提供了bdrv_open函数，见qcow2_open*/
    if (drv->bdrv_file_open) {
        assert(!drv->bdrv_needs_filename || bs->filename[0]);
        ret = drv->bdrv_file_open(bs, options, open_flags, &local_err);
    } else if (drv->bdrv_open) {
        ret = drv->bdrv_open(bs, options, open_flags, &local_err);
    } else {
        ret = 0;
    }

    if (ret < 0) {
        if (local_err) {
            error_propagate(errp, local_err);
        } else if (bs->filename[0]) {
            error_setg_errno(errp, -ret, "Could not open '%s'", bs->filename);
        } else {
            error_setg_errno(errp, -ret, "Could not open image");
        }
        goto open_failed;
    }

    /*更新img size*/
    ret = refresh_total_sectors(bs, bs->total_sectors);
    if (ret < 0) {
        error_setg_errno(errp, -ret, "Could not refresh total sector count");
        return ret;
    }

    /*block IO limits，详见bdrv_refresh_limits*/
    bdrv_refresh_limits(bs, &local_err);
    if (local_err) {
        error_propagate(errp, local_err);
        return -EINVAL;
    }

    assert(bdrv_opt_mem_align(bs) != 0);
    assert(bdrv_min_mem_align(bs) != 0);
    assert(is_power_of_2(bs->bl.request_alignment));

    return 0;
open_failed:
    bs->drv = NULL;
    if (bs->file != NULL) {
        bdrv_unref_child(bs, bs->file);
        bs->file = NULL;
    }
    g_free(bs->opaque);
    bs->opaque = NULL;
    return ret;
}
```

```
static int qcow2_open(BlockDriverState *bs, QDict *options, int flags,
                      Error **errp)
{
    /*qcow2最终又是以raw格式的file作为backend，对应的驱动为bdrv_file，该函数有些复杂，暂时不关注，只需要知道@bs->file对应的BlockDriver为@bdrv_file即可*/
    bs->file = bdrv_open_child(NULL, options, "file", bs, &child_file,
                               false, errp);
    if (!bs->file) {
        return -EINVAL;
    }

    /*这是qcow2自身的open逻辑*/
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

    /*读取header部分，会等待读操作返回*/
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

    /* Initialise cluster size，在qcow2_create中会从创建选项中解析出cluster 
     * size，然后写入header中，如果创建选项中没有指定，则使用默认值64K，另外
     * cluster size必须是512B - 2M之间，且为2的幂次方*/
    if (header.cluster_bits < MIN_CLUSTER_BITS ||
        header.cluster_bits > MAX_CLUSTER_BITS) {
        error_setg(errp, "Unsupported cluster size: 2^%" PRIu32,
                   header.cluster_bits);
        ret = -EINVAL;
        goto fail;
    }

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
        be64_to_cpus(&header.incompatible_features);
        be64_to_cpus(&header.compatible_features);
        be64_to_cpus(&header.autoclear_features);
        /* refcount bits在qcow2_create中会从创建选项中解析出来，如果没有提供，
         * 则默认值为16bit，另外，refcount bits不能超过64，且必须是2的幂次方，
         * 2 ^ refcount_order = refcount_bits*/
        be32_to_cpus(&header.refcount_order);
        be32_to_cpus(&header.header_length);

        if (header.header_length < 104) {
            error_setg(errp, "qcow2 header too short");
            ret = -EINVAL;
            goto fail;
        }
    }

    /*header中记录的header大小不能超过一个cluster大小*/
    if (header.header_length > s->cluster_size) {
        error_setg(errp, "qcow2 header exceeds cluster size");
        ret = -EINVAL;
        goto fail;
    }

    /* 如果header中记录的header大小大于QCowHeader结构体的大小，则header中在
     * QCowHeader后面的是所谓的unknown_header_fileds*/
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

    /*backing file name所在偏移不能超过第一个cluster（即header所在的cluster）*/
    if (header.backing_file_offset > s->cluster_size) {
        error_setg(errp, "Invalid backing file offset");
        ret = -EINVAL;
        goto fail;
    }

    /* 设置extension区域的结束偏移(extension区域就是在第一个cluster中除
     * QCowHeader以外的区域)*/
    if (header.backing_file_offset) {
        ext_end = header.backing_file_offset;
    } else {
        ext_end = 1 << header.cluster_bits;
    }

    /* Handle feature bits */
    s->incompatible_features    = header.incompatible_features;
    s->compatible_features      = header.compatible_features;
    s->autoclear_features       = header.autoclear_features;

    /*处理header extension中的imcompatible_feature部分*/
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

    /* Check support for various header values，refcount_bits最大为64，所以
     * refcount_order至多为6*/
    if (header.refcount_order > 6) {
        error_setg(errp, "Reference count entry width too large; may not "
                   "exceed 64 bits");
        ret = -EINVAL;
        goto fail;
    }
    s->refcount_order = header.refcount_order;
    s->refcount_bits = 1 << s->refcount_order;
    s->refcount_max = UINT64_C(1) << (s->refcount_bits - 1);
    s->refcount_max += s->refcount_max - 1;

    /*从header中拷贝加密相关的设置到BlockDriverState中*/
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

    /*refcount_table_offset必须是cluster对齐的*/
    ret = validate_table_offset(bs, s->refcount_table_offset,
                                s->refcount_table_size, sizeof(uint64_t));
    if (ret < 0) {
        error_setg(errp, "Invalid reference count table offset");
        goto fail;
    }

    /* Snapshot table offset/length*/
    if (header.nb_snapshots > QCOW_MAX_SNAPSHOTS) {
        error_setg(errp, "Too many snapshots");
        ret = -EINVAL;
        goto fail;
    }

    /*snapshot_offset必须是cluster对齐的*/
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

    /*l1_table_offset必须是cluster对齐的*/
    ret = validate_table_offset(bs, header.l1_table_offset,
                                header.l1_size, sizeof(uint64_t));
    if (ret < 0) {
        error_setg(errp, "Invalid L1 table offset");
        goto fail;
    }
    s->l1_table_offset = header.l1_table_offset;

    /*预加载l1 table*/
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

    /* Parse driver-specific options，采用qcow2自身的配置选项来更新@options*/
    ret = qcow2_update_options(bs, options, flags, errp);
    if (ret < 0) {
        goto fail;
    }

    s->cluster_cache_offset = -1;
    s->flags = flags;

    /*预加载refcount table*/
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

    /* read the backing file name，读取backing file name*/
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

    /*暂时不关注之*/
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
void bdrv_refresh_limits(BlockDriverState *bs, Error **errp)
{
    BlockDriver *drv = bs->drv;
    Error *local_err = NULL;

    /*@bs->bl类型为BlockLimits，用于IO相关限制*/
    memset(&bs->bl, 0, sizeof(bs->bl));

    if (!drv) {
        return;
    }

    /* Default alignment based on whether driver has byte interface */
    /*默认地，如果驱动支持字节级读写，则对齐到1，否则对齐到512*/
    bs->bl.request_alignment = drv->bdrv_co_preadv ? 1 : 512;

    /* Take some limits from the children as a default */
    /*如果child driver有其自己的driver，则也应用child driver的IO限制*/
    if (bs->file) {
        /*对于qcow2来说，其child driver是@bdrv_file，所以会递归调用bdrv_refresh_limits*/
        bdrv_refresh_limits(bs->file->bs, &local_err);
        if (local_err) {
            error_propagate(errp, local_err);
            return;
        }
        bdrv_merge_limits(&bs->bl, &bs->file->bs->bl);
    } else {
        /*如果没有child driver，比如@bdrv_file就没有child driver，则设置内存对齐和最大iovec数目相关*/
        bs->bl.min_mem_alignment = 512;
        bs->bl.opt_mem_alignment = getpagesize();

        /* Safe default since most protocols use readv()/writev()/etc */
        bs->bl.max_iov = IOV_MAX;
    }

    /*如果有backing file driver，则也要应用backing file driver的IO限制*/
    if (bs->backing) {
        bdrv_refresh_limits(bs->backing->bs, &local_err);
        if (local_err) {
            error_propagate(errp, local_err);
            return;
        }
        bdrv_merge_limits(&bs->bl, &bs->backing->bs->bl);
    }

    /* Then let the driver override it */
    /*最后应用driver自身的IO限制，对于qcow2来说，因为会递归调用bdrv_refresh_limits，所以会先执行@bdrv_file::bdrv_refresh_limits，即raw_refresh_limits，后执行@bdrv_qcow2::bdrv_refresh_limits，即qcow2_refresh_limits*/
    if (drv->bdrv_refresh_limits) {
        drv->bdrv_refresh_limits(bs, errp);
    }
}
```

```
static void raw_refresh_limits(BlockDriverState *bs, Error **errp)
{
    BDRVRawState *s = bs->opaque;
    struct stat st;

    if (!fstat(s->fd, &st)) {
        /*如果是字符设备或者块设备*/
        if (S_ISBLK(st.st_mode) || S_ISCHR(st.st_mode)) {
            int ret = hdev_get_max_transfer_length(bs, s->fd);
            if (ret > 0 && ret <= BDRV_REQUEST_MAX_BYTES) {
                bs->bl.max_transfer = pow2floor(ret);
            }
            ret = hdev_get_max_segments(&st);
            if (ret > 0) {
                bs->bl.max_transfer = MIN(bs->bl.max_transfer,
                                          ret * getpagesize());
            }
        }
    }

    raw_probe_alignment(bs, s->fd, errp);
    bs->bl.min_mem_alignment = s->buf_align;
    bs->bl.opt_mem_alignment = MAX(s->buf_align, getpagesize());
}
```

```
static void raw_probe_alignment(BlockDriverState *bs, int fd, Error **errp)
{
    BDRVRawState *s = bs->opaque;
    char *buf;
    /*最大对齐*/
    size_t max_align = MAX(MAX_BLOCKSIZE, getpagesize());

    /* For SCSI generic devices the alignment is not really used.
       With buffered I/O, we don't have any restrictions. */
    if (bdrv_is_sg(bs) || !s->needs_alignment) {
        bs->bl.request_alignment = 1;
        s->buf_align = 1;
        return;
    }

    bs->bl.request_alignment = 0;
    s->buf_align = 0;
    /* Let's try to use the logical blocksize for the alignment. */
    /*尝试通过IOCtl获取alignment*/
    if (probe_logical_blocksize(fd, &bs->bl.request_alignment) < 0) {
        bs->bl.request_alignment = 0;
    }
#ifdef CONFIG_XFS
    /*如果配置的是xfs，则使用xfsctl来通过direct io信息获取*/
    if (s->is_xfs) {
        struct dioattr da;
        if (xfsctl(NULL, fd, XFS_IOC_DIOINFO, &da) >= 0) {
            bs->bl.request_alignment = da.d_miniosz;
            /* The kernel returns wrong information for d_mem */
            /* s->buf_align = da.d_mem; */
        }
    }
#endif

    /* If we could not get the sizes so far, we can only guess them */
    /*如果没有指定@s->buf_align，则尝试通过计算来获取，采取的方法是：首先分配一个一定是对齐的空间，大小为2 * max_align，对齐到max_align，然后从align为512字节开始逐级呈指数递增，尝试不同align下执行direct io读取是否返回EINVAL，以此来判断是否对齐的*/
    if (!s->buf_align) {
        size_t align;
        buf = qemu_memalign(max_align, 2 * max_align);
        for (align = 512; align <= max_align; align <<= 1) {
            if (raw_is_io_aligned(fd, buf + align, max_align)) {
                s->buf_align = align;
                break;
            }
        }
        qemu_vfree(buf);
    }

    /*如果指定了@s->buf_align，则在分配对齐空间的时候要对齐到s->buf_align，其它步骤跟上面一样*/
    if (!bs->bl.request_alignment) {
        size_t align;
        buf = qemu_memalign(s->buf_align, max_align);
        for (align = 512; align <= max_align; align <<= 1) {
            if (raw_is_io_aligned(fd, buf, align)) {
                bs->bl.request_alignment = align;
                break;
            }
        }
        qemu_vfree(buf);
    }

    if (!s->buf_align || !bs->bl.request_alignment) {
        error_setg(errp, "Could not find working O_DIRECT alignment");
        error_append_hint(errp, "Try cache.direct=off\n");
    }
}
```

```
/*
 * Get logical block size via ioctl. On success store it in @sector_size_p.
 */
static int probe_logical_blocksize(int fd, unsigned int *sector_size_p)
{
    unsigned int sector_size;
    bool success = false;
    int i;

    errno = ENOTSUP;
    static const unsigned long ioctl_list[] = {
#ifdef BLKSSZGET
        BLKSSZGET,
#endif
#ifdef DKIOCGETBLOCKSIZE
        DKIOCGETBLOCKSIZE,
#endif
#ifdef DIOCGSECTORSIZE
        DIOCGSECTORSIZE,
#endif
    };

    /* Try a few ioctls to get the right size */
    for (i = 0; i < (int)ARRAY_SIZE(ioctl_list); i++) {
        if (ioctl(fd, ioctl_list[i], &sector_size) >= 0) {
            *sector_size_p = sector_size;
            success = true;
        }
    }

    return success ? 0 : -errno;
}

```

```
/* Check if read is allowed with given memory buffer and length.
 *
 * This function is used to check O_DIRECT memory buffer and request alignment.
 */
static bool raw_is_io_aligned(int fd, void *buf, size_t len)
{
    ssize_t ret = pread(fd, buf, len, 0);

    if (ret >= 0) {
        return true;
    }

#ifdef __linux__
    /* The Linux kernel returns EINVAL for misaligned O_DIRECT reads.  Ignore
     * other errors (e.g. real I/O error), which could happen on a failed
     * drive, since we only care about probing alignment.
     */
    if (errno != EINVAL) {
        return true;
    }
#endif

    return false;
}
```

```
static void qcow2_refresh_limits(BlockDriverState *bs, Error **errp)
{
    BDRVQcow2State *s = bs->opaque;

    /*对于加密的情况，直接将alignment修改为512，其它情况保持不变（也就是沿用@bdrv_file::bdrv_refresh_limits中设置的alignment）*/
    if (bs->encrypted) {
        /* Encryption works on a sector granularity */
        bs->bl.request_alignment = BDRV_SECTOR_SIZE;
    }
    
    bs->bl.pwrite_zeroes_alignment = s->cluster_size;
    bs->bl.pdiscard_alignment = s->cluster_size;
}
```




