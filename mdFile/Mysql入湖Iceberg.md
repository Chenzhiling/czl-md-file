# Mysql入湖Iceberg

使用`Scala`实现`Spark`高可用集群读取`Mysql`数据写入`Iceberg`数据湖，数据存储于`Hadoop`高可用集群

- Spark 3.3.3
- Hadoop 3.3.6
- Iceberg 1.3.0

## 代码

```scala
package com.czl.datalake.template.iceberg.mysql

import org.apache.spark.sql.SparkSession

object Test {
  def main(args: Array[String]): Unit = {
    //windows环境测试需要配置
//    System.setProperty("hadoop.home.dir", "D://hadoop3.3.0")
//    System.setProperty("HADOOP_USER_NAME", "czl")
    val spark = SparkSession.builder()
      .appName("mysqlToIceberg")
      .config("spark.sql.catalog.czlCatalog", "org.apache.iceberg.spark.SparkCatalog")
      .config("spark.sql.catalog.czlCatalog.type", "hadoop")
      //hadoop文件系统路径
      .config("spark.sql.catalog.czlCatalog.warehouse", "/data")
      .getOrCreate()

    try {
      // 1. 读取MySQL数据
      val mysqlDF = spark.read
        .format("jdbc")
        .option("url", "jdbc:mysql://10.10.10.10:6033")
        .option("driver", "com.mysql.cj.jdbc.Driver")
        .option("user", "root")
        .option("password", "root")
        //库名.表名
        .option("dbtable", "testDb.testTable")
        .load()
      //写入Iceberg表
      mysqlDF.write
        .format("iceberg")
         //存储在hadoop上的，库名.表名
        .save("czlCatalog.testDb.testTable")
    } catch {
      case e: Exception => e.printStackTrace()
    } finally {
      spark.stop()
    }
  }
}

```

## 依赖参考

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>dataLake-template-iceberg</artifactId>
        <groupId>com.czl</groupId>
        <version>1.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>dataLake-template-iceberg-spark-mysql</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>

    <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.24</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.12</artifactId>
            <version>3.3.3</version>
        </dependency>
        <dependency>
            <groupId>org.apache.iceberg</groupId>
            <artifactId>iceberg-spark-runtime-3.3_2.12</artifactId>
            <version>1.3.0</version>
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
                            <!--编译源码-->
                            <goal>compile</goal>
                            <!--编译测试源码-->
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

## 提示

- `core-site.xml,hdfs-site.xml`文件放置在resources文件夹下，以便于识别到`Hadoop`集群
- Spark高可用集群，每个节点上传`mysql-connector-java-8.0.33.jar`和`iceberg-spark-runtime-3.3_2.12-1.3.0.jar`这两个必须使用的依赖。

## 本地模式提交

```shell
 ./spark-submit --master local[*] --class com.czl.datalake.template.iceberg.mysql.Test /opt/dataLake-template-iceberg-spark-mysql-1.0.jar
```

## 集群模式提交，jar在本地

```shell
./spark-submit --master spark://node1:7077 --deploy-mode cluster --class com.czl.datalake.template.iceberg.mysql.Test /opt/dataLake-template-iceberg-spark-mysql-1.0.jar
```

## 集群模式提交，jar在hadoop

```shell
./spark-submit --master spark://node1:7077 --deploy-mode cluster --class com.czl.datalake.template.iceberg.mysql.Test hdfs://ling//workspace/project/dataLake-template-iceberg-spark-mysql-1.0.jar
```

## Yarn模式提交

```shell
./spark-submit --master yarn --deploy-mode cluster --class com.czl.datalake.template.iceberg.mysql.Test hdfs://ling//workspace/project/dataLake-template-iceberg-spark-mysql-1.0.jar
```

