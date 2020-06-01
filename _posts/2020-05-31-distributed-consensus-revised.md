---
layout: post
title:  "重温分布式共识算法 Paxos"
date:   2020-06-01 10:02:33 +0800
categories: distributed-consensus
---

作为一个分布式系统新手，要想入门这一领域，其中必须跨过的一座大山无疑是分布式的共识问题。分布式的共识问题，是指如何在多个进程中对某些值达成一致的问题。注意这些进程是不可靠的，即有可能发生各种各样的 failure 而导致无法交互或丢失数据等。因此也有很多文章将分布式共识的问题描述为：如何在不可靠的组件上实现整个系统的可靠性。

分布式共识算法的先驱是 Leslie Lamport 提出的 Paxos 算法。直到今天，绝大多数的分布式存储系统的同步问题都是使用 Paxos 延伸出来的算法解决。但由于它的初始版本以既不通俗也不易懂，降低 Paxos 的理解难度的工作从它诞生起就已经开始了。尽管 Google 出来的一部分科普向的博客 [Basic-Paxos](http://www.beyondthelines.net/algorithm/basic-paxos/) 或文章 [Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) 看起来十分简单，但仍然有一些地方不够清晰，以至于无法理解为什么是正确的。导致这个问题的原因有很多，例如用词不统一、定义不规范、夹杂对理解没有帮助的优化等等。

[Distributed Consensus Revised](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-935.pdf) 是一篇出自 University of Cambridge 的计算机 PHD 论文，作者是 Heidi Howard。该论文对分布式共识算法 Paxos 做了一部分修正，包括放宽 quorum 的限制、探索 Proposer 第一阶段的值选择、探索 epoch 值的分配方法等等。此外，这篇论文前两章定义的 Classic Paxos 以及整理的常用的算法优化方法，即使是小白如我也能略懂一二，十分值得涉猎。

虽然 Revised 是修订的含义，但在我这里的意义就是“温习”、“复习”，于是结合个人理解与论文内容，写下这篇博客用于记录（希望能够打通自己的任督二脉）。论文中含有大量的正确性证明，但这里不会贴完整的过程，只会非正式地稍作解释。

## 限制和假设

在讲 Paxos 之前，首先要知道它的局限和所做的假设是什么。任何的机制都是在某些前提下才能适用：以故障恢复为例，即使是 RAID 9999 也抗不住地球爆炸。Paxos 适用的前提包括以下几个方面：

- 无拜占庭错误：系统的参与者中不会出现叛徒。
- 无 Weakened Semantics：系统不会因读延迟、时钟漂移而产生某个算法步骤预期以外的行为。
- 实践上的 Progressive：所有的信息最终都能够被传达到目的地，没有无限延迟。

根据 FLP 定理，在一个全异步系统中，不存在确定性的能保证 Safety 和 Progressive 的共识算法。这其中最关键的困难在于异步系统的 failure 无法被可靠的、在有限时间内检测出来。因此在论证 Paxos 的正确性时，需要带上一定的 liveness 假设；在 Paxos 的实际应用中，需要额外的手段来保证 liveness 假设为真。

## 问题描述和共识算法“接口”

如何让系统中的一组参与者达成某个值的共识？这里，系统的参与者可以有两种角色：Proposer 和 Acceptor。Proposer 的职责是提出某个值让系统达成共识，Acceptor 的职责是决定是否接受 Proposer 提出的值。

如果要说某个算法解决了该问题，必须满足以下五个条件，包括三个 Safety 条件（保证数据一致且有效）和两个 Progress 条件（保证算法能结束）。

- __Non-trivality__: 值必须由某个参与者 propose （提出）。（不能从上帝视角给出固定的一个值
- __Safety__: 如果某个值已经被 decide （最终确定），不会再有其他值被 decide。（不能每个 propose 的值都被 decide
- __Safe Learning__: 如果某个参与者 learn （知悉）到了一个值，该值必须已被 decided。
- __Progress__: 在特定的一组 liveness 条件下，如果有参与者 propose 了值，最终一定会有某个值被 decided。（不能一直不 decide
- __Eventual Learning__: 在特定的一组 liveness 条件下，如果有某个值被 decide 了，最终一定会有某个值被 learned。

注意到，这里 Safety 和 Progressive 的前提是互不干扰的：liveness 条件即使发生变化，也只会影响算法的 Progress，不影响 Safety。换句话说，liveness 条件只是用于防止类似死锁或者无限活锁的情况发生。

至此，我们定义了共识算法的“接口”，至于如何“实现”这个算法，则是另一个问题了。

## 稻草人算法

在介绍经典 Paxos 之前，首先给出一个单 Accpetor 的算法(SAA)，说明它是一种共识算法，并且解决了以上提出的问题。SAA 顾名思义，所有的参与者中只有一个 Acceptor，其他都是 Proposer。类似于 Signleton Pattern 的单例只能初始化一次，Acceptor 只接受 Proposer 提出的第一个值作为 decided 值，并且 decided 后通知其他 Proposer。对照上面的定义，易得该算法满足 Safety（详细的证明可以参考论文原文）。

此外，liveness 条件为：Acceptor 在线，且至少有一个 Proposer 在线。易得该算法满足 Progress。

![](/assets/single-acceptor-algo.png)

分析其所有参与者达成一致的消耗，结果是一个 round trip + 一个同步的持久化（Acceptor 接受该值时用到）。这样的共识算法是否是一个“好”的算法呢？根据 liveness 条件，它的 Acceptor 必须在线，也就是说会存在单点瓶颈和故障问题。从实践的角度来看，相当于只有一个存储节点，违背了分布式存储的初衷。

该例子说明了共识算法的“接口”能够表示多种算法“实现”，或者说有很多种具体方案能够满足共识算法的定义，该定义具备一定的抽象和描述能力。

## 经典 Paxos

这里所说的“经典 Paxos” 并非 Paxos 的第一个版本，而是一个具有最小特性的版本，因此如果觉得有哪些地方遗漏，则它们很可能不是必要的最小组成部分。

基本概念： 
- epoch：一个属于某全序集的数字，常常和某个 value 绑定。
- proposal：一个二元组，包含 epoch 和 value。

经典 Paxos 同样只有 Proposer 和 Acceptor 两个角色，参与者达成某个值共识的判定标准是什么？按照经典 Paxos 的话来说：

`在值 v 上达成共识` 等价于 `值 v 被 decided` 等价于 `大多数的 Acceptor 接受了 v`。

Proposer 和 Acceptor 各自包含两个阶段，可以总结为读阶段和写阶段。Proposer 本身无状态，Acceptor 有三个状态，分别是 `last_promise_epoch`，`last_accept_epoch` 以及 `last_accept_value`，初始值均为空。具体流程如下：

第一阶段——读阶段：
- Proposer: 获取一个新的、当前已分配的最大的 epoch，e，向所有的 Acceptor 发送 `prepare(e)`。
- Acceptor: 接受到 prepare 请求后，如果
  - `last_promise_epoch` 为空，则返回 `primise(e, nil, nil)`
  - `last_promise_epoch` 不为空，则提取 prepare 中的 epoch，`e`，如果
    -  `e` >= `last_promise_epoch`，则更新 `last_promise_epoch` 为 e，并返回 `promise(e, last_accept_epoch, last_accept_value)`。
    -  `e` < `last_promise_epoch`，什么都不做。

第二阶段——写阶段：
- Proposer: 接受到大多数的 Acceptor 发送的 promise 后，提取所有 promise 中，最大的 `last_accept_epoch` 和相应的 `last_accept_value`，
  - 如果最大 epoch 为空（即每个 promise 中的 `last_accept_epoch` 都为空），则 Proposer 可以自行指定一个值来 propose。
  - 如果存在最大 epoch（max_acc_epoch），则向所有的 Acceptor 发送 `propose(epoch, max_acc_epoch, max_acc_val)`。
- Accpetor: 收到 propose 之后，提取 epoch，e，如果
  -  `e` >= `last_promise_epoch` 更新 `last_pro_epoch`/`last_accept_epoch`，保存 value，然后返回 `accept(epoch)`。
  -  `e` < `last_promise_epoch`，什么都不做。

Proposer 还存在一个超时机制，如果始终无法接收到大多数 Acceptor 的回复，则 Proposer 的算法会从头开始执行。

总结亿下，

- Proposer 能够发送两种结构，分别是 `prepare(epoch)` 和 `propose(epoch, max_acc_epoch, value)`。
- Acceptor 也能发送两种结构，分别是 `promise(epoch, last_accept_epoch, last_accept_value)` 和 `accept(epoch)`。

![](/assets/classic-paxos-messages.png)

从 Proposer 的视角来看，prepare 有两个功能：一是通知 Acceptor，比我这次 epoch 小的消息都不要理会；二是了解各个 Acceptor 当前的状态：目前在哪个 epoch 接受了哪个值。当 Proposer 收到了大多数的 Acceptor 回复时，为了让系统快速达成共识，选择 Acceptor 回复中 last_accpt_epoch 最大的 proposal（更有可能被尽可能多的 Acceptor 接受），发送该 proposal。

从 Acceptor 的视角来看，接收到 prepare 时，向 Proposer 给出一个“保证”：接下来如果没有意外，我将接受你的 proposal。这里指的意外情况是，在此期间存在另一个新的 prepare（要求我给出另一个保证），或者另一个新的 propose（要求我接受）。Acceptor 通过只接受 epoch 递增才处理的方式，保证一个 epoch 只能对应一个值。

![](/assets/classic-paxos-best-case.png)

![](/assets/classic-paxos-timeout.png)

基于以上的算法，可以得出以下几个性质：

1. Proposer 每次使用独立的 epoch 作为 proposal。
2. Proposer 只在收到大多数的 Acceptor 的 promise 之后才进行 propose。
3. Proposer 只在收到大多数的 Acceptor 的 accept 之后才进行 return（或 decide）。
4. Proposer 必须根据值的选择规则进行选择。使用收到的所有 promise(epoch, value) 中的 epoch 最大值来 propose。
5. Proposer 每次使用的 epoch 一定大于以前使用过的 epoch。
6. Acceptor 只有当 epoch 大于或等于上次 promise 时的 epoch 才进行处理。
7. Acceptor 对 prepare/propose 进行处理时，会将 last promise 的 epoch 更新。
8. Acceptor 面对每个处理的 prepare，一定会返回 promise。
9. Acceptor 面对每个处理的 propose，在更新 epoch 之后一定返回 accept。
10. Acceptor 的上次 promise/accept 的 epoch 是持久化的，并且只会被 7 或者 9 更新。

这些性质被用于以组合的方式证明经典 Paxos 满足 Safety 和 Progress，这里列出来只是为了帮助理解。详细的证明可参考论文原文。

经典 Paxos 的 liveness 条件是 Acceptor 至少有 `floor(n/2) + 1` 个在线，有且只有一个 Proposer 在线。

- Acceptor 在线数量为 `floor(n/2) + 1` 的原因是 Proposer 在进行下一步行动之前必须收到 “大多数” Acceptor 的回复，而如果 Acceptor 在线数量少于该值，则 Proposer 会处于永远的超时状态；
- 只有一个 Proposer 在线的原因是如果存在多个 Proposer，可能存在 prepare 信号无限活锁的情况。

在最好情况下，参与者达成共识的开销为 2 * majority 个 round trip + 3 * majority 个同步持久化。

相比于稻草人算法，经典 Paxos 实现了 `ceil(n/2) - 1` 个 Acceptor 的 fault tolerance，但仍然存在 Proposer 的单点瓶颈和故障问题。

## 经典 Paxos 的优化

经典的 Paxos 算法已经能够保证 Safety 和给定 liveness 条件下的 Progress，但其中有很多地方值得改进。以下是论文中列出的一些改进方案：

### Negative responses(NACK)

“如果你不接受某件事，应该大胆地说出来，而不是沉默”。

当 Acceptor 不能给出 promise 或者 accept 信息时，应该主动发送一个 `no-promise` 或者 `no-accept`，而不是让 Proposer 等待超时。Proposer 此时还可以继续等待其他的 Acceptor，如果大多数的 Acceptor 都发送了此类 NACK，则可以直接获取下一个 epoch 进行下一轮的 prepare。

这样有助于缩短 Proposer 的平均等待超时时间。

`no-promise`/`no-accept` 中还可以包含 `last_promise_epoch`/`last_accept_epoch`，以及 `last_accept_value`。便于 Proposer 跳过某些 epoch，或者直接发起 propose。（在 Raft 中，这类的 NACK 分别称为 AppendEntries 和 RequestVote）。

![](/assets/paxos-NACK.png)

### Bypassing Phase Two

如果 Proposer 的第一阶段（即 prepare）得到 Acceptor 的 promise 回复中，大多数的 Acceptor 的 `last_accept_epoch` 和 `last_accept_value` 的值都相同，根据定义，可以跳过第二阶段，直接得出值已经 decided 的结论。

![](/assets/paxos-bypass-phase2.png)

为了增加跳过第二阶段的可能性，可以使用以下方法：

- 当得到许多相同的（但未到大多数）`last_accept_epoch` 时，可以继续等待，但需要保留超时机制，来保证 progress。
- 等待的过程中，并发地开启第二阶段，如果第一阶段的 “大多数” 条件已经满足，则可以提前结束。
- 除了最大的 epoch 对应的 proposal 以外，可以关注具有较小 epoch 的 proposal 中的 `last_accept_epoch`。
- 可以重用 promise/记录 NACK 的 proposal/让 acceptor 记录所有的 accept proposal。

### Termination

经典 Paxos 中，即使某个值已经被 decided，参与者不一定能及时知道。

为了加快这一过程，可以增加一个阶段三，当 Proposer 要返回时，可以向所有/其中的几个/一个 Acceptor 发送 decide(value) 消息，让其他 Acceptor 都知道 epoch 对应的值已被 decided。这样未来的 Proposer 和这些 Acceptor 交互时，能够快速的知道是否已经有值被 decided。 

这样 liveness 的要求可以被放松为：大多数 Accetpor 在线 __或者__ 至少一个 Acceptor 能够返回 decide(value)。

（Raft 和 VRR 使用 commit index 而不是阶段三来通知 Acceptors 某个值已经被 decided）

![](/assets/paxos-termination.png)

### Distinguished Proposer

注意到，经典 Paxos 的 liveness 条件之一是，__有且只有一个 Proposer 在线__。原因是多个 Proposer 可能带来无限活锁问题（即一直处于阶段一，多个 Proposer 的 propose 一直超时），违反了 Progress 原则。

为了使存在多个 Propser 也能 Progressive，可以指定一个 distinguished 的 Proposer，让其他的 Proposer 将它们的候选值转发给该 Proposer，代理 propose。

注意，在参与者中，distinguished Proposer 的数量不论是没有、一个或者多个，都不影响 Safety，因此本身不是一个 Leader election 的问题。但是，为了保证 progressive，需要依赖可靠的错误检测，通常可以用带有心跳和超时的检测器模拟，但这同时也需要更强的 liveness 假设，包括：消息延迟时间有上限、时钟漂移的限度等。

（外部客户端直接发送候选值给 distinguished Proposer，例如 Raft 和 VRR；或者客户端广播给所有 Proposer，例如 Moderately Complex Paxos）

### Phase Ordering

如果第一阶段的 promise 返回的 epoch 大多数是 nil，才需要 Proposer 自己去提出一个值。因此可以惰性地获取这个值，提高 Classic Paxos 的性能。

### Multi-Paxos

Multi-Paxos 是针对 Classic Paxos 在值序列上的优化。和使用多个 Classic Paxos 实例不同，Multi-Paxos 的某个 Propser 第一阶段结束后，会成为 Leader，然后它可以对一整个值序列进行不断的 propose。如果该 Leader fail 了，可以再次执行第一阶段选择一个新的 Leader，该过程称为 Leader election。

它的优势在于当处于稳定态时，参与者达成共识只需要一个 round trip + 一次同步持久化。但是，在 Leader 上具有十分显著的负载，包括：

- 接受来自其他 Proposer 的候选值
- 对值序列的每个值进行索引
- 向 Acceptor propose 值
- 收集 Acceptor 回复的消息
- Learn decided 值并通知其他参与者

### Roles

Paxos 中角色的数量是可以变化的。

例如，将 Acceptor 和 Proposer 的角色合并在一起，称为 co-located，可以减少一次消息交流。另外合并后的 Proposer 可以直接使用 Acceptor 上的 `last_promise_epoch` 来 propose。（Simple Paxos, Chubby, Mencius, VRR, Raft 和 Moderately Complex Paxos 采用了合并角色的做法）。

相反，还可以增加一个角色叫 Reader，只与 Acceptor 交互并尝试获取是否有某个值被 decided 的信息。另一种是 recovery proposer，与普通的 Proposer 不同，如果第一阶段没有返回值，它会主动退出而不是继续执行。

### Epochs

获取 epoch 的值也可以有多种方式。

- 最经典的方法是使用全序的自然数，在每个 Proposer 上使用轮巡分配的方式划分。
- Epoch 还可以用 `(sid, pid)` 的方法表示，其中 sid 为每个 Proposer 独立持久化的自然数，pid 是 Proposer 唯一的 ID。

以上两种方法在 Proposer 开始之前需要进行一次同步的持久化。要避免每次分配都进行同步持久化，可以：

- 使用 `(sid, pid, vid)`，其中 `vid` 只在 Proposer 重启的时候持久化更新，用于掉电重启时保证唯一性，而 `sid` 和 `pid` 可以是 volatile 的。
- 给 epoch 设定一个上界也可以避免大部分的同步持久化更新。
- 获取完 epoch 之后，对本地 epoch 的更新还可以和第一阶段并发进行。

（Simple Paxos, Chubby， VRR 和 Moderately Complex Paxos 都使用了不同于 Classic Paxos 的 epoch 分配方式）

### Phase One Voting for Epochs

如果 Acceptor 要求 Proposer 的 epoch 必须严格大于 `last_promise_epoch` 才回复，意味着同一个 epoch 只有一个 Proposer 能够到达第二阶段，这样能够保证一个 epoch 最多对应一个值。此时不同的 Proposer 之间即使使用相同的 epoch，也不会造成冲突。

（在 Raft 中使用 Voting 来达到分配 epoch 的目的）

### Proposal Copying

Proposal Copying 是指向 Acceptor 直接发送现有的 proposal，而不是每次 propose 都使用新的 epoch。

- 利用 NACK 返回的 `no-promise(e,f,g,w)` 或者 `no-accept(e,f,g,w)`，其中 `f`，`g`，`w` 分别代表 `last_promise epoch`、`last_accept_epoch` 和 `last_accept_value`。当一个 Proposer 接收到类似的 NACK 时，它知道 epoch `g` 可能已经和 `w` 绑定，则可以直接复制 `proposal(g,w)`（因为已经存在某个 Proposer 完成了阶段一），跳到阶段二。
- 对于 co-located（即合并 Proposer 和 Accetpor 角色）且使用了 termination 优化（即 Proposer 在返回之前向所有 Acceptor 发送 decide 信息）的 Paxos，如果 Proposer 的部分超时了，且 Acceptor 部分已经 accept 了某个 proposal，则 Proposer 可以复制该 proposal 再次发送。

### Generalisation to Quorums

在某个值被 deicide 的定义中提到，必须 “大多数” 的 Acceptor 接受了才算 decided。但是，实际上不一定需要大多数：只要保证 quorum 集合内任意 quorum 相交即可，即每次 prepare/propose 访问的 Acceptors 之间存在相交的部分。对于 quorum 集合的定义，可以综合 quorum 大小，fault tolerate 数量，节点数量，每个节点所扮演角色来进行调整。

论文的第三章根据阶段划分 quorum 集为 Q1，Q2。并且根据 epoch 划分出某个 epoch 上的 quorum，使 Quorum 的相交条件更具备通用性，使修正后的 Paxos 的灵活性更强，感兴趣的读者可以前往阅读。

### Miscellaneous

- Messages: Proposer 的两个阶段可以只发送 majority 个 message，而不是向所有的 Acceptor 发送，可以降低消息传递的成本，只在有需要时才向更多的 Acceptor 发送（Ring Paxos）。如果想让消息传递成本最小化，可以让 Acceptor 也进行转发（Ring Paxos），但这样做会增加总体延迟。

其他变种还包括 Learner、更严格的 epoch 选择条件、Fail-stop model、Virtual sequences、Fast Paxos，详细的信息可以参考论文原文或者相关的文献。

## 总结

这篇博客搬运了 [Distributed Consensus Revised](https://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-935.pdf) 论文定义的经典 Paxos 以及前人已探索的 Paxos 修订版本和优化方法。

在描述了要解决的问题，以及清晰地定义了“怎样才算解决了问题”之后，可以看到经典 Paxos 是十分简洁的其中一个解决方案。然后，通过在经典 Paxos 的基础上不断的优化，包括提前结束算法、缩短延迟、减少消息传递次数等，能够使 Paxos 的性能得到提升。
