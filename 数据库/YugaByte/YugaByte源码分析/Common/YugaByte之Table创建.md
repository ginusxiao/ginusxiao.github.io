# 提纲
[toc]

## 整体步骤
1. 步骤1：验证待创建的table schema并为该table创建一定数目的tablets，但是这些tablets尚未分配给YB-TServers。
2. 步骤2：复制table schema和tablets信息到YB-Master Raft group中。
3. 步骤3：返回用户，创建成功的消息。
4. 步骤4：执行。将每一个tablet分配给由Replication Factor确定的数目的YB-TServers上。
5. 步骤5：持续监控。持续监控分配到YB-Tservers上的tablet的进展情况。

## 举例说明
[Table creation](https://docs.yugabyte.com/latest/architecture/core-functions/table-creation/)

## 源码分析

