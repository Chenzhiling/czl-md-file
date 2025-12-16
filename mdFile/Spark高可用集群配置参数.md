# Spark高可用集群配置参数

Spark高可用集群配置参数，开启rest提交模式，并搭配日志存储于Hadoop系统

## 1、spark-default.conf参数

```shell
#开启rest
spark.master.rest.enabled        true
spark.master.rest.port           6066
#开启日志
spark.eventLog.enabled           true
spark.eventLog.dir               hdfs://ling/sparkLog
spark.eventLog.compress          true
spark.history.fs.logDirectory    hdfs://ling/sparkLog

spark.deploy.recoveryMode       ZOOKEEPER
spark.deploy.zookeeper.url      node2:2181,node3:2181,node4:2181
spark.deploy.zookeeper.dir      /lingSpark
```

## 2、spark-env.sh参数

### 2.1、主节点node1参数

```shell
//节点名称
export SPARK_MASTER_HOST=node1
export SPARK_MASTER_PORT=7077
//web端口
export SPARK_MASTER_WEBUI_PORT=8080
//java位置
export JAVA_HOME=/opt/java
//hadoop参数
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export YARN_CONF_DIR=/opt/hadoop/etc/hadoop
//日志参数
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=4000 -Dspark.history.retainedApplications=30 -Dspark.history.fs.logDirectory=hdfs://zhenshi/sparkLog"
```

### 2.2从节点node2参数

名称和web端口必须不同

```shell
export SPARK_MASTER_HOST=node2
export SPARK_MASTER_PORT=7077
//web端口
export SPARK_MASTER_WEBUI_PORT=8081
export JAVA_HOME=/opt/java
//hadoop参数
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export YARN_CONF_DIR=/opt/hadoop/etc/hadoop
//日志参数
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=4000 -Dspark.history.retainedApplications=30 -Dspark.history.fs.logDirectory=hdfs://zhenshi/sparkLog"

```

## workers参数

填写计算节点名称

```shell
# A Spark Worker will be started on each of the machines listed below.
node3
node4
node5
```

