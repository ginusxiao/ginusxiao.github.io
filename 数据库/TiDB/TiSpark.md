# 提纲
[toc]

## TiSpark简介
- TiSpark = Spark SQL on TiKV
- TiSpark 是一款深度订制的 Spark Connection Layer
- TiSpark直接对接存储层（TiKV 和 TiFlash）读取数据，并下推可能的计算以加速
- 通过对接 Spark 的 Extension 接口，TiSpark 得以在不直接修改 Spark 源代码的前提下，深度订制 Spark SQL 的根本行为，包括加入算子，扩充语法，修改执行计划等等
- TiSpark当前是一个只读系统(不支持直接将数据写入TiDB集群，但是可以使用Spark原生的JDBC进行写入)
- 为什么不直接使用Spark on JDBC（或者说Spark on TiDB），而是另外创建TiSpark？
    - [有了spark jdbc为什么还要tispark](http://www.zdingke.com/2019/02/26/tidb-%E7%B3%BB%E5%88%97%E4%B8%89%EF%BC%9A%E6%9C%89%E4%BA%86sparkjdbc%E4%B8%BA%E4%BB%80%E4%B9%88%E8%BF%98%E8%A6%81tispark/)


## 演讲
### [当Apache Spark遇见TIDB](https://www.slidestalk.com/u180/When_Apache_Spark_meets_TiDB_)
1. TiSpark所做的工作
- 比较复杂的计算下推
- Key Range Pruning（？这是什么意思）
- 索引
- 基于代价的优化（CBO）

2. TiSpark Architecture

    [TiSpark Architecture](https://note.youdao.com/yws/public/resource/120c035709eb4fb7e29ecb41a653292d/xmlnote/3F860E21290D435290FFFE318AB221E8/81376)
- Spark Driver
    - 读取TiDB元信息（比如表的schema信息），构造Spark Catalog
    - 通过Placement Driver定位数据所在的Region和获取当前的时间戳
        - 将coprocessor请求发送至Region所在的TiKV
        - 获取时间戳是为了进行快照读取
    - 劫持和改写Spark SQL的逻辑执行计划，添加和TiKV兼容的物理算子：
        - 哪些谓词可以转换成索引相关的访问，哪些谓词可以转换成key range相关的访问；
        - 哪些计算可以下推到TiKV，哪些则需要反推回给Spark；
        - 如何劫持Spark SQL逻辑执行计划的？
            - [劫持Spark SQL](https://note.youdao.com/yws/public/resource/120c035709eb4fb7e29ecb41a653292d/xmlnote/0DFC504210E449918828C508897001A5/81490)
    - 将查询任务按照Region拆分
        - 增加并发，加快查询速度
- Spark Executor
    - 将Spark SQL编码成TiKV的coprocessor请求
    - 将TiKV的coprocessor处理的结果翻译成Spark SQL Rows
    - 

## 其它资料
[TiSpark: More Data Insights, No More ETL](https://www.percona.com/community-blog/2018/06/18/tispark-data-insights-no-etl/)

[TiSpark 原理](https://tidb.xx.cn/views/common/audition.html?courseId=750081&courseWareId=870966&designId=930410)