# SQL Phase 2: Aggregation, Joins and Set Operations (Detailed Guide)

> Prerequisite: Phase 1 (SELECT, WHERE, NULL handling, CASE, functions).
> Modules covered:
> - **Module 4:** Aggregation and Grouping
> - **Module 5:** Joins (all types)
> - **Module 6:** Set Operations
>
> Every example shows the input table, the query, the output, and a plain-language explanation.

---

# PART A: Sample Data (Create Once)

Run this in **MySQL** or **PostgreSQL**. It includes the `employees` table from Phase 1, plus new tables.

```sql
-- ============ From Phase 1 ============
CREATE TABLE employees (
    emp_id      INT PRIMARY KEY,
    name        VARCHAR(50),
    department  VARCHAR(20),
    salary      INT,
    hire_date   DATE,
    manager_id  INT,
    city        VARCHAR(20)
);

INSERT INTO employees VALUES
(1, 'Amit Sharma', 'IT',      75000, '2020-03-15', NULL, 'Mumbai'),
(2, 'Priya Singh', 'HR',      52000, '2019-07-01', 1,    'Delhi'),
(3, 'Rahul Verma', 'IT',      68000, '2021-01-10', 1,    'Pune'),
(4, 'Sneha Iyer',  'Finance', 82000, '2018-11-20', 1,    'Mumbai'),
(5, 'Karan Mehta', 'IT',      60000, '2022-05-05', 3,    'Bangalore'),
(6, 'Neha Gupta',  'HR',      NULL,  '2023-02-14', 2,    'Delhi'),
(7, 'Vikram Rao',  'Finance', 95000, '2017-09-30', 4,    'Chennai'),
(8, 'Anita Desai', 'Sales',   45000, '2023-08-01', 1,    'Pune');

-- ============ New tables ============
CREATE TABLE departments (
    dept_name  VARCHAR(20) PRIMARY KEY,
    budget     INT
);
INSERT INTO departments VALUES
('IT', 900000), ('HR', 400000), ('Finance', 700000),
('Sales', 600000), ('Marketing', 300000);

CREATE TABLE customers (
    customer_id    INT PRIMARY KEY,
    customer_name  VARCHAR(50),
    city           VARCHAR(20)
);
INSERT INTO customers VALUES
(1, 'Asha Rao',    'Mumbai'),
(2, 'Ravi Kumar',  'Delhi'),
(3, 'Meera Nair',  'Pune'),
(4, 'Arjun Das',   'Mumbai'),
(5, 'Sita Reddy',  'Chennai');

CREATE TABLE orders (
    order_id     INT PRIMARY KEY,
    customer_id  INT,            -- NULL = guest checkout
    order_date   DATE,
    amount       INT,
    status       VARCHAR(15)
);
INSERT INTO orders VALUES
(101, 1,    '2026-01-05', 1200, 'Delivered'),
(102, 1,    '2026-01-20',  800, 'Delivered'),
(103, 2,    '2026-01-22', 1500, 'Cancelled'),
(104, 3,    '2026-02-02', 2000, 'Delivered'),
(105, 1,    '2026-02-10',  500, 'Delivered'),
(106, 2,    '2026-02-15',  700, 'Delivered'),
(107, 4,    '2026-03-01', 3000, 'Delivered'),
(108, NULL, '2026-03-05',  400, 'Delivered');

CREATE TABLE products (
    product_id    INT PRIMARY KEY,
    product_name  VARCHAR(30),
    price         INT
);
INSERT INTO products VALUES
(1, 'Notebook', 200), (2, 'Pen', 50), (3, 'Bag', 800),
(4, 'Bottle',   300), (5, 'Mouse', 600);

CREATE TABLE order_items (
    order_id    INT,
    product_id  INT,
    qty         INT
);
INSERT INTO order_items VALUES
(101, 3, 1), (101, 1, 2),
(102, 4, 2), (102, 1, 1),
(103, 3, 1), (103, 4, 1), (103, 1, 2),
(104, 3, 2), (104, 4, 1), (104, 2, 2),
(105, 4, 1), (105, 1, 1),
(106, 4, 2), (106, 2, 2),
(107, 3, 3), (107, 4, 2),
(108, 1, 2);

CREATE TABLE grades (
    grade       CHAR(1),
    min_salary  INT,
    max_salary  INT
);
INSERT INTO grades VALUES
('A', 80000, 999999),
('B', 60000, 79999),
('C', 0,     59999);

CREATE TABLE web_users (user_name VARCHAR(20));
INSERT INTO web_users VALUES ('Asha'), ('Ravi'), ('Meera'), ('Dev');

CREATE TABLE app_users (user_name VARCHAR(20));
INSERT INTO app_users VALUES ('Ravi'), ('Arjun'), ('Meera'), ('Kiran');
```

### `orders`

| order_id | customer_id | order_date | amount | status |
|---|---|---|---|---|
| 101 | 1 | 2026-01-05 | 1200 | Delivered |
| 102 | 1 | 2026-01-20 | 800 | Delivered |
| 103 | 2 | 2026-01-22 | 1500 | Cancelled |
| 104 | 3 | 2026-02-02 | 2000 | Delivered |
| 105 | 1 | 2026-02-10 | 500 | Delivered |
| 106 | 2 | 2026-02-15 | 700 | Delivered |
| 107 | 4 | 2026-03-01 | 3000 | Delivered |
| 108 | NULL | 2026-03-05 | 400 | Delivered |

### `customers`

| customer_id | customer_name | city |
|---|---|---|
| 1 | Asha Rao | Mumbai |
| 2 | Ravi Kumar | Delhi |
| 3 | Meera Nair | Pune |
| 4 | Arjun Das | Mumbai |
| 5 | Sita Reddy | Chennai |

Important facts built into this data (we will use them):
- Customer **5 (Sita)** has **no orders**.
- Order **108** has **no customer** (guest checkout, `customer_id` is NULL).
- Department **Marketing** has **no employees**.
- Product **5 (Mouse)** was **never sold**.

### `products`

| product_id | product_name | price |
|---|---|---|
| 1 | Notebook | 200 |
| 2 | Pen | 50 |
| 3 | Bag | 800 |
| 4 | Bottle | 300 |
| 5 | Mouse | 600 |

### `order_items`

| order_id | product_id | qty |
|---|---|---|
| 101 | 3 | 1 |
| 101 | 1 | 2 |
| 102 | 4 | 2 |
| 102 | 1 | 1 |
| 103 | 3 | 1 |
| 103 | 4 | 1 |
| 103 | 1 | 2 |
| 104 | 3 | 2 |
| 104 | 4 | 1 |
| 104 | 2 | 2 |
| 105 | 4 | 1 |
| 105 | 1 | 1 |
| 106 | 4 | 2 |
| 106 | 2 | 2 |
| 107 | 3 | 3 |
| 107 | 4 | 2 |
| 108 | 1 | 2 |

(For every order, `SUM(qty * price)` equals the order's `amount`. Example: order 101 = 1 Bag (800) + 2 Notebooks (400) = 1200.)

### `departments`

| dept_name | budget |
|---|---|
| IT | 900000 |
| HR | 400000 |
| Finance | 700000 |
| Sales | 600000 |
| Marketing | 300000 |

### `grades`

| grade | min_salary | max_salary |
|---|---|---|
| A | 80000 | 999999 |
| B | 60000 | 79999 |
| C | 0 | 59999 |

### `web_users` and `app_users`

| web_users.user_name |
|---|
| Asha |
| Ravi |
| Meera |
| Dev |

| app_users.user_name |
|---|
| Ravi |
| Arjun |
| Meera |
| Kiran |

---

# PART B: MODULE 4 - Aggregation and Grouping

## 4.1 What is Aggregation?

An **aggregate function** takes **many rows** and returns **one value**.

| Function | Meaning | Ignores NULL? |
|---|---|---|
| `COUNT(*)` | number of rows | No (counts every row) |
| `COUNT(col)` | number of non-NULL values | Yes |
| `COUNT(DISTINCT col)` | number of unique non-NULL values | Yes |
| `SUM(col)` | total | Yes |
| `AVG(col)` | average (sum of non-NULL / count of non-NULL) | Yes |
| `MIN(col)` / `MAX(col)` | smallest / largest (works on numbers, text, dates) | Yes |

```sql
SELECT COUNT(*)                  AS total_orders,
       COUNT(customer_id)        AS orders_with_customer,
       COUNT(DISTINCT customer_id) AS unique_customers,
       SUM(amount)               AS total_amount,
       AVG(amount)               AS avg_amount,
       MIN(amount)               AS min_amount,
       MAX(amount)               AS max_amount
FROM orders;
```

| total_orders | orders_with_customer | unique_customers | total_amount | avg_amount | min_amount | max_amount |
|---|---|---|---|---|---|---|
| 8 | 7 | 4 | 10100 | 1262.5000 | 400 | 3000 |

**Explanation:**
- There are 8 orders. Order 108 has a NULL `customer_id`, so `COUNT(customer_id)` = 7.
- Customers 1, 2, 3, 4 placed orders, so `COUNT(DISTINCT customer_id)` = 4.
- `SUM` = 1200+800+1500+2000+500+700+3000+400 = 10100. `AVG` = 10100 / 8 = 1262.5.

**MIN / MAX work on text and dates too:**

```sql
SELECT MIN(hire_date) AS first_hire, MAX(hire_date) AS last_hire,
       MIN(name) AS first_alphabetically, MAX(name) AS last_alphabetically
FROM employees;
```

| first_hire | last_hire | first_alphabetically | last_alphabetically |
|---|---|---|---|
| 2017-09-30 | 2023-08-01 | Amit Sharma | Vikram Rao |

**Empty-set behavior (interview favorite):**

```sql
SELECT COUNT(*) AS cnt, SUM(amount) AS total
FROM orders
WHERE status = 'Refunded';   -- no such rows
```

| cnt | total |
|---|---|
| 0 | NULL |

`COUNT` returns **0** on no rows, but `SUM`, `AVG`, `MIN`, `MAX` return **NULL**. Wrap with `COALESCE(SUM(amount), 0)` when you need 0.

> **Rule:** Without `GROUP BY`, an aggregate query returns exactly **one row**.

---

## 4.2 GROUP BY (Make Groups)

`GROUP BY` splits rows into **groups** with the same value, and the aggregate runs **once per group**.

Think: "for each status, count the orders."

```sql
SELECT status, COUNT(*) AS order_count, SUM(amount) AS total_amount
FROM orders
GROUP BY status
ORDER BY status;
```

| status | order_count | total_amount |
|---|---|---|
| Cancelled | 1 | 1500 |
| Delivered | 7 | 8600 |

**Explanation:** All `Delivered` rows are grouped together (7 rows: 1200+800+2000+500+700+3000+400 = 8600), and the one `Cancelled` row forms its own group.

**Employees per department:**

```sql
SELECT department,
       COUNT(*)                AS emp_count,
       COUNT(salary)           AS with_salary,
       SUM(salary)             AS total_salary,
       ROUND(AVG(salary), 2)   AS avg_salary,
       MAX(salary)             AS max_salary
FROM employees
GROUP BY department
ORDER BY department;
```

| department | emp_count | with_salary | total_salary | avg_salary | max_salary |
|---|---|---|---|---|---|
| Finance | 2 | 2 | 177000 | 88500.00 | 95000 |
| HR | 2 | 1 | 52000 | 52000.00 | 52000 |
| IT | 3 | 3 | 203000 | 67666.67 | 75000 |
| Sales | 1 | 1 | 45000 | 45000.00 | 45000 |

**Explanation:** HR has 2 employees but only 1 salary (Neha's is NULL). So HR's average is 52000 / 1, not 52000 / 2.

### The Golden Rule of GROUP BY

> Every column in `SELECT` must be either **inside an aggregate function** or **listed in `GROUP BY`**.

```sql
-- ERROR (or unpredictable result in old MySQL modes)
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;
```

Why? Finance has two names (Sneha, Vikram). Which one should the single Finance row show? SQL cannot decide, so it refuses.

### NULL forms its own group

```sql
SELECT customer_id, COUNT(*) AS orders, SUM(amount) AS total
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | orders | total |
|---|---|---|
| NULL | 1 | 400 |
| 1 | 3 | 2500 |
| 2 | 2 | 2200 |
| 3 | 1 | 2000 |
| 4 | 1 | 3000 |

(MySQL sorts NULL first in ascending order; PostgreSQL puts it last.)

### Group by multiple columns

```sql
SELECT department, city, COUNT(*) AS emp_count
FROM employees
GROUP BY department, city
ORDER BY department, city;
```

| department | city | emp_count |
|---|---|---|
| Finance | Chennai | 1 |
| Finance | Mumbai | 1 |
| HR | Delhi | 2 |
| IT | Bangalore | 1 |
| IT | Mumbai | 1 |
| IT | Pune | 1 |
| Sales | Pune | 1 |

**Explanation:** A group is a unique **combination** of department and city. Priya and Neha are both `HR + Delhi`, so that group has 2.

### Group by an expression (example: month)

```sql
-- MySQL
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
       COUNT(*) AS orders,
       SUM(amount) AS revenue
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;

-- PostgreSQL: use TO_CHAR(order_date, 'YYYY-MM')
```

| month | orders | revenue |
|---|---|---|
| 2026-01 | 3 | 3500 |
| 2026-02 | 3 | 3200 |
| 2026-03 | 2 | 3400 |

**Explanation:** 2026-01 has 1200+800+1500 = 3500, 2026-02 has 2000+500+700 = 3200, 2026-03 has 3000+400 = 3400. Total 10100.

---

## 4.3 HAVING vs WHERE

| | `WHERE` | `HAVING` |
|---|---|---|
| Filters | individual **rows** | whole **groups** |
| Runs | **before** grouping | **after** grouping |
| Can use aggregates? | No | Yes |

**Departments whose average salary is above 60000:**

```sql
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 60000
ORDER BY department;
```

| department | avg_salary |
|---|---|
| Finance | 88500.00 |
| IT | 67666.67 |

**Explanation:** First, groups are made and averages computed (Finance 88500, HR 52000, IT 67666.67, Sales 45000). Then `HAVING` throws away the groups that fail the test.

**Departments with more than one employee:**

```sql
SELECT department, COUNT(*) AS emp_count
FROM employees
GROUP BY department
HAVING COUNT(*) > 1
ORDER BY department;
```

| department | emp_count |
|---|---|
| Finance | 2 |
| HR | 2 |
| IT | 3 |

### Using WHERE and HAVING together

**Customers who have at least 2 *delivered* orders:**

```sql
SELECT customer_id, COUNT(*) AS delivered_orders
FROM orders
WHERE status = 'Delivered'          -- row filter first
GROUP BY customer_id
HAVING COUNT(*) >= 2;               -- group filter second
```

| customer_id | delivered_orders |
|---|---|
| 1 | 3 |

**Explanation, step by step:**
1. `WHERE` removes the cancelled order 103. Seven rows remain.
2. `GROUP BY customer_id` makes groups: NULL(1 row), 1(3 rows), 2(1 row: only order 106), 3(1), 4(1).
3. `HAVING COUNT(*) >= 2` keeps only customer 1.

Ravi (customer 2) has 2 orders in total, but only 1 delivered, so he is correctly excluded.

> **Rule of thumb:** If the condition does not need an aggregate, put it in `WHERE` (it is faster because fewer rows get grouped).

> **Dialect note:** MySQL lets you use a select-alias in `HAVING`; PostgreSQL and SQL Server do not. Repeating the aggregate expression is always safe.

---

## 4.4 Conditional Aggregation (Top Interview Pattern)

Use `CASE` **inside** an aggregate to compute several filtered numbers in one pass. It is the standard way to "pivot" rows into columns.

**Per customer: delivered amount, cancelled amount, order count.**

```sql
SELECT customer_id,
       SUM(CASE WHEN status = 'Delivered' THEN amount ELSE 0 END) AS delivered_amt,
       SUM(CASE WHEN status = 'Cancelled' THEN amount ELSE 0 END) AS cancelled_amt,
       COUNT(*) AS total_orders
FROM orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | delivered_amt | cancelled_amt | total_orders |
|---|---|---|---|
| 1 | 2500 | 0 | 3 |
| 2 | 700 | 1500 | 2 |
| 3 | 2000 | 0 | 1 |
| 4 | 3000 | 0 | 1 |

**Explanation:** For each row, `CASE` returns the amount if the status matches, else 0. `SUM` adds those per customer. Ravi: delivered 700 (order 106), cancelled 1500 (order 103).

**Monthly pivot of order counts by status:**

```sql
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
       COUNT(CASE WHEN status = 'Delivered' THEN 1 END) AS delivered,
       COUNT(CASE WHEN status = 'Cancelled' THEN 1 END) AS cancelled
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

| month | delivered | cancelled |
|---|---|---|
| 2026-01 | 2 | 1 |
| 2026-02 | 3 | 0 |
| 2026-03 | 2 | 0 |

**Explanation:** `COUNT(CASE WHEN ... THEN 1 END)` works because `CASE` without `ELSE` returns NULL, and `COUNT` skips NULLs. So it counts only matching rows. (Equivalent to `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`.)

---

## 4.5 Percentage and Ratio per Group

**Share of revenue by status:**

```sql
SELECT status,
       SUM(amount) AS total,
       ROUND(100.0 * SUM(amount) / (SELECT SUM(amount) FROM orders), 2) AS pct_of_total
FROM orders
GROUP BY status
ORDER BY status;
```

| status | total | pct_of_total |
|---|---|---|
| Cancelled | 1500 | 14.85 |
| Delivered | 8600 | 85.15 |

**Explanation:** 1500 / 10100 = 14.85%, and 8600 / 10100 = 85.15%. The inner `(SELECT SUM(amount) FROM orders)` gives the grand total (10100). We multiply by `100.0` to avoid integer division. (In Phase 4 you will learn the window-function version: `SUM(SUM(amount)) OVER ()`.)

**Cancellation rate (ratio of two counts):**

```sql
SELECT ROUND(100.0 * SUM(CASE WHEN status = 'Cancelled' THEN 1 ELSE 0 END) / COUNT(*), 2) AS cancel_rate_pct
FROM orders;
```

| cancel_rate_pct |
|---|
| 12.50 |

**Explanation:** 1 cancelled out of 8 orders = 12.5%.

---

## 4.6 String Aggregation

Combine the values of a group into one text value.

| Database | Function |
|---|---|
| MySQL | `GROUP_CONCAT(col ORDER BY col SEPARATOR ', ')` |
| PostgreSQL | `STRING_AGG(col, ', ' ORDER BY col)` |
| SQL Server | `STRING_AGG(col, ', ') WITHIN GROUP (ORDER BY col)` |
| Oracle | `LISTAGG(col, ', ') WITHIN GROUP (ORDER BY col)` |

```sql
-- MySQL
SELECT department,
       GROUP_CONCAT(name ORDER BY name SEPARATOR ', ') AS members
FROM employees
GROUP BY department
ORDER BY department;
```

| department | members |
|---|---|
| Finance | Sneha Iyer, Vikram Rao |
| HR | Neha Gupta, Priya Singh |
| IT | Amit Sharma, Karan Mehta, Rahul Verma |
| Sales | Anita Desai |

---

## 4.7 ROLLUP, CUBE, GROUPING SETS (Subtotals)

`ROLLUP` adds **subtotal and grand total rows** to a grouped result.

```sql
-- MySQL
SELECT COALESCE(department, 'ALL DEPARTMENTS') AS department,
       SUM(salary) AS total_salary
FROM employees
GROUP BY department WITH ROLLUP;

-- PostgreSQL / SQL Server / Oracle:  GROUP BY ROLLUP(department)
```

| department | total_salary |
|---|---|
| Finance | 177000 |
| HR | 52000 |
| IT | 203000 |
| Sales | 45000 |
| ALL DEPARTMENTS | 477000 |

**Explanation:** The last row is the grand total, produced by `ROLLUP`. Its `department` is NULL, which `COALESCE` relabels. (If a real group could also be NULL, use `GROUPING(department)` to tell them apart.)

| Feature | What it produces |
|---|---|
| `ROLLUP(a, b)` | totals for (a,b), then (a), then grand total |
| `CUBE(a, b)` | totals for every combination: (a,b), (a), (b), grand total |
| `GROUPING SETS ((a), (b))` | exactly the groupings you list |

---

## 4.8 Aggregation Pitfalls Summary

| Pitfall | What to do |
|---|---|
| Selecting a non-grouped column | Add it to `GROUP BY` or wrap it in an aggregate |
| `COUNT(col)` vs `COUNT(*)` | `COUNT(col)` skips NULL |
| `AVG` ignoring NULL | Use `AVG(COALESCE(col,0))` if NULL should count as 0 |
| `SUM` returns NULL on empty set | `COALESCE(SUM(x), 0)` |
| Integer division in percentages | Multiply by `100.0` first |
| Filtering aggregates in `WHERE` | Use `HAVING` |
| Joining first and then aggregating | Beware of **fan-out** (see 5.12) |

---

## 4.9 Module 4 Practice Problems

1. For **delivered** orders only: total amount, count, average (rounded to 2 decimals), minimum and maximum.
2. List customers (by `customer_id`, ignoring guest orders) with **at least 2 orders** of any status, showing the order count.
3. Monthly revenue of **delivered** orders as `YYYY-MM`.
4. Departments whose **total salary exceeds 150000**.
5. Show for each customer the count of delivered and cancelled orders in two separate columns.
6. For each product (use `order_items` only), show total units sold, sorted from highest to lowest.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT SUM(amount) AS total, COUNT(*) AS cnt, ROUND(AVG(amount), 2) AS avg_amt,
       MIN(amount) AS min_amt, MAX(amount) AS max_amt
FROM orders WHERE status = 'Delivered';
-- 8600 | 7 | 1228.57 | 400 | 3000

-- 2
SELECT customer_id, COUNT(*) AS order_count
FROM orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id
HAVING COUNT(*) >= 2;
-- 1 -> 3, 2 -> 2

-- 3 (MySQL)
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month, SUM(amount) AS revenue
FROM orders WHERE status = 'Delivered'
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
-- 2026-01 2000 | 2026-02 3200 | 2026-03 3400

-- 4
SELECT department, SUM(salary) AS total_salary
FROM employees
GROUP BY department
HAVING SUM(salary) > 150000;
-- Finance 177000, IT 203000

-- 5
SELECT customer_id,
       COUNT(CASE WHEN status = 'Delivered' THEN 1 END) AS delivered,
       COUNT(CASE WHEN status = 'Cancelled' THEN 1 END) AS cancelled
FROM orders
WHERE customer_id IS NOT NULL
GROUP BY customer_id;
-- 1: 3,0 | 2: 1,1 | 3: 1,0 | 4: 1,0

-- 6
SELECT product_id, SUM(qty) AS units_sold
FROM order_items
GROUP BY product_id
ORDER BY units_sold DESC;
-- 4 -> 9 | 1 -> 8 | 3 -> 7 | 2 -> 4
```
</details>

---

# PART C: MODULE 5 - Joins

## 5.1 Why Joins?

Good databases split data into several tables (normalization). Customer details live in `customers`, their purchases in `orders`. A **join** combines rows from two (or more) tables **based on a related column**, usually a primary key matched to a foreign key.

```text
customers.customer_id  (PK)  <----  orders.customer_id  (FK)
```

## 5.2 Join Types at a Glance

| Join | Returns |
|---|---|
| `INNER JOIN` | only rows that match in **both** tables |
| `LEFT JOIN` | **all** rows from the left table + matches from the right (NULL if no match) |
| `RIGHT JOIN` | **all** rows from the right table + matches from the left (NULL if no match) |
| `FULL OUTER JOIN` | **all** rows from both tables, matched where possible |
| `CROSS JOIN` | every row of A paired with every row of B (Cartesian product) |
| `SELF JOIN` | a table joined to **itself** (using aliases) |

Think of two circles (A = left table, B = right table):
- INNER = the overlap
- LEFT = all of A (overlap + A-only)
- RIGHT = all of B (overlap + B-only)
- FULL = A and B together

Basic syntax:

```sql
SELECT ...
FROM   table_a a
JOIN   table_b b ON a.key = b.key;
```

Always give tables short aliases and prefix columns (`a.name`) to avoid "ambiguous column" errors.

---

## 5.3 INNER JOIN

Keeps only matching rows.

```sql
SELECT c.customer_name, o.order_id, o.amount
FROM customers c
INNER JOIN orders o
        ON c.customer_id = o.customer_id
ORDER BY o.order_id;
```

| customer_name | order_id | amount |
|---|---|---|
| Asha Rao | 101 | 1200 |
| Asha Rao | 102 | 800 |
| Ravi Kumar | 103 | 1500 |
| Meera Nair | 104 | 2000 |
| Asha Rao | 105 | 500 |
| Ravi Kumar | 106 | 700 |
| Arjun Das | 107 | 3000 |

**Explanation:**
- Each order is matched to its customer.
- **Sita Reddy** has no orders, so she disappears.
- **Order 108** has a NULL `customer_id` (it matches no customer), so it disappears.
- Asha appears 3 times because she has 3 orders (one-to-many).

**INNER JOIN with a filter:**

```sql
SELECT e.name, e.department, d.budget
FROM employees e
INNER JOIN departments d
        ON e.department = d.dept_name
WHERE e.salary > 70000
ORDER BY e.emp_id;
```

| name | department | budget |
|---|---|---|
| Amit Sharma | IT | 900000 |
| Sneha Iyer | Finance | 700000 |
| Vikram Rao | Finance | 700000 |

(In a real design you would join on a `dept_id` integer key, not on a name. We use the name here to keep the tables simple.)

---

## 5.4 LEFT JOIN (The Most Used Outer Join)

Keeps **every row from the left table**. If there is no match on the right, the right-side columns are NULL.

```sql
SELECT c.customer_name, o.order_id, o.amount
FROM customers c
LEFT JOIN orders o
       ON c.customer_id = o.customer_id
ORDER BY c.customer_id, o.order_id;
```

| customer_name | order_id | amount |
|---|---|---|
| Asha Rao | 101 | 1200 |
| Asha Rao | 102 | 800 |
| Asha Rao | 105 | 500 |
| Ravi Kumar | 103 | 1500 |
| Ravi Kumar | 106 | 700 |
| Meera Nair | 104 | 2000 |
| Arjun Das | 107 | 3000 |
| Sita Reddy | NULL | NULL |

**Explanation:** Sita now appears, with NULLs where order columns would be. Order 108 still does not appear because it belongs to no customer (it is on the right side only).

### Anti-Join: find rows with NO match (Pattern P5)

Add `WHERE right_key IS NULL` to keep only the left rows with no match.

**Customers who never placed an order:**

```sql
SELECT c.customer_id, c.customer_name
FROM customers c
LEFT JOIN orders o
       ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
```

| customer_id | customer_name |
|---|---|
| 5 | Sita Reddy |

**Products that were never sold:**

```sql
SELECT p.product_id, p.product_name
FROM products p
LEFT JOIN order_items oi
       ON oi.product_id = p.product_id
WHERE oi.order_id IS NULL;
```

| product_id | product_name |
|---|---|
| 5 | Mouse |

**Departments with no employees:**

```sql
SELECT d.dept_name
FROM departments d
LEFT JOIN employees e
       ON e.department = d.dept_name
WHERE e.emp_id IS NULL;
```

| dept_name |
|---|
| Marketing |

> Always test for NULL on a column that is **never NULL in a real match** (the primary key of the right table is ideal).

---

## 5.5 RIGHT JOIN

Same as LEFT JOIN, but keeps all rows from the **right** table. In practice, most people just swap the table order and use `LEFT JOIN`, because it reads more naturally.

```sql
SELECT c.customer_name, o.order_id, o.amount
FROM customers c
RIGHT JOIN orders o
        ON c.customer_id = o.customer_id
ORDER BY o.order_id;
```

| customer_name | order_id | amount |
|---|---|---|
| Asha Rao | 101 | 1200 |
| Asha Rao | 102 | 800 |
| Ravi Kumar | 103 | 1500 |
| Meera Nair | 104 | 2000 |
| Asha Rao | 105 | 500 |
| Ravi Kumar | 106 | 700 |
| Arjun Das | 107 | 3000 |
| NULL | 108 | 400 |

**Explanation:** All 8 orders appear. Order 108 has no customer, so `customer_name` is NULL. Sita is gone (she is on the left side only, and this join keeps all of the right).

---

## 5.6 FULL OUTER JOIN

Keeps all rows from both sides.

```sql
-- PostgreSQL / SQL Server / Oracle
SELECT c.customer_name, o.order_id, o.amount
FROM customers c
FULL OUTER JOIN orders o
             ON c.customer_id = o.customer_id;
```

| customer_name | order_id | amount |
|---|---|---|
| Asha Rao | 101 | 1200 |
| Asha Rao | 102 | 800 |
| Ravi Kumar | 103 | 1500 |
| Meera Nair | 104 | 2000 |
| Asha Rao | 105 | 500 |
| Ravi Kumar | 106 | 700 |
| Arjun Das | 107 | 3000 |
| NULL | 108 | 400 |
| Sita Reddy | NULL | NULL |

(Row order is not guaranteed without `ORDER BY`.)

**Explanation:** 7 matched rows + order 108 (no customer) + Sita (no orders) = 9 rows.

**MySQL does not support `FULL OUTER JOIN`.** Emulate it with `LEFT JOIN ... UNION ... RIGHT JOIN`:

```sql
SELECT c.customer_name, o.order_id, o.amount
FROM customers c LEFT JOIN orders o ON c.customer_id = o.customer_id
UNION
SELECT c.customer_name, o.order_id, o.amount
FROM customers c RIGHT JOIN orders o ON c.customer_id = o.customer_id;
```

`UNION` (not `UNION ALL`) removes the duplicated matched rows.

**Find rows that exist in only one side (FULL JOIN + IS NULL):**

```sql
SELECT c.customer_name, o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.customer_id IS NULL OR o.order_id IS NULL;
```

| customer_name | order_id |
|---|---|
| NULL | 108 |
| Sita Reddy | NULL |

---

## 5.7 CROSS JOIN

Pairs **every** row of A with **every** row of B. If A has m rows and B has n rows, you get m x n rows. There is no `ON` condition.

```sql
SELECT s.sz, c.color
FROM (SELECT 'S' AS sz UNION ALL SELECT 'M') s
CROSS JOIN (SELECT 'Red' AS color UNION ALL SELECT 'Blue' UNION ALL SELECT 'Green') c
ORDER BY s.sz DESC, c.color;
```

| sz | color |
|---|---|
| S | Blue |
| S | Green |
| S | Red |
| M | Blue |
| M | Green |
| M | Red |

**Explanation:** 2 sizes x 3 colors = 6 combinations.

**When is it useful?**
- Generate all combinations (every student x every subject, every store x every date).
- Build a "complete grid" and then `LEFT JOIN` actual data onto it so missing combinations show as 0.
- A cross join between a 1-row table (like a grand total) and a big table.

> **Danger:** Forgetting the `ON` clause (or writing an old-style `FROM a, b` without `WHERE`) silently creates a cross join and explodes the row count.

---

## 5.8 SELF JOIN

A table joined to itself. You **must** use two different aliases.

### (a) Employee and their manager (hierarchy)

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m
       ON e.manager_id = m.emp_id
ORDER BY e.emp_id;
```

| employee | manager |
|---|---|
| Amit Sharma | NULL |
| Priya Singh | Amit Sharma |
| Rahul Verma | Amit Sharma |
| Sneha Iyer | Amit Sharma |
| Karan Mehta | Rahul Verma |
| Neha Gupta | Priya Singh |
| Vikram Rao | Sneha Iyer |
| Anita Desai | Amit Sharma |

**Explanation:** Think of `e` as "the employee table" and `m` as "the manager table", both the same physical table. We match `e.manager_id` to `m.emp_id`. `LEFT JOIN` keeps Amit, who has no manager.

### (b) Compare a row with another row: employees earning more than their manager

```sql
SELECT e.name AS employee, e.salary AS emp_salary,
       m.name AS manager,  m.salary AS mgr_salary
FROM employees e
JOIN employees m
  ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

| employee | emp_salary | manager | mgr_salary |
|---|---|---|---|
| Sneha Iyer | 82000 | Amit Sharma | 75000 |
| Vikram Rao | 95000 | Sneha Iyer | 82000 |

### (c) Pairs within the same group

**All pairs of colleagues in the same department:**

```sql
SELECT e1.department, e1.name AS person_1, e2.name AS person_2
FROM employees e1
JOIN employees e2
  ON e1.department = e2.department
 AND e1.emp_id < e2.emp_id
ORDER BY e1.department, e1.emp_id, e2.emp_id;
```

| department | person_1 | person_2 |
|---|---|---|
| Finance | Sneha Iyer | Vikram Rao |
| HR | Priya Singh | Neha Gupta |
| IT | Amit Sharma | Rahul Verma |
| IT | Amit Sharma | Karan Mehta |
| IT | Rahul Verma | Karan Mehta |

**Explanation:** Without `e1.emp_id < e2.emp_id`, you would get each person paired with themselves and every pair twice (A,B and B,A). The `<` keeps each pair exactly once. (Sales has one person, so no pair.)

---

## 5.9 ON vs WHERE in Outer Joins (Very Common Bug)

For **INNER** joins, putting a condition in `ON` or `WHERE` gives the same result. For **OUTER** joins it does **not**.

**Goal:** list all customers with their *delivered* orders (customers without delivered orders should still appear).

**Version A: condition in `ON` (correct for this goal)**

```sql
SELECT c.customer_name, o.order_id, o.status
FROM customers c
LEFT JOIN orders o
       ON o.customer_id = c.customer_id
      AND o.status = 'Delivered'
ORDER BY c.customer_id, o.order_id;
```

| customer_name | order_id | status |
|---|---|---|
| Asha Rao | 101 | Delivered |
| Asha Rao | 102 | Delivered |
| Asha Rao | 105 | Delivered |
| Ravi Kumar | 106 | Delivered |
| Meera Nair | 104 | Delivered |
| Arjun Das | 107 | Delivered |
| Sita Reddy | NULL | NULL |

**Version B: condition in `WHERE`**

```sql
SELECT c.customer_name, o.order_id, o.status
FROM customers c
LEFT JOIN orders o
       ON o.customer_id = c.customer_id
WHERE o.status = 'Delivered'
ORDER BY c.customer_id, o.order_id;
```

| customer_name | order_id | status |
|---|---|---|
| Asha Rao | 101 | Delivered |
| Asha Rao | 102 | Delivered |
| Asha Rao | 105 | Delivered |
| Ravi Kumar | 106 | Delivered |
| Meera Nair | 104 | Delivered |
| Arjun Das | 107 | Delivered |

**Explanation:**
- `ON` decides **which rows match**. A left row with no match is still kept (with NULLs).
- `WHERE` runs **after** the join and removes rows. Sita's row has `status = NULL`, which fails `status = 'Delivered'`, so she is deleted. Version B accidentally turned the LEFT JOIN into an INNER JOIN.

> **Rule:** Filters on the **right** table of a LEFT JOIN go in `ON`. Filters on the **left** table can go in `WHERE`. The only `WHERE` condition on the right table that is safe is `IS NULL` (for anti-joins).

---

## 5.10 Multiple Conditions, Multiple Tables, Non-Equi Joins

### Joining 3 or more tables

**Revenue per customer, calculated from line items** (customers -> orders -> order_items -> products):

```sql
SELECT c.customer_name,
       SUM(oi.qty * p.price) AS revenue
FROM customers c
JOIN orders o       ON o.customer_id = c.customer_id
JOIN order_items oi ON oi.order_id   = o.order_id
JOIN products p     ON p.product_id  = oi.product_id
GROUP BY c.customer_name
ORDER BY revenue DESC;
```

| customer_name | revenue |
|---|---|
| Arjun Das | 3000 |
| Asha Rao | 2500 |
| Ravi Kumar | 2200 |
| Meera Nair | 2000 |

**Explanation:** Read it as a chain: customer -> their orders -> items in each order -> price of each item. Each `JOIN` adds one more table to the working result. Ravi's 2200 includes his cancelled order (1500) because we did not filter on status. Order 108 (guest) is excluded by the inner join.

**Product-wise units and revenue, keeping never-sold products:**

```sql
SELECT p.product_name,
       COALESCE(SUM(oi.qty), 0)            AS units_sold,
       COALESCE(SUM(oi.qty * p.price), 0)  AS revenue
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name
ORDER BY revenue DESC;
```

| product_name | units_sold | revenue |
|---|---|---|
| Bag | 7 | 5600 |
| Bottle | 9 | 2700 |
| Notebook | 8 | 1600 |
| Pen | 4 | 200 |
| Mouse | 0 | 0 |

**Explanation:** `LEFT JOIN` keeps Mouse (never sold), and `COALESCE` turns its NULL sums into 0. Total revenue = 5600+2700+1600+200 = 10100, matching the sum of order amounts.

### Non-Equi Join (join on a range or inequality)

`ON` does not have to use `=`. You can use `<`, `>`, `BETWEEN`.

**Assign a salary grade to each employee:**

```sql
SELECT e.name, e.salary, g.grade
FROM employees e
LEFT JOIN grades g
       ON e.salary BETWEEN g.min_salary AND g.max_salary
ORDER BY e.emp_id;
```

| name | salary | grade |
|---|---|---|
| Amit Sharma | 75000 | B |
| Priya Singh | 52000 | C |
| Rahul Verma | 68000 | B |
| Sneha Iyer | 82000 | A |
| Karan Mehta | 60000 | B |
| Neha Gupta | NULL | NULL |
| Vikram Rao | 95000 | A |
| Anita Desai | 45000 | C |

**Explanation:** Each employee is matched to the grade whose range contains the salary. Neha's NULL salary matches no range, so with `LEFT JOIN` she stays with a NULL grade (an `INNER JOIN` would drop her). Make sure ranges do not overlap, or an employee would match two grades and get duplicated.

---

## 5.11 Semi-Join and Anti-Join (EXISTS / NOT EXISTS)

- **Semi-join:** "keep rows from A that **have** a match in B" (but do not repeat A's row for every match).
- **Anti-join:** "keep rows from A that have **no** match in B."

**Customers who placed at least one order (semi-join):**

```sql
SELECT c.customer_name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

| customer_name |
|---|
| Asha Rao |
| Ravi Kumar |
| Meera Nair |
| Arjun Das |

Using `INNER JOIN` for the same question would list Asha **3 times** (once per order). `EXISTS` returns her once.

**Customers with no orders (anti-join):**

```sql
SELECT c.customer_name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.customer_id);
```

| customer_name |
|---|
| Sita Reddy |

| Style | Safe with NULLs? | Notes |
|---|---|---|
| `LEFT JOIN ... WHERE right.pk IS NULL` | Yes | Very common |
| `NOT EXISTS (...)` | Yes | Recommended |
| `NOT IN (subquery)` | **No** | Returns nothing if the subquery has a NULL |

---

## 5.12 Join Cardinality and Row Explosion (Fan-Out)

**Cardinality** = how many rows on one side match one row on the other.

| Relationship | Example | Rows after joining |
|---|---|---|
| One-to-one | user -> profile | same as before |
| One-to-many | customer -> orders | the "one" row repeats per match |
| Many-to-many | orders <-> products via `order_items` | can multiply quickly |

### The Fan-Out Trap

When you join a "one" table to a "many" table, the one-side values **repeat**. Summing a one-side value after the join **double counts**.

Joining orders to their items for order 101:

| order_id | amount | product_id | qty |
|---|---|---|---|
| 101 | 1200 | 3 | 1 |
| 101 | 1200 | 1 | 2 |

The order amount **1200 now appears twice**.

**Wrong total revenue:**

```sql
SELECT SUM(o.amount) AS total_revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;
```

| total_revenue |
|---|
| 23300 |

The true total is **10100**. Each order's amount was repeated once per item (for example order 103 has 3 items, so 1500 was added 3 times).

**Correct approaches:**

```sql
-- Option 1: don't join if you don't need to
SELECT SUM(amount) AS total_revenue FROM orders;           -- 10100

-- Option 2: calculate from the many-side only
SELECT SUM(oi.qty * p.price) AS total_revenue
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id;           -- 10100

-- Option 3: aggregate the many-side first, then join
SELECT o.order_id, o.amount, t.items_count
FROM orders o
JOIN (SELECT order_id, COUNT(*) AS items_count
      FROM order_items
      GROUP BY order_id) t
  ON t.order_id = o.order_id
ORDER BY o.order_id;
```

Output of Option 3:

| order_id | amount | items_count |
|---|---|---|
| 101 | 1200 | 2 |
| 102 | 800 | 2 |
| 103 | 1500 | 3 |
| 104 | 2000 | 3 |
| 105 | 500 | 2 |
| 106 | 700 | 2 |
| 107 | 3000 | 2 |
| 108 | 400 | 1 |

**Another fan-out example with departments:**

```sql
-- WRONG: sums each department budget once per employee
SELECT SUM(d.budget) AS total_budget
FROM departments d
JOIN employees e ON e.department = d.dept_name;
```

| total_budget |
|---|
| 5500000 |

IT (900000) is counted 3 times, HR 2, Finance 2, Sales 1, and Marketing 0 (no employees, so inner join drops it). Real total: `SELECT SUM(budget) FROM departments` = **2900000**.

**Pre-aggregate, then join (the safe pattern):**

```sql
SELECT d.dept_name, d.budget, COALESCE(e.headcount, 0) AS headcount
FROM departments d
LEFT JOIN (SELECT department, COUNT(*) AS headcount
           FROM employees
           GROUP BY department) e
       ON e.department = d.dept_name
ORDER BY d.dept_name;
```

| dept_name | budget | headcount |
|---|---|---|
| Finance | 700000 | 2 |
| HR | 400000 | 2 |
| IT | 900000 | 3 |
| Marketing | 300000 | 0 |
| Sales | 600000 | 1 |

> **Habit:** Before writing a join, ask: *"What is one row of my result supposed to represent?"* (the grain). If a join changes that grain, aggregate first.

---

## 5.13 JOIN + GROUP BY Patterns

**Total spent per customer, including customers with no delivered orders:**

```sql
SELECT c.customer_name,
       COUNT(o.order_id)         AS delivered_orders,
       COALESCE(SUM(o.amount),0) AS total_spent
FROM customers c
LEFT JOIN orders o
       ON o.customer_id = c.customer_id
      AND o.status = 'Delivered'
GROUP BY c.customer_id, c.customer_name
ORDER BY total_spent DESC;
```

| customer_name | delivered_orders | total_spent |
|---|---|---|
| Arjun Das | 1 | 3000 |
| Asha Rao | 3 | 2500 |
| Meera Nair | 1 | 2000 |
| Ravi Kumar | 1 | 700 |
| Sita Reddy | 0 | 0 |

**Explanation:**
- The status filter lives in `ON` so Sita is not removed.
- `COUNT(o.order_id)` counts only real orders. If you wrote `COUNT(*)`, Sita would show **1** (her single NULL-filled row). This is a very common mistake.
- Group by `customer_id` as well as the name, in case two customers share a name.

**Number of direct reports for each employee:**

```sql
SELECT m.name AS manager, COUNT(r.emp_id) AS direct_reports
FROM employees m
LEFT JOIN employees r ON r.manager_id = m.emp_id
GROUP BY m.emp_id, m.name
ORDER BY direct_reports DESC, m.emp_id;
```

| manager | direct_reports |
|---|---|
| Amit Sharma | 4 |
| Priya Singh | 1 |
| Rahul Verma | 1 |
| Sneha Iyer | 1 |
| Karan Mehta | 0 |
| Neha Gupta | 0 |
| Vikram Rao | 0 |
| Anita Desai | 0 |

---

## 5.14 USING and NATURAL JOIN (Shortcuts)

```sql
-- USING works when both tables have a column with the same name
SELECT customer_id, customer_name, order_id
FROM customers
JOIN orders USING (customer_id);
```

- `USING (col)` is shorthand for `ON a.col = b.col`, and the column appears once in the output.
- `NATURAL JOIN` joins automatically on **all** same-named columns. **Avoid it**: adding a new column with a common name later silently changes the join.

---

## 5.15 Choosing the Right Join

| Question | Use |
|---|---|
| "Rows that exist in both" | `INNER JOIN` |
| "Everything from A, plus B if available" | `LEFT JOIN` |
| "Rows in A with no match in B" | `LEFT JOIN ... WHERE b.key IS NULL` or `NOT EXISTS` |
| "Rows in A that have a match in B (no duplicates)" | `EXISTS` or `IN` |
| "Everything from both, matched where possible" | `FULL OUTER JOIN` |
| "All combinations" | `CROSS JOIN` |
| "Compare rows within the same table" | `SELF JOIN` |
| "Match on ranges / inequalities" | non-equi join |

---

## 5.16 Module 5 Practice Problems

1. Show each customer's name with their **total delivered spend**, including customers with no orders (show 0), highest first.
2. Find customers who have **never placed an order**.
3. List products that **were never sold**.
4. List departments that have **no employees**.
5. Find all **pairs of employees living in the same city** (each pair once).
6. For each manager, show the **number of direct reports** (0 if none).
7. Find the **top-selling product by revenue**.
8. Show each employee with their **salary grade** and their **department budget**.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT c.customer_name, COALESCE(SUM(o.amount), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id AND o.status = 'Delivered'
GROUP BY c.customer_id, c.customer_name
ORDER BY total_spent DESC;
-- Arjun 3000, Asha 2500, Meera 2000, Ravi 700, Sita 0

-- 2
SELECT c.customer_name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
WHERE o.order_id IS NULL;
-- Sita Reddy

-- 3
SELECT p.product_name
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.product_id
WHERE oi.order_id IS NULL;
-- Mouse

-- 4
SELECT d.dept_name
FROM departments d
LEFT JOIN employees e ON e.department = d.dept_name
WHERE e.emp_id IS NULL;
-- Marketing

-- 5
SELECT e1.city, e1.name AS person_1, e2.name AS person_2
FROM employees e1
JOIN employees e2 ON e1.city = e2.city AND e1.emp_id < e2.emp_id;
-- Mumbai: Amit Sharma & Sneha Iyer
-- Delhi:  Priya Singh & Neha Gupta
-- Pune:   Rahul Verma & Anita Desai

-- 6
SELECT m.name, COUNT(r.emp_id) AS direct_reports
FROM employees m
LEFT JOIN employees r ON r.manager_id = m.emp_id
GROUP BY m.emp_id, m.name;
-- Amit 4, Priya 1, Rahul 1, Sneha 1, others 0

-- 7
SELECT p.product_name, SUM(oi.qty * p.price) AS revenue
FROM products p
JOIN order_items oi ON oi.product_id = p.product_id
GROUP BY p.product_id, p.product_name
ORDER BY revenue DESC
LIMIT 1;
-- Bag 5600

-- 8
SELECT e.name, g.grade, d.budget
FROM employees e
LEFT JOIN grades g ON e.salary BETWEEN g.min_salary AND g.max_salary
LEFT JOIN departments d ON d.dept_name = e.department
ORDER BY e.emp_id;
```
</details>

---

# PART D: MODULE 6 - Set Operations

## 6.1 What are Set Operations?

Joins combine tables **side by side** (more columns). Set operations combine query results **top to bottom** (more rows), like math sets.

**Rules for both queries:**
1. Same **number of columns**.
2. Compatible **data types** in matching positions.
3. Column names in the result come from the **first** query.
4. `ORDER BY` goes **once, at the very end**, and applies to the combined result.

| Operator | Meaning | Duplicates |
|---|---|---|
| `UNION` | all rows from A and B | removed |
| `UNION ALL` | all rows from A and B | **kept** (faster) |
| `INTERSECT` | rows in **both** A and B | removed |
| `EXCEPT` (Oracle: `MINUS`) | rows in A **not in** B | removed |

> MySQL supports `INTERSECT` and `EXCEPT` only from version **8.0.31**. Older versions need the emulations shown below.

Sample data:

| web_users | | app_users |
|---|---|---|
| Asha | | Ravi |
| Ravi | | Arjun |
| Meera | | Meera |
| Dev | | Kiran |

---

## 6.2 UNION vs UNION ALL

```sql
SELECT user_name FROM web_users
UNION
SELECT user_name FROM app_users
ORDER BY user_name;
```

| user_name |
|---|
| Arjun |
| Asha |
| Dev |
| Kiran |
| Meera |
| Ravi |

**Explanation:** 6 unique users. Ravi and Meera exist in both lists but are shown once.

```sql
SELECT user_name FROM web_users
UNION ALL
SELECT user_name FROM app_users
ORDER BY user_name;
```

| user_name |
|---|
| Arjun |
| Asha |
| Dev |
| Kiran |
| Meera |
| Meera |
| Ravi |
| Ravi |

**Explanation:** 8 rows. Nothing is removed, so Meera and Ravi appear twice.

> **Performance:** `UNION` has to sort or hash to remove duplicates. If you know there are no duplicates (or you want them), use `UNION ALL`. It is faster, and it is what you use for most "combine then aggregate" work.

---

## 6.3 INTERSECT (In Both)

```sql
SELECT user_name FROM web_users
INTERSECT
SELECT user_name FROM app_users;
```

| user_name |
|---|
| Meera |
| Ravi |

**Emulation (works in any database / old MySQL):**

```sql
-- Using EXISTS
SELECT DISTINCT w.user_name
FROM web_users w
WHERE EXISTS (SELECT 1 FROM app_users a WHERE a.user_name = w.user_name);

-- Using INNER JOIN
SELECT DISTINCT w.user_name
FROM web_users w
JOIN app_users a ON a.user_name = w.user_name;
```

---

## 6.4 EXCEPT / MINUS (In A But Not In B)

```sql
-- Web users who never used the app
SELECT user_name FROM web_users
EXCEPT
SELECT user_name FROM app_users;
```

| user_name |
|---|
| Asha |
| Dev |

```sql
-- App users who never used the web
SELECT user_name FROM app_users
EXCEPT
SELECT user_name FROM web_users;
```

| user_name |
|---|
| Arjun |
| Kiran |

**Order matters:** `A EXCEPT B` is not the same as `B EXCEPT A`.

**Emulation:**

```sql
-- NOT EXISTS
SELECT DISTINCT w.user_name
FROM web_users w
WHERE NOT EXISTS (SELECT 1 FROM app_users a WHERE a.user_name = w.user_name);

-- LEFT JOIN anti-join
SELECT DISTINCT w.user_name
FROM web_users w
LEFT JOIN app_users a ON a.user_name = w.user_name
WHERE a.user_name IS NULL;
```

---

## 6.5 NULL Behavior in Set Operations

Unlike joins and `=` comparisons, **set operations treat NULLs as equal**.

```sql
SELECT NULL AS x UNION SELECT NULL;       -- 1 row  (the two NULLs are merged)
SELECT NULL AS x INTERSECT SELECT NULL;   -- 1 row  (NULL matches NULL)
```

But with a join, `NULL = NULL` is UNKNOWN, so those rows never match. This is a classic interview question.

---

## 6.6 Set-Operation Patterns (P10, P11)

### Pattern: "Users who did A **or** B"
`UNION`. Result: Arjun, Asha, Dev, Kiran, Meera, Ravi (6 users).

### Pattern: "Users who did A **and** B"
`INTERSECT`, or `GROUP BY ... HAVING COUNT(DISTINCT source) = 2` over a `UNION ALL`:

```sql
SELECT user_name
FROM (
    SELECT user_name, 'web' AS source FROM web_users
    UNION ALL
    SELECT user_name, 'app' AS source FROM app_users
) t
GROUP BY user_name
HAVING COUNT(DISTINCT source) = 2
ORDER BY user_name;
```

| user_name |
|---|
| Meera |
| Ravi |

**Explanation:** `UNION ALL` stacks both lists and tags each row with its source. Grouping by user, a user present in both has 2 distinct sources. This scales to "did all of N actions" by changing 2 to N.

### Pattern: "Users who did A **but not** B"
`EXCEPT` / `NOT EXISTS` (see 6.4).

### Pattern: "Users who did exactly one of A or B" (symmetric difference)

```sql
(SELECT user_name FROM web_users EXCEPT SELECT user_name FROM app_users)
UNION
(SELECT user_name FROM app_users EXCEPT SELECT user_name FROM web_users);
```

| user_name |
|---|
| Asha |
| Dev |
| Arjun |
| Kiran |

### Pattern: "Combine several sources, then aggregate"
Stack tables of the same shape (for example `sales_2025` and `sales_2026`, or logs from several systems) with `UNION ALL`, and aggregate on the combined result in an outer query.

```sql
SELECT source, COUNT(*) AS users
FROM (
    SELECT 'web' AS source, user_name FROM web_users
    UNION ALL
    SELECT 'app' AS source, user_name FROM app_users
) t
GROUP BY source
ORDER BY source;
```

| source | users |
|---|---|
| app | 4 |
| web | 4 |

### Pattern: Unpivot with UNION ALL
Turn columns into rows. Example: a table `scores(student, math, science)` becomes `(student, subject, score)`:

```sql
SELECT student, 'math'    AS subject, math    AS score FROM scores
UNION ALL
SELECT student, 'science' AS subject, science AS score FROM scores;
```

---

## 6.7 Module 6 Practice Problems

1. List users who used **both** web and app.
2. List users who used the **web only**.
3. Count the **total distinct users** across both platforms.
4. List users who used **exactly one** platform.
5. Using `UNION ALL`, list all `customer_id` values from `orders` and `customers` together, and show how many times each appears (hint: `GROUP BY`).

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT user_name FROM web_users INTERSECT SELECT user_name FROM app_users;
-- Meera, Ravi

-- 2
SELECT user_name FROM web_users EXCEPT SELECT user_name FROM app_users;
-- Asha, Dev

-- 3
SELECT COUNT(*) AS distinct_users
FROM (SELECT user_name FROM web_users UNION SELECT user_name FROM app_users) t;
-- 6

-- 4
SELECT user_name
FROM (SELECT user_name FROM web_users UNION ALL SELECT user_name FROM app_users) t
GROUP BY user_name
HAVING COUNT(*) = 1;
-- Asha, Dev, Arjun, Kiran

-- 5
SELECT customer_id, COUNT(*) AS appearances
FROM (
    SELECT customer_id FROM orders
    UNION ALL
    SELECT customer_id FROM customers
) t
GROUP BY customer_id
ORDER BY customer_id;
-- NULL 1 | 1 4 | 2 3 | 3 2 | 4 2 | 5 1
```
</details>

---

# PART E: Phase 2 Wrap-Up

## E.1 Top Mistakes in Phase 2

| Mistake | Fix |
|---|---|
| Non-aggregated column missing from `GROUP BY` | Group by it or aggregate it |
| Aggregate in `WHERE` | Move it to `HAVING` |
| Filter on right table in `WHERE` after `LEFT JOIN` | Move the filter into `ON` |
| `COUNT(*)` after `LEFT JOIN` counts unmatched rows as 1 | `COUNT(right_table.pk)` |
| Summing a parent column after joining to child rows | Aggregate the child first, or sum from one side only |
| Forgetting that `NULL = NULL` never matches in a join | Use `COALESCE` or `IS NOT DISTINCT FROM` / `<=>` |
| `NOT IN` with NULLs in the subquery | `NOT EXISTS` |
| `UNION` when `UNION ALL` is enough | Use `UNION ALL` unless you need de-duplication |
| Self join without a tie-breaker (`<`) | Use `e1.id < e2.id` to avoid duplicate and self pairs |
| Missing `ON` clause | Creates an accidental cross join |

## E.2 Phase 2 Cheat Sheet

```text
Aggregates   : COUNT(*), COUNT(col), COUNT(DISTINCT col), SUM, AVG, MIN, MAX
Grouping     : SELECT cols+aggs FROM t WHERE row_filter GROUP BY cols HAVING group_filter
Execution    : FROM/JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT
Pivot        : SUM(CASE WHEN cond THEN x ELSE 0 END)   or   COUNT(CASE WHEN cond THEN 1 END)
Subtotals    : GROUP BY ROLLUP(a, b)

INNER        : matches only
LEFT         : all left + matches
Anti-join    : LEFT JOIN ... WHERE right.pk IS NULL   |  NOT EXISTS
Semi-join    : EXISTS / IN  (no duplicates)
Self join    : two aliases; use a.id < b.id for pairs
Non-equi     : ON x BETWEEN lo AND hi
Fan-out fix  : aggregate the many-side first, then join

UNION ALL    : stack, keep duplicates
UNION        : stack, remove duplicates
INTERSECT    : in both
EXCEPT       : in first, not second
```

## E.3 Pattern Checklist (P1 to P11)

- [ ] P1 Count/sum per group
- [ ] P2 Filter groups with `HAVING`
- [ ] P3 Conditional aggregation (pivot-like reports)
- [ ] P4 Percentage / ratio per group
- [ ] P5 Find missing / unmatched records (anti-join)
- [ ] P6 Self join (managers, pairs, comparisons)
- [ ] P7 Cross join to generate combinations
- [ ] P8 Non-equi join (ranges)
- [ ] P9 Join + aggregate pitfalls (aggregate first, then join)
- [ ] P10 Combine multiple sources with `UNION ALL`, then aggregate
- [ ] P11 "Did A but not B" and "did both A and B"

## E.4 Self-Test Questions

1. Why can't you write `WHERE COUNT(*) > 1`? What do you use instead?
2. What is the difference between `COUNT(*)`, `COUNT(col)` and `COUNT(DISTINCT col)`?
3. What does `LEFT JOIN ... WHERE right.id IS NULL` return?
4. Why can moving a filter from `ON` to `WHERE` change a LEFT JOIN's result?
5. What is a fan-out, and how do you avoid double counting?
6. How do you get each pair of employees exactly once in a self join?
7. What is the difference between `UNION` and `UNION ALL`? When is each correct?
8. Do `INTERSECT` and `JOIN` treat NULLs the same way?
9. How do you emulate `FULL OUTER JOIN` in MySQL?
10. When would you use a `CROSS JOIN` in real work?

## E.5 What to Do Next

1. Create all tables from Part A and run every query yourself. Compare with the outputs shown.
2. Solve the practice problems before opening the answers.
3. Solve 25 to 30 problems on LeetCode (Database), HackerRank or DataLemur tagged with GROUP BY, JOIN, and UNION. Good Easy/Medium themes: "Customers Who Never Order", "Employees Earning More Than Their Managers", "Duplicate Emails", "Department Highest Salary" (you will fully solve this one with window functions later), "Game Play Analysis".
4. Keep your error log: for every wrong answer, write down whether it was a grain mistake, a NULL mistake, or a join-condition mistake.
5. When ready, say **"Start Phase 3"** and we will go deep into **Subqueries and CTEs** (correlated subqueries, Nth highest salary, recursive CTEs for hierarchies, and generating date series).
