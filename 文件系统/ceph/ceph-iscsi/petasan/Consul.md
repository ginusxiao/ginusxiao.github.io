# 提纲
[toc]

# Consul相关的文章

[Consul 官方入门指南](https://www.consul.io/intro/getting-started/install.html)

[Consul系列文章索引](http://blog.csdn.net/younger_china/article/details/79462530)

本文是关于Consul的系列文章，非常值得一看。

```
在Consul中有Client和Server的概念，这里的Server和Client只是Consul集群层面的区分，与搭建在Cluster之上的应用服务无关。
```

[深入学习consul](http://blog.csdn.net/yeyincai/article/details/51764092)
本文包括以下内容：

```
1. consul的基本概念
2. consul的安装和启动
3. consul中服务注册与发现的两种方式
```

[服务注册发现consul之四：基于Consul的KV存储和分布](https://www.cnblogs.com/duanxz/p/7040968.html)

[consul Leader Election](https://www.consul.io/docs/guides/leader-election.html)

[consule KV](https://www.consul.io/intro/getting-started/kv.html)

[consul KV翻译](https://segmentfault.com/a/1190000005040921)

[consul入门01-07](https://segmentfault.com/a/1190000005005227)

[关于consul session的理解](http://xiaorui.cc/2015/09/29/%E6%89%BE%E5%9D%91%E8%AE%B0%E4%B9%8Bconsul%E7%9A%84session%E6%B3%A8%E5%86%8C%E5%8F%8A%E6%9C%8D%E5%8A%A1%E5%8F%91%E7%8E%B0%E5%8A%9F%E8%83%BDpython/)
本文中聊到session和它所关联的key的生命周期：
“
if you create a session with the 'delete' behavior and acquire the key with it, then when the session expires or is destroyed the key will be deleted; if you release the key, then destroy the session, the key will not be deleted because the session is no longer associated with the key; if you create one session, associate it with multiple keys, then when the single session is deleted, all the keys associated with it will be deleted.
”




# consul关键
## Build Cluster
```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster/deploy.py):

class Wizard:
    def build(self):
        try:
            conf = configuration()

            if len(conf.get_cluster_info().management_nodes) == 0:
                ......
                
            elif len(conf.get_cluster_info().management_nodes) == 1:
                ......
                
            elif len(conf.get_cluster_info().management_nodes) == 2:
                status = self.check_remote_connection()
                    
                NTPConf().setup_ntp_local()

                logger.info("Stopping petasan services on all nodes.")
                self.stop_petasan_services()
                clean_ceph()
                clean_consul()

                status = build_consul()

                status = build_monitors()
                
                status = build_osds()

                logger.info("Main core components deployed.")

                #在Consul KV中分别记录每一个节点的{“node_name” ：“node_info”}
                self.__commit_management_nodes()

                logger.info("Starting all services.")
                #启动petasan服务
                self.start_petasan_services()
                #在/etc/hosts中添加cluster中所有management nodes的信息，每一个节点保存的内容为：{ip}   {host_name}
                #并通过FileSyncManager Commit到Consul的KV中
                self.add__node_to_hosts_file()

                SharedFS().setup_management_nodes()

                #将当前节点添加到cluster的management nodes中
                conf.add_management_node()

                logger.info("Updating rbd pool.")
                create_rbd_pool()

                logger.info("Waiting for ceph to reach active and clean status.")
                test_active_clean()
                #将/opt/petasan/config/cluster_info.json拷贝到cluster中所有其他management node中
                self.__sync_cluster_config_file()

                self.run_post_deploy_script()
                self.kill_petasan_console(True)
                logger.info("Node 3 added and cluster is now ready.")


            elif len(conf.get_cluster_info().management_nodes) == 3 and not os.path.exists(ConfigAPI().get_replace_file_path()):
                ......


            elif len(conf.get_cluster_info().management_nodes) == 3 and os.path.exists(ConfigAPI().get_replace_file_path()):
                ......
                
        except Exception as ex:
            ......

        return BuildStatus().done    
```

为了触发consul的leader选举，必须先将这些节点join到一起，并创建cluster，consul提供了3种join方式（参考[这里](https://www.consul.io/docs/guides/bootstrapping.html)），-retry-join选项的方式就是其中一种，这里就是借助于-retry-join选项来创建cluster。
```
(/usr/lib/python2.7/dist-packages/PetaSAN/core/consul/deploy/build.py):

def build_consul():
    try:
        # Generate a Security Key
        keygen = PetaSAN.core.common.cmd.exec_command('consul keygen')[0]
        keygen = str(keygen).splitlines()[0]
        logger.debug('keygen: ' + keygen)

        conf = configuration()
        cluster_info = conf.get_cluster_info()
        cluster_name = cluster_info.name
        logger.info('cluster_name: ' + cluster_name)

        local_node_info = conf.get_node_info()
        logger.info("local_node_info.name: " + local_node_info.name)

        #在当前节点上执行python /opt/petasan/scripts/create_consul_conf.py -key=kegen
        #命令执行结果为，生成/opt/petasan/config/etc/consul.d/server/config.json，其中
        #包含关于consul在本地启动的参数
        __create_leader_conf_locally(keygen)
        #依次在除当前节点以外的cluster节点中执行python /opt/petasan/scripts/create_consul_conf.py -key=kegen
        #命令执行结果跟在本地执行结果类似
        continue_building_cluster = __create_leader_conf_remotely(keygen, cluster_info, local_node_info)

        if continue_building_cluster is True:
            #在cluster中除本地节点以外的所有节点上执行python /opt/petasan/scripts/consul_start_up.py -retry-join #{local_node_ip}，假设cluster中有3个节点，分别为node1，node2，node3，且local_node是node1，则在node2上
            #执行consul agent -config-dir /opt/petasan/config/etc/consul.d/server -bind {node2_ip} -retry-join {node1_ip}
            #-retry-join {node3_ip} -retry-join {node1_ip}，在node3上执行consul agent -config-dir #/opt/petasan/config/etc/consul.d/server -bind {node3_ip} -retry-join {node1_ip}
            #-retry-join {node2_ip} -retry-join {node1_ip}
            __start_leader_remotely(cluster_info, local_node_info)
            #在本地节点上执行python  /opt/petasan/scripts/consul_start_up.py，假如cluster中有3个节点，分别为node1，
            #node2，node3，且local_node是node1，则在node1上执行/opt/petasan/config/etc/consul.d/server -bind {node1_ip}
            #-retry-join {node2_ip} -retry-join {node3_ip}
            __start_leader_locally()

        # sleep(5)
        consul_status_report = __test_leaders()
        logger.debug(consul_status_report)
        return consul_status_report
    except Exception as ex:
        ......
```

## leader选举
在创建了Cluster之后，就会调用self.start_petasan_services()来启动petasan服务：

```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster/deploy.py):

    def start_petasan_services(self,remote=True):
        cluster_conf = configuration()
        ssh_obj = ssh()
        #在本地执行python /opt/petasan/scripts/start_petasan_services.py build
        exec_command("python {} build ".format(ConfigAPI().get_startup_petasan_services_path()))
        sleep(5)
        if not remote:
            return
        try:
            #依次在远端节点上执行python /opt/petasan/scripts/start_petasan_services.py build
            for ip in cluster_conf.get_remote_ips(cluster_conf.get_node_name()):
                ssh_obj.exec_command(ip,"python {} build ".format(ConfigAPI().get_startup_petasan_services_path()))
                sleep(5)
        except Exception as ex:
            logger.exception(ex.message)
            raise ex

```

在/opt/petasan/scripts/start_petasan_services.py中会执行：

```
(/opt/petasan/scripts/start_petasan_services.py):

cluster_config = configuration()
building_stage = False

#在start_petasan_services中明确指定了build参数，所以building_stage会被设置为true
if len(sys.argv)>1:
    if str(sys.argv[1]) =='build':
        building_stage = True
        
startup_services(building_stage,cluster_config.are_all_mgt_nodes_in_cluster_config())

def startup_services(building_stage=False,cluster_complete=False):
    path = ConfigAPI().get_service_files_path()

    if not building_stage and cluster_complete:
        ......

    elif building_stage:

        exec_command('systemctl start petasan-mount-sharedfs')
        #如果是管理节点，则触发启动leader选举，启动petasan-cluster-leader.service
        if cluster_config.get_node_info().is_management:
            exec_command('systemctl start petasan-cluster-leader')

        logger.info("Starting cluster file sync service")
        exec_command('systemctl start petasan-file-sync')

        exec_command('/opt/petasan/scripts/load_iscsi_mods.sh')
        if cluster_config.get_node_info().is_iscsi:
            logger.info("Starting PetaSAN service")
            exec_command('systemctl start petasan-iscsi')
            sleep(2)

        if cluster_config.get_node_info().is_management:
            logger.info("Starting Cluster Management application")
            exec_command('systemctl start petasan-admin')

        logger.info("Starting Node Stats Service")
        exec_command('systemctl start petasan-node-stats')


    elif not building_stage and not cluster_complete:
        ......
```

systemctl start petasan-cluster-leader实际上会启动petasan-cluster-leader.service
```
(/lib/systemd/system/petasan-cluster-leader.service):


[Unit]
Description= PetaSAN Cluster Leader
After=syslog.target local-fs.target network.target

[Service]
Type=simple
#执行的是/opt/petasan/services/cluster_leader.py
ExecStart=/opt/petasan/services/cluster_leader.py
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```


```
(/opt/petasan/services/cluster_leader.py):

from PetaSAN.backend.cluster_leader import ClusterLeader

# 从start_petasan_services走到这里可知，实际上会在cluster的每个management node上执行ClusterLeader().run()
ClusterLeader().run()
```


```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster_leader.py):

# ClusterLeader继承自LeaderElectionBase
class ClusterLeader(LeaderElectionBase):

    def __init__(self):
        super(ClusterLeader,self).__init__(SERVICE_NAME)


    def start_action(self):
        logger.info('ClusterLeader start action')
        SharedFS().block_till_mounted()
        subprocess.call('/opt/petasan/scripts/stats-setup.sh',shell=True)
        subprocess.call('/opt/petasan/scripts/stats-start.sh',shell=True)

        subprocess.call('systemctl start petasan-notification',shell=True)
        return


    def stop_action(self) :
        logger.info('ClusterLeader stop action')
        subprocess.call('/opt/petasan/scripts/stats-stop.sh',shell=True)

        subprocess.call('systemctl stop petasan-notification',shell=True)
        return
```


```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster_leader.py):

class LeaderElectionBase(object):

    def __init__(self,service_name,watch_timeout='20s'):
        self.service_name = service_name
        self.service_key  = ConfigAPI().get_consul_services_path() + service_name
        self.watch_timeout = watch_timeout
        self.session = '0'
        #self.session_name = ConfigAPI().get_leader_service_session_name()
        self.session_name = self.service_name
        self.leader = False


    def run(self):
        modify_index = '0'
        node_name = configuration().get_node_info().name
        while True:
            try:
                sleep(LOOP_SLEEP_SEC)
                if self.session == '0':
                    self.session =ConsulAPI().get_new_session_ID(self.session_name,node_name)
                #等待直到self.service_key这个键包含了一个大于等于modify_index的修改后才会返回，或者由于超时而返回
                kv = ConsulAPI().get_key_blocking(self.service_key,modify_index,self.watch_timeout)
                if kv:
                    modify_index = kv.ModifyIndex
                    if not hasattr(kv,'Session'):
                        # cluster key not locked
                        if not self.leader :
                            # we had no prev lock, attempt to lock
                            if ConsulAPI().lock_key(self.service_key, self.session, ''):
                                # lock successful, change state and call start action
                                self.leader = True
                                self.start_action()

                        else :
                            # we had a prev lock, change state and call stop action
                            self.leader = False
                            self.stop_action()

                    else:
                        # cluster key locked
                        if self.leader and kv.Session != self.session:
                            # we had a prev lock, but is actually locked by someone else
                            # change state and call stop action
                            self.leader = False
                            self.stop_action()

                else:
                    # service key not found, attempt to lock it
                    if not self.leader :
                        if ConsulAPI().lock_key(self.service_key,  self.session, '') :
                            # lock successful, change state and call start action
                            self.leader = True
                            self.start_action()


            except ConnectionError:
                ......
                
            except Exception as e:
                ......

    def is_leader(self) :
        return self.leader

    def quit_leader(self) :
        if not self.leader :
            return
        self.leader = False
        self.stop_action()
        ConsulAPI().unlock_key(self.service_key, self.session, '')
        return

    def start_action(self):
        ......

    def stop_action(self) :
        ......

    def get_leader_node(self):
        try:
            consul_obj = ConsulAPI()
            kv = consul_obj.get_key(ConfigAPI().get_consul_services_path() + self.service_name)
            if kv is  None or not hasattr(kv,"Session"):
                return None

            sessions = consul_obj.get_sessions_dict()
            node_session = sessions.get(kv.Session,None)
            if not node_session:
                return  None
            return node_session.Node

        except Exception as e:
            pass

        return None
```

通过LeaderElectionBase.run()可知，如果某个节点被选择为leader，则其会调用start_action()，如果某个节点释放leader权利，则其会调用stop_action()。ClusterLeader类重载了start_action()和stop_action()。

```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster_leader.py):

class ClusterLeader(LeaderElectionBase):

    def __init__(self):
        super(ClusterLeader,self).__init__(SERVICE_NAME)


    def start_action(self):
        logger.info('ClusterLeader start action')
        SharedFS().block_till_mounted()
        #实时监控统计相关(grafana+graphite+collectd)的搭建和启动
        subprocess.call('/opt/petasan/scripts/stats-setup.sh',shell=True)
        subprocess.call('/opt/petasan/scripts/stats-start.sh',shell=True)
        #启动petasan-notification.service
        subprocess.call('systemctl start petasan-notification',shell=True)
        return


    def stop_action(self) :
        logger.info('ClusterLeader stop action')
        subprocess.call('/opt/petasan/scripts/stats-stop.sh',shell=True)

        subprocess.call('systemctl stop petasan-notification',shell=True)
        return
```

关于执行systemctl start petasan-notification后都干了些什么事情，请参考“notification.md”。


## leader替换

怎样才会发生replace leader操作？？？

```
(/usr/lib/python2.7/dist-packages/PetaSAN/core/consul/deploy/build.py):

class Wizard:
    def build(self):
        ......
        elif len(conf.get_cluster_info().management_nodes) == 3 and os.path.exists(ConfigAPI().get_replace_file_path()):
            ###################### Replace #########################################
            logger.info("Replace node is starting.")
            status = self.check_remote_connection()
            NTPConf().setup_ntp_local()

            logger.info("Stopping petasan services on local node.")
            self.stop_petasan_services(remote=False)
            logger.info("Starting clean_ceph.")
            clean_ceph_local()
            logger.info("Starting local clean_consul.")
            clean_consul_local()

            #首先，在当前节点上执行python /opt/petasan/scripts/create_consul_conf.py -key=kegen
            #命令执行结果为，生成/opt/petasan/config/etc/consul.d/server/config.json，其中
            #包含关于consul在本地启动的参数
            
            #其次，调用__start_leader_locally()，在本地启动consul，实际上执行python  #/opt/petasan/scripts/consul_start_up.py，
            status = replace_consul_leader()

            status= replace_local_monitor()

            status = create_osds_local()

            logger.info("Main core components deployed.")
            logger.info("Starting all services.")
            self.start_petasan_services(remote=False)
            test_active_clean()

            SharedFS().rebuild_management_node()

            logger.info("Node successfully added to cluster.")
            self.run_post_deploy_script()
            self.kill_petasan_console(False)
            os.remove(ConfigAPI().get_replace_file_path())
            return BuildStatus().done_replace
```

4. ConsulAPI(/usr/lib/python2.7/dist-packages/PetaSAN/core/consul/api.py)

```
待续。。。

```

5. 

# 关于consul的一些tips
1. consul的agent提供了watch功能，但是python客户端并未提供相应的接口。
2. 


# PetaSAN中的session信息列表
curl http://127.0.0.1:8500/v1/session/list
[
{"ID":"0edb7bb9-fcf4-ca7b-51bd-0a00f80b73a1","Name":"ClusterLeader","Node":"petasan-node2","Checks":["serfHealth"],"LockDelay":15000000000,"Behavior":"release","TTL":"","CreateIndex":194,"ModifyIndex":194},
{"ID":"5f88a4e9-2579-33b5-ad8c-b313873d71dd","Name":"iSCSITarget","Node":"petasan-node2","Checks":["serfHealth"],"LockDelay":15000000000,"Behavior":"release","TTL":"","CreateIndex":192,"ModifyIndex":192},
{"ID":"67d6eda1-270a-d9b1-0ab1-0033d9aba6ed","Name":"ClusterLeader","Node":"petasan-node1","Checks":["serfHealth"],"LockDelay":15000000000,"Behavior":"release","TTL":"","CreateIndex":190,"ModifyIndex":190},
{"ID":"782be14d-5e63-ed50-acf5-91156d6439ed","Name":"iSCSITarget","Node":"petasan-node3","Checks":["serfHealth"],"LockDelay":15000000000,"Behavior":"release","TTL":"","CreateIndex":182,"ModifyIndex":182},
{"ID":"840baac4-d388-c242-1443-a71e56b77fa2","Name":"ClusterLeader","Node":"petasan-node3","Checks":["serfHealth"],"LockDelay":15000000000,"Behavior":"release","TTL":"","CreateIndex":184,"ModifyIndex":184},
{"ID":"ff2941b0-b115-f958-2035-cc4f04d802ce","Name":"iSCSITarget","Node":"petasan-node1","Checks":["serfHealth"],"LockDelay":15000000000,"Behavior":"release","TTL":"","CreateIndex":188,"ModifyIndex":188}
]

其中Name为ClusterLeader的都是与选举有关的session，Name为ISCSITarget的都是与iscsi服务相关的session，实际上在PetaSAN中还有一类Name为AssignmentPaths的session，这种session只在发生主动的path assignment的情况下才会产生。



# PetaSAN中各session在失效的情况下对它所关联的KV的影响
iscsi_service.py中：
self.__session = ConsulAPI().get_new_session_ID(self.__session_name,self.__node_info.name)

leader_election_base.py中：
self.session =ConsulAPI().get_new_session_ID(self.session_name,node_name)

manage_path_assignment.py中：
session = consul_api.get_new_session_ID(config_api.get_assignment_session_name(),
                                                configuration().get_node_name(), True)
                                                
consul_api.get_new_session_ID实现如下：

```
    def get_new_session_ID(self,session_name,node_name,is_expire=False ):
        if node_name:
            self.drop_all_node_sessions(session_name,node_name)
        consul_obj = Consul()
        if is_expire:
            session = consul_obj.session.create(name=session_name,node=node_name,behavior="delete",lock_delay=3)
        else:
            session = consul_obj.session.create(name=session_name,node=node_name)
        return session
```

从get_new_session_ID中可以看到，在创建session的时候，如果is_expire参数被指定为true，则创建的session在失效的时候会删除所有与之关联的KV，如果is_expire参数未被指定或者被显示指定为false，则创建的session在失效的时候会保留所有与之关联的KV，但是这些KV的session信息被清空。

而PetaSAN中的3种session中，只有名为AssignmentPaths的session在创建的时候会设置is_expire参数为true，也就是说如果这种类型的session失效，则所有与之关联的path assignment将会被删除。而对于名为ClusterLeader和ISCSITarget的session，在session失效的情况下，所有与之关联的KV将会清除session信息。

对于PetaSAN中的这3种session，我们最为关注的是名为ISCSITarget的session，这种类型的使用如下：

```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/iscsi_service.py):

class Service:
    ......
    # iSCSITarget
    __session_name = ConfigAPI().get_iscsi_service_session_name()
    ......
    
    def start(self):
        self.__image_name_prefix = self.__app_conf.get_image_name_prefix()
        # Handel the case of cluster has just started
        if self.__node_info.is_management:
            clean_thread = threading.Thread(target=self.handle_cluster_startup)
            clean_thread.start()

        try:
            logger.info("Service is starting.")
            self.__clean()
        except Exception as e:
            logger.error("Error could not clean mapped disks")

        while True:
            try:
                if self.__session == "0":
                    ###############################################
                    ###############################################
                    self.__session = ConsulAPI().get_new_session_ID(self.__session_name,self.__node_info.name)
                    ......
                    
    def __acquire_path(self, path,consul_kv):
        if self.__ignored_acquire_paths.get(path):
            logger.info("Ignore forced path {}".format(path))
            return
        logger.debug("Start acquire path {} by node session {}.".format(path, self.__session))
        consul_api = ConsulAPI()
        ceph_api = CephAPI()
        lio_api = LioAPI()
        network_api = NetworkAPI()
        config = configuration()
        try:
            disk_id, path_index = str(path).split("/")
            image_name = self.__image_name_prefix + disk_id
            logger.debug("Start read image meta for acquire path {}.".format(path))
            all_image_meta = ceph_api.read_image_metadata(image_name)
            petasan_meta = all_image_meta.get(self.__app_conf.get_image_meta_key())
            disk_meta = DiskMeta()
            disk_meta.load_json(petasan_meta)
            logger.debug("End read image meta for acquire path {}.".format(path))

            logger.debug("Try to acquire path {}.".format(path))
            node_name = config.get_node_name()
            ###############################################
            ###############################################
            result = consul_api.lock_disk_path(self.__app_conf.get_consul_disks_path() + path,                                     self.__session,node_name,
                                               str(consul_kv.CreateIndex))
            ......
            
            
            
(/usr/lib/python2.7/dist-packages/PetaSAN/core/consul/api.py):

    def lock_disk_path(self, path, session, data, created_index):
        consul_obj = Consul()
        ###############################################
        ###############################################
        result = consul_obj.kv.put(path, data, None, None, str(session))
        kv = self.find_disk_path(path)
        if kv != None and int(kv.CreateIndex) == int(created_index) and result == True:
            return True
        elif kv != None and int(kv.CreateIndex) != int(created_index) and result == True:
                self.delete_disk(path)
                return False
        return False
```

从代码来看，将名为iSCSITarget的session关联到每一条disk path，而名为iSCSITarget的session在失效的时候会将它所关联的所有KV的session清空，所以重新关联到新的session的时候可以成功。





