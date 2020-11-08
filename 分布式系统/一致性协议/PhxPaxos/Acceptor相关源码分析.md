# 提纲
[toc]

# Acceptor相关类定义
## Acceptor类定义
```
class Acceptor : public Base
{
    AcceptorState m_oAcceptorState;
};
```

## Base类定义
Base类作为Acceptor类、Learner类和Proposer类的共同的基类，其中包含Instance相关的配置(m_poConfig)和网络(m_poMsgTransport)信息的指针。

```
class Base
{
protected:
    Config * m_poConfig;
    MsgTransport * m_poMsgTransport;
    Instance * m_poInstance;

private:
    uint64_t m_llInstanceID;
    bool m_bIsTestMode;
};
```

## AcceptorState
系统中采用AcceptorState来管理Acceptor相关的状态信息，比如，已经promise的提案编号，已经accept的提案编号，已经accept的value等。
```
class AcceptorState
{
public:
    /*已经promise过的最大提案*/
    BallotNumber m_oPromiseBallot;
    /*已经Accept过的最大提案*/
    BallotNumber m_oAcceptedBallot;
    /*已经Accept过的value*/
    std::string m_sAcceptedValue;
    uint32_t m_iChecksum;

    Config * m_poConfig;
    /*对LogStorage的封装*/
    PaxosLog m_oPaxosLog;

    int m_iSyncTimes;
};
```

## BallotNumber
系统中采用BallotNumber来标记一个提案编号，其中包含提案的ID和发起提案的节点ID。
```
class BallotNumber
{
public:
    uint64_t m_llProposalID;
    nodeid_t m_llNodeID;
};
```


# Acceptor相关类构造函数
## Base类构造函数

```
Base :: Base(const Config * poConfig, const MsgTransport * poMsgTransport, const Instance * poInstance)
{
    m_poConfig = (Config *)poConfig;
    m_poMsgTransport = (MsgTransport *)poMsgTransport;
    m_poInstance = (Instance *)poInstance;

    m_llInstanceID = 0;

    m_bIsTestMode = false;
}
```

## Acceptor类构造函数

```
Acceptor :: Acceptor(
        const Config * poConfig, 
        const MsgTransport * poMsgTransport, 
        const Instance * poInstance,
        const LogStorage * poLogStorage)
    /*调用基类Base的构造函数*/
    : Base(poConfig, poMsgTransport, poInstance),
    /* 从PNode :: Init -> Group :: Group -> Instance :: Instance -> Acceptor :: Acceptor
     * ->  AcceptorState :: AcceptorState调用链来看，AcceptorState信息就是存储于它所在的
     * paxos group对应的LogStorage中
     */
    m_oAcceptorState(poConfig, poLogStorage)
{
}
```

# Acceptor类方法
## Acceptor初始化

```
int Acceptor :: Init()
{
    uint64_t llInstanceID = 0;
    /* 从leveldb中获取当前最大的InstanceID，从LogStore中获取该Instance对应的状态信息，
     * 并保存在@m_oAcceptorState中
     */
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

    /*设置实例ID为@llInstanceID*/
    SetInstanceID(llInstanceID);

    PLGImp("OK");

    return 0;
}

int AcceptorState :: Load(uint64_t & llInstanceID)
{
    /*从leveldb中读取最大InstanceID*/
    int ret = m_oPaxosLog.GetMaxInstanceIDFromLog(m_poConfig->GetMyGroupIdx(), llInstanceID);
    if (ret != 0 && ret != 1)
    {
        PLGErr("Load max instance id fail, ret %d", ret);
        return ret;
    }

    if (ret == 1)
    {
        /*leveldb中不存在最大InstanceID，则直接初始化当前InstanceID为0*/
        PLGErr("empty database");
        llInstanceID = 0;
        return 0;
    }

    AcceptorStateData oState;
    /*实际上调用的是MultiDatabase::Get*/
    ret = m_oPaxosLog.ReadState(m_poConfig->GetMyGroupIdx(), llInstanceID, oState);
    if (ret != 0)
    {
        return ret;
    }
    
    m_oPromiseBallot.m_llProposalID = oState.promiseid();
    m_oPromiseBallot.m_llNodeID = oState.promisenodeid();
    m_oAcceptedBallot.m_llProposalID = oState.acceptedid();
    m_oAcceptedBallot.m_llNodeID = oState.acceptednodeid();
    m_sAcceptedValue = oState.acceptedvalue();
    m_iChecksum = oState.checksum();
    
    PLGImp("GroupIdx %d InstanceID %lu PromiseID %lu PromiseNodeID %lu"
           " AccectpedID %lu AcceptedNodeID %lu ValueLen %zu Checksum %u", 
            m_poConfig->GetMyGroupIdx(), llInstanceID, m_oPromiseBallot.m_llProposalID, 
            m_oPromiseBallot.m_llNodeID, m_oAcceptedBallot.m_llProposalID, 
            m_oAcceptedBallot.m_llNodeID, m_sAcceptedValue.size(), m_iChecksum);
    
    return 0;
}

int PaxosLog :: GetMaxInstanceIDFromLog(const int iGroupIdx, uint64_t & llInstanceID)
{
    const int m_iMyGroupIdx = iGroupIdx;
    
    /* 结合AcceptorState构造函数中的说明可知，@m_poLogStorage实际上就是来自于
     * PNode::m_oDefaultLogStorage，所以这里实际上最终调用的是MultiDatabase::
     * GetMaxInstanceID -> Database::GetMaxInstanceID，参考“Storage相关源码分析”
     */
    int ret = m_poLogStorage->GetMaxInstanceID(iGroupIdx, llInstanceID);
    if (ret != 0 && ret != 1)
    {
        /*读取leveldb失败*/
        PLG1Err("DB.GetMax fail, groupidx %d ret %d", iGroupIdx, ret);
    }
    else if (ret == 1)
    {
        /*不存在最大InstanceID*/
        PLG1Debug("MaxInstanceID not exist, groupidx %d", iGroupIdx);
    }
    else
    {
        PLG1Imp("OK, MaxInstanceID %llu groupidsx %d", llInstanceID, iGroupIdx);
    }

    return ret;
}
```

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

void Acceptor :: InitForNewPaxosInstance()
{
    /*初始化AcceptorState*/
    m_oAcceptorState.Init();
}

void AcceptorState :: Init()
{
    m_oAcceptedBallot.reset();
    m_sAcceptedValue = "";
    m_iChecksum = 0;
}
```

## Acceptor处理Prepare消息

```
int Acceptor :: OnPrepare(const PaxosMsg & oPaxosMsg)
{
    PLGHead("START Msg.InstanceID %lu Msg.from_nodeid %lu Msg.ProposalID %lu",
            oPaxosMsg.instanceid(), oPaxosMsg.nodeid(), oPaxosMsg.proposalid());

    BP->GetAcceptorBP()->OnPrepare();
    
    /*设置响应消息*/
    PaxosMsg oReplyPaxosMsg;
    oReplyPaxosMsg.set_instanceid(GetInstanceID());
    oReplyPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oReplyPaxosMsg.set_proposalid(oPaxosMsg.proposalid());
    oReplyPaxosMsg.set_msgtype(MsgType_PaxosPrepareReply);

    /*本次Prepare请求对应的提案*/
    BallotNumber oBallot(oPaxosMsg.proposalid(), oPaxosMsg.nodeid());
    
    if (oBallot >= m_oAcceptorState.GetPromiseBallot())
    {
        /*本次提案编号不小于已经承诺的提案*/
        
        /*在响应中设置当前已经Accepted的提案编号和value*/
        oReplyPaxosMsg.set_preacceptid(m_oAcceptorState.GetAcceptedBallot().m_llProposalID);
        oReplyPaxosMsg.set_preacceptnodeid(m_oAcceptorState.GetAcceptedBallot().m_llNodeID);
        if (m_oAcceptorState.GetAcceptedBallot().m_llProposalID > 0)
        {
            oReplyPaxosMsg.set_value(m_oAcceptorState.GetAcceptedValue());
        }

        /*承诺本次提案*/
        m_oAcceptorState.SetPromiseBallot(oBallot);

        /*将AcceptorState信息写入LogStorage*/
        int ret = m_oAcceptorState.Persist(GetInstanceID(), GetLastChecksum());
        if (ret != 0)
        {
            BP->GetAcceptorBP()->OnPreparePersistFail();
            PLGErr("Persist fail, Now.InstanceID %lu ret %d",
                    GetInstanceID(), ret);
            
            return -1;
        }

        BP->GetAcceptorBP()->OnPreparePass();
    }
    else
    {
        /*本次提案编号小于已经承诺的提案，则拒绝之*/
        
        BP->GetAcceptorBP()->OnPrepareReject();

        PLGDebug("[Reject] State.PromiseID %lu State.PromiseNodeID %lu", 
                m_oAcceptorState.GetPromiseBallot().m_llProposalID, 
                m_oAcceptorState.GetPromiseBallot().m_llNodeID);
        
        /*在响应中返回已经承诺的最大编号*/
        oReplyPaxosMsg.set_rejectbypromiseid(m_oAcceptorState.GetPromiseBallot().m_llProposalID);
    }

    nodeid_t iReplyNodeID = oPaxosMsg.nodeid();

    PLGHead("END Now.InstanceID %lu ReplyNodeID %lu",
            GetInstanceID(), oPaxosMsg.nodeid());;

    /*发送响应，目标节点为@iReplyNodeID，实际上调用的是Base::SendMessage*/
    SendMessage(iReplyNodeID, oReplyPaxosMsg);

    return 0;
}
```

## Acceptor处理Accept消息

```
void Acceptor :: OnAccept(const PaxosMsg & oPaxosMsg)
{
    PLGHead("START Msg.InstanceID %lu Msg.from_nodeid %lu Msg.ProposalID %lu Msg.ValueLen %zu",
            oPaxosMsg.instanceid(), oPaxosMsg.nodeid(), oPaxosMsg.proposalid(), oPaxosMsg.value().size());

    BP->GetAcceptorBP()->OnAccept();

    /*设置Accept响应*/
    PaxosMsg oReplyPaxosMsg;
    oReplyPaxosMsg.set_instanceid(GetInstanceID());
    oReplyPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oReplyPaxosMsg.set_proposalid(oPaxosMsg.proposalid());
    oReplyPaxosMsg.set_msgtype(MsgType_PaxosAcceptReply);

    /*收到的提案编号*/
    BallotNumber oBallot(oPaxosMsg.proposalid(), oPaxosMsg.nodeid());

    /*收到的提案编号不小于已经Promise的提案编号*/
    if (oBallot >= m_oAcceptorState.GetPromiseBallot())
    {
        /*设置已经Promise的最大编号、已经Accept的最大编号和已经Accept的value*/
        m_oAcceptorState.SetPromiseBallot(oBallot);
        m_oAcceptorState.SetAcceptedBallot(oBallot);
        m_oAcceptorState.SetAcceptedValue(oPaxosMsg.value());
        
        /*将AcceptorState信息写入LogStorage*/
        int ret = m_oAcceptorState.Persist(GetInstanceID(), GetLastChecksum());
        if (ret != 0)
        {
            BP->GetAcceptorBP()->OnAcceptPersistFail();

            PLGErr("Persist fail, Now.InstanceID %lu ret %d",
                    GetInstanceID(), ret);
            
            return;
        }

        BP->GetAcceptorBP()->OnAcceptPass();
    }
    else
    {
        /*本次提案编号小于已经承诺的提案，则拒绝之*/
        
        BP->GetAcceptorBP()->OnAcceptReject();

        PLGDebug("[Reject] State.PromiseID %lu State.PromiseNodeID %lu", 
                m_oAcceptorState.GetPromiseBallot().m_llProposalID, 
                m_oAcceptorState.GetPromiseBallot().m_llNodeID);
                
        /*在响应中返回已经承诺的最大编号*/
        oReplyPaxosMsg.set_rejectbypromiseid(m_oAcceptorState.GetPromiseBallot().m_llProposalID);
    }

    nodeid_t iReplyNodeID = oPaxosMsg.nodeid();

    PLGHead("END Now.InstanceID %lu ReplyNodeID %lu",
            GetInstanceID(), oPaxosMsg.nodeid());

    /*发送响应，目标节点为@iReplyNodeID，实际上调用的是Base::SendMessage*/
    SendMessage(iReplyNodeID, oReplyPaxosMsg);
}
```

## 持久化AcceptorState到LogStorage

```
int AcceptorState :: Persist(const uint64_t llInstanceID, const uint32_t iLastChecksum)
{
    /*在@iLastChecksum的基础上计算出新的checksum*/
    if (llInstanceID > 0 && iLastChecksum == 0)
    {
        m_iChecksum = 0;
    }
    else if (m_sAcceptedValue.size() > 0)
    {
        m_iChecksum = crc32(iLastChecksum, (const uint8_t *)m_sAcceptedValue.data(), m_sAcceptedValue.size(), CRC32SKIP);
    }
    
    /* 设置AcceptorStateData，实际上是将AcceptorState中的信息转换成AcceptorStateData结构，
     * 再以AcceptorStateData为媒介存储到LogStorage中
     */
    AcceptorStateData oState;
    oState.set_instanceid(llInstanceID);
    oState.set_promiseid(m_oPromiseBallot.m_llProposalID);
    oState.set_promisenodeid(m_oPromiseBallot.m_llNodeID);
    oState.set_acceptedid(m_oAcceptedBallot.m_llProposalID);
    oState.set_acceptednodeid(m_oAcceptedBallot.m_llNodeID);
    oState.set_acceptedvalue(m_sAcceptedValue);
    oState.set_checksum(m_iChecksum);

    /*获取当前Config中关于是否同步的设置*/
    WriteOptions oWriteOptions;
    oWriteOptions.bSync = m_poConfig->LogSync();
    if (oWriteOptions.bSync)
    {
        /* 如果需要同步，则更新同步计数，如果同步计数达到阈值，则在写入之后进行
         * 一次同步，否则写入之后不立即同步
         */
        m_iSyncTimes++;
        if (m_iSyncTimes > m_poConfig->SyncInterval())
        {
            m_iSyncTimes = 0;
        }
        else
        {
            /*本次写入之后不立即同步*/
            oWriteOptions.bSync = false;
        }
    }

    /*写入LogStorage，根据@oWriteOptions.bSync来控制是否在写之后同步*/
    int ret = m_oPaxosLog.WriteState(oWriteOptions, m_poConfig->GetMyGroupIdx(), llInstanceID, oState);
    if (ret != 0)
    {
        return ret;
    }
    
    PLGImp("GroupIdx %d InstanceID %lu PromiseID %lu PromiseNodeID %lu "
            "AccectpedID %lu AcceptedNodeID %lu ValueLen %zu Checksum %u", 
            m_poConfig->GetMyGroupIdx(), llInstanceID, m_oPromiseBallot.m_llProposalID, 
            m_oPromiseBallot.m_llNodeID, m_oAcceptedBallot.m_llProposalID, 
            m_oAcceptedBallot.m_llNodeID, m_sAcceptedValue.size(), m_iChecksum);
    
    return 0;
}
```