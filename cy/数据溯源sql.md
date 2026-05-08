## 每天有多少张流程卡被检验

```sql
SELECT 
    DATE(check_time) AS check_date,
    COUNT(DISTINCT flow_card_id) AS flow_card_count
FROM cy_screw_check
WHERE is_delete = 0
GROUP BY DATE(check_time)
ORDER BY check_date DESC;
```

## 平均每天有多少张不同的流程卡被检验

```sql
SELECT 
    AVG(daily_count) AS avg_flow_card_per_day
FROM (
    SELECT 
        DATE(check_time) AS check_date,
        COUNT(DISTINCT flow_card_id) AS daily_count
    FROM cy_screw_check
    WHERE is_delete = 0
    GROUP BY DATE(check_time)
) AS daily_stats;
```

## 成型总产能趋势

```sql
SELECT
    m.machine_id,
    m.total_quantity as 总产量,
    m.total_weight as 总重量,
    m.record_count as 报工次数,
    ROUND(m.total_quantity / m.record_count, 2) as 平均单次产量
FROM (
    SELECT
        machine_id,
        SUM(total_quantity) as total_quantity,
        SUM(total_weight) as total_weight,
        COUNT(*) as record_count
    FROM cy_screw_daily_report
    WHERE is_delete = 0 and report_type = 0
    GROUP BY machine_id
) m
ORDER BY total_quantity DESC;
```

## 搓牙总产能趋势

```sql
SELECT
    m.machine_id,
    m.total_quantity as 总产量,
    m.total_weight as 总重量,
    m.record_count as 报工次数,
    ROUND(m.total_quantity / m.record_count, 2) as 平均单次产量
FROM (
    SELECT
        machine_id,
        SUM(total_quantity) as total_quantity,
        SUM(total_weight) as total_weight,
        COUNT(*) as record_count
    FROM cy_screw_daily_report
    WHERE is_delete = 0 and report_type = 1
    GROUP BY machine_id
) m
ORDER BY total_quantity DESC;
```

## 成型，搓牙，钻尾，打头产能对比

```sql
SELECT
    machine_id,
    CASE report_type WHEN 0 THEN '成型' WHEN 1 THEN '搓牙' WHEN 2 THEN '铣尾' WHEN 3 THEN '钻尾' ELSE '其他' END as 工序类型,
    SUM(total_quantity) /1000 as 总产量,
    SUM(total_weight) as 总重量
FROM cy_screw_daily_report
WHERE is_delete = 0
GROUP BY machine_id, report_type
ORDER BY machine_id, report_type;
```

## 员工总绩效排行

```sql
SELECT
     user_name as 用户名,
    u.total_quantity as 总产量,
    u.total_weight as 总重量,
    u.work_days as 工作天数,
    ROUND(u.total_quantity / u.work_days, 2) as 日均产量,
    u.record_count as 报工次数
FROM (
    SELECT
        user_id,
        user_name,
        SUM(total_quantity) /1000 as total_quantity,
        SUM(total_weight) as total_weight,
        COUNT(DISTINCT report_date) as work_days,
        COUNT(*) as record_count
    FROM cy_screw_daily_report csdr left join sys_user su on su.id = csdr.user_id
    WHERE csdr.is_delete = 0
    GROUP BY user_id
) u
ORDER BY total_quantity DESC;
```

### 员工每日绩效明细

```sql
SELECT
    user_name 用户名,
    report_date,
    SUM(total_quantity)/1000 as 日产量,
    SUM(total_weight) as 日重量,
    GROUP_CONCAT(DISTINCT machine_id ORDER BY machine_id) as 操作机台,
    COUNT(*) as 报工次数
FROM cy_screw_daily_report csdr left join sys_user su on su.id = csdr.user_id
WHERE csdr.is_delete = 0 and report_date >= '2026-03-01' and report_date <= '2026-03-31'
GROUP BY user_id, report_date
ORDER BY report_date DESC, user_id;
```

## 员工工序效率分析

```sql
SELECT
    su.user_name 用户名,
    CASE report_type WHEN 0 THEN '成型' WHEN 1 THEN '搓牙' WHEN 2 THEN '铣尾' WHEN 3 THEN '钻尾' ELSE '其他' END as 工序类型,
    SUM(total_quantity) / 1000 as 工序总产量,
    COUNT(DISTINCT report_date) as 工序工作天数,
    ROUND(SUM(total_quantity)/1000 / COUNT(DISTINCT report_date), 2) as 工序日均产量
FROM cy_screw_daily_report csdr left join sys_user su on su.id = csdr.user_id
WHERE csdr.is_delete = 0 and report_date >= '2026-03-01' and report_date <= '2026-03-31'
GROUP BY user_id, report_type
ORDER BY user_id, report_type;
```

## 螺丝产品总产量排行

```sql
SELECT
    p.screw_product_name,
    p.screw_spec,
    p.screw_colour,
    p.total_quantity as 总产量,
    p.total_weight as 总重量,
    p.work_order_count as 涉及工单数,
    p.record_count as 报工次数
FROM (
    SELECT
        csp.screw_product_name,
        csp.screw_spec,
        csp.screw_colour,
        SUM(total_quantity) / 1000 as total_quantity,
        SUM(total_weight) as total_weight,
        COUNT(DISTINCT work_order_id) as work_order_count,
        COUNT(*) as record_count
    FROM cy_screw_daily_report csdr left join cy_screw_product csp on csp.id = csdr.screw_product_id
    WHERE csdr.is_delete = 0 and csdr.report_date >= '2026-03-01' and report_date <= '2026-03-31'
    GROUP BY csdr.screw_product_id
) p
ORDER BY work_order_count DESC;
```

## 每日总产量趋势

```sql
SELECT
    report_date,
    SUM(total_quantity) /1000 as 总产量,
    SUM(total_weight) as 总重量,
    COUNT(*) as 报工次数,
    COUNT(DISTINCT machine_id) as 活跃机台数,
    COUNT(DISTINCT user_id) as 活跃员工数,
    ROUND(AVG(total_quantity) /1000, 2) as 平均每单产量
FROM cy_screw_daily_report
WHERE is_delete = 0 and report_type = 0 and report_date >= '2026-03-01' and report_date <= '2026-03-31'
GROUP BY report_date
ORDER BY report_date DESC;
```

