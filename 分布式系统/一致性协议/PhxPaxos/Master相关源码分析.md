# 提纲
[toc]

# Master相关原理
[Paxos理论介绍(1): 朴素Paxos算法理论推导与证明](https://zhuanlan.zhihu.com/p/21438357?refer=lynncui)

[Paxos理论介绍(2): Multi-Paxos与Leader](https://zhuanlan.zhihu.com/p/21466932?refer=lynncui)

[Paxos理论介绍(3): Master选举](https://zhuanlan.zhihu.com/p/21540239)

# Master管理相关类定义

```
/*MasterMgr在其独立的线程中运行*/
class MasterMgr : public Thread
{
private:
    /*本地节点*/
    Node * m_poPaxosNode;
    /*Master相关的状态机*/
    MasterStateMachine m_oDefaultMasterSM;

private:
    int m_iLeaseTime;

    bool m_bIsEnd;
    bool m_bIsStarted;

    /*MasterMgr所属的paxos group*/
    int m_iMyGroupIdx;

    bool m_bNeedDropMaster;
};

class MasterStateMachine : public InsideSM 
{
private:
    /*所在的paxos group*/
    int m_iMyGroupIdx;
    /*所在的节点ID*/
    nodeid_t m_iMyNodeID;

private:
    /*对LogStorage的封装，用于存放Master相关的信息*/
    MasterVariablesStore m_oMVStore;
    /*当前Master所在的节点ID和版本号*/
    nodeid_t m_iMasterNodeID;
    uint64_t m_llMasterVersion;
    /*租约时间*/
    int m_iLeaseTime;
    uint64_t m_llAbsExpireTime;

    std::mutex m_oMutex;
    /*Master改变的时候的回调*/
    MasterChangeCallback m_pMasterChangeCallback;
};

class MasterVariablesStore
{
public:
    MasterVariablesStore(const LogStorage * poLogStorage);
    ~MasterVariablesStore();

    int Write(const WriteOptions & oWriteOptions,  const int iGroupIdx, const MasterVariables & oVariables);

    int Read(const int iGroupIdx, MasterVariables & oVariables);

private:
    /*对LogStorage的封装，任何实现了LogStorage接口的类实例都可以赋值给m_poLogStorage*/
    LogStorage * m_poLogStorage;
};

class MasterVariables : public ::google::protobuf::Message {
    /*Master节点ID*/
    ::google::protobuf::uint64 masternodeid_;
    /*当前Master对应的版本号*/
    ::google::protobuf::uint64 version_;
    /*租约时间*/
    ::google::protobuf::uint32 leasetime_;
};
```
上面几个类的关系是这样的：在每个节点上的每一个paxos group都有一个MasterMgr，这个MasterMgr有一个状态机MasterStateMachine用于管理Master的状态变迁，MasterStateMachine中又需要借助于MasterVariablesStore来存储Master的状态，即MasterVariables，它包括当前Master所在的节点ID、当前Master的版本号和租约时间。

# Master管理相关构造函数

```
MasterMgr :: MasterMgr(
    const Node * poPaxosNode, 
    const int iGroupIdx, 
    const LogStorage * poLogStorage,
    MasterChangeCallback pMasterChangeCallback) 
    : m_oDefaultMasterSM(poLogStorage, poPaxosNode->GetMyNodeID(), iGroupIdx, pMasterChangeCallback) 
{
    m_iLeaseTime = 10000;

    m_poPaxosNode = (Node *)poPaxosNode;
    m_iMyGroupIdx = iGroupIdx;
    
    m_bIsEnd = false;
    m_bIsStarted = false;
    
    m_bNeedDropMaster = false;
}

MasterStateMachine :: MasterStateMachine(
    const LogStorage * poLogStorage, 
    const nodeid_t iMyNodeID, 
    const int iGroupIdx,
    MasterChangeCallback pMasterChangeCallback)
    : m_oMVStore(poLogStorage), m_pMasterChangeCallback(pMasterChangeCallback)
{
    m_iMyGroupIdx = iGroupIdx;
    m_iMyNodeID = iMyNodeID;

    m_iMasterNodeID = nullnode;
    m_llMasterVersion = (uint64_t)-1;
    m_iLeaseTime = 0;
    m_llAbsExpireTime = 0;

}

MasterVariablesStore :: MasterVariablesStore(const LogStorage * poLogStorage) : m_poLogStorage((LogStorage *)poLogStorage)
{
}
```

从PNode :: Init -> MasterMgr :: MasterMgr -> MasterStateMachine :: MasterStateMachine调用链来看，Master管理相关的状态机即MasterStateMachine和业务逻辑的状态机共用存储。

```
int PNode :: Init(const Options & oOptions, NetWork *& poNetWork)
{
    ......
    
    LogStorage * poLogStorage = nullptr;
    ret = InitLogStorage(oOptions, poLogStorage);
   
    ......

    //step3 build masterlist
    for (int iGroupIdx = 0; iGroupIdx < oOptions.iGroupCount; iGroupIdx++)
    {
        /*这里传递的是在InitLogStorage中返回的@poLogStorage*/
        MasterMgr * poMaster = new MasterMgr(this, iGroupIdx, poLogStorage, oOptions.pMasterChangeCallback);
        assert(poMaster != nullptr);
        m_vecMasterList.push_back(poMaster);

        ret = poMaster->Init();
        if (ret != 0)
        {
            return ret;
        }
    }
}
```

# Master初始化

```
int MasterMgr :: Init()
{
    return m_oDefaultMasterSM.Init();
}

int MasterStateMachine :: Init()
{
    MasterVariables oVariables;
    
    /*首先尝试从LogStorage中读取Master相关信息，存放在@oVariables中*/
    int ret = m_oMVStore.Read(m_iMyGroupIdx, oVariables);
    if (ret != 0 && ret != 1)
    {
        PLG1Err("Master variables read from store fail, ret %d", ret);
        return -1;
    }

    if (ret == 1)
    {
        PLG1Imp("no master variables exist");
    }
    else
    {
        m_llMasterVersion = oVariables.version();

        /* 如果MasterStateMachine中记录的master是我自己，则重置@m_iMasterNodeID
         * 和@m_llAbsExpireTime，因为有可能原来的master自己挂了，其它的节点又选出
         * 了新的master？
         */
        if (oVariables.masternodeid() == m_iMyNodeID)
        {
            m_iMasterNodeID = nullnode;
            m_llAbsExpireTime = 0;
        }
        else
        {
            /*如果在本节点挂掉之前的master是其它节点，则暂时选择继续相信它*/
            m_iMasterNodeID = oVariables.masternodeid();
            m_llAbsExpireTime = Time::GetSteadyClockMS() + oVariables.leasetime();
        }
    }
    
    PLG1Head("OK, master nodeid %lu version %lu expiretime %u", 
            m_iMasterNodeID, m_llMasterVersion, m_llAbsExpireTime);
    
    return 0;
}

int MasterVariablesStore :: Read(const int iGroupIdx, MasterVariables & oVariables)
{
    const int m_iMyGroupIdx = iGroupIdx;

    string sBuffer;
    /* 从PNode :: Init -> MasterMgr :: MasterMgr -> MasterStateMachine ::
     * MasterStateMachine调用链来看这里@m_poLogStorage中存放的是
     * MultiDatabase的实例，所以实际调用的是MultiDatabase :: GetMasterVariables
     */
    int ret = m_poLogStorage->GetMasterVariables(iGroupIdx, sBuffer);
    if (ret != 0 && ret != 1)
    {
        PLG1Err("DB.Get fail, groupidx %d ret %d", iGroupIdx, ret);
        return ret;
    }
    else if (ret == 1)
    {
        PLG1Imp("DB.Get not found, groupidx %d", iGroupIdx);
        return 1;
    }

    /*从@sBuffer中解析出@oVariables，ParseFromArray是Google protobuf中提供的函数*/
    bool bSucc = oVariables.ParseFromArray(sBuffer.data(), sBuffer.size());
    if (!bSucc)
    {
        PLG1Err("Variables.ParseFromArray fail, bufferlen %zu", sBuffer.size());
        return -1;
    }

    return 0;
}

int MultiDatabase :: GetMasterVariables(const int iGroupIdx, std::string & sBuffer)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }

    /*找到@iGroupIdx所指代的paxos group对应的Database，然后从该Database中读取*/
    return m_vecDBList[iGroupIdx]->GetMasterVariables(sBuffer);
}

int Database :: GetMasterVariables(std::string & sBuffer)
{
    /*Master特有的key： MASTERVARIABLES_KEY，取值为(uint64_t)-3*/
    static uint64_t llMasterVariablesKey = MASTERVARIABLES_KEY;
    /*从leveldb中读取*/
    return GetFromLevelDB(llMasterVariablesKey, sBuffer);
}

```

# 启动MasterMgr

```
void MasterMgr :: RunMaster()
{
    /*实际是启动它所在的线程，线程执行MasterMgr::run，参考“MasterMgr运行”*/
    start();
}
```

# MasterMgr运行

```
void MasterMgr :: run()
{
    m_bIsStarted = true;

    while(true)
    {
        /*只有在StopMaster的时候才会设置@m_bIsEnd*/
        if (m_bIsEnd)
        {
            return;
        }
        
        int iLeaseTime = m_iLeaseTime;

        uint64_t llBeginTime = Time::GetSteadyClockMS();
        
        /*提议自己成为Master*/
        TryBeMaster(iLeaseTime);

        int iContinueLeaseTimeout = (iLeaseTime - 100) / 4;
        iContinueLeaseTimeout = iContinueLeaseTimeout / 2 + OtherUtils::FastRand() % iContinueLeaseTimeout;

        if (m_bNeedDropMaster)
        {
            BP->GetMasterBP()->DropMaster();
            m_bNeedDropMaster = false;
            iContinueLeaseTimeout = iLeaseTime * 2;
            PLG1Imp("Need drop master, this round wait time %dms", iContinueLeaseTimeout);
        }
        
        uint64_t llEndTime = Time::GetSteadyClockMS();
        int iRunTime = llEndTime > llBeginTime ? llEndTime - llBeginTime : 0;
        int iNeedSleepTime = iContinueLeaseTimeout > iRunTime ? iContinueLeaseTimeout - iRunTime : 0;

        PLG1Imp("TryBeMaster, sleep time %dms", iNeedSleepTime);
        Time::MsSleep(iNeedSleepTime);
    }
}
```


# 提议自己成为Master
```
void MasterMgr :: TryBeMaster(const int iLeaseTime)
{
    nodeid_t iMasterNodeID = nullnode;
    uint64_t llMasterVersion = 0;

    //step 1 check exist master and get version
    /*从Master StateMachine中获取当前的Master所在的NodeID和version*/
    m_oDefaultMasterSM.SafeGetMaster(iMasterNodeID, llMasterVersion);

    /*其它节点是Master，直接返回*/
    if (iMasterNodeID != nullnode && (iMasterNodeID != m_poPaxosNode->GetMyNodeID()))
    {
        PLG1Imp("Ohter as master, can't try be master, masterid %lu myid %lu", 
                iMasterNodeID, m_poPaxosNode->GetMyNodeID());
        return;
    }

    BP->GetMasterBP()->TryBeMaster();

    /*至此，要么Mater节点为空，要么就是我自己*/
    //step 2 try be master
    std::string sPaxosValue;
    
    /*生成提议*/
    if (!MasterStateMachine::MakeOpValue(
                m_poPaxosNode->GetMyNodeID(),
                llMasterVersion,
                iLeaseTime,
                MasterOperatorType_Complete,
                sPaxosValue))
    {
        PLG1Err("Make paxos value fail");
        return;
    }

    const int iMasterLeaseTimeout = iLeaseTime - 100;
    
    uint64_t llAbsMasterTimeout = Time::GetSteadyClockMS() + iMasterLeaseTimeout; 
    uint64_t llCommitInstanceID = 0;

    SMCtx oCtx;
    oCtx.m_iSMID = MASTER_V_SMID;
    oCtx.m_pCtx = (void *)&llAbsMasterTimeout;

    /*提议*/
    int ret = m_poPaxosNode->Propose(m_iMyGroupIdx, sPaxosValue, llCommitInstanceID, &oCtx);
    if (ret != 0)
    {
        BP->GetMasterBP()->TryBeMasterProposeFail();
    }
}
```

# MasterStateMachine
## MasterStateMachine定义

```
class MasterStateMachine : public InsideSM 
{
private:
    /*所属的paxos group*/
    int m_iMyGroupIdx;
    /*所属的Node ID*/
    nodeid_t m_iMyNodeID;

private:
    /*存储Master相关信息*/
    MasterVariablesStore m_oMVStore;
    
    /*Master信息本地缓存*/
    nodeid_t m_iMasterNodeID;
    uint64_t m_llMasterVersion;
    int m_iLeaseTime;
    uint64_t m_llAbsExpireTime;

    std::mutex m_oMutex;

    /*Master变化相关的回调函数*/
    MasterChangeCallback m_pMasterChangeCallback;
};
```

## MasterStateMachine构造函数

```
MasterStateMachine :: MasterStateMachine(
    const LogStorage * poLogStorage, 
    const nodeid_t iMyNodeID, 
    const int iGroupIdx,
    MasterChangeCallback pMasterChangeCallback)
    : m_oMVStore(poLogStorage), m_pMasterChangeCallback(pMasterChangeCallback)
{
    m_iMyGroupIdx = iGroupIdx;
    m_iMyNodeID = iMyNodeID;

    m_iMasterNodeID = nullnode;
    m_llMasterVersion = (uint64_t)-1;
    m_iLeaseTime = 0;
    m_llAbsExpireTime = 0;

}
```

## 生成提议

```
bool MasterStateMachine :: MakeOpValue(
        const nodeid_t iNodeID,
        const uint64_t llVersion,
        const int iTimeout,
        const MasterOperatorType iOp,
        std::string & sPaxosValue)
{
    MasterOperator oMasterOper;
    oMasterOper.set_nodeid(iNodeID);
    oMasterOper.set_version(llVersion);
    oMasterOper.set_timeout(iTimeout);
    oMasterOper.set_operator_(iOp);
    oMasterOper.set_sid(OtherUtils::FastRand());

    return oMasterOper.SerializeToString(&sPaxosValue);
}
```




