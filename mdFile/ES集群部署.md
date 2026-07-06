# ES集群部署

## 1、es用户创建

```shell
# 创建用户组
groupadd es

# 创建用户并指定所属组
useradd es -g es

# 为 es 用户设置密码（密码zhenshi1969）
passwd es

# 授权 ES 安装目录
chown -R es:es /opt/es

# 授权数据和日志目录
chown -R es:es /opt/esData/
chown -R es:es /opt/esLog/
```

## 2、配置yaml

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
# Use a descriptive name for your cluster:
#
cluster.name: zhenshi-es
#
# ------------------------------------ Node ------------------------------------
#
# Use a descriptive name for the node:
#
node.name: node3
#
# Add custom attributes to the node:
#
#node.attr.rack: r1
#
# ----------------------------------- Paths ------------------------------------
#
# Path to directory where to store the data (separate multiple locations by comma):
#
path.data: /opt/esData
#
# Path to log files:
#
path.logs: /opt/esLog
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
# By default Elasticsearch is only accessible on localhost. Set a different
# address here to expose this node on the network:
#
network.host: 10.0.0.67
#
# By default Elasticsearch listens for HTTP traffic on the first free port it
# finds starting at 9200. Set a specific HTTP port here:
#
http.port: 9200
http.cors.enabled: true
http.cors.allow-origin: "*"
#
# For more information, consult the network module documentation.
#
# --------------------------------- Discovery ----------------------------------
#
# Pass an initial list of hosts to perform discovery when this node is started:
# The default list of hosts is ["127.0.0.1", "[::1]"]
#
discovery.seed_hosts: ["node3:9300", "node4:9300", "node5:9300"]
#
# Bootstrap the cluster using an initial set of master-eligible nodes:
#
cluster.initial_master_nodes: ["node3", "node4", "node5"]
#
# For more information, consult the discovery and cluster formation module documentation.
#
# ---------------------------------- Various -----------------------------------
#
# Require explicit names when deleting indices:
#
#action.destructive_requires_name: true
# 启用X-Pack安全认证
xpack.security.enabled: true

# 启用节点间通信SSL加密（多节点集群必须配置）
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

## 3、IK分词器

在 ES 的安装根目录下创建 `plugins/ik` 目录：`mkdir -p plugins/ik`。

将下载的 zip 包解压到该目录中：`unzip elasticsearch-analysis-ik-7.12.1.zip -d plugins/ik/`。

确认目录结构正确（`plugins/ik/` 下应包含 `config` 目录和 `.jar` 文件），然后重启 ES 服务。

## 4、Kibana安装

```shell
#创建用户
useradd kibana
chown -R kibana:kibana /opt/kibana/
```

```yaml
server.port: 5601
server.host: "0.0.0.0"
server.name: "Ling-Kibana"
# 对应的es集群
elasticsearch.hosts: ["http://node3:9200","http://node4:9200","http://node5:9200"]
i18n.locale: "zh-CN"
xpack.reporting.encryptionKey: "mysql_sqlserver_oracle_hive_hbase_mongo_redis"
xpack.security.encryptionKey: "spark_flink_presto_tensorflow_pytorch_keras"
```

## 5、脚本

### 5.1 kibana

```shell
#!/bin/bash

# ================= 配置区域 =================
# Kibana 的安装根目录
KIBANA_HOME="/opt/kibana"
# Kibana 启动脚本
START_SCRIPT="${KIBANA_HOME}/bin/kibana"
# 自定义日志输出路径（避免产生 nohup.out）
LOG_FILE="${KIBANA_HOME}/logs/kibana.log"
# ===========================================

# 确保日志目录存在
mkdir -p $(dirname "$LOG_FILE")

# 获取 Kibana 进程的 PID（精准匹配 node 进程且包含 Kibana 启动特征）
get_pid() {
    PID=$(ps -ef | grep "node" | grep "src/cli/dist" | grep -v grep | head -n 1 | awk '{print $2}')
    echo "$PID"
}

# 启动 Kibana
start() {
    PID=$(get_pid)
    if [ -n "$PID" ]; then
        echo "Kibana 已经在运行中，PID: $PID"
    else
        echo "正在启动 Kibana..."
        # 使用 nohup 后台启动，并将标准输出和错误输出重定向到日志文件
        nohup sh "$START_SCRIPT" > "$LOG_FILE" 2>&1 &
        sleep 2
        PID=$(get_pid)
        if [ -n "$PID" ]; then
            echo "Kibana 启动成功，PID: $PID"
            echo "日志文件位置: $LOG_FILE"
        else
            echo "Kibana 启动失败，请检查日志: $LOG_FILE"
        fi
    fi
}

# 停止 Kibana
stop() {
    PID=$(get_pid)
    if [ -z "$PID" ]; then
        echo "Kibana 当前未在运行。"
    else
        echo "正在停止 Kibana (PID: $PID)..."
        # 1. 发送优雅停止信号
        kill "$PID"
        
        # 2. 等待进程完全退出（最多等待 15 秒）
        COUNT=0
        while [ $COUNT -lt 15 ]; do
            if ! ps -p "$PID" > /dev/null 2>&1; then
                echo "Kibana 已成功停止。"
                return
            fi
            sleep 1
            COUNT=$((COUNT + 1))
        done
        
        # 3. 如果超时仍未停止，强制杀死进程
        echo "Kibana 未能优雅退出，正在强制终止..."
        kill -9 "$PID"
        echo "Kibana 已被强制停止。"
    fi
}

# 检测 Kibana 运行状态
status() {
    PID=$(get_pid)
    if [ -n "$PID" ]; then
        echo "Kibana 正在运行，PID: $PID"
    else
        echo "Kibana 未在运行。"
    fi
}

# 重启 Kibana
restart() {
    stop
    sleep 2
    start
}

# 脚本使用说明
usage() {
    echo "用法: $0 {start|stop|status|restart}"
    echo "  start   - 启动 Kibana"
    echo "  stop    - 停止 Kibana"
    echo "  status  - 查看 Kibana 运行状态"
    echo "  restart - 重启 Kibana"
}

# 主逻辑入口
case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status
        ;;
    restart)
        restart
        ;;
    *)
        usage
        ;;
esac

```

### 5.2 start-es

```shell
#!/bin/bash

# ================= 配置区域 =================
# 定义集群所有节点的 IP 或主机名
NODES=("10.0.0.67" "10.0.0.68" "10.0.0.69")

# 定义 ES 的安装目录（请确保所有节点路径一致）
ES_HOME="/opt/es"

# 定义运行 ES 的系统用户
ES_USER="es"

# 获取 ES 自带的 JDK 路径
ES_JDK_PATH="${ES_HOME}/jdk"
# ===========================================

echo "========================================="
echo "   开始启动 Elasticsearch 集群..."
echo "========================================="

for node in "${NODES[@]}"; do
    echo "[INFO] 正在启动节点: $node"
    
    # 通过 SSH 远程执行启动命令
    # 1. 使用 -f 参数让 ssh 在后台执行时不阻塞
    # 2. 使用 sudo -u 切换到 es 用户
    # 3. 临时注入 ES_JAVA_HOME 环境变量以消除 Warning
    ssh -f "$node" "sudo -u $ES_USER bash -c 'ES_JAVA_HOME=$ES_JDK_PATH $ES_HOME/bin/elasticsearch -d'"
    
    # 检查上一条命令是否执行成功
    if [ $? -eq 0 ]; then
        echo "[SUCCESS] $node 启动命令已下发"
    else
        echo "[ERROR] $node 启动失败，请检查 SSH 免密或权限配置"
    fi
done

echo "========================================="
echo "   所有节点启动命令已下发完毕！"
echo "   集群通常需要 10~30 秒完成选举，请稍后检查状态。"
echo "========================================="
```

### 5.3 status-es

```shell
#!/bin/bash

# ================= 配置区域 =================
NODES=("10.0.0.67" "10.0.0.68" "10.0.0.69")
ES_PORT="9200"
ES_USER="elastic"  # 你的ES用户名

# 密码获取逻辑（优先读取环境变量，其次交互式输入）
if [ -z "$ES_PASSWORD" ]; then
    read -sp "请输入 Elasticsearch 密码: " ES_PASSWORD
    echo ""
fi
# ===========================================

echo "========================================="
echo "   Elasticsearch 集群状态检查报告"
echo "========================================="

# 遍历节点，找到第一个能响应的节点进行查询
for node in "${NODES[@]}"; do
    # 检查节点是否存活（加入 -u 认证）
    if curl -s --connect-timeout 2 -u "${ES_USER}:${ES_PASSWORD}" "http://${node}:${ES_PORT}/" > /dev/null 2>&1; then
        echo "[INFO] 成功连接到节点: $node"
        echo ""
        
        # 1. 检查集群健康度
        echo "--- 集群健康状态 ---"
        curl -s -u "${ES_USER}:${ES_PASSWORD}" -X GET "http://${node}:${ES_PORT}/_cat/health?v"
        echo ""
        
        # 2. 检查节点列表及角色
        echo "--- 节点详细信息 ---"
        curl -s -u "${ES_USER}:${ES_PASSWORD}" -X GET "http://${node}:${ES_PORT}/_cat/nodes?v&h=name,ip,node.role,master,heap.percent,ram.percent,cpu,load_1m,disk.used_percent"
        echo ""
        
        # 3. 检查分片分配状态（排查是否有 unassigned 分片）
        echo "--- 未分配分片统计 ---"
        curl -s -u "${ES_USER}:${ES_PASSWORD}" -X GET "http://${node}:${ES_PORT}/_cat/shards?v&h=index,shard,prirep,state,node&s=state" | grep -E "UNASSIGNED|shard"
        echo ""
        
        exit 0
    fi
done

# 如果所有节点都无法连接
echo "[ERROR] 无法连接到集群中的任何节点！请检查网络、ES进程或密码是否正确。"
```

### 5.4 stop-es

```shell
#!/bin/bash

# ================= 配置区域 =================
NODES=("10.0.0.67" "10.0.0.68" "10.0.0.69")
ES_PORT="9200"
ES_USER="es"
ES_HOME="/opt/es"
# ===========================================

echo "========================================="
echo "   开始优雅停止 Elasticsearch 集群..."
echo "========================================="

# 1. 优先通过 API 进行优雅停机（向任意一个节点发送即可）
echo "[INFO] 正在发送优雅停机指令..."
curl -s -X POST "http://${NODES[0]}:${ES_PORT}/_shutdown" > /dev/null 2>&1
echo "[SUCCESS] 停机指令已下发，等待节点进程安全退出..."

# 2. 给 ES 留出处理数据刷盘和分片同步的时间
sleep 10

# 3. 遍历所有节点，强制清理可能残留的进程
for node in "${NODES[@]}"; do
    echo "[INFO] 正在检查并清理节点: $node"
    # 查找 es 用户的 elasticsearch 进程并 kill
    ssh "$node" "sudo -u $ES_USER bash -c 'pkill -f elasticsearch'" 2>/dev/null
    
    if [ $? -eq 0 ]; then
        echo "[SUCCESS] $node 进程已清理"
    else
        echo "[INFO] $node 没有发现运行中的 ES 进程"
    fi
done

echo "========================================="
echo "   集群停止操作完成！"
echo "========================================="
```

