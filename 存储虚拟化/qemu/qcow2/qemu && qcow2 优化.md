# 参考资料
## qemu thread/event/aio
[Improving the QEMU Event Loop](http://www.linux-kvm.org/images/6/64/03x08-Aspen-Fam_Zheng-Improving_the_QEMU_Event_Loop.pdf)

```
还没看...
```

[Effective multi-threading in QEMU
](http://www.linux-kvm.org/images/1/17/Kvm-forum-2013-Effective-multithreading-in-QEMU.pdf)

```
还没看...
```

## virtio
[Virtio-blk Multi-queue Conversion and QEMU Optimization](https://www.linux-kvm.org/images/6/63/02x06a-VirtioBlk.pdf)

```
还没看...
```

[Virtio-blk Performance Improvement](https://www.linux-kvm.org/images/f/f9/2012-forum-virtio-blk-performance-improvement.pdf)
```
还没看...
```

## migration
[Fast Write	Protection](http://events.linuxfoundation.org/sites/events/files/slides/Guangrong-fast-write-protection.pdf)

```
本文是腾讯的大牛在Linux Foundation上演讲的文章，里面提到如何实现更快的写保护（Write protection）来实现更快的热迁移。
```

[KVM Live Migration Optimization](http://www.linux-kvm.org/images/b/b3/02x-09-Cedar-Liang_Li-KVMLiveMigrationOptimization.pdf)

```
还没看...
```

## qcow2 optimization
[Improving the performance of the qcow2 format](http://events.linuxfoundation.org/sites/events/files/slides/kvm-forum-2017-slides_1.pdf)

```
本文从以下角度优化qcow2性能，值得借鉴：
qcow2 L2 cache:
    Size and cleanup timer are configurable.
    Probably needs better defaults or configuration options.
    
L2 slices:
    Patches in the mailing list.
    COW with two I/O operations instead of five:
    Available in QEMU 2.10.
    
COW with preallocation instead of writing zeroes:
    Patches in the mailing list.
    
Subcluster allocation:
    RFC status. Requires changes to the on-disk format.
    
Metadata overlap checks:
    Slowest check optimized in QEMU 2.9.
    Other checks can be disabled manually if needed.
```

[Mitigating Sync Amplification for Copy-on-Write Virtual Disk](https://www.usenix.org/sites/default/files/conference/protected-files/fast16_slides_chen43.pdf)

```
本文是上海交大的学生和三星的员工共同在Fast16的一篇文章（在此不得不对较大的学生奉上双膝啊），首先阐述了Copy-on-Write类型的虚拟磁盘中同步放大（sync amplification）问题产生的原因，然后提出了三种优化措施，在减少sync操作的同时，确保数据一致性，并且对于IO密集型负载带来可观的性能提升。
```


