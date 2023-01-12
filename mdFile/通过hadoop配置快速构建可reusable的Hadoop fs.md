# 通过hadoop配置快速构建可reusable的Hadoop fs

## 1. 需求

在我们的业务系统中，需要经常对hadoop集群进行访问，所以就需要快速构建一个可reusable的`FileSystem`然后进行对应的`exists`,`create`,`delete`之类的操作。

## 2. 原理

设计程序，使其自动去`$HADOOP_HOME/etc/hadoop`目录下加载配置，生成`hadoopConf`，然后通过`FileSystem.get(hadoopConf)`方法获得`FilsSytem`。

读取`core-site.xml`,`hdfs-site.xml`,`yarn-site.xml`这3个xml文件

采用scala实现

## 3. 实现

- 获取`configuration`，并设置一些必要的属性。

```scala
def hadoopConf: Configuration = Option(reusableConf).getOrElse {
    //获取本机配置好的hadoop配置
    reusableConf = getConfigurationFromHadoopConfDir(hadoopConfDir)
    loadResource(hadoopConfDir)

    //如果没有这个属性,配置临时文件地址
    if (StringUtils.isBlank(reusableConf.get("hadoop.tmp.dir"))) {
        reusableConf.set("hadoop.tmp.dir", "/tmp")
    }
    //如果没有hbase属性,配置临时文件地址
    if (StringUtils.isBlank(reusableConf.get("hbase.fs.tmp.dir"))) {
        reusableConf.set("hbase.fs.tmp.dir", "/tmp")
    }

    reusableConf.set("yarn.timeline-service.enabled", "false")
    reusableConf.set("fs.hdfs.impl", classOf[DistributedFileSystem].getName)
    reusableConf.set("fs.file.impl", classOf[LocalFileSystem].getName)
    reusableConf.set("fs.hdfs.impl.disable.cache", "true")
    reusableConf
}
```

- 将3个配置文件中的属性依次加载到`configuration`

```scala
def getConfigurationFromHadoopConfDir(confDir: String = hadoopConfDir): Configuration = {
    if (!configurationCache.containsKey(confDir)) {
        val hadoopConfDir = new File(confDir)
        val confName = List("core-site.xml", "hdfs-site.xml", "yarn-site.xml")
        //将上面三个文件放入配置文件
        val files: List[File] = hadoopConfDir.
        listFiles().filter((x: File) => x.isFile && confName.contains(x.getName)).toList
        val conf = new Configuration()
        if (files.nonEmpty) {
            files.foreach((x: File) => conf.addResource(new Path(x.getAbsolutePath)))
        }
        configurationCache.put(confDir, conf)
    }
    configurationCache.get(confDir)
}
```

- 获取`hdfs`系统

```scala
def hdfs: FileSystem = {
    Option(reusableHdfs).getOrElse{
        reusableHdfs = FileSystem.get(hadoopConf)
    }
    reusableHdfs
}
```

## 4. 完整代码

`pom.xml`依赖使用hadoop-client.jar

```scala
import org.apache.commons.lang3.StringUtils
import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.fs.{FileSystem, LocalFileSystem, Path}
import org.apache.hadoop.hdfs.DistributedFileSystem

import java.io.File
import java.lang.reflect.{Field, Method}
import java.net.{URL, URLClassLoader}
import java.util.concurrent.ConcurrentHashMap

/**
 * Author: CHEN ZHI LING
 * Date: 2023/1/11
 * Description:
 */
object HadoopUtils {

  //hadoop环境变量的名称
  private[this] lazy val HADOOP_HOME: String = "HADOOP_HOME"

  //配置文件的路径
  private[this] lazy val CONF_SUFFIX: String = "/etc/hadoop"

  //hadoop配置
  private[this] var reusableConf: Configuration = _

  //reusable hadoop fs
  private[this] var reusableHdfs: FileSystem = _

  private[this] lazy val configurationCache = new ConcurrentHashMap[String, Configuration]()

  private[this] lazy val hadoopConfDir: String = getPathFromEnv(HADOOP_HOME) + CONF_SUFFIX


  /**
   * 获取本机配置的HDFS系统
   */
  def hdfs: FileSystem = {
    Option(reusableHdfs).getOrElse{
      reusableHdfs = FileSystem.get(hadoopConf)
    }
    reusableHdfs
  }

  /**
   * 程序自动去$HADOOP_HOME/etc/hadoop下加载配置
   */
  def hadoopConf: Configuration = Option(reusableConf).getOrElse {
    //获取本机配置好的hadoop配置
    reusableConf = getConfigurationFromHadoopConfDir(hadoopConfDir)
    loadResource(hadoopConfDir)

    //如果没有这个属性,配置临时文件地址
    if (StringUtils.isBlank(reusableConf.get("hadoop.tmp.dir"))) {
      reusableConf.set("hadoop.tmp.dir", "/tmp")
    }
    //如果没有hbase属性,配置临时文件地址
    if (StringUtils.isBlank(reusableConf.get("hbase.fs.tmp.dir"))) {
      reusableConf.set("hbase.fs.tmp.dir", "/tmp")
    }

    reusableConf.set("yarn.timeline-service.enabled", "false")
    reusableConf.set("fs.hdfs.impl", classOf[DistributedFileSystem].getName)
    reusableConf.set("fs.file.impl", classOf[LocalFileSystem].getName)
    reusableConf.set("fs.hdfs.impl.disable.cache", "true")
    reusableConf
  }

  /**
   * 从Hadoop配置目录中封装配置
   * @param confDir 配置文件夹
   * @return
   */
  def getConfigurationFromHadoopConfDir(confDir: String = hadoopConfDir): Configuration = {
    if (!configurationCache.containsKey(confDir)) {
      val hadoopConfDir = new File(confDir)
      val confName = List("core-site.xml", "hdfs-site.xml", "yarn-site.xml")
      //将上面三个文件放入配置文件
      val files: List[File] = hadoopConfDir.
        listFiles().filter((x: File) => x.isFile && confName.contains(x.getName)).toList
      val conf = new Configuration()
      if (files.nonEmpty) {
        files.foreach((x: File) => conf.addResource(new Path(x.getAbsolutePath)))
      }
      configurationCache.put(confDir, conf)
    }
    configurationCache.get(confDir)
  }

  /**
   * 根据环境变量获取地址
   * @param env 环境变量
   * @return
   */
  private[this] def getPathFromEnv(env: String): String = {
    val path: String = System.getenv(env)
    val file = new File(path)
    require(file.exists(), s" $env not existed!")
    file.getAbsolutePath
  }

  /**
   * 加载静态配置资源
   * @param filepath 文件路径
   */
  private[this] def loadResource(filepath: String): Unit = {
    val file = new File(filepath)
    addURL(file)
  }

  private[this] def addURL(file: File): Unit = {
    try {
      val classLoader: ClassLoader = ClassLoader.getSystemClassLoader
      classLoader match {
        case c if c.isInstanceOf[URLClassLoader] =>
          val addURL: Method = classOf[URLClassLoader].getDeclaredMethod("addURL", Array(classOf[URL]): _*)
          addURL.setAccessible(true)
          addURL.invoke(c, file.toURI.toURL)
        case _ =>
          val field: Field = classLoader.getClass.getDeclaredField("ucp")
          field.setAccessible(true)
          val ucp: AnyRef = field.get(classLoader)
          val addURL: Method = ucp.getClass.getDeclaredMethod("addURL", Array(classOf[URL]): _*)
          addURL.setAccessible(true)
          addURL.invoke(ucp, file.toURI.toURL)
      }
    } catch {
      case e: Exception => throw e
    }
  }
}
```

