# 提纲
[toc]

# 简介
numa(Non-uniform memory access)是多核架构下的一种内存设计，在numa中内存访问时延取决于内存和CPU之间的相对位置，CPU访问它所在的numa node本地的内存时延会更低。openGauss内存引擎的内存访问被设计为numa感知的。本文将分析openGauss内存引擎(MOT)的内存管理。

![image](https://note.youdao.com/yws/public/resource/e6edfab05e5fc6afcd19b8f7e4006df6/xmlnote/BEC665CD0F004CB8AF0655DD5EDAE62A/117059)

MOT的内存管理中主要涉及到以下几种对象：
- RawChunkStore，用于预分配以Chunk为单位的大块内存，所有分配的Chunk纳入MOT管理，直到MOT退出时将这些Chunk归还给系统
- MemBufferAllocator，用于分配特定大小的内存块，比如1KB，2KB，4KB，...
- ObjectPool，用于特定对象的内存池，比如关于行记录内存池，索引节点内存池等

# RawChunkStore
RawChunkStore中包含一系列的MemRawChunkPool，每个MemRawChunkPool管理一系列的RawChunk。

每个numa node都分别对应一个global的MemRawChunkPool和一个local的MemRawChunkPool，所有numa node的global的MemRawChunkPool都保存在全局的globalChunkPools数组中，所有numa node的local的MemRawChunkPool都保存在全局的localChunkPools数组中。

![image](https://note.youdao.com/yws/public/resource/e6edfab05e5fc6afcd19b8f7e4006df6/xmlnote/FE0982AFB0264DC0B121BB70C7EF642A/117171)

在每个numa node初始化它对应的MemRawChunkPool时，会分配chunkReserveCount个RawChunk，对于global的MemRawChunkPool来说，chunkReserveCount = min_mot_global_memory/${chunk_size}/${numa_nodes_num}，对于local的MemRawChunkPool来说，chunkReserveCount = min_mot_Local_memory/${chunk_size}/${numa_nodes_num}，chunk_size默认是2MB。

在初始化每个CPU core对应的MemRawChunkPool过程中，分配RawChunk的过程是多线程异步执行的，每个线程负责分配一定数目的RawChunk，每个RawChunk的分配过程如下：
- 分配RawChunk，根据alloc_type和chunkAllocPolicy的不同组合，使用不同的分配方法(但是都确保分配2MB的RawChunk，并且起始地址和2MB对齐)：
    - 如果alloc_type为MEM_ALLOC_GLOBAL，即分配的是global的MemRawChunkPool
    	- 如果chunkAllocPolicy为LOCAL
    		- 从当前MemRawChunkPool对应的numa node本地分配
    		- 实现机制：
    		    - 调用mmap，在进程的虚拟地址空间中分配一片匿名的内存区域，返回虚拟内存地址，记为addr
    		        - addr一定是按照2M对齐的
    		    - 调用syscall(__NR_mbind, void *addr, unsigned long len, int mode, const unsigned long *nodemask, unsigned long maxnode, unsigned flags)设定内存分配策略
    		        - nodemask中只设置当前MemRawChunkPool对应的numa node所在的bit位
    		    - 调用munmap解除进程虚拟地址空间中的映射关系
    	- 如果chunkAllocPolicy为CHUNK_INTERLEAVED
    		- 以Chunk为单位进行RoundRobin分配，即如果上一个Chunk在numa node n上分配，则下一个Chunk在numa node n+1上分配
    		- 实现机制：
    		    - 跟chunkAllocPolicy为LOCAL类似，只不过每次Chunk分配的时候所使用的nodemask都会改变
    	- 如果chunkAllocPolicy为PAGE_INTERLEAVED
    		- 以Page为单位进行RoundRobin分配
    		- 实现机制：
    		    - 跟chunkAllocPolicy为LOCAL类似，但是在syscall(__NR_mbind, ...)中指定的nodemask中设置了所有numa node的bit位，并且mode参数被设置为MPOL_INTERLEAVE
    	- 如果chunkAllocPolicy为NATIVE
    		- 直接采用posix_memalign进行分配
    - 如果alloc_type为MEM_ALLOC_LOCAL，即分配的是local的MemRawChunkPool
    	- 如果chunkAllocPolicy为NATIVE
    		- 直接采用posix_memalign进行分配
    	- 否则
    		- 从当前MemRawChunkPool对应的numa node本地分配
    		- 实现机制：
    		    - 调用mmap，在进程的虚拟地址空间中分配一片匿名的内存区域，返回虚拟内存地址，记为addr
    		        - addr一定是按照2M对齐的
    		    - 调用syscall(__NR_mbind, void *addr, unsigned long len, int mode, const unsigned long *nodemask, unsigned long maxnode, unsigned flags)设定内存分配策略
    		        - nodemask中只设置当前MemRawChunkPool对应的numa node所在的bit位
    		    - 调用munmap解除进程虚拟地址空间中的映射关系
- 如果reserveMode为MEM_RESERVE_PHYSICAL，则需要确保真实分配了物理内存，否则如果reserveMode为MEM_RESERVE_VIRTUAL，则无此要求
- 设置chunk_type，node和alloc_type等信息
- 根据RawChunk的虚拟地址，将其插入到对应的ChunkDirParts中
- 从尚未使用的MemLFStackNode栈中pop出来一个MemLFStackNode
- 将pop出来的这个MemLFStackNode指向分配的RawChunk
- 将pop出来的MemLFStackNode push到已经使用的MemLFStackNode栈中


## 关于系统调用syscall(__NR_mbind, ...)的说明
系统调用syscall(__NR_mbind, ...)的原型如下：
```
syscall(__NR_mbind, void *addr, unsigned long len, int mode,
      const unsigned long *nodemask, unsigned long maxnode, unsigned flags)
```

### 函数说明
这是一个系统调用，用于设置给定内存区域的numa内存策略，numa内存策略用于决定从哪个numa node分配内存。

### 参数说明
addr: 给定内存区域的起始地址。

len: 给定内存区域的长度，以bytes为单位。

mode: numa内存策略，可取值为MPOL_DEFAULT, MPOL_BIND, MPOL_INTERLEAVE, MPOL_PREFERRED, 或者MPOL_LOCAL。

nodemask: 关于numa nodes的bit掩码，至多包含maxnode个有效bit位，nodemask所占用的空间会向上对齐到下一个unsigned long，如果nodemask为null，则表示指定了空的numa node集合。

maxnode: nodemask中至多有多少个有效的bit位，如果maxnode为0，则表示指定了空的numa node集合。

flags: 可取值为MPOL_MF_STRICT, MPOL_MF_MOVE, MPOL_MF_MOVE_ALL。

### 关于mode可取值的说明
MPOL_DEFAULT：使用线程级别(与之对应的包括系统级别)的内存策略(可以通过set_mempolicy(2)进行设置)，如果线程级别的内存策略是MPOL_DEFAULT，则使用系统级别默认的内存策略。在系统级别默认策略下，内存页分配会在请求内存页分配的那个CPU所在的numa node上进行。

MPOL_BIND：只能在nodemask中指定的numa nodes上分配内存。如果nodemask中指定了多个numa nodes，则从离请求内存页分配的那个numa node最近且具有充足内存的numa。 node上分配内存页。不会在nodemask中指定的numa nodes以外的numa node上分配内存页。

MPOL_INTERLEAVE：在nodemask中指定的所有numa nodes中交替分配内存页。

MPOL_PREFERRED：优先从preferred node上分配内存页，如果preferred node上内存不足，则从其它非preferred nodes中分配内存。如果nodemask中指定了多个numa nodes，则第一个numa node将作为preferred node，如果nodemask和maxnode指定了空的numa node集合，则请求内存页分配的那个numa node就是preferred node。

MPOL_LOCAL(从Linux 3.8开始支持)：从请求内存页分配的那个numa node上分配内存。nodemask和maxnode都必须指定空的numa node集合。如果请求内存页分配的那个numa node上内存不足，则从其它numa nodes上分配。

### 关于flags可取值的说明
MPOL_MF_STRICT：如果flags被指定为MPOL_MF_STRICT，且mode不为MPOL_DEFAULT，则给定内存区域内如果存在内存页违背了mode所指示的内存策略，则提示EIO错误。

MPOL_MF_MOVE：如果flags被指定为MPOL_MF_MOVE，则内核会尝试移动所有已经存在的页，以便它们遵守mode所指示的内存策略，但是被其它进程共享的内存页则不被移动。

MPOL_MF_MOVE_ALL：如果flags被指定为MPOL_MF_MOVE_ALL，则内核会尝试移动所有已经存在的页，以便它们遵守mode所指示的内存策略，包括被其它进程共享的内存页。

# MemBufferAllocator
MOT中会进一步将2MB的RawChunk拆分为1KB，2KB，4KB，...， 1022KB的MemBuffer，不同大小的MemBuffer由不同的MemBufferAllocator管理。

对于每个numa node来说，它包含11个global的MemBufferAllocator，分别对应1KB，2KB，4KB，...， 1022KB的MemBuffer，同时包含11个local的MemBufferAllocator，分别对应1KB，2KB，4KB，...， 1022KB的MemBuffer。

![image](https://note.youdao.com/yws/public/resource/e6edfab05e5fc6afcd19b8f7e4006df6/xmlnote/60885EB3E14C49FAA04283267FB9DDA7/117162)

## 从MemBufferAllocator中分配一个Buffer
1. 从当前线程对应的MemBufferChunkSnapshot中分配：
    - 找到当前线程对应的MemBufferChunkSnapshot(MemBufferAllocator::m_chunkSnapshots[threadId])，记为chunkSnapshot;
    - 如果chunkSnapshot::m_realChunkHeader(类型为MemBufferChunkHeader，记为realChunkHeader)不为空：
        - 从chunkSnapshot中分配(分配过程见“从MemBufferChunkSnapshot中分配Buffer的过程”)；
    - 否则：
        - 返回null，表示从MemBufferChunkSnapshot分配失败；

2. 如果在MemBufferChunkSnapshot上分配失败，则在MemBufferHeap上分配Chunk，并填充MemBufferChunkSnapshot；
    - 在MemBufferHeap上加锁；
    - 从MemBufferHeap::m_fullnessBitSet中第0个元素开始查找第一个具有free buffer的BitSet，记为firstNonEmptyBitSet；
    - 如果存在这样的firstNonEmptyBitSet
        - 从firstNonEmptyBitset中LSB(least significant bit)向MSB(most significant bit)方向查找第一个不为0的bit位，记为firstNonEmptyBit；
        - 查找firstNonEmptyBit在MemBufferHeap::m_fullnessDirectory中对应的Chunk List，记为firstFreeChunkList；
        - 从firstFreeChunkList头部pop出来一个Chunk；
        - 以该Chunk对应的MemBufferChunkHeader初始化新的MemBufferChunkSnapshot(的m_chunkHeaderSnapshot)；
        - 将该Chunk对应的MemBufferChunkHeader设置为没有free Buffers，并添加到MemBufferHeap::m_fullChunkList中；
        - 释放MemBufferHeap上加的锁；
        - 从当前的MemBufferChunkSnapshot中分配Buffer(分配过程见“从MemBufferChunkSnapshot中分配Buffer的过程”)；
    - 否则：
        - 释放MemBufferHeap上加的锁；
        - 返回null，表示从MemBufferHeap分配失败；

3. 如果从MemBufferHeap中分配失败，则从RawChunkStore中分配Chunk，并填充MemBufferChunkSnapshot；
    - 根据alloc_type的不同，可能从globalChunkPools中分配，也可能从localChunkPools中分配，分配的是RawChunk(通过MemRawChunkHeader引用之)；
    - 对分配的RawChunk格式化为MemBufferChunk(通过MemBufferChunkHeader引用之)；
    - 以该MemBufferChunkHeader初始化新的MemBufferChunkSnapshot(的m_chunkHeaderSnapshot)；
    - 将该MemBufferChunkHeader设置为没有free Buffers；
    - 在MemBufferHeap上加锁；
    - 将MemBufferChunkHeader添加到MemBufferAllocator::m_fullChunkList中；
    - 更新MemBufferHeap中分配的MemBufferChunk计数；
    - 释放MemBufferHeap上加的锁；
    - 设置MemBufferChunkSnapshot::m_newChunk为true；
    - 从当前的MemBufferChunkSnapshot中分配Buffer(分配过程见“从MemBufferChunkSnapshot中分配Buffer的过程”)；

## 从MemBufferChunkSnapshot中分配Buffer
- 如果这是一个从RawChunkStore中分配的一个新的Chunk(MemBufferChunkSnapshot;:m_newChunk为true)：
    - 从MemBufferChunkSnapshot中分配：MemBufferHeader* bufferHeader = MM_CHUNK_BUFFER_HEADER_AT(chunkSnapshot->m_realChunkHeader, chunkHeader->m_allocatedCount++)；
    - 如果该MemBufferChunkSnapshot中没有free Buffers了，则设置MemBufferChunkSnapshot::m_realChunkHeader为null；
- 否则：
    - 从MemBufferChunkSnapshot::m_bitsetIndex开始在MemBufferChunkSnapshot::m_freeBitset数组中查找第一个不为0的Bitset；
    - 从该Bitset中从MSB(most significant bit)开始向LSB(least significant bit)方向查找第一个free的Buffer；
    - 将该Bitset中相应的bit位设置为0；
    - 如果该MemBufferChunkSnapshot中没有free Buffers了，则设置MemBufferChunkSnapshot::m_realChunkHeader为null；
    - 返回该Buffer；

## 向MemBufferAllocator中释放一个Buffer
- 获取Buffer对应的MemBufferHeader；
- 获取Buffer所在的Chunk(通过MemBufferChunkHeader引用之，后文将以MemBufferChunkHeader指代Chunk)；
- 获取当前线程在MemBufferAllocator中的MemBufferList(其中存放最近被释放的buffers，等待批量返还给MemBufferHeap)；
- 将待释放的Buffer添加到上一步骤中获取到的MemBufferList的头部；
- 如果MemBufferList中的Buffer的数目超过了某个阈值(由MemBufferAllocator::m_freeListDrainSize控制)，则将其返还给对应的MemBufferHeap中：
    - 在MemBufferHeap上加锁；
    - 从MemBufferList的头部开始逐一处理每一个元素，对于每一个Buffer(通过MemBufferHeader引用之，后文将以MemBufferHeader指代Buffer)处理如下：
        - 获取MemBufferHeader对应的MemBufferChunkHeader；
        - 将该MemBufferChunkHeader从它所在的Chunk List(对应于MemBufferHeap中的m_fullCHunkList或者m_fullnessDirectory数组中的某个元素)中移除(因为每个Chunk List中存放的都是具有特定数目free buffer的Chunks，当前MemBufferChunkHeader中free buffers的数目即将发生改变)；
            - 如果在MemBufferHeap::m_fullChunkList中，则直接移除之；
            - 否则：			
                - 获取该MemBufferChunkHeader在MemBufferHeap::m_fullnessDirectory中的哪个Chunk List上(根据MemBufferChunkHeader所代表的Chunk中有多少个free buffers来确定)；	
                - 从该Chunk List中移除该MemBufferChunkHeader；
                - 如果移除该MemBufferChunkHeader之后，该Chunk List变为空，则要将MemBufferHeap::m_fullnessBitSet中的相应的bit位设置为0；	
        - 将MemBufferHeader(实际上是MemBuffer)归还给MemBufferChunkHeader(实际上是MemBufferChunk)；
            - 将MemBufferChunkHeader::m_freeBitset中相应的bit位置为1(表示该Buffer是free的了)；
            - 减少MemBufferChunkHeader中记录的已分配的Buffer的数目；
        - 减少MemBufferHeap中记录的已分配的Buffer的数目；
        - 将MemBufferChunkHeader添加到MemBufferHeap中相应的Chunk List中;
            - 如果剩余的buffer是数目为0，则添加到MemBufferHeap::m_fullChunkList中
            - 否则:
                - 如果MemBufferChunkHeader中记录的已分配的Buffer的数目为0，且设置了允许释放Chunk，则将Chunk返还给RawChunkStore；
                - 否则，添加到相应的MemBufferHeap::m_fullnessDirectory中的Chunk List中；
    - 释放MemBufferHeap上加的锁；

# ObjectPool
![image](https://note.youdao.com/yws/public/resource/e6edfab05e5fc6afcd19b8f7e4006df6/xmlnote/8CE3683675B84F3CA814BC4517A49844/115613)

MOT关于ObjPool的设计使用了一些小技巧：
1. 每个ObjPool中最多保存NUM_OBJS=255个Objects；
2. m_objIndexArr数组中包含NUM_OBJS + 1 = 256个Objects，而实际占用ObjPool::m_totalCount个Objects；
3. m_nextFreeObj和m_nextOccupiedObj都是uint8类型，最大只能描述256个Objects，当超过256时，就会从0开始计数；
4. m_nextFreeObj计数从0开始，直到255，然后又从0开始，直到255…；
5. m_nextOccupiedObj计数从ObjPool::m_totalCount - 1开始，直到255，然后又从0开始，直到255，然后又从0开始，直到255...；
6. 因为一个ObjPool中最多包含NUM_OBJS=255个Objects，所以m_nextFreeObj和m_nextOccupiedObj始终不会重叠；

在ObjPool初始状态下，m_objIndexArr[i]一定等于i，即m_objIndexArr[i]中记录的是从m_data开始的第i个Objects的索引，但是随着ObjPool中的Object被分配和释放，m_objIndexArr[i]不一定等于i了。

## 从LocalObjPool分配Object
- 检查m_nextFree链表是否为空；
- 如果为空：
    - 从global或者local(由调用指定到底是从global还是local分配)的MemBufferAllocator中分配一个ObjPool；
    - 对该ObjPool进行格式化(比如，设置m_totalCount，m_freeCount，m_head.m_objIndexArr，每个对象中记录的该对象在m_head.m_data[0]数组中的索引等)；
    - 将新分配的ObjPool添加到m_objList链表中；
    - 将新分配的ObjPool添加到m_nextFree链表中；
- 从m_nextFree所指向的的ObjPool中分配：
    - 获取m_head.m_nextFreeObj，并加1，得到新的m_head.m_nextFreeObj；
    - 获取m_head.m_objIndexArr[m_head.m_nextFreeObj]中记录的Object的索引，记为objectIndex；
    - 设置m_head.m_objIndexArr[m_head.m_nextFreeObj]为-1，表示该元素不再指向有效的Object；
    - 获取从m_head.m_data开始的第objectIndex个Object，返回其起始地址，记为objAddr；
    - 设置以objAddr为起始地址的Object由m_head.m_objIndexArr中第objectIndex个元素指向它；
    - 减少当前ObjPool中m_freeCount计数；
- 如果分配之后，当前ObjPool中m_freeCount变为0，则将该ObjPool从m_nextFree链表中移除；


## 释放从LocalObjPool中分配的Object
- 获取该Object由m_head.m_objIndexArr中的哪个元素指向它，记为objIdx；
- 将该Object中记录的关于“m_head.m_objIndexArr中哪个元素指向它”的信息设置为-1，表示该Object当前已经被返还给了LocalObjectPool；
- 获取该Object所在的ObjPool；
- 获取ObjPool中m_head.m_nextOccupiedObj，并加1，得到新的m_head.m_nextOccupiedObj；
- 设置ObjPool中m_head.m_objIndexArr[m_head.m_nextOccupiedObj]为objIdx，表示从m_head.m_data开始的第objIdx个Object是free的；
- 增加ObjPool中m_freeCount计数；
- 如果该ObjPool中m_freeCount计数变为1，则将这个ObjPool添加到m_nextFree链表中；

