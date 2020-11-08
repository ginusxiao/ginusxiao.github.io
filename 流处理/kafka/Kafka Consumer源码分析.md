# 提纲
[toc]

# 简介
本文讲述的是kafka 0.8.2.2中的High-level Consumer（相对应的另外一个Consumer API是SimpleConsumer）。

High-level Consumer主要功能的实现都是在类kafka.consumer.ZoookeeperCounsumerConnector.scala中。

# 源码走读
## 关于kafka.consumer.ZoookeeperCounsumerConnector的使用
这里以Ignite中的使用为例，在Ignite中是按照如下的方式使用Kafka Consumer的：
```
    public void start() {
        A.notNull(getStreamer(), "streamer");
        A.notNull(getIgnite(), "ignite");
        A.notNull(topic, "topic");
        A.notNull(consumerCfg, "kafka consumer config");
        A.ensure(threads > 0, "threads > 0");
        A.ensure(null != getSingleTupleExtractor() || null != getMultipleTupleExtractor(),
            "Extractor must be configured");

        log = getIgnite().log();

        # 创建Kafka Consumer Connector
        consumer = kafka.consumer.Consumer.createJavaConsumerConnector(consumerCfg);

        # 创建topic到其对应的线程数目的映射，因为可能有多个topic，所以使用一个HashMap
        Map<String, Integer> topicCntMap = new HashMap<>();

        # 这里实际上在@topicCountMap中只存放了1个topic，因为示例中只会用到1个topic
        topicCntMap.put(topic, threads);

        # 创建Kafka streams，这里传递的参数是@topicCountMap，会为@topicCountMap中所有
        # topic创建Kafka streams，为每个topic创建的Kafka streams数目则由@topicCountMap中
        # 该topic对应的线程数目确定
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCntMap);

        # 获取指定@topic对应的Kafka streams
        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);

        # 创建一个固定数目线程的线程池
        executor = Executors.newFixedThreadPool(threads);

        stopped = false;

        # 依次遍历@topic所关联的每个Kafka Stream
        for (final KafkaStream<byte[], byte[]> stream : streams) {
            # 对于每一个Kafka Stream中的消息单独由线程池中的一个线程来处理
            executor.execute(new Runnable() {
                # 对一个Kafka Stream中消息的处理逻辑
                @Override public void run() {
                    while (!stopped) {
                        try {
                            MessageAndMetadata<byte[], byte[]> msg;

                            # 逐条处理Kafka Stream中的消息
                            for (ConsumerIterator<byte[], byte[]> it = stream.iterator(); it.hasNext() && !stopped; ) {
                                # 获取下一条消息
                                msg = it.next();

                                try {
                                    # 将从Kafka Stream中接收到的消息添加到IgniteDataStreamer中
                                    addMessage(msg);
                                }
                                catch (Exception e) {
                                    U.error(log, "Message is ignored due to an error [msg=" + msg + ']', e);
                                }
                            }
                        }
                        catch (Exception e) {
                            U.error(log, "Message can't be consumed from stream. Retry after " +
                                retryTimeout + " ms.", e);

                            try {
                                Thread.sleep(retryTimeout);
                            }
                            catch (InterruptedException ignored) {
                                // No-op.
                            }
                        }
                    }
                }
            });
        }
    }
```

从上面的使用示例中可以看到，关于Kafka ConsumerConnector的几个主要接口为：

```
# 创建Kafka ConsumerConnector
consumer = kafka.consumer.Consumer.createJavaConsumerConnector(consumerCfg)

# 创建Kafka MessageStreams
Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCntMap)

# 获取指定topic的Kafka Stream列表
List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic)

# 迭代获取Kafka Stream中的消息
ConsumerIterator<byte[], byte[]> it = stream.iterator()
msg = it.next()
```

下面逐一分析这几个接口。

## 创建Kafka ConsumerConnector
ignite-2.6.0/modules/kafka/src/main/java/org/apache/ignite/stream/kafka/KafkaStreamer.java:
```
consumer = kafka.consumer.Consumer.createJavaConsumerConnector(consumerCfg);
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ConsumerConnector.scala:
```
  def createJavaConsumerConnector(config: ConsumerConfig): kafka.javaapi.consumer.ConsumerConnector = {
    val consumerConnect = new kafka.javaapi.consumer.ZookeeperConsumerConnector(config)
    consumerConnect
  }
```

kafka-0.8.2.2/core/src/main/scala/kafka/javaapi/consumer/ZookeeperConsumerConnector.scala:
```
# ZookeeperConsumerConnector类继承自ConsumerConnector
private[kafka] class ZookeeperConsumerConnector(val config: ConsumerConfig,
                                                val enableFetcher: Boolean) // for testing only
    extends ConsumerConnector {
    # 实际上是对kafka.consumer.ZookeeperConsumerConnector的封装
    private val underlying = new kafka.consumer.ZookeeperConsumerConnector(config, enableFetcher)
    private val messageStreamCreated = new AtomicBoolean(false)
    
    # 省略ZookeeperConsumerConnector中各种方法
    ......
}    
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala:
```
private[kafka] class ZookeeperConsumerConnector(val config: ConsumerConfig,
                                                val enableFetcher: Boolean) // for testing only
        extends ConsumerConnector with Logging with KafkaMetricsGroup {

  private val isShuttingDown = new AtomicBoolean(false)
  private val rebalanceLock = new Object
  private var fetcher: Option[ConsumerFetcherManager] = None
  private var zkUtils: ZkUtils = null
  private var topicRegistry = new Pool[String, Pool[Int, PartitionTopicInfo]]
  private val checkpointedZkOffsets = new Pool[TopicAndPartition, Long]
  # topic+threadid到BlockingQueue的映射
  private val topicThreadIdAndQueues = new Pool[(String, ConsumerThreadId), BlockingQueue[FetchedDataChunk]]
  # kafka scheduler线程
  private val scheduler = new KafkaScheduler(threads = 1, threadNamePrefix = "kafka-consumer-scheduler-")
  private val messageStreamCreated = new AtomicBoolean(false)

  # 下面的3个listener分别用于监听session超时事件，topic中partition变化事件和partition负载均衡事件
  private var sessionExpirationListener: ZKSessionExpireListener = null
  private var topicPartitionChangeListener: ZKTopicPartitionChangeListener = null
  private var loadBalancerListener: ZKRebalancerListener = null

  private var offsetsChannel: BlockingChannel = null
  private val offsetsChannelLock = new Object

  # 下面2个listener分别用于监听topic通配情况的变化（比如新添加了满足特定模式的topic）
  # 和consumer负载均衡事件（比如新添加了consumer）
  private var wildcardTopicWatcher: ZookeeperTopicEventWatcher = null
  private var consumerRebalanceListener: ConsumerRebalanceListener = null

  // useful for tracking migration of consumers to store offsets in kafka
  private val kafkaCommitMeter = newMeter("KafkaCommitsPerSec", "commits", TimeUnit.SECONDS, Map("clientId" -> config.clientId))
  private val zkCommitMeter = newMeter("ZooKeeperCommitsPerSec", "commits", TimeUnit.SECONDS, Map("clientId" -> config.clientId))
  private val rebalanceTimer = new KafkaTimer(newTimer("RebalanceRateAndTime", TimeUnit.MILLISECONDS, TimeUnit.SECONDS, Map("clientId" -> config.clientId)))
  
  # 获取string类型的consumerId @consumerIdString
  val consumerIdString = {
    var consumerUuid : String = null
    # 根据配置@config中是否制定consumerId分别处理
    config.consumerId match {
      # 如果指定了consumerId，则直接使用之
      case Some(consumerId) // for testing only
      => consumerUuid = consumerId
      # 如果没有指定，则自动生成一个，组成为${groupId}-${hostname}-${timestamp}-${uuid.substring(0,8)}
      case None // generate unique consumerId automatically
      => val uuid = UUID.randomUUID()
      consumerUuid = "%s-%d-%s".format(
        InetAddress.getLocalHost.getHostName, System.currentTimeMillis,
        uuid.getMostSignificantBits().toHexString.substring(0,8))
    }
    
    config.groupId + "_" + consumerUuid
  }
  this.logIdent = "[" + consumerIdString + "], "

  # 连接Zookeeper，直接调用Zookeeper提供的相关接口
  connectZk()
  
  # 创建ConsumerFetcherManager，顾名思义，ConsumerFetcherManager就是消费者获取线程的管理器
  # 它在内存中维护了两个映射关系：
  # 1. 获取者线程与broker的映射，即每个broker上面都有哪些获取者线程
  # 2. topic分区与分区消费信息的映射，这里的分区消费信息包含很多内容，比如底层的消费队列、
  # 保存到zk上的已消费位移、获取过的最大位移以及获取大小等信息
  # 
  # 有了这些信息，一个消费者线程管理器就可以很方便地对消费者线程进行动态地重分配。
  createFetcher()
  
  # 确保连接上offset管理器（就是管理offset的实例，该方法只在offsetsStorage为Kafka的时候起作用）
  ensureOffsetManagerConnected()

  # 启动一个Kafka调度器，并向该调度器中添加一个autocommit任务（该任务会根据调度相关的设置被调度）
  if (config.autoCommitEnable) {
    scheduler.startup
    info("starting auto committer every " + config.autoCommitIntervalMs + " ms")
    scheduler.schedule("kafka-consumer-autocommit",
                       autoCommit,
                       delay = config.autoCommitIntervalMs,
                       period = config.autoCommitIntervalMs,
                       unit = TimeUnit.MILLISECONDS)
  }

  # 启动统计相关的汇报
  KafkaMetricsReporter.startReporters(config.props)
  AppInfo.registerInfo()
  
  # 省略ZookeeperConsumerConnector中各种方法
  ......
}  
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala:
```
  private def createFetcher() {
    # 创建ConsumerFetcherManager
    if (enableFetcher)
      fetcher = Some(new ConsumerFetcherManager(consumerIdString, config, zkUtils))
  }
```

## 创建KafkaStreams，为从Kafka中消费做准备
ignite-2.6.0/modules/kafka/src/main/java/org/apache/ignite/stream/kafka/KafkaStreamer.java:
```
Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCntMap)
```

kafka-0.8.2.2/core/src/main/scala/kafka/javaapi/consumer/ZookeeperConsumerConnector.scala:
```
def createMessageStreams(topicCountMap: java.util.Map[String,java.lang.Integer]):         
    java.util.Map[String,java.util.List[KafkaStream[Array[Byte],Array[Byte]]]] =
    createMessageStreams(topicCountMap, new DefaultDecoder(), new DefaultDecoder())
    
def createMessageStreams[K,V](
        topicCountMap: java.util.Map[String,java.lang.Integer],
        keyDecoder: Decoder[K],
        valueDecoder: Decoder[V])
      : java.util.Map[String,java.util.List[KafkaStream[K,V]]] = {

    # 如果@messageStreamCreated已经为true，表明它已经被创建
    if (messageStreamCreated.getAndSet(true))
      throw new MessageStreamsExistException(this.getClass.getSimpleName +
                                   " can create message streams at most once",null)
                                   
    #  scalaTopicCountMap中记录的是从topic到从该topic中获取消息的线程数的映射
    val scalaTopicCountMap: Map[String, Int] = {
      import JavaConversions._
      Map.empty[String, Int] ++ (topicCountMap.asInstanceOf[java.util.Map[String, Int]]: mutable.Map[String, Int])
    }
    
    # @underlying见ZookeeperConsumerConnector的构造函数，其类型实际为
    # kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala
    # 中定义的ZookeeperConsumerConnector
    # 
    # 为各topic的每个线程（每个topic可能对应多个线程）分别创建一个KafkaStream
    # @scalaReturn中返回的是一个Map{topic -> List{KafkaStream}}
    val scalaReturn = underlying.consume(scalaTopicCountMap, keyDecoder, valueDecoder)
    val ret = new java.util.HashMap[String,java.util.List[KafkaStream[K,V]]]
    for ((topic, streams) <- scalaReturn) {
      var javaStreamList = new java.util.ArrayList[KafkaStream[K,V]]
      for (stream <- streams)
        javaStreamList.add(stream)
      ret.put(topic, javaStreamList)
    }
    
    # 返回的是HashMap{topic -> ArrayList{KafkaStream}}
    ret
  }   
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala：
```
  def consume[K, V](topicCountMap: scala.collection.Map[String,Int], keyDecoder: Decoder[K], valueDecoder: Decoder[V])
      : Map[String,List[KafkaStream[K,V]]] = {
    debug("entering consume ")
    # @topicCountMap中包含的是从topic到消费该topic的线程数目的映射
    if (topicCountMap == null)
      throw new RuntimeException("topicCountMap is null")

    # 根据@consumerIdString和@topicCountMap生成从topic到关于该topic的所有
    # ConsumerThreadId组成的集合映射
    # 
    # 比如topicCountMap为{("topic1", 2), ("topic2", 1)}，@consumerIdString
    # 为igniteX-csmac-1477361897062-93ec1a0e，则生成的映射表可能如下：
    # {
    #   ("topic1" -> ("igniteX-csmac-1477361897062-93ec1a0e-1", 
    #                 "igniteX-csmac-1477361897062-93ec1a0e-2")),
    #   ("topic2" -> ("igniteX-csmac-1477361897062-93ec1a0e-1") 
    # }
    val topicCount = TopicCount.constructTopicCount(consumerIdString, topicCountMap)

    val topicThreadIds = topicCount.getConsumerThreadIdsPerTopic

    # 为@topicThreadIds中的每个ConsumerThreadId创建一个<queue, stream>对，其中@queue
    # 是一个BlockingQueue类型的阻塞队列，队列中存放从Kafka获取到的消息，消息类型为
    # FetchedDataChunk，@stream是KafkaStream，通过下文中提到的KafkaStream和ConsumerIterator
    # 可以看出，KafkaStream实际上就是一个在BlockingQueue上封装的一个迭代器
    val queuesAndStreams = topicThreadIds.values.map(threadIdSet =>
      threadIdSet.map(_ => {
        val queue =  new LinkedBlockingQueue[FetchedDataChunk](config.queuedMaxMessages)
        val stream = new KafkaStream[K,V](
          queue, config.consumerTimeoutMs, keyDecoder, valueDecoder, config.clientId)
        (queue, stream)
      })
    ).flatten.toList

    # 在zookeeper中创建关于该consumer group的节点
    val dirs = new ZKGroupDirs(config.groupId)
    
    # 将@consumerIdString所代表的的consumer添加到consumer group所在的节点中
    registerConsumerInZK(dirs, consumerIdString, topicCount)
    
    # 注册各种事件监听，并获得Map{topic -> List{KafkaStream}}，要着重关注
    # @queuesAndStreams这个参数
    reinitializeConsumer(topicCount, queuesAndStreams)

    # 返回Map{topic -> List{KafkaStream}}
    loadBalancerListener.kafkaMessageAndMetadataStreams.asInstanceOf[Map[String, List[KafkaStream[K,V]]]]
  }
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/TopicCount.scala：
```
private[kafka] object TopicCount extends Logging {
  val whiteListPattern = "white_list"
  val blackListPattern = "black_list"
  val staticPattern = "static"
  
  # 根据consumerIdString和threadId组合成新的threadId
  def makeThreadId(consumerIdString: String, threadId: Int) = consumerIdString + "-" + threadId

  def makeConsumerThreadIdsPerTopic(consumerIdString: String,
                                    topicCountMap: Map[String,  Int]) = {
    # @consumerThreadIdsPerTopicMap用于容纳从Topic到关于该Topic的所有ConsumerThreadId组成的集合映射
    val consumerThreadIdsPerTopicMap = new mutable.HashMap[String, Set[ConsumerThreadId]]()
    
    # 逐一处理@topicCountMap中的元素
    for ((topic, nConsumers) <- topicCountMap) {
      val consumerSet = new mutable.HashSet[ConsumerThreadId]
      assert(nConsumers >= 1)
      
      # 为当前的@topic生成ConsumerThreadId，并添加到@consumerSet中
      for (i <- 0 until nConsumers)
        consumerSet += ConsumerThreadId(consumerIdString, i)
        
      # 将@topic和关于该Topic的所有ConsumerThreadId组成的集合@consumerSet添加到
      # @consumerThreadIdsPerTopicMap这个映射表中
      consumerThreadIdsPerTopicMap.put(topic, consumerSet)
    }
    
    # 返回@consumerThreadIdsPerTopicMap
    consumerThreadIdsPerTopicMap
  }
  
  ......
  
  # 直接调用StaticTopicCount的构造函数
  def constructTopicCount(consumerIdString: String, topicCount: Map[String, Int]) =
    new StaticTopicCount(consumerIdString, topicCount)
}

# StaticTopicCount继承自TopicCount
private[kafka] class StaticTopicCount(val consumerIdString: String,
                                val topicCountMap: Map[String, Int])
                                extends TopicCount {
  # 根据@topicCountMap生成从topic到关于该topic的所有ConsumerThreadId组成的集合映射，
  # 比如topicCountMap为{("topic1", 2), ("topic2", 1)}，则生成的映射表可能如下：
  # {
  #   ("topic1" -> ("igniteX-csmac-1477361897062-93ec1a0e-1", 
  #                 "igniteX-csmac-1477361897062-93ec1a0e-2")),
  #   ("topic2" -> ("igniteX-csmac-2607981637741-819cef2a-1") 
  # }
  def getConsumerThreadIdsPerTopic = TopicCount.makeConsumerThreadIdsPerTopic(consumerIdString, topicCountMap)

  # 重载equals方法
  override def equals(obj: Any): Boolean = {
    obj match {
      case null => false
      case n: StaticTopicCount => consumerIdString == n.consumerIdString && topicCountMap == n.topicCountMap
      case _ => false
    }
  }

  def getTopicCountMap = topicCountMap

  def pattern = TopicCount.staticPattern
}    
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala：
```
  private def reinitializeConsumer[K,V](
      topicCount: TopicCount,
      queuesAndStreams: List[(LinkedBlockingQueue[FetchedDataChunk],KafkaStream[K,V])]) {
    
    val dirs = new ZKGroupDirs(config.groupId)

    # 监听consumer和partition变更事件
    if (loadBalancerListener == null) {
      val topicStreamsMap = new mutable.HashMap[String,List[KafkaStream[K,V]]]
      loadBalancerListener = new ZKRebalancerListener(
        config.groupId, consumerIdString, topicStreamsMap.asInstanceOf[scala.collection.mutable.Map[String, List[KafkaStream[_,_]]]])
    }

    # 监听session超时事件
    if (sessionExpirationListener == null)
      sessionExpirationListener = new ZKSessionExpireListener(
        dirs, consumerIdString, topicCount, loadBalancerListener)

    # 监听topic partition改变事件
    if (topicPartitionChangeListener == null)
      topicPartitionChangeListener = new ZKTopicPartitionChangeListener(loadBalancerListener)

    val topicStreamsMap = loadBalancerListener.kafkaMessageAndMetadataStreams

    val consumerThreadIdsPerTopic: Map[String, Set[ConsumerThreadId]] =
      topicCount.getConsumerThreadIdsPerTopic

    val allQueuesAndStreams = topicCount match {
      case wildTopicCount: WildcardTopicCount =>
        /*
         * Wild-card consumption streams share the same queues, so we need to
         * duplicate the list for the subsequent zip operation.
         */
        (1 to consumerThreadIdsPerTopic.keySet.size).flatMap(_ => queuesAndStreams).toList
      case statTopicCount: StaticTopicCount =>
        queuesAndStreams
    }

    val topicThreadIds = consumerThreadIdsPerTopic.map {
      case(topic, threadIds) =>
        threadIds.map((topic, _))
    }.flatten

    require(topicThreadIds.size == allQueuesAndStreams.size,
      "Mismatch between thread ID count (%d) and queue count (%d)"
      .format(topicThreadIds.size, allQueuesAndStreams.size))
      
    # 合并@topicThreadIds和@allQueuesAndStreams，即将topic -> Set{consumerThreadId}
    # 和 consumerThreadId -> (Queue，KafkaStream) 这两个映射关系，转换成 
    # List{((topic, consumerThreadId), (Queue, KafkaStream))}
    val threadQueueStreamPairs = topicThreadIds.zip(allQueuesAndStreams)

    # 从@threadQueueStreamPairs中提取Map{(topic, consumerThreadId) -> Queue}，
    # 并添加从topicThreadId到queue的映射到@topicThreadIdAndQueues中
    threadQueueStreamPairs.foreach(e => {
      val topicThreadId = e._1
      val q = e._2._1
      topicThreadIdAndQueues.put(topicThreadId, q)
      debug("Adding topicThreadId %s and queue %s to topicThreadIdAndQueues data structure".format(topicThreadId, q.toString))
      newGauge(
        "FetchQueueSize",
        new Gauge[Int] {
          def value = q.size
        },
        Map("clientId" -> config.clientId,
          "topic" -> topicThreadId._1,
          "threadId" -> topicThreadId._2.threadId.toString)
      )
    })

    # 对@threadQueueStreamPairs，在topic维度先进行聚合，得到 Map{topic -> Set{((topic, 
    # consumerThreadId), (Queue, KafkaStream))}}，即@groupedByTopic然后提取出Map{topic -> 
    # List{KafkaStream}}，即@topicStreamsMap
    val groupedByTopic = threadQueueStreamPairs.groupBy(_._1._1)
    groupedByTopic.foreach(e => {
      val topic = e._1
      val streams = e._2.map(_._2._2).toList
      topicStreamsMap += (topic -> streams)
      debug("adding topic %s and %d streams to map.".format(topic, streams.size))
    })

    // listener to consumer and partition changes
    zkUtils.zkClient.subscribeStateChanges(sessionExpirationListener)

    zkUtils.zkClient.subscribeChildChanges(dirs.consumerRegistryDir, loadBalancerListener)

    topicStreamsMap.foreach { topicAndStreams =>
      // register on broker partition path changes
      val topicPath = BrokerTopicsPath + "/" + topicAndStreams._1
      zkUtils.zkClient.subscribeDataChanges(topicPath, topicPartitionChangeListener)
    }

    # 显式的触发LoadBalance，见ZookeeperConsumerConnector.ZKRebalancerListener.syncedRebalance
    loadBalancerListener.syncedRebalance()
  }
```

## 获取指定topic的Kafka Stream列表
ignite-2.6.0/modules/kafka/src/main/java/org/apache/ignite/stream/kafka/KafkaStreamer.java:
```
# @consumerMap就是一个Map<String, List<KafkaStream<byte[], byte[]>>>，
# key为topic，value为该topic相关的KafkaStream列表（每个线程一个KafkaStream）
# 
# 这里就是通过topic，获取其对应的KafkaStream列表
List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic)
```

## 迭代获取Kafka Stream中的消息
ignite-2.6.0/modules/kafka/src/main/java/org/apache/ignite/stream/kafka/KafkaStreamer.java:
```
# 这里的@stream就是一个KafkaStream
ConsumerIterator<byte[], byte[]> it = stream.iterator()
msg = it.next()
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/KafkaStream.scala：
```
class KafkaStream[K,V](private val queue: BlockingQueue[FetchedDataChunk],
                        consumerTimeoutMs: Int,
                        private val keyDecoder: Decoder[K],
                        private val valueDecoder: Decoder[V],
                        val clientId: String)
   extends Iterable[MessageAndMetadata[K,V]] with java.lang.Iterable[MessageAndMetadata[K,V]] {

  # 迭代器类型为ConsumerIterator
  private val iter: ConsumerIterator[K,V] =
    new ConsumerIterator[K,V](queue, consumerTimeoutMs, keyDecoder, valueDecoder, clientId)

  # 创建一个KafkaStream中消息的迭代器
  def iterator(): ConsumerIterator[K,V] = iter

  # 清理KafkaStream中的消息
  def clear() {
    iter.clearCurrentChunk()
  }

  override def toString(): String = {
     "%s kafka stream".format(clientId)
  }
}
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ConsumerIterator.scala：
```
# ConsumerIterator继承了IteratorTemplate模板类，关于IteratorTemplate，请参考
# https://www.cnblogs.com/huxi2b/p/4378439.html
# 
# 集成字IteratorTemtpleate的子类必须要实现makeNext()方法，用于获取下一个消息
class ConsumerIterator[K, V](private val channel: BlockingQueue[FetchedDataChunk],
                             consumerTimeoutMs: Int,
                             private val keyDecoder: Decoder[K],
                             private val valueDecoder: Decoder[V],
                             val clientId: String)
  extends IteratorTemplate[MessageAndMetadata[K, V]] with Logging {

  private val current: AtomicReference[Iterator[MessageAndOffset]] = new AtomicReference(null)
  private var currentTopicInfo: PartitionTopicInfo = null
  private var consumedOffset: Long = -1L
  private val consumerTopicStats = ConsumerTopicStatsRegistry.getConsumerTopicStat(clientId)

  # 先看makeNext()，它用于获取下一个消息
  protected def makeNext(): MessageAndMetadata[K, V] = {
    var currentDataChunk: FetchedDataChunk = null
    // if we don't have an iterator, get one
    var localCurrent = current.get()
    if(localCurrent == null || !localCurrent.hasNext) {
      # 从下面的if...esle...语句可以看出，如果当前没有数据就会调用channel.take
      # 或者channel.poll，而这里的channel就是BlockingQueue类型
      #
      # 其中BlockingQueue.take()取走BlockingQueue里排在首位的对象,若BlockingQueue
      # 为空，则等待直到Blocking有新的对象被加入为止
      # BlockingQueue.poll(time):取走BlockingQueue里排在首位的对象,若不能立即取出,
      # 则可以等time参数规定的时间,取不到时返回null
      if (consumerTimeoutMs < 0)
        currentDataChunk = channel.take
      else {
        currentDataChunk = channel.poll(consumerTimeoutMs, TimeUnit.MILLISECONDS)
        if (currentDataChunk == null) {
          // reset state to make the iterator re-iterable
          resetState()
          throw new ConsumerTimeoutException
        }
      }
      
      ......
    }

    new MessageAndMetadata(currentTopicInfo.topic,
                           currentTopicInfo.partitionId,
                           item.message,
                           item.offset,
                           keyDecoder,
                           valueDecoder,
                           item.message.timestamp,
                           item.message.timestampType)
  }
  
  override def next(): MessageAndMetadata[K, V] = {
    val item = super.next()
    if(consumedOffset < 0)
      throw new KafkaException("Offset returned by the message set is invalid %d".format(consumedOffset))
    currentTopicInfo.resetConsumeOffset(consumedOffset)
    val topic = currentTopicInfo.topic
    trace("Setting %s consumed offset to %d".format(topic, consumedOffset))
    consumerTopicStats.getConsumerTopicStats(topic).messageRate.mark()
    consumerTopicStats.getConsumerAllTopicStats().messageRate.mark()
    item
  }
  
  ......
  
}
```

## 显式的触发LoadBalance
kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala：
```
  class ZKRebalancerListener(val group: String, val consumerIdString: String,
                             val kafkaMessageAndMetadataStreams: mutable.Map[String,List[KafkaStream[_,_]]])
    extends IZkChildListener {
    
    def syncedRebalance() {
      try {
        cluster = zkUtils.getCluster()
        done = rebalance(cluster)
      }
      
      ......
    }
    
    private def rebalance(cluster: Cluster): Boolean = {
      # 根据@topicCount生成从topic到关于该topic的所有ConsumerThreadId组成的集合映射
      val myTopicThreadIdsMap = TopicCount.constructTopicCount(
        group, consumerIdString, zkUtils, config.excludeInternalTopics).getConsumerThreadIdsPerTopic
     
      # 获取集群中所有brokers列表
      val brokers = zkUtils.getAllBrokersInCluster()
      if (brokers.size == 0) {
        // This can happen in a rare case when there are no brokers available in the cluster when the consumer is started.
        // We log an warning and register for child changes on brokers/id so that rebalance can be triggered when the brokers
        // are up.
        warn("no brokers found when trying to rebalance.")
        zkUtils.zkClient.subscribeChildChanges(BrokerIdsPath, loadBalancerListener)
        true
      }
      else {
        /**
         * fetchers must be stopped to avoid data duplication, since if the current
         * rebalancing attempt fails, the partitions that are released could be owned by another consumer.
         * But if we don't stop the fetchers first, this consumer would continue returning data for released
         * partitions in parallel. So, not stopping the fetchers leads to duplicate data.
         */
        # 先关闭Fetchers，关闭相关获取数据的操作，因为在做均衡的过程中，可能会出现分区
        # 被分配给其他消费者，如果不先暂停所有取数据的操作，那就可能导致数据被重复消费，
        # 所以经过这步操作之后，会把当前Queue中的数据都清空，如果设置了offset自动提交，
        # 则会把当前消费到的最新的offset给提交掉，在均衡操作成功后，由下一个消费者获取，
        # 作为开始获取数据的offset；
        closeFetchers(cluster, kafkaMessageAndMetadataStreams, myTopicThreadIdsMap)
        
        # 如果应用设置了ConsumerRebalanceListener，则在释放partition的所属权之前先调用
        # ConsumerRebalanceListener.beforeReleasePartitions钩子函数
        if (consumerRebalanceListener != null) {
          info("Invoking rebalance listener before relasing partition ownerships.")
          consumerRebalanceListener.beforeReleasingPartitions(
            if (topicRegistry.size == 0)
              new java.util.HashMap[String, java.util.Set[java.lang.Integer]]
            else
              mapAsJavaMap(topicRegistry.map(topics =>
                topics._1 -> topics._2.keys
              ).toMap).asInstanceOf[java.util.Map[String, java.util.Set[java.lang.Integer]]]
          )
        }
        
        # 将在zookeeper中注册的分区所属消费者信息给删除掉，因为后面会重新做分配
        releasePartitionOwnership(topicRegistry)
        # 进行分区分配，就是将每个topic下的每个分区，分配给上面已经有的consumerThreadId，
        # 得到一个 Map { TopicAndPartition(topic, partitioinId) ->  consumerThreadId}，
        # 目前有两种分配方法，一种是Range，一种是RoundRobin，默认是Range
        val assignmentContext = new AssignmentContext(group, consumerIdString, config.excludeInternalTopics, zkUtils)
        val globalPartitionAssignment = partitionAssignor.assign(assignmentContext)
        val partitionAssignment = globalPartitionAssignment.get(assignmentContext.consumerId)
        val currentTopicRegistry = new Pool[String, Pool[Int, PartitionTopicInfo]](
          valueFactory = Some((topic: String) => new Pool[Int, PartitionTopicInfo]))

        // fetch current offsets for all topic-partitions
        val topicPartitions = partitionAssignment.keySet.toSeq

        # 获取当前每个分区的offset
        val offsetFetchResponseOpt = fetchOffsets(topicPartitions)

        # 拼装PartitionTopicInfo，并提取Map{topic -> Map {partition -> PartitionTopicInfo}} ，
        # 这样就可以得到topic的注册信息，也就是 topic -> partition -> Queue 的映射关系，
        # 以及每个 topic -> partition 的当前 consumedOffset，和下次要去获取的 
        # fetchedOffset，这些信息就存在变量 topicRegistry 中
        if (isShuttingDown.get || !offsetFetchResponseOpt.isDefined)
          false
        else {
          val offsetFetchResponse = offsetFetchResponseOpt.get
          topicPartitions.foreach(topicAndPartition => {
            val (topic, partition) = topicAndPartition.asTuple
            val offset = offsetFetchResponse.requestInfo(topicAndPartition).offset
            val threadId = partitionAssignment(topicAndPartition)
            addPartitionTopicInfo(currentTopicRegistry, partition, topic, offset, threadId)
          })

          /**
           * move the partition ownership here, since that can be used to indicate a truly successful re-balancing attempt
           * A rebalancing attempt is completed successfully only after the fetchers have been started correctly
           */
          
          if(reflectPartitionOwnershipDecision(partitionAssignment)) {
            allTopicsOwnedPartitionsCount = partitionAssignment.size

            partitionAssignment.view.groupBy { case(topicPartition, consumerThreadId) => topicPartition.topic }
                                      .foreach { case (topic, partitionThreadPairs) =>
              newGauge("OwnedPartitionsCount",
                new Gauge[Int] {
                  def value() = partitionThreadPairs.size
                },
                ownedPartitionsCountMetricTags(topic))
            }

            topicRegistry = currentTopicRegistry
            # 如果应用设置了ConsumerRebalanceListener，则在释放partition的所属权之前先调用
            # ConsumerRebalanceListener.beforeStartingFetchers钩子函数
            if (consumerRebalanceListener != null) {
              info("Invoking rebalance listener before starting fetchers.")

              // Partition assignor returns the global partition assignment organized as a map of [TopicPartition, ThreadId]
              // per consumer, and we need to re-organize it to a map of [Partition, ThreadId] per topic before passing
              // to the rebalance callback.
              val partitionAssginmentGroupByTopic = globalPartitionAssignment.values.flatten.groupBy[String] {
                case (topicPartition, _) => topicPartition.topic
              }
              val partitionAssigmentMapForCallback = partitionAssginmentGroupByTopic.map({
                case (topic, partitionOwnerShips) =>
                  val partitionOwnershipForTopicScalaMap = partitionOwnerShips.map({
                    case (topicAndPartition, consumerThreadId) =>
                      topicAndPartition.partition -> consumerThreadId
                  })
                  topic -> mapAsJavaMap(collection.mutable.Map(partitionOwnershipForTopicScalaMap.toSeq:_*))
                    .asInstanceOf[java.util.Map[java.lang.Integer, ConsumerThreadId]]
              })
              consumerRebalanceListener.beforeStartingFetchers(
                consumerIdString,
                mapAsJavaMap(collection.mutable.Map(partitionAssigmentMapForCallback.toSeq:_*))
              )
            }
            
            # 更新fetcher，将上述分配好的分区信息再分配给相应的fetcher，用来获取数据
            updateFetcher(cluster)
            true
          } else {
            false
          }
        }
      }
    }  
    
    # 从topicRegistry中提取出所有的 PartitionTopicInfo ，然后传输给fetcher的
    # startConnections 方法，用来启动fetcher线程
    private def updateFetcher(cluster: Cluster) {
      // update partitions for fetcher
      var allPartitionInfos : List[PartitionTopicInfo] = Nil
      for (partitionInfos <- topicRegistry.values)
        for (partition <- partitionInfos.values)
          allPartitionInfos ::= partition
      info("Consumer " + consumerIdString + " selected partitions : " +
        allPartitionInfos.sortWith((s,t) => s.partitionId < t.partitionId).map(_.toString).mkString(","))

      fetcher match {
        case Some(f) =>
          f.startConnections(allPartitionInfos, cluster)
        case None =>
      }
    }
```
上面的过程实际上是一个负载均衡的过程，将各topic的所有partition按照指定的partition算法分配给consumer中的各consumer thread。在负载均衡之前要先关闭Fetcher，在负载均衡分配完成之后则重新启动Fetcher。

##  Fetcher从Broker获取数据
kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ZookeeperConsumerConnector.scala:
```
private[kafka] class ZookeeperConsumerConnector(val config: ConsumerConfig,
                                                val enableFetcher: Boolean) // for testing only
        extends ConsumerConnector with Logging with KafkaMetricsGroup {
        
  ......
    
  createFetcher()
    
  ......
    
  private def createFetcher() {
    # 创建ConsumerFetcherManager
    if (enableFetcher)
      fetcher = Some(new ConsumerFetcherManager(consumerIdString, config, zkUtils))
  }
}
```

kafka-0.8.2.2/core/src/main/scala/kafka.consumer.ConsumerFetcherManager.scala：
```
class ConsumerFetcherManager(private val consumerIdString: String,
                             private val config: ConsumerConfig,
                             private val zkUtils : ZkUtils)
        extends AbstractFetcherManager("ConsumerFetcherManager-%d".format(SystemTime.milliseconds),
                                       config.clientId, config.numConsumerFetchers) {
  private var partitionMap: immutable.Map[TopicAndPartition, PartitionTopicInfo] = null
  private var cluster: Cluster = null
  private val noLeaderPartitionSet = new mutable.HashSet[TopicAndPartition]
  private val lock = new ReentrantLock
  private val cond = lock.newCondition()
  private var leaderFinderThread: ShutdownableThread = null
  private val correlationId = new AtomicInteger(0)

  def startConnections(topicInfos: Iterable[PartitionTopicInfo], cluster: Cluster) {
    # 首先会新建一个LeaderFinderThread线程，该线程就是用来为那些leader发生变化的分区，
    # 重新找到其leader，然后分配给相应的fetcherThread去获取数据
    leaderFinderThread = new LeaderFinderThread(consumerIdString + "-leader-finder-thread")
    leaderFinderThread.start()

    inLock(lock) {
      # 根据传入的 List{PartitionTopicInfo}，来初始化 partitionMap{TopicAndPartition ->
      # PartitionTopicInfo}，其中@topicInfos是从ZKRebalancerListener.Rebalance中
      # PartitionTopicInfo中包含
      partitionMap = topicInfos.map(tpi => (TopicAndPartition(tpi.topic, tpi.partitionId), tpi)).toMap
      this.cluster = cluster
      # 先把所有的TopicAndPartition，都加入到noLeaderPartitionSet中，用来寻找其leader
      noLeaderPartitionSet ++= topicInfos.map(tpi => TopicAndPartition(tpi.topic, tpi.partitionId))
      cond.signalAll()
    }
  }
  
  private class LeaderFinderThread(name: String) extends ShutdownableThread(name) {
    // thread responsible for adding the fetcher to the right broker when leader is available
    override def doWork() {
      val leaderForPartitionsMap = new HashMap[TopicAndPartition, BrokerEndPoint]
      lock.lock()
      try {
        while (noLeaderPartitionSet.isEmpty) {
          trace("No partition for leader election.")
          cond.await()
        }
        
        # LeaderFinderThread 线程会不停的对 noLeaderPartitionSet 做检查，发现其不为空，
        # 就会对其中的 TopicAndPartition 寻找新的leader
        trace("Partitions without leader %s".format(noLeaderPartitionSet))
        val brokers = zkUtils.getAllBrokerEndPointsForChannel(SecurityProtocol.PLAINTEXT)
        # 发送fetch topic metadata请求，用于获取各topic的所有partitions对应的leader broker，
        # 返回类似于下：
        # {
        #    topic_name  partition_id    leader_broker_id
        #   {"topic1", {"partition 0", ${broker_id_10}}},
        #   {"topic1", {"partition 1", ${broker_id_12}}},
        #   {"topic2", {"partition 0", ${broker_id_20}}}
        # }
        # 
        # 关于TopicMetadataRequest和TopicMetadataResponse，请参考：
        # https://www.cnblogs.com/Jusfr/p/5257258.html
        val topicsMetadata = ClientUtils.fetchTopicMetadata(noLeaderPartitionSet.map(
            m => m.topic).toSet,
            brokers,
            config.clientId,
            config.socketTimeoutMs,
            correlationId.getAndIncrement).topicsMetadata
        if(logger.isDebugEnabled) topicsMetadata.foreach(topicMetadata => debug(topicMetadata.toString()))
        # 依次获取@topicsMetadata中记录的每一个topic_partition和该topic_partition
        # 所对应的leader broker id，并将该信息添加到@leaderForPartitionsMap中
        topicsMetadata.foreach { tmd =>
          val topic = tmd.topic
          tmd.partitionsMetadata.foreach { pmd =>
            val topicAndPartition = TopicAndPartition(topic, pmd.partitionId)
            if(pmd.leader.isDefined && noLeaderPartitionSet.contains(topicAndPartition)) {
              val leaderBroker = pmd.leader.get
              leaderForPartitionsMap.put(topicAndPartition, leaderBroker)
              noLeaderPartitionSet -= topicAndPartition
            }
          }
        }
      } catch {
        case t: Throwable => {
            if (!isRunning.get())
              throw t /* If this thread is stopped, propagate this exception to kill the thread. */
            else
              warn("Failed to find leader for %s".format(noLeaderPartitionSet), t)
          }
      } finally {
        lock.unlock()
      }

      try {
        # 根据 TopicAndPartition -> leader Broker的映射关系，得到Map{TopicAndPartition ->
        # BrokerAndInitialOffset} 的映射，并调用父类AbstractFetcherManager中的
        # addFetcherForPartitions
        addFetcherForPartitions(leaderForPartitionsMap.map{
          case (topicAndPartition, broker) =>
            topicAndPartition -> BrokerAndInitialOffset(broker, partitionMap(topicAndPartition).getFetchOffset())}
        )
      } catch {
        case t: Throwable => {
          if (!isRunning.get())
            throw t /* If this thread is stopped, propagate this exception to kill the thread. */
          else {
            warn("Failed to add leader for partitions %s; will retry".format(leaderForPartitionsMap.keySet.mkString(",")), t)
            lock.lock()
            noLeaderPartitionSet ++= leaderForPartitionsMap.keySet
            lock.unlock()
          }
        }
      }

      # 清理那些已经没有分区可以取数据的fetcherThread
      shutdownIdleFetcherThreads()
      Thread.sleep(config.refreshLeaderBackoffMs)
    }
  }
}
```

kafka-0.8.2.2/core/src/main/scala/kafka/server/AbstractFetcherManager.scala：
```
abstract class AbstractFetcherManager(protected val name: String, clientId: String, numFetchers: Int = 1)
  extends Logging with KafkaMetricsGroup {
  // map of (source broker_id, fetcher_id per source broker) => fetcher
  private val fetcherThreadMap = new mutable.HashMap[BrokerAndFetcherId, AbstractFetcherThread]
  private val mapLock = new Object
  this.logIdent = "[" + name + "] "
  
  def addFetcherForPartitions(partitionAndOffsets: Map[TopicAndPartition, BrokerAndInitialOffset]) {
    mapLock synchronized {
      # 将从TopicAndPartition到BrokerAndInitialOffset的映射，按照broker进行分组，率属于同一个
      # broker的被合并到一起，最终返回的@paritionsPerFetcher是从broker到以其作为leader broker
      # 的所有partitions列表（这些partitions可能率属于不同的topic）的映射
      val partitionsPerFetcher = partitionAndOffsets.groupBy{ case(topicAndPartition, brokerAndInitialOffset) =>
        BrokerAndFetcherId(brokerAndInitialOffset.broker, getFetcherId(topicAndPartition.topic, topicAndPartition.partition))}
      
      # 为@paritionsPerFetcher中的每一个broker分配1个fetcherThread，并将该fetcherThread
      # 添加到@fetcherThreadMap中，然后启动该fetcherThread
      for ((brokerAndFetcherId, partitionAndOffsets) <- partitionsPerFetcher) {
        var fetcherThread: AbstractFetcherThread = null
        fetcherThreadMap.get(brokerAndFetcherId) match {
          case Some(f) => fetcherThread = f
          case None =>
            fetcherThread = createFetcherThread(brokerAndFetcherId.fetcherId, brokerAndFetcherId.broker)
            fetcherThreadMap.put(brokerAndFetcherId, fetcherThread)
            fetcherThread.start
        }

        # 将从topicAndPartition到其对应的offset的映射添加到fetcherThread的Partition map中
        fetcherThreadMap(brokerAndFetcherId).addPartitions(partitionAndOffsets.map { case (topicAndPartition, brokerAndInitOffset) =>
          topicAndPartition -> brokerAndInitOffset.initOffset
        })
      }
    }
  }
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ConsumerFetcherManager.scala:
```
  override def createFetcherThread(fetcherId: Int, sourceBroker: BrokerEndPoint):
  AbstractFetcherThread = {
    # 创建ConsumerFetcherThread实例，参数中包括@partitionMap，是在
    # ConsumerFetcherManager的startConnections方法中初始化的
    new ConsumerFetcherThread(
      "ConsumerFetcherThread-%s-%d-%d".format(consumerIdString, fetcherId, sourceBroker.id),
      config, sourceBroker, partitionMap, this)
  }
```


kafka-0.8.2.2/core/src/main/scala/kafka/server/AbstractFetcherThread.scala：
```
abstract class AbstractFetcherThread(name: String,
                                     clientId: String,
                                     sourceBroker: BrokerEndPoint,
                                     fetchBackOffMs: Int = 0,
                                     isInterruptible: Boolean = true)
  extends ShutdownableThread(name, isInterruptible) {
  
  override def doWork() {

    val fetchRequest = inLock(partitionMapLock) {
      # 创建fetch request
      val fetchRequest = buildFetchRequest(partitionMap)
      if (fetchRequest.isEmpty) {
        trace("There are no active partitions. Back off for %d ms before sending a fetch request".format(fetchBackOffMs))
        partitionMapCond.await(fetchBackOffMs, TimeUnit.MILLISECONDS)
      }
      fetchRequest
    }

    # 处理fetch request
    if (!fetchRequest.isEmpty)
      processFetchRequest(fetchRequest)
  }  
  
  private def processFetchRequest(fetchRequest: REQ) {
    val partitionsWithError = new mutable.HashSet[TopicAndPartition]
    var responseData: Map[TopicAndPartition, PD] = Map.empty

    try {
      # 执行fetch操作
      responseData = fetch(fetchRequest)
    } catch {
      ......
    }

    if (responseData.nonEmpty) {
      # 处理fetch到的数据
      inLock(partitionMapLock) {

        responseData.foreach { case (topicAndPartition, partitionData) =>
          val TopicAndPartition(topic, partitionId) = topicAndPartition
          partitionMap.get(topicAndPartition).foreach(currentPartitionFetchState =>
            // we append to the log if the current offset is defined and it is the same as the offset requested during fetch
            if (fetchRequest.offset(topicAndPartition) == currentPartitionFetchState.offset) {
              Errors.forCode(partitionData.errorCode) match {
                case Errors.NONE =>
                  try {
                    val messages = partitionData.toByteBufferMessageSet
                    val validBytes = messages.validBytes
                    val newOffset = messages.shallowIterator.toSeq.lastOption match {
                      case Some(m: MessageAndOffset) => m.nextOffset
                      case None => currentPartitionFetchState.offset
                    }
                    partitionMap.put(topicAndPartition, new PartitionFetchState(newOffset))
                    fetcherLagStats.getAndMaybePut(topic, partitionId).lag = Math.max(0L, partitionData.highWatermark - newOffset)
                    fetcherStats.byteRate.mark(validBytes)
                    # 
                    processPartitionData(topicAndPartition, currentPartitionFetchState.offset, partitionData)
                  } catch {
                    ......
                  }
                case Errors.OFFSET_OUT_OF_RANGE =>
                  ......
                case _ =>
                  ......
              }
            })
        }
      }
    }
  }  
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/ConsumerFetcherThread.scala:
```
class ConsumerFetcherThread(name: String,
                            val config: ConsumerConfig,
                            sourceBroker: BrokerEndPoint,
                            partitionMap: Map[TopicAndPartition, PartitionTopicInfo],
                            val consumerFetcherManager: ConsumerFetcherManager)
        extends AbstractFetcherThread(name = name,
                                      clientId = config.clientId,
                                      sourceBroker = sourceBroker,
                                      fetchBackOffMs = config.refreshLeaderBackoffMs,
                                      isInterruptible = true) {
  def processPartitionData(topicAndPartition: TopicAndPartition, fetchOffset: Long, partitionData: PartitionData) {
    val pti = partitionMap(topicAndPartition)
    if (pti.getFetchOffset != fetchOffset)
      throw new RuntimeException("Offset doesn't match for partition [%s,%d] pti offset: %d fetch offset: %d"
                                .format(topicAndPartition.topic, topicAndPartition.partition, pti.getFetchOffset, fetchOffset))
    # 调用的是PartitionTopicInfo.enqueue
    pti.enqueue(partitionData.underlying.messages.asInstanceOf[ByteBufferMessageSet])
  }
  
}  
```

kafka-0.8.2.2/core/src/main/scala/kafka/consumer/PartitionTopicInfo.scala:
```
  def enqueue(messages: ByteBufferMessageSet) {
    val size = messages.validBytes
    if(size > 0) {
      # 
      chunkQueue.put(new FetchedDataChunk(messages, this, fetchedOffset.get))
      fetchedOffset.set(next)
      consumerTopicStats.getConsumerTopicStats(topic).byteRate.mark(size)
      consumerTopicStats.getConsumerAllTopicStats().byteRate.mark(size)
    } else if(messages.sizeInBytes > 0) {
      chunkQueue.put(new FetchedDataChunk(messages, this, fetchedOffset.get))
    }
  }
```


# 参考
[*apache kafka源码分析走读-ZookeeperConsumerConnector分析](https://blog.csdn.net/lizhitao/article/details/38458631)
```
该文中对于client，consumer threads，blocking queue，fetch threads， partitions和broker之间的关系的讲解跟我理解的是一致的，描述的也比较清晰。

具体来说如下：
1个client，也可以称之为1个consumer connector，会启动多个consumer threads（可以通过参数进行设置）；
1个consumer thread对应于1个kafka message stream和1个blocking queue；
1个fetch thread对应于1个broker，负责从该broker中fetch不同以该broker为leader的topic+partition的数据；
1个topic的所有partitions在率属于同一个consumer group中的所有consumer threads之间按照指定的分区分派算法（包括RangeAssigner和RoundRobinAssigner）进行分配；
```

[*Kafka源码系列——Consumer](http://www.ibigdata.io/?p=250)

[kafka consumer源代码分析](https://www.cnblogs.com/huxi2b/p/4563383.html)

[Kafka Consumer接口](http://www.cnblogs.com/fxjwind/p/3794255.html)

[***Fetcher: KafkaConsumer消息消费的管理者](https://blog.csdn.net/zhanyuanlin/article/details/76269308)
```
这篇文章很不错！
```

[apache kafka系列之在zookeeper中存储结构](https://blog.csdn.net/lizhitao/article/details/23744675)

[apache kafka技术分享系列](https://blog.csdn.net/lizhitao/article/details/39499283)

[***KafkaApis-如何处理-Fetch-请求](http://matt33.com/2018/04/15/kafka-server-handle-fetch-request/#KafkaApis-如何处理-Fetch-请求)