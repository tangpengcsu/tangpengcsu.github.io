---
layout: post
title: Spark on YARN
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


## 运行 Spark on YARN

支持在 [YARN (Hadoop NextGen)](http://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/YARN.html "YARN (Hadoop NextGen") 上运行是在 Spark 0.6.0 版本中加入到 Spark 中的，并且在后续的版本中得到改进的。

<!-- more -->

### 启动 Spark on YARN

确保 HADOOP_CONF_DIR 或者 YARN_CONF_DIR 指向包含 Hadoop 集群的（客户端）配置文件的目录。这些配置被用于写入 HDFS 并连接到 YARN ResourceManager 。此目录中包含的配置将被分发到 YARN 集群，以便应用程序 (application) 使用的所有的所有容器 ( containers ) 都使用相同的配置。如果配置引用了 Java 系统属性或者未由 YARN 管理的环境变量，则还应在 Spark 应用程序的配置（驱动程序 (driver)，执行程序 (executors)，和在客户端模式下运行时的 AM ）。

有两种部署模式可以用于在 YARN 上启动 Spark 应用程序。在集群模式下， Spark 驱动程序 (Spark driver) 运行在集群上由 YARN 管理的应用程序主进程 (master process) 内，并且客户端可以在初始化应用程序后离开。在客户端模式下，驱动程序在客户端进程中运行，并且应用程序主服务器仅用于从 YARN 请求资源。

与 [Spark 独立模式](/2016/12/24/spark-running-on-yarn/ "/2016/12/24/spark-running-on-yarn/") 和 [Mesos](/2016/12/24/spark-running-on-mesos/ "/2016/12/24/spark-running-on-mesos/") 模式不同，在这两种模式中，master 的地址在 --master 参数中指定，在 YARN 模式下， ResourceManager 的地址从 Hadoop 配置中选取。因此， --master 参数是 yarn 。

在集群模式下启动 Spark 应用程序：

```bash
$ ./bin/spark-submit --class path.to.your.Class \
      --master yarn \
      --deploy-mode cluster \
      [options] <app jar> [app options]
```

例如：

```bash
$ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
    --master yarn \
    --deploy-mode cluster \
    --driver-memory 4g \
    --executor-memory 2g \
    --executor-cores 1 \
    --queue thequeue \
    lib/spark-examples*.jar \
    10
```

以上启动一个 YARN 客户端程序，启动默认的主应用程序（Application Master）。然后 SparkPi 将作为 Application Master 的子进程运行。客户端将定期轮询 Application Master 以获取状态的更新并在控制台中显示它们。一旦您的应用程序完成运行后，客户端将退出。请参阅下面的 "调试您的应用程序 (Debugging your Application)" 部分，了解如何查看驱动程序 ( driver ) 和 执行程序日志 ( executor logs )。

要在客户端模式下启动 Spark 应用程序，请执行相同的操作，但是将 client 替换 cluster 。下面展示了如何在客户端模式下运行 spark-shell ：

```bash
$ ./bin/spark-shell --master yarn \
    --deploy-mode client
```

添加其他的 JARs

在集群模式下，驱动程序（driver）在与客户端不同的机器上运行，因此 SparkContext.addJar 将不会立即使用客户端本地的文件运行。要使客户端上的文件可用于 SparkContext.addJar ，请在启动命令中使用 --jars 选项来包含这些文件。

```bash
$ ./bin/spark-submit --class my.main.Class \
    --master yarn \
    --deploy-mode cluster \
    --jars my-other-jar.jar,my-other-other-jar.jar \
    my-main-jar.jar \
    app_arg1 app_arg2
```

***

## 准备

在 YARN 上运行 Spark 需要使用 YARN 支持构建的二进制分布式的 Spark （a binary distribution of Spark）。二进制分布式（binary distributions）可以从项目网站的 下载页面 下载。要自己构建 Spark ，请参考 构建 Spark 。

要使 Spark 运行时 jars 可以从 YARN 端访问，您可以指定 spark.yarn.archive 或者 spark.yarn.jars 。有关详细的信息，请参阅 Spark 属性 。如果既没有指定 spark.yarn.archive 也没有指定 spark.yarn.jars ，Spark 将在 $SPARK_HOME/jars 目录下创建一个包含所有 jar 的 zip 文件，并将其上传到分布式缓存（distributed cache）中。

***

## Spark on YARN 配置

对于 Spark on YARN 和其他的部署模式，大多数的配置是相同的。有关这些的更多信息，请参阅 配置页面 。这些是特定于 YARN 上的 Spark 的配置。

***

## 调试您的应用（Debugging your Application）

在 YARN 术语中，执行者（executors）和应用程序（application） masters 在 "容器（containers）" 中运行。在应用程序执行完成后，YARN 提供两种模式处理容器日志（container logs）。如果启用日志聚合（aggregation）（使用 yarn.log-aggregation-enable 配置），容器日志（container logs）将复制到 HDFS 并在本地计算机上删除。可以使用 yarn logs 命令从集群中的任何位置查看这些日志。

```bash
yarn logs -applicationId <app ID>
```

上述命令将打印给定应用程序中所有容器（containers）的全部日志文件内容。你还可以使用 HDFS shell 或者 API 直接在 HDFS 中查看容器日志文件（container log files）。可以通过查看您的 YARN 配置（yarn.nodemanager.remote-app-log-dir 和 yarn.nodemanager.remote-app-log-dir-suffix）找到它们所在的目录。日志还可以在 Spark Web UI 的 "执行程序（Executors）" 选项卡下找到。您需要同时运行 Spark 历史记录服务器（Spark history server） 和 MapReduce 历史记录服务器（MapReduce history server），并在 yarn-site.xml 文件中正确配置 yarn.log.server.url 。 Spark 历史记录服务器 UI 上的日志将重定向您到 MapReduce 历史记录服务器以显示聚合日志（aggregated logs）。

当未启用日志聚合时，日志将在每台计算机上的本地保留在 YARN_APP_LOGS_DIR 目录下，通常配置为 /tmp/logs 或者 $HADOOP_HOME/logs/userlogs ，具体取决于 Hadoop 版本和安装。查看容器（container）的日志需要转到包含它们的主机并在此目录中查看它们。子目录根据应用程序 ID （application ID）和 容器 ID （container ID）组织日志文件。日志还可以在 Spark Web UI 的 "执行程序（Executors）" 选项卡下找到，并且不需要运行 MapReduce 历史记录服务器。

要查看每个容器的启动环境，请将 yarn.nodemanager.delete.debug-delay-sec 增加到一个较大的值（例如 36000），然后通过 yarn.nodemanager.local-dirs 访问应用程序缓存，在容器启动的节点上。此目录包含启动脚本（launch script）， JARs ，和用于启动每个容器的所有的环境变量。这个过程对于调试 classpath 问题特别有用。（请注意，启用此功能需要集群设置的管理员权限并且还需要重新启动所有的节点 (all node managers)，因此这不适用于托管集群）。

要为 application master 或者 executors 使用自定义的 log4j 配置，请选择以下选项：

1. 使用 spark-submit 上传一个自定义的 log4j.properties ，通过将 spark-submit 添加到要与应用程序一起上传的文件的 --files 列表中。
2. 添加 -Dlog4j.configuration=<配置文件的位置（location of configuration file）> 到 spark.driver.extraJavaOptions （对于驱动程序 (for the driver)）或者 spark.executor.extraJavaOptions （对于执行者 (for executors)）。请注意，如果使用文件，文件：协议（protocol ）应该被显式提供，并且该文件需要在所有节点的本地存在。
3. 更新 $SPARK_CONF_DIR/log4j.properties 文件，并且它将与其他配置一起自动上传。请注意，如果指定了多个选项，其他 2 个选项的优先级高于此选项。

请注意，对于第一个选项，执行器（executors）和 主应用程序（application master）将共享相同的 log4j 配置，这当它们在同一个节点上运行的时候，可能会导致问题（例如，试图写入相同的日志文件）。

如果你需要引用正确的位置将日志文件放在 YARN 中，以便 YARN 可以正确显示和聚合它们，请在您的 log4j.properties 中使用 spark.yarn.app.container.log.dir 。例如，log4j.appender.file_appender.File=${spark.yarn.app.container.log.dir}/spark.log 。对于流应用程序（streaming applications），配置 RollingFileAppender 并将文件位置设置为 YARN 的日志目录将避免由于大型日志文件导致的磁盘溢出，并且可以使用 YARN 的日志实用程序（YARN’s log utility）访问日志。

要为主应用程序（application master）和 执行器（executors），请更新 $SPARK_CONF_DIR/metrics.properties 文件。它将自动与其他配置一起上传，因此您不需要使用 --files 手动指定它。

***

## Spark 属性（Properties）

| 属性名称 | 默认 | 含义 |
| ------------- | ------------- | ------------- |
| spark.yarn.am.memory | 512m | 在客户端模式下用于 YARN Application Master 的内存量，与 JVM 内存字符串格式相同（例如，512m，2g）。在集群模式下，请改用 spark.driver.memory 。使用小写字母尾后缀，例如，k，m，g，t 和 p，分别表示 kibi- ，mebi- ，gibi- ，tebi- ，和 pebibytes 。
| spark.driver.memory | 1g | 用于驱动程序进程（driver process）的内存量，即初始化 SparkContext 的位置。（例如 1g，2g）。注意：在客户端模式下，不能通过 SparkConf 直接在应用程序中设置此配置，因为驱动程序 JVM 已在此时启动。相反，请通过 --driver-memory 命令选项或者在默认属性文件中进行设置。|
| spark.driver.cores | 1 | 驱动程序在 YARN 集群模式下使用的核数。由于驱动程序在集群模式下在与 YARN Application Master 相同的 JVM 中运行，因此还会控制 YARN Application Master 使用的核数。在客户端模式下，使用 spark.yarn.am.cores 来控制 YARN Application Master 使用的核数。 |
| spark.yarn.am.cores | 1 | 在客户端模式下用于 YARN Application Master 的核数。在集群模式下，请改用 spark.driver.cores 。 |
| spark.yarn.am.waitTime | 100s | 在集群模式下，YARN Application Master 等待 SparkContext 初始化的时间。在客户端模式下， YARN Application Master 等待驱动程序（driver）连接到它的时间。 |
| spark.yarn.submit.file.replication  | HDFS 默认的备份数（通常为 3）| 用于应用程序上传到 HDFS 的文件的 HDFS 副本级别。这些包括诸如 Spark jar ，app jar， 和任何分布式缓存文件 / 归档之类的东西。 |
| spark.yarn.stagingDir | 文件系统中当前用户的主目录 | 提交应用程序时使用的临时目录。 |
| spark.yarn.preserve.staging.files | false | 设置为 true 以便在作业结束保留暂存文件（Spark jar， app jar，分布式缓存文件），而不是删除它们。 |
| spark.yarn.scheduler.heartbeat.interval-ms | 3000 | Spark application master 心跳到 YARN ResourceManager 中的间隔（以毫秒为单位）。该值的上限为到期时间间隔的 YARN 配置值的一半，即 yarn.am.liveness-monitor.expiry-interval-ms 。 |
| spark.yarn.scheduler.initial-allocation.interval | 200ms | 当存在未决容器分配请求时， Spark application master 心跳到 YARN ResourceManager 的初始间隔。它不应大于 spark.yarn.scheduler.heartbeat.interval-ms 。如果挂起的容器仍然存在，直到达到 spark.yarn.scheduler.heartbeat.interval-ms ，则连续的心跳的分配间隔将加倍。 |
| spark.yarn.max.executor.failures | numExecutors * 2, 最小值为 3 | 在应用程序失败（failing the application）之前，执行器失败（executor failures）的最大数量。 |
| spark.yarn.historyServer.address | （无） | Spark 历史记录服务器（history server）的地址，例如 host.com:18080 。地址不应包含 scheme （http://）。默认为未设置，因为历史记录服务器是可选服务。当 Spark 应用程序完成将应用程序从 ResourceManager UI 链接到 Spark 历史记录服务器 UI 时，此地址将提供给 YARN ResourceManager 。对于此属性， YARN 属性可用作变量，这些属性在运行时由 Spark 替换。例如，如果 Spark 历史记录服务器在与 YARN ResourceManager 相同的节点上运行，则可以将其设置为 $\{hadoopconf-yarn.resourcemanager.hostname\}:18080 。 |
| spark.yarn.dist.archives  | （无） | 逗号分隔的要提取到每个执行器（executor）的工作目录（working directory）中的归档列表。 |
| spark.yarn.dist.files | （无） | 要放到每个执行器（executor）的工作目录（working directory）中的以逗号分隔的文件列表。 |
spark.yarn.dist.jars | （无） |  要放到每个执行器（executor）的工作目录（working directory）中的以逗号分隔的 jar 文件列表。 |
| spark.executor.cores  | 在 YARN 模式下是 1 ，在独立模式下 worker 机器上的所有的可用的核。 | 每个执行器（executor）上使用的核数。仅适用于 YARN 和独立（standalone ）模式。 |
| spark.executor.instances  | 2 | 静态分配的执行器（executor）数量。使用 spark.dynamicAllocation.enabled ，初始的执行器（executor）集至少会是这么大。spark.executor.memory   1g  每个执行器（executor）进程使用的内存量（例如 2g ，8g）。 |
| spark.yarn.executor.memoryOverhead | 执行器内存（executorMemory）* 0.10 ，最小值为 384 | 要为每个执行器（executor）分配的堆外（off-heap）内存量（以兆字节为单位）。这是内存，例如 VM 开销，内部字符串，其他本机开销等。这往往随着执行器（executor）大小（通常为 6-10%）增长。 |
| spark.yarn.driver.memoryOverhead | 驱动程序内存（driverMemory）* 0.10，最小值为 384 | 在集群模式下为每个驱动程序（driver）分配的堆外（off-heap）内存量（以兆字节为单位）。这是内存，例如 VM 开销，内部字符串，其他本机开销等。这往往随着容器（container）大小（通常为 6- 10%）增长。 |
| spark.yarn.am.memoryOverhead | AM 内存（AM Memory） * 0.10，最小 384 | 与 spark.yarn.driver.memoryOverhead 相同，但是适用于客户端模式下的 YARN Application Master 。 |
| spark.yarn.am.port | （随机） | 被 YARN Application Master 监听的端口。在 YARN 客户端模式下，这用于在网关（gateway）上运行的 Spark 驱动程序（driver）和在 YARN 上运行的 YARN Application Master 之间进行通信。在 YARN 集群模式下，这用于动态执行器（executor）功能，其中它处理从调度程序后端的 kill 。 |
| spark.yarn.queue | 默认（default） | 提交应用程序（application）的 YARN 队列名称。 |
| spark.yarn.jars | （无） | 包含要分发到 YARN 容器（container）的 Spark 代码的库（libraries）列表。默认情况下， Spark on YARN 将使用本地安装的 Spark jar ，但是 Spark jar 也可以在 HDFS 上的一个任何位置都可读的位置。这允许 YARN 将其缓存在节点上，使得它不需要在每次运行应用程序时分发。例如，要指向 HDFS 上的 jars ，请将此配置设置为 hdfs:///some/path 。允许使用 globs 。
| spark.yarn.archive | （无）| 包含所需的 Spark jar 的归档（archive），以分发到 YARN 高速缓存。如果设置，此配置将替换 spark.yarn.jars ，并且归档（archive）在所有的应用程序（application）的容器（container）中使用。归档（archive）应该在其根目录中包含 jar 文件。与以前的选项一样，归档也可以托管在 HDFS 上以加快文件分发速度。 |
| spark.yarn.access.namenodes | （无） | 以逗号分隔的 Spark 应用程序要访问的安全的 HDFS namenodes 列表。例如：spark.yarn.access.namenodes=hdfs://nn1.com:8032,hdfs://nn2.com:8032, webhdfs://nn3.com:50070 。Spark 应用程序必须具有对列出的 namenode 的访问权限，并且 Kerberos 必须正确配置为能够访问它们（在同一领域（realm）或者在可信领域（trusted realm））。 Spark 为每个 namenode 获取安全令牌（security tokens），以便 Spark application 可以访问这些远程的 HDFS 集群。 |
| spark.yarn.appMasterEnv.[EnvironmentVariableName] | （无） | 将由 EnvironmentVariableName 指定的环境变量添加到在 YARN 上启动的 Application Master 进程。用户可以指定其中的多个并设置多个环境变量。在集群模式下，这控制 Spark 驱动程序（driver）的环境，在客户端模式下，它只控制执行器（executor）启动器（launcher）的环境。 |
| spark.yarn.containerLauncherMaxThreads | 25 | 在 YARN Application Master 中用于启动执行器容器（executor containers）的最大线程（thread）数。 |
| spark.yarn.am.extraJavaOptions | （无） | 在客户端模式下传递到 YARN Application Master 的一组额外的 JVM 选项。在集群模式下，请改用 spark.driver.extraJavaOptions 。请注意，使用此选项设置最大堆大小（-Xmx）设置是非法的。最大堆大小设置可以使用 spark.yarn.am.memory 。 |
| spark.yarn.am.extraLibraryPath | （无） | 设置在客户端模式下启动 YARN Application Master 时要使用的特殊库路径（special library path）。
| spark.yarn.maxAppAttempts  | yarn.resourcemanager.am.max-attempts 在 YARN 中 | 将要提交应用程序的最大的尝试次数。它应该不大于 YARN 配置中的全局最大尝试次数。 |
| spark.yarn.am.attemptFailuresValidityInterval | （无） | 定义 AM 故障跟踪（failure tracking）的有效性间隔。如果 AM 已经运行至少所定义的间隔，则 AM 故障计数将被重置。如果未配置此功能，则不启用此功能，并且仅在 Hadoop 2.6 + 版本中支持此功能。 |
| spark.yarn.executor.failuresValidityInterval | （无） | 定义执行器（executor）故障跟踪的有效性间隔。超过有效性间隔的执行器故障（executor failures）将被忽略。 |
| spark.yarn.submit.waitAppCompletion | true | 在 YARN 集群模式下，控制客户端是否等待退出，直到应用程序完成。如果设置为 true ，客户端将保持报告应用程序的状态。否则，客户端进程将在提交后退出。 |
| spark.yarn.am.nodeLabelExpression | （无） | 一个将调度限制节点 AM 集合的 YARN 节点标签表达式。只有大于或等于 2.6 版本的 YARN 版本支持节点标签表达式，因此在对较早版本运行时，此属性将被忽略。 |
| spark.yarn.executor.nodeLabelExpression | （无） | 一个将调度限制节点执行器（executor）集合的 YARN 节点标签表达式。只有大于或等于 2.6 版本的 YARN 版本支持节点标签表达式，因此在对较早版本运行时，此属性将被忽略。
| spark.yarn.tags | （无） | 以逗号分隔的字符串列表，作为 YARN application 标记中显示的 YARN application 标记传递，可用于在查询 YARN apps 时进行过滤（filtering ）。 |
| spark.yarn.keytab | （无） | 包含上面指定的主体（principal）的 keytab 的文件的完整路径。此 keytab 将通过安全分布式缓存（Secure Distributed Cache）复制到运行 YARN Application Master 的节点，以定期更新 login tickets 和 delegation tokens 。（运行也使用 "local" master）。 |
spark.yarn.principal | （无） | 在安全的 HDFS 上运行时用于登录 KDC 的主体（Principal ）。（运行也使用 "local" master）。 |
| spark.yarn.config.gatewayPath | （无） | 在网关主机（gateway host）（启动 Spark application 的 host）上有效的路径，但对于集群中其他节点中相同资源的路径可能不同。结合 spark.yarn.config.replacementPath ，者用于支持具有异构配置的集群，以便 Spark 可以正确启动远程进程。替换路径（replacement path）通常将包含对由 YARN 导出的某些环境变量（以及，因此对于 Spark 容器可见）的引用。例如，如果网关节点（gateway node）在 /disk1/hadoop 上安装了 Hadoop 库，并且 Hadoop 安装的位置由 YARN 作为 HADOOP_HOME 环境变量导出，则将此值设置为 /disk1/hadoop ，将替换路径（replacement path）设置为 $HADOOP_HOME 将确保用于启动远程进程的路径正确引用本地 YARN 配置。 |
| spark.yarn.config.replacementPath | （无）| 查看 spark.yarn.config.gatewayPath 。
| spark.yarn.security.tokens.$\{service\}.enabled | true | 控制是否在启用安全性时检索 非 HDFS 服务的 delegation tokens 。默认情况下，当配置了这些服务时，将检索所有支持的服务的 delegation tokens ，但是如果它以某种方式与正在运行的应用程序冲突，则可以禁用该行为。目前支持的服务有：hive，hbase 。 |

***

## Important notes （要点）

- 核心请求在调度决策中是否得到执行取决于使用的调度程序及其配置方式。
- 在集群模式下，Spark executors 和 Spark dirver 使用的本地目录是为 YARN（Hadoop YARN 配置 yarn.nodemanager.local-dirs）配置的本地目录。如果用户指定 spark.local.dir，它将被忽略。在客户端模式下，Spark executors 将使用为 YARN 配置的本地目录，而 Spark dirver 将使用 spark.local.dir 中定义的目录。这是因为 Spark dirver 不在客户端模式下在 YARN 集群上运行，只有 Spark executors。
- --files 和–archives 选项支持用 # 指定文件名，类似于 Hadoop。 例如，你可以指定：--files localtest.txt#appSees.txt，这会将你在本地命名为 localtest.txt 的文件上传到 HDFS，但这将通过名称 appSees.txt 链接，当你的应用程序在 YARN 上运行时，你应该使用名称 appSees.txt 引用它。
- --jars 选项允许你在集群模式下使用本地文件时运行 SparkContext.addJar 函数。 如果你使用 HDFS，HTTP，HTTPS 或 FTP 文件，则不需要使用它。

***

## Running in a Secure Cluster（在一个安全的集群中运行）

如 security 所讲的，Kerberos 被应用在安全的 Hadoop 集群中去验证与服务和客户端相关联的 principals。 这允许客户端请求这些已验证的服务; 向授权的 principals 授予请求服务的权利。

Hadoop 服务发出 hadoop tokens 去授权访问服务和数据。 客户端必须首先获取它们将要访问的服务的 tokens，当启动应用程序时，将它和应用程序一起发送到 YAYN 集群中。

如果 Spark 应用程序与 HDFS，HBase 和 Hive 进行交互，它必须使用启动应用程序的用户的 Kerberos 凭据获取相关 tokens，也就是说身份将成为已启动的 Spark 应用程序的 principal。

这通常在启动时完成：在安全集群中，Spark 将自动为集群的 HDFS 文件系统获取 tokens，也可能为 HBase 和 Hive 获取。

如果 HBase 在类路径中，HBase 配置声明应用程序是安全的（即 hbase-site.xml 将 hbase.security.authentication 设置为 kerberos），并且 spark.yarn.security.tokens.hbase.enabled 未设置为 false，HBase tokens 将被获得。

类似地，如果 Hive 在类路径上，其配置包括元数据存储的 URI（hive.metastore.uris），并且 spark.yarn.security.tokens.hive.enabled 未设置为 false，则将获得 Hive 令牌。

如果应用程序需要与其他安全 HDFS 集群交互，则在启动时必须显式请求访问这些集群所需的 tokens。 这是通过将它们列在

spark.yarn.access.namenodes 属性中来实现的。
spark.yarn.access.namenodes hdfs://ireland.example.org:8020/,hdfs://frankfurt.example.org:8020/

***

##  Launching your application with Apache Oozie（用 Apache Oozie 运行程序）

Apache Oozie 可以将启动 Spark 应用程序作为工作流的一部分。在安全集群中，启动的应用程序将需要相关的 tokens 来访问集群的服务。如果 Spark 使用 keytab 启动，这是自动的。但是，如果 Spark 在没有 keytab 的情况下启动，则设置安全性的责任必须移交给 Oozie。
有关配置 Oozie 以获取安全集群和获取作业凭据的详细信息，请参阅 Oozie web site 上特定版本文档的 "Authentication" 部分。

对于 Spark 应用程序，必须设置 Oozie 工作流以使 Oozie 请求应用程序需要的所有 tokens，包括：

- YARN 资源管理器。
- 本地 HDFS 文件系统。
- 任何用作 I / O 的源或目标的远程 HDFS 文件系统。
- Hive - 如果使用。
- HBase - 如果使用。
- YARN 时间轴服务器，如果应用程序与此交互。

为了避免 Spark 尝试 - 然后失败 - 获取 Hive，HBase 和远程 HDFS 令牌，必须将 Spark 配置收集这些服务 tokens 的选项设置为禁用。
Spark 配置必须包含以下行：

```ini
spark.yarn.security.tokens.hive.enabled false spark.yarn.security.tokens.hbase.enabled false
```

必须取消设置配置选项 spark.yarn.access.namenodes。

***

## Troubleshooting Kerberos（Kerberos 错误排查）


调试 Hadoop / Kerberos 问题可能是 "困难的"。 一个有用的技术是通过设置 HADOOP_JAAS_DEBUG 环境变量在 Hadoop 中启用对 Kerberos 操作的额外记录。

```bash
bash export HADOOP_JAAS_DEBUG=true
```

JDK 类可以配置为通过系统属性 `sun.security.krb5.debug` 和 `sun.security.spnego.debug = true` 启用对 Kerberos 和 SPNEGO / REST 认证的额外日志记录。

```bash
-Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true
```

所有这些选项都可以在 Application Master 中启用：

```bash
spark.yarn.appMasterEnv.HADOOP_JAAS_DEBUG true spark.yarn.am.extraJavaOptions -Dsun.security.krb5.debug=true -Dsun.security.spnego.debug=true
```

最后，如果 org.apache.spark.deploy.yarn.Client 的日志级别设置为 DEBUG，日志将包括获取的所有 tokens 的列表，以及它们的到期详细信息
