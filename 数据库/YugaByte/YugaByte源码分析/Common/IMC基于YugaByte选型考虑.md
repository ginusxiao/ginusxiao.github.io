# 提纲
[toc]

## 选型考虑
1. YugaByteDB存在的共性问题
    - 非In-Memory或者Memory-Centric
    - 各API操作的数据之间相互独立，无法互通
        - 雷老师意见：只要能保证Spark SQL/Presto来访问不同接口写入的数据即可 
    
2. Data Grid
    - 通过兼容Redis的YEDIS API支持
        - cons：
            - 只实现了Redis有序集合中的部分命令（ZCARD, ZADD, ZRANGEBYSCORE, ZREM, ZRANGE, ZREVRANGE, ZSCORE），而其他的命令，比如ZCOUNT, ZREVRANGEBYSCORE, ZRANK, ZREVRANK等都没有实现
            - List, Bitmaps, HyperLogLogs, GeoSpatial相关的命令也没有实现
            - **在短期之内，YugabyteDB将不会太关注于实现YEDIS的新功能的开发**
    - 通过兼容Cassandra的YCQL API支持
        - cons：
            - 类SQL API
            
3. Database
    - 很好的OLTP支持
        - 生而为之
        - cons：
            - 不支持存储过程和触发器等
                - PG的UDF
    - 较弱的OLAP支持
        - Spark + Cassandra/YugaByteDB
        - Presto + Cassandra/YugaByteDB
        - 然而，这种支持更多的是集成，在存储层面，Cassandra是Wide Column store，而不是columnar/column-store/column-oriented database，所以对于OLAP来说，也不是友好的
    - HTAP方案
        - 参考TiDB：OLTP使用TiDB + TiKV，OLAP则使用TiSpark + TiKV或者TiSpark + TiFlash

4. Streaming capabilities
    - 目前官方明确提到的对streaming的支持：
        - Kafka Connect YugabyteDB Source Connector
            - 借助于YugaByteDB的CDC功能 
        - Kafka Connect YugabyteDB Sink Connector
    - 对Apache Camel，Apache Spark Streaming等则没有明确提到
        - 可能需要自己开发
    - 对于不同的streaming平台提供接口（类似于Ignite中的IgniteStreamer）支持
    - 支持Stream ingestion和Stream store，但是对Stream processing的支持则比较缺乏，而且现在的支持也主要是基于YCQL的（对于YEDIS和YSQL则暂未涉及？）
    
5. Distributed computing
    - 不支持（注：TiDB通过TiSpark提供Map-Reduce计算框架的支持）
    - 计算功能的支持主要包括以下：
        - MapReduce/MPP
        - 负载均衡
        - 自动错误容忍
        - checkpoint或者savepoint
        - 自定义调度策略
        - 分布式闭包
        - ...
    - 和Apache Spark或者Apache Flink等的**深度**整合需要基于对Apache Spark/Apache Flink和YugaByteDB的深入理解
    - 
6. Persistence
    - 本身就是持久化的
    - 需要解决内存存储，以及内存存储和持久化存储之间如何高效并存的问题

7. 多协议接口
    - KV：YEDIS
    - SQL：YSQL和YCQL（类SQL）
    - JDBC/ODBC：支持
    - Restful：不支持
  
8. 多模
    - ~~KV：支持~~
    - RDBMS：支持
    - Document：支持
    - Graph：现已经支持和JanusGraph集成

9. 支持多种编程语言
    - 支持C/C++，Java，Go，Python，Ruby等
    
10. 方便部署
    - 本地，私有云，公有云，混合云或者容器