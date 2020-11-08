# 提纲
[toc]

# Proposer类基础
## Proposer定义

```
class Proposer : public Base
{
public:
    ProposerState m_oProposerState;
    MsgCounter m_oMsgCounter;
    Learner * m_poLearner;

    bool m_bIsPreparing;
    bool m_bIsAccepting;

    IOLoop * m_poIOLoop;

    uint32_t m_iPrepareTimerID;
    int m_iLastPrepareTimeoutMs;
    uint32_t m_iAcceptTimerID;
    int m_iLastAcceptTimeoutMs;
    uint64_t m_llTimeoutInstanceID;

    bool m_bCanSkipPrepare;

    bool m_bWasRejectBySomeone;

    TimeStat m_oTimeStat;
};

/*用于管理Proposer的状态*/
class ProposerState
{
public:
    /*本节点上最大的Proposal ID*/
    uint64_t m_llProposalID;
    /*其它节点上最大的Proposal ID*/
    uint64_t m_llHighestOtherProposalID;
    /*提案的value*/
    std::string m_sValue;

    /*其它节点上最大的已经被Promise的提案编号*/
    BallotNumber m_oHighestOtherPreAcceptBallot;

    Config * m_poConfig;
};

/*用于管理Prepare/Accept消息在所有节点的处理情况*/
class MsgCounter
{
public:
    Config * m_poConfig;

    /*成功接收到提议的节点集合*/
    std::set<nodeid_t> m_setReceiveMsgNodeID;
    /*拒绝本次提议的节点集合*/
    std::set<nodeid_t> m_setRejectMsgNodeID;
    /*Promise或者Accept的节点集合*/
    std::set<nodeid_t> m_setPromiseOrAcceptMsgNodeID;
};

```

## ProposerState构造函数

```
ProposerState :: ProposerState(const Config * poConfig)
{
    m_poConfig = (Config *)poConfig;
    m_llProposalID = 1;
    Init();
}

void ProposerState :: Init()
{
    m_llHighestOtherProposalID = 0;
    m_sValue.clear();
}
```

## Proposer构造函数

```
Proposer :: Proposer(
        const Config * poConfig, 
        const MsgTransport * poMsgTransport,
        const Instance * poInstance,
        const Learner * poLearner,
        const IOLoop * poIOLoop)
    : Base(poConfig, poMsgTransport, poInstance), m_oProposerState(poConfig), m_oMsgCounter(poConfig)
{
    m_poLearner = (Learner *)poLearner;
    m_poIOLoop = (IOLoop *)poIOLoop;
    
    m_bIsPreparing = false;
    m_bIsAccepting = false;

    m_bCanSkipPrepare = false;

    InitForNewPaxosInstance();

    m_iPrepareTimerID = 0;
    m_iAcceptTimerID = 0;
    m_llTimeoutInstanceID = 0;

    m_iLastPrepareTimeoutMs = m_poConfig->GetPrepareTimeoutMs();
    m_iLastAcceptTimeoutMs = m_poConfig->GetAcceptTimeoutMs();

    m_bWasRejectBySomeone = false;
}

void Proposer :: InitForNewPaxosInstance()
{
    /*重置消息统计相关*/
    m_oMsgCounter.StartNewRound();
    /*初始化ProposerState*/
    m_oProposerState.Init();

    /*删除Prepare定时器*/
    ExitPrepare();
    /*删除Accept定时器*/
    ExitAccept();
}

void MsgCounter :: StartNewRound()
{
    m_setReceiveMsgNodeID.clear();
    m_setRejectMsgNodeID.clear();
    m_setPromiseOrAcceptMsgNodeID.clear();
}

void ProposerState :: Init()
{
    m_llHighestOtherProposalID = 0;
    m_sValue.clear();
}

void Proposer :: ExitPrepare()
{
    if (m_bIsPreparing)
    {
        /*清除正在Prepare标识*/
        m_bIsPreparing = false;
        /*清理Prepare相关的定时器*/
        m_poIOLoop->RemoveTimer(m_iPrepareTimerID);
    }
}

void Proposer :: ExitAccept()
{
    if (m_bIsAccepting)
    {
        /*清除正在Accept标识*/
        m_bIsAccepting = false;
        /*清理Accept相关的定时器*/
        m_poIOLoop->RemoveTimer(m_iAcceptTimerID);
    }
}
```

# ProposerState方法
## 准备新的Proposal ID

```
void ProposerState :: NewPrepare()
{
    PLGHead("START ProposalID %lu HighestOther %lu MyNodeID %lu",
            m_llProposalID, m_llHighestOtherProposalID, m_poConfig->GetMyNodeID());
        
    uint64_t llMaxProposalID =
        m_llProposalID > m_llHighestOtherProposalID ? m_llProposalID : m_llHighestOtherProposalID;

    m_llProposalID = llMaxProposalID + 1;

    PLGHead("END New.ProposalID %lu", m_llProposalID);

}
```

## 重置本节点了解到的其它节点上已Promise的最大提案

```
void ProposerState :: ResetHighestOtherPreAcceptBallot()
{
    m_oHighestOtherPreAcceptBallot.reset();
}
```

## 设置本节点了解到的其它节点上最大的提案编号

```
void ProposerState :: SetOtherProposalID(const uint64_t llOtherProposalID)
{
    if (llOtherProposalID > m_llHighestOtherProposalID)
    {
        m_llHighestOtherProposalID = llOtherProposalID;
    }
}
```

## 设置本节点上了解到的其它节点已经Accept的最大提案的value

```
void ProposerState :: AddPreAcceptValue(
        const BallotNumber & oOtherPreAcceptBallot, 
        const std::string & sOtherPreAcceptValue)
{
    if (oOtherPreAcceptBallot.isnull())
    {
        return;
    }
    
    if (oOtherPreAcceptBallot > m_oHighestOtherPreAcceptBallot)
    {
        m_oHighestOtherPreAcceptBallot = oOtherPreAcceptBallot;
        m_sValue = sOtherPreAcceptValue;
    }
}
```

# MsgCounter方法
## 初始化MsgCounter

```
void MsgCounter :: StartNewRound()
{
    /*清空如下3个集合*/
    m_setReceiveMsgNodeID.clear();
    m_setRejectMsgNodeID.clear();
    m_setPromiseOrAcceptMsgNodeID.clear();
}
```


# Proposer类方法
## 为新一轮提案做准备

```
Proposer::NewInstance
    - Base::NewInstance
    
void Base :: NewInstance()
{
    /*增长InstanceID*/
    m_llInstanceID++;
    InitForNewPaxosInstance();
}

void Proposer :: InitForNewPaxosInstance()
{
    /*初始化关于Proposer发出去的消息的响应统计相关的信息*/
    m_oMsgCounter.StartNewRound();
    /*初始化ProposerState*/
    m_oProposerState.Init();

    ExitPrepare();
    ExitAccept();
}

void ProposerState :: Init()
{
    m_llHighestOtherProposalID = 0;
    m_sValue.clear();
}

void Proposer :: ExitPrepare()
{
    if (m_bIsPreparing)
    {
        m_bIsPreparing = false;
        
        m_poIOLoop->RemoveTimer(m_iPrepareTimerID);
    }
}

void Proposer :: ExitAccept()
{
    if (m_bIsAccepting)
    {
        m_bIsAccepting = false;
        
        m_poIOLoop->RemoveTimer(m_iAcceptTimerID);
    }
}

```

## 发起提案

```
int Proposer :: NewValue(const std::string & sValue)
{
    BP->GetProposerBP()->NewProposal(sValue);

    if (m_oProposerState.GetValue().size() == 0)
    {
        /*记录本次提案的值*/
        m_oProposerState.SetValue(sValue);
    }

    m_iLastPrepareTimeoutMs = START_PREPARE_TIMEOUTMS;
    m_iLastAcceptTimeoutMs = START_ACCEPT_TIMEOUTMS;

    if (m_bCanSkipPrepare && !m_bWasRejectBySomeone)
    {
        /*直接进入Accept阶段*/
        BP->GetProposerBP()->NewProposalSkipPrepare();

        PLGHead("skip prepare, directly start accept");
        Accept();
    }
    else
    {
        /*首先进入Prepare阶段*/
        //if not reject by someone, no need to increase ballot
        Prepare(m_bWasRejectBySomeone);
    }

    return 0;
}
```

## Proposer Prepare阶段

```
void Proposer :: Prepare(const bool bNeedNewBallot)
{
    PLGHead("START Now.InstanceID %lu MyNodeID %lu State.ProposalID %lu State.ValueLen %zu",
            GetInstanceID(), m_poConfig->GetMyNodeID(), m_oProposerState.GetProposalID(),
            m_oProposerState.GetValue().size());

    BP->GetProposerBP()->Prepare();
    m_oTimeStat.Point();
    
    ExitAccept();
    /*设置正处于Prepare阶段*/
    m_bIsPreparing = true;
    m_bCanSkipPrepare = false;
    m_bWasRejectBySomeone = false;

    /*重置本节点接收到的其它节点已经被Promise的提案编号*/
    m_oProposerState.ResetHighestOtherPreAcceptBallot();
    /* 如果之前被其它节点拒绝过，那么其它节点可能接收到过更大编号的提案，
     * 需要扩大本次提案编号
     */
    if (bNeedNewBallot)
    {
        /*准备Proposal ID*/
        m_oProposerState.NewPrepare();
    }

    PaxosMsg oPaxosMsg;
    oPaxosMsg.set_msgtype(MsgType_PaxosPrepare);
    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalid(m_oProposerState.GetProposalID());

    /*初始化消息处理结果相关的统计信息*/
    m_oMsgCounter.StartNewRound();

    /*添加Prepare相关的定时器到IOLoop中（如果Prepare阶段超时，则重新发送Prepare）*/
    AddPrepareTimer();

    PLGHead("END OK");

    /*广播消息，发送提议给各个节点，首先在本地执行*/
    BroadcastMessage(oPaxosMsg);
}

void Proposer :: AddPrepareTimer(const int iTimeoutMs = 0)
{
    /*如果IOLoop中已经存在Prepare相关的定时器，则首先删除之*/
    if (m_iPrepareTimerID > 0)
    {
        m_poIOLoop->RemoveTimer(m_iPrepareTimerID);
    }

    /* 如果指定了超时时间，则重新添加Prepare相关定时器到IOLoop中，采用指定了超时时间，
     * 当前来说，如果是重试Prepare请求，则会指定超时时间@iTimeoutMs
     */
    if (iTimeoutMs > 0)
    {
        m_poIOLoop->AddTimer(
                iTimeoutMs,
                Timer_Proposer_Prepare_Timeout,
                m_iPrepareTimerID);
        return;
    }

    /* 没有显示指定超时时间，则采用@m_iLastPrepareTimeoutMs作为超时时间，当前来说，
     * 每一个Instance第一次执行Prepare的时候，不会指定超时时间@iTimeoutMs
     */
    m_poIOLoop->AddTimer(
            m_iLastPrepareTimeoutMs,
            Timer_Proposer_Prepare_Timeout,
            m_iPrepareTimerID);

    /*设置超时的Prepare所在的InstanceID*/
    m_llTimeoutInstanceID = GetInstanceID();

    PLGHead("timeoutms %d", m_iLastPrepareTimeoutMs);

    m_iLastPrepareTimeoutMs *= 2;
    if (m_iLastPrepareTimeoutMs > MAX_PREPARE_TIMEOUTMS)
    {
        m_iLastPrepareTimeoutMs = MAX_PREPARE_TIMEOUTMS;
    }
}

```

## Proposer处理Prepare响应

```
void Proposer :: OnPrepareReply(const PaxosMsg & oPaxosMsg)
{
    PLGHead("START Msg.ProposalID %lu State.ProposalID %lu Msg.from_nodeid %lu RejectByPromiseID %lu",
            oPaxosMsg.proposalid(), m_oProposerState.GetProposalID(), 
            oPaxosMsg.nodeid(), oPaxosMsg.rejectbypromiseid());

    BP->GetProposerBP()->OnPrepareReply();
    
    if (!m_bIsPreparing)
    {
        BP->GetProposerBP()->OnPrepareReplyButNotPreparing();
        //PLGErr("Not preparing, skip this msg");
        return;
    }

    /*ProposalID不匹配，其中ProposalID在Proposer::Prepare ->  ProposerState::NewPrepare中设定*/
    if (oPaxosMsg.proposalid() != m_oProposerState.GetProposalID())
    {
        BP->GetProposerBP()->OnPrepareReplyNotSameProposalIDMsg();
        //PLGErr("ProposalID not same, skip this msg");
        return;
    }

    /*更新MsgCounter中关于接收到响应的节点集合*/
    m_oMsgCounter.AddReceive(oPaxosMsg.nodeid());

    if (oPaxosMsg.rejectbypromiseid() == 0)
    {
        /*提案被Promise*/
        BallotNumber oBallot(oPaxosMsg.preacceptid(), oPaxosMsg.preacceptnodeid());
        PLGDebug("[Promise] PreAcceptedID %lu PreAcceptedNodeID %lu ValueSize %zu", 
                oPaxosMsg.preacceptid(), oPaxosMsg.preacceptnodeid(), oPaxosMsg.value().size());
        /*更新Promise的节点集合*/
        m_oMsgCounter.AddPromiseOrAccept(oPaxosMsg.nodeid());
        /*更新本地记录的关于其它节点已经Accepted的最大提案编号和提案value*/
        m_oProposerState.AddPreAcceptValue(oBallot, oPaxosMsg.value());
    }
    else
    {
        /*提案被拒绝*/
        PLGDebug("[Reject] RejectByPromiseID %lu", oPaxosMsg.rejectbypromiseid());
        /*更新Reject节点集合*/
        m_oMsgCounter.AddReject(oPaxosMsg.nodeid());
        /*至少被一个节点拒绝了，接下来的提案不能直接跳过Prepare阶段了*/
        m_bWasRejectBySomeone = true;
        /*设置其它节点已经Promise的最大Proposal ID*/
        m_oProposerState.SetOtherProposalID(oPaxosMsg.rejectbypromiseid());
    }

    /*Promise响应的节点计数是否达到大多数*/
    if (m_oMsgCounter.IsPassedOnThisRound())
    {
        int iUseTimeMs = m_oTimeStat.Point();
        BP->GetProposerBP()->PreparePass(iUseTimeMs);
        PLGImp("[Pass] start accept, usetime %dms", iUseTimeMs);
        /* 设置可以跳过Prepare阶段（但实际上是否可以跳过Prepare阶段，同时取决于
         * @m_bCanSkipPrepare和@m_bWasRejectBySomeone）
         */
        m_bCanSkipPrepare = true;
        /*发送Accept消息*/
        Accept();
    }
    else if (m_oMsgCounter.IsRejectedOnThisRound()
            || m_oMsgCounter.IsAllReceiveOnThisRound())
    {
        /* FIXME：
         * 如果接收到所有节点的响应，但是Promise的节点数目和Reject的节点数目都不
         * 达到大多数，而任何一个节点，要么处于Promise集合，要么处于Reject集合啊，
         * 为什么会出现这种情况呢？难道是所有节点集合为偶数？抑或存在节点down掉？
         */
        BP->GetProposerBP()->PrepareNotPass();
        PLGImp("[Not Pass] wait 30ms and restart prepare");
        /* 在IOLoop中添加关于Prepare的定时器，等待10 ~ 40 ms，当定时器定时时间到的
         * 时候重新进行Prepare，参考Instance::OnTimeout
         */
        AddPrepareTimer(OtherUtils::FastRand() % 30 + 10);
    }

    PLGHead("END");
}
```

## Proposer处理过期的Prepare响应
过期的响应的一个重要特征是：响应所对应的InstanceID和当前的InstanceID不一样。

```
void Proposer :: OnExpiredPrepareReply(const PaxosMsg & oPaxosMsg)
{
    if (oPaxosMsg.rejectbypromiseid() != 0)
    {
        /*只处理过期的Reject的情形*/
        
        PLGDebug("[Expired Prepare Reply Reject] RejectByPromiseID %lu", oPaxosMsg.rejectbypromiseid());
        /*至少被一个节点拒绝了，接下来的提案不能直接跳过Prepare阶段了*/
        m_bWasRejectBySomeone = true;
        /*尝试更新其它节点上看到的最大Proposal ID*/
        m_oProposerState.SetOtherProposalID(oPaxosMsg.rejectbypromiseid());
    }
}
```

## Proposer处理Prepare阶段定时器事件
在Prepare阶段，Proposer在发送Prepare消息之前会启动定时器，如果Proposer在某个超时时间内未接收到多数派的Promise响应，则会重新发起Prepare请求，如果成功接收到多数派的Promise响应，则该定时器会被移除。
```
void Proposer :: OnPrepareTimeout()
{
    PLGHead("OK");

    /*InstanceID已经改变，直接退出*/
    if (GetInstanceID() != m_llTimeoutInstanceID)
    {
        PLGErr("TimeoutInstanceID %lu not same to NowInstanceID %lu, skip",
                m_llTimeoutInstanceID, GetInstanceID());
        return;
    }

    BP->GetProposerBP()->PrepareTimeout();
    
    /*重新发送Prepare消息*/
    Prepare(m_bWasRejectBySomeone);
}
```


## Proposer Accept阶段

```
void Proposer :: Accept()
{
    PLGHead("START ProposalID %lu ValueSize %zu ValueLen %zu", 
            m_oProposerState.GetProposalID(), m_oProposerState.GetValue().size(), m_oProposerState.GetValue().size());

    BP->GetProposerBP()->Accept();
    m_oTimeStat.Point();
    
    /*设置进入Accept过程（这里面会删除Prepare定时器）*/
    ExitPrepare();
    m_bIsAccepting = true;
    
    /*设置Accept消息*/
    PaxosMsg oPaxosMsg;
    oPaxosMsg.set_msgtype(MsgType_PaxosAccept);
    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalid(m_oProposerState.GetProposalID());
    oPaxosMsg.set_value(m_oProposerState.GetValue());
    oPaxosMsg.set_lastchecksum(GetLastChecksum());

    /*重置MsgCounter，即重置关于该Accept消息响应的统计信息*/
    m_oMsgCounter.StartNewRound();

    /* 在IOLoop中添加一个Accept相关的定时器，如果该定时器在超市之前未被移除，
     * 则会重新执行Accept
     */
    AddAcceptTimer();

    PLGHead("END");

    /*广播消息，最后在本地执行*/
    BroadcastMessage(oPaxosMsg, BroadcastMessage_Type_RunSelf_Final);
}

void Proposer :: AddAcceptTimer(const int iTimeoutMs = 0)
{
    /*跟AddPrepareTimer类似*/
    
    /*如果已经存在Accept的定时器，则先删除之*/
    if (m_iAcceptTimerID > 0)
    {
        m_poIOLoop->RemoveTimer(m_iAcceptTimerID);
    }

    /* 如果指定了超时时间，则重新添加Accept相关定时器到IOLoop中，采用指定的超时时间，
     * 当前来说，如果是重试Accept请求，则会指定超时时间@iTimeoutMs
     */
    if (iTimeoutMs > 0)
    {
        m_poIOLoop->AddTimer(
                iTimeoutMs,
                Timer_Proposer_Accept_Timeout,
                m_iAcceptTimerID);
        return;
    }

    /* 没有显示指定超时时间，则采用@m_iLastAcceptTimeoutMs作为超时时间，当前来说，
     * 每一个Instance第一次执行Accept的时候，不会指定超时时间@iTimeoutMs
     */
    m_poIOLoop->AddTimer(
            m_iLastAcceptTimeoutMs,
            Timer_Proposer_Accept_Timeout,
            m_iAcceptTimerID);

    /*设置当前定时器对应的InstanceID*/
    m_llTimeoutInstanceID = GetInstanceID();
    
    PLGHead("timeoutms %d", m_iLastPrepareTimeoutMs);

    m_iLastAcceptTimeoutMs *= 2;
    if (m_iLastAcceptTimeoutMs > MAX_ACCEPT_TIMEOUTMS)
    {
        m_iLastAcceptTimeoutMs = MAX_ACCEPT_TIMEOUTMS;
    }
}
```

## Proposer处理Accept响应

```
void Proposer :: OnAcceptReply(const PaxosMsg & oPaxosMsg)
{
    PLGHead("START Msg.ProposalID %lu State.ProposalID %lu Msg.from_nodeid %lu RejectByPromiseID %lu",
            oPaxosMsg.proposalid(), m_oProposerState.GetProposalID(), 
            oPaxosMsg.nodeid(), oPaxosMsg.rejectbypromiseid());

    BP->GetProposerBP()->OnAcceptReply();

    if (!m_bIsAccepting)
    {
        //PLGErr("Not proposing, skip this msg");
        BP->GetProposerBP()->OnAcceptReplyButNotAccepting();
        return;
    }

    /*ProposalID不匹配，其中ProposalID在Proposer::Prepare ->  ProposerState::NewPrepare中设定*/
    if (oPaxosMsg.proposalid() != m_oProposerState.GetProposalID())
    {
        //PLGErr("ProposalID not same, skip this msg");
        BP->GetProposerBP()->OnAcceptReplyNotSameProposalIDMsg();
        return;
    }

    /*更新接收到Accept响应的节点集合*/
    m_oMsgCounter.AddReceive(oPaxosMsg.nodeid());

    if (oPaxosMsg.rejectbypromiseid() == 0)
    {
        /*更新已经Accept的节点集合*/
        PLGDebug("[Accept]");
        m_oMsgCounter.AddPromiseOrAccept(oPaxosMsg.nodeid());
    }
    else
    {
        /*更新Reject的节点集合*/
        PLGDebug("[Reject]");
        m_oMsgCounter.AddReject(oPaxosMsg.nodeid());
        /*至少被一个节点拒绝了，接下来的提案不能直接跳过Prepare阶段了*/
        m_bWasRejectBySomeone = true;
        /*设置其它节点已经Promise的最大Proposal ID*/
        m_oProposerState.SetOtherProposalID(oPaxosMsg.rejectbypromiseid());
    }

    if (m_oMsgCounter.IsPassedOnThisRound())
    {
        /*Accept消息被多数节点通过*/
        int iUseTimeMs = m_oTimeStat.Point();
        BP->GetProposerBP()->AcceptPass(iUseTimeMs);
        PLGImp("[Pass] Start send learn, usetime %dms", iUseTimeMs);
        
        /*退出Accept状态（同时删除Accept的定时器）*/
        ExitAccept();
        
        /*Proposer向Learner发送本次提案成功的消息*/
        m_poLearner->ProposerSendSuccess(GetInstanceID(), m_oProposerState.GetProposalID());
    }
    else if (m_oMsgCounter.IsRejectedOnThisRound()
            || m_oMsgCounter.IsAllReceiveOnThisRound())
    {
        /*本次的Accept未被通过，等待一段时间（10ms ~ 40ms）后重新进入Prepare阶段*/
        BP->GetProposerBP()->AcceptNotPass();
        PLGImp("[Not pass] wait 30ms and Restart prepare");
        AddAcceptTimer(OtherUtils::FastRand() % 30 + 10);
    }

    PLGHead("END");
}
```

## Proposer处理过期的Accept响应

```
void Proposer :: OnExpiredAcceptReply(const PaxosMsg & oPaxosMsg)
{
    /*只处理过期的Reject的情形*/
        
    if (oPaxosMsg.rejectbypromiseid() != 0)
    {
        PLGDebug("[Expired Accept Reply Reject] RejectByPromiseID %lu", oPaxosMsg.rejectbypromiseid());
        /*至少被一个节点拒绝了，接下来的提案不能直接跳过Prepare阶段了*/
        m_bWasRejectBySomeone = true;
        /*尝试更新其它节点上看到的最大Proposal ID*/
        m_oProposerState.SetOtherProposalID(oPaxosMsg.rejectbypromiseid());
    }
}
```

## Proposer处理Accept阶段定时器事件
在Accept阶段，Proposer在发送Accept消息之前会启动定时器，如果Proposer在某个超时时间内未接收到多数派的Accepted响应，则会重新发起Prepare请求，如果成功接收到多数派的Accepted响应，则该定时器会被移除。
```
void Proposer :: OnAcceptTimeout()
{
    PLGHead("OK");
    
    /*当前的InstanceID与定时器相关的InstanceID不同，直接退出*/
    if (GetInstanceID() != m_llTimeoutInstanceID)
    {
        PLGErr("TimeoutInstanceID %lu not same to NowInstanceID %lu, skip",
                m_llTimeoutInstanceID, GetInstanceID());
        return;
    }
    
    
    BP->GetProposerBP()->AcceptTimeout();
    
    /*重新进入Prepare阶段*/
    Prepare(m_bWasRejectBySomeone);
}
```