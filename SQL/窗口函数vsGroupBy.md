## 什么是 Window Function（窗口函数）？

一句话：**窗口函数是在“不折叠行”的前提下，对“某一组行（窗口）”做计算的函数**。

可以类比成：`GROUP BY` + 聚合 的“升级版”，但保留了原始每一行。

---

## 和 GROUP BY 的核心区别

先对比你课上讲的那点：

- GROUP BY + 聚合函数（SUM/AVG/COUNT…）：
  - 先按某些字段分组（比如国家 country）
  - 再对每组做聚合计算
  - **每组只保留一行**（行被“折叠 / collapse”）

例如：

按国家算平均幸福分：

SELECT
  country,
  AVG(happiness_score) AS avg_score
FROM world_happiness
GROUP BY country;

结果：  
- 每个 country 一行  
- 原始的年份、城市等细节都没了（行粒度从“每年/每城市”折叠成“每国家一行”）

---

- Window Function（窗口函数）：
  - 同样也按某些字段“分组”，但叫做 window / partition
  - 对每组行内部做计算
  - **不折叠行，保留原始每一行**，只是多出一列计算结果

例如，用 ROW_NUMBER 给每个国家的行编号：

SELECT
  country,
  year,
  happiness_score,
  ROW_NUMBER() OVER (PARTITION BY country ORDER BY year) AS rn
FROM world_happiness;

结果：  
- 仍然是“每国家每年一行”，行粒度不变  
- 额外多出 rn 列，在每个国家内部从 1、2、3… 编号  

所以：

- 聚合函数：改变行数（折叠成每组一行）  
- 窗口函数：保持行数不变，只是“在组内看别的行”做计算

---

## Window Function 的语法结构

通用形式：

<窗口函数名> OVER (
  [PARTITION BY 分组字段列表]
  [ORDER BY 排序字段列表]
  [window frame 子句，可选]
)

例子：

ROW_NUMBER() OVER (PARTITION BY country ORDER BY year)

拆解：

- ROW_NUMBER()：窗口函数本体（对窗口里的每一行给一个序号）
- OVER (...)：告诉数据库“在哪个窗口上计算”
- PARTITION BY country：定义“窗口按什么字段分组”
  - 可以理解成“每个 country 一扇窗”
- ORDER BY year：定义窗口内的排序
  - 对每个国家内部，按年份从小到大排

---

## 常见窗口函数类型（课程后面会讲）

只提前给你个直觉：

1) 排序类

- ROW_NUMBER()：组内按排序给唯一行号 1,2,3…
- RANK()：有并列名次，会跳号（1,1,3…）
- DENSE_RANK()：有并列但不跳号（1,1,2…）

2) “取值”类

- FIRST_VALUE(col)：窗口内按排序后的第一行值
- LAST_VALUE(col)：最后一行值
- NTH_VALUE(col, n)：第 n 行的值

3) 相对位置类

- LAG(col, offset)：当前行“往前几行”的值
- LEAD(col, offset)：当前行“往后几行”的值  
  典型用途：计算环比、前后对比

4) 统计类

- SUM(col) OVER (...)  
- AVG(col) OVER (...)  
- COUNT(col) OVER (...)  
- 等等，用法类似聚合，但不折叠行；可以配合 frame 做“滚动窗口”（moving average 等）。

---

## 一个直观例子：按国家做行编号

需求：对每个国家，按年份排序，给行编号。

SQL：

SELECT
  country,
  year,
  happiness_score,
  ROW_NUMBER() OVER (
    PARTITION BY country
    ORDER BY year
  ) AS rn
FROM world_happiness;

解释：

- 对 country 一致的行，构成一个窗口（Afghanistan 一窗，Albania 一窗……）
- 在每个窗口内部，按 year 排序
- ROW_NUMBER() 在每个窗口内部从 1 开始计数

和 GROUP BY 不同的是：  
- 你没丢掉任何行，只是多了一个“组内行号列”。

---

## 小结：如何快速区分 Window Function vs GROUP BY

可以用 3 句话记：

1. 都是“按某字段分组/分窗”。
2. GROUP BY + 聚合：**行数变少**，每组一行，细节丢失。
3. Window Function：**行数不变**，只是“看组内其他行”算出一个新列。

课程里的关键句就是：

> Aggregate functions 把每个 group 的行折叠成一行，  
> Window functions 不折叠行，只在每个 window 内做计算并保留原始粒度。

后面你看到的 ROW_NUMBER / RANK / LEAD / LAG 都是在这个框架下的不同“计算方式”而已。

---

## 窗口函数里的常用“计算方式”概览

四个关键词：  
- ROW_NUMBER：组内“行号”  
- RANK / DENSE_RANK：组内“名次”（有并列时行为不同）  
- LEAD / LAG：看“下一行 / 上一行”的值（做对比、差分）

下面都假设有一个分区：`PARTITION BY country ORDER BY year` 或类似。

---

## 1. ROW_NUMBER：组内唯一行号

**作用**：在每个窗口内，按指定排序给每行一个唯一编号 1,2,3,…，**没有并列**。

典型写法：

ROW_NUMBER() OVER (
  PARTITION BY country
  ORDER BY happiness_score DESC
) AS rn

常见用途：

- 取“组内 Top 1”：每个国家最新一条记录、最高分记录等  

  外层再写：

SELECT *
FROM (
  SELECT
    country,
    year,
    happiness_score,
    ROW_NUMBER() OVER (
      PARTITION BY country
      ORDER BY happiness_score DESC
    ) AS rn
  FROM world_happiness
) t
WHERE rn = 1;

- 对时间序列做行号，后面用行号做 join 或过滤  
- 给每组数据打一个稳定顺序（比单纯 ORDER BY 可控）

特点：即使有两行 score 完全一样，也会强行一个 1，一个 2，没有“并列”。

---

## 2. RANK / DENSE_RANK：组内排名（处理并列）

**共同点**：  
- 都是“按排序字段做名次”的窗口函数。  
- 写法类似，只是并列时行为不同。

### 2.1 RANK：有并列且跳号

例子：

RANK() OVER (
  PARTITION BY country
  ORDER BY happiness_score DESC
) AS rnk

若某国 happiness_score 排序后是：  
- 9.0  
- 8.5  
- 8.5  
- 7.0  

则对应 RANK 为：  
- 1  
- 2  
- 2  
- 4  ← 跳过 3

**用途**：  
- 更贴近“比赛名次”的直觉（两个人并列第 2，下一名第 4）。  
- 做 Top N 排名时，如果要保留所有并列的，就 WHERE rnk <= N。

### 2.2 DENSE_RANK：有并列但不跳号

例子：

DENSE_RANK() OVER (
  PARTITION BY country
  ORDER BY happiness_score DESC
) AS drnk

同样分数 9.0, 8.5, 8.5, 7.0，对应 DENSE_RANK 为：  
- 1  
- 2  
- 2  
- 3  ← 不跳号

**用途**：  
- 需要“等级”而不是“绝对名次”时（比如幸福等级=1、2、3…）  
- 后续要用 rank 作为 bucket/分层时，更适合用 DENSE_RANK。

### RANK vs DENSE_RANK 总结

- RANK：并列会跳号（1,2,2,4）  
- DENSE_RANK：并列不跳号（1,2,2,3）  
- ROW_NUMBER：强行唯一（1,2,3,4）

---

## 3. LAG / LEAD：看前一行/后一行

这两个是典型的“相对位置”窗口函数，**对比当前行与前后行**特别有用。

通用形式：

LAG(col [, offset, default]) OVER (...)
LEAD(col [, offset, default]) OVER (...)

- col：要取的列，比如 score、price、salary  
- offset：往前/后第几行，默认 1  
- default：找不到时的默认值（比如第一行没前一行）

### 3.1 LAG：取“前一行”的值

例子：计算每国每年幸福分数与“上一年”的差值：

SELECT
  country,
  year,
  happiness_score,
  LAG(happiness_score) OVER (
    PARTITION BY country
    ORDER BY year
  ) AS prev_score,
  happiness_score
    - LAG(happiness_score) OVER (
        PARTITION BY country
        ORDER BY year
      ) AS diff_from_prev
FROM world_happiness;

计算方式：

- 对每个 country 按 year 排序。  
- 当前行能看到“上一年”的分数 prev_score。  
- diff_from_prev 就是当前 - 上一年的变化（同比/环比）。

用途：

- 同一实体（国家、用户、股票）不同时间点的差分、增长率。  
- 检测“突然变化”：abs(diff_from_prev) > 阈值。  
- 分析趋势：正负号看是上升还是下降。

### 3.2 LEAD：取“下一行”的值

例子：看“下一个时间点”的值：

SELECT
  country,
  year,
  happiness_score,
  LEAD(happiness_score) OVER (
    PARTITION BY country
    ORDER BY year
  ) AS next_score,
  LEAD(happiness_score) OVER (
    PARTITION BY country
    ORDER BY year
  ) - happiness_score AS diff_to_next
FROM world_happiness;

用途：

- 对未来值做对比（比如当前 vs 下一期指标）。  
- 检查从当前到下一期的变化方向和幅度。  
- 滚动窗口内做“未来 n 步”的对比（改 offset 即可）。

---

## 4. 典型“计算模式”小结

1. ROW_NUMBER 模式：  
   - 先按 `PARTITION BY` 定义组，再按 `ORDER BY` 排序。  
   - 给每行一个唯一行号。  
   - 用于：取组内 Top1/TopN、去重保留一条、为后续 join/过滤打 tag。

2. RANK / DENSE_RANK 模式：  
   - 同样按分组 + 排序。  
   - 做“组内名次”，支持并列。  
   - 用于：榜单、排名、TopN（包含并列）、分层评分。

3. LAG / LEAD 模式：  
   - 按“时间/序列字段”排序。  
   - 当前行读取上一行或下一行的值。  
   - 用于：环比、前后对比、变化量/增长率、异常跳变检测。

如果你愿意，我可以基于你课程里的 happiness_scores 表，写一组“统一 SQL 模板”：把 ROW_NUMBER / RANK / LEAD / LAG 放在一条查询里，方便你直接贴到 MySQL Workbench 里跑。

---

```sql
-- 统一示例：在同一条查询里同时使用 ROW_NUMBER / RANK / DENSE_RANK / LAG / LEAD
-- 表结构假设：
--   happiness_scores(country, year, happiness_score, gdp_per_capita, ...)

SELECT
  country,
  year,
  happiness_score,

  ------------------------------------------------------------------
  -- 1. 行号 / 排名类：在「每个国家(country)」内部，按「年份(year)」排序
  ------------------------------------------------------------------

  -- 每个国家内部按年份升序的唯一行号（1,2,3,...）
  ROW_NUMBER() OVER (
    PARTITION BY country
    ORDER BY year ASC
  ) AS rn_by_year,

  -- 每个国家内部按幸福分降序的“比赛名次”（有并列会跳号：1,2,2,4）
  RANK() OVER (
    PARTITION BY country
    ORDER BY happiness_score DESC
  ) AS rnk_by_score,

  -- 每个国家内部按幸福分降序的“密集名次”（有并列但不跳号：1,2,2,3）
  DENSE_RANK() OVER (
    PARTITION BY country
    ORDER BY happiness_score DESC
  ) AS dense_rnk_by_score,

  ------------------------------------------------------------------
  -- 2. LAG / LEAD：对比同一国家前一年 / 后一年的分数
  ------------------------------------------------------------------

  -- 同一国家「上一年」的幸福分（按 year 升序）
  LAG(happiness_score) OVER (
    PARTITION BY country
    ORDER BY year ASC
  ) AS prev_score,

  -- 与上一年的差值 = 当前 - 上一年（同比/环比）
  happiness_score
    - LAG(happiness_score) OVER (
        PARTITION BY country
        ORDER BY year ASC
      ) AS diff_from_prev,

  -- 同一国家「下一年」的幸福分（按 year 升序）
  LEAD(happiness_score) OVER (
    PARTITION BY country
    ORDER BY year ASC
  ) AS next_score,

  -- 与下一年的差值 = 下一年 - 当前
  LEAD(happiness_score) OVER (
    PARTITION BY country
    ORDER BY year ASC
  ) - happiness_score AS diff_to_next

FROM happiness_scores
-- 外层 ORDER BY 只影响最终展示顺序，不影响窗口里的计算顺序
ORDER BY
  country,
  year;
```

---

## NTILE 是什么？

NTILE(n) 是一个窗口函数，用来把每个分组里的行**平均分成 n 份（分位桶）**，并给每行标一个桶号：1, 2, …, n。

可以理解为：“对每个窗口做分组排序，然后切成 n 段”。

---

## 语法结构

NTILE(n) OVER (
  [PARTITION BY 分组列…]
  ORDER BY 排序列…
)

关键点：

- n：要分成多少个桶（整数，比如 4、10 等）
- PARTITION BY：可选，定义“对谁单独分桶”
  - 比如 `PARTITION BY country` → 每个国家各自分桶
- ORDER BY：必选，定义按什么顺序切桶
  - 比如 `ORDER BY happiness_score DESC` → 高分在前的桶号小

---

## 直观例子：按幸福分做四分位

假设表 happiness_scores(country, year, happiness_score)

每个国家内部，按幸福分从高到低分成 4 桶（大致四分位）：

SELECT
  country,
  year,
  happiness_score,
  NTILE(4) OVER (
    PARTITION BY country
    ORDER BY happiness_score DESC
  ) AS quartile
FROM happiness_scores
ORDER BY country, quartile, happiness_score DESC;

含义：

- quartile = 1：该国“最高分那一批”  
- quartile = 4：该国“最低分那一批”  
- 行数不一定能被 4 整除，NTILE 会尽量把**前面的桶多分几行**，保证差距最多 1 行

---

## NTILE 如何分配行？

示例：某个国家有 10 行数据，用 NTILE(4)

- 10 ÷ 4 = 2 余 2  
- 一共 10 行，要分成 4 桶，理论上平均 2.5 行/桶  
- 实际分配：  
  - 前 2 个桶：3 行（多拿 1 行）  
  - 后 2 个桶：2 行  
- 所以桶大小可能是 `[3,3,2,2]`，但排序后**前面的桶优先分配**

通用规则：

1. 先按 ORDER BY 排好序  
2. 计算每桶“理想大小”，然后前面的桶多拿一行（如果有余数）  
3. 依次给行编号：第 1 桶标 1，第 2 桶标 2…

---

## 常见应用场景

1) 做分位数/等级划分

- 四分位 / 十分位 / 百分位：
  - NTILE(4)、NTILE(10)、NTILE(100)
- KPI 分档：
  - quartile = 1 → “Top performers”  
  - quartile = 4 → “Bottom performers”

2) 打分层标签，便于聚合分析

例如：按全球幸福分从高到低分成 10 桶，看“不同桶的平均 GDP”：

SELECT
  NTILE(10) OVER (ORDER BY happiness_score DESC) AS decile,
  AVG(happiness_score) AS avg_score,
  AVG(gdp_per_capita) AS avg_gdp
FROM happiness_scores
GROUP BY decile
ORDER BY decile;

3) 做简单抽样 / 分流

- 比如想把一批客户分成 5 组，做 A/B/C/D/E 实验，可以用 NTILE(5) 给组号。

---

## 和 RANK / DENSE_RANK 的区别

- RANK / DENSE_RANK：  
  - 是“名次”，多少名取决于数据里不同值的数量。  
  - 并列时名次数量不可控。

- NTILE(n)：  
  - 是“切成 n 份的桶号”，n 是你指定的。  
  - 主要关注“相对分段”（前 25%、中间 50% 等），不是精确名次。

一句话记：

- 要“第几名”：用 RANK / DENSE_RANK  
- 要“属于哪一档”：用 NTILE(n)

