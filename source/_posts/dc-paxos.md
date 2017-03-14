---
layout: post
title: 分布式系统理论进阶 - Paxos
date: 2017-1-17 12:39:04
tags: [Distributed Computing]
categories: [Distributed Computing]
---

## 引言

[分布式系统理论基础 -- 一致性、2PC 和 3PC](/2017/01/17/dc-consistency/ "分布式系统理论基础 -- 一致性、2PC 和 3PC") 一文介绍了一致性、达成一致性需要面临的各种问题以及 2PC、3PC 模型，Paxos 协议在节点宕机恢复、消息无序或丢失、网络分化的场景下能保证决议的一致性，是被讨论最广泛的一致性协议。

Paxos 协议同时又以其 “艰深晦涩” 著称，下面结合 [Paxos Made Simple](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf "Paxos Made Simple")、[The Part-Time Parliament](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf "The Part-Time Parliament") 两篇论文，尝试通过 Paxos 推演、学习和了解 Paxos 协议。

***

<!-- more -->

## Basic Paxos

何为一致性问题？简单而言，一致性问题是在节点宕机、消息无序等场景可能出现的情况下，相互独立的节点之间如何达成决议的问题，作为解决一致性问题的协议，Paxos 的核心是节点间如何确定并只确定一个值 (value)。

也许你会疑惑只确定一个值能起什么作用，在 Paxos 协议里确定并只确定一个值是确定多值的基础，如何确定多值将在第二部分 Multi Paxos 中介绍，这部分我们聚焦在 “Paxos 如何确定并只确定一个值” 这一问题上。


![basic-paxos](/images/dc/basic-paxos.gif "basic-paxos")

和 2PC 类似，Paxos 先把节点分成两类，发起提议 (proposal) 的一方为 proposer，参与决议的一方为 acceptor。假如只有一个 proposer 发起提议，并且节点不宕机、消息不丢包，那么 acceptor 做到以下这点就可以确定一个值：

### P1. 一个 acceptor 接受它收到的第一项提议

当然上面要求的前提条件有些严苛，**节点不能宕机、消息不能丢包**，还只能由一个 proposer 发起提议。我们尝试放宽条件，假设多个 proposer 可以同时发起提议，又怎样才能做到确定并只确定一个值呢？

首先 proposer 和 acceptor 需要满足以下两个条件：

1. proposer 发起的每项提议分别用一个 ID 标识，提议的组成因此变为 (ID, value)
2. acceptor 可以接受 (accept) 不止一项提议，当多数(quorum) acceptor 接受一项提议时该提议被确定(chosen)

> 注意以上 “接受” 和“确定”的区别

我们约定后面发起的提议的 ID 比前面提议的 ID 大，并假设可以有多项提议被确定，为做到确定并只确定一个值 acceptor 要做到以下这点：

### P2. 如果一项值为 v 的提议被确定，那么后续只确定值为 v 的提议

> 乍看这个条件不太好理解，谨记目标是 “确定并只确定一个值”

由于一项提议被确定 (chosen) 前必须先被多数派 acceptor 接受(accepted)，为实现 P2，实质上 acceptor 需要做到：

#### P2a. 如果一项值为 v 的提议被确定，那么 acceptor 后续只接受值为 v 的提议

满足 P2a 则 P2 成立 (P2a => P2)。

目前在多个 proposer 可以同时发起提议的情况下，满足 P1、P2a 即能做到确定并只确定一个值。如果再加上节点宕机恢复、消息丢包的考量呢？

假设 acceptor c 宕机一段时间后恢复，c 宕机期间其他 acceptor 已经确定了一项值为 v 的决议但 c 因为宕机并不知晓；c 恢复后如果有 proposer 马上发起一项值不是 v 的提议，由于条件 P1，c 会接受该提议，这与 P2a 矛盾。为了避免这样的情况出现，进一步地我们对 proposer 作约束：

#### P2b. 如果一项值为 v 的提议被确定，那么 proposer 后续只发起值为 v 的提议

满足 P2b 则 P2a 成立 (P2b => P2a => P2)。

P2b 约束的是提议被确定 (chosen) 后 proposer 的行为，我们更关心提议被确定前 proposer 应该怎么做：

#### P2c. 对于提议 (n,v)，acceptor 的多数派 S 中，如果存在 acceptor 最近一次(即 ID 值最大) 接受的提议的值为 v'，那么要求 v = v'；否则 v 可为任意值

满足 P2c 则 P2b 成立 (P2c => P2b => P2a => P2)。

条件 P2c 是 Basic Paxos 的核心，光看 P2c 的描述可能会觉得一头雾水，我们通过 [The Part-Time Parliament](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf "The Part-Time Parliament") 中的例子加深理解：

<table>
  <tr>
    <th rowspan="2">提议 ID</th>
    <th rowspan="2">提议值</th>
    <th colspan="5">Acceptor</th>
  </tr>
  <tr>
    <th>A</th>
    <th>B</th>
    <th>C</th>
    <th>D</th>
    <th>E</th>
  </tr>
  <tr>
    <td>2</td>
    <td>a</td>
    <td>x</td>
    <td>x</td>
    <td>x</td>
    <td>o</td>
    <td>-</td>
  </tr>
  <tr>
    <td>5</td>
    <td>b</td>
    <td>x</td>
    <td>x</td>
    <td>o</td>
    <td>-</td>
    <td>x</td>
  </tr>
  <tr>
    <td>14</td>
    <td>?</td>
    <td>-</td>
    <td>o</td>
    <td>-</td>
    <td>x</td>
    <td>o</td>
  </tr>
  <tr>
    <td>27</td>
    <td>?</td>
    <td>o</td>
    <td>-</td>
    <td>o</td>
    <td>o</td>
    <td>-</td>
  </tr>
  <tr>
    <td>29</td>
    <td>?</td>
    <td>-</td>
    <td>o</td>
    <td>x</td>
    <td>x</td>
    <td>-</td>
  </tr>
</table>

假设有 A~E 5 个 acceptor，

- \- 表示 acceptor 因宕机等原因缺席当次决议
- x 表示 acceptor 不接受提议
- o 表示接受提议

多数派 acceptor 接受提议后提议被确定，以上表格对应的决议过程如下：

1. ID 为 2 的提议最早提出，根据 P2c 其提议值可为任意值，这里假设为 a
2. acceptor A/B/C/E 在之前的决议中没有接受 (accept) 任何提议，因而 ID 为 5 的提议的值也可以为任意值，这里假设为 b
3. acceptor B/D/E，其中 D 曾接受 ID 为 2 的提议，根据 P2c，该轮 ID 为 14 的提议的值必须与 ID 为 2 的提议的值相同，为 a
4. acceptor A/C/D，其中 D 曾接受 ID 为 2 的提议、C 曾接受 ID 为 5 的提议，相比之下 ID 5 较 ID 2 大，根据 P2c，该轮 ID 为 27 的提议的值必须与 ID 为 5 的提议的值相同，为 b；该轮决议被多数派 acceptor 接受，因此该轮决议得以确定
5. acceptor B/C/D，3 个 acceptor 之前都接受过提议，相比之下 C、D 曾接受的 ID 27 的 ID 号最大，该轮 ID 为 29 的提议的值必须与 ID 为 27 的提议的值相同，为 b

以上提到的各项约束条件可以归纳为 3 点，如果 proposer/acceptor 满足下面 3 点，那么在少数节点宕机、网络分化隔离的情况下，在 “确定并只确定一个值” 这件事情上可以保证一致性(consistency)：

- B1(ß): ß 中每一轮决议都有唯一的 ID 标识
- B2(ß): 如果决议 B 被 acceptor 多数派接受，则确定决议 B
- B3(ß): 对于 ß 中的任意提议 B(n,v)，acceptor 的多数派中如果存在 acceptor 最近一次 (即 ID 值最大) 接受的提议的值为 v'，那么要求 v = v'；否则 v 可为任意值

> 希腊字母 ß 表示多轮决议的集合，字母 B 表示一轮决议

另外为保证 P2c，我们对 acceptor 作两个要求：

1. 记录曾接受的 ID 最大的提议，因 proposer 需要问询该信息以决定提议值
2. 在回应提议 ID 为 n 的 proposer 自己曾接受过 ID 最大的提议时，acceptor 同时保证 (promise) 不再接受 ID 小于 n 的提议

至此，proposer/acceptor 完成一轮决议可归纳为 prepare 和 accept 两个阶段。prepare 阶段 proposer 发起提议问询提议值、acceptor 回应问询并进行 promise；accept 阶段完成决议，图示如下：

![basic-paxos](/images/dc/basic-paxos.png "basic-paxos")

还有一个问题需要考量，假如 proposer A 发起 ID 为 n 的提议，在提议未完成前 proposer B 又发起 ID 为 n+1 的提议，在 n+1 提议未完成前 proposer C 又发起 ID 为 n+2 的提议…… 如此 acceptor 不能完成决议、形成活锁 (livelock)，虽然这不影响一致性，但我们一般不想让这样的情况发生。解决的方法是从 proposer 中选出一个 leader，提议统一由 leader 发起。

最后我们再引入一个新的角色：learner，learner 依附于 acceptor，用于习得已确定的决议。以上决议过程都只要求 acceptor 多数派参与，而我们希望尽量所有 acceptor 的状态一致。如果部分 acceptor 因宕机等原因未知晓已确定决议，宕机恢复后可经本机 learner 采用 pull 的方式从其他 acceptor 习得。

***

## Multi Paxos

通过以上步骤分布式系统已经能确定一个值，“只确定一个值有什么用？这可解决不了我面临的问题。” 你心中可能有这样的疑问。

![multi-paxos](/images/dc/multi-paxos.gif "multi-paxos")

其实不断地进行 “确定一个值” 的过程、再为每个过程编上序号，就能得到具有全序关系 (total order) 的系列值，进而能应用在数据库副本存储等很多场景。我们把单次 “确定一个值” 的过程称为实例(instance)，它由 proposer/acceptor/learner 组成，下图说明了 A/B/C 三机上的实例：

![multi-paxos](/images/dc/multi-paxos.png "multi-paxos")

不同序号的实例之间互相不影响，A/B/C 三机输入相同、过程实质等同于执行相同序列的状态机 (state machine) 指令 ，因而将得到一致的结果。

proposer leader 在 Multi Paxos 中还有助于提升性能，常态下统一由 leader 发起提议，可节省 prepare 步骤 (leader 不用问询 acceptor 曾接受过的 ID 最大的提议、只有 leader 提议也不需要 acceptor 进行 promise) 直至发生 leader 宕机、重新选主。

***

## 小结

以上介绍了 Paxos 的推演过程、如何在 Basic Paxos 的基础上通过状态机构建 Multi Paxos。Paxos 协议比较 “艰深晦涩”，但多读几遍论文一般能理解其内涵，更难的是如何将 Paxos 真正应用到工程实践。

微信后台开发同学实现并开源了一套基于Paxos协议的多机状态拷贝类库 [PhxPaxos](https://github.com/tencent-wechat/phxpaxos "PhxPaxos")，PhxPaxos 用于将单机服务扩展到多机，其经过线上系统验证并在一致性保证、性能等方面作了很多考量。
