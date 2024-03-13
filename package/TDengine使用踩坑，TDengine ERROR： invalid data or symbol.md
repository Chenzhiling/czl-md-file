# TDengine使用踩坑，TDengine ERROR： invalid data or symbol

## 环境

- Ubuntu18.04
- springboot 2.5.3
- mybatis-plus 3.4.2
- TDengine 3.1.0.3

## 错误情况

今天在后台项目中使用`mybatis-plus`框架插入数据到`TDengine`出现错误。

出现如下错误: `Java`的`String`类型无法被识别。

但是使用`Tdengine`终端可以正常插入

```sql
\n### Error updating database.  Cause: java.sql.SQLException: TDengine ERROR (0x80000216): syntax error near 'test)' (invalid data or symbol)\n### The error may exist in com/czl/chaoyi/console/backend/core/socket/dao/ScrewSocketMapper.java (best guess)\n### The error may involve com.czl.chaoyi.console.backend.core.socket.dao.ScrewSocketMapper.insertData-Inline\n### The error occurred while setting parameters\n### SQL: insert into ? values(now(),?,?,?,?,?,?,?,?,?)\n### Cause: java.sql.SQLException: TDengine ERROR (0x80000216): syntax error near 'test)' (invalid data or symbol)
```

数据的实体类

```java
public class ScrewSocket {

    @TableField(value = "ts")
    private Timestamp ts;

    @TableField(value = "machine_id")
    private Integer machineId;

    @TableField(value = "run_mode")
    private Integer runMode;

    @TableField(value = "run_status")
    private Integer runStatus;

    @TableField(value = "warning_status")
    private Integer warningStatus;

    @TableField(value = "speed")
    private Integer speed;

    @TableField(value = "interval_quantity")
    private Integer intervalQuantity;

    @TableField(value = "all_quantity")
    private Integer allQuantity;

    @TableField(value = "finished_quantity")
    private Integer finishedQuantity;

    @TableField(value = "work_order")
    private String workOrder;
}
```

插入函数

```java
    @Insert("insert into #{table} values(" +
            "now()," +
            "#{machineId}," +
            "#{runMode}," +
            "#{runStatus}," +
            "#{warningStatus}," +
            "#{speed}," +
            "#{intervalQuantity}," +
            "#{allQuantity}," +
            "#{finishedQuantity}," +
            "#{workOrder})")
    Boolean insertData(@Param("table") String machineTable,
                       @Param("machineId") Integer machineId,
                       @Param("runMode") Integer runMode,
                       @Param("runStatus") Integer runStatus,
                       @Param("warningStatus") Integer warningStatus,
                       @Param("speed") Integer speed,
                       @Param("intervalQuantity") Integer intervalQuantity,
                       @Param("allQuantity") Integer allQuantity,
                       @Param("finishedQuantity") Integer finishedQuantity, 
                       @Param("workOrder") String workOrder);
```

## 解决

在后台项目中使用`mybatis-plus`框架，插入`Java `的`String`类型到`TDengine`数据库时，数据需要额外加上`""`才能正常插入。

```java
    private String preprocessWorkOrder(String workOrder) {
        return ObjectUtils.isEmpty(workOrder) ? null : "\"" + workOrder + "\"";
    }
```

