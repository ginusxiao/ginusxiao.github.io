# 提纲
[toc]

## 1: What's Special About In-Memory OLTP?
### The In-Memory OLTP Component
#### Memory-optimized tables
##### Entirely in-memory storage
table的所有数据包括索引在内都保存在内存中，借助于全新的乐观并发控制模型，并发操作不再需要lock或者latch了。

为了保证数据的durability，SQL Server还是会执行一些disk IO。

##### Row-based storage structure
对于disk-based tables：
- 为block-addressable disk storage优化
- SQL Server以8KB的page来组织数据

对于memory-optimized tables：
- 为byte-addressable memory storage优化 
- 没有page的概念，只有row
- 每个row都包含2部分：row header和payload

##### Native compilation of tables
没看懂

#### Natively compiled stored procedures
Natively compiled stored procedures由可以被CPU直接执行的处理器指令组成，执行的时候无需再编译或者翻译。

但是Natively compiled stored procedures也有局限性，它的语言结构是受限的(limitations on language constructs)，另外，它只能访问memory-optimized tables。

#### Concurrency improvements: the MVCC model
SQL Server的传统的悲观并发几只，在访问disk-based table的时候，使用锁来防止关于同一行数据上的并发事务之间的相互干扰。

SQL Server 2005借助snapshot-based isolation，一定程度上引入了乐观并发控制，读操作无需再加锁，但是写操作还是需要加锁。

SQL Server In-Memory OLTP则真正的引入了乐观并发控制，读写操作之间都不会阻塞彼此。

#### Indexes on memory-optimized tables
Memory-optimized table支持3种索引：hash index，range index和column-store index。其中，range index采用Bw-tree存储，column-store index允许用户高效的在memory-optimized table上执行分析。

SQL Server memory-optimized table的3种索引，除了column-store index以外，其它2种索引都只保存在内存中，不像传统的SQL Server会将索引信息也记录到log中，如果发生了重启，则在数据导入内存的过程中重建索引。

#### Data durability and recovery
对于memory-optimized table，SQL Server也会将操作记录到日志中，这些日志是保存在硬盘中的。

另外，对于memory-optimized table，SQL Server 还会持续的将内存中的table data持久化到checkpoint文件中。这些checkpoint文件也是append-only的。

通过使用SCHEMA_ONLY选项，In-memory OLTP可以创建non-durable的table，这种情况下，只有table schema是durable的。


## Chapter 3: Row Structure and Multi-Versioning
### Row Structure
![image](https://note.youdao.com/yws/public/resource/c84f877986d06d4747a7e9dd91a1f4ab/xmlnote/E65D42BF60AA4136A944B0EB7D5838B7/97097)

#### Row header
- Begin-Ts: 插入时间戳
- End-Ts: 删除时间戳
- StmtId: 
- IdxLinkCount: 引用这一行数据的索引计数
- Padding：用于对齐
- Index Pointers: table中的每一个非聚簇索引都对应一个index pointer，用于指向索引中的下一行数据。

#### Payload area
Payload area存放的就是row data。

### Row Versions, Timestamps and the MVCC Model
#### Transaction IDs and timestamps
The commit timestamps of existing row versions and the start timestamps of the transaction 
determine which row versions each active transaction can see.

#### Row versions and transaction processing phases
##### Processing phase
##### Validation phase
##### Post-processing
