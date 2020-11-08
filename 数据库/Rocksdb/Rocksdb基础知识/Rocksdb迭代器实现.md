# 提纲
[toc]

# Iterator基础知识
## 一致性视图（Consistent View）

如果ReadOptions.snapshot不为NULL，迭代器将会读取该snapshot的数据，否则如果ReadOptions.snapshot为NULL，迭代器将从迭代器创建时刻的隐示的snapshot中读取数据。该隐示的snapshot通过将相应的资源pin住从而被保留。没有方法将隐示的snapshot转换为显示的snapshot。

## 错误处理（Error Handling）

Iterator::status()返回迭代过程中的错误。迭代器可能会跳过它读取过程中遇到的读取困难（因为IO错误，数据损坏或者其它原因）的数据块或者文件，继续读取下一个可用的keys。即使在Valid() = true的情况下，每一个迭代器操作之后的status()方法都可能不OK： Seek(), Next(), SeekToFirst(), SeekToLast(), SeekForPrev(), and Prev()。

## 迭代上限（Iterating upper bound）

可以通过在传递给NewIterator()方法的选项中设置ReadOptions.iterate_upper_bound选项设定范围查询的上限。通过设置该选项，Rocksdb无需查找该上限后的key。迭代上限既可以在顺序迭代中设置，也可以在逆向迭代中设置。

## 迭代器pin住资源和迭代器刷新（Resource pinned by iterators and iterator refreshing）

迭代器自身不会使用太多的内存，但是它可能导致某些资源不被释放。包括：

迭代器创建时刻的memtables和SST files。即使某些memtables和SST files在flush或者compaction之后应当被删除，但是如果它们被迭代器pin住了，它们也应该被保留。

当前迭代位置的数据块。这些数据块将被保存在内存中，要么pin在Block Cache中，要么在堆中（如果Block Cache未开启的情况下）。

因此使用迭代器时应当确保尽量短的使用之，以便它能够及时的释放被pin住的资源。

迭代器创建会有一些开销，在某些情况下，可以通过复用迭代器来减少该开销，但必须意识到它可能阻止某些旧的资源被释放。为了应对这种情况，在Release 5.7之前，需要先销毁迭代器然后重新创建之。但从Release 5.7开始，可以通过调用Iterator::Refresh()来刷新之，该方法将刷新迭代器使之对应当前的状态，并且它所pin住的旧的资源将被释放。

## 前缀迭代（Prefix Iterating）

前缀迭代允许用户在迭代器上使用bloom filter或者hash index，以此来提升性能。但是该功能有其局限性，错误的使用将会带来错误的结果，并且没有错误提示。因此建议谨慎使用该功能。关于如何使用该功能，参考[Prefix Seek](https://github.com/facebook/rocksdb/wiki/Prefix-Seek-API-Changes)。total_order_seek和prefix_same_as_start选项只能在前缀迭代中使用。

## 预读（Read-ahead）

当前Rocksdb不支持任何通用的预读。如果应用主要是通过迭代器访问，并且只依赖于操作系统page cache作为缓存，可以通过设置DBOptions.advise_random_on_open = false开启预读。对于直连的SSD设备（directly attached SSD devices）来说，预读带来的效果不太明显。

Rocksdb预读功能仍在不断提升过程中。

## SeekForPrev

从4.13开始，Rocksdb添加了Iterator::SeekForPrev() API。这个新的API将会寻找到最后一个不大于指定key的key。
```
    // Suppose we have keys "a1", "a3", "b1", "b2", "c2", "c4".
    auto iter = db->NewIterator(ReadOptions());
    iter->Seek("a1");        // iter->Key() == "a1";
    iter->Seek("a3");        // iter->Key() == "a3";
    iter->SeekForPrev("c4"); // iter->Key() == "c4";
    iter->SeekForPrev("c3"); // iter->Key() == "c2";
```

SeekForPrev()的实现类似于：
```    
    Seek(target); 
    if (!Valid()) {
      SeekToLast();
    } else if (key() > target) { 
      Prev(); 
    }
```

## Tailing Iterator
从Rocksdb 2.7开始，Rocksdb提供了一个特殊类型的称为Tailing Iterator迭代器，专门用于在新数据写入数据库之后尽早读取到该新数据。主要特性如下：

Tailing Iterator创建的时候不会创建任何快照，因此它可以用于读取最新的添加到数据库中的数据（常规迭代器则不会看到任何在它创建之后添加到数据库中的数据）。

Tailing Iterator主要用于优化顺序读 -- 它可以避免很多情况下可能存在的在SST files或者immutable memtable中的昂贵的寻找。

在创建迭代器的时候将选项ReadOptions::tailing设置为true，该迭代器就会成为Tailing Iterator。当前Tailing Iterator只支持顺序迭代（也就是说Prev()和SeekToLast()不被支持）。

并非所有新的数据都一定对Tailing Iterator都是可见的，在Tailing Iterator上执行Seek()或者SeekToFirst()可以被视为创建一个隐示的snapshot -- 任何该snapshot之后写入的数据，可能是可见的，但不保证是可见的。

### Tailing Iterator details

一个Tailing Iterator提供了两个内部迭代器的合并后的视图：
- 一个mutable iterator，用于获取当前memtable中的内容；
- 一个immutable iterator，用于从SST files和immutable memtables中读取数据。

这两个内部迭代器在创建的时候都采用了KMaxSequenceNumber，有效禁用了基于内部序列号的过滤（effectively disabling filtering based on internal sequence numbers），使得这两个迭代器可以访问那些在创建迭代器之后插入的记录。另外，每一个Tailing Iterator都跟踪数据库状态的变化（比如memtable flushes和compactions），并且在数据库状态改变的时候，使得内部迭代器失效。通过这些确保Tailing Iterator 总是最新的。

# 迭代器实现
[Iterator Implementation](https://github.com/facebook/rocksdb/wiki/Iterator-Implementation)