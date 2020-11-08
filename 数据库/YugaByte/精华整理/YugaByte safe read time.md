# 提纲
[toc]

## safe time的计算
MvccManager::DoGetSafeTime的作用是根据ht_lease(参数)和MvccManager::queue_等信息获取一个不小于min_allowed的HybridTime，为了获取满足条件的HybridTime，可能需要等待，参数lock作为条件等待中的锁，而deadline限定了条件等待的最长时间。

关于safe time的计算，在YugaByte官网上说明如下：
```
Now, suppose the current majority-replicated hybrid time leader lease expiration is
replicated_ht_lease_exp. Then the safe timestamp for a read request can be computed
as the maximum of:
- Last committed Raft entry's hybrid time
- One of:
    - If there are uncommitted entries in the Raft log: the minimum ofthe first 
    uncommitted entry's hybrid time - ε (where ε is the smallest possibledifference 
    in hybrid time) and replicated_ht_lease_exp.
    - If there are no uncommitted entries in the Raft log: the minimum of the current
    hybrid time and replicated_ht_lease_exp.

In other words, the last committed entry's hybrid time is always safe to read at, but
for higher hybrid times, the majority-replicated hybrid time leader lease is an upper
bound. That is because we can only guarantee that no future leader will commit an entry
with hybrid time less than ht if ht < replicated_ht_lease_exp.
```

翻译成中文，就是说：
- 如果raft log中存在uncommitted raft entries，则safe time = max{Last committed Raft entry's hybrid time, min{first uncommitted entry's hybrid time - ε, replicated_ht_lease_exp}}
- 如果raft log中不存在uncommitted raft entries，则safe time = max{Last committed Raft entry's hybrid time, min{current hybrid time - ε, replicated_ht_lease_exp}}

其中，Last committed Raft entry's hybrid time就是MvccManager::DoGetSafeTime的实现中用到的last_replicated_，first uncommitted entry's hybrid time就是MvccManager::DoGetSafeTime的实现中用到的queue_.front().Decremented()，replicated_ht_lease_exp就是MvccManager::DoGetSafeTime的实现中用到的ht_lease。

```
HybridTime MvccManager::DoGetSafeTime(const HybridTime min_allowed,
                                      const CoarseTimePoint deadline,
                                      const FixedHybridTimeLease& ht_lease,
                                      std::unique_lock<std::mutex>* lock) const {
  DCHECK_ONLY_NOTNULL(lock);
  CHECK(ht_lease.lease.is_valid()) << InvariantViolationLogPrefix();
  CHECK_LE(min_allowed, ht_lease.lease) << InvariantViolationLogPrefix();

  const bool has_lease = !ht_lease.empty();
  if (has_lease) {
    # max_ht_lease_seen_表示当前看到的最大的lease expiration HybridTime
    max_ht_lease_seen_ = std::max(ht_lease.lease, max_ht_lease_seen_);
  }

  HybridTime result;
  SafeTimeSource source = SafeTimeSource::kUnknown;
  
  # 这是采用了一个匿名函数(Lambda表达式)，返回的predicate是bool类型，这个predicate
  # 将在下面的conditional wait中用到，代码中的&result, &source表示采用引用传递，并
  # 不是取地址的意思
  #
  # 这个匿名函数的作用是根据ht_lease和MvccManager::queue_等信息获取一个不小于min_allowed
  # 的HybridTime
  auto predicate = [this, &result, &source, min_allowed, time = ht_lease.time, has_lease] {
    if (queue_.empty()) {
      result = time.is_valid() ? std::max(max_safe_time_returned_with_lease_.safe_time, time)
                               : clock_->Now();
      source = SafeTimeSource::kNow;
    } else {
      # MvccManager::queue_不为空的情况下，采用MvccManager::queue_队列头部的元素对应的
      # HybridTime减去1作为result
      result = queue_.front().Decremented();
      source = SafeTimeSource::kNextInQueue;
    }

    # result应该不超过max_ht_lease_seen_
    if (has_lease && result > max_ht_lease_seen_) {
      result = max_ht_lease_seen_;
      source = SafeTimeSource::kHybridTimeLease;
    }

    # result应该不小于last_replicated_
    # 其中last_replicated_的更新逻辑在WriteOperationState::Commit -> MvccManager::Replicated中，
    # 通过WriteOperationState::hybrid_time_来更新last_replicated_
    result = std::max(result, last_replicated_);

    # 返回result是否大于min_allowed
    return result >= min_allowed;
  };

  if (deadline == CoarseTimePoint::max()) {
    # 如果deadline被设置为CoarseTimePoint::max()，则表示conditional wait没有超时时间，
    # 也就是说会一直等待，直到predicate变为true(在当前上下文中，就是直到获取一个不小
    # 于min_allowed的result)
    #
    # 在当前上下文中，deadline确实被设置为CoarseTimePoint::max()，见
    # MvccManager::UpdatePropagatedSafeTimeOnLeader
    cond_.wait(*lock, predicate);
  } else if (!cond_.wait_until(*lock, deadline, predicate)) {
    # 这里会进行conditional wait，直到deadline过期，或者predicate为true为止
    return HybridTime::kInvalid;
  }

  # 获取到的result一定大于之前设定的max_safe_time_returned_with_lease_.safe_time或者
  # max_safe_time_returned_without_lease_.safe_time
  auto enforced_min_time = has_lease ? max_safe_time_returned_with_lease_.safe_time
                                     : max_safe_time_returned_without_lease_.safe_time;
  CHECK_GE(result, enforced_min_time);

  # 更新max_safe_time_returned_with_lease_或者max_safe_time_returned_without_lease_
  if (has_lease) {
    max_safe_time_returned_with_lease_ = { result, source };
  } else {
    max_safe_time_returned_without_lease_ = { result, source };
  }
  
  # 返回result
  return result;
}
```

那么ht_lease和MvccManager::queue_分别是怎么维护的呢？且看下文。

## safe time计算中所用到的ht_lease的维护
ht_lease就是leader lease HybridTime的意思。

leader lease主要是用于在网络分区的情况下读取最新的数据。考虑以下的场景：
![image](https://docs.yugabyte.com/images/architecture/txn/leader_leases_network_partition.svg)
- 因为网络分区，old leader和它的followers被分到了不同的partition
- 然后new leader被选举出来
- client更新一条记录，old leader所在的raft group成功的完成了复制
- client从old leader上读取到过时的数据

leader lease就是用于解决这种问题，它的运行逻辑如下：
- 每一个从leader发送给follower的消息(也就是raft术语中的AppendEntries)中都会带上lease duration(可以通过FLAGS_ht_lease_duration_ms设置，默认是2000)信息。
- 每一个从leader发送给follower的消息(也就是raft术语中的AppendEntries)中还会带上leader lease expiration time(leader本地的local monotonic time + lease duration)信息。
- 当follower接收到来自leader的消息的时候，它获取它所在的节点的local monotonic time，并和接收到的消息中的lease duration相加，相加后的和作为该follower上所记录的leader lease expiration time(注意，follower上的leader lease expiration time = follower本地的local monotonic time + lease duration)。如果该follower成为了新的leader，则它只能在它上所记录的leader lease expiration time过期之后才能对外提供一致性读服务和写服务。
- 当follower接收到来自leader的消息的时候，它将从消息中解析出leader lease expiration time(注意，这个leader lease expiration time是在leader发送消息的时候计算出来的)，并在响应中包含该leader lease expiration time信息。当leader接收到大多数节点的响应之后，就会认为该leader lease expiration time已经是majority replicated的了。leader将使用该majority replicated leader lease expiration time来决定它是否可以对外提供一致性读服务和写服务。
- 在选举过程中，当raft peer接收到RequestVote消息的时候，会将它本地所记录的具有最大剩余时间的leader lease expiration time包含在RequestVote消息的响应中，新选举出来的leader至少需要等待该时间过去之后才可以对外提供一致性读服务和写服务。


从上面的分析可知，对于leader lease expiration time，在leader上是通过leader本地的local monotonic time + leader duration计算得到，在follower上是通过follower本地的local monotonic time + leader duration计算得到。对于majority replicated leader lease expiration time则是通过leader本地的local monotonic time + leader duration计算得到，它跟leader上的leader lease expiration time计算方式是一样的，那为什么majority replicated leader lease expiration time不直接使用leader上的leader lease expiration time呢？原因在于majority replicated leader lease expiration time一定是要经过raft group中多数节点replicate通过的，才会成为majority replicated leader lease expiration time，如果直接使用用leader上的leader lease expiration time，则在网络分区的情况下，用leader上的leader lease expiration time仍然会不断更新，相当于leader lease在不断延展，而如果使用majority replicated leader lease expiration time，则在网络分区的情况下，majority replicated leader lease expiration time则不会被更新，leader lease则不会延展，所以为了解决网络分区情况下，从旧的leader读取过时的数据的问题，必须采用majority replicated leader lease expiration time。

leader上记录的majority replicated leader lease expiration time主要用于解决“网络分区的情况下，从旧的leader读取过时的数据的问题”。而follower上记录的leader lease expiration time主要用于解决“网络分区的情况下，新选举出来的leader何时可以对外提供一致性读服务和写服务的问题”。

leader向follower发送消息的时候会附带lease duration和leader lease expiration time(leader本地的local monotonic time + lease duration)相关的代码如下；
```
Status PeerMessageQueue::RequestForPeer(const string& uuid,
                                        ConsensusRequestPB* request,
                                        ReplicateMsgsHolder* msgs_holder,
                                        bool* needs_remote_bootstrap,
                                        RaftPeerPB::MemberType* member_type,
                                        bool* last_exchange_successful) {
  {
    LockGuard lock(queue_lock_);

    HybridTime now_ht;

    bool is_new = peer->is_new;
    if (!is_new) {
      now_ht = clock_->Now();

      # now_ht.GetPhysicalValueMicros()获得的是leader上的local monotonic time
      auto ht_lease_expiration_micros = now_ht.GetPhysicalValueMicros() +
                                        FLAGS_ht_lease_duration_ms * 1000;
      auto leader_lease_duration_ms = GetAtomicFlag(&FLAGS_leader_lease_duration_ms);
      # 在request中设置leader_lease_duration_ms_
      request->set_leader_lease_duration_ms(leader_lease_duration_ms);
      # 在request中设置ht_lease_expiration_
      request->set_ht_lease_expiration(ht_lease_expiration_micros);
    
      # 并更新当前消息对应的目标peer上的leader_ht_lease_expiration::last_sent为ht_lease_expiration_micros
      peer->leader_ht_lease_expiration.last_sent = ht_lease_expiration_micros;
      
      ...
    }
    
    ...
  }
```

follower接收到leader的消息的时候，对于request中的leader_lease_duration_ms_和置ht_lease_expiration_分别进行处理：
- 使用request中的leader_lease_duration_ms_加上local monotonic time来设置ReplicaState::old_leader_lease_
- 使用request中的ht_lease_expiration_来设置ReplicaState::old_leader_ht_lease_

```
Result<RaftConsensus::UpdateReplicaResult> RaftConsensus::UpdateReplica(
    ConsensusRequestPB* request, ConsensusResponsePB* response) {
  if (request->has_leader_lease_duration_ms()) {
    # 使用CoarseMonoClock::now() + request->leader_lease_duration_ms()更新follower上ReplicaState::old_leader_lease_，
    # ReplicaState::old_leader_lease_将作为follower上记录的leader lease expiration time
    # 
    # 使用request->ht_lease_expiration()更新follower上ReplicaState::old_leader_ht_lease_，
    # ReplicaState::old_leader_ht_lease_并没有伴随响应发送回leader
    state_->UpdateOldLeaderLeaseExpirationOnNonLeaderUnlocked(
        CoarseTimeLease(deduped_req.leader_uuid,
                        CoarseMonoClock::now() + request->leader_lease_duration_ms() * 1ms),
        PhysicalComponentLease(deduped_req.leader_uuid, request->ht_lease_expiration()));
  }
}  
```

当接收到follower的响应之后，leader会设置该follower对应的raft peer中的leader_ht_lease_expiration::last_received。当接收到raft group中大多数raft peer的响应的时候，leader会从所有的raft peers中维护的leader_lease_expiration.last_received中获取满足majority数目的leader_lease_expiration.last_received，并以此设置majority_replicated.leader_lease_expiration，最后会将该majority_replicated
```
bool PeerMessageQueue::ResponseFromPeer(const std::string& peer_uuid,
                                        const ConsensusResponsePB& response) {
    ...
    
    # 获取当前响应对应的peer
    TrackedPeer* peer = FindPtrOrNull(peers_map_, peer_uuid);
    
    ...
    
    if (mode_copy == Mode::LEADER) {
      ...
      
      # 使用当前响应对应的peer上的leader_ht_lease_expiration::last_sent来更新
      # 当前响应对应的peer上的leader_ht_lease_expiration::last_received，关于
      # 当前响应对应的peer上的leader_ht_lease_expiration::last_sent的设置是在
      # PeerMessageQueue::RequestForPeer中完成的
      peer->leader_ht_lease_expiration.OnReplyFromFollower();
      
      # 获取raft group中多数成员复制成功的leader_ht_lease_expiration::last_received
      # HybridTimeLeaseExpirationWatermark::ExtractValue中会从每个raft peer中获取
      # leader_ht_lease_expiration.last_received，找出raft group中所有raft peer中
      # 满足majority要求的leader_ht_lease_expiration.last_received，比如说raft group中
      # 有5个raft peer，当前的所有的raft peers对应的leader_ht_lease_expiration.last_received
      # 分别为t3, t1, t4, t2, t5，且满足t1 < t2 < t3 < t4 < t5，则满足majority要求的
      # leader_ht_lease_expiration.last_received为t3
      #
      # majority_replicated是局部变量，类型为MajorityReplicatedData
      majority_replicated.ht_lease_expiration = HybridTimeLeaseExpirationWatermark();
      
      ...
    }
    
    ...
    
  # 更新leader本地的ReplicaState中的相关信息，包括majority_replicated_lease_expiration_
  # 和majority_replicated_ht_lease_expiration_等
  if (mode_copy == Mode::LEADER) {
    NotifyObserversOfMajorityReplOpChange(majority_replicated);
  }
  
  return result；
}    

CoarseTimePoint PeerMessageQueue::LeaderLeaseExpirationWatermark() {
  struct Policy {
    ...

    # 从raft peer中获取leader_ht_lease_expiration.last_received
    static result_type ExtractValue(const TrackedPeer& peer) {
      return peer.leader_ht_lease_expiration.last_received;
    }

    ...
  };
  
  # 从当前raft group中的所有的raft peer中获取满足majority数目的
  # leader_ht_lease_expiration.last_received
  return GetWatermark<Policy>();
}
```

在PeerMessageQueue::NotifyObserversOfMajorityReplOpChange中更新leader本地的ReplicaState中的相关信息，包括majority_replicated_ht_lease_expiration_等。
```
PeerMessageQueue::NotifyObserversOfMajorityReplOpChange(
    const MajorityReplicatedData& majority_replicated_data)
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
RaftConsensus::UpdateMajorityReplicated
ReplicaState::SetMajorityReplicatedLeaseExpirationUnlocked

void ReplicaState::SetMajorityReplicatedLeaseExpirationUnlocked(
    const MajorityReplicatedData& majority_replicated_data,
    EnumBitSet<SetMajorityReplicatedLeaseExpirationFlag> flags) {
  ...
  
  # 更新leader对应的ReplicaState中的majority_replicated_ht_lease_expiration_信息
  majority_replicated_ht_lease_expiration_.store(majority_replicated_data.ht_lease_expiration,
                                                 std::memory_order_release);
  ...                                                 
}
```

Leader本地的Replicate信息中的majority_replicated_ht_lease_expiration_将用于判断Leader的HybridTime lease status。
```
ReplicaState::GetHybridTimeLeaseStatusAtUnlocked
GetHybridTimeLeaseStatusAtPolicy::MajorityReplicatedLeaseExpired
bool ReplicaState::MajorityReplicatedHybridTimeLeaseExpiredAt(MicrosTime hybrid_time) const {
  return hybrid_time >= majority_replicated_ht_lease_expiration_;
}
```

在每次调用MvccManager::DoGetSafeTime获取safe time之前，都必须获取ht_lease作为MvccManager::DoGetSafeTime的参数，这主要是通过TabletPeer::HybridTimeLease获得的。
```
FixedHybridTimeLease TabletPeer::HybridTimeLease(MicrosTime min_allowed, CoarseTimePoint deadline) {
  auto time = clock_->Now();
  
  # 获取leader的lease expiration HybridTime
  # RaftConsensus::MajorityReplicatedHtLeaseExpiration -> 
  # ReplicaState::MajorityReplicatedHtLeaseExpiration
  MicrosTime lease_micros {
      consensus_->MajorityReplicatedHtLeaseExpiration(min_allowed, deadline) };

  FixedHybridTimeLease lease;
  if (!lease_micros) {
	  lease.time = HybridTime::kInvalid;
	  lease.lease = HybridTime::kInvalid;

	  return lease;
  }
  
  lease.time = time;
  lease.lease = HybridTime(lease_micros, /* logical */ 0);

  return lease;
}

MicrosTime ReplicaState::MajorityReplicatedHtLeaseExpiration(
    MicrosTime min_allowed, CoarseTimePoint deadline) const {
  # 获取当前leader的lease expiration HybridTime
  # 关于ReplicaState::majority_replicated_ht_lease_expiration_的维护见下文
  auto result = majority_replicated_ht_lease_expiration_.load(std::memory_order_acquire);
  
  # 如果大于min_allowd，则直接返回获取到的lease expiration HybridTime
  if (result >= min_allowed) { // Fast path
    return result;
  }

  # 至此，result小于min_allowed，则进入条件等待，直到deadline过期，或者
  # leader的lease expiration HybridTime不小于min_allowd为止
  
  UniqueLock l(update_lock_);
  
  # 这是一个匿名函数，用于下面的条件等待，如果获取到的leader的lease expiration
  # HybridTime不小于min_allowed，则返回true，否则返回false
  auto predicate = [this, &result, min_allowed] {
    result = majority_replicated_ht_lease_expiration_.load(std::memory_order_acquire);
    return result >= min_allowed;
  };
  
  # 如果deadline等于CoarseTimePoint::max()，则条件等待没有超时时间，等待直到
  # predicate变为true为止
  if (deadline == CoarseTimePoint::max()) {
    cond_.wait(l, predicate);
  } else if (!cond_.wait_until(l, deadline, predicate)) {
    # 否则，等待直到deadline到达或者predicate为true为止
    return 0;
  }
  
  return result;
}
```

上文在调用ReplicaState::MajorityReplicatedHtLeaseExpiration()获取leader的lease expiration HybridTime的时候会用到ReplicaState::majority_replicated_ht_lease_expiration_，对ReplicaState::majority_replicated_ht_lease_expiration_的更新在下面的调用链中(更新逻辑已经在前面讲解过了)：
```
PeerMessageQueue::ResponseFromPeer
PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
RaftConsensus::UpdateMajorityReplicated
ReplicaState::SetMajorityReplicatedLeaseExpirationUnlocked

void ReplicaState::SetMajorityReplicatedLeaseExpirationUnlocked(
    const MajorityReplicatedData& majority_replicated_data,
    EnumBitSet<SetMajorityReplicatedLeaseExpirationFlag> flags) {
  majority_replicated_lease_expiration_ = majority_replicated_data.leader_lease_expiration;
  # 更新majority_replicated_ht_lease_expiration_
  majority_replicated_ht_lease_expiration_.store(majority_replicated_data.ht_lease_expiration,
                                                 std::memory_order_release);
  ...                                                 
}                                
```

## safe time计算中所用到的MvccManager::queue_的维护
MvccManager中，除了MvccManager::queue_和MvccManager::aborted_，

在操作执行之初，将为操作分配一个HybridTime，并且将该HybridTime添加到MvccManager::queue_中。在将HybridTime添加到MvccManager::queue_之前，还会对MvccManager::queue_和MvccManager::aborted_队列进行整理，从MvccManager::queue_和MvccManager::aborted_队列中移除以MvccManager::aborted_队列头部元素起始的最长的连续的交叠的记录。
```
OperationDriver::StartOperation
Operation::Start
WriteOperation::DoStart
Tablet::StartOperation
MvccManager::AddPending

void MvccManager::AddPending(HybridTime* ht) {
  # 如果ht已经被设置，则表明当前Tablet的角色是follower
  const bool is_follower_side = ht->is_valid();
  HybridTime provided_ht = *ht;

  std::lock_guard<std::mutex> lock(mutex_);

  if (is_follower_side) {
    VLOG_WITH_PREFIX(1) << "AddPending(" << *ht << ")";
  } else {
    # 这是leader上的一个新的事务，为之分配一个HybridTime
    *ht = clock_->Now();
    VLOG_WITH_PREFIX(1) << "AddPending(<invalid>), time from clock: " << *ht;
  }

  # 从queue_和aborted_队列中移除以aborted_队列头部元素起始的最长的连续的交叠的记录
  if (!queue_.empty() && *ht <= queue_.back() && !aborted_.empty()) {
    // To avoid crashing with an invariant violation on leader changes, we detect the case when
    // an entire tail of the operation queue has been aborted. Theoretically it is still possible
    // that the subset of aborted operations is not contiguous and/or does not end with the last
    // element of the queue. In practice, though, Raft should only abort and overwrite all
    // operations starting with a particular index and until the end of the log.
    auto iter = std::lower_bound(queue_.begin(), queue_.end(), aborted_.top());

    // Every hybrid time in aborted_ must also exist in queue_.
    CHECK(iter != queue_.end()) << InvariantViolationLogPrefix();

    auto start_iter = iter;
    while (iter != queue_.end() && *iter == aborted_.top()) {
      aborted_.pop();
      iter++;
    }
    
    queue_.erase(start_iter, iter);
  }
  
  HybridTime last_ht_in_queue = queue_.empty() ? HybridTime::kMin : queue_.back();

  ...
  
  # 将当前操作对应的HybridTime添加到queue_中
  queue_.push_back(*ht);
}
```

如果未能将操作成功提交给raft，则操作被abort。对于被abort的操作，分2中情况分别处理：
- 如果被abort的操作对应的HybridTime等于MvccManager::queue_头部的元素，则直接从MvccManager::queue_中移除该HybridTime，无需添加到MvccManager::aborted_队列中
- 如果被abort的操作对应的HybridTime不等于MvccManager::queue_头部的元素，则将该HybridTime添加到MvccManager::aborted_队列中(该HybridTime也一定在MvccManager::queue_中)
```
# 将操作提交给raft进行复制
TabletPeer::Submit
# 如果未能将操作成功提交给raft(创建或者初始化OperationDriver失败)
Operation::Aborted
WriteOperation::DoAborted
WriteOperationState::Abort
MvccManager::Aborted

void MvccManager::Aborted(HybridTime ht) {
  {
    std::lock_guard<std::mutex> lock(mutex_);
    CHECK(!queue_.empty()) << InvariantViolationLogPrefix();
    if (queue_.front() == ht) {
      # 被abort的操作对应的HybridTime就是queue_头部的元素，则直接从queue_头部pop出来
      PopFront(&lock);
    } else {
      # 否则，添加到aborted_队列中
      aborted_.push(ht);
      return;
    }
  }
  cond_.notify_all();
}
```

当操作成功完成raft复制时，从MvccManager::queue_队列头部pop出来对应的HybridTime。
```
PeerMessageQueue::ResponseFromPeer
PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
RaftConsensus::UpdateMajorityReplicated
ReplicaState::UpdateMajorityReplicatedUnlocked
ReplicaState::AdvanceCommittedOpIdUnlocked
ReplicaState::ApplyPendingOperationsUnlocked
ReplicaState::NotifyReplicationFinishedUnlocked
ConsensusRound::NotifyReplicationFinished
OperationDriver::ReplicationFinished
OperationDriver::ApplyOperation
OperationDriver::ApplyTask
Operation::Replicated
WriteOperation::DoReplicated
# 在执行下面的操作之前会应用更新到rocksdb中
WriteOperationState::Commit
MvccManager::Replicated

void MvccManager::Replicated(HybridTime ht) {
  {
    std::lock_guard<std::mutex> lock(mutex_);
    # 断言：queue_一定不为空
    CHECK(!queue_.empty()) << InvariantViolationLogPrefix();
    # 断言：queue_头部的元素一定是当前完成的操作对应的HybridTime，如果是队列头部
    # 的HybridTime对应的操作被abort了，则对应的HybridTime一定不在queue_中了，所以
    # 这个断言是成立的
    CHECK_EQ(queue_.front(), ht) << InvariantViolationLogPrefix();
    # 从队列头部pop出来对应的HybridTime
    PopFront(&lock);
    last_replicated_ = ht;
  }
  
  cond_.notify_all();
}
```


## safe time的使用
### 更新MvccManager::propagated_safe_time_
每当majority replicated watermarks发生变化的时候，都会：
- 获取当前的HybridTime，和当前raft leader的lease expiration HybridTime，见TabletPeer::MajorityReplicated
- 根据前一步骤中获取的HybridTime，和当前raft leader的lease expiration HybridTime，以及MvccManager::queue_等信息获取当前的safe time，见MvccManager::DoGetSafeTime
- 使用获取到的safe time来更新MvccManager::propagated_safe_time_，见MvccManager::UpdatePropagatedSafeTimeOnLeader
```
PeerMessageQueue::ResponseFromPeer
PeerMessageQueue::NotifyObserversOfMajorityReplOpChange
PeerMessageQueue::NotifyObserversOfMajorityReplOpChangeTask
RaftConsensus::UpdateMajorityReplicated
TabletPeer::MajorityReplicated
MvccManager::UpdatePropagatedSafeTimeOnLeader
MvccManager::DoGetSafeTime


void TabletPeer::MajorityReplicated() {
  # 返回的ht_lease类型是FixedHybridTimeLease，其中包括2个成员：
  # - time：表示当前的HybridTime
  # - lease：表示当前raft leader的lease expiration HybridTime(也就是以HybridTime
  #   表示的lease expiration time)
  auto ht_lease = HybridTimeLease(/* min_allowed */ 0, /* deadline */ CoarseTimePoint::max());
  if (ht_lease.lease) {
    # MvccManager::UpdatePropagatedSafeTimeOnLeader，根据当前raft leader的lease
    # expiration HybridTime等信息来更新MvccManager::propagated_safe_time_
    tablet_->mvcc_manager()->UpdatePropagatedSafeTimeOnLeader(ht_lease);
  }
}

# 更新MvccManager::propagated_safe_time_
void MvccManager::UpdatePropagatedSafeTimeOnLeader(const FixedHybridTimeLease& ht_lease)
    NO_THREAD_SAFETY_ANALYSIS {
  {
    std::unique_lock<std::mutex> lock(mutex_);
    
    # 计算新的safe time
    auto safe_time = DoGetSafeTime(HybridTime::kMin,       // min_allowed
                                   CoarseTimePoint::max(), // deadline
                                   ht_lease,
                                   &lock);
    # safe_time一定不小于当前的propagated_safe_time_                
    CHECK_GE(safe_time, propagated_safe_time_)
        << InvariantViolationLogPrefix()
        << "ht_lease: " << ht_lease;
        
    # 更新propagated_safe_time_
    propagated_safe_time_ = safe_time;
  }
  
  cond_.notify_all();
}

# MvccManager::DoGetSafeTime的作用是根据ht_lease和MvccManager::queue_等
# 信息获取一个不小于min_allowed的HybridTime，为了获取满足条件的HybridTime，
# 可能需要等待，参数lock作为条件等待中的锁，而deadline限定了条件等待的
# 最长时间
HybridTime MvccManager::DoGetSafeTime(const HybridTime min_allowed,
                                      const CoarseTimePoint deadline,
                                      const FixedHybridTimeLease& ht_lease,
                                      std::unique_lock<std::mutex>* lock) const {
  DCHECK_ONLY_NOTNULL(lock);
  CHECK(ht_lease.lease.is_valid()) << InvariantViolationLogPrefix();
  CHECK_LE(min_allowed, ht_lease.lease) << InvariantViolationLogPrefix();

  const bool has_lease = !ht_lease.empty();
  if (has_lease) {
    # max_ht_lease_seen_表示当前看到的最大的lease expiration HybridTime
    max_ht_lease_seen_ = std::max(ht_lease.lease, max_ht_lease_seen_);
  }

  HybridTime result;
  SafeTimeSource source = SafeTimeSource::kUnknown;
  
  # 这是采用了一个匿名函数(Lambda表达式)，返回的predicate是bool类型，这个predicate
  # 将在下面的conditional wait中用到，代码中的&result, &source表示采用引用传递，并
  # 不是取地址的意思
  #
  # 这个匿名函数的作用是根据ht_lease和MvccManager::queue_等信息获取一个不小于min_allowed
  # 的HybridTime
  auto predicate = [this, &result, &source, min_allowed, time = ht_lease.time, has_lease] {
    if (queue_.empty()) {
      result = time.is_valid() ? std::max(max_safe_time_returned_with_lease_.safe_time, time)
                               : clock_->Now();
      source = SafeTimeSource::kNow;
    } else {
      # MvccManager::queue_不为空的情况下，采用MvccManager::queue_队列头部的元素对应的
      # HybridTime减去1作为result
      result = queue_.front().Decremented();
      source = SafeTimeSource::kNextInQueue;
    }

    # result应该不超过max_ht_lease_seen_
    if (has_lease && result > max_ht_lease_seen_) {
      result = max_ht_lease_seen_;
      source = SafeTimeSource::kHybridTimeLease;
    }

    # result应该不小于last_replicated_
    result = std::max(result, last_replicated_);

    # 返回result是否大于min_allowed
    return result >= min_allowed;
  };

  if (deadline == CoarseTimePoint::max()) {
    # 如果deadline被设置为CoarseTimePoint::max()，则表示conditional wait没有超时时间，
    # 也就是说会一直等待，直到predicate变为true(在当前上下文中，就是直到获取一个不小
    # 于min_allowed的result)
    #
    # 在当前上下文中，deadline确实被设置为CoarseTimePoint::max()，见
    # MvccManager::UpdatePropagatedSafeTimeOnLeader
    cond_.wait(*lock, predicate);
  } else if (!cond_.wait_until(*lock, deadline, predicate)) {
    # 这里会进行conditional wait，直到deadline过期，或者predicate为true为止
    return HybridTime::kInvalid;
  }

  # 获取到的result一定大于之前设定的max_safe_time_returned_with_lease_.safe_time或者
  # max_safe_time_returned_without_lease_.safe_time
  auto enforced_min_time = has_lease ? max_safe_time_returned_with_lease_.safe_time
                                     : max_safe_time_returned_without_lease_.safe_time;
  CHECK_GE(result, enforced_min_time);

  # 更新max_safe_time_returned_with_lease_或者max_safe_time_returned_without_lease_
  if (has_lease) {
    max_safe_time_returned_with_lease_ = { result, source };
  } else {
    max_safe_time_returned_without_lease_ = { result, source };
  }
  
  # 返回result
  return result;
}
```

### 设置操作对应的read time
如果DocWriteOperation操作通过了冲突检测，但是DocWriteOperation::read_time_尚未被设置，则；
- 先获取当前的safe time
- 然后根据safe time获取当前操作对应的read time
```
DocWriteOperation::TransactionalConflictsResolved
AbstractTablet::SafeTime
Tablet::DoGetSafeTime
MvccManager::SafeTime(
      HybridTime min_allowed, CoarseTimePoint deadline, const FixedHybridTimeLease& ht_lease)
MvccManager::DoGetSafeTime      
     
class DocWriteOperation : public std::enable_shared_from_this<DocWriteOperation> {      
  void TransactionalConflictsResolved() {
    if (!read_time_) {
      # 获取当前的safe time
      auto safe_time = tablet_.SafeTime(RequireLease::kTrue);
      
      # 生成read_time_
      read_time_ = ReadHybridTime::FromHybridTimeRange(
          {safe_time, tablet_.clock()->NowRange().second});
    } else if (prepare_result_.need_read_snapshot &&
               isolation_level_ == IsolationLevel::SERIALIZABLE_ISOLATION) {
      ...
    }

    Complete();
  }     
}

# 这个逻辑还没看懂！！！
HybridTime Tablet::DoGetSafeTime(
    tablet::RequireLease require_lease, HybridTime min_allowed, CoarseTimePoint deadline) const {
  if (!require_lease) {
    return mvcc_.SafeTimeForFollower(min_allowed, deadline);
  }
  FixedHybridTimeLease ht_lease;
  if (require_lease && ht_lease_provider_) {
    // min_allowed could contain non zero logical part, so we add one microsecond to be sure that
    // the resulting ht_lease is at least min_allowed.
    auto min_allowed_lease = min_allowed.GetPhysicalValueMicros();
    if (min_allowed.GetLogicalValue()) {
      ++min_allowed_lease;
    }
    // This will block until a leader lease reaches the given value or a timeout occurs.
    ht_lease = ht_lease_provider_(min_allowed_lease, deadline);
    if (!ht_lease.lease.is_valid()) {
      // This could happen in case of timeout.
      return HybridTime::kInvalid;
    }
  }
  if (min_allowed > ht_lease.lease) {
    LOG_WITH_PREFIX(DFATAL)
        << "Read request hybrid time after leader lease: " << min_allowed << ", " << ht_lease;
    return HybridTime::kInvalid;
  }
  
  # 调用MvccManager::DoGetSafeTime，已经在前面讲解
  # MvccManager::UpdatePropagatedSafeTimeOnLeader过程中讲解过了
  return mvcc_.SafeTime(min_allowed, deadline, ht_lease);
}
```

### propagate safe time from leader to followers for follower-side reads
YugaByte支持在raft follower节点上读取数据，但是读取到的数据可能是过时的数据。跟在leader上读取数据类似，在follower上读取数据也需要获取safe read time。但是在leader上获取safe read time的方法在follower上不再适用，因此YugaByte中会“propagate the latest safe time from leaders to followers on AppendEntries RPCs”。

实现上，YugaByte在每次向raft peer发送请求(Peer::SendNextRequest)的时候，都会获取当前的safe time(MvccManager::DoGetSafeTime)，并伴随着请求一同发送给raft peer。
```
yb::consensus::Peer::SignalRequest
yb::consensus::Peer::SendNextRequest
PeerMessageQueue::RequestForPeer
TabletPeer::PreparePeerRequest
MvccManager::SafeTime(const FixedHybridTimeLease& ht_lease)
MvccManager::DoGetSafeTime

void Peer::SendNextRequest(RequestTriggerMode trigger_mode) {
  Status s = queue_->RequestForPeer(
      peer_pb_.permanent_uuid(), &request_, &msgs_holder, &needs_remote_bootstrap,
      &member_type, &last_exchange_successful);
      
  ...
  
  # RpcPeerProxy::UpdateAsync
  proxy_->UpdateAsync(&request_, trigger_mode, &response_, &controller_,
                      std::bind(&Peer::ProcessResponse, retain_self));
}

Status PeerMessageQueue::RequestForPeer(const string& uuid,
                                        ConsensusRequestPB* request,
                                        ReplicateMsgsHolder* msgs_holder,
                                        bool* needs_remote_bootstrap,
                                        RaftPeerPB::MemberType* member_type,
                                        bool* last_exchange_successful) {
  HybridTime propagated_safe_time;
  
  ...
  
  if (context_) {
    propagated_safe_time = context_->PreparePeerRequest();
  }                                      
  
  ...
  
  if (propagated_safe_time && !result->have_more_messages && to_index == 0) {
    // Get the current local safe time on the leader and propagate it to the follower.
    request->set_propagated_safe_time(propagated_safe_time.ToUint64());
  } else {
    request->clear_propagated_safe_time();
  }  
  
  ...
}          

HybridTime TabletPeer::PreparePeerRequest() {
  ...

  auto ht_lease = HybridTimeLease(/* min_allowed */ 0, /* deadline */ CoarseTimePoint::max());
  if (!ht_lease.lease) {
    return HybridTime::kInvalid;
  }
  
  # 调用MvccManager::DoGetSafeTime，已经在前面讲解
  # MvccManager::UpdatePropagatedSafeTimeOnLeader过程中讲解过了
  return tablet_->mvcc_manager()->SafeTime(ht_lease);
}

void RpcPeerProxy::UpdateAsync(const ConsensusRequestPB* request,
                               RequestTriggerMode trigger_mode,
                               ConsensusResponsePB* response,
                               rpc::RpcController* controller,
                               const rpc::ResponseCallback& callback) {
  controller->set_timeout(MonoDelta::FromMilliseconds(FLAGS_consensus_rpc_timeout_ms));
  # ConsensusServiceProxy::UpdateConsensusAsync
  consensus_proxy_->UpdateConsensusAsync(*request, response, controller, callback);
}

void ConsensusServiceProxy::UpdateConsensusAsync(const ConsensusRequestPB &req,
                     ConsensusResponsePB *resp, ::yb::rpc::RpcController *controller,
                     ::yb::rpc::ResponseCallback callback) {
  static ::yb::rpc::RemoteMethod method("yb.consensus.ConsensusService", "UpdateConsensus");
  # Proxy::AsyncRequest -> Proxy::DoAsyncRequest
  proxy_->AsyncRequest(&method, req, resp, controller, std::move(callback));
}
```
