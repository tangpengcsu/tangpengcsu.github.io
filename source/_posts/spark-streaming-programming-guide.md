---
layout: post
title: Spark Streaming
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


## Spark Streaming 概述

Spark Streaming 是 Spark 核心 API 的扩展，它支持弹性的，高吞吐的，容错的实时数据流的处理。数据可以通过多种数据源获取，例如 Kafka，Flume，Kinesis 以及 TCP sockets，也可以通过高阶函数例如 map，reduce，join，window 等组成的复杂算法处理。最终，处理后的数据可以输出到文件系统，数据库以及实时仪表盘中。事实上，你还可以在数据流上使用 Spark [机器学习](http://spark.apache.org/docs/latest/ml-guide.html "http://spark.apache.org/docs/latest/ml-guide.html") 以及 [图形处理](http://spark.apache.org/docs/latest/graphx-programming-guide.html "http://spark.apache.org/docs/latest/graphx-programming-guide.html") 算法 。

<!-- more -->

![streaming-arch](/images/spark/streaming-arch.png "streaming-arch")

在内部，它工作原理如下，Spark Streaming 接收实时输入数据流并将数据切分成多个批数据，然后交由 Spark 引擎处理并分批的生成结果数据流。

![streaming-flow](/images/spark/streaming-flow.png "streaming-flow")

Spark Streaming 提供了一个高层次的抽象叫做离散流 (discretized stream) 或者 DStream，它代表一个连续的数据流。DStream 可以通过来自数据源的输入数据流创建，例如 Kafka，Flume 以及 Kinesis，或者在其他 DStream 上进行高层次的操作创建。在内部，一个 DStream 是通过一个 [RDDs](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.RDD "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.RDD") 的序列来表示。

本指南告诉你如何使用 DStream 来编写一个 Spark Streaming 程序。你可以使用 Scala，Java 或者 Python (Spark 1.2 版本后引进) 来编写程序。

***

## 快速开始示例

在我们详细介绍如何你自己的 Spark Streaming 程序的细节之前，让我们先来看一看一个简单的 Spark Streaming 程序的样子。比方说，我们想要计算从一个监听 TCP socket 的数据服务器接收到的文本数据（text data）中的字数。所有你需要做的就是照着下面的步骤做。

首先，我们导入了 Spark Streaming 类和部分从 StreamingContext 隐式转换到我们的环境的名称，目的是添加有用的方法到我们需要的其他类（如 DStream）。 StreamingContext 是所有流功能的主要入口点。我们创建了一个带有 2 个执行线程和间歇时间为 1 秒的本地 StreamingContext 。

**Scala 示例：**

```scala
import org.apache.spark._
import org.apache.spark.streaming._
import org.apache.spark.streaming.StreamingContext._ // 自从 Spark 1.3 开始，不再是必要的了

// 创建一个具有两个工作线程 (working thread) 和批次间隔为 1 秒的本地 StreamingContext
//master 需要 2 个核，以防止饥饿情况 (starvation scenario)。
val conf = new SparkConf().setMaster("local[2]").setAppName("NetworkWordCount")
val ssc = new StreamingContext(conf, Seconds(1))
```

在使用这种背景下，我们可以创建一个代表从 TCP 源流数据的离散流（DStream），指定主机名 (hostname)（例如 localhost）和端口（例如 9999）。

**Scala 示例：**

```scala
// 创建一个将要连接到 hostname:port 的离散流，如 localhost:9999
val lines = ssc.socketTextStream("localhost", 9999)
```

上一步的这个 lines 离散流（DStream）表示将要从数据服务器接收到的数据流。在这个 离散流（DStream）中的每一条记录都是一行文本（text）。接下来，我们想要通过空格字符（space characters）拆分这些数据行（lines）成单词（words）。

**Scala**

```scala
// 将每一行拆分成单词
val words = lines.flatMap(_.split(" "))
```

flatMap 是一种一对多的离散流（DStream）操作，它会通过在源离散流（source DStream）中根据每个记录（record）生成多个新纪录的形式创建一个新的离散流（DStream）。在这种情况下，在这种情况下，每一行（each line）都将被拆分成多个单词（words）和代表单词离散流（words DStream）的单词流。接下来，我们想要计算这些单词。

**Scala 示例：**

```scala
import org.apache.spark.streaming.StreamingContext._ // 自从 Spark 1.3 不再是必要的
// 计算每一个批次中的每一个单词（ Count each word in each batch）
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// 在控制台打印出在这个离散流（DStream）中生成的每个 RDD 的前十个元素（Print the first ten elements of each RDD generated in this DStream to the console）
// 必需要触发 action（很多初学者会忘记触发 action 操作，导致报错：No output operations registered, so nothing to execute）
wordCounts.print()
```

上一步的 words 离散流进行了进一步的映射（一对一的转变）为一个 (word, 1) 对 的离散流（DStream），这个离散流然后被规约（reduce）来获得数据中每个批次（batch）的单词频率。最后， wordCounts.print() 将会打印一些每秒生成的计数。

请注意，当这些行（lines）被执行的时候， Spark Streaming 只有建立在启动时才会执行计算，在它已经开始之后，并没有真正地处理。为了在所有的转换都已经完成之后开始处理，我们在最后运行：

**Scala 示例：**

```scala
ssc.start()             // 启动计算
ssc.awaitTermination()  // 等待计算的终止
```

完整的代码可以在 Spark Streaming 的例子 NetworkWordCount 中找到。

如果你已经 下载 并且 建立 了 Spark，你可以运行下面的例子。你首先需要运行 Netcat(一个在大多数类 Unix 系统中的小工具) 作为我们使用的数据服务器。

```bash
$ nc -lk 9999
```

然后，在另一个不同的终端，你可以运行这个例子通过执行：

**Scala 示例：**

```scala
$ ./bin/run-example streaming.NetworkWordCount localhost 9999
```

然后，在运行在 netcat 服务器上的终端输入的任何行（lines），都将被计算，并且每一秒都显示在屏幕上，它看起来就像下面这样：

```bash
# TERMINAL 1:
# Running Netcat

$ nc -lk 9999

hello world
```

**Scala 示例：**

```scala
# TERMINAL 2: RUNNING NetworkWordCount

$ ./bin/run-example streaming.NetworkWordCount localhost 9999
...
-------------------------------------------
Time: 1357008430000 ms
-------------------------------------------
(hello,1)
(world,1)
...
```

***

## 基本概念

接下来，我们了解完了简单的例子，开始阐述 Spark Streaming 的基本知识。

### Spark Streaming 依赖

与 Spark 类似， Spark Streaming 可以通过 Maven 来管理依赖。为了写你自己的 Spark Streaming 程序，你必须添加以下的依赖到你的 SBT 或者 Maven 项目中。

**Maven 示例：**

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-streaming_2.11</artifactId>
    <version>2.0.0</version>
</dependency>
```

对于从现在没有在 Spark Streaming 核心 API 中的数据源获取数据，如 Kafka，Flume，Kinesis，你必须添加相应的组件 spark-streaming-xyz_2.11 到依赖中。例如，有一些常见的如下。

| Source | Artifact |
| ------------- | ------------- |
| Kafka | spark-streaming-kafka-0-8_2.11 |
| Flume | spark-streaming-flume_2.11 |
| Kinesis | spark-streaming-kinesis-asl_2.11 [Amazon Software License] |


想要查看一个实时更新的列表，请参阅 [Maven repository](http://search.maven.org/ "http://search.maven.org/") 来了解支持的源文件和组件的完整列表。

### 初始化 StreamingContext

为了初始化一个 Spark Streaming 程序，一个 StreamingContext 对象必须要被创建出来，它是所有的 Spark Streaming 功能的主入口点。

**Scala 示例：**

一个 StreamingContext 对象可以从一个 SparkConf 对象中创建出来。

**Scala**

```scala
import org.apache.spark._
import org.apache.spark.streaming._

val conf = new SparkConf().setAppName(appName).setMaster(master)
val ssc = new StreamingContext(conf, Seconds(1))
```

这个 appName 参数是展示在集群用户界面上的你的应用程序的名称。 master 是一个 [Spark, Mesos or YARN cluster URL](/2016/12/24/spark-submitting-applications/#master-url)，或者一个特殊的 "local[\*]" 字符串运行在本地模式下。在实践中，在一个集群上运行时，你不会想 在程序中硬编码 master，而是使用 [spark-submit 启动应用程序](/2016/12/24/spark-submitting-applications/ "/2016/12/24/spark-submitting-applications/")，并且接收这个参数。然而，对于本地测试和单元测试，你可以传递 "local[\*]" 去运行 Spark Streaming 过程（检测本地系统中内核的个数）。请注意，这内部创建了一个 [SparkContext](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkContext "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkContext") （所有 Spark 功能的出发点），它可以像这样被访问 ssc.sparkContext 。

这个批处理间隔（batch interval）必须根据您的应用程序和可用的集群资源的等待时间要求进行设置。详情请参阅 [性能调优](/2016/12/24/spark-streaming-programming-guide/#section-14 "/2016/12/24/spark-streaming-programming-guide/#section-14")（Performance Tuning） 部分。

一个 StreamingContext 对象也可以从一个现有的 SparkContext 对象中创建出。

**Scala 示例：**

```scala
import org.apache.spark.streaming._

val sc = ...                // 现有已存在的 SparkContext
val ssc = new StreamingContext(sc, Seconds(1))
```
一个 context 定义之后，你必须做以下几个方面。

1. 通过创建输入 DStreams 定义输入源。
2. 通过应用转换和输出操作 DStreams 定义流计算（streaming computations）。
3. 开始接收数据，并用 streamingContext.start() 处理它。
4. 等待处理被停止（手动停止或者因为任何错误停止）使用 StreamingContext.awaitTermination() 。
5. 该处理可以使用 streamingContext.stop() 手动停止。

要记住的要点：

1. 一旦一个 context 已经启动，将不会有新的数据流的计算可以被创建或者添加到它。
2. 一旦一个 context 已经停止，它不会被重新启动。
3. 同一时间内在 JVM 中只有一个 StreamingContext 可以被激活。
4. 在 StreamingContext 上的 stop() 同样也停止了 SparkContext 。为了只停止 StreamingContext，设置 stop() 的可选参数，名叫 stopSparkContext 为 false 。
5. 一个 SparkContext 可以被重新用于创建多个 StreamingContexts，只要是当前的 StreamingContext 被停止（不停止 SparkContext）之前创建下一个 StreamingContext 就可以。

### Discretized Streams (DStreams)（离散化流）

**离散化流** 或者 **离散流** 是 Spark Streaming 提供的基本抽象。它代表了一个连续的数据流，无论是从源接收到的输入数据流，还是通过变换输入流所产生的处理过的数据流。在内部，一个离散流（DStream）被表示为一系列连续的 RDDs，RDD 是 Spark 的一个不可改变的，分布式的数据集的抽象（查看 [Spark 编程指南](/2016/12/24/spark-programming-guides/#rdds "/2016/12/24/spark-programming-guides/#rdds") 了解更多）。在一个 DStream 中的每个 RDD 包含来自一定的时间间隔的数据，如下图所示。

![streaming-dstream](/images/spark/streaming-dstream.png "streaming-dstream")

应用于 DStream 的任何操作转化为对于底层的 RDDs 的操作。例如，在 [之前的例子](/2016/12/24/spark-streaming-programming-guide/#section "/2016/12/24/spark-streaming-programming-guide/#section")，转换一个行（lines）流成为单词（words）中，flatMap 操作被应用于在行离散流（lines DStream）中的每个 RDD 来生成单词离散流（words DStream）的 RDDs 。如下图所示。

![streaming-dstream-ops](/images/spark/streaming-dstream-ops.png "streaming-dstream-ops")

这些底层的 RDD 变换由 Spark 引擎（engine）计算。 DStream 操作隐藏了大多数这些细节并为了方便起见，提供给了开发者一个更高级别的 API 。这些操作细节会在后边的章节中讨论。

### 输入 DStreams 以及接收

输入 DStreams 是代表输入数据是从流的源数据（streaming sources）接收到的流的 DStream 。[在快速简单的例子](/2016/12/24/spark-streaming-programming-guide/#section "/2016/12/24/spark-streaming-programming-guide/#section") 中，行（lines）是一个输入 DStream，因为它代表着从 netcat 服务器接收到的数据的流。每个输入离散流（input DStream）（除了文件流（file stream），在后面的章节进行讨论）都会与一个接收器（[Scala doc](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.receiver.Receiver "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.receiver.Receiver")，[Java doc](http://spark.apache.org/docs/latest/api/java/org/apache/spark/streaming/receiver/Receiver.html "http://spark.apache.org/docs/latest/api/java/org/apache/spark/streaming/receiver/Receiver.html")）对象联系，这个接收器对象从一个源头接收数据并且存储到 Sparks 的内存中用于处理。

Spark Streaming 提供了两种内置的流来源（streaming source）。

1. 基本来源（Basic sources）：在 StreamingContext API 中直接可用的源（source）。例如，文件系统（file systems），和 socket 连接（socket connections）。

2. 高级来源（Advanced sources）：就像 Kafka，Flume，Kinesis 之类的通过额外的实体类来使用的来源。这些都需要连接额外的依赖，就像在 [连接](/2016/12/24/spark-streaming-programming-guide/#spark-streaming--1 "/2016/12/24/spark-streaming-programming-guide/#spark-streaming--1") 部分的讨论。

在本节的后边，我们将讨论每种类别中的现有的一些来源。

需要注意的是，如果你想要在你的流处理程序中并行的接收多个数据流，你可以创建多个输入离散流（input DStreams）（在 [性能优化](/2016/12/24/spark-streaming-programming-guide/#section-10 "/2016/12/24/spark-streaming-programming-guide/#section-10") 部分进一步讨论）。这将创建同时接收多个数据流的多个接收器（receivers）。但需要注意，一个 Spark 的 worker/executor 是一个长期运行的任务（task），因此它将占用分配给 Spark Streaming 的应用程序的所有核中的一个核（core）。因此，要记住，一个 Spark Streaming 应用需要分配足够的核（core）（或 线程（threads），如果本地运行的话）来处理所接收的数据，以及来运行接收器（receiver(s)）。

需要记住的要点

1. 当在本地运行一个 Spark Streaming 程序的时候，不要使用 "local" 或者 "local[1]" 作为 master 的 URL 。这两种方法中的任何一个都意味着只有一个线程将用于运行本地任务。如果你正在使用一个基于接收器（receiver）的输入离散流（input DStream）（例如， sockets，Kafka，Flume 等），则该单独的线程将用于运行接收器（receiver），而没有留下任何的线程用于处理接收到的数据。因此，在本地运行时，总是用 "local[n]" 作为 master URL，其中的 n > 运行接收器的数量（查看 [更多Spark 属性](/2016/12/24/spark-configuration/#spark- "/2016/12/24/spark-configuration/#spark-") 来了解怎样去设置 master 的信息）。
2. 将逻辑扩展到集群上去运行，分配给 Spark Streaming 应用程序的内核（core）的内核数必须大于接收器（receiver）的数量。否则系统将接收数据，但是无法处理它。

#### 基本来源（Basic Sources）

我们已经简单地了解过了 ssc.socketTextStream(...) 在 [快速开始的例子](/2016/12/24/spark-streaming-programming-guide/#section "/2016/12/24/spark-streaming-programming-guide/#section")中，例子中是从通过一个 TCP socket 连接接收到的文本数据中创建了一个离散流（DStream）。除了 sockets，StreamingContext API 也提供了根据文件作为输入来源创建离散流（DStreams）的方法。

1\. 文件流（File Streams）：用于从文件中读取数据，在任何与 HDFS API 兼容的文件系统中（即 HDFS，S3，NFS 等），一个离散流（DStream）可以像下面这样创建：

**Scala 示例：**

```scala
streamingContext.fileStream[KeyClass, ValueClass, InputFormatClass](dataDirectory)
```

Spark Streaming 将监控 dataDirectory 目录，并处理任何在该目录下创建的文件（写在嵌套目录中的文件是不支持的）。注意：

- 文件必须具有相同的数据格式。
- 文件必须在 dataDirectory 目录中通过原子移动或者重命名它们到这个 dataDirectory 目录下来创建。
- 一旦移动，这些文件必须不能再更改，因此如果文件被连续地追加，新的数据将不会被读取。

对于简单的文本文件，还有一个更加简单的方法 streamingContext.textFileStream(dataDirectory)。并且文件流（file streams）不需要运行一个接收器（receiver），因此，不需要分配内核（core）。

Python API fileStream 在 Python API 中是不可用的，只有 textFileStream 是可用的。

2\. 基于自定义的接收器（custom receivers）的流（Stream）：离散流（DStreams）可以使用通过自定义的接收器接收到的数据来创建。查看 [自定义接收器指南](http://spark.apache.org/docs/latest/streaming-custom-receivers.html "http://spark.apache.org/docs/latest/streaming-custom-receivers.html") 来了解更多细节。

3\. RDDs 队列作为一个流：为了使用测试数据测试 Spark Streaming 应用程序，还可以使用 streamingContext.queueStream(queueOfRDDs) 创建一个基于 RDDs 队列的离散流（DStream），每个进入队列的 RDD 都将被视为 DStream 中的一个批次数据，并且就像一个流进行处理。
想要了解更多的关于从 sockets 和文件（files）创建的流的细节，查看相关功能的 API 文档，Scala 中的 [StreamingContext](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext")，Java 中的 [JavaStreamingContext](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html "http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html") 和 Python 中的 [StreamingContext](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext "http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.StreamingContext")。

#### 高级来源（Advanced Sources）

Python API  在 Spark 2.0.0 中，这些来源中， Kafka， Kinesis 和 Flume 在 Python API 中都是可用的。

这一类别的来源需要使用非 Spark 库中的外部接口，它们中的其中一些还需要比较复杂的依赖关系（例如， Kafka 和 Flume）。因此，为了最小化有关的依赖关系的版本冲突的问题，这些资源本身不能创建 DStream 的功能，它是通过 [连接](/2016/12/24/spark-streaming-programming-guide/#spark-streaming--1 "/2016/12/24/spark-streaming-programming-guide/#spark-streaming--1")单独的类库实现创建 DStream 的功能。

需要注意的是这些高级来源在 Spark Shell 中是不可用的。因此，基于这些高级来源的应用程序不能在 shell 中被测试。如果你真的想要在 Spark shell 中使用它们，你必须下载带有它的依赖的相应的 Maven 组件的 JAR，并且将其添加到 classpath 。

一些高级来源如下。

- Kafka ： Spark Streaming 2.0.0 与 Kafka 0.8.2.1 以及更高版本兼容。查看 [Kafka 集成指南](http://spark.apache.org/docs/latest/streaming-kafka-integration.html "http://spark.apache.org/docs/latest/streaming-kafka-integration.html") 来了解更多细节。
- Flume ： Spark Streaming 2.0.0 与 Flume 1.6.0 兼容。查看 [Flume 集成指南](http://spark.apache.org/docs/latest/streaming-flume-integration.html "http://spark.apache.org/docs/latest/streaming-flume-integration.html") 来了解更多细节。
- Kinesis ： Spark Streaming 2.0.0 与 Kinesis 客户端库 1.2.1 兼容。查看 [Kinesis 集成指南](http://spark.apache.org/docs/latest/streaming-kinesis-integration.html "http://spark.apache.org/docs/latest/streaming-kinesis-integration.html") 来了解更多细节。

#### 自定义来源（Custom Sources）：

Python API 在 Python 中还不支持自定义来源。

输入离散流（Input DStreams）也可以从创建自定义数据源。所有你需要做的就是实现一个用户定义（user-defined）的接收器（receiver）（查看下一章节去了解那是什么），这个接收器可以从自定义的数据源接收数据并将它推送到 Spark 。查看 [自定义接收器指南](http://spark.apache.org/docs/latest/streaming-custom-receivers.html "http://spark.apache.org/docs/latest/streaming-custom-receivers.html")（Custom Receiver Guide） 来了解更多。

#### 接收器的可靠性（Reveiver Reliability）

可以有两种基于他们的可靠性的数据源。数据源（如 Kafka 和 Flume）允许传输的数据被确认。如果系统从这些可靠的数据来源接收数据，并且被确认（acknowledges）正确地接收数据，它可以确保数据不会因为任何类型的失败而导致数据丢失。这样就出现了 2 种接收器（receivers）：

1. 可靠的接收器（Reliable Receiver） - 当数据被接收并存储在 Spark 中并带有备份副本时，一个可靠的接收器（reliable receiver）正确地发送确认（acknowledgment）给一个可靠的数据源（reliable source）。
2. 不可靠的接收器（Unreliable Receiver） - 一个不可靠的接收器（ unreliable receiver ）不发送确认（acknowledgment）到数据源。这可以用于不支持确认的数据源，或者甚至是可靠的数据源当你不想或者不需要进行复杂的确认的时候。

如何去编写一个可靠的接收器的细节在 自定义接收器指南（Custom Receiver Guide） 中。

### DStreams 中的 Transformations

和 RDD 类似，transformation 允许从输入 [DStream](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.DStream) 来的数据被修改。DStreams 支持很多在 RDD 中可用的 transformation 算子。一些常用的算子如下所示:

| Transformation | Meaning |
| ------------- | ------------- |
| map(func) | 利用函数 func 处理原 DStream 的每个元素，返回一个新的 DStream |
| flatMap(func) |  与 map 相似，但是每个输入项可用被映射为 0 个或者多个输出项 |
| filter(func)  |  返回一个新的 DStream，它仅仅包含源 DStream 中满足函数 func 的项 |
| repartition(numPartitions) | 通过创建更多或者更少的 partition 改变这个 DStream 的并行级别 (level of parallelism) |
| union(otherStream) | 返回一个新的 DStream，它包含源 DStream 和 otherStream 的联合元素 |
| count() | 通过计算源 DStream 中每个 RDD 的元素数量，返回一个包含单元素 (single-element) RDDs 的新 DStream |
| reduce(func) | 利用函数 func 聚集源 DStream 中每个 RDD 的元素，返回一个包含单元素 (single-element) RDDs 的新 DStream。函数应该是相关联的，以使计算可以并行化 |
| countByValue() | 这个算子应用于元素类型为 K 的 DStream 上，返回一个（K,long）对的新 DStream，每个键的值是在原 DStream 的每个 RDD 中的频率。 |
| reduceByKey(func, [numTasks]) | 当在一个由 (K,V) 对组成的 DStream 上调用这个算子，返回一个新的由 (K,V) 对组成的 DStream，每一个 key 的值均由给定的 reduce 函数聚集起来。注意：在默认情况下，这个算子利用了 Spark 默认的并发任务数去分组。你可以用 numTasks 参数设置不同的任务数 |
| join(otherStream, [numTasks]) |  当应用于两个 DStream（一个包含（K,V）对， 一个包含 (K,W) 对），返回一个包含 (K, (V, W)) 对的新 DStream |
| cogroup(otherStream, [numTasks]) | 当应用于两个 DStream（一个包含（K,V）对， 一个包含 (K,W) 对），返回一个包含 (K, Seq[V], Seq[W]) 的元组 |
| transform(func) | 通过对源 DStream 的每个 RDD 应用 RDD-to-RDD 函数，创建一个新的 DStream。这个可以在 DStream 中的任何 RDD 操作中使用 |
| updateStateByKey(func) | 利用给定的函数更新 DStream 的状态，返回一个新 "state" 的 DStream。 |

最后两 个 transformation 算子需要重点介绍一下：

#### UpdateStateByKey 操作

updateStateByKey 操作允许不断用新信息更新它的同时保持任意状态。你需要通过两步来使用它：

1. **定义状态** - 状态可以是任何的数据类型
2. **定义状态更新函数** - 怎样利用更新前的状态和从输入流里面获取的新值更新状态

在每个 batch 中，Spark 会使用更新状态函数为所有的关键字更新状态，不管在 batch 中是否含有新的数据。如果这个更新函数返回一个 none，这个键值对也会被消除。

让我们举个例子说明。在例子中，你想保持一个文本数据流中每个单词的运行次数，运行次数用一个 state 表示，它的类型是整数

**Scala：**

```scala
def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = ...  // add the new values with the previous running count to get the new count
    Some(newCount)
}
```

**Java：**

```java
Function2<List<Integer>, Optional<Integer>, Optional<Integer>> updateFunction =
  new Function2<List<Integer>, Optional<Integer>, Optional<Integer>>() {
    @Override
    public Optional<Integer> call(List<Integer> values, Optional<Integer> state) {
      Integer newSum = ...  // add the new values with the previous running count to get the new count
      return Optional.of(newSum);
    }
  };
```

**Python：**

```python
def updateFunction(newValues, runningCount):
    if runningCount is None:
       runningCount = 0
    return sum(newValues, runningCount)  # add the new values with the previous running count to get the new count
```

这个 DStream 应用在单词计数上（[参考之前的案例](/2016/12/24/spark-streaming-programming-guide/#section "/2016/12/24/spark-streaming-programming-guide/#section")）

```scala
val runningCounts = pairs.updateStateByKey[Int](updateFunction _)
```

更新函数将会被每个单词调用，newValues 拥有一系列的 1（从 (word, 1) 组对应），runningCount 拥有之前的次数。要看完整的代码，见案例

注意， 使用 updateStateByKey 需要配置的检查点的目录，这是 详细的讨论 [CheckPointing](/2016/12/24/spark-streaming-programming-guide/#checkpointing "/2016/12/24/spark-streaming-programming-guide/#checkpointing") 部分。

#### Transform 操作

transform 操作（以及它的变化形式如 transformWith）允许在 DStream 运行任何 RDD-to-RDD 函数。它能够被用来应用任何没在 DStream API 中提供的 RDD 操作（It can be used to apply any RDD operation that is not exposed in the DStream API）。 例如，连接数据流中的每个批（batch）和另外一个数据集的功能并没有在 DStream API 中提供，然而你可以简单的利用 transform 方法做到。如果你想通过连接带有预先计算的垃圾邮件信息的输入数据流 来清理实时数据，然后过了它们，你可以按如下方法来做：

**Scala:**

```scala
val spamInfoRDD = ssc.sparkContext.newAPIHadoopRDD(...) // RDD containing spam information

val cleanedDStream = wordCounts.transform(rdd => {
  rdd.join(spamInfoRDD).filter(...) // join data stream with spam information to do data cleaning
  ...
})
```

**Java:**

```java
import org.apache.spark.streaming.api.java.*;
// RDD containing spam information
final JavaPairRDD<String, Double> spamInfoRDD = jssc.sparkContext().newAPIHadoopRDD(...);

JavaPairDStream<String, Integer> cleanedDStream = wordCounts.transform(
  new Function<JavaPairRDD<String, Integer>, JavaPairRDD<String, Integer>>() {
    @Override public JavaPairRDD<String, Integer> call(JavaPairRDD<String, Integer> rdd) throws Exception {
      rdd.join(spamInfoRDD).filter(...); // join data stream with spam information to do data cleaning
      ...
    }
  });
```

**Python:**

```python
spamInfoRDD = sc.pickleFile(...) # RDD containing spam information

# join data stream with spam information to do data cleaning
cleanedDStream = wordCounts.transform(lambda rdd: rdd.join(spamInfoRDD).filter(...))
```

请注意， 每批间隔提供的函数被调用。 这允许你做 时变抽样操作， 即抽样操作， 数量的分区， 广播变量， 批次之间等可以改变。

#### 窗口 (window) 操作

Spark Streaming 也支持窗口计算，它允许你在一个滑动窗口数据上应用 transformation 算子。下图阐明了这个滑动窗口

![streaming-dstream-window](/images/spark/streaming-dstream-window.png "streaming-dstream-window")

如上图显示，窗口在源 DStream 上滑动，合并和操作落入窗内的源 RDDs，产生窗口化的 DStream 的 RDDs。在这个具体的例子中，程序在三个时间单元的数据上进行窗口操作，并且每两个时间单元滑动一次。 这说明，任何一个窗口操作都需要指定两个参数：

- 窗口长度：窗口的持续时间
- 滑动的时间间隔：窗口操作执行的时间间隔

这两个参数必须是源 DStream 的批时间间隔的倍数。

下面举例说明窗口操作。例如，你想扩展 [前面的例子](/2016/12/24/spark-streaming-programming-guide/#section "/2016/12/24/spark-streaming-programming-guide/#section") 用来计算过去 30 秒的词频，间隔时间是 10 秒。为了达到这个目的，我们必须在过去 30 秒的 pairs DStream 上应用 reduceByKey 操作。用方法 reduceByKeyAndWindow 实现。

**Scala:**

```scala
// Reduce last 30 seconds of data, every 10 seconds
val windowedWordCounts = pairs.reduceByKeyAndWindow((a:Int,b:Int) => (a + b), Seconds(30), Seconds(10))
```

**Java:**

```java
// Reduce function adding two integers, defined separately for clarity
Function2<Integer, Integer, Integer> reduceFunc = new Function2<Integer, Integer, Integer>() {
  @Override public Integer call(Integer i1, Integer i2) {
    return i1 + i2;
  }
};

// Reduce last 30 seconds of data, every 10 seconds
JavaPairDStream<String, Integer> windowedWordCounts = pairs.reduceByKeyAndWindow(reduceFunc, Durations.seconds(30), Durations.seconds(10));
```

**Python:**

```python
# Reduce last 30 seconds of data, every 10 seconds
windowedWordCounts = pairs.reduceByKeyAndWindow(lambda x, y: x + y, lambda x, y: x - y, 30, 10)
```

一些常用的窗口操作如下所示，这些操作都需要用到上文提到的两个参数：窗口长度和滑动的时间间隔

| 转换 | 意义 |
| ------------- | ------------- |
| window (windowLength , slideInterval) | 返回一个新的 DStream 计算基于窗口的批 DStream 来源。 |
| countByWindow (windowLength , slideInterval) | 返回一个滑动窗口中的元素计算流。 |
| reduceByWindow (func, windowLength , slideInterval) | 返回一个新创建的单个元素流， 通过聚合元素流了 滑动时间间隔使用 函数 。 函数应该关联和交换， 以便它可以计算 正确地并行执行。 |
| reduceByKeyAndWindow (func, windowLength , slideInterval , ( numTasks]) | 当呼吁 DStream(K、V) 对， 返回一个新的 DStream(K、V) 对每个键的值在哪里聚合使用给定的 reduce 函数 函数 在一个滑动窗口批次。 注意: 默认情况下， 它使用引发的默认数量 并行任务 (2 为本地模式， 在集群模式是由配置数量 财产 spark.default.parallelism 分组)。 你可以通过一个可选的 numTasks 参数设置不同数量的任务。 |
| reduceByKeyAndWindow (func, invFunc , windowLength , slideInterval ,( numTasks]) | 上面的 reduceByKeyAndWindow() 的一个更有效的版本，其中每个窗口的 reduce 值是使用上一个窗口的 reduce 值递增计算的。 这是通过减少进入滑动窗口的新数据和 "反向减少" 离开窗口的旧数据来完成的。 一个例子是在窗口滑动时 "添加" 和 "减去" 键的计数。 然而，它仅适用于 "可逆缩减函数"，即，具有对应的 "逆缩减" 函数（作为参数 invFunc）的那些缩减函数。 像 reduceByKeyAndWindow 中一样，reduce 任务的数量可通过可选参数进行配置。 请注意，必须启用 [检查点](/2016/12/24/spark-streaming-programming-guide/#checkpointing "/2016/12/24/spark-streaming-programming-guide/#checkpointing") 设置才能使用此操作。 |
| countByValueAndWindow (windowLength , slideInterval ,[ numTasks]) | 当呼吁 DStream(K、V) 对， 返回一个新的 DStream(K, 长) 对的 每个键的值是它的频率在一个滑动窗口。 就像在 reduceByKeyAndWindow，通过一个减少任务的数量是可配置的 可选参数。 |

#### 连接操作

最后， Spark streaming 可以连接其它的数据源

##### Stream-stream 连接

stream 可以很容易与其他 stream 连接

**Scala:**

```scala
val stream1: DStream[String, String] = ...
val stream2: DStream[String, String] = ...
val joinedStream = stream1.join(stream2)
```

**Java:**

```java
JavaPairDStream<String, String> windowedStream1 = stream1.window(Durations.seconds(20));
JavaPairDStream<String, String> windowedStream2 = stream2.window(Durations.minutes(1));
JavaPairDStream<String, Tuple2<String, String>> joinedStream = windowedStream1.join(windowedStream2);
```

**Python:**

```python
stream1 = ...
stream2 = ...
joinedStream = stream1.join(stream2)
```

在每批间隔， 生成的抽样 stream1 将与生成的抽样 stream2。 也可以做 leftOuterJoin，rightOuterJoin，fullOuterJoin。 此外， 它通常是非常有用的做连接的窗口 (window) stream。 这是非常容易的。

也就是说： 时间间隔一致，统计不同时间窗口的 2 个 Dstream 情况。

随便举例，例如： 当前时间点，统计前 10s 内的订单 关联 20s 内的点击情况。 计算最近的 3 个点击。

**Scala:**

```scala
val windowedStream1 = stream1.window(Seconds(20))
val windowedStream2 = stream2.window(Minutes(1))
val joinedStream = windowedStream1.join(windowedStream2)
```
**Java:**

```java
JavaPairDStream<String, String> windowedStream1 = stream1.window(Durations.seconds(20));
JavaPairDStream<String, String> windowedStream2 = stream2.window(Durations.minutes(1));
JavaPairDStream<String, Tuple2<String, String>> joinedStream = windowedStream1.join(windowedStream2);
```

**Python:**

```python
windowedStream1 = stream1.window(20)
windowedStream2 = stream2.window(60)
joinedStream = windowedStream1.join(windowedStream2)
```

##### Stream-dataset 连接

这已经被证明在早些时候解释 DStream.transform 操作。 这是另一个例子， 加入一个有窗口的流数据集。

**Scala:**

```scala
val dataset: RDD[String, String] = ...
val windowedStream = stream.window(Seconds(20))...
val joinedStream = windowedStream.transform {rdd => rdd.join(dataset) }
```

**Java:**

```java
JavaPairRDD<String, String> dataset = ...
JavaPairDStream<String, String> windowedStream = stream.window(Durations.seconds(20));
JavaPairDStream<String, String> joinedStream = windowedStream.transform(
    new Function<JavaRDD<Tuple2<String, String>>, JavaRDD<Tuple2<String, String>>>() {
        @Override
        public JavaRDD<Tuple2<String, String>> call(JavaRDD<Tuple2<String, String>> rdd) {
            return rdd.join(dataset);
        }
    }
);
```

**Python:**

```python
dataset = ... # some RDD
windowedStream = stream.window(20)
joinedStream = windowedStream.transform(lambda rdd: rdd.join(dataset))
```

事实上， 你也可以动态地改变你想加入的数据集。 提供的函数 变换 评估每批间隔， 因此将使用当前数据集 作为参考点。
DStream 转换的完整列表可以在 API 文档。 Scala API， 看到 [DStream](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.DStream "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.DStream") 和 [PairDStreamFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions") 。 Java API， 明白了 [JavaDStream](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html "http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaDStream.html") 和 [JavaPairDStream](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html "http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaPairDStream.html") 。 Python API， 看到 [DStream](http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.DStream "http://spark.apache.org/docs/latest/api/python/pyspark.streaming.html#pyspark.streaming.DStream") 。

### DStreams 上的输出操作

输出操作允许 DStream 的操作推到如数据库、文件系统等外部系统中。因为输出操作实际上是允许外部系统消费转换后的数据，它们触发的实际操作是 DStream 转换。目前，定义了下面几种输出操作：

| Output Operation | Meaning |
| ------------- | ------------- |
| print() | 在 DStream 的每个批数据中打印前 10 条元素，这个操作在开发和调试中都非常有用。在 Python API 中调用 pprint()。 |
|saveAsObjectFiles(prefix, [suffix]) | 保存 DStream 的内容为一个序列化的文件 SequenceFile。每一个批间隔的文件的文件名基于 prefix 和 suffix 生成。"prefix-TIME_IN_MS[.suffix]"，在 Python API 中不可用。 |
| saveAsTextFiles(prefix, [suffix]) | 保存 DStream 的内容为一个文本文件。每一个批间隔的文件的文件名基于 prefix 和 suffix 生成。"prefix-TIME_IN_MS[.suffix]" |
| saveAsHadoopFiles(prefix, [suffix]) | 保存 DStream 的内容为一个 hadoop 文件。每一个批间隔的文件的文件名基于 prefix 和 suffix 生成。"prefix-TIME_IN_MS[.suffix]"，在 Python API 中不可用。 |
| foreachRDD(func) | 在从流中生成的每个 RDD 上应用函数 func 的最通用的输出操作。这个函数应该推送每个 RDD 的数据到外部系统，例如保存 RDD 到文件或者通过网络写到数据库中。需要注意的是，func 函数在驱动程序中执行，并且通常都有 RDD action 在里面推动 RDD 流的计算。 |

提示注意：在 foreachRDD(func) 的 func 中可以输出到 hdfs， 即使用 rdd.saveAsHadoopFile[F <: OutputFormat[K, V]](path: String) 操作

#### foreachRDD 设计模式的使用

dstream.foreachRDD 是一个强大的原生语法，发送数据到外部系统中。然而，明白怎样正确地、有效地用这个原生语法是非常重要的。下面几点介绍了如何避免一般错误。
经常写数据到外部系统需要建一个连接对象（例如到远程服务器的 TCP 连接），用它发送数据到远程系统。为了达到这个目的，开发人员可能不经意的在 Spark 驱动中创建一个连接对象，但是在 Spark worker 中 尝试调用这个连接对象保存记录到 RDD 中，如下:

**Scala:**

```scala
dstream.foreachRDD {rdd =>
  rdd.foreach {record =>
    val connection = createNewConnection()
    connection.send(record)
    connection.close()
  }
}
```

**Python:**

```python
def sendRecord(record):
    connection = createNewConnection()
    connection.send(record)
    connection.close()

dstream.foreachRDD(lambda rdd: rdd.foreach(sendRecord))
```

这是不正确的，因为这需要先序列化连接对象，然后将它从 driver 发送到 worker 中。这样的连接对象在机器之间不能传送。它可能表现为序列化错误（连接对象不可序列化）或者初始化错误（连接对象应该 在 worker 中初始化）等等。正确的解决办法是在 worker 中创建连接对象。

最后，这可以进一步通过再利用在多个 RDDS / 批次的连接对象进行了优化。可以保持连接对象的静态池比可以为多个批次 RDDS 被推到外部系统，从而进一步降低了开销被重用。

然而，这会造成另外一个常见的错误 - 为每一个记录创建了一个连接对象。例如：

**Scala:**

```scala
dstream.foreachRDD {rdd =>
  rdd.foreachPartition {partitionOfRecords =>
    // ConnectionPool is a static, lazily initialized pool of connections
    val connection = ConnectionPool.getConnection()
    partitionOfRecords.foreach(record => connection.send(record))
    ConnectionPool.returnConnection(connection)  // return to the pool for future reuse
  }
}
```

**Python:**

```python
def sendPartition(iter):
    # ConnectionPool is a static, lazily initialized pool of connections
    connection = ConnectionPool.getConnection()
    for record in iter:
        connection.send(record)
    # return to the pool for future reuse
    ConnectionPool.returnConnection(connection)

dstream.foreachRDD(lambda rdd: rdd.foreachPartition(sendPartition))
```

需要注意的是，池中的连接对象应该根据需要延迟创建，并且在空闲一段时间后自动超时。这样就获取了最有效的方式发生数据到外部系统。
其它需要注意的地方：

- 输出操作通过懒执行的方式操作 DStream，正如 RDD action 通过懒执行的方式操作 RDD。具体地看，RDD action 和 DStream 输出操作接收数据的处理。因此，如果你的应用程序没有任何输出操作或者 用于输出操作 dstream.foreachRDD()，但是没有任何 RDD action 操作在 dstream.foreachRDD() 里面，那么什么也不会执行。系统仅仅会接收输入，然后丢弃它们。
- 默认情况下，DStreams 输出操作是分时执行的，它们按照应用程序的定义顺序按序执行

### DataFrame 和 SQL 操作

你可以很容易地使用 [DataFrames 和 SQL](/2016/12/24/spark-sql-programming-guide/ "/2016/12/24/spark-sql-programming-guide/") Streaming 操作数据。 需要使用 SparkContext 或者正在使用的 StreamingContext 创建一个 SparkSession。 这样做的目的就是为了使得驱动程序可以在失败之后进行重启。 使用懒加载模式创建单例的 SparkSession 对象。 下面的示例所示。 在原先的 单词统计 程序的基础上进行修改，使用 DataFrames 和 SQL 生成 [单词统计](/2016/12/24/spark-streaming-programming-guide/#section "/2016/12/24/spark-streaming-programming-guide/#section")。 每个 RDD 转换为 DataFrame， 注册为临时表， 然后使用 SQL 查询。

**Scala([源码](https://github.com/apache/spark/blob/v2.0.2/examples/src/main/scala/org/apache/spark/examples/streaming/SqlNetworkWordCount.scala "Scala 源代码")):**

```scala
/** 流程序中的 DataFrame 操作 */

val words: DStream[String] = ...

words.foreachRDD {rdd =>

  // 获取单例的 SQLContext
  val sqlContext = SQLContext.getOrCreate(rdd.sparkContext)
  import sqlContext.implicits._

  // 将 RDD [String] 转换为 DataFrame
  val wordsDataFrame = rdd.toDF("word")

  // 注册临时表
  wordsDataFrame.registerTempTable("words")

  // 在 DataFrame 上使用 SQL 进行字计数并打印它
  val wordCountsDataFrame =
    sqlContext.sql("select word, count(*) as total from words group by word")
  wordCountsDataFrame.show()
}
```

**Java([源码](https://github.com/apache/spark/blob/v2.0.2/examples/src/main/java/org/apache/spark/examples/streaming/JavaSqlNetworkWordCount.java "JAVA 源代码")):**

```java
/** Java Bean class for converting RDD to DataFrame */
public class JavaRow implements java.io.Serializable {
  private String word;

  public String getWord() {
    return word;
  }

  public void setWord(String word) {
    this.word = word;
  }
}

...

/** DataFrame operations inside your streaming program */

JavaDStream<String> words = ...

words.foreachRDD(
  new Function2<JavaRDD<String>, Time, Void>() {
    @Override
    public Void call(JavaRDD<String> rdd, Time time) {

      // Get the singleton instance of SQLContext
      SQLContext sqlContext = SQLContext.getOrCreate(rdd.context());

      // Convert RDD[String] to RDD[case class] to DataFrame
      JavaRDD<JavaRow> rowRDD = rdd.map(new Function<String, JavaRow>() {
        public JavaRow call(String word) {
          JavaRow record = new JavaRow();
          record.setWord(word);
          return record;
        }
      });
      DataFrame wordsDataFrame = sqlContext.createDataFrame(rowRDD, JavaRow.class);

      // Register as table
      wordsDataFrame.registerTempTable("words");

      // Do word count on table using SQL and print it
      DataFrame wordCountsDataFrame =
          sqlContext.sql("select word, count(*) as total from words group by word");
      wordCountsDataFrame.show();
      return null;
    }
  }
);
```

**Python([源码](https://github.com/apache/spark/blob/v2.0.2/examples/src/main/python/streaming/sql_network_wordcount.py "Python 源代码")):**

```python
# Lazily instantiated global instance of SQLContext
def getSqlContextInstance(sparkContext):
    if ('sqlContextSingletonInstance' not in globals()):
        globals()['sqlContextSingletonInstance'] = SQLContext(sparkContext)
    return globals()['sqlContextSingletonInstance']

...

# DataFrame operations inside your streaming program

words = ... # DStream of strings

def process(time, rdd):
    print("========= %s =========" % str(time))
    try:
        # Get the singleton instance of SQLContext
        sqlContext = getSqlContextInstance(rdd.context)

        # Convert RDD[String] to RDD[Row] to DataFrame
        rowRdd = rdd.map(lambda w: Row(word=w))
        wordsDataFrame = sqlContext.createDataFrame(rowRdd)

        # Register as table
        wordsDataFrame.registerTempTable("words")

        # Do word count on table using SQL and print it
        wordCountsDataFrame = sqlContext.sql("select word, count(*) as total from words group by word")
        wordCountsDataFrame.show()
    except:
        pass

words.foreachRDD(process)
```

你可以在使用其他线程读取的流数据上进行 SQL 查询（就是说，可以异步运行 StreamingContext）。 只要确保 StreamingContext 可以缓存一定量的数据来满足查询的需求。 否则 StreamingContext， 检测不到任何异步 SQL 查询， 在完成查询之前将删除旧的数据。 例如， 如果您想查询最后一批， 但您的查询可以运行需要 5 分钟， 然后调用 streamingContext.remember(Minutes(5)) (在 Scala 中， 或其他语言)。
访问 [DataFrames 和 SQL](/2016/12/24/spark-sql-programming-guide/ "/2016/12/24/spark-sql-programming-guide/") 了解更多关于 DataFrames 指南。

### MLlib 操作

你还可以轻松地使用所提供的机器学习算法 [MLlib](http://spark.apache.org/docs/latest/ml-guide.html "http://spark.apache.org/docs/latest/ml-guide.html") 。 首先这些 streaming，(如机器学习算法。 [Streaming 线性回归](http://spark.apache.org/docs/latest/mllib-linear-methods.html#streaming-linear-regression "http://spark.apache.org/docs/latest/mllib-linear-methods.html#streaming-linear-regression")，[StreamingKMeans](http://spark.apache.org/docs/latest/mllib-clustering.html#streaming-k-means "http://spark.apache.org/docs/latest/mllib-clustering.html#streaming-k-means") 等) 可以同时学习 Streaming 数据的应用模型。 除了这些， 对于一个大得多的机器学习算法， 可以学习模型离线 (即使用历史数据)， 然后应用该模型在线流媒体数据。  访问 [MLlib](http://spark.apache.org/docs/latest/ml-guide.html "http://spark.apache.org/docs/latest/ml-guide.html") 指南更多细节。

### 缓存 / 持久性

类似于抽样， DStreams 还允许开发人员持久化 stream 数据在内存中。 也就是说， 使用 persist() 方法 DStream， DStream 会自动把每个抽样持续化到内存中 。 这个非常有用， 如果数据多次 DStream(如同样的数据进行多次操作)。 像 reduceByWindow、 reduceByKeyAndWindow 和 updateStateByKey 这些都隐式开启了 "persist()"。 因此， DStreams 生成的窗口操作会自动保存在内存中， 如果没有开发人员调用 persist() 。

对于通过网络接收数据的输入流 (如 Kafka、Flume、Sockets 等)， 默认的持久性级别被设置为复制两个节点的数据容错。
注意， 与抽样不同， 默认的序列化数据持久性 DStreams。 这是进一步讨论的 [性能调优](/2016/12/24/spark-streaming-programming-guide/#section-15 "/2016/12/24/spark-streaming-programming-guide/#section-15") 部分。 更多的不同的持久性信息中可以通过 [Spark 编程指南](/2016/12/24/spark-programming-guides/#rdd--2 "/2016/12/24/spark-programming-guides/#rdd--2") 找到标准。

### CheckPointing

一个 Streaming 应用程序必须全天候运行，所有必须能够解决应用程序逻辑无关的故障（如系统错误，JVM 崩溃等）。为了使这成为可能，Spark Streaming 需要 checkpoint 足够的信息到容错存储系统中， 以使系统从故障中恢复。

- Metadata checkpointing : 保存流计算的定义信息到容错存储系统如 HDFS 中。这用来恢复应用程序中运行 worker 的节点的故障。元数据包括
  - Configuration : 创建 Spark Streaming 应用程序的配置信息
  - DStream operations : 定义 Streaming 应用程序的操作集合
  - Incomplete batches : 操作存在队列中的未完成的批
- Data checkpointing : 保存生成的 RDD 到可靠的存储系统中，这在有状态 transformation（如结合跨多个批次的数据）中是必须的。在这样一个 transformation 中，生成的 RDD 依赖于之前批的 RDD，随着时间的推移，这个依赖链的长度会持续增长。在恢复的过程中，为了避免这种无限增长。有状态的 transformation 的中间 RDD 将会定时地存储到可靠存储系统中，以截断这个依赖链。

元数据 checkpoint 主要是为了从 driver 故障中恢复数据。如果 transformation 操作被用到了，数据 checkpoint 即使在简单的操作中都是必须的。

#### 启用 CheckPointing

应用程序在下面两种情况下必须开启 checkpoint

- 使用有状态的 transformation。如果在应用程序中用到了 updateStateByKey 或者 reduceByKeyAndWindow，checkpoint 目录必需提供用以定期 checkpoint RDD。
- 从运行应用程序的 driver 的故障中恢复过来。使用元数据 checkpoint 恢复处理信息。

注意，没有前述的有状态的 transformation 的简单流应用程序在运行时可以不开启 checkpoint。在这种情况下，从 driver 故障的恢复将是部分恢复（接收到了但是还没有处理的数据将会丢失）。 这通常是可以接受的，许多运行的 Spark Streaming 应用程序都是这种方式。

#### 如何配置 CheckPointing

在容错、可靠的文件系统（HDFS、s3 等）中设置一个目录用于保存 checkpoint 信息。着可以通过 streamingContext.checkpoint(checkpointDirectory) 方法来做。这运行你用之前介绍的 有状态 transformation。另外，如果你想从 driver 故障中恢复，你应该以下面的方式重写你的 Streaming 应用程序。

- 当应用程序是第一次启动，新建一个 StreamingContext，启动所有 Stream，然后调用 start() 方法
- 当应用程序因为故障重新启动，它将会从 checkpoint 目录 checkpoint 数据重新创建 StreamingContext

**Scala：**

```scala
// Function to create and setup a new StreamingContext
def functionToCreateContext(): StreamingContext = {
    val ssc = new StreamingContext(...)   // new context
    val lines = ssc.socketTextStream(...) // create DStreams
    ...
    ssc.checkpoint(checkpointDirectory)   // set checkpoint directory
    ssc
}

// Get StreamingContext from checkpoint data or create a new one
val context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext _)

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start()
context.awaitTermination()
```

**Java:**

```java
// Create a factory object that can create and setup a new JavaStreamingContext
JavaStreamingContextFactory contextFactory = new JavaStreamingContextFactory() {
  @Override public JavaStreamingContext create() {
    JavaStreamingContext jssc = new JavaStreamingContext(...);  // new context
    JavaDStream<String> lines = jssc.socketTextStream(...);     // create DStreams
    ...
    jssc.checkpoint(checkpointDirectory);                       // set checkpoint directory
    return jssc;
  }
};

// Get JavaStreamingContext from checkpoint data or create a new one
JavaStreamingContext context = JavaStreamingContext.getOrCreate(checkpointDirectory, contextFactory);

// Do additional setup on context that needs to be done,
// irrespective of whether it is being started or restarted
context. ...

// Start the context
context.start();
context.awaitTermination();
```

**Python:**

```python
# Function to create and setup a new StreamingContext
def functionToCreateContext():
    sc = SparkContext(...)   # new context
    ssc = new StreamingContext(...)
    lines = ssc.socketTextStream(...) # create DStreams
    ...
    ssc.checkpoint(checkpointDirectory)   # set checkpoint directory
    return ssc

# Get StreamingContext from checkpoint data or create a new one
context = StreamingContext.getOrCreate(checkpointDirectory, functionToCreateContext)

# Do additional setup on context that needs to be done,
# irrespective of whether it is being started or restarted
context. ...

# Start the context
context.start()
context.awaitTermination()
```

如果 checkpointDirectory 存在，上下文将会利用 checkpoint 数据重新创建。如果这个目录不存在，将会调用 functionToCreateContext 函数创建一个新的上下文，建立 DStream。 请看 [RecoverableNetworkWordCount](https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala "https://github.com/apache/spark/tree/master/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala") 例子。

除了使用 getOrCreate，开发者必须保证在故障发生时，driver 处理自动重启。只能通过部署运行应用程序的基础设施来达到该目的。在部署章节将有更进一步的讨论。

注意，RDD 的 checkpointing 有存储成本。这会导致批数据（包含的 RDD 被 checkpoint）的处理时间增加。因此，需要小心的设置批处理的时间间隔。在最小的批容量 (包含 1 秒的数据) 情况下，checkpoint 每批数据会显著的减少 操作的吞吐量。相反，checkpointing 太少会导致谱系以及任务大小增大，这会产生有害的影响。因为有状态的 transformation 需要 RDD checkpoint。默认的间隔时间是批间隔时间的倍数，最少 10 秒。它可以通过 dstream.checkpoint 来设置。典型的情况下，设置 checkpoint 间隔是 DStream 的滑动间隔的 5-10 大小是一个好的尝试。

### 累加器，广播变量和 Checkpoints

[累加器](/2016/12/24/spark-programming-guides/#accumulators- "/2016/12/24/spark-programming-guides/#accumulators-") 和 [广播变量](/2016/12/24/spark-programming-guides/#broadcast-variables- "/2016/12/24/spark-programming-guides/#broadcast-variables-") 是不能从 Spark Streaming 中恢复。 如果要使 checkpoint 可用，[累加器](/2016/12/24/spark-programming-guides/#accumulators- "/2016/12/24/spark-programming-guides/#accumulators-") 或 [广播变量](/2016/12/24/spark-programming-guides/#broadcast-variables- "/2016/12/24/spark-programming-guides/#broadcast-variables-")，需要使用懒加载的方式实例化这两种变量以至于他们能够可以在驱动程序失败重启之后进行再次实例化。 下面的示例所示。

**Scala（[源码](https://github.com/apache/spark/blob/v2.0.2/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala "Scala 源代码")）:**

```scala
object WordBlacklist {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordBlacklist = Seq("a", "b", "c")
          instance = sc.broadcast(wordBlacklist)
        }
      }
    }
    instance
  }
}

object DroppedWordsCounter {

  @volatile private var instance: Accumulator[Long] = null

  def getInstance(sc: SparkContext): Accumulator[Long] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.accumulator(0L, "WordsInBlacklistCounter")
        }
      }
    }
    instance
  }
}

wordCounts.foreachRDD((rdd: RDD[(String, Int)], time: Time) => {
  // Get or register the blacklist Broadcast
  val blacklist = WordBlacklist.getInstance(rdd.sparkContext)
  // Get or register the droppedWordsCounter Accumulator
  val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
  // Use blacklist to drop words and use droppedWordsCounter to count them
  val counts = rdd.filter {case (word, count) =>
    if (blacklist.value.contains(word)) {
      droppedWordsCounter += count
      false
    } else {
      true
    }
  }.collect()
  val output = "Counts at time" + time + " " + counts
})
```

**Java（[源码](https://github.com/apache/spark/blob/v2.0.2/examples/src/main/java/org/apache/spark/examples/streaming/JavaRecoverableNetworkWordCount.java "JAVA 源代码")）:**

```java
class JavaWordBlacklist {

  private static volatile Broadcast<List<String>> instance = null;

  public static Broadcast<List<String>> getInstance(JavaSparkContext jsc) {
    if (instance == null) {
      synchronized (JavaWordBlacklist.class) {
        if (instance == null) {
          List<String> wordBlacklist = Arrays.asList("a", "b", "c");
          instance = jsc.broadcast(wordBlacklist);
        }
      }
    }
    return instance;
  }
}

class JavaDroppedWordsCounter {

  private static volatile Accumulator<Integer> instance = null;

  public static Accumulator<Integer> getInstance(JavaSparkContext jsc) {
    if (instance == null) {
      synchronized (JavaDroppedWordsCounter.class) {
        if (instance == null) {
          instance = jsc.accumulator(0, "WordsInBlacklistCounter");
        }
      }
    }
    return instance;
  }
}

wordCounts.foreachRDD(new Function2<JavaPairRDD<String, Integer>, Time, Void>() {
  @Override
  public Void call(JavaPairRDD<String, Integer> rdd, Time time) throws IOException {
    // Get or register the blacklist Broadcast
    final Broadcast<List<String>> blacklist = JavaWordBlacklist.getInstance(new JavaSparkContext(rdd.context()));
    // Get or register the droppedWordsCounter Accumulator
    final Accumulator<Integer> droppedWordsCounter = JavaDroppedWordsCounter.getInstance(new JavaSparkContext(rdd.context()));
    // Use blacklist to drop words and use droppedWordsCounter to count them
    String counts = rdd.filter(new Function<Tuple2<String, Integer>, Boolean>() {
      @Override
      public Boolean call(Tuple2<String, Integer> wordCount) throws Exception {
        if (blacklist.value().contains(wordCount._1())) {
          droppedWordsCounter.add(wordCount._2());
          return false;
        } else {
          return true;
        }
      }
    }).collect().toString();
    String output = "Counts at time" + time + " " + counts;
  }
}
```

**Python（[源码](https://github.com/apache/spark/blob/v2.0.2/examples/src/main/python/streaming/recoverable_network_wordcount.py "Python 源代码")）:**

```python
def getWordBlacklist(sparkContext):
    if ('wordBlacklist' not in globals()):
        globals()['wordBlacklist'] = sparkContext.broadcast(["a", "b", "c"])
    return globals()['wordBlacklist']

def getDroppedWordsCounter(sparkContext):
    if ('droppedWordsCounter' not in globals()):
        globals()['droppedWordsCounter'] = sparkContext.accumulator(0)
    return globals()['droppedWordsCounter']

def echo(time, rdd):
    # Get or register the blacklist Broadcast
    blacklist = getWordBlacklist(rdd.context)
    # Get or register the droppedWordsCounter Accumulator
    droppedWordsCounter = getDroppedWordsCounter(rdd.context)

    # Use blacklist to drop words and use droppedWordsCounter to count them
    def filterFunc(wordCount):
        if wordCount[0] in blacklist.value:
            droppedWordsCounter.add(wordCount[1])
            False
        else:
            True

    counts = "Counts at time %s %s" % (time, rdd.filter(filterFunc).collect())

wordCounts.foreachRDD(echo)
```

### 应用程序部署

本节讨论部署 Spark Streaming 应用程序的步骤。

#### 要求

要运行一个 Spark Streaming 应用，你需要有以下几点。

- 有管理器的集群 - 这是任何 Spark 应用程序都需要的需求，详见 [部署指南](/2016/12/24/spark-cluster-overview/ "/2016/12/24/spark-cluster-overview/")。
- 将应用程序打为 jar 包 - 你必须编译你的应用程序为 jar 包。如果你用 [spark-submit](/2016/12/24/spark-submitting-applications/ "/2016/12/24/spark-submitting-applications/") 启动应用程序，你不需要将 Spark 和 Spark Streaming 打包进这个 jar 包。 如果你的应用程序用到了 [高级源](/2016/12/24/spark-streaming-programming-guide/#advanced-sources)（如 kafka，flume），你需要将它们关联的外部 artifact 以及它们的依赖打包进需要部署的应用程序 jar 包中。例如，一个应用程序用到了 TwitterUtils，那么就需要将 spark-streaming-twitter_2.10 以及它的所有依赖打包到应用程序 jar 中。
- 为 executors 配置足够的内存 - 因为接收的数据必须存储在内存中，executors 必须配置足够的内存用来保存接收的数据。注意，如果你正在做 10 分钟的窗口操作，系统的内存要至少能保存 10 分钟的数据。所以，应用程序的内存需求依赖于使用 它的操作。
- 配置 checkpointing - 如果 stream 应用程序需要 checkpointing，然后一个与 Hadoop API 兼容的容错存储目录必须配置为检查点的目录，流应用程序将 checkpoint 信息写入该目录用于错误恢复。更多信息见 [checkpointing](/2016/12/24/spark-streaming-programming-guide/#checkpointing "/2016/12/24/spark-streaming-programming-guide/#checkpointing")
- 配置应用程序 driver 的自动重启 - 为了自动从 driver 故障中恢复，运行流应用程序的部署设施必须能监控 driver 进程，如果失败了能够重启它。不同的 [集群管理器](/2016/12/24/spark-cluster-overview/#cluster-manager- "/2016/12/24/spark-cluster-overview/#cluster-manager-")，有不同的工具得到该功能
  - Spark Standalone：一个 Spark 应用程序 driver 可以提交到 [Spark 独立集群](/2016/12/24/spark-standalone/#spark- "/2016/12/24/spark-standalone/#spark-") 运行，也就是说 driver 运行在一个 worker 节点上。进一步来看，独立的集群管理器能够被指示用来监控 driver，并且在 driver 失败（或者是由于非零的退出代码如 exit(1)， 或者由于运行 driver 的节点的故障）的情况下重启 driver。更多信息见 [Spark Standalone guide](/2016/12/24/spark-standalone/ "/2016/12/24/spark-standalone/")
  - YARN：YARN 为自动重启应用程序提供了类似的机制。
  - Mesos： Mesos 可以用 [Marathon](https://github.com/mesosphere/marathon "https://github.com/mesosphere/marathon") 提供该功能
- 配置 write ahead logs - 在 Spark 1.2 中，为了获得极强的容错保证，我们引入了一个新的实验性的特性 - 预写日志（write ahead logs）。如果该特性开启，从 receiver 获取的所有数据会将预写日志写入配置的 checkpoint 目录。 这可以防止 driver 故障丢失数据，从而保证零数据丢失。这个功能可以通过设置配置参数 spark.streaming.receiver.writeAheadLogs.enable 为 true 来开启。然而，这些较强的语义可能以 receiver 的接收吞吐量为代价。这可以通过 并行运行多个 receiver 增加吞吐量来解决。另外，当预写日志开启时，Spark 中的复制数据的功能推荐不用，因为该日志已经存储在了一个副本在存储系统中。可以通过设置输入 DStream 的存储级别为 StorageLevel.MEMORY_AND_DISK_SER 获得该功能。

#### 升级应用程序代码

如果运行的 Spark Streaming 应用程序需要升级，有两种可能的方法

- 启动升级的应用程序，使其与未升级的应用程序并行运行。一旦新的程序（与就程序接收相同的数据）已经准备就绪，旧的应用程序就可以关闭。这种方法支持将数据发送到两个不同的目的地（新程序一个，旧程序一个）
- 首先，平滑的关闭（[StreamingContext.stop(...)](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.StreamingContext") 或 [JavaStreamingContext.stop(...)](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html "http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/streaming/api/java/JavaStreamingContext.html")）现有的应用程序。在关闭之前，要保证已经接收的数据完全处理完。然后，就可以启动升级的应用程序，升级 的应用程序会接着旧应用程序的点开始处理。这种方法仅支持具有源端缓存功能的输入源（如 flume，kafka），这是因为当旧的应用程序已经关闭，升级的应用程序还没有启动的时候，数据需要被缓存。

### 监控应用程序

除了 Spark 自己的 [监控功能](/2016/12/24/spark-monitoring/ "/2016/12/24/spark-monitoring/") 之外，针对 Spark Streaming 它也有一些其他的功能。当使用 StreamingContext 时，[Spark Web UI](/2016/12/24/spark-monitoring/#web- "/2016/12/24/spark-monitoring/#web-") 显示了一个额外的 Streaming 标签，它显示了有关正在运行的 Receivers（接收器）（是否 Receivers 处于活动状态，记录接受数量，接收器误差，等等）和已完成的 Batches（批处理时间，查询延迟，等等）的统计信息。这可以用于监控 Spark 应用程序的进度。

在 Web UI 中以下两个指标特别重要 :

- Processing Time（处理时间） - 用来处理每批数据的时间。
- Scheduling Delay（调度延迟） - 一批数据在队列中等待，直到上一批数据处理完成所需的时间。

如果批处理时间始终大于批的间隔 和 / 或者 队列延迟不断增加，那么说明系统不能尽可能快的处理 Batches，它们（Batches）正在被生成并且落后（处理），在这种情况下，可以考虑 [reduce](/2016/12/24/spark-streaming-programming-guide/#section-9 "/2016/12/24/spark-streaming-programming-guide/#section-9") 批处理时间。

Spark Streaming 应用程序处理的进度，也可以使用 [StreamingListener](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener") 接口监控，它允许你获取 Receiver 的状态以及批处理时间。注意，这是一个开发者 API 并且在将来它很可能被改进（即上报更多的信息）。

***

## 性能优化

为了在集群中获得 Spark Streaming 应用程序的最佳性能需要一些优化。这部分解释了一部分能够调整用来提升您应用程序性能的参数和配置。在一个较高的水平上，您需要考虑两件事情 :

1. 通过有效的利用群集资源来减少每批数据的处理时间。
2. 设置正确的 Batch 大小，这样的话当它们被接受时 Batch 数据能够被尽量快的处理（换言之，数据处理能够赶得上数据摄取）。

### 降低批处理的时间

有许多优化可以在 Spark 中来完成，使每批数据的处理时间最小化。这些都在 [优化指南](/2016/12/24/spark-tuning/ "/2016/12/24/spark-tuning/") 中详细讨论。本章介绍一些最重要的问题。

#### 数据接收的并行级别

通过网络（像 Kafka，Flume，socket，等等）接受数据需要数据反序列化然后存在 Spark 中。如果数据接收成为了系统中的瓶颈，则需要考虑并行的数据接收。注意，每个 Input DStream（输入流）创建一个接受单个数据流的单独的 Receiver（接收器）（运行在一个 Worker 机器上）。接受多个数据流因此可以通过创建多个 Input DStreams 以及配置他们去从数据源（S）的不同分区接收数据流来实现。例如，一个单一的 Kafka Input DStream 接收两个 Topic（主题） 的数据能够被拆分成两个 Kafka 输入流，每个仅接收一个 Topic。这将运行两个 Receiver（接收器），使得数据可以被并行接受，因此将提高整体吞吐量。这些多个 DStream 可以合并在一起以创建一个单独的 DStream。然后被用于在一个单独的 Input DStream 上的 Transformations（转换）可以在统一的流上被应用。按照以下步骤进行 :

**Scala :**

```scala
val numStreams = 5
val kafkaStreams = (1 to numStreams).map { i => KafkaUtils.createStream(...) }
val unifiedStream = streamingContext.union(kafkaStreams)
unifiedStream.print()
```

**Java :**

```java
int numStreams = 5;
List<JavaPairDStream<String, String>> kafkaStreams = new ArrayList<>(numStreams);
for (int i = 0; i < numStreams; i++) {
  kafkaStreams.add(KafkaUtils.createStream(...));
}
JavaPairDStream<String, String> unifiedStream = streamingContext.union(kafkaStreams.get(0), kafkaStreams.subList(1, kafkaStreams.size()));
unifiedStream.print();
```

**Python :**

```python
numStreams = 5
kafkaStreams = [KafkaUtils.createStream(...) for _ in range (numStreams)]
unifiedStream = streamingContext.union(*kafkaStreams)
unifiedStream.pprint()
```

应当考虑的另一个参数是 Receiver（接收器）的阻塞间隔，它通 过 [配置参数](/2016/12/24/spark-configuration/ "/2016/12/24/spark-configuration/") spark.streaming.blockInterval 来决定。对于大部分 Receiver（接收器） 来说，在存储到 Spark 的 Memory（内存）之前时接收的数据被合并成数据 Block（块）。每批 Block（块）的数量确定了将用于处理在一个类似 Map Transformation（转换）的接收数据的任务数量。每个 Receiver（接收器）的每个 Batch 的任务数量大约为（Batch 间隔 / Block 间隔）。例如，200ms 的 Block（块）间隔和每 2s 的 Batch 将创建 10 个任务。如果任务的数量过低（即，小于每台机器的 Core（CPU）数量），那么效率会很低，因为所有可用的 Core（CPU）将不会用来处理数据。以增加给定的 Batch 间隔的任务数量，降低该 Block（间隔）。然而，Block（块）间隔推荐的最低值约为 50ms，低于该推荐值的任务的运行开销可能是一个问题。

与多个 Input Streams / Receivers 接受数据的另一种方法是明确的 Repartition （重分区）输入的数据流（使用 inputStream.repartition（<partition（分区）的数量>））。这样在进一步处理之前就分发接收数据的 Batch 到群集中指定数量的机器上去。

#### 数据处理的并行级别

集群资源的利用率可能会很低，如果在任何计算阶段中并行任务数量的不是很多的话。例如，对于像 reduceByKey 和 reduceByKeyAndWindow 这样的分布式 Reduce 操作来说，默认的并行任务数量由 spark.default.parallelism [配置属性](/2016/12/24/spark-configuration/#spark- "/2016/12/24/spark-configuration/#spark-") 控制。您可以传递并行的的参数（请看 [PairDStreamFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.dstream.PairDStreamFunctions") 文档），或者设置 spark.default.parallelism [配置属性](/2016/12/24/spark-configuration/#spark- "/2016/12/24/spark-configuration/#spark-") 来改变默认值。

#### 数据序列化

可以通过调整序列化的格式来减少数据序列化的开销。在流式传输的情况下，有两种数据类型会被序列化。

- Input Data（输入的数据）: 默认情况下，通过 Receivers（接收器）接收的输入数据被存储在 Executor 的内存与 [StorageLevel.MEMORY_AND_DISK_SER_2](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.storage.StorageLevel$ "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.storage.StorageLevel$") 中。也就是说，数据被序列化成 Bytes（字节）以降低 GC 开销，以及被复制用于 Executor 的失败容错。此外，数据首先保存在 Memory（内存）中，如果内存不足已容纳所有用于流计算的输入数据将被溢出到硬盘上。这个序列化操作显然也需要开销 - Receiver（接收器）必须反序列化接收到的数据并且使用 Spark 的序列化格式重新序列化它们。
- 通过 Streaming 操作产生的持久的 RDDs : 通过流计算产生的 RDDs 可能被持久化在内存中。例如，Window（窗口）操作将数据持久化在内存中，因为他们可能被多次处理。然而，不像 Spark Core 默认的 [StorageLevel.MEMORY_ONLY](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.storage.StorageLevel$ "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.storage.StorageLevel$")，通过流计产生的持久化 RDDs 被使用 [StorageLevel.MEMORY_ONLY_SER](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.storage.StorageLevel$ "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.storage.StorageLevel$")（也就是序列化）存储，默认 GC 开销降至最低。

在这两种情况下，使用 Kryo 序列化能够减少 CPU 和 内存的开销。更多细节请看 [Spark 优化指南](/2016/12/24/spark-tuning/#section "/2016/12/24/spark-tuning/#section")。对于 Kyro 来说，考虑注册自定义的 Class，并且禁用 Object（对象）引用跟踪（在 [配置指南](/2016/12/24/spark-configuration/#compression-and-serialization "/2016/12/24/spark-configuration/#compression-and-serialization") 中看 Kyro 相关的配置）。

在特定的情况下，需要用于保留 Streaming 应用程序的数据量不是很大，这样也许是可行的，来保存反序列化的数据（两种类型）不用引起过度的 GC 开销。例如，如果您使用 Batch 的间隔有几秒钟并且没有 Window（窗口）操作，然后你可以通过显式地设置存储相应的级别来尝试禁用序列化保存数据。这将减少由于序列化的 CPU 开销，可能不需要太多的 GC 开销就能提升性能。

#### 任务启动开销

如果每秒任务启动的数据很高（比如，每秒 50 个任务或者更多），那么发送 Task（任务） 到 Slave 的负载可能很大，将很难实现 亚秒级 延迟。可以通过下面的改变来降低负载 :

- Execution Mode（运行模式）: 以 Standlone 模式或者 coarse-grained（粗粒度的） Mesos 模式运行 Spark 会比  fine-grained（细粒度的）Mesos 模式运行 Spark 获得更佳的任务启动时间。更详细的信息请参考 [Mesos 运行指南](/2016/12/24/spark-running-on-mesos/ "/2016/12/24/spark-running-on-mesos/")。

这些改变也许能减少批处理的时间（100s of milliseconds），因此亚秒级的 Batch 大小是可行的。

### 设置合理的批处理间隔

对于群集上运行的 Spark Streaming 应用程序来说应该是稳定的，系统应该尽可能快的处理接收到的数据。换句话说，批数据在生成后应该尽快被处理。对于应用程序来说无论这个是不是真的都可以通过在 Streaming Web UI 找到 [监控](/2016/12/24/spark-streaming-programming-guide/#section-7 "/2016/12/24/spark-streaming-programming-guide/#section-7") 的处理时间，批处理时间应该小于批间隔。

取决于流计算的性质，所用的批间隔可能在通过群集中一组固定资源上的应用程序持续的数据速率有显著的影响。例如，让我们考虑下更早的 WordCountNetwork 例子。对于一个特定的数据速率。系统可能能够保持每 2 秒报告一次单词统计（即，批间隔是 2 秒），而不是每 500 毫秒一次。因此批间隔需要被设置，使得线上的预期数据速率可持续。

要为您的应用程序找出一个合理的批大小是去使用一个保守的批间隔（例如，5~10 秒）和一个较低的数据速率来测试它。为了验证系统是否能够保持数据速率，您可以通过每个被处理的 Batch 来检查端到端的延迟情况（或者在 Spark Driver log4j 日志中看 "Total delay"，或者使用 [StreamingListener](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.streaming.scheduler.StreamingListener") 接口）。如果延迟与批大小相比较处于一个稳定的状态，那么系统是稳定的。否则，如果延迟继续增加，它意味着系统不能保持下去，因此它是不稳定的。一旦你有了一个稳定的配置，你可以去试着增加数据速率 和 / 或者  降低批大小。注意一个短暂的延迟增加是由于临时的数据速率增加可能会变好，只要延迟降低到一个比较低的值。（即，小于批大小）。

### 内存优化

调优 Spark 应用程序的内存使用情况和 GC 行为已经在 [调优指南](/2016/12/24/spark-tuning/#section-1 "/2016/12/24/spark-tuning/#section-1") 中详细讨论了。强烈推荐您阅读它。在这一章，我们讨论在 Spark Streaming 应用程序 Context 中指定的一些优化参数。

Spark Streaming 应用程序在群集中需要的 Memory（内存） 数据取决于使用的 Transformations（转换）上的类型行为。例如，如果您想要在最近 10 分钟的时候上使用 Window（窗口）函数，那么您的群集应有有足够的 Memory 以保存 10 分钟值的数据在内存中。或者您想要在大量的 keys 中使用 updateStateByKey，那么所需要的内存将会更高。与之相反，如果您想要去做一个简单的 map-filter-store 操作，那么所需的内存将会更少。

在一般情况下，从数据通过 Receiver（接收器）被接收时起使用 StorageLevel.MEMORY_AND_DISK_SER_2 存储，数据在内存中放不下时将拆分到硬盘中去。这样也许降低了 Streaming 应用程序的性能，因此建议为您的 Streaming 应用程序提供足够的内存。它最好去试一试，并且在小范围看看 Memory（内存）使用情况然后估算相应的值。

内存调优的另一个方面是垃圾收集。Streaming 应用程序需要低延迟，JVM 垃圾回收造成的大量暂停是不可取的。

这里有一些能够帮助你调整内存使用情况以及 GC 开销的参数 :

- Persistence Level of DStreams（DStream 的持久化级别）: 像前面所提到的 [数据序列化](/2016/12/24/spark-streaming-programming-guide/#section-12 "/2016/12/24/spark-streaming-programming-guide/#section-12") 部分，输入的数据和 RDDs 默认持久化为序列化的字节。和反序列化持久性相比，这样减少了内存使用和 GC 开销。启用 Kryo 序列化进一步减少了序列化大小和内存使用。进一步减少内存使用可以用压缩实现（请看 Spark 配置 spark.rdd.compress），付出的是 CPU 时间。
- Clearing old data（清除旧数据）: 默认情况下，所有的输入数据和通过 DStream transformation（转换）产生的持久化 RDDs 将被自动的清除。Spark Streaming 决定何时清除基于使用 transformation（转换）的数据。例如，如果你使用一个 10 分钟的 Window（窗口）操作，那么 Spark Streaming 将保存最近 10 分钟的数据，并积极扔掉旧的数据。数据也能够通过设置 streamingContext.remeber 保持更久（例如，交互式查询旧数据）。
- CMS Garbage Collector（CMS 垃圾回收器）: 使用并发的 mark-sweep GC 是强烈推荐的用于保持 GC 相关的暂停更低。即使知道并发的 GC 降低了整个系统处理的吞吐量。仍然建议使用，以获得更一致的批处理时间。确定你在 Driver（在 spark-submit 中使用 --driver-java-options）和 Executor（使用 [Spark 配置](/2016/12/24/spark-configuration/#section-2 "/2016/12/24/spark-configuration/#section-2") spark.executor.extraJavaOptions）上设置的 CMS GC。
- Other tips（其它建议）: 为了进一步降低 GC 开销，这里有些更多的建议可以尝试。
  - 持久化 RDDs 使用 OFF_HEAP 存储界别。更多详情请看 [Spark 编程指南](/2016/12/24/spark-programming-guides/#rdd--2)
  - 使用更多的 Executor 和更小的 heap size（堆大小）。这将在每个 JVM heap 内降低 GC 压力。

#### 应该记住的要点

- 一个 DStream 和一个单独的 Receiver（接收器）关联。为了达到并行的读取多个 Receiver（接收器）。例如，多个 DStreams 需要被创建。一个 Receiver 运行在一个 Executor 内。它占有一个 Core（CPU）。确保在 Receiver Slot 被预定后有足够的 Core 。例如，spark.cores.max 应该考虑 Receiver Slot。Receiver 以循环的方式被分配到 Executor。
- 当数据从一个 Stream 源被接收时，Receiver（接收器） 创建了数据块。每个块间隔的毫秒内产生一个新的数据块。N 个数据块在批间隔（N = 批间隔 / 块间隔）的时候被创建。这些 Block（块）通过当前的 Executor 的 BlockManager 发布到其它 Executor 的 BlockManager。在那之后，运行在 Driver 上的  Network Input Tracker 获取 Block 位置用于进一步处理。
- 一个 RDD 创建在 Driver 上，因为 Block 创建在 batchInterval（批间隔）期间。Block 在 batchInterval 划分成 RDD 时生成。每个分区是 Spark 中的一个任务。blockInterval == batchInterval 将意味着那是一个单独的分区被创建并且可能它在本地被处理过了。
- Block 上的 Map 任务在 Executor 中被处理（一个接收 Block，另一个 Block 被复制）无论块的间隔，除非非本地调度死亡。有更大的块间隔意味着更大的块。在本地节点上一个高的值 spark.locality.wait 增加处理 Blcok 的机会。需要发现一个平衡在这两个参数来确保更大的块被本都处理之间。
- 而不是依靠 batchInterval 和 blockInterval，你可以通过调用 inputDstream.repartition(n) 来定义分区的数量。这样会 reshuffles RDD 中的数据随机来创建 N 个分区。是的，为了更好的并行，虽然增加了 shuffle 的成本。一个 RDD 的处理通过 Driver 的 JobScheduler 作为一个 Job 来调度。在给定的时间点仅有一个 Job 是活跃的。所以，如果一个 Job 正在执行那么其它的 Job 会排队。
- 如果你有两个 DStream，那将有两个 RDDs 形成并且将有两个 Job 被创建，他们将被一个一个的调度。为了避免这个，你可以合并两个 DStream。这将确保两个 RDD 的 DStream 形成一个单独的 unionRDD。这个 unionRDD 被作为一个单独的 Job 考虑。然而分区的 RDDs 不受影响。
- 如果批处理时间超过了批间隔，那么显然 Receiver（接收器）的内存将开始填满，最走将抛出异常（最可能的是 BlockNotFoundException）。当前没有方法去暂停 Receiver。使用 SparkConf 配置  spark.streaming.receiver.maxRate，Receiver（接收器）的速率可以被限制。

***

## 容错语义

在这部分，我们将讨论 Spark Streaming 应用程序在发生故障时的行为。

### 背景

为了理解 Spark Streaming 提供的语义，让我们记住 Spark RDDs 最基本的容错语义。

1. RDD 是不可变的，确定重新计算的，分布式的 DataSet（数据集）。
2. 如果任何分区的 RDD 由于 Worker 节点失败而丢失，那么这个分区能够从原来原始容错的 DataSet（数据集）使用操作的继承关系来重新计算。
3. 假设所有的 RDD 转换是确定的，在最终转换的 RDD 中的数据总是相同的，无论 Spark 群集中的失败。

Spark 在数据上的操作像  HDFS 或者 S3 这样容错的文件系统一样。因此，所有从容错的数据中产生的 RDDs 也是容错的。然而，这种情况在 Spark Streaming 中不适用，因为在大多数情况下数据通过网络被接收（除非使用 fileStream）。为了对所有产生的 RDDs 实现相同的容错语义属性，接收的数据被复制到群集中 Worker 节点的多个 Spark Executor 之间（默认复制因子是 2）。这将会造成需要在发生故障时去恢复两种文件系统的数据类型 :

1. Data received and replicated（数据接收并且被复制）- 这份数据幸存于一个单独的 Worker 节点故障中因为其它的节点复制了它。
2. Data received but buffered for replication（数据接收但是缓冲了副本）- 因为这份数据没有被复制，唯一恢复这份数据的方式是从 Source（源 / 数据源）再次获取它。

此外，我们应该关注两种类型的故障 :

1. Failure of a Worker Node（Worker 节点的故障）- 任何运行 Executor 的 Worker 节点都是可以故障，并且这些节点所有在 Memory（内存）中的数据将会丢失。如果任何 Receiver（接收器）运行在故障的节点，那么他们缓存的数据将丢失。
2. Failure of the Driver Node（Driver 节点的故障）- 如果 Driver 节点运行的 Spark Streaming 应用程序发生故障，那么很显然 SparkContext 将会丢失，所有 Executor 与它们在内存中的数据都会丢失。

与这个基础知识一起，让我们理解 Spark Streaming 的容错语义。

#### 定义

Streaming 处理系统经常会捕获系统异常并记录执行次数以保障系统容错，其中在这三种条件下可以保障服务，等等。

1. 最多一次：每个记录将被处理一次或者根本不处理。
2. 至少一次：每个记录将被处理一次或多次。这主要是在最后一次次，因为它确保数据不会丢失。但也有可能是重复处理的。
3. 只有一次：每个记录将被处理一次，确保这一次数据完整。这显然是三者的最强保障。

#### 基础语义

在任何流处理系统，从广义上讲，处理数据有三个步骤。

1. 接收数据：接受数据或者从其他数据源接受数据。
2. 转换数据：所接收的数据使用 DSTREAM 和 RDD 变换转化。
3. 输出数据：最后变换的数据被推送到外部系统，如文件系统，数据库，DashBoard 等。

如果一个 Stream 应用程序来实现端到端的数据传输，则每个环节都需要一次性完整保障。也就是说，每个记录都必须被接收正好一次，恰好转化一次，被推到下游系统一次。让我们了解在 Spark Stream 的情况下这些步骤的语义。

1. 接收数据：不同的输入源提供不同的容错。详细过程在下一小节中讨论。
2. 转换数据：已经接收将被处理一次的所有数据，这要依靠于 RDDS 提供保证。即使有故障，只要接收到的输入数据是可读的，最终通过 RDDS 的转化将始终具有相同的内容。
3. 输出数据：定义了数据至少一次输出，因为它依赖于输出数据操作类型（是否支持转换）和定义的下游系统（是否支持事务）。但是，用户可以实现自己的传输机制，以实现只要一次定义。会在后面小节里有更详细的讨论。

#### 接收数据的语义

不同的输入源提供不同的保障，从至少一次到正好一次。阅读更多的细节。

##### 关于文件

如果所有的输入数据已经存在如 HDFS 的容错文件系统，Spark Stream 总是可以从任何故障中恢复所有数据。这种定义，在一次处理后就能恢复。

##### 关于基于 Receiver 的 Source（源）

对于基于 receiver 的输入源，容错的语义既依赖于故障的情形也依赖于 receiver 的类型。正如 [之前讨论](/2016/12/24/spark-streaming-programming-guide/#reveiver-reliability "/2016/12/24/spark-streaming-programming-guide/#reveiver-reliability") 的，有两种类型的 receiver

- Reliable Receiver：这些 receivers 只有在确保数据复制之后才会告知可靠源。如果这样一个 receiver 失败了，缓冲（非复制）数据不会被源所承认。如果 receiver 重启，源会重发数 据，因此不会丢失数据。
- Unreliable Receiver：当 worker 或者 driver 节点故障，这种 receiver 会丢失数据

选择哪种类型的 receiver 依赖于这些语义。如果一个 worker 节点出现故障，Reliable Receiver 不会丢失数据，Unreliable Receiver 会丢失接收了但是没有复制的数据。如果 driver 节点 出现故障，除了以上情况下的数据丢失，所有过去接收并复制到内存中的数据都会丢失，这会影响有状态 transformation 的结果。

为了避免丢失过去接收的数据，Spark 1.2 引入了一个实验性的特征（预写日志机制）write ahead logs，它保存接收的数据到容错存储系统中。有了 [write ahead logs](/2016/12/24/spark-streaming-programming-guide/#section-5 "/2016/12/24/spark-streaming-programming-guide/#section-5") 和 Reliable Receiver，我们可以 做到零数据丢失以及 exactly-once 语义。

下表总结了根据故障的语义：

| Deployment Scenario | Worker Failure | Driver Failure |
| ------------- | ------------- | ------------- |
| Spark 1.1 或者更早， 没有 write ahead log 的 Spark 1.2 | 在 Unreliable Receiver 情况下缓冲数据丢失；在 Reliable Receiver 和文件的情况下，零数据丢失 | 在 Unreliable Receiver 情况下缓冲数据丢失；在所有 receiver 情况下，过去的数据丢失；在文件的情况下，零数据丢失 |
| 带有 write ahead log 的 Spark 1.2 | 在 Reliable Receiver 和文件的情况下，零数据丢失 | 在 Reliable Receiver 和文件的情况下，零数据丢失 |

##### Kafka Direct API

在 Spark 1.3，我们引入了一个新的 kafka Direct 的 API，它可以保证所有 kafka 数据由 spark stream 收到一次。如果实现仅一次输出操作，就可以实现保证终端到终端的一次。这种方法（版本 Spark2.0.0）中进一步讨论 [kafka 集成指南](http://spark.apache.org/docs/latest/streaming-kafka-integration.html "http://spark.apache.org/docs/latest/streaming-kafka-integration.html")。

### 输出操作的语义

输出操作（例如 foreachRDD）至少被定义一次，即变换后的数据在一次人工故障的情况下可能会不止一次写入外部实体。虽然可以通过操作 saveAs***Files 保存文件到系统上（具有相同的数据将简单地被覆盖），额外的尝试可能是必要的，以实现一次准确的语义。有两种方法。

- 幂等更新：多次尝试写相同的数据。例如，saveAs***Files 总是写入相同数据生成的文件。
- 事务更新：所有更新事务作出这样的更新是恰好遵循原子性。要做到这一点， 如下 2 种方式。
  - 使用批处理时间创建一个标识符（foreachRDD）和 RDD 的分区索引。给这个应用定义唯一标识符标识 blob 数据。
  - 更新外部系统与当前事务（即原子性），如果已经被标识过得数据应用已经存在，那么就跳过更新。

```scala
dstream.foreachRDD {(rdd, time) =>
  rdd.foreachPartition {partitionIterator =>
    val partitionId = TaskContext.get.partitionId()
    val uniqueId = generateUniqueId(time.milliseconds, partitionId)
    // use this uniqueId to transactionally commit the data in partitionIterator
  }
}
```

***

## 迁移指南（从 0.9.1 或者更低版本至 1.x 版本）

在 Spark 0.9.1 和 Spark 1.0 之间，它们有一些 API 的改变以确保未来 API 的稳定性。这一章阐述需要迁移您已经在的代码到 1.0 版本的步骤。

### Input DSteams（输入流）

所有创建 Input Steam（输入流）的操作（例如，StreamingContext.socketStream，FlumeUtils.createStream，等等。）现在返回 InputDStream / ReceiverInputDStream (而不是 DStream) 的 Scala 版本，和 JavaInputDStream / JavaPairInputDStream /JavaReceiverInputDStream / JavaPairReceiverInputDStream (而不是 JavaDStream) 的 Java 版。这将确保特定输入流的功能可以添加到这些类中在未来不需要破坏二进制兼容性。注意您已经存在的 Spark Streaming 应用程序应该不需要任何改变（因为那些类是 DStream / JavaDStream 的子类），但可能需要使用 Spark 1.0 重新编译。

### Custom Network Receivers（自定义网络接收器）

从发布 Spark Streaming 以来，自定义网络接收器能够在 Scala 使用 NetworkReceiver 类自定义，该 API 在错误处理和报告中被限制了，并且不能够被 Java 使用。从 Spark 1.0 开始，自个类被 Receiver 替换了并且带来了下面的好处。

- 像 stop 和 restart 这样的方法已经被添加以更好的控制 Receiver（接收器）的生命周期。更多详细信息请看 自定义 Receiver 指南。
- 自定义 Receiver 能够使用 Scala 和 Java 实现。

为了迁移你已存在的自定义 Receiver，从更早的 NetworkReceiver 到新的 Receiver。您需要去做如下事情。

- 让您自定义的 Receiver 继承 org.apache.spark.streaming.receiver.Receiver 而不是 org.apache.spark.streaming.dstream.NetworkReceiver。
- 之前，一个 BlockGenerator 对象必须去通过自定义 Receiver 创建，它接收的数据被添加然后存在 Spark 中。它必须显示的从 onStart() 和 stop() 方法来启动和停止。新的 Receiver 类让这个是不必要的因为它添加了一组名为 store（<data>）的方法，能够在 Spark 中被调用以存储数据。所以，为了迁移你自定义的网络接收器，删除任何的 BlockGenerator 对象（在 Spark 1.0 中再也不存在），并且在接收数据时使用 storm(...) 方法。

### Actor-base Receivers（基于 Actor 的接收器）

这个基于 Actor 的接收器 API 已经被移到 DStream Akka 中去了。更多详细信息请参考该工程。
