# 提纲
[toc]

# Base类基础
## Base类定义
Base类作为Proposer，Acceptor和Learner类的基类，主要提供消息相关的接口。
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


# Base类方法
## 广播消息

```
int Base :: BroadcastMessage(const PaxosMsg & oPaxosMsg, const int iRunType, const int iSendType)
{
    if (m_bIsTestMode)
    {
        return 0;
    }

    BP->GetInstanceBP()->BroadcastMessage();

    /* 参数@iRunType的默认值就是BroadcastMessage_Type_RunSelf_First，表示现在本节点上
     * 处理消息，如果本节点处理失败，就不再尝试发送给其它节点了
     */
    if (iRunType == BroadcastMessage_Type_RunSelf_First)
    {
        /*首先在本节点上处理该消息*/
        if (m_poInstance->OnReceivePaxosMsg(oPaxosMsg) != 0)
        {
            return -1;
        }
    }
    
    /*本地处理成功，再尝试将消息发送给其它节点*/
    string sBuffer;
    /*打包消息*/
    int ret = PackMsg(oPaxosMsg, sBuffer);
    if (ret != 0)
    {
        return ret;
    }
    
    /* 广播给其它节点
     * 
     * m_poMsgTransport的类型为MsgTransport，这是一个抽象类，在PhxPaxos中类Communicate
     * 会继承该抽象类并实现其方法，而在类Communicate中是对Network类的封装，也就是说消息
     * 的发送最终还是通过Network类实现的，而类Network也是一个抽象类，在PhxPaxos中实现了
     * 类DFNetWork，来实现类Network中的接口，所以消息的传输最终都走到DFNetWork上
     *
     * MsgTransport::BroadcastMessage -> Communicate::BroadcastMessage -> 
     * Communicate::Send -> DFNetWork::SendMessageTCP或者DFNetWork::SendMessageUDP
     */
    ret = m_poMsgTransport->BroadcastMessage(sBuffer, iSendType);

    /*如果设置了最后在本地节点上处理消息的话，则现在处理之*/
    if (iRunType == BroadcastMessage_Type_RunSelf_Final)
    {
        m_poInstance->OnReceivePaxosMsg(oPaxosMsg);
    }

    return ret;
}
```

## 发送消息到指定节点

```
int Base :: SendMessage(const nodeid_t iSendtoNodeID, const PaxosMsg & oPaxosMsg, const int iSendType)
{
    if (m_bIsTestMode)
    {
        return 0;
    }

    BP->GetInstanceBP()->SendMessage();

    /*如果目标节点就是本节点，则直接处理之*/
    if (iSendtoNodeID == m_poConfig->GetMyNodeID())
    {
        m_poInstance->OnReceivePaxosMsg(oPaxosMsg);
        return 0; 
    }
    
    /*序列化消息*/
    string sBuffer;
    int ret = PackMsg(oPaxosMsg, sBuffer);
    if (ret != 0)
    {
        return ret;
    }

    /* 参考Base :: BroadcastMessage中注释，这里的调用链是：
     * Base::SendMessage -> Communicate::SendMessage -> Communicate::Send ->
     * DFNetWork::SendMessageTCP或者DFNetWork::SendMessageUDP
     */
    return m_poMsgTransport->SendMessage(iSendtoNodeID, sBuffer, iSendType);
}
```

## 广播消息给Follower

```
int Base :: BroadcastMessageToFollower(const PaxosMsg & oPaxosMsg, const int iSendType)
{
    string sBuffer;
    /*序列化*/
    int ret = PackMsg(oPaxosMsg, sBuffer);
    if (ret != 0)
    {
        return ret;
    }

    /* 参考Base :: BroadcastMessage中注释，这里的调用链是：
     * Base::BroadcastMessageToFollower -> Communicate :: BroadcastMessageFollower ->
     * Communicate::Send -> DFNetWork::SendMessageTCP或者DFNetWork::SendMessageUDP
     */
    return m_poMsgTransport->BroadcastMessageFollower(sBuffer, iSendType);
}
```

## 广播消息给TempNode集合中的节点

```
int Base :: BroadcastMessageToTempNode(const PaxosMsg & oPaxosMsg, const int iSendType)
{
    string sBuffer;
    int ret = PackMsg(oPaxosMsg, sBuffer);
    if (ret != 0)
    {
        return ret;
    }
    
    /* 参考Base :: BroadcastMessage中注释，这里的调用链是：
     * Base::BroadcastMessageToTempNode -> Communicate :: BroadcastMessageToTempNode ->
     * Communicate::Send -> DFNetWork::SendMessageTCP或者DFNetWork::SendMessageUDP
     */
    return m_poMsgTransport->BroadcastMessageTempNode(sBuffer, iSendType);
}
```

## 序列化消息
```
int Base :: PackMsg(const PaxosMsg & oPaxosMsg, std::string & sBuffer)
{
    std::string sBodyBuffer;
    /*将PaxosMsg序列化为字符串*/
    bool bSucc = oPaxosMsg.SerializeToString(&sBodyBuffer);
    if (!bSucc)
    {
        PLGErr("PaxosMsg.SerializeToString fail, skip this msg");
        return -1;
    }

    /*设置消息类型为MsgCmd_PaxosMsg（以区别于MsgCmd_CheckpointMsg）*/
    int iCmd = MsgCmd_PaxosMsg;
    PackBaseMsg(sBodyBuffer, iCmd, sBuffer);

    return 0;
}

void Base :: PackBaseMsg(const std::string & sBodyBuffer, const int iCmd, std::string & sBuffer)
{
    /*在消息体中添加上header信息和checksum信息*/
    char sGroupIdx[GROUPIDXLEN] = {0};
    int iGroupIdx = m_poConfig->GetMyGroupIdx();
    memcpy(sGroupIdx, &iGroupIdx, sizeof(sGroupIdx));

    Header oHeader;
    /*设置gid*/
    oHeader.set_gid(m_poConfig->GetGid());
    oHeader.set_rid(0);
    oHeader.set_cmdid(iCmd);
    oHeader.set_version(1);

    std::string sHeaderBuffer;
    bool bSucc = oHeader.SerializeToString(&sHeaderBuffer);
    if (!bSucc)
    {
        PLGErr("Header.SerializeToString fail, skip this msg");
        assert(bSucc == true);
    }

    char sHeaderLen[HEADLEN_LEN] = {0};
    uint16_t iHeaderLen = (uint16_t)sHeaderBuffer.size();
    memcpy(sHeaderLen, &iHeaderLen, sizeof(sHeaderLen));

    sBuffer = string(sGroupIdx, sizeof(sGroupIdx)) + string(sHeaderLen, sizeof(sHeaderLen)) + sHeaderBuffer + sBodyBuffer;

    //check sum
    uint32_t iBufferChecksum = crc32(0, (const uint8_t *)sBuffer.data(), sBuffer.size(), NET_CRC32SKIP);
    char sBufferChecksum[CHECKSUM_LEN] = {0};
    memcpy(sBufferChecksum, &iBufferChecksum, sizeof(sBufferChecksum));

    sBuffer += string(sBufferChecksum, sizeof(sBufferChecksum));
}
```

## 反序列化消息

```
/*传入序列化后的消息，解析出header部分、body部分的起始位置和body部分的长度*/
int Base :: UnPackBaseMsg(const std::string & sBuffer, Header & oHeader, size_t & iBodyStartPos, size_t & iBodyLen)
{
    uint16_t iHeaderLen = 0;
    /*消息最开始的部分存放的是group index*/
    memcpy(&iHeaderLen, sBuffer.data() + GROUPIDXLEN, HEADLEN_LEN);

    /*header部分的起点*/
    size_t iHeaderStartPos = GROUPIDXLEN + HEADLEN_LEN;
    /*消息体部分的起点*/
    iBodyStartPos = iHeaderStartPos + iHeaderLen;

    if (iBodyStartPos > sBuffer.size())
    {
        BP->GetAlgorithmBaseBP()->UnPackHeaderLenTooLong();
        NLErr("Header headerlen too loog %d", iHeaderLen);
        return -1;
    }

    /*解析header部分*/
    bool bSucc = oHeader.ParseFromArray(sBuffer.data() + iHeaderStartPos, iHeaderLen);
    if (!bSucc)
    {
        NLErr("Header.ParseFromArray fail, skip this msg");
        return -1;
    }

    NLDebug("buffer_size %zu header len %d cmdid %d gid %lu rid %lu version %d body_startpos %zu", 
            sBuffer.size(), iHeaderLen, oHeader.cmdid(), oHeader.gid(), oHeader.rid(), oHeader.version(), iBodyStartPos);

    if (oHeader.version() >= 1)
    {
        if (iBodyStartPos + CHECKSUM_LEN > sBuffer.size())
        {
            NLErr("no checksum, body start pos %zu buffersize %zu", iBodyStartPos, sBuffer.size());
            return -1;
        }

        iBodyLen = sBuffer.size() - CHECKSUM_LEN - iBodyStartPos;

        /*解析出checksum部分*/
        uint32_t iBufferChecksum = 0;
        memcpy(&iBufferChecksum, sBuffer.data() + sBuffer.size() - CHECKSUM_LEN, CHECKSUM_LEN);
        
        /*重新计算checksum*/
        uint32_t iNewCalBufferChecksum = crc32(0, (const uint8_t *)sBuffer.data(), sBuffer.size() - CHECKSUM_LEN, NET_CRC32SKIP);
        /*检查checksum是否一致*/
        if (iNewCalBufferChecksum != iBufferChecksum)
        {
            BP->GetAlgorithmBaseBP()->UnPackChecksumNotSame();
            NLErr("Data.bring.checksum %u not equal to Data.cal.checksum %u",
                    iBufferChecksum, iNewCalBufferChecksum);
            return -1;
        }

        /*
        NLDebug("Checksum compare ok, Data.bring.checksum %u, Data.cal.checksum %u",
                iBufferChecksum, iNewCalBufferChecksum) 
        */
    }
    else
    {
        iBodyLen = sBuffer.size() - iBodyStartPos;
    }

    return 0;
}
```

