# 提纲
[toc]

## 测试记录
### 16 clients，256 threads
```
[root@localhost sample-test]# java -jar /opt/benchmark/sample-test/yb-sample-apps.jar --concurrent_clients 16 --num_unique_keys 1000000 --workload CassandraKeyValue --nodes 10.10.10.10:9042 --nouuid --value_size 80 --num_threads_read 256 --num_threads_write  0

...

116 [main] INFO com.yugabyte.sample.apps.AppBase  - Connecting with 16 clients to nodes: /10.10.10.10:9042
804 [main] INFO com.yugabyte.sample.apps.AppBase  - Created a Cassandra table using query: [CREATE TABLE IF NOT EXISTS CassandraKeyValue (k varchar, v blob, primary key (k));]
804 [main] INFO com.yugabyte.sample.Main  - Using 16 writer threads for pure read setup.
3868 [main] INFO com.yugabyte.sample.Main  - Setup step for pure reads done.
8872 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 111792.41 ops/sec (0.45 ms/op), 559007 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 8840 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
13873 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 172303.49 ops/sec (0.86 ms/op), 1420882 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 13841 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
18874 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 191789.47 ops/sec (1.25 ms/op), 2380090 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 18842 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
23875 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 196788.47 ops/sec (1.30 ms/op), 3364163 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 23843 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
28875 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 197658.48 ops/sec (1.29 ms/op), 4352586 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 28843 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
33876 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 195978.45 ops/sec (1.31 ms/op), 5332649 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 33844 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 

...

498936 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 196778.32 ops/sec (1.30 ms/op), 96931405 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 498904 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
503936 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 196802.67 ops/sec (1.30 ms/op), 97915516 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 503904 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
508937 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 195976.08 ops/sec (1.31 ms/op), 98895485 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 508905 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
513937 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 196784.86 ops/sec (1.30 ms/op), 99879506 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 513905 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
518946 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 195615.49 ops/sec (1.31 ms/op), 100859267 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 518914 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
523946 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 195377.36 ops/sec (1.31 ms/op), 101836236 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 523914 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
528947 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 196977.47 ops/sec (1.30 ms/op), 102821232 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 528915 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 |
```

### 16 clients，128 threads
```
[root@localhost sample-test]# java -jar /opt/benchmark/sample-test/yb-sample-apps.jar --concurrent_clients 16 --num_unique_keys 1000000 --workload CassandraKeyValue --nodes 10.10.10.10:9042 --nouuid --value_size 80 --num_threads_read 128 --num_threads_write  0

...

119 [main] INFO com.yugabyte.sample.apps.AppBase  - Connecting with 16 clients to nodes: /10.10.10.10:9042
764 [main] INFO com.yugabyte.sample.apps.AppBase  - Created a Cassandra table using query: [CREATE TABLE IF NOT EXISTS CassandraKeyValue (k varchar, v blob, primary key (k));]
764 [main] INFO com.yugabyte.sample.Main  - Using 16 writer threads for pure read setup.
3933 [main] INFO com.yugabyte.sample.Main  - Setup step for pure reads done.
8937 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 111816.61 ops/sec (0.45 ms/op), 559124 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 8904 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
13938 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 170042.59 ops/sec (0.73 ms/op), 1409690 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 13904 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
18938 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 171918.51 ops/sec (0.74 ms/op), 2269393 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 18905 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 

...

504011 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 172236.97 ops/sec (0.74 ms/op), 85979831 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 503978 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
509011 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 172227.45 ops/sec (0.74 ms/op), 86841032 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 508978 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
514012 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 172335.64 ops/sec (0.74 ms/op), 87702773 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 513979 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
519012 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 172550.56 ops/sec (0.74 ms/op), 88565591 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 518979 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
524012 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 172612.71 ops/sec (0.74 ms/op), 89428722 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 523979 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 |
```


### 8 clients, 256 threads
```
[root@localhost sample-test]# java -jar /opt/benchmark/sample-test/yb-sample-apps.jar --concurrent_clients 8 --num_unique_keys 1000000 --workload CassandraKeyValue --nodes 10.10.10.10:9042 --nouuid --value_size 80 --num_threads_read 256 --num_threads_write  0

...

113 [main] INFO com.yugabyte.sample.apps.AppBase  - Connecting with 8 clients to nodes: /10.10.10.10:9042
733 [main] INFO com.yugabyte.sample.apps.AppBase  - Created a Cassandra table using query: [CREATE TABLE IF NOT EXISTS CassandraKeyValue (k varchar, v blob, primary key (k));]
733 [main] INFO com.yugabyte.sample.Main  - Using 16 writer threads for pure read setup.
3825 [main] INFO com.yugabyte.sample.Main  - Setup step for pure reads done.
8829 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 124603.85 ops/sec (0.40 ms/op), 623077 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 8797 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
13830 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 190202.16 ops/sec (0.78 ms/op), 1574478 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 13798 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
18831 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 199667.06 ops/sec (1.20 ms/op), 2573185 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 18799 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
23832 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 203184.43 ops/sec (1.26 ms/op), 3589240 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 23800 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
28832 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 203949.95 ops/sec (1.25 ms/op), 4609101 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 28800 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
33833 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 201959.63 ops/sec (1.27 ms/op), 5618998 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 33801 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 

...

543888 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 202229.40 ops/sec (1.27 ms/op), 108968115 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 543856 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
548888 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 202317.77 ops/sec (1.26 ms/op), 109979777 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 548856 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
553889 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 201983.67 ops/sec (1.27 ms/op), 110989782 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 553857 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
558889 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 201509.37 ops/sec (1.27 ms/op), 111997440 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 558857 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
563890 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 202764.07 ops/sec (1.26 ms/op), 113011346 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 563858 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 |
```

### 8 clients, 128 threads
```
[root@localhost sample-test]# java -jar /opt/benchmark/sample-test/yb-sample-apps.jar --concurrent_clients 8 --num_unique_keys 1000000 --workload CassandraKeyValue --nodes 10.10.10.10:9042 --nouuid --value_size 80 --num_threads_read 128 --num_threads_write  0

...

122 [main] INFO com.yugabyte.sample.apps.AppBase  - Connecting with 8 clients to nodes: /10.10.10.10:9042
760 [main] INFO com.yugabyte.sample.apps.AppBase  - Created a Cassandra table using query: [CREATE TABLE IF NOT EXISTS CassandraKeyValue (k varchar, v blob, primary key (k));]
760 [main] INFO com.yugabyte.sample.Main  - Using 16 writer threads for pure read setup.
3781 [main] INFO com.yugabyte.sample.Main  - Setup step for pure reads done.
8785 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 127169.18 ops/sec (0.39 ms/op), 635900 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 8751 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
13787 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 188529.29 ops/sec (0.66 ms/op), 1579135 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 13753 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
18788 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 193991.71 ops/sec (0.66 ms/op), 2549350 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 18754 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
23788 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 194265.68 ops/sec (0.66 ms/op), 3520807 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 23754 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
28789 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 193860.53 ops/sec (0.66 ms/op), 4490212 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 28755 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 

...

178808 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 194896.68 ops/sec (0.66 ms/op), 33521592 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 178774 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
183808 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 194610.47 ops/sec (0.66 ms/op), 34494727 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 183774 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
188809 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 193582.80 ops/sec (0.66 ms/op), 35462774 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 188775 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
193810 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 195357.99 ops/sec (0.65 ms/op), 36439654 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 193776 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 |
```


### 8 clients, 64 threads

```
[root@localhost sample-test]# java -jar /opt/benchmark/sample-test/yb-sample-apps.jar --concurrent_clients 8 --num_unique_keys 1000000 --workload CassandraKeyValue --nodes 10.10.10.10:9042 --nouuid --value_size 80 --num_threads_read 64 --num_threads_write  0

...

118 [main] INFO com.yugabyte.sample.apps.AppBase  - Connecting with 8 clients to nodes: /10.10.10.10:9042
753 [main] INFO com.yugabyte.sample.apps.AppBase  - Created a Cassandra table using query: [CREATE TABLE IF NOT EXISTS CassandraKeyValue (k varchar, v blob, primary key (k));]
753 [main] INFO com.yugabyte.sample.Main  - Using 16 writer threads for pure read setup.
3794 [main] INFO com.yugabyte.sample.Main  - Setup step for pure reads done.
8798 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 124426.29 ops/sec (0.35 ms/op), 622188 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 8766 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
13798 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 159199.95 ops/sec (0.40 ms/op), 1418473 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 13766 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
18799 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 160069.91 ops/sec (0.40 ms/op), 2218934 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 18767 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
23799 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 159745.38 ops/sec (0.40 ms/op), 3017736 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 23767 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
28800 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 159225.05 ops/sec (0.40 ms/op), 3813941 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 28768 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 

...

123810 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 159716.32 ops/sec (0.40 ms/op), 18990180 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 123778 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
128811 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 160276.60 ops/sec (0.40 ms/op), 19791689 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 128779 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
133811 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 159819.73 ops/sec (0.40 ms/op), 20590879 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 133779 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 | 
138812 [Thread-18] INFO com.yugabyte.sample.common.metrics.MetricsTracker  - Read: 159311.79 ops/sec (0.40 ms/op), 21387509 total ops  |  Write: 0.00 ops/sec (0.00 ms/op), 0 total ops  |  Uptime: 138780 ms | maxWrittenKey: 100014 | maxGeneratedKey: 100014 |
```

## 测试结论
client数目|threads数目|OPS|时延
---|---|---|---
16|256|19.7w|1.3ms
16|128|17.2w|0.74ms
8|256|20.2w|1.27ms
8|128|19.5w|0.66ms
8|64|16w|0.4ms






