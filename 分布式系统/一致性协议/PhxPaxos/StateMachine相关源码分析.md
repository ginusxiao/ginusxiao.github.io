# 提纲
[toc]

# StateMachine基础
## SMFac类基础
### SMFac类定义

```
class SMFac
{
private:
    /*该group中挂载的所有的状态机*/
    std::vector<StateMachine *> m_vecSMList;
    /*该group的编号*/
    int m_iMyGroupIdx;
};
```


### SMFac类构造函数

```
SMFac :: SMFac(const int iMyGroupIdx) : m_iMyGroupIdx(iMyGroupIdx)
{
}
```

在SMFac构造函数中设置了group索引号，但是没有设置@m_vecSMList啊，那在哪里设置的呢？参考Node :: RunNode -> PNode :: Init -> PNode :: InitStateMachine -> PNode :: AddStateMachine -> Group :: AddStateMachine -> Instance :: AddStateMachine -> SMFac :: AddSM调用链，也就是在PNode :: InitStateMachine阶段就已经将相关状态机添加到各group中了。


## SMFac类方法
### 获取当前group中执行了checkpoint的最大InstanceID

```
const uint64_t SMFac :: GetCheckpointInstanceID(const int iGroupIdx) const
{
    uint64_t llCPInstanceID = -1;
    uint64_t llCPInstanceID_Insize = -1;
    bool bHaveUseSM = false;

    /*遍历@m_vecSMList中的所有的StateMachine*/
    for (auto & poSM : m_vecSMList)
    {
        /* 获取当前StateMachine已经执行checkpint的最大InstanceID，该方法是虚函数，
         * 如果用户自己的状态机支持checkpoint，就需要实现该函数，PhxPaxos中默认
         * 返回-1
         */
        uint64_t llCheckpointInstanceID = poSM->GetCheckpointInstanceID(iGroupIdx);
        if (poSM->SMID() == SYSTEM_V_SMID
                || poSM->SMID() == MASTER_V_SMID)
        {
            //system variables 
            //master variables
            //if no user state machine, system and master's can use.
            //if have user state machine, use user'state machine's checkpointinstanceid.
            if (llCheckpointInstanceID == uint64_t(-1))
            {
                continue;
            }
            
            /*如果是system variable或者master variable对应的状态机，则设置@llCPInstanceID_Insize*/
            if (llCheckpointInstanceID > llCPInstanceID_Insize
                    || llCPInstanceID_Insize == (uint64_t)-1)
            {
                llCPInstanceID_Insize = llCheckpointInstanceID;
            }

            continue;
        }

        /*用户注册了自己的状态机*/
        bHaveUseSM = true;

        if (llCheckpointInstanceID == uint64_t(-1))
        {
            continue;
        }
        
        /*设置@llCPInstanceID，表示当前group中执行了checkpoint的最大InstanceID*/
        if (llCheckpointInstanceID > llCPInstanceID
                || llCPInstanceID == (uint64_t)-1)
        {
            llCPInstanceID = llCheckpointInstanceID;
        }
    }
    
    /*如果用户注册了自己的状态机，则返回@llCPInstanceID，否则返回@llCPInstanceID_Insize*/
    return bHaveUseSM ? llCPInstanceID : llCPInstanceID_Insize;
}
```

### 执行Checkpoint
```
bool SMFac :: ExecuteForCheckpoint(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sPaxosValue)
{
    if (sPaxosValue.size() < sizeof(int))
    {
        PLG1Err("Value wrong, instanceid %lu size %zu", llInstanceID, sPaxosValue.size());
        //need do nothing, just skip
        return true;
    }

    int iSMID = 0;
    memcpy(&iSMID, sPaxosValue.data(), sizeof(int));

    if (iSMID == 0)
    {
        PLG1Imp("Value no need to do sm, just skip, instanceid %lu", llInstanceID);
        return true;
    }

    std::string sBodyValue = string(sPaxosValue.data() + sizeof(int), sPaxosValue.size() - sizeof(int));
    if (iSMID == BATCH_PROPOSE_SMID)
    {
        /*在多个StateMachine上批量checkpoint*/
        return BatchExecuteForCheckpoint(iGroupIdx, llInstanceID, sBodyValue);
    }
    else
    {
        /*在单一StateMachine上执行checkpoint*/
        return DoExecuteForCheckpoint(iGroupIdx, llInstanceID, sBodyValue, iSMID);
    }
}

bool SMFac :: BatchExecuteForCheckpoint(const int iGroupIdx, const uint64_t llInstanceID, 
        const std::string & sBodyValue)
{
    BatchPaxosValues oBatchValues;
    bool bSucc = oBatchValues.ParseFromArray(sBodyValue.data(), sBodyValue.size());
    if (!bSucc)
    {
        PLG1Err("ParseFromArray fail, valuesize %zu", sBodyValue.size());
        return false;
    }

    /*分别在每个StateMachine上执行checkpoint*/
    for (int i = 0; i < oBatchValues.values_size(); i++)
    {
        const PaxosValue & oValue = oBatchValues.values(i);
        bool bExecuteSucc = DoExecuteForCheckpoint(iGroupIdx, llInstanceID, oValue.value(), oValue.smid());
        if (!bExecuteSucc)
        {
            return false;
        }
    }

    return true;
}

bool SMFac :: DoExecuteForCheckpoint(const int iGroupIdx, const uint64_t llInstanceID, 
        const std::string & sBodyValue, const int iSMID)
{
    if (iSMID == 0)
    {
        PLG1Imp("Value no need to do sm, just skip, instanceid %lu", llInstanceID);
        return true;
    }

    if (m_vecSMList.size() == 0)
    {
        PLG1Imp("No any sm, need wait sm, instanceid %lu", llInstanceID);
        return false;
    }

    for (auto & poSM : m_vecSMList)
    {
        /*找到对应的StateMachine，执行checkpoint*/
        if (poSM->SMID() == iSMID)
        {
            return poSM->ExecuteForCheckpoint(iGroupIdx, llInstanceID, sBodyValue);
        }
    }

    PLG1Err("Unknown smid %d instanceid %lu", iSMID, llInstanceID);

    return false;
}
```


### 在状态机上运用决议

```
bool SMFac :: Execute(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sPaxosValue, SMCtx * poSMCtx)
{
    if (sPaxosValue.size() < sizeof(int))
    {
        PLG1Err("Value wrong, instanceid %lu size %zu", llInstanceID, sPaxosValue.size());
        //need do nothing, just skip
        return true;
    }

    /*从@sPaxosValue中解析出SMID*/
    int iSMID = 0;
    memcpy(&iSMID, sPaxosValue.data(), sizeof(int));

    if (iSMID == 0)
    {
        PLG1Imp("Value no need to do sm, just skip, instanceid %lu", llInstanceID);
        return true;
    }

    std::string sBodyValue = string(sPaxosValue.data() + sizeof(int), sPaxosValue.size() - sizeof(int));
    if (iSMID == BATCH_PROPOSE_SMID)
    {
        BatchSMCtx * poBatchSMCtx = nullptr;
        if (poSMCtx != nullptr && poSMCtx->m_pCtx != nullptr)
        {
            poBatchSMCtx = (BatchSMCtx *)poSMCtx->m_pCtx;
        }
        return BatchExecute(iGroupIdx, llInstanceID, sBodyValue, poBatchSMCtx);
    }
    else
    {
        return DoExecute(iGroupIdx, llInstanceID, sBodyValue, iSMID, poSMCtx);
    }
}

bool SMFac :: BatchExecute(const int iGroupIdx, const uint64_t llInstanceID, const std::string & sBodyValue, BatchSMCtx * poBatchSMCtx)
{
    /*从@sBodyValue中解析出@oBatchValues*/
    BatchPaxosValues oBatchValues;
    bool bSucc = oBatchValues.ParseFromArray(sBodyValue.data(), sBodyValue.size());
    if (!bSucc)
    {
        PLG1Err("ParseFromArray fail, valuesize %zu", sBodyValue.size());
        return false;
    }

    if (poBatchSMCtx != nullptr) 
    {
        /*@oBatchValues中的每一个value和@poBatchSMCtx中的每一个SMCtx一一对应*/    
        if ((int)poBatchSMCtx->m_vecSMCtxList.size() != oBatchValues.values_size())
        {
            PLG1Err("values size %d not equal to smctx size %zu",
                    oBatchValues.values_size(), poBatchSMCtx->m_vecSMCtxList.size());
            return false;
        }
    }

    /*逐一在每个StateMachine上运用这些决议*/
    for (int i = 0; i < oBatchValues.values_size(); i++)
    {
        const PaxosValue & oValue = oBatchValues.values(i);
        SMCtx * poSMCtx = poBatchSMCtx != nullptr ? poBatchSMCtx->m_vecSMCtxList[i] : nullptr;
        bool bExecuteSucc = DoExecute(iGroupIdx, llInstanceID, oValue.value(), oValue.smid(), poSMCtx);
        if (!bExecuteSucc)
        {
            return false;
        }
    }

    return true;
}

bool SMFac :: DoExecute(const int iGroupIdx, const uint64_t llInstanceID, 
        const std::string & sBodyValue, const int iSMID, SMCtx * poSMCtx)
{
    if (iSMID == 0)
    {
        PLG1Imp("Value no need to do sm, just skip, instanceid %lu", llInstanceID);
        return true;
    }

    /*该SMFac中不包含任何StateMachine*/
    if (m_vecSMList.size() == 0)
    {
        PLG1Imp("No any sm, need wait sm, instanceid %lu", llInstanceID);
        return false;
    }

    /*找到@iSMID所标识的StateMachine并在其上运用决议*/
    for (auto & poSM : m_vecSMList)
    {
        if (poSM->SMID() == iSMID)
        {
            /* 执行状态机的状态转移函数，该函数由提供状态机的应用提供（每个状态机都会
             * 实现Execute方法）
             */
            return poSM->Execute(iGroupIdx, llInstanceID, sBodyValue, poSMCtx);
        }
    }

    PLG1Err("Unknown smid %d instanceid %lu", iSMID, llInstanceID);
    return false;
}


```
