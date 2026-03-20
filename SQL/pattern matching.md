## 总体思路

在这节课里，pattern matching 主要分三层难度：

1. 用字符串函数（`SUBSTRING` + `INSTR`）做“位置型匹配/拆分”  
2. 用 `LIKE` + 通配符做“简单模糊匹配”  
3. 用正则（regex）做“复杂模式匹配/抽取”

---

## 1. 利用 SUBSTRING / INSTR 做“位置型模式匹配”

目标示例：从 `event_name` 中抽取“第一个单词”。

核心函数（MySQL）：

- `INSTR(str, sub)`：返回子串 `sub` 在 `str` 中第一次出现的位置（1-based）。  
- `SUBSTRING(str, pos, len)`：从 `str` 第 `pos` 个字符开始，取 `len` 个字符。

思路：

1. 用 `INSTR(event_name, ' ')` 找到第一个空格位置（即“第一个单词结束处 + 1”）。  
2. 减一得到“第一个单词最后一个字符位置”。  
3. 用 `SUBSTRING(event_name, 1, INSTR(event_name, ' ') - 1)` 截出第一个单词。

问题：  
- 如果 `event_name` 里根本没有空格，`INSTR` 返回 0，`SUBSTRING(..., 1, -1)` 会得到空值。

改进：加 `CASE` 做 if/else：

- 如果 `INSTR(event_name, ' ') = 0`：直接返回整个 `event_name`。  
- 否则：返回上述 `SUBSTRING(...)` 结果。

这种用法本质是在做“基于位置的 pattern matching”，适合：

- 取第一个/最后一个单词  
- 取某个分隔符前后的部分（如 email 的 username、URL 的 domain 等）

---

## 2. 用 LIKE 做“简单文本模式匹配”

`LIKE` 不是函数，是关键字，但也是最常用的 pattern matching 手段之一。

通配符：

- `%`：匹配 0 个或多个任意字符  
- `_`：匹配 1 个任意字符

常见模式：

1）包含某个词：

`WHERE event_description LIKE '%family%'`  
→ 文本中任意位置只要包含 “family”。

2）以某个词开头：

`WHERE event_description LIKE 'A %'`  
→ 以 “A 空格” 起始，后面跟任意内容。

3）更精细一点的长度限制：

- `LIKE 'AB_%'`：以 `AB_` 开头（`_` 是单字符），之后任意内容。  
- `LIKE '____@gmail.com'`：刚好 4 个字符 + `@gmail.com`。

适用场景：

- 业务关键词过滤（日志中有“error”、“timeout”）  
- 粗粒度的前缀/后缀匹配  
- 简单的格式感知（长度弱约束）

注意：

- `LIKE` 匹配的是**具体文本模式**，不理解“单词边界”“重复次数”等复杂概念。  
- 大数据量时，`LIKE '%xxx%'` 会比较慢，也常常走不到索引。

---

## 3. 用正则（regex）做“复杂模式匹配/抽取”

本节课用的是 MySQL 的正则函数（如 `REGEXP_SUBSTR`），核心思想是：

- 不只是查“有没有这个词”，而是查“有没有符合某种规则的词”，甚至把它提取出来。

例子 1：从 `event_description` 中抽取特定词（celebration/festival/holiday）

正则形态类似：

`REGEXP_SUBSTR(event_description, 'celebration|festival|holiday')`

含义：

- 在 event_description 里找出第一个出现的  
  - "celebration" 或 "festival" 或 "holiday"  
- 找到了就返回该词，否则返回 NULL。

例子 2：抽取带连字符的单词（如 “once-in-a-lifetime”）

大致正则逻辑：

- “若干字符 + '-' + 若干字符”，可以重复出现 `(- + 字符)` 这段。  
- 可以表达为类似：`[A-Za-z]+(-[A-Za-z]+)+`  
- 匹配结果就是一个完整的“带一个或多个连字符的单词”。

课程重点不是你记住具体 regex 语法，而是理解能力：

- 可以按“模式（pattern）”而不是具体字面去匹配/抽取文本。  
- 例如“所有带连字符的词”“所有包含数字+单位的片段”等。

优点：

- 表达力极强：可以写出非常复杂的文本规则。  
- 语言无关：SQL、Python、VSCode 都是类似语法。

缺点：

- 写法晦涩、难记，一般需要借助文档或在线 regex 工具 / ChatGPT。  
- **性能开销大**：在大表上跑 regex 很容易把 query 拖慢，一般用于小数据、预处理或离线分析。

---

## 4. 选择什么工具做 pattern matching？

可以用一个“难度阶梯”来选：

1. 能用简单字符串函数搞定（`SUBSTRING`, `INSTR`, `LEFT/RIGHT`）  
   - 如“取第一个空格前的部分”  
   - → 尽量用它们，语义明确、性能好。

2. 需要“模糊匹配某段文本”（含 / 前缀 / 后缀等）  
   - → 用 `LIKE` + `%` / `_`。  

3. 需要识别“结构化模式”（如带连字符的单词、特定数字+单位模式、邮箱格式等）  
   - → 用正则（regex），注意性能。

一句话总结：

- `SUBSTRING` + `INSTR`：位置 → 截取  
- `LIKE`：简单“模糊查字串”  
- Regex：复杂“按规则找模式”，威力大但要慎用在大数据集上。
