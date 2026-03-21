# SQL Knowledge & Skills Summary

https://www.udemy.com/course/sql-essentials-thinking-in-sql/learn/lecture/46176105#overview

> Comprehensive summary derived from notebooks in `notebooks/Completed Chapters/` and related source data in `sql_data/`.

---

## 1. Database & Table Operations

### Database Operations
- **Create**: `CREATE DATABASE IF NOT EXISTS db_name`
- **Show**: `SHOW DATABASES`
- **Describe**: `DESCRIBE DATABASE EXTENDED db_name`
- **Drop**: `DROP DATABASE IF EXISTS db_name CASCADE`

### Table Operations
- **Create**: `CREATE TABLE IF NOT EXISTS db.table_name(column_name TYPE, ...)`
- **Show tables**: `SHOW TABLES IN db_name`
- **Describe**: `DESCRIBE TABLE db.table_name`
- **Alter**:
  - Rename table: `ALTER TABLE ... RENAME TO ...`
  - Add column: `ALTER TABLE ... ADD COLUMNS (col_name TYPE)`
  - Drop column: `ALTER TABLE ... DROP COLUMNS (col1, col2)`
  - Rename column: `ALTER TABLE ... RENAME COLUMN old TO new`
- **Drop**: `DROP TABLE IF EXISTS db.table_name`

---

## 2. Data Manipulation (DML)

### INSERT
- **Single row**: `INSERT INTO table (col1, col2) VALUES (val1, val2)`
- **Multiple rows**: `INSERT INTO table VALUES (val1, val2), (val1, val2), ...`
- **From SELECT**: `INSERT INTO target SELECT col1, col2 FROM source WHERE condition`

### UPDATE
```sql
UPDATE table_name 
SET col1 = value1, col2 = value2
WHERE condition
```

### DELETE
```sql
DELETE FROM table_name WHERE condition
```

---

## 3. Basic Query Structure

### Core SELECT Syntax
```sql
SELECT expression
FROM source
[WHERE condition]
[ORDER BY column [ASC|DESC]]
[LIMIT n]
[OFFSET n]
```

### Key Concepts
- **Column selection**: `SELECT *`, `SELECT col1, col2`, reorder columns
- **Aliases**: `SELECT col AS alias_name`, `SELECT col AS \`Column Name\``
- **DISTINCT**: `SELECT DISTINCT col1, col2 FROM table`
- **Filtering**: `WHERE` with comparison and logical operators
- **Sorting**: `ORDER BY price DESC`, `ORDER BY col ASC NULLS LAST`
- **Pagination**: `LIMIT 5 OFFSET 2` (e.g., 3rd–5th rows)

---

## 4. Mathematical Expressions & CTEs

### Math Operators
| Operator | Symbol |
|----------|--------|
| Addition | `+` |
| Subtraction | `-` |
| Multiplication | `*` |
| Division | `/` |
| Modulo | `%` |

### Common Table Expression (CTE)
```sql
WITH cte_name AS (
    SELECT col1, col2 FROM table
)
SELECT * FROM cte_name
```

- Use CTEs to break complex queries into readable steps
- Computed columns can be referenced in subsequent CTEs

---

## 5. Comparison & Logical Operators

### Comparison
| Operator | Description |
|----------|-------------|
| `=`, `==` | Equality |
| `!=` | Not equal |
| `>`, `<`, `>=`, `<=` | Comparisons |
| `LIKE` | Pattern match (`%` any chars, `_` single char) |
| `BETWEEN low AND high` | Range |
| `IN (val1, val2, ...)` | List membership |
| `IS NULL` / `IS NOT NULL` | Null checks |

### Logical
| Operator | Description |
|----------|-------------|
| `AND`, `&` | Both true |
| `OR`, `|` | Either true |
| `NOT`, `!` | Negation |

---

## 6. CASE Expressions

### Syntax
```sql
CASE WHEN condition1 THEN result1
     [WHEN condition2 THEN result2]
     [ELSE default]
END
```

- Order of `WHEN` clauses matters (first match wins)
- Can be used in `SELECT` or `WHERE`
- Useful for categorization, conditional logic, and maintenance type labels

---

## 7. Advanced Filtering

### Subquery Filters (Single Value)
```sql
SELECT * FROM bookings
WHERE member_id = (
  SELECT member_id FROM members 
  WHERE first_name = "Tracy" AND last_name = "Smith"
)
```

### Subquery Filters (List of Values)
```sql
WHERE facility_id IN (SELECT facility_id FROM facilities WHERE ...)
WHERE member_id NOT IN (SELECT member_id FROM bookings)
```

### Correlated Query (EXISTS)
```sql
SELECT * FROM facilities f
WHERE EXISTS (
  SELECT true FROM bookings b
  WHERE b.facility_id = f.facility_id 
  AND b.slots * f.member_cost > 50
)
```

---

## 8. Joins

### Inner Join
- Returns rows where join condition matches in both tables
```sql
SELECT * FROM t1 
[INNER] JOIN t2 ON t1.id = t2.id
```

### Outer Joins
| Type | Behavior |
|------|----------|
| **LEFT** | All left rows + matching right |
| **RIGHT** | All right rows + matching left |
| **FULL** | All rows from both tables |

```sql
FROM t1 LEFT JOIN t2 ON t1.id = t2.id
FROM t1 RIGHT JOIN t2 ON t1.id = t2.id
FROM t1 FULL JOIN t2 ON t1.id = t2.id
```

### Other Joins
| Type | Description |
|------|-------------|
| **NATURAL JOIN** | Joins on columns with same name |
| **CROSS JOIN** | Cartesian product (all combinations) |
| **Self Join** | Join table to itself (e.g., recommender hierarchy) |
| **SEMI JOIN** | Rows in left that have match in right (like EXISTS) |
| **ANTI JOIN** | Rows in left with no match in right (like NOT EXISTS) |

### String Concatenation
- `||` operator or `concat_ws(separator, col1, col2)`

---

## 9. SQL Functions

### Mathematical
| Function | Example |
|----------|---------|
| `round(expr, d)` | `round(5289.87, 0)` |
| `abs(expr)` | `abs(-274.98)` |
| `ceil(expr)` | `ceil(274.98, 0)` |
| `floor(expr)` | `floor(274.98, 0)` |
| `pow(expr1, expr2)` | `pow(2, 5)` |
| `least(...)`, `greatest(...)` | `greatest(10, 12, 15)` |

### String
| Function | Example |
|----------|---------|
| `concat(col1, col2)` | Concatenate |
| `concat_ws(sep, col1, col2)` | Concatenate with separator |
| `substring_index(str, delim, n)` | Extract nth part |
| `substr(str, start, len)` | Substring |
| `left(str, len)`, `right(str, len)` | Extract from sides |
| `length(str)` | String length |
| `trim()`, `ltrim()`, `rtrim()` | Remove whitespace |
| `replace(str, old, new)` | Replace substring |
| `initcap()`, `ucase()`, `lcase()` | Case conversion |
| `contains(str, substr)`, `instr(str, substr)` | Search |
| `rlike(str, pattern)` | Regex match |

### Date/Time
| Function | Example |
|----------|---------|
| `current_date()`, `current_timestamp()` | Current date/time |
| `to_date(str, fmt)` | Parse date string |
| `to_timestamp(str, fmt)` | Parse timestamp |
| `date_format(timestamp, fmt)` | Format output |
| `day()`, `month()`, `year()`, `quarter()` | Extract parts |
| `hour()`, `minute()`, `second()` | Time parts |
| `add_months(date, n)` | Add months |
| `date_add()`, `date_sub()` | Add/subtract days |
| `date_diff(end, start)` | Days between dates |
| `months_between(ts1, ts2)` | Months between |
| `to_unix_timestamp()`, `from_unixtime()` | Unix timestamp |

### Conditional
- `nvl(expr, default)` - Replace NULL
- `if(condition, true_val, false_val)` - Conditional value

---

## 10. Aggregation

### Simple Aggregation
- `count(*)`, `count(expr)`, `count(DISTINCT expr)`
- `min(expr)`, `max(expr)`, `avg(expr)`, `sum(expr)`

### Grouped Aggregation
```sql
SELECT col, aggregate(expr)
FROM table
GROUP BY col
ORDER BY aggregate(expr) DESC
```

### HAVING
- Filter on aggregated results (cannot use in `WHERE`)
```sql
GROUP BY col1, col2
HAVING sum(amount) > 300
```

### Multilevel Aggregation
| Clause | Purpose |
|--------|---------|
| **ROLLUP** | Hierarchical subtotals |
| **CUBE** | All grouping combinations |
| **GROUPING SETS** | Custom grouping combinations |

```sql
GROUP BY ROLLUP(revenue_from, facility_name)
GROUP BY CUBE(col1, col2)
GROUP BY GROUPING SETS((col1, col2), (col1), ())
```

---

## 11. Window Functions

### Structure
```sql
agg_function() OVER(
  PARTITION BY col_list
  ORDER BY col_list
  ROWS BETWEEN window_start AND window_end
)
```

### Window Frame
- **Start**: `UNBOUNDED PRECEDING`, `n PRECEDING`, `CURRENT ROW`
- **End**: `CURRENT ROW`, `n FOLLOWING`, `UNBOUNDED FOLLOWING`

### Running/Moving Aggregates
```sql
sum(revenue) OVER(ORDER BY date) AS running_total
avg(revenue) OVER(PARTITION BY booked_by 
  ORDER BY date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS 3_day_avg
```

### Ranking
| Function | Behavior |
|----------|----------|
| `rank()` | Ranks with gaps for ties |
| `dense_rank()` | Ranks without gaps |
| `row_number()` | Unique sequence per partition |

### Analytic
| Function | Purpose |
|----------|---------|
| `lead(col, n)` | Value from n rows ahead |
| `lag(col, n)` | Value from n rows behind |

- Used for sequential analysis, gaps between dates, YoY comparison

---

## 12. Set Operations

| Operator | Behavior |
|----------|----------|
| `UNION` | Combine, remove duplicates |
| `UNION ALL` | Combine, keep duplicates |
| `INTERSECT` | Rows in both |
| `EXCEPT` | Rows in first but not second |

- All queries must have same column structure

---

## 13. Pivoting & Unpivoting

### Pivot
- Rows → columns; requires aggregation
```sql
SELECT * FROM data
PIVOT (sum(amount) FOR product_name IN ("Product A", "Product B", "Product C"))
```

### Unpivot
- Columns → rows; does not undo aggregation
```sql
SELECT * FROM pivot_table
UNPIVOT(amount FOR product IN (col1, col2, col3))
```

---

## 14. Views

- Stored query that behaves like a table
- Query runs each time the view is selected
```sql
CREATE OR REPLACE VIEW view_name AS
SELECT ... FROM ...
```

---

## 15. Auxiliary Statements

| Command | Purpose |
|---------|---------|
| `ANALYZE TABLE table COMPUTE STATISTICS FOR ALL COLUMNS` | Compute statistics |
| `DESCRIBE EXTENDED table` | Table metadata |
| `DESCRIBE TABLE EXTENDED table column` | Column statistics |
| `CACHE TABLE table` | Cache in memory |
| `UNCACHE TABLE table` | Remove from cache |
| `CLEAR CACHE` | Clear all cached tables |
| `SET TIME ZONE 'America/Los_Angeles'` | Session timezone |
| `SHOW DATABASES` | List databases |
| `SHOW TABLES IN db` | List tables |
| `SHOW COLUMNS IN table` | List columns |
| `SHOW CREATE TABLE table` | DDL statement |
| `SHOW VIEWS IN db` | List views |

---

## 16. Data Models (from sql_data)

### Club Database
- **facilities**: facility_name, member_cost, guest_cost, initial_outlay, monthly_maintainance
- **members**: member_id, first_name, last_name, address, zip_code, telephone, recommended_by, joining_date
- **bookings**: booking_id, facility_id, member_id, start_time, slots
- **Source**: CSV in `sql_data/club/`

### Corp Database
- **employees**: employee data, department, salary
- **products**: product name, cost, category
- **clients**: transactions, links to employee and product
- **Source**: CSV in `sql_data/corp/`

### DVD Rental Database
- **actor**, **film**, **film_actor**, **category**, **film_category**
- **store**, **inventory**, **rental**, **payment**
- **staff**, **customer**, **address**, **city**, **country**, **language**
- **Source**: Tab-delimited `.dat` files in `sql_data/dvdrental/`

### Diamonds Dataset
- **Columns**: carat, clarity, color, cut, depth, price
- **Source**: JSON in `sql_data/diamonds.json`

---

## 17. SQL Execution Flow (Order of Clauses)

1. FROM
2. JOIN
3. WHERE
4. GROUP BY
5. HAVING
6. SELECT
7. ORDER BY
8. LIMIT/OFFSET

---

## Quick Reference: Common Patterns

```sql
-- Top N per group
WITH ranked AS (
  SELECT *, rank() OVER(PARTITION BY group_col ORDER BY value_col DESC) r
  FROM table
)
SELECT * FROM ranked WHERE r <= N;

-- Running total
SELECT *, sum(amount) OVER(ORDER BY date) AS running_total FROM table;

-- Gaps/leads
SELECT *, lead(date) OVER(PARTITION BY id ORDER BY date) AS next_date FROM table;

-- Pivot by year
PIVOT (sum(sales) FOR year IN (2022 AS Sales_2022, 2021 AS Sales_2021));
```

---

*Summary generated from ScholarNest SQL course materials (Databricks/Spark SQL dialect).*
