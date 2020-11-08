# 提纲
[toc]

## PgsqlWriteOperation::Init
```
Status PgsqlWriteOperation::Init(PgsqlWriteRequestPB* request, PgsqlResponsePB* response) {
  // Initialize operation inputs.
  request_.Swap(request);
  response_ = response;

  // Init DocDB key using either ybctid or partition and range values.
  if (request_.has_ybctid_column_value()) {
    # 如果request中包含ybctid，则可以从中解析出来DocKey
    const string& ybctid = request_.ybctid_column_value().value().binary_value();
    SCHECK(!ybctid.empty(), InternalError, "empty ybctid");
    doc_key_.emplace(schema_);
    RETURN_NOT_OK(doc_key_->DecodeFrom(ybctid));
  } else {
    # 否则，从request中的partition_column_values中解析出DocKey中hash columns，
    # 从request中的range_column_values中解析出来DocKey中range columns
    vector<PrimitiveValue> hashed_components;
    vector<PrimitiveValue> range_components;
    RETURN_NOT_OK(InitKeyColumnPrimitiveValues(request_.partition_column_values(),
                                               schema_,
                                               0,
                                               &hashed_components));
    RETURN_NOT_OK(InitKeyColumnPrimitiveValues(request_.range_column_values(),
                                               schema_,
                                               schema_.num_hash_key_columns(),
                                               &range_components));
    if (hashed_components.empty()) {
      # hash columns可能不存在，则DocKey中只包含range columns
      doc_key_.emplace(schema_, range_components);
    } else {
      # 否则，DocKey中包含hash值 + hash columns + range columnns
      doc_key_.emplace(schema_, request_.hash_code(), hashed_components, range_components);
    }
  }
  
  # 形成编码之后的DocKey
  encoded_doc_key_ = doc_key_->EncodeAsRefCntPrefix();

  return Status::OK();
}
```

## PgsqlWriteOperation::Apply
```
Status PgsqlWriteOperation::Apply(const DocOperationApplyData& data) {
  auto scope_exit = ScopeExit([this] {
    if (!result_buffer_.empty()) {
      NetworkByteOrder::Store64(result_buffer_.data(), result_rows_);
    }
  });

  switch (request_.stmt_type()) {
    # 根据请求类型，分别处理
    case PgsqlWriteRequestPB::PGSQL_INSERT:
      return ApplyInsert(data, IsUpsert::kFalse);

    case PgsqlWriteRequestPB::PGSQL_UPDATE:
      return ApplyUpdate(data);

    case PgsqlWriteRequestPB::PGSQL_DELETE:
      return ApplyDelete(data);

    case PgsqlWriteRequestPB::PGSQL_UPSERT: {
      // Upserts should not have column refs (i.e. require read).
      DSCHECK(!request_.has_column_refs() || request_.column_refs().ids().empty(),
              IllegalState,
              "Upsert operation should not have column references");
      return ApplyInsert(data, IsUpsert::kTrue);
    }

    case PgsqlWriteRequestPB::PGSQL_TRUNCATE_COLOCATED:
      return ApplyTruncateColocated(data);
  }
  return Status::OK();
}
```

## PgsqlWriteOperation::ApplyInsert
```
Status PgsqlWriteOperation::ApplyInsert(const DocOperationApplyData& data, IsUpsert is_upsert) {
  QLTableRow table_row;
  if (!is_upsert) {
    # 不是upsert操作，则需要先读取是否已经存在相同的primary key
    RETURN_NOT_OK(ReadColumns(data, &table_row));
    if (!table_row.IsEmpty()) {
      # 如果已经存在相同primary key的行，则提示duplicate错误
      // Primary key or unique index value found.
      response_->set_status(PgsqlResponsePB::PGSQL_STATUS_DUPLICATE_KEY_ERROR);
      response_->set_error_message("Duplicate key found in primary key or unique index");
      return Status::OK();
    }
  }

  // Add the liveness column.
  static const PrimitiveValue kLivenessColumnId =
      PrimitiveValue::SystemColumnId(SystemColumnIds::kLivenessColumn);
      
  # 添加类似这样的一行：(hash, hash keys, range keys), liveness_column_id, T1 -> [NULL]
  RETURN_NOT_OK(data.doc_write_batch->SetPrimitive(
      # encoded_doc_key_.as_slice()作为DocKey部分，kLivenessColumnId作为SubKey部分
      DocPath(encoded_doc_key_.as_slice(), kLivenessColumnId),
      Value(PrimitiveValue()),
      data.read_time, data.deadline, request_.stmt_id()));

  # 处理每一列
  for (const auto& column_value : request_.column_values()) {
    // Get the column.
    if (!column_value.has_column_id()) {
      return STATUS(InternalError, "column id missing", column_value.DebugString());
    }
    const ColumnId column_id(column_value.column_id());
    const ColumnSchema& column = VERIFY_RESULT(schema_.column_by_id(column_id));

    // Check column-write operator.
    CHECK(GetTSWriteInstruction(column_value.expr()) == bfpg::TSOpcode::kScalarInsert)
      << "Illegal write instruction";

    // Evaluate column value.
    QLExprResult expr_result;
    # 获取当前column对应的value，之所以这里用的是EvalExpr是因为有些column的值是
    # 计算出来的，而不一定是用户提供的value，结果保存在expr_result中
    RETURN_NOT_OK(EvalExpr(column_value.expr(), table_row, expr_result.Writer()));
    const SubDocument sub_doc =
        SubDocument::FromQLValuePB(expr_result.Value(), column.sorting_type());

    # DocKey + ColumnId组成sub doc key，也就是sub doc path
    DocPath sub_path(encoded_doc_key_.as_slice(), PrimitiveValue(column_id));
    # 将sub doc添加到data.doc_write_batch
    RETURN_NOT_OK(data.doc_write_batch->InsertSubDocument(
        sub_path, sub_doc, data.read_time, data.deadline, request_.stmt_id()));
  }

  # 假设我们更新一个表中的columnX和columnY，且columnY中包含2个subkeys， 
  # subkey1和subkey2，则经过上面的操作之后，data.doc_write_batch中包含了如下信息：
  # (hash, hash keys, range keys), liveness_column_id, T1 -> [NULL]
  # (hash, hash keys, range keys), columnX, T1 -> valueX
  # (hash, hash keys, range keys), columnY, subKey1_of_columnY, T1 -> value_of_subKey1_of_columnY
  # (hash, hash keys, range keys), columnY, subKey2_of_columnY, T1 -> value_of_subKey2_of_columnY
  
  # PopulateResultSet对后续操作没什么影响？暂时不关注
  RETURN_NOT_OK(PopulateResultSet(table_row));

  response_->set_status(PgsqlResponsePB::PGSQL_STATUS_OK);
  return Status::OK();
}

DocWriteBatch::InsertSubDocument
    - DocWriteBatch::ExtendSubDocument
        # 咱们假设这里的value的type是 primitive type，暂不考虑object type或者
        # array type的情况，最终保存在rocksdb中的value由value, ttl, user_timestamp
        # 3部分组成
        - DocWriteBatch::SetPrimitive(doc_path, Value(value, ttl, user_timestamp),
                                    read_ht, deadline, query_id)
            - DocWriteBatch::SetPrimitive(const DocPath& doc_path, const Value& value, LazyIterator* iter)
                - const int num_subkeys = doc_path.num_subkeys()
                  key_prefix_ = doc_path.encoded_doc_key()
                - DocWriteBatch::SetPrimitiveInternal
                    # 下面一段代码要结合[这里](https://docs.yugabyte.com/latest/architecture/docdb/persistence/#ycql-collection-type-example)来看，会比较容易理解
                    - const auto write_id = static_cast<IntraTxnWriteId>(put_batch_.size())
                    - const DocHybridTime hybrid_time = DocHybridTime(HybridTime::kMax, write_id)
                    - for (int subkey_index = 0; subkey_index < num_subkeys; ++subkey_index) {
                        const PrimitiveValue& subkey = doc_path.subkey(subkey_index);
                        KeyBytes parent_key(key_prefix_)
                        put_batch_.emplace_back(std::move(*parent_key.mutable_data()),
                            string(1, ValueTypeAsChar::kObject))
                        cache_.Put(KeyBytes(key_prefix_.AsSlice()), hybrid_time,     
                            ValueType::kObject)
                        # 将当前的subkey添加到key_prefix_中形成新的key_prefix_
                        subkey.AppendToKey(&key_prefix_)
                      }
```

## PgsqlWriteOperation::ReadColumns
```
Status PgsqlWriteOperation::ReadColumns(const DocOperationApplyData& data,
                                        QLTableRow* table_row) {
  // Filter the columns using primary key.
  if (doc_key_) {
    Schema projection;
    # 获取除primary key之外的columns，并且为这些columns创建一个新的schema @projection
    RETURN_NOT_OK(CreateProjection(schema_, request_.column_refs(), &projection));
    # *doc_key_会作为scan过程中的lower bound key，lower bound key后面追加
    # ValueType::kHighest后形成的key作为scan过程中的upper bound key，有了
    # lower bound key和upper bound key之后，在迭代查找的过程中就有了查找边界
    DocPgsqlScanSpec spec(projection, request_.stmt_id(), *doc_key_);
    DocRowwiseIterator iterator(projection,
                                schema_,
                                txn_op_context_,
                                data.doc_write_batch->doc_db(),
                                data.deadline,
                                data.read_time);
    # 见DocRowwiseIterator::Init                                
    RETURN_NOT_OK(iterator.Init(spec));
    
    # scan，从spec中指定的lower bound key一直查找到upper bound key为止
    if (VERIFY_RESULT(iterator.HasNext())) {
      # 将一行(postgresql中看到的行)中存在于@projection中的所有的columns的数据
      # 读取出来，存放在table_row中，table_row类型是QLTableRow，它当中包含一个
      # 从column id到column value的映射表：std::unordered_map<ColumnIdRep, QLTableColumn>
      RETURN_NOT_OK(iterator.NextRow(table_row));
    } else {
      table_row->Clear();
    }
    
    # DocRowwiseIterator在scan过程中会更新它所看到的大于data.read_time的
    # 最大的hybrid time，记录在DocRowwiseIterator::IntentAwareIterator::
    # max_seen_ht_中，这里用max_seen_ht_来更新data.restart_read_ht
    data.restart_read_ht->MakeAtLeast(iterator.RestartReadHt());
  }

  return Status::OK();
}

```



