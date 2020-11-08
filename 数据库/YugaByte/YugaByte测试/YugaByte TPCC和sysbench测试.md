# 提纲
[toc]

## 性能测试工具简介
### TPC-C简介
TPC-C的目标是定义一系列可以在任何OLTP系统上运行的功能要求，然后由提供OLTP系统的厂商提供测试报告来证明他们满足了所有这些功能。也就是说，TPC不给出基准程序的代码，而只给出基准程序的标准规范。

在TPC-C业务模型中，模拟了一个比较复杂并具有代表意义的OLTP应用环境：
- 零件批发供应商拥有若干个分布在不同区域的商品库；
- 每个仓库负责为10个销售点供货；
    - 每个仓库中维护零件批发供应商所售卖的100000项产品；
- 每个销售点为3000个客户提供服务；
- 每个客户平均一个订单有10项产品；
- 所有订单中约1%的产品在其直接所属的仓库中没有存货，需要由其他区域的仓库来供货。

![image](http://www.eygle.com/special/pic/image01.gif)

**TPC-C模拟以下事务操作，包括：
- New-Order：录入一笔新的订货交易
    - 事务内容：对于任意一个终端，从固定的仓库(每个终端都会对应到一个特定的仓库)随机选取5-15件商品，创建新订单，其中 1%的订单要由假想的用户操作失败而回滚，1%的订单需要去其它的仓库调货
    - 主要特点：读写频繁、要求响应快
- Payment: 更新客户账户余额，并将其支付结果反映到区域和仓库的销售统计中
    - 事务内容：对于任意一个终端，从固定的仓库(每个终端都会对应到一个特定的仓库)随机选取一个销售点及其内客户，采用随机的金额支付一笔订单，并作相应历史纪录
    - 主要特点：读写频繁，要求响应快
- Delivery: 发货(模拟批处理交易)
    - 事务内容：对于任意一个终端，随机选取一个发货包，更新被处理订单的用户余额，并把该订单从新订单中删除
    - 主要特点：1-10个批量，读写频率低，较宽松的响应时间
- Order-Status: 查询客户最近的订单状态
    - 事务内容：对于任意一个终端，从固定的仓库(每个终端都会对应到一个特定的仓库)随机选取一个销售点及其内客户，读取其最后一条订单，显示订单内每件商品的状态
    - 主要特点：只读频率低，要求响应快
- Stock-Level: 查询最近售卖的产品的库存状况，检查哪些库存较低，以便及时补货
    - 事物内容：对于任意一个终端，从固定的仓库(每个终端都会对应到一个特定的仓库)和销售区(每个终端都会对应到特定仓库下的一个特定的销售区)随机选取最后20条订单，查看订单中所有的货物的库存，计算并显示所有库存低于随机生成域值的商品数量
    - 主要特点：只读频率低，较宽松的响应时间
    
对于前四种类型的交易，要求响应时间在5秒以内；对于库存状况查询交易，要求响应时间在20秒以内。

TPC-C的测试结果主要有两个指标：
- 流量指标(Throughput，简称tpmC)
    按照TPC的定义，流量指标描述了系统在执行Payment、Order-status、Delivery、Stock-Level这四种交易的同时，每分钟可以处理多少个New-Order交易。所有交易的响应时间必须满足TPC-C测试规范的要求。

    流量指标值越大越好！
    
    TPC-C结果中还会展示Efficiency，Efficiency的计算：1.0 * tpmc * 100 / numWarehouses / 12.86

- 性价比(Price/Performance，简称Price/tpmC)
    即测试系统价格（指在美国的报价）与流量指标的比值。

    性价比越小越好！

### YugaByte TPC-C benchmark
YugaByte使用的TPC-C benchmark是fork自OLTPBench，但是从OLTPBench中移除了除TPC-C之外的其它测试，同时使用YugaByte专有驱动来和数据库集群通信。

其中几个参数：
- terminals：终端，实际上就是TPCC测试线程总数目
    - 数目被默认设置为warehouse的数目乘以10，当然也可以通过xml配置文件设置之
    - 在实现上，terminals在所有的warehouse中均匀分配，每一个terminal都是一个TPCCWorker
- loaderThreads：加载数据的线程数目，可以通过tpcbenchmark命令行中--loaderthreads进行设置，默认值是min{10, warehouse数目}
    - 控制load过程的线程数目
- numDBConnections：数据库连接的数目，可以通过tpccbenchmark命令行中--num-connections参数指定，如果没有指定该参数的话，默认是min{warehouse数目，节点数目*200}
    - 如果numDBConnections小于loaderThreads，则会将之设置为loaderThreads 
    - 如果数据库集群中有n个节点，则每个节点对应numDBConnections/n个连接
- batchSize：可以通过xml配置文件设置，默认值是128
    - 主要在load过程中使用
- weights：用于按照指定的比例模拟生成下一个事务，可以通过xml文件配置 ，weights字符串中所有的数字的和应该是100
- rate：可以是disabled，或者unlimited，或者一个数字，可以通过xml文件设置之

关于YugaByte TPC-C benchmark测试工具的说明：
1. 在测试过程中总是出现duplicate request错误，TPC-C会在发生该错误的情况下，进行回滚，导致数据并没有成功写入到，所以在后续测试的时候会出现找不到记录的情况。经过代码分析，产生duplicate request的原因是发生了超时，所以对YugaByte的超时机制进行了分析，并尝试对--retryable_rpc_single_call_timeout_ms从2500ms更新为25000ms，以及对--pg_yb_session_timeout_ms从60000ms更新为120000ms，后续测试就再也没有出现超时错误了，也没有出现duplicate request了。

### sysbench
sysbench可以测试MySQL和Postgres等数据库，也可以测试CPU，memory和I/O等系统能力。

关于sysbench测试工具的说明：
1. Sysbench中包含8中测试负载，分别是:insert, point_select, write_only, read_only, read_write, update_index, update_non_index, delete。
2. Sysbench中包含一个skip_trx参数，如果该参数被设置，则不会显式的启动事务，并且所有的操作都在AUTOCOMMIT模式下执行，默认是false。
3. Sysbench中skip_trx默认为false，对于write_only, read_only, read_write操作会在每次测试中显式的添加begin和commit语句，所以对于write_only, read_only, read_write操作，是在一个transaction block中的。
4. Sysbench中，对于insert, point_select, update_index, update_non_index, delete操作会默认在AUTOCOMMIT模式下执行，相当于在测试中的每条语句执行前后都隐式的加一个begin和commit语句。

### YugaByte sysbench
YugaByte使用的sysbench是fork自官方的sysbench，但是做了少许修改，以更好的反映YugaByte的分布式特性。

关于YugaByte sysbench的特别说明：
1. YugaByte在sysbench中添加了range_key_partitioning参数(默认是false)，如果设置了该参数，则YugaByte在创建表的时候会按照range进行分区，否则默认采用hash分区，但是YugaByte在采用range分区的时候，存在一个问题，那就是select count(*)貌似有误，比如我插入10M行记录，但是select count(*)结果为125M行记录。这个问题具体原因不清楚，但是在yugabyte中range partition下的确存在[类似的bug](https://github.com/yugabyte/yugabyte-db/issues/4455)。

2. YugaByte中如果设置了range_key_partitioning参数为false，会按照hash进行分区，则sysbench进行read_only，read_write测试的时候，因为会涉及range查询，导致性能非常差。


## TPC-C测试
YugaByte官方测试见[这里](https://docs.yugabyte.com/latest/benchmark/tpcc-ysql/)。

官方测试中没有明确说明测试所使用的的YugaByte版本，我们的测试采用YugaByte2.1。官方测试中采用的是AWS的云主机进行测试，我们则采用物理机进行测试。

官方测试中TPC-C测试工具和数据库集群采用相同的配置，分别占用1个和3个节点，总共4个节点。官方测试用到了c5d.large和c5d.4xlarge两种云主机，这两种云主机的配置如下：

型号 | vCPU | 内存 (GiB) | 实例存储 (GiB) | 网络带宽 (Gbps) | EBS 带宽 (Mbps)
---|---|---|---|---|---
c5d.large |	2 | 4 | 1 个 50 NVMe SSD |	最高 10	| 最高 4750
c5d.4xlarge | 16 | 32 |	1 个 400 NVMe SSD |	最高 10	| 4750

我们的物理主机配置如下：
用途 | CPU | 内存 (GiB) | 存储 (GiB) | 网络带宽 (Gbps)
---|---|---|---|---|---
数据库集群 |2 sockets, 8 cores/socket, 2 threads/core | 160 | 1 * 800G SATA SSD |	10000Mb/s
测试客户端 |16 sockets, 1 cores/socket, 1 threads/core 虚拟机| 32 | 1 * 400G HDD |	10000Mb/s

### 测试配置
```
<?xml version="1.0"?>
<parameters>
    <dbtype>postgres</dbtype>
    <driver>org.postgresql.Driver</driver>
    <port>5433</port>
    <username>yugabyte</username>
    <DBName>yugabyte</DBName>
    <password>yugabyte</password>
    <isolation>TRANSACTION_REPEATABLE_READ</isolation>

    <!--
    <terminals>100</terminals>
    -->

    <batchSize>128</batchSize>
    <useKeyingTime>true</useKeyingTime>
    <useThinkTime>true</useThinkTime>
    <enableForeignKeysAfterLoad>true</enableForeignKeysAfterLoad>
    <hikariConnectionTimeoutMs>180000</hikariConnectionTimeoutMs>
   	<transactiontypes>
    	<transactiontype>
    		<name>NewOrder</name>
    	</transactiontype>
    	<transactiontype>
    		<name>Payment</name>
    	</transactiontype>
    	<transactiontype>
    		<name>OrderStatus</name>
    	</transactiontype>
    	<transactiontype>
    		<name>Delivery</name>
    	</transactiontype>
    	<transactiontype>
    		<name>StockLevel</name>
    	</transactiontype>
   	</transactiontypes>
    <works>
        <work>
          <time>1800</time>
          <rate>10000</rate>
          <ratelimited bench="tpcc">true</ratelimited>
          <weights>45,43,4,4,4</weights>
        </work>
    </works>
</parameters>
```


### 测试结果
warehouses | terminals | TPMC | Efficiency | details
---|---|---|---|---
10 | 100 | 126.9 | 98.68% | 10warehouses && oltpbench-10.csv
100 | 1000 | 1,273.47 | 99.03% | 100warehouses && oltpbench-100.csv
1000 | 1000 | 1,262.53 | 9.82% | 1000warehouses-1 && oltpbench-1000-1.csv
1000 | 10000 | 12,628.8 | 98.2% | 1000warehouses-2 && oltpbench-1000-2.csv
2000 | 10000*2 | 22867 | 89% | 2000warehouses-part1 && 2000warehouses-part2 && oltpbench-2000-part1.csv && oltpbench-2000-part2.csv 备注：该测试启动了2个tpcc client，每个cleint 10000个线程
4000 | 10000*4 | 15414 | 40% | 备注：该测试启动了4个tpcc client，每个client 10000个线程，但是最终第4个client在启动的时候提示OOM，没有成功启动，所以实际上只有3个client在运行
4000 | 5000*4 | 22640 | 45.61% | 备注：该测试启动了4个tpcc client，每个cleint 5000个线程，但是数据是沿用的上一个4000 warehouse测试加载的数据，没有重新加载数据

## sysbench测试
### 10张表，每张表100k行记录
#### 测试1
测试参数配置：
```
tables=10
table-size=100000
threads={8, 16, 32, 64, 128, 256}
thread-init-timeout=180
time=120
warmup-time=120
```

测试所用sysbench client数目：1

Threads | Workload | Throughput (txns/sec) | Avg Latency (ms) | 95th percentile Latency (ms)
---|---|---|---|---
16 | read_only | 1110 | 14.4 | 15.8
32 | read_only | 1847 | 17.3 | 20
64 | read_only | 3336 | 19.18 | 21.50
128 | read_only | 2987 | 42.8 | 52.9
16 | read_write | 444 | 36 | 66.8
32 | read_write | 592 | 54 | 114.7
64 | read_write | 609 | 105 | 244
128 | read_write | 488 | 262 | 658
16 | write_only | 1310 | 12.21 | 13.95
32 | write_only | 1499 | 21.3 | 29.7
64 | write_only | 1873 | 34 | 51
128 | write_only | 1915 | 66.8 | 110.7
128 | point_select | 61132 | 2 | 4
16 | point_select | 21298 | 0.75 | 0.97
32 | point_select | 45879 | 0.7 | 0.89
64 | point_select | 64890 | 0.98 | 1.42
16 | insert | 3953 | 4.04 | 5.88
32 | insert | 4805 | 6.66 | 10.27
64 | insert | 4294 | 14.9 | 28.7
128 | insert | 4479 | 28.6 | 54.8
16 | update_index | 1526 | 10.50 | 9.22
32 | update_index | 2329 | 13.7 | 17.3
64 | update_index | 3492 | 18.30 | 30.26
128 | update_index | 3566 | 35.7 | 61
16 | update_non_index | 5426 | 2.95 | 3.82
32 | update_non_index | 
64 | update_non_index | 8196 | 7.8 | 11.2
128 | update_non_index | 9763 | 13.1 | 19
16 | delete | 23474 | 0.68 | 0.83
32 | delete | 39446 | 0.81 | 1.12
64 | delete | 53111 | 1.2 | 1.93
128 | delete | 54203 | 2.4 | 4.6


#### 测试2
测试参数配置：
```
tables=10
table-size=100000
threads={8, 16, 32, 64, 128, 256}
time=300
warmup-time=60
thread-init-timeout=180
# 采用hash分区
range_key_partitioning=false
# 对于read_only和read_write测试负载下，不执行range查询相关的测试
range_selects=false
```

测试所用sysbench client数目：3

另外，配置了每个ysql_num_shards_per_tserver为16，在3个节点的集群中，每个table将被拆分为16*3=48个tablets(默认情况下，ysql_num_shards_per_tserver为8，3节点集群中每个table将被拆分为24个tablets)。

Threads | Workload | Throughput (txns/sec) | Avg Latency (ms) | 95th percentile Latency (ms)
---|---|---|---|---
read_only(屏蔽了其中的range查询相关的测试) | 8 | 3819.95 | 6.28 | 6.91
read_only(屏蔽了其中的range查询相关的测试) | 16 | 7175.83 | 6.68667 | 7.30
read_only(屏蔽了其中的range查询相关的测试) | 32 | 11068.6 | 8.67 | 10.65
read_only(屏蔽了其中的range查询相关的测试) | 64 | 14811.8 | 12.9567 | 15.27
read_only(屏蔽了其中的range查询相关的测试) | 128 | 14649.4 | 26.2 | 35.59
read_only(屏蔽了其中的range查询相关的测试) | 256 | 13775.8 | 55.72 | 95.81
read_write(屏蔽了其中的range查询相关的测试) | 8 | 1186.46 | 20.2267 | 21.50
read_write(屏蔽了其中的range查询相关的测试) | 16 | 1284.16 | 37.36 | 116.80
read_write(屏蔽了其中的range查询相关的测试) | 32 | 1418.13 | 67.68 | 308.84
read_write(屏蔽了其中的range查询相关的测试) | 64 | 1626.38 | 118.513 | 390.30
read_write(屏蔽了其中的range查询相关的测试) | 128 | 1516.02 | 257.117 | 623.33
read_write(屏蔽了其中的range查询相关的测试) | 256 | 1432.48 | 550.573 | 1678.14
write_only | 8 | 1504.84 | 15.9367 | 16.71
write_only | 16 | 1643.26 | 29.1833 | 59.99
write_only | 32 | 1745.06 | 55.68 | 173.58
write_only | 64 | 1787.19 | 110.063 | 325.98
write_only | 128 | 1690.43 | 232.813 | 719.92
write_only | 256 | 1170 | 296.27 | 1836.24
point_select | 8 | 39564.1 | 0.603333 | 0.74
point_select | 16 | 73690 | 0.65 | 0.81
point_select | 32 | 111285 | 0.863333 | 1.23
point_select | 64 | 147584 | 1.3 | 2.03
point_select | 128 | 144823 | 2.65 | 5.47
point_select | 256 | 136274 | 5.63333 | 15.83
insert | 8 | 4962.39 | 4.83333 | 7.84
insert | 16 | 5395.95 | 8.89667 | 18.28
insert | 32 | 5597.52 | 17.2633 | 42.61
insert | 64 | 5691.04 | 33.7967 | 81.48
insert | 128 | 5760.54 | 66.7033 | 155.80
insert | 256 | 6019.16 | 134.85 | 292.60
update_index | 8 | 3310.34 | 7.24333 | 8.58
update_index | 16 | 3767.15 | 12.7533 | 20.74
update_index | 32 | 3964.89 | 24.35 | 56.84
update_index | 64 | 4003.03 | 48.0367 | 114.72
update_index | 128 | 3973.6 | 96.77 | 240.02
update_index | 256 | 3986.69 | 197.013 | 909.80
update_non_index | 8 | 7294.2 | 3.29 | 4.41
update_non_index | 16 | 8685.66 | 5.52667 | 9.56
update_non_index | 32 | 9447.64 | 10.15 | 20.00
update_non_index | 64 | 9927.99 | 19.34 | 39.65
update_non_index | 128 | 10416.6 | 36.9067 | 74.46
update_non_index | 256 | 11091.2 | 69.3633 | 127.81
delete | 8 | 36601.4 | 0.653333 | 0.81
delete | 16 | 65739.9 | 0.726667 | 0.95
delete | 32 | 101722 | 0.943333 | 1.34
delete | 64 | 124002 | 1.55 | 2.57
delete | 128 | 122553 | 3.13 | 6.55
delete | 256 | 117913 | 6.51 | 17.01

#### 测试3
测试参数配置：
```
tables=10
table-size=1000000
threads={8, 16, 32, 64, 128, 256}
time=300
warmup-time=60
thread-init-timeout=180
# 采用hash分区
range_key_partitioning=false
# 对于read_only和read_write测试负载下，不执行range查询相关的测试
range_selects=false
```

测试所用sysbench client数目：3

另外，配置了每个ysql_num_shards_per_tserver为16，在3个节点的集群中，每个table将被拆分为16*3=48个tablets(默认情况下，ysql_num_shards_per_tserver为8，3节点集群中每个table将被拆分为24个tablets)。

Threads | Workload | Throughput (txns/sec) | Avg Latency (ms) | 95th percentile Latency (ms)
---|---|---|---|---
read_only | 32 | 11343.8 | 8.46 | 9.56
read_only | 64 | 14146.5 | 13.5667 | 15.83
read_only | 128 | 14031.8 | 27.3567 | 36.24
read_only | 256 | 13260 | 57.88 | 95.81
read_write | 32 | 1482.54 | 64.72 | 287.38
read_write | 64 | 1645.54 | 118.08 | 376.49
read_write | 128 | 1682.26 | 238.373 | 719.92
read_write | 256 | 1601.92 | 482.457 | 1589.90
write_only | 32 | 1851.89 | 51.8233 | 147.61
write_only | 64 | 1873.49 | 103.24 | 282.25
write_only | 128 | 1770.7 | 226.88 | 816.63
write_only | 256 | 1135.27 | 306.94 | 1803.47
point_select | 32 | 114775 | 0.833333 | 1.16
point_select | 64 | 141201 | 1.35667 | 2.14
point_select | 128 | 138649 | 2.76667 | 5.57
point_select | 256 | 131234 | 5.85 | 16.41
insert | 32 | 5508.51 | 17.43 | 38.94
insert | 64 | 5636.13 | 34.1267 | 82.96
insert | 128 | 5627.65 | 68.36 | 170.48
insert | 256 | 5736.79 | 142.493 | 314.45
update_index | 32 | 3978 | 24.1467 | 52.89
update_index | 64 | 3949.23 | 48.6933 | 125.52
update_index | 128 | 3983.77 | 96.7333 | 253.35
update_index | 256 | 3936.36 | 195.967 | 1013.60
update_non_index | 32 | 9361.78 | 10.2567 | 20.37
update_non_index | 64 | 9886.25 | 19.42 | 38.94
update_non_index | 128 | 10331.4 | 37.2 | 77.19
update_non_index | 256 | 10971.2 | 69.8867 | 130.13
delete | 32 | 22517.2 | 4.26 | 8.43
delete | 64 | 28416.1 | 6.75333 | 18.28
delete | 128 | 32504.1 | 11.81 | 63.32
delete | 256 | 35788 | 21.4633 | 132.49

### 32张表，每张表800K行记录(跟TiDB对比)
> **关于该测试的说明**：

> 在进行该测试的时候，sysbench命令行中设置了--range_key_partitioning=true，该参数下yugabyte会将整个key空间划分为24份，然后在创建table的时候会指定SPLIT AT VALUES(...)参数，当前这个参数还处于BETA，主要用于按照范围分区的表，预先设定每个分区的范围。但是在使用SPLIT AT VALUES参数的时候，如果设置每张表10M行记录，但是当执行完插入之后在某张表上进行select count(*)的时候，发现该表有125M行记录，以为是YugaByte或者Sysbench的问题，所以在测试的时候设置的table-size是800000，这样进行select count(*)的时候显示是10M行记录。

> 但是后来分析了YugaByte改进后的Sysbench代码，怀疑前面出现的“实际插入800000行记录，但是select count(*)结果为10M行记录”的情况可能与SPLIT AT VALUES(...)参数有关，自己建立了一个表，并且在创建表的时候同样指定SPLIT AT VALUES(...)参数，结果是插入8条记录，select count(*)显示12条记录，但是select * 结果显示是对的。于是实锤了在使用SPLIT AT VALUES(...)参数的情况下，YugaByte的select count(*)可能有问题。也就是说，指定table-size为800000的情况下，可能表中并没有10000000条记录。

> **所以下面的测试结果并不能反应每个表10M行记录的情况。**

测试参数配置：
```
tables=32
table-size=800000
# run_threads参数可以通过测试命令行修改
run_threads=${run_threads:-64}
time=120
warmuptime=120
```

测试所用sysbench client数目：3

Workload | Threads | Throughput (txns/sec) | Avg Latency (ms) | 95th percentile Latency (ms)
---|---|---|---|---
read_only | 8 | 1175.38 | 20.4133 | 24.83
read_only | 16 | 2043.41 | 23.4867 | 29.19
read_only | 32 | 2695.57 | 35.6333 | 47.47
read_only | 64 | 2810.27 | 68.3633 | 99.33
read_only | 128 | 2917.03 | 132.27 | 253.35
read_write | 8 | 722.96 | 33.1933 | 41.10
read_write | 16 | 972.107 | 49.3767 | 75.82
read_write | 32 | 1087.85 | 88.25 | 144.97
read_write | 64 | 1116.23 | 172.06 | 297.92
read_write | 128 | 1308.05 | 293.63 | 502.20
write_only | 8 | 1613.75 | 14.8667 | 17.95
write_only | 16 | 1753.17 | 27.41 | 41.85
write_only | 32 | 1871.04 | 51.2933 | 89.16
write_only | 64 | 1964.38 | 97.7133 | 183.21
write_only | 128 | 2160.35 | 177.7 | 383.33
point_select | 8 | 37671.4 | 0.636667 | 0.78
point_select | 16 | 70083 | 0.683333 | 0.87
point_select | 32 | 105022 | 0.913333 | 1.34
point_select | 64 | 137573 | 1.39333 | 2.26
point_select | 128 | 126265 | 3.04667 | 6.67
insert | 8 | 4594.97 | 5.25667 | 8.43
insert | 16 | 5379.62 | 9.05 | 18.28
insert | 32 | 5887.44 | 16.6767 | 40.37
insert | 64 | 6508.71 | 30.38 | 82.96
insert | 128 | 6804.4 | 58.4933 | 161.51
update_index | 8 | 2288.05 | 10.4833 | 9.22
update_index | 16 | 2766.49 | 17.3533 | 23.10
update_index | 32 | 3506.34 | 27.3833 | 137.35
update_index | 64 | 3984.01 | 48.2867 | 121.08
update_index | 128 | 4866.85 | 79.0567 | 223.34
update_non_index | 8 | 7229.09 | 3.31667 | 4.41
update_non_index | 16 | 8942.68 | 5.37667 | 8.58
update_non_index | 32 | 10386.2 | 9.26333 | 17.01
update_non_index | 64 | 12854.7 | 14.9633 | 24.83
update_non_index | 128 | 15942.1 | 24.0867 | 33.72
delete | 8 | 28558.5 | 0.84 | 1.61
delete | 16 | 35038.1 | 1.37 | 3.68
delete | 32 | 21603.7 | 4.44333 | 8.28
delete | 64 | 19604.9 | 9.8 | 29.19
delete | 128 | 83801.2 | 4.58333 | 10.46

### 16张表，每张表10M行记录(跟TiDB对比)
> **关于该测试的说明**：

> 在进行该测试的时候，用到16张表，每张表中记录数为10M条

> 在进行该测试的时候，sysbench命令行中设置了--range_key_partitioning=false，则指定table-size为10M，每张表中确实有10M条记录

> 在进行该测试的时候，sysbench命令行中设置了--range_key_partitioning=false，则YugaByte会按照hash进行分区

> sysbench测试中read_only和read_write的负载都会进行range查询，在基于hash分区的情况下，range查询的性能急剧下降，read_only和read_write负载下的测试结果非常差，所以对于read_only和read_write这2种测试，在sysbench命令行中设置了--range_key_partitioning=true，也就是说这2种测试是在range分区下进行的
 

测试参数配置：
```
tables=16
table-size=10000000
# run_threads参数可以通过测试命令行修改
run_threads=${run_threads:-64}
time=300
warmuptime=60
```

测试所用sysbench client数目：3

Workload | Threads | Throughput (txns/sec) | Avg Latency (ms) | 95th percentile Latency (ms)
---|---|---|---|---
read_only(range partition) | 64 | 4462.94 | 43.3833 | 66.84
read_only(range partition) | 128 | 4900.26 | 78.2767 | 112.67
read_only(range partition) | 256 | 5094.05 | 150.787 | 240.02
read_write(range partition) | 64 | 1192.39 | 161.193 | 231.53
read_write(range partition) | 128 | 1371.18 | 281.043 | 442.73
read_write(range partition) | 256 | 1570.6 | 488.463 | 707.07
write_only | 64 | 1610.3 | 119.34 | 257.95
write_only | 128 | 1540.8 | 252.18 | 623.33
write_only | 256 | 2698.26 | 282.37 | 376.49
point_select | 64 | 136837 | 1.4 | 2.30
point_select | 128 | 134989 | 2.84333 | 5.67
point_select | 256 | 129646 | 5.92333 | 16.41
insert | 64 | 4514.34 | 42.5633 | 97.55
insert | 128 | 4647.84 | 82.9533 | 176.73
insert | 256 | 4910.05 | 168.073 | 303.33
update_index | 64 | 3763.51 | 52.2633 | 139.85
update_index | 128 | 3316.63 | 116.757 | 277.21
update_index(range partition) | 256 | 3845.48 | 218.977 | 1129.24
update_non_index | 64 | 9785 | 19.6433 | 43.39
update_non_index | 128 | 10409.3 | 36.98 | 77.19
update_non_index(range partition) | 256 | 18694.1 | 41.08 | 61.08
delete | 64 | 7505.04 | 25.57 | 75.82
delete | 128 | 8006.34 | 48.0033 | 158.63
delete(range partition) | 256 | 15725.8 | 49.9267 | 219.36

