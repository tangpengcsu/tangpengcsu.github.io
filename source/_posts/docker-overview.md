---
layout: post
title: Docker 概述
date: 2016-10-24 12:39:04
tags: [Docker]
categories: [Docker]
image:
  feature: docker/docker-logo.png
---


`Docker 是一个开源的应用容器引擎`，让开发者可以打包他们的`应用`及`依赖包`到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。

容器是完全 `沙箱机制`，相互之间不会有任何接口（类似 ios app ）。几乎没有任何性能开销，可以很容易地在机器和数据中心运行。最重要的是，他们不依赖于任何语言、框架或包括系统。

Docker 自开源后受到广泛的关注和讨论，以至于 dotCloud 公司后来都改名为 Docker Inc。Redhat 已经在其 RHEL 6.5 中集中支持 Docker，也在其 PaaS 产品中广泛应用。

Docker 项目的目标是实现轻量级的操作系统虚拟化解决方案。Docker 的基础是 Linux 容器（`LXC`）等技术。

在 LXC 的基础上 Docker 进一步封装，让用户不需要去关心容器的管理，使得操作更加简便，用户操作 Docker 的容器就像一个快速轻量级的虚拟机一样简单。

下面对比 Docker 和传统虚拟化（KVM、XEN 等）方式的不同之处，`容器` 是在操作系统层面上实现的虚拟化，直接复用本地主机的操作系统；而 `传统方式` 则是在硬件基础上，虚拟出自己的系统，再在系统上部署相关的 APP 应用。

下图为传统虚拟化方案与 Docker 虚拟化方案对比：

![传统虚拟化方案与 Docker 虚拟化方案对比图](/images/docker/docker-overview.png)

<!-- more -->

***

## 三个概念

1. **镜像**：Docker 的镜像其实就是模版，跟我们常见的 ISO 镜像类似，是一个样板。
2. **容器**：使用镜像构建常见的应用或者系统，我们称之为一个容器。
3. **仓库**：仓库是存放镜像的地方，分为公共仓库(Publish)和私有仓库（Private）两种形式。

***

## Docker 虚拟化特点

跟传统 VM 比较有如下特点：

1. **操作启动快**：运行时的性能可以获得极大提升，管理操作（启动，停止，开始，重启等）都是秒或毫秒为单位的。
2. **轻量级虚拟化**：你会拥有足够的 "操作系统"，仅需添加镜像即可。在一台服务器上部署 100~1000 个 Container 容器，但是传统虚拟化，虚拟 10~20 个虚拟机就不错了。
3. **开源免费**：开源的，免费的，低成本的。有现代 Linux 内核支持并驱动（ps：轻量级的 Container 必定可以在一个物理机上开启更多 "容器"）。
4. **前景及云支持**：越来越受欢迎，包括各大主流公司都在推动 Docker 的快速发展，性能上有很大的优势。

***

## 快速跳转

- [Docker 概述](/2016/10/24/docker-overview/ "/2016/10/24/docker-overview/")
- [Docker Engine 安装](/2016/10/24/docker-install-docker/ "/2016/10/24/docker-install-docker/")
- [镜像](/2016/10/24/docker-image/ "/2016/10/24/docker-image/")
- [容器](/2016/10/24/docker-container/ "/2016/10/24/docker-container/")
- [仓库](/2016/10/24/docker-registry/ "/2016/10/24/docker-registry/")
- [容器数据管理](/2016/10/24/docker-data-volume/ "/2016/10/24/docker-data-volume/")
- [使用网络](/2016/10/24/docker-container-networking/ "/2016/10/24/docker-container-networking/")
- [Dockerfile](/2016/10/24/docker-dockerfile-reference/ "/2016/10/24/docker-dockerfile-reference/")
- [高级网络配置](/2016/10/24/docker-network-containers/ "/2016/10/24/docker-network-containers/")
- [存储驱动](/2016/10/24/docker-storage-drivers/ "/2016/10/24/docker-storage-drivers/")
