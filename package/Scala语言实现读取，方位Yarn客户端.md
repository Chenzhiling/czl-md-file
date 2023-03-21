 

# Scala语言实现读取，访问Yarn客户端

## 1. 概览

使用scala语言访问Yarn客户端，加载Yarn客户端需要yarn-site.xml文件。对Yarns上的程序进行操作，需要applicationId

## 2. pom依赖

```xml
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-yarn-client</artifactId>
    <version>3.2.2</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>3.2.2</version>
</dependency>
<dependency>
```

## 3. 方法

### 3.1 获取YarnConf

加载yarn-site.xml文件获取得到YarnConfiguration，再顺便处理下HA高可用

```scala
def getYarnConf(yarnFile: String): YarnConfiguration = {
    val configuration = new YarnConfiguration()
    val file = new File(yarnFile)
    if (file.exists()) {
        configuration.addResource(file.toURI.toURL)
    }
    haYarnConf(configuration)
    configuration
}
```

对高可用进行处理，主要处理yarn.resourcemanager.hostname这个属性。

```scala
  private[this] def haYarnConf(yarnConfiguration: YarnConfiguration): YarnConfiguration = {
    val iterator: util.Iterator[util.Map.Entry[String, String]] = yarnConfiguration.iterator()
    while (iterator.hasNext) {
      val entry: util.Map.Entry[String, String] = iterator.next()
      val key: String = entry.getKey
      val value: String = entry.getValue
      if (key.startsWith(YARN_RM_HOSTNAME)) {
        val rm: String = key.substring(YARN_RM_HOSTNAME.length)
        val addressKey: String = YARN_RM_ADDRESS + rm
        if (yarnConfiguration.get(addressKey) == null) {
          yarnConfiguration.set(addressKey, value + ":" + YarnConfiguration.DEFAULT_RM_PORT)
        }
      }
    }
    yarnConfiguration
  }
```

### 3.2 构建YarnClient

通过YarnConfiguration直接初始化YarnClient

```scala
def yarnClient(yarnFile: String): YarnClient = {
    if (reusableYarnClient == null || !reusableYarnClient.isInState(STATE.STARTED)) {
        reusableYarnClient = YarnClient.createYarnClient
        val yarnConf: YarnConfiguration = getYarnConf(yarnFile)
        reusableYarnClient.init(yarnConf)
        reusableYarnClient.start()
    }
    reusableYarnClient
}
```

### 3.3 查询YarnApplicationState

传入appliationId，获取yarn上某个程序的状态。

`YarnApplicationState`是一个枚举类，有以下状态

`new`表示程序刚被创建，RMApp的初始状态

`new_saving`表示RM在处理客户端提交作业的请求期间状态

`submitted`表示程序已被提交

`accepted`表示程序已被调度器接受

`running`表示程序正在运行

`finished`表示程序运行成功

`failed`表示程序运行失败

`killed`表示程序被用户终止

```scala
def getState(appId: String, yarnFile:String): YarnApplicationState = {
    yarnClient(yarnFile)
    val id: ApplicationId = ApplicationId.fromString(appId)
    try {
        val report: ApplicationReport = reusableYarnClient.getApplicationReport(id)
        report.getYarnApplicationState
    } catch {
        case _: Exception => null
    }
}
```

### 3.4 查询FinalApplicationStatus

`FinalApplicationStatus`也是一个枚举类，有5个值

`undefined`表示程序还未结束

`succeeded`

`failed`

`killed`

`ended`表示程序具有多个结束状态的子任务

```scala
def getFinalStatue(appId: String, yarnFile:String): FinalApplicationStatus = {
    yarnClient(yarnFile)
    val id: ApplicationId = ApplicationId.fromString(appId)
    try {
        val report: ApplicationReport = reusableYarnClient.getApplicationReport(id)
        report.getFinalApplicationStatus
    } catch {
        case _: Exception => null
    }
}
```

### 3.5 杀死YarnApplication

杀死一个yarn上的程序

```scala
def killYarn(appId: String, yarnFile:String): Boolean = {
    yarnClient(yarnFile)
    try {
        reusableYarnClient.killApplication(ApplicationId.fromString(appId))
        true
    } catch {
        case _: Exception => false
    }
}
```

### 3.6 查询TrackingUrl

yarn上运行程序的web地址

```scala
def getTrackingUrl(appId: String, yarnFile:String): String = {
    yarnClient(yarnFile)
    val id: ApplicationId = ApplicationId.fromString(appId)
    try {
        val report: ApplicationReport = reusableYarnClient.getApplicationReport(id)
        report.getTrackingUrl
    } catch {
        case _: Exception => null
    }
}
```

## 4. 代码

```scala
package com.czl.submitter.spark.service.utils

import org.apache.hadoop.service.Service.STATE
import org.apache.hadoop.yarn.api.records.{ApplicationId, ApplicationReport, FinalApplicationStatus, YarnApplicationState}
import org.apache.hadoop.yarn.client.api.YarnClient
import org.apache.hadoop.yarn.conf.YarnConfiguration

import java.io.File
import java.util

/**
 * Author: CHEN ZHI LING
 * Date: 2023/3/17
 * Description:
 */
object YarnUtils {


  var reusableYarnClient: YarnClient = _


  private[this] lazy val YARN_RM_HOSTNAME: String = "yarn.resourcemanager.hostname."


  private[this] lazy val YARN_RM_ADDRESS: String = "yarn.resourcemanager.address."


  def yarnClient(yarnFile: String): YarnClient = {
    if (reusableYarnClient == null || !reusableYarnClient.isInState(STATE.STARTED)) {
      reusableYarnClient = YarnClient.createYarnClient
      val yarnConf: YarnConfiguration = getYarnConf(yarnFile)
      reusableYarnClient.init(yarnConf)
      reusableYarnClient.start()
    }
    reusableYarnClient
  }


  def getState(appId: String, yarnFile:String): YarnApplicationState = {
    yarnClient(yarnFile)
    val id: ApplicationId = ApplicationId.fromString(appId)
    try {
      val report: ApplicationReport = reusableYarnClient.getApplicationReport(id)
      report.getYarnApplicationState
    } catch {
      case _: Exception => null
    }
  }


  def getFinalStatue(appId: String, yarnFile:String): FinalApplicationStatus = {
    yarnClient(yarnFile)
    val id: ApplicationId = ApplicationId.fromString(appId)
    try {
      val report: ApplicationReport = reusableYarnClient.getApplicationReport(id)
      report.getFinalApplicationStatus
    } catch {
      case _: Exception => null
    }
  }


  def getYarnConf(yarnFile: String): YarnConfiguration = {
    val configuration = new YarnConfiguration()
    val file = new File(yarnFile)
    if (file.exists()) {
      configuration.addResource(file.toURI.toURL)
    }
    haYarnConf(configuration)
    configuration
  }


  def killYarn(appId: String, yarnFile:String): Boolean = {
    yarnClient(yarnFile)
    try {
      reusableYarnClient.killApplication(ApplicationId.fromString(appId))
      true
    } catch {
      case _: Exception => false
    }
  }


  /**
   * support ha
   */
  private[this] def haYarnConf(yarnConfiguration: YarnConfiguration): YarnConfiguration = {
    val iterator: util.Iterator[util.Map.Entry[String, String]] = yarnConfiguration.iterator()
    while (iterator.hasNext) {
      val entry: util.Map.Entry[String, String] = iterator.next()
      val key: String = entry.getKey
      val value: String = entry.getValue
      if (key.startsWith(YARN_RM_HOSTNAME)) {
        val rm: String = key.substring(YARN_RM_HOSTNAME.length)
        val addressKey: String = YARN_RM_ADDRESS + rm
        if (yarnConfiguration.get(addressKey) == null) {
          yarnConfiguration.set(addressKey, value + ":" + YarnConfiguration.DEFAULT_RM_PORT)
        }
      }
    }
    yarnConfiguration
  }
}
```

