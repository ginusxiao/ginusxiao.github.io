# 提纲
[toc]

## 简介
在“YugaByte YSQL之事务”中已经分析了ProcessQuery在处理insert，update和delete的时候，分别会进入ExecInsert，ExecUpdate和ExecDelete，下面分析ExecInsert。

## ExecInsert执行流程
```
# 在src/postgres/src/backend/executor/nodeModifyTable.c中修改ExecInsert的实现
ExecInsert
    # YBCHeapInsert在src/postgres/src/backend/executor/ybcModifyTable.c中定义，
    # ybcModifyTable.c中定义了YugaByte对postgres的某些实现
    # 插入数据到table中    
    - YBCHeapInsert
        # 关于es_yb_is_single_row_modify_txn的设置请参考“Executor状态中记录的es_yb_is_single_row_modify_txn是如何被设置的”
        - 如果Executor状态中记录的es_yb_is_single_row_modify_txn为true，表明这是一个单行事务(并不一定说只修改单行数据的事务就是单行事务)
            - YBCExecuteNonTxnInsert
                - YBCExecuteInsertInternal(... ,true /* is_single_row_txn */)
        - 否则
            - YBCExecuteInsert
                - YBCExecuteInsertInternal(... ,false /* is_single_row_txn */)
    # 修改索引表
    - ExecInsertIndexTuples
        - index_insert
            # YugaByte改写了PostgreSQL的IndexAmRoutine，IndexAmRoutine中定义了索引操作相关的方法，
            # YugaByte在其中添加了yb_aminsert和yb_amdelete这2个成员方法，并在ybcinhandler中重新
            # 实现了IndexAmRoutine中的所有成员方法
            - ybcininsert
                - YBCExecuteInsertIndex
                    - YBCPgNewInsert
                    - PrepareIndexWriteStmt
                    # 该方法在ExecInsert -> YBCHeapInsert -> YBCExecuteInsert -> YBCExecuteInsertInternal中已经讲解
                    - YBCExecWriteStmt
                    - YBCPgDeleteStatement
```

从上面分析来看，无论是单行事务还是分布式事务，最终都会调用YBCExecuteInsertInternal，只不过传递的最后一个参数对于单行事务为true，分布式事务则为false，YBCExecuteInsertInternal的实现如下：
```
YBCExecuteInsertInternal(Relation rel,
                        TupleDesc tupleDesc,
                        HeapTuple tuple,
                        bool is_single_row_txn)
    # 获取database oid，table oid等
    - dboid = YBCGetDatabaseOid(rel)
      relid = RelationGetRelid(rel)
    # 创建INSERT请求，返回YBCPgStatement(实际上就是PgStatement) @insert_stmt
    # 关于YBCPgNewInsert的分析参考“YugaByte之pggate”
    - YBCPgNewInsert(..., &insert_stmt)
    # 获取DocKey，返回的是DocKey在内存中的地址
    - tuple->t_ybctid = YBCGetYBTupleIdFromTuple(insert_stmt, rel, tuple, tupleDesc)
        - Bitmapset *pkey = GetFullYBTablePrimaryKey(rel)
            - GetTablePrimaryKey
                - Oid dboid = YBCGetDatabaseOid(rel)
                  Oid relid = RelationGetRelid(rel)
                # 获取表中总的attributes数目(就是column数目？)
                - int natts = RelationGetNumberOfAttributes(rel)  
                # 获取Table descriptor信息，并通过@ybc_tabledesc返回
                - YBCPgGetTableDesc(ybc_pg_session, dboid, relid, &ybc_tabledesc)
                - 遍历table中每一个attribute(也就是column)，并逐一处理：
                    # 获取当前column是否是primary key，是否是partition key，并
                    # 通过is_primary和is_hash返回
                    # 关于YBCPgGetColumnInfo见“YugaByte之pggate”
                    - YBCPgGetColumnInfo(..., &is_primary, &is_hash)
                    - 如果是primary key，则添加到关于primary key的BitmapSet中
                        - pkey = bms_add_member(pkey, attnum - minattr)
        # 返回primary key BitmapSet中primary key的数目
        - nattrs = bms_num_members(pkey)
        # 分配一个数组YBCPgAttrValueDescriptor，用于存放attribute和它所对应的value信息，
        # YBCPgAttrValueDescriptor中包括attribute的类型，attribute的数据
        - YBCPgAttrValueDescriptor *attrs =
			(YBCPgAttrValueDescriptor*)palloc(nattrs * sizeof(YBCPgAttrValueDescriptor))
        - 逐一处理primary key BitmapSet中的每个primary key，主要逻辑如下：
            - 获取YBCPgAttrValueDescriptor数组中下一个空闲位置，存放当前primary key对应的信息
            - 设置它的数据类型(从PostgreSQL的数据类型映射到YugaByte的数据类型)
            - 设置它的数据在HeapTuple中的位置？
        # 构建yugabyte DocKey
        # 关于YBCPgDmlBuildYBTupleId，参考“YugaByte之pggate”
        - YBCPgDmlBuildYBTupleId
    # 将DocKey添加到insert_stmt中
    - YBCBindTupleId(insert_stmt, tuple->t_ybctid)
        # 生成关于DocKey的constant expresssion
        - YBCPgExpr ybc_expr = YBCNewConstant
        # 将生成的关于DocKey的constant expression绑定到代表DocKey的column上
        # 参数@pg_stmt表示PgStatement，@YBTupleIdAttributeNumber表示column编号，
        # @ybc_expr表示待绑定的value
        # 关于YBCPgDmlBindColumn，参考“YugaByte之pggate”
        - YBCPgDmlBindColumn(pg_stmt, YBTupleIdAttributeNumber, ybc_expr)
    - 逐一处理每一个column(attribute)，对于每一行处理如下：
        - 获取当前column的数据@datum
        - 如果当前column是primary key，但是它的值为空，则提示错误
        # 基于@datum，生成关于当前column的expression @ybc_expr
        - YBCPgExpr ybc_expr = YBCNewConstant(insert_stmt, type_id, datum, is_null)
        # 将生成的关于当前column的constant expression绑定到当前column上
        # 关于YBCPgDmlBindColumn，参考“YugaByte之pggate”
        - YBCPgDmlBindColumn(insert_stmt, attnum, ybc_expr)
    # 以上一步骤中返回的YBCPgStatement @insert_stmt作为参数，其中已经绑定了DocKey
    # 和其它相关Column的value
    - YBCExecWriteStmt(insert_stmt, rel, NULL /* rows_affected_count */)
        # 见“YugaByte之pggate”
        - YBCPgDmlExecWriteOp
```


### EState中记录的es_yb_is_single_row_modify_txn是如何被设置的
```
ProcessQuery
    # 如果满足以下条件，则es_yb_is_single_row_modify_txn被设置为true：
    # 1. isSingleRowModifyTxn为true，它在以下条件下为true：
    #    这是一个modify操作，也就是说，是insert，update或者delete操作
    #    只修改一个target table
    #    没有ON CONFLICT和WITH语句
    #    源数据来自于VALUES语句，且在VALUES中只设置一个value(这里一个value指的是一行，
    #    可以设置一行中的一个或者多个列)
    #    VALUES语句中的值是常量或者bind markers(这是啥？)
    # 2. 不在transaction block中
    # 3. 只修改一个table
    # 4. 被修改的table上没有触发器
    # 5. 被修改的table上没有索引二级索引
    - queryDesc->estate->es_yb_is_single_row_modify_txn =
		isSingleRowModifyTxn && queryDesc->estate->es_num_result_relations == 1 &&
		YBCIsSingleRowTxnCapableRel(&queryDesc->estate->es_result_relations[0])
```
