# 提纲
[toc]

# Committer类基础
## Committer类定义

```
class Committer
{
private:
    /*以下4个域都来直接自于Instance类中的域*/
    Config * m_poConfig;
    CommitCtx * m_poCommitCtx;
    IOLoop * m_poIOLoop;
    SMFac * m_poSMFac;

    WaitLock m_oWaitLock;
    int m_iTimeoutMs;

    uint64_t m_llLastLogTime;
};
```

## Committer类构造函数

```
Committer :: Committer(Config * poConfig, CommitCtx * poCommitCtx, IOLoop * poIOLoop, SMFac * poSMFac)
    : m_poConfig(poConfig), m_poCommitCtx(poCommitCtx), m_poIOLoop(poIOLoop), m_poSMFac(poSMFac), m_iTimeoutMs(-1)
{
    m_llLastLogTime = Time::GetSteadyClockMS();
}
```

## CommitCtx类定义

```
class CommitCtx
{
public:
    CommitCtx(Config * poConfig);
    ~CommitCtx();

    void NewCommit(std::string * psValue, SMCtx * poSMCtx, const int iTimeoutMs);
    
    const bool IsNewCommit() const;

    std::string & GetCommitValue();

    void StartCommit(const uint64_t llInstanceID);

    bool IsMyCommit(const uint64_t llInstanceID, const std::string & sLearnValue, SMCtx *& poSMCtx);

public:
    void SetResult(const int iCommitRet, const uint64_t llInstanceID, const std::string & sLearnValue);

    void SetResultOnlyRet(const int iCommitRet);

    int GetResult(uint64_t & llSuccInstanceID);

public:
    const int GetTimeoutMs() const;

private:
    Config * m_poConfig;

    uint64_t m_llInstanceID;
    int m_iCommitRet;
    bool m_bIsCommitEnd;
    int m_iTimeoutMs;

    std::string * m_psValue;
    SMCtx * m_poSMCtx;
    SerialLock m_oSerialLock;
};
```

## CommitCtx类构造函数

```
CommitCtx :: CommitCtx(Config * poConfig)
    : m_poConfig(poConfig)
{
    /*借助于NewCommit来初始化CommitCtx中的各个域*/
    NewCommit(nullptr, nullptr, 0);
}
```


# Committer类方法
## Committer

```
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
    /*添加一个空消息（nullptr）到IOLoop中（IOLoop拥有独立的线程），在IOLoop中该消息会被调度处理*/
    m_poIOLoop->AddNotify();

    /* 等待提交（Commit）的结果（会同步等待，直到该提案执行完毕），
     * 如果成功提交，则返回对应的InstanceID
     */
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

void CommitCtx :: StartCommit(const uint64_t llInstanceID)
{
    m_oSerialLock.Lock();
    /*设置Instance ID*/
    m_llInstanceID = llInstanceID;
    m_oSerialLock.UnLock();
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

# CommitCtx类方法
## 检查本次学习到的决议是否就是本实例的Committer提交的？

```
bool CommitCtx :: IsMyCommit(const uint64_t llInstanceID, const std::string & sLearnValue,  SMCtx *& poSMCtx)
{
    m_oSerialLock.Lock();

    bool bIsMyCommit = false;

    /*本实例当前的提交尚未结束，且InstanceID和提案内容和决议一致*/
    if ((!m_bIsCommitEnd) && (m_llInstanceID == llInstanceID))
    {
        bIsMyCommit = (sLearnValue == (*m_psValue));
    }

    if (bIsMyCommit)
    {
        poSMCtx = m_poSMCtx;
    }

    m_oSerialLock.UnLock();

    return bIsMyCommit;
}
```

## 设置当前Committer提交的提案执行结果

```
void CommitCtx :: SetResult(
        const int iCommitRet, 
        const uint64_t llInstanceID, 
        const std::string & sLearnValue)
{
    m_oSerialLock.Lock();

    /*设置执行结果的前提是：当前提案提交结果尚未被设置，且InstanceID匹配*/
    if (m_bIsCommitEnd || (m_llInstanceID != llInstanceID))
    {
        m_oSerialLock.UnLock();
        return;
    }

    /*本次提交的结果*/
    m_iCommitRet = iCommitRet;

    if (m_iCommitRet == 0)
    {
        /*决议的结果和提案的value不匹配（冲突）*/
        if ((*m_psValue) != sLearnValue)
        {
            m_iCommitRet = PaxosTryCommitRet_Conflict;
        }
    }

    /*本次提案提交结束*/
    m_bIsCommitEnd = true;
    m_psValue = nullptr;

    /* 唤醒等待提交结果的Committer（见Committer::NewValueGetIDNoRetry ->
     * CommitCtx::GetResult）
     */
    m_oSerialLock.Interupt();
    m_oSerialLock.UnLock();
}
```



