# SQL Phase 1: Foundations (Detailed Guide)

> Written in simple language, with example tables, queries, outputs and explanations.
> Structure of this guide:
> - **Part A:** What is SQL and its features
> - **Part B:** Important DBMS concepts
> - **Part C:** Sample data (create it once, use it everywhere)
> - **Part D:** Module 1 - Core Query Anatomy
> - **Part E:** Module 2 - NULL Handling
> - **Part F:** Module 3 - Data Types and Functions
> - **Part G:** Common mistakes, cheat sheet, and next step

---

# PART A: What is SQL?

## A.1 Definition

**SQL = Structured Query Language.**
It is the standard language used to **talk to relational databases**: to store data, read data, change data, and control access to data.

Think of a database as a big Excel workbook:
- Each **sheet** = a **table**
- Each **column** = an **attribute** (like `name`, `salary`)
- Each **row** = one **record** (like one employee)

SQL is the language you use to *ask questions* about that workbook:
"Show me all employees in IT who earn more than 60000."

```sql
SELECT name, salary
FROM employees
WHERE department = 'IT' AND salary > 60000;
```

## A.2 A Short History
- 1970: Edgar F. Codd (IBM) proposed the **relational model**.
- 1974: IBM created **SEQUEL** (later renamed SQL).
- 1986: SQL became an **ANSI standard**; ISO followed in 1987.
- Since then: SQL-89, SQL-92, SQL:1999 (recursive queries, triggers), SQL:2003 (window functions), SQL:2016 (JSON support), and later revisions.

## A.3 Key Features of SQL

| Feature | Meaning (easy words) |
|---|---|
| **Declarative** | You say *what* you want, not *how* to get it. The database decides the best way. |
| **Standardized** | Core syntax works across MySQL, PostgreSQL, SQL Server, Oracle (with small differences). |
| **Set-based** | SQL works on whole sets of rows at once, not row by row like a loop. |
| **Powerful querying** | Filter, sort, group, join, and calculate with a few lines. |
| **Data definition** | Create and change the structure (tables, indexes, views). |
| **Data manipulation** | Insert, update, delete data. |
| **Security** | Give or take away permissions (`GRANT`, `REVOKE`). |
| **Transaction support** | Group steps so they all succeed or all fail (`COMMIT`, `ROLLBACK`). |
| **Scales to big data** | SQL is also used in Hive, Spark SQL, BigQuery, Snowflake, Redshift, Presto/Trino. |
| **Integrates everywhere** | Python, Java, BI tools, ETL tools, and data pipelines all speak SQL. |

## A.4 Categories (Sub-languages) of SQL

| Category | Full form | Purpose | Main commands |
|---|---|---|---|
| **DQL** | Data Query Language | Read data | `SELECT` |
| **DML** | Data Manipulation Language | Change data | `INSERT`, `UPDATE`, `DELETE`, `MERGE` |
| **DDL** | Data Definition Language | Change structure | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` |
| **DCL** | Data Control Language | Permissions | `GRANT`, `REVOKE` |
| **TCL** | Transaction Control Language | Control transactions | `COMMIT`, `ROLLBACK`, `SAVEPOINT` |

Quick examples:

```sql
-- DDL: create a table
CREATE TABLE students (id INT, name VARCHAR(50));

-- DML: add a row
INSERT INTO students (id, name) VALUES (1, 'Asha');

-- DQL: read rows
SELECT * FROM students;

-- DML: change a row
UPDATE students SET name = 'Asha K' WHERE id = 1;

-- DML: remove a row
DELETE FROM students WHERE id = 1;

-- DDL: delete the table completely
DROP TABLE students;
```

> **Interview note:** `DELETE` removes rows (can use `WHERE`, can be rolled back). `TRUNCATE` removes all rows quickly (no `WHERE`, resets counters, minimal logging). `DROP` removes the whole table structure and data.

## A.5 Popular SQL Databases (RDBMS)

| Database | Where it is common |
|---|---|
| MySQL / MariaDB | Web apps, startups |
| PostgreSQL | Analytics, advanced features, open-source favorite |
| SQL Server (T-SQL) | Enterprises, Microsoft stack |
| Oracle (PL/SQL) | Banks, large enterprises |
| SQLite | Mobile apps, small local databases |
| BigQuery, Snowflake, Redshift | Cloud data warehouses |
| Hive, Spark SQL, Trino | Big data / data lake querying |

## A.6 SQL vs NoSQL (Quick View)

| | SQL (Relational) | NoSQL |
|---|---|---|
| Data shape | Tables with fixed columns | Documents, key-value, graph, wide-column |
| Schema | Fixed (schema-on-write) | Flexible |
| Relationships | Strong, via keys and joins | Often embedded or manual |
| Consistency | Strong (ACID) | Often eventual (varies) |
| Examples | MySQL, PostgreSQL | MongoDB, Cassandra, Redis |

---

# PART B: Important DBMS Concepts

## B.1 Data, Database, DBMS, RDBMS

- **Data:** raw facts (`"Amit"`, `75000`).
- **Database:** an organized collection of data.
- **DBMS (Database Management System):** software that stores, retrieves, and manages the database (example: MySQL).
- **RDBMS:** a DBMS that stores data in **related tables** (rows and columns) using keys.

## B.2 Table Terminology

Example table `employees`:

| emp_id | name | department | salary |
|---|---|---|---|
| 1 | Amit Sharma | IT | 75000 |
| 2 | Priya Singh | HR | 52000 |

- **Table / Relation:** the whole grid.
- **Row / Tuple / Record:** one line (example: Amit's data).
- **Column / Attribute / Field:** one property (example: `salary`).
- **Cell:** one value (example: `75000`).
- **Degree:** number of columns (here 4). **Cardinality:** number of rows (here 2).

## B.3 Keys (Very Important)

| Key | Meaning | Example |
|---|---|---|
| **Primary Key (PK)** | Uniquely identifies each row. Cannot be NULL, cannot repeat. One per table. | `emp_id` |
| **Foreign Key (FK)** | A column that points to the primary key of another table. Creates a relationship. | `orders.customer_id` -> `customers.customer_id` |
| **Unique Key** | Values cannot repeat, but NULL is allowed (rules vary by database). | `email` |
| **Candidate Key** | Any column (or set) that *could* be the primary key. | `emp_id`, `email` |
| **Composite Key** | Primary key made from 2 or more columns. | (`order_id`, `product_id`) |
| **Surrogate Key** | An artificial key, like auto-increment id, with no business meaning. | `id` |
| **Natural Key** | A real-world value used as key. | passport number |
| **Super Key** | Any set of columns that uniquely identifies a row (candidate keys plus extras). | (`emp_id`, `name`) |

**Example of PK and FK:**

`departments`

| dept_id | dept_name |
|---|---|
| 10 | IT |
| 20 | HR |

`employees`

| emp_id | name | dept_id |
|---|---|---|
| 1 | Amit | 10 |
| 2 | Priya | 20 |

`dept_id` in `employees` is a **foreign key**. You cannot insert an employee with `dept_id = 99` if department 99 does not exist. This rule is called **referential integrity**.

## B.4 Constraints (Rules on Data)

| Constraint | Purpose | Example |
|---|---|---|
| `NOT NULL` | Value is mandatory | `name VARCHAR(50) NOT NULL` |
| `UNIQUE` | No duplicates | `email VARCHAR(100) UNIQUE` |
| `PRIMARY KEY` | NOT NULL + UNIQUE, identifies the row | `emp_id INT PRIMARY KEY` |
| `FOREIGN KEY` | Must match a value in another table | `FOREIGN KEY (dept_id) REFERENCES departments(dept_id)` |
| `CHECK` | Value must satisfy a condition | `CHECK (salary >= 0)` |
| `DEFAULT` | Value used if none is given | `status VARCHAR(10) DEFAULT 'active'` |

```sql
CREATE TABLE employees_demo (
    emp_id     INT PRIMARY KEY,
    name       VARCHAR(50) NOT NULL,
    email      VARCHAR(100) UNIQUE,
    salary     INT CHECK (salary >= 0),
    status     VARCHAR(10) DEFAULT 'active',
    dept_id    INT,
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);
```

## B.5 Relationships Between Tables

| Type | Meaning | Example |
|---|---|---|
| **One-to-One (1:1)** | One row in A matches at most one row in B | user -> user_profile |
| **One-to-Many (1:N)** | One row in A matches many rows in B | one customer -> many orders |
| **Many-to-Many (M:N)** | Many rows in A match many rows in B. Needs a **bridge table**. | students <-> courses via `enrollments` |

Many-to-many example:

`students`: (1, Asha), (2, Ravi)
`courses`: (101, SQL), (102, Python)
`enrollments` (bridge table):

| student_id | course_id |
|---|---|
| 1 | 101 |
| 1 | 102 |
| 2 | 101 |

## B.6 Normalization (Removing Redundancy)

Normalization organizes tables so the same data is not stored again and again.

**Unnormalized table (bad):**

| order_id | customer | products |
|---|---|---|
| 1 | Asha | Pen, Book |
| 2 | Ravi | Pen |

**1NF (First Normal Form):** each cell holds a single value (no lists).

| order_id | customer | product |
|---|---|---|
| 1 | Asha | Pen |
| 1 | Asha | Book |
| 2 | Ravi | Pen |

**2NF:** 1NF + no *partial dependency* (a non-key column must depend on the whole key, not part of it). Here the key is (`order_id`, `product`), but `customer` depends only on `order_id`. So we split:

`orders`: (order_id, customer)
`order_items`: (order_id, product)

**3NF:** 2NF + no *transitive dependency* (non-key columns should not depend on other non-key columns). Example: if `employees` stores `dept_id` and `dept_name`, then `dept_name` depends on `dept_id`, not on `emp_id`. Move `dept_name` to the `departments` table.

> **Memory trick:** "Every non-key attribute must depend on **the key, the whole key, and nothing but the key**."

**Denormalization:** deliberately adding redundancy for faster reads (common in data warehouses and analytics).

## B.7 ACID Properties (Transactions)

A **transaction** is a group of SQL steps treated as one unit. Example: transferring money.

```sql
START TRANSACTION;
UPDATE accounts SET balance = balance - 500 WHERE acc_id = 1;
UPDATE accounts SET balance = balance + 500 WHERE acc_id = 2;
COMMIT;   -- or ROLLBACK; to undo both
```

| Letter | Property | Meaning |
|---|---|---|
| **A** | Atomicity | All steps happen, or none happen. |
| **C** | Consistency | Data stays valid (rules and constraints are respected). |
| **I** | Isolation | Parallel transactions do not mess each other up. |
| **D** | Durability | Once committed, data survives crashes. |

## B.8 Indexes (Speed Up Reads)

An **index** is like the index at the back of a book: instead of reading every page, you jump straight to the right place.

```sql
CREATE INDEX idx_emp_dept ON employees(department);
```

- Pros: faster `WHERE`, `JOIN`, `ORDER BY`.
- Cons: uses extra storage, slows `INSERT/UPDATE/DELETE` a little.

## B.9 Views

A **view** is a saved query that behaves like a virtual table.

```sql
CREATE VIEW it_employees AS
SELECT emp_id, name, salary
FROM employees
WHERE department = 'IT';

SELECT * FROM it_employees;
```

## B.10 Schema vs Database vs Table
- **Database:** a container.
- **Schema:** a logical folder inside a database (in PostgreSQL, SQL Server, Oracle) that holds tables, views, etc. In MySQL, "schema" and "database" mean the same thing.
- **Table:** where rows live.

## B.11 OLTP vs OLAP

| | OLTP | OLAP |
|---|---|---|
| Purpose | Day-to-day transactions (orders, payments) | Analysis and reporting |
| Queries | Many small reads/writes | Few big, complex reads |
| Design | Highly normalized | Often denormalized (star schema) |
| Example | Banking app database | Data warehouse |

> Most SQL interview problems are **OLAP-style**: aggregate, rank, compare over time.

## B.12 Common Data Types

| Category | Types |
|---|---|
| Integer | `INT`, `BIGINT`, `SMALLINT` |
| Decimal | `DECIMAL(p,s)`, `NUMERIC`, `FLOAT`, `DOUBLE` |
| Text | `CHAR(n)`, `VARCHAR(n)`, `TEXT` |
| Date and time | `DATE`, `TIME`, `DATETIME / TIMESTAMP` |
| Boolean | `BOOLEAN` (or `TINYINT(1)` in MySQL) |
| Other | `JSON`, `ARRAY` (PostgreSQL), `BLOB` |

`DECIMAL(10,2)` means up to 10 digits total, 2 after the decimal point (example: `12345678.90`).

---

# PART C: Sample Data (Create Once)

Run this in **MySQL** or **PostgreSQL**. Every example below uses these tables.

```sql
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

CREATE TABLE shop_stats (
    shop      VARCHAR(5),
    sales     INT,
    visitors  INT
);

INSERT INTO shop_stats VALUES
('A', 100, 20),
('B', 50,  0),
('C', 80,  40);
```

### Table: `employees`

| emp_id | name | department | salary | hire_date | manager_id | city |
|---|---|---|---|---|---|---|
| 1 | Amit Sharma | IT | 75000 | 2020-03-15 | NULL | Mumbai |
| 2 | Priya Singh | HR | 52000 | 2019-07-01 | 1 | Delhi |
| 3 | Rahul Verma | IT | 68000 | 2021-01-10 | 1 | Pune |
| 4 | Sneha Iyer | Finance | 82000 | 2018-11-20 | 1 | Mumbai |
| 5 | Karan Mehta | IT | 60000 | 2022-05-05 | 3 | Bangalore |
| 6 | Neha Gupta | HR | NULL | 2023-02-14 | 2 | Delhi |
| 7 | Vikram Rao | Finance | 95000 | 2017-09-30 | 4 | Chennai |
| 8 | Anita Desai | Sales | 45000 | 2023-08-01 | 1 | Pune |

Notice: Neha has **no salary** (NULL) and Amit has **no manager** (NULL). We will use this for NULL practice.

### Table: `shop_stats`

| shop | sales | visitors |
|---|---|---|
| A | 100 | 20 |
| B | 50 | 0 |
| C | 80 | 40 |

---

# PART D: MODULE 1 - Core Query Anatomy

## 1.1 SELECT and FROM (Read Data)

`SELECT` chooses **which columns**. `FROM` chooses **which table**.

**Get all columns** (`*` means "everything"):

```sql
SELECT * FROM employees;
```
Output: the full table shown in Part C.

**Get only some columns:**

```sql
SELECT name, salary
FROM employees;
```

| name | salary |
|---|---|
| Amit Sharma | 75000 |
| Priya Singh | 52000 |
| Rahul Verma | 68000 |
| Sneha Iyer | 82000 |
| Karan Mehta | 60000 |
| Neha Gupta | NULL |
| Vikram Rao | 95000 |
| Anita Desai | 45000 |

**Explanation:** Only the two named columns are returned, all 8 rows.

> **Best practice:** Avoid `SELECT *` in real projects and interviews. Name the columns you need. It is faster (especially on big, columnar data) and safer.

---

## 1.2 WHERE (Filter Rows)

`WHERE` keeps only rows that satisfy a condition.

```sql
SELECT name, salary
FROM employees
WHERE salary > 60000;
```

| name | salary |
|---|---|
| Amit Sharma | 75000 |
| Rahul Verma | 68000 |
| Sneha Iyer | 82000 |
| Vikram Rao | 95000 |

**Explanation:**
- Karan earns exactly 60000, which is not `> 60000`, so he is excluded.
- Neha's salary is NULL. `NULL > 60000` is *unknown*, not true, so she is excluded too.

### Comparison Operators

| Operator | Meaning |
|---|---|
| `=` | equal |
| `<>` or `!=` | not equal |
| `>`, `<`, `>=`, `<=` | greater, less, etc. |

### AND, OR, NOT (Combining Conditions)

```sql
SELECT name, department, salary
FROM employees
WHERE department = 'IT' AND salary >= 65000;
```

| name | department | salary |
|---|---|---|
| Amit Sharma | IT | 75000 |
| Rahul Verma | IT | 68000 |

**Explanation:** Both conditions must be true. Karan is IT but earns 60000, so he fails the second condition.

**Operator precedence trap: `AND` runs before `OR`.**

```sql
-- Query 1 (no brackets)
SELECT name FROM employees
WHERE department = 'IT' OR department = 'HR' AND salary > 60000;
```
This is read as: `IT` **OR** (`HR` AND `salary > 60000`).

| name |
|---|
| Amit Sharma |
| Rahul Verma |
| Karan Mehta |

(Priya earns 52000 and Neha is NULL, so no HR employees qualify. All IT employees pass through the `OR`, including Karan at 60000.)

```sql
-- Query 2 (with brackets)
SELECT name FROM employees
WHERE (department = 'IT' OR department = 'HR') AND salary > 60000;
```
This is read as: (`IT` OR `HR`) **AND** `salary > 60000`.

| name |
|---|
| Amit Sharma |
| Rahul Verma |

> **Lesson:** Always use brackets when mixing `AND` and `OR`.

---

## 1.3 IN and NOT IN

`IN` checks if a value is in a list. It is a cleaner way to write many `OR` conditions.

```sql
SELECT name, department
FROM employees
WHERE department IN ('HR', 'Finance');
```

| name | department |
|---|---|
| Priya Singh | HR |
| Sneha Iyer | Finance |
| Neha Gupta | HR |
| Vikram Rao | Finance |

**Explanation:** Same as `department = 'HR' OR department = 'Finance'`.

---

## 1.4 BETWEEN

`BETWEEN a AND b` is **inclusive** on both ends: `>= a AND <= b`.

```sql
SELECT name, salary
FROM employees
WHERE salary BETWEEN 50000 AND 70000;
```

| name | salary |
|---|---|
| Priya Singh | 52000 |
| Rahul Verma | 68000 |
| Karan Mehta | 60000 |

**Explanation:** Amit (75000) and Anita (45000) are outside the range. Neha (NULL) is never matched.

> **Date tip:** `BETWEEN '2023-01-01' AND '2023-12-31'` works for `DATE` columns. For `TIMESTAMP` columns it can miss the last day's times. Prefer `>= '2023-01-01' AND < '2024-01-01'`.

---

## 1.5 LIKE (Pattern Matching)

| Wildcard | Meaning |
|---|---|
| `%` | any number of characters (including none) |
| `_` | exactly one character |

```sql
SELECT name FROM employees WHERE name LIKE 'A%';
```

| name |
|---|
| Amit Sharma |
| Anita Desai |

**Explanation:** Names that *start with* A.

```sql
SELECT name FROM employees WHERE name LIKE '_a%';
```

| name |
|---|
| Rahul Verma |
| Karan Mehta |

**Explanation:** The **second** letter is `a` (`_` is any one character, then `a`, then anything).

More patterns:

| Pattern | Matches |
|---|---|
| `'%a'` | ends with `a` |
| `'%an%'` | contains `an` anywhere |
| `'____'` | exactly 4 characters |
| `NOT LIKE 'A%'` | does not start with A |

> Case sensitivity depends on the database. MySQL is usually case-insensitive by default. PostgreSQL `LIKE` is case-sensitive; use `ILIKE` for case-insensitive.

**Regular expressions** (more powerful): MySQL `REGEXP`, PostgreSQL `~`.

```sql
-- MySQL: names starting with A or K
SELECT name FROM employees WHERE name REGEXP '^[AK]';
```

| name |
|---|
| Amit Sharma |
| Karan Mehta |
| Anita Desai |

---

## 1.6 ORDER BY (Sort Results)

```sql
SELECT name, salary
FROM employees
ORDER BY salary DESC;
```

MySQL result (NULLs are treated as the smallest value):

| name | salary |
|---|---|
| Vikram Rao | 95000 |
| Sneha Iyer | 82000 |
| Amit Sharma | 75000 |
| Rahul Verma | 68000 |
| Karan Mehta | 60000 |
| Priya Singh | 52000 |
| Anita Desai | 45000 |
| Neha Gupta | NULL |

- `ASC` = ascending (default), `DESC` = descending.
- **NULL ordering differs:** PostgreSQL treats NULL as the *largest* value, so in `DESC` order Neha would appear **first**. Control it with `NULLS FIRST` / `NULLS LAST` (PostgreSQL, Oracle).

**Sort by multiple columns:**

```sql
SELECT name, department, salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY department ASC, salary DESC;
```

| name | department | salary |
|---|---|---|
| Vikram Rao | Finance | 95000 |
| Sneha Iyer | Finance | 82000 |
| Priya Singh | HR | 52000 |
| Amit Sharma | IT | 75000 |
| Rahul Verma | IT | 68000 |
| Karan Mehta | IT | 60000 |
| Anita Desai | Sales | 45000 |

**Explanation:** First sorted by department A to Z. Inside each department, highest salary first.

> You can also order by column position (`ORDER BY 2`) or by an alias. Avoid position-based ordering in production code because it is easy to break.

---

## 1.7 LIMIT / TOP / FETCH FIRST (Restrict Rows)

| Database | Syntax |
|---|---|
| MySQL, PostgreSQL, SQLite | `LIMIT n` |
| SQL Server | `SELECT TOP n ...` |
| Oracle 12c+ and standard SQL | `FETCH FIRST n ROWS ONLY` |

**Top 3 highest paid employees:**

```sql
SELECT name, salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 3;
```

| name | salary |
|---|---|
| Vikram Rao | 95000 |
| Sneha Iyer | 82000 |
| Amit Sharma | 75000 |

**Pagination with OFFSET** (skip rows, then take rows). Page 2 with 3 rows per page:

```sql
SELECT name, salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 3 OFFSET 3;
```

| name | salary |
|---|---|
| Rahul Verma | 68000 |
| Karan Mehta | 60000 |
| Priya Singh | 52000 |

**Explanation:** Skip the first 3 (Vikram, Sneha, Amit), then return the next 3.

> **Always use `ORDER BY` with `LIMIT`.** Without it, which rows you get is not guaranteed.

---

## 1.8 DISTINCT (Remove Duplicates)

```sql
SELECT DISTINCT department
FROM employees;
```

| department |
|---|
| IT |
| HR |
| Finance |
| Sales |

**Explanation:** There are 8 employees but only 4 unique departments.

`DISTINCT` on multiple columns removes duplicate **combinations**:

```sql
SELECT DISTINCT department, city
FROM employees;
```

| department | city |
|---|---|
| IT | Mumbai |
| HR | Delhi |
| IT | Pune |
| Finance | Mumbai |
| IT | Bangalore |
| Finance | Chennai |
| Sales | Pune |

**Explanation:** `HR + Delhi` appears twice in the original data (Priya and Neha), so it shows once. All other combinations were already unique.

> `DISTINCT` has a cost on large data (it needs sorting or hashing). Use it only when needed. Do not use it to hide a bad join.

---

## 1.9 Aliases (AS)

An **alias** gives a temporary name to a column or table.

```sql
SELECT name AS employee_name,
       salary * 12 AS annual_salary
FROM employees
WHERE department = 'IT';
```

| employee_name | annual_salary |
|---|---|
| Amit Sharma | 900000 |
| Rahul Verma | 816000 |
| Karan Mehta | 720000 |

**Explanation:** `salary * 12` is a calculated column; the alias makes the header readable. `AS` is optional but recommended.

**Table alias:**

```sql
SELECT e.name, e.city
FROM employees e
WHERE e.city = 'Pune';
```

| name | city |
|---|---|
| Rahul Verma | Pune |
| Anita Desai | Pune |

---

## 1.10 Logical Order of Execution (VERY IMPORTANT)

You *write* SQL in this order, but the database *runs* it in a different order:

| Step | Clause | What it does |
|---|---|---|
| 1 | `FROM` (and `JOIN`) | Pick the tables |
| 2 | `WHERE` | Filter rows |
| 3 | `GROUP BY` | Make groups |
| 4 | `HAVING` | Filter groups |
| 5 | `SELECT` | Pick/calculate columns, create aliases |
| 6 | `DISTINCT` | Remove duplicates |
| 7 | `ORDER BY` | Sort |
| 8 | `LIMIT / OFFSET / TOP` | Cut rows |

**Walk-through example:**

```sql
SELECT name, salary * 12 AS annual
FROM employees
WHERE department = 'IT'
ORDER BY annual DESC
LIMIT 2;
```

1. `FROM employees`: take all 8 rows.
2. `WHERE department = 'IT'`: keep Amit, Rahul, Karan.
3. `SELECT`: calculate `annual` (900000, 816000, 720000).
4. `ORDER BY annual DESC`: sort high to low (allowed here because `SELECT` ran before `ORDER BY`).
5. `LIMIT 2`: keep the first two.

| name | annual |
|---|---|
| Amit Sharma | 900000 |
| Rahul Verma | 816000 |

**Why this matters (a classic error):**

```sql
-- FAILS: WHERE runs before SELECT, so the alias doesn't exist yet
SELECT name, salary * 12 AS annual
FROM employees
WHERE annual > 800000;
```

Fix it by repeating the expression, or use a subquery/CTE:

```sql
SELECT name, salary * 12 AS annual
FROM employees
WHERE salary * 12 > 800000;
```

| name | annual |
|---|---|
| Amit Sharma | 900000 |
| Rahul Verma | 816000 |
| Sneha Iyer | 984000 |
| Vikram Rao | 1140000 |

---

## 1.11 Module 1 Practice Problems

Try each one yourself first, then open the answer.

1. List name and city of employees who live in Mumbai or Delhi.
2. Show employees hired after 2021-01-01, newest first.
3. Show the 2 lowest paid employees (ignore NULL salaries).
4. Find employees whose name contains the letter `v` (any case).
5. Show distinct cities.
6. Show employees who do **not** work in IT, sorted by name.
7. Show name and monthly salary rounded to the nearest whole number, for salary above 50000, assuming `salary` is yearly. Alias it `monthly`.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT name, city FROM employees WHERE city IN ('Mumbai', 'Delhi');

-- 2
SELECT name, hire_date FROM employees
WHERE hire_date > '2021-01-01'
ORDER BY hire_date DESC;

-- 3
SELECT name, salary FROM employees
WHERE salary IS NOT NULL
ORDER BY salary ASC
LIMIT 2;

-- 4 (MySQL is case-insensitive by default; in PostgreSQL use ILIKE)
SELECT name FROM employees WHERE name LIKE '%v%';

-- 5
SELECT DISTINCT city FROM employees;

-- 6
SELECT name, department FROM employees
WHERE department <> 'IT'
ORDER BY name;

-- 7
SELECT name, ROUND(salary / 12) AS monthly
FROM employees
WHERE salary > 50000;
```
</details>

---

# PART E: MODULE 2 - NULL Handling (Heavily Tested)

## 2.1 What is NULL?

**NULL means "unknown" or "missing"**. It is **not** zero, **not** an empty string, and **not** the text "null".

- `0` = a known number.
- `''` = a known empty text.
- `NULL` = we do not know the value at all.

## 2.2 Three-Valued Logic

Normal logic has TRUE and FALSE. SQL has a third result: **UNKNOWN**.

| Expression | Result |
|---|---|
| `5 = 5` | TRUE |
| `5 = 6` | FALSE |
| `5 = NULL` | **UNKNOWN** |
| `NULL = NULL` | **UNKNOWN** (not TRUE!) |
| `NULL <> 5` | **UNKNOWN** |
| `NULL + 10` | **NULL** |

`WHERE` keeps a row **only if the condition is TRUE**. UNKNOWN rows are dropped.

Truth table for `AND` / `OR` with UNKNOWN:

| A | B | A AND B | A OR B |
|---|---|---|---|
| TRUE | UNKNOWN | UNKNOWN | TRUE |
| FALSE | UNKNOWN | FALSE | UNKNOWN |
| UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN |

`NOT UNKNOWN` is still UNKNOWN.

## 2.3 IS NULL and IS NOT NULL

```sql
-- WRONG: returns nothing
SELECT name FROM employees WHERE salary = NULL;
```

| name |
|---|
| (no rows) |

```sql
-- CORRECT
SELECT name, salary FROM employees WHERE salary IS NULL;
```

| name | salary |
|---|---|
| Neha Gupta | NULL |

```sql
SELECT name FROM employees WHERE manager_id IS NULL;
```

| name |
|---|
| Amit Sharma |

**Explanation:** Amit is at the top of the hierarchy, so he has no manager.

## 2.4 NULL Surprises in Comparisons

```sql
SELECT name, salary FROM employees WHERE salary <> 60000;
```

| name | salary |
|---|---|
| Amit Sharma | 75000 |
| Priya Singh | 52000 |
| Rahul Verma | 68000 |
| Sneha Iyer | 82000 |
| Vikram Rao | 95000 |
| Anita Desai | 45000 |

**Neha is missing** even though NULL is "not 60000". Reason: `NULL <> 60000` is UNKNOWN. To include her:

```sql
SELECT name, salary FROM employees
WHERE salary <> 60000 OR salary IS NULL;
```

This returns the 6 rows above plus Neha Gupta (NULL).

## 2.5 COALESCE, IFNULL / NVL, NULLIF

### COALESCE(a, b, c, ...)
Returns the **first non-NULL** value. Works in all major databases. Prefer it over `IFNULL` (MySQL), `NVL` (Oracle), `ISNULL` (SQL Server).

```sql
SELECT name, salary, COALESCE(salary, 0) AS salary_clean
FROM employees
WHERE department IN ('HR', 'IT');
```

| name | salary | salary_clean |
|---|---|---|
| Amit Sharma | 75000 | 75000 |
| Priya Singh | 52000 | 52000 |
| Rahul Verma | 68000 | 68000 |
| Karan Mehta | 60000 | 60000 |
| Neha Gupta | NULL | 0 |

**Explanation:** Where `salary` is NULL, `COALESCE` replaces it with 0.

Another use: show "No Manager" for the top boss.

```sql
SELECT name, COALESCE(CAST(manager_id AS CHAR), 'No Manager') AS manager
FROM employees
WHERE emp_id IN (1, 2);
```

| name | manager |
|---|---|
| Amit Sharma | No Manager |
| Priya Singh | 1 |

(In PostgreSQL use `CAST(manager_id AS VARCHAR)` since `CHAR` has a different meaning.)

### NULLIF(a, b)
Returns **NULL if a = b**, otherwise returns `a`. Mostly used to **avoid divide-by-zero errors**.

```sql
SELECT shop, sales, visitors,
       sales / NULLIF(visitors, 0) AS sales_per_visitor
FROM shop_stats;
```

| shop | sales | visitors | sales_per_visitor |
|---|---|---|---|
| A | 100 | 20 | 5.0000 |
| B | 50 | 0 | NULL |
| C | 80 | 40 | 2.0000 |

**Explanation:** For shop B, `NULLIF(0, 0)` becomes NULL, and anything divided by NULL is NULL (instead of an error in PostgreSQL / SQL Server). (The number of decimals shown depends on the database.)

## 2.6 NULL in Aggregates

Aggregate functions **ignore NULL values** (except `COUNT(*)`).

```sql
SELECT COUNT(*)               AS total_rows,
       COUNT(salary)          AS salary_not_null,
       COUNT(DISTINCT department) AS unique_departments,
       SUM(salary)            AS total_salary,
       AVG(salary)            AS avg_salary
FROM employees;
```

| total_rows | salary_not_null | unique_departments | total_salary | avg_salary |
|---|---|---|---|---|
| 8 | 7 | 4 | 477000 | 68142.8571 |

**Explanation:**
- `COUNT(*)` counts all rows: 8.
- `COUNT(salary)` skips Neha's NULL: 7.
- `COUNT(DISTINCT department)`: IT, HR, Finance, Sales = 4.
- `AVG(salary)` = 477000 / **7** (not 8) = 68142.86, because NULL is ignored.
- If you want NULL treated as 0, the average becomes `AVG(COALESCE(salary, 0))` = 477000 / 8 = **59625**.

| Function | Counts what |
|---|---|
| `COUNT(*)` | every row |
| `COUNT(col)` | rows where `col` is not NULL |
| `COUNT(DISTINCT col)` | unique non-NULL values |

## 2.7 The NOT IN + NULL Trap (Classic Interview Question)

**Goal:** find employees who are **not** a manager of anyone.

```sql
-- LOOKS right, but returns NOTHING
SELECT emp_id, name
FROM employees
WHERE emp_id NOT IN (SELECT manager_id FROM employees);
```

| emp_id | name |
|---|---|
| (no rows) | |

**Why?** The subquery returns the list `(NULL, 1, 1, 1, 3, 2, 4, 1)`. It contains a NULL. For any `emp_id`, `emp_id NOT IN (..., NULL)` means `emp_id <> 1 AND emp_id <> 3 AND ... AND emp_id <> NULL`. The last part is UNKNOWN, so the whole condition is never TRUE.

**Fix 1: filter NULLs out of the subquery**

```sql
SELECT emp_id, name
FROM employees
WHERE emp_id NOT IN (SELECT manager_id FROM employees WHERE manager_id IS NOT NULL);
```

| emp_id | name |
|---|---|
| 5 | Karan Mehta |
| 6 | Neha Gupta |
| 7 | Vikram Rao |
| 8 | Anita Desai |

**Fix 2 (preferred): use `NOT EXISTS`**

```sql
SELECT e.emp_id, e.name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM employees m WHERE m.manager_id = e.emp_id
);
```

Same 4 rows. `NOT EXISTS` is safe with NULLs.

> **Rule:** Prefer `NOT EXISTS` (or `LEFT JOIN ... IS NULL`) over `NOT IN` when the subquery column can be NULL.

## 2.8 NULL in ORDER BY, GROUP BY, DISTINCT
- `ORDER BY`: NULL placement differs by database (see Module 1).
- `GROUP BY`: all NULLs are put in **one group**.
- `DISTINCT`: treats all NULLs as the **same** value (one NULL kept).
- Arithmetic: `NULL + 5 = NULL`, `NULL * 0 = NULL`.
- String concatenation: in many databases `CONCAT('a', NULL)` returns NULL (MySQL) but PostgreSQL's `CONCAT` ignores NULLs. Use `COALESCE` to be safe.

## 2.9 Module 2 Practice Problems

1. Find employees who have a manager.
2. Replace NULL salary with the department-agnostic default of 30000 and show name and new salary for HR employees.
3. How many employees have a missing salary? (Two different ways.)
4. Compute `sales / visitors` for all shops safely, showing 0 when visitors is 0.
5. Find employees who are not managers using `LEFT JOIN`. (Hint: self join, then check for NULL on the right side. Joins are covered in Phase 2, so treat this as a preview.)

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT name FROM employees WHERE manager_id IS NOT NULL;

-- 2
SELECT name, COALESCE(salary, 30000) AS salary
FROM employees WHERE department = 'HR';
-- Priya 52000, Neha 30000

-- 3
SELECT COUNT(*) - COUNT(salary) AS missing_salary FROM employees;   -- 1
SELECT COUNT(*) AS missing_salary FROM employees WHERE salary IS NULL; -- 1

-- 4
SELECT shop, COALESCE(sales / NULLIF(visitors, 0), 0) AS sales_per_visitor
FROM shop_stats;
-- A 5, B 0, C 2

-- 5 (preview)
SELECT e.emp_id, e.name
FROM employees e
LEFT JOIN employees m ON m.manager_id = e.emp_id
WHERE m.emp_id IS NULL;
-- Karan, Neha, Vikram, Anita
```
</details>

---

# PART F: MODULE 3 - Data Types and Functions

> Syntax differs by database. Each section shows MySQL first, with the PostgreSQL / SQL Server version noted.

## 3.1 String Functions

| Function | Example | Result |
|---|---|---|
| `UPPER(s)` / `LOWER(s)` | `UPPER('sql interview')` | `SQL INTERVIEW` |
| `LENGTH(s)` (SQL Server: `LEN`) | `LENGTH('sql interview')` | `13` |
| `CONCAT(a, b, ...)` | `CONCAT('Hello', ' ', 'SQL')` | `Hello SQL` |
| `SUBSTRING(s, start, len)` | `SUBSTRING('Data Engineer', 1, 4)` | `Data` |
| `LEFT(s, n)` | `LEFT('Data Engineer', 4)` | `Data` |
| `RIGHT(s, n)` | `RIGHT('Data Engineer', 8)` | `Engineer` |
| `TRIM(s)` | `TRIM('   hi   ')` | `hi` |
| `REPLACE(s, old, new)` | `REPLACE('2026-10-20', '-', '/')` | `2026/10/20` |
| `INSTR(s, sub)` (PG: `POSITION(sub IN s)`) | `INSTR('Amit Sharma', ' ')` | `5` |

Notes:
- `SUBSTRING` positions start at **1**, not 0.
- `||` is the standard concatenation operator in PostgreSQL / Oracle: `'a' || 'b'`.

**Real example: extract first name and last name.**

MySQL:

```sql
SELECT name,
       SUBSTRING_INDEX(name, ' ', 1)  AS first_name,
       SUBSTRING_INDEX(name, ' ', -1) AS last_name
FROM employees
WHERE department = 'IT';
```

PostgreSQL:

```sql
SELECT name,
       SPLIT_PART(name, ' ', 1) AS first_name,
       SPLIT_PART(name, ' ', 2) AS last_name
FROM employees
WHERE department = 'IT';
```

| name | first_name | last_name |
|---|---|---|
| Amit Sharma | Amit | Sharma |
| Rahul Verma | Rahul | Verma |
| Karan Mehta | Karan | Mehta |

**Portable approach** (works anywhere using `INSTR`/`POSITION`):

```sql
SELECT name,
       LEFT(name, INSTR(name, ' ') - 1) AS first_name
FROM employees
WHERE department = 'IT';
```

**Explanation:** `INSTR(name, ' ')` finds the space position (5 for "Amit Sharma"). Taking `LEFT(name, 5 - 1)` gives the first 4 characters: `Amit`.

**Another example: initials.**

```sql
SELECT name, CONCAT(LEFT(name, 1), '.') AS initial
FROM employees
WHERE emp_id <= 3;
```

| name | initial |
|---|---|
| Amit Sharma | A. |
| Priya Singh | P. |
| Rahul Verma | R. |

## 3.2 Numeric Functions

| Function | Example | Result |
|---|---|---|
| `ROUND(x, d)` | `ROUND(68142.857, 2)` | `68142.86` |
| `CEIL(x)` / `CEILING(x)` | `CEIL(4.2)` | `5` |
| `FLOOR(x)` | `FLOOR(4.8)` | `4` |
| `MOD(a, b)` or `a % b` | `MOD(17, 5)` | `2` |
| `ABS(x)` | `ABS(-25)` | `25` |
| `POWER(x, y)` | `POWER(2, 3)` | `8` |
| `SQRT(x)` | `SQRT(16)` | `4` |

`ROUND` with a negative digit rounds to tens/hundreds: `ROUND(68142, -2)` = `68100`.

**Even/odd check** with `MOD`:

```sql
SELECT emp_id, name
FROM employees
WHERE MOD(emp_id, 2) = 0;
```

| emp_id | name |
|---|---|
| 2 | Priya Singh |
| 4 | Sneha Iyer |
| 6 | Neha Gupta |
| 8 | Anita Desai |

### Integer Division Trap

| Database | `7 / 2` |
|---|---|
| PostgreSQL, SQL Server | `3` (both are integers, result is truncated) |
| MySQL | `3.5000` (use `7 DIV 2` for integer division) |
| Oracle | `3.5` |

Safe habit: when you want decimals, force it: `7 * 1.0 / 2` or `CAST(7 AS DECIMAL) / 2` = `3.5`.

Typical interview use: percentage calculation.

```sql
-- Percentage of employees earning above 60000 (out of those with a salary)
SELECT ROUND(100.0 * SUM(CASE WHEN salary > 60000 THEN 1 ELSE 0 END) / COUNT(salary), 2) AS pct_high
FROM employees;
```

| pct_high |
|---|
| 57.14 |

**Explanation:** 4 of the 7 employees with a salary earn more than 60000 (Amit, Rahul, Sneha, Vikram). 4 / 7 = 57.14%. We multiply by `100.0` (a decimal) to avoid integer division, and use `COUNT(salary)` so the NULL row is not counted.

## 3.3 Date and Time Functions

Dates are the **#1 source of dialect differences**. Learn the idea, then look up the exact name.

| Task | MySQL | PostgreSQL | SQL Server |
|---|---|---|---|
| Today | `CURDATE()` | `CURRENT_DATE` | `CAST(GETDATE() AS DATE)` |
| Now | `NOW()` | `NOW()` | `GETDATE()` |
| Extract year | `YEAR(d)` | `EXTRACT(YEAR FROM d)` | `YEAR(d)` |
| Extract month | `MONTH(d)` | `EXTRACT(MONTH FROM d)` | `MONTH(d)` |
| Add days | `DATE_ADD(d, INTERVAL 10 DAY)` | `d + INTERVAL '10 days'` | `DATEADD(DAY, 10, d)` |
| Days between | `DATEDIFF(d1, d2)` | `d1 - d2` | `DATEDIFF(DAY, d2, d1)` |
| Truncate to month | `DATE_FORMAT(d, '%Y-%m-01')` | `DATE_TRUNC('month', d)` | `DATEFROMPARTS(YEAR(d), MONTH(d), 1)` |
| Format as text | `DATE_FORMAT(d, '%d-%b-%Y')` | `TO_CHAR(d, 'DD-Mon-YYYY')` | `FORMAT(d, 'dd-MMM-yyyy')` |

Examples with fixed values (MySQL):

| Query | Result |
|---|---|
| `DATEDIFF('2026-10-20', '2026-10-01')` | `19` |
| `DATE_ADD('2026-10-20', INTERVAL 10 DAY)` | `2026-10-30` |
| `YEAR('2026-10-20')` | `2026` |
| `DATE_FORMAT('2026-10-20', '%d-%b-%Y')` | `20-Oct-2026` |

> Careful with the order of arguments: MySQL `DATEDIFF(later, earlier)`, SQL Server `DATEDIFF(unit, earlier, later)`.

**Example 1: employees hired in 2023.**

```sql
SELECT name, hire_date
FROM employees
WHERE YEAR(hire_date) = 2023;
```

| name | hire_date |
|---|---|
| Neha Gupta | 2023-02-14 |
| Anita Desai | 2023-08-01 |

> **Performance tip:** `WHERE YEAR(hire_date) = 2023` wraps the column in a function, so an index on `hire_date` usually cannot be used. A better, index-friendly version:
> ```sql
> WHERE hire_date >= '2023-01-01' AND hire_date < '2024-01-01'
> ```

**Example 2: years of service as of 2026-10-20 (MySQL).**

```sql
SELECT name, hire_date,
       TIMESTAMPDIFF(YEAR, hire_date, '2026-10-20') AS years_of_service
FROM employees;
```

| name | hire_date | years_of_service |
|---|---|---|
| Amit Sharma | 2020-03-15 | 6 |
| Priya Singh | 2019-07-01 | 7 |
| Rahul Verma | 2021-01-10 | 5 |
| Sneha Iyer | 2018-11-20 | 7 |
| Karan Mehta | 2022-05-05 | 4 |
| Neha Gupta | 2023-02-14 | 3 |
| Vikram Rao | 2017-09-30 | 9 |
| Anita Desai | 2023-08-01 | 3 |

**Explanation:** `TIMESTAMPDIFF(YEAR, start, end)` counts **completed** years. Sneha joined 2018-11-20, so on 2026-10-20 she has completed 7 years (the 8th would complete on 2026-11-20).

PostgreSQL version: `EXTRACT(YEAR FROM AGE(DATE '2026-10-20', hire_date))`.

**Example 3: group employees by hire month.**

```sql
SELECT MONTH(hire_date) AS hire_month, COUNT(*) AS employees_hired
FROM employees
GROUP BY MONTH(hire_date)
ORDER BY hire_month;
```

| hire_month | employees_hired |
|---|---|
| 1 | 1 |
| 2 | 1 |
| 3 | 1 |
| 5 | 1 |
| 7 | 1 |
| 8 | 1 |
| 9 | 1 |
| 11 | 1 |

(Each month has one hire in this small dataset. `GROUP BY` is covered fully in Phase 2.)

## 3.4 CASE WHEN (Conditional Logic)

`CASE` is SQL's `if / else if / else`. Conditions are checked **top to bottom**, and the **first match wins**.

**Searched CASE:**

```sql
SELECT name, salary,
       CASE
           WHEN salary IS NULL    THEN 'Unknown'
           WHEN salary >= 80000   THEN 'High'
           WHEN salary >= 60000   THEN 'Medium'
           ELSE 'Low'
       END AS salary_band
FROM employees;
```

| name | salary | salary_band |
|---|---|---|
| Amit Sharma | 75000 | Medium |
| Priya Singh | 52000 | Low |
| Rahul Verma | 68000 | Medium |
| Sneha Iyer | 82000 | High |
| Karan Mehta | 60000 | Medium |
| Neha Gupta | NULL | Unknown |
| Vikram Rao | 95000 | High |
| Anita Desai | 45000 | Low |

**Explanation:**
- Sneha (82000) matches `>= 80000` first, so `High`.
- Amit (75000) fails `>= 80000`, then matches `>= 60000`, so `Medium`.
- **Order matters.** If `salary IS NULL` were not checked first, Neha would fall to `ELSE` and be called `Low`, which is wrong.
- If no `ELSE` is given and nothing matches, the result is NULL.

**Simple CASE (compare one column to values):**

```sql
SELECT name, department,
       CASE department
           WHEN 'IT'      THEN 'Tech'
           WHEN 'HR'      THEN 'People'
           WHEN 'Finance' THEN 'Money'
           ELSE 'Other'
       END AS team_type
FROM employees
WHERE emp_id <= 4;
```

| name | department | team_type |
|---|---|---|
| Amit Sharma | IT | Tech |
| Priya Singh | HR | People |
| Rahul Verma | IT | Tech |
| Sneha Iyer | Finance | Money |

> Simple `CASE` uses `=` internally, so it **cannot** match NULL (use the searched form with `IS NULL`).

**CASE inside ORDER BY** (custom sort order):

```sql
SELECT name, department
FROM employees
ORDER BY CASE department
             WHEN 'Finance' THEN 1
             WHEN 'IT'      THEN 2
             WHEN 'HR'      THEN 3
             ELSE 4
         END, name;
```

| name | department |
|---|---|
| Sneha Iyer | Finance |
| Vikram Rao | Finance |
| Amit Sharma | IT |
| Karan Mehta | IT |
| Rahul Verma | IT |
| Neha Gupta | HR |
| Priya Singh | HR |
| Anita Desai | Sales |

**Explanation:** Finance first, then IT, HR, everything else. Inside each, sorted by name.

**CASE inside aggregate (conditional counting, a top interview pattern):**

```sql
SELECT
    SUM(CASE WHEN department = 'IT' THEN 1 ELSE 0 END) AS it_count,
    SUM(CASE WHEN department = 'HR' THEN 1 ELSE 0 END) AS hr_count,
    SUM(CASE WHEN city = 'Mumbai'   THEN 1 ELSE 0 END) AS mumbai_count
FROM employees;
```

| it_count | hr_count | mumbai_count |
|---|---|---|
| 3 | 2 | 2 |

**Explanation:** For each row, `CASE` produces 1 or 0; `SUM` adds them up. This turns rows into a report-style summary in one pass.

Shortcuts: `IF(cond, a, b)` in MySQL and `IIF(cond, a, b)` in SQL Server are shorthand for a simple two-way `CASE`.

## 3.5 Type Casting

Converting one data type to another.

| Style | Example |
|---|---|
| Standard | `CAST(value AS type)` |
| MySQL / SQL Server | `CONVERT(type, value)` / `CONVERT(value, type)` (argument order differs) |
| PostgreSQL | `value::type` |

```sql
SELECT CAST('123' AS INT) + 1          AS num_result,       -- 124
       CAST('2026-10-20' AS DATE)      AS date_result,      -- 2026-10-20
       CAST(75000 AS DECIMAL(10, 2))   AS decimal_result,   -- 75000.00
       CAST(emp_id AS CHAR)            AS id_as_text        -- '1'
FROM employees
WHERE emp_id = 1;
```

| num_result | date_result | decimal_result | id_as_text |
|---|---|---|---|
| 124 | 2026-10-20 | 75000.00 | 1 |

**Explanation:** The text `'123'` is converted to a number so we can add 1. The date string becomes a real date. A failed cast (like `CAST('abc' AS INT)`) raises an error in most databases.

> **Implicit casting trap:** comparing a text column to a number (`WHERE phone = 9876543210`) may force the database to convert every row and skip indexes. Match the types.

## 3.6 Module 3 Practice Problems

1. Show each employee's name in UPPERCASE with its length.
2. Show employees whose city name has more than 6 characters.
3. Show name and a tenure label: `Senior` if hired before 2020-01-01, `Mid` if hired before 2022-01-01, else `Junior`.
4. Show name and the year and month of joining as text like `2020-03`. (MySQL: `DATE_FORMAT`; PostgreSQL: `TO_CHAR`.)
5. Calculate a 10% bonus for every employee with `ROUND`, showing 0 when the salary is NULL.
6. Show employees hired in the last 3 years before 2026-10-20 (hired on or after 2023-10-20). How many are there?
7. Count employees per salary band (`High` >= 80000, `Medium` >= 60000, `Low` otherwise, ignoring NULL salaries) as a single row using conditional aggregation.

<details>
<summary>Answers</summary>

```sql
-- 1
SELECT UPPER(name) AS name_upper, LENGTH(name) AS name_len FROM employees;

-- 2  (Bangalore = 9, Chennai = 7, Mumbai = 6, Delhi = 5, Pune = 4)
SELECT name, city FROM employees WHERE LENGTH(city) > 6;
-- Karan (Bangalore), Vikram (Chennai)

-- 3
SELECT name, hire_date,
       CASE
           WHEN hire_date < '2020-01-01' THEN 'Senior'
           WHEN hire_date < '2022-01-01' THEN 'Mid'
           ELSE 'Junior'
       END AS tenure_label
FROM employees;

-- 4 (MySQL)
SELECT name, DATE_FORMAT(hire_date, '%Y-%m') AS join_month FROM employees;
-- 4 (PostgreSQL)
SELECT name, TO_CHAR(hire_date, 'YYYY-MM') AS join_month FROM employees;

-- 5
SELECT name, ROUND(COALESCE(salary, 0) * 0.10, 2) AS bonus FROM employees;

-- 6
SELECT name, hire_date FROM employees WHERE hire_date >= '2023-10-20';
-- (no rows: the latest hires are 2023-02-14 and 2023-08-01, so the count is 0)

-- 7
SELECT
  SUM(CASE WHEN salary >= 80000 THEN 1 ELSE 0 END)                    AS high_count,    -- 2
  SUM(CASE WHEN salary >= 60000 AND salary < 80000 THEN 1 ELSE 0 END) AS medium_count,  -- 3
  SUM(CASE WHEN salary < 60000 THEN 1 ELSE 0 END)                     AS low_count      -- 2
FROM employees;
```
</details>

---

# PART G: Wrap-Up

## G.1 Top Mistakes Beginners Make

| Mistake | Fix |
|---|---|
| `WHERE col = NULL` | Use `IS NULL` / `IS NOT NULL` |
| Using an alias in `WHERE` | Repeat the expression, or use a subquery/CTE |
| Mixing `AND` / `OR` without brackets | Add brackets |
| `NOT IN` with a subquery that may contain NULL | Use `NOT EXISTS` |
| `LIMIT` without `ORDER BY` | Always sort first |
| `COUNT(col)` when you meant `COUNT(*)` | Remember `COUNT(col)` skips NULL |
| Integer division giving `3` instead of `3.5` | Multiply by `1.0` or cast to decimal |
| `YEAR(col) = 2023` on large tables | Use a date range so indexes can be used |
| Using `SELECT *` everywhere | Select only the columns you need |
| Forgetting that `BETWEEN` is inclusive | Check both boundaries |

## G.2 Phase 1 Cheat Sheet

```text
Order written : SELECT -> FROM -> WHERE -> GROUP BY -> HAVING -> ORDER BY -> LIMIT
Order executed: FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> DISTINCT -> ORDER BY -> LIMIT

NULL tests    : IS NULL, IS NOT NULL            (never = NULL)
NULL helpers  : COALESCE(a,b,...), NULLIF(a,b)
Counting      : COUNT(*) all rows | COUNT(col) non-NULL | COUNT(DISTINCT col)
Pattern       : LIKE 'A%'  '%a'  '%an%'  '_a%'
Range         : BETWEEN a AND b  (inclusive)
List          : IN ('x','y')
Conditional   : CASE WHEN ... THEN ... ELSE ... END
Cast          : CAST(x AS type)
```

## G.3 Self-Test Questions (Answer Without Looking)

1. What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?
2. Why does `SELECT * FROM t WHERE x = NULL` return nothing?
3. In what order does SQL execute `WHERE`, `SELECT`, `ORDER BY`, and `GROUP BY`?
4. What does `COUNT(salary)` return if 1 of 8 salaries is NULL?
5. Why can `NOT IN` return zero rows unexpectedly?
6. Difference between `WHERE` and `HAVING`? (Preview for Phase 2.)
7. What is the difference between a primary key and a unique key?
8. What does ACID stand for?
9. What are 1NF, 2NF, and 3NF in one line each?
10. Why can `WHERE YEAR(date_col) = 2023` be slow?

## G.4 What to Do Next

1. Create the sample tables from Part C in MySQL or PostgreSQL (online options: DB Fiddle, SQLite Online, or a local install).
2. Run **every** example in this guide yourself and compare with the outputs shown.
3. Solve all practice problems without looking at the answers.
4. Solve 15 to 20 **Easy** problems on LeetCode (Database), HackerRank, or DataLemur that use `SELECT`, `WHERE`, `ORDER BY`, `CASE`, and NULL handling.
5. When you are comfortable, say **"Start Phase 2"** and we will move to **Aggregation, Joins, and Set Operations** in the same detailed style.
