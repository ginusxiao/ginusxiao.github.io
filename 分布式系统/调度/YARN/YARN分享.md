# 提纲
[toc]

## Introduction
- Yarn - "Yet Another Resource Negotiator"
- Functionality
    - Resource management
    - Job scheduling
    - Data operating system, it enables Hadoop to process other purpose-built data processing system other than MapReduce.

## Architecture
![image](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2017/05/Apache-YARN-architecture-min.jpg)

### Resource Manager(RM)
- It is the master daemon of Yarn;
- Manage the global assignments of resources(CPU and Memory) among all applications;
    - It also has the ability to symmetrically request back resources from a running application. 
- Has two main components:
    - Scheduler
    - Application Manager

#### Scheduler
- The scheduler is responsible for allocating resources to the running application;
- The scheduler determines how much and where to allocate based on resource availability and the configured sharing policy;
- The Scheduler has a pluggable policy plug-in, which is responsible for partitioning the cluster resources among the various queues, applications etc;
    -  Current Map-Reduce schedulers such as the CapacityScheduler and the FairScheduler would be some examples of the plug-in;

#### Application Manager
- It manages running Application Masters in the cluster, ie., It accepts a job from the client and negotiates for a container to execute the application specific ApplicationMaster and it provide the service for restarting the ApplicationMaster in the case of failure;

### Node Manager(NM)
- It is the slave daemon of Yarn;
- Is responsible for containers, monitoring their resource usage and reporting it to the Resource Manager;
- Manage user process on that machine;
- Tracks the health of node on which it is running;
- It also allows plugging(by specifing application-specific services, as part of the configurations and loaded by the NM during startup) long-running auxiliary services to the NM;

### Application Master(AM)
- One application master per application;
- It negotiates resource from the resource manager and works with the node manager;
- It manages the application life cycle;

## Resource Manager In Depth
![image](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2016/06/Resource-Manager.png)

### Components interfacing RM to the client
- ClientService
    - The client interface to the Resource Manager. 
    - This component handles all the RPC interfaces to the RM from the clients including operations like application submission, application termination, obtaining queue information, cluster statistics etc.
    
- AdminService
    - To make sure that admin requests don’t get starved due to the normal users’ requests and to give the operators’ commands the higher priority, all the admin operations like refreshing node-list, the queues’ configuration etc. are served via this separate interface.

### Components connecting RM to the nodes
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

### Components interacting with the per-application AMs
- ApplicationMasterService
    - Services the RPCs from all the AMs like registration of new AMs, termination/unregister-requests from any finishing AMs, obtaining container-allocation & deallocation requests from all running AMs and forward them over to the YarnScheduler. 
    - ApplicationMasterService and AMLivelinessMonitor work together to maintain the fault tolerance of Application Masters.
    
- AMLivelinessMonitor
    - Maintains the list of live AMs and dead/non-responding AMs.

### The core of the ResourceManager
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

### TokenSecretManagers (for security)
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


### Resource Manager Restart
- RM is the central authority that manages resources and schedules application, so potentially SPOF;
- There are two types of restart of Resource Manager:
    - Non-work-preserving RM restart
        - Enhance RM to persist application/attempt state in a pluggable state-store;
        - RM will reload the state from state-store on the restart and re-kick the previously running apps;
    - Work-preserving RM restart
        - Reconstruct the running state of RM by combining the container status from NM and container requests from AM on restart;

### RM HA
- Before Hadoop v2.4, the RM was the SPOF;
- From Hadoop v2.4, the HA feature adds redundancy in the form of and Active/Standby RM pair;
- The trigger to transition-to-active comes from either of :
    -  Manual transitions and failover
    -  Automatic failover


## Node Manager In Depth
![image](https://d2h0cx97tjks2p.cloudfront.net/blogs/wp-content/uploads/sites/2/2016/06/Node-Manager.jpg)

### NodeStatusUpdater
- On startup, this component registers with the ResourceManager(RM) and sends information about the resources available to every node. 
- Subsequent NM-RM communication exchange updates on container statuses of every node like containers running on the node and completed containers, etc.
- In addition, the RM may signal the NodeStatusUpdater to potentially kill already running containers.

### Container Manager
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

### Container Executor
- Interacts with the underlying operating system to securely place files and directories needed by containers and subsequently to launch and clean up processes corresponding to containers in a secure manner.
- Distinguish from ContainerLauncher
    - ContainerLauncher is responsible for  preparing for container launching(preparation/initialization);
    - Container Executor is responsible for launching the container(execution);

### NodeHealthChecker Service
- Check the health of the node by running a configured script regularly 
- It also monitors the health of the disks specifically by creating temporary files on the disks every so often.
- Any changes in the health of the system are notified to NodeStatusUpdater which in turn passes on the information to the RM.

### Security
- ContainerTokenSecretManager
    - Examines incoming requests for containers to ensure that all the incoming requests are indeed properly authorized by the ResourceManager.

### WebServer
- Exposes the list of applications, container information running on the node at a given point of time, node-health related information and the logs produced by the containers.

## Application Workflow in YARN

![image](https://d1jnx9ba8s6j9r.cloudfront.net/blog/wp-content/uploads/2018/06/Application-Workflow-Hadoop-YARN-Edureka.png)

- Client submits an application
- Resource Manager allocates a container to start Application Manager
- Application Manager registers with Resource Manager
- Application Manager asks containers from Resource Manager
- Application Manager notifies Node Manager to launch containers
- Application code is executed in the container
- Client contacts Resource Manager/Application Manager to monitor application’s status
- Application Manager unregisters with Resource Manager

## Application Example
见192.168.80.30或者[这里](https://github.com/hortonworks/simple-yarn-app/tree/master/src/main/java/com/hortonworks/simpleyarnapp).