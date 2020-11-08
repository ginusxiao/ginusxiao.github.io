# 提纲
[toc]

# 简介
SILO是一个全新的内存OLTP数据库，在现代多核系统上具有非常好的性能和扩展性。openGauss内存引擎参考SILO来实现其事务并发控制。

# 设计
## epoch
SILO中一个重要概念叫做Epoch，recovery和garbage collection都会用到它。SILO通过一个专门的线程去维护一个全局的递增的Epoch，这个全局的Epoch对于所有其它的线程都是可见的。

这个全局的Epoch必须频繁更新，因为Epoch会影响事务的latency(注：SILO只在事务持久化之后才会给用户发送响应，且SILO只在非常确信某个Epoch D之前的所有事务都已经持久化的情况下，才会给用户发送Epoch <= D的事务的响应，这在后面持久化中会讲到，现在知道是这么回事即可)，但是Epoch相对于事务的执行时间来说相对较长，所以全局的Epoch信息会在每个执行事务的线程本地缓存。在SILO中，Epoch每隔40ms更新一次。

## Transaction IDs
Transaction ID，也被简称为TIDs，具有以下用途：标记事务和记录的版本号，充当锁，冲突检测等。

每一个记录中都会包含TID信息，也就是最近成功修改它的那个事务的TID。

TID是64 bit的整形数，它会被划分为3个部分：高位的bits存放Epoch，中间位的bits存放相应事务的commit time，低位的3个bits则存放status bits，status bits包括lock bit，latest-version bit和absent bit。lock bit用于保护记录不被并发更新，latest-version bit标记该记录是否是最新的记录，absent bit则用于标记该记录是不存在的。

SILO采用一种去中心化的方式来分配TID，每个线程独自为它所执行的事务分配TID，**一个线程当且仅当它正在执行的事务可以被应用(已经通过了冲突检测，并且加了相应的锁，在后面的commit-protocol中会讲到)的情况下才会申请分配TID**，TID分配规则如下：
(1) 比当前事务所读取的或者写入的记录中的TID大
(2) 比当前线程最近已经分配的TID都要大
(3) 在当前的全局的Epoch中

从上面的TID分配算法来看，在每个线程内部，TID是有序的，在两个Epoch之间也是有序的，但是对于同一个Epoch内的不同线程上的TID的有序性无法保证。

## 数据组织
SILO中的table被实现为一系列的索引树，包括一个主键索引树和0个或者多个二级索引树。

Table中的每一行记录被保存在一个单独的内存块中，在主键索引中，每一个索引项的value部分都指向一行记录对应的内存块。在二级索引中，每一个索引项的value部分都关联到一个包含主键或者主键集合的记录。

SILO使用Masstree作为它的索引结构。

主键索引必须是unique的(这里指的是主键不能重复)，如果一个table没有唯一主键，SILO会在内部创建一个。二级索引可以是non-unique的，因此在二级索引中，一个key可能对应多行记录，那么SILO中是如何使用Masstree去保存二级索引的呢？SILO对这一点并没有明确说明，我们暂不得而知。但是openGauss的内存引擎是借鉴了SILO的，它里面是如何实现non-unique的二级索引的呢，我们下面一探究竟。

从实现上来说，Masstree是不支持同一个key映射到多个不同的value的，openGauss内存引擎采用二级索引key + suffix作为真正的二级索引key，suffix采用的是行数据在内存中的指针。这样以来，non-unique的二级索引在内部变成了unique的二级索引。
![image](https://note.youdao.com/yws/public/resource/e6cfed8070981eb708f58a466e78e342/xmlnote/CFD3F46B5956416B862150BBD44322F0/115237)

对于non-unique的二级索引来说，可以直接以key(而无需suffix部分)进行查找操作，返回以key作为前缀的所有记录，当然也可以指定更多的过滤条件。对于插入和删除操作，则必须使用完整的key + suffix，因此对于插入和删除操作来说，相应的行必须作为参数(对于删除来说，相应的行作为参赛比较好理解，但是对于插入操作，相应的行不存在啊，事实上，openGauss内存引擎会在插入操作执行之前事先生成一个新的行)。

SILO中一行数据包括以下部分：
- TID
- 指向前一个版本的指针
- 记录数据

## commit protocol
这里首先讨论更新已经存在的行的情况，后面会讨论insert和delete的情况。

当一个线程执行某个事务时，它会将该事务中所有被读取的记录连带这些记录的TID信息保存在线程本地的read-set中，同时将该事务中所有被更新的记录(但是不包括这些记录的TID信息)保存在线程本地的write-set中，对于那些同时被读取和更新的记录，则会同时保存在read-set和write-set中。

当当前事务执行完毕之后，线程会按照如下的commit protocol尝试提交：
```
// Phase 1
for w, v in WriteSet {
    Lock(w); // use a lock bit in TID
}

Fence(); // compiler-only on x86，确保获取Global_Epoch的过程是在之前所有的内存访问之后
e = Global_Epoch; // serialization point
Fence(); // compiler-only on x86，确保获取Global_Epoch的过程是在后续所有的内存访问之前

// Phase 2
for r, t in ReadSet {
    Validate(r, t); // abort if fails
}

tid = Generate_TID(ReadSet, WriteSet, e);

// Phase 3
for w, v in WriteSet {
    Write(w, v, tid);
    Unlock(w);
}
```

Phase 1：
- 对write-set中的所有的记录(非本地记录，而是table中的全局共享的记录)加锁(将TID中的lock bit设置为1)，为了避免死锁，必须按照一定的顺序进行加锁，SILO采用的是按照记录的地址的顺序进行加锁
- 获取全局的Epoch(必须确保是全局的，而非本地缓存的)

Phase 2：
- 检查read-set中的所有的记录，对于每个记录，都要拿read-set中的本地记录跟table中的全局共享的记录进行比较，检查是否发生以下现象：
    - 两者的TID发生了变更
    - table中的全局共享的记录不是最近的记录(TID中的latest-version bit为0)
    - 其它事务已经在table中的全局共享的记录上加了锁(或者说，TID中的lock bit为1，但是这个记录不在当前事务的write-set中)
- 如果检查结果显示上述现象中有至少一种发生，则释放锁，并且abort当前的事务
- 如果检查结果显示上述现象都没有发生，则使用在Phase 1中获取的Epoch结合TID分配算法来为当前的事务分配一个TID

Phase 3：
- 将所有的更新应用到table中所有的全局共享的记录中
- 更新这些记录中的TID为当前事务的TID
- 释放当前事务在table中的全局共享记录上所加的锁

### 示例
假设有两个事务T1和T2，分别执行如下操作：
```
T1：
v1 = read(x)
write(y = v1 + 1)

T2:
v2 = read(y)
write(x = v2 + 1)
```

假设x和y的初始值都是0，则结果中一定不会出现x=1且y=1的情况！我们采用反证法，假设会出现x=1且y=1的情况，则v1一定是0且v2一定是0，且事务T1和事务T2均成功提交。假设事务T1获取到的v1=0，且事务T1成功提交，则事务T1一定通过了phase 2的检查，也就是说x的值和v1相比没有发生变更，x上也没有被加锁，事务T2一定还没有执行commit protocol(否则，要么x上加了锁，要么x的值已经不再等于v1)。当事务T2执行commit protocol的时候，在phase 2的检查过程中，要么y上已经加了锁(事务T1还在执行commit protocol的过程中)，要么y的值和v2相比已经发生了变更(事务T1已经执行完commit protocol)，如果y上加了锁，则事务T2必须abort，对应的结果是x=0且y=1，如果y的值和v2相比已经发生了变更，则对应的结果是y=1且x=2。

## 数据库操作
### Read和write操作
如果一个事务可以应用它的更新(即通过了commit阶段的冲突检测，并且加了相应的锁)，则它会首先尝试执行就地更新(这里要区别于事务在执行阶段，只会更新自己的私有空间的本地记录，而不会就地更新共享的数据)，如果无法就地更新，则会分配一个新的内存块来容纳数据，标记旧的记录不再是最新的数据，同时更新索引，使之指向新的版本的数据。

如果执行了就地更新，则并发的读操作可能会读取到不一致的数据(部分数据是旧版本的，部分数据是新版本的)，为了处理这种情况，采用了如下的version-validation protocol：
- 在commit protocol的Phase 3中，当前事务的执行线程会:
    - (a) 持有锁 
    - (b) 应用更新的table中所有的全局共享的记录 
    - (c) 执行一次memory fence/barrier
    - (d) 更新TID同时释放持有的锁
    - 步骤(c)确保新的记录先于TID可见，步骤(d)则确保TID和lock原子的更新(因为TID和lock是在同一个64 Bits中)
- 在另外一个事务的执行线程读取记录的时候，会:
    - (a) 读取记录中的TID信息，自旋直到lock bit被清除
    - (b) 检查记录是否是最新的版本
    - (c) 读取记录中的数据
    - (d) 执行一次memory fence/barrier
    - (e) 再次检查记录中的TID
    - 如果步骤(b)中发现记录不是最新的版本，或者记录中的TID信息在步骤(a)和步骤(e)之间发生了变更，则事务必须重试或者abort

### insert操作
在commit protocol的Phase 2中通过在待更新的记录上加锁来处理写写冲突，但是对于insert操作来说，不存在这样的记录，因此也就无法加锁。为了避免这种问题，在执行commit protocol之前会为insert操作首先添加一个新的记录。


假设在某个key上执行insert操作，执行流程如下：
- 如果key已经关联到了一个记录且该记录不是absent的(即记录中的TID中的absent bit为0)，则该插入操作失败，事务abort。
- 否则：
    - 构造一个记录record
    - 标记该record为absent的(将记录中TID中的absent bit设置为1)
    - 设置该record的TID中除status bit以外的部分为0
    - 在索引中添加一条关于key到record的映射
    - 将record同时添加到read-set和write-set中

在执行插入操作的过程中，索引操作必须支持insert-if-absent原语，以确保同一个key不会有2个或者更多的记录。同时，commit-protocol中Phase 2中针对read-set的检查过程确保没有其它的并发事务替换了索引树中为insert操作事先添加的记录。

### delete操作
snapshot事务要求被删除的记录依然保留在索引中，因此delte操作会将它的记录标记为absent(通过将记录中TID中的absent bit设置为1)，同时将这个记录注册到garbage collection中，因此，SILO的写操作不会直接在这些被标记为absent的记录上执行就地更新。

## 持久化
虽然SILO中数据是保存在内存中的，但是会通过写持久化的日志来实现事务持久性保证，在事务没有持久化之前，是不能向用户发送成功响应的。SILO只在确信所有不大于某个Epoch D的事务都已经持久化的情况下，才会向用户发送这些Epoch不大于D的事务的响应。

SILO中包括一系列logger线程，每一个logger线程都负责所有线程中一部分线程执行的事务的持久化。每一各logger线程写不同的日志文件。

每个事务执行线程w在应用(commit-protocol中Phase 3)更新到table中的时候，会创建一个日志记录，其中包括事务的TID和table/key/value等信息。这些日志记录会被保存在本地内存缓存中，当本地内存缓存满了或者Epoch发生变更的时候，该线程w会将本地内存缓存发布到相应的logger线程中关于当前事务执行线程w的日志队列中(也就是说，logger中会为它所负责的每个线程维护单独的日志队列)，然后将当前事务的TID发布到一个全局的关于当前线程的ctid[w](表示线程w最新已提交事务的TID)中。

logger线程则不断的循环运行，在每次循环中，它执行如下：
- 计算它所负责的所有线程w的ctid[w]中最小的那个，记为t = min {ctid[w]}
- 计算logger线程自身所看到的最小已持久化的Epoch d[logger] = Epoch(t) - 1，即logger线程所看到的最小已持久化的Epoch是它所负责的所有线程的已提交的事务的TID中最小的那个对应的Epoch减去1，所有Epoch <= d的事务都已经持久化了
- logger线程将它所负责的所有线程的日志队列中的日志记录写入log文件中
- 当所有日志记录都成功写入日志文件之后，logger线程将Epoch d发布到全局的关于每个logger线程的de[logger](表示该logger已经确认所有Epoch <= de[logger]的事务都已经持久化)中

SILO中使用一个单独的线程负责周期性的计算一个全局的Epoch D = min {de[logger]}，所有Epoch <= D的事务都已经确认持久化了。所有的事务执行线程就可以向用户发送所有Epoch <= D的事务的响应了。

## 故障恢复
当系统故障重启情况下，SILO会检查所有的logs，计算每个logger线程对应的de[logger]，并且以Epoch D = min  {dl[logger]}作为恢复的终点。注意，不会恢复Epoch > D的任何日志，是因为对于Epoch > D的任何Epoch，它所相关的事务并没有完全持久化成功，而且这些事务之间的有序性无法保证(因为每个执行事务的线程都是独自分配TID的)，从这些日志中恢复可能会导致不一致的状态。

因为SILO在日志中记录的是REDO日志，所以恢复过程直接replay这些REDO日志即可。关于相同数据行的不同日志记录，必须按照TID的顺序进行恢复。

为了加速恢复，还可以借助checkpoint，并结合持久化日志共同完成恢复认为。

# 参考
[Speedy transactions in multicore in-memory databases](https://dl.acm.org/doi/10.1145/2517349.2522713)

