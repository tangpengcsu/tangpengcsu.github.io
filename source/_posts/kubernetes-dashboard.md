---
layout: post
title: Kubernetes Dashboard
date: 2016-10-26 12:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---


> 原文引用：[https://github.com/kubernetes/dashboard#kubernetes-dashboard](https://github.com/kubernetes/dashboard#kubernetes-dashboard "https://github.com/kubernetes/dashboard#kubernetes-dashboard")

Kubernetes Dashboard 是一个通用的 Kubernetes 集群 web UI，它允许用户管理在集群中运行的应用，同时也能够管理集群本身。

![dashboard-ui](/images/kubernetes/dashboard-ui.png "/images/kubernetes/dashboard-ui.png")

## 部署

通过如下指令检查集群中是否已安装 Dashboard：

```bash
$ kubectl get pods --all-namespaces | grep dashboard
```

如果没有安装，通过以下指令安装最新的稳定版本：

```bash
$ kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
```

你也可安装最新的非稳定版本--[development guide](https://github.com/kubernetes/dashboard/blob/master/docs/devel/head-releases.md "https://github.com/kubernetes/dashboard/blob/master/docs/devel/head-releases.md")

```bash
kubectl create -f https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard-head.yaml
```

> 注意：为了使数据和图表可用，你需要在集群中部署 [heapster](/2016/10/26/kubernetes-heapster/ "/2016/10/26/kubernetes-heapster/")。

<!-- more -->

## 使用

 来访问 Dashboard 最简单的方法是使用 `kubectl`。运行如下指令：

```bash
$ kubectl proxy
```

 `kubectl` 将会处理 `apiserver` 认证并且使 Dashboard 能够通过 `http://localhost:8001/ui` 访问。

  UI 只能在执行命令的的机器上访问。通过 `kubectl proxy --help` 指令查看更多。
