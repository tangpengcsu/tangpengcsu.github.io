---
layout: post
title: Redhat 7/CentOS 7 SSH 免密登录
date: 2016-10-28 18:39:04
tags: [Linux, Redhat, CentOS]
categories: [Linux]
---


## 先决条件

**3 台 CentOS 7**

| HOSTNAME | IP| ROLE |
| ------------- | ------------- | ------------- |
| server1 | 10.8.26.197 | Master |
| server2 | 10.8.26.196 | Slave1 |
| server3 | 10.8.26.195 | Slave2 |

<!-- more -->

## 步骤

1\. 用 root 用户登录。每台服务器都生成公钥，再合并到 authorized_keys。

2\. CentOS 默认没有启动 ssh 无密登录，去掉 `/etc/ssh/sshd_config` 其中 2 行的注释，每台服务器都要设置。

```bash
RSAAuthentication yes
PubkeyAuthentication yes
```

3\. 每台服务器下都输入命令 `ssh-keygen -t rsa`，生成 key，一律不输入密码，直接回车，`/root` 就会生成 `.ssh` 文件夹。

4\. 在 Master 服务器下，合并公钥到 `authorized_keys` 文件，进入 `/root/.ssh` 目录，通过 SSH 命令合并.

```bash
# cat id_rsa.pub>> authorized_keys
# ssh root@10.8.26.196 cat ~/.ssh/id_rsa.pub>> authorized_keys
# ssh root@10.8.26.195 cat ~/.ssh/id_rsa.pub>> authorized_keys
```

5\. 把 Master 服务器的 `authorized_keys`、`known_hosts` 复制到 Slave 服务器的 ``/root/.ssh` 目录

```bash
# scp authorized_keys root@server2:/root/.ssh/
# scp authorized_keys root@server3:/root/.ssh/
# scp known_hosts root@server2:/root/.ssh/
# scp known_hosts root@server3:/root/.ssh/
```

6\. 完成，`ssh root@10.8.26.196`、`ssh root@10.8.26.195` 就不需要输入密码了。
