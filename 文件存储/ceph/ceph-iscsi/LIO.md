# 提纲
[toc]

# LIO Admin Manual
[LIO Admin Manual](http://www.linux-iscsi.org/Doc/LIO%20Admin%20Manual.pdf)一文中对LIO的targetcli部分有个比较好的讲解，从中可以比较好的理解LIO中各对象（Backstore，Target，Target Portal Group，Portals，Network Portal，luns，node ACLs，mapped luns）之间的关系。

# LIO(target_core_*)
在login之后会启动内核线程，线程执行函数为iscsi_target_rx_thread：

```
iscsi_target_do_login
    - iscsi_target_do_tx_login_io
        - iscsit_start_kthreads
            - iscsi_target_rx_thread
                - conn->conn_transport->iscsit_get_rx_pdu(conn)
```

                
对于iscsi类型的transport来说，就是iscsi_target_transport.iscsit_get_rx_pdu：

```
iscsi_target_transport.iscsit_get_rx_pdu
    - iscsit_get_rx_pdu
        - iscsi_target_rx_opcode
```

        
下面会分ISCSI_OP_SCSI_CMD类型的cmd和ISCSI_OP_SCSI_DATA_OUT类型的cmd，关于这两种类型的cmd，可以参考[这里](http://www.docin.com/p-1400817959.html) P14-P16.

```
对于ISCSI_OP_SCSI_CMD类型的cmd，进入：
            - iscsit_handle_scsi_cmd
                - iscsit_setup_scsi_cmd
                    - transport_init_se_cmd
                    - target_setup_cmd_from_cdb
                        - dev->transport->parse_cdb(cmd); //这里dev->transport虽然名为transport，实则为target_backend_ops
                    - list_add_tail(&cmd->i_conn_node, &conn->conn_cmd_list); //生成的cmd会被添加到conn->conn_cmd_list中
                - iscsit_process_scsi_cmd   
                    - transport_generic_new_cmd 分配cmd相关的内存
                
对于ISCSI_OP_SCSI_DATA_OUT类型的cmd，进入：
            - iscsi_target_rx_opcode
                - iscsit_handle_data_out
                    - iscsit_check_dataout_hdr //在这里会从conn->conn_cmd_list中找到相应的cmd(在处理ISCSI_OP_SCSI_CMD类型的cmd的时候添加到链表中的)
                    - iscsit_check_dataout_payload
                        - target_execute_cmd
                            - __target_execute_cmd
                                - cmd->execute_cmd(cmd) //cmd->execute_cmd是在iscsit_handle_scsi_cmd -> ... -> sbc_parse_cdb逻辑中设置的，这里调用之
```

对于ISCSI_OP_SCSI_CMD类型的cmd和iblock类型的backend来说，parse_cdb对应的是iblock_ops.parse_cdb：                   

```
iblock_ops.parse_cdb
    - iblock_parse_cdb
        - sbc_parse_cdb
            - cmd->execute_cmd = sbc_execute_rw(for READ_6, READ_10, READ_12, READ_16, WRITE_6, WRITE_10, WRITE_VERIFY, WRITE_12, WRITE_16, XDWRITEREAD_10, VARIABLE_LENGTH_CMD等)
```

对于ISCSI_OP_SCSI_DATA_OUT类型的cmd和READ_6, READ_10, READ_12, READ_16, WRITE_6, WRITE_10, WRITE_VERIFY, WRITE_12, WRITE_16, XDWRITEREAD_10, VARIABLE_LENGTH_CMD等操作来说，cmd->execute_cmd对应的是sbc_execute_rw，对于iblock类型的backend对应的是sbc_operations是iblock_sbc_ops：

```
sbc_execute_rw
    - cmd->protocol_data->execute_rw
        - iblock_sbc_ops.execute_rw
            - iblock_execute_rw
```

相应的，对于ISCSI_OP_SCSI_CMD类型的cmd和rbd类型的backend来说，parse_cbd和sbc_execute_rw分别会进入tcm_rbd_ops.parse_cdb和tcm_rbd_execute_rw：

```
tcm_rbd_ops.parse_cdb
    - tcm_rbd_parse_cdb
        - sbc_parse_cdb
            - cmd->execute_cmd = sbc_execute_rw(for READ_6, READ_10, READ_12, READ_16, WRITE_6, WRITE_10, WRITE_VERIFY, WRITE_12, WRITE_16, XDWRITEREAD_10, VARIABLE_LENGTH_CMD等)
            
sbc_execute_rw
    - cmd->protocol_data->execute_rw
        - tcm_rbd_sbc_ops.execute_rw
            - tcm_rbd_execute_rw            
```

# target_core_rbd.c中关于pr的操作
1. pr info保存在rdb镜像的扩展属性里面，所有的register信息保存在一起，key为pr_info，value为所有register的信息；

2. register的时候会查找是否有关于该path的register信息，
    如果没有，则append新注册的key到register信息中；
    如果有，且新注册的key不为0，则替换旧注册的key；
    如果有，且新注册的key为0，则清除注册的旧的key，且如果同时满足下列条件，则从reservation holder中移除该key：
        该key是reservation holder；
        当前reservation type不是PR_TYPE_WRITE_EXCLUSIVE_ALLREG | PR_TYPE_EXCLUSIVE_ACCESS_ALLREG，或者该key是当前唯一注册的key；

3. reserve过程中，首先从rbd镜像的扩展属性中解析出pr信息；
    其次，检查是否注册过，没有注册过则报错，如果注册了但是register的key和reserve的key不一样，则报错；
    再次，如果已经有reserve信息，则检查reserve信息是否和当前reserve相关的配置相匹配：
        如果它不是reserver holder，则返回TCM_RESERVATION_CONFLICT；
        如果reserve type跟当前reserve相关的配置不一致，则返回TCM_RESERVATION_CONFLICT；
        返回上层调用；
    最后，到这里说明之前没有任何reserve信息，设置内存中的reserve信息，并最终反映到rbd镜像的扩展属性中；
    
4. preempt过程如下（当前RBD backend不支持preempt and abort）：
    tcm_rbd_gen_it_nexus - 通过cmd->se_sess获取I_T nexus；
    tcm_rbd_pr_info_get - 从rbd镜像的扩展属性中获取当前的persistent reservation相关的信息；
    查找想要抢占的I_T nexus是否注册过，如果没有注册过，则返回，否则继续；
    
    
                
        
# 关于rtslib中某些类的说明

```
class StorageObject(CFSNode):
    '''
    This is an interface to storage objects in configFS. A StorageObject is
    identified by its backstore and its name.
    '''
    
# Used to convert either dirprefix or plugin to the SO. Instead of two
# almost-identical dicts we just have some duplicate entries.
so_mapping = {
    "pscsi": PSCSIStorageObject,
    "rd_mcp": RDMCPStorageObject,
    "ramdisk": RDMCPStorageObject,
    "fileio": FileIOStorageObject,
    "iblock": BlockStorageObject,
    "block": BlockStorageObject,
    "user": UserBackedStorageObject,
}

class Target(CFSNode):
    '''
    This is an interface to Targets in configFS.
    A Target is identified by its wwn.
    To a Target is attached a list of TPG objects.
    '''
    
class TPG(CFSNode):
    '''
    This is a an interface to Target Portal Groups in configFS.
    A TPG is identified by its parent Target object and its TPG Tag.
    To a TPG object is attached a list of NetworkPortals. Targets without
    the 'tpgts' feature cannot have more than a single TPG, so attempts
    to create more will raise an exception.
    '''

class LUN(CFSNode):
    '''
    This is an interface to RTS Target LUNs in configFS.
    A LUN is identified by its parent TPG and LUN index.
    '''

class NetworkPortal(CFSNode):
    '''
    This is an interface to NetworkPortals in configFS.  A NetworkPortal is
    identified by its IP and port, but here we also require the parent TPG, so
    instance objects represent both the NetworkPortal and its association to a
    TPG. This is necessary to get path information in order to create the
    portal in the proper configFS hierarchy.
    '''
```



# tips
1. 为了支持ceph rbd，LIO中专门提供了：RBDBackstore。
2. petasan中将每一个ceph rbd镜像都看作是一个target？
3. 


