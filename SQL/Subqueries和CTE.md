Subqueries 和 CTE 都是“嵌套查询”的写法，本质能力相似，但在可读性、复用性和执行场景上各有优势。

一、基本概念

1) Subquery（子查询）

子查询就是把一个 SELECT 放在另一个查询里面使用。常见位置有：

a) SELECT 子句（标量子查询）

示例：给每一行加上一个“全表平均工资”的列

SELECT
  e.employee_id,
  e.salary,
  (SELECT AVG(salary) FROM employees) AS avg_salary
FROM employees e;

特点：子查询只返回一个值（1 行 1 列）。

b) FROM 子句（把子查询当成临时表）

SELECT
  t.department_id,
  t.total_salary
FROM (
  SELECT department_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY department_id
) t
WHERE t.total_salary > 100000;

特点：把中间结果组织清楚，再在外层继续筛选、聚合。

c) WHERE / HAVING 子句

IN 示例：

SELECT *
FROM employees
WHERE department_id IN (
  SELECT id FROM departments WHERE location = 'NY'
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

特点：更像“条件表达式”，可以写出“是否存在某些行”的逻辑。

2) CTE（Common Table Expression，公用表表达式）

基本语法：

WITH dept_salary AS (
  SELECT department_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY department_id
)
SELECT *
FROM dept_salary
WHERE total_salary > 100000;

可以理解为：在查询开始前先声明一个“只在本次查询有效的临时视图/中间表”。

多个 CTE 示例：

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

还可以写递归 CTE（常用于组织结构树、目录树等层级结构），这是子查询做不到的。

二、Subquery 与 CTE 的相同点

1) 都是用 SELECT 生成中间结果，再在外层查询中使用。
2) 大多数情况下，可以互相改写（FROM 子查询 ↔ CTE）。
3) 性能上，在很多数据库中差不多，取决于具体优化器实现。

三、Subquery 与 CTE 的不同点（关键）

1) 可读性

- Subquery：嵌套层数多时，很容易变成“括号地狱”，逻辑不直观。
- CTE：先在 WITH 部分分块定义逻辑，每一块起一个有意义的名字，可读性更好。

=> 当逻辑比较复杂、多步处理时，CTE 更适合作为“可读的分步骤管道”。

2) 复用性

- Subquery：同样的子查询如果需要用多次，只能复制粘贴，后期维护麻烦。
- CTE：同一个 CTE 可以在下面的查询中引用多次。

示例：多次复用 dept_salary：

WITH dept_salary AS (
  SELECT department_id, SUM(salary) AS total_salary
  FROM employees
  GROUP BY department_id
)
SELECT * FROM dept_salary WHERE total_salary > 100000
UNION ALL
SELECT * FROM dept_salary WHERE total_salary BETWEEN 50000 AND 100000;

3) 调试和重构

- Subquery：中间结果不方便单独跑，通常要从整条 SQL 中剪出来改。
- CTE：WITH 里的每块本质就是一个 SELECT，可以单独执行、调试，然后再拼回去。

4) 递归能力

- Subquery：只能写普通嵌套，没有递归语义。
- CTE：支持 Recursive CTE，可以实现层级结构、树形结构的向上/向下遍历。

5) 优化行为（视具体数据库）

- 有些数据库对 CTE 会物化，像临时表一样先算完再用，有时有利于复杂查询，有时会带来额外 I/O。
- 子查询一般由优化器自动重写、内联，执行计划会和 CTE 略有不同。
- 实战中要看 EXPLAIN / EXPLAIN ANALYZE 来判断具体表现。

四、和临时表 / 视图的比较

1) 临时表（Temporary Table）

- 显式建表：CREATE TEMPORARY TABLE ... AS SELECT ...
- 生命周期可以跨多条 SQL，用于复杂多步处理、加索引、反复读写。
- 更偏“物理层”，通常需要额外权限，也会占用存储和元数据。

2) 视图（View）

- CREATE VIEW 定义的“持久逻辑表”，供多个查询复用。
- 是数据库对象，适合抽象公共查询逻辑、对外提供统一接口。

3) CTE / Subquery

- 完全属于“单条 SQL 内部”的结构，不会在数据库里留下对象。
- 更适合数据分析、报表场景中快速表达复杂逻辑。

可记忆为：

- Subquery/CTE：一次性、单条 SQL 内的逻辑拆分。
- 临时表：多步处理 + 需要加索引 + 跨语句使用。
- 视图：长期复用 + 要共享给他人或上层应用使用的查询逻辑。

五、典型应用场景建议

1) 优先考虑子查询的场景

- 简单“集合/存在性”判断：
  - WHERE IN (SELECT ...)
  - WHERE EXISTS (SELECT 1 FROM ...)
- 简单标量子查询（给每行增加“全表汇总指标”的列）：
  - SELECT ..., (SELECT MAX(...)), (SELECT AVG(...)) ...

这类逻辑短小，用子查询比 CTE 更直接。

2) 优先考虑 CTE 的场景

- 报表 / 分析 SQL，逻辑较复杂，需要多步变换：
  - 第一步：过滤原始数据
  - 第二步：聚合或打标签
  - 第三步：再关联其他维度表
  - 第四步：做窗口函数 / 进一步聚合
- 需要在同一条 SQL 里多次复用相同的中间结果。
- 需要写递归逻辑（组织层级、上下级关系、路径展开等）。

3) 性能与工程实践上的取舍

- 逻辑简单：倾向用 Subquery，语句更短、更易上手。
- 逻辑中等或复杂：优先用 CTE 提升可读性，方便团队协作和 code review。
- 碰到性能瓶颈：结合执行计划分析优化器对 CTE / 子查询的处理：
  - 必要时改成临时表（可建索引）
  - 或物化视图 / 预计算表
  - 或改写连接方式 / 过滤顺序

六、一句话总结

- Subquery：适合简短、局部的“嵌套条件”或“补充一列”的场景。
- CTE：适合复杂、多步骤、需要复用或递归的场景，是“结构化 SQL 逻辑”的利器。
- 临时表 / 视图：从“一次性 SQL 内部结构”升级到“可跨语句 / 跨人员复用的数据库对象”。
