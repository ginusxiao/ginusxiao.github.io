# 提纲
[toc]


# tablet复制过程源码分析
假设一个tablet group中有3个tablet peer，分布在3个tablet server上，突然其中一个tablet server出现故障，该tablet server上的tablet peer就无法访问了，YugaByte的Master中的ClusterLoadBalancer模块会处理这种情况，ClusterLoadBalancer会挑选另一个tablet server来重新启动一个tablet peer（bootstrap过程）。在新的tablet server上启动tablet peer的过程中，该新的tablet peer起初是处于RaftPeerPB::PRE_VOTER状态，然后该tablet peer会从tablet leader（因为tablet group中剩余的2个tablet peer还活着，仍然可以选举出leader，而这个新的tablet peer没有选举权）中拷贝rocksdb数据库文件和wal log文件到本地进行replay，当replay完毕之后，tablet leader会更改它的状态为RaftPeerPB::VOTER，至此该新的tablet获得选举权。具体分析，请看下文。

## ClusterLoadBalancer::RunLoadBalancer
```
void ClusterLoadBalancer::RunLoadBalancer
    - bool ClusterLoadBalancer::HandleAddReplicas
        - ClusterLoadBalancer::AddReplica(const TabletId& tablet_id, const TabletServerId& to_ts)
            - SendReplicaChanges(GetTabletMap().at(tablet_id), to_ts, true /* is_add */, true /* should_remove_leader */)
                # 在tablet上加读锁（出了这个作用域锁就自动释放了，但是在下面的调用栈中一直持有该锁）
                - auto l = tablet->LockForRead()
                # Start a task to change the config to add an additional voter because the specified tablet is under-replicated.
                # GetDefaultMemberType()返回RaftPeerPB::PRE_VOTER，也就是说新添加的tablet peer的初始状态为RaftPeerPB::PRE_VOTER
                - catalog_manager_->SendAddServerRequest(tablet, GetDefaultMemberType(), l->data().pb.committed_consensus_state(), ts_uuid)
                    # 构建AsyncAddServerTask这个task对象，AsyncAddServerTask的继承关系为AsyncAddServerTask => AsyncChangeConfigTask =>
                    # CommonInfoForRaftTask => RetryingTSRpcTask => MonitoredTask
                    - auto task = std::make_shared<AsyncAddServerTask>(master_, worker_pool_.get(), tablet, member_type, cstate, change_config_ts_uuid)
                    # 将该task添加到tablet对应的table中的pending task集合中
                    - tablet->table()->AddTask(task)
                    # RetryingTSRpcTask::Run()
                    - Status status = task->Run()
                        - ResetTSProxy()
                            # replica_picker类型为PickLeaderReplica，所以调用的是PickLeaderReplica::PickReplica
                            # PickLeaderReplica::PickReplica用于获取tablet leader所在的tablet server并存放在
                            # @target_ts_desc_中
                            - replica_picker_->PickReplica(&target_ts_desc_)
                            # 获取@target_ts_desc_对应的@ts_proxy, @ts_admib_proxy, @consensus_proxy
                            - target_ts_desc_->GetProxy(&master_->proxy_cache(), &ts_proxy)
                            - target_ts_desc_->GetProxy(&master_->proxy_cache(), &ts_admin_proxy)
                            - target_ts_desc_->GetProxy(&master_->proxy_cache(), &consensus_proxy)
                            - ts_proxy_.swap(ts_proxy)
                            - ts_admin_proxy_.swap(ts_admin_proxy)
                            - consensus_proxy_.swap(consensus_proxy)
                        # Task状态切换
                        - PerformStateTransition(MonitoredTaskState::kWaiting, MonitoredTaskState::kRunning)
                        # 虚函数，在AsyncChangeConfigTask中重载了该方法
                        - SendRequest(++attempt_)
                            # 在AsyncAddServerTask中重载了该方法，准备consensus::ChangeConfigRequestPB
                            - PrepareRequest(attempt)
                                # 目标被设置为@target_ts_desc_
                                - req_.set_dest_uuid(permanent_uuid())
                                - req_.set_tablet_id(tablet_->tablet_id())
                                - req_.set_type(consensus::ADD_SERVER)
                                - req_.set_cas_config_opid_index(cstate_.config().opid_index())
                                # 设置需要添加tablet replica的tablet server为@replacement_replica
                                - RaftPeerPB* peer = req_.mutable_server()
                                - peer->set_permanent_uuid(replacement_replica->permanent_uuid())
                                # 设置新添加的peer的角色，根据ClusterLoadBalancer::HandleAddReplicas -> 
                                # CatalogManager::SendAddServerRequest可知，@member_type_是RaftPeerPB::PRE_VOTER
                                - peer->set_member_type(member_type_);
                                # 获取@peer的register info
                                - replacement_replica->GetRegistration(&peer_reg)
                                - *peer->mutable_last_known_addr() = peer_reg.common().rpc_addresses(0)
                            # ConsensusServiceProxy的定义需要在build yugabyte之后方可看到，见build/latest/src/yb/consensus/consensus.proxy.h
                            # 最终会发送给远端的ConsensusService，ConsensusService调用ConsensusService::ChangeConfig
                            - consensus_proxy_->ChangeConfigAsync(req_, &resp_, &rpc_, BindRpcCallback())
                                - ConsensusServiceImpl::ChangeConfig
                                    - CheckUuidMatchOrRespond(tablet_manager_, "ChangeConfig", req, resp, &context)
                                    # 这里的tablet_manager_是TSTabletManager类型，该方法确保对应的tablet存在且为running状态
                                    - LookupTabletPeerOrRespond(tablet_manager_, req->tablet_id(), resp, &context, &tablet_peer)
                                        - tablet_manager->GetTabletPeer(tablet_id, peer)
                                            - LookupTablet(tablet_id, tablet_peer)
                                                - LookupTabletUnlocked(tablet_id, tablet_peer)
                                                    # 如果存在则返回，否则返回null
                                                    - FindOrNull(tablet_map_, tablet_id)
                                    - GetConsensusOrRespond(tablet_peer, resp, &context, &consensus)
                                        # 类型为consensus::RaftConsensus
                                        - *consensus = tablet_peer->shared_consensus()
                                    # 参考“RaftConsensus::ChangeConfig”，这里的req->type()是consensus::ADD_SERVER
                                    - consensus->ChangeConfig(*req, BindHandleResponse(resp, context_ptr), &error_code)
``` 

## RaftConsensus::ChangeConfig
```
RaftConsensus::ChangeConfig
    # 这里的@req就是ClusterLoadBalancer::HandleAddReplicas -> ... -> ConsensusServiceProxy::ChangeConfigAsync中发送的request
    - const RaftPeerPB& server = req.server();
    # @lock是一个mutex锁，它的生命周期直到ReplicateConfigChangeUnlocked执行完毕后结束
    - ReplicaState::UniqueLock lock
    - state_->LockForConfigChange(&lock)
    # 确保当前的peer是active leader，方可执行后续步骤
    - state_->CheckActiveLeaderUnlocked(LeaderLeaseCheckMode::DONT_NEED_LEASE)
    # 获取当前的config @committed_config，并将@new_config设置为@committed_config，
    # 后面会将新的tablet replica添加到@new_config中
    - const RaftConfigPB& committed_config = state_->GetCommittedConfigUnlocked()
    - RaftConfigPB new_config = committed_config
    - IsLeaderReadyForChangeConfigUnlocked(type, server_uuid)
    # 如果是ADD_SERVER：
        # 检查待添加的tablet replica是否在当前的config中，如果在就退出
        - IsRaftConfigMember(server_uuid, committed_config)
        # 确保待change role的peer当前的role为PRE_OBSERVER或者PRE_VOTER
        # 根据ClusterLoadBalancer::HandleAddReplicas -> ... -> ConsensusServiceProxy::ChangeConfigAsync
        # -> rpc -> RaftConsensus::ChangeConfig逻辑可知，这里的server.member_type()就是RaftPeerPB::PRE_VOTER
        if (server.member_type() != RaftPeerPB::PRE_VOTER &&
            server.member_type() != RaftPeerPB::PRE_OBSERVER) {
          return STATUS(IllegalState, ...);
        }
        # 将待添加的tablet sever加入到@new_config中，后续会在RaftConsensus::ReplicateConfigChangeUnlocked
        # 中将@new_config设置为active config
        - new_peer = new_config.add_peers()
        - *new_peer = server
    # 如果是CHANGE_ROLE：
        # 获取待change role的peer，存放在@new_peer中
        - GetMutableRaftConfigMember(&new_config, server_uuid, &new_peer)
        # 确保待change role的peer当前的role为PRE_OBSERVER或者PRE_VOTER
        if (new_peer->member_type() != RaftPeerPB::PRE_OBSERVER &&
            new_peer->member_type() != RaftPeerPB::PRE_VOTER) {
          return STATUS(IllegalState, ...);
        }
        # 根据@new_peer之前的角色为PRE_OBSERVER或者PRE_VOTER，设置change role之后的角色为OBSERVER或者VOTER
        if (new_peer->member_type() == RaftPeerPB::PRE_OBSERVER) {
          new_peer->set_member_type(RaftPeerPB::OBSERVER);
        } else {
          new_peer->set_member_type(RaftPeerPB::VOTER);
        }
    - 设置ReplicateMsg和其中的ChangeConfigRecordPB
        auto cc_replicate = std::make_shared<ReplicateMsg>();
        cc_replicate->set_op_type(CHANGE_CONFIG_OP);
        ChangeConfigRecordPB* cc_req = cc_replicate->mutable_change_config_record();
        cc_req->set_tablet_id(tablet_id());
        *cc_req->mutable_old_config() = committed_config;
        *cc_req->mutable_new_config() = new_config;
        cc_replicate->mutable_committed_op_id()->CopyFrom(state_->GetCommittedOpIdUnlocked());
    - auto context = std::make_shared<StateChangeContext>(StateChangeReason::LEADER_CONFIG_CHANGE_COMPLETE,
       *cc_req, (type == REMOVE_SERVER) ? server_uuid : "")
    # 将新的配置@new_config设置为pending active config，并根据配置更新PeerManager中管理的peers集合
    # 同时生成ReplicateMsg(CHANGE_CONFIG)消息并最终将该消息写入到log中
    - ReplicateConfigChangeUnlocked(cc_replicate, new_config, type,
            std::bind(&RaftConsensus::MarkDirtyOnSuccess,
            this,
            std::move(context),
            std::move(client_cb), std::placeholders::_1))
        - scoped_refptr<ConsensusRound> round(new ConsensusRound(this, replicate_ref))
        # Sets the given configuration as pending commit
        # 这个pending config会作为当前的active config
        - state_->SetPendingConfigUnlocked(new_config)
            # 确保当前没有处于pending状态的config
            - CHECK(!cmeta_->has_pending_config()))
            # 将@new_config设置为pending config
            - cmeta_->set_pending_config(new_config)
        # Update the peers and queue to be consistent with a new active configuration
        - RefreshConsensusQueueAndPeersUnlocked
            # 断开与那些不在@active_config中的peer之间的连接
            - peer_manager_->ClosePeersNotInConfig(active_config)
            - queue_->SetLeaderMode(state_->GetCommittedOpIdUnlocked(), state_->GetCurrentTermUnlocked(), active_config)
                - queue_state_.current_term = current_term;
                  queue_state_.committed_index = committed_index;
                  queue_state_.majority_replicated_opid = committed_index;
                  queue_state_.active_config.reset(new RaftConfigPB(active_config));
                  queue_state_.mode = Mode::LEADER;
                - CheckPeersInActiveConfigIfLeaderUnlocked
                    # 如果不是LEADER mode，则直接返回
                    - if (queue_state_.mode != Mode::LEADER) return;
                    # 检查queue_state_.active_config中的peers是否都存在于peers_map_中
                      unordered_set<string> config_peer_uuids;
                      for (const RaftPeerPB& peer_pb : queue_state_.active_config->peers()) {
                          InsertOrDie(&config_peer_uuids, peer_pb.permanent_uuid());
                      }
                      for (const PeersMap::value_type& entry : peers_map_) {
                        if (!ContainsKey(config_peer_uuids, entry.first)) {
                          LOG_WITH_PREFIX_UNLOCKED(FATAL) << Substitute("Peer $0 is not in the active config.");
                        }
                      }
            # Updates 'peers_' according to the new configuration config
            - peer_manager_->UpdateRaftConfig(active_config)
                # 查找@active_config中的所有的peer是否存在，如果不存在则创建之，并将该新创建的peer添加到peer manager中
                # NewRemotePeer中的参数@consensus_类型为RaftConsensus，参考RaftConsensus的构造函数
                - auto remote_peer = Peer::NewRemotePeer(
                    peer_pb, tablet_id_, local_uuid_, queue_, raft_pool_token_,
                    peer_proxy_factory_->NewProxy(peer_pb), consensus_)
                - peers_[peer_pb.permanent_uuid()] = std::move(*remote_peer)
        # 生成ReplicateMsg(CHANGE_CONFIG)消息并将该消息写入到log中
        - scoped_refptr<ConsensusRound> round(new ConsensusRound(this, replicate_ref))
          AppendNewRoundToQueueUnlocked(round)
            - AppendNewRoundsToQueueUnlocked({ round })
                - 实际上在AppendNewRoundsToQueueUnlocked中的参数是round集合，它针对集合中的每个round执行：
                    state_->NewIdUnlocked(round->replicate_msg()->mutable_id())
                    ReplicateMsg* const replicate_msg = round->replicate_msg().get()
                    # record the last committed op id
                    replicate_msg->mutable_committed_op_id()->CopyFrom(state_->GetCommittedOpIdUnlocked())
                    # 在ReplicaState中添加它为pending operation，因为尚未被committed
                    Status s = state_->AddPendingOperation(round)
                        - InsertOrDie(&pending_operations_, round->replicate_msg()->id().index(), round)
                    replicate_msgs.reserve(rounds.size())
                # 将集合中所有的round对应的replicate message都添加到@replicate_msgs数组中
                - replicate_msgs.reserve(rounds.size())
                - replicate_msgs.push_back(round->replicate_msg())
                # Appends a vector of messages to be replicated to the peers.
                # @queue_类型为PeerMessageQueue，其中存放所有必须被发送给各peers的消息
                - queue_->AppendOperations(replicate_msgs, Bind(DoNothingStatusCB))
                    # Append the operations into the log and the cache
                    - log_cache_.AppendOperations(msgs,
                        Bind(&PeerMessageQueue::LocalPeerAppendFinished,
                             Unretained(this),
                             last_id,
                             log_append_callback))
                        # Append the given set of replicate messages, asynchronously
                        - log_->AsyncAppendReplicates(
                            msgs, Bind(&LogCache::LogCallback,
                                       Unretained(this),
                                       last_idx_in_batch,
                                       borrowed_memory,
                                       callback))
                            - CreateBatchFromAllocatedOperations(msgs, &batch)
                            - Reserve(REPLICATE, &batch, &reserved_entry_batch)
                            - reserved_entry_batch->SetReplicates(msgs)
                            - AsyncAppend(reserved_entry_batch, callback)
                                - appender_->Submit(entry_batch)
                                    - task_stream_->Submit(item)
                                        - TaskStreamImpl<T>::Submit
                                            # 入队的task将在TaskStreamImpl::Run中被处理，参考“TaskStreamImpl::Run”
                                            - queue_.BlockingPut(task)
    - peer_manager_->SignalRequest(RequestTriggerMode::kNonEmptyOnly)
        # 针对@peer_manager中的所有peers_执行SignalRequest(Signals that this peer has a new request to replicate/store)
        - iter->second->SignalRequest(trigger_mode)
            # ThreadPoolToken::SubmitClosure，会生成task，并添加到线程池中等待调度处理，最终执行Peer::SendNextRequest
            - raft_pool_token_->SubmitClosure(Bind(&Peer::SendNextRequest, Unretained(this), trigger_mode))
                # 当该任务被调度执行时，执行Peer::SendNextRequest，请参考“Peer::SendNextRequest”
                # Peer::SignalRequest -> Peer::SendNextRequest这个过程是异步的
                - Submit(std::make_shared<FunctionRunnable>((std::bind(&Closure::Run, c))))
                    # Submits a task to be run via token
                    - pool_->DoSubmit(std::move(r), this)
```

## TaskStreamImpl<T>::Run
```
TaskStreamImpl<T>::Run()
    - queue_.BlockingDrainTo(&group, wait_timeout_deadline)
    # 遍历group中的所有item，依次调用ProcessItem
    - ProcessItem(item)
        # process_item是在TaskStreamImpl构造函数中指定的，在当前分析的代码上下文中：Log::Appender::Appender -> new 
        # TaskStream<LogEntryBatch>(std::bind(&Log::Appender::ProcessBatch, this, _1), append_thread_pool)，
        # 所以使用的就是Log::Appender::ProcessBatch
        - process_item_(item)
    # 处理完所有的item之后，执行下面的语句，该语句的作用是进行sync工作，见Log::Appender::ProcessBatch的实现
    - ProcessItem(nullptr);
```

## Log::Appender::ProcessBatch
```
Log::Appender::ProcessBatch(LogEntryBatch* entry_batch)
    # 如果@entry_batch是空，就执行sync操作，然后返回
    - GroupWork()
        - log_->Sync()
            # Makes sure the I/O buffers in the underlying writable file are flushed
            - active_segment_->Sync()
        # 针对sync_batch_集合中的每个LogEntryBatch，调用其callback
        # 在RaftConsensus::ChangeConfig -> RaftConsensus::ReplicateConfigChangeUnlocked ->
        # RaftConsensus::AppendNewRoundToQueueUnlocked -> Log::AsyncAppendReplicates上下文中对应的
        # callback是LogCache::LogCallback，在LogCache::LogCallback中会进一步调用user_callback.Run，
        # 在RaftConsensus::ChangeConfig -> RaftConsensus::ReplicateConfigChangeUnlocked ->
        # RaftConsensus::AppendNewRoundToQueueUnlocked -> PeerMessageQueue::AppendOperations 上下文
        # 中对应的user_callback是DoNothingStatusCB
        - entry_batch->callback().Run(Status::OK())
        # 清理sync_batch_
        - sync_batch_.clear()
    # 否则处理当前的@entry_batch，将LogEntryBatch中的数据写入log中
    - log_->DoAppend(entry_batch)
        - Slice entry_batch_data = entry_batch->data()
        - active_segment_->WriteEntryBatch(entry_batch_data)
            - writable_file_->Append(Slice(header_buf, sizeof(header_buf)))
            - writable_file_->Append(data)
    - sync_batch_.emplace_back(entry_batch)
```

## Peer::SendNextRequest
```
Peer::SendNextRequest
    # 组装发送给peer的请求@request_
    # Peer中的queue_类型为PeerMessageQueue，Peer中的queue_是来自于RaftConsensus，见下面的过程：
    # RaftConsensus::Create {queue = new PeerMessageQueue(); new PeerManager(, queue.get(), );}
    # PeerManager::UpdateRaftConfig {remote_peer = Peer::NewRemotePeer(, queue_, );}
    # 而在RaftConsensus::ChangeConfig -> AppendNewRoundToQueueUnlocked中已经将待发送给各peer的
    # 消息存放在PeerMessageQueue中，各Peer可以直接从其中获取待发送的消息，消息其实是存放在
    # PeerMessageQueue::log_cache_中的
    # 请参考PeerMessageQueue::RequestForPeer
    - queue_->RequestForPeer(peer_pb_.permanent_uuid(), &request_,
        &replicate_msg_refs_, &needs_remote_bootstrap, &member_type, &last_exchange_successful)
        # 如果peer需要remote bootstrap，则设置needs_remote_bootstrap为true并返回，否则设置为false
        - if (PREDICT_FALSE(peer->needs_remote_bootstrap)) {
            *needs_remote_bootstrap = true;
            return Status::OK();
          }
          *needs_remote_bootstrap = false;
        # 从log_cache_中读取消息
        - ReplicateMsgs messages
        - Status s = log_cache_.ReadOps(peer->next_index - 1,
                                  max_batch_size,
                                  &messages,
                                  &preceding_id)
        # 将所有的消息添加到request中，request由RequestForPeer参数传递过来
        - for (const auto& msg : messages) {
            request->mutable_ops()->AddAllocated(msg.get());
          }
        # 返回messages给调用者
        - msg_refs->swap(messages);
    # 如果peer需要bootstrap（请参考“什么情况下需要bootstrap”）
        - SendRemoteBootstrapRequest
            - queue_->GetRemoteBootstrapRequestForPeer(peer_pb_.permanent_uuid(), &rb_request_)
            # 调用RpcPeerProxy::StartRemoteBootstrap -> consensus_proxy_->StartRemoteBootstrapAsync
            # 最终在远端peer中的处理是ConsensusServiceImpl::StartRemoteBootstrap
            # 请参考“ConsensusServiceImpl::StartRemoteBootstrap”
            - proxy_->StartRemoteBootstrap(
                  &rb_request_, &rb_response_, &controller_,
                  std::bind(&Peer::ProcessRemoteBootstrapResponse, this))
        - return;
    # 如果peer不需要bootstrap，但是它是PRE_VOTER或者PRE_OBSERVER，则需要将之转换为VOTER或者OBSERVER
        - consensus::ChangeConfigRequestPB req
        - consensus::ChangeConfigResponsePB resp
        - 设置@req
              req.set_tablet_id(tablet_id_);
              req.set_type(consensus::CHANGE_ROLE);
              RaftPeerPB *peer = req.mutable_server();
              peer->set_permanent_uuid(peer_pb_.permanent_uuid());
        # 这里的consensus_类型为RaftConsensus
        - consensus_->ChangeConfig(req, &DoNothingStatusCB, &error_code)
            # 前面已经分析了RaftConsensus::ChangeConfig，但是此时的req.type()是consensus::CHANGE_ROLE
            - RaftConsensus::ChangeConfig
                - ReplicateConfigChangeUnlocked
                # SignalRequest在前面已经分析过，它最终会调用Peer::SendNextRequest
                - peer_manager_->SignalRequest(RequestTriggerMode::kNonEmptyOnly)
        - return;
    # 否则
        - request_.set_tablet_id(tablet_id_);
        - request_.set_caller_uuid(leader_uuid_);
        - request_.set_dest_uuid(peer_pb_.permanent_uuid());
        # Sends a request, asynchronously, to a remote peer，实际调用关系是：
        # RpcPeerProxy::UpdateAsync -> consensus_proxy_->UpdateConsensusAsync
        - proxy_->UpdateAsync(&request_, trigger_mode, &response_, &controller_,
            std::bind(&Peer::ProcessResponse, this))
            # 发送给远端的peer，远端peer的处理方法是ConsensusServiceImpl::UpdateConsensus
            # 请参考“ConsensusServiceImpl::UpdateConsensus”
            - consensus_proxy_->UpdateConsensusAsync(*request, response, controller, callback)
            
            ......
            
            # 根据Peer::SendNextRequest -> proxy_->UpdateAsync(..., std::bind(&Peer::ProcessResponse, this))可知，
            # 当接收到远端peer的响应之后，处理函数为Peer::ProcessResponse，请参考“Peer::ProcessResponse”
```

## PeerMessageQueue::RequestForPeer
```
PeerMessageQueue::RequestForPeer
    # 根据给定的uuid从peers_map_中找到对应的peer
    - peer = FindPtrOrNull(peers_map_, uuid)
    # 设置request 时间戳和committed index
    - preceding_id = queue_state_.last_appended
    - request->set_propagated_hybrid_time(now_ht.ToUint64())
      request->mutable_committed_index()->CopyFrom(queue_state_.committed_index)
    # 设置@member_type，会返回给上层调用  
    - *member_type = peer->member_type;
    # 从@log_cache_中读取消息填充@request，从peer->next_index开始读取
    - if (!peer->is_new) {
        ReplicateMsgs messages;
        int max_batch_size = FLAGS_consensus_max_batch_size_bytes - request->ByteSize();
        Status s = log_cache_.ReadOps(peer->next_index - 1,
                                  max_batch_size,
                                  &messages,
                                  &preceding_id)
        for (const auto& msg : messages) {
          request->mutable_ops()->AddAllocated(msg.get());
        }
      }
    # 设置request proceeding index
    - request->mutable_preceding_id()->CopyFrom(preceding_id)
```


## 什么情况下需要bootstrap
```
Peer::SendNextRequest
    - Peer::ProcessResponse
        - Peer::DoProcessResponse
            - queue_->ResponseFromPeer(peer_pb_.permanent_uuid(), response_, &more_pending)
                # 如果response中错误码指示出错（包括tablet没找到或者tablet被删除了），则设置needs_remote_bootstrap
                - peer->needs_remote_bootstrap = true;
```

## ConsensusServiceImpl::StartRemoteBootstrap
```
ConsensusServiceImpl::StartRemoteBootstrap
    - CheckUuidMatchOrRespond(tablet_manager_, "StartRemoteBootstrap", req, resp, &context)
        # TSTabletManager::StartRemoteBootstrap
        - tablet_manager_->StartRemoteBootstrap(*req)
            # 当前只考虑需要bootstrap的tablet不在当前tablet server上的情况
            # RemoteBootstrapClient：Client class for using remote bootstrap to copy a tablet from another host
            - gscoped_ptr<YB_EDITION_NS_PREFIX RemoteBootstrapClient> rb_client(
                  new YB_EDITION_NS_PREFIX RemoteBootstrapClient(
                      tablet_id, fs_manager_, fs_manager_->uuid()))
            # Start up a remote bootstrap session to bootstrap from the specified bootstrap peer
            # 
            - rb_client->Start(bootstrap_peer_uuid,
                                 &server_->proxy_cache(),
                                 bootstrap_peer_addr,
                                 &meta,
                                 this)
                # 准备RemoteBootstrapServiceProxy这个RPC proxy
                - proxy_.reset(new RemoteBootstrapServiceProxy(proxy_cache, bootstrap_peer_addr))
                # 准备remote bootstrap session请求
                  BeginRemoteBootstrapSessionRequestPB req;
                  req.set_requestor_uuid(permanent_uuid_);
                  req.set_tablet_id(tablet_id_);
                # 启动remote bootstrap session
                # 调用RemoteBootstrapServiceProxy::BeginRemoteBootstrapSession，该方法会同步等待返回结果
                # 在remote peer端的处理方法是RemoteBootstrapServiceImpl::BeginRemoteBootstrapSession
                # 请参考“RemoteBootstrapServiceImpl::BeginRemoteBootstrapSession”
                - proxy_->BeginRemoteBootstrapSession(req, &resp, &controller)
                # 根据response设置session相关信息
                - session_id_ = resp.session_id()
                  session_idle_timeout_millis_ = resp.session_idle_timeout_millis()
                # 根据response设置superblock相关信息
                  superblock_.reset(resp.release_superblock())
                  superblock_->clear_rocksdb_dir()
                  superblock_->clear_wal_dir()
                  superblock_->set_tablet_data_state(tablet::TABLET_DATA_COPYING)
                # 根据response设置WAL log segments相关信息
                  wal_seqnos_.assign(resp.wal_segment_seqnos().begin(), resp.wal_segment_seqnos().end())
                # 根据response设置consensus state相关信息
                  remote_committed_cstate_.reset(resp.release_initial_committed_cstate())
                # 设置schema相关信息
                - SchemaFromPB(superblock_->schema(), &schema)
                - if (replace_tombstoned_tablet_) {
                    ...
                  } else {
                    # 设置partition相关信息
                    Partition::FromPB(superblock_->partition(), &partition)
                    # 设置PartitionSchema相关信息，其中PartitionSchema用于确定如何进行partition
                    PartitionSchema::FromPB(superblock_->partition_schema(), schema, &partition_schema)
                    # 更新data/wal directory assignment map，具体实现为查找具有最少tablet的data/wal目录，
                    # 然后将这个新的tablet存放到该data/wal目录中
                    ts_manager->GetAndRegisterDataAndWalDir(fs_manager_,
                                              table_id,
                                              tablet_id_,
                                              superblock_->table_type(),
                                              &data_root_dir,
                                              &wal_root_dir)
                    # Create metadata for a new tablet，其中meta_是本地的关于该tablet的元数据文件
                    TabletMetadata::CreateNew(fs_manager_,
                                             table_id,
                                             tablet_id_,
                                             superblock_->table_name(),
                                             superblock_->table_type(),
                                             schema,
                                             partition_schema,
                                             partition,
                                             tablet::TABLET_DATA_COPYING,
                                             &meta_,
                                             data_root_dir,
                                             wal_root_dir)    
                    superblock_->set_rocksdb_dir(meta_->rocksdb_dir())
                    superblock_->set_wal_dir(meta_->wal_dir())
                  }
            # 暂不考虑replacing_tablet的情况
            - RegisterTabletPeerMode mode = replacing_tablet ? REPLACEMENT_PEER : NEW_PEER;
            - TabletPeerPtr tablet_peer = CreateAndRegisterTabletPeer(meta, mode)
                # 创建TabletPeer，在TabletPeer的构造函数中指定了一个回调函数用于向TabletManager汇报tablet状态的变化
                - TabletPeerPtr tablet_peer(
                      new TabletPeerClass(meta,
                                          local_peer_pb_,
                                          apply_pool_.get(),
                                          Bind(&TSTabletManager::ApplyChange,
                                               Unretained(this),
                                               meta->tablet_id())))
                # 将新创建的tablet添加到tablet map中
                - RegisterTablet(meta->tablet_id(), tablet_peer, mode)
            # 从远端的tablet peer中下载所有相关的文件，见“RemoteBootstrapClient::FetchAll”
            - rb_client->FetchAll(tablet_peer->status_listener())
            # 请参考“RemoteBootstrapClient::Finish”
            # 在该函数返回之后，tablet的角色将从PRE_OBSERVER/PRE_VOTER转换为OBSERVER/VOTER
            - rb_client->Finish()
            - OpenTablet(meta, nullptr)
                # bootstrap tablet
                - TabletPeerPtr tablet_peer
                # 设置该tablet为BOOTSTRAPPING状态
                  shared_ptr<TabletClass> tablet
                  consensus::ConsensusBootstrapInfo bootstrap_info
                  tablet_peer->SetBootstrapping()
                  tablet::BootstrapTabletData data = {
                    meta,
                    scoped_refptr<server::Clock>(server_->clock()),
                    server_->mem_tracker(),
                    metric_registry_,
                    tablet_peer->status_listener(),
                    tablet_peer->log_anchor_registry(),
                    tablet_options_,
                    tablet_peer.get(),
                    std::bind(&TSTabletManager::PreserveLocalLeadersOnly, this, _1),
                    tablet_peer.get(),
                    append_pool()};
                # 注意这里的返回信息@bootstrap_info将在后续的TabletPeer::Start中被使用
                - BootstrapTablet(data, &tablet, &log, &bootstrap_info)
                      # Bootstraps an existing tablet by opening the metadata from disk, and rebuilding
                      # soft state by playing log segments
                      - TabletBootstrap bootstrap(data)
                      # Plays the log segments, rebuilding the portion of the Tablet's soft state that is present in the log
                      # 调用TabletBootstrap::Bootstrap，请参考“TabletBootstrap::Bootstrap”
                      - bootstrap.Bootstrap(rebuilt_tablet, rebuilt_log, consensus_info)
                      # 在bootstrap过程中，为了加速bootstrap过程，关闭了sync，所以这里需要再次打开
                      - (*rebuilt_log)->ReEnableSyncIfRequired()
                # 初始化tablet peer，主要是初始化log和consensus，请参考“TabletPeer::InitTabletPeer”
                # 这里用到的参数@log就是在BootstrapTablet -> TabletBootstrap::Bootstrap -> FinishBootstrap
                # 中使用的log_
                - tablet_peer->InitTabletPeer(tablet,
                    async_client_init_.get_client_future(),
                    scoped_refptr<server::Clock>(server_->clock()),
                    server_->messenger(),
                    &server_->proxy_cache(),
                    log,
                    tablet->GetMetricEntity(),
                    raft_pool(),
                    append_pool())
                # 启动tablet peer，请参考“TabletPeer::Start”
                - tablet_peer->Start(bootstrap_info)  
            # Verify that the remote bootstrap was completed successfully by verifying that the ChangeConfig
            # request was propagated，即检查@tablet_peer是否已经变为RaftPeerPB::VOTER或者RaftPeerPB::OBSERVER状态
            - rb_client->VerifyChangeRoleSucceeded(tablet_peer->shared_consensus())
```

## RemoteBootstrapServiceImpl::BeginRemoteBootstrapSession
```
RemoteBootstrapServiceImpl::BeginRemoteBootstrapSession
    # 查找对应的tablet是否存在，tablet_peer_lookup_类型实际为TSTabletManager
    - tablet_peer_lookup_->GetTabletPeer(tablet_id, &tablet_peer),
        RemoteBootstrapErrorPB::TABLET_NOT_FOUND,
        Substitute("Unable to find specified tablet: $0", tablet_id))
    # 在sessions_中查找特定名称的session是否存在，如果不存在，则创建一个新的session，如果存在，则初始化该session
        const string session_id = Substitute("$0-$1-$2", requestor_uuid, tablet_id, now.ToString())
        if (!FindCopy(sessions_, session_id, &session)) {
            session.reset(new RemoteBootstrapSessionClass(tablet_peer, session_id, requestor_uuid, fs_manager_));
            InsertOrDie(&sessions_, session_id, session);
        } else {
            # 见“RemoteBootstrapSession::Init”
            session->Init();
        }
    # 设置session的过期时间
    - ResetSessionExpirationUnlocked(session_id);
    # 设置response，其中包括tablet superblock、consensus state summary信息和WAL log segments信息
    - resp->set_session_id(session_id);
      resp->set_session_idle_timeout_millis(FLAGS_remote_bootstrap_idle_timeout_ms);
      resp->mutable_superblock()->CopyFrom(session->tablet_superblock());
      resp->mutable_initial_committed_cstate()->CopyFrom(session->initial_committed_cstate());
      for (const scoped_refptr<log::ReadableLogSegment>& segment : session->log_segments()) {
        resp->add_wal_segment_seqnos(segment->header().sequence_number());
      }
    # 发送响应
    - context.RespondSuccess()
```

## RemoteBootstrapSession::Init
```
RemoteBootstrapSession::Init
    - UnregisterAnchorIfNeededUnlocked
        # release the anchor on a log index if it is registered
        # LogAnchorRegistry：allows callers to register their interest in (anchor) a particular log index
        - tablet_peer_->log_anchor_registry()->UnregisterIfAnchored(&log_anchor_)
            # 如果anchor尚未注册，则直接返回
            - if (!anchor->is_registered) return Status::OK()
            # 否则unregister之，从anchors_集合中查找给定的anchor，如果存在则删除之
            - UnregisterUnlocked(anchor)
    # 注册当前session的@log_anchor_到log anchor registry中
    # 设置anchor log index为MinimumOpId().index()，即为0
    # 关于LogAnchorRegistry：The primary use case for this is to prevent the deletion
    # of segments of the WAL that reference as-yet unflushed in-memory operations
    # 关于如何通过LogAnchorRegistry去阻止WAL中某些segments被删除的请参考：
    # TabletPeer::RunLogGC
    #   # 获取最小的不能执行GC的log index
    #   - GetEarliestNeededLogIndex(&min_log_index)
    #       - log_anchor_registry_->GetEarliestRegisteredLogIndex(&min_anchor_index)
    #   # @min_log_index决定了最小的在GC中需要被保留的log index
    #   - log_->GC(min_log_index, &num_gced)
    #
    # 综上，这里设置最小的必须被保留的log index为MinimumOpId().index(),即为0的目的就是防止在
    # 执行该方法的过程中执行GC，在该方法的尾部会重新设置在log anchor registry中的最小的log index
    - tablet_peer_->log_anchor_registry()->Register(
      MinimumOpId().index(), anchor_owner_token, &log_anchor_);
    # 从磁盘读取superblock，读取到RemoteBootstrapSession::tablet_superblock_中
    - const scoped_refptr<TabletMetadata>& metadata = tablet_peer_->tablet_metadata()
      metadata->ReadSuperBlockFromDisk(&tablet_superblock_)
        - string path = fs_manager_->GetTabletMetadataPath(tablet_id_)
        - pb_util::ReadPBContainerFromPath(fs_manager_->env(), path, superblock)
    # 从log中获取last-used OpId
    - last_logged_opid = tablet_peer_->log()->GetLatestEntryOpId()
    # 获取对应的tablet，类型为Tablet
    - tablet = tablet_peer_->shared_tablet()
    # 如果checkpoints_dir不存在，则创建之
      auto checkpoints_dir = JoinPathSegments(tablet_superblock_.rocksdb_dir(), "checkpoints")
      metadata->fs_manager()->CreateDirIfMissing(checkpoints_dir)
    # 清除tablet_superblock_中记录的rocksdb files列表
      tablet_superblock_.clear_rocksdb_files();
    # 创建当前bootstrap session的checkpoint directory（放在checkpoints_dir目录下面）
    - auto session_checkpoint_dir = std::to_string(last_logged_opid.index) + "_" + now.ToString();
      checkpoint_dir_ = JoinPathSegments(checkpoints_dir, session_checkpoint_dir);
    # 在checkpoint_dir_目录下创建一个RocksDB checkpoint，调用的是Tablet::CreateCheckpoint
    - tablet->CreateCheckpoint(checkpoint_dir_)
        # 在temp_intents_dir目录创建关于intents_db_的checkpoint，同时在dir目录下创建关于regular_db_的checkpoint
        # 如果前面两个checkpoint都创建成功，则将temp_intents_dir目录下的checkpoint转移到final_intents_dir目录下
        - auto temp_intents_dir = dir + kIntentsDBSuffix;
          auto final_intents_dir = JoinPathSegments(dir, kIntentsSubdir);
          rocksdb::checkpoint::CreateCheckpoint(intents_db_.get(), temp_intents_dir)
          rocksdb::checkpoint::CreateCheckpoint(regular_db_.get(), dir)
          Env::Default()->RenameFile(temp_intents_dir, final_intents_dir)
    # 列举checkpoint_dir_目录下的所有文件，将这些文件集合存放于tablet_superblock_中
    - *tablet_superblock_.mutable_rocksdb_files() = VERIFY_RESULT(ListFiles(checkpoint_dir_))
    # 将snapshot文件添加到tablet_superblock_中
    - InitSnapshotFiles()
        # !!!在当前社区版本中不支持snapshot!!!
        - tablet_superblock_.clear_snapshot_files()
    # Copies a snapshot of the current sequence of segments into 'segments'.
    # tablet_peer_->log()类型为Log，在Yugabyte中Log同时充当WAL和consensus state machine中的持久化存储的角色
    # @log_segments中存放的是那些尚未被GC的segments
    - tablet_peer_->log()->GetLogReader()->GetSegmentsSnapshot(&log_segments_)
    - 针对@log_segments中的所有segment，逐一获取其size并将之添加到segment map中
      for (const scoped_refptr<ReadableLogSegment>& segment : log_segments_) {
        RETURN_NOT_OK(OpenLogSegmentUnlocked(segment->header().sequence_number()));
      }
    - SetInitialCommittedState()
        # 获取tablet对应的consensus，类型为RaftConsensus
        - shared_ptr <consensus::Consensus> consensus = tablet_peer_->shared_consensus()
        - initial_committed_cstate_ = consensus->ConsensusState(consensus::CONSENSUS_CONFIG_COMMITTED)
            - state_->LockForRead(&lock)
            - ConsensusStateUnlocked(type, leader_lease_status)
                # @state_类型为ReplicaState，获取当前的consensus state summary
                - state_->ConsensusStateUnlocked(type)
                    # @cmeta_：Consensus metadata persistence object，类型为ConsensusMetadata
                    - cmeta_->ToConsensusStatePB(type)
    # 重新调整@last_logged_opid.index为log anchor registry中的最小需要被保留的log index
    - tablet_peer_->log_anchor_registry()->UpdateRegistration(
        last_logged_opid.index, anchor_owner_token, &log_anchor_)
        - UnregisterUnlocked(anchor)
        - RegisterUnlocked(log_index, owner, anchor)
```

## RemoteBootstrapClient::FetchAll
```
RemoteBootstrapClient::FetchAll
    - DownloadRocksDBFiles()
        # 设置superblock @new_sb
        - gscoped_ptr<TabletSuperBlockPB> new_sb(new TabletSuperBlockPB())
          new_sb->CopyFrom(*superblock_)
          rocksdb_dir = meta_->rocksdb_dir()
          new_sb->set_rocksdb_dir(rocksdb_dir)
        - CreateTabletDirectories(rocksdb_dir, meta_->fs_manager())
        # 逐一下载rocksdb文件，参考请参考“RemoteBootstrapClient::DownloadFile”
        - DataIdPB data_id;
          data_id.set_type(DataIdPB::ROCKSDB_FILE);
          for (auto const& file_pb : new_sb->rocksdb_files()) {
            RETURN_NOT_OK(DownloadFile(file_pb, rocksdb_dir, &data_id));
          }
        # 将intents文件从@intents_tmp_dir移动到@intents_dir
        - auto intents_tmp_dir = JoinPathSegments(rocksdb_dir, tablet::kIntentsSubdir)
          if (fs_manager_->env()->FileExists(intents_tmp_dir)) {
            auto intents_dir = rocksdb_dir + tablet::kIntentsDBSuffix;
            LOG(INFO) << "Moving intents DB: " << intents_tmp_dir << " => " << intents_dir;
            RETURN_NOT_OK(fs_manager_->env()->RenameFile(intents_tmp_dir, intents_dir));
          }
        - new_superblock_.swap(new_sb);
        - downloaded_rocksdb_files_ = true;
    - DownloadWALs()
        # 如果@wal_dir已经存在，则删除其中的所有文件
        - const string& wal_dir = meta_->wal_dir()
          if (fs_manager_->env()->FileExists(wal_dir)) {
            RETURN_NOT_OK(fs_manager_->env()->DeleteRecursively(wal_dir));
          }
        # @wal_table_top_dir中存放的是@wal_dir的上级目录，在创建@wal_dir之前和之后都会对@wal_table_top_dir执行fsync
        - auto wal_table_top_dir = DirName(wal_dir);
          fs_manager_->CreateDirIfMissing(wal_table_top_dir)
          fs_manager_->env()->SyncDir(DirName(wal_table_top_dir))
          fs_manager_->env()->CreateDir(wal_dir)
          fs_manager_->env()->SyncDir(wal_table_top_dir)
        # 逐一下载WAL log segments，请参考“RemoteBootstrapClient::DownloadWAL(2)”
        - int num_segments = wal_seqnos_.size()
          for (uint64_t seg_seqno : wal_seqnos_) {
            DownloadWAL(seg_seqno)
            ++counter;
          }

          downloaded_wal_ = true
```

## RemoteBootstrapClient::DownloadWAL
```
RemoteBootstrapClient::DownloadWAL(uint64_t wal_segment_seqno)
    - DataIdPB data_id;
      data_id.set_type(DataIdPB::LOG_SEGMENT)
      data_id.set_wal_segment_seqno(wal_segment_seqno)
      const string dest_path = fs_manager_->GetWalSegmentFileName(meta_->wal_dir(), wal_segment_seqno)
    # 下载@wal_segment_seqno所代表的log segment
    - fs_manager_->env()->NewWritableFile(opts, dest_path, &writer)
      DownloadFile(data_id, writer.get())
```

## RemoteBootstrapClient::DownloadFile
```
RemoteBootstrapClient::DownloadFile(
    const tablet::FilePB& file_pb, const std::string& dir, DataIdPB *data_id)
    - gscoped_ptr<WritableFile> file
      fs_manager_->env()->NewWritableFile(opts, file_path, &file)
      data_id->set_file_name(file_pb.name())
      DownloadFile(*data_id, file.get())
```

## RemoteBootstrapClient::DownloadFile(2)
```
template<class Appendable>
Status RemoteBootstrapClient::DownloadFile(const DataIdPB& data_id,
                                           Appendable* appendable)
    # 分片下载整个文件                                           
    - bool done = false;
      while (!done) {     
          controller.Reset()
          # 准备请求
          req.set_session_id(session_id_)
          req.mutable_data_id()->CopyFrom(data_id)
          req.set_offset(offset)
          req.set_max_length(max_length)
          # RemoteBootstrapServiceProxy::FetchData
          # 最终在远端peer中的处理是RemoteBootstrapServiceImpl::FetchData
          # 请参考“ConsensusServiceImpl::FetchData”
          proxy_->FetchData(req, &resp, &controller)
          VerifyData(offset, resp.chunk())
          appendable->Append(resp.chunk().data())
          if (offset + resp.chunk().data().size() == resp.chunk().total_data_length()) {
            done = true;
          }
          offset += resp.chunk().data().size()
      }
```

## RemoteBootstrapServiceImpl::FetchData
```
RemoteBootstrapServiceImpl::FetchData
    # 查找之前创建的session，并重置session过期时间
    - const string& session_id = req->session_id()
      FindSessionUnlocked(session_id, &app_error, &session)
      ResetSessionExpirationUnlocked(session_id)
    - ValidateFetchRequestDataId(data_id, &error_code, session)
    # 获取数据并设置数据大小和crc等信息
    - DataChunkPB* data_chunk = resp->mutable_chunk();
      string* data = data_chunk->mutable_data()
      GetDataFilePiece(data_id, session, offset, client_maxlen, data,
                                     &total_data_length, &error_code)
      data_chunk->set_total_data_length(total_data_length)
      data_chunk->set_offset(offset)
      uint32_t crc32 = Crc32c(data->data(), data->length());
      data_chunk->set_crc32(crc32);
    # 发送响应
    - context.RespondSuccess()
```

## RemoteBootstrapClient::Finish
```
RemoteBootstrapClient::Finish
    - WriteConsensusMetadata()
        # 如果consensus meta文件不存在，则创建之
        - if (!cmeta_) {
            std::unique_ptr<ConsensusMetadata> cmeta;
            return ConsensusMetadata::Create(fs_manager_, tablet_id_, fs_manager_->uuid(),
                                             remote_committed_cstate_->config(),
                                             remote_committed_cstate_->current_term(),
                                             &cmeta);
          }
        # 根据远端的bootstrap源更新consensus meta文件
        - cmeta_->MergeCommittedConsensusStatePB(*remote_committed_cstate_)
        # 持久化consensus meta文件到disk
        - cmeta_->Flush()
    # 将tablet data state的状态从tablet::TABLET_DATA_COPYING切换为tablet::TABLET_DATA_READY
    - new_superblock_->set_tablet_data_state(tablet::TABLET_DATA_READY)
    # 替换superblock（将新的superblock信息持久化到文件）
    - meta_->ReplaceSuperBlock(*new_superblock_)
    # 结束本次的remote bootstrap session，请参考“RemoteBootstrapClient::EndRemoteSession”
    # 因为RemoteBootstrapClient::EndRemoteSession是同步调用，这里会同步等待
    - EndRemoteSession()
```

## RemoteBootstrapClient::EndRemoteSession
```
RemoteBootstrapClient::EndRemoteSession
    # 初始化EndRemoteBootstrapSession的request和response
    - EndRemoteBootstrapSessionRequestPB req;
      req.set_session_id(session_id_);
      req.set_is_success(succeeded_);
      EndRemoteBootstrapSessionResponsePB resp;
    # RemoteBootstrapServiceProxy::EndRemoteBootstrapSession
    # 最终远端的调用是RemoteBootstrapServiceImpl::EndRemoteBootstrapSession
    # 注意这里是同步调用，会等待返回结果！
    - proxy_->EndRemoteBootstrapSession(req, &resp, &controller)
```

## RemoteBootstrapServiceImpl::EndRemoteBootstrapSession
```
RemoteBootstrapServiceImpl::EndRemoteBootstrapSession
    - DoEndRemoteBootstrapSessionUnlocked(req->session_id(), req->is_success(), &app_error)
        # 找到对应的session                                          
        - FindSessionUnlocked(session_id, app_error, &session)
        # 更改远端peer的Role
        - session->ChangeRole()
            # 如果“the being bootstrapped peer”处于RaftPeerPB::OBSERVER/RaftPeerPB::VOTER状态，则直接返回：
              ...
            # 如果“the being bootstrapped peer”处于RaftPeerPB::PRE_OBSERVER/RaftPeerPB::PRE_VOTER状态，则执行如下：
            - consensus::ChangeConfigRequestPB req;
              consensus::ChangeConfigResponsePB resp;
              req.set_tablet_id(tablet_peer_->tablet_id());
              req.set_type(consensus::CHANGE_ROLE);
              RaftPeerPB* peer = req.mutable_server();
              # requestor_uuid_就是“the being bootstrapped peer”对应的uuid
              peer->set_permanent_uuid(requestor_uuid_);
              # 参考前面关于“RaftConsensus::ChangeConfig”的分析
              # 会尝试将“the being bootstrapped peer”的角色修改为OBSERVER或者VOTER
              consensus->ChangeConfig(req, &DoNothingStatusCB, &error_code)
        # 删除该session
        - sessions_.erase(session_id)
        - session_expirations_.erase(session_id)
    # 返回响应
    - context.RespondSuccess()
```

## ConsensusServiceImpl::UpdateConsensus
```
ConsensusServiceImpl::UpdateConsensus
    - GetConsensusOrRespond(tablet_peer, resp, &context, &consensus)
    # RaftConsensus::Update
    - consensus->Update(const_cast<ConsensusRequestPB*>(req), resp)
        # Updates the state in a replica by storing the received operations in the log and triggering the required operations
        # 请参考“RaftConsensus::UpdateReplica”
        - UpdateReplica(request, response)
    # 发送响应
    - context.RespondSuccess()
```

## RaftConsensus::UpdateReplica
```
RaftConsensus::UpdateReplica
    - 更新本地的hybrid time  
      if (request->has_propagated_hybrid_time()) {
        clock_->Update(HybridTime(request->propagated_hybrid_time()));
      }
    # 通过Synchronizer来使得异步调用同步化
    - auto log_synchronizer = std::make_shared<Synchronizer>();
      StatusCallback sync_status_cb = Synchronizer::AsStatusCallback(log_synchronizer);
    # ！！！这里的逻辑还不太清楚！！！
    - ReplicaState::UniqueLock lock
      state_->LockForUpdate(&lock)
      ......
```

## Peer::ProcessResponse
```
Peer::ProcessResponse
    # 丢给thread pool来处理，最终被调度执行的时候处理函数是Peer::DoProcessResponse
    # 请参考"Peer::DoProcessResponse"
    - raft_pool_token_->SubmitClosure(Bind(&Peer::DoProcessResponse, Unretained(this)))
```

## Peer::DoProcessResponse
```
Peer::DoProcessResponse
    # 接收到远端的peer的响应后的处理
    - queue_->ResponseFromPeer(peer_pb_.permanent_uuid(), response_, &more_pending)
        - MajorityReplicatedData majority_replicated
        # 设置last_received和next_index信息
        - bool peer_has_prefix_of_log = IsOpInLog(status.last_received());
          if (peer_has_prefix_of_log) {
            peer->last_received = status.last_received();
            peer->next_index = peer->last_received.index() + 1;
          } else if (!OpIdEquals(status.last_received_current_leader(), MinimumOpId())) {
            peer->last_received = status.last_received_current_leader();
            peer->next_index = peer->last_received.index() + 1;
          } else {
            DCHECK_GE(peer->last_known_committed_idx, 0);
            peer->next_index = peer->last_known_committed_idx + 1;
          }
        - *more_pending = log_cache_.HasOpBeenWritten(peer->next_index) ||
            (peer->last_known_committed_idx < queue_state_.committed_index.index())
        # 设置majority_replicated
        - if (mode_copy == Mode::LEADER) {
            majority_replicated.op_id = queue_state_.majority_replicated_opid;
            majority_replicated.leader_lease_expiration = LeaderLeaseExpirationWatermark();
            majority_replicated.ht_lease_expiration = HybridTimeLeaseExpirationWatermark();
          }
        # 更新在所有peer上都已经replicated的Op id
        - UpdateAllReplicatedOpId(&queue_state_.all_replicated_opid);
        # 剔除那些已经在所有peer上都replicated的消息
        - log_cache_.EvictThroughOp(queue_state_.all_replicated_opid.index())
        - if (mode_copy == Mode::LEADER) {
            NotifyObserversOfMajorityReplOpChange(majority_replicated);
                # 通知所有的Observers
                - raft_pool_observers_token_->SubmitClosure(
                      Bind(&PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask,
                           Unretained(this),
                           majority_replicated_data))
          }
    # 如果还有更多的待处理的请求，则执行SendNextRequest，请参考“Peer::SendNextRequest”
    - if (more_pending && state_.load(std::memory_order_acquire) != kPeerClosed) {
        lock.release();
        SendNextRequest(RequestTriggerMode::kAlwaysSend);
      }
```

## TabletBootstrap::Bootstrap
```
TabletBootstrap::Bootstrap
    - string tablet_id = meta_->tablet_id()
    # 从磁盘中加载ConsensusMetadata，加载的信息存放在@cmeta_中
    - ConsensusMetadata::Load(meta_->fs_manager(), tablet_id,
                                meta_->fs_manager()->uuid(), &cmeta_)
    # 注意这里的TabletBootstrap::OpenTablet和ConsensusServiceImpl::StartRemoteBootstrap ->
    # TSTabletManager::OpenTablet中的OpenTablet是不一样的
    - bool has_blocks = OpenTablet()
        # 构造一个Tablet
        - auto tablet = std::make_unique<TabletClass>(
              meta_, data_.clock, mem_tracker_, metric_registry_, log_anchor_registry_, tablet_options_,
              data_.transaction_participant_context, data_.local_tablet_filter,
              data_.transaction_coordinator_context)
        # 打开tablet
        - tablet->Open()
            # 关于当前tablet的状态和schema检查
            - CHECK_EQ(state_, kInitialized) << "already open"
              CHECK(schema()->has_column_ids())
            # 打开tablet，实际上是为该tablet分别创建一个被称为regular_db_和intents_db_的RocksDB实例，
            # 然后通过构建DocDB实例来统一管理regular_db_和intents_db_，然后又在DocDB的基础上封装了
            # QLRocksDBStorage，并最终对外暴露QLRocksDBStorage类型的实例@ql_storage_
            - OpenKeyValueTablet()
                # 创建regular_db_实例
                - rocksdb::Status rocksdb_open_status = rocksdb::DB::Open(rocksdb_options, db_dir, &db)
                  regular_db_.reset(db)
                # 构建intents_db_实例
                - rocksdb::DB::Open(rocksdb_options, db_dir + kIntentsDBSuffix, &intents_db)
                  intents_db_.reset(intents_db)
                # 构建QLRocksDBStorage类型的实例，并赋值给@ql_storage_
                - ql_storage_.reset(new docdb::QLRocksDBStorage({regular_db_.get(), intents_db_.get()}))
            - state_ = kBootstrapping
        # 检查该tablet中是否包含数据，如果含有数据则返回true，否则false
        - Result<bool> has_ss_tables = tablet->HasSSTables()
            - std::vector<rocksdb::LiveFileMetaData> live_files_metadata
            - regular_db_->GetLiveFilesMetaData(&live_files_metadata);
              return !live_files_metadata.empty();
        - tablet_ = std::move(tablet)
        - return has_ss_tables.get()
    - bool needs_recovery
      PrepareRecoveryDir(&needs_recovery)
        - FsManager* fs_manager = tablet_->metadata()->fs_manager();
          const string& log_dir = tablet_->metadata()->wal_dir();
        # 根据log_dir获取recovery目录的路径，存放在@recovery_path中
        - const string recovery_path = fs_manager->GetTabletWalRecoveryDir(log_dir)
        # 如果@recovery_path存在，则删除@log_dir所在的目录，并重新创建@log_dir所在的目录
        # 先理解@recovery_path不存在的情况，可能会更好理解这里的处理
        - if (fs_manager->Exists(recovery_path)) {
            if (fs_manager->env()->FileExists(log_dir)) {
                fs_manager->env()->DeleteRecursively(log_dir)
                fs_manager->CreateDirIfMissing(DirName(log_dir))
                fs_manager->CreateDirIfMissing(log_dir)
                *needs_recovery = true;
                # 返回
                return Status::OK();
            }
          }
        # 如果@recovery_path不存在，但是@log_dir目录下有日志文件，则需要执行recovery
        - fs_manager->CreateDirIfMissing(DirName(log_dir))
          fs_manager->CreateDirIfMissing(log_dir)
          fs_manager->ListDir(log_dir, &children)
          for (const string& child : children) {
            if (!log::IsLogFileName(child)) {
              continue;
            }
        
            *needs_recovery = true;
          }
        # 如果需要recovery，则将@log_dir重命名为@recovery_path，并重新创建@log_dir目录
        - if (*needs_recovery) {
            fs_manager->env()->RenameFile(log_dir, recovery_path)
            fs_manager->env()->CreateDir(log_dir)
          }
    # 如果需要recovery，则打开关于recovery path的log reader
    - if (needs_recovery) {
        OpenLogReaderInRecoveryDir();
            # 打开recovery path(可以通过tablet_->metadata()->wal_dir()获取之)上的LogReader
            - LogReader::OpenFromRecoveryDir(tablet_->metadata()->fs_manager(),
                                               tablet_->metadata()->tablet_id(),
                                               tablet_->metadata()->wal_dir(),
                                               tablet_->GetMetricEntity().get(),
                                               &log_reader_)
      }
    # 如果是一个全新的tablet，则在tablet的log directory下打开一个log文件，并结束bootstrap过程
    - if (!has_blocks && !needs_recovery) {
        OpenNewLog()
            # 在log directory下打开一个日志文件@log_
            - Log::Open(LogOptions(),
                          tablet_->metadata()->fs_manager(),
                          tablet_->tablet_id(),
                          tablet_->metadata()->wal_dir(),
                          *tablet_->schema(),
                          tablet_->metadata()->schema_version(),
                          tablet_->GetMetricEntity(),
                          append_pool_,
                          &log_)
            # 暂时禁用sync，以加速bootstrap过程
            - log_->DisableSync()
        FinishBootstrap("No bootstrap required, opened a new log",
                                  rebuilt_log,
                                  rebuilt_tablet)
        # @consensus_info将作为返回值并通过TabletBootstrap::Bootstrap返回给上层调用BootstrapTablet，
        # 进而通过BootstrapTablet返回给上层调用TSTabletManager::OpenTablet
        consensus_info->last_id = MinimumOpId();
        consensus_info->last_committed_id = MinimumOpId();
        return Status::OK();                                  
      }
    # 如果有数据但又无需recovery，则肯定出现了问题
    - if (has_blocks && !needs_recovery) {
        //报错并返回
        ...
      }
    # 见“TabletBootstrap::PlaySegments”
    # 这里也会设置@consensus_info，并通过TabletBootstrap::Bootstrap返回给上层调用BootstrapTablet，
    # 进而通过BootstrapTablet返回给上层调用TSTabletManager::OpenTablet
    - PlaySegments(consensus_info)
    # Flush the consensus metadata
    - cmeta_->Flush()
    - RemoveRecoveryDir()
    - FinishBootstrap("Bootstrap complete.", rebuilt_log, rebuilt_tablet)
```

## TabletBootstrap::PlaySegments
```
TabletBootstrap::PlaySegments
    # 分别获取regular db和intent db中最大的已经持久化的op id，统一存放在result中
    - auto flushed_op_id = tablet_->MaxPersistentOpId()
        - DocDbOpIds result
          auto temp = regular_db_->GetFlushedFrontier()
          if (temp) {
            result.regular = down_cast<docdb::ConsensusFrontier*>(temp.get())->op_id();
          }
          if (intents_db_) {
            temp = intents_db_->GetFlushedFrontier();
            if (temp) {
              result.intents = down_cast<docdb::ConsensusFrontier*>(temp.get())->op_id();
            }
          }
          return result;
    # 根据@flushed_op_id设置ReplayState @state，顾名思义ReplayState用于控制replay过程的状态
    - consensus::OpId regular_op_id;
      regular_op_id.set_term(flushed_op_id.regular.term);
      regular_op_id.set_index(flushed_op_id.regular.index);
      consensus::OpId intents_op_id;
      intents_op_id.set_term(flushed_op_id.intents.term);
      intents_op_id.set_index(flushed_op_id.intents.index);
      ReplayState state(regular_op_id, intents_op_id);
    # @log_reader_是在TabletBootstrap::OpenLogReaderInRecoveryDir中获取的
    # 获取所有的LogSegments，存放在@segments中，这里log_reader_读取的都是recovery目录下的文件，
    - log::SegmentSequence segments
      log_reader_->GetSegmentsSnapshot(&segments)
    # 在log directory下打开一个新的log文件
    - OpenNewLog()
    # 逐一处理各log segments
    - for (const scoped_refptr<ReadableLogSegment>& segment : segments) {
        log::LogEntries entries;
        # 读取当前log segment中的所有log entry
        Status read_status = segment->ReadEntries(&entries);
        # 逐一处理当前log segment中的所有log entry
        for (int entry_idx = 0; entry_idx < entries.size(); ++entry_idx) {
          Status s = HandleEntry(&state, &entries[entry_idx]);
            # 请参考“TabletBootstrap::HandleReplicateMessage”
            - HandleReplicateMessage(state, entry_ptr)
        }
        segment_count++;
      }
    # 设置@consensus_info，并通过TabletBootstrap::Bootstrap返回给上层调用BootstrapTablet，
    # 进而通过BootstrapTablet返回给上层调用TSTabletManager::OpenTablet
    # 关于state.prev_op_id的设置，见TabletBootstrap::HandleReplicateMessage -> ReplayState::CheckSequentialReplicateId
    # 关于state.committed_op_id的设置，见TabletBootstrap::HandleReplicateMessage -> ReplayState::UpdateCommittedOpId
    - tablet_->mvcc_manager()->SetLastReplicated(state.rocksdb_last_entry_hybrid_time);
      consensus_info->last_id = state.prev_op_id;
      consensus_info->last_committed_id = state.committed_op_id;
```


## TabletBootstrap::HandleReplicateMessage
```
TabletBootstrap::HandleReplicateMessage
    - const ReplicateMsg& replicate = replicate_entry.replicate()
    - UpdateClock(replicate.hybrid_time())
    - tablet_->UpdateMonotonicCounter(replicate.monotonic_counter())
    # 同步添加一个log entry到log文件中
    - log_->Append(replicate_entry_ptr->get())
        - LogEntryBatchPB entry_batch_pb;
          entry_batch_pb.mutable_entry()->AddAllocated(phys_entry);
          LogEntryBatch entry_batch(phys_entry->type(), &entry_batch_pb, 1);
          entry_batch.state_ = LogEntryBatch::kEntryReserved;
          entry_batch.MarkReady();
          DoAppend(&entry_batch, false);
            # 将log entry写入到active_segment_中
            # active_segment_对应的文件存放在log directory下
            - Slice entry_batch_data = entry_batch->data()
              active_segment_->WriteEntryBatch(entry_batch_data)
          Sync()
            # Makes sure the I/O buffers in the underlying writable file are flushed
            - active_segment_->Sync()
            # Update the reader on how far it can read the active segment
            - reader_->UpdateLastSegmentOffset(active_segment_->written_offset())
```

## TabletPeer::Start
```
TabletPeer::Start
    # RaftConsensus::Start，启动raft共识算法
    - consensus_->Start(bootstrap_info)
        # 启动consensus，通过传递进来的@bootstrap_info来设置ReplicaState
        - ReplicaState::UniqueLock lock;
          state_->LockForStart(&lock);
          state_->ClearLeaderUnlocked();
          state_->StartUnlocked(info.last_id)
              # 设置next_index和last_received_op_id_等
              - next_index_ = last_id_in_wal.index() + 1
                last_received_op_id_.CopyFrom(last_id_in_wal)
                state_ = kRunning
          state_->InitCommittedIndexUnlocked(info.last_committed_id)
          queue_->Init(state_->GetLastReceivedOpIdUnlocked())
        # 承担replica的角色
        - ReplicaState::UniqueLock lock;
          state_->LockForConfigChange(&lock)
          BecomeReplicaUnlocked()
        # 向master汇报consensus状态变更
        - MarkDirty(context)
            - mark_dirty_clbk_.Run(context)
    # 更新状态为RUNNING
    - UpdateState(TabletStatePB::BOOTSTRAPPING, TabletStatePB::RUNNING,
                              "Incorrect state to start TabletPeer, ")
    # 因为更新了tablet的状态，需要向master汇报，mark_dirty_clbk_是在TabletPeer构造函数中设置的，
    # 用于向TabletManager汇报tablet状态变更，关于mark_dirty_clbk_的设置请参考
    # TSTabletManager::CreateAndRegisterTabletPeer -> TSTabletManager::ApplyChange
    - auto context =
      std::make_shared<StateChangeContext>(StateChangeReason::TABLET_PEER_STARTED, false);
      # 实际调用的是TSTabletManager::ApplyChange
      mark_dirty_clbk_.Run(context)
        # 最终调用的是TSTabletManager::MarkTabletDirty
        - apply_pool_->SubmitFunc(
          std::bind(&TSTabletManager::MarkTabletDirty, this, tablet_id, context))
            - MarkDirtyUnlocked(tablet_id, context)
                # 如果在@dirty_tablets_中没找到关于该tablet的TabletReportState，则将该tablet的TabletReportState
                # 信息插入到@dirty_tablets_中
                - TabletReportState* state = FindOrNull(dirty_tablets_, tablet_id);
                  if (state != nullptr) {
                    CHECK_GE(next_report_seq_, state->change_seq);
                    state->change_seq = next_report_seq_;
                  } else {
                    TabletReportState state;
                    state.change_seq = next_report_seq_;
                    InsertOrDie(&dirty_tablets_, tablet_id, state);
                  }
                  # 在和master的心跳中汇报关于tablet的变更
                  # 会依次进入Heartbeater::Thread::RunThread -> Heartbeater::Thread::DoHeartbeat ->
                  # Heartbeater::Thread::TryHeartbeat，请参考Heartbeater::Thread::TryHeartbeat
                  server_->heartbeater()->TriggerASAP()
```

## Heartbeater::Thread::TryHeartbeat
```
Heartbeater::Thread::TryHeartbeat
    # 填充heartbeat请求
    - master::TSHeartbeatRequestPB req;
      SetupCommonField(req.mutable_common());
      if (last_hb_response_.needs_reregister()) {
        SetupRegistration(req.mutable_registration());
      }
    
      if (last_hb_response_.needs_full_tablet_report()) {
        server_->tablet_manager()->GenerateFullTabletReport(req.mutable_tablet_report());
      } else {
        server_->tablet_manager()->GenerateIncrementalTabletReport(req.mutable_tablet_report());
      }
      req.set_num_live_tablets(server_->tablet_manager()->GetNumLiveTablets());
      req.set_leader_count(server_->tablet_manager()->GetLeaderCount());
      req.set_config_index(server_->GetCurrentMasterIndex())
    # 发送heartbeat消息，会同步等待，在Master端的处理逻辑是“MasterServiceImpl::TSHeartbeat”
      proxy_->TSHeartbeat(req, &resp, &rpc)
    # 如果响应中包含了master config信息，则更新master
      if (resp.has_master_config()) {
        server_->UpdateMasterAddresses(resp.master_config());
      }
    # 标记Master已经成功处理了tablet report信息
    - server_->tablet_manager()->MarkTabletReportAcknowledged(req.tablet_report())
    # 更新tablet servers列表
    - server_->PopulateLiveTServers(resp)
        - live_tservers_.assign(heartbeat_resp.tservers().begin(), heartbeat_resp.tservers().end())
```

## MasterServiceImpl::TSHeartbeat
```
MasterServiceImpl::TSHeartbeat
    # 其它逻辑暂时不关注
    - ...
    # 处理新的tablet汇报，见“CatalogManager::ProcessTabletReport”
    - if (req->has_tablet_report()) {
        s = server_->catalog_manager()->ProcessTabletReport(
          ts_desc.get(), req->tablet_report(), resp->mutable_tablet_report(), &rpc);
      }
```

## CatalogManager::ProcessTabletReport
```
CatalogManager::ProcessTabletReport
    # 其它逻辑暂时不关注
    - ...
    # 逐一处理汇报上来的tablets
    - for (const ReportedTabletPB& reported : report.updated_tablets()) {
        ReportedTabletUpdatesPB *tablet_report = report_update->add_tablets();
        tablet_report->set_tablet_id(reported.tablet_id());
        # 处理该汇报上来的tablet
        HandleReportedTablet(ts_desc, reported, tablet_report);
  }
```



上述过程分析了新添加的tablet peer将自身数据同步到某个特定的时间点的过程，即bootstrap过程。但是在该新添加的tablet peer bootstrap过程中，其它的2个tablet peer还可以选举出leader，也能保持该tablet的可用性，那么在新的tablet peer bootstrap过程中，可能有新的数据IO写入到tablet，那么在tablet peer bootstrap过程结束之后，它依然跟另外2个tablet peer之间不同步，这又是如何处理的呢？答案是raft机制。

当tablet replica上bootstrap过程结束之后，会执行TabletPeer::Start，会根据bootstrap过程中返回的consensus::ConsensusBootstrapInfo信息来启动tablet的consensus，并根据该信息设置ReplicaState，具体过程请参考“TSTabletManager::OpenTablet”。

当tablet replica处理完consensus更新请求之后（见ConsensusServiceImpl::UpdateConsensus），会发送响应给tablet leader，响应就是根据tablet replica上的ReplicaState进行设置的（见RaftConsensus::FillConsensusResponseOKUnlocked）。

当tablet leader接收到tablet replica的响应之后，处理逻辑是Peer::ProcessResponse，其中会更新tablet leader上所看到的tablet peer的状态，包括last_known_committed_idx（The last committed index this peer knows about）、last_received（The last operation that we've sent to this peer and that it acked）和next_index（Next index to send to the peer，通常就是last_received + 1）等（见Peer::ProcessResponse -> Peer::DoProcessResponse -> PeerMessageQueue::ResponseFromPeer）。

当tablet leader所在的节点向tablet replica发送请求的时候（借助于Peer::SendNextRequest），会调用PeerMessageQueue::RequestForPeer准备一个请求发送给该peer，请求中会包含一系列消息（从tablet leader的PeerMessageQueue中获取这些消息），第一个消息的编号记录在TrackedPeer::next_index中，而TrackedPeer::next_index又是在tablet leader接收到tablet peer的响应的时候根据响应中的信息设置的，以此保证发送过去的消息的连续性（不丢失）。

# tablet复制过程总结
综上所述，在新的tablet server上启动tablet replica的步骤如下：
- Master向tablet leader所在的tablet server发送“Add Server”消息，消息中包括“在哪个tablet server上启动新的tablet replica”的信息，即为new_tablet_server
- tablet leader生成CHANGE_CONFIG(ADD_SERVER)请求，并在raft group中执行该配置变更请求(tablet raft group中成员发生了变更)
    - tablet leader会在PeerManager中添加一个新的代表该tablet replica的tablet peer，见RaftConsensus::ChangeConfig -> ... -> PeerManager::UpdateRaftConfig
    - tablet leader将这个请求记录在log中，见RaftConsensus::ChangeConfig -> RaftConsensus::ReplicateConfigChangeUnlocked -> ... -> LogCache::AppendOperations
    - tablet leader向new_tablet_server发送bootstrap请求，见RaftConsensus::ChangeConfig -> ... -> PeerManager::SignalRequest -> ... -> Peer::SendNextRequest -> ... -> Peer::SendRemoteBootstrapRequest()
- new_tablet_server上处理来自tablet leader的bootstrap请求，见ConsensusServiceImpl::StartRemoteBootstrap
    - new_tablet_server上启动一个RemoteBootstrapClient，并主动请求跟tablet leader之间建立一个RemoteBootstrapSession，见ConsensusServiceImpl::StartRemoteBootstrap -> RemoteBootstrapClient::Start
- tablet leader处理来自new_tablet_server的建立session的请求，见RemoteBootstrapServiceImpl::BeginRemoteBootstrapSession
    - 建立一个RemoteBootstrapSession
    - 返回tablet superblock、consensus state summary和WAL log segments等信息给new_tablet_server
- new_tablet_server处理tablet leader成功建立RemoteBootstrapSession的响应，见ConsensusServiceImpl::StartRemoteBootstrap
    - 根据tablet leader的响应设置session、superblock、wal log、consensus state、schema和partition等信息，见ConsensusServiceImpl::StartRemoteBootstrap -> RemoteBootstrapClient::Start
    - 启动一个tablet peer，见ConsensusServiceImpl::StartRemoteBootstrap -> TSTabletManager::CreateAndRegisterTabletPeer
    - 从tablet leader上下载数据到本地，包括WAL log segments和Rocksdb文件，见ConsensusServiceImpl::StartRemoteBootstrap -> RemoteBootstrapClient::FetchAll
    - 主动结束RemoteBootStrapSession，发送EndRemoteSession请求给tablet leader，见ConsensusServiceImpl::StartRemoteBootstrap -> RemoteBootstrapClient::Finish -> RemoteBootstrapClient::EndRemoteSession
- tablet leader处理EndRemoteSession请求，见RemoteBootstrapServiceImpl::EndRemoteBootstrapSession
    - 找到对应的RemoteBootStrapSession并删除之
    - tablet leader修改对应的tablet replica的状态为RaftPeerPB::VOTER   
    - tablet leader将这个请求记录在log中，见RaftConsensus::ChangeConfig -> RaftConsensus::ReplicateConfigChangeUnlocked -> ... -> LogCache::AppendOperations
    - tablet leader异步的向new_tablet_server发送请求，请求中包括该CHANGE_CONFIG(CHANGE_ROLE)相关的消息，见RaftConsensus::ChangeConfig -> ... -> PeerManager::SignalRequest -> ... -> Peer::SendNextRequest -> RpcPeerProxy::UpdateAsync -> ConsensusServiceProxy::UpdateConsensusAsync
- new_tablet_server根据之前从tablet leader中下载的数据bootstrap一个新的tablet，并启动之，见ConsensusServiceImpl::StartRemoteBootstrap -> TSTabletManager::StartRemoteBootstrap -> TSTabletManager::OpenTablet -> TabletPeer::Start -> RaftConsensus::Start -> RaftConsensus::BecomeReplicaUnlocked
- new_tablet_server向Master汇报自己加入consensus group中了，见ConsensusServiceImpl::StartRemoteBootstrap -> TSTabletManager::StartRemoteBootstrap -> TSTabletManager::OpenTablet -> TabletPeer::Start -> RaftConsensus::Start -> RaftConsensus::MarkDirty
- new_tablet_server处理tablet leader发送过来的Update Consensus请求，处理完毕后向tablet leader发送响应，见ConsensusServiceImpl::UpdateConsensus
- tablet leader处理new_tablet_server中关于Update Consensus请求的响应，更新tablet leader上所看到的tablet peer的状态，包括last_known_committed_idx（The last committed index this peer knows about）、last_received（The last operation that we've sent to this peer and that it acked）和next_index（Next index to send to the peer，通常就是last_received + 1）等，见Peer::ProcessResponse -> Peer::DoProcessResponse -> PeerMessageQueue::ResponseFromPeer
- 当tablet leader所在的节点向tablet replica发送请求的时候，请求中会包含一系列消息，这些消息中第一个消息就是编号为next_index的那条消息，以此保证发送过去的消息的连续性，见Peer::SendNextRequest -> PeerMessageQueue::RequestForPeer

总之，对于一个bootstrap的tablet replica来说，它会从tablet leader中拷贝某个“特定时间点的状态数据”（可以理解为tablet的一个快照，包括rocksdb数据库文件和wal log），然后tablet replica对wal log进行replay，以使得该tablet replica的状态变为该“特定时间点的状态”，然后tablet leader又不断的向该tablet replica发送从这个“特定时间点”开始的wal log，从而使得该tablet replica逐渐catch up该tablet leader。

上面的分析过程在CockroachDB的相关文档中得到了印证（请参考[这里](https://github.com/cockroachdb/cockroach/blob/master/docs/design.md#splitting--merging-ranges)）：
```
Splitting, merging, rebalancing and recovering all follow the same basic algorithm for moving data between roach nodes. New target replicas are created and added to the replica set of source range. Then each new replica is brought up to date by either replaying the log in full or copying a snapshot of the source replica data and then replaying the log from the timestamp of the snapshot to catch up fully. Once the new replicas are fully up to date, the range metadata is updated and old, source replica(s) deleted if applicable.
```


# 参考
[How Does Consensus-Based Replication Work in Distributed Databases?](https://blog.yugabyte.com/how-does-consensus-based-replication-work-in-distributed-databases/)

[How Does the Raft Consensus-Based Replication Protocol Work in YugaByte DB?](https://blog.yugabyte.com/how-does-the-raft-consensus-based-replication-protocol-work-in-yugabyte-db/)
