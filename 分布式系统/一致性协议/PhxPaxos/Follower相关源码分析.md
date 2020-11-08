# 提纲
[toc]

# 说明
PhxPaxos中有Follower这个角色，但是没有为之单独定义类，Follower角色功能的实现是借助Learner类来实现的。

# Follower相关的方法
## 接收到Learner发送过来的由Learner学习到的决议（message type：MsgType_PaxosLearner_SendLearnValue）

```
void Learner :: OnSendLearnValue(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnSendLearnValue();

    PLGHead("START Msg.InstanceID %lu Now.InstanceID %lu Msg.ballot_proposalid %lu Msg.ballot_nodeid %lu Msg.ValueSize %zu",
            oPaxosMsg.instanceid(), GetInstanceID(), oPaxosMsg.proposalid(), 
            oPaxosMsg.nodeid(), oPaxosMsg.value().size());

    /*比Follower上的Instance要新，但是要先学习GetInstanceID()对应的Instance的决议*/
    if (oPaxosMsg.instanceid() > GetInstanceID())
    {
        PLGDebug("[Latest Msg] i can't learn");
        return;
    }

    /*比Follower上的Instance要新，不用再学习旧的了*/
    if (oPaxosMsg.instanceid() < GetInstanceID())
    {
        PLGDebug("[Lag Msg] no need to learn");
    }
    else
    {
        /*本次要学习的决议的提案编号*/
        BallotNumber oBallot(oPaxosMsg.proposalid(), oPaxosMsg.proposalnodeid());
        /*学习决议，同时保存在内存中和LogStorage中*/
        int ret = m_oLearnerState.LearnValue(oPaxosMsg.instanceid(), oBallot, oPaxosMsg.value(), GetLastChecksum());
        if (ret != 0)
        {
            PLGErr("LearnState.LearnValue fail, ret %d", ret);
            return;
        }
        
        PLGHead("END LearnValue OK, proposalid %lu proposalid_nodeid %lu valueLen %zu", 
                oPaxosMsg.proposalid(), oPaxosMsg.nodeid(), oPaxosMsg.value().size());
    }

    if (oPaxosMsg.flag() == PaxosMsgFlagType_SendLearnValue_NeedAck)
    {
        /*需要发送ACK*/
        
        //every time' when receive valid need ack learn value, reset noop timeout.
        Reset_AskforLearn_Noop();

        SendLearnValue_Ack(oPaxosMsg.nodeid());
    }
}
```

## 发送Ack给相应的Learner（发送学习到的决议给本Follower的那个Learner）

```
void Learner :: SendLearnValue_Ack(const nodeid_t iSendNodeID)
{
    PLGHead("START LastAck.Instanceid %lu Now.Instanceid %lu", m_llLastAckInstanceID, GetInstanceID());

    /*自从上一次Ack以来，经历的Instance尚未超过阈值，则无需Ack*/
    if (GetInstanceID() < m_llLastAckInstanceID + LearnerReceiver_ACK_LEAD)
    {
        PLGImp("No need to ack");
        return;
    }
    
    BP->GetLearnerBP()->SendLearnValue_Ack();

    /*更新Follower上最后一次Ack的InstanceID*/
    m_llLastAckInstanceID = GetInstanceID();

    /* 发送MsgType_PaxosLearner_SendLearnValue_Ack消息给Learner，
     * 作为MsgType_PaxosLearner_SendLearnValue的响应
     */
    PaxosMsg oPaxosMsg;
    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_SendLearnValue_Ack);
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());

    SendMessage(iSendNodeID, oPaxosMsg);

    PLGHead("End. ok");
}
```

