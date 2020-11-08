# 提纲
[toc]

# 前言
在YugaByte上进行TPCC测试的过程中，总是出现“Already present ... Duplicate request”错误，经Debug发现这个错误是在TPCC测试过程中出现了超时所致。于是学习了一下YugaByte RPC请求超时处理机制。本文将记录目前的学习成果。

本文将主要从以下3个方面描述YugaByte RPC请求超时处理机制：
- client端对请求超时的判断机制
- client端对超时的请求的处理
- server端对于重试的RPC请求的处理

本文将以WriteRpc的处理为例来分析YugaByte RPC请求超时处理机制。

## Client端RPC请求发送逻辑
### 与RPC超时处理相关的数据结构
每个RPC中都会关联一个RpcRetrier，顾名思义，RpcRetrier就是用于对失败的RPC进行重试的。
```
class Rpc : public RpcCommand {
 public:
  Rpc(CoarseTimePoint deadline, Messenger* messenger, ProxyCache* proxy_cache)
      : retrier_(deadline, messenger, proxy_cache) {
  }

 private:
  friend class RpcRetrier;

  // Used to retry some failed RPCs.
  RpcRetrier retrier_;

  DISALLOW_COPY_AND_ASSIGN(Rpc);
};
```

在RpcRetrier中有一个成员deadline_，用于定义用户期望的RPC响应时间，也就是说用户期望该RPC在deadline_之前响应。另外，RpcRetrier中还包括一个RpcController类型的成员controller_，在发送RPC的时候要用到这个RpcController，因为每个RPC对应一个RpcRetrier，而每个RpcRetrier也对应一个RpcController，所以一个RPC对应一个RpcController。
```
class RpcRetrier {
 public:
  RpcRetrier(CoarseTimePoint deadline, Messenger* messenger, ProxyCache *proxy_cache);

  const CoarseTimePoint deadline_;

  ...
  
  // RPC controller to use when sending the RPC.
  RpcController controller_;
};
```

### WriteRpc请求发送流程
以WriteRpc为例，请求发送流程如下：
```
# AsyncRpc作为WriteRpc的基类，WriteRpc的发送调用的是基类AsyncRpc的SendRpc
AsyncRpc::SendRpc   
    - TabletInvoker::Execute
        # AsyncRpcBase是WriteRpc的基类，AsyncRpc则是AsyncRpcBase的基类
        - AsyncRpcBase<Req, Resp>::SendRpcToTserver
            - AsyncRpc::SendRpcToTserver
                - WriteRpc::CallRemoteMethod
                    - TabletServerServiceProxy::WriteAsync(req_, &resp_, PrepareController(timeout_),
                    std::bind(&WriteRpc::Finished, this, Status::OK()))
                            - Proxy::AsyncRequest
                                - Proxy::DoAsyncRequest
```

#### 设置RpcController中的timeout信息
WriteRpc::PrepareController会进一步调用RpcRetrier::PrepareController，并对RpcRetrier中的类型为RpcController的controller_进行设置，设置其timeout信息，最终返回该controller。RpcController中的timeout信息将会作为WriteRpc所对应的OutboundCall的超时时间(后面将OutboundCall添加到Connection对应的expiration_queue_的时候会用到这个超时时间)。
```
class Rpc : public RpcCommand {
  RpcController* PrepareController(MonoDelta single_call_timeout = MonoDelta()) {
    return retrier_.PrepareController(single_call_timeout);
  }
}
  
RpcController* RpcRetrier::PrepareController(MonoDelta single_call_timeout) {
  if (!single_call_timeout) {
    # 如果没有指定single_call_timeout的话，则采用FLAGS_retryable_rpc_single_call_timeout_ms
    # 作为single_call_timeout的值
    single_call_timeout = MonoDelta::FromMilliseconds(FLAGS_retryable_rpc_single_call_timeout_ms);
  }
  
  # 设置controller_中的timeout信息
  controller_.set_timeout(std::min<MonoDelta>(
      deadline_ - CoarseMonoClock::now(), single_call_timeout));
  return &controller_;
}  
```

#### 根据WriteRpc生成OutboundCall并添加到Reactor的发送队列中
Proxy::DoAsyncRequest中的参数controller和callback分别对应于前面PrepareController(timeout_)返回的RpcController和std::bind(&WriteRpc::Finished, this, Status::OK())所返回的ResponseCallback。在Proxy::DoAsyncRequest中主要执行如下逻辑：
- 如果是本地调用，则生成LocalOutboundCall，如果是远程调用，则生成OutboundCall，并将其赋值给RpcController::call_，本文将以远程调用为例，生成的OutboundCall中包含以下信息：
    - RpcController* controller_：表示这个OutboundCall对应的RpcController，实际上是由前面的Rpc::PrepareController(timeout_)中返回的RpcController指针赋值而来
    - int32_t call_id_：这是一个单调递增的ID，用于唯一标识一个OutboundCall
    - ResponseCallback callback_：在当前上下文中，实际上就是WriteRpc::Finished
    - 【注：】经过上述步骤之后，OutboundCall和RpcController之间相互引用
- 为该RpcController寻找一个Reactor，并将该OutboundCall添加到Reactor的发送队列中
```
void Proxy::DoAsyncRequest(const RemoteMethod* method,
                           const google::protobuf::Message& req,
                           google::protobuf::Message* resp,
                           RpcController* controller,
                           ResponseCallback callback,
                           bool force_run_callback_on_reactor) {
  CHECK(controller->call_.get() == nullptr) << "Controller should be reset";
  is_started_.store(true, std::memory_order_release);

  # 如果是本地调用，则生成LocalOutboundCall，如果是远程调用，则生成OutboundCall，并将其赋值给RpcController::call_，本文将以远程调用为例
  controller->call_ =
      call_local_service_ ?
      std::make_shared<LocalOutboundCall>(method,
                                          outbound_call_metrics_,
                                          resp,
                                          controller,
                                          &context_->rpc_metrics(),
                                          std::move(callback)) :
      # 生成OutboundCall
      std::make_shared<OutboundCall>(method,
                                     outbound_call_metrics_,
                                     resp,
                                     controller,
                                     &context_->rpc_metrics(),
                                     std::move(callback),
                                     GetCallbackThreadPool(
                                         force_run_callback_on_reactor,
                                         controller->invoke_callback_mode()));
  auto call = controller->call_.get();
  if (call_local_service_) {
    ...
  } else {
    auto ep = resolved_ep_.Load();
    if (ep.address().is_unspecified()) {
      ...
    } else {
      # 为RpcController寻找一个连接，通过这个连接将OutboundCall发送出去
      QueueCall(controller, ep);
    }
  }
}

void Proxy::QueueCall(RpcController* controller, const Endpoint& endpoint) {
  # 为RpcController寻找一个连接(对应的ID)
  uint8_t idx = num_calls_.fetch_add(1) % num_connections_to_server_;
  ConnectionId conn_id(endpoint, idx, protocol_);
  controller->call_->SetConnectionId(conn_id, &remote_.host());
  # 将OutboundCall添加到连接的发送队列中，见Messenger::QueueOutboundCall
  context_->QueueOutboundCall(controller->call_);
}

void Messenger::QueueOutboundCall(OutboundCallPtr call) {
  # 根据连接的ID和远端地址(remote)获取一个Reactor，用于发送该OutboundCall
  const auto& remote = call->conn_id().remote();
  Reactor *reactor = RemoteToReactor(remote, call->conn_id().idx());

  # 并将OutboundCall添加到Reactor中的outbound_queue_中
  reactor->QueueOutboundCall(std::move(call));
}

void Reactor::QueueOutboundCall(OutboundCallPtr call) {
  bool was_empty = false;
  bool closing = false;
  {
    std::lock_guard<simple_spinlock> lock(outbound_queue_lock_);
    if (!outbound_queue_stopped_) {
      # 将当前的OutboundCall插入之前，该Reactor的发送队列是否为空
      was_empty = outbound_queue_.empty();
      
      # 将当前的OutboundCall添加到该Reactor的发送队列中
      outbound_queue_.push_back(call);
    } else {
      closing = true;
    }
  }
  
  if (closing) {
    call->Transferred(AbortedError(), nullptr /* conn */);
    return;
  }
  
  if (was_empty) {
    # 如果添加打过去的OutboundCall之前，Reactor的发送队列为空，则在成功添加当前的OutboundCall
    # 之后，需要调度一个处理任务，用于处理Reactor的发送队列中的请求
    auto scheduled = ScheduleReactorTask(process_outbound_queue_task_);
    LOG_IF_WITH_PREFIX(WARNING, !scheduled) << "Failed to schedule process outbound queue task";
  }
}
```

#### Reactor发送OutboundCall
处于Reactor的发送队列(outbound_queue)中的请求将在Reactor::ProcessOutboundQueue中被处理：
- 对发送队列(outbound_queue)中的每个OutboundCall依次进行处理：
    - 为当前的OutboundCall分配(查找或者创建)一个连接
    - 将当前的OutboundCall转换为byte buffer，并添加到为之分配的连接对应的sending队列中
    - 将当前的OutboundCall添加到为之分配的连接的超时队列中，在超时队列中每个OutboundCall会按照它所对应的deadline时间排序
- 对发送队列(outbound_queue)中每一个OutboundCall所分配的连接分别处理：
    - 将连接的sending队列中的关于OutboundCall的数据通过网络发送出去
    - 对于成功发送的OutboundCall，还会添加到为之分配的连接对应的等待响应的请求队列中

这里总结一下要点：每个OutboundCall都会通过某个连接发送出去，在发送出去之前，会将OutboundCall添加到连接对应的超时队列expiration_queue_中，用于跟踪OutboundCall是否超时，在成功将OutboundCall发送出去之后，会将OutboundCall添加到连接对应的等待响应的请求队列awaiting_response_中，该队列中的每个OutboundCall都已经成功发送，正在等待响应。

```
void Reactor::ProcessOutboundQueue() {
  CHECK(processing_outbound_queue_.empty()) << yb::ToString(processing_outbound_queue_);
  {
    std::lock_guard<simple_spinlock> lock(outbound_queue_lock_);
    # outbound_queue_中保存的是当前尚未发送的请求
    # 将outbound_queue_中所有尚未发送的请求交换到processing_outbound_queue_中
    outbound_queue_.swap(processing_outbound_queue_);
  }
  
  processing_connections_.reserve(processing_outbound_queue_.size());
  # 为processing_outbound_queue_中的每个待发送的OutboundCall分配连接，然后将
  # OutboundCall转换为Byte buffer，并添加到为当前OutboundCall分配的连接对应的
  # sending队列中
  for (auto& call : processing_outbound_queue_) {
    auto conn = AssignOutboundCall(call);
    processing_connections_.push_back(std::move(conn));
  }
  processing_outbound_queue_.clear();

  # 对processing_connections_中的连接进行去重，因为不同的OutboundCall可能分配相同的连接
  std::sort(processing_connections_.begin(), processing_connections_.end());
  auto new_end = std::unique(processing_connections_.begin(), processing_connections_.end());
  # 从processing_connections_中去除那些重复的连接
  processing_connections_.erase(new_end, processing_connections_.end());
  
  # 对每个连接对应的sending队列中的请求进行发送(在前面的AssignOutboundCall中只是将
  # OutboundCall添加到连接对应的sending队列中，并没有发送出去)
  for (auto& conn : processing_connections_) {
    if (conn) {
      # 见Connection::OutboundQueued
      conn->OutboundQueued();
    }
  }
  processing_connections_.clear();
}

ConnectionPtr Reactor::AssignOutboundCall(const OutboundCallPtr& call) {
  ...
  
  # 根据当前的OutboundCall对应的ConnectionId，查找或者创建一个连接
  Status s = FindOrStartConnection(call->conn_id(), call->hostname(), deadline, &conn);
  
  ...
  
  # 将OutboundCall添加到连接对应的待发送队列中，并将OutboundCall添加到连接对应的
  # 超时队列expiration_queue_中
  conn->QueueOutboundCall(call);
  return conn;
}

void Connection::QueueOutboundCall(const OutboundCallPtr& call) {
  # 将OutboundCall添加到当前连接对应的待发送队列中
  auto handle = DoQueueOutboundData(call, true);

  // Set up the timeout timer.
  const MonoDelta& timeout = call->controller()->timeout();
  if (timeout.Initialized()) {
    # 根据OutboundCall的timeout计算deadline，
    auto expires_at = CoarseMonoClock::Now() + timeout.ToSteadyDuration();
    # 如果当前连接对应的超时队列为空，或者当前OutboundCall对应的超时时间比超时队列中的
    # 最小超时时间还要小，则需要重新设置定时器的超时时间
    auto reschedule = expiration_queue_.empty() || expiration_queue_.top().time > expires_at;
    # 将该OutboundCall按照deadline的顺序插入到当前连接对应的超时队列(expiration_queue)中
    expiration_queue_.push({expires_at, call, handle});
    if (reschedule && (stream_->IsConnected() ||
           expires_at < last_activity_time_ + FLAGS_rpc_connection_timeout_ms * 1ms)) {
      # 重新设置定时器的触发时间
      timer_.Start(timeout.ToSteadyDuration());
    }
  }

  call->SetQueued();
}

size_t Connection::DoQueueOutboundData(OutboundDataPtr outbound_data, bool batch) {
  ...
  
  # 最终调用的是TcpStream::Send
  auto result = stream_->Send(std::move(outbound_data));
  
  ...

  return result;
}

size_t TcpStream::Send(OutboundDataPtr data) {
  size_t result = data_blocks_sent_ + sending_.size();

  # 将OutboundData(OutboundCall是OutboundData的子类)添加到连接对应的待发送队列(sending_)中
  // Serialize the actual bytes to be put on the wire.
  sending_.emplace_back(std::move(data), mem_tracker_);

  return result;
}

void Connection::OutboundQueued() {
  # 对当前连接对应的sending队列中的请求进行发送，见TcpStream::TryWrite
  auto status = stream_->TryWrite();
}

Status TcpStream::TryWrite() {
  auto result = DoWrite();
  ...
}

Status TcpStream::DoWrite() {
  // If we weren't waiting write to be ready, we could try to write data to socket.
  while (!sending_.empty()) {
    iovec iov[kMaxIov];
    # 使用待发送队列sending_中的数据来填充iovec数组，iovec数组中至多保存kMaxIov(16)个元素
    auto fill_result = FillIov(iov);

    # 将iovec数组中的数据通过网络进行发送
    int32_t written = 0;
    auto status = fill_result.len != 0
        ? socket_.Writev(iov, fill_result.len, &written)
        : Status::OK();

    context_->UpdateLastWrite();

    send_position_ += written;
    # 逐一处理sending_队列中的每个已经发送的请求
    while (!sending_.empty()) {
      auto& front = sending_.front();
      size_t full_size = front.bytes_size();
      if (front.skipped) {
        PopSending();
        continue;
      }
      if (send_position_ < full_size) {
        break;
      }
      auto data = front.data;
      send_position_ -= full_size;
      PopSending();
      if (data) {
        # 如果当前请求已被发送，则通知对应的OutboundCall，它已经被成功发送了
        # 见Connection::Transferred
        context_->Transferred(data, Status::OK());
      }
    }
  }

  return Status::OK();
}

void Connection::Transferred(const OutboundDataPtr& data, const Status& status) {
  # 见RpcCall::Transferred(参数中的data类型实际上是OutboundCall，OutboundCall是
  # OutboundData的子类，而OutboundCall又是RpcCall的子类)
  data->Transferred(status, this);
}

void RpcCall::Transferred(const Status& status, Connection* conn) {
  state_ = status.ok() ? TransferState::FINISHED : TransferState::ABORTED;
  # 在当前上下文中RpcCall的实际类型是OutboundCall，所以调用的是
  # OutboundCall::NotifyTransferred
  NotifyTransferred(status, conn);
}

void OutboundCall::NotifyTransferred(const Status& status, Connection* conn) {
  if (status.ok()) {
    # 将OutboundCall添加到它所关联的连接对应的等待响应的请求队列(awaiting_response_)
    # 中，见Connection::CallSent
    conn->CallSent(shared_from(this));
  }

  ...
}

void Connection::CallSent(OutboundCallPtr call) {
  # 将OutboundCall添加到等待响应的请求队列(awaiting_response_)
  awaiting_response_.emplace(call->call_id(), !call->IsFinished() ? call : nullptr);
}
```

## RPC超时情况下Client端的处理逻辑
在Connection启动过程中，为RPC请求超时事件设置的处理函数为Connection::HandleTimeout：
```
Status Connection::Start(ev::loop_ref* loop) {
  context_->SetEventLoop(loop);

  timer_.Init(*loop);
  timer_.SetCallback<Connection, &Connection::HandleTimeout>(this); // NOLINT

  ...
}
```

当RPC请求超时事件被触发时，Connection::HandleTimeout会被调用来处理该超时事件：
```
void Connection::HandleTimeout(ev::timer& watcher, int revents) {  // NOLINT
  # 获取当前的时间，后面会用该时间来判断超时队列中哪些OutboundCall已经超时了
  auto now = CoarseMonoClock::Now();
  CoarseTimePoint deadline = CoarseTimePoint::max();
  if (!stream_->IsConnected()) {
    # 对连接中断的情况的处理，暂略
    ...
  }

  # 检查超时队列中所有过期时间小于now的OutboundCall，这些OutboundCall都超时了
  while (!expiration_queue_.empty() && expiration_queue_.top().time <= now) {
    auto& top = expiration_queue_.top();
    auto call = top.call.lock();
    auto handle = top.handle;
    # 从超时队列中移除当前已经超时的OutboundCall
    expiration_queue_.pop();
    if (call && !call->IsFinished()) {
      # 如果该OutboundCall还没有完成(比如失败，或者已经响应给了用户)，则设置该
      # OutboundCall的为TIMED_OUT状态，同时调用OutboundCall中设定的回调，见
      # OutboundCall::SetTimedOut
      call->SetTimedOut();
      
      # 将该OutboundCall从awaiting_response_队列中移除
      auto i = awaiting_response_.find(call->call_id());
      if (i != awaiting_response_.end()) {
        i->second.reset();
      }
    }
  }

  if (!expiration_queue_.empty()) {
    deadline = std::min(deadline, expiration_queue_.top().time);
  }

  if (deadline != CoarseTimePoint::max()) {
    # 重新设置定时器下一次超时时间
    timer_.Start(deadline - now);
  }
}

void OutboundCall::SetTimedOut() {
  bool invoke_callback;
  {
    std::lock_guard<simple_spinlock> l(lock_);
    status_ = std::move(status);
    # 设置OutboundCall::status_为TimedOut，OutboundCall::state_为TIMED_OUT，
    # 如果设置成功，则返回true
    invoke_callback = SetState(TIMED_OUT);
  }
  
  if (invoke_callback) {
    # 调用OutboundCall中设置的回调
    # 
    # 以WriteRpc为例，它对应的OutboundCall中的回调是在WriteRpc::CallRemoteMethod
    # 中设置的std::bind(&WriteRpc::Finished, this, Status::OK())，所以最终
    # WriteRpc::Finished会被调用
    #
    # WriteRpc::Finished实际上就是AsyncRpc::Finished
    InvokeCallback();
  }
}

void AsyncRpc::Finished(const Status& status) {
  Status new_status = status;
  # TabletInvoker::Done，如果返回false，则该RPC会重试，如果返回true，则该RPC就结束了
  if (tablet_invoker_.Done(&new_status)) {
    ...
  }
}
```

RPC超时事件的主要处理逻辑是TabletInvoker::Done，如果RPC请求的deadline(RpcRetrier::deadline_)还没有到，则尝试重新发送RPC请求，否则将当前的Tablet标记为failed状态并且返回该RPC失败。

重新发送RPC的主要逻辑由RpcRetrier负责，RpcRetrier包含一个状态机，初始是RpcRetrierState::kIdle状态，在将重试的RPC请求提交给Messenger调度执行之前，会切换为RpcRetrierState::kScheduling，然后提交给Messenger调度执行，若成功提交给Messenger调度执行，则切换到RpcRetrierState::kWaiting，若没有成功提交给Messenger调度执行，则切换到RpcRetrierState::kFinished，如果成功提交给Messenger调度执行，则在真正执行RPC发送之前，会切换到RpcRetrierState::kRunning，当真正发送了RPC之后，则切换回RpcRetrierState::kIdle。
```
bool TabletInvoker::Done(Status* status) {
  # 如果status是Aborted或者RpcRetrier的状态是finished(即该RPC请求已经结束了，不再重试)
  if (status->IsAborted() || retrier_->finished()) {
    return true;
  }

  # status->ok()是否为true，取决于AsyncRpc::Finished的调用时机，如果是
  # Connection::HandleTimeout -> ... -> AsyncRpc::Finished，则此时的status->ok()为true，
  # 如果是在TabletInvoker::Done，TabletInvoker::Execute，RpcRetrier::DoRetry等方法中被
  # 调用，则status->ok()为false，则这些方法中可能是以Rpc::Finished()或者
  # RpcCommand::Finished()等形式调用
  # 
  # 对于超时的情况，在RpcRetrier::HandleResponse中不会被处理，但是当RpcRetrier::HandleResponse
  # 返回时，会将status设置为OutboundCall::status_
  if (status->ok() && retrier_->HandleResponse(command_, status)) {
    return false;
  }

  # 至此，对于RPC超时的情况，status为TimedOut，下面只分析status为Timedout情况下的逻辑
  
  # RPC超时，且RpcRetrier的deadline时间还没有到达的情况下，尝试重新发送
  if (... ||
      (status->IsTimedOut() && CoarseMonoClock::Now() < retrier_->deadline())) {
    ...

    if (status->IsIllegalState() || TabletNotFoundOnTServer(rsp_err, *status)) {
      return !FailToNewReplica(*status, rsp_err).ok();
    } else {
      # 如果RPC超时，会进入这里
      #
      # 稍微delay一段时间之后，尝试重新发送，见RpcRetrier::DelayedRetry
      tserver::TabletServerDelay delay(*status);
      auto retry_status = delay.value().Initialized()
          ? retrier_->DelayedRetry(command_, *status, delay.value())
          : retrier_->DelayedRetry(command_, *status);
      
      # 如果重发失败，则会再次调用RpcCommand::Finished，在当前上下文中实际上就是
      # AsyncRpc::Finished
      if (!retry_status.ok()) {
        command_->Finished(retry_status);
      }
    }
    
    return false;
  }

  # RPC超时，且RpcRetrier的deadline时间到达的情况下，会进入这里
  if (!status->ok()) {
    if (status->IsTimedOut()) {
      # 对于RPC超时的情况，将当前的Tablet标记为failed状态
      if (tablet_ != nullptr && current_ts_ != nullptr) {
        tablet_->MarkReplicaFailed(current_ts_, *status);
      }
    }
    
    # 当前的RPC失败，在RPC response中设置status，error message，pg error code
    # 和transaction error code等
    rpc_->Failed(*status);
  }

  return true;
}

Status RpcRetrier::DoDelayedRetry(RpcCommand* rpc, const Status& why_status) {
  # 修改重试次数
  attempt_num_++;

  # RpcRetrier包含一个状态机，初始是RpcRetrierState::kIdle状态，在提交给Messenger调度执行
  # 之前，会切换为RpcRetrierState::kScheduling，然后提交给Messenger调度执行，若成功提交给
  # Messenger调度执行，则切换到RpcRetrierState::kWaiting，若没有成功提交给Messenger调度执
  # 行，则切换到RpcRetrierState::kFinished，如果成功提交给Messenger调度执行，在真正执行
  # 发送之前，会切换到RpcRetrierState::kRunning，当真正执行发送之后，则切换回
  # RpcRetrierState::kIdle
  
  # 先从kIdle切换到kScheduling，表示正在尝试调度
  RpcRetrierState expected_state = RpcRetrierState::kIdle;
  while (!state_.compare_exchange_strong(expected_state, RpcRetrierState::kScheduling)) {
    # 如果RpcRetrier当前的状态已经不是kIdle，而是kFinished，则会返回，错误码为IllegalState
    if (expected_state == RpcRetrierState::kFinished) {
      auto result = STATUS_FORMAT(IllegalState, "Retry of finished command: $0", rpc);
      LOG(WARNING) << result;
      return result;
    }
    
    # 如果RpcRetrier当前的状态已经不是kIdle，而是kWaiting，则会返回，错误码为IllegalState
    if (expected_state == RpcRetrierState::kWaiting) {
      auto result = STATUS_FORMAT(IllegalState, "Retry of already waiting command: $0", rpc);
      LOG(WARNING) << result;
      return result;
    }
  }

  # 提交给Messenger进行调度，如果被成功调度，则会返回一个有效的task_id，调度成功的情况下，
  # RpcRetrier::DoRetry会被执行
  auto retain_rpc = rpc->shared_from_this();
  task_id_ = messenger_->ScheduleOnReactor(
      std::bind(&RpcRetrier::DoRetry, this, rpc, _1),
      retry_delay_ + MonoDelta::FromMilliseconds(RandomUniformInt(0, 4)),
      SOURCE_LOCATION(), messenger_);

  expected_state = RpcRetrierState::kScheduling;
  
  # 如果调度失败，则返回，错误码为Aborted
  if (task_id_.load(std::memory_order_acquire) == kInvalidTaskId) {
    auto result = STATUS_FORMAT(Aborted, "Failed to schedule: $0", rpc);
    LOG(WARNING) << result;
    CHECK(state_.compare_exchange_strong(
        expected_state, RpcRetrierState::kFinished, std::memory_order_acq_rel));
    return result;
  }
  
  # 至此，已经成功调度，将状态从kScheduling切换到kWaiting
  CHECK(state_.compare_exchange_strong(
      expected_state, RpcRetrierState::kWaiting, std::memory_order_acq_rel));
  return Status::OK();
}

void RpcRetrier::DoRetry(RpcCommand* rpc, const Status& status) {
  auto retain_rpc = rpc->shared_from_this();

  # 首先将状态从kWaiting切换到kRunning
  RpcRetrierState expected_state = RpcRetrierState::kWaiting;
  bool run = state_.compare_exchange_strong(expected_state, RpcRetrierState::kRunning);
  
  # 如果当前状态是kScheduling，则进入busy wait，直到成功切换到kRunning状态，或者虽然没有
  # 成功切换到kRunning状态，但是当前状态不是kScheduling
  while (!run && expected_state == RpcRetrierState::kScheduling) {
    expected_state = RpcRetrierState::kWaiting;
    run = state_.compare_exchange_strong(expected_state, RpcRetrierState::kRunning);
    if (run) {
      break;
    }
    std::this_thread::sleep_for(1ms);
  }
  
  
  task_id_ = kInvalidTaskId;
  if (!run) {
    # 没有成功切换到kRunning状态，但是当前状态不是kScheduling，则Abort
    rpc->Finished(STATUS_FORMAT(
        Aborted, "$0 aborted: $1", rpc->ToString(), yb::rpc::ToString(expected_state)));
    return;
  }
  
  
  Status new_status = status;
  if (new_status.ok()) {
    # 如果RpcRetrier::deadline_已经过期，结束当前的重发操作
    if (deadline_ != CoarseTimePoint::max()) {
      auto now = CoarseMonoClock::Now();
      if (deadline_ < now) {
        string err_str = Format(
          "$0 passed its deadline $1 (passed: $2)", *rpc, deadline_, now - start_);
        if (!last_error_.ok()) {
          SubstituteAndAppend(&err_str, ": $0", last_error_.ToString());
        }
        new_status = STATUS(TimedOut, err_str);
      }
    }
  }
  
  if (new_status.ok()) {
    # 重新发送RPC
    controller_.Reset();
    
    # 在当前上下文中，会进入AsyncRpc::SendRpc逻辑
    rpc->SendRpc();
  } else {
    # 否则，再次调用WriteRpc::Finished进行处理
    if (new_status.IsServiceUnavailable()) {
      new_status = STATUS_FORMAT(Aborted, "Aborted because of $0", new_status);
    }
    rpc->Finished(new_status);
  }
  
  # 更新RpcRetrier的状态为kIdle
  expected_state = RpcRetrierState::kRunning;
  state_.compare_exchange_strong(expected_state, RpcRetrierState::kIdle);
}
```

## Server端的RPC处理逻辑
对于Server端来说，当接收到重发的RPC请求的时候，需要至少提供以下保证：
- 如果之前接收到的关于相同RPC的请求已经成功执行，则无需再度执行；
- 如果之前接收到的关于相同RPC请求正在执行，则当前接收到的RPC无需执行，而是等待之前接收到的关于相同RPC请求执行完毕后一并响应；

### 相关的数据结构
```
class RetryableRequests::Impl {
  # 来自不同client的请求被分别保存在不同的ClientRetryableRequests中
  std::unordered_map<ClientId, ClientRetryableRequests, ClientIdHash> clients_;
  ...
}

struct ClientRetryableRequests {
  # 正在处于running状态的请求集合，表示已经提交给raft进行复制，但是尚未复制完成的请求的集合
  RunningRetryableRequests running;
  # 已经完成raft复制的请求的集合
  ReplicatedRetryableRequestRanges replicated;
  # 处于running状态的最小的request id
  RetryableRequestId min_running_request_id = 0;
  RestartSafeCoarseTimePoint empty_since;
};

# 这是一个multi index container，其中保存的是RunningRetryableRequest，但是对container中的
# 每个元素建立了2个索引，分别是基于RunningRetryableRequest::request_id建立的hashed_unique
# 索引和基于RunningRetryableRequest::op_id建立的ordered_unique索引
typedef boost::multi_index_container <
    RunningRetryableRequest,
    boost::multi_index::indexed_by <
        boost::multi_index::hashed_unique <
            boost::multi_index::tag<RequestIdIndex>,
            boost::multi_index::member <
                RunningRetryableRequest, RetryableRequestId, &RunningRetryableRequest::request_id
            >
        >,
        boost::multi_index::ordered_unique <
            boost::multi_index::tag<OpIdIndex>,
            boost::multi_index::member <
                RunningRetryableRequest, yb::OpId, &RunningRetryableRequest::op_id
            >
        >
    >
> RunningRetryableRequests;

# 这是一个multi index container，其中保存的是ReplicatedRetryableRequestRange，但是对
# container中的每个元素建立了2个索引，分别是基于ReplicatedRetryableRequestRange::last_id
# 建立的ordered_unique索引和基于ReplicatedRetryableRequestRange::min_op_id建立的
# ordered_unique索引
typedef boost::multi_index_container <
    ReplicatedRetryableRequestRange,
    boost::multi_index::indexed_by <
        boost::multi_index::ordered_unique <
            boost::multi_index::tag<LastIdIndex>,
            boost::multi_index::member <
                ReplicatedRetryableRequestRange, RetryableRequestId,
                &ReplicatedRetryableRequestRange::last_id
            >
        >,
        boost::multi_index::ordered_unique <
            boost::multi_index::tag<OpIdIndex>,
            boost::multi_index::member <
                ReplicatedRetryableRequestRange, yb::OpId,
                &ReplicatedRetryableRequestRange::min_op_id
            >
        >
    >
> ReplicatedRetryableRequestRanges;

struct RunningRetryableRequest {
  RetryableRequestId request_id;
  yb::OpId op_id;
  RestartSafeCoarseTimePoint time;
  # 用于保存具有相同request id的不同ConsensusRound，对于RPC重发的请求，它们具有相同
  # 的request id，但是都会对应不同的ConsensusRound
  mutable std::vector<ConsensusRoundPtr> duplicate_rounds;

  RunningRetryableRequest(
      RetryableRequestId request_id_, const OpId& op_id_, RestartSafeCoarseTimePoint time_)
      : request_id(request_id_), op_id(yb::OpId::FromPB(op_id_)), time(time_) {}
};

# 用于记录request id介于[first_id, last_id]之间的已经Replicated的请求
struct ReplicatedRetryableRequestRange {
  mutable RetryableRequestId first_id;
  RetryableRequestId last_id;
  yb::OpId min_op_id;
  mutable RestartSafeCoarseTimePoint min_time;
  mutable RestartSafeCoarseTimePoint max_time;

  ReplicatedRetryableRequestRange(RetryableRequestId id, const yb::OpId& op_id,
                              RestartSafeCoarseTimePoint time)
      : first_id(id), last_id(id), min_op_id(op_id), min_time(time),
        max_time(time) {}
};
```

### Server端与RPC请求超时处理相关的主要逻辑
Server端与RPC请求超时处理相关的主要逻辑体现在下面的调用链：
```
RaftConsensus::ReplicateBatch
    RaftConsensus::AppendNewRoundsToQueueUnlocked
        ReplicaState::AddPendingOperation
            RetryableRequests::Register
                RetryableRequests::Impl::Register

class RetryableRequests::Impl {
  ...
  
  bool Register(const ConsensusRoundPtr& round, RestartSafeCoarseTimePoint entry_time) {
    auto data = ReplicateData::FromMsg(*round->replicate_msg());
    if (!data) {
      return true;
    }

    # 根据RPC消息中的client ID找到对应的ClientRetryableRequests
    ClientRetryableRequests& client_retryable_requests = clients_[data.client_id()];

    # 关于RPC请求中的min running request id，请参考"Client端对是如何给RPC请求赋值
    # request id和min running request id的"
    #
    # 如果data.write_request().min_running_request_id()比client_retryable_requests中
    # 记录的min_running_request_id大，则从client_retryable_requests::replicated
    # 中移除所有比data.write_request().min_running_request_id()小的请求，并且将
    # client_retryable_requests中记录的min_running_request_id更新为
    # data.write_request().min_running_request_id()
    CleanupReplicatedRequests(
        data.write_request().min_running_request_id(), &client_retryable_requests);

    # 关于RPC请求中的request id，请参考"Client端对是如何给RPC请求赋值request id和
    # min running request id的"
    #
    # 如果data.request_id()比client_retryable_requests中记录的min_running_request_id
    # 小，则直接返回错误
    if (data.request_id() < client_retryable_requests.min_running_request_id) {
      round->NotifyReplicationFinished(
          STATUS_FORMAT(
              Expired, "Request id $0 is below than min running $1", data.request_id(),
              client_retryable_requests.min_running_request_id),
          round->bound_term(), nullptr /* applied_op_ids */);
      return false;
    }

    auto& replicated_indexed_by_last_id = client_retryable_requests.replicated.get<LastIdIndex>();
    auto it = replicated_indexed_by_last_id.lower_bound(data.request_id());
    
    # 如果当前RPC请求对应的request id已经存在于client_retryable_requests::replicated中，
    # 则认为这是一个duplicate请求，直接返回错误
    if (it != replicated_indexed_by_last_id.end() && it->first_id <= data.request_id()) {
      round->NotifyReplicationFinished(
          STATUS(AlreadyPresent, "Duplicate request"), round->bound_term(),
          nullptr /* applied_op_ids */);
      return false;
    }

    # 如果client_retryable_requests::running中已经存在关于相同request id的RPC请求，
    # 则认为这是一个重发请求，添加到RunningRetryableRequest::duplicate_rounds中
    auto& running_indexed_by_request_id = client_retryable_requests.running.get<RequestIdIndex>();
    auto emplace_result = running_indexed_by_request_id.emplace(
        data.request_id(), round->replicate_msg()->id(), entry_time);
    if (!emplace_result.second) {
      emplace_result.first->duplicate_rounds.push_back(round);
      return false;
    }

    VLOG_WITH_PREFIX(4) << "Running added " << data;
    if (running_requests_gauge_) {
      running_requests_gauge_->Increment();
    }

    return true;
  }
  
  void CleanupReplicatedRequests(
      RetryableRequestId new_min_running_request_id,
      ClientRetryableRequests* client_retryable_requests) {
    auto& replicated_indexed_by_last_id = client_retryable_requests->replicated.get<LastIdIndex>();
    
    # 如果new_min_running_request_id比client_retryable_requests中记录的
    # min_running_request_id大，则从client_retryable_requests::replicated
    # 中移除所有比new_min_running_request_id小的请求，并且将
    # client_retryable_requests中记录的min_running_request_id更新为
    # new_min_running_request_id
    if (new_min_running_request_id > client_retryable_requests->min_running_request_id) {
      auto it = replicated_indexed_by_last_id.lower_bound(new_min_running_request_id);
      if (it != replicated_indexed_by_last_id.end() &&
          it->first_id < new_min_running_request_id) {
        it->first_id = new_min_running_request_id;
      }
      
      // Remove all intervals that has ids below write_request.min_running_request_id().
      replicated_indexed_by_last_id.erase(replicated_indexed_by_last_id.begin(), it);
      client_retryable_requests->min_running_request_id = new_min_running_request_id;
    }
  }  
```

### Client端对是如何给RPC请求赋值request id和min running request id的
Client端为每个Tablet维护了：
- Tablet::request_id_seq：一个单调递增的request id seq
    - 每当创建一个关于给定Tablet的WriterRpc请求的时候，就为之分配一个request id
- Tablet::running_requests: 已经发送，但尚未完成(完成，可能成功，也可能失败，但是完成之后，RPC就被销毁了)的RPC请求
    - 每当创建一个关于给定Tablet的WriterRpc请求的时候，就会将为之分配的request id添加到Tablet::running_requests中
    - 每当一个WriteRpc请求完成时，就将该RPC请求所对应的request id从相应Tablet的running_requests集合中移除
```
class YBClient::Data {
  ...
  
  struct TabletRequests {
    RetryableRequestId request_id_seq = 0;
    std::set<RetryableRequestId> running_requests;
  };

  simple_spinlock tablet_requests_mutex_;
  std::unordered_map<TabletId, TabletRequests> tablet_requests_;
  
  ...
}
```

每当创建一个关于给定Tablet的WriterRpc请求的时候，都会为之分配一个request id，并且将当前RPC请求对应的min running request id设置为RPC请求所关联的Tablet中尚未完成的RPC请求中具有最小request id的那个RPC请求所对应的request id。
```
WriteRpc::WriteRpc(AsyncRpcData* data, MonoDelta timeout)
    : AsyncRpcBase(data, YBConsistencyLevel::STRONG, timeout) {
  ...

  const auto& client_id = batcher_->client_id();
  if (!client_id.IsNil() && FLAGS_detect_duplicates_for_retryable_requests) {
    auto temp = client_id.ToUInt64Pair();
    req_.set_client_id1(temp.first);
    req_.set_client_id2(temp.second);
    # 获取当前RPC请求对应的request id，以及它所看到的关于特定tablet的尚未完成的RPC请求
    # 中具有最小request id的那个请求对应的request id
    auto request_pair = batcher_->NextRequestIdAndMinRunningRequestId(data->tablet->tablet_id());
    # 将获取到的request id设置给当前的RPC请求
    req_.set_request_id(request_pair.first);
    # 将获取到的min running request id设置给当前的RPC请求
    req_.set_min_running_request_id(request_pair.second);
  }
}

std::pair<RetryableRequestId, RetryableRequestId> YBClient::NextRequestIdAndMinRunningRequestId(
    const TabletId& tablet_id) {
  std::lock_guard<simple_spinlock> lock(data_->tablet_requests_mutex_);
  # 找到对应的tablet
  auto& tablet = data_->tablet_requests_[tablet_id];
  # 从该tablet中分配一个递增的request id
  auto id = tablet.request_id_seq++;
  # 将当前RPC请求对应的request id加入到该tablet对应的running_requests集合中
  tablet.running_requests.insert(id);
  # tablet.running_requests.begin()将返回当前tablet对应的min running request id
  return std::make_pair(id, *tablet.running_requests.begin());
}
```

每当一个RPC请求完成时，就将该RPC请求所对应的request id从相应Tablet的running_requests集合中移除：
```
AsyncRpc::Finished
    # AsyncRpc::Finished会调用TabletInvoker::Done，而WriteRpc的析构只会在
    # TabletInvoker::Done返回true的情况下被调用，TabletInvoker::Done返回true
    # 表示该RPC请求结束了
    WriteRpc::~WriteRpc
        Batcher::RequestFinished
            YBClient::RequestFinished
            
            
void YBClient::RequestFinished(const TabletId& tablet_id, RetryableRequestId request_id) {
  std::lock_guard<simple_spinlock> lock(data_->tablet_requests_mutex_);
  # 找到对应的Tablet
  auto& tablet = data_->tablet_requests_[tablet_id];
  # 从Tablet对应的running_requests集合中找到并移除之
  auto it = tablet.running_requests.find(request_id);
  if (it != tablet.running_requests.end()) {
    tablet.running_requests.erase(it);
  } else {
    ...
  }
}
```

### ClientRetryableRequests::running中的请求和ClientRetryableRequests::replicated请求的状态变迁
在RetryableRequests::Impl::Register中，会根据Write请求(通过ConsensusRound可以获取之)对应的request id等信息生成一个RunningRetryableRequest，并添加到client_retryable_requests.running中，如果已经存在具有相同的request id的RunningRetryableRequest，则通过RunningRetryableRequest::duplicate_rounds来保存所有具有相同的request id的请求。
```
class RetryableRequests::Impl {
  bool Register(const ConsensusRoundPtr& round, RestartSafeCoarseTimePoint entry_time) {
    auto data = ReplicateData::FromMsg(*round->replicate_msg());

    ClientRetryableRequests& client_retryable_requests = clients_[data.client_id()];

    ...

    # 将WriteRpc请求对应的request id添加到client_retryable_requests.running中
    auto& running_indexed_by_request_id = client_retryable_requests.running.get<RequestIdIndex>();
    auto emplace_result = running_indexed_by_request_id.emplace(
        data.request_id(), round->replicate_msg()->id(), entry_time);
        
    # 如果已经存在具有相同的request id的RunningRetryableRequest，则通过
    # RunningRetryableRequest::duplicate_rounds来保存所有具有相同的request id的请求
    if (!emplace_result.second) {
      emplace_result.first->duplicate_rounds.push_back(round);
      return false;
    }

    return true;
  }
}
```

如果成功完成raft复制，会从ClientRetryableRequests::running中移除相应的RunningRetryableRequest，如果RunningRetryableRequest::duplicate_rounds不为空，则向RunningRetryableRequest::duplicate_rounds中所有的请求发送响应，然后生成ReplicatedRetryableRequestRange，并添加到ClientRetryableRequests::replicated中。

如果未能成功完成raft复制，则会从ClientRetryableRequests::running中移除相应的RunningRetryableRequest，如果RunningRetryableRequest::duplicate_rounds不为空，则向RunningRetryableRequest::duplicate_rounds中所有的请求发送响应。

```
# 成功完成raft复制的情况
PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
RaftConsensus::UpdateMajorityReplicated
ReplicaState::UpdateMajorityReplicatedUnlocked
ReplicaState::AdvanceCommittedOpIdUnlocked
ReplicaState::ApplyPendingOperationsUnlocked
ReplicaState::NotifyReplicationFinishedUnlocked
RetryableRequests::ReplicationFinished
RetryableRequests::Impl::ReplicationFinished

# 未能成功完成raft复制的情况
ReplicaState::AbortOpsAfterUnlocked/ReplicaState::CancelPendingOperations
ReplicaState::NotifyReplicationFinishedUnlocked
RetryableRequests::ReplicationFinished
RetryableRequests::Impl::ReplicationFinished

class RetryableRequests::Impl {
  void ReplicationFinished(
      const ReplicateMsg& replicate_msg, const Status& status, int64_t leader_term) {
    auto data = ReplicateData::FromMsg(replicate_msg);
    
    # 找到请求对应的client_retryable_requests
    auto& client_retryable_requests = clients_[data.client_id()];
    auto& running_indexed_by_request_id = client_retryable_requests.running.get<RequestIdIndex>();
    
    # 从client_retryable_requests::running中查找对应的request id
    auto running_it = running_indexed_by_request_id.find(data.request_id());
    
    static Status duplicate_write_status = STATUS(AlreadyPresent, "Duplicate request");
    auto status_for_duplicate = status.ok() ? duplicate_write_status : status;
    
    # 响应在duplicate_rounds中的请求
    for (const auto& duplicate : running_it->duplicate_rounds) {
      duplicate->NotifyReplicationFinished(status_for_duplicate, leader_term,
                                           nullptr /* applied_op_ids */);
    }
    
    # 从client_retryable_requests::running中移除相应的请求
    running_indexed_by_request_id.erase(running_it);
    if (running_requests_gauge_) {
      running_requests_gauge_->Decrement();
    }

    if (status.ok()) {
      # 如果成功的完成了复制，则将请求添加到client_retryable_requests::replicated中
      AddReplicated(
          yb::OpId::FromPB(replicate_msg.id()), data, entry_time, &client_retryable_requests);
    }
  }
  
  void AddReplicated(yb::OpId op_id, const ReplicateData& data, RestartSafeCoarseTimePoint time,
                     ClientRetryableRequests* client) {
    # 获取当前请求对应的request id
    auto request_id = data.request_id();
    auto& replicated_indexed_by_last_id = client->replicated.get<LastIdIndex>();
    
    # 在ClientRetryableRequests::replicated中查找不小于给定request_id的第一个元素，
    # 返回的ReplicatedRetryableRequestRange一定满足：
    # ReplicatedRetryableRequestRange::last_id >= request_id
    auto request_it = replicated_indexed_by_last_id.lower_bound(request_id);
    
    # 如果找到了这样的元素，并且ReplicatedRetryableRequestRange::first_id不大于request_id，
    # 则表明给定的request_id已经被包含在ReplicatedRetryableRequestRange中了
    if (request_it != replicated_indexed_by_last_id.end() && request_it->first_id <= request_id) {
      LOG_WITH_PREFIX(DFATAL) << "Request already replicated: " << data;
      return;
    }

    # 下面尝试将request_id添加到ClientRetryableRequests::replicated中
    
    
    if (request_it != replicated_indexed_by_last_id.end() &&
        request_it->first_id == request_id + 1) {
      # 如果满足request_it->first_id == request_id + 1，则将当前的request_id添加到
      # request_it所代表的ReplicatedRetryableRequestRange中
      op_id = std::min(request_it->min_op_id, op_id);
      request_it->InsertTime(time);
      // If previous range is right before this id, then we could just join those ranges.
      if (!TryJoinRanges(request_it, op_id, &replicated_indexed_by_last_id)) {
        --(request_it->first_id);
        UpdateMinOpId(request_it, op_id, &replicated_indexed_by_last_id);
      }
      return;
    }

    # 如果满足request_it->last_id + 1 == request_id，则尝试将request_id添加到
    # request_it所代表的ReplicatedRetryableRequestRange中
    if (TryJoinToEndOfRange(request_it, op_id, request_id, time, &replicated_indexed_by_last_id)) {
      return;
    }

    # 根据request_id单独生成一个ReplicatedRetryableRequestRange，并添加到
    # ClientRetryableRequests::replicated中
    client->replicated.emplace(request_id, op_id, time);
    if (replicated_request_ranges_gauge_) {
      replicated_request_ranges_gauge_->Increment();
    }
  }
}
```

在RetryableRequests::Impl::Register -> RetryableRequests::Impl::CleanupReplicatedRequests中，会从ClientRetryableRequests::replicated中移除所有小于给定min running request id的请求。
```
class RetryableRequests::Impl {
  bool Register(const ConsensusRoundPtr& round, RestartSafeCoarseTimePoint entry_time) {
    auto data = ReplicateData::FromMsg(*round->replicate_msg());

    ClientRetryableRequests& client_retryable_requests = clients_[data.client_id()];
    
    # 从ClientRetryableRequests::replicated中移除所有小于
    # data.write_request().min_running_request_id()的请求
    CleanupReplicatedRequests(
        data.write_request().min_running_request_id(), &client_retryable_requests);
    ...

    return true;
  }
}
```

## WriteRpc，RpcRetrier，RpcController和OutboundCall之间的关系
### WriteRpc和RpcRetrier之间的关系
一个WriteRpc对应一个RpcRetrier，通过WriteRpc的继承关系可以很容易看到这一点。
```
class WriteRpc : public AsyncRpcBase<tserver::WriteRequestPB, tserver::WriteResponsePB> {

}

class AsyncRpcBase : public AsyncRpc {

}

class AsyncRpc : public rpc::Rpc, public TabletRpc {

}

class Rpc : public RpcCommand {
  ...
  RpcRetrier retrier_;
}
```

### RpcRetrier和RpcController之间的关系
一个RpcRetrier对应一个RpcController，从RpcRetrier的类定义中可以看到这一点。又因为一个WriteRpc对应一个RpcRetrier，所以WriteRpc，RpcRetrier，以及RpcController，这三者之间都是一一对应的。
```
class RpcRetrier {
 public:
  RpcRetrier(CoarseTimePoint deadline, Messenger* messenger, ProxyCache *proxy_cache);

  ~RpcRetrier();

  ...

  
  int attempt_num_ = 1;

  const CoarseTimePoint deadline_;

  RpcController controller_;

  std::atomic<RpcRetrierState> state_{RpcRetrierState::kIdle};
};
```

### RpcController和OutboundCall之间的关系
从RpcController的类定义来看，一个RpcController中会包含一个OutboundCall指针，那么在RpcController的整个生命周期中是否就只对应一个OutboundCall呢？还需要进一步分析。
```
class RpcController {
  ...
  
  MonoDelta timeout_;
  OutboundCallPtr call_;
  
  ...
};
```

在发送WriteRpc的时候会生成一个OutboundCall/LocalOutboundCall并赋值给RpcController::call_。
```
AsyncRpc::SendRpc   
TabletInvoker::Execute
# AsyncRpcBase是WriteRpc的基类，AsyncRpc则是AsyncRpcBase的基类
AsyncRpcBase<Req, Resp>::SendRpcToTserver
AsyncRpc::SendRpcToTserver
WriteRpc::CallRemoteMethod
TabletServerServiceProxy::WriteAsync
Proxy::AsyncRequest
Proxy::DoAsyncRequest

void Proxy::DoAsyncRequest(const RemoteMethod* method,
                           const google::protobuf::Message& req,
                           google::protobuf::Message* resp,
                           RpcController* controller,
                           ResponseCallback callback,
                           bool force_run_callback_on_reactor) {
  CHECK(controller->call_.get() == nullptr) << "Controller should be reset";
  is_started_.store(true, std::memory_order_release);

  # 设置RpcController::call_
  controller->call_ =
      call_local_service_ ?
      std::make_shared<LocalOutboundCall>(method,
                                          outbound_call_metrics_,
                                          resp,
                                          controller,
                                          &context_->rpc_metrics(),
                                          std::move(callback)) :
      std::make_shared<OutboundCall>(method,
                                     outbound_call_metrics_,
                                     resp,
                                     controller,
                                     &context_->rpc_metrics(),
                                     std::move(callback),
                                     GetCallbackThreadPool(
                                         force_run_callback_on_reactor,
                                         controller->invoke_callback_mode()));

  ...                                         
}                                 
```

当进行RPC请求重发的时候，会在RpcRetrier::DoRetry中会首先调用RpcController::Reset将RpcController::call_重置为null，然后再调用AsyncRpc::SendRpc发送该RPC请求。根据前面对AsyncRpc::SendRpc -> ... -> Proxy::DoAsyncRequest的分析可知，其中会重新创建一个OutboundCall/LocalOutboundCall并赋值给RpcController::call_。
```
# 在WriteRpc::CallRemoteMethod中指定AsyncRpc::Finished为接收到RPC响应或者超时事件发生时的回调
AsyncRpc::Finished
TabletInvoker::Done
RpcRetrier::DelayedRetry
RpcRetrier::DoDelayedRetry
RpcRetrier::DoRetry
RpcController::Reset + AsyncRpc::SendRpc

void RpcController::Reset() {
  std::lock_guard<simple_spinlock> l(lock_);
  if (call_) {
    CHECK(finished());
  }
  
  # 将RpcController::call_重置为null
  call_.reset();
}
```

经过上面的分析可知，RPC请求重发过程中，会在每次重发的时候创建一个新的OutboundCall/LocalOutboundCall，并赋值给RpcController，所以在RpcController的整个生命周期中，它和OutboundCall之间可能是1对多的关系，但是在任意时刻，RpcController和OutboundCall总是1对1的关系。

### 总结
每个WriteRpc请求都包含一个request id，WriteRpc请求的每次发送(第一次发送和后续重发)都会生成一个OutboundCall/LocalOutboundCall，并且OutboundCall/LocalOutboundCall会对应一个call id。

每个WriteRpc请求都唯一对应一个RpcRetrier和RpcController，RpcRetrier主要用于WriteRpc请求重发，而RpcController中主要包含该WriteRpc请求的每次发送所用到的OutboundCall/LocalOutboundCall，以及该OutboundCall/LocalOutboundCall对应的超时时间【注：OutboundCall/LocalOutboundCall的超时时间和WriteRpc请求的超时时间是不同的，WriteRpc请求的超时时间通常几倍于OutboundCall/LocalOutboundCall的超时时间，在WriteRpc请求超时之前，是可以多次重发该WriteRpc请求的】