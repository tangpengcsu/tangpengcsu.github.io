---
layout: post
title: Spark 配置
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


Spark 提供了三个位置对系统进行配置：

- [Spark 属性](/2016/12/24/spark-configuration/#spark-)：通过 [SparkConf](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkConf) 对象或者通过 Java 系统参数来控制大多数的参数，
- [环境变量](/2016/12/24/spark-configuration/#section-5)：用于对每个节点进行设置， 例如通过 conf/spark-env.sh 脚本配置 IP 等。
- [日志](/2016/12/24/spark-configuration/#logging)：通过 log4j.properties 配置。

<!-- more -->

## Spark 属性

Spark 属性可以控制大多数的应用程序设置，并且每个应用的设置都是分开的。这些属性可以通过 [SparkConf](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkConf) 对象直接设定。SparkConf 为一些常用的属性定制了专用方法（如，master URL 和 application name），其他属性都可以用键值对做参数，调用 set() 方法来设置。例如，我们可以初始化一个包含 2 个本地线程的 Spark 应用，代码如下：

注意，local[2] 代表 2 个本地线程 – 这是最小的并发方式，可以帮助我们发现一些只有在分布式上下文才能复现的 bug。

```scala
val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
val sc = new SparkContext(conf)
```

注意，本地模式下，我们可以使用多个线程。而且在像 Spark Streaming 这样的场景下，我们可能需要多个线程来防止类似线程饿死这样的问题。

配置时间段的属性应该写明时间单位，如下格式都是可接受的：

```
25ms (milliseconds)
5s (seconds)
10m or 10min (minutes)
3h (hours)
5d (days)
1y (years)
```

配置大小的属性也应该写明单位，如下格式都是可接受的：

```
1b (bytes)
1k or 1kb (kibibytes = 1024 bytes)
1m or 1mb (mebibytes = 1024 kibibytes)
1g or 1gb (gibibytes = 1024 mebibytes)
1t or 1tb (tebibytes = 1024 gibibytes)
1p or 1pb (pebibytes = 1024 tebibytes)
```

### 动态加载 Spark 属性

在某些场景下，你可能想避免将属性值写死在 SparkConf 中。例如，你可能希望在同一个应用上使用不同的 master 或不同的内存总量。Spark 允许你简单地创建一个空的 SparkConf 对象：

```scala
val sc = new SparkContext(new SparkConf())
```

然后在运行时设置这些属性：

```bash
./bin/spark-submit --name "My app" --master local[4] --conf spark.eventLog.enabled=false
  --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" myApp.jar
```

Spark shell 和 [spark-submit](/2016/12/24/spark-submitting-applications/ "/2016/12/24/spark-submitting-applications/") 工具支持两种动态加载配置的方法。第一种，通过命令行选项，如：上面提到的–master。spark-submit 可以在启动 Spark 应用时，通过–conf 标志接受任何属性配置，同时有一些特殊配置参数同样可用。运行./bin/spark-submit –help 可以展示这些选项的完整列表。

同时，bin/spark-submit 也支持从 conf/spark-defaults.conf 中读取配置选项，在该文件中每行是一个键值对，并用空格分隔，如下：

```ini
spark.master            spark://5.6.7.8:7077
spark.executor.memory   4g
spark.eventLog.enabled  true
spark.serializer        org.apache.spark.serializer.KryoSerializer
```

这些通过参数或者属性配置文件传递的属性，最终都会在 SparkConf 中合并。其优先级是：首先是 SparkConf 代码中写的属性值，其次是 spark-submit 或 spark-shell 的标志参数，最后是 spark-defaults.conf 文件中的属性。有一些配置项被重命名过，这种情形下，老的名字仍然是可以接受的，只是优先级比新名字优先级低。

### 查看 Spark 属性

Spark 应用程序 http://\<driver\>:4040” 的 Environment“ tab 页可以查看 Spark 属性。如果你真的想确认一下属性设置是否正确的话，这个功能就非常有用了。注意，只有显式地通过 SparkConf 对象、在命令行参数、或者 spark-defaults.conf 设置的参数才会出现在页面上。其他属性，你可以认为都是默认值。

### 可用的属性

绝大多数属性都有合理的默认值。这里是部分常用的选项：

#### 应用程序属性

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.app.name | (none) | Spark 应用的名字。会在 SparkUI 和日志中出现。 |
| spark.dirver.cores | 1 |	在 cluster 模式下，用几个 core 运行 driver 进程。 |
| spark.driver.maxResultSize | 1g | Spark action 算子返回的结果集的最大数量。至少要 1M，可以设为 0 表示无限制。如果结果超过这一大小，Spark job 会直接中断退出。但是，设得过高有可能导致 driver 出现 out-of-memory 异常（取决于 spark.driver.memory 设置，以及驱动器 JVM 的内存限制）。设一个合理的值，以避免 driver 出现 out-of-memory 异常。 |
| spark.driver.memory	| 1g | driver 进程可以使用的内存总量（如：1g，2g）。注意，在 client 模式下，这个配置不能在 SparkConf 中直接设置，应为在那个时候 driver 进程的 JMR 已经启动了。因此需要在命令行里用 –-driver-memory 选项 或者在默认属性配置文件里设置。 |
| spark.executor.memory | 1g | 每个 executor 进程使用的内存总量（如，2g，8g）。 |
| spark.extraListeners	| (none) | 逗号分隔的实现 SparkListener 接口的类名列表；初始化 SparkContext 时，这些类的实例会被创建出来，并且注册到 Spark 的监听器上。如果这些类有一个接受 SparkConf 作为唯一参数的构造函数，那么这个构造函数会被调用；否则，就调用无参构造函数。如果没有合适的构造函数，SparkContext 创建的时候会抛异常。 |
| spark.local.dir	| /tmp | Spark 的” 草稿 “目录，包括 map 输出的临时文件以及 RDD 存在磁盘上的数据。这个目录最好在本地文件系统中。这个配置可以接受一个以逗号分隔的多个挂载到不同磁盘上的目录列表。注意：Spark-1.0 及以后版本中，这个属性会被 cluster manager 设置的环境变量覆盖：SPARK_LOCAL_DIRS（Standalone,Mesos）或者 LOCAL_DIRS（YARN）。 |
| spark.logConf	| false | SparkContext 启动时是否把生效的 SparkConf 属性以 INFO 日志打印到日志里
| spark.master	| (none) | 要连接的 cluster manager。参考 [Cluster Manager](/2016/12/24/spark-submitting-applications/#master-url "/2016/12/24/spark-submitting-applications/#master-url") 类型 |
| spark.submit.deployMode |	(none) | Spark driver 程序的部署模式，可以是 "client" 或 "cluster"，意味着部署 dirver 程序本地 ("client") 或者远程 ("cluster") 在 Spark 集群的其中一个节点上。 |

#### 运行时环境

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.driver.extraClassPath	| (none) | 额外的路径需预先考虑到驱动程序 classpath。注意: 在客户端模式下，这一套配置不能通过 SparkConf 直接在应用在应用程序中，因为 JVM 驱动已经启用了。相反，请在配置文件中通过设置 ----driver-class-path 选项或者选择默认属性。 |
| spark.driver.extraJavaOptions |	(none) | 一些额外的 JVM 属性传递给驱动。例如，GC 设置或其他日志方面设置。注意，设置最大堆大小 (-Xmx) 是不合法的。最大堆大小设置可以通过在集群模式下设置 spark.driver.memory 选项，并且可以通过 --driver-memory 在客户端模式设置。注意：在客户端模式下，这一套配置不能通过 SparkConf 直接应用在应用程序中，因为 JVM 驱动已经启用了。相反，请在配置文件中通过设置 --driver-java-options 选项或者选择默认属性。 |
| spark.driver.extraLibraryPath	| (none) | 当启动 JVM 驱动程序时设置一个额外的库路径。注意: 在客户端模式下，这一套配置不能通过 SparkConf 直接在应用在应用程序中，因为 JVM 驱动已经启用了。相反，请在配置文件中通过设置 --driver-library-path 选项或者选择默认属性。 |
| spark.driver.userClassPathFirst |	(none) | （实验）在驱动程序加载类库时，用户添加的 Jar 包是否优先于 Spark 自身的 Jar 包。这个特性可以用来缓解冲突引发的依赖性和用户依赖。目前只是实验功能。这是仅用于集群模式。 |
| spark.executor.extraClassPath |	(none) | 额外的类路径要预先考虑到 executor 的 classpath。这主要是为与旧版本的 Spark 向后兼容。用户通常不应该需要设置这个选项。 |
| spark.executor.extraJavaOptions |	(none) | 一些额外的 JVM 属性传递给 executor。例如，GC 设置或其他日志方面设置。注意，设置最大堆大小 (-Xmx) 是不合法的。Spark 应该使用 SparkConf 对象或 Spark 脚本中使用的 spark-defaults.conf 文件中设置。最大堆大小设置可以在 spark.executor.memory 进行设置。 |
| spark.executor.extraLibraryPath	| (none) | 当启动 JVM 的可执行程序时设置额外的类库路径。 |
| spark.executor.logs.rolling.maxRetainedFiles | (none) | 最新回滚的日志文件将被系统保留。旧的日志文件将被删除。默认情况下禁用。 |
| spark.executor.logs.rolling.maxSize	| (none) | 设置最大文件的大小, 以字节为单位日志将被回滚。默认禁用。见 spark.executor.logs.rolling.maxRetainedFiles 旧日志的自动清洗。 |
| spark.executor.logs.rolling.strategy |	(none) | 设置 executor 日志的回滚策略。它可以被设置为 “时间”（基于时间的回滚）或 “大小”（基于大小的回滚）。对于 “时间”，使用 spark.executor.logs.rolling.time.interval 设置回滚间隔。用 spark.executor.logs.rolling.maxSize 设置最大文件大小回滚。 |
| spark.executor.logs.rolling.time.interval |	daily	| 设定的时间间隔，executor 日志将回滚。默认情况下是禁用的。有效值是每天，每小时，每分钟或任何时间间隔在几秒钟内。见 spark.executor.logs.rolling.maxRetainedFiles 旧日志的自动清洗。 |
| spark.executor.userClassPathFirst	| false | （实验）与 spark.driver.userClassPathFirst 相同的功能，但适用于执行程序的实例。 |
| spark.executorEnv.[EnvironmentVariableName] |	(none) | 通过添加指定的环境变量 EnvironmentVariableName 给 executor 进程。用户可以设置多个环境变量。 |
| spark.python.profile	| false |	启用在 python 中的 profile。结果将由 sc.show_profiles() 显示, 或者它将会在驱动程序退出后显示。它还可以通过 sc.dump_profiles dump 到磁盘。如果一些 profile 文件的结果已经显示，那么它们将不会再驱动程序退出后再次显示。默认情况下，pyspark.profiler.BasicProfiler 将被使用，但这可以通过传递一个 profile 类作为一个参数到 SparkContext 中进行覆盖。 |
| spark.python.profile.dump	| (none) |	这个目录是在驱动程序退出后，proflie 文件 dump 到磁盘中的文件目录。结果将为每一个 RDD dump 为分片文件。它们可以通过 ptats.Stats() 加载。如果指定，profile 结果将不会自动显示。 |
| spark.python.worker.memory | 512m | 在聚合过程中，每个 python 进程所用的内存大小，和 JVM 内存相同的格式。如果内存中超过设定值，就会溢出到磁盘。 |
| spark.python.worker.reuse	| true | 重用 python worker。如果为 true，它将使用固定数量的 worker 数量。不需要为每一个任务分配 python 进程。如果是大型的这将是非常有用。 |

#### Shuffle 行为 (Behavior)

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.reducer.maxSizeInFlight | 48m | 从每个 Reduce 任务中并行的 fetch 数据的最大大小。因为每个输出都要求我们创建一个缓冲区，这代表要为每一个 Reduce 任务分配一个固定大小的内存。除非内存足够大否则尽量设置小一点。 |
| spark.reducer.maxReqsInFlight |	Int.MaxValue | 在集群节点上，这个配置限制了远程 fetch 数据块的连接数目。当集群中的主机数量的增加时候，这可能导致大量的到一个或多个节点的主动连接，导致负载过多而失败。通过限制获取请求的数量，可以缓解这种情况。 |
| spark.shuffle.compress | true | 是否要对 map 输出的文件进行压缩。默认为 true，使用 spark.io.compression.codec 。 |
| spark.shuffle.file.buffer | 32k	| 每个 shuffle 文件输出流的内存大小。这些缓冲区的数量减少了磁盘寻道和系统调用创建的 shuffle 文件。 |
| spark.shuffle.io.maxRetries	| 3	| (仅适用于 Netty) 如果设置了非 0 值，与 IO 异常相关失败的 fetch 将自动重试。在遇到长时间的 GC 问题或者瞬态网络连接问题时候，这种重试有助于大量 shuffle 的稳定性。 |
| spark.shuffle.io.numConnectionsPerPeer | 1	| (仅适用于 Netty) 主机之间的连接被重用，以减少更多集群创建连接。对于硬盘和一些主机的集群，这可能导致磁盘并发不足，所以需要考虑到这一点。 |
| spark.shuffle.io.preferDirectBufs	| true |(仅适用于 Netty) 堆缓冲区用于减少在 shuffle 和缓存块传输中的垃圾回收。对于严格限制的堆内存环境中，用户可能希望把这个设置关闭，使得所有分配从 Netty 到堆。
| spark.shuffle.io.retryWait | 5s | (仅适用于 Netty)fetch 重试的等待时长。默认 15s。计算公式是 maxRetries * retryWait。 |
| spark.shuffle.service.enabled	| false	| 使用外部 shuffle。这个服务提供了由 executor 写出的 shuffle 文件所以 executor 可以被安全移除。 如果 spark.dynamicAllocation.enabled 设置为 true 那么这个选项一定是 true。外部 shuffle 服务必须通过设置来启用它。通过 [动态分配配置和设置文档](/2016/12/24/spark-job-scheduling/#section-3 "/2016/12/24/spark-job-scheduling/#section-3") 了解更多信息。 |
| spark.shuffle.service.port	| 7337 | 外部 shuffle 的运行端口。 |
| spark.shuffle.sort.bypassMergeThreshold |	200	| 在基于排序的 shuffle 管理中，如果没有在 map 端的数据聚集并且如果有这个数据量的 reduce partition，可以避免归并排序。 |
| spark.shuffle.spill.compress | true	| shuffle 过程中对溢出的文件是否压缩。使用 spark.io.compression.codec. |
| spark.io.encryption.enabled	| false |	Enable IO encryption. Currently supported by all modes except Mesos. It's recommended that RPC encryption be enabled when using this feature. |
| spark.io.encryption.keySizeBits	| 128	| IO encryption key size in bits. Supported values are 128, 192 and 256. |
| spark.io.encryption.keygen.algorithm | HmacSHA1	| The algorithm to use when generating the IO encryption key. The supported algorithms are described in the KeyGenerator section of the Java Cryptography Architecture Standard Algorithm Name Documentation. |

#### Spark UI

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.eventLog.compress	| false | 是否压缩日志文件。如果设置为 ture 即为压缩。 |
| spark.eventLog.dir | file:///tmp/spark-events | Spark 事件日志的文件路径。如果 spark.eventLog.enabled 为 true。在这个基本目录下，Spark 为每个应用程序创建一个二级目录，日志事件特定于应用程序的目录。用户可能希望设置一个统一的文件目录像一个 HDFS 目录那样，所以历史文件可以从历史文件服务器中读取。 |
| spark.eventLog.enabled | false | 是否对 Spark 事件记录日志。在应用程序启动后有助于重建 Web UI。|
| spark.ui.killEnabled | true | 允许从 Web UI 中结束相应的工作进程。 |
| spark.ui.port |	4040 |	应用 UI 的端口，用于显示内存和工作负载数据。 |
| spark.ui.retainedJobs | 1000 | 在垃圾回收前，Spark UI 和 API 有多少 Job 可以留存。 |
| spark.ui.retainedStages	| 1000 | 在垃圾回收前，Spark UI 和 API 有多少 Stage 可以留存。 |
| spark.ui.retainedTasks	| 100000 | 在垃圾回收前，Spark UI 和 API 有多少 Task 可以留存。 |
| spark.worker.ui.retainedExecutors	| 1000 | 在垃圾回收前，Spark UI 和 API 有多少 executor 已经完成。 |
| spark.worker.ui.retainedDrivers	| 1000 | 在垃圾回收前，Spark UI 和 API 有多少 driver 已经完成。 |
| spark.sql.ui.retainedExecutions	| 1000 | 在垃圾回收前，Spark UI 和 API 有多少 execution 已经完成。 |
| spark.streaming.ui.retainedBatches	| 1000 | 在垃圾回收前，Spark UI 和 API 有多少 batch 已经完成。 |
| spark.ui.retainedDeadExecutors	| 1000 | 在垃圾回收前，Spark UI 和 API 有多少 dead executors。 |

#### 压缩和序列化（Compression and Serialization）

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.broadcast.compress | true | 是否在发送广播变量前压缩。通常是个好主意。 |
| spark.io.compression.codec | lz4 | 内部数据使用的压缩编解码器，如 RDD 分区，广播变量和混洗输出。 默认情况下，Spark 提供三种编解码器：lz4, lzf 和 snappy。您还可以使用完全限定类名来指定编码解码器，例如：org.apache.spark.io.LZ4CompressionCodec，org.apache.spark.io.LZFCompressionCodec 和 org.apache.spark.io.SnappyCompressionCodec。 |
| spark.io.compression.lz4.blockSize | 32k | 在采用 LZ4 压缩编解码器的情况下，LZ4 压缩使用的块大小。减少块大小还将降低采用 LZ4 时的混洗内存使用。 |
| spark.io.compression.snappy.blockSize | 32k | 在采用 Snappy 压缩编解码器的情况下，Snappy 压缩使用的块大小。减少块大小还将降低采用 Snappy 时的混洗内存使用。 |
| spark.kryo.classesToRegister | (none) | 如果你采用 Kryo 序列化，给一个以逗号分隔的自定义类名列以注册 Kryo。有关详细信息，请参阅 [调优指南](/2016/12/24/spark-tuning/#section "/2016/12/24/spark-tuning/#section")。 |
| spark.kryo.referenceTracking | true | (当使用 Spark SQL Thrift Server 为 false) 是否在采用 Kryo 序列化数据时跟踪对同一对象的引用，如果你的对象图形包含循环则是有必要的，并且如果它们包含同一对象的多个副本则对效率有用。 如果你知道这不是这样，可以禁用提高性能。 |
| spark.kryo.registrationRequired | false | 是否需要注册 Kryo。 如果设置为'true'，如果未注册的类被序列化，Kryo 将抛出异常。如果设置为 false（默认值），Kryo 将与每个对象一起写入未注册的类名。 编写类名可能会导致显著的性能开销，因此启用此选项可以严格强制用户没有从注册中省略类。 |
| spark.kryo.registrator | (none) | 如果你采用 Kryo 序列化，则给一个逗号分隔的类列表，以使用 Kryo 注册你的自定义类。 如果你需要以自定义方式注册你的类，则此属性很有用，例如以指定自定义字段序列化程序。 否则，使用 spark.kryo.classesToRegisteris 更简单。 它应该设置为 [KryoRegistrator](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.serializer.KryoRegistrator "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.serializer.KryoRegistrator") 的子类。 详见：[调优指南](/2016/12/24/spark-tuning/#section "/2016/12/24/spark-tuning/#section")。 |
| spark.kryoserializer.buffer.max | 64m | Kryo 序列化缓冲区的最大允许大小。 这必须大于你需要序列化的任何对象。 如果你在 Kryo 中得到一个 “buffer limit exceeded” 异常，你就需要增加这个值。 |
| spark.kryoserializer.buffer | 64k | Kryo 序列化缓冲区的初始大小。 注意，每个 worker 上每个 core 会有一个缓冲区。 如果需要，此缓冲区将增长到 spark.kryoserializer.buffer.max。 |
| spark.rdd.compress | false | 是否压缩序列化的 RDD 分区（例如，在 Java 和 Scala 中为 forStorageLevel.MEMORY_ONLY_SER 或在 Python 中为 StorageLevel.MEMORY_ONLY）。 能节省大量空间，但多消耗一些 CPU 时间。 |
| spark.serializer | org.apache.spark.serializer.JavaSerializer | (当使用 Spark SQL Thrift Server 时为 org.apache.spark.serializer.KryoSerializer) 用于序列化将通过网络发送或需要以序列化形式缓存的对象的类。 Java 序列化的默认值适用于任何可序列化的 Java 对象，但是速度相当慢，因此我们建议使用 [org.apache.spark.serializer.KryoSerializer](/2016/12/24/spark-tuning/ "/2016/12/24/spark-tuning/") 并在需要速度时配置 Kryo 序列化。当然你可以通过继承 [org.apache.spark.Serializer](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.serializer.Serializer "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.serializer.Serializer") 类自定义一个序列化器 。 |
| spark.serializer.objectStreamReset | 100 | 当正使用 org.apache.spark.serializer.JavaSerializer 序列化时, 序列化器缓存对象虽然可以防止写入冗余数据，但是却停止这些缓存对象的垃圾回收。通过调用'reset'你从序列化程序中清除该信息，并允许收集旧的对象。 要禁用此周期性重置，请将其设置为 - 1。 默认情况下，序列化器会每过 100 个对象被重置一次。 |

#### 内存管理

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.memory.fraction | 0.6 | 用于执行和存储的（堆空间 - 300MB）的分数。这个值越低，溢出和缓存数据逐出越频繁。 此配置的目的是在稀疏、异常大的记录的情况下为内部元数据，用户数据结构和不精确的大小估计预留内存。推荐使用默认值。 有关更多详细信息，包括关于在增加此值时正确调整 JVM 垃圾回收的重要信息，请参阅 [this description](/2016/12/24/spark-tuning/#section-2 "/2016/12/24/spark-tuning/#section-2")。 |
| spark.memory.storageFraction | 0.5 | 不会被逐出内存的总量，表示为 spark.memory.fraction 留出的区域大小的一小部分。 这个越高，工作内存可能越少，执行和任务可能更频繁地溢出到磁盘。 推荐使用默认值。有关更多详细信息，请参阅 [this description](/2016/12/24/spark-tuning/#section-2 "/2016/12/24/spark-tuning/#section-2")。 |
| spark.memory.offHeap.enabled | false | 如果为 true，Spark 会尝试对某些操作使用堆外内存。 如果启用了堆外内存使用，则 spark.memory.off Heap.size 必须为正值。 |
| spark.memory.offHeap.size | 0 | 可用于堆外分配的绝对内存量（以字节为单位）。 此设置对堆内存使用没有影响，因此如果您的执行器的总内存消耗必须满足一些硬限制，那么请确保相应地缩减 JVM 堆大小。 当 spark.memory.offHeap.enabled = true 时，必须将此值设置为正值。 |
| spark.memory.useLegacyMode | false | 是否启用 Spark 1.5 及以前版本中使用的传统内存管理模式。 传统模式将堆空间严格划分为固定大小的区域，如果未调整应用程序，可能导致过多溢出。 必须启用本参数，以下选项才可用：spark.shuffle.memoryFraction spark.storage.memoryFraction spark.storage.unrollFraction |
| spark.shuffle.memoryFraction | 0.2 | （废弃）只有在启用 spark.memory.useLegacyMode 时，此属性才是可用的。 混洗期间用于聚合和 cogroups 的 Java 堆的分数。 在任何给定时间，用于混洗的所有内存映射的集合大小不会超过这个上限，超过该限制的内容将开始溢出到磁盘。 如果溢出频繁，请考虑增加此值，但这以 spark.storage.memoryFraction 为代价。 |
| spark.storage.memoryFraction | 0.6 | （废弃）只有在启用 spark.memory.useLegacyMode 时，此属性才是可用的。 Java 堆的分数，用于 Spark 的内存缓存。 这个值不应该大于 JVM 中老生代（old generation) 对象所占用的内存，默认情况下，它提供 0.6 的堆，但是如果配置你所用的老生代对象大小，你可以增加它。 |
| spark.storage.unrollFraction | 0.2 | （废弃）只有在启用 spark.memory.useLegacyMode 时，此属性才是可用的。spark.storage.memoryFraction 用于在内存中展开块的分数。 当没有足够的空闲存储空间来完全展开新块时，通过删除现有块来动态分配。 |

#### 执行行为 (Execution Behavior)

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.broadcast.blockSize | 4m | TorrentBroadcastFactory 的一个块的每个分片大小。 过大的值会降低广播期间的并行性（更慢了）; 但是，如果它过小，BlockManager 可能会受到性能影响。 |
| spark.executor.cores | 在 YARN 模式下默认为 1，独立和 Mesos 粗粒度模型中的 worker 节点的所有可用的 core。 | 单个执行器上使用的 core 数。 在独立和 Mesos 粗粒度模式下，设置此参数允许应用在同一 worker 上运行多个执行器，只要该 worker 上有足够的 core。 否则，每个应用在单个 worker 上只会启动一个执行器。 |
| spark.default.parallelism | 对于分布式混洗（shuffle）操作，如 reduceByKey 和 join，父 RDD 中分区的最大数量。 对于没有父 RDD 的并行操作，它取决于集群管理器：本地模式：本地机器上的 core 数；Mesos 细粒度模式：8；其他：所有执行器节点上的 core 总数或者 2，以较大者为准 | 如果用户没有指定参数值，则这个属性是 join，reduceByKey 和 parallelize 等转换返回的 RDD 中的默认分区数。 |
| spark.executor.heartbeatInterval | 10s | 每个执行器的心跳与驱动程序之间的间隔。 心跳让驱动程序知道执行器仍然存活，并用正在进行的任务的指标更新它。 |
| spark.files.fetchTimeout | 60s | 获取文件的通讯超时，所获取的文件是从驱动程序通过 SparkContext.addFile（）添加的。 |
| spark.files.useFetchCache | true | 如果设置为 true（默认），文件提取将使用由属于同一应用程序的执行器共享的本地缓存，这可以提高在同一主机上运行许多执行器时的任务启动性能。 如果设置为 false，这些缓存优化将被禁用，所有执行器将获取它们自己的文件副本。 如果使用驻留在 NFS 文件系统上的 Spark 本地目录，可以禁用此优化（有关详细信息，请参阅 [SPARK-6313](https://issues.apache.org/jira/browse/SPARK-6313 "https://issues.apache.org/jira/browse/SPARK-6313") ）。 |
| spark.files.overwrite | false | 当目标文件存在且其内容与源不匹配的情况下，是否覆盖通过 SparkContext.addFile（）添加的文件。 |
| spark.hadoop.cloneConf | false | 如果设置为 true，则为每个任务克隆一个新的 Hadoop 配置对象。 应该启用此选项以解决配置线程安全问题（有关详细信息，请参阅 [SPARK-2546](https://issues.apache.org/jira/browse/SPARK-2546 "https://issues.apache.org/jira/browse/SPARK-2546") ）。 默认情况下禁用此功能，以避免不受这些问题影响的作业的意外性能回退。 |
| spark.hadoop.validateOutputSpecs | true | 如果设置为 true，则验证 saveAsHadoopFile 和其他变体中使用的输出规范（例如，检查输出目录是否已存在）。 可以禁用此选项以静默由于预先存在的输出目录而导致的异常。 我们建议用户不要禁用此功能，除非需要实现与以前版本的 Spark 的兼容性。 可以简单地使用 Hadoop 的 FileSystem API 手动删除输出目录。 对于通过 Spark Streaming 的 StreamingContext 生成的作业会忽略此设置，因为在检查点恢复期间可能需要将数据重写到预先存在的输出目录。 |
 | spark.storage.memoryMapThreshold | 2m | 当从磁盘读取块时，Spark 内存映射的块大小。 这会阻止 Spark 从内存映射过小的块。 通常，存储器映射对于接近或小于操作系统的页大小的块具有高开销。 |

#### 网络（Networking）

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.rpc.message.maxSize | 128 | 在 “control plane” 通信中允许的最大消息大小（以 MB 为单位）; 一般只适用于在 executors 和 driver 之间发送的映射输出大小信息。 如果您正在运行带有数千个 map 和 reduce 任务的作业，并查看有关 RPC 消息大小的消息，请增加此值。 |
| spark.blockManager.port | (random) | 所有块管理器监听的端口。 这些都存在于 driver 和 executors 上。 |
| spark.driver.host | (local hostname) | 要监听的 driver 的主机名或 IP 地址。 这用于与 executors 和 standalone Master 进行通信。 |
| spark.driver.port | (random) | 要监听的 driver 的端口。这用于与 executors 和 standalone Master 进行通信。 |
| spark.network.timeout | 120s | 所有网络交互的默认超时。 如果未配置此项，将使用此配置替换 spark.core.connection.ack.wait.timeout，spark.storage.blockManagerSlaveTimeoutMs，spark.shuffle.io.connectionTimeout，spark.rpc.askTimeout 或 spark.rpc.lookupTimeout 。 |
| spark.port.maxRetries | 16 | 在绑定端口放弃之前的最大重试次数。 当端口被赋予特定值（非 0）时，每次后续重试将在重试之前将先前尝试中使用的端口增加 1。 这本质上允许它尝试从指定的开始端口到端口 + maxRetries 的一系列端口。 |
| spark.rpc.numRetries | 3 | 在 RPC 任务放弃之前重试的次数。 RPC 任务将在此数字的大多数时间运行。 |
| spark.rpc.retry.wait | 3s | RPC 请求操作在重试之前等待的持续时间。 |
| spark.rpc.askTimeout | 120s | RPC 请求操作在超时前等待的持续时间。 |
| spark.rpc.lookupTimeout | 120s | RPC 远程端点查找操作在超时之前等待的持续时间。 |

#### 调度（Scheduling）

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.cores.max | (not set) | 当以 “coarse-grained” 共享模式在 [standalone deploy 集群](/2016/12/24/spark-standalone/ "/2016/12/24/spark-standalone/") 或 [Mesos 集群上运行](/2016/12/24/spark-running-on-mesos/#mesos- "/2016/12/24/spark-running-on-mesos/#mesos-") 时，从集群（而不是每台计算机）请求应用程序的最大 CPU 内核数量。 如果未设置，默认值将是 Spark 的 standalone deploy 管理器上的 spark.deploy.defaultCores，或者 Mesos 上的无限（所有可用核心）。 |
| spark.locality.wait | 3s | 等待启动本地数据任务多长时间，然后在较少本地节点上放弃并启动它。 相同的等待将用于跨越多个地点级别（process-local, node-local, rack-local ，等所有）。 也可以通过设置 spark.locality.wait.node 等来自定义每个级别的等待时间。如果任务很长并且局部性较差，则应该增加此设置，但是默认值通常很好。 |
| spark.locality.wait.node | spark.locality.wait | 自定义 node locality 等待时间。 例如，您可以将其设置为 0 以跳过 node locality，并立即搜索机架位置（如果群集具有机架信息）。 |
| spark.locality.wait.process | spark.locality.wait | 自定义 process locality 等待时间。这会影响尝试访问特定执行程序进程中的缓存数据的任务。 |
| spark.locality.wait.rack | spark.locality.wait | 自定义 rack locality 等待时间。 |
| spark.scheduler.maxRegisteredResourcesWaitingTime | 30s | 在调度开始之前等待资源注册的最大时间量。 |
| spark.scheduler.minRegisteredResourcesRatio | 0.8 for YARN mode; 0.0 for standalone mode and Mesos coarse-grained mode	| 注册资源（注册资源 / 总预期资源）的最小比率（资源是 yarn 模式下的执行程序，standalone 模式下的 CPU 核心和 Mesos coarsed-grained 模式 ['spark.cores.max'值是 Mesos  coarse-grained 模式下的总体预期资源]）在调度开始之前等待。 指定为 0.0 和 1.0 之间的双精度。 无论是否已达到资源的最小比率，在调度开始之前将等待的最大时间量由 config spark.scheduler.maxRegisteredResourcesWaitingTime 控制。 |
| spark.scheduler.mode | FIFO | 作业之间的 [调度模式](/2016/12/24/spark-job-scheduling/#section-7 "/2016/12/24/spark-job-scheduling/#section-7") 提交到同一个 SparkContext。 可以设置为 FAIR 使用公平共享，而不是一个接一个排队作业。 对多用户服务有用。 |
| spark.scheduler.revive.interval | 1s | 调度程序复活工作资源去运行任务的间隔长度。 |
| spark.speculation | false | 如果设置为 “true”，则执行任务的推测执行。 这意味着如果一个或多个任务在一个阶段中运行缓慢，则将重新启动它们。 |
| spark.speculation.interval | 100ms | Spark 检查要推测的任务的时间间隔。 |
| spark.speculation.multiplier | 1.5 | 一个任务的速度可以比推测的平均值慢多少倍。 |
| spark.speculation.quantile | 0.75 | 对特定阶段启用推测之前必须完成的任务的分数。 |
| spark.task.cpus | 1 | 要为每个任务分配的核心数。 |
| spark.task.maxFailures | 4 | 放弃作业之前任何特定任务的失败次数。 分散在不同任务中的故障总数不会导致作业失败; 一个特定的任务允许失败这个次数。 应大于或等于 1. 允许重试次数 = 此值 - 1。 |

#### 动态分配（Dynamic Allocation）

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.dynamicAllocation.enabled | false | 是否使用动态资源分配，它根据工作负载调整为此应用程序注册的执行程序数量。 有关更多详细信息，请参阅 [此处](/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-2") 的说明。这需要设置 spark.shuffle.service.enabled。 以下配置也相关：spark.dynamicAllocation.minExecutors，spark.dynamicAllocation.maxExecutors 和 spark.dynamicAllocation.initialExecutors |
| spark.dynamicAllocation.executorIdleTimeout | 60s | 如果启用动态分配，并且执行程序已空闲超过此持续时间，则将删除执行程序。 有关更多详细信息，请参阅此 [描述](/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4 "/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4")。 |
| spark.dynamicAllocation.cachedExecutorIdleTimeout | infinity | 如果启用动态分配，并且已缓存数据块的执行程序已空闲超过此持续时间，则将删除执行程序。 有关详细信息，请参阅此 [描述](/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4 "/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4")。 |
| spark.dynamicAllocation.initialExecutors | spark.dynamicAllocation.minExecutors | 启用动态分配时要运行的执行程序的初始数。如果 `--num-executors`（或 `spark.executor.instances`）被设置并大于此值，它将被用作初始执行器数。 |
| spark.dynamicAllocation.maxExecutors | infinity | 启用动态分配的执行程序数量的上限。 |
| spark.dynamicAllocation.minExecutors | 0 | 启用动态分配的执行程序数量的下限。 |
| spark.dynamicAllocation.schedulerBacklogTimeout | 1s | 如果启用动态分配，并且有超过此持续时间的挂起任务积压，则将请求新的执行者。 有关更多详细信息，请参阅此 [描述](/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4 "/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4")。 |
| spark.dynamicAllocation.sustainedSchedulerBacklogTimeout | schedulerBacklogTimeout | 与 spark.dynamicAllocation.schedulerBacklogTimeout 相同，但仅用于后续执行者请求。 有关更多详细信息，请参阅此 [描述](/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4 "/2016/12/24/spark-job-scheduling/#section-2 "/2016/12/24/spark-job-scheduling/#section-4")。 |

#### 安全性（Security）

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.acls.enable | false | 是否开启 Spark acls。如果开启了，它检查用户是否有权限去查看或修改 job。 Note this requires the user to be known, so if the user comes across as null no checks are done。UI 利用使用过滤器验证和设置用户 |
| spark.admin.acls | Empty | 逗号分隔的用户或者管理员列表，列表中的用户或管理员有查看和修改所有 Spark job 的权限。如果你运行在一个共享集群，有一组管理员或开发者帮助 debug，这个选项有用 |
| spark.admin.acls.groups | Empty | 逗号分隔的用户或者管理员列表，列表中的用户或管理员有查看和修改所有 Spark job 的权限。如果你运行在一个共享集群，有一组管理员或开发者帮助 debug，这个是针对组管理员组用户 设置 spark.user.groups.mapping 和 spark.user.groups.mapping |
| spark.user.groups.mapping | org.apache.spark.security.ShellBasedGroupsMappingProvider | 用户组列表是由一组地图服务定义的特征 org.apache.spark.security。 GroupMappingServiceProvider 可以由这个属性配置。 提供一个默认的基于 unix shell 实现 org.apache.spark.security.ShellBasedGroupsMappingProvider 可以指定解决用户组列表。 注意: 这个实现只支持基于 Unix / Linux 环境。 Windows 环境 目前不支持。 然而, 一个新的平台 / 协议可以支持的实现 的特征 org.apache.spark.security.GroupMappingServiceProvider |
| spark.authenticate | false | 是否 Spark 验证其内部连接。如果不是运行在 YARN 上，请看 spark.authenticate.secret |
| spark.authenticate.secret | None | 设置密钥用于 spark 组件之间进行身份验证。 这需要设置 不启用运行在 yarn 和身份验证。 |
| spark.authenticate.enableSaslEncryption | false | 身份验证时启用加密通信。 这是块传输服务和支持 RPC 的端点。 |
| spark.network.sasl.serverAlwaysEncrypt | false | 禁用未加密的连接服务, 支持 SASL 验证。 这是目前支持的外部转移服务。 |
| spark.core.connection.ack.wait.timeout | 60s | 连接等待回答的时间。单位为秒。为了避免不希望的超时，你可以设置更大的值 |
| spark.core.connection.auth.wait.timeout | 30s | 连接时等待验证的实际。单位为秒 |
| spark.modify.acls | Empty | 逗号分隔的用户列表，列表中的用户有查看 (view)Spark web UI 的权限。默认情况下，只有启动 Spark job 的用户有查看权限 |
| spark.modify.acls.groups | Empty | 针对 spark.modify.acls 设置组权限 |
| spark.ui.filters | None | 应用到 Spark web UI 的用于过滤类名的逗号分隔的列表。过滤器必须是标准的 [javax servlet Filter](http://docs.oracle.com/javaee/6/api/javax/servlet/Filter.html "http://docs.oracle.com/javaee/6/api/javax/servlet/Filter.html")。通过设置 java 系统属性也可以指定每个过滤器的参数。spark.<class name of filter>.params='param1=value1,param2=value2'。例如 - Dspark.ui.filters=com.test.filter1、-Dspark.com.test.filter1.params='param1=foo,param2=testing' |
| spark.ui.view.acls | Empty | 逗号分隔的用户列表，列表中的用户有查看 (view)Spark web UI 的权限。默认情况下，只有启动 Spark job 的用户有查看权限 |
| spark.ui.view.acls.groups | Empty | 针对 spark.ui.view.acls 设置组权限 |


#### 加密（Encryption）

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
spark.ssl.enabled	| false | 所有支持的协议是否启用 [SSL 连接](/2016/12/24/spark-security/#ssl- "/2016/12/24/spark-security/#ssl-")；所有的 SSL 设置 spark.ssl.xxx 在哪里 xxx 是一个 特定的配置属性, 表示全局所有支持的配置协议。 为了覆盖全局配置特定的协议, 属性必须被覆盖特定于协议的名称空间；使用 spark.ssl.YYY.XXX 覆盖全局配置设置特定的协议用。 示例值多包括 fs , 用户界面 , standlone , historyServer。查看 SSL 配置 为服务细节层次 SSL 配置 |
| spark.ssl.enabledAlgorithms	| Empty	| 密码的逗号分隔列表。 指定的密码必须由 JVM 支持。 协议的参考列表可以查看这个 [页面](https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https "https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https")。 注意: 如果没有设置, 它将使用 JVM 的缺省密码 |
| spark.ssl.keyPassword	| None |  钥的密钥存储库的密码。 |
| spark.ssl.keyStore | None | 密钥存储库文件的路径。 可以绝对或相对路径的目录 组件开始。 |
| spark.ssl.keyStorePassword | None | 密钥存储密码 |
| spark.ssl.keyStoreType | JKS | 密钥存储库的类型 |
| spark.ssl.protocol | None | 协议名称。 协议必须由 JVM 支持。 协议的参考列表 一个可以找到这 [页面](https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https "https://blogs.oracle.com/java-platform-group/entry/diagnosing_tls_ssl_and_https") |
| spark.ssl.needClientAuth | false | 设置真实如果 SSL 客户机身份验证需求。 |
| spark.ssl.trustStore | None | 信任存储文件的路径。 可以绝对或相对路径的目录 组件是开始的地方 |
| spark.ssl.trustStorePassword | None | 信任存储密码 |
| spark.ssl.trustStoreType | JKS | 信任存储库的类型 |


#### Spark SQL

运行 SET -v 命令将显示 SQL 配置的完整列表。

```scala
// spark is an existing SparkSession  # spark 是一个现有的 SparkSession，可以用如下语法进行设置
spark.sql("SET -v").show(numRows = 200, truncate = false)
```

#### Spark Streaming

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.streaming.backpressure.enabled | false | 开启或关闭 Spark Streaming 内部的 backpressure mecheanism（自 1.5 开始）。基于当前批次调度延迟和处理时间，这使得 Spark Streaming 能够控制数据的接收率，因此，系统接收数据的速度会和系统处理的速度一样快。从内部来说，这动态地设置了系统的最大接收率。这个速率上限通过 spark.streaming.receiver.maxRate 和 spark.streaming.kafka.maxRatePerPartition 两个参数设定（如下）。 |
| spark.streaming.backpressure.initialRat | not set | 当 backpressure mecheanism 开启时，每个 receiver 接受数据的初始最大值。 |
| spark.streaming.blockInterval | 200ms | 在这个时间间隔（ms）内，通过 Spark Streaming receivers 接收的数据在保存到 Spark 之前，chunk 为数据块。推荐的最小值为 50ms。具体细节见 Spark Streaming 指南的 [performance tuning](/2016/12/24/spark-streaming-programming-guide/#section-10 "/2016/12/24/spark-streaming-programming-guide/#section-10") 一节。 |
| spark.streaming.receiver.maxRate | not set | 每秒钟每个 receiver 将接收的数据的最大速率（每秒钟的记录数目）。有效的情况下，每个流每秒将最多消耗这个数目的记录。设置这个配置为 0 或者 - 1 将会不作限制。细节参见 Spark Streaming 编程指南的 [deployment guide](/2016/12/24/spark-streaming-programming-guide/#section-4 "/2016/12/24/spark-streaming-programming-guide/#section-4") 一节。 |
| spark.streaming.receiver.writeAheadLog.enable | false | 为 receiver 启用 write ahead logs。所有通过接收器接收输入的数据将被保存到 write ahead logs，以便它在驱动程序故障后进行恢复。见星火流编程指南部署指南了解更多详情。细节参见 Spark Streaming 编程指南的 [deployment guide](/2016/12/24/spark-streaming-programming-guide/#section-4 "/2016/12/24/spark-streaming-programming-guide/#section-4") 一节。 |
| spark.streaming.unpersist | true | 强制通过 Spark Streaming 生成并持久化的 RDD 自动从 Spark 内存中非持久化。通过 Spark Streaming 接收的原始输入数据也将清除。设置这个属性为 false 允许流应用程序访问原始数据和持久化 RDD，因为它们没有被自动清除。但是它会造成更高的内存花费 |
| spark.streaming.stopGracefullyOnShutdown | false | 如果为 true，Spark 将缓慢地 (gracefully) 关闭在 JVM 运行的 StreamingContext ，而非立即执行。 |
| spark.streaming.kafka.maxRatePerPartition | not set | 在使用新的 Kafka direct stream API 时，从每个 kafka 分区读到的最大速率（每秒的记录数目）。详见 [Kafka Integration guide](http://spark.apache.org/docs/latest/streaming-kafka-integration.html "http://spark.apache.org/docs/latest/streaming-kafka-integration.html")。 |
| spark.streaming.kafka.maxRetries | 1 | driver 连续重试的最大次数，以此找到每个分区 leader 的上次 (latest) 的偏移量（默认为 1 以意味着 driver 将尝试最多两次）。仅应用于新的 kafka direct stream API。 |
| spark.streaming.ui.retainedBatches | 1000 | 在垃圾回收之前，Spark Streaming UI 和状态 API 所能记得的 批处理 (batches) 数量。 |
| spark.streaming.driver.writeAheadLog.closeFileAfterWrite | false | 在写入一条 driver 中的 write ahead log 记录 之后，是否关闭文件。如果你想为 driver 中的元数据 WAL 使用 S3（或者任何文件系统而不支持 flushing），设定为 true。 |
| spark.streaming.receiver.writeAheadLog.closeFileAfterWrite | false | 在写入一条 reveivers 中的 write ahead log 记录 之后，是否关闭文件。如果你想为 reveivers 中的元数据 WAL 使用 S3（或者任何文件系统而不支持 flushing），设定为 true。 |

#### SparkR

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.r.numRBackendThreads | 2 | 使用 RBackend 处理来自 SparkR 包中的 RPC 调用的线程数。 |
| spark.r.command | Rscript | 在 driver 和 worker 两种集群模式下可执行的 R 脚本。 |
| spark.r.driver.command | spark.r.command | 在 driver 的 client 模式下可执行的 R 脚本。在集群模式下被忽略。 |

#### 部署

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.deploy.recoveryMode | NONE | 集群模式下，Spark jobs 执行失败或者重启时，恢复提交 Spark jobs 的恢复模式设定。 |
| spark.deploy.zookeeper.url | None | 当 `spark.deploy.recoveryMode` 被设定为 ZOOKEEPER，这一配置被用来连接 zookeeper URL。 |
| spark.deploy.zookeeper.dir | None | 当 `spark.deploy.recoveryMode` 被设定为 ZOOKEEPER，这一配置被用来设定 zookeeper 目录为 store recovery state |

#### Cluster Manager（集群管理器）

Spark 中的每个集群管理器都有额外的配置选项，这些配置可以在每个模式的页面中找到。

- YARN
- Mesos
- Standalone Mode

## 环境变量

通过环境变量配置特定的 Spark 设置。环境变量从 Spark 安装目录下的 conf/spark-env.sh 脚本读取（或者是 window 环境下的 conf/spark-env.cmd）。在 Standalone 或 Mesos 模式下，这个文件可以指定机器的特定信息，比如主机名。它也可以为正在运行的 Spark Application 或者提交脚本提供来源。

注意，当 Spark 被安装，默认情况下 conf/spark-env.sh 是不存在的。但是，你可以通过拷贝 conf/spark-env.sh.template 来创建它。确保你的拷贝文件时可执行的。

spark-env.sh: 中有有以下变量可以被设置：

| Environment Variable | Meaning |
| ------------- | ------------- |
| JAVA_HOME	Java | 的安装路径（如果不在你的默认路径下） |
| PYSPARK_PYTHON | 在 driver 和 worker 中 PySpark 用到的 Python 二进制可执行文件（如何有默认为 Python2.7，否则为 python） |
| PYSPARK_DRIVER_PYTHON | 只在 driver 中 PySpark 用到的 Python 二进制可执行文件（默认为 PySpark_Python）
| PYSPARK_DRIVER_R | SparkR shell 用到的 R 二进制可执行文件（默认为 R） |
| SPARK_DRIVER_IP | 机器绑定的 IP 地址 |
| SPARK_PUBLIC_DNS | 你的 Spark 程序通知其他机器的主机名 |

除了以上参数，standalone cluster scripts 也可以设置其他选项，比如每个机器使用的 CPU 核数和最大内存。
因为 spark-env.sh 是 shell 脚本，一些可以通过程序的方式来设置，比如你可以通过特定的网络接口来计算 SPARK_LOCAL_IP。

注意：当以集群模式运行 Spark on YARN 时，环境变量需要通过 spark.yarn.appMasterEnv 来设定。在你的 conf/spark-defaults.conf 文件中的 [EnvironmentVariableName] 属性。集群模式下，spark-env.sh 中设定的环境变量将不会在 YARN Application Master 过程中反应出来。详见 YARN-related Spark PropertiesProperties。

## 配置 Logging（日志）

Spark 用 log4j 生成日志，你可以通过在 conf 目录下添加 log4j.properties 文件来配置。一种方法时拷贝 log4j.properties.template 文件。

## 覆盖配置目录

如果你想指定不同的配置目录，而不是默认的 SPARK_HOME/conf，你可以设置 SPARK_CONF_DIR。Spark 将从这一目录下读取（spark-defaults.conf, spark-env.sh, log4j.properties, 等）

## 继承 Hadoop 集群配置

如果你想用 Spark 来读写 HDFS，在 Spark 的 classpath 就需要包括两个 Hadoop 配置文件。

- hdfs-site.xml，为 HDFS client 提供默认的行为。
- core-site.xml，设定默认的文件系统名称。

这两个配置文件的位置视 CDH 和 HDP 两个版本而不同，不过一般来说在 /etc/hadoop/conf 下。一些工具，如 Cloudera Manager，创建即时 (one-the-fly) 的配置文件，但提供一个机制来下载它们的副本。

为了使这些文件对 Spark 可见，需要设定  $SPARK_HOME/spark-env.sh 中的 HADOOP_CONF_DIR 到一个包含配置文件的位置。
