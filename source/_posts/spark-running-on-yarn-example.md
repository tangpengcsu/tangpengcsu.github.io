---
layout: post
title: Spark on YARN 部署案例
date: 2016-12-24 12:39:04
categories:
  - Spark
  - Examples
tags:
  - Spark
  - Examples
---

## 环境准备

### 1. 服务器角色分配

| ip | hostname | role |
| ------------- | ------------- |
| 10.8.26.197 | server1 | 主名字节点 (NodeManager) |
| 10.8.26.196 | server2 | 备名字节点 (SecondaryNameNode) |
| 10.8.26.196 | server2 | 数据字节点 (DataNode) |

<!-- more -->

### 2. 软件设施

- jdk1.8.0_102
- scala-2.11.0:
- hadoop-2.7.0
- spark-2.0.2-bin-hadoop2.7：对应 scala 版本不能是 scala-2.11.x

### 3. HOSTS 设置

在每台服务器的 "/etc/hosts" 文件，添加如下内容：

```bash
10.8.26.197   server1   
10.8.26.196   server2  
10.8.26.196   server2
```

### 4. SSH 免密码登录

[Redhat 7/CentOS 7 SSH 免密登录](/2016/10/28/linux-ssh-authentication/ "{{ site.url }}/linux/linux-ssh-authentication/")

***

## Hadoop YARN 分布式集群配置

> 注：1-8 所有节点都做同样配置

### 1. 环境变量设置

```bash
# vim /etc/profile

# hadoop env set
export HADOOP_HOME=/usr/local/hadoop-2.7.0
export HADOOP_PID_DIR=/data/hadoop/pids
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"

export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME

export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HDFS_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop

export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native

# jdk env set
export JAVA_HOME=/usr/local/jdk1.8.0_102
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

# scala env set
export SCALA_HOME=/usr/local/scala-2.11.0
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$JAVA_HOME/bin:$SCALA_HOME/bin:$PATH
```

变量立即生效

```bash
# source /etc/profile
```

### 2. 相关路径创建

```bash
# mkdir -p /data/hadoop/{pids,storage}
# mkdir -p /data/hadoop/storage/{hdfs,tmp}
# mkdir -p /data/hadoop/storage/hdfs/{name,data}
```

### 3. 配置 core-site.xml

> 目录：$HADOOP_HOME/etc/hadoop/core-site.xml

```xml
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://server1:9000</value>
    </property>
    <property>
        <name>io.file.buffer.size</name>
        <value>131072</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/data/hadoop/storage/tmp</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hadoop.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.hadoop.groups</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.native.lib</name>
        <value>true</value>
    </property>
</configuration>
```

### 4. 配置 hdfs-site.xml

> 目录：$HADOOP_HOME/etc/hadoop/hdfs-site.xml

```xml
<configuration>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>server2:9000</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/data/hadoop/storage/hdfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/data/hadoop/storage/hdfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
        <name>dfs.webhdfs.enabled</name>
        <value>true</value>
    </property>

</configuration>
```

### 5. 配置 mapred-site.xml

> 目录：$HADOOP_HOME/etc/hadoop/mapred-site.xml

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>server1:10020</value>
    </property>

    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>server1:19888</value>
    </property>
</configuration>
```

### 6. 配置 yarn-site.xml

> 目录：$HADOOP_HOME/etc/hadoop/yarn-site.xml

```xml
<configuration>
<!-- Site specific YARN configuration properties -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
    <property>
        <name>yarn.resourcemanager.scheduler.address</name>
        <value>server1:8030</value>
    </property>
    <property>
        <name>yarn.resourcemanager.resource-tracker.address</name>
        <value>server1:8031</value>
    </property>
    <property>
        <name>yarn.resourcemanager.address</name>
        <value>server1:8032</value>
    </property>
    <property>
        <name>yarn.resourcemanager.admin.address</name>
        <value>server1:8033</value>
    </property>
    <property>
        <name>yarn.resourcemanager.webapp.address</name>
        <value>server1:80</value>
    </property>
</configuration>
```

### 7. 配置 hadoop-env.sh、mapred-env.sh、yarn-env.sh

> 均在文件开头添加

目录：

- $HADOOP_HOME/etc/hadoop/hadoop-env.sh
- $HADOOP_HOME/etc/hadoop/mapred-env.sh
- $HADOOP_HOME/etc/hadoop/yarn-env.sh

在以上三个文件开头添加如下内容：

```bash
export JAVA_HOME=/usr/local/jdk1.8.0_102
export CLASS_PATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib

export HADOOP_HOME=/usr/local/hadoop-2.7.0
export HADOOP_PID_DIR=/data/hadoop/pids
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="$HADOOP_OPTS -Djava.library.path=$HADOOP_HOME/lib/native"

export HADOOP_PREFIX=$HADOOP_HOME

export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME

export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HDFS_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop

export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native

export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

### 8. 数据节点配置

```bash
# vim $HADOOP_HOME/etc/hadoop/slaves
server1
server2
server3
```

### 9. Hadoop 简单测试

> 工作目录 master $HADOOP_HOME

```bash
# cd $HADOOP_HOME
```

首次启动集群时，做如下操作 [主名字节点上执行]

```bash
# hdfs namenode -format
# sbin/start-dfs.sh
# sbin/start-yarn.sh
```

检查进程是否正常启动

主名字节点 - server1：

```bash
# jps
11842 Jps
11363 ResourceManager
10981 NameNode
11113 DataNode
11471 NodeManager
```

备名字节点 - server2：

```bash
# jps
7172 SecondaryNameNode
7252 NodeManager
7428 Jps
7063 DataNode
```

数据节点 - server3：

```bash
# jps
6523 NodeManager
6699 Jps
6412 DataNode
```


hdfs 与 mapreduce 测试

```bash
#  hdfs dfs -mkdir -p /user/root
# hdfs dfs -put ~/text /user/root
# hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar wordcount /user/root /user/out
16/12/30 14:01:51 INFO client.RMProxy: Connecting to ResourceManager at server1/10.8.26.197:8032
16/12/30 14:01:55 INFO input.FileInputFormat: Total input paths to process : 1
16/12/30 14:01:55 INFO mapreduce.JobSubmitter: number of splits:1
16/12/30 14:01:56 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1483076482233_0001
16/12/30 14:01:58 INFO impl.YarnClientImpl: Submitted application application_1483076482233_0001
16/12/30 14:01:58 INFO mapreduce.Job: The url to track the job: http://server1:80/proxy/application_1483076482233_0001/
16/12/30 14:01:58 INFO mapreduce.Job: Running job: job_1483076482233_0001
16/12/30 14:02:23 INFO mapreduce.Job: Job job_1483076482233_0001 running in uber mode : false
16/12/30 14:02:24 INFO mapreduce.Job:  map 0% reduce 0%
16/12/30 14:02:36 INFO mapreduce.Job:  map 100% reduce 0%
16/12/30 14:02:44 INFO mapreduce.Job:  map 100% reduce 100%
16/12/30 14:02:45 INFO mapreduce.Job: Job job_1483076482233_0001 completed successfully
16/12/30 14:02:46 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=242
		FILE: Number of bytes written=230317
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=493
		HDFS: Number of bytes written=172
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=7899
		Total time spent by all reduces in occupied slots (ms)=6754
		Total time spent by all map tasks (ms)=7899
		Total time spent by all reduce tasks (ms)=6754
		Total vcore-seconds taken by all map tasks=7899
		Total vcore-seconds taken by all reduce tasks=6754
		Total megabyte-seconds taken by all map tasks=8088576
		Total megabyte-seconds taken by all reduce tasks=6916096
	Map-Reduce Framework
		Map input records=8
		Map output records=56
		Map output bytes=596
		Map output materialized bytes=242
		Input split bytes=99
		Combine input records=56
		Combine output records=16
		Reduce input groups=16
		Reduce shuffle bytes=242
		Reduce input records=16
		Reduce output records=16
		Spilled Records=32
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=231
		CPU time spent (ms)=1720
		Physical memory (bytes) snapshot=293462016
		Virtual memory (bytes) snapshot=4158427136
		Total committed heap usage (bytes)=139976704
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters
		Bytes Read=394
	File Output Format Counters
		Bytes Written=172
```

执行完成后查看输出，

```bash
# hdfs dfs -ls /user/out
Found 2 items
-rw-r--r--   3 root supergroup          0 2016-12-30 14:02 /user/out/_SUCCESS
-rw-r--r--   3 root supergroup        172 2016-12-30 14:02 /user/out/part-r-00000
```

也可以通过 UI （http://server1/cluster/apps） 查看：

![hadoop ui](/images/spark/hadoop-apps.png)

HDFS 信息查看

```bash
# hdfs dfsadmin -report
# hdfs fsck / -files -blocks
```

UI（http://server1:50070/） ：

![hdfs ui](/images/spark/hdfs-ui.png)

集群的后续维护

```bash
# sbin/start-all.sh
# sbin/stop-all.sh
```

监控页面 URL

http://10.8.26.197:80


## Spark 分布式集群配置

> 注：所有节点都做同样配置

### 1. Spark 相关配置

#### Spark 环境变量设置

```bash
# vim /etc/profile
# spark env set
export SPARK_HOME=/usr/local/spark-2.0.2-bin-hadoop2.7

export PATH=$SPARK_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$JAVA_HOME/bin:$SCALA_HOME/bin:$PATH
```

```bash
# source /etc/profile
```

#### 配置 spark-env.sh

```bash
# cd $SPARK_HOME/conf
# mv spark-env.sh.template spark-env.sh
# vim spark-env.sh
## 添加如下内容
export JAVA_HOME=/usr/local/jdk1.8.0_102
export SCALA_HOME=/usr/local/scala-2.11.0
export HADOOP_HOME=/usr/local/hadoop-2.7.0
```

#### 配置 worker 节点的主机名列表

```bash
# cd $SPARK_HOME/conf
```

```bash
# vim slaves
server1
server2
server3
```

#### 其他配置


```bash
# cd $SPARK_HOME/conf
# mv log4j.properties.template log4j.properties
```

### 在 Master 节点上执行

```bash
# cd $SPARK_HOME && sbin/start-all.sh
```

### 检查进程是否启动

在 master 节点上出现 "Master"，在 slave 节点上出现 "Worker"

#### Master 节点：

```bash
[root@server1 spark-2.0.2-bin-hadoop2.7]# jps
11363 ResourceManager
10981 NameNode
13176 Master
13256 Worker
11113 DataNode
13435 Jps
11471 NodeManager
```

#### Slave 节点：

```bash
[root@server2 conf]# jps
7172 SecondaryNameNode
7252 NodeManager
7063 DataNode
8988 Worker
9133 Jps
```

### 相关测试

监控页面 URL

http://10.8.26.197:8080/ 或 http://server1:8080/

![监控页面](/images/spark/spark-master.png)

切换到 "$SPARK_HOME/bin" 目录

1\. 本地模式

```bash
# .bin/run-example org.apache.spark.examples.SparkPi local
```

2\. 普通集群模式

```bash
# ./bin/run-example org.apache.spark.examples.SparkPi spark://10.8.26.197:7077
# ./bin/run-example org.apache.spark.examples.SparkLR spark://10.8.26.197:7077
```

3\. 结合 HDFS 的集群模式

> 工作目录 $SPARK_HOME

```bash
scala> val file=sc.textFile("hdfs://server1:9000/user/root/README.md")
file: org.apache.spark.rdd.RDD[String] = hdfs://server1:9000/user/root/README.md MapPartitionsRDD[5] at textFile at <console>:24

scala> val count = file.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_+_)
count: org.apache.spark.rdd.RDD[(String, Int)] = ShuffledRDD[8] at reduceByKey at <console>:26

scala> count.collect()
res3: Array[(String, Int)] = Array((package,1), (this,1), (Version"](http://spark.apache.org/docs/latest/building-spark.html#specifying-the-hadoop-version),1), (Because,1), (Python,2), (cluster.,1), (its,1), ([run,1), (general,2), (have,1), (pre-built,1), (YARN,,1), (locally,2), (changed,1), (locally.,1), (sc.parallelize(1,1), (only,1), (Configuration,1), (This,2), (basic,1), (first,1), (learning,,1), ([Eclipse](https://cwiki.apache.org/confluence/display/SPARK/Useful+Developer+Tools#UsefulDeveloperTools-Eclipse),1), (documentation,3), (graph,1), (Hive,2), (several,1), (["Specifying,1), ("yarn",1), (page](http://spark.apache.org/documentation.html),1), ([params]`.,1), ([project,2), (prefer,1), (SparkPi,2), (<http://spark.apache.org/>,1), (engine,1), (version,1), (file,1), (documentation...
scala> :quit
```

4\. 基于 YARN 模式

```bash
# ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
    --master yarn \
    --deploy-mode cluster \
    --driver-memory 1g \
    --executor-memory 1g \
    --executor-cores 1 \
     examples/jars/spark-examples*.jar \
    10
```

执行结果：  

`$HADOOP_HOME/logs/userlogs/application_*/container*_***/stdout`

或

http://10.8.26.197/logs/userlogs/application_1483076482233_0009/container_1483076482233_0009_01_000001/stdout

![监控页面](/images/spark/yarn-res.png)
