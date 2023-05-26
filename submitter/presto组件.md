# Presto-submitter组件

## 1. 介绍

这个组件可以帮助你提交，查询，暂停presto任务到相对应的preosto集群，采用scala语言实现。

## 2. 版本 

- Presto 0.278

## 3. 功能

- 提交presto任务
- 获取单个presto任务状态
- 获取所有presto任务状态
- 获取presto节点信息
- 获取presto集群信息
-  kill单个正在运行的presto任务

## 4. 测试用例

### 4.1提交presto任务，并返回查询结果

- `SqlQueryRequest` 参数
  - `url`presto集群地址
  - `sql `sql语句
  - `user` 指定查询的用户，必须填写

- `SqlQueryResponse`返回
  - data
  - schema 

```java
@Test
void sqlQuery() {
    String url = "http://ip:8090";
    String sql = "select * from test";
    String user = "czl";
    SqlQueryRequest request = new SqlQueryRequest(url, sql, user);
    SqlQueryResponse submit = PrestoSubmitter.sqlQuery(request);
    System.out.println(submit.schema());
    System.out.println(submit.data());
}
```

### 4.2 获取单个presto任务状态

- `StatusQueryRequest ` 参数
  - url 
  - taskId

- `StatusQueryResponse`返回
  - `taskId`
  - `state`
  - `query` 查询的sql
  - `queryType`
  - `user`
  - elapsedTime持续时间

```java
@Test
void statusQueryById() {
    String url = "http://ip:8090";
    String taskId = "id";
    StatusQueryRequest statusQueryRequest = new StatusQueryRequest(url, taskId);
    StatusQueryResponse statusQueryResponse = PrestoSubmitter.statusQueryById(statusQueryRequest);
    System.out.println(statusQueryResponse.taskId());
    System.out.println(statusQueryResponse.state());
    System.out.println(statusQueryResponse.query());
    System.out.println(statusQueryResponse.queryType());
    System.out.println(statusQueryResponse.user());
    System.out.println(statusQueryResponse.elapsedTime());
}
```

### 4.3 获取所有presto任务状态

```jav
    @Test
    void statusQuery() {
        String url = "http://ip:8090";
        List<StatusQueryResponse> statusQueryResponses = PrestoSubmitter.statusQuery(url);
        System.out.println(statusQueryResponses.size());
    }
```

### 4.4 获取presto节点信息

- `NodeInfo`返回
  - `nodeVersion`
  - `coordinate` 是否是coordinate
  - `environment`
  - `uptime` 持续时间

```java
@Test
void nodeInfoQuery() {
    String url = "http://ip:8090";
    NodeInfo nodeInfo = PrestoSubmitter.nodeInfoQuery(url);
    System.out.println(nodeInfo.nodeVersion());
    System.out.println(nodeInfo.coordinate());
    System.out.println(nodeInfo.environment());
    System.out.println(nodeInfo.uptime());
}
```

### 4.5 获取presto集群信息

- `ClusterInfo`返回
  - `runningQueries`
  - `blockedQueries`
  - `queuedQueries` 排队查询个数
  - `activeWorkers`
  - `runningDrivers `当前运行driver个数
  - `runningTasks`
  - `reservedMemory` 当前使用保留内存
  - `totalInputRows `当前读入数据的总行数
  - `totalInputBytes` 当前读入数据的总大小
  - `totalCpuTimeSecs`
  - `adjustedQueueSize` 调整后的队列大小

```java
@Test
void clusterInfoQuery() {
    String url = "http://ip:8090";
    ClusterInfo clusterInfo = PrestoSubmitter.clusterInfoQuery(url);
    System.out.println(clusterInfo.runningQueries());
    System.out.println(clusterInfo.blockedQueries());
    System.out.println(clusterInfo.queuedQueries());
    System.out.println(clusterInfo.activeWorkers());
    System.out.println(clusterInfo.runningDrivers());
    System.out.println(clusterInfo.runningTasks());
    System.out.println(clusterInfo.reservedMemory());
    System.out.println(clusterInfo.totalInputRows());
    System.out.println(clusterInfo.totalCpuTimeSecs());
    System.out.println(clusterInfo.adjustedQueueSize());
}
```

### 4.6 kill单个正在运行的presto任务

```java
@Test
void kill() {
    String url = "http://ip:8090";
    String taskId = "";
    KillRequest killRequest = new KillRequest(url, taskId);
    KillResponse kill = PrestoSubmitter.kill(killRequest);
    System.out.println(kill.result());
}
```

## 5.仓库

源码可以参考[plugin-submiiter-api](https://github.com/Chenzhiling/plugin-submitter-api)

presto集群部署可以参考[博文](https://blog.csdn.net/czladamling/article/details/128205876?spm=1001.2014.3001.5501 )



