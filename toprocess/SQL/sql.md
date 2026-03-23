## 课程到目前为止的关键点总结（按主题）

https://www.udemy.com/course/sql-advanced-queries/learn/lecture/47028783#overview

### 1. SQL 基础：Big 6 子句顺序

一条典型查询由这 6 个核心子句组成，顺序固定：

SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY

- SELECT：要显示哪些列（或表达式、函数）。
- FROM：从哪张表 / 哪些表取数据。
- WHERE：在分组前过滤行（行级过滤）。
- GROUP BY：按某些字段分组，对每组做聚合。
- HAVING：在分组后过滤组（聚合结果过滤）。
- ORDER BY：最后对结果排序。

口诀（课程里的 mnemonic）：  
“Start Fridays With Grandma’s Homemade Oatmeal”  
对应：Select, From, Where, Group, Having, Order。

只要有 SELECT，就算一条合法 SQL；FROM 常见但不是强制（例如 `SELECT CURRENT_DATE;`）。

---

### 2. 多表分析：各种 JOIN

你已经覆盖了：

- INNER JOIN：只要两表都匹配的行（交集）。
- LEFT JOIN：保留左表所有行，右表匹配不上则为 NULL。
- RIGHT JOIN：保留右表所有行，左表匹配不上为 NULL（实战中多用“换表顺序 + LEFT”代替）。
- FULL OUTER JOIN：两表所有行（左独有 + 右独有 + 共有），MySQL/SQLite 不原生支持，可用 UNION 组合模拟。
- SELF JOIN：同一张表起两个别名，互相 JOIN，用于：
  - 找“同属性”的行对（同薪资员工）。
  - 做“大小比较”的成对行（工资高于谁）。
  - 解析自引用关系（员工–经理：`e1.manager_id = e2.employee_id`）。
- CROSS JOIN：笛卡尔积（所有行两两组合），用于生成“所有组合”或 pair；表大时小心爆炸。

---

### 3. UNION 与 UNION ALL：纵向“叠表”

- UNION：把多个 SELECT 结果“堆叠”起来，然后去掉完全重复的行。
- UNION ALL：只堆叠，不去重，性能更好。

使用前提：

- 每个 SELECT 列数相同。
- 对应列类型兼容。
- 最终列名来自第一个 SELECT。

常见用途：

- 合并历史表 + 当前表（你看到的 `happiness_scores` + `happiness_scores_current`）。
- 按月份/地区拆表后，合成一个大视图。

经验：

- 确定不会有重复、或重复有意义（统计计数系） → 用 UNION ALL。
- 需要“唯一集合” → 用 UNION。

---

### 4. 子查询 & CTE：复杂查询的拆分

你前面已经学过，这里结合回顾：

- Subquery：`SELECT` 里面再写 `SELECT`，常见位置：
  - SELECT：标量子查询（给每行加一个全局指标）。
  - FROM：子查询当临时表。
  - WHERE/HAVING：IN / EXISTS / ANY / ALL / 相关子查询。
- CTE（WITH）：在查询前面定义“只在本次查询内部使用的临时视图”。

对比要点：

- 能力上大多互相改写；
- CTE 可读性更好、可复用（同一个 CTE 可用多次）、支持递归；
- 子查询更适合局部简单逻辑（IN/EXISTS、简单标量补充列）。

---

### 5. 窗口函数 & 常见模式

窗口函数的核心：  
在不折叠行的前提下，对“分组内”做计算，形式：

FUNC(...) OVER (
  PARTITION
