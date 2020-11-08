# 提纲
[toc]

## Container启动流程
### AM和RM通信
- AM通过AMRMClientAsync.registerApplicationMaster()向RM注册
- AM提交对Container的需求
    - 调用 setupContainerAskForRM()设置对Container的具体需求（优先级、资源等）
    - 调用 AMRMClientAsync.addContainerRequest()把需求提交给RM
        - AMRMClientAsync是一个抽象类，具体的实现是AMRMClientAsyncImpl
        - 最终会调用AMRMClientImpl.allocate()组装成一个 AllocateRequest，再通过RPC调用 RM 的相关方法进行申请
            - 具体来说，是通过ApplicationMasterProtocol.allocate()，发送Google protocol Buffe格式的RPC进行调用
                - 在RM端对应的处理逻辑是ApplicationMasterService.allocate()
                    - 调用YarnScheduler.allocate()方法，最后调用的是具体调度器的 allocate()方法
- AMRMClientAsync需要调用方提供一个回调类，AM实现为RMCallbackHandler
    - onContainersCompleted
        - Called when the ResourceManager responds to a heartbeat with completed containers. 
    - onContainersAllocated
        - Called when the ResourceManager responds to a heartbeat with allocated containers.
    - onContainersUpdated
        -  Called when the ResourceManager responds to a heartbeat with containers whose resource allocation has been changed.
    - onShutdownRequest
        - Called when the ResourceManager wants the ApplicationMaster to shutdown for being out of sync etc. 
    - onNodesUpdated
        - Called when nodes tracked by the ResourceManager have changed in health, availability etc. 
    - getProgress
    - onError

### AM和NM通信
- AM与NM之间通过NMClientAsync通信，AM实现为NMCallbackHandler
    - onContainerStarted
        - called when NodeManager responds to indicate its acceptance of the starting container request.
    - onContainerStatusReceived
        - called when NodeManager responds with the status of the container.
    - onContainerStopped
        - called when NodeManager responds to indicate the container is stopped.
    - onStartContainerError
        - called when an exception is raised in the process of starting a container.
    - onContainerResourceIncreased
        - called when NodeManager responds to indicate the container resource has been successfully increased.
    - onGetContainerStatusError
        - called when an exception is raised in the process of querying the status of a container.
    - onIncreaseContainerResourceError
        - called when an exception is raised in the process of increasing container resource. 
    - onStopContainerError
        - called when an exception is raised in the process of stoppin a container. 
 - AM从RM申请到container之后，请求相应的NM启动该container
    - NMClientAsync 中有一个名叫events 的事件队列，同时，NMClientAsync 还启动这一个线程，不断地从 events 中取出事件进行处理
    - AM 设置好ContainerLaunchContext, 调用NMClientAsync.startContainerAsync()启动Container
        - 生成一个 ContainerEvent(type=START_CONTAINER) 事件放入 events 队列 
        - 事件对应的处理逻辑是调用NMClient.startContainer() 同步地启动 Container
            - NMClient最终会调用 ContainerManagementProtocol.startContainers() ，以 Google Protocol Buffer 格式，通过 RPC 调用 NM 的对应方法

### NM和RM之间通信
- NM启动时会向RM注册自己，RM生成对应的RMNode结构，代表这个NM，并存放这个NM的资源信息以及其他一些统计信息
- NM和RM之间，通过心跳来维护NM信息
    - Request(NM->RM) : NM上所有Container的状态
    - Response(RM->NM) : 待删除和待清理的Container列表
    - NM这边的NodeStatusUpdater(它实际上是ResourceTrackerService的client)和RM这边的ResourceTrackerService共同负责心跳维护
    - NM这边NodeStatusUpdater周期性地调用nodeHeartbeat()向RM汇报本节点的信息
        - RM这边的处理逻辑为ResourceTrackerService.nodeHeartbeat()
            - 触发一个RMNodeStatusEvent(RMNodeEventType.STATUS_UPDATE)事件
                - RMNode接收RMNodeStatusEvent(RMNodeEventType.STATUS_UPDATE) 消息，更新自己的状态机，然后调用StatusUpdateWhenHealthyTransition.transition处理
                    - 触发NodeUpdateSchedulerEvent（SchedulerEventType.NODE_UPDATE）事件，由调度器处理
                        - 调度器处理已申请到资源但是尚未启动的(newly launched)container和已经结束的(completed)container

### NodeManager中启动Container
- NM 中负责响应来自 AM 的 RPC 请求的是 ContainerManagerImpl，该类实现了接口ContainerManagementProtocol ，接到 RPC 请求后，会调用 ContainerManagerImpl.startContainers() 
    - 首先进行 APP 的初始化（如果还没有的话），生成一个ApplicationImpl实例，然后根据请求，生成一堆 ContainerImpl 实例
    - 触发一个新事件：ApplicationContainerInitEvent ，之前生成的ApplicationImpl收到该事件，又触发一个 ContainerEvent(type=INIT_CONTAINER) 事件，这个事件由 ContainerImpl 处理
    - ContainerImpl 收到事件，更新状态机，启动辅助服(AuxServices)，然后触发一个新事件 ContainersLaucherEvent(type=LAUNCH_CONTAINER) 
        - 处理这个事件的是ContainersLauncher，它会组装出一个ContainerLaunch类并使用ExecutorService执行
            - 设置运行环境，包括生成运行脚本，Local Resource ，环境变量，工作目录，输出目录等
            - 调用 ContainerExecutor.launchContainer() 执行Container的工作
            - 执行结束后，根据执行的结果设置 Container 的状态

#### ContainerExecutor
- ContainerExecutor是抽象类，可以有不同的实现，如 DefaultContainerExecutor ，DockerContainerExecutor ，LinuxContainerExecutor 等。根据 YARN 的配置，NodeManager 启动时，会初始化具体的 ContainerExecutor
- ContainerExecutor 最主要的方法是 launchContainer() ，该方法阻塞，直到执行的命令结束
- DefaultContainerExecutor 是默认的 ContainerExecutor ，支持 Windows 和 Linux
    - 创建 Container 需要的目录
    - 拷贝 Token、运行脚本到工作目录
    - 做一些脚本的封装，然后执行脚本，返回状态码

### 启动过程图示
![image](http://dongxicheng.org/wp-content/uploads/2012/12/container_launch_proccess.jpg)

## Yarn Linux Container
在NodeManager中，有三种运行Container的方式，它们分别是:
- DefaultContainerExecutor
- LinuxContainerExecutor
- DockerContainerExecutor

### DefaultContainerExecutor
这个ContainerExecutor的实现实际上很简单，就是通过构建脚本来执行，支持Windows脚本以及Linux脚本。
在ContainerExecutor启动一个Container的过程中，涉及到了三个脚本，它们分别是:
- default_container_executor.sh
- default_container_executor_session.sh
- launch_container.sh

这3个脚本的内容如下(我们事先启动了一个application，yarn会为该application分配container，在为该application分配的每个container中都会包含这3个脚本)：

default_container_executor.sh
```
[root@dbus-n1 yarnbook]# vim /var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/default_container_executor.sh 

#!/bin/bash
/bin/bash "/var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/default_container_executor_session.sh"
rc=$?
echo $rc > "/var/data/yarn/nmPrivate/application_1546841638568_0006/container_1546841638568_0006_01_000002/container_1546841638568_0006_01_000002.pid.exitcode.tmp"
/bin/mv -f "/var/data/yarn/nmPrivate/application_1546841638568_0006/container_1546841638568_0006_01_000002/container_1546841638568_0006_01_000002.pid.exitcode.tmp" "/var/data/yarn/nmPrivate/application_1546841638568_0006/container_1546841638568_0006_01_000002/container_1546841638568_0006_01_000002.pid.exitcode"
exit $rc
```
该脚本中会启动default_container_executor_session.sh这个脚本，并将执行结果写入到相应的文件中。

default_container_executor_session.sh
```
[root@dbus-n1 yarnbook]# vim /var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/default_container_executor_session.sh 

#!/bin/bash

echo $$ > /var/data/yarn/nmPrivate/application_1546841638568_0006/container_1546841638568_0006_01_000002/container_1546841638568_0006_01_000002.pid.tmp
/bin/mv -f /var/data/yarn/nmPrivate/application_1546841638568_0006/container_1546841638568_0006_01_000002/container_1546841638568_0006_01_000002.pid.tmp /var/data/yarn/nmPrivate/application_1546841638568_0006/container_1546841638568_0006_01_000002/container_1546841638568_0006_01_000002.pid
exec setsid /bin/bash "/var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/launch_container.sh"
```
该脚本中主要是启动launch_container.sh。

launch_container.sh
```
#!/bin/bash

export LOCAL_DIRS="/var/data/yarn/usercache/root/appcache/application_1546841638568_0006"
export HADOOP_CONF_DIR="/home/rtdp/hadoop-2.6.5/etc/hadoop"
export NM_HTTP_PORT="8042"
export JAVA_HOME="/usr/lib/jvm/java-1.8.0/"
export LOG_DIRS="/home/rtdp/hadoop-2.6.5/logs/userlogs/application_1546841638568_0006/container_1546841638568_0006_01_000002"
export NM_PORT="43353"
export USER="root"
export HADOOP_YARN_HOME="/home/rtdp/hadoop-2.6.5"
export NM_HOST="dbus-n1"
export HADOOP_TOKEN_FILE_LOCATION="/var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/container_tokens"
export HADOOP_HDFS_HOME="/home/rtdp/hadoop-2.6.5"
export LOGNAME="root"
export JVM_PID="$$"
export PWD="/var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002"
export HADOOP_COMMON_HOME="/home/rtdp/hadoop-2.6.5"
export HOME="/home/"
export CONTAINER_ID="container_1546841638568_0006_01_000002"
export MALLOC_ARENA_MAX="4"
ln -sf "/var/data/yarn/usercache/root/appcache/application_1546841638568_0006/filecache/10/JBossApp.jar" "JBossApp.jar"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi

ln -sf "/var/data/yarn/usercache/root/appcache/application_1546841638568_0006/filecache/11/wildfly-14.0.0.Final.tar.gz" "jboss"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
exec /bin/bash -c "chmod -R 777 /var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/jboss/wildfly-14.0.0.Final  &&  $JAVA_HOME/bin/java -cp /home/rtdp/hadoop-2.6.5/share/hadoop/common/lib/*:/var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/JBossApp.jar org.yarnbook.JBossConfiguration --home /var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/jboss/wildfly-14.0.0.Final --server_group application_1546841638568_0006 --server container_1546841638568_0006_01_000002 --port_offset 0 --admin_user yarn --admin_password yarn --domain_controller dbus-n1 --host dbus-n1  &&  /var/data/yarn/usercache/root/appcache/application_1546841638568_0006/container_1546841638568_0006_01_000002/jboss/wildfly-14.0.0.Final/bin/domain.sh -Djboss.bind.address=dbus-n1 -Djboss.bind.address.management=dbus-n1 -Djboss.bind.address.unsecure=dbus-n1"
hadoop_shell_errorcode=$?
if [ $hadoop_shell_errorcode -ne 0 ]
then
  exit $hadoop_shell_errorcode
fi
```
该脚本就负责运行相应的Container，更确切的说，就是在NodeManager中启动应用程序。那么，自然地，DefaultContainerExecutor对于资源隔离做的不好。

### Linux Container Executor
#### Linux Container Executor配置
[Yarn Linux Container Executor配置](https://blog.csdn.net/picway/article/details/74299086)

### Docker Container Executor
[ocker Container Executor](https://hadoop.apache.org/docs/r2.7.5/hadoop-yarn/hadoop-yarn-site/DockerContainerExecutor.html)

### Container Executor的进化
请参考[这里](https://docs.hortonworks.com/HDPDocuments/HDP3/HDP-3.0.1/data-operating-system/content/run_docker_containers_on_yarn.html)。

该文中首先回顾了Yarn ContainerExecutor的背景，起初Yarn抽象出来了ContainerExecutor这个概念，并且支持DefaultContainerExecutor，LinuxContainerExecutor和WindowsSecureContainerExecutor。其中DefaultContainerExecutor 主要用于非安全的集群环境，因为这种类型的Yarn containers以和NodeManager相同的用户启动，因此不提供安全性保证。LinuxContainerExecutor 则主要用于安全的集群环境，这种类型的containers以提交作业的用户来启动和运行之。WindowsSecureContainerExecutorprovides 和LinuxContainerExecutor功能相似，但主要用于Windows平台。

后来随着Docker越来越流行，DockerContainerExecutor 也被添加到ContainerExecutors中，它允许NodeManager运行Docker命令来启动，监测和清理Docker containers。但是，DockerContainerExecutor有一些局限性 - 一些是实现上的，另一些则是架构上的。实现上的局限性比如不允许用户指定他们想使用的镜像（同一个NodeManager上的所有用户都必须使用同一个镜像）。架构上的局限性则是最大的问题，每个NodeManager上都可以配置一个ContainerExecutor，所有的任务都必须使用该ContainerExecutor，因此一旦该集群被配置为使用DockerContainerExecutor，用户将无法启动MapReduce，Tez或者Spark作业。因此，DockerContainerExecutor被丢弃，以支持一种新的抽象 - container runtimes，DockerContainerExecutor将会从Hadoop中移除。

为了解决这些缺陷，Yarn在LinuxContainerExecutor中提供container runtimes的支持，container runtimes将ContainerExecutor拆分为2部分 - 功能相关的底层框架部分和可以随着想要启动的container类型而改变的runtime部分。通过这些改变，解决了架构上的问题，可以在运行Docker containers的同时运行Yarn containers。Docker container的生命周期也有Yarn进行控制。同时这种架构上的改变，允许Yarn支持其他的容器化技术。

目前主要存在2中runtime，一种是基于进程树的runtime（process tree based runtime，也就是DefaultLinuxContainerRuntime），另一种是Docker runtime（DockerLinuxContainerRuntime）。DefaultLinuxContainerRuntime还是跟以前的Yarn一样启动Yarn container，DockerLinuxContainerRuntime则启动Docker container。

DockerLinuxContainerRuntime在YARN-3611中添加支持，对应的目标版本则是Hadoop 2.8.0.

## [First-Class Support for Long Running Services on Apache Hadoop YARN](https://hortonworks.com/blog/first-class-support-long-running-services-apache-hadoop-yarn/)

Apache Hadoop YARN is well known as the general resource-management platform for big-data applications such as MapReduce, Hive/Tez and Spark. In addition to big-data apps, another broad spectrum of workloads we see today are long running services such as HBase, Hive/LLAP and container (e.g. Docker) based services.

**YARN SERVICE FRAMEWORK COMING IN APACHE HADOOP 3.1!** This feature primarily includes the following:
- A core framework (ApplicationMaster) running on YARN serving as a container orchestrator and responsible for all service lifecycle managements.
- A RESTful API-service for users to deploy and manage their services on YARN using a simple JSON spec.
- A YARN DNS server backed by YARN service registry to enable discovering services on YARN with standard DNS lookup.
- Advanced container placement scheduling such as affinity and anti-affinity for each application, container resizing, and node labels.
- Rolling upgrades for containers and the service as a whole.

![image](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2018/01/full-flesged-YARN-cluster-illustration.jpg)
A typical workflow is:
- User posts a JSON request that describes the specification of the service such as the container memory size, number of CPU cores, Docker image ID, etc. to the YARN Service REST API. Similarly, user can submit their service creation request using the YARN CLI also.
- RM, after accepting the request, launches an ApplicationMaster (i.e. the container orchestration framework).
- The orchestration framework requests resources from RM adhering to the user’s resource requirements and then, when a container has been allocated, launches the container on NodeManager.
- NodeManager in turn launches the container process (where user code lives) or uses the Docker container runtime to launch the Docker container.
- The orchestration framework monitors containers’ health and readiness and acts upon container failures or unhealthiness. It writes the service lifecycle events and metrics into the YARN Timeline Service (backed by HBase). It also writes the additional service meta info (such as container IP and host) into the YARN service registry backed by a ZooKeeper quorum.
- The Registry DNS server listens on znode creation or deletion in ZooKeeper and creates all sorts of DNS records such as the A record and Service Record to serve DNS queries.
- Each Docker container is given a user-friendly hostname based on information provided in the JSON spec and the YARN configuration. Client can then lookup the container IP by the container hostname using standard DNS lookup.


## [Trying out Containerized Applications on Apache Hadoop YARN 3.1](https://hortonworks.com/blog/trying-containerized-applications-apache-hadoop-yarn-3-1/)

The YARN containerization feature led to two main additions to YARN: the core Docker runtime support and YARN Services.

YARN Services is a higher level abstraction that provides an easy way to define the layout of components within a service and the service’s execution parameters, such as retry policies and configuration, in a single service specification file. When the specification file is submitted to the cluster, the lower level core Docker runtime support is used to satisfy that request and start the requested Docker containers on the YARN NodeManager systems.

![image](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2018/05/NodeManager-default-container-450x346.png)

The core Docker runtime support is configured using environment variables at application submission time, which means existing Hadoop applications will not require changes to leverage containerization support.

