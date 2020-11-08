# SUMMARY
The goal of this project is to modify the Linux target layer, LIO, to be able to support active/active access to a device across multiple nodes running LIO. The changes to LIO are being done in a generic way to allow other cluster aware devices to be used, but our focus is on using RBD.

# OWNERS
Mike Christie (Red Hat)

# INTERESTED PARTIES
Name (Affiliation)

# CURRENT STATUS
There are many methods to configure LIO's iSCSI target for High Availability (HA). None support active/active, and most open source implementations do not support distributed SCSI Persistent Group Reservations.

# DETAILED DESCRIPTION
In order for some operating systems to be able to access RBD they must go through a SCSI target gateway. Support for RBD with target layers like LIO and TGT exist today, but the HA implementations are lacking features, difficult to use, or only support one transport like iSCSI. To resolve these issues, **we are modifying LIO, so that it can be run on multiple nodes and provide SCSI active-optimized access through all ports on all nodes at the same time**.

**There are several areas where active/active support in LIO requires distributed meta data and/or locking**: SCSI task management/Unit Attention/PREEMPT AND ABORT handling, COMPARE_AND_WRITE support, Persistent Group Reservations, and INQUIRY/ALUA/discovery/setup related commands and state.

## SCSI task management (TMF) / Unit Attention (UA) / PREEMPT AND ABORT handling:
When a initiator cannot determine the state of a device or commands running on the device, it will send TMFs like LOGICAL UNIT RESET. Depending on the SCSI settings used (TAS, QERR, TST), requests like this may require actions to be taked on all LIO nodes. For example, running commands might need to be aborted, notifications like Unit Attentions must be sent, etc.

Other non TMF requests like PERSISTENT RESERVE IN - PREEMPT AND ABORT may also require commands to be abort on remote nodes.

To synchronize TMF execution across nodes, the ceph watch notify feature will be used. The initial patches for this were posted on ceph-devel here:

http://thread.gmane.org/gmane.comp.file-systems.ceph.devel/24553

The current version with fixes and additions by Douglas Fuller can be found here:

https://github.com/fullerdj/ceph-client/tree/wip-djf-cls-lock

Status:
We are currently modifying the block layer to support task management requests, so LIO's iblock backend module can call into Low Level Drivers (LLD) like krbd to perform driver specific actions.

### annotation
TMF：Task Management Functions
The Task Management functions provide an initiator with a way to explicitly control the execution of one or more Tasks (SCSI and iSCSI tasks). The Task Management functions are (for a more detailed description of SCSI task management see [SAM2]):
    1    ABORT TASK - aborts the task identified by the Referenced Task Tag field.
    2    ABORT TASK SET - aborts all Tasks issued by this initiator on the Logical Unit.
    3    CLEAR ACA - clears the Auto Contingent Allegiance condition.
    4    CLEAR TASK SET - Aborts all Tasks (from all initiators) for the Logical Unit.
    5    LOGICAL UNIT RESET
    6    TARGET WARM RESET
    7    TARGET COLD RESET
    8    TASK REASSIGN - reassign connection allegiance for the task identified by the Initiator Task Tag field on this         connection, thus resuming the iSCSI exchanges for the task

Unit Attention:
SCSI协议采用Unit Attention的方式让存储告诉主机，存储发生变化，需要主机查询。

[Unit Attention Condition](http://rakesh-storage-scsi.blogspot.com/2013/02/unit-attention-condition.html)

PREEMPT AND ABORT:


## COMPARE_AND_WRITE (CAW) support:

CAW is a SCSI command used by ESX to perform finely grained locking. The execution of CAW requires that the handler atomically read N blocks of data, compare them to a buffer passed in with the command, then if matching write N blocks of data. To guarantee this operation is done atomically, LIO uses a standard linux kernel mutex. For multiple node active/active support, we have proposed to pass this request to the backing storage. This will allow the backing storage to utilize its own locking and serialization support, and LIO will not need to use a clustered lock like DLM.

Patches for passing COMPARE_AND_WRITE directly to the backing store have been sent upstream for review:
http://www.spinics.net/lists/target-devel/msg07823.html

The current patches along with ceph/rbd support are here:

https://github.com/mikechristie/linux-kernel/commits/ceph

Status:
The request/bio operation patches are waiting to be merged. The ceph/rbd patches will be posted for review when that is done.

## Persistent Group Reservations (PGR):

PGRs allow a initiator to control access to a device. This access information needs to be distributed across all nodes and can be dynamically updated while other commands are being processed.

David Disseldorp has implemented ceph/rbd PGR support here:

https://git.samba.org/?p=ddiss/linux.git;a=shortlog;h=refs/heads/target_rbd_pr_sq_20160126

Status:
This code is now being ported to the upstream linux kernel reservation API added in this commit:

https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/block/ioctl.c?id=bbd3e064362e5057cc4799ba2e4d68c7593e490b

When this is completed, LIO will call into the iblock backend which will then call rbd's pr_ops.

### annotation


## Device state and configuration:

SUSE's lrbd package will manage configuration:

https://github.com/SUSE/lrbd

Status:

This currently only supports iSCSI. The LIO and ceph/rbd modifications are not tied to specific SCSI transports. lrbd will be modified to support Fibre Channel, SRP, etc.

## Extra:

The most common use will likely be with VShpere/ESX. In this initial version most VAAI functions will be supported. VVOL/VASA support is being investigated for the next release.

VAAI:

Delete/UNAMP - already completed and upstream.

ATS/COMPARE_AND_WRITE - completed and waiting posting/review. Current patches:
https://github.com/mikechristie/linux-kernel/tree/ceph

Zero/WRITE_SAME - completed and waiting posting/review. Current patches:
https://github.com/mikechristie/linux-kernel/tree/ceph.

Clone/XCOPY/EXTENDED_COPY - This operation can be done in chunks of 4 - 16 MBs, but ceph/rbd's cloning function works at the device level. Currently, LIO will export that we support this, but the target node will have to do a read and write of the data, so we do not have true offloading.

XCOPY/EXTENDED_COPY is being implemented in the upstream kernel. For the next version, we will investigate ceph/rbd support. We are also looking into the VASA cloneVirtualVolume operation.

VVOL/VASA:

This will not be completed in this version.

WORK ITEMS
Coding tasks
Task 1
Task 2
Task 3
Build / release tasks
Task 1
Task 2
Task 3
Documentation tasks
Task 1
Task 2
Task 3
Deprecation tasks
Task 1
Task 2
Task 3
Powered by Redmine © 2006-2016 Jean-Philippe Lang
