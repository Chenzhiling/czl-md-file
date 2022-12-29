# 1. 获取已经存在的hudi表中的元数据信息

业务上有一个需求，需要把hadoop上的hudi表的元数据信息进行展示。

查询源码，发现可以通过构建`HoodieTableMetaClient`实现，参数只需要`hadoop conf`和hudi表路径

```java
public static HoodieTableMetaClient getMetaClient(Configuration conf, String basePath) {
    return HoodieTableMetaClient
        .builder()
        .setConf(conf)
        .setBasePath(basePath)
        .setLoadActiveTimelineOnLoad(true)
        .build();
}
```

## 2. 1 获得Schema

schema的获取可以new 一个`TableSchemaResolver`实现。

```java
public static Schema getArvoSchema(HoodieTableMetaClient client) {
    TableSchemaResolver schemaResolver = new TableSchemaResolver(client,true);
    try {
        return schemaResolver.getTableAvroSchemaWithoutMetadataFields();
    } catch (Exception e) {
        return null;
    }
}
```

## 2.2 获得分区字段

```java
public static List<String> getPartitionFields(HoodieTableMetaClient metaClient) {
    Option<String[]> partitionFields = metaClient.getTableConfig().getPartitionFields();
    String[] partitions = partitionFields.orElse(new String[0]);
    return Arrays.asList(partitions);
}
```

## 2.3 获得huidi表的AllCommitsTimeline

```java
public static HoodieTimeline getAllHoodieCommitsTimeline(HoodieTableMetaClient client) {
    return client.getActiveTimeline().getAllCommitsTimeline();
}
```

## 2.4 获得hudi表的WriteOperationType

```java
public static String getOperationType(HoodieTimeline allCommitsTimeline, Integer n) throws IOException {
    HoodieInstant instant = allCommitsTimeline.nthInstant(n).get();
    byte[] details = allCommitsTimeline.getInstantDetails(instant).get();
    try {
        HoodieCommitMetadata metadata = HoodieCommitMetadata.fromBytes(details, HoodieCommitMetadata.class);
        return metadata.getOperationType().value();
    } catch (IOException exception) {
        throw new IOException(exception.getMessage());
    }
}
```

