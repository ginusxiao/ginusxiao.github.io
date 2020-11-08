# 提纲
[toc]

## ~~实现思路1~~(废弃)
1. 建立一个关于“实时伴随的anchor号码”的cassandra table，暂且记为AccompanyAnchorNum，其中包括的字段：号码，算法，其它配置字段；

2. 从LatestNCall生成一个RDD，暂且记为LatestNCallRDD，从Latest1Call生成一个RDD，暂且记为Latest1CallRdd;

3. 从AccompanyAnchorNum生成RDD，暂且记为AccompanyAnchorNumRDD；

4. 对AccompanyAnchorNumRDD和Latest1CallRdd进行join，获得每个号码当前的基站信息，暂且记为AccompanyAnchorNumStationRDD；

5. 对AccompanyAnchorNumStationRDD和LatestNCallRDD进行join，获得每个基站当前有哪些人，这些人作为邻居，join之后的RDD暂且记为NeighbourRDD； 

6. 对NeighbourRDD进行map，计算相似度；


### 疑问


## 实现思路2
### 整体实现思路
1. 用spark streaming来接收从UI上传递过来的实时伴随任务；
    - 专门由一个线程负责执行；
    - 对于任何通过spark streaming接收到的实施伴随任务都当做是一个新的实时伴随任务，即使它已经存在；
        - 避免可能出现的，evt/cdr kafka流导致实时伴随任务延迟较长时间才被处理的情况； 
    - 首先，计算当前这个实时伴随任务的伴随情况，将计算出来的伴随情况保存在名为accompanyResult的cassandra table中；
        - 计算逻辑见“单个号码的实时伴随计算”；
        - 如果计算出错，则accompanyResult table中的statusCode和errorMsg字段会被设置；
        - 如果计算成功，则accompanyResult table中的statusCode字段设置为0，errorMsg字段为空；
    - 然后，将这个新的实时伴随任务相关的参数信息添加到名为anchorNum的cassandra table中；
        - anchorNum需要自定义UDT，schema为<String, UDT>
2. UI直接调用cassandra table相关接口去accompanyResult table中获取实时伴随结果；
    - 在UI发送一个关于某个号码的新的实施伴随任务之后，过一定时间(比如1s)之后去获取结果；
    - 如果UI没有获取到相关的记录，则给出提示信息；
    - 如果UI获取到伴随结果，但是statusCode不为0，则根据errorMsg字段给出提示信息；
    - 如果UI获取到伴随结果，且statusCode为0，则展示伴随结果；
3. 后台还有一个定时任务，负责周期性的去重新计算anchorNum中的所有号码的最新的伴随情况；
    - anchorNum中所有号码可以进行人为的分区，交给多个线程来进行处理，每个分区对应一个线程；
    - 在每个线程内部，逐一计算各号码的伴随情况；
        - 计算逻辑见“单个号码的实时伴随计算”；

### 单个号码的实时伴随计算
1. 当前进行实时伴随计算的号码记为anchorNum；
2. 获取anchorNum当前所在的geoHash及其相邻的geoHashes；
    - 首先通过cassandra driver操作Latest1Call，获取geoHash：select geoHash from Latest1Call where usernum = anchorNum;
    - 计算geoHash所有相邻的geoHash，记为neighbourGeoHashes；
    - 获取neighbourGeoHashes中具有基站信息的所有geoHashes(某些计算出来的geoHash没有对应的cgiList)，记为effectiveGeoHashes；
        - 方案1：
            - 此时在geoHash2CgiListRDD上操作：geoHash2CgiListRDD.where("geoHash in ?", effectiveGeoHashes)?
            - 是否可以使用where in? 
                - 参考：https://www.datastax.com/blog/2015/06/deep-look-cql-where-clause
        - 方案2：
            - 在effectiveGeoHashesSet上操作：求neighbourGeoHashes和effectiveGeoHashesSet的交集；
            - 该方案更好?
3. 获取effectiveGeoHashes中的所有的号码组成的集合neighbours(每个元素中只包括usernum这一个字段)；
    - 直接借助cassandra driver在Latest1Call上执行：select usernum from Latest1Call where geoHash in effectiveGeoHashes；
    - 这里需要在geoHash之上建立二级索引；
        - 深入cassandra二级索引：https://www.datastax.com/blog/2016/04/cassandra-native-secondary-index-deep-dive
        - 深入理解cassandra where语句：https://www.datastax.com/blog/2015/06/deep-look-cql-where-clause
            - cassandra可以在非primary key字段上进行where in查询，但是它实际上是先进行scan，然后进行filter，这样的查询效率不高，所以最好要建立二级索引；
                - 但是如果要在某个表上建立二级索引，则该表必须开启事务(见：https://docs.yugabyte.com/latest/api/ycql/ddl_create_index/#create-an-index-for-query-by-the-jsonb-attribute-product-name)； 
            - 但是对于非partition column和clustering column，不支持in语句；
                - 一种解决方案是：将Latest1Call中所有字段组合成一个partition key；
4. 基于LatestNCall构建LatestNCallRDD；
5. 基于neighbours构建RDD，记为neighbourUsernumRDD；
    - 设定其分区方式跟LatestNCall一样；
        - 借助repartitionByCassandraReplica；
6. 对neighboursUsernumRDD和LatestNCallRDD进行join；
    - 借助joinWithCassandraTable；
    - join后的结果是：所有相邻号码对应的轨迹信息，记为neighbourTracksRDD；
7. 对neighbourTracksRdd在每一个分区内部计算相似度；
    - mapPartitions?
        - java api中没有提供mapPartitions?

### 相关cassandra table设计
1. geoHash2CgiList table
    - 用于存放geoHash到cgi list的映射；
    - ~~这个表是静态的，在程序初始启动的时候被加载到geoHash2CgiListRDD中~~；
    - ~~加载之后，这个表可以被persist/cache~~；
    - 将geoHash2CgiListRDD中所有的keys形成一个java hashSet：effectiveGeoHashesSet；
        - 这在获取有效的geoHashes列表的时候会比较有用；
    - schema：geoHash(String)，cgiList(String)
2.  LatestNCall
    - 用于获取每个电话号码当前的轨迹信息；
        - neighboursUsernumRDD和LatestNCallRDD进行join后就获得了对应的号码的轨迹信息；
        - 对于anchorNum，则执行LatestNCall.where("usernum = ?", ${anchorNum})来获取它对应的轨迹信息；
    - 如果是关于一个新的电话号码的实施伴随任务，则会新创建一个新的LatestNCallRDD;
    - 如果是后台任务在进行recompute，此时对所有需要进行recompute的号码共用一个LatestNCallRDD；
    - schema：usernum(String, primary key), List(beginTime, lai, homeArea, curArea, ci, spcode, geoHash, longitude, latitude)
3.  Latest1Call
    - 用于获取anchorNum最新的geoHash；
    - 用于获取anchorNum对应的homeArea；
    - 用于获取当前在某个基站中的所有电话号码；
        - 有以下方案：
            - scan，然后filter，实际上就是select usernum where geoHash in (${geoHash list});
                - 性能堪忧；
            - 在geoHash字段上建立索引，且geoHash字段作为组合的partitionKey的一部分，然后执行select usernum where geoHash in (${geoHash list});
                - 前提是Latest1Call这个表必须是开启了事务，且geoHash字段作为组合的partitionKey的一部分；
    - schema: usernum(String primary key), beginTime(long), homeArea(String), curArea(String), geoHash(String), longitude(double), latitude(double), area(String);
        - 可能需要在curArea，geoHash，area等字段上进行查询； 
4.  anchorNum table
    - 记录被布控的实施伴随任务；
    - schema：usernum(String, primary key)，min_similarity(double)，algorithm(String)，...
5.  accompanyResult
    - 记录实施伴随结果；
    - schema：anchorNum(String, partition key), accompanyNum(String, clustering key), tag(int, 用于标记是否是朋友，是否是同一个homeArea的人), similarity(double), statusCode(int), errMsg(String), insertTime(long，标记该条记录插入时间，用于相似度衰减和清理)
    - 获取某个anchorNum对应的伴随结果：select * from accompanyResult where anchorNum = ${anchorNum};

## 关于某个AccompanyNum在上一轮实施伴随计算中是邻居，但是当前不是邻居的处理
专门启动一个线程，扫描accompanyResult这个表，根据insertTime字段进行相似度衰减处理，直至清理。
