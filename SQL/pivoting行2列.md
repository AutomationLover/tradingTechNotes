## 核心直觉：什么是 pivoting？

pivoting 就是：**把“长表（行）”翻成“宽表（列）”**，或者反过来。

- 原始：一行代表一个“维度 + 指标”的组合，比如  
  `year, metric, value`  
- Pivot 之后：一行代表一个维度（比如 year），不同 metric 变成多列：  
  `year, metric1_value, metric2_value, ...`

在数据分析里，它常用于：

- 把“类别”展开成多个指标列，方便报表/可视化  
- 或把“多列”压回一列做后续处理（unpivot，课程里主要讲前者）

---

## 1. 典型场景：按国家把指标拆成多列

想象有一张“长”的表：

year | metric      | value
---- | ----------- | -----
2015 | happiness   | 7.1
2015 | gdp         | 1.2
2015 | life_expect | 0.9
2016 | happiness   | 7.0
…    | …           | …

你想要的结果是：

year | happiness | gdp | life_expect
---- | --------- | --- | -----------
2015 | 7.1       | 1.2 | 0.9
2016 | 7.0       | …   | …

这就是一个 pivot：  
- `year` 作为行  
- `metric` 里的不同取值（happiness, gdp, life_expect）变成列  

---

## 2. 在没有 PIVOT 语法时的常用写法（MySQL 思路）

很多 OLTP 数据库（比如 MySQL）没有内建 `PIVOT` 关键字，课程里的思路是：

**用条件聚合模拟 pivot：`SUM(CASE WHEN ... THEN ... END)`**

示意：

SELECT
  year,
  AVG(CASE WHEN metric = 'happiness'   THEN value END) AS avg_happiness,
  AVG(CASE WHEN metric = 'gdp'         THEN value END) AS avg_gdp,
  AVG(CASE WHEN metric = 'life_expect' THEN value END) AS avg_life_expect
FROM metrics_long
GROUP BY year;

思路拆解：

- `CASE WHEN metric = 'happiness' THEN value END`：  
  对于 happiness 那些行给出 value，其它 metric 行给 NULL。  
- `AVG(...)`：对每年求平均，就只会对非 NULL 的 value 生效。  
- 最终得到：同一年里，各 metric 的值各在自己那一列。

课程里 pivot 的 SQL 结构基本都是这种 pattern 的变体。

---

## 3. 与“行列互转”的关系

你可以这样理解：

- **pivot**：行 → 列  
  - 把某个“类别字段”的不同值，展开成多个列。  
  - 常见：`SUM(CASE WHEN category=... THEN amount END) AS col_name`

- **unpivot**（本课没重点讲）：列 → 行  
  - 适合“多列结构不方便处理”，要压成 key-value 的长表。  
  - 一般通过 `UNION ALL` 或专用 UNPIVOT 语法实现。

在分析里，我们更多遇到的是 pivot：

- 做数据透视表（类似 Excel Pivot Table）  
- 把原本“多行多个指标”变成“一行多个指标列”

---

## 4. 典型应用场景

1）报表/仪表盘前的数据预处理

比如你要给 BI / 报表工具一个宽表：

country | year | avg_happiness | avg_gdp | avg_life_expect  
→ 用 pivot 把所有指标拼到一行，前端直接画图或导出。

2）对比不同类别的指标

例如：  
- 同一年里比较不同地区（region）在同一列里展示：  
  - `sales_region_na`, `sales_region_eu`, `sales_region_apac`  
- 某个维度的 TopN / Max 的指标拆列看。

3）把“事件类型”转成“特征列”

在风控/建模里，经常会把：

- `event_type`（登录、下单、退款）  
- 转成诸如 `cnt_login`, `cnt_order`, `cnt_refund` 这样每列一个特征。

实现上，本质还是条件聚合 + GROUP BY。

---

## 5. 小结：记住一个通用 SQL 模板

pivot 在 MySQL/课程场景下，本质就是这个模板：

SELECT
  分组字段...,
  聚合函数(CASE WHEN category_col = 'A' THEN value_col END) AS col_A,
  聚合函数(CASE WHEN category_col = 'B' THEN value_col END) AS col_B,
  ...
FROM 原表
GROUP BY 分组字段...;

- `category_col`：你要“转成列”的那个字段（比如 metric、region、event_type）  
- `value_col`：你要汇总的值（比如 amount、score、count）  
- 聚合函数：`SUM`, `COUNT`, `AVG`, `MAX`, `MIN` 等

把这个模式掌握住，你就能在任何没有 PIVOT 关键字的数据库里实现 pivoting。
