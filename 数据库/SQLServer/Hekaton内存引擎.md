# 提纲
[toc]

# 1. Hekaton简介
Hekaton是MS-SQLServer专门为基于内存的OLTP工作负载优化的存储引擎。该内存引擎采用无锁数据结构和乐观的多版本并发控制技术，从而实现了非常高的并发和10倍以上的性能提升。

# 2. 为什么要重新设计Hekaton内存引擎
MS-SQLServer团队分析发现，仅仅优化现有的SQL Server无法实现10-100倍的性能提升。而提升性能主要有3种方式：提升可扩展性、提高CPI(即每条指令所需要的CPU时钟)、减少每个请求所执行的指令数目。分析还发现，即使是最优的情况下，提升可扩展性和提高CPI都只能带来3-4倍的性能提升。所以希望寄托在减少每个请求所执行的指令数目上，如果想要10倍的性能提升，则必须减少90%的指令，如果想要100倍的性能提升，则必须减少99%的指令，直接在现有的存储引擎上去优化是做不到的。

# 3. 设计原则
## 3.1 为内存优化索引
现在主流的数据库系统都是面向硬盘的存储结构，记录都按page保存在硬盘上，在需要的时候加载到内存中。这需要维护复杂的buffer pool，在访问一个page的时候必须加latch锁保护。即使所有的pages都在内存中，在B-tree索引中执行一个简单的查询也需要数千条指令。

所以Hekaton重新设计实现了常驻内存的数据的索引，索引相关的操作则不记录到日志，通过日志和检查点机制来保证数据的durability。在恢复的时候，Hekaton通过最新的检查点和日志来重建表和索引。

## 3.2 消除latches和locks
数十甚至上百核心的CPU逐渐流行，因此scalability非常关键。如果系统中有频繁更新的共享内存和高竞争的资源，比如latches，spinlocks，lock manager，transaction log tail，last page of B-tree，可扩展性将受到严重影响。

Hekaton的所有的关键数据结构，比如内存分配器，hash/range索引，transaction map等都是latch-free(lock-free)的。在性能关键的路径上也没有任何latches或者spinlocks。

Hekaton使用新的乐观的多版本并发控制来提供隔离语义，没有locks，也没有lock table。借助于乐观的并发控制，多版本和latch-free数据结构，Hekaton线程不会stall或者wait。

## 3.3 将请求编译为机器码(machine code/native code)
解释执行是RDBMS的标准且通用的做法，解释执行的优点是灵活，缺点是执行慢。一个简单的查询语句，如果是使用解释执行的话，大概要执行10万量级的指令。

Hekaton解决方法是，对于T-SQL编写的语句和存储过程，Hekaton会将其编译成高度优化的机器码，编译会花一些时间，但执行期间效率很高。

## 3.4 无分区
HStore/VoltDB这类内存数据库会将数据按照CPU core进行分区，但是经过认真评估分区方式之后，Hekaton最后拒绝了它。因为在应用可以分区的情况下，分区的确很好，但如果应用分区做的不好，平均下来每个事务都需要访问多个分区的话，性能退化非常严重。因此，Hekaton不对数据库进行分区，任何线程都可以访问该数据库的任何一部分。

# 4. 架构
Hekaton主要由3部分组成：
- Hekaton storage engine
    - 管理用户数据和索引
    - 支持事务操作
    - 支持hash和range索引
    - 支持检查点，数据恢复和高可用
- Hekaton compiler
    - 负责将请求编译为机器码
- Hekaton runtime system
    - 与SQL Server的资源结合，作为编译执行时所依赖功能的库
    
Hekaton和SQL Server主要在以下几个方面进行集成：
- 元数据
    - Hekaton表和索引等的元数据存储在SQL Server catalog中
- 查询优化
- 查询算子
    - Hekaton提供一些访问Hekaton表中数据的operator，可被SQL Server的执行计划使用。另外，还提供了insert，delete和update相关的operator
- 事务
    - SQL Server事务可以访问或者更新SQL Server表或者Hekaton表中的数据
- 高可用
- 检查点和日志
    - Hekaton日志被更新到SQL Server的transaction log中
    - Hekaton使用SQL Server的file stream来实现检查点

# 5. 存储和索引
## 5.1 存储
Hekaton按行存储，每一行数据存在多个版本，每个版本的数据的布局如下：
![Row Layout](https://note.youdao.com/yws/public/resource/c84f877986d06d4747a7e9dd91a1f4ab/xmlnote/7CC084B0F6EA4C38918FA20DF43C6059/112608)

每一行数据主要包括2部分：Row Header和Payload。Row Header中主要保存该版本数据的元数据，Payload中则存储的是这一行的用户数据。

其中RowHeader包括以下字段：
- BeginTs：表示创建该记录的事务的commit时间戳，可能值为当前在该记录的最新版本上活跃的事务的BeginTs，或者创建该版本的事务的CommitTs
    > 关于事务的BeginTs和CommitTs的说明：Hekaton中每个事务都包含2个时间戳：BeginTs和CommitTs，分别表示事务开始的时间戳和事务commit的时间戳
- EndTs：表示删除或者替换该记录的事务的commit时间戳，可能值为当前在该记录的下一个版本上活跃的事务的BeginTs，或者无穷大，或者当前在该记录的下一个版本上活跃的事务的CommitTs
    > 如果EndTs是无穷大，则表明该版本的记录依然有效
- StmtId：表示事务中创建该版本的记录的语句的ID
- IdxLinkCount：表示索引数目
- Padding：用于字节对齐
- Pointers：用于索引管理，每个索引占用一个8 Bytes的指针，有多少个索引，就有多少个8 Bytes的指针

## 5.2 索引
Hekaton支持两种索引类型：hash索引和range索引。

hash索引通过lock-free的hash table实现，range索引则通过无锁的[Bw-tree](https://www.cnblogs.com/Amaranthus/p/4375331.html)实现。

一个表可以有多个索引，记录总是通过索引来访问。Hekaton使用多版本，更新总是创建一个新的版本。

## 5.3 示例
![image](https://note.youdao.com/yws/public/resource/c84f877986d06d4747a7e9dd91a1f4ab/xmlnote/21C875F0A5134CB98B0CB728BABD6FA3/112958)

# 6. 事务管理

## 6.1 TransactionID和时间戳
为了支持多版本，Hekaton内部维护了两个计数器：
- TransactionID：TransactionID是全局递增的ID，用于唯一标识一个活跃的事务，每当启动一个新的事务的时候，都会为之分配一个TransactionID，在Hekaton重启的时候会重置
- Global Transaction Timestamp：是一个全局递增的时间戳，每当事务commit的时候，会为之分配一个commit timestamp，在Hekaton重启的时候它不会重置

## 6.2 版本可见性
Logical Read Time：事务的read time是事务开始时间(如果是read committed隔离级别，则对应的是语句执行的时间)。

Valid Time：所有的版本都包含2个时间戳，BeginTs和EndTs，BeginTs表示创建该记录的事务的commit时间戳，EndTs表示删除或者替换该记录的事务的commit时间戳，BeginTs和EndTs就界定了该版本的Valid Time。

Hekaton每条记录中的BeginTs和EndTs这2个时间戳，决定了记录的可见性。logical read time为RT的事务，只能看到BeginTs不大于RT且EndTs大于RT的记录。

![image](https://note.youdao.com/yws/public/resource/c84f877986d06d4747a7e9dd91a1f4ab/xmlnote/002B3E997EF045F28BC3488AB936D227/112967)

## 6.3 基于MVCC的乐观并发控制
在Hekaton的乐观并发控制中事务会经历3个阶段：Processing阶段，Validation阶段和Post-processing阶段。

下面我们以一个例子来讲述Hekaton在这3个阶段都分别做了些什么。

假设在事务开始之前，Hekaton中包含如下2条记录：

BeginTs | EndTs | ... | Name | City 
-|-|-|-|-|
20 | Infinite | ... | Susan | Bogota
20 | Infinite | ... | Greg | Beijiing 

现在用户发起一个事务，TransactionId为Tx1，事务开始时间为90，事务执行如下操作：
- delete <Susan, Bogota>
- update <Greg, Beijing> to <Greg, Lisbon>
- insert <Jane, Helsinki>

下面分析在事务的不同阶段，这些记录的版本变迁，以及不同版本的数据对于并发事务的可见性。

### 6.3.1 Processing阶段
对于事务Tx1中的delete语句，Hekaton会通过索引找到Susan对应的记录，并且将记录中的EndTs设置为Tx1。

对于事务Tx1中的update语句，Hekaton会创建一个新的记录<Greg, Lisbon>，将记录中的BeginTs设置为Tx1，将记录中的EndTs设置为infinite，然后通过索引找到Greg对应的记录，并且将记录中的EndTs设置为Tx1。

对于事务Tx1中的insert语句，Hekaton会创建一个新的记录<Jane, Helsinki>，将记录中的BeginTs设置为Tx1，将记录中的EndTs设置为infinite。

Hekaton会通过一个特定标记位来区分BeginTs和EndTs中记录的是TransactionId还是一个时间戳。

BeginTs | EndTs | ... | Name | City 
-|-|-|-|-|
20 | Tx1 | ... | Susan | Bogota
20 | Tx1 | ... | Greg | Beijiing 
Tx1 | infinite | ... | Greg | Lisbon 
Tx1 | infinite | ... | Jane | Helsinki

假设现在接收到关于Tx1的commit请求，Hekaton会为Tx1分配一个commit时间戳，假设是120，然后将Tx1的commit时间戳保存到全局的事务表中。注意，此时虽然接收到了commit请求，但是事务Tx1仍然可能abort，Hekaton也不能向用户发送commit响应。但是，一旦接收到事务Tx1的commit请求，Hekaton会乐观的认为事务Tx1最终会commit成功，所以会让事务Tx1中的所有更新对其它事务可见。

#### 6.3.1.1 读写冲突处理
假设另外一个事务Tx2，事务开始时间戳为100，此时事务Tx1已经开始了，但是事务Tx1还没有接收到commit请求。对各记录的处理如下：
- 当事务Tx2读取到<Susan, Bogota>记录时，发现它的EndTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1仍然处于active状态，<Susan, Bogota>记录尚未完成删除，所以事务Tx2可以访问<Susan, Bogota>记录。

- 当事务Tx2读取到<Greg, Beijiing>记录时，发现它的EndTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1仍然处于active状态，<Greg, Beijiing>记录尚未完成删除，所以事务Tx2可以访问<Greg, Beijiing>记录。

- 当事务Tx2读取到<Greg, Lisbon>记录时，发现它的BeginTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1仍然处于active状态，<Greg, Lisbon>记录尚未完成更新，所以事务Tx2不可以访问<Greg, Lisbon>记录，而只能访问前一个版本<Greg, Beijing>记录。

- 当事务Tx2读取到<Jane, Helsinki>记录时，发现它的BeginTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1仍然处于active状态，<Jane, Helsinki>记录尚未完成更新，所以事务Tx2不可以访问<Jane, Helsinki>记录。

假设事务Tx2的事务开始时间戳为121，此时事务Tx1已经接收到了commit请求，但是事务Tx1还没有完成Validation阶段的工作。此时，对各记录的处理如下：
- 当事务Tx2读取到<Susan, Bogota>记录时，发现它的EndTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1已经分配了120这个commit时间戳，Hekaton会乐观的认为事务Tx1最终会commit成功，认为<Susan, Bogota>记录被删除了，所以事务Tx2不可以访问<Susan, Bogota>记录。

- 当事务Tx2读取到<Greg, Beijing>记录时，发现它的EndTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1已经分配了120这个commit时间戳，Hekaton会乐观的认为事务Tx1最终会commit成功，认为<Susan, Bogota>记录被删除了，所以事务Tx2不可以访问<Greg, Beijing>记录。

- 当事务Tx2读取到<Greg, Lisbon>记录时，发现它的BeginTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1已经分配了120这个commit时间戳，Hekaton会乐观的认为事务Tx1最终会commit成功，进而认为<Greg, Lisbon>记录已经完成了更新，所以事务Tx2可以访问<Greg,Lisbon>记录。

- 当事务Tx2读取到<Jane, Helsinki>记录时，发现它的BeginTs中记录的是TransactionId，事务Tx2会到全局的事务表中查找事务Tx1的状态信息，发现Tx1已经分配了120这个commit时间戳，Hekaton会乐观的认为事务Tx1最终会commit成功，进而认为<Jane, Helsinki>记录已经完成了插入，所以事务Tx2可以访问<Jane, Helsinki>记录。

- 因为Hekaton只是乐观的认为事务Tx1最终会commit成功，而事务Tx1最终是否commit成功，还是未知的，所以Hekaton会将事务Tx2注册到事务Tx1的commit-dependency列表中(称Tx1是Tx2依赖提交的事务)，表示直到事务Tx1完成Validation阶段之后，事务Tx2才可以完成Validation阶段。

#### 6.3.1.2 写写冲突处理
假设事务Tx2的事务开始时间是100，它不仅会读取事务Tx1修改的记录，还会将记录<Greg, Beijing>更新为<Greg, Porto>，那么在事务Tx2执行更新操作的时候，发现记录<Greg, Beijing>的EndTs中记录的是TransactionId，而Hekaton会乐观的认为事务Tx1会coomit成功，所以Hekaton会立即abort事务Tx2。

### 6.3.2 Validation阶段
一旦事务Tx1接收到commit请求，并且Hekaton为事务Tx1分配了commit时间戳，事务Tx1就将进入Validation阶段。

在Validation阶段主要执行以下逻辑：
- 检查是否破坏了隔离级别的要求（我们将在后面着重讲解，见“检查是否破坏了隔离级别的要求”）
- 等待事务Tx1所依赖提交的事务数目降为0
    - 在处理并发事务的读写冲突过程中，可能会将事务Tx1注册到其它事务Txm的commit-dependency列表中，我们称Tx1依赖于Txm，Tx1必须在它所依赖的所有的事务都已经完成了Validation阶段之后，才能完成Validation阶段
    - 实现上，Hekaton每当将事务Tx1注册到其它事务Txm的commit-dependency列表中的时候，会对Tx1所依赖的事务的数目加1，当事务Txm完成Validation阶段之后，它会将它的commit-dependency列表中的每个事务所依赖的事务的数目减1
- 写持久化的事务日志
    - 只在创建表的时候指定了SCHEMA_AND_DATA参数的情况下，才会将这些日志持久化
    - 事务Tx1会为它的所有修改维护一个write-set，其中存放的是一系列的insert或者delete操作，每个操作指向某个版本的记录
        ![image](https://note.youdao.com/yws/public/resource/c84f877986d06d4747a7e9dd91a1f4ab/xmlnote/07FA8A0759F54896A64512581F08F40C/112964)
    - 事务日志中包含：TransactionId，commit时间戳和该事务insert或者delete的所有的记录
- 更新全局的事务表，标记事务状态为committed
- 减少所有依赖于Tx1提交的事务所依赖提交的事务的数目
    - 遍历Tx1的commit-dependency列表中的每个事务，将该事务所依赖的其它事务的数目减1

#### 6.3.2.1 检查是否破坏了隔离级别的要求
Hekaton通过乐观的多版本并发控制(MVCC)，无需锁就可以支持snapshot，repeatable read和serializable隔离级别。

如果事务的读和写逻辑上发生在同一时刻，则事务就是可串行化的。SI(Snapshot Isolation)隔离级别并不满足这个情况，SI的读实际上是发生在事务开启时（事务开启时取快照，快照一旦确定了，即使读请求是过一段时间之后才发起的，仍然等价于是去快照后立刻执行的），写实际上发生在事务提交时（事务提交前，所有写操作都不生效）。

如果在事务结束时我们能够确认该事务之前发起的读操作，再次发起时返回相同的结果，那么等价于读操作和写操作是同时发生在事务结束这个时间点上，因此事务就是可串行化的。

基于上述观察，如果要实现Serializable隔离级别，则如下两个条件都要满足：
- 可重复读(Read Stability)：如果T读取行的某个版本V1，那么V1直到事务结束都应该继续有效而不是被V2更新了。
    - 可以通过加读锁或者在事务结束前做validation来达到这一效果。
- 避免幻读(Phantom Avoidance)：事务的scan不会返回额外的新版本。
    - 可以通过加范围锁或者validation来实现。
 
相比较于Serializable隔离级别，如果要实现其它更低的隔离级别，就容易的多：
- Read Committed隔离级别：总是读取已提交的最新数据，不需要加锁，也不需要valdation。
- Repeatable Read隔离级别：只需要保证可重复读，而不解决幻读问题。
- SI(Snapshot Isolation)隔离级别：总是使用事务开始时的快照，不需要加锁，也不需要validation。

在事务Processing阶段，会维护read-set，write-set和scan-set。read-set中保存该事务所读取的所有记录的指针，write-set中保存该事务所更新的所有记录的指针，scan-set中则保存谓词所涉及的所有的记录的相关信息。read-set用于检查是否可重复读，write-set用于记录事务日志，scan-set则用于检查是否发生了幻读。

当然，对于某些隔离级别来说，read-set或scan-set是不需要的，如下：

隔离级别 | read-set | scan-set
-|-|-|
snapshot | No | No
repeatable read | Yes | No
serializable | Yes | Yes

##### 6.3.2.2 破坏snapshot隔离级别要求
在snapshot隔离级别下，需要检查是否存在如下的破坏snapshot隔离级别要求的情形：
- 事务Tx1开始，并且插入一个新的行
- 事务Tx2开始，并且插入一个具有相同主键的行
- 事务Tx2提交
- 事务Tx1提交
    - 事务Tx1在Validation阶段，发现两行记录具有相同的主键，abort

##### 6.3.2.3 破坏repeatable read隔离级别要求
在repeatable read隔离级别下，需要检查是否存在类似于“破坏snapshot隔离级别要求”的情形。

在repeatable read隔离级别下，还需要检查事务的read-set中的记录是否具有更新的版本(也就是说，其它事务在当前事务commit之前更新了该记录)，如果是，则当前事务必须abort。

##### 6.3.2.4 破坏serializable隔离级别要求的情况
在serializable隔离级别下，除了需要检查是否存在类似于“破坏repeatable read隔离级别要求”的情形之外，还需要检查事务的scan-set中的记录是否存在下面2种情况：
- scan-set中的某个记录现在读取不到了，表明该记录被其它事务删除了
- 某记录不在scan-set中，但是它满足scan-set对应的谓词，表明其它事务插入了新的记录

如果发生上述情况中的任意一种，则当前事务必须abort。

### 6.3.3 Post-processing阶段
该阶段比较简单，将当前事务所涉及的所有旧版本记录中的EndTs字段更新为它的commit时间戳，同时将所有新版本记录中的BeginTs字段更新为它的commit时间戳。

如果事务失败或者显式回滚，则新版本的记录将被标记为garbage，同时旧版本的记录中的EndTs将被修改为infinite。被标记为garbage的记录将由garbage collector负责回收。

### 6.3.4 事务durability
Hekaton采用事务日志(transaction log)和checkpoint来保证记录的durability。

Hekaton中索引更新不会写log，而是在恢复的时候进行重建。

transaction log, checkpoint和recovery组件的设计原则：
- 顺序访问IO
- 主流程尽量减少操作，能移到Recovery阶段的操作都移过去
- 消除扩展性瓶颈
- 在Recovery阶段要做IO和CPU的并发

#### 6.3.4.1 事务日志
Hekaton事务日志的要点：
- 每个事务记录一个日志记录，而不是事务中每个操作记录一个日志记录
- 只在transaction commit的时候生成日志记录
    - 没有使用WAL
- 索引更新不记录日志
- 只支持Redo日志，不支持Undo日志
- 支持group commit
    - 多条日志记录组成一个大的IO
- 支持多个并发的log stream，减少log tail竞争
    - 因为事务顺序由事务的commit时间戳决定，而不是由日志记录的顺序决定

### 6.3.5 Checkpoint
执行checkpoint的目的是为了减少恢复时间。

Checkpoint的设计主要围绕两个需求：
- 持续的checkpoint：采用持续的增量的checkpoint，避免突发的checkpoint
- 流式IO：尽量保证顺序流式IO

Checkpoint触发时机：
- 手动触发：用户显式执行checkpoint命令
- 自动触发：当自从上次checkpoint以来，日志记录容量增加超过1.5GB的时候，就会执行checkpoint

#### 6.3.5.1 Checkpoint文件
checkpoint数据保存在2种checkpoint files中：data files和delta files，一个完整的checkpoint包含多个data files，多个delta files以及一个用于记录该checkpoint中包含哪些files的checkpoint file inventory。

data file中只包含特定时间范围内插入的记录的版本，如果某个版本的记录的BeginTs字段所代表的时间戳在该data file的特定时间范围内，则该版本的记录就被包含在data file中

delta file中记录了某个data file中哪些版本的数据被删除了，delta file和data file之间是1:1的关系。

checkpoint file inventory中记录组成一个完整的checkpoint的所有的data和delta files。checkpoint file inventory保存在system table中。

#### 6.3.5.2 Checkpoint过程
一个checkpoint任务会将transaction log中的一部分内容转换为data files和delta files，并更新inventory。

持续一段时间之后，checkpoint文件就会逐渐变多，导致recovery时间变长。为此，Hekaton支持当data file中未删除的记录所占的比例达到某个阈值时对相邻的data files进行合并。

# 7. 垃圾回收
Hekaton根据记录的可见性来决定它是否是garbage，如果一条记录不再对任何活跃的事务可见，则该记录就是garbage。GC线程周期地扫描全局活跃事务map，得到最老的活跃事务的BeginTs作为gc-timestamp。EndTs小于gc-timestamp的事务产生的旧版本都可以回收。

Hekaton的garbage collection具有以下特点：
- 非阻塞：和事务并发运行
- 协作执行：运行事务的工作线程在遇到garbage的时候可以自己删除它
- 增量运行：garbage collection过程可以被限流，可以被停止或者重新启动，以避免它过度消耗CPU资源
- 可并行化扩展

# 8. 参考
Hekaton: SQL Server’s Memory-Optimized OLTP Engine

SQL Server Internals: In-Memory OLTP Inside the SQL Server 2016 Hekaton Engine