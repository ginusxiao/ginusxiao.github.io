# 提纲
[toc]

## Spark standalone模式安装
下载的spark版本是：spark-2.1.3-bin-without-hadoop

安装过程参考[这里](http://spark.apache.org/docs/2.1.3/spark-standalone.html)。

### 安装过程中的错误处理
1. java.lang.ClassNotFoundException: org.slf4j.Logger
```
Error: A JNI error has occurred, please check your installation and try again
Exception in thread "main" java.lang.NoClassDefFoundError: org/slf4j/Logger
	at java.lang.Class.getDeclaredMethods0(Native Method)
	at java.lang.Class.privateGetDeclaredMethods(Class.java:2701)
	at java.lang.Class.privateGetMethodRecursive(Class.java:3048)
	at java.lang.Class.getMethod0(Class.java:3018)
	at java.lang.Class.getMethod(Class.java:1784)
	at sun.launcher.LauncherHelper.validateMainClass(LauncherHelper.java:544)
	at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:526)
Caused by: java.lang.ClassNotFoundException: org.slf4j.Logger
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:352)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
	... 7 more
```

开始我使用的是spark-2.1.3-bin-without-hadoop，如果切换到spark-2.1.3-bin-hadoop2.7之后就好了。

我在采用spark local模式下，没这个问题，因为spark local模式下，在pom文件中指定了slf4j-api和slf4j-log4j12等。但是这里我们采用的是spark standalone模式，并使用spark-2.1.3-bin-without-hadoop，它不包括slf4j-api-1.7.7.jar和 slf4j-log4j12-1.7.7.jar这2个jar包，所以需要将slf4j-api-1.7.7.jar和 slf4j-log4j12-1.7.7.jar这2个jar包给拷贝到spark-2.1.3-bin-without-hadoop的jars目录中，执行：
```
cp /opt/yujingPOC/target/lib/slf4j-api-1.7.7.jar /opt/spark-2.1.3-bin-without-hadoop/jars

cp /opt/yujingPOC/target/lib/slf4j-log4j12-1.7.7.jar  /opt/spark-2.1.3-bin-without-hadoop/jars
```

2. java.lang.ClassNotFoundException: org.apache.spark.streaming.kafka010.
KafkaRDDPartition
```
20/03/26 19:52:25 WARN TaskSetManager: Lost task 0.0 in stage 0.0 (TID 0, 10.10.10.11, executor 13): java.lang.ClassNotFoundException: org.apache.spark.streaming.kafka010.
KafkaRDDPartition
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:348)
	at org.apache.spark.serializer.JavaDeserializationStream$$anon$1.resolveClass(JavaSerializer.scala:67)
	at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1867)
	at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1750)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2041)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1572)
	at java.io.ObjectInputStream.defaultReadFields(ObjectInputStream.java:2286)
	at java.io.ObjectInputStream.readSerialData(ObjectInputStream.java:2210)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2068)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1572)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:430)
	at org.apache.spark.serializer.JavaDeserializationStream.readObject(JavaSerializer.scala:75)
	at org.apache.spark.serializer.JavaSerializerInstance.deserialize(JavaSerializer.scala:114)
	at org.apache.spark.executor.Executor$TaskRunner.run(Executor.scala:301)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

我在采用spark local模式下，没这个问题，因为spark local模式下，在pom文件中指定了spark-core_2.11，spark-streaming_2.11，spark-sql_2.11和spark-streaming-kafka-0-10_2.11等。但是这里我们采用的是spark standalone模式，并使用spark-2.1.3-bin-without-hadoop，它不包括spark-streaming-kafka-0-10_2.11这个jar包，所以需要将spark-streaming-kafka-0-10_2.11给拷贝到spark-2.1.3-bin-without-hadoop的jars目录中，执行：
```
cp /opt/yujingPOC/target/lib/spark-streaming-kafka-0-10_2.11-2.1.3.jar /opt/spark-2.1.3-bin-without-hadoop/jars
```

3. java.lang.ClassNotFoundException: org.apache.kafka.common.TopicPartition

我在采用spark local模式下，没这个问题，因为spark local模式下，在pom文件中指定了spark-streaming-kafka-0-10_2.11，它会自动将spark kafka streaming所依赖的jar包，包括kafka相关的jar包，给down下来。但是这里我们采用的是spark standalone模式，并使用spark-2.1.3-bin-without-hadoop，它不包括kafka_2.11-0.10.0.1.jar和kafka-clients-0.10.0.1.jar这2个jar包，所以需要将kafka_2.11-0.10.0.1.jar和kafka-clients-0.10.0.1.jar这2个jar包给拷贝到spark-2.1.3-bin-without-hadoop的jars目录中，执行：
```
cp /opt/yujingPOC/target/lib/kafka_2.11-0.10.0.1.jar /opt/spark-2.1.3-bin-without-hadoop/jars

cp /opt/yujingPOC/target/lib/kafka-clients-0.10.0.1.jar /opt/spark-2.1.3-bin-without-hadoop/jars
```

4. java.lang.ClassCastException: cannot assign instance of java.lang.invo
ke.SerializedLambda to field org.apache.spark.api.java.JavaPairRDD$$anonfun$pairFunToScalaFun$1.x$334 of type org.apache.spark.api.java.function.PairFunction in instance o
f org.apache.spark.api.java.JavaPairRDD$$anonfun$pairFunToScalaFun$1

根据[这里](https://blog.csdn.net/hotdust/article/details/61671448)，在代码中new SparkConf的时候添加.setJars(new String[]{"/opt/yujingPOC/target/yujingPOC.jar"})。其中/opt/yujingPOC/target/yujingPOC.jar就是我们最终运行的程序的classpath。

5. 测试过程中虽然针对某个号码的实时伴随完成了，但是在yugabyte中没有查找到对应的实时伴随结果。

在local模式下没有问题，因为在pom文件中指定了spark-cassandra-connector_2.11，但是在spark standalone模式下，使用的是spark-2.1.3-bin-without-hadoop，它的jars目录中没有spark-cassandra-connector_2.11这个jar包，所以需要拷贝到spark-2.1.3-bin-without-hadoop/jars/目录下：
```
cp /opt/yujingPOC/target/lib/spark-cassandra-connector_2.11-2.0.5-yb-2.jar /opt/spark-2.1.3-bin-without-hadoop/jars/
```










