# 提纲
[toc]

## [Yarn 调度器Scheduler详解](https://www.cnblogs.com/gxc2015/p/5267957.html)


## [YARN源码解析(6)-CapacityScheduler](https://www.jianshu.com/p/7fd0159bd095)
### 介绍
- CapacityScheduler中也是以Hierarchical Queue的形式来组织不同的queues。

### Queue的属性
#### capacity
- Queue capacity in percentage (%) as a float (e.g. 12.5).
- The sum of capacities for all queues, at each level, must be equal to 100. 
- Applications in the queue may consume more resources than the queue’s capacity if there are free resources, providing elasticity.

#### maximum-capacity
- Maximum queue capacity in percentage (%) as a float.
- This limits the elasticity for applications in the queue. 
- Defaults to -1 which disables it.

#### minimum-user-limit-percent
- Each queue enforces a limit on the percentage of resources allocated to a user at any given time, if there is demand for resources. 
- The user limit can vary between a minimum and maximum value. The minimum value is set to this property value and the maximum value depends on the number of users who have submitted applications.
    - For e.g., suppose the value of this property is 25. If two users have submitted applications to a queue, no single user can use more than 50% of the queue resources. If a third user submits an application, no single user can use more than 33% of the queue resources. With 4 or more users, no user can use more than 25% of the queues resources. 
- A value of 100 implies no user limits are imposed. The default is 100. 
- Value is specified as a integer.

#### user-limit-factor
- The multiple(It's not necessarily an integer, and decimal is OK, eg., 0.3 or 1.4) of the queue capacity which can be configured to allow a single user to acquire more resources. 
- By default this is set to 1 which ensures that a single user can never take more than the queue’s configured capacity irrespective of how idle the cluster is.
- Value is specified as a float.

#### maximum-allocation-mb
- The per queue maximum limit of memory to allocate to each container request at the Resource Manager.

#### maximum-allocation-vcores
- The per queue maximum limit of virtual cores to allocate to each container request at the Resource Manager. 

#### user-settings.<user-name>.weight
- This floating point value is used when calculating the user limit resource values for users in a queue. This value will weight each user more or less than the other users in the queue.
- For example, if user A should receive 50% more resources in a queue than users B and C, this property will be set to 1.5 for user A. Users B and C will default to 1.0.

#### aximum-applications
- Maximum number of applications in the system which can be concurrently active both running and pending. 
- This is a hard limit and any applications submitted when this limit is reached will be rejected.
- Default is 10000. 

#### maximum-am-resource-percent
- Maximum percent of resources in the cluster which can be used to run application masters - controls number of concurrent active applications.



## [Untangling Apache Hadoop YARN, Part 4: Fair Scheduler Queue Basics](https://blog.cloudera.com/blog/2016/06/untangling-apache-hadoop-yarn-part-4-fair-scheduler-queue-basics/)
### Introducing Queues
- Queues are the organizing structure for YARN schedulers, allowing multiple tenants to share the cluster. 
- The root queue is the parent of all queues. All other queues are each a child of the root queue or another queue (also called hierarchical queues).
- Hierarchical Queues example
    - ![image](http://blog.cloudera.com/wp-content/uploads/2016/01/untangling-yarn-3-f2.png)
        - The marketing queue has a weight of 3.0, the sales queue has a weight of 4.0, and the datascience queue has a weight of 13.0. So, the allocation from the root will be 15% to marketing, 20% to sales, and 65% to datascience.
        - In the marketing queue, there are two child queues of nonequal weight: reports and website. This means that jobs in the reports queue are allocated twice as many resources as jobs in the website queue. However, together, their weight is still governed by the marketing queue’s weight.

### Queue Priorities
#### Share Weights
```
<weight>Numerical Value 0.0 or greater</weight>
```
- The weight determines the amount of resources a queue deserves in relation to its siblings.
- This limit will be satisfied quickly if there is an equivalent amount of free resources on the cluster. Otherwise, the resources will be made available as tasks in other queues finish.
- This is the **recommended** method of performing queue resource allocation.

#### Resource Constraints
There are two types of resource constraints:
```
<minResources>20000 mb, 10 vcores</minResources>
<maxResources>2000000 mb, 1000 vcores</maxResources>
```
- The minResources limit is a soft limit. It is enforced as long as the queue total resource requirement is greater than or equal to the minResources requirement and it will get at least the amount specified as long as the following holds true:
    - The resources are available or can be preempted from other queues.
    - The sum of all minResources is not larger than the cluster’s total resources.
- The maxResources limit is a hard limit, meaning it is constantly enforced. The total resources used by the queue with this property plus any of its child and descendant queues must obey the property.
- Using minResources/maxResources is **not recommended**. There are several disadvantages:
    - Values are static values, so they need updating if the cluster size changes.
    - maxResources limits the utilization of the cluster, as a queue can’t take up any free resources on the cluster beyond the configured maxResources.

#### Limiting Applications
There are two ways to limit applications on the queue:
```
<maxRunningApps>10</maxRunningApps>
<maxAMShare>0.3</maxAMShare>
```
- The maxRunningApps limit is a hard limit. The total resources used by the queue plus any child and descendant queues must obey the property.
- The maxAMShare limit is a hard limit. This fraction represents the percentage of the queue’s resources that is allowed to be allocated for ApplicationMasters.
    - The maxAMShare is handy to set on very small clusters running a large number of applications.
    - The default value is 0.5. This default ensures that half the resources are available to run non-AM containers.

#### Users and Administrators
These properties are used to limit who can submit to the queue and who can administer applications (i.e. kill) on the queue.
```
<aclSubmitApps>user1,user2,user3,... group1,group2,...</aclSubmitApps>
<aclAdministerApps>userA,userB,userC,... groupA,groupB,...</aclAdministerApps>
```
Note: If yarn.acl.enable is set to true in yarn-site.xml, then the yarn.admin.acl property will also be considered a list of valid administrators in addition to the aclAdministerApps queue property.

#### Queue Placement Policy
Once you configure the queues, queue placement policies serve the following purpose:
- As applications are submitted to the cluster, assign each application to the appropriate queue.
- Define what types of queues are allowed to be created “on the fly”.

```
<queuePlacementPolicy>
  <Rule #1>
  <Rule #2>
  <Rule #3>
  .
  .
 </queuePlacementPolicy>
```
- will match against “Rule #1” and if that fails, match against “Rule #2” and so on until a rule matches. 
- If no queue-placement policy is set, then FairScheduler will use a default rule based on the properties yarn.scheduler.fair.user-as-default-queue and yarn.scheduler.fair.allow-undeclared-pools properties in the yarn-site.xml file.

**Special: Creating queues on the fly using the “create” attribute**
All the queue-placement policy rules allow for an XML attribute called create which can be set to “true” or “false”. For example:
```
<rule name=”specified” create=”false”>
```
If the create attribute is true, then the rule is allowed to create the queue by the name determined by the particular rule if it doesn’t already exist. If the create attribute is false, the rule is not allowed to create the queue and queue placement will move on to the next rule.

[More about Queue Placement Rules](https://blog.cloudera.com/blog/2016/06/untangling-apache-hadoop-yarn-part-4-fair-scheduler-queue-basics/)

### Configuring Fair Scheduler for Preemption
To turn on preemption, set this property in yarn-site.xml:
```
<property>yarn.scheduler.fair.preemption</property>
<value>true</value>

```
Then, in your FairScheduler allocation file, preemption can be configured on a queue via fairSharePreemptionThreshold and fairSharePreemptionTimeout as shown in the example below. 
```
<allocations>
  <queue name="busy">
    <weight>1.0</weight>
  </queue>
  <queue name="sometimes_busy">
    <weight>3.0</weight>
    <fairSharePreemptionThreshold>0.50</fairSharePreemptionThreshold>
    <fairSharePreemptionTimeout>60</fairSharePreemptionTimeout>
  </queue>

 <queuePlacementPolicy>
    <rule name="specified" />
    <rule name=”reject” />
 </queuePlacementPolicy>
</allocations>
```

- The fairSharePreemptionTimeout is the number of seconds the queue is under fairSharePreemptionThreshold before it will try to preempt containers to take resources from other queues. If not set, the queue will inherit the value from its parent queue. Default value is Long.MAX_VALUE, which means that it will not preempt containers until you set a meaningful value.
- The fairSharePreemptionThreshold is the fair share preemption threshold for the queue. If the queue waits fairSharePreemptionTimeout without receiving fairSharePreemptionThreshold*fairShare resources, it is allowed to preempt containers to take resources from other queues. If not set, the queue will inherit the value from its parent queue. Default value is 0.5f.

A couple things of note:
1. The value of fairSharePreemptionThreshold should be greater than 0.0 (setting to 0.0 would be like turning off preemption) and not greater than 1.0 (since 1.0 will return the full FairShare to the queue needing resources).
2. If fairSharePreemptionTimeout is not set for a given queue or one of its ancestor queues, and the defaultFairSharePreemptionTimeout is not set, pre-emption by this queue will never occur, even if pre-emption is enabled.
3. FairShare preemption is currently recommended instead of minResources and minSharePreemptionTimeout.

## [Capacity Scheduler vs Fair Scheduler](https://www.jianshu.com/p/fa16d7d843af)
![image](https://upload-images.jianshu.io/upload_images/204749-f89646fd8b6efd9d.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)


