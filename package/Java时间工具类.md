# Java时间工具类

记录在日常开发中用到的关于时间转换的函数

## 1. 毫秒转时分秒

```java
public static String convertMilliSecondToHMS(Long milliSecond) {
    long seconds = TimeUnit.MILLISECONDS.toSeconds(milliSecond) % 60;
    long minutes = TimeUnit.MILLISECONDS.toMinutes(milliSecond) % 60;
    long hours = TimeUnit.MILLISECONDS.toHours(milliSecond);
    return hours + "小时" + minutes + "分钟" + seconds + "秒";
}
```

## 2. 分钟转时分

```java
public static String convertMinToHMS(Integer minute) {
    int hours = minute / 60;
    int minutes = minute % 60;
    return hours + "小时" + minutes + "分钟";
}
```

## 3. 获取往前或这往后n天的时间

```java
public static String getBeforeOrAfterDay(String format, int n) {
    SimpleDateFormat sdf = new SimpleDateFormat(format);
    Calendar current = Calendar.getInstance();
    current.add(Calendar.DAY_OF_MONTH, n);
    return sdf.format(current.getTime());
}
```

## 4. yyyy-MM-dd String转Date

```java
public static Date StringToDate(String format, String date) {
    SimpleDateFormat simpleDateFormat = new SimpleDateFormat(format);
    try {
        return simpleDateFormat.parse(date);
    } catch (ParseException e) {
        return null;
    }
}
```

## 5. yyyy-MM-dd Date转String

```java
public static String dateToString(String format, Date date) {
    SimpleDateFormat simpleDateFormat = new SimpleDateFormat(format);
    return simpleDateFormat.format(date);
}
```

