
```
@disk_controller.route('/disk/add/save', methods=['POST'])
@requires_auth
@authorization("AddDisk")
def save_disk():
    if request.method == 'POST':
        try:
            disk = DiskMeta()
            disk.size = int(request.form['diskSize'])
            disk.disk_name = request.form['diskName']
            if 'orpUseFirstRange' not in request.values:
                isAutomaticIp = "Yes"
            else:
                isAutomaticIp = request.form['orpUseFirstRange']

            activePathsCount = int(request.form['ActivePaths'])
            if activePathsCount >= 3:
                isAutomaticIp = "Yes"
            path_type = int(request.form['ISCSISubnet'])
            if path_type == 1:
                path_type = PathType.iscsi_subnet1
            elif path_type == 2:
                path_type = PathType.iscsi_subnet2
            elif path_type == 3:
                path_type = PathType.both
            manual_ips = []

            if isAutomaticIp != "Yes":
                manual_ips.append(request.form['path1'])
                isAutomaticIp = False
                if activePathsCount == 2:
                    manual_ips.append(request.form['path2'])
            else:
                isAutomaticIp = True

            disk.orpUseFirstRange = isAutomaticIp
            disk.ISCSISubnet = int(request.form['ISCSISubnet'])

            usedACL = request.form['orpACL']
            if usedACL == "Iqn":
                disk.acl = request.form['IqnVal']
            else:
                disk.acl = ""

            usedAutentication = request.form['orpAuth']
            if usedAutentication == "Yes":
                auth_auto = False
                disk.user = request.form['UserName']
                disk.password = request.form['Password']
            else:
                auth_auto = True

            manage_config = ManageConfig()
            subnet1_info = manage_config.get_iscsi1_subnet()
            subnet2_info = manage_config.get_iscsi2_subnet()
            disk.subnet1 = subnet1_info.subnet_mask
            #########################################################
            #########################################################
            #（1）在ceph中创建该disk，并在镜像的元数据文件中添加key为petasan-metada的属性信息
            #（2）在consul.kv中添加关于该disk的资源信息
            manageDisk = ManageDisk()
            status = manageDisk.add_disk(disk, manual_ips, path_type, activePathsCount, auth_auto, isAutomaticIp)
            if status == ManageDiskStatus.done:
                # session['success'] = "ui_admin_add_disk_success"
                return redirect(url_for('disk_controller.disk_list'))

            elif status == ManageDiskStatus.done_metaNo:
                session['success'] = "ui_admin_add_disk_created_with_no_metadata"
                return redirect(url_for('disk_controller.disk_list'))

            elif status == ManageDiskStatus.error:
                session['err'] = "ui_admin_add_disk_error"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.disk_created_cant_start:
                session['warning'] = "ui_admin_add_disk_created_not_start"
                return redirect(url_for('disk_controller.disk_list'))

            elif status == ManageDiskStatus.data_missing:
                session['err'] = "ui_admin_manage_disk_data_missing"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.disk_meta_cant_read:
                session['warning'] = "ui_admin_add_disk_error_created_not_read_metadata"
                return redirect(url_for('disk_controller.disk_list'))

            elif status == ManageDiskStatus.disk_exists:
                session['err'] = "ui_admin_manage_disk_exist"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.disk_name_exists:
                session['err'] = "ui_admin_manage_disk_name_exist"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.ip_out_of_range:
                session['err'] = "ui_admin_manage_disk_no_auto_ip"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.wrong_subnet:
                session['err'] = "ui_admin_manage_disk_wrong_subnet"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.wrong_data:
                session['err'] = "ui_admin_manage_disk_subnet_count"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.used_already:
                session['err'] = "ui_admin_manage_disk_used_already"
                return redirect(url_for('disk_controller.add_disk'), 307)

            elif status == ManageDiskStatus.disk_get__list_error:
                session['err'] = "ui_admin_manage_disk_disk_get_list_error"
                return redirect(url_for('disk_controller.add_disk'), 307)

        except Exception as e:
            session['err'] = "ui_admin_add_disk_error"
            logger.error(e)
            return redirect(url_for('disk_controller.add_disk'), 307)
```
经过save_disk之后，已经分配了关于该disk的path资源，但是尚未将这些资源assign给该path，就返回成功，并且进入disk list页面，在该页面中会展示关于所有已经添加的disk的信息，包括他们的path数目信息，然后点击关于任何disk的path数目，都会展示这些path对应的节点及其ip信息，直到这些path确实assign给了该disk之后，才会进入started状态，否则状态栏一直都显示为starting状态。

那么关于disk的path的assign逻辑在哪里实现的呢？这就需要进入到iscsi_service了（见/usr/lib/python2.7/dist-packages/PetaSAN/backend/iscsi_service.py）。要注意的是，这里并不会进入到MangePathAssignment.auto或者MangePathAssignment.manual逻辑，这两个逻辑只在通过WEB进行显示地path assignment的情况下才会进入。


```
class ManageDisk:
    def __init__(self):
        pass

    def add_disk(self, disk_meta, manual_ips, path_type, paths_count, auth_auto=True, auto_ip=True, pool="rbd"):

        """
        :type path_type: PathType
        :type manual_ips: [string]
        :type paths_count: int
        :type disk_meta: DiskMeta
        """

        cfg = ManageConfig()
        paths = []
        try:
            if not disk_meta.disk_name or not disk_meta.size or type(disk_meta.size) != int:
                return ManageDiskStatus.data_missing
            elif not auth_auto and (not disk_meta.user or not disk_meta.password):
                return ManageDiskStatus.data_missing
            elif not auto_ip and int(paths_count) > 2:
                return ManageDiskStatus.wrong_data
            elif not auto_ip and int(paths_count) != len(manual_ips):
                return ManageDiskStatus.wrong_data
            elif not auto_ip:
                ip_status = cfg.validate_new_iscsi_ips(manual_ips, path_type)
                if ip_status == NewIPValidation.valid:
                    for ip in manual_ips:
                        paths.append(cfg.get_path(ip))
                elif ip_status == NewIPValidation.used_already:
                    return ManageDiskStatus.used_already
                elif ip_status == NewIPValidation.wrong_subnet:
                    return ManageDiskStatus.wrong_subnet
                else:
                    return ManageDiskStatus.wrong_data
            elif auto_ip:
                #分配paths_count个连续ip，保存在paths中
                paths.extend(cfg.get_new_iscsi_ips(path_type, paths_count))

            if not paths or len(paths) == 0:
                return ManageDiskStatus.ip_out_of_range

            #分配一个新的disk id给即将添加的disk
            ceph_api = CephAPI()
            images = ceph_api.get_disks_meta_ids(pool)
            #images = [i.replace(ConfigAPI().get_image_name_prefix(), "") for i in disk_metas]
            new_id = get_next_id(images, 5)
            disk_meta.id = new_id
            disk_meta.pool = pool
            disk_meta.paths = paths

            if auth_auto:
                disk_meta.user = ""
                disk_meta.password = ""

            #设置disk的iqn为iqn_prefix + diskid
            disk_meta.iqn = ":".join([cfg.get_iqn_base(), new_id])
            #检查consul.kv中是否已经存在该disk，如果存在则返回该disk资源
            consul_api = ConsulAPI()
            disk_data = consul_api.find_disk(disk_meta.id)

            #如果disk已经存在，则报错
            if disk_data is not None:
                return ManageDiskStatus.disk_exists
            else:
                #在ceph中创建该disk，并在镜像的元数据文件中添加key为petasan-metada的属性信息（包括
                #id,disk_name,size,create_data,acl,user,password,iqn,pool,paths等信息）
                status = ceph_api.add_disk(disk_meta, True, pool)
                #如果在ceph中成功创建该disk，则在consul.kv中添加关于该disk的信息（disk及其所有paths）
                if status == ManageDiskStatus.done:
                    #在consul.kv中添加关于该disk的资源
                    consul_api.add_disk_resource(disk_meta.id, "disk")
                    i = 0
                    #在consul.kv中添加关于该disk的所有paths
                    for p in paths:
                        i += 1
                        consul_api.add_disk_resource("/".join(["", disk_meta.id, str(i)]), None)

        except DiskListException as e:
            status = ManageDiskStatus.disk_get__list_error
            logger.exception(e.message)
        except Exception as e:
            status = ManageDiskStatus.error
            logger.exception(e.message)
        return status
```

```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/manage_config.py):

class ManageConfig:
    def get_new_iscsi_ips(self, path_type, paths_count):

        """
        :rtype paths_list: [Path]
        :type path_type: PathType
        """

        #通过ceph中所有镜像的元数据获取所有已经使用的ips
        ips_used = set()
        # get the image metada from ceph
        ceph_api = CephAPI()
        for i in ceph_api.get_disks_meta():
            for p in i.get_paths():
                ips_used.add(p.ip)

        iscsi1_subnet = self.get_iscsi1_subnet()
        iscsi2_subnet = self.get_iscsi2_subnet()

        eth1 = configuration().get_cluster_info().iscsi_1_eth_name
        eth2 = configuration().get_cluster_info().iscsi_2_eth_name

        path_list = []

        #从iscsi subnet1中分配（paths_count个连续）ip
        if path_type == PathType.iscsi_subnet1:
            gen1 = IPAddressGenerator(iscsi1_subnet.auto_ip_from, iscsi1_subnet.auto_ip_to)
            while gen1.has_next():
                ip1 = gen1.get_next()
                if (not ip1 in ips_used):
                    path_list.append(self.get_path(ip1))
                    if len(path_list) == paths_count:
                        return path_list
                else:
                    path_list = []

            # return None
            return []

        #从iscsi subnet2中分配（paths_count个连续）ip
        if path_type == PathType.iscsi_subnet2:
            gen2 = IPAddressGenerator(iscsi2_subnet.auto_ip_from, iscsi2_subnet.auto_ip_to)
            while gen2.has_next():
                ip2 = gen2.get_next()
                if (not ip2 in ips_used):
                    path_list.append(self.get_path(ip2))
                    if len(path_list) == paths_count:
                        return path_list
                else:
                    path_list = []

            # return None
            return []

        #从iscsi subnet1和iscsi subnet2中交替分配（paths_count个连续）ip
        if path_type == PathType.both:

            gen1 = IPAddressGenerator(iscsi1_subnet.auto_ip_from, iscsi1_subnet.auto_ip_to)
            gen2 = IPAddressGenerator(iscsi2_subnet.auto_ip_from, iscsi2_subnet.auto_ip_to)

            while gen1.has_next() and gen2.has_next():
                # we increment both ranges together so they remain related
                ip1 = gen1.get_next()
                ip2 = gen2.get_next()
                if ((not ip1 in ips_used) and (not ip2 in ips_used)):

                    path_list.append(self.get_path(ip1))
                    if len(path_list) == paths_count:
                        return path_list

                    path_list.append(self.get_path(ip2))
                    if len(path_list) == paths_count:
                        return path_list

                else:
                    path_list = []

            # return None
            return []


        # return None
        return []    
```


```
(/usr/lib/python2.7/dist-packages/PetaSAN/core/ceph/api.py):

class CephAPI:
    conf_api = ConfigAPI()
    def __init__(self):
        pass

    def add_disk(self, disk_meta, create_disk=True,
                pool="rbd",
                cureate_date=datetime.datetime.now()):
        """

        :type disk_meta: DiskMeta
        """
        status = ManageDiskStatus.error
        cluster = self.connect()
        disk = DiskMeta()
        if cluster != -1:

            try:
                io_ctx = cluster.open_ioctx(pool)
                status = 0
                #用于镜像的CRUD操作
                rbd_inst = rbd.RBD()
                disk_metas = self.get_disks_meta()
                disk_name_is_exists = False

                for i in disk_metas:
                        if i.disk_name == disk_meta.disk_name:
                            #disk已经存在
                            disk_name_is_exists = True
                            break
                            
                if create_disk:
                    size = disk_meta.size * 1024 ** 3
                    if disk_name_is_exists:
                        #在create_disk为true的情况下，如果disk已经存在，则报错
                        status = ManageDiskStatus.disk_name_exists

                    if status != ManageDiskStatus.disk_name_exists:
                        #创建镜像
                        rbd_inst.create(io_ctx,self.conf_api.get_image_name_prefix() + disk_meta.id, size,old_format=False)
                        logger.info("Disk %s created" % disk_meta.disk_name)
                        status = ManageDiskStatus.done_metaNo
                if not create_disk:
                        #self.conf_api.get_image_name_prefix() + disk_meta.id表示镜像文件名， #获取镜像的所有属性信息
                        attr = self.read_image_metadata(self.conf_api.get_image_name_prefix() + disk_meta.id)
                        #获取key为petasan-metada的属性
                        petasan = attr.get(self.conf_api.get_image_meta_key())
                        if petasan:
                            disk.load_json(petasan)
                        if disk_meta.disk_name is not None and disk_meta.disk_name != disk.disk_name and disk_name_is_exists:
                            status = ManageDiskStatus.disk_name_exists

                if (status == ManageDiskStatus.done_metaNo or not create_disk) and status != ManageDiskStatus.disk_name_exists:
                    #成功
                    #返回镜像对应的属性文件名
                    attr_object = self.__get_meta_object(self.conf_api.get_image_name_prefix() + disk_meta.id,pool)

                    if create_disk:
                        disk.id = disk_meta.id
                        disk.disk_name = disk_meta.disk_name
                        disk.size = disk_meta.size
                        disk.create_date = str(cureate_date.date())
                    else:  # edit or attach or detach
                        if not disk.id or len(disk.id)==0:
                            disk.id = disk_meta.id
                        if disk_meta.disk_name is not None :# edit and attach
                            disk.disk_name = disk_meta.disk_name
                        else:  # detach
                            disk.disk_name = ""
                        if disk_meta.size is not None:  # edit and attach
                            if disk_meta.size > disk.size:
                                size = disk_meta.size * 1024 ** 3
                                rbd.Image(io_ctx, str(self.conf_api.get_image_name_prefix() + disk_meta.id)).resize(size)
                                disk.size = disk_meta.size
                    disk.acl = disk_meta.acl
                    disk.user = disk_meta.user
                    disk.password = disk_meta.password
                    disk.iqn = disk_meta.iqn
                    disk.pool = disk_meta.pool
                    disk.paths=disk_meta.paths
                    #在镜像的元数据文件中添加key为petasan-metada的属性信息（包括id,disk_name,size,create_data,acl,user,password,iqn,pool,paths等信息）
                    io_ctx.set_xattr(str(attr_object), str(self.conf_api.get_image_meta_key()), disk.write_json())
                    status = ManageDiskStatus.done
                io_ctx.close()

            except MetadataException as e:
                status = ManageDiskStatus.disk_meta_cant_read
            except DiskListException as e:
                status = ManageDiskStatus.disk_get__list_error
                logger.exception(e.message)
            except rbd.ImageExists as e:
                logger.error("Error while creating image, image %s already exists." % disk_meta.id)
                status = ManageDiskStatus.disk_exists
            except Exception as e:
                if status == -1:
                    logger.error("Error while creating image %s, cannot connect to cluster." % disk_meta.disk_name)
                elif status == 2:
                    logger.error(
                        "Error while creating image %s, could not add metadata." % disk_meta.id)
                elif status == 1:
                    logger.warning(
                        "Error while creating image %s, connection not closed" % disk_meta.id)

                logger.exception(e.message)

            cluster.shutdown()
            return status
        else:
            return -1
```

```
class ConsulAPI:
    root=ConfigAPI().get_consul_disks_path()
    assignment_path = ConfigAPI().get_consul_assignment_path()
    def __int__(self):
        pass

    def add_disk_resource(self, resource, data, flag=None, current_index=0):
        consul_obj = Consul()
        status = ManageDiskStatus.error
        try:
            result = consul_obj.kv.put(self.root + resource, data, current_index, flag)
            if result:
                status = ManageDiskStatus.done
                logger.info("Successfully created key %s for new disk." % resource)
            else:
                status = ManageDiskStatus.disk_exists
                logger.error("Could not create key %s for new disk, key already exists." % resource)
        except Exception as e:
            status = ManageDiskStatus.error
            logger.error("Could not create key %s for new disk." % resource)
            logger.exception(e.message)
        return status
```
