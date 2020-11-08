# 提纲
[toc]

# WAL简介
WAL是write ahead logging的简称，它的基本思想是数据文件的修改必须发生在这些修改被记录到日志文件之后。WAL至少可以带来以下两方面的好处：
- 日志文件是顺序写入的，同步开销较小
- 针对日志文件的多次写入可以通过依次fsync()完成同步

# openGauss WAL写入流程
openGauss中WAL日志记录写入流程如下：
1. 申请WALInsertLock
2. 在WAL文件中预留写入位置
3. 拷贝WAL日志记录到WAL buffer中
4. 释放WALInsertLock

## WALInsertLock的申请和释放
openGauss中默认包含8个WALInsertLock，在写WAL之前必须先持有WALInsertLock，所以任意时刻至多有8个线程正在执行插入。WALInsertLock的定义如下：
```
typedef struct {
    LWLock lock;
    XLogRecPtr insertingAt;
} WALInsertLock;
```

WALInsertLock中的lock字段是PostgreSQL中实现的一种[轻量级锁  - LWLock](https://www.slideshare.net/jkshah/understanding-postgresql-lw-locks)，主要提供对共享内存变量的互斥访问，WALInsertLock中的insertingAt字段用于表示正在进行中的写入操作的执行进度。

LWLock支持独占(Exclusive)和共享(Shared)模式。独占模式下，只能有一个线程持有该锁，共享模式下，可以同时有一个或者多个线程持有该锁。

LWLock包含一个等待队列，用于保存那些因为申请该锁而被阻塞的线程，该等待队列是一个FIFO队列。如果一个线程释放它所持有的LWLock，且没有其它任何线程持有该LWLock的情况下，当前线程会唤醒等待队列中的背阻塞的线程：
- 如果等待队列头部的那个线程以独占模式申请锁，则只有该线程被唤醒，并且从等待队列中移除
- 如果等待队列头部的那个线程以共享模式申请锁，则会唤醒尽可能多的连续的以共享模式申请锁的线程，并且将这些线程从等待队列中移除

WALInsertLock只用到了独占模式。

## 在WAL文件中预留写入位置
逻辑上WAL文件是一个具有无限大空间的文件，物理上WAL文件会被划分为固定大小的WAL segment，每个WAL segment对应一个文件。

在WAL文件中预留写入位置，实际上是分配数据记录对应的LSN的过程。为了保证LSN顺序递增的特性，这个过程必须串行化。实现上，openGauss在内存中维护了一个CurrBytePos变量，用于记录逻辑上在WAL文件中最后一次成功预留的写入位置，假设当前在WAL文件中预留写入位置的WAL日志记录的大小为size，则采用原子的CAS操作将CurrBytePos更新为CurrBytePos + size即可。

需要特别指出的是，CurrBytePos是单调递增的，它不是在某个WAL segment文件中的偏移，而是将WAL文件想象成一个无限大的文件的情况下，在该文件中的偏移。

## WAL buffer
为了进一步的减少WAL文件的I/O操作，PostgreSQL中引入了WAL buffer。在将WAL日志记录写入WAL文件之前，会先写入WAL buffer中，然后再从WAL buffer写入WAL文件。WAL buffer可以看做是一个环形缓冲区，当整个WAL buffer被写满的时候，会从头开始填写。为了更好的管理， WAL buffer被进一步划分为一系列连续的WAL page buffer，每一个WAL page buffer都是一个







WAL的写入必须是顺序写入。openGauss是通过以下方式保证的：
1. 如果没有冲突，则可以并发写入，如t2先写，然后t1写
2. 如果有冲突，则被冲突的必须先写，如t3和t1冲突，t1必须先写
3. 从低地址向高地址写

2和3可以得出4：如果有冲突，则被冲突的必须先写，如t4和t2冲突，则t2和t1都必须先写

xlog write

xlog write不一定都发生在冲突的时候才触发，可能也有一些background的触发机制，那么在触发的时候，如果会发生t1已经分配了buffer1，但是t1还没拷贝到buffer1，t2已经拷贝到了buffer，这时候接收到了写请求，说要将t2之前的buffer写入，则在写之前也会等待t1的数据拷贝到buffer1之后再写。

xlog flush