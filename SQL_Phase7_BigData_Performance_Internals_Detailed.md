# SQL Phase 7: Big Data SQL, Performance and Internals (Detailed Guide)

> This is the **senior-level** module: why queries are slow, how engines execute SQL, how to design tables for billions of rows, how Spark/Hive behave, how data warehouses and pipelines are modeled, and how transactions really work.
> Covers **Module 14**: execution plans, indexes, join algorithms, optimization, partitioning and bucketing, columnar formats, data skew, Hive/Spark specifics, approximate functions, SCD types, incremental loads, star schemas, OLTP vs OLAP, ACID, isolation levels and deadlocks.
> Every topic has: easy explanation, example tables, queries, outputs, and step-by-step reasoning.

> **Honesty note about outputs:** EXPLAIN output (plan text, costs, timings) depends on your database version, data size and statistics, so the plans shown here are **illustrative and simplified**. They teach you *what to look for*. All **data outputs** (rows, counts, sums) in this guide are exact and computed by hand from the sample data.

---

## Table of Contents

- **Part 0:** The performance mindset
- **Part 1:** Sample data and an optional lab
- **Part 2:** Query performance: plans, indexes, SARGable SQL, joins, optimization patterns
- **Part 3:** Big data layout: partitioning, columnar formats, skew, Hive/Spark, approximate functions
- **Part 4:** Warehousing and pipelines: star schema, SCD, incremental loads
- **Part 5:** Transactions: ACID, isolation levels, locks, deadlocks
- **Part 6:** Interview playbook (Q&A outlines)
- **Part 7:** Wrap-up (mistakes, cheat sheet, practice, self-test, resources, next step)

> **Prerequisites:** Phases 1 to 6, especially window functions (Phase 4) and CTEs (Phase 3).

---

# PART 0: The Performance Mindset

## 0.1 What makes a query slow? Four costs

| Cost | Meaning | Typical fix |
|---|---|---|
| **Rows scanned** | how much data is read from disk | indexes, partition pruning, column pruning, columnar formats |
| **Rows shuffled or sorted** | data moved across the network or sorted/grouped | filter early, pre-aggregate, broadcast small tables, avoid unnecessary `ORDER BY`/`DISTINCT` |
| **Rows produced by joins** | row explosion (fan-out) | join on unique keys, aggregate before joining |
| **Waiting** | locks, queueing, skew (one worker does all the work) | shorter transactions, consistent lock order, salting |

> **Golden rule:** *make the engine touch less data, as early as possible.*

## 0.2 The optimization checklist (use in interviews)

1. **Measure first.** Reproduce the slowness; look at the plan (`EXPLAIN`), not guesses.
2. **Read fewer columns** (no `SELECT *`).
3. **Filter early** and make predicates **index-friendly (SARGable)**.
4. **Reduce data before joining** (pre-aggregate, filter both sides).
5. **Check join order, join type, and join keys** (same data types, unique keys on at least one side).
6. **Use the right physical design:** indexes (OLTP) or partitions/clustering/columnar (big data).
7. **Update statistics** so the optimizer estimates row counts correctly.
8. **Look for skew and shuffles** in distributed engines.
9. **Verify** results are unchanged after the rewrite.

---

# PART 1: Sample Data and Optional Lab

## 1.1 Small sample tables (exact outputs in this guide)

```sql
-- 1) orders (same 12 rows as Phases 5 and 6)
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

-- 2) hash-join demo
CREATE TABLE hj_customers (id INT PRIMARY KEY, name VARCHAR(10));
INSERT INTO hj_customers VALUES (1,'Asha'), (2,'Ravi'), (3,'Meena');

CREATE TABLE hj_orders (order_id INT PRIMARY KEY, customer_id INT);
INSERT INTO hj_orders VALUES (101,1), (102,1), (103,2), (104,4);

-- 3) fan-out demo
CREATE TABLE fo_orders (order_id INT PRIMARY KEY, customer_id INT, amount INT);
INSERT INTO fo_orders VALUES (1,1,500), (2,1,300), (3,2,200);

CREATE TABLE fo_items (order_id INT, item VARCHAR(5));
INSERT INTO fo_items VALUES
(1,'pen'), (1,'ink'),
(2,'bag'), (2,'cap'), (2,'tie'),
(3,'cup');

-- 4) page_views (skew demo: user u1 is a "hot key")
CREATE TABLE page_views (view_id INT PRIMARY KEY, user_id VARCHAR(3), page VARCHAR(5));
INSERT INTO page_views VALUES
(1,'u1','home'), (2,'u1','home'), (3,'u1','cart'), (4,'u1','home'),
(5,'u1','cart'), (6,'u1','pay'),  (7,'u1','home'), (8,'u1','cart'),
(9,'u2','home'), (10,'u3','cart');

-- 5) star schema
CREATE TABLE dim_date (date_key INT PRIMARY KEY, full_date DATE, year_month CHAR(7));
INSERT INTO dim_date VALUES
(20261001,'2026-10-01','2026-10'),
(20261002,'2026-10-02','2026-10'),
(20261101,'2026-11-01','2026-11');

CREATE TABLE dim_product (product_key INT PRIMARY KEY, product_name VARCHAR(10), category VARCHAR(12));
INSERT INTO dim_product VALUES (1,'Laptop','Electronics'), (2,'Mouse','Electronics'), (3,'Desk','Furniture');

CREATE TABLE fact_sales (
    sale_id INT PRIMARY KEY, date_key INT, product_key INT, qty INT, amount INT
);
INSERT INTO fact_sales VALUES
(1, 20261001, 1, 1, 1200),
(2, 20261001, 2, 5,  100),
(3, 20261002, 3, 1,  300),
(4, 20261101, 1, 1,  600),
(5, 20261101, 3, 2,  500);

-- 6) slowly changing dimension (state BEFORE the nightly load)
CREATE TABLE dim_customer_scd2 (
    sk           INT AUTO_INCREMENT PRIMARY KEY,   -- PostgreSQL: sk INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
    customer_id  INT,
    name         VARCHAR(10),
    city         VARCHAR(10),
    valid_from   DATE,
    valid_to     DATE,           -- exclusive end; '9999-12-31' means "still current"
    is_current   INT
);
INSERT INTO dim_customer_scd2 (sk, customer_id, name, city, valid_from, valid_to, is_current) VALUES
(1, 7, 'Asha', 'Mumbai', '2025-01-01', '9999-12-31', 1),
(2, 8, 'Ravi', 'Delhi',  '2025-01-01', '9999-12-31', 1);

CREATE TABLE stg_customer (          -- today's snapshot from the source system
    customer_id    INT,
    name           VARCHAR(10),
    city           VARCHAR(10),
    effective_date DATE
);
INSERT INTO stg_customer VALUES
(7, 'Asha',  'Pune',    '2026-06-01'),   -- moved city: CHANGED
(8, 'Ravi',  'Delhi',   '2026-06-01'),   -- UNCHANGED
(9, 'Meera', 'Chennai', '2026-06-01');   -- NEW

CREATE TABLE fact_cust_sales (
    sale_id INT PRIMARY KEY, customer_id INT, sale_date DATE, amount INT
);
INSERT INTO fact_cust_sales VALUES
(1, 7, '2026-03-15', 100),
(2, 7, '2026-07-10', 200),
(3, 8, '2026-07-11',  50),
(4, 9, '2026-06-02',  75);

-- 7) incremental load demo
CREATE TABLE src_changes (id INT, val VARCHAR(5), updated_at DATETIME);   -- a change feed (many versions per key)
INSERT INTO src_changes VALUES
(1, 'a',  '2026-10-20 10:00:00'),
(2, 'b',  '2026-10-20 11:00:00'),
(1, 'a2', '2026-10-21 09:00:00'),
(3, 'c',  '2026-10-21 09:30:00'),
(2, 'b2', '2026-10-21 10:00:00'),
(3, 'c2', '2026-10-21 10:30:00');

CREATE TABLE target_t (id INT PRIMARY KEY, val VARCHAR(5), updated_at DATETIME);   -- state after batch 1
INSERT INTO target_t VALUES
(1, 'a', '2026-10-20 10:00:00'),
(2, 'b', '2026-10-20 11:00:00');

-- 8) accounts (transactions demo)
CREATE TABLE accounts (id INT PRIMARY KEY, balance INT);
INSERT INTO accounts VALUES (1, 100);
```

## 1.2 Optional lab: a big table to see real plans (PostgreSQL)

Small tables always use full scans because that is cheapest. To **see real plans**, create a bigger table:

```sql
-- PostgreSQL: 1,000,000 orders and 10,000 customers
CREATE TABLE big_orders AS
SELECT g                                   AS order_id,
       (random() * 9999)::int + 1          AS customer_id,
       DATE '2026-01-01' + (random() * 299)::int AS order_date,
       (random() * 1000)::int              AS amount,
       CASE WHEN random() < 0.5 THEN 'paid' ELSE 'pending' END AS status
FROM generate_series(1, 1000000) g;

CREATE TABLE big_customers AS
SELECT g AS customer_id, 'cust_' || g AS name
FROM generate_series(1, 10000) g;
ALTER TABLE big_customers ADD PRIMARY KEY (customer_id);

ANALYZE big_orders;
ANALYZE big_customers;
```

Then run `EXPLAIN ANALYZE` before and after creating each index shown in this guide. Your numbers will differ from the illustrative ones in the text, but the **shape** of the plans will match.

---

# PART 2: Query Performance

## 14.1 Execution Plans (EXPLAIN)

### What is a plan?

You write **what** you want (declarative SQL). The **optimizer** decides **how**: which index to use, which join order, which join algorithm. The result is a **plan tree**. Reading it is the #1 performance skill.

| Command | What it does |
|---|---|
| `EXPLAIN query` | shows the **estimated** plan (query is not run) |
| `EXPLAIN ANALYZE query` | **runs** the query and shows estimated vs **actual** rows and timings (PostgreSQL; MySQL 8.0.18+) |
| Spark SQL: `EXPLAIN FORMATTED query` | shows the physical plan with details (also `EXPLAIN EXTENDED`, `EXPLAIN COST`) |

> **Careful:** `EXPLAIN ANALYZE` on `UPDATE`/`DELETE` really executes it. Wrap in a transaction and `ROLLBACK`.

### Reading a MySQL plan (illustrative)

Query: `SELECT order_id, amount FROM big_orders WHERE customer_id = 42;`

**Before any index:**

```text
+----+-------------+------------+------+---------------+------+------+------+---------+-------------+
| id | select_type | table      | type | possible_keys | key  | ref  | rows | filtered| Extra       |
+----+-------------+------------+------+---------------+------+------+------+---------+-------------+
|  1 | SIMPLE      | big_orders | ALL  | NULL          | NULL | NULL |998000|   10.00 | Using where |
+----+-------------+------------+------+---------------+------+------+------+---------+-------------+
```

**After `CREATE INDEX idx_customer ON big_orders(customer_id);`:**

```text
+----+-------------+------------+------+---------------+--------------+-------+------+----------+-------+
| id | select_type | table      | type | possible_keys | key          | ref   | rows | filtered | Extra |
+----+-------------+------------+------+---------------+--------------+-------+------+----------+-------+
|  1 | SIMPLE      | big_orders | ref  | idx_customer  | idx_customer | const |  100 |   100.00 | NULL  |
+----+-------------+------------+------+---------------+--------------+-------+------+----------+-------+
```

**How to read the columns:**

| Column | Meaning |
|---|---|
| `type` | **access method**, best to worst: `system`, `const`, `eq_ref`, `ref`, `range`, `index`, `ALL` |
| `key` | the index actually used (`NULL` means none) |
| `rows` | estimated rows to examine |
| `Extra` | `Using index` = covering index (great), `Using where` = extra filtering, `Using filesort` = a sort step, `Using temporary` = temp table (often from `GROUP BY`/`DISTINCT`) |

`ALL` means a **full table scan** of about a million rows. `ref` means an **index lookup** on about 100 rows.

### Reading a PostgreSQL plan (illustrative)

**Before the index:**

```text
Seq Scan on big_orders  (cost=0.00..20834.00 rows=100 width=8)
                        (actual time=0.020..95.100 rows=98 loops=1)
  Filter: (customer_id = 42)
  Rows Removed by Filter: 999902
Planning Time: 0.1 ms
Execution Time: 95.2 ms
```

**After `CREATE INDEX idx_big_orders_customer ON big_orders(customer_id);`:**

```text
Bitmap Heap Scan on big_orders  (cost=5.2..380.1 rows=100 width=8)
                                (actual time=0.05..0.30 rows=98 loops=1)
  Recheck Cond: (customer_id = 42)
  ->  Bitmap Index Scan on idx_big_orders_customer  (cost=0.0..5.2 rows=100 width=0)
        Index Cond: (customer_id = 42)
Execution Time: 0.4 ms
```

**Reading it:**
- **Seq Scan** + `Rows Removed by Filter: 999902` = the database read a million rows to keep 98. That is the red flag.
- **Index Scan / Bitmap Index Scan** jumps straight to the matching entries.
- `cost=start..total` are optimizer units, not milliseconds; `actual time` is real.
- Always compare **estimated `rows`** with **actual `rows`**. A big mismatch (estimated 100, actual 500,000) means bad statistics, which leads to bad join choices. Fix with `ANALYZE`.

### Common plan nodes (PostgreSQL)

| Node | Meaning | Good or bad? |
|---|---|---|
| `Seq Scan` | read the whole table | fine for small tables or when most rows are needed; red flag when selecting few rows from a big table |
| `Index Scan` / `Index Only Scan` | use an index (the latter does not touch the table) | good for selective filters |
| `Bitmap Heap Scan` | collect many index hits, then read the table pages in order | good for medium selectivity |
| `Nested Loop` | for each outer row, look up inner rows | great with an index on the inner side and a small outer side |
| `Hash Join` | build a hash table on one side, probe from the other | good for big joins without useful indexes |
| `Merge Join` | merge two sorted inputs | good when both sides are already sorted |
| `Sort` | explicit sort (watch for "Sort Method: external merge Disk") | red flag when it spills to disk |
| `HashAggregate` / `GroupAggregate` | `GROUP BY` via hash table or sorted input | normal |

### A plan-reading checklist

1. Which table is scanned fully? Is that expected?
2. Are estimated rows close to actual rows?
3. Which join algorithm, and is the big table on the right side?
4. Is there a `Sort` that spills to disk, or a `Filter` that removes almost everything?
5. Where is most of the time spent (find the node with the largest actual time)?

### Spark SQL plan clues

| Plan text | Meaning |
|---|---|
| `FileScan parquet ... PartitionFilters: [...]` | partition pruning is working (filters on partition columns) |
| `PushedFilters: [GreaterThan(amount, 700)]` | predicate pushdown into the Parquet reader |
| `Exchange hashpartitioning(...)` | a **shuffle** (expensive network step) |
| `BroadcastHashJoin` | small table broadcast to all workers: no shuffle of the big table |
| `SortMergeJoin` | both sides shuffled and sorted: the default for big-big joins |
| `AdaptiveSparkPlan` | adaptive query execution (AQE) may re-optimize at runtime |

---

## 14.2 Indexes

### What an index is

An index is a separate, sorted structure that lets the database **jump** to rows instead of reading everything (like a book's index).

| Index type | Good for | Notes |
|---|---|---|
| **B-tree** (default) | equality, ranges, `ORDER BY`, prefix `LIKE 'abc%'` | lookup in about `O(log n)` |
| **Hash** | equality only | no ranges or sorting |
| **Bitmap** (Oracle, warehouses) | low-cardinality columns (status, gender) | not for heavy-write tables |
| **GIN / GiST** (PostgreSQL) | full text, JSON, arrays, geometry | specialized |
| **Clustered** (InnoDB primary key, SQL Server) | the table itself is stored in key order | one per table; range scans on the key are very fast |
| **Secondary / non-clustered** | other lookup paths | in InnoDB it stores the primary key, so a lookup may need a second hop |
| **Covering** | the index contains every column the query needs | no table access (`Using index` / `Index Only Scan`) |
| **Partial / filtered** | only rows matching a condition | `CREATE INDEX ... WHERE status = 'pending'` (PostgreSQL) |
| **Functional / expression** | queries that apply a function | `CREATE INDEX ... ON t (LOWER(email))` (PostgreSQL; MySQL 8.0.13+) |

**Trade-offs:** every index makes `INSERT/UPDATE/DELETE` slower and takes storage. Index what you **query**, not everything.

### Selectivity: when an index helps

**Selectivity** = fraction of rows a predicate keeps. An index helps when the predicate keeps **few** rows (roughly under 5 to 15%).

| Predicate | Rows kept | Index useful? |
|---|---|---|
| `order_id = 12345` | 1 of 1,000,000 | Yes, excellent |
| `customer_id = 42` | about 100 | Yes |
| `status = 'paid'` | about 50% | No: scanning the table is cheaper; the optimizer ignores the index |
| `gender = 'F'` | about 50% | No |

Tiny tables also ignore indexes (a scan of a few pages is cheaper). That is why your 12-row tables never show index use.

### Composite (multi-column) indexes: column order matters

An index on `(customer_id, order_date, status)` is sorted by `customer_id`, then by `order_date` within each customer, then by `status`. It follows the **leftmost-prefix rule**: the index can only be **used to seek** starting from the first column.

| Query predicate | Uses the index how? |
|---|---|
| `customer_id = 5` | Seeks on `customer_id` (prefix) |
| `customer_id = 5 AND order_date >= '2026-01-01'` | Seeks on both (equality, then range) |
| `customer_id = 5 AND order_date >= '2026-01-01' AND status = 'paid'` | Seeks on the first two; `status` is only filtered inside the range (columns after a **range** column cannot narrow the seek) |
| `customer_id = 5 AND status = 'paid'` | Seeks on `customer_id`; `status` checked on index entries within that customer |
| `order_date >= '2026-01-01'` (no `customer_id`) | **Poor**: the leading column is missing, so a full index or table scan (some engines can "skip scan") |
| `WHERE customer_id = 5 ORDER BY order_date` | Index order satisfies the sort: **no filesort** |

**Rule of thumb for column order:**
1. **Equality** predicates first.
2. Then the **range** or **sort** column.
3. Then columns needed only for covering.
4. Among equality columns, put the one used by **most queries** first (a common leading column serves many queries).

**Example:** the query `SELECT order_id, amount FROM big_orders WHERE customer_id = 42 AND order_date >= '2026-06-01' ORDER BY order_date;` is served perfectly by `CREATE INDEX idx_cust_date ON big_orders (customer_id, order_date);`. It seeks on both columns and returns rows already sorted.

### Covering index

```sql
CREATE INDEX idx_cover ON big_orders (customer_id, order_date, amount);

SELECT order_date, amount
FROM big_orders
WHERE customer_id = 42;
```

Every column the query needs (`customer_id`, `order_date`, `amount`) is inside the index, so the database never reads the table: MySQL shows `Extra: Using index`; PostgreSQL shows `Index Only Scan`. This is often the single biggest speedup for read-heavy OLTP queries.

### When an index is silently ignored

| Cause | Example |
|---|---|
| Function on the indexed column | `WHERE YEAR(order_date) = 2026` |
| Leading wildcard | `WHERE name LIKE '%son'` |
| Implicit type conversion | `WHERE phone = 9876543210` when `phone` is `VARCHAR` |
| Low selectivity | `WHERE status = 'paid'` (50% of rows) |
| `OR` across different columns | `WHERE a = 1 OR b = 2` (may need two indexes merged) |
| Arithmetic on the column | `WHERE amount + 10 > 100` |
| Stale statistics | optimizer thinks the table is tiny |
| Leading index column missing | see composite index table |

---

## 14.3 Writing Index-Friendly (SARGable) SQL

**SARGable** = "Search ARGument ABLE": a predicate the engine can answer with an index seek. The rule: **keep the column bare on one side of the comparison; put all computation on the other side.**

| Non-SARGable (slow) | SARGable rewrite (fast) |
|---|---|
| `WHERE YEAR(order_date) = 2026` | `WHERE order_date >= '2026-01-01' AND order_date < '2027-01-01'` |
| `WHERE DATE(created_at) = '2026-10-21'` | `WHERE created_at >= '2026-10-21' AND created_at < '2026-10-22'` |
| `WHERE amount + 10 > 100` | `WHERE amount > 90` |
| `WHERE amount * 2 = 1000` | `WHERE amount = 500` |
| `WHERE LOWER(email) = 'a@x.com'` | create an expression index on `LOWER(email)`, or store emails lowercase and compare directly |
| `WHERE name LIKE '%son'` or `'%son%'` | `LIKE 'son%'` can use an index; for "contains" use full-text or trigram (`pg_trgm`) indexes |
| `WHERE phone = 9876543210` (phone is VARCHAR) | `WHERE phone = '9876543210'` (match the column type) |
| `WHERE COALESCE(discount, 0) = 0` | `WHERE discount = 0 OR discount IS NULL` (or store 0 instead of NULL) |
| `WHERE a = 1 OR b = 2` | `SELECT ... WHERE a = 1 UNION ALL SELECT ... WHERE b = 2 AND a <> 1` (when each side has an index) |

### Proof with the sample data

The two queries below return the **same** rows; only the second can use an index on `order_date`:

```sql
-- Non-SARGable
SELECT order_id, amount FROM orders WHERE MONTH(order_date) = 3;

-- SARGable
SELECT order_id, amount FROM orders
WHERE order_date >= '2026-03-01' AND order_date < '2026-04-01';
```

| order_id | amount |
|---|---|
| 7 | 400 |
| 8 | 350 |
| 9 | 150 |
| 10 | 800 |

**Why the half-open range is also safer:** it handles `DATE`, `DATETIME` and `TIMESTAMP` columns correctly (no missing the last second of the day).

### Data types in joins

Joining `INT` to `VARCHAR` (or `INT` to `BIGINT` in some engines) forces **implicit casts** on every row and disables index use. Keep join keys the **same type** and collation on both sides.

---

## 14.4 Join Algorithms

How does the database actually combine two tables? Three classic algorithms. Use these two tiny tables:

`hj_customers`

| id | name |
|---|---|
| 1 | Asha |
| 2 | Ravi |
| 3 | Meena |

`hj_orders`

| order_id | customer_id |
|---|---|
| 101 | 1 |
| 102 | 1 |
| 103 | 2 |
| 104 | 4 |

```sql
SELECT o.order_id, c.name
FROM hj_orders o
JOIN hj_customers c ON c.id = o.customer_id
ORDER BY o.order_id;
```

| order_id | name |
|---|---|
| 101 | Asha |
| 102 | Asha |
| 103 | Ravi |

(Order 104 is dropped: customer 4 does not exist, and this is an `INNER JOIN`.)

### A) Nested Loop Join

**Idea:** for each row of the **outer** table, search the **inner** table for matches.

```text
for each order o:                       (4 orders)
    for each customer c:                 (3 customers)  ->  3 comparisons per order
        if c.id = o.customer_id: output
```

- **Without an index** on the inner table: 4 x 3 = **12 comparisons** (`O(N x M)`: terrible on big tables).
- **With an index on `hj_customers.id`:** each order does one index lookup: 4 lookups (`O(N log M)`).
- **Best when:** the outer side is **small**, or the inner side has an index; typical in OLTP (look up a few rows).

### B) Hash Join

**Idea:** build a **hash table** from the smaller table, then **probe** it with each row of the larger table.

```text
Build phase (customers, the smaller side):
    hash table:  1 -> Asha,   2 -> Ravi,   3 -> Meena

Probe phase (scan orders once):
    101 (customer 1) -> found Asha          -> output (101, Asha)
    102 (customer 1) -> found Asha          -> output (102, Asha)
    103 (customer 2) -> found Ravi          -> output (103, Ravi)
    104 (customer 4) -> not found           -> no output (inner join)
```

- Cost: one pass over each table: `O(N + M)`, but the hash table must fit in memory (otherwise it spills to disk in batches).
- **Best when:** joining big tables on **equality** with no useful index; the default choice for analytics.
- Only works for **equality** joins (`=`), not `<` or `BETWEEN`.

### C) Sort-Merge Join

**Idea:** sort both inputs by the join key, then walk both with two pointers.

```text
customers sorted by id:           1, 2, 3
orders sorted by customer_id:     1(101), 1(102), 2(103), 4(104)

pointer walk:
  1 = 1  -> output (101, Asha), (102, Asha)
  2 = 2  -> output (103, Ravi)
  3 < 4  -> advance customers (3 has no orders)
  customers exhausted -> remaining order 104 has no match
```

- Cost: sorting dominates (`O(N log N + M log M)`), but if inputs are **already sorted** (index, or clustered storage) it is just `O(N + M)`.
- **Best when:** both inputs are huge and/or already sorted; works for **range** joins too; **default for big-big joins in Spark** (after a shuffle).

### Summary table

| Algorithm | Needs | Strength | Weakness |
|---|---|---|---|
| Nested loop | index on inner side (ideally) | tiny outer side, selective lookups | explodes on big x big without an index |
| Hash join | memory for the build side | big equality joins | equality only; spills if build side is too big |
| Sort-merge | sorted inputs (or sort cost) | huge inputs, range joins | sort cost |

### Distributed engines (Spark, Hive, BigQuery, Trino)

Data lives on many machines. Joining means getting matching keys onto the **same machine**.

| Strategy | How it works | When it wins |
|---|---|---|
| **Shuffle hash / sort-merge join** | both tables are **shuffled** (repartitioned by join key across the network), then joined locally | both tables big |
| **Broadcast hash join** | the **small table is copied to every worker**; the big table is not moved at all | one side is small (Spark's default threshold is 10 MB, tunable via `spark.sql.autoBroadcastJoinThreshold`) |

```sql
-- Spark SQL: force a broadcast of the small dimension table
SELECT /*+ BROADCAST(d) */ f.order_id, d.region
FROM fact_orders f
JOIN dim_region d ON f.region_id = d.region_id;
```

**Reading it:** a 10 TB fact table joined to a 50 MB dimension should never shuffle the 10 TB. Broadcasting the 50 MB copy is vastly cheaper. Other Spark hints: `SHUFFLE_MERGE`, `SHUFFLE_HASH`.

> **Shuffle is the enemy.** In big data, the cost of a query is dominated by how many bytes cross the network. Look for `Exchange` in the plan.

---

## 14.5 Query Optimization Patterns

### A) Select only the columns you need

```sql
-- Wasteful
SELECT * FROM big_orders WHERE customer_id = 42;
-- Better
SELECT order_id, order_date, amount FROM big_orders WHERE customer_id = 42;
```

- Row stores (MySQL, PostgreSQL) still read whole rows, but you avoid sending extra data and enable **covering indexes**.
- **Columnar stores** (Parquet, BigQuery, Snowflake, Redshift): each column is stored separately, so unused columns are **never read**. In BigQuery you pay per byte scanned, so `SELECT *` literally costs more money.

### B) Filter early and prune before joining

```sql
-- Join first, filter later (the optimizer may fix it, but don't rely on it)
SELECT ...
FROM big_orders o JOIN big_customers c ON ...
WHERE o.order_date >= '2026-10-01';

-- Better: reduce the big side first (and in distributed engines this cuts the shuffle)
WITH recent AS (
    SELECT order_id, customer_id, amount
    FROM big_orders
    WHERE order_date >= '2026-10-01'
)
SELECT ... FROM recent r JOIN big_customers c ON c.customer_id = r.customer_id;
```

### C) Avoid join fan-out: aggregate before joining

**Fan-out** happens when you join on a **non-unique** key: one row on the left matches many rows on the right, so left-side measures are **repeated** and `SUM` double counts.

`fo_orders`

| order_id | customer_id | amount |
|---|---|---|
| 1 | 1 | 500 |
| 2 | 1 | 300 |
| 3 | 2 | 200 |

`fo_items` (order 1 has 2 items, order 2 has 3, order 3 has 1)

| order_id | item |
|---|---|
| 1 | pen |
| 1 | ink |
| 2 | bag |
| 2 | cap |
| 2 | tie |
| 3 | cup |

**Wrong (fan-out):**

```sql
SELECT o.customer_id, SUM(o.amount) AS revenue
FROM fo_orders o
JOIN fo_items i ON i.order_id = o.order_id
GROUP BY o.customer_id
ORDER BY o.customer_id;
```

| customer_id | revenue |
|---|---|
| 1 | 1900 |
| 2 | 200 |

**Why 1900?** Order 1 (500) is repeated for each of its 2 items = 1000; order 2 (300) is repeated for its 3 items = 900; total 1900. The true revenue for customer 1 is 500 + 300 = **800**.

**Right: pre-aggregate the many-side to one row per order, then join:**

```sql
WITH item_counts AS (
    SELECT order_id, COUNT(*) AS items
    FROM fo_items
    GROUP BY order_id
)
SELECT o.customer_id,
       SUM(o.amount)  AS revenue,
       SUM(ic.items)  AS total_items
FROM fo_orders o
LEFT JOIN item_counts ic ON ic.order_id = o.order_id
GROUP BY o.customer_id
ORDER BY o.customer_id;
```

| customer_id | revenue | total_items |
|---|---|---|
| 1 | 800 | 5 |
| 2 | 200 | 1 |

**Explanation:** `item_counts` has exactly one row per order, so the join is one-to-one and `SUM(o.amount)` is correct. This is both a **correctness** and a **performance** fix (fewer rows in the join).

> **Join checklist:** before any join, ask "is the join key unique on each side?" If neither side is unique you get a many-to-many explosion.

### D) Keyset pagination instead of OFFSET

```sql
-- Page 3 with OFFSET: the database must read and throw away 100,000 rows first
SELECT order_id, amount FROM big_orders ORDER BY order_id LIMIT 20 OFFSET 100000;

-- Keyset ("seek") pagination: jump straight to the position using the last id you saw
SELECT order_id, amount FROM big_orders
WHERE order_id > 100000        -- the last order_id from the previous page
ORDER BY order_id
LIMIT 20;
```

With the small `orders` table, "page 2" of size 3 after the last seen `order_id = 5`:

```sql
SELECT order_id, amount
FROM orders
WHERE order_id > 5
ORDER BY order_id
LIMIT 3;
```

| order_id | amount |
|---|---|
| 6 | 650 |
| 7 | 400 |
| 8 | 350 |

**Why it is fast:** the primary-key index seeks directly to `order_id > 5`; the cost does not grow with page depth. `OFFSET n` costs `O(n)`. (For non-unique sort columns, use a tuple: `WHERE (order_date, order_id) > (?, ?)`.)

### E) UNION ALL instead of UNION (when duplicates do not matter or cannot occur)

`UNION` removes duplicates, which requires a sort or hash over **all** rows.

| Query | Result rows |
|---|---|
| `SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 3 UNION ALL SELECT 4` | 5 rows (1, 2, 3, 3, 4) |
| `SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 3 UNION SELECT 4` | 4 rows (1, 2, 3, 4) |

Use `UNION ALL` by default; use `UNION` only when you truly need de-duplication.

### F) Other common rewrites

| Slow pattern | Better |
|---|---|
| Correlated subquery per row | pre-aggregate in a CTE and join, or use a window function |
| `SELECT DISTINCT` used to hide a bad join | fix the join (see fan-out) |
| `NOT IN (subquery)` with possible NULLs | `NOT EXISTS` (correct and usually faster) |
| `ORDER BY` on a huge result without `LIMIT` | add `LIMIT`, or remove the sort |
| Self-join to get the "latest row per group" | `ROW_NUMBER()` window function |
| `COUNT(DISTINCT x)` over billions of rows | approximate distinct (see 14.10), or pre-aggregate |
| Large single `UPDATE`/`DELETE` | batch it (for example 10,000 rows at a time) to avoid long locks and huge logs |
| Many small queries in an application loop (N+1) | one set-based query with a join or `IN` |
| Repeated heavy subquery logic | materialize once (temp table or CTE that your engine materializes) |
| Wide `OR` chains | `IN (...)` or a join to a small lookup table |

### G) Statistics

The optimizer chooses plans from **statistics** (row counts, distinct counts, histograms). Stale stats lead to bad plans.

| Database | Command |
|---|---|
| PostgreSQL | `ANALYZE table_name;` |
| MySQL | `ANALYZE TABLE table_name;` |
| SQL Server | `UPDATE STATISTICS table_name;` |
| Spark / Hive | `ANALYZE TABLE t COMPUTE STATISTICS;` (and `FOR COLUMNS ...`) |

---

# PART 3: Big Data Layout

## 14.6 Partitioning, Bucketing, Clustering and Sort Keys

### Partitioning

**Partitioning** physically splits a table into chunks by a column's value (usually a **date**). A query that filters on the partition column reads only the matching chunks: **partition pruning**.

```sql
-- Hive / Spark SQL
CREATE TABLE sales_events (
    order_id    BIGINT,
    customer_id BIGINT,
    amount      DOUBLE
)
USING parquet
PARTITIONED BY (order_month STRING);
```

```sql
-- PostgreSQL declarative partitioning
CREATE TABLE orders_p (
    order_id INT, customer_id INT, order_date DATE, amount INT
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_p_2026_03 PARTITION OF orders_p
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
```

```sql
-- BigQuery
CREATE TABLE sales.orders (...) PARTITION BY DATE(order_ts) CLUSTER BY customer_id;
```

**Pruning illustration** using the sample `orders` split by month (rows per partition: Jan 3, Feb 3, Mar 4, Apr 2; total 12):

| Query | Partitions read | Rows read |
|---|---|---|
| `WHERE order_date >= '2026-03-01' AND order_date < '2026-04-01'` | March only | 4 of 12 (33%) |
| `WHERE order_date = '2026-03-15'` | March only | 4 of 12 |
| `WHERE order_date BETWEEN '2026-02-15' AND '2026-03-10'` | February and March | 7 of 12 |
| `WHERE MONTH(order_date) = 3` | **all four** (function on the partition column defeats pruning in most engines) | 12 of 12 |
| `WHERE customer_id = 1` | **all four** (not the partition column) | 12 of 12 |

**Good partition keys:**
- Low or moderate cardinality (dozens to thousands of values): date, region.
- Almost always used in `WHERE` filters.
- Roughly even sizes.

**Bad partition keys:**
- High cardinality like `user_id` creates millions of tiny partitions (**small file problem**: huge metadata overhead, slow listing, many tiny tasks).
- A column rarely filtered on.
- Over-partitioning (for example by hour for a small table).

**Target file size:** for data lakes, aim for files of roughly **128 MB to 1 GB** each. Millions of KB-sized files kill performance; compact them regularly.

### Bucketing (hash partitioning inside partitions)

**Bucketing** hashes a column into a fixed number of buckets (`hash(customer_id) MOD n`). If two tables are bucketed on the **same key and bucket count**, a join between them can match bucket k with bucket k directly, with **no shuffle**.

```sql
-- Spark SQL
CREATE TABLE orders_b (order_id BIGINT, customer_id BIGINT, amount DOUBLE)
USING parquet
CLUSTERED BY (customer_id) INTO 8 BUCKETS;

-- Hive
CREATE TABLE orders_b (order_id BIGINT, customer_id BIGINT, amount DOUBLE)
PARTITIONED BY (order_month STRING)
CLUSTERED BY (customer_id) INTO 8 BUCKETS
STORED AS ORC;
```

| | Partitioning | Bucketing |
|---|---|---|
| Splits by | exact **value** (one folder per value) | **hash** of value into a fixed number of files |
| Best for | filtering (pruning) | joins and aggregations on the bucket key; sampling |
| Cardinality | low to medium | any (high is fine) |

### Clustering, sort keys and distribution keys (warehouses)

| System | Feature | What it does |
|---|---|---|
| BigQuery | `CLUSTER BY col` | sorts data within partitions so filters on `col` skip blocks |
| Snowflake | micro-partitions (automatic) plus optional **clustering key** | prunes micro-partitions using min/max metadata |
| Redshift | `SORTKEY` and `DISTKEY` | sort order for range filters; distribution key controls which node stores each row (co-located joins) |
| Delta Lake / Databricks | `OPTIMIZE ... ZORDER BY (col)` | co-locates related values in files |
| Hive / Spark | `SORT BY`, `bucketed + sorted` tables | local sorting within files |

**Idea behind all of them:** keep rows with similar values **physically together**, so that min/max statistics let the engine **skip** whole blocks (see 14.7).

### Idempotent partition loads

```sql
-- Hive / Spark: rewrite one partition completely; re-running gives the same result
INSERT OVERWRITE TABLE sales_events PARTITION (order_month = '2026-03')
SELECT order_id, customer_id, amount
FROM staging_orders
WHERE order_date >= '2026-03-01' AND order_date < '2026-04-01';
```

Overwriting a whole partition is the standard way to make batch loads **idempotent** (safe to retry).

---

## 14.7 Columnar Formats (Parquet, ORC) and Predicate Pushdown

### Row store vs column store

Table:

| order_id | customer_id | amount |
|---|---|---|
| 1 | 1 | 500 |
| 2 | 2 | 300 |
| 3 | 3 | 450 |
| 4 | 1 | 700 |

```text
ROW layout (OLTP, e.g. MySQL/PostgreSQL pages):
   [1,1,500] [2,2,300] [3,3,450] [4,1,700]
   -> reading one whole row is fast; SUM(amount) still walks past every column

COLUMN layout (OLAP, e.g. Parquet/ORC/warehouses):
   order_id:    [1, 2, 3, 4]
   customer_id: [1, 2, 3, 1]
   amount:      [500, 300, 450, 700]
   -> SUM(amount) reads ONLY the amount column
```

**Benefits of columnar storage:**
- **Column pruning:** read only needed columns.
- **Better compression:** similar values together (for example run-length and dictionary encoding of `customer_id`).
- **Vectorized processing** and **statistics per block** (min/max).

**Cost:** reading or writing a **single full row** is slower, so columnar is for analytics (OLAP), not row-by-row transactions (OLTP).

### Predicate pushdown with row-group statistics

Parquet files are divided into **row groups**; each stores **min and max** per column. The engine reads the stats first and **skips** row groups that cannot contain matches.

Suppose the 12 `orders` are written to Parquet **in `order_id` order**, 4 rows per row group:

| Row group | Orders | amounts | min(amount) | max(amount) |
|---|---|---|---|---|
| RG1 | 1 to 4 | 500, 300, 450, 700 | 300 | 700 |
| RG2 | 5 to 8 | 200, 650, 400, 350 | 200 | 650 |
| RG3 | 9 to 12 | 150, 800, 600, 250 | 150 | 800 |

**Query:** `WHERE amount > 700`

- RG1: max is 700, not greater than 700, so **skip**.
- RG2: max 650, **skip**.
- RG3: max 800, must **read**.
- **Result: 1 of 3 row groups read**, returning order 10 (800).

**Now sort the data by `amount` before writing** (this is what clustering/sorting does):

| Row group | amounts | min | max |
|---|---|---|---|
| RG1 | 150, 200, 250, 300 | 150 | 300 |
| RG2 | 350, 400, 450, 500 | 350 | 500 |
| RG3 | 600, 650, 700, 800 | 600 | 800 |

**Query:** `WHERE amount BETWEEN 350 AND 500`

| Layout | Row groups read |
|---|---|
| Unsorted (ranges [300,700], [200,650], [150,800] all overlap 350 to 500) | **3 of 3** |
| Sorted by amount (only RG2 [350,500] overlaps) | **1 of 3** |

**Result in both cases:** 4 rows (orders 1, 3, 7, 8 with amounts 500, 450, 400, 350). Sorting changes only **how much data is read**, not the answer. That is the entire point of sort keys, Z-ORDER and clustering.

### File formats overview

| Format | Type | Notes |
|---|---|---|
| CSV / JSON | row, text | easy, but slow and large; no schema or stats |
| Avro | row, binary | good for streaming and schema evolution |
| **Parquet** | columnar | the default for data lakes; excellent compression and stats |
| **ORC** | columnar | Hive's native columnar format; similar benefits |
| **Delta Lake / Iceberg / Hudi** | table formats **on top of** Parquet files | add **ACID transactions, MERGE/UPDATE/DELETE, time travel, schema evolution** to a data lake |

---

## 14.8 Data Skew and Mitigation

### What is skew?

In a distributed engine, rows with the same key go to the **same worker** (for `GROUP BY`, joins, window functions). If one key has far more rows than others (a **hot key**: a bot user, a `NULL` key, a "default" id, a mega-customer), one worker gets most of the work while the rest sit idle. The job's runtime equals the **slowest task**.

### Detecting skew

```sql
SELECT user_id,
       COUNT(*) AS views,
       ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct_of_rows
FROM page_views
GROUP BY user_id
ORDER BY views DESC;
```

| user_id | views | pct_of_rows |
|---|---|---|
| u1 | 8 | 80.0 |
| u2 | 1 | 10.0 |
| u3 | 1 | 10.0 |

**Reading it:** `u1` holds 80% of the rows. `GROUP BY user_id` would send 8 of 10 rows to one task. Other signs: in the Spark UI one stage has a few tasks taking much longer than the median; "executor lost / out of memory" on a single task.

### Fix 1: Salting (two-stage aggregation)

Add a **salt** (a small random or deterministic number) to the key so the hot key is split into several sub-keys; aggregate on `(key, salt)` first, then combine.

To keep our example exact, we use `view_id MOD 4` as the salt (in real jobs use a random salt such as `FLOOR(RAND() * N)`).

**Stage 1: partial counts per (user, salt):**

```sql
SELECT user_id,
       MOD(view_id, 4) AS salt,
       COUNT(*)        AS partial_cnt
FROM page_views
GROUP BY user_id, MOD(view_id, 4)
ORDER BY user_id, salt;
```

| user_id | salt | partial_cnt |
|---|---|---|
| u1 | 0 | 2 |
| u1 | 1 | 2 |
| u1 | 2 | 2 |
| u1 | 3 | 2 |
| u2 | 1 | 1 |
| u3 | 2 | 1 |

**Stage 2: combine the partials:**

```sql
WITH stage1 AS (
    SELECT user_id, MOD(view_id, 4) AS salt, COUNT(*) AS partial_cnt
    FROM page_views
    GROUP BY user_id, MOD(view_id, 4)
)
SELECT user_id, SUM(partial_cnt) AS views
FROM stage1
GROUP BY user_id
ORDER BY user_id;
```

| user_id | views |
|---|---|
| u1 | 8 |
| u2 | 1 |
| u3 | 1 |

**Result:** the same answer as the direct `GROUP BY`. In stage 1 the largest group is **2 rows** instead of **8**, so no single task is overloaded; stage 2 handles only the small partial results.

**Limits:** salting works for **decomposable** aggregates (`COUNT`, `SUM`, `MIN`, `MAX`, and `AVG` via `SUM/COUNT`). Exact `COUNT(DISTINCT x)` is not decomposable this way.

### Fix 2: Salted join (when the skew is in a join key)

Skewed big table `big` joined to a smaller table `dim` on `key`:

```sql
-- Spark SQL sketch; salts is a small table with salt = 0..7
SELECT b.*, d.attr
FROM (SELECT *, CAST(FLOOR(RAND() * 8) AS INT) AS salt FROM big) b
JOIN (SELECT d.*, s.salt
      FROM dim d CROSS JOIN salts s) d
  ON b.key = d.key AND b.salt = d.salt;
```

**Idea:** split the hot key across 8 sub-keys on the big side, and **replicate** each dim row 8 times (once per salt) so every sub-key finds its match. Cost: the dim table grows 8x (fine if it is small).

### Other skew fixes

| Fix | When |
|---|---|
| **Broadcast join** | one side is small enough to broadcast: no shuffle, no skew on the join key |
| **Adaptive Query Execution (AQE) skew join** (Spark 3: `spark.sql.adaptive.enabled`, `spark.sql.adaptive.skewJoin.enabled`) | Spark automatically splits oversized partitions at runtime |
| **Handle hot keys separately** | `UNION ALL` of (hot keys handled with a broadcast join) and (everything else with a normal join) |
| **Filter `NULL` keys before joining** | all `NULL` keys land in one partition and never match anyway in an inner join |
| **Pre-aggregate before the join** | reduces rows per key |
| **Use a more granular key** | for example group by `(user_id, hour)` then roll up |
| **Increase shuffle partitions** | helps general imbalance but not a single dominant key |

### Skew in window functions

`ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY ts)` sends **all** of a user's rows to one task. A bot with 500 million events overwhelms it. Fixes: filter or cap hot keys, pre-aggregate, or add a secondary partition column where the logic allows it (for example partition by `(user_id, event_date)`).

---

## 14.9 Hive and Spark SQL Specifics

### ORDER BY vs SORT BY vs DISTRIBUTE BY vs CLUSTER BY

| Clause | What it does |
|---|---|
| `ORDER BY` | **global** total order (one reducer: slow on big data) |
| `SORT BY` | sorts **within each reducer/partition** only (no global order) |
| `DISTRIBUTE BY col` | decides **which reducer** each row goes to (rows with the same `col` go together) |
| `CLUSTER BY col` | shorthand for `DISTRIBUTE BY col SORT BY col` |

**Illustration:** with 2 reducers and `DISTRIBUTE BY customer_id`, suppose the (illustrative) hash is `customer_id MOD 2`. For the sample orders:

| Partition | Customers | Orders (row count) |
|---|---|---|
| 1 | 1, 3, 5 | customer 1: 4 orders, customer 3: 2, customer 5: 1 = **7 rows** |
| 0 | 2, 4, 6 | customer 2: 2 orders, customer 4: 2, customer 6: 1 = **5 rows** |

All rows of one customer sit in the same partition, so per-customer logic (windows, sessionization, dedup) can run locally. `SORT BY order_date` inside each partition then gives per-customer chronological order without a global sort.

```sql
SELECT customer_id, order_id, order_date, amount
FROM orders
DISTRIBUTE BY customer_id
SORT BY customer_id, order_date;
```

(Real engines use a hash function, not `MOD`, so the actual split differs; the principle is identical.)

### LATERAL VIEW EXPLODE, POSEXPLODE

```sql
SELECT emp_id, name, skill
FROM employees_skills
LATERAL VIEW explode(split(skills, ',')) t AS skill;

-- keep rows whose array is empty or NULL
LATERAL VIEW OUTER explode(...)
```

`posexplode` also returns the position. (See Phase 6, P53.)

### COLLECT_LIST vs COLLECT_SET

Using `page_views` for user `u1`'s pages in `view_id` order (`home, home, cart, home, cart, pay, home, cart`):

```sql
SELECT user_id,
       collect_list(page) AS pages_list,
       collect_set(page)  AS pages_set,
       size(collect_list(page)) AS list_size,
       size(collect_set(page))  AS set_size
FROM page_views
GROUP BY user_id;
```

| user_id | pages_list | pages_set | list_size | set_size |
|---|---|---|---|---|
| u1 | [home, home, cart, home, cart, pay, home, cart] | [home, cart, pay] (order not guaranteed) | 8 | 3 |
| u2 | [home] | [home] | 1 | 1 |
| u3 | [cart] | [cart] | 1 | 1 |

- `collect_list` keeps **duplicates** and (without sorting) has no guaranteed order.
- `collect_set` removes duplicates; order not guaranteed.
- These pull a whole group into **one row's memory**: dangerous for hot keys (skew plus out-of-memory). Prefer counts and aggregates when you can.

### Spark tuning essentials

| Topic | Key points |
|---|---|
| **Shuffle partitions** | `spark.sql.shuffle.partitions` (default 200). Too few = big tasks and spills; too many = tiny tasks and overhead. AQE can coalesce them automatically. |
| **AQE** | `spark.sql.adaptive.enabled` re-plans at runtime: coalesces partitions, switches to broadcast joins, splits skewed partitions. |
| **Broadcast** | `spark.sql.autoBroadcastJoinThreshold` (default 10 MB) or the `BROADCAST` hint. |
| **repartition vs coalesce** | `repartition(n)` shuffles to exactly n balanced partitions; `coalesce(n)` only merges partitions without a shuffle (can only reduce). |
| **Caching** | `CACHE TABLE t` helps when a result is reused several times; do not cache everything. |
| **Small files** | many tiny files slow reads; compact (`OPTIMIZE` in Delta, `coalesce` before write, or rewrite). |
| **Statistics** | `ANALYZE TABLE ... COMPUTE STATISTICS` improves join strategy choice. |
| **File format** | Parquet or Delta with partition pruning and predicate pushdown. |
| **Avoid UDFs** | built-in functions are optimized; Python UDFs are slow (serialization). |

### Hive notes

| Topic | Note |
|---|---|
| Managed vs external tables | dropping a managed table deletes the data; dropping an external table keeps the files |
| Partition discovery | `MSCK REPAIR TABLE t` registers partition folders created outside Hive |
| ACID tables | Hive supports transactional ORC tables (`UPDATE/DELETE/MERGE`) |
| Execution engines | MapReduce (legacy), Tez (faster), or Spark |

---

## 14.10 Approximate Functions and Sampling

### Why approximate?

Exact `COUNT(DISTINCT user_id)` over billions of rows must remember **every** distinct value (or shuffle them all): huge memory and network cost. Many business questions only need "about 5.2 million, plus or minus 2%".

| Function | Where | Note |
|---|---|---|
| `APPROX_COUNT_DISTINCT(col)` | Spark, BigQuery, Snowflake, Redshift, Presto/Trino (`approx_distinct`) | HyperLogLog, typical error about 1 to 3% |
| `PERCENTILE_APPROX(col, 0.95)` | Spark, Hive | approximate percentiles with a bounded error |
| `APPROX_QUANTILES` | BigQuery | approximate quantiles |
| `TABLESAMPLE (1 PERCENT)` | Spark, PostgreSQL (`TABLESAMPLE SYSTEM (1)`), Hive | read a random subset |

```sql
-- Spark SQL
SELECT APPROX_COUNT_DISTINCT(user_id)  AS approx_users,
       COUNT(DISTINCT user_id)         AS exact_users
FROM events;
```

### HyperLogLog intuition

Hash every value to a random-looking bit string. Rare patterns reveal how many items you have seen: if the **longest run of leading zeros** among the hashes is 10, you have probably seen about 2^10 = 1,024 distinct values (like inferring many coin flips from the longest streak of heads). HyperLogLog averages many such estimates over many small buckets to reduce the error.

**Why it is powerful:**
- Uses only **kilobytes** of memory regardless of the number of distinct values.
- Sketches are **mergeable**: store one HLL sketch per day, then **union the sketches** to get distinct users for any week or month without rescanning raw data. This solves "rolling distinct users" cheaply (compare with Phase 5, P39).

### Sampling

- **Random sample:** quick exploration. `TABLESAMPLE` takes a random subset of pages or rows (not always uniformly random).
- **Deterministic sample by key** (consistent across queries and joins): keep users whose hash falls in a bucket:

```sql
-- MySQL
SELECT * FROM events WHERE MOD(CRC32(user_id), 100) < 1;     -- about 1% of users
-- Spark SQL
SELECT * FROM events WHERE pmod(hash(user_id), 100) < 1;
```

Sampling by **user** keeps all events of the sampled users, so funnels and sessions stay intact.

> **Bloom filters** (another probabilistic structure) answer "is this key possibly present?" with no false negatives; engines use them to skip files or filter join inputs early.

---

# PART 4: Warehousing and Pipelines

## 14.11 OLTP vs OLAP, Fact and Dimension Tables, Star vs Snowflake

### OLTP vs OLAP

| | OLTP (transactions) | OLAP (analytics) |
|---|---|---|
| Typical queries | read or write a few rows by key | scan and aggregate millions of rows |
| Workload | many small, concurrent | fewer, heavy |
| Design | normalized (3NF) | star schema, denormalized, wide tables |
| Storage | row store, B-tree indexes | column store, partitions, clustering |
| Engines | MySQL, PostgreSQL, SQL Server | Snowflake, BigQuery, Redshift, Spark, Hive, ClickHouse |
| Priority | low latency, correctness (ACID) | throughput, scan efficiency |

### Fact and dimension tables

| | Fact table | Dimension table |
|---|---|---|
| Holds | **events / measurements** (sales, clicks) with numeric measures | **descriptive context** (who, what, where, when) |
| Size | very large, grows fast | smaller, changes slowly |
| Keys | foreign keys to dimensions | primary (surrogate) key |
| Examples | `fact_sales` (qty, amount) | `dim_product`, `dim_date`, `dim_customer` |

**Grain:** the exact meaning of one fact row (here: **one sale line**). Always decide the grain first.

**Measure types:** *additive* (can be summed across all dimensions: amount), *semi-additive* (can be summed across some, like account balance across customers but not across time), *non-additive* (ratios and percentages: recompute from components).

### Star schema example

```text
                 dim_date
                    |
dim_product --- fact_sales --- (dim_customer ...)
```

`fact_sales`

| sale_id | date_key | product_key | qty | amount |
|---|---|---|---|---|
| 1 | 20261001 | 1 | 1 | 1200 |
| 2 | 20261001 | 2 | 5 | 100 |
| 3 | 20261002 | 3 | 1 | 300 |
| 4 | 20261101 | 1 | 1 | 600 |
| 5 | 20261101 | 3 | 2 | 500 |

`dim_product`: (1, Laptop, Electronics), (2, Mouse, Electronics), (3, Desk, Furniture).
`dim_date`: (20261001, 2026-10-01, 2026-10), (20261002, 2026-10-02, 2026-10), (20261101, 2026-11-01, 2026-11).

**Revenue by month and category (one join per dimension, then group):**

```sql
SELECT d.year_month,
       p.category,
       SUM(f.amount) AS revenue
FROM fact_sales f
JOIN dim_date    d ON d.date_key    = f.date_key
JOIN dim_product p ON p.product_key = f.product_key
GROUP BY d.year_month, p.category
ORDER BY d.year_month, p.category;
```

| year_month | category | revenue |
|---|---|---|
| 2026-10 | Electronics | 1300 |
| 2026-10 | Furniture | 300 |
| 2026-11 | Electronics | 600 |
| 2026-11 | Furniture | 500 |

**Step-by-step:** October electronics = sale 1 (1200, Laptop) + sale 2 (100, Mouse) = 1300. October furniture = sale 3 (300, Desk). November electronics = sale 4 (600). November furniture = sale 5 (500).

**Why star schemas are fast:** the big fact table joins to small dimensions on integer keys; filters on dimensions (for example `category = 'Electronics'`) are cheap; the fact table is scanned once.

### Snowflake schema

A **snowflake schema** further normalizes dimensions: `dim_product` gets `category_key` and a separate `dim_category` table.

```sql
-- extra join to reach the category name
SELECT d.year_month, c.category_name, SUM(f.amount) AS revenue
FROM fact_sales f
JOIN dim_date     d ON d.date_key     = f.date_key
JOIN dim_product  p ON p.product_key  = f.product_key
JOIN dim_category c ON c.category_key = p.category_key
GROUP BY d.year_month, c.category_name;
```

| | Star | Snowflake |
|---|---|---|
| Dimensions | denormalized (flat) | normalized (split into sub-dimensions) |
| Joins per query | fewer | more |
| Storage | slightly more redundancy | less redundancy |
| Query speed and simplicity | better | slightly worse |
| Typical choice | **preferred in warehouses** | when dimension maintenance or storage matters |

### Normalization vs denormalization

| | Normalized | Denormalized |
|---|---|---|
| Redundancy | minimal | intentional |
| Writes | simple, consistent | updates touch many copies |
| Reads | many joins | few or no joins |
| Use for | OLTP | OLAP, reporting |

In modern columnar warehouses, **wide denormalized tables** ("one big table") are common because storage is cheap and joins are expensive; compression hides the redundancy.

Other warehouse vocabulary: **surrogate key** (artificial integer key), **conformed dimension** (the same dimension shared by several fact tables, like `dim_date`), **degenerate dimension** (an identifier like an order number stored in the fact table), **junk dimension** (several low-cardinality flags combined into one dimension).

---

## 14.12 Slowly Changing Dimensions (SCD)

**Problem:** dimension attributes change over time (a customer moves city). Do reports show the **current** value or the value **at the time of the event**?

| Type | Strategy | History kept? |
|---|---|---|
| **Type 0** | never change (fixed) | n/a |
| **Type 1** | **overwrite** the old value | No |
| **Type 2** | **add a new row** per version (with validity dates) | Full history |
| **Type 3** | add a **column** for the previous value | Only one prior value |
| **Type 4** | keep history in a separate history table | Yes |
| **Type 6** | hybrid of 1, 2 and 3 | Yes, with current value on every row |

Scenario: customer 7 (Asha) moves from Mumbai to Pune on **2026-06-01**.

### Type 1: overwrite

| customer_id | name | city |
|---|---|---|
| 7 | Asha | Pune |

History is lost: every past sale now looks like a Pune sale.

### Type 3: previous-value column

| customer_id | name | city | previous_city |
|---|---|---|---|
| 7 | Asha | Pune | Mumbai |

Only one change of history; a second move overwrites `previous_city`.

### Type 2: the standard for history

Each version is a **row** with a **validity range** and a flag. Fact rows join to the version that was valid on the **fact's date**. The sample `dim_customer_scd2` starts like this (before the load):

| sk | customer_id | name | city | valid_from | valid_to | is_current |
|---|---|---|---|---|---|---|
| 1 | 7 | Asha | Mumbai | 2025-01-01 | 9999-12-31 | 1 |
| 2 | 8 | Ravi | Delhi | 2025-01-01 | 9999-12-31 | 1 |

Today's snapshot from the source (`stg_customer`):

| customer_id | name | city | effective_date |
|---|---|---|---|
| 7 | Asha | Pune | 2026-06-01 |
| 8 | Ravi | Delhi | 2026-06-01 |
| 9 | Meera | Chennai | 2026-06-01 |

Customer 7 **changed**, 8 is **unchanged**, 9 is **new**.

**Convention used:** `valid_from` is **inclusive**, `valid_to` is **exclusive**, and `9999-12-31` marks the current row. (Half-open ranges avoid gaps and overlaps, as in Phase 5.)

#### Step 1: close the current row of changed customers

```sql
-- MySQL
UPDATE dim_customer_scd2 d
JOIN stg_customer s ON s.customer_id = d.customer_id
SET d.valid_to = s.effective_date,
    d.is_current = 0
WHERE d.is_current = 1
  AND (d.name <> s.name OR d.city <> s.city);
-- PostgreSQL: UPDATE dim_customer_scd2 d SET valid_to = s.effective_date, is_current = 0
--             FROM stg_customer s WHERE s.customer_id = d.customer_id AND d.is_current = 1
--               AND (d.name <> s.name OR d.city <> s.city);
```

Only Asha's row (sk 1) changes: `valid_to = 2026-06-01`, `is_current = 0`. Ravi is identical, so untouched.

#### Step 2: insert new current rows for customers that now have no current row

```sql
INSERT INTO dim_customer_scd2 (customer_id, name, city, valid_from, valid_to, is_current)
SELECT s.customer_id, s.name, s.city, s.effective_date, '9999-12-31', 1
FROM stg_customer s
LEFT JOIN dim_customer_scd2 d
       ON d.customer_id = s.customer_id AND d.is_current = 1
WHERE d.customer_id IS NULL
ORDER BY s.customer_id;
```

**Why it works:** after step 1, Asha has **no current row**, and Meera never had one; the anti-join (`LEFT JOIN ... IS NULL`) picks exactly those two. Ravi still has a current row, so he is skipped.

> **NULL-safe comparison:** `d.city <> s.city` returns unknown when either side is NULL, so a change **to or from NULL** would be missed. Use `NOT (d.city <=> s.city)` in MySQL or `d.city IS DISTINCT FROM s.city` in PostgreSQL.

#### Result after the load

```sql
SELECT sk, customer_id, name, city, valid_from, valid_to, is_current
FROM dim_customer_scd2
ORDER BY customer_id, valid_from;
```

| sk | customer_id | name | city | valid_from | valid_to | is_current |
|---|---|---|---|---|---|---|
| 1 | 7 | Asha | Mumbai | 2025-01-01 | 2026-06-01 | 0 |
| 3 | 7 | Asha | Pune | 2026-06-01 | 9999-12-31 | 1 |
| 2 | 8 | Ravi | Delhi | 2025-01-01 | 9999-12-31 | 1 |
| 4 | 9 | Meera | Chennai | 2026-06-01 | 9999-12-31 | 1 |

(`sk` 3 and 4 are assigned in insertion order.)

### Querying SCD2

**1) Current state:** `WHERE is_current = 1` (or `valid_to = '9999-12-31'`).

**2) "As of" a date:**

```sql
SELECT customer_id, name, city
FROM dim_customer_scd2
WHERE '2026-03-15' >= valid_from
  AND '2026-03-15' <  valid_to
ORDER BY customer_id;
```

| customer_id | name | city |
|---|---|---|
| 7 | Asha | Mumbai |
| 8 | Ravi | Delhi |

(Meera did not exist yet on 2026-03-15: her row starts 2026-06-01.)

**3) Join facts to the version valid at the sale date (point-in-time correct):**

`fact_cust_sales`

| sale_id | customer_id | sale_date | amount |
|---|---|---|---|
| 1 | 7 | 2026-03-15 | 100 |
| 2 | 7 | 2026-07-10 | 200 |
| 3 | 8 | 2026-07-11 | 50 |
| 4 | 9 | 2026-06-02 | 75 |

```sql
SELECT f.sale_id, f.sale_date, d.city, f.amount
FROM fact_cust_sales f
JOIN dim_customer_scd2 d
  ON d.customer_id = f.customer_id
 AND f.sale_date >= d.valid_from
 AND f.sale_date <  d.valid_to
ORDER BY f.sale_id;
```

| sale_id | sale_date | city | amount |
|---|---|---|---|
| 1 | 2026-03-15 | Mumbai | 100 |
| 2 | 2026-07-10 | Pune | 200 |
| 3 | 2026-07-11 | Delhi | 50 |
| 4 | 2026-06-02 | Chennai | 75 |

**Revenue by city, "as it was at sale time" (SCD2) vs "current city" (SCD1 view):**

| City | SCD2 (as-was) | Current-city view |
|---|---|---|
| Mumbai | 100 | 0 |
| Pune | 200 | 300 (Asha's 100 + 200 both attributed to Pune) |
| Delhi | 50 | 50 |
| Chennai | 75 | 75 |

Total is 425 in both views; the **attribution** differs. Choose based on the business question ("where was the customer when they bought?" vs "where do our customers live now?").

**4) Data-quality check: ranges must never overlap (reuse the overlap test from Phase 5):**

```sql
SELECT a.customer_id, a.sk AS sk_a, b.sk AS sk_b
FROM dim_customer_scd2 a
JOIN dim_customer_scd2 b
  ON a.customer_id = b.customer_id
 AND a.sk < b.sk
 AND a.valid_from < b.valid_to
 AND a.valid_to   > b.valid_from;
```

| customer_id | sk_a | sk_b |
|---|---|---|
| (no rows) | | |

A healthy SCD2 table returns **no rows**. Also check "exactly one current row per customer" with `GROUP BY customer_id HAVING SUM(is_current) <> 1`.

### Late-arriving dimensions and timing

If a sale for customer 9 arrives **before** the dimension row exists, an `INNER JOIN` silently **drops revenue**. Before the load, a `LEFT JOIN` with `COALESCE` keeps it:

```sql
SELECT COALESCE(d.city, 'Unknown') AS city, SUM(f.amount) AS revenue
FROM fact_cust_sales f
LEFT JOIN dim_customer_scd2 d
  ON d.customer_id = f.customer_id
 AND f.sale_date >= d.valid_from
 AND f.sale_date <  d.valid_to
GROUP BY COALESCE(d.city, 'Unknown')
ORDER BY city;
```

Output **if run before the nightly load** (dimension rows: Asha = Mumbai still open-ended, Ravi, no Meera):

| city | revenue |
|---|---|
| Delhi | 50 |
| Mumbai | 300 |
| Unknown | 75 |

(Asha's July sale is wrongly shown as Mumbai because the Pune version did not exist yet.) After the load the same query returns Chennai 75, Delhi 50, Mumbai 100, Pune 200. **Lesson:** keep an **"Unknown" member** (a dimension row with key -1) and plan for **inferred members** (placeholder dimension rows created when a fact arrives early, filled in later). Never drop facts silently.

---

## 14.13 Incremental Loads, Idempotent Pipelines, Late Data and Deduplication

### Full vs incremental loads

| | Full load | Incremental load |
|---|---|---|
| What | reload everything each run | load only new or changed rows |
| Pros | simple, self-healing | fast, cheap at scale |
| Cons | slow and costly for big data | needs change tracking; risk of missing data |

**Change tracking options:** an `updated_at` **watermark** (high-water mark), an auto-increment id, CDC (change data capture) logs, or partition-by-date loads.

### The watermark pattern and its traps

```sql
-- Typical incremental extract
SELECT *
FROM source_table
WHERE updated_at >= :last_watermark;       -- note >= and the lookback below
```

**Traps:**
1. **Equal timestamps:** `>` can skip rows that share the watermark timestamp; use `>=` and rely on idempotent merges to handle repeats.
2. **Late-arriving data:** records with an *older* `updated_at` can show up after you have moved the watermark. Fix: re-read a **lookback window** (for example the last 3 days) every run.
3. **Deletes** are invisible to watermark queries; use soft-delete flags or CDC.
4. **Clock skew** between systems: use database time or an ingestion timestamp.

### Deduplicating a change feed (latest version per key)

The feed `src_changes` has many versions per key:

| id | val | updated_at |
|---|---|---|
| 1 | a | 2026-10-20 10:00 |
| 2 | b | 2026-10-20 11:00 |
| 1 | a2 | 2026-10-21 09:00 |
| 3 | c | 2026-10-21 09:30 |
| 2 | b2 | 2026-10-21 10:00 |
| 3 | c2 | 2026-10-21 10:30 |

The target already holds batch 1 (`target_t`): `(1, a, 10-20 10:00)` and `(2, b, 10-20 11:00)`.

**Batch 2 = changes on or after 2026-10-21, deduplicated to the latest version per id:**

```sql
SELECT id, val, updated_at
FROM (
    SELECT id, val, updated_at,
           ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) AS rn
    FROM src_changes
    WHERE updated_at >= '2026-10-21 00:00:00'
) t
WHERE rn = 1
ORDER BY id;
```

| id | val | updated_at |
|---|---|---|
| 1 | a2 | 2026-10-21 09:00:00 |
| 2 | b2 | 2026-10-21 10:00:00 |
| 3 | c2 | 2026-10-21 10:30:00 |

**Explanation:** id 3 has two versions in the batch (`c` at 09:30 and `c2` at 10:30); `ROW_NUMBER` keeps only the newest (`c2`). Without deduplication, the merge would see two source rows for one key (ambiguous, and an error in standard `MERGE`).

### Merge into the target (upsert, last-write-wins)

```sql
-- MySQL (guards against older late rows overwriting newer data)
INSERT INTO target_t (id, val, updated_at)
SELECT id, val, updated_at
FROM (
    SELECT id, val, updated_at,
           ROW_NUMBER() OVER (PARTITION BY id ORDER BY updated_at DESC) AS rn
    FROM src_changes
    WHERE updated_at >= '2026-10-21 00:00:00'
) t
WHERE rn = 1
ON DUPLICATE KEY UPDATE
    val        = IF(VALUES(updated_at) > updated_at, VALUES(val), val),   -- uses the OLD updated_at (assigned first)
    updated_at = GREATEST(updated_at, VALUES(updated_at));
```

**Standard MERGE version (SQL Server, Oracle, PostgreSQL 15+, Snowflake, Delta):**

```sql
MERGE INTO target_t t
USING (SELECT id, val, updated_at FROM latest_batch) s     -- latest_batch = the deduplicated query above
ON t.id = s.id
WHEN MATCHED AND s.updated_at > t.updated_at THEN
    UPDATE SET val = s.val, updated_at = s.updated_at
WHEN NOT MATCHED THEN
    INSERT (id, val, updated_at) VALUES (s.id, s.val, s.updated_at);
```

`target_t` after batch 2:

| id | val | updated_at |
|---|---|---|
| 1 | a2 | 2026-10-21 09:00:00 |
| 2 | b2 | 2026-10-21 10:00:00 |
| 3 | c2 | 2026-10-21 10:30:00 |

### Idempotency: safe to re-run

An **idempotent** job gives the **same final result** no matter how many times it runs on the same input (essential, because jobs retry after failures).

| Approach | Run once | Run twice |
|---|---|---|
| Append-only `INSERT ... SELECT` | 2 + 3 = 5 rows | 2 + 3 + 3 = **8 rows** (duplicate ids!) |
| `MERGE` / upsert keyed on `id` with last-write-wins | 3 rows | **3 rows** (same final state) |
| `INSERT OVERWRITE` of a whole partition | same data | **same data** |

**Pipeline design rules:**
- Have a **natural or business key**, and merge on it.
- Prefer **overwrite-by-partition** or **MERGE** over blind appends.
- Deduplicate on an event id inside the batch and against the target.
- Make steps **atomic** (write to a staging table, then swap or commit once).
- Track **batch ids and watermarks** in a control table.
- Validate after loading with quick **data quality checks**:

```sql
-- duplicate keys?
SELECT id, COUNT(*) AS n FROM target_t GROUP BY id HAVING COUNT(*) > 1;     -- expect no rows
-- row counts match?
SELECT COUNT(*) FROM target_t;                                              -- expect 3
-- NULLs in required columns?
SELECT COUNT(*) FROM target_t WHERE val IS NULL;                            -- expect 0
```

**Delivery guarantees:** *at-most-once* (may lose data), *at-least-once* (may duplicate), *exactly-once* (the goal). In practice you get **at-least-once delivery plus idempotent processing = effectively exactly-once**.

---

# PART 5: Transactions

## 14.14 ACID, MVCC and Locks

| Letter | Property | Meaning | How databases do it |
|---|---|---|---|
| **A** | Atomicity | all steps happen or none | write-ahead log (undo/rollback) |
| **C** | Consistency | constraints always hold | constraints, triggers, application rules |
| **I** | Isolation | concurrent transactions do not see each other's half-done work | locks and/or **MVCC** |
| **D** | Durability | committed data survives a crash | write-ahead log flushed to disk, replication |

**MVCC (Multi-Version Concurrency Control):** the database keeps **multiple versions** of each row. Readers see a consistent **snapshot** and do **not** block writers (and vice versa). PostgreSQL and InnoDB (MySQL) both use MVCC.

**Locks:**
- **Shared (S)** lock: many readers allowed, no writers.
- **Exclusive (X)** lock: one writer, no one else.
- **Row locks** vs **table locks**; `SELECT ... FOR UPDATE` takes exclusive row locks; `SELECT ... FOR SHARE` takes shared locks.

## 14.15 Isolation Levels and Anomalies

### The anomalies

Use `accounts`: `(id = 1, balance = 100)`.

**1) Dirty read:** reading **uncommitted** data.

| Step | Transaction 1 | Transaction 2 |
|---|---|---|
| 1 | `UPDATE accounts SET balance = 50 WHERE id = 1;` (not committed) | |
| 2 | | `SELECT balance FROM accounts WHERE id = 1;` returns **50** (dirty) |
| 3 | `ROLLBACK;` (balance is really 100 again) | T2 acted on a value that never existed |

**2) Non-repeatable read:** the same row read twice gives **different values**.

| Step | Transaction 1 | Transaction 2 |
|---|---|---|
| 1 | `SELECT balance` returns **100** | |
| 2 | | `UPDATE accounts SET balance = 150 WHERE id = 1; COMMIT;` |
| 3 | `SELECT balance` returns **150** (changed mid-transaction) | |

**3) Phantom read:** the same **query** returns a **different set of rows**.

| Step | Transaction 1 | Transaction 2 |
|---|---|---|
| 1 | `SELECT COUNT(*) FROM accounts WHERE balance > 50;` returns **1** | |
| 2 | | `INSERT INTO accounts VALUES (2, 200); COMMIT;` |
| 3 | `SELECT COUNT(*) ... WHERE balance > 50;` returns **2** (a phantom row appeared) | |

**4) Lost update:** two transactions read the same value, both write based on it, one overwrites the other.

| Step | Transaction 1 | Transaction 2 |
|---|---|---|
| 1 | reads balance = 100 | reads balance = 100 |
| 2 | computes 100 + 10 = 110 | computes 100 + 10 = 110 |
| 3 | writes 110, commits | writes 110, commits |
| Result | balance = **110** | expected **120**: one +10 is **lost** |

**Fixes for lost updates:**

```sql
-- 1) atomic update (best): the database does read-modify-write under a lock
UPDATE accounts SET balance = balance + 10 WHERE id = 1;

-- 2) pessimistic lock
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;   -- others wait
UPDATE accounts SET balance = 110 WHERE id = 1;
COMMIT;

-- 3) optimistic concurrency: add a version column and check it
UPDATE accounts SET balance = 110, version = version + 1
WHERE id = 1 AND version = 7;                            -- 0 rows updated means someone else changed it: retry
```

### Isolation levels (SQL standard)

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| **Read Uncommitted** | possible | possible | possible |
| **Read Committed** | prevented | possible | possible |
| **Repeatable Read** | prevented | prevented | possible (by the standard) |
| **Serializable** | prevented | prevented | prevented |

**Real databases differ from the textbook:**

| Database | Default level | Notes |
|---|---|---|
| PostgreSQL | Read Committed | its Repeatable Read is snapshot isolation (also prevents phantoms); Serializable uses SSI |
| MySQL (InnoDB) | Repeatable Read | uses MVCC plus gap/next-key locks to limit phantoms |
| SQL Server | Read Committed | optional `READ_COMMITTED_SNAPSHOT` / snapshot isolation |
| Oracle | Read Committed | Serializable is snapshot-based |

```sql
-- set the level (MySQL)
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- PostgreSQL
BEGIN ISOLATION LEVEL REPEATABLE READ;
```

> **Trade-off:** higher isolation = fewer anomalies but more blocking, aborts or retries. Most systems use the default and fix specific hot spots with atomic updates or explicit locks.

**Write skew** (a subtler anomaly): two transactions each check a rule using data the **other** will change (for example, "at least one doctor must be on call": both read two doctors on call, both go off call, and zero remain). Snapshot isolation allows it; **Serializable** (or explicit locks) prevents it.

### Job-queue pattern with SKIP LOCKED

Several workers must pick **different** pending jobs without waiting for each other:

```sql
START TRANSACTION;
SELECT job_id
FROM jobs
WHERE status = 'new'
ORDER BY job_id
LIMIT 1
FOR UPDATE SKIP LOCKED;          -- MySQL 8.0+, PostgreSQL 9.5+
UPDATE jobs SET status = 'running' WHERE job_id = :picked_id;
COMMIT;
```

`SKIP LOCKED` ignores rows another worker has locked, so workers do not block each other.

## 14.16 Deadlocks

A **deadlock** happens when two transactions each hold a lock the other needs, so neither can continue.

| Step | Transaction 1 | Transaction 2 |
|---|---|---|
| 1 | `UPDATE accounts SET balance = balance - 10 WHERE id = 1;` (locks row 1) | |
| 2 | | `UPDATE accounts SET balance = balance - 10 WHERE id = 2;` (locks row 2) |
| 3 | `UPDATE accounts SET balance = balance + 10 WHERE id = 2;` **waits** for T2 | |
| 4 | | `UPDATE accounts SET balance = balance + 10 WHERE id = 1;` **waits** for T1, a cycle |
| 5 | **The database detects the cycle, aborts one transaction (the "victim") and rolls it back** | |

Typical errors: MySQL `ERROR 1213: Deadlock found when trying to get lock; try restarting transaction`; PostgreSQL `ERROR: deadlock detected` (SQLSTATE `40P01`).

**How to prevent and handle deadlocks:**

| Technique | Why it works |
|---|---|
| **Access rows in a consistent order** (for example always lock the lower id first) | removes the circular wait |
| **Keep transactions short** and do not wait for user input inside one | locks are held for less time |
| **Lock up front** (`SELECT ... FOR UPDATE` in a fixed order) | acquire all locks before doing work |
| **Index the columns used in `WHERE` of updates** | without an index, the engine may lock many more rows than needed |
| **Use a lower isolation level** where acceptable | fewer locks held |
| **Retry the victim transaction** in application code | deadlocks are normal in busy systems; handle them gracefully |

---

# PART 6: Interview Playbook (Q&A Outlines)

## Q1. "This query is slow. What do you do?"

1. **Reproduce and measure** (baseline timing, data volume).
2. **Look at the plan** (`EXPLAIN ANALYZE`): find the node with the highest actual time; check scans, joins, sorts, estimated vs actual rows.
3. **Check predicates:** SARGable? type mismatches? leading wildcards?
4. **Reduce data:** needed columns only; filter early; pre-aggregate; avoid fan-out joins.
5. **Physical design:** add or adjust an index (composite order, covering); partitioning or clustering for big tables.
6. **Statistics** up to date?
7. **Concurrency:** blocked by locks? long transactions?
8. **Distributed engine:** check for shuffles, skew (task durations), small files, broadcast opportunities, partition pruning.
9. **Verify** results are identical after each change; re-measure.

## Q2. "How would you deduplicate a table with a billion rows?"

- First **define a duplicate** (same business key? entire row? keep newest or oldest?).
- **Do not delete in place.** Build a new table: `CREATE TABLE new AS SELECT ... FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY key ORDER BY updated_at DESC) rn FROM old) WHERE rn = 1`, then swap tables atomically. In Spark, `dropDuplicates(keys)` after ordering, or window functions.
- Do it **per partition** (for example per day) to limit shuffle size and make it restartable.
- Watch for **skew** (a hot key with millions of duplicates): pre-aggregate or salt.
- **Prevent recurrence:** unique constraint where possible, `MERGE` on the key in the ingestion job, idempotent loads.
- Validate: compare counts, `GROUP BY key HAVING COUNT(*) > 1` returns no rows.

## Q3. "How do you handle a skewed join in Spark?"

1. **Detect:** key distribution query (top keys by count), Spark UI task times.
2. If one side is small: **broadcast join**.
3. Enable **AQE skew join handling**.
4. **Isolate hot keys:** handle them separately (broadcast) and union with the normal join.
5. **Salt** the key (and replicate the small side).
6. Filter `NULL` or junk keys; pre-aggregate before joining.

## Q4. "Model a customer dimension so we can report history correctly."

- **SCD Type 2** with surrogate key, `valid_from`, `valid_to` (half-open), `is_current`.
- Load: close changed rows, insert new versions (two-step or `MERGE`), NULL-safe change detection.
- Query: point-in-time join with `fact_date >= valid_from AND fact_date < valid_to`.
- Quality checks: no overlapping ranges, exactly one current row per key; an "Unknown" member for late-arriving dimensions.

## Q5. "Count distinct users per day for a year, efficiently?"

- Exact: pre-aggregate daily distinct users into a small table (`day, user_id`) once, then query it (columnar, partitioned by day).
- For rolling windows or huge scale: **HyperLogLog sketches** per day (`APPROX_COUNT_DISTINCT` and mergeable sketches).
- Explain the accuracy trade-off (about 1 to 3% error).

## Q6. "Design storage for 10 billion event rows queried by date and user."

- Store as **Parquet/ORC** (or Delta/Iceberg) in a data lake or warehouse.
- **Partition by event date** (low cardinality, always filtered).
- **Cluster/sort/Z-order by `user_id`**; **bucket** by `user_id` if joining with a user table often.
- Control file sizes (128 MB to 1 GB); compaction job; retention policy.
- Queries must filter on the partition column; select only needed columns.

## Q7. "How do you build a pipeline that is safe to retry?"

- Idempotent writes: **MERGE on a key** or **overwrite a partition**; never blind append.
- Deterministic transformations; watermark with a **lookback window**; dedupe by event id.
- Atomic publish (staging then swap), batch/control table, data-quality checks, alerting.

## Q8. "Explain isolation levels and how to avoid lost updates."

- State the four levels and the anomalies each prevents (table in 14.15).
- Give the defaults of the databases you know.
- Lost update fixes: atomic `UPDATE x = x + 1`, `SELECT ... FOR UPDATE`, optimistic version column, higher isolation with retry.

## Q9. "Why is `OFFSET 1000000` slow, and what is the alternative?"

- The database must produce and discard a million rows; cost grows with depth.
- **Keyset pagination:** `WHERE (sort_key) > :last_seen ORDER BY sort_key LIMIT n`, using an index; stable and fast.

## Q10. "OLTP vs OLAP, row vs column?"

- Table in 14.11 plus 14.7: row stores for point lookups and updates; column stores for scans and aggregates with column pruning, compression and stats-based skipping.

---

# PART 7: Wrap-Up

## 7.1 Top Mistakes in Performance and Big Data SQL

| Mistake | Fix |
|---|---|
| `SELECT *` everywhere | name the columns; enables pruning and covering indexes |
| Function or arithmetic on an indexed column | rewrite the predicate to be SARGable |
| Comparing different data types in a join or filter | match types; avoid implicit casts |
| Indexing every column | index for real query patterns; indexes slow writes |
| Wrong composite index order | equality columns first, then range/sort |
| Trusting a plan without checking actual rows | use `EXPLAIN ANALYZE`; fix stale statistics |
| Joining on a non-unique key and summing | pre-aggregate; check fan-out |
| `OFFSET` pagination on deep pages | keyset pagination |
| `UNION` when `UNION ALL` suffices | `UNION ALL` |
| Partitioning by a high-cardinality column | partition by date/region; cluster or bucket the rest |
| Filtering on a function of the partition column | filter the raw partition column |
| Millions of tiny files | compact to 128 MB to 1 GB files |
| Ignoring skew | detect with a key-count query; salt, broadcast, or AQE |
| `collect_list` on hot keys | aggregate instead, or cap |
| Overwriting history in a dimension that needs it | SCD Type 2 |
| Dropping facts with an inner join to a dimension | LEFT JOIN plus an "Unknown" member |
| Append-only loads that duplicate on retry | MERGE or partition overwrite |
| Watermark with `>` and no lookback | `>=` plus a lookback window and idempotent merge |
| Long transactions and inconsistent lock order | short transactions, consistent ordering, retries |

## 7.2 Phase 7 Cheat Sheet

```text
PLAN        EXPLAIN / EXPLAIN ANALYZE | red flags: Seq Scan on big table, big Sort spill,
            estimated rows far from actual, Exchange (shuffle) everywhere
MySQL type  system > const > eq_ref > ref > range > index > ALL   | Extra: Using index = covering

INDEX       B-tree: =, range, ORDER BY, 'abc%' | leftmost prefix | equality first, then range/sort
            covering index -> no table access | selectivity < ~10% to be useful
SARGABLE    keep the column bare: col >= x AND col < y   (not YEAR(col) = ...)
            match types | avoid leading wildcard | functional index if a function is unavoidable

JOINS       nested loop (small outer + indexed inner) | hash (big equality, memory) | merge (sorted/huge, range)
            Spark: broadcast (small side) vs shuffle sort-merge (big-big)
OPTIMIZE    fewer columns | filter early | aggregate before join | UNION ALL | keyset pagination
            NOT EXISTS | window instead of self-join | batch big writes | update statistics

PARTITION   by low-cardinality filtered column (date) -> pruning | do not wrap it in functions
BUCKET      hash(key) into N files -> shuffle-free joins on that key
CLUSTER     keep similar values together so min/max stats skip blocks
PARQUET/ORC columnar: column pruning + compression + row-group min/max predicate pushdown
TABLE FMT   Delta / Iceberg / Hudi = ACID + MERGE + time travel on lake files

SKEW        detect: GROUP BY key COUNT(*) top N | fix: broadcast, AQE, salting, hot-key split, filter NULL keys
HIVE/SPARK  ORDER BY global | SORT BY per reducer | DISTRIBUTE BY routes keys | CLUSTER BY = both
            explode / LATERAL VIEW | collect_list (dups) vs collect_set | AQE | shuffle.partitions
APPROX      APPROX_COUNT_DISTINCT (HyperLogLog ~1-3% error, mergeable) | PERCENTILE_APPROX | TABLESAMPLE

WAREHOUSE   fact (events, measures, grain) + dimensions (context) | star (flat dims) vs snowflake (normalized)
SCD         T1 overwrite | T2 new row (valid_from inclusive, valid_to exclusive, is_current) | T3 previous column
            as-of join: f.date >= valid_from AND f.date < valid_to
PIPELINE    watermark >= + lookback | dedupe ROW_NUMBER latest | MERGE/overwrite = idempotent | Unknown member

ACID        atomicity (undo log) consistency isolation (MVCC/locks) durability (WAL)
ISOLATION   RU: dirty | RC: no dirty | RR: no non-repeatable | SERIALIZABLE: none
            lost update -> atomic UPDATE x = x + 1 | FOR UPDATE | version column
DEADLOCK    consistent lock order, short transactions, index update predicates, retry the victim
```

## 7.3 Practice Problems

1. Rewrite these predicates to be SARGable: (a) `WHERE YEAR(order_date) = 2026 AND MONTH(order_date) = 3`, (b) `WHERE amount * 2 > 1000`, (c) `WHERE DATE(created_at) = '2026-10-21'`.
2. For the query `WHERE customer_id = ? AND order_date >= ? ORDER BY order_date`, design the best composite index. Which column goes first and why?
3. Index on `(a, b, c)`. For each predicate say whether the index is used for seeking: (i) `a = 1`, (ii) `a = 1 AND c = 3`, (iii) `b = 2`, (iv) `a = 1 AND b > 5 AND c = 3`.
4. Using `hj_customers` and `hj_orders`, how many comparisons would a nested loop join (no index) need? Write the output of a `LEFT JOIN` from orders to customers.
5. A Parquet file has 3 row groups with amount ranges [100, 200], [150, 400], [500, 900]. How many row groups are read for `amount > 450`? For `amount BETWEEN 180 AND 190`?
6. Write the query to detect skew in `page_views` and state what percentage the top key holds.
7. Compute the stage-1 salted partial counts for `page_views` using `MOD(view_id, 2)` as the salt (2 salts instead of 4).
8. Using the `fo_orders` / `fo_items` data, write a correct query returning revenue per customer and the number of items per customer.
9. Write the keyset query for the next page of 3 orders after `order_id = 8`.
10. After the SCD2 load, write an "as of 2026-06-15" query for the whole dimension. What rows return?
11. Which isolation anomaly occurs in each case? (a) T1 reads a row, T2 updates it and commits, T1 reads it again and sees the new value. (b) T2 reads a row T1 changed but has not committed, then T1 rolls back. (c) T1 counts rows with a condition, T2 inserts a matching row and commits, T1 counts again.
12. Two transactions deadlock updating rows 1 and 2 in opposite orders. Give two ways to prevent it.
13. Given the change feed in 14.13, what does `target_t` look like if the batch is run **twice** using the MERGE approach? And with a naive append?
14. Choose partition, bucket and sort columns for a 5-billion-row `clicks(user_id, url, click_ts, country)` table that is queried by date range and joined to `users` on `user_id`.
15. Challenge: you need exact distinct users per week for 3 years of events (500 billion rows). Describe two approaches (exact and approximate) and their trade-offs.

<details>
<summary>Answers</summary>

```text
1  (a) WHERE order_date >= '2026-03-01' AND order_date < '2026-04-01'
   (b) WHERE amount > 500
   (c) WHERE created_at >= '2026-10-21' AND created_at < '2026-10-22'

2  CREATE INDEX idx ON orders (customer_id, order_date);
   customer_id first (equality), order_date second (range and sort). The index returns rows already
   sorted by order_date for one customer, so there is no filesort.

3  (i) a = 1                 -> seeks on a
   (ii) a = 1 AND c = 3      -> seeks on a only; c is filtered on index entries (b is skipped)
   (iii) b = 2               -> no seek (leading column missing)
   (iv) a = 1 AND b > 5 AND c = 3 -> seeks on a and the range on b; c cannot narrow the seek (after a range column)

4  Nested loop without an index: 4 orders x 3 customers = 12 comparisons.
   LEFT JOIN from orders to customers (ordered by order_id):
   101 Asha | 102 Asha | 103 Ravi | 104 NULL

5  amount > 450          -> only [500, 900] qualifies: 1 row group read
   amount BETWEEN 180 AND 190 -> [100,200] and [150,400] both overlap 180..190: 2 row groups read
   ([500, 900] is skipped)

6  SELECT user_id, COUNT(*) AS views,
          ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct_of_rows
   FROM page_views GROUP BY user_id ORDER BY views DESC;
   Top key u1 holds 80.0% of the rows.

7  SELECT user_id, MOD(view_id, 2) AS salt, COUNT(*) AS partial_cnt
   FROM page_views GROUP BY user_id, MOD(view_id, 2) ORDER BY user_id, salt;
   u1 salt 0: views 2,4,6,8 -> 4 | u1 salt 1: views 1,3,5,7 -> 4
   u2 (view 9) salt 1 -> 1 | u3 (view 10) salt 0 -> 1
   Largest group drops from 8 to 4. (More salts = smaller pieces.)

8  WITH ic AS (SELECT order_id, COUNT(*) AS items FROM fo_items GROUP BY order_id)
   SELECT o.customer_id, SUM(o.amount) AS revenue, SUM(ic.items) AS total_items
   FROM fo_orders o LEFT JOIN ic ON ic.order_id = o.order_id
   GROUP BY o.customer_id ORDER BY o.customer_id;
   -> customer 1: revenue 800, items 5 | customer 2: revenue 200, items 1

9  SELECT order_id, amount FROM orders WHERE order_id > 8 ORDER BY order_id LIMIT 3;
   -> (9, 150), (10, 800), (11, 600)

10 SELECT customer_id, name, city FROM dim_customer_scd2
   WHERE '2026-06-15' >= valid_from AND '2026-06-15' < valid_to ORDER BY customer_id;
   -> 7 Asha Pune | 8 Ravi Delhi | 9 Meera Chennai

11 (a) non-repeatable read  (b) dirty read  (c) phantom read

12 Access rows in the same order in both transactions (always row 1 then row 2);
   keep transactions short; lock both rows up front with SELECT ... FOR UPDATE in a fixed order;
   index the WHERE columns; retry the victim transaction.

13 MERGE approach, run twice: target_t = (1, a2), (2, b2), (3, c2): 3 rows (second run changes nothing,
   because s.updated_at is not greater than t.updated_at).
   Naive append, run twice: 2 existing + 3 + 3 = 8 rows with duplicate ids.

14 PARTITIONED BY (click_date) (derived from click_ts) -> pruning for date-range queries.
   BUCKET (CLUSTER) BY user_id INTO e.g. 256 buckets -> shuffle-free join with a users table bucketed
   the same way.  SORT within buckets by click_ts (or Z-order by user_id) so min/max stats skip blocks.
   Not partitioning by user_id (too many partitions) or country alone (not the main filter).
   Parquet/ORC, files 128 MB to 1 GB, compaction job.

15 Exact: pre-aggregate distinct (week, user_id) pairs per day or week into a much smaller table
   (dedupe early, partitioned by week), then COUNT(DISTINCT) per week; cost: a heavy shuffle once,
   storage for the pairs; weekly distinct from daily needs the raw pairs again.
   Approximate: APPROX_COUNT_DISTINCT / HyperLogLog sketch per day, union sketches for any week or month;
   error about 1-3%, tiny storage, very fast; cannot list the users or get exact numbers.
```
</details>

## 7.4 Self-Test Questions

1. What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`? Why be careful with the latter on `UPDATE`?
2. What does `type = ALL` mean in a MySQL plan, and what is the goal?
3. What does `Rows Removed by Filter: 999902` tell you?
4. What is the leftmost-prefix rule? Why should equality columns come before range columns in a composite index?
5. What is a covering index and how do you recognize it in a plan?
6. Name five reasons an index is not used.
7. Explain nested loop, hash join and sort-merge join in one sentence each, and when each wins.
8. What is the difference between a shuffle join and a broadcast join?
9. What is join fan-out and how do you avoid double counting?
10. Why is keyset pagination faster than `OFFSET`?
11. Partitioning vs bucketing: what does each do, and when should you use which?
12. Why is partitioning by `user_id` a bad idea? What is the small file problem?
13. How do row groups and min/max statistics let Parquet skip data? How does sorting improve it?
14. What is data skew? Describe three ways to fix it and the limits of salting.
15. `ORDER BY` vs `SORT BY` vs `DISTRIBUTE BY` vs `CLUSTER BY` in Hive/Spark?
16. How does HyperLogLog estimate distinct counts with tiny memory? Why can its sketches be merged?
17. Star vs snowflake schema: differences and trade-offs.
18. Describe SCD Types 1, 2 and 3. How do you join a fact table to a Type 2 dimension correctly?
19. What makes a data load idempotent? Give three techniques.
20. List the four ACID properties and how databases implement them.
21. Describe dirty read, non-repeatable read, phantom read and lost update, and the isolation level that prevents each.
22. What causes a deadlock and how do you prevent it?
23. What does `FOR UPDATE SKIP LOCKED` do and when is it useful?
24. What is MVCC and how does it let readers and writers not block each other?

## 7.5 Recommended Resources

| Resource | Why |
|---|---|
| *Use The Index, Luke* (use-the-index-luke.com) | the best free guide to SQL indexing and execution plans |
| PostgreSQL docs: "Using EXPLAIN" and "Performance Tips" | authoritative plan-reading guide |
| MySQL docs: "Understanding the Query Execution Plan", "Optimization and Indexes" | MySQL-specific |
| *High Performance MySQL* | deep internals and tuning |
| *Designing Data-Intensive Applications* (Martin Kleppmann) | transactions, isolation, storage engines, distributed data |
| *The Data Warehouse Toolkit* (Kimball) | star schemas, fact/dimension design, SCD types |
| Apache Spark docs: "SQL Performance Tuning" | broadcast joins, AQE, skew, partitions |
| Delta Lake / Apache Iceberg docs | table formats, MERGE, time travel, compaction |

**Hands-on practice:** run `EXPLAIN ANALYZE` on the lab data in Part 1 before and after every index and rewrite; try TPC-H style queries on a local PostgreSQL; use Spark locally (PySpark) with a million-row Parquet dataset and inspect `EXPLAIN FORMATTED` plans.

## 7.6 What to Do Next

1. Create the sample tables from Part 1 and run **every** data query; compare with the outputs shown. Then build the optional PostgreSQL lab and watch real plans change as you add indexes.
2. Re-derive by hand: the hash-join walk, the Parquet row-group skipping tables, the salted two-stage aggregation, and the SCD2 load.
3. Practice speaking the **interview playbook** answers (Part 6) aloud in 2 to 3 minutes each.
4. Complete the 15 practice problems without opening the answers.
5. When you are ready, say **"Start Phase 8"** for the final phase: the **dialect cheat sheet** (MySQL, PostgreSQL, SQL Server, Oracle, BigQuery, Snowflake, Spark SQL) and **interview execution skills** (clarifying questions, edge-case checklist, mock interviews). If you still want the earlier gap filled, say **"Start Phase 2"** for **Aggregation, Joins and Set Operations**.
