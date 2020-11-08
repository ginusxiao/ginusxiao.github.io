# 提纲
[toc]

## 演讲
### [TiDB with TiFlash Extension](https://myslide.cn/slides/19950)
1. 在OLTP之上实现OLAP和在OLAP之上实现OLTP存在的问题：
- 一种存储格式很难同时兼顾两种访问模式，OLTP侧重于行存，而OLAP侧重于列存；
- 一套系统上两种工作负载相互干扰，一个较大的长查询可能对于OLTP来说是一个灾难；
- HTAP的核心价值：解决当前各类数据平台上广泛存在的工具链过于复杂，运维成本高，数据时效性和一致性等问题；

2. TiFlash是什么？
- TiDB的分析引擎
    - 列存，向量化处理
    - 基于ClickHouse，做了数十处改进
- 通过扩展的Raft一致性协议来进行数据同步
- 工作负载隔离，物理资源隔离（借助于label来标记节点TiFlash节点还是TiKV节点），不影响OLTP
- 和TiDB紧密集成

3. TiFlash架构

    [TiFlash Architecture](https://note.youdao.com/yws/public/resource/120c035709eb4fb7e29ecb41a653292d/xmlnote/5EFBEC5D46C44798A7965B894B468238/81225)

4. TiDB是如何支持TiFlash的？

    [TiFlash replication](https://note.youdao.com/yws/public/resource/120c035709eb4fb7e29ecb41a653292d/xmlnote/2B8344959F404725AA7B63B92443AA89/81261)

- TiFlash通过raft learner（TiFlash伪装成一个TiKV节点加入TiKV raft group）来异步的从TiKV复制数据，raft learner是只读的；
- TiFlash暂不支持来自SQL端（TiDB或者TiSpark）的直接写入，稍后会支持；
- 写操作无需等待raft learner复制完数据即可返回（Raft协议只要记录背负知道了Raft Leader和Raft Follower中的多数节点，就认为已经成功commit了，逐一这里多数节点是不包括Raft Learner的，因此Learner上的数据可能会有延迟）；
- 强一致性：读请求发起的时候获取一个全局的时间戳read timestamp，然后确保Raft Learner上的副本数据已经足够新，就可以找出所有commit timestamp <= read timestamp的所有版本中commit timestamp最大的版本的数据就是尧都区的数据，那么如何确保Raft Learner上的副本数据足够新呢？Raft Leaner在读数据之前，会带上read timestamp向Raft Leader发送一次请求，获取确保Raft Learner上数据足够新的Raft Log的偏移量，然后TiFlash等待本地副本数据同步到足够新，直到超时，当然这是目前的策略，后面可能会添加其它策略，比如主动要求同步数据，主动要求同步数据的示意图如下：

    [TiFlash Learner Read 1/2](https://note.youdao.com/yws/public/resource/120c035709eb4fb7e29ecb41a653292d/xmlnote/D42A8DFA53804583B961658D8846BC9F/81291)

    [TiFlash Learner Read 2/2](https://note.youdao.com/yws/public/resource/120c035709eb4fb7e29ecb41a653292d/xmlnote/90A2AA513DD94479801C5C06F2534612/81294)

5. TiFlash上数据是如何更新的
- 要求实时、事务性的更新
- 列式存储更新相对困难
    - 列存往往使用块压缩
    - 且列存更容易引起很多小IO
    - 另外，要保证高速更新的同时满足AP业务需要大量SCAN操作的要求
- 目前TiFlash使用类LSM-Tree的存储架构（基于ClickHouse的MergeTree改进的MutableMergeTree架构）
    - 在内存中采用行存
    - 在持久化存储中采用列存
- 使用MVCC来实现Snapshot Isolation隔离级别

6. 未来工作
- TiFlash SQL MPP计算下推

    [TiFlash SQL MPP push down](https://note.youdao.com/yws/public/resource/120c035709eb4fb7e29ecb41a653292d/xmlnote/1D8605E5427048138D9E745DC286A3C5/81328)
    
- 性能提升： 
    
    新的存储引擎正在研发过程中，约3倍的性能加速

7. TiFlash所带来的优势
- 架构简化
    - 同一套平台覆盖多个场景
    - 统一的运维
- 同一份数据的另一个副本
    - 无需数据转移和复杂的增量合并流程
    - 行存 + 索引 -> 高并发短查询
    - 列存 + 向量化引擎 -> 低并发快速批量扫表
    - 完整的资源隔离
    

### 疑问
TiFlash是以ClickHouse作为基础开发的，那么为什么它不支持SQL端（TiSpark或者TiDB）的直接写入呢？ClickHouse是支持的？