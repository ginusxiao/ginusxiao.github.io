# 提纲
[toc]

# 相关文章
[iSCSI](http://linux-iscsi.org/wiki/ISCSI)

[Anatomy of the Linux SCSI subsystem](http://blog.csdn.net/hshl1214/article/details/8836164)

[Linux SCSI 子系统剖析](https://www.ibm.com/developerworks/cn/linux/l-scsi-subsystem/)

[LinuxIO with Sun COMSTAR and other Linux open-source SCSI targets comparasion](http://www.linux-iscsi.org/wiki/Features)
本文中对LIO和Sub COMSTAR以及其它Linux SCSI targets（比如SCST、IET和STGT）进行了全方位对比。

[SCSI](https://en.wikipedia.org/wiki/SCSI#SCSI_command_protocol)
本文中关于SCSI commands的讲解值得一看，见“SCSI command protocol”这一小节。

[iSCSI协议](https://wenku.baidu.com/view/318b04846c85ec3a86c2c541.html)

[IP Storage Protocol: ISCSI](https://www.snia.org/sites/default/education/tutorials/2011/spring/networking/HufferdJohn-IP_Storage_Protocols-iSCSI.pdf)

[MC/S VS MPIO](http://scst.sourceforge.net/mc_s.html)

# Persistent Reservation
[SCSI Reservation - IBM](https://books.google.com/books?id=X9skDwAAQBAJ&pg=PA417&lpg=PA417&dq=persistent+reservations+preempt&source=bl&ots=e11A_p-7lN&sig=lETekiphg8U0HBaN__ygnnZMlxA&hl=zh-CN&sa=X&ved=0ahUKEwjBtoO7sOjaAhVGyWMKHRzOAP84ChDoAQgtMAE#v=onepage&q&f=false)

[PNFS SCSI Reservation](https://events.static.linuxfound.org/sites/events/files/slides/Linux-Vault-pNFS-MDS-E-Series_v2.pdf)

[Block layer support for Persistent Reservations](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/block/pr.txt?id=bbd3e064362e5057cc4799ba2e4d68c7593e490b)

[Persistent Reservations wiki](http://linux-iscsi.org/wiki/Persistent_Reservations)

[存储SCSI锁解读：Windows Cluster篇](http://www.dostor.com/article/2013-01-08/97518.shtml)

[What is Persistent Reservation from OS Perspective](https://blogs.msdn.microsoft.com/scw/2015/04/19/what-is-persistent-reservation-from-os-perspective/)

[***CONFIGURING FENCING USING SCSI PERSISTENT RESERVATIONS](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/configuration_example_-_fence_devices/scsi_configuration)

# tips
## 关于iscsi target portal group的理解
A target portal group is a set of network portals within an iSCSI node over which an iSCSI session is conducted.

In a target, a network portal is identified by its IP address and listening TCP port. For storage systems, each network interface can have one or more IP addresses and therefore one or more network portals. A network interface can be an Ethernet port, virtual local area network (VLAN), or interface group.

The assignment of target portals to portal groups is important for two reasons:
The iSCSI protocol allows only one session between a specific iSCSI initiator port and a single portal group on the target.
All connections within an iSCSI session must use target portals that belong to the same portal group.
By default, Data ONTAP maps each Ethernet interface on the storage system to its own default portal group. You can create new portal groups that contain multiple interfaces.

You can have only one session between an initiator and target using a given portal group. To support some multipath I/O (MPIO) solutions, you need to have separate portal groups for each path. Other initiators, including the Microsoft iSCSI initiator version 2.0, support MPIO to a single target portal group by using different initiator session IDs (ISIDs) with a single initiator node name.

## How iSCSI works with HA pairs
HA pairs provide high availability because one system in the HA pair can take over if its partner fails. During failover, the working system assumes the IP addresses of the failed partner and can continue to support iSCSI LUNs.

The two systems in the HA pair should have identical networking hardware with equivalent network configurations. The target portal group tags associated with each networking interface must be the same on both systems in the configuration. This ensures that the hosts see the same IP addresses and target portal group tags whether connected to the original storage system or connected to the partner during failover.


## [NetApp Simple HA pairs with iSCSI](https://library.netapp.com/ecmdocs/ECMP1368845/html/GUID-686321A3-8699-41DC-A375-0331D13A9B3A.html)

## Target portal group management
A target portal group is a set of one or more storage system network interfaces that can be used for an iSCSI session between an initiator and a target. A target portal group is identified by a name and a numeric tag. If you want to have multiple connections per session across more than one interface for performance and reliability reasons, then you must use target portal groups.

Note: If you are using MultiStore, you can also configure non-default vFiler units for target portal group management based on IP address.
For iSCSI sessions that use multiple connections, all of the connections must use interfaces in the same target portal group. Each interface belongs to one and only one target portal group. Interfaces can be physical interfaces or logical interfaces (VLANs and interface groups).

Prior to Data ONTAP 7.1, each interface was automatically assigned to its own target portal group when the interface was added. The target portal group tag was assigned based on the interface location and could not be modified. This works fine for single-connection sessions.

You can explicitly create target portal groups and assign tag values. If you want to increase performance and reliability by using multi-connections per session across more than one interface, you must create one or more target portal groups.

Because a session can use interfaces in only one target portal group, you might want to put all of your interfaces in one large group. However, some initiators are also limited to one session with a given target portal group. To support multipath I/O (MPIO), you need to have one session per path, and therefore more than one target portal group.

When a new network interface is added to the storage system, that interface is automatically assigned to its own target portal group.

## iscsi中几个关键字
### InitialR2T
R2T表示"Ready to Transmit"，该关键字表示启用或者禁止R2T流控，如果启用，则initiator在发送任何数据前，必须等待一个R2T命令，默认为no。

### tpg_enabled_sendtargets
if set, the sendtargets discovery infomation will only include those target portal groups in 'enabled' state.

### [深入了解iSCSI的2种多路径访问机制](http://blog.51cto.com/rootking/476189)






