# Outline
[toc]

# Stream-Based Architecture
## Essential characteristics in messaging technology
- Full independence of producer and consumer
- Persistency
- Enormously high rates of messages/second
- Naming of topics
- A replaceable sequence with strong ordering preserved in the stream of events
- Fault tolerance
- Geo-distributed replication
    
## Difference between Apache Kafka and older message-passing system such as Apache ActiveMQ and RabbitMQ
One big difference is that persistence was a high-cost, optional capability for older systems and typically decresed performance by as much as two ofders of magnitude. In contrast, systems like Kafka and MapR Streams persists all messages automatically while still handling a gigabytes or more per second of message traffic per server.

One big reason for the large discrepancy in performance is that Kafka and related systems do not support message-by-message acknowledgement. Instead, services read messages in order and simply occasionally update a cursor with the offset of the latest unread message. Furthermore, Kafka is focused specifically on message handling rather than providing data transformations or task scheduling. That limited scope helps Kafka achieve very high performance.
    
## Choice of Real-Time Aalytics
### Apache Storm
Storm’s approach is real-time processing of unbounded streams.
"doing for realtime processing what Hadoop did for batch processing."

### Apache Spark Streaming
Spark Streaming is one of the subprojects that comprise Apache Spark.

Spark accelerated the evolution of computation in the Hadoop ecosystem by providing speed through an innovation that allowed data to be loaded into memory and then queried repeatedly. Spark Core uses an abstraction known as a Resilient Distributed Dataset (RDD). When jobs are too large for in-memory, Spark spills data to disk. Spark requires a distributed data storage system (such as Cassandra, HDFS, or MapR-FS) and a framework to manage it. 

Spark Streaming uses microbatching to approximate real-time stream analytics. This means that a batch program is run at frequent intervals to process all recently arrived data together with state stored from previous data.

### Apache Flink
Apache Flink is a highly scalable, high-performance processing engine that can handle low latency, real-time analytics as well as batch analytics. 

Like Spark or Storm, Flink requires a distributed storage system. Flink has already demonstrated very high performance at scale, even while providing a real-time level of latency.

### Apache Apex
Apache Apex is a scalable, high-performance processing engine that, like Apache Flink, is designed to provide both batch and low-latency stream processing. Like the other streaming analytics tools described here, Apex requires a storage platform. 

### Comparison of Capabilities for Streaming analytics
**Fundamentals**

Any technology used for analytics in this style of architecture needs to be highly scalable, capable of starting and stopping without losing information, and able to interface with messaging technologies with capabilities similar to Kafka and MapR Streams.
    
**Performance and low latency**

At present, Flink and Apex probably have the strongest performance at very low latency of the given choices, with Storm providing a medium level of performance with real-time processing.

**Exactly-once delivery**

It is useful to provide exactly-once guarantees because many situations require them. Spark Streaming, Flink, and Apex all guarantee exactly-once processing. Storm works with at-least-once delivery. With the use of an extension called Trident, it is possible to reach exactly-once behavior with Storm, but this may cause some reduction in performance.
    
**Windowing**

This term refers to the time period over which aggregations are made in stream processing. Windowing can be defined in dif‐ ferent ways, and these vary in their application to particular use cases. Time-based windowing groups together events that occur during a specific time interval, and is useful for asking questions such as, “how many transactions have taken place in the last minute?” Spark Streaming, Flink, and Apex all have configurable, time-based windowing capabilities. Windowing in Storm is a bit more primitive.
    
Time may not always be the best way to determine an aggregation window. Another way to define a window is to build programmatic triggers, which allow windows to vary in length, but still require synchronization of windows for different aggregations. Flink, Apex, and Spark do trigger-based windowing. Flink and Apex also do windowing based on the content of data.
            
# Kafka as Streaming Transport
## Key technical innovations of Kafka
**Requiring all messages to be acknowledged in order.**

This eliminated the need to track acknowledgements on a permessage, per-listener basis and allowed a reader’s operations to be very similar to reading a file.

**Setting the expectation that messages would be persisted for days or even weeks.**

This eliminated the requirement to track when readers have finished with particular messages by allowing the retention time to be set so long that readers are almost certain to be finished with messages before they are deleted.

**Requiring consumers to manage the oﬀset of the next message that they will process.**

While the actual offsets for committed messages are stored by Kafka (using Apache Zookeeper), offsets for independent consumers are independent. Applications can even manage their offsets outside of Kafka entirely.

The technical impact of these innovations is that Kafka can write messages to a file system. The files are written sequentially as messages are produced, and they are read sequentially as messages are consumed. These design decisions mean that nonsequential reading or writing of files by a Kafka message broker is very, very rare, and that lets Kafka handle messages at very high speeds.

## Kafka
[Kafka](http://note.youdao.com/noteshare?id=dec284abbce6e426a62426890843d087&sub=AF7E81AEA11F4362B9AFAD86E20C8847)

# MapR Streams
Developed as a ground-up reimplementation of the Apache Kafka API, MapR Streams provides the same basic functions of Kafka but also some additional capabilities.

## Innovations in MapR Streams
1. MapR Streams includes a new file system object type known as a stream that has no parallel in Kafka. Streams are first-class objects in the MapR file system, alongside files, directories, links, and NoSQL tables.
2. A Kafka cluster consists of a number of server processes called brokers that collectively manage message topics, while a MapR cluster has no equivalent of a broker.
3. Topics and partitions are stored in the stream objects on a MapR cluster. There is no equivalent of a stream in a Kafka cluster since topics and partitions are the only visible objects.
4. Each MapR stream can contain hundreds of thousands or more topics and partitions, and each MapR cluster can have millions of streams. In comparison, it is not considered good practice to have more than about a thousand partitions on any single Kafka broker.
5. MapR streams can be replicated to different clusters across intermittent network connections. The replication pattern can contain cycles without causing problems, and streams can be updated in multiple locations at once. Message offsets are preserved in all such replicated copies.
6. The distribution of topic partitions and portions of partitions across a MapR cluster is completely automated, with no admin‐ istrative actions required. This is different from Kafka, where it is assumed that administrators will manually reposition partition replicas in many situations.
7. The streams in a MapR cluster inherit all of the security, permissioning, and disaster-recovery capabilities of the basic MapR platform.
8. Most configuration parameters for producers and consumers that are used by Kafka are not supported by MapR Streams.

## History and Context of MapR’s Streaming System
The adoption of Apache Kafka for building large-scale applications over the last few years has been dramatic, and it has opened the way for support of a streaming approach. 

Naturally a large number of those using Kafka up to now have been technology early-adopters, which is typical in this phase of the lifecycle of an open source project. Early adopters are often able to achieve surprising levels of success very quickly with new projects like Kafka, as they have done previously with Apache projects Hadoop, Hive, Drill, Solr/Lucene, and others. 

These projects are groundbreaking in terms of what they make possible, and it is a natural evolution for new innovations to be implemented in emerging technology businesses before they are mature enough to be adopted by large enterprises as core technologies.To improve the maturity of these projects, we need to solve standard questions of manageability, complexity, scalability, integration with other major technology investments, and security. In the past, with other open source projects, there have been highly variable success rates in dealing with these “enterprisey” issues. Solr, for instance, responded to security concerns by simply ruling any such concerns as out of scope, to be handled by perimeter security. The Hive community has responded to concerns about integration with SQL-generating tools by adopting Apache Calcite as a query parser and planner.

In some cases, these issues are very difficult to address within the context of the existing open source implementation. For instance, Hadoop’s default file system, HDFS, supports append-only files, can not support very many files, has had a checkered history of instability, and imposes substantial overhead because it requires use of machines that serve only to maintain metadata. Truly fixing these problems using evolutionary improvements to the existing architecture and code base would be extremely difficult. Some improvements can be made by introducing namespace federation, but while these changes may help with one problem (the limited number of files), they may exacerbate another (the amount of nonproductive overhead and instability).

If, however, a project establishes solid interface standards in the form of simple APIs early on, these problems admit a solution in the form of a complete reimplementation, possibly in open source but not necessarily. As such, Accumulo was a reimplementation of Apache HBase, incorporating features that HBase was having difficulties providing. Hypertable was another reimplementation done commercially. MapR-DB is a third reimplementation of HBase that stays close to the original HBase API as well as providing a document-style version with a JSON API.

**The outcome of a reimplementation typically depends on whether or not the new project actually solves an important problem that is difficult to change in the original project, and whether or not the original project has defined a clean enough API to be able to be reliably implemented.**

With Kafka, there are issues that look like they will turn out to be important, and many of these appear to be difficult to resolve in the original project. These include complexity of administration, scaling, security, and the question of how to handle multiple models of data storage (such as files and tables), as well as data streams in a single, converged architecture.

MapR Streams is a reimplementation of Kafka that aims to solve these problems while keeping very close to the API that Kafka provides. Even though the interface is similar, the implementation is very different. MapR Streams is fundamentally based on the same MapR technology core that anchors MapR-FS and MapR-DB. This allows MapR Streams to reuse many of the solutions already available in MapR’s technology core for administration, security, disaster protection, and scale. Interestingly, reimplementing Kafka using different technology only became possible with the Kafka 0.9 API. Earlier APIs exposed substantial amounts of internal implementation details that made it nearly impossible to reimplement in a substantially improved way.

## How MapR Streams Works
