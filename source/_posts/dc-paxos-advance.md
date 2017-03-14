---
layout: post
title: 分布式系统理论进阶 - Paxos 变种和优化
date: 2017-1-17 12:39:04
tags: [Distributed Computing]
categories: [Distributed Computing]
---

## 引言

[分布式系统理论进阶 - Paxos](/2017/01/17/dc-paxos/ "分布式系统理论进阶 - Paxos") 中我们了解了 Basic Paxos、Multi Paxos 的基本原理，但如果想把 Paxos 应用于工程实践，了解基本原理还不够。

![paxos-a](/images/dc/paxos-a.jpg "paxos-a")

有很多基于 Paxos 的优化，在保证一致性协议正确 (safety) 的前提下，减少 Paxos 决议通信步骤、避免单点故障、实现节点负载均衡，从而降低时延、增加吞吐量、提升可用性，下面我们就来了解这些 Paxos 变种。

<!-- more -->

## Multi Paxos

首先我们来回顾一下 Multi Paxos，Multi Paxos 在 Basic Paxos 的基础上确定一系列值，其决议过程如下：

![multi-paxos-1](/images/dc/multi-paxos-1.png "multi-paxos-1")

1. phase1a: leader 提交提议给 acceptor
2. phase1b: acceptor 返回最近一次接受的提议 (即曾接受的最大的提议 ID 和对应的 value)，未接受过提议则返回空
3. phase2a: leader 收集 acceptor 的应答，分两种情况处理
  - phase2a.1: 如果应答内容都为空，则自由选择一个提议 value
  - phase2a.2: 如果应答内容不为空，则选择应答里面 ID 最大的提议的 value
4. phase2b: acceptor 将决议同步给 learner

Multi Paxos 中 leader 用于避免活锁，但 leader 的存在会带来其他问题，一是如何选举和保持唯一 leader(虽然无 leader 或多 leader 不影响一致性，但影响决议进程 progress)，二是充当 leader 的节点会承担更多压力，如何均衡节点的负载。Mencius提出节点轮流担任 leader，以达到均衡负载的目的；租约 (lease) 可以帮助实现唯一 leader，但 leader 故障情况下可导致服务短期不可用。

## Fast Paxos

在 Multi Paxos 中，proposer -> leader -> acceptor -> learner，从提议到完成决议共经过 3 次通信，能不能减少通信步骤？

对 Multi Paxos phase2a，如果可以自由提议 value，则可以让 proposer 直接发起提议、leader 退出通信过程，变为 proposer -> acceptor -> learner，这就是 Fast Paxos 的由来。

![fast-paxos](/images/dc/fast-paxos.png "fast-paxos")

Multi Paxos 里提议都由 leader 提出，因而不存在一次决议出现多个 value，Fast Paxos 里由 proposer 直接提议，一次决议里可能有多个 proposer 提议、出现多个 value，即出现提议冲突 (collision)。leader 起到初始化决议进程(progress) 和解决冲突的作用，当冲突发生时 leader 重新参与决议过程、回退到 3 次通信步骤。

Paxos 自身隐含的一个特性也可以达到减少通信步骤的目标，如果 acceptor 上一次确定 (chosen) 的提议来自 proposerA，则当次决议 proposerA 可以直接提议减少一次通信步骤。如果想实现这样的效果，需要在 proposer、acceptor 记录上一次决议确定 (chosen) 的历史，用以在提议前知道哪个 proposer 的提议上一次被确定、当次决议能不能节省一次通信步骤。

## EPaxos

除了从减少通信步骤的角度提高 Paxos 决议效率外，还有其他方面可以降低 Paxos 决议时延，比如 Generalized Paxos 提出不冲突的提议 (例如对不同 key 的写请求) 可以同时决议、以降低 Paxos 时延。

更进一步地，EPaxos(Egalitarian Paxos) 提出一种既支持不冲突提议同时提交降低时延、还均衡各节点负载、同时将通信步骤减少到最少的 Paxos 优化方法。

为达到这些目标，EPaxos 的实现有几个要点。一是 EPaxos 中没有全局的 leader，而是每一次提议发起提议的 proposer 作为当次提议的 leader(command leader)；二是不相互影响 (interfere) 的提议可以同时提交；三是跳过 prepare，直接进入 accept 阶段。EPaxos 决议的过程如下：

![epaxos-1](/images/dc/epaxos-1.png "epaxos-1")

左侧展示了互不影响的两个 update 请求的决议过程，右侧展示了相互影响的两个 update 请求的决议。Multi Paxos、Mencius、EPaxos 时延和吞吐量对比：

![epaxos-2](/images/dc/epaxos-2.png "epaxos-2")

为判断决议是否相互影响，实现 EPaxos 得记录决议之间的依赖关系。

***

## 小结

以上介绍了几个基于 Paxos 的变种，Mencius 中节点轮流做 leader、均衡节点负载，Fast Paxos 减少一次通信步骤，Generalized Paxos 允许互不影响的决议同时进行，EPaxos 无全局 leader、各节点平等分担负载。

优化无止境，对Paxos也一样，应用在不同场景和不同范围的Paxos变种和优化将继续不断出现。
