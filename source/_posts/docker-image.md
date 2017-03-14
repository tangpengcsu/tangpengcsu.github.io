---
layout: post
title: 镜像
date: 2016-10-24 12:39:04
tags: [Docker]
categories: [Docker]
---

镜像是 Docker 的三大组件之一。Docker 运行容器前需要本地存在对应的镜像，如果镜像不存在本地，Docker  会从镜像仓库下载（[Docker Store](https://store.docker.com/ "https://store.docker.com/") ，[阿里云](https://dev.aliyun.com/search.html "https://dev.aliyun.com/search.html")，[Harbor](https://github.com/vmware/harbor "https://github.com/vmware/harbor")...）。下面主要从如下两个方面介绍镜像相关内容：

- 拉取仓库镜像。
- 管理本地主机上的镜像。

<!-- more -->

## 拉取镜像

使用 `docker pull` 命令来从仓库获取所需要的镜像。例如，将从 Docker Store 仓库下载一个 Ubuntu 12.04 操作系统的镜像。

```bash
$ sudo docker pull ubuntu:12.04
Pulling repository ubuntu
ab8e2728644c: Pulling dependent layers
511136ea3c5a: Download complete
5f0ffaa9455e: Download complete
a300658979be: Download complete
904483ae0c30: Download complete
ffdaafd1ca50: Download complete
d047ae21eeaf: Download complete
```

下载过程中，会输出获取镜像的每一层信息。该命令实际上相当于 `$ sudo docker pull store.docker.com/ubuntu:12.04 ` 命令，即从注册服务器 **store.docker.com** 中的 ubuntu 仓库来下载标记为 12.04 的镜像。通常官方仓库注册服务器下载较慢，可以从其他仓库下载（阿里镜像库，daoclound 以及自己搭建的私有仓库等）。从其它仓库下载时需要指定完整的仓库注册服务器地址。例如：

```bash
$ sudo docker pull 192.168.1.40/library/ubuntu:12.04
Pulling 192.168.1.40/library/ubuntu
ab8e2728644c: Pulling dependent layers
511136ea3c5a: Download complete
5f0ffaa9455e: Download complete
a300658979be: Download complete
904483ae0c30: Download complete
ffdaafd1ca50: Download complete
d047ae21eeaf: Download complete
```

完成后，即可随时使用该镜像了，例如创建一个容器，让其中运行 bash 应用。

```bash
$ sudo docker run -ti 192.168.1.40/library/ubuntu:12.04 /bin/bash
root@fe7fc4bd8fc9:/#
```

## 查看本地镜像

使用 `docker images` 命令显示本地已有的镜像。

```bash
$ sudo docker images
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
192.168.1.40/library/ubuntu 12.04 74fe38d11401 4 weeks ago 209.6 MB
ubuntu precise 74fe38d11401 4 weeks ago 209.6 MB
ubuntu 14.04 99ec81b80c55 4 weeks ago 266 MB
ubuntu latest 99ec81b80c55 4 weeks ago 266 MB
ubuntu trusty 99ec81b80c55 4 weeks ago 266 MB
……
```

在列出信息中，可以看到几个字段信息

- REPOSITORY: 来自于哪个仓库，比如 192.168.1.40/library/ubuntu
- IMAGES 镜像的标记，比如 14.04
- ID: 它的 ID 号（唯一）
- CREATED: 创建时间
- SIZE: 镜像大小

其中镜像的 ID 唯一标识了镜像，注意到 ubuntu:14.04 和 ubuntu:trusty 具有相同的镜像 ID，说明它们实际上是同一镜像。

TAG 信息用来标记来自同一个仓库的不同镜像。例如 ubuntu 仓库中有多个镜像，通过 TAG 信息来区分发行版本，例如 10.04、12.04、12.10、13.04、14.04 等。例如下面的命令指定使用镜像 ubuntu:14.04 来启动一个容器。

```bash
$ sudo docker run -ti ubuntu:14.04 /bin/bash
```

如果不指定具体的标记，则默认使用 latest 标记信息。

## 搜索镜像

> docker search [--automated=false] [--no-trunc=false] [-s] [IMAGE_NAME]

- --automated=false: 仅显示自动创建的镜像。
- --no-trunc-false: 输出信息不截断显示。    
- -s, --stars=0: 指定仅显示评价为指定星级以上的镜像。

使用 `docker search` 命令搜索镜像仓库（[Docker Store](https://store.docker.com/ "https://store.docker.com/") ，[阿里云](https://dev.aliyun.com/search.html "https://dev.aliyun.com/search.html")，[Harbor](https://github.com/vmware/harbor "https://github.com/vmware/harbor")...）中共享的镜像，默认搜索 Docker Store 官方仓库中的镜像。例如：

```bash
$ docker search ubuntu
```

## 删除镜像

> docker rmi [--force]  IMAGE [IMAGE…]

使用 `docker rmi` 命令删除镜像，例如。

```bash
$ docker rmi 192.168.1.40/library/ubuntu:12.04
```

## 镜像构建

`镜像构建` 的方法很多，用户可以从 Docker Store 获取已有镜像并更新，也可以利用本地文件系统创建一个。`修改已有镜像`，先使用已下载的镜像启动容器。

```bash
$ sudo docker run -t -I ubuntu:14.04 /bin/bash
```

- 注意：记住容器的 ID，稍后还会用到。在容器中添加 gcc。

```bash
root@0b2616b0e5a8:/# apt-get install gcc
```

当结束后，我们使用 `exit` 来退出，现在我们的容器已经被我们改变了，使用 docker commit 命令来提交更新后的副本。

```bash
$ sudo docker commit -m "Added gcc" -a "Docker Newbee" 0b2616b0e5a8 test/ubuntu:v1
4f177bd27a9ff0f6dc2a830403925b5360bfe0b93d476f7fc3231110e7f71b1c
```
- 其中，-m 来指定提交的说明信息，跟我们使用的版本控制工具一样；-a 可以指定更新的用户信息；之后是用来创建镜像的容器的 ID；最后指定目标镜像的仓库名和 tag 信息。

创建成功后会返回这个镜像的 ID 信息。使用 `docker images` 来查看新创建的镜像。

```bash
$ sudo docker images
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
ubuntu 14.04 99ec81b80c55 4 weeks ago 266 MB
test/ubuntu:v1 v1 4f177bd27a9ff0f6 10 hours ago 287 MB
```

之后，可以使用新的镜像来启动容器

```bash
$ sudo docker run -t -i test/ubuntu:v1 /bin/bash
```

### 利用 Dockerfile 来创建镜像

使用 `docker commit` 来扩展一个镜像比较简单，但是`不方便在一个团队中分享`。我们可以使用 `docker build` 来创建一个新的镜像。为此，首先需要创建一个 `Dockerfile`，包含一些如何创建镜像的指令。新建一个目录和一个 Dockerfile。

```bash
$ mkdir ubuntu-test
$ cd ubuntu-test
$ touch Dockerfile
```

Dockerfile 中每一条指令都创建镜像的一层，例如：

```bash
# This is a comment
FROM ubuntu:14.04
MAINTAINER tangpeng tang.peng@hotmail.com
RUN apt-get  update
RUN apt-get install -y gcc
```

Dockerfile 基本的语法是使用 # 注释内容，FROM 指令告诉 Docker 使用哪个镜像作为基础，接着是维护者的信息 RUN 开头的指令会在创建中运行，比如安装一个软件包，在这里使用 apt-get 来安装了一些软件，编写完成 Dockerfile 后可以使用 docker build 命令来生成镜像。

```bash
$ sudo docker build –t test/ubuntu:v2 .
Uploading context 2.56 kB
Uploading context
Step 0 : FROM ubuntu:14.04
 ---> 99ec81b80c55
Step 1 : MAINTAINER tangpeng tang.peng@hotmail.com
 ---> Running in 7c5664a8a0c1
 ---> 2fa8ca4e2a13
Removing intermediate container 7c5664a8a0c1
Step 2 : RUN apt-get update
 ---> Running in b07cc3fb4256
 ---> 50d21070ec0c
Removing intermediate container b07cc3fb4256
Step 3 : RUN apt-get install -y gcc
 ---> Running in a5b038dd127e
 ---> 324104cde6ad
Removing intermediate container a5b038dd127e
Successfully built 324104cde6ad
```

其中 `-t 标记来添加 tag`，指定新的镜像的用户信息。"`.`" 是 Dockerfile 所在的`路径`（当前目录），也可以替换为一个具体的 Dockerfile 的路径。可以看到 build 进程在执行操作。它要做的第一件事情就是上传这个 Dockerfile 内容，因为所有的操作都要依据 Dockerfile 来进行。然后，Dockerfile 中的指令被一条一条的执行。每一步都创建了一个新的容器，在容器中执行指令并提交修改（就跟之前介绍过的 docker commit 一样）。当所有的指令都执行完毕之后，返回了最终的`镜像 id`。所有的中间步骤所产生的容器都被删除和清理了。

注意一个`镜像不能超过 127 层`。此外，还可以利用 ADD 或 COPY 命令复制本地文件到镜像；用 `EXPOSE` 命令来向外部开放端口；用 `CMD` 命令来描述容器启动后运行的程序等。例如:

```bash
# put my local web site in myApp folder to /var/www

ADD myApp /var/www

# expose httpd port

EXPOSE 80

# the command to run

CMD ["/usr/sbin/apachectl", "-D", "FOREGROUND"]
```

现在可以利用新创建的镜像来启动一个容器。

```bash
$ sudo docker run -ti test/ubuntu:v2 /bin/bash
root@8196968dac35:/#
```

## 载出与载入镜像

### 语法

> docker save -o [IMAGE_FILE_NAME] [IMAGE_NAME:TAG]
> docker load <[IMAGE_FILE_NAME] 或 docker load --input [IMAGE_FILE_NAME]

### 载出镜像

使用 `docker save` 命令打包镜像到本地。例如，载出一个 ubuntu:v2 镜像为文件 ubuntu_v2.tar。

```bash
$ docker save -o ubuntu_v2.tar ubuntu:v2
```

### 载入镜像

使用 `docker load` 命令将本地镜像文件导入到本地镜像库中。例如，载入一个 ubuntu_v2.tar  镜像文件。

```bash
$ docker load < ubuntu:v2.tar
```

或

```bash
$ docker load --input ubuntu:v2.tar
```

### 标记镜像

使用 `docker tag` 命令来修改镜像的标签。

```bash
$ sudo docker tag 324104cde6ad test/ubuntu:v2.1
$ sudo docker images test/ubuntu
REPOSITORY TAG IMAGE ID CREATED VIRTUAL SIZE
test/ubuntu v1 5db5f8471261 11 hours ago 287  MB
test/ubuntu v2 5db5f8471261 11 hours ago 446.7 MB
test/ubuntu v2.1 5db5f8471261 11 hours ago 446.7 MB
```

> 更多用法，参考 [Dockerfile](/2016/10/24/docker-dockerfile-reference/ "/2016/10/24/docker-dockerfile-reference/") 章节。

## 上传镜像

上传镜像用户可以通过 `docker push` 命令，把自己创建的镜像上传到仓库中来共享。例如，用户在 Docker Store 上完成注册并审核后，可以推送自己的镜像到仓库中。

```bash
$ sudo docker push test/ubuntu:v2.1

The push refers to a repository [test/ubuntu:v2.1] (len: 1)

Sending image list

Pushing repository test/ubuntu:v2.1 (3 tags)
```
