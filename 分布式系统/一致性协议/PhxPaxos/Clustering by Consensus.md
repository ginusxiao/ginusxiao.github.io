# Outline
[toc]

# Reference
[源码](https://github.com/djmitche/500lines/tree/master/cluster)

[原文](http://aosabook.org/en/500L/clustering-by-consensus.html)

# Introduction
本文将会探索旨在支持高可靠分布式计算的网络协议。

# Motivating Example
首先以银行账户管理系统为例，来阐述实现该分布式网络协议的动机。在银行账户管理系统中，每一个账户都有一个账户编号及其当前账目信息，用户可以对其账户执行查询、存款和转账等操作。转账操作会涉及两个账户，转账的源账户和目标账户。

如果该系统搭建在单节点服务器上，则实现转账功能会非常容易：通过锁来控制转账操作的并发即可。但是银行不能仅仅依赖于单节点服务器来管理其至关重要的用户账户信息，取而代之的是，银行账户管理系统运行于多节点服务器上，每一个节点上运行单独的实例，用户可以通过任何一个节点执行账户操作。

一种简单的实现方式为：每个节点都保存每个账户的一个副本，这样每个节点都可以处理账户操作，并且将账户更新操作（而不是账目信息）发送给其它节点。但是这种方式引入了一个非常严重的错误：如果两个节点同时处理关于同一个账户的操作，那么哪个账户的丈母信息会是正确的呢？虽然每一个节点都会向其它节点发送账户操作，但是两个同时进行的转账操作可能透支该账户（账户A中余额为1000，节点M收到转账给账户B 600 元的请求，与此同时，节点N收到转账给账户C 800元的请求，那么节点M和节点N分别可以完成该操作，但是当节点M/节点N收到来自于节点N/节点M转发过来的更新请求时，账户余额会变为-400，账户透支了），但是在银行账户管理系统中，透支应该通常是不被允许的。

产生这种错误的原因是每个节点都根据它本地的状态来执行账户操作，没有事先确保本地状态和其它节点状态一致。

# Distributed State Machines
解决上述问题的技术被称作为“分布式状态机”，基本思想是每个节点在相同的状态机上执行相同的输入，则每个节点都会看到相同的输出。

一个简单的银行账户管理系统的状态机如下：
```
    def execute_operation(state, operation):
        if operation.name == 'deposit':
            if not verify_signature(operation.deposit_signature):
                return state, False
            state.accounts[operation.destination_account] += operation.amount
            return state, True
        elif operation.name == 'transfer':
            if state.accounts[operation.source_account] < operation.amount:
                return state, False
            state.accounts[operation.source_account] -= operation.amount
            state.accounts[operation.destination_account] += operation.amount
            return state, True
        elif operation.name == 'get-balance':
            return state, state.accounts[operation.account]
```

这和计算机科学课程中学习的典型的状态机有所不同，在计算机科学中学习的状态机是带转换标签的有限状态集合的有限状态机，这里的状态机中的状态则是所有用户的账目信息，可能有无限多的可能状态。但是有限状态机的规则仍然适用：基于同样的状态，进行相同的操作会产生相同的输出。

因此分布式状态机技术确保每一个节点上发生同样的操作，但是如何确保每一个节点在状态机的输入上达成一致呢，这个问题被称为分布式共识（consensus），将采用Paxos算法解决之。

# Consensus by Paxos
最简单的Paxos协议提供了一种确保所有节点关于某个值达成一致的方法。Multi-Paxos则基于Paxos，通过运行多个paxos实例来对多个值达成一致。

## Simple Paxos
![image](http://aosabook.org/en/500L/cluster-images/ballot.png)
上图描述了一个投票过程（ballot），一次投票过程起始于Proposer发送编号为N的Prepare请求给所有的Acceptors，并且等待多数Acceptors的响应。

Prepare请求用于获取小于N的编号最大的已经被接受的value（accepted value，如果有的话）。Acceptor则回复Promise响应给Proposer，其中包含它们所接受的value（accepted value），并承诺不会接受任何小于N的请求。如果Acceptor已经承诺了（Promise）某个更大编号的请求，该更大请求的编号也将包含在回复给Proposer的Promise响应中，表示当前Proposer已经被抢占，这种情况下，当前投票过程结束，但是Proposer可以以一个更大的编号重试它主导的投票过程。说的更直观一些，当Proposer发送编号为N的Prepare请求给所有Acceptors的时候，每个Acceptor的反应如下：返回Promise响应给Proposer，Promise中包括它已经承诺的最大编号和它已经接受的value（accepted value）。

当Proposer收到来自于大多数Acceptors的响应之后，它向所有的Acceptors发送包括编号和value的Accept消息。如果Proposer收到的promise中不包含任何已经被接受的value，它将发送自己的value，否则，它将发送包含最大编号的promise中的value。

除非Acceptor违背承诺，否则每一个Acceptor都会标记它接收到的Accept消息中的value为accepted，并且回复Accepted消息给Proposer。至此，一个投票过程结束，如果Proposer收到大多数Acceptors的Accepted消息中包含它发起的本次投票的编号，那么value就被确定了（value decided）。

考虑下面一种情况：
- Proposer A启动编号为1的投票，并执行了Prepare/Promise阶段；
- 在Proposer A发送Accept消息之前，Proposer B 启动编号为2的投票，并执行了Prepare/Promise阶段；
- Proposer A发送编号为1的Accept消息，此时所有acceptors都会拒绝Proposer A，因为它们已经承诺了Proposer B的编号为2的提议；
- 在Proposer B发送Accept消息之前，Proposer A重新启动编号为3的投票，发出Prepare消息；
- Proposer B发送Accept消息，所有的acceptors都会拒绝之，因为它们已经承诺了Proposer A的编号为3的提议；
- 重复上述过程。

像上面这样，多个Proposer同时启动投票的情况，在简单的Paxos协议中，很容易出现所有Proposer的投票过程都无法完成的情况。这就需要Multi-Paxos来解决了。

## Multi-Paxos

Multi-Paxos，事实上是一系列的Simple Paxos实例（每一个实例被称为slot），每一个实例都被顺序编号。每一次状态转换都有一个slot编号（slot number），每一个cluster节点都严格按照编号顺序进行状态转换。每一个消息中都会添加slot编号信息，所有协议状态都是基于slot编号。


## Paxos Made Pretty Hard

首先，由于每个cluster节点都会提出关于自身状态转换的提议，上面描述的多Proposer问题在一个繁忙的环境中将愈发明显，解决这个问题的办法是，选择一个“leader”专门负责向每个Paxos实例（slot）提交提议，所有其他节点都将操作发送给该这个leader来执行。

虽然Simple Paxos确保cluster不会产生冲突的决定（conflicting decisions），但是它不保证一定产生决定。比如，Prepare消息丢失了，Proposer一直等待着永远不会到达的Promise消息。重传（re-transmissiion）能够解决这个问题，但是需要精心设计：确保协议可以运行，但不要产生packet风暴。

其次，决议（Decision）传播也是一个问题，通常来说广播就可以，但是如果决议丢失，那么某个节点无法应用相应的状态转换。在工程实现中，需要采用某些机制确保决议能够到达正确传播。

再次，当一个新的节点启动的时候，它需要“追赶上”已经存在的cluster的状态，虽然这可以通过学习自第一个slot以来的所有决议来实现，但是一个成熟的cluster中可能包含多大数百万的slots。另外，还需要一个高效的方法来初始化新的cluster。

# Introducing Cluster
本节采用Python语言实现了Multi-Paxos，它被实现为库（library），称为cluster library。下面会有一个比较简单的讲解。

## Types and Constants

协议中用到15种不同的消息类型，每一种消息类型都采用Python namedtuple来定义。

```
    Accepted = namedtuple('Accepted', ['slot', 'ballot_num'])
    Accept = namedtuple('Accept', ['slot', 'ballot_num', 'proposal'])
    Decision = namedtuple('Decision', ['slot', 'proposal'])
    Invoked = namedtuple('Invoked', ['client_id', 'output'])
    Invoke = namedtuple('Invoke', ['caller', 'client_id', 'input_value'])
    Join = namedtuple('Join', [])
    Active = namedtuple('Active', [])
    Prepare = namedtuple('Prepare', ['ballot_num'])
    Promise = namedtuple('Promise', ['ballot_num', 'accepted_proposals'])
    Propose = namedtuple('Propose', ['slot', 'proposal'])
    Welcome = namedtuple('Welcome', ['state', 'slot', 'decisions'])
    Decided = namedtuple('Decided', ['slot'])
    Preempted = namedtuple('Preempted', ['slot', 'preempted_by'])
    Adopted = namedtuple('Adopted', ['ballot_num', 'accepted_proposals'])
    Accepting = namedtuple('Accepting', ['leader'])
```

协议中也会用到一些常量，多数用于定义消息超时时间。

```
    JOIN_RETRANSMIT = 0.7
    CATCHUP_INTERVAL = 0.6
    ACCEPT_RETRANSMIT = 1.0
    PREPARE_RETRANSMIT = 1.0
    INVOKE_RETRANSMIT = 0.5
    LEADER_TIMEOUT = 1.0
    NULL_BALLOT = Ballot(-1, -1)  # sorts before all real ballots
    NOOP_PROPOSAL = Proposal(None, None, None)  # no-op to fill otherwise empty slots
```

最后，协议中也会定义两个数据类型，用于描述协议。

```
    Proposal = namedtuple('Proposal', ['caller', 'client_id', 'input'])
    Ballot = namedtuple('Ballot', ['n', 'leader'])
```

## Component Model

为了实现更好的可测试性，在代码中根据角色（role）划分将协议分解为一系列类。每一个子类都继承自Role类。

```
class Role(object):

    def __init__(self, node):
        self.node = node
        self.node.register(self)
        self.running = True
        self.logger = node.logger.getChild(type(self).__name__)

    def set_timer(self, seconds, callback):
        return self.node.network.set_timer(self.node.address, seconds,
                                           lambda: self.running and callback())

    def stop(self):
        self.running = False
        self.node.unregister(self)
```

所有的角色（role）都通过Node类关联起来。Node类表示网络中的一个节点。随着协议的运转，不同的角色被添加到node中或者从node中移除。到达节点的消息被中继给各个不同的角色，并调用角色相关的do_{message type}方法，这些方法将接收到的消息中的属性作为键值对处理。为了方便，Node类实现了send方法（self.send = functools.partial(self.network.send, self)）。

```
class Node(object):
    unique_ids = itertools.count()

    def __init__(self, network, address):
        self.network = network
        self.address = address or 'N%d' % self.unique_ids.next()
        self.logger = SimTimeLogger(
            logging.getLogger(self.address), {'network': self.network})
        self.logger.info('starting')
        self.roles = []
        self.send = functools.partial(self.network.send, self)

    def register(self, roles):
        self.roles.append(roles)

    def unregister(self, roles):
        self.roles.remove(roles)

    def receive(self, sender, message):
        handler_name = 'do_%s' % type(message).__name__

        for comp in self.roles[:]:
            if not hasattr(comp, handler_name):
                continue
            comp.logger.debug("received %s from %s", message, sender)
            fn = getattr(comp, handler_name)
            fn(sender=sender, **message._asdict())
```

## Application interface

应用程序在每个cluster节点上创建并启动Member对象，提供应用相关的状态机和peer列表。如果某个cluster节点（cluster member）加入cluster，则member对象向node中添加bootstrap角色，如果某个cluster节点（cluster member）创建一个新的cluster，则member对象向node中添加seed角色。然后该member对象在一个单独的线程中运行协议。

应用程序通过invoke方法和cluster交互，该方法会启动一个proposal以进行状态转换。一旦决议产生并应用到状态机，invoke方法将返回状态机的执行结果。

```
class Member(object):

    def __init__(self, state_machine, network, peers, seed=None,
                 seed_cls=Seed, bootstrap_cls=Bootstrap):
        self.network = network
        self.node = network.new_node()
        if seed is not None:
            self.startup_role = seed_cls(self.node, initial_state=seed, peers=peers,
                                      execute_fn=state_machine)
        else:
            self.startup_role = bootstrap_cls(self.node,
                                      execute_fn=state_machine, peers=peers)
        self.requester = None

    def start(self):
        self.startup_role.start()
        self.thread = threading.Thread(target=self.network.run)
        self.thread.start()

    def invoke(self, input_value, request_cls=Requester):
        assert self.requester is None
        q = Queue.Queue()
        self.requester = request_cls(self.node, input_value, q.put)
        self.requester.start()
        output = q.get()
        self.requester = None
        return output
```

## Role classes
接下来看下协议中的各个role相关的类。

**Acceptor**

Acceptor类实现了acceptor角色，它必须存储表征其最新的承诺的编号，以及它所接受的关于每一个slot的提议（accepted proposal for each slot）。它根据协议规则响应Prepare和Accept消息。

对于acceptors来说，Multi-Paxos除了在每条消息中额外添加了slot编号以外，看起来跟简单Paxos（Simple Paxos）相似。

```
class Acceptor(Role):

    def __init__(self, node):
        super(Acceptor, self).__init__(node)
        self.ballot_num = NULL_BALLOT
        self.accepted_proposals = {}  # {slot: (ballot_num, proposal)}

    def do_Prepare(self, sender, ballot_num):
        if ballot_num > self.ballot_num:
            self.ballot_num = ballot_num
            # we've heard from a scout, so it might be the next leader
            self.node.send([self.node.address], Accepting(leader=sender))

        self.node.send([sender], Promise(
            ballot_num=self.ballot_num, 
            accepted_proposals=self.accepted_proposals
        ))

    def do_Accept(self, sender, ballot_num, slot, proposal):
        if ballot_num >= self.ballot_num:
            self.ballot_num = ballot_num
            acc = self.accepted_proposals
            if slot not in acc or acc[slot][0] < ballot_num:
                acc[slot] = (ballot_num, proposal)

        self.node.send([sender], Accepted(
            slot=slot, ballot_num=self.ballot_num))
```

**Replica**
Replica类是最复杂的角色类，它起到如下作用：
- 创建新的提议；
- 在决议产生（when proposals are decided）的时候调用状态机；
- 跟踪当前的leader；
- 添加新的节点到cluster中；

在接收到client端的Invoke消息的时候，Replica创建一个新的提议，选择一个它认为未被使用的slot，并发送Propose消息给当前的leader。如果分布式共识发现Replica选择的那个slot正在被另外一个proposal使用，那么Replica必须使用一个新的slot来重新提议。
![image](http://aosabook.org/en/500L/cluster-images/replica.png)

Decision表示某个Paxos实例（slot）已经达成共识，Replica存储该决议，运行状态机，执行状态转换直到它遇到一个未决的Paxos实例（slot）。Replica会区分已决定的paxos实例（decided slot）和已提交的paxos实例（commited slot），前者表示整个cluster达成了一致（the cluster has agreed），后者表示本地状态机已经处理（local state machine has processed）。

Replica需要知道哪个leader是活跃的，以便向它发送Propose消息。每个Replica通过如下3种信息源来跟踪活跃的leader。

当某个leader角色变为活跃状态时，它发送Adopted消息给其本地Replica。
![image](http://aosabook.org/en/500L/cluster-images/adopted.png)

当Acceptor发送Promise消息给某个新的leader的时候，它发送Accepting消息给其本地Replica。
![image](http://aosabook.org/en/500L/cluster-images/accepting.png)

活跃的leader向其本地Replica发送Active消息作为心跳信息，如果在LEADER_TIMEOUT超时之前，本地Replica没有收到Active消息，则它认为当前活跃的leader已经死亡，于是切换到下一个leader。在这种情况下，所有的Replica选择相同的leader非常重要，这里的实现方式是：对所有的成员进行排序，并选择紧接着先前活跃的leader的那个成员作为下一个leader。
![image](http://aosabook.org/en/500L/cluster-images/active.png)

最后，当一个节点加入网络，bootstrap角色将发送Join消息给所有的Replicas，每一个Replica都响应一个Welcom消息，其中包含它的最近的状态，以便新的节点尽快加入。
![image](http://aosabook.org/en/500L/cluster-images/bootstrap.png)

```
class Replica(Role):

    def __init__(self, node, execute_fn, state, slot, decisions, peers):
        super(Replica, self).__init__(node)
        self.execute_fn = execute_fn
        self.state = state
        self.slot = slot
        self.decisions = decisions
        self.peers = peers
        self.proposals = {}
        # next slot num for a proposal (may lead slot)
        self.next_slot = slot
        self.latest_leader = None
        self.latest_leader_timeout = None

    # making proposals

    def do_Invoke(self, sender, caller, client_id, input_value):
        proposal = Proposal(caller, client_id, input_value)
        slot = next((s for s, p in self.proposals.iteritems() if p == proposal), None)
        # propose, or re-propose if this proposal already has a slot
        self.propose(proposal, slot)

    def propose(self, proposal, slot=None):
        """Send (or resend, if slot is specified) a proposal to the leader"""
        if not slot:
            slot, self.next_slot = self.next_slot, self.next_slot + 1
        self.proposals[slot] = proposal
        # find a leader we think is working - either the latest we know of, or
        # ourselves (which may trigger a scout to make us the leader)
        leader = self.latest_leader or self.node.address
        self.logger.info(
            "proposing %s at slot %d to leader %s" % (proposal, slot, leader))
        self.node.send([leader], Propose(slot=slot, proposal=proposal))

    # handling decided proposals

    def do_Decision(self, sender, slot, proposal):
        assert not self.decisions.get(self.slot, None), \
                "next slot to commit is already decided"
        if slot in self.decisions:
            assert self.decisions[slot] == proposal, \
                "slot %d already decided with %r!" % (slot, self.decisions[slot])
            return
        self.decisions[slot] = proposal
        self.next_slot = max(self.next_slot, slot + 1)

        # re-propose our proposal in a new slot if it lost its slot and wasn't a no-op
        our_proposal = self.proposals.get(slot)
        if (our_proposal is not None and 
            our_proposal != proposal and our_proposal.caller):
            self.propose(our_proposal)

        # execute any pending, decided proposals
        while True:
            commit_proposal = self.decisions.get(self.slot)
            if not commit_proposal:
                break  # not decided yet
            commit_slot, self.slot = self.slot, self.slot + 1

            self.commit(commit_slot, commit_proposal)

    def commit(self, slot, proposal):
        """Actually commit a proposal that is decided and in sequence"""
        decided_proposals = [p for s, p in self.decisions.iteritems() if s < slot]
        if proposal in decided_proposals:
            self.logger.info(
                "not committing duplicate proposal %r, slot %d", proposal, slot)
            return  # duplicate

        self.logger.info("committing %r at slot %d" % (proposal, slot))
        if proposal.caller is not None:
            # perform a client operation
            self.state, output = self.execute_fn(self.state, proposal.input)
            self.node.send([proposal.caller], 
                Invoked(client_id=proposal.client_id, output=output))

    # tracking the leader

    def do_Adopted(self, sender, ballot_num, accepted_proposals):
        self.latest_leader = self.node.address
        self.leader_alive()

    def do_Accepting(self, sender, leader):
        self.latest_leader = leader
        self.leader_alive()

    def do_Active(self, sender):
        if sender != self.latest_leader:
            return
        self.leader_alive()

    def leader_alive(self):
        if self.latest_leader_timeout:
            self.latest_leader_timeout.cancel()

        def reset_leader():
            idx = self.peers.index(self.latest_leader)
            self.latest_leader = self.peers[(idx + 1) % len(self.peers)]
            self.logger.debug("leader timed out; tring the next one, %s", 
                self.latest_leader)
        self.latest_leader_timeout = self.set_timer(LEADER_TIMEOUT, reset_leader)

    # adding new cluster members

    def do_Join(self, sender):
        if sender in self.peers:
            self.node.send([sender], Welcome(
                state=self.state, slot=self.slot, decisions=self.decisions))
```


**Leader, Scout, and Commander**
为了遵从每个角色一个类的设计原则（class-per-role model），leader被委派给scout角色和commander角色，分别执行协议中Prepare/Promise部分和Accept/Accepted部分。

```
class Leader(Role):

    def __init__(self, node, peers, commander_cls=Commander, scout_cls=Scout):
        super(Leader, self).__init__(node)
        self.ballot_num = Ballot(0, node.address)
        self.active = False
        self.proposals = {}
        self.commander_cls = commander_cls
        self.scout_cls = scout_cls
        self.scouting = False
        self.peers = peers

    def start(self):
        # reminder others we're active before LEADER_TIMEOUT expires
        def active():
            if self.active:
                self.node.send(self.peers, Active())
            self.set_timer(LEADER_TIMEOUT / 2.0, active)
        active()

    def spawn_scout(self):
        assert not self.scouting
        self.scouting = True
        self.scout_cls(self.node, self.ballot_num, self.peers).start()

    def do_Adopted(self, sender, ballot_num, accepted_proposals):
        self.scouting = False
        self.proposals.update(accepted_proposals)
        # note that we don't re-spawn commanders here; if there are undecided
        # proposals, the replicas will re-propose
        self.logger.info("leader becoming active")
        self.active = True

    def spawn_commander(self, ballot_num, slot):
        proposal = self.proposals[slot]
        self.commander_cls(self.node, ballot_num, slot, proposal, self.peers).start()

    def do_Preempted(self, sender, slot, preempted_by):
        if not slot:  # from the scout
            self.scouting = False
        self.logger.info("leader preempted by %s", preempted_by.leader)
        self.active = False
        self.ballot_num = Ballot((preempted_by or self.ballot_num).n + 1, 
                                 self.ballot_num.leader)

    def do_Propose(self, sender, slot, proposal):
        if slot not in self.proposals:
            if self.active:
                self.proposals[slot] = proposal
                self.logger.info("spawning commander for slot %d" % (slot,))
                self.spawn_commander(self.ballot_num, slot)
            else:
                if not self.scouting:
                    self.logger.info("got PROPOSE when not active - scouting")
                    self.spawn_scout()
                else:
                    self.logger.info("got PROPOSE while scouting; ignored")
        else:
            self.logger.info("got PROPOSE for a slot already being proposed")
```
如果leader处于inactive状态，但是它接收到Propose消息，那么它会创建一个scout角色，以便切换到active状态。scout角色发送（或者重新发送，如果有必要的话）Prepare消息，并且收集Promise响应，直到它收到来自多数派的响应，或者它被抢占。然后它发送Adopted或者Preemted消息给leader。
![image](http://aosabook.org/en/500L/cluster-images/leaderscout.png)


```
class Scout(Role):

    def __init__(self, node, ballot_num, peers):
        super(Scout, self).__init__(node)
        self.ballot_num = ballot_num
        self.accepted_proposals = {}
        self.acceptors = set([])
        self.peers = peers
        self.quorum = len(peers) / 2 + 1
        self.retransmit_timer = None

    def start(self):
        self.logger.info("scout starting")
        self.send_prepare()

    def send_prepare(self):
        self.node.send(self.peers, Prepare(ballot_num=self.ballot_num))
        self.retransmit_timer = self.set_timer(PREPARE_RETRANSMIT, self.send_prepare)

    def update_accepted(self, accepted_proposals):
        acc = self.accepted_proposals
        for slot, (ballot_num, proposal) in accepted_proposals.iteritems():
            if slot not in acc or acc[slot][0] < ballot_num:
                acc[slot] = (ballot_num, proposal)

    def do_Promise(self, sender, ballot_num, accepted_proposals):
        if ballot_num == self.ballot_num:
            self.logger.info("got matching promise; need %d" % self.quorum)
            self.update_accepted(accepted_proposals)
            self.acceptors.add(sender)
            if len(self.acceptors) >= self.quorum:
                # strip the ballot numbers from self.accepted_proposals, now that it
                # represents a majority
                accepted_proposals = \ 
                    dict((s, p) for s, (b, p) in self.accepted_proposals.iteritems())
                # We're adopted; note that this does *not* mean that no other
                # leader is active.  # Any such conflicts will be handled by the
                # commanders.
                self.node.send([self.node.address],
                    Adopted(ballot_num=ballot_num, 
                            accepted_proposals=accepted_proposals))
                self.stop()
        else:
            # this acceptor has promised another leader a higher ballot number,
            # so we've lost
            self.node.send([self.node.address], 
                Preempted(slot=None, preempted_by=ballot_num))
            self.stop()
```

对于每一个活跃的提议（active proposal，已经接收到多数派Acceptors的promise的提议？）所在的Paxos实例（slot），leader会为之创建一个commander角色。commander发送或者重新发送Accept消息给Acceptors，并且等待多数派的Accepted消息，或者等待它被抢占的消息。当决议产生（a proposal is accepted）之后，commander将该决议（Decision）消息广播给所有的节点。commander发送给leader的响应是Decided或者Preempted消息。
![image](http://aosabook.org/en/500L/cluster-images/leadercommander.png)


```
class Commander(Role):

    def __init__(self, node, ballot_num, slot, proposal, peers):
        super(Commander, self).__init__(node)
        self.ballot_num = ballot_num
        self.slot = slot
        self.proposal = proposal
        self.acceptors = set([])
        self.peers = peers
        self.quorum = len(peers) / 2 + 1

    def start(self):
        self.node.send(set(self.peers) - self.acceptors, Accept(
            slot=self.slot, ballot_num=self.ballot_num, proposal=self.proposal))
        self.set_timer(ACCEPT_RETRANSMIT, self.start)

    def finished(self, ballot_num, preempted):
        if preempted:
            self.node.send([self.node.address], 
                           Preempted(slot=self.slot, preempted_by=ballot_num))
        else:
            self.node.send([self.node.address], 
                           Decided(slot=self.slot))
        self.stop()

    def do_Accepted(self, sender, slot, ballot_num):
        if slot != self.slot:
            return
        if ballot_num == self.ballot_num:
            self.acceptors.add(sender)
            if len(self.acceptors) < self.quorum:
                return
            self.node.send(self.peers, Decision(
                           slot=self.slot, proposal=self.proposal))
            self.finished(ballot_num, False)
        else:
            self.finished(ballot_num, True)
```

**Bootstrap**
当一个节点加入cluster的时候，它必须确切知道当前cluster的状态，bootstrap角色起到这个作用，它轮流向所有节点发送Join消息，直到它接收到Welcome消息。
![image](http://aosabook.org/en/500L/cluster-images/bootstrap.png)


```
class Bootstrap(Role):

    def __init__(self, node, peers, execute_fn,
                 replica_cls=Replica, acceptor_cls=Acceptor, leader_cls=Leader,
                 commander_cls=Commander, scout_cls=Scout):
        super(Bootstrap, self).__init__(node)
        self.execute_fn = execute_fn
        self.peers = peers
        self.peers_cycle = itertools.cycle(peers)
        self.replica_cls = replica_cls
        self.acceptor_cls = acceptor_cls
        self.leader_cls = leader_cls
        self.commander_cls = commander_cls
        self.scout_cls = scout_cls

    def start(self):
        self.join()

    def join(self):
        self.node.send([next(self.peers_cycle)], Join())
        self.set_timer(JOIN_RETRANSMIT, self.join)

    def do_Welcome(self, sender, state, slot, decisions):
        self.acceptor_cls(self.node)
        self.replica_cls(self.node, execute_fn=self.execute_fn, peers=self.peers,
                         state=state, slot=slot, decisions=decisions)
        self.leader_cls(self.node, peers=self.peers, commander_cls=self.commander_cls,
                        scout_cls=self.scout_cls).start()
        self.stop()
```

**seed**
创建一个新的cluster完全由用户来决定，有且仅有一个节点运行seed角色，其它节点则运行bootstrap角色。seed角色等待多数派节点的Join消息，然后发送Welcome消息给这些节点，其中附带初始状态和一个空的决议集合。然后seed角色所在的节点启动bootstrap角色来加入到新创建的cluster，然后停止seed角色。

```
class Seed(Role):

    def __init__(self, node, initial_state, execute_fn, peers, 
                 bootstrap_cls=Bootstrap):
        super(Seed, self).__init__(node)
        self.initial_state = initial_state
        self.execute_fn = execute_fn
        self.peers = peers
        self.bootstrap_cls = bootstrap_cls
        self.seen_peers = set([])
        self.exit_timer = None

    def do_Join(self, sender):
        self.seen_peers.add(sender)
        if len(self.seen_peers) <= len(self.peers) / 2:
            return

        # cluster is ready - welcome everyone
        self.node.send(list(self.seen_peers), Welcome(
            state=self.initial_state, slot=1, decisions={}))

        # stick around for long enough that we don't hear any new JOINs from
        # the newly formed cluster
        if self.exit_timer:
            self.exit_timer.cancel()
        self.exit_timer = self.set_timer(JOIN_RETRANSMIT * 2, self.finish)

    def finish(self):
        # bootstrap this node into the cluster we just seeded
        bs = self.bootstrap_cls(self.node, 
                                peers=self.peers, execute_fn=self.execute_fn)
        bs.start()
        self.stop()
```

**Requester**
requester角色负责产生一个发往分布式状态机的请求，它发送Invoke消息给本地replica直到接收到Invoked响应。
![image](http://aosabook.org/en/500L/cluster-images/replica.png)

```
class Requester(Role):

    client_ids = itertools.count(start=100000)

    def __init__(self, node, n, callback):
        super(Requester, self).__init__(node)
        self.client_id = self.client_ids.next()
        self.n = n
        self.output = None
        self.callback = callback

    def start(self):
        self.node.send([self.node.address], 
                       Invoke(caller=self.node.address, 
                              client_id=self.client_id, input_value=self.n))
        self.invoke_timer = self.set_timer(INVOKE_RETRANSMIT, self.start)

    def do_Invoked(self, sender, client_id, output):
        if client_id != self.client_id:
            return
        self.logger.debug("received output %r" % (output,))
        self.invoke_timer.cancel()
        self.callback(output)
        self.stop()
```

## summary
cluster中各个角色总结如下：
- Acceptor -- 承诺并接受提议（make promise and accept proposals）
- Replica -- 管理分布式状态机：提交提议（submitting proposals），提交决议（committing proposals），响应requesters（responding to requesters）
- Leader -- 引导Multi-Paxos过程（lead rountd of the Multi-Paxos algorithm）
- Scout -- 代理leader执行Multi-Paxos中的Prepare/Promise阶段
- Commander -- 代理leader执行Multi-Paxos中的Accept/Accepted阶段
- Bootstrap -- 引导新节点加入一个已经存在的cluster
- Seed -- 创建一个新的cluster
- Requester -- 请求一个分布式状态机操作

接下来就只需要添加各节点间通信所需的网络就可以让cluster运行起来了。

# Network
任何网络协议都需要提供如下能力：发送消息，接收消息，在未来某个时刻调用某个函数。

Network类提供了一个简单的支持这些能力的模拟网络，并且支持包丢失和消息延时传播。

定时器（Timer）采用了Python的heapq模块，提供高效选择下一个定时器的能力。设定一个定时器只需要将一个Timer对象push到heap中即可。由于从heap中移除一个元素不太高效，所以取消的定时器仍然保存在heap中，但是被标记为cancelled。

消息传输采用定时器来实现设定一个较晚时间点的消息传输。

运行模拟器就是不断地从heap中poo出来定时器，并执行它们（如果它们没有被cancel且目标节点仍然是活着的话）。

```
class Timer(object):

    def __init__(self, expires, address, callback):
        self.expires = expires
        self.address = address
        self.callback = callback
        self.cancelled = False

    def __cmp__(self, other):
        return cmp(self.expires, other.expires)

    def cancel(self):
        self.cancelled = True


class Network(object):
    PROP_DELAY = 0.03
    PROP_JITTER = 0.02
    DROP_PROB = 0.05

    def __init__(self, seed):
        self.nodes = {}
        self.rnd = random.Random(seed)
        self.timers = []
        self.now = 1000.0

    def new_node(self, address=None):
        node = Node(self, address=address)
        self.nodes[node.address] = node
        return node

    def run(self):
        while self.timers:
            next_timer = self.timers[0]
            if next_timer.expires > self.now:
                self.now = next_timer.expires
            heapq.heappop(self.timers)
            if next_timer.cancelled:
                continue
            if not next_timer.address or next_timer.address in self.nodes:
                next_timer.callback()

    def stop(self):
        self.timers = []

    def set_timer(self, address, seconds, callback):
        timer = Timer(self.now + seconds, address, callback)
        heapq.heappush(self.timers, timer)
        return timer

    def send(self, sender, destinations, message):
        sender.logger.debug("sending %s to %s", message, destinations)
        # avoid aliasing by making a closure containing distinct deep copy of
        # message for each dest
        def sendto(dest, message):
            if dest == sender.address:
                # reliably deliver local messages with no delay
                self.set_timer(sender.address, 0,  
                               lambda: sender.receive(sender.address, message))
            elif self.rnd.uniform(0, 1.0) > self.DROP_PROB:
                delay = self.PROP_DELAY + self.rnd.uniform(-self.PROP_JITTER, 
                                                           self.PROP_JITTER)
                self.set_timer(dest, delay, 
                               functools.partial(self.nodes[dest].receive, 
                                                 sender.address, message))
        for dest in (d for d in destinations if d in self.nodes):
            sendto(dest, copy.deepcopy(message))
```

# Source Code
run.py

```
from cluster import *
import sys

def key_value_state_machine(state, input_value):
    if input_value[0] == 'get':
        return state, state.get(input_value[1], None)
    elif input_value[0] == 'set':
        state[input_value[1]] = input_value[2]
        return state, input_value[2]

sequences_running = 0
def do_sequence(network, node, key):
    global sequences_running
    sequences_running += 1
    reqs = [
        (('get', key), None),
        (('set', key, 10), 10),
        (('get', key), 10),
        (('set', key, 20), 20),
        (('set', key, 30), 30),
        (('get', key), 30),
    ]
    def request():
        # if current sequence is done, then decrease global @sequences_running,
        if not reqs:
            global sequences_running
            sequences_running -= 1
            if not sequences_running:
                network.stop()
            return
        input, exp_output = reqs.pop(0)
        def req_done(output):
            assert output == exp_output, "%r != %r" % (output, exp_output)
            # trigger the next request in current sequence
            request()
            
        # here should be Requester instead ???
        # requester send Invoke message to its local replica, and req_done acts as
        # callback which is invoked in Requester.do_Invoked, req_done will trigger
        # next request 
        Request(node, input, req_done).start()

    # add a new timer expired in 1.0 second later, and the expiration callback
    # is method request()
    network.set_timer(None, 1.0, request)


def main():
    logging.basicConfig(
        format="%(name)s - %(message)s", level=logging.DEBUG)

    # sys.argv[1] acts as seed to random generator for Network, random generator
    # is used for simulating randomly message lost
    network = Network(int(sys.argv[1]))

    peers = ['N%d' % i for i in range(7)]
    for p in peers:
        # newly create node and add into network
        node = network.new_node(address=p)
        
        # node "N0" acts as seed role when creating a new cluster, and other nodes
        # act as bootstrap role, the seed role will wait until it receives Join
        # message from majority of its peers, and then send Welcome message to these
        # peers, and then the seed role stops itself and starts  a bootstrap role
        # to join the newly-seeded cluster
        if p == 'N0':
            Seed(node, initial_state={}, peers=peers, execute_fn=key_value_state_machine)
        else:
            Bootstrap(node, execute_fn=key_value_state_machine, peers=peers).start()

    for key in 'abcdefg':
        do_sequence(network, node, key)
    network.run()

if __name__ == "__main__":
    main()
```

cluster.py

```
from collections import namedtuple
import functools
import heapq
import itertools
import logging
import Queue
import random
import threading

# data types
Proposal = namedtuple('Proposal', ['caller', 'client_id', 'input'])
Ballot = namedtuple('Ballot', ['n', 'leader'])

# message types
Accepted = namedtuple('Accepted', ['slot', 'ballot_num'])
Accept = namedtuple('Accept', ['slot', 'ballot_num', 'proposal'])
Decision = namedtuple('Decision', ['slot', 'proposal'])
Invoked = namedtuple('Invoked', ['client_id', 'output'])
Invoke = namedtuple('Invoke', ['caller', 'client_id', 'input_value'])
Join = namedtuple('Join', [])
Active = namedtuple('Active', [])
Prepare = namedtuple('Prepare', ['ballot_num'])
Promise = namedtuple('Promise', ['ballot_num', 'accepted_proposals'])
Propose = namedtuple('Propose', ['slot', 'proposal'])
Welcome = namedtuple('Welcome', ['state', 'slot', 'decisions'])
Decided = namedtuple('Decided', ['slot'])
Preempted = namedtuple('Preempted', ['slot', 'preempted_by'])
Adopted = namedtuple('Adopted', ['ballot_num', 'accepted_proposals'])
Accepting = namedtuple('Accepting', ['leader'])

# constants - these times should really be in terms of average round-trip time
JOIN_RETRANSMIT = 0.7
CATCHUP_INTERVAL = 0.6
ACCEPT_RETRANSMIT = 1.0
PREPARE_RETRANSMIT = 1.0
INVOKE_RETRANSMIT = 0.5
LEADER_TIMEOUT = 1.0
NULL_BALLOT = Ballot(-1, -1)  # sorts before all real ballots
NOOP_PROPOSAL = Proposal(None, None, None)  # no-op to fill otherwise empty slots

class Node(object):
    # itertools.count(start=0, step=1) -> 0, 1, 2, 3, 4 ...
    unique_ids = itertools.count()

    def __init__(self, network, address):
        self.network = network
        self.address = address or 'N%d' % self.unique_ids.next()
        self.logger = SimTimeLogger(logging.getLogger(self.address), {'network': self.network})
        self.logger.info('starting')
        self.roles = []
        # here use "self" to act as positional argument "sender" in method self.network.send,
        self.send = functools.partial(self.network.send, self)

    def register(self, roles):
        self.roles.append(roles)

    def unregister(self, roles):
        self.roles.remove(roles)

    def receive(self, sender, message):
        # message handler name will be do_{message_type}
        handler_name = 'do_%s' % type(message).__name__

        # traverse all roles in node, and call role specific message handler if the role do have
        # corresponding message handler
        for comp in self.roles[:]:
            # role has no corresponding message handler
            if not hasattr(comp, handler_name):
                continue
            comp.logger.debug("received %s from %s", message, sender)
            fn = getattr(comp, handler_name)
            # invoke role specific message handler
            fn(sender=sender, **message._asdict())

class Timer(object):

    def __init__(self, expires, address, callback):
        self.expires = expires
        self.address = address
        self.callback = callback
        self.cancelled = False

    # for timer sorting
    def __cmp__(self, other):
        return cmp(self.expires, other.expires)

    def cancel(self):
        self.cancelled = True

class Network(object):
    # propagation delay: 传播时延
    PROP_DELAY = 0.03
    PROP_JITTER = 0.02
    # drop probability
    DROP_PROB = 0.05

    def __init__(self, seed):
        self.nodes = {}
        self.rnd = random.Random(seed)
        self.timers = []
        self.now = 1000.0

    def new_node(self, address=None):
        node = Node(self, address=address)
        self.nodes[node.address] = node
        return node

    def run(self):
        while self.timers:
            next_timer = self.timers[0]
            if next_timer.expires > self.now:
                self.now = next_timer.expires
            heapq.heappop(self.timers)
            if next_timer.cancelled:
                continue
            if not next_timer.address or next_timer.address in self.nodes:
                next_timer.callback()

    def stop(self):
        self.timers = []

    # add a new timer
    def set_timer(self, address, seconds, callback):
        timer = Timer(self.now + seconds, address, callback)
        heapq.heappush(self.timers, timer)
        return timer

    def send(self, sender, destinations, message):
        sender.logger.debug("sending %s to %s", message, destinations)
        for dest in (d for d in destinations if d in self.nodes):
            if dest == sender.address:
                # reliably deliver local messages with no delay
                self.set_timer(sender.address, 0, lambda: sender.receive(sender.address, message))
            elif self.rnd.uniform(0, 1.0) > self.DROP_PROB:
                # not drop the message, then simulate receiving the message in random time delay
                delay = self.PROP_DELAY + self.rnd.uniform(-self.PROP_JITTER, self.PROP_JITTER)
                # functools.partial(self.nodes[dest].receive, sender.address, message) equals to
                # self.nodes[dest].receive(sender.address, message)
                self.set_timer(dest, delay, functools.partial(self.nodes[dest].receive,
                                                              sender.address, message))

class SimTimeLogger(logging.LoggerAdapter):

    def process(self, msg, kwargs):
        return "T=%.3f %s" % (self.extra['network'].now, msg), kwargs

    def getChild(self, name):
        return self.__class__(self.logger.getChild(name),
                              {'network': self.extra['network']})

class Role(object):

    def __init__(self, node):
        self.node = node
        self.node.register(self)
        self.running = True
        self.logger = node.logger.getChild(type(self).__name__)

    # wrapper for self.node.network.set_timer, called only if self.running is True
    def set_timer(self, seconds, callback):
        return self.node.network.set_timer(self.node.address, seconds,
                                           lambda: self.running and callback())

    def stop(self):
        # no timer callback will be invoked any more from now on
        self.running = False
        # unregister current role from node
        self.node.unregister(self)

class Acceptor(Role):

    def __init__(self, node):
        super(Acceptor, self).__init__(node)
        self.ballot_num = NULL_BALLOT
        self.accepted_proposals = {}  # {slot: (ballot_num, proposal)}

    def do_Prepare(self, sender, ballot_num):
        if ballot_num > self.ballot_num:
            self.ballot_num = ballot_num
            # we've heard from a scout, so it might be the next leader
            # tell myself that the next leader should be current sender
            self.node.send([self.node.address], Accepting(leader=sender))

        self.node.send([sender], Promise(ballot_num=self.ballot_num, accepted_proposals=self.accepted_proposals))

    def do_Accept(self, sender, ballot_num, slot, proposal):
        if ballot_num >= self.ballot_num:
            self.ballot_num = ballot_num
            acc = self.accepted_proposals
            if slot not in acc or acc[slot][0] < ballot_num:
                acc[slot] = (ballot_num, proposal)

        self.node.send([sender], Accepted(
            slot=slot, ballot_num=self.ballot_num))

class Replica(Role):

    def __init__(self, node, execute_fn, state, slot, decisions, peers):
        super(Replica, self).__init__(node)
        self.execute_fn = execute_fn
        self.state = state
        self.slot = slot
        self.decisions = decisions.copy()
        self.peers = peers
        self.proposals = {}
        # next slot num for a proposal (may lead slot)
        self.next_slot = slot
        self.latest_leader = None
        self.latest_leader_timeout = None

    # making proposals

    def do_Invoke(self, sender, caller, client_id, input_value):
        proposal = Proposal(caller, client_id, input_value)
        # find the already existed slot satisfying (p == proposal), otherwise return None
        slot = next((s for s, p in self.proposals.iteritems() if p == proposal), None)
        # propose, or re-propose if this proposal already has a slot
        self.propose(proposal, slot)

    def propose(self, proposal, slot=None):
        """Send (or resend, if slot is specified) a proposal to the leader"""
        if not slot:
            slot, self.next_slot = self.next_slot, self.next_slot + 1
        self.proposals[slot] = proposal
        # find a leader we think is working - either the latest we know of, or
        # ourselves (which may trigger a scout to make us the leader)
        leader = self.latest_leader or self.node.address
        self.logger.info("proposing %s at slot %d to leader %s" % (proposal, slot, leader))
        self.node.send([leader], Propose(slot=slot, proposal=proposal))

    # handling decided proposals

    def do_Decision(self, sender, slot, proposal):
        assert not self.decisions.get(self.slot, None), \
                "next slot to commit is already decided"
        if slot in self.decisions:
            assert self.decisions[slot] == proposal, \
                "slot %d already decided with %r!" % (slot, self.decisions[slot])
            return
        self.decisions[slot] = proposal
        self.next_slot = max(self.next_slot, slot + 1)

        # re-propose our proposal in a new slot if it lost its slot and wasn't a no-op
        our_proposal = self.proposals.get(slot)
        if our_proposal is not None and our_proposal != proposal and our_proposal.caller:
            self.propose(our_proposal)

        # execute any pending, decided proposals
        while True:
            commit_proposal = self.decisions.get(self.slot)
            if not commit_proposal:
                break  # not decided yet
            commit_slot, self.slot = self.slot, self.slot + 1

            self.commit(commit_slot, commit_proposal)

    def commit(self, slot, proposal):
        """Actually commit a proposal that is decided and in sequence"""
        decided_proposals = [p for s, p in self.decisions.iteritems() if s < slot]
        if proposal in decided_proposals:
            self.logger.info("not committing duplicate proposal %r at slot %d", proposal, slot)
            return  # duplicate

        self.logger.info("committing %r at slot %d" % (proposal, slot))
        if proposal.caller is not None:
            # perform a client operation
            self.state, output = self.execute_fn(self.state, proposal.input)
            self.node.send([proposal.caller], Invoked(client_id=proposal.client_id, output=output))

    # tracking the leader

    def do_Adopted(self, sender, ballot_num, accepted_proposals):
        self.latest_leader = self.node.address
        self.leader_alive()

    def do_Accepting(self, sender, leader):
        self.latest_leader = leader
        self.leader_alive()

    def do_Active(self, sender):
        if sender != self.latest_leader:
            return
        self.leader_alive()

    def leader_alive(self):
        if self.latest_leader_timeout:
            self.latest_leader_timeout.cancel()

        def reset_leader():
            idx = self.peers.index(self.latest_leader)
            # try to elect the peer next to latest leader as new leader
            self.latest_leader = self.peers[(idx + 1) % len(self.peers)]
            self.logger.debug("leader timed out; tring the next one, %s", self.latest_leader)
        # reset leader when LEADER_TIMEOUT expires
        self.latest_leader_timeout = self.set_timer(LEADER_TIMEOUT, reset_leader)

    # adding new cluster members

    def do_Join(self, sender):
        if sender in self.peers:
            self.node.send([sender], Welcome(
                state=self.state, slot=self.slot, decisions=self.decisions))

class Commander(Role):

    def __init__(self, node, ballot_num, slot, proposal, peers):
        super(Commander, self).__init__(node)
        self.ballot_num = ballot_num
        self.slot = slot
        self.proposal = proposal
        self.acceptors = set([])
        self.peers = peers
        self.quorum = len(peers) / 2 + 1

    def start(self):
        # send Accept message to peers excluding those already accepted ones
        self.node.send(set(self.peers) - self.acceptors, Accept(
                            slot=self.slot, ballot_num=self.ballot_num, proposal=self.proposal))
        self.set_timer(ACCEPT_RETRANSMIT, self.start)

    def finished(self, ballot_num, preempted):
        if preempted:
            self.node.send([self.node.address], Preempted(slot=self.slot, preempted_by=ballot_num))
        else:
            self.node.send([self.node.address], Decided(slot=self.slot))
        self.stop()

    def do_Accepted(self, sender, slot, ballot_num):
        if slot != self.slot:
            return
        if ballot_num == self.ballot_num:
            self.acceptors.add(sender)
            if len(self.acceptors) < self.quorum:
                return
            self.node.send(self.peers, Decision(slot=self.slot, proposal=self.proposal))
            self.finished(ballot_num, False)
        else:
            self.finished(ballot_num, True)

class Scout(Role):

    def __init__(self, node, ballot_num, peers):
        super(Scout, self).__init__(node)
        self.ballot_num = ballot_num
        self.accepted_proposals = {}
        self.acceptors = set([])
        self.peers = peers
        self.quorum = len(peers) / 2 + 1
        self.retransmit_timer = None

    def start(self):
        self.logger.info("scout starting")
        self.send_prepare()

    def send_prepare(self):
        # send Prepare message to all its peers(Acceptors)
        self.node.send(self.peers, Prepare(ballot_num=self.ballot_num))
        self.retransmit_timer = self.set_timer(PREPARE_RETRANSMIT, self.send_prepare)

    def update_accepted(self, accepted_proposals):
        acc = self.accepted_proposals
        for slot, (ballot_num, proposal) in accepted_proposals.iteritems():
            if slot not in acc or acc[slot][0] < ballot_num:
                acc[slot] = (ballot_num, proposal)

    def do_Promise(self, sender, ballot_num, accepted_proposals):
        if ballot_num == self.ballot_num:
            self.logger.info("got matching promise; need %d" % self.quorum)
            self.update_accepted(accepted_proposals)
            self.acceptors.add(sender)
            if len(self.acceptors) >= self.quorum:
                # strip the ballot numbers from self.accepted_proposals, now that it
                # represents a majority
                accepted_proposals = dict((s, p) for s, (b, p) in self.accepted_proposals.iteritems())
                # We're adopted; note that this does *not* mean that no other leader is active.
                # Any such conflicts will be handled by the commanders.
                self.node.send([self.node.address],
                               Adopted(ballot_num=ballot_num, accepted_proposals=accepted_proposals))
                self.stop()
        else:
            # this acceptor has promised another leader a higher ballot number, so we've lost
            self.node.send([self.node.address], Preempted(slot=None, preempted_by=ballot_num))
            self.stop()

class Leader(Role):

    def __init__(self, node, peers, commander_cls=Commander, scout_cls=Scout):
        super(Leader, self).__init__(node)
        self.ballot_num = Ballot(0, node.address)
        self.active = False
        self.proposals = {}
        self.commander_cls = commander_cls
        self.scout_cls = scout_cls
        self.scouting = False
        self.peers = peers

    def start(self):
        # reminder others we're active before LEADER_TIMEOUT expires
        def active():
            if self.active:
                # send Active message to all replicas, handler will be Replica.do_Active
                self.node.send(self.peers, Active())
            self.set_timer(LEADER_TIMEOUT / 2.0, active)
        active()

    def spawn_scout(self):
        assert not self.scouting
        self.scouting = True
        self.scout_cls(self.node, self.ballot_num, self.peers).start()

    def do_Adopted(self, sender, ballot_num, accepted_proposals):
        self.scouting = False
        self.proposals.update(accepted_proposals)
        # note that we don't re-spawn commanders here; if there are undecided
        # proposals, the replicas will re-propose
        self.logger.info("leader becoming active")
        self.active = True

    def spawn_commander(self, ballot_num, slot):
        proposal = self.proposals[slot]
        self.commander_cls(self.node, ballot_num, slot, proposal, self.peers).start()

    def do_Preempted(self, sender, slot, preempted_by):
        if not slot:  # from the scout
            self.scouting = False
        self.logger.info("leader preempted by %s", preempted_by.leader)
        self.active = False
        self.ballot_num = Ballot((preempted_by or self.ballot_num).n + 1, self.ballot_num.leader)

    def do_Propose(self, sender, slot, proposal):
        if slot not in self.proposals:
            if self.active:
                self.proposals[slot] = proposal
                self.logger.info("spawning commander for slot %d" % (slot,))
                self.spawn_commander(self.ballot_num, slot)
            else:
                # scouting is not in-progress, then spawn it
                if not self.scouting:
                    self.logger.info("got PROPOSE when not active - scouting")
                    self.spawn_scout()
                else:
                    self.logger.info("got PROPOSE while scouting; ignored")
        else:
            self.logger.info("got PROPOSE for a slot already being proposed")

class Bootstrap(Role):

    def __init__(self, node, peers, execute_fn,
                 replica_cls=Replica, acceptor_cls=Acceptor, leader_cls=Leader,
                 commander_cls=Commander, scout_cls=Scout):
        super(Bootstrap, self).__init__(node)
        self.execute_fn = execute_fn
        self.peers = peers
        # itertools module contains functions creating iterators for efficient loop
        # itertools.cycle('ABCD') --> A B C D A B C D A B C D ...
        self.peers_cycle = itertools.cycle(peers)
        self.replica_cls = replica_cls
        self.acceptor_cls = acceptor_cls
        self.leader_cls = leader_cls
        self.commander_cls = commander_cls
        self.scout_cls = scout_cls

    def start(self):
        self.join()

    def join(self):
        # invoke Network's send eventually
        # send Join message to next one in @peers
        self.node.send([next(self.peers_cycle)], Join())
        self.set_timer(JOIN_RETRANSMIT, self.join)

    def do_Welcome(self, sender, state, slot, decisions):
        self.acceptor_cls(self.node)
        self.replica_cls(self.node, execute_fn=self.execute_fn, peers=self.peers,
                         state=state, slot=slot, decisions=decisions)
        # here, Leader.start() will not send Active message to all replicas, as currently
        # it is not in atcive state(Leader.active = False)
        self.leader_cls(self.node, peers=self.peers, commander_cls=self.commander_cls,
                        scout_cls=self.scout_cls).start()
        self.stop()

class Seed(Role):

    def __init__(self, node, initial_state, execute_fn, peers, bootstrap_cls=Bootstrap):
        super(Seed, self).__init__(node)
        self.initial_state = initial_state
        self.execute_fn = execute_fn
        self.peers = peers
        self.bootstrap_cls = bootstrap_cls
        self.seen_peers = set([])
        self.exit_timer = None

    # callback against Join message
    def do_Join(self, sender):
        # got @sender's Join message
        self.seen_peers.add(sender)
        # less than majority
        if len(self.seen_peers) <= len(self.peers) / 2:
            return

        # cluster is ready - welcome everyone
        self.node.send(list(self.seen_peers), Welcome(
            state=self.initial_state, slot=1, decisions={}))

        # stick around for long enough that we don't hear any new JOINs from
        # the newly formed cluster
        if self.exit_timer:
            self.exit_timer.cancel()
        self.exit_timer = self.set_timer(JOIN_RETRANSMIT * 2, self.finish)

    def finish(self):
        # bootstrap this node into the cluster we just seeded
        bs = self.bootstrap_cls(self.node, peers=self.peers, execute_fn=self.execute_fn)
        # @self.node acts as both seed role and bootstrap role currently
        bs.start()
        # self.stop() will set self.running to false, which will disable the timer's
        # callback, so here self.finish will not be called again from now on
        self.stop()

class Requester(Role):

    # itertools.count(10) --> 10 11 12 13 14 ...
    client_ids = itertools.count(start=100000)

    def __init__(self, node, n, callback):
        super(Requester, self).__init__(node)
        self.client_id = self.client_ids.next()
        self.n = n
        self.output = None
        self.callback = callback

    def start(self):
        # send Invoke message to its local replica
        self.node.send([self.node.address], Invoke(caller=self.node.address,
                                                   client_id=self.client_id, input_value=self.n))
        # resend Invoke message if INVOKE_RETRANSMIT expires and not got Invoked message                                           
        self.invoke_timer = self.set_timer(INVOKE_RETRANSMIT, self.start)

    def do_Invoked(self, sender, client_id, output):
        if client_id != self.client_id:
            return
        self.logger.debug("received output %r" % (output,))
        # cancle the timer regarding resending Invoke message
        self.invoke_timer.cancel()
        self.callback(output)
        self.stop()

class Member(object):

    def __init__(self, state_machine, network, peers, seed=None,
                 seed_cls=Seed, bootstrap_cls=Bootstrap):
        self.network = network
        self.node = network.new_node()
        if seed is not None:
            self.startup_role = seed_cls(self.node, initial_state=seed, peers=peers,
                                      execute_fn=state_machine)
        else:
            self.startup_role = bootstrap_cls(self.node, execute_fn=state_machine, peers=peers)
        self.requester = None

    def start(self):
        self.startup_role.start()
        self.thread = threading.Thread(target=self.network.run)
        self.thread.start()

    def invoke(self, input_value, request_cls=Requester):
        assert self.requester is None
        q = Queue.Queue()
        self.requester = request_cls(self.node, input_value, q.put)
        self.requester.start()
        output = q.get()
        self.requester = None
        return output
```





