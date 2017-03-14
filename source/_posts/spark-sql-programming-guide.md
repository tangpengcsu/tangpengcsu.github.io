---
layout: post
title: "Spark SQL"
date: 2016-12-24 18:39:04
tags: [Spark]
categories: [Spark]
---


## Spark SQL 概述 (DataFrames, Datasets 和 SQL)

Spark SQL 是一个用户结构化数据处理的 Spark 模块。不像基础的 Spark RDD API 一样，Spark SQL 提供的接口提供了 Spark 与构造数据和执行计算之间的更多信息。在内部，Spark SQL 使用这个额外的信息去执行额外的优化。有几种方式可以跟 Spark SQL 进行交互，包括 SQL 和 Dataset API。当使用相同执行引擎进行计算时，无论使用哪种 API / 语言，都可以快速的计算。这种统一意味着开发人员能够在基于提供最自然的方式来表达一个给定的 transformation API 之间实现轻松的来回切换不同的 。

该页面所有例子使用的示例数据都包含在 Spark 的发布中，并且可以使用 spark-shell，pyspark，或者 sparkR 来运行。

<!-- more -->

### SQL

Spark SQL 的功能之一是执行 SQL 查询。Spark SQL 也能够被用于从已存在的 Hive 环境中读取数据。更多关于如何配置这个特性的信息，请参考 [Hive 表](/2016/12/24/spark-sql-programming-guide/#hive- "/2016/12/24/spark-sql-programming-guide/#hive-") 这部分。当以另外的编程语言运行 SQL 时，查询结果将以 [Dataset/DataFrame](/2016/12/24/spark-sql-programming-guide/#datasets--dataframes "/2016/12/24/spark-sql-programming-guide/#datasets--dataframes") 的形式返回。您也可以使用 [命令行](/2016/12/24/spark-sql-programming-guide/#spark-sql-cli "/2016/12/24/spark-sql-programming-guide/#spark-sql-cli") 或者通过 [JDBC/ODBC](/2016/12/24/spark-sql-programming-guide/#thrift-jdbcodbc-server "/2016/12/24/spark-sql-programming-guide/#thrift-jdbcodbc-server") 与 SQL 接口交互。

### Datasets 和 DataFrames

一个 Dataset 是一个分布式的数据集合。Dataset 是在 Spark 1.6 中被添加的新接口，它提供了 RDD 的好处（强类型化, 能够使用强大的 lambda 函数）与 Spark SQL 优化的执行引擎的好处。一个 Dataset 可以从 JVM 对象来 [构造](/2016/12/24/spark-sql-programming-guide/#dataset "/2016/12/24/spark-sql-programming-guide/#dataset") 并且使用转换功能（map，flatMap，filter，等等）。Dataset API 在 [Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset") 和 [Java](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html "http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html") 中是可用的。Python 不支持 Dataset API。但是由于 Python 的动态特性，许多 Dataset API 的有点已经可用了（也就是说，你可能通过 name 天生的 row.columnName 属性访问一行中的字段）。这种情况和 R 相似。

一个 DataFrame 是一个 Dataset 组织成的指定列。它的概念与一个在关系型数据库或者在 R/Python 中的表是相等的，但是有更多的优化。DataFrame 可以从大量的 [Source](/2016/12/24/spark-sql-programming-guide/#section "/2016/12/24/spark-sql-programming-guide/#section") 中构造出来，像 : 结构化的数据文件，Hive 中的表，外部的数据库，或者已存在的 RDD。DataFrame API 在 Scala，Java，[Python](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame "http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame") 和 [R](http://spark.apache.org/docs/latest/api/R/index.html "http://spark.apache.org/docs/latest/api/R/index.html") 中是可用的。在 Scala 和 Java 中，一个 DataFrame 所代表的是一个  Dataset 的 Row（行）。在 [Scala API](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset") 中，DataFrame 仅仅是一个 Dataset[Row] 类型的别名 。然而，在 [Java API](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html "http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html") 中，用户需要去使用 Dataset<Row> 来表示 DataFrame。

在这个文档中，我们将常常会引用 Scala / Java  的 Dataset 的 Row（行）作为 DataFrame。

***

## Spark SQL 入门指南

### 起始点 : SparkSession

Spark 中所有功能的入口点是 [SparkSession](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.SparkSession "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.SparkSession") 类。去创建一个基本的 SparkSession，仅使用 SparkSession.builder() :

**Scala**

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("Spark SQL Example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()

// For implicit conversions like converting RDDs to DataFrames
import spark.implicits._

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 中找到。
```

**Java**

```java
import org.apache.spark.sql.SparkSession;

SparkSession spark = SparkSession
  .builder()
  .appName("Java Spark SQL Example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate();

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 中找到。
```

**Python**

```python
from pyspark.sql import SparkSession

spark = SparkSession\
    .builder\
    .appName("PythonSQL")\
    .config("spark.some.config.option", "some-value")\
    .getOrCreate()

# 所有的示例代码可以在 Spark repo 的 "examples/src/main/python/sql.py" 中找到。
```

**R**

```r
sparkR.session(appName = "MyApp", sparkConfig = list(spark.executor.memory = "1g"))

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/r/RSparkSQLExample.R" 中找到。
// 注意 : 当第一次调用时，sparkR.session() 初始化了一个全局的 SparkSession 单例实例，并且为了连续的调用总是返回一个指向这个实力的引用。
// 用这种方式，用户只需要去初始化 SparkSession 一次，然后 SparkR 函数可以像 read.df 一样可以隐式的访问这个全局实例，并且用户不需要通过 SparkSession 实例访问。
```

在 Spark 2.0 中 SparkSession 为 Hive 特性提供了内嵌的支持，包括使用 HiveQL 编写查询的能力，访问 Hive UDF，以及从 Hive 表中读取数据的能力。为了使用这些特性，你不需要去有一个已存在的 Hive 设置。

### 创建 DataFrame

与一个 SparkSession 一起，应用程序可以从一个 [已存在的 RDD](/2016/12/24/spark-sql-programming-guide/#rdd- "/2016/12/24/spark-sql-programming-guide/#rdd-")，或者一个 Hive 表中，或者从 Spark [数据源](/2016/12/24/spark-sql-programming-guide/#section "/2016/12/24/spark-sql-programming-guide/#section") 中创建 DataFrame。
举个例子，下面基于一个 JSON 文件的内容创建一个 DataFrame :

**Scala**

```scala
val df = spark.read.json("examples/src/main/resources/people.json")

// Displays the content of the DataFrame to stdout
df.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 中找到。
```

**Java**

```java
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

Dataset<Row> df = spark.read().json("examples/src/main/resources/people.json");

// Displays the content of the DataFrame to stdout
df.show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 中找到。
```

**Python**

```python
# spark is an existing SparkSession
df = spark.read.json("examples/src/main/resources/people.json")

# Displays the content of the DataFrame to stdout
df.show()
```

**R**

```r
df <- read.json("examples/src/main/resources/people.json")

# Displays the content of the DataFrame
head(df)

# Another method to print the first few rows and optionally truncate the printing of long values
showDF(df)

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/r/RSparkSQLExample.R" 中找到。
```

### 无类型 Dataset 操作（aka DataFrame 操作）

DataFrame 提供了一个 DSL（domain-specific language）用于在 [Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset")，[Java](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html "http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/sql/Dataset.html")，[Python](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame "http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.DataFrame") 或者 [R](http://spark.apache.org/docs/latest/api/R/SparkDataFrame.html "http://spark.apache.org/docs/latest/api/R/SparkDataFrame.html") 中的结构化数据操作。

正如上面提到的一样，Spark 2.0 中 DataFrame 在 Scala 和 JavaAPI 中仅仅 Dataset 的 RowS（行）。这些操作也参考了与强类型的 Scala/Java Datasets 的 "类型转换" 相对应的 "无类型转换"。

这里包括一些使用 Dataset 进行结构化数据处理的示例 :

**Scala**

```scala
// This import is needed to use the $-notation
import spark.implicits._
// Print the schema in a tree format
df.printSchema()
// root
// |-- age: long (nullable = true)
// |-- name: string (nullable = true)

// Select only the "name" column
df.select("name").show()
// +-------+
// |   name|
// +-------+
// |Michael|
// |   Andy|
// | Justin|
// +-------+

// Select everybody, but increment the age by 1
df.select($"name", $"age" + 1).show()
// +-------+---------+
// |   name|(age + 1)|
// +-------+---------+
// |Michael|     null|
// |   Andy|       31|
// | Justin|       20|
// +-------+---------+

// Select people older than 21
df.filter($"age"> 21).show()
// +---+----+
// |age|name|
// +---+----+
// | 30|Andy|
// +---+----+

// Count people by age
df.groupBy("age").count().show()
// +----+-----+
// | age|count|
// +----+-----+
// |  19|    1|
// |null|    1|
// |  30|    1|
// +----+-----+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 中找到。
```

Java

```java
// col("...") is preferable to df.col("...")
import static org.apache.spark.sql.functions.col;

// Print the schema in a tree format
df.printSchema();
// root
// |-- age: long (nullable = true)
// |-- name: string (nullable = true)

// Select only the "name" column
df.select("name").show();
// +-------+
// |   name|
// +-------+
// |Michael|
// |   Andy|
// | Justin|
// +-------+

// Select everybody, but increment the age by 1
df.select(col("name"), col("age").plus(1)).show();
// +-------+---------+
// |   name|(age + 1)|
// +-------+---------+
// |Michael|     null|
// |   Andy|       31|
// | Justin|       20|
// +-------+---------+

// Select people older than 21
df.filter(col("age").gt(21)).show();
// +---+----+
// |age|name|
// +---+----+
// | 30|Andy|
// +---+----+

// Count people by age
df.groupBy("age").count().show();
// +----+-----+
// | age|count|
// +----+-----+
// |  19|    1|
// |null|    1|
// |  30|    1|
// +----+-----+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 中找到。
```

在 Python 中它既可以通过（df.age）属性又可以通过（df['age']）下标去访问一个 DataFrame 的列。前者是方便交互数据探索，用户使用后者的形式，它在以后并且不会破坏 DataFrame class 上属性的列名。

```python
# spark is an existing SparkSession

# Create the DataFrame
df = spark.read.json("examples/src/main/resources/people.json")

# Show the content of the DataFrame
df.show()
## age  name
## null Michael
## 30   Andy
## 19   Justin

# Print the schema in a tree format
df.printSchema()
## root
## |-- age: long (nullable = true)
## |-- name: string (nullable = true)

# Select only the "name" column
df.select("name").show()
## name
## Michael
## Andy
## Justin

# Select everybody, but increment the age by 1
df.select(df['name'], df['age'] + 1).show()
## name    (age + 1)
## Michael null
## Andy    31
## Justin  20

# Select people older than 21
df.filter(df['age'] > 21).show()
## age name
## 30  Andy

# Count people by age
df.groupBy("age").count().show()
## age  count
## null 1
## 19   1
## 30   1
```

**R**

```r
 # Create the DataFrame
df <- read.json("examples/src/main/resources/people.json")

# Show the content of the DataFrame
head(df)
## age  name
## null Michael
## 30   Andy
## 19   Justin

# Print the schema in a tree format
printSchema(df)
## root
## |-- age: long (nullable = true)
## |-- name: string (nullable = true)

# Select only the "name" column
head(select(df, "name"))
## name
## Michael
## Andy
## Justin

# Select everybody, but increment the age by 1
head(select(df, df$name, df$age + 1))
## name    (age + 1)
## Michael null
## Andy    31
## Justin  20

# Select people older than 21
head(where(df, df$age> 21))
## age name
## 30  Andy

# Count people by age
head(count(groupBy(df, "age")))
## age  count
## null 1
## 19   1
## 30   1

# 所有的示例代码可以在 Spark repo 的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 中找到。
```

能够在 DataFrame 上被执行的操作类型的完整列表请参考 [API 文档](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset")。

除了简单的列引用和表达式之外，DataFrame 也有丰富的函数库，包括 string 操作，date 算术，常见的 math 操作以及更多。可用的完整列表请参考 [DataFrame 函数参考](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$ "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$")。

### 以编程的方式运行 SQL 查询

**Scala**

```scala
// SparkSession 使应用程序的 SQL 函数能够以编程的方式运行 SQL 查询并且将查询结果以一个 DataFrame。

// Register the DataFrame as a SQL temporary view
df.createOrReplaceTempView("people")

val sqlDF = spark.sql("SELECT * FROM people")
sqlDF.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 中找到。
```

**Java**

```java
// SparkSession 使应用程序的 SQL 函数能够以编程的方式运行 SQL 查询并且将查询结果以一个 Dataset<Row>。

import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;

// Register the DataFrame as a SQL temporary view
df.createOrReplaceTempView("people");

Dataset<Row> sqlDF = spark.sql("SELECT * FROM people");
sqlDF.show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 中找到。
```

**Python**

```python
# SparkSession 使应用程序的 SQL 函数能够以编程的方式运行 SQL 查询并且将查询结果以一个 DataFrame。

# spark is an existing SparkSession
df = spark.sql("SELECT * FROM table")
```

**R**

```bash
df <- sql("SELECT * FROM table")

# 所有的示例代码可以在 Spark repo 的 "examples/src/main/r/RSparkSQLExample.R" 中找到。
```

### 创建 Dataset

Dataset 与 RDD 相似，然而，并不是使用 Java 序列化或者 Kryo，他们使用一个指定的 Encoder（编码器） 来序列化用于处理或者通过网络进行传输的对象。虽然编码器和标准的序列化都负责将一个对象序列化成字节，编码器是动态生成的代码，并且使用了一种允许 Spark 去执行许多像 filtering，sorting 以及 hashing 这样的操作，不需要将字节反序列化成对象的格式。

**Scala**

```scala
// Note: Case classes in Scala 2.10 can support only up to 22 fields. To work around this limit,
// you can use custom classes that implement the Product interface
case class Person(name: String, age: Long)

// Encoders are created for case classes
val caseClassDS = Seq(Person("Andy", 32)).toDS()
caseClassDS.show()
// +----+---+
// |name|age|
// +----+---+
// |Andy| 32|
// +----+---+

// Encoders for most common types are automatically provided by importing spark.implicits._
val primitiveDS = Seq(1, 2, 3).toDS()
primitiveDS.map(_ + 1).collect() // Returns: Array(2, 3, 4)

// DataFrames can be converted to a Dataset by providing a class. Mapping will be done by name
val path = "examples/src/main/resources/people.json"
val peopleDS = spark.read.json(path).as[Person]
peopleDS.show()
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 中找到。
```

**Java**

```java
import java.util.Arrays;
import java.util.Collections;
import java.io.Serializable;

import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;

public static class Person implements Serializable {
  private String name;
  private int age;

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public int getAge() {
    return age;
  }

  public void setAge(int age) {
    this.age = age;
  }
}

// Create an instance of a Bean class
Person person = new Person();
person.setName("Andy");
person.setAge(32);

// Encoders are created for Java beans
Encoder<Person> personEncoder = Encoders.bean(Person.class);
Dataset<Person> javaBeanDS = spark.createDataset(
  Collections.singletonList(person),
  personEncoder
);
javaBeanDS.show();
// +---+----+
// |age|name|
// +---+----+
// | 32|Andy|
// +---+----+

// Encoders for most common types are provided in class Encoders
Encoder<Integer> integerEncoder = Encoders.INT();
Dataset<Integer> primitiveDS = spark.createDataset(Arrays.asList(1, 2, 3), integerEncoder);
Dataset<Integer> transformedDS = primitiveDS.map(new MapFunction<Integer, Integer>() {
  @Override
  public Integer call(Integer value) throws Exception {
    return value + 1;
  }
}, integerEncoder);
transformedDS.collect(); // Returns [2, 3, 4]

// DataFrames can be converted to a Dataset by providing a class. Mapping based on name
String path = "examples/src/main/resources/people.json";
Dataset<Person> peopleDS = spark.read().json(path).as(personEncoder);
peopleDS.show();
// +----+-------+
// | age|   name|
// +----+-------+
// |null|Michael|
// |  30|   Andy|
// |  19| Justin|
// +----+-------+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 中找到。
```

### RDD 的互操作性

Spark SQL 支持两种不同的方法用于转换已存在的 RDD 成为 Dataset。

第一种方法是使用反射去推断一个包含指定的对象类型的 RDD 的 Schema。在你的 Spark 应用程序中当你已知 Schema 时这个基于方法的反射可以让你的代码更简洁。

第二种用于创建 Dataset 的方法是通过一个允许你构造一个 Schema 然后把它应用到一个已存在的 RDD 的编程接口。然而这种方法更繁琐，当列和它们的类型知道运行时都是未知时它允许你去构造 Dataset。

#### 使用反射推断 Schema

**Scala**

```scala
// Spark SQL 的 Scala 接口支持自动转换一个包含 case classes 的 RDD 为 DataFrame。 Case class 定义了表的 Schema。Case class 的参数名使用反射读取并且成为了列名。
// Case class 也可以是嵌套的或者包含像 SeqS 或者 ArrayS 这样的复杂类型。这个 RDD 能够被隐式转换成一个 DataFrame 然后被注册为一个表。表可以用于后续的 SQL 语句。

import org.apache.spark.sql.catalyst.encoders.ExpressionEncoder
import org.apache.spark.sql.Encoder

// For implicit conversions from RDDs to DataFrames
import spark.implicits._

// Create an RDD of Person objects from a text file, convert it to a Dataframe
val peopleDF = spark.sparkContext
  .textFile("examples/src/main/resources/people.txt")
  .map(_.split(","))
  .map(attributes => Person(attributes(0), attributes(1).trim.toInt))
  .toDF()
// Register the DataFrame as a temporary view
peopleDF.createOrReplaceTempView("people")

// SQL statements can be run by using the sql methods provided by Spark
val teenagersDF = spark.sql("SELECT name, age FROM people WHERE age BETWEEN 13 AND 19")

// The columns of a row in the result can be accessed by field index
teenagersDF.map(teenager => "Name:" + teenager(0)).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// or by field name
teenagersDF.map(teenager => "Name:" + teenager.getAs[String]("name")).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// No pre-defined encoders for Dataset[Map[K,V]], define explicitly
implicit val mapEncoder = org.apache.spark.sql.Encoders.kryo[Map[String, Any]]
// Primitive types and case classes can be also defined as
implicit val stringIntMapEncoder: Encoder[Map[String, Int]] = ExpressionEncoder()

// row.getValuesMap[T] retrieves multiple columns at once into a Map[String, T]
teenagersDF.map(teenager => teenager.getValuesMap[Any](List("name", "age"))).collect()
// Array(Map("name" -> "Justin", "age" -> 19))

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/scala/org/apache/spark/examples/sql/SparkSQLExample.scala" 中找到。
```

**Java**

```java
// Spark SQL 支持自动转换一个 JavaBean 的 RDD 为一个 DataFrame。这个 BeanInfo（Bean 的信息），可以使用反射获取，定义表的 Schema。目前，Spark SQL 不支持包含 Map 字段的 JavaBean。嵌套的 JavaBean 和 List 或者 Array 字段已经支持。
// 您可以通过创建一个实现了序列化的和拥有它的所有字段的 getter 以及 setter 方法的 class 来创建一个 JavaBean。

import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.Encoder;
import org.apache.spark.sql.Encoders;

// Create an RDD of Person objects from a text file
JavaRDD<Person> peopleRDD = spark.read()
  .textFile("examples/src/main/resources/people.txt")
  .javaRDD()
  .map(new Function<String, Person>() {
    @Override
    public Person call(String line) throws Exception {
      String[] parts = line.split(",");
      Person person = new Person();
      person.setName(parts[0]);
      person.setAge(Integer.parseInt(parts[1].trim()));
      return person;
    }
  });

// Apply a schema to an RDD of JavaBeans to get a DataFrame
Dataset<Row> peopleDF = spark.createDataFrame(peopleRDD, Person.class);
// Register the DataFrame as a temporary view
peopleDF.createOrReplaceTempView("people");

// SQL statements can be run by using the sql methods provided by spark
Dataset<Row> teenagersDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19");

// The columns of a row in the result can be accessed by field index
Encoder<String> stringEncoder = Encoders.STRING();
Dataset<String> teenagerNamesByIndexDF = teenagersDF.map(new MapFunction<Row, String>() {
  @Override
  public String call(Row row) throws Exception {
    return "Name:" + row.getString(0);
  }
}, stringEncoder);
teenagerNamesByIndexDF.show();
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// or by field name
Dataset<String> teenagerNamesByFieldDF = teenagersDF.map(new MapFunction<Row, String>() {
  @Override
  public String call(Row row) throws Exception {
    return "Name:" + row.<String>getAs("name");
  }
}, stringEncoder);
teenagerNamesByFieldDF.show();
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+

// 所有的示例代码可以在 Spark repo 的 "examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java" 中找到。
```

**Python**

```python
# Spark SQL 可以转换一个 Row 的 RDD 对象为一个 DataFrame，然后推断数据类型。Row 通过传递一系列 key/value（键 / 值）对作为 kwargs 到 Row class 从而被构造出来。
# 列出的 key 定义了表的列名，通过抽样整个数据库推断类型，与 JSON 文件上的执行的推断是相似的。

# spark is an existing SparkSession.
from pyspark.sql import Row
sc = spark.sparkContext

# Load a text file and convert each line to a Row.
lines = sc.textFile("examples/src/main/resources/people.txt")
parts = lines.map(lambda l: l.split(","))
people = parts.map(lambda p: Row(name=p[0], age=int(p[1])))

# Infer the schema, and register the DataFrame as a table.
schemaPeople = spark.createDataFrame(people)
schemaPeople.createOrReplaceTempView("people")

# SQL can be run over DataFrames that have been registered as a table.
teenagers = spark.sql("SELECT name FROM people WHERE age>= 13 AND age <= 19")

# The results of SQL queries are RDDs and support all the normal RDD operations.
teenNames = teenagers.map(lambda p: "Name:" + p.name)
for teenName in teenNames.collect():
  print(teenName)
```

#### 以编程的方式指定 Schema

当 case class 不能够在执行之前被定义（例如，records 记录的结构在一个 string 字符串中被编码了，或者一个 text 文本 datase 将被解析并且不同的用户投影的字段是不一样的）。一个 DataFrame 可以使用下面的三步以编程的方式来创建。

1. 从原始的 RDD 创建 RDD 的 RowS（行）。
2. Step 1 被创建后，创建 Schema 表示一个 StructType 匹配 RDD 中的 Rows（行）的结构。
3. 通过 SparkSession 提供的 createDataFrame 方法应用 Schema 到 RDD 的 RowS（行）。

例如 :

**Scala**

```scala
import org.apache.spark.sql.types._
import org.apache.spark.sql.Row


// Create an RDD
val peopleRDD = spark.sparkContext.textFile("examples/src/main/resources/people.txt")

// The schema is encoded in a string
val schemaString = "name age"

// Generate the schema based on the string of schema
val fields = schemaString.split(" ")
  .map(fieldName => StructField(fieldName, StringType, nullable = true))
val schema = StructType(fields)

// Convert records of the RDD (people) to Rows
val rowRDD = peopleRDD
  .map(_.split(","))
  .map(attributes => Row(attributes(0), attributes(1).trim))

// Apply the schema to the RDD
val peopleDF = spark.createDataFrame(rowRDD, schema)

// Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

// SQL can be run over a temporary view created using DataFrames
val results = spark.sql("SELECT name FROM people")

// The results of SQL queries are DataFrames and support all the normal RDD operations
// The columns of a row in the result can be accessed by field index or by field name
results.map(attributes => "Name:" + attributes(0)).show()
// +-------------+
// |        value|
// +-------------+
// |Name: Michael|
// |   Name: Andy|
// | Name: Justin|
// +-------------+
```

***

## 数据源

Spark SQL 支持通过 DataFrame 接口操作多种数据源。一个 DataFrame 可以通过关联转换来操作，也可以被创建为一个临时的 view。注册一个 DataFrame 作为一个临时的 view 就可以允许你在数据集上运行 SQL 查询。本节介绍了一些通用的方法通过使用 Spark Data Sources 来加载和保存数据以及一些可用的内置数据源的特定选项。

### 通用的 Load/Save 函数

在最简单的方式下，默认的数据源 (parquet 除非另外配置通过 spark.sql.sources.default) 将会用于所有的操作。

**Scala**

```scala
val usersDF = spark.read.load("examples/src/main/resources/users.parquet")
usersDF.select("name", "favorite_color").write.save("namesAndFavColors.parquet")
```

#### 手动指定选项

你也可以手动的指定数据源，并且将与你想要传递给数据源的任何额外选项一起使用。数据源由其完全限定名指定 (例如：org.apache.spark.sql.parquet)，不过对于内置数据源你也可以使用它们的缩写名 (json,parquet,jdbc)。使用下面这个语法可以将从任意类型数据源加载的 DataFrames 转换为其他类型。

**Scala**

```scala
val peopleDF = spark.read.format("json").load("examples/src/main/resources/people.json")
peopleDF.select("name", "age").write.format("parquet").save("namesAndAges.parquet")
```

#### 直接在文件上运行 SQL

你也可以直接在文件上运行 SQL 查询来替代使用 API 将文件加载到 DataFrame 再进行查询

**Scala**

```scala
val sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`")
```

#### 保存模式

Save 操作可以使用 SaveMode，可以指定如何处理已经存在的数据。这是很重要的要意识到这些保存模式没有利用任何锁并且也不是原子操作。另外，当执行 Overwrite，新数据写入之前会先将旧数据删除。

| Scala/Java | Any Language | Meaning |
| ------------- | ------------- | ------------- |
| SaveMode.ErrorIfExists(default) | "error"(default) | 当保存 DataFrame 到一个数据源，如果数据已经存在，将会抛出异常。 |
| SaveMode.Append | "append" | 当保存 DataFrame 到一个数据源，如果数据 / 表已经存在, DataFrame 的内容将会追加到已存在的数据后面。 |
| SaveMode.Overwrite | "overwrite" | Overwrite 模式意味着当保存 DataFrame 到一个数据源，如果数据 / 表已经存在，那么已经存在的数据将会被 DataFrame 的内容覆盖。 |
| SaveMode.Ignore "ignore" | Ignore | 模式意味着当保存 DataFrame 到一个数据源，如果数据已经存在，save 操作不会将 DataFrame 的内容保存，也不会修改已经存在的数据。这个和 SQL 中的'CREATE TABLE IF NOT EXISTS'类似 。 |

#### 保存为持久化的表

DataFrames 也可以通过 saveAsTable 命令来保存为一张持久表到 Hive metastore 中。值得注意的是对于这个功能来说已经存在的 Hive 部署不是必须的。Spark 将会为你创造一个默认的本地 Hive metastore（使用 Derby)。不像 createOrReplaceTempView 命令，saveAsTable 将会持久化 DataFrame 的内容并在 Hive metastore 中创建一个指向数据的指针。持久化的表将会一直存在甚至当你的 Spark 应用已经重启，只要保持你的连接是和一个相同的 metastore。一个相对于持久化表的 DataFrame 可以通过在 SparkSession 中调用 table 方法创建。

默认的话 saveAsTable 操作将会创建一个 "managed table"，意味着数据的位置将会被 metastore 控制。Managed tables 在表 drop 后也数据也会自动删除。

### Parquet 文件

Parquet 是一个列式存储格式的文件，被许多其他数据处理系统所支持。Spark SQL 支持对 Parquet 文件的读写还可以自动的保存源数据的模式

#### 以编程的方式加载数据

Scala

```scala
import spark.implicits._
val peopleDF = spark.read.json("examples/src/main/resources/people.json")
peopleDF.write.parquet("people.parquet")
val parquetFileDF = spark.read.parquet("people.parquet")
parquetFileDF.createOrReplaceTempView("parquetFile")
val namesDF = spark.sql("SELECT name FROM parquetFile WHERE age BETWEEN 13 AND 19")
namesDF.map(attributes => "Name:" + attributes(0)).show()
// +------------+
// | value|
// +------------+
// |Name: Justin|
// +------------+
```

#### 分区发现

在系统中，比如 Hive，表分区是一个很常见的优化途径。在一个分区表中 ，数据通常存储在不同的文件目录中，对每一个分区目录中的途径按照分区列的值进行编码。Parquet 数据源现在可以自动地发现并且推断出分区的信息。例如，我们可以将之前使用的人口数据存储成下列目录结构的分区表，两个额外的列，gender 和 country 作为分区列：

```
path
└── to
    └── table
        ├── gender=male
        │   ├── ...
        │   │
        │   ├── country=US
        │   │   └── data.parquet
        │   ├── country=CN
        │   │   └── data.parquet
        │   └── ...
        └── gender=female
            ├── ...
            │
            ├── country=US
            │   └── data.parquet
            ├── country=CN
            │   └── data.parquet
            └── ...
```

通过向 SparkSession.read.parquet 或 SparkSession.read.load 中传入 path/to/table,，Spark SQL 将会自动地从路径中提取分区信息。现在返回的 DataFrame schema 变成：

```
root

\|-- name: string (nullable = true)
\|-- age: long (nullable = true)
\|-- gender: string (nullable = true)
\|-- country: string (nullable = true)
```

需要注意的是分区列的数据类型是自动推导出的。当前，支持数值数据类型以及 string 类型。有些时候用户可能不希望自动推导出分区列的数据类型。对于这些使用场景，自动类型推导功能可以通过 spark.sql.sources.partitionColumnTypeInference.enabled 来配置，默认值是 true。当自动类型推导功能禁止，分区列的数据类型是 string。

从 Spark 1.6.0 开始，分区发现只能发现在默认给定的路径下的分区。对于上面那个例子，如果用户向 SparkSession.read.parquet 或 SparkSession.read.load, gender 传递 path/to/table/gender=male 将不会被当做分区列。如果用户需要指定发现的根目录，可以在数据源设置 basePath 选项。比如，将 path/to/table/gender=male 作为数据的路径并且设置 basePath 为 path/to/table/，gender 将会作为一个分区列。

#### Schema 合并

类似 ProtocolBuffer，Avro，以及 Thrift，Parquet 也支持 schema 演变。用户可以从一个简单的 schema 开始，并且根据需要逐渐地向 schema 中添加更多的列。这样，用户最终可能会有多个不同但是具有相互兼容 schema 的 Parquet 文件。Parquet 数据源现在可以自动地发现这种情况，并且将所有这些文件的 schema 进行合并。

由于 schema 合并是一个性格开销比较高的操作，并且在大部分场景下不是必须的，从 Spark 1.5.0 开始默认关闭了这项功能。你可以通过以下方式开启：

1. 设置数据源选项 mergeSchema 为 true 当读取 Parquet 文件时（如下面展示的例子），或者
2. 这是全局 SQL 选项 spark.sql.parquet.mergeSchema 为 true。

**Scala**

```scala
import spark.implicits._

// Create a simple DataFrame, store into a partition directory
val squaresDF = spark.sparkContext.makeRDD(1 to 5).map(i => (i, i * i)).toDF("value", "square")
squaresDF.write.parquet("data/test_table/key=1")

// Create another DataFrame in a new partition directory,
// adding a new column and dropping an existing column
val cubesDF = spark.sparkContext.makeRDD(6 to 10).map(i => (i, i * i * i)).toDF("value", "cube")
cubesDF.write.parquet("data/test_table/key=2")

// Read the partitioned table
val mergedDF = spark.read.option("mergeSchema", "true").parquet("data/test_table")
mergedDF.printSchema()

// The final schema consists of all 3 columns in the Parquet files together
// with the partitioning column appeared in the partition directory paths
// root
//  |-- value: int (nullable = true)
//  |-- square: int (nullable = true)
//  |-- cube: int (nullable = true)
//  |-- key: int (nullable = true)
```

#### Hive metastore Parquet 表转换

当从 Hive metastore 里读写 Parquet 表时，为了更好地提升新能 Spark SQL 会尝试用自己支持的 Parquet 替代 Hive SerDe。这个功能通过 spark.sql.hive.convertMetastoreParquet 选项来控制，默认是开启的。

##### Hive/Parquet Schema Reconciliation

从 Hive 和 Parquet 处理表 schema 过程的角度来看有两处关键的不同。

1. Hive 对大小写不敏感，而 Parquet 不是
2. Hive 认为所有列都是 nullable 可为空的，在再 Parquet 中为空性 nullability 是需要显示声明的。

由于这些原因，当我们将 Hive metastore Parquet table 转换为 Spark SQLtable 时必须使 Hive metastore schema 与 Parquet schema 相兼容。兼容规则如下：

1. 相同 schema 的字段的数据类型必须相同除了 nullability。要兼容的字段应该具有 Parquet 的数据类型，因此 nullability 是被推崇的。
2. reconciled  schema 包含了这些 Hive metastore schema 里定义的字段。
  - 任何字段只出现在 Parquet schema 中会被 reconciled schema 排除。
  - 任何字段只出现在 Hive metastore schema 中会被当做 nullable 字段来添加到 reconciled schema 中。

##### Metadata 刷新

为了提高性能 Spark SQL 缓存了 Parquet metadata。当 Hive metastore Parquet table 转换功能开启，这些转换后的元数据信息也会被缓存。如果这些表被 Hive 或者其他外部的工具更新，你需要手动刷新以确保元数据信息保持一致。

**Scala**

```scala
spark.catalog.refreshTable("my_table")
```

#### Parquet 配置

Parquet 的配置可以使用 SparkSession 的 setConf 来设置或者通过使用 SQL 运行 SET key=value 命令

| Property Name | Default | Meaning |
| ------------- | ------------- | ------------- |
| spark.sql.parquet.binaryAsString | false | 其他的一些产生 Parquet 的系统，特别是 Impala 和 SparkSQL 的老版本，当将 Parquet 模式写出时不会区分二进制数据和字符串。这个标志告诉 Spark SQL 将二进制数据解析成字符串，以提供对这些系统的兼容。 |
| spark.sql.parquet.int96AsTimestamp | true | 其他的一些产生 Parquet 的系统，特别是 Impala，将时间戳存储为 INT96 的形式。Spark 也将时间戳存储为 INT96，因为我们要避免纳秒级字段的精度的损失。这个标志告诉 Spark SQL 将 INT96 数据解析为一个时间戳，以提供对这些系统的兼容。 |
| spark.sql.parquet.cacheMetadata | true | 打开 Parquet 模式的元数据的缓存。能够加快对静态数据的查询。 |
| spark.sql.parquet.compression.codec | gzip | 设置压缩编码解码器，当写入一个 Parquet 文件时。可接收的值包括：uncompressed, snappy, gzip, lzo |
| spark.sql.parquet.filterPushdown | false | 打开 Parquet 过滤器的后进先出存储的优化。这个功能默认是被关闭的，因为一个 Parquet 中的一个已知的 bug 1.6.0rc3 (PARQUET-136)。然而，如果你的表中不包含任何的可为空的 (nullable) 字符串或者二进制列，那么打开这个功能是安全的。 |
| spark.sql.hive.convertMetastoreParquet | true | 当设置成 false，Spark SQL 会为 parquet 表使用 Hive SerDe(Serialize/Deserilize |
| spark.sql.parquet.mergeSchema  | false | 当设置 true，Parquet 数据源从所有的数据文件中合并 schemas，否则 schema 来自 summary file 或随机的数据文件当 summary file 不可得时. |

### JSON Datasets

Spark SQL 可以自动的推断出 JSON 数据集的 schema 并且将它作为 DataFrame 进行加载。这个转换可以通过使用 SparkSession.read.json() 在字符串类型的 RDD 中或者 JSON 文件。
注意作为 json file 提供的文件不是一个典型的 [JSON](http://jsonlines.org/ "http://jsonlines.org/") 文件。每一行必须包含一个分开的独立的有效 JSON 对象。因此，常规的多行 JSON 文件通常会失败。

**Scala**

```scala
 // A JSON dataset is pointed to by path.
// The path can be either a single text file or a directory storing text files
val path = "examples/src/main/resources/people.json"
val peopleDF = spark.read.json(path)

// The inferred schema can be visualized using the printSchema() method
peopleDF.printSchema()
// root
// |-- age: long (nullable = true)
// |-- name: string (nullable = true)

// Creates a temporary view using the DataFrame
peopleDF.createOrReplaceTempView("people")

// SQL statements can be run by using the sql methods provided by spark
val teenagerNamesDF = spark.sql("SELECT name FROM people WHERE age BETWEEN 13 AND 19")
teenagerNamesDF.show()
// +------+
// | name|
// +------+
// |Justin|
// +------+

// Alternatively, a DataFrame can be created for a JSON dataset represented by
// an RDD[String] storing one JSON object per string
val otherPeopleRDD = spark.sparkContext.makeRDD(
  """{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}""" :: Nil)
val otherPeople = spark.read.json(otherPeopleRDD)
otherPeople.show()
// +---------------+----+
// | address|name|
// +---------------+----+
// |[Columbus,Ohio]| Yin|
// +---------------+----+
```

### Hive 表

Spark SQL 还支持在 [Apache Hive](http://hive.apache.org/ "http://hive.apache.org/") 中读写数据。然而，由于 Hive 依赖项太多，这些依赖没有包含在默认的 Spark 发行版本中。如果在 classpath 上配置了 Hive 依赖，那么 Spark 会自动加载它们。注意，Hive 依赖也必须放到所有的 worker 节点上，因为如果要访问 Hive 中的数据它们需要访问 Hive 序列化和反序列化库（SerDes)。

Hive 配置是通过将 hive-site.xml，core-site.xml（用于安全配置）以及 hdfs-site.xml（用于 HDFS 配置）文件放置在 conf/ 目录下来完成的。

下面给出示例：

Scala 版：

如果要使用 Hive, 你必须要实例化一个支持 Hive 的 SparkSession，包括连接到一个持久化的 Hive metastore, 支持 Hive 序列化反序列化库以及 Hive 用户自定义函数。即使用户没有安装部署 Hive 也仍然可以启用 Hive 支持。如果没有在 hive-site.xml 文件中配置，Spark 应用程序启动之后，上下文会自动在当前目录下创建一个 metastore_db 目录并创建一个由 spark.sql.warehouse.dir 配置的、默认值是当前目录下的 spark-warehouse 目录的目录。请注意：从 Spark 2.0.0 版本开始, hive-site.xml 中的 hive.metastore.warehouse.dir 属性就已经过时了，你可以使用 spark.sql.warehouse.dir 来指定仓库中数据库的默认存储位置。你可能还需要给启动 Spark 应用程序的用户赋予写权限。

**Scala**

```scala
import org.apache.spark.sql.Row
import org.apache.spark.sql.SparkSession


case class Record(key: Int, value: String)


// warehouseLocation points to the default location for managed databases and tables
val warehouseLocation = "file:${system:user.dir}/spark-warehouse"


val spark = SparkSession
  .builder()
  .appName("Spark Hive Example")
  .config("spark.sql.warehouse.dir", warehouseLocation)
  .enableHiveSupport()
  .getOrCreate()


import spark.implicits._
import spark.sql


sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)")
sql("LOAD DATA LOCAL INPATH'examples/src/main/resources/kv1.txt'INTO TABLE src")


// Queries are expressed in HiveQL
sql("SELECT * FROM src").show()
// +---+-------+
// |key|  value|
// +---+-------+
// |238|val_238|
// | 86| val_86|
// |311|val_311|
// ...


// Aggregation queries are also supported.
sql("SELECT COUNT(*) FROM src").show()
// +--------+
// |count(1)|
// +--------+
// |    500 |
// +--------+


// The results of SQL queries are themselves DataFrames and support all normal functions.
val sqlDF = sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key")


// The items in DaraFrames are of type Row, which allows you to access each column by ordinal.
val stringsDS = sqlDF.map {
  case Row(key: Int, value: String) => s"Key: $key, Value: $value"
}
stringsDS.show()
// +--------------------+
// |               value|
// +--------------------+
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// ...


// You can also use DataFrames to create temporary views within a HiveContext.
val recordsDF = spark.createDataFrame((1 to 100).map(i => Record(i, s"val_$i")))
recordsDF.createOrReplaceTempView("records")


// Queries can then join DataFrame data with data stored in Hive.
sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show()
// +---+------+---+------+
// |key| value|key| value|
// +---+------+---+------+
// |  2| val_2|  2| val_2|
// |  2| val_2|  2| val_2|
// |  4| val_4|  4| val_4|
// ...
```

Java 版：

如果要使用 Hive, 你必须要实例化一个支持 Hive 的 SparkSession，包括连接到一个持久化的 Hive metastore, 支持 Hive 序列化反序列化库以及 Hive 用户自定义函数。即使用户没有安装部署 Hive 也仍然可以启用 Hive 支持。如果没有在 hive-site.xml 文件中配置，Spark 应用程序启动之后，上下文会自动在当前目录下创建一个 metastore_db 目录并创建一个由 spark.sql.warehouse.dir 配置的、默认值是当前目录下的 spark-warehouse 目录的目录。请注意：从 Spark 2.0.0 版本开始, hive-site.xml 中的 hive.metastore.warehouse.dir 属性就已经过时了，你可以使用 spark.sql.warehouse.dir 来指定仓库中数据库的默认存储位置。你可能还需要给启动 Spark 应用程序的用户赋予写权限。

**Java**

```java
import java.io.Serializable;
import java.util.ArrayList;
import java.util.List;


import org.apache.spark.api.java.function.MapFunction;
import org.apache.spark.sql.Dataset;
import org.apache.spark.sql.Encoders;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SparkSession;


public static class Record implements Serializable {
  private int key;
  private String value;


  public int getKey() {
    return key;
  }


  public void setKey(int key) {
    this.key = key;
  }


  public String getValue() {
    return value;
  }


  public void setValue(String value) {
    this.value = value;
  }
}


// warehouseLocation points to the default location for managed databases and tables
String warehouseLocation = "file:" + System.getProperty("user.dir") + "spark-warehouse";
SparkSession spark = SparkSession
  .builder()
  .appName("Java Spark Hive Example")
  .config("spark.sql.warehouse.dir", warehouseLocation)
  .enableHiveSupport()
  .getOrCreate();


spark.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)");
spark.sql("LOAD DATA LOCAL INPATH'examples/src/main/resources/kv1.txt'INTO TABLE src");


// Queries are expressed in HiveQL
spark.sql("SELECT * FROM src").show();
// +---+-------+
// |key|  value|
// +---+-------+
// |238|val_238|
// | 86| val_86|
// |311|val_311|
// ...


// Aggregation queries are also supported.
spark.sql("SELECT COUNT(*) FROM src").show();
// +--------+
// |count(1)|
// +--------+
// |    500 |
// +--------+


// The results of SQL queries are themselves DataFrames and support all normal functions.
Dataset<Row> sqlDF = spark.sql("SELECT key, value FROM src WHERE key < 10 ORDER BY key");


// The items in DaraFrames are of type Row, which lets you to access each column by ordinal.
Dataset<String> stringsDS = sqlDF.map(new MapFunction<Row, String>() {
  @Override
  public String call(Row row) throws Exception {
    return "Key:" + row.get(0) + ", Value:" + row.get(1);
  }
}, Encoders.STRING());
stringsDS.show();
// +--------------------+
// |               value|
// +--------------------+
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// |Key: 0, Value: val_0|
// ...


// You can also use DataFrames to create temporary views within a HiveContext.
List<Record> records = new ArrayList<>();
for (int key = 1; key < 100; key++) {
  Record record = new Record();
  record.setKey(key);
  record.setValue("val_" + key);
  records.add(record);
}
Dataset<Row> recordsDF = spark.createDataFrame(records, Record.class);
recordsDF.createOrReplaceTempView("records");


// Queries can then join DataFrames data with data stored in Hive.
spark.sql("SELECT * FROM records r JOIN src s ON r.key = s.key").show();
// +---+------+---+------+
// |key| value|key| value|
// +---+------+---+------+
// |  2| val_2|  2| val_2|
// |  2| val_2|  2| val_2|
// |  4| val_4|  4| val_4|
// ...
```

Python 版：

如果要使用 Hive, 你必须要实例化一个支持 Hive 的 SparkSession，包括连接到一个持久化的 Hive metastore, 支持 Hive 序列化反序列化库以及 Hive 用户自定义函数。即使用户没有安装部署 Hive 也仍然可以启用 Hive 支持。如果没有在 hive-site.xml 文件中配置，Spark 应用程序启动之后，上下文会自动在当前目录下创建一个 metastore_db 目录并创建一个由 spark.sql.warehouse.dir 配置的、默认值是当前目录下的 spark-warehouse 目录的目录。请注意：从 Spark 2.0.0 版本开始, hive-site.xml 中的 hive.metastore.warehouse.dir 属性就已经过时了，你可以使用 spark.sql.warehouse.dir 来指定仓库中数据库的默认存储位置。你可能还需要给启动 Spark 应用程序的用户赋予写权限。

**Python**

```python
# spark is an existing SparkSession


spark.sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)")
spark.sql("LOAD DATA LOCAL INPATH'examples/src/main/resources/kv1.txt'INTO TABLE src")


# Queries can be expressed in HiveQL.
results = spark.sql("FROM src SELECT key, value").collect()
```

R 版：

如果要使用 Hive, 你必须要实例化一个支持 Hive 的 SparkSession。这添加了在 MetaStore 中查找表和使用 HiveQL 写查询的支持。

**R**

```r
# enableHiveSupport defaults to TRUE
sparkR.session(enableHiveSupport = TRUE)
sql("CREATE TABLE IF NOT EXISTS src (key INT, value STRING)")
sql("LOAD DATA LOCAL INPATH'examples/src/main/resources/kv1.txt'INTO TABLE src")


# Queries can be expressed in HiveQL.
results <- collect(sql("FROM src SELECT key, value"))
```

完整示例代码参见 Spark 仓库中的 "examples/src/main/r/RSparkSQLExample.R"。

#### 与不同版本的 Hive Metastore 交互

Spark SQL 对 Hive 最重要的一个支持就是可以和 Hive metastore 进行交互，这使得 Spark SQL 可以访问 Hive 表的元数据。从 Spark 1.4.0 版本开始，通过使用下面描述的配置, Spark SQL 一个简单的二进制编译版本可以用来查询不同版本的 Hive metastore。注意，不管用于访问 metastore 的 Hive 是什么版本，Spark SQL 内部都使用 Hive 1.2.1 版本进行编译, 并且使用这个版本的一些类用于内部执行（serdes，UDFs，UDAFs 等）。

下面的选项可用来配置用于检索元数据的 Hive 版本：

| 属性名 | 默认值 | 含义 |
| ------------- | ------------- | ------------- |
| spark.sql.hive.metastore.version | 1.2.1 | Hive metastore 版本。可用选项从 0.12.0 到 1.2.1 。 |
| spark.sql.hive.metastore.jars | builtin | 存放用于实例化 HiveMetastoreClient 的 jar 包位置。这个属性可以是下面三个选项之一：1\. builtin：使用 Hive 1.2.1 版本，当启用 -Phive 时会和 Spark 一起打包。如果使用了这个选项, 那么 spark.sql.hive.metastore.version 要么是 1.2.1，要么就不定义。2\. maven：使用从 Maven 仓库下载的指定版本的 Hive jar 包。生产环境部署通常不建议使用这个选项。3\. 标准格式的 JVM classpath。这个 classpath 必须包含所有 Hive 及其依赖的 jar 包，并且包含正确版本的 hadoop。这些 jar 包只需要部署在 driver 节点上，但是如果你使用 yarn cluster 模式运行，那么你必须要确保这些 jar 包是和应用程序一起打包的。 |
| spark.sql.hive.metastore.sharedPrefixes | com.mysql.jdbc, org.postgresql, com.microsoft.sqlserver, oracle.jdbc |  一个逗号分隔的类名前缀列表，这些类使用 classloader 加载，且可以在 Spark SQL 和特定版本的 Hive 间共享。一个共享类的示例就是用来访问 Hive metastore 的 JDBC driver。其它需要共享的类，是需要与已经共享的类进行交互的。例如，log4j 使用的自定义 appender 。 |
| spark.sql.hive.metastore.barrierPrefixes | (empty) | 一个逗号分隔的类名前缀列表，这些类需要在 Spark SQL 访问的每个 Hive 版本中显式地重新加载。例如，在一个共享前缀列表（org.apache.spark.*）中声明的 Hive UDF 通常需要被共享。|

### JDBC 连接其它数据库

Spark SQL 还有一个能够使用 JDBC 从其他数据库读取数据的数据源。当使用 JDBC 访问其它数据库时，应该首选 [JdbcRDD](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.JdbcRDD "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.rdd.JdbcRDD")。这是因为结果是以数据框（DataFrame）返回的，且这样 Spark SQL 操作轻松或便于连接其它数据源。因为这种 JDBC 数据源不需要用户提供 ClassTag，所以它也更适合使用 Java 或 Python 操作。（注意，这与允许其它应用使用 Spark SQL 执行查询操作的 Spark SQL JDBC 服务器是不同的）。

使用 JDBC 访问特定数据库时，需要在 spark classpath 上添加对应的 JDBC 驱动配置。例如，为了从 Spark Shell 连接 postgres，你需要运行如下命令：

通过调用数据源 API，远程数据库的表可以被加载为 DataFrame 或 Spark SQL 临时表。支持的参数有：

```bash
bin/spark-shell --driver-class-path postgresql-9.4.1207.jar --jars postgresql-9.4.1207.jar
```

| 属性名 | 含义 |
| ------------- | ------------- |
| url | 要连接的 JDBC URL。 |
| dbtable | 要读取的 JDBC 表。 注意，一个 SQL 查询的 From 分语句中的任何有效表都能被使用。例如，既可以是完整表名，也可以是括号括起来的子查询语句。|
| driver | 用于连接 URL 的 JDBC 驱动的类名。 |
| partitionColumn, lowerBound, upperBound, numPartitions | 这 几个选项，若有一个被配置，则必须全部配置。它们描述了当从多个 worker 中并行的读取表时，如何对它分区。partitionColumn 必须时所查询表的一个数值字段。注意，lowerBound 和 upperBound 都只是用于决定分区跨度的，而不是过滤表中的行。因此，表中的所有行将被分区并返回。 |
| fetchSize | JDBC fetch size, 决定每次读取多少行数据。 默认将它设为较小值（如，Oracle 上设为 10）有助于 JDBC 驱动上的性能优化。 |

代码示例如下 :

**Scala**

```scala
val jdbcDF = spark.read
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .load()
```

**Java**

```java
Dataset<Row> jdbcDF = spark.read()
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .load();
```

**Python**

```python
jdbcDF = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:postgresql:dbserver") \
    .option("dbtable", "schema.tablename") \
    .option("user", "username") \
    .option("password", "password") \
    .load()
```
**R**

```r
df <- read.jdbc("jdbc:postgresql:dbserver", "schema.tablename", user = "username", password = "password")
```

**SQL**

```sql
CREATE TEMPORARY VIEW jdbcTable
USING org.apache.spark.sql.jdbc
OPTIONS (
  url "jdbc:postgresql:dbserver",
  dbtable "schema.tablename"
)
```

### 故障排除

- 在客户端会话（client session) 中或者所有 executor 上，JDBC 驱动类必须可见于原生的类加载器。这是因为 Java 的驱动管理（DriverManager）类在打开一个连接之前会做一个安全检查，这就导致它忽略了所有对原生类加载器不可见的驱动。一个方便的方法，就是修改所有 worker 节点上的 compute_classpath.sh 以包含你的驱动 Jar 包。
- 一些数据库，如 H2，会把所有的名称转为大写。在 Spark SQL 中你也需要使用大写来引用这些名称。

***

## 性能调优

对一些工作负载，可能的性能改进的方式，不是把数据缓存在内存里，就是调整一些试验选项。

### 缓存数据到内存

Spark SQL 可以通过调用 spark.cacheTable("tableName") 或者 dataFrame.cache() 以列存储格式缓存表到内存中。随后，Spark SQL 将会扫描必要的列，并自动调整压缩比例，以减少内存占用和 GC 压力。你可以调用 spark.uncacheTable("tableName") 来删除内存中的表。

你可以在 SparkSession 上使用 setConf 方法或在 SQL 语句中运行 `SET key=value` 命令，来配置内存中的缓存。

| 属性名 | 默认值 | 含义 |
| ------------- | ------------- | ------------- |
| spark.sql.inMemoryColumnarStorage.compressed | true | 当设置为 true 时，Spark SQL 将会基于数据的统计信息自动地为每一列选择单独的压缩编码方式。 |
| spark.sql.inMemoryColumnarStorage.batchSize | 10000 | 控制列式缓存批量的大小。当缓存数据时，增大批量大小可以提高内存利用率和压缩率，但同时也会带来 OOM（Out Of Memory）的风险。|

### 其它配置选项

下面的选项也可以用来提升查询执行的性能。随着 Spark 自动地执行越来越多的优化操作，这些选项在未来的发布版本中可能会过时。

| 属性名 | 默认值 | 含义 |
| ------------- | ------------- | ------------- |
| spark.sql.files.maxPartitionBytes | 134217728 (128 MB) | 读取文件时单个分区可容纳的最大字节数。 |
| spark.sql.files.openCostInBytes | 4194304 (4 MB) | 打开文件的估算成本，按照同一时间能够扫描的字节数来测量。当往一个分区写入多个文件的时候会使用。高估更好, 这样的话小文件分区将比大文件分区更快 (先被调度)。 |
| spark.sql.autoBroadcastJoinThreshold | 10485760 (10 MB) | 配置一个表在执行 join 操作时能够广播给所有 worker 节点的最大字节大小。通过将这个值设置为 -1，可以禁用广播。注意，目前的数据统计仅支持已经运行了 ANALYZE TABLE <tableName> COMPUTE STATISTICS noscan 命令的 Hive metastore 表。 |
| spark.sql.shuffle.partitions | 200 | 配置为连接或聚合操作混洗（shuffle）数据时使用的分区数。 |

***

## 分布式 SQL 引擎

通过使用 Spark SQL 的 JDBC/ODBC 或者命令行接口，它还可以作为一个分布式查询引擎。在这种模式下，终端用户或应用程序可以运行 SQL 查询来直接与 Spark SQL 交互，而不需要编写任何代码。

### 运行 Thrift JDBC/ODBC server

这里实现的 Thrift JDBC/ODBC server 对应于 Hive 1.2.1 版本中的 [HiveServer2](https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2 "https://cwiki.apache.org/confluence/display/Hive/Setting+Up+HiveServer2")。你可以使用 Spark 或者 Hive 1.2.1 自带的 beeline 脚本来测试这个 JDBC server。

要启动 JDBC/ODBC server， 需要在 Spark 安装目录下运行如下命令：

```bash
./sbin/start-thriftserver.sh
```

这个脚本能接受所有 bin/spark-submit 命令行选项，外加一个用于指定 Hive 属性的 --hiveconf 选项。你可以运行 ./sbin/start-thriftserver.sh --help 来查看所有可用选项的完整列表。默认情况下，这启动的 server 将会在 localhost:10000 上进行监听。你可以覆盖该行为，比如使用以下环境变量：

```bash
export HIVE_SERVER2_THRIFT_PORT=<listening-port>
export HIVE_SERVER2_THRIFT_BIND_HOST=<listening-host>
./sbin/start-thriftserver.sh \
  --master <master-uri> \
  ...
```

或者系统属性：

```bash
./sbin/start-thriftserver.sh \
  --hiveconf hive.server2.thrift.port=<listening-port> \
  --hiveconf hive.server2.thrift.bind.host=<listening-host> \
  --master <master-uri>
  ...
```

现在你可以使用 beeline 来测试这个 Thrift JDBC/ODBC server：

```bash
./bin/beeline
```

在 beeline 中使用以下命令连接到 JDBC/ODBC server :

```bash
beeline> !connect jdbc:hive2://localhost:10000
```

Beeline 会要求你输入用户名和密码。在非安全模式下，只需要输入你本机的用户名和一个空密码即可。对于安全模式，请参考 [beeline 文档](https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients "https://cwiki.apache.org/confluence/display/Hive/HiveServer2+Clients") 中的指示。

将 hive-site.xml，core-site.xml 以及 hdfs-site.xml 文件放置在 conf 目录下可以完成 Hive 配置。

你也可以使用 Hive 自带的 beeline 的脚本。

Thrift JDBC server 还支持通过 HTTP 传输来发送 Thrift RPC 消息。使用下面的设置作为系统属性或者对 conf 目录中的 hive-site.xml 文件配置来启用 HTTP 模式：

```bash
hive.server2.transport.mode - Set this to value: http
hive.server2.thrift.http.port - HTTP port number fo listen on; default is 10001
hive.server2.http.endpoint - HTTP endpoint; default is cliservice
```

为了测试，在 HTTP 模式中使用 beeline 连接到 JDBC/ODBC server：

```bash
beeline> !connect jdbc:hive2://<host>:<port>/<database>?hive.server2.transport.mode=http;hive.server2.thrift.http.path=<http_endpoint>
```

### 运行 Spark SQL CLI

Spark SQL CLI 是一个很方便的工具，它可以在本地模式下运行 Hive metastore 服务，并且执行从命令行中输入的查询语句。注意：Spark SQL CLI 无法与 Thrift JDBC server 通信。

要启动 Spark SQL CLI, 可以在 Spark 安装目录运行下面的命令:

```bash
./bin/spark-sql
```

将 hive-site.xml，core-site.xml 以及 hdfs-site.xml 文件放置在 conf 目录下可以完成 Hive 配置。你可以运行 ./bin/spark-sql --help 来获取所有可用选项的完整列表。

***

## 迁移指南

### 从 Spark SQL 1.6 升级到 2.0

- SparkSession 现在是 Spark 新的切入点, 它替代了老的 SQLContext 和 HiveContext。注意：为了向下兼容, 老的 SQLContext 和 HiveContext 仍然保留。可以从 SparkSession 获取一个新的 catalog 接口——现有的访问数据库和表的 API, 如 listTables, createExternalTable, dropTempView, cacheTable 都被移到该接口。
- Dataset API 和 DataFrame API 进行了统一。在 Scala 中，DataFrame 变成了 Dataset[Row] 的一个类型别名, 而 Java API 使用者必须将 DataFrame 替换成 Dataset<Row>。Dataset 类既提供了强类型转换操作 (如 map, filter 以及 groupByKey) 也提供了非强类型转换操作 (如 select 和 groupBy) 。由于编译期的类型安全不是 Python 和 R 语言的一个特性,  Dataset 的概念并不适用于这些语言的 API。相反，DataFrame 仍然是最基本的编程抽象, 就类似于这些语言中单节点数据帧的概念。
- Dataset 和 DataFrame API 中 unionAll 已经过时并且由 union 替代。
- Dataset 和 DataFrame API 中 explode 已经过时，作为选择，可以结合 select 或 flatMap 使用 functions.explode() 。
- Dataset 和 DataFrame API 中 registerTempTable 已经过时并且由 createOrReplaceTempView 替代。

### 从 Spark SQL 1.5 升级到 1.6

- Spark 1.6 中，默认情况下服务器在多会话模式下运行。这意味着每个 JDBC / ODBC 连接拥有一份自己的 SQL 配置和临时注册表。缓存表仍在并共享。如果你想在单会话模式服务器运行，请设置选项 spark.sql.hive.thriftServer.singleSession 为 true。您既可以将此选项添加到 spark-defaults.conf，或者通过 --conf 将它传递给 start-thriftserver.sh。

```bash
./sbin/start-thriftserver.sh \ --conf spark.sql.hive.thriftServer.singleSession=true \ ...
```

- 从 1.6.1 开始，在 sparkR 中 withColumn 方法支持添加一个新列或更换数据框同名的现有列。
- 从 Spark 1.6 开始，LongType 强制转换为 TimestampType 秒，而不是微秒。这一变化是为了匹配 Hive 1.2 ，保证从数值类型转换到 TimestampType 的一致性。见 [SPARK-11724](https://issues.apache.org/jira/browse/SPARK-11724 "https://issues.apache.org/jira/browse/SPARK-11724") 了解详情。

### 从 Spark SQL 1.4 升级到 1.5

- 使用手动管理的内存优化执行，现在是默认启用的，以及代码生成表达式求值。这些功能既可以通过设置 spark.sql.tungsten.enabled 到 false 来禁止使用。
- Parquet 的模式合并默认情况下不再启用。它可以通过设置重新启用 spark.sql.parquet.mergeSchema 到 true 。
- 字符串在 Python 列的分辨率现在支持使用点（.）来限定列或访问嵌套值。例如 df['table.column.nestedField']。但是，这意味着如果你的列名中包含任何圆点，你现在必须避免使用反引号（如 table.`column.with.dots`.nested）。
- 在内存中的列存储分区修剪默认是开启的。它可以通过设置 spark.sql.inMemoryColumnarStorage.partitionPruning 到 false 来禁用。
- 无限精度的小数列不再支持，而不是 Spark SQL 最大精度为 38 。当从 BigDecimal 对象推断模式时，现在使用（38，18）。当 DDL 没有指定精度，则默认保留 Decimal(10, 0)。
- 时间戳现在存储在 1 微秒的精度，而不是 1 纳秒的。
- 在 sql 语句中，浮点数现在解析为十进制。HiveQL 解析保持不变。
- SQL/DateFrame 数据帧功能的规范名称现在是小写（e.g. sum vs SUM）。
- JSON 数据源不会自动加载由其他应用程序（未通过 Spark SQL 插入到数据集的文件）创建的新文件。对于 JSON 持久表（即表的元数据存储在 Hive Metastore），用户可以使用 REFRESH TABLE SQL 命令或 HiveContext 的 refreshTable 方法，把那些新文件列入到表中。对于代表一个 JSON 数据集的数据帧，用户需要重新创建数据框，同时数据框中将包括新的文件。
- PySpark DataFrame 的 withColumn 方法支持添加新的列或替换现有的同名列。

### 从 Spark SQL 1.3 升级到 1.4

#### 数据帧的数据读 / 写器接口

根据用户的反馈，我们创建了一个新的更快速的 API 中读取数据 ( SQLContext.read）和写入数据（DataFrame.write）。同时废弃的过时的 API（例如 SQLContext.parquetFile，SQLContext.jsonFile）。

请参阅 API 文档 SQLContext.read（[Scala](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.SQLContext@read:DataFrameReader "http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.SQLContext@read:DataFrameReader") ，[Java](http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/SQLContext.html#read() "http://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/SQLContext.html#read()")， [Python](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.SQLContext.read "http://spark.apache.org/docs/latest/api/python/pyspark.sql.html#pyspark.sql.SQLContext.read") ）和 DataFrame.write（ Scala ，Java， Python ）的更多信息。

#### DataFrame.groupBy 保留分组列

根据用户反馈，我们改变的默认行为 DataFrame.groupBy().agg() 保留在 DataFrame 的分组列。为了维持 1.3 的行为特征，设置 spark.sql.retainGroupColumns 为 false。

Scala 示例：

```scala
// 在 1.3.x 中, 为了让 "department" 列得到展示，
// 必须包括明确作为 gg 函数调用的一部分。
df.groupBy("department").agg($"department", max("age"), sum("expense"))

// 在 1.4 以上版本, "department" 列自动包含了.
df.groupBy("department").agg(max("age"), sum("expense"))

// 恢复到 1.3 版本（不保留分组列）
sqlContext.setConf("spark.sql.retainGroupColumns", "false")
```

#### 在 DataFrame.withColumn 中的改变

之前 1.4 版本中，DataFrame.withColumn（）只支持添加列。该列将始终在 DateFrame 结果中被加入作为新的列，即使现有的列可能存在相同的名称。从 1.4 版本开始，DataFrame.withColumn（）支持添加与所有现有列的名称不同的列或替换现有的同名列。

请注意，这一变化仅适用于 Scala API , 并不适用于 PySpark 和 SparkR。

### 从 Spark SQL 1.0~1.2 升级到 1.3

在 Spark 1.3 中，我们从 Spark SQL 中删除了 "Alpha" 的标签，作为一部分已经清理过的可用的 API 。从 Spark 1.3 版本以上，Spark SQL 将提供在 1.X 系列的其他版本的二进制兼容性。这种兼容性保证不包括被明确标记为不稳定的（即 DeveloperApi 类或 Experimental）的 API。

#### 重命名 SchemaRDD 到 DateFrame

升级到 Spark SQL 1.3 版本时，用户会发现最大的变化是，SchemaRDD 已更名为 DataFrame。这主要是因为 DataFrames 不再从 RDD 直接继承，而是由 RDDS 自己来实现这些功能。DataFrames 仍然可以通过调用 .rdd 方法转换为 RDDS 。

在 Scala 中，有一个从 SchemaRDD 到 DataFrame 类型别名，可以为一些情况提供源代码兼容性。它仍然建议用户更新他们的代码以使用 DataFrame 来代替。Java 和 Python 用户需要更新他们的代码。

#### 在 Java 和 Scala API 的统一

此前 Spark 1.3 有单独的 Java 兼容类（JavaSQLContext 和 JavaSchemaRDD），借鉴于 Scala API。在 Spark 1.3 中，Java API 和 Scala API 已经统一。两种语言的用户可以使用 SQLContext 和 DataFrame 。一般来说论文类尝试使用两种语言的共有类型（如 Array 替代了一些特定集合）。在某些情况下不通用的类型情况下，（例如，passing in closures 或 Maps）使用函数重载代替。

此外，该 Java 的特定类型的 API 已被删除。Scala 和 Java 的用户可以使用存在于 org.apache.spark.sql.types 类来描述编程模式。

#### 隐式转换和 DSL 包的移除（仅限于 Scala）

许多 Spark 1.3 版本以前的代码示例都以 import sqlContext._ 开始，这提供了从 sqlContext 到 cope 的所有功能。在 Spark 1.3 中，我们移除了从 RDDs 到 DateFrame 再到 SQLContext 内部对象的隐式转换。

此外，隐式转换现在只是通过 toDF 方法增加 RDDs 所组成的一些类型（例如 classes 或 tuples），而不是自动应用。

当使用 DSL 的内部函数（现在由 DataFrame API 代替）的时候，用于一般会导入 org.apache.spark.sql.catalyst.dsl 来代替一些公有的 DataFrame 的 API 函数 ：import org.apache.spark.sql.functions._。

#### 删除在 org.apache.spark.sql 包中的一些 DataType 别名（仅限于 Scala）

Spark 1.3 移除存在于基本 SQL 包的 DataType 类型别名。开发人员应改为导入类 org.apache.spark.sql.types。

#### UDF 注册迁移到 sqlContext.udf 中 (针对 Java 和 Scala)

用于注册 UDF 的函数，不管是 DataFrame DSL 还是 SQL 中用到的，都被迁移到 SQLContext 中的 udf 对象中。

**Scala**

```scala
sqlContext.udf.register("strLen", (s: String) => s.length())
```

Python UDF 注册保持不变。

#### Python 的 DataType 不再是单例的

在 Python 中使用 DataTypes 时，你需要先构造它们（如：StringType（）），而不是引用一个单例对象。

### 兼容 Apache Hive

Spark SQL 在设计时就考虑到了和 Hive metastore，SerDes 以及 UDF 之间的兼容性。目前 Hive SerDes 和 UDF 都是基于 Hive 1.2.1 版本，并且 Spark SQL 可以连接到不同版本的 Hive metastore（从 0.12.0 到 1.2.1，可以参考［与不同版本的 Hive Metastore 交互］）

#### 在现有的 Hive 仓库中部署

Spark SQL Thrift JDBC server 采用了开箱即用的设计以兼容已有的 Hive 安装版本。你不需要修改现有的 Hive Metastore , 或者改变数据的位置和表的分区。

#### 支持 Hive 的特性

Spark SQL 支持绝大部分的 Hive 功能，如：

- Hive 查询语句, 包括：
  - SELECT
  - GROUP BY
  - ORDER BY
  - CLUSTER BY
  - SORT BY
- 所有的 Hive 运算符， 包括：
  - 关系运算符 (=, ⇔, ==, <>, <, >, >=, <=, etc)
  - 算术运算符 (+, -, *, /, %, etc)
  - 逻辑运算符 (AND, &&, OR, \|\|, etc)
  - 复杂类型构造器 - 数学函数 (sign, ln, cos 等)
  - String 函数 (instr, length, printf 等)
- 用户自定义函数（UDF）
- 用户自定义聚合函数（UDAF）
- 用户自定义序列化格式（SerDes）
- 窗口函数
- Joins
  - JOIN
  - {LEFT\|RIGHT\|FULL} OUTER JOIN
  - LEFT SEMI JOIN - CROSS JOIN
- Unions
- 子查询
  - SELECT col FROM (SELECT a + b AS col from t1) t2
- 采样
- Explain
- 分区表，包括动态分区插入
- 视图
- 所有 Hive DDL 功能, 包括：
  - CREATE TABLE
  - CREATE TABLE AS SELECT
  - ALTER TABLE
- 绝大多数 Hive 数据类型，包括
  - TINYINT
  - SMALLINT
  - INT
  - BIGINT
  - BOOLEAN
  - FLOAT
  - DOUBLE
  - STRING
  - BINARY
  - TIMESTAMP
  - DATE
  - ARRAY<>
  - MAP<>
  - STRUCT<>

#### 不支持的 Hive 功能

以下是目前还不支持的 Hive 功能列表。在 Hive 部署中这些功能大部分都用不到。

#### Hive 核心功能

bucket：bucket 是 Hive 表分区内的一个哈希分区，Spark SQL 目前还不支持 bucket。

#### Hive 高级功能

- UNION 类型
- Unique join
- 列统计数据收集：Spark SQL 目前不依赖扫描来收集列统计数据并且仅支持填充 Hive metastore 的 sizeInBytes 字段。

#### Hive 输入输出格式

- CLI 文件格式：对于回显到 CLI 中的结果，Spark SQL 仅支持 TextOutputFormat。
- Hadoop archive

#### Hive 优化

有少数 Hive 优化还没有包含在 Spark 中。其中一些（比如索引）由于 Spark SQL 的这种内存计算模型而显得不那么重要。另外一些在 Spark SQL 未来的版本中会持续跟踪。

- 块级别位图索引和虚拟列（用来建索引）
- 自动为 join 和 groupBy 计算 reducer 个数：目前在 Spark SQL 中，你需要使用 "SET spark.sql.shuffle.partitions=[num_tasks];"
- 来控制后置混洗的并行程度。
- 仅查询元数据：对于只需要使用元数据的查询请求，Spark SQL 仍需要启动任务来计算结果。
- 数据倾斜标志：Spark SQL 不遵循 Hive 中的数据倾斜标志
- STREAMTABLE join 操作提示：Spark SQL 不遵循 STREAMTABLE 提示。
- 对于查询结果合并多个小文件：如果返回的结果有很多小文件，Hive 有个选项设置，来合并小文件，以避免超过HDFS的文件数额度限制。Spark SQL 不支持这个。
