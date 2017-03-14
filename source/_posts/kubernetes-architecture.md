---
layout: post
title: Kubernetes 构架设计
date: 2016-10-26 12:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---


Kubernetes 集群包含有节点代理 kubelet 和 Master 组件 (APIs, scheduler, etc)，一切都基于分布式的存储系统。下面这张图是 Kubernetes 的架构图。

![kubernetes 构架图](/images/architecture.png)

> 原文：[kubernetes architecture](https://github.com/kubernetes/kubernetes/blob/release-1.3/docs/design/architecture.md "Kubernetes 构架设计")

## Kubernetes 节点

在这张系统架构图中，我们把服务分为运行在工作节点上的服务和组成集群级别控制板的服务。

Kubernetes 节点有运行应用容器必备的服务，而这些都是受 Master 的控制。

每次个节点上当然都要运行 Docker。Docker 来负责所有具体的映像下载和容器运行。

<!-- more -->

### kubelet

kubelet 负责管理 pods 和它们上面的容器，images 镜像、volumes、etc。

### kube-proxy

每一个节点也运行一个简单的网络代理和负载均衡（详见 services FAQ )（PS: 官方 英文）。 正如 Kubernetes API 里面定义的这些服务（详见 the services doc）（PS: 官方 英文）也可以在各种终端中以轮询的方式做一些简单的 TCP 和 UDP 传输。

服务端点目前是通过 DNS 或者环境变量 (Docker-links-compatible 和 Kubernetes{FOO}_SERVICE_HOST 及 {FOO}_SERVICE_PORT 变量都支持)。这些变量由服务代理所管理的端口来解析。

## Kubernetes 控制面板

Kubernetes 控制面板可以分为多个部分。目前它们都运行在一个 master 节点，然而为了达到高可用性，这需要改变。不同部分一起协作提供一个统一的关于集群的视图。

### etcd

所有 master 的持续状态都存在 etcd 的一个实例中。这可以很好地存储配置数据。因为有 watch(观察者) 的支持，各部件协调中的改变可以很快被察觉。

### Kubernetes API Server

API 服务提供 Kubernetes API （PS: 官方 英文）的服务。这个服务试图通过把所有或者大部分的业务逻辑放到不两只的部件中从而使其具有 CRUD 特性。它主要处理 REST 操作，在 etcd 中验证更新这些对象（并最终存储）。

### Scheduler

调度器把未调度的 pod 通过 binding api 绑定到节点上。调度器是可插拔的，并且我们期待支持多集群的调度，未来甚至希望可以支持用户自定义的调度器。

### Kubernetes 控制管理服务器

所有其它的集群级别的功能目前都是由控制管理器所负责。例如，端点对象是被端点控制器来创建和更新。这些最终可以被分隔成不同的部件来让它们独自的可插拔。

replicationcontroller 是一种建立于简单的 pod API之上的一种机制。一旦实现，我们最终计划把这变成一种通用的插件机制。
