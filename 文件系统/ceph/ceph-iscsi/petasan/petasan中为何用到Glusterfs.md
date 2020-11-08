[这里](http://www.petasan.org/forums/?view=thread&id=59)可知，Glusterfs只适用于在不同节点之间共享统计图表？
just to add a little on the above,  the GlusterFS errors you mentioned are intriguing. GlusterFS is a system totally separate from Ceph and iSCSI and we use it internally to share the statistic graphs we draw amount the nodes. A failure there is another indication that the system is either over loaded or the network is not reliable, GlusterFS nodes talk over the Backend 1 network.

