# 提纲
[toc]

## CQLServer启动入口
CQLServer在TServer启动过程中被创建并启动。在TabletServerMain中有如下代码：
```
int TabletServerMain(int argc, char** argv) {
    enterprise::Factory factory;
    std::unique_ptr<CQLServer> cql_server;
    
    # 如果设置了在启动TServer的过程中启动CQLServer
    if (FLAGS_start_cql_proxy) {
        # 设置CQLServer配置项
        CQLServerOptions cql_server_options;
        cql_server_options.rpc_opts.rpc_bind_addresses = FLAGS_cql_proxy_bind_address;
        cql_server_options.broadcast_rpc_address = FLAGS_cql_proxy_broadcast_rpc_address;
        cql_server_options.webserver_opts.port = FLAGS_cql_proxy_webserver_port;
        cql_server_options.master_addresses_flag = tablet_server_options->master_addresses_flag;
        cql_server_options.SetMasterAddresses(tablet_server_options->GetMasterAddresses());
        
        boost::asio::io_service io;
        # 借助于enterprise::Factory来创建CQLServer
        cql_server = factory.CreateCQLServer(cql_server_options, &io, server.get());
        LOG(INFO) << "Starting CQL server...";
        LOG_AND_RETURN_FROM_MAIN_NOT_OK(cql_server->Start());
    }
}
```

在enterprise::Factory中CreateCQLServer实际上是会调用CQLServerEnt（企业版CQLServer）的构造方法来创建CLQServer实例（最终实际调用的是CQLServer的构造方法）：
```
class Factory {
 public:
  std::unique_ptr<cqlserver::CQLServer> CreateCQLServer(
      const cqlserver::CQLServerOptions& options, rpc::IoService* io,
      tserver::TabletServer* tserver) {
    # 实际上是调用CQLServerEnt的构造方法，而CQLServerEnt实际上是在CQLServer的基础上
    # 添加了安全相关的内容，所以最终会调用CQLServer的构造方法
    return std::make_unique<CQLServerEnt>(options, io, tserver);
  }
};
```

CQLServer的构造方法执行以下工作：
- 构造它的基类RpcAndWebServerBase的成员；
- 保存CQLServer的配置项到CQLServer::opts_中；
- 创建一个boost定时器CQLServer::timer_；
- 设置当前CQLServer所在的TServer；
- 设置RpcServer（主要是设置ConnectionContextFactory）；
- 代码如下：
```
CQLServer::CQLServer(const CQLServerOptions& opts,
                     boost::asio::io_service* io,
                     tserver::TabletServer* tserver)
    # 调用基类RpcAndWebServerBase的构造方法，初始化RPCServer和WebServer
    : RpcAndWebServerBase(
          "CQLServer", opts, "yb.cqlserver",
          MemTracker::CreateTracker(
              "CQL", tserver ? tserver->mem_tracker() : MemTracker::GetRootTracker(),
              AddToParent::kTrue, CreateMetrics::kFalse)),
      opts_(opts),
      timer_(*io, refresh_interval()),
      tserver_(tserver) {
  # 设置RPCServer，对应的名称为CQLServer
  SetConnectionContextFactory(rpc::CreateConnectionContextFactory<CQLConnectionContext>(
      FLAGS_cql_rpc_memory_limit, mem_tracker()->parent()));
}
```

CQLServer的启动流程如下：
- 初始化CQLServer的基类RpcAndWebServerBase；
    - 打开FsManager，加载文件布局？
    - 初始化clock；
    - 创建Messenger；
    - 初始化RPCServer；
    - 绑定RCPServer到给定地址和端口；
- 创建CQLService（类型为CQLServiceImpl，继承自CQLServerServiceIf，是一个RPC服务），用于处理CQLServer接收到的RPC请求；
    - 设置该CQLService所对应的CQLServer为当前的CQLServer；
    - 创建CQLService和YB-Master或者YB-TServer之间通信的CQLServiceImpl::async_client_init_；
    - 设置RPC messenger为CQLServer对应的messenger；
    - 设置CQLServiceImpl::transaction_pool_provider_为当前CQLServer所在的TServer的TransactionPool（Transaction pool中存放一些事先分配好的transactions）；
    - 启动CQLService和YB-Master或者YB-TServer之间通信的CQLServiceImpl::async_client_init_；
        - 创建并启动一个线程用于执行CQLServiceImpl::async_client_init_的初始化，运行主体为AsyncClientInitialiser::InitClient；
- 注册到CQLService到CQLServer；
- 启动RpcAndWebServerBase，主要包括：
    - 产生一个InstanceID；
    - 添加Web page path handler到WebServer；
    - 启动WebServer；
    - 启动RPCServer；
- 启动CQL node list refresh timer，用于集群中CQL节点视图的维护？
```
Status CQLServer::Start() {
  RETURN_NOT_OK(server::RpcAndWebServerBase::Init());

  auto cql_service = std::make_shared<CQLServiceImpl>(
      this, opts_, std::bind(&tserver::TabletServerIf::TransactionPool, tserver_));
  cql_service->CompleteInit();

  RETURN_NOT_OK(RegisterService(FLAGS_cql_service_queue_length, std::move(cql_service)));

  RETURN_NOT_OK(server::RpcAndWebServerBase::Start());

  // Start the CQL node list refresh timer.
  timer_.async_wait(boost::bind(&CQLServer::CQLNodeListRefresh, this,
                                boost::asio::placeholders::error));
  return Status::OK();
}
```

## RPC Messenger的构建
在CQLServer启动过程中有如下的调用链：CQLServer::Start -> server::RpcAndWebServerBase::Init -> RpcServerBase::Init，其中会借助于MessengerBuilder来构建Messenger。
```
Status RpcServerBase::Init() {
  ...
  
  # 创建MessengerBuilder，并借助该MessengerBuilder来创建Messenger
  rpc::MessengerBuilder builder(name_);
  builder.UseDefaultConnectionContextFactory(mem_tracker());
  RETURN_NOT_OK(SetupMessengerBuilder(&builder));
  messenger_ = VERIFY_RESULT(builder.Build());
  proxy_cache_ = std::make_unique<rpc::ProxyCache>(messenger_.get());

  RETURN_NOT_OK(rpc_server_->Init(messenger_.get()));
  RETURN_NOT_OK(rpc_server_->Bind());

  ...
}
```

其中MessengerBuilder的构造方法如下：
```
MessengerBuilder::MessengerBuilder(std::string name)
    : name_(std::move(name)),
      connection_keepalive_time_(FLAGS_rpc_default_keepalive_time_ms * 1ms),
      coarse_timer_granularity_(100ms),
      listen_protocol_(TcpStream::StaticProtocol()),
      queue_limit_(FLAGS_rpc_queue_limit),
      workers_limit_(FLAGS_rpc_workers_limit),
      num_connections_to_server_(GetAtomicFlag(&FLAGS_num_connections_to_server)) {
  AddStreamFactory(TcpStream::StaticProtocol(), TcpStream::Factory());
}
```

MessengerBuilder中的一些成员的设置会用到的参数简介：
- rpc_default_keepalive_time_ms： 如果一个client rpc连接处于idle状态的时间超过该时间，则server会主动与之断开连接，如果设置为0，则不会断开连接；
- rpc_queue_limit： RPC Server端队列的最大长度；
- rpc_workers_limit：RPC Server端workers的最大数目；
- num_connections_to_server：与每个RPC Server的最大连接数目；

真正构建Messenger的过程在MessengerBuilder::Build()中，过程如下：
- 采用默认的ConnectionContextFactory作为ConnectionContextFactory；
- 以MessengerBuilder作为参数来创建Messenger；
    - 直接采用MessengerBuilder中的name_, connection_context_factory_, stream_factories, listen_protocol_, num_connections_to_server等来设置Messenger；
    - 创建Messenger::io_thread_pool_，顾名思义，这是一个io thread线程池，线程池的名称跟Messenger::name_一样，线程池中线程的数目则通过io_thread_pool_size参数进行设置；
    - 创建Messenger::scheduler_，这是一个调度器；
    - 创建Messenger::normal_thread_pool_；
    - 创建Messegner::reactors_，数目由MessengerBuilder::num_reactors_决定（默认值是4，可以通过MessengerBuilder::set_num_reactors设置），每一个Reactor的创建过程如下：
        - 设置它所关联的Messenger；
        - 设置它的name为Messenger的名称 + “_” + 当前reactor的索引编号；
        - 创建一个Reactor::process_outbound_queue_task；
- 初始化Messenger；
    - 初始化Messenger::reactors_，对每一个reactor初始化工作如下；
        - 设置Reactor::loop_；
        - 设置每个Reactor和Server之间最大的连接数目Reactor::num_connections_to_server_；
        - 

```
Result<std::unique_ptr<Messenger>> MessengerBuilder::Build() {
  # 如果没有设置connection_context_factory_，则直接使用默认的ConnectionContextFactory，
  # 其中connection_context_factory_可以通过UseConnectionContextFactory进行设置，
  # ConnectionContextFactory顾名思义，用于创建ConnectionContext，
  if (!connection_context_factory_) {
    UseDefaultConnectionContextFactory();
  }
  
  # 以MessengerBuilder作为参数来创建Messenger
  std::unique_ptr<Messenger> messenger(new Messenger(*this));
  
  # 初始化Messenger
  RETURN_NOT_OK(messenger->Init());

  return messenger;
}
```

## CQLServer/RPCServer是如何接收远端的RPC请求并处理的
远端如果要调用RPC，则肯定需要首先建立一个RPC连接，而RPCServer中的Messenger就会监听这些连接，具体见RPCServer::Bind流程：
```
Status RpcServer::Bind() {
  # 确认已经执行了初始化过程
  CHECK_EQ(server_state_, INITIALIZED);

  rpc_bound_addresses_.resize(rpc_bind_addresses_.size());
  # Messenger在给定的地址列表上监听，针对每个给定的地址都会创建一个Acceptor去负责监听
  for (size_t i = 0; i != rpc_bind_addresses_.size(); ++i) {
    RETURN_NOT_OK(messenger_->ListenAddress(
        connection_context_factory_, rpc_bind_addresses_[i], &rpc_bound_addresses_[i]));
  }

  server_state_ = BOUND;
  return Status::OK();
}
```

在Messenger::ListenAddress()中，会首先创建一个Acceptor，然后由该Acceptor在给定的地址上监听。在创建Acceptor的时候，会指定一个NewSocketHandler（原型为：typedef std::function<void(Socket *new_socket, const Endpoint& remote)> NewSocketHandler;），该NewSocketHandler会赋值给Acceptor::handler_，Acceptor::handler_则会在处理新的连接请求时被调用。

Messenger::ListenAddress()的代码如下：
```
Status Messenger::ListenAddress(
    ConnectionContextFactoryPtr factory, const Endpoint& accept_endpoint,
    Endpoint* bound_endpoint) {
  Acceptor* acceptor;
  {
    std::lock_guard<percpu_rwlock> guard(lock_);
    if (!acceptor_) {
      # 创建一个Acceptor，在Acceptor的构造方法中第2个参数类型为NewSocketHandler，实际上是
      # std::function<void(Socket *new_socket, const Endpoint& remote)>，该方法接收2个参数，
      # 但是在这里传递的是std::bind(***)，除了Messenger::RegisterInboundSocket是方法以外，
      # 后面的“this, factory, _1, _2 一共4个参数，且Messenger::RegisterInboundSocket
      # 不是只有3个参数吗？什么情况？原来，_1, _2对应的是NewSocketHandler中的两个参数，
      # 而this是C++的每个方法中默认的参数，factory和_1, _2则分别对应于
      # Messenger::RegisterInboundSocket的3个参数。
      #
      # 可以简单理解为在调用NewSocketHandler时候，实际调用的是Messenger::RegisterInboundSocket，
      # 并把传递给NewSocketHandler的参数进一步传递给Messenger::RegisterInboundSocket
      acceptor_.reset(new Acceptor(
          metric_entity_, std::bind(&Messenger::RegisterInboundSocket, this, factory, _1, _2)));
    }
    auto accept_host = accept_endpoint.address();
    auto& outbound_address = accept_host.is_v6() ? outbound_address_v6_
                                                 : outbound_address_v4_;
    if (outbound_address.is_unspecified() && !accept_host.is_unspecified()) {
      outbound_address = accept_host;
    }
    acceptor = acceptor_.get();
  }
  
  # 监听在给定地址上，最终调用的是Socket::Listen
  return acceptor->Listen(accept_endpoint, bound_endpoint);
}
```

Acceptor监听逻辑：
- 创建并初始化一个Socket，用于监听；
- bind到给定监听地址上；
- 在bound到的地址上监听；
- 设置关注用于监听的socket上的EV_READ事件，并设置事件处理方法为Acceptor::IOHandler(具体实现见Acceptor::AsyncHandler)；
- 代码如下：
```
Status Acceptor::Listen(const Endpoint& endpoint, Endpoint* bound_endpoint) {
  # 创建并初始化Socket
  Socket socket;
  RETURN_NOT_OK(socket.Init(endpoint.address().is_v6() ? Socket::FLAG_IPV6 : 0));
  RETURN_NOT_OK(socket.SetReuseAddr(true));
  # 绑定到某个地址
  RETURN_NOT_OK(socket.Bind(endpoint));
  if (bound_endpoint) {
    RETURN_NOT_OK(socket.GetSocketAddress(bound_endpoint));
  }
  RETURN_NOT_OK(socket.SetNonBlocking(true));
  # 监听
  RETURN_NOT_OK(socket.Listen(FLAGS_rpc_acceptor_listen_backlog));

  bool was_empty;
  {
    std::lock_guard<std::mutex> lock(mutex_);
    if (closing_) {
      return STATUS_SUBSTITUTE(ServiceUnavailable, "Acceptor closing");
    }
    was_empty = sockets_to_add_.empty();
    # 将用于监听的socket添加到sockets_to_add_中等待事件处理主循环的处理
    sockets_to_add_.push_back(std::move(socket));
  }

  if (was_empty) {
    # 通知Reactor线程有任务待处理
    async_.send();
  }

  return Status::OK();
}
```

当有新的连接请求到达时，用于监听的Socket上的EV_READ事件将被触发，相应的处理方法为Acceptor::IoHandler。处理逻辑就是Accept新的连接请求，并将新建立的连接请求注册到Messenger的Reactor中。
```
void Acceptor::IoHandler(ev::io& io, int events) {
  auto it = sockets_.find(&io);
  if (it == sockets_.end()) {
    LOG(ERROR) << "IoHandler for unknown socket: " << &io;
    return;
  }
  Socket& socket = it->second.socket;
  if (events & EV_ERROR) {
    LOG(INFO) << "Acceptor socket failure: " << socket.GetFd()
              << ", endpoint: " << it->second.endpoint;
    sockets_.erase(it);
    return;
  }

  if (events & EV_READ) {
    # 循环处理新的连接请求
    for (;;) {
      Socket new_sock;
      Endpoint remote;
      # Accept新的连接请求，建立新的连接
      Status s = socket.Accept(&new_sock, &remote, Socket::FLAG_NONBLOCKING);
      if (!s.ok()) {
        if (!Socket::IsTemporarySocketError(s)) {
          LOG(WARNING) << "Acceptor: accept failed: " << s.ToString();
        }
        return;
      }
      s = new_sock.SetNoDelay(true);
      rpc_connections_accepted_->Increment();
      
      # 这里的handler_就是在Messenger::ListenAddress中设置的Messenger::RegisterInboundSocket，
      # 注册该新的连接到Messenger的Reactor中
      handler_(&new_sock, remote);
    }
  }
}
```

在Messenger::RegisterInboundSocket中，首先为该新的连接选择一个Reactor，然后通过Reactor::RegisterInboundSocket将该新的连接注册到该Reactor上。
```
void Messenger::RegisterInboundSocket(
    const ConnectionContextFactoryPtr& factory, Socket *new_socket, const Endpoint& remote) {
  # 一些检查
  ...
  
  # 选择一个Reactor来处理该新的连接
  int idx = num_connections_accepted_.fetch_add(1) % num_connections_to_server_;
  Reactor *reactor = RemoteToReactor(remote, idx);
  # 注册该新的连接
  reactor->RegisterInboundSocket(
      new_socket, remote, factory->Create(*receive_buffer_size), factory->buffer_tracker());
}
```

在Reactor::RegisterInboundSocket中会基于Socket创建一个Stream，然后再基于Stream创建一个Connection，最后将这个Connection注册到Reactor中：
```
void Reactor::RegisterInboundSocket(
    Socket *socket, const Endpoint& remote, std::unique_ptr<ConnectionContext> connection_context,
    const MemTrackerPtr& mem_tracker) {
  # 基于socket创建stream，messenger_->stream_factories_存放了所有注册的StreamFactory，
  # 默认注册的是TcpStreamFactory，messenger_->listen_protocol_中默认的是TcpStream::
  # StaticProtocol，这里会去messenger_->stream_factories_messenger_->listen_protocol_
  # 中找到messenger_->listen_protocol_对应的StreamFactory，也就是TcpStreamFactory，
  # 最后调用TcpCreateFactory来基于给定Socket创建TcpStream
  auto stream = CreateStream(
      messenger_->stream_factories_, messenger_->listen_protocol_,
      {remote, std::string(), socket, mem_tracker});
  if (!stream.ok()) {
    LOG_WITH_PREFIX(DFATAL) << "Failed to create stream for " << remote << ": " << stream.status();
    return;
  }
  
  # 基于创建的Stream来创建Connection
  auto conn = std::make_shared<Connection>(this,
                                           std::move(*stream),
                                           ConnectionDirection::SERVER,
                                           &messenger()->rpc_metrics(),
                                           std::move(connection_context));
  # 在Reactor线程中调度执行当前任务，任务内容为将“Connection”注册到Reactor中                       
  ScheduleReactorFunctor([conn = std::move(conn)](Reactor* reactor) {
    reactor->RegisterConnection(conn);
  }, SOURCE_LOCATION());
}
```

在Reactor::RegisterConnection中会执行以下操作：
- 设置Connection对应的事件主循环为Reactor::loop_；
- 启动对应的Stream，比如对于TcpStream来说，主要是设置：关注对应Socket上的Read事件，对应的事件主循环为Reactor::loop_，对应的事件处理方法为TcpStream::Handler；
- 初始化并启动一个定时器，专门用于处理连接超时事件；

TcpStream::Handler用于处理连接上接收到的事件。针对读事件和写事件的处理逻辑分别为ReadHandler和WriteHandler。
```
void TcpStream::Handler(ev::io& watcher, int revents) {  // NOLINT
  DVLOG_WITH_PREFIX(4) << "Handler(revents=" << revents << ")";
  Status status = Status::OK();

  # 读事件的处理
  if (status.ok() && (revents & ev::READ)) {
    status = ReadHandler();
  }

  # 写事件的处理
  if (status.ok() && (revents & ev::WRITE)) {
    bool just_connected = !connected_;
    if (just_connected) {
      connected_ = true;
      context_->Connected();
    }
    status = WriteHandler(just_connected);
  }

  if (status.ok()) {
    UpdateEvents();
  } else {
    context_->Destroy(status);
  }
}
```

在TcpStream::ReadHandler中处理读事件，首先会借助于Socket::Recvv方法进行读取，然后调用Connection::ProcessReceived()进行处理。
```
Status TcpStream::ReadHandler() {
  context_->UpdateLastRead();

  for (;;) {
    # 封装了Socket::Recvv进行读取
    auto received = Receive();
    
    # 错误处理
    ...
    
    # 处理读取到的数据，最终调用的是Connection::ProcessReceived
    auto continue_receiving = TryProcessReceived();
    
    if (!continue_receiving.ok()) {
      return continue_receiving.status();
    }
    
    if (!continue_receiving.get()) {
      return Status::OK();
    }
  }
}
```

在Connection::ProcessReceived中会进一步调用Connection::context_对应的ProcessCalls方法，在当前CQLServer的上下文中，Connection::context_对应的是啥呢？经过代码追溯发现在CQLServer中采用rpc::CreateConnectionContextFactory<CQLConnectionContext>(FLAGS_cql_rpc_memory_limit, mem_tracker()->parent())来创建ConnectionContext，实际创建的ConnectionContext类型为CQLConnectionContext，追溯过程如下：
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
  # 设置CQLServer中用到的ConnectionContextFactory
  SetConnectionContextFactory(rpc::CreateConnectionContextFactory<CQLConnectionContext>(
      FLAGS_cql_rpc_memory_limit, mem_tracker()->parent()));
}

void RpcServerBase::SetConnectionContextFactory(
    rpc::ConnectionContextFactoryPtr connection_context_factory) {
  # 在构建RpcServer实例的时候，会传递ConnectionContextFactory作为参数
  rpc_server_.reset(new RpcServer(name_, options_.rpc_opts, std::move(connection_context_factory)));
}

RpcServer::RpcServer(const std::string& name, RpcServerOptions opts,
                     rpc::ConnectionContextFactoryPtr connection_context_factory)
    : name_(name),
      server_state_(UNINITIALIZED),
      options_(std::move(opts)),
      # 设置RpcServer::connection_context_factory_
      connection_context_factory_(std::move(connection_context_factory)) {}
      
Status RpcServer::Bind() {
  CHECK_EQ(server_state_, INITIALIZED);

  rpc_bound_addresses_.resize(rpc_bind_addresses_.size());
  for (size_t i = 0; i != rpc_bind_addresses_.size(); ++i) {
    # 这里用到RpcServer::connection_context_factory_
    RETURN_NOT_OK(messenger_->ListenAddress(
        connection_context_factory_, rpc_bind_addresses_[i], &rpc_bound_addresses_[i]));
  }

  server_state_ = BOUND;
  return Status::OK();
}

Status Messenger::ListenAddress(
    ConnectionContextFactoryPtr factory, const Endpoint& accept_endpoint,
    Endpoint* bound_endpoint) {
  Acceptor* acceptor;
  {
    std::lock_guard<percpu_rwlock> guard(lock_);
    if (!acceptor_) {
      # 参数factory将作为Messenger::RegisterInboundSocket的参数
      acceptor_.reset(new Acceptor(
          metric_entity_, std::bind(&Messenger::RegisterInboundSocket, this, factory, _1, _2)));
    }
    
    ...
  }
  return acceptor->Listen(accept_endpoint, bound_endpoint);
}

void Messenger::RegisterInboundSocket(
    const ConnectionContextFactoryPtr& factory, Socket *new_socket, const Endpoint& remote) {
  int idx = num_connections_accepted_.fetch_add(1) % num_connections_to_server_;
  Reactor *reactor = RemoteToReactor(remote, idx);
  # factory->Create(*receive_buffer_size)将返回ConnectionContext
  reactor->RegisterInboundSocket(
      new_socket, remote, factory->Create(*receive_buffer_size), factory->buffer_tracker());
}

void Reactor::RegisterInboundSocket(
    Socket *socket, const Endpoint& remote, std::unique_ptr<ConnectionContext> connection_context,
    const MemTrackerPtr& mem_tracker) {
  ...
  
  # 将方法调用参数中的connection_context赋值给新建立的Connection::context_
  auto conn = std::make_shared<Connection>(this,
                                           std::move(*stream),
                                           ConnectionDirection::SERVER,
                                           &messenger()->rpc_metrics(),
                                           std::move(connection_context));
  ScheduleReactorFunctor([conn = std::move(conn)](Reactor* reactor) {
    reactor->RegisterConnection(conn);
  }, SOURCE_LOCATION());
}

Connection::Connection(Reactor* reactor,
                       std::unique_ptr<Stream> stream,
                       Direction direction,
                       RpcMetrics* rpc_metrics,
                       std::unique_ptr<ConnectionContext> context)
    : reactor_(reactor),
      stream_(std::move(stream)),
      direction_(direction),
      last_activity_time_(CoarseMonoClock::Now()),
      rpc_metrics_(rpc_metrics),
      # 使用构造方法中的context来初始化Connection::context_
      context_(std::move(context)) {
  const auto metric_entity = reactor->messenger()->metric_entity();
  handler_latency_outbound_transfer_ = metric_entity ?
      METRIC_handler_latency_outbound_transfer.Instantiate(metric_entity) : nullptr;
  IncrementCounter(rpc_metrics_->connections_created);
  IncrementGauge(rpc_metrics_->connections_alive);
}
```

进入CQLConnectionContext::ProcessCalls后的调用栈如下：
```
yb::cqlserver::CQLConnectionContext::ProcessCalls
    - yb::rpc::BinaryCallParser::Parse
        # 在CQLConnectionContext的构造方法中可知，BinaryCallParser::listener_实际
        # 上就是CQLConnectionContext，所以实际调用的是CQLConnectionContext::HandleCall
        - BinaryCallParserListener::HandleCall
            - CQLConnectionContext::HandleCall
```

在CQLConnectionContext::HandleCall中会执行以下操作：
- 创建一个CQLInboundCall（CQLInBoundCall继承自rpc::InboundCall）;
- 从给定的rpc::CallData中解析出新创建的CQLInboundCall相关的信息，比如CQLRequest和对应的CQLServiceImpl等；
- 将新创建的CQLInboundCall添加到calls_being_handled_集合中；
- 将新创建的CQLInboundCall提交给对应的RpcService进行处理，那么这里的“对应的RpcService”是什么呢？又是怎么找到的呢？
    - 在Messenger::RegisterService的时候会将service name -> rpc service的映射添加到Messenger::rpc_services_中，然后会采用Messenger::rpc_services_来更新Messenger::rpc_services_cache_；
    - 沿着CQLConnectionContext -> reactor -> messenger的引用关系，可以找到Messenger；
    - InboundCall中包含Service name（当前上下文中Service name是yb.cqlserver.CQLServerService），借助该Service name，可以从Messenger中找到对应的RpcService，结果找到的是RpcService是谁呢？
        - 原来在CQLServer::Start -> RpcServerBase::RegisterService -> RpcServer::RegisterService
          中注册的RpcService确实是CQLServiceImpl，但是在RpcServer::RegisterService内部对待注册的
          CQLServiceImpl进行了封装，封装出来了一个叫做ServicePool的RpcService（ServicePool也继承自RpcService），然后调用Messenger::RegisterService将这个ServicePool真正注册；
        - 也就是说，对上层调用来说，在调用RegisterService的时候，传递的是真实提供rpc服务的RpcService，但是在RpcServer内部会对传递进来的RpcService进行封装，封装成一个叫做ServicePool的RpcService，真正注册的也是这个封装之后的RpcService；
        - 那么ServicePool是如何进行封装的呢？在创建ServicePool的时候，会传递一个创建好的线程池rpc::ThreadPool，一个调度器Messenger::scheduler_，一个真实提供rpc服务的RpcService，并以这些参数构建一个ServicePoolImpl，这个ServicePoolImpl继承自InboundCallHandler，最终ServicePool的所有处理都交给ServicePoolImpl来处理；
        - 这个ServicePool中的重要域成员有：线程池thread_pool_，调度器scheduler_，真正提供RPC服务的service_等等；
    - 最后将CQLInboundCall提交给RpcService（RpcService::QueueInboundCall）；
        - 经过上面的分析可知，最终会提交给ServicePool这个RpcService。
```
Status CQLConnectionContext::HandleCall(
    const rpc::ConnectionPtr& connection, rpc::CallData* call_data) {
  auto reactor = connection->reactor();
  DCHECK(reactor->IsCurrentThread());

  auto call = rpc::InboundCall::Create<CQLInboundCall>(
      connection, call_processed_listener(), ql_session_);

  Status s = call->ParseFrom(call_tracker_, call_data);
  if (!s.ok()) {
    LOG(WARNING) << connection->ToString() << ": received bad data: " << s.ToString();
    return STATUS_SUBSTITUTE(NetworkError, "Bad data: $0", s.ToUserMessage());
  }

  s = Store(call.get());
  if (!s.ok()) {
    return s;
  }

  # 经过上面的分析可知，最终调用的是ServicePool::QueueInboundCall
  reactor->messenger()->QueueInboundCall(call);

  return Status::OK();
}
```

经过上面的分析可知，最终会通过ServicePool::QueueInboundCall提交给ServicePool这个RpcService，进一步会调用ServicePoolImpl::Enqueue。
```
void ServicePool::QueueInboundCall(InboundCallPtr call) {
  # 实际上是ServicePoolImpl::Enqueue
  impl_->Enqueue(std::move(call));
}
```

在ServicePoolImpl::Enqueue中会首先将InboundCall和对应的InboundCallHandler（就是当前的ServicePool）进行绑定，然后将InboundCall对应的InboundCallTask（每个InboundCall在创建的时候都会对应一个InboundCallTask）提交给线程池。
```
  void ServicePoolImpl::Enqueue(const InboundCallPtr& call) {
    # 将将InboundCall和对应的InboundCallHandler（就是这里的this，也就是ServicePoolImpl）进行绑定
    auto task = call->BindTask(this);

    ...

    # 提交InboundCall对应的InboundCallTask到线程池的任务队列中
    thread_pool_.Enqueue(task);
  }
```

线程池中的每一个线程类型为Worker，在Worker的构造方法中会创建一个线程，也就是这个Worker的工作线程，运行的主体逻辑是Worker::Execute，它会不断的从线程池的任务队列中获取一个Task，并执行之：
```
  void Execute() {
    Thread::current_thread()->SetUserData(share_);
    while (!stop_requested_) {
      ThreadPoolTask* task = nullptr;
      # 从线程池的任务队列中pop出来一个task执行
      if (PopTask(&task)) {
        # 运行task
        task->Run();
        # task执行完后的回调
        task->Done(Status::OK());
      }
    }
  }
```

在当前上下文中task类型是CQLInboundCall::InboundCallTask，对应的Run()方法和Done()方法分别如下：
```
void InboundCall::InboundCallTask::Run() {
  # 直接调用InboundCallHandler::Handle，这里的InboundCallHandler实际类型是ServicePoolImpl
  # 所以调用的是ServicePoolImpl::Handle
  handler_->Handle(call_);
}

void InboundCall::InboundCallTask::Done(const Status& status) {
  # 当前只是检查是否发生了错误，如果错误了则调用InboundCallHandler::Failure
  auto call = std::move(call_);
  if (!status.ok()) {
    handler_->Failure(call, status);
  }
}
```

ServicePoolImpl::Handle中最终会调用真正提供RPC服务的RpcService的Handle方法进行处理，在当前上下文中对应的是CQLServiceImpl::Handle。
```
  void Handle(InboundCallPtr incoming) override {
    incoming->RecordHandlingStarted(incoming_queue_time_);
    const char* error_message;
    if (PREDICT_FALSE(incoming->ClientTimedOut())) {
      error_message = kTimedOutInQueue;
    } else if (PREDICT_FALSE(ShouldDropRequestDuringHighLoad(incoming))) {
      error_message = "The server is overloaded. Call waited in the queue past max_time_in_queue.";
    } else {

      if (incoming->TryStartProcessing()) {
        # 调用真正提供RPC服务的RpcService的Handle方法进行处理，在当前上下文中对应的是
        # CQLServiceImpl::Handle
        service_->Handle(std::move(incoming));
      }
      return;
    }

    // Respond as a failure, even though the client will probably ignore
    // the response anyway.
    TimedOut(incoming.get(), error_message, rpcs_timed_out_in_queue_.get());
  }
```

在CQLServiceImpl::Handle中会进一步交由CQLProcessor进行处理。
```
void CQLServiceImpl::Handle(yb::rpc::InboundCallPtr inbound_call) {
  // Collect the call.
  CQLInboundCall* cql_call = down_cast<CQLInboundCall*>(CHECK_NOTNULL(inbound_call.get()));
  DVLOG(4) << "Handling " << cql_call->ToString();

  # 获取下一个可用的CQLProcessor，如果没有可用的，则创建一个新的
  CQLProcessor *processor = GetProcessor();
  CHECK(processor != nullptr);
  
  # 交由CQLProcessor::ProcessCall进行处理
  processor->ProcessCall(std::move(inbound_call));
}

void CQLProcessor::ProcessCall(rpc::InboundCallPtr call) {
  call_ = std::dynamic_pointer_cast<CQLInboundCall>(std::move(call));
  unique_ptr<CQLRequest> request;
  unique_ptr<CQLResponse> response;

  # 从InboundCall中解析出CQLRequest和CQLResponse
  const auto compression_scheme = context.compression_scheme();
  if (!CQLRequest::ParseRequest(call_->serialized_request(), compression_scheme,
                                &request, &response)) {
    cql_metrics_->num_errors_parsing_cql_->Increment();
    PrepareAndSendResponse(response);
    return;
  }

  execute_begin_ = MonoTime::Now();
  cql_metrics_->time_to_parse_cql_wrapper_->Increment(
      execute_begin_.GetDeltaSince(parse_begin_).ToMicroseconds());

  // Execute the request (perhaps asynchronously).
  SetCurrentSession(call_->ql_session());
  request_ = std::move(request);
  call_->SetRequest(request_, service_impl_);
  retry_count_ = 0;
  # 处理请求并设置response
  response.reset(ProcessRequest(*request_));
  # 发送响应
  PrepareAndSendResponse(response);
}
```

在CQLProcessor::ProcessRequest中会根据CQLRequest对应的opcode来决定请求类型，然后将CQLRequest转换为对应的具体类型的CQLRequest，最后调用ProcessRequest进行处理：
```
CQLResponse* CQLProcessor::ProcessRequest(const CQLRequest& req) {
  switch (req.opcode()) {
    case CQLMessage::Opcode::OPTIONS:
      return ProcessRequest(static_cast<const OptionsRequest&>(req));
    case CQLMessage::Opcode::STARTUP:
      return ProcessRequest(static_cast<const StartupRequest&>(req));
    case CQLMessage::Opcode::PREPARE:
      return ProcessRequest(static_cast<const PrepareRequest&>(req));
    case CQLMessage::Opcode::EXECUTE:
      return ProcessRequest(static_cast<const ExecuteRequest&>(req));
    case CQLMessage::Opcode::QUERY:
      return ProcessRequest(static_cast<const QueryRequest&>(req));
    case CQLMessage::Opcode::BATCH:
      return ProcessRequest(static_cast<const BatchRequest&>(req));
    case CQLMessage::Opcode::AUTH_RESPONSE:
      return ProcessRequest(static_cast<const AuthResponseRequest&>(req));
    case CQLMessage::Opcode::REGISTER:
      return ProcessRequest(static_cast<const RegisterRequest&>(req));

    case CQLMessage::Opcode::ERROR: FALLTHROUGH_INTENDED;
    case CQLMessage::Opcode::READY: FALLTHROUGH_INTENDED;
    case CQLMessage::Opcode::AUTHENTICATE: FALLTHROUGH_INTENDED;
    case CQLMessage::Opcode::SUPPORTED: FALLTHROUGH_INTENDED;
    case CQLMessage::Opcode::RESULT: FALLTHROUGH_INTENDED;
    case CQLMessage::Opcode::EVENT: FALLTHROUGH_INTENDED;
    case CQLMessage::Opcode::AUTH_CHALLENGE: FALLTHROUGH_INTENDED;
    case CQLMessage::Opcode::AUTH_SUCCESS:
      break;
  }

  LOG(FATAL) << "Invalid CQL request: opcode = " << static_cast<int>(req.opcode());
  return nullptr;
}
```

下面以处理QueryRequest为例来分析CQLProcessor::ProcessRequest，首先进行parse和analyze，生成ParseTree，然后将ParseTree交由Executor进行处理：
```
CQLResponse* CQLProcessor::ProcessRequest(const QueryRequest& req) {
  # req.query()返回的是查询语句，req.params()则返回查询参数，statement_executed_cb_
  # 为请求执行后的回调
  RunAsync(req.query(), req.params(), statement_executed_cb_);
  return nullptr;
}

void QLProcessor::RunAsync(const string& stmt, const StatementParameters& params,
                           StatementExecutedCallback cb, const bool reparsed) {
  ParseTree::UniPtr parse_tree;
  # 首先进行parse和analyze阶段，生成ParseTree
  const Status s = Prepare(stmt, &parse_tree, reparsed);
  
  # 如果失败，则直接调用回调
  if (PREDICT_FALSE(!s.ok())) {
    return cb.Run(s, nullptr /* result */);
  }
  
  const ParseTree* ptree = parse_tree.release();
  # 交给Tree Executor进行处理，具体见Executor::ExecuteAsync
  ExecuteAsync(*ptree, params, Bind(&QLProcessor::RunAsyncDone, Unretained(this), ConstRef(stmt),
                                    ConstRef(params), Owned(ptree), cb));
}
```

Executor::ExecuteAsync的执行流程如下：
```
Executor::ExecuteAsync
    - Executor::Execute
        - Executor::ExecTreeNode
            # 对update语句的处理TreeNodeOpcode::kPTInsertStmt
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
                    - write_batch_.Add(op)
                        - 对当前的更新请求作如下处理：
                            - 如果当前操作会读primary key，且当前的write_batch_中已经有一个关于相同key的写操作，则返回false，表明当前操作和另一个操作之间存在冲突；
                            - 如果当前操作会读hash key，且当前的write_batch_中已经有一个关于相同key的写操作，则返回false，表明当前操作和另一个操作之间存在冲突；
                            - 否则：
                                - 如果当前的write_batch_需要读primary key，则将之添加到ops_by_primary_key_中；
                                - 如果当前write_batch_需要读hash key，则将之添加到ops_by_hash_key_中；
                                - 返回true，表明可以添加到当前的write_batch_
                    - session->Apply(op)
                        - YBSession::batcher_.Add
                            - 首先成成一个InFlightOp，InFlightOp表示一个操作已经提交给Batcher，但是尚未执行完成
                              auto in_flight_op = std::make_shared<InFlightOp>(yb_op);
                            - yb::client::internal::Batcher::Add
                                - 将当前in_flight_op添加到InFlightOp集合中，并增加正在查找tablet的操作的数目
                                  AddInFlightOp(in_flight_op)
                                - yb::client::internal::MetaCache::LookupTabletByKey
                                    # 首先尝试在本地的MetaCache中进行查找
                                    - FastLookupTabletByKeyUnlocked
                                    # 如果前一步骤中没有查找到，则发送RPC消息
                                    - rpc::StartRpc<LookupByKeyRpc>(...)
                                        - LookupByKeyRpc::DoSendRpc
                                            # 发送消息给Master，获取Tablet信息
                                            - master_proxy()->GetTableLocationsAsync
                                # 当成功查找到Tablet信息之后，则调用TabletLookupFinished处理当前的InFlightOp
                                - TabletLookupFinished(std::move(in_flight_op), yb_op->tablet())
                                    - 将当前的InFlightOp的状态从InFlightOpState::kLookingUpTablet转换为InFlightOpState::kBufferedToTabletServer
                                    - 将当前的InFlightOp添加到yb::client::internal::Batcher::ops_queue_中，ops_queue_是一个关于InFlightOp的数组
                                    - 如果正在进行tablet查找的操作的数目为0（记录在outstanding_lookups_中），则执行**Batcher::FlushBuffersIfReady**
                                        - 首先对yb::client::internal::Batcher::ops_queue_中的所有operation进行排序，排序的依据是：首先比较tablet id，然后比较操作类型（分为3中类型：OpGroup::kWrite，OpGroup::kConsistentPrefixRead和OpGroup::kLeaderRead），最后比较操作的sequence_number_；
                                        - 执行这些操作：Batcher::ExecuteOperations
                                            - 如果是在事务上下文中，则先执行YBTransaction::Prepare
                                            - 然后遍历yb::client::internal::Batcher::ops_queue_中的所有的操作，关于相同tablet的相同操作类型（OpGroup类型）的操作聚合在一起生成一个RPC请求（类型为AsyncRpc，继承自TabletRpc），如果当前的tablet和前一个tablet发生变更，或者当前OpGroup和前一个OpGroup发生变更，则生成一个新的RPC请求，根据OpGroup类型不同，生成的RPC请求也不同，可能的RPC请求类型分别为WriteRpc，ReadRpc；
                                            - 清除yb::client::internal::Batcher::ops_queue_中所有的操作；
                                            - 遍历所有生成的RPC请求，并借助AsyncRpc::SendRpc发送；
                                                - tablet_invoker_.Execute(std::string(), num_attempts() > 1)
                                                    # 按照一定的算法选择一个目标TServer，优先选择Tablet Leader所在的TServer，如果没有这样的TServer，则从Tablet Replica中选择一个
                                                    - TabletInvoker::SelectTabletServer
                                                    # 以WriteRpc为例，这里调用的是AsyncRpc::SendRpcToTserver
                                                    - rpc_->SendRpcToTserver(retrier_->attempt_num())
                                                        - WriteRpc::CallRemoteMethod
                                                            # tablet_invoker_.proxy()类型为TabletServerServiceProxy
                                                            # 将RPC请求发送给TabletServer，在TServer端处理的RPC Service是TabletServiceImpl
                                                            # 当接收到RPC请求的响应的时候WriteRpc::Finished会被调用
                                                            - tablet_invoker_.proxy()->WriteAsync(
                                                              req_, &resp_, PrepareController(),
                                                              std::bind(&WriteRpc::Finished, this, Status::OK()))
                                                        - ...略过RPC请求在TServer端处理流程...
                                                        - 当接收到RPC请求的响应的时候WriteRpc::Finished会被调用
                                                            - AsyncRpc::Finished
                                                                - tablet_invoker_.Done(&new_status)
                                                                - ProcessResponseFromTserver(new_status)
                                                                    - WriteRpc::ProcessResponseFromTserver
                                                                        - batcher_->ProcessWriteResponse(*this, status)
                                                                        # 将RPC请求的request映射到该RPC请求所关联的每个操作的request（在前面分析中提到关于相同tablet的相同操作类型会生成一个RPC请求，也就是说一个RPC请求可能会对应多个操作），将RPC请求的response映射到该RPC请求所关联的每个操作的response
                                                                        - SwapRequestsAndResponses(false)
                                                                - batcher_->RemoveInFlightOpsAfterFlushing(ops_, new_status, MakeFlushExtraResult())
                                                                - batcher_->CheckForFinishedFlush()
                                    - 如果正在进行tablet查找的操作的数目不为0（记录在outstanding_lookups_中），则直接返回
    # 为什么要执行这个呢？因为在分析TabletLookupFinished中提到，只有在判断“正在进行tablet查找的操作的数目为0”
    # 情况下才会将请求发送给TServer端执行，否则就需要借助这里的Executor::FlushAsync来将请求发送给TServer端执行
    - Executor::FlushAsync
        # FlushAsync() is called to flush buffered operations in the non-transactional
        # session in the Executor or the transactional session in each ExecContext if 
        # any. Also, transactions in any ExecContext ready to commit with no more 
        # pending operation are also committed. If there is no session to flush nor
        # transaction to commit, the statement is executed.
        # 
        # 遍历所有需要commit的execution context，并执行commit，commit成功后的回调为CommitDone
        - for (auto* exec_context : commit_contexts) {
            exec_context->CommitTransaction([this, exec_context](const Status& s) {
                CommitDone(s, exec_context);
              });
          }
        # 遍历所有需要flush的session，并执行flush，flush成功后的回调为FlushAsyncDone
        - for (const auto& pair : flush_sessions) {
            auto session = pair.first;
            auto exec_context = pair.second;
            session->SetRejectionScoreSource(rejection_score_source);
            session->FlushAsync([this, exec_context](const Status& s) {
                FlushAsyncDone(s, exec_context);
              });
          }
        # 对于session->FlushAsync（YBSession::FlushAsync），会进一步调用yb::client::internal::Batcher::FlushAsync
        - yb::client::internal::Batcher::FlushAsync
            # 在前面分析TabletLookupFinished的过程中已经分析了Batcher::FlushBuffersIfReady，在此不再赘述
            - Batcher::FlushBuffersIfReady
```

## CQLServer是如何跟TServer交互的
在前面的分析可知，CQLServer上在处理接收到的RPC请求，如Insert语句相关的RPC请求之后，会进一步生成新的RPC请求并交给yb::client::internal::TabletInvoker::Execute，以发送到对应的TServer上去处理，这个流程是怎样的呢？下面进行分析。
```
void TabletInvoker::Execute(const std::string& tablet_id, bool leader_only) {
  # 如果tablet_id为空，则从tablet中获取tablet id
  if (tablet_id_.empty()) {
    if (!tablet_id.empty()) {
      tablet_id_ = tablet_id;
    } else {
      tablet_id_ = CHECK_NOTNULL(tablet_.get())->tablet_id();
    }
  }

  # 选择tablet对应的TServer，设置在current_ts_中
  ...
  
  # 在发送RPC之前，需要确保TabletServerServiceProxy存在，如果不存在，
  # 则会新建一个TabletServerServiceProxy，TabletServerServiceProxy实际上
  # 是对yb::rpc::Proxy的封装，在TabletServerServiceProxy的构造方法中会
  # 传递一个参数：remote，它表示本次RPC服务提供者所在的主机和端口号，该
  # 参数会进一步传递给yb::rpc::Proxy的构造方法，在yb::rpc::Proxy的构造
  # 方法中有一个成员叫做call_local_service_，它会根据这个remote参数是否
  # 代表本地的一个主机和端口号来进行设置为true或者false，如果设置为true，
  # 则表明这是一个本地调用
  auto status = current_ts_->InitProxy(client_);

  # 这里的rpc_类型为TabletRpc，但真实类型可能是WriteRpc或者ReadRpc（类关系为WriteRpc/ReadRpc -> 
  # AsyncRpc -> TabletRpc），TabletRpc::SendRpcToTserver是虚方法，AsyncRpc::SendRpcToTserver则会
  # 进一步调用AsyncRpc::CallRemoteMethod，这也是一个虚方法，具体的实现为WriteRpc::CallRemoteMethod
  # 或者ReadRpc::CallRemoteMethod
  rpc_->SendRpcToTserver(retrier_->attempt_num());
}
```

以WriteRpc::CallRemoteMethod为例，它会借助于TabletServerServiceProxy::WriteAsync将RPC请求发送出去：
```
void WriteRpc::CallRemoteMethod() {
  # TabletServerServiceProxy::WriteAsync，其中WriteRpc::Finished为RPC请求的回调
  tablet_invoker_.proxy()->WriteAsync(
      req_, &resp_, PrepareController(),
      std::bind(&WriteRpc::Finished, this, Status::OK()));
}

void TabletServerServiceProxy::WriteAsync(const WriteRequestPB &req,
                     WriteResponsePB *resp, ::yb::rpc::RpcController *controller,
                     ::yb::rpc::ResponseCallback callback) {
  # 生成一个RemoteMethod，指定对应的RPC服务为yb.tserver.TabletServerService，对应的方法名称为Write
  static ::yb::rpc::RemoteMethod method("yb.tserver.TabletServerService", "Write");
  # proxy_类型为Proxy，用于将RPC请求发送给远端或者本地，它会进一步调用Proxy::DoAsyncRequest
  proxy_->AsyncRequest(&method, req, resp, controller, std::move(callback));
}

void Proxy::DoAsyncRequest(const RemoteMethod* method,
                           const google::protobuf::Message& req,
                           google::protobuf::Message* resp,
                           RpcController* controller,
                           ResponseCallback callback,
                           bool force_run_callback_on_reactor) {
  CHECK(controller->call_.get() == nullptr) << "Controller should be reset";
  is_started_.store(true, std::memory_order_release);

  # 如果是本地调用，则生成一个LocalOutboundCall，否则生成一个OutboundCall
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
  # 获取call，类型为OutboundCall，并设置request参数                                         
  auto call = controller->call_.get();
  Status s = call->SetRequestParam(req, mem_tracker_);

  if (call_local_service_) {
    # 如果是本地RPC调用
    resp->Clear();
    # 设置该call的状态为ON_OUTBOUND_QUEUE
    call->SetQueued();
    # 设置该call的状态为SENT
    call->SetSent();
    
    // If currrent thread is RPC worker thread, it is ok to call the handler in the current thread.
    // Otherwise, enqueue the call to be handled by the service's handler thread.
    # 将LocalOutboundCall转换为LocalYBInboundCall
    const shared_ptr<LocalYBInboundCall>& local_call =
        static_cast<LocalOutboundCall*>(call)->CreateLocalInboundCall();
    if (controller->allow_local_calls_in_curr_thread() && ThreadPool::IsCurrentThreadRpcWorker()) {
      # 如果当前线程是RPC线程，且RpcController中设置了允许在当前线程执行本地调用，
      # 则直接在当前线程处理，这里的context，类型为ProxyContext，实际类型为Messenger
      context_->Handle(local_call);
    } else {
      # 否则加入RPC请求队列，这里的context，类型为ProxyContext，实际类型为Messenger
      context_->QueueInboundCall(local_call);
    }
  } else {
    auto ep = resolved_ep_.Load();
    if (ep.address().is_unspecified()) {
      CHECK(resolve_waiters_.push(controller));
      Resolve();
    } else {
      # 最终调用的是context_->QueueOutboundCall(controller->call_)，将RPC请求添加到OutboundCall队列中，
      # context类型为ProxyContext，实际类型为Messenger
      #
      # 在QueueCall中会为每个OutboundCall分配一个ConnectionId，这个ConnectionId由3部分组成，分别为：
      # remote Endpoint，connection index（针对同一个Server支持多个connection），以及对应的protocol
      QueueCall(controller, ep);
    }
  }
}
```

### 直接在当前线程处理从OutboundCall转换过来的LocalInboundCall
根据前文的分析，CQLServer在向TServer发送RPC请求调用的时候，如果发现这是一个本地调用，且当前线程就是一个RPC工作线程，且当前的RPC请求可以由当前的线程进行处理，则会将OutboundCall转换为LocalInboundCall，并交给本地的Messenger进行处理。
```
void Messenger::Handle(InboundCallPtr call) {
  # 获取对应的RPC service
  auto service = rpc_service(call->service_name());
  # 交由RPC Service来处理，因为在Messenger::RegisterService中对实际注册的RPC服务进行了封装，
  # 封装成了ServicePool这个RPC service，所以这里调用的是ServicePool::Handle —>
  # ServicePoolImpl::Handle -> 真实的RPC Service的Handle，因为我们当前是沿着CQLServer向
  # TServer发送RPC请求的路径进行分析的，这里的真实的RPC Service是TabletServerService，
  # 所以调用的是TabletServerService::Handle，即TabletServerServiceIf::Handle
  # 
  # 直接在当前线程处理
  service->Handle(std::move(call));
}
```

TabletServerServiceIf::Handle中对写操作的RPC请求的处理如下：
```
void TabletServerServiceIf::Handle(::yb::rpc::InboundCallPtr call) {
  auto yb_call = std::static_pointer_cast<::yb::rpc::YBInboundCall>(call);
  if (call->method_name() == "Write") {
    # 创建RpcContext，会区分本地调用还是远程调用
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
      Write(req, resp, std::move(rpc_context));
    }
    return;
  }
  
  ...
}
```

### 将从OutboundCall转换过来的LocalInboundCall加入本地InboundCall请求队列后处理
根据前文的分析，CQLServer在向TServer发送RPC请求调用的时候，如果发现这是一个本地调用，但当前线程不是一个RPC工作线程，或者当前的RPC请求不可以由当前的线程进行处理，则会将OutboundCall转换为LocalInboundCall，并添加到LocalInboundCall的请求队列中，等待处理。
```
void Messenger::QueueInboundCall(InboundCallPtr call) {
  # 获取对应的RPC service
  auto service = rpc_service(call->service_name());

  # 加入RPC Service的InboundCall请求队列，见ServicePool::QueueInboundCall ->
  # 见ServicePoolImpl::Enqueue加入到线程池的任务队列中（队列中的元素类型是
  # InboundCallTask），等待线程池中的线程去 执行该任务，当某个线程被调度执
  # 行这个任务的时候，会调用ServicePoolImpl::Handle进行处理，最终调用真实的
  # RPC Service的Handle，因为我们当前是沿着CQLServer向 TServer发送RPC请求的
  # 路径进行分析的，这里的真实的RPC Service是TabletServerService，
  # 所以调用的是TabletServerService::Handle，即TabletServerServiceIf::Handle
  service->QueueInboundCall(std::move(call));
}
```

### 将OutboundCall添加到OutboundCall队列后处理
根据前文的分析，CQLServer在向TServer发送RPC请求调用的时候，如果发现这不是一个本地调用，则会将当前的OutboundCall添加到OutboundCall队列中，等待处理。
```
void Messenger::QueueOutboundCall(OutboundCallPtr call) {
  # 根据OutboundCall的目标来选择一个Reactor
  const auto& remote = call->conn_id().remote();
  Reactor *reactor = RemoteToReactor(remote, call->conn_id().idx());

  # 将OutboundCall添加到Reactor::outbound_queue中，等待调度处理，最终的处理逻辑为
  # Reactor::ProcessOutboundQueue
  reactor->QueueOutboundCall(std::move(call));
}

void Reactor::ProcessOutboundQueue() {
  ...
  
  # 为每一个OutboundCall分配一个Connection，并将该OutboundCall添加到该Connection的队列中
  processing_connections_.reserve(processing_outbound_queue_.size());
  for (auto& call : processing_outbound_queue_) {
    # 首先以OutboundCall对应的ConnectionId为key，到Reactor::client_conns_中查找是否已经
    # 存在关于该ConnectionId的连接，如果不存在则创建一个新的，并启动该新创建的连接（见
    # Connection::Start），然后将该新创建的连接添加到Reactor::client_conns_中，方便后续
    # 使用，最后将当前的OutboundCall添加到该连接中，见Connection::QueueOutboundCall
    auto conn = AssignOutboundCall(call);
    processing_connections_.push_back(std::move(conn));
  }
  processing_outbound_queue_.clear();

  std::sort(processing_connections_.begin(), processing_connections_.end());
  auto new_end = std::unique(processing_connections_.begin(), processing_connections_.end());
  processing_connections_.erase(new_end, processing_connections_.end());
  # 依次处理每一个连接对应的TcpStream::sending_中的数据，见Connection::OutboundQueued
  for (auto& conn : processing_connections_) {
    if (conn) {
      conn->OutboundQueued();
    }
  }
  processing_connections_.clear();
}
```

在Connection::QueueOutboundCall中会将OutboundCall添加到TcpStream::sending_中，这是一个关于OutboundData的队列，每一个OutboundCall就是一种OutboundDtata。
```
Connection::QueueOutboundCall
    - Connection::DoQueueOutboundData
        - TcpStream::Send
            - sending_.emplace_back(std::move(data), mem_tracker_);
```

在Connection::OutboundQueued中
```
Connection::OutboundQueued
    - TcpStream::TryWrite
        - TcpStream::DoWrite
            - 遍历TcpStream::sending_中每一个元素，尝试按照iovec的形式来组织这些数据，然后调用Socket::Writev来发送iovec中的数据：
                - Socket::Writev
                - 通知关于当前iovec中涉及的所有元素已经发送完毕，调用Connection::Transferred来通知其中的每一个元素
                    - Connection::Transferred
                        - RpcCall::Transferred
                            - OutboundCall::NotifyTransferred
                                - 设置OutboundCall的状态为已经发送OutboundCall::SetSent
```

## 关于Messenger::scheduler_
Messenger::scheduler_的类型是Scheduler，它提供了一系列的Schedule方法（3个），用于调度执行某个任务，这些方法最终都是通过Scheduler::DoSchedule方法实现的。

Scheduler类的具体实现则由Scheduler::Impl完成。

Scheduler在创建的时候会关联到一个boost::asio::io_service，这个boost::asio::io_service会进一步传递给Scheduler::Impl。
```
class Scheduler {
  # 构造方法
  Scheduler(IoService* io_service) : impl_(new Impl(io_service)) {}
  
  # 第一种Schedule方法
  template<class F>
  ScheduledTaskId Schedule(const F& f, std::chrono::steady_clock::duration delay) {
    return Schedule(f, std::chrono::steady_clock::now() + delay);
  }

  # 第二种Schedule方法
  template<class F>
  auto Schedule(const F& f, std::chrono::steady_clock::time_point time) ->
      decltype(f(Status()), ScheduledTaskId()) {
    auto id = NextId();
    DoSchedule(std::make_shared<ScheduledTask<F>>(id, time, f));
    return id;
  }

  # 第三种Schedule方法
  template<class F>
  auto Schedule(const F& f, std::chrono::steady_clock::time_point time) ->
      decltype(f(ScheduledTaskId(), Status()), ScheduledTaskId()) {
    auto id = NextId();
    DoSchedule(std::make_shared<ScheduledTaskWithId<F>>(id, time, f));
    return id;
  }
  
  # 上面3中Schedule方法都是借助Scheduler::Impl::DoSchedule来实现的
  DoSchedule(std::shared_ptr<ScheduledTaskBase> task) {
    impl_->DoSchedule(std::move(task));
  }
  ...
}
```

Scheduler::Impl::DoSchedule的实现如下：
```
class Scheduler::Impl {
 public:
  # io_service就是来自于Scheduler构造方法中的参数
  # 
  # 这里借助boost::asio::io_service::strand来提供顺序化的事件执行器，即，
  # 如果以“work1->work2->work3”的顺序post任务，则不管有多少个工作线程，
  # 它们依然会以这样的顺序执行任务
  # 
  # 这里借助boost::asio::steady_timer来实现定时器功能
  explicit Impl(IoService* io_service)
      : io_service_(*io_service), strand_(*io_service), timer_(*io_service) {}

  void DoSchedule(std::shared_ptr<ScheduledTaskBase> task) {
    # 提交任务，通过strand_确保任务按照提交的顺序被执行？
    strand_.dispatch([this, task] {
      # 如果当前Scheduler被关闭
      if (closing_.load(std::memory_order_acquire)) {
        io_service_.post([task] {
          task->Run(STATUS(Aborted, "Scheduler shutdown", "" /* msg2 */, Errno(ESHUTDOWN)));
        });
        return;
      }

      # 否则将当前的任务添加到任务队列中
      auto pair = tasks_.insert(task);
      CHECK(pair.second);
      if (pair.first == tasks_.begin()) {
        StartTimer();
      }
    });
  }

  typedef boost::multi_index_container<
      std::shared_ptr<ScheduledTaskBase>,
      boost::multi_index::indexed_by<
          ordered_non_unique<
              const_mem_fun<ScheduledTaskBase, SteadyTimePoint, &ScheduledTaskBase::time>
          >,
          hashed_unique<
              boost::multi_index::tag<IdTag>,
              const_mem_fun<ScheduledTaskBase, ScheduledTaskId, &ScheduledTaskBase::id>
          >
      >
  > Tasks;

  IoService& io_service_;
  std::atomic<ScheduledTaskId> id_ = {0};
  Tasks tasks_;
  // Strand that protects tasks_ and timer_ fields.
  boost::asio::io_service::strand strand_;
  boost::asio::steady_timer timer_;
  int timer_counter_ = 0;
  std::atomic<bool> closing_ = {false};
};
```

### Messenger::scheduler_所关联的boost::asio::io_service来自于哪里
在Messenger的构造方法中会调用Scheduler的构造方法来创建Messenger::scheduler_，在Scheduler的构造方法中用到的boost::asio::io_service来自于Messenger::io_thread_pool_.io_service()，最终则来自于IoThreadPool::Impl::io_service_。在IoThreadPool::Impl中会将IoThreadPool::Impl::threads_关联到IoThreadPool::Impl::io_service_，其中IoThreadPool::Impl::io_service_用于分发任务，而IoThreadPool::Impl::threads_是一个线程池/线程数组，用于执行经由IoThreadPool::Impl::io_service_分发的任务。
```
Messenger::Messenger(const MessengerBuilder &bld)
    : name_(bld.name_),
      connection_context_factory_(bld.connection_context_factory_),
      stream_factories_(bld.stream_factories_),
      listen_protocol_(bld.listen_protocol_),
      metric_entity_(bld.metric_entity_),
      io_thread_pool_(name_, FLAGS_io_thread_pool_size),
      scheduler_(&io_thread_pool_.io_service()),
      normal_thread_pool_(new rpc::ThreadPool(name_, bld.queue_limit_, bld.workers_limit_)),
      rpc_metrics_(new RpcMetrics(bld.metric_entity_)),
      num_connections_to_server_(bld.num_connections_to_server_) {
#ifndef NDEBUG
  creation_stack_trace_.Collect(/* skip_frames */ 1);
#endif
  VLOG(1) << "Messenger constructor for " << this << " called at:\n" << GetStackTrace();
  for (int i = 0; i < bld.num_reactors_; i++) {
    reactors_.emplace_back(std::make_unique<Reactor>(this, i, bld));
  }
}
```

**综上分析可知，Messenger::scheduler_会借助Messenger::io_thread_pool_.io_service()来进行任务调度，这些任务最终由关联到Messenger::io_thread_pool_.io_service()的线程池来执行。**

**Messenger::scheduler_提供了3中Schedule接口来进行任务调度。**















