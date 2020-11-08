# 提纲
[toc]

## MOT和Opengauss交互的纽带 - FDW(Foreign Data Wrapper)
FDW是PostgreSQL中一项非常有用的技术，通过它可以将PostgreSQL变成一个通用的SQL引擎，使得用户可以通过SQL访问存储在PostgreSQL之外的数据。

### FDW的通用用法
使用FDW的核心就在于使用外部表（FOREIGN TABLE）。尽管面向不同数据源的FDW实现各有不同，但是创建不同数据源的外部表的方法都是一样的，分别需要在PG端依次创建以下几个数据库对象：
1. 向PG安装某个数据源的FDW扩展；
```
CREATE EXTENSION extension_name
```

2. 使用CREATE SERVER语句创建该数据源的服务器对象；
```
CREATE SERVER server_name 
    FOREIGN DATA WRAPPER fdw_name
    OPTIONS ( option 'value' [, ... ] )
```

3. 使用CREATE USER MAPPING语句创建外部数据源用户与PG用户的映射关系（这一步是可选的。比如外部数据源根本没有权限控制时，也就无需创建USER MAPPING了）；
```
CREATE USER MAPPING FOR { user_name | USER | CURRENT_USER | PUBLIC }
    SERVER server_name
    OPTIONS ( option 'value' [ , ... ] )
```

4. 使用CREATE FOREIGN TABLE语句创建外部表；
```
CREATE FOREIGN TABLE table_name 
    (attribute_name type, ...)
    SERVER server_name
    OPTIONS ( option 'value' [, ... ] )
```

### FDW回调函数
一个FDW实现的核心就是实现一组回调函数，用于在查询外部表对象的执行过程中将运行逻辑切换至自定义的扩展代码中，进而遵照PG的内部机制实现对外部数据源的访问。

FDW在FdwRoutine中定义了一系列回调函数：
```
typedef struct FdwRoutine {
    NodeTag type;

    /* Functions for scanning foreign tables */
    GetForeignRelSize_function GetForeignRelSize;
    GetForeignPaths_function GetForeignPaths;
    GetForeignPlan_function GetForeignPlan;
    BeginForeignScan_function BeginForeignScan;
    IterateForeignScan_function IterateForeignScan;
    ReScanForeignScan_function ReScanForeignScan;
    EndForeignScan_function EndForeignScan;

    /*
     * These functions are optional.  Set the pointer to NULL for any that are
     * not provided.
     */

    /* Functions for updating foreign tables */
    AddForeignUpdateTargets_function AddForeignUpdateTargets;
    PlanForeignModify_function PlanForeignModify;
    BeginForeignModify_function BeginForeignModify;
    ExecForeignInsert_function ExecForeignInsert;
    ExecForeignUpdate_function ExecForeignUpdate;
    ExecForeignDelete_function ExecForeignDelete;
    EndForeignModify_function EndForeignModify;
    IsForeignRelUpdatable_function IsForeignRelUpdatable;

    /* Support functions for EXPLAIN */
    ExplainForeignScan_function ExplainForeignScan;
    ExplainForeignModify_function ExplainForeignModify;

    /* @hdfs Support functions for ANALYZE */
    AnalyzeForeignTable_function AnalyzeForeignTable;

    /* @hdfs Support function for sampling */
    AcquireSampleRowsFunc AcquireSampleRows;

    /* @hdfs Support Vector Interface */
    VecIterateForeignScan_function VecIterateForeignScan;

    /* @hdfs This function is uesed to return the type of FDW */
    GetFdwType_function GetFdwType;

    /* @hdfs Validate table definition */
    ValidateTableDef_function ValidateTableDef;

    /* @hdfs
     * Partition foreign table process: create/drop
     */
    PartitionTblProcess_function PartitionTblProcess;

    /* @hdfs Runtime dynamic predicate push down like bloom filter. */
    BuildRuntimePredicate_function BuildRuntimePredicate;

    /* Support truncate for foreign table */
    TruncateForeignTable_function TruncateForeignTable;

    /* Support vacuum */
    VacuumForeignTable_function VacuumForeignTable;

    /* Get table/index memory size */
    GetForeignRelationMemSize_function GetForeignRelationMemSize;

    /* Get memory size */
    GetForeignMemSize_function GetForeignMemSize;

    /* Get all session memory size */
    GetForeignSessionMemSize_function GetForeignSessionMemSize;

    /* Notify engine that envelope configuration changed */
    NotifyForeignConfigChange_function NotifyForeignConfigChange;
} FdwRoutine;
```

#### 必需接口
无论外部数据源自身能力如何，这些接口中，有7个接口是实现通过外部表对象访问该数据源的必需接口，它们会在Optimizer和Executor阶段介入。它们的接口定义如下：
```
# 获取外部表的预估大小，在plan开始的时候被调用
typedef void (*GetForeignRelSize_function) (PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid);

# 获取各种可能的访问路径(Access Paths，包括这种plan处理方式和成本预估等)，在plan过程中被调用
typedef void (*GetForeignPaths_function) (PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid);

# 从多个plan path中选出一个最佳路径之后，从这个最佳路径中恢复出相应的SQL语句，
# 在plan结束的时候被调用
typedef ForeignScan *(*GetForeignPlan_function) (PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid, ForeignPath *best_path, List *tlist, List *scan_clauses, Plan *outer_plan);

# 执行foreign scan前的准备工作，在Executor启动阶段被调用
typedef void (*BeginForeignScan_function) (ForeignScanState *node, int eflags);

# 从外部表中获取一行数据，返回的数据包含在TupleTableSlot中，如果没有找到对应的行，
# 则返回null
typedef TupleTableSlot *(*IterateForeignScan_function) (ForeignScanState *node);

# 重新开始scan
typedef void (*ReScanForeignScan_function) (ForeignScanState *node);

# 结束scan过程，释放资源
typedef void (*EndForeignScan_function) (ForeignScanState *node);
```

#### DML操作相关接口
除了上述7个必须接口，FDW为DML操作提供了以下接口：
```
# 用于处理update和delete过程中，添加额外的columns到待获取的columns列表中，在plan阶段调用
typedef void (*AddForeignUpdateTargets_function)(Query* parsetree, RangeTblEntry* target_rte, Relation target_relation);

# 执行额外的plan相关的操作，将会产生一些FDW私有的信息
typedef List* (*PlanForeignModify_function)(
    PlannerInfo* root, ModifyTable* plan, Index resultRelation, int subplan_index);

# 用于在更新table之前执行一些必要的初始化操作，在executor启动阶段执行
typedef void (*BeginForeignModify_function)(
    ModifyTableState* mtstate, ResultRelInfo* rinfo, List* fdw_private, int subplan_index, int eflags);

# 插入一个tuple到外部表中
typedef TupleTableSlot* (*ExecForeignInsert_function)(
    EState* estate, ResultRelInfo* rinfo, TupleTableSlot* slot, TupleTableSlot* planSlot);

# 更新外部表中的某个tuple
typedef TupleTableSlot* (*ExecForeignUpdate_function)(
    EState* estate, ResultRelInfo* rinfo, TupleTableSlot* slot, TupleTableSlot* planSlot);

# 从外部表中删除一个tuple
typedef TupleTableSlot* (*ExecForeignDelete_function)(
    EState* estate, ResultRelInfo* rinfo, TupleTableSlot* slot, TupleTableSlot* planSlot);

# 在更新table之后释放相关的资源
typedef void (*EndForeignModify_function)(EState* estate, ResultRelInfo* rinfo);

# 检查外部表支持哪些更新操作
typedef int (*IsForeignRelUpdatable_function)(Relation rel);

# explain相关
typedef void (*ExplainForeignModify_function)(
    ModifyTableState* mtstate, ResultRelInfo* rinfo, List* fdw_private, int subplan_index, struct ExplainState* es);
```

### 事务/子事务回调
PostgreSQL还为动态加载的模块提供了注册事务相关的回调的方法，以及注册子事务相关的回调的方法。这些回调多是在执行事务/子事务控制命令的时候被触发的。事务控制命令包括BEGIN, END, ROLLBACK, PREPARE TRANSACTION, COMMIT PREPARED, ROLLBACK PREPARED等，其中PREPARE TRANSACTION, COMMIT PREPARED, ROLLBACK PREPARED与2PC事务有关。子事务控制命令包括SAVEPOINT, RELEASE SAVEPOINT, ROLLBACK TO SAVEPOINT等。
```
void RegisterXactCallback(XactCallback callback, void* arg)
typedef void (*XactCallback)(XactEvent event, void* arg);

void RegisterSubXactCallback(SubXactCallback callback, void* arg)
typedef void (*SubXactCallback)(SubXactEvent event, SubTransactionId mySubid, SubTransactionId parentSubid, void* arg);
```
这些回调中会根据事务/子事务控制命令对应的事件，分别进行处理。各命令和对应的事件之间的关系如下：
事务/子事务控制命令 | 事件
---|---
BEGIN | XACT_EVENT_START
END | XACT_EVENT_COMMIT + XACT_EVENT_END_TRANSACTION
ROLLBACK | XACT_EVENT_PREROLLBACK_CLEANUP + XACT_EVENT_ABORT
PREPARE TRANSACTION | XACT_EVENT_PREPARE
COMMIT PREPARED | XACT_EVENT_COMMIT_PREPARED + XACT_EVENT_END_TRANSACTION
ROLLBACK PREPARED | XACT_EVENT_ROLLBACK_PREPARED + XACT_EVENT_END_TRANSACTION
SAVEPOINT | SUBXACT_EVENT_START_SUB
RELEASE SAVEPOINT | SUBXACT_EVENT_COMMIT_SUB
ROLLBACK TO SAVEPOINT | SUBXACT_EVENT_ABORT_SUB 

## MOT FDW回调函数中与DML操作相关的接口
MOT(Opengauss Memory Optimized Table)作为外部数据源，是通过FDW的方式接入Opengauss的，所以它提供了FDW handler，即mot_fdw_handler。在mot_fdw_handler中除了实现了7个必需接口之外，还实现了DML操作相关的接口：
```
Datum mot_fdw_handler(PG_FUNCTION_ARGS)
{
    FdwRoutine* fdwroutine = makeNode(FdwRoutine);
    ...
    fdwroutine->BeginForeignModify = MOTBeginForeignModify;
    fdwroutine->ExecForeignInsert = MOTExecForeignInsert;
    fdwroutine->ExecForeignUpdate = MOTExecForeignUpdate;
    fdwroutine->ExecForeignDelete = MOTExecForeignDelete;
    fdwroutine->EndForeignModify = MOTEndForeignModify;
    ...
}
```

## MOT中注册的事务/子事务相关的回调
MOT中也分别注册了事务相关的回调MOTSubxactCallback和子事务相关的回调MOTSubxactCallback：
```
Datum mot_fdw_handler(PG_FUNCTION_ARGS)
{
    ...
    
    if (!u_sess->mot_cxt.callbacks_set) {
        GetCurrentTransactionIdIfAny();
        # 注册事务控制命令相关的回调
        RegisterXactCallback(MOTXactCallback, NULL);
        
        # 注册子事务控制命令相关的回调
        RegisterSubXactCallback(MOTSubxactCallback, NULL);
        u_sess->mot_cxt.callbacks_set = true;
    }

    ...
}
```

### 事务/子事务回调触发的时机
事务相关的回调和子事务相关的回调都是事件触发的，MOTXactCallback是事务回调的总入口，而MOTSubxactCallback是子事务回调的总入口。在它们内部都会根据触发的事件进行分别处理。

#### XACT_EVENT_START
```
# 在每条命令(包括BEGIN和END命令)执行前都会执行
StartTransactionCommand
    # 2个调用时机
    # (1) 非transaction block中一条命令执行的时候
    # (2) transaction block中执行了BEGIN命令的时候
    StartTransaction
        CallXactCallbacks(XACT_EVENT_START)
            MOTXactCallback
```

#### XACT_EVENT_COMMIT
```
# 在执行完每条命令(包括BEGIN和END命令)的时候都会执行
CommitTransactionCommand
    # 2个调用时机：
    # (1) 非transaction block中一条命令执行完毕后
    # (2) transaction block中执行了END命令后
    CommitTransaction
        CallXactCallbacks(XACT_EVENT_COMMIT)
            MOTXactCallback
```

#### XACT_EVENT_END_TRANSACTION       
```
# 在执行完每条命令(包括BEGIN和END命令)的时候都会执行
CommitTransactionCommand
    # 2个调用时机：
    # (1) 非transaction block中一条命令执行完毕后
    # (2) transaction block中执行了END命令后
    CommitTransaction
        # XACT_EVENT_COMMIT事件在XACT_EVENT_END_TRANSACTION之前发生
        CallXactCallbacks(XACT_EVENT_END_TRANSACTION)
            MOTXactCallback
            
standard_ProcessUtility
    # 2个调用时机：
    # (1) 2PC分布式事务中提交一个prepared的事务(TRANS_STMT_COMMIT_PREPARED)
    # (2) 2PC分布式事务中回滚一个prepared的事务(TRANS_STMT_ROLLBACK_PREPARED)
    FinishPreparedTransaction
        CallXactCallbacks(XACT_EVENT_END_TRANSACTION)
            MOTXactCallback
```

#### XACT_EVENT_COMMIT_START
暂未发现使用的地方

#### XACT_EVENT_ABORT
```
AbortCurrentTransaction
    AbortTransaction
        # post-rollback cleanup
        CallXactCallbacks(XACT_EVENT_ABORT)
            MOTXactCallback
```

#### XACT_EVENT_PREROLLBACK_CLEANUP
```
AbortCurrentTransaction
    AbortTransaction
        # pre-rollback cleanup
        CallXactCallbacks(XACT_EVENT_PREROLLBACK_CLEANUP)
            MOTXactCallback
```            

#### 其它
XACT_EVENT_PREPARE, XACT_EVENT_COMMIT_PREPARED和XACT_EVENT_ROLLBACK_PREPARED分别对应2PC分布式事务中prepare阶段，2PC分布式事务中commit prepared阶段和2PC分布式事务中rollback prepared阶段，暂不关注。


## MOTXactCallback中针对各事件的处理
从上面的分析可以看出，每种事件的回调，具体实现都是MOTXactCallback，那么在MOTXactCallback中是如何处理这些事件的呢？

这里着重关注XACT_EVENT_START，XACT_EVENT_COMMIT和XACT_EVENT_END_TRANSACTION事件的处理。在MOTXactCallback中对于这几个事件处理的简化逻辑如下：
```
MOTXactCallback(XactEvent event, void* arg)
{
    if (event == XACT_EVENT_START) {
        ...
        
        # transaction state初始状态为TXN_START
        if (txnState != MOT::TxnState::TXN_PREPARE) {
            # 调用的是TxnManager::StartTransaction
            mgr->StartTransaction(tid, u_sess->utils_cxt.XactIsoLevel);
        }
    } else if (event == XACT_EVENT_COMMIT) {
        ...
        
        if (txnState == MOT::TxnState::TXN_PREPARE) {
            ...
        } else {
            # 在暂不考虑coordinator的情况下，调用的是TxnManager::Commit
            rc = MOTAdaptor::Commit(tid);
        }

        # 设置transaction state为TXN_COMMIT
        mgr->SetTxnState(MOT::TxnState::TXN_COMMIT);
    } else if (event == XACT_EVENT_END_TRANSACTION) {
        ...
        
        # 在暂不考虑coordinator的情况下，调用的是TxnManager::EndTransaction
        rc = MOTAdaptor::EndTransaction(tid);
        
        # 设置transaction state为TXN_END_TRANSACTION
        mgr->SetTxnState(MOT::TxnState::TXN_END_TRANSACTION);
    }
}
```

### TxnManager::StartTransaction
主要是设置transaction id，isolation level和transaction state。
```
RC TxnManager::StartTransaction(uint64_t transactionId, int isolationLevel)
{
    m_transactionId = transactionId;
    m_isolationLevel = isolationLevel;
    m_state = TxnState::TXN_START;
    GcSessionStart();
    return RC_OK;
}
```

### TxnManager::Commit


## 参考
[科普一种可以将PG变成通用SQL引擎的技术](https://www.sohu.com/a/235705985_411876)


