# 提纲
[toc]

## MemSQL的数据存储在哪里
在创建Table的时候，用于指定Table相关的数据是存储在内存中还是硬盘中。

MemSQL有两种类型的Table：rowstore table和columnstore table。RowStore table将数据全部保存在内存中，使用日志和snapshot来确保数据的持久性，主要用于OLTP场景，尤其是需要快速更新的场景。ColumnStore table，将数据保存在硬盘中，但是会在需要的时候利用内存来缓存数据，主要用于OLAP场景。

## MemSQL是如何确保rowstore table的数据的持久性
MemSQL借助于log和snapshot来确保数据的持久性，在数据发生变更时，MemSQL会写日志(关于MemSQL写日志下面会有进一步的解释)，同时MemSQL会周期性的触发关于内存中数据的全量备份(也就是snapshot)，用户可以配置全量快照的频率。

MemSQL中，当一个事务被写入内存中的transaction buffer之后就认为该事务已经被成功提交了(committed)，在后台会有一个线程不断的将transaction buffer中的数据写入到日志文件中。transaction buffer的大小是可以配置的，如果将transaction buffer设置为0 MB，则意味着当且仅当事务被写入日志之后才被成功提交(committed)，如果将transaction buffer设置为其它大小，则MemSQL可以将较大块的数据一次性的写入硬盘中。

![image](https://www.memsql.com/images/content/durability/durability-1.png)

## 对MemSQL HTAP的可能的误解
为MemSQL既支持适用于OLTP场景的rowstore，又支持适用于OLAP场景的columnstore，那么它是不是就认为是比较好的支持HTAP了呢？其实不是，因为MemSQL在创建table的时候必须指定是创建基于rowstore的table还是创建基于columnstore的table，最终创建的这个表，如果是rowstore则对OLTP支持比较好，如果是columnstore则对OLAP支持比较好，并没有达到HTAP所要求的一套存储同时很好的解决OLTP和OLAP的要求。

同时在MemSQL的官方博客中，也明确提到：基于内存的rowstore适用于OLTP和HTAP场景，而基于硬盘的columnstore适用于OLAP场景。

## MemSQL SingleStore – And Then There Was One
SingleStore的命名可能源于原来MemSQL支持rowstore和columnstore，也就是DualStore，现在想将二者统一，因此命名为SingleStore。

SingleStore的产生背景：
- 用户要求MemSQL提升rowstore的TCO，因为当table变得越来越大的时候，用户必须配置更大的内存，成本较高；
- 用户要求在ColumnStore上提供类OLTP的功能特性，比如快速的UPSERTs等；

因此，MemSQL希望打造了SingleStore，终极目标是：
- 如果有足够的内存，针对OLTP和HTAP场景，SingleStore的性能可以和现有的rowstore媲美
- 如果没有足够的内存(也就是说可能需要将数据存储到硬盘上)，针对OLTP场景， SingleStore的性能优雅的下降(degrade gracefully)
- 针对OLAP场景，SingleStore的性能跟ColumnStore相媲美

为了实现上述目标，SingleStore在2方面进行改进：
- 改进RowStore，以使得在内存大小相同的情况下，稀疏的rowstore能够存放更多的数据，以在保持性能的前提下提升TCO
    - 借助于内存压缩(in-memory compresssion) 
- 改进ColumnStore，以允许高并发的随机读写访问
    - 借助于hash索引、新的行级锁和subsegment

SingleStore解决了DualStore中的什么问题：

![image](https://www.memsql.com/blog/wp-content/uploads/2019/10/MemSQL-70-Dual-Store-is-becoming-SingleStore.png)
> 图片中删除线代表的就是SingleStore解决的问题


但是从上文来看，SingleStore并不是真正的“一个同时适用于OLTP和OLAP的存储引擎”，至少现在不是，在未来可能会变成这样：“ In MemSQL 7.0, SingleStore is delivered through improvements to both rowstore and columnstore tables, allowing each to handle workloads that previously only worked well on the other. With SingleStore, in future versions of MemSQL, we will eliminate the need to choose one table type or another. The system will optimize storage and data access for you.  In the future, SingleStore will offer the fastest possible performance, at the lowest possible cost, for every kind of workload – transactional, analytical, and hybrid – by storing all data in a single table type.”。

![image](https://www.memsql.com/blog/wp-content/uploads/2019/10/MemSQL-70-Vision-for-SingleStore.png)

参考：https://www.memsql.com/blog/memsql-singlestore-then-there-was-one/

## MemSQL 7.0的System of Record能力
System of Record能力：如果一个数据库充当的是System of Record的角色，则它决不能丢失任何一个它已经告诉用户接收了该事务的事务。

在MemSQL 7.0中，通过2个功能来提供System of Record能力：
- fast synchronous replication
    ![image](https://www.memsql.com/blog/wp-content/uploads/2019/09/diagram_sync-replication-and-durability-1.png)
    >     当且仅当事务在Master上成功持久化，且在Slave上成功持久化，才会任务事务成功提交
- incremental backups
    - 只有自上次backup之后发生了编程的数据需要被backup(之前只支持full backup)
        - 减少了backup所需要的时间
        - RPO(recovery point objective)也缩短了
        - 更少的数据丢失

参考：https://www.memsql.com/blog/replication-system-of-record-memsql-7-0/

## The Story Behind MemSQL’s Skiplist Indexes
Skiplist是一个有序数据结构，对于查找，插入和删除提供o(Log(n))的时间复杂度，不像BTree，红黑树和AVL tree，它无需进行复杂的balancing和page splitting工作。

Lock-free的skiplist近些年已有所发展，请参考[Lock-Free Linked Lists and Skiplists](http://www.cse.yorku.ca/~ruppert/papers/lfll.pdf)

关于Skiplist的中文文档，参考[跳表SkipList的原理和实现](https://www.pianshen.com/article/1455120955/)。

