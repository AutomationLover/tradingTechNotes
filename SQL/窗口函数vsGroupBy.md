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
