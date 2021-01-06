# 提纲
[toc]

# 源码分析
```
(/opt/petasan/services/iscsi_service.py):

from PetaSAN.backend.iscsi_service import Service

app_service = Service()
app_service.start()
```

```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/iscsi_service.py):

'''
 Copyright (C) 2016 Maged Mokhtar <mmokhtar <at> petasan.org>
 Copyright (C) 2016 PetaSAN www.petasan.org


 This program is free software; you can redistribute it and/or
 modify it under the terms of the GNU Affero General Public License
 as published by the Free Software Foundation

 This program is distributed in the hope that it will be useful,
 but WITHOUT ANY WARRANTY; without even the implied warranty of
 MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 GNU Affero General Public License for more details.
'''

from PetaSAN.backend.maintenance import ManageMaintenance
from PetaSAN.backend.mange_path_assignment import MangePathAssignment
from PetaSAN.core.consul.ps_consul import RetryConsulException
#from ceph_disk import main as ceph_disk
import threading
from time import sleep
import math
from PetaSAN.core.ceph.api import CephAPI
from PetaSAN.core.cluster.configuration import configuration
from PetaSAN.core.cluster.network import Network
from PetaSAN.core.common.log import logger
from PetaSAN.core.config.api import ConfigAPI
from PetaSAN.core.consul.api import ConsulAPI
from PetaSAN.core.entity.disk_info import DiskMeta
from PetaSAN.core.lio.api import LioAPI
from PetaSAN.core.lio.network import NetworkAPI
from requests.exceptions import ConnectionError
from PetaSAN.core.common.enums import Status, MaintenanceMode, MaintenanceConfigState, ReassignPathStatus
from PetaSAN.core.ssh.ssh import ssh
from datetime import date, datetime,timedelta
import random


class Service:
    __cluster_info = configuration().get_cluster_info()
    __node_info = configuration().get_node_info()
    __app_conf = ConfigAPI()
    __session_name = ConfigAPI().get_iscsi_service_session_name()
    __paths_local = set()
    __session = '0'
    __paths_per_disk_local = dict()
    __paths_per_session = dict()
    __total_cluster_paths = 0
    __iqn_tpgs = dict()
    __local_ips = set()
    __backstore = set()
    __current_lock_index = None
    __image_name_prefix = ""
    __cluster_info = configuration().get_cluster_info()
    __node_info = configuration().get_node_info()
    __exception_retry_timeout = 0
    __failure_timeout = timedelta(minutes=5) + datetime.utcnow()
    __acquire_warning_counter = 0
    __last_acquire_succeeded = True
    __paths_consul_unlocked_firstborn = dict()
    __paths_consul_unlocked_siblings = dict()
    __paths_consul_locked_node = set()
    __disk_consul_stopped = set()
    __ignored_acquire_paths = dict()
    __force_acquire_paths = dict()

    is_service_running = False

    def __init__(self):
        if Service.is_service_running:
            logger.error("The service is already running.")
            raise Exception("The service is already running.")
        Service.is_service_running = True

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
                    self.__session = ConsulAPI().get_new_session_ID(self.__session_name,self.__node_info.name)

                consul_api = ConsulAPI()
                self.__current_lock_index = consul_api.current_index()
                if not self.__current_lock_index:
                    sleep(1)
                    continue
                #######################################
                #######################################
                self.__process()
                old_index = self.__current_lock_index
                self.__current_lock_index = consul_api.watch(self.__current_lock_index)
                #disk资源视图发生改变
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

```
    def __process(self):
        logger.debug("Start process, node session id is {}.".format(self.__session))
        self.__last_acquire_succeeded = True
        self.__ignored_acquire_paths = dict()
        while self.__do_process() != True:
            pass
        logger.debug("End process.")

```

```
    def __do_process(self):
        self.__paths_local = set()
        self.__paths_per_disk_local = dict()
        self.__paths_per_session = dict()
        self.__iqn_tpgs = dict()
        self.__local_ips = set()
        self.__backstore = set()
        self.__paths_consul_unlocked_firstborn = dict()
        self.__paths_consul_unlocked_siblings = dict()
        self.__paths_consul_locked_node = set()
        self.__disk_consul_stopped = set()
        self.__force_acquire_paths = dict()

        self.__read_resources_local()
        self.__read_resources_consul()

        state_change = False

        # ====== Step 1: delete any local paths not locked by us in consul ======
        for path in self.__paths_local:
            if path not in self.__paths_consul_locked_node:
                state_change = True
                self.__clean_local_path(path)

        if state_change:
            logger.info("PetaSAN cleaned local paths not locked by this node in consul.")
            return False  # refresh and reprocess


        # ====== Step 2: remove any consul locks we have but not configured locally  ======
        for path in self.__paths_consul_locked_node:
            if path not in self.__paths_local:
                state_change = True
                self.__unlock_consul_path(path)

        if state_change:
            logger.info("PetaSAN unlocked any consul locks not configured in this node.")
            return False  # refresh and reprocess

        # ====== Step 3: handle stopped disks  ======
        for disk in self.__disk_consul_stopped:
            self.__stop_disk(disk)

        # ====== Step 4: Clean any unused iqns ======
        # clean those iqns with none of its target portal group in enabled state
        if self.__clean_unused_iqns():
            logger.info("PetaSAN cleaned iqns.")
            return False  # refresh and reprocess

        # ====== Step 5: Clean any unused rbd backstores ======
        if self.__clean_unused_rbd_backstore():
            logger.info("PetaSAN Cleaned rbd backstores.")
            return False  # refresh and reprocess

        # ====== Step 6: Clean any unused ips ======
        self.__clean_unused_ips()

        # ====== Step 7: Clean any unused mapped rbd images ======
        self.__clean_unused_rbd_images()

        # ====== Step 8: try to acquire unlocked  paths  ======
        if len(self.__force_acquire_paths) > 0:
            path,value = self.__force_acquire_paths.items()[0]
            if path:
                self.__acquire_path(str(path), value)
                return False

        if len(self.__paths_consul_unlocked_firstborn) > 0:
            path = random.sample(self.__paths_consul_unlocked_firstborn, 1)[0]
            self.__wait_before_lock(path)
            self.__acquire_path(str(path), self.__paths_consul_unlocked_firstborn.get(path))
            return False

        if len(self.__paths_consul_unlocked_siblings) > 0:
            path = random.sample(self.__paths_consul_unlocked_siblings, 1)[0]
            self.__wait_before_lock(path)
            self.__acquire_path(str(path), self.__paths_consul_unlocked_siblings.get(path))
            return False

        return True

```

```
    #借助于lio api来获取本地的资源，返回本地通过LIO可以看到的所有paths（包括disabled状态的path）和其对应的ips
    def __read_resources_local(self):
        logger.debug("Start read local resources.")
        lio_api = LioAPI()
        try:
            self.__backstore = lio_api.get_backstore_image_names()
            self.__iqn_tpgs = lio_api.get_iqns_with_enabled_tpgs()
            for iqn, tpgs in self.__iqn_tpgs.iteritems():
                disk_id = str(iqn).split(":")[1]
                for tpg_index, ips in tpgs.iteritems():
                    self.__paths_local.add("/".join([disk_id, str(tpg_index)]))
                    if ips and len(ips) > 0:
                        for ip in ips:
                            self.__local_ips.add(ip)
        except Exception as e:
            logger.error("Could not read consul resources.")
            raise e
        logger.debug("End read local resources.")

```

```
    #借助于consul api来获取全局资源
    def __read_resources_consul(self):
        logger.debug("Start read resources consul.")
        self.__paths_per_session = {}
        self.__total_cluster_paths = 0
        unlock_kvs= set()
        consul_api = ConsulAPI()
        try:
            disk_kvs = consul_api.get_disk_kvs()
            #依次遍历disk_kvs中的每一个disk，获取如下信息：
            #self.__disk_consul_stopped -》处于停止状态的disk集合
            #self.__total_cluster_paths -》cluster中的paths数目
            #self.__paths_consul_locked_node -》被当前node的当前session lock住的paths
            #self.__paths_per_disk_local -》每一个disk被当前节点lock住的path数目
            #self.__paths_per_session -》每一个session中paths数目
            #unlock_kvs -》未被任何node的任何sessionlock住的disk或者disk path
            for kv in disk_kvs:
                key = str(kv.Key).replace(self.__app_conf.get_consul_disks_path(), "")
                disk_id = str(key).split('/')[0]
                if disk_id in self.__disk_consul_stopped:
                    continue
                #kv.value可能是"disk"，也可能是none，见ManageDisk.add_disk()
                if kv.Value == "disk":
                    disk_id = str(key).split('/')[0]
                    self.__paths_per_disk_local[disk_id] = 0
                    #kv.Flags为1，表示该disk被删除，见ManageDisk.stop()
                    if str(kv.Flags) == "1":
                        self.__disk_consul_stopped.add(disk_id)
                    continue
                # Count paths in the cluster.
                self.__total_cluster_paths += 1

                if hasattr(kv, "Session"):
                    # locked paths
                    if kv.Session == self.__session:
                        self.__paths_consul_locked_node.add(key)
                        disk_paths_count = self.__paths_per_disk_local.get(disk_id, 0) + 1
                        self.__paths_per_disk_local[disk_id] = disk_paths_count
                    # Total count of paths for each session
                    if self.__paths_per_session.has_key(kv.Session):
                        count = self.__paths_per_session.get(kv.Session)
                        self.__paths_per_session[kv.Session] = count + 1
                    else:
                        self.__paths_per_session[kv.Session] = 1
                # unlocked paths
                elif not hasattr(kv, "Session"):
                    unlock_kvs.add(kv)
            # Filter unlocked paths
            reassignments = None
            if len(unlock_kvs) > 0:
                #见MangePathAssignment.get_forced_paths()
                reassignments = MangePathAssignment().get_forced_paths()
            for kv in unlock_kvs:
                key = str(kv.Key).replace(self.__app_conf.get_consul_disks_path(), "")
                if reassignments:
                    path_assignment_info = reassignments.get(key)
                    if path_assignment_info and path_assignment_info.target_node == self.__node_info.name:
                        self.__force_acquire_paths[key] = kv
                        continue
                    else:
                        self.__ignored_acquire_paths[key] = kv
                        continue

                disk_id = str(key).split('/')[0]
                if self.__paths_per_disk_local.get(disk_id,0) > 0:
                    #关于key的disk的所有paths中至少有一个path被当前节点的当前session lock住
                    self.__paths_consul_unlocked_siblings[key] = kv
                else:
                    #关于key的disk的所有paths都未被当前节点的当前session lock
                    self.__paths_consul_unlocked_firstborn[key] = kv
        except Exception as e:
            logger.error("Could not read consul resources.")
            logger.exception(e)
            raise e
        logger.debug("End read resources consul.")

    def __clean_local_path(self, path, pool="rbd"):
        disk_id, path_index = str(path).split("/")
        logger.debug("Start clean disk path {}.".format(path))
        image_name = self.__image_name_prefix + str(disk_id)
        ceph_api = CephAPI()
        lio_api = LioAPI()
        network_api = NetworkAPI()

        try:
            # Get iqn.
            logger.debug("Start get disk meta to clean path {}.".format(path))
            iqn = ceph_api.get_disk_meta(disk_id, pool).iqn
            logger.debug("End get disk meta to clean path {}.".format(path))
            # Get tpgs for iqn.
            tpgs = self.__iqn_tpgs.get(iqn, None)
            if not iqn or not tpgs or len(tpgs) == 0:
                logger.info("Could not find ips for %s " % image_name)
            # Remove the assigned ips from our interfaces
            elif tpgs and len(tpgs) > 0:
                # Get assigned ips for each path.
                for tpg, ips in tpgs.iteritems():
                    if tpg == path_index:
                        for ip in ips:
                            logger.debug("Delete ip {} to clean path {}.".format(ip, path))
                            if not network_api.delete_ip(ip, self.__cluster_info.iscsi_1_eth_name):
                                network_api.delete_ip(ip, self.__cluster_info.iscsi_2_eth_name)

                        lio_api.disable_path(iqn, path_index)
                        logger.info("Cleaned disk path {}.".format(path))
                        break
        except Exception as e:
            logger.error("Could not clean disk path for %s" % image_name)
            raise e
        logger.debug("End clean disk path {}.".format(path))
        return

    # If all tpgs related to iqn are disable, system will remove iqn.
    def __clean_unused_iqns(self, pool="rbd"):
        status = False
        lio_api = LioAPI()
        for iqn in lio_api.get_unused_iqns():
            disk_id = str(iqn).split(":")[1]
            image_name = self.__image_name_prefix + str(disk_id)
            lio_api.delete_target(image_name, iqn)
            CephAPI().unmap_image(image_name, pool)
            status = True
            logger.debug("Clean unused iqn {}.".format(iqn))
        return status

    def __clean_unused_rbd_backstore(self):
        status = False
        iqns = self.__iqn_tpgs.keys()
        for rbd_backstore in self.__backstore:
            rbd_backstore_disk_id = str(rbd_backstore).replace(self.__image_name_prefix, "")
            is_used = False
            for iqn in iqns:
                disk_id = str(iqn).split(":")[1]
                if disk_id == rbd_backstore_disk_id:
                    is_used = True
                    break
            if not is_used:
                LioAPI().delete_backstore_image(rbd_backstore)
                logger.debug("Clean unused lio backstore {}.".format(rbd_backstore))
                status = True
        return status

    def __clean_unused_ips(self):
        ips = Network().get_all_configured_ips()
        for ip, eth_name in ips.iteritems():
            ip, netmask = str(ip).split("/")
            if ip not in self.__local_ips and ip != self.__node_info.backend_1_ip and \
                            ip != self.__node_info.backend_2_ip and ip != self.__node_info.management_ip:
                NetworkAPI().delete_ip(ip, eth_name, netmask)
                logger.debug("Clean unused ip {} on interface {}.".format(ip, eth_name))

    def __clean_unused_rbd_images(self, pool="rbd"):
        ceph_api = CephAPI()
        rbd_images = ceph_api.get_mapped_images(pool)
        for image, mapped_count in rbd_images.iteritems():
            if image not in self.__backstore:
                if int(mapped_count) > 0:
                    for i in range(0, int(mapped_count)):
                        ceph_api.unmap_image(image, pool)
                        logger.debug("Unmapped unused image {}.".format(image))

    def __unlock_consul_path(self, path, pool="rbd"):
        try:
            logger.debug("Unlock {} path locked by session {}.".format(path, self.__session))
            consul_api = ConsulAPI()
            consul_api.release_disk_path(self.__app_conf.get_consul_disks_path() + path, self.__session, None)
            logger.info("Unlock path %s" % path)
        except Exception as e:
            logger.error("Could not unlock path %s" % path)
            raise e

    def __stop_disk(self, disk_id, pool="rbd"):
        consul_api = ConsulAPI()
        ceph_api = CephAPI()
        lio_api = LioAPI()
        network_api = NetworkAPI()
        logger.info("Stopping disk %s" % disk_id)
        image_name = self.__image_name_prefix + str(disk_id)

        try:
            # Get iqn.
            iqn = ceph_api.get_disk_meta(disk_id, pool).iqn
            # Get tpgs for iqn.
            tpgs = self.__iqn_tpgs.get(iqn, None)
            if not iqn or not tpgs or len(tpgs) == 0:
                logger.error("Could not find ips for %s " % image_name)
            # Remove the assigned ips from our interfaces
            elif tpgs and len(tpgs) > 0:
                # Get assigned ips for each path.
                for tpg, ips in tpgs.iteritems():
                    for ip in ips:
                        if not network_api.delete_ip(ip, self.__cluster_info.iscsi_1_eth_name):
                            network_api.delete_ip(ip, self.__cluster_info.iscsi_2_eth_name)

            lio_api.delete_target(image_name, iqn)
            ceph_api.unmap_image(image_name, pool)
            sleep(2)
            if not ceph_api.is_image_busy(image_name):
                consul_api.delete_disk(self.__app_conf.get_consul_disks_path() + disk_id, None, True)
                logger.info("PetaSAN removed key of stopped disk {} from consul.".format(disk_id))
        except Exception as e:
            logger.info("Could not stop  disk %s" % disk_id)
        return

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
            result = consul_api.lock_disk_path(self.__app_conf.get_consul_disks_path() + path, self.__session,node_name,
                                               str(consul_kv.CreateIndex))
            if not result:
                logger.info("Could not lock path {} with session {}.".format(path, self.__session))
            elif result:
                if consul_kv.Value != None and len(str(consul_kv.Value))>0 and  node_name != str(consul_kv.Value):
                    logger.info("The path {} was locked by {}.".format(path,str(consul_kv.Value)))
                    logger.debug("Node {} will kill node {}.".format(config.get_node_name(),str(consul_kv.Value)))
                    self.__fencing(str(consul_kv.Value))

                # we locked it
                if disk_meta.paths:
                    # if lio has the image name in its backstore already, do not perform rbd mapping
                    if image_name not in self.__backstore:
                        status = ceph_api.map_iamge(image_name)
                    else:
                        status = Status.done
                    if Status.done == status:
                        # Get path info from metadata
                        path_obj = disk_meta.get_paths()[int(path_index) - 1]
                        # add path ips to our network interfaces
                        network_api.add_ip(path_obj.ip, path_obj.subnet_mask, path_obj.eth)
                        #update neighbors arp table
                        network_api.update_neighbors_arp(path_obj.ip,path_obj.eth)
                        # add new target in lio if not there already
                        if not lio_api.is_backstore_image_found(image_name):
                            # Give ceph map image complete it job
                            sleep(3)
                            # Add rbd backstores and target
                            status = lio_api.add_target(disk_meta, disk_meta.pool)
                            """
                            wwn = self.calculate_disk_wwn(disk_meta)
                            status = lio_api.add_target(disk_meta, wwn, disk_meta.pool)
                            """
                        if Status.done == status:
                            # enable the path we locked to true
                            self.__last_acquire_succeeded = True
                            lio_api.enable_path(disk_meta.iqn, path_index, True)
                            logger.info("Path %s acquired successfully" % path)

                            if self.__acquire_warning_counter > 2:
                                logger.info("PetaSAN finally succeeded to acquire path after retrying {} times.".
                                            format(self.__acquire_warning_counter))
                                self.__acquire_warning_counter = 0
                            path_assignment_info = self.__force_acquire_paths.get(path)

                            if path_assignment_info:
                                logger.info("++++++++++++1++++++++++++")
                                MangePathAssignment().update_path(path_obj.ip,ReassignPathStatus.succeeded)
                        else:
                            path_assignment_info = self.__force_acquire_paths.get(path)
                            if path_assignment_info:
                                logger.info("Acquired forced path {}".format(path))
                                MangePathAssignment().update_path(path_obj.ip,ReassignPathStatus.failed)
                            self.__last_acquire_succeeded = False
                            if self.__acquire_warning_counter > 2:
                                logger.warning("PetaSAN failed to acquire path after {} times.".
                                               format(self.__acquire_warning_counter))
                                self.__acquire_warning_counter += 1
                            logger.error("Error could not acquire path %s" % path)

                    else:
                        self.__unlock_consul_path(path)

        except Exception as e:
            if str(e.message).find("invalid session") > -1:
                logger.error("Session is invalid")
                try:
                    logger.info("Trying to create new session id")
                    self.__session = ConsulAPI().get_new_session_ID(self.__session_name,self.__node_info.name)
                    logger.info("New session id is {}".format(self.session))
                    logger.info("Cleaning all mapped disks from old session")
                    self.__clean()
                except Exception as ex:
                    logger.exception(ex)
            logger.exception("Could not acquire path %s" % path)
            raise e
        logger.debug("End acquire path {}.".format(path))
        return

    def __clean(self, pool="rbd"):
        logger.info("Cleaning unused configurations. ")
        logger.info("Cleaning all mapped disks")
        ceph_api = CephAPI()
        lio_api = LioAPI()
        network_api = NetworkAPI()
        # Get tpgs of each iqn
        for iqn, tpgs in lio_api.get_iqns_with_tpgs().iteritems():
            try:
                disk_id = str(iqn).split(":")[1]
                # Get assigned ips for each tpg
                for tpg, ips in tpgs.iteritems():
                    if ips and len(ips) > 0:
                        for ip in ips:
                            # 1- Remove ip from network interface.
                            if not network_api.delete_ip(ip, self.__cluster_info.iscsi_1_eth_name):
                                network_api.delete_ip(ip, self.__cluster_info.iscsi_2_eth_name)

                # 2- Delete iqn ,delete image from rbd backstore and unmap image.
                image_name = self.__image_name_prefix + str(disk_id)
                lio_api.delete_target(image_name, iqn)
                ceph_api.unmap_image(image_name, pool)

            except Exception as e:
                logger.error("Error cleaning all mapped disks, disk %s " % image_name)
                logger.exception(e.message)
        # 3- From backstore
        for image_name in lio_api.get_backstore_image_names():
            try:
                lio_api.delete_backstore_image(image_name)
                ceph_api.unmap_image(image_name, pool)
            except Exception as e:
                logger.error("Error cleaning all mapped disks, disk %s " % image_name)

        logger.info("Cleaning unused rbd images.")
        try:
            self.__clean_unused_rbd_images()
        except:
            logger.error("Error cleaning unused rbd images.")

        logger.info("Cleaning unused ips.")
        try:
            self.__local_ips = set()
            self.__clean_unused_ips()
        except:
            logger.error("Cleaning unused ips.")

    def __wait_before_lock(self, path=None):

        disk_id, path_index = str(path).split("/")
        wait_time = 0
        if path:
            # 1- Calc wait time if path has siblings.
            wait_time = int(self.__app_conf.get_siblings_paths_delay()) * int(
                self.__paths_per_disk_local.get(disk_id, 0))

        logger.debug("Wait time for siblings is {}.".format(wait_time))
        total_nodes = len(ConsulAPI().get_consul_members())
        # 2- Calc average paths per node.
        average_node_paths = float(self.__total_cluster_paths) / float(total_nodes)
        # Calc the percent of local paths according to average paths.
        percent = float(self.__paths_per_session.get(self.__session, 0)) / average_node_paths
        # 3- Calc total wait time
        if self.__last_acquire_succeeded:
            wait_time += int(self.__app_conf.get_average_delay_before_lock()) * percent
        else:
            logger.debug("Skipping wait time for average delay.")
        logger.debug("Wait time depending on average and siblings is {}.".format(math.ceil(wait_time)))
        sleep(math.ceil(wait_time))

    def __wait_after_lock(self):
       pass

    def __fencing(self,node_name):
        maintenance = ManageMaintenance()
        if maintenance.get_maintenance_config().fencing == MaintenanceConfigState.off:
            logger.warning("Fencing action will not fire the admin stopped it,the cluster is in maintenance mode.")
            return

        node_list = ConsulAPI().get_node_list()
        for node in node_list:

            if str(node.name) == node_name:
                if Network().ping(node.backend_2_ip):
                    logger.info("This node will stop node {}/{}.".format(node_name, node.backend_2_ip))
                    ssh().call_command(node.backend_2_ip, " poweroff ", 5)
                    break
                elif Network().ping(node.management_ip):
                    logger.info("This node will stop node {}/{}.".format(node_name, node.management_ip))
                    ssh().call_command(node.management_ip, " poweroff ", 5)
                    break
                elif Network().ping(node.backend_1_ip):
                    logger.info("This node will stop node {}/{}.".format(node_name, node.backend_1_ip))
                    ssh().call_command(node.backend_1_ip, " poweroff ", 5)
                    break

    def handle_cluster_startup(self):
        i = 0
        consul_api = ConsulAPI()
        logger.debug("Check cluster startup.")
        while True:
            try:

                current_node_name = self.__node_info.name
                result = consul_api.set_leader_startup_time(current_node_name, str(i))
                if i == 0 and not result:
                    sleep(2)
                    continue
                elif result:
                    # value returned, consul is up and running
                    sleep(2)
                    number_of_started_nodes = 0
                    for kv in consul_api.get_leaders_startup_times():
                        node_name = str(kv.Key).replace(ConfigAPI().get_consul_leaders_path(), "")
                        if node_name != current_node_name:
                            if int(kv.Value) == 0:
                                number_of_started_nodes += 1

                    logger.debug("Number of started nodes = {}.".format(number_of_started_nodes))
                    # Another management node is just starting
                    if i == 0 and number_of_started_nodes > 0:
                        logger.info("Cluster is just starting, system will delete all active disk resources")
                        consul_api.delete_disk(ConfigAPI().get_consul_disks_path(), recurse=True)
                i += 1
                sleep(58)

            except Exception as ex:
                logger.debug("Start up error")
                logger.exception(ex)
                # maybe other management nodes are starting, give them a chance to start
                if i == 0:
                    sleep(2)
                else:
                    i += 1
                    sleep(58)
```


""" # access two isci disks as one so change wwn to be unique
    # change wwn disk identifier value cluster fsid exclude last part to append disk_meta id
    def calculate_disk_wwn(self, disk_meta):
        fsid = ceph_disk.get_fsid(configuration().get_cluster_name())
        fsid_split = fsid[: -disk_meta.id]
        wwn = fsid_split + disk_meta.id
        return wwn
"""


# ARP欺骗原理
在[这里](http://www.ixueshu.com/document/d8915f732710eb9d318947a18e7f9386.html)谈到利用ARP欺骗原理来实现IP地址接管，在需要切换iscsi target服务到备端iscsi target的时候，在备端iscsi target上触发一个进程，广播一个ARP应答，使用主端iscsi target的IP地址和备端iscsi target的MAC地址来更新iscsi initiator的ARP缓存，因为在通信过程中，真正识别的是MAC地址，从而iscsi initiator和iscsi target的通信能够成功转移到备端。
在/usr/lib/python2.7/dist-package/PetaSAN/backend/iscsi_service.py中的__acquire_path方法中，会调用network_api.update_neighbors_arp(path_obj.ip,path_obj.eth)，难道就是这个目的？？？？

