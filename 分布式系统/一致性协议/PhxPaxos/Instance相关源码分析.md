# 提纲
[toc]

# Instance类基础
## Instance类定义

```
class Instance
{
private:
    /*配置，直接来自于Group::m_oConfig*/
    Config * m_poConfig;
    /*通信相关，直接来自于Group::m_oCommunicate*/
    MsgTransport * m_poMsgTransport;

    /* 每一个Group都包含一个实例Instance（见Group::m_oInstance），每个Group中
     * 都可以挂多个状态机，这些状态机实际上都是在这里管理着
     */
    SMFac m_oSMFac;

    IOLoop m_oIOLoop;

    /*Acceptor, Learner和Proposer*/
    Acceptor m_oAcceptor;
    Learner m_oLearner;
    Proposer m_oProposer;

    /*对LogStorage指针的封装*/
    PaxosLog m_oPaxosLog;

    uint32_t m_iLastChecksum;

private:
    CommitCtx m_oCommitCtx;
    uint32_t m_iCommitTimerID;

    Committer m_oCommitter;

private:
    CheckpointMgr m_oCheckpointMgr;

private:
    TimeStat m_oTimeStat;
    Options m_oOptions;

    bool m_bStarted;
};
```

## Instance类构造函数

```
Instance :: Instance(
        const Config * poConfig, 
        const LogStorage * poLogStorage,
        const MsgTransport * poMsgTransport,
        const Options & oOptions)
    : m_oSMFac(poConfig->GetMyGroupIdx()),
    m_oIOLoop((Config *)poConfig, this),
    m_oAcceptor(poConfig, poMsgTransport, this, poLogStorage), 
    m_oLearner(poConfig, poMsgTransport, this, &m_oAcceptor, poLogStorage, &m_oIOLoop, &m_oCheckpointMgr, &m_oSMFac),
    m_oProposer(poConfig, poMsgTransport, this, &m_oLearner, &m_oIOLoop),
    m_oPaxosLog(poLogStorage),
    m_oCommitCtx((Config *)poConfig),
    m_oCommitter((Config *)poConfig, &m_oCommitCtx, &m_oIOLoop, &m_oSMFac),
    m_oCheckpointMgr((Config *)poConfig, &m_oSMFac, (LogStorage *)poLogStorage, oOptions.bUseCheckpointReplayer),
    m_oOptions(oOptions), m_bStarted(false)
{
    m_poConfig = (Config *)poConfig;
    m_poMsgTransport = (MsgTransport *)poMsgTransport;
    m_iCommitTimerID = 0;
    m_iLastChecksum = 0;
}
```

# Instance类方法
## Instance 初始化

```
int Instance :: Init()
{
    //Must init acceptor first, because the max instanceid is record in acceptor state.
    /*初始化Acceptor*/
    int ret = m_oAcceptor.Init();
    if (ret != 0)
    {
        PLGErr("Acceptor.Init fail, ret %d", ret);
        return ret;
    }

    /*初始化CheckpointMgr*/
    ret = m_oCheckpointMgr.Init();
    if (ret != 0)
    {
        PLGErr("CheckpointMgr.Init fail, ret %d", ret);
        return ret;
    }

    /* 获取已经执行了checkpoint的最大InstanceID的下一个ID，表示接下来需要执行
     * checkpoint的最大InstanceID
     */
    uint64_t llCPInstanceID = m_oCheckpointMgr.GetCheckpointInstanceID() + 1;

    PLGImp("Acceptor.OK, Log.InstanceID %lu Checkpoint.InstanceID %lu", 
            m_oAcceptor.GetInstanceID(), llCPInstanceID);

    /*从@llCPInstanceID开始将各实例的决议运用到状态机中*/
    uint64_t llNowInstanceID = llCPInstanceID;
    if (llNowInstanceID < m_oAcceptor.GetInstanceID())
    {
        /* @llNowInstanceID表示接下来第一个要执行checkpoint的InstanceID，
         * @m_oAcceptor.GetInstanceID()表示最后一个要执行checkpoint的InstanceID
         */
        ret = PlayLog(llNowInstanceID, m_oAcceptor.GetInstanceID());
        if (ret != 0)
        {
            return ret;
        }

        PLGImp("PlayLog OK, begin instanceid %lu end instanceid %lu", llNowInstanceID, m_oAcceptor.GetInstanceID());

        /*当前最大的InstanceID*/
        llNowInstanceID = m_oAcceptor.GetInstanceID();
    }
    else
    {
        if (llNowInstanceID > m_oAcceptor.GetInstanceID())
        {
            ret = ProtectionLogic_IsCheckpointInstanceIDCorrect(llNowInstanceID, m_oAcceptor.GetInstanceID());
            if (ret != 0)
            {
                return ret;
            }
            m_oAcceptor.InitForNewPaxosInstance();
        }
        
        m_oAcceptor.SetInstanceID(llNowInstanceID);
    }

    PLGImp("NowInstanceID %lu", llNowInstanceID);

    /*更新Learner、Proposer中相关信息*/
    m_oLearner.SetInstanceID(llNowInstanceID);
    m_oProposer.SetInstanceID(llNowInstanceID);
    m_oProposer.SetStartProposalID(m_oAcceptor.GetAcceptorState()->GetPromiseBallot().m_llProposalID + 1);

    /*更新已经执行checkpoint的最大的InstanceID*/
    m_oCheckpointMgr.SetMaxChosenInstanceID(llNowInstanceID);

    ret = InitLastCheckSum();
    if (ret != 0)
    {
        return ret;
    }

    /*重新为learner添加定时器，定时learn*/
    m_oLearner.Reset_AskforLearn_Noop();

    PLGImp("OK");

    return 0;
}

```

## 运用指定范围的Instance的决议到状态机

```
int Instance :: PlayLog(const uint64_t llBeginInstanceID, const uint64_t llEndInstanceID)
{
    if (llBeginInstanceID < m_oCheckpointMgr.GetMinChosenInstanceID())
    {
        PLGErr("now instanceid %lu small than min chosen instanceid %lu", 
                llBeginInstanceID, m_oCheckpointMgr.GetMinChosenInstanceID());
        return -2;
    }

    for (uint64_t llInstanceID = llBeginInstanceID; llInstanceID < llEndInstanceID; llInstanceID++)
    {
        /*读取每个InstanceID对应的AcceptorState*/
        AcceptorStateData oState; 
        int ret = m_oPaxosLog.ReadState(m_poConfig->GetMyGroupIdx(), llInstanceID, oState);
        if (ret != 0)
        {
            PLGErr("log read fail, instanceid %lu ret %d", llInstanceID, ret);
            return ret;
        }

        /*将每个InstanceID对应的决议运用到状态机中，建SMFac::Execute*/
        bool bExecuteRet = m_oSMFac.Execute(m_poConfig->GetMyGroupIdx(), llInstanceID, oState.acceptedvalue(), nullptr);
        if (!bExecuteRet)
        {
            PLGErr("Execute fail, instanceid %lu", llInstanceID);
            return -1;
        }
    }

    return 0;
}
```

## Instance启动

```
void Instance :: Start()
{
    //start learner sender
    m_oLearner.StartLearnerSender();
    //start ioloop
    m_oIOLoop.start();
    //start checkpoint replayer and cleaner
    m_oCheckpointMgr.Start();

    m_bStarted = true;
}
```

## Instance检查是否有新的提议

```
void Instance :: CheckNewValue()
{
    if (!m_oCommitCtx.IsNewCommit())
    {
        return;
    }

    if (!m_oLearner.IsIMLatest())
    {
        return;
    }

    if (m_poConfig->IsIMFollower())
    {
        PLGErr("I'm follower, skip this new value");
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Follower_Cannot_Commit);
        return;
    }

    if (!m_poConfig->CheckConfig())
    {
        PLGErr("I'm not in membership, skip this new value");
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Im_Not_In_Membership);
        return;
    }

    if ((int)m_oCommitCtx.GetCommitValue().size() > MAX_VALUE_SIZE)
    {
        PLGErr("value size %zu to large, skip this new value",
            m_oCommitCtx.GetCommitValue().size());
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Value_Size_TooLarge);
        return;
    }

    m_oCommitCtx.StartCommit(m_oProposer.GetInstanceID());

    /*添加一个Instance Commit定时器*/
    if (m_oCommitCtx.GetTimeoutMs() != -1)
    {
        m_oIOLoop.AddTimer(m_oCommitCtx.GetTimeoutMs(), Timer_Instance_Commit_Timeout, m_iCommitTimerID);
    }
    
    m_oTimeStat.Point();

    if (m_poConfig->GetIsUseMembership()
            && (m_oProposer.GetInstanceID() == 0 || m_poConfig->GetGid() == 0))
    {
        //Init system variables.
        /*初始化系统变量*/
        PLGHead("Need to init system variables, Now.InstanceID %lu Now.Gid %lu", 
                m_oProposer.GetInstanceID(), m_poConfig->GetGid());

        uint64_t llGid = OtherUtils::GenGid(m_poConfig->GetMyNodeID());
        string sInitSVOpValue;
        int ret = m_poConfig->GetSystemVSM()->CreateGid_OPValue(llGid, sInitSVOpValue);
        assert(ret == 0);

        m_oSMFac.PackPaxosValue(sInitSVOpValue, m_poConfig->GetSystemVSM()->SMID());
        /*提议*/
        m_oProposer.NewValue(sInitSVOpValue);
    }
    else
    {
        if (m_oOptions.bOpenChangeValueBeforePropose) {
            m_oSMFac.BeforePropose(m_poConfig->GetMyGroupIdx(), m_oCommitCtx.GetCommitValue());
        }
        
        /*提议*/
        m_oProposer.NewValue(m_oCommitCtx.GetCommitValue());
    }
}
```

## 处理从网络接收到的消息

```
int Instance :: OnReceiveMessage(const char * pcMessage, const int iMessageLen)
{
    /*将接收到的消息添加到IOLoop等待队列中*/
    m_oIOLoop.AddMessage(pcMessage, iMessageLen);

    return 0;
}
```

## 处理被（IOLoop）调度到的消息

```
void Instance :: OnReceive(const std::string & sBuffer)
{
    BP->GetInstanceBP()->OnReceive();

    if (sBuffer.size() <= 6)
    {
        PLGErr("buffer size %zu too short", sBuffer.size());
        return;
    }

    Header oHeader;
    size_t iBodyStartPos = 0;
    size_t iBodyLen = 0;
    /* 反序列化消息，从消息@sBuffer中解析出header部分@oHeader，body部分的起点
     * @iBodyStartPos和body部分的长度@iBodyLen
     */
    int ret = Base::UnPackBaseMsg(sBuffer, oHeader, iBodyStartPos, iBodyLen);
    if (ret != 0)
    {
        return;
    }

    /*获取消息类型*/
    int iCmd = oHeader.cmdid();
    if (iCmd == MsgCmd_PaxosMsg)
    {
        /*处理paxos消息*/
        if (m_oCheckpointMgr.InAskforcheckpointMode())
        {
            PLGImp("in ask for checkpoint mode, ignord paxosmsg");
            return;
        }
        
        PaxosMsg oPaxosMsg;
        /*解析出PaxosMsg*/
        bool bSucc = oPaxosMsg.ParseFromArray(sBuffer.data() + iBodyStartPos, iBodyLen);
        if (!bSucc)
        {
            /*解析失败*/
            BP->GetInstanceBP()->OnReceiveParseError();
            PLGErr("PaxosMsg.ParseFromArray fail, skip this msg");
            return;
        }

        /*检查@oHeader中的gid是否正确（gid是在Base :: PackBaseMsg中设置的）*/
        if (!ReceiveMsgHeaderCheck(oHeader, oPaxosMsg.nodeid()))
        {
            return;
        }
        
        /*处理Paxos协议相关消息*/
        OnReceivePaxosMsg(oPaxosMsg);
    }
    else if (iCmd == MsgCmd_CheckpointMsg)
    {
        /*处理checkpoint消息*/
        CheckpointMsg oCheckpointMsg;
        /*解析出CheckpointMsg*/
        bool bSucc = oCheckpointMsg.ParseFromArray(sBuffer.data() + iBodyStartPos, iBodyLen);
        if (!bSucc)
        {
            BP->GetInstanceBP()->OnReceiveParseError();
            PLGErr("PaxosMsg.ParseFromArray fail, skip this msg");
            return;
        }

        /*检查@oHeader中的gid是否正确（gid是在Base :: PackBaseMsg中设置的）*/
        if (!ReceiveMsgHeaderCheck(oHeader, oCheckpointMsg.nodeid()))
        {
            return;
        }
        
        /*处理Checkpoint相关消息*/
        OnReceiveCheckpointMsg(oCheckpointMsg);
    }
}
```

## 处理Paxos协议相关消息

```
int Instance :: OnReceivePaxosMsg(const PaxosMsg & oPaxosMsg, const bool bIsRetry)
{
    BP->GetInstanceBP()->OnReceivePaxosMsg();

    PLGImp("Now.InstanceID %lu Msg.InstanceID %lu MsgType %d Msg.from_nodeid %lu My.nodeid %lu Seen.LatestInstanceID %lu",
            m_oProposer.GetInstanceID(), oPaxosMsg.instanceid(), oPaxosMsg.msgtype(),
            oPaxosMsg.nodeid(), m_poConfig->GetMyNodeID(), m_oLearner.GetSeenLatestInstanceID());

    if (oPaxosMsg.msgtype() == MsgType_PaxosPrepareReply
            || oPaxosMsg.msgtype() == MsgType_PaxosAcceptReply
            || oPaxosMsg.msgtype() == MsgType_PaxosProposal_SendNewValue)
    {
        /*处理发送给Proposer的消息*/
        
        /*检查成员组中是否包含该node*/
        if (!m_poConfig->IsValidNodeID(oPaxosMsg.nodeid()))
        {
            BP->GetInstanceBP()->OnReceivePaxosMsgNodeIDNotValid();
            PLGErr("acceptor reply type msg, from nodeid not in my membership, skip this message");
            return 0;
        }
        
        /*交由Proposer处理*/
        return ReceiveMsgForProposer(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosPrepare
            || oPaxosMsg.msgtype() == MsgType_PaxosAccept)
    {
        /*处理Acceptor接收到的消息*/
        
        /* 如果当前的gid为0，则将oPaxosMsg.nodeid()这个node添加到
         * @m_mapTmpNodeOnlyForLearn中
         */
        if (m_poConfig->GetGid() == 0)
        {
            m_poConfig->AddTmpNodeOnlyForLearn(oPaxosMsg.nodeid());
        }
        
        /* 检查成员组中是否包含该node，如果不在，则将oPaxosMsg.nodeid()这个node添加到
         * @m_mapTmpNodeOnlyForLearn中
         */
        if ((!m_poConfig->IsValidNodeID(oPaxosMsg.nodeid())))
        {
            PLGErr("prepare/accept type msg, from nodeid not in my membership(or i'm null membership), "
                    "skip this message and add node to tempnode, my gid %lu",
                    m_poConfig->GetGid());

            m_poConfig->AddTmpNodeOnlyForLearn(oPaxosMsg.nodeid());

            return 0;
        }

        ChecksumLogic(oPaxosMsg);
        /*交由Acceptor处理*.
        return ReceiveMsgForAcceptor(oPaxosMsg, bIsRetry);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforLearn
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_ProposerSendSuccess
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_ComfirmAskforLearn
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendNowInstanceID
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue_Ack
            || oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforCheckpoint)
    {
        /*处理Learner相关的消息*/
        
        ChecksumLogic(oPaxosMsg);
        /*交由Learner处理*/
        return ReceiveMsgForLearner(oPaxosMsg);
    }
    else
    {
        BP->GetInstanceBP()->OnReceivePaxosMsgTypeNotValid();
        PLGErr("Invaid msgtype %d", oPaxosMsg.msgtype());
    }

    return 0;
}
```

## 转交请求给Acceptor处理
```
int Instance :: ReceiveMsgForAcceptor(const PaxosMsg & oPaxosMsg, const bool bIsRetry)
{
    if (m_poConfig->IsIMFollower())
    {
        PLGErr("I'm follower, skip this message");
        return 0;
    }
    
    if (oPaxosMsg.instanceid() != m_oAcceptor.GetInstanceID())
    {
        BP->GetInstanceBP()->OnReceivePaxosAcceptorMsgInotsame();
    }
    
    if (oPaxosMsg.instanceid() == m_oAcceptor.GetInstanceID() + 1)
    {
        //skip success message
        PaxosMsg oNewPaxosMsg = oPaxosMsg;
        oNewPaxosMsg.set_instanceid(m_oAcceptor.GetInstanceID());
        oNewPaxosMsg.set_msgtype(MsgType_PaxosLearner_ProposerSendSuccess);

        ReceiveMsgForLearner(oNewPaxosMsg);
    }
            
    if (oPaxosMsg.instanceid() == m_oAcceptor.GetInstanceID())
    {
        if (oPaxosMsg.msgtype() == MsgType_PaxosPrepare)
        {
            /*处理Prepare消息*/
            return m_oAcceptor.OnPrepare(oPaxosMsg);
        }
        else if (oPaxosMsg.msgtype() == MsgType_PaxosAccept)
        {
            /*处理Accept消息*/
            m_oAcceptor.OnAccept(oPaxosMsg);
        }
    }
    else if ((!bIsRetry) && (oPaxosMsg.instanceid() > m_oAcceptor.GetInstanceID()))
    {
        //retry msg can't retry again.
        if (oPaxosMsg.instanceid() >= m_oLearner.GetSeenLatestInstanceID())
        {
            if (oPaxosMsg.instanceid() < m_oAcceptor.GetInstanceID() + RETRY_QUEUE_MAX_LEN)
            {
                //need retry msg precondition
                //1. prepare or accept msg
                //2. msg.instanceid > nowinstanceid. 
                //    (if < nowinstanceid, this msg is expire)
                //3. msg.instanceid >= seen latestinstanceid. 
                //    (if < seen latestinstanceid, proposer don't need reply with this instanceid anymore.)
                //4. msg.instanceid close to nowinstanceid.
                m_oIOLoop.AddRetryPaxosMsg(oPaxosMsg);
                
                BP->GetInstanceBP()->OnReceivePaxosAcceptorMsgAddRetry();

                //PLGErr("InstanceID not same, get in to retry logic");
            }
            else
            {
                //retry msg not series, no use.
                m_oIOLoop.ClearRetryQueue();
            }
        }
    }

    return 0;
}
```

## 转交请求给Proposer处理
```
int Instance :: ReceiveMsgForProposer(const PaxosMsg & oPaxosMsg)
{
    if (m_poConfig->IsIMFollower())
    {
        PLGErr("I'm follower, skip this message");
        return 0;
    }

    if (oPaxosMsg.instanceid() != m_oProposer.GetInstanceID())
    {
        if (oPaxosMsg.instanceid() + 1 == m_oProposer.GetInstanceID())
        {
            /*处理过期的Prepare/Accept响应*/
            
            //Exipred reply msg on last instance.
            //If the response of a node is always slower than the majority node, 
            //then the message of the node is always ignored even if it is a reject reply.
            //In this case, if we do not deal with these reject reply, the node that 
            //gave reject reply will always give reject reply. 
            //This causes the node to remain in catch-up state.
            //
            //To avoid this problem, we need to deal with the expired reply.
            if (oPaxosMsg.msgtype() == MsgType_PaxosPrepareReply)
            {
                m_oProposer.OnExpiredPrepareReply(oPaxosMsg);
            }
            else if (oPaxosMsg.msgtype() == MsgType_PaxosAcceptReply)
            {
                m_oProposer.OnExpiredAcceptReply(oPaxosMsg);
            }
        }

        BP->GetInstanceBP()->OnReceivePaxosProposerMsgInotsame();
        //PLGErr("InstanceID not same, skip msg");
        return 0;
    }

    if (oPaxosMsg.msgtype() == MsgType_PaxosPrepareReply)
    {
        /*处理未过期的Prepare响应*/
        m_oProposer.OnPrepareReply(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosAcceptReply)
    {
        /*处理未过期的Accept响应*/
        m_oProposer.OnAcceptReply(oPaxosMsg);
    }

    return 0;
}
```

## 转交请求给Learner处理

```
int Instance :: ReceiveMsgForLearner(const PaxosMsg & oPaxosMsg)
{
    if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforLearn)
    {
        /*处理请求学习决议的Learner发送过来的“AskForLearn”消息*/
        m_oLearner.OnAskforLearn(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue)
    {
        /*处理被学习的Learner发送过来的“决议值”的消息*/
        m_oLearner.OnSendLearnValue(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_ProposerSendSuccess)
    {
        /*处理来自于Proposer的“本次提案成功”的消息*/
        m_oLearner.OnProposerSendSuccess(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendNowInstanceID)
    {
        /* 处理被学习的Learner发送回来的关于“AskForLearn的响应”
         * （被学习的Learner的最大InstanceID）
         */
        m_oLearner.OnSendNowInstanceID(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_ComfirmAskforLearn)
    {
        /*处理请求学习决议的Learner的关于“AskForLearn的Confirm”消息*/
        m_oLearner.OnComfirmAskForLearn(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue_Ack)
    {
        /* 处理来自于请求学习决议的Learner关于“已经成功学习到某个Instance的决议值的Ack”
         * 消息，或者来自于Follower的关于“已经成功学习到某个Instance的决议值的Ack”消息
         */
        m_oLearner.OnSendLearnValue_Ack(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforCheckpoint)
    {
        m_oLearner.OnAskforCheckpoint(oPaxosMsg);
    }

    if (m_oLearner.IsLearned())
    {
        BP->GetInstanceBP()->OnInstanceLearned();

        SMCtx * poSMCtx = nullptr;
        bool bIsMyCommit = m_oCommitCtx.IsMyCommit(m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue(), poSMCtx);

        if (!bIsMyCommit)
        {
            BP->GetInstanceBP()->OnInstanceLearnedNotMyCommit();
            PLGDebug("this value is not my commit");
        }
        else
        {
            int iUseTimeMs = m_oTimeStat.Point();
            BP->GetInstanceBP()->OnInstanceLearnedIsMyCommit(iUseTimeMs);
            PLGHead("My commit ok, usetime %dms", iUseTimeMs);
        }

        if (!SMExecute(m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue(), bIsMyCommit, poSMCtx))
        {
            BP->GetInstanceBP()->OnInstanceLearnedSMExecuteFail();

            PLGErr("SMExecute fail, instanceid %lu, not increase instanceid", m_oLearner.GetInstanceID());
            m_oCommitCtx.SetResult(PaxosTryCommitRet_ExecuteFail, 
                    m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue());

            m_oProposer.CancelSkipPrepare();

            return -1;
        }
        
        {
            //this paxos instance end, tell proposal done
            m_oCommitCtx.SetResult(PaxosTryCommitRet_OK
                    , m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue());

            if (m_iCommitTimerID > 0)
            {
                m_oIOLoop.RemoveTimer(m_iCommitTimerID);
            }
        }
        
        PLGHead("[Learned] New paxos starting, Now.Proposer.InstanceID %lu "
                "Now.Acceptor.InstanceID %lu Now.Learner.InstanceID %lu",
                m_oProposer.GetInstanceID(), m_oAcceptor.GetInstanceID(), m_oLearner.GetInstanceID());
        
        PLGHead("[Learned] Checksum change, last checksum %u new checksum %u",
                m_iLastChecksum, m_oLearner.GetNewChecksum());

        m_iLastChecksum = m_oLearner.GetNewChecksum();

        NewInstance();

        PLGHead("[Learned] New paxos instance has started, Now.Proposer.InstanceID %lu "
                "Now.Acceptor.InstanceID %lu Now.Learner.InstanceID %lu",
                m_oProposer.GetInstanceID(), m_oAcceptor.GetInstanceID(), m_oLearner.GetInstanceID());

        m_oCheckpointMgr.SetMaxChosenInstanceID(m_oAcceptor.GetInstanceID());

        BP->GetInstanceBP()->NewInstance();
    }

    return 0;
}
```

## 处理Checkpoint相关消息

```
void Instance :: OnReceiveCheckpointMsg(const CheckpointMsg & oCheckpointMsg)
{
    if (oCheckpointMsg.msgtype() == CheckpointMsgType_SendFile)
    {
        /*CheckpointReceiver接收到CheckpointSender发送过来的数据*/
        if (!m_oCheckpointMgr.InAskforcheckpointMode())
        {
            PLGImp("not in ask for checkpoint mode, ignord checkpoint msg");
            return;
        }

        m_oLearner.OnSendCheckpoint(oCheckpointMsg);
    }
    else if (oCheckpointMsg.msgtype() == CheckpointMsgType_SendFile_Ack)
    {
        /*CheckpointSender接收到CheckpointReceiver发送过来的Ack消息*/
        m_oLearner.OnSendCheckpointAck(oCheckpointMsg);
    }
}
```

## 处理超时事件

```
void Instance :: OnTimeout(const uint32_t iTimerID, const int iType)
{
    if (iType == Timer_Proposer_Prepare_Timeout)
    {
        /*Prepare 超时，重新发起Prepare*/
        m_oProposer.OnPrepareTimeout();
    }
    else if (iType == Timer_Proposer_Accept_Timeout)
    {
        /*Accept 超时，重新发起Accept*/
        m_oProposer.OnAcceptTimeout();
    }
    else if (iType == Timer_Learner_Askforlearn_noop)
    {
        /*AskForLearn 超时*/
        m_oLearner.AskforLearn_Noop();
    }
    else if (iType == Timer_Instance_Commit_Timeout)
    {
        /*NewValueCommit 超时*/
        OnNewValueCommitTimeout();
    }
    else
    {
        PLGErr("unknown timer type %d, timerid %u", iType, iTimerID);
    }
}
```

## 检查是否存在由Committer提交的请求等待处理

```
void Instance :: CheckNewValue()
{
    /*是否是一个新的Commit请求*/
    if (!m_oCommitCtx.IsNewCommit())
    {
        return;
    }

    if (!m_oLearner.IsIMLatest())
    {
        return;
    }

    if (m_poConfig->IsIMFollower())
    {
        PLGErr("I'm follower, skip this new value");
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Follower_Cannot_Commit);
        return;
    }

    /*是否在成员组中*/
    if (!m_poConfig->CheckConfig())
    {
        PLGErr("I'm not in membership, skip this new value");
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Im_Not_In_Membership);
        return;
    }

    /*Commit value太大*/
    if ((int)m_oCommitCtx.GetCommitValue().size() > MAX_VALUE_SIZE)
    {
        PLGErr("value size %zu to large, skip this new value",
            m_oCommitCtx.GetCommitValue().size());
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Value_Size_TooLarge);
        return;
    }

    /*设置Instance ID*/
    m_oCommitCtx.StartCommit(m_oProposer.GetInstanceID());

    /* 如果本次Commit请求设置了超时时间，则在IOLoop中添加一个
     * Timer_Instance_Commit_Timeout类型的定时器
     */
    if (m_oCommitCtx.GetTimeoutMs() != -1)
    {
        m_oIOLoop.AddTimer(m_oCommitCtx.GetTimeoutMs(), Timer_Instance_Commit_Timeout, m_iCommitTimerID);
    }
    
    m_oTimeStat.Point();

    if (m_poConfig->GetIsUseMembership()
            && (m_oProposer.GetInstanceID() == 0 || m_poConfig->GetGid() == 0))
    {
        //Init system variables.
        /*初始化系统变量*/
        PLGHead("Need to init system variables, Now.InstanceID %lu Now.Gid %lu", 
                m_oProposer.GetInstanceID(), m_poConfig->GetGid());

        uint64_t llGid = OtherUtils::GenGid(m_poConfig->GetMyNodeID());
        string sInitSVOpValue;
        int ret = m_poConfig->GetSystemVSM()->CreateGid_OPValue(llGid, sInitSVOpValue);
        assert(ret == 0);

        m_oSMFac.PackPaxosValue(sInitSVOpValue, m_poConfig->GetSystemVSM()->SMID());
        m_oProposer.NewValue(sInitSVOpValue);
    }
    else
    {
        if (m_oOptions.bOpenChangeValueBeforePropose) {
            m_oSMFac.BeforePropose(m_poConfig->GetMyGroupIdx(), m_oCommitCtx.GetCommitValue());
        }
        
        /*发起提案*/
        m_oProposer.NewValue(m_oCommitCtx.GetCommitValue());
    }
}
```

## 运用学习到的决议到状态机中

```
bool Instance :: SMExecute(
        const uint64_t llInstanceID, 
        const std::string & sValue, 
        const bool bIsMyCommit,
        SMCtx * poSMCtx)
{
    return m_oSMFac.Execute(m_poConfig->GetMyGroupIdx(), llInstanceID, sValue, poSMCtx);
}
```

## 启动新的实例（为接下来的提案做准备）

```
void Instance :: NewInstance()
{
    m_oAcceptor.NewInstance();
    m_oLearner.NewInstance();
    m_oProposer.NewInstance();
}
```







