# “Analyse time series exchange network captures” 需要的 SQL 和 Python 技巧

这部分总结的是：为了分析“交易所网络抓包的时间序列”，在 **SQL** 和 **Python** 方面应该掌握哪些具体技能。

---

## 一、SQL 技巧

假设有一张表（或视图）存储从抓包/测量 ETL 出来的数据，例如：

- `ts`：时间戳（微秒/纳秒精度）
- `exchange` / `venue`：交易所/线路
- `flow_id` / `session_id`：连接或会话标识
- `latency_us`：某个延迟指标（tick-to-trade、order-to-ack、hop RTT 等）
- `msg_type`：消息类型（行情、订单、ACK 等）

### 1. 时间序列分桶与对齐

需要熟练：

- 按固定时间粒度做聚合：
  - PostgreSQL：
    - `date_trunc('second', ts)`
    - `date_trunc('minute', ts)`
  - MySQL：
    - `DATE_FORMAT(ts, '%Y-%m-%d %H:%i:00')`
- 按时间窗口过滤：
  - 最近 1 分钟、5 分钟、1 小时等：
    - `WHERE ts BETWEEN now() - interval '5 minutes' AND now()`

用途：生成“每秒/每分钟的汇总指标曲线”。

### 2. 聚合统计与分位数（尤其 P99）

基础聚合：

- `COUNT(*)`, `AVG(latency_us)`, `MIN`, `MAX` 按时间分桶、按 exchange 分组。

分位数（P50 / P90 / P99）：

- PostgreSQL / MySQL 8+：
  - `percentile_disc(0.99) WITHIN GROUP (ORDER BY latency_us)`
- Presto / Trino / Hive / ClickHouse 等：
  - `approx_percentile(latency_us, 0.99)`
- 无分位函数时：
  - 使用窗口函数：
    - `ROW_NUMBER() OVER (ORDER BY latency_us)`
    - `COUNT(*) OVER ()`
  - 取第 `ceil(0.99 * N)` 行近似 P99。

用途：画出“按时间/按 venue 的 P50/P90/P99 延迟”时间序列，用来对比正常 vs 异常。

### 3. 维度拆分与 drill-down

会按多种维度拆分延迟分布，找出问题范围：

- `GROUP BY exchange`：看哪个交易所的 tail 抬高；
- `GROUP BY exchange, instrument`：看具体产品线；
- `GROUP BY msg_type`：行情 vs 订单 vs ACK；
- `WHERE flow_id = 'X'`：查看单条链路/会话的统计。

结合时间分桶：

- 对特定时间窗口、特定 venue 做 P99 分析；
- 从“全局 P99 变坏” drill down 到“哪条线/哪种消息导致”。

### 4. 窗口函数：排序、相邻差值、top-N

熟悉窗口函数用法：

- `ROW_NUMBER()`, `RANK()`：
  - 挑出每分钟最慢的 N 条记录，用于异常样本分析。
- `LAG`, `LEAD`：
  - 计算相邻消息之间的时间差（inter-arrival time）：
    - 如：`ts - LAG(ts) OVER (PARTITION BY session_id ORDER BY ts)`
- `SUM() OVER`, `COUNT() OVER`：
  - 在窗口内做滑动统计。

用途：分析 microstructure 级别的行为（如 burst、gap）、消息间隔等。

### 5. 时间序列间的关联 / Join

将不同来源的数据在 SQL 层面关联：

- 抓包/测量表（网络延迟） vs 业务事件表（订单、成交）：
  - 按 `order_id` / `seq_num` / `flow_id` + 时间窗口 join；
- 对齐“tick→send→ack”三个时间点的数据，算出各段延迟；
- 结合系统 metrics 表（如 eBPF 输出写入的表），用 `ts` + `host` / `core` join 关联。

要求：懂得如何处理时间容差、重复匹配和主键设计。

---

## 二、Python 技巧（pandas + 可视化 + 简单统计）

SQL 负责粗粒度聚合和 dashboard，Python 更适合深入分析和实验。

### 1. pandas 时间序列 DataFrame 操作

- 数据读取：
  - `pandas.read_csv`, `read_parquet`, `read_sql` 等；
- 时间戳解析：
  - 转为 `datetime64[ns]`，设置为索引：`df = df.set_index('timestamp')`；
- 重采样（resample）：
  - `df.resample('1S').agg({'latency_us': ['mean', 'max']})`
  - `df.resample('1Min').quantile(0.99)` 等。

用途：快速生成各种时间粒度的统计曲线，用于探索性分析。

### 2. 分位数与分布分析

- 分位数：
  - `df['latency_us'].quantile(0.99)`；
  - `df.groupby('exchange')['latency_us'].quantile(0.99)`；
- 分布形状：
  - 直方图：`df['latency_us'].hist(bins=100)`；
  - 使用 seaborn `displot` / `histplot` 比较不同 venue 的分布；
- CDF / tail 分布（可选）：
  - 排序后画累计分布，直观看 tail 的粗细。

用途：看“今天 vs 昨天”、“venue A vs venue B”的延迟分布差异。

### 3. 时间对齐与多源数据同步

将不同系统的数据在 Python 里对齐成一条时间轴：

- 解析不同时间格式（包括 PTP/NIC 时间戳、应用日志时间）；
- 使用 `pandas.merge` / `merge_asof`：
  - `merge_asof` 对按时间排序的 DataFrame 按“最近时间”对齐，支持 tolerance；
- 例如：
  - 抓包中的 `T_send_hw`, `T_ack_hw`；
  - 应用日志中的 `T_app_send`, `T_app_recv`；
  - 将它们合并成一行，构造：
    - tick-to-trade；
    - order-to-exchange RTT；
    - kernel 内延迟等。

用途：重建“一个订单/一个 tick 在系统里的完整时间线”。

### 4. 可视化：matplotlib / seaborn / 交互图

常用图形：

- 时间序列曲线：
  - 每秒/每分钟 P50/P99 延迟；
  - 按 venue/产品线多条曲线对比；
- 箱线图 / violin plot：
  - 同一时间窗口下不同 venue 的延迟分布对比；
- 热力图：
  - CPU 核 vs 时间：软中断/IRQ 占用（结合 eBPF 指标）；
  - 队列 vs 时间：每队列 PPS / P99 poll 时间等。

用途：快速定位异常时间窗口和异常维度。

### 5. 简单统计与异常检测（加分项）

不需要深度机器学习，但需要一些基础统计直觉：

- 基线比较：
  - 当前 P99 vs 历史 P99 中位数；
  - 当前分布 vs 正常日分布（KS 检验等，可选）；
- 滑动窗口：
  - 用 `rolling('5min')` 做滑动窗口平均/分位数；
- 简单告警逻辑（原型）：
  - 当前窗口的 P99 > 历史均值 + k * 历史标准差；
  - 把这些时间区间标记为候选“事件窗口”，进一步 drill-down。

---

## 三、结合 JD 的技能总结

那句 “Analyse time series exchange network captures” 背后的技能可以总结为：

- **SQL：**
  - 时间分桶（秒/分钟）、按 venue/flow/group 聚合；
  - 计算 P50/P90/P99 等分位数；
  - 利用窗口函数做 top-N、相邻差值、滑动分析；
  - 用 join 把网络测量和业务事件对齐。

- **Python：**
  - pandas DataFrame 时间序列处理（resample、groupby、quantile）；
  - 分位数和分布分析（直方图、CDF）；
  - 多源数据按时间对齐（包括硬件时间戳 + 应用日志 + eBPF 输出）；
  - 用 matplotlib/seaborn 做延迟曲线、heatmap、分布对比；
  - 基本异常检测 / baseline 对比能力。

结合你现有的 Linux 网络 + eBPF 知识，这些 SQL/Python 技巧就是让你把“抓到的时间戳和事件”变成可以回答问题的图表和结论的关键工具。
