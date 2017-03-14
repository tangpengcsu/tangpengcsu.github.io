---
layout: post
title: 配置 HTTPS 访问的 Harbor:v05.0
date: 2016-10-24 12:39:04
tags: [Harbor]
categories: [Harbor]
---

Harbor 默认 HTTP 请求注册服务器，并不分发任何证书。使得其配置相对简单。然而，强烈建议在生产环境中增强安全性。Harbor 有一个 Nginx 实例为所有服务提供反向代理，你可以在 Nginx 配置中启用 HTTPS 。

## 获取证书

假定你的仓库的 **hostname** 是 **reg.yourdomain.com**，并且其 DNS 记录指向主机正在运行的 Harbor。首先，你需要获取一个 CA 证书。证书通常包含一个 .crt 文件和 .key 文件，例如， **yourdomain.com.crt** 和 **yourdomain.com.key**。

在测试或开发环境中，你可能会选择使用自签名证书，而不是购买 CA 证书。下面的命令会生成自签名证书：

1) 生成自签名 CA 证书:

```bash
openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout ca.key \
  -x509 -days 365 -out ca.crt
```

<!-- more -->

2) 生成证书签名请求:

如果使用 FQDN (完全限定域名) 如 **reg.yourdomain.com** 作为你的注册服务器连接，那你必须使用 **reg.yourdomain.com** 作为 CN (Common Name)。另外，如果使用 IP 地址作为你的注册服务器连接，CN 可以是你的名字等等：

```bash
openssl req \
  -newkey rsa:4096 -nodes -sha256 -keyout yourdomain.com.key \
  -out yourdomain.com.csr
```

3) 生成注册服务器证书:

假定你使用类似 **reg.yourdomain.com** 这样的 FQND 作为注册服务器连接，运行下面的命令为你的注册服务器生成证书：

```bash
openssl x509 -req -days 365 -in yourdomain.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out yourdomain.com.crt
```

假定你使用 **IP** ， 如 **192.168.1.101** 作为注册服务器连接，你可以运行以下命令：

```bash
echo subjectAltName = IP:192.168.1.101 > extfile.cnf

openssl x509 -req -days 365 -in yourdomain.com.csr -CA ca.crt -CAkey ca.key -CAcreateserial -extfile extfile.cnf -out yourdomain.com
.crt
```

## 配置和安装

在取得 **yourdomain.com.crt** 和 **yourdomain.com.key** 文件后，你可以将它们放置在如  ```/root/cert/``` 目录下：

```bash
cp yourdomain.com.crt /root/cert/
cp yourdomain.com.key /root/cert/
```

接下来，编辑 harbor/make/harbor.cfg, 更新 ```hostname``` ， ```protocol```，  ```ssl_cert``` 和 ```ssl_cert_key``` 属性：

```bash
#set hostname
hostname = reg.yourdomain.com
#set ui_url_protocol
ui_url_protocol = https
......
#The path of cert and key files for nginx, they are applied only the protocol is set to https
ssl_cert = /root/cert/yourdomain.com.crt
ssl_cert_key = /root/cert/yourdomain.com.key
```

为 Harbor 生成配置文件：

```bash
# 工作目录 harbor/make
./prepare
```

如果 Harbor 已经在运行，则停止并删除实例。你的镜像数据将保留在文件系统中。

```bash
# 工作目录 harbor/make
# 在运行 compose 前， 需将 docker-compose.tpl 文件重命名为 docker-compose.yaml
mv docker-compose.tpl docker-compose.yaml
docker-compose down  
```

最后，重启 Harbor:

```bash
docker-compose up -d
```

在为 Harbor 配置 HTTPS 完成后，你可以通过以下步骤对它进行验证：

1. 打开浏览器，并输入地址：https://reg.yourdomain.com。应该会显示 Harbor 的界面。
2. 在装有 Docker daemon 的机器上，确保 Docker engine 配置文件中没有配置 "-insecure-registry"，并且，你必须将已生成的 ca.crt 文件放入 /etc/docker/certs.d/yourdomain.com(或 / etc/docker/certs.d/your registry host IP) 目录下，如果这个目录不存在，则创建它。
如果你将 nginx 的 443 端口映射到其他端口上了，你需要创建目录的为 /etc/docker/certs.d/yourdomain.com:port(/etc/docker/certs.d/your registry host IP:port)。然后，运行如下命令验证是否安装成功。

```bash
docker login reg.yourdomain.com
```

如果你将 nginx 443 端口映射到其他端口上，则需要在登录时添加端口号，如：

```bash
docker login reg.yourdomain.com:port
```

## 疑难解答

1\. 你可能是通过证书发行商获取中间证书。在这种情况下，你应该合并中间证书与自签证书，通过如下命令即可实现：

```bash
cat intermediate-certificate.pem >> yourdomain.com.crt
```

2\. 在一些运行着 docker daemon 的系统中，你可能需要操作系统级别的信任证书。    

  - 在 Ubuntu 上, 可以通过以下命令来完成：

```bash
cp youdomain.com.crt /usr/local/share/ca-certificates/reg.yourdomain.com.crt
update-ca-certificates      
```       

  - 在 Red Hat (CentOS etc) 上, 命令如下:       

```
cp yourdomain.com.crt /etc/pki/ca-trust/source/anchors/reg.yourdomain.com.crt
update-ca-trust
```
