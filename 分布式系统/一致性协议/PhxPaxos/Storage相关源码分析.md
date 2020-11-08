# 提纲
[toc]

# Storage中各类组织关系
## MultiDatabase和Database的关系
待续...

## Database中leveldb和LogStore的关系
待续...


## PaxosLog和LogStorage的关系
待续...


## LogStorage和LogStore的关系
待续...


# Storage初始化调用关系

```
Node :: RunNode(const Options & oOptions, Node *& poNode)
    - PNode :: Init(const Options & oOptions, NetWork *& poNetWork)
        - PNode :: InitLogStorage(const Options & oOptions, LogStorage *& poLogStorage)
            - MultiDatabase :: Init(const std::string & sDBPath, const int iGroupCount)
```


# MultiDatabase
## MultiDatabase定义
```
class MultiDatabase : public LogStorage
{
private:
    std::vector<Database *> m_vecDBList;
};
```

## MultiDatabase方法
### MultiDatabase初始化
```
int MultiDatabase :: Init(const std::string & sDBPath, const int iGroupCount)
{
    /*log storage path @sDBPath 所指定的路径对应的文件必须存在*/
    if (access(sDBPath.c_str(), F_OK) == -1)
    {
        PLErr("DBPath not exist or no limit to open, %s", sDBPath.c_str());
        return -1;
    }

    if (iGroupCount < 1 || iGroupCount > 100000)
    {
        PLErr("Groupcount wrong %d", iGroupCount);
        return -2;
    }

    std::string sNewDBPath = sDBPath;

    if (sDBPath[sDBPath.size() - 1] != '/')
    {
        sNewDBPath += '/';
    }

    /*在log storage path @sDBPath下面为每一个paxos group创建一个独立的Database实例*/
    for (int iGroupIdx = 0; iGroupIdx < iGroupCount; iGroupIdx++)
    {
        /*@iGroupIdx对应的paxos group的log storage path*/
        char sGroupDBPath[512] = {0};
        snprintf(sGroupDBPath, sizeof(sGroupDBPath), "%sg%d", sNewDBPath.c_str(), iGroupIdx);

        Database * poDB = new Database();
        assert(poDB != nullptr);
        m_vecDBList.push_back(poDB);

        /*初始化@iGroupIdx对应的paxos group的Database实例*/
        if (poDB->Init(sGroupDBPath, iGroupIdx) != 0)
        {
            return -1;
        }
    }

    PLImp("OK, DBPath %s groupcount %d", sDBPath.c_str(), iGroupCount);

    return 0;
}
```

### 向MultiDatabase中写入指定paxos group的指定InstanceID相关的信息

```
int MultiDatabase :: Put(const WriteOptions & oWriteOptions, const int iGroupIdx, const uint64_t llInstanceID, const std::string & sValue)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }
    
    /*写入指定paxos group对应的Database实例*/
    return m_vecDBList[iGroupIdx]->Put(oWriteOptions, llInstanceID, sValue);
}
```

### 从MultiDatabase中读取指定paxos group的指定InstanceID相关的信息

```
int MultiDatabase :: Get(const int iGroupIdx, const uint64_t llInstanceID, std::string & sValue)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }

    /*从指定paxos group对应的Database实例中读取*/
    return m_vecDBList[iGroupIdx]->Get(llInstanceID, sValue);
}
```

### 向MultiDatabase中写入

```
int MultiDatabase :: SetMinChosenInstanceID(const WriteOptions & oWriteOptions, const int iGroupIdx, const uint64_t llMinInstanceID)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }

    return m_vecDBList[iGroupIdx]->SetMinChosenInstanceID(oWriteOptions, llMinInstanceID);
}
```

### 从MultiDatabase中获取

```
int MultiDatabase :: GetMinChosenInstanceID(const int iGroupIdx, uint64_t & llMinInstanceID)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }

    return m_vecDBList[iGroupIdx]->GetMinChosenInstanceID(llMinInstanceID);
}
```

### 从MultiDatabase中删除指定group中指定Instance的数据
```
int MultiDatabase :: Del(const WriteOptions & oWriteOptions, const int iGroupIdx, const uint64_t llInstanceID)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }
    
    return m_vecDBList[iGroupIdx]->Del(oWriteOptions, llInstanceID);
}
```

### 从MultiDatabase中删除指定group相关的日志数据

```
int MultiDatabase :: ClearAllLog(const int iGroupIdx)
{
    if (iGroupIdx >= (int)m_vecDBList.size())
    {
        return -2;
    }

    return m_vecDBList[iGroupIdx]->ClearAllLog();
}
```

# Database
## Database定义
```
class Database
{
public:
    leveldb::DB * m_poLevelDB;
    PaxosComparator m_oPaxosCmp;
    bool m_bHasInit;
    
    LogStore * m_poValueStore;
    std::string m_sDBPath;

    int m_iMyGroupIdx;

private:
    TimeStat m_oTimeStat;
};
```

## Database方法
### Database初始化
```
int Database :: Init(const std::string & sDBPath, const int iMyGroupIdx)
{
    if (m_bHasInit)
    {
        return 0;
    }

    m_iMyGroupIdx = iMyGroupIdx;
    m_sDBPath = sDBPath;
    
    /*设置leveldb相关的选项*/
    leveldb::Options oOptions;
    oOptions.create_if_missing = true;
    oOptions.comparator = &m_oPaxosCmp;
    //every group have different buffer size to avoid all group compact at the same time.
    oOptions.write_buffer_size = 1024 * 1024 + iMyGroupIdx * 10 * 1024;
    /*打开（如果不存在则需要事先创建）leveldb实例@m_poLevelDB*/
    leveldb::Status oStatus = leveldb::DB::Open(oOptions, sDBPath, &m_poLevelDB);
    if (!oStatus.ok())
    {
        PLG1Err("Open leveldb fail, db_path %s", sDBPath.c_str());
        return -1;
    }

    /* 创建并打开LogStore，前面不是已经打开了leveldb了吗，怎么还有所谓的LogStore呢？
     * 在PhxPaxos中，leveldb和LogStore都是用于存储的，但是二者的分工是不同的，其中
     * LogStore是专门用于存储PhxPaxos相关的数据的文件，leveldb用于存储instanceid到
     * LogStore文件偏移的映射
     */
    m_poValueStore = new LogStore(); 
    assert(m_poValueStore != nullptr);
    int ret = m_poValueStore->Init(sDBPath, iMyGroupIdx, (Database *)this);
    if (ret != 0)
    {
        PLG1Err("value store init fail, ret %d", ret);
        return -1;
    }

    m_bHasInit = true;

    PLG1Imp("OK, db_path %s", sDBPath.c_str());

    return 0;
}
```

### 获取最大的instance ID
```
int Database :: GetMaxInstanceID(uint64_t & llInstanceID)
{
    llInstanceID = MINCHOSEN_KEY;

    /*创建leveldb迭代器，用于读取，采用默认的读取选项来设置迭代器*/
    leveldb::Iterator * it = m_poLevelDB->NewIterator(leveldb::ReadOptions());
    /*定位到最后一个key*/
    it->SeekToLast();

    while (it->Valid())
    {
        /*从key中解析出instance ID*/
        llInstanceID = GetInstanceIDFromKey(it->key().ToString());
        if (llInstanceID == MINCHOSEN_KEY
                || llInstanceID == SYSTEMVARIABLES_KEY
                || llInstanceID == MASTERVARIABLES_KEY)
        {
            /* 如果是非法instance ID，则尝试查找当前迭代器前面一个key，其中MINCHOSEN_KEY、
             * SYSTEMVARIABLES_KEY和MASTERVARIABLES_KEY分别对应(uint64_t)-1, (uint64_t)-2,
             * (uint64_t)-3
             */
            it->Prev();
        }
        else
        {
            /*否则，已经找到了最大的instance ID，返回成功*/
            delete it;
            return 0;
        }
    }

    /*没有合法的instance ID，返回1*/
    delete it;
    return 1;
}
```

### 获取最大的instance ID及其对应的File ID
```
int Database :: GetMaxInstanceIDFileID(std::string & sFileID, uint64_t & llInstanceID)
{
    uint64_t llMaxInstanceID = 0;
    /*获取最大的instance ID*/
    int ret = GetMaxInstanceID(llMaxInstanceID);
    if (ret != 0 && ret != 1)
    {
        return ret;
    }

    if (ret == 1)
    {
        /*没有找到合法的instance ID，则@sFileID也不存在*/
        sFileID = "";
        return 0;
    }

    /*通过instance ID获取key*/
    string sKey = GenKey(llMaxInstanceID);
    /*从leveldb中读取@sKey对应的File ID @sFileID*/
    leveldb::Status oStatus = m_poLevelDB->Get(leveldb::ReadOptions(), sKey, &sFileID);
    if (!oStatus.ok())
    {
        /*读取失败*/
        if (oStatus.IsNotFound())
        {
            BP->GetLogStorageBP()->LevelDBGetNotExist();
            //PLG1Err("LevelDB.Get not found %s", sKey.c_str());
            return 1;
        }
        
        BP->GetLogStorageBP()->LevelDBGetFail();
        PLG1Err("LevelDB.Get fail");
        return -1;
    }

    llInstanceID = llMaxInstanceID;

    return 0;
}
```

### 存储InstanceID到FileID的映射到Database中

```
int Database :: RebuildOneIndex(const uint64_t llInstanceID, const std::string & sFileID)
{
    /*根据@llInstanceID生成key*/
    string sKey = GenKey(llInstanceID);

    leveldb::WriteOptions oLevelDBWriteOptions;
    oLevelDBWriteOptions.sync = false;

    /*将(sKey, sFileID)写入leveldb中*/
    leveldb::Status oStatus = m_poLevelDB->Put(oLevelDBWriteOptions, sKey, sFileID);
    if (!oStatus.ok())
    {
        BP->GetLogStorageBP()->LevelDBPutFail();
        PLG1Err("LevelDB.Put fail, instanceid %lu valuelen %zu", llInstanceID, sFileID.size());
        return -1;
    }

    return 0;
}
```

### 向Database中写入指定InstanceID的信息

```
int Database :: Put(const WriteOptions & oWriteOptions, const uint64_t llInstanceID, const std::string & sValue)
{
    if (!m_bHasInit)
    {
        PLG1Err("no init yet");
        return -1;
    }

    std::string sFileID;
    /* 将@sValue写入LogStore，同时返回本实例对应的FileID @sFileID（本次写入所在的
     * 文件ID、文件偏移和写入内容的校验和组合而成）
     */
    int ret = ValueToFileID(oWriteOptions, llInstanceID, sValue, sFileID);
    if (ret != 0)
    {
        return ret;
    }

    /*将实例@llInstanceID到FileID @sFileID的映射写入leveldb中*/
    ret = PutToLevelDB(false, llInstanceID, sFileID);
    
    return ret;
}

int Database :: PutToLevelDB(const bool bSync, const uint64_t llInstanceID, const std::string & sValue)
{
    /*根据InstanceID生成key*/
    string sKey = GenKey(llInstanceID);

    leveldb::WriteOptions oLevelDBWriteOptions;
    oLevelDBWriteOptions.sync = bSync;

    m_oTimeStat.Point();

    /*将键值对写入leveldb*/
    leveldb::Status oStatus = m_poLevelDB->Put(oLevelDBWriteOptions, sKey, sValue);
    if (!oStatus.ok())
    {
        BP->GetLogStorageBP()->LevelDBPutFail();
        PLG1Err("LevelDB.Put fail, instanceid %lu valuelen %zu", llInstanceID, sValue.size());
        return -1;
    }

    BP->GetLogStorageBP()->LevelDBPutOK(m_oTimeStat.Point());

    return 0;
}
```
从Database::Put接口可知，某个Instance相关的数据信息（对应于方法中的@sValue参数）被写入到文件中（由LogStore管理），同时将该Instance在LogStore中的位置信息（包括文件ID，文件偏移和校验和组合而成）写入到数据库中（默认的采用的是leveldb）。


### 从Database中读取指定InstanceID的信息

```
int Database :: Get(const uint64_t llInstanceID, std::string & sValue)
{
    if (!m_bHasInit)
    {
        PLG1Err("no init yet");
        return -1;
    }

    /*从leveldb中获取Instance @llInstanceID对应的FileID信息*/
    string sFileID;
    int ret = GetFromLevelDB(llInstanceID, sFileID);
    if (ret != 0)
    {
        return ret;
    }

    /* 根据FileID（其中包括该Instance所在的文件以及在文件中的位置信息）在LogStore
     * 中找到对应的文件的特定位置，并从中读取出其中记录的InstanceID和value信息
     */
    uint64_t llFileInstanceID = 0;
    ret = FileIDToValue(sFileID, llFileInstanceID, sValue);
    if (ret != 0)
    {
        BP->GetLogStorageBP()->FileIDToValueFail();
        return ret;
    }

    /*文件中读取到的InstanceID应该和参数中传递的InstanceID相同*/
    if (llFileInstanceID != llInstanceID)
    {
        PLG1Err("file instanceid %lu not equal to key.instanceid %lu", llFileInstanceID, llInstanceID);
        return -2;
    }

    return 0;
}

int Database :: GetFromLevelDB(const uint64_t llInstanceID, std::string & sValue)
{
    /*将@llInstanceID转换为key*/
    string sKey = GenKey(llInstanceID);
    
    /*从leveldb中读取相应的value*/
    leveldb::Status oStatus = m_poLevelDB->Get(leveldb::ReadOptions(), sKey, &sValue);
    if (!oStatus.ok())
    {
        if (oStatus.IsNotFound())
        {
            BP->GetLogStorageBP()->LevelDBGetNotExist();
            PLG1Debug("LevelDB.Get not found, instanceid %lu", llInstanceID);
            return 1;
        }
        
        BP->GetLogStorageBP()->LevelDBGetFail();
        PLG1Err("LevelDB.Get fail, instanceid %lu", llInstanceID);
        return -1;
    }

    return 0;
}
```

### 向Database中写入某个Instance的value，并且返回相应的FileID

```
int Database :: ValueToFileID(const WriteOptions & oWriteOptions, const uint64_t llInstanceID, const std::string & sValue, std::string & sFileID)
{
    /*@m_poValueStore类型是LogStore，采用文件存储，将Instance @llInstanceID的 Value @sValue写入文件，并返回*/
    int ret = m_poValueStore->Append(oWriteOptions, llInstanceID, sValue, sFileID);
    if (ret != 0)
    {
        BP->GetLogStorageBP()->ValueToFileIDFail();
        PLG1Err("fail, ret %d", ret);
        return ret;
    }

    return 0;
}
```

### 从Database中获取某个Instance的value

```
int Database :: FileIDToValue(const std::string & sFileID, uint64_t & llInstanceID, std::string & sValue)
{
    /*@m_poValueStore类型是LogStore，采用文件存储，从文件中读取该实例@llInstanceID对应的value*/
    int ret = m_poValueStore->Read(sFileID, llInstanceID, sValue);
    if (ret != 0)
    {
        PLG1Err("fail, ret %d", ret);
        return ret;
    }

    return 0;
}
```

### 向Database中写入最小的chosen InstanceID

```
int Database :: SetMinChosenInstanceID(const WriteOptions & oWriteOptions, const uint64_t llMinInstanceID)
{
    if (!m_bHasInit)
    {
        PLG1Err("no init yet");
        return -1;
    }

    /*采用系统预留的的MINCHOSEN_KEY（(uint64_t)-1）作为其key*/
    static uint64_t llMinKey = MINCHOSEN_KEY;
    char sValue[sizeof(uint64_t)] = {0};
    memcpy(sValue, &llMinInstanceID, sizeof(uint64_t));

    /*写入leveldb*/
    int ret = PutToLevelDB(true, llMinKey, string(sValue, sizeof(uint64_t)));
    if (ret != 0)
    {
        return ret;
    }

    PLG1Imp("ok, min chosen instanceid %lu", llMinInstanceID);

    return 0;
}
```

### 从Database中读取最小的chosen InstanceID

```
int Database :: GetMinChosenInstanceID(uint64_t & llMinInstanceID)
{
    if (!m_bHasInit)
    {
        PLG1Err("no init yet");
        return -1;
    }

    static uint64_t llMinKey = MINCHOSEN_KEY;
    std::string sValue;
    /*从leveldb中读取最小的chosen InstanceID*/
    int ret = GetFromLevelDB(llMinKey, sValue);
    if (ret != 0 && ret != 1)
    {
        PLG1Err("fail, ret %d", ret);
        return ret;
    }

    if (ret == 1)
    {
        /*在leveldb中没有找到key为MINCHOSEN_KEY的项*/
        PLG1Err("no min chosen instanceid");
        llMinInstanceID = 0;
        return 0;
    }

    //old version, minchonsenid store in logstore.
    //new version, minchonsenid directly store in leveldb.
    /* 这里是为了处理新旧两个版本的phxpaxos的兼容性，旧版本中将最小chosen
     * InstanceID存放在LogStore中，而新版本则存放在LevelDB中，对于新版本而
     * 言，该判断不成立
     */
    if (m_poValueStore->IsValidFileID(sValue))
    {
        ret = Get(llMinKey, sValue);
        if (ret != 0 && ret != 1)
        {
            PLG1Err("Get from log store fail, ret %d", ret);
            return ret;
        }
    }

    /*至此，@sValue中存放的一定是InstanceID*/
    if (sValue.size() != sizeof(uint64_t))
    {
        PLG1Err("fail, mininstanceid size wrong");
        return -2;
    }

    /*从@sValue中解析出InstanceID*/
    memcpy(&llMinInstanceID, sValue.data(), sizeof(uint64_t));

    PLG1Imp("ok, min chosen instanceid %lu", llMinInstanceID);

    return 0;
}
```

### 从Database删除指定Instance的数据
```
int Database :: Del(const WriteOptions & oWriteOptions, const uint64_t llInstanceID)
{
    if (!m_bHasInit)
    {
        PLG1Err("no init yet");
        return -1;
    }

	/*根据InstanceID生成key*/
    string sKey = GenKey(llInstanceID);

	/*并非每一次都会执行删除，是否删除是随机的*/
    if (OtherUtils::FastRand() % 100 < 1)
    {
        //no need to del vfile every times.
		
		/*从leveldb中获取该实例对应的FileID*/
        string sFileID;
        leveldb::Status oStatus = m_poLevelDB->Get(leveldb::ReadOptions(), sKey, &sFileID);
        if (!oStatus.ok())
        {
            if (oStatus.IsNotFound())
            {
                PLG1Debug("LevelDB.Get not found, instanceid %lu", llInstanceID);
                return 0;
            }
            
            PLG1Err("LevelDB.Get fail, instanceid %lu", llInstanceID);
            return -1;
        }

		/*从LogStore的给定@sFileID中删除给定实例@llInstanceID的数据*/
        int ret = m_poValueStore->Del(sFileID, llInstanceID);
        if (ret != 0)
        {
            return ret;
        }
    }

    leveldb::WriteOptions oLevelDBWriteOptions;
    oLevelDBWriteOptions.sync = oWriteOptions.bSync;
    /*从leveldb中删除该实例的记录*/
    leveldb::Status oStatus = m_poLevelDB->Delete(oLevelDBWriteOptions, sKey);
    if (!oStatus.ok())
    {
        PLG1Err("LevelDB.Delete fail, instanceid %lu", llInstanceID);
        return -1;
    }

    return 0;
}
```

### 从Database中删除指定group相关的日志数据

```
int Database :: ClearAllLog()
{
    /*从Leveldb中读取System Variables*/
    string sSystemVariablesBuffer;
    int ret = GetSystemVariables(sSystemVariablesBuffer);
    if (ret != 0 && ret != 1)
    {
        PLG1Err("GetSystemVariables fail, ret %d", ret);
        return ret;
    }

    /*从Leveldb中读取Master Variables*/
    string sMasterVariablesBuffer;
    ret = GetMasterVariables(sMasterVariablesBuffer);
    if (ret != 0 && ret != 1)
    {
        PLG1Err("GetMasterVariables fail, ret %d", ret);
        return ret;
    }

    m_bHasInit = false;

    delete m_poLevelDB;
    m_poLevelDB = nullptr;

    delete m_poValueStore;
    m_poValueStore = nullptr;

    string sBakPath = m_sDBPath + ".bak";

    ret = FileUtils::DeleteDir(sBakPath);
    if (ret != 0)
    {
        PLG1Err("Delete bak dir fail, dir %s", sBakPath.c_str());
        return -1;
    }

    /*将当前目录重命名*/
    ret = rename(m_sDBPath.c_str(), sBakPath.c_str());
    assert(ret == 0);

    /* 重新初始化该group在LogStorage中的存储（这里会重新打开leveldb 
     * @m_poLevelDB和文件存储@m_poValueStore）
     */
    ret = Init(m_sDBPath, m_iMyGroupIdx);
    if (ret != 0)
    {
        PLG1Err("Init again fail, ret %d", ret);
        return ret;
    }

    /*重新在leveldb中写入System Variables*/
    WriteOptions oWriteOptions;
    oWriteOptions.bSync = true;
    if (sSystemVariablesBuffer.size() > 0)
    {
        ret = SetSystemVariables(oWriteOptions, sSystemVariablesBuffer);
        if (ret != 0)
        {
            PLG1Err("SetSystemVariables fail, ret %d", ret);
            return ret;
        }
    }

    /*重新在leveldb中写入Master Variables*/
    if (sMasterVariablesBuffer.size() > 0)
    {
        ret = SetMasterVariables(oWriteOptions, sMasterVariablesBuffer);
        if (ret != 0)
        {
            PLG1Err("SetMasterVariables fail, ret %d", ret);
            return ret;
        }
    }

    return 0;
}
```

# LogStore相关的接口
## LogStore初始化
### LogStore初始化 - 入口
```
int LogStore :: Init(const std::string & sPath, const int iMyGroupIdx, Database * poDatabase)
{
    m_iMyGroupIdx = iMyGroupIdx;
    
    /*确保@m_sPath所指定的目录存在（不存在则创建之）*/
    m_sPath = sPath + "/" + "vfile";
    if (access(m_sPath.c_str(), F_OK) == -1)
    {
        if (mkdir(m_sPath.c_str(), S_IRWXU | S_IRWXG | S_IROTH | S_IXOTH) == -1)
        {
            PLG1Err("Create dir fail, path %s", m_sPath.c_str());
            return -1;
        }
    }

    /*在@m_sPath所指定的目录下打开名为LOG的文件，@m_oFileLogger中存放打开的文件句柄*/
    m_oFileLogger.Init(m_sPath);

    /*在@m_sPath目录下打开名为meta的文件，@m_iMetaFd表示打开的文件句柄*/
    string sMetaFilePath = m_sPath + "/meta";
    m_iMetaFd = open(sMetaFilePath.c_str(), O_CREAT | O_RDWR, S_IREAD | S_IWRITE);
    if (m_iMetaFd == -1)
    {
        PLG1Err("open meta file fail, filepath %s", sMetaFilePath.c_str());
        return -1;
    }

    /*将meta文件读写指针定位到文件开头*/
    off_t iSeekPos = lseek(m_iMetaFd, 0, SEEK_SET);
    if (iSeekPos == -1)
    {
        return -1;
    }

    /* 尝试读取meta文件中的前sizeof(int)字节，如果能够成功读取，则其中存放的是
     * 当前系统中用到的最大FileID信息
     */
    ssize_t iReadLen = read(m_iMetaFd, &m_iFileID, sizeof(int));
    if (iReadLen != (ssize_t)sizeof(int))
    {
        if (iReadLen == 0)
        {
            /*meta文件中没有记录FileID信息，则直接设置@m_iFileID为0*/
            m_iFileID = 0;
        }
        else
        {
            PLG1Err("read meta info fail, readlen %zd", iReadLen);
            return -1;
        }
    }

    /* 尝试读取meta文件中接下来的sizeof(uint32_t)字节，如果能够成功读取，则其中
     * 存放的是FileID的checksum信息
     */
    uint32_t iMetaChecksum = 0;
    iReadLen = read(m_iMetaFd, &iMetaChecksum, sizeof(uint32_t));
    if (iReadLen == (ssize_t)sizeof(uint32_t))
    {
        /*读取成功，则检查计算得到的checksum和读取到的checksum是否一致*/
        uint32_t iCheckSum = crc32(0, (const uint8_t*)(&m_iFileID), sizeof(int));
        if (iCheckSum != iMetaChecksum)
        {
            PLG1Err("meta file checksum %u not same to cal checksum %u, fileid %d",
                    iMetaChecksum, iCheckSum, m_iFileID);
            return -2;
        }
    }

    /* 从leveldb中记录的最大instance ID对应的FileID开始，重建instance ID到File ID
     * 的映射，并将该映射信息写入leveldb中
     */
    int ret = RebuildIndex(poDatabase, m_iNowFileOffset);
    if (ret != 0)
    {
        PLG1Err("rebuild index fail, ret %d", ret);
        return -1;
    }

    /*打开系统中最大FileID的文件*/
    ret = OpenFile(m_iFileID, m_iFd);
    if (ret != 0)
    {
        return ret;
    }

    /*将@m_iFileID对应的文件扩大到LOG_FILE_MAX_SIZE大小*/
    ret = ExpandFile(m_iFd, m_iNowFileSize);
    if (ret != 0)
    {
        return ret;
    }

    /*将文件读写指针定位到@m_iNowFileOffset这一位置，后续的写紧接着这里*/
    m_iNowFileOffset = lseek(m_iFd, m_iNowFileOffset, SEEK_SET);
    if (m_iNowFileOffset == -1)
    {
        /*lseek失败*/
        PLG1Err("seek to now file offset %d fail", m_iNowFileOffset);
        return -1;
    }

    m_oFileLogger.Log("init write fileid %d now_w_offset %d filesize %d", 
            m_iFileID, m_iNowFileOffset, m_iNowFileSize);

    PLG1Head("ok, path %s fileid %d meta checksum %u nowfilesize %d nowfilewriteoffset %d", 
            m_sPath.c_str(), m_iFileID, iMetaChecksum, m_iNowFileSize, m_iNowFileOffset);

    return 0;
}

void LogStoreLogger :: Init(const std::string & sPath)
{
    /*在目录@sPath下打开名为LOG的文件*/
    char sFilePath[512] = {0};
    snprintf(sFilePath, sizeof(sFilePath), "%s/LOG", sPath.c_str());
    m_iLogFd = open(sFilePath, O_CREAT | O_RDWR | O_APPEND, S_IWRITE | S_IREAD);
}

void LogStore :: ParseFileID(const std::string & sFileID, int & iFileID, int & iOffset, uint32_t & iCheckSum)
{
    memcpy(&iFileID, (void *)sFileID.c_str(), sizeof(int));
    memcpy(&iOffset, (void *)(sFileID.c_str() + sizeof(int)), sizeof(int));
    memcpy(&iCheckSum, (void *)(sFileID.c_str() + sizeof(int) + sizeof(int)), sizeof(uint32_t));

    PLG1Debug("fileid %d offset %d checksum %u", iFileID, iOffset, iCheckSum);
}

int LogStore :: ExpandFile(int iFd, int & iFileSize)
{
    iFileSize = lseek(iFd, 0, SEEK_END);
    if (iFileSize == -1)
    {
        /*lseek失败*/
        PLG1Err("lseek fail, ret %d", iFileSize);
        return -1;
    }

    if (iFileSize == 0)
    {
        //new file
        iFileSize = lseek(iFd, LOG_FILE_MAX_SIZE - 1, SEEK_SET);
        if (iFileSize != LOG_FILE_MAX_SIZE - 1)
        {
            /*lseek失败*/
            return -1;
        }

        /*在文件最后写入1个字节*/
        ssize_t iWriteLen = write(iFd, "\0", 1);
        if (iWriteLen != 1)
        {
            PLG1Err("write 1 bytes fail");
            return -1;
        }

        iFileSize = LOG_FILE_MAX_SIZE;
        int iOffset = lseek(iFd, 0, SEEK_SET);
        m_iNowFileOffset = 0;
        if (iOffset != 0)
        {
            /*lseek失败*/
            return -1;
        }
    }

    return 0;
}
```

### LogStore初始化 - 重建索引
```
int LogStore :: RebuildIndex(Database * poDatabase, int & iNowFileWriteOffset)
{
    string sLastFileID;

    uint64_t llNowInstanceID = 0;
    /* 从leveldb中读取最大的instance ID及其对应的File ID，分别存放于@llNowInstanceID
     * 和@sLastFileID中
     */
    int ret = poDatabase->GetMaxInstanceIDFileID(sLastFileID, llNowInstanceID);
    if (ret != 0)
    {
        return ret;
    }

    int iFileID = 0;
    int iOffset = 0;
    uint32_t iCheckSum = 0;

    if (sLastFileID.size() > 0)
    {
        /* 从@sLastFileID中解析出@iFileID、@iOffset和@iCheckSum，其实@sLastFileID就是
         * 由@iFileID、@iOffset和@iCheckSum这三者组合而成的
         */
        ParseFileID(sLastFileID, iFileID, iOffset, iCheckSum);
    }

    /* 在LogStore::Init中会尝试从meta文件中获取@m_iFileID，如果获取不到，则设置为0.
     * leveldb中记录的最大的instance ID对应的@iFileID不应该大于@m_iFileID
     */
    if (iFileID > m_iFileID)
    {
        PLG1Err("LevelDB last fileid %d larger than meta now fileid %d, file error",
                iFileID, m_iFileID);
        return -2;
    }

    PLG1Head("START fileid %d offset %d checksum %u", iFileID, iOffset, iCheckSum);
    /*从@iFileID所代表的文件开始逐一重建索引*/
    for (int iNowFileID = iFileID; ;iNowFileID++)
    {
        /* 重建当前@iNowFileID所代表的文件相关的索引，会返回已经重建的最大的instance
         * ID，存放于@llNowInstanceID中
         */
        ret = RebuildIndexForOneFile(iNowFileID, iOffset, poDatabase, iNowFileWriteOffset, llNowInstanceID);
        if (ret != 0 && ret != 1)
        {
            /*重建失败*/
            break;
        }
        else if (ret == 1)
        {
            /* 文件不存在的处理（@m_iFileID + 1之后的文件应该不存在，因为@m_iFileID中记录
             * 的是当前系统中用到的最大FileID）
             */
            
            /* @iNowFileID不为0，表示不是新建的系统，如果当前被处理的文件不存在，则应该是
             * @m_iFileID + 1这个文件，否则meta文件中记录的信息可能有误
             */
            if (iNowFileID != 0 && iNowFileID != m_iFileID + 1)
            {
                PLG1Err("meta file wrong, nowfileid %d meta.nowfileid %d", iNowFileID, m_iFileID);
                return -1;
            }

            /*重建完毕，结束于@m_iFileID这个文件*/
            ret = 0;
            PLG1Imp("END rebuild ok, nowfileid %d", iNowFileID);
            break;
        }

        /*重建下一个文件相关的索引之前重置@iOffset为0*/
        iOffset = 0;
    }
    
    return ret;
}

int LogStore :: RebuildIndexForOneFile(const int iFileID, const int iOffset, 
        Database * poDatabase, int & iNowFileWriteOffset, uint64_t & llNowInstanceID)
{
    char sFilePath[512] = {0};
    snprintf(sFilePath, sizeof(sFilePath), "%s/%d.f", m_sPath.c_str(), iFileID);

    int ret = access(sFilePath, F_OK);
    if (ret == -1)
    {
        /*文件不存在，返回1*/
        PLG1Debug("file not exist, filepath %s", sFilePath);
        return 1;
    }

    /*打开文件，返回文件句柄@iFd*/
    int iFd = -1;
    ret = OpenFile(iFileID, iFd);
    if (ret != 0)
    {
        return ret;
    }

    /*将文件读写指针定位到文件末尾，可以借此获取文件长度*/
    int iFileLen = lseek(iFd, 0, SEEK_END);
    if (iFileLen == -1)
    {
        /*lseek返回错误*/
        close(iFd);
        return -1;
    }
    
    off_t iSeekPos = lseek(iFd, iOffset, SEEK_SET);
    if (iSeekPos == -1)
    {
        /*lseek返回错误*/
        close(iFd);
        return -1;
    }

    /*至此，lseek返回成功*/
    int iNowOffset = iOffset;
    bool bNeedTruncate = false;

    while (true)
    {
        int iLen = 0;
        /* 首先读取sizeof(int)字节，其中存放的是当前重建的instance相关的数据（包括
         * instance ID和AcceptorStateData信息）长度
         */
        ssize_t iReadLen = read(iFd, (char *)&iLen, sizeof(int));
        if (iReadLen == 0)
        {
            PLG1Head("File End, fileid %d offset %d", iFileID, iNowOffset);
            iNowFileWriteOffset = iNowOffset;
            break;
        }
        
        /*读取出错*/
        if (iReadLen != (ssize_t)sizeof(int))
        {
            bNeedTruncate = true;
            PLG1Err("readlen %zd not qual to %zu, need truncate", iReadLen, sizeof(int));
            break;
        }

        /*到达文件尾*/
        if (iLen == 0)
        {
            PLG1Head("File Data End, fileid %d offset %d", iFileID, iNowOffset);
            iNowFileWriteOffset = iNowOffset;
            break;
        }

        /* 当前被重建的instance相关的数据长度比文件长度大，或者比instance ID占用的字节数
         * 要少，都是不正常的情况
         */
        if (iLen > iFileLen || iLen < (int)sizeof(uint64_t))
        {
            PLG1Err("File data len wrong, data len %d filelen %d",
                    iLen, iFileLen);
            ret = -1;
            break;
        }

        /*准备buffer @m_oTmpBuffer，用于存放接下来的长度为@iLen的数据*/
        m_oTmpBuffer.Ready(iLen);
        /*读取接下来的长度为@iLen的数据，并存放于@m_oTmpBuffer中*/
        iReadLen = read(iFd, m_oTmpBuffer.GetPtr(), iLen);
        if (iReadLen != iLen)
        {
            bNeedTruncate = true;
            PLG1Err("readlen %zd not qual to %zu, need truncate", iReadLen, iLen);
            break;
        }

        /*从@m_oTmpBuffer中中解析出instance ID*/
        uint64_t llInstanceID = 0;
        memcpy(&llInstanceID, m_oTmpBuffer.GetPtr(), sizeof(uint64_t));

        /*Instance ID 一定是升序的*/
        if (llInstanceID < llNowInstanceID)
        {
            PLG1Err("File data wrong, read instanceid %lu smaller than now instanceid %lu",
                    llInstanceID, llNowInstanceID);
            ret = -1;
            break;
        }

        /*更新当前重建的最大instance ID，记录于@llNowInstanceID中*/        
        llNowInstanceID = llInstanceID;

        /*从@m_oTmpBuffer中中解析出AcceptorStateData信息*/
        AcceptorStateData oState;
        bool bBufferValid = oState.ParseFromArray(m_oTmpBuffer.GetPtr() + sizeof(uint64_t), iLen - sizeof(uint64_t));
        if (!bBufferValid)
        {
            /*解析出错*/
            m_iNowFileOffset = iNowOffset;
            PLG1Err("This instance's buffer wrong, can't parse to acceptState, instanceid %lu bufferlen %d nowoffset %d",
                    llInstanceID, iLen - sizeof(uint64_t), iNowOffset);
            bNeedTruncate = true;
            break;
        }

        /*计算@m_oTmpBuffer中数据的checksum*/
        uint32_t iFileCheckSum = crc32(0, (const uint8_t *)m_oTmpBuffer.GetPtr(), iLen, CRC32SKIP);

        /*通过@iFileID、@iNowOffset和@iFileCheckSum产生@sFileID*/
        string sFileID;
        GenFileID(iFileID, iNowOffset, iFileCheckSum, sFileID);

        /*将@llInstanceID和对应的@sFileID信息存储于leveldb中*/
        ret = poDatabase->RebuildOneIndex(llInstanceID, sFileID);
        if (ret != 0)
        {
            break;
        }

        PLG1Imp("rebuild one index ok, fileid %d offset %d instanceid %lu checksum %u buffer size %zu", 
                iFileID, iNowOffset, llInstanceID, iFileCheckSum, iLen - sizeof(uint64_t));

        /*更新@iNowOffset，为后续重建做准备*/
        iNowOffset += sizeof(int) + iLen; 
    }
    
    close(iFd);

    if (bNeedTruncate)
    {
        m_oFileLogger.Log("truncate fileid %d offset %d filesize %d", 
                iFileID, iNowOffset, iFileLen);
        if (truncate(sFilePath, iNowOffset) != 0)
        {
            PLG1Err("truncate fail, file path %s truncate to length %d errno %d", 
                    sFilePath, iNowOffset, errno);
            return -1;
        }
    }

    return ret;
}

void LogStore :: GenFileID(const int iFileID, const int iOffset, const uint32_t iCheckSum, std::string & sFileID)
{
    char sTmp[sizeof(int) + sizeof(int) + sizeof(uint32_t)] = {0};
    memcpy(sTmp, (char *)&iFileID, sizeof(int));
    memcpy(sTmp + sizeof(int), (char *)&iOffset, sizeof(int));
    memcpy(sTmp + sizeof(int) + sizeof(int), (char *)&iCheckSum, sizeof(uint32_t));

    sFileID = std::string(sTmp, sizeof(int) + sizeof(int) + sizeof(uint32_t));
}

```

### 向LogStore中追加写某个Instance相关的数据，并返回本次写入对应的FileID

```
int LogStore :: Append(const WriteOptions & oWriteOptions, const uint64_t llInstanceID, const std::string & sBuffer, std::string & sFileID)
{
    m_oTimeStat.Point();
    std::lock_guard<std::mutex> oLock(m_oMutex);

    int iFd = -1;
    int iFileID = -1;
    int iOffset = -1;

    /*InstanceID的长度 + @sBuffer的大小*/
    int iLen = sizeof(uint64_t) + sBuffer.size();
    /*@m_oTmpAppendBuffer中还要存放@iLen信息*/
    int iTmpBufferLen = iLen + sizeof(int);

    /* 根据要写入的数据总大小来确定本次写应该写入的文件（比如当前正在使用的文件
     * 已经无法容纳@iTmpBufferLen这么多字节的数据了，因为LogStore中每个文件的大
     * 小都不能超过LOG_FILE_MAX_SIZE，就需要切换到下一个文件）
     * 参考“获取LogStore中本次写应该写入的文件句柄及其位置信息”
     */
    int ret = GetFileFD(iTmpBufferLen, iFd, iFileID, iOffset);
    if (ret != 0)
    {
        return ret;
    }

    /*准备好大小为@iTmpBufferLen的buffer*/
    m_oTmpAppendBuffer.Ready(iTmpBufferLen);

    /* 依次将即将写入的数据的长度信息@iLen，InstanceID @llInstanceID，要写的数
     * 据@sBuffer拷贝到@m_oTmpAppendBuffer中
     */
    memcpy(m_oTmpAppendBuffer.GetPtr(), &iLen, sizeof(int));
    memcpy(m_oTmpAppendBuffer.GetPtr() + sizeof(int), &llInstanceID, sizeof(uint64_t));
    memcpy(m_oTmpAppendBuffer.GetPtr() + sizeof(int) + sizeof(uint64_t), sBuffer.c_str(), sBuffer.size());

    /*写入@m_oTmpAppendBuffer中的内容到文件中*/
    size_t iWriteLen = write(iFd, m_oTmpAppendBuffer.GetPtr(), iTmpBufferLen);

    if (iWriteLen != (size_t)iTmpBufferLen)
    {
        BP->GetLogStorageBP()->AppendDataFail();
        PLG1Err("writelen %d not equal to %d, buffersize %zu errno %d", 
                iWriteLen, iTmpBufferLen, sBuffer.size(), errno);
        return -1;
    }

    /*如果需要同步，则同步之*/
    if (oWriteOptions.bSync)
    {
        int fdatasync_ret = fdatasync(iFd);
        if (fdatasync_ret == -1)
        {
            PLG1Err("fdatasync fail, writelen %zu errno %d", iWriteLen, errno);
            return -1;
        }
    }

    /*更新当前文件下一次写入的位置*/
    m_iNowFileOffset += iWriteLen;

    int iUseTimeMs = m_oTimeStat.Point();
    BP->GetLogStorageBP()->AppendDataOK(iWriteLen, iUseTimeMs);
    
    /*计算@llInstanceID和@sBuffer一起的checksum*/
    uint32_t iCheckSum = crc32(0, (const uint8_t*)(m_oTmpAppendBuffer.GetPtr() + sizeof(int)), iTmpBufferLen - sizeof(int), CRC32SKIP);

    /* 根据本次写入对应的文件ID @iFileID，文件偏移@iOffset和校验和@iChecksum生成
     * 本次实例@llInstanceID对应的FileID
     */
    GenFileID(iFileID, iOffset, iCheckSum, sFileID);

    PLG1Imp("ok, offset %d fileid %d checksum %u instanceid %lu buffer size %zu usetime %dms sync %d",
            iOffset, iFileID, iCheckSum, llInstanceID, sBuffer.size(), iUseTimeMs, (int)oWriteOptions.bSync);

    return 0;
}
```

### 从LogStore中读取某个Instance相关的数据

```
int LogStore :: Read(const std::string & sFileID, uint64_t & llInstanceID, std::string & sBuffer)
{
    int iFileID = -1;
    int iOffset = -1;
    uint32_t iCheckSum = 0;
    /*从@sFileID中解析出来该实例所在的文件ID、文件偏移和checksum信息*/
    ParseFileID(sFileID, iFileID, iOffset, iCheckSum);

    /*打开相应的文件@iFileID*/    
    int iFd = -1;
    int ret = OpenFile(iFileID, iFd);
    if (ret != 0)
    {
        return ret;
    }
    
    /*定位到该实例信息所在的偏移@iOffset*/
    off_t iSeekPos = lseek(iFd, iOffset, SEEK_SET);
    if (iSeekPos == -1)
    {
        return -1;
    }
    
    /*读取该实例相关信息的长度（包括InstanceID和信息内容）*/
    int iLen = 0;
    ssize_t iReadLen = read(iFd, (char *)&iLen, sizeof(int));
    if (iReadLen != (ssize_t)sizeof(int))
    {
        close(iFd);
        PLG1Err("readlen %zd not qual to %zu", iReadLen, sizeof(int));
        return -1;
    }
    
    std::lock_guard<std::mutex> oLock(m_oReadMutex);
    /*准备一个buffer，用于存放即将读取的数据*/
    m_oTmpBuffer.Ready(iLen);
    iReadLen = read(iFd, m_oTmpBuffer.GetPtr(), iLen);
    if (iReadLen != iLen)
    {
        close(iFd);
        PLG1Err("readlen %zd not qual to %zu", iReadLen, iLen);
        return -1;
    }

    close(iFd);

    /*计算校验和并比较和记录在FileID中的校验和是否一致*/
    uint32_t iFileCheckSum = crc32(0, (const uint8_t *)m_oTmpBuffer.GetPtr(), iLen, CRC32SKIP);
    if (iFileCheckSum != iCheckSum)
    {
        BP->GetLogStorageBP()->GetFileChecksumNotEquel();
        PLG1Err("checksum not equal, filechecksum %u checksum %u", iFileCheckSum, iCheckSum);
        return -2;
    }

    /*从读取的内容中解析出InstanceID和对应的value*/
    memcpy(&llInstanceID, m_oTmpBuffer.GetPtr(), sizeof(uint64_t));
    sBuffer = string(m_oTmpBuffer.GetPtr() + sizeof(uint64_t), iLen - sizeof(uint64_t));

    PLG1Imp("ok, fileid %d offset %d instanceid %lu buffer size %zu", 
            iFileID, iOffset, llInstanceID, sBuffer.size());

    return 0;
}
```

### 获取LogStore中本次写应该写入的文件句柄及其位置信息

```
int LogStore :: GetFileFD(const int iNeedWriteSize, int & iFd, int & iFileID, int & iOffset)
{
    /*@m_iFd在LogStore::Init中已经初始化为非-1的值了*/
    if (m_iFd == -1)
    {
        PLG1Err("File aready broken, fileid %d", m_iFileID);
        return -1;
    }

    /*找到文件当前的写指针的位置*/
    iOffset = lseek(m_iFd, m_iNowFileOffset, SEEK_SET);
    assert(iOffset != -1);

    /*剩余空间不足以容纳@iNeedWriteSize大小的数据*/
    if (iOffset + iNeedWriteSize > m_iNowFileSize)
    {
        /*关闭当前文件，重新打开新的文件*/
        
        /*关闭当前文件*/
        close(m_iFd);
        m_iFd = -1;

        /*获取下一个File ID，并写入到meta文件中*/
        int ret = IncreaseFileID();
        if (ret != 0)
        {
            m_oFileLogger.Log("new file increase fileid fail, now fileid %d", m_iFileID);
            return ret;
        }

        /*打开新的文件*/
        ret = OpenFile(m_iFileID, m_iFd);
        if (ret != 0)
        {
            m_oFileLogger.Log("new file open file fail, now fileid %d", m_iFileID);
            return ret;
        }

        iOffset = lseek(m_iFd, 0, SEEK_END);
        if (iOffset != 0)
        {
            assert(iOffset != -1);

            m_oFileLogger.Log("new file but file aready exist, now fileid %d exist filesize %d", 
                    m_iFileID, iOffset);

            PLG1Err("IncreaseFileID success, but file exist, data wrong, file size %d", iOffset);
            assert(false);
            return -1;
        }

        /*将文件扩大到LOG_FILE_MAX_SIZE大小*/
        ret = ExpandFile(m_iFd, m_iNowFileSize);
        if (ret != 0)
        {
            PLG1Err("new file expand fail, fileid %d fd %d", m_iFileID, m_iFd);

            m_oFileLogger.Log("new file expand file fail, now fileid %d", m_iFileID);

            close(m_iFd);
            m_iFd = -1;
            return -1;
        }

        m_oFileLogger.Log("new file expand ok, fileid %d filesize %d", m_iFileID, m_iNowFileSize);
    }

    /*返回打开的文件句柄和文件ID给上层调用*/
    iFd = m_iFd;
    iFileID = m_iFileID;

    return 0;
}
```

### 从LogStore中获取下一个File ID并写入Meta文件中

```
int LogStore :: IncreaseFileID()
{
    /*增长当前的@m_iFileID作为下一个FileID，并计算其checksum*/
    int iFileID = m_iFileID + 1;
    uint32_t iCheckSum = crc32(0, (const uint8_t*)(&iFileID), sizeof(int));

    /*定位meta文件的读写指针到文件开头*/
    off_t iSeekPos = lseek(m_iMetaFd, 0, SEEK_SET);
    if (iSeekPos == -1)
    {
        return -1;
    }

    /*写入新的FileID信息*/
    size_t iWriteLen = write(m_iMetaFd, (char *)&iFileID, sizeof(int));
    if (iWriteLen != sizeof(int))
    {
        PLG1Err("write meta fileid fail, writelen %zu", iWriteLen);
        return -1;
    }

    /*写入checksum信息*/
    iWriteLen = write(m_iMetaFd, (char *)&iCheckSum, sizeof(uint32_t));
    if (iWriteLen != sizeof(uint32_t))
    {
        PLG1Err("write meta checksum fail, writelen %zu", iWriteLen);
        return -1;
    }

    /*同步，确保持久化*/
    int ret = fsync(m_iMetaFd);
    if (ret != 0)
    {
        return -1;
    }

    /* 正式更新@m_iFileID，前面首先将增长后的@m_iFileID信息持久化到meta文件中，
     * 然后再更新内存中记录的@m_iFileID
     */
    m_iFileID++;

    return 0;
}
```

### 从LogStore中删除给定实例@llInstanceID的数据
```
int LogStore :: Del(const std::string & sFileID, const uint64_t llInstanceID)
{
    int iFileID = -1;
    int iOffset = -1;
    uint32_t iCheckSum = 0;
	/*从给定的@sFileID中解析出文件ID，文件内偏移，checksum*/
    ParseFileID(sFileID, iFileID, iOffset, iCheckSum);

	/*@m_iFileID是LogStore中正在被使用的最大的文件ID*/
    if (iFileID > m_iFileID)
    {
        PLG1Err("del fileid %d large than useing fileid %d", iFileID, m_iFileID);
        return -2;
    }

    if (iFileID > 0)
    {
		/*删除@iFileID所代表的文件的前一个文件*/
        return DeleteFile(iFileID - 1);
    }

    return 0;
}

int LogStore :: DeleteFile(const int iFileID)
{
	/*@m_iDeletedMaxFileID记录已经被删除的最大的文件ID*/
    if (m_iDeletedMaxFileID == -1)
    {
        if (iFileID - 2000 > 0)
        {
            m_iDeletedMaxFileID = iFileID - 2000;
        }
    }

    if (iFileID <= m_iDeletedMaxFileID)
    {
        PLG1Debug("file already deleted, fileid %d deletedmaxfileid %d", iFileID, m_iDeletedMaxFileID);
        return 0;
    }
    
	/*删除从@m_iDeletedMaxFileID + 1开始，到@iFileID之间的所有文件*/
    int ret = 0;
    for (int iDeleteFileID = m_iDeletedMaxFileID + 1; iDeleteFileID <= iFileID; iDeleteFileID++)
    {
        char sFilePath[512] = {0};
        snprintf(sFilePath, sizeof(sFilePath), "%s/%d.f", m_sPath.c_str(), iDeleteFileID);

		/*检查文件是否存在*/
        ret = access(sFilePath, F_OK);
        if (ret == -1)
        {
            PLG1Debug("file already deleted, filepath %s", sFilePath);
            m_iDeletedMaxFileID = iDeleteFileID;
            ret = 0;
            continue;
        }

		/*删除之*/
        ret = remove(sFilePath);
        if (ret != 0)
        {
            PLG1Err("remove fail, filepath %s ret %d", sFilePath, ret);
            break;
        }
        
		/*更新已删除的最大文件ID*/
        m_iDeletedMaxFileID = iDeleteFileID;
        m_oFileLogger.Log("delete fileid %d", iDeleteFileID);
    }

    return ret;
}
```

# PaxosLog
## PaxosLog基础

PaxosLog中直接封装了LogStorage指针，而LogStorage中定义了存储必须提供的接口，PaxosLog中提供的方法也是对LogStorage中接口的封装。通常的使用都是某个子类如MultiDatabase继承自LogStorage，并实现LogStorage中定义的接口，然后将MultiDatabase子类的实例传递给PaxosLog的构造函数，然后就可以调用PaxosLog中的方法了，这样实际上调用的是MultiDatabase这个子类的方法。

```
class PaxosLog
{
public:
    PaxosLog(const LogStorage * poLogStorage);
    ~PaxosLog();

    int WriteLog(const WriteOptions & oWriteOptions, const int iGroupIdx, const uint64_t llInstanceID, const std::string & sValue);

    int ReadLog(const int iGroupIdx, const uint64_t llInstanceID, std::string & sValue);
    
    int GetMaxInstanceIDFromLog(const int iGroupIdx, uint64_t & llInstanceID);

    int WriteState(const WriteOptions & oWriteOptions, const int iGroupIdx, const uint64_t llInstanceID, const AcceptorStateData & oState);

    int ReadState(const int iGroupIdx, const uint64_t llInstanceID, AcceptorStateData & oState);
    
private:
    LogStorage * m_poLogStorage;
};
```

## PaxosLog方法
### 将指定paxos group的指定Instance的已经accepted value记录到storage中

```
int PaxosLog :: WriteLog(const WriteOptions & oWriteOptions, const int iGroupIdx, const uint64_t llInstanceID, const std::string & sValue)
{
    const int m_iMyGroupIdx = iGroupIdx;

    AcceptorStateData oState;
    oState.set_instanceid(llInstanceID);
    oState.set_acceptedvalue(sValue);
    oState.set_promiseid(0);
    oState.set_promisenodeid(nullnode);
    oState.set_acceptedid(0);
    oState.set_acceptednodeid(nullnode);

    /*写入指定paxos group的指定instance的AcceptorState信息到storage中*/
    int ret = WriteState(oWriteOptions, iGroupIdx, llInstanceID, oState);
    if (ret != 0)
    {
        PLG1Err("WriteState to db fail, groupidx %d instanceid %lu ret %d", iGroupIdx, llInstanceID, ret);
        return ret;
    }

    PLG1Imp("OK, groupidx %d InstanceID %lu valuelen %zu", 
            iGroupIdx, llInstanceID, sValue.size());

    return 0;
}
```

### 从storage中读取指定paxos group的指定Instance的已经accepted value

```
int PaxosLog :: ReadLog(const int iGroupIdx, const uint64_t llInstanceID, std::string & sValue)
{
    const int m_iMyGroupIdx = iGroupIdx;

    AcceptorStateData oState;
    /*从storage中读取指定paxos group的指定instance的AcceptorState信息*/
    int ret = ReadState(iGroupIdx, llInstanceID, oState); 
    if (ret != 0)
    {
        PLG1Err("ReadState from db fail, groupidx %d instanceid %lu ret %d", 
                iGroupIdx, llInstanceID, ret);
        return ret;

    }

    /*从AcceptorState中解析出@sValue*/
    sValue = oState.acceptedvalue();

    PLG1Imp("OK, groupidx %d InstanceID %lu value %zu", 
            iGroupIdx, llInstanceID, sValue.size());

    return 0;
}
```

### 写入指定paxos group的指定instance的AcceptorState信息到storage中

```
int PaxosLog :: WriteState(const WriteOptions & oWriteOptions, const int iGroupIdx, const uint64_t llInstanceID, const AcceptorStateData & oState)
{
    const int m_iMyGroupIdx = iGroupIdx;

    string sBuffer;
    /*将AcceptorStateData序列化为字符串*/
    bool sSucc = oState.SerializeToString(&sBuffer);
    if (!sSucc)
    {
        PLG1Err("State.Serialize fail");
        return -1;
    }
    
    /*如果m_poLogStorage采用的是MultiDatabase的实例初始化的，则调用MultiDatabase::Put*/
    int ret = m_poLogStorage->Put(oWriteOptions, iGroupIdx, llInstanceID, sBuffer);
    if (ret != 0)
    {
        PLG1Err("DB.Put fail, groupidx %d bufferlen %zu ret %d", 
                iGroupIdx, sBuffer.size(), ret);
        return ret;
    }

    return 0;
}
```

### 从storage中读取指定paxos group的指定instance的AcceptorState信息
```
int PaxosLog :: ReadState(const int iGroupIdx, const uint64_t llInstanceID, AcceptorStateData & oState)
{
    const int m_iMyGroupIdx = iGroupIdx;

    string sBuffer;
    /*如果m_poLogStorage采用的是MultiDatabase的实例初始化的，则调用MultiDatabase::Put*/
    int ret = m_poLogStorage->Get(iGroupIdx, llInstanceID, sBuffer);
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

    /*从@sBuffer中解析出AcceptorStateData*/
    bool bSucc = oState.ParseFromArray(sBuffer.data(), sBuffer.size());
    if (!bSucc)
    {
        PLG1Err("State.ParseFromArray fail, bufferlen %zu", sBuffer.size());
        return -1;
    }

    return 0;
}
```

### 从storage中获取指定paxos group的最大InstanceID
```
int PaxosLog :: GetMaxInstanceIDFromLog(const int iGroupIdx, uint64_t & llInstanceID)
{
    const int m_iMyGroupIdx = iGroupIdx;

    /*如果m_poLogStorage采用的是MultiDatabase的实例初始化的，则调用MultiDatabase::GetMaxInstanceID*/
    int ret = m_poLogStorage->GetMaxInstanceID(iGroupIdx, llInstanceID);
    if (ret != 0 && ret != 1)
    {
        PLG1Err("DB.GetMax fail, groupidx %d ret %d", iGroupIdx, ret);
    }
    else if (ret == 1)
    {
        PLG1Debug("MaxInstanceID not exist, groupidx %d", iGroupIdx);
    }
    else
    {
        PLG1Imp("OK, MaxInstanceID %llu groupidsx %d", llInstanceID, iGroupIdx);
    }

    return ret;
}
```

