# zookeeper + kafka + kafka-eagle集群搭建

## 1. 前言

记录zookeeper集群和kafka集群的搭建步骤，并配置可视化工具kafka-eagle。

主要记录配置文件相关信息

在ubuntu18.04环境下，可以正常使用，运行。

注：kafka集群的使用，需要zookeeper集群

## 2. 节点架构信息

| 节点  | Zoo  | kafka | kafka-Eagle |
| ----- | ---- | ----- | ----------- |
| node3 | √    | √     | √           |
| node4 | √    | √     |             |
| node5 | √    | √     |             |

| 集群       | 版本号    | 端口      |
| ---------- | --------- | --------- |
| Zookeeper  | 3.4.6     | 2181      |
| Kafka      | 2.12-2.71 |           |
| KafkaEagle | 2.0.8     | 8048(web) |

## 3. zookeeper集群搭建

- 环境变量

```shell
export ZOOKEEPER_HOME=/usr/local/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

- 配置文件

```properties
# The number of milliseconds of each tick 心跳检测
tickTime=2000
# The number of ticks that the initial
# synchronization phase can take 
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.数据存放的位置
dataDir=/usr/local/zookeeper/data/zkdata/
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1 日志位置
dataLogDir=/usr/local/zookeeperlog/zklog/
#server：固定写法
#serverid：每个服务器的指定ID（必须处于1-255之间，必须每一台机器不能重复）
#host：主机名
#tickpot：心跳通信端口
#electionport：选举端口
server.1=node3:2888:3888
server.2=node4:2888:3888
server.3=node5:2888:3888
```

- 建立myid文件

```sh
 #里面存放的内容就是服务器的 id,就是 server.1=hadoop01:2888:3888 当中的 id, 就是 1，那么对应的每个服务器节点都应该做类似的操作
 mkdir /usr/local/zookeeper/data/zkdata/
 echo 1 > myid
```

- 一键启动关闭脚本

```shell
for host in node3 node4 node5
do
 echo "开启$host的zk"
 ssh $host "source /etc/profile;/usr/local/zookeeper/bin/zkServer.sh start"
done
```

```shell
for host in node3 node4 node 5
do
 echo "关闭$host的zk"
 ssh $host "source /etc/profile;/usr/local/zookeeper/bin/zkServer.sh stop"
done
```

------

## 4. kafka集群搭建

- 环境变量

```shell
export KAFKA_HOME=/usr/local/kafka
export PATH=$PATH:$KAFKA_HOME/bin
```

- 修改配置文件

```properties
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# see kafka.server.KafkaConfig for additional details and defaults

############################# Server Basics #############################

# The id of the broker. This must be set to a unique integer for each broker.修改
broker.id=0

############################# Socket Server Settings #############################

# The address the socket server listens on. It will get the value returned from 
# java.net.InetAddress.getCanonicalHostName() if not configured.
#   FORMAT:
#     listeners = listener_name://host_name:port
#   EXAMPLE:
#     listeners = PLAINTEXT://your.host.name:9092 修改
listeners=PLAINTEXT://node3:9092

# Hostname and port the broker will advertise to producers and consumers. If not set, 
# it uses the value for "listeners" if configured.  Otherwise, it will use the value
# returned from java.net.InetAddress.getCanonicalHostName().修改
advertised.listeners=PLAINTEXT://node3:9092

# Maps listener names to security protocols, the default is for them to be the same. See the config documentation for more details
#listener.security.protocol.map=PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL

# The number of threads that the server uses for receiving requests from the network and sending responses to the network
num.network.threads=3

# The number of threads that the server uses for processing requests, which may include disk I/O
num.io.threads=8

# The send buffer (SO_SNDBUF) used by the socket server
socket.send.buffer.bytes=102400

# The receive buffer (SO_RCVBUF) used by the socket server
socket.receive.buffer.bytes=102400

# The maximum size of a request that the socket server will accept (protection against OOM)
socket.request.max.bytes=104857600


############################# Log Basics #############################

# A comma separated list of directories under which to store log files 修改
log.dirs=/usr/local/kafka/kafkaLog

# The default number of log partitions per topic. More partitions allow greater
# parallelism for consumption, but this will also result in more files across
# the brokers.
num.partitions=1

# The number of threads per data directory to be used for log recovery at startup and flushing at shutdown.
# This value is recommended to be increased for installations with data dirs located in RAID array.
num.recovery.threads.per.data.dir=1

############################# Internal Topic Settings  #############################
# The replication factor for the group metadata internal topics "__consumer_offsets" and "__transaction_state"
# For anything other than development testing, a value greater than 1 is recommended to ensure availability such as 3.
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

############################# Log Flush Policy #############################

# Messages are immediately written to the filesystem but by default we only fsync() to sync
# the OS cache lazily. The following configurations control the flush of data to disk.
# There are a few important trade-offs here:
#    1. Durability: Unflushed data may be lost if you are not using replication.
#    2. Latency: Very large flush intervals may lead to latency spikes when the flush does occur as there will be a lot of data to flush.
#    3. Throughput: The flush is generally the most expensive operation, and a small flush interval may lead to excessive seeks.
# The settings below allow one to configure the flush policy to flush data after a period of time or
# every N messages (or both). This can be done globally and overridden on a per-topic basis.

# The number of messages to accept before forcing a flush of data to disk
#log.flush.interval.messages=10000

# The maximum amount of time a message can sit in a log before we force a flush
#log.flush.interval.ms=1000

############################# Log Retention Policy #############################

# The following configurations control the disposal of log segments. The policy can
# be set to delete segments after a period of time, or after a given size has accumulated.
# A segment will be deleted whenever *either* of these criteria are met. Deletion always happens
# from the end of the log.

# The minimum age of a log file to be eligible for deletion due to age
log.retention.hours=24

# A size-based retention policy for logs. Segments are pruned from the log unless the remaining
# segments drop below log.retention.bytes. Functions independently of log.retention.hours.
#log.retention.bytes=1073741824

# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=1073741824

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000

############################# Zookeeper #############################

# Zookeeper connection string (see zookeeper docs for details).
# This is a comma separated host:port pairs, each corresponding to a zk
# server. e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002".
# You can also append an optional chroot string to the urls to specify the
# root directory for all kafka znodes.
zookeeper.connect=node3:2181,node4:2181,node5:2181

# Timeout in ms for connecting to zookeeper
zookeeper.connection.timeout.ms=18000

############################# Group Coordinator Settings #############################
# The following configuration specifies the time, in milliseconds, that the GroupCoordinator will delay the initial consumer rebalance.
# The rebalance will be further delayed by the value of group.initial.rebalance.delay.ms as new members join the group, up to a maximum of max.poll.interval.ms.
# The default value for this is 3 seconds.
# We override this to 0 here as it makes for a better out-of-the-box experience for development and testing.
# However, in production environments the default value of 3 seconds is more suitable as this will help to avoid unnecessary, and potentially expensive, rebalances during application startup.
group.initial.rebalance.delay.ms=0
#删除kafka topic需要设置
delete.topic.enable=true
```

- 修改每个节点对应的**server.properties 文件的 broker.id和listenrs**

```properties
broker.id=1
listeners=PLAINTEXT://node4:9092
advertised.listeners=PLAINTEXT://node4:9092
 

broker.id=2
listeners=PLAINTEXT://node5:9092
advertised.listeners=PLAINTEXT://node5:9092
```

- 启动脚本

```shell
for host in node3 node4 node5
do
 echo "开启$host的kafka"
 ssh $host "source /etc/profile;/usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties >/dev/null 2>&1 &"
done
#---------------------------------------------------------------------------------------------------
for host in node3 node4 node5
do
 echo "关闭$host的kafka"
 ssh $host "source /etc/profile;/usr/local/kafka/bin/kafka-server-stop.sh >/dev/null 2>&1 &"
done
```

------

## 5. kafka-eagle 搭建

- 环境变量

```shell
#kfaka-eagle
export KE_HOME=/usr/local/kafkaEagle
export PATH=$PATH:$KE_HOME/bin
```

- 配置文件 使用mysql作为存储

```properties
######################################
# multi zookeeper & kafka cluster list
# Settings prefixed with 'kafka.eagle.' will be deprecated, use 'efak.' instead 修改
######################################
efak.zk.cluster.alias=cluster1
cluster1.zk.list=node3:2181,node4:2181,node5:2181
#cluster2.zk.list=xdn10:2181,xdn11:2181,xdn12:2181

######################################
# zookeeper enable acl
######################################
#cluster1.zk.acl.enable=false
#cluster1.zk.acl.schema=digest
#cluster1.zk.acl.username=test
#cluster1.zk.acl.password=test123

######################################
# broker size online list
######################################
cluster1.efak.broker.size=20

######################################
# zk client thread limit
######################################
kafka.zk.limit.size=32

######################################
# EFAK webui port
######################################
efak.webui.port=8048

######################################
# kafka jmx acl and ssl authenticate
######################################
#cluster1.efak.jmx.acl=false
#cluster1.efak.jmx.user=keadmin
#cluster1.efak.jmx.password=keadmin123
#cluster1.efak.jmx.ssl=false
#cluster1.efak.jmx.truststore.location=/data/ssl/certificates/kafka.truststore
#cluster1.efak.jmx.truststore.password=ke123456

######################################
# kafka offset storage
######################################
cluster1.efak.offset.storage=kafka
#cluster2.efak.offset.storage=zk

######################################
# kafka jmx uri
######################################
#cluster1.efak.jmx.uri=service:jmx:rmi:///jndi/rmi://%s/jmxrmi

######################################
# kafka metrics, 15 days by default
######################################
efak.metrics.charts=true
efak.metrics.retain=15

######################################
# kafka sql topic records max
######################################
efak.sql.topic.records.max=5000
efak.sql.topic.preview.records.max=10

######################################
# delete kafka topic token
######################################
efak.topic.token=keadmin

######################################
# kafka sasl authenticate
######################################
cluster1.efak.sasl.enable=false
cluster1.efak.sasl.protocol=SASL_PLAINTEXT
cluster1.efak.sasl.mechanism=SCRAM-SHA-256
cluster1.efak.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="kafka" password="kafka-eagle";
cluster1.efak.sasl.client.id=
cluster1.efak.blacklist.topics=
cluster1.efak.sasl.cgroup.enable=false
cluster1.efak.sasl.cgroup.topics=
cluster2.efak.sasl.enable=false
cluster2.efak.sasl.protocol=SASL_PLAINTEXT
cluster2.efak.sasl.mechanism=PLAIN
cluster2.efak.sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required username="kafka" password="kafka-eagle";
cluster2.efak.sasl.client.id=
cluster2.efak.blacklist.topics=
cluster2.efak.sasl.cgroup.enable=false
cluster2.efak.sasl.cgroup.topics=

######################################
# kafka ssl authenticate
######################################
cluster3.efak.ssl.enable=false
cluster3.efak.ssl.protocol=SSL
cluster3.efak.ssl.truststore.location=
cluster3.efak.ssl.truststore.password=
cluster3.efak.ssl.keystore.location=
cluster3.efak.ssl.keystore.password=
cluster3.efak.ssl.key.password=
cluster3.efak.ssl.endpoint.identification.algorithm=https
cluster3.efak.blacklist.topics=
cluster3.efak.ssl.cgroup.enable=false
cluster3.efak.ssl.cgroup.topics=

######################################
# kafka sqlite jdbc driver address
######################################
#efak.driver=org.sqlite.JDBC
#efak.url=jdbc:sqlite:/hadoop/kafka-eagle/db/ke.db
#efak.username=root
#efak.password=www.kafka-eagle.org

######################################
# kafka mysql jdbc driver address 修改
######################################
efak.driver=com.mysql.cj.jdbc.Driver
efak.url=jdbc:mysql://ip:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
efak.username=username
efak.password=password
```

- 启动

```shell
cd /usr/local/kafkaEagle/bin
./ke.sh start
#web页面
localhost:8048
```

