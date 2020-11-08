# 提纲
[toc]

## 测试说明
- 测试中会用到两条轨迹信息，分别是out1.json和out2.json，这两条轨迹是比较相似的；
- 每个号码50条轨迹；
- 测试中会用到两组号码，一组从15000000001到15000000020，共20个，记为group1，另一组从15000000020到15000050020，共50000个，记为group2；
- group1中的所有的号码的轨迹都相同，且都来自于out1.json，group2中的所有的号码的轨迹都相同，且都来自于out2.json；
- group1中的所有的号码彼此都是friend；
- 所有的号码都相邻；
- 轨迹相似度采用TRaDW算法，最小相似度设定为0.1，friendsOnly设置为false；
- 以下测试均在一个节点上运行；

## 测试结果
### 基于Ignite

在计算开始之前会打印begin @的字样，在计算完成之后会打印end @的字样，表明开始和完成的时间，时间单位是ms，测试结果显示每一个号码的实时伴随计算时间均不超过1.5s：
```
/var/log/ivylite/ivylite.log:20-04-01 16:00:08,393 rest-#158%STREAMING-SERVER% INFO  - Usernum 15000000002 begin @1585728008393
/var/log/ivylite/ivylite.log:20-04-01 16:00:09,886 pub-#159%STREAMING-SERVER% INFO  - Usernum 15000000002 end @1585728009886
/var/log/ivylite/ivylite.log:20-04-01 16:05:05,649 rest-#200%STREAMING-SERVER% INFO  - Usernum 15000000003 begin @1585728305649
/var/log/ivylite/ivylite.log:20-04-01 16:05:07,018 pub-#201%STREAMING-SERVER% INFO  - Usernum 15000000003 end @1585728307018
/var/log/ivylite/ivylite.log:20-04-01 16:05:07,593 rest-#202%STREAMING-SERVER% INFO  - Usernum 15000000003 begin @1585728307593
/var/log/ivylite/ivylite.log:20-04-01 16:05:08,951 pub-#203%STREAMING-SERVER% INFO  - Usernum 15000000003 end @1585728308951
/var/log/ivylite/ivylite.log:20-04-01 16:05:52,239 rest-#209%STREAMING-SERVER% INFO  - Usernum 15000000004 begin @1585728352239
/var/log/ivylite/ivylite.log:20-04-01 16:05:53,436 pub-#210%STREAMING-SERVER% INFO  - Usernum 15000000004 end @1585728353436
/var/log/ivylite/ivylite.log:20-04-01 16:05:54,188 rest-#211%STREAMING-SERVER% INFO  - Usernum 15000000004 begin @1585728354188
/var/log/ivylite/ivylite.log:20-04-01 16:05:55,489 pub-#212%STREAMING-SERVER% INFO  - Usernum 15000000004 end @1585728355489
/var/log/ivylite/ivylite.log:20-04-01 16:06:08,632 rest-#215%STREAMING-SERVER% INFO  - Usernum 15000000005 begin @1585728368632
/var/log/ivylite/ivylite.log:20-04-01 16:06:09,927 pub-#216%STREAMING-SERVER% INFO  - Usernum 15000000005 end @1585728369927
/var/log/ivylite/ivylite.log:20-04-01 16:06:10,262 rest-#217%STREAMING-SERVER% INFO  - Usernum 15000000005 begin @1585728370262
/var/log/ivylite/ivylite.log:20-04-01 16:06:11,753 pub-#218%STREAMING-SERVER% INFO  - Usernum 15000000005 end @1585728371753
/var/log/ivylite/ivylite.log:20-04-01 16:06:20,874 rest-#220%STREAMING-SERVER% INFO  - Usernum 15000000006 begin @1585728380874
/var/log/ivylite/ivylite.log:20-04-01 16:06:22,295 pub-#221%STREAMING-SERVER% INFO  - Usernum 15000000006 end @1585728382295
/var/log/ivylite/ivylite.log:20-04-01 16:06:22,651 rest-#222%STREAMING-SERVER% INFO  - Usernum 15000000006 begin @1585728382651
/var/log/ivylite/ivylite.log:20-04-01 16:06:24,057 pub-#223%STREAMING-SERVER% INFO  - Usernum 15000000006 end @1585728384057
/var/log/ivylite/ivylite.log:20-04-01 16:06:36,283 rest-#225%STREAMING-SERVER% INFO  - Usernum 15000000007 begin @1585728396283
/var/log/ivylite/ivylite.log:20-04-01 16:06:37,693 pub-#226%STREAMING-SERVER% INFO  - Usernum 15000000007 end @1585728397693
/var/log/ivylite/ivylite.log:20-04-01 16:06:38,013 rest-#228%STREAMING-SERVER% INFO  - Usernum 15000000007 begin @1585728398013
/var/log/ivylite/ivylite.log:20-04-01 16:06:39,348 pub-#229%STREAMING-SERVER% INFO  - Usernum 15000000007 end @1585728399348
/var/log/ivylite/ivylite.log:20-04-01 16:06:49,624 rest-#231%STREAMING-SERVER% INFO  - Usernum 15000000008 begin @1585728409624
/var/log/ivylite/ivylite.log:20-04-01 16:06:50,870 pub-#232%STREAMING-SERVER% INFO  - Usernum 15000000008 end @1585728410870
/var/log/ivylite/ivylite.log:20-04-01 16:06:51,225 rest-#233%STREAMING-SERVER% INFO  - Usernum 15000000008 begin @1585728411225
/var/log/ivylite/ivylite.log:20-04-01 16:06:52,456 pub-#234%STREAMING-SERVER% INFO  - Usernum 15000000008 end @1585728412456
/var/log/ivylite/ivylite.log:20-04-01 16:07:01,788 rest-#236%STREAMING-SERVER% INFO  - Usernum 15000000009 begin @1585728421788
/var/log/ivylite/ivylite.log:20-04-01 16:07:02,952 pub-#237%STREAMING-SERVER% INFO  - Usernum 15000000009 end @1585728422952
/var/log/ivylite/ivylite.log:20-04-01 16:07:03,272 rest-#238%STREAMING-SERVER% INFO  - Usernum 15000000009 begin @1585728423272
/var/log/ivylite/ivylite.log:20-04-01 16:07:04,534 pub-#239%STREAMING-SERVER% INFO  - Usernum 15000000009 end @1585728424534
```

### 基于YugaByte + Spark实现
测试中会在每个号码计算相似度之前和之后分别打印当前的时间戳，以此来统计每个号码相似度计算所花费的时间：
```
20/04/01 16:49:24 INFO BatchRunner: Running for the 1th round
20/04/01 16:49:39 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000016', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:49:52 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000008', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:50:04 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000005', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:50:17 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000001', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:50:30 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000004', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:50:42 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000007', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:50:54 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000010', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:51:06 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000012', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:51:18 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000006', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:51:30 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000013', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:51:42 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000019', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:51:55 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000011', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:52:07 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000014', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:52:19 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000018', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:52:30 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000015', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:52:42 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000003', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:52:54 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000002', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:53:06 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000020', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:53:18 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000009', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:53:30 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000017', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:53:30 INFO BatchRunner: Running for the 1th round done, takes 246426 ms

20/04/01 16:53:30 INFO BatchRunner: Running for the 2th round
20/04/01 16:53:43 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000016', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:53:55 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000008', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:54:07 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000005', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:54:19 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000001', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:54:31 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000004', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:54:43 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000007', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:54:54 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000010', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:55:07 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000012', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:55:18 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000006', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:55:31 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000013', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:55:43 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000019', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:55:55 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000011', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:56:07 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000014', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:56:19 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000018', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:56:31 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000015', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:56:44 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000003', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:56:56 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000002', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:57:08 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000020', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:57:20 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000009', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:57:32 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000017', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 16:57:32 INFO BatchRunner: Running for the 2th round done, takes 241695 ms
```

从上面打印日志来看，每个号码的相似度计算整体花费约12s，包括访问数据库和spark计算。那么这12s的时间，在访问数据库和spark计算上的分配情况是怎样的呢？我对每个号码相似度计算过程分为3个阶段：
- phase 1：获取自身轨迹，获取自身最后一条记录，获取所有邻居号码，获取所有friend，对所有的邻居进行repartition等；
- phase 2：获取repartition之后的每一个partition中的邻居号码对应的轨迹信息；
- phase 3：计算与repartition之后的每一个partition中的邻居号码的相似度，并将伴随结果插入数据库；

摘取一个号码(15000000016)的处理过程，分阶段展示如下：
```
Phase 1 of usernum: 15000000016 begin @1585730966052
Phase 1 of usernum: 15000000016 end @1585730967129


Phase 2 of usernum: 15000000016 begin @1585730967919-> partition: -256640778
Phase 2 of usernum: 15000000016 begin @1585730967921-> partition: 51590350
Phase 2 of usernum: 15000000016 begin @1585730967922-> partition: -2012605118
Phase 2 of usernum: 15000000016 begin @1585730967923-> partition: -1981347187
Phase 2 of usernum: 15000000016 begin @1585730967924-> partition: 395082631
Phase 2 of usernum: 15000000016 begin @1585730967924-> partition: -500286786
Phase 2 of usernum: 15000000016 begin @1585730967926-> partition: 875257411
Phase 2 of usernum: 15000000016 begin @1585730967927-> partition: -635061989
Phase 2 of usernum: 15000000016 begin @1585730967927-> partition: 561231252
Phase 2 of usernum: 15000000016 begin @1585730967930-> partition: 1746074449
Phase 2 of usernum: 15000000016 end @1585730976036-> partition: -500286786
Phase 2 of usernum: 15000000016 end @1585730976049-> partition: 51590350
Phase 2 of usernum: 15000000016 end @1585730976066-> partition: 1746074449
Phase 2 of usernum: 15000000016 end @1585730976070-> partition: -635061989
Phase 2 of usernum: 15000000016 end @1585730976078-> partition: -1981347187
Phase 2 of usernum: 15000000016 end @1585730976081-> partition: -256640778
Phase 2 of usernum: 15000000016 end @1585730976088-> partition: 395082631
Phase 2 of usernum: 15000000016 end @1585730976089-> partition: -2012605118
Phase 2 of usernum: 15000000016 end @1585730976092-> partition: 561231252
Phase 2 of usernum: 15000000016 end @1585730976099-> partition: 875257411


Phase 3 of usernum: 15000000016 begin @1585730976036-> partition: -500286786
Phase 3 of usernum: 15000000016 begin @1585730976049-> partition: 51590350
Phase 3 of usernum: 15000000016 begin @1585730976066-> partition: 1746074449
Phase 3 of usernum: 15000000016 begin @1585730976070-> partition: -635061989
Phase 3 of usernum: 15000000016 begin @1585730976078-> partition: -1981347187
Phase 3 of usernum: 15000000016 begin @1585730976081-> partition: -256640778
Phase 3 of usernum: 15000000016 begin @1585730976088-> partition: 395082631
Phase 3 of usernum: 15000000016 begin @1585730976089-> partition: -2012605118
Phase 3 of usernum: 15000000016 begin @1585730976092-> partition: 561231252
Phase 3 of usernum: 15000000016 begin @1585730976099-> partition: 875257411
Phase 3 of usernum: 15000000016 end @1585730979498-> partition: -500286786
Phase 3 of usernum: 15000000016 end @1585730979498-> partition: 51590350
Phase 3 of usernum: 15000000016 end @1585730979521-> partition: 1746074449
Phase 3 of usernum: 15000000016 end @1585730979531-> partition: -256640778
Phase 3 of usernum: 15000000016 end @1585730979532-> partition: -635061989
Phase 3 of usernum: 15000000016 end @1585730979536-> partition: -2012605118
Phase 3 of usernum: 15000000016 end @1585730979541-> partition: 561231252
Phase 3 of usernum: 15000000016 end @1585730979554-> partition: 395082631
Phase 3 of usernum: 15000000016 end @1585730979556-> partition: -1981347187
Phase 3 of usernum: 15000000016 end @1585730979583-> partition: 875257411
```

从结果来看各阶段花费时间如下：
- phase 1: 1077ms
- phase 2：总共有10个partition，每个partition处理时间多集中在7s到8s，最大的为8173ms
- phase 3：总共有10个partition，每个partition处理时间多集中在3500ms左右，最大的为3500ms

结合各阶段的作用：
- phase 1：获取自身轨迹，获取自身最后一条记录，获取所有邻居号码，获取所有friend，对所有的邻居进行repartition等；
- phase 2：获取repartition之后的每一个partition中的邻居号码对应的轨迹信息；
- phase 3：计算与repartition之后的每一个partition中的邻居号码的相似度，并将伴随结果插入数据库；

可知，在整个12s的处理时间中，phase1和phase2花费的时间主要访问数据库，这个一共消耗了9s多的时间，而spark计算则花费了3s左右的时间。

### 调整测试中neighbour repartition之后的parition数目
需要指出的是，在YugaByte + Spark的方案的上面测试中，partition数目是10个，如果能将partition的数目增大到20甚至更大(比如40)，那么纯计算这块的处理时间可能跟Ignite整体处理时间比较接近，下面不妨一试，将partition数目设置为32个(因为节点的逻辑cpu的数目是32)。

处理每个号码所花费的时间如下：
```
20/04/01 18:02:01 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000016', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:02:11 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000008', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:02:19 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000005', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:02:28 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000001', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:02:37 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000004', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:02:45 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000007', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:02:54 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000010', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:03:02 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000012', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:03:10 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000006', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:03:19 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000013', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:03:27 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000019', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:03:35 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000011', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:03:44 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000014', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:03:52 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000018', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:04:00 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000015', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:04:08 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000003', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:04:16 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000002', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:04:25 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000020', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:04:33 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000009', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
20/04/01 18:04:41 INFO BatchRunner: Process AccompanyAnchor(Batch): AccompanyAnchor{usernum='15000000017', userId='alice', algorithm='TRaDW', minSimilarity=0.1, friendsOnly=false}, Done
```

从上面打印日志来看，每个号码的相似度计算整体花费约8 - 10s，包括访问数据库和spark计算。

摘取一个号码(15000000016)的处理过程，分阶段展示如下：
```
Phase 1 of usernum: 15000000016 begin @1585735311397
Phase 1 of usernum: 15000000016 end @1585735312454


Phase 2 of usernum: 15000000016 begin @1585735313465-> partition: 1236397982
Phase 2 of usernum: 15000000016 begin @1585735313464-> partition: 2042420515
Phase 2 of usernum: 15000000016 begin @1585735313474-> partition: -582056909
Phase 2 of usernum: 15000000016 begin @1585735313479-> partition: -4270188
Phase 2 of usernum: 15000000016 begin @1585735313480-> partition: -1062648354
Phase 2 of usernum: 15000000016 begin @1585735313481-> partition: -1741941168
Phase 2 of usernum: 15000000016 begin @1585735313481-> partition: 28101961
Phase 2 of usernum: 15000000016 begin @1585735313482-> partition: -1952423391
Phase 2 of usernum: 15000000016 begin @1585735313488-> partition: -1128174607
Phase 2 of usernum: 15000000016 begin @1585735313492-> partition: -1719959317
Phase 2 of usernum: 15000000016 begin @1585735313502-> partition: 1189907169
Phase 2 of usernum: 15000000016 begin @1585735313503-> partition: -82615712
Phase 2 of usernum: 15000000016 begin @1585735313505-> partition: 1805746999
Phase 2 of usernum: 15000000016 begin @1585735313507-> partition: -406469567
Phase 2 of usernum: 15000000016 begin @1585735313508-> partition: 590068181
Phase 2 of usernum: 15000000016 begin @1585735313513-> partition: 1492750169
Phase 2 of usernum: 15000000016 begin @1585735313515-> partition: -1184969042
Phase 2 of usernum: 15000000016 begin @1585735313515-> partition: -1991523173
Phase 2 of usernum: 15000000016 begin @1585735313517-> partition: 1501507572
Phase 2 of usernum: 15000000016 begin @1585735313517-> partition: -203485426
Phase 2 of usernum: 15000000016 begin @1585735313518-> partition: 851131018
Phase 2 of usernum: 15000000016 begin @1585735313518-> partition: 1075105837
Phase 2 of usernum: 15000000016 begin @1585735313521-> partition: -1875902719
Phase 2 of usernum: 15000000016 begin @1585735313525-> partition: 2011832268
Phase 2 of usernum: 15000000016 begin @1585735313538-> partition: 964518002
Phase 2 of usernum: 15000000016 begin @1585735313538-> partition: 681669571
Phase 2 of usernum: 15000000016 begin @1585735313538-> partition: 103158997
Phase 2 of usernum: 15000000016 begin @1585735313541-> partition: 54849671
Phase 2 of usernum: 15000000016 begin @1585735313546-> partition: 12084958
Phase 2 of usernum: 15000000016 begin @1585735313547-> partition: 2112161575
Phase 2 of usernum: 15000000016 begin @1585735313547-> partition: -1776951505
Phase 2 of usernum: 15000000016 begin @1585735313547-> partition: 1486597899
Phase 2 of usernum: 15000000016 end @1585735314961-> partition: -1062648354
Phase 2 of usernum: 15000000016 end @1585735314976-> partition: 1236397982
Phase 2 of usernum: 15000000016 end @1585735315012-> partition: -1952423391
Phase 2 of usernum: 15000000016 end @1585735315163-> partition: -4270188
Phase 2 of usernum: 15000000016 end @1585735315205-> partition: -582056909
Phase 2 of usernum: 15000000016 end @1585735315431-> partition: 2042420515
Phase 2 of usernum: 15000000016 end @1585735315477-> partition: 1189907169
Phase 2 of usernum: 15000000016 end @1585735315776-> partition: -1128174607
Phase 2 of usernum: 15000000016 end @1585735315812-> partition: 28101961
Phase 2 of usernum: 15000000016 end @1585735316303-> partition: -1719959317
Phase 2 of usernum: 15000000016 end @1585735316330-> partition: -1741941168
Phase 2 of usernum: 15000000016 end @1585735316681-> partition: -82615712
Phase 2 of usernum: 15000000016 end @1585735316719-> partition: -406469567
Phase 2 of usernum: 15000000016 end @1585735317110-> partition: -1991523173
Phase 2 of usernum: 15000000016 end @1585735317127-> partition: 590068181
Phase 2 of usernum: 15000000016 end @1585735317524-> partition: 1805746999
Phase 2 of usernum: 15000000016 end @1585735317581-> partition: 1492750169
Phase 2 of usernum: 15000000016 end @1585735318158-> partition: 851131018
Phase 2 of usernum: 15000000016 end @1585735318224-> partition: -1184969042
Phase 2 of usernum: 15000000016 end @1585735318538-> partition: 1075105837
Phase 2 of usernum: 15000000016 end @1585735318603-> partition: 2011832268
Phase 2 of usernum: 15000000016 end @1585735318888-> partition: 1501507572
Phase 2 of usernum: 15000000016 end @1585735318892-> partition: 681669571
Phase 2 of usernum: 15000000016 end @1585735319195-> partition: -203485426
Phase 2 of usernum: 15000000016 end @1585735319284-> partition: -1875902719
Phase 2 of usernum: 15000000016 end @1585735319498-> partition: -1776951505
Phase 2 of usernum: 15000000016 end @1585735319522-> partition: 964518002
Phase 2 of usernum: 15000000016 end @1585735319668-> partition: 2112161575
Phase 2 of usernum: 15000000016 end @1585735319682-> partition: 54849671
Phase 2 of usernum: 15000000016 end @1585735319747-> partition: 12084958
Phase 2 of usernum: 15000000016 end @1585735319792-> partition: 1486597899
Phase 2 of usernum: 15000000016 end @1585735319800-> partition: 103158997


Phase 3 of usernum: 15000000016 begin @1585735314961-> partition: -1062648354
Phase 3 of usernum: 15000000016 begin @1585735314976-> partition: 1236397982
Phase 3 of usernum: 15000000016 begin @1585735315012-> partition: -1952423391
Phase 3 of usernum: 15000000016 begin @1585735315163-> partition: -4270188
Phase 3 of usernum: 15000000016 begin @1585735315205-> partition: -582056909
Phase 3 of usernum: 15000000016 begin @1585735315431-> partition: 2042420515
Phase 3 of usernum: 15000000016 begin @1585735315477-> partition: 1189907169
Phase 3 of usernum: 15000000016 end @1585735315747-> partition: -1062648354
Phase 3 of usernum: 15000000016 begin @1585735315777-> partition: -1128174607
Phase 3 of usernum: 15000000016 end @1585735315779-> partition: 1236397982
Phase 3 of usernum: 15000000016 begin @1585735315812-> partition: 28101961
Phase 3 of usernum: 15000000016 end @1585735315814-> partition: -1952423391
Phase 3 of usernum: 15000000016 end @1585735316078-> partition: -4270188
Phase 3 of usernum: 15000000016 end @1585735316253-> partition: -582056909
Phase 3 of usernum: 15000000016 begin @1585735316303-> partition: -1719959317
Phase 3 of usernum: 15000000016 begin @1585735316331-> partition: -1741941168
Phase 3 of usernum: 15000000016 end @1585735316531-> partition: 2042420515
Phase 3 of usernum: 15000000016 end @1585735316588-> partition: 1189907169
Phase 3 of usernum: 15000000016 begin @1585735316682-> partition: -82615712
Phase 3 of usernum: 15000000016 begin @1585735316720-> partition: -406469567
Phase 3 of usernum: 15000000016 end @1585735316971-> partition: -1128174607
Phase 3 of usernum: 15000000016 end @1585735316994-> partition: 28101961
Phase 3 of usernum: 15000000016 begin @1585735317110-> partition: -1991523173
Phase 3 of usernum: 15000000016 begin @1585735317127-> partition: 590068181
Phase 3 of usernum: 15000000016 end @1585735317438-> partition: -1719959317
Phase 3 of usernum: 15000000016 end @1585735317467-> partition: -1741941168
Phase 3 of usernum: 15000000016 begin @1585735317524-> partition: 1805746999
Phase 3 of usernum: 15000000016 begin @1585735317581-> partition: 1492750169
Phase 3 of usernum: 15000000016 end @1585735318147-> partition: -82615712
Phase 3 of usernum: 15000000016 begin @1585735318158-> partition: 851131018
Phase 3 of usernum: 15000000016 end @1585735318203-> partition: -406469567
Phase 3 of usernum: 15000000016 begin @1585735318224-> partition: -1184969042
Phase 3 of usernum: 15000000016 begin @1585735318538-> partition: 1075105837
Phase 3 of usernum: 15000000016 begin @1585735318603-> partition: 2011832268
Phase 3 of usernum: 15000000016 end @1585735318675-> partition: -1991523173
Phase 3 of usernum: 15000000016 end @1585735318707-> partition: 590068181
Phase 3 of usernum: 15000000016 begin @1585735318888-> partition: 1501507572
Phase 3 of usernum: 15000000016 begin @1585735318893-> partition: 681669571
Phase 3 of usernum: 15000000016 end @1585735319157-> partition: 1805746999
Phase 3 of usernum: 15000000016 begin @1585735319195-> partition: -203485426
Phase 3 of usernum: 15000000016 end @1585735319224-> partition: 1492750169
Phase 3 of usernum: 15000000016 begin @1585735319284-> partition: -1875902719
Phase 3 of usernum: 15000000016 begin @1585735319498-> partition: -1776951505
Phase 3 of usernum: 15000000016 begin @1585735319522-> partition: 964518002
Phase 3 of usernum: 15000000016 end @1585735319657-> partition: 851131018
Phase 3 of usernum: 15000000016 begin @1585735319669-> partition: 2112161575
Phase 3 of usernum: 15000000016 begin @1585735319682-> partition: 54849671
Phase 3 of usernum: 15000000016 end @1585735319722-> partition: -1184969042
Phase 3 of usernum: 15000000016 begin @1585735319747-> partition: 12084958
Phase 3 of usernum: 15000000016 begin @1585735319792-> partition: 1486597899
Phase 3 of usernum: 15000000016 begin @1585735319800-> partition: 103158997
Phase 3 of usernum: 15000000016 end @1585735320265-> partition: 1075105837
Phase 3 of usernum: 15000000016 end @1585735320315-> partition: 2011832268
Phase 3 of usernum: 15000000016 end @1585735320697-> partition: 681669571
Phase 3 of usernum: 15000000016 end @1585735320707-> partition: 1501507572
Phase 3 of usernum: 15000000016 end @1585735321063-> partition: -203485426
Phase 3 of usernum: 15000000016 end @1585735321137-> partition: -1875902719
Phase 3 of usernum: 15000000016 end @1585735321407-> partition: -1776951505
Phase 3 of usernum: 15000000016 end @1585735321435-> partition: 964518002
Phase 3 of usernum: 15000000016 end @1585735321626-> partition: 2112161575
Phase 3 of usernum: 15000000016 end @1585735321650-> partition: 54849671
Phase 3 of usernum: 15000000016 end @1585735321739-> partition: 12084958
Phase 3 of usernum: 15000000016 end @1585735321797-> partition: 1486597899
Phase 3 of usernum: 15000000016 end @1585735321829-> partition: 103158997
```

从结果来看各阶段花费时间如下：
- phase 1: 1057ms
- phase 2：总共有32个partition，每个partition处理时间多集中在4.5s左右，最大的为6262ms
- phase 3：总共有32个partition，每个partition处理时间多集中在2s左右，最大的为2029ms

可知，在整个8 - 10s的处理时间中，phase1和phase2花费的时间主要访问数据库，这个一共消耗了6 -7s的时间，而spark计算则花费了2s左右的时间。


## 结论
从上面的分析来看，对于一个号码的实时伴随计算来说，基于Ignie方案整体处理时间为1.5s，而基于YugaByte + Spark的方案，在10个parition的情况下，访问数据库的时间约为9s，而真正计算的时间则为3s(实际上少于3s，因为这个阶段也要插入数据库)，计算部分所花费的时间是Ignite整体处理时间的2倍，究其原因，可能是Spark计算框架的开销比较大，如果是计算量比较大的情况下，从纯计算的角度来说，Spark和Ignite的差距也许没有这么大。在32个partition的情况下，Spark计算这块所花费的时间为2s左右，相比较于10个parition的情况下，提升了1s左右，但是仍然比Ignite差。



