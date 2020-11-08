# 提纲
[toc]

# Paxos
[wiki](https://en.wikipedia.org/wiki/Paxos_(computer_science))

[Paxos made simple](https://github.com/oldratlee/translations/tree/master/paxos-made-simple)

[Paxos made live](http://research.google.com/pubs/pub33002.html)

[Paxos - made moderately complex](http://paxos.systems/replica.html)

[Paxos made code](http://www.inf.usi.ch/faculty/pedone/MScThesis/marco.pdf)

[Paxos理论介绍(1): 朴素Paxos算法理论推导与证明](https://zhuanlan.zhihu.com/p/21438357?refer=lynncui)

[Paxos理论介绍(2): Multi-Paxos与Leader](https://zhuanlan.zhihu.com/p/21466932?refer=lynncui)

[Paxos理论介绍(3): Master选举](https://zhuanlan.zhihu.com/p/21540239)

[Paxos理论介绍(4): 动态成员变更](https://zhuanlan.zhihu.com/p/22148265)

[架构师需要了解的Paxos原理、历程及实战](https://mp.weixin.qq.com/s?__biz=MzAwMDU1MTE1OQ==&mid=403582309&idx=1&sn=80c006f4e84a8af35dc8e9654f018ace&3rd=MzA3MDU4NTYzMw==&scene=6#rd)

```
本文中关于Paxos的两个阶段的讲解比较通俗易懂，粘贴如下：

1、第一阶段 Prepare
P1a：Proposer 发送 Prepare
Proposer 生成全局唯一且递增的提案 ID（Proposalid，以高位时间戳 + 低位机器 IP 可以保证唯一性和递增性），向 Paxos 集群的所有机器发送 PrepareRequest，这里无需携带提案内容，只携带 Proposalid 即可。

P1b：Acceptor 应答 Prepare
Acceptor 收到 PrepareRequest 后，做出“两个承诺，一个应答”。

两个承诺：
第一，不再应答 Proposalid 小于等于（注意：这里是 <= ）当前请求的 PrepareRequest；
第二，不再应答 Proposalid 小于（注意：这里是 < ）当前请求的 AcceptRequest

一个应答：
返回自己已经 Accept 过的提案中 ProposalID 最大的那个提案的内容，如果没有则返回空值;

注意：这“两个承诺”中，蕴含两个要点：

就是应答当前请求前，也要按照“两个承诺”检查是否会违背之前处理 PrepareRequest 时做出的承诺；
应答前要在本地持久化当前 Propsalid。

2、第二阶段 Accept
P2a：Proposer 发送 Accept
“提案生成规则”：Proposer 收集到多数派应答的 PrepareResponse 后，从中选择proposalid最大的提案内容，作为要发起 Accept 的提案，如果这个提案为空值，则可以自己随意决定提案内容。然后携带上当前 Proposalid，向 Paxos 集群的所有机器发送 AccpetRequest。

P2b：Acceptor 应答 Accept
Accpetor 收到 AccpetRequest 后，检查不违背自己之前作出的“两个承诺”情况下，持久化当前 Proposalid 和提案内容。最后 Proposer 收集到多数派应答的 AcceptResponse 后，形成决议。

这里的“两个承诺”很重要，后面也会提及，请大家细细品味。
```


[使用Basic-Paxos协议的日志同步与恢复](http://oceanbase.org.cn/?p=90)

[使用Multi-Paxos协议的日志同步与恢复](http://oceanbase.org.cn/?p=111)

[Paxos成员组变更](http://oceanbase.org.cn/?p=160)

[如何浅显易懂地解说 Paxos 的算法？](https://www.jianshu.com/p/06a477a576bf)


# PhxPaxos
[微信自研生产级paxos类库PhxPaxos实现原理介绍](https://mp.weixin.qq.com/s?__biz=MzI4NDMyNTU2Mw==&mid=2247483695&idx=1&sn=91ea422913fc62579e020e941d1d059e#rd)

[PhxPaxos code reading note](https://github.com/Tencent/phxpaxos/blob/master/doc/pdf/code_reading_note.pdf)

[PhxPaxos架构设计、实现分析](https://zhuanlan.zhihu.com/p/26329762)

[PhxPaxos源码分析——网络](http://linbingdong.com/2017/11/20/PhxPaxos%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E7%BD%91%E7%BB%9C/)

[PhxPaxos源码分析——Paxos算法实现](http://linbingdong.com/2017/11/21/PhxPaxos%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E2%80%94%E2%80%94Paxos%E7%AE%97%E6%B3%95%E5%AE%9E%E7%8E%B0/)



