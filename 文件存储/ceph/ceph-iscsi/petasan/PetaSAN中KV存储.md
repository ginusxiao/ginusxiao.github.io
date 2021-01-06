# 提纲
[toc]

# Consul中的KV
```
root@petasan-node1:~# consul kv get -recurse PetaSAN/
PetaSAN/Cluster_info:{
    "backend_1_base_ip": "192.168.74.0",
    "backend_1_eth_name": "eth3",
    "backend_1_mask": "255.255.240.0",
    "backend_2_base_ip": "192.168.74.0",
    "backend_2_eth_name": "eth4",
    "backend_2_mask": "255.255.240.0",
    "bonds": [],
    "eth_count": 5,
    "iscsi_1_eth_name": "eth1",
    "iscsi_2_eth_name": "eth2",
    "jumbo_frames": [],
    "management_eth_name": "eth0",
    "management_nodes": [
        {
            "backend_1_ip": "192.168.74.241",
            "backend_2_ip": "192.168.74.251",
            "is_iscsi": true,
            "is_management": true,
            "is_storage": true,
            "management_ip": "192.168.76.251",
            "name": "petasan-node1"
        },
        {
            "backend_1_ip": "192.168.74.242",
            "backend_2_ip": "192.168.74.252",
            "is_iscsi": true,
            "is_management": true,
            "is_storage": true,
            "management_ip": "192.168.76.252",
            "name": "petasan-node2"
        }
    ],
    "name": "ceph"
}
PetaSAN/Config:{"email_notify_smtp_email": "", "email_notify_smtp_password": "", "email_notify_smtp_port": "", "email_notify_smtp_security": 1, "email_notify_smtp_server": "", "iqn_base": "iqn.2016-05.com.petasan", "iscsi1_auto_ip_from": "192.168.76.140", "iscsi1_auto_ip_to": "192.168.76.200", "iscsi1_subnet_mask": "255.255.240.0", "iscsi2_auto_ip_from": "192.168.74.140", "iscsi2_auto_ip_to": "192.168.74.200", "iscsi2_subnet_mask": "255.255.240.0"}
PetaSAN/Config/Files/etc/hosts:MTkyLjE2OC43Ni4yNTMgICBwZXRhc2FuLW5vZGUzCjEyNy4wLjAuMSAgIGxvY2FsaG9zdAoxOTIuMTY4Ljc2LjI1MSAgIHBldGFzYW4tbm9kZTEKMTkyLjE2OC43Ni4yNTIgICBwZXRhc2FuLW5vZGUyCg==
PetaSAN/Disks/00003:disk
PetaSAN/Disks/00003/1:petasan-node2
PetaSAN/Disks/00003/2:petasan-node1
PetaSAN/Disks/00003/3:
PetaSAN/Disks/00003/4:
PetaSAN/Disks/00004:disk
PetaSAN/Disks/00004/1:petasan-node1
PetaSAN/Disks/00004/2:petasan-node3
PetaSAN/Disks/00004/3:petasan-node3
PetaSAN/Disks/00004/4:petasan-node2
PetaSAN/Disks/00005:disk
PetaSAN/Disks/00005/1:petasan-node3
PetaSAN/Disks/00005/2:petasan-node3
PetaSAN/Disks/00005/3:petasan-node1
PetaSAN/Disks/00005/4:petasan-node3
PetaSAN/Disks/00006:disk
PetaSAN/Disks/00006/1:
PetaSAN/Disks/00006/2:petasan-node2
PetaSAN/Disks/00006/3:
PetaSAN/Disks/00006/4:petasan-node3
PetaSAN/Disks/00006/5:petasan-node1
PetaSAN/Disks/00006/6:
PetaSAN/Disks/00007:disk
PetaSAN/Disks/00007/1:petasan-node1
PetaSAN/Disks/00007/2:
PetaSAN/Disks/00007/3:petasan-node3
PetaSAN/Disks/00007/4:petasan-node2
PetaSAN/Nodes/petasan-node1:{
    "backend_1_ip": "192.168.74.241",
    "backend_2_ip": "192.168.74.251",
    "is_iscsi": true,
    "is_management": true,
    "is_storage": true,
    "management_ip": "192.168.76.251",
    "name": "petasan-node1"
}
PetaSAN/Nodes/petasan-node2:{
    "backend_1_ip": "192.168.74.242",
    "backend_2_ip": "192.168.74.252",
    "is_iscsi": true,
    "is_management": true,
    "is_storage": true,
    "management_ip": "192.168.76.252",
    "name": "petasan-node2"
}
PetaSAN/Nodes/petasan-node3:{
    "backend_1_ip": "192.168.74.243",
    "backend_2_ip": "192.168.74.253",
    "is_iscsi": true,
    "is_management": true,
    "is_storage": true,
    "management_ip": "192.168.76.253",
    "name": "petasan-node3"
}
PetaSAN/Services/ClusterLeader:
PetaSAN/Sessions/01f1b0c5-82c6-4095-b46e-779d24031f85/_exp:2018-04-11 12:31:07.177802
PetaSAN/Sessions/01f1b0c5-82c6-4095-b46e-779d24031f85/_permanent:True
PetaSAN/Sessions/01f1b0c5-82c6-4095-b46e-779d24031f85/role_id:1
PetaSAN/Sessions/01f1b0c5-82c6-4095-b46e-779d24031f85/user:admin
PetaSAN/Sessions/038cb0df-e291-4e4f-aa15-f19b50963c13/_exp:2018-04-12 02:26:44.815646
PetaSAN/Sessions/038cb0df-e291-4e4f-aa15-f19b50963c13/_permanent:True
PetaSAN/Sessions/038cb0df-e291-4e4f-aa15-f19b50963c13/role_id:1
PetaSAN/Sessions/038cb0df-e291-4e4f-aa15-f19b50963c13/user:admin
PetaSAN/Sessions/1fabf2dc-b0b3-4099-8311-ef4255c85ead/_exp:2018-04-12 01:33:26.899100
PetaSAN/Sessions/1fabf2dc-b0b3-4099-8311-ef4255c85ead/_permanent:True
PetaSAN/Sessions/280f67aa-0dc8-432a-be64-061c52d7f310/_exp:2018-04-12 03:08:43.882097
PetaSAN/Sessions/280f67aa-0dc8-432a-be64-061c52d7f310/_permanent:True
PetaSAN/Sessions/280f67aa-0dc8-432a-be64-061c52d7f310/role_id:1
PetaSAN/Sessions/280f67aa-0dc8-432a-be64-061c52d7f310/user:admin
PetaSAN/Sessions/3f35e1b7-66bc-4f40-bc96-9743575352a3/_exp:2018-04-12 03:59:41.151850
PetaSAN/Sessions/3f35e1b7-66bc-4f40-bc96-9743575352a3/_permanent:True
PetaSAN/Sessions/3f35e1b7-66bc-4f40-bc96-9743575352a3/role_id:1
PetaSAN/Sessions/3f35e1b7-66bc-4f40-bc96-9743575352a3/user:admin
PetaSAN/Sessions/4c93878f-c050-4528-a00a-4812c33ec6d3/_exp:2018-04-11 12:17:48.175872
PetaSAN/Sessions/4c93878f-c050-4528-a00a-4812c33ec6d3/_permanent:True
PetaSAN/Sessions/4c93878f-c050-4528-a00a-4812c33ec6d3/role_id:1
PetaSAN/Sessions/4c93878f-c050-4528-a00a-4812c33ec6d3/user:admin
PetaSAN/Sessions/52c5ba2f-8388-4199-9561-f28bddf02d8e/_exp:2018-04-12 03:29:17.595977
PetaSAN/Sessions/52c5ba2f-8388-4199-9561-f28bddf02d8e/_permanent:True
PetaSAN/Sessions/52c5ba2f-8388-4199-9561-f28bddf02d8e/role_id:1
PetaSAN/Sessions/52c5ba2f-8388-4199-9561-f28bddf02d8e/user:admin
PetaSAN/Sessions/62cb0d7e-14ce-4b7c-b3ba-2f0ee636630c/_exp:2018-04-12 03:27:19.754286
PetaSAN/Sessions/62cb0d7e-14ce-4b7c-b3ba-2f0ee636630c/_permanent:True
PetaSAN/Sessions/6f405563-063e-4f5d-8ce1-e202d2cee81f/_exp:2018-04-11 12:43:16.761505
PetaSAN/Sessions/6f405563-063e-4f5d-8ce1-e202d2cee81f/_permanent:True
PetaSAN/Sessions/7a70bab0-e7b4-4c98-8349-e8d1116dddaf/_exp:2018-04-11 14:50:05.561961
PetaSAN/Sessions/7a70bab0-e7b4-4c98-8349-e8d1116dddaf/_permanent:True
PetaSAN/Sessions/7db1c4e7-2674-437b-961a-61e3aa743815/_exp:2018-04-11 12:05:35.058524
PetaSAN/Sessions/7db1c4e7-2674-437b-961a-61e3aa743815/_permanent:True
PetaSAN/Sessions/7db1c4e7-2674-437b-961a-61e3aa743815/role_id:1
PetaSAN/Sessions/7db1c4e7-2674-437b-961a-61e3aa743815/user:admin
PetaSAN/Sessions/cd98681c-4bd8-458c-9630-f5209334917a/_exp:2018-04-11 10:56:44.396607
PetaSAN/Sessions/cd98681c-4bd8-458c-9630-f5209334917a/_permanent:True
PetaSAN/Sessions/cd98681c-4bd8-458c-9630-f5209334917a/role_id:1
PetaSAN/Sessions/cd98681c-4bd8-458c-9630-f5209334917a/user:admin
PetaSAN/Sessions/cee9f40d-a58d-4c8c-b149-b7c7ac51c60a/_exp:2018-04-12 02:43:51.325884
PetaSAN/Sessions/cee9f40d-a58d-4c8c-b149-b7c7ac51c60a/_permanent:True
PetaSAN/Sessions/cee9f40d-a58d-4c8c-b149-b7c7ac51c60a/role_id:1
PetaSAN/Sessions/cee9f40d-a58d-4c8c-b149-b7c7ac51c60a/user:admin
PetaSAN/Sessions/eyJfcGVybWFuZW50Ijp0cnVlLCJzdWNjZXNzIjp7IiBiIjoiVG05a1pTQlRaWEoyYVdObElGTmxkSFJwYm1keklGTmhkbVZrIn19.Da9fUQ.0sK9Ur_KAQMmOqUvgG31Jw2oVwQ/_exp:2018-04-11 10:50:24.694995
PetaSAN/Sessions/eyJfcGVybWFuZW50Ijp0cnVlLCJzdWNjZXNzIjp7IiBiIjoiVG05a1pTQlRaWEoyYVdObElGTmxkSFJwYm1keklGTmhkbVZrIn19.Da9fUQ.0sK9Ur_KAQMmOqUvgG31Jw2oVwQ/_permanent:True
PetaSAN/Sessions/eyJfcGVybWFuZW50Ijp0cnVlLCJzdWNjZXNzIjp7IiBiIjoiVG05a1pTQlRaWEoyYVdObElGTmxkSFJwYm1keklGTmhkbVZrIn19.Da9fUQ.0sK9Ur_KAQMmOqUvgG31Jw2oVwQ/role_id:1
PetaSAN/Sessions/eyJfcGVybWFuZW50Ijp0cnVlLCJzdWNjZXNzIjp7IiBiIjoiVG05a1pTQlRaWEoyYVdObElGTmxkSFJwYm1keklGTmhkbVZrIn19.Da9fUQ.0sK9Ur_KAQMmOqUvgG31Jw2oVwQ/user:admin
PetaSAN/Sessions/fa86df41-024e-484e-bd7a-5a2a258f858f/_exp:2018-04-11 11:53:20.919854
PetaSAN/Sessions/fa86df41-024e-484e-bd7a-5a2a258f858f/_permanent:True
PetaSAN/Sessions/fa86df41-024e-484e-bd7a-5a2a258f858f/role_id:1
PetaSAN/Sessions/fa86df41-024e-484e-bd7a-5a2a258f858f/user:admin
PetaSAN/Users/admin:{"email": "", "name": "", "notfiy": false, "password": "password", "role_id": 1, "user_name": "admin"}
PetaSAN/leaders/petasan-node1:1235
PetaSAN/leaders/petasan-node2:1235
PetaSAN/leaders/petasan-node3:1
```


path assignment：
```
class PathAssignmentInfo(object):
    def __init__(self):
        self.ip = ""
        self.disk_id = ""
        self.disk_name = ""
        self.node = ""
        self.status = -1
        self.target_node = ""
```

```
class Path(object):
    def __init__(self):
        self.ip = ""
        self.subnet_mask = ""
        self.eth = ""
        self.locked_by=""
```



# ceph RBD镜像中的KV
在ceph RBD镜像的元信息文件中记录名为petasan-metada的扩展属性信息，属性实体类型为DiskMeta，包含以下信息：
```
class DiskMeta(object):
    def __init__(self):
        self.disk_name = ""
        self.description = ""
        self.acl = ""
        self.user = ""
        self.password = ""
        self.size = 0
        self.iqn = ""
        self.create_date = ""
        self.id = ""
        self.pool = ""
        self.paths = []
```


