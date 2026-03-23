## Subqueries 与 CTE：概念、异同与应用场景

https://www.udemy.com/course/sql-advanced-queries/learn/lecture/47029165#overview

本文用 Markdown 结构组织概念说明；所有 SQL 代码以纯文本形式嵌入，方便你直接复制。

---

## 一、基本概念

### 1. Subquery（子查询）

子查询就是把一个 SELECT 放在另一个查询里面使用，是“查询里的查询”。  
常见位置有：SELECT、FROM、WHERE/HAVING 等。

#### a) SELECT 子句（标量子查询）

用途：给每一行增加一个“全表级别”或“其他条件下”的汇总值。

示例：

SELECT
  e.employee_id,
  e.salary,
  (SELECT AVG(salary) FROM employees) AS avg_salary
FROM employees e;

要点：  
- 内层子查询返回 1 行 1 列（标量）。  
- 外层把这个值当作一个普通列使用。

#### b) FROM 子句（子查询当成临时表）

用途：把复杂逻辑先算成一个中间结果，再在外层继续筛选或聚合。

示例：

SELECT
  t.department_id,
  t.total_salary
FROM (
  SELECT department_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY department_id
) t
WHERE t.total_salary > 100000;

要点：  
- 内层子查询产出一张临时表 t。  
- 外层可以继续做 WHERE、JOIN、GROUP BY 等操作。

#### c) WHERE / HAVING 子句（条件子查询）

常见写法：IN、EXISTS、NOT EXISTS 等。

IN 示例：

SELECT *
FROM employees
WHERE department_id IN (
  SELECT id
  FROM departments
  WHERE location = 'NY'
);

EXISTS / 相关子查询示例：

SELECT d.id, d.name
FROM departments d
WHERE EXISTS (
  SELECT 1
  FROM employees e
  WHERE e.department_id = d.id
    AND e.salary > 100000
);

要点：  
- 强调“集合”或“存在性”逻辑（是否有满足条件的行）。  
- 相关子查询（correlated subquery）会引用外层表（如 d.id）。

---

### 2. CTE（Common Table Expression，公用表表达式）

语法：在查询开头用 WITH 定义一个或多个“中间结果”，在后续查询中像表一样使用。

基本语法：

WITH dept_salary AS (
  SELECT department_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY department_id
)
SELECT *
FROM dept_salary
WHERE total_salary > 100000;

理解方式：  
- 可以把 dept_salary 看成“只在这条 SQL 里有效的临时视图/中间表”。

#### 多个 CTE 示例

WITH dept_salary AS (
  SELECT department_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY department_id
),
high_salary_dept AS (
  SELECT department_id
  FROM dept_salary
  WHERE total_salary > 100000
)
SELECT *
FROM employees
WHERE department_id IN (SELECT department_id FROM high_salary_dept);

要点：  
- 按“步骤”拆分逻辑，每一步有清晰的名字。  
- 下方主查询可以多次引用这些 CTE。

#### 递归 CTE（Recursive CTE）

特点：  
- 支持自我引用，用于树/层级结构（组织架构、目录树等）。  
- 这是普通子查询不具备的能力。

---

## 二、Subquery 与 CTE 的相同点

1. 本质都用 SELECT 生成中间结果，再在外层查询中使用。  
2. 大多数非递归场景，子查询和 CTE 可以互相改写：  
   - FROM (SELECT ...) t ↔ WITH t AS (SELECT ...) SELECT ... FROM t  
3. 在很多数据库（如 MySQL、PostgreSQL）里，简单场景下性能差异通常不大，优化器会进行等价转换。

---

## 三、Subquery 与 CTE 的主要差异

### 1. 可读性

- Subquery：  
  - 简单场景语句短、直接。  
  - 多层嵌套很容易变成“括号地狱”，阅读困难。  

- CTE：  
  - 前面用 WITH 分块写每一步逻辑。  
  - 每一步起有意义的名字，可读性更好。

结论：  
- 逻辑复杂、多步处理时，用 CTE 更利于理解与维护。

---

### 2. 复用性

- Subquery：  
  - 同一段逻辑在一个查询里要用多次，只能复制粘贴。  
  - 维护成本高，修改容易漏。  

- CTE：  
  - 在 WITH 里定义一次，下面可以多次引用。  

示例（多次复用同一 CTE）：

WITH dept_salary AS (
  SELECT department_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY department_id
)
SELECT * FROM dept_salary WHERE total_salary > 100000
UNION ALL
SELECT * FROM dept_salary WHERE total_salary BETWEEN 50000 AND 100000;

---

### 3. 调试与重构

- Subquery：  
  - 想看中间结果，通常要把子查询剪出来单独跑，比较麻烦。  

- CTE：  
  - WITH 里的每一块本质都是普通 SELECT，可以单独执行。  
  - 调试时可先跑 CTE，再逐步加后续逻辑。

---

### 4. 递归能力

- Subquery：  
  - 只能做普通嵌套，没有递归语义。  

- CTE：  
  - 支持 Recursive CTE，可在一条 SQL 中实现树遍历、层级展开等。  

这是 CTE 在功能上的明显扩展点。

---

### 5. 优化行为（与具体数据库实现相关）

- 一些数据库会对 CTE 做“物化”（materialize）：  
  - 优点：复用 CTE 时可以避免重复计算。  
  - 缺点：数据量大时，可能增加 I/O 和内存占用。  

- 子查询通常由优化器重写/内联：  
  - 比如 IN (SELECT ...) 被转成半连接（semi-join）等执行计划。  

实战建议：  
- 写完复杂查询后，用 EXPLAIN / EXPLAIN ANALYZE 看执行计划。  
- 若发现 CTE/子查询成为瓶颈，可考虑改写为临时表、物化视图等。

---

## 四、与临时表 / 视图的比较

### 1. 临时表（Temporary Table）

特点：  
- 显式建表，例如：

CREATE TEMPORARY TABLE dept_salary_tmp AS
SELECT department_id, SUM(salary) AS total_salary
FROM employees
GROUP BY department_id;

- 生命周期可以跨多条 SQL。  
- 可建索引、更新数据，更偏“物理层”。  
- 适用于海量数据、多步计算、需要物理调优的场景。

### 2. 视图（View）

特点：  

CREATE VIEW dept_salary AS
SELECT department_id, SUM(salary) AS total_salary
FROM employees
GROUP BY department_id;

- 数据库级别对象，供多个查询、多位用户复用。  
- 好处：统一逻辑、统一权限控制、减少重复 SQL。

### 3. CTE / Subquery

特点：  
- 完全属于“单条 SQL 内部”：  
  - 不会在数据库中留下持久对象。  
  - 非常适合一次性的分析、报表、数据探索。

可以总结为：  
- Subquery/CTE：一次性、单条 SQL 内部的逻辑拆分。  
- 临时表：跨语句、多步处理、可建索引、偏物理优化。  
- 视图：长期复用、对外共享的逻辑抽象。

---

## 五、典型应用场景建议

### 1. 更适合用 Subquery 的场景

- 简单集合 / 存在性判断：  
  - WHERE IN (SELECT ...)  
  - WHERE EXISTS (SELECT 1 FROM ...)  

- 简单标量汇总，给每行增加一个“全局指标”列：

SELECT
  e.employee_id,
  e.salary,
  (SELECT AVG(salary) FROM employees) AS avg_salary
FROM employees e;

特点：  
- 语句短小，直接写子查询即可，不必上升到 CTE。

---

### 2. 更适合用 CTE 的场景

- 报表、分析类 SQL，逻辑有多步变换，比如：  
  1. 预过滤原始数据；  
  2. 聚合或打标签；  
  3. 再 JOIN 其他维度表；  
  4. 再做窗口函数或二次聚合；  

- 同一中间结果需要在后续多次使用（多处 JOIN、UNION、过滤）。  

- 需要递归逻辑（组织结构、目录树、上下级关系等）。

---

### 3. 性能与工程实践取舍

- 逻辑简单：  
  - 倾向用 Subquery，SQL 更短。  

- 逻辑复杂：  
  - 倾向用 CTE，结构更清晰，方便代码评审与维护。  

- 遇到性能问题：  
  - 看执行计划，检查 CTE 是否被重复扫描或物化成本过高；  
  - 必要时改成临时表、物化视图，或调整写法（JOIN 顺序、过滤顺序等）。

---

## 六、一句话总结

- Subquery：适合“局部嵌套”和“简单条件/标量补充”的场景，语句短小精悍。  
- CTE：适合“复杂多步骤逻辑”“需要复用中间结果”“需要递归”的场景，是组织大 SQL 的利器。  
- 临时表 / 视图：适合跨语句、跨团队复用和物理调优，是从“SQL 内部结构”升级到“数据库对象”的方案。
