# 提纲
[toc]

# IOLoop类基础
## IOLoop类定义

```
/*IOLoop类有自己单独的线程*/
class IOLoop : public Thread
{
private:
    bool m_bIsEnd;
    bool m_bIsStart;
    Timer m_oTimer;
    std::map<uint32_t, bool> m_mapTimerIDExist;

    Queue<std::string *> m_oMessageQueue;
    std::queue<PaxosMsg> m_oRetryQueue;

    int m_iQueueMemSize;

    Config * m_poConfig;
    Instance * m_poInstance;
};
```

## IOLoop类构造函数

```
IOLoop :: IOLoop(Config * poConfig, Instance * poInstance)
    : m_poConfig(poConfig), m_poInstance(poInstance)
{
    m_bIsEnd = false;
    m_bIsStart = false;

    m_iQueueMemSize = 0;
}
```

# IOLoop类方法
## IOLoop运行

```
void IOLoop :: run()
{
    m_bIsEnd = false;
    m_bIsStart = true;
    while(true)
    {
        BP->GetIOLoopBP()->OneLoop();

        int iNextTimeout = 1000;
        
        /*处理定时器，并且返回下一个即将超时的定时器还有多久才会超时，保存于@iNextTimeout中*/
        DealwithTimeout(iNextTimeout);

        //PLGHead("nexttimeout %d", iNextTimeout);

        /*尝试处理至多一个消息*/
        OneLoop(iNextTimeout);

        if (m_bIsEnd)
        {
            PLGHead("IOLoop [End]");
            break;
        }
    }
}

void IOLoop :: DealwithTimeout(int & iNextTimeout)
{
    bool bHasTimeout = true;

    while(bHasTimeout)
    {
        uint32_t iTimerID = 0;
        int iType = 0;
        /*检查下一个定时器是否超时*/
        bHasTimeout = m_oTimer.PopTimeout(iTimerID, iType);

        if (bHasTimeout)
        {
            /*处理下一个超时的定时器*/
            DealwithTimeoutOne(iTimerID, iType);

            /*紧接着的下一个定时器还有多久超时*/
            iNextTimeout = m_oTimer.GetNextTimeout();
            if (iNextTimeout != 0)
            {
                break;
            }
        }
    }
}

void IOLoop :: DealwithTimeoutOne(const uint32_t iTimerID, const int iType)
{
    auto it = m_mapTimerIDExist.find(iTimerID);
    if (it == end(m_mapTimerIDExist))
    {
        //PLGErr("Timeout aready remove!, timerid %u iType %d", iTimerID, iType);
        return;
    }

    /*从@m_mapTimerIDExist中删除之*/
    m_mapTimerIDExist.erase(it);

    m_poInstance->OnTimeout(iTimerID, iType);
}

void IOLoop :: OneLoop(const int iTimeoutMs)
{
    std::string * psMessage = nullptr;

    m_oMessageQueue.lock();
    /*从@m_oMessageQueue中获取下一个待处理的消息，如果没有待处理的消息，则等待至多@iTimeoutMs的时间*/
    bool bSucc = m_oMessageQueue.peek(psMessage, iTimeoutMs);
    
    if (!bSucc)
    {
        m_oMessageQueue.unlock();
    }
    else
    {
        m_oMessageQueue.pop();
        m_oMessageQueue.unlock();

        /*处理该消息*/
        if (psMessage != nullptr && psMessage->size() > 0)
        {
            m_iQueueMemSize -= psMessage->size();
            m_poInstance->OnReceive(*psMessage);
        }

        delete psMessage;

        BP->GetIOLoopBP()->OutQueueMsg();
    }

    /*尝试处理重试请求*/
    DealWithRetry();

    //must put on here
    //because addtimer on this funciton
    /*检查是否存在由Committer提交的提议需要处理*/
    m_poInstance->CheckNewValue();
}

void IOLoop :: DealWithRetry()
{
    /*重试队列为空*/
    if (m_oRetryQueue.empty())
    {
        return;
    }
    
    bool bHaveRetryOne = false;
    while (!m_oRetryQueue.empty())
    {
        PaxosMsg & oPaxosMsg = m_oRetryQueue.front();
        if (oPaxosMsg.instanceid() > m_poInstance->GetNowInstanceID() + 1)
        {
            break;
        }
        else if (oPaxosMsg.instanceid() == m_poInstance->GetNowInstanceID() + 1)
        {
            /* 这里要和后面一个else if联系起来看，重试了m_poInstance->GetNowInstanceID()的
             * 请求之后，才能重试m_poInstance->GetNowInstanceID() + 1的请求
             */
            //only after retry i == now_i, than we can retry i + 1.
            if (bHaveRetryOne)
            {
                BP->GetIOLoopBP()->DealWithRetryMsg();
                PLGDebug("retry msg (i+1). instanceid %lu", oPaxosMsg.instanceid());
                m_poInstance->OnReceivePaxosMsg(oPaxosMsg, true);
            }
            else
            {
                break;
            }
        }
        else if (oPaxosMsg.instanceid() == m_poInstance->GetNowInstanceID())
        {
            BP->GetIOLoopBP()->DealWithRetryMsg();
            PLGDebug("retry msg. instanceid %lu", oPaxosMsg.instanceid());
            m_poInstance->OnReceivePaxosMsg(oPaxosMsg);
            bHaveRetryOne = true;
        }

        m_oRetryQueue.pop();
    }
}
```

## 向IOLoop中添加一个通知

```
void IOLoop :: AddNotify()
{
    m_oMessageQueue.lock();
    /*添加一个空的消息nullptr*/
    m_oMessageQueue.add(nullptr);
    m_oMessageQueue.unlock();
}
```
IOLoop一直在不断的处理到达的消息，但是如果消息队列中待处理的请求为空的话，则IOLoop会等待超时时间（该时间不是固定值，与当前定时器状态等有关），然后才检查消息队列中是否有新的消息要处理，这样没法实现在IOLoop中立即处理某个事件这样的情况，而通过发送一个空消息的话，无需等待即可立即处理。

## 向IOLoop中添加一个消息

```
int IOLoop :: AddMessage(const char * pcMessage, const int iMessageLen)
{
    m_oMessageQueue.lock();

    BP->GetIOLoopBP()->EnqueueMsg();

    /*消息队列已满，则不能添加*/
    if ((int)m_oMessageQueue.size() > QUEUE_MAXLENGTH)
    {
        BP->GetIOLoopBP()->EnqueueMsgRejectByFullQueue();

        PLGErr("Queue full, skip msg");
        m_oMessageQueue.unlock();
        return -2;
    }

    /*消息队列中消息长度超过阈值，则不能添加*/
    if (m_iQueueMemSize > MAX_QUEUE_MEM_SIZE)
    {
        PLErr("queue memsize %d too large, can't enqueue", m_iQueueMemSize);
        m_oMessageQueue.unlock();
        return -2;
    }
    
    /*添加到消息队列中*/
    m_oMessageQueue.add(new string(pcMessage, iMessageLen));
    /*更新消息队列中消息的总长度*/
    m_iQueueMemSize += iMessageLen;

    m_oMessageQueue.unlock();

    return 0;
}
```

## 向IOLoop中添加一个重试消息

```
int IOLoop :: AddRetryPaxosMsg(const PaxosMsg & oPaxosMsg)
{
    BP->GetIOLoopBP()->EnqueueRetryMsg();

    /*如果重试队列已满，则先pop出来一个消息*/
    if (m_oRetryQueue.size() > RETRY_QUEUE_MAX_LEN)
    {
        BP->GetIOLoopBP()->EnqueueRetryMsgRejectByFullQueue();
        m_oRetryQueue.pop();
    }
    
    /*添加到重试队列中*/
    m_oRetryQueue.push(oPaxosMsg);
    return 0;
}
```


## 向IOLoop中添加定时器

```
bool IOLoop :: AddTimer(const int iTimeout, const int iType, uint32_t & iTimerID)
{
    if (iTimeout == -1)
    {
        return true;
    }
    
    /* 计算绝对超时时间，并将定时器添加到@m_oTimer中，返回本次添加的定时器对应的ID，
     * 记录在@iTimerID中
     */
    uint64_t llAbsTime = Time::GetSteadyClockMS() + iTimeout;
    m_oTimer.AddTimerWithType(llAbsTime, iType, iTimerID);
    
    /*将@iTimerID添加到@m_mapTimerIDExist中*/
    m_mapTimerIDExist[iTimerID] = true;

    return true;
}
```

## 从IOLoop中删除指定ID的定时器

```
void IOLoop :: RemoveTimer(uint32_t & iTimerID)
{
    /*从@m_mapTimerIDExist中删除指定的定时器*/
    auto it = m_mapTimerIDExist.find(iTimerID);
    if (it != end(m_mapTimerIDExist))
    {
        m_mapTimerIDExist.erase(it);
    }

    iTimerID = 0;
}
```








