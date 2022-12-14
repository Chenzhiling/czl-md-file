# Unbuntu 18.04 ElasticSearch集群+Head插件+Kibana安装

## 1. 前言

因为部分任务需要使用到Es，所以这次将Es集群搭建在原先已经安装好的大数据集群上，顺便搭建了Head，Kibana等工具。

## 2. 集群设计

这次选择将Es集群搭建在3台机器上，再另外选择一台机器搭建Head，KIbana等可视化工具。

ElasticSearch版本7.8.0

Kibana版本7.8.0

安装Es需要满足Java1.8以上

安装Head，Kibana需要node，npm

| 节点  | ElasticSearch | Head | Kibana |
| ----- | ------------- | ---- | ------ |
| node2 |               | √    | √      |
| node3 | √             |      |        |
| node4 | √             |      |        |
| node5 | √             |      |        |

## 3. ElasticSearch安装

- 解压安装，赋予普通用户权限，es不能使用root用户

```shell
sudo chown -R node3:node3 /usr/local/es-cluster
```

- 修改elsaticSearch.yaml，进行分发，并进行对应修改

```yaml
# ======================== Elasticsearch Configuration =========================
#
# NOTE: Elasticsearch comes with reasonable defaults for most settings.
#       Before you set out to tweak and tune the configuration, make sure you
#       understand what are you trying to accomplish and the consequences.
#
# The primary way of configuring a node is via this file. This template lists
# the most important settings you may want to configure for a production cluster.
#
# Please consult the documentation for further information on configuration options:
# https://www.elastic.co/guide/en/elasticsearch/reference/index.html
#
# ---------------------------------- Cluster -----------------------------------
#
# es集群的名称
#
cluster.name: Ling-Es
#
# ------------------------------------ Node ------------------------------------
#
# es集群当前节点的名称
#
node.name: node3
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# 数据路径
#
path.data: /usr/local/es-cluster/data
#jiqun
# 日志路径
#
path.logs: /usr/local/es-cluster/logs
#
# ----------------------------------- Memory -----------------------------------
#
# Lock the memory on startup:
#
#bootstrap.memory_lock: true
#
# Make sure that the heap size is set to about half the memory available
# on the system and that the owner of the process is allowed to use this
# limit.
#
# Elasticsearch performs poorly when the system is swapping the memory.
#
# ---------------------------------- Network -----------------------------------
#
# 当前节点ip
#
network.host: ip
#
# 端口
#
http.port: 9200
#
# For more information, consult the network module documentation.
#
#cluster transport port
transport.tcp.port: 9300
transport.tcp.compress: true
#允许跨域访问，使用head插件需要配置
http.cors.enabled: true
http.cors.allow-origin: "*"
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
# 填写集群所有节点的ip
#
discovery.seed_hosts: ["node3:9300", "node4:9300","node5:9300"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
# es集群主节点
#
cluster.initial_master_nodes: ["node3"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# ---------------------------------- Gateway -----------------------------------
#
# Block initial recovery after a full cluster restart until N nodes are started:
#
#gateway.recover_after_nodes: 3
#
# For more information, consult the gateway module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
```

- **修改系统配置，满足Es集群启动的最低要求，需切换到root用户，进行操作。普通用户执行失败**

```shell
##############################
vim /etc/sysctl.conf
#添加以下配置
vm.max_map_count=655360
##############################
vim /etc/security/limits.conf
#添加以下配置
node3 soft nofile 65536
node3 hard nofile 65536
##############################
#重启生效
sysctl -p
```

- 后台启动

```shell
cd /usr/local/elastic/es-cluster/bin/
./elasticsearch -d
```

## 4. Head插件安装

- 需要提前安装node，npm，或者cnpm

```shell
node -v
npm -v
cnpm -v
```

- 安装

```shell
git clone git://github.com/mobz/elasticsearch-head.git
cd elasticsearch-head/
cnpm install
```

- 后台启动

```shell
nohup npm start &
```

## 5. Kibana安装

- 安装

```shell
sudo tar -zxvf kibana-7.8.0-linux-x86_64.tar.gz -C /usr/local/
```

- 配置kibana.yml

```yaml
# 端口号
server.port: 5601

# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid values.
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
# 配置成了0.0.0.0这样别的主机也能访问
server.host: "0.0.0.0"

# Enables you to specify a path to mount Kibana at if you are running behind a proxy.
# Use the `server.rewriteBasePath` setting to tell Kibana if it should remove the basePath
# from requests it receives, and to prevent a deprecation warning at startup.
# This setting cannot end in a slash.
#server.basePath: ""

# Specifies whether Kibana should rewrite requests that are prefixed with
# `server.basePath` or require that they are rewritten by your reverse proxy.
# This setting was effectively always `false` before Kibana 6.3 and will
# default to `true` starting in Kibana 7.0.
#server.rewriteBasePath: false

# The maximum payload size in bytes for incoming server requests.
#server.maxPayloadBytes: 1048576

# The Kibana server's name.  This is used for display purposes.
# 随便配了个名字
server.name: "Ling-Kibana"

# The URLs of the Elasticsearch instances to use for all your queries.
# 对应的es集群
elasticsearch.hosts: ["http://node3:9200","http://node4:9200","http://node5:9200"]

# When this setting's value is true Kibana uses the hostname specified in the server.host
# setting. When the value of this setting is false, Kibana uses the hostname of the host
# that connects to this Kibana instance.
#elasticsearch.preserveHost: true

# Kibana uses an index in Elasticsearch to store saved searches, visualizations and
# dashboards. Kibana creates a new index if the index doesn't already exist.
#kibana.index: ".kibana"

# The default application to load.
#kibana.defaultAppId: "home"

# If your Elasticsearch is protected with basic authentication, these settings provide
# the username and password that the Kibana server uses to perform maintenance on the Kibana
# index at startup. Your Kibana users still need to authenticate with Elasticsearch, which
# is proxied through the Kibana server.
#elasticsearch.username: "kibana_system"
#elasticsearch.password: "pass"

# Enables SSL and paths to the PEM-format SSL certificate and SSL key files, respectively.
# These settings enable SSL for outgoing requests from the Kibana server to the browser.
#server.ssl.enabled: false
#server.ssl.certificate: /path/to/your/server.crt
#server.ssl.key: /path/to/your/server.key

# Optional settings that provide the paths to the PEM-format SSL certificate and key files.
# These files are used to verify the identity of Kibana to Elasticsearch and are required when
# xpack.security.http.ssl.client_authentication in Elasticsearch is set to required.
#elasticsearch.ssl.certificate: /path/to/your/client.crt
#elasticsearch.ssl.key: /path/to/your/client.key

# Optional setting that enables you to specify a path to the PEM file for the certificate
# authority for your Elasticsearch instance.
#elasticsearch.ssl.certificateAuthorities: [ "/path/to/your/CA.pem" ]

# To disregard the validity of SSL certificates, change this setting's value to 'none'.
#elasticsearch.ssl.verificationMode: full

# Time in milliseconds to wait for Elasticsearch to respond to pings. Defaults to the value of
# the elasticsearch.requestTimeout setting.
#elasticsearch.pingTimeout: 1500

# Time in milliseconds to wait for responses from the back end or Elasticsearch. This value
# must be a positive integer.
#elasticsearch.requestTimeout: 30000

# List of Kibana client-side headers to send to Elasticsearch. To send *no* client-side
# headers, set this value to [] (an empty list).
#elasticsearch.requestHeadersWhitelist: [ authorization ]

# Header names and values that are sent to Elasticsearch. Any custom headers cannot be overwritten
# by client-side headers, regardless of the elasticsearch.requestHeadersWhitelist configuration.
#elasticsearch.customHeaders: {}

# Time in milliseconds for Elasticsearch to wait for responses from shards. Set to 0 to disable.
#elasticsearch.shardTimeout: 30000

# Time in milliseconds to wait for Elasticsearch at Kibana startup before retrying.
#elasticsearch.startupTimeout: 5000

# Logs queries sent to Elasticsearch. Requires logging.verbose set to true.
#elasticsearch.logQueries: false

# Specifies the path where Kibana creates the process ID file.
#pid.file: /var/run/kibana.pid

# Enables you specify a file where Kibana stores log output.
#logging.dest: stdout

# Set the value of this setting to true to suppress all logging output.
#logging.silent: false

# Set the value of this setting to true to suppress all logging output other than error messages.
#logging.quiet: false

# Set the value of this setting to true to log all events, including system usage information
# and all requests.
#logging.verbose: false

# Set the interval in milliseconds to sample system and process performance
# metrics. Minimum is 100ms. Defaults to 5000.
#ops.interval: 5000

# Specifies locale to be used for all localizable strings, dates and number formats.
# Supported languages are the following: English - en , by default , Chinese - zh-CN .
# 改成中文
i18n.locale: "zh-CN"
# 可以配置也可以不配置
# 配置了可以避免启动出现警告，输入超过32个字符串即可
xpack.reporting.encryptionKey: "mysql_sqlserver_oracle_hive_hbase_mongo_redis"
xpack.security.encryptionKey: "spark_flink_presto_tensorflow_pytorch_keras"
```

- 后台启动

```shell
nohup /usr/local/kibana/bin/kibana &
```

## 6. 安装完成

- ElasticSearch node3,4,5均可访问9200端口
- Head               node2可访问9100端口
- Kibana            node2可访问5601端口