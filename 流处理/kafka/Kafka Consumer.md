# 提纲
[toc]

# [Chapter 4. Kafka Consumers: Reading Data from Kafka](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch04.html)
## Kafka Consumer相关的概念
### consumer和consumer group
如果率属于同一个consumer group的多个consumers订阅了同一个topic，则该consumer group中的每一个consumer都将接收关于该topic的所有分区中的一个子集的数据,。

举个栗子，假设topic T1有4个partition，consumer group G1中只有1个consumer C1，则C1将从T1的4个partitions中获取数据，如下：
![image](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in01.png)

如果向consumer group G1中添加另1个consumer C2，则每个consumer都将只会从T1的2个partitions中获取数据，如下：
![image](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in02.png)

如果向consmer group G1中再添加另外2个consumer C3和C4，共4个consumers，则每一个consumer刚好从1个partition中获取数据：
![image](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in03.png)

如果继续向consumer group G1中添加更多的consumer，使得consumers的数目超过T1中partitions的数目，则某些consumers将会处于空闲状态，不会获取到任何消息，如下：
![image](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in04.png)

kafka可以很好的支持大量的consumer group，而不会降低性能（Unlike many traditional messaging systems, Kafka scales to a large number of consumers and consumer groups without reducing performance.）。

接着上面的例子，如果再添加1个新的consumer group G2，则G2会独立于G1，也就是说kafka中的不同consumer group之间是相互独立的，如下：
![image](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in05.png)

### Partition负载均衡
如果向consumer group中添加consumer的时候，新添加的consumer将会从某些之前被其它consumer消费的partition中消费数据，如果从consumer group中移除某些consumer的时候，这些consumer消费的那些partition将会被其它尚存的consumer消费。

如果topic被修改（比如在topic中增加了更多的partitions），则关于该topic的partition也会重新分配给不同的consumers。

将partition的拥有者从1个consumer移动给另1个consumer的过程被称为rebalance，当一个partition从1个consumer移动给另1个consumer的时候，当前的consumer关于该partition的状态将会丢失。

同一个consumer group中的不同consumers通过向该consumer group的group coordinator（是一个kafka broker，不同的consumer group的group coordinator可能不同）发送心跳信息来维持各成员之间的关系以及各partition归哪个consumer所属。当consumer调用poll获取消息的时候，或者提交消息消费确认请求的时候会发送心跳信息。

如果consumer以正常的时间间隔发送心跳信息，则该consumer被认为是存活的，该consumer可以从相应的partition中消费数据。如果group coordinator在较长时间没有接收到某个consumer的心跳信息，则group coordinator将会视该consumer为dead状态，从而触发rebalance。如果正常关闭一个consumer，则该consumer会通知group coordinator它即将离开，group coordinator将会立即出发rebalance。

### Partition是如何分配给各consumer的
当一个consumer想要加入某个consumer group的时候，它会发送一个JoinGroup请求给相应的group coordinator。第一个加入该group的consumer，则会成为group leader（注意区别于group coordinator）。Group leader从group coordinator中获取率属于该group的所有的consumer列表（存活的consumers），并且该group leader负责将相应topic的所有partition分配给每一个consumer。

在Kafka中，采用PartitionAssignor来决定哪个partition被分配给哪个consumer。一旦确定了partition的分配情况之后，group leader会将分配表发送给group coordinator，由group coordinator负责将分配信息发送给各consumer，每一个consumer只会看到它自己从哪些partition消费数据，只有group leader（它也是一个consumer）会看到全局的分配情况。


## 创建kafka consumer
1. 创建一个Java Properties实例，并创建KafkaConsumer实例
    
通过为Properties实例指定不同的属性来配置Kafka Consumer。有3个必须的属性：bootstrap.servers, key.deserializer, 和value.deserializer，其中：
    
bootstrap.servers: 指定kafka broker列表，包括每个broker的hostname和port，形式类似于hostname1:port1, hostname2:port2,...

key.deserializer: key 反序列化用到的类

value.deserializer: value 反序列化用到的类

另外有一些非必须的属性，比如group.id：

group.id：指定KafkaConsumer实例所属的consumer group

下面的代码展示了如何创建KafkaConsumer：
```
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "CountryCounter");
props.put("key.deserializer",
    "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer",
    "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer =
    new KafkaConsumer<String, String>(props);
```

2. 订阅Topics
通过指定topic列表，subscribe()方法可以订阅一个或者多个topics：
```
consumer.subscribe(Collections.singletonList("customerCountries")); 
```

调用subscribe的时候，还可以指定一个正则表达式，该表达式可以匹配多个topic。


3. Poll轮询
在poll轮询中处理分区负载均衡、心跳和获取数据等，返回给用户的就是关于某些区的数据。在第一次调用poll的时候，它会负责找到当前consumer所在的group coordinator，并将当前的consumer添加到consumer group中，然后consumer group leader会为该consumer分配partition（即consumer从这些partition消费数据），如果触发了负载均衡，则也是在poll内部处理的，在poll内部还会发送心跳信息给group coordinator，以保持当前consumer的存活状态，正是因为在poll中发送心跳信息，而2次心跳之间的时间间隔是有限的，所以consumer在处理poll到的数据的时候必须快而高效。

下面展示了poll的用法以及对poll返回的数据的处理：
```
try {
    while (true) { 
        # poll(100)中的参数100表示poll的超时时间，用于控制当consumer 
        # buffer中没有可用的数据的情况下，poll轮询阻塞多长时间才返回
        ConsumerRecords<String, String> records = consumer.poll(100);
        
        # poll()方法会返回一系列记录，每一个记录中都包含topic，partition
        # 和该record在partition中的offset，以及对应的key和value
        for (ConsumerRecord<String, String> record : records)
        {
            log.debug("topic = %s, partition = %d, offset = %d,"
                customer = %s, country = %s\n",
                record.topic(), record.partition(), record.offset(),
                record.key(), record.value());

            int updatedCount = 1;
            if (custCountryMap.countainsValue(record.value())) {
                updatedCount = custCountryMap.get(record.value()) + 1;
            }
            custCountryMap.put(record.value(), updatedCount);

            JSONObject json = new JSONObject(custCountryMap);
            System.out.println(json.toString(4));
        }
    }
} finally {
    consumer.close();
}
```

## 配置Consumer
kafka consumer支持多个配置属性，除了前面提到的bootstrap.servers, key.deserializer, value.deserializer, group.id等以外，还有如下对consumer性能和可用性具有影响的属性：

**fetch.min.bytes**
指定consumer从broker获取数据的时候，最少返回的数据量。broker直到有这么多数据的情况下，才会返回给consumer响应。

**fetch.max.wait.ms**
指定consumer最长等待多长时间，超过该时间，broker就必须响应consumer，哪怕是没有达到{fetch.min.bytes}的数据量。

**max.partition.fetch.bytes**
控制从每一个partition返回给consumer的最大数据量，默认是1MB。假设1个topic具有20个partition，同一个consumer group中一共有5个consumers，那么每个consumer将必须至少有4MB的空间来存放poll()返回的数据。在实践中，必须为每个consumer分配更多的空间来存放poll()返回的数据，以防该consumer需要处理更多的partition的数据（比如该consumer group中的某些consumer失败退出了）。

关于max.partition.fetch.bytes的设置需要额外考虑的是，consumer处理max.partition.fetch.bytes这么多的数据所需要的时间，因为在consumer在poll()中向group coordinator发送心跳信息，如果一次poll()返回大量的数据，从而花费较长的时间去处理这些数据，那么可能会发生consumer timeout的情况。

**session.timeout.ms**
在kafka中，如果一个consumer和group coordinator失联时间在3s以内，group coordinator依然认为该consumer是存活的，但是如果该consumer在超过{session.timeout.ms}的时间内都没有向group coordinator发送心跳信息，则group coordinator将会认为该consumer已经dead，group coordinator就会触发rebalance操作，将之前率属于该dead的consumer的partitions重新分配给其它尚存的consumer。

{session.timeout.ms}通常和{heartbeat.interval.ms}一同修改，{heartbeat.interval.ms}表示poll()每隔多久就要向group coordinator发送一个心跳信息，因此{heartbeat.interval.ms}必须小于{session.timeout.ms}，而且通常{heartbeat.interval.ms}都被设置为{session.timeout.ms}的1/3。

如果{session.timeout.ms}被设置为较小的值，则group coordinator可以更早的发现consumer失败的情况，但是也可能引起不必要的rebalance操作；如果{session.timeout.ms}被设置为较大的值，则会减少不必要的rebalance的发生，但是可能花费较长的时间才能检测到consumer失败的情况。

** auto.offset.reset**
控制当consumer读取一个尚无committed offset记录（表示consumer下一个即将消费的数据在partition中的偏移）的partition，或者consumer读取一个committed offset记录为非法值（通常是由于consumer down掉较长的时间，之前的committed offset对应的记录已经过期了）的partition的时候的行为。

默认值是“latest”，表示在无有效的committed offset的情况下，consumer将会从最新的记录（consumer开始运行之后才写入的数据）开始读取。可选的另外一个值是“earliest”，表示在无有效的committed offset的情况下，consumer将会从头开始读取数据。

**enable.auto.commit**
控制consumer是否自动提交committed offset，默认为true。如果被设置为false，则需要用户自己控制何时提交committed offset

如果{enable.auto.commit}被设置为true，则同时需要设置{auto.commit.interval.ms}表示提交的时间间隔。

**partition.assignment.strategy**
用于控制哪些partitions被分配给哪些consumers。默认的，kafka提供了2中分配策略：

Range

基于范围的分配，每个Topic中的partitions都被单独的分配，同一个Topic的1个或者多个连续的分区被分配给同一个consumer，假设有2个Topic，分别为T1和T2，Topic T1具有3个partitions，分别为T1P1，T1P2和T1P3，Topic T2具有5个partitions，分别为T2P1，T2P2，T2P3，T2P4和T2P5，Consumer Group G1中有2个consumer，分别为C1和C2，这2个consumer都订阅了T1和T2这2个Topic，则分配给C1的分区集合为{T1P1，T1P2，T2P1，T2P2，T2P3}，分配给C2的分区集合{T1P3，T2P4，T2P5}。

RoundRobin

将所有的Topics中的所有的partitions依次分配给各consumers，假设有2个Topic，分别为T1和T2，Topic T1具有3个partitions，分别为T1P1，T1P2和T1P3，Topic T2具有5个partitions，分别为T2P1，T2P2，T2P3，T2P4和T2P5，Consumer Group G1中有2个consumer，分别为C1和C2，这2个consumer都订阅了T1和T2这2个Topic，则分配给C1的分区集合为{T1P1，T1P3，T2P2，T2P4}，分配给C2的分区集合{T1P2，T2P1，T2P3，T2P5}。

默认的采用的是org.apache.kafka.clients.consumer.RangeAssignor，它实现了Range strategy，可以使用org.apache.kafka.clients.consumer.RoundRobinAssignor来替换之，它则实现了RoundRobin strategy，当然，用户还可以自定义自己的assignment strategy。

**client.id**
用于brokers区分是从哪个client发送过来的消息，主要用于日志和统计信息。

**max.poll.records**
控制单次调用poll()中至多返回的记录数。

**receive.buffer.bytes和send.buffer.bytes**
控制读写数据的时候TCP receive buffer和TCP send buffer大小，如果设置为-1，则采用OS中默认大小。

## Commits和offsets
在kafka中，将更新consumer消费数据offset（每个partition中已经消费了哪些数据，或者说接下来要消费的数据从哪开始）的过程称为commit。

当一个Consumer Group初始化后，每个consumer开始从partition顺序读取，consumer会定期commit offset。如下图中，当前consumer读取到offset 6处并且最后一个commit是在offset 1处，如果此时该consumer 挂了，group coordinator会分配一个新的consumer从offset 1开始读取，我们可以发现，新接管的consumer 会再一次重复读取offset 1~offset 6的message：
![image](https://cdn2.hubspot.net/hubfs/540072/New_Consumer_Figure_2.png)

另外，上图的High Watermark代表partition当前最后一个成功拷贝到所有replica的offset，在consumer视角中，最多只能读取到High Watermark所在的offset，即上图中的offset 10中，即使后面还是offset11~14，由于他们尚未完成全部replica，所以暂时无法读取，这种机制是为了防止consumer读取到unreplicated message，因为这些message之后可能丢失。

### consumer是如何commit partition offset的
consumer产生（produce）一个消息，其中包含它所消费的关于每个partition的offset信息，发送给一个特殊的名为__consumer_offsets的topic，就完成了一次commit partition offset过程。

如果某个consumer down掉或者新加入了consumer，都会触发rebalance，导致每个consumer被分配新的partition集合，consumer为了知道从partition的哪里开始消费，需要读取每个partition的committed offset。

如果__consumer_offsets中记录的committed offset（记为last_committed_offset）小于consumer最后处理的消息中记录的offset（记为last_processed_offset），则介于last_committed_offset和last_processed_offset之间的消息将被消费2次，如下：
![image](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in06.png)

如果last_committed_offset大于last_processed_offset，则介于last_processed_offset和last_committed_offset之间的消息将不被消费，如下：
![image](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/assets/ktdg_04in07.png)

因此，如何管理commit offset非常重要，KafkaConsumer API中提供了多种commit offsets的方法：
- Automatic Commit
- Commit Current Offset
- Asynchronous Commit
- Combining Synchronous and Asynchronous Commits
- Commit Specified Offset

#### Automatic Commit
由consumer自动提交，而无需应用介入，可以通过设置{enable.auto.commit = true}来设置，通过{auto.commit.interval.ms}来控制commit的频率，基本实现就是：在consumer每次调用poll()的时候，都会检查是否距上次commit过了{auto.commit.interval.ms}的时间，如果是，则将上一次poll()返回的关于各partition的最大offset给commit了。除了poll()以外，close()也会自动commit offset。

从automatic commit的实现可知，它总是将上一次poll()返回的关于各partition的最大offset给commit了，那么应用必须确保在2次调用poll()之间，将上次poll()返回的所有的记录都处理完。

虽然automatic commit使用起来很方便，但是用户对commit offset缺乏有效的控制。

#### Commit Current Offset
通过设置{enable.auto.commit=false}，commit offset可以由应用程序显示的控制。

最简单可靠的commit API是commitSync()，它会将commit最近的poll()返回的最新的offset，直到offset被成功提交，或者抛出异常。因为commitSync()会将最近的poll()返回的最新的offset提交，因此必须确保在调用commitSync()之前，应用已经处理完最近的poll()返回的所有的消息。

下面的例子展示了如何使用commitSync():
```
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %s, offset =
            %d, customer = %s, country = %s\n",
            record.topic(), record.partition(),
            record.offset(), record.key(), record.value());
    }
    try {
        # commit此前poll()的offset，应用会阻塞，直到commit成功，或者抛出异常
        # 只要没发生不可恢复的错误，commitSync()将会不断重试
        consumer.commitSync();
    } catch (CommitFailedException e) {
        log.error("commit failed", e)
    }
}
```

#### Asynchronous Commit
commitSync()的缺点是应用程序必须等待，直到broker响应了该commit请求。因此Kafka又提供了异步的commit API，即commitAsync()。

下面的例子展示了如何使用commitSync():
```
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %s,
            offset = %d, customer = %s, country = %s\n",
            record.topic(), record.partition(), record.offset(),
            record.key(), record.value());
    }
    
    # commit此前poll()返回的最新的offset，但不会等待broker响应本次的commit请求
    consumer.commitAsync();
}
```

commitAsync()还可以提供一个callback，供broker响应了commit请求之后调用，下面的例子展示了该如何使用之：
```
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %s,
        offset = %d, customer = %s, country = %s\n",
        record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
    }
    consumer.commitAsync(new OffsetCommitCallback() {
        public void onComplete(Map<TopicPartition,
        OffsetAndMetadata> offsets, Exception e) {
            if (e != null)
                log.error("Commit failed for offsets {}", offsets, e);
        }
    }); 1
}
```

#### Combining Synchronous and Asynchronous Commits
一个通常的用法是，在正常运行过程中使用commitAsync()，但在即将关闭consumer之前使用commitSync()，下面展示了这种用法：
```
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s, partition = %s, offset = %d,
                customer = %s, country = %s\n",
                record.topic(), record.partition(),
                record.offset(), record.key(), record.value());
        }
        # 正常运行过程中，直接使用commitAsync()，即使失败也没多大关系，
        # 因为后续的成功的commitAsync()或者commitSync()总是会commit正确
        # 的offset
        consumer.commitAsync();
    }
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try {
        # 在即将关闭consumer之前，调用commitSync()，它将在关闭consumer之前
        # commit成功，除非发生了不可恢复的错误
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```

#### Commit Specified Offset
前面讲到的commitSync()或者commitAsync()总是commit最近的poll()返回的最新的offset，但是如果想commit某个特定的offset呢？Kafka的commitSync()和commitAsync()接口也是支持的，如下：
```
# 用于跟踪各partition的offsets
private Map<TopicPartition, OffsetAndMetadata> currentOffsets =
    new HashMap<>();
int count = 0;

....

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %s, offset = %d,
            customer = %s, country = %s\n",
            record.topic(), record.partition(), record.offset(),
            record.key(), record.value());
        # 更新@currentOffsets，即更新自身维护的各partition的offset
        currentOffsets.put(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset()+1, "no metadata"));
        # 每当处理完1000条记录，就异步的commit一次
        if (count % 1000 == 0)
            consumer.commitAsync(currentOffsets, null);
        count++;
    }
}
```

## Rebalance Listeners
因为partition rebalance，一个consumer可能会失去某些partition的所属权，那么如果consumer想在失去partition的所属权之前做一些工作该怎么办？Kafka consumer提供了相关的API，允许应用程序在添加或者删除partition的时候运行应用自身的代码，这个API就是在调用subscribe()的时候，传递一个ConsumerRebalanceListener。

ConsumerRebalanceListener中定义了2个接口，用户可以实现这2个接口：
```
# 在consumer停止从这些partitions中消费数据到启动rebalance之前调用
public void onPartitionsRevoked(Collection<TopicPartition> partitions)

# 在partitions被重新分配之后，且在consumer从这些partitions消费数据之前调用
public void onPartitionsAssigned(Collection<TopicPartition> partitions)
```

下面的例子中展示了在consumer失去partitions的所属权之前如何调用onPartitionsRevoked来commit offsets：
```
# 用于管理partition和offset的映射关系
private Map<TopicPartition, OffsetAndMetadata> currentOffsets =
    new HashMap<>();

# HandleRebalance继承了ConsumerRebalanceListener接口
private class HandleRebalance implements ConsumerRebalanceListener {
    # 实现onPartitionsAssigned接口
    public void onPartitionsAssigned(Collection<TopicPartition>
        partitions) {
    }

    # 实现onPartitionsRevoked接口
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Lost partitions in rebalance. " +
            "Committing current offsets:" + currentOffsets);
        # 在失去对partitions的所属权之前，commit当前consumer所看到的offsets
        consumer.commitSync(currentOffsets);
    }
}

try {
    # 订阅@topics中的所有topic，并指定rebalanceListener
    consumer.subscribe(topics, new HandleRebalance());

    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s, partition = %s, offset = %d,
                 customer = %s, country = %s\n",
                 record.topic(), record.partition(), record.offset(),
                 record.key(), record.value());
             currentOffsets.put(
                 new TopicPartition(record.topic(), record.partition()),
                 new OffsetAndMetadata(record.offset()+1, "no metadata"));
        }
        consumer.commitAsync(currentOffsets, null);
    }
} catch (WakeupException e) {
    // ignore, we're closing
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try {
        consumer.commitSync(currentOffsets);
    } finally {
        consumer.close();
        System.out.println("Closed consumer and we are done");
    }
}
```

## 从指定的offsets开始消费
如果希望从partition的头部开始消费消息，或者只想从partition的尾部来消费新的消息，Kafka提供了相关的API：
```
seekToBeginning(Collection<TopicPartition> tp)

seekToEnd(Collection<TopicPartition> tp).
```

其实Kafka也提供了API供应用从任意offset开始消费消息，那就是seek()，举例如下（在该示例中，处理一条记录都会将该记录存放在数据库中，同时会将partition offset信息保存在database中，并且为了确保partition offset能正确反映已经存放的记录，将存放记录和保存partition offset作为一个事务来处理，在consumer第一次启动或者发生partition Rebalance之后，consumer都是先从database中获取partition offset，然后seek到该offset，从该offset开始消费）：
```
public class SaveOffsetsOnRebalance implements ConsumerRebalanceListener {
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        # 在失去对partition的所属权之前，确保记录和partition offset被事务性提交
        commitDBTransaction();
    }

    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        for(TopicPartition partition: partitions)
            # 从database中获取partition对应的offset，并seek到该offset
            consumer.seek(partition, getOffsetFromDB(partition));
    }
}

# 订阅@topics中的所有的topic，并注册RebalanceListener
consumer.subscribe(topics, new SaveOffsetOnRebalance(consumer));
consumer.poll(0);

# 对于每一个partition都是先从database中获取partition offset，
# 然后seek到该offset，从该offset开始消费
for (TopicPartition partition: consumer.assignment())
    consumer.seek(partition, getOffsetFromDB(partition));

while (true) {
    # 轮询
    ConsumerRecords<String, String> records =
        consumer.poll(100);
    # 逐一处理poll()中返回的结果
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
        # 存储record到database中
        storeRecordInDB(record);
        # 存储partition offset到database中
        storeOffsetInDB(record.topic(), record.partition(),
            record.offset());
    }
    
    # 事务性提交（确保record和partition offset原子性提交，以保证partition
    # offset能正确反映已经存储到database中的记录）
    commitDBTransaction();
}
```

## 如何退出poll()轮询
有时候可能希望退出poll()轮询，那么就需要在另一个线程中调用consumer.wakeup()了，如果是在主线程中运行poll()轮询的话，则可以直接通过ShutdownHook来启动一个新的线程并调用consumer.wakeup()。

consumer.wakeup()是唯一的可以在另一个线程中调用的方法，它会使得poll()抛出WakeupException，然后退出。在退出线程之前，必须调用consumer.close()，在consumer.close()中consumer会根据需要执行commit offsets操作，同时发送给group coordinator一条退出consumer group的消息。

```
    # 获取主线程
    final Thread mainThread = Thread.currentThread();

    # 注册一个shutdown hook，shutdown hook必须在一个单独的线程中运行
    Runtime.getRuntime().addShutdownHook(new Thread() {
        public void run() {
            System.out.println("Starting exit...");
            # 调用wakeup()来退出主线程中的poll()轮询
            consumer.wakeup();
            try {
                # 等待主线程退出
                mainThread.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    });
    
    try {
        // looping until ctrl-c, the shutdown hook will cleanup on exit
        while (true) {
            # poll()轮询
            ConsumerRecords<String, String> records =
                movingAvg.consumer.poll(1000);
            System.out.println(System.currentTimeMillis() +
                "--  waiting for data...");
            for (ConsumerRecord<String, String> record : records) {
                System.out.printf("offset = %d, key = %s, value = %s\n",
                    record.offset(), record.key(), record.value());
            }
            for (TopicPartition tp: consumer.assignment())
                System.out.println("Committing offset at position:" +
                    consumer.position(tp));
                consumer.commitSync();
        }
    } catch (WakeupException e) {
        // poll()轮询被wakeup()唤醒，会抛出WakeupException，可以直接忽略之
    } finally {
        # “干净的”退出consumer
        consumer.close();
        System.out.println("Closed consumer and we are done");
    }

```

## 反序列化器（deserializer）
在Kafka中producer需要序列化器，将发送给Kafka的对象转换为字节数组，同样的consumer需要反序列化器，将从Kafka接收到的字节数组转换为对象。

在Kafka的代码中提供了ByteArrayDeserializer，ByteBufferDeserializer，BytesDeserializer，DoubleDeserializer，IntegerDserializer，LongDeserializer和StringDeserializer等。

这里主要讲解如何创建自定义的反序列化器，以及如何使用Avro(Avro是一个数据序列化系统，设计用于支持大批量数据交换的应用)中的反序列化器。

### 自定义反序列化器
首先定义一个自定义类Customer：
```
public class Customer {
    private int customerID;
    private String customerName;

    public Customer(int ID, String name) {
        this.customerID = ID;
        this.customerName = name;
    }

    public int getID() {
        return customerID;
    }

    public String getName() {
        return customerName;
    }
}
```

其次，为Customer类定义反序列化器，实现Deserializer类中的相关接口：
```
import org.apache.kafka.common.errors.SerializationException;

import java.nio.ByteBuffer;
import java.util.Map;

public class CustomerDeserializer implements Deserializer<Customer> {

    @Override
    public void configure(Map configs, boolean isKey) {
        // nothing to configure
    }

    # 实现deserialize方法，从字节数组中反序列化出Java对象
    @Override
    public Customer deserialize(String topic, byte[] data) {
        int id;
        int nameSize;
        String name;

        try {
            if (data == null)
                return null;
            if (data.length < 8)
                throw new SerializationException("Size of data received " +
                    "by IntegerDeserializer is shorter than expected");

            ByteBuffer buffer = ByteBuffer.wrap(data);
            id = buffer.getInt();
            nameSize = buffer.getInt();

            byte[] nameBytes = new byte[nameSize];
            buffer.get(nameBytes);
            name = new String(nameBytes, "UTF-8");

            return new Customer(id, name);

        } catch (Exception e) {
  	        throw new SerializationException("Error when serializing " +
  	            "Customer to byte[] " + e);
        }
    }

    @Override
    public void close() {
        // nothing to close
    }
}
```

相应的，在Consumer中使用Customer类和Customer反序列化器如下：
```
Properties props = new Properties();
props.put("key.deserializer",
    "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer",
    CustomerDeserializer.class.getName());
    
KafkaConsumer<String, Customer> consumer =
    new KafkaConsumer<>(props);
    
......
```

### Avro反序列化器
假设还是使用前面定义的Customer类，使用KafkaAvroDeserializer来反序列化Avro消息：
```
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "CountryCounter");
props.put("key.deserializer",
    "org.apache.kafka.common.serialization.StringDeserializer");
# 使用KafkaAvroDeserializer来反序列化Avro消息
props.put("value.deserializer",
    "io.confluent.kafka.serializers.KafkaAvroDeserializer");
# 指向schema存放的位置，这样consumer就可以使用producer注册的schema来反序列化消息
props.put("schema.registry.url", schemaUrl);
String topic = "customerContacts"

KafkaConsumer<String, Customer> consumer =
    new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList(topic));

System.out.println("Reading topic:" + topic);

while (true) {
    ConsumerRecords<String, Customer> records = consumer.poll(1000); 3

    for (ConsumerRecord<String, Customer> record: records) {
        System.out.println("Current customer name is: " +
            record.value().getName());
    }
    consumer.commitSync();
}
```

## Consumer消费指定的Partitions
```
List<PartitionInfo> partitionInfos = null;
# 获取关于指定topic的partition列表，如果不想从该topic的所有
# 的partitions中消费，可以不执行该方法
partitionInfos = consumer.partitionsFor("topic");

if (partitionInfos != null) {
    for (PartitionInfo partition : partitionInfos)
        partitions.add(new TopicPartition(partition.topic(),
            partition.partition()));
    # consumer从指定的partitions集合中消费，这个partitions集合
    # 可能是某个topic的所有partitions集合的子集
    consumer.assign(partitions);

    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(1000);

        for (ConsumerRecord<String, String> record: records) {
            System.out.printf("topic = %s, partition = %s, offset = %d,
                customer = %s, country = %s\n",
                record.topic(), record.partition(), record.offset(),
                record.key(), record.value());
        }
        consumer.commitSync();
    }
}
```

要注意的是，如果通过consumer.assign()来消费关于某个topic的partitions，该consumer不会被通知任何可能的新添加的partition，需要consumer子集主动调用consumer.partitionsFor()来主动获取之。

## 多个consumer的例子
首先定义单个Consumer需要执行的工作：
```
# ConsumerLoop集成Runnable接口，因此可以被调用
public class ConsumerLoop implements Runnable {
  private final KafkaConsumer<String, String> consumer;
  private final List<String> topics;
  private final int id;

  public ConsumerLoop(int id,
                      String groupId, 
                      List<String> topics) {
    this.id = id;
    this.topics = topics;
    
    # 配置Consumer
    Properties props = new Properties();
    props.put("bootstrap.servers", "localhost:9092");
    props.put(“group.id”, groupId);
    props.put(“key.deserializer”, StringDeserializer.class.getName());
    props.put(“value.deserializer”, StringDeserializer.class.getName());
    this.consumer = new KafkaConsumer<>(props);
  }
 
  @Override
  public void run() {
    try {
      # 订阅topics
      consumer.subscribe(topics);

      # 不断的调用poll()从@topics中消费
      while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Long.MAX_VALUE);
        for (ConsumerRecord<String, String> record : records) {
          Map<String, Object> data = new HashMap<>();
          data.put("partition", record.partition());
          data.put("offset", record.offset());
          data.put("value", record.value());
          System.out.println(this.id + ": " + data);
        }
      }
    } catch (WakeupException e) {
      # 应用主动中断了poll()的情况下，会抛出WakeupException
    } finally {
      consumer.close();
    }
  }

  # 关闭consumer之前，首先中断poll()
  public void shutdown() {
    consumer.wakeup();
  }
}
```

主线程中启动3个consumer，组成1个consumer group，共同消费名为“consumer-tutorial”的topic：
```
public static void main(String[] args) { 
  int numConsumers = 3;
  String groupId = "consumer-tutorial-group"
  List<String> topics = Arrays.asList("consumer-tutorial");
  # 创建一个固定数目线程的线程池
  ExecutorService executor = Executors.newFixedThreadPool(numConsumers);

  final List<ConsumerLoop> consumers = new ArrayList<>();
  for (int i = 0; i < numConsumers; i++) {
    # 创建一个新的consumer，添加到@consumers集合中，并提交该executor处理
    ConsumerLoop consumer = new ConsumerLoop(i, groupId, topics);
    consumers.add(consumer);
    # 将当前的consumer作为一个任务提交给线程池
    executor.submit(consumer);
  }

  Runtime.getRuntime().addShutdownHook(new Thread() {
    @Override
    public void run() {
      # 依次中断各consumer的poll()轮询
      for (ConsumerLoop consumer : consumers) {
        consumer.shutdown();
      } 
      
      # 关闭executor
      executor.shutdown();
      try {
        executor.awaitTermination(5000, TimeUnit.MILLISECONDS);
      } catch (InterruptedException e) {
        e.printStackTrace;
      }
    }
  });
}
```

## Delivery Semantics
当Consumer Group创建后，每个consumer从partition的哪个offset开始读取是由auto.offset.reset控制的。如果consumer在commit offset之前crash了，那么下一个接手的consumer会重复读取一部分消息。

commit默认是自动的{enable.auto.commit=true}，consumer周期性的{auto.commit.interval.ms} commit offset，如果希望手动commit，首先得取消自动commit，设置enable.auto.commit为false。

使用commit api的关键点在于如何结合poll()，它决定了Delivery Semantics。

### at least once
当commit policy保证last commit offset一定在当前处理的消息的offset之前：
![image](https://cdn2.hubspot.net/hubfs/540072/New_Consumer_Figure_2.png)

可以发现，Automatic Commit时，就是使用的at least once，因为Automatic Commit的基本实现就是：在consumer每次调用poll()的时候，都会检查是否距上次commit过了{auto.commit.interval.ms}的时间，如果是，则将**上一次poll()**返回的关于各partition的最大offset给commit了。


### at most once
当commit policy保证last commit offset一定在当前offset之后：
![image](https://cdn2.hubspot.net/hubfs/540072/consumer-atmostonce-1.png)

使用at most once交付语义时，如果consumer尚未完成当前poll loop的所有message前crash了，则会丢失未处理的那部分数据，因为新接管的consumer不会感知到这部分message。

## Consumer Group Inspection
当激活一个consumer group后，可以在命令行查看该consumer group中各consumer的进度：
```
# bin/kafka-consumer-groups.sh –new-consumer –describe –group consumer-tutorial-group –bootstrap-server localhost:9092
```
返回类似如下：

```
GROUP, TOPIC, PARTITION, CURRENT OFFSET, LOG END OFFSET, LAG, OWNER
consumer-tutorial-group, consumer-tutorial, 0, 6667, 6667, 0, consumer-1_/127.0.0.1
consumer-tutorial-group, consumer-tutorial, 1, 6667, 6667, 0, consumer-2_/127.0.0.1
consumer-tutorial-group, consumer-tutorial, 2, 6666, 6666, 0, consumer-3_/127.0.0.1
```
管理员可以通过该功能检查consumer group是否跟上producers。

## 关于KafkaConsumer的说明
本文中谈到的Java KafkaConsumer来自于org.apache.kafka.clients包，实际上早较早的版本的Kafka中还有2个用scala编写的Consumers，分别是SimpleConsumer和ZookeeperConsumerConnector，ZookeeperConsumerConnector也被称为high-level consumer，它和本文中讲到的KafkaConsumer类似，但是它使用Zookeeper来管理consumer groups并且不提供本文中所谈到的关于commit offset的自由控制（比如Commit Specified Offset），也不提供rebalance相关的控制（比如添加Rebalance Listener）。

Kafka开发之初，它只提供了采用Scala语言编写的producer和consumer client，随着时间的推移，渐渐发现这些API具有许多的局限性，因此决定重新设计Producer和Consumer clients。在0.8.1版本中重写了Producer API，在0.9版本中则重写了Consumer API。新版本的Consumer API带来了如下优势：
- 整合旧版本中的“SimpleConsumer”和“High-level Consumer”的优势；
- 纯Java编写，不再依赖于Scala运行时环境；
- 不再依赖于Zookeeper，老版本则基于Zookeeper来实现group coordination protocol；
- 更安全：Kafka0.9提供security extensions；
- 添加一系列协议来更好的管理fault-tolerant groups of consumer processes；

虽然new consumer重构API并且使用新的coordination protocol，但是概念并没有根本改变，所以熟悉old consumer的用户不会难以理解new consumer。然而，需要额外关心下group management 和threading model。

## [一个consumer group中多个consumer的例子](https://www.confluent.io/blog/tutorial-getting-started-with-the-new-apache-kafka-0-9-consumer-client/)


# 参考
[Chapter 4. Kafka Consumers: Reading Data from Kafka](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/ch04.html)

[Introducing the Kafka Consumer: Getting Started with the New Apache Kafka 0.9 Consumer Client](https://www.confluent.io/blog/tutorial-getting-started-with-the-new-apache-kafka-0-9-consumer-client/)

[Kafka New Consumer API](https://blog.csdn.net/opensure/article/details/72419701)

