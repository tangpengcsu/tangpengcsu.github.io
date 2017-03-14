---
layout: post
title: 分布式系统理论基础 -- 选举、多数派和租约
date: 2017-1-17 12:39:04
tags: [Distributed Computing]
categories: [Distributed Computing]
---

选举 (election) 是分布式系统实践中常见的问题，通过打破节点间的对等关系，选得的 leader(或叫 master、coordinator)有助于实现事务原子性、提升决议效率。 多数派 (quorum) 的思路帮助我们在网络分化的情况下达成决议一致性，在 leader 选举的场景下帮助我们选出唯一 leader。租约 (lease) 在一定期限内给予节点特定权利，也可以用于实现 leader 选举。

下面我们就来学习分布式系统理论中的选举、多数派和租约。

<!-- more -->

## 选举 (electioin)

一致性问题 (consistency) 是独立的节点间如何达成决议的问题，选出大家都认可的 leader 本质上也是一致性问题，因而如何应对宕机恢复、网络分化等在 leader 选举中也需要考量。

Bully 算法是最常见的选举算法，其要求每个节点对应一个序号，序号最高的节点为 leader。leader 宕机后次高序号的节点被重选为 leader，过程如下：

![electioin](/images/dc/electioin.png "/images/dc/electioin.png")

- (a). 节点 4 发现 leader 不可达，向序号比自己高的节点发起重新选举，重新选举消息中带上自己的序号
- (b)(c). 节点 5、6 接收到重选信息后进行序号比较，发现自身的序号更大，向节点 4 返回 OK 消息并各自向更高序号节点发起重新选举
- (d). 节点 5 收到节点 6 的 OK 消息，而节点 6 经过超时时间后收不到更高序号节点的 OK 消息，则认为自己是 leader
- (e). 节点 6 把自己成为 leader 的信息广播到所有节点

回顾 [分布式系统理论基础 - 一致性、2PC 和 3PC](/2017/01/17/dc-consistency/) 就可以看到，Bully 算法中有 2PC 的身影，都具有提议 (propose) 和收集反馈 (vote) 的过程。

在一致性算法 [Paxos](/2017/01/17/dc-paxos/)、ZAB、Raft 中，为提升决议效率均有节点充当 leader 的角色。ZAB、Raft 中描述了具体的 leader 选举实现，与 Bully 算法类似 ZAB 中使用 zxid 标识节点，具有最大 zxid 的节点表示其所具备的事务 (transaction) 最新、被选为 leader。

***

## 多数派 (quorum)

在网络分化的场景下以上 Bully 算法会遇到一个问题，被分隔的节点都认为自己具有最大的序号、将产生多个 leader，这时候就需要引入多数派 (quorum)。多数派的思路在分布式系统中很常见，其确保网络分化情况下决议唯一。

多数派的原理说起来很简单，假如节点总数为 2f+1，则一项决议得到多于 f 节点赞成则获得通过。leader 选举中，网络分化场景下只有具备多数派节点的部分才可能选出 leader，这避免了多 leader 的产生。

![quorum](/images/dc/quorum.png "/images/dc/quorum.png")

多数派的思路还被应用于副本 (replica) 管理，根据业务实际读写比例调整写副本数 Vw、读副本数 Vr，用以在可靠性和性能方面取得平衡。

***

## 租约 (lease)

选举中很重要的一个问题，以上尚未提到：怎么判断 leader 不可用、什么时候应该发起重新选举？最先可能想到会通过心跳 (heart beat) 判别 leader 状态是否正常，但在网络拥塞或瞬断的情况下，这容易导致出现双主。

租约 (lease) 是解决该问题的常用方法，其最初提出时用于解决分布式缓存一致性问题，后面在分布式锁 等很多方面都有应用。

![lease](/images/dc/lease.png "/images/dc/lease.png")

租约的原理同样不复杂，中心思想是每次租约时长内只有一个节点获得租约、到期后必须重新颁发租约。假设我们有租约颁发节点 Z，节点 0、1 和 2 竞选 leader，租约过程如下：

- (a). 节点 0、1、2 在 Z 上注册自己，Z 根据一定的规则 (例如先到先得) 颁发租约给节点，该租约同时对应一个有效时长；这里假设节点 0 获得租约、成为 leader
- (b). leader 宕机时，只有租约到期 (timeout) 后才重新发起选举，这里节点 1 获得租约、成为 leader

租约机制确保了一个时刻最多只有一个 leader，避免只使用心跳机制产生双主的问题。在实践应用中，zookeeper、ectd 可用于租约颁发。

***

## 小结

在分布式系统理论和实践中，常见 leader、quorum 和 lease 的身影。分布式系统内不一定事事协商、事事民主，leader 的存在有助于提升决议效率。

本文以 leader 选举作为例子引入和讲述 quorum、lease，当然 quorum 和 lease 是两种思想，并不限于 leader 选举应用。
