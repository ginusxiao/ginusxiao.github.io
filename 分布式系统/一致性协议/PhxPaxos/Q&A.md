# 提纲
[toc]

# PhxPaxos中InstanceID是怎么管理的？
PhxPaxos中Proposer，Acceptor和Learner都需要维护InstanceID，但是都是通过继承Base类来实现InstanceID的管理，如下：

```
class Base
{
protected:
    Config * m_poConfig;
    MsgTransport * m_poMsgTransport;
    Instance * m_poInstance;

private:
    /*InstanceID*/
    uint64_t m_llInstanceID;
    bool m_bIsTestMode;
};

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

class Acceptor : public Base
{
public:
    AcceptorState m_oAcceptorState;
};

class Learner : public Base
{
private:
    LearnerState m_oLearnerState;

    Acceptor * m_poAcceptor;
    PaxosLog m_oPaxosLog;

    uint32_t m_iAskforlearn_noopTimerID;
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
```

首先看Base类总提供了哪些操作InstanceID的接口：

```
/*获取InstanceID*/
uint64_t Base :: GetInstanceID()
{
    return m_llInstanceID;
}

/*设置InstanceID*/
void Base :: SetInstanceID(const uint64_t llInstanceID)
{
    m_llInstanceID = llInstanceID;
}

/*新建Instance*/
void Base :: NewInstance()
{
    m_llInstanceID++;
    InitForNewPaxosInstance();
}
```

接着看代码中哪些地方使用了上述的SetInstanceID和NewInstance接口：
在Instance初始化上下文中--->

```
int Instance :: Init()
{
    /* 从Acceptor中加载它所看到的最大的InstanceID，这里面会设置
     * Acceptor看到的最大InstanceID
     */
    int ret = m_oAcceptor.Init();
    if (ret != 0)
    {
        PLGErr("Acceptor.Init fail, ret %d", ret);
        return ret;
    }

    ret = m_oCheckpointMgr.Init();
    if (ret != 0)
    {
        PLGErr("CheckpointMgr.Init fail, ret %d", ret);
        return ret;
    }

    /*最大的checkpoint的InstanceID + 1*/
    uint64_t llCPInstanceID = m_oCheckpointMgr.GetCheckpointInstanceID() + 1;

    PLGImp("Acceptor.OK, Log.InstanceID %lu Checkpoint.InstanceID %lu", 
            m_oAcceptor.GetInstanceID(), llCPInstanceID);

    uint64_t llNowInstanceID = llCPInstanceID;
    if (llNowInstanceID < m_oAcceptor.GetInstanceID())
    {
        ret = PlayLog(llNowInstanceID, m_oAcceptor.GetInstanceID());
        if (ret != 0)
        {
            return ret;
        }

        PLGImp("PlayLog OK, begin instanceid %lu end instanceid %lu", llNowInstanceID, m_oAcceptor.GetInstanceID());

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
        
        /*设置Acceptor看到的最大InstanceID*/
        m_oAcceptor.SetInstanceID(llNowInstanceID);
    }

    PLGImp("NowInstanceID %lu", llNowInstanceID);

    /* 分别设置Learner，Proposer看到的最大InstanceID（和Acceptor看到的最大InstanceID
     * 相同），Instance::Init结束后，Proposer，Acceptor和Learner都具有相同的InstanceID
     */
    m_oLearner.SetInstanceID(llNowInstanceID);
    m_oProposer.SetInstanceID(llNowInstanceID);
    m_oProposer.SetStartProposalID(m_oAcceptor.GetAcceptorState()->GetPromiseBallot().m_llProposalID + 1);

    ......

    return 0;
}

int Acceptor :: Init()
{
    uint64_t llInstanceID = 0;
    /*从AcceptorState中获取最大的InstanceID*/
    int ret = m_oAcceptorState.Load(llInstanceID);
    if (ret != 0)
    {
        NLErr("Load State fail, ret %d", ret);
        return ret;
    }

    if (llInstanceID == 0)
    {
        PLGImp("Empty database");
    }

    /*设置其当前最大InstanceID*/
    SetInstanceID(llInstanceID);

    PLGImp("OK");

    return 0;
}
```

在Learner上下文中：
```
Instance :: ReceiveMsgForLearner
    - Instance :: NewInstance
    
void Instance :: NewInstance()
{
    /*分别增加Acceptor，Learner和Proposer看到的InstanceID*/
    m_oAcceptor.NewInstance();
    m_oLearner.NewInstance();
    m_oProposer.NewInstance();
}
```

# SystemVariables和MasterVariables分别是干什么用的？
SystemVariables是集群成员组管理使用的；
MasterVariables是集群Master管理使用的；


# 如果在提案在Prepare + Accept阶段成功，但是在运用决议阶段失败会怎样？


