# Lesson 05 — Aggregations

## Goal
Master GROUP BY and every aggregate function. Aggregations are the heart of
analytics queries — you'll use them in every real project.

## Prerequisites
- [Lesson 04](04-filtering-deep-dive.md) — filtering

## After This Lesson You Will Be Able To
- Write GROUP BY queries to summarize data
- Use HAVING to filter groups
- Use COUNT, SUM, AVG, MIN, MAX correctly
- Understand the difference between WHERE and HAVING
- Use FILTER for conditional aggregation
- Use ROLLUP and CUBE for multi-level summaries

---

## The Mental Model

Without GROUP BY, aggregate functions collapse ALL rows to ONE result:

```sql
SELECT COUNT(*), SUM(price), AVG(price) FROM products;
-- Returns: 1 row with totals across all products
```

With GROUP BY, you split rows into groups FIRST, then aggregate each group:

```sql
SELECT category, COUNT(*), AVG(price) FROM products GROUP BY category;
-- Returns: 1 row per category, with stats for each category
```

**The ORDER OF EXECUTION rule:**
```
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

- WHERE filters BEFORE grouping (filters individual rows)
- HAVING filters AFTER grouping (filters groups)

---

## COUNT — Counting Rows

```sql
-- Count ALL rows
SELECT COUNT(*) FROM orders;           -- 10

-- Count non-NULL values in a column
SELECT COUNT(customer_id) FROM orders; -- rows where customer_id is not NULL
SELECT COUNT(manager_id) FROM employees; -- 9 (1 employee has NULL manager_id)

-- Count distinct values
SELECT COUNT(DISTINCT country) FROM customers;     -- unique countries
SELECT COUNT(DISTINCT customer_id) FROM orders;    -- customers who ordered

-- Count per group
SELECT
    status,
    COUNT(*) AS order_count
FROM orders
GROUP BY status
ORDER BY order_count DESC;

-- Count with condition using CASE
SELECT
    COUNT(*)                                              AS total,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END)     AS delivered,
    COUNT(CASE WHEN status = 'pending'   THEN 1 END)     AS pending,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END)     AS cancelled,
    COUNT(CASE WHEN total > 500          THEN 1 END)     AS large_orders
FROM orders;

-- Modern alternative: FILTER clause (PostgreSQL 9.4+)
SELECT
    COUNT(*)                                   AS total,
    COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
    COUNT(*) FILTER (WHERE status = 'pending')   AS pending,
    COUNT(*) FILTER (WHERE total > 500)          AS large_orders
FROM orders;
```

---

## SUM — Totals

```sql
-- Total revenue
SELECT SUM(total) AS total_revenue FROM orders;

-- Total revenue by status
SELECT
    status,
    SUM(total) AS revenue
FROM orders
GROUP BY status
ORDER BY revenue DESC NULLS LAST;

-- SUM with condition
SELECT
    SUM(total)                            AS total_revenue,
    SUM(total) FILTER (WHERE status = 'delivered') AS delivered_revenue,
    SUM(total) FILTER (WHERE status = 'cancelled') AS cancelled_revenue
FROM orders;

-- Sum per customer
SELECT
    customer_id,
    COUNT(*) AS order_count,
    SUM(total) AS lifetime_value
FROM orders
GROUP BY customer_id
ORDER BY lifetime_value DESC;

-- SUM of computed column
SELECT
    product_id,
    SUM(quantity * unit_price) AS total_revenue,
    SUM(quantity)              AS total_units_sold
FROM order_items
GROUP BY product_id
ORDER BY total_revenue DESC;
```

---

## AVG — Averages

```sql
-- Average price
SELECT AVG(price) FROM products;                     -- 256.39 (example)
SELECT ROUND(AVG(price), 2) AS avg_price FROM products;

-- Average by category
SELECT
    category,
    COUNT(*) AS product_count,
    ROUND(AVG(price), 2) AS avg_price,
    MIN(price) AS min_price,
    MAX(price) AS max_price
FROM products
GROUP BY category
ORDER BY avg_price DESC;

-- Weighted average
SELECT
    SUM(unit_price * quantity) / SUM(quantity) AS weighted_avg_price
FROM order_items;

-- Average order value per customer
SELECT
    customer_id,
    COUNT(*) AS orders,
    ROUND(AVG(total), 2) AS avg_order_value,
    ROUND(SUM(total), 2) AS lifetime_value
FROM orders
WHERE status != 'cancelled'
GROUP BY customer_id
HAVING COUNT(*) >= 1
ORDER BY avg_order_value DESC;
```

---

## MIN and MAX

```sql
-- Simplest uses
SELECT MIN(price) AS cheapest, MAX(price) AS most_expensive FROM products;
SELECT MIN(order_date) AS first_order, MAX(order_date) AS last_order FROM orders;
SELECT MIN(salary), MAX(salary) FROM employees;

-- Min/max per group
SELECT
    department,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary,
    MAX(salary) - MIN(salary) AS salary_spread
FROM employees
GROUP BY department;

-- Min/max with text (alphabetical)
SELECT MIN(name), MAX(name) FROM customers;

-- Min/max date per customer
SELECT
    customer_id,
    MIN(order_date) AS first_purchase,
    MAX(order_date) AS last_purchase,
    MAX(order_date) - MIN(order_date) AS customer_lifespan_days
FROM orders
GROUP BY customer_id;
```

---

## GROUP BY — Grouping Rules

```sql
-- Rule: every column in SELECT that is NOT an aggregate
--       MUST appear in GROUP BY

-- WRONG:
SELECT category, name, AVG(price) FROM products GROUP BY category;
-- ERROR: column "name" must appear in GROUP BY

-- CORRECT:
SELECT category, AVG(price) FROM products GROUP BY category;
-- Or include name in GROUP BY (but then you get one row per category+name combo)
SELECT category, name, price FROM products GROUP BY category, name, price;

-- Group by multiple columns
SELECT
    country,
    city,
    COUNT(*) AS customer_count
FROM customers
GROUP BY country, city
ORDER BY country, customer_count DESC;

-- Group by expression
SELECT
    DATE_TRUNC('month', order_date) AS month,
    COUNT(*) AS orders_count,
    SUM(total) AS monthly_revenue
FROM orders
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;

-- Group by column position (works but fragile)
SELECT category, COUNT(*) FROM products GROUP BY 1;

-- Group by alias — PostgreSQL DOES allow this
SELECT
    DATE_TRUNC('month', order_date) AS month,
    COUNT(*) AS order_count
FROM orders
GROUP BY month  -- PostgreSQL allows grouping by alias
ORDER BY month;
```

---

## HAVING — Filtering Groups

HAVING filters groups AFTER aggregation. Think of it as WHERE for groups.

```sql
-- Only show categories with more than 2 products
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category
HAVING COUNT(*) > 2;

-- Only show customers with more than 1 order
SELECT customer_id, COUNT(*) AS orders
FROM orders
GROUP BY customer_id
HAVING COUNT(*) > 1;

-- Combined: categories with avg price over $100
SELECT
    category,
    COUNT(*) AS products,
    ROUND(AVG(price), 2) AS avg_price
FROM products
WHERE stock > 0              -- WHERE: filter individual products (before grouping)
GROUP BY category
HAVING AVG(price) > 100      -- HAVING: filter groups (after grouping)
ORDER BY avg_price DESC;

-- HAVING with FILTER
SELECT
    customer_id,
    COUNT(*) AS total_orders,
    COUNT(*) FILTER (WHERE status = 'delivered') AS delivered
FROM orders
GROUP BY customer_id
HAVING COUNT(*) FILTER (WHERE status = 'delivered') >= 1;

-- WHERE vs HAVING — the key difference:
-- WHERE runs BEFORE GROUP BY and cannot use aggregate results
-- HAVING runs AFTER GROUP BY and CAN use aggregate results

-- WRONG: can't use aggregate in WHERE
SELECT category FROM products WHERE AVG(price) > 100 GROUP BY category;

-- CORRECT:
SELECT category FROM products GROUP BY category HAVING AVG(price) > 100;
```

---

## Aggregate with DISTINCT

```sql
-- Count distinct values
SELECT COUNT(DISTINCT country) FROM customers;

-- Sum distinct values (less common but valid)
SELECT SUM(DISTINCT salary) FROM employees;

-- Count distinct items per order
SELECT
    order_id,
    COUNT(DISTINCT product_id) AS unique_products,
    COUNT(*) AS total_line_items,
    SUM(quantity) AS total_units
FROM order_items
GROUP BY order_id;
```

---

## STRING_AGG — Aggregate Strings

```sql
-- Concatenate names into a single string
SELECT
    country,
    STRING_AGG(name, ', ' ORDER BY name) AS customer_names
FROM customers
GROUP BY country;
-- Output for 'US': 'Alice Johnson, David Brown, Frank Lee, Jack Davis'

-- Collect product names by category
SELECT
    category,
    STRING_AGG(name, ' | ' ORDER BY price DESC) AS products
FROM products
GROUP BY category;

-- Array aggregation
SELECT
    category,
    ARRAY_AGG(name ORDER BY price) AS product_names,
    ARRAY_AGG(price ORDER BY price) AS prices
FROM products
GROUP BY category;
```

---

## ROLLUP — Subtotals and Grand Total

### Start with what plain GROUP BY gives you

```sql
SELECT country, city, COUNT(*) AS customers
FROM customers
GROUP BY country, city;

-- US    | New York       | 2
-- US    | San Francisco  | 1
-- US    | Chicago        | 1
-- UK    | London         | 2
```

This only gives you the **most detailed level** — one row per exact
`country + city` combination. There's no row telling you "how many
customers in the US overall," or "how many customers total." You'd have
to run separate queries for those, then stitch the results together
yourself.

### What ROLLUP actually does

Think of `country, city` as a **hierarchy** — city sits inside country,
country sits inside "everything." `ROLLUP` walks back up that hierarchy
one level at a time, generating a subtotal row at each level, ending in
one grand-total row.

For `ROLLUP(country, city)`, it's exactly equivalent to running three
separate `GROUP BY` queries and stacking their results with `UNION ALL`:

```sql
-- Level 1: full detail, grouped by both columns
SELECT country, city, COUNT(*) FROM customers GROUP BY country, city

UNION ALL

-- Level 2: one step up — grouped by country only (city collapsed)
SELECT country, NULL, COUNT(*) FROM customers GROUP BY country

UNION ALL

-- Level 3: all the way up — nothing grouped, grand total
SELECT NULL, NULL, COUNT(*) FROM customers;
```

`ROLLUP` just does all three passes for you in one query, and uses
`NULL` to mark "this column was collapsed at this row" — `NULL` doesn't
mean "no city," it means **"this row is a subtotal that no longer
breaks things down by city."**

```sql
SELECT
    country,
    city,
    COUNT(*) AS customers
FROM customers
GROUP BY ROLLUP(country, city)
ORDER BY country NULLS LAST, city NULLS LAST;
```

| country | city | customers | What this row means |
|---|---|---|---|
| US | New York | 2 | Detail row — exact country+city |
| US | San Francisco | 1 | Detail row |
| US | Chicago | 1 | Detail row |
| US | **NULL** | **4** | Subtotal — all US customers, city collapsed (2+1+1) |
| UK | London | 2 | Detail row |
| UK | **NULL** | **2** | Subtotal — all UK customers |
| **NULL** | **NULL** | **10** | Grand total — everything collapsed |

**How to read any row:** the further left a `NULL` appears, the higher
up the hierarchy that row's total is. A `NULL` in `city` only = "summed
across all cities in that country." `NULL` in both = "summed across
everything."

**Why the column order in `ROLLUP(country, city)` matters:** ROLLUP
collapses columns **right to left**. `ROLLUP(country, city)` gives you
subtotals by country, then a grand total — never a subtotal by city
alone (that would require `city` to collapse before `country`, which
isn't how the hierarchy was declared). If you wrote `ROLLUP(city, country)`
instead, you'd get city-level subtotals and no country-level ones — the
order encodes which column is the "outer" grouping.

```sql
-- Another example: revenue rollup by category, then product name
SELECT
    category,
    name,
    SUM(stock * price) AS inventory_value
FROM products
GROUP BY ROLLUP(category, name)
ORDER BY category NULLS LAST, name NULLS LAST;
-- Detail rows per product → subtotal per category → grand total
```

---

## CUBE — All Combinations of Subtotals

```sql
-- CUBE generates subtotals for ALL combinations of columns
SELECT
    country,
    city,
    COUNT(*) AS customers
FROM customers
GROUP BY CUBE(country, city);

-- For 2 dimensions, CUBE produces 2^2 = 4 grouping sets:
-- (country, city), (country), (city), ()
```

---

## GROUPING SETS — Custom Combinations

```sql
-- Explicitly specify which groupings you want
SELECT
    country,
    city,
    COUNT(*) AS customers
FROM customers
GROUP BY GROUPING SETS (
    (country, city),  -- country + city
    (country),        -- country only
    ()                -- grand total
);

-- Equivalent to ROLLUP(country, city) in this case
```

---

## Practical Analytics Patterns

### Monthly Revenue Report

```sql
SELECT
    TO_CHAR(order_date, 'YYYY-MM') AS month,
    COUNT(*) AS order_count,
    COUNT(DISTINCT customer_id) AS unique_customers,
    ROUND(SUM(total), 2) AS revenue,
    ROUND(AVG(total), 2) AS avg_order_value,
    ROUND(MAX(total), 2) AS largest_order
FROM orders
WHERE status != 'cancelled'
GROUP BY TO_CHAR(order_date, 'YYYY-MM')
ORDER BY month;
```

### Product Sales Summary

```sql
SELECT
    p.name,
    p.category,
    COUNT(DISTINCT oi.order_id) AS orders_containing,
    SUM(oi.quantity)            AS total_units_sold,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS total_revenue,
    ROUND(AVG(oi.unit_price), 2) AS avg_selling_price
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
GROUP BY p.id, p.name, p.category
ORDER BY total_revenue DESC NULLS LAST;
```

### Department Salary Report

```sql
SELECT
    department,
    COUNT(*) AS headcount,
    ROUND(MIN(salary), 2) AS min_salary,
    ROUND(MAX(salary), 2) AS max_salary,
    ROUND(AVG(salary), 2) AS avg_salary,
    ROUND(SUM(salary), 2) AS total_payroll,
    ROUND(MAX(salary) - MIN(salary), 2) AS salary_range
FROM employees
GROUP BY department
ORDER BY total_payroll DESC;
```

### Customer Segmentation

```sql
SELECT
    customer_id,
    COUNT(*) AS total_orders,
    ROUND(SUM(total), 2) AS lifetime_value,
    ROUND(AVG(total), 2) AS avg_order_value,
    MAX(order_date) AS last_order_date,
    CASE
        WHEN SUM(total) >= 1000 THEN 'VIP'
        WHEN SUM(total) >= 300  THEN 'Regular'
        ELSE                         'Occasional'
    END AS customer_tier
FROM orders
WHERE status != 'cancelled'
GROUP BY customer_id
ORDER BY lifetime_value DESC;
```

---

## Exercises

**Exercise 1:** Find the total revenue, average order value, and number of orders
per status. Include only statuses with at least 1 order.

**Exercise 2:** Which categories have an average product price above $100?
Show category name, product count, and rounded average price.

**Exercise 3:** Find each customer's total spending (exclude cancelled orders).
Show only customers who have spent more than $200 total.
Order by total spending descending.

**Exercise 4:** Calculate monthly order stats: month, order count, total revenue,
and average order value. Only include months with revenue above $500.

**Exercise 5:** For each department, show:
- Number of employees
- Min, max, average salary
- Number of employees earning above department average
(Hint for the last part: use a CASE WHEN salary > AVG(salary) THEN 1 END)

---

## Key Takeaways

1. GROUP BY collapses rows into groups; every non-aggregate SELECT column must be in GROUP BY
2. WHERE filters before grouping; HAVING filters after grouping
3. `COUNT(*)` counts all rows; `COUNT(col)` skips NULLs
4. `FILTER (WHERE condition)` is the modern way to do conditional aggregation
5. `STRING_AGG` and `ARRAY_AGG` collect values across rows
6. `ROLLUP` adds subtotals at each grouping level plus a grand total
7. NULL in GROUP BY: NULL values form their own group

---

## Next Lesson
[Lesson 06 — JOINs: Complete Guide](06-joins-complete.md)
