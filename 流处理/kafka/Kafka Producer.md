# 提纲
[toc]

## 源码分析
[***Kafka1.1.0源码分析——KafkaProducer发送消息的过程](http://cxy7.com/articles/2018/06/24/1529818465073.html)

[Kafka Producer整体流程](https://www.jianshu.com/p/51b5e350971a)

## 使用经验总结

[关于高并发下kafka producer send异步发送耗时问题的分析](https://www.cnblogs.com/dafanjoy/p/10292875.html)
```
作者在测试中发现采用同一个kafka producer实例进行异步send的时候，单线程发送反而比多线程发送效率高出几倍，究其原因，是因为send操作并不是真正的数据发送，真正的数据发送由kafka producer中的另一个ProducerSendThread线程执行的，即使是多线程发送，但是因为多个线程共用同一个kafka producer实例，所以最终真正发送数据到broker的线程仍然只有一个。
```


[kafka producer线程与吞吐量](https://blog.csdn.net/iteye_4143/article/details/82651062)
```
本文提到为了充分利用kafka的高吞吐量，最好用多个kafka producer实例，因为在进行异步send的时候，每个kafka producer实例只包含一个ProducerSendThread线程去真正执行send操作。
```

[kafka producer性能调优](https://www.cnblogs.com/videring/articles/6438912.html)

[Kafka producer 发送效率低下问题解决与原因分析](https://bigzuo.github.io/2017/03/13/Kafka-producer-%E5%8F%91%E9%80%81%E6%95%88%E7%8E%87%E4%BD%8E%E4%B8%8B%E9%97%AE%E9%A2%98%E8%A7%A3%E5%86%B3%E4%B8%8E%E5%8E%9F%E5%9B%A0%E5%88%86%E6%9E%90/)
```
本文中作者首先整理了旧版本的API中同步发送和异步发送的流程，比较浅显易懂。

然后列举出了一些新版本的API中的一些特性。

最后特别强调：“无论是新版API还是旧版本API的异步发送模式，其实执行真正发送操作的只有一个线程，并不存在发送线程池，所以，在一台机器上，如果只有一个producer实例，则一旦写入数据继续增大，超过本地缓存设置的最大容量，就会造成阻塞或者抛出异常，并且所有使用该producer实例的线程都会受影响。所以实际使用时，如果业务数据量过大，建议自己维护一个线程池，创建多个producer实例，实现发送的最大效率，并且某个producer异常时，不影响其他producer实例工作。但同时也注意本次内存分配的开销，避免内存分配过大，影响系统其他性能。”。
```

[kafka性能调优解密（二）-- Producer端](https://bbs.huaweicloud.com/blogs/053ca8b8873311e7b8317ca23e93a891)

[Kafka多线程生产消费](https://blog.csdn.net/charry_a/article/details/79621324)
```
作者在这里跟我们举了一个如何使用多线程的kafka producer和consumer的例子。
```

