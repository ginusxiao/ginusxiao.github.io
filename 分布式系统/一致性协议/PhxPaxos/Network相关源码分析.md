**==Network相关代码比较简单，暂时不分析...==**
# 提纲
[toc]


# Network初始化
## 调用链
```
Node :: RunNode(const Options & oOptions, Node *& poNode)
    - PNode :: Init(const Options & oOptions, NetWork *& poNetWork)
        - PNode :: InitNetWork(const Options & oOptions, NetWork *& poNetWork)
            - DFNetWork :: Init(const std::string & sListenIp, const int iListenPort) 
```

## 网络初始化

```
int DFNetWork :: Init(const std::string & sListenIp, const int iListenPort) 
{
    /*UDP发送相关初始化*/
    int ret = m_oUDPSend.Init();
    if (ret != 0)
    {
        return ret;
    }

    /*UDP接收相关初始化*/
    ret = m_oUDPRecv.Init(iListenPort);
    if (ret != 0)
    {
        return ret;
    }

    /*TCP相关初始化（本文重点关注TCP初始化）*/
    ret = m_oTcpIOThread.Init(sListenIp, iListenPort);
    if (ret != 0)
    {
        PLErr("m_oTcpIOThread Init fail, ret %d", ret);
        return ret;
    }

    return 0;
}
```

## TCP网络初始化

```
int TcpIOThread :: Init(const std::string & sListenIp, const int iListenPort)
{
    /*分别初始化TCP read和TCP write*/
    int ret = m_oTcpRead.Init(sListenIp, iListenPort);
    if (ret == 0)
    {
        return m_oTcpWrite.Init();
    }

    return ret;
}
```

## 处理从网络接收到的消息

```
int NetWork :: OnReceiveMessage(const char * pcMessage, const int iMessageLen)
{
    if (m_poNode != nullptr)
    {
        /* 调用链：PNode :: OnReceiveMessage -> Instance :: OnReceiveMessage -> 
         * IOLoop :: AddMessage，首先从消息中解析出paxos group索引，然后交给该
         * paxos group中的Instance来处理
         */
        m_poNode->OnReceiveMessage(pcMessage, iMessageLen);
    }
    else
    {
        PLHead("receive msglen %d", iMessageLen);
    }

    return 0;
}
```

