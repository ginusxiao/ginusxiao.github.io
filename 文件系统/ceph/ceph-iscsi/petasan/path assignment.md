
```
@disk_controller.route('/disk/path_assignment/assign_paths/auto/<type>', methods=['POST'])
@requires_auth
@authorization("PathAssignment")
def auto_assign_paths(type):
    if request.method == 'POST':
        try:
            manage_path_assignment = MangePathAssignment()
            manage_path_assignment.auto(int(type))
            return redirect(url_for('disk_controller.paths_list'))

        except Exception as e:
            ......


@disk_controller.route('/disk/path_assignment/assign_paths/manual', methods=['POST'])
@requires_auth
@authorization("PathAssignment")
def manual_assign_paths():
    if request.method == 'POST':
        try:
            selected_paths = request.form.getlist('option_path[]')
            selected_paths_lst = [str(x) for x in selected_paths]
            paths_assign_info_lst = []
            logger.info("User starts manual assignments.")
            for path in selected_paths_lst:
                selected_paths_assign_info = path.split('##')

                paths_assign_info = PathAssignmentInfo()
                paths_assign_info.node = selected_paths_assign_info[0]
                paths_assign_info.disk_name = selected_paths_assign_info[1]
                paths_assign_info.ip = selected_paths_assign_info[2]
                paths_assign_info.disk_id = selected_paths_assign_info[3]

                paths_assign_info_lst.append(paths_assign_info)
                logger.info("User selected path {} {} {}.".format(paths_assign_info.disk_id ,
                                                                  paths_assign_info.disk_name,paths_assign_info.node))

            assign_to = request.form.get('assign_to')
            manage_path_assignment = MangePathAssignment()
            if assign_to == "auto":
                logger.info("User selected auto option in manual assignment.")
                manage_path_assignment.manual(paths_assign_info_lst)
                logger.info("System started auto assignments.")
                #pass
            else:
                logger.info("User selected manual option in assignment.")
                manage_path_assignment.manual(paths_assign_info_lst , assign_to)
                logger.info("System started manual assignments.")

            session['success'] = "ui_admin_disks_paths_assignment_list_success"
            return redirect(url_for('disk_controller.paths_list'))

        except Exception as e:
            ......
```


```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/mange_path_assignment.py):

auto_plugins = ['PetaSAN.core.path_assignment.plugins.average_paths']

class MangePathAssignment(object):
    def auto(self, type=1):
        logger.info("User start auto reassignment paths.")
        assignments_stats = self.get_assignments_stats()
        if assignments_stats.is_reassign_busy:
            logger.error("There is already reassignment running.")
            raise Exception("There is already reassignment running.")

        ConsulAPI().drop_all_node_sessions(self.__app_conf.get_consul_assignment_path(),
                                           configuration().get_node_name())
        sleep(3)

        assignments_stats.paths = [path for path in assignments_stats.paths if
                                   len(path.node.strip()) > 0 and path.status == -1]
        self.__context.paths = assignments_stats.paths
        self.__context.nodes = assignments_stats.nodes
        for plugin in self._get_new_plugins_instances(auto_plugins):
            if plugin.is_enable() and plugin.get_plugin_id() == type:
                paths_assignments = plugin.get_new_assignments()
                if len(paths_assignments) == 0:
                    logger.info("There is no node under average.")
                    return
                self.set_new_assignments(paths_assignments)
                break
        self.run()

    def manual(self, paths_assignment_info, assign_to="auto"):

        assignments_stats = self.get_assignments_stats()
        if assignments_stats.is_reassign_busy:
            logger.error("There is already reassignment running.")
            raise Exception("There is already reassignment running.")
        ConsulAPI().drop_all_node_sessions(self.__app_conf.get_consul_assignment_path(),
                                           configuration().get_node_name())
        sleep(3)  # Wait to be sure the session dropped
        if assign_to == "auto":

            logger.info("User start auto reassignment paths for selected paths.")
            assignments_stats.paths = [path for path in assignments_stats.paths if
                                       len(path.node.strip()) > 0 and path.status == -1]
            self.__context.paths = assignments_stats.paths
            self.__context.nodes = assignments_stats.nodes
            self.__context.user_input_paths = paths_assignment_info
            for plugin in self._get_new_plugins_instances(auto_plugins):
                if plugin.is_enable() and plugin.get_plugin_id() == 1:
                    paths_assignments = plugin.get_new_assignments()
                    self.set_new_assignments(paths_assignments)
                    logger.info("User start auto reassignment paths for selected paths.")
                    self.run()
                    break
            pass
        else:

            for path_assignment_info in paths_assignment_info:
                path_assignment_info.target_node = assign_to
                path_assignment_info.status = ReassignPathStatus.pending
            logger.info("User start manual reassignment paths for selected paths.")
            self.set_new_assignments(paths_assignment_info)

            self.run()
```

无论是MangePathAssignment.manual()还是MangePathAssignment.auto()最终都会调用到MangePathAssignment.run()，MangePathAssignment.run()调用关系如下：

```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/manage_path_assignment.py):

class MangePathAssignment(object):
    def run(self):
        #执行python /opt/petasan/scripts/admin/reassignment_paths.py server &
        cmd = "python {} server &".format(ConfigAPI().get_assignment_script_path())
        call_cmd(cmd)
```

```
(/opt/petasan/scripts/admin/reassignment_paths.py):

if __name__ == '__main__':
   main(sys.argv[1:])
   
def main(argv):
    args = Prepare().parser()
    main_catch(args.func, args)
    
class Prepare(object):
    @staticmethod
    def parser():
        parser = argparse.ArgumentParser(add_help=True)
        subparser = parser.add_subparsers()
        subnp_server = subparser.add_parser('server')
        subnp_server.set_defaults(func=server)


        subp_path_host = subparser.add_parser('path_host')
        subp_path_host.set_defaults(func=path_host)
        subp_path_host.add_argument(
            '-ip',
            help='Disk path ip.', required=True
        )
        subp_path_host.add_argument(
            '-disk_id',
            help='Disk ID.', required=True
        )
        args = parser.parse_args()
        return args
    
def main_catch(func, args):
    try:
        func(args)

    except Exception as e:
        logger.error(e.message)
        print ('-1')
        
因为在MangePathAssignment.run()中给定的参数是server，所以执行的是server()。

def server(args):

    try:
        logger.info("Reassignment paths script invoked to run process action.")
        #
        MangePathAssignment().process()
    except Exception as ex:
        logger.error("error process reassignments actions.")
        logger.exception(ex.message)
        print (-1)
        sys.exit(-1)
        
```


```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/manage_path_assignment.py):

class MangePathAssignment(object):
    def process(self):
        logger.info("Start process reassignments paths.")
        max_retry = 100
        current_reassignments = self.get_current_reassignment()
        config = configuration()
        assignment_script_path = ConfigAPI().get_assignment_script_path()
        if current_reassignments is None:
            return
        for ip, path_assignment_info in current_reassignments.iteritems():
            logger.info("process path {} and its status is {}".format(ip, path_assignment_info.status))
            if path_assignment_info.status == ReassignPathStatus.pending:
                logger.info(
                    "Move action,try clean disk {} path {} remotely on node {}.".format(path_assignment_info.disk_name,
                                                                                        path_assignment_info.disk_id,
                                                                                        path_assignment_info.node))

                status = False
                try:
                    #执行命令python /opt/petasan/scripts/admin/reassignment_paths.py path_host -ip {ip} -disk_id {disk_id}
                    cmd = "python {} path_host -ip {} -disk_id {}".format(assignment_script_path,
                                                                          path_assignment_info.ip
                                                                          , path_assignment_info.disk_id)
                    out, err = ssh().exec_command(path_assignment_info.node, cmd)
                    logger.info(cmd)
                    # self.clean_source_node(path_assignment_info.ip,path_assignment_info.disk_id)
                except Exception as ex:
                    logger.exception(ex.message)
                    out = ""

                if str(out).strip() == "0":
                    logger.info("Move action passed")
                    status = True

                current_path_assignment_info = None
                if status:
                    for i in xrange(0, max_retry):
                        logger.debug("Wait to update status of path {}.".format(path_assignment_info.ip))
                        sleep(0.25)
                        reassignments = self.get_current_reassignment()
                        if reassignments:
                            current_path_assignment_info = reassignments.get(path_assignment_info.ip)
                            if current_path_assignment_info and current_path_assignment_info.status == ReassignPathStatus.moving:
                                continue
                            else:
                                logger.info(
                                    "Process completed for path {} with status {}.".format(
                                        current_path_assignment_info.ip,
                                        current_path_assignment_info.status))
                                break
                    if current_path_assignment_info and current_path_assignment_info.status == ReassignPathStatus.moving:
                        self.update_path(current_path_assignment_info, ReassignPathStatus.failed)
                        logger.info("Move action,failed ,disk {} path {}.".format(path_assignment_info.disk_name,
                                                                                  path_assignment_info.disk_id,
                                                                                  path_assignment_info.node))

                else:
                    self.update_path(path_assignment_info, ReassignPathStatus.failed)
                    logger.info("Move action ,failed to clean disk {} path {} remotely on node .".format(
                        path_assignment_info.disk_name,
                        path_assignment_info.disk_id,
                        path_assignment_info.node))
        sleep(10)  # wait for display status to user if needed
        logger.info("Process completed.")
        self.remove_assignment()
        ConsulAPI().drop_all_node_sessions(self.__app_conf.get_consul_assignment_path(), config.get_node_name())
```

```
(/opt/petasan/scripts/admin/reassignment_paths.py):

if __name__ == '__main__':
   main(sys.argv[1:])
   
def main(argv):
    args = Prepare().parser()
    main_catch(args.func, args)
    
class Prepare(object):
    @staticmethod
    def parser():
        parser = argparse.ArgumentParser(add_help=True)
        subparser = parser.add_subparsers()
        subnp_server = subparser.add_parser('server')
        subnp_server.set_defaults(func=server)


        subp_path_host = subparser.add_parser('path_host')
        subp_path_host.set_defaults(func=path_host)
        subp_path_host.add_argument(
            '-ip',
            help='Disk path ip.', required=True
        )
        subp_path_host.add_argument(
            '-disk_id',
            help='Disk ID.', required=True
        )
        args = parser.parse_args()
        return args
    
def main_catch(func, args):
    try:
        func(args)

    except Exception as e:
        logger.error(e.message)
        print ('-1')
        
因为在MangePathAssignment.process()中给定的参数是path_host，所以执行的是path_host()。

def path_host(args):
    logger.info("Reassignment paths script invoked to run clean action.")
    if MangePathAssignment().clean_source_node(args.ip,args.disk_id):
        print "0"
        return

    print "-1"
```


```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/manage_path_assignment.py):

class MangePathAssignment(object):
    def clean_source_node(self, ip, disk_id):
        if not self.update_path(ip, ReassignPathStatus.moving):
            return False
        disk = CephAPI().get_disk_meta(disk_id)
        paths_list = disk.paths
        disk_path = None
        path_index = -1

        for i in xrange(0, len(paths_list)):
            path_str = paths_list[i]
            path = Path()
            path.load_json(json.dumps(path_str))
            if path.ip == ip:
                disk_path = path
                path_index = i
                break
        if disk_path:
            self._clean_iscsi_config(disk_id, path_index, disk.iqn)
            network = Network()
            NetworkAPI().delete_ip(path.ip, path.eth, path.subnet_mask)
            if network.is_ip_configured(ip):
                logger.error(
                    "Move action,cannot clean newtwork config for disk {} path {}.".format(disk_id, path_index))
                self.update_path(ip, ReassignPathStatus.failed)
                return False
            logger.info("Move action,clean newtwork config for disk {} path {}.".format(disk_id, path_index))
            key = self.__app_conf.get_consul_disks_path() + disk_id + "/" + str(path_index + 1)
            consul_api = ConsulAPI()
            session = self._get_node_session(configuration().get_node_name())
            if ConsulAPI().is_path_locked_by_session(key, session):
                consul_api.release_disk_path(key, session, None)
                logger.info("Move action,release disk {} path {}.".format(disk_id, path_index + 1))
        else:
            self.update_path(ip, ReassignPathStatus.failed)
            return False

        return True
```


在代码中只有在Service.__acquire_path中才会调用MangePathAssignment().update_path(path_obj.ip,ReassignPathStatus.succeeded)设置path assignments成功。



```
class MangePathAssignment(object):
    def _filter_assignments_stats(self, filter_type=0, filter_text=None, set_session=False):

        __disk_consul_stopped = set()
        running_paths = dict()
        ceph_api = CephAPI()
        consul_api = ConsulAPI()
        disk_kvs = consul_api.get_disk_kvs()

        # Step 1 get all running paths.
        # 获取所有running paths（剔除那些stop的disks，这些stop的disks存放在__disk_consul_stopped中）
        for consul_kv_obj in disk_kvs:
            path_key = str(consul_kv_obj.Key).replace(self.__app_conf.get_consul_disks_path(), "")
            disk_id = str(path_key).split('/')[0]
            if disk_id in __disk_consul_stopped:
                continue
            if consul_kv_obj.Value == "disk":
                disk_id = str(path_key).split('/')[0]

                # Step 2 avoid stopping disks
                if str(consul_kv_obj.Flags) == "1":
                    __disk_consul_stopped.add(disk_id)
                continue

            running_paths[path_key] = consul_kv_obj

        if len(running_paths) == 0:
            return AssignmentStats()

        # Step 3 get all images metadata
        #获取所有image的元数据文件
        images = ceph_api.get_disks_meta()

        assignment_stats = AssignmentStats()

        # Step 4 get current reassignments
        current_running_assignments = self.get_current_reassignment()
        if current_running_assignments is not None:
            assignment_stats.is_reassign_busy = True
            filter_type = 0  # we will stop any filter and get all data if here is running reassignment

        # Step 5 fill paths assignment info
        for path_key, consul_kv_obj in running_paths.iteritems():
            disk_id = str(path_key).split('/')[0]
            disk = next((img for img in images if img.id == disk_id), None)
            if disk is None:
                continue
            disk_path = Path()
            path_index = int(str(path_key).split(disk_id + "/")[1])
            path_str = disk.paths[path_index - 1]
            disk_path.load_json(json.dumps(path_str))

            path_assignment_info = PathAssignmentInfo()
            path_assignment_info.ip = disk_path.ip
            path_assignment_info.disk_name = disk.disk_name
            path_assignment_info.disk_id = disk_id
            path_assignment_info.index = path_index
            current_path = None
            if current_running_assignments is not None:
                current_path = current_running_assignments.get(disk_path.ip)
            if hasattr(consul_kv_obj, "Session") and self.__session_dict.has_key(consul_kv_obj.Session):
                # Fill status and node name for started paths
                path_assignment_info.node = self.__session_dict.get(consul_kv_obj.Session).Node

                if current_running_assignments is not None:

                    if current_path is not None and current_path.status != -1:
                        path_assignment_info.status = current_path.status
                        path_assignment_info.target_node = current_path.target_node
                        if set_session:
                            # session refers to the node that lock this path assignment,This property helps to know the
                            # status of path and the node will handle this path
                            path_assignment_info.session = current_path.session
            elif current_path:
                path_assignment_info.node = current_path.node
                path_assignment_info.target_node = current_path.target_node
                path_assignment_info.status = current_path.status
                if set_session:
                    path_assignment_info.session = current_path.session

            # Step 6 search or get all
            if filter_type == 1 and filter_text is not None and len(str(filter_text).strip()) > 0:  # by disk name
                if filter_text.strip().lower() in path_assignment_info.disk_name.lower():
                    assignment_stats.paths.append(path_assignment_info)
            elif filter_type == 2 and filter_text is not None and len(str(filter_text).strip()) > 0:  # by ip
                if filter_text.strip() == path_assignment_info.ip.strip():
                    assignment_stats.paths.append(path_assignment_info)
                    break
            else:
                assignment_stats.paths.append(path_assignment_info)

            # Step 7 set all online nodes
        assignment_stats.nodes = self._get_nodes()

        return assignment_stats
```

```
class MangePathAssignment(object):
    def get_current_reassignment(self):
        paths = ConsulAPI().get_assignments()
        if paths is not None:
            for ip, path_assignment_info in paths.iteritems():
                if not hasattr(path_assignment_info, "session"):
                    logger.info("Path {} not locked by node.".format(path_assignment_info.ip))
                if not hasattr(path_assignment_info, "session") and path_assignment_info.status not in [
                    ReassignPathStatus.succeeded, ReassignPathStatus.failed]:
                    path_assignment_info.status = ReassignPathStatus.failed
        return paths
```
