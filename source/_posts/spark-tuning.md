---
layout: post
title: Spark 优化指南
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---

大多数 Saprk 计算本质是内存，Spark 程序可以碰到集群中的 CPU、网络带宽或存储 资源上的瓶颈。大多数情况下，如果数据加载到内存中，网络带宽就是瓶颈。但有时候，还需要做一些调整，比如 [序列化形式存储 RDD](/2016/12/24/spark-programming-guides/#rdd--2 "/2016/12/24/spark-programming-guides/#rdd--2") （storing RDDs in serialized），以减少内存使用情况。还涉及两个主要方面：原始数据的系列化，良好的网络性能，以减少内存使用情况。下面做几个主要的讨论。

<!-- more -->

## 数据序列化

序列化在任何分布式应用程序提高性能方面都是至关重要的思路。格式对象序列化对降低消耗大量的字节数，将大大减少计算量。通常情况下，这将是 Spark 调整以优化应用程序的最先考虑的。

Spark 开发目的就在方便（可以让您在操作中与任何的 Java Type 类型一起工作）和性能之间最大化。它提供了两个序列化库 :

- [Java serialization](http://docs.oracle.com/javase/6/docs/api/java/io/Serializable.html "http://docs.oracle.com/javase/6/docs/api/java/io/Serializable.html") : 默认情况下，Spark 序列化使用 Java 对象 ObjectOutputStream 框架，你创建了 [java.io.Serializable](http://docs.oracle.com/javase/6/docs/api/java/io/Serializable.html "http://docs.oracle.com/javase/6/docs/api/java/io/Serializable.html")。 实现类就可以工作，可以更有效的延伸序列化的性能 [java.io.Externalizable](http://docs.oracle.com/javase/6/docs/api/java/io/Externalizable.html "http://docs.oracle.com/javase/6/docs/api/java/io/Externalizable.html") 中。Java 序列化优点灵活，但速度慢，序列化，并导致许多格式 class。
- [Kryo serialization](https://github.com/EsotericSoftware/kryo "https://github.com/EsotericSoftware/kryo") : Spark 同样可以使用 Kryo 序列库（版本 2），Kryo 序列化比 java 序列化在速度上更快（一般在 10 倍左右） 缺点就是不支持所有 Serializable 类 ，但有时在项目中是比较好的性能提升。

你可以通过设置 conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer") 初始化 [SparkConf](/2016/12/24/spark-configuration/#spark- "/2016/12/24/spark-configuration/#spark-") 来转化成 Kryo 序列化此设置同样转化了 Spark 程序 Shuffling 内存数据和磁盘数据 RDDS 之间序列化，Kryo 不能成为默认方式的唯一原因是需要用户进行注册 在任何 "网络密集" 应用，跨语言支持较复杂。

Spark 底层自动把 Twitter chill 库中 AllScalaRegistrar Scala classes 包括在 Kryo 序列化中了 Kryo 注册的自定义类，使用 registerKryo 类的方法。

```bash
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
val sc = new SparkContext(conf)
```

[Kryo documentation](https://github.com/EsotericSoftware/kryo "https://github.com/EsotericSoftware/kryo") 提供更多高效的注册选项，比如添加自定义序列化。

针对对象是很大，你可能还需要增加的 spark.kryoserializer.buffer [配置](http://spark.apache.org/docs/latest/configuration.html#compression-and-serialization "http://spark.apache.org/docs/latest/configuration.html#compression-and-serialization")。该值对大量序列化有带来一定个性能提升最后，如果您没有注册自定义类，Kryo 仍然可以使用，但它有完整的类名称存储每个对象时，这是一种浪费...

## 内存优化

内存优化有三个方面的考虑 : 对象所占用的内存，访问对象的消耗以及垃圾回收所占用的开销。

默认情况下，Java 对象存取速度快，但可以很容易地比内部 “raw” 数据的字段的消耗的 2-5 倍以及更多空间。这是由于以下几个原因 :

- 每个不同的 Java 对象都有一个 “对象头” 大约是 16 个字节，并包含诸如一个指向它的类。并且包含了指向对象所对应的类的指针等信息。如果对象本身包含的数据非常少（比如就一个 Int 字段），那么对象头可能会比对象数据还要大。
- Java String 在实际的 raw 字符串数据之外，还需要大约 40 字节的额外开销（因为 String 使用一个 Char 数组来保存字符串，而且需要保存长度等额外数据）；同时，因为 String 在内部使用 UTF-16 编码，每一个字符需要占用两个字节，所以，一个长度为 10 字节的字符串需要占用 60 个字节。
- 常见的集合类，如 LinkedList 和 HashMap，使用 LinkedList 的数据结构，每条是一个 “warpper” 的对象（例如为 Map.Entry）。每一个条目不仅包含对象头，还包含了一个指向下一条目的指针（通常每个指针占 8 字节）。
- 基本类型的集合常常将它们存储为 “boxed” 对象 比如 java.lang.Integer 中的对象。

先介绍 Spark 内存管理的概述，然后再讨论具体策略使 "用户" 更有效地在应用去申请内存。特别是，我们将介绍对象的内存使用情况在哪里寻找，以及从哪些方面提升 - 改变你的数据结构，或存储的数据就是序列化格式。以及调整 Spark 缓存、Java 的垃圾收集器。

### 内存管理概述

Execution 与 Storge : 在 Spark 内存使用两类主要类型。execution 是指内存在 shuffles，joins，sorts 和 aggregations 的计算，而对于缓存存储器是指使用整个集群内部和数据。在 Spark，执行和存储共享统一区域（M）。当没有执行占用内存，存储可以获取所有的可用内存，反之亦然。如果必要的执行可能驱动存储，但只有等到总存储内存满足使用情况下一定的阈值（R）下降。换句话说，RM 区域 在哪里缓存块从未驱逐。存储可能无法执行复杂应用。

这种设计保证几个显著的特性。首先，应用程序不需要缓存整个执行到空间中，避免不必要的磁盘溢出。其次，应用程序都使用缓存，一个最小的可以保留的存储空间（R）模块，随着数据增加会别移出缓存中。最后，这种方法提供了现成合理的性能，适用于各种工作负载，而无需懂内部存储器划分的专业知识。

尽管提供两种相关配置，特殊用户一般是不会只用这两种配置作为适用于大多数的工作负载来调整的默认值 :

- spark.memory.fraction : 表示配置当前的内存管理器的最大内存使用比例，（默认 0.6）剩余 40% 部分被保留用于用户数据的结构，在 Spark 内部元数据，保障 OOM 错误，在异常大而稀疏的记录情况。
- spark.memory.storageFraction : 表示配置用于配置 rdd 的 storage 与 cache 的默认分配的内存池大小（默认值 0.5）。

spark.memory.fraction 值的配置不仅仅以调试的 JVM 堆空间或 “tenured” 设置。还有一些 GC 优化。

### 确定内存消耗

确定数据集所需内存量的最佳方法就是创建一个 RDD，把它放到缓存中，并查看网络用户界面 “Storage” 页面。该页面将显示有多少内存占用了 RDD。为了估算特定对象的内存消耗，使用 SizeEstimator 的方法估算内存使用情况，每个分区占用了多少内存量，合计这些内存占用量以确定 RDD 所需的内存总量。

### 数据结构优化

减少内存消耗的首要方式是避免了 Java 特性开销，如基于指针的数据结构和二次封装对象。有几个方法可以做到这一点 :

- 使用对象数组以及原始类型数组来设计数据结构，以替代 Java 或者 Scala 集合类（eg : HashMap）[fastutil](http://fastutil.di.unimi.it/ "http://fastutil.di.unimi.it/") 库提供了原始数据类型非常方便的集合类，同时兼容 Java 标准类库。
- 尽可能地避免使用包含大量小对象和指针的嵌套数据结构。
- 采用数字 ID 或者枚举类型以便替代 String 类型的主键。
- 假如内存少于 32GB，设置 JVM 参数 -XX:+UseCom­pressedOops 以便将 8 字节指针修改成 4 字节。将这个配置添加到 [spark-env.sh](/2016/12/24/spark-configuration/#section-5 "/2016/12/24/spark-configuration/#section-5") 中。

### 序列化 RDD 存储

当上面的优化都尝试过了对象同样很大。那么，还有一种减少内存的使用方法 “以序列化形式存储数据”，在 RDD 持久化 API 中（[RDD persistence API](/2016/12/24/spark-programming-guides/#rdd--2 "/2016/12/24/spark-programming-guides/#rdd--2")）使用序列化的 StorageLevel 例如 MEMORY_ONLY_SER 。 Spark 将每个 RDD 分区都保存为 byte 数组。序列化带来的唯一不足就是会降低访问速度，因为需要将对象反序列化（using Kryo）。如果需要采用序列化的方式缓存数据，我们强烈建议采用 Kryo，Kryo 序列化结果比 Java 标准序列化的更小（某种程度，甚至比对象内部的 raw 数据都还要小）。

### 垃圾回收优化

如果你需要不断的 “翻动” 程序保存的 RDD 数据，JVM 内存回收就可能成为问题（通常，如果只需进行一次 RDD 读取然后进行操作是不会带来问题的）。当需要回收旧对象以便为新对象腾内存空间时，JVM 需要跟踪所有的 Java 对象以确定哪些对象是不再需要的。需要记住的一点是，内存回收的代价与对象的数量正相关；因此，使用对象数量更小的数据结构（例如使用 int 数组而不是 LinkedList）能显著降低这种消耗。另外一种更好的方法是采用对象序列化，如上面所描述的一样；这样，RDD 的每一个 partition 都会保存为唯一一个对象（一个 byte 数组）。如果内存回收存在问题，在尝试其他方法之前，首先尝试使用 序列化缓存（serialized caching） 。

每项任务（task）的工作内存（运行 task 所需要的空间）以及缓存在节点的 RDD 之间会相互影响，这种影响也会带来内存回收问题。下面我们讨论如何为 RDD 分配空间以便减轻这种影响。

### 估算 GC 的影响

优化内存回收的第一步是获取一些统计信息，包括内存回收的频率、内存回收耗费的时间等。为了获取这些统计信息，我们可以把参数 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps 添加到 java 选项中（[配置指南](/2016/12/24/spark-configuration/#spark--1 "/2016/12/24/spark-configuration/#spark--1") 里面有关于传 java 选项参数到 Spark job 的信息）。设置完成后，Spark 作业运行时，我们可以在日志中看到每一次内存回收的信息。注意，这些日志保存在集群的工作节点（在他们工作目录下的 stout 文件中）而不是你的驱动程序（driver program )。

### 高级 GC 优化

为了进一步优化内存回收，我们需要了解 JVM 内存管理的一些基本知识。

- Java 堆（heap）空间分为两部分 : 新生代和老生代。新生代用于保存生命周期较短的对象；老生代用于保存生命周期较长的对象。
- 新生代进一步划分为三部分 [Eden，Survivor1，Survivor2]
- 内存回收过程的简要描述 : 如果 Eden 区域已满则在 Eden 执行 minor GC 并将 Eden 和 Survivor1 中仍然活跃的对象拷贝到 Survivor2 。然后将 Survivor1 和 Survivor2 对换。如果对象活跃的时间已经足够长或者 Survivor2 区域已满，那么会将对象拷贝到 Old 区域。最终，如果 Old 区域消耗殆尽，则执行 full GC 。

Spark 内存回收优化的目标是确保只有长时间存活的 RDD 才保存到老生代区域；同时，新生代区域足够大以保存生命周期比较短的对象。这样，在任务执行期间可以避免执行 full GC 。下面是一些可能有用的执行步骤 :

- 通过收集 GC 信息检查内存回收是不是过于频繁。如果在任务结束之前执行了很多次 full GC ，则表明任务执行的内存空间不足。
- 如果有过多的 minor GC 而不是 full GC，那么为 Eden 分配更大的内存是有益的。你可以为 Eden 分配大于任务执行所需要的内存空间。如果 Eden 的大小确定为 E，那么可以通过 -Xmn=4/3*E 来设置新生代的大小（将内存扩大到 4/3 是考虑到 survivor 所需要的空间)。
- 在打印的内存回收信息中，如果老生代接近消耗殆尽，那么减少用于缓存的内存空间。可这可以通过属性 spark.storage.memoryFraction 来完成。减少缓存对象以提高执行速度是非常值得的。
- 尝试设置 -XX:+UseG1GC 来使用垃圾回收器 G1GC 。在垃圾回收是瓶颈的场景使用它有助改善性能。当 executor 的 heap 很大时，使用 -XX:G1HeapRegionSize 增大 [G1 区](https://blogs.oracle.com/g1gc/entry/g1_gc_tuning_a_case "https://blogs.oracle.com/g1gc/entry/g1_gc_tuning_a_case") 大小很有必要。
- 举一个例子，如果任务从 HDFS 读取数据，那么任务需要的内存空间可以从读取的 block 数量估算出来。注意，解压后的 blcok 通常为解压前的 2-3 倍。所以，如果我们需要同时执行 3 或 4 个任务，block 的大小为 64M，我们可以估算出 Eden 的大小为 4*3*64MB。
- 监控内存回收的频率以及消耗的时间并修改相应的参数设置。

我们的经历表明有效的内存回收优化取决于你的程序和内存大小。 在网上还有很多 [更多其他调优选项](http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html "http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html") ， 总体而言有效控制内存回收的频率非常有助于降低额外开销。

executor 的 GC 调优标志位可以在 job 的配置中设置 spark.executor.extraJavaOptions 来指定。

## 其他优化

### 并行度级别

除非你为每步操作设置的 并行度 足够大，否则集群的资源是无法被充分利用的。Spark 自动根据文件大小设置运行在每个文件上的 map 任务的数量（虽然你可以通过 SparkContext.textFile 的可选参数来控制数量，等等），而且对于分布式 reduce 操作，例如 groupByKey 和 reduceByKey ，它使用最大父 RDD 的分区数。你可以通过第二个参数传入并行度（阅读文档 [spark.PairRDDFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.PairRDDFunctions")）或者通过设置系统参数 spark.default.parallelism 来改变默认值。通常来讲，在集群中，我们建议为每一个 CPU 核（core）分配 2-3 个任务。

### Reduce Tasks 的内存使用

有时，你会碰到 OutOfMemory 错误，这不是因为你的 RDD 不能加载到内存，而是因为 task 执行的数据集过大，例如正在执行 groupByKey 操作的 reduce 任务。Spark 的 shuffle 操作（sortByKey 、groupByKey 、reduceByKey 、join 等）为了实现 group 会为每一个任务创建哈希表，哈希表有可能非常大。最简单的修复方法是增加并行度，这样，每一个 task 的输入会变小。Spark 能够非常有效的支持短的 task（例如 200ms)，因为他会复用一个 executor 的 JVM 来执行多个 task，这样能减小 task 启动的消耗，所以你可以放心的增加任务的并行度到大于集群的 CPU 核数。

### 广播大的变量

使用 SparkContext 的 [广播功能](/2016/12/24/spark-programming-guides/#broadcast-variables- "/2016/12/24/spark-programming-guides/#broadcast-variables-") 可以有效减小每个序列化的 task 的大小以及在集群中启动 job 的消耗。如果 task 使用 driver program 中比较大的对象（例如静态查找表），考虑将其变成广播变量。Spark 会在 master 打印每一个 task 序列化后的大小，所以你可以通过它来检查 task 是不是过于庞大。通常来讲，大于 20KB 的 task 可能都是值得优化的。

### 数据本地性

数据本地性会对 Spark jobs 造成重大影响。如果数据和操作数据的代码在一起，计算就会变快。但如果代码和数据是分离的，其中一个必须移动到另一个。一般来说，移动序列化的代码比移动数据块来的快，因为代码大小远小于数据大小。Spark 根据该数据本地性的统一法则来构建 scheduling 计划。

数据本地性是指数据和操作该数据的代码有多近。根据数据的当前路径，有以下几个本地性级别。根据从近到远的数据排列 :

- PROCESS_LOCAL 数据跟代码在同一个 JVM。这个是最好的本地性。
- NODE_LOCAL 所有数据都在同一个节点。例如在同一个节点的 HDFS，或在同一个节点的另一个 executor 上。这个级别比 PROCESS_LOCAL 慢一点因为数据需要在多个 process（进程）间移动。
- NO_PREF 从任何地方访问数据都一样快，没有本地性的偏好。
- RACK_LOCAL 数据在同样的服务器机架上。数据在同一个机架的不同服务器上，所以需要通过网络传输，典型场景是通过单个交换机。
- ANY 数据在网络的不同地方，不咋同一个机架。

Spark 倾向于调度所有的 task 在最好的本地性级别，但未必总是行得通。如果所有空闲的 executor 都没有未处理的数据，Spark 就会切换到更低的本地性级别。

这样就有两个选择 :

1. 一直等待直到忙碌的 CPU 释放下来对同一个服务器的数据启动 task 。
2. 马上在远的、需要传输数据的地方启动一个 task。

Spark 典型做法是等待一段 timeout（超时）时间直到 CPU 释放资源。一旦 timeout 结束，它就开始移动数据到远处、有空闲 CPU 的地方。各个级别切换所等待的 timeout 时间可以单独配置或统一通过一个参数配置，需要更多细节可以查看 [配置页面](/2016/12/24/spark-configuration/#scheduling "/2016/12/24/spark-configuration/#scheduling") 的 spark.locality 参数。如果你的 task 看起来很长而且本地性差，就要考虑增加这些设置值，但默认设置一般都运行良好。

## 总结

该文指出了 Spark 程序优化所需要关注的几个关键点 - 最主要的是 数据序列化 和 内存优化 。对于大多数程序而言，采用 Kryo 序列化以及以序列化方式存储数据能够解决大部分性能问题。非常欢迎在 [Spark mailing list](https://spark.apache.org/community.html "https://spark.apache.org/community.html") 提问优化相关的问题。
