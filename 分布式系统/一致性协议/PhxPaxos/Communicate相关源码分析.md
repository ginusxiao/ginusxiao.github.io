# 提纲
[toc]

# Communicate类基础
## Communicate相关类定义

```
class Communicate : public MsgTransport
{
private:
    Config * m_poConfig;
    NetWork * m_poNetwork;

    nodeid_t m_iMyNodeID;
    size_t m_iUDPMaxSize; 
};
```
Communicate类是对Network类的封装，它继承MsgTransport类并实现了MsgTransport中的接口。

## Communicate类构造函数

```
Communicate :: Communicate(
        const Config * poConfig,
        const nodeid_t iMyNodeID, 
        const int iUDPMaxSize,
        NetWork * poNetwork)
    : m_poConfig((Config *)poConfig), m_poNetwork(poNetwork),
    m_iMyNodeID(iMyNodeID), m_iUDPMaxSize(iUDPMaxSize)
{
}
```

# Communicate类方法
## 广播消息

```
int Communicate :: BroadcastMessage(const std::string & sMessage,
const int iSendType = Message_SendType_UDP)
{
    /*从成员组中获取所有节点信息*/
    const std::set<nodeid_t> & setNodeInfo = m_poConfig->GetSystemVSM()->GetMembershipMap();
    
    /*向成员组中每一个节点发送消息*/
    for (auto & it : setNodeInfo)
    {
        if (it != m_iMyNodeID)
        {
            /*发送消息*/
            Send(it, NodeInfo(it), sMessage, iSendType);
        }
    }

    return 0;
}
```

## 广播消息给Follower

```
int Communicate :: BroadcastMessageFollower(const std::string & sMessage, const int iSendType)
{
    const std::map<nodeid_t, uint64_t> & mapFollowerNodeInfo = m_poConfig->GetMyFollowerMap(); 
    
    /*依次向每一个Follower发送消息*/
    for (auto & it : mapFollowerNodeInfo)
    {
        if (it.first != m_iMyNodeID)
        {
            Send(it.first, NodeInfo(it.first), sMessage, iSendType);
        }
    }
    
    PLGDebug("%zu node", mapFollowerNodeInfo.size());

    return 0;
}
```

## 广播消息给TempNode集合中的节点

```
int Communicate :: BroadcastMessageTempNode(const std::string & sMessage, const int iSendType)
{
    const std::map<nodeid_t, uint64_t> & mapTempNode = m_poConfig->GetTmpNodeMap(); 
    
    for (auto & it : mapTempNode)
    {
        if (it.first != m_iMyNodeID)
        {
            Send(it.first, NodeInfo(it.first), sMessage, iSendType);
        }
    }
    
    PLGDebug("%zu node", mapTempNode.size());

    return 0;
}
```



## 发送消息到指定节点

```
int Communicate :: Send(const nodeid_t iNodeID, const NodeInfo & oNodeInfo, 
        const std::string & sMessage, const int iSendType)
{
    /*消息过大*/
    if ((int)sMessage.size() > MAX_VALUE_SIZE)
    {
        BP->GetNetworkBP()->SendRejectByTooLargeSize();
        PLGErr("Message size too large %zu, max size %u, skip message", 
                sMessage.size(), MAX_VALUE_SIZE);
        return 0;
    }

    BP->GetNetworkBP()->Send(sMessage);
    
    if (sMessage.size() > m_iUDPMaxSize || iSendType == Message_SendType_TCP)
    {
        /*采用TCP协议发送，内置的，最终会到达DFNetWork :: SendMessageTCP*/
        BP->GetNetworkBP()->SendTcp(sMessage);
        return m_poNetwork->SendMessageTCP(oNodeInfo.GetIP(), oNodeInfo.GetPort(), sMessage);
    }
    else
    {
        /*采用UDP协议发送，内置的，最终会到达DFNetWork :: SendMessageUDP*/
        BP->GetNetworkBP()->SendUdp(sMessage);
        return m_poNetwork->SendMessageUDP(oNodeInfo.GetIP(), oNodeInfo.GetPort(), sMessage);
    }
}
```




