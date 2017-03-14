---
layout: post
title: 分布式系统理论基础 -- 一致性、2PC 和 3PC
date: 2017-1-17 12:39:04
tags: [Distributed Computing]
categories: [Distributed Computing]
---

## 引言

**狭义的分布式系统** 指 *由网络连接的计算机系统，每个节点独立地承担计算或存储任务，节点间通过网络协同工作*。**广义的分布式系统** 是一个相对的概念，正如 [Leslie Lamport](https://en.wikipedia.org/wiki/Leslie_Lamport "https://en.wikipedia.org/wiki/Leslie_Lamport") 所说：

> What is a distributed systeme. Distribution is in the eye of the beholder.
To the user sitting at the keyboard, his IBM personal computer is a nondistributed system.
To a flea crawling around on the circuit board, or to the engineer who designed it, it's very much a distributed system.

**一致性** 是分布式理论中的 **根本性问题**，近半个世纪以来，科学家们围绕着一致性问题提出了很多理论模型，依据这些理论模型，业界也出现了很多工程实践投影。下面我们从一致性问题、特定条件下解决一致性问题的两种方法 (`2PC`、`3PC`) 入门，了解最基础的分布式系统理论。

<!-- more -->

***

## 一致性 (consensus)

何为一致性问题？简单而言，**一致性问题** 就是`相互独立的节点之间如何达成一项决议的问题`。分布式系统中，进行 *数据库事务提交 (commit transaction)、Leader 选举、序列号生成* 等都会遇到一致性问题。这个问题在我们的日常生活中也很常见，比如牌友怎么商定几点在哪打几圈麻将：

![赌圣,1990](/images/dc/ds.jpg "/images/dc/ds.jpg")

假设一个具有 N 个节点的分布式系统，当其满足以下条件时，我们说这个系统满足一致性：

- **全认同** (agreement): 所有 N 个节点都认同一个结果
- **值合法** (validity): 该结果必须由 N 个节点中的节点提出
- **可结束** (termination): 决议过程在一定时间内结束，不会无休止地进行下去

有人可能会说，决定什么时候在哪搓搓麻将，4 个人商量一下就 ok，这不很简单吗？

但就这样看似简单的事情，分布式系统实现起来并不轻松，因为它面临着这些问题：

- **消息传递异步无序(asynchronous)**: 现实网络不是一个可靠的信道，存在消息延时、丢失，节点间消息传递做不到同步有序 (synchronous)
- **节点宕机(fail-stop)**: 节点持续宕机，不会恢复
- **节点宕机恢复(fail-recover)**: 节点宕机一段时间后恢复，在分布式系统中最常见
- **网络分化(network partition)****: 网络链路出现问题，将 N 个节点隔离成多个部分
- **拜占庭将军问题(byzantine failure)**: 节点或宕机或逻辑失败，甚至不按套路出牌抛出干扰决议的信息

假设现实场景中也存在这样的问题，我们看看结果会怎样：

```
我: 老王，今晚 7 点老地方，搓够 48 圈不见不散！
……
（第二天凌晨 3 点） 隔壁老王: 没问题！       // 消息延迟
我: ……
----------------------------------------------
我: 小张，今晚 7 点老地方，搓够 48 圈不见不散！
小张: No ……                           
（两小时后……）
小张: No problem！                     // 宕机节点恢复
我: ……
-----------------------------------------------
我: 老李头，今晚 7 点老地方，搓够 48 圈不见不散！
老李: 必须的，大保健走起！               // 拜占庭将军
（这是要打麻将呢？还是要大保健？还是一边打麻将一边大保健……）
```

还能不能一起愉快地玩耍...

我们把以上所列的问题称为 **系统模型 (system model)**，讨论分布式系统理论和工程实践的时候，必先划定模型。例如有以下 *两种模型*：

- 异步环境 (asynchronous) 下，`节点宕机`(fail-stop)
- 异步环境 (asynchronous) 下，`节点宕机恢复`(fail-recover)、`网络分化`(network partition)

2 比 1 多了节点恢复、网络分化的考量，因而对这两种模型的理论研究和工程解决方案必定是不同的，在还没有明晰所要解决的问题前谈解决方案都是一本正经地耍流氓。

**一致性** 还具备**两个属性**，一个是 **强一致(safety)**，它要求 *所有节点状态一致、共进退*；一个是 **可用 (liveness)**，它要求 *分布式系统 24\*7 无间断对外服务*。`FLP 定理` (FLP impossibility) 已经证明在一个收窄的模型中 (异步环境并只存在节点宕机)，不能同时满足 safety 和 liveness。

FLP 定理是分布式系统理论中的基础理论，正如物理学中的能量守恒定律彻底否定了永动机的存在，FLP 定理 **否定** 了同时满足 safety 和 liveness 的一致性协议的存在。

![怦然心动 (Flipped)，2010](/images/dc/flipped.jpg "/images/dc/flipped.jpg")

工程实践上根据具体的业务场景，或保证强一致 (safety)，或在节点宕机、网络分化的时候保证可用 (liveness)。`2PC、3PC` 是相对简单的解决 `一致性` 问题的协议，下面我们就来了解 2PC 和 3PC。

***

## 2PC

**2PC(two phase commit)两阶段提交**，顾名思义它分成 *两个阶段*，*先由一方进行提议 (propose) 并收集其他节点的反馈 (vote)，再根据反馈决定提交(commit) 或中止 (abort) 事务*。我们将提议的节点称为 **协调者(coordinator)**，其他参与决议节点称为 **参与者(participants，或 cohorts)**：

![2pc](/images/dc/2pc.png "/images/dc/2pc.png")

### 2PC, phase one

在阶段 1 中，**coordinator** 发起一个提议，分别问询各 **participant** `是否接受`。

![2pc-1](/images/dc/2pc-1.png "/images/dc/2pc-1.png")

### 2PC, phase two

在阶段 2 中，coordinator 根据 participant 的反馈，`提交`或`中止`事务，*如果 participant 全部同意则提交，只要有一个 participant 不同意就中止*。

在异步环境 (asynchronous) 并且没有节点宕机 (fail-stop) 的模型下，2PC 可以满足`全认同`、`值合法`、`可结束`，是解决一致性问题的一种协议。但如果再加上节点宕机 (fail-recover) 的考虑，2PC 是否还能解决一致性问题呢？

`coordinator` 如果在`发起提议后宕机`，那么 `participant` 将进入`阻塞` (block) 状态、一直等待 coordinator 回应以完成该次决议。这时需要另一角色把系统从不可结束的状态中带出来，我们把新增的这一角色叫`协调者备份` (`coordinator watchdog`)。`coordinator 宕机`一定时间后，`watchdog` 接替原 coordinator 工作，通过问询(query) 各 participant 的状态，决定阶段 2 是`提交`还是`中止`。这也要求 coordinator/participant 记录(logging) 历史状态，以备 coordinator 宕机后 watchdog 对 participant 查询、coordinator 宕机恢复后`重新找回状态`。

从 coordinator 接收到一次事务请求、发起提议到事务完成，经过 2PC 协议后增加了 2 次 RTT(propose+commit)，带来的时延 (latency) 增加相对较少。

***

## 3PC

**3PC(three phase commit) 即三阶段提交**，既然 2PC 可以在`异步网络` + `节点宕机恢复`的模型下实现一致性，那还需要 3PC 做什么，**3PC 是什么鬼？**

在 2PC 中一个 participant 的状态只有它自己和 coordinator 知晓，假如 `coordinator 提议后自身宕机，在 watchdog 启用前一个 participant 又宕机`，其他 participant 就会进入`既不能回滚、又不能强制 commit 的阻塞状态`，`直到 participant 宕机恢复`。这引出两个疑问：

- 能不能去掉阻塞，使系统可以在 commit/abort 前回滚 (rollback) 到决议发起前的初始状态
- 当次决议中，participant 间能不能相互知道对方的状态，又或者 participant 间根本不依赖对方的状态

相比 2PC，3PC 增加了一个准备提交 (prepare to commit) 阶段来解决以上问题：

![3pc](/images/dc/3pc.png "/images/dc/3pc.png")

coordinator 接收完 participant 的反馈 (vote) 之后，进入阶段 2，给各个 participant 发送准备提交 (prepare to commit) 指令。participant 接到准备提交指令后可以锁资源，但要求相关操作必须可回滚。coordinator 接收完确认 (ACK) 后进入阶段 3、进行 commit/abort，3PC 的阶段 3 与 2PC 的阶段 2 无异。协调者备份 (coordinator watchdog)、状态记录(logging) 同样应用在 3PC。

participant 如果在`不同阶段宕机`，我们来看看 3PC 如何应对：

- **阶段 1**: coordinator 或 watchdog 未收到宕机 participant 的 vote，`直接中止事务`；宕机的 participant 恢复后，读取 logging 发现未发出赞成 vote，自行中止该次事务
- **阶段 2**: coordinator 未收到宕机 participant 的 precommit ACK，但因为之前已经收到了宕机 participant 的赞成反馈 (不然也不会进入到阶段 2)，coordinator 进行 `commit`；watchdog 可以通过问询其他 participant 获得这些信息，过程同理；宕机的 participant 恢复后发现收到 precommit 或已经发出赞成 vote，则自行 commit 该次事务
- **阶段 3**: 即便 coordinator 或 watchdog 未收到宕机 participant 的 commit ACK，也`结束该次事务`；宕机的 participant 恢复后发现收到 commit 或者 precommit，也将`自行 commit 该次事务`

因为有了准备提交 (prepare to commit) 阶段，3PC 的事务处理延时也增加了 1 个 RTT，变为 3 个 RTT(propose+precommit+commit)，但是它防止 participant 宕机后整个系统进入阻塞态，增强了系统的可用性，对一些现实业务场景是非常值得的。

## 小结

以上介绍了分布式系统理论中的部分基础知识，阐述了一致性 (consensus) 的定义和实现一致性所要面临的问题，最后讨论在异步网络 (asynchronous)、节点宕机恢复(fail-recover) 模型下 2PC、3PC 怎么解决一致性问题。
