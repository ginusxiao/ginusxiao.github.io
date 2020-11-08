# 提纲
[toc]

## 测试方案
1. 创建名为latestNCall的table
```
CREATE TABLE IF NOT EXISTS latestNCall (usernum text, begin_time bigint, home_area text, cur_area text, geo_hash text, longtitude double, latitude double, lai int, ci text, spcode text,  primary key(usernum, begin_time));
```

2. 分别测试Unprepared statement, Prepared statement和Batch statement
```
    # 测试Unprepared statement
    Long startNum = 13000000000L;
    Long beginTime = System.currentTimeMillis();
    logger.info("Unprepared statement begin @" + System.currentTimeMillis());
    for (int i = 0; i < 1000000; i++) {
        String usernum = "\'" + (startNum + i) + "\'";
        String homeArea = "\'027\'";
        String curArea = "\'0757\'";
        String geoHash = "\'w3eader\'";
        String ci = "\'211715973\'";
        String spcode = "\'1\'";

        session.execute("INSERT INTO " + latestNCallTableFullName +
                " (usernum, begin_time, home_area, cur_area, geo_hash, longtitude, latitude, lai, ci, spcode) " +
                "VALUES ("+ usernum + "," +  beginTime + "," + homeArea + "," + curArea + "," +
                geoHash + "," + 135.12345 + "," + 32.123456 + "," + 10438 + "," + ci + "," + spcode + ");");
    }
    logger.info("Unprepared statement end @" + System.currentTimeMillis());
    
    # 测试Prepared statement
    startNum = 15000000000L;
    String latestNCallInsertStmt = "INSERT INTO " + latestNCallTableFullName +
        " (usernum, begin_time, home_area, cur_area, geo_hash, longtitude, latitude, lai, ci, spcode) " + "VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?);";
    PreparedStatement latestNCallInsertPreparedStmt = prepare(latestNCallInsertStmt);
        logger.info("Prepared statement begin @" + System.currentTimeMillis());
    for (int i = 0; i < 1000000; i++) {
        executePreparedStatement(latestNCallInsertPreparedStmt, String.valueOf(startNum + i),
                beginTime, "027", "0757", "w3eader", 135.12345, 32.123456, 10438, "211715973", "1");
    }
    logger.info("Prepared statement end @" + System.currentTimeMillis());
    
    # 测试Batch statement
    BatchStatement latestNCallInsertBatchStmt = new BatchStatement();
    startNum = 18000000000L;
    logger.info("Batch statement begin @" + System.currentTimeMillis());
    for (int i = 0; i < 1000000; i++) {
        latestNCallInsertBatchStmt.add(latestNCallInsertPreparedStmt.bind(String.valueOf(startNum + i),
                beginTime, "027", "0757", "w3eader", 135.12345, 32.123456, 10438, "211715973", "1"));
        if (latestNCallInsertBatchStmt.size() == 200) {
            executeBatchStatement(latestNCallInsertBatchStmt);
            latestNCallInsertBatchStmt.clear();
        }
    }
```

## 测试记录
```
1线程，10w次测试：
20/04/03 11:43:43 INFO StoreEnv: Unprepared statement begin  @1585885423605
20/04/03 11:45:20 INFO StoreEnv: Unprepared statement end @1585885520563
20/04/03 11:45:20 INFO StoreEnv: Prepared statement begin @1585885520564
20/04/03 11:46:38 INFO StoreEnv: Prepared statement end @1585885598268
20/04/03 11:46:38 INFO StoreEnv: Batch statement begin @1585885598268
20/04/03 11:46:41 INFO StoreEnv: Batch statement end @1585885601151

Unprepared insert花费97s；
Prepared insert花费了78s；
Batch Prepared insert（每200个作为一个batch）花费了3s；


1线程，100w次测试：
20/04/03 12:28:49 INFO StoreEnv: Unprepared statement begin @1585888129171
20/04/03 12:44:51 INFO StoreEnv: Unprepared statement end @1585889091603
20/04/03 12:44:51 INFO StoreEnv: Prepared statement begin @1585889091603
20/04/03 12:55:45 INFO StoreEnv: Prepared statement end @1585889745042
20/04/03 12:55:45 INFO StoreEnv: Batch statement begin @1585889745042
20/04/03 12:56:12 INFO StoreEnv: Batch statement end @1585889772887

Unprepared insert花费962s；
Prepared insert花费了653s；
Batch Prepared insert（每200个作为一个batch）花费了28s；
```

## 测试结论
线程数|insert次数|Unprepared statement所需时间|Prepared statement所需时间|Batch statement所需时间
---|---|---|---|---
1|10w|97s|78s|3s(每200个insert组成一个batch)
1|100w|962s|653s|28s(每200个insert组成一个batch)
