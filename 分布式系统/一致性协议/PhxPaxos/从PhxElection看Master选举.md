# 提纲
[toc]

# 使用PhxPaxos的Master，给你的Server提供一个选举功能

这里先解释一下Master的定义，Master是指在多台机器构建的集合里面，任一时刻，只有一台机器认为自己是Master或者没有任何机器认为自己是Master。

这个功能非常实用。假设有那么一个多台机器组成的集群，我希望任一时刻只有一台机器在提供服务，相信大家可能会遇到这样的场景， 而通常的做法可能是使用ZooKeeper来搭建分布式锁。那么使用我们的Master功能，只需编写短短的几十行代码， 即可跟你现有的服务无缝结合起来，而不用引入额外的一些庞大的模块。

下面展示如何嵌入Master到自己的代码里面。

首先我们构建一个选举类PhxElection，这个类供已有的模块代码使用，如下：
```
class PhxElection
{
public:
    PhxElection(const phxpaxos::NodeInfo & oMyNode, const phxpaxos::NodeInfoList & vecNodeList);
    ~PhxElection();

    int RunPaxos();
    const phxpaxos::NodeInfo GetMaster();
    const bool IsIMMaster();

private:
    phxpaxos::NodeInfo m_oMyNode;
    phxpaxos::NodeInfoList m_vecNodeList;
    phxpaxos::Node * m_poPaxosNode;
};
```
这个类提供两个功能函数，GetMaster获得当前集群的Master，IsIMMaster判断自己是否当前Master。

RunPaxos是运行PhxPaxos的函数，代码如下：
```
int PhxElection :: RunPaxos()
{
    Options oOptions;

    /*创建PhxPaxos相关的目录*/
    int ret = MakeLogStoragePath(oOptions.sLogStoragePath);
    if (ret != 0)
    {   
        return ret;
    }   

    /*通过PhxElection中的域成员（在main函数中从命令行中解析到的）来设置@oOptions*/
    oOptions.iGroupCount = 1;
    /*本节点IP/PORT信息*/
    oOptions.oMyNode = m_oMyNode;
    /*所有成员节点IP/PORT信息*/
    oOptions.vecNodeInfoList = m_vecNodeList;

    //open inside master state machine
    GroupSMInfo oSMInfo;
    oSMInfo.iGroupIdx = 0;
    /*启用内置的Master状态机*/
    oSMInfo.bIsUseMaster = true;

    oOptions.vecGroupSMInfoList.push_back(oSMInfo);

    /* 在Node::RunNode -> PNode::Init -> PNode::RunMaster -> MasterMgr::RunMaster
     * -> MasterMgr::run调用链中，每一个paxos group中都会运行一个MasterMgr，并执行
     * MasterMgr::run
     */
    ret = Node::RunNode(oOptions, m_poPaxosNode);
    if (ret != 0) 
    {   
        printf("run paxos fail, ret %d\n", ret);
        return ret;
    }   

    //you can change master lease in real-time.
    m_poPaxosNode->SetMasterLease(0, 3000);

    printf("run paxos ok\n");
    return 0;
}
```
与Echo不一样的是，这次我们并不需要实现自己的状态机，而是通过将oSMInfo.bIsUseMaster设置为true，开启我们内置的一个Master状态机。 相同的，通过Node::RunNode即可获得PhxPaxos的实例指针。通过SetMasterLease可以随时修改Master的租约时间。 最后，我们通过这个指针获得集群的Master信息，代码如下：
```
const phxpaxos::NodeInfo PhxElection :: GetMaster()
{
    //only one group, so groupidx is 0.
    return m_poPaxosNode->GetMaster(0);
}

const bool PhxElection :: IsIMMaster()
{
    return m_poPaxosNode->IsIMMaster(0);
}
```
通过这个简单的选举类，每台机器都可以获知当前的Master信息。

# MasterMgr初始化

```
MasterMgr :: MasterMgr(
    const Node * poPaxosNode, 
    const int iGroupIdx, 
    const LogStorage * poLogStorage,
    MasterChangeCallback pMasterChangeCallback) 
    : m_oDefaultMasterSM(poLogStorage, poPaxosNode->GetMyNodeID(), iGroupIdx, pMasterChangeCallback) 
{
    /*默认lease time是10ms*/
    m_iLeaseTime = 10000;

    /*设置MasterMgr所在的Node实例和paxos group*/
    m_poPaxosNode = (Node *)poPaxosNode;
    m_iMyGroupIdx = iGroupIdx;
    
    /*状态信息，尚未启动，也尚未停止*/
    m_bIsEnd = false;
    m_bIsStarted = false;
    /*是否需要放弃自己作为Master*/
    m_bNeedDropMaster = false;
}

int MasterMgr :: Init()
{
    /*初始化默认的MasterStateMachine*/
    return m_oDefaultMasterSM.Init();
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

int MasterStateMachine :: Init()
{
    /*首先尝试从MasterVariablesStore中加载MasterVariables*/
    MasterVariables oVariables;
    int ret = m_oMVStore.Read(m_iMyGroupIdx, oVariables);
    if (ret != 0 && ret != 1)
    {
        /*失败*/
        PLG1Err("Master variables read from store fail, ret %d", ret);
        return -1;
    }

    if (ret == 1)
    {
        /* 不存在与MasterVariablesStore中，如果是第一次在该节点上运行PhxPaxos，
         * 则肯定是不存在的
         */
        PLG1Imp("no master variables exist");
    }
    else
    {
        /*成功从MasterVariablesStore中读取到Master信息*/
        
        /*设置当前Master的版本号*/
        m_llMasterVersion = oVariables.version();

        if (oVariables.masternodeid() == m_iMyNodeID)
        {
            /* 如果之前的Master是我自己，则现在的Master可能已经发生了改变，
             * 先设置为nullnode
             */
            m_iMasterNodeID = nullnode;
            m_llAbsExpireTime = 0;
        }
        else
        {
            /*如果是其它节点，则暂且认为它仍然是Master*/
            m_iMasterNodeID = oVariables.masternodeid();
            m_llAbsExpireTime = Time::GetSteadyClockMS() + oVariables.leasetime();
        }
    }
    
    PLG1Head("OK, master nodeid %lu version %lu expiretime %u", 
            m_iMasterNodeID, m_llMasterVersion, m_llAbsExpireTime);
    
    return 0;
}
```
初始化阶段尝试从MasterVariablesStore中加载信息到MasterStateMachine，设置MasterStateMachine中的Master节点ID、Master版本号和Master信息过期时间：
如果MasterVariablesStore中不存在Master相关信息，则设置当前的Master节点ID为nullnode，Master版本号为-1，Master信息过期时间为0；
如果MasterVariablesStore中存在Master相关信息，且记录的Master是我自己，则设置当前Master节点ID为nullnode（因为可能其它节点已经选举出来了新的Master），Master版本号为MasterVariablesStore中记录的版本号，Master信息过期时间为0；
如果MasterVariablesStore中存在Master相关信息，且记录的Master是其它节点，则暂且认为它仍然是Master，设置Master节点ID为该Master所在的节点，Master版本号为MasterVariablesStore中记录的版本号，Master信息过期时间为从现在起经过lease时间；


# Master选举
## 不考虑Drop Master的情况下MasterMgr::run的每一轮运行过程

1. 如果TryBeMaster不用做任何事情，即既不用续租也不用提议自己作为Master，那么当前在本节点看来在其它节点上存在有效的Master，TryBeMaster很快就会返回，只需要等待Master节点的续租请求（即Master所在的节点再次发送TryBeMaster提议）即可，等待时间由iContinueLeaseTimeout、iRunTime来控制，其中iContinueLeaseTimeout介于[(iLeaseTime - 100)/8, (3*(iLeaseTime - 100)/8)]，iRunTime表示TryBeMaster调用执行的时间，如果iContinueLeaseTimeout大于iRunTime，则等待iContinueLeaseTimeout - iRunTime的时间，该等待时间至多是无限接近于(3*(iLeaseTime - 100)/8) ，但绝不会超过(3*(iLeaseTime - 100)/8)，然后再次执行TryBeMaster，如果iContinueLeaseTimeout小于iRunTime，则直接执行下一次TryBeMaster，而无需等待。

    - 如果在等待过程中，Master节点成功续租：
    
        - 如果本节点成功学习到当前最新的Master信息，则在等待结束再次进入TryBeMaster
        逻辑的时候检查到仍然无需做任何事情，因为等待的时间不超过(3*(iLeaseTime - 100)/8)，而它成功学习到的当前最新的Master的任期时间是从它成功学习到该Master的
        时间加上iLeaseTime的时间；
        
        - 如果本节点未成功学习到当前最新的Master信息，则在等待结束再次进入TryBeMaster
        逻辑的时候：
        
            A. 本节点可能会检查到当前它所看到的Master已经过期了，那么就会提议自己作为
            Master，但是由于在提议自己作为Master的过程中会携带版本号（关于版本号，请参
            考“为什么Master选举过程中需要携带Version信息”），就不会接受它的提议；
            
            B. 本节点也可能会检查到它所看到的Master仍未过期，那么就继续进入等待，但是它
            所看到的Master终会过期，然后本节点会尝试提议自己作为Master，但是同样在提议
            中携带有版本号，集群不会通过它的提议；
            
    - 如果在等待过程中，Master节点没有续租，本节点在等待结束后会再次进入TryBeMaster逻辑：
    
        - 本节点可能会检查到当前它所看到的Master已经过期了，当前Master变为无效，那么就
        会提议自己作为Master，
            
        - 本节点也可能会检查到它所看到的Master仍未过期，那么就继续进入等待；
        
    - 如果在等待过程中，Master节点成功续租：
    
        - 如果本节点成功学习到当前最新的Master信息，则在等待结束再次进入TryBeMaster
        逻辑的时候检查到仍然无需做任何事情，因为等待的时间不超过(3*(iLeaseTime - 100)/8)，而它成功学习到的当前最新的Master的任期时间是从它成功学习到该Master的
        时间加上iLeaseTime的时间；
        
        - 如果本节点未成功学习到当前最新的Master信息，则在等待结束再次进入TryBeMaster
        逻辑的时候：
        
            A. 本节点可能会检查到当前它所看到的Master已经过期了，那么就会提议自己作为
            Master，但是由于在提议自己作为Master的过程中会携带版本号（关于版本号，请参
            考“为什么Master选举过程中需要携带Version信息”），就不会接受它的提议；
            
            B. 本节点也可能会检查到它所看到的Master仍未过期，那么就继续进入等待，但是它
            所看到的Master终会过期，然后本节点会尝试提议自己作为Master，但是同样在提议
            中携带有版本号，集群不会通过它的提议；
            
    - 如果在等待过程中，Master节点没有续租，本节点在等待结束后会再次进入TryBeMaster逻辑：
    
        - 本节点可能会检查到当前它所看到的Master已经过期了，当前Master变为无效，那么就
        会提议自己作为Master，
            
        - 本节点也可能会检查到它所看到的Master仍未过期，那么就继续进入等待；
    
2. 如果TryBeMaster需要续租，那么它自己就是Master节点，它会提出续租请求（作为一个提议）。

    - 在Master节点自身看来，任期尚未结束，那么其它节点一定仍然信任该Master节点，因为其它节点上关于该Master节点的任期一定也未过期（被选作Master的节点自己看到的任期是从它调用TryBeMaster的那一时刻开始计时的iLeaseTime时间段内，而其它节点上信任的Master的任期是从它学习到该Master节点开始的那一时刻开始计时的iLeaseTime时间段之内，而Master上调用TryBeMaster的时间点一定早于其它节点上学习到该Master节点的时间点），除Master外的其它节点都不会提议自己作为Master，只有Master节点自身提议自己续
    租Master任期，很容易获得多数派；
        
    - 在成功续租之后，Master节点上会更新它的新一轮的任期，成功学习到该Master的非Master的节点上也会更新它们所看到的Master的任期；
        
    - 成功续租之后，Master节点等待iContinueLeaseTimeout - iRunTime的时间，该等待时间至多是无限接近于(3*(iLeaseTime - 100)/8) ，但绝不会超过(3*(iLeaseTime - 100)/8)，如果iContinueLeaseTimeout小于iRunTime，则无需等待，然后再次执行TryBeMaster，即再次续租；

3. 如果当前的Master不存在或者过期，则本节点会尝试提议自己作为Master。

    - 如果提议被通过，本节点成为Master，则本节点更新它的任期为从调用TryBeMaster开始的iLeaseTime时间段，其它成功学习到该Master信息的节点也更新它的任期为从成功学习到该Master的时间点开始的iLeaseTine时间段，然后本节点等待iContinueLeaseTimeout - iRunTime的时间，该等待时间至多是无限接近于(3*(iLeaseTime - 100)/8) ，但绝不会超过(3*(iLeaseTime - 100)/8)，如果iContinueLeaseTimeout小于iRunTime，则无需等待，然后再次执行TryBeMaster，即进行续租；
    
    - 如果提议未被通过，那么可能存在以下情况：

        - PaxosTryCommitRet_TooManyThreadWaiting_Reject：
        太多的提议处于排队状态，暂时拒绝该提议，则等待一段时间之后，如果在等待期间成功学习到新的Master，则在等待时间结束再次进入TryBeMaster的时候无需做任何工作；如果在等待期间未学习到新的Master，则在等待时间结束之后再次进入TryBeMaster提
        议自己作为Master；
        
        - PaxosTryCommitRet_Conflict：
        提议的值被其它提议抢占，如果其它节点成为了Master，且本节点成功学习到，则等待一段时间之后再次进入TryBeMaster的时候无需执行任何操作；如果其它节点成为了Master，但是本节点未成功学习到，则等待一段时间之后再次进入TryBeMaster的时候会再次提议自己作为Master，但是此时因为版本号的缘故，无法获得多数派；如果仍然没有任何节点成为新的Master，则等待一段时间再
        次进入TryBeMaster的时候会再次提议自己作为Master；
        
        - PaxosTryCommitRet_ExecuteFail：
        将决议运用到状态机的时候失败，则等待一段时间之后，如果在等待期间成功学习到新的Master，则在等待时间结束再次进入TryBeMaster的时候无需做任何工作；如果在等待期间未学习到新的Master，则在等待时间结束之后再次进入TryBeMaster提议自己作为Master；
        
        - PaxosTryCommitRet_Timeout：
        本次提议的提交超时，则等待一段时间之后，如果在等待期间成功学习到新的Master，则在等待时间结束再次进入TryBeMaster的时候无需做任何工作；如果在等待期间未学习到新的Master，则在等待时间结束之后再次进入TryBeMaster提议自己作为Master；

## Drop Master是怎么回事？
字面理解，就是放弃作为Master。在PhxPaxos中为业务提供了一个接口，专用于放弃自己作为Master。一旦MasterMgr检测到应用调用了DropMaster接口，则会设置iContinueLeaseTimeout = (2 * iLeaseTime)，并在调用TryBeMaster之后等待iContinueLeaseTimeout - iRunTime的时间之后才会调用下一次TryBeMaster。

如果本节点自身是Master，那么本节点当前的任期是从调用TryBeMaster时刻开始的iLeaseTime时间段，其它节点认为的Master节点的任期是从学习到该Master的时刻开始的iLeaseTime时间段，因为本节点从提议自己作为Master节点（无论是续租还是因为当前Master节点任期结束）的那一刻起，会启动一个ID为m_iCommitTimerID的定时器，该定时器的超时时间为从添加该定时器开始的时刻起的iLeaseTime时间段，既然本节点提议自己作为Master的提议获得通过，那么该定时器一定未过期，也就是说TryBeMaster所花费的时间iRunTime一定不超过iLeaseTime，那么该节点被其它节点学习到的时刻也不会晚于从调用TryBeMaster开始的时间加上iLeaseTime，然后这些学习到该Master的节点从学习到的时刻开始的iLeaseTime时间内都会认为该Master是有效的，也就是说从TryBeMaster开始的(2 * iLeaseTime)的时间之后所有节点都不会再认可之前学习到的Master，而在调用了DropMaster接口的情况下，Master节点自身会在上一次调用TryBeMaster到下一次调用TryBeMaster之间花费(2 * iLeaseTime)的时间，所以其它节点一定会提议自己作为Master，本节点将不再作为Master了。

## 源码分析
### MasterMgr运行主体（不断尝试提议自己作为Master）

```
void MasterMgr :: run()
{
    m_bIsStarted = true;

    while(true)
    {
        if (m_bIsEnd)
        {
            return;
        }
        
        int iLeaseTime = m_iLeaseTime;

        uint64_t llBeginTime = Time::GetSteadyClockMS();
        
        /* 如果当前Master为nullnode，或者当前Master的租约过期了，就要主动提议我自己
         * 作为Master，如果当前的Master就是我自己，则考虑续租，否则不会做任何事情
         */
        TryBeMaster(iLeaseTime);

        int iContinueLeaseTimeout = (iLeaseTime - 100) / 4;
        /*@iContinueLeaseTimeout介于[(iLeaseTime - 100)/8, 3*(iLeaseTime - 100)/8]*/
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

void MasterMgr :: TryBeMaster(const int iLeaseTime)
{
    nodeid_t iMasterNodeID = nullnode;
    uint64_t llMasterVersion = 0;

    //step 1 check exist master and get version
    /* 检查当前本地记录的Master信息，如果已经超时，则@iMasterNodeID将被设置
     * 为nullnode，否则设置为MasterStateMachine中所记录的Master
     */
    m_oDefaultMasterSM.SafeGetMaster(iMasterNodeID, llMasterVersion);
    /* 如果当前的Master有效，且不是我自己，则直接退出，因为当前的Master还在有效期内，
     * 很可能就会续租了，而如果自身是Master的话，那么自身就需要考虑续租了（也就是执行
     * TryBeMaster）
     */
    if (iMasterNodeID != nullnode && (iMasterNodeID != m_poPaxosNode->GetMyNodeID()))
    {
        PLG1Imp("Ohter as master, can't try be master, masterid %lu myid %lu", 
                iMasterNodeID, m_poPaxosNode->GetMyNodeID());
        return;
    }

    BP->GetMasterBP()->TryBeMaster();

    //step 2 try be master
    /* 至此，要么当前Master节点不存在，要么当前Master节点失效，要么我自己是Master节点，
     * 对于Master节点不存在或者Master节点失效的情况，我都要自告奋勇争做Master，对于
     * Master节点是我自己的情况，则考虑续租
     */
     
    /* 生成提议，存放于@sPaxosValue中，这里第3个参数是@iLeaseTime，表示Master节点的
     * 超时时间，主要用于其它节点（非Master节点）学习到新的Master节点信息的时候，计
     * 算Master节点的任期（从学习到Master节点的那一时刻开始的@iLeaseTime的时间之内
     * 就是该节点认可的Master的任期，见MasterStateMachine::Execute -> 
     * MasterStateMachine::LearnMaster）
     */
    std::string sPaxosValue;
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

    /* @iMasterLeaseTimeout在@iLeaseTime的基础上减了100us，是对从调用TryBeMaster
     * 到现在所花费的时间的预估
     *
     * @llAbsMasterTimeout表示如果我被选作为Master节点，那么我所认为的我的任期是
     * 从现在开始到@llAbsMasterTimeout结束的时间段内，如果我被选作Master节点且
     * 还想继续作为Master节点，那么我必须在该时间段以内进行续约
     *
     * 那么为什么不从我确定被选择为Master的那一时刻开始计算我的任期呢？因为如果
     * 我被选择为Master，那么我必须在其它节点认为我的任期到期之前就续租，如果
     * Master节点自身也从学习到Master的那一时刻开始计算它的任期，那么没法保证在
     * 其它任何节点认为的Master到期时间之前续租，因为没法保证Master比其它节点更
     * 早学习到它作为Master这一信息，参考“https://zhuanlan.zhihu.com/p/21540239”。
     */
    const int iMasterLeaseTimeout = iLeaseTime - 100;
    uint64_t llAbsMasterTimeout = Time::GetSteadyClockMS() + iMasterLeaseTimeout; 
    uint64_t llCommitInstanceID = 0;

    SMCtx oCtx;
    /*采用MasterStateMachine特定的SMID*/
    oCtx.m_iSMID = MASTER_V_SMID;
    /*如果我自己最终成为了Master，则我的任期时间*/
    oCtx.m_pCtx = (void *)&llAbsMasterTimeout;

    /*提议，实际调用PNode::Propose*/
    int ret = m_poPaxosNode->Propose(m_iMyGroupIdx, sPaxosValue, llCommitInstanceID, &oCtx);
    if (ret != 0)
    {
        BP->GetMasterBP()->TryBeMasterProposeFail();
    }
}

void MasterStateMachine :: SafeGetMaster(nodeid_t & iMasterNodeID, uint64_t & llMasterVersion)
{
    std::lock_guard<std::mutex> oLockGuard(m_oMutex);

    if (Time::GetSteadyClockMS() >= m_llAbsExpireTime)
    {
        /* 如果已经超时（Master没有来续租），则当前记录的Master已经失效，
         * 尝试提议我自己为Master，返回nullnode给调用者
         */
        iMasterNodeID = nullnode;
    }
    else
    {
        /*返回当前的Master*/
        iMasterNodeID = m_iMasterNodeID;
    }

    llMasterVersion = m_llMasterVersion;
}

int PNode :: Propose(const int iGroupIdx, const std::string & sValue, uint64_t & llInstanceID, SMCtx * poSMCtx)
{
    if (!CheckGroupID(iGroupIdx))
    {
        return Paxos_GroupIdxWrong;
    }

    /*找到该Paxos group对应的Committer，由它来负责提交提案*/
    return m_vecGroupList[iGroupIdx]->GetCommitter()->NewValueGetID(sValue, llInstanceID, poSMCtx);
}

int Committer :: NewValueGetID(const std::string & sValue, uint64_t & llInstanceID, SMCtx * poSMCtx)
{
    BP->GetCommiterBP()->NewValue();

    /*重试至多3次*/
    int iRetryCount = 3;
    int ret = PaxosTryCommitRet_OK;
    while(iRetryCount--)
    {
        TimeStat oTimeStat;
        oTimeStat.Point();

        /*提交提案，这里会同步等待提案执行结果*/
        ret = NewValueGetIDNoRetry(sValue, llInstanceID, poSMCtx);
        if (ret != PaxosTryCommitRet_Conflict)
        {
            /*如果提案执行完毕，没有冲突，则退出*/
            if (ret == 0)
            {
                BP->GetCommiterBP()->NewValueCommitOK(oTimeStat.Point());
            }
            else
            {
                BP->GetCommiterBP()->NewValueCommitFail();
            }
            break;
        }

        BP->GetCommiterBP()->NewValueConflict();

        /*本提案和其它提案存在冲突，需要重试，但是关于Master的提案不重试*/
        if (poSMCtx != nullptr && poSMCtx->m_iSMID == MASTER_V_SMID)
        {
            //master sm not retry
            break;
        }
    }

    return ret;
}

int Committer :: NewValueGetIDNoRetry(const std::string & sValue, uint64_t & llInstanceID, SMCtx * poSMCtx)
{
    LogStatus();

    int iLockUseTimeMs = 0;
    /* 尝试申请锁，至多等待@m_iTimeoutMs的时间，如果成功申请到锁，或者该锁等待超时，
     * 则返回该锁申请所花费的时间，如果成功申请到锁，则返回true，否则返回false
     */
    bool bHasLock = m_oWaitLock.Lock(m_iTimeoutMs, iLockUseTimeMs);
    if (!bHasLock)
    {
        /*没有申请到锁*/
        
        if (iLockUseTimeMs > 0)
        {
            /*该请求等待超时了*/
            BP->GetCommiterBP()->NewValueGetLockTimeout();
            PLGErr("Try get lock, but timeout, lockusetime %dms", iLockUseTimeMs);
            return PaxosTryCommitRet_Timeout; 
        }
        else
        {
            /*有过多的请求在等待，所以本请求不能进入等待队列，直接被拒绝了*/
            BP->GetCommiterBP()->NewValueGetLockReject();
            PLGErr("Try get lock, but too many thread waiting, reject");
            return PaxosTryCommitRet_TooManyThreadWaiting_Reject;
        }
    }

    /*成功申请到锁*/
    int iLeftTimeoutMs = -1;
    if (m_iTimeoutMs > 0)
    {
        iLeftTimeoutMs = m_iTimeoutMs > iLockUseTimeMs ? m_iTimeoutMs - iLockUseTimeMs : 0;
        if (iLeftTimeoutMs < 200)
        {
            /*但是获取锁花费的时间太长了，释放锁，重新申请*/
            PLGErr("Get lock ok, but lockusetime %dms too long, lefttimeout %dms", iLockUseTimeMs, iLeftTimeoutMs);

            BP->GetCommiterBP()->NewValueGetLockTimeout();

            m_oWaitLock.UnLock();
            return PaxosTryCommitRet_Timeout;
        }
    }

    PLGImp("GetLock ok, use time %dms", iLockUseTimeMs);
    
    BP->GetCommiterBP()->NewValueGetLockOK(iLockUseTimeMs);

    //pack smid to value
    /*获取SMID，并打包到提议中*/
    int iSMID = poSMCtx != nullptr ? poSMCtx->m_iSMID : 0;
    
    string sPackSMIDValue = sValue;
    m_poSMFac->PackPaxosValue(sPackSMIDValue, iSMID);

    /*初始化CommitCtx*/
    m_poCommitCtx->NewCommit(&sPackSMIDValue, poSMCtx, iLeftTimeoutMs);
    /* 添加一个空消息（nullptr）到IOLoop中（IOLoop拥有独立的线程），在IOLoop中
     * 该消息会被调度处理，见IOLoop::run -> IOLoop::OneLoop -> Instance::CheckNewValue，
     */
    m_poIOLoop->AddNotify();

    /*等待提交（Commit）的结果（会同步等待，直到该请求被提交），如果成功提交，则返回对应的Instance ID*/
    int ret = m_poCommitCtx->GetResult(llInstanceID);

    m_oWaitLock.UnLock();
    return ret;
}

void CommitCtx :: NewCommit(std::string * psValue, SMCtx * poSMCtx, const int iTimeoutMs)
{
    m_oSerialLock.Lock();

    m_llInstanceID = (uint64_t)-1;
    m_iCommitRet = -1;
    m_bIsCommitEnd = false;
    m_iTimeoutMs = iTimeoutMs;

    m_psValue = psValue;
    m_poSMCtx = poSMCtx;

    if (psValue != nullptr)
    {
        PLGHead("OK, valuesize %zu", psValue->size());
    }

    m_oSerialLock.UnLock();
}

/*处理接收到的消息、提议和retry事件*/
void IOLoop :: OneLoop(const int iTimeoutMs)
{
    std::string * psMessage = nullptr;

    m_oMessageQueue.lock();
    bool bSucc = m_oMessageQueue.peek(psMessage, iTimeoutMs);
    
    if (!bSucc)
    {
        m_oMessageQueue.unlock();
    }
    else
    {
        m_oMessageQueue.pop();
        m_oMessageQueue.unlock();

        /*处理接收到的消息*/
        if (psMessage != nullptr && psMessage->size() > 0)
        {
            m_iQueueMemSize -= psMessage->size();
            m_poInstance->OnReceive(*psMessage);
        }

        delete psMessage;

        BP->GetIOLoopBP()->OutQueueMsg();
    }

    /*处理retry事件*/
    DealWithRetry();

    /*处理新的提议*/
    m_poInstance->CheckNewValue();
}


void Instance :: CheckNewValue()
{
    /* 在Committer::NewValueGetIDNoRetry -> CommitCtx::NewCommit中会设置
     * @Commiter::m_poCommitCtx，而@Commiter::m_poCommitCtx实际上就是
     * Instance::m_oCommitCtx的地址，所以一旦调用了CommitCtx::NewCommit，
     * 这里的m_oCommitCtx.IsNewCommit()判断就会成立（这是一个新的提交）
     */
    if (!m_oCommitCtx.IsNewCommit())
    {
        return;
    }

    /* 判断我是否已经对齐到其它节点，在AskForLearn过程中，Learner会记录下来
     * 它所看到的其它节点上的最大的InstanceID，如果自己当前的InstanceID小于
     * 它所看到的其它节点上最大的InstanceID，就需要先对齐，然后才能接受新的
     * 提案（参考“微信自研生产级paxos类库PhxPaxos实现原理介绍”一文中的“实例
     * 的对齐”一节）
     */
    if (!m_oLearner.IsIMLatest())
    {
        return;
    }

    /*我是Follower，不接受提案*/
    if (m_poConfig->IsIMFollower())
    {
        PLGErr("I'm follower, skip this new value");
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Follower_Cannot_Commit);
        return;
    }

    /*我不在成员组管理中*/
    if (!m_poConfig->CheckConfig())
    {
        PLGErr("I'm not in membership, skip this new value");
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Im_Not_In_Membership);
        return;
    }

    /*提案内容太大*/
    if ((int)m_oCommitCtx.GetCommitValue().size() > MAX_VALUE_SIZE)
    {
        PLGErr("value size %zu to large, skip this new value",
            m_oCommitCtx.GetCommitValue().size());
        m_oCommitCtx.SetResultOnlyRet(PaxosTryCommitRet_Value_Size_TooLarge);
        return;
    }

    /*OK，可以提交了，给它分配一个InstanceID*/
    m_oCommitCtx.StartCommit(m_oProposer.GetInstanceID());

    if (m_oCommitCtx.GetTimeoutMs() != -1)
    {
        /* 如果指定了超时时间，则添加一个定时器到IOLoop中，定时器类型为
         * Timer_Instance_Commit_Timeout，如果发生超时，则回调函数为
         * Instance::OnNewValueCommitTimeout
         */
        m_oIOLoop.AddTimer(m_oCommitCtx.GetTimeoutMs(), Timer_Instance_Commit_Timeout, m_iCommitTimerID);
    }
    
    m_oTimeStat.Point();

    if (m_poConfig->GetIsUseMembership()
            && (m_oProposer.GetInstanceID() == 0 || m_poConfig->GetGid() == 0))
    {
        /*第一个Propose操作，且支持成员组管理，那么执行集群初始化*/
        
        /*初始化SystemVariables*/
        PLGHead("Need to init system variables, Now.InstanceID %lu Now.Gid %lu", 
                m_oProposer.GetInstanceID(), m_poConfig->GetGid());

        /*生成Gid*/
        uint64_t llGid = OtherUtils::GenGid(m_poConfig->GetMyNodeID());
        string sInitSVOpValue;
        /* 设置SystemVariables::Gid，并将SystemVariables序列化为字符串@sInitSVOpValue
         * SystemVariables中包含gid，version和membership信息，见“成员组管理”
         */
        int ret = m_poConfig->GetSystemVSM()->CreateGid_OPValue(llGid, sInitSVOpValue);
        assert(ret == 0);

        m_oSMFac.PackPaxosValue(sInitSVOpValue, m_poConfig->GetSystemVSM()->SMID());
        m_oProposer.NewValue(sInitSVOpValue);
    }
    else
    {
        /* 如果设置了使用BeforePropose功能，则在各状态机上调用BeforePropose，
         * BeforePropose函数主要用于修改提议？参考sm.h中关于BeforePropose的说明。
         * 那为什么要修改提议呢？因为提议会被首先交给Committer，由Committer触发
         * IOLoop去调度执行，Committer的提交和IOLoop的调度执行是在不同的线程中，
         * Committer提交和IOLoop调度执行之间可能存在较大时间差，如果该提议的内容
         * 与系统状态有关，在该较大时间差内系统状态可能会发生改变，而提议中的用到
         * 的是提议的那一时刻的状态，而系统当前处于另外一个新状态，那么在Propose之前
         * 这是最后一个机会来更新提议内容到最新状态了。
         *
         * MasterStateMachine中提供了该函数，参考MasterStateMachine::BeforePropose，
         * 通过MasterStateMachine::BeforePropose可以更清楚看到BeforePropse的作用。
         */
        if (m_oOptions.bOpenChangeValueBeforePropose) {
            m_oSMFac.BeforePropose(m_poConfig->GetMyGroupIdx(), m_oCommitCtx.GetCommitValue());
        }
        
        /*交由Proposer，进入Prepare + Accept阶段*/
        m_oProposer.NewValue(m_oCommitCtx.GetCommitValue());
    }
}

void CommitCtx :: StartCommit(const uint64_t llInstanceID)
{
    m_oSerialLock.Lock();
    /*设置Instance ID*/
    m_llInstanceID = llInstanceID;
    m_oSerialLock.UnLock();
}

void SMFac :: BeforePropose(const int iGroupIdx, std::string & sValue)
{
    int iSMID = 0;
    /*从@sValue中解析出SMID @iSMID*/
    memcpy(&iSMID, sValue.data(), sizeof(int));

    if (iSMID == 0)
    {
        return;
    }

    if (iSMID == BATCH_PROPOSE_SMID)
    {
        BeforeBatchPropose(iGroupIdx, sValue);
    }
    else
    {
        bool change = false;
        string sBodyValue = string(sValue.data() + sizeof(int), sValue.size() - sizeof(int));
        /*找到该@iSMID对应的状态机，并调用其BeforePropose函数*/
        BeforeProposeCall(iGroupIdx, iSMID, sBodyValue, change);
        if (change)
        {
            sValue.erase(sizeof(int));
            sValue.append(sBodyValue);
        }
    }
}

void SMFac :: BeforeProposeCall(const int iGroupIdx, const int iSMID, std::string & sBodyValue, bool & change)
{
    if (iSMID == 0)
    {
        return;
    }

    if (m_vecSMList.size() == 0)
    {
        return;
    }

    /* 在该group中找到给定SMID @iSMID对应的状态机，如果其NeedCallBeforePropose()返回
     * true，则调用BeforePropose()函数
     */
    for (auto & poSM : m_vecSMList)
    {
        if (poSM->SMID() == iSMID)
        {
            if (poSM->NeedCallBeforePropose()) {
                /*的确发生了改变*/
                change  = true;
                return poSM->BeforePropose(iGroupIdx, sBodyValue);
            }
        }
    }
}

/*MasterStateMachine提供了BeforePropose功能，用于在Propose之前更新关于Master的提议*/
void MasterStateMachine :: BeforePropose(const int iGroupIdx, std::string & sValue)
{
    std::lock_guard<std::mutex> oLockGuard(m_oMutex);
    MasterOperator oMasterOper;
    /*从@sValue中解析出Master选举相关的信息@oMasterOper*/
    bool bSucc = oMasterOper.ParseFromArray(sValue.data(), sValue.size());
    if (!bSucc)
    {
        return;
    }

    /* 设置MasterOperator::lastversion_，在MasterMgr::TryBeMaster中调用
     * MasterStateMachine::MakeOpValue生成提议的时候，已经设置了版本号，
     * 但是设置的是MasterOperator::version_，表示在提议的那一时刻看到的
     * MasterStateMachine的版本号，从关于Master的提议提出到现在即将进入
     * Prepare阶段的期间，可能MasterStateMachine中的版本号发生了改变，这
     * 里通过设置MasterOperator::lastversion_来反映该改变
     *
     * 那么MasterOperator::version_和MasterOperator::lastversion_在又是
     * 如何使用的呢？
     * 请参考MasterStateMachine::Execute -> MasterStateMachine::LearnMaster
     */
    oMasterOper.set_lastversion(m_llMasterVersion);
    /*清空@sValue*/
    sValue.clear();
    /*重新将@oMasterOper序列化到@sValue中*/
    bSucc = oMasterOper.SerializeToString(&sValue);
    assert(bSucc == true);
} 

/*如果提案提交超时*/
void Instance :: OnNewValueCommitTimeout()
{
    BP->GetInstanceBP()->OnNewValueCommitTimeout();

    /* 退出Prepare状态，删除Prepare相关的定时器，
     * 退出Accept状态，删除Accept相关的定时器
     */
    m_oProposer.ExitPrepare();
    m_oProposer.ExitAccept();

    /*设置该提案超时*/
    m_oCommitCtx.SetResult(PaxosTryCommitRet_Timeout, m_oProposer.GetInstanceID(), "");
}

int CommitCtx :: GetResult(uint64_t & llSuccInstanceID)
{
    m_oSerialLock.Lock();

    /*在NewCommit中@m_bIsCommitEnd被设置为false，等待直到@m_IsCommitEnd被设置为true*/
    while (!m_bIsCommitEnd)
    {
        m_oSerialLock.WaitTime(1000);
    }

    if (m_iCommitRet == 0)
    {
        /*提交成功，则返回对应的Instance ID*/
        llSuccInstanceID = m_llInstanceID;
        PLGImp("commit success, instanceid %lu", llSuccInstanceID);
    }
    else
    {
        PLGErr("commit fail, ret %d", m_iCommitRet);
    }
    
    m_oSerialLock.UnLock();

    return m_iCommitRet;
}
```

### 关于Master的提议（提议新的Master，或者现有Master续租任期）获得多数派通过

```
void Proposer :: OnAcceptReply(const PaxosMsg & oPaxosMsg)
{
    ......

    if (m_oMsgCounter.IsPassedOnThisRound())
    {
        /*多数派通过*/
        int iUseTimeMs = m_oTimeStat.Point();
        BP->GetProposerBP()->AcceptPass(iUseTimeMs);
        PLGImp("[Pass] Start send learn, usetime %dms", iUseTimeMs);
        ExitAccept();
        /*本地Learner广播提案成功的消息给其它各Learner*/
        m_poLearner->ProposerSendSuccess(GetInstanceID(), m_oProposerState.GetProposalID());
    }
    else if (m_oMsgCounter.IsRejectedOnThisRound()
            || m_oMsgCounter.IsAllReceiveOnThisRound())
    {
        /*未通过*/
        BP->GetProposerBP()->AcceptNotPass();
        PLGImp("[Not pass] wait 30ms and Restart prepare");
        AddAcceptTimer(OtherUtils::FastRand() % 30 + 10);
    }
}

void Learner :: ProposerSendSuccess(
        const uint64_t llLearnInstanceID,
        const uint64_t llProposalID)
{
    BP->GetLearnerBP()->ProposerSendSuccess();

    PaxosMsg oPaxosMsg;
    
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_ProposerSendSuccess);
    oPaxosMsg.set_instanceid(llLearnInstanceID);
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalid(llProposalID);
    oPaxosMsg.set_lastchecksum(GetLastChecksum());

    //run self first
    BroadcastMessage(oPaxosMsg, BroadcastMessage_Type_RunSelf_First);
}
```

### 各节点上的Learner接收到提案成功的消息，运用决议到状态机

```
int Instance :: ReceiveMsgForLearner(const PaxosMsg & oPaxosMsg)
{
    ......
    
    else if (oPaxosMsg.msgtype() == MsgType_PaxosLearner_ProposerSendSuccess)
    {
        /*接收到提案成功的消息*/
        m_oLearner.OnProposerSendSuccess(oPaxosMsg);
    }
    
    ......

    /*成功学习到决议*/
    if (m_oLearner.IsLearned())
    {
        BP->GetInstanceBP()->OnInstanceLearned();

        SMCtx * poSMCtx = nullptr;
        /* 当前学习到的决议对应的是否是我自己提出的最新的提案（因为只有一个
         * Committer，所以只有一个Learner在IsMyCommit的时候返回true）
         */
        bool bIsMyCommit = m_oCommitCtx.IsMyCommit(m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue(), poSMCtx);

        if (!bIsMyCommit)
        {
            BP->GetInstanceBP()->OnInstanceLearnedNotMyCommit();
            PLGDebug("this value is not my commit");
        }
        else
        {
            int iUseTimeMs = m_oTimeStat.Point();
            BP->GetInstanceBP()->OnInstanceLearnedIsMyCommit(iUseTimeMs);
            PLGHead("My commit ok, usetime %dms", iUseTimeMs);
        }

        /* 无论该学习到的决议对应的是否是我提交的提案，都运用该决议到状态机
         * （以实现状态机状态切换），这里会针对当前group中所挂载的所有状态机
         * 上运用该决议，对于Master选举来说，也有其自己的状态机，MasterStateMachine，
         * 所以最终会执行到MasterStateMachine :: Execute
         */
        if (!SMExecute(m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue(), bIsMyCommit, poSMCtx))
        {
            BP->GetInstanceBP()->OnInstanceLearnedSMExecuteFail();

            PLGErr("SMExecute fail, instanceid %lu, not increase instanceid", m_oLearner.GetInstanceID());
            /* SetResult中会设置该提案的执行结果，并设置CommitCtx::m_bIsCommitEnd
             * 为true，同时唤醒正在等待提案执行结果的Committer(Committer通过
             * CommitCtx::GetResult同步等待当前提案的执行结果，CommitCtx :: SetResult
             * 则唤醒之)
             */
            m_oCommitCtx.SetResult(PaxosTryCommitRet_ExecuteFail, 
                    m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue());

            m_oProposer.CancelSkipPrepare();

            return -1;
        }
        
        {
            /*设置本次提交的结果为OK*/
            m_oCommitCtx.SetResult(PaxosTryCommitRet_OK
                    , m_oLearner.GetInstanceID(), m_oLearner.GetLearnValue());

            if (m_iCommitTimerID > 0)
            {
                m_oIOLoop.RemoveTimer(m_iCommitTimerID);
            }
        }
        
        m_iLastChecksum = m_oLearner.GetNewChecksum();

        /*为下一个提议做准备（增长InstanceID）*/
        NewInstance();

        /*设置当前已经成功运用到状态机的决议对应的最大InstanceID*/
        m_oCheckpointMgr.SetMaxChosenInstanceID(m_oAcceptor.GetInstanceID());

        BP->GetInstanceBP()->NewInstance();
    }

    return 0;
}

void Learner :: OnProposerSendSuccess(const PaxosMsg & oPaxosMsg)
{
    BP->GetLearnerBP()->OnProposerSendSuccess();

    /*InstanceID不匹配*/
    if (oPaxosMsg.instanceid() != GetInstanceID())
    {
        //Instance id not same, that means not in the same instance, ignord.
        PLGDebug("InstanceID not same, skip msg");
        return;
    }

    /*尚未接受任何的提案*/
    if (m_poAcceptor->GetAcceptorState()->GetAcceptedBallot().isnull())
    {
        //Not accept any yet.
        BP->GetLearnerBP()->OnProposerSendSuccessNotAcceptYet();
        PLGDebug("I haven't accpeted any proposal");
        return;
    }

    BallotNumber oBallot(oPaxosMsg.proposalid(), oPaxosMsg.nodeid());

    /*和Acceptor已经接受的最新的提案不匹配*/
    if (m_poAcceptor->GetAcceptorState()->GetAcceptedBallot()
            != oBallot)
    {
        //Proposalid not same, this accept value maybe not chosen value.
        PLGDebug("ProposalBallot not same to AcceptedBallot");
        BP->GetLearnerBP()->OnProposerSendSuccessBallotNotSame();
        return;
    }

    /*学习（最新成功的）提案，只在内存中更新学习到的提案内容*/
    m_oLearnerState.LearnValueWithoutWrite(
            oPaxosMsg.instanceid(),
            m_poAcceptor->GetAcceptorState()->GetAcceptedValue(),
            m_poAcceptor->GetAcceptorState()->GetChecksum());
    
    BP->GetLearnerBP()->OnProposerSendSuccessSuccessLearn();

    PLGHead("END Learn value OK, value %zu", m_poAcceptor->GetAcceptorState()->GetAcceptedValue().size());

    /*转发给各Follower*/
    TransmitToFollower();
}

void LearnerState :: LearnValueWithoutWrite(const uint64_t llInstanceID, 
        const std::string & sValue, const uint32_t iNewChecksum)
{
    /*更新学习到的最新的决议*/
    m_sLearnedValue = sValue;
    /*设置成功学习到了最新的决议*/
    m_bIsLearned = true;
    m_iNewChecksum = iNewChecksum;
}

bool MasterStateMachine :: Execute(const int iGroupIdx, const uint64_t llInstanceID, 
        const std::string & sValue, SMCtx * poSMCtx)
{
    MasterOperator oMasterOper;
    /*从@sValue中解析出@oMasterOper*/
    bool bSucc = oMasterOper.ParseFromArray(sValue.data(), sValue.size());
    if (!bSucc)
    {
        PLG1Err("oMasterOper data wrong");
        return false;
    }

    if (oMasterOper.operator_() == MasterOperatorType_Complete)
    {
        /* @oMasterOper中会记录Master节点自身任期的截止时间，存放在@pAbsMasterTimeout
         * 中，参考MasterMgr :: TryBeMaster
         */
        uint64_t * pAbsMasterTimeout = nullptr;
        if (poSMCtx != nullptr && poSMCtx->m_pCtx != nullptr)
        {
            pAbsMasterTimeout = (uint64_t *)poSMCtx->m_pCtx;
        }

        uint64_t llAbsMasterTimeout = pAbsMasterTimeout != nullptr ? *pAbsMasterTimeout : 0;

        PLG1Imp("absmaster timeout %lu", llAbsMasterTimeout);

        /*学习Master信息*/
        int ret = LearnMaster(llInstanceID, oMasterOper, llAbsMasterTimeout);
        if (ret != 0)
        {
            return false;
        }
    }
    else
    {
        PLG1Err("unknown op %u", oMasterOper.operator_());
        //wrong op, just skip, so return true;
        return true;
    }

    return true;
}

int MasterStateMachine :: LearnMaster(
        const uint64_t llInstanceID, 
        const MasterOperator & oMasterOper, 
        const uint64_t llAbsMasterTimeout)
{
    std::lock_guard<std::mutex> oLockGuard(m_oMutex);

    PLG1Debug("my last version %lu other last version %lu this version %lu instanceid %lu",
            m_llMasterVersion, oMasterOper.lastversion(), oMasterOper.version(), llInstanceID);

    if (oMasterOper.lastversion() != 0
            && llInstanceID > m_llMasterVersion
            && oMasterOper.lastversion() != m_llMasterVersion)
    {
        /* 决议对应的提议被Propose时刻的版本号跟我现在看到的最新版本号不一致，
         * 尝试更新我的版本号为决议中的版本号
         *
         * 因为当走到MasterStateMachine :: LearnMaster这里的时候，前面一定经过了
         * Instance :: ReceiveMsgForLearner，在该函数中学习任何决议之前都会检查
         * InstanceID是否匹配，不匹配就不能学习之，既然InstanceID匹配，那么在该
         * Master提议过程中不可能有任何其它关于新的Master提议的Instance，所以选择
         * 相信决议对应的提议被Propose时刻的版本号，即oMasterOper.lastversion()
         */
        BP->GetMasterBP()->MasterSMInconsistent();
        PLG1Err("other last version %lu not same to my last version %lu, instanceid %lu",
                oMasterOper.lastversion(), m_llMasterVersion, llInstanceID);

        PLG1Err("try to fix, set my master version %lu as other last version %lu, instanceid %lu",
                m_llMasterVersion, oMasterOper.lastversion(), llInstanceID);
        m_llMasterVersion = oMasterOper.lastversion();
    }

    /* 在MasterStateMachine::BeforePropose中为什么没有比较MasterOperator::version()
     * 和m_llMasterVersion？
     */
    if (oMasterOper.version() != m_llMasterVersion)
    {
        PLG1Debug("version conflit, op version %lu now master version %lu",
                oMasterOper.version(), m_llMasterVersion);
        return 0;
    }

    /*更新Master相关信息到MasterVariablesStore中*/
    int ret = UpdateMasterToStore(oMasterOper.nodeid(), llInstanceID, oMasterOper.timeout());
    if (ret != 0)
    {
        PLG1Err("UpdateMasterToStore fail, ret %d", ret);
        return -1;
    }

    /*当前的Master所在的节点跟决议中Master不一样，表示Master发生了切换*/
    bool bMasterChange = false;
    if (m_iMasterNodeID != oMasterOper.nodeid())
    {
        bMasterChange = true;
    }

    /*更新Master节点*/
    m_iMasterNodeID = oMasterOper.nodeid();
    if (m_iMasterNodeID == m_iMyNodeID)
    {
        /*我自己作为Master，设置我的任期结束时间*/
        //self be master
        //use local abstimeout
        m_llAbsExpireTime = llAbsMasterTimeout;

        BP->GetMasterBP()->SuccessBeMaster();
        PLG1Head("Be master success, absexpiretime %lu", m_llAbsExpireTime);
    }
    else
    {
        /*我不是Master，设置我所信任的Master的任期为从现在起的oMasterOper.timeout()时间*/
        //other be master
        //use new start timeout
        m_llAbsExpireTime = Time::GetSteadyClockMS() + oMasterOper.timeout();

        BP->GetMasterBP()->OtherBeMaster();
        PLG1Head("Ohter be master, absexpiretime %lu", m_llAbsExpireTime);
    }

    /*更新LeaseTime和Master版本号（采用当前最大的InstanceID作为Master版本号）*/
    m_iLeaseTime = oMasterOper.timeout();
    m_llMasterVersion = llInstanceID;

    if (bMasterChange)
    {
        /*如果发生了Master切换，则调用注册的Master切换回调函数*/
        if (m_pMasterChangeCallback != nullptr)
        {
            m_pMasterChangeCallback(m_iMyGroupIdx, NodeInfo(m_iMasterNodeID), m_llMasterVersion);
        }
    }

    PLG1Imp("OK, masternodeid %lu version %lu abstimeout %lu",
            m_iMasterNodeID, m_llMasterVersion, m_llAbsExpireTime);

    return 0;
}
```

# Q & A
## 关于PhxPaxos中的Master
在Phxpaxos的设计中弱化了Multi-Paxos中提到的Leader角色，而是使用了Master的概念，在PhxSQL项目的文档中也强调了：Master是唯一具有外部写权限的Node，所以可以保证这个Node的数据肯定是最新的副本，Client的所有写请求和强一致性的读请求都需要直接或者由其他Node代理转发到这个Master Node上面，而对数据一致性要求不高的普通读请求，其读请求才可以在非Master Node上面执行。

## 为什么Master选举过程中每一个Master任期都有一个Version信息
![image](https://pic1.zhimg.com/80/78965fda565993c28f328e909d2d0785_hd.jpg)

这个图示情况是NodeA不断的在续任，但NodeC可能与NodeA无法通信或者其他原因，在获知NodeA第二次续任成功后就再也收不到任何消息了，于是当NodeC认为A的Master任期过期后，即可尝试发起BeMaster操作。这就违背了算法的保证了，出现了NodeA在任期内，但NodeC发起BeMaster操作的情况。
这里问题的本质是，NodeC还未获得最新的Master情况，所以发起了一次错误的BeMaster。version的加入是参考了乐观锁来解决这个问题。发起BeMaster的时候携带上一次的version，如果这个version已经不是最新，那么这一次BeMaster自然会失效，从而解决问题。

# 参考资料
[Master选举](https://zhuanlan.zhihu.com/p/21540239)

[从PhxPaxos中再看Paxos协议工程实现](https://taozj.net/201611/learn-note-of-distributed-system-(3)-see-paxos-from-phxpaxos.html)