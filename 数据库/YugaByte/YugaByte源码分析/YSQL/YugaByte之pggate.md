# 提纲
[toc]

## 说明
从“YugaByte之修改操作”分析来看，YugaByte在PostgreSQL代码中加入了自己的代码，以求将DocDB整体作为PostgreSQL的后端存储，但是YugaByte在PostgreSQL中加入的代码最终都是走YugaByte在yql中定义的pggate来实现的。鉴于此，我将pggate理解为PostgreSQL和DocDB之间的适配层，因此通过分析pggate就可以得知它需要怎样的一个DocDB，进一步的，DocDB又需要怎样的一个存储引擎。

## PgGate初始化 - YBCInitPgGate
```
PostgresMain
    - InitPostgres
        - YBInitPostgresBackend
            - YBCGetTypeTable(&type_table, &count)
                - 获取PostgreSQL和YugaByte类型之间的转换表，在静态变量YBCTypeEntityTable中定义
            - YBCInitPgGate(type_table, count)
                - pgapi = new pggate::PgApiImpl(YBCDataTypeTable, count)
                    # 创建rpc::Messenger，reactor数目通过参数FLAGS_pggate_ybclient_reactor_threads
                    # 进行控制，默认是2
                    - BuildMessenger
                        - client::CreateClientMessenger
                            - 创建rpc::Messenger
                    # AsyncClientInitialiser类用于异步的创建YBClient，YBClient被用来和yb-master
                    # 或者yb-tserver通信
                    - 通过AsyncClientInitialiser的构造方法来创建async_client_init_
                        - 主要是初始化类型为YBClientBuilder的client_builder_，供后续创建YBClient使用
                            - 其中会关联到前面创建的rpc::Messenger
                            - 其中会设置yb-master的地址(用于设置YBClient中的master_server_addrs_)
                    # 创建混合时钟
                    - clock_ = new server::HybridClock()
                    # 初始化pg_txn_manager_
                    - pg_txn_manager_ = new PgTxnManager(&async_client_init_, clock_, tserver_shared_object_.get())
                        - 根据参数设置async_client_init_
                        - 根据参数这只混合时钟clock_
                        - ...
                    # 初始化混合时钟
                    - clock_->Init
                    - 设置PostgreSQL和YugaByte之间的类型映射，其实是对静态变量YBCTypeEntityTable的转换
                    # 启动async_client_init_
                    - async_client_init_.Start()
                        # 实际上是启动一个线程，线程的运行主体是AsyncClientInitialiser::InitClient
                        # 该线程的主要作用是构建YBClient
                        - init_client_thread_ = std::thread(std::bind(&AsyncClientInitialiser::InitClient, this))
                            - YBClientBuilder::Build
                                - YBClientBuilder::DoBuild
                                    - 新创建一个YBClient
                                    - 借助于YBClientBuilder中保存的信息来初始化新创建的YBClient
```

## 创建Insert请求 - YBCPgNewInsert
```
YBCPgNewInsert
    - const PgObjectId table_id(database_oid, table_oid)
    - PgApiImpl::NewInsert
        - 创建类型为PgInsert的PgStatement
        - PgDmlWrite::Prepare
            # PgSession中封装了YBClient，YBSession，PgTxnManager，HybridClock等，
            # 其中还有一个关于YBTable的cache(TableId到YBTable的映射表)
            - PgSession::LoadTable
                - 首先在table_cache_中查找对应的YBTable
                - 如果找到了，则直接返回对应的YBTable
                - 如果没有找到，则通过YBClient去获取之
                    - YBClient::OpenTable
                        - 新构建一个YBTable
                        - YBTable::Open
                            - 发送类型为GetTableLocationsRequestPB的RPC请求给Master，获取Table对应的partition信息，保存在YBTable::partitions_中
                    - 将获取的YBTable添加到PgSession的table_cache_中
            # 分配INSERT，UPDATE或者DELETE请求
            - AllocWriteRequest
                # 对于insert请求
                - PgInsert::AllocWriteRequest
                    # 分配YBPgsqlWriteOp
                    - client::YBPgsqlWriteOp *insert_op = table_desc_->NewPgsqlInsert()
                    # 设置is_single_row_txn_标识，表示是单行事务还是分布式事务
                    - insert_op->set_is_single_row_txn(is_single_row_txn_)
                    # 创建PgDocWriteOp 
                    - doc_op_ = make_shared<PgDocWriteOp>(pg_session_, insert_op)
                    # 从PgDocWriteOp中获取PgsqlWriteRequestPB
                    - write_req_ = doc_op->write_op()->mutable_request()
            - PrepareColumns
```

## 获取table descriptor - YBCPgGetTableDesc
```
YBCPgGetTableDesc
    - PgApiImpl::GetTableDesc
        # 返回PgTableDesc
        # LoadTable会首先在PgSession对应的table_cache_中查找，如果没找到则去Master上获取
        # 并缓存到table_cache_中，方便后续使用
        - pg_session->LoadTable(table_id)
```

## 获取某一列是否是primary key，是否是partition key
```
YBCPgGetColumnInfo
    # table_desc和attr_number作为输入，is_primary和is_hash作为输出
    - pgapi->GetColumnInfo(table_desc, attr_number, is_primary, is_hash)
        - table_desc->GetColumnInfo(attr_number, is_primary, is_hash)
            
            - 到attr_num_map_中查找给定的attr_number，也就是column编号，对应的在columns_数组中的索引号
            - 根据返回的索引号到columns_数组中找到指定的元素，就是对应的ColumnDesc
            - 通过ColumnDesc中的信息来设置is_primary和is_hash
```

## 构建YugaByte DocKey - YBCPgDmlBuildYBTupleId
```
# @handle表示当前的PgStatement，@attrs表示column及其对应的value信息，
# @nattrs表示@attrs数组的大小，@ybctid作为输出参数
YBCPgDmlBuildYBTupleId(YBCPgStatement handle, const YBCPgAttrValueDescriptor *attrs,
                     int32_t nattrs, uint64_t *ybctid)
    - PgApiImpl::DmlBuildYBTupleId
        # 构建DocKey
        - PgDml::BuildYBTupleId
            # 下面的两层遍历的原因是：DocDB要求DocKey中partition columns必须以它们
            # 被创建表的时候相同的顺序存放
            - 遍历table_desc_->columns()中的每个Column @c
                - 遍历PgAttrValueDescriptor数组@attrs中的每个元素@attr
                    - 如果满足attr->attr_num == c.attr_num()，则表示在@attrs中找到了@c对应的column
                        - 如果当前的column是YugaByte的自增的row id(对于没有指定primary key的情况下，YugaByte会使用自增的row id作为primary key)，则：
                            - 为之生成一个新的row id
                        - 检查当前column是hash key还是range key
                            - 如果是hash key，则将该column对应的value添加到@hash_components数组中
                            - 否则，将该column对应的value添加到@range_components数组中
                        - 结束本轮遍历
                    - 否则，继续遍历
            # 生成DocKey
            - 如果@hash_components数组中没有元素
                - docdb::DocKey(move(range_components)).Encode().data()
            - 否则：
                # 生成partition key，
                - string partition_key;
                  const PartitionSchema& partition_schema = table_desc_->table()->partition_schema()
                  partition_schema.EncodeKey(hashed_values, &partition_key)
                # 计算出hash值
                -  const uint16_t hash = PartitionSchema::DecodeMultiColumnHashValue(partition_key)
                # 生成DocKey
                - docdb::DocKey(hash, move(hashed_components), move(range_components)).Encode().data()
        # 将DocKey转换为PostgreSQL在内存中的datam，参数中的id就是DocKey所在的字符串，
        # 返回的ybctid就是一个地址，转换方法在YBCTypeEntityTable中事先已经定义好了
        - *ybctid = type_entity->yb_to_datum(id.data(), id.size(), nullptr /* type_attrs */)
```

## 绑定value到某个column上 - YBCPgDmlBindColumn
```
YBCPgDmlBindColumn(PgStatement *handle, int attr_num, PgExpr *attr_value)
    - PgApiImpl::DmlBindColumn
        - PgDml::BindColumn
            # 从table descriptor中找到@attr_num所代表的column
            - PgColumn *col = nullptr
              FindColumn(attr_num, &col))
            # 为该column准备一个PgsqlExpressionPB @bind_pb
            - PgsqlExpressionPB *bind_pb = col->bind_pb()
            # 关联PgsqlExpressionPB和PgExpr，因为这个PgsqlExpressionPB是column中的一个成员，
            # 因此也就将column和value(PgExpr)关联起来了
            - expr_binds_[bind_pb] = attr_value
```

## 执行insert/update/delete操作 - YBCPgDmlExecWriteOp
```
YBCPgDmlExecWriteOp
    - PgApiImpl::DmlExecWriteOp
        - PgDmlWrite::Exec
            # 设置protobuf中引用到的columns(在column上读或者写)
            - SetColumnRefIds(table_desc_, write_req_->mutable_column_refs())
            # 发送请求，返回请求是否已经被发送，如果当前操作被buffered，则不算做已经发送
            - PgDocOp::Execute
                # 加锁(PgDocOp内部的锁)
                - std::unique_lock<std::mutex> lock(mtx_)
                # 检查是否可以发送当前请求
                - InitUnlocked(&lock)
                    # 如果正在等待响应的过程中，则持续等待，直到接收到响应(此时条件等待会被唤醒)
                    # waiting_for_response_如果为true，则表示正在等待上一个发送的请求的响应，在
                    # 接收到响应之后才能发送下一个请求
                    - if (waiting_for_response_) {
                        # 条件等待，直到唤醒
                        while (waiting_for_response_) {
                          cv_.wait(*lock);
                        }
                      
                        CHECK(!waiting_for_response_);
                      }
                      
                      # 至此，表示可以发送下一个请求了，清空result cache
                      result_cache_.clear()
                # 发送请求
                - SendRequestUnlocked
                    - PgDocWriteOp::SendRequestUnlocked
                        # 执行操作，如果当前写操作被buffered，则返回true，否则返回false
                        # 如果当前写操作被buffered，则将执行批量flush，否则立即执行flush
                        - PgSession::PgApplyAsync
                            # 如果当前操作是写类型的操作，且当前被buffered write operations不为0，则
                            # 当前的写操作被buffered，如果当前操作是事务型操作，则添加到buffered_txn_write_ops_
                            # 中，否则添加到buffered_write_ops_，返回OpBuffered::kTrue，表示当前事务被buffered
                            - if (buffer_write_ops_ > 0 && op->type() == YBOperation::Type::PGSQL_WRITE) {
                                if (op->IsTransactional()) {
                                  buffered_txn_write_ops_.push_back(op);
                                } else {
                                  buffered_write_ops_.push_back(op);
                                }
                                
                                # 返回OpBuffered::kTrue，表示当前操作被buffered
                                return OpBuffered::kTrue;
                              }
                            # 获取一个YBSession，用于执行当前的操作
                            - GetSessionForOp(op)
                                - PgSession::GetSession(bool transactional, bool read_only_op)
                                    - 如果是事务性操作(transactional为true)，则：
                                        - YBSession* txn_session = pg_txn_manager_->GetTransactionalSession()
                                            # 如果txn_in_progress_为false，表示当前不存在活跃的事务，重新启动一个事务，
                                            # 在启动事务的过程中会创建新的YBSession
                                            # 如果txn_in_progress_为true，则直接复用当前的YBSession
                                            - if (!txn_in_progress_) {
                                                RETURN_NOT_OK(BeginTransaction());
                                              }
                                              return session_.get();
                                        # 从这个方法的名字上看很容易认为是写类型的事务，事实上对读类型的事务也会走到这里
                                        - pg_txn_manager_->BeginWriteTransactionIfNecessary(read_only_op)
                                            # 如果当前已经存在一个事务，则直接返回ok
                                            - if (txn_) {
                                                return Status::OK();
                                              }
                                            # 否则，如果PostgresMain和TServer之间通过共享内存通信，则发送RPC请求给TabletServer，
                                            # 从TabletServer的transaction pool中获取一个transaction，当前是在PostgresMain
                                            # 进程中，只需要返回这个transaction的metadata即可，metadata保存在response中
                                            # 关于TakeTransaction，参考“PgTxnManager向tablet server发送的TakeTransaction
                                            # RPC到底做了什么”
                                                # 发送rpc给TabletServer，获取一个可用的transaction，在@resp中会包含该transaction
                                                # 相关的metadata信息
                                                - tablet_server_proxy_->TakeTransaction(req, &resp, &controller)
                                                # 根据rpc响应中的metadata信息在当前的PostgresMain进程中创建一个YBTransaction
                                                # GetOrCreateTransactionManager()用于获取client::TransactionManager，它不同于
                                                # PgTxnManager，前者是YQL层的TransactionManager
                                                - txn_ = YBTransaction::Take(GetOrCreateTransactionManager(),
                                                  VERIFY_RESULT(TransactionMetadata::FromPB(resp.metadata())))
                                                    # 创建YBTransaction
                                                    - auto result = std::make_shared<YBTransaction>(manager, metadata, PrivateOnlyTag())
                                                        # 初始化YBTransaction::Impl，包括readpoint，transaction id，transaction priority
                                                        - impl_ = new Impl(manager, this, metadata)
                                                            - manager_= manager
                                                            - transaction_ = transaction
                                                            - metadata_ = metadata
                                                            - read_point_ = manager->clock()
                                                            - child_ = Child::kFalse
                                                    - result->impl_->StartHeartbeat()
                                                        # 查找Status tablet
                                                        - RequestStatusTablet(TransactionRpcDeadline())
                                                            # 如果metadata中没有status tablet信息，则由TransactionManager挑选一个，当
                                                            # 挑选到的时候Impl::StatusTabletPicked -> Impl::LookupStatusTablet会被调用
                                                            # 如果metadata中包含status tablet信息，则查找对应的status tablet，当找到之
                                                            # 后，Impl::LookupTabletDone会被调用，因为RPC是异步的，所以这里不会阻塞
                                                            # YBTransaction的后续操作
                                                            -   if (metadata_.status_tablet.empty()) {
                                                                  manager_->PickStatusTablet(
                                                                      std::bind(&Impl::StatusTabletPicked, this, _1, deadline, transaction));
                                                                } else {
                                                                  LookupStatusTablet(metadata_.status_tablet, deadline, transaction);
                                                                }
                                            - 否则直接分配一个transaction(注意：这里没有传递metadata作为参数，在构造方法中没有设置metadata中的status_tablet信息)：
                                                - txn_ = std::make_shared<YBTransaction>(GetOrCreateTransactionManager())
                                                    - manager_= manager
                                                      transaction_ = transaction
                                                      read_point_= manager->clock()
                                                      child_= Child::kFalse
                                                    # 分配transaction id
                                                    - metadata_.transaction_id = GenerateTransactionId()
                                                    # 分配一个优先级
                                                    - metadata_.priority = RandomUniformInt<uint64_t>()
                                            - 如果隔离级别是SNAPSHOT_ISOLATION，则：
                                                - txn_->InitWithReadPoint(isolation, std::move(*session_->read_point()))
                                                    - Impl::InitWithReadPoint
                                                        # 设置元数据中记录的隔离级别
                                                        - metadata_.isolation = isolation
                                                        # 设置read point
                                                        - read_point_ = std::move(read_point)
                                            - 否则，是其它隔离级别的话：
                                                - txn_->Init(isolation)
                                            - session_->SetTransaction(txn_)
                                                # 设置YBSession的当前的transaction
                                                - transaction_ = std::move(transaction)
                                                # 将当前的batcher_交换到old_batcher中，同时将batcher_重置，如果old_batcher
                                                # 不为空，则终止之
                                                - internal::BatcherPtr old_batcher;
                                                  old_batcher.swap(batcher_);
                                                  if (old_batcher) {
                                                    old_batcher->Abort(STATUS(Aborted, "Transaction changed"));
                                                        - 终止当前所有处于in-flight状态的operations
                                                        - 运行flush_callback_，也就是FlushAsyncDone
                                                  }
                                        - return txn_session
                                    - 否则：
                                        # 直接返回当前的YBSession
                                        - return session_.get()
                            # 该方法我们之前在CQL的分析中分析过了，包括后面的逻辑也都分析过了
                            - YBSession::Apply
                                # 将当前的操作添加到Batcher的inflight operation集合中，并查找相应的tablet
                                - Batcher().Add
                                    # 将当前的操作添加到Batcher的inflight operation集合@ops_中
                                    - Batcher::AddInFlightOp
                                        # 插入
                                        - ops_.insert(op)
                                        # 增加正在查找tablet的操作的计数
                                        - ++outstanding_lookups_
                                    # 查找操作对应的tablet(可能在meta cache中找到，也可能需要发送rpc查找)，
                                    # 如果找到之后，Batcher::TabletLookupFinished会被调用
                                    - client_->data_->meta_cache_->LookupTabletByKey(..., 
                                      std::bind(&Batcher::TabletLookupFinished, BatcherPtr(this), in_flight_op, _1))
                            - 返回OpBuffered::kFalse，表示当前操作没有被buffered
                        # 至此，表明PgSession::PgApplyAsync返回的是OpBuffered::kFalse，也就是说
                        # 当前操作没有被buffered
                        # 设置在发送下一个请求之前，需要等待当前请求的响应
                        - waiting_for_response_ = true
                        # 如果当前操作没有被buffered，则flush之，这里传递了一个回调PgDocWriteOp::ReceiveResponse，
                        # 在操作被成功flush，且接收到了响应之后，该回调被调用
                        - PgSession::PgFlushAsync
                            # 该方法我们之前在YCQL的分析中分析过了，包括后面的逻辑也都分析过了
                            - YBSession::FlushAsync
                                #  结束当前的Batcher，将之交换到old_batcher中，同时重置当前的Batcher
                                - internal::BatcherPtr old_batcher;
                                  old_batcher.swap(batcher_);
                                # 将old_batcher添加到待flush的batcher集合中
                                - flushed_batchers_.insert(old_batcher)
                                # 异步的flush
                                - old_batcher->FlushAsync(std::move(callback))
                                    # 检查是否当前的batcher中的所有的操作都已经被flush，且接收
                                    # 到了响应，如果是则调用回调PgDocWriteOp::ReceiveResponse
                                    - CheckForFinishedFlush
                                    # 如果还有待flush的操作，则flush之
                                    - FlushBuffersIfReady
                                        - 如果所有发送出去的tablet lookup请求都接收到了Batcher::TabletLookupFinished回调，且当前Batcher的状态不再是ResolvingTablets，则将Batcher状态修改为kTransactionPrepare，表示进入prepare阶段，比如查找status tablet等
                                        - 对所有待flush的操作进行排序
                                        # 执行操作(实际上是发送rpc到tablet server上执行)
                                        - Batcher::ExecuteOperations
                                            - auto transaction = this->transaction()
                                            # 见YBTransaction::Prepare
                                            # 在YBTransaction::Prepare中传递了一个回调
                                            # Batcher::TransactionReady，如果在执行
                                            # YBTransaction::Prepare的时候transaction还没有
                                            # 处于ready状态的话，则会等待YBTransaction::Impl::
                                            # LookupTabletDone被执行之后，再调用这里的回调
                                            - transaction->Prepare(..., 
                                            std::bind(&Batcher::TransactionReady, this, _1, BatcherPtr(this)), &transaction_metadata_)
                                            - 如果transaction prepare返回false，则表明当前还没有找到status tablet，直接返回，那么在这种情况下，何时才能flush Batcher中的操作呢，原来在YBTransaction::Prepare中传递了一个回调Batcher::TransactionReady，当该回调被调用的时候，他就会再次执行Batcher::ExecuteOperations，就会flush这些操作了
                                            # 至此，prepare ok了
                                            - 修改Batcher的状态为BatcherState::kTransactionReady
                                            - 对所有待flush的操作按照tablet和operation group进行分组，同一组的操作生成一个rpc，一并发送，生成的rpc都存放在@rpcs中
                                            - 逐一发送rpc
            - 检查发送状态：如果请求被buffered，则不算已经发送，否则认为已经发送
                # 如果请求未被buffered，则获取执行结果
                - doc_op_->GetResult(&row_batch_)
                    - SendRequestIfNeededUnlocked
                        # 如果cache中没有数据，且尚未完全获取到所需要的数据，且当前不在
                        # 等待响应的过程中，则再次发送请求
                        - PgDocWriteOp::SendRequestUnlocked
                    # 等待DocDB的响应
                    - while (!has_cached_data_ && !end_of_data_) {
                        cv_.wait(lock);
                      }
                    # 从cache中读取响应结果
                    - ReadFromCacheUnlocked
                    # 如果已经消费完了cache中的数据，则这里会预取后续数据
                    - SendRequestIfNeededUnlocked
```

### PgTxnManager向tablet server发送的TakeTransaction RPC到底做了什么
```
void TabletServerServiceIf::Handle(::yb::rpc::InboundCallPtr call) {
  ...
  
  if (call->method_name() == "TakeTransaction") {
    ...
    
    if (!rpc_context.responded()) {
      const auto* req = static_cast<const TakeTransactionRequestPB*>(rpc_context.request_pb());
      auto* resp = static_cast<TakeTransactionResponsePB*>(rpc_context.response_pb());
      # 实际调用的是TabletServiceImpl::TakeTransaction
      TakeTransaction(req, resp, std::move(rpc_context));
    }
    return;
  }
}

TabletServiceImpl::TakeTransaction
    # 从tserver的transaction pool中分配一个transaction，执行以下步骤：
    # 1. 如果transaction pool中有可用的transaction，则它一定是已经执行过YBTransaction::Prepare的
    #    (其中会查找一个status tablet供其使用)，直接pop出来一个
    # 2. 如果transaction pool中没有可用的transaction，则临时分配一个transaction，给当前的调用，
    #    但是这个临时分配的transaction是没有执行YBTransaction::Prepare的
    # 3. 创建一个新的transaction，并调用YBTransaction::Prepare来查找可用的status tablet，当
    #     成功查找到status tablet，也就是LookupTabletDone被调用之后，YBTransaction::Prepare
    #     中设定的回调YBTransaction::Impl::TransactionReady会被调用，在其中会将创建的新的
    #     transaction添加到transaction pool中
    #
    # 从上面的分析可知，TakeTransaction就是为了从tablet server的transaction pool中返回一个
    # transaction，这个transaction可能是已经执行过YBTransaction::Prepare的，那么它的metadata
    # 中就已经包括了status tablet信息，这个transaction也可能是一个临时分配的一个transaction，
    # 那么他的metadata中就不包括status tablet信息。TakeTransaction在每次执行时，都会再次分配
    # 一个新的transaction，并调用YBTransaction::Prepare，这是为了方便后续TakeTransaction请求
    # 到达时，有可用的transaction。
    - auto transaction = server_->TransactionPool()->Take()
    - auto metadata = transaction->Release()
        # 从代码来看，只有transaction处于TransactionState::kRunning状态才会去release，否则不
        # release，那么这里执行release操作的意图何在呢？
        - YBTransaction::Impl::Release
    # 将metadata信息保存到response中
    - metadata->ForceToPB(resp->mutable_metadata())
    # 发送rpc响应
    - context.RespondSuccess()
```


### 查找到status tablet之后的回调 - LookupTabletDone
之前我们分析过LookupTabletFinished，这里又有一个LookupTabletDone，前者是查找到操作所对应的tablet的时候的回调，而后者则是在查找到status tablet的时候的回调。
```
YBTransaction::Impl::LookupTabletDone(const Result<client::internal::RemoteTabletPtr>& result,
                const YBTransactionPtr& transaction)
    # 如果响应中提示错误，则通知处于等待队列中的所有的waiters，然后返回
    - if (!result.ok()) {
        NotifyWaiters(result.status());
        return;
      }
    # 根据响应来设置status tablet
    - status_tablet_ = std::move(*result)
    # 如果当前的metadata中记录的status tablet id是空的，则根据响应来设置
    # metadata中的status tablet id
    - if (metadata_.status_tablet.empty())
        metadata_.status_tablet = status_tablet_->tablet_id();
    # 否则，设置当前事务为ready状态，然后调用等待队列中的回调，并清空等待队列
    - ready_ = true
      for (const auto& waiter : waiters) {
        waiter(Status::OK());
      }
      waiters_.swap(waiters)
    # 向status tablet发送心跳
    - SendHeartbeat
        # 创建一个UpdateTransactionRequestPB，并设置之
        - tserver::UpdateTransactionRequestPB req;
          req.set_tablet_id(status_tablet_->tablet_id());
          req.set_propagated_hybrid_time(manager_->Now().ToUint64());
          auto& state = *req.mutable_state();
          state.set_transaction_id(metadata_.transaction_id.begin(), metadata_.transaction_id.size());
          state.set_status(status)
        # 注册一个rpc call和一个heartbeat handle，不断的向status tablet发送心跳
        - manager_->rpcs().RegisterAndStart(
            # 生成一个transaction rpc，并设置rpc的回调为Impl::HeartbeatDone
            UpdateTransaction(
                TransactionRpcDeadline(),
                status_tablet_.get(),
                manager_->client(),
                &req,
                std::bind(&Impl::HeartbeatDone, this, _1, _2, status, transaction)),
            &heartbeat_handle_)
```

#### 跟status tablet之间的心跳rpc的回调 - Impl::HeartbeatDone
```
YBTransaction::Impl::HeartbeatDone
    # 如果心跳成功，则每个一定时间再次发送心跳
    - if (status.ok()) {
        if (transaction_status == TransactionStatus::CREATED) {
          NotifyWaiters(Status::OK());
        }
        std::weak_ptr<YBTransaction> weak_transaction(transaction);
        manager_->client()->messenger()->scheduler().Schedule(
          [this, weak_transaction](const Status&) {
              SendHeartbeat(TransactionStatus::PENDING, metadata_.transaction_id, weak_transaction);
          },
          std::chrono::microseconds(FLAGS_transaction_heartbeat_usec));
      } else {
        ...
      }
```

### 查找到操作相应的tablet之后的回调 - Batcher::TabletLookupFinished
```
# 当成功查找到Tablet信息之后，则调用TabletLookupFinished处理当前
# 完成的的InFlightOp
- TabletLookupFinished(std::move(in_flight_op), yb_op->tablet())
    - 减少正在查找tablet的in_flight_op的数目(--outstanding_lookups_)
    - 将当前的InFlightOp的状态从InFlightOpState::kLookingUpTablet转换
      为InFlightOpState::kBufferedToTabletServer
    - 将当前的InFlightOp添加到yb::client::internal::Batcher::ops_queue_中，
      ops_queue_是一个关于InFlightOp的数组，ops_queue_中存放所有即将flush的操作
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


### 什么情况下请求会被添加到buffered_txn_write_ops_和buffered_write_ops_中
```
Datum
ybcinhandler(PG_FUNCTION_ARGS)
{
    # IndexAmRoutine中定义了所有index access methods
	IndexAmRoutine *amroutine = makeNode(IndexAmRoutine);
	
	...
	
	# 构建索引的时候使用
	amroutine->ambuild = ybcinbuild;
}

IndexBuildResult *
ybcinbuild(Relation heap, Relation index, struct IndexInfo *indexInfo)
{
	YBCBuildState	buildstate;
	double			heap_tuples = 0;

	PG_TRY();
	{
		# 如果当前是BootstrapProcessing模式，则所有的write operations都将被buffered
		if (IsBootstrapProcessingMode())
			YBCStartBufferingWriteOperations();

        /* Do the heap scan */
		buildstate.isprimary = index->rd_index->indisprimary;
		buildstate.index_tuples = 0;
		heap_tuples = IndexBuildHeapScan(heap, index, indexInfo, true, ybcinbuildCallback,
										 &buildstate, NULL);
	}
	PG_CATCH();
	{
	    # 如果当前是BootstrapProcessing模式，则统一flush所有buffered write operations
		if (IsBootstrapProcessingMode())
			YBCFlushBufferedWriteOperations();
		PG_RE_THROW();
	}
	PG_END_TRY();

	if (IsBootstrapProcessingMode())
		YBCFlushBufferedWriteOperations();

	...
}

YBCStartBufferingWriteOperations
    - YBCPgStartBufferingWriteOperations
        - PgApiImpl::StartBufferingWriteOperations
            - PgSession::StartBufferingWriteOperations
                # buffer_write_ops_不为0，则表示write operations将被buffered
                - buffer_write_ops_++
        
YBCFlushBufferedWriteOperations
    - YBCPgFlushBufferedWriteOperations
        - PgApiImpl::FlushBufferedWriteOperations
            - PgSession::FlushBufferedWriteOperations
                # flush non-transaction buffered write operations
                - FlushBufferedWriteOperations(&buffered_write_ops_, false /* transactional */)
                # flush transaction buffered write operations
                - FlushBufferedWriteOperations(&buffered_txn_write_ops_, true /* transactional */)
```
从上面可以看出，在构建索引的过程中，如果处于PostgreSQL处于BootstrapProcessing模式，则所有的write operations都将被buffered。那么PostgreSQL何时处于BootstrapProcessing模式呢？

```
# 用于启动辅助进程(由AuxProcType决定是哪种进程)
StartChildProcess(AuxProcType type)
    # 所有的辅助进程(bgwriter, walwriter, walreceiver, bootstrapper和shared memory checker等的主要入口)
    - AuxiliaryProcessMain
        - SetProcessingMode(BootstrapProcessing)
```
从上面代码可以看出，只有在PostgreSQL bootstrap过程中，才会设置它为BootstrapProcessing模式。

### 查找status tablet - YBTransaction::Prepare
```
YBTransaction::Prepare(const internal::InFlightOps& ops,
                            ForceConsistentRead force_consistent_read,
                            CoarseTimePoint deadline,
                            Waiter waiter,
                            TransactionMetadata* metadata)
    - YBTransaction::Impl::Prepare
        # 如果当前事务不处于ready状态，则将参数waiter添加到等待队列，并查找status tablet，然后返回
        - if (!ready_) {
            if (waiter) {
              waiters_.push_back(std::move(waiter));
            }
            
            lock.unlock();
            RequestStatusTablet(deadline);
            return false;
          }
        - 否则，当前事务处于ready状态，则：
            - 检查所有待发送的操作，他们涉及到多少个tablet @num_tablets
            # 如果跨越多个tablets，或者force_consistent_read被设置为true，
            # 且没有设置有效的read time，且当前是snapshot isolation，则设置
            # read time
            - SetReadTimeIfNeeded(num_tablets > 1 || force_consistent_read)
            - 修改running_requests计数，增加所有待发送的操作的数目
```
