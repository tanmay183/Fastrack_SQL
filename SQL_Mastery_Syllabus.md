# SQL Mastery Syllabus: From Scratch to Hard-Level Interview Problems

> Goal: be able to solve any hard-level SQL coding problem (LeetCode Hard, FAANG / big data interviews).
> Method: learn each **pattern**, then solve 5 to 10 problems per pattern, easy to hard.

---

## How to Use This Syllabus

1. Study one module at a time. Do not skip the foundations.
2. For every pattern: read the concept, write the template query from memory, then solve the practice problems.
3. Keep an **error log** (what you got wrong and why). Revisit it weekly.
4. Always reason about: *grain of the result (one row per what?)*, *NULL behavior*, *duplicates*, *ties*.
5. Practice in one dialect (MySQL or PostgreSQL) first, then learn differences (see Module 15).

**Suggested pace:** 12 to 16 weeks at 1.5 to 2 hours per day.

---

## Phase 1: Foundations (Weeks 1-2)

### Module 1: Core Query Anatomy
- SELECT, FROM, WHERE, ORDER BY, LIMIT / TOP / FETCH FIRST
- Logical order of execution: `FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT`
- Aliases and where they can be used
- DISTINCT and its performance cost
- Operators: `=, <>, IN, NOT IN, BETWEEN, LIKE, IS NULL`
- Pattern matching: `LIKE`, `%`, `_`, `REGEXP / RLIKE / ~`

### Module 2: NULL Handling (Heavily Tested)
- Three-valued logic (TRUE / FALSE / UNKNOWN)
- `IS NULL`, `COALESCE`, `NULLIF`, `IFNULL / NVL`
- NULL in aggregates, joins, `NOT IN` (the classic trap), `ORDER BY`
- `COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`

### Module 3: Data Types and Functions
- **String:** `CONCAT, SUBSTRING, LEFT/RIGHT, LENGTH, TRIM, REPLACE, UPPER/LOWER, SPLIT_PART, POSITION/INSTR`
- **Numeric:** `ROUND, CEIL, FLOOR, MOD, ABS, integer division traps`
- **Date/Time:** `DATE_ADD/DATEADD, DATEDIFF, DATE_TRUNC, EXTRACT, DATE_FORMAT/TO_CHAR, current date functions`
- **Conditional:** `CASE WHEN` (simple and searched), `IF`, `IIF`
- Type casting: `CAST, CONVERT, ::`

**Practice patterns:** filter + sort problems, string cleanup, CASE-based categorization.

---

## Phase 2: Aggregation and Joins (Weeks 3-5)

### Module 4: Aggregation and Grouping
- `COUNT, SUM, AVG, MIN, MAX`
- `GROUP BY` (single and multiple columns), `HAVING` vs `WHERE`
- Conditional aggregation: `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`
- `GROUP_CONCAT / STRING_AGG / LISTAGG`
- `ROLLUP`, `CUBE`, `GROUPING SETS`
- Aggregating before joining (avoiding fan-out / double counting)

**Patterns:**
- P1. Count/sum per group
- P2. Filter groups with HAVING (e.g., customers with more than N orders)
- P3. Conditional aggregation (pivot-like reports)
- P4. Percentage / ratio per group (`SUM(x) / SUM(SUM(x)) OVER ()`)

### Module 5: Joins (All Types)
- `INNER, LEFT, RIGHT, FULL OUTER, CROSS, SELF JOIN`
- `ON` vs `WHERE` placement in outer joins (a very common bug)
- Join on multiple conditions, non-equi joins (range joins, `BETWEEN`)
- Anti-join and semi-join concepts
- Join cardinality: 1:1, 1:N, M:N and row explosion

**Patterns:**
- P5. Find missing / unmatched records (`LEFT JOIN ... IS NULL`)
- P6. Self join: employees vs managers, pairs, consecutive rows
- P7. Cross join to generate combinations (all students x all subjects)
- P8. Non-equi join (e.g., match events within a time range)
- P9. Join + aggregate pitfalls (aggregate first, then join)

### Module 6: Set Operations
- `UNION, UNION ALL, INTERSECT, EXCEPT / MINUS`
- Column count and type compatibility
- Emulating INTERSECT and EXCEPT with joins / `EXISTS`

**Patterns:**
- P10. Combine multiple sources and then aggregate
- P11. "Users who did A but not B" and "did both A and B"

---

## Phase 3: Subqueries, CTEs, Logic (Weeks 5-7)

### Module 7: Subqueries
- Scalar, row, table (derived) subqueries
- Correlated vs non-correlated subqueries
- `IN` vs `EXISTS` vs `JOIN` (performance and NULL differences)
- `ANY / ALL`
- Subquery in SELECT, FROM, WHERE, HAVING

**Patterns:**
- P12. Compare against an aggregate (above-average salary, max per group)
- P13. Nth highest value (subquery, `LIMIT OFFSET`, `DENSE_RANK`)
- P14. `NOT EXISTS` for "never did X"
- P15. Correlated subquery for "latest record per group"

### Module 8: CTEs (Common Table Expressions)
- Basic and chained CTEs, readability and modular design
- Reusing CTEs, CTE vs temp table vs view
- **Recursive CTEs** (anchor + recursive member + termination)

**Patterns:**
- P16. Multi-step logic broken into CTEs
- P17. Hierarchies: org charts, category trees, bill of materials
- P18. Generate number / date series (calendar tables, filling gaps)
- P19. Graph traversal basics (paths, levels, cycle prevention)

---

## Phase 4: Window Functions (Weeks 7-10) - THE MOST IMPORTANT MODULE

### Module 9: Window Function Fundamentals
- Syntax: `func() OVER (PARTITION BY ... ORDER BY ... frame)`
- Window vs GROUP BY (row-preserving vs collapsing)
- **Ranking:** `ROW_NUMBER, RANK, DENSE_RANK, NTILE, PERCENT_RANK, CUME_DIST`
- **Value:** `LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE`
- **Aggregate windows:** `SUM/AVG/COUNT/MIN/MAX OVER`
- **Frames:** `ROWS` vs `RANGE`, `UNBOUNDED PRECEDING`, `CURRENT ROW`, `n PRECEDING / FOLLOWING`
- Default frame gotcha with `LAST_VALUE` and `ORDER BY`
- Named windows (`WINDOW w AS (...)`)
- Filtering on window results (must use a CTE or subquery, since `WHERE` runs before windows)

### Module 10: Window Function Patterns

- P20. **Top N per group** (`ROW_NUMBER` / `DENSE_RANK` + filter)
- P21. **Nth highest / lowest overall and per group**
- P22. **Deduplication** (keep latest or first row per key)
- P23. **Running totals / cumulative sums**
- P24. **Moving averages** (7-day rolling, `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`)
- P25. **Previous / next row comparison** (`LAG/LEAD`): day-over-day, month-over-month growth
- P26. **Percent of total / share of group**
- P27. **Percentiles and median** (`NTILE`, `PERCENTILE_CONT`, row-number tricks)
- P28. **Gaps and Islands** (consecutive sequences) <- *top interview pattern*
  - Technique A: `date - ROW_NUMBER()` grouping
  - Technique B: `LAG` + flag + cumulative sum to create group IDs
  - Problems: consecutive login days, consecutive numbers, streaks, session grouping
- P29. **Sessionization** (new session if gap > 30 min)
- P30. **First / last event per user** (first purchase, last login) with `FIRST_VALUE` or `ROW_NUMBER`
- P31. **Rank with ties and tiebreakers**
- P32. **Difference from group aggregate** (salary minus department average)
- P33. **Cumulative distinct counts** (first-seen flag with `ROW_NUMBER` = 1, then cumulative sum)

---

## Phase 5: Date, Time and Analytics Patterns (Weeks 10-12)

### Module 11: Time-Based Analytics

- P34. **Daily / weekly / monthly aggregation** (`DATE_TRUNC`)
- P35. **Month-over-month, year-over-year growth**
- P36. **Retention and cohort analysis**
  - Cohort = first activity month; retention = activity in month N after
  - Day-1 / Day-7 / Day-30 retention
- P37. **Churn and activation**
- P38. **Funnel analysis** (step conversion, drop-off rate)
- P39. **Active users: DAU / WAU / MAU, rolling active users**
- P40. **Fill missing dates** (calendar CTE + LEFT JOIN)
- P41. **Overlapping intervals / date ranges** (booking conflicts, merging intervals)
- P42. **Time difference between events** (time to first purchase, response time)
- P43. **Weekday / weekend, business-day calculations**
- P44. **Consecutive activity with gaps tolerated**

### Module 12: Business and Product Metrics Patterns

- P45. **Customer lifetime value, average order value, repeat purchase rate**
- P46. **New vs returning customers**
- P47. **Conversion rate, click-through rate, ratio of two counts**
- P48. **A/B test metrics by variant**
- P49. **Top / bottom N with ties and thresholds**
- P50. **Market basket**: items bought together (self join on order ID)
- P51. **Median and percentile-based metrics** (e.g., median salary, p95 latency)
- P52. **Pivot and unpivot** (conditional aggregation, `UNION ALL`, `PIVOT/UNPIVOT`)

---

## Phase 6: Advanced Techniques (Weeks 12-14)

### Module 13: Advanced SQL Constructs
- Complex `CASE` inside aggregates and window functions
- Lateral joins / `CROSS APPLY / OUTER APPLY` (top N per group alternative)
- Array / JSON functions (`JSON_EXTRACT`, `->>`, `UNNEST`, `EXPLODE`, `LATERAL VIEW`)
- Regex extraction and parsing messy strings
- Bitwise operations and number tricks
- `MERGE / UPSERT`, `INSERT ... SELECT`, `UPDATE ... JOIN`, `DELETE` with CTE
- Views, materialized views, temp tables

**Patterns:**
- P53. Exploding delimited / array columns into rows
- P54. Parsing JSON or semi-structured logs
- P55. Update and delete problems (delete duplicate emails keeping the smallest id)
- P56. Relational division ("customers who bought ALL products")
  - `GROUP BY ... HAVING COUNT(DISTINCT x) = (SELECT COUNT(*) ...)`
- P57. Complex multi-CTE problems combining 3 or more patterns
- P58. Graph problems in SQL (friend recommendations, mutual friends, shortest hop via recursive CTE)
- P59. Tournament / matches / scoreboard computation (UNION two sides, aggregate)
- P60. Interval merging and overlapping ranges (hard)

---

## Phase 7: Big Data SQL (Weeks 14-15)

### Module 14: Performance, Internals and Big Data Concepts
- **Execution plans:** `EXPLAIN / EXPLAIN ANALYZE`, reading scan, join and sort nodes
- **Indexes:** B-tree, composite index order, covering index, when indexes are ignored (functions on columns, leading wildcards, implicit casts)
- **Join algorithms:** nested loop, hash join, sort-merge, broadcast join
- **Query optimization:**
  - Filter early, select only needed columns, avoid `SELECT *`
  - Push predicates, avoid correlated subqueries on large data
  - `EXISTS` vs `IN`, `UNION ALL` vs `UNION`
  - Avoid fan-out joins, pre-aggregate
- **Partitioning, bucketing, clustering, sort keys**
- **Data skew** and mitigation (salting, broadcast joins, filtering hot keys)
- **Columnar formats** (Parquet / ORC), predicate pushdown, partition pruning
- **Hive / Spark SQL specifics:** `DISTRIBUTE BY, CLUSTER BY, SORT BY`, `LATERAL VIEW EXPLODE`, `COLLECT_LIST/SET`, `window + skew`
- **Approximate functions:** `APPROX_COUNT_DISTINCT`, HyperLogLog, sampling
- Slowly Changing Dimensions (SCD Type 1, 2, 3) and how to query them
- Incremental loads, idempotent pipelines, late-arriving data, deduplication in pipelines
- Star vs snowflake schema, fact vs dimension tables, normalization vs denormalization
- OLTP vs OLAP, ACID, isolation levels (read committed, repeatable read, serializable), deadlocks

**Interview-style questions:**
- Why is my query slow and how do you fix it?
- How do you deduplicate a billion-row table?
- How do you handle skewed joins in Spark?
- How do you model an SCD2 dimension and query "as of" a date?

---

## Phase 8: Dialect Differences and Interview Readiness (Week 16)

### Module 15: Dialect Cheat Sheet (MySQL / PostgreSQL / SQL Server / Oracle / BigQuery / Snowflake / Spark SQL)
| Concept | Differences to know |
|---|---|
| Limiting rows | `LIMIT` / `TOP` / `FETCH FIRST` |
| String aggregation | `GROUP_CONCAT` / `STRING_AGG` / `LISTAGG` |
| Date truncation | `DATE_TRUNC` / `DATE_FORMAT` / `TRUNC` |
| Date difference | `DATEDIFF` / `AGE` / subtraction |
| Median | `PERCENTILE_CONT` availability |
| Pivot | native `PIVOT` vs `CASE` |
| Null-safe equality | `<=>` / `IS NOT DISTINCT FROM` |
| `FULL OUTER JOIN` | not in older MySQL, emulate with `UNION` |

### Module 16: Interview Execution Skills
- Clarifying questions: grain, duplicates, NULLs, ties, time zones, inclusive vs exclusive ranges
- Writing readable SQL: CTE naming, formatting, comments
- Edge cases checklist: empty groups, ties, NULLs, duplicates, single-row groups, boundary dates
- Verifying correctness with tiny hand-made datasets
- Explaining time and space complexity and optimization options
- Mock interviews: 45 minutes, 2 problems (1 medium + 1 hard)

---

## Pattern Quick Reference (Cheat Sheet)

| If the problem says... | Think... |
|---|---|
| "Top N per group" | `DENSE_RANK / ROW_NUMBER` in CTE, filter |
| "Nth highest salary" | `DENSE_RANK`, or `LIMIT OFFSET` with `DISTINCT` |
| "Consecutive days / numbers" | Gaps and islands |
| "Compared to previous row" | `LAG / LEAD` |
| "Running total / rolling average" | `SUM/AVG OVER (ORDER BY ... ROWS ...)` |
| "Never / no records" | `NOT EXISTS` or `LEFT JOIN ... IS NULL` |
| "Customers who did all X" | Relational division with `HAVING COUNT(DISTINCT)` |
| "Duplicates" | `ROW_NUMBER() OVER (PARTITION BY key)` > 1 |
| "Retention / cohort" | First-activity CTE + join to activity + `DATEDIFF` |
| "Missing dates / zero rows" | Calendar CTE + `LEFT JOIN` + `COALESCE` |
| "Hierarchy / tree" | Recursive CTE |
| "Rows to columns" | Conditional aggregation |
| "Overlapping ranges" | Self join on `a.start <= b.end AND a.end >= b.start`, or sort + running max |
| "Median / percentile" | `PERCENTILE_CONT`, or `ROW_NUMBER` + count trick |
| "Session" | `LAG` gap flag + cumulative sum |

---

## Practice Plan by Difficulty

| Stage | Goal | Volume |
|---|---|---|
| Easy | Syntax, joins, aggregates, NULLs | 40 problems |
| Medium | Window functions, subqueries, CTEs, date logic | 80 problems |
| Hard | Gaps and islands, recursive CTEs, cohorts, intervals, multi-pattern | 40+ problems |

### Recommended Platforms
- **LeetCode** (Database section: Easy / Medium / Hard)
- **HackerRank** (SQL track, Advanced SQL)
- **StrataScratch** and **DataLemur** (real company interview questions)
- **SQLZoo**, **Mode SQL Tutorial** (fundamentals)
- **Advent of SQL** and **pgexercises** (PostgreSQL practice)

### Recommended Reading
- *SQL Antipatterns* by Bill Karwin
- *Learning SQL* by Alan Beaulieu
- *High Performance MySQL* (for internals and tuning)
- *The Data Warehouse Toolkit* by Ralph Kimball (for modeling and SCD)
- PostgreSQL and Spark SQL official documentation

---

## Progress Tracker

- [ ] Phase 1: Foundations (Modules 1-3)
- [ ] Phase 2: Aggregation and Joins (Modules 4-6)
- [ ] Phase 3: Subqueries and CTEs (Modules 7-8)
- [ ] Phase 4: Window Functions (Modules 9-10)
- [ ] Phase 5: Time-Based and Business Analytics (Modules 11-12)
- [ ] Phase 6: Advanced Techniques (Module 13)
- [ ] Phase 7: Big Data SQL and Performance (Module 14)
- [ ] Phase 8: Dialects and Interview Readiness (Modules 15-16)

---

## Next Step

Start with **Phase 1, Module 1**. When you are ready, tell me and I will teach it in full detail, with concepts, templates, worked examples, and graded practice problems.
