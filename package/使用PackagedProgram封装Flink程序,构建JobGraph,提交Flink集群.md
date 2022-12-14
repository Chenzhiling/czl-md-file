## 使用PackagedProgram封装Flink程序,构建JobGraph,提交Flink集群

## 1.PackagedProgram简介

- 官方介绍

This class encapsulates represents a program, packaged in a jar file. It supplies functionality to extract nested libraries, search for the program entry point, and extract a program plan.

- 个人理解

将写好的Flink应用程序封装起来,包括jar文件,mainClass(主函数入口),args(参数),savepoint这些

## 2.使用

最近在开发一个flink相关的组件,发现除了通过restful api的形式远程提交flink任务外,还可以通过构建PackagedProgram,然后创建JobGraph,可以将编写的Flink程序提交给local或者remote模式的flink集群运行.

## 3.PackagedProgram类创建

### 3.1 PackaghedProgram.newBuilder

可以通过它的Builder方法实现,需要设置一些必备的信息,比如mainClass,jarPath这些

```scala
val packagedProgram: PackagedProgram = PackagedProgram
    .newBuilder
    .setEntryPointClassName("你的flink程序文件主函数入口")
     //"你的flink程序文件"
    .setJarFile(jarFile)
     //"savepoint的信息"
    .setSavepointRestoreSettings(SavepointRestoreSettings.none)
    .setArguments("你的flink程序可能需要的参数","1","2","3")
    .build()
```

### 3.2 SavepointRestoreSettings

主要就是保存点的设置,可以直接调用类中的static方法创建

```scala
//不设置的情况
SavepointRestoreSettings.none
//通过path创建
SavepointRestoreSettings.forPath(path, allowNonRestoredState)
```

## 4.通过PackagedProgramUtils创建JobGraph

### 4.1 createJobGraph源码

这个类下有两个static方法方法可以直接创建JobGraph,区别是一个指定生成JobId,一个随机生成JobId

```java
    /**
     * Creates a {@link JobGraph} with a specified {@link JobID} from the given {@link
     * PackagedProgram}.
     *
     * @param packagedProgram to extract the JobGraph from
     * @param configuration to use for the optimizer and job graph generator
     * @param defaultParallelism for the JobGraph
     * @param jobID the pre-generated job id
     * @return JobGraph extracted from the PackagedProgram
     * @throws ProgramInvocationException if the JobGraph generation failed
     */
    public static JobGraph createJobGraph(
            PackagedProgram packagedProgram,
            Configuration configuration,
            int defaultParallelism,
            @Nullable JobID jobID,
            boolean suppressOutput)
            throws ProgramInvocationException {
        final Pipeline pipeline =
                getPipelineFromProgram(
                        packagedProgram, configuration, defaultParallelism, suppressOutput);
        final JobGraph jobGraph =
                FlinkPipelineTranslationUtil.getJobGraphUnderUserClassLoader(
                        packagedProgram.getUserCodeClassLoader(),
                        pipeline,
                        configuration,
                        defaultParallelism);
        if (jobID != null) {
            jobGraph.setJobID(jobID);
        }
        jobGraph.addJars(packagedProgram.getJobJarAndDependencies());
        jobGraph.setClasspaths(packagedProgram.getClasspaths());
        jobGraph.setSavepointRestoreSettings(packagedProgram.getSavepointSettings());

        return jobGraph;
    }
```

### 4.2 创建JobGraph

- PakcageProgram 
- configuration flink的conf
- parallelism  并行度,设置默认的也可以
- jobId
- suppressoutOutPut boolean类型,是否打印stdout/stderr在jobGraph创建阶段

```scala
val jobGraph: JobGraph = PackagedProgramUtils.createJobGraph(
    packagedProgram,
    flinkConfig,
    parallelism,
    null,
    false
)
```

- configuration也可以通过加载本地flink-conf.yaml获得,传入flink的安装路径即可.

```scala
  private def getFlinkDefaultConfiguration(flinkHome: String): Configuration = {
    Try(GlobalConfiguration.loadConfiguration(s"$flinkHome/conf")).getOrElse(new Configuration())
  }
```

## 5. 将JobGraph提交local模式的Flink

### 5.1 构建MiniCluster

MiniCluster的构建还是很简单,只要设置些必要参数就可以了,包括taskManager数量,slot数量之类的

```scala
//设置必要的属性,包括taskManager数量,slot之类
val miniClusterConfig: MiniClusterConfiguration =
      new MiniClusterConfiguration.Builder()
        .setConfiguration(flinkConfig)
        .setNumTaskManagers(numTaskManagers)
        .setNumSlotsPerTaskManager(numSlotsPerTaskManager)
        .build()
val cluster = new MiniCluster(miniClusterConfig)
cluster.start()
```

### 5.2 构建MiniClusterClient

有了cluster之后,构建client,配置ip,端口就可以交互了

```scala
val host: String = "localhost"
val port: Int = miniCluster.getRestAddress.get.getPort
flinkConfig
    .set(JobManagerOptions.ADDRESS,host)
    .set[JavaInt](JobManagerOptions.PORT,port)
    .set(RestOptions.ADDRESS, host)
    .set[JavaInt](RestOptions.PORT, port)
    .set(DeploymentOptions.TARGET, RemoteExecutor.NAME)
var client: ClusterClient[MiniClusterId] = new MiniClusterClient(flinkConfig,miniCluster)
```

### 5.3 提交任务

成功的话就会返回JobId

```scala
val jobId: String = client.submitJob(jobGraph).get().toString
```

## 6 结束

以上就完成了,后续尝试下提交到remote集群