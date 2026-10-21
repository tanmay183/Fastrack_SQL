# SQL Phase 4: Window Functions (Detailed Guide)

> The **single most important module** for SQL interviews. Hard-level problems (top N per group, running totals, gaps and islands, sessionization, retention, medians) almost always use window functions.
> Covers **Module 9 (Fundamentals)** and **Module 10 (Patterns P20 to P33)**.
> Every topic has: easy explanation, input table, query, output table, and step-by-step reasoning.

---

## Table of Contents

- **Part 0:** The big idea (mental model)
- **Part 1:** Sample data (create once)
- **Part 2:** Module 9: Window function fundamentals
- **Part 3:** Module 10: Interview patterns P20 to P33
- **Part 4:** Wrap-up (mistakes, cheat sheet, practice, self-test, next step)

> **Requirements:** MySQL **8.0+** or PostgreSQL (any modern version). Older MySQL (5.7) has no window functions.

---

# PART 0: The Big Idea

## 0.1 Problem that window functions solve

`GROUP BY` **collapses** rows: 10 employees become 4 department rows. You lose the individual rows.

Often you need **both**: the individual row **and** a calculation over a related set of rows.
- "Show each employee **and** the average salary of their department."
- "Show each order **and** the running total so far."
- "Show each employee **and** their rank inside the department."

A **window function** calculates a value for each row by looking at a **window** (a set of related rows) **without collapsing** the rows.

> **Mental model:** Imagine a sliding magnifying glass over your sorted rows. For each row, the function looks through the glass (the window), computes something, and writes the answer next to that row. All rows survive.

## 0.2 GROUP BY vs Window: side by side

**GROUP BY (collapses):**

```sql
SELECT department, AVG(salary) AS avg_salary
FROM employees
GROUP BY department;
```

| department | avg_salary |
|---|---|
| IT | 71500.00 |
| HR | 55000.00 |
| Finance | 86333.33 |
| Sales | 45000.00 |

(4 rows)

**Window (keeps every row):**

```sql
SELECT name, department, salary,
       AVG(salary) OVER (PARTITION BY department) AS dept_avg
FROM employees
ORDER BY emp_id;
```

| name | department | salary | dept_avg |
|---|---|---|---|
| Amit Sharma | IT | 90000 | 71500.00 |
| Priya Singh | HR | 52000 | 55000.00 |
| Rahul Verma | IT | 68000 | 71500.00 |
| Sneha Iyer | Finance | 82000 | 86333.33 |
| Karan Mehta | IT | 60000 | 71500.00 |
| Neha Gupta | HR | 58000 | 55000.00 |
| Vikram Rao | Finance | 95000 | 86333.33 |
| Anita Desai | Sales | 45000 | 45000.00 |
| Rohan Das | IT | 68000 | 71500.00 |
| Meera Nair | Finance | 82000 | 86333.33 |

(10 rows: every employee is kept, and the department average is attached to each.)

---

# PART 1: Sample Data (Create Once)

```sql
-- 1) employees
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(50),
    department  VARCHAR(20),
    salary      INT
);
INSERT INTO employees VALUES
(1,  'Amit Sharma', 'IT',      90000),
(2,  'Priya Singh', 'HR',      52000),
(3,  'Rahul Verma', 'IT',      68000),
(4,  'Sneha Iyer',  'Finance', 82000),
(5,  'Karan Mehta', 'IT',      60000),
(6,  'Neha Gupta',  'HR',      58000),
(7,  'Vikram Rao',  'Finance', 95000),
(8,  'Anita Desai', 'Sales',   45000),
(9,  'Rohan Das',   'IT',      68000),
(10, 'Meera Nair',  'Finance', 82000);

-- 2) daily_revenue
CREATE TABLE daily_revenue (
    sale_date  DATE,
    revenue    INT
);
INSERT INTO daily_revenue VALUES
('2026-10-01', 100),
('2026-10-02', 150),
('2026-10-03', 200),
('2026-10-04', 120),
('2026-10-05', 180),
('2026-10-06',  90),
('2026-10-07', 210);

-- 3) monthly_revenue
CREATE TABLE monthly_revenue (
    month_start  DATE,
    revenue      INT
);
INSERT INTO monthly_revenue VALUES
('2026-01-01', 1000),
('2026-02-01', 1200),
('2026-03-01',  900),
('2026-04-01', 1350),
('2026-05-01', 1350),
('2026-06-01', 1500);

-- 4) orders
CREATE TABLE orders (
    order_id     INT PRIMARY KEY,
    customer_id  INT,
    order_date   DATE,
    amount       INT
);
INSERT INTO orders VALUES
(101, 1, '2026-01-05', 500),
(102, 1, '2026-02-10', 700),
(103, 2, '2026-01-20', 300),
(104, 3, '2026-02-15', 900),
(105, 2, '2026-03-01', 450),
(106, 1, '2026-03-12', 200),
(107, 4, '2026-03-18', 650);

-- 5) contacts (has duplicate emails)
CREATE TABLE contacts (
    id          INT PRIMARY KEY,
    email       VARCHAR(30),
    created_at  DATE
);
INSERT INTO contacts VALUES
(1, 'a@x.com', '2026-01-01'),
(2, 'b@x.com', '2026-01-02'),
(3, 'a@x.com', '2026-01-05'),
(4, 'c@x.com', '2026-01-06'),
(5, 'b@x.com', '2026-01-07'),
(6, 'a@x.com', '2026-01-09');

-- 6) user_logins (for streaks)
CREATE TABLE user_logins (
    user_id     INT,
    login_date  DATE
);
INSERT INTO user_logins VALUES
(1, '2026-10-01'), (1, '2026-10-02'), (1, '2026-10-03'),
(1, '2026-10-05'), (1, '2026-10-06'), (1, '2026-10-09'),
(2, '2026-10-01'), (2, '2026-10-03'), (2, '2026-10-04'), (2, '2026-10-05');

-- 7) logs (for consecutive numbers)
CREATE TABLE logs (
    id   INT PRIMARY KEY,
    num  INT
);
INSERT INTO logs VALUES (1,1),(2,1),(3,1),(4,2),(5,1),(6,2),(7,2);

-- 8) scores (for rank scores)
CREATE TABLE scores (
    id     INT PRIMARY KEY,
    score  DECIMAL(3,2)
);
INSERT INTO scores VALUES (1,3.50),(2,3.65),(3,4.00),(4,3.85),(5,4.00),(6,3.65);

-- 9) events (for sessionization)
CREATE TABLE events (
    user_id     INT,
    event_time  TIMESTAMP
);
INSERT INTO events VALUES
(1, '2026-10-21 10:00:00'),
(1, '2026-10-21 10:10:00'),
(1, '2026-10-21 10:25:00'),
(1, '2026-10-21 11:30:00'),
(1, '2026-10-21 11:40:00'),
(1, '2026-10-21 14:00:00'),
(2, '2026-10-21 09:00:00'),
(2, '2026-10-21 09:50:00'),
(2, '2026-10-21 10:15:00');

-- 10) visits (for cumulative distinct users)
CREATE TABLE visits (
    user_id     INT,
    visit_date  DATE
);
INSERT INTO visits VALUES
(1,'2026-10-01'),(2,'2026-10-01'),
(1,'2026-10-02'),(3,'2026-10-02'),
(2,'2026-10-03'),(3,'2026-10-03'),(4,'2026-10-03'),
(1,'2026-10-04'),(5,'2026-10-04');
```

### Key tables (for reference)

**`employees`** (same data as Phase 3, department as text)

| emp_id | name | department | salary |
|---|---|---|---|
| 1 | Amit Sharma | IT | 90000 |
| 2 | Priya Singh | HR | 52000 |
| 3 | Rahul Verma | IT | 68000 |
| 4 | Sneha Iyer | Finance | 82000 |
| 5 | Karan Mehta | IT | 60000 |
| 6 | Neha Gupta | HR | 58000 |
| 7 | Vikram Rao | Finance | 95000 |
| 8 | Anita Desai | Sales | 45000 |
| 9 | Rohan Das | IT | 68000 |
| 10 | Meera Nair | Finance | 82000 |

Ties on purpose: **68000** (Rahul, Rohan) and **82000** (Sneha, Meera).

**`daily_revenue`**

| sale_date | revenue |
|---|---|
| 2026-10-01 | 100 |
| 2026-10-02 | 150 |
| 2026-10-03 | 200 |
| 2026-10-04 | 120 |
| 2026-10-05 | 180 |
| 2026-10-06 | 90 |
| 2026-10-07 | 210 |

**`monthly_revenue`**

| month_start | revenue |
|---|---|
| 2026-01-01 | 1000 |
| 2026-02-01 | 1200 |
| 2026-03-01 | 900 |
| 2026-04-01 | 1350 |
| 2026-05-01 | 1350 |
| 2026-06-01 | 1500 |

**`orders`**

| order_id | customer_id | order_date | amount |
|---|---|---|---|
| 101 | 1 | 2026-01-05 | 500 |
| 102 | 1 | 2026-02-10 | 700 |
| 103 | 2 | 2026-01-20 | 300 |
| 104 | 3 | 2026-02-15 | 900 |
| 105 | 2 | 2026-03-01 | 450 |
| 106 | 1 | 2026-03-12 | 200 |
| 107 | 4 | 2026-03-18 | 650 |

**`contacts`**

| id | email | created_at |
|---|---|---|
| 1 | a@x.com | 2026-01-01 |
| 2 | b@x.com | 2026-01-02 |
| 3 | a@x.com | 2026-01-05 |
| 4 | c@x.com | 2026-01-06 |
| 5 | b@x.com | 2026-01-07 |
| 6 | a@x.com | 2026-01-09 |

**`user_logins`**

| user_id | login_date |
|---|---|
| 1 | 2026-10-01 |
| 1 | 2026-10-02 |
| 1 | 2026-10-03 |
| 1 | 2026-10-05 |
| 1 | 2026-10-06 |
| 1 | 2026-10-09 |
| 2 | 2026-10-01 |
| 2 | 2026-10-03 |
| 2 | 2026-10-04 |
| 2 | 2026-10-05 |

**`logs`**: (id, num) = (1,1), (2,1), (3,1), (4,2), (5,1), (6,2), (7,2)

**`scores`**: (id, score) = (1,3.50), (2,3.65), (3,4.00), (4,3.85), (5,4.00), (6,3.65)

**`events`**

| user_id | event_time |
|---|---|
| 1 | 2026-10-21 10:00:00 |
| 1 | 2026-10-21 10:10:00 |
| 1 | 2026-10-21 10:25:00 |
| 1 | 2026-10-21 11:30:00 |
| 1 | 2026-10-21 11:40:00 |
| 1 | 2026-10-21 14:00:00 |
| 2 | 2026-10-21 09:00:00 |
| 2 | 2026-10-21 09:50:00 |
| 2 | 2026-10-21 10:15:00 |

**`visits`**

| user_id | visit_date |
|---|---|
| 1 | 2026-10-01 |
| 2 | 2026-10-01 |
| 1 | 2026-10-02 |
| 3 | 2026-10-02 |
| 2 | 2026-10-03 |
| 3 | 2026-10-03 |
| 4 | 2026-10-03 |
| 1 | 2026-10-04 |
| 5 | 2026-10-04 |

---

# PART 2: MODULE 9: Window Function Fundamentals

## 9.1 Syntax Anatomy

```sql
function_name(arguments) OVER (
    PARTITION BY col1, col2        -- optional: split rows into groups
    ORDER BY col3 [ASC|DESC]       -- optional: order inside each group
    frame_clause                   -- optional: which rows around the current row
)
```

| Part | Meaning | If omitted |
|---|---|---|
| `function_name(...)` | what to calculate (`SUM`, `RANK`, `LAG`, ...) | required |
| `OVER (...)` | marks it as a window function | required |
| `PARTITION BY` | like `GROUP BY` but **does not collapse rows**; calculation restarts for each partition | the **whole result** is one partition |
| `ORDER BY` | order of rows inside the partition (needed for ranking, running totals, LAG/LEAD) | rows are unordered; aggregates cover the entire partition |
| frame clause | the exact slice of rows (relative to the current row) the function sees | depends on the function and on whether `ORDER BY` exists (see 9.7) |

**Three kinds of window functions:**

| Kind | Functions |
|---|---|
| **Ranking** | `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, `PERCENT_RANK`, `CUME_DIST` |
| **Value (navigation)** | `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE` |
| **Aggregate** | `SUM`, `AVG`, `COUNT`, `MIN`, `MAX` used with `OVER` |

---

## 9.2 Ranking Functions

### ROW_NUMBER, RANK, DENSE_RANK (the big three)

| Function | Ties | Example for values 95, 90, 82, 82, 68 |
|---|---|---|
| `ROW_NUMBER()` | never ties; assigns 1, 2, 3, ... (tie order is arbitrary unless you add a tie-breaker) | 1, 2, 3, 4, 5 |
| `RANK()` | ties get the **same** rank, then **skips** numbers | 1, 2, 3, 3, 5 |
| `DENSE_RANK()` | ties get the same rank, **no gaps** | 1, 2, 3, 3, 4 |

**Whole company, highest salary first:**

```sql
SELECT name, salary,
       ROW_NUMBER() OVER (ORDER BY salary DESC, emp_id) AS row_num,
       RANK()       OVER (ORDER BY salary DESC)         AS rnk,
       DENSE_RANK() OVER (ORDER BY salary DESC)         AS dense_rnk
FROM employees
ORDER BY salary DESC, emp_id;
```

| name | salary | row_num | rnk | dense_rnk |
|---|---|---|---|---|
| Vikram Rao | 95000 | 1 | 1 | 1 |
| Amit Sharma | 90000 | 2 | 2 | 2 |
| Sneha Iyer | 82000 | 3 | 3 | 3 |
| Meera Nair | 82000 | 4 | 3 | 3 |
| Rahul Verma | 68000 | 5 | 5 | 4 |
| Rohan Das | 68000 | 6 | 5 | 4 |
| Karan Mehta | 60000 | 7 | 7 | 5 |
| Neha Gupta | 58000 | 8 | 8 | 6 |
| Priya Singh | 52000 | 9 | 9 | 7 |
| Anita Desai | 45000 | 10 | 10 | 8 |

**Reading it:**
- Sneha and Meera tie at 82000. `ROW_NUMBER` still separates them (using `emp_id` as tie-breaker). `RANK` gives both 3, and the next person gets **5** (4 is skipped). `DENSE_RANK` gives both 3, and the next gets **4**.
- **Always add a tie-breaker** (`, emp_id`) to `ROW_NUMBER` for deterministic results.

**Which one to use?**

| Interview wording | Use |
|---|---|
| "Nth highest **distinct** salary", "top 3 salaries (ties share)" | `DENSE_RANK` |
| "Top N **rows**, exactly N per group", "pick one row", "remove duplicates" | `ROW_NUMBER` |
| "Competition ranking: ties share the rank, next is skipped" | `RANK` |

### PARTITION BY: ranking restarts in each group

```sql
SELECT department, name, salary,
       ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC, emp_id) AS row_num,
       RANK()       OVER (PARTITION BY department ORDER BY salary DESC)         AS rnk,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC)         AS dense_rnk
FROM employees
ORDER BY department, salary DESC, emp_id;
```

| department | name | salary | row_num | rnk | dense_rnk |
|---|---|---|---|---|---|
| Finance | Vikram Rao | 95000 | 1 | 1 | 1 |
| Finance | Sneha Iyer | 82000 | 2 | 2 | 2 |
| Finance | Meera Nair | 82000 | 3 | 2 | 2 |
| HR | Neha Gupta | 58000 | 1 | 1 | 1 |
| HR | Priya Singh | 52000 | 2 | 2 | 2 |
| IT | Amit Sharma | 90000 | 1 | 1 | 1 |
| IT | Rahul Verma | 68000 | 2 | 2 | 2 |
| IT | Rohan Das | 68000 | 3 | 2 | 2 |
| IT | Karan Mehta | 60000 | 4 | 4 | 3 |
| Sales | Anita Desai | 45000 | 1 | 1 | 1 |

**Reading it:** The numbering restarts at 1 for each department. In IT, Rahul and Rohan tie: `RANK` = 2, 2 and then Karan gets 4; `DENSE_RANK` gives Karan 3.

### NTILE(n): split rows into n roughly equal buckets

```sql
SELECT name, salary,
       NTILE(4) OVER (ORDER BY salary DESC, emp_id) AS quartile
FROM employees
ORDER BY salary DESC, emp_id;
```

| name | salary | quartile |
|---|---|---|
| Vikram Rao | 95000 | 1 |
| Amit Sharma | 90000 | 1 |
| Sneha Iyer | 82000 | 1 |
| Meera Nair | 82000 | 2 |
| Rahul Verma | 68000 | 2 |
| Rohan Das | 68000 | 2 |
| Karan Mehta | 60000 | 3 |
| Neha Gupta | 58000 | 3 |
| Priya Singh | 52000 | 4 |
| Anita Desai | 45000 | 4 |

**Reading it:** 10 rows cannot split evenly into 4. The **extra rows go to the first buckets**: sizes are 3, 3, 2, 2. Quartile 1 = top earners.

### PERCENT_RANK and CUME_DIST

| Function | Formula | Meaning |
|---|---|---|
| `PERCENT_RANK()` | `(rank - 1) / (total_rows - 1)` | relative position, from 0 to 1 |
| `CUME_DIST()` | `rows with value <= current / total_rows` | fraction of rows at or below the current value |

```sql
SELECT name, salary,
       ROUND(PERCENT_RANK() OVER (ORDER BY salary), 4) AS pct_rank,
       ROUND(CUME_DIST()    OVER (ORDER BY salary), 4) AS cume_dist
FROM employees
ORDER BY salary, emp_id;
```

| name | salary | pct_rank | cume_dist |
|---|---|---|---|
| Anita Desai | 45000 | 0.0000 | 0.1000 |
| Priya Singh | 52000 | 0.1111 | 0.2000 |
| Neha Gupta | 58000 | 0.2222 | 0.3000 |
| Karan Mehta | 60000 | 0.3333 | 0.4000 |
| Rahul Verma | 68000 | 0.4444 | 0.6000 |
| Rohan Das | 68000 | 0.4444 | 0.6000 |
| Sneha Iyer | 82000 | 0.6667 | 0.8000 |
| Meera Nair | 82000 | 0.6667 | 0.8000 |
| Amit Sharma | 90000 | 0.8889 | 0.9000 |
| Vikram Rao | 95000 | 1.0000 | 1.0000 |

**Check for 68000:** `RANK` (ascending) is 5, so `PERCENT_RANK = (5-1)/(10-1) = 0.4444`. Six rows have salary <= 68000, so `CUME_DIST = 6/10 = 0.6`.

---

## 9.3 Value (Navigation) Functions

### LAG and LEAD

| Function | Meaning |
|---|---|
| `LAG(col, n, default)` | value from **n rows before** the current row (default n = 1) |
| `LEAD(col, n, default)` | value from **n rows after** the current row |

The third argument is what to return when no such row exists (otherwise NULL).

```sql
SELECT sale_date, revenue,
       LAG(revenue)  OVER (ORDER BY sale_date) AS prev_day,
       LEAD(revenue) OVER (ORDER BY sale_date) AS next_day
FROM daily_revenue;
```

| sale_date | revenue | prev_day | next_day |
|---|---|---|---|
| 2026-10-01 | 100 | NULL | 150 |
| 2026-10-02 | 150 | 100 | 200 |
| 2026-10-03 | 200 | 150 | 120 |
| 2026-10-04 | 120 | 200 | 180 |
| 2026-10-05 | 180 | 120 | 90 |
| 2026-10-06 | 90 | 180 | 210 |
| 2026-10-07 | 210 | 90 | NULL |

**Reading it:** The first row has no previous day (NULL). The last row has no next day (NULL). With `PARTITION BY`, LAG/LEAD never cross group boundaries.

With a default value and offset 2:

```sql
SELECT sale_date, revenue,
       LAG(revenue, 2, 0) OVER (ORDER BY sale_date) AS two_days_ago
FROM daily_revenue;
```

| sale_date | revenue | two_days_ago |
|---|---|---|
| 2026-10-01 | 100 | 0 |
| 2026-10-02 | 150 | 0 |
| 2026-10-03 | 200 | 100 |
| 2026-10-04 | 120 | 150 |
| 2026-10-05 | 180 | 200 |
| 2026-10-06 | 90 | 120 |
| 2026-10-07 | 210 | 180 |

### FIRST_VALUE, LAST_VALUE, NTH_VALUE

| Function | Returns |
|---|---|
| `FIRST_VALUE(col)` | the first value in the frame |
| `LAST_VALUE(col)` | the last value in the frame |
| `NTH_VALUE(col, n)` | the n-th value in the frame |

```sql
SELECT sale_date, revenue,
       FIRST_VALUE(revenue) OVER (ORDER BY sale_date) AS first_day_rev,
       LAST_VALUE(revenue)  OVER (ORDER BY sale_date) AS last_default,
       LAST_VALUE(revenue)  OVER (ORDER BY sale_date
                                  ROWS BETWEEN UNBOUNDED PRECEDING
                                           AND UNBOUNDED FOLLOWING) AS last_correct
FROM daily_revenue;
```

| sale_date | revenue | first_day_rev | last_default | last_correct |
|---|---|---|---|---|
| 2026-10-01 | 100 | 100 | 100 | 210 |
| 2026-10-02 | 150 | 100 | 150 | 210 |
| 2026-10-03 | 200 | 100 | 200 | 210 |
| 2026-10-04 | 120 | 100 | 120 | 210 |
| 2026-10-05 | 180 | 100 | 180 | 210 |
| 2026-10-06 | 90 | 100 | 90 | 210 |
| 2026-10-07 | 210 | 100 | 210 | 210 |

**The classic trap:** `last_default` just equals the current row's revenue. Why? When `ORDER BY` exists and no frame is written, the default frame ends at the **current row** (see 9.7), so the "last" row of the frame is the current row. To get the true last value, extend the frame to `UNBOUNDED FOLLOWING` (the `last_correct` column).

**NTH_VALUE example: second highest salary in each department** (NULL if the department has only one person):

```sql
SELECT DISTINCT department,
       NTH_VALUE(salary, 2) OVER (PARTITION BY department
                                  ORDER BY salary DESC
                                  ROWS BETWEEN UNBOUNDED PRECEDING
                                           AND UNBOUNDED FOLLOWING) AS second_salary
FROM employees
ORDER BY department;
```

| department | second_salary |
|---|---|
| Finance | 82000 |
| HR | 52000 |
| IT | 68000 |
| Sales | NULL |

(`NTH_VALUE` counts rows, not distinct values: in Finance the second row is Sneha at 82000.)

---

## 9.4 Aggregate Window Functions

Any aggregate (`SUM, AVG, COUNT, MIN, MAX`) becomes a window function when you add `OVER`.

```sql
SELECT name, department, salary,
       SUM(salary) OVER ()                            AS company_total,
       SUM(salary) OVER (PARTITION BY department)     AS dept_total,
       COUNT(*)    OVER (PARTITION BY department)     AS dept_headcount,
       MAX(salary) OVER (PARTITION BY department)     AS dept_max
FROM employees
WHERE department IN ('HR', 'Sales')
ORDER BY emp_id;
```

| name | department | salary | company_total | dept_total | dept_headcount | dept_max |
|---|---|---|---|---|---|---|
| Priya Singh | HR | 52000 | 155000 | 110000 | 2 | 58000 |
| Neha Gupta | HR | 58000 | 155000 | 110000 | 2 | 58000 |
| Anita Desai | Sales | 45000 | 155000 | 45000 | 1 | 45000 |

**Important:** `WHERE` filters rows **before** window functions run. So `company_total` here is the total of only the filtered rows (52000 + 58000 + 45000 = 155000), not the whole company (700000).

- `OVER ()` (empty) means the window is the **entire result set**.
- `OVER (PARTITION BY department)` means the window is the **department**.

### Combining GROUP BY and window functions

Window functions run **after** `GROUP BY`, so you can wrap an aggregate inside a window function (`SUM(SUM(x)) OVER ()`).

```sql
SELECT department,
       SUM(salary) AS dept_total,
       ROUND(100.0 * SUM(salary) / SUM(SUM(salary)) OVER (), 2) AS pct_of_company
FROM employees
GROUP BY department
ORDER BY department;
```

| department | dept_total | pct_of_company |
|---|---|---|
| Finance | 259000 | 37.00 |
| HR | 110000 | 15.71 |
| IT | 286000 | 40.86 |
| Sales | 45000 | 6.43 |

**Reading it:** First `GROUP BY` produces 4 rows. Then `SUM(SUM(salary)) OVER ()` adds those four totals = 700000, and each share is computed against that.

---

## 9.5 Frames: Which Rows Does the Function See?

The **frame** is a slice of the partition, defined **relative to the current row**.

```text
{ ROWS | RANGE } BETWEEN <start> AND <end>

<start> / <end> can be:
  UNBOUNDED PRECEDING   first row of the partition
  n PRECEDING           n rows before the current row
  CURRENT ROW           the current row
  n FOLLOWING           n rows after the current row
  UNBOUNDED FOLLOWING   last row of the partition
```

### Common frames

| Goal | Frame |
|---|---|
| Running total / cumulative | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |
| 3-row moving average | `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` |
| Centered moving average | `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING` |
| Whole partition (for `LAST_VALUE`) | `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| Remaining rows | `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` |

### Default frames (very important)

| `OVER` clause | Default frame |
|---|---|
| **No** `ORDER BY` | the **whole partition** |
| **With** `ORDER BY`, no frame written | `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |

> Ranking functions (`ROW_NUMBER`, `RANK`, ...) and `LAG/LEAD` ignore frames. Frames matter for aggregates, `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`.

### ROWS vs RANGE (the tie difference)

- `ROWS` counts **physical rows**.
- `RANGE` treats rows with the **same ORDER BY value (peers)** as one unit: they are all included together.

```sql
SELECT name, salary,
       SUM(salary) OVER (ORDER BY salary)                                      AS running_range,
       SUM(salary) OVER (ORDER BY salary, emp_id ROWS UNBOUNDED PRECEDING)     AS running_rows
FROM employees
ORDER BY salary, emp_id;
```

| name | salary | running_range | running_rows |
|---|---|---|---|
| Anita Desai | 45000 | 45000 | 45000 |
| Priya Singh | 52000 | 97000 | 97000 |
| Neha Gupta | 58000 | 155000 | 155000 |
| Karan Mehta | 60000 | 215000 | 215000 |
| Rahul Verma | 68000 | 351000 | 283000 |
| Rohan Das | 68000 | 351000 | 351000 |
| Sneha Iyer | 82000 | 515000 | 433000 |
| Meera Nair | 82000 | 515000 | 515000 |
| Amit Sharma | 90000 | 605000 | 605000 |
| Vikram Rao | 95000 | 700000 | 700000 |

**Reading it:**
- `running_range` (the default): Rahul and Rohan both have salary 68000, so they are "peers" and both see **both** 68000 rows: 215000 + 68000 + 68000 = **351000** for each.
- `running_rows`: strictly row by row: Rahul = 283000, then Rohan = 351000.

> **Rule for running totals:** use `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` explicitly, and add a unique tie-breaker to `ORDER BY`. This avoids surprises with duplicate dates or values. (It is also usually faster than `RANGE`.)

---

## 9.6 Named Windows (WINDOW clause)

If several functions share the same window, define it once.

```sql
SELECT name, department, salary,
       RANK()       OVER w AS rnk,
       DENSE_RANK() OVER w AS dense_rnk,
       SUM(salary)  OVER w AS running_dept_total
FROM employees
WHERE department = 'IT'
WINDOW w AS (PARTITION BY department ORDER BY salary DESC)
ORDER BY salary DESC, emp_id;
```

| name | department | salary | rnk | dense_rnk | running_dept_total |
|---|---|---|---|---|---|
| Amit Sharma | IT | 90000 | 1 | 1 | 90000 |
| Rahul Verma | IT | 68000 | 2 | 2 | 226000 |
| Rohan Das | IT | 68000 | 2 | 2 | 226000 |
| Karan Mehta | IT | 60000 | 4 | 3 | 286000 |

**Reading it:** The `WINDOW` clause goes **after** `WHERE`/`GROUP BY`/`HAVING` and **before** `ORDER BY`. (Supported in MySQL 8 and PostgreSQL; not in SQL Server.) Note that `running_dept_total` uses the default `RANGE` frame, so the tied 68000 rows both show 226000.

---

## 9.7 Filtering on Window Results (CTE or Subquery Required)

You **cannot** use a window function in `WHERE`, `GROUP BY`, or `HAVING`.

```sql
-- ERROR: window functions are not allowed in WHERE
SELECT name, salary
FROM employees
WHERE DENSE_RANK() OVER (ORDER BY salary DESC) <= 3;
```

**Why?** Logical order of execution:

```text
FROM -> WHERE -> GROUP BY -> HAVING -> WINDOW FUNCTIONS -> SELECT -> DISTINCT -> ORDER BY -> LIMIT
```

`WHERE` runs **before** the window functions exist. The fix: compute the window function in a CTE (or subquery), then filter in the outer query.

```sql
WITH ranked AS (
    SELECT name, salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT name, salary, rnk
FROM ranked
WHERE rnk <= 3
ORDER BY rnk, name;
```

| name | salary | rnk |
|---|---|---|
| Vikram Rao | 95000 | 1 |
| Amit Sharma | 90000 | 2 |
| Meera Nair | 82000 | 3 |
| Sneha Iyer | 82000 | 3 |

> (Some databases like Snowflake and BigQuery offer `QUALIFY` to filter directly on window results. MySQL and PostgreSQL do not.)

---

## 9.8 Performance Notes

- Window functions need rows **sorted by the partition and order columns**. An index on `(partition_col, order_col)` can help.
- Windows with the **same** `PARTITION BY` / `ORDER BY` can share one sort; many different windows mean many sorts.
- Filter early (`WHERE`) to shrink the data before windowing, **but remember** this changes what the window "sees."
- In big data engines (Spark, Hive), a window with a huge partition key (one value with billions of rows) causes **data skew**: one worker does all the work.
- Window functions replace most slow correlated subqueries and self-joins.

---

## 9.9 Module 9 Practice Problems

1. Show each employee's rank within their department by salary (ties share rank, no gaps).
2. Show each employee's salary as a percentage of their department's total salary.
3. Show each day's revenue and the previous day's revenue.
4. Show each employee, the department average, and the difference from the average.
5. Divide the employees into 3 salary tiers (NTILE) and show the tier.
6. For each customer, show every order and a running total of `amount` ordered by date.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT department, name, salary,
       DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees
ORDER BY department, dept_rank, emp_id;

-- 2  (Amit 31.47, Priya 47.27, Rahul 23.78, Sneha 31.66, Karan 20.98,
--     Neha 52.73, Vikram 36.68, Anita 100.00, Rohan 23.78, Meera 31.66)
SELECT name, department, salary,
       ROUND(100.0 * salary / SUM(salary) OVER (PARTITION BY department), 2) AS pct_of_dept
FROM employees
ORDER BY emp_id;

-- 3
SELECT sale_date, revenue,
       LAG(revenue) OVER (ORDER BY sale_date) AS prev_day
FROM daily_revenue;

-- 4
SELECT name, department, salary,
       ROUND(AVG(salary) OVER (PARTITION BY department), 2) AS dept_avg,
       ROUND(salary - AVG(salary) OVER (PARTITION BY department), 2) AS diff_from_avg
FROM employees
ORDER BY emp_id;

-- 5  Tier 1: Vikram, Amit, Sneha, Meera (4 rows)
--    Tier 2: Rahul, Rohan, Karan (3 rows)
--    Tier 3: Neha, Priya, Anita (3 rows)
SELECT name, salary,
       NTILE(3) OVER (ORDER BY salary DESC, emp_id) AS tier
FROM employees
ORDER BY salary DESC, emp_id;

-- 6  (customer 1: 500, 1200, 1400 | customer 2: 300, 750 | customer 3: 900 | customer 4: 650)
SELECT customer_id, order_id, order_date, amount,
       SUM(amount) OVER (PARTITION BY customer_id
                         ORDER BY order_date, order_id
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders
ORDER BY customer_id, order_date;
```
</details>

---

# PART 3: MODULE 10: Interview Patterns (P20 to P33)

For each pattern: **when to use it**, **template**, **example**, **output**, **explanation**.

---

## P20: Top N Per Group

**Use when:** "top 3 earners in each department", "most recent 2 orders per customer", "best product per category".

**Template:**

```sql
WITH ranked AS (
    SELECT cols...,
           DENSE_RANK() OVER (PARTITION BY group_col ORDER BY metric DESC) AS rnk
    FROM t
)
SELECT * FROM ranked WHERE rnk <= N;
```

**Example A: top 2 distinct salary levels per department (ties included) using `DENSE_RANK`**

```sql
WITH ranked AS (
    SELECT department, name, salary,
           DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT department, name, salary, rnk
FROM ranked
WHERE rnk <= 2
ORDER BY department, rnk, name;
```

| department | name | salary | rnk |
|---|---|---|---|
| Finance | Vikram Rao | 95000 | 1 |
| Finance | Meera Nair | 82000 | 2 |
| Finance | Sneha Iyer | 82000 | 2 |
| HR | Neha Gupta | 58000 | 1 |
| HR | Priya Singh | 52000 | 2 |
| IT | Amit Sharma | 90000 | 1 |
| IT | Rahul Verma | 68000 | 2 |
| IT | Rohan Das | 68000 | 2 |
| Sales | Anita Desai | 45000 | 1 |

(9 rows: ties at rank 2 are all kept, so Finance and IT return 3 rows.)

**Example B: exactly 2 rows per department using `ROW_NUMBER`**

```sql
WITH ranked AS (
    SELECT department, name, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC, emp_id) AS rn
    FROM employees
)
SELECT department, name, salary, rn
FROM ranked
WHERE rn <= 2
ORDER BY department, rn;
```

| department | name | salary | rn |
|---|---|---|---|
| Finance | Vikram Rao | 95000 | 1 |
| Finance | Sneha Iyer | 82000 | 2 |
| HR | Neha Gupta | 58000 | 1 |
| HR | Priya Singh | 52000 | 2 |
| IT | Amit Sharma | 90000 | 1 |
| IT | Rahul Verma | 68000 | 2 |
| Sales | Anita Desai | 45000 | 1 |

(7 rows: exactly 2 per department, and the tie-breaker `emp_id` decided that Sneha beats Meera.)

> **Decision:** "Top 3 salaries" (LeetCode *Department Top Three Salaries*) means distinct salary levels, so use `DENSE_RANK`. "Top 3 employees" with exactly 3 rows, use `ROW_NUMBER`. **Ask the interviewer.**

---

## P21: Nth Highest / Lowest (Overall and Per Group)

**Use when:** "second highest salary", "3rd highest per department".

**Overall 3rd highest salary:**

```sql
WITH ranked AS (
    SELECT name, salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT name, salary
FROM ranked
WHERE rnk = 3
ORDER BY name;
```

| name | salary |
|---|---|
| Meera Nair | 82000 |
| Sneha Iyer | 82000 |

**Second highest salary per department:**

```sql
WITH ranked AS (
    SELECT department, name, salary,
           DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT department, name, salary
FROM ranked
WHERE rnk = 2
ORDER BY department, name;
```

| department | name | salary |
|---|---|---|
| Finance | Meera Nair | 82000 |
| Finance | Sneha Iyer | 82000 |
| HR | Priya Singh | 52000 |
| IT | Rahul Verma | 68000 |
| IT | Rohan Das | 68000 |

(Sales has only one salary level, so it returns nothing.)

**LeetCode-style "return NULL if there is no Nth salary" (value only):**

```sql
SELECT (SELECT DISTINCT salary
        FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
              FROM employees) t
        WHERE rnk = 10) AS tenth_highest;
```

| tenth_highest |
|---|
| NULL |

(A scalar subquery with no rows returns NULL.)

---

## P22: Deduplication (Keep One Row per Key)

**Use when:** "remove duplicate emails/users, keep the latest (or earliest) record".

**Template:**

```sql
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY dup_key ORDER BY created_at DESC, id DESC) AS rn
    FROM t
)
SELECT * FROM ranked WHERE rn = 1;
```

**Example: keep the latest contact per email**

```sql
WITH ranked AS (
    SELECT id, email, created_at,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC, id DESC) AS rn
    FROM contacts
)
SELECT id, email, created_at
FROM ranked
WHERE rn = 1
ORDER BY id;
```

| id | email | created_at |
|---|---|---|
| 4 | c@x.com | 2026-01-06 |
| 5 | b@x.com | 2026-01-07 |
| 6 | a@x.com | 2026-01-09 |

**Step-by-step:**

| id | email | created_at | rn (latest first) |
|---|---|---|---|
| 6 | a@x.com | 2026-01-09 | 1 |
| 3 | a@x.com | 2026-01-05 | 2 |
| 1 | a@x.com | 2026-01-01 | 3 |
| 5 | b@x.com | 2026-01-07 | 1 |
| 2 | b@x.com | 2026-01-02 | 2 |
| 4 | c@x.com | 2026-01-06 | 1 |

Keep `rn = 1`, discard the rest.

**Deleting duplicates (keep the earliest record per email):**

```sql
DELETE FROM contacts
WHERE id IN (
    SELECT id FROM (
        SELECT id,
               ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at, id) AS rn
        FROM contacts
    ) t
    WHERE rn > 1
);
```

Rows deleted: ids **3** and **6** (duplicates of `a@x.com`) and **5** (duplicate of `b@x.com`). Remaining: ids 1, 2, 4. (The extra derived table `t` is required in MySQL, which does not allow deleting from a table you select from directly.)

> **Tip:** First run the `SELECT` version to preview exactly which rows will be deleted.

---

## P23: Running Totals (Cumulative Sums)

**Use when:** "cumulative sales", "balance after each transaction", "running count".

**Template:**

```sql
SUM(x) OVER (PARTITION BY grp ORDER BY dt, id ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

**Example: running revenue**

```sql
SELECT sale_date, revenue,
       SUM(revenue) OVER (ORDER BY sale_date
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM daily_revenue;
```

| sale_date | revenue | running_total |
|---|---|---|
| 2026-10-01 | 100 | 100 |
| 2026-10-02 | 150 | 250 |
| 2026-10-03 | 200 | 450 |
| 2026-10-04 | 120 | 570 |
| 2026-10-05 | 180 | 750 |
| 2026-10-06 | 90 | 840 |
| 2026-10-07 | 210 | 1050 |

**Running total per customer (restarts for each customer):**

```sql
SELECT customer_id, order_id, order_date, amount,
       SUM(amount) OVER (PARTITION BY customer_id
                         ORDER BY order_date, order_id
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders
ORDER BY customer_id, order_date;
```

| customer_id | order_id | order_date | amount | running_total |
|---|---|---|---|---|
| 1 | 101 | 2026-01-05 | 500 | 500 |
| 1 | 102 | 2026-02-10 | 700 | 1200 |
| 1 | 106 | 2026-03-12 | 200 | 1400 |
| 2 | 103 | 2026-01-20 | 300 | 300 |
| 2 | 105 | 2026-03-01 | 450 | 750 |
| 3 | 104 | 2026-02-15 | 900 | 900 |
| 4 | 107 | 2026-03-18 | 650 | 650 |

**Variation: running count and "first time the total crosses a threshold":**

```sql
WITH t AS (
    SELECT sale_date, revenue,
           SUM(revenue) OVER (ORDER BY sale_date
                              ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
    FROM daily_revenue
)
SELECT sale_date, running_total
FROM t
WHERE running_total >= 500
ORDER BY sale_date
LIMIT 1;
```

| sale_date | running_total |
|---|---|
| 2026-10-04 | 570 |

---

## P24: Moving Averages (Rolling Windows)

**Use when:** "7-day rolling average", "3-month moving sum".

**Template:** `AVG(x) OVER (ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)`

**Example: 3-day moving average**

```sql
SELECT sale_date, revenue,
       ROUND(AVG(revenue) OVER (ORDER BY sale_date
                                ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3d,
       COUNT(*) OVER (ORDER BY sale_date
                      ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS days_in_window
FROM daily_revenue;
```

| sale_date | revenue | moving_avg_3d | days_in_window |
|---|---|---|---|
| 2026-10-01 | 100 | 100.00 | 1 |
| 2026-10-02 | 150 | 125.00 | 2 |
| 2026-10-03 | 200 | 150.00 | 3 |
| 2026-10-04 | 120 | 156.67 | 3 |
| 2026-10-05 | 180 | 166.67 | 3 |
| 2026-10-06 | 90 | 130.00 | 3 |
| 2026-10-07 | 210 | 160.00 | 3 |

**Step-by-step (2026-10-04):** window = 10-02, 10-03, 10-04 = (150 + 200 + 120) / 3 = **156.67**.

**Reading it:** The first two rows have fewer than 3 days available, so they average what exists. If the interview requires "only when a full 3-day window exists", filter with `days_in_window = 3` (or compare `ROW_NUMBER() >= 3`).

> **Caution:** `ROWS` counts rows, not calendar days. If dates are missing (gaps), "6 PRECEDING" is not 7 calendar days. For real calendar logic, first fill missing dates (Phase 3, calendar CTE) or use `RANGE BETWEEN INTERVAL '6' DAY PRECEDING AND CURRENT ROW` (PostgreSQL 11+ / MySQL 8 support interval ranges on date columns).

---

## P25: Previous / Next Row Comparison (LAG / LEAD)

**Use when:** day-over-day or month-over-month change, "increase compared to previous", time between events.

**Example A: month-over-month growth %**

```sql
SELECT month_start, revenue,
       LAG(revenue) OVER (ORDER BY month_start) AS prev_revenue,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month_start))
             / LAG(revenue) OVER (ORDER BY month_start), 2) AS mom_growth_pct
FROM monthly_revenue;
```

| month_start | revenue | prev_revenue | mom_growth_pct |
|---|---|---|---|
| 2026-01-01 | 1000 | NULL | NULL |
| 2026-02-01 | 1200 | 1000 | 20.00 |
| 2026-03-01 | 900 | 1200 | -25.00 |
| 2026-04-01 | 1350 | 900 | 50.00 |
| 2026-05-01 | 1350 | 1350 | 0.00 |
| 2026-06-01 | 1500 | 1350 | 11.11 |

**Step-by-step (March):** (900 - 1200) / 1200 = -0.25 = **-25.00%**.

> **Year-over-year:** use `LAG(revenue, 12) OVER (ORDER BY month_start)` when data has one row per month with no missing months.

**Example B: days since the customer's previous order**

```sql
SELECT order_id, customer_id, order_date,
       LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_date,
       DATEDIFF(order_date,
                LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)) AS days_since_prev
FROM orders
ORDER BY customer_id, order_date;
```

(MySQL: `DATEDIFF(later, earlier)`. PostgreSQL: `order_date - LAG(order_date) OVER (...)`.)

| order_id | customer_id | order_date | prev_order_date | days_since_prev |
|---|---|---|---|---|
| 101 | 1 | 2026-01-05 | NULL | NULL |
| 102 | 1 | 2026-02-10 | 2026-01-05 | 36 |
| 106 | 1 | 2026-03-12 | 2026-02-10 | 30 |
| 103 | 2 | 2026-01-20 | NULL | NULL |
| 105 | 2 | 2026-03-01 | 2026-01-20 | 40 |
| 104 | 3 | 2026-02-15 | NULL | NULL |
| 107 | 4 | 2026-03-18 | NULL | NULL |

**Example C: "days when revenue was higher than both the previous and next day" (peaks)**

```sql
WITH t AS (
    SELECT sale_date, revenue,
           LAG(revenue)  OVER (ORDER BY sale_date) AS prev_rev,
           LEAD(revenue) OVER (ORDER BY sale_date) AS next_rev
    FROM daily_revenue
)
SELECT sale_date, revenue
FROM t
WHERE revenue > prev_rev AND revenue > next_rev
ORDER BY sale_date;
```

| sale_date | revenue |
|---|---|
| 2026-10-03 | 200 |
| 2026-10-05 | 180 |

(10-01 and 10-07 have a NULL neighbor, so the comparison is unknown and they are excluded.)

---

## P26: Percent of Total / Share of Group

**Use when:** "contribution of each employee to department payroll", "market share".

**Template:** `100.0 * x / SUM(x) OVER (PARTITION BY grp)`

```sql
SELECT name, department, salary,
       ROUND(100.0 * salary / SUM(salary) OVER (PARTITION BY department), 2) AS pct_of_dept
FROM employees
ORDER BY emp_id;
```

| name | department | salary | pct_of_dept |
|---|---|---|---|
| Amit Sharma | IT | 90000 | 31.47 |
| Priya Singh | HR | 52000 | 47.27 |
| Rahul Verma | IT | 68000 | 23.78 |
| Sneha Iyer | Finance | 82000 | 31.66 |
| Karan Mehta | IT | 60000 | 20.98 |
| Neha Gupta | HR | 58000 | 52.73 |
| Vikram Rao | Finance | 95000 | 36.68 |
| Anita Desai | Sales | 45000 | 100.00 |
| Rohan Das | IT | 68000 | 23.78 |
| Meera Nair | Finance | 82000 | 31.66 |

**Step-by-step (Amit):** IT total = 286000, so 90000 / 286000 = 31.47%. (Rounded values may add to 100.01; that is only rounding.)

---

## P27: Median and Percentiles

**Use when:** "median salary", "p90 response time", "find median".

### Median without `PERCENTILE_CONT` (works in MySQL 8 and PostgreSQL)

Idea: number the rows in sorted order, count them, and average the middle one or two rows.

```sql
WITH ranked AS (
    SELECT salary,
           ROW_NUMBER() OVER (ORDER BY salary) AS rn,
           COUNT(*)     OVER ()                AS cnt
    FROM employees
)
SELECT AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0));
```

| median_salary |
|---|
| 68000.00 |

**Step-by-step:** Sorted salaries: 45000, 52000, 58000, 60000, **68000, 68000**, 82000, 82000, 90000, 95000. `cnt = 10`, so `(10+1)/2.0 = 5.5`: `FLOOR = 5`, `CEIL = 6`. Rows 5 and 6 are 68000 and 68000, average = **68000**. (For an odd count like 3, both FLOOR and CEIL give the same row, so the "average of one row" is that row.)

> Use `2.0` (a decimal), not `2`, to avoid integer division in PostgreSQL / SQL Server.

### Median per department

```sql
WITH ranked AS (
    SELECT department, salary,
           ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary) AS rn,
           COUNT(*)     OVER (PARTITION BY department)                AS cnt
    FROM employees
)
SELECT department, AVG(salary) AS median_salary
FROM ranked
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0))
GROUP BY department
ORDER BY department;
```

| department | median_salary |
|---|---|
| Finance | 82000.00 |
| HR | 55000.00 |
| IT | 68000.00 |
| Sales | 45000.00 |

**Check:** IT sorted: 60000, 68000, 68000, 90000 gives the middle two = 68000 and 68000. HR: (52000 + 58000) / 2 = 55000. Finance: 82000, 82000, 95000 gives the middle = 82000.

### PostgreSQL shortcut

```sql
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;     -- 68000
```

`PERCENTILE_CONT(p)` interpolates between values; `PERCENTILE_DISC(p)` returns an actual value. (MySQL has no built-in percentile function: use the row-number method or `NTILE` / `PERCENT_RANK`.)

### Approximate percentile with NTILE

`NTILE(100)` buckets rows into percentiles; the bucket boundaries approximate p-th percentile values. Good for large data when exactness is not required.

---

## P28: Gaps and Islands (Consecutive Sequences)  *** Top Interview Pattern ***

**What is it?** Find **runs of consecutive values** (the islands) separated by **breaks** (the gaps). Examples: consecutive login days, consecutive IDs, streaks, consecutive available seats, continuous periods of activity.

### Technique A: value minus ROW_NUMBER

**Key insight:** In a run of consecutive dates, `date - row_number` stays **constant**. When there is a gap, it jumps to a new constant. That constant becomes the **group id** for the island.

Step table for **user 1** (dates in October):

| login_date | rn | login_date minus rn days (group key) |
|---|---|---|
| 2026-10-01 | 1 | 2026-09-30 |
| 2026-10-02 | 2 | 2026-09-30 |
| 2026-10-03 | 3 | 2026-09-30 |
| 2026-10-05 | 4 | 2026-10-01 |
| 2026-10-06 | 5 | 2026-10-01 |
| 2026-10-09 | 6 | 2026-10-03 |

Three different keys mean three islands. Now the full query (MySQL):

```sql
WITH dedup AS (
    SELECT DISTINCT user_id, login_date
    FROM user_logins
),
numbered AS (
    SELECT user_id, login_date,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) AS rn
    FROM dedup
),
grouped AS (
    SELECT user_id, login_date,
           DATE_SUB(login_date, INTERVAL rn DAY) AS grp
    FROM numbered
)
SELECT user_id,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end,
       COUNT(*)        AS streak_days
FROM grouped
GROUP BY user_id, grp
ORDER BY user_id, streak_start;
```

PostgreSQL: replace the `grp` expression with `login_date - CAST(rn AS INT)`.

| user_id | streak_start | streak_end | streak_days |
|---|---|---|---|
| 1 | 2026-10-01 | 2026-10-03 | 3 |
| 1 | 2026-10-05 | 2026-10-06 | 2 |
| 1 | 2026-10-09 | 2026-10-09 | 1 |
| 2 | 2026-10-01 | 2026-10-01 | 1 |
| 2 | 2026-10-03 | 2026-10-05 | 3 |

**Why `dedup` first?** If a user logs in twice on the same day, `ROW_NUMBER` would advance without the date advancing and break the trick. Always make (key, date) unique first.

**Longest streak per user:**

```sql
-- append to the query above, wrapping it as a CTE named streaks
SELECT user_id, MAX(streak_days) AS longest_streak
FROM streaks
GROUP BY user_id;
```

| user_id | longest_streak |
|---|---|
| 1 | 3 |
| 2 | 3 |

### Technique B: LAG + flag + cumulative sum (more flexible)

**Idea:** Mark the **first row of every island** with 1 (when there is no previous row, or the previous row is not "consecutive"), then take a running sum of the flags. The running sum is the island number.

```sql
WITH prev AS (
    SELECT user_id, login_date,
           LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS prev_date
    FROM (SELECT DISTINCT user_id, login_date FROM user_logins) d
),
flagged AS (
    SELECT user_id, login_date, prev_date,
           CASE WHEN prev_date IS NULL OR DATEDIFF(login_date, prev_date) > 1
                THEN 1 ELSE 0 END AS new_island
    FROM prev
),
islands AS (
    SELECT user_id, login_date,
           SUM(new_island) OVER (PARTITION BY user_id ORDER BY login_date) AS island_id
    FROM flagged
)
SELECT user_id, island_id,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end,
       COUNT(*)        AS streak_days
FROM islands
GROUP BY user_id, island_id
ORDER BY user_id, island_id;
```

Step table for **user 1**:

| login_date | prev_date | new_island | island_id |
|---|---|---|---|
| 2026-10-01 | NULL | 1 | 1 |
| 2026-10-02 | 2026-10-01 | 0 | 1 |
| 2026-10-03 | 2026-10-02 | 0 | 1 |
| 2026-10-05 | 2026-10-03 | 1 (gap of 2 days) | 2 |
| 2026-10-06 | 2026-10-05 | 0 | 2 |
| 2026-10-09 | 2026-10-06 | 1 (gap of 3 days) | 3 |

Final output: the same 5 streaks as Technique A.

**Why learn Technique B?** It works for any definition of "same island": a gap larger than N days, a change in a status column, a time gap (sessions), or "value changed from the previous row." Technique A only works for strictly consecutive integers/dates.

### Classic: Consecutive Numbers (LeetCode style)

**Problem:** find numbers that appear at least **3 times in a row** in `logs`.

`logs` = (1,1) (2,1) (3,1) (4,2) (5,1) (6,2) (7,2)

**Approach 1: LAG / LEAD (fixed length of 3)**

```sql
SELECT DISTINCT num AS consecutive_num
FROM (SELECT num,
             LAG(num)  OVER (ORDER BY id) AS prev_num,
             LEAD(num) OVER (ORDER BY id) AS next_num
      FROM logs) t
WHERE num = prev_num AND num = next_num;
```

| consecutive_num |
|---|
| 1 |

**Approach 2: islands (any length)**

```sql
WITH g AS (
    SELECT id, num,
           id - ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS grp
    FROM logs
)
SELECT DISTINCT num AS consecutive_num
FROM g
GROUP BY num, grp
HAVING COUNT(*) >= 3;
```

| consecutive_num |
|---|
| 1 |

**Step table (island keys):**

| id | num | rn within num | id - rn |
|---|---|---|---|
| 1 | 1 | 1 | 0 |
| 2 | 1 | 2 | 0 |
| 3 | 1 | 3 | 0 |
| 5 | 1 | 4 | 1 |
| 4 | 2 | 1 | 3 |
| 6 | 2 | 2 | 4 |
| 7 | 2 | 3 | 4 |

Groups: (num 1, key 0) has 3 rows (qualifies). (num 1, key 1) has 1 row. (num 2, key 3) has 1 row. (num 2, key 4) has 2 rows.

---

## P29: Sessionization

**Use when:** "group user events into sessions where a new session starts after 30 minutes of inactivity."

**Same idea as Technique B**, with a time gap instead of a date gap.

```sql
WITH prev AS (
    SELECT user_id, event_time,
           LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) AS prev_time
    FROM events
),
flagged AS (
    SELECT user_id, event_time,
           CASE WHEN prev_time IS NULL
                  OR TIMESTAMPDIFF(MINUTE, prev_time, event_time) > 30
                THEN 1 ELSE 0 END AS new_session
    FROM prev
),
sessions AS (
    SELECT user_id, event_time,
           SUM(new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_no
    FROM flagged
)
SELECT user_id, session_no,
       MIN(event_time) AS session_start,
       MAX(event_time) AS session_end,
       COUNT(*)        AS events,
       TIMESTAMPDIFF(MINUTE, MIN(event_time), MAX(event_time)) AS duration_min
FROM sessions
GROUP BY user_id, session_no
ORDER BY user_id, session_no;
```

PostgreSQL: replace the gap test with `EXTRACT(EPOCH FROM (event_time - prev_time)) / 60 > 30`.

| user_id | session_no | session_start | session_end | events | duration_min |
|---|---|---|---|---|---|
| 1 | 1 | 2026-10-21 10:00:00 | 2026-10-21 10:25:00 | 3 | 25 |
| 1 | 2 | 2026-10-21 11:30:00 | 2026-10-21 11:40:00 | 2 | 10 |
| 1 | 3 | 2026-10-21 14:00:00 | 2026-10-21 14:00:00 | 1 | 0 |
| 2 | 1 | 2026-10-21 09:00:00 | 2026-10-21 09:00:00 | 1 | 0 |
| 2 | 2 | 2026-10-21 09:50:00 | 2026-10-21 10:15:00 | 2 | 25 |

**Step table for user 1:**

| event_time | prev_time | gap (min) | new_session | session_no |
|---|---|---|---|---|
| 10:00 | NULL | n/a | 1 | 1 |
| 10:10 | 10:00 | 10 | 0 | 1 |
| 10:25 | 10:10 | 15 | 0 | 1 |
| 11:30 | 10:25 | 65 | 1 | 2 |
| 11:40 | 11:30 | 10 | 0 | 2 |
| 14:00 | 11:40 | 140 | 1 | 3 |

**Notes:** a gap of **exactly** 30 minutes stays in the same session here (`> 30`). Confirm the boundary rule (`>` vs `>=`) with the interviewer.

---

## P30: First / Last Event per User

**Use when:** "first purchase amount per customer", "last login", "status at the first event".

**Option 1: `ROW_NUMBER = 1`** (the most common; shown in P22). Example: the first order of each customer:

```sql
WITH t AS (
    SELECT customer_id, order_id, order_date, amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date, order_id) AS rn
    FROM orders
)
SELECT customer_id, order_id, order_date, amount
FROM t
WHERE rn = 1
ORDER BY customer_id;
```

| customer_id | order_id | order_date | amount |
|---|---|---|---|
| 1 | 101 | 2026-01-05 | 500 |
| 2 | 103 | 2026-01-20 | 300 |
| 3 | 104 | 2026-02-15 | 900 |
| 4 | 107 | 2026-03-18 | 650 |

**Option 2: `FIRST_VALUE` / `LAST_VALUE`** (good when you need first **and** last side by side):

```sql
SELECT DISTINCT customer_id,
       FIRST_VALUE(amount) OVER w AS first_amount,
       LAST_VALUE(amount)  OVER w AS last_amount
FROM orders
WINDOW w AS (PARTITION BY customer_id
             ORDER BY order_date, order_id
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
ORDER BY customer_id;
```

| customer_id | first_amount | last_amount |
|---|---|---|
| 1 | 500 | 200 |
| 2 | 300 | 450 |
| 3 | 900 | 900 |
| 4 | 650 | 650 |

**Reading it:** The full-partition frame is essential for `LAST_VALUE`. `DISTINCT` collapses the per-row repetition into one row per customer.

---

## P31: Rank With Ties and Tie-Breakers

**Classic: LeetCode *Rank Scores*:** rank scores from highest to lowest; ties share a rank; ranks are consecutive (no gaps).

```sql
SELECT score,
       DENSE_RANK() OVER (ORDER BY score DESC) AS score_rank
FROM scores
ORDER BY score DESC;
```

| score | score_rank |
|---|---|
| 4.00 | 1 |
| 4.00 | 1 |
| 3.85 | 2 |
| 3.65 | 3 |
| 3.65 | 3 |
| 3.50 | 4 |

**Compare with `RANK`:** it would give 1, 1, 3, 4, 4, 6 (gaps appear after ties).

**Tie-breakers** (when ranking must be unique and deterministic):

```sql
ROW_NUMBER() OVER (ORDER BY salary DESC, hire_date ASC, emp_id ASC)
```

Always finish `ORDER BY` with a unique column.

---

## P32: Difference From a Group Aggregate

**Use when:** "how far is each employee from the department average / top salary?"

```sql
SELECT name, department, salary,
       ROUND(AVG(salary) OVER (PARTITION BY department), 2)                    AS dept_avg,
       ROUND(salary - AVG(salary) OVER (PARTITION BY department), 2)           AS diff_from_avg,
       MAX(salary) OVER (PARTITION BY department) - salary                     AS gap_to_top
FROM employees
ORDER BY emp_id;
```

| name | department | salary | dept_avg | diff_from_avg | gap_to_top |
|---|---|---|---|---|---|
| Amit Sharma | IT | 90000 | 71500.00 | 18500.00 | 0 |
| Priya Singh | HR | 52000 | 55000.00 | -3000.00 | 6000 |
| Rahul Verma | IT | 68000 | 71500.00 | -3500.00 | 22000 |
| Sneha Iyer | Finance | 82000 | 86333.33 | -4333.33 | 13000 |
| Karan Mehta | IT | 60000 | 71500.00 | -11500.00 | 30000 |
| Neha Gupta | HR | 58000 | 55000.00 | 3000.00 | 0 |
| Vikram Rao | Finance | 95000 | 86333.33 | 8666.67 | 0 |
| Anita Desai | Sales | 45000 | 45000.00 | 0.00 | 0 |
| Rohan Das | IT | 68000 | 71500.00 | -3500.00 | 22000 |
| Meera Nair | Finance | 82000 | 86333.33 | -4333.33 | 13000 |

**Filter example: employees above their department average (window version of the Phase 3 correlated subquery):**

```sql
WITH t AS (
    SELECT name, department, salary,
           AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM employees
)
SELECT name, department, salary
FROM t
WHERE salary > dept_avg
ORDER BY name;
```

| name | department | salary |
|---|---|---|
| Amit Sharma | IT | 90000 |
| Neha Gupta | HR | 58000 |
| Vikram Rao | Finance | 95000 |

(One pass over the data instead of a subquery per row.)

---

## P33: Cumulative Distinct Counts (First-Seen Flag)

**Use when:** "running count of **unique** users", "cumulative new customers per day".

**Problem:** a plain running `COUNT(DISTINCT ...) OVER (...)` is not supported. Trick: flag each user's **first appearance** with `ROW_NUMBER() = 1`, then take a running sum of those flags.

```sql
WITH flagged AS (
    SELECT user_id, visit_date,
           CASE WHEN ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY visit_date) = 1
                THEN 1 ELSE 0 END AS is_new
    FROM (SELECT DISTINCT user_id, visit_date FROM visits) v
),
daily AS (
    SELECT visit_date, SUM(is_new) AS new_users
    FROM flagged
    GROUP BY visit_date
)
SELECT visit_date,
       new_users,
       SUM(new_users) OVER (ORDER BY visit_date
                            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_distinct_users
FROM daily
ORDER BY visit_date;
```

| visit_date | new_users | cumulative_distinct_users |
|---|---|---|
| 2026-10-01 | 2 | 2 |
| 2026-10-02 | 1 | 3 |
| 2026-10-03 | 1 | 4 |
| 2026-10-04 | 1 | 5 |

**Step-by-step:**
- 10-01: users 1 and 2 appear for the first time, so 2 new, 2 total.
- 10-02: user 1 is returning; user 3 is new, so 1 new, 3 total.
- 10-03: users 2 and 3 are returning; user 4 is new, so 1 new, 4 total.
- 10-04: user 1 is returning; user 5 is new, so 1 new, 5 total.

---

## Bonus: Choosing the Right Tool

| Task | Tool |
|---|---|
| One row per group (collapse) | `GROUP BY` |
| Keep every row + group info | aggregate window `OVER (PARTITION BY ...)` |
| Rank / top N | `ROW_NUMBER`, `RANK`, `DENSE_RANK` + CTE filter |
| Previous / next row | `LAG`, `LEAD` |
| Cumulative / rolling | `SUM/AVG OVER (ORDER BY ... ROWS ...)` |
| First / last in group | `ROW_NUMBER = 1`, or `FIRST_VALUE` / `LAST_VALUE` (full frame) |
| Streaks / consecutive | `value - ROW_NUMBER()` or `LAG` + flag + running `SUM` |
| Sessions | `LAG` time gap + flag + running `SUM` |
| Median / percentiles | `ROW_NUMBER + COUNT`, `PERCENTILE_CONT`, `NTILE` |
| Distinct running count | first-seen flag + running `SUM` |

---

# PART 4: Wrap-Up

## 4.1 Practice Problems for Module 10

1. Find the top 3 distinct salaries per department (with ties).
2. Find the 2nd highest salary in the whole company (return NULL if none).
3. Delete duplicate emails from `contacts`, keeping the earliest record. Which ids get deleted?
4. Show the 3-day moving average of revenue, but only for days that have a full 3-day window.
5. Show each employee's highest-paid colleague in the same department (name of the top earner) using `FIRST_VALUE`.
6. List all days where revenue was lower than the previous day.
7. Find each user's longest login streak and the dates of that streak.
8. Find numbers that appear at least 2 times consecutively in `logs`.
9. Compute the median salary of the company and of each department.
10. Show cumulative new users per day from `visits`.
11. Show the highest-paid employee of each department using `ROW_NUMBER` (ties broken by lowest `emp_id`).
12. For each customer, show the order with the largest `amount` and its rank among the customer's orders.

<details>
<summary>Answers</summary>

```sql
-- 1  (same output as P20 Example A for rnk <= 3; IT has only 3 levels, HR 2, Finance 2, Sales 1)
WITH r AS (
  SELECT department, name, salary,
         DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
  FROM employees)
SELECT department, name, salary, rnk FROM r WHERE rnk <= 3
ORDER BY department, rnk, name;

-- 2  (90000)
SELECT (SELECT DISTINCT salary FROM (
          SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk FROM employees) t
        WHERE rnk = 2) AS second_highest;

-- 3  deletes ids 3, 6 and 5
DELETE FROM contacts
WHERE id IN (SELECT id FROM (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at, id) AS rn
    FROM contacts) t WHERE rn > 1);

-- 4  rows for 10-03 to 10-07: 150.00, 156.67, 166.67, 130.00, 160.00
WITH t AS (
  SELECT sale_date, revenue,
         AVG(revenue) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS ma,
         COUNT(*)     OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS n
  FROM daily_revenue)
SELECT sale_date, ROUND(ma, 2) AS moving_avg_3d FROM t WHERE n = 3;

-- 5  (Amit for IT, Vikram for Finance, Neha for HR, Anita for Sales)
SELECT name, department, salary,
       FIRST_VALUE(name) OVER (PARTITION BY department ORDER BY salary DESC, emp_id) AS top_earner
FROM employees
ORDER BY emp_id;

-- 6  (10-04: 120 < 200, 10-06: 90 < 180)
WITH t AS (
  SELECT sale_date, revenue, LAG(revenue) OVER (ORDER BY sale_date) AS prev_rev
  FROM daily_revenue)
SELECT sale_date, revenue, prev_rev FROM t WHERE revenue < prev_rev;

-- 7  user 1: 2026-10-01 to 2026-10-03 (3 days); user 2: 2026-10-03 to 2026-10-05 (3 days)
--    (build `streaks` with Technique A, then:)
WITH streaks AS (
   /* Technique A query from P28 */
   SELECT user_id, streak_start, streak_end, streak_days FROM ...
),
ranked AS (
   SELECT *, ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY streak_days DESC, streak_start) AS rn
   FROM streaks)
SELECT user_id, streak_start, streak_end, streak_days FROM ranked WHERE rn = 1;

-- 8  (1 qualifies: ids 1-3; 2 qualifies: ids 6-7)
SELECT DISTINCT num FROM (
  SELECT num, LAG(num) OVER (ORDER BY id) AS p FROM logs) t
WHERE num = p;

-- 9  company 68000; Finance 82000, HR 55000, IT 68000, Sales 45000  (see P27)

-- 10 see P33 (2, 3, 4, 5 cumulative)

-- 11
WITH r AS (
  SELECT department, name, salary,
         ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC, emp_id) AS rn
  FROM employees)
SELECT department, name, salary FROM r WHERE rn = 1 ORDER BY department;
-- Finance Vikram 95000 | HR Neha 58000 | IT Amit 90000 | Sales Anita 45000

-- 12  customer 1 -> order 102 (700); 2 -> 105 (450); 3 -> 104 (900); 4 -> 107 (650)
WITH r AS (
  SELECT customer_id, order_id, amount,
         RANK() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rnk
  FROM orders)
SELECT customer_id, order_id, amount, rnk FROM r WHERE rnk = 1 ORDER BY customer_id;
```
(In answer 7, replace the placeholder CTE body with the full Technique A query from P28.)
</details>

## 4.2 Top Mistakes With Window Functions

| Mistake | Fix |
|---|---|
| Using a window function in `WHERE` / `HAVING` | Compute in a CTE/subquery, filter outside |
| `ROW_NUMBER` without a tie-breaker | Add a unique column at the end of `ORDER BY` |
| Using `RANK` when you need "Nth distinct value" | Use `DENSE_RANK` |
| `LAST_VALUE` returns the current row | Add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| Running total shows equal values for ties | Use `ROWS` (not the default `RANGE`) with a unique `ORDER BY` |
| Forgetting `PARTITION BY` (calculation spans all groups) | Add it whenever the logic is "per group" |
| Moving average over rows when dates have gaps | Fill missing dates first, or use interval `RANGE` |
| Gaps and islands on duplicate (key, date) rows | `DISTINCT` first |
| Dividing integers in percentages | Multiply by `100.0` and use `ROUND` |
| Assuming `WHERE` does not affect window results | `WHERE` runs first, so the window only sees the filtered rows |
| Median with `/ 2` integer division | Use `/ 2.0` |
| Using `COUNT(DISTINCT x) OVER (...)` | Not supported in most databases; use first-seen flag (P33) |
| Using window functions on MySQL 5.7 | Upgrade to 8.0+ or rewrite with variables/self-joins |

## 4.3 Window Functions Cheat Sheet

```text
SYNTAX     func(args) OVER (PARTITION BY a ORDER BY b [ROWS|RANGE BETWEEN x AND y])

RANKING    ROW_NUMBER()  unique 1,2,3,...            (dedupe, top N rows)
           RANK()        ties share, gaps after      (1,2,2,4)
           DENSE_RANK()  ties share, no gaps         (1,2,2,3)   (Nth highest distinct)
           NTILE(n)      n buckets
           PERCENT_RANK  (rank-1)/(rows-1)     CUME_DIST  rows<=current / rows

VALUE      LAG(x,n,def)  LEAD(x,n,def)  FIRST_VALUE(x)  LAST_VALUE(x)  NTH_VALUE(x,n)

AGGREGATE  SUM / AVG / COUNT / MIN / MAX (...) OVER (...)

FRAMES     running total : ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           moving avg 7d : ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
           full partition: ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
           DEFAULT       : no ORDER BY -> whole partition
                           with ORDER BY -> RANGE UNBOUNDED PRECEDING .. CURRENT ROW

ORDER OF EXECUTION
           FROM > WHERE > GROUP BY > HAVING > WINDOW > SELECT > DISTINCT > ORDER BY > LIMIT

PATTERNS
 Top N per group      : DENSE_RANK/ROW_NUMBER in CTE, filter rnk <= N
 Dedup                : ROW_NUMBER() PARTITION BY key ... keep rn = 1
 Running total        : SUM() OVER (ORDER BY ... ROWS UNBOUNDED PRECEDING)
 Moving average       : AVG() OVER (ORDER BY ... ROWS n PRECEDING)
 Compare to previous  : LAG()
 Streaks (islands)    : date - ROW_NUMBER()   OR   LAG + flag + running SUM
 Sessions             : LAG time gap > N -> flag -> running SUM
 Median               : ROW_NUMBER + COUNT, take middle one/two rows
 Distinct running cnt : first-seen flag (ROW_NUMBER = 1) + running SUM
```

## 4.4 Self-Test Questions

1. What is the difference between `GROUP BY` and a window function?
2. What do `PARTITION BY` and `ORDER BY` do inside `OVER`?
3. Differences between `ROW_NUMBER`, `RANK`, and `DENSE_RANK` for values 100, 90, 90, 80?
4. Why can't you write a window function in `WHERE`, and what is the workaround?
5. What is the default frame when `ORDER BY` is present? When it is absent?
6. Why does `LAST_VALUE` often return the current row's value, and how do you fix it?
7. `ROWS` vs `RANGE`: what changes when there are ties?
8. How do you compute a 7-day moving average? What goes wrong if some dates are missing?
9. Explain the gaps-and-islands trick (`date - ROW_NUMBER()`) in your own words.
10. How would you sessionize events with a 30-minute inactivity timeout?
11. How do you find the median without `PERCENTILE_CONT`?
12. How do you compute a cumulative count of **distinct** users?
13. Why must you add a unique tie-breaker to `ROW_NUMBER`'s `ORDER BY`?
14. Which of `RANK` / `DENSE_RANK` / `ROW_NUMBER` would you use for "3rd highest salary" and why?

## 4.5 LeetCode / Practice Checklist

Solve these (Database section), roughly in this order:

| Level | Problems |
|---|---|
| Medium | Second Highest Salary, Nth Highest Salary, Rank Scores, Consecutive Numbers, Department Top Three Salaries, Delete Duplicate Emails, Exchange Seats, Last Person to Fit in the Bus, Game Play Analysis III, Consecutive Available Seats |
| Hard / Medium-Hard | Human Traffic of Stadium, Report Contiguous Dates, Active Users (consecutive-day streak), Median Employee Salary, Find Median Given Frequency of Numbers |

Also practice on **StrataScratch** and **DataLemur** using the filters "window function" and "rank".

## 4.6 What to Do Next

1. Create all tables from Part 1 and run **every** query. Compare your output with the tables in this guide.
2. Re-solve each pattern **from memory** (template first, then the full query).
3. For each problem you solve, write the **tie behavior** and **NULL behavior** in your error log.
4. Solve 25 to 30 window-function problems (mix of Medium and Hard).
5. When ready, say **"Start Phase 5"** for **Date, Time and Analytics Patterns** (retention, cohorts, funnels, DAU/MAU, overlapping intervals). If you still want the earlier gap filled, say **"Start Phase 2"** for **Aggregation, Joins and Set Operations**.
