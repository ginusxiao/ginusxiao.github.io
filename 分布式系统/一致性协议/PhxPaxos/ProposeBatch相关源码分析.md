# 提纲
[toc]

# ProposeBatch类定义

```
/*ProposerBatch在其独立的线程中运行*/
class ProposeBatch
{
private:
    const int m_iMyGroupIdx;
    Node * m_poPaxosNode;
    NotifierPool * m_poNotifierPool;

    std::mutex m_oMutex;
    std::condition_variable m_oCond;
    /*等待提交的proposal队列*/
    std::queue<PendingProposal> m_oQueue;
    bool m_bIsEnd;
    bool m_bIsStarted;
    /*等待批量提交的队列中value的总大小*/
    int m_iNowQueueValueSize;

private:
    /*批量提交控制相关*/
    int m_iBatchCount;
    int m_iBatchDelayTimeMs;
    int m_iBatchMaxSize;

    std::thread * m_poThread;
};
```

# ProposeBatch构造函数

```
ProposeBatch :: ProposeBatch(const int iGroupIdx, Node * poPaxosNode, NotifierPool * poNotifierPool)
    : m_iMyGroupIdx(iGroupIdx), m_poPaxosNode(poPaxosNode), 
    m_poNotifierPool(poNotifierPool), m_bIsEnd(false), m_bIsStarted(false), m_iNowQueueValueSize(0),
    m_iBatchCount(5), m_iBatchDelayTimeMs(20), m_iBatchMaxSize(500 * 1024),
    m_poThread(nullptr)
{
}
```

# ProposerBatch运行

```
void ProposeBatch :: Start()
{
    /*启动线程，线程执行函数为ProposerBatch::Run*/
    m_poThread = new std::thread(&ProposeBatch::Run, this);
    assert(m_poThread != nullptr);
}

void ProposeBatch :: Run()
{
    m_bIsStarted = true;
    //daemon thread for very low qps.
    TimeStat oTimeStat;
    while (true)
    {
        std::unique_lock<std::mutex> oLock(m_oMutex);

        if (m_bIsEnd)
        {
            break;
        }

        oTimeStat.Point();

        vector<PendingProposal> vecRequest;
        /*从等待队列中拿去请求，并存放到@vecRequest中*/
        PluckProposal(vecRequest);

        oLock.unlock();
        /*批量Propose*/
        DoPropose(vecRequest);
        oLock.lock();

        int iPassTime = oTimeStat.Point();
        int iNeedSleepTime = iPassTime < m_iBatchDelayTimeMs ?
            m_iBatchDelayTimeMs - iPassTime : 0;

        /* 如果等待队列中的请求数目或者请求总大小达到阈值，或者等待队列中存在至少
         * 一个请求等待的时间超过预设定的最大延迟处理时间，则继续处理（无需休眠）
         */
        if (NeedBatch())
        {
            iNeedSleepTime = 0;
        }

        if (iNeedSleepTime > 0)
        {
            /*休眠一段时间*/
            m_oCond.wait_for(oLock, std::chrono::milliseconds(iNeedSleepTime));
        }

        //PLG1Debug("one loop, sleep time %dms", iNeedSleepTime);
    }

    //notify all waiting thread.
    std::unique_lock<std::mutex> oLock(m_oMutex);
    while (!m_oQueue.empty())
    {
        PendingProposal & oPendingProposal = m_oQueue.front();
        oPendingProposal.poNotifier->SendNotify(Paxos_SystemError);
        m_oQueue.pop();
    }

    PLG1Head("Ended.");
}

void ProposeBatch :: PluckProposal(std::vector<PendingProposal> & vecRequest)
{
    int iPluckCount = 0;
    int iPluckSize = 0;

    uint64_t llNowTime = Time::GetSteadyClockMS();

    while (!m_oQueue.empty())
    {
        /*从@m_oQueue队列中拿出请求并存放到@vecRequest中*/
        PendingProposal & oPendingProposal = m_oQueue.front();
        vecRequest.push_back(oPendingProposal);

        iPluckCount++;
        iPluckSize += oPendingProposal.psValue->size();
        m_iNowQueueValueSize -= oPendingProposal.psValue->size();

        {
            int iProposalWaitTime = llNowTime > oPendingProposal.llAbsEnqueueTime ?
                llNowTime - oPendingProposal.llAbsEnqueueTime : 0;
            BP->GetCommiterBP()->BatchProposeWaitTimeMs(iProposalWaitTime);
        }

        m_oQueue.pop();

        /*如果拿取的请求数目达到@m_iBatchCount或者请求的总大小大于@m_iBatchMaxSize，则停止*/
        if (iPluckCount >= m_iBatchCount
                || iPluckSize >= m_iBatchMaxSize)
        {
            break;
        }
    }

    if (vecRequest.size() > 0)
    {
        PLG1Debug("pluck %zu request", vecRequest.size());
    }
}

const bool ProposeBatch :: NeedBatch()
{
    /*等待队列中的请求数目或者请求总大小超过预设阈值，则继续进行批量Propose*/
    if ((int)m_oQueue.size() >= m_iBatchCount
            || m_iNowQueueValueSize >= m_iBatchMaxSize)
    {
        return true;
    }
    else if (m_oQueue.size() > 0)
    {
        /* 等待队列中队首的请求等待时间是否超过m_iBatchDelayTimeMs，如果超过了，
         * 则继续进行批量Propose
         */
        PendingProposal & oPendingProposal = m_oQueue.front();
        uint64_t llNowTime = Time::GetSteadyClockMS();
        int iProposalPassTime = llNowTime > oPendingProposal.llAbsEnqueueTime ?
            llNowTime - oPendingProposal.llAbsEnqueueTime : 0;
        if (iProposalPassTime > m_iBatchDelayTimeMs)
        {
            return true;
        }
    }

    return false;
}
```





