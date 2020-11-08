# 提纲
[toc]

## TiDB对HTAP的理解 - Real-Time HTAP
TiDB CTO谈到了他自己对Real-Time HTAP的理解，主要有以下标准：
- 业务透明的**无限水平扩展能力**
- 业务层几乎**无需妥协**
    - SQL支持
    - 分布式事务
    - 复杂查询的能力
- 故障自恢复的**高可用**能力
- 高性能**实时分析**能力
    - 列式存储引擎是必选项
- **在混合负载下，实时OLAP分析不影响OLTP事务**

参考：[未来数据库应具备什么核心能力](https://pingcap.com/blog-cn/core-competence-of-future-database/)

## TiDB之TiFlash
### TiFlash是什么
TiFlash 是一款支持更新的列存引擎，在实时镜像行存数据的同时，提供数倍乃至数十倍以上的对行存的访问加速。它可以使用独立节点，以完全隔绝两种业务之间的互相影响。它部分基于 Clickhouse，继承了 Clickhouse 优秀的向量化计算引擎。

架构上，TiFlash 是一个提供与 TiKV 类似功能的存储层模块，它使用 Raft Learner 进行数据传输并转存为列存。这意味着，TiFlash 节点的状态和延迟，不会影响 TiKV 的运行，哪怕 TiFlash 节点崩溃，TiKV 也能毫无影响地运行；另一方面也可以提供最新（线性一致 + 快照隔离），和 TiKV 一致的数据。

![image](https://download.pingcap.com/images/blog-cn/10x-improving-analytical-processing-ability-of-tidb-with-tiflash/2-TiFlash.png)
![image](https://download.pingcap.com/images/blog-cn/10x-improving-analytical-processing-ability-of-tidb-with-tiflash/3-TiFlash.png)
### 查询
- 用户无需主动选择是使用TiKV还是TiFlash进行查询
    - TiDB支持从TiFlash读取，同时将列存纳入了优化器代价估算中，因此优化器可以帮用户决定是从TiKV读取还是从TiFlash读取。
- 用户可以选择强制从TiFlash查询
    - 如果你有业务隔离的需求，也可以简单执行如下命令强制隔离：
    ```
    set @@session.tidb_isolation_read_engines = "tiflash";
    ```
### 特点
- 高性能
- 简化技术栈
    - TiFlash可以近似看做是一种特殊的TiKV节点
- 新鲜且一致的数据
- 隔离
    - 关闭 TiDB 自动选择，或者用 TiSpark 开启 TiFlash 模式 
- 智能
    - 关闭隔离设定，你可以让 TiDB 自主选择是使用TiKV还是TiFlash进行查询
- 灵活扩容

### TiDB为什么这么快
- 采用Delta Main列存方案
    - 将需要更新数据与整理好的不可变列存块分开存放，读时归并，定期 Compact
- 参考其他成熟系统设计
    -  Apache Kudu，CWI 的 Positional Delta Tree 等
- 牺牲了对 TiFlash 场景无用的点查性能
    - 由于无需考虑点查，因此 TiFlash 可以以进行惰性数据整理加速写入
        - 相对 LSM 引擎减小了写放大比率
            - LSM 结构下，RocksDB 的写放大在 10 倍左右
            - TiFlash 的写放大大约在 3-7 倍之间
- 引入了读时排序索引回写
    - 哪怕 Compact 不那么频繁仍可以保持扫描高效，进一步减小写放大加速写入

![image](https://download.pingcap.com/images/blog-cn/tiflash-column-database/1-tiflash-design.png)

## TiSpark
TiSpark 是将 Spark SQL 直接运行在分布式存储引擎 TiKV 上的 OLAP 解决方案。

![image](https://download.pingcap.com/images/docs-cn/tispark-architecture.png)

[TiSpark简单总结](http://note.youdao.com/noteshare?id=8d64d05c107d20d734229d7cd5e93af4&sub=C261C538544146958EB860059000FE47)

## Titan 的设计与实现
Titan 是由 PingCAP 研发的一个基于 RocksDB 的高性能单机 key-value 存储引擎，其主要设计灵感来源于 USENIX FAST 2016 上发表的一篇论文 WiscKey。WiscKey 提出了一种高度基于 SSD 优化的设计，利用 SSD 高效的随机读写性能，通过将 value 分离出 LSM-tree 的方法来达到降低写放大的目的。

### 设计目标
- 支持将 value 从 LSM-tree 中分离出来单独存储，以降低写放大
- 已有 RocksDB 实例可以平滑地升级到 Titan，这意味着升级过程不需要人工干预，并且不会影响线上服务
- 100% 兼容目前 TiKV 所使用的所有 RocksDB 的特性
- 尽量减少对 RocksDB 的侵入性改动，保证 Titan 更加容易升级到新版本的 RocksDB

![image](https://download.pingcap.com/images/blog-cn/titan-design-and-implementation/1.jpg)
> Titan 在 Flush 和 Compaction 的时候将 value 分离出 LSM-tree，这样做的好处是写入流程可以和 RockDB 保持一致，减少对 RocksDB 的侵入性改动。

### 核心组件
Titan 的核心组件主要包括：BlobFile、TitanTableBuilder、Version 和 GC。

#### BlobFile
BlobFile 是用来存放从 LSM-tree 中分离出来的 value 的文件。
![image](https://download.pingcap.com/images/blog-cn/titan-design-and-implementation/2.jpg)
> BlobFile 主要由 blob record 、meta block、meta index block 和 footer 组成。其中每个 blob record 用于存放一个 key-value 对；meta block 支持可扩展性，可以用来存放和 BlobFile 相关的一些属性等；meta index block 用于检索 meta block。

#### TitanTableBuilder
TitanTableBuilder是实现分离 key-value 的关键。我们知道 RocksDB 支持使用用户自定义 table builder 创建 SST，这使得我们可以不对 build table 流程做侵入性的改动就可以将 value 从 SST 中分离出来。

![image](https://download.pingcap.com/images/blog-cn/titan-design-and-implementation/3.jpg)
> TitanTableBuilder 通过判断 value size 的大小来决定是否将 value 分离到 BlobFile 中去。如果 value size 大于等于 min_blob_size 则将 value 分离到 BlobFile ，并生成 index 写入 SST；如果 value size 小于 min_blob_size 则将 value 直接写入 SST。

#### Version
Titan 使用 Version 来代表某个时间点所有有效的 BlobFile，这是从 LevelDB 中借鉴过来的管理数据文件的方法，其核心思想便是 MVCC，好处是在新增或删除文件的同时，可以做到并发读取数据而不需要加锁。每次新增文件或者删除文件的时候，Titan 都会生成一个新的 Version ，并且每次读取数据之前都要获取一个最新的 Version。

![image](https://download.pingcap.com/images/blog-cn/titan-design-and-implementation/4.png)

#### Garbage Collection
Garbage Collection (GC) 的目的是回收空间，一个高效的 GC 算法应该在权衡写放大和空间放大的同时，用最少的周期来回收最多的空间。在设计 GC 的时候有两个主要的问题需要考虑：
- 何时进行 GC
- 挑选哪些文件进行 GC

Titan 使用 RocksDB 提供的两个特性来解决这两个问题，这两个特性分别是 TablePropertiesCollector 和 EventListener 。下面将讲解我们是如何通过这两个特性来辅助 GC 工作的。

**BlobFileSizeCollector**
RocksDB 允许我们使用自定义的 TablePropertiesCollector 来搜集 SST 上的 properties 并写入到对应文件中去。Titan 通过一个自定义的 TablePropertiesCollector —— BlobFileSizeCollector 来搜集每个 SST 中有多少数据是存放在哪些 BlobFile 上的，我们将它收集到的 properties 命名为 BlobFileSizeProperties，它的工作流程和数据格式如下图所示：

![image](https://download.pingcap.com/images/blog-cn/titan-design-and-implementation/5.jpg)
> 图 5：左边 SST 中 Index 的格式为：第一列代表 BlobFile 的文件 ID，第二列代表 blob record 在 BlobFile 中的 offset，第三列代表 blob record 的 size。右边 BlobFileSizeProperties 中的每一行代表一个 BlobFile 以及 SST 中有多少数据保存在这个 BlobFile 中，第一列代表 BlobFile 的文件 ID，第二列代表数据大小。

**EventListener**
我们知道 RocksDB 是通过 Compaction 来丢弃旧版本数据以回收空间的，因此每次 Compaction 完成后 Titan 中的某些 BlobFile 中便可能有部分或全部数据过期。因此我们便可以通过监听 Compaction 事件来触发 GC，通过搜集比对 Compaction 中输入输出 SST 的 BlobFileSizeProperties 来决定挑选哪些 BlobFile 进行 GC。其流程大概如下图所示：

![image](https://download.pingcap.com/images/blog-cn/titan-design-and-implementation/6.jpg)
> 图 6：inputs 代表参与 Compaction 的所有 SST 的 BlobFileSizeProperties，outputs 代表 Compaction 生成的所有 SST 的 BlobFileSizeProperties，discardable size 是通过计算 inputs 和 outputs 得出的每个 BlobFile 被丢弃的数据大小，第一列代表 BlobFile 的文件 ID，第二列代表被丢弃的数据大小。

Titan 会为每个有效的 BlobFile 在内存中维护一个 discardable size 变量，每次 Compaction 结束之后都对相应的 BlobFile 的 discardable size 变量进行累加。每次 GC 开始时就可以通过挑选 discardable size 最大的 BlobFile 来作为作为候选的文件。

## PAX：一个 Cache 友好高效的行列混存方案
PAX是google spanner中使用的一个行列混存的数据组织方式。一个基本的思想就是：同一行的数据尽量保持在同一个page里面，而同一个page里面的数据又会按照列分为不同的mini page，每一个mini page存放一个列的数据。

参考：https://pingcap.com/blog-cn/pax/

## 基于 Tile 连接 Row-Store 和 Column-Store
简单来说，Peloton 使用 physical tile 将数据切分成了不同的单元，同时用 logical tile 来隐藏了相关的实现，并提供 algebra 让上层更方便的操作。在后台，统计 query 等信息，并定期增量的对 layout 进行 reorganization。

参考：https://pingcap.com/blog-cn/tile-row-store/

## TiKV 功能介绍 - Raft 的优化
本文主要提到如下几个可行的优化方案：
- Batch and Pipeline
    - batch就是批量提交请求给raft
    - pipeline就是说leader在给follower发送了复制请求之后，无需等待follower的响应就可以继续发送下一个复制请求
- Append Log Parallelly
    - leader在接收到请求之后，立即发送请求到follower上，同时自己进行append log操作
- Asynchronous Apply
    - 当一个log被committed之后，用另一个线程去异步的apply这个log
- SST Snapshot
    - 在 Raft 里面，如果 Follower 落后 Leader 太多，Leader 就可能会给 Follower 直接发送 snapshot，如果一个节点上面同时有多个 Raft Group 的 Follower 在处理 snapshot file，RocksDB 的写入压力会非常的大，然后极易引起 RocksDB 因为 compaction 处理不过来导致的整体写入 slow 或者 stall
    - 借助RocksDB 提供的SST 机制，可以直接生成一个 SST 的 snapshot file，然后 Follower 通过 injest 接口直接将 SST file load 进入 RocksDB
- Asynchronous Lease Read
    - 将 Leader Lease 的判断移到另一个线程异步进行，Raft 这边的线程会定期的通过消息去更新 Lease，这样就能保证 Raft 的 write 流程不会影响到 read

参考：https://pingcap.com/blog-cn/optimizing-raft-in-tikv/


