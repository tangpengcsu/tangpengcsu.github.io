---
layout: post
title: 分布式系统理论基础 - 时间、时钟和事件顺序
date: 2017-1-17 12:39:04
tags: [Distributed Computing]
categories: [Distributed Computing]
---

> 十六号…… 四月十六号。一九六零年四月十六号下午三点之前的一分钟你和我在一起，因为你我会记住这一分钟。从现在开始我们就是一分钟的朋友，这是事实，你改变不了，因为已经过去了。我明天会再来。    -- 《阿飞正传》

现实生活中时间是很重要的概念，时间可以记录事情发生的时刻、比较事情发生的先后顺序。分布式系统的一些场景也需要记录和比较不同节点间事件发生的顺序，但不同于日常生活使用物理时钟记录时间，分布式系统使用逻辑时钟记录事件顺序关系，下面我们来看分布式系统中几种常见的逻辑时钟。

***

<!-- more -->

## 物理时钟 vs 逻辑时钟

可能有人会问，为什么分布式系统不使用物理时钟 (physical clock) 记录事件？每个事件对应打上一个时间戳，当需要比较顺序的时候比较相应时间戳就好了。

这是因为现实生活中物理时间有统一的标准，而分布式系统中每个节点记录的时间并不一样，即使设置了 NTP 时间同步节点间也存在毫秒级别的偏差。因而分布式系统需要有另外的方法记录事件顺序关系，这就是逻辑时钟 (logical clock)。

![time-vs](/images/dc/time-vs.jpg "time-vs")

## Lamport timestamps

Leslie Lamport 在 1978 年提出逻辑时钟的概念，并描述了一种逻辑时钟的表示方法，这个方法被称为 Lamport 时间戳 (Lamport timestamps)。

分布式系统中按是否存在节点交互可分为三类事件，一类发生于节点内部，二是发送事件，三是接收事件。Lamport 时间戳原理如下：

![lamport-timestamps](/images/dc/lamport-timestamps.png "lamport-timestamps")

1. 每个事件对应一个 Lamport 时间戳，初始值为 0
2. 如果事件在节点内发生，时间戳加 1
3. 如果事件属于发送事件，时间戳加 1 并在消息中带上该时间戳
4. 如果事件属于接收事件，时间戳 = Max(本地时间戳，消息中的时间戳) + 1

假设有事件 a、b，C(a)、C(b) 分别表示事件 a、b 对应的 Lamport 时间戳，如果 C(a) < C(b)，则有 a 发生在 b 之前 (happened before)，记作 a -> b，例如图 1 中有 C1 -> B1。通过该定义，事件集中 Lamport 时间戳不等的事件可进行比较，我们获得事件的偏序关系 (partial order)。

如果 C(a) = C(b)，那 a、b 事件的顺序又是怎样的？假设 a、b 分别在节点 P、Q 上发生，Pi、Qj 分别表示我们给 P、Q 的编号，如果 C(a) = C(b) 并且 Pi < Qj，同样定义为 a 发生在 b 之前，记作 a => b。假如我们对图 1 的 A、B、C 分别编号 Ai = 1、Bj = 2、Ck = 3，因 C(B4) = C(C3) 并且 Bj < Ck，则 B4 => C3。

通过以上定义，我们可以对所有事件排序、获得事件的全序关系 (total order)。上图例子，我们可以从 C1 到 A4 进行排序。

***

## Vector clock

Lamport 时间戳帮助我们得到事件顺序关系，但还有一种顺序关系不能用 Lamport 时间戳很好地表示出来，那就是同时发生关系 (concurrent)。例如图 1 中事件 B4 和事件 C3 没有因果关系，属于同时发生事件，但 Lamport 时间戳定义两者有先后顺序。

Vector clock 是在 Lamport 时间戳基础上演进的另一种逻辑时钟方法，它通过 vector 结构不但记录本节点的 Lamport 时间戳，同时也记录了其他节点的 Lamport 时间戳。Vector clock 的原理与 Lamport 时间戳类似，使用图例如下：

![vector-clock](/images/dc/vector-clock.png "vector-clock")

假设有事件 a、b 分别在节点 P、Q 上发生，Vector clock 分别为 Ta、Tb，如果 Tb[Q] > Ta[Q] 并且 Tb[P] >= Ta[P]，则 a 发生于 b 之前，记作 a -> b。到目前为止还和 Lamport 时间戳差别不大，那 Vector clock 怎么判别同时发生关系呢？

如果 Tb[Q] > Ta[Q] 并且 Tb[P] < Ta[P]，则认为 a、b 同时发生，记作 a <-> b。例如图 2 中节点 B 上的第 4 个事件 (A:2，B:4，C:1) 与节点 C 上的第 2 个事件 (B:3，C:2) 没有因果关系、属于同时发生事件。

***

## Version vector

基于 Vector clock 我们可以获得任意两个事件的顺序关系，结果或为先后顺序或为同时发生，识别事件顺序在工程实践中有很重要的引申应用，最常见的应用是发现数据冲突 (detect conflict)。

分布式系统中数据一般存在多个副本 (replication)，多个副本可能被同时更新，这会引起副本间数据不一致 [7]，Version vector 的实现与 Vector clock 非常类似 [8]，目的用于发现数据冲突 [9]。下面通过一个例子说明 Version vector 的用法 [10]：

![vector-vector](/images/dc/vector-vector.png "vector-vector")

- client 端写入数据，该请求被 Sx 处理并创建相应的 vector ([Sx, 1])，记为数据 D1
- 第 2 次请求也被 Sx 处理，数据修改为 D2，vector 修改为 ([Sx, 2])
- 第 3、第 4 次请求分别被 Sy、Sz 处理，client 端先读取到 D2，然后 D3、D4 被写入 Sy、Sz
- 第 5 次更新时 client 端读取到 D2、D3 和 D4 3 个数据版本，通过类似 Vector clock 判断同时发生关系的方法可判断 D3、D4 存在数据冲突，最终通过一定方法解决数据冲突并写入 D5

Vector clock 只用于发现数据冲突，不能解决数据冲突。如何解决数据冲突因场景而异，具体方法有以最后更新为准 (last write win)，或将冲突的数据交给 client 由 client 端决定如何处理，或通过 quorum 决议事先避免数据冲突的情况发生。

由于记录了所有数据在所有节点上的逻辑时钟信息，Vector clock 和 Version vector 在实际应用中可能面临的一个问题是 vector 过大，用于数据管理的元数据 (meta data) 甚至大于数据本身。

解决该问题的方法是使用 server id 取代 client id 创建 vector (因为 server 的数量相对 client 稳定)，或设定最大的 size、如果超过该 size 值则淘汰最旧的 vector 信息。

***

## 小结

以上介绍了分布式系统里逻辑时钟的表示方法，通过Lamport timestamps可以建立事件的全序关系，通过Vector clock可以比较任意两个事件的顺序关系并且能表示无因果关系的事件，将Vector clock的方法用于发现数据版本冲突，于是有了Version vector。
