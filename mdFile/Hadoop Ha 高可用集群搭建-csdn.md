# Hadoop Ha + Hbase+ Spark高可用集群搭建

# 前言

记录下Hadoop Ha + Hbase+ Spark高可用集群的搭建，主要包括每个组件的配置信息，以及启动步骤。

在ubuntu18.04环境下，可以正常使用，运行。

## 1.Ling-Ha集群架构信息

| 节点  | Nn   | Rm   | DFSZK | Dn   | Nm   | Jn   | Zoo  | spark | Hm   | Hr   |
| ----- | ---- | ---- | ----- | ---- | ---- | ---- | ---- | ----- | ---- | ---- |
| node1 | √    | √    | √     |      |      |      |      | √     | √    |      |
| node2 | √    | √    | √     |      |      |      |      | √     |      |      |
| node3 |      |      |       | √    | √    | √    | √    | √     |      | √    |
| node4 |      |      |       | √    | √    | √    | √    | √     |      | √    |
| node5 |      |      |       | √    | √    | √    | √    | √     |      | √    |
| node6 |      |      |       | √    | √    |      |      | √     |      | √    |

| 集群                        | 版本号 | 端口  |
| --------------------------- | ------ | ----- |
| Hadoop                      | 3.2.2  | 9870  |
| Yarn                        | 3.2.2  | 8088  |
| MapReduce JobHistory Server | 3.2.2  | 19888 |
| Spark-master                | 3.1.2  | 8080  |
| Spark-histoory              | 3.1.2  | 4000  |
| hbase                       | 2.2.7  | 16010 |
| Zookeeper                   | 3.4.6  | 2181  |

| 缩写  | 全称                    | 作用                        |
| ----- | ----------------------- | --------------------------- |
| Nm    | Namenode                | 元数据节点                  |
| Rm    | ResourceManager         | yarn资源管理节点            |
| DFSZK | DFSZKFailoverController | zookeeper监控节点,Ha配置    |
| Dn    | Datanode                | 数据节点                    |
| Nm    | NodeManager             | yarn单节点管理,与Rm通信     |
| Jn    | JournalNode             | 同步NameNode之间数据,Ha配置 |
| Zoo   | Zookeeper               | zookeeper集群               |
| Hm    | HMaster                 | Hbase主节点                 |
| Hr    | HRegionServer           | Hbase从节点                 |

------

## 2. Hadoop-Ha配置文件

- core-site.xml

```xml
<configuration>
        <!--Yarn 需要使用 fs.defaultFS 指定NameNode URI -->
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://Ling-Ha</value>
        </property>
       <!--指定hadoop临时目录 -->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>file:/usr/local/hadoop/tmp</value>
                <description>Abase for other temporary directories.</description>
        </property>
        <!-- 指定zookeeper地址 -->
        <property>
                <name>ha.zookeeper.quorum</name>
                <value>node3:2181,node4:2181,node5:2181</value>
        </property>
        <!--指定ZooKeeper超时间隔，单位毫秒 -->
        <property>
                <name>ha.zookeeper.session-timeout.ms</name>
                <value>10000</value>
        </property>
</configuration>
```

- hdfs-site.xml

```xml
<configuration>
        <!--集群名称，此值在接下来的配置中将多次出现务必注意同步修改-->
        <property>
                <name>dfs.nameservices</name>
                <value>Ling-Ha</value>
        </property>
        <!--所有的namenode列表，此处也只是逻辑名称，非namenode所在的主机名称-->
        <property>
                <name>dfs.ha.namenodes.Ling-Ha</name>
                <value>nn1,nn2</value>
        </property>
        <!--namenode之间用于RPC通信的地址，value填写namenode所在的主机地址-->
        <!--默认端口8020，注意Ling-Ha与nn1要和上文的配置一致-->
        <property> 
                <name>dfs.namenode.rpc-address.Ling-Ha.nn1</name> 
                <value>node1:9000</value> 
        </property>
        <!-- namenode的web访问地址，默认端口9870-->
	<property>
	       <name>dfs.namenode.http-address.Ling-Ha.nn1</name>
	       <value>node1:9870</value>
	</property>
        <!--备份NameNode node2主机配置-->
        <property>
                <name>dfs.namenode.rpc-address.Ling-Ha.nn2</name>
                <value>node2:9000</value>
        </property>
        <property>
	       <name>dfs.namenode.http-address.Ling-Ha.nn2</name>
	       <value>node2:9870</value>
	</property>
        <!--journalnode主机地址，最少三台，默认端口8485--> 
        <!--格式为 qjournal://jn1:port;jn2:port;jn3:port/${nameservices}--> 
        <property>
               <name>dfs.namenode.shared.edits.dir</name>
               <value>qjournal://node3:8485;node4:8485;node5:8485/Ling-Ha</value>
        </property>
        <!--日志文件输出路径，即journalnode读取变更的位置-->
        <!--NameNode的元数据在JournalNode上的存放位置 --> 
        <property>
              <name>dfs.journalnode.edits.dir</name>
              <value>/usr/local/hadoop/journal</value>
        </property>
        <!--启用自动故障转移-->
        <property>
             <name>dfs.ha.automatic-failover.enabled</name>
             <value>true</value>
        </property>
        <!--故障自动切换实现类-->
        <property>
             <name>dfs.client.failover.proxy.provider.Ling-Ha</name>
             <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
        </property>
        <!--故障时相互操作方式(namenode要切换active和standby)，选ssh方式-->
        <property>
             <name>dfs.ha.fencing.methods</name>
	     <value>sshfence</value>
	     <value>shell(true)</value>
        </property>
        <!--修改为自己用户的ssh key存放地址-->
        <property>
             <name>dfs.ha.fencing.ssh.private-key-files</name>
	     <value>/home/node1/.ssh/id_rsa</value>
        </property>
        <!--ssh连接超时-->
        <property>
             <name>dfs.ha.fencing.ssh.connect-timeout</name>
             <value>60000</value>
        </property>
        <!--*************************************************以上为高可用配置***************************************************-->
        <!--指定冗余副本个数-->
        <property>
                <name>dfs.replication</name>
                <value>3</value>
        </property>
        <!--指定namenode名称空间的存储地址--> 
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/usr/local/hadoop/tmp/dfs/name</value>
        </property>
        <!--指定datanode数据存储地址-->
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/usr/local/hadoop/tmp/dfs/data</value>
        </property>
        <property>
                <name>dfs.permissions</name>
                <value>false</value>
        </property>
</configuration>
```

- yarn-site.xml

```xml
<configuration>
 <!--********************************配置resourcemanager*******************************************************-->
	 <property>  
		<name>yarn.resourcemanager.cluster-id</name>  
		<value>Ling-yarn</value>  
	 </property>  
         <!--开启高可用-->
         <property>
                <name>yarn.resourcemanager.ha.enabled</name>
                <value>true</value>
         </property>
         <!-- 指定RM的名字 -->
         <property> 
                <name>yarn.resourcemanager.ha.rm-ids</name> 
                <value>rm1,rm2</value> 
         </property>
         <!-- 分别指定RM的地址 -->
         <property> 
                <name>yarn.resourcemanager.hostname.rm1</name> 
                <value>node1</value> 
         </property>     
         <property> 
		<name>yarn.resourcemanager.hostname.rm2</name> 
		<value>node2</value> 
         </property>
	<!--配置rm1--> 
	<property>
		<name>yarn.resourcemanager.webapp.address.rm1</name>
		<value>node1:8088</value>
	</property>
	<!--配置rm2-->  
	<property>
		<name>yarn.resourcemanager.webapp.address.rm2</name>
		<value>node2:8088</value>
        </property>
        <!-- 指定zk集群地址 -->
        <property>
                <name>yarn.resourcemanager.zk-address</name>
                <value>node3:2181,node4:2181,node5:2181</value>
        </property>
        <!--开启故障自动切换-->  
	<property>
		<name>yarn.resourcemanager.ha.automatic-failover.enabled</name>
		<value>true</value>
	</property>
	<property>
		<name>yarn.resourcemanager.ha.automatic-failover.zk-base-path</name>
		<value>/yarn-leader-election</value>
	</property>
        <!--开启自动恢复功能-->  
	<property> 
	        <name>yarn.resourcemanager.recovery.enabled</name>  
		<value>true</value>  
	</property> 
	<property>
	        <name>yarn.resourcemanager.store.class</name>
	        <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
	</property>
 <!--********************************配置resourcemanager*******************************************************-->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>
```

- mapred-site.xml

```xml
<configuration>
        <!-- 指定mr框架为yarn方式 -->
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
        <!-- 配置 MapReduce JobHistory Server 地址 ，默认端口10020 -->
        <property>
                <name>mapreduce.jobhistory.address</name>
                <value>node1:10020</value>
        </property>
        <!-- 配置 MapReduce JobHistory Server web ui 地址， 默认端口19888 -->    list.write.format("redis-nanhu").save("czl")
        <property>
                <name>mapreduce.jobhistory.webapp.address</name>
                <value>node1:19888</value>
        </property>
        <property>
                <name>yarn.app.mapreduce.am.env</name>
                <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
        </property>
        <property>
                <name>mapreduce.map.env</name>
                <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
        </property>
        <property>
                <name>mapreduce.reduce.env</name>
                <value>HADOOP_MAPRED_HOME=/usr/local/hadoop</value>
        </property> 
</configuration>
```

- workers

```xml
node3
node4
node5
node6
```

------

## 3.Hadoop-Ha初始化启动顺序

- 启动zookeep集群
- 依次启动所有journalnode

```shell
 #sbin目录下
 ./hadoop-daemon.sh start journalnode
```

- 格式化主节点namenode,然后启动

```shell
hdfs namenode -format
./hadoop-daemon.sh start namenode
```

- 在从节点 同步主节点namenode到备份namenode

```sh
hdfs namenode -bootstrapStandby
```

- 在主节点namenode格式化zk

```shell
hdfs zkfc -formatZK
```

- 停止主节点namenode和所有节点的journalnode

```sh
 #sbin目录下
./hadoop-daemon.sh stop namenode
./hadoop-daemon.sh stop journalnode
```

- 执行脚本,启动所有节点

```sh
start-dfs.sh
```

------

## 4.Hadoop Ha模式下 配置spark和日志服务器

- 配置spark-env.sh

```shell
#!/usr/bin/env bash
export SPARK_DIST_CLASSPATH=$(/usr/local/hadoop/bin/hadoop classpath)
export HADOOP_CONF_DIR=/usr/local/hadoop/etc/hadoop
export SPARK_MASTER_IP="填写主节点ip"
#"配置日志服务器"
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=4000 -Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=hdfs://Ling-Ha/sparkLog"
```

- 配置spark-default.conf

```shell
spark.eventLog.enabled           true
spark.eventLog.dir               hdfs://Ling-Ha/sparkLog
spark.eventLog.compress          true
spark.files                      file:///usr/local/spark/conf/hdfs-site.xml,file:///usr/local/spark/conf/core-site.xml
```

- 配置workers

```shell
node1
node2
node3
node4
node5
node6
```

- **spark.eventLog.dir 路径需修改成hdfs-site.xml 中dfs.nameservices配置的值,且不需要指定端口**
- **将hadoop的core-site.xml和hdfs-site.xml，放到spark的conf目录下，让spark能找到Hadoop的配置**

## 5.Hadoop Ha模式下配置Hbase Ha

- 修改hbase-env.sh

```shell
#java
export JAVA_HOME=/usr/loca/jvm/hbase
#hbase
export HBASE_CLASSPATH=/usr/local/hbase/conf
#不使用自带zookeeper
export HBASE_MANAGES_ZK=false
#不包含hadoop包
export HBASE_DISABLE_HADOOP_CLASSPATH_LOOKUP="true"
```

- 修改hbase-site.xml

```xml
<!--指定分布式-->
	<property>
	    <name>hbase.cluster.distributed</name>
	    <value>true</value>
	</property>
  <!--hbase在hdfs上路径-->
	<property>
	    <name>hbase.rootdir</name>
	    <value>hdfs://Ling-Ha/hbase</value>
	</property>
  <!--配置zk本地数据存放目录-->
	<property>
	    <name>hbase.zookeeper.property.dataDir</name>
	    <value>/usr/local/hbase/zookeeper</value>
	</property>
  <!--zoo集群-->
	<property>
	    <name>hbase.zookeeper.quorum</name>
	    <value>node3:2181,node4:2181,node5:2181</value>
	</property>
```

- 修改regionservers

```sh
#添加节点名称
node3
node4
node5
node6
```

- **将hadoop的core-site.xml和hdfs-site.xml，放到HBase的conf目录下，让HBase能找到Hadoop的配置**
- **配置Hbase Ha模式 在conf目录下创建backup-masters文件,输入备份节点名称**
