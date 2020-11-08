# 提纲
[toc]

## YugaByte中所用到的线程
1. 在Messenger构建过程中，会创建Messenger::io_thread_pool_，它的类型是IoThreadPool，具体实现是在IoThreadPool::Impl中，它包括一个线程数组和一个boost::asio::io_service，应用通过boost::asio::io_service来发布任务，最终由线程数组中的线程来执行，每个线程的名字是“iotp_” + Messenger::name() + ${线程编号}，每个线程的运行主体是IoThreadPool::Impl::Execute，它会执行调度给它的任务（类型为ScheduledTaskBase），线程池中线程数目由FLAGS_io_thread_pool_size进行控制；

    Messenger::io_thread_pool_会关联到一个boost::asio::io_service，主要用于执行通过boost::asio::io_service分发的任务，而boost::asio::io_service主要被Messenger::Scheduler()使用，Messenger::Scheduler()则主要在以下地方被使用：
    - ServicePool中用于调度一些任务，到底是什么任务？


2. 在Messenger构建过程中，会创建Messenger::normal_thread_pool_，它的类型是ThreadPool，具体实现是ThreadPool::Impl，在ThreadPool::Impl中有一系列的Worker，在构建每一个Worker的时候会为之创建一个线程，线程的执行主体是Worker::Execute，它会先后执行分配给它的ThreadPoolTask的run()和Done()方法，线程池内Worker的数目最终是通过FLAGS_rpc_workers_limit控制的；

    ThreadPool是一个线程池，应用调用ThreadPool::Enqueue来将ThreadPoolTask添加到ThreadPool的任务队列中，然后ThreadPool将这些ThreadPoolTask分配给线程池内的各Workers。

    在Messenger::ThreadPool(rpc::ServicePriority::kNormal)获取ThreadPool的时候，返回的是Messenger::normal_thread_pool_，它主要在以下地方被使用：
    - 在RpcServerBase::Start()中，注册GenericServiceImpl时会使用；
    - 在CQLServer::Start()中，注册CQLServiceImpl时会使用；
    - 在Master::RegisterServices()中，注册MasterServiceImpl、MasterTabletServiceImpl和RemoteBootstrapServiceImpl时会使用；
    - 在TabletServer::RegisterServices()中，注册TabletServiceImpl、TabletServiceAdminImpl和RemoteBootstrapServiceImpl时会使用；
    - RPC请求回调的执行，如果没有强制在Reactor线程上执行，则会在该线程池中执行（见Proxy::DoAsyncRequest）；

3. Messenger中还有一个叫做Messenger::high_priority_thread_pool_的成员域，它在Messenger构建的过程中并不会创建，而是在Messenger::ThreadPool()调用过程中，如果指定获取的是high priority的线程池，则会检查Messenger::high_priority_thread_pool_是否存在，如果存在则直接返回，否则创建之，Messenger::high_priority_thread_pool_中每个线程的名字是Messenger::name() + "-high-pri"，线程池的具体实现是ThreadPool::Impl，在ThreadPool::Impl中有一系列的Worker，在构建每一个Worker的时候会为之创建一个线程，线程的执行主体是Worker::Execute，它会先后执行分配给它的ThreadPoolTask的run()和Done()方法，线程池内Worker的数目最终是通过FLAGS_rpc_workers_limit控制的；

    ThreadPool是一个线程池，应用调用ThreadPool::Enqueue来将ThreadPoolTask添加到ThreadPool的任务队列中，然后ThreadPool将这些ThreadPoolTask分配给线程池内的各Workers。
    
    在Messenger::ThreadPool(rpc::ServicePriority::kHigh)获取ThreadPool的时候，返回的是Messenger::normal_thread_pool_，它主要在以下地方被使用：
    - 在Master::RegisterServices()中，注册ConsensusServiceImpl时会使用；
    - 在TabletServer::RegisterServices()中，注册ConsensusServiceImpl时会使用；

4. 在Messenger初始化过程中（Messenger::Init）会为Messenger::reactors_中的每个Reactor创建一个线程，线程的名字是Messenger::name() + "_reactor"，线程的运行主体是Reactor::RunThread，线程的数目就是Messenger::reactors_这个数组的大小，可以通过FLAGS_num_reactor_threads进行设置；

5. 在RpcServer启动过程中（RpcServer::Start）会为Messenger::acceptor_创建一个线程，线程的名字是acceptor，线程的运行主体是Acceptor::RunThread，它会不断的处理新的连接请求，并最终调用Messenger::RegisterInboundSocket来将新的连接请求对应的连接注册到Reactor中，每个Messenger会对应有一个Acceptor线程；

## 这些线程之间的关系
### 关于RPC请求的监听 - 接收 - 处理 - 发送(进一步调用)
1. Messenger::acceptor_用于处理新的连接请求，并调用Messenger::RegisterInboundSocket来将新的连接请求对应的连接注册到Reactor中，Messenger中会有多个Reactors，具体选择哪个Reactor，则通过RemoteToReactor()方法来确定；注册到Reactor中的Connection，会立即启动一个Stream，假设是TcpStream，并监听该连接上的Read事件，同时设置事件处理方法为TcpStream::Handler。

2. （以CQLServer为例进行说明）在Reactor线程中，当TcpStream接收到可read的事件的时候，会调用TcpStream::Handler -> TcpStream::ReadHandler -> ... -> CQLConnectionContext::HandleCall进行处理，在CQLConnectionContext::HandleCall中会创建一个CQLInboundCall，并将这个CQLInboundCall提交给相应的RPCService进行处理（RpcService::QueueInboundCall），每个RPCService都会绑定到Messenger::normal_thread_pool_或者Messenger::high_priority_thread_pool_线程池中，提交给RPCService的CQLInboundCall最终由线程池来处理。

3. （以CQLServer为例进行说明）Messenger::normal_thread_pool_或者Messenger::high_priority_thread_pool_两个线程池中的线程类型都是Worker，它的运行主体是Worker::Execute，其中会通过调用分配给它的任务的Run方法来执行分配给它的任务。每一个CQLInboundCall都是一个InboundCall，在InboundCall中包含一个成员：InboundCallTask，于是在RpcService::QueueInboundCall -> ... -> ServicePoolImpl::Enqueue -> 将该InboundCall::task_(类型是InboundCallTask)添加到线程池的任务队列中，等待被调度处理，最终由线程池中的Worker进行处理，处理逻辑在Worker::Execute -> ... -> ServicePoolImpl::Handle -> 调用真正的RPCService的Handle方法（如CQLServiceImpl::Handle） -> 获取一个可用的CQLProcessor，并最终调用CQLProcessor::ProcessCall进行处理。

    在CQLProcessor::ProcessCall处理某个RPC调用的时候，可能会涉及跟其他组件的交互，比如CQLServer在执行Write操作的时候，可能会进一步跟Master进行交互以获取Tablet信息，也可能会进一步跟TServer进行交互以真正执行写相关的操作。

    以CQLServer和TServer交互为例，如果是和本地的TServer进行交互，则又进一步分为两种情况：
    - 当前线程就是一个RPC工作线程，且当前的RPC请求可以由当前的线程进行处理，则直接在当前线程进行处理；
        - 当RPC请求被执行完毕之后，RPC请求执行完成的回调会在当前线程被执行；
    - 否则，将RPC调用添加到相应的RPCService的LocalInboundCall的请求队列中等待处理，最终由RPCService所关联的线程池来进行处理；
        - 当RPC请求被执行完毕之后，RPC请求执行完成的回调会在真正执行该RPC请求的线程中被执行；

    如果是和远程的TServer进行交互，则：
    - 根据OutboundCall的目标来选择一个Reactor，并将当前的RPC调用添加到Reactor的OutboundCall队列中，等待处理，于是将RPC调用交给了Reactor线程来进行处理，最终由该Reactor线程来负责发送RPC消息，在发送之前如果没有建立连接，则需要先建立一个连接，然后通过该连接来发送；
        - 当RPC请求被执行完毕之后，RPC请求执行完成的回调会在哪个线程被执行又进一步分为两种情况：
            - 如果force_run_callback_on_reactor被设置，则回调在接收到该RPC请求执行完毕（无论是否成功）的消息所在的Reactor线程中被执行；
            - 否则，回调在Messenger::normal_thread_pool_这个线程池中被处理；

### CQLServer端线程情况
因为“关于RPC请求的监听 - 接收 - 处理 - 发送(进一步调用)”是以CQLServer为例进行阐述的，所以可以直接参考“关于RPC请求的监听 - 接收 - 处理 - 发送(进一步调用)”。

### TServer端线程情况
TServer端关于RPC请求的监听 - 接收 - ~~处理~~ - 发送(进一步调用)流程跟CQLServer端类似，区别在于RPC请求处理阶段，下面进行分析。

