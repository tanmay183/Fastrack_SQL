# 100 LeetCode SQL Questions – Pattern Wise (Data Engineering Interview Prep)

> **Dialect:** MySQL 8 (LeetCode default). Window functions & CTEs are used heavily. For PostgreSQL / SQL Server / Snowflake / BigQuery, the logic is the same – only small function names change (see cheat sheet).
> **Note:** Question numbers, difficulty and "asked in" tags are from memory / commonly reported interview experience. Please verify the LC number on leetcode.com. Some questions are 🔒 Premium.

## Legend

| Tag | Meaning |
|---|---|
| 🏢 | Commonly asked in **service-based** companies (TCS, Infosys, Wipro, Cognizant, Capgemini, Accenture, HCL, LTIMindtree) |
| 🚀 | Commonly asked in **product-based** companies (Amazon, Google, Microsoft, Uber, Flipkart, Walmart, Swiggy, PhonePe, Atlassian etc.) |
| ⭐⭐⭐ | Must-do – asked again and again |
| ⭐⭐ | Common |
| ⭐ | Good for practice / variation |
| 🔒 | LeetCode Premium |

---

## 🔥 Top 30 Most Asked Questions (Do these first!)

| # | Question | LC | Pattern | Asked in |
|---|---|---|---|---|
| Q48 | Second Highest Salary | 176 | Ranking | 🏢🚀 |
| Q49 | Nth Highest Salary | 177 | Ranking | 🏢🚀 |
| Q50 | Rank Scores | 178 | Ranking | 🏢🚀 |
| Q51 | Department Highest Salary | 184 | Ranking / Group | 🏢🚀 |
| Q52 | Department Top Three Salaries | 185 | Ranking | 🏢🚀 |
| Q59 | Consecutive Numbers | 180 | Gaps & Islands | 🏢🚀 |
| Q60 | Human Traffic of Stadium | 601 | Gaps & Islands | 🚀 |
| Q12 | Employees Earning More Than Managers | 181 | Self Join | 🏢🚀 |
| Q26 | Duplicate Emails | 182 | Group/Having | 🏢🚀 |
| Q13 | Customers Who Never Order | 183 | Anti Join | 🏢🚀 |
| Q46 | Delete Duplicate Emails | 196 | Delete/Dedup | 🏢🚀 |
| Q11 | Combine Two Tables | 175 | Join | 🏢 |
| Q21 | Rising Temperature | 197 | Self Join/Date | 🏢🚀 |
| Q68 | Trips and Users | 262 | Join + Rate | 🚀 |
| Q67 | Game Play Analysis IV | 550 | Retention | 🚀 |
| Q63 | Restaurant Growth | 1321 | Moving Avg | 🚀 |
| Q64 | Last Person to Fit in the Bus | 1204 | Running Total | 🚀 |
| Q55 | Immediate Food Delivery II | 1174 | First-row per group | 🚀 |
| Q58 | Movie Rating | 1341 | Top-1 + UNION | 🚀 |
| Q36 | Customers Who Bought All Products | 1045 | Relational Division | 🚀 |
| Q28 | Managers with ≥5 Direct Reports | 570 | Group/Having | 🏢🚀 |
| Q37 | Confirmation Rate | 1934 | Left Join + Avg | 🚀 |
| Q44 | Exchange Seats | 626 | CASE | 🏢🚀 |
| Q56 | Product Price at a Given Date | 1164 | Latest-before-date | 🚀 |
| Q54 | Product Sales Analysis III | 1070 | First-year per group | 🚀 |
| Q85 | Median Employee Salary | 569 | Median | 🚀 |
| Q65 | Game Play Analysis III | 534 | Cumulative Sum | 🚀 |
| Q33 | Monthly Transactions I | 1193 | Conditional Agg | 🏢🚀 |
| Q30 | Average Selling Price | 1251 | Weighted Avg | 🚀 |
| Q31 | Percentage of Users Attended a Contest | 1633 | Percentage | 🏢🚀 |

---

## 🧭 Pattern Index

| Pattern | Questions |
|---|---|
| 1. Basic Select, Filter & String/Date Functions | Q1 – Q10 |
| 2. Joins (Inner / Left / Self / Anti / Cross) | Q11 – Q22 |
| 3. Aggregation, GROUP BY & HAVING | Q23 – Q37 |
| 4. CASE WHEN, Pivot & Unpivot | Q38 – Q45 |
| 5. Duplicates, DELETE & UPDATE | Q46 – Q47 |
| 6. Ranking, Top-N & First/Latest per Group (Window Functions) | Q48 – Q58 |
| 7. Consecutive Rows / Gaps & Islands | Q59 – Q62 |
| 8. Running Total & Moving Average | Q63 – Q66 |
| 9. Date-based Analysis, Retention & Rates | Q67 – Q73 |
| 10. Hierarchy / Recursive CTE | Q74 – Q75 |
| 11. Subqueries, EXISTS / NOT IN & Set Logic | Q76 – Q84 |
| 12. Median, Percentile, Recursive & Advanced Windows | Q85 – Q90 |
| 13. Mixed Practice (Strings, Buckets, Windows) | Q91 – Q100 |

---

## 🧰 SQL Cheat Sheet (read before you start)

| Need | MySQL | PostgreSQL / Others |
|---|---|---|
| Top-N rows | `LIMIT n` | `LIMIT n` / `TOP n` (SQL Server) |
| Skip rows | `LIMIT 1 OFFSET 1` | same / `OFFSET .. FETCH` |
| Date difference (days) | `DATEDIFF(a, b)` | `a - b` / `DATEDIFF(day,b,a)` |
| Add days | `DATE_ADD(d, INTERVAL 1 DAY)` | `d + INTERVAL '1 day'` |
| Date → text | `DATE_FORMAT(d,'%Y-%m')` | `TO_CHAR(d,'YYYY-MM')` |
| String aggregate | `GROUP_CONCAT()` | `STRING_AGG()` |
| Null default | `IFNULL(x,0)` / `COALESCE` | `COALESCE` |
| Regex | `REGEXP` | `~` / `REGEXP_LIKE` |

**Golden rules for interviews**

1. **Logical order of execution:** `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → Window functions → DISTINCT → ORDER BY → LIMIT`. You can't use a window function in `WHERE` – wrap it in a subquery/CTE.
2. **`ROW_NUMBER` vs `RANK` vs `DENSE_RANK`:** 1,2,3,4 vs 1,1,3,4 vs 1,1,2,3.
3. **`COUNT(*)` vs `COUNT(col)`** – the latter ignores NULLs.
4. **`NOT IN` + NULL = trap.** Prefer `NOT EXISTS` or `LEFT JOIN … IS NULL`.
5. **Filter on the right table in a LEFT JOIN? Put it in `ON`, not `WHERE`**, otherwise it silently becomes an INNER JOIN.
6. `AVG(condition)` in MySQL = percentage of rows satisfying the condition (true = 1, false = 0).

---

# PATTERN 1 – Basic Select, Filter & String / Date Functions

> Warm-up questions. Mostly asked in **service-based** companies and as the first round screening.

### Q1. Big Countries (LC 595 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** A country is "big" if its area is at least 3 million km² **or** its population is at least 25 million. List the name, population and area of all big countries.

**🎯 Outcome:** One row per big country with `name, population, area`.

```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000
   OR population >= 25000000;
```

**🧠 Explanation:** The condition says "or", so any one of the two rules qualifies a country. `WHERE` filters rows before output. Tip: for performance, some people write it as two queries joined with `UNION` so each uses its own index.

---

### Q2. Recyclable and Low Fat Products (LC 1757 · Easy) — 🏢 ⭐

**📖 Question in simple words:** From the Products table find product ids that are both low fat and recyclable.

**🎯 Outcome:** A list of `product_id`.

```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y' AND recyclable = 'Y';
```

**🧠 Explanation:** Both columns hold `'Y'/'N'`. Use `AND` because both must be true.

---

### Q3. Find Customer Referee (LC 584 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Return the names of customers who were **not** referred by customer with id 2. Customers with no referee (NULL) must also be included.

**🎯 Outcome:** List of customer `name`.

```sql
SELECT name
FROM Customer
WHERE referee_id IS NULL OR referee_id <> 2;
```

**🧠 Explanation:** In SQL, `NULL <> 2` is **UNKNOWN**, not TRUE, so those rows would be dropped. That is why we explicitly add `IS NULL`. This is a classic NULL-handling interview question.

---

### Q4. Article Views I (LC 1148 · Easy) — 🏢 ⭐

**📖 Question in simple words:** Find authors who have viewed at least one of their own articles. Sort ids ascending.

**🎯 Outcome:** Unique `id` of such authors.

```sql
SELECT DISTINCT author_id AS id
FROM Views
WHERE author_id = viewer_id
ORDER BY id;
```

**🧠 Explanation:** A row where `author_id = viewer_id` means the author viewed his own article. `DISTINCT` removes repeats because one author may have multiple such rows.

---

### Q5. Invalid Tweets (LC 1683 · Easy) — 🏢 ⭐

**📖 Question in simple words:** A tweet is invalid if its content is longer than 15 characters. Return the ids of invalid tweets.

**🎯 Outcome:** `tweet_id` list.

```sql
SELECT tweet_id
FROM Tweets
WHERE CHAR_LENGTH(content) > 15;
```

**🧠 Explanation:** Use `CHAR_LENGTH` (characters) and not `LENGTH` (bytes in MySQL) so that emojis / non-English characters are counted correctly.

---

### Q6. Not Boring Movies (LC 620 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Show movies with an **odd id** and description that is **not "boring"**, sorted by rating (highest first).

**🎯 Outcome:** All columns of qualifying movies, high rating first.

```sql
SELECT *
FROM Cinema
WHERE id % 2 = 1
  AND description <> 'boring'
ORDER BY rating DESC;
```

**🧠 Explanation:** `id % 2 = 1` checks for odd number (modulo operator). Filtering and sorting are straightforward.

---

### Q7. Calculate Special Bonus (LC 1873 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** An employee gets a bonus equal to his salary if his id is odd **and** his name doesn't start with 'M'. Otherwise the bonus is 0.

**🎯 Outcome:** `employee_id, bonus` sorted by `employee_id`.

```sql
SELECT employee_id,
       CASE WHEN employee_id % 2 = 1 AND name NOT LIKE 'M%'
            THEN salary ELSE 0 END AS bonus
FROM Employees
ORDER BY employee_id;
```

**🧠 Explanation:** `CASE WHEN` is SQL's if-else. `LIKE 'M%'` = starts with M. We negate it using `NOT LIKE`.

---

### Q8. Fix Names in a Table (LC 1667 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Names are in random case (e.g. `aLice`). Make the first letter capital and the rest small.

**🎯 Outcome:** `user_id, name` sorted by `user_id`.

```sql
SELECT user_id,
       CONCAT(UPPER(LEFT(name,1)), LOWER(SUBSTRING(name,2))) AS name
FROM Users
ORDER BY user_id;
```

**🧠 Explanation:** `LEFT(name,1)` takes the first char → `UPPER`. `SUBSTRING(name,2)` takes everything from the 2nd char → `LOWER`. `CONCAT` joins them. (PostgreSQL: `INITCAP(name)`.)

---

### Q9. Patients With a Condition (LC 1527 · Easy) — 🏢 ⭐

**📖 Question in simple words:** Find patients who have Type I Diabetes, i.e., a condition code that **starts with `DIAB1`**. A patient may have multiple space-separated codes.

**🎯 Outcome:** All columns of those patients.

```sql
SELECT *
FROM Patients
WHERE conditions LIKE 'DIAB1%'
   OR conditions LIKE '% DIAB1%';
```

**🧠 Explanation:** The code may be the first word (`DIAB1%`) or come after a space (`% DIAB1%`). A plain `LIKE '%DIAB1%'` would wrongly match something like `ADIAB100`.

---

### Q10. Find Users With Valid E-Mails (LC 1517 · Easy) — 🏢🚀 ⭐

**📖 Question in simple words:** A valid email starts with a letter, may contain letters, digits, `_`, `.`, `-`, and must end with `@leetcode.com`.

**🎯 Outcome:** All columns of users with valid emails.

```sql
SELECT *
FROM Users
WHERE mail REGEXP '^[A-Za-z][A-Za-z0-9_.-]*@leetcode\\.com$';
```

**🧠 Explanation:** `^` start, first char letter, `[A-Za-z0-9_.-]*` allowed characters zero or more times, `@leetcode\\.com` literal domain (dot escaped), `$` end of string. Regex questions test whether you know pattern matching.

---

# PATTERN 2 – Joins (Inner / Left / Self / Anti / Cross)

> The most fundamental pattern. **Anti-join** (`LEFT JOIN … WHERE right IS NULL`) and **self-join** appear in almost every interview.

### Q11. Combine Two Tables (LC 175 · Easy) — 🏢 ⭐⭐⭐

**📖 Question in simple words:** Show first name, last name, city and state of every person. If the address is missing, show NULL.

**🎯 Outcome:** One row per person (even without address).

```sql
SELECT p.firstName, p.lastName, a.city, a.state
FROM Person p
LEFT JOIN Address a ON p.personId = a.personId;
```

**🧠 Explanation:** `LEFT JOIN` keeps all rows from the left table (Person). Where there's no match in Address, columns become NULL. `INNER JOIN` would drop people without address.

---

### Q12. Employees Earning More Than Their Managers (LC 181 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Each employee has a `managerId` pointing to another employee in the same table. Find employees whose salary is higher than their manager's.

**🎯 Outcome:** `Employee` names.

```sql
SELECT e.name AS Employee
FROM Employee e
JOIN Employee m ON e.managerId = m.id
WHERE e.salary > m.salary;
```

**🧠 Explanation:** **Self join** – treat the same table as two copies: `e` (employee) and `m` (manager). Join where `e.managerId = m.id`, then compare salaries.

---

### Q13. Customers Who Never Order (LC 183 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** List customers who have never placed any order.

**🎯 Outcome:** `Customers` name column.

```sql
SELECT c.name AS Customers
FROM Customers c
LEFT JOIN Orders o ON c.id = o.customerId
WHERE o.id IS NULL;
```

**🧠 Explanation:** **Anti-join pattern**: LEFT JOIN, then keep rows where the right side is NULL = no match found. Alternative: `WHERE c.id NOT IN (SELECT customerId FROM Orders)` or `NOT EXISTS`.

---

### Q14. Replace Employee ID With The Unique Identifier (LC 1378 · Easy) — 🏢 ⭐

**📖 Question in simple words:** Show each employee's `unique_id` next to their name. If an employee has none, show NULL.

**🎯 Outcome:** `unique_id, name`.

```sql
SELECT eu.unique_id, e.name
FROM Employees e
LEFT JOIN EmployeeUNI eu ON e.id = eu.id;
```

**🧠 Explanation:** Simple LEFT JOIN so that employees without a unique id are not lost.

---

### Q15. Product Sales Analysis I (LC 1068 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** For each sale, display product name, year and price.

**🎯 Outcome:** `product_name, year, price`.

```sql
SELECT p.product_name, s.year, s.price
FROM Sales s
JOIN Product p ON s.product_id = p.product_id;
```

**🧠 Explanation:** Sales stores only `product_id`. Join to Product to fetch the name (a basic fact–dimension join, which is common in data engineering).

---

### Q16. Customer Who Visited but Did Not Make Any Transactions (LC 1581 · Easy) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** For customers who visited the mall but bought nothing in that visit, count how many such visits each customer had.

**🎯 Outcome:** `customer_id, count_no_trans`.

```sql
SELECT v.customer_id, COUNT(*) AS count_no_trans
FROM Visits v
LEFT JOIN Transactions t ON v.visit_id = t.visit_id
WHERE t.transaction_id IS NULL
GROUP BY v.customer_id;
```

**🧠 Explanation:** Anti-join at visit level (visit with no transaction), then `GROUP BY` customer and count those visits.

---

### Q17. Employee Bonus (LC 577 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Report name and bonus of employees whose bonus is less than 1000. Employees with no bonus record count as well.

**🎯 Outcome:** `name, bonus`.

```sql
SELECT e.name, b.bonus
FROM Employee e
LEFT JOIN Bonus b ON e.empId = b.empId
WHERE b.bonus < 1000 OR b.bonus IS NULL;
```

**🧠 Explanation:** After LEFT JOIN, employees without a bonus row have `bonus = NULL`. `NULL < 1000` is not true, so we add `OR b.bonus IS NULL`.

---

### Q18. Students and Examinations (LC 1280 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** For **every student and every subject** (even if the student never attended that exam) show how many times the student attended that exam.

**🎯 Outcome:** `student_id, student_name, subject_name, attended_exams` – includes zeros.

```sql
SELECT s.student_id, s.student_name, sub.subject_name,
       COUNT(e.subject_name) AS attended_exams
FROM Students s
CROSS JOIN Subjects sub
LEFT JOIN Examinations e
       ON s.student_id = e.student_id
      AND sub.subject_name = e.subject_name
GROUP BY s.student_id, s.student_name, sub.subject_name
ORDER BY s.student_id, sub.subject_name;
```

**🧠 Explanation:** `CROSS JOIN` creates all (student × subject) combinations. `LEFT JOIN` attaches exam attempts. `COUNT(e.subject_name)` counts only non-NULL → gives 0 where no attempts. Pattern: **"generate all combinations, then left join facts"** – very common in reporting.

---

### Q19. Employees With Missing Information (LC 1965 · Easy) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** Some employees have a name but no salary; some have salary but no name. Find ids with either missing.

**🎯 Outcome:** `employee_id` sorted ascending.

```sql
SELECT employee_id FROM Employees
WHERE employee_id NOT IN (SELECT employee_id FROM Salaries)
UNION
SELECT employee_id FROM Salaries
WHERE employee_id NOT IN (SELECT employee_id FROM Employees)
ORDER BY employee_id;
```

**🧠 Explanation:** It's a **FULL OUTER JOIN anti-match**. MySQL has no FULL JOIN, so we do two anti-selects and `UNION` them. `UNION` also removes duplicates.

---

### Q20. Employees Whose Manager Left the Company (LC 1978 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Find employees earning less than 30000 whose manager no longer exists in the company (manager_id is present but that id is not an employee).

**🎯 Outcome:** `employee_id` sorted.

```sql
SELECT employee_id
FROM Employees
WHERE salary < 30000
  AND manager_id IS NOT NULL
  AND manager_id NOT IN (SELECT employee_id FROM Employees)
ORDER BY employee_id;
```

**🧠 Explanation:** Manager "left" = `manager_id` is not found in the `employee_id` column. `employee_id` is a primary key (never NULL) so `NOT IN` is safe here.

---

### Q21. Rising Temperature (LC 197 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Find the ids of days where the temperature was higher than the **previous day**.

**🎯 Outcome:** `id` list.

```sql
SELECT w1.id
FROM Weather w1
JOIN Weather w2
  ON DATEDIFF(w1.recordDate, w2.recordDate) = 1
WHERE w1.temperature > w2.temperature;
```

**🧠 Explanation:** Self join today (`w1`) with yesterday (`w2`) using date difference = 1. Don't use `id-1` since ids may not be in date order. Alternative: `LAG(temperature) OVER (ORDER BY recordDate)` with a check that the previous date is exactly one day before.

---

### Q22. Average Time of Process per Machine (LC 1661 · Easy) — 🚀 ⭐⭐

**📖 Question in simple words:** Each machine's process has a `start` and `end` timestamp (in two rows). Find average processing time per machine, rounded to 3 decimals.

**🎯 Outcome:** `machine_id, processing_time`.

```sql
SELECT a1.machine_id,
       ROUND(AVG(a2.timestamp - a1.timestamp), 3) AS processing_time
FROM Activity a1
JOIN Activity a2
  ON a1.machine_id = a2.machine_id
 AND a1.process_id = a2.process_id
 AND a1.activity_type = 'start'
 AND a2.activity_type = 'end'
GROUP BY a1.machine_id;
```

**🧠 Explanation:** Self-join pairs the `start` row with the matching `end` row of the same machine & process. The difference is the time taken; then average per machine. Alternative (no self-join): `SUM(CASE WHEN activity_type='end' THEN timestamp ELSE -timestamp END) / COUNT(DISTINCT process_id)` per machine.

---

# PATTERN 3 – Aggregation, GROUP BY & HAVING

> `WHERE` filters **rows**, `HAVING` filters **groups**. Expect "count / average / percentage per group" in every interview.

### Q23. Classes More Than 5 Students (LC 596 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Show the classes that have at least 5 students.

**🎯 Outcome:** `class` list.

```sql
SELECT class
FROM Courses
GROUP BY class
HAVING COUNT(DISTINCT student) >= 5;
```

**🧠 Explanation:** Group by class, count students, keep only groups with count ≥ 5 using `HAVING`. `DISTINCT` protects against duplicate (student, class) rows.

---

### Q24. Find Followers Count (LC 1729 · Easy) — 🏢 ⭐

**📖 Question in simple words:** For each user, count how many followers they have.

**🎯 Outcome:** `user_id, followers_count` sorted by `user_id`.

```sql
SELECT user_id, COUNT(follower_id) AS followers_count
FROM Followers
GROUP BY user_id
ORDER BY user_id;
```

**🧠 Explanation:** Each row is one follower of `user_id`. Group and count.

---

### Q25. Biggest Single Number (LC 619 · Easy) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** Among the numbers that appear **exactly once**, return the largest. If none, return NULL.

**🎯 Outcome:** One value `num`.

```sql
SELECT MAX(num) AS num
FROM (
    SELECT num
    FROM MyNumbers
    GROUP BY num
    HAVING COUNT(*) = 1
) t;
```

**🧠 Explanation:** Inner query keeps only numbers appearing once. Outer `MAX` on an empty set returns NULL automatically – exactly what is required.

---

### Q26. Duplicate Emails (LC 182 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Find email addresses that appear more than once.

**🎯 Outcome:** `Email` list.

```sql
SELECT email AS Email
FROM Person
GROUP BY email
HAVING COUNT(*) > 1;
```

**🧠 Explanation:** The most basic duplicate-detection template: `GROUP BY col HAVING COUNT(*) > 1`. Data engineers use it every day for data-quality checks.

---

### Q27. Actors and Directors Who Cooperated At Least Three Times (LC 1050 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Find (actor, director) pairs who worked together 3+ times.

**🎯 Outcome:** `actor_id, director_id`.

```sql
SELECT actor_id, director_id
FROM ActorDirector
GROUP BY actor_id, director_id
HAVING COUNT(*) >= 3;
```

**🧠 Explanation:** Group on a **pair** of columns. Each row is a cooperation, so count rows per pair.

---

### Q28. Managers with at Least 5 Direct Reports (LC 570 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Find names of managers who have 5 or more employees reporting directly to them.

**🎯 Outcome:** `name`.

```sql
SELECT name
FROM Employee
WHERE id IN (
    SELECT managerId
    FROM Employee
    GROUP BY managerId
    HAVING COUNT(*) >= 5
);
```

**🧠 Explanation:** The subquery counts reports per `managerId` and keeps those with ≥ 5. The outer query converts those ids into names. (A join works too.) Note `NULL` managerId forms its own group but is never equal to an `id`, so it's harmless.

---

### Q29. Project Employees I (LC 1075 · Easy) — 🏢 ⭐

**📖 Question in simple words:** For each project, find the average years of experience of the employees working on it (2 decimals).

**🎯 Outcome:** `project_id, average_years`.

```sql
SELECT p.project_id, ROUND(AVG(e.experience_years), 2) AS average_years
FROM Project p
JOIN Employee e ON p.employee_id = e.employee_id
GROUP BY p.project_id;
```

**🧠 Explanation:** Join to bring experience, group by project and take `AVG`.

---

### Q30. Average Selling Price (LC 1251 · Easy) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** The price of a product changes over time (date ranges). For each product compute the **weighted average** selling price = total revenue / total units. If no units were sold, show 0.

**🎯 Outcome:** `product_id, average_price`.

```sql
SELECT p.product_id,
       IFNULL(ROUND(SUM(p.price * u.units) / SUM(u.units), 2), 0) AS average_price
FROM Prices p
LEFT JOIN UnitsSold u
       ON p.product_id = u.product_id
      AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY p.product_id;
```

**🧠 Explanation:** **Range join** – a sale belongs to the price row whose date range contains the purchase date. Weighted average = `SUM(price*units)/SUM(units)`. LEFT JOIN keeps products with no sales (their sum is NULL → `IFNULL` gives 0).

---

### Q31. Percentage of Users Attended a Contest (LC 1633 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** For each contest, what percentage of **all users** registered? Round to 2 decimals. Sort by percentage (high→low) then contest_id.

**🎯 Outcome:** `contest_id, percentage`.

```sql
SELECT contest_id,
       ROUND(COUNT(DISTINCT user_id) * 100.0 / (SELECT COUNT(*) FROM Users), 2) AS percentage
FROM Register
GROUP BY contest_id
ORDER BY percentage DESC, contest_id;
```

**🧠 Explanation:** Numerator = registered users per contest, denominator = total users (scalar subquery). Multiply by `100.0` to avoid integer division issues.

---

### Q32. Queries Quality and Percentage (LC 1211 · Easy) — 🚀 ⭐⭐

**📖 Question in simple words:** For each query name compute **quality** = average of `rating/position`, and **poor_query_percentage** = % of rows with rating < 3.

**🎯 Outcome:** `query_name, quality, poor_query_percentage` (2 decimals).

```sql
SELECT query_name,
       ROUND(AVG(rating / position), 2)            AS quality,
       ROUND(SUM(rating < 3) * 100 / COUNT(*), 2)  AS poor_query_percentage
FROM Queries
WHERE query_name IS NOT NULL
GROUP BY query_name;
```

**🧠 Explanation:** `rating < 3` returns 1/0 in MySQL, so `SUM(...)` counts poor rows. Divide by total rows ×100 for the percentage. (Portable version: `SUM(CASE WHEN rating<3 THEN 1 ELSE 0 END)`.)

---

### Q33. Monthly Transactions I (LC 1193 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** For each month and country, report: total transactions, approved transactions, total amount, approved amount.

**🎯 Outcome:** `month, country, trans_count, approved_count, trans_total_amount, approved_total_amount`.

```sql
SELECT DATE_FORMAT(trans_date, '%Y-%m') AS month,
       country,
       COUNT(*)                                             AS trans_count,
       SUM(state = 'approved')                              AS approved_count,
       SUM(amount)                                          AS trans_total_amount,
       SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END) AS approved_total_amount
FROM Transactions
GROUP BY DATE_FORMAT(trans_date, '%Y-%m'), country;
```

**🧠 Explanation:** **Conditional aggregation** – put a condition inside `SUM`/`COUNT` to compute several metrics in one scan. It is the most useful reporting pattern in data engineering.

---

### Q34. User Activity for the Past 30 Days I (LC 1141 · Easy) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** Count daily active users for the 30 days ending 2019-07-27 (a user is active if he did any activity that day).

**🎯 Outcome:** `day, active_users`.

```sql
SELECT activity_date AS day, COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date BETWEEN DATE_SUB('2019-07-27', INTERVAL 29 DAY) AND '2019-07-27'
GROUP BY activity_date;
```

**🧠 Explanation:** "30 days ending X" = X-29 to X inclusive. `COUNT(DISTINCT user_id)` because a user can have many sessions a day. **DAU** is a classic product-analytics metric.

---

### Q35. Sales Analysis III (LC 1084 · Easy) — 🚀 ⭐⭐

**📖 Question in simple words:** Find products that were sold **only** in Q1 2019 (Jan 1 – Mar 31) and never outside.

**🎯 Outcome:** `product_id, product_name`.

```sql
SELECT p.product_id, p.product_name
FROM Product p
JOIN Sales s ON p.product_id = s.product_id
GROUP BY p.product_id, p.product_name
HAVING MIN(s.sale_date) >= '2019-01-01'
   AND MAX(s.sale_date) <= '2019-03-31';
```

**🧠 Explanation:** "Only in a range" = earliest sale ≥ start **and** latest sale ≤ end. Using `MIN/MAX` in `HAVING` avoids a NOT-IN subquery.

---

### Q36. Customers Who Bought All Products (LC 1045 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Find customers who purchased **every** product listed in the Product table.

**🎯 Outcome:** `customer_id`.

```sql
SELECT customer_id
FROM Customer
GROUP BY customer_id
HAVING COUNT(DISTINCT product_key) = (SELECT COUNT(*) FROM Product);
```

**🧠 Explanation:** **Relational division** pattern: a customer who bought all products has a distinct-product count equal to the total number of products.

---

### Q37. Confirmation Rate (LC 1934 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** For every user, confirmation rate = confirmed requests / total requests. Users with no requests have 0. Round 2 decimals.

**🎯 Outcome:** `user_id, confirmation_rate`.

```sql
SELECT s.user_id,
       ROUND(IFNULL(AVG(c.action = 'confirmed'), 0), 2) AS confirmation_rate
FROM Signups s
LEFT JOIN Confirmations c ON s.user_id = c.user_id
GROUP BY s.user_id;
```

**🧠 Explanation:** `AVG(condition)` = share of rows where condition is true. For a user without confirmations the LEFT JOIN gives NULL → `AVG` returns NULL → `IFNULL(...,0)`.

---

# PATTERN 4 – CASE WHEN, Pivot & Unpivot

### Q38. Count Salary Categories (LC 1907 · Medium) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** Bucket accounts into Low (<20000), Average (20000–50000), High (>50000) and count each bucket. All 3 buckets must appear, even with 0.

**🎯 Outcome:** 3 rows: `category, accounts_count`.

```sql
SELECT 'Low Salary' AS category,
       COUNT(CASE WHEN income < 20000 THEN 1 END) AS accounts_count
FROM Accounts
UNION ALL
SELECT 'Average Salary',
       COUNT(CASE WHEN income BETWEEN 20000 AND 50000 THEN 1 END)
FROM Accounts
UNION ALL
SELECT 'High Salary',
       COUNT(CASE WHEN income > 50000 THEN 1 END)
FROM Accounts;
```

**🧠 Explanation:** If we did `GROUP BY` on a CASE, empty buckets would vanish. Building each bucket as its own SELECT guarantees all 3 rows. `COUNT(CASE … THEN 1 END)` ignores NULL, so it counts only matching rows.

---

### Q39. Reformat Department Table (LC 1179 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Turn month rows into columns: one row per department id with columns `Jan_Revenue … Dec_Revenue`.

**🎯 Outcome:** `id, Jan_Revenue, Feb_Revenue, …, Dec_Revenue`.

```sql
SELECT id,
  SUM(CASE WHEN month='Jan' THEN revenue END) AS Jan_Revenue,
  SUM(CASE WHEN month='Feb' THEN revenue END) AS Feb_Revenue,
  SUM(CASE WHEN month='Mar' THEN revenue END) AS Mar_Revenue,
  SUM(CASE WHEN month='Apr' THEN revenue END) AS Apr_Revenue,
  SUM(CASE WHEN month='May' THEN revenue END) AS May_Revenue,
  SUM(CASE WHEN month='Jun' THEN revenue END) AS Jun_Revenue,
  SUM(CASE WHEN month='Jul' THEN revenue END) AS Jul_Revenue,
  SUM(CASE WHEN month='Aug' THEN revenue END) AS Aug_Revenue,
  SUM(CASE WHEN month='Sep' THEN revenue END) AS Sep_Revenue,
  SUM(CASE WHEN month='Oct' THEN revenue END) AS Oct_Revenue,
  SUM(CASE WHEN month='Nov' THEN revenue END) AS Nov_Revenue,
  SUM(CASE WHEN month='Dec' THEN revenue END) AS Dec_Revenue
FROM Department
GROUP BY id;
```

**🧠 Explanation:** The **PIVOT** pattern without a PIVOT keyword: `GROUP BY id` and one `SUM(CASE …)` per output column. Months with no data are NULL.

---

### Q40. Triangle Judgement (LC 610 · Easy) — 🏢 ⭐

**📖 Question in simple words:** Given three side lengths x, y, z, say `Yes` if they can form a triangle, else `No`.

**🎯 Outcome:** `x, y, z, triangle`.

```sql
SELECT x, y, z,
       CASE WHEN x + y > z AND x + z > y AND y + z > x THEN 'Yes' ELSE 'No' END AS triangle
FROM Triangle;
```

**🧠 Explanation:** Triangle inequality: sum of any two sides must be greater than the third.

---

### Q41. Rearrange Products Table (LC 1795 · Easy) — 🚀 ⭐⭐

**📖 Question in simple words:** The table has columns `store1, store2, store3` with prices. Convert it to rows `(product_id, store, price)`, skipping stores where the price is NULL.

**🎯 Outcome:** One row per (product, store) having a price.

```sql
SELECT product_id, 'store1' AS store, store1 AS price FROM Products WHERE store1 IS NOT NULL
UNION ALL
SELECT product_id, 'store2', store2 FROM Products WHERE store2 IS NOT NULL
UNION ALL
SELECT product_id, 'store3', store3 FROM Products WHERE store3 IS NOT NULL;
```

**🧠 Explanation:** **UNPIVOT / melt** – wide → long. Select each column separately and stack with `UNION ALL` (no dedupe needed, so faster than `UNION`). Normalization of wide tables is a common ETL task.

---

### Q42. Capital Gain/Loss (LC 1393 · Medium) — 🚀 ⭐⭐

**📖 Question in simple words:** A stock is bought and sold many times. Capital gain/loss = total sell price − total buy price per stock.

**🎯 Outcome:** `stock_name, capital_gain_loss`.

```sql
SELECT stock_name,
       SUM(CASE WHEN operation = 'Sell' THEN price ELSE -price END) AS capital_gain_loss
FROM Stocks
GROUP BY stock_name;
```

**🧠 Explanation:** Convert buys to negative numbers and sells to positive, then simply sum.

---

### Q43. Apples & Oranges (LC 1445 · Medium) — 🏢 ⭐

**📖 Question in simple words:** For each day, the difference between the number of apples and oranges sold.

**🎯 Outcome:** `sale_date, diff` sorted by date.

```sql
SELECT sale_date,
       SUM(CASE WHEN fruit = 'apples' THEN sold_num ELSE -sold_num END) AS diff
FROM Sales
GROUP BY sale_date
ORDER BY sale_date;
```

**🧠 Explanation:** Same sign-flip trick as above: apples positive, oranges negative, sum per day.

---

### Q44. Exchange Seats (LC 626 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Swap seat ids of every two consecutive students (1↔2, 3↔4 …). If total students are odd, the last one stays.

**🎯 Outcome:** `id, student` sorted by new id.

```sql
SELECT CASE
         WHEN id % 2 = 1 AND id = (SELECT MAX(id) FROM Seat) THEN id
         WHEN id % 2 = 1 THEN id + 1
         ELSE id - 1
       END AS id,
       student
FROM Seat
ORDER BY id;
```

**🧠 Explanation:** Odd ids move forward (+1), even ids move back (−1). The last odd id has no partner so it stays. Alternative using window: `COALESCE(IF(id%2=1, LEAD(student) OVER (ORDER BY id), LAG(student) OVER (ORDER BY id)), student)`.

---

### Q45. Tree Node (LC 608 · Medium) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** Label every node in a tree as `Root` (no parent), `Inner` (has parent and children) or `Leaf` (has parent but no children).

**🎯 Outcome:** `id, type`.

```sql
SELECT id,
       CASE
         WHEN p_id IS NULL THEN 'Root'
         WHEN id IN (SELECT p_id FROM Tree WHERE p_id IS NOT NULL) THEN 'Inner'
         ELSE 'Leaf'
       END AS type
FROM Tree;
```

**🧠 Explanation:** Order of `CASE` matters (first match wins). A node is `Inner` if its id appears as someone's parent. Filter out NULLs inside `IN (…)` to avoid NULL pitfalls.

---

# PATTERN 5 – Duplicates, DELETE & UPDATE

### Q46. Delete Duplicate Emails (LC 196 · Easy) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Delete duplicate rows in Person, keeping only the row with the **smallest id** for each email.

**🎯 Outcome:** Person table with unique emails.

```sql
DELETE p1
FROM Person p1
JOIN Person p2
  ON p1.email = p2.email
 AND p1.id > p2.id;
```

**🧠 Explanation:** Join the table to itself on same email. When `p1.id > p2.id`, `p1` is a "later" duplicate – delete it. The smallest id never has a smaller partner, so it survives. Alternative: `DELETE FROM Person WHERE id NOT IN (SELECT MIN(id) FROM (SELECT * FROM Person) t GROUP BY email)`; with window functions in other DBs: delete rows where `ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) > 1`.

---

### Q47. Swap Salary (LC 627 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Swap all `'m'` and `'f'` values in `sex` with a **single UPDATE** and no temp table.

**🎯 Outcome:** Updated Salary table.

```sql
UPDATE Salary
SET sex = CASE WHEN sex = 'm' THEN 'f' ELSE 'm' END;
```

**🧠 Explanation:** Within one UPDATE each row is evaluated using its old value, so a CASE can flip it in one go.

---

# PATTERN 6 – Ranking, Top-N & First/Latest per Group (Window Functions)

> **Most important pattern for product-based companies.** Template: `fn() OVER (PARTITION BY group ORDER BY col)` inside a subquery/CTE, then filter on the rank.

### Q48. Second Highest Salary (LC 176 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Return the 2nd highest **distinct** salary. If there is none, return NULL.

**🎯 Outcome:** One column `SecondHighestSalary`.

```sql
SELECT (
    SELECT DISTINCT salary
    FROM Employee
    ORDER BY salary DESC
    LIMIT 1 OFFSET 1
) AS SecondHighestSalary;
```

**🧠 Explanation:** `DISTINCT` + sort descending + skip 1 row. Wrapping it in an outer `SELECT (…)` converts "no rows" into a single NULL – which the problem demands. Alternative: `SELECT MAX(salary) FROM Employee WHERE salary < (SELECT MAX(salary) FROM Employee)`.

---

### Q49. Nth Highest Salary (LC 177 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Write a function that returns the N-th highest distinct salary (NULL if it doesn't exist).

**🎯 Outcome:** A single value.

```sql
CREATE FUNCTION getNthHighestSalary(N INT) RETURNS INT
BEGIN
  RETURN (
    SELECT DISTINCT salary
    FROM (
        SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
        FROM Employee
    ) t
    WHERE rnk = N
  );
END
```

**🧠 Explanation:** `DENSE_RANK` gives equal salaries the same rank with no gaps, so rank N = N-th distinct salary. `DISTINCT` returns a single value even for ties. If no row has rank N the subquery returns NULL. Alternative: `LIMIT 1 OFFSET N-1` (set `SET N = N-1` first because MySQL doesn't allow expressions in `OFFSET`).

---

### Q50. Rank Scores (LC 178 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Rank scores from highest to lowest. Same scores get the same rank and the next rank is **not skipped**.

**🎯 Outcome:** `score, rank`.

```sql
SELECT score,
       DENSE_RANK() OVER (ORDER BY score DESC) AS `rank`
FROM Scores;
```

**🧠 Explanation:** No gaps in ranking → `DENSE_RANK`. (`RANK` would give 1,1,3 and `ROW_NUMBER` would never tie.) `rank` is a reserved word in MySQL 8, hence the backticks.

---

### Q51. Department Highest Salary (LC 184 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** For every department, list the employee(s) with the highest salary (ties included).

**🎯 Outcome:** `Department, Employee, Salary`.

```sql
SELECT d.name AS Department, e.name AS Employee, e.salary AS Salary
FROM Employee e
JOIN Department d ON e.departmentId = d.id
WHERE (e.departmentId, e.salary) IN (
    SELECT departmentId, MAX(salary)
    FROM Employee
    GROUP BY departmentId
);
```

**🧠 Explanation:** The subquery finds each department's max salary. The tuple `IN` keeps all employees matching their own department's max – so ties are naturally included. Window version: `RANK() OVER (PARTITION BY departmentId ORDER BY salary DESC) = 1`.

---

### Q52. Department Top Three Salaries (LC 185 · Hard) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** In each department, list employees who earn one of the **top 3 unique salaries**.

**🎯 Outcome:** `Department, Employee, Salary`.

```sql
SELECT Department, Employee, Salary
FROM (
    SELECT d.name AS Department,
           e.name AS Employee,
           e.salary AS Salary,
           DENSE_RANK() OVER (PARTITION BY e.departmentId ORDER BY e.salary DESC) AS rnk
    FROM Employee e
    JOIN Department d ON e.departmentId = d.id
) t
WHERE rnk <= 3;
```

**🧠 Explanation:** `PARTITION BY departmentId` restarts ranking in each department. `DENSE_RANK` makes sure "top 3 *unique* salaries" is respected (several people can share rank 2). Since window functions can't be used in `WHERE`, we wrap in a subquery. **This is THE top-N-per-group template.**

---

### Q53. Game Play Analysis I (LC 511 · Easy) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** Find the first login date of every player.

**🎯 Outcome:** `player_id, first_login`.

```sql
SELECT player_id, MIN(event_date) AS first_login
FROM Activity
GROUP BY player_id;
```

**🧠 Explanation:** First = earliest = `MIN(date)` per player.

---

### Q54. Product Sales Analysis III (LC 1070 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** For every product, show all sales rows of its **first year** of sale.

**🎯 Outcome:** `product_id, first_year, quantity, price`.

```sql
SELECT product_id, year AS first_year, quantity, price
FROM Sales
WHERE (product_id, year) IN (
    SELECT product_id, MIN(year)
    FROM Sales
    GROUP BY product_id
);
```

**🧠 Explanation:** Find each product's minimum year in the subquery, then pull every row that matches that (product, year) pair. Window alternative: `RANK() OVER (PARTITION BY product_id ORDER BY year) = 1`.

---

### Q55. Immediate Food Delivery II (LC 1174 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Looking at only each customer's **first order**, what percentage were "immediate" (order date = preferred delivery date)?

**🎯 Outcome:** One number `immediate_percentage` (2 decimals).

```sql
SELECT ROUND(AVG(order_date = customer_pref_delivery_date) * 100, 2) AS immediate_percentage
FROM Delivery
WHERE (customer_id, order_date) IN (
    SELECT customer_id, MIN(order_date)
    FROM Delivery
    GROUP BY customer_id
);
```

**🧠 Explanation:** Step 1 – keep only the first order per customer (min date). Step 2 – `AVG(condition)` gives the fraction of immediate orders; ×100 for percentage.

---

### Q56. Product Price at a Given Date (LC 1164 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Prices change on certain dates. What was each product's price on **2019-08-16**? If it had not changed by then, price is 10.

**🎯 Outcome:** `product_id, price` for all products.

```sql
SELECT p.product_id, IFNULL(x.new_price, 10) AS price
FROM (SELECT DISTINCT product_id FROM Products) p
LEFT JOIN (
    SELECT product_id, new_price,
           ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY change_date DESC) AS rn
    FROM Products
    WHERE change_date <= '2019-08-16'
) x ON p.product_id = x.product_id AND x.rn = 1
;
```

**🧠 Explanation:** (1) Keep only changes on/before the date. (2) Rank them newest-first per product; `rn = 1` is the latest applicable price. (3) LEFT JOIN from the full product list so products with no earlier change fall back to the default 10. This is **"point-in-time / SCD Type 2 lookup"** logic used in data warehouses.

---

### Q57. Primary Department for Each Employee (LC 1789 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** An employee may belong to many departments. Pick his primary department: the one flagged `Y`; if he belongs to only one department, that department is primary.

**🎯 Outcome:** `employee_id, department_id`.

```sql
SELECT employee_id, department_id
FROM Employee
WHERE primary_flag = 'Y'
   OR employee_id IN (
        SELECT employee_id
        FROM Employee
        GROUP BY employee_id
        HAVING COUNT(*) = 1
   );
```

**🧠 Explanation:** Either the row is flagged `Y`, or the employee has exactly one row (single department, flag may be `N`).

---

### Q58. Movie Rating (LC 1341 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Return (1) the name of the user who rated the most movies (tie → alphabetically smaller) and (2) the title of the movie with highest average rating in **Feb 2020** (tie → alphabetically smaller). Output both in one column.

**🎯 Outcome:** 2 rows in column `results`.

```sql
(SELECT u.name AS results
 FROM MovieRating r
 JOIN Users u ON r.user_id = u.user_id
 GROUP BY u.user_id, u.name
 ORDER BY COUNT(*) DESC, u.name
 LIMIT 1)
UNION ALL
(SELECT m.title
 FROM MovieRating r
 JOIN Movies m ON r.movie_id = m.movie_id
 WHERE r.created_at BETWEEN '2020-02-01' AND '2020-02-29'
 GROUP BY m.movie_id, m.title
 ORDER BY AVG(r.rating) DESC, m.title
 LIMIT 1);
```

**🧠 Explanation:** Two independent "top-1" queries, each with a tie-breaker in `ORDER BY`, are stacked using `UNION ALL`. Parentheses are needed so each query can have its own `ORDER BY … LIMIT`.

---

# PATTERN 7 – Consecutive Rows / Gaps & Islands

> Key trick: **`value − ROW_NUMBER()`** is constant for a run of consecutive values → use it as group key. Or use `LAG/LEAD`.

### Q59. Consecutive Numbers (LC 180 · Medium) — 🏢🚀 ⭐⭐⭐

**📖 Question in simple words:** Find numbers that appear **at least three times in a row** in the log.

**🎯 Outcome:** `ConsecutiveNums`.

```sql
SELECT DISTINCT num AS ConsecutiveNums
FROM (
    SELECT num,
           LAG(num, 1) OVER (ORDER BY id) AS p1,
           LAG(num, 2) OVER (ORDER BY id) AS p2
    FROM Logs
) t
WHERE num = p1 AND num = p2;
```

**🧠 Explanation:** `LAG(num,1)` and `LAG(num,2)` bring the previous two values onto the current row. If all three are equal, it's a streak of 3. `DISTINCT` removes repeated reports of the same number. Self-join alternative: `l1.id = l2.id-1 = l3.id-2` and equal `num`s.

---

### Q60. Human Traffic of Stadium (LC 601 · Hard) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Display rows from days where **3 or more consecutive ids** each had 100+ people.

**🎯 Outcome:** `id, visit_date, people` sorted by `visit_date`.

```sql
WITH t AS (
    SELECT *, id - ROW_NUMBER() OVER (ORDER BY id) AS grp
    FROM Stadium
    WHERE people >= 100
)
SELECT id, visit_date, people
FROM t
WHERE grp IN (SELECT grp FROM t GROUP BY grp HAVING COUNT(*) >= 3)
ORDER BY visit_date;
```

**🧠 Explanation:** **Gaps & Islands.** After filtering to 100+ days, subtract the row number from `id`. For consecutive ids (e.g. 5,6,7,8 with rn 1,2,3,4) the difference is constant (4) → same group. Groups with ≥ 3 rows are the answer.

---

### Q61. Consecutive Available Seats (LC 603 · Easy 🔒) — 🏢 ⭐⭐

**📖 Question in simple words:** Cinema seats are free (1) or taken (0). Find all seats that are free **and** next to at least one other free seat.

**🎯 Outcome:** `seat_id` sorted.

```sql
SELECT DISTINCT c1.seat_id
FROM Cinema c1
JOIN Cinema c2
  ON ABS(c1.seat_id - c2.seat_id) = 1
 AND c1.free = 1
 AND c2.free = 1
ORDER BY c1.seat_id;
```

**🧠 Explanation:** Self join each seat with its neighbours (`|diff| = 1`) and require both free. `DISTINCT` because a seat with two free neighbours matches twice.

---

### Q62. Find the Start and End Number of Continuous Ranges (LC 1285 · Medium 🔒) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Given a list of log ids with gaps (e.g. 1,2,3,7,8,10), print each continuous range as `start_id, end_id` (1–3, 7–8, 10–10).

**🎯 Outcome:** `start_id, end_id`.

```sql
SELECT MIN(log_id) AS start_id, MAX(log_id) AS end_id
FROM (
    SELECT log_id, log_id - ROW_NUMBER() OVER (ORDER BY log_id) AS grp
    FROM Logs
) t
GROUP BY grp
ORDER BY start_id;
```

**🧠 Explanation:** Pure Gaps & Islands. Each island of consecutive numbers has a constant `log_id - row_number`. Min/max inside each group gives the boundaries. Used in real life to find **missing-data windows** and **session detection**.

---

# PATTERN 8 – Running Total & Moving Average

> Syntax: `SUM(x) OVER (PARTITION BY … ORDER BY … ROWS BETWEEN … AND …)`. Remember: with `ORDER BY` and no frame, the default frame is `RANGE UNBOUNDED PRECEDING → CURRENT ROW` (ties are summed together).

### Q63. Restaurant Growth (LC 1321 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** For each day (starting from the 7th day of data), compute the **7-day moving sum** of amount and its average (round 2).

**🎯 Outcome:** `visited_on, amount, average_amount`.

```sql
SELECT visited_on, amount, ROUND(amount / 7, 2) AS average_amount
FROM (
    SELECT visited_on,
           SUM(SUM(amount)) OVER (
               ORDER BY visited_on
               RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW
           ) AS amount
    FROM Customer
    GROUP BY visited_on
) t
WHERE visited_on >= (SELECT MIN(visited_on) + INTERVAL 6 DAY FROM Customer);
```

**🧠 Explanation:** (1) `GROUP BY visited_on` first collapses several customers per day to one daily total. (2) `SUM(SUM(amount)) OVER (...)` = window sum over the daily totals of the last 7 days (current + 6 preceding). (3) The first 6 days don't have a full window, so we drop dates earlier than first date + 6 days. If days are guaranteed continuous you can use `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`.

---

### Q64. Last Person to Fit in the Bus (LC 1204 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** People board a bus in `turn` order. The bus carries max 1000 kg. Who is the last person that can board without exceeding the limit?

**🎯 Outcome:** `person_name` (one row).

```sql
SELECT person_name
FROM (
    SELECT person_name,
           SUM(weight) OVER (ORDER BY turn) AS total_weight
    FROM Queue
) t
WHERE total_weight <= 1000
ORDER BY total_weight DESC
LIMIT 1;
```

**🧠 Explanation:** Running total of weight by turn. Keep those whose cumulative weight ≤ 1000; the one with the highest cumulative weight is the last who fits.

---

### Q65. Game Play Analysis III (LC 534 · Medium 🔒) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** For each player and each date, show how many games they have played **so far** (cumulative).

**🎯 Outcome:** `player_id, event_date, games_played_so_far`.

```sql
SELECT player_id, event_date,
       SUM(games_played) OVER (PARTITION BY player_id ORDER BY event_date) AS games_played_so_far
FROM Activity;
```

**🧠 Explanation:** `PARTITION BY player_id` restarts the running sum for each player; `ORDER BY event_date` accumulates in time order.

---

### Q66. Running Total for Different Genders (LC 1308 · Medium 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** For each gender, show the cumulative score of the players day by day.

**🎯 Outcome:** `gender, day, total` sorted by gender, day.

```sql
SELECT gender, day,
       SUM(score_points) OVER (PARTITION BY gender ORDER BY day) AS total
FROM Scores
ORDER BY gender, day;
```

**🧠 Explanation:** Same template: partition by group, order by time, running `SUM`.

---

# PATTERN 9 – Date-based Analysis, Retention & Rates

> Product companies love **DAU / retention / cancellation-rate** questions.

### Q67. Game Play Analysis IV (LC 550 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** What fraction of players logged in again **on the day right after their first login**? (Round 2 decimals) – this is **Day-1 retention**.

**🎯 Outcome:** One value `fraction`.

```sql
WITH f AS (
    SELECT player_id, MIN(event_date) AS first_day
    FROM Activity
    GROUP BY player_id
)
SELECT ROUND(COUNT(a.player_id) / COUNT(f.player_id), 2) AS fraction
FROM f
LEFT JOIN Activity a
       ON a.player_id = f.player_id
      AND a.event_date = DATE_ADD(f.first_day, INTERVAL 1 DAY);
```

**🧠 Explanation:** CTE finds each player's first login. LEFT JOIN to activity on exactly day+1. `COUNT(a.player_id)` = players who returned, `COUNT(f.player_id)` = all players.

---

### Q68. Trips and Users (LC 262 · Hard) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** For each day between 2013-10-01 and 2013-10-03, compute the **cancellation rate** = cancelled trips / all trips, counting only trips where **both the client and the driver are not banned**.

**🎯 Outcome:** `Day, Cancellation Rate` (2 decimals).

```sql
SELECT t.request_at AS Day,
       ROUND(AVG(t.status <> 'completed'), 2) AS `Cancellation Rate`
FROM Trips t
JOIN Users c ON t.client_id = c.users_id AND c.banned = 'No'
JOIN Users d ON t.driver_id = d.users_id AND d.banned = 'No'
WHERE t.request_at BETWEEN '2013-10-01' AND '2013-10-03'
GROUP BY t.request_at;
```

**🧠 Explanation:** Join the Users table twice – once as client, once as driver – and keep only unbanned. Statuses like `cancelled_by_driver` / `cancelled_by_client` are both "not completed", so `AVG(status <> 'completed')` gives the cancellation share.

---

### Q69. User Activity for the Past 30 Days II (LC 1142 · Easy 🔒) — 🚀 ⭐

**📖 Question in simple words:** Average number of sessions per user in the 30 days ending 2019-07-27 (users with at least one session).

**🎯 Outcome:** `average_sessions_per_user` (2 decimals; 0 if none).

```sql
SELECT IFNULL(ROUND(COUNT(DISTINCT session_id) / COUNT(DISTINCT user_id), 2), 0) AS average_sessions_per_user
FROM Activity
WHERE activity_date BETWEEN '2019-06-28' AND '2019-07-27';
```

**🧠 Explanation:** Distinct sessions ÷ distinct users in the window. `IFNULL` handles divide-by-zero → NULL.

---

### Q70. Reported Posts II (LC 1132 · Medium 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** Among posts reported as spam each day, what percentage got removed later? Then average this daily percentage across days.

**🎯 Outcome:** `average_daily_percent` (2 decimals).

```sql
SELECT ROUND(AVG(daily) * 100, 2) AS average_daily_percent
FROM (
    SELECT a.action_date,
           COUNT(DISTINCT r.post_id) / COUNT(DISTINCT a.post_id) AS daily
    FROM Actions a
    LEFT JOIN Removals r ON a.post_id = r.post_id
    WHERE a.action = 'report' AND a.extra = 'spam'
    GROUP BY a.action_date
) t;
```

**🧠 Explanation:** For each day: distinct removed posts / distinct reported posts. LEFT JOIN leaves `r.post_id` NULL for posts not removed (ignored by `COUNT`). The outer query averages the per-day ratio – note it's the **average of ratios**, not a global ratio.

---

### Q71. The Latest Login in 2020 (LC 1890 · Easy) — 🏢🚀 ⭐

**📖 Question in simple words:** For each user who logged in during 2020, show his most recent login in 2020.

**🎯 Outcome:** `user_id, last_stamp`.

```sql
SELECT user_id, MAX(time_stamp) AS last_stamp
FROM Logins
WHERE time_stamp >= '2020-01-01' AND time_stamp < '2021-01-01'
GROUP BY user_id;
```

**🧠 Explanation:** Use a half-open date range instead of `YEAR(time_stamp)=2020` – it is **sargable** (can use an index). `MAX` gives the latest.

---

### Q72. Convert Date Format (LC 1853 · Easy 🔒) — 🏢 ⭐

**📖 Question in simple words:** Convert date `2022-04-12` into text like `Tuesday, April 12, 2022`.

**🎯 Outcome:** `day` as formatted string.

```sql
SELECT DATE_FORMAT(day, '%W, %M %e, %Y') AS day
FROM Days;
```

**🧠 Explanation:** `%W` weekday name, `%M` month name, `%e` day without leading zero, `%Y` 4-digit year.

---

### Q73. Daily Leads and Partners (LC 1693 · Easy 🔒) — 🏢 ⭐

**📖 Question in simple words:** For each date and product (`make_name`), count the unique leads and unique partners.

**🎯 Outcome:** `date_id, make_name, unique_leads, unique_partners`.

```sql
SELECT date_id, make_name,
       COUNT(DISTINCT lead_id)    AS unique_leads,
       COUNT(DISTINCT partner_id) AS unique_partners
FROM DailySales
GROUP BY date_id, make_name;
```

**🧠 Explanation:** Group on date and product; `COUNT(DISTINCT …)` removes repeated ids.

---

# PATTERN 10 – Hierarchy / Recursive CTE

### Q74. The Number of Employees Which Report to Each Employee (LC 1731 · Easy) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** For every manager, count how many people report directly to him and the **average age** of those reports (rounded to integer).

**🎯 Outcome:** `employee_id, name, reports_count, average_age` sorted by `employee_id`.

```sql
SELECT m.employee_id, m.name,
       COUNT(*)          AS reports_count,
       ROUND(AVG(e.age)) AS average_age
FROM Employees e
JOIN Employees m ON e.reports_to = m.employee_id
GROUP BY m.employee_id, m.name
ORDER BY m.employee_id;
```

**🧠 Explanation:** Self-join: `e` = subordinate, `m` = manager. Group by manager and aggregate the subordinates. Inner join ensures only real managers appear.

---

### Q75. All People Report to the Given Manager (LC 1270 · Medium 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** Employee 1 is the head. List everyone who reports to him **directly or indirectly** (any depth), excluding the head himself.

**🎯 Outcome:** `employee_id`.

```sql
WITH RECURSIVE r AS (
    SELECT employee_id
    FROM Employees
    WHERE manager_id = 1 AND employee_id <> 1      -- anchor: direct reports
    UNION ALL
    SELECT e.employee_id
    FROM Employees e
    JOIN r ON e.manager_id = r.employee_id         -- recursive: reports of reports
)
SELECT employee_id FROM r;
```

**🧠 Explanation:** A **recursive CTE** has an *anchor* (starting rows) and a *recursive member* that keeps joining the previous result until no new rows appear. We exclude `employee_id <> 1` because the head's manager_id may be himself (infinite loop). Without recursion you can chain 3 self-joins for a fixed depth of 3.

---

# PATTERN 11 – Subqueries, EXISTS / NOT IN & Set Logic

### Q76. Investments in 2016 (LC 585 · Medium) — 🚀 ⭐⭐

**📖 Question in simple words:** Sum `tiv_2016` for policyholders who (a) have the same `tiv_2015` as at least one other policyholder and (b) are **not** in the same city (lat, lon) as any other policyholder. Round 2 decimals.

**🎯 Outcome:** One number `tiv_2016`.

```sql
SELECT ROUND(SUM(tiv_2016), 2) AS tiv_2016
FROM Insurance
WHERE tiv_2015 IN (
        SELECT tiv_2015 FROM Insurance GROUP BY tiv_2015 HAVING COUNT(*) > 1)
  AND (lat, lon) IN (
        SELECT lat, lon FROM Insurance GROUP BY lat, lon HAVING COUNT(*) = 1);
```

**🧠 Explanation:** Two subqueries produce the allowed `tiv_2015` values (duplicated) and the allowed locations (unique). Combine both filters, then sum. Window version: `COUNT(*) OVER (PARTITION BY tiv_2015) > 1 AND COUNT(*) OVER (PARTITION BY lat, lon) = 1`.

---

### Q77. Bank Account Summary II (LC 1587 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Report users whose total balance (sum of transaction amounts) is **more than 10000**.

**🎯 Outcome:** `name, balance`.

```sql
SELECT u.name, SUM(t.amount) AS balance
FROM Users u
JOIN Transactions t ON u.account = t.account
GROUP BY u.account, u.name
HAVING SUM(t.amount) > 10000;
```

**🧠 Explanation:** Join, group by account, filter groups with `HAVING`.

---

### Q78. Market Analysis I (LC 1158 · Medium) — 🚀 ⭐⭐

**📖 Question in simple words:** For every user show their join date and the number of orders they placed **as a buyer in 2019** (0 if none).

**🎯 Outcome:** `buyer_id, join_date, orders_in_2019`.

```sql
SELECT u.user_id AS buyer_id, u.join_date,
       COUNT(o.order_id) AS orders_in_2019
FROM Users u
LEFT JOIN Orders o
       ON u.user_id = o.buyer_id
      AND o.order_date BETWEEN '2019-01-01' AND '2019-12-31'
GROUP BY u.user_id, u.join_date;
```

**🧠 Explanation:** The 2019 condition is in `ON` (not `WHERE`) so users with no 2019 orders still appear with count 0. This is the classic "filter in ON vs WHERE" question.

---

### Q79. Customers Who Bought Products A and B but Not C (LC 1398 · Medium 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** Find customers who purchased product A **and** B but **never** C.

**🎯 Outcome:** `customer_id, customer_name`.

```sql
SELECT c.customer_id, c.customer_name
FROM Customers c
JOIN Orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING SUM(o.product_name = 'A') > 0
   AND SUM(o.product_name = 'B') > 0
   AND SUM(o.product_name = 'C') = 0;
```

**🧠 Explanation:** One pass with conditional sums in `HAVING`: >0 means "bought", =0 means "never bought". Much cleaner than three subqueries.

---

### Q80. List the Products Ordered in a Period (LC 1327 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Show products that had at least 100 units ordered in February 2020, along with the total units.

**🎯 Outcome:** `product_name, unit`.

```sql
SELECT p.product_name, SUM(o.unit) AS unit
FROM Products p
JOIN Orders o ON p.product_id = o.product_id
WHERE o.order_date >= '2020-02-01' AND o.order_date < '2020-03-01'
GROUP BY p.product_id, p.product_name
HAVING SUM(o.unit) >= 100;
```

**🧠 Explanation:** `WHERE` limits to Feb rows, `GROUP BY` sums per product, `HAVING` keeps ≥ 100.

---

### Q81. Top Travellers (LC 1407 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** Show total distance travelled by each user (0 if no rides), sorted by distance desc then name asc.

**🎯 Outcome:** `name, travelled_distance`.

```sql
SELECT u.name, IFNULL(SUM(r.distance), 0) AS travelled_distance
FROM Users u
LEFT JOIN Rides r ON u.id = r.user_id
GROUP BY u.id, u.name
ORDER BY travelled_distance DESC, u.name ASC;
```

**🧠 Explanation:** LEFT JOIN preserves users without rides; `IFNULL` turns the NULL sum into 0. Group by `u.id` too (names can repeat).

---

### Q82. Number of Calls Between Two Persons (LC 1699 · Medium 🔒) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Calls are stored directed (from→to). Treat (1,2) and (2,1) as the same pair; give number of calls and total duration for each pair.

**🎯 Outcome:** `person1, person2, call_count, total_duration` with `person1 < person2`.

```sql
SELECT LEAST(from_id, to_id)    AS person1,
       GREATEST(from_id, to_id) AS person2,
       COUNT(*)                 AS call_count,
       SUM(duration)            AS total_duration
FROM Calls
GROUP BY person1, person2;
```

**🧠 Explanation:** **Normalizing an undirected pair**: `LEAST/GREATEST` always puts the smaller id first so A→B and B→A fall in the same group. Useful for friend/edge tables in graph-like data.

---

### Q83. Friend Requests II: Who Has the Most Friends (LC 602 · Medium) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** A friendship has a requester and an accepter; it counts as a friend for **both**. Find the person with the most friends and the count.

**🎯 Outcome:** `id, num`.

```sql
SELECT id, COUNT(*) AS num
FROM (
    SELECT requester_id AS id FROM RequestAccepted
    UNION ALL
    SELECT accepter_id  AS id FROM RequestAccepted
) t
GROUP BY id
ORDER BY num DESC
LIMIT 1;
```

**🧠 Explanation:** Stack both columns into one id column with `UNION ALL` (we need duplicates!). Each appearance = one friend. Group, count, top 1.

---

### Q84. Active Businesses (LC 1126 · Medium 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** A business is active if it has **more than one event type** whose occurrences are higher than the **average occurrences of that event type across all businesses**.

**🎯 Outcome:** `business_id`.

```sql
SELECT business_id
FROM (
    SELECT business_id,
           occurrences > AVG(occurrences) OVER (PARTITION BY event_type) AS flag
    FROM Events
) t
GROUP BY business_id
HAVING SUM(flag) > 1;
```

**🧠 Explanation:** The window `AVG … PARTITION BY event_type` attaches the global average of that event type to each row without collapsing rows. `flag` = 1 when the row beats the average. Count flags per business and keep those > 1.

---

# PATTERN 12 – Median, Percentile, Recursive & Advanced Windows

### Q85. Median Employee Salary (LC 569 · Hard 🔒) — 🚀 ⭐⭐⭐

**📖 Question in simple words:** Return the median salary row(s) for each company. For an even number of employees return the two middle rows.

**🎯 Outcome:** `id, company, salary`.

```sql
SELECT id, company, salary
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY company ORDER BY salary, id) AS rn,
           COUNT(*)     OVER (PARTITION BY company)                     AS cnt
    FROM Employee
) t
WHERE rn BETWEEN cnt / 2 AND cnt / 2 + 1;
```

**🧠 Explanation:** Number rows in salary order and get the total count per company. Odd count 5 → `2.5 ≤ rn ≤ 3.5` → row 3. Even count 4 → `2 ≤ rn ≤ 3` → rows 2 and 3. (PostgreSQL has `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)`.)

---

### Q86. Find Median Given Frequency of Numbers (LC 571 · Hard 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** The table stores `num` and how many times it occurs (frequency). Imagine the fully expanded list and find its median (round to 1 decimal).

**🎯 Outcome:** `median`.

```sql
SELECT ROUND(AVG(num), 1) AS median
FROM (
    SELECT num, frequency,
           SUM(frequency) OVER (ORDER BY num) AS cum,
           SUM(frequency) OVER ()             AS total
    FROM Numbers
) t
WHERE cum >= total / 2
  AND cum - frequency <= total / 2;
```

**🧠 Explanation:** Without expanding the data, use cumulative frequency. A number covers positions `(cum − frequency, cum]`. The median position(s) is `total/2`; any number whose interval contains it qualifies. For an even total the two middle numbers may differ, hence `AVG`.

---

### Q87. Compute the Rank as a Percentage (LC 2346 · Medium 🔒) — 🚀 ⭐

**📖 Question in simple words:** Within each department compute `(rank − 1) × 100 / (students_in_dept − 1)` where rank is by mark descending (ties share a rank). Round 2 decimals.

**🎯 Outcome:** `student_id, department_id, percentage`.

```sql
SELECT student_id, department_id,
       ROUND(PERCENT_RANK() OVER (PARTITION BY department_id ORDER BY mark DESC) * 100, 2) AS percentage
FROM Students;
```

**🧠 Explanation:** `PERCENT_RANK()` is defined as `(rank − 1) / (rows − 1)` – exactly the formula, and it returns 0 for a single-row partition. Knowing `PERCENT_RANK`, `CUME_DIST` and `NTILE` is a bonus in interviews.

---

### Q88. Get the Second Most Recent Activity (LC 1369 · Hard 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** For each user show their 2nd most recent activity; if the user has only one activity, show that one.

**🎯 Outcome:** `username, activity, startDate, endDate`.

```sql
SELECT username, activity, startDate, endDate
FROM (
    SELECT *,
           RANK()  OVER (PARTITION BY username ORDER BY startDate DESC) AS rk,
           COUNT(*) OVER (PARTITION BY username)                        AS cnt
    FROM UserActivity
) t
WHERE rk = 2 OR cnt = 1;
```

**🧠 Explanation:** Rank activities newest-first per user and also count activities per user. Pick rank 2, or the single row when `cnt = 1`.

---

### Q89. Maximum Transaction Each Day (LC 1831 · Medium 🔒) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** For each day, list the ids of the transaction(s) with the highest amount (ties included).

**🎯 Outcome:** `transaction_id` sorted ascending.

```sql
SELECT transaction_id
FROM (
    SELECT transaction_id,
           RANK() OVER (PARTITION BY DATE(day) ORDER BY amount DESC) AS rk
    FROM Transactions
) t
WHERE rk = 1
ORDER BY transaction_id;
```

**🧠 Explanation:** `day` is a datetime, so partition by `DATE(day)`. `RANK` (not `ROW_NUMBER`) keeps ties at rank 1.

---

### Q90. Find the Subtasks That Did Not Execute (LC 1767 · Hard 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** Each task has N subtasks numbered 1..N. Some executed. List the subtasks that **were not executed**.

**🎯 Outcome:** `task_id, subtask_id`.

```sql
WITH RECURSIVE s AS (
    SELECT task_id, subtasks_count AS subtask_id FROM Tasks
    UNION ALL
    SELECT task_id, subtask_id - 1 FROM s WHERE subtask_id > 1
)
SELECT s.task_id, s.subtask_id
FROM s
LEFT JOIN Executed e
       ON s.task_id = e.task_id AND s.subtask_id = e.subtask_id
WHERE e.subtask_id IS NULL;
```

**🧠 Explanation:** Use a recursive CTE as a **number generator** to expand each task into N rows (N, N−1, …, 1). Then anti-join with the executed list. "Generate a sequence / calendar and left join facts" is a key data-engineering skill (e.g. building a date dimension).

---

# PATTERN 13 – Mixed Practice (Strings, Buckets, Windows, Pivot)

### Q91. Group Sold Products By The Date (LC 1484 · Easy) — 🏢 ⭐⭐

**📖 Question in simple words:** For each sell date, count the **different products** sold and list their names alphabetically, comma-separated.

**🎯 Outcome:** `sell_date, num_sold, products`.

```sql
SELECT sell_date,
       COUNT(DISTINCT product) AS num_sold,
       GROUP_CONCAT(DISTINCT product ORDER BY product SEPARATOR ',') AS products
FROM Activities
GROUP BY sell_date
ORDER BY sell_date;
```

**🧠 Explanation:** `GROUP_CONCAT` is MySQL's string aggregation (`STRING_AGG` in PostgreSQL/SQL Server, `LISTAGG` in Oracle/Snowflake). `DISTINCT` and `ORDER BY` inside it dedupe and sort.

---

### Q92. Number of Comments per Post (LC 1241 · Easy 🔒) — 🏢 ⭐

**📖 Question in simple words:** In one table, posts have `parent_id = NULL` and comments point to their post via `parent_id`. Count unique comments per post (the table may contain duplicate rows).

**🎯 Outcome:** `post_id, number_of_comments` sorted by `post_id`.

```sql
SELECT s1.sub_id AS post_id,
       COUNT(DISTINCT s2.sub_id) AS number_of_comments
FROM Submissions s1
LEFT JOIN Submissions s2 ON s1.sub_id = s2.parent_id
WHERE s1.parent_id IS NULL
GROUP BY s1.sub_id
ORDER BY post_id;
```

**🧠 Explanation:** Self-join post ↔ its comments. `LEFT JOIN` keeps posts with zero comments; `COUNT(DISTINCT …)` ignores duplicate rows.

---

### Q93. Find the Team Size (LC 1303 · Easy 🔒) — 🏢 ⭐⭐

**📖 Question in simple words:** For every employee show the size of the team he belongs to.

**🎯 Outcome:** `employee_id, team_size`.

```sql
SELECT employee_id,
       COUNT(*) OVER (PARTITION BY team_id) AS team_size
FROM Employees;
```

**🧠 Explanation:** A window `COUNT` attaches the group count to **every** row – no need to join back an aggregate subquery. Key difference between `GROUP BY` (collapses rows) and window functions (keeps rows).

---

### Q94. Find the Quiet Students in All Exams (LC 1412 · Hard 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** A "quiet" student took at least one exam and **never** scored the highest or lowest in any exam.

**🎯 Outcome:** `student_id, student_name` sorted by id.

```sql
WITH r AS (
    SELECT student_id,
           RANK() OVER (PARTITION BY exam_id ORDER BY score DESC) AS hi,
           RANK() OVER (PARTITION BY exam_id ORDER BY score ASC)  AS lo
    FROM Exam
)
SELECT s.student_id, s.student_name
FROM Student s
JOIN r ON s.student_id = r.student_id
GROUP BY s.student_id, s.student_name
HAVING MIN(r.hi) > 1 AND MIN(r.lo) > 1
ORDER BY s.student_id;
```

**🧠 Explanation:** Rank each student in each exam from both ends. If their *best* top-rank is still > 1 and *best* bottom-rank is > 1, they were never highest or lowest. Inner join ensures only students who took exams.

---

### Q95. Create a Session Bar Chart (LC 1435 · Medium 🔒) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** Session durations are in seconds. Put them into bins `[0-5>`, `[5-10>`, `[10-15>`, `15 or more` (minutes) and count sessions in each (0 if empty).

**🎯 Outcome:** 4 rows `bin, total`.

```sql
SELECT '[0-5>' AS bin,
       COUNT(CASE WHEN duration < 300 THEN 1 END) AS total
FROM Sessions
UNION ALL
SELECT '[5-10>', COUNT(CASE WHEN duration >= 300 AND duration < 600 THEN 1 END) FROM Sessions
UNION ALL
SELECT '[10-15>', COUNT(CASE WHEN duration >= 600 AND duration < 900 THEN 1 END) FROM Sessions
UNION ALL
SELECT '15 or more', COUNT(CASE WHEN duration >= 900 THEN 1 END) FROM Sessions;
```

**🧠 Explanation:** Histogram bucketing. Convert minutes to seconds (5 min = 300 s). Separate SELECTs guarantee that empty bins still show 0.

---

### Q96. Calculate Salaries (LC 1468 · Medium 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** Tax depends on the **highest salary of the company**: <1000 → 0%, 1000–10000 → 24%, >10000 → 49%. Compute each employee's salary after tax (rounded).

**🎯 Outcome:** `company_id, employee_id, employee_name, salary`.

```sql
SELECT company_id, employee_id, employee_name,
       ROUND(salary * (1 - CASE
                             WHEN MAX(salary) OVER (PARTITION BY company_id) < 1000 THEN 0
                             WHEN MAX(salary) OVER (PARTITION BY company_id) <= 10000 THEN 0.24
                             ELSE 0.49
                           END)) AS salary
FROM Salaries;
```

**🧠 Explanation:** Use `MAX(salary) OVER (PARTITION BY company_id)` to see the company's top salary on each row, then pick the tax rate via `CASE`.

---

### Q97. Product's Price for Each Store (LC 1777 · Easy 🔒) — 🏢 ⭐⭐

**📖 Question in simple words:** Turn rows `(product, store, price)` into columns `store1, store2, store3` – one row per product (NULL if not sold in that store).

**🎯 Outcome:** `product_id, store1, store2, store3`.

```sql
SELECT product_id,
       SUM(CASE WHEN store = 'store1' THEN price END) AS store1,
       SUM(CASE WHEN store = 'store2' THEN price END) AS store2,
       SUM(CASE WHEN store = 'store3' THEN price END) AS store3
FROM Products
GROUP BY product_id;
```

**🧠 Explanation:** Classic pivot (long → wide) with conditional aggregation. `MAX` would work equally well since each (product, store) has one price.

---

### Q98. Find Customers With Positive Revenue this Year (LC 1821 · Easy 🔒) — 🏢 ⭐

**📖 Question in simple words:** Return customers who had positive revenue in **2021**.

**🎯 Outcome:** `customer_id`.

```sql
SELECT customer_id
FROM Customers
WHERE year = 2021 AND revenue > 0;
```

**🧠 Explanation:** Simple filter. Revenue `NULL` or ≤ 0 is excluded automatically.

---

### Q99. Group Employees of the Same Salary (LC 1875 · Medium 🔒) — 🚀 ⭐⭐

**📖 Question in simple words:** Teams are made of employees with the **same salary**; a team must have at least 2 members. Give each team an id by ascending salary order (smallest salary → team 1).

**🎯 Outcome:** `employee_id, name, salary, team_id` sorted by team_id, employee_id.

```sql
SELECT employee_id, name, salary,
       DENSE_RANK() OVER (ORDER BY salary) AS team_id
FROM Employees
WHERE salary IN (
    SELECT salary FROM Employees GROUP BY salary HAVING COUNT(*) > 1
)
ORDER BY team_id, employee_id;
```

**🧠 Explanation:** First filter to salaries shared by 2+ people, then `DENSE_RANK` over the remaining salaries yields 1,2,3… without gaps. Order matters: the filter runs *before* the window function, so ranks are computed only on valid teams.

---

### Q100. Weather Type in Each Country (LC 1294 · Easy 🔒) — 🏢🚀 ⭐⭐

**📖 Question in simple words:** For **November 2019**, calculate each country's average `weather_state`: ≤ 15 → `Cold`, ≥ 25 → `Hot`, otherwise `Warm`.

**🎯 Outcome:** `country_name, weather_type`.

```sql
SELECT c.country_name,
       CASE WHEN AVG(w.weather_state) <= 15 THEN 'Cold'
            WHEN AVG(w.weather_state) >= 25 THEN 'Hot'
            ELSE 'Warm' END AS weather_type
FROM Countries c
JOIN Weather w ON c.country_id = w.country_id
WHERE w.day BETWEEN '2019-11-01' AND '2019-11-30'
GROUP BY c.country_id, c.country_name;
```

**🧠 Explanation:** Aggregate first (`AVG`), then bucket the aggregate with `CASE`. You can use aggregate functions inside `CASE` in the `SELECT` list when grouped.

---

# 🎓 Final Section – How to Prepare for Data Engineering SQL Rounds

## Pattern → When to use (quick recall)

| If the question says… | Use this |
|---|---|
| "Nth highest / top N per group" | `DENSE_RANK() / ROW_NUMBER() OVER (PARTITION BY …)` |
| "Never / not present / missing" | `LEFT JOIN … IS NULL`, `NOT EXISTS` |
| "Duplicates" | `GROUP BY … HAVING COUNT(*) > 1`, `ROW_NUMBER()` |
| "Consecutive" | `LAG/LEAD`, `id − ROW_NUMBER()` |
| "Running total / moving average" | `SUM() OVER (ORDER BY … ROWS BETWEEN …)` |
| "Compare with previous row" | `LAG()` / self join on date |
| "Percentage / rate / ratio" | `AVG(condition)` or `SUM(CASE…)/COUNT(*)` |
| "Rows → columns" | `SUM(CASE WHEN …)` + `GROUP BY` |
| "Columns → rows" | `UNION ALL` |
| "All combinations, including zero" | `CROSS JOIN` + `LEFT JOIN` |
| "Hierarchy / any depth" | Recursive CTE |
| "Latest / first record per key" | `ROW_NUMBER() … = 1` |
| "Retention" | First-activity CTE + self-join on `date + 1` |

## Service-based vs Product-based – what to expect

| | Service-based (TCS / Infosys / Wipro / Cognizant / Capgemini) | Product-based (Amazon / Google / Uber / Flipkart / Walmart) |
|---|---|---|
| Level | Easy–Medium | Medium–Hard |
| Focus | Joins, GROUP BY/HAVING, subqueries, 2nd/Nth highest, duplicates, CASE, string/date functions | Window functions, gaps & islands, moving averages, retention, top-N per group, median, recursive CTE, query optimisation |
| Extra | Basic DDL/DML, constraints, normalization, views, stored procedures | Indexing, execution plans, partitioning, data modelling (star schema), SCD, performance on large data |

## Data Engineering follow-ups interviewers add on top

1. **Optimization:** "How would you speed this up on a 1-billion-row table?" → filter early, avoid functions on indexed columns (sargable), partition pruning, correct indexes/clustering, avoid `SELECT *`, use `EXISTS` over `IN` for big subqueries.
2. **`UNION` vs `UNION ALL`**, `RANK` vs `DENSE_RANK` vs `ROW_NUMBER`, `WHERE` vs `HAVING`, `DELETE` vs `TRUNCATE` vs `DROP`, CTE vs temp table vs view.
3. **Data quality:** find duplicates, nulls, orphan records, late-arriving data.
4. **Modelling:** star vs snowflake schema, SCD Type 1/2/3 (Q56 touches SCD2 logic), fact vs dimension tables.
5. **Incremental loads:** `MERGE`/upsert, watermark columns, idempotent pipelines.

## Suggested 4-week plan

| Week | Focus | Questions |
|---|---|---|
| 1 | Patterns 1 – 3 | Q1 – Q37 |
| 2 | Patterns 4 – 6 | Q38 – Q58 (the top-N template!) |
| 3 | Patterns 7 – 9 | Q59 – Q73 |
| 4 | Patterns 10 – 13 + revise Top 30 | Q74 – Q100 |

> ✅ **Tip:** After solving each question, rewrite it with a **different technique** (e.g. subquery → window function, join → `EXISTS`). That is what separates a good answer from a great one in interviews.

**Good luck with your interviews! 🚀**
