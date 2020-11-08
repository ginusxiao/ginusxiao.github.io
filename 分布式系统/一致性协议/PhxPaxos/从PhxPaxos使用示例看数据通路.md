# 提纲
[toc]

# 示例入口

```
int main(int argc, char ** argv)
{
    /*
     * argv[0]: executable
     * argv[1]: local node's IP/PORT
     * argv[2]: all cluster nodes' IP/PORT
     */
    if (argc < 3)
    {
        printf("%s <myip:myport> <node0_ip:node_0port,node1_ip:node_1_port,node2_ip:node2_port,...>\n", argv[0]);
        return -1;
    }

    /*获取本地节点的IP/PORT，并通过IP/PORT生成NODEID，存放于@oMyNode中*/
    NodeInfo oMyNode;
    if (parse_ipport(argv[1], oMyNode) != 0)
    {
        printf("parse myip:myport fail\n");
        return -1;
    }

    /*获取集群中所有节点的IP/PORT，并分别生成NODEID，存放于@vecNodeInfoList中*/
    NodeInfoList vecNodeInfoList;
    if (parse_ipport_list(argv[2], vecNodeInfoList) != 0)
    {
        printf("parse ip/port list fail\n");
        return -1;
    }

    /*运行PhxPaxos*/
    PhxEchoServer oEchoServer(oMyNode, vecNodeInfoList);
    int ret = oEchoServer.RunPaxos();
    if (ret != 0)
    {
        return -1;
    }

    printf("echo server start, ip %s port %d\n", oMyNode.GetIP().c_str(), oMyNode.GetPort());
    
    /*通过Echo不断发起请求*/
    string sEchoReqValue;
    while (true)
    {
        printf("\nplease input: <echo req value>\n");
        getline(cin, sEchoReqValue);
        string sEchoRespValue;
        ret = oEchoServer.Echo(sEchoReqValue, sEchoRespValue);
        if (ret != 0)
        {
            printf("Echo fail, ret %d\n", ret);
        }
        else
        {
            printf("echo resp value %s\n", sEchoRespValue.c_str());
        }
    }

    return 0;
}
```

# 运行实例

```
int PhxEchoServer :: RunPaxos()
{
    Options oOptions;

    /*初始化@oOptions.sLogStoragePath*/
    int ret = MakeLogStoragePath(oOptions.sLogStoragePath);
    if (ret != 0)
    {
        return ret;
    }

    //this groupcount means run paxos group count.
    //every paxos group is independent, there are no any communicate between any 2 paxos group.
    oOptions.iGroupCount = 1;

    oOptions.oMyNode = m_oMyNode;
    oOptions.vecNodeInfoList = m_vecNodeList;

    GroupSMInfo oSMInfo;
    /*每一个paxos group都有一个唯一的group index*/
    oSMInfo.iGroupIdx = 0;
    //one paxos group can have multi state machine.
    /*每一个paxos group都可以拥有多个状态机，统一存放在GroupSMInfo::vecSMList中*/
    oSMInfo.vecSMList.push_back(&m_oEchoSM);
    /*每一个paxos group的状态机集合以paxos group为单位存放于Options::vecGroupSMInfoList中*/
    oOptions.vecGroupSMInfoList.push_back(oSMInfo);

    //use logger_google to print log
    LogFunc pLogFunc;
    ret = LoggerGoogle :: GetLogger("phxecho", "./log", 3, pLogFunc);
    if (ret != 0)
    {
        printf("get logger_google fail, ret %d\n", ret);
        return ret;
    }

    //set logger
    oOptions.pLogFunc = pLogFunc;

    ret = Node::RunNode(oOptions, m_poPaxosNode);
    if (ret != 0)
    {
        printf("run paxos fail, ret %d\n", ret);
        return ret;
    }

    printf("run paxos ok\n");
    return 0;
}

int PhxEchoServer :: MakeLogStoragePath(std::string & sLogStoragePath)
{
    char sTmp[128] = {0};
    snprintf(sTmp, sizeof(sTmp), "./logpath_%s_%d", m_oMyNode.GetIP().c_str(), m_oMyNode.GetPort());

    /*在当前目录下创建名为logpath_{IP}_{PORT}的目录*/
    sLogStoragePath = string(sTmp);
    if (access(sLogStoragePath.c_str(), F_OK) == -1)
    {
        if (mkdir(sLogStoragePath.c_str(), S_IRWXU | S_IRWXG | S_IROTH | S_IXOTH) == -1)
        {       
            printf("Create dir fail, path %s\n", sLogStoragePath.c_str());
            return -1;
        }       
    }

    return 0;
}
```

# 发起请求

```
int PhxEchoServer :: Echo(const std::string & sEchoReqValue, std::string & sEchoRespValue)
{
    SMCtx oCtx;
    PhxEchoSMCtx oEchoSMCtx;
    //smid must same to PhxEchoSM.SMID().
    oCtx.m_iSMID = 1;
    oCtx.m_pCtx = (void *)&oEchoSMCtx;

    uint64_t llInstanceID = 0;
    int ret = m_poPaxosNode->Propose(0, sEchoReqValue, llInstanceID, &oCtx);
    if (ret != 0)
    {
        printf("paxos propose fail, ret %d\n", ret);
        return ret;
    }

    if (oEchoSMCtx.iExecuteRet != 0)
    {
        printf("echo sm excute fail, excuteret %d\n", oEchoSMCtx.iExecuteRet);
        return oEchoSMCtx.iExecuteRet;
    }

    sEchoRespValue = oEchoSMCtx.sEchoRespValue.c_str();

    return 0;
}
```








