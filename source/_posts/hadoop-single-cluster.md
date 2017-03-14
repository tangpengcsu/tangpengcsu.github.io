---
layout: post
title: 配置 Hadoop 单节点集群
date: 2016-12-24 12:39:04
tags: [Hadoop]
categories: [Hadoop]
---


## 目的

本文描述如何配置一个单节点 Hadoop 集群，使得你可以使用 Hadoop MapReduce 和 Hadoop 分布式文件系统（HDFS）快速执行简单的操作。

<!-- more -->

***

## 先决条件

### 平台支持

- GNU/Linux
- windows

### 所需软件

Linux 所需软件：

 1. [Java](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)
 2. SSH

### 软件安装

以 ubuntu 为例：

```bash
$ sudo apt-get install ssh
$ sudo apt-get install rsync
```

***

## 下载

在 [Apache Download Mirrors](http://www.apache.org/dyn/closer.cgi/hadoop/common/) 下载最新的稳定版 Hadoop。

***

## 准备启动 Hadoop 集群

解压已下载 Hadoop 包， 编辑 `etc/hadoop/hadoop-env.sh` 文件添加如下变量：

```bash
# set to the root of your Java installation
export JAVA_HOME=/usr/java/latest
---------------------------------------------
## 例如
export JAVA_HOME=/usr/local/jdk1.8.0_102
```

尝试执行如下命令

```bash
[root@server1 hadoop-2.7.0]# bin/hadoop
```

这条命令将显示 hadoop 脚本如何使用的。

现在已经装备好启动 Hadoop 集群了，集群支持一下三种模式：

- 单机模式 - Local (Standalone) Mode
- 伪集群模式 - Pseudo-Distributed Mode
- 集群分布式模式 - Fully-Distributed Mode

***

## 单机运行 - Standalone Operation

```bash
[root@server1 hadoop-2.7.0]# mkdir input
[root@server1 hadoop-2.7.0]# cp etc/hadoop/*.xml input
[root@server1 hadoop-2.7.0]# bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar grep input output 'dfs[a-z.]+'
[root@server1 hadoop-2.7.0]# cat output/*
```

***

## 伪分布模式运行 - Pseduo-Distributed Operation

### 配置

修改下面的文件

**etc/hadoop/core-site.xml:**

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

**etc/hadoop/hdfs-site.xml:**

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

### 设置 SSH 免密登录 - Setup passphraseless ssh

现在，检查是否可以 SSH 免密登录 localhost:

```bash
# ssh localhost
```
如果登录失败，执行以下命令：

```bash
# ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
# cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
# export HADOOP\_PREFIX=/usr/local/hadoop
```

> ps: 免密登录配置详情见 [Redhat7/CentOS7 SSH 免密登录]({{ site.url }}/linux/linux-ssh-authentication/)

### 执行

1\. 格式化文件系统：

```bash
[root@server1 hadoop-2.7.0]# bin/hdfs namenode -format
```

> Hadoop 临时文件放在 `/tmp/hadoop-root/` 目录下，如果需要可以在重置前删除。

2\. 启动 NameNode daemon 和 DataNode daemon:

```bash
[root@server1 hadoop-2.7.0]# sbin/start-dfs.sh
```

Hadoop daemon 日志会将输出到 `$HADOOP_LOG_DIR` (默认 `$HADOOP_HOME/logs`)。

3\. web 界面浏览 NameNode；默认 URL：

- NameNode - http://localhost:50070/

4\. 为执行 MapReduce jobs 创建所需的 HDFS 目录：

```bash
[root@server1 hadoop-2.7.0]# bin/hdfs dfs -mkdir /user
[root@server1 hadoop-2.7.0]# bin/hdfs dfs -mkdir /user/<username>
## 这里我用 tpfs 代替 username:
[root@server1 hadoop-2.7.0]# bin/hdfs dfs -mkdir /user/root/
```

5\. 将 `etc/hadoop` 目录复制到分布式文件系统（HDFS）的 `input` 目录下

```bash
[root@server1 hadoop-2.7.0]# bin/hdfs dfs -put etc/hadoop input
```

6\. 执行一个已经提供的案例

```bash
[root@server1 hadoop-2.7.0]# bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar grep input output 'dfs[a-z.]+'
```

7\. 检查输出文件，拷贝 HDFS 上的输出文件到本机，执行如下命令：

```bash
[root@server1 hadoop-2.7.0]# bin/hdfs dfs -get output output
[root@server1 hadoop-2.7.0]# ls output/
part-r-00000  _SUCCESS
```

或

在 HDFS 上查看输出文件

```bash
[root@server1 hadoop-2.7.0]# bin/hdfs dfs -ls output/*
-rw-r--r--   1 root supergroup          0 2016-12-29 11:21 output/part-r-00000
-rw-r--r--   1 root supergroup          0 2016-12-29 11:21 output/_SUCCESS
```

8\. 当你执行完成后，执行如下停止 daemons：

```bash
[root@server1 hadoop-2.7.0]# sbin/stop-dfs.sh
```

***

## 集群分布式运行

详情见[Cluster Setup](http://hadoop.apache.org/docs/r2.7.0/hadoop-project-dist/hadoop-common/ClusterSetup.html)
