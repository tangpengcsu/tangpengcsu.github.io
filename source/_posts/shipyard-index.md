---
layout: post
title: Docker 集中化 Web 界面管理平台 shipyard
date: 2016-10-25 18:39:04
tags: [Shipyard]
categories: [Shipyard]
---


## 简介

shipyard 是一个集成管理 Docker 容器、镜像、仓库的系统，他最大亮点应该是支持多节点的集成管理，可以动态加载节点，可托管 node 下的容器。

<!-- more -->

***

## 首次部署脚本

```bash
$ curl -sSL https://shipyard-project.com/deploy | bash -s
```

ACTION: 可以使用的指令 (deploy, upgrade, node, remove)

- DISCOVERY: 集群系统采用 Swarm 进行采集和管理 (在节点管理中可以使用‘node’)
- IMAGE: 镜像，默认使用 shipyard 的镜像
- PREFIX: 容器名字的前缀
- SHIPYARD_ARGS: 容器的常用参数
- TLS_CERT_PATH: TLS 证书路径
- PORT: 主程序监听端口 (默认端口: 8080)
- PROXY_PORT: 代理端口 (默认: 2375)

***

## 脚本可选项

如果你要自定义部署，请参考以下规范

- 部署 action：指令有效变量
- deploy: 部署新的 shipyard 实例
- upgrade: 更新已存在的实例（注意：你要保持相同的系统环境、变量来部署同样的配置）
- node: 使用 Swarm 增加一个新的 node
- remove: 删除已存在的 shipyard 实例（容器）

***

## 镜像使用

你可以采取规范的镜像来部署实例，比如以下的测试版本，你也已这样做

```bash
$ curl -sSL https://shipyard-project.com/deploy | IMAGE=shipyard/shipyard:test bash -s
```

***

## 前缀使用

你可以定义你想要的前缀，比如

```bash
$ curl -sSL https://shipyard-project.com/deploy | PREFIX=shipyard-test bash -s
```

***

## 参数使用

这里增加一些 shipyard 运行参数，你可以像这样进行调整：

```bash
$ curl -sSL https://shipyard-project.com/deploy | SHIPYARD_ARGS="--ldap-server=ldap.example.com --ldap-autocreate-users" bash -s
```

***

## TLS 证书使用

1\. 启用 TLS 对组建进行部署，包括代理（proxy）、swarm 集群系统、shipyard 管理平台的配置，这是一个配置规范。证书必须采用以下命名规范：
    - ca.pem: 安全认证证书
    - server.pem: 服务器证书
    - server-key.pem: 服务器私有证书
    - cert.pem: 客户端证书
    - key.pem: 客户端证书的 key
2\. 注意：证书将被放置在一个 docker 容器中，并在各个组成部分之间共享。如果需要调试，可以将此容器连接到调试容器。数据容器名称为前缀的证书。

```bash
$ docker run --rm \
    -v $(pwd)/certs:/certs \
    ehazlett/certm \
    -d /certs \
    bundle \
    generate \
    -o shipyard \
    --host proxy \
    --host 127.0.0.1
```

3\. 你也可以按如下指令来部署 shipyard   

```bash
$ curl -sSL https://shipyard-project.com/deploy | TLS_CERT_PATH=$(pwd)/certs bash -s
```

***

## 增加一个部署节点

shipyard 节点部署脚本将自动的安装 key/value 存储系统（etcd 系统）。增加一个节点到 swarm 集群，你可以使用以下的节点部署脚本

```bash
$ curl -sSL https://shipyard-project.com/deploy | ACTION=node  DISCOVERY=etcd://10.0.1.10:4001 bash -s
```

> 10.0.1.10 这个 ip 地址你需要修改为你的首次初始化 shipyard 系统的主机地址

***

## 删除 shipyard 系统

```bash
$ curl -sSL https://shipyard-project.com/deploy | ACTION=remove bash -s
```

***

## 实例

1\. 在 manager0 节点上执行部署脚本

```bash
$ curl -sSL https://shipyard-project.com/deploy | bash -s
```

```bash
$ docker ps    
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                                            NAMES
5636aef7bec1        shipyard/shipyard:latest       "/bin/controller --de"   4 minutes ago       Up 4 minutes        0.0.0.0:8080->8080/tcp                           shipyard-controller
8a0f609b05f1        swarm:latest                   "/swarm j --addr 192."   4 minutes ago       Up 4 minutes        2375/tcp                                         shipyard-swarm-agent
916aaa6a59ba        swarm:latest                   "/swarm m --replicati"   4 minutes ago       Up 4 minutes        2375/tcp                                         shipyard-swarm-manager
20619c435f9f        shipyard/docker-proxy:latest   "/usr/local/bin/run"     4 minutes ago       Up 4 minutes        0.0.0.0:2375->2375/tcp                           shipyard-proxy
98e7f61e436f        alpine                         "sh"                     4 minutes ago       Up 4 minutes                                                         shipyard-certs
b09af2fc0b92        microbox/etcd:latest           "/bin/etcd -addr 192."   5 minutes ago       Up 4 minutes        0.0.0.0:4001->4001/tcp, 0.0.0.0:7001->7001/tcp   shipyard-discovery
3d294a262ea0        rethinkdb                      "rethinkdb --bind all"   5 minutes ago       Up 4 minutes        8080/tcp, 28015/tcp, 29015/tcp                   shipyard-rethinkdb
```

2\. 部署节点 node3

```bash
$ curl -sSL https://shipyard-project.com/deploy | ACTION=node PREFIX=shipyard-node3  DISCOVERY=etcd://192.168.93.145:4001 bash –s
```

```bash
$ docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                    NAMES
195acdbe1818        swarm:latest                   "/swarm j --addr 192."   2 minutes ago       Up 2 minutes        2375/tcp                 shipyard-node3-swarm-agent
3e43f1c85af0        swarm:latest                   "/swarm m --replicati"   2 minutes ago       Up 2 minutes        2375/tcp                 shipyard-node3-swarm-manager
5a38d292f81e        shipyard/docker-proxy:latest   "/usr/local/bin/run"     3 minutes ago       Up 3 minutes        0.0.0.0:2375->2375/tcp   shipyard-node3-proxy
f8d80de583e6        alpine                         "sh"                     3 minutes ago       Up 3 minutes                                 shipyard-node3-certs
```

3\. 部署节点 node4

```bash
$ curl -sSL https://shipyard-project.com/deploy | ACTION=node PREFIX=shipyard-node4  DISCOVERY=etcd://192.168.93.145:4001 bash –s
```

```bash
$ docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                    NAMES
b9a4f9f6d4c5        swarm:latest                   "/swarm j --addr 192."   3 minutes ago       Up 3 minutes        2375/tcp                 shipyard-node4-swarm-agent
765f3898c925        swarm:latest                   "/swarm m --replicati"   3 minutes ago       Up 3 minutes        2375/tcp                 shipyard-node4-swarm-manager
16e000eae193        shipyard/docker-proxy:latest   "/usr/local/bin/run"     3 minutes ago       Up 3 minutes        0.0.0.0:2375->2375/tcp   shipyard-node4-proxy
ae224eafce6a        alpine                         "sh"                     3 minutes ago       Up 3 minutes                                 shipyard-node4-certs
```

4\. 启动界面（ip:8080）

 ![shipyard_1](/images/shipyard_1.png)

5\. 容器详细情况

![shipyard_2](/images/shipyard_2.png)

6\. 镜像

![shipyard_3](/images/shipyard_3.png)

7\. 节点

![shipyard_4](/images/shipyard_4.png)
