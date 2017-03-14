---
layout: post
title: Spark on Mesos
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


Spark 可以在 Apache Mesos 管理的硬件集群上运行。

使用 Mesos 部署 Spark 的优点包括：

- Spark 和 其他框架 frameworks  之间的动态分区
- Spark 的多个实例之间的可扩展分区

<!-- more -->

***

## 怎么运行

在一个独立集群部署中，下图的集群管理器是 Spark master 的一个实例。当使用 Mesos 管理时，Mesos master 会替代 Spark master 作为集群的管理器。

现在，Driver 程序创建一个 job 并开始分发调度任务时，Mesos 会决定什么机器处理什么任务。 因为 Mesos 调度这些短期任务时会考虑到其他的框架，许多框架将会在同一个集群上共存，而不是借助资源的静态分区。

开始，请按照以下步骤安装 Mesos 并通过 Mesos 部署 Spark 作业。

***

## 安装 Mesos

Spark 2.0.1 设计用于 Mesos 0.21.0 或更新版本，不需要任何特殊的 Mesos 补丁。

如果你早已经有一个 Mesos 集群在运行，你可以跳过这个 Mesos 的安装步骤。

否则，安装 Mesos for Spark 与安装 Mesos for 其他的框架没有什么不同。你可以通过源码或者预构建软件安装包来安装 Mesos。

### 通过源码：

通过源码安装 Apache Mesos，按照以下步骤：

1. 从 镜像 mirror 下载 Mesos 版本
2. 按照 Mesos 开始页面 Getting Started 来编译和安装 Mesos

> 注意: 如果你希望运行 Mesos 又不希望安装在系统的默认位置（例如：如果你没有默认路径的管理权限），传递 --prefix 选项进行配置来告诉它安装在什么地方。例如： 传递 --prefix=/home/me/mesos。默认情况下，前缀是：/usr/local 。

### 第三方包

Apache Mesos 项目只发布了源码的版本，而不是二进制包。但是其他的第三方项目发布的二进制版本，可能对设置 Mesos 有帮助。

其中之一是中间层。用使用中间层提供的二进制版本安装 Mesos。

1. 从下载页面下载 Mesos 的安装包
2. 按照说明进行安装和配置

中间层的安装文档建议设置 zookeeper 来处理 Mesos Master 的故障转移，但是 Mesos 通过使用 Single Master 模式，可以在没有 zookeeper 的情况下运行。

### 验证

要验证 Mesos 集群是否可用于 Spark，导航到 Mesos Master 的 Web UI 界面，端口是：5050 来确认所有预期的机器都会与从属选项卡中。

***

## 连接 Spark 到 Mesos

通过使用 Spark 中的 Mesos, 你需要一个 Spark 的二进制包放到 Mesos 可以访问的地方，然后配置 Spark driver 程序来连接 Mesos。

或者你也可以在所有 Mesos slaves 位置安装 Spark，然后配置 spark.mesos.executor.home (默认是 SPARK_HOME) 来指向这个位置。

### 上传 Spark 包

当 Mesos 第一次在 Mesos 从服务器上运行任务时，该从服务器必须有一个 Spark 二进制包来运行 Spark Mesos 执行器后端。 Spark 包可以在任何 Hadoop 可访问的 URI 上托管，包括 HTTP 通过 http://，Amazon Simple Storage Service 通过 s3n:// 或 HDFS 通过 hdfs://。

要使用预编译包：

1. 从 Spark download page 下载 Spark 二进制包
2. 上传到 hdfs/http/s3

要在 HDFS 上主机，请使用 Hadoop fs put 命令：hadoop fs -put spark-2.0.1.tar.gz /path/to/spark-2.0.1.tar.gz

或者如果您使用的是自定义编译版本的 Spark，则需要使用包含在 Spark 源代码 tarball / checkout 中的 dev / make-distribution.sh 脚本创建一个包。

1. 使用 here 的说明下载并构建 Spark
2. 使用./dev/make-distribution.sh --tgz 创建二进制包。
3. 将归档文件上传到 http / s3 / hdfs

### 使用 Mesos Master 的 URL

Mesos 的 Master URL 以 mesos://host:5050 形式表示 single-master Mesos 集群，或者 mesos://zk://host1:2181,host2:2181,host3:2181/mesos 为一个 multi-master Mesos 集群使用 ZooKeeper。

### 客户端模式

在客户端模式下，Spark Mesos 框架直接在客户端计算机上启动，并等待驱动程序输出。

驱动程序需要在 spark-env.sh 中进行一些配置才能与 Mesos 正常交互：

1. 在 spark-env.sh 中设置一些环境变量：
  1. export MESOS_NATIVE_JAVA_LIBRARY=<path to libmesos.so> 的路径。此路径通常为 < prefix>/lib/libmesos.so，其中前缀默认为 / usr/local。请参阅上面的 Mesos 安装说明。在 Mac OS X 上，库称为 libmesos.dylib，而不是 libmesos.so。
  2. export SPARK_EXECUTOR_URI=<URL of spark-2.0.1.tar.gz 上传>。
2. 还将 spark.executor.uri 设置为 <URL of spark-2.0.1.tar.gz>。

现在，当针对集群启动 Spark 应用程序时，在创建 SparkContext 时传递一个 mesos:// URL 作为主服务器。例如：

**Scala**

```scala
val conf = new SparkConf()
  .setMaster("mesos://HOST:5050")
  .setAppName("My app")
  .set("spark.executor.uri", "<path to spark-2.0.1.tar.gz uploaded above>")
val sc = new SparkContext(conf)
```

（您还可以在 conf/spark-defaults.conf 文件中使用 spark-submit 并配置 spark.executor.uri。）
运行 shell 时，spark.executor.uri 参数从 SPARK_EXECUTOR_URI 继承，因此不需要作为系统属性冗余传递。

```bash
./bin/spark-shell --master mesos://host:5050
```

### 集群模式

Spark on Mesos 还支持集群模式，其中驱动程序在集群中启动，客户端可以从 Mesos Web UI 中查找驱动程序的结果。

要使用集群模式，必须通过 sbin / start-mesos-dispatcher.sh 脚本启动集群中的 MesosClusterDispatcher，并传递 Mesos 主 URL（例如：mesos://host:5050）。这将启动 MesosClusterDispatcher 作为在主机上运行的守护程序。

如果你喜欢用 Marathon 运行 MesosClusterDispatcher，你需要在前台运行 MesosClusterDispatcher（即：bin/spark-class org.apache.spark.deploy.mesos.MesosClusterDispatcher）。请注意，MesosClusterDispatcher 尚不支持 HA 的多个实例。

MesosClusterDispatcher 还支持将恢复状态写入 Zookeeper。这将允许 MesosClusterDispatcher 能够在重新启动时恢复所有提交和运行的容器。为了启用此恢复模式，您可以通过配置 spark.deploy.recoveryMode 和相关的 spark.deploy.zookeeper.* 配置在 spark-env 中设置 SPARK_DAEMON_JAVA_OPTS。有关这些配置的更多信息，请参阅配置 (doc)[configurations.html#deploy]。

从客户端，您可以通过运行 spark-submit 并指定到 MesosClusterDispatcher 的 URL（例如：mesos://dispatcher:7077）的主 URL 将作业提交到 Mesos 集群。您可以在 Spark 集群 Web UI 上查看驱动程序状态。

例如：

```bash
./bin/spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://207.184.161.138:7077 \
  --deploy-mode cluster \
  --supervise \
  --executor-memory 20G \
  --total-executor-cores 100 \
  http://path/to/examples.jar \
  1000
```

请注意，传递给 spark-submit 的 jar 或 python 文件应该是 Mesos 从设备可达的 URI，因为 Spark 驱动程序不会自动上传本地 jar。

***

## Mesos 启动模式

Spark 可以在两种模式下运行 Mesos："粗粒度"（默认）和 "细粒度"（已弃用）。

### 粗粒度

在 "粗粒度" 模式下，每个 Spark 执行程序作为单个 Mesos 任务运行。 Spark 执行程序的大小根据以下配置变量确定：

1. 执行器内存：spark.executor.memory
2. 执行器核心：spark.executor.cores
3. 执行器数：spark.cores.max / spark.executor.cores

有关详细信息和默认值，请参阅 Spark Configuration 。

应用程序启动时，热切地启动执行器，直到达到 spark.cores.max。如果不设置 spark.cores.max，Spark 应用程序将保留 Mesos 提供给它的所有资源，因此我们当然敦促您在任何类型的多租户集群中设置此变量，包括运行多个并发 Spark 应用程序。

调度程序将启动执行者循环提供 Mesos 给它，但没有传播保证，因为 Mesos 不提供这样的保证提供流。
粗粒度模式的好处是启动开销低得多，但是在应用程序的整个持续时间内保留 Mesos 资源的代价。要配置作业以动态调整其资源要求，请查看 Dynamic Allocation 。

### 细粒度（已弃用）

> 注意: 细粒度模式自 Spark 2.0.0 起已弃用。考虑使用 Dynamic Allocation 有一些好处。有关完整说明，请参阅 SPARK-11857

在 "细粒度" 模式下，Spark 执行程序中的每个 Spark 任务作为单独的 Mesos 任务运行。这允许 Spark（和其他框架）的多个实例以非常精细的粒度共享核心，其中每个应用程序随着其上升和下降而获得更多或更少的核心，但是它在启动每个任务时带来额外的开销。此模式可能不适合低延迟要求，如交互式查询或服务 Web 请求。

请注意，虽然细粒度的 Spark 任务会在核心终止时放弃核心，但它们不会放弃内存，因为 JVM 不会将内存回馈给操作系统。执行器在空闲时也不会终止。

要以细粒度模式运行，请在 SparkConf 中将 spark.mesos.coarse 属性设置为 false：


```scala
conf.set("spark.mesos.coarse", "false")
```

您还可以使用 spark.mesos.constraints 在 Mesos 资源提供上设置基于属性的约束。默认情况下，所有资源优惠都将被接受。

```scala
conf.set("spark.mesos.constraints", "os:centos7;us-east-1:false")
```

例如，假设 spark.mesos.constraints 设置为 os:centos7;us-east-1:false，那么将检查资源提供以查看它们是否满足这两个约束，然后才会被接受以启动新的执行程序。

***

## Mesos Docker 的支持

Spark 可以通过在你的 SparkConf 中 设置 spark.mesos.executor.docker.image 的属性，从而来使用 Mesos Docker 容器。

所使用的 Docker 镜像必须有一个合适的版本的 Spark 已经是映像的一部分，或者您可以通过通常的方法让 Mesos 下载 Spark。

需要 Mesos 版本 0.20.1 或更高版本。

***

## 独立于 Hadoop 运行

您可以在现有的 Hadoop 集群旁边运行 Spark 和 Mesos，只需将它们作为计算机上的单独服务启动即可。 要从 Spark 访问 Hadoop 数据，需要一个完整的 hdfs:// URL（通常为 hdfs://<namenode>:9000/path，但您可以在 Hadoop Namenode Web UI 上找到正确的 URL）。

此外，还可以在 Mesos 上运行 Hadoop MapReduce，以便在两者之间实现更好的资源隔离和共享。 在这种情况下，Mesos 将作为统一的调度程序，将核心分配给 Hadoop 或 Spark，而不是通过每个节点上的 Linux 调度程序共享资源。 请参考 Hadoop on Mesos 。

在任一情况下，HDFS 与 Hadoop MapReduce 分开运行，而不通过 Mesos 调度。

***

## 通过 Mesos 动态分配资源

Mesos 仅支持使用粗粒度模式的动态分配，这可以基于应用程序的统计信息调整执行器的数量。 有关一般信息，请参阅 Dynamic Resource Allocation 。

要使用的外部 Shuffle 服务是 Mesos Shuffle 服务。 它在 Shuffle 服务之上提供 shuffle 数据清理功能，因为 Mesos 尚不支持通知另一个框架的终止。 要启动它，在所有从节点上运 $SPARK_HOME/sbin/start-mesos-shuffle-service.sh，并将 spark.shuffle.service.enabled 设置为 true。

这也可以通过 Marathon，使用唯一的主机约束和以下命令实现：bin/spark-class org.apache.spark.deploy.mesos.MesosExternalShuffleService。

***

## 配置

有关 Spark 配置的信息，请参阅 configuration page 。 以下配置特定于 Mesos 上的 Spark。

### Spark 属性

| 属性名称 | 默认值 | 含义 |
| ------------- | ------------- | ------------- |
| spark.mesos.coarse | true | 如果设置为 true，则以 "粗粒度" 共享模式在 Mesos 集群上运行，其中 Spark 在每台计算机上获取一个长期存在的 Mesos 任务。如果设置为 false，则以 "细粒度" 共享模式在 Mesos 集群上运行，其中每个 Spark 任务创建一个 Mesos 任务。'Mesos Run Modes' 中的详细信息。 |
| spark.mesos.extra.cores | 0 |  设置执行程序公布的额外核心数。这不会导致分配更多的内核。它代替意味着执行器将 "假装" 它有更多的核心，以便驱动程序将发送更多的任务。使用此来增加并行度。 此设置仅用于 Mesos 粗粒度模式。 |
| spark.mesos.mesosExecutor.cores | 1.0 |  （仅限细粒度模式）给每个 Mesos 执行器的内核数。这不包括用于运行 Spark 任务的核心。换句话说，即使没有运行 Spark 任务，每个 Mesos 执行器将占用这里配置的内核数。 该值可以是浮点数。 |
| spark.mesos.executor.docker.image  | (none) | 设置 Spark 执行器将运行的 docker 映像的名称。所选映像必须安装 Spark，以及兼容版本的 Mesos 库。Spark 在图像中的安装路径可以通过 spark.mesos.executor.home 来指定; 可以使用 spark.executorEnv.MESOS_NATIVE_JAVA_LIBRARY 指定 Mesos 库的安装路径。 |
| spark.mesos.executor.docker.volumes | (none) | 设置要装入到 Docker 镜像中的卷列表，这是使用 spark.mesos.executor.docker.image 设置的。此属性的格式是以逗号分隔的映射列表，后面的形式传递到 docker run -v。 这是他们采取的形式： [host_path:]container_path\[:ro\|:rw\] |
| spark.mesos.executor.docker.portmaps | (none) | 设置由 Docker 镜像公开的入站端口的列表，这是使用 spark.mesos.executor.docker.image 设置的。此属性的格式是以逗号分隔的映射列表，格式如下：host_port:container_port\[:tcp\|:udp\] |
| spark.mesos.executor.home | driver sideSPARK_HOME |  在 Mesos 中的执行器上设置 Spark 安装目录。默认情况下，执行器将只使用驱动程序的 Spark 主目录，它们可能不可见。请注意，这只有当 Spark 二进制包没有通过 spark.executor.uri 指定时才是相关的。 |
| spark.mesos.executor.memoryOverhead | executor memory * 0.10, with minimum of 384 | 以每个执行程序分配的额外内存量（以 MB 为单位）。默认情况下，开销将大于 spark.executor.memory 的 384 或 10％。如果设置，最终开销将是此值。 |
| spark.mesos.uris | (none) | 当驱动程序或执行程序由 Mesos 启动时，要下载到沙箱的 URI 的逗号分隔列表。这适用于粗粒度和细粒度模式。 |
| spark.mesos.principal | (none) | 设置 Spark 框架将用来与 Mesos 进行身份验证的主体。 |
| spark.mesos.secret | (none) | 设置 Spark 框架将用来与 Mesos 进行身份验证的机密。 |
| spark.mesos.role | * | 设置这个 Spark 框架对 Mesos 的作用。角色在 Mesos 中用于预留和资源权重共享。 |
| spark.mesos.constraints | (none) | 基于属性的约束对 mesos 资源提供。 默认情况下，所有资源优惠都将被接受。有关属性的更多信息，请参阅 Mesos Attributes & Resources 。标量约束与 "小于等于" 语义匹配，即约束中的值必须小于或等于资源提议中的值。范围约束与 "包含" 语义匹配，即约束中的值必须在资源提议的值内。集合约束与语义的 "子集" 匹配，即约束中的值必须是资源提供的值的子集。文本约束与 "相等" 语义匹配，即约束中的值必须完全等于资源提议的值。如果没有作为约束的一部分存在的值，则将接受具有相应属性的任何报价（没有值检查）。 |
| spark.mesos.driver.webui.url | (none) | 设置 Spark Mesos 驱动程序 Web UI URL 以与框架交互。如果取消设置，它将指向 Spark 的内部 Web UI。 |
| spark.mesos.dispatcher.webui.url | (none) | 设置 Spark Mesos 分派器 Web UI URL 以与框架交互。如果取消设置，它将指向 Spark 的内部 Web UI。 |

***

## 故障排查和调试

在调试中可以看的地方

- Mesos Master 的端口：5050
  - Slaves 应该出现在 Slavas 那一栏
  - Spark 应用应该出现在框架那一栏
  - 任务应该出现在在一个框架的详情
  - 检查失败任务沙箱的输出和错误
- Mesos 的日志
  - Master 和 Slave 的日志默认在： /var/log/mesos  目录

常见的陷阱：

- Spark 装配不可达、不可访问
  - Slave 必须可以从你给的 http://, hdfs:// or s3n:// URL 地址下载到 Spark 的二进制包
- 防火墙拦截通讯
  - 检查信息是否是连接失败。
  - 临时禁用防火墙来调试，然后戳出适当的漏洞
