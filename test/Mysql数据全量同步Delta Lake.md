# Mysql数据全量同步Delta Lake

## Hadoop存储集群

| 部署模式     | DataNode节点 | 容量(TB) | 版本  |
| ------------ | ------------ | -------- | ----- |
| 分布式高可用 | 4            | 3.6      | 3.2.2 |

## Spark计算集群

| 部署模式   | worker数 | cores | 内存(GB) | 版本  |
| ---------- | -------- | ----- | -------- | ----- |
| Standalone | 5        | 76    | 135      | 3.1.2 |

## 入湖实验

Mysql数据单表全量入湖Delta Lake，存储在hdfs上。

集群5个节点共同参与，使用66cores，每个executor分配12G内存

- spark.executor.memory 12g

| 数据源 | 行数    | 字段数 | 表存储大小(MB) | 数据湖大小(MB) | 入湖耗时(秒) |
| ------ | ------- | ------ | -------------- | -------------- | ------------ |
| Mysql  | 8519406 | 41     | 2560           | 584.13         | 204          |
| Mysql  | 3474439 | 48     | 1075.2         | 136.44         | 84           |
| Mysql  | 1373584 | 44     | 368.64         | 52.62          | 44           |
| Mysql  | 1000000 | 16     | 71.68          | 40.91          | 24           |
| Mysql  | 552975  | 48     | 163.84         | 42.01          | 23           |

