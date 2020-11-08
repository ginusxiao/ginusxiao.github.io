# 提纲
[toc]

# 参考
## [Yarn进化历程](http://hadoop.apache.org/docs/)
这里汇总了Hadoop版本的变迁，从版本变迁中（阅读各版本中关于Yarn的部分）我们可以找到Hadoop Yarn的进化历程。

### [How Apache Hadoop 3 Adds Value Over Apache Hadoop 2](https://hortonworks.com/blog/hadoop-3-adds-value-hadoop-2/)

**Agility & Time to Market**
Although Hadoop 2 uses containers, Hadoop 3 containerization brings agility and package isolation story of Docker.  A container-based service makes it possible to build apps quickly and roll one out in minutes. It also brings faster time to market for services.

**Total Cost of Ownership**
Hadoop 2 has a lot more storage overhead than Hadoop 3. For example, in Hadoop 2, if there are 6 blocks and 3x replication of each block, the result will be 18 blocks of space.

With erasure coding in Hadoop 3, if there are 6 blocks, it will occupy a 9 block space – 6 blocks and 3 for parity – resulting in less storage overhead.  The end result -instead of the 3x hit on storage, the erasure coding storage method will incur an overhead of 1.5x, while maintaining the same level of data recoverability. It halves the storage cost of HDFS while also retaining data durability.  Storage overhead can be reduced from 200% to 50%. In addition, you benefit from the tremendous cost savings.

**Scalability & Availability**
Hadoop 2 and Hadoop 1 only use a single NameNode to manage all Namespaces. Hadoop 3 has multiple Namenodes for multiple namespaces for NameNode Federation which improves scalability.

In Hadoop 2, there is only one standby NameNode.  Hadoop 3 supports multiple standby NameNodes. If one standby node goes down over the weekend, you have the benefit of other standby NameNodes so the cluster can continue to operate.  This feature gives you a longer servicing window.

Hadoop 2 uses an old timeline service which has scalability issues.  Hadoop 3 improves the timeline service v2 and improves the scalability and reliability of timeline service.

**New Use Cases**
Hadoop 2 doesn’t support GPUs. Hadoop 3 enables scheduling of additional resources, such as disks and GPUs for better integration with containers, **deep learning & machine learning**.  This feature provides the basis for supporting GPUs in Hadoop clusters, which enhances the performance of computations required for Data Science and AI use cases.

Hadoop 2 cannot accommodate intra-node disk balancing. Hadoop 3 has **intra-node disk balancing**. If you are repurposing or adding new storage to an existing server with older capacity drives, this leads to unevenly disks space in each server.   With intra-node disk balancing, the space in each disk is evenly distributed.

Hadoop 2 has only inter-queue preemption across queues. Hadoop 3 introduces **intra-queue preemption** which goes to the next level time by allowing preemption between application within a single queue. This means that you can prioritize jobs within the queue based on user limits and/or application priority

In conclusion, we are very excited about the upcoming releases on Hadoop 3.  The accelerated release schedule plans anticipated for this year will bring even more capabilities into the hands of the users as soon as possible.  If you look at the blog published last year called Data Lake 3.0: The Ez Button To Deploy In Minutes And Cut TCO By Half, we will see many of the Data Lake 3.0 architecture and innovations from the Apache Hadoop community come to life in our next release of the Hortonworks Data Platform.

## [Hadoop Yarn Tutorial for Beginners](https://data-flair.training/blogs/hadoop-yarn-tutorial/)
### Introduction
- Yarn - "Yet Another Resource Negotiator"
- Functionality
    - Resource management
    - Job scheduling
    - Data operating system, it enables Hadoop to process other purpose-built data processing system other than MapReduce.

### Architecture
![image](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2017/05/Apache-YARN-architecture-min.jpg)

#### Resource Manager(RM)
- It is the master daemon of Yarn;
- Manage the global assignments of resources(CPU and Memory) among all applications;
    - It also has the ability to symmetrically request back resources from a running application. 
- Has two main components:
    - Scheduler
    - Application Manager

##### Scheduler
- The scheduler is responsible for allocating resources to the running application;
- The scheduler determines how much and where to allocate based on resource availability and the configured sharing policy;
- The Scheduler has a pluggable policy plug-in, which is responsible for partitioning the cluster resources among the various queues, applications etc;
    -  Current Map-Reduce schedulers such as the CapacityScheduler and the FairScheduler would be some examples of the plug-in;

##### Application Manager
- It manages running Application Masters in the cluster, ie., It accepts a job from the client and negotiates for a container to execute the application specific ApplicationMaster and it provide the service for restarting the ApplicationMaster in the case of failure;

#### Node Manager(NM)
- It is the slave daemon of Yarn;
- Is responsible for containers, monitoring their resource usage and reporting it to the Resource Manager;
- Manage user process on that machine;
- Tracks the health of node on which it is running;
- It also allows plugging(by specifing application-specific services, as part of the configurations and loaded by the NM during startup) long-running auxiliary services to the NM;

#### Application Master(AM)
- One application master per application;
- It negotiates resource from the resource manager and works with the node manager;
- It manages the application life cycle;

### Resource Manager In Depth
![image](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2016/06/Resource-Manager.png)

#### Components interfacing RM to the client
- ClientService
    - The client interface to the Resource Manager. 
    - This component handles all the RPC interfaces to the RM from the clients including operations like application submission, application termination, obtaining queue information, cluster statistics etc.
    
- AdminService
    - To make sure that admin requests don’t get starved due to the normal users’ requests and to give the operators’ commands the higher priority, all the admin operations like refreshing node-list, the queues’ configuration etc. are served via this separate interface.

#### Components connecting RM to the nodes
- ResourceTrackerService
    - Obtains heartbeats from nodes in the cluster and forwards them to YarnScheduler. 
    - Responds to RPCs from all the nodes, registers new nodes, rejecting requests from any invalid/decommissioned nodes.

- NMLivelinessMonitor
    - To keep track of live nodes and dead nodes.
        - Keeps track of each node’s its last heartbeat time. 
        - Any node that doesn’t send a heartbeat within a configured interval of time, by default 10 minutes, is deemed dead and is expired by the RM. 
        - All the containers currently running on an expired node are marked as dead and no new containers are scheduling on such node.

- NodesListManager
    - Manages valid and excluded nodes.

#### Components interacting with the per-application AMs
- ApplicationMasterService
    - Services the RPCs from all the AMs like registration of new AMs, termination/unregister-requests from any finishing AMs, obtaining container-allocation & deallocation requests from all running AMs and forward them over to the YarnScheduler. 
    - ApplicationMasterService and AMLivelinessMonitor work together to maintain the fault tolerance of Application Masters.
    
- AMLivelinessMonitor
    - Maintains the list of live AMs and dead/non-responding AMs.

#### The core of the ResourceManager
- ApplicationsManager
    - Responsible for maintaining a collection of submitted applications. 
        - After application submission, it first validates the application's specifications and rejects any application that requests unsatisfiable resources for its ApplicationMaster.
        - It then ensuers that not other application was already submitted with the same application ID.
        - Finally, it forwards the admitted application to the scheduler.
    - Also, keeps a cache of completed applications so as to serve users' requests via web UI or command line long after the applications in question finished.

- ApplicationACLsManager
    - Maintains the ACLs lists per application and enforces them whenever a request like killing an application, viewing an application status is received.

- ApplicationMasterLauncher
    - Maintains a thread-pool to launch AMs of newly submitted applications as well as applications whose previous AM attempts exited due to some reason. 
    - Also responsible for cleaning up the AM when an application has finished normally or forcefully terminated.

- YarnScheduler
    - It is responsible for allocating resources to the various running applications subject to constraints of capacities, queues etc. 
    - It also performs its scheduling function based on the resource requirements of the applications.

- ContainerAllocationExpirer
    - This component is in charge of ensuring that all allocated containers are used by AMs and subsequently launched on the correspond NMs. 
    - It maintains the list of allocated containers that are still not used on the corresponding NMs. For any container, if the corresponding NM doesn’t report to the RM that the container has started running within a configured interval of time, by default 10 minutes, then the container is deemed as dead and is expired by the RM.

#### TokenSecretManagers (for security)
- ApplicationTokenSecretManager
    - RM uses the per-application tokens called ApplicationTokens to avoid arbitrary processes from sending RM scheduling requests. 

- ContainerTokenSecretManager
    - RM issues special tokens called Container Tokens to ApplicationMaster(AM) for a container on the specific node. 
    - These tokens are used by AM to create a connection with NodeManager having the container in which job runs.
    - A container token consists of the following fields
        - Container ID
        - NodeManager address
        - Application submitter
        - Resource: informs the NodeManager about the amount of each resource that the ResourceManager has authorized an ApplicationMaster to start.
        - Expiry timestamp: NodeManager look at this timestamp to determine if the container token passed is still valid.
        - Master key identifier
        - ResourceManager identifier

- RMDelegationTokenSecretManager
     - A ResourceManager specific delegation-token secret-manager. 
     - It is responsible for generating delegation tokens to clients which can also be passed on to unauthenticated processes that wish to be able to talk to RM.


#### Resource Manager Restart
- RM is the central authority that manages resources and schedules application, so potentially SPOF;
- There are two types of restart of Resource Manager:
    - Non-work-preserving RM restart
        - Enhance RM to persist application/attempt state in a pluggable state-store;
        - RM will reload the state from state-store on the restart and re-kick the previously running apps;
    - Work-preserving RM restart
        - Reconstruct the running state of RM by combining the container status from NM and container requests from AM on restart;

#### RM HA
- Before Hadoop v2.4, the RM was the SPOF;
- From Hadoop v2.4, the HA feature adds redundancy in the form of and Active/Standby RM pair;
- The trigger to transition-to-active comes from either of :
    -  Manual transitions and failover
    -  Automatic failover


### Node Manager In Depth
![image](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2016/06/Node-Manager.jpg)

#### NodeStatusUpdater
- On startup, this component registers with the ResourceManager(RM) and sends information about the resources available to every node. 
- Subsequent NM-RM communication exchange updates on container statuses of every node like containers running on the node and completed containers, etc.
- In addition, the RM may signal the NodeStatusUpdater to potentially kill already running containers.

#### Container Manager
- RPC server
    - It accepts requests from Application Masters (AMs) to start new containers, or to stop running ones. 
    - It works in association with ContainerTokenSecretManager to authorize all requests. 

- ResourceLocalizationService
    - Responsible for securely downloading and organizing various file resources needed by containers.

- ContainersLauncher
    - Maintains a pool of threads to prepare and launch containers as quickly as possible. 
    - Also, cleans up the containers’ processes when such a request is sent by the RM or the ApplicationMasters (AMs).
    
- AuxServices
    - The NM provides a framework for extending its functionality by configuring auxiliary services.
    - These services have to be configured before NM starts. Auxiliary services are notified when an application’s first container starts on the node, and when the application is considered to be complete.

- ContainersMonitor
    - Monitors each container’s resource usage continuously and if a container exceeds its allocation, it signals the container to be killed. 
    
- LogHandler
    - A pluggable component with the option of either keeping the containers’ logs on the local disks or zipping them together and uploading them onto a file-system.

#### Container Executor
- Interacts with the underlying operating system to securely place files and directories needed by containers and subsequently to launch and clean up processes corresponding to containers in a secure manner.
- Distinguish from ContainerLauncher
    - ContainerLauncher is responsible for  preparing for container launching(preparation/initialization);
    - Container Executor is responsible for launching the container(execution);

#### NodeHealthChecker Service
- Check the health of the node by running a configured script regularly 
- It also monitors the health of the disks specifically by creating temporary files on the disks every so often.
- Any changes in the health of the system are notified to NodeStatusUpdater which in turn passes on the information to the RM.

#### Security
- ContainerTokenSecretManager
    - Examines incoming requests for containers to ensure that all the incoming requests are indeed properly authorized by the ResourceManager.

#### WebServer
- Exposes the list of applications, container information running on the node at a given point of time, node-health related information and the logs produced by the containers.


### Yarn Docker Container Executor
- It allows the Yarn NM to launch yarn container to Docker container, thus the custom software environment is isolated from NM;

### Yarn Timeline Server
- To store and retrive application's current and historic infomation in a generic fashion;

### Application Workflow in YARN

![image](https://d1jnx9ba8s6j9r.cloudfront.net/blog/wp-content/uploads/2018/06/Application-Workflow-Hadoop-YARN-Edureka.png)

- Client submits an application
- Resource Manager allocates a container to start Application Manager
- Application Manager registers with Resource Manager
- Application Manager asks containers from Resource Manager
- Application Manager notifies Node Manager to launch containers
- Application code is executed in the container
- Client contacts Resource Manager/Application Manager to monitor application’s status
- Application Manager unregisters with Resource Manager

## [Introduction to YARN](https://developer.ibm.com/tutorials/bd-yarn-intro/)
### Hadoop YARN Architecture
![image](https://wdc.objectstorage.softlayer.net/v1/AUTH_7046a6f4-79b7-4c6c-bdb7-6f68e920f6e5/Code-Tutorials/bd-yarn-intro/images/Figure3Architecture-of-YARN.png)

### Application submission in YARN
![image](https://wdc.objectstorage.softlayer.net/v1/AUTH_7046a6f4-79b7-4c6c-bdb7-6f68e920f6e5/Code-Tutorials/bd-yarn-intro/images/fig04.png)

- ResourceManager accepts a new application submission.
- The Scheduler selects a container in which the ApplicationMaster will run, and then launch the ApplicationMaster.
- After the ApplicationMaster is started, it will be responsible for a whole life cycle of this application.
- ApplicationMaster sends resource requests to the ResourceManager to ask for containers needed to run the application’s tasks.
- If and when it is possible, the ResourceManager grants a container that satisfies the requirements requested by the ApplicationMaster in the resource request.
- After a container is granted, the ApplicationMaster will ask the NodeManager to use these resources(granted to container) to launch an application-specific task. 
    - This task can be any process written in any framework (such as a MapReduce task or a Giraph task).
- The ApplicationMaster spends its whole life negotiating containers to launch all of the tasks needed to complete its application. It also monitors the progress of an application and its tasks, restarts failed tasks in newly requested containers, and reports progress back to the client that submitted the application. 
- After the application is complete, the ApplicationMaster shuts itself down and releases its own container.

## [Apache Hadoop Yarn Concepts and Applications](https://hortonworks.com/blog/apache-hadoop-yarn-concepts-and-applications/)
### ResourceRequest and Container
- ResourceRequest has the following form:
    - <resource-name, priority, resource-requirement, number-of-containers>
    - resource-name is either hostname, rackname or * to indicate no preference. In future, we expect to support even more complex topologies for virtual machines on a host, more complex networks etc.
    - priority is intra-application priority for this request (to stress, this isn’t across multiple applications).
    - resource-requirement is required capabilities such as memory, cpu etc.(at the time of writing YARN only supports memory and cpu).
    - number-of-containers is just a multiple of such containers.

- Container is a logical bundle of resources(eg., memory, cores) bound to a particular cluster node.

### Application Execution Sequence
![image](https://2xbbhjxc6wk3v21p62t8n4d4-wpengine.netdna-ssl.com/wp-content/uploads/2012/08/yarnflow-600x487.png)

Application execution consists of the following steps:
- Application submission.
- Bootstrapping the ApplicationMaster instance for the application.
- Application execution managed by the ApplicationMaster instance.

Application execution sequence illustrated in the diagram:
- A client program submits the application, including the necessary specifications to launch the application-specific ApplicationMaster itself.
- The ResourceManager assumes the responsibility to negotiate a specified container in which to start the ApplicationMaster and then launches the ApplicationMaster.
- The ApplicationMaster, on boot-up, registers with the ResourceManager – the registration allows the client program to query the ResourceManager for details, which allow it to  directly communicate with its own ApplicationMaster.
- During normal operation the ApplicationMaster negotiates appropriate resource containers via the resource-request protocol.
- On successful container allocations, the ApplicationMaster launches the container by providing the container launch specification to the NodeManager. The launch specification, typically, includes the necessary information to allow the container to communicate with the ApplicationMaster itself.
- The application code executing within the container then provides necessary information (progress, status etc.) to its ApplicationMaster via an application-specific protocol.
- During the application execution, the client that submitted the program communicates directly with the ApplicationMaster to get status, progress updates etc. via an application-specific protocol.
- Once the application is complete, and all necessary work has been finished, the ApplicationMaster deregisters with the ResourceManager and shuts down, allowing its own container to be repurposed.


## [Comparison between Apache Mesos vs Hadoop YARN](https://data-flair.training/blogs/comparison-between-apache-mesos-vs-hadoop-yarn/)

## Excellent tutorial
### [***Hadoop YARN Tutorial – Learn the Fundamentals of YARN Architecture](https://www.edureka.co/blog/hadoop-yarn-tutorial/)