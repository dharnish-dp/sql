# Lesson 07 — Subqueries & CTEs

## Goal
Learn to write complex queries using subqueries and Common Table Expressions
(CTEs). These tools let you build queries in layers, making the impossible possible.

## Prerequisites
- [Lesson 06](06-joins-complete.md) — JOINs

## After This Lesson You Will Be Able To
- Write scalar, row, table, and correlated subqueries
- Use EXISTS and NOT EXISTS correctly
- Use WITH (CTEs) to break queries into readable steps
- Write recursive CTEs for hierarchical data
- Know when to use a subquery vs a CTE vs a JOIN

---

## What Is a Subquery?

A subquery is a query nested inside another query. It can appear in:
- SELECT (scalar subquery)
- FROM (derived table / inline view)
- WHERE / HAVING (filter subquery)
- JOIN (join to a derived table)

```
Outer query
└── Subquery (inner query, runs first)
```

---

## Scalar Subqueries — Return One Value

A scalar subquery returns exactly ONE row and ONE column.
You can use it anywhere a single value is expected.

```sql
-- How does each product price compare to the overall average?
SELECT
    name,
    price,
    (SELECT AVG(price) FROM products) AS avg_all_products,
    price - (SELECT AVG(price) FROM products) AS price_vs_avg,
    ROUND(price / (SELECT AVG(price) FROM products) * 100, 1) AS pct_of_avg
FROM products
ORDER BY price_vs_avg DESC;

-- Most expensive product name in each category
SELECT
    name,
    category,
    price,
    (SELECT MAX(price) FROM products p2 WHERE p2.category = p.category) AS category_max
FROM products p;

-- Employee's manager name (scalar subquery instead of self-JOIN)
SELECT
    name,
    salary,
    (SELECT name FROM employees m WHERE m.id = e.manager_id) AS manager_name
FROM employees e;
-- Equivalent to a LEFT JOIN — both work, JOIN is usually faster
```

---

## Table Subqueries in FROM — Derived Tables

A subquery in FROM acts like a temporary table. Must have an alias.

```sql
-- Average order value per customer, then find customers above average
SELECT *
FROM (
    SELECT
        customer_id,
        COUNT(*) AS orders,
        ROUND(AVG(total), 2) AS avg_order_value
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
) customer_stats
WHERE avg_order_value > (SELECT AVG(total) FROM orders WHERE status != 'cancelled');

-- Multi-layer derived tables
SELECT dept_stats.department, dept_stats.avg_salary
FROM (
    SELECT
        department,
        AVG(salary) AS avg_salary,
        COUNT(*) AS headcount
    FROM employees
    GROUP BY department
) dept_stats
WHERE dept_stats.headcount >= 2
  AND dept_stats.avg_salary > 70000;

-- Derived table for pagination with total count
SELECT
    p.*,
    total_count.n AS total_rows
FROM (
    SELECT * FROM products ORDER BY price DESC LIMIT 5
) p
CROSS JOIN (SELECT COUNT(*) AS n FROM products) total_count;
```

---

## Subqueries in WHERE — Filtering with Subqueries

### Comparison Operators

```sql
-- Products priced above the average
SELECT name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products)
ORDER BY price DESC;

-- Most expensive product in each category
SELECT name, category, price
FROM products p
WHERE price = (
    SELECT MAX(price)
    FROM products
    WHERE category = p.category   -- correlated subquery!
);

-- Orders with total above the average order total
SELECT id, customer_id, total
FROM orders
WHERE total > (SELECT AVG(total) FROM orders WHERE status != 'cancelled')
  AND status != 'cancelled';
```

### IN and NOT IN with Subqueries

```sql
-- Customers who have placed at least one order
SELECT name, email
FROM customers
WHERE id IN (SELECT DISTINCT customer_id FROM orders);

-- Customers who have NEVER ordered
SELECT name, email
FROM customers
WHERE id NOT IN (SELECT DISTINCT customer_id FROM orders WHERE customer_id IS NOT NULL);
-- ⚠️ The WHERE customer_id IS NOT NULL is CRITICAL
-- If ANY customer_id in orders is NULL, NOT IN returns nothing!

-- Products that have been ordered
SELECT name, category, price
FROM products
WHERE id IN (SELECT DISTINCT product_id FROM order_items);

-- Products never ordered
SELECT name, category, price
FROM products
WHERE id NOT IN (SELECT DISTINCT product_id FROM order_items);
```

### ANY and ALL with Subqueries

```sql
-- Products more expensive than any Electronics product
SELECT name, price
FROM products
WHERE price > ANY (SELECT price FROM products WHERE category = 'Electronics');
-- = "more expensive than the cheapest Electronics product"

-- Products more expensive than ALL Stationery products
SELECT name, price
FROM products
WHERE price > ALL (SELECT price FROM products WHERE category = 'Stationery');
-- = "more expensive than every Stationery product"
```

---

## Correlated Subqueries — Reference the Outer Query

A correlated subquery references columns from the OUTER query.
It runs ONCE PER ROW of the outer query (can be slow on large tables).

```sql
-- For each employee, is their salary above their department average?
SELECT
    name,
    department,
    salary,
    (SELECT AVG(salary) FROM employees e2 WHERE e2.department = e.department) AS dept_avg,
    CASE
        WHEN salary > (SELECT AVG(salary) FROM employees e2 WHERE e2.department = e.department)
        THEN 'Above Average'
        ELSE 'Below Average'
    END AS vs_dept_avg
FROM employees e
ORDER BY department, salary DESC;

-- Products with above-average price in their category
SELECT name, category, price
FROM products p
WHERE price > (
    SELECT AVG(price)
    FROM products
    WHERE category = p.category   -- correlated: references outer p.category
)
ORDER BY category, price DESC;

-- Most recent order per customer (correlated subquery approach)
SELECT id, customer_id, order_date, total
FROM orders o
WHERE order_date = (
    SELECT MAX(order_date)
    FROM orders
    WHERE customer_id = o.customer_id  -- correlated
);
-- Better alternative: Window functions (Lesson 08) or DISTINCT ON
```

---

## EXISTS and NOT EXISTS

EXISTS returns TRUE if the subquery returns ANY row.
More efficient than IN for large datasets because it short-circuits.

```sql
-- Customers who have placed at least one order
SELECT c.name, c.email
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
-- The SELECT 1 is conventional — the actual value doesn't matter
-- EXISTS only cares if ANY row is returned

-- Customers with NO orders
SELECT c.name, c.email
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
-- NULL-safe! Unlike NOT IN, NOT EXISTS handles NULL correctly

-- Products that exist in some order AND have stock > 0
SELECT p.name, p.price, p.stock
FROM products p
WHERE EXISTS (
    SELECT 1 FROM order_items oi WHERE oi.product_id = p.id
)
AND p.stock > 0;

-- Customers who have ordered EVERY product category
SELECT c.name
FROM customers c
WHERE NOT EXISTS (
    SELECT DISTINCT category FROM products
    EXCEPT
    SELECT DISTINCT p.category
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.id
    JOIN products p ON p.id = oi.product_id
    WHERE o.customer_id = c.id
);
```

---

## CTEs (Common Table Expressions) — WITH Clause

CTEs let you name intermediate results and reference them like tables.
Think of them as named subqueries defined at the top of the query.

### Basic CTE Syntax

```sql
WITH cte_name AS (
    SELECT ...
    FROM ...
)
SELECT * FROM cte_name;
```

### Single CTE

```sql
-- Average order value per customer
WITH customer_stats AS (
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        ROUND(SUM(total), 2) AS lifetime_value,
        ROUND(AVG(total), 2) AS avg_order_value
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
)
SELECT
    c.name,
    cs.order_count,
    cs.lifetime_value,
    cs.avg_order_value,
    CASE
        WHEN cs.lifetime_value >= 1000 THEN 'VIP'
        WHEN cs.lifetime_value >= 300  THEN 'Regular'
        ELSE                                'Occasional'
    END AS tier
FROM customers c
JOIN customer_stats cs ON cs.customer_id = c.id
ORDER BY cs.lifetime_value DESC;
```

### Multiple CTEs (Chained)

```sql
WITH
-- Step 1: Order totals per customer
customer_orders AS (
    SELECT
        customer_id,
        COUNT(*) AS order_count,
        SUM(total) AS total_spent
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY customer_id
),
-- Step 2: Add customer info
customer_summary AS (
    SELECT
        c.name,
        c.country,
        co.order_count,
        co.total_spent,
        ROUND(co.total_spent / co.order_count, 2) AS avg_order
    FROM customers c
    JOIN customer_orders co ON co.customer_id = c.id
),
-- Step 3: Rank by spending
ranked AS (
    SELECT
        *,
        RANK() OVER (ORDER BY total_spent DESC) AS spending_rank
    FROM customer_summary
)
-- Final: top spenders
SELECT *
FROM ranked
WHERE spending_rank <= 5;
```

### CTE vs Subquery — When to Use Which

| Situation | Use |
|-----------|-----|
| Query used once, simple | Subquery |
| Query used multiple times | CTE |
| Need readable step-by-step logic | CTE |
| Need recursion | CTE (must be CTE) |
| Performance matters (large data) | Test both — optimizer may inline CTEs |

```sql
-- CTE referenced multiple times
WITH expensive_products AS (
    SELECT id, name, price
    FROM products
    WHERE price > 200
)
SELECT
    ep.name,
    ep.price,
    (SELECT COUNT(*) FROM order_items oi
     WHERE oi.product_id = ep.id) AS times_ordered
FROM expensive_products ep
ORDER BY ep.price DESC;
```

---

## Recursive CTEs — Hierarchical Data

Recursive CTEs can traverse trees and graphs: org charts, folder structures,
bill of materials, category hierarchies.

```sql
WITH RECURSIVE org_chart AS (
    -- Base case: top-level employees (no manager)
    SELECT
        id,
        name,
        manager_id,
        department,
        salary,
        0 AS depth,              -- depth in hierarchy
        name AS path             -- breadcrumb path
    FROM employees
    WHERE manager_id IS NULL     -- start from the top

    UNION ALL

    -- Recursive case: find subordinates of current level
    SELECT
        e.id,
        e.name,
        e.manager_id,
        e.department,
        e.salary,
        oc.depth + 1,
        oc.path || ' → ' || e.name   -- build the path
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id   -- join to previous level
)
SELECT
    REPEAT('  ', depth) || name AS indented_name,  -- indent by depth
    department,
    salary,
    depth AS level,
    path
FROM org_chart
ORDER BY path;

-- Output:
-- Sarah Connor        | Engineering | 120000 | 0 | Sarah Connor
--   John Smith        | Engineering |  95000 | 1 | Sarah Connor → John Smith
--   Jane Doe          | Engineering |  90000 | 1 | Sarah Connor → Jane Doe
--   Nina Patel        | Engineering |  88000 | 1 | Sarah Connor → Nina Patel
-- Mike Johnson        | Sales       |  75000 | 0 | Mike Johnson
--   Lisa Chen         | Sales       |  68000 | 1 | Mike Johnson → Lisa Chen
-- ...
```

### Recursive CTE — Fibonacci Example

```sql
WITH RECURSIVE fib(n, a, b) AS (
    SELECT 0, 0, 1
    UNION ALL
    SELECT n + 1, b, a + b
    FROM fib
    WHERE n < 20
)
SELECT n, a AS fibonacci_number FROM fib;
```

### Recursive CTE — Date Series

```sql
-- Generate a series of dates (PostgreSQL has generate_series() built in,
-- but this shows the recursive pattern)
WITH RECURSIVE dates AS (
    SELECT '2024-01-01'::DATE AS d
    UNION ALL
    SELECT d + 1
    FROM dates
    WHERE d < '2024-01-31'
)
SELECT d AS date FROM dates;
```

---

## Subquery Factoring — Materialized vs Non-Materialized

PostgreSQL 12+ CTEs have an important behavior change:

```sql
-- By default (PostgreSQL 12+): optimizer may inline the CTE
-- The CTE is re-evaluated each time it's referenced if inlined
WITH expensive AS (
    SELECT * FROM products WHERE price > 100
)
SELECT * FROM expensive WHERE category = 'Electronics';
-- PostgreSQL may choose to run the WHERE before materializing

-- Force materialization (run CTE exactly once, result is cached)
WITH expensive AS MATERIALIZED (
    SELECT * FROM products WHERE price > 100
)
SELECT * FROM expensive WHERE category = 'Electronics';
-- Good when: CTE is expensive and used multiple times

-- Force inlining (let optimizer optimize through CTE)
WITH expensive AS NOT MATERIALIZED (
    SELECT * FROM products WHERE price > 100
)
SELECT * FROM expensive WHERE category = 'Electronics';
```

---

## Complete Real-World Examples

**Example 1 — Rank customers by spending with CTE:**
```sql
WITH spending AS (
    SELECT
        customer_id,
        SUM(total)  AS total_spent,
        COUNT(*)    AS order_count,
        MAX(order_date) AS last_order
    FROM orders
    WHERE status = 'delivered'
    GROUP BY customer_id
),
ranked AS (
    SELECT
        c.name,
        c.email,
        c.country,
        s.total_spent,
        s.order_count,
        s.last_order,
        RANK() OVER (ORDER BY s.total_spent DESC) AS rank
    FROM customers c
    JOIN spending s ON s.customer_id = c.id
)
SELECT * FROM ranked WHERE rank <= 3;
```

**Example 2 — Products with above-category-average price:**
```sql
WITH category_avg AS (
    SELECT category, AVG(price) AS avg_price
    FROM products
    GROUP BY category
)
SELECT
    p.name,
    p.category,
    p.price,
    ROUND(ca.avg_price, 2) AS category_avg,
    ROUND(p.price - ca.avg_price, 2) AS above_avg_by
FROM products p
JOIN category_avg ca ON ca.category = p.category
WHERE p.price > ca.avg_price
ORDER BY p.category, above_avg_by DESC;
```

---

## Exercises

**Exercise 1:** Using a subquery, find all products priced above the average product price.
Show name, category, price, and how much above average they are.

**Exercise 2:** Find customers who have never placed an order using:
a) NOT IN with subquery  
b) NOT EXISTS  
c) LEFT JOIN with NULL check  
Compare: are the results the same?

**Exercise 3:** Write a CTE that calculates total spending per customer, then
use it to label customers as 'VIP' (>$500), 'Regular' ($100–$500), or 'New' (<$100).

**Exercise 4:** Write a recursive CTE to display the management hierarchy for
the Engineering department, showing each employee's level in the hierarchy.

**Exercise 5:** Using a correlated subquery, find the most expensive product
in each category (without using window functions).

---

## Key Takeaways

1. Scalar subqueries return one value — use them in SELECT or WHERE
2. Table subqueries in FROM act as temporary tables — always alias them
3. Correlated subqueries reference the outer query and run once per outer row
4. `NOT EXISTS` is NULL-safe; `NOT IN` is NOT NULL-safe (breaks on NULL in list)
5. CTEs (`WITH`) make complex queries readable by naming each logical step
6. Recursive CTEs use `UNION ALL` with a base case + recursive case
7. PostgreSQL 12+: CTEs are inlined by default; use `MATERIALIZED` to cache

---

## Next Lesson
[Lesson 08 — Window Functions](08-window-functions.md)
