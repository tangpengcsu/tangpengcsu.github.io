---
layout: post
title: 构建 Spark
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---

## Apache Maven

构建 Apache Spark 参考了基于 Maven 的构建。构建 Spark 会使用 Maven ，而且需要 3.3.9 或者更新版本的 Maven 以及 Java 7 及以上版本。

<!-- more -->

### 设置 Maven 的内存使用率

想要使用更多内存，需要配置 Maven 设置中的 MAVEN_OPTS 项

```
export MAVEN_OPTS="-Xmx2g -XX : ReservedCodeCacheSize=512m"
```

在使用 Java 7 来进行编译时，您需要添加额外选项 “-XX : MaxPermSize=512M” 到 MAVEN_OPTS 项

如果不添加这些参数，那么您可能看到如下错误和警告 :

```bash
[INFO] Compiling 203 Scala sources and 9 Java sources to /Users/me/Development/spark/core/target/scala-2.11/classes...
[ERROR] PermGen space -> [Help 1]

[INFO] Compiling 203 Scala sources and 9 Java sources to /Users/me/Development/spark/core/target/scala-2.11/classes...
[ERROR] Java heap space -> [Help 1]

[INFO] Compiling 233 Scala sources and 41 Java sources to /Users/me/Development/spark/sql/core/target/scala-{site.SCALA_BINARY_VERSION}/classes...
OpenJDK 64-Bit Server VM warning :  CodeCache is full. Compiler has been disabled.
OpenJDK 64-Bit Server VM warning :  Try increasing the code cache size using -XX : ReservedCodeCacheSize=
```

您可以通过以上方法设置 MAVEN_OPTS 项来解决这一问题。

注意 :

- 如果使用 build/mvn 但是没有设置 MAVEN_OPTS，那么脚本会自动的将以上选项添加到 MAVEN_OPTS 环境变量
- 构建 Spark 的测试阶段将自动把这些选项添加到 MAVEN_OPTS ，即使不使用 build/mvn
- 在您使用 Java 8 和 build/mvn 来构建和运行测试时，您可能会看到类似于 "ignoring option MaxPermSize=1g; support was removed in 8.0" 的警告。这些警告无伤大雅。

### build / mvn

Spark 现在与一个独立的 Maven 安装包封装到了一起，使从源目录 build/ 下的原始位置来构建和部署 Spark 更加容易。这个脚本能自动在本地的 build/ 目录下载和安装所有需要的构建需求包，例如 Maven、Scala、Zinc。其尊重任何已经存在的 mvn 二进制文件，但是却无论如何将自己的 Scala 和 Zinc 的副本摧毁，来确保需求包的版本是合适的。build/mvn 的执行通过 mvn 调用允许过渡到之前的构建方法。例如，可以使用如下方法构建一个版本的 Spark :

```
./build/mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.4.0 -DskipTests clean package
```

其他的构建例子能在下文找到

## 构建一个可运行的 Distribution 版本

想要创建一个像是那些 Spark 下载页中的 Spark 的分布式版本，并且能使其运行，使用项目中 root 目录下的 ./dev/make-distribution.sh 脚本。它可以使用 Maven 的配置文件进行配置，如直接使用 Maven 构建。例如 :

```bash
./dev/make-distribution.sh --name custom-spark --tgz -Psparkr -Phadoop-2.4 -Phive -Phive-thriftserver -Pyarn
```

想要获取更详细的信息，可以运行

```bash
./dev/make-distribution.sh --help
```

## 指定 Hadoop 版本

因为 HDFS 并非跨版本协议兼容的，如果您想从 HDFS 读数据，那么您就需要构建与您的环境中 HDFS 版本相符的 Spark 。您可以通过设置属性 hadoop.version 来实现。如果不设置，那么 Spark 将默认按照 Hadoop 2.2.0 来进行构建。注意对于特定的 Hadoop 版本，需要某些特定的构造文件。

| Hadoop version | Profile required |
| ------------- | ------------- |
| 2.2.x | hadoop-2.2 |
| 2.3.x |	hadoop-2.3 |
| 2.4.x |	hadoop-2.4 |
| 2.6.x |	hadoop-2.6 |
| 2.7.x and later 2.x |	hadoop-2.7 |

您可以配置 yarn 的配置文件和可选设置 yarn.version 属性，如果其与 hadoop.version 不同的时候。 Spark 只支持 Yarn2.2.0 或者之后版本。

例如 :

```bash
# Apache Hadoop 2.2.X
./build/mvn -Pyarn -Phadoop-2.2 -DskipTests clean package

# Apache Hadoop 2.3.X
./build/mvn -Pyarn -Phadoop-2.3 -Dhadoop.version=2.3.0 -DskipTests clean package

# Apache Hadoop 2.4.X or 2.5.X
./build/mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=VERSION -DskipTests clean package

# Apache Hadoop 2.6.X
./build/mvn -Pyarn -Phadoop-2.6 -Dhadoop.version=2.6.0 -DskipTests clean package

# Apache Hadoop 2.7.X and later
./build/mvn -Pyarn -Phadoop-2.7 -Dhadoop.version=VERSION -DskipTests clean package

# Different versions of HDFS and YARN.
./build/mvn -Pyarn -Phadoop-2.3 -Dhadoop.version=2.3.0 -Dyarn.version=2.2.0 -DskipTests clean package
```

## 构建支持 Hive 和 JDBC 的支持

想要使 Spark SQL 集成 Hive 以及 JDBC 服务和 CLI ，请增加 -Phive 和 Phive-thriftserver 配置到您已经存在的配置选项。默认 Spark 绑定 Hive 1.2.1

```bash
# Apache Hadoop 2.4.X with Hive 1.2.1 support
./build/mvn -Pyarn -Phadoop-2.4 -Dhadoop.version=2.4.0 -Phive -Phive-thriftserver -DskipTests clean package
```

## 打包时排除针对 YARN 的 Hadoop 依赖

mvn package 生成的汇编目录，默认情况下将包括所有 Spark 的依赖，包括 Hadoop 和一些生态系统项目。在 YARN 的部署上，这会导致这些项目的许多版本都出现在执行目录中 : 包装在 Spark 组建在没个结点的版本，包括在属性 yarn.application.classpath 中。配置 hadoop-provided 可以在构建时不包括 Hadoop 生态项目，例如 ZooKeeper 和 Hadoop 本身。

## 构建适用于 Scala 2.10

如果希望使用 Scala 2.10 来编译 Spark 包，使用属性 -Dscala-2.10

```bash
./dev/change-scala-version.sh 2.10
./build/mvn -Pyarn -Phadoop-2.4 -Dscala-2.10 -DskipTests clean package
```

## 构建单个子模块

可以使用选项 mvn -pl 来构建 Spark 的子模块。

例如，您可以构建 Spark Streaming 通过 :

```bash
./build/mvn -pl  : spark-streaming_2.11 clean install
```
这里 spark-streaming_2.11 是在文件 streaming/pom.xml 中定义的 artifactId

## 连续的编译

我们使用插件 scala-maven-plugin 来支持增量和连续编译。例如 :

```bash
./build/mvn scala : cc
```

这里应该运行连续编译 (即等待变化)。然而这并没得到广泛的测试，有一些需要注意的陷阱 :

- 它只扫描路径 src/main 和 src/test ，所以其只对含有该结构的某些子模块起作用
- 您通常需要在工程的根目录运行 mvn install 在编译某些特定子模块时。这是因为依赖于其他子模块的子模块需要通过 spark-parent 模块实现。

因此，运行 core 子模块的连续编译总流程如下 :

```bash
$ ./build/mvn install
$ cd core
$ ../build/mvn scala : cc
```

## 用 Zinc 加速编译

Zinc 是 SBT 增量编译器的长期运行的服务器版本，当作为后台进程在本地运行时，它可有加速编译基于 Scala 的项目 : 如 Spark 。定期使用 Maven 重新编译 Spark 的开发人员会对 Zinc 最感兴趣。此章节将说明如何构建和运行 Zinc.OS X 用户可有通过 brew 安装 Zinc。

如果使用 bulid/mvn 软件包，ZInc 将会自动下载并且用于所有版本。 此进程将会在第一次 build/mvn 编译后自动开启并绑定到 3030 端口，除非自定义设置了环境变量 ZINC_PORT。可有在任何时候通过运行 build/zinc-<version>/bin/zinc -shutdown 来停止 Zinc 进程，但每当通过 build/mvn 编译的时候它又会重新启动。

## 通过 SBT 编译

Maven 是 Spark 官方推荐的编译工具。但是当 SBT 能够提供快速迭代编译的功能时，它逐渐被应用于日常开发。更多高级的开发者希望能够运用 SBT 。

SBT 通过 Maven POM 文件进行编译，因此也可以设置相同的 Maven 配置文件和变量来控制 SBT 的构建。 例如 :

```bash
./build/sbt -Pyarn -Phadoop-2.3 package
```

为了避免每次启动 SBT 都需要重新编译，可以通过运行 build/sbt 以交互模式来启动 SBT，然后在命令提示符下运行所有 build 命令。 有关减少构建时间的更多建议，请参阅 wiki 页面。

## 加密的文件系统

当构建在加密的文件系统上（例如，如果您的主目录被加密），那么 Spark 构建可能会失败，并出现 “Filename too long” 错误。 作为解决方法，在项目的 pom.xml 中的 scala-maven-plugin 的配置参数中添加以下代码 :

```xml
<arg>-Xmax-classfile-name</arg>
<arg>128</arg>
```
并在 project/SparkBuild.scala 中添加 :

```scala
scalacOptions in Compile ++= Seq("-Xmax-classfile-name", "128"),
```

关于 sharedSettings val，如果您不确定在哪里添加这些行，请参阅此 PR。

## IntelliJ IDEA 或者 Eclipse

有关设置 IntelliJ IDEA 或 Eclipse 以用来进行 Spark 开发和故障排除的帮助文档，请参阅 IDE 设置的 wiki 页面。

# 运行测试

默认情况下通过 ScalaTest Maven 组件进行测试。

一些测试需要首先打包 Spark ，因此，第一次总是使用 -DskipTests 运行 mvn package。 以下是正确（构建，测试）顺序的示例 :

```bash
./build/mvn -Pyarn -Phadoop-2.3 -DskipTests -Phive -Phive-thriftserver clean package
./build/mvn -Pyarn -Phadoop-2.3 -Phive -Phive-thriftserver test
```

ScalaTest 组件还支持运行特定的 Scala test 套件。如下 :

```bash
./build/mvn -P... -Dtest=none -DwildcardSuites=org.apache.spark.repl.ReplSuite test
./build/mvn -P... -Dtest=none -DwildcardSuites=org.apache.spark.repl.* test
```
或者一个 java 测试用例 :

```bash
./build/mvn test -P... -DwildcardSuites=none -Dtest=org.apache.spark.streaming.JavaAPISuite
```

## 使用 SBT 测试

一些测试需要首先打包 Spark，所以总是第一次总是运行 build/sbt。 以下是正确（构建，测试）顺序的示例 :

```bash
./build/sbt -Pyarn -Phadoop-2.3 -Phive -Phive-thriftserver package
./build/sbt -Pyarn -Phadoop-2.3 -Phive -Phive-thriftserver test
```
要运行特定的测试套件

```bash
./build/sbt -Pyarn -Phadoop-2.3 -Phive -Phive-thriftserver "test-only org.apache.spark.repl.ReplSuite"
./build/sbt -Pyarn -Phadoop-2.3 -Phive -Phive-thriftserver "test-only org.apache.spark.repl.*"
```

要运行特定子项目的测试套件，如下所示 :

```bash
./build/sbt -Pyarn -Phadoop-2.3 -Phive -Phive-thriftserver core/test
```

## 运行 JAVA 8 测试组件

只能运行 JAVA 8 测试组件，不能运行其他版本

```bash
./build/mvn install -DskipTests
./build/mvn -pl  : java8-tests_2.11 test
```

或者

```bash
./build/sbt java8-tests/test
```

当检测到 Java 8 JDK 时，将自动启用 Java 8 测试。 如果您安装了 JDK 8，但它不是系统默认值，则可以在运行测试之前将 JAVA_HOME 设置为指向 JDK 8 。

## 用 Maven 做 PySpark 测试

如果您正在构建 PySpark 并希望运行 PySpark 测试，您将需要使用 Hive 支持构建 Spark 。

```bash
./build/mvn -DskipTests clean package -Phive
./python/run-tests
```

运行测试脚本也可以限于特定的 Python 版本或特定的模块

```bash
./python/run-tests --python-executables=python --modules=pyspark-sql
```

## 运行 R 测试

要运行 SparkR 测试，您需要安装 R 包 testthat（从 R shell 运行 install.packages（testthat））。 您可以使用命令只运行 SparkR 测试 :

```bash
./R/run-tests.sh
```

注意 : 您还可以使用 SBT 构建运行 Python 测试，前提是您使用 Hive 支持构建 Spark 。

运行基于 Docker 集成的测试组件

为了运行 Docker 集成测试，您必须在您的盒子上安装 docker 引擎。 有关安装的说明，请访问 Docker 站点。 一旦安装，docker 服务需要启动，如果还没有运行。 在 Linux 上，这可以通过 sudo service docker start 来完成。

```bash
./build/mvn install -DskipTests
./build/mvn -Pdocker-integration-tests -pl  : spark-docker-integration-tests_2.11
```

或者

```bash
./build/sbt docker-integration-tests/test
```
