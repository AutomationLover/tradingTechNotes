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





