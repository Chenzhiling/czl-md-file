# java实现统计一段时间内的数据，没有的话默认补零。

## 业务上的小小需求

**统计2023-07-01到2023-2023-07-07内的数据，希望得到下面的结果：**

| date       | total |
| ---------- | ----- |
| 2023-07-01 | 100   |
| 2023-07-02 | 200   |
| 2023-07-03 | 300   |
| 2023-07-04 | 400   |
| 2023-07-05 | 500   |
| 2023-07-06 | 600   |
| 2023-07-07 | 700   |

**但实际上可能某些日子，并没有产生数据，并存入数据库，所有查出来的结果可能是像下面这样的：**

| date       | total |
| ---------- | ----- |
| 2023-07-02 | 200   |
| 2023-07-03 | 300   |
| 2023-07-05 | 500   |

**所有这时候需要对缺失的数据，进行补零，使用`calendar`类进行实现。**

## 具体实现

`DateCount`类包含时间和数据两个字段

```java
public class DateCount {

    private String date;

    private Long total;

    public String getDate() {
        return date;
    }

    public Long getTotal() {
        return total;
    }

    public DateCount(String date, Long total) {
        this.date = date;
        this.total = total;
    }

    @Override
    public String toString() {
        return "DateCount{" +
                "date='" + date + '\'' +
                ", total=" + total +
                '}';
    }
}
```

`DateTimeUtils`类实现了方法

```java
public class DateTimeUtils {

    public static List<DateCount> supplementZero(String startTime, String endTime, List<DateCount> list) {
        //排序
        List<DateCount> sort = list
                .stream()
                .sorted(Comparator.comparing(DateCount::getDate)).collect(Collectors.toList());
        Map<String, Long> map = sort
                .stream()
                .collect(Collectors.toMap(DateCount::getDate, DateCount::getTotal, (k1, k2) -> k1));

        Timestamp start = stringToTimestamp(startTime);
        Timestamp end = stringToTimestamp(endTime);

        Calendar calendar = Calendar.getInstance();
        calendar.setTime(new Date(start.getTime()));
        calendar.set(Calendar.HOUR_OF_DAY, 0);
        calendar.set(Calendar.MINUTE, 0);
        calendar.set(Calendar.SECOND, 0);
        calendar.set(Calendar.MILLISECOND, 0);

        int i = 0;
        while (calendar.getTime().getTime() <= end.getTime()) {
            String date = dateToString(calendar.getTime(), "yyyy-MM-dd");
            if (null == map.get(date)) {
                DateCount dateCount = new DateCount(date, 0L);
                sort.add(i, dateCount);
            }
            calendar.add(Calendar.DAY_OF_MONTH, 1);
            i++;
        }
        return sort;
    }


    public static Timestamp stringToTimestamp(String time) {
        return Timestamp.valueOf(time);
    }

    public static String dateToString(Date date, String format) {
        SimpleDateFormat dateFormat = new SimpleDateFormat(format);
        return dateFormat.format(date);
    }
}
```

## 测试demo

```java
public class Demo {


    public static void main(String[] args) {
        DateCount data2 = new DateCount("2023-07-02", 200L);
        DateCount data3 = new DateCount("2023-07-03", 300L);
        DateCount data5 = new DateCount("2023-07-05", 500L);
        List<DateCount> dateCounts = new ArrayList<>();
        dateCounts.add(data3);
        dateCounts.add(data2);
        dateCounts.add(data5);
        String startTime = "2023-07-01 00:00:00.000";
        String endTime = "2023-07-07 00:00:00.000";
        List<DateCount> supplementZero = DateTimeUtils.supplementZero(startTime, endTime, dateCounts);
        for (DateCount count : supplementZero) {
            System.out.println(count.toString());
        }
    }
}
```

