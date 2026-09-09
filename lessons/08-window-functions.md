# Lesson 08 — Window Functions

## Goal
Window functions are the most powerful feature in SQL for analytics.
They let you compute aggregates across related rows WITHOUT collapsing them
into a single result — you keep every row AND get the aggregated context.

## Prerequisites
- [Lesson 07](07-subqueries-and-ctes.md) — subqueries & CTEs

## After This Lesson You Will Be Able To
- Understand the OVER() clause and window concept
- Use PARTITION BY and ORDER BY inside windows
- Use ROW_NUMBER, RANK, DENSE_RANK
- Use LAG and LEAD for comparing adjacent rows
- Use SUM/AVG as running totals
- Use FIRST_VALUE, LAST_VALUE, NTH_VALUE
- Use NTILE for percentile bucketing

---

## The Window Function Concept

Normal aggregation (GROUP BY):
```
Input:  [10, 20, 30, 40]
Output: [25]           ← 4 rows collapse to 1 (the average)
```

Window function:
```
Input:  [10, 20, 30, 40]
Output: [10/25, 20/25, 30/25, 40/25]  ← each row KEEPS its value + sees the aggregate
```

This is the key insight: **window functions don't reduce rows**. Every row in
the result still corresponds to one input row. The function just gives each
row access to calculations across a "window" of related rows.

---

## Basic Syntax

```sql
function_name(expression) OVER (
    [PARTITION BY column1, column2, ...]  -- define groups
    [ORDER BY column3 [ASC|DESC], ...]    -- define order within group
    [frame_clause]                         -- define row range
)
```

The OVER() clause defines the "window" — which rows each row can "see".

---

## Numbering Functions

### ROW_NUMBER — Unique sequential integer

```sql
-- Number every employee
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS overall_rank
FROM employees;

-- Number within each department (restart at 1 for each dept)
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;

-- Get the top earner in each department (using subquery/CTE)
WITH ranked AS (
    SELECT
        name, department, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE rn = 1;

-- Get the 2nd highest earner per department
WITH ranked AS (
    SELECT
        name, department, salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT name, department, salary
FROM ranked
WHERE rn = 2;
```

**Output of the first query** (`ROW_NUMBER() OVER (ORDER BY salary DESC)`):

| name | department | salary | overall_rank |
|---|---|---|---|
| Sarah Connor | Engineering | 120000 | 1 |
| John Smith | Engineering | 95000 | 2 |
| Jane Doe | Engineering | 90000 | 3 |
| Mike Johnson | Sales | 75000 | 4 |
| Tom Brown | Sales | 72000 | 5 |

**Output of the second query** (`PARTITION BY department` — numbering restarts at 1 per department):

| name | department | salary | dept_rank |
|---|---|---|---|
| Sarah Connor | Engineering | 120000 | 1 |
| John Smith | Engineering | 95000 | 2 |
| Jane Doe | Engineering | 90000 | 3 |
| Mike Johnson | Sales | 75000 | 1 |
| Tom Brown | Sales | 72000 | 2 |

Notice `dept_rank` restarts at `1` for Sales even though Mike Johnson's
overall rank was `4` — that's the entire point of `PARTITION BY`: each
department gets its own independent numbering.

**Output of the "top earner per department" query:**

| name | department | salary |
|---|---|---|
| Sarah Connor | Engineering | 120000 |
| Mike Johnson | Sales | 75000 |

Only the `rn = 1` row from each partition survives the `WHERE` filter.

### RANK — Same rank for ties, gaps after ties

```sql
SELECT
    name,
    salary,
    RANK() OVER (ORDER BY salary DESC) AS rank
FROM employees;

-- Example output:
-- Sarah Connor  | 120000 | 1
-- John Smith    |  95000 | 2
-- Jane Doe      |  90000 | 3
-- Tom Brown     |  72000 | 4   ← same salary as Oscar
-- Oscar Rivera  |  70000 | 4   ← tied at rank 4
-- Mike Johnson  |  75000 | 6   ← gap! (no rank 5)

-- RANK with PARTITION BY
SELECT
    name, department, salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank
FROM employees;
```

**Output:**

| name | department | salary | dept_rank |
|---|---|---|---|
| Sarah Connor | Engineering | 120000 | 1 |
| John Smith | Engineering | 95000 | 2 |
| Jane Doe | Engineering | 90000 | 3 |
| Mike Johnson | Sales | 75000 | 1 |
| Tom Brown | Sales | 72000 | 2 |
| Oscar Rivera | Sales | 72000 | 2 |

Tom Brown and Oscar Rivera tie at `72000` within Sales, so both get
`dept_rank = 2` — same tie behavior as the overall example above, just
restarted per department.

### DENSE_RANK — Same rank for ties, NO gaps

```sql
SELECT
    name,
    salary,
    RANK()       OVER (ORDER BY salary DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank
FROM employees;

-- RANK:       1, 2, 3, 4, 4, 6   (gap after tie)
-- DENSE_RANK: 1, 2, 3, 4, 4, 5   (no gap)

-- Use DENSE_RANK when: "top 3 salary levels" (not "top 3 individuals")
SELECT DISTINCT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS level
FROM employees
WHERE dense_rank <= 3;
```

**Output of the combined `RANK`/`DENSE_RANK` query:**

| name | salary | rank | dense_rank |
|---|---|---|---|
| Sarah Connor | 120000 | 1 | 1 |
| John Smith | 95000 | 2 | 2 |
| Jane Doe | 90000 | 3 | 3 |
| Mike Johnson | 75000 | 4 | 4 |
| Tom Brown | 72000 | 5 | 5 |
| Oscar Rivera | 72000 | 5 | 5 |
| Lisa Chen | 68000 | 7 | 6 |

At the tie (72000), both columns agree: `5`. **After** the tie is where
they diverge — `RANK` jumps to `7` (skipping `6`, because two rows
already claimed `5`), while `DENSE_RANK` continues at `6`, treating the
tie as occupying only one "level."

---

## LAG and LEAD — Compare to Adjacent Rows

### LAG — Look at the previous row

```sql
-- Compare each month's revenue to the previous month
WITH monthly_revenue AS (
    SELECT
        DATE_TRUNC('month', order_date) AS month,
        SUM(total) AS revenue
    FROM orders
    WHERE status = 'delivered'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS change,
    ROUND(
        (revenue - LAG(revenue) OVER (ORDER BY month))
        / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100,
        1
    ) AS pct_change
FROM monthly_revenue
ORDER BY month;

-- LAG syntax: LAG(column, offset, default)
LAG(salary, 1, 0) OVER (ORDER BY hire_date)  -- previous row's salary, default 0
LAG(salary, 2)    OVER (ORDER BY hire_date)  -- 2 rows back
```

**Output:**

| month | revenue | prev_month_revenue | change | pct_change |
|---|---|---|---|---|
| 2024-01-01 | 1200.00 | NULL | NULL | NULL |
| 2024-02-01 | 900.00 | 1200.00 | -300.00 | -25.0 |
| 2024-03-01 | 1500.00 | 900.00 | 600.00 | 66.7 |

The very first row has no "previous month," so `LAG` returns `NULL` —
and everything computed from it (`change`, `pct_change`) becomes `NULL`
too. This is exactly why the syntax supports a third argument
(`LAG(revenue, 1, 0)`) — passing a default instead of letting the first
row fall back to `NULL`.

### LEAD — Look at the next row

```sql
-- Each order and the next order by the same customer
SELECT
    customer_id,
    order_date,
    total,
    LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_date,
    LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date)
        - order_date AS days_to_next_order
FROM orders
ORDER BY customer_id, order_date;

-- LEAD syntax: LEAD(column, offset, default)
LEAD(order_date, 1, NULL) OVER (PARTITION BY customer_id ORDER BY order_date)
```

**Output:**

| customer_id | order_date | total | next_order_date | days_to_next_order |
|---|---|---|---|---|
| 1 | 2024-01-05 | 999.99 | 2024-02-10 | 36 |
| 1 | 2024-02-10 | 49.99 | NULL | NULL |
| 2 | 2024-01-20 | 129.99 | 2024-03-01 | 41 |
| 2 | 2024-03-01 | 89.99 | NULL | NULL |

`LEAD` looks *forward* instead of backward — each customer's **last**
order (per `PARTITION BY customer_id`) has no "next order" yet, so
`next_order_date` is `NULL` there, mirroring how `LAG`'s *first* row was
`NULL` above.

---

## Running Aggregates — SUM, AVG, COUNT as Windows

When you add ORDER BY to an aggregate window function, it computes a RUNNING aggregate.

```sql
-- Running total revenue (cumulative sum)
SELECT
    order_date,
    total,
    SUM(total) OVER (ORDER BY order_date) AS running_total
FROM orders
WHERE status = 'delivered'
ORDER BY order_date;

-- Running total per customer
SELECT
    customer_id,
    order_date,
    total,
    SUM(total) OVER (
        PARTITION BY customer_id
        ORDER BY order_date
    ) AS customer_running_total
FROM orders
ORDER BY customer_id, order_date;

-- Running average
SELECT
    order_date,
    total,
    AVG(total) OVER (ORDER BY order_date) AS running_avg
FROM orders
ORDER BY order_date;

-- Running count
SELECT
    order_date,
    total,
    COUNT(*) OVER (ORDER BY order_date) AS orders_so_far
FROM orders
ORDER BY order_date;

-- Window aggregate without ORDER BY = same as GROUP BY result but without collapsing rows
SELECT
    name,
    salary,
    department,
    AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary,
    salary - AVG(salary) OVER (PARTITION BY department) AS vs_dept_avg
FROM employees;
```

**Output of the first running-total query** (`SUM ... OVER (ORDER BY order_date)`):

| order_date | total | running_total |
|---|---|---|
| 2024-01-05 | 999.99 | 999.99 |
| 2024-01-20 | 129.99 | 1129.98 |
| 2024-02-10 | 49.99 | 1179.97 |
| 2024-03-01 | 89.99 | 1269.96 |

Every row still exists (unlike `GROUP BY`, which would collapse this
into one total) — each row just additionally shows the sum of itself
plus every row before it.

**Output of the per-customer running total** (`PARTITION BY customer_id`):

| customer_id | order_date | total | customer_running_total |
|---|---|---|---|
| 1 | 2024-01-05 | 999.99 | 999.99 |
| 1 | 2024-02-10 | 49.99 | 1049.98 |
| 2 | 2024-01-20 | 129.99 | 129.99 |
| 2 | 2024-03-01 | 89.99 | 219.98 |

Notice the running total **resets per customer** — customer 2's total
starts fresh at `129.99`, unaffected by customer 1's running total. This
is `PARTITION BY` doing the same "restart per group" behavior seen with
`ROW_NUMBER` earlier, just applied to a running sum instead of a count.

**Output of the department-average query** (no `ORDER BY` inside `OVER`
— every row in a partition gets the *same* value, since there's no
running/cumulative behavior without `ORDER BY`):

| name | salary | department | dept_avg_salary | vs_dept_avg |
|---|---|---|---|---|
| Sarah Connor | 120000 | Engineering | 101000.00 | 19000.00 |
| John Smith | 95000 | Engineering | 101000.00 | -6000.00 |
| Jane Doe | 90000 | Engineering | 101000.00 | -11000.00 |
| Mike Johnson | 75000 | Sales | 71666.67 | 3333.33 |
| Tom Brown | 72000 | Sales | 71666.67 | 333.33 |

Every Engineering row shows the *same* `dept_avg_salary` — this is the
"GROUP BY without collapsing" behavior called out right above the query.

---

## The Frame Clause — Defining Row Range

The FRAME clause controls which rows within the window partition are included.

```sql
-- Default (when ORDER BY is present): RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- Default (no ORDER BY): ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- Frame clause syntax:
ROWS BETWEEN start AND end
RANGE BETWEEN start AND end

-- Start/end options:
UNBOUNDED PRECEDING  -- first row of partition
n PRECEDING          -- n rows before current row
CURRENT ROW          -- current row
n FOLLOWING          -- n rows after current row
UNBOUNDED FOLLOWING  -- last row of partition
```

```sql
-- 3-day moving average of daily revenue
WITH daily AS (
    SELECT order_date, SUM(total) AS revenue
    FROM orders
    WHERE status = 'delivered'
    GROUP BY order_date
)
SELECT
    order_date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY order_date
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW  -- current + 2 before = 3 rows
    ) AS moving_avg_3day
FROM daily
ORDER BY order_date;
```

**Output:**

| order_date | revenue | moving_avg_3day |
|---|---|---|
| 2024-01-01 | 300.00 | 300.00 |
| 2024-01-02 | 600.00 | 450.00 |
| 2024-01-03 | 900.00 | 600.00 |
| 2024-01-04 | 300.00 | 600.00 |

Row 1 has no prior rows, so its "3-day window" only contains itself
(`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` simply can't go back further
than the partition's start). Row 4's average `600.00` comes from
`(900 + 300 + 300) / 3` — the current row plus the 2 immediately before
it, not all 4 rows.

```sql
-- 7-day rolling sum
SELECT
    order_date,
    SUM(total) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS rolling_7day
FROM orders
ORDER BY order_date;

-- Entire partition (same as no ORDER BY)
SUM(salary) OVER (PARTITION BY department ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- From start of partition to current row
SUM(salary) OVER (PARTITION BY department ORDER BY salary ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
```

**Output of the 7-day rolling sum**, using the same 4-order sample data
from earlier in this lesson (note: with only 4 orders spanning under 7
days, every row's window still covers all rows up to itself — the cap
of "6 rows back" never actually gets reached here, but the mechanism is
identical to the 3-day moving average above, just with a wider frame):

| order_date | total | rolling_7day |
|---|---|---|
| 2024-01-05 | 999.99 | 999.99 |
| 2024-01-20 | 129.99 | 1129.98 |
| 2024-02-10 | 49.99 | 1179.97 |
| 2024-03-01 | 89.99 | 1269.96 |

---

## FIRST_VALUE, LAST_VALUE, NTH_VALUE

```sql
-- Compare each employee's salary to the highest in their department
SELECT
    name,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS dept_highest_salary,
    FIRST_VALUE(name) OVER (
        PARTITION BY department
        ORDER BY salary DESC
    ) AS top_earner_in_dept
FROM employees;

-- LAST_VALUE requires explicit frame to work correctly
SELECT
    name,
    department,
    salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING  -- REQUIRED
    ) AS dept_lowest_salary
FROM employees;

-- NTH_VALUE: the value at position N in the window
SELECT
    name,
    department,
    salary,
    NTH_VALUE(salary, 2) OVER (
        PARTITION BY department
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS second_highest_in_dept
FROM employees;
```

**Output of the `FIRST_VALUE` query:**

| name | department | salary | dept_highest_salary | top_earner_in_dept |
|---|---|---|---|---|
| Sarah Connor | Engineering | 120000 | 120000 | Sarah Connor |
| John Smith | Engineering | 95000 | 120000 | Sarah Connor |
| Jane Doe | Engineering | 90000 | 120000 | Sarah Connor |
| Mike Johnson | Sales | 75000 | 75000 | Mike Johnson |
| Tom Brown | Sales | 72000 | 75000 | Mike Johnson |

Every row in a department sees the *same* `dept_highest_salary` —
`FIRST_VALUE` always looks at the first row of the ordered window
(here, the highest earner), regardless of which row is currently being
evaluated.

**Output of the `LAST_VALUE` query** — this is exactly why the lesson
flags the explicit frame as **REQUIRED**: without it, `LAST_VALUE`'s
default frame only extends to `CURRENT ROW`, so it would just return
each row's own salary instead of the department's actual lowest. With
the frame forced to the full partition (`UNBOUNDED PRECEDING AND
UNBOUNDED FOLLOWING`), every row correctly sees the true minimum:

| name | department | salary | dept_lowest_salary |
|---|---|---|---|
| Sarah Connor | Engineering | 120000 | 90000 |
| John Smith | Engineering | 95000 | 90000 |
| Jane Doe | Engineering | 90000 | 90000 |
| Mike Johnson | Sales | 75000 | 72000 |
| Tom Brown | Sales | 72000 | 72000 |

**Output of the `NTH_VALUE(salary, 2)` query** (2nd-highest salary per
department):

| name | department | salary | second_highest_in_dept |
|---|---|---|---|
| Sarah Connor | Engineering | 120000 | 95000 |
| John Smith | Engineering | 95000 | 95000 |
| Jane Doe | Engineering | 90000 | 95000 |
| Mike Johnson | Sales | 75000 | 72000 |
| Tom Brown | Sales | 72000 | 72000 |

---

## NTILE — Percentile Bucketing

NTILE(n) divides rows into n equal-sized buckets.

```sql
-- Divide products into 4 price quartiles
SELECT
    name,
    price,
    NTILE(4) OVER (ORDER BY price) AS price_quartile
FROM products;
-- Quartile 1: cheapest 25%, Quartile 4: most expensive 25%

-- Divide employees into salary percentiles
SELECT
    name,
    salary,
    NTILE(10) OVER (ORDER BY salary) AS salary_decile
FROM employees;
-- Decile 10 = top 10% earners

-- Label with bucket names
SELECT
    name,
    price,
    CASE NTILE(4) OVER (ORDER BY price)
        WHEN 1 THEN 'Budget'
        WHEN 2 THEN 'Economy'
        WHEN 3 THEN 'Premium'
        WHEN 4 THEN 'Luxury'
    END AS price_tier
FROM products;
```

**Output of the price-quartile query**, on 8 sample products sorted
cheapest to most expensive:

| name | price | price_quartile |
|---|---|---|
| USB Cable | 8.99 | 1 |
| Mouse | 24.99 | 1 |
| Keyboard | 49.99 | 2 |
| Headphones | 79.99 | 2 |
| Webcam | 129.99 | 3 |
| Monitor | 249.99 | 3 |
| Tablet | 399.99 | 4 |
| Laptop | 999.99 | 4 |

`NTILE(4)` splits the 8 rows into 4 groups of 2 — quartile 1 gets the 2
cheapest, quartile 4 gets the 2 most expensive. With a row count that
isn't evenly divisible by `n`, the extra rows get distributed to the
earliest buckets first (e.g., `NTILE(3)` on these 8 rows would produce
buckets of size 3, 3, 2, not 3 equal groups).

---

## PERCENT_RANK and CUME_DIST

```sql
-- PERCENT_RANK: relative rank as 0.0 to 1.0
-- (rank - 1) / (total rows - 1)
SELECT
    name,
    salary,
    ROUND(PERCENT_RANK() OVER (ORDER BY salary) * 100, 1) AS percentile
FROM employees;
-- Sarah Connor (highest) = 100.0
-- The lowest earner = 0.0

-- CUME_DIST: cumulative distribution (fraction of rows <= current row)
SELECT
    name,
    salary,
    ROUND(CUME_DIST() OVER (ORDER BY salary) * 100, 1) AS cumulative_pct
FROM employees;
-- Tells you: "X% of employees earn this salary or less"

-- Find top 25% earners
SELECT name, salary
FROM (
    SELECT
        name, salary,
        PERCENT_RANK() OVER (ORDER BY salary) AS pr
    FROM employees
) t
WHERE pr >= 0.75;
```

**Output of the `PERCENT_RANK` and `CUME_DIST` queries**, on 5 sorted
salaries (68000, 72000, 75000, 90000, 120000):

| name | salary | percentile (PERCENT_RANK) | cumulative_pct (CUME_DIST) |
|---|---|---|---|
| Lisa Chen | 68000 | 0.0 | 20.0 |
| Tom Brown | 72000 | 25.0 | 40.0 |
| Mike Johnson | 75000 | 50.0 | 60.0 |
| Jane Doe | 90000 | 75.0 | 80.0 |
| Sarah Connor | 120000 | 100.0 | 100.0 |

**The distinction between the two, made concrete:** `PERCENT_RANK` for
Mike Johnson is `50.0` — meaning "exactly halfway through the ranking
positions" `((3-1)/(5-1))`. `CUME_DIST` for the same row is `60.0` —
meaning "60% of all rows earn this salary or less" (3 out of 5 rows:
Lisa, Tom, and Mike himself). They answer subtly different questions:
*rank position* vs. *share of the data at or below this point*.

---

## WINDOW Clause — Reuse Window Definitions

```sql
-- Instead of repeating the window definition:
SELECT
    name,
    salary,
    ROW_NUMBER() OVER w AS row_num,
    RANK()       OVER w AS rank,
    DENSE_RANK() OVER w AS dense_rank,
    PERCENT_RANK() OVER w AS pct_rank
FROM employees
WINDOW w AS (ORDER BY salary DESC);

-- Multiple named windows
SELECT
    name,
    department,
    salary,
    ROW_NUMBER() OVER dept_win AS dept_row,
    AVG(salary)  OVER dept_win AS dept_avg,
    ROW_NUMBER() OVER overall_win AS overall_row
FROM employees
WINDOW
    dept_win    AS (PARTITION BY department ORDER BY salary DESC),
    overall_win AS (ORDER BY salary DESC);
```

**Output of the first query** — this produces the *exact same result*
as writing `OVER (ORDER BY salary DESC)` four separate times; `WINDOW w
AS (...)` just lets you define it once and reference it by name:

| name | salary | row_num | rank | dense_rank | pct_rank |
|---|---|---|---|---|---|
| Sarah Connor | 120000 | 1 | 1 | 1 | 0.0 |
| John Smith | 95000 | 2 | 2 | 2 | 25.0 |
| Jane Doe | 90000 | 3 | 3 | 3 | 50.0 |
| Mike Johnson | 75000 | 4 | 4 | 4 | 75.0 |
| Tom Brown | 72000 | 5 | 5 | 5 | 100.0 |

---

## Complete Real-World Examples

**Example 1 — Customer purchase sequence with gap analysis:**
```sql
SELECT
    customer_id,
    order_date,
    total,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_num,
    LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order,
    order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS days_since_last,
    SUM(total) OVER (PARTITION BY customer_id ORDER BY order_date) AS cumulative_spent
FROM orders
ORDER BY customer_id, order_date;
```

**Output**, combining everything this lesson covered into one query:

| customer_id | order_date | total | order_num | prev_order | days_since_last | cumulative_spent |
|---|---|---|---|---|---|---|
| 1 | 2024-01-05 | 999.99 | 1 | NULL | NULL | 999.99 |
| 1 | 2024-02-10 | 49.99 | 2 | 2024-01-05 | 36 | 1049.98 |
| 2 | 2024-01-20 | 129.99 | 1 | NULL | NULL | 129.99 |
| 2 | 2024-03-01 | 89.99 | 2 | 2024-01-20 | 41 | 219.98 |

Every one of `order_num`, `prev_order`, `days_since_last`, and
`cumulative_spent` restarts per customer — four different window
functions, all sharing the same `PARTITION BY customer_id`.

**Example 2 — Product sales ranking per category:**
```sql
WITH product_sales AS (
    SELECT
        p.id,
        p.name,
        p.category,
        p.price,
        COALESCE(SUM(oi.quantity), 0) AS units_sold,
        COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS revenue
    FROM products p
    LEFT JOIN order_items oi ON oi.product_id = p.id
    GROUP BY p.id, p.name, p.category, p.price
)
SELECT
    name,
    category,
    units_sold,
    revenue,
    RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS revenue_rank_in_category,
    RANK() OVER (ORDER BY revenue DESC) AS overall_revenue_rank,
    ROUND(revenue / SUM(revenue) OVER (PARTITION BY category) * 100, 1) AS pct_of_category_revenue
FROM product_sales
ORDER BY category, revenue_rank_in_category;
```

**Output:**

| name | category | units_sold | revenue | revenue_rank_in_category | overall_revenue_rank | pct_of_category_revenue |
|---|---|---|---|---|---|---|
| Laptop | Electronics | 3 | 2999.97 | 1 | 1 | 78.4 |
| Monitor | Electronics | 4 | 799.96 | 2 | 2 | 21.0 |
| Webcam | Electronics | 1 | 129.99 | 3 | 3 | 3.4 |
| Chair | Furniture | 5 | 599.95 | 1 | 4 | 100.0 |

Notice `revenue_rank_in_category` restarts at `1` for Furniture even
though its top product's `overall_revenue_rank` is `4` — the same
"independent numbering per partition" idea from the very first
`ROW_NUMBER` example, now combined with a percentage-of-partition
calculation in the same query.

**Example 3 — Employee salary gap to next level:**
```sql
SELECT
    name,
    department,
    salary,
    LEAD(salary) OVER (PARTITION BY department ORDER BY salary) AS next_salary_in_dept,
    LEAD(salary) OVER (PARTITION BY department ORDER BY salary) - salary AS gap_to_next,
    LEAD(name)   OVER (PARTITION BY department ORDER BY salary) AS next_person
FROM employees
ORDER BY department, salary;
```

**Output:**

| name | department | salary | next_salary_in_dept | gap_to_next | next_person |
|---|---|---|---|---|---|
| Jane Doe | Engineering | 90000 | 95000 | 5000 | John Smith |
| John Smith | Engineering | 95000 | 120000 | 25000 | Sarah Connor |
| Sarah Connor | Engineering | 120000 | NULL | NULL | NULL |
| Tom Brown | Sales | 72000 | 75000 | 3000 | Mike Johnson |
| Mike Johnson | Sales | 75000 | NULL | NULL | NULL |

Each department's **highest** earner has no "next" salary to look
forward to, so `LEAD` returns `NULL` there — same reasoning as the
`LEAD` output example earlier in this lesson, just ordered ascending
this time instead of descending.

---

## Exercises

**Exercise 1:** Number each customer's orders chronologically.
Show customer_id, order_date, total, and the order number (1st, 2nd, 3rd, etc.).

**Exercise 2:** For each employee, show their salary, the highest salary in their
department, and the percentage of the department max they earn.

**Exercise 3:** Calculate a 2-order moving average of order total for each customer
(partition by customer, ordered by date, 2-row window).

**Exercise 4:** Rank products by revenue within each category. Show the top 2
revenue-generating products per category.

**Exercise 5:** Using NTILE(4), divide employees into salary quartiles. Then count
how many employees are in each quartile.

---

## Key Takeaways

1. Window functions don't collapse rows — every row in = every row out
2. PARTITION BY creates sub-windows (like GROUP BY but without collapsing)
3. ORDER BY inside OVER changes behavior from full-partition to running aggregate
4. ROW_NUMBER: unique, RANK: gaps, DENSE_RANK: no gaps (for ties)
5. LAG/LEAD let you compare a row to the row before/after it
6. Running totals: `SUM(x) OVER (ORDER BY date)` — cumulative
7. LAST_VALUE needs explicit frame `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`
8. The WINDOW clause reuses window definitions instead of repeating them

---

## Next Lesson
[Lesson 09 — Indexes & Performance](09-indexes-and-performance.md)
