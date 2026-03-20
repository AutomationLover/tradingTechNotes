  ## 总体思路

在 SQL 里，日期时间类型（DATE / TIME / DATETIME / TIMESTAMP）最常见的用法只有几类：

- 取“当前时间”
- 从时间里“拆字段”（年、月、日、周几、小时…）
- 计算“时间差”
- 在原时间基础上“加减一段时间”
- 用时间做过滤 / 分组（比如按天、按月、按周的指标）

下面结合你课程里的 MySQL 风格，按“用途 + 小技巧”来讲。

---

## 1. 取当前时间：做“最近一次、距今多久”

常用函数（MySQL）：

- 当前日期：`CURRENT_DATE` 或 `CURRENT_DATE()`  
- 当前日期时间：`NOW()` 或 `CURRENT_TIMESTAMP()`

典型场景：

1）算“距今多少天”（Recency：最近活动、最近下单）

示意：

SELECT
  user_id,
  last_purchase_date,
  DATEDIFF(CURRENT_DATE, last_purchase_date) AS days_since_last_purchase
FROM users;

技巧：

- 不要直接 `CURRENT_DATE - last_purchase_date`，返回的是内部数值，不直观。  
  用 `DATEDIFF` 这类差值函数，明确单位（天）。

2）筛选“最近 7 天 / 最近 30 分钟的日志”

示意：

SELECT *
FROM events
WHERE event_time >= NOW() - INTERVAL 7 DAY;

SELECT *
FROM requests
WHERE request_time >= NOW() - INTERVAL 30 MINUTE;

技巧：

- 用 `NOW() - INTERVAL n UNIT` 做滑动时间窗过滤，是运维 / 数据分析非常常用的模式。

---

## 2. 拆字段：按年 / 月 / 日 / 周分析

常见函数（MySQL）：

- `YEAR(dt)`，`MONTH(dt)`，`DAY(dt)`  
- `HOUR(dt)`，`MINUTE(dt)`  
- `DAYOFWEEK(dt)`（返回 1–7，1=周日）

典型场景：

1）按年、月做聚合（趋势、季节性）

示意：

SELECT
  YEAR(order_time) AS y,
  MONTH(order_time) AS m,
  COUNT(*) AS order_cnt,
  SUM(amount) AS revenue
FROM orders
GROUP BY
  YEAR(order_time),
  MONTH(order_time)
ORDER BY y, m;

2）按“周几”看分布（流量、报警、CPU 峰值）

示意：

SELECT
  DAYOFWEEK(event_time) AS dow_num,
  COUNT(*) AS cnt
FROM events
GROUP BY DAYOFWEEK(event_time)
ORDER BY dow_num;

如果你想看到“Mon/Tue”这样的字符，可以用 CASE 或映射表：

CASE DAYOFWEEK(event_time)
  WHEN 1 THEN 'Sun'
  WHEN 2 THEN 'Mon'
  ...
END AS dow_name

技巧：

- 生产代码里，最好把 `YEAR(dt)`, `MONTH(dt)` 这些表达式只写一次，放在 SELECT 和 GROUP BY 中保持一致，方便读懂执行计划。
- 要注意，按表达式 GROUP BY 会让某些数据库无法用到索引，需要时可以在 ETL 中预先生成维度列（`event_date`, `event_month` 等）。

---

## 3. 计算时间差：间隔、停留时长、SLA

常用函数（MySQL）：

- `DATEDIFF(date2, date1)`：返回 date2 - date1 的**天数**  
- `TIMESTAMPDIFF(unit, dt1, dt2)`：按指定单位计算差值

典型场景：

1）计算“事件间的天数”

SELECT
  event_name,
  event_date,
  CURRENT_DATE AS today,
  DATEDIFF(CURRENT_DATE, event_date) AS days_since_event
FROM my_events;

2）更精细单位：秒、分钟、小时（例如 API 延迟、任务执行时间）

SELECT
  job_id,
  start_time,
  end_time,
  TIMESTAMPDIFF(SECOND, start_time, end_time) AS duration_sec,
  TIMESTAMPDIFF(MINUTE, start_time, end_time) AS duration_min
FROM jobs;

技巧：

- DATEDIFF 只适合“天”级；需要秒 / 分钟 / 小时就用 TIMESTAMPDIFF。  
- 做 SLO/SLA 时，常见 pattern：  
  `WHERE TIMESTAMPDIFF(MILLISECOND, req_start, req_end) > 500`  
  然后算超时率。

---

## 4. 加减时间：窗口对齐、过期判断、滚动计算

常用函数（MySQL）：

- `DATE_ADD(dt, INTERVAL n UNIT)`  
- `DATE_SUB(dt, INTERVAL n UNIT)`  
- 或简写：`dt + INTERVAL n UNIT`, `dt - INTERVAL n UNIT`

典型场景：

1）生成“过期时间”：如 token、缓存、会话

SELECT
  token_id,
  issued_at,
  DATE_ADD(issued_at, INTERVAL 1 HOUR) AS expires_at
FROM tokens;

2）配合 WHERE 做“最近 N 天的数据”

前面提过：

WHERE event_time >= NOW() - INTERVAL 7 DAY;

3）滚动窗口边界（和窗口函数配合）

比如做一个“过去 30 天滚动 PV”，有些数据库可以用 window frame；在 MySQL 不支持时，可以用子查询 + `event_date BETWEEN dt-30 AND dt` 这种方式。  
在 SQL 数据仓库（Snowflake/BigQuery/Redshift）里，会配合 window frame：

SUM(pv) OVER (
  PARTITION BY site
  ORDER BY event_date
  ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
) AS rolling_30d_pv

（这里更多是窗口函数话题，但本质是对时间做“相对偏移”。）

技巧：

- 不同 RDBMS 对 interval 语法差异比较大，上线前看官方文档，不要硬背。  
- 要注意时区问题（业务上统一用 UTC，比本地时间安全）。

---

## 5. 用日期时间做过滤 / 分区：常见 Query 模板

几个高频 Query 模板，可以直接记下来：

1）某天的数据

WHERE event_date = '2026-03-21'

2）某个月的数据（简单写法）

WHERE event_date >= '2026-03-01'
  AND event_date <  '2026-04-01'

比用 `MONTH(event_date)=3 AND YEAR(event_date)=2026` 更友好索引。

3）最近 N 天

WHERE event_time >= CURRENT_DATE - INTERVAL 7 DAY

4）按“日期”聚合，但原字段是 datetime

SELECT
  DATE(event_time) AS d,
  COUNT(*) AS cnt
FROM events
GROUP BY DATE(event_time)
ORDER BY d;

技巧：

- 过滤条件尽量写成“字段在一个范围内”，避免在 WHERE 里对字段包函数（那样索引可能失效）。  
- 可以在表设计里单独加一列 `event_date DATE`（从 datetime 拆出来），专门用于按天聚合/过滤。

---

## 6. 常见坑 & 实战小建议

1）时区

- `NOW()` / `CURRENT_TIMESTAMP` 受会话时区影响；  
- 若系统以 UTC 为准，建议显式统一：建表用 `TIMESTAMP` + 设置 session timezone，或者所有应用层使用 UTC。

2）字符串 vs 日期类型

- 看到 `'2026-03-21'` 这种，最好在 schema 里是 DATE/TIMESTAMP 类型，而不是 `VARCHAR`;  
- 一旦时间存成字符串，排序/比较都会出问题（尤其是非 ISO 格式）。

3）跨数据库迁移

- 日期函数名变化很大：  
  - MySQL：`DATE_ADD`, `DATEDIFF`, `DAYOFWEEK`  
  - Postgres：`now()`, `age()`, `extract(dow from ts)`，interval 写法也不一样  
- 迁移时优先抽象成“逻辑”，再查对应 RDBMS 的函数。

---

## 一句话总结

- **当前时间** → `CURRENT_DATE` / `NOW()`  
- **拆字段** → `YEAR/MONTH/DAY/HOUR/DAYOFWEEK` 做按时间维度的聚合和分析  
- **算差值** → `DATEDIFF` / `TIMESTAMPDIFF`（SLA、间隔、停留时长）  
- **加减时间** → `DATE_ADD` / `INTERVAL`（过期时间、滑动窗口）  
- **过滤** → 尽量用“范围 + 原始字段”，避免 `WHERE FUNC(col)=...` 破坏索引

这些 pattern 记熟，基本能覆盖绝大多数日期时间分析场景。
