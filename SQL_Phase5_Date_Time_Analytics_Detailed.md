# SQL Phase 5: Date, Time and Analytics Patterns (Detailed Guide)

> This is where SQL turns into **real business analytics**: monthly reports, growth, retention, cohorts, funnels, DAU/MAU, overlapping bookings, churn, A/B tests, market basket, medians and pivots.
> Covers **Module 11 (Time-Based Analytics, P34 to P44)** and **Module 12 (Business and Product Metrics, P45 to P52)**.
> Every topic has: easy explanation, input table, query, output table, and step-by-step reasoning.

---

## Table of Contents

- **Part 0:** How to approach an analytics question + dialect conventions
- **Part 1:** Sample data (create once)
- **Part 2:** Module 11: Time-based analytics (date toolkit + P34 to P44)
- **Part 3:** Module 12: Business and product metrics (P45 to P52)
- **Part 4:** Wrap-up (metric crib sheet, mistakes, cheat sheet, practice, self-test, next step)

> **Prerequisites:** Phase 1 (functions, `CASE`, NULLs), Phase 3 (CTEs, recursive calendar) and Phase 4 (window functions). Basic `GROUP BY` and `JOIN` are used throughout.

---

# PART 0: Approach and Conventions

## 0.1 The 5-Step Framework for Any Analytics Question

Interviewers care more about **how you define things** than about syntax. Before writing SQL:

| Step | Ask yourself | Example (retention) |
|---|---|---|
| 1. **Grain** | One row per what? | one row per (cohort, month offset) |
| 2. **Metric formula** | Numerator / denominator exactly? | active users in month N / users in cohort |
| 3. **Time bucket and boundaries** | Day, week (Mon or Sun start?), month? Inclusive or exclusive ends? Time zone? | calendar months, UTC |
| 4. **Missing data and zero denominators** | What if a period has no rows? Divide by zero? | fill missing months, use `NULLIF` |
| 5. **Validate** | Check with a tiny hand-made dataset | 5 users, compute by hand |

> Say the definitions **out loud** in the interview ("I'll define Day-1 retention as users active exactly one day after signup, divided by signups"). It shows seniority and avoids wrong-question mistakes.

## 0.2 Dialect Conventions Used in This Guide

Examples use **MySQL 8** syntax. PostgreSQL equivalents:

| Task | MySQL (used here) | PostgreSQL |
|---|---|---|
| First day of month | `DATE_SUB(d, INTERVAL DAYOFMONTH(d)-1 DAY)` | `DATE_TRUNC('month', d)::date` |
| Add days | `DATE_ADD(d, INTERVAL n DAY)` | `d + n` |
| Date difference (days) | `DATEDIFF(later, earlier)` | `later - earlier` |
| Whole minutes between timestamps | `TIMESTAMPDIFF(MINUTE, a, b)` | `FLOOR(EXTRACT(EPOCH FROM (b - a)) / 60)` |
| Whole months between month-starts | `TIMESTAMPDIFF(MONTH, a, b)` | `(EXTRACT(YEAR FROM AGE(b, a)) * 12 + EXTRACT(MONTH FROM AGE(b, a)))` |
| Weekday number, Monday first | `WEEKDAY(d)` (Mon = 0) | `EXTRACT(ISODOW FROM d)` (Mon = 1) |
| Year-month text | `DATE_FORMAT(d, '%Y-%m')` | `TO_CHAR(d, 'YYYY-MM')` |

---

# PART 1: Sample Data (Create Once)

Works in MySQL 8+ (and PostgreSQL with minor type changes such as `DATETIME` to `TIMESTAMP`).

```sql
-- 1) orders (12 orders, 6 customers, Jan to Apr 2026)
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

-- 2) users and activity (product usage, Oct 2026)
CREATE TABLE users (
    user_id      INT PRIMARY KEY,
    signup_date  DATE
);
INSERT INTO users VALUES
(1, '2026-10-01'), (2, '2026-10-01'), (3, '2026-10-02'),
(4, '2026-10-03'), (5, '2026-10-03');

CREATE TABLE activity (
    user_id        INT,
    activity_date  DATE
);
INSERT INTO activity VALUES
(1, '2026-10-01'), (1, '2026-10-02'), (1, '2026-10-08'),
(2, '2026-10-01'), (2, '2026-10-02'),
(3, '2026-10-02'), (3, '2026-10-09'),
(4, '2026-10-03'), (4, '2026-10-04'), (4, '2026-10-10'),
(5, '2026-10-03');

-- 3) funnel_events
CREATE TABLE funnel_events (
    user_id     INT,
    event_name  VARCHAR(20),
    event_time  DATETIME
);
INSERT INTO funnel_events VALUES
(1, 'view',     '2026-10-21 10:00:00'),
(1, 'cart',     '2026-10-21 10:05:00'),
(1, 'checkout', '2026-10-21 10:10:00'),
(1, 'purchase', '2026-10-21 10:15:00'),
(2, 'view',     '2026-10-21 10:30:00'),
(2, 'cart',     '2026-10-21 10:35:00'),
(2, 'checkout', '2026-10-21 10:50:00'),
(3, 'view',     '2026-10-21 11:00:00'),
(3, 'cart',     '2026-10-21 11:05:00'),
(4, 'view',     '2026-10-21 11:10:00'),
(5, 'view',     '2026-10-21 11:20:00'),
(5, 'cart',     '2026-10-21 11:28:00'),
(5, 'checkout', '2026-10-21 11:32:00'),
(5, 'purchase', '2026-10-21 11:40:00'),
(6, 'view',     '2026-10-21 12:00:00'),
(6, 'cart',     '2026-10-21 12:10:00');

-- 4) bookings (date ranges)
CREATE TABLE bookings (
    booking_id  INT PRIMARY KEY,
    room_id     INT,
    start_date  DATE,
    end_date    DATE
);
INSERT INTO bookings VALUES
(1, 101, '2026-10-01', '2026-10-05'),
(2, 101, '2026-10-04', '2026-10-08'),
(3, 101, '2026-10-10', '2026-10-12'),
(4, 101, '2026-10-11', '2026-10-15'),
(5, 102, '2026-10-01', '2026-10-03'),
(6, 102, '2026-10-03', '2026-10-06'),
(7, 102, '2026-10-07', '2026-10-09');

-- 5) price_log (sparse updates)
CREATE TABLE price_log (
    log_date  DATE,
    price     INT
);
INSERT INTO price_log VALUES
('2026-10-01', 50), ('2026-10-04', 55), ('2026-10-06', 53);

-- 6) support_tickets
CREATE TABLE support_tickets (
    ticket_id          INT PRIMARY KEY,
    created_at         DATETIME,
    first_response_at  DATETIME,
    resolved_at        DATETIME
);
INSERT INTO support_tickets VALUES
(1, '2026-10-20 09:00:00', '2026-10-20 09:20:00', '2026-10-20 11:00:00'),
(2, '2026-10-20 10:00:00', '2026-10-20 10:45:00', '2026-10-21 10:00:00'),
(3, '2026-10-20 14:00:00', NULL, NULL),
(4, '2026-10-21 08:30:00', '2026-10-21 08:40:00', '2026-10-21 09:30:00');

-- 7) user_logins (streaks with tolerance)
CREATE TABLE user_logins (
    user_id     INT,
    login_date  DATE
);
INSERT INTO user_logins VALUES
(1, '2026-10-01'), (1, '2026-10-02'), (1, '2026-10-03'),
(1, '2026-10-05'), (1, '2026-10-06'), (1, '2026-10-09'),
(2, '2026-10-01'), (2, '2026-10-03'), (2, '2026-10-04'), (2, '2026-10-05');

-- 8) holidays
CREATE TABLE holidays (
    holiday_date  DATE,
    holiday_name  VARCHAR(30)
);
INSERT INTO holidays VALUES
('2026-10-22', 'Company Day'),
('2026-10-26', 'Founders Day');

-- 9) monthly_sales (two years, for YoY)
CREATE TABLE monthly_sales (
    month_start  DATE,
    revenue      INT
);
INSERT INTO monthly_sales VALUES
('2025-01-01',  800), ('2025-02-01',  900), ('2025-03-01', 1000),
('2026-01-01', 1000), ('2026-02-01', 1170), ('2026-03-01',  900);

-- 10) experiment (A/B test)
CREATE TABLE experiment (
    user_id    INT PRIMARY KEY,
    variant    CHAR(1),
    converted  INT
);
INSERT INTO experiment VALUES
(1,'A',1),(2,'A',0),(3,'A',0),(4,'A',1),(5,'A',0),
(6,'B',1),(7,'B',1),(8,'B',0),(9,'B',1),(10,'B',0);

-- 11) ad_events (for CTR)
CREATE TABLE ad_events (
    ad_id       INT,
    event_type  VARCHAR(12)
);
INSERT INTO ad_events VALUES
(1,'impression'),(1,'impression'),(1,'impression'),(1,'impression'),(1,'click'),
(2,'impression'),(2,'impression'),(2,'impression'),(2,'click'),
(3,'impression'),(3,'impression');

-- 12) order_items (market basket)
CREATE TABLE order_items (
    order_id  INT,
    product   VARCHAR(10)
);
INSERT INTO order_items VALUES
(1,'Bread'),(1,'Butter'),(1,'Jam'),
(2,'Bread'),(2,'Butter'),
(3,'Bread'),(3,'Milk'),
(4,'Butter'),(4,'Milk'),
(5,'Bread'),(5,'Butter'),(5,'Milk');

-- 13) quarterly_wide (for unpivot)
CREATE TABLE quarterly_wide (
    product  VARCHAR(5),
    q1 INT, q2 INT, q3 INT, q4 INT
);
INSERT INTO quarterly_wide VALUES
('A', 100, 120,  90, 150),
('B',  80,  60, 110,  70);
```

### Key tables (for reference)

**`orders`**

| order_id | customer_id | order_date | amount |
|---|---|---|---|
| 1 | 1 | 2026-01-05 | 500 |
| 2 | 2 | 2026-01-12 | 300 |
| 3 | 3 | 2026-01-20 | 450 |
| 4 | 1 | 2026-02-03 | 700 |
| 5 | 2 | 2026-02-14 | 200 |
| 6 | 4 | 2026-02-18 | 650 |
| 7 | 1 | 2026-03-07 | 400 |
| 8 | 3 | 2026-03-15 | 350 |
| 9 | 4 | 2026-03-21 | 150 |
| 10 | 5 | 2026-03-25 | 800 |
| 11 | 1 | 2026-04-02 | 600 |
| 12 | 6 | 2026-04-10 | 250 |

Facts to remember: total revenue **5350**, 12 orders, 6 customers. Active customers per month: Jan {1,2,3}, Feb {1,2,4}, Mar {1,3,4,5}, Apr {1,6}.

**`users`**: (1, 10-01), (2, 10-01), (3, 10-02), (4, 10-03), (5, 10-03)

**`activity`**

| user_id | activity_date |
|---|---|
| 1 | 2026-10-01, 2026-10-02, 2026-10-08 |
| 2 | 2026-10-01, 2026-10-02 |
| 3 | 2026-10-02, 2026-10-09 |
| 4 | 2026-10-03, 2026-10-04, 2026-10-10 |
| 5 | 2026-10-03 |

(Shown grouped for compactness; the table has one row per activity.)

**`funnel_events`** (user: steps taken): u1: view, cart, checkout, purchase | u2: view, cart, checkout | u3: view, cart | u4: view | u5: view, cart, checkout, purchase | u6: view, cart.

**`bookings`**

| booking_id | room_id | start_date | end_date |
|---|---|---|---|
| 1 | 101 | 2026-10-01 | 2026-10-05 |
| 2 | 101 | 2026-10-04 | 2026-10-08 |
| 3 | 101 | 2026-10-10 | 2026-10-12 |
| 4 | 101 | 2026-10-11 | 2026-10-15 |
| 5 | 102 | 2026-10-01 | 2026-10-03 |
| 6 | 102 | 2026-10-03 | 2026-10-06 |
| 7 | 102 | 2026-10-07 | 2026-10-09 |

---

# PART 2: MODULE 11: Time-Based Analytics

## 11.1 The Date and Time Toolkit

### Core functions (reference date: **2026-10-21**, a Wednesday)

| Task | MySQL | PostgreSQL | Result |
|---|---|---|---|
| Month start | `DATE_SUB(d, INTERVAL DAYOFMONTH(d)-1 DAY)` | `DATE_TRUNC('month', d)::date` | 2026-10-01 |
| Month end | `LAST_DAY(d)` | `(DATE_TRUNC('month', d) + INTERVAL '1 month - 1 day')::date` | 2026-10-31 |
| Week start (Monday) | `DATE_SUB(d, INTERVAL WEEKDAY(d) DAY)` | `DATE_TRUNC('week', d)::date` | 2026-10-19 |
| Quarter | `QUARTER(d)` | `EXTRACT(QUARTER FROM d)` | 4 |
| Day name | `DAYNAME(d)` | `TO_CHAR(d, 'FMDay')` | Wednesday |
| Weekday number | `WEEKDAY(d)` (Mon = 0) | `EXTRACT(ISODOW FROM d)` (Mon = 1) | 2 (MySQL) / 3 (PG) |
| Day of year | `DAYOFYEAR(d)` | `EXTRACT(DOY FROM d)` | 294 |
| ISO week number | `WEEK(d, 3)` | `EXTRACT(WEEK FROM d)` | 43 |
| Add one month | `DATE_ADD(d, INTERVAL 1 MONTH)` | `d + INTERVAL '1 month'` | 2026-11-21 |
| Days between | `DATEDIFF('2026-10-21', '2026-10-01')` | `DATE '2026-10-21' - DATE '2026-10-01'` | 20 |
| Whole months between | `TIMESTAMPDIFF(MONTH, '2026-01-15', d)` | via `AGE` | 9 |
| Year-month text | `DATE_FORMAT(d, '%Y-%m')` | `TO_CHAR(d, 'YYYY-MM')` | 2026-10 |
| Timestamp to date | `DATE(ts)` | `ts::date` | date part |

**Month arithmetic clamps to the last day of the month:** `2026-01-31 + 1 month = 2026-02-28` (both MySQL and PostgreSQL). Remember this when computing "same day next month."

### Bucketing pattern (the heart of time analytics)

To group by a time bucket, **convert each date to the first day of its bucket**, then `GROUP BY` that value:

```sql
-- month bucket
DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY)

-- week bucket (Monday start)
DATE_SUB(order_date, INTERVAL WEEKDAY(order_date) DAY)
```

Using a **date** (not a string like `'2026-03'`) keeps correct sorting, lets you add or subtract months, and makes joins easy.

### Boundary rule: use half-open ranges

For timestamps, always filter with `>= start AND < next_start`:

```sql
WHERE created_at >= '2026-10-21' AND created_at < '2026-10-22'    -- the whole day of Oct 21
```

`BETWEEN '2026-10-21' AND '2026-10-21'` on a timestamp column would only match midnight exactly. `BETWEEN ... AND '2026-10-21 23:59:59'` misses fractions of a second. Half-open ranges are always right.

### Time zones

Store timestamps in **UTC**, convert when reporting. India (IST) is UTC+05:30.

| Task | MySQL | PostgreSQL |
|---|---|---|
| UTC to IST | `CONVERT_TZ(ts, '+00:00', '+05:30')` | `(ts AT TIME ZONE 'UTC') AT TIME ZONE 'Asia/Kolkata'` |

Example: `2026-10-21 18:30:00` UTC becomes `2026-10-22 00:00:00` IST. The **calendar date changes**, so "daily active users" counted by UTC date and by IST date can differ. State which time zone you use for the day boundary.

---

## P34: Daily / Weekly / Monthly Aggregation

**Use when:** "monthly revenue", "orders per week", "active customers per month".

**Template:** bucket the date, then `GROUP BY` the bucket.

### Monthly report

```sql
SELECT DATE_FORMAT(order_date, '%Y-%m')   AS month,
       COUNT(*)                           AS orders,
       COUNT(DISTINCT customer_id)        AS customers,
       SUM(amount)                        AS revenue,
       ROUND(AVG(amount), 2)              AS avg_order_value
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

| month | orders | customers | revenue | avg_order_value |
|---|---|---|---|---|
| 2026-01 | 3 | 3 | 1250 | 416.67 |
| 2026-02 | 3 | 3 | 1550 | 516.67 |
| 2026-03 | 4 | 4 | 1700 | 425.00 |
| 2026-04 | 2 | 2 | 850 | 425.00 |

**Step-by-step (March):** orders 7, 8, 9, 10 (400 + 350 + 150 + 800 = 1700), customers 1, 3, 4, 5 = 4 distinct, average = 1700 / 4 = 425.

> **`COUNT(*)` vs `COUNT(DISTINCT customer_id)`:** orders vs unique customers. Interviewers often test whether you notice the difference.

### Weekly report (Monday-start weeks), March only

```sql
SELECT DATE_SUB(order_date, INTERVAL WEEKDAY(order_date) DAY) AS week_start,
       COUNT(*)      AS orders,
       SUM(amount)   AS revenue
FROM orders
WHERE order_date >= '2026-03-01' AND order_date < '2026-04-01'
GROUP BY DATE_SUB(order_date, INTERVAL WEEKDAY(order_date) DAY)
ORDER BY week_start;
```

| week_start | orders | revenue |
|---|---|---|
| 2026-03-02 | 1 | 400 |
| 2026-03-09 | 1 | 350 |
| 2026-03-16 | 1 | 150 |
| 2026-03-23 | 1 | 800 |

**Explanation:** 2026-03-07 is a Saturday, so its week starts Monday 2026-03-02. 2026-03-15 is a Sunday, so its week starts Monday 2026-03-09. 2026-03-21 (Saturday) belongs to the week of 03-16, and 2026-03-25 (Wednesday) to the week of 03-23.

> **Missing buckets:** if a week or month has **no rows**, it will simply not appear. To show zeros, build a calendar (see P40) and `LEFT JOIN`.

---

## P35: Month-over-Month and Year-over-Year Growth

**Formula:** `growth % = (current - previous) / previous * 100`

### Month-over-month (MoM)

```sql
WITH monthly AS (
    SELECT DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS month_start,
           SUM(amount) AS revenue
    FROM orders
    GROUP BY DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY)
)
SELECT month_start,
       revenue,
       LAG(revenue) OVER (ORDER BY month_start) AS prev_revenue,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (ORDER BY month_start))
             / NULLIF(LAG(revenue) OVER (ORDER BY month_start), 0), 2) AS mom_growth_pct,
       SUM(revenue) OVER (ORDER BY month_start
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative_revenue
FROM monthly
ORDER BY month_start;
```

| month_start | revenue | prev_revenue | mom_growth_pct | cumulative_revenue |
|---|---|---|---|---|
| 2026-01-01 | 1250 | NULL | NULL | 1250 |
| 2026-02-01 | 1550 | 1250 | 24.00 | 2800 |
| 2026-03-01 | 1700 | 1550 | 9.68 | 4500 |
| 2026-04-01 | 850 | 1700 | -50.00 | 5350 |

**Step-by-step:**
- Feb: (1550 - 1250) / 1250 = 0.24 = **24.00%**
- Mar: (1700 - 1550) / 1550 = 0.0968 = **9.68%**
- Apr: (850 - 1700) / 1700 = -0.50 = **-50.00%**
- `NULLIF(..., 0)` protects against divide-by-zero when a previous month has 0 revenue.

> **Warning about LAG on months:** `LAG` takes the **previous row**, not the previous **calendar month**. If a month has no sales it is missing from the table, and `LAG` silently compares with an older month. Fill missing months with a calendar first (P40) when gaps are possible.

### Year-over-year (YoY): same month, previous year

Trick: **partition by the month number**, so "previous row" in each partition is the same month of the previous year.

```sql
SELECT month_start,
       revenue,
       LAG(revenue) OVER (PARTITION BY MONTH(month_start) ORDER BY month_start) AS same_month_last_year,
       ROUND(100.0 * (revenue - LAG(revenue) OVER (PARTITION BY MONTH(month_start)
                                                   ORDER BY month_start))
             / LAG(revenue) OVER (PARTITION BY MONTH(month_start)
                                  ORDER BY month_start), 2) AS yoy_growth_pct
FROM monthly_sales
ORDER BY month_start;
```

| month_start | revenue | same_month_last_year | yoy_growth_pct |
|---|---|---|---|
| 2025-01-01 | 800 | NULL | NULL |
| 2025-02-01 | 900 | NULL | NULL |
| 2025-03-01 | 1000 | NULL | NULL |
| 2026-01-01 | 1000 | 800 | 25.00 |
| 2026-02-01 | 1170 | 900 | 30.00 |
| 2026-03-01 | 900 | 1000 | -10.00 |

**Step-by-step (Feb):** (1170 - 900) / 900 = 0.30 = **30.00%**.

### Year-to-date (YTD) running total

```sql
SELECT month_start, revenue,
       SUM(revenue) OVER (PARTITION BY YEAR(month_start)
                          ORDER BY month_start
                          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS ytd_revenue
FROM monthly_sales
WHERE YEAR(month_start) = 2026
ORDER BY month_start;
```

| month_start | revenue | ytd_revenue |
|---|---|---|
| 2026-01-01 | 1000 | 1000 |
| 2026-02-01 | 1170 | 2170 |
| 2026-03-01 | 900 | 3070 |

---

## P36: Retention and Cohort Analysis

**Definitions:**
- **Cohort:** a group of users who started in the same period (here: **month of first order**).
- **Month number (offset):** how many months after the cohort month an activity happened (0 = the cohort month itself).
- **Retention %:** `active users in month N / users in the cohort`.

### Step 1: customer-month activity

`orders` reduced to one row per (customer, month):

| customer_id | active_month |
|---|---|
| 1 | Jan, Feb, Mar, Apr |
| 2 | Jan, Feb |
| 3 | Jan, Mar |
| 4 | Feb, Mar |
| 5 | Mar |
| 6 | Apr |

### Step 2: cohort assignment (first month per customer)

| customer_id | cohort_month |
|---|---|
| 1, 2, 3 | 2026-01 |
| 4 | 2026-02 |
| 5 | 2026-03 |
| 6 | 2026-04 |

### Step 3: the full retention query (long format)

```sql
WITH m AS (
    SELECT DISTINCT customer_id,
           DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS active_month
    FROM orders
),
cohort AS (
    SELECT customer_id, MIN(active_month) AS cohort_month
    FROM m
    GROUP BY customer_id
),
joined AS (
    SELECT c.cohort_month,
           TIMESTAMPDIFF(MONTH, c.cohort_month, m.active_month) AS month_number,
           m.customer_id
    FROM m
    JOIN cohort c ON c.customer_id = m.customer_id
),
agg AS (
    SELECT cohort_month, month_number, COUNT(DISTINCT customer_id) AS active_users
    FROM joined
    GROUP BY cohort_month, month_number
)
SELECT cohort_month,
       month_number,
       active_users,
       FIRST_VALUE(active_users) OVER (PARTITION BY cohort_month ORDER BY month_number) AS cohort_size,
       ROUND(100.0 * active_users /
             FIRST_VALUE(active_users) OVER (PARTITION BY cohort_month ORDER BY month_number), 2) AS retention_pct
FROM agg
ORDER BY cohort_month, month_number;
```

| cohort_month | month_number | active_users | cohort_size | retention_pct |
|---|---|---|---|---|
| 2026-01-01 | 0 | 3 | 3 | 100.00 |
| 2026-01-01 | 1 | 2 | 3 | 66.67 |
| 2026-01-01 | 2 | 2 | 3 | 66.67 |
| 2026-01-01 | 3 | 1 | 3 | 33.33 |
| 2026-02-01 | 0 | 1 | 1 | 100.00 |
| 2026-02-01 | 1 | 1 | 1 | 100.00 |
| 2026-03-01 | 0 | 1 | 1 | 100.00 |
| 2026-04-01 | 0 | 1 | 1 | 100.00 |

**Step-by-step (January cohort = customers 1, 2, 3):**
- Month 0 (Jan): all 3 ordered, so 100%.
- Month 1 (Feb): customers 1 and 2 ordered (3 did not), so 2/3 = **66.67%**.
- Month 2 (Mar): customers 1 and 3 ordered, so 2/3 = **66.67%** (customer 2 came back? no; customer 3 returned after skipping Feb).
- Month 3 (Apr): only customer 1, so 1/3 = **33.33%**.
- February cohort (customer 4): active in Feb and Mar, absent in Apr.
- Why `FIRST_VALUE`? Month 0's active users **equal the cohort size** (everyone is active in their first month), so it gives the denominator without another join.

### Step 4: the classic cohort triangle (pivot)

```sql
-- continue from the long result as a CTE named r
SELECT cohort_month,
       MAX(cohort_size) AS cohort_size,
       MAX(CASE WHEN month_number = 0 THEN retention_pct END) AS m0,
       MAX(CASE WHEN month_number = 1 THEN retention_pct END) AS m1,
       MAX(CASE WHEN month_number = 2 THEN retention_pct END) AS m2,
       MAX(CASE WHEN month_number = 3 THEN retention_pct END) AS m3
FROM r
GROUP BY cohort_month
ORDER BY cohort_month;
```

| cohort_month | cohort_size | m0 | m1 | m2 | m3 |
|---|---|---|---|---|---|
| 2026-01-01 | 3 | 100.00 | 66.67 | 66.67 | 33.33 |
| 2026-02-01 | 1 | 100.00 | 100.00 | NULL | NULL |
| 2026-03-01 | 1 | 100.00 | NULL | NULL | NULL |
| 2026-04-01 | 1 | 100.00 | NULL | NULL | NULL |

**Reading it:** NULL means "no active users", which is either a true 0% retention (for example Feb cohort in April) or a month that has not happened yet (for example any cohort in May). A production report distinguishes the two by comparing `cohort_month + N months` with the last month of data.

### Day-N retention (Day-1, Day-7), product-style

**Definition:** Day-N retention = users active **exactly N days after signup** / users who signed up.

```sql
WITH r AS (
    SELECT u.signup_date, u.user_id,
           MAX(CASE WHEN DATEDIFF(a.activity_date, u.signup_date) = 1 THEN 1 ELSE 0 END) AS d1,
           MAX(CASE WHEN DATEDIFF(a.activity_date, u.signup_date) = 7 THEN 1 ELSE 0 END) AS d7
    FROM users u
    LEFT JOIN activity a ON a.user_id = u.user_id
    GROUP BY u.signup_date, u.user_id
)
SELECT signup_date,
       COUNT(*)                                  AS cohort_size,
       SUM(d1)                                   AS d1_users,
       ROUND(100.0 * SUM(d1) / COUNT(*), 2)      AS d1_pct,
       SUM(d7)                                   AS d7_users,
       ROUND(100.0 * SUM(d7) / COUNT(*), 2)      AS d7_pct
FROM r
GROUP BY signup_date
ORDER BY signup_date;
```

| signup_date | cohort_size | d1_users | d1_pct | d7_users | d7_pct |
|---|---|---|---|---|---|
| 2026-10-01 | 2 | 2 | 100.00 | 1 | 50.00 |
| 2026-10-02 | 1 | 0 | 0.00 | 1 | 100.00 |
| 2026-10-03 | 2 | 1 | 50.00 | 1 | 50.00 |

**Step-by-step:**
- Signup 10-01 (users 1, 2): both were active on 10-02 (Day 1). Only user 1 was active on 10-08 (Day 7).
- Signup 10-02 (user 3): not active on 10-03, but active on 10-09 (Day 7).
- Signup 10-03 (users 4, 5): user 4 was active on 10-04 (Day 1) and 10-10 (Day 7); user 5 never returned.
- **Overall:** Day-1 = 3/5 = **60%**, Day-7 = 3/5 = **60%**.
- The inner `MAX(CASE ...)` per **user** guarantees each user counts once even if they have many activity rows.
- The `LEFT JOIN` keeps users who never came back (so the denominator stays correct).

> **Variants to know:** "rolling retention" (active on **or after** Day N), "unbounded retention," "week-N retention" (use `FLOOR(DATEDIFF/7)`). This is LeetCode's *Game Play Analysis IV/V* family.

---

## P37: Churn and Activation

### Monthly churn

**Definition:** a customer **churned in month M** if they were active in M but **not** active in M+1.

```sql
WITH cm AS (
    SELECT DISTINCT customer_id,
           DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS m
    FROM orders
)
SELECT a.m                                            AS from_month,
       COUNT(*)                                       AS active_customers,
       COUNT(b.customer_id)                           AS retained_next_month,
       COUNT(*) - COUNT(b.customer_id)                AS churned,
       ROUND(100.0 * (COUNT(*) - COUNT(b.customer_id)) / COUNT(*), 2) AS churn_rate_pct
FROM cm a
LEFT JOIN cm b
       ON b.customer_id = a.customer_id
      AND b.m = DATE_ADD(a.m, INTERVAL 1 MONTH)
WHERE a.m < (SELECT MAX(m) FROM cm)
GROUP BY a.m
ORDER BY a.m;
```

| from_month | active_customers | retained_next_month | churned | churn_rate_pct |
|---|---|---|---|---|
| 2026-01-01 | 3 | 2 | 1 | 33.33 |
| 2026-02-01 | 3 | 2 | 1 | 33.33 |
| 2026-03-01 | 4 | 1 | 3 | 75.00 |

**Step-by-step (March to April):** March actives are {1, 3, 4, 5}. April actives are {1, 6}. Only customer 1 appears in both, so retained = 1 and churned = 3 (customers 3, 4, 5), giving 3/4 = **75%**.

**Explanation of the self-join:**
- `a` is each customer-month; `b` is the **same customer in the next month**.
- `LEFT JOIN` keeps every `a` row; `b.customer_id` is NULL when the customer did not return.
- `COUNT(b.customer_id)` counts only non-NULL matches (= retained).
- The `WHERE a.m < MAX(m)` filter drops April, because May has not been observed yet.

### Activation (time to second order)

**Definition:** a customer is "activated" if they place a **second order within 30 days** of their first.

```sql
WITH ranked AS (
    SELECT customer_id, order_date,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date, order_id) AS rn
    FROM orders
),
firsts AS (
    SELECT customer_id,
           MAX(CASE WHEN rn = 1 THEN order_date END) AS first_order,
           MAX(CASE WHEN rn = 2 THEN order_date END) AS second_order
    FROM ranked
    GROUP BY customer_id
)
SELECT customer_id, first_order, second_order,
       DATEDIFF(second_order, first_order) AS days_to_second
FROM firsts
ORDER BY customer_id;
```

| customer_id | first_order | second_order | days_to_second |
|---|---|---|---|
| 1 | 2026-01-05 | 2026-02-03 | 29 |
| 2 | 2026-01-12 | 2026-02-14 | 33 |
| 3 | 2026-01-20 | 2026-03-15 | 54 |
| 4 | 2026-02-18 | 2026-03-21 | 31 |
| 5 | 2026-03-25 | NULL | NULL |
| 6 | 2026-04-10 | NULL | NULL |

**Activation rate (within 30 days):** only customer 1 qualifies (29 days), so 1 of 6 = **16.67%**:

```sql
-- appended after the firsts CTE
SELECT ROUND(100.0 * SUM(CASE WHEN DATEDIFF(second_order, first_order) <= 30 THEN 1 ELSE 0 END)
             / COUNT(*), 2) AS activation_rate_pct
FROM firsts;
```

| activation_rate_pct |
|---|
| 16.67 |

(`NULL <= 30` is unknown, so customers with no second order fall to `ELSE 0`: correct.)

---

## P38: Funnel Analysis

**Definition:** a funnel counts how many users reach each step, and the **drop-off** between steps.

`funnel_events` steps: `view` then `cart` then `checkout` then `purchase`.

### Step counts and conversion

```sql
WITH step_users AS (
    SELECT event_name, COUNT(DISTINCT user_id) AS users
    FROM funnel_events
    GROUP BY event_name
),
ordered AS (
    SELECT CASE event_name WHEN 'view' THEN 1 WHEN 'cart' THEN 2
                           WHEN 'checkout' THEN 3 WHEN 'purchase' THEN 4 END AS step_no,
           event_name, users
    FROM step_users
)
SELECT step_no, event_name, users,
       ROUND(100.0 * users / FIRST_VALUE(users) OVER (ORDER BY step_no), 2) AS pct_of_first,
       ROUND(100.0 * users / LAG(users) OVER (ORDER BY step_no), 2)         AS pct_of_prev,
       LAG(users) OVER (ORDER BY step_no) - users                           AS dropped
FROM ordered
ORDER BY step_no;
```

| step_no | event_name | users | pct_of_first | pct_of_prev | dropped |
|---|---|---|---|---|---|
| 1 | view | 6 | 100.00 | NULL | NULL |
| 2 | cart | 5 | 83.33 | 83.33 | 1 |
| 3 | checkout | 3 | 50.00 | 60.00 | 2 |
| 4 | purchase | 2 | 33.33 | 66.67 | 1 |

**Reading it:**
- 6 users viewed; 5 added to cart (83.33%); 3 reached checkout (60% of cart users; 50% of viewers); 2 purchased (66.67% of checkout users; **33.33% overall conversion**).
- The biggest absolute drop is **cart to checkout** (2 users lost).

### Strict (ordered) funnel: furthest step per user

Counting events by name can be wrong if users do steps out of order or repeat them. A stricter approach finds the **furthest step each user reached**:

```sql
WITH user_steps AS (
    SELECT user_id,
           MAX(CASE event_name WHEN 'view' THEN 1 WHEN 'cart' THEN 2
                               WHEN 'checkout' THEN 3 WHEN 'purchase' THEN 4 END) AS furthest_step
    FROM funnel_events
    GROUP BY user_id
)
SELECT user_id, furthest_step
FROM user_steps
ORDER BY user_id;
```

| user_id | furthest_step |
|---|---|
| 1 | 4 |
| 2 | 3 |
| 3 | 2 |
| 4 | 1 |
| 5 | 4 |
| 6 | 2 |

Users who reached **at least** step k = users with `furthest_step >= k`: step 1: 6, step 2: 5, step 3: 3, step 4: 2 (the same counts as above). If the strict order of **timestamps** matters (cart must be after view), compare `MIN(event_time)` per step per user and require increasing times.

### Time to convert (view to purchase)

```sql
SELECT ROUND(AVG(TIMESTAMPDIFF(MINUTE, view_time, purchase_time)), 2) AS avg_minutes_to_purchase
FROM (SELECT user_id,
             MIN(CASE WHEN event_name = 'view'     THEN event_time END) AS view_time,
             MIN(CASE WHEN event_name = 'purchase' THEN event_time END) AS purchase_time
      FROM funnel_events
      GROUP BY user_id
      HAVING MIN(CASE WHEN event_name = 'purchase' THEN event_time END) IS NOT NULL) t;
```

| avg_minutes_to_purchase |
|---|
| 17.50 |

**Step-by-step:** user 1: 10:00 to 10:15 = 15 min; user 5: 11:20 to 11:40 = 20 min; average = (15 + 20) / 2 = **17.50**.

---

## P39: Active Users (DAU / WAU / MAU, Rolling Actives)

**Definitions:**

| Metric | Meaning |
|---|---|
| **DAU** | distinct users active on a day |
| **WAU** | distinct users active in the last 7 days (rolling, including today) |
| **MAU** | distinct users active in the last 30 days, or in the calendar month |
| **Stickiness** | `DAU / MAU` (how often monthly users come back daily) |

> **Why not `COUNT(DISTINCT user_id) OVER (ORDER BY date ROWS ...)`?** Most databases do not support `DISTINCT` inside window functions. A distinct rolling count needs a **calendar + range join** (or a correlated subquery).

```sql
WITH RECURSIVE cal AS (
    SELECT CAST('2026-10-01' AS DATE) AS d
    UNION ALL
    SELECT DATE_ADD(d, INTERVAL 1 DAY) FROM cal WHERE d < '2026-10-10'
)
SELECT c.d AS activity_date,
       (SELECT COUNT(DISTINCT a.user_id)
        FROM activity a
        WHERE a.activity_date = c.d)                                              AS dau,
       (SELECT COUNT(DISTINCT a.user_id)
        FROM activity a
        WHERE a.activity_date BETWEEN DATE_SUB(c.d, INTERVAL 6 DAY) AND c.d)      AS wau_rolling
FROM cal c
ORDER BY c.d;
```

| activity_date | dau | wau_rolling |
|---|---|---|
| 2026-10-01 | 2 | 2 |
| 2026-10-02 | 3 | 3 |
| 2026-10-03 | 2 | 5 |
| 2026-10-04 | 1 | 5 |
| 2026-10-05 | 0 | 5 |
| 2026-10-06 | 0 | 5 |
| 2026-10-07 | 0 | 5 |
| 2026-10-08 | 1 | 5 |
| 2026-10-09 | 1 | 4 |
| 2026-10-10 | 1 | 3 |

**Step-by-step:**
- **DAU 10-02** = users {1, 2, 3} = 3.
- **WAU 10-03** = window 10-01 to 10-03 = users {1, 2, 3, 4, 5} = 5.
- **WAU 10-09** = window 10-03 to 10-09: 10-03 {4, 5}, 10-04 {4}, 10-08 {1}, 10-09 {3} gives {1, 3, 4, 5} = **4** (user 2 aged out).
- **WAU 10-10** = window 10-04 to 10-10: {4}, {1}, {3}, {4} gives {1, 3, 4} = **3**.
- The calendar CTE guarantees a row (with DAU = 0) for days without activity.

**MAU for October:** distinct users with any activity in the month = 5. **Stickiness on 10-02** = DAU / MAU = 3 / 5 = **60%**.

```sql
SELECT COUNT(DISTINCT user_id) AS mau_oct
FROM activity
WHERE activity_date >= '2026-10-01' AND activity_date < '2026-11-01';
```

| mau_oct |
|---|
| 5 |

---

## P40: Fill Missing Dates (Calendar / Date Spine)

You already built a zero-filled calendar in Phase 3. Two more variations:

### A) Fill missing **months** with zero

```sql
WITH RECURSIVE months AS (
    SELECT CAST('2026-01-01' AS DATE) AS m
    UNION ALL
    SELECT DATE_ADD(m, INTERVAL 1 MONTH) FROM months WHERE m < '2026-06-01'
),
monthly AS (
    SELECT DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS m,
           SUM(amount) AS revenue
    FROM orders
    GROUP BY DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY)
)
SELECT mo.m AS month_start, COALESCE(mt.revenue, 0) AS revenue
FROM months mo
LEFT JOIN monthly mt ON mt.m = mo.m
ORDER BY mo.m;
```

| month_start | revenue |
|---|---|
| 2026-01-01 | 1250 |
| 2026-02-01 | 1550 |
| 2026-03-01 | 1700 |
| 2026-04-01 | 850 |
| 2026-05-01 | 0 |
| 2026-06-01 | 0 |

### B) Carry forward the last known value (forward fill)

`price_log` only stores days when the price **changed**: 10-01 = 50, 10-04 = 55, 10-06 = 53. We want the price **in effect on every day**.

**Trick:** join to a calendar, then create a **group id** that increments every time a real value appears (`COUNT(price)` ignores NULLs). Every date until the next real value shares the same group, and `MAX(price)` within the group is the value to carry.

```sql
WITH RECURSIVE cal AS (
    SELECT CAST('2026-10-01' AS DATE) AS d
    UNION ALL
    SELECT DATE_ADD(d, INTERVAL 1 DAY) FROM cal WHERE d < '2026-10-07'
),
j AS (
    SELECT c.d, p.price,
           COUNT(p.price) OVER (ORDER BY c.d) AS grp
    FROM cal c
    LEFT JOIN price_log p ON p.log_date = c.d
)
SELECT d AS price_date,
       MAX(price) OVER (PARTITION BY grp) AS price_in_effect
FROM j
ORDER BY d;
```

Intermediate table `j`:

| d | price | grp |
|---|---|---|
| 2026-10-01 | 50 | 1 |
| 2026-10-02 | NULL | 1 |
| 2026-10-03 | NULL | 1 |
| 2026-10-04 | 55 | 2 |
| 2026-10-05 | NULL | 2 |
| 2026-10-06 | 53 | 3 |
| 2026-10-07 | NULL | 3 |

Final output:

| price_date | price_in_effect |
|---|---|
| 2026-10-01 | 50 |
| 2026-10-02 | 50 |
| 2026-10-03 | 50 |
| 2026-10-04 | 55 |
| 2026-10-05 | 55 |
| 2026-10-06 | 53 |
| 2026-10-07 | 53 |

> PostgreSQL has no `IGNORE NULLS` either (Snowflake, Oracle and BigQuery do: `LAST_VALUE(price IGNORE NULLS)`). This `COUNT` + `MAX` trick works everywhere.

---

## P41: Overlapping Intervals / Date Ranges

**Overlap rule (two ranges A and B overlap if each starts before the other ends):**

```text
A.start <= B.end   AND   A.end >= B.start        (inclusive ends)
A.start <  B.end   AND   A.end >  B.start        (exclusive ends: checkout day free for the next guest)
```

### A) Find overlapping booking pairs in the same room

```sql
SELECT a.room_id,
       a.booking_id AS booking_a, a.start_date AS a_start, a.end_date AS a_end,
       b.booking_id AS booking_b, b.start_date AS b_start, b.end_date AS b_end
FROM bookings a
JOIN bookings b
  ON a.room_id = b.room_id
 AND a.booking_id < b.booking_id          -- each pair once, never with itself
 AND a.start_date <= b.end_date
 AND a.end_date   >= b.start_date
ORDER BY a.room_id, a.booking_id;
```

| room_id | booking_a | a_start | a_end | booking_b | b_start | b_end |
|---|---|---|---|---|---|---|
| 101 | 1 | 2026-10-01 | 2026-10-05 | 2 | 2026-10-04 | 2026-10-08 |
| 101 | 3 | 2026-10-10 | 2026-10-12 | 4 | 2026-10-11 | 2026-10-15 |
| 102 | 5 | 2026-10-01 | 2026-10-03 | 6 | 2026-10-03 | 2026-10-06 |

**Explanation:**
- (1, 2): booking 1 ends 10-05, booking 2 starts 10-04, so they overlap on 10-04 to 10-05.
- (5, 6): booking 5 ends 10-03 and booking 6 starts 10-03. With **inclusive** ends they overlap (same day). If checkout day is free, change `<=` and `>=` to `<` and `>` and this pair disappears.
- `a.booking_id < b.booking_id` avoids duplicate pairs (A,B) and (B,A) and self-pairs.

### B) Does a requested range conflict? (2026-10-06 to 2026-10-09)

```sql
SELECT booking_id, room_id, start_date, end_date
FROM bookings
WHERE start_date <= '2026-10-09'
  AND end_date   >= '2026-10-06'
ORDER BY booking_id;
```

| booking_id | room_id | start_date | end_date |
|---|---|---|---|
| 2 | 101 | 2026-10-04 | 2026-10-08 |
| 6 | 102 | 2026-10-03 | 2026-10-06 |
| 7 | 102 | 2026-10-07 | 2026-10-09 |

**Reading it:** Booking 2 (room 101) and bookings 6 and 7 (room 102) conflict with the requested dates. The same test, wrapped in `NOT EXISTS`, finds **rooms that are free** for the range.

### C) Merge overlapping intervals into continuous blocks (hard, very common)

**Idea:** sort by start date. Track the **largest end date seen so far** (`prev_max_end`). If a row starts **after** that, it begins a new block. Running-sum the "new block" flags to get a block id (same trick as gaps and islands).

```sql
WITH ordered AS (
    SELECT room_id, booking_id, start_date, end_date,
           MAX(end_date) OVER (PARTITION BY room_id
                               ORDER BY start_date, booking_id
                               ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS prev_max_end
    FROM bookings
),
flagged AS (
    SELECT *,
           CASE WHEN prev_max_end IS NULL OR start_date > prev_max_end THEN 1 ELSE 0 END AS new_block
    FROM ordered
),
blocks AS (
    SELECT *,
           SUM(new_block) OVER (PARTITION BY room_id ORDER BY start_date, booking_id) AS block_id
    FROM flagged
)
SELECT room_id,
       MIN(start_date) AS block_start,
       MAX(end_date)   AS block_end,
       COUNT(*)        AS bookings_merged
FROM blocks
GROUP BY room_id, block_id
ORDER BY room_id, block_start;
```

Step table for room 101:

| booking_id | start_date | end_date | prev_max_end | new_block | block_id |
|---|---|---|---|---|---|
| 1 | 2026-10-01 | 2026-10-05 | NULL | 1 | 1 |
| 2 | 2026-10-04 | 2026-10-08 | 2026-10-05 | 0 (starts before 10-05) | 1 |
| 3 | 2026-10-10 | 2026-10-12 | 2026-10-08 | 1 (starts after 10-08) | 2 |
| 4 | 2026-10-11 | 2026-10-15 | 2026-10-12 | 0 | 2 |

Final output:

| room_id | block_start | block_end | bookings_merged |
|---|---|---|---|
| 101 | 2026-10-01 | 2026-10-08 | 2 |
| 101 | 2026-10-10 | 2026-10-15 | 2 |
| 102 | 2026-10-01 | 2026-10-06 | 2 |
| 102 | 2026-10-07 | 2026-10-09 | 1 |

**Why `MAX(end_date)` over all earlier rows (not just `LAG`)?** An early long interval can swallow several later ones. Example: [1, 10], [2, 3], [4, 5]: the third starts after the second ends but still sits inside the first. Comparing with only the previous row would wrongly split it.

> **Adjacent days:** here a booking starting the day **after** the previous block ends (10-06 then 10-07 in room 102) is a **new** block. To treat adjacent days as continuous, use `start_date > DATE_ADD(prev_max_end, INTERVAL 1 DAY)`. Always confirm with the interviewer.

---

## P42: Time Difference Between Events

**Use when:** response time, time to resolve, time to first purchase, time between steps.

```sql
SELECT ticket_id, created_at, first_response_at, resolved_at,
       TIMESTAMPDIFF(MINUTE, created_at, first_response_at) AS response_min,
       TIMESTAMPDIFF(MINUTE, created_at, resolved_at)       AS resolution_min,
       CASE WHEN TIMESTAMPDIFF(MINUTE, created_at, first_response_at) <= 30
            THEN 'Met' ELSE 'Breached' END                  AS sla_30m
FROM support_tickets
ORDER BY ticket_id;
```

| ticket_id | created_at | first_response_at | resolved_at | response_min | resolution_min | sla_30m |
|---|---|---|---|---|---|---|
| 1 | 2026-10-20 09:00 | 2026-10-20 09:20 | 2026-10-20 11:00 | 20 | 120 | Met |
| 2 | 2026-10-20 10:00 | 2026-10-20 10:45 | 2026-10-21 10:00 | 45 | 1440 | Breached |
| 3 | 2026-10-20 14:00 | NULL | NULL | NULL | NULL | Breached |
| 4 | 2026-10-21 08:30 | 2026-10-21 08:40 | 2026-10-21 09:30 | 10 | 60 | Met |

**Notes:**
- Ticket 3 has no response yet: the difference is NULL. In the `CASE`, `NULL <= 30` is unknown, so it falls to `ELSE` ('Breached'). That is a **business decision** (an unanswered ticket counts as a breach); state it explicitly.

### Summary metrics

```sql
SELECT ROUND(AVG(TIMESTAMPDIFF(MINUTE, created_at, first_response_at)), 2) AS avg_response_min,
       ROUND(AVG(TIMESTAMPDIFF(MINUTE, created_at, resolved_at)), 2)       AS avg_resolution_min,
       ROUND(100.0 * SUM(CASE WHEN TIMESTAMPDIFF(MINUTE, created_at, first_response_at) <= 30
                              THEN 1 ELSE 0 END) / COUNT(*), 2)            AS sla_met_pct
FROM support_tickets;
```

| avg_response_min | avg_resolution_min | sla_met_pct |
|---|---|---|
| 25.00 | 540.00 | 50.00 |

**Step-by-step:**
- Average response ignores the NULL: (20 + 45 + 10) / 3 = **25.00** minutes.
- Average resolution: (120 + 1440 + 60) / 3 = **540.00** minutes (9 hours).
- SLA: 2 of 4 tickets met (tickets 1 and 4) = **50.00%**. The denominator is `COUNT(*)` (all tickets), so the unanswered ticket counts against the SLA.

> `TIMESTAMPDIFF` returns **whole units** (truncated), not rounded. For fractional hours, compute minutes and divide: `TIMESTAMPDIFF(MINUTE, a, b) / 60.0`.

---

## P43: Weekday / Weekend and Business-Day Calculations

### Weekday vs weekend revenue

```sql
SELECT CASE WHEN DAYOFWEEK(order_date) IN (1, 7) THEN 'Weekend' ELSE 'Weekday' END AS day_type,
       COUNT(*)     AS orders,
       SUM(amount)  AS revenue
FROM orders
GROUP BY CASE WHEN DAYOFWEEK(order_date) IN (1, 7) THEN 'Weekend' ELSE 'Weekday' END
ORDER BY day_type DESC;
```

(`DAYOFWEEK`: Sunday = 1 ... Saturday = 7. PostgreSQL: `EXTRACT(ISODOW FROM order_date) IN (6, 7)`.)

| day_type | orders | revenue |
|---|---|---|
| Weekday | 8 | 4250 |
| Weekend | 4 | 1100 |

**Which orders were on weekends?** Order 5 (Sat 02-14, 200), order 7 (Sat 03-07, 400), order 8 (Sun 03-15, 350), order 9 (Sat 03-21, 150): 200 + 400 + 350 + 150 = 1100. The remaining 8 orders total 4250.

### Business days between two dates (excluding weekends and holidays)

Calendar from **2026-10-19 (Mon) to 2026-10-30 (Fri)**; holidays: **10-22 (Thu)** and **10-26 (Mon)**.

| Date | Day | Business day? |
|---|---|---|
| 10-19 | Mon | Yes |
| 10-20 | Tue | Yes |
| 10-21 | Wed | Yes |
| 10-22 | Thu | No (Company Day) |
| 10-23 | Fri | Yes |
| 10-24 | Sat | No |
| 10-25 | Sun | No |
| 10-26 | Mon | No (Founders Day) |
| 10-27 | Tue | Yes |
| 10-28 | Wed | Yes |
| 10-29 | Thu | Yes |
| 10-30 | Fri | Yes |

```sql
WITH RECURSIVE cal AS (
    SELECT CAST('2026-10-19' AS DATE) AS d
    UNION ALL
    SELECT DATE_ADD(d, INTERVAL 1 DAY) FROM cal WHERE d < '2026-10-30'
)
SELECT COUNT(*) AS business_days
FROM cal c
WHERE WEEKDAY(c.d) < 5                                   -- Mon..Fri
  AND NOT EXISTS (SELECT 1 FROM holidays h WHERE h.holiday_date = c.d);
```

| business_days |
|---|
| 8 |

**Step-by-step:** 12 calendar days, 4 are weekend/holiday-free weekdays... in detail: 10 weekdays (Mon to Fri twice) minus 2 holidays that fall on weekdays = **8**.

> **Interview note:** "Add N business days" and "business days between" are usually solved with a **calendar table** that has an `is_business_day` flag. In real systems, keep a permanent calendar (date dimension) table.

---

## P44: Consecutive Activity With Gaps Tolerated

**Problem:** treat logins as one "streak" even if the user skips **one day** (gap of up to 2 days between logins). A gap of 3 or more days starts a new streak.

This is **Technique B** from Phase 4 with a looser gap rule.

```sql
WITH prev AS (
    SELECT user_id, login_date,
           LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date) AS prev_date
    FROM (SELECT DISTINCT user_id, login_date FROM user_logins) d
),
flagged AS (
    SELECT user_id, login_date,
           CASE WHEN prev_date IS NULL OR DATEDIFF(login_date, prev_date) > 2
                THEN 1 ELSE 0 END AS new_streak
    FROM prev
),
streaks AS (
    SELECT user_id, login_date,
           SUM(new_streak) OVER (PARTITION BY user_id ORDER BY login_date) AS streak_id
    FROM flagged
)
SELECT user_id, streak_id,
       MIN(login_date) AS streak_start,
       MAX(login_date) AS streak_end,
       COUNT(*)        AS login_days
FROM streaks
GROUP BY user_id, streak_id
ORDER BY user_id, streak_id;
```

| user_id | streak_id | streak_start | streak_end | login_days |
|---|---|---|---|---|
| 1 | 1 | 2026-10-01 | 2026-10-06 | 5 |
| 1 | 2 | 2026-10-09 | 2026-10-09 | 1 |
| 2 | 1 | 2026-10-01 | 2026-10-05 | 4 |

**Step-by-step (user 1):**

| login_date | prev_date | gap (days) | new_streak | streak_id |
|---|---|---|---|---|
| 10-01 | NULL | n/a | 1 | 1 |
| 10-02 | 10-01 | 1 | 0 | 1 |
| 10-03 | 10-02 | 1 | 0 | 1 |
| 10-05 | 10-03 | 2 (one day skipped, tolerated) | 0 | 1 |
| 10-06 | 10-05 | 1 | 0 | 1 |
| 10-09 | 10-06 | 3 (too long) | 1 | 2 |

User 2: 10-01 to 10-03 is a gap of 2 (tolerated), then consecutive days, so a single streak with 4 login days.

> `login_days` counts **days with a login** (5), not the calendar span of the streak (6 days from 10-01 to 10-06).

---

# PART 3: MODULE 12: Business and Product Metrics

## P45: Customer Lifetime Value (CLV), Average Order Value (AOV), Repeat Purchase Rate

**Formulas:**

| Metric | Formula |
|---|---|
| **AOV** | total revenue / number of orders |
| **Revenue per customer** | total revenue / distinct customers |
| **Historical CLV (simple)** | total revenue per customer so far |
| **Repeat purchase rate** | customers with 2 or more orders / all customers |

### Per-customer profile

```sql
SELECT customer_id,
       COUNT(*)                              AS orders,
       SUM(amount)                           AS total_spent,
       ROUND(AVG(amount), 2)                 AS aov,
       MIN(order_date)                       AS first_order,
       MAX(order_date)                       AS last_order,
       DATEDIFF(MAX(order_date), MIN(order_date)) AS lifetime_days
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | orders | total_spent | aov | first_order | last_order | lifetime_days |
|---|---|---|---|---|---|---|
| 1 | 4 | 2200 | 550.00 | 2026-01-05 | 2026-04-02 | 87 |
| 2 | 2 | 500 | 250.00 | 2026-01-12 | 2026-02-14 | 33 |
| 3 | 2 | 800 | 400.00 | 2026-01-20 | 2026-03-15 | 54 |
| 4 | 2 | 800 | 400.00 | 2026-02-18 | 2026-03-21 | 31 |
| 5 | 1 | 800 | 800.00 | 2026-03-25 | 2026-03-25 | 0 |
| 6 | 1 | 250 | 250.00 | 2026-04-10 | 2026-04-10 | 0 |

### Company KPIs

```sql
SELECT COUNT(*)                                          AS orders,
       COUNT(DISTINCT customer_id)                       AS customers,
       SUM(amount)                                       AS revenue,
       ROUND(SUM(amount) / COUNT(*), 2)                  AS aov,
       ROUND(SUM(amount) / COUNT(DISTINCT customer_id), 2) AS revenue_per_customer
FROM orders;
```

| orders | customers | revenue | aov | revenue_per_customer |
|---|---|---|---|---|
| 12 | 6 | 5350 | 445.83 | 891.67 |

### Repeat purchase rate

```sql
SELECT ROUND(100.0 * SUM(CASE WHEN order_count >= 2 THEN 1 ELSE 0 END) / COUNT(*), 2) AS repeat_rate_pct
FROM (SELECT customer_id, COUNT(*) AS order_count
      FROM orders
      GROUP BY customer_id) t;
```

| repeat_rate_pct |
|---|
| 66.67 |

**Step-by-step:** customers 1, 2, 3, 4 have 2 or more orders; customers 5 and 6 have exactly one. 4 / 6 = **66.67%**.

> **AOV vs revenue per customer:** AOV divides by **orders**, revenue per customer divides by **customers**. Mixing them up is a very common interview error.

---

## P46: New vs Returning Customers

**Definition:** in a given month, a customer is **new** if it is their first-ever order month; otherwise **returning**.

```sql
WITH o AS (
    SELECT customer_id, order_date, amount,
           MIN(order_date) OVER (PARTITION BY customer_id) AS first_order_date
    FROM orders
)
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
       COUNT(DISTINCT customer_id) AS active_customers,
       COUNT(DISTINCT CASE WHEN DATE_FORMAT(order_date, '%Y-%m') = DATE_FORMAT(first_order_date, '%Y-%m')
                           THEN customer_id END) AS new_customers,
       COUNT(DISTINCT CASE WHEN DATE_FORMAT(order_date, '%Y-%m') <> DATE_FORMAT(first_order_date, '%Y-%m')
                           THEN customer_id END) AS returning_customers,
       SUM(CASE WHEN DATE_FORMAT(order_date, '%Y-%m') = DATE_FORMAT(first_order_date, '%Y-%m')
                THEN amount ELSE 0 END) AS new_revenue,
       SUM(CASE WHEN DATE_FORMAT(order_date, '%Y-%m') <> DATE_FORMAT(first_order_date, '%Y-%m')
                THEN amount ELSE 0 END) AS returning_revenue
FROM o
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;
```

| month | active_customers | new_customers | returning_customers | new_revenue | returning_revenue |
|---|---|---|---|---|---|
| 2026-01 | 3 | 3 | 0 | 1250 | 0 |
| 2026-02 | 3 | 1 | 2 | 650 | 900 |
| 2026-03 | 4 | 1 | 3 | 800 | 900 |
| 2026-04 | 2 | 1 | 1 | 250 | 600 |

**Step-by-step (February):** active customers are {1, 2, 4}. Customer 4 placed their first order in February, so **new = 1** (revenue 650). Customers 1 and 2 ordered before, so **returning = 2** (revenue 700 + 200 = 900).

**Key trick:** `MIN(order_date) OVER (PARTITION BY customer_id)` attaches each customer's first order date to every one of their orders, without a join.

---

## P47: Conversion Rate, Click-Through Rate, Ratios of Two Counts

**General pattern:** conditional aggregation for numerator and denominator in one pass, `NULLIF` to protect against zero.

```sql
SELECT ad_id,
       SUM(CASE WHEN event_type = 'impression' THEN 1 ELSE 0 END) AS impressions,
       SUM(CASE WHEN event_type = 'click'      THEN 1 ELSE 0 END) AS clicks,
       ROUND(100.0 * SUM(CASE WHEN event_type = 'click' THEN 1 ELSE 0 END)
             / NULLIF(SUM(CASE WHEN event_type = 'impression' THEN 1 ELSE 0 END), 0), 2) AS ctr_pct
FROM ad_events
GROUP BY ad_id
ORDER BY ad_id;
```

| ad_id | impressions | clicks | ctr_pct |
|---|---|---|---|
| 1 | 4 | 1 | 25.00 |
| 2 | 3 | 1 | 33.33 |
| 3 | 2 | 0 | 0.00 |

**Reading it:** CTR = clicks / impressions. Ad 3 has impressions but no clicks, so its CTR is **0.00** (not NULL). If an ad had **zero impressions**, `NULLIF(..., 0)` would return NULL instead of an error.

> **Choose the denominator carefully:** "conversion rate" can be conversions / sessions, / unique users, or / visitors. State which. For user-level rates, use `COUNT(DISTINCT user_id)` for both numerator and denominator.

---

## P48: A/B Test Metrics by Variant

```sql
SELECT variant,
       COUNT(*)                                   AS users,
       SUM(converted)                             AS conversions,
       ROUND(100.0 * SUM(converted) / COUNT(*), 2) AS conversion_rate_pct
FROM experiment
GROUP BY variant
ORDER BY variant;
```

| variant | users | conversions | conversion_rate_pct |
|---|---|---|---|
| A | 5 | 2 | 40.00 |
| B | 5 | 3 | 60.00 |

**Relative lift of B over A:**

```sql
WITH rates AS (
    SELECT variant, 1.0 * SUM(converted) / COUNT(*) AS rate
    FROM experiment
    GROUP BY variant
)
SELECT ROUND(100.0 * (b.rate - a.rate) / a.rate, 2) AS lift_pct
FROM rates a
JOIN rates b ON a.variant = 'A' AND b.variant = 'B';
```

| lift_pct |
|---|
| 50.00 |

**Step-by-step:** B's rate is 0.60 and A's is 0.40. Lift = (0.60 - 0.40) / 0.40 = **50%** relative improvement (or 20 percentage points absolute).

**What SQL gives you:** counts and rates per variant. **Statistical significance** (z-test or chi-square) needs the counts and sample sizes: with only 5 users per group this difference is **not** significant. In interviews, mention sample-size and significance, and check:
- Users assigned to **both** variants (contamination)
- Equal exposure windows
- Segment checks (new vs returning users)

---

## P49: Top / Bottom N With Ties and Thresholds

### Top customers by revenue (ties handled explicitly)

Customer revenue: 1 = 2200; 3 = 800; 4 = 800; 5 = 800; 2 = 500; 6 = 250. Three customers tie at 800.

```sql
WITH cust AS (
    SELECT customer_id, SUM(amount) AS revenue
    FROM orders
    GROUP BY customer_id
),
ranked AS (
    SELECT customer_id, revenue,
           RANK()       OVER (ORDER BY revenue DESC) AS rnk,
           DENSE_RANK() OVER (ORDER BY revenue DESC) AS dense_rnk,
           ROW_NUMBER() OVER (ORDER BY revenue DESC, customer_id) AS rn
    FROM cust
)
SELECT customer_id, revenue, rnk, dense_rnk, rn
FROM ranked
ORDER BY rn;
```

| customer_id | revenue | rnk | dense_rnk | rn |
|---|---|---|---|---|
| 1 | 2200 | 1 | 1 | 1 |
| 3 | 800 | 2 | 2 | 2 |
| 4 | 800 | 2 | 2 | 3 |
| 5 | 800 | 2 | 2 | 4 |
| 2 | 500 | 5 | 3 | 5 |
| 6 | 250 | 6 | 4 | 6 |

**"Top 3 customers" depends on the definition:**

| Definition | Rows returned |
|---|---|
| `rn <= 3` (exactly 3 rows) | customers 1, 3, 4 (customer 5 is cut off arbitrarily by the tie-breaker) |
| `rnk <= 3` (standard competition rank) | customers 1, 3, 4, 5 (the tied group stays together) |
| `dense_rnk <= 3` (top 3 revenue levels) | customers 1, 3, 4, 5, 2 |

### Pareto: the customers who produce 80% of revenue

```sql
WITH cust AS (
    SELECT customer_id, SUM(amount) AS revenue
    FROM orders
    GROUP BY customer_id
)
SELECT customer_id, revenue,
       ROUND(100.0 * SUM(revenue) OVER (ORDER BY revenue DESC, customer_id
                                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
             / SUM(revenue) OVER (), 2) AS cumulative_pct
FROM cust
ORDER BY revenue DESC, customer_id;
```

| customer_id | revenue | cumulative_pct |
|---|---|---|
| 1 | 2200 | 41.12 |
| 3 | 800 | 56.07 |
| 4 | 800 | 71.03 |
| 5 | 800 | 85.98 |
| 2 | 500 | 95.33 |
| 6 | 250 | 100.00 |

**Reading it:** 4 customers (1, 3, 4, 5) account for **85.98%** of revenue. To list "customers needed to reach 80%," wrap this in a CTE and keep rows where the **previous** cumulative percent is below 80 (the row that crosses the threshold must be included).

### Threshold filters

"Customers whose revenue is above the average customer revenue" combines a CTE with a scalar subquery:

```sql
WITH cust AS (SELECT customer_id, SUM(amount) AS revenue FROM orders GROUP BY customer_id)
SELECT customer_id, revenue
FROM cust
WHERE revenue > (SELECT AVG(revenue) FROM cust)
ORDER BY customer_id;
```

| customer_id | revenue |
|---|---|
| 1 | 2200 |

(Average customer revenue = 5350 / 6 = 891.67; only customer 1 is above it.)

---

## P50: Market Basket (Items Bought Together)

**Goal:** find product **pairs** that appear in the same order, and how often.

**Technique:** self-join `order_items` on `order_id` with `a.product < b.product` (so each pair is counted once and never paired with itself).

```sql
SELECT a.product AS product_a,
       b.product AS product_b,
       COUNT(*)  AS orders_together,
       ROUND(100.0 * COUNT(*) / (SELECT COUNT(DISTINCT order_id) FROM order_items), 2) AS support_pct
FROM order_items a
JOIN order_items b
  ON a.order_id = b.order_id
 AND a.product < b.product
GROUP BY a.product, b.product
ORDER BY orders_together DESC, product_a, product_b;
```

| product_a | product_b | orders_together | support_pct |
|---|---|---|---|
| Bread | Butter | 3 | 60.00 |
| Bread | Milk | 2 | 40.00 |
| Butter | Milk | 2 | 40.00 |
| Bread | Jam | 1 | 20.00 |
| Butter | Jam | 1 | 20.00 |

**Step-by-step (pairs generated per order):**

| Order | Items | Pairs |
|---|---|---|
| 1 | Bread, Butter, Jam | (Bread, Butter), (Bread, Jam), (Butter, Jam) |
| 2 | Bread, Butter | (Bread, Butter) |
| 3 | Bread, Milk | (Bread, Milk) |
| 4 | Butter, Milk | (Butter, Milk) |
| 5 | Bread, Butter, Milk | (Bread, Butter), (Bread, Milk), (Butter, Milk) |

**Metrics:**
- **Support** = orders containing both / all orders. Bread + Butter: 3 / 5 = **60%**.
- **Confidence (Bread to Butter)** = orders with both / orders with Bread = 3 / 4 = **75%** (Bread appears in orders 1, 2, 3, 5).
- **Lift** = confidence / support(Butter). Butter appears in 4 of 5 orders (80%), so lift = 0.75 / 0.80 = **0.94** (slightly below 1, meaning Bread does not increase the chance of Butter).

> Pair generation grows quadratically with basket size (a 100-item basket creates 4,950 pairs). On big data, filter to popular items first and use a minimum support threshold.

---

## P51: Median and Percentile-Based Metrics

Order amounts sorted: 150, 200, 250, 300, 350, 400, 450, 500, 600, 650, 700, 800 (12 values). Mean = 445.83.

### Median order amount

```sql
WITH r AS (
    SELECT amount,
           ROW_NUMBER() OVER (ORDER BY amount) AS rn,
           COUNT(*)     OVER ()                AS cnt
    FROM orders
)
SELECT AVG(amount) AS median_order_amount
FROM r
WHERE rn IN (FLOOR((cnt + 1) / 2.0), CEIL((cnt + 1) / 2.0));
```

| median_order_amount |
|---|
| 425.00 |

**Step-by-step:** `cnt = 12`, `(12 + 1) / 2.0 = 6.5`, so take rows 6 and 7 (400 and 450), average = **425**.

### 90th percentile (nearest-rank method)

The value at rank `CEIL(0.90 * n)`:

```sql
WITH r AS (
    SELECT amount,
           ROW_NUMBER() OVER (ORDER BY amount) AS rn,
           COUNT(*)     OVER ()                AS cnt
    FROM orders
)
SELECT MIN(amount) AS p90_order_amount
FROM r
WHERE rn >= CEIL(0.90 * cnt);
```

| p90_order_amount |
|---|
| 700 |

**Step-by-step:** `CEIL(0.9 * 12) = CEIL(10.8) = 11`. Row 11 is **700**.

> **Why report medians and percentiles?** Means are distorted by outliers. For revenue, latency, and session length, the **median** (typical) and **p90/p95/p99** (tail) are the standard metrics. PostgreSQL offers `PERCENTILE_CONT(0.9) WITHIN GROUP (ORDER BY amount)`, which interpolates (for these 12 values it returns 685, between the 10th and 11th values). Different methods give slightly different percentiles: say which one you use.

---

## P52: Pivot and Unpivot

### Pivot: rows to columns (conditional aggregation)

**Customer revenue by month:**

```sql
SELECT customer_id,
       SUM(CASE WHEN MONTH(order_date) = 1 THEN amount ELSE 0 END) AS jan,
       SUM(CASE WHEN MONTH(order_date) = 2 THEN amount ELSE 0 END) AS feb,
       SUM(CASE WHEN MONTH(order_date) = 3 THEN amount ELSE 0 END) AS mar,
       SUM(CASE WHEN MONTH(order_date) = 4 THEN amount ELSE 0 END) AS apr,
       SUM(amount)                                                 AS total
FROM orders
GROUP BY customer_id
ORDER BY customer_id;
```

| customer_id | jan | feb | mar | apr | total |
|---|---|---|---|---|---|
| 1 | 500 | 700 | 400 | 600 | 2200 |
| 2 | 300 | 200 | 0 | 0 | 500 |
| 3 | 450 | 0 | 350 | 0 | 800 |
| 4 | 0 | 650 | 150 | 0 | 800 |
| 5 | 0 | 0 | 800 | 0 | 800 |
| 6 | 0 | 0 | 0 | 250 | 250 |

**Check:** column totals: jan = 1250, feb = 1550, mar = 1700, apr = 850 (matches P34). Grand total = 5350.

**How it works:** each `CASE` returns the amount only for its month (else 0), and `SUM` adds those up per customer. The column list is **fixed in the query**: SQL cannot create columns dynamically from data without dynamic SQL (or a `PIVOT` keyword in SQL Server / Oracle / Snowflake).

> For a pivot with **counts**, use `COUNT(CASE WHEN ... THEN 1 END)` or `SUM(CASE WHEN ... THEN 1 ELSE 0 END)`.

### Unpivot: columns to rows (UNION ALL)

`quarterly_wide`:

| product | q1 | q2 | q3 | q4 |
|---|---|---|---|---|
| A | 100 | 120 | 90 | 150 |
| B | 80 | 60 | 110 | 70 |

```sql
SELECT product, 'Q1' AS quarter, q1 AS sales FROM quarterly_wide
UNION ALL
SELECT product, 'Q2', q2 FROM quarterly_wide
UNION ALL
SELECT product, 'Q3', q3 FROM quarterly_wide
UNION ALL
SELECT product, 'Q4', q4 FROM quarterly_wide
ORDER BY product, quarter;
```

| product | quarter | sales |
|---|---|---|
| A | Q1 | 100 |
| A | Q2 | 120 |
| A | Q3 | 90 |
| A | Q4 | 150 |
| B | Q1 | 80 |
| B | Q2 | 60 |
| B | Q3 | 110 |
| B | Q4 | 70 |

**Explanation:** each `SELECT` pulls one column and labels it; `UNION ALL` stacks them (use `UNION ALL`, not `UNION`, so identical rows are not removed). Alternatives: PostgreSQL `CROSS JOIN LATERAL (VALUES ('Q1', q1), ...)`, SQL Server/Oracle `UNPIVOT`.

> **Why unpivot?** Long (tidy) data is much easier to filter, group, and chart than wide data with one column per period.

---

# PART 4: Wrap-Up

## 4.1 Metric Definition Crib Sheet

| Metric | Definition | Watch out for |
|---|---|---|
| DAU / WAU / MAU | distinct active users per day / 7 days / 30 days (or month) | define "active" (any event? purchase?) |
| Stickiness | DAU / MAU | same date range for both |
| Day-N retention | users active exactly N days after signup / signups | exact day vs "on or after" |
| Cohort retention | active in month N / cohort size | fill missing months; "not yet observed" |
| Churn | active in period P but not in P+1 / active in P | define inactivity window |
| Activation | users reaching a key action within X days / signups | choose the action and window |
| Conversion rate | conversions / (visitors or sessions or users) | pick one denominator |
| CTR | clicks / impressions | `NULLIF` for zero impressions |
| AOV | revenue / orders | not revenue / customers |
| ARPU | revenue / users | include inactive users? |
| CLV (historical) | total revenue per customer to date | predictive CLV is a model, not plain SQL |
| Repeat rate | customers with 2 or more orders / customers | time window |
| MoM / YoY growth | (current - previous) / previous | previous = 0, missing periods |
| Median / p90 | middle / 90th percentile value | method (nearest-rank vs interpolated) |
| Funnel conversion | users at step k / users at step 1 (or k-1) | strict order, repeated events |
| Support SLA | tickets answered within X / all tickets | unanswered tickets count as breach? |

## 4.2 Top Mistakes in Time and Analytics Queries

| Mistake | Fix |
|---|---|
| `BETWEEN` on timestamps with end date (`BETWEEN '2026-10-01' AND '2026-10-31'`) | use `>= start AND < next_start` |
| Grouping by month name or text and sorting alphabetically | bucket to a date (month start) |
| `LAG` across missing months | fill the calendar first |
| Integer division in percentages | multiply by `100.0` and use `ROUND` |
| Dividing by zero | `NULLIF(denominator, 0)` |
| `COUNT(*)` instead of `COUNT(DISTINCT user_id)` for users | choose carefully |
| Dropping users who never returned (inner join) | `LEFT JOIN` to keep the denominator |
| Counting a user multiple times in retention | `MAX(CASE ...)` per user first |
| Ignoring time zones at day boundaries | state the zone; convert before truncating |
| Mixing week definitions (Sunday vs Monday) | state it; use one convention |
| Overlap test with wrong inequality direction | `A.start <= B.end AND A.end >= B.start` |
| Merging intervals using only the previous row | use running `MAX(end)` over all earlier rows |
| Percentile method differences | state nearest-rank vs interpolation |
| Reporting the NULL of "not observable yet" as 0 | compare with the last data date |
| Using `UNION` instead of `UNION ALL` in unpivot | `UNION ALL` |

## 4.3 Phase 5 Cheat Sheet

```text
BUCKETING   month_start : DATE_SUB(d, INTERVAL DAYOFMONTH(d)-1 DAY)   | PG: DATE_TRUNC('month', d)
            week_start  : DATE_SUB(d, INTERVAL WEEKDAY(d) DAY)         | PG: DATE_TRUNC('week', d)
RANGE RULE  ts >= start AND ts < next_start   (half-open)

GROWTH      (cur - prev) / NULLIF(prev,0) * 100 with LAG()
            YoY: LAG(rev) OVER (PARTITION BY MONTH(m) ORDER BY m)

COHORT      cohort = MIN(active_month) per user
            month_number = TIMESTAMPDIFF(MONTH, cohort_month, active_month)
            retention = active_users / FIRST_VALUE(active_users)

DAY-N RET   MAX(CASE WHEN DATEDIFF(activity, signup) = N THEN 1 ELSE 0 END) per user

CHURN       self-join customer-month to customer-next-month, LEFT JOIN, count NULLs
FUNNEL      distinct users per step, LAG for step-to-step, FIRST_VALUE for overall
ACTIVE USERS calendar CTE + range join (COUNT DISTINCT not allowed in windows)
FILL DATES  calendar CTE LEFT JOIN + COALESCE
FORWARD FILL COUNT(value) OVER (ORDER BY d) as group id, MAX(value) OVER (PARTITION BY grp)
OVERLAP     A.start <= B.end AND A.end >= B.start  and  A.id < B.id
MERGE       running MAX(end) of previous rows -> new_block flag -> running SUM
TIME DIFF   TIMESTAMPDIFF(MINUTE, a, b)  | PG: EXTRACT(EPOCH FROM (b-a))/60
BIZ DAYS    calendar CTE, WEEKDAY(d) < 5, NOT EXISTS holiday
GAP TOLERANT streak: new flag when DATEDIFF(login, prev) > tolerance

AOV         SUM(amount)/COUNT(*)          REV/CUSTOMER  SUM(amount)/COUNT(DISTINCT customer)
NEW/RETURN  MIN(order_date) OVER (PARTITION BY customer) vs order month
CTR / CONV  SUM(CASE ...)/NULLIF(SUM(CASE ...),0)
TOP N       RANK / DENSE_RANK / ROW_NUMBER   (decide tie behavior)
PARETO      running SUM(revenue) / SUM(revenue) OVER ()
BASKET      self-join on order_id with a.product < b.product
MEDIAN      ROW_NUMBER + COUNT, rows FLOOR((n+1)/2.0) and CEIL((n+1)/2.0)
PIVOT       SUM(CASE WHEN key = x THEN val ELSE 0 END)
UNPIVOT     SELECT ... UNION ALL SELECT ...
```

## 4.4 Practice Problems

1. Monthly orders, distinct customers and revenue (all months).
2. Weekly (Monday-start) revenue for **February** orders.
3. Month-over-month revenue growth % (with the previous month's revenue).
4. Overall **Day-1 retention** (a single percentage) for users and activity.
5. How many customers churned between **March and April**, and which ones?
6. Funnel: for each step, users and % of the previous step.
7. Which rooms have at least one overlapping pair of bookings?
8. Number of business days between 2026-10-19 and 2026-10-30, excluding weekends and the holidays table. (Expected: 8)
9. Revenue split between weekdays and weekends.
10. The repeat purchase rate.
11. The top 3 customers by revenue **including ties** (`RANK`).
12. The median order amount.
13. Forward-fill `price_log` for 2026-10-01 to 2026-10-07.
14. New vs returning customers per month.
15. **Challenge:** customers who ordered in **3 or more consecutive months**.

<details>
<summary>Answers</summary>

```sql
-- 1  2026-01: 3 orders, 3 customers, 1250 | 02: 3, 3, 1550 | 03: 4, 4, 1700 | 04: 2, 2, 850
SELECT DATE_FORMAT(order_date, '%Y-%m') AS month,
       COUNT(*) AS orders, COUNT(DISTINCT customer_id) AS customers, SUM(amount) AS revenue
FROM orders
GROUP BY DATE_FORMAT(order_date, '%Y-%m')
ORDER BY month;

-- 2  February orders: 02-03 (Tue), 02-14 (Sat), 02-18 (Wed)
--    week_start 2026-02-02: 700 ; 2026-02-09: 200 (02-14 is Sat) ; 2026-02-16: 650
SELECT DATE_SUB(order_date, INTERVAL WEEKDAY(order_date) DAY) AS week_start,
       SUM(amount) AS revenue
FROM orders
WHERE order_date >= '2026-02-01' AND order_date < '2026-03-01'
GROUP BY DATE_SUB(order_date, INTERVAL WEEKDAY(order_date) DAY)
ORDER BY week_start;

-- 3  see P35 (NULL, 24.00, 9.68, -50.00)

-- 4  Day-1 retention = 3 / 5 = 60.00
SELECT ROUND(100.0 * COUNT(DISTINCT CASE WHEN DATEDIFF(a.activity_date, u.signup_date) = 1
                                         THEN u.user_id END) / COUNT(DISTINCT u.user_id), 2) AS d1_retention_pct
FROM users u
LEFT JOIN activity a ON a.user_id = u.user_id;

-- 5  customers 3, 4, 5 (3 churned)
WITH cm AS (
  SELECT DISTINCT customer_id,
         DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS m
  FROM orders)
SELECT a.customer_id
FROM cm a
LEFT JOIN cm b ON b.customer_id = a.customer_id
              AND b.m = DATE_ADD(a.m, INTERVAL 1 MONTH)
WHERE a.m = '2026-03-01' AND b.customer_id IS NULL
ORDER BY a.customer_id;

-- 6  see P38 (view 6, cart 5 = 83.33, checkout 3 = 60.00, purchase 2 = 66.67)

-- 7  rooms 101 and 102
SELECT DISTINCT a.room_id
FROM bookings a
JOIN bookings b ON a.room_id = b.room_id
               AND a.booking_id < b.booking_id
               AND a.start_date <= b.end_date
               AND a.end_date   >= b.start_date
ORDER BY a.room_id;

-- 8  see P43 (8)

-- 9  Weekday 8 orders / 4250 ; Weekend 4 orders / 1100  (see P43)

-- 10 66.67  (see P45)

-- 11 customers 1, 3, 4, 5
WITH cust AS (SELECT customer_id, SUM(amount) AS revenue FROM orders GROUP BY customer_id),
r AS (SELECT customer_id, revenue, RANK() OVER (ORDER BY revenue DESC) AS rnk FROM cust)
SELECT customer_id, revenue, rnk FROM r WHERE rnk <= 3 ORDER BY rnk, customer_id;

-- 12 425.00  (see P51)

-- 13 50, 50, 50, 55, 55, 53, 53  (see P40 B)

-- 14 see P46

-- 15 Customer 1 only (ordered Jan, Feb, Mar, Apr)
WITH cm AS (
  SELECT DISTINCT customer_id,
         DATE_SUB(order_date, INTERVAL DAYOFMONTH(order_date)-1 DAY) AS m
  FROM orders),
g AS (
  SELECT customer_id,
         TIMESTAMPDIFF(MONTH, '2026-01-01', m)
           - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY m) AS grp
  FROM cm)
SELECT DISTINCT customer_id
FROM g
GROUP BY customer_id, grp
HAVING COUNT(*) >= 3;
-- Month index minus row number stays constant inside a run of consecutive months (gaps and islands, Phase 4).
```

Check for problem 2: 2026-02-03 is a Tuesday (week starts Mon 02-02), 2026-02-14 is a Saturday (week starts Mon 02-09), 2026-02-18 is a Wednesday (week starts Mon 02-16).
</details>

## 4.5 Self-Test Questions

1. Why should timestamp ranges be half-open (`>= start AND < next_start`)?
2. How do you bucket dates by month so that sorting and joins stay correct?
3. Why can `LAG` give a wrong month-over-month result, and how do you fix it?
4. Define Day-1 retention precisely. Why do you need a `LEFT JOIN`?
5. How do you build a cohort retention table? What does "month number" mean?
6. How do you compute churn using a self-join? Why filter out the last month?
7. Why can't you use `COUNT(DISTINCT user_id) OVER (...)` for rolling active users, and what is the alternative?
8. Write the overlap condition for two date ranges. How do you avoid duplicate pairs?
9. Why does merging intervals need a running `MAX(end_date)` instead of `LAG(end_date)`?
10. How do you forward-fill missing values without `IGNORE NULLS`?
11. What is the difference between AOV and revenue per customer?
12. Explain `RANK` vs `DENSE_RANK` vs `ROW_NUMBER` for "top 3 customers."
13. How do you compute a median without `PERCENTILE_CONT`? How does nearest-rank p90 differ from interpolated p90?
14. How would you pivot rows to columns, and what is the limitation?
15. How does a time zone change the answer to "daily active users"?

## 4.6 LeetCode / Practice Checklist

| Level | Problems |
|---|---|
| Medium | Game Play Analysis IV (Day-1 style retention), Monthly Transactions I, Trips and Users, User Activity for the Past 30 Days I, Restaurant Growth (7-day moving average), Active Users, Report Contiguous Dates |
| Hard | Game Play Analysis V (retention by install date), Monthly Transactions II, Human Traffic of Stadium |

Also practice on **StrataScratch** and **DataLemur** with tags like "retention", "cohort", "funnel", "DAU", "growth", "churn".

## 4.7 What to Do Next

1. Create all tables from Part 1 and run **every** query. Compare your output with the tables in this guide.
2. For each pattern, **say the metric definition out loud first**, then write the query from memory.
3. Re-solve the cohort retention query, the interval-merge query, and the funnel query **without looking**: these are the three most common "hard" analytics questions.
4. Solve 25 to 30 problems tagged retention, funnel, growth, and date logic.
5. When you are ready, say **"Start Phase 6"** for **Advanced Techniques** (lateral joins, JSON and array functions, regex parsing, UPDATE/DELETE/MERGE problems, relational division, graph problems in SQL, tournament and scoreboard problems). If you want the earlier gap filled first, say **"Start Phase 2"** for **Aggregation, Joins and Set Operations**.
