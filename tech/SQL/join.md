## 四种基本 JOIN 类型总结（基于视频内容）

### 1. INNER JOIN（内连接）

- 含义：只保留**两张表都存在匹配值**的行。  
- 直观类比：Venn 图中间的交集部分。  
- 示例场景：  
  - `happiness_scores` 内的国家 + `country_stats` 内也有的国家。  
  - 输出里不会出现 NULL（在 join 键上）。

---

### 2. LEFT JOIN（左连接）

- 含义：保留**左表的所有行**，右表只在有匹配时补充，否则右表列为 NULL。  
- 直观类比：Venn 图中“左边整块（包括交集）”。  
- 示例场景（视频）：  
  - 左表：`happiness_scores`，右表：`country_stats`。  
  - 结果：所有在 `happiness_scores` 中出现的国家都在结果里；  
    - 若某国家在 `country_stats` 里不存在，则 `country_stats` 的列为 NULL。  
- 常用：非常常见的 join 类型，用来“以某张主表为准做补充信息”。

---

### 3. RIGHT JOIN（右连接）

- 含义：保留**右表的所有行**，左表只在有匹配时补充，否则左表列为 NULL。  
- 直观类比：Venn 图中“右边整块（包括交集）”。  
- 示例场景（视频）：  
  - 左表：`happiness_scores`，右表：`country_stats`。  
  - 结果：所有在 `country_stats` 中的国家都在结果里；  
    - 若某国家在 `happiness_scores` 里不存在，则 `happiness_scores` 的列为 NULL。  
- 实战中较少用：  
  - 通常可以**换表顺序 + 用 LEFT JOIN 代替**，逻辑等价，更容易读。

---

### 4. FULL OUTER JOIN（全外连接）

- 含义：返回**两表所有行**：  
  - 左表独有的  
  - 右表独有的  
  - 两边都有的  
- 直观类比：Venn 图的“左右两边整个并集”。  
- 结果行数 < 左表行数 + 右表行数，因为交集部分只算一次。  
- 支持情况（视频强调）：  
  - 不是所有数据库都支持：  
    - MySQL / SQLite 不支持 `FULL OUTER JOIN`。  
  - 可以用**INNER + LEFT** 等组合 + `UNION` 来模拟。

---

## 工程实践中的使用频率（课程观点）

- 最常用：  
  - `INNER JOIN`  
  - `LEFT JOIN`  
- 较少用：  
  - `RIGHT JOIN`（通常换表顺序写成 LEFT JOIN）  
- 有限制：  
  - `FULL OUTER JOIN` 在一些数据库（MySQL/SQLite）不可用，需要手动改写。

---

## 利用 JOIN 找“只在一边存在的值”（视频里的技巧）

用 LEFT / RIGHT JOIN + `WHERE 某一侧为 NULL`：

1. 找“左表有，但右表没有”的国家：

SELECT DISTINCT hs.country
FROM happiness_scores hs
LEFT JOIN country_stats cs
  ON hs.country = cs.country
WHERE cs.country IS NULL;

2. 找“右表有，但左表没有”的国家：

SELECT DISTINCT cs.country
FROM happiness_scores hs
RIGHT JOIN country_stats cs
  ON hs.country = cs.country
WHERE hs.country IS NULL;

这两个模式就是视频最后演示的“用 join + null 来找 only-in-left / only-in-right”。

---

## 一句话记忆

- INNER：只要“双方都有”的交集。  
- LEFT：以左表为主，右表来“补充信息”。  
- RIGHT：以右表为主（实战可用“调换表 + LEFT”替代）。  
- FULL OUTER：所有都要（交集 + 左独有 + 右独有），部分数据库需要自己拼。

---

## 默认通常是哪个 JOIN？

在绝大多数 SQL 数据库里：

- **不写类型、只写 `JOIN`，就等价于 `INNER JOIN`**。

也就是说：

SELECT ...
FROM a
JOIN b ON a.id = b.id;

和

SELECT ...
FROM a
INNER JOIN b ON a.id = b.id;

效果完全一样，都是内连接，只保留两表都匹配到的行。

其他类型（LEFT / RIGHT / FULL OUTER）都必须显式写出来，数据库不会默认成它们。

---

## cross join 是什么？

cross join 会返回 **两张表所有行的“笛卡尔积”**：  
左表每一行，都会和右表每一行配对一次。

如果：

- 表 A 有 m 行  
- 表 B 有 n 行  

那：

- `A CROSS JOIN B` 的结果就是 `m * n` 行。

---

## 基本语法和直观例子

示例表：

- tops：`t-shirt`, `hoodie`（2 行）  
- sizes：`small`, `medium`, `large`（3 行）

SQL：

SELECT
  t.item,
  s.size
FROM tops t
CROSS JOIN sizes s;

结果：

- t-shirt + small  
- t-shirt + medium  
- t-shirt + large  
- hoodie + small  
- hoodie + medium  
- hoodie + large  

也就是“每件上衣 × 每个尺码”的所有组合；2 × 3 = 6 行。

---

## 和其他 JOIN 的区别

- INNER / LEFT / RIGHT / FULL：
  - 都有一个「ON 条件」，控制**哪些行才能配对**。
  - 比如 `ON a.country = b.country`。

- CROSS JOIN：
  - **没有 ON 条件**（标准写法下）。
  - 不做匹配判断，直接“所有行两两配对”。

很多数据库里，写：

FROM a, b

在没有 WHERE 条件时，本质上也是一个 cross join。

---

## 典型应用场景

1) 生成“所有组合”

- 商品 × 尺码  
- 日期 × 货币种类  
- 维度 × 指标

2) 做“全对全”比较的基础

课程里提到的 candy 示例：

- 把 products 表和自己 cross join，得到所有糖果对（product1, product2）。  
- 再在 WHERE 里加条件：  
  - 价格差在 0.25 以内  
  - 不和自己配对  
  - 只保留 alphabetically 较小的在左边（去重）

示意：

SELECT
  p1.product_name,
  p2.product_name,
  ABS(p1.unit_price - p2.unit_price) AS price_diff
FROM products p1
CROSS JOIN products p2
WHERE ABS(p1.unit_price - p2.unit_price) <= 0.25
  AND p1.product_name < p2.product_name;

这里 cross join 的作用就是“先生成所有可能的配对，再用 WHERE 过滤”。

---

## 注意事项：行数爆炸

因为是笛卡尔积，行数是乘法：

- 2,000 行 × 3,000 行 → 6,000,000 行  
- 再加上列数、网络传输，很容易搞出性能 / 资源问题。

所以：

- 只在**小表**上用，或者  
- 先有限制（比如事先过滤、抽样），再 cross join。

---

## 一句话记忆

- cross join：**不看条件，所有行两两配对**。  
- 用于生成“所有组合”或做“全 pair 比较”的基础，注意行数爆炸风险。




