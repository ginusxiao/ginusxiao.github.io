# 提纲
[toc]

## 性能测试
### 验证reactor数目对性能的影响
测试中固定value大小为80字节。

reactor数目 | 并发client数目 | 写线程数目 | value大小|性能
---|---|---|---|---
16|1|1|80|10000(1个reactor提供服务，cpu：30%)
16|2|2|80|20000(2个reactor提供服务，cpu：30%*2)
16|4|4|80|44000
16|8|8|80|80000(7个reactor提供服务)
16|16|16|80|130000(11个reactor提供服务)
16|32|32|80|170000(14个reactor提供服务)
16|64|64|80|190000(15个reactor提供服务)
16|128|128|80|200000(16个reactor提供服务，cpu：70%*16)
32|1|1|80|9500(1个reactor提供服务，cpu：30%)
32|2|2|80|20000(1个reactor提供服务，cpu：50%)
32|4|4|80|45000(4个reactor提供服务，cpu：30%*4)
32|8|8|80|80000(8个reactor提供服务，cpu：30%*6 + 25% + 15%)
32|16|16|80|135000(14个reactor提供服务，cpu：65% + 60% + 40%*3 + 30%*5 + 25% + 18% + 8% + 1%)
32|32|32|80|170000(21个reactor提供服务，cpu：5% - 65%)
32|64|64|80|190000(28个reactor提供服务，cpu：16% - 60%)
32|128|128|80|180000(32个reactor提供服务，cpu：40% - 80%)

### 验证value大小对性能的影响
测试中固定reactor数目为16.

reactor数目 | 并发client数目 | 写线程数目 | value大小|性能
---|---|---|---|---
16|1|1|80|10000(1个reactor提供服务，cpu：30%)
16|2|2|80|20000(2个reactor提供服务，cpu：30%*2)
16|4|4|80|44000
16|8|8|80|80000(7个reactor提供服务)
16|16|16|80|130000(11个reactor提供服务)
16|32|32|80|170000(14个reactor提供服务)
16|64|64|80|190000(15个reactor提供服务)
16|1|1|40|9500(1个reactor提供服务，cpu：30%)
16|2|2|40|22000(2个reactor提供服务，cpu：30%*2)
16|4|4|40|44000(2个reactor提供服务，cpu：70% + 30%)
16|8|8|40|80000(7个reactor提供服务，cpu：15% - 54%)
16|16|16|40|135000(10个reactor提供服务，cpu：16% - 70%)
16|32|32|40|170000(14个reactor提供服务，cpu：25% - 70%)
16|64|64|40|190000(16个reactor提供服务，cpu：25% - 75%)


### 验证写线程数目对性能的影响
测试中固定reactor数目为16，并发client数目为16，value大小为80.

reactor数目 | 并发client数目 | 写线程数目 | value大小|性能
---|---|---|---|---
16|16|1|80|10000(1个reactor提供服务，cpu：30%)
16|16|2|80|22000(2个reactor提供服务，cpu：30%*2)
16|16|4|80|45000(3个reactor提供服务，cpu：30%*2 + 45%)
16|16|8|80|82000(6个reactor提供服务，cpu：16% - 55% )
16|16|16|80|132000(10个reactor提供服务，cpu：17% - 85% )
16|16|32|80|180000(12个reactor提供服务，cpu：35% - 75% )
16|16|64|80|210000(11个reactor提供服务，cpu：70%*11 )
16|16|96|80|210000(9个reactor提供服务，cpu：75%*9 )
16|16|128|80|216000(11个reactor提供服务，cpu：70%*11 )
16|16|256|80|215000(11个reactor提供服务，cpu：70%*11 )

### 验证并发client数目对性能的影响
测试中固定reactor数目为16，写线程数目为64，value大小为80.

reactor数目 | 写线程数目 | 并发client数目 | value大小|性能
---|---|---|---|---
16|64|1|80|80000(1个reactor提供服务，cpu：99%以上)
16|64|2|80|120000(2个reactor提供服务，cpu：97%*2)
16|64|4|80|180000(4个reactor提供服务，cpu：93%*4)
16|64|8|80|200000(7个reactor提供服务，cpu：82%*7)
16|64|16|80|210000(11个reactor提供服务，cpu：65% - 75% )
16|64|32|80|200000(14个reactor提供服务，cpu：40% - 75% )
16|64|64|80|190000(16个reactor提供服务，cpu：25% - 70% )

### 验证Reactor数目对性能的影响2
测试中固定写线程数目为64，并发client数目为16，value大小为80.

reactor数目 | 写线程数目 | 并发client数目 | value大小|性能
---|---|---|---|---
1|64|16|80|100000(1个reactor提供服务，cpu：99%以上)
2|64|16|80|110000(2个reactor提供服务，cpu：97%*2)
4|64|16|80|155000(4个reactor提供服务，cpu：93%*3 + 83%)
8|64|16|80|195000(7个reactor提供服务，cpu：84%*6 + 75%)
16|64|16|80|210000(11个reactor提供服务，cpu：65% - 75% )
32|64|16|80|220000(13个reactor提供服务，cpu：62% - 71% )

****下面的测试中，去掉了CQLProcessor::ProcessCall中关于OPCODE的打印信息。**

reactor数目 | 写线程数目 | 并发client数目 | value大小|性能
---|---|---|---|---
1|64|16|80|130000 - 160000(1个reactor提供服务，cpu：99%以上)
2|64|16|80|160000 - 190000(2个reactor提供服务，cpu：98%*2)

****下面的测试中，去掉了CQLProcessor::ProcessCall中关于OPCODE的打印信息，同时对CQLServiceImpl::GetProcessor()方法进行了优化，去掉了CQLServiceImpl::GetProcessor()中的锁**
reactor数目 | 写线程数目 | 并发client数目 | value大小|性能
---|---|---|---|---
1|64|16|80|120000 - 130000(1个reactor提供服务，cpu：99%左右)
2|64|节点1上16个，节点2上16个|80|170000 - 200000(2个reactor提供服务，cpu：99%*2)
16|64|16|80|300000 - 360000



## Messenger::num_connections_to_server_
```
void Messenger::RegisterInboundSocket(...) {
  int idx = num_connections_accepted_.fetch_add(1) % num_connections_to_server_;
  Reactor *reactor = RemoteToReactor(remote, idx);
  reactor->RegisterInboundSocket(
      new_socket, remote, factory->Create(*receive_buffer_size), factory->buffer_tracker());
}
```

在Connection到Reactor的映射的地方，如果将Messenger::num_connections_to_server_调整的大一点(可以通过FLAGS_num_connections_to_server进行设置)，则可能会有更多的Reactor参与工作(或者说，不同的Connection尽可能分配到不同的Reactor)。

## Socket的receive_buffer_size和ConnectionContextFactory
```
void Messenger::RegisterInboundSocket(
    const ConnectionContextFactoryPtr& factory, Socket *new_socket, const Endpoint& remote) {
  # 可以通过FLAGS_socket_receive_buffer_size来设置Socket receive buffer size
  if (FLAGS_socket_receive_buffer_size) {
    WARN_NOT_OK(new_socket->SetReceiveBufferSize(FLAGS_socket_receive_buffer_size),
                "Set receive buffer size failed: ");
  }

  # 获取Socket对应的receive buffer size
  auto receive_buffer_size = new_socket->GetReceiveBufferSize();
  if (!receive_buffer_size.ok()) {
    LOG(WARNING) << "Register inbound socket failed: " << receive_buffer_size.status();
    return;
  }

  int idx = num_connections_accepted_.fetch_add(1) % num_connections_to_server_;
  Reactor *reactor = RemoteToReactor(remote, idx);
  # 在factory->Create(*receive_buffer_size)中用到了Socket receive buffer size，
  reactor->RegisterInboundSocket(
      new_socket, remote, factory->Create(*receive_buffer_size), factory->buffer_tracker());
}
```

在CQLServer中，Messenger::RegisterInboundSocket中用到的ConnectionContextFactory是啥呢?经过代码追溯，发现是在CQLServer::CQLServer中设置的：
```
CQLServer::CQLServer(const CQLServerOptions& opts,
                     boost::asio::io_service* io,
                     tserver::TabletServer* tserver)
    : RpcAndWebServerBase(
          "CQLServer", opts, "yb.cqlserver",
          MemTracker::CreateTracker(
              "CQL", tserver ? tserver->mem_tracker() : MemTracker::GetRootTracker(),
              AddToParent::kTrue, CreateMetrics::kFalse)),
      opts_(opts),
      timer_(*io, refresh_interval()),
      tserver_(tserver) {
  # 通过rpc::CreateConnectionContextFactory<CQLConnectionContext>来创建ConnectionContextFactory
  SetConnectionContextFactory(rpc::CreateConnectionContextFactory<CQLConnectionContext>(
      FLAGS_cql_rpc_memory_limit, mem_tracker()->parent()));
}
```

在rpc::CreateConnectionContextFactory<CQLConnectionContext>中会用到2个参数，一个是FLAGS_cql_rpc_memory_limit，另一个是MemTracker，那这两个参数都有什么作用呢？
```
template <class ContextType, class... Args>
std::shared_ptr<ConnectionContextFactory> CreateConnectionContextFactory(Args&&... args) {
  return std::make_shared<ConnectionContextFactoryImpl<ContextType>>(std::forward<Args>(args)...);
}

template <class ContextType>
class ConnectionContextFactoryImpl : public ConnectionContextFactory {
 public:
  ConnectionContextFactoryImpl(
      int64_t memory_limit = 0,
      const std::shared_ptr<MemTracker>& parent_mem_tracker = nullptr)
      : ConnectionContextFactory(
          memory_limit, ContextType::Name(), parent_mem_tracker) {}

  std::unique_ptr<ConnectionContext> Create(size_t receive_buffer_size) override {
    return std::make_unique<ContextType>(receive_buffer_size, buffer_tracker_, call_tracker_);
  }

  virtual ~ConnectionContextFactoryImpl() {}
};
```

可以看到FLAGS_cql_rpc_memory_limit，另一个是MemTracker这2个参数进一步传递给了ConnectionContextFactory：
```
ConnectionContextFactory::ConnectionContextFactory(
    int64_t memory_limit, const std::string& name,
    const std::shared_ptr<MemTracker>& parent_mem_tracker)
    : parent_tracker_(parent_mem_tracker) {
  # 计算root_limit
  # 默认是5%的总内存，可以通过FLAGS_read_buffer_memory_limit进行设置，
  # 如果是正数，则是以字节为单位的内存大小，如果是负数，则表示百分比
  int64_t root_limit = AbsRelMemLimit(FLAGS_read_buffer_memory_limit, [] {
    int64_t total_ram;
    CHECK_OK(Env::Default()->GetTotalRAMBytes(&total_ram));
    return total_ram;
  });

  # 设置名为Read Buffer的MemTracker的最大内存是root_limit
  auto root_buffer_tracker = MemTracker::FindOrCreateTracker(
      root_limit, "Read Buffer", parent_mem_tracker);

  # 设置memory limit，如果传递进来的memory_limit是正数，则直接返回，
  # 否则设置为(-memory_limit)% * root_limit
  memory_limit = AbsRelMemLimit(memory_limit, [&root_buffer_tracker] {
    return root_buffer_tracker->limit();
  });
  
  # 在root_buffer_tracker下面设置名为给定name的MemTracker的最大内存是memory_limit
  buffer_tracker_ = MemTracker::FindOrCreateTracker(memory_limit, name, root_buffer_tracker);
  
  # parent_mem_tracker下的名为Call的MemTracker最大内存无限制(但也不能超过parent_mem_tracker
  # 所允许的最大内存？)
  auto root_call_tracker = MemTracker::FindOrCreateTracker("Call", parent_mem_tracker);
  
  # 在root_call_tracker下面设置名为给定name的MemTracker的最大内存无限制(但
  # 也不能超过root_call_tracker所允许的最大内存？)
  call_tracker_ = MemTracker::FindOrCreateTracker(name, root_call_tracker);
}
```

根据前面的分析，在CQLServer的上下文中，创建了以下的MemTracker结构：
```
Server [内存大小不限]
    CQL  [内存大小不限]
    Read Buffer  [内存默认是5%的总内存，可以通过FLAGS_read_buffer_memory_limit进行设置]
        CQL [内存大小不限，可以通过FLAGS_cql_rpc_memory_limit进行设置]
    Call  [内存大小不限]
        CQL [内存大小不限]
```

由上面的分析可知，在CQLServer上下文中，对应的ConnectionContextFactory是ConnectionContextFactoryImpl<CQLConnectionContext>，它的Create方法实际上会调用CQLConnectionContext的构造方法来创建CQLConnectionContext：
```
template <class ContextType>
class ConnectionContextFactoryImpl : public ConnectionContextFactory {
 public:
  std::unique_ptr<ConnectionContext> Create(size_t receive_buffer_size) override {
    # 调用ContextType的构造方法
    return std::make_unique<ContextType>(receive_buffer_size, buffer_tracker_, call_tracker_);
  }
};

# 在CQLServer上下文中，参数buffer_tracker和call_tracker实际使用的是
# ConnectionContextFactory::buffer_tracker_和ConnectionContextFactory::call_tracker_，
# 在CQLConnectionContext构造方法中创建QLSession，BinaryCallParser和CircularReadBuffer，
# 且CircularReadBuffer的容量通过参数receive_buffer_size进行设置
CQLConnectionContext::CQLConnectionContext(
    size_t receive_buffer_size, const MemTrackerPtr& buffer_tracker,
    const MemTrackerPtr& call_tracker)
    : ql_session_(new ql::QLSession()),
      parser_(buffer_tracker, CQLMessage::kMessageHeaderLength, CQLMessage::kHeaderPosLength,
              FLAGS_max_message_length, rpc::IncludeHeader::kTrue, rpc::SkipEmptyMessages::kFalse,
              this),
      read_buffer_(receive_buffer_size, buffer_tracker),
      call_tracker_(call_tracker) {
  VLOG(1) << "CQL Connection Context: FLAGS_cql_server_always_send_events = " <<
      FLAGS_cql_server_always_send_events;

  if (FLAGS_cql_server_always_send_events) {
    registered_events_ = CQLMessage::kAllEvents;
  }
}
```

那么CQLConnectionContext中的CircularReadBuffer作用是啥呢？且看下面代码：
```
class TcpStream : public Stream {
  ...
  
  StreamReadBuffer& ReadBuffer() {
    return context_->ReadBuffer();
  }
  
  ...
  
private:
  ...
  StreamContext* context_;  
}

Result<bool> TcpStream::Receive() {
  auto iov = ReadBuffer().PrepareAppend();
  if (!iov.ok()) {
    if (iov.status().IsBusy()) {
      read_buffer_full_ = true;
      return false;
    }
    return iov.status();
  }
  read_buffer_full_ = false;

  auto nread = socket_.Recvv(iov.get_ptr());
  if (!nread.ok()) {
    if (Socket::IsTemporarySocketError(nread.status())) {
      return false;
    }
    return nread.status();
  }

  ReadBuffer().DataAppended(*nread);
  return *nread != 0;
}

Result<bool> TcpStream::TryProcessReceived() {
  auto& read_buffer = ReadBuffer();
  if (!read_buffer.ReadyToRead()) {
    return false;
  }

  auto result = VERIFY_RESULT(context_->ProcessReceived(
      read_buffer.AppendedVecs(), ReadBufferFull(read_buffer.Full())));

  read_buffer.Consume(result.consumed, result.buffer);
  return true;
}

StreamReadBuffer& Connection::ReadBuffer() {
  return context_->ReadBuffer();
}

class CQLConnectionContext : public rpc::ConnectionContextWithCallId,
                             public rpc::BinaryCallParserListener {
  rpc::StreamReadBuffer& ReadBuffer() override {
    return read_buffer_;
  }
}
```
可见，在TcpStream接收数据并处理的过程中会用到CQLConnectionContext::read_buffer_。

## Reactor线程之间可能存在竞争的资源
测试发现，1个Reactor线程可以提供高达15万的性能，而2个Reactor的情况下，相比较于1个Reactor，最高性能提升非常小，怀疑Reactor线程之间可能存在资源竞争。

### 资源竞争1：内存资源竞争
- 在CQLConnectionContext::HandleCall中会临时创建InboundCall；
- 在CQLProcessor::ProcessCall -> CQLRequest::ParseRequest中会创建CQLRequest；

关于内存申请这块的资源竞争，暂不考虑。

### 资源竞争2：并发操作队列
- 在CQLConnectionContext::HandleCall中会将InboundCall添加到对应的RPC Service的线程池中，最终实际上是添加到线程池中的叫做BasketQueue的队列中；

**但是当前直接在Reactor线程中返回响应，会直接在Reactor线程中处理，不会加入到RPC Service的线程池中**。


### 资源竞争3：并发访问CQLProcessorList
- 在CQLServiceImpl::Handle中会调用CQLServiceImpl::GetProcessor来获取下一个可用的CQLProcessor，在获取之前会加mutex锁；
    - 这里可以进行优化，基于我们目前的使用Reactor的方式，可以每个Reactor绑定一个CQLProcessor； 
        - 但是仅仅为了测试验证“不同Reactor线程之间，在CQLServiceImpl::GetProcessor()中的锁竞争对性能影响”的话，目前做法比较简单：在CQLServiceImpl中启动256个CQLProcessor，每次调用CQLServiceImpl::GetProcessor()时，按照round-robin的方式返回CQLProcessor。
        - 对应的代码如下；
```
# 添加一个CQLServiceImpl::Init方法，专门用于初始化CQLServiceImpl中的256个CQLProcessor。
void CQLServiceImpl::Init() {
  processors_.reserve(256);
  CQLProcessorListPos pos;
  for (int i = 0; i < 256; i++) {
    pos = processors_.emplace(processors_.end());
    pos->reset(new CQLProcessor(this, pos));
    //std::cout << "processors_ " << i << " " << processors_[i].get() << ", current size " << processors_.size();
  }

  next_available_processor_ = processors_.begin();
}

# 在CQLServiceImpl中添加一个院子的计数器next_sequence_，用于实现round-robin
class CQLServiceImpl : public CQLServerServiceIf,
                       public GarbageCollector,
                       public std::enable_shared_from_this<CQLServiceImpl> {
  ...
  
  std::atomic<int64_t> next_sequence_ = {0};                      
}                       

# 按照round-robin的方式从CQLProcessor集合中获取下一个CQLProcessor
CQLProcessor *CQLServiceImpl::GetProcessor() {
  CQLProcessor* processor = processors_[(next_sequence_++) % processors_.size()].get();
  //std::cout << next_sequence_ << " " << processor;
  return processor;
}

# 修改CQLServiceImpl::ReturnProcessor，直接return
void CQLServiceImpl::ReturnProcessor(const CQLProcessorListPos& pos) {
  // Put the processor back before the next available one.
//  std::lock_guard<std::mutex> guard(processors_mutex_);
//  processors_.splice(next_available_processor_, processors_, pos);
//  next_available_processor_ = pos;
}

# 在CQLServer启动过程中，调用CQLServiceImpl::Init()方法来初始化CQLServiceImpl中的CQLProcessor集合
Status CQLServer::Start() {
  ...
  
  auto cql_service = std::make_shared<CQLServiceImpl>(
      this, opts_, std::bind(&tserver::TabletServerIf::TransactionPool, tserver_));
  cql_service->Init();
  cql_service->CompleteInit();

  ...
}
```

经过在前面的测试：“验证Reactor数目对性能的影响2”中，在优化掉CQLServiceImpl::GetProcessor()中的锁的情况下，性能有一定的提升，性能在一定程度上展现出近线性提升，因此，可以确认不同的Reactor线程之间会在CQLServiceImpl::GetProcessor()中竞争锁资源。

## 总结
可以通过FLAGS_socket_receive_buffer_size来设置Socket receive buffer size，该值会同时影响2个地方：TCP socket receive buffer的大小，以及TCPStream中ReadBuffer的容量，因此大一点好(如果不设置的话，直接使用操作系统默认的设置)？

可以通过FLAGS_cql_rpc_memory_limit来设置CQL Read Buffer的内存大小，默认值是-1，表示不限；

可以通过FLAGS_read_buffer_memory_limit来设置Read Buffer的内存大小，默认值是-5，表示总内存大小的5%；

在CQLServer上下文中，每当接受一个新的Socket连接，都会为之创建一个新的CQLConnectionContext，并注册到对应的Reactor中，每个CQLConnectionContext中都包含QLSession，BinaryCallParser和CircularReadBuffer，这个CircularReadBuffer最终会被TCPStream接收数据和处理 接收到的数据的时候使用；

FLAGS_num_connections_to_server在为连接选择Reactor的时候会用到，默认值是8，可以设置大一些，这样在连接数大于8的情况下，每个连接更大可能性被不同的Reactor所处理；

MemTracker用于监控内存使用情况，它被组织成树状结构，**可以尝试关掉MemTracker，看下对性能是否有影响(经测试，发现有一定的影响，但是咱可以不予考虑)**。关闭MemTracker的方法是直接注释掉MemTracker::Consume和MemTracker::Release中的逻辑。

**不同的Reactor线程在调用CQLServiceImpl::GetProcessor()去获取可用的CQLProcessor的时候，会首先获取锁，因此存在锁竞争，在直接在CQLServer Reactor线程中返回响应情况下，对CQLServiceImpl::GetProcessor()进行优化，去掉了锁，能带来性能提升，并且在一定程度上可以呈现出近线性的提升。**










