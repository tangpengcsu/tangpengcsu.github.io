---
layout: post
title: 分布式系统理论基础 -- CAP
date: 2017-1-17 12:39:04
tags: [Distributed Computing]
categories: [Distributed Computing]
---

## 引言

CAP 是分布式系统、特别是分布式存储领域中被讨论最多的理论，“**什么是 CAP 定理？**” 在 Quora 分布式系统分类下排名 FAQ 的 No.1。CAP 在程序员中也有较广的普及，它不仅仅是 “`C、A、P 不能同时满足，最多只能 3 选 2`”，以下尝试综合各方观点，从发展历史、工程实践等角度讲述 CAP 理论。希望大家透过本文对 CAP 理论有更多地了解和认识。

***

<!-- more -->

## CAP 定理

**CAP** 由 *Eric Brewer* 在 2000 年 PODC 会议上提出，是 Eric Brewer 在 Inktomi 期间研发搜索引擎、分布式 web 缓存时得出的关于`数据一致性(consistency)`、`服务可用性(availability)`、`分区容错性(partition-tolerance)` 的猜想：

> It is impossible for a web service to provide the three following guarantees : Consistency, Availability and Partition-tolerance.

该猜想在提出两年后被证明成立，成为我们熟知的 **CAP 定理**：

- **数据一致性** (consistency)：如果系统对一个写操作返回成功，那么之后的读请求都必须读到这个新数据；如果返回失败，那么所有读操作都不能读到这个数据，对调用者而言数据具有`强一致性 (strong consistency)` (又叫`原子性 atomic`、`线性一致性 linearizable consistency`)
- **服务可用性** (availability)：所有读写请求在一定时间内得到响应，可终止、不会一直等待
- **分区容错性** (partition-tolerance)：在`网络分区`的情况下，被分隔的节点仍能正常对外服务

在某时刻如果满足 AP，分隔的节点同时对外服务但不能相互通信，将导致状态不一致，即不能满足 C；如果满足 CP，网络分区的情况下为达成 C，请求只能一直等待，即不满足 A；如果要满足 CA，在一定时间内要达到节点状态一致，要求不能出现网络分区，则不能满足 P。

`C、A、P 三者最多只能满足其中两个`，和 FLP 定理一样，CAP 定理也指示了一个不可达的结果 (impossibility result)。

![CAP-1](/images/dc/cap-1.jpg)

***

## CAP 的工程启示

CAP 理论提出 7、8 年后，NoSql 圈将 CAP 理论当作对抗传统关系型数据库的依据、阐明自己放宽对数据一致性 (consistency) 要求的正确性，随后引起了大范围关于 CAP 理论的讨论。

CAP 理论看似给我们出了一道 3 选 2 的选择题，但在工程实践中存在很多现实限制条件，需要我们做更多地考量与权衡，避免进入 CAP 认识误区。

### 关于 P 的理解

Partition 字面意思是网络分区，即因网络因素将系统分隔为多个单独的部分，有人可能会说，网络分区的情况发生概率非常小啊，是不是不用考虑 P，保证 CA 就好。要理解 P，我们看回 CAP 证明中 **P 的定义**：

> In order to model partition tolerance, the network will be allowed to lose arbitrarily many messages sent from one node to another.

**网络分区** 的情况符合该定义，*网络丢包* 的情况也符合以上定义，另外 *节点宕机*，其他节点发往宕机节点的包也将丢失，这种情况同样符合定义。*现实情况下我们面对的是一个不可靠的网络、有一定概率宕机的设备，这两个因素都会导致 Partition，因而分布式系统实现中 P 是一个必须项，而不是可选项*。

对于分布式系统工程实践，CAP 理论更合适的描述是：`在满足分区容错的前提下，没有算法能同时满足数据一致性和服务可用性`

> In a network subject to communication failures, it is impossible for any web service to implement an atomic read/write shared memory that guarantees a response to every request.

### CA 非 0/1 的选择

**P 是必选项**，那 3 选 2 的选择题不就变成数据一致性 (consistency)、服务可用性 (availability) 2 选 1？工程实践中一致性有不同程度，可用性也有不同等级，在保证分区容错性的前提下，`放宽约束后可以兼顾一致性和可用性，两者不是非此即彼`。

![CAP-2](/images/dc/cap-2.jpg)

CAP 定理证明中的一致性指 **强一致性**，强一致性`要求多节点组成的被调要能像单节点一样运作、操作具备原子性，数据在时间、时序上都有要求`。如果放宽这些要求，还有其他一致性类型：

- **序列一致性(sequential consistency)**：不要求时序一致，A 操作先于 B 操作，在 B 操作后如果所有调用端读操作得到 A 操作的结果，满足序列一致性
- **最终一致性(eventual consistency)**：放宽对时间的要求，在被调完成操作响应后的某个时间点，被调多个节点的数据最终达成一致

可用性在 CAP 定理里指所有读写操作必须要能终止，实际应用中从主调、被调两个不同的视角，可用性具有不同的含义。当 P(网络分区) 出现时，主调可以只支持读操作，通过牺牲部分可用性达成数据一致。

工程实践中，较常见的做法是通过异步拷贝副本 (asynchronous replication)、quorum/NRW，实现在调用端看来数据强一致、被调端最终一致，在调用端看来服务可用、被调端允许部分节点不可用(或被网络分隔) 的效果。

### 跳出 CAP

CAP 理论对实现分布式系统具有指导意义，但 CAP 理论并没有涵盖分布式工程实践中的所有重要因素。

例如`延时 (latency)`，它是衡量`系统可用性`、与`用户体验`直接相关的一项重要指标。CAP 理论中的可用性要求操作能终止、不无休止地进行，除此之外，我们还关心到底需要多长时间能结束操作，这就是延时，它值得我们设计、实现分布式系统时单列出来考虑。

延时与数据一致性也是一对 “冤家”，如果要达到强一致性、多个副本数据一致，必然增加延时。加上延时的考量，我们得到一个 CAP 理论的修改版本 PACELC：如果出现 P(网络分区)，如何在 A(服务可用性)、C(数据一致性) 之间选择；否则，如何在 L(延时)、C(数据一致性)之间选择。

***

## 小结

以上介绍了 CAP 理论的源起和发展，介绍了 CAP 理论给分布式系统工程实践带来的启示。

CAP 理论对分布式系统实现有非常重大的影响，我们可以根据自身的业务特点，在数据一致性和服务可用性之间作出倾向性地选择。通过放松约束条件，我们可以实现在不同时间点满足 CAP(此 CAP 非 CAP 定理中的 CAP，如 C 替换为最终一致性)。
