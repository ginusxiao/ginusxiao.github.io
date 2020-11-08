# 提纲
[toc]

# Config相关类定义
## Config类定义
```
class Config
{
private:
    /*是否同步*/
    bool m_bLogSync;
    /*同步时间间隔*/
    int m_iSyncInterval;
    /*是否采用Membership管理*/
    bool m_bUseMembership;

    /*本地节点ID*/
    nodeid_t m_iMyNodeID;
    /*总的节点数目*/
    int m_iNodeCount;
    /*所在的paxos group*/
    int m_iMyGroupIdx;
    /*总的paxos group数目*/
    int m_iGroupCount;

    /*集群中所有节点信息列表*/
    NodeInfoList m_vecNodeInfoList;

    /*本节点是否是Follower*/
    bool m_bIsIMFollower;
    /*如果是Follower，那么记录下来Follow哪个节点*/
    nodeid_t m_iFollowToNodeID;

    /*成员组（membership）管理相关状态机*/
    SystemVSM m_oSystemVSM;
    /*Master管理相关的状态机，实际类型为MasterStateMachine*/
    InsideSM * m_poMasterSM;

    std::map<nodeid_t, uint64_t> m_mapTmpNodeOnlyForLearn;
    std::map<nodeid_t, uint64_t> m_mapMyFollower;
};
```

## 成员组管理状态机类(SystemVSM)定义
```
class SystemVSM : public InsideSM 
{
private:
    int m_iMyGroupIdx;
    /*成员组信息*/
    SystemVariables m_oSystemVariables;
    
    /*对LogStorage的封装，用于存放成员组管理相关的信息*/
    SystemVariablesStore m_oSystemVStore;

    /*所有成员的节点ID集合*/
    std::set<nodeid_t> m_setNodeID;

    /*本节点ID*/
    nodeid_t m_iMyNodeID;

    /*成员组变化回调函数*/
    MembershipChangeCallback m_pMembershipChangeCallback;
};

class SystemVariables : public ::google::protobuf::Message {
    ::google::protobuf::uint64 gid_;
    ::google::protobuf::RepeatedPtrField< ::phxpaxos::PaxosNodeInfo > membership_;
    ::google::protobuf::uint64 version_;
};

class SystemVariablesStore
{
private:
    LogStorage * m_poLogStorage;
};
```

## Config相关的类说明
Config类与某个节点上的某个paxos group相关，即每个节点上的每个paxos group都有一个自身相关的配置类Config，在Config类中有两个重要的状态机：成员组管理相关的状态机SystemVSM和Master管理相关的状态机MasterStateMachine。

# Config相关类构造函数
## Config类构造函数
```
Config :: Config(
        const LogStorage * poLogStorage,
        const bool bLogSync,
        const int iSyncInterval,
        const bool bUseMembership,
        const NodeInfo & oMyNode, 
        const NodeInfoList & vecNodeInfoList,
        const FollowerNodeInfoList & vecFollowerNodeInfoList,
        const int iMyGroupIdx,
        const int iGroupCount,
        MembershipChangeCallback pMembershipChangeCallback)
    : m_bLogSync(bLogSync), 
    m_iSyncInterval(iSyncInterval),
    m_bUseMembership(bUseMembership),
    m_iMyNodeID(oMyNode.GetNodeID()), 
    m_iNodeCount(vecNodeInfoList.size()), 
    m_iMyGroupIdx(iMyGroupIdx),
    m_iGroupCount(iGroupCount),
    m_oSystemVSM(iMyGroupIdx, oMyNode.GetNodeID(), poLogStorage, pMembershipChangeCallback),
    m_poMasterSM(nullptr)
{
    m_vecNodeInfoList = vecNodeInfoList;

    m_bIsIMFollower = false;
    m_iFollowToNodeID = nullnode;

    for (auto & oFollowerNodeInfo : vecFollowerNodeInfoList)
    {
        /*本节点是Follower节点*/
        if (oFollowerNodeInfo.oMyNode.GetNodeID() == oMyNode.GetNodeID())
        {
            PLG1Head("I'm follower, ip %s port %d nodeid %lu",
                    oMyNode.GetIP().c_str(), oMyNode.GetPort(), oMyNode.GetNodeID());
            /*设置@m_bIsIMFollower为true*/
            m_bIsIMFollower = true;
            /*设置我所Follow的节点的ID*/
            m_iFollowToNodeID = oFollowerNodeInfo.oFollowNode.GetNodeID();

            InsideOptions::Instance()->SetAsFollower();
        }
    }
}
```

## SystemVSM类构造函数

```
SystemVSM :: SystemVSM(
        const int iGroupIdx, 
        const nodeid_t iMyNodeID,
        const LogStorage * poLogStorage,
        MembershipChangeCallback pMembershipChangeCallback) 
    : m_iMyGroupIdx(iGroupIdx), m_oSystemVStore(poLogStorage), 
    m_iMyNodeID(iMyNodeID), m_pMembershipChangeCallback(pMembershipChangeCallback)
{
}
```

可以看到，SystemVSM中用到的LogStorage依然是这个paxos group的业务逻辑状态机所使用的LogStorage。


# Config初始化
## Config初始化入口
```
int Config :: Init()
{
    /*成员组管理相关的初始化（从LogStorage中恢复成员组信息）*/
    int ret = m_oSystemVSM.Init();
    if (ret != 0)
    {
        PLG1Err("fail, ret %d", ret);
        return ret;
    }

    /*尝试更新成员组信息（这只在成员组信息并未在LogStorage中持久化的情况下有效）*/
    m_oSystemVSM.AddNodeIDList(m_vecNodeInfoList);

    /*MasterStateMachine相关的初始化是在MasterMgr中完成的，不在这里*/
    
    PLG1Head("OK");
    return 0;
}
```

## 成员组管理相关的初始化

```
int SystemVSM :: Init()
{
    /*从LogStorage中读取@m_iMyGroupIdx对应的paxos group相关的成员组信息*/
    int ret = m_oSystemVStore.Read(m_iMyGroupIdx, m_oSystemVariables);
    if (ret != 0 && ret != 1)
    {
        return ret;
    }

    if (ret == 1)
    {
        /*在数据库中没有找到*/
        m_oSystemVariables.set_gid(0);
        m_oSystemVariables.set_version(-1);
        PLG1Imp("variables not exist");
    }
    else
    {
        RefleshNodeID();
        PLG1Imp("OK, gourpidx %d gid %lu version %lu", 
                m_iMyGroupIdx, m_oSystemVariables.gid(), m_oSystemVariables.version());
    }

    return 0;
}

int SystemVariablesStore :: Read(const int iGroupIdx, SystemVariables & oVariables)
{
    const int m_iMyGroupIdx = iGroupIdx;

    string sBuffer;
    /* 这里@m_poLogStorage实际上是MultiDatabase类型，所以最终调用的是
     * MultiDatabase :: GetSystemVariables
     */
    int ret = m_poLogStorage->GetSystemVariables(iGroupIdx, sBuffer);
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

int MultiDatabase :: GetSystemVariables(const int iGroupIdx, std::string & sBuffer)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }

    /*从@iGroupIdx对应的Database实例中获取*/
    return m_vecDBList[iGroupIdx]->GetSystemVariables(sBuffer);
}

int Database :: GetSystemVariables(std::string & sBuffer)
{
    /*key为成员组管理特定的key：SYSTEMVARIABLES_KEY，即(uint64_t)-2*/
    static uint64_t llSystemVariablesKey = SYSTEMVARIABLES_KEY;
    /*从leveldb中读取*/
    return GetFromLevelDB(llSystemVariablesKey, sBuffer);
}

void SystemVSM :: RefleshNodeID()
{
    /*清空@m_setNodeID中记录的旧的成员信息*/
    m_setNodeID.clear();

    NodeInfoList vecNodeInfoList;
    
    /*@m_oSystemVariables.membership_size()表示成员数目，依次将各成员添加到@m_setNodeID中*/
    for (int i = 0; i < m_oSystemVariables.membership_size(); i++)
    {
        PaxosNodeInfo oNodeInfo = m_oSystemVariables.membership(i);
        NodeInfo tTmpNode(oNodeInfo.nodeid());

        PLG1Head("ip %s port %d nodeid %lu", 
                tTmpNode.GetIP().c_str(), tTmpNode.GetPort(), tTmpNode.GetNodeID());

        m_setNodeID.insert(tTmpNode.GetNodeID());

        vecNodeInfoList.push_back(tTmpNode);
    }

    /*PhxPaxos中提供的Options中默认情况下该函数指针为nullptr*/
    if (m_pMembershipChangeCallback != nullptr)
    {
        m_pMembershipChangeCallback(m_iMyGroupIdx, vecNodeInfoList);
    }
}

void SystemVSM :: AddNodeIDList(const NodeInfoList & vecNodeInfoList)
{
    /*如果成员组信息中gid不为0，则表明已经有了成员组信息，无需再添加了*/
    if (m_oSystemVariables.gid() != 0)
    {
        PLG1Err("No need to add, i already have membership info.");
        return;
    }

    /*清理@m_setNodeID和@m_oSystemVariables*/
    m_setNodeID.clear();
    m_oSystemVariables.clear_membership();

    /*依次将@vecNodeInfoList中的节点信息添加到成员组中*/
    for (auto & tNodeInfo : vecNodeInfoList)
    {
        /*在成员组中添加一个元素*/
        PaxosNodeInfo * poNodeInfo = m_oSystemVariables.add_membership();
        /*初始化该成员的rid和nodeid信息，什么是rid???*/
        poNodeInfo->set_rid(0);
        poNodeInfo->set_nodeid(tNodeInfo.GetNodeID());

        NodeInfo tTmpNode(poNodeInfo->nodeid());
    }

    RefleshNodeID();
}
```


