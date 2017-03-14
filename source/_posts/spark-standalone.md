---
layout: post
title: Spark Standalone 模式
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


Spark 不仅可以运行在 Mesos 或者 Yarn 上，而且还提供独立部署模式。可以手动启动一个 master 和 多个 worker，或选用我们提供的 [脚本](http://spark.apache.org/docs/latest/spark-standalone.html#cluster-launch-scripts "启动脚本") 来启动 standalone 集群。

<!-- more -->

***

## 安装 Spark  standalone 集群

独立安装 Spark 集群，只需要把编译好的版本部署在每个节点上，然后启动，或者也可以自己 [编译指定版本](http://spark.apache.org/docs/latest/building-spark.html "编译指定版本")。

***

## 手动启动集群

通过下面脚本启动集群的主节点:

```bash
./sbin/start-master.sh
```

通过上面脚本启动 master ，master 会通过 spark://host:portURL 生成链接，你可以手动访问，或者通过 master 的 SparkContext 来创建访问，或者通过浏览器访问 Spark 的 Web UI , 默认情况： http://localhost:8080

同样，你可以启动一个和多个 worker ，并通过 主节点把他们连接到一起:

```bash
./sbin/start-slave.sh <master-spark-URL>
```

一旦你已经启动成功了一个 worker , 即可访问 master 的 WEB UI（http://localhost:8080，默认），不仅  可以看到新添加的节点，还可以看到新的节点列表、节点的 CPU 数以及内存。

最后, 下面的配置参数可以传递给 master 和 worker

| 参数 | 说明 |
| ------------- | ------------- |
| -h HOST， --host HOST | 监听的主机名 |
| -i HOST， --ip HOST | 同上，不建议使用 |
| -p PORT， --port PORT | 监听的服务的端口（master 默认是 7077，worker 随机） |
| --webui-port PORT | web UI 的端口 (master 默认是 8080，worker 默认是 8081) |
| -c CORES， --cores CORES | Spark 应用程序可以使用的 CPU 核数（默认是所有可用）；这个选项仅在 worker 上可用 |
| -m MEM， --memory MEM | Spark 应用程序可以使用的内存数（默认情况是你的机器内存数减去 1g）；这个选项仅在 worker 上可用 |
| -d DIR， --work-dir DIR | 用于暂存空间和工作输出日志的目录（默认是 SPARK_HOME/work）；这个选项仅在 worker 上可用 |
| --properties-file FILE | 自定义的 Spark 配置文件的加载目录（默认是 conf/spark-defaults.conf） |

***

## 脚本启动集群

为了用启动脚本启动 Spark 独立集群，你应该在你的 Spark 目录下建立一个名为 conf/slaves 的文件，这个文件必须包含所有你要启动的 Spark worker 所在机器的主机名，一行一个。如果 conf/slaves 不存在，启动脚本默认为单个机器（localhost），这台机器对于测试是有用的。注意，master 机器通过 ssh 访问所有的 worker。在默认情况下，SSH 是并行运行，需要设置无密码（采用私有密钥）的访问。 如果你没有设置为无密码访问，你可以设置环境变量 SPARK_SSH_FOREGROUND，为每个 worker 提供密码。

一旦你设置了这个文件，你就可以通过下面的 shell 脚本启动或者停止你的集群。

- sbin/start-master.sh：在机器上启动一个 master 实例
- sbin/start-slaves.sh：在每台机器上启动一个 slave 实例
- sbin/start-slave.sh - Starts a slave instance on the machine the script is executed on.
- sbin/start-all.sh：同时启动一个 master 实例和所有 slave 实例
- sbin/stop-master.sh：停止 master 实例
- sbin/stop-slaves.sh：停止所有 slave 实例
- sbin/stop-all.sh：停止 master 实例和所有 slave 实例

> 注意，这些脚本必须在你的 Spark
master 运行的机器上执行，而不是在你的本地机器上面。

你可以在 conf/spark-env.sh 中设置环境变量进一步配置集群。利用 conf/spark-env.sh.template 创建这个文件，然后将它复制到所有的 worker 机器上使设置有效。下面的设置可以起作用：

| 环境变量 | 说明 |
| ------------- | ------------- |
| SPARK_MASTER_HOST | 绑定 master 到一个指定的 ip 地址 |
| SPARK_MASTER_PORT | 不同的端口上启动主（默认值：7077） |
| SPARK_MASTER_WEBUI_PORT | master web UI 的端口（默认是 8080） |
| SPARK_MASTER_OPTS | 应用到 master 的配置属性，格式是 "-Dx=y"（默认是 none），查看下面的表格的选项以组成一个可能的列表 |
| SPARK_LOCAL_DIRS |  Spark 中暂存空间的目录。包括 map 的输出文件和存储在磁盘上的 RDDs(including map output files and RDDs that get stored on disk)。这必须在一个快速的、可靠的系统本地磁盘上。它可以是一个逗号分隔的列表，代表不同磁盘的多个目录 |
| SPARK_WORKER_CORES | Spark 应用程序可以用到的核心数（默认是所有可用） |
| SPARK_WORKER_MEMORY | Spark 应用程序用到的内存总数（默认是内存总数减去 1G）。注意，每个应用程序个体的内存通过 spark.executor.memory 设置 |
| SPARK_WORKER_PORT | 在指定的端口上启动 Spark worker(默认是随机) |
| SPARK_WORKER_WEBUI_PORT | worker UI 的端口（默认是 8081） |
| SPARK_WORKER_DIR | Spark worker 运行目录，该目录包括日志和暂存空间（默认是 SPARK_HOME/work） |
| SPARK_WORKER_OPTS | 应用到 worker 的配置属性，格式是 "-Dx=y"（默认是 none），查看下面表格的选项以组成一个可能的列表 |
| SPARK_DAEMON_MEMORY | 分配给 Spark master 和 worker 守护进程的内存（默认是 512m） |
| SPARK_DAEMON_JAVA_OPTS | Spark master 和 worker 守护进程的 JVM 选项，格式是 "-Dx=y"（默认为 none） |
| SPARK_PUBLIC_DNS | Spark master 和 worker 公共的 DNS 名（默认是 none） |

注意，启动脚本还不支持 windows。为了在 windows 上启动 Spark 集群，需要手动启动 master 和 workers。

SPARK_MASTER_OPTS 支持以下系统属性：

| 参数 | 默认 | 说明 |
| ------------- | ------------- | ------------- |
| spark.deploy.retainedApplications  | 200 | 展示完成的应用程序的最大数目。老的应用程序会被删除以满足该限制 |
| spark.deploy.retainedDrivers | 200 | 展示完成的 drivers 的最大数目。老的应用程序会被删除以满足该限制 |
| spark.deploy.spreadOut | true | 这个选项控制独立的集群管理器是应该跨节点传递应用程序还是应努力将程序整合到尽可能少的节点上。在 HDFS 中，传递程序是数据本地化更好的选择，但是，对于计算密集型的负载，整合会更有效率。 |
| spark.deploy.defaultCores | （infinite） | 在 Spark 独立模式下，给应用程序的默认核数（如果没有设置 spark.cores.max）。如果没有设置，应用程序总数获得所有可用的核，除非设置了 spark.cores.max。在共享集群上设置较低的核数，可用防止用户默认抓住整个集群 |
| spark.deploy.maxExecutorRetries | 10 | 限制上的独立的集群管理器中删除有问题的应用程序之前可能发生的后端到回执行人失败的最大数量。应用程序将永远不会被删除，如果有任何正在运行的执行人。如果一个应用程序的经验超过 spark.deploy.maxExecutorRetries 连续的失败，没有遗嘱执行人成功启动这些失败之间运行，并且应用程序没有运行执行人那么独立的集群管理器将删除该应用程序并将其标记为失败。要禁用此自动删除，设置 spark.deploy.maxExecutorRetries 为 -1 |
| spark.worker.timeout | 60 |   独立部署的 master 认为 worker 失败（没有收到心跳信息）的间隔时间 |

SPARK_WORKER_OPTS 支持以下系统属性：

| 参数 | 默认 | 说明 |
| ------------- | ------------- | ------------- |
| spark.worker.cleanup.enabled | false | 周期性的清空 worker / 应用程序目录。注意，这仅仅影响独立部署模式。不管应用程序是否还在执行，用于程序目录都会被清空 |
| spark.worker.cleanup.interval | 1800（30 分钟） |  在本地机器上，worker 清空老的应用程序工作目录的时间间隔 |
| spark.worker.cleanup.appDataTtl | 7 * 24 * 3600（7 天） | 每个 worker 中应用程序工作目录的保留时间。这个时间依赖于你可用磁盘空间的大小。应用程序日志和 jar 包上传到每个应用程序的工作目录。随着时间的推移，工作目录会很快的填满磁盘空间，特别是如果你运行的作业很频繁 |
| spark.worker.ui.compressedLogFileLengthCacheSize | 100 | For compressed log files, the uncompressed file can only be computed by uncompressing the files. Spark caches the uncompressed file size of compressed log files. This property controls the cache size. |

***

## 提交应用程序到集群

为了简单的运行 Spark 的应用程序，只需要通过 [SparkContext Constructor](http://spark.apache.org/docs/latest/programming-guide.html#initializing-spark "SparkContext Constructor") spark://IP:PORT 这个 URL 。

集群的交互式运行可以通过 Spark shell，运行下面命名:

```bash
./bin/spark-shell --master spark://IP:PORT
```

你也可以通过选项 --total-executor-cores <numCores> 传递参数去控制 spark-shell 的核数

***

## 启动 Spark 应用程序

[spark-submit](/2016/12/24/spark-submitting-applications/ "/2016/12/24/spark-submitting-applications/") 脚本提供了最直接的方式将一个 Spark 应用程序提交到集群。对于独立部署的集群，Spark 目前支持两种部署模式。在 client 模式中，driver 与客户端提交应用程序运行在同一个进程中。然而，在 cluster 模式中，driver 在集群的某个 worker 进程中启动，一旦客户端进程完成了已提交任务，它就会立即退出，并不会等到应用程序完成后再退出。

如果你的应用程序通过 Spark submit 启动，你的应用程序 jar 包将会自动分发到所有的 worker 节点。对于你的应用程序依赖的其它 jar 包，你应该用 --jars 符号指定（如 --jars jar1,jar2）。 更多配置详见 [Spark Configuration](http://spark.apache.org/docs/latest/configuration.html "Spark Configuration.").

另外，如果你的应用程序以非 0 (non-zero) 状态退出，独立集群模式支持重启程序。要支持自动重启，需要向 spark-submit 传递 --supervise 标志。如果你想杀掉一个重复失败的应用程序，你可以使用如下方式：

```bash
./bin/spark-class org.apache.spark.deploy.Client kill <master url> <driver ID>
```

你可以在 Master 的 web UI 上 http://\<master url\>:8080 找到 driver ID。

***

## 资源调度

独立部署的集群模式仅仅支持简单的 FIFO 调度器。然而，为了允许多个并行的用户，你能够控制每个应用程序能用的最大资源数。在默认情况下，它将获得集群的所有核，这只有在某一时刻只允许一个应用程序才有意义。你可以通过 spark.cores.max 在 [SparkConf](http://spark.apache.org/docs/latest/configuration.html#spark-properties "SparkConf") 中设置核数。

```scala
val conf = new SparkConf()
             .setMaster(...)
             .setAppName(...)
             .set("spark.cores.max", "10")
val sc = new SparkContext(conf)
```

另外，你可以在集群的 master 进程中配置 spark.deploy.defaultCores 来改变默认的值。在 conf/spark-env.sh 添加下面的行：

```bash
export SPARK_MASTER_OPTS="-Dspark.deploy.defaultCores=<value>"
```

这在用户没有配置最大核数的共享集群中是有用的。

***

## 监控和日志记录

Spark 的独立模式提供了一个基于 Web 的用户界面来监控集群。Master 和 每个 Worker 都有自己的 web 用户界面，显示集群和作业统计信息。默认情况下你可以在 8080 端口访问 Web UI 的主端口也可以在配置文件中或通过命令行选项进行更改。

此外，对于每个作业详细的日志也会输出每个 worker 目录下（默认情况工作目录 SPARK_HOME/work）。你会看到两个文件为每个作业，stdout 并 stderr 与所有输出它写信给其控制台。

***

## 与 hadoop 的集成

你可以在 Hadoop 上单独启动一个 Spark 服务。若要从 Spark 访问 Hadoop 的数据，只需使用一个 HDFS:// URL（通常是 hdfs://\<namenode\>:9000/path，也可以找到 Hadoop 的 Web UI 找到 Namenode 的 URL）。另外，您也可以单独启动 Spark 集群，而且还有它通过网络进行访问 HDFS， 但这将是比本地磁盘访问速度慢，最好是在一个局域网之内（例如，Hadoop 各项服务或者各个节点在同一个机架上）。

***

## 端口配置和网络安全

Spark  对网络的需求比较高，对网络环境有紧密联系，需要严谨设置防火墙。端口配置列表，请参阅 [安全网页](http://spark.apache.org/docs/latest/security.html#configuring-ports-for-network-security "安全网页")。

***

## 高可用性

默认情况下，独立的调度集群对 worker 失败是有弹性的（在 Spark 本身的范围内是有弹性的，对丢失的工作通过转移它到另外的 worker 来解决）。然而，调度器通过 master 去执行调度决定， 这会造成单点故障：如果 master 死了，新的应用程序就无法创建。为了避免这个，我们有两个高可用的模式。

### 用 ZooKeeper 的备用 master

#### 概述

利用 ZooKeeper 去支持 leader 选举以及一些状态存储，你能够在你的集群中启动多个 master，这些 master 连接到同一个 ZooKeeper 实例上。一个被选为 "leader" ，其它的保持备用模式。如果当前的 leader 死了，另一个 master 将会被选中，恢复老 master 的状态，然后恢复调度。整个的恢复过程大概需要 1 到 2 分钟。注意，这个恢复时间仅仅会影响调度新的应用程序——运行在失败 master 中的 应用程序将不受影响。

了解更多关于如何开始使用的 ZooKeeper [这里](http://zookeeper.apache.org/doc/trunk/zookeeperStarted.html "这里")。

#### 配置

为了开启这个恢复模式，你可以用下面的属性在 spark-env 中设置 SPARK_DAEMON_JAVA_OPTS  spark.deploy.recoveryMode 及相关 spark.deploy.zookeeper.* 的配置。有关这些配置的详细信息，请参阅 [配置文档](/2016/12/24/spark-configuration/#section-4 "/2016/12/24/spark-configuration/#section-4")

可能的陷阱：如果你在集群中有多个 masters，但是没有用 zookeeper 正确的配置这些 masters，这些 masters 不会发现彼此，会认为它们都是 leaders。这将会造成一个不健康的集群状态（因为所有的 master 都会独立的调度）。

#### 细节

zookeeper 集群启动之后，开启高可用是简单的。在相同的 zookeeper 配置（zookeeper URL 和目录）下，在不同的节点上简单地启动多个 master 进程。master 可以随时添加和删除。

为了调度新的应用程序或者添加 worker 到集群，它需要知道当前 leader 的 IP 地址。这可以通过简单的传递一个 master 列表来完成。例如，你可能启动你的 SparkContext 指向 spark://host1:port1,host2:port2。 这将造成你的 SparkContext 同时注册这两个 master -- 如果 host1 死了，这个配置文件将一直是正确的，因为我们将找到新的 leader-host2。

"registering with a Master" 和正常操作之间有重要的区别。当启动时，一个应用程序或者 worker 需要能够发现和注册当前的 leader master。一旦它成功注册，它就在系统中了。如果 错误发生，新的 leader 将会接触所有之前注册的应用程序和 worker，通知他们领导关系的变化，所以它们甚至不需要事先知道新启动的 leader 的存在。

由于这个属性的存在，新的 master 可以在任何时候创建。你唯一需要担心的问题是新的应用程序和 workers 能够发现它并将它注册进来以防它成为 leader master。

### 用本地文件系统做单节点恢复

#### 概述

zookeeper 是生产环境下最好的选择，但是如果你想在 master 死掉后重启它，FILESYSTEM (文件系统) 模式可以解决。当应用程序和 worker 注册，它们拥有足够的状态写入提供的目录，以至于在重启 master 进程时它们能够恢复。

#### 配置

为了开启这个恢复模式，你可以用下面的属性在 spark-env 中设置 SPARK_DAEMON_JAVA_OPTS。

| System property | Meaning |
| ------------- | ------------- |
| spark.deploy.recoveryMode | 设置为 FILESYSTEM 开启单节点恢复模式（默认为 none） |
| spark.deploy.recoveryDirectory | 用来恢复状态的目录 |

#### 细节

- 这个解决方案可以和监控器 / 管理器（如 [monit](https://mmonit.com/monit/ "monit")）相配合，或者仅仅通过重启开启手动恢复。
- 虽然文件系统的恢复似乎比没有做任何恢复要好，但对于特定的开发或实验目的，这种模式可能是次优的。特别是，通过 stop-master.sh 杀掉 master 不会清除它的恢复状态，所以，不管你何时启动一个新的 master，它都将进入恢复模式。这可能使启动时间增加到 1 分钟。
- 虽然它不是官方支持的方式，你也可以创建一个NFS目录作为恢复目录。如果原始的 master节点完全死掉，你可以在不同的节点启动 master，它可以正确的恢复之前注册的所有应用程序和 workers。未来的应用程序会发现这个新的 master。
