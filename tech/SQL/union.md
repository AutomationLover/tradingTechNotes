## 1. UNION 是什么？

UNION/UNION ALL 不是“横向拼列”（像 JOIN），而是**纵向“叠行”**：

- JOIN：把**不同表的列**放到一行里（横着拼）。
- UNION：把**多条查询的结果**一行接一行“堆叠起来”（竖着拼）。

语法骨架：

SELECT ...   -- 查询 A
FROM ...

UNION [ALL]

SELECT ...   -- 查询 B
FROM ...;

要求：

1. 上下两个 `SELECT` 的**列数必须相同**。
2. 每一列的**类型能兼容**（比如 int 对 int，varchar 对 varchar）。
3. 最终结果的列名来自**第一个 SELECT**。

---

## 2. UNION vs UNION ALL 的区别

视频里的关键点：

- UNION：**去重**  
  - 先把两边结果堆在一起，再做一次“distinct”，把**完全相同的行**合并成一条。  
  - 例子：tops 和 outerwear 里都有 `hoodie`，UNION 后只保留一行 hoodie。

- UNION ALL：**不去重**  
  - 只是简单“堆叠”，不检查重复。  
  - 例子：tops 和 outerwear 都有 hoodie，UNION ALL 后会有两行 hoodie。

性能：

- UNION 比 UNION ALL 慢，因为多了一步“全表去重（类似 DISTINCT）”。
- **如果你确定上下两边不会有重复，优先用 UNION ALL。**

---

## 3. 什么时候用 UNION？

典型场景都是：**结构相同，但数据来源不同**，你想把它们当成“一张更大的表”来分析。

### 场景 1：合并“同结构的多张表”

比如课程里的：

- `happiness_scores`：2015–2023 各国幸福分数。
- `happiness_scores_current`：2024 年的最新数据（字段略有不同：`ladder_score`）。

为了把 2015–2024 放在一起，写法类似：

-- 历史数据
SELECT
  year,
  country,
  happiness_score
FROM happiness_scores

UNION ALL

-- 当前年：强行加上 year=2024，并用 ladder_score 对齐
SELECT
  2024 AS year,
  country,
  ladder_score AS happiness_score
FROM happiness_scores_current;

结果：得到一张“2015–2024 全部年份”的统一视图。

注意点：

- 第二个 SELECT 里直接写常量 `2024`，在每行重复这个值。  
- 用别名让列名对齐（`ladder_score AS happiness_score`）。

### 场景 2：多表 / 多分区日志合并

在真实工作里很常见：

- 每个月一张分区表：`logs_202601`, `logs_202602`…  
- 多个 Region 拆成多表：`metrics_us`, `metrics_eu`, `metrics_apac`…

可以用 UNION / UNION ALL 把它们合起来当一张表来查：

SELECT * FROM logs_202601
UNION ALL
SELECT * FROM logs_202602
UNION ALL
SELECT * FROM logs_202603;

或：

SELECT 'us'   AS region, * FROM metrics_us
UNION ALL
SELECT 'eu'   AS region, * FROM metrics_eu
UNION ALL
SELECT 'apac' AS region, * FROM metrics_apac;

### 场景 3：构造“虚拟维度行”

有时你会从不同来源构造相同结构的数据，然后用 UNION 拼成一张“统一表”来做进一步聚合。

例如：错误告警来自不同系统，字段不完全相同，通过 SELECT 对齐后，用 UNION ALL 拼接，再在外层做 GROUP BY、统计。

---

## 4. 什么时候用 UNION，什么时候用 UNION ALL？

简单决策：

- **你关心“有没有重复”，要去重** → 用 UNION。  
  - 比如：想看“出现在哪些不同数据源中的唯一用户列表”。

- **你关心“真实行数/统计结果”，不想丢信息** → 用 UNION ALL。  
  - 比如：合并多个月日志，再统计总请求数、总错误数。  
  - 去重会导致计数变少，结果是错的。

再结合性能：

- 在大数据量、无重复或不需要去重的场景：  
  - **优先 UNION ALL**（视频里说，会快很多）。

---

## 5. 和 JOIN 的对比心智模型

- JOIN：  
  - **横向**：把“同一实体的不同属性”拼成一行。  
  - 前提：两表之间有可匹配的 key，比如 `country`。

- UNION：  
  - **纵向**：把“结构一样的结果集”堆在一起。  
  - 前提：列数和列类型对得上，不要求有 key 关系。

一句话记：

- 想“多表一起看成一张长表” → 用 UNION / UNION ALL。  
- 想“从多表拿字段拼到一行” → 用 JOIN。
