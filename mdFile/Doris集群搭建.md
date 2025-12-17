# Doris集群搭建

## 1、简介

搭建三节点存算一体`Doris`集群

`FE`、`BE`混合部署

版本号`2.1.9`

| 节点  | ip        | 作用  |
| ----- | --------- | ----- |
| node3 | 10.1.0.21 | FE,BE |
| node4 | 10.1.0.18 | FE,BE |
| node4 | 10.1.0.19 | FE,BE |

## 2、配置

配置每个节点`Ip`和`Java`环境

### 2.1、FE

```shell
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

#####################################################################
## The uppercase properties are read and exported by bin/start_fe.sh.
## To see all Frontend configurations,
## see fe/src/org/apache/doris/common/Config.java
#####################################################################

CUR_DATE=`date +%Y%m%d-%H%M%S`

# Log dir
LOG_DIR = ${DORIS_HOME}/log

# For jdk 8
JAVA_OPTS="-Xmx16384m -XX:+UseMembar -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=7 -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:-CMSParallelRemarkEnabled -XX:CMSInitiatingOccupancyFraction=80 -XX:SoftRefLRUPolicyMSPerMB=0 -Xloggc:$DORIS_HOME/log/fe.gc.log.$DATE"
# For jdk 17, this JAVA_OPTS will be used as default JVM options
JAVA_OPTS_FOR_JDK_17="-Dfile.encoding=UTF-8 -Djavax.security.auth.useSubjectCredsOnly=false -Xmx8192m -Xms8192m -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=$LOG_DIR -Xlog:gc*,classhisto*=trace:$LOG_DIR/fe.gc.log.$CUR_DATE:time,uptime:filecount=10,filesize=50M --add-opens=java.base/java.nio=ALL-UNNAMED --add-opens java.base/jdk.internal.ref=ALL-UNNAMED --add-opens java.base/sun.nio.ch=ALL-UNNAMED --add-opens java.xml/com.sun.org.apache.xerces.internal.jaxp=ALL-UNNAMED"

# Set your own JAVA_HOME
JAVA_HOME=/opt/java

##
## the lowercase properties are read by main program.
##

# store metadata, must be created before start FE.
# Default value is ${DORIS_HOME}/doris-meta
# meta_dir = ${DORIS_HOME}/doris-meta

# Default dirs to put jdbc drivers,default value is ${DORIS_HOME}/jdbc_drivers
# jdbc_drivers_dir = ${DORIS_HOME}/jdbc_drivers

http_port = 8030
rpc_port = 9020
query_port = 9030
edit_log_port = 9010
arrow_flight_sql_port = -1

# Choose one if there are more than one ip except loopback address. 
# Note that there should at most one ip match this list.
# If no ip match this rule, will choose one randomly.
# use CIDR format, e.g. 10.10.10.0/24 or IP format, e.g. 10.10.10.1
# Default value is empty.
priority_networks = 10.1.0.21

# Advanced configurations 
# log_roll_size_mb = 1024
# INFO, WARN, ERROR, FATAL
sys_log_level = INFO
# NORMAL, BRIEF, ASYNC
sys_log_mode = ASYNC
# sys_log_roll_num = 10
# sys_log_verbose_modules = org.apache.doris
# audit_log_dir = $LOG_DIR
# audit_log_modules = slow_query, query
# audit_log_roll_num = 10
# meta_delay_toleration_second = 10
# qe_max_connection = 1024
# qe_query_timeout_second = 300
# qe_slow_log_ms = 5000

```

### 2.2、BE

`8040`端口冲突，`Yarn`占用，修改为`8041`

配置`storage_root_path`存储位置

```shell
# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing,
# software distributed under the License is distributed on an
# "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
# KIND, either express or implied.  See the License for the
# specific language governing permissions and limitations
# under the License.

CUR_DATE=`date +%Y%m%d-%H%M%S`

# Log dir
LOG_DIR="${DORIS_HOME}/log/"

# For jdk 8
JAVA_OPTS="-Dfile.encoding=UTF-8 -Xmx2048m -DlogPath=$LOG_DIR/jni.log -Xloggc:$DORIS_HOME/log/be.gc.log.$CUR_DATE -Djavax.security.auth.useSubjectCredsOnly=false -Dsun.security.krb5.debug=true -Dsun.java.command=DorisBE -XX:-CriticalJNINatives  -Darrow.enable_null_check_for_get=false"

# For jdk 9+, this JAVA_OPTS will be used as default JVM options
JAVA_OPTS_FOR_JDK_9="-Dfile.encoding=UTF-8 -Xmx2048m -DlogPath=$DORIS_HOME/log/jni.log -Xlog:gc:$LOG_DIR/be.gc.log.$CUR_DATE -Djavax.security.auth.useSubjectCredsOnly=false -Dsun.security.krb5.debug=true -Dsun.java.command=DorisBE -XX:-CriticalJNINatives --add-opens=java.base/java.nio=ALL-UNNAMED  -Darrow.enable_null_check_for_get=false"

# For jdk 17+, this JAVA_OPTS will be used as default JVM options
JAVA_OPTS_FOR_JDK_17="-Dfile.encoding=UTF-8 -Xmx2048m -DlogPath=$LOG_DIR/jni.log -Xlog:gc:$LOG_DIR/be.gc.log.$CUR_DATE -Djavax.security.auth.useSubjectCredsOnly=false -Dsun.security.krb5.debug=true -Dsun.java.command=DorisBE -XX:-CriticalJNINatives --add-opens=java.base/java.net=ALL-UNNAMED --add-opens=java.base/java.nio=ALL-UNNAMED  -Darrow.enable_null_check_for_get=false"

# since 1.2, the JAVA_HOME need to be set to run BE process.
JAVA_HOME=/opt/java

# https://github.com/apache/doris/blob/master/docs/zh-CN/community/developer-guide/debug-tool.md#jemalloc-heap-profile
# https://jemalloc.net/jemalloc.3.html
JEMALLOC_CONF="percpu_arena:percpu,background_thread:true,metadata_thp:auto,muzzy_decay_ms:5000,dirty_decay_ms:5000,oversize_threshold:0,prof:true,prof_active:false,lg_prof_interval:-1"
JEMALLOC_PROF_PRFIX="jemalloc_heap_profile_"

# ports for admin, web, heartbeat service 
be_port = 9060
webserver_port = 8041
heartbeat_service_port = 9050
brpc_port = 8060
arrow_flight_sql_port = -1

# HTTPS configures
enable_https = false
# path of certificate in PEM format.
ssl_certificate_path = "$DORIS_HOME/conf/cert.pem"
# path of private key in PEM format.
ssl_private_key_path = "$DORIS_HOME/conf/key.pem"


# Choose one if there are more than one ip except loopback address. 
# Note that there should at most one ip match this list.
# If no ip match this rule, will choose one randomly.
# use CIDR format, e.g. 10.10.10.0/24 or IP format, e.g. 10.10.10.1
# Default value is empty.
priority_networks = 10.1.0.21

# data root path, separate by ';'
# You can specify the storage type for each root path, HDD (cold data) or SSD (hot data)
# eg:
storage_root_path = /opt/doris2Data,medium:SSD
# storage_root_path = /home/disk1/doris,medium:SSD;/home/disk2/doris,medium:SSD;/home/disk2/doris,medium:HDD
# /home/disk2/doris,medium:HDD(default)
# 
# you also can specify the properties by setting '<property>:<value>', separate by ','
# property 'medium' has a higher priority than the extension of path
#
# Default value is ${DORIS_HOME}/storage, you should create it by hand.
# storage_root_path = ${DORIS_HOME}/storage

# Default dirs to put jdbc drivers,default value is ${DORIS_HOME}/jdbc_drivers
# jdbc_drivers_dir = ${DORIS_HOME}/jdbc_drivers

# Advanced configurations
# INFO, WARNING, ERROR, FATAL
sys_log_level = INFO
# sys_log_roll_mode = SIZE-MB-1024
# sys_log_roll_num = 10
# sys_log_verbose_modules = *
# log_buffer_level = -1
# palo_cgroups 

# aws sdk log level
#    Off = 0,
#    Fatal = 1,
#    Error = 2,
#    Warn = 3,
#    Info = 4,
#    Debug = 5,
#    Trace = 6
# Default to turn off aws sdk log, because aws sdk errors that need to be cared will be output through Doris logs
aws_log_level=0
## If you are not running in aws cloud, you can disable EC2 metadata
AWS_EC2_METADATA_DISABLED=true

```



```mysql
ALTER SYSTEM ADD FOLLOWER "10.1.0.18:9010";
ALTER SYSTEM ADD FOLLOWER "10.1.0.19:9010";
```

```she
//填写主节点ip
start_fe.sh --helper 10.1.0.21:9010 --daemon
start_fe.sh --helper 10.1.0.21:9010 --daemon
```

```sh
mysql -h 10.1.0.21 -P 9030 -uroot
```

```sql
ALTER SYSTEM ADD BACKEND "10.1.0.21:9050";
ALTER SYSTEM ADD BACKEND "10.1.0.18:9050";
ALTER SYSTEM ADD BACKEND "10.1.0.19:9050";
```

```sql
SET PASSWORD = PASSWORD('zhenshi');
```

## 3、启动FE

### 3.1、单独建立元数据存储位置

```shell
## Use a separate disk for FE metadata
mkdir -p <doris_meta_created>
   
## Create FE metadata directory symlink
ln -s <doris_meta_created> <doris_meta_original>
```

### 3.2、启动

```shell
bin/start_fe.sh --daemon
```

### 3.3、部署`FE Follower `节点

将node4、node5添加为Follower节点

```sql
mysql -h 10.1.0.21 -P 9030 -u root
ALTER SYSTEM ADD FOLLOWER "10.1.0.18:9010";
ALTER SYSTEM ADD FOLLOWER "10.1.0.19:9010";
```

### 3.4、启动`FE Follower `节点

```shell
//填写主节点node3的ip
start_fe.sh --helper 10.1.0.21:9010 --daemon
start_fe.sh --helper 10.1.0.21:9010 --daemon
```

## 4、启动BE

### 4.1、添加BE节点

```sql
ALTER SYSTEM ADD BACKEND "10.1.0.21:9050";
ALTER SYSTEM ADD BACKEND "10.1.0.18:9050";
ALTER SYSTEM ADD BACKEND "10.1.0.19:9050";
```

### 4.2、启动BE

```shell
bin/start_be.sh --daemon
```

## 5、验证

`Mysql`环境输入下列指令

```sql
show frontends \G
show backends \G
```

`Web`环境可打卡以下网址

- Web UI: [http://10.1.0.21:8030](http://10.1.0.21:8030/)
- 监控指标: http://10.1.0.21:8041/metrics