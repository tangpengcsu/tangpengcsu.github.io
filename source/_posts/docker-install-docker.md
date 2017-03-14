---
layout: post
title: Docker Engine 安装
date: 2016-10-24 12:39:04
tags: [Docker]
categories: [Docker]
---


## 参考

> Docker 官网用户指南：[https://docs.docker.com/engine/installation/linux/](https://docs.docker.com/engine/installation/linux/)

## 环境依赖

- Linux：64 位 Kernel 3.10 及以上（RHEL7 以上版本）
- Windows：64 位

<!-- more -->

## 在 Ubuntu 上安装 Docker

Docker 支持以下的 Ubuntu 版本

- Ubuntu Xenial 16.04 (LTS)
- Ubuntu Wily 15.10
- Ubuntu Trusty 14.04 (LTS)
- Ubuntu Precise 12.04 (LTS)

### 先决条件

Docker 需要在 64 位版本的 Ubuntu 上安装。此外，你还需要保证你的 Ubuntu 内核的最小版本不低于 3.10，其中 3.10 小版本和更新维护版也是可以使用的。

在低于 3.10 版本的内核上运行 Docker 会丢失一部分功能。在这些旧的版本上运行 Docker 会出现一些 BUG，这些 BUG 在一定的条件里会导致数据的丢失，或者报一些严重的错误。

打开控制台使用 uname -r 命令来查看你当前的内核版本。

```bash
$ uname -r
3.11.0-15-generic
```

Docker 要求 Ubuntu 系统的内核版本高于 3.10，查看本页面的前提条件来验证你的 Ubuntu 版本是否支持 Docker 。

### 更新 APT 源

通过下边的操作来升级你的内核和安装额外的包

1\. 登录到主机，切换到 root 用户，取得权限。

2\. 打开 Ubuntu 命令行控制台。

3\. 升级你的包管理器，确保 apt 能够使用 htpps，安装 CA 证书

```bash
$ sudo apt-get update
$ sudo apt-get install apt-transport-https ca-certificates
```

4\. 添加 Docker 公钥。

配置 Apt 来使用新仓库的第一步是想 Apt 缓存中添加该库的公钥。使用 apt-key 命令：

```bash
$ sudo apt-key adv \
               --keyserver hkp://ha.pool.sks-keyservers.net:80 \
               --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
```

以上的 apt-key 命令向密钥服务器 hkp://ha.pool.sks-keyservers.net 请求一个特定的密钥（ 58118E89F3A912897C070ADBF76221572C52609D ）。公钥将会被用来验证从新仓库下载的所有包。

5\. 依据你的 Ubuntu 发行版本，在下表中找到对应条目。这个将决定 APT 将搜索的 Docker 包。如果可能，请运行一个长期支持 (LTS) 版的 Ubuntu。

| Ubuntu version | Repository |
| ------------- | ------------- |
| Precise 12.04 (LTS) | deb https://apt.dockerproject.org/repo ubuntu-precise main |
| Trusty 14.04 (LTS) | deb https://apt.dockerproject.org/repo ubuntu-trusty main |
| Wily 15.10 | deb https://apt.dockerproject.org/repo ubuntu-wily main |
| Xenial 16.04 (LTS) | deb https://apt.dockerproject.org/repo ubuntu-xenial main |

6\. 指定 Docker 仓库的位置，根据第 5 步确定的发行版本，替换下面命令行中的 `<REPO>` 参数即可。

```bash
$ echo "<REPO>" | sudo tee /etc/apt/sources.list.d/docker.list

## 例如：
$ echo "deb https://apt.dockerproject.org/repo ubuntu-xenial" | sudo tee /etc/apt/sources.list.d/docker.list
```

> 引入 Docker 的公钥，我们可以配置 Apt 使用 Docker 的仓库服务器。新建文件：`/etc/apt/sources.list.d/docker.list`，在里面添加一个条目：`deb https://apt.dockerproject.org/repo ubuntu-xenial main`

7\. 更新 APT 软件包索引

```bash
$ sudo apt-get update
```

8\. 验证 apt 拉取正确的 repo

```bash
$ apt-cache policy docker-engine

  docker-engine:
    Installed: 1.12.2-0~trusty
    Candidate: 1.12.2-0~trusty
    Version table:
   *** 1.12.2-0~trusty 0
          500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
          100 /var/lib/dpkg/status
       1.12.1-0~trusty 0
          500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
       1.12.0-0~trusty 0
          500 https://apt.dockerproject.org/repo/ ubuntu-trusty/main amd64 Packages
```

### Ubuntu Xenial 16.04 (LTS), Wily 15.10, Trusty 14.04 (LTS) 先决条件

> 对于支持 aufs 存储驱动的 Ubuntu `Trusty`, `Wily`, and `Xenial` 安装 `linux-image-extra-* kernel` 包

1\. 打开 Ubuntu 命令行控制台

2\. 更新 apt 包缓存

```bash
$ sudo apt-get update
```

这会触发 apt 重新读取配置文件，刷新仓库列表，包含进我们添加的那个仓库。该命令也会查询这些仓库来缓存可用的包列表。

3\. 安装 linux-image-extra-* 包

在安装 Docker Engine 之前，我们需要安装一个先决软件包（ prerequisite package ）。linux-image-extra 包是一个内核相关的包，Ubuntu 系统需要它来支持 aufs 存储设备驱动。Docker 会使用该驱动来加载卷。

```bash
$ sudo apt-get install linux-image-extra-$(uname -r) linux-image-extra-virtual
```

在 apt-get 命令中，$(uname -r) 将返回正在运行的内核的版本。任何对于该系统的内核更新应当包括安装 linux-image-extra，它的版本需要与新内核版本相一致。如果该包没有正确更新的话，Docker 加载卷的功能可能受到影响。

4\. 安装 Docker

> 转到下一节 安装 Docker Engine


### Ubuntu Precise 12.04 (LTS) 先决条件

> 对于 Ubuntu Precise 发行版本， Docker 要求 Kernel 版本至少 3.13。如果你的 Kernel 版本老于 3.13，那你必须先升级内核。

1\. 打开 Ubuntu 命令行控制台

2\. 更新 apt 包缓存

```bash
$ sudo apt-get update
```

3\. 安装包

```bash
$ sudo apt-get install linux-image-generic-lts-trusty
```

4\. 重启宿主机

```bash
$ sudo reboot
```

4\. 安装 Docker

> 转到下一节 安装 Docker Engine

### 安装最新版本的 Docker Engine

1\. 以 sudo 的用户权限登录 Ubuntu。

2\. 更新 APT 包索引

```bash
$ sudo apt-get update
```

3\. 安装 Docker

```bash
$ sudo apt-get install docker-engine
```

4\. 启动 Docker 守护进程

```bash
$ sudo service docker start
```

5\. 验证 Docker 是否安装成功

```bash
$ sudo docker run hello-world
```

### 升级

```bash
$ sudo apt-get update
$ sudo apt-get upgrade docker-engine
```

### 卸载

1\. 卸载 Docker 包

```bash
$ sudo apt-get purge docker-engine
```

2\. 卸载 docker 依赖包

```bash
$ sudo apt-get autoremove --purge docker-engine
```

3\. 删除相关文件

```bash
$ rm -rf /var/lib/docker
```

## 在 RedHat 上安装 Docker

### 先决条件

Docker 需要在 64 位且内核版本不低于 3.10 的 Linux 上运行。

打开控制台使用 uname -r 命令来查看你当前的内核版本。

```bash
$ uname -r
3.10.0-229.el7.x86_64
```

### 安装 Docker Engine

如下两种安装方式：

1. Yum 安装
2. Script 安装

### Yum 安装

1\. 以 sudo 或 root 权限登录宿主机

2\. yum 更新。

```bash
$ sudo yum update
```

3\. 添加 yum 源

```bash
$ sudo tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
```

4\. 安装。

```bash
$ sudo yum install docker-engine
```

5\. 启用服务。

```bash
$ sudo systemctl enable docker.service
```

6\. 现在 Docker 已经安装好了，我来启动 Docker 守护进程。

```bash
$ sudo systemctl start docker
Redirecting to /bin/systemctl start  docker.service
```

> ps: sudo systemctl start docker.service

7\. 验证 Docker 是否安装成功

```bash
 $ sudo docker run --rm hello-world

 Unable to find image 'hello-world:latest' locally
 latest: Pulling from library/hello-world
 c04b14da8d14: Pull complete
 Digest: sha256:0256e8a36e2070f7bf2d0b0763dbabdd67798512411de4cdcf9431a1feb60fd9
 Status: Downloaded newer image for hello-world:latest

 Hello from Docker!
 This message shows that your installation appears to be working correctly.

 To generate this message, Docker took the following steps:
  1. The Docker client contacted the Docker daemon.
  2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
  3. The Docker daemon created a new container from that image which runs the
     executable that produces the output you are currently reading.
  4. The Docker daemon streamed that output to the Docker client, which sent it
     to your terminal.

 To try something more ambitious, you can run an Ubuntu container with:
  $ docker run -it ubuntu bash

 Share images, automate workflows, and more with a free Docker Hub account:
  https://hub.docker.com

 For more examples and ideas, visit:
  https://docs.docker.com/engine/userguide/
```

### 脚本安装

1\. 以 sudo 或 root 权限登录宿主机

2\. yum 更新。

```bash
$ sudo yum update
```

3\. 运行 Docker 安装脚本。

```bash
$ curl -fsSL https://get.docker.com/ | sh
```

4\. 然后启用 Docker 服务

```bash
$ sudo systemctl enable docker.service
```

5\. 运行 Docker 线程

```bash
$ sudo systemctl start docker
```

6\. 验证 Docker 是否安装成功

```bash
$ sudo docker run hello-world
```

### 卸载

1\. 列出已安装的 Docker 包

```bash
$ yum list installed | grep docker
```

2\. 删除包

```bash
$ sudo yum -y remove docker-engine.x86_64
```

3\. 删除镜像，容器，数据卷

```bash
$ rm -rf /var/lib/docker
```

4\. 删除用户数据

## Windows

### Dokcer for Windows

1. 64 位 windows10 pro。
2. 开启 Hyper-V（控制面板 \ 程序 \ 程序和功能 \ 启用或关闭 Windows 功能 \ Hyper-V）。
3. 官网下载并安装 Docker for Windows（[https://docs.docker.com/docker-for-windows/](https://docs.docker.com/docker-for-windows/)）。

### Docker Toolbox

> Windows10 pro 之前的 windows 系统。

官网下载并安装 Docker Toolbox（[https://www.docker.com/products/docker-toolbox](https://www.docker.com/products/docker-toolbox)），包括了 docker 可视化软件Docker Kitematic
