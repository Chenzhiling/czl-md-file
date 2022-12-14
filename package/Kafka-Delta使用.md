# Kafka-Delta使用

## 1. 前言

Delta Version : 1.0

spark : 3.1.2

scala : 2.12.10

将Kafka的数据流式写入到Delta数据湖中

## 2. kafka原始数据格式

| 名称         | 类型   |
| ------------ | ------ |
| user_id      | Long   |
| station_time | String |
| score        | Int    |
| local_time   | String |

## 3.1 构建spark

```scala
val spark: SparkSession = SparkSession.builder()
    .appName("KafkaToDelta")
    .master("local[*]")
    .config("spark.sql.extensions","io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog","org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .getOrCreate()
```

## 3.2 读取Kafka

```scala
val df: DataFrame = spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", KAFKA_CONSUMER)
    .option("subscribe", KAFKA_TOPIC)
    .option("startingOffsets", "earliest")
    .option("failOnDataLoss", "false")
    .load()
```

## 3.3 进行Etl

对local_time列进行Etl，生成rank_hour和rank_day两列

```scala
val frame: Dataset[Row] = df.select(from_json('value.cast("string"), schema) as "value").select($"value.*")
    .select(date_format($"local_time".cast(TimestampType), "HH").as("rank_hour"), $"*")
    .select(date_format($"local_time".cast(TimestampType), "yyyy-MM-dd").as("rank_day"), $"*")
```

## 3.4 写入Delta

```scala
val query: StreamingQuery = frame.writeStream.format("delta")
    .option("path", DELTA_PATH)
    .option("checkpointLocation", DELTA_PATH).partitionBy("rank_day","rank_hour")
    .start()
query.awaitTermination()
```

## 4. 完整代码

```scala
object kafkaToDelta {

  val KAFKA_CONSUMER = "kafka地址"
  val KAFKA_TOPIC = "topic"
  val DELTA_PATH = "hdfs://ip:9000/kafkaToDelta/"

  def main(args: Array[String]): Unit = {
    val spark: SparkSession = SparkSession.builder()
      .appName("KafkaToDelta")
      .master("local[*]")
      .config("spark.sql.extensions","io.delta.sql.DeltaSparkSessionExtension")
      .config("spark.sql.catalog.spark_catalog","org.apache.spark.sql.delta.catalog.DeltaCatalog")
      .getOrCreate()
    spark.sparkContext.setLogLevel("WARN")
    import spark.implicits._
    val df: DataFrame = spark
      .readStream
      .format("kafka")
      .option("kafka.bootstrap.servers", KAFKA_CONSUMER)
      .option("subscribe", KAFKA_TOPIC)
      .option("startingOffsets", "earliest")
      .option("failOnDataLoss", "false")
      .load()
    val schema: StructType = StructType(List(
      StructField("user_id", LongType),
      StructField("station_time", StringType),
      StructField("score", IntegerType),
      StructField("local_time", StringType)
    ))

    val frame: Dataset[Row] = df.select(from_json('value.cast("string"), schema) as "value").select($"value.*")
      .select(date_format($"local_time".cast(TimestampType), "HH").as("rank_hour"), $"*")
      .select(date_format($"local_time".cast(TimestampType), "yyyy-MM-dd").as("rank_day"), $"*")

 //   val query: StreamingQuery = frame.writeStream.format("console").start()
    val query: StreamingQuery = frame.writeStream.format("delta")
      .option("path", DELTA_PATH)
      .option("checkpointLocation", DELTA_PATH).partitionBy("rank_day","rank_hour")
      .start()
    query.awaitTermination()
  }
}
```

## 5. pom文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.chenzhiling.bigdata</groupId>
    <artifactId>ApplicationScenarios</artifactId>
    <version>1.0</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <spark.version>3.1.2</spark.version>
    </properties>
    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.12</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql-kafka-0-10_2.12</artifactId>
            <version>${spark.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.commons</groupId>
                    <artifactId>commons-pool2</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!--不使用这个会找不到方法-->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-pool2</artifactId>
            <version>2.6.2</version>
        </dependency>
        <dependency>
            <groupId>io.delta</groupId>
            <artifactId>delta-core_2.12</artifactId>
            <version>1.0.0</version>
            <exclusions>
                <exclusion>
                    <groupId>org.antlr</groupId>
                    <artifactId>antlr4-runtime</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>2.12.10</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.scala-tools</groupId>
                <artifactId>maven-scala-plugin</artifactId>
                <version>2.15.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

