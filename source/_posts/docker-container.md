---
layout: post
title: 容器
date: 2016-10-24 12:39:04
tags: [Docker]
categories: [Docker]
---


`容器`是 Docker 又一核心概念。Docker 的容器十分`轻量级`，用户可以`随时创建或删除`容器。容器是`直接提供应用服务`的`组件`，也是已经掌握了对容器整个生命周期进行管理的各项操作命令。

简单的说，容器是独立运行的一个或一组应用，以及它们的运行态环境。而虚拟机可以理解为模拟运行的一整套操作系统（提供了运行态环境和其他系统环境）和运行在上面的应用。

下面将具体介绍如何来管理一个容器，包括`创建`、`启动`、`停止`和`删除`等。

<!-- more -->

## 新建容器

### 语法：
> docker create [-i] [-t] [-d] [IMAGE_NAME]

可以使用 docker create 命令新建一个处于停止状态的容器。然后可以通过 `docker start` 命令来启动这个容器。

## 新建并启动容器

启动容器的两种方式：

1.  基于镜像新建一个容器并启动 -- `docker run` 。
2. 重新启动已处于终止/退出状态（Stopped/Exited）的容器 -- `docker start` 。

docker run 命令等价于先执行 docker create 命令，再执行 docker start 命令。当用 docker run 命令创建并启动容器时，Docker 在后台运行的标准操作包括：

- 检查本地时候存在指定的镜像，不存在就会从 Docker Store 下载。
- 利用镜像创建并启动一个容器。
- 分配一个文件系统，并在只读的镜像层外面挂载一个可读写层。
- 从宿主机配置的网桥接口中桥接一个虚拟接口到容器中去。
- 从地址池配置一个 IP 地址给容器。
- 执行用户指定的应用程序。
- 执行完毕后容器被终止。

交互模式运行容器，例如：

```bash
$ docker run -it ubuntu:14.04 /bin/bash
root@0b2616b0e5a8:/#
```

- -t：让 Docker 分配一个伪终端（ pseudo-tty ) 并绑定到容器的标准输入上。
- -i：保持容器的标准输入处于打开状态。

用户可以通过  `Ctrl+d` 或输入 `exit` 命令来退出容器。

```bash
root@0b2616b0e5a8:/#exit
exit
```

对于所创建的 bash 容器， 当使用 `exit` 命令退出后，该容器就自动处于终止状态。这是应为对于 Docker 容器来说，当运行的应用（此例中为 bash）退出后，容器也就没有必要运行了。

## 守护态运行

很多时候，需要让 Docker 容器在后台以`守护态`形式（Daemonized）运行。用户可以通过添加 `-d` 参数来实现。

```bash
$ docker run -d ubuntu:14.04 /bin/sh -c "while true; do echo hello world;  sleep 1; done"
 cce55…
```

启动容器后会返回一个唯一的 ID。通过 `docker logs` 命令来获取容器的输出信息。

```bash
$  docker logs ce55
```

## 终止，重启容器

使用 `Docker stop` 命令终止的一个或多个处于运行态的容器。

```bash
$ docker stop cce55
```

使用 `Docker restart` 命令重启一个或多个处于运行态的容器。

```bash
$ docker restart cce55
```

## 进入容器

### 语法：

> docker exec -it [CONTAINER_ID] [COMMAND]        

使用 `-d` 参数时，容器启动后会进入后台，用户无法看到容器中的信息。如果需要进入容器进行操作，有如下两种方法：

- `docker attach` 命令：不推荐使用，容易阻塞。
- `docker exec` 命令： 是 1.3 版本提供的一个更加方便的工具。

## 删除容器

### 语法：

> docker rm [CONTAINER_ID/CONTINER_NAME] [CONTAINER_ID/CONTINER_NAME...]
