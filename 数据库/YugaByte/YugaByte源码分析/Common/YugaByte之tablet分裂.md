# 提纲
[toc]

# 关于YugaByte中tablet的分裂
当前YugaByte只支持table的静态分区，也就是说在创建table的时候，为table创建指定数目的tablet，但是不支持tablet再分裂，也就是说不支持动态分区（官方说法是计划实现动态分区的支持，参考[这里](https://github.com/YugaByte/yugabyte-db/issues/15)）。


# YugaByte中table分区相关源码分析

```
Status YBClient::Data::CreateTable(
    YBClient* client,
    const CreateTableRequestPB& req,
    const YBSchema& schema,
    const MonoTime& deadline) {
  CreateTableResponsePB resp;

  int attempts = 0;
  Status s = SyncLeaderMasterRpc<CreateTableRequestPB, CreateTableResponsePB>(
      deadline, client, req, &resp, &attempts, "CreateTable", &MasterServiceProxy::CreateTable);
  RETURN_NOT_OK(s);
  if (resp.has_error()) {
    if (resp.error().code() == MasterErrorPB::TABLE_ALREADY_PRESENT && attempts > 1) {
      // If the table already exists and the number of attempts is >
      // 1, then it means we may have succeeded in creating the
      // table, but client didn't receive the successful
      // response (e.g., due to failure before the successful
      // response could be sent back, or due to a I/O pause or a
      // network blip leading to a timeout, etc...)
      YBTable::Info info;
      string keyspace = req.has_namespace_() ? req.namespace_().name() :
                        (req.name() == common::kRedisTableName ? common::kRedisKeyspaceName : "");
      const YBTableName table_name(!keyspace.empty()
          ? YBTableName(keyspace, req.name()) : YBTableName(req.name()));

      // A fix for https://yugabyte.atlassian.net/browse/ENG-529:
      // If we've been retrying table creation, and the table is now in the process is being
      // created, we can sometimes see an empty schema. Wait until the table is fully created
      // before we compare the schema.
      RETURN_NOT_OK_PREPEND(
          WaitForCreateTableToFinish(client, table_name, deadline),
          Substitute("Failed waiting for table $0 to finish being created", table_name.ToString()));

      RETURN_NOT_OK_PREPEND(
          GetTableSchema(client, table_name, deadline, &info),
          Substitute("Unable to check the schema of table $0", table_name.ToString()));
      if (!schema.Equals(info.schema)) {
         string msg = Format("Table $0 already exists with a different "
                             "schema. Requested schema was: $1, actual schema is: $2",
                             table_name,
                             internal::GetSchema(schema),
                             internal::GetSchema(info.schema));
        LOG(ERROR) << msg;
        return STATUS(AlreadyPresent, msg);
      } else {
        PartitionSchema partition_schema;
        // We need to use the schema received from the server, because the user-constructed
        // schema might not have column ids.
        RETURN_NOT_OK(PartitionSchema::FromPB(req.partition_schema(),
                                              internal::GetSchema(info.schema),
                                              &partition_schema));
        if (!partition_schema.Equals(info.partition_schema)) {
          string msg = Substitute("Table $0 already exists with a different partition schema. "
              "Requested partition schema was: $1, actual partition schema is: $2",
              table_name.ToString(),
              partition_schema.DebugString(internal::GetSchema(schema)),
              info.partition_schema.DebugString(internal::GetSchema(info.schema)));
          LOG(ERROR) << msg;
          return STATUS(AlreadyPresent, msg);
        } else {
          return Status::OK();
        }
      }
    }
    return StatusFromPB(resp.error().status());
  }
  return Status::OK();
}


void MasterServiceImpl::CreateTable(const CreateTableRequestPB* req,
                                    CreateTableResponsePB* resp,
                                    RpcContext rpc) {
  HandleIn(req, resp, &rpc, &CatalogManager::CreateTable);
}

Status CatalogManager::CreateTable(const CreateTableRequestPB* orig_req,
                                   CreateTableResponsePB* resp,
                                   rpc::RpcContext* rpc) {
  RETURN_NOT_OK(CheckOnline());
  Status s;

  const char* const object_type = orig_req->indexed_table_id().empty() ? "table" : "index";

  // Copy the request, so we can fill in some defaults.
  CreateTableRequestPB req = *orig_req;
  LOG(INFO) << "CreateTable from " << RequestorString(rpc)
            << ":\n" << req.DebugString();

  // Lookup the namespace and verify if it exists.
  TRACE("Looking up namespace");
  scoped_refptr<NamespaceInfo> ns;
  RETURN_NAMESPACE_NOT_FOUND(FindNamespace(req.namespace_(), &ns), resp);
  NamespaceId namespace_id = ns->id();

  // Validate schema.
  Schema client_schema;
  RETURN_NOT_OK(SchemaFromPB(req.schema(), &client_schema));
  if (client_schema.has_column_ids()) {
    s = STATUS(InvalidArgument, "User requests should not have Column IDs");
    return SetupError(resp->mutable_error(), MasterErrorPB::INVALID_SCHEMA, s);
  }
  if (PREDICT_FALSE(client_schema.num_key_columns() <= 0)) {
    s = STATUS(InvalidArgument, "Must specify at least one key column");
    return SetupError(resp->mutable_error(), MasterErrorPB::INVALID_SCHEMA, s);
  }
  for (int i = 0; i < client_schema.num_key_columns(); i++) {
    if (!IsTypeAllowableInKey(client_schema.column(i).type_info())) {
      Status s = STATUS(InvalidArgument,
        "Invalid datatype for primary key column");
      return SetupError(resp->mutable_error(), MasterErrorPB::INVALID_SCHEMA, s);
    }
  }

  // checking that referenced user-defined types (if any) exist.
  {
    boost::shared_lock<LockType> l(lock_);
    for (int i = 0; i < client_schema.num_columns(); i++) {
      for (const auto &udt_id : client_schema.column(i).type()->GetUserDefinedTypeIds()) {
        if (FindPtrOrNull(udtype_ids_map_, udt_id) == nullptr) {
          Status s = STATUS(InvalidArgument, "Referenced user-defined type not found");
          return SetupError(resp->mutable_error(), MasterErrorPB::INVALID_SCHEMA, s);
        }
      }
    }
  }
  // TODO (ENG-1860) The referenced namespace and types retrieved/checked above could be deleted
  // some time between this point and table creation below.
  Schema schema = client_schema.CopyWithColumnIds();

  if (schema.table_properties().HasCopartitionTableId()) {
    return CreateCopartitionedTable(req, resp, rpc, schema, namespace_id);
  }

  // If hashing scheme is not specified by protobuf request, table_type and hash_key are used to
  // determine which hashing scheme should be used.
  if (!req.partition_schema().has_hash_schema()) {
    if (req.table_type() == REDIS_TABLE_TYPE) {
      req.mutable_partition_schema()->set_hash_schema(PartitionSchemaPB::REDIS_HASH_SCHEMA);
    } else if (schema.num_hash_key_columns() > 0) {
      req.mutable_partition_schema()->set_hash_schema(PartitionSchemaPB::MULTI_COLUMN_HASH_SCHEMA);
    }

    // TODO(neil) Need to fix the tests that don't define hash_schema and don't have hash column.
    // Even though we only support REDIS, CQL, and PGSQL schema, we don't raise error for undefined
    // schema due to testing issue.  Once we fix the tests, error can be raised here.
    //
    // else {
    //   Status s = STATUS(InvalidArgument, "Unknown table type or unknown partitioning method");
    //   return SetupError(resp->mutable_error(), MasterErrorPB::INVALID_SCHEMA, s);
    // }
  }

  // Get cluster level placement info.
  ReplicationInfoPB replication_info;
  {
    auto l = cluster_config_->LockForRead();
    replication_info = l->data().pb.replication_info();
  }

  // Calculate number of tablets to be used.
  int num_tablets = req.num_tablets();
  if (num_tablets <= 0) {
    // Use default as client could have gotten the value before any tserver had heartbeated
    // to (a new) master leader.
    TSDescriptorVector ts_descs;
    master_->ts_manager()->GetAllLiveDescriptorsInCluster(
        &ts_descs, replication_info.live_replicas().placement_uuid());
    num_tablets = ts_descs.size() * FLAGS_yb_num_shards_per_tserver;
    LOG(INFO) << "Setting default tablets to " << num_tablets << " with "
              << ts_descs.size() << " primary servers";
  }

  // Create partitions.
  PartitionSchema partition_schema;
  vector<Partition> partitions;
  s = PartitionSchema::FromPB(req.partition_schema(), schema, &partition_schema);
  switch (partition_schema.hash_schema()) {
    case YBHashSchema::kPgsqlHash:
      // TODO(neil) After a discussion, PGSQL hash should be done appropriately.
      // For now, let's not doing anything. Just borrow the multi column hash.
      FALLTHROUGH_INTENDED;
    case YBHashSchema::kMultiColumnHash: {
      // Use the given number of tablets to create partitions and ignore the other schema options
      // in the request.
      RETURN_NOT_OK(partition_schema.CreatePartitions(num_tablets, &partitions));
      break;
    }
    case YBHashSchema::kRedisHash: {
      RETURN_NOT_OK(partition_schema.CreatePartitions(num_tablets, &partitions,
          kRedisClusterSlots));
      break;
    }
  }

  // Validate the table placement rules are a subset of the cluster ones.
  s = ValidateTableReplicationInfo(req.replication_info());
  if (PREDICT_FALSE(!s.ok())) {
    return SetupError(resp->mutable_error(), MasterErrorPB::INVALID_SCHEMA, s);
  }

  TSDescriptorVector all_ts_descs;
  master_->ts_manager()->GetAllLiveDescriptors(&all_ts_descs);
  s = CheckValidReplicationInfo(replication_info, all_ts_descs, partitions, resp);
  if (!s.ok()) {
    return s;
  }

  scoped_refptr<TableInfo> table;
  vector<TabletInfo*> tablets;
  {
    std::lock_guard<LockType> l(lock_);
    TRACE("Acquired catalog manager lock");

    // Verify that the table does not exist.
    table = FindPtrOrNull(table_names_map_, {namespace_id, req.name()});

    if (table != nullptr) {
      s = STATUS(AlreadyPresent, Substitute("Target $0 already exists", object_type), table->id());
      return SetupError(resp->mutable_error(), MasterErrorPB::TABLE_ALREADY_PRESENT, s);
    }

    RETURN_NOT_OK(CreateTableInMemory(req, schema, partition_schema, false, namespace_id,
                                      partitions, &tablets, resp, &table));
  }
  TRACE("Inserted new table and tablet info into CatalogManager maps");

  // NOTE: the table and tablets are already locked for write at this point,
  // since the CreateTableInfo/CreateTabletInfo functions leave them in that state.
  // They will get committed at the end of this function.
  // Sanity check: the tables and tablets should all be in "preparing" state.
  CHECK_EQ(SysTablesEntryPB::PREPARING, table->metadata().dirty().pb.state());
  for (const TabletInfo *tablet : tablets) {
    CHECK_EQ(SysTabletsEntryPB::PREPARING, tablet->metadata().dirty().pb.state());
  }

  // Write Tablets to sys-tablets (in "preparing" state).
  s = sys_catalog_->AddItems(tablets);
  if (PREDICT_FALSE(!s.ok())) {
    return AbortTableCreation(table.get(), tablets,
                              s.CloneAndPrepend(
                                  Substitute("An error occurred while inserting to sys-tablets: $0",
                                             s.ToString())),
                              resp);
  }
  TRACE("Wrote tablets to system table");

  // Update the on-disk table state to "running".
  table->mutable_metadata()->mutable_dirty()->pb.set_state(SysTablesEntryPB::RUNNING);
  s = sys_catalog_->AddItem(table.get());
  if (PREDICT_FALSE(!s.ok())) {
    return AbortTableCreation(table.get(), tablets,
                              s.CloneAndPrepend(
                                  Substitute("An error occurred while inserting to sys-tablets: $0",
                                             s.ToString())),
                              resp);
  }
  TRACE("Wrote table to system table");

  // For index table, insert index info in the indexed table.
  if (req.has_indexed_table_id()) {
    s = AddIndexInfoToTable(req.indexed_table_id(), table->id(), schema, req.is_local_index(),
                            req.is_unique_index());
    if (PREDICT_FALSE(!s.ok())) {
      return AbortTableCreation(table.get(), tablets,
                                s.CloneAndPrepend(
                                    Substitute("An error occurred while inserting index info: $0",
                                               s.ToString())),
                                resp);
    }
  }

  // Commit the in-memory state.
  table->mutable_metadata()->CommitMutation();

  for (TabletInfo *tablet : tablets) {
    tablet->mutable_metadata()->CommitMutation();
  }

  VLOG(1) << "Created table " << table->ToString();
  LOG(INFO) << "Successfully created " << object_type << " " << table->ToString()
            << " per request from " << RequestorString(rpc);
  background_tasks_->Wake();
  return Status::OK();
}

Status CatalogManager::CreateTableInMemory(const CreateTableRequestPB& req,
                                           const Schema& schema,
                                           const PartitionSchema& partition_schema,
                                           const bool is_copartitioned,
                                           const NamespaceId& namespace_id,
                                           const std::vector<Partition>& partitions,
                                           std::vector<TabletInfo*>* tablets,
                                           CreateTableResponsePB* resp,
                                           scoped_refptr<TableInfo>* table) {
  // Verify we have catalog manager lock.
  if (!lock_.is_locked()) {
    return STATUS(IllegalState, "We don't have the catalog manager lock!");
  }

  // Add the new table in "preparing" state.
  table->reset(CreateTableInfo(req, schema, partition_schema, namespace_id));
  table_ids_map_[(*table)->id()] = *table;
  table_names_map_[{namespace_id, req.name()}] = *table;

  if (!is_copartitioned) {
    RETURN_NOT_OK(CreateTabletsFromTable(partitions, *table, tablets));
  }

  if (resp != nullptr) {
    resp->set_table_id((*table)->id());
  }

  return Status::OK();
}

Status CatalogManager::CreateTabletsFromTable(const vector<Partition>& partitions,
                                              const scoped_refptr<TableInfo>& table,
                                              std::vector<TabletInfo*>* tablets) {
  // Create the TabletInfo objects in state PREPARING.
  for (const Partition& partition : partitions) {
    PartitionPB partition_pb;
    partition.ToPB(&partition_pb);
    tablets->push_back(CreateTabletInfo(table.get(), partition_pb));
  }

  // Add the table/tablets to the in-memory map for the assignment.
  table->AddTablets(*tablets);
  for (TabletInfo* tablet : *tablets) {
    InsertOrDie(&tablet_map_, tablet->tablet_id(), tablet);
  }
  return Status::OK();
}

TabletInfo* CatalogManager::CreateTabletInfo(TableInfo* table,
                                             const PartitionPB& partition) {
  TabletInfo* tablet = new TabletInfo(table, GenerateId());
  tablet->mutable_metadata()->StartMutation();
  SysTabletsEntryPB *metadata = &tablet->mutable_metadata()->mutable_dirty()->pb;
  metadata->set_state(SysTabletsEntryPB::PREPARING);
  metadata->mutable_partition()->CopyFrom(partition);
  metadata->set_table_id(table->id());
  // This is important: we are setting the first table id in the table_ids list
  // to be the id of the original table that creates the tablet.
  metadata->add_table_ids(table->id());
  return tablet;
}

void CatalogManager::HandleAssignPreparingTablet(TabletInfo* tablet,
                                                 DeferredAssignmentActions* deferred) {
  // The tablet was just created (probably by a CreateTable RPC).
  // Update the state to "creating" to be ready for the creation request.
  tablet->mutable_metadata()->mutable_dirty()->set_state(
    SysTabletsEntryPB::CREATING, "Sending initial creation of tablet");
  deferred->tablets_to_update.push_back(tablet);
  deferred->needs_create_rpc.push_back(tablet);
  VLOG(1) << "Assign new tablet " << tablet->ToString();
}

void CatalogManager::HandleAssignCreatingTablet(TabletInfo* tablet,
                                                DeferredAssignmentActions* deferred,
                                                vector<scoped_refptr<TabletInfo>>* new_tablets) {
  MonoDelta time_since_updated =
      MonoTime::Now().GetDeltaSince(tablet->last_update_time());
  int64_t remaining_timeout_ms =
      FLAGS_tablet_creation_timeout_ms - time_since_updated.ToMilliseconds();

  // Skip the tablet if the assignment timeout is not yet expired.
  if (remaining_timeout_ms > 0) {
    VLOG(2) << "Tablet " << tablet->ToString() << " still being created. "
            << remaining_timeout_ms << "ms remain until timeout.";
    return;
  }

  const PersistentTabletInfo& old_info = tablet->metadata().state();

  // The "tablet creation" was already sent, but we didn't receive an answer
  // within the timeout. So the tablet will be replaced by a new one.
  TabletInfo *replacement = CreateTabletInfo(tablet->table().get(),
                                             old_info.pb.partition());
  LOG(WARNING) << "Tablet " << tablet->ToString() << " was not created within "
               << "the allowed timeout. Replacing with a new tablet "
               << replacement->tablet_id();

  tablet->table()->AddTablet(replacement);
  {
    std::lock_guard<LockType> l_maps(lock_);
    tablet_map_[replacement->tablet_id()] = replacement;
  }

  // Mark old tablet as replaced.
  tablet->mutable_metadata()->mutable_dirty()->set_state(
    SysTabletsEntryPB::REPLACED,
    Substitute("Replaced by $0 at $1",
               replacement->tablet_id(), LocalTimeAsString()));

  // Mark new tablet as being created.
  replacement->mutable_metadata()->mutable_dirty()->set_state(
    SysTabletsEntryPB::CREATING,
    Substitute("Replacement for $0", tablet->tablet_id()));

  deferred->tablets_to_update.push_back(tablet);
  deferred->tablets_to_add.push_back(replacement);
  deferred->needs_create_rpc.push_back(replacement);
  VLOG(1) << "Replaced tablet " << tablet->tablet_id()
          << " with " << replacement->tablet_id()
          << " (table " << tablet->table()->ToString() << ")";

  new_tablets->push_back(replacement);
}
```
