# 提纲
[toc]

## 设计上的考虑
分析发现，仅仅优化现有的SQL Server无法实现10-100倍的性能提升。提升性能主要有3种方式：提升可扩展性，提高CPI(即每条指令所需要的CPU时钟)，减少每个请求所执行的指令数目。分析还发现，即使是最乐观的情况下，提升可扩展性和提高CPI都只能带来3-4倍的性能提升。所以希望寄托在减少每个请求所执行的指令数目上，如果想要10倍的性能提升，则必须减少90%的指令，如果想要100倍的提升，则必须减少99%的指令。

### 架构原则
#### 为内存优化索引
现在主流的数据库系统都是面向硬盘的存储结构，记录都按页保存在硬盘上，在需要的时候加载到内存中。这需要复杂的buffer pool，一个页在访问之前必须加latch锁保护。即使所有的pages都在内存中，在B-tree索引中执行一个简单的查询也需要数千条指令。

Hekaton索引专门为基于内存的存储做了优化，通过日志和检查点机制来保证durability；索引相关的操作则不记录到日志。在恢复的时候，Hekaton通过最新的检查点和日志来重建表和索引。

#### 消除latches和locks
如果系统中有频繁更新的共享内存，比如latches，spinlocks和高竞争的资源，比如lock manager，transaction log 的尾部，B-tree索引的最后一个page等，可扩展性将受到严重影响。

HeKaton的所有的内部数据结构，比如memory allocator，hash和range索引，transaction map等都是latch-free(lock-free)的。在性能关键的路径上也没有任何latches或者spinlocks。Hekaton使用新的乐观的多版本并发控制来提供隔离语义，没有locks，也没有lock table。借助于乐观的并发控制，多版本和latch-free数据结构，Hekaton线程不会stall或者wait。

#### 将请求编译为机器码(machine code/native code)

### 无分区
Hekaton不对数据库进行分区，任何线程都可以访问该数据库的任何一部分。作者提到，他们认真的评估了分区的方式，但是最后拒绝了它。他们认为分区是很好，但只是在应用可以分区的情况下。如果应用分区做的不好，平均下来每个事务都需要访问多个分区的话，性能退化非常严重。

## 架构
Hekaton主要由3部分组成：
- Hekaton storage engine
    - 管理用户数据和索引
    - 支持事务操作
    - 支持hash和range索引
    - 支持检查点，数据恢复和高可用
- Hekaton compiler
- Hekaton runtime system

![Hekaton主要组件及Hekaton与SQL Server的集成](https://note.youdao.com/yws/public/resource/c84f877986d06d4747a7e9dd91a1f4ab/xmlnote/59CEBDD571694C2D8148EB3100AF4D55/107709)

Hekaton和SQL Server的主要在以下几个方面进行集成：
- 元数据：Hekaton表和索引等的元数据存储在SQL Server catalog中
- 查询优化
- 互查询：Hekaton提供一些访问Hekaton表中数据的operator，可被SQL Server的执行计划使用。另外，还提供了insert，delete和update相关的operator。
- 事务：SQL Server事务可以访问或者更新SQL Server表或者Hekaton表中的数据。
- 高可用
- 存储，日志：Hekaton日志被更新到SQL Server的transaction log中；Hekaton使用SQL Server的file stream来实现检查点。

## 存储和索引
创建table时如果指定了memory_optimized选项，则该表会被保存在Hekaton中。Hekaton支持两种索引类型：hash索引和range索引。hash索引通过lock-free的hash table实现，range索引则通过Bw-tree实现。一个表可以有多个索引，记录总是通过索引来访问。Hekaton使用多版本，更新总是创建一个新的版本。

![举例说明：在具有2个索引的account表中transaction 75从larry的账户向john转账20元](https://note.youdao.com/yws/public/resource/c84f877986d06d4747a7e9dd91a1f4ab/xmlnote/53E2831B3E634F70B24A3E9237D22550/107749)

关于在上面的例子说明如下：
- account表中有3个字段：name，city和amount。
- 有2个索引：基于hash table的hash索引，和基于Bw-tree的range索引。hash索引建立在name字段上，range索引建立在city字段上。Record format中的Begin和End用于界定该版本的数据的有效时间。
- Hash bucket J包含4条记录：关于John的3个版本的记录和关于Jane的1个版本的记录。Jane的记录的有效时间是从15到无穷大，15表明这条记录是由一个在15这个时刻commit的事务创建的，无穷大表明这条记录现在依然有效。

### 读
每一个读操作都会指定一个read time，只有当某个版本的数据的有效时间和read time重叠时，该版本的数据才对当前的读可见。

### 更新
transaction 75正在从larry的账户向john转账20元。那么他会为larry创建一个新的版本的记录(larry, rome, 150)，同时为john创建一个新的版本的记录(john, london, 130)，然后将这2条记录分别添加到对应的索引中。

对于larry和john的上一版本的记录，记录中的End字段会被设置为transaction 75的transaction id，而对于larry和john的新版本的记录，记录中的Begin字段会被设置为transaction 75的transaction id。假设transaction 75在时刻100的时候commit了，那么transaction 75会将larry和john上一版本的记录中的End字段设置为100，而将larry和john新版本的记录中的Begin字段设置为100。

## 事务管理
Hekaton采用乐观的并发控制机制来提供snapshot，repeatable read和serializable隔离级别，而无需锁。

下面的讨论以serializable为例进行。

### 时间戳和版本可见性
- Logical Read Time：事务的read time可以是事务开始时间和当前时间之间的任意一个时间戳。
- Commit/End Time
- Valid Time：所有的记录都包含2个时间戳，Begin和End。Begin表示创建该记录的事务的commit时间戳，End表示删除或者替换该记录的事务的commit时间戳。

关于版本可见性，logical read time为RT的事务只能看到满足如下要求的版本的记录：
- 记录的Begin时间戳小于RT且记录的End时间戳大于RT

### 事务commit
#### 验证和依赖
在事务commit的时候，一个serializable的事务必须验证它所读取的版本的记录没有被更新且没有幻读发生。在验证阶段，事务会先获取一个end时间戳。在验证它所读取的版本的记录没有被更新的时候，会检查它所读取的记录的版本号是否依然对它可见的。在验证没有幻读发生的时候，会再次扫描索引，找出事务开始以后所有可见的版本的记录。

在事务T2验证过程中开始的其它任何事务T1，如果它尝试读取T2创建的记录版本或者忽略T2删除的记录版本，则认为事务T1依赖于事务T2。在这种情况下，T1有2种选择：T1阻塞直到T2 commit或者abort，或者T1继续执行，但是记录它和T2之间存在commit dependency。目前Hekaton采用的是后者，也就是说只有T2提交了的情况下，T1才能提交，如果T2 abort，那么T1也必须abort，因此可能存在级联abort。

#### Commit Logging和Post-processing
一旦事务T对数据库的更新都持久化到transaction log中，就认为事务T成功提交了。事务T会在transaction log中写入它所创建的新版本的记录，和它删除的所有的记录的primary key。

一旦事务T提交成功，它就开始Post-processing阶段，在该阶段，它会将它所涉及的所有旧版本记录中的End字段和所有新版本记录中的Begin字段更新为它的commit timestamp。

## 事务durability
Hekaton采用transaction log和checkpoint来保证durability。

Hekaton中索引更新不会写log，在恢复的时候进行重建。

### Transaction logging
- 每个事务记录一个日志记录，而不是每个operation记录一个日志记录
- 只在transaction commit的时候生成日志记录
    - Hekaton没有使用WAL
- Hekaton支持group commit
    - 多条日志记录组成一个大的IO
- Hekaton支持多个并发的log stream

### Checkpoints
#### Checkpoint Files
checkpoint数据保存在2种checkpoint files中：data files和delta files，一个完整的checkpoint包含多个data files，多个delta files以及一个用于记录该checkpoint中包含哪些files的checkpoint file inventory。

data file中只包含特定时间范围内插入的记录的版本，如果某个版本的记录的Begin字段所代表的的时间戳在该data file的特定时间范围内，则该版本的记录就被包含在data file中

delta file中记录了某个data file中哪些版本的数据被删除了，delta file和data file之间是1:1的关系。

checkpoint file inventory中记录组成一个完整的checkpoint的所有的data和delta files。checkpoint file inventory保存在system table中。

#### Checkpoint process
一个checkpoint任务会将transaction log中的一部分内容转换为data files和delta files。

Hekaton支持对相邻的data files进行合并，这只在当这些data file中未删除的记录所占的比例达到某个阈值时才会发生。

## Garbage Collection
Hekaton根据记录的可见性来决定它是否是garbage，如果一条记录不再对任何活跃的事务可见，则该记录就是garbage。

Hekaton的garbage collection具有以下特点：
- non-blocking：和事务并发运行
- cooperative：运行事务的工作线程在遇到garbage的时候可以自己删除它
- incremental：garbage collection过程可以被限流，可以被停止或者重新启动，以避免它过度消耗CPU资源
- parallilizable and scalable

### Garbage Collection Details
#### GC Correctness
以下情况会导致记录成为garbage：
- 成功提交的事务执行了delete或者update，从而删除了记录且某个版本的记录已经不可能被读取
- 某个事务创建记录之后发生了回滚

## 参考
[论文阅读 - Hekaton内存引擎](http://loopjump.com/pr-hekaton/)
[论文阅读 - Haketon的高性能并发控制算法](http://loopjump.com/pr-haketon-high-perf-concurrency-control/)