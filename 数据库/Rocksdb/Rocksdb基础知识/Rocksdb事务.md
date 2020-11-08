# Outline
[toc]

# Transactions
Rocksdb中的TransactionDB或者OptimisticTransactionDB支持事务，为事务提供了BEGIN/COMMIT/ROLLBACK，允许用户并发更新数据。Rocksdb同时支持乐观的和悲观的并发控制机制。

## TransactionDB
使用TransactionDB的情况下，Rocksdb内部采用锁机制来进行冲突检测。如果在某个key上无法获取锁，则当前操作返回错误。在事务被Commit的时候，只要数据库能够写入，Rocksdb就确保写入成功。

相较于OptimisticTransactionDB而言，TransactionDB更适用于具有较高并发的负载。但是因为TransactionDB采用锁机制进行冲突检测，所以具有一定的所开销。在TransactionDBOptions中可以对锁超时等进行调优。

```
    TransactionDB* txn_db;
    Status s = TransactionDB::Open(options, path, &txn_db);
    
    Transaction* txn = txn_db->BeginTransaction(write_options, txn_options);
    s = txn->Put(“key”, “value”);
    s = txn->Delete(“key2”);
    s = txn->Merge(“key3”, “value”);
    s = txn->Commit();
    delete txn;
```

默认的写策略是WriteCommitted，也支持WritePrepared和WriteUnprepared。关于写策略，参考[这里](https://github.com/facebook/rocksdb/wiki/WritePrepared-Transactions)或者[这里](http://note.youdao.com/noteshare?id=c2fab9ebda1bc03e421be12526fdc0f3&sub=CC4D6554FCF548F4BFB950E8DE295AF0)。

## OptimisticTransactionDB

针对并发不是非常高的情形（比如只会偶尔发生写冲突），OptimisticTransactionDB提供了一种轻量级的乐观并发控制机制。

OptimisticTransactionDB在commit的时候，检查是否有其它写操作修改了当前事务正在写的keys，以此来辨别是否存在冲突。如果存在写冲突，或者无法确定是否存在写冲突，则当前事务的commit将返回错误并且当前的写不会被写入。

相较于TransactionDB而言，OptimisticTransactionDB更适合于那些具有较多的非事务写（non-transactional writes）和少量事务写（transactional writes）的负载。

```
    DB* db;
    OptimisticTransactionDB* txn_db;
    
    Status s = OptimisticTransactionDB::Open(options, path, &txn_db);
    db = txn_db->GetBaseDB();
    
    OptimisticTransaction* txn = txn_db->BeginTransaction(write_options, txn_options);
    txn->Put(“key”, “value”);
    txn->Delete(“key2”);
    txn->Merge(“key3”, “value”);
    s = txn->Commit();
    delete txn;
```

## Reading from a Transaction

Transaction支持从当前事务中读取尚未提交的更新：

```
    db->Put(write_options, “a”, “old”);
    db->Put(write_options, “b”, “old”);
    txn->Put(“a”, “new”);
    
    vector<string> values;
    vector<Status> results = txn->MultiGet(read_options, {“a”, “b”}, &values);
    //  The value returned for key “a” will be “new” since it was written by this transaction.
    //  The value returned for key “b” will be “old” since it is unchanged in this transaction.
```

也可以通过Transaction::GetIterator()来遍历存在于数据库中的和存在于当前事务中的keys。

## Setting a Snapshot

默认地，事务写冲突判断检查在该事务中关于某个key的第一次写之后是否存在其它的关于该key的写操作，这种隔离级别对于大多数情况就足够了。但是有时候应用可能希望在启动某个事务之后，没有关于某个key的任何的写操作，这可以通过在创建事务之后调用SetSnapshot()来实现。

默认的隔离级别:

```
    // Create a txn using either a TransactionDB or OptimisticTransactionDB
    txn = txn_db->BeginTransaction(write_options);
    
    // Write to key1 OUTSIDE of the transaction
    db->Put(write_options, “key1”, “value0”);
    
    // Write to key1 IN transaction
    s = txn->Put(“key1”, “value1”);
    s = txn->Commit();
    
    // There is no conflict since the write to key1 outside of the transaction happened before it was written in this transaction.
```

采用SetSnapshot()情况下的隔离级别:

```
    txn = txn_db->BeginTransaction(write_options);
    txn->SetSnapshot();
    
    // Write to key1 OUTSIDE of the transaction
    db->Put(write_options, “key1”, “value0”);
    
    // Write to key1 IN transaction
    s = txn->Put(“key1”, “value1”);
    s = txn->Commit();
    
    // Transaction will NOT commit since key1 was written outside of this transaction after SetSnapshot() was called (even though this write occurred before this key was written in this transaction).
```

在上面的示例中，如果采用TransactionDB，则Put()将会失败，如果采用OptimisticTransactionDB，则Commit()将会失败。

## Repeatable Read

可以通过在事务读的时候在ReadOptions中设置snapshot实现repeatable reads。

```
    read_options.snapshot = db->GetSnapshot();
    s = txn->GetForUpdate(read_options, “key1”, &value);
    …
    s = txn->GetForUpdate(read_options, “key1”, &value);
    db->ReleaseSnapshot(read_options.snapshot);
```

## Guarding against Read-Write Conflicts:

GetForUpdate()将确保没有任何的写操作修改当前事务中本次读所涉及的keys。

```
    // Start a transaction 
    txn = txn_db->BeginTransaction(write_options);
    
    // Read key1 in this transaction
    Status s = txn->GetForUpdate(read_options, “key1”, &value);
    
    // Write to key1 OUTSIDE of the transaction
    s = db->Put(write_options, “key1”, “value0”);
```

如果通过TransactionDB创建事务，则Put操作要么超时要么阻塞直到事务提交或者终止。如果通过OptimisticTransactionDB创建事务，则Put操作将会成功，但是在事务调用txn->Commit()的时候将失败。
    
```
    // Repeat the previous example but just do a Get() instead of a GetForUpdate()
    txn = txn_db->BeginTransaction(write_options);
    
    // Read key1 in this transaction
    Status s = txn->Get(read_options, “key1”, &value);
    
    // Write to key1 OUTSIDE of the transaction
    s = db->Put(write_options, “key1”, “value0”);
    
    // No conflict since transactions only do conflict checking for keys read using GetForUpdate().
    s = txn->Commit();
```

上面的示例中没有冲突，因为在读的时候只有当使用了GetForUpdate()的情况下事务才会进行冲突检测。

## Save Points

如果采用了SavePoints，事务也支持部分回滚。

```
    s = txn->Put("A", "a");
    txn->SetSavePoint();
    s = txn->Put("B", "b");
    txn->RollbackToSavePoint()
    s = txn->Commit()
    // Since RollbackToSavePoint() was called, this transaction will only write key A and not write key B.
```

# Under the hood

简单的看下事务是如何实现的。

## Read Snapshot

Each update in RocksDB is done by inserting an entry tagged with a monotonically increasing sequence number. Assigning a seq to read_options.snapshot will be used by (transactional or non-transactional) db to read only values with seq smaller than that, i.e., read snapshots → DBImpl::GetImpl

That aside, transactions can call TransactionBaseImpl::SetSnapshot which will invoke DBImpl::GetSnapshot. It achieves two goals:

Return the current seq: transactions will use the seq (instead of the seq of its written value) to check for write-write conflicts → TransactionImpl::TryLock → TransactionImpl::ValidateSnapshot → TransactionUtil::CheckKeyForConflicts
Makes sure that values with smaller seq will not be erased by compaction jobs, etc. (snapshots_.GetAll). Such marked snapshots must be released by the callee (DBImpl::ReleaseSnapshot)

## Read-Write conflict detection

Read-Write conflicts can be prevented by escalating them to write-write conflict: doing reads via GetForUpdate (instead of Get).

## Write-Write conflict detection: pessimistic approach

Write-write conflicts are detected at the write time using a lock table.

Non-transactional updates (put, merge, delete) are internally run under a transaction. So every update is through transactions → TransactionDBImpl::Put

Every update acquires a lock beforehand → TransactionImpl::TryLock TransactionLockMgr::TryLock has only 16 locks per column family → size_t num_stripes = 16

Commit simply writes the write batch to WAL as well as to Memtable by calling DBImpl::Write → TransactionImpl::Commit

To support distributed transactions, the clients can call Prepare after performing the writes. It writes the value to WAL but not to the MemTable, which allows recovery in the case of machine crash → TransactionImpl::Prepare If Prepare is invoked, Commit writes a commit marker to the WAL and writes values to MemTable. This is done by calling MarkWalTerminationPoint() before adding value to the write batch.

## Write-Write conflict detection: optimistic approach

Write-write conflicts are detected using seq of the latest values at the commit time.

Each update adds the key to an in-memory vector → TransactionDBImpl::Put and OptimisticTransactionImpl::TryLock

Commit links OptimisticTransactionImpl::CheckTransactionForConflicts as a callback to the write batch → OptimisticTransactionImpl::Commit which will be invoked in DBImpl::WriteImpl via write->CheckCallback

The conflict detection logic is implemented at TransactionUtil::CheckKeysForConflicts

only checks for conflicts against keys present in memory and fails otherwise.
conflict detection is done by checking the latest seq of each key (DBImpl::GetLatestSequenceForKey) against the seq used for writing it.


# 源码分析
## 待续...
## 待续...
## 待续...


# 参考

[RocksDB事务实现TransactionDB分析](https://yq.aliyun.com/articles/257424?spm=5176.10695662.1996646101.searchclickresult.698075a3hXT2YW)

