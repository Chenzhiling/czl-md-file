# Kafka-Hudi使用

## 1. 前言

Hudi Version : 0.10.1

Spark Version : 3.1.2

Scala Version : 2.12.10

使用Spark Structured Streaming将Kafka的数据写入到Hudi数据湖中

## 2. kafka原始数据格式

| 名称         | 类型      |
| ------------ | --------- |
| user_id      | Long      |
| station_time | String    |
| score        | Int       |
| local_time   | Timestamp |

## 3. 步骤详解

### 3.1 构建Spark

大致的步骤，与入湖delta和iceberg是一致的

```scala
val sparkSession: SparkSession = SparkSession.builder().master("local[*]")
  .config("spark.serializer","org.apache.spark.serializer.KryoSerializer")
  .config("spark.sql.extensions","org.apache.spark.sql.hudi.HoodieSparkSessionExtension")
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

### 3.4 写入Hudi

指定以下参数

`RECORDKEY_FIELD.key`,

`PRECOMBINE_FIELD.key`，

``HoodieWriteConfig.TBL_NAME.key`，

`PARTITIONPATH_FIELD.key()`分区可以为空

`outputMode`,

`trigger`,

`path`,

`checkPointLocation`

```scala
val query: StreamingQuery = frame
  .writeStream
  .format("hudi")
  .option(RECORDKEY_FIELD.key, "user_id")
  .option(PRECOMBINE_FIELD.key, "user_id")
  .option(PARTITIONPATH_FIELD.key(), partitionFields)
  .option(HoodieWriteConfig.TBL_NAME.key, hoodieTableName)
  .outputMode("append")
  .option("path", lakePath)
  .option("checkpointLocation", checkpointLocation)
  .trigger(Trigger.ProcessingTime(10,TimeUnit.SECONDS))
  .start()
query.awaitTermination()
```

## 4. 完整代码

```scala
package com.czl.datalake.template.hudi.kafka

import org.apache.hudi.DataSourceWriteOptions.{PARTITIONPATH_FIELD, PRECOMBINE_FIELD, RECORDKEY_FIELD}
import org.apache.hudi.config.HoodieWriteConfig
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
object KafkaToHudi {

  def main(args: Array[String]): Unit = {

    val kafkaConsumer: String = "kafka consumer"
    val kafkaTopic: String = "topic"
    val startingOffsets: String = "latest"
    val failOnDataLoss: String = "true"
    val maxOffsetsPerTrigger: Int = 3000
    val hoodieTableName : String  = "hoodieTableName"
    val lakePath : String = "lakePath"
    val checkpointLocation : String = "checkpointLocation"
    val partitionFields : String = Array().mkString(",")

    val schema: StructType = StructType(List(
      StructField("user_id", LongType),
      StructField("station_time", StringType),
      StructField("score", IntegerType),
      StructField("local_time", TimestampType)
    ))

    val sparkSession: SparkSession = SparkSession.builder().master("local[*]")
      .config("spark.serializer","org.apache.spark.serializer.KryoSerializer")
      .config("spark.sql.extensions","org.apache.spark.sql.hudi.HoodieSparkSessionExtension")
      .getOrCreate()

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

    val frame: Dataset[Row] = df.select(from_json('value.cast("string"), schema) as "value").select($"value.*")

    val query: StreamingQuery = frame
      .writeStream
      .format("hudi")
      .option(RECORDKEY_FIELD.key, "user_id")
      .option(PRECOMBINE_FIELD.key, "user_id")
      .option(PARTITIONPATH_FIELD.key(), partitionFields)
      .option(HoodieWriteConfig.TBL_NAME.key, hoodieTableName)
      .outputMode("append")
      .option("path", lakePath)
      .option("checkpointLocation", checkpointLocation)
      .trigger(Trigger.ProcessingTime(10,TimeUnit.SECONDS))
      .start()

    query.awaitTermination()
  }
}
```

## 5. pom文件参考

```xml
<dependency>
    <groupId>org.apache.hudi</groupId>
    <artifactId>hudi-spark3.1.2-bundle_2.12</artifactId>
    <version>0.10.1</version>
</dependency>
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql_2.12</artifactId>
    <version>3.1.2</version>
</dependency>
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-sql-kafka-0-10_2.12</artifactId>
    <version>3.1.2</version>
</dependency>
```

## 6 git地址

[Chenzhiling/dataLake-template: Some demos of using Spark to write MySQL and Kafka data to data lake,such as Delta,Hudi,Iceberg (github.com)](https://github.com/Chenzhiling/dataLake-template)

