
```
/lib/systemd/system/petasan-deploy.service
    ExecStart=/opt/petasan/services/web/deploy.py
```




```
/opt/petasan/services/web/deploy.py
from PetaSAN.web.deploy_controller.wizard import wizard_controller
app.register_blueprint(wizard_controller)

if __name__ == '__main__':
    #app.run("192.168.57.199",port=5001)
    #启动服务器，执行轮询（默认使用单进程单线程的werkzeug.serving.BaseWSGIServer处理请求， 
    #实际上还是使用标准库BaseHTTPServer.HTTPServer，通过select.select做0.5秒的“while #TRUE”的事件轮询）
    app.run(Network().get_node_management_ip(),port=5001)
```

        
        


```
/usr/lib/python2.7/dist-packages/PetaSAN/web/deploy_controller/wizard.py
@wizard_controller.route('/', methods=['GET', 'POST'])
def main():
    if "submitted_page" not in session:
        ......
    
    else:
        page = session["submitted_page"]
        session.pop("submitted_page")
        if page == "cluster_interface_setting":
            ......
        
        elif page == "cluster_network_setting":
            ......
            
        elif page == "node_summary_setting":
            ......
            
        elif page == "node_network_setting":
            ......
            
        elif page == "node_role_setting":
            ......
            
        elif page == "join_cluster":
            ......
            
        elif page == "replace_node":
            ......
            
        elif page == "cluster_info":
            ......
            
        elif page == "tuning_form":
            ......
            
        elif page == "build_load":
            #这是关于某一个node部署完毕的时候，在build.html中会打印“Final Deployment #Stage”，如果尚未达到3个管理节点的规模，则会提示继续添加新的管理节点
            return render_template('deploy/build.html')
```



对于后续以“Join Existing Cluster”的方式启动的节点，会调用save_join_cluster
    
```
/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster/deploy.py
    class Wizard:
        def join(self, ip, password):
            #主要执行：建立和第一个cluster节点的无密码ssh登录；拷贝在添加第一个节点的时候关于cluster的配置；
```



```
/usr/lib/python2.7/dist-packages/PetaSAN/web/deploy_controller/wizard.py            
@wizard_controller.route('/build', methods=['GET'])
def build_cluster():
    if request.method == 'GET':
        try:
            wizerd = Wizard()
            management_urls = []
            tasks_mssages = []

            status = wizerd.build()
            #status = -3
            if status == BuildStatus.done:
                management_urls = wizerd.get_management_urls()

            tasks = wizerd.get_status_reports().failed_tasks
            for task in tasks:
                msg = gettext(task)
                tasks_mssages.append(msg)
            # tasks_mssages.append("test error1")
            execution_status = executionStatus()
            execution_status.status = status
            execution_status.report_status = tasks_mssages
            execution_status.management_url = management_urls
            data = execution_status.write_json()
            return data

        except Exception as e:
            session['err'] = "ui_deploy_build_err_exception"
            logger.error(e)
```




```
/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster/deploy.py
class Wizard:
    def build(self):
        try:
            conf = configuration()

            if len(conf.get_cluster_info().management_nodes) == 0:
                ......
                
            elif len(conf.get_cluster_info().management_nodes) == 1:
                ......
                
            elif len(conf.get_cluster_info().management_nodes) == 2:
                #cluster中已经有2个节点，当前正在添加第3个节点
                ......            
                self.stop_petasan_services()
                clean_ceph()
                clean_consul()
                status = build_consul()
                status = build_monitors()
                status = build_osds()
                self.__commit_management_nodes()
                self.start_petasan_services()
                self.add__node_to_hosts_file()
                SharedFS().setup_management_nodes()
                conf.add_management_node()
                create_rbd_pool()
                test_active_clean()
                self.__sync_cluster_config_file()
                self.run_post_deploy_script()
                self.kill_petasan_console(True)
                ......
                
            elif len(conf.get_cluster_info().management_nodes) == 3 and not os.path.exists(ConfigAPI().get_replace_file_path()):
                #cluster中已经有3个节点，虽然已经形成了cluster，但是cluster中可以添加更多的节点，
                #这些新添加的节点关于consul的角色是client
                self.stop_petasan_services(remote=False)
                clean_ceph_local()
                clean_consul_local()
                status = build_consul_client()
                copy_ceph_config_from_mon()
                create_osds_local()
                self.start_petasan_services()
                test_active_clean()
                self.__commit_local_node()
                self.add__node_to_hosts_file(remote=False)
                self.kill_petasan_console(False)
                self.run_post_deploy_script()
                ......

            elif len(conf.get_cluster_info().management_nodes) == 3 and os.path.exists(ConfigAPI().get_replace_file_path()):
                #替换节点的情况
                ......            
                self.start_petasan_services()
                ......
```



```
/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster/deploy.py
class Wizard:
    def start_petasan_services(self,remote=True):
        cluster_conf = configuration()
        ssh_obj = ssh()
        #执行python /opt/petasan/scripts/start_petasan_services.py build
        exec_command("python {} build ".format(ConfigAPI().get_startup_petasan_services_path()))
        sleep(5)
        if not remote:
            return
        try:
            for ip in cluster_conf.get_remote_ips(cluster_conf.get_node_name()):
                ssh_obj.exec_command(ip,"python {} build ".format(ConfigAPI().get_startup_petasan_services_path()))
                sleep(5)
        except Exception as ex:
            logger.exception(ex.message)
            raise ex
```



```
/opt/petasan/scripts/start_petasan_services.py
    cluster_config = configuration()
    building_stage = False
    
    if len(sys.argv)>1:
        if str(sys.argv[1]) =='build':
            building_stage = True
    
    startup_services(building_stage,cluster_config.are_all_mgt_nodes_in_cluster_config())
```




```
/opt/petasan/scripts/start_petasan_services.py
startup_services(building_stage=False,cluster_complete=False):
    if not building_stage and cluster_complete:
        ......
    
    elif building_stage:
        #走这个分支？
        exec_command('systemctl start petasan-mount-sharedfs')
        if cluster_config.get_node_info().is_management:
            #leader选举，最终会执行/opt/petasan/services/cluster_leader.py
            exec_command('systemctl start petasan-cluster-leader')

        #
        exec_command('systemctl start petasan-file-sync')
        exec_command('/opt/petasan/scripts/load_iscsi_mods.sh')
        if cluster_config.get_node_info().is_iscsi:
            exec_command('systemctl start petasan-iscsi')
            sleep(2)

        if cluster_config.get_node_info().is_management:
            exec_command('systemctl start petasan-admin')

        exec_command('systemctl start petasan-node-stats')

    elif not building_stage and not cluster_complete:
        exec_command(python /opt/petasan/scripts/node_start_ips.py)
```

  
  

```
/opt/petasan/services/cluster_leader.py：
    from PetaSAN.backend.cluster_leader import ClusterLeader
    ClusterLeader().run()
```




```
/usr/lib/python2.7/dist-packages/PetaSAN/backend/cluster_leader.py        
class ClusterLeader(LeaderElectionBase):
    # ClusterLeader继承自LeaderElectionBase，在LeaderElectionBase中定义了run函数
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
                logger.error("LeaderElectionBase connection error")
                sleep(CONNECTION_RETRY_TIMEOUT_SEC)
            except Exception as e:
                logger.error(e.message)
                sleep(CONNECTION_RETRY_TIMEOUT_SEC)
                if str(e.message).find("invalid session") > -1:
                    logger.error("LeaderElectionBase session is invalid")
                    self.stop_action()
                    try:
                        self.session = ConsulAPI().get_new_session_ID(self.session_name,node_name)
                        logger.info("LeaderElectionBase new session id created {}".format(self.session))
                    except:
                        logger.error("LeaderElectionBase connection error")
                        sleep(CONNECTION_RETRY_TIMEOUT_SEC)

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
        logger.info("LeaderElectionBase start action")
        return

    def stop_action(self) :
        logger.info("LeaderElectionBase stop action")
        return




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

        
        

```
/opt/petasan/services/file_sync.py:        
    from PetaSAN.backend.file_sync_manager import FileSyncManager
    FileSyncManager().sync()
```

    



```
/opt/petasan/scripts/load_iscsi_mods.sh
    modprobe configfs
    modprobe rbd single_major=Y
    modprobe target_core_mod //iscsi target相关
    modprobe target_core_rbd //iscsi target for rbd相关
    modprobe iscsi_target_mod //iscsi target相关
    mkdir /sys/kernel/config/target/iscsi
```

    
    

```
systemctl start petasan-iscsi
    ExecStart=/opt/petasan/services/iscsi_service.py
```

    
    


```
/opt/petasan/services/iscsi_service.py
    from PetaSAN.backend.iscsi_service import Service
    #启动iscsi service
    app_service = Service()
    app_service.start()
```

    
    


```
/usr/lib/python2.7/dist-packages/PetaSAN/backend/iscsi_service.py    
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

    # 这是一个死循环，也就是说iscsi service在不停的执行下述工作
    while True:
        try:
            if self.__session == "0":
                self.__session = ConsulAPI().get_new_session_ID(self.__session_name,self.__node_info.name)

            consul_api = ConsulAPI()
            self.__current_lock_index = consul_api.current_index()
            if not self.__current_lock_index:
                sleep(1)
                continue
            self.__process()
            old_index = self.__current_lock_index
            self.__current_lock_index = consul_api.watch(self.__current_lock_index)
            if old_index != self.__current_lock_index:
                # Give a chance to get all changes that occurred in the same time in cosnul.
                sleep(2)

            self.__exception_retry_timeout = 0
            self.__failure_timeout = timedelta(minutes=self.__app_conf.get_failure_timeout_duration_min()) +datetime.utcnow()
        except (ConnectionError , RetryConsulException) as ex:
            logger.error("Error on consul connection.")
            logger.exception(ex)
            self.__exception_retry_timeout += 5
        except Exception as ex:
            logger.error("Error during process.")
            logger.exception(ex)
            self.__exception_retry_timeout += 1

        sleep(self.__exception_retry_timeout)
        if self.__exception_retry_timeout > 10:
            logger.warning("PetaSAN could not complete process, there are too many exceptions.")
            self.__exception_retry_timeout = 1
        sleep(self.__exception_retry_timeout)

        # Clean all installed configurations if service did not successfully for 5 minutes.
        if self.__failure_timeout < datetime.utcnow():
            logger.warning("There are too many exceptions.Service will clean this node.")
            self.__clean()
            self.__failure_timeout = timedelta(minutes=self.__app_conf.get_failure_timeout_duration_min()) +datetime.utcnow()
```






