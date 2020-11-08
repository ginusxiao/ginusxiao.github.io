# 提纲
[toc]

# leveldb SST文件 format

    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1]
    ...
    [meta block K]
    [metaindex block]
    [index block]
    [Footer]        (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>
    
在metaindex block和index block这两种索引块中中也存放的是键值对，其中的值被称为BlockHandle（可以理解为指向data/meta block的指针），BlockHandle中包括如下两个信息：

    offset:   varint64
    size:     varint64
    
（1）SST文件（SST File, or SST Table）中存放的是键值对，这些键值对按照key的顺序存放，所有的键值对并被划分为很多分区，每一个分区都被称为一个data block。每一个data block都按照block_builder.cc中的方式进行格式化，这些data blocks也可能被压缩。

（2）在data blocks之后存放了一系列meta blocks，每一个meta block也是按照block_builder.cc中的方式进行格式化，这些meta block是也可能被压缩。

（3）对于metaindex block来说，每一个meta block在metaindex block中存在一个索引项，该索引项的key是meta block的名字，value则是一个指向该meta block的BlockHandle。

（4）对于index block来说，每一个data block在index block中存在一个索引项，该索引项的key是一个字符串，且该字符串满足：不小于该data block中最后一个key，同时小于该data block的后面一个data block的第一个key。该索引项的value也是一个指向data block的BlockHandle。

（5）在SST文件的末尾是一个固定长度的footer，它包含指向metaindex block和index block的BlockHandles，以及一个魔数：

    metaindex_handle: char[p];     // Block handle for metaindex
    index_handle:     char[q];     // Block handle for index
    padding:          char[40-p-q];// zeroed bytes to make fixed length
                                // (40==2*BlockHandle::kMaxEncodedLength)
    magic:            fixed64;     // == 0xdb4775248b80fb57 (little-endian)

# 元Block（Meta Block）
## "filter" Meta Block

如果在打开数据库的时候指定了FilterPolicy，则会在每个SST文件中存储一个filter block，metaindex block中则会包含一个名为filter.<N>的索引项，该索引项的value指向该filter block，其中<N>是通过FilterPolicy的Name()方法返回的。

filter block中存储一系列filters，每一个filter都是通过在某个范围内的keys上调用FilterPolicy::CreateFilter()产生，第i个filter存放的是在所有起始偏移介于[i*base ... (i+1)*base-1]的blocks中的keys上调用FilterPolicy::CreateFilter()产生的filter。假设"base"是2KB，X和Y这两个blocks的起始偏移都介于[0LB ... 2KB-1]，那么将在X和Y这两个blocks中所有的keys上调用FilterPolicy::CreateFilter()产生一个filter，该filter将作为filter block中的第一个filter。

filter block的格式如下:

    [filter 0]
    [filter 1]
    [filter 2]
    ...
    [filter N-1]
    
    [offset of filter 0]                  : 4 bytes
    [offset of filter 1]                  : 4 bytes
    [offset of filter 2]                  : 4 bytes
    ...
    [offset of filter N-1]                : 4 bytes
    
    [offset of beginning of offset array] : 4 bytes
    lg(base)                              : 1 byte
    
filter block中的offset数组实现了data block offset和其对应的filter之间的高效映射。

## "stats" Meta Block

“stats”类型的meta block中包含一系列统计信息，key是统计信息的名称，value则是统计内容。

# Rocksdb BlockBasedTable Format
BlockBasedTable是Rocksdb中默认SST文件格式.

## File format

    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1: filter block]                  (参考: "filter" Meta Block)
    [meta block 2: stats block]                   (参考: "properties" Meta Block)
    [meta block 3: compression dictionary block]  (参考: "compression dictionary" Meta Block)
    ...
    [meta block K: future extended block]  (we may add more meta blocks in the future)
    [metaindex block]
    [index block]
    [Footer]                               (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>
    
（1）SST文件（SST File, or SST Table）中存放的是键值对，这些键值对按照key的顺序存放，所有的键值对并被划分为很多分区，每一个分区都被称为一个data block。每一个data block都按照block_builder.cc中的方式进行格式化，这些data blocks也可能被压缩。

（2）在data blocks之后存放了一系列meta blocks，每一个meta block也是按照block_builder.cc中的方式进行格式化，这些meta block是也可能被压缩。

（3）对于metaindex block来说，每一个meta block在metaindex block中存在一个索引项，该索引项的key是meta block的名字，value则是一个指向该meta block的BlockHandle。

（4）对于index block来说，每一个data block在index block中存在一个索引项，该索引项的key是一个字符串，且该字符串满足：不小于该data block中最后一个key，同时小于该data block的后面一个data block的第一个key。该索引项的value也是一个指向data block的BlockHandle。如果IndexType类型为kTwoLevelIndexSearch，那么index block包括2层，第一层index block中包含一系列关于data block的索引项，而第二层index block中包含一系列关于第一层的index block的索引项，如下：

    [index block - 1st level]
    [index block - 1st level]
    ...
    [index block - 1st level]
    [index block - 2nd level]
    
（5）在SST文件的末尾是一个固定长度的footer，它包含指向metaindex block和index block的BlockHandles，以及一个魔数：

    metaindex_handle: char[p];      // Block handle for metaindex
    index_handle:     char[q];      // Block handle for index
    padding:          char[40-p-q]; // zeroed bytes to make fixed length
                                   // (40==2*BlockHandle::kMaxEncodedLength)
    magic:            fixed64;      // 0x88e241b785f4cff7 (little-endian)
 
## 各种Meta Block   
### Filter Meta Block
#### Full filter

表示整个SST file中只对应一个filter block。

#### Partitioned Filter

Full filter被划分成多个partitioned filter blocks。需要增加一个顶层的index block来管理keys到相应的partitioned filter的映射。读取filter信息的时候，首先将顶层的index block被加载到内存中，通过该顶层index block来按需的将Partitioned filter加载到block cache中。该顶层的index block，只会占用较少的内存，根据cache_index_and_filter_blocks的设置，可以存放于堆中或者block cache中。

#### Block-based filter

也就是Leveldb中的"filter" Meta Block，已经被废弃。

### Properties Meta Block

这种meta block包含一系列属性信息，属性名作为key，属性内容作为value。

默认地，Rocksdb为每个SST文件提供如下属性：

    data size               // the total size of all data blocks. 
    index size              // the size of the index block.
    filter size             // the size of the filter block.
    raw key size            // the size of all keys before any processing.
    raw value size          // the size of all value before any processing.
    number of entries
    number of data blocks
    
Rocksdb为用户提供了收集他们感兴趣的属性的回调，请参考UserDefinedPropertiesCollector。

### Compression Dictionary Meta Block

这种meta block中包含一个为压缩库准备的字典。目的是解决在小的data blocks上执行动态字典压缩算法的一个问题：在遍历data block建立字典的过程中，小的data block具有较小的且可能无效的字典。

解决该问题的方法是，在已经压缩过的data block中抽样，并采用这些抽样数据来为压缩库构建字典。该字典的最大大小可以通过CompressionOptions::max_dict_bytes来配置，默认是0，也就是不产生这种类型的meta block。目前kZlibCompression，kLZ4Compression，kLZ4HCCompression和kZSTDNotFinalCompression压缩算法都支持这种类型的meta block。

更具体的说，Rocksdb只在向最底层合并的过程中构建压缩字典，这时候数据比较大且比较稳定。为了避免多次遍历输入数据，只在compaction中产生的第一个文件中取样，并基于该取样数据构建压缩字典。然后该压缩字典被存放于所有后续产生的SST文件的meta blocks中，并运用到compaction过程中（如果支持压缩的话）。但是该压缩字典不会存储于compaction过程中产生的第一个文件中。

# Rocksdb PlainTable Format

PlainTable格式专门用于纯内存的或者是低时延的介质。

优势：

- 基于hash + binary search的内存索引

- 旁路block cache，避免block拷贝和LRU维护

- 避免查询过程中的内存拷贝（mmap）

# Bloom Filter
## Bloom Filter是什么?

Bloom Filter是一个bit array。对于任意的keys集合，都可以创建Bloom Filter，给定任意一个key，该Bloom Filter可以用于判断一个key是否可能存在，抑或一定不存在。关于Bloom Filter是如何工作的细节，可以参考[维基百科](http://en.wikipedia.org/wiki/Bloom_filter)。

在Rocksdb中，每一个SST文件都包含一个Bloom Filter，用于判断该SST文件中是否包含正在查找的key。

### Bloom Filter的生命周期

在Rocksdb中，每一个SST文件都对应一个Bloom Filter，它在SST文件写入存储的时候被创建，并作为该SST文件的一部分。

Bloom Filter只能通过keys集合来创建，而不能通过结合两个Bloom Filters来得到。当合并两个SST文件的时候，会通过新产生的文件中的所有keys来产生一个新的Bloom Filter。

当打开SST文件的时候，相应的Bloom FIlter会被加载到内存中，当SST文件关闭的时候，该Bloom Filter则从内存中删除。

如果想在Block Cache中缓存Bloom Filter，则设置BlockBasedTableOptions::cache_index_and_filter_blocks=true。

### 内存使用情况
为了构建Bloom Filter，必须确保内存足以容纳构建Bloom Filter的keys集合，因此不太可能为整个SST文件构建Bloom Filter，但是可以为每一个data blocks构建Bloom Filter，这样只需在内存中保留较少的keys。

随着SST文件中键值对的顺序写入，每当一个data block写满的时候，Rocksdb就为之构建一个Bloom Filter并写入该SST文件中。

## 新Bloom filter格式

上述讲到的原始的Bloom Filter模式中，Rocksdb会为每一个data block创建一个Bloom Filter，因此在构建Bloom Filter的过程中无需太多的内存。

在前面讲到有一种Filter类型叫做“Full Filter”，它当中包含整个SST文件的所有的keys。这种Bloom Filter模式可以提升读性能，因为它无需遍历相对较复杂的SST文件格式了，但是它需要更多的内存来构建该Bloom Filter。

用户可以通过tableOptions::FilterBlockType选项来指定使用哪种类型的Bloom Filter。默认地，Rocksdb使用原始的Bloom Filter（即为每个data block建立Bloom Filter）。

Full Filter格式如下：

    [filter for all keys in SST file][num probes]
    
(1) Full Filter是一个大的bit array，用于检查SST文件中的所有的keys。

(2) num probes指定创建Bloom Filter时使用的hash函数的个数[probe表示hash函数产生的bit位]。

### 新的Bloom Filter格式的优化

(1) 在内存中保存key的hash值，以减少内存消耗。

(2) 在同一个CPU cache line上读/写多个probes，使得读写更快。

### 新的Bloom Filter的使用

默认使用原始的Bloom Filter，如果想使用新的Full Filter，则在创建FilterPolicy的时候添加一个参数：

    NewBloomFilterPolicy(10, false);    // The second parameter means "not use original filter format"
    
### 自定义FilterPolicy

原始的Filter policy接口比较固定，不太适合于新的Full Filter格式，为此定义了两个新的接口（include/rocksdb/filter_policy.h）：

    FilterBitsBuilder* GetFilterBitsBuilder()
    FilterBitsReader* GetFilterBitsReader(const Slice& contents)
    
通过这两个接口，新的Filter policy可以作为FilterBitsBuilder和FilterBitsReader的工厂。FilterBitsBuilder提供接口用于产生filter，FilterBitsReader则提供接口用于检查某个key是否在filter中。

注意：这两个新的接口只适用于Full Filter。

## Partitioned Bloom Filter

参考[这里](https://github.com/facebook/rocksdb/wiki/Partitioned-Index-Filters)。