## SQL 里的 function 是什么？

在 SQL 里，**function 就是一段“可复用的小计算”**：  
- 接收 0 个或多个输入（参数）  
- 对这些输入做计算 / 转换  
- 返回 1 个输出值  

语法特征：  
- 关键字后面跟一对括号：`FUNC(...)`  
- 有的里面有参数，有的没有，比如：`COUNT(*)`、`ROUND(score, 1)`、`CURRENT_DATE()`。

例子：

- 计算行数：  
  `COUNT(*)`  
- 四舍五入：  
  `ROUND(happiness_score, 1)`  
- 全部转大写：  
  `UPPER(country)`  
- 当前日期：  
  `CURRENT_DATE()`（无参数）

---

## 几类常见 function

课程里把 SQL 函数分成 3 大类，你可以这样记：

1. 聚合函数（Aggregate Functions）  
   - 输入：多行  
   - 输出：一行（一个值）  
   - 常见：`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`  
   - 用法：
     - 不分组：对整张表算一个值  
       `SELECT AVG(happiness_score) FROM happiness_scores;`  
     - 配合 `GROUP BY`：对每个分组算一个值  
       `SELECT country, AVG(happiness_score) FROM happiness_scores GROUP BY country;`

2. 窗口函数（Window Functions）  
   - 输入：一组“窗口内”的多行  
   - 输出：**每行一个值**，但这个值可以“看到同组其他行”（不折叠行）  
   - 常见：`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `SUM() OVER(...)` 等  
   - 形式：`FUNC(...) OVER (PARTITION BY ... ORDER BY ...)`

3. 一般函数 / 标量函数（General / Scalar Functions）  
   - 输入：一行中的某个（或几个）字段  
   - 输出：一行中的一个新值  
   - 不改变行数，只改变“这一列怎么展示”。  
   - 又按数据类型细分：
     - 数值函数：`ROUND`, `CEIL`, `FLOOR`, `ABS`, `POWER` …  
     - 日期时间函数：`CURRENT_DATE`, `DATE_ADD`, `DATEDIFF`, `EXTRACT(YEAR FROM ...)` …  
     - 字符串函数：`UPPER`, `LOWER`, `TRIM`, `SUBSTRING`, `CONCAT` …  
     - NULL 相关函数：`COALESCE`, `IFNULL`, `NVL` 等  

例子（一般函数）：

- `UPPER(country)`：字符串 → 全大写  
- `ROUND(happiness_score, 1)`：数值 → 保留 1 位小数  

---

## 如何从语法上识别 function？

1. **有括号**：  
   - `COUNT(*)`、`ROUND(x, 2)`、`CURRENT_DATE()` → 函数  
2. **没有括号的是关键字，不是函数**：  
   - `DISTINCT`、`GROUP BY`、`ORDER BY` 等是关键字，不是函数。  

课程里举例：`DISTINCT` 不是函数，因为没有 `DISTINCT(...)` 这种形式，它只是 SELECT 里的修饰符。

---

## 小实践心智模型

写 SQL 时可以这样判断：

- “我要对**多行**一起算一个结果（总数、平均、最大）” → 用 **聚合函数**。  
- “我要保留每一行，但为每行算一个‘组内相关’的值（名次、前一行、累计）” → 用 **窗口函数**。  
- “我要对某个字段做类型/格式转换（四舍五入、大小写、截取字符串、日期运算）” → 用 **一般函数**。

你后面这节课的内容，其实就是在系统过第三类：按数据类型（numeric / datetime / string / NULL）把常用函数过一遍，方便你遇到分析需求时，能快速选对函数。






