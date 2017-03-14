---
layout: post
title: Spark 监控
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


有几种方法可以监视 Spark 应用 : Web UI，metrics 和其他扩展工具。

## Web 接口

每一个 SparkContext 启动一个 web UI，默认情况下使用端口 4040，可以显示关于运行程序的有用信息。这包括 :

- 调度器阶段和任务的列表 　　
- RDD 大小和内存使用的概要信息 　　
- 环境信息　
- 正在运行的程序的信息

您只需打开 *http://\<driver-node\>:4040* 的 web 浏览器就可以访问。如果在同一主机上运行多个 SparkContexts，他们将开始连续绑定到端口 4040（4041、4042、等）。

注意，**默认情况下这些信息仅在有程序的执行时显示**。你可以在启动 Spark 之前修改配置，设置 **spark.eventLog.enabled** 为 **true**。让 Spark 记录并持久化存储 Spark 事件使其可以在 UI 中显示。

<!-- more -->

### 历史信息

如果 Spark 在 Mesos 或者 YARN 上运行，它仍有可能用已存在的程序日志通过 Spark history server（历史信息记录服务）来显示该程序运行时的详细信息。启动命令如下:

```bash
./sbin/start-history-server.sh
```

这个会默认创建一个 web 接口 : *http://\<server-url\>:18080*，显示未完成、完成以及其他尝试的任务信息。

当指定使用一个文件系统提供 class 类（具体见下 spark.history.provider），那么基本的日志存储路径应该在 spark.history.fs.logDirectory 这个配置中指定，并且会有子目录，每个都表示某个程序信息的日志 log。

Spark 任务本身必须配置启用日志，并用相同的、共享的、可写的目录记录他们。例如，如果服务器配置的日志目录为

```
spark.eventLog.enabled true
spark.eventLog.dir hdfs://namenode/shared/spark-logs
```

那么 history server 的配置信息可以如下 :

#### 环境变量

| 环境变量 | 含义 |
| ------------- | ------------- |
| SPARK_DAEMON_MEMORY |	history server 分配内存（默认 : 1g）. |
| SPARK_DAEMON_JAVA_OPTS |	history server JVM 配置（默认 : none）. |
| SPARK_PUBLIC_DNS |	history server 的公用地址. 如果未配置， 可能会使用服务器的内部地址， 导致连接失效 （默认 : none）. |
| SPARK_HISTORY_OPTS |	spark.history.*  history server 的相关配置选项 （默认 : none）. |

> 文件：conf/spark-env.sh

#### Spark 配置选项

| 属性名称 | 默认 | 含义 |
| ------------- | ------------- | ------------- |
| spark.history.provider | org.apache.spark.deploy.history.FsHistoryProvider | 实现了 history backend 的 class 类的名称。目前只有一个实现，Spark 本身提供，用于检测存在文件系统中的程序的日志文件。 |
| spark.history.fs.logDirectory | http://file/tmp/spark-events | 提供历史日志文件存储路径，URL地址（包含可加载的程序日志的目录）。可以配置本地路径 file://，或者 HDFS 路径 hdfs://namenode/shared/spark-logs 或其他 Hadoop API 支持的文件系统。 |
| spark.history.fs.update.interval | 10s | 以秒为单位，更新日志相关信息的时间间隔，更短的时间间隔帮助检测到新的程序更快，但是牺牲更多的服务器负载。一旦更新完成，完成和未完成的程序的都会发生变化。 |
| spark.history.retainedApplications | 50 | 保存在 UI 缓存中的程序数量。如果超过这个上限，那么时间最老的程序将从缓存中移除。如果一个程序未被缓存，它就必须从磁盘加载。 |
| spark.history.ui.maxApplications | Int.MaxValue | 显示在总历史页面中的程序的数量。如果总历史页面未显示，程序 UI 仍可通过访问其 URL 来显示。 |
| spark.history.ui.port | 18080 | UI 端口设置. |
| spark.history.kerberos.enabled | false | 表明 history server 是否应该使用 kerberos 登录。如果历史服务器访问 HDFS 文件安全的 Hadoop 集群，这是必需的。如果设置为 true，将会使用 spark.history.kerberos.principal 和 spark.history.kerberos.keytab 两个配置。 |
| spark.history.kerberos.principal | （none） | Kerberos 凭证名称。 |
| spark.history.kerberos.keytab | （none） |	kerberos keytab 文件位置。 |
| spark.history.ui.acls.enable | false | 指定是否通过 acl 授权限制用户查看程序。如果启用，程序运行时会强制执行访问控制检查，检测程序是否设置 spark.ui.acls.enable，程序所有者总能够查看自己的应用程序和任何指定 spark.ui.view.acls 和 spark.ui.view.acls.groups 的程序。当程序运行也将要授权查看。如果禁用，就没有访问控制检查。 |
| spark.history.fs.cleaner.enabled | false | 是否定期清理历史日志文件。 |
| spark.history.fs.cleaner.interval | 1d | 清理历史日志文件的间隔，只会清理比 spark.history.fs.cleaner.maxAge 更老的日志文件。 |
| spark.history.fs.cleaner.maxAge | 7d | 日志文件的最大年龄，超过就会被清理。 |
| spark.history.fs.numReplayThreads | 25% of available cores histroy sever | 操作日志的最大线程数。 |

> 文件： conf/spark-defaults.conf

请注意，在所有这些 UI，表是可排序的点击他们的表头，让它容易更容易识别任务缓慢，数据倾斜的任务。

注意 :

1. history server 会显示完成和未完成的 Spark 任务。如果一个程序多次尝试失败后，这时会显示失败的尝试，或任何正在进行的却未完成的尝试，或者最终成功的尝试。
2. 未完成的程序只会间歇性地更新。更新之间的时间间隔定义在的检查文件变化间隔（spark.history.fs.update.interval）。在更大的集群上更新间隔可能被设置为更大值。可以通过 web UI 查看正在运行的程序。
3. 程序如果退出时没有表明注册已经完成，将会被列入未完成的一栏，尽管他们不再运行。如果一个应用程序崩溃可能导致该情况发生，。
4. 一种表明 Spark 工作完成方法是显式调用停止 Spack Context 的 sc.stop() 方法，或者在 Python 中使用 with SparkContext() as sc : 构造方法来加载和去除 Spark Context。

#### REST API

除了 UI 中查看这些指标，也可以得到任务信息的 JSON ，这个能够是开发者更方便的创造新的 Spark 可视化和监控。 正在运行的程序和历史的程序都可以得到他们的 JSON 信息。挂载在 /api/v1，比如，对于 histroy server，他们通常会访问 *http://\<server-url\>:18080/api/v1* ，对于运行的程序，则是 *http://localhost:4040/api/v1*。

API 中，程序拥有一个程序 ID， [app-id]。在 YARN 上执行时，每个程序可能进行有多次尝试，但尝试的 ID（attempt IDs）只有在集群模式下有，客户端模式的程序没有。在 YARN 集群模式上执行的程序会有 [attempt-id]。在下面列出的 API 中，是在 YARN 集群模式下运行时， [app-id] 就是 [base-app-id]/[attempt-id]，[base-app-id] 是 YARN 的程序 ID。

| 位置 | 含义 |
| ------------- | ------------- |
| /applications | 所有的应用程序的列表。?status=[completed\|running] 显示特定状态的程序。?minDate=[date] 显示的最早时间.。示例 :?minDate=2015-02-10；?minDate=2015-02-03T16:42:40.000GMT；?maxDate=[date] 显示的最新的时间; 格式和 minDate 相同；?limit=[limit] 程序显示数量的限制。 |
| /applications/[app-id]/jobs | 指定应用的所有 Job（作业）列表 :?status=[complete\|succeeded\|failed] 显示特定状态的信息。 |
| /applications/[app-id]/jobs/[job-id] | 某个 job 的详情。 |
| /applications/[app-id]/stages | 显示某个程序的 stages 列表。 |
| /applications/[app-id]/stages/[stage-id] | 显示给定的 stages-id 状态。?status=[active\|complete\|pending\|failed] 显示特定状态的信息。 |
| /applications/[app-id]/stages/[stage-id]/[stage-attempt-id] | 指定 stage attempt 的详情。 |
| /applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskSummary | stage attempt 的指标集合统计。?quantiles 统计指定的 quantiles。示例 : ?quantiles=0.01，0.5，0.99 |
| /applications/[app-id]/stages/[stage-id]/[stage-attempt-id]/taskList | 指定 stage attempt 的 task 列表。?offset=[offset]&length=[len] 显示指定范围的task.。?sortBy=[runtime\|-runtime] task 排序.。示例 : ?offset=10&length=50&sortBy=runtime |
| /applications/[app-id]/executors | 程序的 executors。 |
| /applications/[app-id]/storage/rdd | 程序的 RDDs。 |
| /applications/[app-id]/storage/rdd/[rdd-id] | 指定 RDD 详情。 |
| /applications/[base-app-id]/logs | 以 zip 文件形式下载指定程序的 log。 |
| /applications/[base-app-id]/[attempt-id]/logs | 以 zip 文件形式下载指定程序 atempt 的 log。 |

可恢复的 jobs 和 stages 的数量被独立的 Spark UI 的相同保留机制所约束;  "spark.ui.retainedJobs" 定义触发回收垃圾 jobs 阈值，和 spark.ui.retainedStages 限定 stages。注意，配置需要重启才能生效:通过增加这些值和重新启动服务器才可以保存获取更多的信息。

#### API 版本政策

这些 endpoints 都已经版本化以便在其之上开发，Spark 保证 :

- endpoints 不会被移除
- 对于任何给定的 endpoint，不会删除个别 fields
- 可能新增 endpoints
- 可能新增已有 endpoints 的 fields
- 将来可能在单独的 endpoint（例如，api / v2）添加 api 的新版本。 新版本不需要向后兼容。
- API 版本可能被删除，但之前至少有版本老的 API 与新的 API 版本共存。

注意，即使检查正在运行的程序的 UI，applications/[app-id] 部分仍然是必需的，尽管只有一个程序可用。如。查看正在运行的应用程序的工作列表，你会去 *http://localhost:4040/api/v1/applications/[app-id]/jobs*。这是保持在两种模式下的路径一致。

### Metrics

Spark 拥有可配置的 metrics system （度量系统） 其基于 [Dropwizard Metrics Library](http://metrics.dropwizard.io/3.1.0/ "http://metrics.dropwizard.io/3.1.0/")。这使用户可以通过多种 sinks 比如 HTTP，JMX，CSV 文件报告 Spark 的各项指标。Metrics system 是通过一个配置文件配置， Spark 需要其在路径 $SPARK_HOME/conf/metrics.properties 下。 可以通过 spark.metrics.conf [配置属性](/2016/12/24/spark-configuration/#spark- "/2016/12/24/spark-configuration/#spark-") 指定自定义文件的位置 。Spark 的指标拥有不同实例，对于不同的 Spark 组件是解耦的。在每个实例中，您可以配置一组需要的 Metrics 。目前支持以下 :

- master :  Spark 独立的 master 进程。
- applications : 一个 master 组件用于报告各种程序。
- worker : Spark 独立的 worker 进程。
- executor :  Spark executor。
- driver : Spark 驱动进程，（创建 SparkContext 的进程）。

每个例子都可以报告 0 项以上的 sinks，包含在 org.apache.spark.metrics.sink 之中

- ConsoleSink : 记录指标信息到控制台。
- CSVSink : 周期记录到 CSV 文件。
- JmxSink : JMX 监控。
- MetricsServlet : 启动 Servlet 向 Spark UI 提供 JSON 格式信息。
- GraphiteSink : 传送到 Graphite 节点。
- Slf4jSink : 存到 slf4j 。

Spark 同样支持 Ganglia sink ，因为版权原因无法默认安装 :

- GangliaSink : 发到 Ganglia 节点或者组播组（multicast group）。

安装 GangliaSink 需要构建一个定制的 Spark。注意，通过嵌入这个插件库 Spark 将包括 [LGPL](http://www.gnu.org/copyleft/lesser.html "http://www.gnu.org/copyleft/lesser.html")-licensed 授权的代码。对于 sbt 用户，构建前设置 SPARK_GANGLIA_LGPL 环境变量。对于 Maven 用户，要使 -Pspark-ganglia-lgpl 生效，除了修改 Spark 集群构建的环境配置，用户程序也需要加入 spark-ganglia-lgpl 组件。标准配置的语法在示例文件中，$SPARK_HOME/conf/metrics.properties.template。

## 高级工具

一些扩展工具可以用来帮助监控 Spark 任务性能 :

- 集群监控工具， 比如 [Ganglia](http://ganglia.sourceforge.net/ "http://ganglia.sourceforge.net/")， 可以对整个集群的利用率和性能瓶颈监控。比如， 一个 Ganglia 监控表盘可以很快地展示一个特定任务是否占用磁盘，网络，cpu。
- 操作系统分析工具如 [dstat](http://dag.wiee.rs/home-made/dstat/ "http://dag.wiee.rs/home-made/dstat/")， [iostat](https://linux.die.net/man/1/iostat "https://linux.die.net/man/1/iostat")， 和 [iotop](https://linux.die.net/man/1/iotop "https://linux.die.net/man/1/iotop") 可以在单个节点上提供细粒度分析。
- JVM 工具例如 jstack 生成 stack traces（线程堆栈信息）， jmap 生成 heap-dump（内存信息）， jstat 报告运行状态信息，并且 jconsole 提供对帮助理解 JVM 核心的各项属性可视化监视、管理。
