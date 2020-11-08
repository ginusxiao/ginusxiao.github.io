# 提纲
[toc]

## SparkContextJavaFunctions
该类中主要封装了一个SparkContext。

主要方法：
```
- public <T> CassandraJavaRDD<T> toJavaRDD(CassandraRDD<T> rdd, Class<T> targetClass)
    - 将CassandraRDD转换为CassandraJavaRDD 
- public <K, V> CassandraJavaPairRDD<K, V> toJavaPairRDD(CassandraRDD<Tuple2<K, V>> rdd, Class<K> keyClass, Class<V> valueClass)
    - 将CassandraRDD转换为CassandraJavaPairRDD
- public CassandraTableScanJavaRDD<CassandraRow> cassandraTable(String keyspace, String table) 
    - 基于给定的keyspace和table创建一个CassandraTableScanJavaRDD，使用GenericJavaRowReaderFactory，返回关于CassandraRow的RDD
- public <T> CassandraTableScanJavaRDD<T> cassandraTable(String keyspace, String table, RowReaderFactory<T> rrf)
    - 基于给定的keyspace和table创建一个CassandraTableScanJavaRDD，使用自定义的RowReaderFactory，返回关于T类型的对象的RDD
```


## StreamingContextJavaFunctions
该类中主要封装了一个StreamingContext。

## RDDAndDStreamCommonJavaFunctions
该类是一个抽象类，定义了一些抽象方法，以及一个内部类WriterBuilder。

主要抽象方法(由子类负责实现)：
```
- public abstract CassandraConnector defaultConnector()
    - 获取CassandraConnector实例
- protected abstract SparkConf getConf()
    - 获取SparkConf
- protected abstract void saveToCassandra(String keyspace, String table, RowWriterFactory<T> rowWriterFactory, ColumnSelector columnNames, WriteConf conf, CassandraConnector connector)
    - 将RDD/DStream写入到cassandra table中
```


主要方法：
- public WriteConf defaultWriteConf()
    - 从getConf()中返回的SparkConf获取默认的WriteConf

### 内部类WriterBuilder
该类主要用于存放“控制RDD或者DStream中的对象如何写入到Cassandra table”的相关参数。

主要成员：
- keyspaceName：keyspace
- tableName：table
- rowWriterFactory：用于控制RDD/DStream的对象如何映射到Cassandra row
- columnSelector：控制哪些列可以被写入
- connector：关于Cassandra server的连接
- writeConf：写操作相关的参数，主要是关于性能和错误处理相关的参数

主要方法：
```
- public WriterBuilder(String keyspaceName, String tableName, RowWriterFactory<T> rowWriterFactory,
ColumnSelector columnSelector, CassandraConnector connector, WriteConf writeConf)
    - 构造方法
- public WriterBuilder withConnector(CassandraConnector connector)
    - 修改WriterBuilder中的connector信息
- public WriterBuilder withWriteConf(WriteConf writeConf)
    - 修改WriterBuilder中的writeConf信息
- public WriterBuilder withRowWriterFactory(RowWriterFactory<T> factory)
    - 修改WriterBuilder中的rowWriterFactory信息
- public WriterBuilder withColumnSelector(ColumnSelector columnSelector)
    - 修改WriterBuilder中的columnSelector信息
- public WriterBuilder withBatchSize(BatchSize batchSize)
- public WriterBuilder withBatchGroupingBufferSize(int batchGroupingBufferSize)
- public WriterBuilder withBatchGroupingKey(BatchGroupingKey batchGroupingKey)
- public WriterBuilder withConsistencyLevel(ConsistencyLevel consistencyLevel)
- public WriterBuilder withParallelismLevel(int parallelismLevel)
- public WriterBuilder withThroughputMBPS(int throughputMBPS)
- public WriterBuilder withTaskMetricsEnabled(boolean taskMetricsEnabled)
- public WriterBuilder withIfNotExists(boolean ifNotExists)
- public WriterBuilder withIgnoreNulls(boolean ignoreNulls)
- private WriterBuilder withTimestamp(TimestampOption timestamp)
    - 上述所有方法，分别用于修改WriterBuilder中的writeConf信息中的相关信息
- public WriterBuilder withConstantTimestamp(long timeInMicroseconds)
- public WriterBuilder withConstantTimestamp(Date timestamp)
- public WriterBuilder withConstantTimestamp(DateTime timestamp)
    - 上述3个方法，均用于设置 WriterBuilder中的writeConf信息中的timestamp信息
- public WriterBuilder withAutoTimestamp()
    - 设置 WriterBuilder中的writeConf中关于timestamp的设置：使用Cassandra自己的timestamp
- public WriterBuilder withPerRowTimestamp(String placeholder)
- private WriterBuilder withTTL(TTLOption ttl)
- public WriterBuilder withConstantTTL(int ttlInSeconds)
- public WriterBuilder withAutoTTL()
- public WriterBuilder withPerRowTTL(String placeholder)
    - 设置 WriterBuilder中的writeConf中关于TTL的设置 
- public void saveToCassandra()
    - 调用RDDAndDStreamCommonJavaFunctions中的saveToCassandra方法，并采用WriterBuilder中的配置
```


## RDDJavaFunctions
该类继承自RDDAndDStreamCommonJavaFunctions，主要是对RDDFunctions(spark cassandra connector中自己提供的，scala语言编写)的封装。

主要成员：
- rdd：spark rdd，scala语言编写
- rddFunctions：spark-cassandra connector中实现的RDD相关的方法集合，scala语言编写

主要方法：

```
- public RDDJavaFunctions(RDD<T> rdd)
    - 构造方法，会基于传递进来的rdd参数来设置类成员rdd和rddFunctions
- public CassandraConnector defaultConnector()
    - 实现在RDDAndDStreamCommonJavaFunctions中定义的抽象的defaultConnector方法，用于获取CassandraConnector
- public SparkConf getConf()
    - 实现在RDDAndDStreamCommonJavaFunctions中定义的抽象的getConf方法，用于获取RDD对应的SparkConf
- protected abstract void saveToCassandra(String keyspace, String table, RowWriterFactory<T> rowWriterFactory, ColumnSelector columnNames, WriteConf conf, CassandraConnector connector)
    - 实现在RDDAndDStreamCommonJavaFunctions中定义的抽象的saveToCassandra方法
    - 通过调用rddFunctions.saveToCassandra来实现
- public void deleteFromCassandra(String keyspace, String table, RowWriterFactory<T> rowWriterFactory, ColumnSelector deleteColumns, ColumnSelector keyColumns, WriteConf conf,  CassandraConnector connector)
    - 从cassandra中删除数据，调用rddFunctions.deleteFromCassandra来实现
- public <U> JavaPairRDD<U, Iterable<T>> spanBy(final Function<T, U> f, ClassTag<U> keyClassTag)
    - 该方法还有待深入理解
- public <R> CassandraJavaPairRDD<T, R> joinWithCassandraTable(String keyspaceName, String tableName, ColumnSelector selectedColumns, ColumnSelector joinColumns, RowReaderFactory<R> rowReaderFactory, RowWriterFactory<T> rowWriterFactory)
    - 对RDD和Cassandra table进行join
- public JavaRDD<T> repartitionByCassandraReplica(String keyspaceName, String tableName, int partitionsPerHost, ColumnSelector partitionkeyMapper, RowWriterFactory<T> rowWriterFactory)
    - 基于给定的table对RDD进行重新分区
```


## PairRDDJavaFunctions
该类继承自RDDJavaFunctions，而RDDJavaFunctions则继承自RDDAndDStreamCommonJavaFunctions，主要是对PairRDDFunctions的封装。

主要成员：
- pairRDDFunctions：spark-cassandra connector中实现的cassandra相关的方法集合，scala语言编写
    - PairRDDJavaFunctions类中只有这一个成员
    - rdd则封装在PairRDDJavaFunctions所继承的RDDJavaFunctions中

主要方法：
```
- public PairRDDJavaFunctions(RDD<Tuple2<K, V>> rdd)
    - 构造方法，首先调用父类RDDJavaFunctions的构造方法，然后创建一个PairRDDFunctions
    - 因为它继承自RDDJavaFunctions，所以它也具有RDDJavaFunctions中所有的方法
- public JavaPairRDD<K, Collection<V>> spanByKey(ClassTag<K> keyClassTag)
    - 该方法有待进一步理解
```


## DStreamJavaFunctions
该类继承自RDDAndDStreamCommonJavaFunctions。

主要成员：
- dstream：类型为DStream，DStream是在spark中为spark streaming而定义的
- dsf：类型为DStreamFunctions，DStreamFunctions是spark-cassandra connector中实现的DStream相关的方法集合，scala语言编写

主要方法：
```
- DStreamJavaFunctions(DStream<T> dStream)
    - 构造方法
- public CassandraConnector defaultConnector()
- protected SparkConf getConf()
- protected void saveToCassandra
    - 上述3个方法，分别用于重载父类RDDAndDStreamCommonJavaFunctions中定义的3个方法
```


## CassandraJavaRDD
CassandraJavaRDD是对CassandraRDD的封装，它继承自JavaRDD(在Spark中定义)。

主要方法：
```
- public CassandraJavaRDD<R> select(String... columnNames)
    - 从CassandraJavaRDD中选择指定列中的数据
- public CassandraJavaRDD<R> where(String cqlWhereClause, Object... args)
    - 从CassandraJavaRDD中执行where语句
- public CassandraJavaRDD<R> withAscOrder()
    - 升序排列
- public CassandraJavaRDD<R> withDescOrder()
    - 降序排列
- public CassandraJavaRDD<R> withConnector(CassandraConnector connector)
    - 修改CassandraJavaRDD中的CassandraConnector
- public CassandraJavaRDD<R> withReadConf(ReadConf config)
    - 修改CassandraJavaRDD中的ReadConf
- public CassandraJavaRDD<R> limit(Long rowsNumber)
- public CassandraJavaRDD<R> perPartitionLimit(Long rowsNumber)
    - 从每个Cassandra Partition中最多返回给定数目的条目
- public <K> JavaPairRDD<K, Iterable<R>> spanBy(Function<R, K> f, ClassTag<K> keyClassTag)
- public <K> JavaPairRDD<K, Iterable<R>> spanBy(Function<R, K> f, Class<K> keyClass)
- public long cassandraCount()
    - 返回CassandraJavaRDD中数据条目
```

## CassandraJavaPairRDD
CassandraJavaPairRDD是对CassandraRDD<Tuple2<K, V>>的封装，继承自JavaPairRDD(Spark中定义).

主要方法：
```
- public CassandraJavaPairRDD(CassandraRDD<Tuple2<K, V>> rdd, ClassTag<K> keyClassTag,  ClassTag<V> valueClassTag)
- public CassandraJavaPairRDD(CassandraRDD<Tuple2<K, V>> rdd, Class<K> keyClass, Class<V> valueClass
)
- public CassandraRDD<Tuple2<K, V>> rdd()
- public CassandraJavaPairRDD<K, V> select(String... columnNames)
- public CassandraJavaPairRDD<K, V> select(ColumnRef... selectionColumns) 
- public CassandraJavaPairRDD<K, V> where(String cqlWhereClause, Object... args)
- public CassandraJavaPairRDD<K, V> withAscOrder()
- public CassandraJavaPairRDD<K, V> withDescOrder()
- public CassandraJavaPairRDD<K, V> limit(Long rowsNumber)
- public CassandraJavaPairRDD<K, V> perPartitionLimit(Long rowsNumber)
- public CassandraJavaPairRDD<K, V> withConnector(CassandraConnector connector)
- public CassandraJavaPairRDD<K, V> withReadConf(ReadConf config)
- public JavaPairRDD<K, Collection<V>> spanByKey()
    - 利用PairRDDJavaFunctions.spanByKey来实现
- public <U> JavaPairRDD<U, Iterable<Tuple2<K, V>>> spanBy(Function<Tuple2<K, V>, U> function, ClassTag<U> uClassTag)
    - 利用PairRDDJavaFunctions.spanBy来实现
- public <U> JavaPairRDD<U, Iterable<Tuple2<K, V>>> spanBy(Function<Tuple2<K, V>, U> function, Class<U> uClass)
    - 利用PairRDDJavaFunctions.spanBy来实现
- public long cassandraCount()
```

## CassandraJoinJavaRDD
CassandraJoinJavaRDD继承自CassandraJavaPairRDD，它是对CassandraJoinRDD的封装。

## CassandraTableScanJavaRDD
CassandraTableScanJavaRDD继承自CassandraJavaRDD，它是对CassandraTableScanRDD的封装。

## CassandraJavaUtil
CassandraJavaUtil是Cassandra connector Java API的主要入口。

主要方法：
```
- public static SparkContextJavaFunctions javaFunctions(SparkContext sparkContext)
- public static SparkContextJavaFunctions javaFunctions(JavaSparkContext sparkContext)
    - 上面2个方法用于获取SparkContextJavaFunctions
- public static <T> RDDJavaFunctions<T> javaFunctions(RDD<T> rdd)
- public static <T> RDDJavaFunctions<T> javaFunctions(JavaRDD<T> rdd)
    - 基于给定RDD获取RDDJavaFunctions
- public static <K, V> PairRDDJavaFunctions<K, V> javaFunctions(JavaPairRDD<K, V> rdd)
    - 基于给定JavaPairRDD获取PairRDDJavaFunctions
- public static <T> RowReaderFactory<T> mapColumnTo(Class<T> targetClass)
    - 构建一个RowReaderFactory，用于将一个Column映射为targetClass类型的实例
- public static <T> RowReaderFactory<List<T>> mapColumnToListOf(Class<T> targetClass)
    - 构建一个RowReaderFactory，用于将一个Column映射为一个list，list中的元素类型targetClass类型
- public static <T> RowReaderFactory<Set<T>> mapColumnToSetOf(Class<T> targetClass)
    - 构建一个RowReaderFactory，用于将一个Column映射为一个set，set中的元素类型targetClass类型
- public static <K, V> RowReaderFactory<Map<K, V>> mapColumnToMapOf(Class<K> targetKeyClass, Class<V> targetValueClass)
    - 构建一个RowReaderFactory，用于将一个Column映射为一个map，map中的key类型为targetKeyClass类型，value类型为targetValueClass类型
- public static <T> RowReaderFactory<T> mapColumnTo(Class targetClass, Class typeParam1, Class... typeParams)
    - 构建一个RowReaderFactory，用于将一个Column映射为targetClass类型的实例，构建targetClass类型的实例时所用到的参数为typeParam1，...，typeParams
- public static <T> RowReaderFactory<T> mapColumnTo(TypeConverter<T> typeConverter)
    - 构建一个RowReaderFactory，其采用typeConverter将一个Column进行转换
- public static <T> RowReaderFactory<T> mapRowTo(Class<T> targetClass, Map<String, String> columnMapping)
- public static <T> RowReaderFactory<T> mapRowTo(Class<T> targetClass, Pair... columnMappings)
- public static <T> RowReaderFactory<T> mapRowTo(Class<T> targetClass, ColumnMapper<T> columnMapper)
    - 上述3个方法，构建一个RowReaderFactory，用于将一个row的数据转换为targetClass类型的实例
- public static <A> RowReaderFactory<Tuple1<A>> mapRowToTuple(Class<A> a)
- public static <A, B> RowReaderFactory<Tuple2<A, B>> mapRowToTuple(Class<A> a, Class<B> b)
- ...
- public static <A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P, Q, R, S, T, U, V> RowReaderFactory<Tuple22<A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P, Q, R, S, T, U, V>> mapRowToTuple(Class<A> a,Class<B> b,Class<C> c,Class<D> d,Class<E> e,Class<F> f,Class<G> g,Class<H> h,Class<I> i,Class<J> j,Class<K> k,Class<L> l,Class<M> m,Class<N> n,Class<O> o,Class<P> p,Class<Q> q,Class<R> r,Class<S> s,Class<T> t,Class<U> u,Class<V> v)
    - 上述22个方法，均用于创建一个RowReaderFactory，并分别将一个row映射为1 - 22个元素的tuple
- public static <T> RowWriterFactory<T> mapToRow(Class<?> targetClass, ColumnMapper<T> columnMapper)
    - 
- public static <T> RowWriterFactory<T> safeMapToRow(Class<T> targetClass, ColumnMapper<T> columnMapper)
    - 
- public static <T> RowWriterFactory<T> mapToRow(Class<T> sourceClass, Map<String, String> fieldToColumnNameMap)
    - 
- public static <T> RowWriterFactory<T> mapToRow(Class<T> sourceClass, Pair... fieldToColumnNameMappings)
    - 
- public static <A> RowWriterFactory<Tuple1<A>> mapTupleToRow(Class<A> a)
- public static <A, B> RowWriterFactory<Tuple2<A, B>> mapTupleToRow(Class<A> a, Class<B> b)
- ...
- public static <A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P, Q, R, S, T, U, V> RowWriterFactory<Tuple22<A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P, Q, R, S, T, U, V>> mapTupleToRow(Class<A> a,Class<B> b,Class<C> c,Class<D> d,Class<E> e,Class<F> f,Class<G> g,Class<H> h,Class<I> i,Class<J> j,Class<K> k,Class<L> l,Class<M> m,Class<N> n,Class<O> o,Class<P> p,Class<Q> q,Class<R> r,Class<S> s,Class<T> t,Class<U> u,Class<V> v)
    - 上述22个方法，均用于创建一个RowWriterFactory，并分别将1 - 22个元素的tuple映射为一个row
- public static ColumnSelector someColumns(String... columnNames)
    - 创建一个关于参数中指定的这些ColumnNames的ColumnSelector
- public static ColumnRef[] toSelectableColumnRefs(String... columnNames)
    - 获取关于参数中指定的这些ColumnNames的ColumnRef集合
```

## CassandraStreamingJavaUtil
```
- public static StreamingContextJavaFunctions javaFunctions(StreamingContext streamingContext)
    - 基于给定的StreamingContext创建StreamingContextJavaFunctions
- public static StreamingContextJavaFunctions javaFunctions(JavaStreamingContext streamingContext)
    - 基于给定的JavaStreamingContext创建StreamingContextJavaFunctions
- public static <T> DStreamJavaFunctions<T> javaFunctions(DStream<T> dStream)
    - 基于给定的DStream创建DStreamJavaFunctions
- public static <T> DStreamJavaFunctions<T> javaFunctions(JavaDStream<T> dStream)
    - 基于给定的JavaDStream创建DStreamJavaFunctions
```



## 参考
[Accessing Cassandra from Spark in Java](https://www.datastax.com/blog/2014/08/accessing-cassandra-spark-java)
