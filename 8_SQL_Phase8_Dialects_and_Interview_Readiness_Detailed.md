# SQL Phase 8: Dialects and Interview Readiness (Detailed Guide)

> The final phase. Two goals:
> 1. **Module 15: Dialect cheat sheet.** Be fluent across MySQL, PostgreSQL, SQL Server, Oracle, BigQuery, Snowflake and Spark SQL, and know the traps that change query results.
> 2. **Module 16: Interview execution.** Clarify, plan, code, test, debug and communicate like a senior engineer, with full mock walk-throughs and a mock interview bank.
> Every topic has: easy explanation, example tables, queries, exact outputs, and step-by-step reasoning.

> **Note:** Phase 2 (Aggregation, Joins and Set Operations) has not been written yet. This phase assumes you know `GROUP BY` and `JOIN` basics (the earlier phases used them lightly). Part 3 (the pattern index) lists the Phase 2 patterns P1 to P11 so you can see the whole map.

---

## Table of Contents

- **Part 0:** Sample data (create once)
- **Part 1:** Module 15: Dialect cheat sheet
- **Part 2:** Module 16: Interview execution skills
- **Part 3:** Master pattern index (P1 to P60)
- **Part 4:** Wrap-up (final checklists, self-test, resources, next steps)

---

# PART 0: Sample Data (Create Once)

Runs in MySQL 8 and PostgreSQL (the dialect examples later show other engines' syntax for the same logic).

```sql
-- 1) orders (same 12 rows as Phases 5 to 7)
CREATE TABLE orders (
    order_id     INT PRIMARY KEY,
    customer_id  INT,
    order_date   DATE,
    amount       INT
);
INSERT INTO orders VALUES
(1,  1, '2026-01-05', 500),
(2,  2, '2026-01-12', 300),
(3,  3, '2026-01-20', 450),
(4,  1, '2026-02-03', 700),
(5,  2, '2026-02-14', 200),
(6,  4, '2026-02-18', 650),
(7,  1, '2026-03-07', 400),
(8,  3, '2026-03-15', 350),
(9,  4, '2026-03-21', 150),
(10, 5, '2026-03-25', 800),
(11, 1, '2026-04-02', 600),
(12, 6, '2026-04-10', 250);

-- 2) customers (customer 7 has never ordered)
CREATE TABLE customers (
    customer_id  INT PRIMARY KEY,
    name         VARCHAR(10)
);
INSERT INTO customers VALUES
(1,'Asha'), (2,'Ravi'), (3,'Meena'), (4,'Kabir'), (5,'Divya'), (6,'Esha'), (7,'Farah');

-- 3) departments and employees (same data as Phase 3)
CREATE TABLE departments (
    dept_id    INT PRIMARY KEY,
    dept_name  VARCHAR(20)
);
INSERT INTO departments VALUES
(10,'IT'), (20,'HR'), (30,'Finance'), (40,'Sales'), (50,'Marketing');

CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(50),
    dept_id     INT,
    salary      INT,
    manager_id  INT
);
INSERT INTO employees VALUES
(1,  'Amit Sharma', 10, 90000, NULL),
(2,  'Priya Singh', 20, 52000, 1),
(3,  'Rahul Verma', 10, 68000, 1),
(4,  'Sneha Iyer',  30, 82000, 1),
(5,  'Karan Mehta', 10, 60000, 3),
(6,  'Neha Gupta',  20, 58000, 2),
(7,  'Vikram Rao',  30, 95000, 4),
(8,  'Anita Desai', 40, 45000, 1),
(9,  'Rohan Das',   10, 68000, 3),
(10, 'Meera Nair',  30, 82000, 4);
```

Quick facts to remember (used to verify answers):
- Total order revenue = **5350** (12 orders, 6 ordering customers, plus customer 7 who never ordered).
- Monthly revenue: Jan **1250**, Feb **1550**, Mar **1700**, Apr **850**.
- Active customers per month: Jan {1,2,3}, Feb {1,2,4}, Mar {1,3,4,5}, Apr {1,6}.
- Company average salary = **70000**; department averages: IT 71500, HR 55000, Finance 86333.33, Sales 45000.

---

# PART 1: MODULE 15: Dialect Cheat Sheet

## 15.1 Strategy: How to Handle Dialects in an Interview

1. **Ask first:** "Which SQL dialect should I use: MySQL, PostgreSQL, SQL Server, or something else?" Most interviews accept any mainstream dialect, but many are run on a specific engine (LeetCode: MySQL/PostgreSQL/SQL Server/Oracle; Snowflake, BigQuery or Spark at data-heavy companies).
2. **Default to standard SQL** (`CASE`, `COALESCE`, `CAST`, `JOIN ... ON`, CTEs, window functions, `EXTRACT`). It runs almost everywhere.
3. **Know your "home" dialect deeply** (MySQL or PostgreSQL) and the **top 20 differences** (this part) for the others.
4. **State the dialect-specific bits aloud:** "I'll use `DATE_TRUNC` here; in MySQL you'd use `DATE_FORMAT`." That shows range and costs nothing.
5. **Mention semantic differences** that change results (NULL sorting, integer division, case sensitivity), not just function names.

## 15.2 The Cheat Sheet

> Table cells show the typical way in each engine. `n` = number, `s` = string, `d` = date, `ts` = timestamp. (The double-pipe operator is the standard string concatenation operator.)

### A) Basics

| Task | MySQL | PostgreSQL | SQL Server | Oracle | BigQuery | Snowflake | Spark SQL |
|---|---|---|---|---|---|---|---|
| First n rows | `LIMIT n` | `LIMIT n` | `SELECT TOP n ...` | `FETCH FIRST n ROWS ONLY` (12c+) | `LIMIT n` | `LIMIT n` | `LIMIT n` |
| Skip m, take n | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` | `ORDER BY ... OFFSET m ROWS FETCH NEXT n ROWS ONLY` | `OFFSET m ROWS FETCH NEXT n ROWS ONLY` | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` | `LIMIT n OFFSET m` (newer versions) |
| SELECT without a table | `SELECT 1` | `SELECT 1` | `SELECT 1` | `SELECT 1 FROM DUAL` | `SELECT 1` | `SELECT 1` | `SELECT 1` |
| Quote an identifier | backticks | double quotes | `[col]` or double quotes | double quotes | backticks | double quotes | backticks |
| Unquoted names are stored as | as written | **lowercase** | as written | **UPPERCASE** | columns case-insensitive, tables case-sensitive | **UPPERCASE** | case-insensitive |
| Date literal | `'2026-10-21'` | `DATE '2026-10-21'` | `'2026-10-21'` | `DATE '2026-10-21'` | `DATE '2026-10-21'` | `DATE '2026-10-21'` | `DATE '2026-10-21'` |
| String comparison | **case-insensitive** (default collation) | case-sensitive | depends on collation (often insensitive) | case-sensitive | case-sensitive | case-sensitive | case-sensitive |
| Case-insensitive LIKE | plain `LIKE` | `ILIKE` | plain `LIKE` (collation) | `UPPER(col) LIKE UPPER(p)` | `LOWER(col) LIKE ...` | `ILIKE` | `ILIKE` (3.3+) |
| Rows from a query into a new table | `CREATE TABLE t AS SELECT` | `CREATE TABLE t AS SELECT` | `SELECT ... INTO t FROM ...` | `CREATE TABLE t AS SELECT` | `CREATE TABLE t AS SELECT` | `CREATE TABLE t AS SELECT` | `CREATE TABLE t AS SELECT` |

### B) Strings

| Task | MySQL | PostgreSQL | SQL Server | Oracle | BigQuery | Snowflake | Spark SQL |
|---|---|---|---|---|---|---|---|
| Concatenate | `CONCAT(a, b)` (NULL if any arg is NULL) | `CONCAT(a, b)` (**ignores** NULLs) or the double-pipe operator (NULL if any) | `CONCAT(a, b)` (NULL treated as empty) or `a + b` (NULL if any) | double-pipe operator (NULL treated as empty); `CONCAT` takes 2 args | `CONCAT(a, b)` (NULL if any) | `CONCAT(a, b)` or double-pipe (NULL if any) | `concat(a, b)` (NULL if any); `concat_ws` skips NULLs |
| Length in characters | `CHAR_LENGTH(s)` (`LENGTH` is bytes) | `LENGTH(s)` | `LEN(s)` (ignores trailing spaces) | `LENGTH(s)` | `LENGTH(s)` | `LENGTH(s)` | `length(s)` |
| Substring (1-based) | `SUBSTRING(s, 2, 3)` | `SUBSTRING(s, 2, 3)` | `SUBSTRING(s, 2, 3)` (length required) | `SUBSTR(s, 2, 3)` | `SUBSTR(s, 2, 3)` | `SUBSTR(s, 2, 3)` | `substring(s, 2, 3)` |
| Left n characters | `LEFT(s, n)` | `LEFT(s, n)` | `LEFT(s, n)` | `SUBSTR(s, 1, n)` | `LEFT(s, n)` | `LEFT(s, n)` | `left(s, n)` |
| Position of text | `INSTR(s, 'x')` or `LOCATE('x', s)` | `POSITION('x' IN s)` or `STRPOS(s, 'x')` | `CHARINDEX('x', s)` | `INSTR(s, 'x')` | `STRPOS(s, 'x')` or `INSTR(s, 'x')` | `POSITION('x', s)` or `CHARINDEX('x', s)` | `instr(s, 'x')` or `locate('x', s)` |
| n-th piece of a delimited string | `SUBSTRING_INDEX(SUBSTRING_INDEX(s, ',', n), ',', -1)` | `SPLIT_PART(s, ',', n)` | `STRING_SPLIT(s, ',')` returns rows | `REGEXP_SUBSTR(s, '[^,]+', 1, n)` | `SPLIT(s, ',')[SAFE_ORDINAL(n)]` | `SPLIT_PART(s, ',', n)` | `split(s, ',')[n-1]` |
| Aggregate strings | `GROUP_CONCAT(x ORDER BY x SEPARATOR ',')` | `STRING_AGG(x, ',' ORDER BY x)` | `STRING_AGG(x, ',') WITHIN GROUP (ORDER BY x)` (2017+) | `LISTAGG(x, ',') WITHIN GROUP (ORDER BY x)` | `STRING_AGG(x, ',' ORDER BY x)` | `LISTAGG(x, ',') WITHIN GROUP (ORDER BY x)` | `array_join(sort_array(collect_list(x)), ',')` |
| Regex test | `s REGEXP 'p'` | `s ~ 'p'` | limited (`LIKE`, `PATINDEX`) | `REGEXP_LIKE(s, 'p')` | `REGEXP_CONTAINS(s, r'p')` | `REGEXP_LIKE(s, 'p')` or `RLIKE` | `s RLIKE 'p'` |
| Regex extract | `REGEXP_SUBSTR(s, 'p')` | `SUBSTRING(s FROM 'p(group)')` | n/a | `REGEXP_SUBSTR(s, 'p')` | `REGEXP_EXTRACT(s, r'p')` | `REGEXP_SUBSTR(s, 'p')` | `regexp_extract(s, 'p', 1)` |
| Left-pad with zeros | `LPAD(s, 5, '0')` | `LPAD(s, 5, '0')` | `RIGHT('00000' + s, 5)` | `LPAD(s, 5, '0')` | `LPAD(s, 5, '0')` | `LPAD(s, 5, '0')` | `lpad(s, 5, '0')` |

### C) NULLs, Types and Arithmetic

| Task | MySQL | PostgreSQL | SQL Server | Oracle | BigQuery | Snowflake | Spark SQL |
|---|---|---|---|---|---|---|---|
| First non-NULL | `COALESCE` | `COALESCE` | `COALESCE` | `COALESCE` | `COALESCE` | `COALESCE` | `coalesce` |
| Two-argument NULL replace | `IFNULL(a, b)` | (use `COALESCE`) | `ISNULL(a, b)` | `NVL(a, b)` | `IFNULL(a, b)` | `IFNULL(a, b)` / `NVL` | `ifnull` / `nvl` |
| NULL-safe equality | `a <=> b` | `a IS NOT DISTINCT FROM b` | `a IS NOT DISTINCT FROM b` (2022+) | no operator (use `DECODE` or manual check) | `a IS NOT DISTINCT FROM b` | `a IS NOT DISTINCT FROM b` / `EQUAL_NULL` | `a <=> b` |
| Cast | `CAST(x AS type)` | `CAST(x AS type)` or `x::type` | `CAST` / `CONVERT(type, x)` | `CAST(x AS type)` / `TO_NUMBER`, `TO_DATE` | `CAST(x AS type)` | `CAST` or `x::type` | `CAST(x AS type)` |
| Cast that returns NULL instead of an error | none (best-effort conversion with warning) | none (check with regex or `CASE`) | `TRY_CAST` | `CAST(x AS type DEFAULT NULL ON CONVERSION ERROR)` (12.2+) | `SAFE_CAST` | `TRY_CAST` | `try_cast` (newer) |
| `7 / 2` for integers | `3.5000` | `3` | `3` | `3.5` | `3.5` | `3.500000` | `3.5` |
| Integer division | `7 DIV 2` | `7 / 2` (integers) | `7 / 2` (integers) | `TRUNC(7 / 2)` | `DIV(7, 2)` | `FLOOR(7 / 2)` | `7 div 2` |
| Divide by zero | NULL (default mode) | **error** | **error** | **error** | **error** (use `SAFE_DIVIDE`) | **error** (use `DIV0`) | NULL (non-ANSI mode) |
| Boolean type | `BOOLEAN` = `TINYINT(1)` | true `BOOLEAN` | `BIT` (no boolean expressions as values) | no SQL boolean column before 23ai | `BOOL` | `BOOLEAN` | `BOOLEAN` |
| Empty string `''` is NULL? | no | no | no | **YES** | no | no | no |
| Where NULLs sort (ASC) | first | **last** | first | **last** | first | **last** | first |
| NULLS FIRST / LAST keyword | no | yes | no | yes | yes | yes | yes |

### D) Dates and Times (reference date 2026-10-21, a Wednesday)

| Task | MySQL | PostgreSQL | SQL Server | Oracle | BigQuery | Snowflake | Spark SQL |
|---|---|---|---|---|---|---|---|
| Today | `CURDATE()` | `CURRENT_DATE` | `CAST(GETDATE() AS DATE)` | `TRUNC(SYSDATE)` | `CURRENT_DATE()` | `CURRENT_DATE()` | `current_date()` |
| Now | `NOW()` | `NOW()` | `GETDATE()` | `SYSTIMESTAMP` | `CURRENT_TIMESTAMP()` | `CURRENT_TIMESTAMP()` | `current_timestamp()` |
| Add 7 days | `DATE_ADD(d, INTERVAL 7 DAY)` | `d + 7` or `d + INTERVAL '7 days'` | `DATEADD(day, 7, d)` | `d + 7` | `DATE_ADD(d, INTERVAL 7 DAY)` | `DATEADD(day, 7, d)` | `date_add(d, 7)` |
| Add 1 month | `DATE_ADD(d, INTERVAL 1 MONTH)` | `d + INTERVAL '1 month'` | `DATEADD(month, 1, d)` | `ADD_MONTHS(d, 1)` | `DATE_ADD(d, INTERVAL 1 MONTH)` | `DATEADD(month, 1, d)` | `add_months(d, 1)` |
| Days between a (later) and b (earlier) | `DATEDIFF(a, b)` | `a - b` | `DATEDIFF(day, b, a)` | `a - b` | `DATE_DIFF(a, b, DAY)` | `DATEDIFF(day, b, a)` | `datediff(a, b)` |
| Month start | `DATE_SUB(d, INTERVAL DAYOFMONTH(d)-1 DAY)` | `DATE_TRUNC('month', d)::date` | `DATEFROMPARTS(YEAR(d), MONTH(d), 1)` | `TRUNC(d, 'MM')` | `DATE_TRUNC(d, MONTH)` | `DATE_TRUNC('month', d)` | `trunc(d, 'MM')` |
| Month end | `LAST_DAY(d)` | `(DATE_TRUNC('month', d) + INTERVAL '1 month - 1 day')::date` | `EOMONTH(d)` | `LAST_DAY(d)` | `LAST_DAY(d, MONTH)` | `LAST_DAY(d)` | `last_day(d)` |
| Extract the year | `YEAR(d)` or `EXTRACT(YEAR FROM d)` | `EXTRACT(YEAR FROM d)` | `YEAR(d)` | `EXTRACT(YEAR FROM d)` | `EXTRACT(YEAR FROM d)` | `YEAR(d)` | `year(d)` |
| Format as `2026-10` | `DATE_FORMAT(d, '%Y-%m')` | `TO_CHAR(d, 'YYYY-MM')` | `FORMAT(d, 'yyyy-MM')` | `TO_CHAR(d, 'YYYY-MM')` | `FORMAT_DATE('%Y-%m', d)` | `TO_CHAR(d, 'YYYY-MM')` | `date_format(d, 'yyyy-MM')` |
| Weekday number of 2026-10-21 | `WEEKDAY(d)` = 2 (Mon = 0); `DAYOFWEEK(d)` = 4 (Sun = 1) | `EXTRACT(ISODOW FROM d)` = 3 (Mon = 1); `DOW` = 3 (Sun = 0) | `DATEPART(weekday, d)` depends on `SET DATEFIRST` | `TO_CHAR(d, 'D')` depends on NLS territory | `EXTRACT(DAYOFWEEK FROM d)` = 4 (Sun = 1) | `DAYOFWEEK(d)` (depends on `WEEK_START` setting) | `dayofweek(d)` = 4 (Sun = 1) |
| Week start (Monday) | `DATE_SUB(d, INTERVAL WEEKDAY(d) DAY)` | `DATE_TRUNC('week', d)` (Monday) | `DATEADD(day, 1 - DATEPART(weekday, d), d)` with Monday first, or `DATETRUNC` (2022+) | `TRUNC(d, 'IW')` | `DATE_TRUNC(d, WEEK(MONDAY))` | `DATE_TRUNC('week', d)` (depends on setting) | `date_trunc('week', d)` (Monday) |
| Timestamp to date | `DATE(ts)` | `ts::date` | `CAST(ts AS DATE)` | `TRUNC(ts)` | `DATE(ts)` | `ts::date` | `to_date(ts)` |
| UTC to IST | `CONVERT_TZ(ts, '+00:00', '+05:30')` | `(ts AT TIME ZONE 'UTC') AT TIME ZONE 'Asia/Kolkata'` | `ts AT TIME ZONE 'UTC' AT TIME ZONE 'India Standard Time'` | `FROM_TZ(ts, 'UTC') AT TIME ZONE 'Asia/Kolkata'` | `DATETIME(ts, 'Asia/Kolkata')` | `CONVERT_TIMEZONE('UTC', 'Asia/Kolkata', ts)` | `from_utc_timestamp(ts, 'Asia/Kolkata')` |

### E) Analytics Features

| Feature | MySQL | PostgreSQL | SQL Server | Oracle | BigQuery | Snowflake | Spark SQL |
|---|---|---|---|---|---|---|---|
| Window functions | 8.0+ | yes | yes | yes | yes | yes | yes |
| `QUALIFY` (filter on window results) | no | no | no | no | **yes** (needs a `WHERE`, `GROUP BY` or `HAVING` clause present; `WHERE TRUE` works) | **yes** | no in open-source Spark (Databricks SQL: yes) |
| `DISTINCT ON (col)` (first row per group) | no | **yes** | no | no | no | no | no |
| `IGNORE NULLS` in `LAST_VALUE` etc. | no | no | 2022+ | yes | yes | yes | `last(x, true)` |
| Use output alias in `GROUP BY` | yes | yes | **no** | **no** | yes | yes | yes |
| `FILTER (WHERE ...)` on aggregates | no | **yes** | no | no | no (use `COUNTIF`, `SUM(IF(...))`) | no | yes (3.0+) |
| Median / percentile | no built-in | `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY x)` | `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY x) OVER ()` | `MEDIAN(x)` | `PERCENTILE_CONT(x, 0.5) OVER ()` | `MEDIAN(x)` | `percentile(x, 0.5)` / `percentile_approx` |
| `FULL OUTER JOIN` | **no** (emulate: `LEFT JOIN UNION RIGHT JOIN`) | yes | yes | yes | yes | yes | yes |
| Set difference | `EXCEPT` (8.0.31+) | `EXCEPT` | `EXCEPT` | `MINUS` | `EXCEPT DISTINCT` | `EXCEPT` / `MINUS` | `EXCEPT` / `MINUS` |
| Intersection | `INTERSECT` (8.0.31+) | `INTERSECT` | `INTERSECT` | `INTERSECT` | `INTERSECT DISTINCT` | `INTERSECT` | `INTERSECT` |
| Recursive CTE | `WITH RECURSIVE` | `WITH RECURSIVE` | `WITH` (no keyword; `OPTION (MAXRECURSION n)`) | `WITH` (or `CONNECT BY`) | `WITH RECURSIVE` | `WITH RECURSIVE` | limited or only in newest versions |
| Generate a series | recursive CTE | `generate_series(a, b)` | recursive CTE or `GENERATE_SERIES` (2022+) | `CONNECT BY LEVEL <= n` | `UNNEST(GENERATE_DATE_ARRAY(a, b))` | `TABLE(GENERATOR(ROWCOUNT => n))` | `explode(sequence(a, b))` |
| Pivot keyword | no (use `CASE`) | no (use `CASE` or `crosstab`) | `PIVOT` | `PIVOT` | `PIVOT` | `PIVOT` | `PIVOT` |
| JSON field | `col->>'$.a'` | `col->>'a'` (jsonb) | `JSON_VALUE(col, '$.a')` | `JSON_VALUE(col, '$.a')` | `JSON_VALUE(col, '$.a')` | `col:a::string` | `get_json_object(col, '$.a')` |
| Array to rows | `JSON_TABLE` (JSON arrays) | `unnest(arr)` | `OPENJSON` / `CROSS APPLY` | `JSON_TABLE` | `CROSS JOIN UNNEST(arr)` | `LATERAL FLATTEN(input => arr)` | `LATERAL VIEW explode(arr)` |
| `GROUP BY ALL` | no | no | no | no | yes (newer) | yes | yes (Databricks) |

### F) DML and Table Design

| Task | MySQL | PostgreSQL | SQL Server | Oracle | BigQuery | Snowflake | Spark SQL |
|---|---|---|---|---|---|---|---|
| Auto-generated key | `AUTO_INCREMENT` | `GENERATED ... AS IDENTITY` | `IDENTITY(1,1)` | `GENERATED AS IDENTITY` (12c+) | none (use `GENERATE_UUID()`) | `AUTOINCREMENT` / `IDENTITY` | identity columns on Delta, or `monotonically_increasing_id()` |
| Upsert | `INSERT ... ON DUPLICATE KEY UPDATE` | `INSERT ... ON CONFLICT DO UPDATE` | `MERGE` | `MERGE` | `MERGE` | `MERGE` | `MERGE` (Delta / Iceberg tables) |
| Update with a join | `UPDATE t JOIN u ON ... SET ...` | `UPDATE t SET ... FROM u WHERE ...` | `UPDATE t SET ... FROM t JOIN u ON ...` | `MERGE`, or correlated subquery | `UPDATE t SET ... FROM u WHERE ...` | `UPDATE t SET ... FROM u WHERE ...` | `MERGE` |
| Get the changed rows back | none | `RETURNING` | `OUTPUT` | `RETURNING ... INTO` | none | none | none |
| Temporary table | `CREATE TEMPORARY TABLE` | `CREATE TEMP TABLE` | `#temp_table` | `GLOBAL TEMPORARY TABLE` | `CREATE TEMP TABLE` | `CREATE TEMPORARY TABLE` | `CREATE TEMPORARY VIEW` |
| Indexes | yes | yes | yes | yes | **no** (partition and cluster instead) | **no** (micro-partitions, clustering keys) | **no** (partition, bucket, Z-order) |
| Cost model | your server | your server | your server | your server | **bytes scanned** | compute time (warehouse size) | cluster time |

## 15.3 Semantics That Change Your Answer (Not Just the Spelling)

These are the differences that make a **correct-looking query return different results**.

| Behavior | Details |
|---|---|
| **Double-pipe in MySQL** | In default mode it is **logical OR**, not concatenation. Use `CONCAT`. |
| **Concatenation with NULL** | MySQL `CONCAT`, BigQuery, Snowflake, Spark: NULL if any argument is NULL. PostgreSQL `CONCAT`, SQL Server `CONCAT`, Oracle: NULL treated as empty. Always wrap in `COALESCE` for portability. |
| **Oracle `''` = NULL** | `WHERE col = ''` never matches in Oracle; `LENGTH('')` is NULL. |
| **Integer division** | PostgreSQL and SQL Server truncate `7/2` to 3; the others return 3.5. Multiply by `1.0` to force decimals everywhere. |
| **Division by zero** | Error in most engines, NULL in MySQL and non-ANSI Spark. Use `NULLIF(den, 0)`. |
| **`DATEDIFF` argument order** | MySQL and Spark: `DATEDIFF(later, earlier)`. SQL Server and Snowflake: `DATEDIFF(unit, earlier, later)`. BigQuery: `DATE_DIFF(later, earlier, DAY)`. |
| **`DATEDIFF` in SQL Server counts boundaries** | `DATEDIFF(year, '2025-12-31', '2026-01-01')` = 1 (one year boundary crossed), not "less than a year". |
| **NULL sort order** | MySQL, SQL Server, BigQuery, Spark: NULLs first in ASC. PostgreSQL, Oracle, Snowflake: NULLs last in ASC. Add `NULLS FIRST/LAST` or sort on `col IS NULL` where unsupported. |
| **`GROUP BY` strictness** | MySQL's default `ONLY_FULL_GROUP_BY` rejects selecting non-aggregated, non-grouped columns (older modes silently picked a random row). SQL Server, Oracle and PostgreSQL always reject it (PostgreSQL allows columns functionally dependent on the primary key). |
| **Alias in `GROUP BY` / `HAVING`** | Works in MySQL, PostgreSQL, BigQuery, Snowflake, Spark; **not** in SQL Server or Oracle (repeat the expression). |
| **Case sensitivity of strings** | MySQL (default) is insensitive; PostgreSQL, Oracle, BigQuery, Snowflake, Spark are sensitive. `WHERE name = 'asha'` may or may not match `'Asha'`. |
| **Identifier case** | Snowflake and Oracle fold unquoted names to **UPPERCASE**; PostgreSQL folds to lowercase. A quoted `"Name"` is then case-sensitive forever. |
| **`TIMESTAMP` meanings** | MySQL `TIMESTAMP` is converted to UTC and back by the session time zone; `DATETIME` is not. PostgreSQL `timestamp` has no zone, `timestamptz` does. Oracle `DATE` **includes a time**. SQL Server `DATETIME` rounds to .000, .003 or .007 seconds. |
| **Window default frame** | With `ORDER BY` and no frame the default is `RANGE ... CURRENT ROW` (ties included); add `ROWS` explicitly for running totals. |
| **`LIMIT` without `ORDER BY`** | Row choice is not guaranteed in any engine. |
| **`COUNT(DISTINCT)` with NULLs** | NULL is ignored by `COUNT(col)` and `COUNT(DISTINCT col)` everywhere; multi-column `COUNT(DISTINCT a, b)` exists in MySQL but not PostgreSQL (use a subquery or row constructor). |
| **`TOP` with ties** | SQL Server `TOP n WITH TIES` includes ties; Oracle `FETCH FIRST n ROWS WITH TIES` too; other engines need `RANK()`. |
| **Spark ANSI mode** | In ANSI mode (default in newer versions) invalid casts and overflow raise errors instead of returning NULL; check the version. |

## 15.4 Behavior Test: Same Literal Query, Different Results

| Query | MySQL | PostgreSQL | SQL Server | Oracle | BigQuery | Snowflake | Spark SQL |
|---|---|---|---|---|---|---|---|
| `SELECT 7 / 2` | 3.5000 | 3 | 3 | 3.5 | 3.5 | 3.500000 | 3.5 |
| `SELECT CONCAT('a', NULL)` | NULL | `a` | `a` | `a` | NULL | NULL | NULL |
| `SELECT 1 / 0` | NULL | error | error | error | error | error | NULL (non-ANSI) |
| Is `''` NULL? | no | no | no | **yes** | no | no | no |
| Where is NULL in `ORDER BY x` (ASC)? | first | last | first | last | first | last | first |
| `SELECT 'abc' = 'ABC'` | 1 (true) | false | depends on collation | false | false | false | false |

(SQL Server and Oracle cannot return a boolean value directly in a `SELECT` list; read the last row as "what that comparison evaluates to inside a `WHERE` clause".)

**Why it matters:** the same logic can silently produce different numbers. Percentage calculations on integer columns (`100 * a / b`) are the classic victim: write `100.0 * a / NULLIF(b, 0)`.

## 15.5 The Same Task in Every Dialect (With Exact Outputs)

Five common tasks on the `orders` table. Each result is identical across engines; only the spelling changes.

### Task 1: Monthly revenue

Expected output:

| order_month | revenue |
|---|---|
| 2026-01 | 1250 |
| 2026-02 | 1550 |
| 2026-03 | 1700 |
| 2026-04 | 850 |

```sql
-- MySQL
SELECT DATE_FORMAT(order_date, '%Y-%m') AS order_month, SUM(amount) AS revenue
FROM orders GROUP BY DATE_FORMAT(order_date, '%Y-%m') ORDER BY order_month;

-- PostgreSQL / Snowflake / Oracle
SELECT TO_CHAR(order_date, 'YYYY-MM') AS order_month, SUM(amount) AS revenue
FROM orders GROUP BY TO_CHAR(order_date, 'YYYY-MM') ORDER BY order_month;

-- SQL Server (no alias in GROUP BY: repeat the expression)
SELECT FORMAT(order_date, 'yyyy-MM') AS order_month, SUM(amount) AS revenue
FROM orders GROUP BY FORMAT(order_date, 'yyyy-MM') ORDER BY order_month;

-- BigQuery
SELECT FORMAT_DATE('%Y-%m', order_date) AS order_month, SUM(amount) AS revenue
FROM orders GROUP BY order_month ORDER BY order_month;

-- Spark SQL
SELECT date_format(order_date, 'yyyy-MM') AS order_month, SUM(amount) AS revenue
FROM orders GROUP BY date_format(order_date, 'yyyy-MM') ORDER BY order_month;
```

### Task 2: Top 3 orders by amount

Expected output:

| order_id | amount |
|---|---|
| 10 | 800 |
| 4 | 700 |
| 6 | 650 |

```sql
-- MySQL, PostgreSQL, BigQuery, Snowflake, Spark SQL
SELECT order_id, amount FROM orders ORDER BY amount DESC LIMIT 3;

-- SQL Server
SELECT TOP 3 order_id, amount FROM orders ORDER BY amount DESC;

-- Oracle (12c+) and standard SQL (also works in PostgreSQL and SQL Server with OFFSET)
SELECT order_id, amount FROM orders ORDER BY amount DESC FETCH FIRST 3 ROWS ONLY;
```

### Task 3: Days between a customer's first and last order

Expected output:

| customer_id | days_between |
|---|---|
| 1 | 87 |
| 2 | 33 |
| 3 | 54 |
| 4 | 31 |
| 5 | 0 |
| 6 | 0 |

```sql
-- MySQL
SELECT customer_id, DATEDIFF(MAX(order_date), MIN(order_date)) AS days_between
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- PostgreSQL / Oracle (date minus date gives days)
SELECT customer_id, MAX(order_date) - MIN(order_date) AS days_between
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- SQL Server / Snowflake
SELECT customer_id, DATEDIFF(day, MIN(order_date), MAX(order_date)) AS days_between
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- BigQuery
SELECT customer_id, DATE_DIFF(MAX(order_date), MIN(order_date), DAY) AS days_between
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- Spark SQL
SELECT customer_id, datediff(MAX(order_date), MIN(order_date)) AS days_between
FROM orders GROUP BY customer_id ORDER BY customer_id;
```

**Step-by-step (customer 1):** first order 2026-01-05, last order 2026-04-02: 26 days left in January + 28 (February) + 31 (March) + 2 = **87**.

### Task 4: List of order ids per customer (string aggregation)

Expected output:

| customer_id | order_ids |
|---|---|
| 1 | 1,4,7,11 |
| 2 | 2,5 |
| 3 | 3,8 |
| 4 | 6,9 |
| 5 | 10 |
| 6 | 12 |

```sql
-- MySQL
SELECT customer_id, GROUP_CONCAT(order_id ORDER BY order_id SEPARATOR ',') AS order_ids
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- PostgreSQL
SELECT customer_id, STRING_AGG(order_id::text, ',' ORDER BY order_id) AS order_ids
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- SQL Server (2017+)
SELECT customer_id, STRING_AGG(CAST(order_id AS VARCHAR(10)), ',') WITHIN GROUP (ORDER BY order_id) AS order_ids
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- Oracle / Snowflake
SELECT customer_id, LISTAGG(order_id, ',') WITHIN GROUP (ORDER BY order_id) AS order_ids
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- BigQuery
SELECT customer_id, STRING_AGG(CAST(order_id AS STRING), ',' ORDER BY order_id) AS order_ids
FROM orders GROUP BY customer_id ORDER BY customer_id;

-- Spark SQL (sort the integers first, then convert; sorting strings would give 1,11,4,7)
SELECT customer_id,
       array_join(transform(sort_array(collect_list(order_id)), x -> CAST(x AS STRING)), ',') AS order_ids
FROM orders GROUP BY customer_id ORDER BY customer_id;
```

### Task 5: Latest order per customer

Expected output:

| customer_id | order_id | order_date | amount |
|---|---|---|---|
| 1 | 11 | 2026-04-02 | 600 |
| 2 | 5 | 2026-02-14 | 200 |
| 3 | 8 | 2026-03-15 | 350 |
| 4 | 9 | 2026-03-21 | 150 |
| 5 | 10 | 2026-03-25 | 800 |
| 6 | 12 | 2026-04-10 | 250 |

```sql
-- Portable: works everywhere (MySQL 8+, PostgreSQL, SQL Server, Oracle, BigQuery, Snowflake, Spark)
SELECT customer_id, order_id, order_date, amount
FROM (SELECT o.*,
             ROW_NUMBER() OVER (PARTITION BY customer_id
                                ORDER BY order_date DESC, order_id DESC) AS rn
      FROM orders o) t
WHERE rn = 1
ORDER BY customer_id;

-- PostgreSQL shortcut: DISTINCT ON
SELECT DISTINCT ON (customer_id) customer_id, order_id, order_date, amount
FROM orders
ORDER BY customer_id, order_date DESC, order_id DESC;

-- Snowflake shortcut: QUALIFY
SELECT customer_id, order_id, order_date, amount
FROM orders
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC, order_id DESC) = 1
ORDER BY customer_id;

-- BigQuery: QUALIFY needs a WHERE / GROUP BY / HAVING clause present
SELECT customer_id, order_id, order_date, amount
FROM orders
WHERE TRUE
QUALIFY ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC, order_id DESC) = 1
ORDER BY customer_id;
```

## 15.6 Dialect Profiles: Strengths, Quirks and When You Meet Them

| Engine | You meet it in | Strengths | Quirks to remember |
|---|---|---|---|
| **MySQL** | web apps, startups, many interview platforms | simple, fast reads, huge community | double-pipe is OR; case-insensitive strings; no `FULL OUTER JOIN`; weak date truncation; window functions only from 8.0; `ONLY_FULL_GROUP_BY` |
| **PostgreSQL** | analytics, startups, data engineering | richest standard SQL: `DISTINCT ON`, `FILTER`, arrays, `jsonb`, `generate_series`, strong types, `RETURNING` | integer division; NULLs sort last; strict about types; `timestamptz` conversions |
| **SQL Server (T-SQL)** | enterprises, Microsoft stack | `TOP`, `CROSS/OUTER APPLY`, temp tables with `#`, `MERGE`, good tooling | `+` for strings; `DATEDIFF` boundaries; no alias in `GROUP BY`; no boolean values; `LEN` ignores trailing spaces |
| **Oracle (PL/SQL)** | banks, large enterprises | `MERGE`, `CONNECT BY`, analytic functions, `MINUS` | `''` is NULL; `DUAL`; `DATE` has time; uppercase folding; no `LIMIT` before 12c |
| **BigQuery** | Google Cloud analytics | serverless, partitioning, `QUALIFY`, nested/repeated data (`STRUCT`, `ARRAY`), `SAFE_` functions | pay per bytes scanned (`SELECT *` is expensive); no indexes; `UNNEST`; `DATE_DIFF` order |
| **Snowflake** | cloud warehouses | `QUALIFY`, `VARIANT` semi-structured data, `FLATTEN`, time travel, cloning | uppercase identifiers; `LISTAGG`; warehouse (compute) sizing drives cost |
| **Spark SQL / Databricks** | big data lakes | `LATERAL VIEW explode`, Delta `MERGE`, `collect_list`, hints (`BROADCAST`) | shuffle cost; skew; case-insensitive names; check ANSI mode and version |

## 15.7 Writing Portable SQL (Rules of Thumb)

1. Use `COALESCE`, `CASE`, `CAST`, `NULLIF`, `EXTRACT`, `CURRENT_DATE` (standard).
2. Use explicit `JOIN ... ON`; never comma joins.
3. Use `CONCAT` plus `COALESCE`, not `+` or the double pipe, when NULLs are possible.
4. Multiply by `1.0` before dividing; wrap denominators in `NULLIF`.
5. Use half-open date ranges: `>= start AND < next_start`.
6. Add tie-breakers and `ORDER BY` everywhere order matters.
7. Prefer `ROW_NUMBER()` subqueries over `DISTINCT ON` or `QUALIFY` when portability matters.
8. Keep dialect-specific bits (date truncation, string aggregation, limits) **isolated in one small CTE** so they are easy to swap.
9. Put dialect notes in comments when asked to write for one engine ("MySQL syntax; Postgres equivalent is ...").

---

# PART 2: MODULE 16: Interview Execution Skills

## 16.1 What Interviewers Actually Evaluate

| Dimension | What a strong answer shows |
|---|---|
| **Correctness** | right result on the sample **and** on edge cases |
| **Problem understanding** | you asked questions and defined terms before coding |
| **Approach and pattern recognition** | you named the technique (window function, gaps and islands, anti-join) |
| **Code quality** | readable CTEs, consistent naming, no unnecessary complexity |
| **Edge-case thinking** | NULLs, ties, duplicates, empty groups, boundaries |
| **Performance awareness** | you can discuss indexes, scans, shuffles, and scale |
| **Communication** | you explain as you go, accept hints, and verify aloud |
| **Composure** | you recover from mistakes calmly |

> **Scoring reality:** a slower, correct, well-explained solution beats a fast, silent, buggy one.

## 16.2 A 45-Minute Interview Timeline (2 Problems)

| Minutes | Activity | What to say or do |
|---|---|---|
| 0 to 3 | **Clarify** | restate the problem; ask the checklist questions (16.3) |
| 3 to 7 | **Plan in English** | name the pattern, outline CTE steps, state the grain of each step |
| 7 to 20 | **Write the query** | build step by step; narrate; keep each CTE single-purpose |
| 20 to 25 | **Test and edge cases** | run through the sample by hand; try ties, NULLs, empty groups |
| 25 to 28 | **Optimize and discuss** | indexes, scan size, scale to billions of rows |
| 28 to 45 | **Problem 2** | same loop, faster |

**If you get stuck for more than 3 minutes:** say what you know, simplify (solve a smaller version first), ask for a hint, or write the "obvious but slow" solution and optimize after. Silence is the real failure.

## 16.3 The Clarifying-Questions Checklist

| Category | Ask | Why it matters |
|---|---|---|
| **Grain** | "What is one row in the output? One per customer? Per customer per month?" | prevents wrong aggregation level |
| **Definitions** | "How do we define active / retained / churned / conversion?" | metrics are ambiguous |
| **Duplicates** | "Can the same user/event/order appear twice?" | `DISTINCT`, dedup, fan-out |
| **NULLs** | "Can this column be NULL? How should NULL be treated?" | `COUNT`, `NOT IN`, sorting, averages |
| **Ties** | "If two customers tie for top, return both or one?" | `RANK` vs `DENSE_RANK` vs `ROW_NUMBER` |
| **Boundaries** | "Are the date ranges inclusive? Is the end date included?" | off-by-one results |
| **Time** | "Which time zone? Is the date a date or a timestamp?" | day boundaries |
| **Missing data** | "Should months/days with no data appear as zero?" | calendar tables, `LEFT JOIN` |
| **Output** | "Which columns, which sort order, which column names?" | exact deliverable |
| **Scale** | "Roughly how many rows? Is performance a concern?" | affects approach |
| **Dialect** | "MySQL, Postgres, SQL Server, or other?" | syntax |
| **Schema** | "Which columns are unique or indexed? Are there constraints?" | join keys, fan-out |

### Example clarifying dialogue

**Problem given:** "Find the top-spending customers each month."

| You ask | Interviewer answers | Effect on your solution |
|---|---|---|
| "Top how many? And by order amount sum per month?" | "Top 2, by total spend in that calendar month." | `SUM(amount)` per customer-month, top 2 per month |
| "If there is a tie for second place?" | "Include everyone tied." | use `RANK`, not `ROW_NUMBER` |
| "Months with no orders: show them?" | "No." | no calendar table needed |
| "Output columns?" | "month, customer_id, total_spend, rank." | exact columns |

## 16.4 The 7-Step Problem-Solving Framework

1. **Restate** the problem in your own words and the output shape.
2. **Identify the tables, keys and grain** of each table (what one row means).
3. **Name the pattern** (use the table in 16.5) and write the plan in plain English, as numbered steps.
4. **Build in CTEs**, one idea per CTE, testing each with `SELECT * FROM cte` mentally or on the sample.
5. **Walk the sample** by hand through each step; state intermediate results.
6. **Attack your own query:** NULLs, ties, empty groups, duplicates, boundaries (16.7).
7. **Discuss performance and alternatives** (indexes, window vs self-join, scale).

## 16.5 Pattern Recognition: Words in the Problem Point to a Technique

| If the problem says... | Think... | Template | Phase |
|---|---|---|---|
| "top N per group", "best in each category" | rank inside a partition | `DENSE_RANK`/`ROW_NUMBER() OVER (PARTITION BY g ORDER BY m DESC)` in a CTE, filter | 4 (P20) |
| "Nth highest" | distinct ranking | `DENSE_RANK() = N`, or `DISTINCT ... LIMIT 1 OFFSET N-1` | 3, 4 (P13, P21) |
| "never", "without", "no orders", "missing" | anti-join | `NOT EXISTS` or `LEFT JOIN ... IS NULL` | 2, 3 (P5, P14) |
| "consecutive", "streak", "in a row" | gaps and islands | `value - ROW_NUMBER()` or `LAG` flag plus running `SUM` | 4 (P28) |
| "running total", "cumulative" | running window | `SUM() OVER (ORDER BY ... ROWS UNBOUNDED PRECEDING)` | 4 (P23) |
| "moving/rolling average" | rolling window | `AVG() OVER (ORDER BY ... ROWS n PRECEDING)` | 4 (P24) |
| "compared to previous", "change from last" | neighbor row | `LAG` / `LEAD` | 4 (P25) |
| "latest/first record per user", "dedupe" | pick one row per key | `ROW_NUMBER() ... = 1` | 4 (P22, P30) |
| "retention", "cohort" | cohort month plus month offset | first-activity CTE, `TIMESTAMPDIFF`, ratio | 5 (P36) |
| "funnel", "conversion by step" | step counts | distinct users per step, `LAG` for step-to-step | 5 (P38) |
| "session", "inactivity gap" | gap flag | `LAG` time diff, flag, running `SUM` | 4 (P29) |
| "every day/month, even with no data" | calendar join | recursive calendar CTE `LEFT JOIN`, `COALESCE` | 3, 5 (P18, P40) |
| "org chart", "all reports", "ancestors" | recursion | `WITH RECURSIVE` anchor plus step | 3 (P17) |
| "bought all", "has every" | relational division | `HAVING COUNT(DISTINCT) = total` | 6 (P56) |
| "A and B but not C" | conditional aggregation | `HAVING SUM(CASE ...) > 0 AND ... = 0` | 6 (P56) |
| "median", "percentile" | ordered middle | `ROW_NUMBER + COUNT`, `PERCENTILE_CONT` | 4, 5 (P27, P51) |
| "% of total", "share" | window total | `x / SUM(x) OVER (PARTITION BY ...)` | 4 (P26) |
| "growth vs last month/year" | `LAG` | `(cur - prev) / NULLIF(prev, 0)` | 5 (P35) |
| "overlap", "conflict", "merge ranges" | interval logic | `a.s <= b.e AND a.e >= b.s`; running `MAX(end)` | 5, 6 (P41, P60) |
| "how many at the same time", "rooms needed" | sweep line | +1/-1 events, running sum, max | 6 (P60) |
| "pairs", "bought together" | self-join | `a.x < b.x` on the same order | 5 (P50) |
| "rows to columns" / "columns to rows" | pivot / unpivot | `SUM(CASE ...)` / `UNION ALL` | 5 (P52) |
| "comma-separated", "JSON", "log line" | parse then explode | split/explode, JSON functions, regex | 6 (P53, P54) |
| "friends of friends", "mutual", "shortest path" | graph | bidirectional edges, self-joins, recursive CTE | 6 (P58) |
| "standings", "team points" | unpivot both sides | `UNION ALL` home and away, aggregate | 6 (P59) |
| "unique running count" | first-seen flag | `ROW_NUMBER = 1` flag plus running `SUM` | 4 (P33) |
| "update/delete duplicates/upsert" | DML patterns | self-join delete, `MERGE`, `ON CONFLICT` | 6 (P55) |

## 16.6 Readable SQL: Before and After

**Problem:** list employees who earn more than their department's average salary.

**Unreadable but correct:**

```sql
select d.dept_name,e.name,e.salary from employees e,departments d where e.dept_id=d.dept_id and e.salary>(select avg(salary) from employees where dept_id=e.dept_id) order by 1;
```

**Clean version:**

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY dept_id
)
SELECT d.dept_name,
       e.name,
       e.salary
FROM employees e
JOIN departments d  ON d.dept_id = e.dept_id
JOIN dept_avg    da ON da.dept_id = e.dept_id
WHERE e.salary > da.avg_salary
ORDER BY d.dept_name, e.name;
```

Both return:

| dept_name | name | salary |
|---|---|---|
| Finance | Vikram Rao | 95000 |
| HR | Neha Gupta | 58000 |
| IT | Amit Sharma | 90000 |

**Style rules that interviewers notice:**

| Rule | Example |
|---|---|
| Uppercase keywords, lowercase names | `SELECT name FROM employees` |
| One clause per line, aligned columns | see above |
| Explicit `JOIN ... ON`, never comma joins | `JOIN departments d ON ...` |
| Meaningful aliases and CTE names | `dept_avg`, not `t1` |
| One idea per CTE | `monthly`, `ranked`, `latest` |
| No `SELECT *` in final output | list the columns |
| Always qualify columns in joins | `e.salary`, `d.dept_name` |
| Comment the **why**, not the what | `-- exclude refunds: finance asked` |
| Deterministic `ORDER BY` with a tie-breaker | `ORDER BY total DESC, customer_id` |

## 16.7 The Edge-Case Checklist (With Tiny Tests)

Before saying "done", attack your query with these. Build a **3 to 6 row** test table by hand.

| Edge case | Test to try | Typical bug |
|---|---|---|
| **NULLs** | a NULL in the filter, join key, or measured column | `= NULL`, `NOT IN` returning nothing, `AVG` ignoring NULL |
| **Ties** | two rows with the same top value | `ROW_NUMBER` dropping one, `LIMIT 1` picking arbitrarily |
| **Duplicates** | the same event twice | double counting, wrong streaks, `COUNT(*)` vs `COUNT(DISTINCT)` |
| **Empty groups** | a customer/department with no rows | inner join dropping them; `COUNT(*)` giving 1 |
| **Single row** | a group with one row | `LAG` NULL, stddev NULL, division by zero |
| **Boundary dates** | events exactly on the start or end date; end of month; leap day | `BETWEEN` on timestamps, off-by-one |
| **Division by zero** | denominator 0 | error or NULL; use `NULLIF` |
| **Case/whitespace** | `'Asha'` vs `'asha '` | failed matches |
| **Data types** | text vs number join keys, integer division | silent wrong results |
| **Order** | tied sort keys | nondeterministic output |
| **Scale** | 10^9 rows | full scans, huge sorts, skew |

### Worked example: attacking a query

**Problem:** "Average order amount per customer, including customers with no orders."

**First attempt:**

```sql
SELECT c.name, AVG(o.amount) AS avg_amount
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.name;
```

**Edge tests:**
1. **Empty group:** customer 7 (Farah) has no orders. The inner join **drops her**. Switch to `LEFT JOIN`.
2. **NULL average:** after `LEFT JOIN`, Farah's `AVG` is NULL (no rows). Decide with the interviewer whether to show NULL or 0 (`COALESCE(AVG(...), 0)`).
3. **Duplicate names:** two customers named "Asha" would be merged by `GROUP BY c.name`. Group by `c.customer_id, c.name`.

**Fixed query:**

```sql
SELECT c.customer_id,
       c.name,
       COALESCE(ROUND(AVG(o.amount), 2), 0) AS avg_amount
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.name
ORDER BY c.customer_id;
```

| customer_id | name | avg_amount |
|---|---|---|
| 1 | Asha | 550.00 |
| 2 | Ravi | 250.00 |
| 3 | Meena | 400.00 |
| 4 | Kabir | 400.00 |
| 5 | Divya | 800.00 |
| 6 | Esha | 250.00 |
| 7 | Farah | 0 |

**Check:** customer 1 has orders 500, 700, 400, 600: (500 + 700 + 400 + 600) / 4 = 2200 / 4 = **550**.

## 16.8 Debugging a Wrong Answer: Classic Bugs With Proof

### Bug 1: A filter on the right table in a `LEFT JOIN` silently turns it into an inner join

**Goal:** per department, the number of employees earning **more than 60000**, including departments with none.

**Buggy:**

```sql
SELECT d.dept_name, COUNT(e.emp_id) AS high_earners
FROM departments d
LEFT JOIN employees e ON e.dept_id = d.dept_id
WHERE e.salary > 60000
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

| dept_name | high_earners |
|---|---|
| Finance | 3 |
| IT | 3 |

**Why wrong:** for HR, Sales and Marketing the right side is NULL or fails the filter; the `WHERE` removes those rows **after** the join, so those departments vanish (the `LEFT JOIN` was cancelled).

**Fix: move the condition into `ON`:**

```sql
SELECT d.dept_name, COUNT(e.emp_id) AS high_earners
FROM departments d
LEFT JOIN employees e
       ON e.dept_id = d.dept_id
      AND e.salary > 60000
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

| dept_name | high_earners |
|---|---|
| Finance | 3 |
| HR | 0 |
| IT | 3 |
| Marketing | 0 |
| Sales | 0 |

**Check:** IT employees above 60000: Amit 90000, Rahul 68000, Rohan 68000 = 3 (Karan has exactly 60000, not above). Finance: 82000, 95000, 82000 = 3.

### Bug 2: `COUNT(*)` after a `LEFT JOIN` counts the unmatched row

```sql
SELECT d.dept_name, COUNT(*) AS employees
FROM departments d
LEFT JOIN employees e ON e.dept_id = d.dept_id
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

Marketing returns **1** (the NULL-extended row counts). Use `COUNT(e.emp_id)` to get **0**.

### Bug 3: "Average of averages"

**Goal:** the company-wide average salary, computed from department averages.

```sql
SELECT AVG(avg_salary) AS company_avg
FROM (SELECT dept_id, AVG(salary) AS avg_salary
      FROM employees
      GROUP BY dept_id) t;
```

| company_avg |
|---|
| 64458.33 |

**Why wrong:** each department gets equal weight regardless of size. IT has 4 people, Sales has 1. The correct company average is `AVG(salary) FROM employees` = **70000** (total 700000 / 10 employees). Compute ratios from **sums and counts**, not from averages of averages.

### Bug 4: `NOT IN` with NULLs (Phase 3)

`WHERE emp_id NOT IN (SELECT manager_id FROM employees)` returns **no rows** because `manager_id` contains NULL. Use `NOT EXISTS`.

### Bug 5: Fan-out inflating sums (Phase 7)

Joining a header table to a many-row detail table and summing a header measure repeats the measure. Pre-aggregate the detail table first.

### A debugging routine

1. Run each CTE **separately** and look at the row counts.
2. Compare **expected vs actual** row counts at each step (`SELECT COUNT(*)`).
3. Check **join cardinality**: is each join key unique where you think it is? (`GROUP BY key HAVING COUNT(*) > 1`).
4. Test with **one entity** (one customer, one day) and compute by hand.
5. Look for **NULL behavior** in each predicate.
6. Remove `DISTINCT`: if the result changes, you were hiding a bad join.

## 16.9 Two Full Mock Walk-Throughs

### Mock 1 (Medium): Month-over-month customer retention

**Prompt:** "For each month, what percentage of the previous month's customers ordered again?"

**Step 1: Clarify.**

| Question | Assumed answer |
|---|---|
| Month = calendar month? | Yes |
| "Customers" = distinct customers who placed at least one order? | Yes |
| "Ordered again" = ordered in the current month, whatever the amount? | Yes |
| Denominator = previous month's active customers? | Yes |
| First month (no previous month)? | Not shown |
| Months with zero orders? | None in this data; mention it |

**Step 2: Plan (English).**
1. One row per (customer, month): `cm`.
2. For each previous-month row, find the same customer in the **next** month (`LEFT JOIN`).
3. Per month: previous actives = rows; retained = matched rows; ratio.

**Step 3: Query (MySQL).**

```sql
WITH cm AS (
    SELECT DISTINCT customer_id,
           DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS m
    FROM orders
)
SELECT DATE_ADD(p.m, INTERVAL 1 MONTH)                          AS report_month,
       COUNT(*)                                                 AS prev_month_customers,
       COUNT(c.customer_id)                                     AS retained,
       ROUND(100.0 * COUNT(c.customer_id) / COUNT(*), 2)        AS retention_pct
FROM cm p
LEFT JOIN cm c
       ON c.customer_id = p.customer_id
      AND c.m = DATE_ADD(p.m, INTERVAL 1 MONTH)
WHERE DATE_ADD(p.m, INTERVAL 1 MONTH) <= (SELECT MAX(m) FROM cm)
GROUP BY p.m
ORDER BY p.m;
```

**Step 4: Walk the sample.**

`cm` (customer-months): Jan {1,2,3}, Feb {1,2,4}, Mar {1,3,4,5}, Apr {1,6}.

| report_month | prev_month_customers | retained | retention_pct |
|---|---|---|---|
| 2026-02-01 | 3 | 2 | 66.67 |
| 2026-03-01 | 3 | 2 | 66.67 |
| 2026-04-01 | 4 | 1 | 25.00 |

- **Feb:** previous (Jan) {1,2,3}; of those, 1 and 2 ordered in Feb (3 did not): 2/3 = **66.67%**.
- **Mar:** previous (Feb) {1,2,4}; 1 and 4 ordered in Mar: 2/3 = **66.67%**.
- **Apr:** previous (Mar) {1,3,4,5}; only 1 ordered in Apr: 1/4 = **25.00%**.

**Step 5: Edge cases.**
- **Duplicates:** several orders by one customer in a month are collapsed by `DISTINCT` in `cm`.
- **Gap months** (a month with no orders at all): that month would be missing from `cm`, so its neighbors would not line up; for such data build a calendar table and join to it.
- **Division by zero:** `COUNT(*)` is at least 1 for any month that appears as `p.m`, so no zero denominators.
- **Last month:** the `WHERE` drops the final month as a "previous" month because its next month is not yet observed.

**Step 6: Performance and scale.** Index `(customer_id, order_date)`; at scale, pre-aggregate to customer-month once (a small table), then self-join that. In Spark, partition by month; the join is on `customer_id` (watch skew on heavy customers).

### Mock 2 (Hard): Longest ordering streak of consecutive months

**Prompt:** "For each customer, find the longest run of consecutive months with orders, and the month it started. On ties, take the earliest streak."

**Step 1: Clarify.** Consecutive calendar months; one order in a month is enough; ties resolved by earliest start; one row per customer.

**Step 2: Plan.** Pattern: **gaps and islands** on months.
1. `cm`: distinct (customer, month).
2. Convert months to an integer index (months since a fixed anchor) and subtract `ROW_NUMBER()` per customer: constant within a run of consecutive months.
3. Group by (customer, island key): streak length and start.
4. Pick the best streak per customer with `ROW_NUMBER` (length desc, start asc).

**Step 3: Query (MySQL).**

```sql
WITH cm AS (
    SELECT DISTINCT customer_id,
           DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS m
    FROM orders
),
g AS (
    SELECT customer_id, m,
           TIMESTAMPDIFF(MONTH, '2026-01-01', m)
             - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY m) AS grp
    FROM cm
),
streaks AS (
    SELECT customer_id,
           MIN(m)   AS streak_start,
           COUNT(*) AS streak_months
    FROM g
    GROUP BY customer_id, grp
),
ranked AS (
    SELECT customer_id, streak_start, streak_months,
           ROW_NUMBER() OVER (PARTITION BY customer_id
                              ORDER BY streak_months DESC, streak_start) AS rn
    FROM streaks
)
SELECT customer_id, streak_months AS longest_streak, streak_start
FROM ranked
WHERE rn = 1
ORDER BY customer_id;
```

**Step 4: Walk the sample.** Intermediate for customer 3 (orders in January and March, a gap in February):

| m | month index | row number | index minus row number (grp) |
|---|---|---|---|
| 2026-01-01 | 0 | 1 | -1 |
| 2026-03-01 | 2 | 2 | 0 |

Two different keys: two streaks of length 1. For customer 1 (Jan to Apr: indexes 0, 1, 2, 3 and row numbers 1, 2, 3, 4) the key is `-1` for all four: one streak of length 4.

Final output:

| customer_id | longest_streak | streak_start |
|---|---|---|
| 1 | 4 | 2026-01-01 |
| 2 | 2 | 2026-01-01 |
| 3 | 1 | 2026-01-01 |
| 4 | 2 | 2026-02-01 |
| 5 | 1 | 2026-03-01 |
| 6 | 1 | 2026-04-01 |

**Step 5: Edge cases.**
- **Tie** (customer 3 has two streaks of length 1): the tie-break `streak_start` ascending picks January.
- **Duplicates within a month:** removed by `DISTINCT` (otherwise `ROW_NUMBER` would advance without the month advancing and break the trick).
- **Single month customer** (5, 6): streak of 1.
- **Anchor date:** the fixed anchor `'2026-01-01'` only needs to be on or before the earliest month; any constant works because it cancels out inside a customer.

**Step 6: Discussion.** Alternative: `LAG` + flag + running `SUM` (Technique B), which generalizes to "gaps up to N months allowed." At scale this is one sort per customer partition; beware skew from customers with huge histories.

## 16.10 Mock Interview Bank

Use these as timed practice. **Do not open the answers until you have tried.** Each problem uses the Part 0 tables.

### Round 1: Warm-up (20 minutes)

1. Find the second highest salary in `employees`.
2. Find customers who never placed an order.
3. List departments whose average salary is above the company average, with the average.
4. Count orders per month.

### Round 2: Core (45 minutes, 2 problems each time)

5. Month-over-month retention (Mock 1) and the longest streak (Mock 2): redo both from scratch.
6. For each month, the top 2 customers by spend (ties included), with their rank.
7. For each customer, the date when their cumulative spend first reached or exceeded 1000 (customers who never reach it are omitted).
8. Customers who ordered in **every** month that appears in the data.

### Round 3: Depth (30 minutes, discussion)

9. Explain how you would find and fix a query that suddenly became 50x slower after data grew.
10. Design a table for 5 billion order events queried by date and customer.
11. Explain how you would make a nightly load of changed orders idempotent.
12. Explain two ways to compute exact distinct customers per week at 500 billion rows, and when you would use an approximation.
13. A report shows revenue 5% higher than finance's number. List five likely SQL causes and how to check each.

<details>
<summary>Answers</summary>

```sql
-- 1  Second highest salary: 90000
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- 2  Customers who never ordered: customer 7 (Farah)
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- 3  IT 71500.00 and Finance 86333.33 (company average 70000)
SELECT d.dept_name, ROUND(AVG(e.salary), 2) AS avg_salary
FROM employees e
JOIN departments d ON d.dept_id = e.dept_id
GROUP BY d.dept_name
HAVING AVG(e.salary) > (SELECT AVG(salary) FROM employees)
ORDER BY d.dept_name;

-- 4  2026-01: 3 | 2026-02: 3 | 2026-03: 4 | 2026-04: 2
SELECT DATE_FORMAT(order_date, '%Y-%m') AS order_month, COUNT(*) AS orders
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY order_month;

-- 5  see Mock 1 and Mock 2 above

-- 6  Top 2 per month by spend (ties included via RANK)
WITH cust_month AS (
    SELECT DATE_FORMAT(order_date, '%Y-%m') AS order_month,
           customer_id,
           SUM(amount) AS total_spend
    FROM orders
    GROUP BY DATE_FORMAT(order_date, '%Y-%m'), customer_id
),
ranked AS (
    SELECT *, RANK() OVER (PARTITION BY order_month ORDER BY total_spend DESC) AS rnk
    FROM cust_month
)
SELECT order_month, customer_id, total_spend, rnk
FROM ranked
WHERE rnk <= 2
ORDER BY order_month, rnk, customer_id;
-- 2026-01: customer 1 (500, rank 1), customer 3 (450, rank 2)
-- 2026-02: customer 1 (700, rank 1), customer 4 (650, rank 2)
-- 2026-03: customer 5 (800, rank 1), customer 1 (400, rank 2)
-- 2026-04: customer 1 (600, rank 1), customer 6 (250, rank 2)

-- 7  Date cumulative spend first reached 1000: only customer 1 (500, then 1200 on 2026-02-03)
WITH running AS (
    SELECT customer_id, order_date,
           SUM(amount) OVER (PARTITION BY customer_id
                             ORDER BY order_date, order_id
                             ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative
    FROM orders
),
crossed AS (
    SELECT customer_id, order_date, cumulative,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS rn
    FROM running
    WHERE cumulative >= 1000
)
SELECT customer_id, order_date AS date_reached, cumulative
FROM crossed
WHERE rn = 1;
-- customer 1 | 2026-02-03 | 1200   (customers 2 to 6 never reach 1000: max totals 500, 800, 800, 800, 250)

-- 8  Customers who ordered in every month in the data (4 months): customer 1
SELECT customer_id
FROM (SELECT DISTINCT customer_id, DATE_FORMAT(order_date, '%Y-%m') AS order_month FROM orders) cm
GROUP BY customer_id
HAVING COUNT(*) = (SELECT COUNT(DISTINCT DATE_FORMAT(order_date, '%Y-%m')) FROM orders);
```

```text
9   Measure (EXPLAIN ANALYZE), find the heaviest node; check scans on big tables, estimated vs actual rows,
    stale statistics (ANALYZE), non-SARGable predicates, missing or unusable index, join fan-out, sort spills,
    parameter or plan changes; in a distributed engine check shuffles, skew, small files, partition pruning.
    Fix the cause, re-measure, verify results.

10  Parquet/ORC (or Delta/Iceberg); partition by event date; cluster/sort/bucket by customer_id;
    files 128 MB to 1 GB with compaction; queries filter on the partition column; retention policy.

11  Merge on a key with last-write-wins (guard with updated_at), or overwrite whole partitions;
    dedupe the batch with ROW_NUMBER; watermark with >= and a lookback window; atomic publish;
    control table and data-quality checks.

12  Exact: dedupe (week, customer) pairs early into a small table, then COUNT(DISTINCT) per week (one heavy shuffle).
    Approximate: HyperLogLog (APPROX_COUNT_DISTINCT), mergeable per-day sketches, about 1 to 3% error, tiny memory.
    Use approximation for dashboards and trends; exact for billing and audits.

13  Fan-out from a join on a non-unique key; duplicates in the source; including cancelled or test orders;
    time zone or date boundary differences; inclusive vs exclusive ranges; double counting after UNION ALL;
    currency or tax inclusion. Check by reconciling counts and sums step by step, and by testing one day or one customer.
```
</details>

## 16.11 Communication Scripts

### While solving

| Moment | Say something like |
|---|---|
| Starting | "Let me restate the problem and confirm the output shape." |
| Choosing an approach | "This looks like a gaps-and-islands problem, so I'll number rows per customer and compare with the month index." |
| Writing a CTE | "This CTE reduces orders to one row per customer-month; I'm using `DISTINCT` to avoid double counting." |
| Noticing a risk | "Customers with no orders would disappear with an inner join, so I'll use a `LEFT JOIN`." |
| Verifying | "On the sample, January has customers 1, 2 and 3, so I expect..." |
| Trade-off | "A correlated subquery is simpler, but a window function scans the table once, so I prefer it." |

### When you are stuck

1. **Simplify:** "Let me first solve it for one customer."
2. **Write the naive version:** "A slow but obviously correct version is a self-join; then I'll optimize."
3. **Ask for a hint** politely and specifically: "Would a window function be appropriate here?"
4. **Break the problem:** list the sub-results you need as a table, then build each.

### Common follow-up questions and short answers

| Follow-up | Strong short answer |
|---|---|
| "How would this run on a billion rows?" | "Filter early and use partition pruning; avoid correlated subqueries; pre-aggregate; index `(customer_id, order_date)` in OLTP or cluster by customer in a warehouse; watch for skew." |
| "Can you do it without a window function?" | "Yes with a correlated subquery or self-join, but it scans more; I'd use it only if windows are unavailable." |
| "What if there are NULLs?" | "Explain exactly how each step treats NULL; use `COALESCE` or `IS NOT NULL` explicitly." |
| "What if there are ties?" | "Depends on the requirement: `RANK` keeps ties, `DENSE_RANK` keeps ties without gaps, `ROW_NUMBER` forces a single winner with a tie-breaker." |
| "How would you test this?" | "Hand-built tiny dataset with edge cases, compare row counts at each CTE, and reconcile totals against a simple independent query." |
| "How would you productionize it?" | "Idempotent incremental load, data-quality checks, partitioning, monitoring, and a documented metric definition." |

## 16.12 The 30-Day Revision Plan

Adjust the pace, but keep the sequence: **learn, solve, review errors, repeat.**

| Days | Focus | Daily work | Targets |
|---|---|---|---|
| 1 to 3 | Phase 1: filters, NULLs, `CASE`, functions | 5 easy problems per day plus re-read NULL rules | 15 easy |
| 4 to 7 | Aggregation, joins, set operations (Phase 2) | 4 problems per day; focus on `LEFT JOIN` filters, `HAVING`, conditional aggregation | 16 easy/medium |
| 8 to 10 | Subqueries and CTEs (Phase 3) | 4 problems per day; `NOT EXISTS`, correlated, recursive CTE | 12 medium |
| 11 to 15 | Window functions (Phase 4) | 4 to 5 problems per day; rank, `LAG`, running totals, gaps and islands | 22 medium/hard |
| 16 to 19 | Date and analytics (Phase 5) | 3 problems per day; retention, funnel, intervals, DAU/MAU | 12 hard |
| 20 to 22 | Advanced (Phase 6) | 3 problems per day; division, graph, JSON/regex, DML | 9 hard |
| 23 to 24 | Performance and modeling (Phase 7) | read plans, do index and SCD exercises; answer Q&A aloud | 10 concepts |
| 25 | Dialect drill (this phase) | translate 10 queries across two dialects | 10 translations |
| 26 to 28 | **Mock interviews** | one 45-minute mock per day (2 problems), timed, aloud | 3 mocks |
| 29 | Error-log review | redo every problem you missed, from scratch | all misses |
| 30 | Light review | pattern index (Part 3), cheat sheets, sleep | 0 new problems |

**Rules that make this work:**
- Keep an **error log** (problem, what you got wrong, the pattern you missed).
- **Redo** hard problems after 3 days and again after a week without looking.
- Always **write the plan in English first**, then SQL.
- Practice **out loud** for at least the mock days.
- Target roughly **150 to 200** problems total, with at least **50 medium/hard** window, date and CTE problems.

## 16.13 Interview-Day Checklist

**Before:**
- [ ] Know the target dialect; review its cheat sheet row from 15.2 (limits, dates, strings, NULL sort).
- [ ] Skim the pattern index (Part 3) and your error log.
- [ ] Have a notepad for the schema, assumptions and a tiny sample.

**During:**
- [ ] Clarify the grain, definitions, ties, NULLs and boundaries.
- [ ] State the plan before coding.
- [ ] Use CTEs, aliases and tie-breakers.
- [ ] Walk the sample by hand.
- [ ] Test edge cases aloud.
- [ ] Mention performance and alternatives.

**After the code:**
- [ ] Re-read the problem statement once more against your output columns and order.
- [ ] Ask: "Is there anything you'd like me to extend or optimize?"

---

# PART 3: Master Pattern Index (P1 to P60)

One row per pattern from the whole course. Use it as your final revision sheet.

| # | Pattern | Trigger words | Core technique | Phase |
|---|---|---|---|---|
| P1 | Count/sum per group | "per", "for each" | `GROUP BY` plus aggregate | 2 |
| P2 | Filter groups | "at least N", "more than N" | `HAVING` | 2 |
| P3 | Conditional aggregation | "side by side", "how many of X and Y" | `SUM(CASE ...)` | 2, 6 |
| P4 | Percentage/ratio per group | "% of", "share" | `SUM(x) / SUM(SUM(x)) OVER ()` | 2, 4 |
| P5 | Missing/unmatched records | "never", "without" | `LEFT JOIN ... IS NULL` | 2 |
| P6 | Self join | "employee and manager", "pairs" | join a table to itself | 2 |
| P7 | Cross join combinations | "all combinations" | `CROSS JOIN` | 2 |
| P8 | Non-equi join | "within range", "between" | `JOIN ... ON x BETWEEN a AND b` | 2 |
| P9 | Join plus aggregate pitfalls | totals look too big | aggregate before joining | 2, 7 |
| P10 | Combine sources | "from both tables" | `UNION ALL`, then aggregate | 2 |
| P11 | A but not B / both A and B | "did A and not B" | `HAVING SUM(CASE ...)`, `EXCEPT`, `INTERSECT` | 2, 6 |
| P12 | Compare to an aggregate | "above average" | scalar or correlated subquery | 3 |
| P13 | Nth highest value | "second highest" | `DISTINCT ... OFFSET`, `DENSE_RANK` | 3, 4 |
| P14 | "Never did X" | "no orders" | `NOT EXISTS` | 3 |
| P15 | Latest record per group | "most recent" | `MAX(date)` subquery or `ROW_NUMBER` | 3, 4 |
| P16 | Multi-step logic | complex requirement | chained CTEs | 3 |
| P17 | Hierarchies | "org chart", "all reports" | recursive CTE | 3 |
| P18 | Number/date series | "every day" | recursive calendar CTE | 3 |
| P19 | Graph traversal basics | "paths", "reachable" | recursive CTE with depth limit | 3 |
| P20 | Top N per group | "top 3 in each" | `DENSE_RANK`/`ROW_NUMBER` plus filter | 4 |
| P21 | Nth highest (overall/group) | "3rd highest per department" | `DENSE_RANK = N` | 4 |
| P22 | Deduplication | "remove duplicates" | `ROW_NUMBER ... = 1` | 4 |
| P23 | Running totals | "cumulative" | `SUM() OVER (... ROWS UNBOUNDED PRECEDING)` | 4 |
| P24 | Moving averages | "rolling 7-day" | `AVG() OVER (... ROWS n PRECEDING)` | 4 |
| P25 | Previous/next comparison | "vs last", "change" | `LAG`/`LEAD` | 4 |
| P26 | Percent of total | "contribution" | `x / SUM(x) OVER (PARTITION BY ...)` | 4 |
| P27 | Median and percentiles | "median", "p90" | `ROW_NUMBER + COUNT`, `PERCENTILE_CONT` | 4 |
| P28 | Gaps and islands | "consecutive", "streak" | `value - ROW_NUMBER()` or `LAG` flag plus `SUM` | 4 |
| P29 | Sessionization | "30 minutes of inactivity" | `LAG` gap flag plus running `SUM` | 4 |
| P30 | First/last event per user | "first purchase" | `ROW_NUMBER = 1`, `FIRST_VALUE` | 4 |
| P31 | Rank with ties | "leaderboard" | `RANK`/`DENSE_RANK` plus tie-breakers | 4 |
| P32 | Difference from group aggregate | "vs department average" | `salary - AVG(salary) OVER (PARTITION BY ...)` | 4 |
| P33 | Cumulative distinct count | "running unique users" | first-seen flag plus running `SUM` | 4 |
| P34 | Daily/weekly/monthly aggregation | "per month" | bucket the date, `GROUP BY` | 5 |
| P35 | MoM / YoY growth | "growth" | `LAG`, `NULLIF` | 5 |
| P36 | Retention and cohorts | "retention", "cohort" | cohort CTE plus month offset | 5 |
| P37 | Churn and activation | "churned", "activated" | self-join to next period; time to second event | 5 |
| P38 | Funnel analysis | "step conversion" | distinct users per step, `LAG` | 5 |
| P39 | Active users | "DAU/WAU/MAU" | calendar plus range join | 5 |
| P40 | Fill missing dates | "show zero days" | calendar `LEFT JOIN`, forward fill trick | 5 |
| P41 | Overlapping intervals | "conflicts", "overlap" | `a.s <= b.e AND a.e >= b.s` | 5 |
| P42 | Time between events | "response time" | `TIMESTAMPDIFF`, `LAG` | 5 |
| P43 | Weekday/weekend/business days | "business days" | calendar plus `WEEKDAY`, holiday anti-join | 5 |
| P44 | Gap-tolerant streaks | "allow one missed day" | `LAG` flag with looser threshold | 5 |
| P45 | CLV, AOV, repeat rate | "lifetime value" | per-customer aggregates | 5 |
| P46 | New vs returning | "new customers" | `MIN(date) OVER (PARTITION BY ...)` | 5 |
| P47 | Conversion and CTR | "rate" | `SUM(CASE)/NULLIF(...)` | 5 |
| P48 | A/B test metrics | "variant" | group by variant, lift | 5 |
| P49 | Top/bottom N, Pareto | "top customers", "80% of revenue" | ranks, cumulative share | 5 |
| P50 | Market basket | "bought together" | self-join on order, `a.x < b.x` | 5 |
| P51 | Median/percentile metrics | "typical value" | median and nearest-rank percentile | 5 |
| P52 | Pivot/unpivot | "rows to columns" | `SUM(CASE)`, `UNION ALL` | 5 |
| P53 | Explode lists | "comma-separated" | split/explode, numbers table | 6 |
| P54 | JSON and log parsing | "JSON", "log line" | JSON functions, regex | 6 |
| P55 | UPDATE/DELETE/UPSERT | "delete duplicates", "upsert" | self-join delete, `MERGE`, `ON CONFLICT` | 6 |
| P56 | Relational division | "bought all" | `HAVING COUNT(DISTINCT) = total`, double `NOT EXISTS` | 6 |
| P57 | Multi-pattern capstone | long business questions | name the patterns, one CTE each | 6 |
| P58 | Graph problems | "mutual friends", "shortest path" | bidirectional edges, self-joins, recursive CTE | 6 |
| P59 | Tournament scoreboard | "standings" | `UNION ALL` home and away, aggregate | 6 |
| P60 | Hard intervals | "rooms needed", "merge", "free time" | sweep line, running `MAX(end)` | 6 |

---

# PART 4: Wrap-Up

## 4.1 Final Pre-Interview Cheat Sheet

```text
ALWAYS     ask: grain, definitions, duplicates, NULLs, ties, boundaries, time zone, dialect, scale
           plan in English first | CTE per idea | tie-breaker in every ORDER BY | verify on the sample

NULL       = NULL never true | IS NULL | NOT IN + NULL = empty -> NOT EXISTS | COUNT(col) skips NULL
           COALESCE | NULLIF(den, 0) | AVG ignores NULL | concat with NULL differs by dialect

JOIN       LEFT JOIN filter on right table goes in ON, not WHERE | COUNT(right.key) not COUNT(*)
           check key uniqueness before joining (fan-out) | aggregate before joining

WINDOW     rank: ROW_NUMBER (unique) RANK (gaps) DENSE_RANK (no gaps) | LAG/LEAD | running: ROWS UNBOUNDED PRECEDING
           filter on window results via CTE | LAST_VALUE needs full frame | gaps and islands: value - ROW_NUMBER

DATES      half-open ranges | bucket to period start | calendar CTE for missing periods | state the time zone
           DATEDIFF argument order differs | week start differs | integer division differs

ANALYTICS  retention = active in period N / cohort size | churn = active P and not P+1 | funnel = distinct users per step
           average of averages is wrong -> divide sums by counts

PERF       SARGable predicates | select needed columns | filter early | index equality-then-range | partition by date
           broadcast small tables | watch skew | EXPLAIN ANALYZE first

SCALE      idempotent loads (MERGE / overwrite partition) | SCD2 for history | approx distinct when exact is too costly

DIALECT    limit: LIMIT / TOP / FETCH FIRST | concat: CONCAT | string agg: GROUP_CONCAT / STRING_AGG / LISTAGG
           date trunc: DATE_TRUNC / TRUNC / DATE_FORMAT | QUALIFY only Snowflake/BigQuery | DISTINCT ON only PostgreSQL
```

