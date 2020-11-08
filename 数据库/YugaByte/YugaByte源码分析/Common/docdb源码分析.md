# 提纲
[toc]

## 

```
TabletServerServiceIf::Handle
    - TabletServiceImpl::Write
        - auto operation_state = std::make_unique<WriteOperationState>(tablet.peer->tablet(), req, resp)
        - operation_state->set_completion_callback(std::make_unique<WriteOperationCompletionCallback>(...))
        - TabletPeer::WriteAsync(std::move(operation_state), tablet.leader_term, context_ptr->GetClientDeadline())
            - auto operation = std::make_unique<WriteOperation>(std::move(state), term, deadline, this)
            - Tablet::AcquireLocksAndPerformDocOperations(std::move(operation))
                - Tablet::KeyValueBatchFromQLWriteBatch(std::move(operation))
```

                