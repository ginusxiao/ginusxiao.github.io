# 提纲
[toc]

## 说明
YSQL分布式事务的执行和单行事务的执行流程大致是类似的，但是也存在着一些异同，本文主要列举出来写流程的执行框架，并且列举出所有存在差异的地方。

## TServer接收到写请求
主要执行如下步骤：
- 查找tablet leader
- 生成一个WriteOperationState，其中会关联RPC request和response
- 在WriteOperationState中设置WriteOperationCompletionCallback，该回调会在请求执行完成时被调用，请求执行完成，可能是请求执行成功，也可能是请求执行失败
- 如果是YSQL system catalog相关的操作，则设置WriteOperationState::force_txn_path_为true，表示该操作必须走事务流程
- 将WriteOperationState提交给对应的tablet异步的执行，提交给tablet时会带上当前tablet的term信息
```
TabletServiceImpl::Write
    - LookupLeaderTabletOrRespond
    - auto operation_state = std::make_unique<WriteOperationState>(tablet.peer->tablet(), req, resp);
    - operation_state->set_completion_callback(
        std::make_unique<WriteOperationCompletionCallback>(...))
    - AdjustYsqlOperationTransactionality
    - tablet.peer->WriteAsync(std::move(operation_state), tablet.leader_term, context_ptr->GetClientDeadline())
```

## TabletPeer将写请求提交给Tablet
执行以下步骤：
- 检查当前TabletPeer是否是leader，如果不是，则退出
- 检查当前TabletPeer是否处于running状态，如果不是，则退出
- 生成WriteOperation，其中包含WriteOperationState和tablet leader term等信息
- 提交给Tablet来执行
```
TabletPeer::WriteAsync
    - auto operation = std::make_unique<WriteOperation>(
      std::move(state), term, std::move(preparing_token), deadline, this);
    - tablet_->AcquireLocksAndPerformDocOperations(std::move(operation))
```

## Tablet上执行写操作
执行以下步骤：
- 填充WriteOperation::doc_ops_，这是一个关于DocOperation的数组，填充过程如下：
    - 为当前的WriteOperation创建一个TransactionOperationContext
        - 其中包括transaction id和 TransactionParticipant
        - 对于分布式事务，transaction id已经包含在WriteRequestPB中
        - 对于非分布式事务，如果它操作的table是transactional的，或者该table是ysql system catalog table，则会随机分配一个transaction id
        - TransactionParticipant则在每个tablet上都有一个
    - WriteRequestPB会包含一个关于PgsqlWriteRequestPB的数组，每一个PgsqlWriteRequestPB都会生成一个对应的PgsqlWriteOperation保存到WriteOperation::doc_ops_数组中
        - PgsqlWriteOperation中会关联该table的schema，以及前面创建的TransactionOperationContext
        - PgsqlWriteOperation中会关联对应的PgsqlWriteRequestPB
        - PgsqlWriteOperation中会关联对应的DocKey
            - 这个DocKey可能从PgsqlWriteRequestPB中的ybctid中解析出来，也可能通过PgsqlWriteRequestPB中的partition column信息和range column信息生成
            - 所关联的DocKey还会被编码成encoded_doc_key
- 生成一个DocWriteOperation
    - 它会关联到对应的WriteOperation
    - 它会关联一个DocWriteOperationCallback，该回调会在将操作提交给raft之前被回调
- 获取当前WriteOperation需要申请的锁的信息，记录在LockBatchEntries中(一个数组)，过程如下：
    - 遍历当前的WriteOperation所涉及的所有的PgsqlWriteOperation，对每个PgsqlWriteOperation执行如下：
        - 获取该PgsqlWriteOperation的完整的DocKey
        - 获取该PgsqlWriteOperation在申请锁的时候应该使用哪种隔离级别(这里所使用的的隔离级别不会影响WriteOperation的真实的隔离级别，只用于决定获取什么类型的锁)
            - 如果当前WriteOperation的隔离级别是snapshot或者serializable，则直接使用指定的隔离级别
            - 如果当前WriteOperation的隔离级别是non-transactional(也就是单行事务?)，则根据是否需要读取snapshot来确定是使用哪种隔离级别
                - 如果是insert/update/delete操作，则使用snapshot隔离级别
                - 如果是upsert操作，则使用serializable隔离级别
        - 根据是否指定了行锁，采用哪种隔离级别来申请锁，当前操作类型，来共同决定申请哪种类型的strong锁
            - 如果指定了行锁
                - 如果行锁类型是ROW_MARK_EXCLUSIVE或者ROW_MARK_NOKEYEXCLUSIVE，则使用{IntentType::kStrongRead, IntentType::kStrongWrite}
                - 如果行锁类型是ROW_MARK_SHARE或者ROW_MARK_KEYSHARE，则使用{IntentType::kStrongRead}
            - 否则：
                - 如果是SNAPSHOT_ISOLATION，则使用{IntentType::kStrongRead, IntentType::kStrongWrite}
                - 如果是SERIALIZABLE_ISOLATION，则进一步根据操作类型来决定申请哪种类型的strong锁
                    - 如果是read类型的操作，则使用{IntentType::kStrongRead}
                    - 否则，是write类型的操作，则使用{IntentType::kStrongWrite}
                - 如果是NON_TRANSACTIONAL，则不加锁
        - 如果是SERIALIZABLE_ISOLATION，且是write类型的操作，且需要读取snapshot，也就是说是serializable的insert/update/delete，则调整它所需要申请的strong锁类型为：{IntentType::kStrongRead, IntentType::kStrongWrite}
        - 记录在当前WriteOperation所涉及的表上加weak锁，记录到LockBatchEntries中
        - 记录在完整的DocKey上加strong锁，记录到LockBatchEntries中
        - 记录在DocKey的所有前缀部分分别加weak锁，记录到LockBatchEntries中
    - 遍历当前WriteOperation所涉及的所有的Read Pairs，对于Read pairs中的每一个KV pair执行如下：
        - 记录在当前KV pair中的key的完整路径上加{IntentType::kStrongRead}锁，记录到LockBatchEntries中
        - 记录在当前KV pair中的key的前缀路径上加{IntentType::kWeakRead}锁，记录到LockBatchEntries中
- 对LockBatchEntries中所记录的所有的key和它对应的锁信息，进行合并，相同的key，只在LockBatchEntries占用一个entry，而它们需要申请的锁进行合并(并集)
- 按照LockBatchEntries中的所有的key和它对应的锁信息，进行加锁
    - 当前tablet中已经持有的锁的信息在SharedLockManager中维护
    - 如果在加锁过程中，发现存在锁冲突，则等待，直到超时发生或者被锁的其它持有者唤醒
    - 如果不存在锁冲突，则直接加锁，并且将加锁信息记录在SharedLockManager中
- 至此，已经加上锁了，但是锁冲突检测的时候，只会检测当前操作的DocKey和其它操作的DocKey之间是否存在冲突，而没有关注Intent部分是否存在冲突，而且YugaByte会在数据成功写入Rocksdb(对于分布式事务是成功写入intent DB，对于单行事务则是成功写入regular DB)只会就立即释放持有的锁，所以需要检查和Intent记录之间是否存在冲突
- 检查和其它操作之间是否存在冲突
    - 如果隔离级别是NON_TRANSACTIONAL，则：
        - 生成OperationConflictResolverContext
        - 生成ConflictResolver，它会关联到OperationConflictResolverContext，ConflictResolver提供了检查和解决冲突的框架，而OperationConflictResolverContext提供了检查和解决冲突过程中的一些具体实现
        - 借助于ConflictResolver来检查是否存在冲突
            - 从intent DB中获取那些和当前WriteOperation之间存在冲突的所有的记录
                - 遍历当前的WriteOperation所涉及的所有的PgsqlWriteOperation，对每个PgsqlWriteOperation执行如下：
                    - 获取该PgsqlWriteOperation会使用哪种类型的strong锁
                    - 获取该PgsqlWriteOperation中所涉及的所有的Columns对应的DocPaths，每个Column对应的DocPath为：DocKey + Column_id，如果涉及n个Column，则返回的DocPaths中包括n个DocPath
                    - 遍历为当前PgsqlWriteOperation获取到的所有的DocPaths，对于每个DocPath处理如下：
                        - 设置在完整的DocPath上加strong锁，读取intent DB中以该完整DocPath为前缀的所有intent记录所持有的锁和当前完整的DocPath上需要加的strong锁之间是否存在冲突
                            - 如果存在冲突，则将对应的事务的transaction id添加到冲突列表中
                        - 设置在DocPath的所有的前缀部分加weak锁，对于DocPath的每个前缀部分(记为DocPath_prefix)，读取intent DB中以DocPath_prefix为前缀的所有intent记录所持有的锁和DocPath_prefix上需要加的weak锁之间是否存在冲突
                            - 如果存在冲突，则将对应的事务的transaction id添加到冲突列表中
        - 检查冲突列表是否为空
            - 如果冲突列表不为空，则借助ConflictResolver来解决冲突
                - 从冲突列表中移除所有已经在本地commit的事务对应的transaction id
                - 如果冲突列表为空
                    - 冲突解决完毕，调用冲突解决完毕之后的回调
                - 否则：
                    - 遍历冲突列表中所有的transactions，逐一向status tablet 发送消息查询事务的状态
                        - 查询事务的状态，可能成功，也可能失败
                            - 如果成功，则设置相应transaction在本地的状态
                            - 如果失败，则标记相应transaction失败
                    - 检查冲突列表中所有的transactions，移除不再冲突的transactions
                        - 如果当前冲突的transaction被标记为失败，则会abort当前操作(不是当前冲突的transaction)
                        - 如果当前冲突的transaction状态是committed，则将当前冲突的transaction从冲突列表中移除
                        - 如果当前冲突的transaction状态是aborted，则将当前冲突的transaction从冲突列表中移除
                    - 检查冲突列表是否为空
                        - 如果冲突列表仍不为空
                            - 则针对每个冲突的transaction主动发送请求给status tablet来abort之
                                - 如果rpc成功，响应中会附带transaction的状态信息，会根据响应中的状态来设置本地记录的关于该transaction的状态
                                - 如果rpc失败，则标记相应transaction失败
                            - 再次进入“检查冲突列表中所有的transactions，移除不再冲突的transactions”
                            - 检查冲突列表是否为空
                                - 如果冲突列表仍不为空，则再次进入“如果冲突列表不为空，则借助ConflictResolver来解决冲突”
                                - 如果冲突列表为空，则冲突解决完毕，调用冲突解决完毕之后的回调
                        - 如果冲突列表为空
                            - 冲突解决完毕，调用冲突解决完毕之后的回调
            - 如果冲突列表为空
                - 调用冲突解决完毕之后的回调
    - 如果隔离级别是SNAPSHOT_ISOLATION，则：
        - 生成TransactionConflictResolverContext
        - 生成ConflictResolver，它会关联到TransactionConflictResolverContext，ConflictResolver提供了检查和解决冲突的框架，而TransactionConflictResolverContext提供了检查和解决冲突过程中的一些具体实现
        - 检测冲突和解决冲突的过程跟NON_TRANSACTIONAL的过程类似，只不过要注意以下几点差异：
            - “从intent DB中获取那些和当前WriteOperation之间存在冲突的所有的记录”过程不同：
                - 在NON_TRANSACTIONAL中，只对当前的WriteOperation所涉及的所有的PgsqlWriteOperation进行处理，但是在SNAPSHOT_ISOLATION中，还需要对所有涉及的read pairs进行类似处理
                - 如果设置了有效的read_time，则对于所有需要加strong锁的DocPath，需要读取regular DB，找出regular DB中所有以该DocPath作为key的前缀或者等于该DocPath的key对应的记录，要比较该记录的commit时间戳是否是大于read_time，如果是，则认为该记录对应的事务是在当前WriteOperation之后提交的，所以必须让当前WriteOperation重新执行，会返回TryAgain
            - “如果冲突列表不为空，则借助ConflictResolver来解决冲突”过程不同：
                - “从冲突列表中移除所有已经在本地commit的事务对应的transaction id”过程不同：
                    - 移除之前，还必须确保已经在本地commit的事务对应的commit_time小于当前WriteOperation的read_time，否则当前WriteOperation必须重新执行，返回TryAgain
                    - 在abort其它与之冲突的transactions之前，会检查当前WriteOperation所在的transaction与冲突的transaction之间的优先级，如果当前WriteOperation所在的transaction的优先级较低，则当前WriteOperation必须重新执行，返回TryAgain
    - 如果隔离级别是SERIALIZABLE_ISOLATION，则：
        - 处理过程跟SNAPSHOT_ISOLATION隔离级别类似
        - 但是对于需要读取snapshot的情况，要注意以下几点差异：
            - 额外会将WriteOperation所涉及的所有的PgsqlWriteOperation对应的DocKey部分添加到read pairs中
- 当没有冲突或者冲突解决完毕之后，会调用冲突解决完毕之后的回调ResolutionCallback
    - 对于NON_TRANSACTIONAL
        - 更新当前tablet上的时钟
        - 
    - 对于SNAPSHOT_ISOLATION或者SERIALIZABLE_ISOLATION
        - 设置read_time
