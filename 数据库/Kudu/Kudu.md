# 提纲
[toc]

## Kudu - 一个融合低延迟写入和高性能分析的存储系统
这里我们主要关注一下Kudu中用到的存储引擎 - Tablet Storage。

Kudu 自己实现了一套 tablet storage，而没有用现有的开源解决方案。Tablet storage 目标主要包括：
- 快速的按照 Column 扫描数据
- 低延迟的随机更新
- 一致的性能

对于 Tablet Storage，虽然 Kudu 是自己实现的，但很多方面其实跟 RocksDB 差不了多少，类似 LSM 架构，只是可能这套系统专门为 Kudu 做了定制优化，而不像 RocksDB 那样具有普适性。

参考：https://pingcap.com/blog-cn/kudu/


## Kudu design-docs/tablet.md
Kudu中的Table也被划分为一个个的Tablet：
- 每一个Tablet存储连续的区间范围的行，且每一个Tablet的区间范围之间不存在交叠(Each tablet hosts a contiguous range of rows which does not overlap with any other tablet's range)
    - 这么说，是按照range进行分区的？
- 关于同一个Table的所有的Tablet共同组成该Table的key空间
- 每一个Tablet又进一步划分为一系列的RowSets
    - 每一个RowSet包含一系列的Rows
    - 不同的RowSet之间不存在交集，但是，不同的RowSet之间可能会交叠
        - 前面半句是说，任何一个给定的key都至多存在于一个RowSet中
        - 后面半句是说，不同的RowSet在区间范围上可能存在交叠

RowSet又进一步划分为MemRowSet和DiskRowSet。

MemRowSet
- 它是一个根据primary key排序的基于内存的BTree
- 所有的插入操作都首先进入MemRowSet
- 每一行都对应于MemRowSet中的一个entry

### MVCC Mutations in MemRowSet
在第一次插入一行数据的时候，在MemRowSet中会插入一个带insertion timestamp的记录，后续的关于该行数据的修改(mutation)都会添加到一个单向链表中，链表中的每一个元素都是一个带mutation timestamp的记录。
```
 MemRowSet Row
+----------------------------------------------------+
| insertion timestamp  | mutation head | row data... |
+-------------------------|--------------------------+
                          |
                          v          First mutation
                  +-----------------------------------------------+
                  | mutation timestamp | next_mut | change record |
                  +--------------------|--------------------------+
                            __________/
                           /
                           |         Second mutation
                  +--------v--------------------------------------+
                  | mutation timestamp | next_mut | change record |
                  +--------------------|--------------------------+
                            __________/
                           /
                           ...
```

这里的mutation可能是：
- UPDATE: 更新一个或者多个列的值
- DELETE: 删除一行
- REINSERT: 删除一行之后，重新插入一行

举个例子，考虑如下的操作序列:
```
  INSERT INTO t VALUES ("row", 1);         [timestamp 1]
  UPDATE t SET val = 2 WHERE key = "row";  [timestamp 2]
  DELETE FROM t WHERE key = "row";         [timestamp 3]
  INSERT INTO t VALUES ("row", 3);         [timestamp 4]
```

执行上面的操作序列之后，对应的MemRowSet可能如下：
```
  +-----------------------------------+
  | tx 1 | mutation head | ("row", 1) |
  +----------|------------------------+
             |
             |
         +---v--------------------------+
         | tx 2 | next ptr | SET val=2  |
         +-----------|------------------+
              ______/
             |
         +---v-------v----------------+
         | tx 3 | next ptr | DELETE   |
         +-----------|----------------+
              ______/
             |
         +---v------------------------------------+
         | tx 4 | next ptr | REINSERT ("row", 3)  |
         +----------------------------------------+
```

这种实现有一些缺陷：
- readers必须按照单向链表的顺序来读取数据，非常可能造成CPU cache miss；
- 每一次mutation都必须添加到单向链表的尾部，时间复杂度是o(n);

但是Kudu方面基于以下假设，认为这些缺陷是可以容忍的：
- Kudu的目标场景是：具有相对较少的行更新；
- 只有很少一部分数据是在MemRowSet中的；

### MemRowSet Flushes
当MemRowSet写满之后，会将其中的数据flush到DiskRowSet中。
```
+------------+
| MemRowSet  |
+------------+
     |
     | Flush process writes entries in memory to a new DiskRowSet on disk
     v
+--------------+  +--------------+    +--------------+
| DiskRowSet 0 |  | DiskRowSet 1 | .. | DiskRowSet N |
+-------------+-  +--------------+    +--------------+
```
DiskRowSet中的每一行都具有一个在DiskRowSet内部唯一的rowid，比如一个DiskRowSet包含5行，则rowid将从0到4。

不同的DiskRowSet中的不同的行可能具有相同的rowid。

有些读请求可能需要在主键和rowid之间进行映射(即，通过主键找到rowid)，这需要一个主键到rowid的索引表。如果主键不是组合key类型，则该索引标保存在主键列对应的CFile中，否则该索引表保存在一个单独的CFile中。


### Historical MVCC in DiskRowSets
为了在DiskRowSet上支持MVCC，每一个DiskRowSet不仅仅存放当前的数据，同时会存放UNDO记录：
```
+--------------+       +-----------+
| UNDO records | <---  | base data |
+--------------+       +-----------+
- time of data progresses to the right --->
```

举个例子，考虑如下的操作序列，假设table schema为(key STRING, val UINT32):
```
  INSERT INTO t VALUES ("row", 1);         [timestamp 1]
  UPDATE t SET val = 2 WHERE key = "row";  [timestamp 2]
  DELETE FROM t WHERE key = "row";         [timestamp 3]
  INSERT INTO t VALUES ("row", 3);         [timestamp 4]
```

执行上面的操作序列之后，对应的MemRowSet可能如下：
```
  +-----------------------------------+
  | tx 1 | mutation head | ("row", 1) |
  +----------|------------------------+
             |
             |
         +---v--------------------------+
         | tx 2 | next ptr | SET val=2  |
         +-----------|------------------+
              ______/
             |
         +---v-------v----------------+
         | tx 3 | next ptr | DELETE   |
         +-----------|----------------+
              ______/
             |
         +---v------------------------------------+
         | tx 4 | next ptr | REINSERT ("row", 3)  |
         +----------------------------------------+
```

如果将对应的MemRowSet flush到DiskRowSet之后，DiskRowSet按如下方式存储：
```
Base data:
   ("row", 3)
UNDO records (roll-back):
   Before Tx 4: DELETE
   Before Tx 3: INSERT ("row", 2")
   Before Tx 2: SET row=1
   Before Tx 1: DELETE
```

### Handling mutations against on-disk files
对于已经从MemRowSet flush到DiskRowSet中的rows的update或者delete操作将不会再进入MemRowSet，而是：
- 首先在所有的RowSets中找到该key对应的RowSet
- 然后找到对应的rowid
- 最后将update或者delete所带来的mutation写入DeltaMemStore中

DeltaMemStore是一个存储于内存中的并发的BTree，它的key由rowid和mutation timestamp组成。

当DeltaMemStore达到一定容量后，会被flush到on-disk的DeltaFile中，DeltaMemStore则被清空。

因为DeltaFile中包含那些需要被应用到Base data的数据，因此DeltaFile也被称为Redo File，其中的记录也被称为Redo records。

### Summary of delta file processing
概括来说，DiskRowSet逻辑上包括3个部分：
```
+--------------+       +-----------+      +--------------+
| UNDO records | <---  | base data | ---> | REDO records |
+--------------+       +-----------+      +--------------+
```

其中：
- Base data：表示在MemRowSet flush到DiskRowSet的时候的列数据
- UNDO records：表示在MemRowSet flush到DiskRowSet之前的历史数据，主要用户rollback
- REDO records：表示在MemRowSet flush到DiskRowSet之后的mutation 数据，主要用于获取截止到某个时间戳的数据

UNDO records和REDO records都保存在DeltaFile中。

### Delta Compactions
Delta compactions的主要目的如下：
- 减少DeltaFiles的数目
- 将REDO records转换为UNDO records
    - 因为通常的查询都是查询最新版本的数据，如果直接基于旧的Base Data进行读取，则需要一次读取Base Data和所有的REDO records，这是非常低效的，所以KUDU会尝试更新Base Data（合并就得Base Data和REDO records）并将原来的REDO records转换为UNDO records
- 回收旧的UNDO records
    - UNDO records至多需要保存到用户配置的historical retention period，超过这个period的UNDO records可以被移除

Delta compaction包括如下类型：
- Minor REDO delta compaction： 只将REDO DeltaFiles进行合并
```
+------------+      +---------+     +---------+     +---------+     +---------+
| base data  | <--- | delta 0 + <-- | delta 1 + <-- | delta 2 + <-- | delta 3 +
+------------+      +---------+     +---------+     +---------+     +---------+
                    \_________________________________________/
                           files selected for compaction

  =====>

+------------+      +---------+     +-----------------------+
| base data  | <--- | delta 0 + <-- | delta 1 (old delta 3) +
+------------+      +---------+     +-----------------------+
                    \_________/
                  compaction result
```

- Major REDO delta compaction：将Redo DeltaFiles和Base Data进行合并
```
+------------+      +---------+     +---------+     +---------+     +---------+
| base data  | <--- | delta 0 + <-- | delta 1 + <-- | delta 2 + <-- | delta 3 +
+------------+      +---------+     +---------+     +---------+     +---------+
\_____________________________________________/
      files selected for compaction

  =====>

+------------+      +----------------+      +-----------------------+     +-----------------------+
| new UNDOs  | -->  | new base data  | <--- | delta 0 (old delta 2) + <-- | delta 1 (old delta 3) +
+------------+      +----------------+      +-----------------------+     +-----------------------+
\____________________________________/
           compaction result
```

- Merging compactions：将RowSets进行合并
```
+------------+
| RowSet 0   |
+------------+

+------------+ \
| RowSet 1   | |
+------------+ |
               |
+------------+ |                            +--------------+
| RowSet 2   | |===> RowSet compaction ===> | new RowSet 1 |
+------------+ |                            +--------------+
               |
+------------+ |
| RowSet 3   | |
+------------+ /
```

> 不同的RowSet之间合并是因为不同的RowSet所代表的的区间范围上可能存在交叠，在查找某些行的数据的时候，因为不确定在哪个RowSet中，所以需要逐个RowSet进行查找，这会比较低效，但是合并之后，就可以一定程度上解决该问题？


### 一张图来描述MemRowSet和DiskRowSet的变迁
```
+-----------+
| MemRowSet |
+-----------+
  |
  | flush: creates a new DiskRowSet 0
  v
+---------------+
| DiskRowSet 0  |
+---------------+

DiskRowSet 1:
+---------+     +------------+      +---------+     +---------+     +---------+     +---------+
| UNDOs 0 | --> | base data  | <--- | REDOs 0 | <-- | REDOS 1 | <-- | REDOs 2 | <-- | REDOs 3 |
+---------+     +------------+      +---------+     +---------+     +---------+     +---------+
\____________________________________________________________/
                           | major compaction
                           v

+---------+     +------------+      +---------+     +---------+
| UNDOs 0'| --> | base data' | <--- | REDOs 2 | <-- | REDOs 3 |
+---------+     +------------+      +---------+     +---------+
\____________________________/
      compaction result


DiskRowSet 2:
+---------+     +------------+      +---------+     +---------+     +---------+     +---------+
| UNDOs 0 | --> | base data  | <--- | REDOs 0 | <-- | REDOS 1 | <-- | REDOs 2 | <-- | REDOs 3 |
+---------+     +------------+      +---------+     +---------+     +---------+     +---------+
                                    \_________________________/
                                         | minor compaction
                                         v
+---------+     +------------+      +---------+      +---------+     +---------+
| UNDOs 0 | --> | base data  | <--- | REDOS 0'|  <-- | REDOs 2 | <-- | REDOs 3 |
+---------+     +------------+      +---------+      +---------+     +---------+
                                    \_________/
                                 compaction result

+-----------------+ \
| DiskRowSet 3    | |
+-----------------+ |
                    |
+-----------------+ |                              +----------------+
| DiskRowSet 4    | |===> Merging compaction ===>  | new DiskRowSet |
+-----------------+ |                              +----------------+
                    |
+-----------------+ |
| DiskRowSet 5    | |
+-----------------+ /
```


## Kudu design-docs/cfile.md
每一个DiskRowSet可能会包含以下CFile：
- 每一列数据；
- 每一个DeltaFile
- DiskRowSet的bloomfileter
- 如果Table具有组合主键，则从主键到rowid的索引也会存放在一个单独的CFile中

一个CFile并不一定对应于文件系统中的一个文件，CFile到文件的映射由BlockManager决定。

一个CFile由header，blocks和footer组成，blocks又进一步划分为多种类型：data blocks, nullable data blocks, index blocks和dictionary blocks。

每一个CFile可选的可以包含一个positional(ordinal) index和value index。Positional index主要用于类似这样的查询：seek to the data block containing the Nth entry in this CFile；而value index主要用于类似这样的查询：seek to the data block containing 123 in this CFile。Value index只在数据有序的CFile中存在(比如primary key column)。

positional(ordinal) index和value index被组织成BTree结构。
