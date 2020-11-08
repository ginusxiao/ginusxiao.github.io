# 提纲
[toc]

## 内存索引
### 相关数据结构
#### 索引树中的索引节点
```
typedef struct as_index_s {

	// offset: 0
	uint16_t rc; // for now, incremented & decremented only when reducing sprig

	// offset: 2
	uint8_t : 8; // reserved for bigger rc, if needed

	// offset: 3
	uint8_t tree_id: 6;
	uint8_t color: 1;
	uint8_t : 1;

	// offset: 4
	cf_digest keyd;

	// offset: 24
	uint64_t left_h: 40;
	uint64_t right_h: 40;

	// offset: 34
	uint16_t set_id_bits: 10;
	uint16_t : 5;
	uint16_t xdr_write : 1;

	// offset: 36
	uint32_t xdr_tombstone : 1;
	uint32_t xdr_nsup_tombstone : 1;
	uint32_t void_time: 30;

	// offset: 40
	uint64_t last_update_time: 40;
	uint64_t generation: 16;

	// offset: 47
	// Used by the storage engines.
	uint64_t rblock_id: 37;		// can address 2^37 * 16b = 2Tb drive
	uint64_t n_rblocks: 19;		// is enough for 8Mb/16b = 512K rblocks
	uint64_t file_id: 7;		// can spec 2^7 = 128 drives
	uint64_t key_stored: 1;

	// offset: 55
	// In single-bin mode for data-in-memory namespaces, this offset is cast to
	// an as_bin, but only 4 bits get used (for the iparticle state). The other
	// 4 bits are used for replication state and index flags.
	uint8_t repl_state: 2;
	uint8_t tombstone: 1;
	uint8_t cenotaph: 1;
	uint8_t single_bin_state: 4; // used indirectly, only in single-bin mode

	// offset: 56
	// For data-not-in-memory namespaces, these 8 bytes are currently unused.
	// For data-in-memory namespaces: in single-bin mode the as_bin is embedded
	// here (these 8 bytes plus 4 bits in flex_bits above), but in multi-bin
	// mode this is a pointer to either of:
	// - an as_bin_space containing n_bins and an array of as_bin structs
	// - an as_rec_space containing an as_bin_space pointer and other metadata
	void* dim;

	// final size: 64

} __attribute__ ((__packed__)) as_index;
```

#### 索引树
```
typedef struct as_index_tree_s {
	uint8_t					id;
	as_index_tree_done_fn	done_cb;
	void					*udata;

	// Data common to all trees in a namespace.
	as_index_tree_shared	*shared;

	cf_atomic64				n_elements;

    # data中的布局: 
    #   如果index存放在RAM或者PMEM中：as lock pairs(256个), sprigs(至少256个，至多1<<28个)
    #   如果index存放在FLASH中：as lock pairs, sprigs, puddles
	// Variable length data, dependent on configuration.
	uint8_t					data[];
} as_index_tree;

typedef struct as_index_tree_shared_s {
	cf_arenax*		arena;

	as_index_value_destructor destructor;
	void*			destructor_udata;

	// Number of sprigs per partition tree.
	uint32_t		n_sprigs;

	// Bit-shifts used to calculate indexes from digest bits.
	uint32_t		locks_shift;
	uint32_t		sprigs_shift;

	// Offsets into as_index_tree struct's variable-sized data.
	uint32_t		sprigs_offset;
	uint32_t		puddles_offset;
} as_index_tree_shared;
```

对于每个key的digest(20 bytes)，会通过digest计算出该key应该由哪个lock pair，sprig和puddle来管理它，如果索引保存在内存中的话，则没有对应的puddle。

```
as_write_start
write_master
as_record_get_create
as_index_get_insert_vlock

int
as_index_get_insert_vlock(as_index_tree *tree, const cf_digest *keyd,
		as_index_ref *index_ref)
{
    # 从as_index_tree中获取该key由哪个sprig来管理
	as_index_sprig isprig;
	as_index_sprig_from_keyd(tree, &isprig, keyd);

	int result = as_index_sprig_get_insert_vlock(&isprig, tree->id, keyd,
			index_ref);

	if (result == 1) {
		cf_atomic64_incr(&tree->n_elements);
	}

	return result;
} 

int
as_index_sprig_get_insert_vlock(as_index_sprig *isprig, uint8_t tree_id,
		const cf_digest *keyd, as_index_ref *index_ref)
{
	int cmp = 0;

	// Use a stack as_index object for the root's parent, for convenience.
	as_index root_parent;

	// Save parents as we search for the specified element's insertion point.
	as_index_ele eles[64]; // must be >= (24 * 2)
	as_index_ele *ele;

	while (true) {
		ele = eles;

        # 对当前的sprig加锁
		cf_mutex_lock(&isprig->pair->lock);

		// Search for the specified element, or a parent to insert it under.

		root_parent.left_h = isprig->sprig->root_h;
		root_parent.color = AS_BLACK;

		ele->parent = NULL; // we'll never look this far up
		ele->me_h = 0; // root parent has no handle, never used
		ele->me = &root_parent;

		cf_arenax_handle t_h = isprig->sprig->root_h;
		as_index *t = RESOLVE_H(t_h);

		while (t_h != SENTINEL_H) {
			ele++;
			ele->parent = ele - 1;
			ele->me_h = t_h;
			ele->me = t;

			_mm_prefetch(t, _MM_HINT_NTA);

			if ((cmp = cf_digest_compare(keyd, &t->keyd)) == 0) {
				// The element already exists, simply return it.

				index_ref->r = t;
				index_ref->r_h = t_h;

				index_ref->puddle = isprig->puddle;
				index_ref->olock = &isprig->pair->lock;

				return 0;
			}

			t_h = cmp > 0 ? t->left_h : t->right_h;
			t = RESOLVE_H(t_h);
		}

		// We didn't find the tree element, so we'll be inserting it.

		if (cf_mutex_trylock(&isprig->pair->reduce_lock)) {
			break; // no reduce in progress - go ahead and insert new element
		}

		// The tree is being reduced - could take long, unlock so reads and
		// overwrites aren't blocked.
		cf_mutex_unlock(&isprig->pair->lock);

		// Wait until the tree reduce is done...
		cf_mutex_lock(&isprig->pair->reduce_lock);
		cf_mutex_unlock(&isprig->pair->reduce_lock);

		// ... and start over - we unlocked, so the tree may have changed.
	}

	// Create a new element and insert it.

	// Save the root so we can detect whether it changes.
	cf_arenax_handle old_root = isprig->sprig->root_h;

	// Make the new element.
	cf_arenax_handle n_h = cf_arenax_alloc(isprig->arena, isprig->puddle);

	if (n_h == 0) {
		cf_ticker_warning(AS_INDEX, "arenax alloc failed");
		cf_mutex_unlock(&isprig->pair->reduce_lock);
		cf_mutex_unlock(&isprig->pair->lock);
		return -1;
	}

	as_index *n = RESOLVE_H(n_h);

	memset(n, 0, sizeof(as_index));

	n->tree_id = tree_id;
	n->keyd = *keyd;

	n->left_h = n->right_h = SENTINEL_H; // n starts as a leaf element
	n->color = AS_RED; // n's color starts as red

	// Insert the new element n under parent ele.
	if (ele->me == &root_parent || 0 < cmp) {
		ele->me->left_h = n_h;
	}
	else {
		ele->me->right_h = n_h;
	}

	ele++;
	ele->parent = ele - 1;
	ele->me_h = n_h;
	ele->me = n;

	// Rebalance the sprig as needed.
	as_index_sprig_insert_rebalance(isprig, &root_parent, ele);

	// If insertion caused the root to change, save the new root.
	if (root_parent.left_h != old_root) {
		isprig->sprig->root_h = root_parent.left_h;
	}

	isprig->sprig->n_elements++;

	// Surely we won't hit 16M elements, but...
	cf_assert(isprig->sprig->n_elements != 0, AS_INDEX, "sprig overflow");

	cf_mutex_unlock(&isprig->pair->reduce_lock);

	index_ref->r = n;
	index_ref->r_h = n_h;

	index_ref->puddle = isprig->puddle;
	index_ref->olock = &isprig->pair->lock;

	return 1;
}
```


