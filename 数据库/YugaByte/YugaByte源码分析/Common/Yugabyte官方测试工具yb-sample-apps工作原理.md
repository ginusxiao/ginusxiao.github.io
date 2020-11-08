# 提纲
[toc]

## 源码解读
### 测试主程序入口
```
  public static void main(String[] args) throws Exception {
    BasicConfigurator.configure();
    # 解析命令行参数，形成配置信息
    CmdLineOpts configuration = CmdLineOpts.createFromArgs(args);
    
    # Main的构造方法中主要设置配置信息，启动对应的测试实例(配置参数中会指定启动哪种类型的实例)，
    # 比如我们在测试YugaByte CQL的时候常用的是CassandraKeyValue类型
    Main main = new Main(configuration);
    
    # 
    main.run();
    
    LOG.info("The sample app has finished");
  }
  
  # Main构造方法
  public Main(CmdLineOpts cmdLineOpts) {
    # 设置配置参数
    this.cmdLineOpts = cmdLineOpts;
    
    # 启动指定类型的测试实例
    this.app = cmdLineOpts.createAppInstance(cmdLineOpts.getNumWriterThreads() != 0 &&
                                             cmdLineOpts.shouldDropTable());
    this.app.setMainInstance(true);
  }

  public void run() throws Exception {
    try {
      // If this is a simple app, run it and return.
      if (app.appConfig.appType == AppConfig.Type.Simple) {
        app.run();
        return;
      }

      # (如果需要)创建table
      app.createTablesIfNeeded();

      // Create the reader and writer threads.
      int idx = 0;
      # 创建writer线程
      for (; idx < cmdLineOpts.getNumWriterThreads(); idx++) {
        iopsThreads.add(new IOPSThread(idx, cmdLineOpts.createAppInstance(),
                                       IOType.Write, app.appConfig.printAllExceptions));
      }
      
      # 创建reader线程
      for (; idx < cmdLineOpts.getNumWriterThreads() + cmdLineOpts.getNumReaderThreads(); idx++) {
        iopsThreads.add(new IOPSThread(idx, cmdLineOpts.createAppInstance(),
                                       IOType.Read, app.appConfig.printAllExceptions));
      }

      # 启动所有的线程
      // Start the reader and writer threads.
      for (IOPSThread iopsThread : iopsThreads) {
        iopsThread.start();
        Thread.sleep(50);
      }

      # 等待所有的线程执行完成
      // Wait for the various threads to exit.
      while (!iopsThreads.isEmpty()) {
        try {
          iopsThreads.get(0).join();
          iopsThreads.remove(0);
        } catch (InterruptedException e) {
          LOG.error("Error waiting for thread join()", e);
        }
      }
    } finally {
      terminate();
    }
  }
```

### 测试线程IOPSThread
#### 创建IOPSThread
创建IOPSThread的时候会指定线程的类型：IOType.Write和IOType.Read，分别表示写测试线程和读测试线程。
```
  public IOPSThread(int threadIdx, AppBase app, IOType ioType, boolean printAllExceptions) {
    this.threadIdx = threadIdx;
    this.app = app;
    this.ioType = ioType;
    this.printAllExceptions = printAllExceptions;
  }
```

#### IOPSThread线程运行
```
  public void run() {
    try {
      int numConsecutiveExceptions = 0;
      while (!app.hasFinished()) {
        try {
          # 如果是读线程，则执行读测试，否则是写线程，则执行写测试
          switch (ioType) {
            case Write: app.performWrite(threadIdx); break;
            case Read: app.performRead(); break;
          }
          numConsecutiveExceptions = 0;
        } catch (RuntimeException e) {
          ...
        } 
      } 
    } finally {
      # 测试线程退出
      LOG.debug("IOPS thread #" + threadIdx + " finished");
      app.terminate();
    }
  }
```

### App执行读写测试
因为读测试和写测试实现逻辑比较类似，下面以读逻辑为例进行分析。
```
  # performRead是在AppBase中定义的，AppBase是所有不同类型的测试实例(比如CassandraKeyValue,
  # RedisKeyValue等)的父类
  public void performRead() {
    # 检查是否达到了结束测试的条件
    // If we have read enough keys we are done.
    if (appConfig.numKeysToRead >= 0 && numKeysRead.get() >= appConfig.numKeysToRead
        || isOutOfTime()) {
      hasFinished.set(true);
      return;
    }
    // Perform the read and track the number of successfully read keys.
    long startTs = System.nanoTime();
    # 执行读测试，doRead在AppBase中的实现是直接返回0，所以需要在继承AppBase的子类中
    # 重载doRead()方法，以实现相应的读逻辑
    long count = doRead();
    long endTs = System.nanoTime();
    if (count > 0) {
      numKeysRead.addAndGet(count);
      if (metricsTracker != null) {
        metricsTracker.getMetric(MetricName.Read).accumulate(count, endTs - startTs);
      }
    }
  }
```

#### CassandraKeyValue的doRead实现
```
  # CassandraKeyValue相关的doRead是在CassandraKeyValueBase中定义的，CassandraKeyValueBase
  # 是为了提供Cassandra KV读写的抽象
  public long doRead() {
    # 生成一个key
    SimpleLoadGenerator.Key key = getSimpleLoadGenerator().getKeyToRead();
    if (key == null) {
      // There are no keys to read yet.
      return 0;
    }
    
    # 生成BoundStatement，并执行之
    // Do the read from Cassandra.
    // Bind the select statement.
    BoundStatement select = bindSelect(key.asString());
    ResultSet rs = getCassandraClient().execute(select);
    
    # 处理执行结果
    List<Row> rows = rs.all();
    if (rows.size() != 1) {
      // If TTL is enabled, turn off correctness validation.
      if (appConfig.tableTTLSeconds <= 0) {
        LOG.fatal("Read key: " + key.asString() + " expected 1 row in result, got " + rows.size());
      }
      return 1;
    }
    
    if (appConfig.valueSize == 0) {
      ByteBuffer buf = rows.get(0).getBytes(1);
      String value = new String(buf.array());
      key.verify(value);
    } else {
      ByteBuffer value = rows.get(0).getBytes(1);
      byte[] bytes = new byte[value.capacity()];
      value.get(bytes);
      verifyRandomValue(key, bytes);
    }
    LOG.debug("Read key: " + key.toString());
    return 1;
  }
```