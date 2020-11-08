# 提纲
[toc]

# 简介
WAL(Write Ahead Logging)是业界通用的保证数据durability的方法，它的基本思想是：数据更新(表数据或者索引数据等的更新)只能在这些更新被写入日志文件并且flush之后才能完成。openGauss内存引擎MOT(Memory Optimized Table)也不例外。本文将从以下几个方面分析MOT WAL的实现：
- WAL整体架构
- 事务操作记录
- RedoLogBuffer
- RedoLogHandler
- XLog

# 整体架构
MOT WAL的整体架构如下：
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/877ECB64C5904A198B5BC5B1B4F234BC/113467)

事务记录WAL日志的过程如下：
- 组装生成事务相关的操作记录
- 将事务相关的操作记录写入基于内存的RedoLogBuffer
- 当RedoLogBuffer写满或者事务没有更多的日志记录要写入RedoLogBuffer的时候，就会将该RedoLogBuffer提交给RedoLogHandler
- RedoLogHandler将日志记录提交给XLog
    - 在XLog中先写WAL Log Buffer
    - 在合适时机，将WAL Log Buffer中的数据写入WAL segment文件

# 事务操作记录
MOT写入WAL的是redo log，其格式如下：
![Redo Log Layout](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/D965779FB4D841278F6484BDFDD98E96/113252)

redo log主要由3部分组成：
- DDL操作日志区：当前事务中所有DDL相关的操作(如CreateTable，DropTable，CreateIndex，DropIndex)的日志记录保存在这里
- DML操作日志区：当前事务中所有DDL相关的操作(如Insert，update，delete)的日志记录保存在这里
- EndBlock：标记当前事务日志记录部分结束或者全部结束

对于不同的DDL操作(如CreateTable，DropTable，CreateIndex，DropIndex)或者DML操作(如Insert，update，delete)的操作记录，其格式不尽相同，下面会逐一讲述。

EndBlock包括2种，分别是用于标记当前事务日志记录部分结束的PartialRedoTx记录和用于标记当前事务日志记录全部结束的CommitTx记录。

## DDL操作日志记录
### CreateTable日志记录
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/C197E25F296049CFBEFED5CE025A8297/115874)

### DropTable日志记录
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/C4C675D094534ECE822E5D9D3F8F82F5/115877)

### CreateIndex日志记录
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/49AF35E527CA43898FE6DD52211BD6A3/115879)

### DropIndex日志记录
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/8D6EAFA4994448438FAECBD17C4722D4/115882)

## DML操作日志记录
### Insert日志记录
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/7A6840C0364348FC886B7663E520B96F/115885)

> 需要指出的是，insert操作会更新索引，但是MOT中索引是不持久化的，所以也不需要记录关于索引的WAL日志，索引会在recovery过程中重建

### Update日志记录
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/BAC6CC867B5D44C5AF84B7F08012918A/115888)

### Delete日志记录布局
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/B5744272ECC5480EB332E5B2B9007B4C/115891)

> 需要指出的是，insert操作会更新索引，但是MOT中索引是不持久化的，所以也不需要记录关于索引的WAL日志，索引会在recovery过程中重建

## EndBlock类型的日志记录
### PartialRedoTx记录
事务写日志记录的过程中，如果RedoLogBuffer的空间不足以容纳当前DDL操作或者DML操作的日志记录，MOT会在当前RedoLogBuffer的尾部写入一个PartialRedoTx记录，表示当前事务的日志记录还没有写完。PartialRedoTx记录格式如下：
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/DB0854E278AC4CCDA714603E81C43B49/115894)

### CommitTx记录
CommitTx记录用于标记当前事务日志记录已经全部写完。CommitTx记录格式如下：
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/77A88ECC35A9490C856693E8B9AF1725/115897)


# RedoLogBuffer
一个事务中可能包含多个操作，每个操作都有对应的操作记录，这些操作记录会先保存在RedoLogBuffer中一并提交。RedoLogBuffer实际上是一个字节数组，其格式如下：
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/D2E6BABFF50F4A9C9E7CBB9EDB9B3F17/115900)

每个事务都有自己单独的RedoLogBuffer，且任意时刻每个事务都只有一个可供写入的RedoLogBuffer。在写事务日志记录到RedoLogBuffer的过程中，如果事务的RedoLogBuffer无法容纳当前DDL操作或者DML操作的记录，则会执行以下步骤：
- 在当前RedoLogBuffer的尾部写入一个PartialRedoTx记录
- 将该RedoLogBuffer提交给RedoLogHandler
- 重新获取一个可用的RedoLogBuffer，这个RedoLogBuffer可能是之前的那个，也可能是重新分配的一个，具体取决于该RedoLogBuffer中的数据是否已经成功写入到XLOG中，如果已经成功写入XLOG，就可以复用之前的那个RedoLogBuffer，否则必须重新分配一个新的RedoLogBuffer
- 继续向RedoLogBuffer中写入当前事务的其它操作的日志记录

# RedoLogHandler
MOT提供了3种RedoLogHandler，分别是：
- 同步模式的RedoLogHandler(SynchronousRedoLogHandler)
- 异步模式的RedoLogHandler(AsyncRedoLogHandler)
- NUMA感知的组提交模式的RedoLogHandler(SegmentedGroupSyncRedoLogHandler)

## 同步模式的RedoLogHandler
在该模式下，对接收到的RedoLogBuffer直接提交给XLog做进一步处理。事务所在的线程被阻塞，直到事务日志记录写入XLog为止。

## 异步模式的RedoLogHandler
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/47CAAF0AC8BD4AF5836B7452073F5B76/113773)

AsynchronousRedoLogHandler由一个RedoLogBuffer Pool和一个包括3个元素的RedoLogBufferArray数组组成。

RedoLogBuffer Pool作为RedoLogBuffer的缓存池，RedoLogHandler中用到的RedoLogBuffer都从RedoLogBuffer Pool中分配，不再使用的RedoLogBuffer也会释放到RedoLogBuffer Pool中。

每个RedoLogBufferArray都是一个关于RedoLogBuffer的数组，其中最多存放1000个RedoLogBuffer。AsynchronousRedoLogHandler轮流使用这3个RedoLogBufferArray来存放接收到的RedoLogBuffer，每个RedoLogBufferArray使用一段时间之后，切换到下一个RedoLogBufferArray，在发生RedoLogBufferArray切换之前，所有接收到的RedoLogBuffer都会被添加到当前的RedoLogBufferArray中。为了指示当前正在使用哪个RedoLogBufferArray，AsynchronousRedoLogHandler特地维护了一个ActiveArrayIdx索引。

### 向RedoLogBufferArray中添加接收到的RedoLogBuffer
步骤如下：
- 获取ActiveArrayIdx，表示当前的RedoLogBuffer将被添加到哪个RedoLogBufferArray
- 将接收到的RedoLogBuffer添加到ActiveArrayIdx所指示的RedoLogBufferArray的尾部
- 根据ActiveArrayIdx所指示的RedoLogBufferArray中保存的RedoLogBuffer数目，进行相应的处理：
    - 如果数目超过RedoLogBufferArray容量的一半，则唤醒WALWriter，通知其将RedoLogBufferArray中的日志记录flush到XLog中
    - 如果ActiveArrayIdx所指示的RedoLogBufferArray已经填满，当前的RedoLogBuffer无法添加进去，则睡眠等待10000us之后重试
- 如果当前RedoLogBuffer成功添加到RedoLogBufferArray中，则从RedoLogBuffer Pool中分配一个新的RedoLogBuffer，供后续使用

### 将RedoLogBufferArray中的RedoLogBuffer写入XLog
对于AsynchronousRedoLogHandler来说，接收到的RedoLogBuffer都保存在RedoLogBufferArray中，在WALWriter执行的时候，才将RedoLogBufferArray中的所有的RedoLogBuffer写入XLog，写入流程如下：
- 获取ActiveArrayIdx，表示当前正在使用哪个RedoLogBufferArray来存放接收到的RedoLogBuffer
- 计算nextArrayIdx = ((ActiveArrayIdx + 1) % 3)，计算prevArrayIdx = ((ActiveArrayIdx + 2) % 3)
- 设置新的ActiveArrayIdx = nextArrayIdx，接下来的RedoLogBuffer都添加到新的nextArrayIdx所指示的RedoLogBufferArray中
- 将prevArrayIdx所指示的RedoLogBufferArray中所有的RedoLogBuffer写入XLog中
- 释放prevArrayIdx所指示的RedoLogBufferArray中所有的RedoLogBuffer到RedoLogBufferPool中


## NUMA感知的组提交模式的RedoLogHandler
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/278DEE5A34EE4467A156E5C44130DC56/113770)

SegmentedGroupSyncRedoLogHandler按照NUMA node进行分组，绑定在相同NUMA node上的线程共享同一个GroupSyncRedoHandler，接收到的RedoLogBuffer会添加到相应的GroupSyncRedoHandler的CommitGroup中，CommitGroup包含一个RedoLogBuffer指针数组，添加到CommitGroup中的RedoLogBuffer的指针就保存在这个数组中，当一定条件满足时，CommitGroup中所有的RedoLogBuffer会被写入XLog。

一个GroupSyncRedoHandler中可能同时存在0个或者多个CommitGroup，但是任意时刻最多只有一个活跃的(与之对应的是关闭的，表示不再接收更多的RedoLogBuffer)CommitGroup，活跃的CommitGroup表示可以继续向其中添加RedoLogBuffer。

向GroupSyncRedoHandler中添加RedoLogBuffer的流程如下：
- 检查当前是否存在活跃的CommitGroup
    - 如果不存在，则生成一个新的CommitGroup，并设置它为活跃的CommitGroup
- 将RedoLogBuffer添加到活跃的CommitGroup中
- 如果当前活跃的CommitGroup无法容纳RedoLogBuffer，则：
    - 将当前的CommitGroup关闭，并标记当前不存在活跃的CommitGroup
    - 从头开始，重新执行
- 如果添加成功，且RedoLogBuffer被添加在当前活跃的CommitGroup中的最后一个位置，则：
    - 将当前的CommitGroup关闭，并标记当前不存在活跃的CommitGroup
- 至此，已经将RedoLogBuffer成功添加到CommitGroup中，如果该RedoLogBuffer是该CommitGroup中的第一个元素，则将提交该RedoLogBuffer的线程作为当前活跃的CommitGroup的leader，否则将提交该RedoLogBuffer的线程作为当前活跃的CommitGroup的follower
- 根据提交该RedoLogBuffer的线程的角色是leader还是follower分别处理
    - 如果提交该RedoLogBuffer的线程是leader，则：
        - 等待，直到CommitGroup填充满或者超时(自从该CommitGroup确定了leader之后，到现在的时间是否超过了一定的阈值)
        - 当该CommitGroup填充满或者超时发生时，执行：
            - 如果该CommitGroup是当前活跃的CommitGroup，则标记当前不存在活跃的CommitGroup
            - 将该CommitGroup中所有的RedoLogBuffer写入XLog中
            - 唤醒该CommitGroup相关的所有的follower
    - 如果提交该RedoLogBuffer的线程是follower，则：
        - 检查当前的CommitGroup是否填充满，如果填充满了，则唤醒leader线程
        - 等待，直到leader线程唤醒之

# XLog
XLog是Transaction Log的简称，它接收来自于RedoLogHandler的RedoLogBuffer或者RedoLogBuffer数组。它会先将接收到的RedoLogBuffer或者RedoLogBuffer数组组装成WAL log record，然后将这些WAL log record写入WAL log buffer中，最后再将WAL log buffer中的数据写入WAL log file中。

## WAL log record
如果接收的是RedoLogBuffer数组，则会遍历RedoLogBuffer数组中的所有的元素，在每个RedoLogBuffer头部4字节中记录RedoLogBuffer大小，并且将RedoLogBuffer组装成WAL log record中。如果接收到的是一个RedoLogBuffer，则相当于接收到的是一个只包含一个元素的RedoLogBuffer数组，处理过程类似。

openGauss WAL log record格式如下：
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/07DB28DBD8074F2E9358CF5903F0A34B/113988)

WAL log record由以下几部分组成：
- 0...N个XLogRecordBlockHeader，每一个XLogRecordBlockHeader对应一个Blockdata
- 0或1个XLogRecordDataHeaderShort(或者XLogRecordDataHeaderLong)，对应0或1个Maindata，如果数据小于256 Bytes，则使用XLogRecordDataHeaderShort，否则使用XLogRecordDataHeaderLong
- Blockdata：full-write-page data或tuple data
- Maindata：日志数据

## WAL log file
逻辑上，WAL log file的大小是16EB(8 Byte地址空间)，但是系统中不可能有这么大的文件，因此openGauss会将WAL log file切分成16MB(可配置)的片段，这些片段被称为WAL log segment。

### WAL log segment布局
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/479733AF7E17403C815D106C03288C5B/114033)

WAL log segment会进一步划分为一系列8KB大小的page，每个page的起始位置保存的都是PageHeader，对于每个WAL log segment中的第一个page，其PageHeader类型是XLogLongPageHeaderData，其它page的PageHeader类型是pageXLogPageHeaderData。每个page中，紧接着PageHeader保存的是一系列的WAL log record。

### WAL log buffer
为了进一步的减少WAL log file的I/O操作，PostgreSQL中引入了WAL log buffer，对产生的WAL log record日志进行缓存，合并I/O操作。

WAL log buffer可以理解为一个环形的共享缓存，在每次写入新的日志记录时:
- 当WAL log buffer中有足够的空间，顺序写入到缓存区中
- 当WAL log buffer写到尾部且空间不足时，从头部刷出信息后重复利用，并将头部向后移动

WAL log buffer中的数据最终必须写入WAL log file中，这会在以下时机发生：
- WAL log buffer满
- WALWriter进程周期性工作
- 事务提交
- 创建Checkpoint

WAL log buffer写入WAL log file之后，可能在操作系统的page cache中缓存，并没有真正写入到文件中，所以还需要有flush机制，openGauss支持同步和异步两种方式：
- synchronous_commit(默认值为ON)为ON，则为同步方式，写入WAL log file之后会flush，然后才返回
- synchronous_commit为OFF，则为异步方式，写入WAL log file之后立即返回

