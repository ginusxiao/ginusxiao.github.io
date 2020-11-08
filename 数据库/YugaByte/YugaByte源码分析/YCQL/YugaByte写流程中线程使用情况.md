# 提纲
[toc]

## 前言
下面的分析以单个节点组成的测试环境中的写请求为例进行。

## RPC相关
下面的分析会着重分析CQLServer，TServer，RedisServer等组件（下面以${component_name}代表），每个组件都会自己的RPCServer，因此也都具有自己的Messenger。

### Messenger中的acceptor线程
该类型的线程对应于Messenger::acceptor_，一个Messenger只有一个acceptor线程。

该线程主要用于监听连接请求，对于一个新的连接，会从Messenger::reactors_中为其分配一个Reactor，然后将这个新的连接对应的Socket注册到该Reactor中。

### Messenger中的${component_name}_reactor线程
该类型的线程对应于Messenger::reactors_，这是一个线程集合，但不是一个线程池。该线程集合的默认大小是min{16, 节点中的CPU的个数}，可以通过FLAGS_num_reactor_threads进行设置。

在reactor线程中有一个pending_tasks_，它当中存放着所有等待reactor线程处理的ReactorTask，对于这些处于pending_tasks_中的每一个ReactorTask，reactor线程会调用ReactorTask::Run()来执行该Task。

### Messenger中的iotp_${component_name}_${index}线程池中的线程
该线程池对应于Messenger::io_thread_pool_，类型为yb::rpc::IoThreadPool，该线程池中最大线程数目默认为4，可以通过io_thread_pool_size参数设置.

### Messenger中的rpc_tp_${component_name}_${index}线程池中的线程
该线程池对应于Messenger::normal_thread_pool_，类型为yb::rpc::ThreadPool，该线程池中最大线程数目默认为256，可以通过rpc_workers_limit参数设置.

### Messenger中的rpc_tp_${component_name}-high-pri_${index}线程池中的线程
该线程池对应于Messenger::high_priority_thread_pool_，类型为yb::rpc::ThreadPool，该线程池中最大线程数目是直接通过Messenger::normal_thread_pool_的最大线程数目进行设置的，因此该线程池中最大线程数目默认为256，可以通过rpc_workers_limit参数设置.

### RPC ServicePool中的用于提供RPC服务的线程池
该线程池对应于ServicePoolImpl::thread_pool_，类型为yb::rpc::ThreadPool，**但是该线程池不是ServicePool中自有的线程池，而是，复用Messenger中的线程池，要么是Messenger::normal_thread_pool_，要么是Messenger::high_priority_thread_pool_**。

### RPC请求从发送到接收到响应的整个过程
#### Client端发送RPC请求的过程
Client端发送RPC请求，都会借助于Proxy::DoAsyncRequest，它会根据是否是本地调用分别处理：
- 如果是本地调用：
    - 生成一个LocalOutboundCall；
    - 将LocalOutboundCall转换为LocalYBInboundCall；
    - **如果允许在当前线程执行本地rpc调用，且当前线程是RPC线程**：
        - **直接在当前线程处理，见Messenger::Handle -> ServicePool::Handle -> ServicePoolImpl::Handle -> 真实的RPC Service的Handle方法**。
    - 否则：
        - 添加到LocalInboundCall的请求队列中Messenger::QueueInboundCall -> ServicePool::QueueInboundCall -> ServicePoolImpl::Enqueue -> **添加到RPC ServicePool中的用于提供RPC服务的线程池。**
        - 当LocalInboundCall被线程池中的线程调度执行时，ServicePoolImpl::Handle将被执行，然后进一步的会调用真实的RPC Service的Handle方法。
- 如果是一个远程调用：
    - 则生成一个OutboundCall；
    - 将OutboundCall添加到OutboundCall队列Proxy::QueueCall -> Messenger::QueueOutboundCall -> 根据调用的目标选择一个Reactor，并添加到该Reactor对应的OutboundCall队列中Reactor::QueueOutboundCall；
    - **如果没有Reactor::process_outbound_queue_task_任务在运行，则通过Reactor::ScheduleReactorTask来提交一个名为Connection::process_outbound_queue_task_的任务**。该任务的运行主体是Reactor::ProcessOutboundQueue，它的执行逻辑如下：
        - 遍历每一个OutboundCall，执行：
            - 为当前的OutboundCall创建或者查找到一个连接；
            - 将当前的OutboundCall添加到该连接的待发送的OutboundData集合中；
            - 将当前的OutboundCall对应的连接添加到待处理的Connection集合processing_connections_中；
        - 对processing_connections_集合中的Connection进行排序；
        - 依次处理processing_connections_中的每一个Connection，执行：
            - 通知该Connection中有待处理的OutboundCall，见Connection::OutboundQueued -> TcpStream::TryWrite -> TcpStream::DoWrite -> Socket::WriteV。

**综上，如果是一个远程调用，RPC请求的发送还是通过Reactor线程处理的；如果是一个本地调用，则可能直接在当前线程执行，也可能是交由ServicePool中用于提供RPC服务的线程池来处理。**

#### Server端接收到RPC请求时的处理
**Server端接收到RPC请求时，会首先在Messenger的Reactor线程中处理**。以CQLServer为例，关于某个连接上的Read事件到达时，对应的Reactor的处理逻辑在TcpStream::Handler -> TcpStream::ReadHandler -> TcpStream::TryProcessReceived -> yb::rpc::Connection::ProcessReceived -> yb::cqlserver::CQLConnectionContext::ProcessCalls -> yb::rpc::BinaryCallParser::Parse -> yb::cqlserver::CQLConnectionContext::HandleCall -> yb::rpc::Messenger::QueueInboundCall -> yb::rpc::ServicePool::QueueInboundCall -> yb::rpc::ServicePoolImpl::Enqueue -> **将InboundCall提交给ServicePoolImpl的线程池**，因为InboundCall中包含一个类型为ThreadPoolTask的成员，所以当该InboundCall被该线程池调度给某个线程执行时，对应的ThreadPoolTask会被执行。

#### Server端发送RPC响应的过程
**Server端接收到RPC响应时，会在Messenger的Reactor线程中处理**。以CQLServer为例，当请求处理完成之后，处理线程（不一定是Reactor线程，但有可能是Reactor线程）会调用ConnectionContext::QueueResponse将响应添加到对应的Connection中，并等待处理，如果当前线程就是该Connection对应的Reactor线程，则直接在当前线程处理，否则向该Connection对应的Reactor线程发送一个事件，通知它有数据待处理。以CQLServer端CQLInboundCall的响应为例，当CQLInboundCall执行成功之后会依次调用：CQLInboundCall::RespondSuccess -> InboundCall::QueueResponse -> ConnectionContextWithCallId::QueueResponse -> Connection::QueueOutboundData(这里的OutboundData实际上是InboundCall，类继承关系为InboundCall -> RpcCall -> OutboundData)。在Connection::QueueOutboundData中，会判断当前线程是否就是Connection对应的Reactor的线程，并根据判断结果分别处理：
- 如果当前线程就是Connection对应的Reactor的线程，则直接调用Connection::DoQueueOutboundData -> Connection::OutboundQueued -> TcpStream::TryWrite -> TcpStream::DoWrite -> Socket::WriteV调用链。
- 否则，
    - 首先将OutboundData添加到Connection的OutboundData数组中(见Connection::outbound_data_to_process_)；
    - **检查当前是否有用于处理Connection中OutboundData数组中的OutboundData的任务正在运行，如果没有，则向Reactor线程提交一个任务**(通过Reactor::ScheduleReactorTask来提交一个名为Connection::process_response_queue_task_的任务，Reactor::ScheduleReactorTask会向Reactor::loop_主循环发送一个异步事件，进而由Reactor::loop_主循环，也就是Reactor线程来执行该任务)，**用于处理OutboundData数组中的OutboundData**。该任务的运行主体是Connection::ProcessResponseQueue，它会依次遍历Connection::outbound_data_to_process_中的每一个OutboundData，在当前上下文中也就是每一个InboundCall，对每一个InboundCall执行Connection::DoQueueOutboundData -> Connection::OutboundQueued -> TcpStream::TryWrite -> TcpStream::DoWrite -> Socket::WriteV调用链。

#### Client端接收RPC响应的过程
**Client端接收RPC响应时，会首先在Messenger的Reactor线程中处理**。以CQLServer为例，当接收到RPC响应的时候，会触发某个连接上的Read事件，对应的Reactor的处理逻辑在TcpStream::Handler -> TcpStream::ReadHandler -> TcpStream::TryProcessReceived -> yb::rpc::Connection::ProcessReceived -> yb::rpc::YBOutboundConnectionContext::ProcessCalls -> yb::rpc::BinaryCallParser::Parse -> YBOutboundConnectionContext::HandleCall -> Connection::HandleCallResponse -> 从Connection::awaiting_response_中找到对应的OutboundCall，并设置OutboundCall的Response，见OutboundCall::SetResponse -> OutboundCall::InvokeCallback，在OutboundCall::InvokeCallback中会根据OutboundCall对应的callback_thread_pool_(目前对应的是Messenger::normal_thread_pool_)是否为空来区别处理：
- 如果callback_thread_pool_不为空：
    - **将类型为InvokeCallbackTask的OutboundCall::callback_task_任务添加到callback_thread_pool_中**；
    - **当InvokeCallbackTask任务被线程池调度执行之后，会运行InvokeCallbackTask::Run -> OutboundCall::InvokeCallbackSync -> OutboundCall::callback_**；
    - 那么哪些情况下callback_thread_pool_会被设置呢(见Proxy::DoAsyncRequest)：
        - 如果是LocalOutboundCall，则callback_thread_pool_为null；
        - 如果不是LocalOutboundCall，且在调用Proxy::DoAsyncRequest时，force_run_callback_on_reactor参数被设置，则callback_thread_pool_为null；
        - 如果不是LocalOutboundCall，且在调用Proxy::DoAsyncRequest时，RpcController的invoke_callback_mode被设置为InvokeCallbackMode::kReactorThread，则callback_thread_pool_为null；
        - 否则callback_thread_pool_不为null；
- 否则：
    - 接在当前线程中执行InvokeCallbackSync -> OutboundCall::callback_；

## CQLServer
关于CQLServer中线程情况的分析是根据CQLServiceImpl::Handle开始进行分析。

### CQLServer中处理写请求
CQLServiceImpl::Handle -> CQLProcessor::ProcessCall -> CQLProcessor::ProcessRequest -> QLProcessor::RunAsync -> 先通过QLProcessor::Prepare进行parse和analyse，生成ParseTree，然后调用QLProcessor::ExecuteAsync执行该ParseTree，QLProcessor::ExecuteAsync -> Executor::ExecuteAsync -> 依次执行Executor::Execute和Executor::FlushAsync。下面分别分析Executor::Execute和Executor::FlushAsync。

Executor::Execute执行流程如下：
```
Executor::ExecuteAsync
    - Executor::Execute
        - Executor::ExecTreeNode
            # 将当前的TreeNode添加到Executor::exec_context_中，返回对应的TnodeContext
            - tnode_context = exec_context_->AddTnode(tnode)
            # 对insert语句(类型为TreeNodeOpcode::kPTInsertStmt)的处理
            - ExecPTNode(static_cast<const PTInsertStmt *>(tnode), tnode_context)
                - 生成一个YBqlWriteOp，并获取其中的write request protobuf
                  YBqlWriteOpPtr insert_op(table->NewQLInsert())
                  QLWriteRequestPB *req = insert_op->mutable_request()
                - 根据tnode信息填充write request protobuf
                  TtlToPB(tnode, req)
                  TimestampToPB(tnode, req)
                  ColumnArgsToPB(tnode, req)
                  ColumnRefsToPB(tnode, req->mutable_column_refs())
                  insert_op->set_writes_static_row(tnode->ModifiesStaticRow())
                  insert_op->set_writes_primary_row(tnode->ModifiesPrimaryRow())
                - Executor::AddOperation(insert_op, tnode_context)
                    - TnodeContext::AddOperation
                    # 将当前的写操作添加到Executor::write_batch_中
                    - write_batch_.Add(op)
                        - 对当前的更新请求作如下处理：
                            - 如果当前操作会读primary key，且当前的write_batch_中已经有一个关于相同key的写操作，
                            则返回false，表明当前操作和另一个操作之间存在冲突；
                            - 如果当前操作会读hash key，且当前的write_batch_中已经有一个关于相同key的写操作，
                            则返回false，表明当前操作和另一个操作之间存在冲突；
                            - 否则：
                                - 如果当前的write_batch_需要读primary key，则将之添加到ops_by_primary_key_中；
                                - 如果当前write_batch_需要读hash key，则将之添加到ops_by_hash_key_中；
                                - 返回true，表明可以添加到当前的write_batch_
                    - session->Apply(op)
                        - YBSession::batcher_.Add
                            - 首先生成一个InFlightOp，InFlightOp表示一个操作已经添加到Batcher中，
                            但是尚未执行完成
                              auto in_flight_op = std::make_shared<InFlightOp>(yb_op);
                            - yb::client::internal::Batcher::Add
                                # 将当前in_flight_op添加到InFlightOp集合(Batcher::ops_，
                                # 表示正处于InFlightOpState::kLookingUpTablet状态的操作集合)中，
                                # 并增加正在查找tablet的in_flight_op的数目(++outstanding_lookups_)
                                - AddInFlightOp(in_flight_op)
                                # 在MetaCache::LookupTabletByKey中会传递一个回调Batcher::TabletLookupFinished，
                                # 当查找到Tablet之后，该回调会被调用
                                - yb::client::internal::MetaCache::LookupTabletByKey
                                    # 首先尝试在本地的MetaCache中进行查找，如果在本地MetaCache中成功找到，
                                    # 则会调用回调Batcher::TabletLookupFinished
                                    - FastLookupTabletByKeyUnlocked
                                        - Batcher::TabletLookupFinished
                                    # !!!!如果前一步骤中没有查找到，则发送RPC消息***!!!!
                                    # 当接收到RPC响应之后，回调Batcher::TabletLookupFinished会被调用
                                    - rpc::StartRpc<LookupByKeyRpc>(...)
                                        - LookupByKeyRpc::DoSendRpc
                                            # 发送消息给Master，获取Tablet信息
                                            - master_proxy()->GetTableLocationsAsync
```
从Executor::ExecuteAsync -> Executor::Execute逻辑分析来看，最终会将ParseTree转换成YBOperation，并通过Batcher::Add添加到Batcher中，在Batcher::Add中除了将YBOperation添加到到InFlightOp集合(Batcher::ops_)之外，还会为每一个YBOperation查找对应的Tablet，查找Tablet可能直接在本地的MetaCache中找到，也可能必须发送RPC请求到Master中去查找，如果在本地MetaCache中查找，则查找仍然是在当前线程进行，**如果通过RPC请求去Master中查找，则会产生由于RPC相关的线程切换**。当查找到对应的Tablet之后，回调方法Batcher::TabletLookupFinished会被调用。

如果是在本地MetaCache中查找到了Tablet，则会在执行了Batcher::TabletLookupFinished之后，Executor::ExecuteAsync -> Executor::Execute逻辑就返回，如果是通过RPC请求去Master中查找Tablet，则Executor::ExecuteAsync -> Executor::Execute逻辑在发送出去该RPC请求之后就返回。当Executor::ExecuteAsync -> Executor::Execute返回之后，接着会进入Executor::ExecuteAsync -> Executor::FlushAsync逻辑。

Executor::ExecuteAsync -> Executor::FlushAsync逻辑的执行如下：
```
Executor::ExecuteAsync
    - Executor::FlushAsync
        # YBSession::FlushAsync里面会传递一个回调Executor::FlushAsyncDone
        - YBSession::FlushAsync
            # 将当前的batcher_赋值给old_batcher
            - old_batcher.swap(batcher_)
            # 添加到Executor::flushed_batchers_中
            - flushed_batchers_.insert(old_batcher)
            # 如果Batcher::ops_不为空，则执行Batcher::FlushAsync
            # 回调Executor::FlushAsyncDone会进一步传递到Batcher::FlushAsync
            - Batcher::FlushAsync
                # 修改Batcher的状态为state_ = BatcherState::kResolvingTablets
                - state_ = BatcherState::kResolvingTablets
                # 设置Batcher::flush_callback_为回调Executor::FlushAsyncDone
                - flush_callback_ = callback
                - Batcher::CheckForFinishedFlush
                    - 检查如果!ops_.empty()，则表明Batcher::ops_中有尚未执行完成(
                    即没有接收到RPC响应)的Operation，则直接返回，也就是说这个Batcher
                    还没有Finished
                    - 否则，
                        # 修改Batcher的状态为BatcherState::kComplete
                        - state_ = BatcherState::kComplete
                        - YBSession::FlushFinished
                            # 从YBSession::flushed_batchers_中移除当前的Batcher
                            - flushed_batchers_.erase(batcher)
                - Batcher::FlushBuffersIfReady
                    - 检查outstanding_lookups_是否为0，如果不为0，则表明还有Operation
                    正在查找Tablet过程中，则直接返回
                    - 如果Batcher当前的状态不是BatcherState::kResolvingTablets，则直接返回
                    - 否则，
                        - 对yb::client::internal::Batcher::ops_queue_中的所有operation进行排序，
                        排序的依据是：首先比较tablet id，然后比较操作类型（分为3中类型：OpGroup::kWrite，
                        OpGroup::kConsistentPrefixRead和OpGroup::kLeaderRead），最后比较操作的sequence_number_；
                        - 然后执行这些操作：Batcher::ExecuteOperations
                            - 如果是在事务上下文中，则先执行YBTransaction::Prepare
                            - 然后遍历yb::client::internal::Batcher::ops_queue_中的所有的操作，
                            关于相同tablet的相同操作类型（OpGroup类型）的操作聚合在一起生成一个
                            RPC请求（类型为AsyncRpc，继承自TabletRpc），如果当前的tablet和前一个
                            tablet发生变更，或者当前OpGroup和前一个OpGroup发生变更，则生成一个
                            新的RPC请求(根据OpGroup类型不同，生成的RPC请求也不同，可能的RPC请求
                            类型分别为WriteRpc，ReadRpc)，生成的RPC请求存放在rpcs数组中；
                            - 清除yb::client::internal::Batcher::ops_queue_中所有的操作；
                            - 遍历rpcs数组中所有的RPC请求，并借助AsyncRpc::SendRpc发送；
                                - tablet_invoker_.Execute(std::string(), num_attempts() > 1)
                                    # 按照一定的算法选择一个目标TServer，优先选择Tablet Leader所在的TServer，
                                    # 如果没有这样的TServer，则从Tablet Replica中选择一个
                                    - TabletInvoker::SelectTabletServer
                                    # 以WriteRpc为例，这里调用的是AsyncRpc::SendRpcToTserver
                                    - rpc_->SendRpcToTserver(retrier_->attempt_num())
                                        - WriteRpc::CallRemoteMethod
                                            # tablet_invoker_.proxy()类型为TabletServerServiceProxy
                                            # 将RPC请求发送给TabletServer，在TServer端处理的RPC Service是TabletServiceImpl
                                            # 当接收到RPC请求的响应的时候WriteRpc::Finished会被调用，
                                            # 如果RPC调用是远程调用，则是异步的，所以Executor::Execute至此就返回了，
                                            # 如果RPC调用是本地调用，则当前线程就是RPC线程，所以会在该线程中处理
                                            - tablet_invoker_.proxy()->WriteAsync(
                                              req_, &resp_, PrepareController(),
                                              std::bind(&WriteRpc::Finished, this, Status::OK()))
```
综上分析在Executor::ExecuteAsync -> Executor::Execute + Executor::FlushAsync过程中，先执行Executor::ExecuteAsync -> Executor::Execute，后执行Executor::ExecuteAsync -> Executor::FlushAsync。在Executor::ExecuteAsync -> Executor::Execute中会将Operation添加到Batcher中，见Batcher::Add -> Batcher::AddInFlightOp会将Operation添加到当前的Batcher中，且当前Batcher处于BatcherState::kGatheringOps状态；在执行了Executor::ExecuteAsync -> Executor::Execute之后，接着马上执行Executor::ExecuteAsync -> Executor::FlushAsync，在Executor::ExecuteAsync -> Executor::FlushAsync -> YBSession::FlushAsync -> Batcher::FlushAsync中会将Batcher修改为BatcherState::kResolvingTablets状态，这样就不能再向当前Batcher添加新的Operation了(因为只有Batcher处于BatcherState::kGatheringOps状态时才可以添加新的Operation)，**这样以来，每一个Batcher中只包含一个Operation？**

假设Executor::ExecuteAsync -> Executor::FlushAsync -> YBSession::FlushAsync -> Batcher::FlushAsync -> Batcher::FlushBuffersIfReady中Batcher::outstanding_lookups_不为0，即当前Batcher中还有Operation还在查找Tablet的过程中，则不会将该Batcher中的Operations生成RPC请求发送出去，那么这些Operations又是在哪里生成RPC请求并发送出去的呢？答案是在Batcher::TabletLookupFinished，它会检查Batcher::outstanding_lookups_是否为0，如果是0，它会执行Batcher::FlushBuffersIfReady。

在Executor::ExecuteAsync -> Executor::FlushAsync -> YBSession::FlushAsync -> Batcher::FlushAsync -> Batcher::CheckForFinishedFlush中，如果检测到Batcher中所有Operations都执行完毕，则会将对应的Batcher从YBSession::flushed_batchers_中移除。

在Batcher::TabletLookupFinished被调用的时候，Batcher可能处于的状态为：BatcherState::kGatheringOps状态或者BatcherState::kResolvingTablets状态。
```                                            
# 当成功查找到Tablet信息之后，则调用TabletLookupFinished处理当前
# 完成的的InFlightOp
- TabletLookupFinished(std::move(in_flight_op), yb_op->tablet())
    - 减少正在查找tablet的in_flight_op的数目(--outstanding_lookups_)
    - 将当前的InFlightOp的状态从InFlightOpState::kLookingUpTablet转换
      为InFlightOpState::kBufferedToTabletServer
    - 将当前的InFlightOp添加到yb::client::internal::Batcher::ops_queue_中，
      ops_queue_是一个关于InFlightOp的数组
    - 如果正在进行tablet查找的操作的数目为0，即outstanding_lookups_为0，
      则执行Batcher::FlushBuffersIfReady
        - 首先对yb::client::internal::Batcher::ops_queue_中的所有operation进行排序，
          排序的依据是：首先比较tablet id，然后比较操作类型（分为3中类型：OpGroup::kWrite，
          OpGroup::kConsistentPrefixRead和OpGroup::kLeaderRead），最后比较操作的sequence_number_；
        - 然后执行这些操作：Batcher::ExecuteOperations
            - 如果是在事务上下文中，则先执行YBTransaction::Prepare
            - 然后遍历yb::client::internal::Batcher::ops_queue_中的所有的操作，
            关于相同tablet的相同操作类型（OpGroup类型）的操作聚合在一起生成一个
            RPC请求（类型为AsyncRpc，继承自TabletRpc），如果当前的tablet和前一个
            tablet发生变更，或者当前OpGroup和前一个OpGroup发生变更，则生成一个
            新的RPC请求(根据OpGroup类型不同，生成的RPC请求也不同，可能的RPC请求
            类型分别为WriteRpc，ReadRpc)，生成的RPC请求存放在rpcs数组中；
            - 清除yb::client::internal::Batcher::ops_queue_中所有的操作；
            - 遍历rpcs数组中所有的RPC请求，并借助AsyncRpc::SendRpc发送；
                - tablet_invoker_.Execute(std::string(), num_attempts() > 1)
                    # 按照一定的算法选择一个目标TServer，优先选择Tablet Leader所在的TServer，
                    # 如果没有这样的TServer，则从Tablet Replica中选择一个
                    - TabletInvoker::SelectTabletServer
                    # 以WriteRpc为例，这里调用的是AsyncRpc::SendRpcToTserver
                    - rpc_->SendRpcToTserver(retrier_->attempt_num())
                        - WriteRpc::CallRemoteMethod
                            # tablet_invoker_.proxy()类型为TabletServerServiceProxy
                            # 将RPC请求发送给TabletServer，在TServer端处理的RPC Service是TabletServiceImpl
                            # 当接收到RPC请求的响应的时候WriteRpc::Finished会被调用，
                            # 如果RPC调用是远程调用，则是异步的，所以Executor::Execute至此就返回了，
                            # 如果RPC调用是本地调用，则当前线程就是RPC线程，所以会在该线程中处理
                            - tablet_invoker_.proxy()->WriteAsync(
                              req_, &resp_, PrepareController(),
                              std::bind(&WriteRpc::Finished, this, Status::OK()))
    - 如果正在进行tablet查找的操作的数目不为0（记录在outstanding_lookups_中），则直接返回
```       
假如是本地调用，则TabletLookupFinished会在当前线程被调用，因而也可能进一步调用FlushBuffersIfReady，但是跟踪调用栈并没有发现TabletLookupFinished -> FlushBuffersIfReady，这是为什么呢？因为FlushBuffersIfReady中会检测当前Batcher的状态是否为 BatcherState::kResolvingTablets，如果不是，则直接退出。这也是为什么在本地调用的情况下，在火焰图分析中并没有看到TabletLookupFinished -> FlushBuffersIfReady这一个调用关系的原因。

**在发送RPC调用给TServer时，如果是远程调用，则会发生由RPC调用所引起的线程切换，当然，如果是本地调用，则会直接在当前线程中处理**。

当接收到来自TServer的关于Write RPC请求的响应的时候WriteRpc::Finished会被调用，调用流程如下：
```
AsyncRpc::Finished
    - tablet_invoker_.Done(&new_status)
    - ProcessResponseFromTserver(new_status)
        - WriteRpc::ProcessResponseFromTserver
            - batcher_->ProcessWriteResponse(*this, status)
            # 将RPC请求的request映射到该RPC请求所关联的每个操作的request（在前面
            分析中提到关于相同tablet的相同操作类型会生成一个RPC请求，也就是说一个
            RPC请求可能会对应多个操作），将RPC请求的response映射到该RPC请求所关联
            的每个操作的response
            - SwapRequestsAndResponses(false)
    # 从Batcher::ops_集合中移除所有的InFlightOp，表示这些InFlightOp已经处理完成了
    - batcher_->RemoveInFlightOpsAfterFlushing(ops_, new_status, MakeFlushExtraResult())
    - batcher_->CheckForFinishedFlush()
        - 检查如果!ops_.empty()，则表明Batcher::ops_中有尚未执行完成(
        即没有接收到RPC响应)的Operation，则直接返回，也就是说这个Batcher
        还没有Finished
        - 否则，
            # 修改Batcher的状态为BatcherState::kComplete
            - state_ = BatcherState::kComplete
            - YBSession::FlushFinished
                # 从YBSession::flushed_batchers_中移除当前的Batcher
                - flushed_batchers_.erase(batcher)
            # 在当前线程或者Callback ThreadPool中执行Batcher::flush_callback_
            # 实际的Callback是在YBSession::FlushAsync中设置的Executor::FlushAsyncDone
            - Batcher::RunCallback
                # Executor::FlushAsyncDone中暂无我们需要关注的
                - Executor::FlushAsyncDone
```
**发送给TServer的关于Write RPC请求的回调被触发之后，可能会在当前线程或者Callback ThreadPool中执行Batcher::flush_callback_回调，即Executor::FlushAsyncDone。**

Executor::FlushAsyncDone的调用关系如下：
```
Executor::FlushAsyncDone
    - Executor::ProcessAsyncResults
        - Executor::FlushAsync
            # 如果对应的Batcher::ops_还存在Operation，则将session_添加到待
            # flush的session集合@flush_sessions中
            - if (session_->CountBufferedOperations() > 0) {
                flush_sessions.push_back({session_, nullptr});
              }
            - ...
            # 只有在flush_sessions和commit_contexts均未空的情况下才会执行下面的语句
            - if (flush_sessions.empty() && commit_contexts.empty()) {
                Executor::StatementExecuted(Status::OK())
              }
              
Executor::StatementExecuted
    # 获取Executor中的statement executed callback，并执行之
    # 该callback是在QLProcessor::RunAsync -> QLProcessor::ExecuteAsync ->
    # Executor::ExecuteAsync中设置的，最终执行的callback是在QLProcessor
    # 的构造方法中传递进来的CQLProcessor::StatementExecuted
    - StatementExecutedCallback cb = std::move(cb_);
      Reset();
      cb.Run(s, result);
```
在Executor::FlushAsyncDone中会调用Executor::StatementExecuted，最终会调用QLProcessor的构造方法中传递进来的CQLProcessor::StatementExecuted。

CQLProcessor::StatementExecuted的调用关系如下：
```
CQLProcessor::StatementExecuted
    # 如果Status OK，则调用ProcessResult来生成CQLResponse，
    # 否则调用ProcessError来生成CQLResponse
    - unique_ptr<CQLResponse> response(s.ok() ? ProcessResult(result) : ProcessError(s));
    # 发送响应
    - CQLProcessor::PrepareAndSendResponse
        - CQLProcessor::SendResponse
            - CQLInboundCall::RespondSuccess
                - InboundCall::QueueResponse
```
在CQLProcessor::StatementExecuted中会发送响应。

#### 释疑
1. 在QLProcessor::RunAsync中有一个回调StatementExecutedCallback，是在何时被调用的？

QLProcessor::RunAsync中的回调StatementExecutedCallback(记为Callback1)会被QLProcessor::ExecuteAsync中的回调StatementExecutedCallback(记为Callback2)封装，当Executor::StatementExecuted中被调用时，Callback2会被调用，然后进一步调用Callback1。

那么在返回成功的情况下，Executor::StatementExecuted在哪里被调用的呢？见Executor::FlushAsync中设置的回调Executor::FlushAsyncDone -> Executor::ProcessAsyncResults -> Executor::FlushAsync -> Executor::StatementExecuted.

2. 在QLProcessor::ExecuteAsync中有一个回调StatementExecutedCallback，是在何时被调用的？

QLProcessor::ExecuteAsync中的回调StatementExecutedCallback(记为Callback1)是对QLProcessor::RunAsync中的回调StatementExecutedCallback(记为Callback2)的封装，Callback1最终会在Executor::StatementExecuted中被调用，Callback1进一步会调用Callback2。

那么在返回成功的情况下，Executor::StatementExecuted在哪里被调用的呢？见Executor::FlushAsync中设置的回调Executor::FlushAsyncDone -> Executor::ProcessAsyncResults -> Executor::FlushAsync -> Executor::StatementExecuted.

3. 在Executor::ExecuteAsync中每当执行一个新的ParseTree的时候，都会检查Executor::cb_为空，否则表明正在执行另一个ParseTree，这个Executor::cb_的生命周期是怎样的？

```
- Executor::ExecuteAsync
    # 根据参数中传递进来的StatementExecutedCallback来设置Executor::cb_
    - cb_ = std::move(cb)
    
- Executor::StatementExecuted
    - StatementExecutedCallback cb = std::move(cb_)
    - Executor::Reset
        # 将cb_的bind_state_设置为null
        - cb_.Reset()
    # 运行Executor::cb_
    - cb.Run
```

### CQLServer发送写请求的响应
CQLProcessor::ProcessRequest在处理QueryRequest的时候，会传递给QLProcessor::RunAsync一个回调方法CQLProcessor::StatementExecuted，当Executor::FlushAsyncDone被调用之后，会进一步调Executor::StatementExecuted，进入发送写请求的响应的过程。

Executor::StatementExecuted调用过程如下：
```
Executor::StatementExecuted
    # 获取Executor中的statement executed callback，并执行之
    # 该callback是在QLProcessor::RunAsync -> QLProcessor::ExecuteAsync ->
    # Executor::ExecuteAsync中设置的，最终执行的callback是在QLProcessor
    # 的构造方法中传递进来的CQLProcessor::StatementExecuted
    - StatementExecutedCallback cb = std::move(cb_);
      Reset();
      cb.Run(s, result);
```
Executor::StatementExecuted最终会调用QLProcessor的构造方法中传递进来的CQLProcessor::StatementExecuted。

CQLProcessor::StatementExecuted的调用关系如下：
```
CQLProcessor::StatementExecuted
    # 如果Status OK，则调用ProcessResult来生成CQLResponse，
    # 否则调用ProcessError来生成CQLResponse
    - unique_ptr<CQLResponse> response(s.ok() ? ProcessResult(result) : ProcessError(s));
    # 发送响应
    - CQLProcessor::PrepareAndSendResponse
        - CQLProcessor::SendResponse
            - CQLInboundCall::RespondSuccess
                - InboundCall::QueueResponse
```
在CQLProcessor::StatementExecuted中会发送响应。

## TServer
### TServer中的主要线程池和线程简介
在TServer中主要包含以下线程池：
- TSTabletManager中的名为prepare线程池
- TSTabletManager中的名为append线程池
- TSTabletManager中的名为raft的线程池
- TSTabletManager中的名为apply线程池
- TSTabletManager中的tablet-bootstrap线程池
- TSTabletManager中的名为read-parallel线程池

这些线程池的类型都是ThreadPool。

另外TServer中还包括一个BackgroundTask线程：
- TSTabletManager中的名为flush scheduler bgtask的BackgroundTask


### TSTabletManager中的名为prepare线程池
对应于TSTabletManager::tablet_prepare_pool_，线程池类型为ThreadPool(注意：跟前面Messenger中的rpc::ThreadPool不同)，在TSTabletManager::TSTabletManager中创建：
```
TSTabletManager::TSTabletManager(...) {
    ...
    
    CHECK_OK(ThreadPoolBuilder("prepare")
               .unlimited_threads()
               .Build(&tablet_prepare_pool_));
        
    ...
}
```

**该线程池中的线程数目无上限。**

#### prepare线程池中的线程的使用
该线程池中的线程用于批量的将Operation提交给Raft进行复制，该线程池被所有的Tablet所共享。

每当创建/打开一个Tablet的时候，会初始化对应的TabletPeer，见TSTabletManager::OpenTablet：
```
TSTabletManager::OpenTablet
    # 这里的TSTabletManager::tablet_prepare_pool()就是Prepare线程池
    - TabletPeer::InitTabletPeer(..., TSTabletManager::tablet_prepare_pool(), ...)
        # 参数中的tablet_prepare_pool就是Prepare线程池，
        # 这里会创建TabletPeer::prepare_thread_，从名字上看，TabletPeer::prepare_thread_
        # 看似是一个线程，但实际上它不是一个线程，它的实现则依赖于PreparerImpl
        - prepare_thread_ = std::make_unique<Preparer>(consensus_.get(), tablet_prepare_pool)
            - PreparerImpl::PreparerImpl(..., tablet_prepare_pool)
                # 通过传递进来的prepare线程池创建一个ThreadPoolToken并
                # 赋值给PreparerImpl::tablet_prepare_pool_token_
                - tablet_prepare_pool_token_ = tablet_prepare_pool
                     ->NewToken(ThreadPool::ExecutionMode::SERIAL))
        - prepare_thread_->Start
            - Preparer::impl_->Start
                # 该方法直接返回Status::OK()
                - PreparerImpl::Start
```
可见，则初始化TabletPeer的时候，会为之创建一个Preparer(TabletPeer::prepare_thread_)，这个Preparer的实现则依赖于PreparerImpl，PreparerImpl主要包含如下成员和接口：
```
class PreparerImpl {
public：
    # 提交OperationDriver到queue_中
    CHECKED_STATUS Submit(OperationDriver* operation_driver);

private：
    using OperationDrivers = std::vector<OperationDriver*>;
    consensus::Consensus* const consensus_;
    
    # 通过Submit方法提交进来的待处理的OperationDriver
    MPSCQueue<OperationDriver> queue_;
    
    # 当prepare线程池中的线程处理PreparerImpl::queue_中的OperationDriver时，
    # 会按照一定的规则进行批量处理，同一批的OperationDriver都存放在这里，
    # 然后同一批的OperationDriver会尽量批量进行raft复制
    OperationDrivers leader_side_batch_;
    
    # PreparerImpl，进一步的说是Preparer，会借助于该ThreadPoolToken
    # 来向prepare线程中提交任务来执行PreparerImpl::queue_中的OperationDriver，
    # ThreadPoolToken可以认为是关于一个线程池的通行证
    std::unique_ptr<ThreadPoolToken> tablet_prepare_pool_token_;
    
    # 进行raft复制的时候，同一批批量进行raft复制的OperationDriver对应的
    # ConsensusRound都保存在这里
    consensus::ConsensusRounds rounds_to_replicate_;
}
```

那么TabletPeer::prepare_thread_是如何被使用的呢？且看下面的调用过程(以写流程为例)：
```
# 这里的方法参数中会传递进来WriteRequestPB，WriteResponsePB等，
# 会被转换为WriteOperationState
TabletServiceImpl::Write
    # 这里会将WriteOperationState转换为WriteOperation
    - TabletPeer::WriteAsync
        - Tablet::AcquireLocksAndPerformDocOperations
            - Tablet::KeyValueBatchFromQLWriteBatch
                - Tablet::UpdateQLIndexes
                    - Tablet::CompleteQLWriteBatch
                        - WriteOperation::StartSynchronization
                            - WriteOperation::DoStartSynchronization
                                # 这里会根据WriteOperation生成OperationDriver
                                - TabletPeer::Submit
                                    - OperationDriver::ExecuteAsync
                                        - Preparer::Submit
                                            - PreparerImpl::Submit
                                                # 将提交进来的OperationDriver添加到queue_中
                                                - queue_.Push(operation_driver)
                                                # 执行该语句的前置条件是：没有关于当前Tablet
                                                # 的Task在运行，则提交一个Task到prepare线程池
                                                # 中，以处理queue_中的OperationDriver
                                                - tablet_prepare_pool_token_->SubmitFunc(
                                                std::bind(&PreparerImpl::Run, this))
```

当prepare线程池调度一个线程来服务某个Tablet时，PreparerImpl::Run方法会被执行，它的逻辑中主要包括PreparerImpl::ProcessItem，PreparerImpl::ProcessAndClearLeaderSideBatch，PreparerImpl::ReplicateSubBatch等，下面逐一分析。
```
void PreparerImpl::Run() {
  for (;;) {
    # 逐一遍历PreparerImpl::queue_中的每一个OperationDriver
    while (OperationDriver *item = queue_.Pop()) {
      active_tasks_.fetch_sub(1, std::memory_order_release);
      # 处理该OperationDriver
      ProcessItem(item);
    }
    
    # 批量处理PreparerImpl::leader_side_batch_中剩余的OperationDriver
    ProcessAndClearLeaderSideBatch();
    
    # 因为PreparerImpl::queue_中的OperationDriver已经处理完了，所以首先
    # 尝试将其running_状态修改为false，但是如果在修改状态的过程中，又有了
    # 新的OperationDriver添加到了PreparerImpl::queue_中，则重新修改running_
    # 状态为true，继续在当前的for循环中运行，以处理PreparerImpl::queue_中
    # 的OperationDriver
    std::unique_lock<std::mutex> stop_lock(stop_mtx_);
    running_.store(false, std::memory_order_release);
    // Check whether tasks we added while we were setting running to false.
    if (active_tasks_.load(std::memory_order_acquire)) {
      // Got more operations, try stay in the loop.
      bool expected = false;
      if (running_.compare_exchange_strong(expected, true, std::memory_order_acq_rel)) {
        continue;
      }
      
      # 如果其它地方已经将running_状态修改为true，则可以结束当前的Task，
      # 因为已经有了其它的Task在为PreparerImpl::queue_服务
      // If someone else has flipped running_ to true, we can safely exit this function because
      // another task is already submitted to the same token.
    }
    
    # 如果接收到了停止请求
    if (stop_requested_.load(std::memory_order_acquire)) {
      stop_cond_.notify_all();
    }
    
    # 至此，可以结束当前的Task
    return;
  }
}

void PreparerImpl::ProcessItem(OperationDriver* item) {
  if (item->is_leader_side()) {
    # 获取当前的OperationDriver对应的operation type，并判断该
    # OperationDriver是否需要单独处理
    auto operation_type = item->operation_type();
    const bool apply_separately = operation_type == OperationType::kChangeMetadata ||
                                  operation_type == OperationType::kEmpty;
    const int64_t bound_term = apply_separately ? -1 : item->consensus_round()->bound_term();

    # 尽量批量的提交一批OperationDriver给raft，但是同一批OperationDriver的数目
    # 不能超过FLAGS_max_group_replicate_batch_size(默认值是16)，且同一批中的
    # 所有的OperationDriver都必须具有相同的term
    if (leader_side_batch_.size() >= FLAGS_max_group_replicate_batch_size ||
        (!leader_side_batch_.empty() &&
            bound_term != leader_side_batch_.back()->consensus_round()->bound_term())) {
      # 不能添加到当前的一批中，则处理当前的一批，处理完成之后，会清空leader_side_batch_
      ProcessAndClearLeaderSideBatch();
    }
    
    # 添加到当前的一批中
    leader_side_batch_.push_back(item);
    if (apply_separately) {
      # 如果当前的OperationDriver需要单独处理，则处理当前的一批
      ProcessAndClearLeaderSideBatch();
    }
  } else {
    ...
  }
}

void PreparerImpl::ProcessAndClearLeaderSideBatch() {
  auto iter = leader_side_batch_.begin();
  auto replication_subbatch_begin = iter;
  auto replication_subbatch_end = iter;

  # 逐一遍历leader_side_batch_中的OperationDriver
  while (iter != leader_side_batch_.end()) {
    auto* operation_driver = *iter;

    # PrepareAndStart
    Status s = operation_driver->PrepareAndStart();

    if (PREDICT_TRUE(s.ok())) {
      # 如果PrepareAndStart返回成功，则将当前的OperationDriver添加到当前这一批
      # 批量执行raft复制的OperationDriver集合中
      replication_subbatch_end = ++iter;
    } else {
      # 否则，PrepareAndStart失败，则对当前的一批OperationDriver批量执行raft复制，
      # 接着，对当前的OperationDriver进行错误处理，最后，从下一个OperationDriver
      # 开始进行新的一批OperationDriver的收集
      ReplicateSubBatch(replication_subbatch_begin, replication_subbatch_end);

      // Handle failure for this operation itself.
      operation_driver->HandleFailure(s);

      // Now we'll start accumulating a new batch.
      replication_subbatch_begin = replication_subbatch_end = ++iter;
    }
  }

  # 最后一批OperationDriver统一处理
  ReplicateSubBatch(replication_subbatch_begin, replication_subbatch_end);

  # 清空leader_side_batch_
  leader_side_batch_.clear();
}

void PreparerImpl::ReplicateSubBatch(
    OperationDrivers::iterator batch_begin,
    OperationDrivers::iterator batch_end) {
  # rounds_to_replicate_用于存放即将批量提交给raft进行复制的所有
  # OperationDriver对应的ConsensusRound，首先清空rounds_to_replicate_，
  # 并预留一定的容量
  rounds_to_replicate_.clear();
  rounds_to_replicate_.reserve(std::distance(batch_begin, batch_end));
  
  # 将这一批OperationDriver对应的ConsensusRound添加到rounds_to_replicate_
  for (auto batch_iter = batch_begin; batch_iter != batch_end; ++batch_iter) {
    rounds_to_replicate_.push_back((*batch_iter)->consensus_round());
  }

  # raft批量复制
  const Status s = consensus_->ReplicateBatch(&rounds_to_replicate_);
  
  # 清空rounds_to_replicate_
  rounds_to_replicate_.clear();
}
```

### TSTabletManager中的名为append线程池
对应于TSTabletManager::append_pool_，线程池类型为ThreadPool(注意：跟前面Messenger中的rpc::ThreadPool不同)，在TSTabletManager::TSTabletManager中创建：
```
TSTabletManager::TSTabletManager(...) {
    ...
    
    CHECK_OK(ThreadPoolBuilder("append")
               .unlimited_threads()
               .set_idle_timeout(MonoDelta::FromMilliseconds(10000))
               .Build(&append_pool_));
        
    ...
}
```

**该线程池中的线程数目无上限。**

#### append线程池中的线程的使用
append线程被Tablet对应的Log的Log::Appender所使用，更进一步的说，是被Log::Appender中的TaskStream所使用，被所有的Tablet所共享。
```
TSTabletManager::OpenTablet
    - tablet::BootstrapTabletData data = {
        ...,
        # 也就是TSTabletManager::append_pool_
        append_pool(),
        ...
      }
    - BootstrapTablet(data, ...)
        - enterprise::TabletBootstrap bootstrap(data)
            - TabletBootstrap::TabletBootstrap(data)
                # 设置TabletBootstrap::append_pool_
                - append_pool_ = data.append_pool
        # TabletBootstrap::Bootstrap
        - bootstrap.Bootstrap
            - TabletBootstrap::OpenNewLog
                # 将TabletBootstrap::append_pool_作为参数
                - Log::Open(..., append_pool_, ...)
                    # append_thread_pool就是TabletBootstrap::append_pool_
                    - new_log = new Log(..., append_thread_pool)
                        - appender_ = new Appender(this, append_thread_pool)
                            # 初始化Log::Appender中的TaskStream
                            - task_stream_(new TaskStream<LogEntryBatch>(..., append_thread_pool, ...)
                                # TaskStream的实现则是在TaskStreamImpl中
                                - impl_(std::make_unique<TaskStreamImpl<T>>(
                                process_item, append_thread_pool, queue_max_size, queue_max_wait))
                                    # 传递进来的append_thread_pool用于创建taskstream_pool_token_
                                    # 这一ThreadPoolToken
                                    - taskstream_pool_token_(append_thread_pool->NewToken(
                                    ThreadPool::ExecutionMode::SERIAL))
```

那么Log::Appender的作用又是什么呢？且看如下代码：
```
RaftConsensus::ReplicateBatch
    RaftConsensus::AppendNewRoundsToQueueUnlocked
        PeerMessageQueue::AppendOperations
            LogCache::AppendOperations
                # 添加到LogCache.MessageCache中
                LogCache::PrepareAppendOperations
                # 异步写入WAL log中
                Log::AsyncAppendReplicates
                    Log::AsyncAppend
                        Log::Appender::Submit 
                            TaskStream<T>::Submit
                                TaskStreamImpl<T>::Submit
                                    # 将LogEntryBatch提交到TaskStreamImpl的队列中
                                    queue_.BlockingPut(task)
                                    # 如果没有关于当前Tablet的Log的append Task正在运行，
                                    # 则提交一个任务到由taskstream_pool_token_确定的一个
                                    # 线程池中，任务的运行逻辑是TaskStreamImpl::Run，
                                    - taskstream_pool_token_->SubmitFunc(
                                    std::bind(&TaskStreamImpl::Run, this))
```
从上面的分析可见，Log::Appender用于在raft复制过程中写WAL log。Log::Appender中的TaskStream就类似于前面分析prepare线程时的Preparer，TaskStreamImpl则类似于前面分析prepare线程时的PreparerImpl，他们的数据结构和运转机制是类似的。

当append线程池中调度一个线程来执行关于某个Tablet的WAL log的写入时，TaskStreamImpl::Run会被执行，TaskStreamImpl::Run(跟前面分析prepare线程时分析到的PreparerImpl::Run是类似的)的分析中主要包括TaskStreamImpl<T>::Run，Log::Appender::ProcessBatch和Log::Appender::GroupWork等逻辑：
```
template <typename T> void TaskStreamImpl<T>::Run() {
  for (;;) {
    # 将TaskStreamImpl::queue_中的所有的LogEntryBatch添加到group数组中
    std::vector<T *> group;
    queue_.BlockingDrainTo(&group, wait_timeout_deadline);
    
    if (!group.empty()) {
      # 逐一处理group数组中的每一个LogEntryBatch
      for (T* item : group) {
        # 处理当前的LogEntryBatch，实际执行的是Log::Appender::ProcessBatch
        ProcessItem(item);
      }
      
      # 处理一个空的LogEntryBatch，以作为执行wal log sync操作的信号
      ProcessItem(nullptr);
      group.clear();
      continue;
    }
    
    # 至此，TaskStreamImpl::queue_中没有待处理的LogEntryBatch
    // Not processing and queue empty, return from task.
    std::unique_lock<std::mutex> stop_lock(stop_mtx_);
    running_--;
    if (!queue_.empty()) {
      // Got more operations, stay in the loop.
      running_++;
      continue;
    }
    
    if (stop_requested_.load(std::memory_order_acquire)) {
      stop_cond_.notify_all();
      return;
    }
    
    return;
  }
}

void Log::Appender::ProcessBatch(LogEntryBatch* entry_batch) {
  # 当传递的LogEntryBatch为空的情况下，是作为WAL log sync的信号
  if (entry_batch == nullptr) {
    # 执行WAL log sync
    GroupWork();
    return;
  }

  # 将LogEntryBatch添加到WAL log中
  Status s = log_->DoAppend(entry_batch);

  if (!log_->sync_disabled_) {
    # 如果没有禁用wal log sync，则更新wal log sync时的控制信息，
    # periodic_sync_earliest_unsync_entry_time_和periodic_sync_unsynced_bytes_
    # 都会在Log::Sync中被使用
    bool expected = false;
    if (log_->periodic_sync_needed_.compare_exchange_strong(expected, true,
                                                            std::memory_order_acq_rel)) {
      log_->periodic_sync_earliest_unsync_entry_time_ = MonoTime::Now();
    }
    log_->periodic_sync_unsynced_bytes_ += entry_batch->total_size_bytes();
  }
  
  # 将当前的LogEntryBatch添加到Log::Appender::sync_batch_中
  sync_batch_.emplace_back(entry_batch);
}

void Log::Appender::GroupWork() {
  # 该方法的主要目的就是确保Log::Appender::sync_batch_中的LogEntryBatch
  # 对应的wal log持久化
  
  # 首先检查sync_batch_是否为空，若为空，则直接执行Log::Sync
  if (sync_batch_.empty()) {
    Status s = log_->Sync();
    return;
  }
  
  # 如果sync_batch_不为空，则除了执行Log::Sync以外，还要对sync_batch_中的
  # 所有的LogEntryBatch运行其回调
  Status s = log_->Sync();
  if (PREDICT_FALSE(!s.ok())) {
    ...
  } else {
    # 逐一遍历sync_batch_中的LogEntryBatch，并运行其回调
    for (std::unique_ptr<LogEntryBatch>& entry_batch : sync_batch_) {
      if (PREDICT_TRUE(!entry_batch->failed_to_append() && !entry_batch->callback().is_null())) {
        entry_batch->callback().Run(Status::OK());
      }
      
      entry_batch.reset();
    }
    
    # 清空sync_batch_
    sync_batch_.clear();
  }
}
```

### TSTabletManager中的名为raft的线程池
对应于TSTabletManager::raft_pool_，线程池类型为ThreadPool(注意：跟前面Messenger中的rpc::ThreadPool不同)，在TSTabletManager::TSTabletManager中创建：
```
TSTabletManager::TSTabletManager(...) {
    ...
    
    CHECK_OK(ThreadPoolBuilder("raft")
               .unlimited_threads()
               .Build(&raft_pool_));
        
    ...
}
```

**该线程池中的线程数目无上限。**

#### raft线程池中的线程的使用
该线程池主要用于raft相关的操作，被所有的Tablet所共享。

对raft线程池的使用主要是通过ThreadPoolToken来进行的，主要有3个地方用到了关于raft线程池的ThreadPoolToken，分别是在PeerMessageQueue::raft_pool_observers_token_，PeerManager::raft_pool_token_和RaftConsensus::raft_pool_token_，其中PeerManager::raft_pool_token_和RaftConsensus::raft_pool_token_共用同一个ThreadPoolToken。
```
TSTabletManager::OpenTablet
    # raft_pool()就是TSTabletManager::raft_pool_
    - auto s = tablet_peer->InitTabletPeer(..., raft_pool(), ...)
        - consensus_ = RaftConsensus::Create(..., raft_pool, ...)
            # 为PeerMessageQueue创建一个关于raft pool的通行证
            - auto queue = std::make_unique<PeerMessageQueue>(..., 
                raft_pool->NewToken(ThreadPool::ExecutionMode::SERIAL))
                # 设置PeerMessageQueue::raft_pool_observers_token_
                - raft_pool_observers_token_ = std::move(raft_pool_token)
            # 创建一个关于raft pool的通行证，供RaftConsensus使用
            - unique_ptr<ThreadPoolToken> raft_pool_token(raft_pool->NewToken(
                ThreadPool::ExecutionMode::CONCURRENT))
            # 在PeerManager中会用到前面创建的raft_pool_token
            - auto peer_manager = std::make_unique<PeerManager>(...,
                raft_pool_token.get(), ...)
                # 设置PeerManager::raft_pool_token_
                - raft_pool_token_ = raft_pool_token)
            # 在RaftConsensus中会用到前面创建的raft_pool_token
            - return std::make_shared<RaftConsensus>(..., std::move(queue),              
                std::move(peer_manager), std::move(raft_pool_token), ...)
                # 设置RaftConsensus::raft_pool_token_
                - raft_pool_token_ = std::move(raft_pool_token)
                # 设置RaftConsensus::peer_manager_
                - peer_manager_ = std::move(peer_manager)
                # 设置RaftConsensus::queue_，这是一个PeerMessageQueue
                - queue_ = std::move(queue)
```

其中：

PeerMessageQueue::raft_pool_observers_token_主要被PeerMessageQueue用来通知PeerMessageQueueObserver(实际上是RaftConsensus)相关事件(比如Majority Replicated，term change，failed follower等)。PeerMessageQueueObserver只是一个接口类，RaftConsensus则继承了PeerMessageQueueObserver的接口。PeerMessageQueueObserver中主要提供了3个接口：
```
  virtual void UpdateMajorityReplicated(const MajorityReplicatedData& data,
                                        OpId* committed_index) = 0;
  virtual void NotifyTermChange(int64_t term) = 0;
  virtual void NotifyFailedFollower(const std::string& peer_uuid,
                                    int64_t term,
                                    const std::string& reason) = 0;
```

PeerManager::raft_pool_token_会进一步传递给Peer使用，最终用到的实际上是Peer::raft_pool_token_，除此之外，PeerManager::raft_pool_token_没有其它地方使用。
```
PeerManager::UpdateRaftConfig
    - auto remote_peer = Peer::NewRemotePeer(..., raft_pool_token_, ...)
        # 设置Peer::raft_pool_token_
        - raft_pool_token_ = raft_pool_token
```

RaftConsensus::raft_pool_token_主要用于构建发往Raft peer的请求，处理RPC回调等。


下面分别分析PeerMessageQueue::raft_pool_observers_token_，Peer::raft_pool_token_和RaftConsensus::raft_pool_token_这3个ThreadPoolToken(其实，Peer::raft_pool_token_和RaftConsensus::raft_pool_token_共用一个ThreadPoolToken)。

##### Peer::raft_pool_token_的使用
使用Peer::raft_pool_token_来向raft线程池提交一个任务，以向远端的peer发送本地peer中待进行raft复制的请求。
```
RaftConsensus::BecomeLeaderUnlocked/RaftConsensus::ChangeConfig/RaftConsensus::ReplicateBatch/RaftConsensus::UpdateMajorityReplicated
    PeerManager::SignalRequest
        # 通知远端的peer，有请求待复制
        Peer::SignalRequest
            # 提交一个任务到Peer::raft_pool_token_中，该任务负责将请求发送给远端peer
            - auto status = raft_pool_token_->SubmitFunc(
              std::bind(&Peer::SendNextRequest, shared_from_this(), trigger_mode))
```

##### PeerMessageQueue::raft_pool_observers_token_的使用
1. 在Peer通过Peer::SendNextRequest发送replicate请求给远端的peer的时候会设置一个回调Peer::ProcessResponse，当接收到响应的时候，该回调Peer::ProcessResponse会被调用，如果response中存在错误，且错误码是tserver::TabletServerErrorPB::WRONG_SERVER_UUID，则会通过PeerMessageQueue::raft_pool_observers_token_提交一个任务到raft线程池中，以通知RaftConsensus关于failed Follower信息。
```
Peer::SendNextRequest
    # 发送replicate请求给远端的peer，当接收到响应的时候会执行回调
    # Peer::ProcessResponse
    - proxy_->UpdateAsync(&request_, trigger_mode, &response_, &controller_,
      std::bind(&Peer::ProcessResponse, retain_self))
                      
void Peer::ProcessResponse() {
  ...
  
  # 如果返回错误响应，且响应的错误码是WRONG_SERVER_UUID，则通知RaftConsensus，
  # 对应的Follower发生了错误
  if (response_.has_error() &&
      response_.error().code() == tserver::TabletServerErrorPB::WRONG_SERVER_UUID) {
    queue_->NotifyObserversOfFailedFollower(...);
    ProcessResponseError(StatusFromPB(response_.error().status()));
    return;
  }
  
  ...
}

PeerMessageQueue::NotifyObserversOfFailedFollower
    # 提交一个任务到raft pool中，以通知RaftConsensus关于Follower failed事件
    - raft_pool_observers_token_->SubmitClosure(
        Bind(&PeerMessageQueue::NotifyObserversOfFailedFollowerTask, ...)

PeerMessageQueue::NotifyObserversOfFailedFollowerTask
    - RaftConsensus::NotifyFailedFollower
```

2. 在Peer通过Peer::SendNextRequest发送replicate请求给远端的peer的时候会设置一个回调Peer::ProcessResponse，当接收到响应的时候，该回调Peer::ProcessResponse会被调用，如果response中存在错误，且错误码是ConsensusErrorPB::INVALID_TERM，则会通过PeerMessageQueue::raft_pool_observers_token_提交一个任务到raft线程池中，以通知RaftConsensus更新term信息。
```
Peer::SendNextRequest
    # 发送replicate请求给远端的peer，当接收到响应的时候会执行回调
    # Peer::ProcessResponse
    - proxy_->UpdateAsync(&request_, trigger_mode, &response_, &controller_,
      std::bind(&Peer::ProcessResponse, retain_self))
                      
void Peer::ProcessResponse() {
    ...
  
    # 如果响应提示发生了错误，且错误码是ConsensusErrorPB::INVALID_TERM，则表明
    # 远端的peer具有新的term，则通知本地的RaftConsensus更新term
    if (PREDICT_FALSE(status.has_error())) {
      peer->is_last_exchange_successful = false;
      switch (status.error().code()) {
        ...
        
        case ConsensusErrorPB::INVALID_TERM: {
          # 通知本地的RaftConsensus更新term，response.responder_term()表示的是
          # 远端peer的term
          NotifyObserversOfTermChange(response.responder_term());
          *more_pending = false;
          return;
        }
        
        default: {
          ...
        }
      }
    }
  
    ...
}

PeerMessageQueue::NotifyObserversOfTermChange
    # 提交一个任务到raft pool中，以通知RaftConsensus关于term change事件
    - raft_pool_observers_token_->SubmitClosure(
      Bind(&PeerMessageQueue::NotifyObserversOfTermChangeTask, ...))
      
PeerMessageQueue::NotifyObserversOfTermChangeTask
    - RaftConsensus::NotifyTermChange
```

3. 在Peer通过Peer::SendNextRequest发送replicate请求给远端的peer的时候会设置一个回调Peer::ProcessResponse，当接收到响应的时候，该回调Peer::ProcessResponse会被调用，在Peer::ProcessResponse  -> PeerMessageQueue::ResponseFromPeer中会计算当前已经达到majority replicated的OpId，并提交一个任务去通知RaftConsensus关于majority replicated OpId change信息。
```
Peer::SendNextRequest
    # 发送replicate请求给远端的peer，当接收到响应的时候会执行回调
    # Peer::ProcessResponse
    - proxy_->UpdateAsync(&request_, trigger_mode, &response_, &controller_,
      std::bind(&Peer::ProcessResponse, retain_self))
      
Peer::ProcessResponse
    - PeerMessageQueue::ResponseFromPeer
        - PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
            # 提交一个任务到raft pool中，以通知RaftConsensus关于majority replicated OpId change
            - raft_pool_observers_token_->SubmitClosure(
              Bind(&PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask, ...))

PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
    - RaftConsensus::UpdateMajorityReplicated
```

4. 在RaftConsensus::ReplicateBatch总会写本地WAL log，如果写入成功，则PeerMessageQueue::LocalPeerAppendFinished会被调用，在PeerMessageQueue::LocalPeerAppendFinished -> PeerMessageQueue::ResponseFromPeer中会计算当前已经达到majority replicated的OpId，并提交一个任务去通知RaftConsensus关于majority replicated OpId change信息。
```
RaftConsensus::ReplicateBatch
    - RaftConsensus::AppendNewRoundsToQueueUnlocked
        - PeerMessageQueue::AppendOperations
            - log_cache_.AppendOperations(
              msgs, committed_op_id, batch_mono_time,
              Bind(&PeerMessageQueue::LocalPeerAppendFinished, Unretained(this), last_id)))
      
PeerMessageQueue::LocalPeerAppendFinished
    - PeerMessageQueue::ResponseFromPeer
        - PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
            # 提交一个任务到raft pool中，以通知RaftConsensus关于majority replicated OpId change
            - raft_pool_observers_token_->SubmitClosure(
              Bind(&PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask, ...))

PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
    - RaftConsensus::UpdateMajorityReplicated
```

##### RaftConsensus::raft_pool_token_的使用
1. 在选举过程中，当接收到选举响应的时候，会提交一个任务到raft pool中，以处理选举结果。
```
# 当接收到RunLeaderElection RPC请求时的处理逻辑
ConsensusServiceImpl::RunLeaderElection
    # 当然，RaftConsensus::StartElection还有其它的地方也会调用它
    RaftConsensus::StartElection
        RaftConsensus::DoStartElection
            RaftConsensus::CreateElectionUnlocked
                - LeaderElectionPtr result(new LeaderElection(
                      ...,
                      std::bind(&RaftConsensus::ElectionCallback, shared_from_this(), data, _1)))
                      # 设置LeaderElection::decision_callback_
                      - decision_callback_ = std::move(decision_callback)

# 当接收到选举响应的时候LeaderElection::VoteResponseRpcCallback会被调用
LeaderElection::VoteResponseRpcCallback
    - LeaderElection::CheckForDecision
        # 实际上是RaftConsensus::ElectionCallback
        - decision_callback_(result_)

RaftConsensus::ElectionCallback
    # 提交一个任务到raft pool中，以处理选举结果
    - raft_pool_token_->SubmitFunc(
          std::bind(&RaftConsensus::DoElectionCallback, shared_from_this(), data, result))
```

2. 当RaftConsensus接收到关于failed Follower的信息的时候，会提交一个任务到raft pool中，以移除该failed Follower。
```
RaftConsensus::NotifyFailedFollower
    # 提交一个任务到raft pool中，以移除failed Follower
    - raft_pool_token_->SubmitFunc(std::bind(&RaftConsensus::TryRemoveFollowerTask,
       shared_from_this(), uuid, committed_config, reason))
```

3. 在选举过程中，如果在在一定时间内没有成功，则会提交一个任务到raft pool中，以重新启动选举。
```
RaftConsensus::Start
    # failure_detector_用于选举过程，在选举之前，会启动failure_detector_，
    # 以监测选举过程是否在一定时间内成功，如果不成功，则会执行
    # RaftConsensus::ReportFailureDetected，以重新启动选举
    - failure_detector_ = PeriodicTimer::Create(
          peer_proxy_factory_->messenger(),
          [w]() {
            if (auto consensus = w.lock()) {
              consensus->ReportFailureDetected();
            }
          },
          MinimumElectionTimeout());
          

RaftConsensus::ReportFailureDetected
    # 提交一个任务到raft pool中，以重新启动选举
    - raft_pool_token_->SubmitFunc(std::bind(&RaftConsensus::ReportFailureDetectedTask,
        shared_from_this()))
        
RaftConsensus::ReportFailureDetectedTask
    # 重新启动选举
    - StartElection({ElectionMode::NORMAL_ELECTION})
```

### TSTabletManager中的名为apply线程池
对应于TSTabletManager::apply_pool_，线程池类型为ThreadPool(注意：跟前面Messenger中的rpc::ThreadPool不同)，在TSTabletManager::TSTabletManager中创建：
```
TSTabletManager::TSTabletManager(...) {
    ...
    
    CHECK_OK(ThreadPoolBuilder("apply")
               .set_metrics(std::move(metrics))
               .Build(&apply_pool_));
        
    ...
}
```

该线程池中的最大线程数目是CPU的个数。

#### apply线程池中的线程的使用
该线程池主要用于在TSTabletManager中标记TabletPeer/Tablet的状态发生了变更。在Raft启动或者Raft leader发生变更的情况下，以及TabletPeer启动的时候，都会向该线程池中提交一个任务，任务的执行主体是TSTabletManager::MarkTabletDirty -> TSTabletManager::MarkDirtyUnlocked，在TSTabletManager::MarkDirtyUnlocked中会伴随着心跳信息向Master汇报Tablet的状态变更。

该线程池被所有的Tablet所共享。

暂不深入分析该线程池。


### TSTabletManager中的tablet-bootstrap线程池
对应于TSTabletManager::open_tablet_pool_，线程池类型为ThreadPool(注意：跟前面Messenger中的rpc::ThreadPool不同)，在TSTabletManager::Init中创建.
```
TSTabletManager::Init(...) {
    ...
    
    RETURN_NOT_OK(ThreadPoolBuilder("tablet-bootstrap")
        .set_max_threads(max_bootstrap_threads)
        .set_metrics(std::move(metrics))
        .Build(&open_tablet_pool_));
        
    ...
}
```

最大线程数目通过max_bootstrap_threads控制，关于max_bootstrap_threads的设置，见TSTabletManager::Init：
```
TSTabletManager::Init(...) {
  ...
  
  int max_bootstrap_threads = FLAGS_num_tablets_to_open_simultaneously;
  if (max_bootstrap_threads == 0) {
    size_t num_cpus = base::NumCPUs();
    if (num_cpus <= 2) {
      max_bootstrap_threads = 2;
    } else {
      max_bootstrap_threads = min(num_cpus - 1, fs_manager_->GetDataRootDirs().size() * 8);
    }
    LOG_WITH_PREFIX(INFO) <<  "max_bootstrap_threads=" << max_bootstrap_threads;
  }
  
  ...
}
```

该线程池主要用于创建/打开Tablet，见TSTabletManager::CreateNewTablet：
```
TSTabletManager::CreateNewTablet
    - open_tablet_pool_->SubmitFunc(std::bind(&TSTabletManager::OpenTablet, this, meta, deleter))
```

### TSTabletManager中的名为read-parallel线程池
对应于TSTabletManager::read_pool_，线程池类型为ThreadPool(注意：跟前面Messenger中的rpc::ThreadPool不同)，在TSTabletManager::TSTabletManager中创建：
```
TSTabletManager::TSTabletManager(...) {
    ...
    
    CHECK_OK(ThreadPoolBuilder("read-parallel")
               .set_max_threads(FLAGS_read_pool_max_threads)
               .set_max_queue_size(FLAGS_read_pool_max_queue_size)
               .set_metrics(std::move(read_metrics))
               .Build(&read_pool_));
        
    ...
}
```

该线程池中的最大线程数目通过FLAGS_read_pool_max_threads参数进行设置。

### TSTabletManager中的名为flush scheduler bgtask的BackgroundTask
对应于TSTabletManager::background_task_，在TSTabletManager::TSTabletManager中创建：
```
TSTabletManager::TSTabletManager(...) {
  ...
    
  if (should_count_memory) {
    background_task_.reset(new BackgroundTask(
      std::function<void()>([this](){ MaybeFlushTablet(); }),
      "tablet manager",
      "flush scheduler bgtask",
      std::chrono::milliseconds(FLAGS_flush_background_task_interval_msec)));
  }

  ...
}
```

在TSTabletManager::Init中启动：
```
TSTabletManager::Init(...) {
    ...
    
    if (background_task_) {
      # BackgroundTask::Init中会创建线程
      RETURN_NOT_OK(background_task_->Init());
    }
        
    ...
}
```

BackgroundTask所在的线程中，线程的运行主体为BackgroundTask::Run -> TSTabletManager::MaybeFlushTablet，用于当memstore达到内存上限的情况下，flush某些Tablet。

### Rocksdb相关的线程


## RedisServer
