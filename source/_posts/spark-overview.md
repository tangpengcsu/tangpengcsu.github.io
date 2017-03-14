---
layout: post
title: Spark 概述
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
comments: true
image:
  feature: spark/spark-logo.png
---


Apache Spark 是一个快速的、多用途的集群计算系统。在 Java，Scala，Python 和 R 语言以及一个支持常见的图计算的经过优化的引擎中提供了高级 API。它还支持一组丰富的高级工具，包括用于 SQL 和结构化数据处理的 Spark SQL，用于机器学习的 [MLlib](http://spark.apache.org/docs/latest/ml-guide.html "http://spark.apache.org/docs/latest/ml-guide.html")，用于图形处理的 [GraphX](http://spark.apache.org/docs/latest/graphx-programming-guide.html "http://spark.apache.org/docs/latest/graphx-programming-guide.html") 以及 [Spark Streaming](/2016/12/24/spark-streaming-programming-guide/ "/2016/12/24/spark-streaming-programming-guide/")。

<!-- more -->

***

## 下载

从该项目官网的 [下载页面](http://spark.apache.org/downloads.html "http://spark.apache.org/downloads.html") 获取 Spark，该文档用于 Spark 2.0.2 版本。Spark 使用了用于 HDFS 和 YRAN 的 Hadoop client 的库。为了适用于主流的 Hadoop 版本可以下载先前的 package。用户还可以下载 "Hadoop free" binary 并且可以 通过增加 Spark 的 classpath 来与任何的 Hadoop 版本一起运行 Spark。

如果您希望从源码中构建 Spark，请访问 [构建 Spark](/2016/12/24/spark-building-spark/ "/2016/12/24/spark-building-spark/")。

Spark 既可以在 Windows 上运行又可以在类似 UNIX 的系统（例如，Linux，Mac OS）上运行。它很容易在一台机器上本地运行 - 您只需要在您的系统 PATH 上安装 Java，或者将 JAVA_HOME 环境变量指向一个 Java 安装目录。

Spark 可运行在 Java 7+，Python 2.6+/3.4 和 R 3.1+ 的环境上。 针对 Scala API，Spark 2.0.1 使用了 Scala 2.11。 您将需要去使用一个可兼容的 Scala 版本（2.11.x）。

***

## 运行示例和 Shell

Spark 自带了几个示例程序。 Scala，Java，Python 和 R 的示例在 examples/src/main 目录中。在最顶层的 Spark 目录中使用 bin/run-example <class> [params] 该命令来运行 Java 或者 Scala 中的某个示例程序。（在该例子的底层，调用了 spark-submit 脚本以启动应用程序 ）。 例如，

```bash
./bin/run-example SparkPi 10
```

您也可以通过一个改进版的 Scala shell 来运行交互式的 Spark。这是一个来学习该框架比较好的方式。

```bash
./bin/spark-shell --master local[2]
```

这个 --master 选项可以指定为 分布式集群中的 master URL，或者指定为 local 以使用 1 个线程在本地运行，或者指定为 local[N] 以使用 N 个线程在本地运行 。您应该指定为 local 来启动以便测试。该选项的完整列表，请使用 --help 选项来运行 Spark shell。

Spark 同样支持 Python API。在 Python interpreter（解释器）中运行交互式的 Spark，请使用 bin/pyspark :

```bash
./bin/pyspark --master local[2]
```

Python 中也提供了应用示例。例如，

```bash
./bin/spark-submit examples/src/main/python/pi.py 10
```

从 1.4 开始（仅包含了 DataFrames API）Spark 也提供了一个用于实验性的 R API。为了在 R interpreter（解释器）中运行交互式的 Spark，请执行 bin/sparkR :

```bash
./bin/sparkR --master local[2]
```

R 中也提供了应用示例。例如，

```bash
./bin/spark-submit examples/src/main/r/dataframe.R
```

***

## 在集群上运行

Spark 集群模式概述 说明了在集群上运行的主要的概念。Spark 既可以独立运行，也可以在几个已存在的 Cluster Manager（集群管理器）上运行。它当前提供了几种用于部署的选项 :

- [Spark Standalone 模式](/2016/12/24/spark-standalone/) : 在私有集群上部署 Spark 最简单的方式。
- [Spark on Mesos](/2016/12/24/spark-running-on-mesos/)
- [Spark on YARN](/2016/12/24/spark-running-on-yarn/)

***

## 快速跳转

### 编程指南 :

- [快速入门](http://spark.apache.org/docs/latest/quick-start.html) : 简单的介绍 Spark API，从这里开始！~
- [Spark 编程指南](/2016/12/24/spark-programming-guides/) : 在所有 Spark 支持的语言（Scala，Java，Python，R）中的详细概述。
- 构建在 Spark 之上的模块 :
  - [Spark Streaming](/2016/12/24/spark-streaming-programming-guide/) : 实时数据流处理。
  - [Spark SQL，Datasets，和 DataFrames](/2016/12/24/spark-sql-programming-guide/) : 支持结构化数据和关系查询。
  - [MLlib](http://spark.apache.org/docs/latest/ml-guide.html) : 内置的机器学习库。
  - [GraphX](http://spark.apache.org/docs/latest/graphx-programming-guide.html) : 新一代用于图形处理的 Spark API。

### API 文档:

- [Spark Scala API(Scaladoc)](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.package)
- [Spark Java API(Javadoc)](http://spark.apache.org/docs/latest/api/java/index.html)
- [Spark Python API(Sphinx)](http://spark.apache.org/docs/latest/api/python/index.html)
- [Spark R API(Roxygen2)](http://spark.apache.org/docs/latest/api/R/index.html)

### 部署指南:

- [集群模式概述](/2016/12/24/spark-cluster-overview/) : 在集群上运行时概念和组件的概述。
- [提交应用程序](/2016/12/24/spark-submitting-applications/) : 打包和部署应用。
- 部署模式 :
  - [Amazon EC2](https://github.com/amplab/spark-ec2) : 花费大约 5 分钟的时间让您在 EC2 上启动一个集群的介绍
  - [Spark Standalone 模式](/2016/12/24/spark-standalone/) : 在不依赖第三方 Cluster Manager 的情况下快速的启动一个独立的集群
    - [部署案例](/2016/12/24/examples/spark-standalone-example/ "/2016/12/24/examples/spark-standalone-example/")
  - [Spark on Mesos](/2016/12/24/spark-running-on-mesos/) : 使用 Apache Mesos 来部署一个私有的集群
  - [Spark on YARN](/2016/12/24/spark-running-on-yarn/) : 在 Hadoop NextGen（YARN）上部署 Spark
    - [部署案例](/2016/12/24/examples/spark-running-on-yarn-example/ "/2016/12/24/examples/spark-running-on-yarn-example/")

### 其他文件:

- [配置](/2016/12/24/spark-configuration/ "/2016/12/24/spark-configuration/"): 通过它的配置系统定制 Spark
- [监控](/2016/12/24/spark-monitoring/ "/2016/12/24/spark-monitoring/") : 监控应用程序的运行情况
- [优化指南](/2016/12/24/spark-tuning/ "/2016/12/24/spark-tuning/") : 性能优化和内存调优的最佳实践
- [作业调度](/2016/12/24/spark-job-scheduling/ "/2016/12/24/spark-job-scheduling/") : 资源调度和任务调度
- [安全性](/2016/12/24/spark-security/ "/2016/12/24/spark-security/") : Spark 安全性支持
- [硬件配置](/2016/12/24/spark-hardware-provisioning/ "/2016/12/24/spark-hardware-provisioning/") : 集群硬件挑选的建议
- 与其他存储系统的集成 :
  - [OpenStack Swift](http://spark.apache.org/docs/latest/storage-openstack-swift.html "http://spark.apache.org/docs/latest/storage-openstack-swift.html")
- [构建 Spark](/2016/12/24/spark-building-spark/ "/2016/12/24/spark-building-spark/") : 使用 Maven 来构建 Spark
- [Contributing to Spark](http://spark.apache.org/contributing.html "http://spark.apache.org/contributing.html")
- [Third Party Projects](http://spark.apache.org/third-party-projects.html "http://spark.apache.org/third-party-projects.html") : 其它第三方 Spark 项目的支持

### 外部资源:

- [Spark 主页](http://spark.apache.org/)
- [Spark Wiki](https://en.wikipedia.org/wiki/Apache_Spark)
- [Spark 社区](http://spark.apache.org/community.html) 资源，包括当地的聚会
- [StackOverflow tag apache-spark](http://stackoverflow.com/questions/tagged/apache-spark)
- [邮件列表](http://spark.apache.org/mailing-lists.html) : 在这里询问关于 Spark 的问题
- [AMP 营地](http://ampcamp.berkeley.edu/) 在加州大学伯克利分校: 一系列的训练营, 特色和讨论 练习对 Spark，Spark Steaming，Mesos 以及更多。可以免费通过 [视频](http://ampcamp.berkeley.edu/6/) , [幻灯片](http://ampcamp.berkeley.edu/6/) 和 [练习](http://ampcamp.berkeley.edu/6/exercises/) 学习。
- [代码示例](http://spark.apache.org/examples.html) : 更多示例可以在 Spark 的子文件夹中（Scala , Java , Python , R ）获得。
