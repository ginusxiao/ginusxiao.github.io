# 提纲
[toc]

## TcpStream的构建
TcpStream会关联到一个Socket。
```
TcpStream::TcpStream(const StreamCreateData& data)
    : socket_(std::move(*data.socket)),
      remote_(data.remote) {
  if (data.mem_tracker) {
    mem_tracker_ = MemTracker::FindOrCreateTracker("Sending", data.mem_tracker);
  }
}
```

## TcpStream的启动
在调用TcpStream::Start的时候，参数connect是这么设置的：如果这个TcpStream是在连接的Server侧启动，则设置为false，如果这个TcpStream是在连接的Client侧启动，则设置为true；参数loop被设置为Reator::loop_；参数context被设置为对应的Connection的ConnectionContext。
```
Status TcpStream::Start(bool connect, ev::loop_ref* loop, StreamContext* context) {
  context_ = context;
  # 如果该TcpStream是在连接的Server侧启动，则connected为true，否则为false
  connected_ = !connect;

  # 设置关联的socket的属性
  RETURN_NOT_OK(socket_.SetNoDelay(true));
  // These timeouts don't affect non-blocking sockets:
  RETURN_NOT_OK(socket_.SetSendTimeout(FLAGS_rpc_connection_timeout_ms * 1ms));
  RETURN_NOT_OK(socket_.SetRecvTimeout(FLAGS_rpc_connection_timeout_ms * 1ms));

  # 启动
  return DoStart(loop, connect);
}

Status TcpStream::DoStart(ev::loop_ref* loop, bool connect) {
  if (connect) {
    # 该TcpStream是作为连接的Client侧启动，则调用Connect主动建立连接
    auto status = socket_.Connect(remote_);
  }

  RETURN_NOT_OK(socket_.GetSocketAddress(&local_));
  log_prefix_.clear();

  # 设置Reactor::loop_关注Socket上的Read和Write事件，如果该TcpStream是在连接的
  # Server侧启动，则只关注Read事件，否则同时关注Read和Write事件
  io_.set(*loop);
  io_.set<TcpStream, &TcpStream::Handler>(this);
  int events = ev::READ | (!connected_ ? ev::WRITE : 0);
  io_.start(socket_.GetFd(), events);

  DVLOG_WITH_PREFIX(3) << "Starting, listen events: " << events << ", fd: " << socket_.GetFd();

  is_epoll_registered_ = true;

  if (connected_) {
    context_->Connected();
  }

  return Status::OK();
}
```

## 通过TcpStream发送数据
TcpStream::Send被用来发送数据，它会将本次待发送的数据添加到sending_队列中。
```
size_t TcpStream::Send(OutboundDataPtr data) {
  // In case of TcpStream handle is absolute index of data block, since stream start.
  // So it could be cacluated as index in sending_ plus number of data blocks that were already
  // transferred.
  size_t result = data_blocks_sent_ + sending_.size();

  # 将本次待发送的数据添加到sending_中
  sending_.emplace_back(std::move(data), mem_tracker_);
  # 更新总的待发送的字节数queued_bytes_to_send_
  queued_bytes_to_send_ += sending_.back().bytes_size();

  return result;
}
```

## TcpStream对于可读可写事件的处理入口
在TcpStream::Handler中，首先检查是否有可读事件，如果有，则处理可读事件；然后检查是否有可写事件，如果有，则处理可写事件；最后重新设置接下来该关注的事件。
```
void TcpStream::Handler(ev::io& watcher, int revents) {  // NOLINT
  Status status = Status::OK();

  # 对于可读事件的处理
  if (status.ok() && (revents & ev::READ)) {
    status = ReadHandler();
  }

  # 对于可写事件的处理
  if (status.ok() && (revents & ev::WRITE)) {
    bool just_connected = !connected_;
    if (just_connected) {
      connected_ = true;
      context_->Connected();
    }
    
    status = WriteHandler(just_connected);
  }

  if (status.ok()) {
    # 重新设置接下来该关注的事件
    UpdateEvents();
  } else {
    context_->Destroy(status);
  }
}

# 在每次有可读事件或者可写事件的时候被调用，用以更新接下来关注哪些事件
void TcpStream::UpdateEvents() {
  int events = 0;
  
  # 如果read_buffer还没有满，则继续关注可读事件
  if (!read_buffer_full_) {
    events |= ev::READ;
  }
  
  # 如果存在待发送的数据，则继续关注可写事件
  waiting_write_ready_ = !sending_.empty() || !connected_;
  if (waiting_write_ready_) {
    events |= ev::WRITE;
  }
  
  # 设置关注的事件集合
  if (events) {
    io_.set(events);
  }
}
```

## TcpStream对于可读事件的处理
```
Status TcpStream::ReadHandler() {
  context_->UpdateLastRead();

  # 如果能持续从Socket中读取数据并且对读取到的数据的处理没有出错，则循环不退出
  for (;;) {
    # 从Socket中读取数据，并填充到ReadBuffer中
    auto received = Receive();
    if (PREDICT_FALSE(!received.ok())) {
      return received.status();
    }
    
    # 没有读取到任何数据
    if (!received.get()) {
      return Status::OK();
    }
    
    // If we were not able to process next call exit loop.
    // If status is ok, it means that we just do not have enough data to process yet.
    # 在TryProcessReceived中处理下一个调用
    auto continue_receiving = TryProcessReceived();
    if (!continue_receiving.ok()) {
      return continue_receiving.status();
    }
    
    if (!continue_receiving.get()) {
      return Status::OK();
    }
  }
}

Result<bool> TcpStream::Receive() {
  # ReadBuffer()对应的是ConnectionContext的ReadBuffer，比如对于
  # CQLConnectionContext来说，就是CQLConnectionContext::read_buffer_，
  # ReadBuffer()用于存放读取到的数据，PrepareAppend()方法用于在
  # ReadBuffer()中准备一些用于存放读取到的数据的IOVecs
  auto iov = ReadBuffer().PrepareAppend();
  if (!iov.ok()) {
    if (iov.status().IsBusy()) {
      # ReadBuffer中没有可用的空间
      read_buffer_full_ = true;
      return false;
    }
    
    return iov.status();
  }
  
  # ReadBuffer还没满
  read_buffer_full_ = false;

  # 读取数据到PrepareAppend阶段返回的IOVecs中
  auto nread = socket_.Recvv(iov.get_ptr());
  if (!nread.ok()) {
    if (Socket::IsTemporarySocketError(nread.status())) {
      return false;
    }
    return nread.status();
  }

  # 当读取了*nread字节的数据到ReadBuffer()中之后，更新ReadBuffer()中的相关元信息，
  # 以便后续读取数据到ReadBuffer()时，可以将数据填充在正确的位置
  ReadBuffer().DataAppended(*nread);
  return *nread != 0;
}


Result<bool> TcpStream::TryProcessReceived() {
  auto& read_buffer = ReadBuffer();
  if (!read_buffer.ReadyToRead()) {
    # ReadBuffer中没有可用读取的数据，则返回
    return false;
  }

  # 处理从ReadBuffer中读取到的数据，这里的context_类型为Connection
  # 分析见Connection::ProcessReceived
  auto result = VERIFY_RESULT(context_->ProcessReceived(
      read_buffer.AppendedVecs(), ReadBufferFull(read_buffer.Full())));

  # 当从ReadBuffer中消费数据之后，更新相关元信息，并设置prepend_
  read_buffer.Consume(result.consumed, result.buffer);
  return true;
}

Result<ProcessDataResult> Connection::ProcessReceived(
    const IoVecs& data, ReadBufferFull read_buffer_full) {
  # 假设这里的context_类型是CQLConnectionContext，则分析见
  # 
  auto result = context_->ProcessCalls(shared_from_this(), data, read_buffer_full);
  return result;
}

Result<rpc::ProcessDataResult> CQLConnectionContext::ProcessCalls(
    const rpc::ConnectionPtr& connection, const IoVecs& data,
    rpc::ReadBufferFull read_buffer_full) {
  # BinaryCallParser::Parse
  return parser_.Parse(connection, data, read_buffer_full);
}
```

## TcpStream对于可写事件的处理
```
Status TcpStream::WriteHandler(bool just_connected) {
  # 设置Socket现在可写了
  waiting_write_ready_ = false;
  if (sending_.empty()) {
    # 没有数据需要发送
    LOG_IF_WITH_PREFIX(WARNING, !just_connected) <<
        "Got a ready-to-write callback, but there is nothing to write.";
    return Status::OK();
  }

  # 发送数据
  return DoWrite();
}

Status TcpStream::DoWrite() {
  # 如果连接没有成功，或者Socket现在不可写，或者还没有注册到事件主循环
  if (!connected_ || waiting_write_ready_ || !is_epoll_registered_) {
    DVLOG_WITH_PREFIX(5)
        << "connected_: " << connected_
        << " waiting_write_ready_: " << waiting_write_ready_
        << " is_epoll_registered_: " << is_epoll_registered_;
    return Status::OK();
  }

  # 将sending_中待发送的数据发送到Socket中
  while (!sending_.empty()) {
    # 根据待写的数据生成iovec数组，这里不一定会将sending_中的所有的数据
    # 都存放到iovec数组中，因为iovec数组的大小的最大值当前被设置为16
    iovec iov[kMaxIov];
    auto fill_result = FillIov(iov);

    if (!fill_result.only_heartbeats) {
      # 待发送的数据中不是只有heartbeats相关的数据
      context_->UpdateLastActivity();
    }

    # 将iov中的数据写入Socket中
    int32_t written = 0;
    auto status = fill_result.len != 0
        ? socket_.Writev(iov, fill_result.len, &written)
        : Status::OK();

    if (PREDICT_FALSE(!status.ok())) {
      # 发送失败
      if (!Socket::IsTemporarySocketError(status)) {
        # 不是临时的Socket错误
        return status;
      } else {
        # 临时的Socket错误
        return Status::OK();
      }
    }

    context_->UpdateLastWrite();

    # 更新剩余的尚未发送的数据的起始位置
    send_position_ += written;
    
    # 将已经发送的数据从sending_队列中移除
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
        context_->Transferred(data, Status::OK());
      }
    }
  }

  return Status::OK();
}
```

## 释疑
1. TcpStream中的sending_是一个双向队列，TcpStream::Send和TcpStream::DoWrite中都会用到它，但是没有使用锁，会不会存在问题呢？

不会存在问题，因为TcpStream::Send和TcpStream::DoWrite的使用都一定是在Reactor线程中，所以不需要锁。其中TcpStream::DoWrite的调用链是：TcpStream::Handler -> TcpStream::WriteHandler -> TcpStream::DoWrite，这是在Reactor的事件主循环中接收到可写事件时的处理逻辑；TcpStream::Send的调用链是：Connection::ProcessResponseQueue/Connection::QueueOutboundCall/Connection::QueueOutboundData/Connection::QueueOutboundDataBatch -> Connection::DoQueueOutboundData -> TcpStream::Send，在调用Connection::DoQueueOutboundData的时候，会断言“当前的线程一定是Reactor线程”。

