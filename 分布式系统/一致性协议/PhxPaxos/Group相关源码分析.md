# 提纲
[toc]

# Group类定义

```
class Group
{
private:
    /*通信相关，对Network类的封装*/
    Communicate m_oCommunicate;
    /*配置相关*/
    Config m_oConfig;
    /*实例*/
    Instance m_oInstance;

    int m_iInitRet;
    std::thread * m_poThread;
};
```

# Group类构造函数

```
Group :: Group(LogStorage * poLogStorage, 
            NetWork * poNetWork,    
            InsideSM * poMasterSM,
            const int iGroupIdx,
            const Options & oOptions) : 
    m_oCommunicate(&m_oConfig, oOptions.oMyNode.GetNodeID(), oOptions.iUDPMaxSize, poNetWork),
    m_oConfig(poLogStorage, oOptions.bSync, oOptions.iSyncInterval, oOptions.bUseMembership, 
            oOptions.oMyNode, oOptions.vecNodeInfoList, oOptions.vecFollowerNodeInfoList, 
            iGroupIdx, oOptions.iGroupCount, oOptions.pMembershipChangeCallback),
    m_oInstance(&m_oConfig, poLogStorage, &m_oCommunicate, oOptions),
    m_iInitRet(-1), m_poThread(nullptr)
{
    m_oConfig.SetMasterSM(poMasterSM);
}
```

# Group初始化

```
void Group :: StartInit()
{
    /*创建线程，线程主函数为Group::Init*/
    m_poThread = new std::thread(&Group::Init, this);
    assert(m_poThread != nullptr);
}

void Group :: Init()
{
    /*初始化Config*/
    m_iInitRet = m_oConfig.Init();
    if (m_iInitRet != 0)
    {
        return;
    }

    /*将成员组管理状态机添加到Group中*/
    AddStateMachine(m_oConfig.GetSystemVSM());
    /*将Master管理状态机添加到Group中*/
    AddStateMachine(m_oConfig.GetMasterSM());
    
    /* 业务逻辑相关的状态机已经在PNode::Init -> PNode :: InitStateMachine调用链中
     * 添加到相应的paxos group中了
     */
    
    /*初始化Instance，见"Instance相关源码分析"*/
    m_iInitRet = m_oInstance.Init();
}
```

# Group启动

```
void Group :: Start()
{
    m_oInstance.Start();
}
```

