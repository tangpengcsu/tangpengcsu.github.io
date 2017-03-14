---
layout: post
title: Spark Standalone 模式案例
date: 2016-12-24 18:39:04
tags: [Spark, Examples]
categories: [Spark, Examples]
---


## 先决条件：

### 环境

- rhel 7.2
- jdk-8u102-linux-x64
- spark-2.0.2-bin-hadoop2.7
- Scala 2.11，注意：2.11.x 版本是不兼容的，见官网：[http://spark.apache.org/docs/latest/](http://spark.apache.org/docs/latest/)。

<!-- more -->

### 准备 master 主机和 worker 分机

- server1 机器：10.8.26.197，master
- server2 机器：10.8.26.196，worker
- server3 机器：10.8.26.195，worker

### 修改 host

```bash
[root@server1 ~]# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

10.8.26.197   server1
10.8.26.196   server2
10.8.26.195   server3
```

### 关闭所有节点机防火墙

```bash
# systemctl status firewalld
# systemctl stop firewalld
# systemctl disable firewalld
```

***

## 启动集群

### 主节点

```bash
./sbin/start-master.sh
```

查看输出日志：

```bash
cat logs/spark....
[root@server1 spark-2.0.2-bin-hadoop2.7]# cat logs/spark-root-org.apache.spark.deploy.master.Master-1-server1.out
Spark Command: /usr/local/jdk1.8.0_102/bin/java -cp /usr/local/spark-2.0.2-bin-hadoop2.7/conf/:/usr/local/spark-2.0.2-bin-hadoop2.7/jars/* -Xmx1g org.apache.spark.deploy.master.Master --host server1 --port 7077 --webui-port 8080
========================================
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
16/12/26 10:35:34 INFO Master: Started daemon with process name: 8671@server1
16/12/26 10:35:34 INFO SignalUtils: Registered signal handler for TERM
16/12/26 10:35:34 INFO SignalUtils: Registered signal handler for HUP
16/12/26 10:35:34 INFO SignalUtils: Registered signal handler for INT
16/12/26 10:35:35 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
16/12/26 10:35:35 INFO SecurityManager: Changing view acls to: root
16/12/26 10:35:35 INFO SecurityManager: Changing modify acls to: root
16/12/26 10:35:35 INFO SecurityManager: Changing view acls groups to:
16/12/26 10:35:35 INFO SecurityManager: Changing modify acls groups to:
16/12/26 10:35:35 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(root); groups with view permissions: Set(); users  with modify permissions: Set(root); groups with modify permissions: Set()
16/12/26 10:35:36 INFO Utils: Successfully started service 'sparkMaster' on port 7077.
16/12/26 10:35:36 INFO Master: Starting Spark master at spark://server1:7077
16/12/26 10:35:36 INFO Master: Running Spark version 2.0.2
16/12/26 10:35:36 INFO Utils: Successfully started service 'MasterUI' on port 8080.
16/12/26 10:35:36 INFO MasterWebUI: Bound MasterWebUI to 0.0.0.0, and started at http://10.8.26.197:8080
16/12/26 10:35:36 INFO Utils: Successfully started service on port 6066.
16/12/26 10:35:36 INFO StandaloneRestServer: Started REST server for submitting applications on port 6066
16/12/26 10:35:37 INFO Master: I have been elected leader! New state: ALIVE
```

通过 master-ip:8080 访问 master 的 web UI


![spark-setup](/images/spark/spark-setup.png)

### 各 worker 节点

```bash
./sbin/start-slave.sh spark://server1:7077
```

节点输出日志：

```bash
cat logs/spark....
[root@server2 spark-2.0.2-bin-hadoop2.7]# cat logs/spark-root-org.apache.spark.deploy.worker.Worker-1-server2.out
Spark Command: /usr/local/jdk1.8.0_102/bin/java -cp /usr/local/spark-2.0.2-bin-hadoop2.7/conf/:/usr/local/spark-2.0.2-bin-hadoop2.7/jars/* -Xmx1g org.apache.spark.deploy.worker.Worker --webui-port 8081 spark://server1:7077
========================================
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
16/12/26 10:43:04 INFO Worker: Started daemon with process name: 7466@server2
16/12/26 10:43:04 INFO SignalUtils: Registered signal handler for TERM
16/12/26 10:43:04 INFO SignalUtils: Registered signal handler for HUP
16/12/26 10:43:04 INFO SignalUtils: Registered signal handler for INT
16/12/26 10:43:05 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
16/12/26 10:43:05 INFO SecurityManager: Changing view acls to: root
16/12/26 10:43:05 INFO SecurityManager: Changing modify acls to: root
16/12/26 10:43:05 INFO SecurityManager: Changing view acls groups to:
16/12/26 10:43:05 INFO SecurityManager: Changing modify acls groups to:
16/12/26 10:43:05 INFO SecurityManager: SecurityManager: authentication disabled; ui acls disabled; users  with view permissions: Set(root); groups with view permissions: Set(); users  with modify permissions: Set(root); groups with modify permissions: Set()
16/12/26 10:43:06 INFO Utils: Successfully started service 'sparkWorker' on port 47422.
16/12/26 10:43:06 INFO Worker: Starting Spark worker 10.8.26.196:47422 with 1 cores, 1024.0 MB RAM
16/12/26 10:43:06 INFO Worker: Running Spark version 2.0.2
16/12/26 10:43:06 INFO Worker: Spark home: /usr/local/spark-2.0.2-bin-hadoop2.7
16/12/26 10:43:06 INFO Utils: Successfully started service 'WorkerUI' on port 8081.
16/12/26 10:43:06 INFO WorkerWebUI: Bound WorkerWebUI to 0.0.0.0, and started at http://10.8.26.196:8081
16/12/26 10:43:06 INFO Worker: Connecting to master server1:7077...
16/12/26 10:43:07 INFO TransportClientFactory: Successfully created connection to server1/10.8.26.197:7077 after 109 ms (0 ms spent in bootstraps)
16/12/26 10:43:07 INFO Worker: Successfully registered with master spark://server1:7077
```

通过 master-ip:8080 访问 master 的 web UI

![spark-setup](/images/spark/master-ui.png)

通过 worker-ip:8081 访问 worker 的 web UI

![spark-setup](/images/spark/worker-ui.png)

***

## 提交应用程序到集群

### 集成 shell 测试环境

#### 切换至 bin 目录

```bash
[root@server1 spark-2.0.2-bin-hadoop2.7]# cd bin
```

#### 进入运行在集群上的 spark 的 集成调试环境。

**Python**

```bash
[root@server1 bin]# ./pyspark --master spark://server1:7077
Python 2.7.5 (default, Nov  6 2016, 00:28:07)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-11)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel).
16/12/26 10:59:10 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.0.2
      /_/

Using Python version 2.7.5 (default, Nov  6 2016 00:28:07)
SparkSession available as 'spark'.
```

以统计文本行数为例：

```bash
>>> textFile=sc.textFile("../README.md")
>>> textFile.count()
99   
```

输出 README.md 中共有 99 行

**scala**

```bash
[root@server1 spark-2.0.2-bin-hadoop2.7]# ./bin/spark-shell --master spark://server1:7077
scala> val textFile=sc.textFile("README.md")
textFile: org.apache.spark.rdd.RDD[String] = README.md MapPartitionsRDD[9] at textFile at <console>:24

scala> textFile.count()
res0: Long = 99
```

可以在 master 的 web 界面里面看到任务执行情况

![spark-setup](/images/spark/scala-res.png)

也可以在 worker 的 web 界面里面看单个 worker 的情况

![spark-setup](/images/spark/scala-res-1.png)
