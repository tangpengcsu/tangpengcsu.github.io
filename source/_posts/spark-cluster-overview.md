---
layout: post
title: Spark 集群模式概述
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


该文档给出了 Spark 如何在集群上运行、使之更容易来理解所涉及到的组件的简短概述。通过阅读 [应用提交指南](/2016/12/24/spark-submitting-applications/ "/2016/12/24/spark-submitting-applications/") 来学习关于在集群上启动应用。

<!-- more -->

***

## 组件

Spark 应用在集群上作为独立的进程组来运行，在您的 main 程序中通过 SparkContext 来协调（称之为 driver 程序）。

具体的说，为了运行在集群上，SparkContext 可以连接至几种类型的 Cluster Manager（既可以用 Spark 自己的 Standlone Cluster Manager，或者 Mesos，也可以使用 YARN），它们会分配应用的资源。一旦连接上，Spark 获得集群中节点上的 Executor，这些进程可以运行计算并且为您的应用存储数据。接下来，它将发送您的应用代码（通过 JAR 或者 Python 文件定义传递给 SparkContext）至 Executor。最终，SparkContext 将发送 Task 到 Executor 运行。

![Spark cluster components](/images/cluster-overview.png "Spark cluster components")

这里有几个关于这个架构需要注意的地方 :

1. 每个应用获取到它自己的 Executor 进程，它们会保持在整个应用的生命周期中并且在多个线程中运行 Task（任务）。这样做的优点是把应用互相隔离，在调度方面（每个 driver 调度它自己的 task）和 Executor 方面（来自不同应用的 task 运行在不同的 JVM 中）。然而，这也意味着若是不把数据写到外部的存储系统中的话，数据就不能够被不同的 Spark 应用（SparkContext 的实例）之间共享。
2. Spark 是不知道底层的 Cluster Manager 到底是什么类型的。只要它能够获得 Executor 进程，并且它们可以和彼此之间通信，那么即使是在一个也支持其它应用的 Cluster Manager（例如，Mesos / YARN）上来运行它也是相对简单的。
3. Driver 程序必须在自己的生命周期内（例如，请看 在网络中配置 [spark.driver.port](/2016/12/24/spark-configuration/#networking "/2016/12/24/spark-configuration/#networking") 章节）监听和接受来自它的 Executor 的连接请求。同样的，driver 程序必须可以从 worker 节点上网络寻址（即网络没问题）。
4. 因为 driver 调度了集群上的 task（任务），更好的方式应该是在相同的局域网中靠近 worker 的节点上运行。如果您不喜欢发送请求到远程的集群，倒不如打开一个 RPC 至 driver 并让它就近提交操作而不是从很远的 worker 节点上运行一个 driver。

***

## Cluster Manager 类型

该系统当前支持三种 Cluster Manager :

- [Standalone](/2016/12/24/spark-standalone/ "/2016/12/24/spark-standalone/") – 包含在 Spark 中使得它更容易来安装集群的一个简单的 Cluster Manager。
- [Apache Mesos](/2016/12/24/spark-running-on-mesos/ "/2016/12/24/spark-running-on-mesos/") – 一个通用的 Cluster Manager，它也可以运行 Hadoop MapReduce 和其它服务应用。
- [Hadoop YARN](/2016/12/24/spark-running-on-yarn/ "/2016/12/24/spark-running-on-yarn/") – Hadoop 2 中的 resource manager（资源管理器）。

***

## 提交应用

使用 spark-submit 脚本可以提交应用至任何类型的集群。在 [应用提交指南](/2016/12/24/spark-submitting-applications/ "/2016/12/24/spark-submitting-applications/") 中介绍了如何来做到这一点。

***

## 监控

每个 driver 程序有一个 Web UI，通常在端口 4040 上，它展示了关于运行 task，executor，和存储使用情况的信息。在网页浏览器中访问这个 UI : http://\<driver-node\>:4040。 [监控指南](/2016/12/24/spark-monitoring/ "/2016/12/24/spark-monitoring/") 也描述了其它的监控选项。

## Job 调度

Spark 即可以在应用间（Cluster Manager 级别），也可以在应用内（如果多个计算发生在相同的 SparkContext 上时）控制资源分配。[Job 调度指南](/2016/12/24/spark-job-scheduling/ "/2016/12/24/spark-job-scheduling/") 描述的更详细。

***

## 术语

下表中总结了您将会看到用于涉及到集群时的术语 :

| Term | Meaning |
| ------------- | ------------- |
| Application | 用户构建在 Spark 上的程序。由集群上的一个 driver 程序和多个 executor 组成。 |
| Application jar | 一个包含用户 Spark 应用的 Jar。有时候用户会想要去创建一个包含他们应用以及它的依赖的 "uber jar"。用户的 Jar 应该没有包括 Hadoop 或者 Spark 库，然而，它们将会在运行时被添加。 |
| Driver program  | 该进程运行应用的 main() 方法并且创建了 SparkContext。 |
| Cluster manager | 一个外部的用于获取集群上资源的服务。（例如，Standlone Manager，Mesos，YARN） |
| Deploy mode | 根据 driver 程序运行的地方区别。在 "Cluster" 模式中，框架在群集内部启动 driver。在 "Client" 模式中，submitter（提交者）在 Custer 外部启动 driver。 |
| Worker node | 任何在集群中可以运行应用代码的节点。 |
| Executor | 一个为了在 worker 节点上的应用而启动的进程，它运行 task 并且将数据保持在内存中或者硬盘存储。每个应用有它自己的 Executor。 |
| Task | 一个将要被发送到 Executor 中的工作单元。 |
| Job | 一个由多个 task 组成的并行计算，并且能从 Spark action 中获取响应（例如，save，collect），您将在 driver 的日志中看到这个术语。 |
| Stage | 每个 Job 被拆分成更小的被称作 stage（阶段） 的 task（任务） 组，stage 彼此之间是相互依赖的（与 MapReduce 中的 map 和 reduce stage 相似）。您将在 driver 的日志中看到这个术语。 |
