# 提纲
[toc]

# [CLUSTERED SCSI TARGET USING RBD](https://tracker.ceph.com/projects/ceph/wiki/Clustered_SCSI_target_using_RBD)

# [Ceph RBD and iSCSI](https://ceph.com/planet/ceph-rbd-and-iscsi/)

# [*****Ceph iscsi gateway](http://docs.ceph.com/docs/master/rbd/iscsi-overview/)

1. 本文种讲到为什么ceph中采用LIO而非TGT：
```
After a couple of months of testing by community users, it came out that this early prototype was lacking features and robustness, thus making it unsuitable for enterprises demand.
Enterprises require advanced features such as high availability with multipath, persistent reservation, low latency, high throughput, parallelism and strong authentication methods.
All those things could not be achieved by TGT.
As a result, work has now started on the LIO target in the Linux kernel to provide HA capabilities.
```

2. 本文中讲述了如何通过LIO + RBD实现高可用，错误处理和认证机制等。
3. 本文中讲述了LIO + RBD实现ISCSI的缺陷，并提出了基于tcmu + librbd的替代方案：

```
The current LIO iblock + krbd iSCSI implementation has some limitations:
    Limited to Active/Passive (ALUA active optimized/active non-optimized) because of RBD exclusive lock feature
    Eventual support for PGRs will require many new callouts and hooks into the block layer
    Kernel development only, so unless you’re Red Hat, SUSE or someone constantly upgrading your Kernel to the last one, it’s tough to deliver a solution

That is why developers are currently investigating switching to an LIO tcmu + librbd iSCSI.
TCM is another name of LIO, which is kernel space.
TCMU is an userland implementation for TCM (thanks Andy Grover!).
TCMU is the LIO target_core_user kernel module that passes SCSI commands to userspace and tcmu-runner is the userspace component that processes those commands and passes them to drivers for device specific execution.
tcmu-rbd is the tcmu-runner driver that converts SCSI commands to ceph/rbd requests.

Using a userspace component brings numerous benefits like:
    No kernel code needed
    Easier to ship the software
    Focus on your own backend, in our case RBD
```

4. 网上也说到了LIO +RBD实现ISCSI的有点和缺点：

```
LIO 也即 Linux-IO，是目前 GNU/Linux 内核自带的 SCSI target 框架（自 2.6.38版本开始引入，真正支持 iSCSI 需要到 3.1 版本） ，对 iSCSI RFC 规范的支持非常好，包括完整的错误恢复都有支持。整个 LIO 是纯内核态实现的，包括前端接入和后端存储模块，为了支持用户态后端，从内核 3.17 开始引入用户态后端支持，即 TCMU(Target Core Module in Userspace)
优点：
1）支持较多传输协议
2）代码并入linux内核，减少了手动编译内核的麻烦
3）提供了python版本的编程接口rtslib
4）LIO在不断backport SCST的功能到linux内核，社区的力量是强大的
5）LIO也支持一些SCST没有的功能，如LIO 还支持“会话多连接”（MC/S）
6）LIO支持最高级别的ERL

缺点：
不支持AEN，所以target状态发生变化时，只能通过IO或者用户手动触发以检测处理变化
结构相对复杂，二次开发成本较高
工作在内核态，出现问题会影响其他程序的运行
```

# [Ceph iSCSI Gateway Demo安装配置](https://wenku.baidu.com/view/1b3f3e973b3567ec112d8a44.html)

1. 本文对比了各种SCSI Target，比如TGT，SCST，LIO等，并最终总结了他们为什么选择SCST的原因：

```
总的来说，stgt构建一个规模不是很大的iSCSI target。
LIO支持FC、SRP等协议，但性能、稳定性不是很好。
SCST适合构建一个企业级的高性能、高稳定性的存储方案，各大存储服务提供商都是基于SCST。因此我们选择了SCST。
```

2. 本文讲述了多路径软件的作用及其常见的多路径软件：

```
多路径的主要功能就是和存储设备一起配合实现如下功能：
1.故障的切换和恢复
2.IO流量的负载均衡
3.磁盘的虚拟化

业界比较常见的多路径功能软件有 EMC 的PowerPath， IBM 的 SDD，日立的 Hitachi Dynamic Link Manager 和广泛使用的 linux 开源软件 multipath和windows server的MPIO。
```

3. Ceph-iscsi gateway demo 安装

4. Ceph-iscsi gateway demo 配置


# [基于ceph-RBD的iSCSI-target实现分析](https://wenku.baidu.com/view/071299907fd5360cbb1adb91.html)


# [tcmu+librbd 导出iscsi卷](https://my.oschina.net/hanhanztj/blog/842869)


# [Linux LIO 与 TCMU 用户空间透传](http://www.itdks.com/dakalive/detail/6142)

本文讲的挺好的，分别讲述了以下3个部分：

```
Overview of Ceph RBD iscsi
LIO, TCMU and passthrough
The status of TCMU-runner
```

# [Linux-IO Target介绍](https://www.ibm.com/developerworks/community/blogs/5144904d-5d75-45ed-9d2b-cf1754ee936a/entry/linux_io_target%25e4%25bb%258b%25e7%25bb%258d_%25e4%25b8%2580?lang=en_us)

本文主要包括以下两点：
1. Linux-IO的iSCSI Target架构（摘自LIO官网）
2. targetcli介绍及其使用

# [Ceph-users中关于iscsi multipath和rbd exclusive lock的讨论](https://www.mail-archive.com/ceph-users@lists.ceph.com/msg44778.html)

```
1. PetaSAN为了实现active/active，在suse kernel中打了一个patch：target_core_rbd。

2. Under load balanced multipathing(multiple iscsi gateways) of iscsi backed with RBD images. Should I disable exclusive lock feature? What if I don't disable that feature?

if using LIO/TGT + krbd, then shouldn't do active/active multipathing, if you have the lock enabled then it bounces between paths for each IO and will be slow. But, if you don't have it enabled then you can end up with staled IO overwriting current data.

3. Is it safe to use active/active multipath If use suse kernel with target_core_rbd?
A cross-gateway failover race-condition similar to what Mike described
is currently possible with active/active target_core_rbd. It's a corner
case that is dependent on a client assuming that unacknowledged I/O has
been implicitly terminated and can be resumed via an alternate path,
while the original gateway at the same time issues the original request
such that it reaches the Ceph cluster after differing I/O to the same
region via the alternate path.
It's not something that we've observed in the wild, but is nevertheless
a bug that is being worked on, with a resolution that should also be
usable for active/active tcmu-runner.

For example, Write operation (A) is sent to gateway X who cannot access the Ceph
cluster so the IO is queued. The initiator's multipath layer times out
and resents write operation (A) to gateway Y, followed by write
operation (A') to gateway Y. Shortly thereafter, gateway X is able to
send its delayed write operation (A) to the Ceph cluster and
overwrites write operation (A') -- thus your data went back in time.


Here is a simple but long example of the problem. Sorry for the length,
but I want to make sure people know the risks.

You have 2 iscsi target nodes and 1 iscsi initiator connected to both
doing active/active over them.

To make it really easy to hit, the iscsi initiator should be connected
to the target with a different nic port or network than what is being
used for ceph traffic.

(1). Prep the data. Just clear the first sector of your iscsi disk. On the
initiator system do:

dd if=/dev/zero of=/dev/sdb count=1 ofile=direct

(2). Kill the network/port for one of the iscsi targets ceph traffic. So
for example on target node 1 pull its cable for ceph traffic if you set
it up where iscsi and ceph use different physical ports. iSCSI traffic
should be unaffected for this test.

(3). Write some new data over the sector we just wrote in #1. This will
get sent from the initiator to the target ok, but get stuck in the
rbd/ceph layer since that network is down:

dd if=somefile of=/dev/sdb count=1 ofile=direct ifile=direct

(4). The initiator's eh timers will fire and that will fail and will the
command will get failed and retired on the other path. After that dd in
#(3) completes run:

dd if=someotherfile of=/dev/sdb count=1 ofile=direct ifile=direct

This should execute quickly since it goes through the good iscsi and
ceph path right away.

(5). Now plug the cable back in and wait for maybe 30 seconds for the
network to come back up and the stuck command to run.

(6). Now do

dd if=/dev/sdb of=somenewfile count=1 ifile=direct ofile=direct

The data is going to be the data sent in step 3 and not the new data in
step (4).


4. Is it safe to use active/passive multipath with krbd with exclusive lock for lio/tgt/scst/tcmu?
No. We tried to use lio and krbd initially, but there is a issue where
IO might get stuck in the target/block layer and get executed after new
IO. So for lio, tgt and tcmu it is not safe as is right now. We could
add some code tcmu's file_example handler which can be used with krbd so
it works like the rbd one.

5. One other case we have been debating about is if krbd/librbd is able to
put the ceph request on the wire but then the iscsi connection goes
down, will the ceph request always get sent to the OSD before the
initiator side failover timeouts have fired and it starts using a
different target node. If krbd/librbd is able to put the ceph request on the wire, then that could
cause data corruption in the active/passive case too, right?

In general, yes. However, that's why the LIO/librbd approach uses the
RBD exclusive-lock feature in combination w/ Ceph client blacklisting
to ensure that cannot occur. Upon path failover, the old RBD client is
blacklisted from the Ceph cluster to ensure it can never complete its
(possible) in-flight writes.

6. So for now only suse kernel with target_rbd_core and tcmu-runner can run active/passive multipath safely?

Negative, the LIO / tcmu-runner implementation documented here [1] is
safe for active/passive.

7. I think the stuck io get excuted cause overwrite
problem can happen with both active/active and active/passive. What makes the active/passive safer than active/active?

As discussed in this thread, for active/passive, upon initiator
failover, we used the RBD exclusive-lock feature to blacklist the old
"active" iSCSI target gateway so that it cannot talk w/ the Ceph
cluster before new writes are accepted on the new target gateway.


8. What mechanism should be implement to avoid the problem with active/passive and active/active multipath?

Active/passive it solved as discussed above. For active/active, we
don't have a solution that is known safe under all failure conditions.
If LIO supported MCS (multiple connections per session) instead of
just MPIO (multipath IO), the initiator would provide enough context
to the target to detect IOs from a failover situation.

注释：关于LIO MCS，core-iscsi支持，但是open-iscsi不支持？参考[这里](http://linux-iscsi.org/wiki/Multiple_Connections_per_Session).

9. Petasan say they can do active/active iscsi with patched suse kernel. Is it the truth?

We are not currently handling these corner cases. We have not hit this in practice but will work on it. We need to account for in-flight time early in the target stack before reaching krbd/tcmu.
/Maged

10. How the old target gateway is blacklisted? Is it a feature of the target gateway(which can support active/passive multipath) should provide or is it only by rbd excusive lock?

When the newly active target gateway breaks the lock of the old target
gateway, that process will blacklist the old client [1].

In general, yes -- but blacklist on lock break has been part of
exclusive-lock since the start. I am honestly not just making this up,
this is how it works.
```

# [iSCSI active/active stale io guard](https://www.spinics.net/lists/ceph-devel/msg40590.html)

# [Practise of ISCSI target for Ceph](http://geek.csdn.net/news/detail/125662)

# [企业级ceph之路-iSCSI实践与优化](https://max.book118.com/html/2017/1229/146392479.shtm)
本文很好，杉岩数据CTO写的。

# [Ceph分布式系统的ISCSI高可用集群](https://max.book118.com/html/2017/0722/123614109.shtm)

# [基于iSCSI存储集群的设计与实现](http://www.ixueshu.com/document/d8915f732710eb9d318947a18e7f9386.html)

# [iSCSI高可用存储解决方案的设计和实现](http://www.ixueshu.com/document/1f8b2e9f4c5240e6.html)

# [http://www.kernsafe.com/tech/iStorage-Server/iStorage-Server-High-Availability-iSCSI-SAN-for-Windows-2012-Clustering-Hyper-V.pdf](http://www.kernsafe.com/tech/iStorage-Server/iStorage-Server-High-Availability-iSCSI-SAN-for-Windows-2012-Clustering-Hyper-V.pdf)

# [http://www.kernsafe.com/tech/iStorage-Server/iStorage-Server-High-Availability-iSCSI-SAN-for-Windows-2012-Clustering-Hyper-V.pdf](http://www.kernsafe.com/tech/iStorage-Server/iStorage-Server-High-Availability-iSCSI-Target-SAN-for-VMWare-ESX-ESXi.pdf)

# [ISCSi_target_for_ceph](http://www.docin.com/p-1951744143.html)

# [看Ceph如何实现原生的ISCSI](http://blog.51cto.com/devingeng/2125656)



