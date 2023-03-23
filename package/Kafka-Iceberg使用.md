# Kafka-Iceberg使用

## 1. 前言

Iceberg Version : 0.12.1

Spark Version : 3.1.2

Scala Version : 2.12.10

将Kafka的数据,使用Spark DataSourceV2 api写入到Iceberg数据湖中

## 2. kafka原始数据格式

| 名称         | 类型      |
| ------------ | --------- |
| user_id      | Long      |
| station_time | String    |
| score        | Int       |
| local_time   | Timestamp |

## 3. 步骤详解

### 3.1 构建spark

这里选择了HadoopCatalog，将数据都放在hdfs上

catolog名称为czl_iceberg

```scala
val sparkSession: SparkSession = SparkSession.builder().master("local[*]")
    //define catalog name
    .config("spark.sql.catalog.czl_iceberg", "org.apache.iceberg.spark.SparkCatalog")
    .config("spark.sql.catalog.czl_iceberg.type", "hadoop")
    .config("spark.sql.catalog.czl_iceberg.warehouse", wareHousePath)
    .getOrCreate()
```

### 3.2 读取Kafka

使用`spark-sql-kafka-0-10_2.12`加载kafka数据

`kafka.bootstrap.servers`和`subscribe`是必须的，指定kafka的`ip`以及订阅的`topic`，其余参数可选

```scala
import sparkSession.implicits._
val df: DataFrame = sparkSession
  .readStream
  .format("kafka")
  .option("kafka.bootstrap.servers", kafkaConsumer)
  .option("subscribe", kafkaTopic)
  .option("startingOffsets", startingOffsets)
  .option("failOnDataLoss", failOnDataLoss)
  .option("maxOffsetsPerTrigger", maxOffsetsPerTrigger)
  .load()
 val frame: Dataset[Row] = df
  .select(from_json('value.cast("string"), schema) as "value")
  .select($"value.*")
```

### 3.3 Create iceberg table

在使用`Spark Structured Streaming`写入iceberg时，必须先创建iceberg表，参照官方文档

The table should be created in prior to start the streaming query. Refer [SQL create table](https://iceberg.apache.org/docs/latest/spark-ddl/#create-table) on Spark page to see how to create the Iceberg table.

```scala
sparkSession.sql("create table czl_iceberg.demo.kafka_spark_iceberg (" +
  "user_id bigint, station_time string, score integer, local_time timestamp" +
  ") using iceberg")
```

### 3.4 写入iceberg

指定`outputMode`,`trigger`,`path`,`checkPointLocation`

```scala
val query: StreamingQuery = frame
  .writeStream
  .format("iceberg")
  //Iceberg supports append and complete output modes:
  //append: appends the rows of every micro-batch to the table
  //complete: replaces the table contents every micro-batch
  .outputMode("append")
  .trigger(Trigger.ProcessingTime(10, TimeUnit.SECONDS))
  .option("path", icebergPath)
  .option("checkpointLocation", checkpointPath)
  .start()
query.awaitTermination()
```

## 4. 完整代码

```scala
package com.czl.datalake.template.iceberg.kafka

import org.apache.spark.sql.functions.from_json
import org.apache.spark.sql.streaming.{StreamingQuery, Trigger}
import org.apache.spark.sql.types._
import org.apache.spark.sql.{DataFrame, Dataset, Row, SparkSession}

import java.util.concurrent.TimeUnit

/**
 * Author: CHEN ZHI LING
 * Date: 2023/3/22
 * Description:
 */
object KafkaToIceberg {

  def main(args: Array[String]): Unit = {

    val kafkaConsumer: String = "kafka ip"
    val kafkaTopic: String = "topic"
    val startingOffsets: String = "latest"
    val failOnDataLoss: String = "true"
    val maxOffsetsPerTrigger: Int = 3000
    val wareHousePath : String = "warehouse path"
    //A table name if the table is tracked by a catalog, like catalog.database.table_name
    val icebergPath : String = "czl_iceberg.demo.kafka_spark_iceberg"
    val checkpointPath : String = "checkpoint path"

    val schema: StructType = StructType(List(
      StructField("user_id", LongType),
      StructField("station_time", StringType),
      StructField("score", IntegerType),
      StructField("local_time", TimestampType)
    ))

    val sparkSession: SparkSession = SparkSession.builder().master("local[*]")
      //define catalog name
      .config("spark.sql.catalog.czl_iceberg", "org.apache.iceberg.spark.SparkCatalog")
      .config("spark.sql.catalog.czl_iceberg.type", "hadoop")
      .config("spark.sql.catalog.czl_iceberg.warehouse", wareHousePath)
      .getOrCreate()

    //The table should be created in prior to start the streaming query.
    //Refer SQL create table on Spark page to see how to create the Iceberg table.
    sparkSession.sql("create table czl_iceberg.demo.kafka_spark_iceberg (" +
      "user_id bigint, station_time string, score integer, local_time timestamp" +
      ") using iceberg")

    import sparkSession.implicits._
    val df: DataFrame = sparkSession
      .readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", kafkaConsumer)
      .option("subscribe", kafkaTopic)
      .option("startingOffsets", startingOffsets)
      .option("failOnDataLoss", failOnDataLoss)
      .option("maxOffsetsPerTrigger",maxOffsetsPerTrigger)
      .load()

    val frame: Dataset[Row] = df
      .select(from_json('value.cast("string"), schema) as "value")
      .select($"value.*")

    val query: StreamingQuery = frame
      .writeStream
      .format("iceberg")
      //Iceberg supports append and complete output modes:
      //append: appends the rows of every micro-batch to the table
      //complete: replaces the table contents every micro-batch
      .outputMode("append")
      .trigger(Trigger.ProcessingTime(10, TimeUnit.SECONDS))
      .option("path", icebergPath)
      .option("checkpointLocation", checkpointPath)
      .start()
    query.awaitTermination()
  }
}
```

## 5. pom文件参考

```xml
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_2.12</artifactId>
    <version>3.1.2</version>
</dependency>
<dependency>
    <groupId>org.apache.iceberg</groupId>
    <artifactId>iceberg-spark3-runtime</artifactId>
    <version>0.12.1</version>
</dependency>
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql-kafka-0-10_2.12</artifactId>
    <version>3.1.2</version>
</dependency>
```

## 6 git地址

[Chenzhiling/dataLake-template: some demos of delta,iceberg,hudi (github.com)](https://github.com/Chenzhiling/dataLake-template)

