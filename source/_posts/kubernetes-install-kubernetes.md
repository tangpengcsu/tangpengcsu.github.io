---
layout: post
title: 在 Linux 上安装 kubernetes
date: 2016-10-26 12:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---


## 先决条件

1. 操作系统： Ubuntu 16.04, CentOS 7 or HypriotOS v1.0.1+
2. 至少 1GB RAM
3. 确保集群内所有计算机之间的网络连接（公共或专用网络都行）

## 目标

- 在你的机器上安装一个安全的 Kubernetes 集群
- 在集群上安装一个 pod 网络，一遍应用组件（pods）之间可以正常通信。

<!-- more -->

## 安装

### 在主机上安装 kubelet 和 kubeadm

以下为将要在你的主机上安装的包：

- docker
- kubelet
- kubectl
- kubeadm

依次为每个主机进行安装配置：

1\. 切换为 root 用户 `su root`

2\. 如果你的机器是运行的 ubuntu 16.04 或 HypriotOS v1.0.1，执行如下命令：

```bash
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# cat <<EOF> /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
# apt-get update
# # Install docker if you don't have it already.
# apt-get install -y docker.io
# apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

3\. CentOS 7，执行如下命令：

```bash
# cat <<EOF> /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
       https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
# setenforce 0
# yum install -y docker kubelet kubeadm kubectl kubernetes-cni
# systemctl enable docker && systemctl start docker
# systemctl enable kubelet && systemctl start kubelet
```

### 初始化 master


```bash
 # kubeadm init
```

输出结果大体这样：


```bash
<master/tokens> generated token: "f0c861.753c505740ecde4c"
<master/pki> created keys and certificates in "/etc/kubernetes/pki"
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"
<util/kubeconfig> created "/etc/kubernetes/admin.conf"
<master/apiclient> created API client configuration
<master/apiclient> created API client, waiting for the control plane to become ready
<master/apiclient> all control plane components are healthy after 61.346626 seconds
<master/apiclient> waiting for at least one node to register and become ready
<master/apiclient> first node is ready after 4.506807 seconds
<master/discovery> created essential addon: kube-discovery
<master/addons> created essential addon: kube-proxy
<master/addons> created essential addon: kube-dns

Kubernetes master initialised successfully!

You can connect any number of nodes by running:

kubeadm join --token <token> <master-ip>
```

记录下 **kubeadm init** 输出的 **kubeadm join** 命令行。

### 安装节点网络插件

你必须在安装一个 pod 网络插件，以确保 pods 之间能够相互通信。

- [kubernetes 支持的 pod 网络插件类型](http://kubernetes.io/docs/admin/addons/)

    1. Calico
    2. Canal
    4. Flannel
    5. Romana
    6. Weave

通过如下命令安装 pod 网络插件：

```bash
# kubectl apply -f <add-on.yaml>
```

以 Calico 网络插件为例，在 [Calico 官网](http://docs.projectcalico.org/v1.6/getting-started/kubernetes/installation/hosted/) 上下载 calico.yaml 文件到本地，然后执行如下命令：

```bash
# kubectl apply -f calico.yaml
```

具体细节请参阅特定插件安装指南。一个集群中只能安装一个 pod 网络。

### 添加节点

节点作为工作负载运行容器和 pods 等。如果你要将一个新的机器作为节点加入集群中，须将每个机器切换为 root 用户，并执行之前 **kubeadm init** 的输出命令，例如：

```bash
# kubeadm join --token <token> <master-ip>
<util/tokens> validating provided token
<node/discovery> created cluster info discovery client, requesting info from "http://138.68.156.129:9898/cluster-info/v1/?token-id=0f8588"
<node/discovery> cluster info object received, verifying signature using given token
<node/discovery> cluster info signature and contents are valid, will use API endpoints [https://138.68.156.129:443]
<node/csr> created API client to obtain unique certificate for this node, generating keys and certificate signing request
<node/csr> received signed certificate from the API server, generating kubelet configuration
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"

Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```

在 master 上运行 **kubectl get nodes** 命名即可插件节点集群信息。

## 可选配置

### 非 master 节点控制集群

```bash
# scp root@<master ip>:/etc/kubernetes/admin.conf .
# kubectl --kubeconfig ./admin.conf get nodes
```

## 撤销 **kubeadm**

撤销 **kubeadm**，只需执行如下命令：

```bash
# kubeadm reset
```

如果你想重新启动集群，执行 **systemctl start kubelet** ，再执行 **kubeadm init** 或 **kubeadm join** 。
