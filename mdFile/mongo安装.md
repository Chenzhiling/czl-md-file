

# 前言

ubuntu18.04 安装mongodb，通过配置文件启动

# mongodb安装

- 安装curllib4

```shell
 sudo apt install curl
```

- 创建配置文件 mongodb.conf

```sh
systemLog:
  #MongoDB发送所有日志输出的目标指定为文件
  destination: file
  path: "/usr/local/mongo/log/mongodb.log"
  logAppend: true
storage:
  #mongod实例存储其数据的目录
  dbPath: "/usr/local/mongo/db"
  journal:
    #启用或禁用持久性日志以确保数据文件保持有效和可恢复。 
    enabled: true
processManagement: 
   #启用在后台运行mongos或mongod进程的守护进程模式。 
   fork: true
net:
   #服务实例绑定的IP，默认是localhost
   bindIp: 0.0.0.0
   port: 27017
```

- 环境变量

```shell
#mongo
export MONGO_HOME=/usr/local/mongo
export PATH=$PATH:$MONGO_HOME/bin
```

- 通过配置文件 启动mongo

```shell
mongod --config /usr/local/mongo/bin/mongodb.conf
```

- 关闭mongo

```shell
mongo --port 27017
use admin
db.shutdownServer()
```

