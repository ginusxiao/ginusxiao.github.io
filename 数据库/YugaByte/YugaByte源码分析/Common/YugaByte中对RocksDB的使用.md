# 提纲
[toc]

## 从YugaByte的代码中查找Intent DB和Regular DB的使用的地方
### 关于Intent DB的使用
```
WriteOperation::DoReplicated
Tablet::ApplyRowOperations
Tablet::ApplyOperationState
Tablet::ApplyKeyValueRowOperations
Tablet::WriteToRocksDB
rocksdb::DB::Write

TransactionParticipant::Impl::ProcessApply
Tablet::ApplyIntents
Tablet::WriteToRocksDB
rocksdb::DB::Write

ConflictResolver::ReadIntentConflicts/TransactionConflictResolverContext::ReadConflicts
ConflictResolver::EnsureIntentIteratorCreated
CreateRocksDBIterator

PgsqlWriteOperation::ApplyInsert
DocWriteBatch::SetPrimitive
CreateIntentAwareIterator
CreateRocksDBIterator

QLWriteOperation::Apply
QLWriteOperation::HasDuplicateUniqueIndexValue
CreateIntentAwareIterator
CreateRocksDBIterator
```

### 关于Regular DB的使用
```
SysCatalogTable::Visit
Tablet::NewRowIterator
DocRowwiseIterator::Init
CreateIntentAwareIterator
CreateRocksDBIterator

PgsqlWriteOperation::ApplyInsert
DocWriteBatch::SetPrimitive
CreateIntentAwareIterator
CreateRocksDBIterator

QLWriteOperation::Apply
QLWriteOperation::HasDuplicateUniqueIndexValue
CreateIntentAwareIterator
CreateRocksDBIterator

TransactionConflictResolverContext::ReadConflicts
StrongConflictChecker::Check
CreateRocksDBIterator

OperationDriver::ReplicationFinished
OperationDriver::ApplyOperation
OperationDriver::ApplyTask
Operation::Replicated
HistoryCutoffOperation::DoReplicated
HistoryCutoffOperationState::Replicated
rocksdb::DB::Write
```

## 从RocksDB自身API的角度查看YugaByte中哪些地方使用了RocksDB
- YugaByte用到的RocksDB中与KV操作相关的API
    - Open
    - Write(参数是一个WriteBatch)
    - NewIterator
        - BoundedRocksDbIterator
        - TabletServiceAdminImpl::CountIntents -> Tablet::CountIntents
    - Flush
        - DocDBRocksDBUtil::FlushRocksDbAndWait
        - Tablet::DoCleanupIntentFiles
        - Tablet::Flush
        - Tablet::FlushIntentsDbIfNecessary
        - Tablet::ForceRocksDBCompactInTest
        - Tablet::IntentsDbFlushFilter
        - TabletSnapshots::Create
    - WaitForFlush
        - Tablet::Flush
        - Tablet::WaitForFlush
    - DeleteFile
        - Tablet::OpenKeyValueTablet -> Tablet::CleanupIntentFiles -> Tablet::DoCleanupIntentFiles
- YugaByte用到的与RocksDB实现息息相关的API(也就是说，如果存储不使用RocksDB，YugaByte就可能不用到类似这样的API)
    - CompactRange
        - TabletServiceAdminImpl::FlushTablets -> Tablet::ForceRocksDBCompactInTest -> ForceRocksDBCompact
    - CompactFiles
        - CompactionTask::Run(tools/yb-bulk_load.cc)
    - GetLatestSequenceNumber
        - TabletSnapshots::RestoreCheckpoint
- YugaByte没有使用但RocksDB提供的KV操作相关的接口
    - Put
    - Get
    - MultiGet
    - KeyMayExist
    - Delete(YugaByte中通过写入kTombstone记录来实现删除？那么这样的话，需要YugaByte自己实现Compaction？)
- RocksDB提供的其它接口，暂不关注