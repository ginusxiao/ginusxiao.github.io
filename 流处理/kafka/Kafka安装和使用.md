# 提纲
[toc]

# Kafka安装
## 安装环境说明
编号 | 节点名 | IP | 软件 | 备注
---|---|---|---|---
1 | node19216886145 | 192.168.86.145 | kafka，zookeeper | broker.id=145
2 | node19216886146 | 192.168.86.146 | kafka，zookeeper | broker.id=146
3 | node19216886147 | 192.168.86.147 | kafka，zookeeper | broker.id=147


## 安装配置zookeeper集群
### 下载zookeeper，并拷贝到node19216886{145,146,147}这3个节点上
```
[root@node19216886145 ~]# wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.10/zookeeper-3.4.10.tar.gz
[root@node19216886145 ~]# scp /root/zookeeper-3.4.10.tar.gz node19216886146:/root/
[root@node19216886145 ~]# scp /root/zookeeper-3.4.10.tar.gz node19216886147:/root/
```

### 配置zookeeper
解压zookeeper(在node19216886{145,146,147}这3个节点上分别执行)：
```
[root@node19216886145 ~]# tar -zxvf zookeeper-3.4.10.tar.gz
```

拷贝zoo_sample.cfg为zoo.cfg(在node19216886{145,146,147}这3个节点上分别执行)：
```
[root@node19216886145 ~]# cp /root/zookeeper-3.4.10/conf/zoo_sample.cfg /root/zookeeper-3.4.10/conf/zoo.cfg
```


创建/root/zookeeper-3.4.10/data目录，供zookeeper使用(在node19216886{145,146,147}这3个节点上分别执行)：
```
[root@node19216886145 ~]# mkdir /root/zookeeper-3.4.10/data
[root@node19216886145 ~]# mkdir /root/zookeeper-3.4.10/log
```

编辑zoo.cfg(在node19216886{145,146,147}这3个节点上分别执行)：
```
[root@node19216886145 ~]# vi /root/zookeeper-3.4.10/conf/zoo.cfg
#服务器之间或客户端与服务器之间维持心跳的时间间隔，每隔tickTime时间就会发送一个心跳
ticketTime=2000

#配置 Zookeeper 接受客户端（此客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数，当已超过initLimit个tickTime长度后 Zookeeper 服务器还没有收到客户端的返回信息，则表明客户端连接失败
initLimit=10 

#配置 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度
syncLimit=5

#Zookeeper服务器监听的端口，以接受客户端的访问请求
clientPort=2181

#保存快照数据的目录，默认情况下，Zookeeper 将事务日志也保存在这个目录里
dataDir=/root/zookeeper-3.4.10/data

#保存事务日志的目录，若没提供的话则用dataDir
dataLogDir=/root/zookeeper-3.4.10/log

#server.A=B：C：D：其中 A 是一个数字，表示这个是第几号服务器；B 是这个服务器的 ip 地址；C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，此端口就是用来执行选举时服务器相互通信的端口，如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号
server.1=192.168.86.145:2888:3888
server.2=192.168.86.146:2888:3888
server.3=192.168.86.147:2888:3888
```

配置zookeeper环境变量(在node19216886{145,146,147}这3个节点上分别执行):
```
[root@node19216886145 ~]# vim /etc/profile
#添加如下内容
export ZOOKEEPER_HOME=/root/zookeeper-3.4.10
export PATH=$PATH:$ZOOKEEPER_HOME/bin

#保存修改
[root@node19216886145 ~]# source /etc/profile
```

添加myid文件，该文件在上述dataDir指定的目录下，用于指定当前节点到底是哪个server：
```
[root@node19216886145 ~]# echo "1" >> /root/zookeeper-3.4.10/data/myid
[root@node19216886146 ~]# echo "2" >> /root/zookeeper-3.4.10/data/myid
[root@node19216886147 ~]# echo "3" >> /root/zookeeper-3.4.10/data/myid
```

### 启动zookeeper

在ZooKeeper集群的每个结点上，执行启动ZooKeeper服务的脚本:
```
[root@node19216886145 ~]# zkServer.sh start
[root@node19216886146 ~]# zkServer.sh start
[root@node19216886147 ~]# zkServer.sh start
```

### 验证zookeeper各个节点状态
```
[root@node19216886145 ~]# zkServer.sh status
[root@node19216886146 ~]# zkServer.sh status
[root@node19216886147 ~]# zkServer.sh status
```

## 安装配置kafka集群 
### 下载kafka，并拷贝到node19216886{145,146,147}这3个节点上
```
[root@node19216886145 ~]# wget https://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.0.0/kafka_2.11-2.0.0.tgz
[root@node19216886145 ~]# scp /root/kafka_2.11-2.0.0.tgz node19216886146:/root/
[root@node19216886145 ~]# scp /root/kafka_2.11-2.0.0.tgz node19216886147:/root/
```

### 配置kafka
解压kafka(在node19216886{145,146,147}这3个节点上分别执行)：
```
[root@node19216886145 ~]# tar -zxvf kafka_2.11-2.0.0.tgz
```

配置kafka环境变量(在node19216886{145,146,147}这3个节点上分别执行):
```
[root@node19216886145 ~]# vim /etc/profile
#添加如下内容
export KAFKA_HOME=/root/kafka_2.11-2.0.0
export PATH=$PATH:$ZOOKEEPER_HOME/bin

#保存修改
[root@node19216886145 ~]# source /etc/profile
```

创建日志存放目录，需要分别为每个节点指定日志存储目录，如果所有节点都采用本地文件系统作为日志存储，则只需在每个节点上分别创建一个目录即可，**如果采用共享分布式存储作为日志从年初，则需要确保每个运行kafka broker的节点上采用不同的目录**：
```
#如果规划日志存储在本地，则可以设置类似于如下的目录
#为节点node19216886145准备日志存储
[root@node19216886145 ~]# mkdir -p /root/kafka_2.11-2.0.0/log
#为节点node19216886146准备日志存储
[root@node19216886146 ~]# mkdir -p /root/kafka_2.11-2.0.0/log
#为节点node19216886147准备日志存储
[root@node19216886147 ~]# mkdir -p /root/kafka_2.11-2.0.0/log

#如果规划日志存储在共享分布式存储tianfs中，则可以设置类似于如下的目录（其中/mnt/tian为tian的挂载目录，必须确保各节点使用共享存储中的不同目录）
#为节点node19216886145准备日志存储
[root@node19216886145 ~]# mkdir -p /mnt/tian/kafka/log1 
#为节点node19216886146准备日志存储
[root@node19216886145 ~]# mkdir -p /mnt/tian/kafka/log2
#为节点node19216886147准备日志存储
[root@node19216886145 ~]# mkdir -p /mnt/tian/kafka/log3
```

修改配置文件server.properties文件（当前示例中在/root/kafka_2.11-2.0.0/config/server.properties文件中），配置kafka(在node19216886{145,146,147}这3个节点上分别执行，但要确保在不同节点上的broker.id不同，如分别采用145,146和147):

节点node19216886145
```
[root@node19216886145 ~]# vi /root/kafka_2.11-2.0.0/config/server.properties
broker.id=145
delete.topic.enable=true
# 如果日志存储在本地，则设置如下（如果要设置多个日志存放目录，则用逗号分隔）
log.dirs=/root/kafka_2.11-2.0.0/log
# 如果日志存储在共享分布式存储tianfs中，则设置如下（如果要设置多个日志存放目录，则用逗号分隔）
#log.dirs=/mnt/tian/kafka/log1
zookeeper.connect=192.168.86.145:2181,192.168.86.146:2181,192.168.86.147:2181
```

节点node19216886145
```
[root@node19216886146 ~]# vi /root/kafka_2.11-2.0.0/config/server.properties
broker.id=146
delete.topic.enable=true
# 如果日志存储在本地，则设置如下（如果要设置多个日志存放目录，则用逗号分隔）
log.dirs=/root/kafka_2.11-2.0.0/log
# 如果日志存储在共享分布式存储tianfs中，则设置如下（如果要设置多个日志存放目录，则用逗号分隔）
#log.dirs=/mnt/tian/kafka/log6
zookeeper.connect=192.168.86.145:2181,192.168.86.146:2181,192.168.86.147:2181
```

节点node19216886145
```
[root@node19216886145 ~]# vi /root/kafka_2.11-2.0.0/config/server.properties
broker.id=147
delete.topic.enable=true
# 如果日志存储在本地，则设置如下（如果要设置多个日志存放目录，则用逗号分隔）
log.dirs=/root/kafka_2.11-2.0.0/log
# 如果日志存储在共享分布式存储tianfs中，则设置如下（如果要设置多个日志存放目录，则用逗号分隔）
#log.dirs=/mnt/tian/kafka/log3
zookeeper.connect=192.168.86.145:2181,192.168.86.146:2181,192.168.86.147:2181
```

### 启动zookeeper集群（如果已经启动，则略过此步骤）
```
[root@node19216886145 ~]# zkServer.sh start
[root@node19216886146 ~]# zkServer.sh start
[root@node19216886147 ~]# zkServer.sh start

[root@node19216886145 ~]# zkServer.sh status
[root@node19216886146 ~]# zkServer.sh status
[root@node19216886147 ~]# zkServer.sh status
```

### 启动kafka集群
```
[root@node19216886145 ~]# kafka-server-start.sh -daemon /root/kafka_2.11-2.0.0/config/server.properties
[root@node19216886146 ~]# kafka-server-start.sh -daemon /root/kafka_2.11-2.0.0/config/server.properties
[root@node19216886147 ~]# kafka-server-start.sh -daemon /root/kafka_2.11-2.0.0/config/server.properties
```

# Kafka使用
## 使用示例
在集群中创名为test的topic，并设置partitions为2，replication-factor为3，命令如下：
```
[root@node19216886145 ~]# kafka-topics.sh --create --zookeeper 192.168.86.145:2181,192.168.86.146:2181,192.168.86.147:2181 --topic test --partitions 2 --replication-factor 3
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
Created topic "test".
```

使用describe命令查看创建的topic：
```
[root@node19216886145 ~]# kafka-topics.sh --describe --zookeeper 192.168.86.145:2181,192.168.86.146:2181,192.168.86.147:2181 --topic test
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
Topic:test	PartitionCount:2	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 145	Replicas: 145,147,146	Isr: 145,147,146
	Topic: test	Partition: 1	Leader: 146	Replicas: 146,145,147	Isr: 146,145,147
```

向名为test的topic发送新的message：
```
[root@node19216886145 ~]# kafka-console-producer.sh --broker-list 192.168.86.145:9092,192.168.86.146:9092 --topic test
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
>hello, world!
>this is my first message!
>
```

查看名为test的topic中刚刚发送的的message：
```
[root@node19216886145 ~]# kafka-console-consumer.sh --bootstrap-server 192.168.86.145:9092,192.168.86.146:9092 --topic test --from-beginning
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
this is my first message!
hello, world!
```

采用kafka自带压测命令：
```
[root@node19216886145 ~]# kafka-producer-perf-test.sh --num-records 100000 --topic test --record-size 10 --throughput 1000  --producer-props bootstrap.servers=192.168.86.145:9092,192.168.86.146:9092,192.168.86.147:9092
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
1 records sent, 0.2 records/sec (0.00 MB/sec), 4920.0 ms avg latency, 4920.0 max latency.
10471 records sent, 2091.3 records/sec (0.02 MB/sec), 1508.5 ms avg latency, 5477.0 max latency.
5008 records sent, 1000.2 records/sec (0.01 MB/sec), 14.0 ms avg latency, 49.0 max latency.
4991 records sent, 995.6 records/sec (0.01 MB/sec), 11.0 ms avg latency, 40.0 max latency.
5029 records sent, 1005.2 records/sec (0.01 MB/sec), 43.2 ms avg latency, 563.0 max latency.
5008 records sent, 996.0 records/sec (0.01 MB/sec), 10.3 ms avg latency, 136.0 max latency.
5022 records sent, 1004.4 records/sec (0.01 MB/sec), 10.5 ms avg latency, 103.0 max latency.
4997 records sent, 998.6 records/sec (0.01 MB/sec), 8.7 ms avg latency, 42.0 max latency.
4962 records sent, 991.4 records/sec (0.01 MB/sec), 9.7 ms avg latency, 141.0 max latency.
5058 records sent, 1011.6 records/sec (0.01 MB/sec), 223.5 ms avg latency, 1442.0 max latency.
5003 records sent, 1000.6 records/sec (0.01 MB/sec), 4.3 ms avg latency, 39.0 max latency.
4980 records sent, 995.4 records/sec (0.01 MB/sec), 5.2 ms avg latency, 77.0 max latency.
5026 records sent, 1004.8 records/sec (0.01 MB/sec), 6.4 ms avg latency, 143.0 max latency.
5005 records sent, 1001.0 records/sec (0.01 MB/sec), 3.9 ms avg latency, 37.0 max latency.
4994 records sent, 998.4 records/sec (0.01 MB/sec), 47.6 ms avg latency, 639.0 max latency.
5009 records sent, 1001.6 records/sec (0.01 MB/sec), 4.7 ms avg latency, 34.0 max latency.
4999 records sent, 999.8 records/sec (0.01 MB/sec), 5.2 ms avg latency, 133.0 max latency.
5006 records sent, 1000.4 records/sec (0.01 MB/sec), 4.4 ms avg latency, 42.0 max latency.
5002 records sent, 1000.4 records/sec (0.01 MB/sec), 5.2 ms avg latency, 90.0 max latency.
100000 records sent, 999.490260 records/sec (0.01 MB/sec), 179.16 ms avg latency, 5477.00 ms max latency, 6 ms 50th, 1072 ms 95th, 4648 ms 99th, 4948 ms 99.9th.
```

# 参考
[Kafka Quickstart](http://kafka.apache.org/quickstart)

[CetnOS7.0安装配置Kafka集群](https://blog.csdn.net/jssg_tzw/article/details/73106299)

[kafka命令大全](http://orchome.com/454)

[Kafka常用命令使用说明](https://blog.csdn.net/lkforce/article/details/77776684)
