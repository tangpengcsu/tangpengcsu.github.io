---
layout: post
title: Docker Swarm Mode
date: 2016-10-24 12:39:04
tags: [Docker]
categories: [Docker]
---


## Docker Swarm Mode特性

1. 集群管理与 Docker 引擎相结合：使用 Docker 引擎 CLI 便可创建一个 Docker 引擎的 Swarm，在这个集群中进行应用服务的部署。对于Swarm 集群的创建和管理，无需其他额外的编排软件。
2. 分散设计：Docker 引擎不是在部署时处理节点角色之间的差异化内容，而是在运行时处理特殊化内容。通过使用 Docker 引擎可以部署管理节点和工作节点。这就意味着你可从一个单一的磁盘映像上创建一个完整的 Swarm 集群。
3. 支持面向服务的组件模型：Docker 引擎运用描述性方式，让用户在应用堆栈上定义各种服务的理想状态。例如，可以这样描述一个应用：包含一个配有信息排队服务的 Web 前端服务和一个数据库后端。
4. 弹性放缩：对于每项服务，你都可以明确想要运行的任务数。当你增加或减少任务数时，Swarm 管理器可自动增加或删除任务，以保持理想状态。
5. 理想状态的调整：Swarm 管理节点持续对集群状态进行监控，并对实际状态和你明确的理想状态之间的差异进行调整。例如，当你创建一个服务来运行一个容器的10个副本，托管其中两个副本的工作机崩溃时，管理器将创建两个新的副本，代替之前崩溃的两个。Swarm管理器会将新的副本分配给正在运行且可用的工作机。
6. 多主机网络：你可以针对你的服务指定一个的 Overlay 网络。Swarm管理器初始化或更新应用时，它会自动将地址分配给 Overlay 网上的容器。
7. 服务发现：Swarm管理节点给Swarm集群上的每项服务分配一个唯一的 DNS 名称以及负载均衡的运行容器。可通过嵌入 Swarm 的 DNS 服务器对集群上运行的各个容器进行查询。
8. 负载均衡：可以把服务端口暴露给外部负载均衡器。Swarm 内部允许你明确如何分配节点间的服务容器。
9. 默认安全：Swarm 上的各个节点强制 TLS 互相授权和加密，从而确保自身与其他所有节点之间的通讯安全。可选择使用自签的根证书或自定义根 CA 证书。
10. 滚动升级：升级时，可以逐增地将服务更新应用到节点上。Swarm 管理器允许你对部署服务到不同节点集而产生的时滞进行控制。如果出错，可以回退任务到服务的前一个版本。

<!-- more -->

## 关键概念

下面介绍了一些 Docker 引擎1.12集群管理及编排特点的专门概念。

### Swarm

Docker 引擎内置的集群管理和编排功能是利用 SwarmKit 工具进行开发。加入集群的引擎在 Swarm 模式下运行。可通过初始化 Swarm 集群或加入现有的 Swarm 集群启动引擎的 Swarm 模式。

Swarm 集群指的是一个可以部署服务的Docker引擎集群。Docker引擎 CLI 包含 Swarm 集群管理指令，如增加或删除节点。CLI 也包含将服务部署到 Swarm 集群以及对服务编排进行管理的指令。

在 Swarm 模式外运行 Docker 引擎时，需执行容器命令。在 Swarm模式下运行引擎时，则是对服务进行编排。

### 节点

节点指加入Swarm集群的Docker引擎的一个实例。

为将应用部署到Swarm中，你需要向管理节点提交一个服务定义。管理节点会将被称为任务的工作单元分配给工作节点。

同时，管理节点也执行编排和集群管理功能，以维持Swarm集群的理想状态。管理节点选择一个单一的引导段来执行编排任务。

工作节点接收并执行管理节点分配的任务。默认管理节点也作为工作节点，但可将其设置为仅为管理节点。代理将所分配的任务的当前状态通知管理节点，以便管理节点能够维持理想状态。


### 服务与任务
服务指的是在工作节点上执行的任务。服务是Swarm集群系统的中心结构，也是用户与Swarm互动的主根。

当创建一项服务时，需明确使用哪个容器映像以及在运行的容器中执行何种指令。

在复制型服务模型下，Swarm管理器依据在理想状态下所设定的规模将一定数目的副本任务在节点中进行分配。

对于global服务，Swarm在集群中各个可用的节点上运行任务。

一项任务承载一个Docker容器以及在容器内运行的指令。任务是Swarm集群的最小调度单元。管理节点按照在服务规模中所设定的副本数量将任务分配给工作节点。任务一旦被分配到一个节点，便不可能转移到另一个节点。它只能在被分配的节点上运行或失效。

### 负载均衡

Swarm 管理器使用入口负载均衡来暴露服务，使这些服务对 Swarm 外部可用。Swarm 管理器能自动给服务分配一个 PublishedPort，或你可为服务在30000-32767范围内设置一个 PublishedPort。

外部组件，例如云负载均衡器，能够在集群中任一节点的PublishedPort 上访问服务，无论节点是否正在执行任务。Swarm 中的所有节点会将入口连接按路径发送到一个正在运行任务的实例中。

Swarm 模式有一个内部的 DNS 组件，可自动给集群中的每项服务分配一个 DNS 入口。Swarm 管理器依据服务的 DNS 名称，运用内部负载均衡在集群内的服务之间进行请求的分配。

## 先决条件

1\. 开放端口：

- TCP port 2377 for cluster management communications（集群管理）
- TCP and UDP port 7946 for communication among nodes（节点通信）
- TCP and UDP port 4789 for overlay network traffic（overlay 网络流量）

2\. 节点类型：

- 管理节点：负责执行维护 Swarm 必要状态所必需的编排与集群管理功能。管理节点会选择单一主管理方执行编排任务。Swarm Mode 要求利用奇数台管理节点以维持容错能力。
- 工作节点：负责从管理节点处接收并执行各项任务。

>在默认情况下，管理节点本身同时也作为工作节点存在，但大家可以通过配置保证其仅执行管理任务。

3\. 查看端口开放信息

```bash
$ iptables -L
$ netstat -nulp  (UDP类型的端口)
$ netstat -ntlp  (TCP类型的端口)
```

4\. 查看防火墙信息

```bash
$ systemctl status iptables
```

> RHEL 7 系列 iptables 已经 firewalld 取代。

## Swarm 构建

1\. 当创建 Swarm 集群时，我们需要指定一个节点。在这个例子中，我们会使用主机名为manager0的主机作为 manager 节点。为了使 manager01 成为 manager 节点，我们需要首先在 manager0 执行命令来创建 Swarm 集群。这个命令就是 Docker 命令的 swarm init 选项。

```bash
$ docker swarm init --advertise-addr <MANAGER-IP>
```

- 上述命令中，除了 swarm init 之外，我们还指定了 --advertise-addr为<MANAGER-IP>。Swarm manager  节点会使用该 IP 地址来广告 Swarm 集群服务。虽然该地址可以是私有地址，重要的是，为了使节点加入该集群，那些节点需要能通过该 IP 的2377端口来访问 manager 节点。
- 在运行  docker swarm init 命令之后，我们可以看到 manager0 被赋予了一个节点名字（ awwiap1z5vtxponawdqndl0e7 ），并被选为 Swarm 集群的管理器。输出中也提供了两个命令：一个命令可以添加 worker 节点到 swarm 中，另一个命令可以添加另一个 manager 节点到该 Swarm 中。
- Docker Swarm Mode 可以支持多个manager 节点。然而，其中的一个会被选举为主节点服务器，它会负责 Swarm 的编排。

2\. 根据 docker info 信息，查看 swarm 状态（ Active/Inactive ）。

3\. 添加 worker节点到Swarm 集群中。

- Swarm 集群建立之后，我们需要添加一个新的 worker 节点到集群中。在 manger0 节点执行如下指令。

```bash
$ docker swarm join-token worker
```

- 根据上述命令输出，复制粘贴到 worker 节点并运行。Swarm 集群中的 worker 节点的角色是用来运行任务（ tasks ）的；这里所说的任务（ tasks ）就是容器（ containers ）。另一方面，manager 节点的角色是管理任务（容器）的编排，并维护 Swarm 集群本身。

4\. 添加 manager 节点。在 manger0 节点执行如下指令，根据提示添加 manager 节点。

```bash
$ docker swarm join-token manager
```

## 节点管理

1\. 查看当前 Swarm 节点。我们可以执行 Docker 命令的 node ls 选项来验证集群的状态。

```bash
$ docker node ls
```

2\. 查看节点信息

```bash
$ docker node inspect <NODE-ID> [--pretty]
```

3\. 提升节点

```bash
$ docker node promote  <HOSTNAME>
```

4\. 降级节点

```bash
$ docker node demote <HOSTNAME>
```

5\. 删除节点

```bash
$ docker node rm <NODE_ID>
```

6\. 脱离docker swarm

```bash
$ docker swarm leave [--force]
```

## 服务部署

在 Docker Swarm Mode中，服务是指一个长期运行（ long-running ）的 Docker 容器，它可以被部署到任意一台 worker 节点上，可以被远端系统或者 Swarm 中其他容器连接和消费（ consume ）的。

1\. 创建一个服务

```bash
$ docker service create <IMAGE ID>
```

例如：

```bash
$ docker service create --name nginx --replicas 2 --publish 8080:80 nginx
        一个有副本的服务是一个 Docker Swarm 服务，运行了特定数目的副本（ --replicas ）。这些副本是由多个 Docker 容器的实例组成的。
```

2\. Scale

```bash
$ docker service scale <SERVICE-ID>=<NUMBER-OF-TASKS>
```

3\. 查看所有服务

```bash
$ docker service ls
```

4\. 查看单个服务部署详情

```bash
$ docker service ps <service_id|service_name>
```

5\. 查看服务信息

```bash
$ docker service inspect <service_id|service_name>
```

6\. 删除服务

```bash
$ docker service rm <service_id>
```

## 服务发布

当我们创建了 nginx 服务时，我们使用了 --publish 选项。该选项用来告知 Docker 将端口8080发布为 nginx 服务的可用端口。

当 Docker 发布了服务端口时，它在 Swarm 集群上的所有节点上监听该端口。当流量到达该端口时，该流量将被路由到运行该服务的容器上。如果所有节点都运行着一个服务的容器，那么概念是相对标准的；然而，当我们的节点数比副本多时，概念就变得有趣了。

## 服务global化

假设 Swarm 集群中有三个节点，此时，我们已经建立了 nginx 服务，运行了2个副本，这意味着，3个节点中的2个正在运行容器。

如果我们希望 nginx 服务在每一个 worker 节点上运行一个实例，我们可以简单地修改服务的副本数目，从2增加到3。这意味着，如果我们增加或者减少 worker 节点数目，我们需要调整副本数目。

我们可以自动化地做这件事，只要把我们的服务变成一个 Global Service。Docker Swarm Mode 中的 Global Service 使用了创建一个服务，该服务会自动地在每个 worker 节点上运行任务。这种方法对于像 nginx 这样的一般服务都是有效的。

让我们重新创建 redis 服务为 Global Service。

```bash
$ docker service create --name nginx --mode global --publish 8080:80 nginx
```

同样是 docker service create 命令，唯一的区别是指定了 --mode 参数为 global。
服务建立好之后，运行 docker service ps nginx 命令的，我们可以看到，Docker 是如何分发该服务的。
