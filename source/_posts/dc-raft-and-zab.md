---
layout: post
title: 分布式系统理论进阶 - Raft、Zab
date: 2017-1-17 12:39:04
tags: [Distributed Computing]
categories: [Distributed Computing]
---


## 引言

[分布式系统理论进阶 - Paxos](/2017/01/17/dc-paxos/ "分布式系统理论进阶 - Paxos") 介绍了一致性协议 Paxos，今天我们来学习另外两个常见的一致性协议——Raft 和 Zab。通过与 Paxos 对比，了解 Raft 和 Zab 的核心思想、加深对一致性协议的认识。

***

<!-- more -->

## Raft

Paxos 偏向于理论、对如何应用到工程实践提及较少。理解的难度加上现实的骨感，在生产环境中基于 Paxos 实现一个正确的分布式系统非常难：

> There are significant gaps between the description of the Paxos algorithm and the needs of a real-world system. In order to build a real-world system, an expert needs to use numerous ideas scattered in the literature and make several relatively small protocol extensions. The cumulative effort will be substantial and the final system will be based on an unproven protocol.

![raft](/images/dc/raft.png "raft")

Raft 在 2013 年提出，提出的时间虽然不长，但已经有很多系统基于 Raft 实现。相比 Paxos，Raft 的买点就是更利于理解、更易于实行。

为达到更容易理解和实行的目的，Raft 将问题分解和具体化：Leader 统一处理变更操作请求，一致性协议的作用具化为保证节点间操作日志副本 (log replication) 一致，以 term 作为逻辑时钟 (logical clock) 保证时序，节点运行相同状态机 (state machine) 得到一致结果。Raft 协议具体过程如下：

![raft-1](/images/dc/raft-1.png "raft-1")

1. Client 发起请求，每一条请求包含操作指令
2. 请求交由 Leader 处理，Leader 将操作指令 (entry) 追加 (append) 至操作日志，紧接着对 Follower 发起 AppendEntries 请求、尝试让操作日志副本在 Follower 落地
3. 如果 Follower 多数派 (quorum) 同意 AppendEntries 请求，Leader 进行 commit 操作、把指令交由状态机处理
4. 状态机处理完成后将结果返回给 Client

指令通过 log index(指令 id) 和 term number 保证时序，正常情况下 Leader、Follower 状态机按相同顺序执行指令，得出相同结果、状态一致。

宕机、网络分化等情况可引起 Leader 重新选举 (每次选举产生新 Leader 的同时，产生新的 term)、Leader/Follower 间状态不一致。Raft 中 Leader 为自己和所有 Follower 各维护一个 nextIndex 值，其表示 Leader 紧接下来要处理的指令 id 以及将要发给 Follower 的指令 id，LnextIndex 不等于 FnextIndex 时代表 Leader 操作日志和 Follower 操作日志存在不一致，这时将从 Follower 操作日志中最初不一致的地方开始，由 Leader 操作日志覆盖 Follower，直到 LnextIndex、FnextIndex 相等。

Paxos 中 Leader 的存在是为了提升决议效率，Leader 的有无和数目并不影响决议一致性，Raft 要求具备唯一 Leader，并把一致性问题具体化为保持日志副本的一致性，以此实现相较 Paxos 而言更容易理解、更容易实现的目标。

***

## Zab

Zab 的全称是 Zookeeper atomic broadcast protocol，是 Zookeeper 内部用到的一致性协议。相比 Paxos，Zab 最大的特点是保证强一致性 (strong consistency，或叫线性一致性 linearizable consistency)。

和 Raft 一样，Zab 要求唯一 Leader 参与决议，Zab 可以分解成 discovery、sync、broadcast 三个阶段：

![zab](/images/dc/zab.jpg "zab")

- discovery: 选举产生 PL(prospective leader)，PL 收集 Follower epoch(cepoch)，根据 Follower 的反馈 PL 产生 newepoch(每次选举产生新 Leader 的同时产生新 epoch，类似 Raft 的 term)
- sync: PL 补齐相比 Follower 多数派缺失的状态、之后各 Follower 再补齐相比 PL 缺失的状态，PL 和 Follower 完成状态同步后 PL 变为正式 Leader(established leader)
- broadcast: Leader 处理 Client 的写操作，并将状态变更广播至 Follower，Follower 多数派通过之后 Leader 发起将状态变更落地 (deliver/commit)

Leader 和 Follower 之间通过心跳判别健康状态，正常情况下 Zab 处在 broadcast 阶段，出现 Leader 宕机、网络隔离等异常情况时 Zab 重新回到 discovery 阶段。

了解完 Zab 的基本原理，我们再来看 Zab 怎样保证强一致性，Zab 通过约束事务先后顺序达到强一致性，先广播的事务先 commit、FIFO，Zab 称之为 primary order(以下简称 PO)。实现 PO 的核心是 zxid。

Zab 中每个事务对应一个 zxid，它由两部分组成：<e, c>，e 即 Leader 选举时生成的 epoch，c 表示当次 epoch 内事务的编号、依次递增。假设有两个事务的 zxid 分别是 z、z'，当满足 z.e < z'.e 或者 z.e = z'.e && z.c < z'.c 时，定义 z 先于 z'发生 (z < z')。

为实现 PO，Zab 对 Follower、Leader 有以下约束：

1. 有事务 z 和 z'，如果 Leader 先广播 z，则 Follower 需保证先 commit z 对应的事务
2. 有事务 z 和 z'，z 由 Leader p 广播，z'由 Leader q 广播，Leader p 先于 Leader q，则 Follower 需保证先 commit z 对应的事务
3. 有事务 z 和 z'，z 由 Leader p 广播，z'由 Leader q 广播，Leader p 先于 Leader q，如果 Follower 已经 commit z，则 q 需保证已 commit z 才能广播 z'

第 1、2 点保证事务 FIFO，第 3 点保证 Leader 上具备所有已 commit 的事务。

相比 Paxos，Zab 约束了事务顺序、适用于有强一致性需求的场景。

## Paxos、Raft、Zab 再比较

除 Paxos、Raft 和 Zab 外，Viewstamped Replication(简称 VR)也是讨论比较多的一致性协议。这些协议包含很多共同的内容 (Leader、quorum、state machine 等)，因而我们不禁要问：Paxos、Raft、Zab 和 VR 等分布式一致性协议区别到底在哪，还是根本就是一回事？

Paxos、Raft、Zab 和 VR 都是解决一致性问题的协议，Paxos 协议原文倾向于理论，Raft、Zab、VR 倾向于实践，一致性保证程度等的不同也导致这些协议间存在差异。下图帮助我们理解这些协议的相似点和区别：

![p-r-z](/images/dc/p-r-z.jpg "p-r-z")

相比 Raft、Zab、VR，Paxos 更纯粹、更接近一致性问题本源，尽管 Paxos 倾向理论，但不代表 Paxos 不能应用于工程。基于 Paxos 的工程实践，须考虑具体需求场景 (如一致性要达到什么程度)，再在 Paxos 原始语意上进行包装。

***

## 小结

以上介绍分布式一致性协议Raft、Zab的核心思想，分析Raft、Zab与Paxos的异同。实现分布式系统时，先从具体需求和场景考虑，Raft、Zab、VR、Paxos等协议没有绝对地好与不好，只是适不适合。
