# SQL Phase 6: Advanced Techniques (Detailed Guide)

> The toolkit that separates "good" from "hard-level ready": LATERAL joins, strings and arrays into rows, JSON, regex parsing, bitwise tricks, UPDATE/DELETE/UPSERT problems, relational division, graph problems, tournament scoreboards, and hard interval problems.
> Covers **Module 13 (Advanced SQL Constructs)** and patterns **P53 to P60**.
> Every topic has: easy explanation, input table, query, output table, and step-by-step reasoning.

---

## Table of Contents

- **Part 0:** Overview, dialect notes, and DML safety rules
- **Part 1:** Sample data (create once)
- **Part 2:** Module 13: Advanced constructs (CASE in aggregates and windows, LATERAL, bitwise, views and temp tables)
- **Part 3:** Interview patterns P53 to P60
- **Part 4:** Wrap-up (mistakes, cheat sheet, practice, self-test, next step)

> **Prerequisites:** Phase 3 (CTEs, recursive CTEs), Phase 4 (window functions), Phase 5 (calendar and date logic). Basic `GROUP BY` and `JOIN` are used throughout.

---

# PART 0: Overview and Conventions

## 0.1 Dialect Notes

Examples use **MySQL 8.0+** syntax, with **PostgreSQL**, **SQL Server**, and **Hive/Spark SQL** alternatives shown where they differ. This phase has the **largest dialect differences** of the whole course: always read the notes.

| Task | MySQL 8 | PostgreSQL | Spark / Hive SQL |
|---|---|---|---|
| Split string to rows | numbers table + `SUBSTRING_INDEX` | `unnest(string_to_array(s, ','))` | `LATERAL VIEW explode(split(s, ','))` |
| JSON field | `payload->>'$.user'` | `payload->>'user'` | `get_json_object(payload, '$.user')` |
| JSON array to rows | `JSON_TABLE(...)` | `jsonb_array_elements_text(...)` | `explode(from_json(...))` |
| Regex extract | `REGEXP_SUBSTR` | `substring(s from 'pattern')` | `regexp_extract(s, pattern, 1)` |
| Top N per group via lateral | `JOIN LATERAL (... LIMIT n) ON TRUE` | same | not supported (use window functions) |
| Upsert | `ON DUPLICATE KEY UPDATE` | `ON CONFLICT ... DO UPDATE` | `MERGE` (Delta/Iceberg tables) |
| String aggregate | `GROUP_CONCAT(x ORDER BY x SEPARATOR ', ')` | `STRING_AGG(x, ', ' ORDER BY x)` | `concat_ws(', ', sort_array(collect_list(x)))` |

## 0.2 DML Safety Rules (UPDATE / DELETE / MERGE)

Data-changing statements are **dangerous in real life** and are favorite interview questions. Always:

1. Write the matching **`SELECT` first** and check which rows it returns.
2. Run inside a **transaction** (`START TRANSACTION; ...; ROLLBACK;` to test, `COMMIT;` when sure).
3. Never run `UPDATE` / `DELETE` without a `WHERE` (unless truly intended).
4. Take a **backup copy** of affected rows (`CREATE TABLE backup AS SELECT ...`).
5. In interviews, say: "I would preview with a SELECT and wrap it in a transaction."

---

# PART 1: Sample Data (Create Once)

```sql
-- 1) departments and employees (same data as Phase 3)
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

-- 2) orders (same data as Phase 5)
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

-- 3) employees_skills (comma-separated list: an anti-pattern we must be able to handle)
CREATE TABLE employees_skills (
    emp_id  INT PRIMARY KEY,
    name    VARCHAR(20),
    skills  VARCHAR(100)
);
INSERT INTO employees_skills VALUES
(1, 'Amit',  'sql,python,spark'),
(2, 'Priya', 'sql,excel'),
(3, 'Rahul', 'python'),
(4, 'Sneha', 'spark,sql,airflow,python');

-- 4) events_json
CREATE TABLE events_json (
    event_id  INT PRIMARY KEY,
    payload   JSON
);
INSERT INTO events_json VALUES
(1, '{"user":"u1","action":"click","device":{"os":"android","version":"13"},"tags":["promo","mobile"],"amount":120.5}'),
(2, '{"user":"u2","action":"purchase","device":{"os":"ios","version":"17"},"tags":["web"],"amount":300}'),
(3, '{"user":"u1","action":"purchase","device":{"os":"android","version":"13"},"tags":[],"amount":99.99}'),
(4, '{"user":"u3","action":"click","device":{"os":"ios","version":"16"},"tags":["promo"]}');

-- 5) access_logs (semi-structured text)
CREATE TABLE access_logs (
    log_id  INT PRIMARY KEY,
    line    VARCHAR(200)
);
INSERT INTO access_logs VALUES
(1, '2026-10-21 10:15:03 INFO user=u1 action=login ip=10.0.0.5'),
(2, '2026-10-21 10:16:10 ERROR user=u2 action=upload ip=10.0.0.7 code=500'),
(3, '2026-10-21 10:17:45 INFO user=u3 action=logout ip=10.0.0.9');

-- 6) signups (email validation)
CREATE TABLE signups (
    signup_id  INT PRIMARY KEY,
    email      VARCHAR(50)
);
INSERT INTO signups VALUES
(1, 'a@x.com'), (2, 'bad.email'), (3, 'b@y.co.in'), (4, '@no.com');

-- 7) user_perms (bit flags: 1 = read, 2 = write, 4 = delete, 8 = admin)
CREATE TABLE user_perms (
    user_id  INT PRIMARY KEY,
    perms    INT
);
INSERT INTO user_perms VALUES (1, 7), (2, 1), (3, 10), (4, 15);

-- 8) person (duplicate emails) and salary_swap
CREATE TABLE person (
    id     INT PRIMARY KEY,
    email  VARCHAR(30)
);
INSERT INTO person VALUES
(1, 'john@example.com'), (2, 'bob@example.com'), (3, 'john@example.com'),
(4, 'alice@example.com'), (5, 'bob@example.com'), (6, 'john@example.com');

CREATE TABLE salary_swap (
    id      INT PRIMARY KEY,
    name    VARCHAR(5),
    sex     CHAR(1),
    salary  INT
);
INSERT INTO salary_swap VALUES
(1,'A','m',2500), (2,'B','f',1500), (3,'C','m',5500), (4,'D','f',500);

-- 9) inventory and incoming (upsert)
CREATE TABLE inventory (
    product_id  INT PRIMARY KEY,
    product     VARCHAR(10),
    stock       INT
);
INSERT INTO inventory VALUES (1,'Pen',100), (2,'Book',50), (3,'Bag',20);

CREATE TABLE incoming (
    product_id  INT PRIMARY KEY,
    product     VARCHAR(10),
    qty         INT
);
INSERT INTO incoming VALUES (2,'Book',30), (3,'Bag',10), (4,'Lamp',25);

-- 10) products and purchases (relational division)
CREATE TABLE products (
    product_id    INT PRIMARY KEY,
    product_name  VARCHAR(10)
);
INSERT INTO products VALUES (1,'Laptop'), (2,'Mouse'), (3,'Keyboard');

CREATE TABLE purchases (
    customer_id  INT,
    product_id   INT
);
INSERT INTO purchases VALUES
(1,1),(1,2),(1,3),
(2,1),(2,2),
(3,1),(3,2),(3,3),(3,3),
(4,3),
(5,1),(5,2),(5,3);

-- 11) people and friendships (graph; each friendship stored once)
CREATE TABLE people (
    id    INT PRIMARY KEY,
    name  VARCHAR(10)
);
INSERT INTO people VALUES
(1,'Asha'), (2,'Ravi'), (3,'Meena'), (4,'Kabir'), (5,'Divya'), (6,'Esha');

CREATE TABLE friendships (
    user_a  INT,
    user_b  INT
);
INSERT INTO friendships VALUES
(1,2), (1,3), (2,3), (2,4), (3,5), (4,5), (5,6);

-- 12) matches (tournament)
CREATE TABLE matches (
    match_id    INT PRIMARY KEY,
    home_team   CHAR(1),
    away_team   CHAR(1),
    home_goals  INT,
    away_goals  INT
);
INSERT INTO matches VALUES
(1,'A','B',2,1), (2,'C','D',0,0), (3,'A','C',1,3),
(4,'B','D',2,2), (5,'A','D',3,0), (6,'B','C',1,2);

-- 13) meetings (intervals within one day)
CREATE TABLE meetings (
    meeting_id  INT PRIMARY KEY,
    start_time  TIME,
    end_time    TIME
);
INSERT INTO meetings VALUES
(1,'09:00:00','10:30:00'),
(2,'10:00:00','11:00:00'),
(3,'10:15:00','10:45:00'),
(4,'11:00:00','12:00:00'),
(5,'13:00:00','14:00:00'),
(6,'13:30:00','15:00:00');
```

### Key tables (for reference)

**`employees_skills`**

| emp_id | name | skills |
|---|---|---|
| 1 | Amit | sql,python,spark |
| 2 | Priya | sql,excel |
| 3 | Rahul | python |
| 4 | Sneha | spark,sql,airflow,python |

**`events_json.payload`** (4 rows)

| event_id | payload |
|---|---|
| 1 | `{"user":"u1","action":"click","device":{"os":"android","version":"13"},"tags":["promo","mobile"],"amount":120.5}` |
| 2 | `{"user":"u2","action":"purchase","device":{"os":"ios","version":"17"},"tags":["web"],"amount":300}` |
| 3 | `{"user":"u1","action":"purchase","device":{"os":"android","version":"13"},"tags":[],"amount":99.99}` |
| 4 | `{"user":"u3","action":"click","device":{"os":"ios","version":"16"},"tags":["promo"]}` (no amount) |

**`purchases`** (customer: products bought): c1: 1, 2, 3 | c2: 1, 2 | c3: 1, 2, 3, 3 (3 twice) | c4: 3 | c5: 1, 2, 3. Products: 1 = Laptop, 2 = Mouse, 3 = Keyboard.

**Friendship graph** (ids: 1 Asha, 2 Ravi, 3 Meena, 4 Kabir, 5 Divya, 6 Esha):

```text
Edges: 1-2, 1-3, 2-3, 2-4, 3-5, 4-5, 5-6

Neighbors:  1: {2,3}   2: {1,3,4}   3: {1,2,5}   4: {2,5}   5: {3,4,6}   6: {5}
```

**`matches`**

| match_id | home_team | away_team | home_goals | away_goals |
|---|---|---|---|---|
| 1 | A | B | 2 | 1 |
| 2 | C | D | 0 | 0 |
| 3 | A | C | 1 | 3 |
| 4 | B | D | 2 | 2 |
| 5 | A | D | 3 | 0 |
| 6 | B | C | 1 | 2 |

**`meetings`**

| meeting_id | start_time | end_time |
|---|---|---|
| 1 | 09:00 | 10:30 |
| 2 | 10:00 | 11:00 |
| 3 | 10:15 | 10:45 |
| 4 | 11:00 | 12:00 |
| 5 | 13:00 | 14:00 |
| 6 | 13:30 | 15:00 |

---

# PART 2: MODULE 13: Advanced SQL Constructs

## 13.1 Complex CASE Inside Aggregates and Windows

`CASE` is the Swiss Army knife of analytics SQL. You already used `SUM(CASE ...)` for conditional counting. Here are the harder combinations.

### A) CASE on top of an aggregate (segmenting groups)

**Problem:** label each customer by total spend: Platinum (2000 or more), Gold (800 or more), Silver (500 or more), otherwise Bronze.

```sql
SELECT customer_id,
       SUM(amount) AS total_spent,
       CASE WHEN SUM(amount) >= 2000 THEN 'Platinum'
            WHEN SUM(amount) >= 800  THEN 'Gold'
            WHEN SUM(amount) >= 500  THEN 'Silver'
            ELSE 'Bronze' END AS tier
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | total_spent | tier |
|---|---|---|
| 1 | 2200 | Platinum |
| 2 | 500 | Silver |
| 3 | 800 | Gold |
| 4 | 800 | Gold |
| 5 | 800 | Gold |
| 6 | 250 | Bronze |

**Explanation:** The `CASE` runs **after** `GROUP BY`, so it can look at `SUM(amount)`. Conditions are checked top to bottom, first match wins (800 is "Gold" because `>= 800` comes before `>= 500`).

**Count customers per tier (CTE then group again):**

```sql
WITH t AS (
    SELECT customer_id,
           CASE WHEN SUM(amount) >= 2000 THEN 'Platinum'
                WHEN SUM(amount) >= 800  THEN 'Gold'
                WHEN SUM(amount) >= 500  THEN 'Silver'
                ELSE 'Bronze' END AS tier
    FROM orders
    GROUP BY customer_id
)
SELECT tier, COUNT(*) AS customers
FROM t
GROUP BY tier
ORDER BY customers DESC, tier;
```

| tier | customers |
|---|---|
| Gold | 3 |
| Bronze | 1 |
| Platinum | 1 |
| Silver | 1 |

### B) Several CASE buckets in one pass (conditional aggregation)

**Problem:** per month, count small (under 300), medium (300 to 599) and large (600 or more) orders.

```sql
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
       SUM(CASE WHEN amount < 300                  THEN 1 ELSE 0 END) AS small_orders,
       SUM(CASE WHEN amount BETWEEN 300 AND 599    THEN 1 ELSE 0 END) AS medium_orders,
       SUM(CASE WHEN amount >= 600                 THEN 1 ELSE 0 END) AS large_orders
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

| month | small_orders | medium_orders | large_orders |
|---|---|---|---|
| 2026-01 | 0 | 3 | 0 |
| 2026-02 | 1 | 0 | 2 |
| 2026-03 | 1 | 2 | 1 |
| 2026-04 | 1 | 0 | 1 |

**Step-by-step (February):** amounts 700 (large), 200 (small), 650 (large) gives small 1, medium 0, large 2.

> **NULL trap:** `SUM(CASE WHEN cond THEN x END)` (no `ELSE`) returns **NULL** when no row matches. Use `ELSE 0` or wrap in `COALESCE(..., 0)`.
> PostgreSQL shortcut: `SUM(amount) FILTER (WHERE amount >= 600)`.

### C) CASE inside a window function

**Problem:** for each customer, a running count of **high-value orders** (amount 500 or more).

```sql
SELECT customer_id, order_id, order_date, amount,
       SUM(CASE WHEN amount >= 500 THEN 1 ELSE 0 END)
           OVER (PARTITION BY customer_id
                 ORDER BY order_date, order_id
                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS high_value_so_far
FROM orders
ORDER BY customer_id, order_date;
```

| customer_id | order_id | order_date | amount | high_value_so_far |
|---|---|---|---|---|
| 1 | 1 | 2026-01-05 | 500 | 1 |
| 1 | 4 | 2026-02-03 | 700 | 2 |
| 1 | 7 | 2026-03-07 | 400 | 2 |
| 1 | 11 | 2026-04-02 | 600 | 3 |
| 2 | 2 | 2026-01-12 | 300 | 0 |
| 2 | 5 | 2026-02-14 | 200 | 0 |
| 3 | 3 | 2026-01-20 | 450 | 0 |
| 3 | 8 | 2026-03-15 | 350 | 0 |
| 4 | 6 | 2026-02-18 | 650 | 1 |
| 4 | 9 | 2026-03-21 | 150 | 1 |
| 5 | 10 | 2026-03-25 | 800 | 1 |
| 6 | 12 | 2026-04-10 | 250 | 0 |

**Reading it:** `CASE` converts each row to 1 or 0 first; the window `SUM` then accumulates those flags per customer. This "flag then running sum" idea powers streaks, sessions and first-seen counts.

### D) Other places CASE is useful

| Place | Example |
|---|---|
| `ORDER BY` | `ORDER BY CASE status WHEN 'urgent' THEN 1 WHEN 'normal' THEN 2 ELSE 3 END` (custom sort order) |
| `GROUP BY` | `GROUP BY CASE WHEN age < 18 THEN 'minor' ELSE 'adult' END` (group by a computed bucket) |
| `UPDATE SET` | `SET sex = CASE WHEN sex = 'm' THEN 'f' ELSE 'm' END` (see P55) |
| Join condition | `ON a.x = CASE WHEN b.type = 1 THEN b.y ELSE b.z END` (use sparingly: it blocks index use) |

---

## 13.2 LATERAL Joins, CROSS APPLY and OUTER APPLY

### The problem

A normal derived table in `FROM` **cannot** refer to columns of tables listed before it. So "for each department, get the top 2 earners" with `LIMIT 2` inside a subquery is impossible with a plain join: the `LIMIT` would apply to the whole table, not per department.

### The solution: LATERAL

`LATERAL` (standard SQL, PostgreSQL, MySQL 8.0.14+) lets a subquery in `FROM` **see columns from tables on its left**. Think of it as a **for-each loop**: for every row on the left, run the subquery with that row's values.

| Dialect | Syntax |
|---|---|
| PostgreSQL, MySQL 8.0.14+ | `JOIN LATERAL (subquery) alias ON TRUE` / `LEFT JOIN LATERAL ... ON TRUE` |
| SQL Server | `CROSS APPLY (subquery)` (like `INNER`) and `OUTER APPLY (subquery)` (like `LEFT`), with `TOP n` |
| Oracle 12c+ | `CROSS APPLY` / `OUTER APPLY` / `LATERAL` |
| Spark / Hive | no `LATERAL` join; use window functions (`LATERAL VIEW` is different: it explodes arrays) |

### Example: top 2 earners per department (departments with no employees kept)

```sql
SELECT d.dept_name, t.name, t.salary
FROM departments d
LEFT JOIN LATERAL (
    SELECT e.name, e.salary
    FROM employees e
    WHERE e.dept_id = d.dept_id          -- refers to the OUTER row d
    ORDER BY e.salary DESC, e.emp_id
    LIMIT 2
) t ON TRUE
ORDER BY d.dept_id, t.salary DESC;
```

| dept_name | name | salary |
|---|---|---|
| IT | Amit Sharma | 90000 |
| IT | Rahul Verma | 68000 |
| HR | Neha Gupta | 58000 |
| HR | Priya Singh | 52000 |
| Finance | Vikram Rao | 95000 |
| Finance | Sneha Iyer | 82000 |
| Sales | Anita Desai | 45000 |
| Marketing | NULL | NULL |

**Step-by-step:**
1. For each department row `d`, the inner query runs with `d.dept_id` filled in.
2. It sorts that department's employees by salary and keeps 2 (`LIMIT 2`). In IT, Rahul and Rohan tie at 68000; `emp_id` breaks the tie, so Rahul wins.
3. `LEFT JOIN ... ON TRUE` keeps departments even when the inner query returns **no rows** (Marketing shows NULLs). With a plain `JOIN LATERAL ... ON TRUE`, Marketing would disappear (`CROSS APPLY` behavior).

**SQL Server equivalent:**

```sql
SELECT d.dept_name, t.name, t.salary
FROM departments d
OUTER APPLY (SELECT TOP 2 e.name, e.salary
             FROM employees e
             WHERE e.dept_id = d.dept_id
             ORDER BY e.salary DESC, e.emp_id) t;
```

### LATERAL vs window function for top N per group

| | `ROW_NUMBER() ... WHERE rn <= N` | `LATERAL ... LIMIT N` |
|---|---|---|
| Works in Spark/Hive/BigQuery | Yes | No |
| Needs scanning all rows then ranking | Yes | Can use an index per group (fast for few groups on a large table) |
| Keeps groups with no rows | Needs an extra join | Natural with `LEFT JOIN LATERAL` |
| Simplicity | Very common, easy | Great when you have a small list of groups and a big detail table |

> **Rule of thumb:** use window functions by default (portable). Use `LATERAL` / `APPLY` when you want "for each X, the top N related rows" and an index on `(group_col, sort_col)` makes it fast, or when you must call a set-returning function per row (for example `unnest`, `JSON_TABLE`).

---

## 13.3 Views, Temp Tables and Materialized Views

| Object | Lifetime | Stores data? | Use for |
|---|---|---|---|
| **CTE** | one statement | no (usually) | readable step-by-step logic |
| **Subquery / derived table** | one clause | no | small one-off logic |
| **Temporary table** | your session | yes (can be indexed) | big intermediate result reused many times |
| **View** | permanent | no (it is a stored query) | reusable logic, security (hide columns/rows) |
| **Materialized view** (PostgreSQL, Oracle, Snowflake, BigQuery) | permanent | yes (snapshot, refreshed on demand or schedule) | expensive aggregates read often |

### View

```sql
CREATE VIEW dept_payroll AS
SELECT d.dept_id, d.dept_name,
       COUNT(e.emp_id)              AS headcount,
       COALESCE(SUM(e.salary), 0)   AS payroll
FROM departments d
LEFT JOIN employees e ON e.dept_id = d.dept_id
GROUP BY d.dept_id, d.dept_name;

SELECT * FROM dept_payroll ORDER BY dept_id;
```

| dept_id | dept_name | headcount | payroll |
|---|---|---|---|
| 10 | IT | 4 | 286000 |
| 20 | HR | 2 | 110000 |
| 30 | Finance | 3 | 259000 |
| 40 | Sales | 1 | 45000 |
| 50 | Marketing | 0 | 0 |

**Explanation:** the view stores the **query**, not the rows; every `SELECT` on it re-runs the query on fresh data. Useful for hiding salary details from some users (grant access to the view, not the table).

### Temporary table

```sql
CREATE TEMPORARY TABLE tmp_high_earners AS
SELECT emp_id, name, salary
FROM employees
WHERE salary >= 80000;

SELECT * FROM tmp_high_earners ORDER BY emp_id;
```

| emp_id | name | salary |
|---|---|---|
| 1 | Amit Sharma | 90000 |
| 4 | Sneha Iyer | 82000 |
| 7 | Vikram Rao | 95000 |
| 10 | Meera Nair | 82000 |

The table disappears when the session ends. Unlike a CTE, it can be indexed and reused by many later queries.

### Materialized view (PostgreSQL)

```sql
CREATE MATERIALIZED VIEW mv_monthly_revenue AS
SELECT DATE_TRUNC('month', order_date) AS month_start, SUM(amount) AS revenue
FROM orders
GROUP BY 1;

REFRESH MATERIALIZED VIEW mv_monthly_revenue;   -- recompute when data changes
```

MySQL has no native materialized views: emulate with a summary table refreshed by a scheduled job.

---

## 13.4 Bitwise Operations and Number Tricks

Bit flags pack several yes/no settings into one integer. Each permission is a power of two.

| Bit value | Meaning |
|---|---|
| 1 | read |
| 2 | write |
| 4 | delete |
| 8 | admin |

**Operators:** `&` (AND: test a bit), `|` (OR: set a bit), `^` (XOR), `~` (NOT), `<<` and `>>` (shifts).

```sql
SELECT user_id, perms,
       (perms & 1) > 0 AS can_read,
       (perms & 2) > 0 AS can_write,
       (perms & 4) > 0 AS can_delete,
       (perms & 8) > 0 AS is_admin,
       BIT_COUNT(perms) AS permission_count
FROM user_perms
ORDER BY user_id;
```

| user_id | perms | can_read | can_write | can_delete | is_admin | permission_count |
|---|---|---|---|---|---|---|
| 1 | 7 | 1 | 1 | 1 | 0 | 3 |
| 2 | 1 | 1 | 0 | 0 | 0 | 1 |
| 3 | 10 | 0 | 1 | 0 | 1 | 2 |
| 4 | 15 | 1 | 1 | 1 | 1 | 4 |

**Step-by-step (user 3, perms = 10):** in binary `1010` = 8 + 2, so write and admin are on. `10 & 2 = 2` (greater than 0, so can write), `10 & 1 = 0` (cannot read). `BIT_COUNT(10) = 2` counts the 1-bits.

(PostgreSQL: booleans print as `t`/`f`; `BIT_COUNT` exists from version 14 on `bit`/`bytea` values, so cast first or use `get_bit`.)

**Grant and revoke a permission:**

```sql
UPDATE user_perms SET perms = perms | 4  WHERE user_id = 2;   -- grant delete: 1 becomes 5
UPDATE user_perms SET perms = perms & ~4 WHERE user_id = 1;   -- revoke delete: 7 becomes 3
```

**Handy number tricks:**

| Trick | Expression | Example |
|---|---|---|
| Even / odd | `n % 2 = 0` or `MOD(n, 2) = 0` | 10 is even |
| Power of two | `n > 0 AND (n & (n - 1)) = 0` | 8 yes (1000 & 0111 = 0), 6 no |
| Find the number that appears once (all others twice) | `BIT_XOR(n)` over all values (a ^ a = 0) | below |

```sql
WITH t(n) AS (SELECT 4 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 1 UNION ALL SELECT 2)
SELECT BIT_XOR(n) AS single_number FROM t;
```

| single_number |
|---|
| 4 |

**Explanation:** `4 ^ 1 ^ 2 ^ 1 ^ 2`: pairs cancel (`1 ^ 1 = 0`, `2 ^ 2 = 0`), leaving 4. In SQL you would normally write `GROUP BY n HAVING COUNT(*) = 1`, but XOR is a nice interview trick. (`BIT_XOR` exists in MySQL and PostgreSQL 14+.)

---

# PART 3: Interview Patterns P53 to P60

---

## P53: Exploding Delimited / Array Columns Into Rows

**Use when:** a column stores a list (`'sql,python,spark'`), and you must count, filter or join on the individual items. This violates **1NF** (one value per cell), but you meet it constantly in logs and legacy tables.

### MySQL: numbers table + SUBSTRING_INDEX

**Idea:** build numbers 1, 2, 3, ...; for each row, keep numbers up to the item count; extract the n-th item.

- Item count = `1 + (number of commas)` = `1 + LENGTH(s) - LENGTH(REPLACE(s, ',', ''))`.
- n-th item = `SUBSTRING_INDEX(SUBSTRING_INDEX(s, ',', n), ',', -1)` (take the first n items, then the last of those).

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 10        -- upper bound: at least the max items per row
)
SELECT s.emp_id, s.name,
       nums.n AS skill_pos,
       TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(s.skills, ',', nums.n), ',', -1)) AS skill
FROM employees_skills s
JOIN nums
  ON nums.n <= 1 + LENGTH(s.skills) - LENGTH(REPLACE(s.skills, ',', ''))
ORDER BY s.emp_id, nums.n;
```

| emp_id | name | skill_pos | skill |
|---|---|---|---|
| 1 | Amit | 1 | sql |
| 1 | Amit | 2 | python |
| 1 | Amit | 3 | spark |
| 2 | Priya | 1 | sql |
| 2 | Priya | 2 | excel |
| 3 | Rahul | 1 | python |
| 4 | Sneha | 1 | spark |
| 4 | Sneha | 2 | sql |
| 4 | Sneha | 3 | airflow |
| 4 | Sneha | 4 | python |

**Step-by-step for Amit (`sql,python,spark`):**
- Commas = 2, so item count = 3: the join keeps n = 1, 2, 3.
- n = 2: `SUBSTRING_INDEX(s, ',', 2)` = `sql,python`; then `SUBSTRING_INDEX(..., ',', -1)` = `python`.
- `TRIM` removes spaces if the data looks like `'sql, python'`.

### PostgreSQL

```sql
SELECT s.emp_id, s.name, t.pos AS skill_pos, t.skill
FROM employees_skills s
CROSS JOIN LATERAL unnest(string_to_array(s.skills, ',')) WITH ORDINALITY AS t(skill, pos)
ORDER BY s.emp_id, t.pos;
```

### Spark / Hive

```sql
SELECT emp_id, name, pos + 1 AS skill_pos, skill
FROM employees_skills
LATERAL VIEW posexplode(split(skills, ',')) t AS pos, skill;
```

(`LATERAL VIEW explode(...)` is Hive/Spark's way to turn an array into rows. Use `OUTER` to keep rows whose array is empty.)

### Now analyze the exploded data

Wrap the exploding query in a CTE named `skills_long (emp_id, name, skill)`:

**Skill popularity with the list of people (collapse rows back into a list):**

```sql
SELECT skill,
       COUNT(*) AS people,
       GROUP_CONCAT(name ORDER BY name SEPARATOR ', ') AS who
FROM skills_long
GROUP BY skill
ORDER BY people DESC, skill;
```

| skill | people | who |
|---|---|---|
| python | 3 | Amit, Rahul, Sneha |
| sql | 3 | Amit, Priya, Sneha |
| spark | 2 | Amit, Sneha |
| airflow | 1 | Sneha |
| excel | 1 | Priya |

(PostgreSQL: `STRING_AGG(name, ', ' ORDER BY name)`.)

**People who know BOTH sql and python:**

```sql
SELECT emp_id, name
FROM skills_long
WHERE skill IN ('sql', 'python')
GROUP BY emp_id, name
HAVING COUNT(DISTINCT skill) = 2
ORDER BY emp_id;
```

| emp_id | name |
|---|---|
| 1 | Amit |
| 4 | Sneha |

**Explanation:** Priya knows only sql and Rahul only python, so each has just 1 distinct matching skill. Only Amit and Sneha reach 2. (This is **relational division**: see P56.)

> **Quick filter without exploding (MySQL):** `WHERE FIND_IN_SET('sql', skills) > 0`. It works, but cannot use an index and does not extend to counting; also beware that `'sql'` could match inside another word with `LIKE '%sql%'` (e.g., `'mysql'`). Prefer a proper **junction table** (`emp_id, skill`) in real designs.

---

## P54: Parsing JSON and Semi-Structured Logs

### A) JSON fields

JSON paths: `$.user` (top-level key), `$.device.os` (nested), `$.tags[0]` (array element), `$.tags[*]` (all elements).

| Operator / function | MySQL | PostgreSQL (`jsonb`) |
|---|---|---|
| Extract as JSON | `JSON_EXTRACT(p, '$.user')` or `p->'$.user'` | `p->'user'` |
| Extract as text | `p->>'$.user'` (unquoted) | `p->>'user'` |
| Nested | `p->>'$.device.os'` | `p->'device'->>'os'` |
| Cast to number | `CAST(p->>'$.amount' AS DECIMAL(10,2))` | `(p->>'amount')::numeric` |
| Array to rows | `JSON_TABLE(...)` | `jsonb_array_elements_text(p->'tags')` |

```sql
SELECT event_id,
       payload->>'$.user'                                AS user_id,
       payload->>'$.action'                              AS action,
       payload->>'$.device.os'                           AS os,
       CAST(payload->>'$.amount' AS DECIMAL(10,2))       AS amount
FROM events_json
ORDER BY event_id;
```

| event_id | user_id | action | os | amount |
|---|---|---|---|---|
| 1 | u1 | click | android | 120.50 |
| 2 | u2 | purchase | ios | 300.00 |
| 3 | u1 | purchase | android | 99.99 |
| 4 | u3 | click | ios | NULL |

**Explanation:**
- `->>` returns **text** without quotes (`->` would return `"u1"` with JSON quotes).
- Event 4 has no `amount` key: a missing path gives **NULL** (not an error).
- Cast extracted text before doing arithmetic.

**Revenue by operating system (purchases only):**

```sql
SELECT payload->>'$.device.os' AS os,
       SUM(CAST(payload->>'$.amount' AS DECIMAL(10,2))) AS revenue,
       COUNT(*) AS purchases
FROM events_json
WHERE payload->>'$.action' = 'purchase'
GROUP BY payload->>'$.device.os'
ORDER BY os;
```

| os | revenue | purchases |
|---|---|---|
| android | 99.99 | 1 |
| ios | 300.00 | 1 |

**Exploding a JSON array (tags) into rows, keeping events with no tags:**

MySQL:

```sql
SELECT e.event_id, t.tag
FROM events_json e
LEFT JOIN JSON_TABLE(e.payload, '$.tags[*]'
                     COLUMNS (tag VARCHAR(20) PATH '$')) AS t ON TRUE
ORDER BY e.event_id, t.tag;
```

PostgreSQL:

```sql
SELECT e.event_id, t.tag
FROM events_json e
LEFT JOIN LATERAL jsonb_array_elements_text(e.payload->'tags') AS t(tag) ON TRUE
ORDER BY e.event_id, t.tag;
```

| event_id | tag |
|---|---|
| 1 | mobile |
| 1 | promo |
| 2 | web |
| 3 | NULL |
| 4 | promo |

**Explanation:** event 3 has an **empty** array, so the lateral/table function produces no rows; `LEFT JOIN ... ON TRUE` keeps the event with NULL. An inner join would drop event 3.

**Spark:** `get_json_object(payload, '$.device.os')`, or parse with `from_json(payload, schema)` and `explode(tags)`.

### B) Regex parsing of log lines

`access_logs.line` looks like: `2026-10-21 10:16:10 ERROR user=u2 action=upload ip=10.0.0.7 code=500`.

**Technique:** pull out the `key=value` piece with a regex, then remove the `key=` part.

MySQL 8:

```sql
SELECT log_id,
       LEFT(line, 19)                                                              AS ts,
       REGEXP_SUBSTR(line, 'INFO|ERROR|WARN')                                      AS level,
       SUBSTRING_INDEX(REGEXP_SUBSTR(line, 'user=[a-z0-9]+'), '=', -1)             AS user_id,
       SUBSTRING_INDEX(REGEXP_SUBSTR(line, 'action=[a-z]+'), '=', -1)              AS action,
       SUBSTRING_INDEX(REGEXP_SUBSTR(line, 'ip=[0-9.]+'), '=', -1)                 AS ip,
       SUBSTRING_INDEX(REGEXP_SUBSTR(line, 'code=[0-9]+'), '=', -1)                AS error_code
FROM access_logs
ORDER BY log_id;
```

| log_id | ts | level | user_id | action | ip | error_code |
|---|---|---|---|---|---|---|
| 1 | 2026-10-21 10:15:03 | INFO | u1 | login | 10.0.0.5 | NULL |
| 2 | 2026-10-21 10:16:10 | ERROR | u2 | upload | 10.0.0.7 | 500 |
| 3 | 2026-10-21 10:17:45 | INFO | u3 | logout | 10.0.0.9 | NULL |

**Explanation:** `REGEXP_SUBSTR` returns the first match (`user=u2`) or NULL when absent (so `error_code` is NULL for rows 1 and 3). `SUBSTRING_INDEX(x, '=', -1)` keeps everything after the last `=`.

PostgreSQL equivalent for one field (the **capture group** is returned): `substring(line from 'user=([a-z0-9]+)')`. Spark: `regexp_extract(line, 'user=([a-z0-9]+)', 1)`.

### C) Other regex tools

**Strip non-digits from a phone number:**

```sql
SELECT REGEXP_REPLACE('(+91) 98765-43210', '[^0-9]', '') AS digits_only;
```

| digits_only |
|---|
| 919876543210 |

**Validate emails with a pattern:**

```sql
SELECT signup_id, email
FROM signups
WHERE email REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$'
ORDER BY signup_id;
```

| signup_id | email |
|---|---|
| 1 | a@x.com |
| 3 | b@y.co.in |

**Explanation:** `bad.email` has no `@`; `@no.com` has nothing before the `@`. (In MySQL string literals the backslash is doubled: `\\.` means a literal dot. PostgreSQL: use `~` instead of `REGEXP`.) This is the LeetCode *Find Users With Valid E-Mails* family.

| Regex piece | Meaning |
|---|---|
| `^` and `$` | start and end of string |
| `[a-z0-9]` | one character from a set |
| `+`, `*`, `?` | one or more, zero or more, optional |
| `{2,}` | at least 2 |
| `\\.` | a literal dot |
| `(a|b)` | alternation |
| `.` | any character (literal dot must be escaped) |

---

## P55: UPDATE, DELETE and UPSERT Problems

> Reminder: preview with `SELECT`, wrap in a transaction (see 0.2). The examples change data, so run them on a copy or `ROLLBACK`.

### A) UPDATE with a JOIN

**Problem:** give IT employees a 10% raise.

MySQL:

```sql
UPDATE employees e
JOIN departments d ON d.dept_id = e.dept_id
SET e.salary = ROUND(e.salary * 1.10)
WHERE d.dept_name = 'IT';
```

PostgreSQL:

```sql
UPDATE employees e
SET salary = ROUND(salary * 1.10)
FROM departments d
WHERE d.dept_id = e.dept_id AND d.dept_name = 'IT';
```

**Before and after (IT employees):**

| emp_id | name | salary before | salary after |
|---|---|---|---|
| 1 | Amit Sharma | 90000 | 99000 |
| 3 | Rahul Verma | 68000 | 74800 |
| 5 | Karan Mehta | 60000 | 66000 |
| 9 | Rohan Das | 68000 | 74800 |

**Explanation:** the join identifies which employees belong to IT; only those rows are changed. Other departments are untouched. SQL Server uses `UPDATE e SET ... FROM employees e JOIN departments d ON ...`.

### B) UPDATE with CASE (the "swap values" trick)

**Problem (LeetCode *Swap Salary*):** swap `'m'` and `'f'` in one statement.

```sql
UPDATE salary_swap
SET sex = CASE WHEN sex = 'm' THEN 'f' ELSE 'm' END;
```

| id | name | sex before | sex after |
|---|---|---|---|
| 1 | A | m | f |
| 2 | B | f | m |
| 3 | C | m | f |
| 4 | D | f | m |

**Explanation:** a single statement evaluates the `CASE` against each row's **old** value, so no temporary column is needed.

### C) DELETE duplicates, keeping the smallest id

**Problem (LeetCode *Delete Duplicate Emails*):**

`person` before:

| id | email |
|---|---|
| 1 | john@example.com |
| 2 | bob@example.com |
| 3 | john@example.com |
| 4 | alice@example.com |
| 5 | bob@example.com |
| 6 | john@example.com |

**Approach 1: self-join (MySQL):** delete any row that has a **smaller id** with the same email.

```sql
DELETE p1
FROM person p1
JOIN person p2
  ON p1.email = p2.email
 AND p1.id > p2.id;
```

PostgreSQL:

```sql
DELETE FROM person p1
USING person p2
WHERE p1.email = p2.email AND p1.id > p2.id;
```

**Approach 2: window function (portable idea):**

```sql
DELETE FROM person
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
        FROM person
    ) t
    WHERE rn > 1
);
```

`person` after:

| id | email |
|---|---|
| 1 | john@example.com |
| 2 | bob@example.com |
| 4 | alice@example.com |

**Step-by-step:**
- For `john@example.com` the ids are 1, 3, 6. Rows 3 and 6 each have a smaller id (1) with the same email, so both are deleted.
- For `bob@example.com`, ids 2 and 5: row 5 is deleted.
- `alice@example.com` has one row: unchanged. **Deleted ids: 3, 5, 6.**
- The extra derived table `t` is required in MySQL (it forbids selecting from the table being deleted in a subquery directly).

**DELETE using a CTE:**
- SQL Server: `WITH d AS (SELECT ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) rn FROM person) DELETE FROM d WHERE rn > 1;`
- PostgreSQL: `WITH d AS (SELECT id FROM (...) t WHERE rn > 1) DELETE FROM person WHERE id IN (SELECT id FROM d);`

### D) INSERT ... SELECT

Copy rows from one table to another:

```sql
INSERT INTO orders_archive (order_id, customer_id, order_date, amount)
SELECT order_id, customer_id, order_date, amount
FROM orders
WHERE order_date < '2026-03-01';
```

(Archive pattern: copy first, verify counts, then delete from the source.)

### E) UPSERT / MERGE (update if exists, insert if not)

`inventory` before, and `incoming` shipments:

| product_id | product | stock |
|---|---|---|
| 1 | Pen | 100 |
| 2 | Book | 50 |
| 3 | Bag | 20 |

| product_id | product | qty |
|---|---|---|
| 2 | Book | 30 |
| 3 | Bag | 10 |
| 4 | Lamp | 25 |

**Goal:** add incoming quantities to existing products; insert new products (Lamp).

**MySQL (`ON DUPLICATE KEY UPDATE`; needs a primary or unique key):**

```sql
INSERT INTO inventory (product_id, product, stock)
SELECT product_id, product, qty FROM incoming
ON DUPLICATE KEY UPDATE stock = inventory.stock + VALUES(stock);
```

**PostgreSQL (`ON CONFLICT`):**

```sql
INSERT INTO inventory (product_id, product, stock)
SELECT product_id, product, qty FROM incoming
ON CONFLICT (product_id) DO UPDATE SET stock = inventory.stock + EXCLUDED.stock;
```

**Standard `MERGE` (SQL Server, Oracle, PostgreSQL 15+, Snowflake, Delta/Iceberg in Spark):**

```sql
MERGE INTO inventory t
USING incoming s ON t.product_id = s.product_id
WHEN MATCHED THEN
    UPDATE SET stock = t.stock + s.qty
WHEN NOT MATCHED THEN
    INSERT (product_id, product, stock) VALUES (s.product_id, s.product, s.qty);
```

`inventory` after:

| product_id | product | stock |
|---|---|---|
| 1 | Pen | 100 |
| 2 | Book | 80 |
| 3 | Bag | 30 |
| 4 | Lamp | 25 |

**Explanation:** products 2 and 3 already existed: stock increases by 30 and 10. Product 4 did not exist: a new row is inserted. Pen is untouched. (In MySQL, `VALUES(stock)` means "the value that would have been inserted into `stock`"; it is deprecated in newer 8.0 releases in favor of a row alias, but still works.)

> **Idempotency:** running this upsert twice would **add the shipment twice**. For data pipelines, make loads idempotent (use a batch id, or overwrite instead of add).

---

## P56: Relational Division and Set Membership

**Relational division** answers: "which X are related to **ALL** of the Y's?" (customers who bought **all** products, students who took **all** required courses, actors who appeared in **all** films of a franchise).

`purchases`: c1: Laptop, Mouse, Keyboard | c2: Laptop, Mouse | c3: Laptop, Mouse, Keyboard, Keyboard (duplicate) | c4: Keyboard | c5: all three.

### Method 1: count distinct and compare with the total (most common)

```sql
SELECT customer_id
FROM purchases
GROUP BY customer_id
HAVING COUNT(DISTINCT product_id) = (SELECT COUNT(*) FROM products)
ORDER BY customer_id;
```

| customer_id |
|---|
| 1 |
| 3 |
| 5 |

**Step-by-step:**

| customer_id | distinct products | equals 3? |
|---|---|---|
| 1 | 3 | Yes |
| 2 | 2 | No |
| 3 | 3 (the duplicate Keyboard is ignored by `DISTINCT`) | Yes |
| 4 | 1 | No |
| 5 | 3 | Yes |

> **Critical:** `COUNT(DISTINCT ...)`. Without `DISTINCT`, customer 3's 4 rows would produce 4, which does not match 3, and customer 3 would be wrongly dropped. (With other duplicates it could also wrongly **include** someone.) This is LeetCode *Customers Who Bought All Products*.

### Method 2: double NOT EXISTS ("there is no product they did not buy")

```sql
SELECT DISTINCT c.customer_id
FROM purchases c
WHERE NOT EXISTS (
    SELECT 1
    FROM products p
    WHERE NOT EXISTS (
        SELECT 1
        FROM purchases x
        WHERE x.customer_id = c.customer_id
          AND x.product_id  = p.product_id
    )
)
ORDER BY c.customer_id;
```

Same result: customers 1, 3, 5. **Reading it:** for a customer, look for a product `p` that has no purchase row by that customer; if no such product exists, the customer bought everything.

### Variations (set membership patterns)

**1) Bought Laptop AND Mouse (at least these two):**

```sql
SELECT customer_id
FROM purchases
WHERE product_id IN (1, 2)
GROUP BY customer_id
HAVING COUNT(DISTINCT product_id) = 2
ORDER BY customer_id;
```

| customer_id |
|---|
| 1 |
| 2 |
| 3 |
| 5 |

**2) Bought Laptop and Mouse but NOT Keyboard (conditional aggregation):**

```sql
SELECT customer_id
FROM purchases
GROUP BY customer_id
HAVING SUM(CASE WHEN product_id = 1 THEN 1 ELSE 0 END) > 0
   AND SUM(CASE WHEN product_id = 2 THEN 1 ELSE 0 END) > 0
   AND SUM(CASE WHEN product_id = 3 THEN 1 ELSE 0 END) = 0;
```

| customer_id |
|---|
| 2 |

**3) Bought EXACTLY {Laptop, Mouse} and nothing else:**

```sql
SELECT customer_id
FROM purchases
GROUP BY customer_id
HAVING COUNT(DISTINCT product_id) = 2
   AND COUNT(DISTINCT CASE WHEN product_id IN (1, 2) THEN product_id END) = 2;
```

| customer_id |
|---|
| 2 |

**Explanation:** the first condition says "exactly 2 distinct products overall"; the second says "both of those are in {1, 2}". Together they mean exactly that set.

| Question | Technique |
|---|---|
| Bought **all** | `HAVING COUNT(DISTINCT y) = (SELECT COUNT(*) FROM Y)` |
| Bought **at least** a given set | filter `IN (...)`, `HAVING COUNT(DISTINCT y) = size_of_set` |
| Bought A **but not** B | conditional `SUM(CASE ...)` in `HAVING` |
| Bought **exactly** a set | distinct total equals set size AND all in the set |
| Never bought anything | `NOT EXISTS` / `LEFT JOIN ... IS NULL` |

---

## P57: Multi-Pattern CTE Capstone

Hard interview questions combine **3 or more patterns**. Practice reading a problem and naming the patterns before you write any SQL.

**Problem:** for each month, report: the month revenue, its **month-over-month growth %**, the **top-spending customer** of that month with their spend and **share of the month's revenue**, and whether that customer is **new or returning**.

**Patterns used:** monthly bucketing (P34), MoM with `LAG` (P35), top 1 per group with `ROW_NUMBER` (P20), percent of total (P26), new vs returning (P46).

```sql
WITH monthly AS (                                   -- P34: bucket by month
    SELECT DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS month_start,
           SUM(amount) AS revenue
    FROM orders
    GROUP BY DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY)
),
growth AS (                                         -- P35: growth vs previous month
    SELECT month_start, revenue,
           ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month_start))
                 / NULLIF(LAG(revenue) OVER (ORDER BY month_start), 0), 2) AS mom_pct
    FROM monthly
),
cust_month AS (                                     -- spend per customer per month
    SELECT customer_id,
           DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS month_start,
           SUM(amount) AS spend
    FROM orders
    GROUP BY customer_id, DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY)
),
ranked AS (                                         -- P20: top 1 per month
    SELECT customer_id, month_start, spend,
           ROW_NUMBER() OVER (PARTITION BY month_start ORDER BY spend DESC, customer_id) AS rn
    FROM cust_month
),
first_month AS (                                    -- P46: when did each customer start?
    SELECT customer_id,
           MIN(DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY)) AS first_month
    FROM orders
    GROUP BY customer_id
)
SELECT g.month_start,
       g.revenue                                      AS month_revenue,
       g.mom_pct,
       r.customer_id                                  AS top_customer,
       r.spend                                        AS top_spend,
       ROUND(100.0 * r.spend / g.revenue, 2)          AS share_pct,                 -- P26
       CASE WHEN f.first_month = g.month_start THEN 'New' ELSE 'Returning' END AS customer_type
FROM growth g
JOIN ranked r        ON r.month_start = g.month_start AND r.rn = 1
JOIN first_month f   ON f.customer_id = r.customer_id
ORDER BY g.month_start;
```

| month_start | month_revenue | mom_pct | top_customer | top_spend | share_pct | customer_type |
|---|---|---|---|---|---|---|
| 2026-01-01 | 1250 | NULL | 1 | 500 | 40.00 | New |
| 2026-02-01 | 1550 | 24.00 | 1 | 700 | 45.16 | Returning |
| 2026-03-01 | 1700 | 9.68 | 5 | 800 | 47.06 | New |
| 2026-04-01 | 850 | -50.00 | 1 | 600 | 70.59 | Returning |

**Step-by-step:**
- **January:** spends are customer 1 = 500, customer 2 = 300, customer 3 = 450, so customer 1 is top (500 / 1250 = 40.00%). All January customers are new.
- **February:** customer 1 spent 700 (customer 4 spent 650, customer 2 spent 200). Customer 1 started in January, so they are returning: 700 / 1550 = 45.16%.
- **March:** customer 5 spent 800 (the highest) and placed their first order in March, so they are new: 800 / 1700 = 47.06%.
- **April:** customer 1 spent 600 vs customer 6's 250: 600 / 850 = 70.59%, returning.
- **How to approach:** one idea per CTE, each CTE small enough to test alone with `SELECT * FROM cte`. Build and verify step by step.

---

## P58: Graph Problems in SQL

A graph is a table of **edges**. Here `friendships(user_a, user_b)` stores each friendship once. For undirected graphs, **first make edges bidirectional**:

```sql
WITH e AS (
    SELECT user_a AS u, user_b AS v FROM friendships
    UNION ALL
    SELECT user_b, user_a FROM friendships
)
```

Now `e` has 14 rows: each friendship appears in both directions, so "friends of u" = `SELECT v FROM e WHERE u = ...`.

### A) Friend counts (degree)

```sql
WITH e AS (
    SELECT user_a AS u, user_b AS v FROM friendships
    UNION ALL
    SELECT user_b, user_a FROM friendships
)
SELECT p.id, p.name, COUNT(e.v) AS friends
FROM people p
LEFT JOIN e ON e.u = p.id
GROUP BY p.id, p.name
ORDER BY friends DESC, p.id;
```

| id | name | friends |
|---|---|---|
| 2 | Ravi | 3 |
| 3 | Meena | 3 |
| 5 | Divya | 3 |
| 1 | Asha | 2 |
| 4 | Kabir | 2 |
| 6 | Esha | 1 |

(This is the idea behind LeetCode *Friend Requests II: Who Has the Most Friends*.)

### B) Mutual friends of two users

**Problem:** mutual friends of Asha (1) and Kabir (4).

```sql
WITH e AS (
    SELECT user_a AS u, user_b AS v FROM friendships
    UNION ALL
    SELECT user_b, user_a FROM friendships
)
SELECT a.v AS mutual_friend_id, p.name
FROM e a
JOIN e b ON a.v = b.v
JOIN people p ON p.id = a.v
WHERE a.u = 1 AND b.u = 4;
```

| mutual_friend_id | name |
|---|---|
| 2 | Ravi |

**Explanation:** friends of 1 are {2, 3}, friends of 4 are {2, 5}; the intersection is {2}. The join on `a.v = b.v` keeps people who appear as a friend of both.

### C) Friend recommendations (friends of friends who are not yet friends)

**Rule:** recommend `w` to `u` if `w` is a friend of a friend of `u`, `w` is not `u`, and `w` is not already a friend. Rank by the number of **mutual friends**.

```sql
WITH e AS (
    SELECT user_a AS u, user_b AS v FROM friendships
    UNION ALL
    SELECT user_b, user_a FROM friendships
)
SELECT e1.u         AS user_id,
       e2.v         AS recommended_id,
       COUNT(*)     AS mutual_friends
FROM e e1
JOIN e e2 ON e1.v = e2.u AND e2.v <> e1.u          -- friend of a friend, not myself
LEFT JOIN e e3 ON e3.u = e1.u AND e3.v = e2.v       -- already a friend?
WHERE e3.u IS NULL                                   -- keep only non-friends
GROUP BY e1.u, e2.v
ORDER BY user_id, mutual_friends DESC, recommended_id;
```

| user_id | recommended_id | mutual_friends |
|---|---|---|
| 1 | 4 | 1 |
| 1 | 5 | 1 |
| 2 | 5 | 2 |
| 3 | 4 | 2 |
| 3 | 6 | 1 |
| 4 | 3 | 2 |
| 4 | 1 | 1 |
| 4 | 6 | 1 |
| 5 | 2 | 2 |
| 5 | 1 | 1 |
| 6 | 3 | 1 |
| 6 | 4 | 1 |

**Step-by-step (user 2 = Ravi, friends {1, 3, 4}):**
- Friends of Asha (1): {2, 3} gives candidates 2 (self, removed) and 3 (already a friend, removed).
- Friends of Meena (3): {1, 2, 5} gives 1 (already a friend), 2 (self), 5 (new).
- Friends of Kabir (4): {2, 5} gives 2 (self), 5 (new).
- Candidate **5 (Divya)** appears via Meena **and** Kabir, so `COUNT(*) = 2` mutual friends. Result: (2, 5, 2).

**How the three joins work:**
- `e1`: edge (u, v): "u's friend v".
- `e2`: edge (v, w): "v's friend w" (this makes w a friend of a friend). `e2.v <> e1.u` removes the user themselves.
- `e3` (anti-join): finds an existing direct friendship (u, w); `WHERE e3.u IS NULL` keeps only pairs with **no** existing friendship.
- Each (u, intermediate friend v, w) row is one mutual friend, so `COUNT(*)` counts mutual friends.

(This is the pattern behind LeetCode *Page Recommendations*.)

### D) Shortest path (degrees of separation) with a recursive CTE

**Problem:** the minimum number of hops from Asha (1) to every other user.

```sql
WITH RECURSIVE e AS (
    SELECT user_a AS u, user_b AS v FROM friendships
    UNION ALL
    SELECT user_b, user_a FROM friendships
),
walk AS (
    SELECT 1 AS node,
           0 AS hops,
           CAST('1' AS CHAR(200)) AS path               -- start at user 1
    UNION ALL
    SELECT e.v,
           w.hops + 1,
           CAST(CONCAT(w.path, '>', e.v) AS CHAR(200))
    FROM walk w
    JOIN e ON e.u = w.node
    WHERE FIND_IN_SET(e.v, REPLACE(w.path, '>', ',')) = 0   -- never revisit a node on this path
)
SELECT node, MIN(hops) AS shortest_hops
FROM walk
GROUP BY node
ORDER BY node;
```

| node | shortest_hops |
|---|---|
| 1 | 0 |
| 2 | 1 |
| 3 | 1 |
| 4 | 2 |
| 5 | 2 |
| 6 | 3 |

**Step-by-step:**
- Level 0: user 1.
- Level 1: neighbors of 1: users 2 and 3.
- Level 2: neighbors of 2 and 3 (not already on the path): 4 (via 2), 5 (via 3).
- Level 3: neighbors of 5: user 6.
- The recursive CTE enumerates **all simple paths**; `MIN(hops)` per node picks the shortest. The `FIND_IN_SET` check (`path` as a comma list) prevents cycles like 1 > 2 > 1.

> **Warning:** enumerating all paths grows **exponentially** in dense or large graphs. Real systems use BFS-style iteration (level by level, keeping only unvisited nodes), a graph database, or Spark GraphFrames. For interviews, small graphs with a depth limit are fine; always mention the blow-up risk.

---

## P59: Tournament and Scoreboard Computation

**Core trick:** each match involves **two teams**, but the table has one row per match. **Unpivot** with `UNION ALL`: one row per team per match (as "home" and as "away"), swapping goals for the away side. Then aggregate.

```sql
WITH t AS (
    SELECT home_team AS team, home_goals AS gf, away_goals AS ga FROM matches
    UNION ALL
    SELECT away_team AS team, away_goals AS gf, home_goals AS ga FROM matches
)
SELECT team,
       COUNT(*)                                          AS played,
       SUM(CASE WHEN gf > ga THEN 1 ELSE 0 END)          AS wins,
       SUM(CASE WHEN gf = ga THEN 1 ELSE 0 END)          AS draws,
       SUM(CASE WHEN gf < ga THEN 1 ELSE 0 END)          AS losses,
       SUM(gf)                                           AS goals_for,
       SUM(ga)                                           AS goals_against,
       SUM(gf) - SUM(ga)                                 AS goal_diff,
       SUM(CASE WHEN gf > ga THEN 3 WHEN gf = ga THEN 1 ELSE 0 END) AS points
FROM t
GROUP BY team
ORDER BY points DESC, goal_diff DESC, goals_for DESC;
```

Intermediate view (`t`, 12 rows, 2 per match):

| team | gf | ga | (from match) |
|---|---|---|---|
| A | 2 | 1 | 1 (home) |
| C | 0 | 0 | 2 (home) |
| A | 1 | 3 | 3 (home) |
| B | 2 | 2 | 4 (home) |
| A | 3 | 0 | 5 (home) |
| B | 1 | 2 | 6 (home) |
| B | 1 | 2 | 1 (away) |
| D | 0 | 0 | 2 (away) |
| C | 3 | 1 | 3 (away) |
| D | 2 | 2 | 4 (away) |
| D | 0 | 3 | 5 (away) |
| C | 2 | 1 | 6 (away) |

Final standings:

| team | played | wins | draws | losses | goals_for | goals_against | goal_diff | points |
|---|---|---|---|---|---|---|---|---|
| C | 3 | 2 | 1 | 0 | 5 | 2 | 3 | 7 |
| A | 3 | 2 | 0 | 1 | 6 | 4 | 2 | 6 |
| D | 3 | 0 | 2 | 1 | 2 | 5 | -3 | 2 |
| B | 3 | 0 | 1 | 2 | 4 | 6 | -2 | 1 |

**Step-by-step (team C):**
- Match 2 vs D: 0-0, draw (1 point).
- Match 3 at A: C scored 3, A scored 1, win (3 points).
- Match 6 at B: C scored 2, B scored 1, win (3 points).
- Total: 2 wins, 1 draw, 0 losses, 5 goals for, 2 against, goal difference +3, **7 points**.

**Sanity check:** total goals for (6 + 4 + 5 + 2 = 17) equals total goals against (17), and matches the sum of all goals in the match table. Always run such a check.

**Adding the rank (with a tie-break rule: points, then goal difference, then goals scored):**

```sql
-- append after the aggregate, wrapping it as a CTE named standings
SELECT RANK() OVER (ORDER BY points DESC, goal_diff DESC, goals_for DESC) AS position,
       team, points, goal_diff
FROM standings
ORDER BY position;
```

| position | team | points | goal_diff |
|---|---|---|---|
| 1 | C | 7 | 3 |
| 2 | A | 6 | 2 |
| 3 | D | 2 | -3 |
| 4 | B | 1 | -2 |

**Variations:** teams that never lost (`losses = 0`: here only C), head-to-head records (filter the unpivoted rows by opponent), winning streaks (gaps and islands on win flags, Phase 4), group-stage qualification (top 2 per group using `ROW_NUMBER`). Related LeetCode: *Team Scores in Football Tournament*, *Tournament Winners*.

> **Avoid `SUM(boolean)`:** MySQL allows `SUM(gf > ga)` but PostgreSQL does not. The `CASE` form works everywhere.

---

## P60: Hard Interval Problems (Sweep Line, Merge, Free Time)

Phase 5 (P41) covered pairwise overlaps and basic merging. Here are the harder classics, using `meetings`.

### A) Maximum concurrent intervals (sweep line): "how many rooms do we need?"

**Idea:** turn each interval into two **events**: `+1` at the start, `-1` at the end. Sort events by time and take a **running sum**: the running sum is "how many meetings are happening now." Its maximum is the answer.

**Tie rule:** if one meeting ends exactly when another starts (11:00), process the **end first** (`-1` before `+1`), because ends are exclusive here. Sorting by `delta` ascending does this automatically (-1 sorts before +1).

```sql
WITH ev AS (
    SELECT start_time AS t,  1 AS delta FROM meetings
    UNION ALL
    SELECT end_time   AS t, -1 AS delta FROM meetings
),
run AS (
    SELECT t, delta,
           SUM(delta) OVER (ORDER BY t, delta
                            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS concurrent
    FROM ev
)
SELECT t, delta, concurrent
FROM run
ORDER BY t, delta;
```

| t | delta | concurrent |
|---|---|---|
| 09:00:00 | 1 | 1 |
| 10:00:00 | 1 | 2 |
| 10:15:00 | 1 | 3 |
| 10:30:00 | -1 | 2 |
| 10:45:00 | -1 | 1 |
| 11:00:00 | -1 | 0 |
| 11:00:00 | 1 | 1 |
| 12:00:00 | -1 | 0 |
| 13:00:00 | 1 | 1 |
| 13:30:00 | 1 | 2 |
| 14:00:00 | -1 | 1 |
| 15:00:00 | -1 | 0 |

```sql
-- append after the run CTE
SELECT MAX(concurrent) AS max_concurrent FROM run;
```

| max_concurrent |
|---|
| 3 |

**Reading it:** at 10:15 meetings 1, 2 and 3 are all running, so **3 rooms are needed** (the classic *Meeting Rooms II* problem). The same pattern gives "peak concurrent users" from login/logout times, or "maximum overlapping bookings."

### B) Merge intervals and compute total busy time

Same technique as Phase 5 (P41-C): running `MAX(end)` of earlier rows, flag a new block when the start is **after** that.

```sql
WITH ordered AS (
    SELECT meeting_id, start_time, end_time,
           MAX(end_time) OVER (ORDER BY start_time, meeting_id
                               ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS prev_max_end
    FROM meetings
),
flagged AS (
    SELECT *, CASE WHEN prev_max_end IS NULL OR start_time > prev_max_end THEN 1 ELSE 0 END AS new_block
    FROM ordered
),
blocks AS (
    SELECT *, SUM(new_block) OVER (ORDER BY start_time, meeting_id) AS block_id
    FROM flagged
)
SELECT block_id,
       MIN(start_time) AS block_start,
       MAX(end_time)   AS block_end,
       COUNT(*)        AS meetings_merged,
       TIME_TO_SEC(TIMEDIFF(MAX(end_time), MIN(start_time))) DIV 60 AS busy_minutes
FROM blocks
GROUP BY block_id
ORDER BY block_id;
```

Step table:

| meeting_id | start | end | prev_max_end | new_block | block_id |
|---|---|---|---|---|---|
| 1 | 09:00 | 10:30 | NULL | 1 | 1 |
| 2 | 10:00 | 11:00 | 10:30 | 0 | 1 |
| 3 | 10:15 | 10:45 | 11:00 | 0 | 1 |
| 4 | 11:00 | 12:00 | 11:00 | 0 (starts exactly when the block ends, so it touches and merges) | 1 |
| 5 | 13:00 | 14:00 | 12:00 | 1 | 2 |
| 6 | 13:30 | 15:00 | 14:00 | 0 | 2 |

Final output:

| block_id | block_start | block_end | meetings_merged | busy_minutes |
|---|---|---|---|---|
| 1 | 09:00:00 | 12:00:00 | 4 | 180 |
| 2 | 13:00:00 | 15:00:00 | 2 | 120 |

**Total busy time = 180 + 120 = 300 minutes.** Note that you **cannot** simply add each meeting's duration (that would count the overlaps twice): merging first removes the double counting.

> **Touching intervals:** here a meeting that starts exactly when the previous block ends merges into it (`start_time > prev_max_end` is false at equality). If touching intervals must stay separate, use `>=`. Always confirm.

### C) Free time slots (gaps between merged blocks within working hours 09:00 to 17:00)

```sql
-- append after the blocks CTE and a merged CTE named merged(block_start, block_end)
SELECT block_end AS free_start,
       COALESCE(LEAD(block_start) OVER (ORDER BY block_start), '17:00:00') AS free_end
FROM merged
```

Complete version with minutes:

```sql
WITH ordered AS (
    SELECT meeting_id, start_time, end_time,
           MAX(end_time) OVER (ORDER BY start_time, meeting_id
                               ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS prev_max_end
    FROM meetings
),
flagged AS (
    SELECT *, CASE WHEN prev_max_end IS NULL OR start_time > prev_max_end THEN 1 ELSE 0 END AS new_block
    FROM ordered
),
blocks AS (
    SELECT *, SUM(new_block) OVER (ORDER BY start_time, meeting_id) AS block_id
    FROM flagged
),
merged AS (
    SELECT block_id, MIN(start_time) AS block_start, MAX(end_time) AS block_end
    FROM blocks
    GROUP BY block_id
),
gaps AS (
    SELECT block_end AS free_start,
           COALESCE(LEAD(block_start) OVER (ORDER BY block_start), '17:00:00') AS free_end
    FROM merged
)
SELECT free_start, free_end,
       TIME_TO_SEC(TIMEDIFF(free_end, free_start)) DIV 60 AS free_minutes
FROM gaps
WHERE free_start < free_end
ORDER BY free_start;
```

| free_start | free_end | free_minutes |
|---|---|---|
| 12:00:00 | 13:00:00 | 60 |
| 15:00:00 | 17:00:00 | 120 |

**Step-by-step:**
- Merged blocks: 09:00 to 12:00 and 13:00 to 15:00.
- Free time after block 1 = from 12:00 to the next block's start (13:00) = 60 minutes.
- Free time after block 2: there is no next block (`LEAD` is NULL), so `COALESCE` uses the end of the working day, 17:00: 120 minutes.
- A **leading gap** (free time before the first meeting) exists only if the first block starts after 09:00; add it with a `UNION ALL` row `('09:00:00', MIN(block_start))` filtered by `free_start < free_end`. Here the first meeting starts at 09:00, so there is none.
- PostgreSQL: `EXTRACT(EPOCH FROM (free_end - free_start)) / 60`.

| Interval problem | Technique |
|---|---|
| Do two intervals overlap? | `A.start <= B.end AND A.end >= B.start` |
| Max concurrent / rooms needed | events (+1, -1), running sum, max |
| Merge into blocks | sort by start, running `MAX(end)`, new-block flag, running sum |
| Total covered time | merge first, then sum block lengths |
| Free slots / gaps | `LEAD(block_start)` over merged blocks, with day boundaries |
| Coverage by date | calendar CTE joined by `date BETWEEN start AND end` |

---

# PART 4: Wrap-Up

## 4.1 Top Mistakes in Advanced SQL

| Mistake | Fix |
|---|---|
| Using `JOIN LATERAL` and losing groups with no rows | `LEFT JOIN LATERAL ... ON TRUE` (or `OUTER APPLY`) |
| Splitting strings without `TRIM` or with a numbers table that is too short | `TRIM` items; make the numbers table at least as long as the longest list |
| `LIKE '%sql%'` or `FIND_IN_SET` for list membership at scale | explode to rows or use a junction table |
| Using `->` when you need text | use `->>` and cast numeric values |
| Forgetting that a missing JSON key returns NULL | `COALESCE` and check key existence |
| Inner join with `JSON_TABLE` / `unnest` drops rows with empty arrays | `LEFT JOIN ... ON TRUE` |
| Regex special characters (`.`) not escaped; backslashes not doubled in MySQL | `\\.` in MySQL strings |
| `UPDATE` / `DELETE` without a preview, without `WHERE`, or without a transaction | preview with `SELECT`; transaction; backup |
| Deleting from a table while selecting from it in MySQL | wrap in a derived table or use the self-join `DELETE` |
| Upsert adds twice when a job re-runs | make it idempotent (batch id, overwrite) |
| `COUNT(*)` instead of `COUNT(DISTINCT ...)` in relational division | duplicates break the comparison: use `DISTINCT` |
| Treating an undirected graph as directed | make edges bidirectional with `UNION ALL` |
| Counting friends-of-friends who are already friends, or the user themselves | anti-join and `<>` check |
| Path enumeration on large graphs | depth limit; BFS iteration; graph tools |
| Tournament: forgetting to unpivot home and away | `UNION ALL` the two sides, swapping goals |
| Sweep line: wrong order of start/end events at equal times | sort by `time, delta` so ends come first (or last, if intervals are inclusive) |
| Adding up overlapping durations directly | merge intervals first |
| Putting too much logic in one huge query | one idea per CTE; test each CTE separately |

## 4.2 Phase 6 Cheat Sheet

```text
CASE (agg)      CASE WHEN SUM(x) >= ... THEN ... END   after GROUP BY
CASE (cond agg) SUM(CASE WHEN cond THEN 1 ELSE 0 END)    PG: SUM(x) FILTER (WHERE cond)
CASE (window)   SUM(CASE WHEN cond THEN 1 ELSE 0 END) OVER (PARTITION BY k ORDER BY d ROWS UNBOUNDED PRECEDING)

LATERAL         FROM A LEFT JOIN LATERAL (SELECT ... WHERE b.k = a.k ORDER BY ... LIMIT n) t ON TRUE
APPLY (SQLSrv)  OUTER APPLY (SELECT TOP n ... WHERE b.k = a.k ORDER BY ...) t

SPLIT           MySQL  : numbers CTE + SUBSTRING_INDEX(SUBSTRING_INDEX(s,',',n),',',-1)
                PG     : unnest(string_to_array(s, ','))
                Spark  : LATERAL VIEW explode(split(s, ','))
JSON            MySQL  : col->>'$.a.b'     JSON_TABLE(col, '$.arr[*]' COLUMNS (...))
                PG     : col->'a'->>'b'    jsonb_array_elements_text(col->'arr')
REGEX           MySQL  : REGEXP_SUBSTR / REGEXP_REPLACE / REGEXP     PG: substring(s from 're') / ~
BITS            test (x & mask) > 0    set x | mask    clear x & ~mask    BIT_COUNT   XOR finds the odd one out

UPDATE JOIN     MySQL : UPDATE t JOIN u ON ... SET ...        PG : UPDATE t SET ... FROM u WHERE ...
DELETE DUPES    DELETE p1 FROM person p1 JOIN person p2 ON p1.email = p2.email AND p1.id > p2.id
UPSERT          MySQL : INSERT ... ON DUPLICATE KEY UPDATE    PG : ON CONFLICT (k) DO UPDATE    std : MERGE

DIVISION        HAVING COUNT(DISTINCT y) = (SELECT COUNT(*) FROM Y)     or double NOT EXISTS
BUT NOT         HAVING SUM(CASE WHEN y=A ...) > 0 AND SUM(CASE WHEN y=B ...) = 0

GRAPH           edges both ways: UNION ALL swap | mutual friends: self-join on neighbor
                recommendations: e1 JOIN e2 (fof) + LEFT JOIN e3 IS NULL (not friends) + COUNT(*)
                shortest path: recursive CTE with visited-path check, MIN(hops)

SCOREBOARD      UNION ALL home and away (swap goals) -> SUM(CASE ...) -> ORDER BY points, goal_diff, goals_for

INTERVALS       overlap: A.s <= B.e AND A.e >= B.s    rooms: events +1/-1 running SUM, MAX
                merge: running MAX(end) of previous rows + new-block flag + running SUM
                free time: LEAD(block_start) over merged blocks
```

## 4.3 Practice Problems

1. Explode `employees_skills` and count how many people know each skill.
2. List people who know **sql, python and spark** (all three).
3. From `events_json`, show the **total purchase amount per user**.
4. From `access_logs`, count **ERROR** lines per user.
5. Delete duplicate rows from `person`, keeping the smallest id. Which ids remain?
6. Upsert the `incoming` shipments into `inventory`. What is Book's final stock?
7. Which customers bought **Laptop but never Keyboard**?
8. Which customers bought **all** products? (Use both methods.)
9. List the **mutual friends** of Ravi (2) and Meena (3).
10. Compute the **shortest number of hops from Esha (6) to Asha (1)**.
11. Produce the football **standings with positions** from `matches`.
12. How many **meeting rooms** are required for `meetings`?
13. Using `LATERAL`, show the **latest 2 orders per customer**.
14. Challenge: using window functions only (no LATERAL), show the top 2 earners per department, then explain when you would prefer `LATERAL`.
15. Challenge: show, for each month, the customers who ordered in that month **and** the previous month (month-to-month repeat customers).

<details>
<summary>Answers</summary>

```sql
-- 1  python 3, sql 3, spark 2, airflow 1, excel 1   (see P53)
--    use the exploding query as a CTE skills_long, then:
SELECT skill, COUNT(*) AS people FROM skills_long GROUP BY skill ORDER BY people DESC, skill;

-- 2  Amit (sql, python, spark) and Sneha (spark, sql, airflow, python): both know all three
SELECT emp_id, name
FROM skills_long
WHERE skill IN ('sql', 'python', 'spark')
GROUP BY emp_id, name
HAVING COUNT(DISTINCT skill) = 3
ORDER BY emp_id;

-- 3  u1 = 99.99, u2 = 300.00  (only purchase events count)
SELECT payload->>'$.user' AS user_id,
       SUM(CAST(payload->>'$.amount' AS DECIMAL(10,2))) AS total_purchase
FROM events_json
WHERE payload->>'$.action' = 'purchase'
GROUP BY payload->>'$.user'
ORDER BY user_id;

-- 4  u2 -> 1
SELECT SUBSTRING_INDEX(REGEXP_SUBSTR(line, 'user=[a-z0-9]+'), '=', -1) AS user_id,
       COUNT(*) AS errors
FROM access_logs
WHERE line LIKE '% ERROR %'
GROUP BY SUBSTRING_INDEX(REGEXP_SUBSTR(line, 'user=[a-z0-9]+'), '=', -1);

-- 5  remaining ids: 1, 2, 4
DELETE p1 FROM person p1 JOIN person p2 ON p1.email = p2.email AND p1.id > p2.id;

-- 6  Book = 80 (50 + 30); Bag = 30; Lamp inserted with 25; Pen unchanged at 100
INSERT INTO inventory (product_id, product, stock)
SELECT product_id, product, qty FROM incoming
ON DUPLICATE KEY UPDATE stock = inventory.stock + VALUES(stock);

-- 7  customer 2 only
SELECT customer_id
FROM purchases
GROUP BY customer_id
HAVING SUM(CASE WHEN product_id = 1 THEN 1 ELSE 0 END) > 0
   AND SUM(CASE WHEN product_id = 3 THEN 1 ELSE 0 END) = 0;

-- 8  customers 1, 3, 5  (see P56, both methods)

-- 9  friends of 2 = {1,3,4}; friends of 3 = {1,2,5}; intersection = {1} -> Asha
WITH e AS (SELECT user_a AS u, user_b AS v FROM friendships
           UNION ALL SELECT user_b, user_a FROM friendships)
SELECT a.v AS mutual_friend_id, p.name
FROM e a JOIN e b ON a.v = b.v JOIN people p ON p.id = a.v
WHERE a.u = 2 AND b.u = 3;

-- 10 3 hops (6 > 5 > 3 > 1). Start the recursive walk at node 6 (change the anchor row) and filter node = 1.
WITH RECURSIVE e AS (SELECT user_a AS u, user_b AS v FROM friendships
                     UNION ALL SELECT user_b, user_a FROM friendships),
walk AS (
    SELECT 6 AS node, 0 AS hops, CAST('6' AS CHAR(200)) AS path
    UNION ALL
    SELECT e.v, w.hops + 1, CAST(CONCAT(w.path, '>', e.v) AS CHAR(200))
    FROM walk w JOIN e ON e.u = w.node
    WHERE FIND_IN_SET(e.v, REPLACE(w.path, '>', ',')) = 0
)
SELECT MIN(hops) AS shortest_hops FROM walk WHERE node = 1;

-- 11 positions: C 1, A 2, D 3, B 4  (see P59)

-- 12 3  (see P60-A)

-- 13 latest 2 orders per customer
SELECT c.customer_id, t.order_id, t.order_date, t.amount
FROM (SELECT DISTINCT customer_id FROM orders) c
LEFT JOIN LATERAL (
    SELECT o.order_id, o.order_date, o.amount
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC, o.order_id DESC
    LIMIT 2
) t ON TRUE
ORDER BY c.customer_id, t.order_date DESC;
-- customer 1: orders 11 (04-02), 7 (03-07) | 2: 5, 2 | 3: 8, 3 | 4: 9, 6 | 5: 10 | 6: 12

-- 14 window version
WITH r AS (
  SELECT d.dept_name, e.name, e.salary,
         ROW_NUMBER() OVER (PARTITION BY e.dept_id ORDER BY e.salary DESC, e.emp_id) AS rn
  FROM employees e JOIN departments d ON d.dept_id = e.dept_id)
SELECT dept_name, name, salary FROM r WHERE rn <= 2;
-- Prefer LATERAL when there are few groups, a big detail table and an index on (group, sort column),
-- or when you must keep empty groups. Prefer windows for portability (Spark, BigQuery, Hive).

-- 15 customers active in a month and the previous month  (Feb: 1, 2 | Mar: 1, 4 | Apr: 1)
WITH cm AS (
  SELECT DISTINCT customer_id,
         DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS m
  FROM orders)
SELECT b.m AS month_start, b.customer_id
FROM cm b
JOIN cm a ON a.customer_id = b.customer_id
         AND a.m = DATE_SUB(b.m, INTERVAL 1 MONTH)
ORDER BY b.m, b.customer_id;
```
</details>


