---
layout: post
title: Volumes
date: 2016-10-26 18:39:04
tags: [Kubernetes]
categories: [Kubernetes]
---


## k8s 支持一下类型的 volumes

1. emptyDir
2. hostPath
3. gcePersistentDisk
4. awsElasticBlockStore
5. nfs
6. iscsi
7. flocker
8. glusterfs
9. rbd
10. cephfs
11. gitRepo
12. secret
13. persistentVolumeClaim
14. downwardAPI
15. azureFileVolume
16. azureDisk
17. vsphereVolume
18. Quobyte

<!-- more -->

## emptyDir

当一个 pod 指定到一个 node 的时候，一个 emptyDir volumn 就会被创建，只要 pod 在 node 上运行，该 volume 就会一直存在。就像他的名称一样，初始化的时候是个空的目录。在这个 emptyDir 中，该 Pod 中的所有容器可以读写到相同的文件。

虽然，这个 emptyDir 会被挂载到每一个容器相同或者不同的目录。当一个 Pod 被从 node 移除后，对应的 emptyDir 中的数据会被永久删除。 注意：一个容器崩溃不会将 pod 从 node 上移除，所以一个 container 崩溃，在 emptyDir 中的数据是安全的。 

Some uses for an emptyDir are:
  scratch space, such as for a disk-based merge sort
  checkpointing a long computation for recovery from crashes
  holding files that a content-manager container fetches while a webserver container serves the data

默认情况下，emptyDir volume are stored 不论你后台机器的存储媒介是什么。媒介可能是磁盘或 SSD，或者网络存储，这依赖于你的环境。你可以通过把 emptyDir.medium 设置成 "memory" 来告诉 k8s, 挂载一个 tmpfs（RAM-backed filesystem）。tmpfs 非常快，但是它不像 disk，当你的物理机重启后，tmpfs 将会被清空，你所生成的任何文件，都会对你的 container 的内存限制造成影响。

创建一个 pod 并且使用 emptyDir volume:

```
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: empty
      mountPath: /mnt
  volumes:
  - name: empty
    emptyDir:
      Medium: ""# 这里可以使用空或者使用"memory" 关键字，来制定不通的存储媒介
```

创建完后，我们通过进入 container 中查看 / mnt 所挂载的类型

```bash
$ kubectl exec -p test-emptydir -it /bin/bash
root@test-emptydir:/# findmnt -m /mnt
```

## hostPath 

一个 hostPath Volume 挂载一个来自 host node 的文件系统的一个文件或者目录到你的 pod，这里不需要 pod 做什么， it offers a powerful escape hatch for some applications。

下面是，一些使用 hostPath 的例子

1. 运行一个容器，需要访问 Docker 的内部，则用 hostPath 挂载 / var/lib/docker
2. 在 container 中运行一个 cAdvisor，使用 hostPath 挂载 /dev/cgroups

当使用 hostPath 的时候需要注意，因为：

1. 因为在不同的 node 中，文件不同，所以使用相同配置的 pod（比如，从 podTemplate 创建的）可能在不同的 node 上，有不同的行为。
2. 当 k8s 添加 resource-aware scheduling，就不能认为资源是由 hostPath 使用（ when Kubernetes adds resource-aware scheduling, as is planned, it will not be able to account for resources used by a hostPath）
3. 在 underlying hosts 创建的文件夹，只能被 root 写，你需要在一个特权容器中，使用 root 运行你的进程，或者更改 host 的文件权限，以便可以写入 hostPath volume。

**例：**

```
apiVersion: v1
kind: Pod
metadata:
  name: test-pdspec:
  containers:
  - image: gcr.io/google_containers/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
--- 
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpathspec:
  containers:
  - image: myimage
    name: test-container
    volumeMounts:
    - mountPath: /test-hostpath
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /path/to/my/dir
```

## NFS 
一个 nfs volume 允许一个已存在的 NFS（Network file system） 可以共享的挂载到你的 pod， 不像 emptyDir, 当一个 Pod 移除的时候，就会被擦出， nfs volume 中的内容是保留的，volume 仅仅是变为 unmounted，这意味着一个 NFS volume 可以在其中预先放入一些数据。这些数据可以被 "handed off" 在 pod 之间。 NFS 可以被多个写入着同时挂载（  NFS can be mounted by multiple writers simultaneously）。
 
重要： 在你可以使用 share exports 前，你必须让你自己的 NFS 服务器运行起来。 

一些 k8s 的概念（pv,pvc）

1. Persistent Volumes 定义一个持久的 disk（disk 的生命周期不和 Pod 相关联）
2. Services  to enable Pods to locate one another.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: zk-mysql-pv-2
  labels:
    type: nfs
    app: mysql
    tier: zk
spec:
 capacity:
   storage: 20Gi
 accessModes:
   - ReadWriteMany
 persistentVolumeReclaimPolicy: Recycle
 nfs:
   path: /mnt/usercenter/zk/mysql-pv-2
   server: 192.168.1.41
```

## glusterfs

> 参考：[https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/glusterfs](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/glusterfs)

GlusterFS 是一个开源的分布式文件系统，具有强大的横向扩展能力，通过扩展能够支持数 PB 存储容量和处理数千客户端。同样地，Kubernetes 支持 Pod 挂载到 GlusterFS，这样数据将会永久保存。

首先需要一个 GlusterFS 环境，本文使用 2 台机器（CentOS 7）安装 GlusterFS 服务端，在 2 台机器的 /etc/hosts 配置以下信息：

- 192.168.3.150    gfs1
- 192.168.3.151    gfs2

在 2 台机器上安装：

```bash
$ wget http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-5.noarch.rpm
$ rpm -ivh epel-release-7-5.noarch.rpm
$ wget -P /etc/yum.repos.d http://download.gluster.org/pub/gluster/glusterfs/LATEST/EPEL.repo/glusterfs-epel.repo
$ yum install glusterfs-server
$ service glusterd start
```

注意: GlusterFS 服务端需要关闭 SELinux：修改 / etc/sysconfig/selinux 中 SELINUX=disabled。然后清空 iptables: iptables -F

安装后查询状态：

```bash
$ service glusterd status
```

添加集群：

在 gfs1 上操作：

```bash
$ gluster peer probe gfs2
```

在 gfs2 上操作：

```bash
$ gluster peer probe gfs1
```

创建 volume ：

在 gfs1 上操作：

```bash
$ mkdir /data/brick/gvol0
$ gluster volume create gvol0 replica 2 gfs1:/data/brick/gvol0 gfs2:/data/brick/gvol0
$ gluster volume start gvol0
$ gluster volume info
```

安装成功的话可以挂载试验下：

```bash
$ mount -t glusterfs gfs1:/gvol0 /mnt
```

GlusterFS 服务端安装成功后，需要在每个 Node 上安装 GlusterFS 客户端:

```bash
$ yum install glusterfs-client
```

接着创建 GlusterFS Endpoint，gluasterfs-endpoints.json：

```json
{
"kind": "Endpoints",
"apiVersion": "v1",
"metadata": {
"name": "glusterfs-cluster"
},
"subsets": [
{
  "addresses": [
    {
      "IP": "192.168.3.150"
    }
  ],
  "ports": [
    {
      "port": 1
    }
  ]
},
{
  "addresses": [
    {
      "IP": "192.168.3.151"
    }
  ],
  "ports": [
    {
      "port": 1
    }
  ]
}
]
}
```

最后创建一个 Pod 挂载 GlusterFS Volume, busybox-pod.yaml:

```
apiVersion: v1
kind: Pod
metadata:
name: busybox
labels:
name: busybox
spec:
containers:
- image: busybox
  command:
    - sleep
    - "3600"
  imagePullPolicy: IfNotPresent
  name: busybox
  volumeMounts:
    - mountPath: /busybox-data
      name: data
volumes:
- glusterfs:
  endpoints: glusterfs-cluster
  path: gvol0
  name: data
```

查看 Pod 的容器，可以看到容器挂载

```bash
/var/lib/kubelet/pods/<id>/volumes/kubernetes.io~glusterfs/data => /busybox-data
```

在 kubernetes.io~glusterfs/data 目录下执行查询：

```bash
$ mount | grep gvol0
192.168.3.150:gvol0 on /var/lib/kubelet/pods/<id>/volumes/kubernetes.io~glusterfs/data type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072)
```
