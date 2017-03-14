---
layout: post
title: Firewalld 防火墙
date: 2016-10-28 18:39:04
tags: [Linux]
categories: [Linux]
---


Firewalld 服务是红帽 RHEL7 系统中默认的防火墙管理工具，特点是拥有运行时配置与永久配置选项且能够支持动态更新以及 "zone" 的区域功能概念，使用图形化工具 firewall-config 或文本管理工具 firewall-cmd，下面实验中会讲到~

<!-- more -->

## 区域概念与作用

防火墙的网络区域定义了网络连接的可信等级，我们可以根据不同场景来调用不同的 firewalld 区域，区域规则有：

| 区域 | 默认规则策略 |
| ------------- | ------------- |
| trusted | 允许所有的数据包。 |
| home | 拒绝流入的数据包，除非与输出流量数据包相关或是 ssh,mdns,ipp-client,samba-client 与 dhcpv6-client 服务则允许。 |
| internal | 等同于 home 区域 |
| work | 拒绝流入的数据包，除非与输出流量数据包相关或是 ssh,ipp-client 与 dhcpv6-client 服务则允许。 |
| public | 拒绝流入的数据包，除非与输出流量数据包相关或是 ssh,dhcpv6-client 服务则允许。 |
| external | 拒绝流入的数据包，除非与输出流量数据包相关或是 ssh 服务则允许。 |
| dmz | 拒绝流入的数据包，除非与输出流量数据包相关或是 ssh 服务则允许。 |
| block | 拒绝流入的数据包，除非与输出流量数据包相关。 |
| drop | 拒绝流入的数据包，除非与输出流量数据包相关。 |

简单来讲就是为用户预先准备了几套规则集合，我们可以根据场景的不同选择合适的规矩集合，而默认区域是 public。

## 字符管理工具

如果想要更高效的配置妥当防火墙，那么就一定要学习字符管理工具 firewall-cmd 命令, 命令参数有：

| 参数 | 作用 |
| ------------- | ------------- |
| --get-default-zone | 查询默认的区域名称。 |
| --set-default-zone=<区域名称> | 设置默认的区域，永久生效。 |
| --get-zones | 显示可用的区域。 |
| --get-services | 显示预先定义的服务。 |
| --get-active-zones | 显示当前正在使用的区域与网卡名称。 |
| --add-source= | 将来源于此 IP 或子网的流量导向指定的区域。 |
| --remove-source= | 不再将此 IP 或子网的流量导向某个指定区域。
| --add-interface=<网卡名称> | 将来自于该网卡的所有流量都导向某个指定区域。 |
| --change-interface=<网卡名称> | 将某个网卡与区域做关联。 |
| --list-all | 显示当前区域的网卡配置参数，资源，端口以及服务等信息。 |
| --list-all-zones | 显示所有区域的网卡配置参数，资源，端口以及服务等信息。 |
| --add-service=<服务名> | 设置默认区域允许该服务的流量。 |
| --add-port=<端口号/协议> | 允许默认区域允许该端口的流量。 |
| --remove-service=<服务名> | 设置默认区域不再允许该服务的流量。 |
| --remove-port=<端口号/协议> | 允许默认区域不再允许该端口的流量。 |
| --reload | 让 “永久生效” 的配置规则立即生效，覆盖当前的。 |

特别需要注意的是 `firewalld 服务有两份规则策略配置记录`，必需要能够区分：

- RunTime: 当前正在生效的。
- Permanent: 永久生效的。

当下面实验修改的是永久生效的策略记录时，必须执行 "`--reload`" 参数后才能立即生效，否则要重启后再生效。

### 查看当前的区域：

```bash
$ firewall-cmd --get-default-zone
public
```

### 查询 eno16777728 网卡的区域：

```bash
$ firewall-cmd --get-zone-of-interface=eno16777728
public
```

### 在 public 中分别查询 ssh 与 http 服务是否被允许：

```bash
$ firewall-cmd --zone=public --query-service=ssh
yes
$ firewall-cmd --zone=public --query-service=http
no
```

### 设置默认规则为 dmz：

```bash
$ firewall-cmd --set-default-zone=dmz
```

### 让 “永久生效” 的配置文件立即生效：

```bash
$ firewall-cmd --reload
success
```

### 启动/关闭应急状况模式，阻断所有网络连接：

应急状况模式启动后会禁止所有的网络连接，一切服务的请求也都会被拒绝，当心，请慎用。

```bash
$ firewall-cmd --panic-on
success
$ firewall-cmd --panic-off
success
```

如果您已经能够完全理解上面练习中 firewall-cmd 命令的参数作用，不妨来尝试完成下面的模拟训练吧：

### 模拟训练 A: 允许 https 服务流量通过 public 区域，要求立即生效且永久有效：

#### 方法一: 分别设置当前生效与永久有效的规则记录：

```bash
$ firewall-cmd --zone=public --add-service=https
$ firewall-cmd --permanent --zone=public --add-service=https
```

#### 方法二: 设置永久生效的规则记录后读取记录：

```bash
$ firewall-cmd --permanent --zone=public --add-service=https
$ firewall-cmd --reload
```

### 模拟训练 B: 不再允许 http 服务流量通过 public 区域，要求立即生效且永久生效：

```bash
$ firewall-cmd --permanent --zone=public --remove-service=http
 success
```

使用参数 "--reload" 让永久生效的配置文件立即生效：

```bash
$ firewall-cmd --reload
success
```

### 模拟训练 C: 允许 8080 与 8081 端口流量通过 public 区域，立即生效且永久生效：

```bash
$ firewall-cmd --permanent --zone=public --add-port=8080-8081/tcp
$ firewall-cmd --reload
```

### 模拟训练 D: 查看模拟实验 C 中要求加入的端口操作是否成功：

```bash
$ firewall-cmd --zone=public --list-ports
8080-8081/tcp
$ firewall-cmd --permanent --zone=public --list-ports
8080-8081/tcp
```

### 模拟实验 E: 将 eno16777728 网卡的区域修改为 external，重启后生效：

```bash
$ firewall-cmd --permanent --zone=external --change-interface=eno16777728
success
$ firewall-cmd --get-zone-of-interface=eno16777728
public
```

### 端口转发功能可以将原本到某端口的数据包转发到其他端口:

```bash
firewall-cmd --permanent --zone=<区域> --add-forward-port=port=<源端口号>:proto=<协议>:toport=<目标端口号>:toaddr=<目标 IP 地址>
```

### 将访问 192.168.10.10 主机 888 端口的请求转发至 22 端口：

```bash
$ firewall-cmd --permanent --zone=public --add-forward-port=port=888:proto=tcp:toport=22:toaddr=192.168.10.10
success
```

### 使用客户机的 ssh 命令访问 192.168.10.10 主机的 888 端口：

```bash
$ ssh -p 888 192.168.10.10
The authenticity of host '[192.168.10.10]:888 ([192.168.10.10]:888)' can't be established.
ECDSA key fingerprint is b8:25:88:89:5c:05:b6:dd:ef:76:63:ff:1a:54:02:1a.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[192.168.10.10]:888' (ECDSA) to the list of known hosts.
root@192.168.10.10's password:
Last login: Sun Jul 19 21:43:48 2015 from 192.168.10.10
```

再次提示: 请读者们再仔细琢磨下立即生效与重启后依然生效的差别，千万不要修改错了。

### 模拟实验 F: 设置富规则，拒绝 192.168.10.0/24 网段的用户访问 ssh 服务：

firewalld 服务的富规则用于对服务、端口、协议进行更详细的配置，规则的优先级最高。

```bash
$ firewall-cmd --permanent --zone=public --add-rich-rule="rule family="ipv4"source address="192.168.10.0/24"service name="ssh"reject"
success
```

## 图形化工具 firewall-config

...
