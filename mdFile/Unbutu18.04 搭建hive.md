# Unbutu18.04 搭建hive,使用mysql作为元数据仓库

## 1.前言

在已经搭建的Hadoop Ha集群上搭建hive

hive版本 3.1.2

mysql版本 5.7.36

## 2. mysql安装

- 安装

```shell
sudo apt install mysql-server
```

- 使用debian-sys-maint账号登录

```shell
mysql -udebian-sys-maint -p
# 密码在etc/mysql/debian.cnf
```

- 修改root密码

```shell
use mysql;
update user set authentication_string=PASSWORD('123456') where user='root' and host= 'localhost';
# 这一步很重要
update user set plugin="mysql_native_password";
flush privileges;
```

- 重启

```shell
sudo service mysql restart
```

## 3. hive安装

- 环境变量

```shell
export HIVE_HOME=/usr/local/hive
export PATH=$PATH:$HIVE_HOME/bin
```

- 配置hive-env.sh

```shell
# Set HADOOP_HOME to point to a specific hadoop install directory
# Hadoop目录
HADOOP_HOME=/usr/local/hadoop
# Hive Configuration Directory can be controlled by:
# Hive 配置文件路径
export HIVE_CONF_DIR=/usr/local/hive/conf
```

- 配置hive-site.xml

```xml
<!--mysql配置 更改ip,用户名，密码-->  
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://ip:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false</value>
    <description>JDBC connect string for a JDBC metastore</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
    <description>username to use against metastore database</description>
  </property>
 <!--mysql密码-->
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>123456</value>
    <description>password to use against metastore database</description>
  </property>
  <!--hive在hdfs配置-->
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/hive</value>
    <description>location of default database for the warehouse</description>
  </property>
  <property>
    <!--此处使用了hdfs-master的ip-->
    <name>hive.server2.thrift.bind.host</name>
    <value>ip</value>
   </property>
  <!--这是hiveserver2 -->
   <property>
     <name>hive.server2.thrift.port</name>
     <value>10000</value>
   </property>
```

- 配置log4j.properties

```properties
property.hive.log.dir = /usr/local/hive/log
```

- hadoop core-site.xml增加配置

```xml
        <property>
                <name>hadoop.proxyuser.root.hosts</name>
                <value>*</value>
        </property>
        <property>
                <name>hadoop.proxyuser.root.groups</name>
                <value>*</value>
        </property>
        <!--配置hiveserver安装节点的用户名 -->
        <property>
                <name>hadoop.proxyuser.用户名.hosts</name>
                <value>*</value>
        </property>
        <property>
                <name>hadoop.proxyuser.用户名.groups</name>
                <value>*</value>
        </property>
```

- 启动

```sh
#beeline启动
beeline
!connect jdbc:hive2://ip:10000
```

