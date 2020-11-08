# 提纲
[toc]

# Learner类基础
## 术语
Learner：学习者，在代码中它可能是主动学习的一方，也可能是被动学习的一方；

LearnerSender：被动学习者，被请求学习的Learner，这些称谓在后文中会混用；

LearnerReceiver：主动学习者，请求学习的Learner，这些称谓在后文中会混用；

## Learner类定义

```
class Learner : public Base
{
private:
    LearnerState m_oLearnerState;

    Acceptor * m_poAcceptor;
    PaxosLog m_oPaxosLog;

    uint32_t m_iAskforlearn_noopTimerID;
    /*直接来自于Instance类*/
    IOLoop * m_poIOLoop;

    uint64_t m_llHighestSeenInstanceID;
    nodeid_t m_iHighestSeenInstanceID_FromNodeID;

    bool m_bIsIMLearning;
    LearnerSender m_oLearnerSender;
    uint64_t m_llLastAckInstanceID;

    CheckpointMgr * m_poCheckpointMgr;
    SMFac * m_poSMFac;

    CheckpointSender * m_poCheckpointSender;
    CheckpointReceiver m_oCheckpointReceiver;
};

class LearnerState
{
private:
    std::string m_sLearnedValue;
    bool m_bIsLearned;
    uint32_t m_iNewChecksum;

    Config * m_poConfig;
    PaxosLog m_oPaxosLog;
};

/*LearnerSender在单独的线程*/
class LearnerSender : public Thread
{
private:
    Config * m_poConfig;
    Learner * m_poLearner;
    PaxosLog * m_poPaxosLog;
    SerialLock m_oLock;

    bool m_bIsIMSending;
    uint64_t m_llAbsLastSendTime;

    uint64_t m_llBeginInstanceID;
    nodeid_t m_iSendToNodeID;

    bool m_bIsComfirmed;

    uint64_t m_llAckInstanceID;
    uint64_t m_llAbsLastAckTime;
    int m_iAckLead;

    bool m_bIsEnd;
    bool m_bIsStart;
};

class CheckpointReceiver
{
private:
    Config * m_poConfig;
    LogStorage * m_poLogStorage;

private:
    nodeid_t m_iSenderNodeID; 
    uint64_t m_llUUID;
    uint64_t m_llSequence;

private:
    std::map<std::string, bool> m_mapHasInitDir;
};
```

## Learner类构造函数

```
Learner :: Learner(
        const Config * poConfig, 
        const MsgTransport * poMsgTransport,
        const Instance * poInstance,
        const Acceptor * poAcceptor,
        const LogStorage * poLogStorage,
        const IOLoop * poIOLoop,
        const CheckpointMgr * poCheckpointMgr,
        const SMFac * poSMFac)
    : Base(poConfig, poMsgTransport, poInstance), m_oLearnerState(poConfig, poLogStorage), 
    m_oPaxosLog(poLogStorage), m_oLearnerSender((Config *)poConfig, this, &m_oPaxosLog),
    m_oCheckpointReceiver((Config *)poConfig, (LogStorage *)poLogStorage)
{
    m_poAcceptor = (Acceptor *)poAcceptor;
    InitForNewPaxosInstance();

    m_iAskforlearn_noopTimerID = 0;
    m_poIOLoop = (IOLoop *)poIOLoop;

    m_poCheckpointMgr = (CheckpointMgr *)poCheckpointMgr;
    m_poSMFac = (SMFac *)poSMFac;
    m_poCheckpointSender = nullptr;

    m_llHighestSeenInstanceID = 0;
    m_iHighestSeenInstanceID_FromNodeID = nullnode;

    m_bIsIMLearning = false;

    m_llLastAckInstanceID = 0;
}

LearnerState :: LearnerState(const Config * poConfig, const LogStorage * poLogStorage)
    : m_oPaxosLog(poLogStorage)
{
    m_poConfig = (Config *)poConfig;

    Init();
}

void LearnerState :: Init()
{
    m_sLearnedValue = "";
    m_bIsLearned = false;
    m_iNewChecksum = 0;
}

LearnerSender :: LearnerSender(Config * poConfig, Learner * poLearner, PaxosLog * poPaxosLog)
    : m_poConfig(poConfig), m_poLearner(poLearner), m_poPaxosLog(poPaxosLog)
{
    m_iAckLead = LearnerSender_ACK_LEAD; 
    m_bIsEnd = false;
    m_bIsStart = false;
    /*借助SendDone来实现LearnerSender的初始化*/
    SendDone();
}

void LearnerSender :: SendDone()
{
    m_oLock.Lock();

    m_bIsIMSending = false;
    m_bIsComfirmed = false;
    m_llBeginInstanceID = (uint64_t)-1;
    m_iSendToNodeID = nullnode;
    m_llAbsLastSendTime = 0;
    
    m_llAckInstanceID = 0;
    m_llAbsLastAckTime = 0;

    m_oLock.UnLock();
}

CheckpointReceiver :: CheckpointReceiver(Config * poConfig, LogStorage * poLogStorage) :
    m_poConfig(poConfig), m_poLogStorage(poLogStorage)
{
    Reset();
}

void CheckpointReceiver :: Reset()
{
    m_mapHasInitDir.clear();
    
    m_iSenderNodeID = nullnode;
    m_llUUID = 0;
    m_llSequence = 0;
}

void Learner :: InitForNewPaxosInstance()
{
    m_oLearnerState.Init();
}

```

# LearnerState类方法
## 在内存中保存提案内容

```
void LearnerState :: LearnValueWithoutWrite(const uint64_t llInstanceID, 
        const std::string & sValue, const uint32_t iNewChecksum)
{
    m_sLearnedValue = sValue;
    m_bIsLearned = true;
    m_iNewChecksum = iNewChecksum;
}
```

## 同时在内存中和LogStorage中保存决议

```
int LearnerState :: LearnValue(const uint64_t llInstanceID, const BallotNumber & oLearnedBallot, 
        const std::string & sValue, const uint32_t iLastChecksum)
{
    if (llInstanceID > 0 && iLastChecksum == 0)
    {
        m_iNewChecksum = 0;
    }
    else if (sValue.size() > 0)
    {
        m_iNewChecksum = crc32(iLastChecksum, (const uint8_t *)sValue.data(), sValue.size(), CRC32SKIP);
    }
    
    /*设置AcceptorStateData*/
    AcceptorStateData oState;
    oState.set_instanceid(llInstanceID);
    oState.set_acceptedvalue(sValue);
    oState.set_promiseid(oLearnedBallot.m_llProposalID);
    oState.set_promisenodeid(oLearnedBallot.m_llNodeID);
    oState.set_acceptedid(oLearnedBallot.m_llProposalID);
    oState.set_acceptednodeid(oLearnedBallot.m_llNodeID);
    oState.set_checksum(m_iNewChecksum);

    WriteOptions oWriteOptions;
    oWriteOptions.bSync = false;

    /*写入LogStorage*/
    int ret = m_oPaxosLog.WriteState(oWriteOptions, m_poConfig->GetMyGroupIdx(), llInstanceID, oState);
    if (ret != 0)
    {
        PLGErr("LogStorage.WriteLog fail, InstanceID %lu ValueLen %zu ret %d",
                llInstanceID, sValue.size(), ret);
        return ret;
    }

    /*同时在内存中保存提案内容*/
    LearnValueWithoutWrite(llInstanceID, sValue, m_iNewChecksum);

    PLGDebug("OK, InstanceID %lu ValueLen %zu checksum %u",
            llInstanceID, sValue.size(), m_iNewChecksum);

    return 0;
}
```

# LearnerSender类方法
## 启动LearnerSender

```
void Learner :: StartLearnerSender()
{
    /* LeanerSender继承自Thread，所以直接调用Thread::start，执行函数是
     * LearnerSender::run()
     */
    m_oLearnerSender.start();
}
```

## LearnerSender运转
```
void LearnerSender :: run()
{
    m_bIsStart = true;

    while (true)
    {
        /*等待被唤醒，当Confirm了任何远端Learner的AskForLearn请求时，就会唤醒LearnerSender*/
        WaitToSend();

        if (m_bIsEnd)
        {
            PLGHead("Learner.Sender [END]");
            return;
        }

        /*从@m_llBeginInstanceID开始发送决议给远端的Learner @m_iSendToNodeID*/
        SendLearnedValue(m_llBeginInstanceID, m_iSendToNodeID);

        /*发送完毕，清理状态*/
        SendDone();
    }
}

void LearnerSender :: WaitToSend()
{
    m_oLock.Lock();
    /*只要是LearnerSender尚未Confirm任何Learner的AskForLearn请求，就一直等待*/
    while (!m_bIsComfirmed)
    {
        m_oLock.WaitTime(1000);
        if (m_bIsEnd)
        {
            break;
        }
    }
    m_oLock.UnLock();
}

void LearnerSender :: SendLearnedValue(const uint64_t llBeginInstanceID, const nodeid_t iSendToNodeID)
{
    PLGHead("BeginInstanceID %lu SendToNodeID %lu", llBeginInstanceID, iSendToNodeID);

    uint64_t llSendInstanceID = llBeginInstanceID;
    int ret = 0;
    
    uint32_t iLastChecksum = 0;

    //control send speed to avoid affecting the network too much.
    /*控制发送速度*/
    int iSendQps = LearnerSender_SEND_QPS;
    int iSleepMs = iSendQps > 1000 ? 1 : 1000 / iSendQps;
    int iSendInterval = iSendQps > 1000 ? iSendQps / 1000 + 1 : 1; 

    PLGDebug("SendQps %d SleepMs %d SendInterval %d AckLead %d",
            iSendQps, iSleepMs, iSendInterval, m_iAckLead);

    /*从@llBeginInstanceID开始，至m_poLearner->GetInstanceID()结束*/
    int iSendCount = 0;
    while (llSendInstanceID < m_poLearner->GetInstanceID())
    {    
        /*发送Instance @llSendInstanceID的决议给@iSendToNodeID所代表的Learner*/
        ret = SendOne(llSendInstanceID, iSendToNodeID, iLastChecksum);
        if (ret != 0)
        {
            PLGErr("SendOne fail, SendInstanceID %lu SendToNodeID %lu ret %d",
                    llSendInstanceID, iSendToNodeID, ret);
            return;
        }

        /* 如果CheckAck返回false，则表明远端的Learner已经学习到了比当前正在发送的
         * Instance更新的Instance的决议，或者较长时间未收到Ack了，那么就要停止发送
         */
        if (!CheckAck(llSendInstanceID))
        {
            return;
        }

        iSendCount++;
        llSendInstanceID++;
        ReleshSending();
        /*控制发送速度，如果一口气发送了@iSendInterval个Instance的决议，则暂停一会*/
        if (iSendCount >= iSendInterval)
        {
            iSendCount = 0;
            Time::MsSleep(iSleepMs);
        }
    }

    /*重置@m_iAckLead*/
    m_iAckLead = LearnerSender_ACK_LEAD;
    PLGImp("SendDone, SendEndInstanceID %lu", llSendInstanceID);
}

int LearnerSender :: SendOne(const uint64_t llSendInstanceID, const nodeid_t iSendToNodeID, uint32_t & iLastChecksum)
{
    BP->GetLearnerBP()->SenderSendOnePaxosLog();

    /*从LogStorage中读取出指定Instance @llSendInstanceID的决议*/
    AcceptorStateData oState;
    int ret = m_poPaxosLog->ReadState(m_poConfig->GetMyGroupIdx(), llSendInstanceID, oState);
    if (ret != 0)
    {
        return ret;
    }

    /*@llSendInstanceID这一Instance对应的提案*/
    BallotNumber oBallot(oState.acceptedid(), oState.acceptednodeid());

    /*发送决议（包括提案，决议，checksum）给远端的Learner，调用Learner :: SendLearnValue接口来实现*/
    ret = m_poLearner->SendLearnValue(iSendToNodeID, llSendInstanceID, oBallot, oState.acceptedvalue(), iLastChecksum);

    /*从AcceptorStateData中解析出checksum信息，返回给上层调用*/
    iLastChecksum = oState.checksum();

    return ret;
}

const bool LearnerSender :: CheckAck(const uint64_t llSendInstanceID)
{
    m_oLock.Lock();

    /* 已经Ack过的InstanceID超过了当前正在发送决议的InstanceID，说明[llSendInstanceID,
     * m_llAckInstanceID]区间内的Instance的决议都已经学习到了，返回false，通知调用者
     * 不再发送后续Instance的决议了
     */
    if (llSendInstanceID < m_llAckInstanceID)
    {
        m_iAckLead = LearnerSender_ACK_LEAD;
        PLGImp("Already catch up, ack instanceid %lu now send instanceid %lu", 
                m_llAckInstanceID, llSendInstanceID);
        m_oLock.UnLock();
        return false;
    }

    /*自从上次Ack以来，发送了决议的Instance超过了@m_iAckLead（表示较长时间没有收到Ack消息了）*/
    while (llSendInstanceID > m_llAckInstanceID + m_iAckLead)
    {
        uint64_t llNowTime = Time::GetSteadyClockMS();
        uint64_t llPassTime = llNowTime > m_llAbsLastAckTime ? llNowTime - m_llAbsLastAckTime : 0;

        /* 自从上次Ack以来，到现在为止的时间超过了阈值LearnerSender_ACK_TIMEOUT，
         * 就返回false，通知调用者不再发送后续Instance的决议了，等待Ack
         */
        if ((int)llPassTime >= LearnerSender_ACK_TIMEOUT)
        {
            BP->GetLearnerBP()->SenderAckTimeout();
            PLGErr("Ack timeout, last acktime %lu now send instanceid %lu", 
                    m_llAbsLastAckTime, llSendInstanceID);
            /*削减m_iAckLead，以便能尽快接收到Ack*/
            CutAckLead();
            m_oLock.UnLock();
            return false;
        }

        BP->GetLearnerBP()->SenderAckDelay();
        
        /*休息一会，降低发送决议的速度*/        
        m_oLock.WaitTime(20);
    }

    m_oLock.UnLock();

    return true;
}

void LearnerSender :: CutAckLead()
{
    int iReceiveAckLead = LearnerReceiver_ACK_LEAD;
    if (m_iAckLead - iReceiveAckLead > iReceiveAckLead)
    {
        m_iAckLead = m_iAckLead - iReceiveAckLead;
    }
}

void LearnerSender :: SendDone()
{
    m_oLock.Lock();

    /*重置学习状态*/
    
    /*退出当前的学习过程*/
    m_bIsIMSending = false;
    m_bIsComfirmed = false;
    m_llBeginInstanceID = (uint64_t)-1;
    m_iSendToNodeID = nullnode;
    m_llAbsLastSendTime = 0;
    
    m_llAckInstanceID = 0;
    m_llAbsLastAckTime = 0;

    m_oLock.UnLock();
}
```

## LearnerSender预处理（Prepare）AskForLearn请求（message type：MsgType_PaxosLearner_AskforLearn）

```
const bool LearnerSender :: Prepare(const uint64_t llBeginInstanceID, const nodeid_t iSendToNodeID)
{
    m_oLock.Lock();
    
    bool bPrepareRet = false;
    /*如果当前尚未进入sending状态，且没有Confirm任何Learner的AskForLearn请求*/
    if (!IsIMSending() && !m_bIsComfirmed)
    {
        bPrepareRet = true;

        /*切换为sending状态*/
        m_bIsIMSending = true;
        m_llAbsLastSendTime = m_llAbsLastAckTime = Time::GetSteadyClockMS();
        /*启动即将发送给远端Learner的起始InstanceID*/
        m_llBeginInstanceID = m_llAckInstanceID = llBeginInstanceID;
        /*设置远端的Learner所在的节点*/
        m_iSendToNodeID = iSendToNodeID;
    }
    
    m_oLock.UnLock();

    return bPrepareRet;
}
```

## LearnerSender确认（Confirm）AskForLearn请求（message type：MsgType_PaxosLearner_ComfirmAskforLearn）

```
const bool LearnerSender :: Comfirm(const uint64_t llBeginInstanceID, const nodeid_t iSendToNodeID)
{
    m_oLock.Lock();

    bool bComfirmRet = false;

    /*如果当前正处于sending状态，且尚未Confirm任何Learner的AskForLearn请求*/
    if (IsIMSending() && (!m_bIsComfirmed))
    {
        /*该Confirm消息和Prepare阶段设置的@m_llBeginInstanceID、@m_iSendToNodeID等信息匹配*/
        if (m_llBeginInstanceID == llBeginInstanceID && m_iSendToNodeID == iSendToNodeID)
        {
            bComfirmRet = true;

            /*我已经Confirm了@m_iSendToNodeID所标识的远端Learner的AskForLearn请求*/
            m_bIsComfirmed = true;
            /*唤醒LearnerSender，开始发送决议给远端的Learner*/
            m_oLock.Interupt();
        }
    }

    m_oLock.UnLock();

    return bComfirmRet;
}
```



# Learner类方法
## 重置AskForLearn定时器

```
void Learner :: Reset_AskforLearn_Noop(const int iTimeout)
{
    if (m_iAskforlearn_noopTimerID > 0)
    {
        m_poIOLoop->RemoveTimer(m_iAskforlearn_noopTimerID);
    }

    m_poIOLoop->AddTimer(iTimeout, Timer_Learner_Askforlearn_noop, m_iAskforlearn_noopTimerID);
}
```

## 设置本Learner所了解到的其它Learner中最大的InstanceID

```
void Learner :: SetSeenInstanceID(const uint64_t llInstanceID, const nodeid_t llFromNodeID)
{
    if (llInstanceID > m_llHighestSeenInstanceID)
    {
        m_llHighestSeenInstanceID = llInstanceID;
        m_iHighestSeenInstanceID_FromNodeID = llFromNodeID;
    }
}
```


## Proposer通知Learner本次提案成功

```
void Learner :: ProposerSendSuccess(
        const uint64_t llLearnInstanceID,
        const uint64_t llProposalID)
{
    BP->GetLearnerBP()->ProposerSendSuccess();

    PaxosMsg oPaxosMsg;
    
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_ProposerSendSuccess);
    oPaxosMsg.set_instanceid(llLearnInstanceID);
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalid(llProposalID);
    oPaxosMsg.set_lastchecksum(GetLastChecksum());

    /*广播消息，首先在本节点上运行*/
    BroadcastMessage(oPaxosMsg, BroadcastMessage_Type_RunSelf_First);
}
```

## 处理来自于Proposer的“本次提案成功”的消息（message type：MsgType_PaxosLearner_ProposerSendSuccess）

```
void Learner :: OnProposerSendSuccess(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnProposerSendSuccess();

    /*不在同一个Instance*/
    if (oPaxosMsg.instanceid() != GetInstanceID())
    {
        //Instance id not same, that means not in the same instance, ignord.
        PLGDebug("InstanceID not same, skip msg");
        return;
    }

    /*Acceptor尚未接受任何提案*/
    if (m_poAcceptor->GetAcceptorState()->GetAcceptedBallot().isnull())
    {
        //Not accept any yet.
        BP->GetLearnerBP()->OnProposerSendSuccessNotAcceptYet();
        PLGDebug("I haven't accpeted any proposal");
        return;
    }

    /*消息中记录的成功的提案*/
    BallotNumber oBallot(oPaxosMsg.proposalid(), oPaxosMsg.nodeid());

    /* 消息（可能来自于本地节点，也可能来自于其它节点）中记录的提案和本地在Acceptor中
     * 记录的提案都有可能不一样，因为任何一个节点都可能会拒绝该提案，但是存在多数派
     * 节点成功接受了该提案，该提案依然会通过
     */
    if (m_poAcceptor->GetAcceptorState()->GetAcceptedBallot()
            != oBallot)
    {
        //Proposalid not same, this accept value maybe not chosen value.
        PLGDebug("ProposalBallot not same to AcceptedBallot");
        BP->GetLearnerBP()->OnProposerSendSuccessBallotNotSame();
        return;
    }

    //learn value.
    /*将提案对应的value和checksum保存在LearnerState中*/
    m_oLearnerState.LearnValueWithoutWrite(
            oPaxosMsg.instanceid(),
            m_poAcceptor->GetAcceptorState()->GetAcceptedValue(),
            m_poAcceptor->GetAcceptorState()->GetChecksum());
    
    BP->GetLearnerBP()->OnProposerSendSuccessSuccessLearn();

    PLGHead("END Learn value OK, value %zu", m_poAcceptor->GetAcceptorState()->GetAcceptedValue().size());

    /*将提案转发给所有follow我的follower*/
    TransmitToFollower();
}
```

## 转发Learner学习到的提案给所有follow我的Follower（message type: MsgType_PaxosLearner_SendLearnValue）

```
void Learner :: TransmitToFollower()
{
    if (m_poConfig->GetMyFollowerCount() == 0)
    {
        return;
    }
    
    PaxosMsg oPaxosMsg;
    
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_SendLearnValue);
    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalnodeid(m_poAcceptor->GetAcceptorState()->GetAcceptedBallot().m_llNodeID);
    oPaxosMsg.set_proposalid(m_poAcceptor->GetAcceptorState()->GetAcceptedBallot().m_llProposalID);
    oPaxosMsg.set_value(m_poAcceptor->GetAcceptorState()->GetAcceptedValue());
    oPaxosMsg.set_lastchecksum(GetLastChecksum());

    /*调用Base :: BroadcastMessageToFollower*/
    BroadcastMessageToFollower(oPaxosMsg, Message_SendType_TCP);

    PLGHead("ok");
}
```

## 处理来自于Follower的关于MsgType_PaxosLearner_SendLearnValue的Ack（message type: MsgType_PaxosLearner_SendLearnValue_Ack）

```
void Learner :: OnSendLearnValue_Ack(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnSendLearnValue_Ack();

    PLGHead("Msg.Ack.Instanceid %lu Msg.from_nodeid %lu", oPaxosMsg.instanceid(), oPaxosMsg.nodeid());

    /*更新LearnerSender的状态*/
    m_oLearnerSender.Ack(oPaxosMsg.instanceid(), oPaxosMsg.nodeid());
}
```

## 为新一轮提案做准备

```
Learner::NewInstance
    - Base::NewInstance
    
void Base :: NewInstance()
{
    /*增长InstanceID*/
    m_llInstanceID++;
    InitForNewPaxosInstance();
}

void Learner :: InitForNewPaxosInstance()
{
    /*初始化LearnerState*/
    m_oLearnerState.Init();
}

void LearnerState :: Init()
{
    m_sLearnedValue = "";
    m_bIsLearned = false;
    m_iNewChecksum = 0;
}
```


## Learner执行实例对齐过程
### Instance初始化过程中初始化Learner定时器

```
int Instance :: Init()
{
    ......

    m_oLearner.SetInstanceID(llNowInstanceID);
    m_oProposer.SetInstanceID(llNowInstanceID);
    m_oProposer.SetStartProposalID(m_oAcceptor.GetAcceptorState()->GetPromiseBallot().m_llProposalID + 1);

    ......

    /*重置Learner的AskForLearn定时器*/
    m_oLearner.Reset_AskforLearn_Noop();

    ......
}

void Learner :: Reset_AskforLearn_Noop(const int iTimeout)
{
    /*如果已经为该Learner设置了AskForLearn定时器，则先删除之*/
    if (m_iAskforlearn_noopTimerID > 0)
    {
        m_poIOLoop->RemoveTimer(m_iAskforlearn_noopTimerID);
    }

    /*重新为Learner添加AskForLearn定时器，当定时器时间到的时候，在Instance::OnTimeout中处理*/
    m_poIOLoop->AddTimer(iTimeout, Timer_Learner_Askforlearn_noop, m_iAskforlearn_noopTimerID);
}
```

### 处理Learner的AskForLearn定时器事件
```
void Instance :: OnTimeout(const uint32_t iTimerID, const int iType)
{
    if (iType == Timer_Proposer_Prepare_Timeout)
    {
        m_oProposer.OnPrepareTimeout();
    }
    else if (iType == Timer_Proposer_Accept_Timeout)
    {
        m_oProposer.OnAcceptTimeout();
    }
    else if (iType == Timer_Learner_Askforlearn_noop)
    {
        /*Learner的AskForLearn定时器被激活*/
        m_oLearner.AskforLearn_Noop();
    }
    else if (iType == Timer_Instance_Commit_Timeout)
    {
        OnNewValueCommitTimeout();
    }
    else
    {
        PLGErr("unknown timer type %d, timerid %u", iType, iTimerID);
    }
}

void Learner :: AskforLearn_Noop(const bool bIsStart)
{
    /*重新为该Learner添加AskForLearn定时器*/
    Reset_AskforLearn_Noop();

    /*只有当Confirm了AskForLearn之后，才会设置@m_bIsIMLearning为true*/
    m_bIsIMLearning = false;

    /* 在checkpoint过程中，不会处理Paxos协议相关的消息，所以这里要设置
     * 不在checkpoint过程中
     */
    m_poCheckpointMgr->ExitCheckpointMode();

    /*向所有节点发送学习请求*/
    AskforLearn();
    
    /* 这一步是干嘛的？目前PhxPaxos中并未看到在调用AskforLearn_Noop的时
     * 候传递的参数@bIsStart为true的情况
     */
    if (bIsStart)
    {
        AskforLearn();
    }
}
```

### Learner处理所有接收到的请求

```
int Instance :: ReceiveMsgForLearner(const PaxosMsg & oPaxosMsg)
{
    ========================================================================================
    友情提示： 
        按照编号的顺序来阅读代码，理清LearnerReceiver和LearnerSender之间的交互过程
    ========================================================================================
    
    if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforLearn)
    {
        /*（1）接收到LearnerReceiver的AskForLearn消息，首先LearnerSender进行Prepare，
         * 即检查LearnerSender是否正在为其它LearnerReceiver服务。如果正在为其它
         * LearnerReceiver服务，但是LearnerReceiver请求的InstanceID只比LearnerSender
         * 当前的InstanceID小1，那么直接将LearnerReceiver请求学习的Instance的决议发送
         * 给它，否则不能满足本次的AskForLearn请求；如果没有为其它LearnerReceiver服务，
         * 则将LearnerSender（CheckpointMgr中最小chosen InstanceID，当前的InstanceID）
         * 发送给LearnerReceiver
         */
        m_oLearner.OnAskforLearn(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue)
    {
        /*（4）LearnerReceiver接收到LearnerSender发送过来的决议，LearnerReceiver
         * 学习决议（同时将决议保存在内存中并持久化到LogStorage中），如果需要Ack，
         * 则发送MsgType_PaxosLearner_SendLearnValue_Ack类型的消息给LearnerSender，
         * 以告知对方LearnerReceiver上当前已经学习到的最大InstanceID
         */
        m_oLearner.OnSendLearnValue(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_ProposerSendSuccess)
    {
        /* Learner接收到来自于某Proposer发送过来的关于它本次提案成功的消息，该Learner
         * 将学习该决议（将决议保存在Learner的内存中），同时将该决议推向所有follow它
         * 的Follower
         */
        m_oLearner.OnProposerSendSuccess(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendNowInstanceID)
    {
        /*（2）LearnerReceiver接收到LearnerSender发送过来的（CheckpointMgr中最小
         * chosen InstanceID，当前的InstanceID），如果LearnerSender上CheckpointMgr
         * 中最小chosen InstanceID比LearnerReceiver当前InstanceID要大，则发送
         * AskForCheckpoint消息给LearnerSender，否则如果LearnerSender的当前InstanceID
         * 比LearnerReceiver的要大，则发送ConfirmAskForLearn消息
         */
        m_oLearner.OnSendNowInstanceID(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_ComfirmAskforLearn)
    {
        /*（3b）LearnerSender接收到LearnerReceiver发送过来的ConfirmAskForLearn消息，
         * LearnerSender进行Confirm，如果Confirm成功，则唤醒处于等待状态的LearnerSender，
         * 紧接着LearnerSender源源不断的发送各Instance的决议值给LearnerReceiver，消息
         * 类型为MsgType_PaxosLearner_SendLearnValue
         */
        m_oLearner.OnComfirmAskForLearn(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_SendLearnValue_Ack)
    {
        /*（5）LearnerSender接收到来自于LearnerReceiver的Ack消息，设置本地关于该
         * LearnerReceiver的已经学习到的InstanceID，并且唤醒可能因为长时间没有Ack
         * 而进入等待状态的LearnerSender
         */
        m_oLearner.OnSendLearnValue_Ack(oPaxosMsg);
    }
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_AskforCheckpoint)
    {
        /*（3a）LearnerSender接收到LearnerReceiver发送过来的AskForCheckpoint消息*/
        m_oLearner.OnAskforCheckpoint(oPaxosMsg);
    }

    /*如果LearnerReceiver成功学习了LearnerSender发送过来的决议*/
    if (m_oLearner.IsLearned())
    {
        BP->GetInstanceBP()->OnInstanceLearned();

        SMCtx * poSMCtx = nullptr;
        /*判断本次学习的决议是否和本实例自己的Committer提交的提案一致，见CommitCtx :: IsMyCommit*/
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

        /*将学习到的决议运用到状态机中*/
        if (!SMExecute(m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue(), bIsMyCommit, poSMCtx))
        {
            BP->GetInstanceBP()->OnInstanceLearnedSMExecuteFail();

            PLGErr("SMExecute fail, instanceid %lu, not increase instanceid", m_oLearner.GetInstanceID());
            
            /*设置本次Commiter提交的失败*/
            m_oCommitCtx.SetResult(PaxosTryCommitRet_ExecuteFail, 
                    m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue());

            /*不能再略过Prepare阶段了*/
            m_oProposer.CancelSkipPrepare();

            return -1;
        }
        
        {
            /*设置本次Committer的提交成功*/
            //this paxos instance end, tell proposal done
            m_oCommitCtx.SetResult(PaxosTryCommitRet_OK
                    , m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue());

            /*移除Committer相关定时器*/
            if (m_iCommitTimerID > 0)
            {
                m_oIOLoop.RemoveTimer(m_iCommitTimerID);
            }
        }
        
        m_iLastChecksum = m_oLearner.GetNewChecksum();
        /*为新的Instance做准备*/
        NewInstance();

        PLGHead("[Learned] New paxos instance has started, Now.Proposer.InstanceID %lu "
                "Now.Acceptor.InstanceID %lu Now.Learner.InstanceID %lu",
                m_oProposer.GetInstanceID(), m_oAcceptor.GetInstanceID(), m_oLearner.GetInstanceID());

        /*更新CheckpointMgr中max chosen InstanceID*/
        m_oCheckpointMgr.SetMaxChosenInstanceID(m_oAcceptor.GetInstanceID());

        BP->GetInstanceBP()->NewInstance();
    }

    return 0;
}
```

### 向所有节点发送学习请求（message type：MsgType_PaxosLearner_AskforLearn）

```
void Learner :: AskforLearn()
{
    BP->GetLearnerBP()->AskforLearn();

    PLGHead("START");

    PaxosMsg oPaxosMsg;
    /*设置本Learner当前看到的最大的InstanceID*/
    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_AskforLearn);

    if (m_poConfig->IsIMFollower())
    {
        /*设置本Learner所Follow的节点ID*/
        //this is not proposal nodeid, just use this val to bring followto nodeid info.
        oPaxosMsg.set_proposalnodeid(m_poConfig->GetFollowToNodeID());
    }

    PLGHead("END InstanceID %lu MyNodeID %lu", oPaxosMsg.instanceid(), oPaxosMsg.nodeid());

    /*广播学习请求给所有节点（除本节点除外，因为无需从自己这里学习）*/
    BroadcastMessage(oPaxosMsg, BroadcastMessage_Type_RunSelf_None, Message_SendType_TCP);
    /*广播学习请求给所有TempNode*/
    BroadcastMessageToTempNode(oPaxosMsg, Message_SendType_UDP);
}
```

### 处理来自于其它Learner的学习请求

```
void Learner :: OnAskforLearn(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnAskforLearn();
    
    PLGHead("START Msg.InstanceID %lu Now.InstanceID %lu Msg.from_nodeid %lu MinChosenInstanceID %lu", 
            oPaxosMsg.instanceid(), GetInstanceID(), oPaxosMsg.nodeid(),
            m_poCheckpointMgr->GetMinChosenInstanceID());
    
    /*更新本Learner所了解到的其它Learner上最大的InstanceID和对应的节点ID*/
    SetSeenInstanceID(oPaxosMsg.instanceid(), oPaxosMsg.nodeid());

    /* 根据AskforLearn中的注释，@oPaxosMsg.proposalnodeid()中存放的是想要
     * 学习的Learner所Follow的节点ID
     */
    if (oPaxosMsg.proposalnodeid() == m_poConfig->GetMyNodeID())
    {
        //Found a node follow me.
        /*记录下来，想要学习的Learner所在的节点Follow我*/
        PLImp("Found a node %lu follow me.", oPaxosMsg.nodeid());
        m_poConfig->AddFollowerNode(oPaxosMsg.nodeid());
    }
    
    /*远端Learner比我更新*/
    if (oPaxosMsg.instanceid() >= GetInstanceID())
    {
        return;
    }

    if (oPaxosMsg.instanceid() >= m_poCheckpointMgr->GetMinChosenInstanceID())
    {
        if (!m_oLearnerSender.Prepare(oPaxosMsg.instanceid(), oPaxosMsg.nodeid()))
        {
            /*LearnerSender准备阶段失败（正在为其它Learner服务）*/
            
            BP->GetLearnerBP()->OnAskforLearnGetLockFail();

            PLGErr("LearnerSender working for others.");

            if (oPaxosMsg.instanceid() == (GetInstanceID() - 1))
            {
                /*请求学习的远端Learner的InstanceID只比本地Learner小1，直接发送给它*/
                
                PLGImp("InstanceID only difference one, just send this value to other.");
                /*从LogStorage中读取出来远端Learner想要学习的Instance的决议*/
                AcceptorStateData oState;
                int ret = m_oPaxosLog.ReadState(m_poConfig->GetMyGroupIdx(), oPaxosMsg.instanceid(), oState);
                if (ret == 0)
                {
                    /*直接发送给它*/
                    BallotNumber oBallot(oState.acceptedid(), oState.acceptednodeid());
                    SendLearnValue(oPaxosMsg.nodeid(), oPaxosMsg.instanceid(), oBallot, oState.acceptedvalue(), 0, false);
                }
            }
            
            return;
        }
    }
    
    /*将本地的InstanceID信息发送给请求学习的远端Learner*/
    SendNowInstanceID(oPaxosMsg.instanceid(), oPaxosMsg.nodeid());
}
```

### 发送本地最新的InstanceID信息给请求学习的远端Learner（message type: MsgType_PaxosLearner_SendNowInstanceID）

```
void Learner :: SendNowInstanceID(const uint64_t llInstanceID, const nodeid_t iSendNodeID)
{
    BP->GetLearnerBP()->SendNowInstanceID();

    PaxosMsg oPaxosMsg;
    /*远端请求学习的Learner自身的InstanceID*/
    oPaxosMsg.set_instanceid(llInstanceID);
    /*我的节点ID*/
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_SendNowInstanceID);
    /*我当前的最大InstanceID*/
    oPaxosMsg.set_nowinstanceid(GetInstanceID());
    oPaxosMsg.set_minchoseninstanceid(m_poCheckpointMgr->GetMinChosenInstanceID());

    if ((GetInstanceID() - llInstanceID) > 50)
    {
        /* 如果本地最大InstanceID和请求学习的Learner的InstanceID之间差距过大，
         * 则同时需要发送System variable和Master的checkpoint相关的信息
         */
        //instanceid too close not need to send vsm/master checkpoint. 
        string sSystemVariablesCPBuffer;
        int ret = m_poConfig->GetSystemVSM()->GetCheckpointBuffer(sSystemVariablesCPBuffer);
        if (ret == 0)
        {
            oPaxosMsg.set_systemvariables(sSystemVariablesCPBuffer);
        }

        string sMasterVariablesCPBuffer;
        if (m_poConfig->GetMasterSM() != nullptr)
        {
            int ret = m_poConfig->GetMasterSM()->GetCheckpointBuffer(sMasterVariablesCPBuffer);
            if (ret == 0)
            {
                oPaxosMsg.set_mastervariables(sMasterVariablesCPBuffer);
            }
        }
    }

    /*发送响应给请求学习的Learner*/
    SendMessage(iSendNodeID, oPaxosMsg);
}
```

### 接收到被学习的Learner返回的最大InstanceID

```
void Learner :: OnSendNowInstanceID(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnSendNowInstanceID();

    PLGHead("START Msg.InstanceID %lu Now.InstanceID %lu Msg.from_nodeid %lu Msg.MaxInstanceID %lu systemvariables_size %zu mastervariables_size %zu",
            oPaxosMsg.instanceid(), GetInstanceID(), oPaxosMsg.nodeid(), oPaxosMsg.nowinstanceid(), 
            oPaxosMsg.systemvariables().size(), oPaxosMsg.mastervariables().size());
    
    /*更新本Learner所了解到的其它Learner上最大的InstanceID和对应的节点ID*/
    SetSeenInstanceID(oPaxosMsg.nowinstanceid(), oPaxosMsg.nodeid());

    /*更新SystemVariables，如果SystemVariables发生了改变，则直接退出*/
    bool bSystemVariablesChange = false;
    int ret = m_poConfig->GetSystemVSM()->UpdateByCheckpoint(oPaxosMsg.systemvariables(), bSystemVariablesChange);
    if (ret == 0 && bSystemVariablesChange)
    {
        PLGHead("SystemVariables changed!, all thing need to reflesh, so skip this msg");
        return;
    }

    bool bMasterVariablesChange = false;
    if (m_poConfig->GetMasterSM() != nullptr)
    {
        /*更新MasterVariables*/
        ret = m_poConfig->GetMasterSM()->UpdateByCheckpoint(oPaxosMsg.mastervariables(), bMasterVariablesChange);
        if (ret == 0 && bMasterVariablesChange)
        {
            PLGHead("MasterVariables changed!");
        }
    }

    /*已经学习到了请求学习的InstanceID*/
    if (oPaxosMsg.instanceid() != GetInstanceID())
    {
        PLGErr("Lag msg, skip");
        return;
    }

    /*被学习的Learner的InstanceID比我的还小*/
    if (oPaxosMsg.nowinstanceid() <= GetInstanceID())
    {
        PLGErr("Lag msg, skip");
        return;
    }

    /* 被请求学习的Learner上最小chosen InstanceID都比我当前的最大InstanceID要大，
     * 表明，在被请求学习的Learner上我当前最大InstanceID对应的决议已经从LogStorage
     * 中删除了，只能从它的镜像状态机中学习了（拷贝数据）
     */
    if (oPaxosMsg.minchoseninstanceid() > GetInstanceID())
    {
        BP->GetCheckpointBP()->NeedAskforCheckpoint();

        PLGHead("my instanceid %lu small than other's minchoseninstanceid %lu, other nodeid %lu",
                GetInstanceID(), oPaxosMsg.minchoseninstanceid(), oPaxosMsg.nodeid());

        /*从被请求学习的Learner的镜像状态机中拷贝数据过来*/
        AskforCheckpoint(oPaxosMsg.nodeid());
    }
    else if (!m_bIsIMLearning)
    {
        /*如果尚未Confirm AskForLearn，则Confirm之*/
        ComfirmAskForLearn(oPaxosMsg.nodeid());
    }
}
```

### 确认从特定节点的镜像状态机学习（AskForCheckpoint）（message type：MsgType_PaxosLearner_AskforCheckpoint）
在从其它节点学习过程中，如果发现请求学习的Learner本地最大的InstanceID在被请求的Learner上已经删除了，那么就从被请求的Learner的镜像状态机中拷贝数据，以此来完成学习。
```
void Learner :: AskforCheckpoint(const nodeid_t iSendNodeID)
{
    PLGHead("START");

    /* 检查是否进入AskForCheckpoint模式（当有多数派节点告知我需要AskForCheckpoint
     * ，或者从第一个告知我需要AskForCheckpoint的节点开始的时间超过了60ms，才会
     * 进入AskForCheckpoint模式，发送AskForCheckpoint消息）
     */
    int ret = m_poCheckpointMgr->PrepareForAskforCheckpoint(iSendNodeID);
    if (ret != 0)
    {
        return;
    }

    PaxosMsg oPaxosMsg;

    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_AskforCheckpoint);

    PLGHead("END InstanceID %lu MyNodeID %lu", GetInstanceID(), oPaxosMsg.nodeid());
    
    SendMessage(iSendNodeID, oPaxosMsg);
}
```

### 接收到请求学习的Learner的“从镜像状态机学习”的请求

```
void Learner :: OnAskforCheckpoint(const PaxosMsg & oPaxosMsg)
{
    /* 为当前的请求学习的Learner准备一个单独的CheckpointSender线程，如果
     * 不能为该请求学习的Learner创建新的CheckpointSender线程，则表明当前
     * 的CheckpointSender线程正忙
     */
    CheckpointSender * poCheckpointSender = GetNewCheckpointSender(oPaxosMsg.nodeid());
    if (poCheckpointSender != nullptr)
    {
        /*为该请求学习的Learner启动新的CheckpointSender线程，进入CheckpointSender::start()*/
        poCheckpointSender->start();
        PLGHead("new checkpoint sender started, send to nodeid %lu", oPaxosMsg.nodeid());
    }
    else
    {
        PLGErr("Checkpoint Sender is running");
    }
}

CheckpointSender * Learner :: GetNewCheckpointSender(const nodeid_t iSendNodeID)
{
    /* 如果已经存在一个CheckpointSender线程，那么检查它是否结束了，
     * 如果结束了，就停止该线程，重新为新的请求学习的Learner启动一个
     * CheckpointSender线程；
     * 
     * 如果当前已经存在的CheckpointSender线程尚未结束，就不能为新的
     * 请求学习的Learner启动新的CheckpointSender线程；
     * 
     * 如果当前不存在任何CheckpointSender线程，那么直接为新的请求学习
     * 的Learner启动一个CheckpointSender线程
     */
    if (m_poCheckpointSender != nullptr)
    {
        if (m_poCheckpointSender->IsEnd())
        {
            m_poCheckpointSender->join();
            delete m_poCheckpointSender;
            m_poCheckpointSender = nullptr;
        }
    }

    if (m_poCheckpointSender == nullptr)
    {
        m_poCheckpointSender = new CheckpointSender(iSendNodeID, m_poConfig, this, m_poSMFac, m_poCheckpointMgr);
        return m_poCheckpointSender;
    }

    return nullptr;
}
```


### 确认从某个特定节点学习（ConfirmAskForLearn）（message type：MsgType_PaxosLearner_ComfirmAskforLearn）
再从其它节点学习过程中，如果请求学习的Learner本地最大的InstanceID在被请求的Learner上尚未被删除，则可以直接（从Paxos log中）学习（区别于从被请求的Learner的镜像状态机中学习）。
```
void Learner :: ComfirmAskForLearn(const nodeid_t iSendNodeID)
{
    BP->GetLearnerBP()->ComfirmAskForLearn();

    PLGHead("START");

    PaxosMsg oPaxosMsg;

    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_ComfirmAskforLearn);

    PLGHead("END InstanceID %lu MyNodeID %lu", GetInstanceID(), oPaxosMsg.nodeid());

    /*发送Confirm AskForLearn消息给@iSendNodeID代表的Learner（我将从该Learner学习）*/
    SendMessage(iSendNodeID, oPaxosMsg);

    /*正式Confirm了 AskForLearn消息，所以我正处于AskForLearn过程中*/
    m_bIsIMLearning = true;
}
```

### 接收到远端Learner的关于AskForLearn的确认消息（message type：MsgType_PaxosLearner_ComfirmAskforLearn）

```
void Learner :: OnComfirmAskForLearn(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnComfirmAskForLearn();

    PLGHead("START Msg.InstanceID %lu Msg.from_nodeid %lu", oPaxosMsg.instanceid(), oPaxosMsg.nodeid());

    /* 由LearnerSender来Confirm，一旦LearnerSender Confirm成功，LearnerSender就会进入
     * SendLearnedValue逻辑，依次发送各Instance的决议给远端的Learner，在SendLearnedValue
     * 中，每一个Instance的决议的发送都是通过LearnerSender::SendOne -> Learner::SendLearnValue
     * 实现的
     */
    if (!m_oLearnerSender.Comfirm(oPaxosMsg.instanceid(), oPaxosMsg.nodeid()))
    {
        BP->GetLearnerBP()->OnComfirmAskForLearnGetLockFail();

        PLGErr("LearnerSender comfirm fail, maybe is lag msg");
        return;
    }

    PLGImp("OK, success comfirm");
}
```

### 本地Learner通过LearnerSender向远端请求学习的Learner发送决议

```
void LearnerSender :: SendLearnedValue(const uint64_t llBeginInstanceID, const nodeid_t iSendToNodeID)
{
    PLGHead("BeginInstanceID %lu SendToNodeID %lu", llBeginInstanceID, iSendToNodeID);

    uint64_t llSendInstanceID = llBeginInstanceID;
    int ret = 0;
    
    uint32_t iLastChecksum = 0;

    //control send speed to avoid affecting the network too much.
    /*控制发送速度*/
    int iSendQps = LearnerSender_SEND_QPS;
    int iSleepMs = iSendQps > 1000 ? 1 : 1000 / iSendQps;
    int iSendInterval = iSendQps > 1000 ? iSendQps / 1000 + 1 : 1; 

    PLGDebug("SendQps %d SleepMs %d SendInterval %d AckLead %d",
            iSendQps, iSleepMs, iSendInterval, m_iAckLead);

    /*从@llBeginInstanceID开始，至m_poLearner->GetInstanceID()结束*/
    int iSendCount = 0;
    while (llSendInstanceID < m_poLearner->GetInstanceID())
    {    
        /*发送Instance @llSendInstanceID的决议给@iSendToNodeID所代表的Learner*/
        ret = SendOne(llSendInstanceID, iSendToNodeID, iLastChecksum);
        if (ret != 0)
        {
            PLGErr("SendOne fail, SendInstanceID %lu SendToNodeID %lu ret %d",
                    llSendInstanceID, iSendToNodeID, ret);
            return;
        }

        /* 如果CheckAck返回false，则表明远端的Learner已经学习到了比当前正在发送的
         * Instance更新的Instance的决议，或者较长时间未收到Ack了，那么就要停止发送
         */
        if (!CheckAck(llSendInstanceID))
        {
            return;
        }

        iSendCount++;
        llSendInstanceID++;
        ReleshSending();
        /*控制发送速度，如果一口气发送了@iSendInterval个Instance的决议，则暂停一会*/
        if (iSendCount >= iSendInterval)
        {
            iSendCount = 0;
            Time::MsSleep(iSleepMs);
        }
    }

    /*重置@m_iAckLead*/
    m_iAckLead = LearnerSender_ACK_LEAD;
    PLGImp("SendDone, SendEndInstanceID %lu", llSendInstanceID);
}

int LearnerSender :: SendOne(const uint64_t llSendInstanceID, const nodeid_t iSendToNodeID, uint32_t & iLastChecksum)
{
    BP->GetLearnerBP()->SenderSendOnePaxosLog();

    /*从LogStorage中读取出指定Instance @llSendInstanceID的决议*/
    AcceptorStateData oState;
    int ret = m_poPaxosLog->ReadState(m_poConfig->GetMyGroupIdx(), llSendInstanceID, oState);
    if (ret != 0)
    {
        return ret;
    }

    /*@llSendInstanceID这一Instance对应的提案*/
    BallotNumber oBallot(oState.acceptedid(), oState.acceptednodeid());

    /*发送决议（包括提案，决议，checksum）给远端的Learner，调用Learner :: SendLearnValue接口来实现*/
    ret = m_poLearner->SendLearnValue(iSendToNodeID, llSendInstanceID, oBallot, oState.acceptedvalue(), iLastChecksum);

    /*从AcceptorStateData中解析出checksum信息，返回给上层调用*/
    iLastChecksum = oState.checksum();

    return ret;
}

int Learner :: SendLearnValue(
        const nodeid_t iSendNodeID,
        const uint64_t llLearnInstanceID,
        const BallotNumber & oLearnedBallot,
        const std::string & sLearnedValue,
        const uint32_t iChecksum,
        const bool bNeedAck = true)
{
    BP->GetLearnerBP()->SendLearnValue();

    PaxosMsg oPaxosMsg;
    
    /*设置Instance @llLearnInstanceID相关的决议信息*/
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_SendLearnValue);
    oPaxosMsg.set_instanceid(llLearnInstanceID);
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalnodeid(oLearnedBallot.m_llNodeID);
    oPaxosMsg.set_proposalid(oLearnedBallot.m_llProposalID);
    oPaxosMsg.set_value(sLearnedValue);
    oPaxosMsg.set_lastchecksum(iChecksum);
    
    /*默认是需要发送Ack信息的*/
    if (bNeedAck)
    {
        oPaxosMsg.set_flag(PaxosMsgFlagType_SendLearnValue_NeedAck);
    }

    /*发送消息给远端的Learner @iSendNodeID*/
    return SendMessage(iSendNodeID, oPaxosMsg, Message_SendType_TCP);
}

const bool LearnerSender :: CheckAck(const uint64_t llSendInstanceID)
{
    m_oLock.Lock();

    /* 已经Ack过的InstanceID超过了当前正在发送决议的InstanceID，说明[llSendInstanceID,
     * m_llAckInstanceID]区间内的Instance的决议都已经学习到了，返回false，通知调用者
     * 不再发送后续Instance的决议了
     */
    if (llSendInstanceID < m_llAckInstanceID)
    {
        m_iAckLead = LearnerSender_ACK_LEAD;
        PLGImp("Already catch up, ack instanceid %lu now send instanceid %lu", 
                m_llAckInstanceID, llSendInstanceID);
        m_oLock.UnLock();
        return false;
    }

    /*自从上次Ack以来，发送了决议的Instance超过了@m_iAckLead（表示较长时间没有收到Ack消息了）*/
    while (llSendInstanceID > m_llAckInstanceID + m_iAckLead)
    {
        uint64_t llNowTime = Time::GetSteadyClockMS();
        uint64_t llPassTime = llNowTime > m_llAbsLastAckTime ? llNowTime - m_llAbsLastAckTime : 0;

        /* 自从上次Ack以来，到现在为止的时间超过了阈值LearnerSender_ACK_TIMEOUT，
         * 就返回false，通知调用者不再发送后续Instance的决议了，等待Ack
         */
        if ((int)llPassTime >= LearnerSender_ACK_TIMEOUT)
        {
            BP->GetLearnerBP()->SenderAckTimeout();
            PLGErr("Ack timeout, last acktime %lu now send instanceid %lu", 
                    m_llAbsLastAckTime, llSendInstanceID);
            /*削减m_iAckLead，以便能尽快接收到Ack*/
            CutAckLead();
            m_oLock.UnLock();
            return false;
        }

        BP->GetLearnerBP()->SenderAckDelay();
        
        /*休息一会，降低发送决议的速度*/        
        m_oLock.WaitTime(20);
    }

    m_oLock.UnLock();

    return true;
}

void LearnerSender :: CutAckLead()
{
    int iReceiveAckLead = LearnerReceiver_ACK_LEAD;
    if (m_iAckLead - iReceiveAckLead > iReceiveAckLead)
    {
        m_iAckLead = m_iAckLead - iReceiveAckLead;
    }
}

```

### 请求学习的Learner接收到被学习的Learner发送过来的决议

```
void Learner :: OnSendLearnValue(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnSendLearnValue();

    PLGHead("START Msg.InstanceID %lu Now.InstanceID %lu Msg.ballot_proposalid %lu Msg.ballot_nodeid %lu Msg.ValueSize %zu",
            oPaxosMsg.instanceid(), GetInstanceID(), oPaxosMsg.proposalid(), 
            oPaxosMsg.nodeid(), oPaxosMsg.value().size());

    /*要按序学习*/
    if (oPaxosMsg.instanceid() > GetInstanceID())
    {
        PLGDebug("[Latest Msg] i can't learn");
        return;
    }

    /*@oPaxosMsg.instanceid()对应的决议已经学习过了*/
    if (oPaxosMsg.instanceid() < GetInstanceID())
    {
        PLGDebug("[Lag Msg] no need to learn");
    }
    else
    {
        /*同时在内存中和LogStorage中保存学习到的决议*/
        BallotNumber oBallot(oPaxosMsg.proposalid(), oPaxosMsg.proposalnodeid());
        int ret = m_oLearnerState.LearnValue(oPaxosMsg.instanceid(), oBallot, oPaxosMsg.value(), GetLastChecksum());
        if (ret != 0)
        {
            PLGErr("LearnState.LearnValue fail, ret %d", ret);
            return;
        }
        
        PLGHead("END LearnValue OK, proposalid %lu proposalid_nodeid %lu valueLen %zu", 
                oPaxosMsg.proposalid(), oPaxosMsg.nodeid(), oPaxosMsg.value().size());
    }

    /*如果需要Ack，则发送Ack消息*/
    if (oPaxosMsg.flag() == PaxosMsgFlagType_SendLearnValue_NeedAck)
    {
        //every time' when receive valid need ack learn value, reset noop timeout.
        /*重新添加AskForLearn定时器*/
        Reset_AskforLearn_Noop();

        /*发送Ack消息给被学习的Learner（该函数内部会判断是否真正需要发送）*/
        SendLearnValue_Ack(oPaxosMsg.nodeid());
    }
}

void Learner :: SendLearnValue_Ack(const nodeid_t iSendNodeID)
{
    PLGHead("START LastAck.Instanceid %lu Now.Instanceid %lu", m_llLastAckInstanceID, GetInstanceID());

    /* 自从上次Ack以来，请求学习的Learner学习的决议数目不超过阈值LearnerReceiver_ACK_LEAD,
     * 则无需Ack
     */
    if (GetInstanceID() < m_llLastAckInstanceID + LearnerReceiver_ACK_LEAD)
    {
        PLGImp("No need to ack");
        return;
    }
    
    BP->GetLearnerBP()->SendLearnValue_Ack();

    /*设置当前最大的Ack过的InstanceID*/
    m_llLastAckInstanceID = GetInstanceID();

    PaxosMsg oPaxosMsg;
    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_SendLearnValue_Ack);
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());

    /*发送Ack消息*/
    SendMessage(iSendNodeID, oPaxosMsg);

    PLGHead("End. ok");
}
```

### 被学习的Learner接收到请求学习的Learner的关于“已经成功学习到某个Instance的决议值的Ack”消息

```
void Learner :: OnSendLearnValue_Ack(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnSendLearnValue_Ack();

    PLGHead("Msg.Ack.Instanceid %lu Msg.from_nodeid %lu", oPaxosMsg.instanceid(), oPaxosMsg.nodeid());

    /* 更新LearnerSender中记录的关于当前请求学习的Learner中已经Ack的最大
     * InstanceID(表示已经学习到该Instance的决议了)
     */
    m_oLearnerSender.Ack(oPaxosMsg.instanceid(), oPaxosMsg.nodeid());
}

void LearnerSender :: Ack(const uint64_t llAckInstanceID, const nodeid_t iFromNodeID)
{
    m_oLock.Lock();

    /*当前正处于sending过程中，且已经Confirm了本次学习过程*/
    if (IsIMSending() && m_bIsComfirmed)
    {
        /*接收到的Ack消息是关于本次请求学习的Learner发送过来的*/
        if (m_iSendToNodeID == iFromNodeID)
        {
            /*新Ack的InstanceID确实比之前记录的Ack的InstanceID要大*/
            if (llAckInstanceID > m_llAckInstanceID)
            {
                /*更新最大的Ack的InstanceID*/
                m_llAckInstanceID = llAckInstanceID;
                m_llAbsLastAckTime = Time::GetSteadyClockMS();
                /* 唤醒可能处于等待状态的LearnerSender
                 * 
                 * 在LearnerSender::SendLearnedValue -> LearnerSender::CheckAck中
                 * LearnerSender可能因为发送了很多Instance的决议值，但是一直未接收
                 * 到Ack消息而等待，这里就是为了唤醒这种情况的等待
                 */
                m_oLock.Interupt();
            }
        }
    }

    m_oLock.UnLock();
}    
```

### 被请求学习的Learner上的CheckpointSender发送镜像状态机数据

```
/*发送开始，让目标节点上的CheckpointReceiver做好接收准备*/
int Learner :: SendCheckpointBegin(
        const nodeid_t iSendNodeID,
        const uint64_t llUUID,
        const uint64_t llSequence,
        const uint64_t llCheckpointInstanceID)
{
    CheckpointMsg oCheckpointMsg;

    oCheckpointMsg.set_msgtype(CheckpointMsgType_SendFile);
    oCheckpointMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oCheckpointMsg.set_flag(CheckpointSendFileFlag_BEGIN);
    oCheckpointMsg.set_uuid(llUUID);
    oCheckpointMsg.set_sequence(llSequence);
    oCheckpointMsg.set_checkpointinstanceid(llCheckpointInstanceID);

    return SendMessage(iSendNodeID, oCheckpointMsg, Message_SendType_TCP);
}

/*发送Checkpoint相关的数据*/
int Learner :: SendCheckpoint(
        const nodeid_t iSendNodeID,
        const uint64_t llUUID,
        const uint64_t llSequence,
        const uint64_t llCheckpointInstanceID,
        const uint32_t iChecksum,
        const std::string & sFilePath,
        const int iSMID,
        const uint64_t llOffset,
        const std::string & sBuffer)
{
    CheckpointMsg oCheckpointMsg;

    /*消息类型为CheckpointMsgType_SendFile*/
    oCheckpointMsg.set_msgtype(CheckpointMsgType_SendFile);
    oCheckpointMsg.set_nodeid(m_poConfig->GetMyNodeID());
    /*消息标识：CheckpointSendFileFlag_ING，表示正在发送过程中*/
    oCheckpointMsg.set_flag(CheckpointSendFileFlag_ING);
    oCheckpointMsg.set_uuid(llUUID);
    oCheckpointMsg.set_sequence(llSequence);
    oCheckpointMsg.set_checkpointinstanceid(llCheckpointInstanceID);
    oCheckpointMsg.set_checksum(iChecksum);
    oCheckpointMsg.set_filepath(sFilePath);
    oCheckpointMsg.set_smid(iSMID);
    oCheckpointMsg.set_offset(llOffset);
    oCheckpointMsg.set_buffer(sBuffer);

    /*发送消息*/
    return SendMessage(iSendNodeID, oCheckpointMsg, Message_SendType_TCP);
}

/*结束发送*/
int Learner :: SendCheckpointEnd(
        const nodeid_t iSendNodeID,
        const uint64_t llUUID,
        const uint64_t llSequence,
        const uint64_t llCheckpointInstanceID)
{
    CheckpointMsg oCheckpointMsg;

    /*消息类型：CheckpointMsgType_SendFile*/
    oCheckpointMsg.set_msgtype(CheckpointMsgType_SendFile);
    oCheckpointMsg.set_nodeid(m_poConfig->GetMyNodeID());
    /*消息标识：CheckpointSendFileFlag_END*/
    oCheckpointMsg.set_flag(CheckpointSendFileFlag_END);
    oCheckpointMsg.set_uuid(llUUID);
    oCheckpointMsg.set_sequence(llSequence);
    oCheckpointMsg.set_checkpointinstanceid(llCheckpointInstanceID);

    return SendMessage(iSendNodeID, oCheckpointMsg, Message_SendType_TCP);
}
```


### 接收到被学习的Learner上的CheckpointSender发送过来的“准备发送镜像状态机数据”、“发送镜像状态机的数据”或者“结束发送镜像状态机数据”的消息

```
void Learner :: OnSendCheckpoint(const CheckpointMsg & oCheckpointMsg)
{
    int ret = 0;
    
    if (oCheckpointMsg.flag() == CheckpointSendFileFlag_BEGIN)
    {
        /*接收到begin信号，表示被学习的Learner准备发送镜像状态数据了*/
        ret = OnSendCheckpoint_Begin(oCheckpointMsg);
    }
    else if (oCheckpointMsg.flag() == CheckpointSendFileFlag_ING)
    {
        /*接收到ing信号，表示被学习的Learner又发送过来了新的镜像状态数据*/
        ret = OnSendCheckpoint_Ing(oCheckpointMsg);
    }
    else if (oCheckpointMsg.flag() == CheckpointSendFileFlag_END)
    {
        /*接收到end信号，表示被学习的Learner结束发送镜像状态数据*/
        ret = OnSendCheckpoint_End(oCheckpointMsg);
    }

    if (ret != 0)
    {
        PLGErr("[FAIL] reset checkpoint receiver and reset askforlearn");
        /* 发生错误，则重置CheckpointReceiver和AskForLearn定时器，同时发送携带
         * 失败标识的Ack消息给被请求学习的Learner
         */
        m_oCheckpointReceiver.Reset();
        Reset_AskforLearn_Noop(5000);
        /*发送携带失败标识的Ack消息给被请求学习的Learner*/
        SendCheckpointAck(oCheckpointMsg.nodeid(), oCheckpointMsg.uuid(), 
            oCheckpointMsg.sequence(), CheckpointSendFileAckFlag_Fail);
    }
    else
    {
        /* 成功，则重置AskForLearn定时器，同时发送携带成功标识的Ack消息给
         * 被请求学习的Learner
         */
        SendCheckpointAck(oCheckpointMsg.nodeid(), oCheckpointMsg.uuid(), 
            oCheckpointMsg.sequence(), CheckpointSendFileAckFlag_OK);
        Reset_AskforLearn_Noop(120000);
    }
}

/*开始发送镜像状态机数据*/
int Learner :: OnSendCheckpoint_Begin(const CheckpointMsg & oCheckpointMsg)
{
    /*新建一个CheckpointReceiver*/
    int ret = m_oCheckpointReceiver.NewReceiver(oCheckpointMsg.nodeid(), oCheckpointMsg.uuid());
    if (ret == 0)
    {
        PLGImp("NewReceiver ok");

        /*设置当前实例上看到的最小的执行了checkpoint但是尚未被删除的InstanceID*/
        ret = m_poCheckpointMgr->SetMinChosenInstanceID(oCheckpointMsg.checkpointinstanceid());
        if (ret != 0)
        {
            PLGErr("SetMinChosenInstanceID fail, ret %d CheckpointInstanceID %lu",
                    ret, oCheckpointMsg.checkpointinstanceid());

            return ret;
        }
    }

    return ret;
}

/*发送镜像状态机数据*/
int Learner :: OnSendCheckpoint_Ing(const CheckpointMsg & oCheckpointMsg)
{
    BP->GetCheckpointBP()->OnSendCheckpointOneBlock();
    return m_oCheckpointReceiver.ReceiveCheckpoint(oCheckpointMsg);
}

/*结束发送镜像状态机数据*/
int Learner :: OnSendCheckpoint_End(const CheckpointMsg & oCheckpointMsg)
{
    /*检查是否接收到正确的结束checkpoint的消息*/
    if (!m_oCheckpointReceiver.IsReceiverFinish(oCheckpointMsg.nodeid(), 
                oCheckpointMsg.uuid(), oCheckpointMsg.sequence()))
    {
        PLGErr("receive end msg but receiver not finish");
        return -1;
    }
    
    BP->GetCheckpointBP()->ReceiveCheckpointDone();

    /*对于该group中的每一个状态机执行如下操作：从该状态机相关的文件集合中加载状态机*/
    std::vector<StateMachine *> vecSMList = m_poSMFac->GetSMList();
    for (auto & poSM : vecSMList)
    {
        if (poSM->SMID() == SYSTEM_V_SMID
                || poSM->SMID() == MASTER_V_SMID)
        {
            //system variables sm no checkpoint
            //master variables sm no checkpoint
            continue;
        }

        /*获取该状态机@poSM->SMID()相关的用于临时存放Checkpoint数据的目录*/
        string sTmpDirPath = m_oCheckpointReceiver.GetTmpDirPath(poSM->SMID());
        std::vector<std::string> vecFilePathList;

        /*获取该目录下所有的子文件（会递归到下一级的目录中的子文件）*/
        int ret = FileUtils :: IterDir(sTmpDirPath, vecFilePathList);
        if (ret != 0)
        {
            PLGErr("IterDir fail, dirpath %s", sTmpDirPath.c_str());
        }

        if (vecFilePathList.size() == 0)
        {
            PLGImp("this sm %d have no checkpoint", poSM->SMID());
            continue;
        }
        
        /*从这些文件集合@vecFilePathList中加载状态机数据到@poSM中，该接口需要业务自己提供*/
        ret = poSM->LoadCheckpointState(
                m_poConfig->GetMyGroupIdx(),
                sTmpDirPath,
                vecFilePathList,
                oCheckpointMsg.checkpointinstanceid());
        if (ret != 0)
        {
            BP->GetCheckpointBP()->ReceiveCheckpointAndLoadFail();
            return ret;
        }

    }

    BP->GetCheckpointBP()->ReceiveCheckpointAndLoadSucc();
    PLGImp("All sm load state ok, start to exit process");
    exit(-1);

    return 0;
}

const bool CheckpointReceiver :: IsReceiverFinish(const nodeid_t iSenderNodeID, 
        const uint64_t llUUID, const uint64_t llEndSequence)
{
    /*是否是正确的结束checkpoint的消息*/
    if (iSenderNodeID == m_iSenderNodeID
            && llUUID == m_llUUID
            && llEndSequence == m_llSequence + 1)
    {
        return true;
    }
    else
    {
        return false;
    }
}

int FileUtils :: IterDir(const std::string & sDirPath, std::vector<std::string> & vecFilePathList)
{
    DIR * dir = nullptr;
    struct dirent  * ptr;

    /*打开目录*/
    dir = opendir(sDirPath.c_str());
    if (dir == nullptr)
    {
        return 0;
    }


    /*读取目录下的每一个dir entry*/
    int ret = 0;
    while ((ptr = readdir(dir)) != nullptr)
    {
        /*略过当前目录和父目录*/
        if (strcmp(ptr->d_name, ".") == 0
                || strcmp(ptr->d_name, "..") == 0)
        {
            continue;
        }

        /*获取绝对文件路径*/
        char sChildPath[1024] = {0};
        snprintf(sChildPath, sizeof(sChildPath), "%s/%s", sDirPath.c_str(), ptr->d_name);

        /*检查当前文件是否是目录*/
        bool bIsDir = false;
        ret = FileUtils::IsDir(sChildPath, bIsDir);
        if (ret != 0)
        {
            break;
        }

        if (bIsDir)
        {
            /*如果是目录，则递归下去*/
            ret = IterDir(sChildPath, vecFilePathList);
            if (ret != 0)
            {
                break;
            }
        }
        else
        {
            /*如果是文件，则直接添加到@vecFilePathList中*/
            vecFilePathList.push_back(sChildPath);
        }
    }

    closedir(dir);

    return ret;
}
```

### 请求学习的Learner发送关于“接收到镜像数据”的Ack消息给被学习的Learner

```
int Learner :: SendCheckpointAck(
        const nodeid_t iSendNodeID,
        const uint64_t llUUID,
        const uint64_t llSequence,
        const int iFlag)
{
    CheckpointMsg oCheckpointMsg;

    oCheckpointMsg.set_msgtype(CheckpointMsgType_SendFile_Ack);
    oCheckpointMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oCheckpointMsg.set_uuid(llUUID);
    oCheckpointMsg.set_sequence(llSequence);
    oCheckpointMsg.set_flag(iFlag);

    return SendMessage(iSendNodeID, oCheckpointMsg, Message_SendType_TCP);
}
```

### 接收到请求学习的Learner上的CheckpointReceiver发送过来的Ack消息

```
void Learner :: OnSendCheckpointAck(const CheckpointMsg & oCheckpointMsg)
{
    if (m_poCheckpointSender != nullptr && !m_poCheckpointSender->IsEnd())
    {
        /* 如果当前的CheckpointSender尚未终止，且Ack消息中设置了
         * CheckpointSendFileAckFlag_OK标识，则检查Ack消息是否符合
         * 预期，如果符合预期则尝试继续发送
         */
        if (oCheckpointMsg.flag() == CheckpointSendFileAckFlag_OK)
        {
            m_poCheckpointSender->Ack(oCheckpointMsg.nodeid(), oCheckpointMsg.uuid(), oCheckpointMsg.sequence());
        }
        else
        {
            /*否则，结束CheckpointSender的生命周期，停止发送*/
            m_poCheckpointSender->End();
        }
    }
}
```















