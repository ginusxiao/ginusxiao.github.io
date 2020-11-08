# Outline
[toc]

# Streams Concepts
如果不想看下面的英文，则可以直接跳到[**Further more readings**]。

## Stream
A stream is the most important abstraction provided by Kafka Streams: it represents an unbounded, continuously updating data set, where unbounded means “of unknown or of unlimited size”. Just like a topic in Kafka, a stream in the Kafka Streams API consists of one or more stream partitions.

A stream partition is an, ordered, replayable, and fault-tolerant sequence of immutable data records, where a data record is defined as a key-value pair.

## Stream Processing Application
A stream processing application is any program that makes use of the Kafka Streams library. It may define its computational logic through one or more processor topologies.

Your stream processing application doesn't run inside a broker. Instead, it runs in a separate JVM instance, or in a separate cluster entirely.

![image](https://docs.confluent.io/current/_images/streams-apps-not-running-in-brokers.png)

An application instance is any running instance or “copy” of your application. Application instances are the primary means to elasticly scale and parallelize your application, and they also contribute to making it fault-tolerant. You could opt to run multiple instances of your application, and these instances would automatically collaborate on the data processing – even as new instances/machines are added or existing ones removed during live operation.

![image](https://docs.confluent.io/current/_images/scale-out-streams-app.png)

## Processor Topology
A processor topology or simply topology defines the computational logic of the data processing that needs to be performed by a stream processing application. A topology is a graph of stream processors (nodes) that are connected by streams (edges). Developers can define topologies either via the low-level Processor API or via the Kafka Streams DSL, which builds on top of the former.

![image](https://docs.confluent.io/current/_images/streams-concepts-topology.jpg)


## Stream Processor
A stream processor is a node in the processor topology as shown in the diagram of section Processor Topology. It represents a processing step in a topology, i.e. it is used to transform data. Standard operations such as map or filter, joins, and aggregations are examples of stream processors that are available in Kafka Streams out of the box. A stream processor receives one input record at a time from its upstream processors in the topology, applies its operation to it, and may subsequently produce one or more output records to its downstream processors.

Kafka Streams provides two APIs to define stream processors:

The declarative, functional DSL is the recommended API for most users – because most data processing use cases can be expressed in just a few lines of DSL code.

The imperative, lower-level Processor API provides you with even more flexibility than the DSL but at the expense of requiring more manual coding work.

## Stateful Stream Processing
Some stream processing applications don't require state – they are stateless – which means the processing of a message is independent from the processing of other messages.

In practice, however, most applications require state – they are stateful – in order to work correctly, and this state must be managed in a fault-tolerant manner. Your application is stateful whenever, for example, it needs to join, aggregate, or window its input data. Kafka Streams provides your application with powerful, elastic, highly scalable, and fault-tolerant stateful processing capabilities.

## Duality of Streams and Tables
When implementing stream processing use cases in practice, you typically need both streams and also databases. Any stream processing technology must therefore provide first-class support for streams and tables. An interesting observation is that there is actually a close relationship between streams and tables, the so-called stream-table duality. Essentially, this duality means that a stream can be viewed as a table, and a table can be viewed as a stream. 

The stream-table duality describes the close relationship between streams and tables.

- **Table as Stream**: A table can be considered a snapshot, at a point in time, of the latest value for each key in a stream (a stream’s data records are key-value pairs). A table is thus a stream in disguise, and it can be easily turned into a “real” stream by iterating over each key-value entry in the table.
![image](https://docs.confluent.io/current/_images/streams-table-duality-02.jpg)

- **Stream as Table**: A stream can be considered a changelog of a table, where each data record in the stream captures a state change of the table. A stream is thus a table in disguise, and it can be easily turned into a “real” table by replaying the changelog from beginning to end to reconstruct the table. Similarly, aggregating data records in a stream will return a table.
![image](https://docs.confluent.io/current/_images/streams-table-duality-03.jpg)


## KStream
*Note: Only the Kafka Streams DSL has the notion of a KStream.*
A KStream is an abstraction of a record stream, where each data record represents a self-contained datum in the unbounded data set. Using the table analogy, data records in a record stream are always interpreted as an “INSERT”.

To illustrate, let’s imagine the following two data records are being sent to the stream:
```
("alice", 1) --> ("alice", 3)
```
If your stream processing application were to sum the values per user, it would return 4 for alice. Why? Because the second data record would not be considered an update of the previous record. 

## KTable
*Note: Only the Kafka Streams DSL has the notion of a KTable.*
A KTable is an abstraction of a changelog stream, where each data record represents an update. More precisely, the value in a data record is interpreted as an “UPDATE” of the last value for the same record key, if any (if a corresponding key doesn’t exist yet, the update will be considered an INSERT). Using the table analogy, a data record in a changelog stream is interpreted as an UPSERT aka INSERT/UPDATE because any existing row with the same key is overwritten. Also, null values are interpreted in a special way: a record with a null value represents a “DELETE” or tombstone for the record’s key.

To illustrate, let’s imagine the following two data records are being sent to the stream:
```
("alice", 1) --> ("alice", 3)
```
If your stream processing application were to sum the values per user, it would return 3 for alice. Why? Because the second data record would be considered an update of the previous record. 

KTable also provides an ability to look up current values of data records by keys. This table-lookup functionality is available through join operations (see also Joining in the Developer Guide) as well as through Interactive Queries.


## GlobalKTable
*Note: Only the Kafka Streams DSL has the notion of a GlobalKTable.*
Like a KTable, a GlobalKTable is an abstraction of a changelog stream, where each data record represents an update.

A GlobalKTable differs from a KTable in the data that they are being populated with, i.e. which data from the underlying Kafka topic is being read into the respective table. Slightly simplified, imagine you have an input topic with 5 partitions. In your application, you want to read this topic into a table. Also, you want to run your application across 5 application instances for maximum parallelism.

- If you read the input topic into a KTable, then the “local” KTable instance of each application instance will be populated with data from only 1 partition of the topic’s 5 partitions.
- If you read the input topic into a GlobalKTable, then the local GlobalKTable instance of each application instance will be populated with data from all partitions of the topic.

GlobalKTable provides the ability to look up current values of data records by keys. This table-lookup functionality is available through join operations (see also Joining in the Developer Guide) and Interactive Queries.

Benefits of global tables:
- More convenient and/or efficient joins: Notably, global tables allow you to perform star joins, they support “foreign-key” lookups (i.e., you can lookup data in the table not just by record key, but also by data in the record values), and they are more efficient when chaining multiple joins. Also, when joining against a global table, the input data does not need to be co-partitioned.
- Can be used to “broadcast” information to all the running instances of your application.

Downsides of global tables:
- Increased local storage consumption compared to the (partitioned) KTable because the entire topic is tracked.
- Increased network and Kafka broker load compared to the (partitioned) KTable because the entire topic is read.


## Time
A critical aspect in stream processing is the the notion of time, and how it is modeled and integrated. For example, some operations such as Windowing are defined based on time boundaries.

Kafka Streams supports the following notions of time:
- Event-time: The point in time when an event or data record occurred, i.e. was originally created “by the source”. Achieving event-time semantics typically requires embedding timestamps in the data records at the time a data record is being produced.
- Processing-time: The point in time when the event or data record happens to be processed by the stream processing application, i.e. when the record is being consumed. The processing-time may be milliseconds, hours, or days etc. later than the original event-time.
- Ingestion-time: The point in time when an event or data record is stored in a topic partition by a Kafka broker. Ingestion-time is similar to event-time, as a timestamp gets embedded in the data record itself. The difference is, that the timestamp is generated when the record is appended to the target topic by the Kafka broker, not when the record is created “at the source”. Ingestion-time may approximate event-time reasonably well if we assume that the time difference between creation of the record and its ingestion into Kafka is sufficiently small, where “sufficiently” depends on the specific use case. Thus, ingestion-time may be a reasonable alternative for use cases where event-time semantics are not possible, e.g. because the data producers don’t embed timestamps (e.g. older versions of Kafka’s Java producer client) or the producer cannot assign timestamps directly (e.g., it does not have access to a local clock).


## Aggregations
An aggregation operation takes one input stream or table, and yields a new table by combining multiple input records into a single output record. Examples of aggregations are computing counts or sum.

In the Kafka Streams DSL, an input stream of an aggregation operation can be a KStream or a KTable, but the output stream will always be a KTable. This allows Kafka Streams to update an aggregate value upon the late arrival of further records after the value was produced and emitted. When such late arrival happens, the aggregating KStream or KTable emits a new aggregate value. Because the output is a KTable, the new value is considered to overwrite the old value with the same key in subsequent processing steps.


## Joins
A join operation merges two input streams and/or tables based on the keys of their data records, and yields a new stream/table.

The join operations available in the Kafka Streams DSL differ based on which kinds of streams and tables are being joined; for example, KStream-KStream joins versus KStream-KTable joins.


## Windowing
Windowing lets you control how to group records that have the same key for stateful operations such as aggregations or joins into so-called windows. Windows are tracked per record key.

Windowing operations are available in the Kafka Streams DSL. When working with windows, you can specify a retention period for the window. This retention period controls how long Kafka Streams will wait for out-of-order or late-arriving data records for a given window. If a record arrives after the retention period of a window has passed, the record is discarded and will not be processed in that window.

Late-arriving records are always possible in the real world and should be properly accounted for in your applications. It depends on the effective time semantics how late records are handled. In the case of processing-time, the semantics are “when the record is being processed”, which means that the notion of late records is not applicable as, by definition, no record can be late. Hence, late-arriving records can only be considered as such (i.e. as arriving “late”) for event-time or ingestion-time semantics. In both cases, Kafka Streams is able to properly handle late-arriving records.


## Interactive Queries
Interactive queries allow you to treat the stream processing layer as a lightweight embedded database, and to directly query the latest state of your stream processing application. You can do this without having to first materialize that state to external databases or external storage.

Interactive queries simplify the architecture and lead to more application-centric architectures.

Without interactive queries: increased complexity and heavier footprint of architecture:
![Without interactive queries: increased complexity and heavier footprint of architecture.](https://docs.confluent.io/current/_images/streams-interactive-queries-01.png)

With interactive queries: simplified, more application-centric architecture:
![With interactive queries: simplified, more application-centric architecture.](https://docs.confluent.io/current/_images/streams-interactive-queries-02.png)


## Processing Guarantees
Kafka Streams supports at-least-once and exactly-once processing guarantees.

At-least-once semantics
Records are never lost but may be redelivered. If your stream processing application fails, no data records are lost and fail to be processed, but some data records may be re-read and therefore re-processed. At-least-once semantics is enabled by default (processing.guarantee="at_least_once") in your Streams configuration.
Exactly-once semantics
Records are processed once. Even if a producer sends a duplicate record, it is written to the broker exactly once. Exactly-once stream processing is the ability to execute a read-process-write operation exactly one time. All of the processing happens exactly once, including the processing and the materialized state created by the processing job that is written back to Kafka. To enable exactly-once semantics, set processing.guarantee="exactly_once" in your Streams configuration.


# Stream Architecture
## Processor Topology
A processor topology or simply topology defines the stream processing computational logic for your application, i.e., how input data is transformed into output data. A topology is a graph of stream processors (nodes) that are connected by streams (edges). There are two special processors in the topology:

Source Processor: A source processor is a special type of stream processor that does not have any upstream processors. It produces an input stream to its topology from one or multiple Kafka topics by consuming records from these topics and forward them to its down-stream processors.
Sink Processor: A sink processor is a special type of stream processor that does not have down-stream processors. It sends any received records from its up-stream processors to a specified Kafka topic.

## Parallelism Model
### Stream Partitions and Tasks
The messaging layer of Kafka partitions data for storing and transporting it. Kafka Streams partitions data for processing it. In both cases, this partitioning is what enables data locality, elasticity, scalability, high performance, and fault tolerance.

Kafka Streams uses the concepts of stream partitions and stream tasks as logical units of its parallelism model. There are close links between Kafka Streams and Kafka in the context of parallelism:
- Each stream partition is a totally ordered sequence of data records and maps to a Kafka topic partition.
- A data record in the stream maps to a Kafka message from that topic.
The keys of data records determine the partitioning of data in both Kafka and Kafka Streams, i.e., how data is routed to specific partitions within topics.

**An application’s processor topology is scaled by breaking it into multiple stream tasks.** More specifically, Kafka Streams creates a fixed number of stream tasks based on the input stream partitions for the application, with each task being assigned a list of partitions from the input streams (i.e., Kafka topics). **The assignment of stream partitions to stream tasks never changes**, hence the stream task is a fixed unit of parallelism of the application. Tasks can then instantiate their own processor topology based on the assigned partitions; they also maintain a buffer for each of its assigned partitions and process input data one-record-at-a-time from these record buffers. As a result stream tasks can be processed independently and in parallel without manual intervention.

Slightly simplified, the** maximum parallelism** at which your application may run is bounded by the maximum number of stream tasks, which itself is determined by maximum number of partitions of the input topic(s) the application is reading from. For example, if your input topic has 5 partitions, then you can run up to 5 applications instances. These instances will collaboratively process the topic’s data. If you run a larger number of app instances than partitions of the input topic, the “excess” app instances will launch but remain idle; however, if one of the busy instances goes down, one of the idle instances will resume the former’s work. 

**Sub-topologies aka topology sub-graphs: **If there are multiple processor topologies specified in a Kafka Streams application, each task will only instantiate one of the topologies for processing. In addition, a single processor topology may be decomposed into independent sub-topologies (sub-graphs) as long as sub-topologies are not connected by any streams in the topology; here, each task may instantiate only one such sub-topology for processing. This further scales out the computational workload to multiple tasks.

It is important to understand that Kafka Streams is not a resource manager, but a library that “runs” anywhere its stream processing application runs. Multiple instances of the application are executed either on the same machine, or spread across multiple machines and tasks can be distributed automatically by the library to those running application instances. The assignment of partitions to tasks never changes; if an application instance fails, all its assigned tasks will be restarted on other instances and continue to consume from the same stream partitions.

### Threading Model
Kafka Streams allows the user to configure the number of threads that the library can use to parallelize processing within an application instance. Each thread can execute one or more stream tasks with their processor topologies independently.

![image](https://docs.confluent.io/current/_images/streams-architecture-threads.jpg)

Starting more stream threads or more instances of the application merely amounts to replicating the topology and having it process a different subset of Kafka partitions, effectively parallelizing processing. It is worth noting that there is no shared state amongst the threads, so no inter-thread coordination is necessary. This makes it very simple to run topologies in parallel across the application instances and threads. The assignment of Kafka topic partitions amongst the various stream threads is transparently handled by Kafka Streams leveraging Kafka’s server-side coordination functionality.

As we described above, scaling your stream processing application with Kafka Streams is easy: you merely need to start additional instances of your application, and Kafka Streams takes care of distributing partitions across stream tasks that run in the application instances. You can start as many threads of the application as there are input Kafka topic partitions so that, across all running instances of an application, every thread (or rather, the stream tasks that the thread executes) has at least one input partition to process.

### Example
Imagine a Kafka Streams application that consumes from two topics, A and B, with each having 3 partitions. If we now start the application on a single machine with the number of threads configured to 2, we end up with two stream threads instance1-thread1 and instance1-thread2. Kafka Streams will break this topology by default into three tasks because the maximum number of partitions across the input topics A and B is max(3, 3) == 3, and then distribute the six input topic partitions evenly across these three tasks; in this case, each task will process records from one partition of each input topic, for a total of two input partitions per task. Finally, these three tasks will be spread evenly – to the extent this is possible – across the two available threads, which in this example means that the first thread will run 2 tasks (consuming from 4 partitions) and the second thread will run 1 task (consuming from 2 partitions).

![image](https://docs.confluent.io/current/_images/streams-architecture-example-01.png)

Now imagine we want to scale out this application later on, perhaps because the data volume has increased significantly. We decide to start running the same application but with only a single thread on another, different machine. A new thread instance2-thread1 will be created, and input partitions will be re-assigned similar to:

![image](https://docs.confluent.io/current/_images/streams-architecture-example-02.png)


## State
Kafka Streams provides so-called state stores, which can be used by stream processing applications to store and query data, which is an important capability when implementing stateful operations. The Kafka Streams DSL, for example, automatically creates and manages such state stores when you are calling stateful operators such as count() or aggregate(), or when you are windowing a stream.

Every stream task in a Kafka Streams application may embed one or more local state stores that can be accessed via APIs to store and query data required for processing. These state stores can either be a RocksDB database, an in-memory hash map, or another convenient data structure. Kafka Streams offers fault-tolerance and automatic recovery for local state stores.

![image](https://docs.confluent.io/current/_images/streams-architecture-states.jpgo)

Two stream tasks with their dedicated local state stores

A Kafka Streams application is typically running on many application instances. Because Kafka Streams partitions the data for processing it, an application’s entire state is spread across the local state stores of the application’s running instances. The Kafka Streams API lets you work with an application’s state stores both locally (e.g., on the level of an instance of the application) as well as in its entirety (on the level of the “logical” application), for example through stateful operations such as count() or through interactive queries.

## Memory management
### Record caches
With Kafka Streams, you can specify the total memory (RAM) size that is used for an instance of a processing topology. This memory is used for internal caching and compacting of records before they are written to state stores, or forwarded downstream to other nodes. These caches differ slightly in implementation in the DSL and Processor API.

The specified cache size is divided equally among the Kafka Stream threads of a topology. Memory is shared over all threads per instance. Each thread maintains a memory pool accessible by its tasks’ processor nodes for caching. Specifically, this is used by stateful processor nodes that perform aggregates and thus have a state store.

![image](https://docs.confluent.io/current/_images/streams-record-cache.png)

The cache has three functions. First, it serves as a read cache to speed up reading data from a state store. Second, it serves as a write-back buffer for a state store. A write-back cache allows for batching multiple records instead of sending each record individually to the state store. It also reduces the number of requests going to a state store (and its changelog topic stored in Kafka if it is a persistent state store) because records with the same key are compacted in cache. Third, the write-back cache reduces the number of records going to downstream processor nodes as well.

Thus, without requiring you to invoke any explicit processing operators in the API, these caches allow you to make trade-off decisions between:
- When using smaller cache sizes: larger rate of downstream updates with shorter intervals between updates.
- When using larger cache sizes: smaller rate of downstream updates with larger intervals between updates. Typically, this results reduced network IO to Kafka and reduced local disk IO to RocksDB-backed state stores, for example.

The final computation results are identical regardless of the cache size (including a disabled cache), which means it is safe to enable or disable the cache. It is not possible to predict when or how updates will be compacted because this depends on many factors, including:
- Cache size.
- Characteristics of the data being processed.
- Configuration parameters, for example commit.interval.ms.

## Fault Tolerance
Kafka Streams builds on fault-tolerance capabilities integrated natively within Kafka. Kafka partitions are highly available and replicated; so when stream data is persisted to Kafka it is available even if the application fails and needs to re-process it. Tasks in Kafka Streams leverage the fault-tolerance capability offered by the Kafka consumer client to handle failures. If a task runs on a machine that fails, Kafka Streams automatically restarts the task in one of the remaining running instances of the application.

In addition, Kafka Streams makes sure that the local state stores are robust to failures, too. For each state store, it maintains a replicated changelog Kafka topic in which it tracks any state updates. These changelog topics are partitioned as well so that each local state store instance, and hence the task accessing the store, has its own dedicated changelog topic partition. Log compaction is enabled on the changelog topics so that old data can be purged safely to prevent the topics from growing indefinitely. If tasks run on a machine that fails and are restarted on another machine, Kafka Streams guarantees to restore their associated state stores to the content before the failure by replaying the corresponding changelog topics prior to resuming the processing on the newly started tasks. As a result, failure handling is completely transparent to the end user.


## Flow Control with Timestamps
Kafka Streams regulates the progress of streams by the timestamps of data records by attempting to synchronize all source streams in terms of time. By default, Kafka Streams will provide your application with event-time processing semantics. This is important especially when an application is processing multiple streams (i.e., Kafka topics) with a large amount of historical data. For example, a user may want to re-process past data in case the business logic of an application was changed significantly, e.g. to fix a bug in an analytics algorithm. Now it is easy to retrieve a large amount of past data from Kafka; however, without proper flow control, the processing of the data across topic partitions may become out-of-sync and produce incorrect results.

As mentioned in the Concepts section, each data record in Kafka Streams is associated with a timestamp. Based on the timestamps of the records in its stream record buffer, stream tasks determine the next assigned partition to process among all its input streams. However, Kafka Streams does not reorder records within a single stream for processing since reordering would break the delivery semantics of Kafka and make it difficult to recover in the face of failure. This flow control is best-effort because it is not always possible to strictly enforce execution order across streams by record timestamp; in fact, in order to enforce strict execution ordering, one must either wait until the system has received all the records from all streams (which may be quite infeasible in practice) or inject additional information about timestamp boundaries or heuristic estimates such as MillWheel’s watermarks.


## Backpressure
Kafka Streams does not use a backpressure mechanism because it does not need one. Using a depth-first processing strategy, each record consumed from Kafka will go through the whole processor (sub-)topology for processing and for (possibly) being written back to Kafka before the next record will be processed. As a result, no records are being buffered in-memory between two connected stream processors. Also, Kafka Streams leverages Kafka’s consumer client behind the scenes, which works with a pull-based messaging model that allows downstream processors to control the pace at which incoming data records are being read.

The same applies to the case of a processor topology that contains multiple independent sub-topologies, which will be processed independently from each other (cf. Parallelism Model). For example, the following code defines a topology with two independent sub-topologies:
```
stream1.to("my-topic");
stream2 = builder.stream("my-topic");
```
Any data exchange between sub-topologies will happen through Kafka, i.e. there is no direct data exchange (in the example above, data would be exchanged through the topic “my-topic”). For this reason there is no need for a backpressure mechanism in this scenario, too.

# Quickstart
[Quickstart](https://docs.confluent.io/current/streams/quickstart.html#streams-quickstart)

启动Kafka Cluster，参考[这里](http://note.youdao.com/noteshare?id=845ec73e19686b89d91e448ecddd8ab8&sub=622869918F0A4D389FCE1B5FA8F8B885).

分别创建streams-plaintext-input和streams-wordcount-output这两个topic：
```
[root@node19216886145 ~]# kafka-topics.sh --create --zookeeper 192.168.86.145:2181 --replication-factor 2 --partitions 2 --topic streams-plaintext-input
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
Created topic "streams-plaintext-input".
[root@node19216886145 ~]# kafka-topics.sh --create --zookeeper 192.168.86.145:2181 --replication-factor 2 --partitions 2 --topic streams-wordcount-output
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
Created topic "streams-wordcount-output".
```

向临时文件/tmp/file-input.txt中写入数据：
```
[root@node19216886145 ~]# echo -e "all streams lead to kafka\nhello kafka streams\njoin kafka summit" > /tmp/file-input.txt
```

将临时文件中的数据写入streams-plaintext-input这个topic中：
```
[root@node19216886145 ~]# cat /tmp/file-input.txt | kafka-console-producer.sh --broker-list localhost:9092 --topic streams-plaintext-input
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
>>>>[root@node19216886145 ~]#
```

在kafka streams(word counter)中处理streams-plaintext-input这个topic中的数据：
```
[root@node19216886145 ~]# kafka-run-class.sh org.apache.kafka.streams.examples.wordcount.WordCountDemo
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
[2018-09-17 18:28:33,147] WARN The configuration 'admin.retries' was supplied but isn't a known config. (org.apache.kafka.clients.consumer.ConsumerConfig)
```

从streams-wordcount-output这个topic中读取kafka streams(word counter)处理的结果：
```
[root@node19216886145 ~]# kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-wordcount-output --from-beginning --formatter kafka.tools.DefaultMessageFormatter --property print.key=true --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N

all     1
streams 1
lead    1
to      1
kafka   1
hello   1
kafka   2
streams 2
join    1
kafka   3
summit  1
```
这里，第一列是Kafka消息的key的字符串格式，第二列是消息的值，long类型。可以通过Ctrl+c命令来终止控制台输出。

但是为什么会出现重复的条目？比如streams出现了两次？这是因为wordCount应用程序的输出实际上是持续更新的流，其中每行记录是一个单一的word(即Message Key，比如Kafka)的计数，对于同一个Key的多个记录，后一个记录是前一个的更新。

下面的两个图说明了在输出之后发生了什么。第一列显示KTable<String, Long>即countByKey的计数当前状态的演化，第二列表示从状态更新到KTable的结果和最终结果，一旦产生从KTable<String, Long>转到KStream<String, Long>的记录，相应结果就会被输出到Kafka。

![image](https://img-blog.csdn.net/20160721193931314?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![image](https://img-blog.csdn.net/20160721193950580?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
         

# Further more readings
[Introducing Kafka Streams: Stream Processing Made Simple](https://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple/)

[kafka Stream概念](https://blog.csdn.net/u012373815/article/details/53648757)

[Kafka Streams介绍(—)](https://blog.csdn.net/ransom0512/article/details/51971112)

[KafkaStreams介绍(二)](https://blog.csdn.net/ransom0512/article/details/51985983)

[Kafka Streams介绍(三)-概念](https://blog.csdn.net/ransom0512/article/details/52038548)

[KafkaStreams介绍(四)-架构](https://blog.csdn.net/ransom0512/article/details/52105379)

[Kafka Stream介绍](https://blog.csdn.net/opensure/article/details/51507698)
