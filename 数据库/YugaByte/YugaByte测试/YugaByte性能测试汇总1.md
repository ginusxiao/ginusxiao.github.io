# 提纲
[toc]

## YugaByte noop read性能测试
### read响应中带伪造的数据
#### 在同一个节点上测试(yb-sample-test测试程序和yugabyte server处于同一个节点)
connection数目|threads数目|OPS|时延
---|---|---|---
8|64|16w|0.4ms
8|128|19.5w|0.66ms
8|256|20.2w|1.27ms
16|128|17.2w|0.74ms
16|256|19.7w|1.3ms


#### 在不同节点上测试(yb-sample-test测试程序和yugabyte server处于不同节点)
connection数目|threads数目|OPS|时延
---|---|---|---
8|128|30w|0.43ms
8|256|34w|0.76ms
16|128|30.5w|0.42ms
16|256|32.5w|0.79ms

#### 不同节点上测试情况下网络流量
![image](https://note.youdao.com/yws/public/resource/6f3352f863a6003aa3d542bd271c2fab/xmlnote/370174A3F0A04D8AA7402AEEC8B9D38A/93898)

#### 在不同节点上测试(yb-sample-test测试程序和yugabyte server处于不同节点)，增加reactor线程数目(通过修改--num_reactor_threads参数，从16增加到32)
connection数目|threads数目|OPS|时延
---|---|---|---
16|128|28w|0.46ms
16|256|31w|0.83ms

从结果来看，没有带来性能提升。

#### 在不同节点上测试(yb-sample-test测试程序和yugabyte server处于不同节点)，增加rpc thread pool线程数目(通过修改--rpc_workers_limit参数，默认为256)
##### 修改rpc_workers_limit为384
connection数目|threads数目|OPS|时延
---|---|---|---
16|128|28.4w|0.45ms
16|256|32.5w|0.78ms

##### 修改rpc_workers_limit为128
connection数目|threads数目|OPS|时延
---|---|---|---
16|128|28.6w|0.45ms
16|256|33.2w|0.77ms

##### 修改rpc_workers_limit为64
connection数目|threads数目|OPS|时延
---|---|---|---
16|128|29.2w|0.44ms
16|256|34.5w|0.74ms

从结果来看，没有带来性能提升。且无论rpc_workers_limit被设置为64,128,256还是384，性能都差不太多。

#### 在不同节点上测试(yb-sample-test测试程序和yugabyte server处于不同节点)，修改--cql_proxy_bind_address参数，从0.0.0.0:9042修改为10.10.10.10:9042
connection数目|threads数目|OPS|时延
---|---|---|---
16|128|28.5w|0.45ms
16|256|33w|0.77ms

从结果来看，没有带来性能提升。

### read响应中不带数据，只带状态
#### 在同一个节点上测试(yb-sample-test测试程序和yugabyte server处于同一个节点)
connection数目|threads数目|OPS|时延
---|---|---|---
8|128|20w|0.64ms
8|256|21.5w|1.2ms
16|128|18w|0.72ms
16|256|21w|1.23ms


#### 在不同节点上测试(yb-sample-test测试程序和yugabyte server处于不同节点)
connection数目|threads数目|OPS|时延
---|---|---|---
8|128|29.3w|0.44ms
8|256|33w|0.76ms
16|128|30w|0.42ms
16|256|34w|0.76ms

## YugaByte Unprepared vs Prepared vs Batch statement 
线程数|insert次数|Unprepared statement所需时间|Prepared statement所需时间|Batch statement所需时间
---|---|---|---|---
1|10w|97s|78s|3s(每200个insert组成一个batch)
1|100w|962s|653s|28s(每200个insert组成一个batch)

## YugaByte read执行cql解析，不执行read rpc(但依然执行Tablet lookup rpc)，直接执行read rpc回调
### 在同一个节点上测试(yb-sample-test测试程序和yugabyte server处于同一个节点)
connection数目|threads数目|OPS|时延
---|---|---|---
4|128|23w|0.56ms
4|256|23.5w|1.09ms
8|128|25.8w|0.5ms
8|256|26.6w|0.97ms
16|128|22.5w|0.57ms
16|256|26.2w|0.98ms
32|128|19.1w|0.67ms
32|256|24w|1.07ms

### 在不同节点上测试(yb-sample-test测试程序和yugabyte server处于不同节点)
connection数目|threads数目|OPS|时延
---|---|---|---
8|128|37.5w|0.34ms
8|256|43.5w|0.59ms
16|128|41w|0.31ms
16|256|47.5w|0.54ms

### 不同节点测试情况下，网络之间流量情况
![image](https://note.youdao.com/yws/public/resource/6f3352f863a6003aa3d542bd271c2fab/xmlnote/FAAF4C2A00804633B16623DA33A1AA81/93910)


## YugaByte read执行cql解析，在Tablet lookup rpc的回调TabletLookupFinished中直接返回响应

*说明：该测试程序，暂时有点问题，测试一段时间之后可能会报错。*

### 在同一个节点上测试(yb-sample-test测试程序和yugabyte server处于同一个节点)
connection数目|threads数目|OPS|时延
---|---|---|---
4|128|25.6|0.5ms
4|256|26.5w|0.98ms
8|128|28.9w|0.44ms
8|256|30w|0.85ms
8|384|30w|1.28ms
16|128|26w|0.49ms
16|256|30.2w|0.84ms

### 在不同节点上测试(yb-sample-test测试程序和yugabyte server处于不同节点)
connection数目|threads数目|OPS|时延
---|---|---|---
8|128|38w|0.34ms
8|256|47w|0.54ms
16|128|46w|0.28ms
16|256|50w|0.51ms

## 测试总结
### 不同阶段返回read响应的性能情况
- 正常测试，性能约为14w；
- noop read，响应中带假数据，最好性能约34w；
- noop read，响应中不带数据，只带状态的话，最好性能约34w；
- 不发送ReadRpc，在发送ReadRpc的地方直接调用ReadRpc::AsyncFinished，且响应中不带数据，只带状态的话，则最好性能约为47.5w；
- TabletLookupFinished结束之后就返回响应，且响应中只带状态，则最好性能约50w；
- 在CQLProcessor::ProcessRequest中返回响应，且响应中只带状态，则最好性能约65w；

从上可知：
- 在CQLProcessor::ProcessRequest到发送ReadRpc这个过程中，损失了18w左右的性能；
- 在发送ReadRpc到接收到ReadRpc的回调这个过程中，损失了14w左右的性能；
- 在TabletServer中处理ReadRpc到发送ReadRpc响应这个过程中，损失了约20w左右的性能；


## 待进一步探寻
1. 尝试确保CQLServer发送给TServer的RPC请求的回调是在发送RPC请求的线程中执行的，而不是在其它线程中执行，这样的话，当一个请求被CQLServer rpc thread pool线程处理的时候，则如下流程都是在当前线程执行的：CQLServiceImpl::Handle -> Executor::Execute -> LookupTabletByKey -> TabletLookupFinished -> Executor::FlushAsync -> Batcher::FlushBuffersIfReady -> Create ReadRpc -> Send ReadRpc -> ReadRpc::Finished -> Batcher::CheckForFinishedFlush -> Batcher::RunCallback -> Executor::FlushAsyncDone -> Executor::StatementExecuted -> CQLProcessor::StatementExecuted -> CQLProcessor::PrepareAndSendResponse -> CQLProcessor::SendResponse。
2. Messenger中的RPC ThreadPool中的任务队列是否存在性能问题？
3. 找出“测试总结”中出现的性能损失到底损失在哪里？
4. yugabyte线程cpu绑定
5. 
