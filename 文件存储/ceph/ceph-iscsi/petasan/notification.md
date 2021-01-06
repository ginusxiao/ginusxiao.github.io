# notification机制
当选举出leader节点之后，会在leader节点上执行systemctl start petasan-notification，petasan-notification.service如下:
```
(/lib/systemd/system/petasan-notification.service):

[Unit]
Description= PetaSAN Notification Service
After=syslog.target local-fs.target network.target

[Service]
Type=simple
ExecStart=/opt/petasan/scripts/start_notification_service.py
Restart=always
RestartSec=3
```

```
(/opt/petasan/scripts/start_notification_service.py):

from PetaSAN.backend.notification_manager import Service
from PetaSAN.core.entity.app_config import AppConfig
from PetaSAN.core.config.api import ConfigAPI

__upgrade_email_config()
#启动notification service
Service().start()
```


```
(/usr/lib/python2.7/dist-packages/PetaSAN/backend/notification_manager.py):

checks_modules = ['PetaSAN.core.notification.plugins.check.cluster',
                  'PetaSAN.core.notification.plugins.check.node',
                  'PetaSAN.core.notification.plugins.check.osd']

notify_modules = ['PetaSAN.core.notification.plugins.notify.message']

class Service:
    def __init__(self, sleep_time=60):
        """
        :type sleep_time: int
        """
        self.__context = NotifyContext()
        self.__sleep_time = sleep_time
        self.__count_of_notify_plugins = -1
        pass

    def start(self):
        self.__process()

    def __process(self):
        while True:
            try:
                #检查（node failue，osd failure and storage approaching full capacity）， #检查结果会存放在self.__context中
                self.__do_check()
                #邮件通知，从self.__context中获取检查结果，并发送给指定的email
                self.__do_notify()
            except Exception as ex:
                logger.exception(ex.message)
            sleep(self.__sleep_time)


    def __do_check(self):
        #根据checks_modules定义可知，目前只支持关于cluster，node和osd的健康检查
        threads_list = []

        #依次为cluster，node和osd启动健康检查线程
        for plugin in self.__get_new_plugins_instances(checks_modules):
            if plugin.is_enable:
                plugin.get_plugin_name()
                #对于cluster来说，线程执行函数为ClusterStorageSizePlugin.run()
                #对于node来说，线程执行函数为NodeDownPlugin.run()
                #对于osd来说，线程执行函数为OSDDownPlugin.run()
                th = Thread(target=plugin.run)
                th.start()
                threads_list.append(th)

        for t in threads_list:
            t.join()

    def __do_notify(self):
        #目前只支持基于Email的notification
        threads_list = []
        plugins = self.__get_new_plugins_instances(notify_modules)
        if self.__count_of_notify_plugins > -1:
            self.__clean_context_results(self.__count_of_notify_plugins)
        for plugin in plugins:
            if plugin.is_enable:
                plugin.get_plugin_name()
                th = Thread(target=plugin.notify)
                th.start()
                threads_list.append(th)

        for t in threads_list:
            t.join()


        self.__count_of_notify_plugins =len(plugins)

    def __clean_context_results(self, plugins_count):
        ......

    def __get_new_plugins_instances(self, modules):
        ......

```

对于cluster容量检查，线程执行函数为ClusterStorageSizePlugin.run()

```
class ClusterStorageSizePlugin(CheckBasePlugin):
    def __init__(self, context):
        self.__context = context

    def is_enable(self):
        return True

    def run(self):
        try:
            #通过ceph API获取cluster中used的容量是否超过设定的阈值，如果超过阈值，则记录下来
            result = Result()
            ceph_api = CephAPI()
            cluster_status = ceph_api.get_ceph_cluster_status()
            if cluster_status is not None:
                cluster_status = json.loads(cluster_status)
                available_size = cluster_status['pgmap']['bytes_avail'] * 100.0 / cluster_status['pgmap']['bytes_total']
                used_size = cluster_status['pgmap']['bytes_used'] * 100.0 / cluster_status['pgmap']['bytes_total']
                notify_cluster_space_percent = ConfigAPI().get_notify_cluster_used_space_percent()
                if float(used_size) > float(notify_cluster_space_percent):
                    check_state = self.__context.state.get(self.get_plugin_name(), False)
                    #如果check_state为true，则表明之前已经记录过used容量超过阈值这条消息了，无需再次记录？
                    if check_state == False:
                        result.title = gettext("core_message_notify_title_cluster_out_space")
                        result.message = '\n'.join(gettext("core_message_notify_cluster_out_space").split("\\n")).format(int(available_size))
                        #logger.warning(result.message)
                        result.plugin_name = str(self.get_plugin_name())
                        self.__context.results.append(result)
                        self.__context.state[self.get_plugin_name()] = True
                        logger.warning("Cluster is running out of disk space")
                    return
                self.__context.state[self.get_plugin_name()] = False

        except:
            logger.exception("Error occur during get cluster state")

    def get_plugin_name(self):
        return self.__class__.__name__
```

对于node failue检查，线程执行函数为NodeDownPlugin.run()

```
class NodeDownPlugin(CheckBasePlugin):
    def __init__(self,context):
        self.__context = context

    def is_enable(self):
        return True

    def run(self):
        self.__notify_list()

    def get_plugin_name(self):
        return self.__class__.__name__

    def __get_down_node_list(self):
        down_node_list = []
        try:
            con_api = ConsulAPI()
            node_list = con_api.get_node_list()
            consul_members = con_api.get_consul_members()
            for i in node_list:
                if i.name not in consul_members:
                    i.status = NodeStatus.down
                    down_node_list.append(i.name)
            return down_node_list
        except Exception as e:
            logger.exception("error get down node list")
            return down_node_list

    def __notify_list(self):
        try:
            result = Result()
            #之前记录的down掉的node列表
            old_node_list = self.__context.state.get(self.get_plugin_name(),[])
            #当前的down掉的node列表
            down_node_list = self.__get_down_node_list()
            #逐一遍历当前down掉的node列表
            for node in down_node_list:
                #不在之前记录的down掉的node列表中才会为之记录一条消息（否则已经记录过了，无需再次记录）
                if node not in old_node_list:
                    result.message= '\n'.join(gettext("core_message_notify_down_node_list").split("\\n")).format(''.join('\n- node:{} '.format(node) for node in down_node_list))
                    #logger.warning(result.message)
                    result.plugin_name=str(self.get_plugin_name())
                    result.title = gettext("core_message_notify_down_node_list_title")
                    self.__context.results.append(result)
                    break
            self.__context.state[self.get_plugin_name()]= down_node_list
        except Exception as e:
            logger.exception("error notify down node list")
```

对于osd failuer检查，线程执行函数为OSDDownPlugin.run()

```
class OSDDownPlugin(CheckBasePlugin):
    def __init__(self, context):
        self.__context = context

    def is_enable(self):
        return True

    def run(self):
        try:
            #跟NodeDownPlugin.run()类似
            old_osd_down_list = self.__context.state.get(self.get_plugin_name(), {})
            current_osd_down_list = ceph_down_osds()
            for osd_id, node_name in current_osd_down_list.iteritems():
                if osd_id not in old_osd_down_list.keys():
                    result = Result()
                    result.plugin_name = self.get_plugin_name()
                    result.title = gettext("core_message_notify_osd_down_title")
                    result.message = '\n'.join(gettext("core_message_notify_osd_down_body").split("\\n")).format \
                        (''.join('\n- osd.{}/{} '.format(key, val) for key, val in current_osd_down_list.iteritems()))
                    #logger.warning(result.message)
                    self.__context.results.append(result)
                    break
            self.__context.state[self.get_plugin_name()] = current_osd_down_list

        except Exception as e:
            logger.exception(e)
            logger.error("An error occurred while OSDDownPlugin was running.")

    def get_plugin_name(self):
        return self.__class__.__name__
```

将检查结果通过email发送给用户，执行EmailNotificationPlugin.notify():

```
class EmailNotificationPlugin(NotifyBasePlugin):
    def notify(self):
        try:
            messages = self.__context.state.get(self.get_plugin_name(), None)
            if not messages:
                messages = []
            config_api = ConfigAPI()
            smtp_config = config_api.read_app_config()
            if smtp_config.email_notify_smtp_server == "" and smtp_config.email_notify_smtp_email == "":
                # logger.warning("SMTP configuration not set.")
                return

            followers = self._get_followers()
            for result in self.__context.results:
                if hasattr(result, "is_email_process") and result.is_email_process:
                    continue
                for user in followers:
                    fromaddr = smtp_config.email_notify_smtp_email
                    toaddr = user.email
                    # msg = MIMEMultipart()
                    msg = email_utils.create_msg(fromaddr, toaddr, result.title, result.message)

                    msg.smtp_server = smtp_config.email_notify_smtp_server
                    msg.smtp_server_port = smtp_config.email_notify_smtp_port
                    msg.email_password = smtp_config.email_notify_smtp_password
                    msg.retry_counter = 0
                    msg.full_msg = result.title + "\n" + result.message
                    msg.security = smtp_config.email_notify_smtp_security
                    messages.append(msg)
                    result.is_email_process = True
                    result.count_of_notify_plugins += 1

            unsent_messages = []
            for msg in messages:
                # Note: any error log and continua
                # Note: if pass remove message from messages

                status = email_utils.send_email(msg.smtp_server, msg.smtp_server_port, msg, msg.email_password,
                                                msg.security)
                if not status.success:
                    msg.retry_counter += 1
                    if msg.retry_counter < 1440:
                        unsent_messages.append(msg)
                    if msg.retry_counter in (1, 120, 480, 960, 1440):
                        logger.error("PetaSAN tried to send this email {} times, Can't send this message: {}.".format(
                            msg.retry_counter, msg.full_msg))
                        logger.exception(status.exception)

            self.__context.state[self.get_plugin_name()] = unsent_messages
        except Exception as ex:
            logger.exception(ex)
```


# notification使用
petasan在cluster storage approaching full capacity，或者node failure，或者osd failure的情况下会给用户发送email。当检测到cluster storage approaching full capacity，或者node failure，或者osd failure的情况下，则会借助self.__do_check()于将这些情况相关的消息记录在NotifyContext（见/usr/lib/python2.7/dist-packages/PetaSAN/core/notification/base.py）中，并借助于self.__do_notify()获取保存在NotifyContext中的消息发送email给用户。
