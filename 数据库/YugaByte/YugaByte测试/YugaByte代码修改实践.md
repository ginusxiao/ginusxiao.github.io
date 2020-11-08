# 提纲
[toc]

## 修改YugaByte代码，确保开启tserver_noop_read_write的情况下读写请求可以正常运行
### 确保开启tserver_noop_read_write的情况下写请求可以正常运行
```
void TabletServiceImpl::Write(const WriteRequestPB* req,
                              WriteResponsePB* resp,
                              rpc::RpcContext context) {
  if (FLAGS_tserver_noop_read_write) {
    for (int i = 0; i < req->ql_write_batch_size(); ++i) {
      resp->add_ql_response_batch();
      resp->mutable_ql_response_batch(i)->set_status(
        QLResponsePB_QLStatus::QLResponsePB_QLStatus_YQL_STATUS_OK);
    }
    context.RespondSuccess();
    return;
  }

  ...
}
```

### 确保开启tserver_noop_read_write的情况下读请求可以正常运行
#### 对yugabyte的读逻辑进行修改
##### 响应中带数据
```
void TabletServiceImpl::Read(const ReadRequestPB* req,
                             ReadResponsePB* resp,
                             rpc::RpcContext context) {
  if (FLAGS_tserver_noop_read_write) {
	int num = 1;
	// *************添加下面for循环，当前修改主要是适配主要是针对yb-sample-app**************** \\
    for (int i = 0; i < req->ql_batch_size(); ++i) {
      QLReadRequestPB ql_read_req = req->ql_batch(i);
      QLRSRowDesc rsrow_desc(ql_read_req.rsrow_desc());
      tablet::QLReadRequestResult result;
      QLResultSet resultset(&rsrow_desc, &result.rows_data);
      string columnVal = std::to_string(num++);
      int rows_data_sidecar_idx = 0;

      for (int rscol_index = 0; rscol_index < rsrow_desc.rscol_count(); rscol_index++) {
    	  if (rsrow_desc.rscol_descs().at(rscol_index).name() == "k") {
    		  if ((ql_read_req.has_where_expr()) && (ql_read_req.where_expr().has_value())) {
    			  resultset.AppendColumn(rscol_index, ql_read_req.where_expr().value());
    		  } else {
    			  std::shared_ptr<QLType> ql_type = rsrow_desc.rscol_descs().at(rscol_index).ql_type();
    			  QLValue value;
    			  if (ql_type->main() == STRING) {
					  value.set_string_value("key");
					  resultset.AppendColumn(rscol_index, value);
    			  } else {

    			  }
    		  }
		  } else {
			  std::shared_ptr<QLType> ql_type = rsrow_desc.rscol_descs().at(rscol_index).ql_type();
			  QLValue value;
			  if (ql_type->main() == BINARY) {
				  value.set_binary_value("value");
				  resultset.AppendColumn(rscol_index, value);
			  } else if (ql_type->main() == STRING) {
				  value.set_string_value("value");
				  resultset.AppendColumn(rscol_index, value);
			  } else {

			  }
		  }
	  }

	  context.AddRpcSidecar(RefCntBuffer(result.rows_data), &rows_data_sidecar_idx);
	  result.response.set_rows_data_sidecar(rows_data_sidecar_idx);
      resp->add_ql_batch();
      resp->mutable_ql_batch(i)->Swap(&result.response);
      resp->mutable_ql_batch(i)->set_status(
        QLResponsePB_QLStatus::QLResponsePB_QLStatus_YQL_STATUS_OK);
    }

    LOG(INFO) << resp->trace_buffer();

    context.RespondSuccess();

    return;
  }
  
  ...
}
```

##### 响应中不带数据
```
diff --git a/src/yb/tserver/tablet_service.cc b/src/yb/tserver/tablet_service.cc
index 342e5c7..7851a10 100644
--- a/src/yb/tserver/tablet_service.cc
+++ b/src/yb/tserver/tablet_service.cc
@@ -40,6 +40,7 @@
 #include "yb/client/transaction.h"
 #include "yb/client/transaction_pool.h"
 
+#include "yb/common/ql_resultset.h"
 #include "yb/common/ql_value.h"
 #include "yb/common/row_mark.h"
 #include "yb/common/schema.h"
@@ -836,7 +837,7 @@ void TabletServiceAdminImpl::CountIntents(
 void TabletServiceImpl::Write(const WriteRequestPB* req,
                               WriteResponsePB* resp,
                               rpc::RpcContext context) {
-  if (FLAGS_tserver_noop_read_write) {
+  if (false && FLAGS_tserver_noop_read_write) {
     for (int i = 0; i < req->ql_write_batch_size(); ++i) {
       resp->add_ql_response_batch();
     }
@@ -1194,6 +1195,55 @@ void TabletServiceImpl::Read(const ReadRequestPB* req,
                              ReadResponsePB* resp,
                              rpc::RpcContext context) {
   if (FLAGS_tserver_noop_read_write) {
+    int num = 1;
+    for (int i = 0; i < req->ql_batch_size(); ++i) {
+      QLReadRequestPB ql_read_req = req->ql_batch(i);
+      QLRSRowDesc rsrow_desc(ql_read_req.rsrow_desc());
+      tablet::QLReadRequestResult result;
+      QLResultSet resultset(&rsrow_desc, &result.rows_data);
+      string columnVal = std::to_string(num++);
+      int rows_data_sidecar_idx = 0;
+
+      context.AddRpcSidecar(RefCntBuffer(result.rows_data), &rows_data_sidecar_idx);
+      result.response.set_rows_data_sidecar(rows_data_sidecar_idx);
+      resp->add_ql_batch();
+      resp->mutable_ql_batch(i)->Swap(&result.response);
+      resp->mutable_ql_batch(i)->set_status(
+        QLResponsePB_QLStatus::QLResponsePB_QLStatus_YQL_STATUS_OK);
+    }
+
+	//LOG(INFO) << resp->trace_buffer();
     context.RespondSuccess();
     return;
   }
```


#### 对YugaByte性能测试程序yb-sample-apps的修改
按照“对yugabyte的读逻辑进行修改”修改后，YugaByte cqlsh执行select时，对任何key都返回空，但是使用yb-sample-apps进行性能测试的时候，会提示应该读取到1条数据，但是只读取到了0条数据，鉴于此，需要对yb-sample-apps中的CassandraKeyValueBase.java进行修改如下：
```
  public long doRead() {
    SimpleLoadGenerator.Key key = getSimpleLoadGenerator().getKeyToRead();
    if (key == null) {
      // There are no keys to read yet.
      return 0;
    } 
    // Do the read from Cassandra.
    // Bind the select statement.
    BoundStatement select = bindSelect(key.asString());
    ResultSet rs = getCassandraClient().execute(select);
    // *************注释掉下面的代码**************** \\
    /*
    List<Row> rows = rs.all();
    if (rows.size() != 1) {
      // If TTL is enabled, turn off correctness validation.
      if (appConfig.tableTTLSeconds <= 0) {
        LOG.fatal("Read key: " + key.asString() + " expected 1 row in result, got " + rows.size());
      }
      return 1;
    }
    if (appConfig.valueSize == 0) {
      ByteBuffer buf = rows.get(0).getBytes(1);
      String value = new String(buf.array());
      key.verify(value);
    } else {
      ByteBuffer value = rows.get(0).getBytes(1);
      byte[] bytes = new byte[value.capacity()];
      value.get(bytes);
      verifyRandomValue(key, bytes);
    }
    LOG.debug("Read key: " + key.toString());
    */
    return 1;
  }
```

## CQLServer处理请求后，不发送read RPC请求给TServer(但是依然会执行TabletLookup)，而是直接调用RPC回调
```
diff --git a/src/yb/client/async_rpc.cc b/src/yb/client/async_rpc.cc
index a3af614..1b37ee7 100644
--- a/src/yb/client/async_rpc.cc
+++ b/src/yb/client/async_rpc.cc
@@ -698,10 +698,14 @@ void ReadRpc::SwapRequestsAndResponses(bool skip_responses) {
         break;
       }
       case YBOperation::Type::QL_READ: {
+        // modify by xf
         if (ql_idx >= resp_.ql_batch().size()) {
-          batcher_->AddOpCountMismatchError();
-          return;
+          resp_.add_ql_batch();
+          resp_.mutable_ql_batch(ql_idx)->set_status(QLResponsePB_QLStatus::QLResponsePB_QLStatus_YQL_STATUS_OK);
+          //batcher_->AddOpCountMismatchError();
+          //return;
         }
+        
         // Restore QL read request PB and extract response.
         auto* ql_op = down_cast<YBqlReadOp*>(yb_op);
         ql_op->mutable_response()->Swap(resp_.mutable_ql_batch(ql_idx));
diff --git a/src/yb/client/async_rpc.h b/src/yb/client/async_rpc.h
index 1d62cee..fc8d80c 100644
--- a/src/yb/client/async_rpc.h
+++ b/src/yb/client/async_rpc.h
@@ -82,10 +82,10 @@ class AsyncRpc : public rpc::Rpc, public TabletRpc {
   const YBTable* table() const;
   const RemoteTablet& tablet() const { return *tablet_invoker_.tablet(); }
   const InFlightOps& ops() const { return ops_; }
-
- protected:
+  // modify by xf
   void Finished(const Status& status) override;
-
+ 
+ protected:
   void SendRpcToTserver(int attempt_num) override;
 
   virtual void CallRemoteMethod() = 0;
diff --git a/src/yb/client/batcher.cc b/src/yb/client/batcher.cc
index d3ee7e1..e46a59f 100644
--- a/src/yb/client/batcher.cc
+++ b/src/yb/client/batcher.cc
@@ -222,6 +222,8 @@ void Batcher::CheckForFinishedFlush() {
   }
 
   Status s;
+  // modify by xf
+  /*
   if (!combined_error_.ok()) {
     s = combined_error_;
   } else if (had_errors_.load(std::memory_order_acquire)) {
@@ -230,7 +232,9 @@ void Batcher::CheckForFinishedFlush() {
     // https://github.com/YugaByte/yugabyte-db/issues/702
     s = STATUS(IOError, kErrorReachingOutToTServersMsg);
   }
+  */
 
+  s = Status::OK();
   RunCallback(s);
 }
 
@@ -593,7 +597,14 @@ void Batcher::ExecuteOperations() {
   ops_queue_.clear();
 
   for (const auto& rpc : rpcs) {
-    rpc->SendRpc();
+    // modify by xf
+    if (rpc->ops().at(0)->yb_op->read_only() && 
+      //(rpc->table()->name().table_name().find("cassandrakeyvalue") != std::string::npos)) {
+      (rpc->table()->name().table_name().find("andrak") != std::string::npos)) {
+      rpc->Finished(yb::Status::OK()); 
+    } else {
+      rpc->SendRpc();
+    }
   }
 }
 
diff --git a/src/yb/yql/cql/ql/exec/executor.cc b/src/yb/yql/cql/ql/exec/executor.cc
index 6150f31..dccb94b 100644
--- a/src/yb/yql/cql/ql/exec/executor.cc
+++ b/src/yb/yql/cql/ql/exec/executor.cc
@@ -1754,8 +1754,9 @@ Result<bool> Executor::ProcessTnodeResults(TnodeContext* tnode_context) {
   for (auto op_itr = ops.begin(); op_itr != ops.end(); ) {
     YBqlOpPtr& op = *op_itr;
 
+    // modify by xf
     // Apply any op that has not been applied and executed.
-    if (!op->response().has_status()) {
+    if (false && !op->response().has_status()) {
       DCHECK_EQ(op->type(), YBOperation::Type::QL_WRITE);
       if (write_batch_.Add(std::static_pointer_cast<YBqlWriteOp>(op))) {
         YBSessionPtr session = GetSession(exec_context_);
@@ -1822,7 +1823,7 @@ Result<bool> Executor::ProcessTnodeResults(TnodeContext* tnode_context) {
     }
 
     // For SELECT statement, check if there are more rows to fetch and apply the op as needed.
-    if (tnode->opcode() == TreeNodeOpcode::kPTSelectStmt) {
+    if (false && (tnode->opcode() == TreeNodeOpcode::kPTSelectStmt)) {
       const auto* select_stmt = static_cast<const PTSelectStmt *>(tnode);
       // Do this except for the parent SELECT with an index. For covered index, we will select
       // from the index only. For uncovered index, the parent SELECT will fetch using the primary
```

## YugaByte read执行cql解析，在Tablet lookup rpc的回调TabletLookupFinished中直接返回响应
```
diff --git a/src/yb/client/batcher.cc b/src/yb/client/batcher.cc
index d3ee7e1..3897d5a 100644
--- a/src/yb/client/batcher.cc
+++ b/src/yb/client/batcher.cc
@@ -117,10 +117,12 @@ Batcher::Batcher(YBClient* client,
                  const YBSessionPtr& session,
                  YBTransactionPtr transaction,
                  ConsistentReadPoint* read_point,
-                 bool force_consistent_read)
+                 bool force_consistent_read,
+                StatusFunctor flush_callback)
   : client_(client),
     weak_session_(session),
     error_collector_(error_collector),
+    flush_callback_(flush_callback), 
     next_op_sequence_number_(0),
     async_rpc_metrics_(session->async_rpc_metrics()),
     transaction_(std::move(transaction)),
@@ -371,6 +373,9 @@ void Batcher::TabletLookupFinished(
   // 2. Change the op state.
 
   bool all_lookups_finished;
+  bool userTableRead = false;
+  YBOperation* yb_op = op->yb_op.get();
+
   {
     std::lock_guard<decltype(mutex_)> lock(mutex_);
 
@@ -436,6 +441,44 @@ void Batcher::TabletLookupFinished(
     } else {
       MarkInFlightOpFailedUnlocked(op, lookup_result.status());
     }
+
+    if (yb_op->read_only() &&
+         (yb_op->table()->name().table_name().find("cassandrakeyvalue") != std::string::npos)) {
+       userTableRead = true;
+        // modify by xf
+        ops_.erase(op);
+    }
+  }
+
+  // modify by xf
+  if (userTableRead) {
+       switch (yb_op->type()) {
+         case YBOperation::Type::REDIS_READ: {
+               break;
+         }
+         case YBOperation::Type::QL_READ: {
+               auto* ql_op = down_cast<YBqlReadOp*>(yb_op);
+               tserver::ReadResponsePB resp;
+               //resp.add_ql_batch();
+               //resp.mutable_ql_batch(0)->set_status(QLResponsePB_QLStatus::QLResponsePB_QLStatus_YQL_STATUS_OK);
+               ql_op->mutable_response()->set_status(QLResponsePB_QLStatus::QLResponsePB_QLStatus_YQL_STATUS_OK);
+               break;
+         }
+         case YBOperation::Type::PGSQL_READ: {
+               break;
+         }
+         case YBOperation::Type::PGSQL_WRITE: FALLTHROUGH_INTENDED;
+         case YBOperation::Type::REDIS_WRITE: FALLTHROUGH_INTENDED;
+         case YBOperation::Type::QL_WRITE:
+               LOG(FATAL) << "Not a read operation " << op->yb_op->type();
+               break;
+       }
+
+       if (nullptr != flush_callback_) {
+               flush_callback_(Status::OK());
+       }
+
+        return;
   }
 
   if (!lookup_result.ok()) {
diff --git a/src/yb/client/batcher.h b/src/yb/client/batcher.h
index ebd47c4..ffe6ef7 100644
--- a/src/yb/client/batcher.h
+++ b/src/yb/client/batcher.h
@@ -109,7 +109,8 @@ class Batcher : public RefCountedThreadSafe<Batcher> {
           const YBSessionPtr& session,
           YBTransactionPtr transaction,
           ConsistentReadPoint* read_point,
-          bool force_consistent_read);
+          bool force_consistent_read,
+          StatusFunctor flush_callback);
 
   // Abort the current batch. Any writes that were buffered and not yet sent are
   // discarded. Those that were sent may still be delivered.  If there is a pending Flush
diff --git a/src/yb/client/session.cc b/src/yb/client/session.cc
index 36ee3d8..2ac53cf 100644
--- a/src/yb/client/session.cc
+++ b/src/yb/client/session.cc
@@ -189,7 +189,7 @@ internal::Batcher& YBSession::Batcher() {
   if (!batcher_) {
     batcher_.reset(new internal::Batcher(
         client_, error_collector_.get(), shared_from_this(), transaction_, read_point(),
-        force_consistent_read_));
+        force_consistent_read_, flush_callback_));
     if (timeout_.Initialized()) {
       batcher_->SetTimeout(timeout_);
     }
diff --git a/src/yb/client/session.h b/src/yb/client/session.h
index 64840e3..8881f5b 100644
--- a/src/yb/client/session.h
+++ b/src/yb/client/session.h
@@ -117,6 +117,15 @@ class YBSession : public std::enable_shared_from_this<YBSession> {
     return timeout_;
   }
 
+  // modify by xf
+  void SetFlushCallback(StatusFunctor callback) {
+         flush_callback_ = callback;
+  }
+
+  StatusFunctor GetFlushCallback() {
+         return flush_callback_;
+  }
+
   CHECKED_STATUS ReadSync(std::shared_ptr<YBOperation> yb_op);
 
   void ReadAsync(std::shared_ptr<YBOperation> yb_op, StatusFunctor callback);
@@ -268,6 +277,10 @@ class YBSession : public std::enable_shared_from_this<YBSession> {
   // The current batcher being prepared.
   scoped_refptr<internal::Batcher> batcher_;
 
+  // modify by xf
+  // The flush callback
+  StatusFunctor flush_callback_;
+     
   // Any batchers which have been flushed but not yet finished.
   //
   // Upon a batch finishing, it will call FlushFinished(), which removes the batcher from
diff --git a/src/yb/yql/cql/ql/exec/executor.cc b/src/yb/yql/cql/ql/exec/executor.cc
index 6150f31..cd9ec1d 100644
--- a/src/yb/yql/cql/ql/exec/executor.cc
+++ b/src/yb/yql/cql/ql/exec/executor.cc
@@ -91,6 +91,11 @@ void Executor::ExecuteAsync(const ParseTree& parse_tree, const StatementParamete
   } else {
     session_->SetReadPoint(client::Restart::kFalse);
   }
+ 
+  // modify by xf 
+  session_->SetFlushCallback([this](const Status& s) {
+    FlushAsyncDone(s, nullptr);
+  });
   RETURN_STMT_NOT_OK(Execute(parse_tree, params));
   FlushAsync();
 }
@@ -1465,6 +1470,10 @@ void Executor::FlushAsync() {
   // FlushAsync() and CommitTransaction(). This is necessary so that only the last callback will
   // correctly detect that all async calls are done invoked before processing the async results
   // exclusively.
+
+  // modify by xf
+  return StatementExecuted(Status::OK());
+
   write_batch_.Clear();
   std::vector<std::pair<YBSessionPtr, ExecContext*>> flush_sessions;
   std::vector<ExecContext*> commit_contexts;
```

## 直接在reactor中返回响应(直接在CQLConnectionContext::HandleCall中返回响应)
```
diff --git a/src/postgres/src/bin/initdb/initdb.c b/src/postgres/src/bin/initdb/initdb.c
index b0b4c74..d9fbd72 100644
--- a/src/postgres/src/bin/initdb/initdb.c
+++ b/src/postgres/src/bin/initdb/initdb.c
@@ -2364,6 +2364,11 @@ check_locale_name(int category, const char *locale, char **canonname)
        if (canonname)
                *canonname = NULL;              /* in case of failure */
 
+        if (locale) {
+            fprintf(stderr, _("%s: check_locale_name: category %d, locale: %s \n"),
+                            progname, category, locale);
+        }
+
        save = setlocale(category, NULL);
        if (!save)
        {
diff --git a/src/yb/rpc/inbound_call.h b/src/yb/rpc/inbound_call.h
index 2a1114a..cf035b2 100644
--- a/src/yb/rpc/inbound_call.h
+++ b/src/yb/rpc/inbound_call.h
@@ -194,7 +194,7 @@ class InboundCall : public RpcCall, public MPSCQueueEntry<InboundCall> {
 
   size_t DynamicMemoryUsage() const override;
 
- protected:
+// protected:
   void NotifyTransferred(const Status& status, Connection* conn) override;
 
   virtual void Clear();
diff --git a/src/yb/rpc/rpc_with_call_id.cc b/src/yb/rpc/rpc_with_call_id.cc
index 58ade71..bb930ea 100644
--- a/src/yb/rpc/rpc_with_call_id.cc
+++ b/src/yb/rpc/rpc_with_call_id.cc
@@ -66,13 +66,18 @@ void ConnectionContextWithCallId::CallProcessed(InboundCall* call) {
   ++processed_call_count_;
   auto id = ExtractCallId(call);
   auto it = calls_being_handled_.find(id);
+  /*
   if (it == calls_being_handled_.end() || it->second != call) {
     std::string existing = it == calls_being_handled_.end() ? "<NONE>" : it->second->ToString();
     LOG(DFATAL) << "Processed call with invalid id: " << id << ", call: " << call->ToString()
                 << ", existing: " << existing;
     return;
   }
-  calls_being_handled_.erase(it);
+  */
+  if (it != calls_being_handled_.end()) {
+    calls_being_handled_.erase(it);
+  }
+
   if (Idle() && idle_listener_) {
     idle_listener_();
   }
diff --git a/src/yb/yql/cql/cqlserver/cql_rpc.cc b/src/yb/yql/cql/cqlserver/cql_rpc.cc
index 58ace7d..e085754 100644
--- a/src/yb/yql/cql/cqlserver/cql_rpc.cc
+++ b/src/yb/yql/cql/cqlserver/cql_rpc.cc
@@ -95,6 +95,27 @@ Status CQLConnectionContext::HandleCall(
     return STATUS_SUBSTITUTE(NetworkError, "Bad data: $0", s.ToUserMessage());
   }
 
+  // modify by xf
+  if (increaseNumCalls() >= 20000) {
+         faststring msg;
+         std::unique_ptr<CQLRequest> request;
+         std::unique_ptr<CQLResponse> response;
+         bool success = CQLRequest::ParseRequest(call->serialized_request(), compression_scheme(),
+                                                                       &request, &response);
+         if (!success) {
+               response->Serialize(compression_scheme(), &msg);
+               call->response_msg_buf_ = RefCntBuffer(msg);
+               call->QueueResponse(true);
+               return Status::OK();
+         } else if (request->opcode() == CQLMessage::Opcode::EXECUTE) {
+               VoidResultResponse* response = new VoidResultResponse(*request);
+               response->Serialize(compression_scheme(), &msg);
+               call->response_msg_buf_ = RefCntBuffer(msg);
+               call->QueueResponse(true);
+               return Status::OK();
+         }
+  }
+
   s = Store(call.get());
   if (!s.ok()) {
     return s;
@@ -194,7 +215,7 @@ void CQLInboundCall::RespondFailure(rpc::ErrorStatusPB::RpcErrorCodePB error_cod
 
 void CQLInboundCall::RespondSuccess(const RefCntBuffer& buffer,
                                     const yb::rpc::RpcMethodMetrics& metrics) {
-  RecordHandlingCompleted(metrics.handler_latency);
+  //RecordHandlingCompleted(metrics.handler_latency);
   response_msg_buf_ = buffer;
 
   QueueResponse(/* is_success */ true);
diff --git a/src/yb/yql/cql/cqlserver/cql_rpc.h b/src/yb/yql/cql/cqlserver/cql_rpc.h
index 9beed7f..478b12a 100644
--- a/src/yb/yql/cql/cqlserver/cql_rpc.h
+++ b/src/yb/yql/cql/cqlserver/cql_rpc.h
@@ -41,6 +41,14 @@ class CQLConnectionContext : public rpc::ConnectionContextWithCallId,
   void DumpPB(const rpc::DumpRunningRpcsRequestPB& req,
               rpc::RpcConnectionPB* resp) override;
 
+  int increaseNumCalls() {
+    return num_calls.fetch_add(1);
+  }
+
+  int getNumCalls() {
+    return num_calls.load(std::memory_order_acquire);
+  }
+
   // Accessor methods for CQL message compression scheme to use.
   CQLMessage::CompressionScheme compression_scheme() const {
     return compression_scheme_;
@@ -78,6 +86,8 @@ class CQLConnectionContext : public rpc::ConnectionContextWithCallId,
     return read_buffer_;
   }
 
+  std::atomic_long num_calls{0};
+
   // SQL session of this CQL client connection.
   ql::QLSession::SharedPtr ql_session_;
 
@@ -146,7 +156,7 @@ class CQLInboundCall : public rpc::InboundCall {
     return DynamicMemoryUsageOf(response_msg_buf_);
   }
 
- private:
+ public:
   RefCntBuffer response_msg_buf_;
   const ql::QLSession::SharedPtr ql_session_;
   uint16_t stream_id_;
```
