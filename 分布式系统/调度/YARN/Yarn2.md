# 提纲
[toc]

## [YARN Node Labels: Label-based scheduling and resource isolation](https://developer.ibm.com/hadoop/2017/03/10/yarn-node-labels/)
With YARN Node Labels, you can mark nodes with labels such as “memory” (for nodes with more RAM) or “high_cpu” (for nodes with powerful CPUs) or any other meaningful label so that applications can choose the nodes on which to run their containers. 

The YARN ResourceManager will schedule jobs based on those node labels.

Currently, a node can have only one label assigned to it. Nodes that do not have a label belong to the “Default” partition.

### How does YARN Node Labels work?
When you submit an application, you can specify a **node label expression** to tell YARN where it should run. Containers are then allocated only on those nodes that have the specified node label. A node label expression is a phrase that contains node labels that can be specified for an application or for a single ResourceRequest. The expression can be a single label or a logical combination of labels, such as “x&&y” or “x||y”. **Currently, we only support the form of a single label.**

YARN manages resources through a hierarchy of queues. Each queue’s capacity specifies how much cluster resource it can consume. You can associate node labels with queues. Each queue can have a list of accessible node labels and the capacity for every label to which it has access. A queue can also have its own default node label expression. Applications that are submitted to this queue will use this default value if there are no specified labels of their own. If neither of the above two are specified, Default partition will be considered.

A queue’s accessible node label list determines the nodes on which applications that are submitted to this queue can run. That is, an application can specify only node labels to which the target queue has access; otherwise, it is rejected. All queues have access to the Default partition.

In the following example, Queue A has access to both partition X (nodes with label X) and partition Y (nodes with label Y). Queue B has access to only partition Y, and Queue C has access to only the Default partition (nodes with no label). Partition X is accessible only by Queue A with a capacity of 100%, whereas Partition Y is shared between Queue A and Queue B with a capacity of 50% each.
![image](https://developer.ibm.com/hadoop/wp-content/uploads/sites/28/2017/02/Node-labels.png)

When you submit an application, it is routed to the target queue according to queue mapping rules, and containers are allocated on the matching nodes if a node label has been specified. In the following example, User_1 has submitted App_1 and App_2 to Queue A with node label expression “X” and “Y”, respectively. Containers for App_1 have been allocated on Partition X, and containers for App_2 have been allocated on Partition Y.
![image](https://developer.ibm.com/hadoop/wp-content/uploads/sites/28/2017/03/submit-applications-to-queues.png)

### Exclusive and non-exclusive node labels
There are two kinds of node labels:
Resources on nodes with exclusive node labels can be allocated only to applications that request them.
Resources on nodes with non-exclusive node labels can be shared by applications that request the Default partition.
**The exclusivity attribute must be specified when you add a node label; the default is “exclusive”.** In the following example, an exclusive label “X” and a non-exclusive label “Y” are added:
```
yarn rmadmin -addToClusterNodeLabels "X,Y(exclusive=false)"
```
![image](https://developer.ibm.com/hadoop/wp-content/uploads/sites/28/2017/02/non-exclusive-node-labels.png)

When a queue is associated with one or more exclusive node labels, all applications that are submitted by the queue have exclusive access to nodes with those labels. When a queue is associated with one or more non-exclusive node labels, all applications that are submitted by the queue get first priority on nodes with those labels. If idle capacity is available on those nodes, resources are shared with applications that are requesting resources on the Default partition. In this case, with preemption enabled, the shared resources are preempted if there are applications asking for resources on non-exclusive partitions, to ensure that labeled applications have the highest priority.

### Associating node labels with queues
The following properties in the capacty-scheduler.xml file are used to associate node labels with queues for the CapacityScheduler:
- yarn.scheduler.capacity.<queue-path>.capacity defines the queue capacity for resources on nodes that belong to the Default partition.
- yarn.scheduler.capacity.<queue-path>.accessible-node-labels defines the node labels that the queue can access. “*” means that the queue can access all the node labels; ” ” (a blank space) means that the queue can access only the Default partition.
- yarn.scheduler.capacity.<queue-path>.accessible-node-labels.<label>.capacity defines the queue capacity for accessing nodes that belong to partition “label”. The default is 0.
- yarn.scheduler.capacity.<queue-path>.accessible-node-labels.<label>.maximum-capacity defines the maximum queue capacity for accessing nodes that belong to partition “label”. The default is 100.
- yarn.scheduler.capacity.<queue-path>.default-node-label-expression defines the queue’s default node label expression. If applications that are submitted to the queue don’t have their own node label expression, the queue’s default node label expression is used.

You can set these properties for the root queue or for any child queue as long as the following items are true:
- Capacity was specified for each node label to which the queue has access.
- For each node label, the sum of the capacities of the direct children of a parent queue at every level is 100%.
- Node labels that a child queue can access are the same as (or a subset of) the accessible node labels of its parent queue.

### Specifying a node label in your application
You can specify a node label in one of several ways.
- By using provided Java APIs:
    - ApplicationSubmissionContext.setNodeLabelExpression(..) to set the node label expression for all containers of the application.
    - ResourceRequest.setNodeLabelExpression(..) to set the node label expression for individual resource requests. This can overwrite the node label expression set in ApplicationSubmissionContext.
    - Specify setAMContainerResourceRequest.setNodeLabelExpression in ApplicationSubmissionContext to indicate the expected node label for the ApplicationMaster container. This can overwrite the node label expression set in ApplicationSubmissionContext.

- By specifying a node label for jobs that are submitted through the distributed shell. You can set this node label through -node_label_expression. For example:
```
yarn org.apache.hadoop.yarn.applications.distributedshell.Client
  -jar /usr/iop/<iopversion>/hadoop-yarn/hadoop-yarn-applications-distributedshell.jar
  -num_containers 10 --container_memory 1536
  -node_label_expression "X" -shell_command "sleep 60"
```

- By specifying a node label for Spark jobs. Spark enables you to set a node label expression for ApplicationMaster containers and task containers separately through --conf spark.yarn.am.nodeLabelExpression and --conf spark.yarn.executor.nodeLabelExpression. For example:
```
./bin/spark-submit --class org.apache.spark.examples.SparkPi
  --master yarn --deploy-mode cluster --driver-memory 3g --executor-memory 2g
  --conf spark.yarn.am.nodeLabelExpression=Y
  --conf spark.yarn.executor.nodeLabelExpression=Y jars/spark-examples.jar 10
```
