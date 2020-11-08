# 提纲
[toc]

## Connection中QueueOutboundData方法的说明
Connection::QueueOutboundData有可能在Reactor线程中被调用，也可能不在Reactor线程中被调用，两种情况下的处理不同：
- 如果是在Reactor线程中调用，则：
    - 首先，将outbound_data写入TcpStream的待发送的数据的队列中；
    - 然后，直接调用Connection::OutboundQueued -> TcpStream::TryWrite -> TcpStream::DoWrite将TcpStream的待发送的数据的队列中的数据发送到Socket中，**这里发送的很可能只是当前这一个outbound_data对应的数据**。
- 如果不是在Reactor线程中调用，则：
    - 首先，将outbound_data添加到Connection::outbound_data_to_process_集合中；
    - 然后，检查是否需要创建Connection::process_response_queue_task_，如果需要，则创建之；
    - 最后，检查是否需要提交Connection::process_response_queue_task_到Reactor的任务队列中，该任务用于执行Connection::outbound_data_to_process_集合中所有的outbound_data；
        - 当Connection::process_response_queue_task_被调度执行后，它会执行Connection::ProcessResponseQueue：
            - 将Connection::outbound_data_to_process_中的数据存放到Connection::outbound_data_being_processed_中；
            - 逐一遍历Connection::outbound_data_being_processed_中的每个outbound_data，并执行：
                - 将outbound_data写入TcpStream的待发送的数据的队列中；
            - 调用Connection::OutboundQueued -> TcpStream::TryWrite -> TcpStream::DoWrite将TcpStream的待发送的数据的队列中的数据发送到Socket中，**这里发送的可能是一批数据**。
```
void Connection::QueueOutboundData(OutboundDataPtr outbound_data) {
  # 如果当前线程就是Reactor的线程，则直接调用DoQueueOutboundData
  if (reactor_->IsCurrentThread()) {
    DoQueueOutboundData(std::move(outbound_data), /* batch */ false);
    return;
  }

  # 当前线程不是Reactor线程
  bool was_empty;
  {
    std::unique_lock<simple_spinlock> lock(outbound_data_queue_lock_);
    
    # 检查在添加当前这个outbound_data之前，outbound_data_to_process_是否为空
    was_empty = outbound_data_to_process_.empty();
    
    # 将outbound_data添加到outbound_data_to_process_集合中
    outbound_data_to_process_.push_back(std::move(outbound_data));
    
    # 如果本次添加之前，outbound_data_to_process_集合是空的，且没有创建
    # process_response_queue_task_，则创建之
    if (was_empty && !process_response_queue_task_) {
      process_response_queue_task_ =
          MakeFunctorReactorTask(std::bind(&Connection::ProcessResponseQueue, this),
                                 shared_from_this(), SOURCE_LOCATION());
    }
  }

  # 如果本次添加outbound_data之前，outbound_data_to_process_集合是空的，
  # 则提交process_response_queue_task_给Reactor，等待其调度，当
  # process_response_queue_task_被调度之后，会处理outbound_data_to_process_
  # 集合中的所有的outbound_data
  if (was_empty) {
    // TODO: what happens if the reactor is shutting down? Currently Abort is ignored.
    auto scheduled = reactor_->ScheduleReactorTask(process_response_queue_task_);
    LOG_IF_WITH_PREFIX(WARNING, !scheduled)
        << "Failed to schedule Connection::ProcessResponseQueue";
  }
}
```

Connection::DoQueueOutboundData一定是在Reactor线程中执行的，它会将outbound_data写入TcpStream的待发送的数据的队列中，然后检查是否是批量发送，如果不是批量发送，则直接调用Connection::OutboundQueued -> TcpStream::TryWrite -> TcpStream::DoWrite将TcpStream的待发送的数据的队列中的数据发送到Socket中。
```
size_t Connection::DoQueueOutboundData(OutboundDataPtr outbound_data, bool batch) {
  # 当前线程一定是Reactor线程
  DCHECK(reactor_->IsCurrentThread());

  ...
  
  # 将outbound_data写入TcpStream的待发送的数据的队列中(注意：并没有发送给Socket)
  auto result = stream_->Send(std::move(outbound_data));

  ...
  
  # 如果不是批量发送，则直接调用Connection::OutboundQueued -> TcpStream::TryWrite 
  # -> TcpStream::DoWrite将TcpStream的待发送的数据的队列中的数据发送到Socket中
  if (!batch) {
    OutboundQueued();
  }

  return result;
}
```

