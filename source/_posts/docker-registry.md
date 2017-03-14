---
layout: post
title: 仓库
date: 2016-10-24 12:39:04
tags: [Docker]
categories: [Docker]
---

`仓库（Repository）`是集中 *存放镜像* 的地方。仓库概念的引入，为 Docker 镜像文件的分发和管理提供了便捷的途径。

一个容易混淆的概念是`注册服务器（Registry）`。实际上注册服务器是管理仓库的具体服务器，每个服务器上可以有多个仓库，而每个仓库下面有多个镜像。从这方面来说，仓库可以被认为是一个具体的项目或目录。例如对于仓库地址 store.docker.com/ubuntu 来说，store.docker.com 是注册服务器地址，ubuntu 是仓库名。大部分时候，并 *不需要严格区分这两者的概念* 。

仓库分为`公有仓库`和`私有仓库`。

<!-- more -->

**公有仓库地址：**

1. Docker Store 官方镜像库: [https://store.docker.com/](https://store.docker.com/ "https://store.docker.com/")
2. 阿里云镜像库： [https://dev.aliyun.com/search.html](https://dev.aliyun.com/search.html "https://dev.aliyun.com/search.html")
3. DaoClound 镜像库： [https://hub.daocloud.io/](https://hub.daocloud.io/ "https://hub.daocloud.io/")

**私有仓库** 搭建与使用见： [Harbor 私有仓库部署]({{ site.url }}/harbor/harbor-configure-https-v05.0-rc2/ "{{ site.url }}/harbor/harbor-configure-https-v05.0-rc2/")章节。
