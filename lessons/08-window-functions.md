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
