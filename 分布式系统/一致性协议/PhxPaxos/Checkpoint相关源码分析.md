# 提纲
[toc]

# Checkpoint基础
## 术语说明
Instance：实例

InstanceID：实例编号

min chosen InstanceID：已经执行了checkpoint，但是仍然被保留（尚未被Cleaner删除）的最小的InstanceID

max chosen InstanceID：已经被运用到状态机中的最大InstanceID

## PhxPaxos中的checkpoint是什么
PhxPaxos中的checkpoint用于用于删除Paxos log，同时满足其他节点关于业务状态机的数据拷贝需求。

## 为什么需要checkpoint
可以参考[状态机Checkpoint详解](https://github.com/Tencent/phxpaxos/wiki/%E7%8A%B6%E6%80%81%E6%9C%BACheckpoint%E8%AF%A6%E8%A7%A3)

PhxPaxos通过运行Paxos协议会产生一系列的决议，每产生一个决议，都会记录在Paxos log中（最终存放的介质是磁盘），这些决议作为业务状态机的输入来实现业务状态变迁。业务状态是有限的，但是作为业务状态机输入的决议是无限的，而磁盘空间是有限的，所以Paxos log中的记录不能长期保存，必须想办法删除一部分记录。

理论上来说，只要记录下来已经运用到业务状态机中的决议的最大InstanceID，那么在重新启动的时候，从这个最大的InstanceID进行有序重放就可以将业务状态机更新到最新状态，所有小于这个最大InstanceID的Instance对应的日志就可以删除了。但是因为Paxos是允许少于多数派的机器挂掉的，这个挂掉可能是机器永远离线，一般都会采用启用一台新的机器来代替，这台新的机器必须从其它机器上学习从InstanceID 0开始的决议并进行重放，问题是，其它机器上的决议可能已经被删除了，那么该怎么办呢？

为了避免新加入的机器从其他机器上从InstanceID 0开始学习并重放决议，可以尝试从其他机器上拷贝业务状态机的数据到新加入的机器上，但是其他机器上的业务状态机可能是无时不刻不再写入的，拷贝一个正在写入的数据，结果是不可预期的，那么停机进行拷贝？这影响到业务了，也不是一个好方法。但是如果有一个关于业务状态机的镜像状态机，然后在需要拷贝的时候先停止该镜像状态机的写入，然后从该镜像状态机拷贝就可以了。有了这个镜像状态机，就可以堂而皇之地删除Paxos log了。

一个状态机能构建出一份状态数据，那么一个镜像状态机就可以同样构建出一份镜像状态数据了。 
![image](http://mmbiz.qpic.cn/mmbiz/YriaiaJPb26VMYLQYjgJZMwLprxbicgVa9ia5d2KmJbwckVH4zEKxHf4glZGn23FQYBTfJRPSv7BNVMEmBLK2tSAxQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1)

如上图，用两个状态转移完全一致的状态机，分别管理不同的状态数据，通过灌入相同的Paxos log，最终出来的状态数据是完全一致的。

这就是checkpoint，它用于删除Paxos log，同时满足其他节点关于业务状态机的数据拷贝需求。

## PhxPaxos中的checkpoint提供了哪些功能
（1）构建镜像状态机（藉由Replayer来实现）；

（2）删除Paxos log（藉由Cleaner来实现）；

（3）满足其他节点关于状态机的数据拷贝需求（藉由Learner和CheckpointSender来实现）；


# Checkpoint相关类基础
## CheckpointMgr类基础
### CheckpointMgr类定义
```
class CheckpointMgr
{
private:
    /*以下3个域都直接来自于Instance类中的域*/
    Config * m_poConfig;
    LogStorage * m_poLogStorage;
    SMFac * m_poSMFac;
    
    Replayer m_oReplayer;
    Cleaner m_oCleaner;

    /*已经执行了checkpoint，但是仍然被保留（尚未被Cleaner删除）的最小的InstanceID*/
    uint64_t m_llMinChosenInstanceID;
    /*已经被运用到状态机中的最大InstanceID*/
    uint64_t m_llMaxChosenInstanceID;

private:
    bool m_bInAskforCheckpointMode;
    std::set<nodeid_t> m_setNeedAsk;
    uint64_t m_llLastAskforCheckpointTime;

    bool m_bUseCheckpointReplayer;
};
```

### CheckpointMgr类构造函数

```
CheckpointMgr :: CheckpointMgr(
        Config * poConfig,
        SMFac * poSMFac, 
        LogStorage * poLogStorage,
        const bool bUseCheckpointReplayer) 
    : m_poConfig(poConfig),
    m_poLogStorage(poLogStorage),
    m_poSMFac(poSMFac),
    m_oReplayer(poConfig, poSMFac, poLogStorage, this),
    m_oCleaner(poConfig, poSMFac, poLogStorage, this),
    m_llMinChosenInstanceID(0),
    m_llMaxChosenInstanceID(0),
    m_bInAskforCheckpointMode(false),
    m_bUseCheckpointReplayer(bUseCheckpointReplayer)
{
    m_llLastAskforCheckpointTime = 0;
}
```

## Cleaner类基础
PhxPaxos中Cleaner在一个单独的线程中执行，并为之提供了暂停运行（Pause）、恢复运行（Continue）和停止运行（Stop）等接口，同时供PhxPaxos内部和应用程序使用。Cleaner的作用就是删除那些已经执行过checkpoint的Instance，但要确保保留一定数目执行过checkpoint的Instance，在删除的过程中也要控制删除的速度，避免对正常IO造成影响。

### Cleaner类定义

```
class Cleaner : public Thread
{
private:
    /*以下4个域都直接来自于Instance类中的域*/
    Config * m_poConfig;
    SMFac * m_poSMFac;
    LogStorage * m_poLogStorage;
    CheckpointMgr * m_poCheckpointMgr;

    uint64_t m_llLastSave;

    /*控制Cleaner的启停*/
    bool m_bCanrun;
    bool m_bIsPaused;
    bool m_bIsEnd;
    bool m_bIsStart;

    /*保留这么多的Instance不被清理*/
    uint64_t m_llHoldCount;
};
```


### Cleaner类构造函数

```
Cleaner :: Cleaner(
    Config * poConfig, 
    SMFac * poSMFac, 
    LogStorage * poLogStorage, 
    CheckpointMgr * poCheckpointMgr)
    : m_poConfig(poConfig), 
    m_poSMFac(poSMFac), 
    m_poLogStorage(poLogStorage), 
    m_poCheckpointMgr(poCheckpointMgr),
    m_llLastSave(0),
    m_bCanrun(false),
    m_bIsPaused(true),
    m_bIsEnd(false),
    m_bIsStart(false),
    m_llHoldCount(CAN_DELETE_DELTA)
{
}
```

## Replayer类基础
PhxPaxos中Replayer在一个单独的线程中执行，并为之提供了暂停运行（Pause）、恢复运行（Continue）和停止运行（Stop）等接口，同时供PhxPaxos内部和应用程序使用。Replayer的作用就是用于异步的构建镜像状态机，来实现checkpoint的目的。

### Replayer类定义

```
/*Replayer类有其自己独立的线程*/
class Replayer : public Thread
{
private:
    Config * m_poConfig;
    SMFac * m_poSMFac;
    PaxosLog m_oPaxosLog;
    /*反向索引到CheckpointMgr*/
    CheckpointMgr * m_poCheckpointMgr;

    bool m_bCanrun;
    bool m_bIsPaused;
    bool m_bIsEnd;
};
```

### Replayer类构造函数

```
Replayer :: Replayer(
    Config * poConfig, 
    SMFac * poSMFac, 
    LogStorage * poLogStorage, 
    CheckpointMgr * poCheckpointMgr)
    : m_poConfig(poConfig), 
    m_poSMFac(poSMFac), 
    m_oPaxosLog(poLogStorage), 
    m_poCheckpointMgr(poCheckpointMgr),
    m_bCanrun(false),
    m_bIsPaused(true),
    m_bIsEnd(false)
{
}
```

## CheckpointSender类基础
CheckpointSender用于发送本节点的镜像状态机中的数据到其它节点，它在自己独立的线程中运行，从源码来看，CheckpointSender线程是按需启用的（这区别于Replayer和Cleaner这类常驻线程），需要发送镜像数据的时候的时候就启动一个新的CheckpointSender线程，当镜像数据发送完毕的时候，就停止该CheckpointSender的线程。

### CheckpointSender类定义

```
class CheckpointSender : public Thread
{
private:
    nodeid_t m_iSendNodeID;

    Config * m_poConfig;
    Learner * m_poLearner;
    SMFac * m_poSMFac;
    CheckpointMgr * m_poCheckpointMgr;

    /*是否需要停止*/
    bool m_bIsEnd;
    /*是否已经停止*/
    bool m_bIsEnded;
    /*是否已经启动*/
    bool m_bIsStarted;

private:
    uint64_t m_llUUID;
    uint64_t m_llSequence;

private:
    uint64_t m_llAckSequence;
    uint64_t m_llAbsLastAckTime;

private:
    char m_sTmpBuffer[1048576];
    
    std::map<std::string, bool> m_mapAlreadySendedFile;
};
```

### CheckpointSender类构造函数

```
CheckpointSender :: CheckpointSender(
    const nodeid_t iSendNodeID,
    Config * poConfig, 
    Learner * poLearner,
    SMFac * poSMFac, 
    CheckpointMgr * poCheckpointMgr) :
    m_iSendNodeID(iSendNodeID),
    m_poConfig(poConfig),
    m_poLearner(poLearner),
    m_poSMFac(poSMFac),
    m_poCheckpointMgr(poCheckpointMgr)
{
    m_bIsEnded = false;
    m_bIsEnd = false;
    m_bIsStarted = false;
    m_llUUID = (m_poConfig->GetMyNodeID() ^ m_poLearner->GetInstanceID()) + OtherUtils::FastRand();
    m_llSequence = 0;

    m_llAckSequence = 0;
    m_llAbsLastAckTime = 0;
}
```

## CheckpointReceiver类基础
CheckpointSender用于接收从其它节点发送过来的镜像状态机数据。

### CheckpointReceiver类定义
```
class CheckpointReceiver
{
private:
    Config * m_poConfig;
    LogStorage * m_poLogStorage;

private:
    nodeid_t m_iSenderNodeID; 
    uint64_t m_llUUID;
    uint64_t m_llSequence;

private:
    std::map<std::string, bool> m_mapHasInitDir;
};
```

### CheckpointReceiver类构造函数
```
CheckpointReceiver :: CheckpointReceiver(Config * poConfig, LogStorage * poLogStorage) :
    m_poConfig(poConfig), m_poLogStorage(poLogStorage)
{
    Reset();
}

void CheckpointReceiver :: Reset()
{
    m_mapHasInitDir.clear();
    
    m_iSenderNodeID = nullnode;
    m_llUUID = 0;
    m_llSequence = 0;
}
```

# CheckPointMgr方法
## CheckpointMgr初始化

```
int CheckpointMgr :: Init()
{
    /* 实际调用MultiDatabase::GetMinChosenInstanceID，获取该paxos group
     * 对应的最小chosen InstanceID，那么最小chosen InstanceID表示啥呢???
     */
    int ret = m_poLogStorage->GetMinChosenInstanceID(m_poConfig->GetMyGroupIdx(), m_llMinChosenInstanceID);
    if (ret != 0)
    {
        return ret;
    }

    ret = m_oCleaner.FixMinChosenInstanceID(m_llMinChosenInstanceID);
    if (ret != 0)
    {
        return ret;
    }
    
    return 0;
}
```

## CheckpointMgr启动

```
void CheckpointMgr :: Start()
{
    if (m_bUseCheckpointReplayer)
    {
        /*启动replayer*/
        m_oReplayer.start();
    }
    
    /*启动cleaner*/
    m_oCleaner.start();
}
```

## 更新CheckpointMgr中记录的最小chosen InstanceID缓存

```
void CheckpointMgr :: SetMinChosenInstanceIDCache(const uint64_t llMinChosenInstanceID)
{
    /*更新CheckpointMgr中记录的最小chosen InstanceID缓存*/
    m_llMinChosenInstanceID = llMinChosenInstanceID;
}
```

## 更新CheckpointMgr中记录的最小chosen InstanceID缓存，同时持久化到数据库

```
int CheckpointMgr :: SetMinChosenInstanceID(const uint64_t llMinChosenInstanceID)
{ 
    WriteOptions oWriteOptions;
    oWriteOptions.bSync = true;

    /*持久化到数据库*/
    int ret = m_poLogStorage->SetMinChosenInstanceID(oWriteOptions, m_poConfig->GetMyGroupIdx(), llMinChosenInstanceID);
    if (ret != 0)
    {
        return ret;
    }

    /*更新CheckpointMgr中记录的最小chosen InstanceID缓存*/
    m_llMinChosenInstanceID = llMinChosenInstanceID;

    return 0;
}
```

## AskForCheckpoint的前期检查工作

```
int CheckpointMgr :: PrepareForAskforCheckpoint(const nodeid_t iSendNodeID)
{
    /*将@iSendNodeID添加到@m_setNeedAsk集合*/
    if (m_setNeedAsk.find(iSendNodeID) == m_setNeedAsk.end())
    {
        m_setNeedAsk.insert(iSendNodeID);
    }

    /* 当@m_setNeedAsk集合中的第一个节点到达时，设置@m_llLastAskforCheckpointTime，
     * 表示AskForCheckpoint的起始时间
     */
    if (m_llLastAskforCheckpointTime == 0)
    {
        m_llLastAskforCheckpointTime = Time::GetSteadyClockMS();
    }

    uint64_t llNowTime = Time::GetSteadyClockMS();
    if (llNowTime > m_llLastAskforCheckpointTime + 60000)
    {
        /* 如果超过60ms，且@m_setNeedAsk集合中的节点数目未达到多数派，则直接进入
         * AskForCheckpoint模式，执行AskForCheckpoint
         */
        PLGImp("no majority reply, just ask for checkpoint");
    }
    else
    {
        /* 如果在60ms以内，@m_setNeedAsk集合中的节点数目未达到多数派，则继续等待，
         * 直到超过60ms或者多数派节点告知我需要执行AskForCheckpoint
         */
        if ((int)m_setNeedAsk.size() < m_poConfig->GetMajorityCount())
        {
            PLGImp("Need more other tell us need to askforcheckpoint");
            return -2;
        }
    }
    
    /*重置最后一次AskForCheckpoint的时间*/
    m_llLastAskforCheckpointTime = 0;
    /*设置进入AskForCheckpoint模式（在AskForLearn_noop中会清理该状态）*/
    m_bInAskforCheckpointMode = true;

    return 0;
}
```



# Cleaner方法
## Cleaner启动
Cleaner在自己独立的线程中执行，Cleaner启动实际上就是启动线程，线程启动后运行Cleaner::run，参考“Cleaner运行”。

## Cleaner运行
Cleaner运行主体执行以下工作：

（0）首先进入正常状态；

（1）在正常状态下，逐一删除那些已经执行过checkpoint的Instance，但同时要保留一定数目的执行过checkpoint的Instance；

（2）在正常状态下，检查是否接收到暂停指令（Pause），如果是，则暂停工作，直到接收到恢复指令（Continue）或者停止指令（Stop）；

（3）在正常状态下，检查是否接收到停止指令（Stop），如果是，则停止工作，退出线程；

（4）在暂停状态下，如果接收到恢复指令（Continue），则恢复正常运行，切换到正常状态；

（5）在暂停状态下，如果接收到停止指令（Stop），则停止运行；

```
void Cleaner :: run()
{
    m_bIsStart = true;
    /*设置@m_bIsPaused为false，设置@m_bCanrun为true，进入正常状态*/
    Continue();

    //control delete speed to avoid affecting the io too much.
    /*控制删除速度，避免对正常IO的影响*/
    int iDeleteQps = Cleaner_DELETE_QPS;
    int iSleepMs = iDeleteQps > 1000 ? 1 : 1000 / iDeleteQps;
    int iDeleteInterval = iDeleteQps > 1000 ? iDeleteQps / 1000 + 1 : 1; 

    PLGDebug("DeleteQps %d SleepMs %d DeleteInterval %d",
            iDeleteQps, iSleepMs, iDeleteInterval);

    while (true)
    {
        /*如果执行了Cleaner::Stop，则退出循环*/
        if (m_bIsEnd)
        {
            PLGHead("Checkpoint.Cleaner [END]");
            return;
        }
        
        /* 如果设置了暂停运行，则休眠1000ms并检查是否恢复运行或者停止运行，
         * 并根据后续指令做出相应的处理
         */
        if (!m_bCanrun)
        {
            PLGImp("Pausing, sleep");
            m_bIsPaused = true;
            Time::MsSleep(1000);
            continue;
        }

        /*从CheckpointMgr中获取最小chosen InstanceID*/
        uint64_t llInstanceID = m_poCheckpointMgr->GetMinChosenInstanceID();
        /* 从所有StateMachine中获取已经执行checkpoint的最大的InstanceID的下一个InstanceID
         *（或者说尚未执行checkpoint的最小InstanceID）
         */
        uint64_t llCPInstanceID = m_poSMFac->GetCheckpointInstanceID(m_poConfig->GetMyGroupIdx()) + 1;
        /*从CheckpointMgr中获取最大chosen InstanceID*/
        uint64_t llMaxChosenInstanceID = m_poCheckpointMgr->GetMaxChosenInstanceID();

        int iDeleteCount = 0;
        /* 删除那些已经执行过checkpoint的Instance，保留至少m_llHoldCount个Instance，
         * 且不能删除那些尚未执行checkpoint的Instance，且不能超过最大chosen InstanceID
         */
        while ((llInstanceID + m_llHoldCount < llCPInstanceID)
                && (llInstanceID + m_llHoldCount < llMaxChosenInstanceID))
        {
            /*删除@llInstanceID代表的Instance*/
            bool bDeleteRet = DeleteOne(llInstanceID);
            if (bDeleteRet)
            {
                /*删除成功*/
                
                //PLGImp("delete one done, instanceid %lu", llInstanceID);
                llInstanceID++;
                iDeleteCount++;
                /* 删除@iDeleteInterval个Instance之后就休息一会（因为删除过程中会操作LogStorage，
                 * 有可能会影响Paxos协议的正常运行）
                 */
                if (iDeleteCount >= iDeleteInterval)
                {
                    iDeleteCount = 0;
                    Time::MsSleep(iSleepMs);
                }
            }
            else
            {
                PLGDebug("delete system fail, instanceid %lu", llInstanceID);
                break;
            }
        }

        if (llCPInstanceID == 0)
        {
            PLGStatus("sleep a while, max deleted instanceid %lu checkpoint instanceid (no checkpoint) now instanceid %lu",
                    llInstanceID, m_poCheckpointMgr->GetMaxChosenInstanceID());
        }
        else
        {
            PLGStatus("sleep a while, max deleted instanceid %lu checkpoint instanceid %lu now instanceid %lu",
                    llInstanceID, llCPInstanceID, m_poCheckpointMgr->GetMaxChosenInstanceID());
        }

        Time::MsSleep(OtherUtils::FastRand() % 500 + 500);
    }
}
```

## 删除某个Instance

```
bool Cleaner :: DeleteOne(const uint64_t llInstanceID)
{
    WriteOptions oWriteOptions;
    oWriteOptions.bSync = false;

    /*从LogStorage中删除指定group中的指定Instance，调用的是MultiDatabase::Del*/
    int ret = m_poLogStorage->Del(oWriteOptions, m_poConfig->GetMyGroupIdx(), llInstanceID);
    if (ret != 0)
    {
        return false;
    }

    /* 更新了CheckpointMgr中维护的关于最小chosen InstanceID的缓存*/
    m_poCheckpointMgr->SetMinChosenInstanceIDCache(llInstanceID);

    /* 如果自从上次更新最小chosen InstanceID以来，删除了DELETE_SAVE_INTERVAL个
     * Instance，则同时更新LogStorage中记录的最小chosen InstanceID和CheckpointMgr
     * 中维护的最小chosen InstanceID缓存
     */
    if (llInstanceID >= m_llLastSave + DELETE_SAVE_INTERVAL)
    {
        int ret = m_poCheckpointMgr->SetMinChosenInstanceID(llInstanceID + 1);
        if (ret != 0)
        {
            PLGErr("SetMinChosenInstanceID fail, now delete instanceid %lu", llInstanceID);
            return false;
        }

        /*更新@m_llLastSave，其表示最后一次更新到LogStorage中的最小chosen InstanceID*/
        m_llLastSave = llInstanceID;

        PLGImp("delete %d instance done, now minchosen instanceid %lu", 
                DELETE_SAVE_INTERVAL, llInstanceID + 1);
    }

    return true;
}
```


## Cleaner暂停

```
void Cleaner :: Pause()
{
    /*这里没有设置m_bIsPaused，但是在Cleaner::Run中检测到@m_bCanrun被设置的时候就会设置之*/
    m_bCanrun = false;
}
```

## Cleaner恢复运行

```
void Cleaner :: Continue()
{
    m_bIsPaused = false;
    m_bCanrun = true;
}
```

## Cleaner停止

```
void Cleaner :: Stop()
{
    /*设置@m_bIsEnd为true，那么Cleaner::Run就会退出循环，然后调用join，退出线程*/
    m_bIsEnd = true;
    if (m_bIsStart)
    {
        join();
    }
}
```

## 校正最小chosen InstanceID

```
int Cleaner :: FixMinChosenInstanceID(const uint64_t llOldMinChosenInstanceID)
{
    /*获取该group中所有StateMachine已经执行checkpoint的最大的InstanceID*/
    uint64_t llCPInstanceID = m_poSMFac->GetCheckpointInstanceID(m_poConfig->GetMyGroupIdx()) + 1;
    /*参数传递进来的@llOldMinChosenInstanceID表示数据库中记录的最小的chosen InstanceID*/
    uint64_t llFixMinChosenInstanceID = llOldMinChosenInstanceID;
    int ret = 0;

    /*从@llOldMinChosenInstanceID开始，至多检查DELETE_SAVE_INTERVAL个InstanceID*/
    for (uint64_t llInstanceID = llOldMinChosenInstanceID; llInstanceID < llOldMinChosenInstanceID + DELETE_SAVE_INTERVAL;
           llInstanceID++)    
    {
        /*超过已经执行checkpoint的最大InstanceID，则退出*/
        if (llInstanceID >= llCPInstanceID)
        {
            break;
        }
        
        /*检查LogStorage中是否存在该InstanceID相关的信息*/
        std::string sValue;
        ret = m_poLogStorage->Get(m_poConfig->GetMyGroupIdx(), llInstanceID, sValue);
        if (ret != 0 && ret != 1)
        {
            /*读取leveldb或者LogStore失败*/
            return -1;
        }
        else if (ret == 1)
        {
            /* 没有找到关于@llInstanceID这一实例相关的信息，则表明该InstanceID未被使用，
             * 它接下来的那个InstanceID可能是最小chosen InstanceID
             */
            llFixMinChosenInstanceID = llInstanceID + 1;
        }
        else
        {
            /* 在LogStorage中找到了关于@llInstanceID这一实例相关的信息，则表明该InstanceID
             * 正在被使用，它可能是最小chosen InstanceID，但是它之后的InstanceID就一定不是
             */
            break;
        }
    }
    
    /*如果修正后的最小chosen InstanceID比原来在数据库中记录的要大，则更新之*/
    if (llFixMinChosenInstanceID > llOldMinChosenInstanceID)
    {
        /*设置最小chosen InstanceID，更新数据库，同时更新CheckpointMgr中关于该信息的缓存*/
        ret = m_poCheckpointMgr->SetMinChosenInstanceID(llFixMinChosenInstanceID);
        if (ret != 0)
        {
            return ret;
        }
    }

    PLGImp("ok, old minchosen %lu fix minchosen %lu", llOldMinChosenInstanceID, llFixMinChosenInstanceID);

    return 0;
}
```

# Replayer方法
## Replayer启动
Replayer在自己独立的线程中执行，Replayer启动实际上就是启动线程，线程启动后运行Replayer::run，参考“Replayer运行”。

## Replayer运行
Replayer运行主体执行以下工作：

（0）首先进入暂停状态（见Replayer的构造函数Replayer::Replayer()），并检查是否接收到恢复指令（Continue）或者停止指令（Stop）；

（1）在暂停状态下，如果接收到停止指令（Stop），则停止工作，退出线程；

（2）在暂停状态下，如果接收到恢复指令（Continue），则恢复正常运行，切换到正常状态；

（3）在正常状态下，在镜像状态机上逐一在Instance上执行Checkpoint，并检查是否接收到暂停指令（Pause）或者停止指令（Stop）；

（4）在正常状态下，如果接收到暂停指令（Pause），则暂停工作，直到接收到恢复指令（Continue）或者停止指令（Stop）；

（5）在正常状态下，如果接收到停止指令（Stop），则停止工作，退出线程；
```
void Replayer :: run()
{
    PLGHead("Checkpoint.Replayer [START]");
    /*获取当前group中执行了checkpoint的最大InstanceID的下一个ID，本次checkpoint将从这里开始*/
    uint64_t llInstanceID = m_poSMFac->GetCheckpointInstanceID(m_poConfig->GetMyGroupIdx()) + 1;

    while (true)
    {
        if (m_bIsEnd)
        {
            PLGHead("Checkpoint.Replayer [END]");
            return;
        }
        
        if (!m_bCanrun)
        {
            //PLGImp("Pausing, sleep");
            m_bIsPaused = true;
            Time::MsSleep(1000);
            continue;
        }
        
        /*即将checkpoint的Instance尚未运用到状态机中（所以不能运用到镜像状态机中）*/
        if (llInstanceID >= m_poCheckpointMgr->GetMaxChosenInstanceID())
        {
            //PLGImp("now maxchosen instanceid %lu small than excute instanceid %lu, wait", 
                    //m_poCheckpointMgr->GetMaxChosenInstanceID(), llInstanceID);
            Time::MsSleep(1000);
            continue;
        }
        
        bool bPlayRet = PlayOne(llInstanceID);
        if (bPlayRet)
        {
            PLGImp("Play one done, instanceid %lu", llInstanceID);
            llInstanceID++;
        }
        else
        {
            PLGErr("Play one fail, instanceid %lu", llInstanceID);
            Time::MsSleep(500);
        }
    }
}

bool Replayer :: PlayOne(const uint64_t llInstanceID)
{
    AcceptorStateData oState;
    /*读取该group中指定Instance在Acceptor中记录的AcceptorState*/
    int ret = m_oPaxosLog.ReadState(m_poConfig->GetMyGroupIdx(), llInstanceID, oState);
    if (ret != 0)
    {
        return false;
    }

    /*在镜像状态机上运用决议*/
    bool bExecuteRet = m_poSMFac->ExecuteForCheckpoint(
            m_poConfig->GetMyGroupIdx(), llInstanceID, oState.acceptedvalue());
    if (!bExecuteRet)
    {
        PLGErr("Checkpoint sm excute fail, instanceid %lu", llInstanceID);
    }

    return bExecuteRet;
}
```

## 暂停Replayer运行
如果有其它节点的拷贝镜像状态机的需求，那么本节点上的镜像状态机数据就必须停止更新，这是通过Replayer::Pause()实现的。下面展现了Replayer::Pause()的调用关系：

```
/*该接口提供给应用使用，用于从应用层面控制Replayer*/
PNode :: PauseCheckpointReplayer()
    - Replayer::Continue()

/* PhxPaxos内部，通过CheckpointSender来控制Replayer，这是为什么呢，因为
 * CheckpointSender是用于发送镜像状态机的数据的，而在发送镜像状态机的数
 * 据的时候，必须确保镜像状态机是处于暂停状态，同时在发送完镜像状态机的
 * 数据的时候，需要恢复Replayer为运行状态
 */
CheckpointSender :: run()
    - Replayer::Pause()
```


## 恢复Replayer运行
在“Replayer运行”一节中讲到Replayer线程启动之后，首先进入暂停状态，需要外界主动调用“Replayer::Continue()”唤醒之，开始执行Checkpoint。那么Replayer在哪些地方被唤醒呢？下面是Replayer::Continue()的调用关系：

```
/*该接口提供给应用使用，用于从应用层面控制Replayer*/
PNode :: ContinueCheckpointReplayer()
    - Replayer::Continue()

/* PhxPaxos内部，通过CheckpointSender来控制Replayer，这是为什么呢，因为
 * CheckpointSender是用于发送镜像状态机的数据的，而在发送镜像状态机的数
 * 据的时候，必须确保镜像状态机是处于暂停状态，同时在发送完镜像状态机的
 * 数据的时候，需要恢复Replayer为运行状态
 */
CheckpointSender :: run()
    - Replayer::Continue()
```

# CheckpointSender方法
## 其它节点的Learner请求拷贝我的镜像状态机的数据（从镜像状态机中学习）

```
void Learner :: OnAskforCheckpoint(const PaxosMsg & oPaxosMsg)
{
    /* 为当前的请求学习的Learner准备一个单独的CheckpointSender线程，如果
     * 不能为该请求学习的Learner创建新的CheckpointSender线程，则表明当前
     * 的CheckpointSender线程正忙
     */
    CheckpointSender * poCheckpointSender = GetNewCheckpointSender(oPaxosMsg.nodeid());
    if (poCheckpointSender != nullptr)
    {
        /* 为该请求学习的Learner启动新的CheckpointSender线程，进入CheckpointSender::start(),
         * 因为CheckpointSender继承自Thread，所以实际上是调用Thread::start()，执行函数为
         * CheckpointSender :: run()
         */
        poCheckpointSender->start();
        PLGHead("new checkpoint sender started, send to nodeid %lu", oPaxosMsg.nodeid());
    }
    else
    {
        PLGErr("Checkpoint Sender is running");
    }
}

CheckpointSender * Learner :: GetNewCheckpointSender(const nodeid_t iSendNodeID)
{
    /* 如果已经存在一个CheckpointSender线程，那么检查它是否结束了，
     * 如果结束了，就停止该线程，重新为新的请求学习的Learner启动一个
     * CheckpointSender线程；
     * 
     * 如果当前已经存在的CheckpointSender线程尚未结束，就不能为新的
     * 请求学习的Learner启动新的CheckpointSender线程；
     * 
     * 如果当前不存在任何CheckpointSender线程，那么直接为新的请求学习
     * 的Learner启动一个CheckpointSender线程
     */
    if (m_poCheckpointSender != nullptr)
    {
        if (m_poCheckpointSender->IsEnd())
        {
            m_poCheckpointSender->join();
            delete m_poCheckpointSender;
            m_poCheckpointSender = nullptr;
        }
    }

    if (m_poCheckpointSender == nullptr)
    {
        m_poCheckpointSender = new CheckpointSender(iSendNodeID, m_poConfig, this, m_poSMFac, m_poCheckpointMgr);
        return m_poCheckpointSender;
    }

    return nullptr;
}

void CheckpointSender :: run()
{
    /*已经启动*/
    m_bIsStarted = true;
    m_llAbsLastAckTime = Time::GetSteadyClockMS();

    /*在发送之前，必须等待Replayer切换到暂停状态（Pause）*/
    bool bNeedContinue = false;
    while (!m_poCheckpointMgr->GetReplayer()->IsPaused())
    {
        if (m_bIsEnd)
        {
            m_bIsEnded = true;
            return;
        }

        /*发送前需要先暂停Replayer，发送之后需要恢复Replayer*/
        bNeedContinue = true;
        
        /*给Replayer发送暂停指令*/
        m_poCheckpointMgr->GetReplayer()->Pause();
        PLGDebug("wait replayer paused.");
        Time::MsSleep(100);
    }

    /*至此，Replayer已经暂停了，可以发送数据了*/
    
    int ret = LockCheckpoint();
    if (ret == 0)
    {
        /*发送镜像状态机数据*/
        SendCheckpoint();

        UnLockCheckpoint();
    }

    /*发送前已经暂停了Replayer，发送完毕之后就需要恢复之*/
    if (bNeedContinue)
    {
        m_poCheckpointMgr->GetReplayer()->Continue();
    }

    PLGHead("Checkpoint.Sender [END]");
    m_bIsEnded = true;
}

int CheckpointSender :: LockCheckpoint()
{
    std::vector<StateMachine *> vecSMList = m_poSMFac->GetSMList();
    std::vector<StateMachine *> vecLockSMList;
    int ret = 0;
    
    /*逐一遍历每一个状态机*/
    for (auto & poSM : vecSMList)
    {
        /* 将镜像状态机（Checkpoint）相关的文件锁住，在此期间，这些文件不能
         * 被修改、移动或者删除，但是业务状态机相关的文件依然可以被修改
         */
        ret = poSM->LockCheckpointState();
        if (ret != 0)
        {
            break;
        }

        /*记录下来那些成功锁住的状态机*/
        vecLockSMList.push_back(poSM);
    }

    /*如果存在至少一个状态机加锁失败，则解锁那些成功锁住的状态机*/
    if (ret != 0)
    {
        for (auto & poSM : vecLockSMList)
        {
            poSM->UnLockCheckpointState();
        }
    }

    return ret;
}

void CheckpointSender :: UnLockCheckpoint()
{
    std::vector<StateMachine *> vecSMList = m_poSMFac->GetSMList();
    
    /*解锁所有状态机相关的文件*/
    for (auto & poSM : vecSMList)
    {
        poSM->UnLockCheckpointState();
    }
}
```

## 发送镜像状态机数据

```
void CheckpointSender :: SendCheckpoint()
{
    int ret = -1;
    
    /*这里执行2次是为啥？*/
    for (int i = 0; i < 2; i++)
    {
        /*发送一个begin信号给请求学习的Learner，表明发送开始，让对方做好准备*/
        ret = m_poLearner->SendCheckpointBegin(
                m_iSendNodeID, m_llUUID, m_llSequence, 
                m_poSMFac->GetCheckpointInstanceID(m_poConfig->GetMyGroupIdx()));
        if (ret != 0)
        {
            PLGErr("SendCheckpointBegin fail, ret %d", ret);
            return;
        }
    }

    BP->GetCheckpointBP()->SendCheckpointBegin();

    m_llSequence++;

    /*依次发送每一个（镜像）状态机的数据*/
    std::vector<StateMachine *> vecSMList = m_poSMFac->GetSMList();
    for (auto & poSM : vecSMList)
    {
        ret = SendCheckpointFofaSM(poSM);
        if (ret != 0)
        {
            return;
        }
    }

    /* 发送一个end信号给请求学习的Learner，表明发送结束*，此时的@m_llSequence
     * 已经跟SendCheckpointBegin中不一样了
     */
    ret = m_poLearner->SendCheckpointEnd(
            m_iSendNodeID, m_llUUID, m_llSequence, 
            m_poSMFac->GetCheckpointInstanceID(m_poConfig->GetMyGroupIdx()));
    if (ret != 0)
    {
        PLGErr("SendCheckpointEnd fail, sequence %lu ret %d", m_llSequence, ret);
    }
    
    BP->GetCheckpointBP()->SendCheckpointEnd();
}

int Learner :: SendCheckpointBegin(
        const nodeid_t iSendNodeID,
        const uint64_t llUUID,
        const uint64_t llSequence,
        const uint64_t llCheckpointInstanceID)
{
    CheckpointMsg oCheckpointMsg;

    /*消息类型CheckpointMsgType_SendFile*/
    oCheckpointMsg.set_msgtype(CheckpointMsgType_SendFile);
    oCheckpointMsg.set_nodeid(m_poConfig->GetMyNodeID());
    /*消息标识：CheckpointSendFileFlag_BEGIN*/
    oCheckpointMsg.set_flag(CheckpointSendFileFlag_BEGIN);
    oCheckpointMsg.set_uuid(llUUID);
    oCheckpointMsg.set_sequence(llSequence);
    oCheckpointMsg.set_checkpointinstanceid(llCheckpointInstanceID);

    PLGImp("END, SendNodeID %lu uuid %lu sequence %lu cpi %lu",
            iSendNodeID, llUUID, llSequence, llCheckpointInstanceID);

    return SendMessage(iSendNodeID, oCheckpointMsg, Message_SendType_TCP);
}

int CheckpointSender :: SendCheckpointFofaSM(StateMachine * poSM)
{
    string sDirPath;
    std::vector<std::string> vecFileList;

    /*返回该group的checkpoint数据的根目录，以及该根目录下的checkpoint数据的文件列表*/
    int ret = poSM->GetCheckpointState(m_poConfig->GetMyGroupIdx(), sDirPath, vecFileList);
    if (ret != 0)
    {
        PLGErr("GetCheckpointState fail ret %d, smid %d", ret, poSM->SMID());
        return -1;
    }

    if (sDirPath.size() == 0)
    {
        PLGImp("No Checkpoint, smid %d", poSM->SMID());
        return 0;
    }

    if (sDirPath[sDirPath.size() - 1] != '/')
    {
        sDirPath += '/';
    }

    /*逐一发送@sDirPath目录下的每一个文件@sFilePath*/
    for (auto & sFilePath : vecFileList)
    {
        ret = SendFile(poSM, sDirPath, sFilePath);
        if (ret != 0)
        {
            PLGErr("SendFile fail, ret %d smid %d", ret , poSM->SMID());
            return -1;
        }
    }

    PLGImp("END, send ok, smid %d filelistcount %zu", poSM->SMID(), vecFileList.size());
    return 0;
}

int CheckpointSender :: SendFile(const StateMachine * poSM, const std::string & sDirPath, const std::string & sFilePath)
{
    PLGHead("START smid %d dirpath %s filepath %s", poSM->SMID(), sDirPath.c_str(), sFilePath.c_str());

    /*目录加文件名组成文件的绝对路径*/
    string sPath = sDirPath + sFilePath;

    /*如果已经发送，则直接退出，@m_mapAlreadySendFile表示已经发送的文件集合*/
    if (m_mapAlreadySendedFile.find(sPath) != end(m_mapAlreadySendedFile))
    {
        PLGErr("file already send, filepath %s", sPath.c_str());
        return 0;
    }

    /*打开将要被发送的文件*/
    int iFD = open(sPath.c_str(), O_RDWR, S_IREAD);

    if (iFD == -1)
    {
        PLGErr("Open file fail, filepath %s", sPath.c_str());
        return -1;
    }

    ssize_t iReadLen = 0;
    size_t llOffset = 0;
    /*分段读取数据到@s_sTmpBuffer中，并通过SendBuffer接口发送数据给CheckpointReceiver*/
    while (true)
    {
        iReadLen = read(iFD, m_sTmpBuffer, sizeof(m_sTmpBuffer));
        if (iReadLen == 0)
        {
            break;
        }

        if (iReadLen < 0)
        {
            close(iFD);
            return -1;
        }

        /*发送数据*/
        int ret = SendBuffer(poSM->SMID(), poSM->GetCheckpointInstanceID(m_poConfig->GetMyGroupIdx()), 
                sFilePath, llOffset, string(m_sTmpBuffer, iReadLen));
        if (ret != 0)
        {
            close(iFD);
            return ret;
        }

        PLGDebug("Send ok, offset %zu readlen %d", llOffset, iReadLen);

        /*当前文件处理完毕*/
        if (iReadLen < (ssize_t)sizeof(m_sTmpBuffer))
        {
            break;
        }

        llOffset += iReadLen;
    }

    /*记录当前文件@sPath已经发送*/
    m_mapAlreadySendedFile[sPath] = true;

    /*关闭文件*/
    close(iFD);
    PLGImp("END");

    return 0;
}

int CheckpointSender :: SendBuffer(const int iSMID, const uint64_t llCheckpointInstanceID, 
        const std::string & sFilePath, const uint64_t llOffset, const std::string & sBuffer)
{
    /*计算待发送数据的crc*/
    uint32_t iChecksum = crc32(0, (const uint8_t *)sBuffer.data(), sBuffer.size(), CRC32SKIP);

    int ret = 0;
    while (true)
    {
        /*如果检测到CheckpointSender需要停止，则停止之*/
        if (m_bIsEnd)
        {
            return -1;
        }
        
        /* 如果自从上次Ack以来，发送了超过Checkpoint_ACK_LEAD段数据，则检查是否发生了
         * Ack超时，如果Ack超时，则退出本次发送，等待Ack
         */
        if (!CheckAck(m_llSequence))
        {
            return -1;
        }
        
        /*通过Learner接口发送数据*/
        ret = m_poLearner->SendCheckpoint(
                m_iSendNodeID, m_llUUID, m_llSequence, llCheckpointInstanceID,
                iChecksum, sFilePath, iSMID, llOffset, sBuffer);

        BP->GetCheckpointBP()->SendCheckpointOneBlock();

        if (ret == 0)
        {
            /*发送成功，则更新@m_llSequence*/
            m_llSequence++;
            break;
        }
        else
        {
            /*发送失败，则休眠30ms，再重新发送*/
            PLGErr("SendCheckpoint fail, ret %d need sleep 30s", ret);
            Time::MsSleep(30000);
        }
    }

    return ret;
}

const bool CheckpointSender :: CheckAck(const uint64_t llSendSequence)
{
    /* 如果自从上次Ack以来，发送了超过Checkpoint_ACK_LEAD段数据，则检查是否发生了
     * Ack超时，如果Ack超时，则返回false，否则返回true
     */
    while (llSendSequence > m_llAckSequence + Checkpoint_ACK_LEAD)
    {
        uint64_t llNowTime = Time::GetSteadyClockMS();
        uint64_t llPassTime = llNowTime > m_llAbsLastAckTime ? llNowTime - m_llAbsLastAckTime : 0;

        if (m_bIsEnd)
        {
            return false;
        }

        if (llPassTime >= Checkpoint_ACK_TIMEOUT)
        {       
            PLGErr("Ack timeout, last acktime %lu", m_llAbsLastAckTime);
            return false;
        }       

        //PLGErr("Need sleep to slow down send speed, sendsequence %lu acksequence %lu",
                //llSendSequence, m_llAckSequence);
        Time::MsSleep(20);
    }

    return true;
}

int Learner :: SendCheckpoint(
        const nodeid_t iSendNodeID,
        const uint64_t llUUID,
        const uint64_t llSequence,
        const uint64_t llCheckpointInstanceID,
        const uint32_t iChecksum,
        const std::string & sFilePath,
        const int iSMID,
        const uint64_t llOffset,
        const std::string & sBuffer)
{
    CheckpointMsg oCheckpointMsg;

    /*消息类型为CheckpointMsgType_SendFile*/
    oCheckpointMsg.set_msgtype(CheckpointMsgType_SendFile);
    oCheckpointMsg.set_nodeid(m_poConfig->GetMyNodeID());
    /*消息标识：CheckpointSendFileFlag_ING，表示正在发送过程中*/
    oCheckpointMsg.set_flag(CheckpointSendFileFlag_ING);
    oCheckpointMsg.set_uuid(llUUID);
    oCheckpointMsg.set_sequence(llSequence);
    oCheckpointMsg.set_checkpointinstanceid(llCheckpointInstanceID);
    oCheckpointMsg.set_checksum(iChecksum);
    oCheckpointMsg.set_filepath(sFilePath);
    oCheckpointMsg.set_smid(iSMID);
    oCheckpointMsg.set_offset(llOffset);
    oCheckpointMsg.set_buffer(sBuffer);

    /*发送消息*/
    return SendMessage(iSendNodeID, oCheckpointMsg, Message_SendType_TCP);
}

int Learner :: SendCheckpointEnd(
        const nodeid_t iSendNodeID,
        const uint64_t llUUID,
        const uint64_t llSequence,
        const uint64_t llCheckpointInstanceID)
{
    CheckpointMsg oCheckpointMsg;

    /*消息类型：CheckpointMsgType_SendFile*/
    oCheckpointMsg.set_msgtype(CheckpointMsgType_SendFile);
    oCheckpointMsg.set_nodeid(m_poConfig->GetMyNodeID());
    /*消息标识：CheckpointSendFileFlag_END*/
    oCheckpointMsg.set_flag(CheckpointSendFileFlag_END);
    oCheckpointMsg.set_uuid(llUUID);
    oCheckpointMsg.set_sequence(llSequence);
    oCheckpointMsg.set_checkpointinstanceid(llCheckpointInstanceID);

    PLGImp("END, SendNodeID %lu uuid %lu sequence %lu cpi %lu",
            iSendNodeID, llUUID, llSequence, llCheckpointInstanceID);

    return SendMessage(iSendNodeID, oCheckpointMsg, Message_SendType_TCP);
}
```

## 接收到CheckpointReceiver发送过来的关于checkpointSendFile的Ack信息

```
/*接收到CheckpointReceiver发送过来的Ack OK信息*/
void CheckpointSender :: Ack(const nodeid_t iSendNodeID, const uint64_t llUUID, const uint64_t llSequence)
{
    if (iSendNodeID != m_iSendNodeID)
    {
        PLGErr("send nodeid not same, ack.sendnodeid %lu self.sendnodeid %lu", iSendNodeID, m_iSendNodeID);
        return;
    }

    if (llUUID != m_llUUID)
    {
        PLGErr("uuid not same, ack.uuid %lu self.uuid %lu", llUUID, m_llUUID);
        return;
    }

    if (llSequence != m_llAckSequence)
    {
        PLGErr("ack_sequence not same, ack.ack_sequence %lu self.ack_sequence %lu", llSequence, m_llAckSequence);
        return;
    }

    /*更新@m_llAckSequence*/
    m_llAckSequence++;
    m_llAbsLastAckTime = Time::GetSteadyClockMS();
}

/*接收到CheckpointReceiver发送过来的Ack fail信息*/
void CheckpointSender :: End()
{   
    /*停止CheckpointSender*/
    m_bIsEnd = true;
}
```

# CheckpointReceiver方法
## 新建一个CheckpointReceiver

```
int CheckpointReceiver :: NewReceiver(const nodeid_t iSenderNodeID, const uint64_t llUUID)
{
    /*清理CheckpointReceiver用于存放学习到的镜像状态机数据的临时文件*/
    int ret = ClearCheckpointTmp();
    if (ret != 0)
    {
        return ret;
    }

    /*删除LogStorage中该paxos group相关的所有文件，具体请参考MultiDatabase::ClearAllLog*/
    ret = m_poLogStorage->ClearAllLog(m_poConfig->GetMyGroupIdx());
    if (ret != 0)
    {
        PLGErr("ClearAllLog fail, groupidx %d ret %d", 
                m_poConfig->GetMyGroupIdx(), ret);
        return ret;
    }
    
    m_mapHasInitDir.clear();

    m_iSenderNodeID = iSenderNodeID;
    m_llUUID = llUUID;
    m_llSequence = 0;

    return 0;
}

int CheckpointReceiver :: ClearCheckpointTmp()
{
    /*找到该paxos group在LogStorage中的存储路径*/
    string sLogStoragePath = m_poLogStorage->GetLogStorageDirPath(m_poConfig->GetMyGroupIdx());

    DIR * dir = nullptr;
    struct dirent  * ptr;

    /*打开该paxos group所在的目录*/
    dir = opendir(sLogStoragePath.c_str());
    if (dir == nullptr)
    {
        return -1;
    }

    /* 依次读取该paxos group下的每一个文件，查找CheckpointReceiver用于存放
     * 学习到的镜像状态机数据的临时文件
     */
    int ret = 0;
    while ((ptr = readdir(dir)) != nullptr)
    {
        /*查找到了文件名中包含"cm_tmp_"关键字的文件*/
        if (string(ptr->d_name).find("cp_tmp_") != std::string::npos)
        {
            char sChildPath[1024] = {0};
            snprintf(sChildPath, sizeof(sChildPath), "%s/%s", sLogStoragePath.c_str(), ptr->d_name);
            /*删除该文件*/
            ret = FileUtils::DeleteDir(sChildPath);
            
            if (ret != 0)
            {
                break;
            }

            PLGHead("rm dir %s done!", sChildPath);
        }
    }

    /*关闭paxos group所在的目录*/
    closedir(dir);

    return ret;
}
```

## 接收到CheckpointSender发送过来的数据

```
int CheckpointReceiver :: ReceiveCheckpoint(const CheckpointMsg & oCheckpointMsg)
{
    if (oCheckpointMsg.nodeid() != m_iSenderNodeID
            || oCheckpointMsg.uuid() != m_llUUID)
    {
        PLGErr("msg not valid, Msg.SenderNodeID %lu Receiver.SenderNodeID %lu Msg.UUID %lu Receiver.UUID %lu",
                oCheckpointMsg.nodeid(), m_iSenderNodeID, oCheckpointMsg.uuid(), m_llUUID);
        return -2;
    }

    if (oCheckpointMsg.sequence() == m_llSequence)
    {
        PLGErr("msg already receive, skip, Msg.Sequence %lu Receiver.Sequence %lu",
                oCheckpointMsg.sequence(), m_llSequence);
        return 0;
    }

    /*oCheckpointMsg.sequence() == m_llSequence + 1才能接收之*/
    if (oCheckpointMsg.sequence() != m_llSequence + 1)
    {
        PLGErr("msg sequence wrong, Msg.Sequence %lu Receiver.Sequence %lu",
                oCheckpointMsg.sequence(), m_llSequence);
        return -2;
    }

    /*在@oCheckpointMsg.smid()所代表的状态机的临时目录下创建名为@oCheckpointMsg.filepath()的文件*/
    string sFilePath = GetTmpDirPath(oCheckpointMsg.smid()) + "/" + oCheckpointMsg.filepath();
    string sFormatFilePath;
    /*检查@sFilePath中的各目录是否存在，如果不存在则创建之，并添加到已创建的目录集合中*/
    int ret = InitFilePath(sFilePath, sFormatFilePath);
    if (ret != 0)
    {
        return -1;
    }

    /*打开该临时文件*/
    int iFd = open(sFormatFilePath.c_str(), O_CREAT | O_RDWR | O_APPEND, S_IWRITE | S_IREAD);
    if (iFd == -1)
    {
        PLGErr("open file fail, filepath %s", sFormatFilePath.c_str());
        return -1;
    }

    /*该文件大小应该和@oCheckpointMsg.offset()相同*/
    size_t llFileOffset = lseek(iFd, 0, SEEK_END);
    if ((uint64_t)llFileOffset != oCheckpointMsg.offset())
    {
        PLGErr("file.offset %zu not equal to msg.offset %lu", llFileOffset, oCheckpointMsg.offset());
        close(iFd);
        return -2;
    }

    /*写入数据*/
    size_t iWriteLen = write(iFd, oCheckpointMsg.buffer().data(), oCheckpointMsg.buffer().size());
    if (iWriteLen != oCheckpointMsg.buffer().size())
    {
        PLGImp("write fail, writelen %zu buffer size %zu", iWriteLen, oCheckpointMsg.buffer().size());
        close(iFd);
        return -1;
    }

    /*增加@m_llSequence*/
    m_llSequence++;
    /*关闭文件*/
    close(iFd);

    PLGImp("END ok, writelen %zu", iWriteLen);

    return 0;
}

const std::string CheckpointReceiver :: GetTmpDirPath(const int iSMID)
{
    /*该paxos group在LogStorage中的目录*/
    string sLogStoragePath = m_poLogStorage->GetLogStorageDirPath(m_poConfig->GetMyGroupIdx());
    char sTmpDirPath[512] = {0};

    /*@iSMID对应的状态机的临时目录*/
    snprintf(sTmpDirPath, sizeof(sTmpDirPath), "%s/cp_tmp_%d", sLogStoragePath.c_str(), iSMID);

    return string(sTmpDirPath);
}

int CheckpointReceiver :: InitFilePath(const std::string & sFilePath, std::string & sFormatFilePath)
{
    PLGHead("START filepath %s", sFilePath.c_str());

    string sNewFilePath = "/" + sFilePath + "/";
    vector<std::string> vecDirList;

    /*将@sFilePath中路径进行拆分，拆分成各目录，单独存放于@vecDirList中*/
    std::string sDirName;
    for (size_t i = 0; i < sNewFilePath.size(); i++)
    {
        if (sNewFilePath[i] == '/')
        {
            if (sDirName.size() > 0)
            {
                vecDirList.push_back(sDirName);
            }

            sDirName = "";
        }
        else
        {
            sDirName += sNewFilePath[i];
        }
    }

    /* 依次遍历@vecDirList，检查该目录是否存在，如果不存在则创建之，并添加到已创
     * 建目录集合@m_mapHasInitDir中
     */
    sFormatFilePath = "/";
    for (size_t i = 0; i < vecDirList.size(); i++)
    {
        if (i + 1 == vecDirList.size())
        {
            sFormatFilePath += vecDirList[i];
        }
        else
        {
            sFormatFilePath += vecDirList[i] + "/";
            if (m_mapHasInitDir.find(sFormatFilePath) == end(m_mapHasInitDir))
            {
                int ret = CreateDir(sFormatFilePath);
                if (ret != 0)
                {
                    return ret;
                }

                m_mapHasInitDir[sFormatFilePath] = true;
            }
        }
    }

    PLGImp("ok, format filepath %s", sFormatFilePath.c_str());

    return 0;
}
```

## 接收到CheckpointSender发送过来的“发送结束”的信号

```
const bool CheckpointReceiver :: IsReceiverFinish(const nodeid_t iSenderNodeID, 
        const uint64_t llUUID, const uint64_t llEndSequence)
{
    /*检查是否是正确的结束checkpoint的消息*/
    if (iSenderNodeID == m_iSenderNodeID
            && llUUID == m_llUUID
            && llEndSequence == m_llSequence + 1)
    {
        return true;
    }
    else
    {
        return false;
    }
}
```





