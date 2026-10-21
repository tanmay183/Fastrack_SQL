# SQL Phase 3: Subqueries, CTEs and Logic (Detailed Guide)

> Covers **Module 7 (Subqueries)** and **Module 8 (CTEs, including Recursive CTEs)**.
> Every topic has: easy explanation, example input table, query, output table, and step-by-step reasoning.
> Interview patterns covered: **P12 to P19** (from the syllabus).

---

## Table of Contents

- **Part 0:** Prerequisites (mini refresher of GROUP BY and JOIN)
- **Part 1:** Sample data (create once, use everywhere)
- **Part 2:** Module 7: Subqueries
- **Part 3:** Module 8: CTEs (normal and recursive)
- **Part 4:** Wrap-up (mistakes, cheat sheet, self-test, next step)

---

# PART 0: Prerequisites (Mini Refresher)

Phase 3 uses `GROUP BY`, `HAVING`, and `JOIN` lightly. If these are new to you, read this section first (they are taught fully in Phase 2). All queries here use the tables created in Part 1.

### GROUP BY (collapse many rows into one row per group)

```sql
SELECT dept_id,
       COUNT(*)      AS emp_count,
       AVG(salary)   AS avg_salary
FROM employees
GROUP BY dept_id
ORDER BY dept_id;
```

| dept_id | emp_count | avg_salary |
|---|---|---|
| 10 | 4 | 71500.00 |
| 20 | 2 | 55000.00 |
| 30 | 3 | 86333.33 |
| 40 | 1 | 45000.00 |

**Explanation:** Rows with the same `dept_id` are put in one bucket. Aggregate functions (`COUNT`, `AVG`) are computed per bucket. `HAVING` filters buckets (like `WHERE`, but after grouping).

### INNER JOIN (keep only matching rows from both tables)

```sql
SELECT e.name, d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE e.emp_id <= 3;
```

| name | dept_name |
|---|---|
| Amit Sharma | IT |
| Priya Singh | HR |
| Rahul Verma | IT |

### LEFT JOIN (keep ALL rows from the left table, NULL where no match)

```sql
SELECT d.dept_name, COUNT(e.emp_id) AS emp_count
FROM departments d
LEFT JOIN employees e ON e.dept_id = d.dept_id
GROUP BY d.dept_name
ORDER BY d.dept_name;
```

| dept_name | emp_count |
|---|---|
| Finance | 3 |
| HR | 2 |
| IT | 4 |
| Marketing | 0 |
| Sales | 1 |

**Explanation:** `Marketing` has no employees, but `LEFT JOIN` keeps it (with NULL on the employee side). `COUNT(e.emp_id)` counts only non-NULL values, so Marketing gets 0.

---

# PART 1: Sample Data (Create Once)

Works in **MySQL 8+** and **PostgreSQL**.

```sql
-- 1) departments
CREATE TABLE departments (
    dept_id    INT PRIMARY KEY,
    dept_name  VARCHAR(20)
);
INSERT INTO departments VALUES
(10, 'IT'), (20, 'HR'), (30, 'Finance'), (40, 'Sales'), (50, 'Marketing');

-- 2) employees
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

-- 3) customers
CREATE TABLE customers (
    customer_id  INT PRIMARY KEY,
    name         VARCHAR(30)
);
INSERT INTO customers VALUES
(1, 'Asha'), (2, 'Ravi'), (3, 'Meena'), (4, 'Kabir'), (5, 'Divya');

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

-- 5) daily_sales (has missing dates on purpose)
CREATE TABLE daily_sales (
    sale_date  DATE,
    amount     INT
);
INSERT INTO daily_sales VALUES
('2026-10-01', 100),
('2026-10-02', 150),
('2026-10-04', 200),
('2026-10-07', 120);

-- 6) routes (a small graph)
CREATE TABLE routes (
    src  VARCHAR(20),
    dst  VARCHAR(20)
);
INSERT INTO routes VALUES
('Mumbai', 'Delhi'),
('Delhi', 'Kolkata'),
('Mumbai', 'Pune'),
('Pune', 'Bangalore'),
('Bangalore', 'Kolkata');
```

### `departments`

| dept_id | dept_name |
|---|---|
| 10 | IT |
| 20 | HR |
| 30 | Finance |
| 40 | Sales |
| 50 | Marketing |

### `employees`

| emp_id | name | dept_id | salary | manager_id |
|---|---|---|---|---|
| 1 | Amit Sharma | 10 | 90000 | NULL |
| 2 | Priya Singh | 20 | 52000 | 1 |
| 3 | Rahul Verma | 10 | 68000 | 1 |
| 4 | Sneha Iyer | 30 | 82000 | 1 |
| 5 | Karan Mehta | 10 | 60000 | 3 |
| 6 | Neha Gupta | 20 | 58000 | 2 |
| 7 | Vikram Rao | 30 | 95000 | 4 |
| 8 | Anita Desai | 40 | 45000 | 1 |
| 9 | Rohan Das | 10 | 68000 | 3 |
| 10 | Meera Nair | 30 | 82000 | 4 |

Useful facts to remember (we will use them to verify answers):
- Company total salary = **700000**, average = **70000**.
- Department averages: IT = **71500**, HR = **55000**, Finance = **86333.33**, Sales = **45000**. Marketing has no employees.
- Duplicate salaries exist on purpose: **68000** (Rahul, Rohan) and **82000** (Sneha, Meera). They help test ties.

### `customers`

| customer_id | name |
|---|---|
| 1 | Asha |
| 2 | Ravi |
| 3 | Meena |
| 4 | Kabir |
| 5 | Divya |

### `orders`

| order_id | customer_id | order_date | amount |
|---|---|---|---|
| 101 | 1 | 2026-01-05 | 500 |
| 102 | 1 | 2026-02-10 | 700 |
| 103 | 2 | 2026-01-20 | 300 |
| 104 | 3 | 2026-02-15 | 900 |
| 105 | 2 | 2026-03-01 | 450 |
| 106 | 1 | 2026-03-12 | 200 |
| 107 | 4 | 2026-03-18 | 650 |

Divya (customer 5) has **no orders**.

### `daily_sales`

| sale_date | amount |
|---|---|
| 2026-10-01 | 100 |
| 2026-10-02 | 150 |
| 2026-10-04 | 200 |
| 2026-10-07 | 120 |

Dates 03, 05, 06 are missing.

### `routes`

| src | dst |
|---|---|
| Mumbai | Delhi |
| Delhi | Kolkata |
| Mumbai | Pune |
| Pune | Bangalore |
| Bangalore | Kolkata |

### Org chart implied by `employees.manager_id`

```text
Amit Sharma (1)
 |-- Priya Singh (2)
 |     `-- Neha Gupta (6)
 |-- Rahul Verma (3)
 |     |-- Karan Mehta (5)
 |     `-- Rohan Das (9)
 |-- Sneha Iyer (4)
 |     |-- Vikram Rao (7)
 |     `-- Meera Nair (10)
 `-- Anita Desai (8)
```

---

# PART 2: MODULE 7: SUBQUERIES

## 7.1 What is a Subquery?

A **subquery** (inner query) is a `SELECT` written **inside another query** (outer query), always wrapped in parentheses.

Think of it as asking a question in two steps:
1. First answer a small question (inner query): "What is the company's average salary?"
2. Then use that answer in the big question (outer query): "Who earns more than that?"

```sql
SELECT name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

| name | salary |
|---|---|
| Amit Sharma | 90000 |
| Sneha Iyer | 82000 |
| Vikram Rao | 95000 |
| Meera Nair | 82000 |

**Step-by-step:**
1. Inner query runs: `SELECT AVG(salary) FROM employees` returns **70000**.
2. Outer query becomes: `WHERE salary > 70000`.
3. Rows with salary above 70000: Amit (90000), Sneha (82000), Vikram (95000), Meera (82000).

**Why can't we just write `WHERE salary > AVG(salary)`?** Because `WHERE` runs before aggregation, and aggregate functions are not allowed there. The subquery solves this.

---

## 7.2 Types of Subqueries by What They Return

| Type | Returns | Used with |
|---|---|---|
| **Scalar** | exactly 1 row, 1 column (a single value) | `=, >, <, >=, <=, <>` |
| **Column / List** | many rows, 1 column | `IN, NOT IN, ANY, ALL, EXISTS` |
| **Row** | 1 row, many columns | `(a, b) = (SELECT ...)` |
| **Table (derived table)** | many rows, many columns | `FROM (SELECT ...) alias` |

### A) Scalar subquery

Returns **one value**. If it returns more than one row you get an error.

```sql
SELECT name, salary
FROM employees
WHERE salary = (SELECT MAX(salary) FROM employees);
```

| name | salary |
|---|---|
| Vikram Rao | 95000 |

**Common error:**

```sql
-- ERROR: Subquery returns more than 1 row
SELECT name FROM employees
WHERE salary = (SELECT salary FROM employees WHERE dept_id = 10);
```

IT has 4 salaries, but `=` expects one value. Fix: use `IN`, or an aggregate such as `MAX`.

### B) Column / List subquery (with IN)

```sql
SELECT name
FROM employees
WHERE dept_id IN (SELECT dept_id
                  FROM departments
                  WHERE dept_name IN ('IT', 'HR'))
ORDER BY emp_id;
```

| name |
|---|
| Amit Sharma |
| Priya Singh |
| Rahul Verma |
| Karan Mehta |
| Neha Gupta |
| Rohan Das |

**Step-by-step:**
1. Inner query returns the list `(10, 20)` (IT and HR dept ids).
2. Outer query becomes `WHERE dept_id IN (10, 20)`.
3. Six employees match.

### C) Row subquery

Compares several columns at once.

```sql
SELECT name, dept_id, salary
FROM employees
WHERE (dept_id, salary) IN (SELECT dept_id, MAX(salary)
                            FROM employees
                            GROUP BY dept_id)
ORDER BY emp_id;
```

| name | dept_id | salary |
|---|---|---|
| Amit Sharma | 10 | 90000 |
| Neha Gupta | 20 | 58000 |
| Vikram Rao | 30 | 95000 |
| Anita Desai | 40 | 45000 |

**Explanation:** The inner query produces the pairs `(10, 90000), (20, 58000), (30, 95000), (40, 45000)` (the highest salary in each department). The outer query keeps employees whose `(dept_id, salary)` pair is in that list. This finds the **top earner per department**.

> Works in MySQL and PostgreSQL. SQL Server does not support this tuple syntax; use `JOIN` or `EXISTS` instead.

### D) Table subquery (derived table in FROM)

```sql
SELECT dept_id, avg_sal
FROM (SELECT dept_id, AVG(salary) AS avg_sal
      FROM employees
      GROUP BY dept_id) AS t
WHERE avg_sal > 60000
ORDER BY dept_id;
```

| dept_id | avg_sal |
|---|---|
| 10 | 71500.00 |
| 30 | 86333.33 |

**Step-by-step:**
1. The inner query builds a temporary table `t` with one row per department and its average salary.
2. The outer query treats `t` like a normal table and filters `avg_sal > 60000`.
3. HR (55000) and Sales (45000) are removed.

> A subquery in `FROM` **must have an alias** (`AS t`). It is the "temporary table you can query."

---

## 7.3 Where Can Subqueries Appear?

| Location | Typical use |
|---|---|
| `WHERE` | filter using a computed value or list |
| `SELECT` | add a computed column per row |
| `FROM` | pre-aggregate or reshape data |
| `HAVING` | filter groups by a computed value |

### In SELECT (scalar subquery as a column)

```sql
SELECT name,
       salary,
       (SELECT AVG(salary) FROM employees)          AS company_avg,
       salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees
WHERE dept_id = 10
ORDER BY emp_id;
```

| name | salary | company_avg | diff_from_avg |
|---|---|---|---|
| Amit Sharma | 90000 | 70000 | 20000 |
| Rahul Verma | 68000 | 70000 | -2000 |
| Karan Mehta | 60000 | 70000 | -10000 |
| Rohan Das | 68000 | 70000 | -2000 |

(Decimal places for `company_avg` depend on the database, for example `70000.0000` in MySQL.)

**Explanation:** The inner query returns one number (70000) that is repeated on every row.

Another common use: fetch a name from another table without a join.

```sql
SELECT o.order_id,
       o.amount,
       (SELECT c.name FROM customers c WHERE c.customer_id = o.customer_id) AS customer_name
FROM orders o
WHERE o.order_id IN (101, 104, 107);
```

| order_id | amount | customer_name |
|---|---|---|
| 101 | 500 | Asha |
| 104 | 900 | Meena |
| 107 | 650 | Kabir |

### In HAVING

**Departments whose average salary is above the company average:**

```sql
SELECT dept_id, AVG(salary) AS avg_sal
FROM employees
GROUP BY dept_id
HAVING AVG(salary) > (SELECT AVG(salary) FROM employees)
ORDER BY dept_id;
```

| dept_id | avg_sal |
|---|---|
| 10 | 71500.00 |
| 30 | 86333.33 |

**Explanation:** The company average is 70000. IT (71500) and Finance (86333.33) are above it; HR (55000) and Sales (45000) are not.

---

## 7.4 Correlated vs Non-Correlated Subqueries

### Non-correlated (independent)
The inner query does **not** depend on the outer query. It runs **once**, and its result is reused.

All the examples above were non-correlated.

### Correlated (dependent)
The inner query **refers to a column from the outer query**. Conceptually, it is re-evaluated **for every outer row**.

**Problem:** Find employees who earn more than the **average of their own department**.

```sql
SELECT e.name, e.dept_id, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(e2.salary)
                  FROM employees e2
                  WHERE e2.dept_id = e.dept_id)
ORDER BY e.emp_id;
```

| name | dept_id | salary |
|---|---|---|
| Amit Sharma | 10 | 90000 |
| Neha Gupta | 20 | 58000 |
| Vikram Rao | 30 | 95000 |

**How it runs, row by row:**

| Outer row (e) | Inner query: AVG salary of e's department | Condition | Keep? |
|---|---|---|---|
| Amit (dept 10, 90000) | 71500 | 90000 > 71500 | Yes |
| Priya (dept 20, 52000) | 55000 | 52000 > 55000 | No |
| Rahul (dept 10, 68000) | 71500 | 68000 > 71500 | No |
| Sneha (dept 30, 82000) | 86333.33 | 82000 > 86333.33 | No |
| Karan (dept 10, 60000) | 71500 | 60000 > 71500 | No |
| Neha (dept 20, 58000) | 55000 | 58000 > 55000 | Yes |
| Vikram (dept 30, 95000) | 86333.33 | 95000 > 86333.33 | Yes |
| Anita (dept 40, 45000) | 45000 | 45000 > 45000 | No |
| Rohan (dept 10, 68000) | 71500 | 68000 > 71500 | No |
| Meera (dept 30, 82000) | 86333.33 | 82000 > 86333.33 | No |

**Key points:**
- `e` and `e2` are table aliases. The inner query uses `e.dept_id` from the outer query, so it is correlated.
- Correlated subqueries are easy to write but can be **slow on big tables**. The better approach is to pre-aggregate in a CTE or derived table and join (see Part 3), or use window functions (Phase 4).

**Same problem, faster style (join to a derived table):**

```sql
SELECT e.name, e.dept_id, e.salary
FROM employees e
JOIN (SELECT dept_id, AVG(salary) AS avg_sal
      FROM employees
      GROUP BY dept_id) d
  ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal
ORDER BY e.emp_id;
```

Same 3 rows. The average per department is computed **once** per department, not once per employee.

---

## 7.5 EXISTS, NOT EXISTS, IN, NOT IN

### EXISTS
`EXISTS (subquery)` is **TRUE if the subquery returns at least one row**, else FALSE. The selected columns do not matter (people write `SELECT 1`). It usually stops at the first match.

**Customers who placed at least one order above 600:**

```sql
SELECT c.customer_id, c.name
FROM customers c
WHERE EXISTS (SELECT 1
              FROM orders o
              WHERE o.customer_id = c.customer_id
                AND o.amount > 600)
ORDER BY c.customer_id;
```

| customer_id | name |
|---|---|
| 1 | Asha |
| 3 | Meena |
| 4 | Kabir |

**Explanation:** Asha has an order of 700, Meena 900, Kabir 650. Ravi's biggest order is 450, and Divya has none.

### NOT EXISTS
TRUE if the subquery returns **no rows**. This is the safest way to find "who never did X."

**Customers who never placed an order:**

```sql
SELECT c.customer_id, c.name
FROM customers c
WHERE NOT EXISTS (SELECT 1
                  FROM orders o
                  WHERE o.customer_id = c.customer_id);
```

| customer_id | name |
|---|---|
| 5 | Divya |

**Departments with no employees:**

```sql
SELECT d.dept_name
FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);
```

| dept_name |
|---|
| Marketing |

### IN vs EXISTS vs JOIN: Which to Use?

| | `IN` | `EXISTS` | `JOIN` |
|---|---|---|---|
| Meaning | value is in a list | at least one matching row exists | combines columns from both tables |
| Duplicates | no duplicates in output | no duplicates in output | **can multiply rows** if many matches |
| Needs columns from the other table? | No | No | Yes |
| NULL safety (negated form) | `NOT IN` is **unsafe** with NULLs | `NOT EXISTS` is safe | `LEFT JOIN ... IS NULL` is safe |
| Typical use | small static lists, simple filters | "exists / does not exist" checks | when you need data from both tables |

> **Rule of thumb:**
> - Need "who has / has not"? Use `EXISTS` / `NOT EXISTS`.
> - Need columns from both tables? Use `JOIN`.
> - Small fixed list or simple subquery with no NULLs? `IN` is fine.
> - Modern optimizers often turn `IN`, `EXISTS`, and `JOIN` into the same plan, but `NOT IN` behaves differently because of NULLs.

### The NOT IN + NULL Trap (again, because interviewers love it)

**Goal:** employees who are not managers of anyone.

```sql
-- Returns NOTHING, because the subquery list contains a NULL (Amit has no manager)
SELECT emp_id, name
FROM employees
WHERE emp_id NOT IN (SELECT manager_id FROM employees);
```

| emp_id | name |
|---|---|
| (no rows) | |

**Fix (safe):**

```sql
SELECT e.emp_id, e.name
FROM employees e
WHERE NOT EXISTS (SELECT 1 FROM employees m WHERE m.manager_id = e.emp_id)
ORDER BY e.emp_id;
```

| emp_id | name |
|---|---|
| 5 | Karan Mehta |
| 6 | Neha Gupta |
| 7 | Vikram Rao |
| 8 | Anita Desai |
| 9 | Rohan Das |
| 10 | Meera Nair |

Managers are employees 1, 2, 3, and 4 (they appear in `manager_id`). Everyone else is not a manager.

---

## 7.6 ANY and ALL

| Keyword | Meaning |
|---|---|
| `> ANY (list)` | greater than **at least one** value (same as `> MIN`) |
| `> ALL (list)` | greater than **every** value (same as `> MAX`) |
| `< ANY (list)` | less than at least one value (same as `< MAX`) |
| `< ALL (list)` | less than every value (same as `< MIN`) |
| `= ANY (list)` | same as `IN` |

**Employees who earn more than everyone in IT:**

```sql
SELECT name, salary
FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE dept_id = 10);
```

| name | salary |
|---|---|
| Vikram Rao | 95000 |

**Explanation:** IT salaries are 90000, 68000, 60000, 68000. `> ALL` means greater than 90000 (the max). Only Vikram (95000) qualifies.

**Employees who earn less than at least one HR employee:**

```sql
SELECT name, salary
FROM employees
WHERE salary < ANY (SELECT salary FROM employees WHERE dept_id = 20)
ORDER BY emp_id;
```

| name | salary |
|---|---|
| Priya Singh | 52000 |
| Anita Desai | 45000 |

**Explanation:** HR salaries are 52000 and 58000. `< ANY` means less than the max (58000). Priya (52000) and Anita (45000) qualify. Neha (58000) does not (not less than herself).

> `ALL` on an **empty** list is TRUE for every row, and any comparison against a list containing NULL can turn UNKNOWN. In interviews, rewriting with `MAX` / `MIN` is often clearer.

---

## 7.7 Interview Patterns with Subqueries (P12 to P15)

### P12: Compare against an aggregate (above average, max per group)

**Template:**
```sql
SELECT ...
FROM t
WHERE col > (SELECT AVG(col) FROM t);          -- company level

SELECT ...
FROM t x
WHERE col > (SELECT AVG(col) FROM t y WHERE y.grp = x.grp);   -- per group (correlated)
```

**Example: employees earning the maximum salary in their own department**

```sql
SELECT e.name, e.dept_id, e.salary
FROM employees e
WHERE e.salary = (SELECT MAX(e2.salary)
                  FROM employees e2
                  WHERE e2.dept_id = e.dept_id)
ORDER BY e.emp_id;
```

| name | dept_id | salary |
|---|---|---|
| Amit Sharma | 10 | 90000 |
| Neha Gupta | 20 | 58000 |
| Vikram Rao | 30 | 95000 |
| Anita Desai | 40 | 45000 |

**Note:** If two people tie for the maximum in a department, **both** appear (this is usually what you want).

---

### P13: Nth highest value

The data has salaries (distinct, high to low): `95000, 90000, 82000, 68000, 60000, 58000, 52000, 45000`.

**Second highest salary** (classic):

```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

| second_highest |
|---|
| 90000 |

**Explanation:** The inner query gets 95000. The outer query takes the maximum salary below that: 90000.

**Third highest salary using `DISTINCT` + `LIMIT OFFSET`:**

```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;
```

| salary |
|---|
| 82000 |

**Explanation:** Distinct salaries sorted descending: 95000, 90000, 82000, ... `OFFSET 2` skips the first two, `LIMIT 1` takes one: 82000. `DISTINCT` is essential, otherwise ties (82000 appears twice) would break the logic.

**Nth highest with a correlated subquery (works in any database):**

```sql
SELECT DISTINCT e1.salary
FROM employees e1
WHERE 2 = (SELECT COUNT(DISTINCT e2.salary)
           FROM employees e2
           WHERE e2.salary > e1.salary);
```

| salary |
|---|
| 82000 |

**Explanation (for N = 3):** A salary is the 3rd highest if exactly **2 distinct salaries are greater than it**. For 82000: only 95000 and 90000 are greater, so count = 2. Use `N - 1` for the Nth highest.

**Handling "no Nth value" (returns NULL instead of empty):**

```sql
SELECT (SELECT DISTINCT salary
        FROM employees
        ORDER BY salary DESC
        LIMIT 1 OFFSET 20) AS tenth_highest;
```

| tenth_highest |
|---|
| NULL |

A scalar subquery that returns no rows becomes NULL, which is what LeetCode "Nth Highest Salary" expects.

> **Preview (Phase 4):** with `DENSE_RANK() OVER (ORDER BY salary DESC)` you get ties handled automatically and can find "Nth highest per department" easily.

---

### P14: NOT EXISTS for "never did X"

Already shown in 7.5. Template:

```sql
SELECT ...
FROM A a
WHERE NOT EXISTS (SELECT 1 FROM B b WHERE b.a_id = a.id);
```

**More examples of the same pattern:**
- Customers who never ordered
- Products never sold
- Employees with no direct reports
- Users who signed up but never logged in

**Variation: "bought in Jan but not in Feb":**

```sql
SELECT DISTINCT o1.customer_id
FROM orders o1
WHERE o1.order_date >= '2026-01-01' AND o1.order_date < '2026-02-01'
  AND NOT EXISTS (SELECT 1
                  FROM orders o2
                  WHERE o2.customer_id = o1.customer_id
                    AND o2.order_date >= '2026-02-01' AND o2.order_date < '2026-03-01');
```

| customer_id |
|---|
| 2 |

**Explanation:** January buyers are customers 1 (Asha) and 2 (Ravi). Asha also ordered in February (order 102), so she is excluded. Ravi did not order in February, so he remains.

---

### P15: Latest (or earliest) record per group

**Problem:** show each customer's most recent order.

```sql
SELECT o.order_id, o.customer_id, o.order_date, o.amount
FROM orders o
WHERE o.order_date = (SELECT MAX(o2.order_date)
                      FROM orders o2
                      WHERE o2.customer_id = o.customer_id)
ORDER BY o.customer_id;
```

| order_id | customer_id | order_date | amount |
|---|---|---|---|
| 106 | 1 | 2026-03-12 | 200 |
| 105 | 2 | 2026-03-01 | 450 |
| 104 | 3 | 2026-02-15 | 900 |
| 107 | 4 | 2026-03-18 | 650 |

**Step-by-step:**
- Customer 1 has orders on Jan 5, Feb 10, Mar 12. The max is Mar 12, which is order 106.
- Customer 2: Jan 20 and Mar 1, so Mar 1 (order 105).
- Customers 3 and 4 have one order each.

**Three equivalent ways:**

```sql
-- Way 2: row subquery with IN
SELECT order_id, customer_id, order_date, amount
FROM orders
WHERE (customer_id, order_date) IN (SELECT customer_id, MAX(order_date)
                                    FROM orders
                                    GROUP BY customer_id)
ORDER BY customer_id;

-- Way 3: join to a derived table
SELECT o.order_id, o.customer_id, o.order_date, o.amount
FROM orders o
JOIN (SELECT customer_id, MAX(order_date) AS last_date
      FROM orders
      GROUP BY customer_id) l
  ON o.customer_id = l.customer_id AND o.order_date = l.last_date
ORDER BY o.customer_id;

-- Way 4 (Phase 4 preview): window function
SELECT order_id, customer_id, order_date, amount
FROM (SELECT *,
             ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
      FROM orders) t
WHERE rn = 1
ORDER BY customer_id;
```

All four return the same 4 rows.

> **Tie warning:** if a customer had two orders on the same latest date, ways 1 to 3 return **both**, while `ROW_NUMBER` returns **one**. Always ask the interviewer which behavior they want.

---

## 7.8 Common Subquery Mistakes

| Mistake | Fix |
|---|---|
| Scalar subquery returns many rows (`= (SELECT ...)`) | Use `IN`, or add an aggregate / `LIMIT 1` |
| Forgetting the alias on a `FROM` subquery | Always add `AS alias` |
| `NOT IN` with a subquery that may return NULL | Use `NOT EXISTS` or add `WHERE col IS NOT NULL` |
| Correlated subquery on millions of rows | Pre-aggregate in a CTE/derived table and `JOIN` |
| Using `SELECT *` inside `EXISTS` (harmless but noisy) | Use `SELECT 1` |
| Putting `LIMIT` inside `IN (...)` subquery in MySQL | MySQL does not support it; wrap in a derived table or use `JOIN` |
| Wrong correlation (inner alias not linked to outer) | Check the `e2.col = e.col` link carefully |
| Counting duplicates when finding Nth highest | Use `DISTINCT` or `DENSE_RANK` |

---

## 7.9 Module 7 Practice Problems

1. Employees who earn more than the company average salary.
2. Department names that have at least one employee earning more than 80000.
3. The second highest salary.
4. Departments that have **no** employees.
5. Customers who never placed an order.
6. For each order, show the customer name using a scalar subquery.
7. Each customer's latest order.
8. Employees whose salary is shared with at least one other employee.
9. Departments whose total payroll is higher than the average total payroll of all departments (hint: derived table inside a scalar subquery).
10. Employees who earn more than every employee of department 20.

<details>
<summary>Answers</summary>

```sql
-- 1  (Amit, Sneha, Vikram, Meera)
SELECT name, salary FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);

-- 2  (IT, Finance)
SELECT d.dept_name FROM departments d
WHERE EXISTS (SELECT 1 FROM employees e
              WHERE e.dept_id = d.dept_id AND e.salary > 80000);

-- 3  (90000)
SELECT MAX(salary) FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);

-- 4  (Marketing)
SELECT dept_name FROM departments d
WHERE NOT EXISTS (SELECT 1 FROM employees e WHERE e.dept_id = d.dept_id);

-- 5  (Divya)
SELECT name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);

-- 6
SELECT o.order_id, o.amount,
       (SELECT c.name FROM customers c WHERE c.customer_id = o.customer_id) AS customer_name
FROM orders o;

-- 7  (orders 106, 105, 104, 107)
SELECT * FROM orders o
WHERE order_date = (SELECT MAX(order_date) FROM orders WHERE customer_id = o.customer_id);

-- 8  (Rahul, Rohan, Sneha, Meera)
SELECT name, salary FROM employees
WHERE salary IN (SELECT salary FROM employees GROUP BY salary HAVING COUNT(*) > 1)
ORDER BY emp_id;

-- 9  Total payroll per dept: IT 286000, HR 110000, Finance 259000, Sales 45000.
--    Average of those = 175000. Departments above that: IT, Finance.
SELECT dept_id, SUM(salary) AS total_payroll
FROM employees
GROUP BY dept_id
HAVING SUM(salary) > (SELECT AVG(t.total) FROM (SELECT SUM(salary) AS total
                                                 FROM employees
                                                 GROUP BY dept_id) t);

-- 10  HR max is 58000 -> everyone above 58000
SELECT name, salary FROM employees
WHERE salary > ALL (SELECT salary FROM employees WHERE dept_id = 20)
ORDER BY emp_id;
-- Amit, Rahul, Sneha, Karan, Vikram, Rohan, Meera
```
</details>

---

# PART 3: MODULE 8: CTEs (Common Table Expressions)

## 8.1 What is a CTE?

A **CTE** is a **named temporary result set** that exists only for the duration of **one query**. You define it at the top with `WITH`, then use it like a table.

**Why use CTEs?**
- Break a big, messy query into small, named, readable steps
- Reuse the same result more than once in the same query
- Avoid deeply nested subqueries
- Needed for **recursive** queries (hierarchies, series, graphs)

**Syntax:**

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT ...
FROM cte_name;
```

---

## 8.2 Basic CTE

**Problem:** employees earning more than their department average (same as 7.4, but readable).

```sql
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY dept_id
)
SELECT e.name, e.salary, d.avg_sal
FROM employees e
JOIN dept_avg d ON e.dept_id = d.dept_id
WHERE e.salary > d.avg_sal
ORDER BY e.emp_id;
```

| name | salary | avg_sal |
|---|---|---|
| Amit Sharma | 90000 | 71500.00 |
| Neha Gupta | 58000 | 55000.00 |
| Vikram Rao | 95000 | 86333.33 |

**Step-by-step:**
1. `dept_avg` is computed first: one row per department with its average salary.
2. The main query joins employees to `dept_avg` on `dept_id`.
3. It keeps employees above their department's average.

You read it top to bottom like a story: "first get department averages, then compare."

---

## 8.3 Multiple (Chained) CTEs

You can define several CTEs with **one `WITH`**, separated by commas. Later CTEs can use earlier ones.

**Problem:** show each department's total payroll and its percentage of the company payroll.

```sql
WITH dept_totals AS (
    SELECT dept_id, SUM(salary) AS total_salary
    FROM employees
    GROUP BY dept_id
),
company_total AS (
    SELECT SUM(total_salary) AS grand_total
    FROM dept_totals
)
SELECT d.dept_name,
       t.total_salary,
       ROUND(100.0 * t.total_salary / c.grand_total, 2) AS pct_of_company
FROM dept_totals t
JOIN departments d ON d.dept_id = t.dept_id
CROSS JOIN company_total c
ORDER BY t.total_salary DESC;
```

| dept_name | total_salary | pct_of_company |
|---|---|---|
| IT | 286000 | 40.86 |
| Finance | 259000 | 37.00 |
| HR | 110000 | 15.71 |
| Sales | 45000 | 6.43 |

**Step-by-step:**
1. `dept_totals`: IT 286000, HR 110000, Finance 259000, Sales 45000.
2. `company_total` (uses `dept_totals`): 700000.
3. Final query: `286000 / 700000 = 40.86%`, and so on. The percentages add up to 100.
4. `CROSS JOIN` attaches the single-row `company_total` to every row.

**More complex example: combine three CTEs**

**Problem:** among departments whose average salary is above the company average, show the top earner of each.

```sql
WITH dept_stats AS (
    SELECT dept_id,
           AVG(salary) AS avg_sal,
           MAX(salary) AS max_sal
    FROM employees
    GROUP BY dept_id
),
company AS (
    SELECT AVG(salary) AS company_avg
    FROM employees
),
strong_depts AS (
    SELECT s.dept_id, s.avg_sal, s.max_sal
    FROM dept_stats s
    CROSS JOIN company c
    WHERE s.avg_sal > c.company_avg
)
SELECT dp.dept_name,
       e.name,
       e.salary,
       ROUND(sd.avg_sal, 2) AS dept_avg
FROM strong_depts sd
JOIN departments dp ON dp.dept_id = sd.dept_id
JOIN employees e    ON e.dept_id = sd.dept_id AND e.salary = sd.max_sal
ORDER BY dp.dept_name;
```

| dept_name | name | salary | dept_avg |
|---|---|---|---|
| Finance | Vikram Rao | 95000 | 86333.33 |
| IT | Amit Sharma | 90000 | 71500.00 |

**Step-by-step:**
1. `dept_stats`: average and maximum salary per department.
2. `company`: company average = 70000.
3. `strong_depts`: departments with average above 70000 are IT (71500) and Finance (86333.33).
4. The final query finds the employee in each of those departments whose salary equals the department maximum.

> This is how real interview solutions are structured: **one idea per CTE**, with names that explain the step.

---

## 8.4 CTE vs Subquery vs Temp Table vs View

| | Subquery | CTE | Temp table | View |
|---|---|---|---|---|
| Lifetime | one clause | one statement | session (or until dropped) | permanent (stored query) |
| Readability | poor when nested | good | good | good |
| Reusable inside the same query | No (repeat it) | Yes | Yes | Yes |
| Stores data physically | No | Usually no | Yes | No (unless materialized view) |
| Can have indexes | No | No | Yes | No (materialized: yes) |
| Best for | simple one-off checks | step-by-step logic, recursion | very large reused intermediate results | shared, reusable query logic |

**Important:** a CTE is mainly for **readability**, not automatically for speed.
- Many engines (PostgreSQL 12+, MySQL 8, SQL Server) treat a CTE like an inline subquery.
- Some engines (older PostgreSQL, some warehouses) may compute it once and store it. Check your database's behavior and `EXPLAIN` plan.

---

## 8.5 CTE Rules and Tips

- A CTE exists **only for the one statement** that follows it.
- Use **one** `WITH`, then separate CTEs with commas (do not repeat `WITH`).
- A CTE can refer to **earlier** CTEs in the same `WITH` (not later ones, except in recursion).
- Give CTEs **meaningful names** (`dept_totals`, not `t1`).
- CTE column names can be given explicitly: `WITH t(dept, total) AS (SELECT ...)`.
- CTEs can be followed by `SELECT`, `INSERT`, `UPDATE`, `DELETE`.

---

## 8.6 Recursive CTEs

A **recursive CTE** is a CTE that **refers to itself**. It is how SQL walks through **hierarchies** (org charts, categories), **generates series**, and **traverses graphs**.

### Anatomy

```sql
WITH RECURSIVE name AS (
    -- 1) ANCHOR member: the starting rows (runs once)
    SELECT ...

    UNION ALL

    -- 2) RECURSIVE member: refers to `name`, builds the next level
    SELECT ...
    FROM source JOIN name ON ...
    WHERE <stop condition>      -- 3) TERMINATION
)
SELECT * FROM name;
```

**How it executes (think of a loop):**
1. Run the **anchor** query. Its rows become the "current level."
2. Run the **recursive** query using the current level as input. New rows are produced.
3. Add the new rows to the result and make them the new "current level."
4. Repeat step 2 until the recursive query returns **no new rows**.
5. Return everything collected.

> Keyword: **MySQL and PostgreSQL** need `WITH RECURSIVE`. **SQL Server and Oracle** use just `WITH`.

### Example 1: Generate numbers 1 to 5

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n                     -- anchor
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 5   -- recursive + stop condition
)
SELECT n FROM nums;
```

| n |
|---|
| 1 |
| 2 |
| 3 |
| 4 |
| 5 |

**Trace:**

| Iteration | Input | Output added |
|---|---|---|
| Anchor | none | 1 |
| 1 | 1 | 2 |
| 2 | 2 | 3 |
| 3 | 3 | 4 |
| 4 | 4 | 5 |
| 5 | 5 (fails `n < 5`) | nothing, so stop |

**Without the `WHERE n < 5` condition, it would loop forever** (or until the database's recursion limit is hit). MySQL's limit is 1000 levels by default (`cte_max_recursion_depth`); SQL Server's is 100 (`OPTION (MAXRECURSION n)`).

**Squares example:**

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 5
)
SELECT n, n * n AS square FROM nums;
```

| n | square |
|---|---|
| 1 | 1 |
| 2 | 4 |
| 3 | 9 |
| 4 | 16 |
| 5 | 25 |

### Example 2: Fill missing dates (very common interview problem)

`daily_sales` has no rows for Oct 3, 5, 6. We want every day from Oct 1 to Oct 7, showing 0 for missing days.

**MySQL:**

```sql
WITH RECURSIVE calendar AS (
    SELECT CAST('2026-10-01' AS DATE) AS d
    UNION ALL
    SELECT DATE_ADD(d, INTERVAL 1 DAY)
    FROM calendar
    WHERE d < '2026-10-07'
)
SELECT c.d AS sale_date,
       COALESCE(s.amount, 0) AS amount
FROM calendar c
LEFT JOIN daily_sales s ON s.sale_date = c.d
ORDER BY c.d;
```

**PostgreSQL version of the calendar CTE:**

```sql
WITH RECURSIVE calendar AS (
    SELECT DATE '2026-10-01' AS d
    UNION ALL
    SELECT d + 1 FROM calendar WHERE d < DATE '2026-10-07'
)
...
```
(PostgreSQL also has `generate_series('2026-10-01'::date, '2026-10-07'::date, '1 day')`.)

| sale_date | amount |
|---|---|
| 2026-10-01 | 100 |
| 2026-10-02 | 150 |
| 2026-10-03 | 0 |
| 2026-10-04 | 200 |
| 2026-10-05 | 0 |
| 2026-10-06 | 0 |
| 2026-10-07 | 120 |

**Step-by-step:**
1. The recursive CTE builds 7 consecutive dates.
2. `LEFT JOIN` keeps every date, matching sales where they exist.
3. For dates with no sale, `s.amount` is NULL, and `COALESCE` turns it into 0.

### Example 3: Org chart (hierarchy, top-down)

**Problem:** list every employee with their level in the hierarchy and the full reporting path.

```sql
WITH RECURSIVE org AS (
    -- anchor: the top boss (no manager)
    SELECT emp_id,
           name,
           manager_id,
           1 AS level,
           CAST(name AS CHAR(300)) AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- recursive: employees whose manager is already in the result
    SELECT e.emp_id,
           e.name,
           e.manager_id,
           o.level + 1,
           CAST(CONCAT(o.path, ' > ', e.name) AS CHAR(300))
    FROM employees e
    JOIN org o ON e.manager_id = o.emp_id
)
SELECT emp_id, name, level, path
FROM org
ORDER BY level, emp_id;
```

| emp_id | name | level | path |
|---|---|---|---|
| 1 | Amit Sharma | 1 | Amit Sharma |
| 2 | Priya Singh | 2 | Amit Sharma > Priya Singh |
| 3 | Rahul Verma | 2 | Amit Sharma > Rahul Verma |
| 4 | Sneha Iyer | 2 | Amit Sharma > Sneha Iyer |
| 8 | Anita Desai | 2 | Amit Sharma > Anita Desai |
| 5 | Karan Mehta | 3 | Amit Sharma > Rahul Verma > Karan Mehta |
| 6 | Neha Gupta | 3 | Amit Sharma > Priya Singh > Neha Gupta |
| 7 | Vikram Rao | 3 | Amit Sharma > Sneha Iyer > Vikram Rao |
| 9 | Rohan Das | 3 | Amit Sharma > Rahul Verma > Rohan Das |
| 10 | Meera Nair | 3 | Amit Sharma > Sneha Iyer > Meera Nair |

**Trace of iterations:**

| Iteration | Looking for employees whose manager is in | Rows added |
|---|---|---|
| Anchor | (manager is NULL) | Amit (1) |
| 1 | {1} | Priya, Rahul, Sneha, Anita (level 2) |
| 2 | {2, 3, 4, 8} | Neha, Karan, Rohan, Vikram, Meera (level 3). Anita (8) has no reports. |
| 3 | {5, 6, 7, 9, 10} | none, so stop |

**PostgreSQL notes:** use `CAST(name AS VARCHAR(300))` and `||` for concatenation (`o.path || ' > ' || e.name`).

> **MySQL tip:** the data type of a recursive column is decided by the anchor part. Without the `CAST(... AS CHAR(300))`, a growing text path can fail with "Data too long". Always cast the starting string wide enough.

**Count of employees at each level:**

```sql
WITH RECURSIVE org AS (
    SELECT emp_id, manager_id, 1 AS level
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.emp_id, e.manager_id, o.level + 1
    FROM employees e JOIN org o ON e.manager_id = o.emp_id
)
SELECT level, COUNT(*) AS employees_at_level
FROM org
GROUP BY level
ORDER BY level;
```

| level | employees_at_level |
|---|---|
| 1 | 1 |
| 2 | 4 |
| 3 | 5 |

### Example 4: Bottom-up (management chain of one employee)

**Problem:** show Karan's chain of command up to the top.

```sql
WITH RECURSIVE chain AS (
    SELECT emp_id, name, manager_id, 0 AS steps_up
    FROM employees
    WHERE emp_id = 5                 -- start at Karan

    UNION ALL

    SELECT e.emp_id, e.name, e.manager_id, c.steps_up + 1
    FROM employees e
    JOIN chain c ON e.emp_id = c.manager_id    -- move UP to the manager
)
SELECT emp_id, name, steps_up
FROM chain
ORDER BY steps_up;
```

| emp_id | name | steps_up |
|---|---|---|
| 5 | Karan Mehta | 0 |
| 3 | Rahul Verma | 1 |
| 1 | Amit Sharma | 2 |

**Explanation:** The join direction decides the walk. Top-down uses `e.manager_id = o.emp_id` (find my children). Bottom-up uses `e.emp_id = c.manager_id` (find my parent). It stops when Amit's `manager_id` is NULL, because no row matches.

### Example 5: Graph traversal (all paths between two nodes)

**Problem:** list every route from Mumbai to Kolkata.

```sql
WITH RECURSIVE paths AS (
    SELECT src,
           dst,
           1 AS hops,
           CAST(CONCAT(src, ' > ', dst) AS CHAR(300)) AS route
    FROM routes
    WHERE src = 'Mumbai'

    UNION ALL

    SELECT p.src,
           r.dst,
           p.hops + 1,
           CAST(CONCAT(p.route, ' > ', r.dst) AS CHAR(300))
    FROM paths p
    JOIN routes r ON p.dst = r.src
    WHERE p.hops < 5                     -- safety limit against cycles
)
SELECT route, hops
FROM paths
WHERE dst = 'Kolkata'
ORDER BY hops;
```

| route | hops |
|---|---|
| Mumbai > Delhi > Kolkata | 2 |
| Mumbai > Pune > Bangalore > Kolkata | 3 |

**Explanation:**
- Start with routes leaving Mumbai: Delhi, Pune.
- Extend each path by one more hop, repeatedly.
- Keep only paths whose final node is Kolkata.
- `hops < 5` stops infinite loops if the graph contains a cycle (for example A to B to A).

### Preventing infinite loops (cycles)

| Technique | How |
|---|---|
| Depth limit | `WHERE level < N` or `hops < N` |
| Path check | refuse to visit a node already in the path (e.g., `LOCATE(...) = 0` in MySQL, `NOT (id = ANY(path_array))` in PostgreSQL) |
| Engine limit | MySQL `cte_max_recursion_depth`, SQL Server `OPTION (MAXRECURSION n)` |
| `UNION` instead of `UNION ALL` | removes duplicate rows, which can stop some cycles (slower) |
| `CYCLE` clause | SQL standard / PostgreSQL 14+ built-in cycle detection |

---

## 8.7 Interview Patterns with CTEs (P16 to P19)

### P16: Multi-step logic broken into CTEs

**Template:**
```sql
WITH step1 AS (...),
     step2 AS (... FROM step1 ...),
     step3 AS (... FROM step2 ...)
SELECT ... FROM step3;
```
Used for: funnels, cohorts, "find X, then compare to Y", ranking then filtering. Example shown in 8.3.

### P17: Hierarchies (org charts, category trees, bill of materials)

**Template:**
```sql
WITH RECURSIVE tree AS (
    SELECT id, parent_id, 1 AS lvl FROM t WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, p.lvl + 1
    FROM t c JOIN tree p ON c.parent_id = p.id
)
SELECT * FROM tree;
```
Typical questions: all subordinates of a manager, depth of the tree, full category path, total cost of an assembly.

**Example: all employees under Sneha (any depth)**

```sql
WITH RECURSIVE team AS (
    SELECT emp_id, name
    FROM employees
    WHERE manager_id = 4
    UNION ALL
    SELECT e.emp_id, e.name
    FROM employees e
    JOIN team t ON e.manager_id = t.emp_id
)
SELECT emp_id, name FROM team ORDER BY emp_id;
```

| emp_id | name |
|---|---|
| 7 | Vikram Rao |
| 10 | Meera Nair |

### P18: Generate number or date series (fill gaps)

Shown in Examples 1 and 2. Use when:
- You need a row for every day / month, even with no data
- You need a list of numbers to join against (for example splitting strings)
- You need to show zero values in reports

### P19: Graph traversal basics

Shown in Example 5. Typical questions: reachability, shortest number of hops, friends of friends, dependency chains. Always include a **depth limit or cycle check**.

---

## 8.8 Module 8 Practice Problems

1. Use a CTE to list departments whose total payroll is more than 100000.
2. Use a CTE to find employees earning above the company average, along with the company average beside each row.
3. Generate the numbers 1 to 10 with a recursive CTE.
4. Generate the first 10 squares (n and n squared).
5. Show every employee with their level in the org chart (top boss = level 1).
6. Show Meera's management chain up to the CEO.
7. List all employees under Rahul (any depth).
8. Fill missing dates between 2026-10-01 and 2026-10-07 in `daily_sales`, and also add a running note column `has_sales` = `'Yes'` or `'No'`.
9. Show every route from Mumbai to Bangalore.
10. Challenge: use two CTEs to find the department with the highest average salary and list all its employees.

<details>
<summary>Answers</summary>

```sql
-- 1  (IT 286000, HR 110000, Finance 259000)
WITH payroll AS (
    SELECT dept_id, SUM(salary) AS total FROM employees GROUP BY dept_id
)
SELECT d.dept_name, p.total
FROM payroll p JOIN departments d ON d.dept_id = p.dept_id
WHERE p.total > 100000
ORDER BY p.total DESC;

-- 2  (Amit, Sneha, Vikram, Meera; company_avg = 70000)
WITH co AS (SELECT AVG(salary) AS company_avg FROM employees)
SELECT e.name, e.salary, co.company_avg
FROM employees e CROSS JOIN co
WHERE e.salary > co.company_avg;

-- 3
WITH RECURSIVE n AS (
    SELECT 1 AS x UNION ALL SELECT x + 1 FROM n WHERE x < 10
)
SELECT x FROM n;

-- 4  (1,4,9,16,25,36,49,64,81,100)
WITH RECURSIVE n AS (
    SELECT 1 AS x UNION ALL SELECT x + 1 FROM n WHERE x < 10
)
SELECT x AS n, x * x AS square FROM n;

-- 5  see Example 3 (levels 1,2,2,2,2,3,3,3,3,3)

-- 6  (Meera 0, Sneha 1, Amit 2)
WITH RECURSIVE chain AS (
    SELECT emp_id, name, manager_id, 0 AS steps_up FROM employees WHERE emp_id = 10
    UNION ALL
    SELECT e.emp_id, e.name, e.manager_id, c.steps_up + 1
    FROM employees e JOIN chain c ON e.emp_id = c.manager_id
)
SELECT emp_id, name, steps_up FROM chain ORDER BY steps_up;

-- 7  (Karan, Rohan)
WITH RECURSIVE team AS (
    SELECT emp_id, name FROM employees WHERE manager_id = 3
    UNION ALL
    SELECT e.emp_id, e.name FROM employees e JOIN team t ON e.manager_id = t.emp_id
)
SELECT emp_id, name FROM team ORDER BY emp_id;

-- 8 (MySQL)
WITH RECURSIVE calendar AS (
    SELECT CAST('2026-10-01' AS DATE) AS d
    UNION ALL
    SELECT DATE_ADD(d, INTERVAL 1 DAY) FROM calendar WHERE d < '2026-10-07'
)
SELECT c.d AS sale_date,
       COALESCE(s.amount, 0) AS amount,
       CASE WHEN s.sale_date IS NULL THEN 'No' ELSE 'Yes' END AS has_sales
FROM calendar c
LEFT JOIN daily_sales s ON s.sale_date = c.d
ORDER BY c.d;
-- Yes, Yes, No, Yes, No, No, Yes

-- 9  (Mumbai > Pune > Bangalore, 2 hops)
WITH RECURSIVE paths AS (
    SELECT src, dst, 1 AS hops, CAST(CONCAT(src, ' > ', dst) AS CHAR(300)) AS route
    FROM routes WHERE src = 'Mumbai'
    UNION ALL
    SELECT p.src, r.dst, p.hops + 1, CAST(CONCAT(p.route, ' > ', r.dst) AS CHAR(300))
    FROM paths p JOIN routes r ON p.dst = r.src
    WHERE p.hops < 5
)
SELECT route, hops FROM paths WHERE dst = 'Bangalore';

-- 10  Finance has the highest average (86333.33) -> Sneha, Vikram, Meera
WITH dept_avg AS (
    SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id
),
top_dept AS (
    SELECT dept_id FROM dept_avg
    WHERE avg_sal = (SELECT MAX(avg_sal) FROM dept_avg)
)
SELECT e.name, e.salary
FROM employees e JOIN top_dept t ON e.dept_id = t.dept_id
ORDER BY e.emp_id;
```
</details>

---

# PART 4: Wrap-Up

## 4.1 Top Mistakes in Phase 3

| Mistake | Fix |
|---|---|
| `NOT IN` with possible NULLs | Use `NOT EXISTS` |
| Scalar subquery returns multiple rows | Aggregate, use `IN`, or limit |
| Missing alias on derived table | Add `AS alias` |
| Recursive CTE without a stop condition | Add a depth limit or cycle check |
| Forgetting `RECURSIVE` keyword in MySQL/PostgreSQL | `WITH RECURSIVE` |
| Recursive text column gets truncated (MySQL) | `CAST(... AS CHAR(300))` in the anchor |
| Using `UNION` instead of `UNION ALL` in recursion | Use `UNION ALL` unless you need de-duplication |
| Using a correlated subquery on very large data | Pre-aggregate in a CTE and join |
| Assuming CTE is always faster | CTE is mostly for readability; check `EXPLAIN` |
| Ignoring ties in "top 1 per group" or "Nth highest" | Decide: all tied rows or just one; use `DISTINCT` / `DENSE_RANK` accordingly |
| Joining a single-row CTE incorrectly | Use `CROSS JOIN` to attach a one-row result |

## 4.2 Phase 3 Cheat Sheet

```text
SUBQUERY TYPES
  scalar   : (SELECT MAX(x) FROM t)            -> one value, use with = > <
  list     : x IN (SELECT y FROM t)            -> many rows, one column
  row      : (a,b) IN (SELECT a,b FROM t)      -> compare tuples
  table    : FROM (SELECT ...) AS alias        -> derived table

CORRELATED : inner query uses outer alias (runs per outer row, conceptually)
EXISTS     : TRUE if inner returns >= 1 row      (safe, stops early)
NOT EXISTS : TRUE if inner returns 0 rows        (use instead of NOT IN)
ANY / ALL  : > ANY = > MIN   | > ALL = > MAX

CTE        : WITH a AS (...), b AS (... FROM a ...) SELECT ... FROM b;
RECURSIVE  : WITH RECURSIVE r AS (anchor UNION ALL recursive-step WHERE stop) SELECT ...

PATTERNS
  Above average / max per group   -> scalar or correlated subquery
  Nth highest                     -> DISTINCT + LIMIT OFFSET, or DENSE_RANK
  Never did X                     -> NOT EXISTS
  Latest per group                -> MAX(date) subquery, or ROW_NUMBER
  Multi-step logic                -> chained CTEs
  Hierarchy / tree                -> recursive CTE (anchor = root, step = join children)
  Missing dates / numbers         -> recursive calendar + LEFT JOIN + COALESCE
  Graph paths                     -> recursive CTE + hop limit
```

## 4.3 Self-Test Questions

1. What is the difference between a correlated and a non-correlated subquery?
2. Why must a subquery in `FROM` have an alias?
3. Why does `NOT IN` fail when the subquery returns a NULL, and how do you fix it?
4. What is the difference between `> ANY` and `> ALL`?
5. When would you choose `EXISTS` over `IN`?
6. How do you find the 3rd highest salary when duplicates exist?
7. What are the two parts of a recursive CTE and what does each do?
8. How do you prevent a recursive CTE from looping forever?
9. How do you walk up a hierarchy instead of down?
10. Is a CTE faster than a subquery? Explain.
11. How would you list every date in a month even when there are no sales on some days?
12. Why might a correlated subquery be slow on a billion-row table, and what are two alternatives?

## 4.4 What to Do Next

1. Create all tables from Part 1 and run **every** query in this guide. Compare your output with the tables shown.
2. Solve all practice problems without looking at the answers, then compare.
3. Practice on LeetCode (Database): *Second Highest Salary*, *Nth Highest Salary*, *Employees Earning More Than Their Managers*, *Customers Who Never Order*, *Department Highest Salary*, *Rank Scores*, *Human Traffic of Stadium* (medium/hard), *Find the Missing Dates* style problems.
4. Rewrite each correlated subquery you wrote as (a) a CTE + join and (b) a window function (after Phase 4).
5. When you are ready, say **"Start Phase 4"** for **Window Functions**, the most important interview module. If you want to fill the gap first, say **"Start Phase 2"** for Aggregation, Joins and Set Operations.
