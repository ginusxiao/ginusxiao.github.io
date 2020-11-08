# 提纲
[toc]

# node相关的接口

```
int Node :: RunNode(const Options & oOptions, Node *& poNode)
{
    if (oOptions.bIsLargeValueMode)
    {
        InsideOptions::Instance()->SetAsLargeBufferMode();
    }
    
    InsideOptions::Instance()->SetGroupCount(oOptions.iGroupCount);
        
    poNode = nullptr;
    NetWork * poNetWork = nullptr;

    Breakpoint::m_poBreakpoint = nullptr;
    BP->SetInstance(oOptions.poBreakpoint);

    /* Node类中主要提供PhxPaxos的节点相关的接口抽象，PNode则是PhxPaxos中的Node类
     * 的实现，这里创建PNode并初始化之
     */
    PNode * poRealNode = new PNode();
    int ret = poRealNode->Init(oOptions, poNetWork);
    if (ret != 0)
    {
        delete poRealNode;
        return ret;
    }

    //step1 set node to network
    //very important, let network on recieve callback can work.
    poNetWork->m_poNode = poRealNode;

    //step2 run network.
    //start recieve message from network, so all must init before this step.
    //must be the last step.
    poNetWork->RunNetWork();


    poNode = poRealNode;

    return 0;
}
```

# 初始化Node
```
int PNode :: Init(const Options & oOptions, NetWork *& poNetWork)
{
    /*检查@oOptions中的设置是否符合预期*/
    int ret = CheckOptions(oOptions);
    if (ret != 0)
    {
        PLErr("CheckOptions fail, ret %d", ret);
        return ret;
    }

    /*设置PNode对应的NODE ID*/
    m_iMyNodeID = oOptions.oMyNode.GetNodeID();

    /*初始化LogStorage*/
    LogStorage * poLogStorage = nullptr;
    ret = InitLogStorage(oOptions, poLogStorage);
    if (ret != 0)
    {
        return ret;
    }

    /*初始化网络*/
    ret = InitNetWork(oOptions, poNetWork);
    if (ret != 0)
    {
        return ret;
    }

    /*为每个paxos group初始化MasterMgr*/
    for (int iGroupIdx = 0; iGroupIdx < oOptions.iGroupCount; iGroupIdx++)
    {
        /*为 @iGroupIdx所指代的paxos group创建MasterMgr*/
        MasterMgr * poMaster = new MasterMgr(this, iGroupIdx, poLogStorage, oOptions.pMasterChangeCallback);
        assert(poMaster != nullptr);
        /*添加到@m_vecMasterList数组中统一管理*/
        m_vecMasterList.push_back(poMaster);
        /*初始化当前MasterMgr*/
        ret = poMaster->Init();
        if (ret != 0)
        {
            return ret;
        }
    }

    /*为每个paxos group创建Group实例*/
    for (int iGroupIdx = 0; iGroupIdx < oOptions.iGroupCount; iGroupIdx++)
    {
        /* 在Group的构造函数中用到了每个paxos group自己的MasterStateMachine，即
         * m_vecMasterList[iGroupIdx]->GetMasterSM()
         */
        Group * poGroup = new Group(poLogStorage, poNetWork, m_vecMasterList[iGroupIdx]->GetMasterSM(), iGroupIdx, oOptions);
        assert(poGroup != nullptr);
        /*添加到@m_vecGroupList数组中统一管理*/
        m_vecGroupList.push_back(poGroup);
    }

    /*如果使用批量提交的话，则为每一个paxos group创建一个批量提交控制结构ProposeBatch*/
    if (oOptions.bUseBatchPropose)
    {
        for (int iGroupIdx = 0; iGroupIdx < oOptions.iGroupCount; iGroupIdx++)
        {
            ProposeBatch * poProposeBatch = new ProposeBatch(iGroupIdx, this, &m_oNotifierPool);
            assert(poProposeBatch != nullptr);
            /*添加到@m_vecProposeBatch数组中统一管理*/
            m_vecProposeBatch.push_back(poProposeBatch);
        }
    }

    /*初始化状态机*/
    InitStateMachine(oOptions);    

    /*并发的初始化各个paxos group，每个paxos group都会在自己的线程中初始化*/
    for (auto & poGroup : m_vecGroupList)
    {
        /*见Group :: StartInit()*/
        poGroup->StartInit();
    }

    /*结束每个paxos group的初始化线程并返回初始化结果*/
    for (auto & poGroup : m_vecGroupList)
    {
        int initret = poGroup->GetInitRet();
        if (initret != 0)
        {
            ret = initret;
        }
    }

    /*如果有任何一个paxos group初始化失败，则退出*/
    if (ret != 0)
    {
        return ret;
    }

    //last step. must init ok, then should start threads.
    //because that stop threads is slower, if init fail, we need much time to stop many threads.
    //so we put start threads in the last step.
    /*启动各个paxos group的实例*/
    for (auto & poGroup : m_vecGroupList)
    {
        //start group's thread first.
        poGroup->Start();
    }
    
    /*启动Mater管理*/
    RunMaster(oOptions);
    RunProposeBatch();

    PLHead("OK");

    return 0;
}

int PNode :: InitLogStorage(const Options & oOptions, LogStorage *& poLogStorage)
{
    /*如果用户设置了自己的log storage，则使用用户设置的*/
    if (oOptions.poLogStorage != nullptr)
    {
        poLogStorage = oOptions.poLogStorage;
        PLImp("OK, use user logstorage");
        return 0;
    }

    /*用户没有设置自己的log storage，但是也没有设置log storage path，就报错*/
    if (oOptions.sLogStoragePath.size() == 0)
    {
        PLErr("LogStorage Path is null");
        return -2;
    }

    /* 用户没有设置自己的log storage，但是设置了log storage path，则采用该log storage
     * path初始化默认的log storage @m_oDefaultLogStorage，其中@m_oDefaultLogStorage
     * 的类型为MultiDatabase。
     */
    int ret = m_oDefaultLogStorage.Init(oOptions.sLogStoragePath, oOptions.iGroupCount);
    if (ret != 0)
    {
        PLErr("Init default logstorage fail, logpath %s ret %d",
                oOptions.sLogStoragePath.c_str(), ret);
        return ret;
    }

    /*返回@m_oDefaultLogStorage给调用者*/
    poLogStorage = &m_oDefaultLogStorage;
    
    PLImp("OK, use default logstorage");

    return 0;
}

int PNode :: InitNetWork(const Options & oOptions, NetWork *& poNetWork)
{
    /*如果用户自定义了网络，则直接使用之*/
    if (oOptions.poNetWork != nullptr)
    {
        poNetWork = oOptions.poNetWork;
        PLImp("OK, use user network");
        return 0;
    }

    /*初始化本节点网络*/
    int ret = m_oDefaultNetWork.Init(oOptions.oMyNode.GetIP(), oOptions.oMyNode.GetPort());
    if (ret != 0)
    {
        PLErr("init default network fail, listenip %s listenport %d ret %d",
                oOptions.oMyNode.GetIP().c_str(), oOptions.oMyNode.GetPort(), ret);
        return ret;
    }

    /*返回给上层调用*/
    poNetWork = &m_oDefaultNetWork;
    
    PLImp("OK, use default network");

    return 0;
}

void PNode :: InitStateMachine(const Options & oOptions)
{
    /* 遍历每一个paxos group对应的GroupSMInfo，其中oOptions.vecGroupSMInfoList中
     * 存放着每一个paxos group对应的GroupSMInfo
     */
    for (auto & oGroupSMInfo : oOptions.vecGroupSMInfoList)
    {
        /* 遍历当前paxos group的GroupSMInfo中的每一个StateMachine，其中
         * oGroupSMInfo.vecSMList中存放着当前paxos group中的所有StateMachine，
         * 每一个paxos group可以拥有多个StateMachine
         */
        for (auto & poSM : oGroupSMInfo.vecSMList)
        {
            /* 前面的两层for循环中遍历的StateMachine都保存在@oOptions中，需要将之
             * 真正关联到每一个Paxos group
             */
            AddStateMachine(oGroupSMInfo.iGroupIdx, poSM);
        }
    }
}

void PNode :: AddStateMachine(const int iGroupIdx, StateMachine * poSM)
{
    if (!CheckGroupID(iGroupIdx))
    {
        return;
    }
    
    /*添加到对应的paxos group中，实际上最终添加到该paxos group对应的Instance中*/
    m_vecGroupList[iGroupIdx]->AddStateMachine(poSM);
}

void Group :: AddStateMachine(StateMachine * poSM)
{
    m_oInstance.AddStateMachine(poSM);
}

void Instance :: AddStateMachine(StateMachine * poSM)
{
    /*@m_oSMFac类型为SMFac，实际上就是简单的对StateMachine数组的封装而已*/
    m_oSMFac.AddSM(poSM);
}

void SMFac :: AddSM(StateMachine * poSM)
{
    /*首先查找是否存在，如果已存在，则直接返回，否则加入*/
    for (auto & poSMt : m_vecSMList)
    {
        if (poSMt->SMID() == poSM->SMID())
        {
            return;
        }
    }

    /*加入到数组中*/
    m_vecSMList.push_back(poSM);
}

void PNode :: RunMaster(const Options & oOptions)
{
    /*对于每一个paxos group，检查是否需要运行Master，如果需要则运行之*/
    for (auto & oGroupSMInfo : oOptions.vecGroupSMInfoList)
    {
        //check if need to run master.
        if (oGroupSMInfo.bIsUseMaster)
        {
            if (!m_vecGroupList[oGroupSMInfo.iGroupIdx]->GetConfig()->IsIMFollower())
            {
                m_vecMasterList[oGroupSMInfo.iGroupIdx]->RunMaster();
            }
            else
            {
                PLImp("I'm follower, not run master damon.");
            }
        }
    }
}

void PNode :: RunProposeBatch()
{
    for (auto & poProposeBatch : m_vecProposeBatch)
    {
        poProposeBatch->Start();
    }
}
```

# Propose

```
int PNode :: Propose(const int iGroupIdx, const std::string & sValue, uint64_t & llInstanceID, SMCtx * poSMCtx)
{
    if (!CheckGroupID(iGroupIdx))
    {
        return Paxos_GroupIdxWrong;
    }

    /*提交给Committer*/
    return m_vecGroupList[iGroupIdx]->GetCommitter()->NewValueGetID(sValue, llInstanceID, poSMCtx);
}
```


