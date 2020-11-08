# 提纲
[toc]

## 前言
下面的分析以write请求为例。

## CQLServer端RPC请求的发送逻辑
根据“YugeByte CQLServer”中的分析，CQLServer的RPC请求都会经过Proxy::DoAsyncRequest，在Proxy::DoAsyncRequest中，会根据是否是本地调用分别处理：
- 如果是本地调用：
    - 则生成一个LocalOutboundCall；
    - 将LocalOutboundCall转换为LocalYBInboundCall；
    - 如果允许在当前线程执行本地rpc调用，且当前线程是RPC线程：
        - 直接在当前线程处理（见Messenger::Handle(InboundCallPtr call)）；
    - 否则：
        - 添加到LocalInboundCall的请求队列中，等待处理（见Messenger::QueueInboundCall(InboundCallPtr call)）；
- 如果是一个远程调用：
    - 则生成一个OutboundCall；
    - 将OutboundCall添加到OutboundCall队列，等待处理；

## 运行单个节点，且没有特殊配置情况下走的是本地调用还是远程调用
在启动TServer的时候会指定--cql_proxy_bind_address参数，假如当前的节点的IP是10.10.10.10，若指定--cql_proxy_bind_address=10.10.10.10:****，则CQLServer向TServer的RPC会走本地调用，否则会走远程调用。

## 如果是本地调用，在没有特殊配置的情况下是直接在当前线程处理还是添加到LocalInboundCall的请求队列中
根据调试发现，会直接在当前线程处理，也就是走的如下的逻辑：
```
Proxy::DoAsyncRequest
    - ...
        # Messenger::Handle
        - context_->Handle(local_call);
```

### 如果CQLServer端RPC请求直接在当前线程执行，则进入Messenger::Handle之后的逻辑分析
在当前上下文中，Messenger::Handle最终调用的是TabletServerServiceIf::Handle。
```
Messenger::Handle
    # 找到对应的RPC Service
    - auto service = rpc_service(call->service_name())
    # 经过追溯，在当前上下文中调用的是TabletServerServiceIf::Handle
    - service->Handle(std::move(call))
```

TabletServerServiceIf::Handle中对写操作的RPC请求最终会调用TabletServiceImpl::Write：
```
void TabletServerServiceIf::Handle(::yb::rpc::InboundCallPtr call) {
  auto yb_call = std::static_pointer_cast<::yb::rpc::YBInboundCall>(call);
  if (call->method_name() == "Write") {
    # 创建RpcContext，会区分本地调用还是远程调用，在当前上下文中，
    # 走的是本地调用
    auto rpc_context = yb_call->IsLocalCall() ?
        ::yb::rpc::RpcContext(
            std::static_pointer_cast<::yb::rpc::LocalYBInboundCall>(yb_call), 
            metrics_[kMetricIndexWrite]) :
        ::yb::rpc::RpcContext(
            yb_call, 
            std::make_shared<WriteRequestPB>(),
            std::make_shared<WriteResponsePB>(),
            metrics_[kMetricIndexWrite]);
    # 如果还没有发送响应，则执行之
    if (!rpc_context.responded()) {
      const auto* req = static_cast<const WriteRequestPB*>(rpc_context.request_pb());
      auto* resp = static_cast<WriteResponsePB*>(rpc_context.response_pb());
      # 调用的实际是TabletServiceImpl::Write
      Write(req, resp, std::move(rpc_context));
    }
    return;
  }
  
  ...
}
```

在TabletServiceImpl::Write的逻辑见“TabletServiceImpl::Write分析”。

综上，如果CQLServer端RPC请求直接在当前线程执行，对于write请求来说，直到真实调用TabletServiceImpl::Write之前都一直在当前线程执行。

## TabletServiceImpl::Write分析
TabletServiceImpl::Write

```
void TabletServiceImpl::Write(const WriteRequestPB* req,
                              WriteResponsePB* resp,
                              rpc::RpcContext context) {
  auto tablet = LookupLeaderTabletOrRespond(
      server_->tablet_peer_lookup(), req->tablet_id(), resp, &context);
  auto operation_state = std::make_unique<WriteOperationState>(tablet.peer->tablet(), req, resp);
  auto context_ptr = std::make_shared<RpcContext>(std::move(context));
  
  # 设置一个回调，该回调会在PeerMessageQueue::ResponseFromPeer -> ...
  # -> OperationDriver::ReplicationFinished -> OperationDriver::ApplyOperation -> ...
  # -> OperationState::CompleteWithStatus中被调用
  operation_state->set_completion_callback(
        std::make_unique<WriteOperationCompletionCallback>(
            tablet.peer, context_ptr, resp, operation_state.get(), server_->Clock(),
            req->include_trace()));
            
  # TabletPeer::WriteAsync     
  tablet.peer->WriteAsync(
      std::move(operation_state), tablet.leader_term, context_ptr->GetClientDeadline());
}
```

TabletPeer::WriteAsync中会生成WriteOperation，并调用Tablet::AcquireLocksAndPerformDocOperations：
```
TabletPeer::WriteAsync
    # 生成一个WriteOperation，设置其OperationState为@state，OperationType为OperationType::kWrite
    - auto operation = std::make_unique<WriteOperation>(std::move(state), term, deadline, this);
    # Tablet::AcquireLocksAndPerformDocOperations
    - tablet_->AcquireLocksAndPerformDocOperations(std::move(operation));
```

Tablet::AcquireLocksAndPerformDocOperations中对于cql类型的write请求，会直接调用KeyValueBatchFromQLWriteBatch进行处理：
```
Tablet::AcquireLocksAndPerformDocOperations
    - Tablet::KeyValueBatchFromQLWriteBatch
```

关于Tablet::KeyValueBatchFromQLWriteBatch的处理流程，见“YugaByte TServer中写流程分析”一文。

## 当写请求在Raft group中多数节点复制成功之后发送RPC响应给CQLServer的流程
根据“Yugabyte Raft相关分析”一文，当Leader接收到本地或者远端的Raft peer的响应之后，会执行PeerMessageQueue::ResponseFromPeer，在PeerMessageQueue::ResponseFromPeer中会通过OpIdWatermark()方法计算出哪些OpId已经在多数节点中复制成功，并据此更新局部变量MajorityReplicatedData，然后执行PeerMessageQueue::NotifyObserversOfMajorityReplOpChange，并以MajorityReplicatedData作为参数。**在PeerMessageQueue::NotifyObserversOfMajorityReplOpChange中会提交一个任务到raft_pool_observers_token_（对应的线程池是TSTabletManager::raft_pool_）中**，任务的运行主体是PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask。PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask会依次执行如下调用过程，该过程中没有发生线程切换。
```
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
        RaftConsensus::UpdateMajorityReplicated
            ReplicaState::UpdateMajorityReplicatedUnlocked
                ReplicaState::AdvanceCommittedOpIdUnlocked
                    ReplicaState::ApplyPendingOperationsUnlocked    
                        ReplicaState::NotifyReplicationFinishedUnlocked   
                            OperationDriver::ReplicationFinished
                                OperationDriver::ApplyOperation
                                    OperationDriver::ApplyTask
                                        Operation::Replicated
                                            WriteOperation::DoReplicated
                                                Tablet::ApplyRowOperations
                                                    Tablet::ApplyKeyValueRowOperations
                                                WriteOperationState::Commit()
                                            OperationState::CompleteWithStatus
                                                # 调用的是completion_clbk_->CompleteWithStatus，
                                                # completion_clbk_实际上是
                                                # WriteOperationCompletionCallback，
                                                # 它是在TabletServiceImpl::Write中被设置的
                                                WriteOperationCompletionCallback::OperationCompleted
                                                
WriteOperationCompletionCallback::OperationCompleted     
    # 将响应数据添加到YBInboundCall的sidecars_中
    RpcContext::AddRpcSidecar
        YBInboundCall::AddRpcSidecar
    RpcContext::RespondSuccess
        YBInboundCall::RespondSuccess
            # 本来调用的是YBInboundCall::Respond，
            # 实际调用的是yb::rpc::LocalYBInboundCall::Respond
            yb::rpc::LocalYBInboundCall::Respond
                auto call = outbound_call()
                # 本来调用的是call->SetFinished()，
                # 实际调用的是yb::rpc::OutboundCall::SetFinished
                yb::rpc::OutboundCall::SetFinished
                    SetState(FINISHED_SUCCESS)
                    InvokeCallback
                        # 如果callback_thread_pool_不为空，即指定了callback的线程池，
                        # 则在callback_thread_pool_中执行，否则在当前线程中执行
                        # 那么哪些情况下callback_thread_pool_会被设置呢：
                        # 如果是LocalOutboundCall，则callback_thread_pool_为null，
                        # 否则callback_thread_pool_不为null（见Proxy::DoAsyncRequest）
                        #
                        # 如果callback_thread_pool_不为null，则会在callback_thread_pool_
                        # 中执行回调，否则直接在当前线程中执行，因为当前上下文的分析中，
                        # 用到的是LocalOutboundCall，所以直接在当前线程中执行
                        InvokeCallbackSync
                            # 调用callback_，它是在OutboundCall的构造方法中设置的，
                            # 在当前上下文中对应的是WriteRpc::Finished，具体见
                            # WriteRpc::CallRemoteMethod
                            AsyncRpc::Finished
                                WriteRpc::ProcessResponseFromTserver
                                    Batcher::ProcessWriteResponse
                                Batcher::RemoveInFlightOpsAfterFlushing
                                Batcher::CheckForFinishedFlush
                                    # 如果Batcher对应的YBClient中的callback_threadpool为空，
                                    # 则直接执行回调，
                                    # 否则向callback_threadpool提交关于回调的runnable，
                                    # 如果提交失败，则直接执行回调
                                    # Batcher::RunCallback的实现见下面
                                    Batcher::RunCallback
         
void Batcher::RunCallback(const Status& status) {
  # 生成一个关于回调方法的runnable
  auto runnable = std::make_shared<yb::FunctionRunnable>(
      [ cb{std::move(flush_callback_)}, status ]() { cb(status); });

  # 如果Batcher对应的YBClient中的callback_threadpool为空，
  # 则直接执行回调，
  # 否则向callback_threadpool提交关于回调的runnable，
  # 如果提交失败，则直接执行回调
  if (!client_->callback_threadpool() || !client_->callback_threadpool()->Submit(runnable).ok()) {
    runnable->Run();
  }
}
```

**从Batcher::RunCallback来看，如果Batcher对应的YBClient中的callback_threadpool不为空，则会尝试在callback_threadpool中执行回调，此时会涉及到线程切换，否则不会发生线程切换。**

那么Batcher中的flush_callback_是啥，在哪里设置的呢？原来是通过CQLServiceImpl::Handle -> ... -> CQLProcessor::ProcessRequest -> Executor::ExecuteAsync -> ... -> Executor::FlushAsync -> YBSession::Flush -> YBSession::FlushAsync -> Batcher::FlushAsync过程进行设置的，且在YBSession::Flush中设置为Synchronizer::AsStatusFunctor()，它的作用是等待数据持久化之后来通知CQLServer，否则调用一直会在Executor::FlushAsync处等待？















