# Lesson 06 — JOINs: The Complete Guide

## Goal
Master every type of JOIN. JOINs are the most powerful and most misunderstood
part of SQL. After this lesson, you'll understand them visually, conceptually,
and know exactly which to use when.

## Prerequisites
- [Lesson 05](05-aggregations.md) — aggregations

## After This Lesson You Will Be Able To
- Write INNER, LEFT, RIGHT, FULL OUTER, CROSS, and SELF joins
- Join three or more tables together
- Understand the difference between all join types with visual diagrams
- Avoid common JOIN pitfalls (row multiplication, NULL confusion)
- Use JOIN with aggregations correctly

---

## The JOIN Concept

When data is split across multiple tables (normalization), JOINs bring it back
together for querying. The ON clause specifies how tables relate.

```
customers table          orders table
┌────┬────────┐          ┌────┬─────────────┬────────┐
│ id │  name  │          │ id │ customer_id │  total │
├────┼────────┤          ├────┼─────────────┼────────┤
│  1 │ Alice  │          │  1 │      1      │ 999.99 │
│  2 │ Bob    │          │  2 │      1      │  49.99 │
│  3 │ Carol  │          │  3 │      2      │ 129.99 │
└────┴────────┘          │  4 │      5      │ 299.99 │  ← no customer 5!
                         └────┴─────────────┴────────┘
```

Different JOINs answer different questions about this relationship.

---

## INNER JOIN — Only Matching Rows

Returns rows where the ON condition matches in BOTH tables.

```
customers  ∩  orders  (intersection)
```

```sql
-- Syntax
SELECT c.name, o.id AS order_id, o.total
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id;

-- INNER is the default — these are identical:
SELECT c.name, o.id, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id;

-- Real example: Show order details with customer names
SELECT
    c.name        AS customer_name,
    c.email,
    o.id          AS order_id,
    o.order_date,
    o.status,
    o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
ORDER BY c.name, o.order_date;

-- INNER JOIN excludes:
-- Customers with no orders (customer 3, Carol — no order record)
-- Orders with no matching customer (order 4 — customer_id=5 doesn't exist)
```

**Output**, using the sample data from the diagram above:

| customer_name | order_id | total |
|---|---|---|
| Alice | 1 | 999.99 |
| Alice | 2 | 49.99 |
| Bob | 3 | 129.99 |

Carol is gone (no matching order). Order 4 is gone (no matching customer,
`customer_id = 5` doesn't exist). Only rows that matched **on both sides**
survive.

```sql

-- Join with filter
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.total > 100
  AND o.status = 'delivered'
ORDER BY o.total DESC;
```

---

## LEFT JOIN — All Left Rows + Matching Right Rows

Returns ALL rows from the left table, plus matching rows from the right.
Non-matching right side comes back as NULL.

```
customers (all)  +  orders (matching only)
```

```sql
-- Find ALL customers and their orders (customers without orders show NULL)
SELECT
    c.name       AS customer_name,
    o.id         AS order_id,
    o.total,
    o.status
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
ORDER BY c.name;

-- Customers with no orders have NULL in the order columns:
-- Carol White | NULL | NULL | NULL
```

**Output:**

| customer_name | order_id | total | status |
|---|---|---|---|
| Alice | 1 | 999.99 | delivered |
| Alice | 2 | 49.99 | pending |
| Bob | 3 | 129.99 | delivered |
| Carol | NULL | NULL | NULL |

Every customer appears at least once — Carol shows up with `NULL`s
instead of disappearing like she did with `INNER JOIN`. Order 4
(`customer_id = 5`) still doesn't appear — it's not in the *left* table.

```sql

-- Find customers who have NEVER ordered:
SELECT c.name, c.email
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;   -- NULL means no matching order row

-- Count orders per customer (including customers with 0 orders):
SELECT
    c.name,
    COUNT(o.id) AS order_count,   -- COUNT(col) skips NULLs → 0 for no orders
    COALESCE(SUM(o.total), 0) AS total_spent
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name
ORDER BY total_spent DESC;

-- LEFT JOIN vs INNER JOIN:
-- INNER JOIN: only Alice and Bob (they have orders)
-- LEFT JOIN:  Alice, Bob, AND Carol (Carol shows with NULLs)
```

---

## RIGHT JOIN — All Right Rows + Matching Left Rows

Mirror of LEFT JOIN. Returns ALL rows from the right table.
(Usually rewrite as LEFT JOIN by swapping table order — LEFT JOIN is more readable.)

```sql
-- All orders, including those without a matching customer
SELECT
    c.name      AS customer_name,
    o.id        AS order_id,
    o.total
FROM customers c
RIGHT JOIN orders o ON o.customer_id = c.id;

-- Equivalent (and cleaner) with LEFT JOIN:
SELECT
    c.name      AS customer_name,
    o.id        AS order_id,
    o.total
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id;
```

**Output** (either query above produces this):

| customer_name | order_id | total |
|---|---|---|
| Alice | 1 | 999.99 |
| Alice | 2 | 49.99 |
| Bob | 3 | 129.99 |
| NULL | 4 | 299.99 |

Now it's flipped from `LEFT JOIN` — every **order** appears, including
order 4 whose `customer_id = 5` doesn't exist (`customer_name` is
`NULL`). Carol doesn't appear at all — she's not in the *right* table's
matches, and `RIGHT JOIN` only guarantees every right-table row survives.

---

## FULL OUTER JOIN — All Rows from Both Tables

Returns ALL rows from both tables. Non-matches get NULL on the opposite side.

```
customers (all)  ∪  orders (all)
```

```sql
-- All customers and all orders, NULLs where no match
SELECT
    c.name      AS customer_name,
    o.id        AS order_id,
    o.total
FROM customers c
FULL OUTER JOIN orders o ON o.customer_id = c.id;

-- Find rows with NO match on either side:
SELECT
    c.name,
    o.id AS orphan_order_id
FROM customers c
FULL OUTER JOIN orders o ON o.customer_id = c.id
WHERE c.id IS NULL OR o.id IS NULL;
-- Rows where customer has no order AND orders with no customer
```

**Output of the first query** — every customer AND every order, matched
where possible:

| customer_name | order_id | total |
|---|---|---|
| Alice | 1 | 999.99 |
| Alice | 2 | 49.99 |
| Bob | 3 | 129.99 |
| Carol | NULL | NULL |
| NULL | 4 | 299.99 |

This is `LEFT JOIN` and `RIGHT JOIN` combined — Carol survives (no
matching order) *and* order 4 survives (no matching customer). Nothing
from either table is dropped.

---

## CROSS JOIN — Every Combination

Returns the Cartesian product: every row from table A × every row from table B.

```
customers (10 rows) × products (10 rows) = 100 rows
```

```sql
-- Explicit CROSS JOIN syntax
SELECT c.name, p.name AS product
FROM customers c
CROSS JOIN products p;

-- Implicit CROSS JOIN (comma syntax — avoid this)
SELECT c.name, p.name FROM customers c, products p;
```

**Output**, with just 2 customers and 2 products to keep it readable —
every customer paired with every product, 2 × 2 = 4 rows:

| name | product |
|---|---|
| Alice | Laptop |
| Alice | Mouse |
| Bob | Laptop |
| Bob | Mouse |

No `ON` condition, no matching logic — it's pure combination. This is
why `CROSS JOIN` explodes fast: 10 customers × 10 products = 100 rows,
100 × 100 = 10,000 rows. Only use it deliberately (report grids, date
series generation below), never by accident.

```sql

-- When is CROSS JOIN useful?
-- 1. Generate all possible combinations for a report
SELECT
    c.name   AS customer,
    p.name   AS product,
    p.price
FROM customers c
CROSS JOIN products p
WHERE c.country = 'US'
  AND p.category = 'Electronics'
ORDER BY c.name, p.price;

-- 2. Combine with values to create a date series
SELECT
    d.day,
    COALESCE(COUNT(o.id), 0) AS orders
FROM generate_series(
    '2024-01-01'::DATE,
    '2024-03-31'::DATE,
    '1 day'::INTERVAL
) AS d(day)
LEFT JOIN orders o ON o.order_date = d.day::DATE
GROUP BY d.day
ORDER BY d.day;
```

---

## SELF JOIN — A Table Joining Itself

A table joins itself when rows relate to OTHER rows in the same table.
Classic example: employees and their managers (both are employees).

```sql
-- Employee hierarchy: show each employee and their manager's name
SELECT
    e.name         AS employee,
    e.department,
    e.salary,
    m.name         AS manager_name,
    m.salary       AS manager_salary
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id   -- LEFT so top-level managers appear
ORDER BY m.name NULLS FIRST, e.name;
```

**Output** — the same `employees` table plays two roles at once: `e` is
"the employee," `m` is "that employee's manager," matched via
`e.manager_id = m.id`:

| employee | department | salary | manager_name | manager_salary |
|---|---|---|---|---|
| Grace (CEO) | Exec | 250000 | NULL | NULL |
| Alice | Engineering | 95000 | Grace | 250000 |
| Dan | Engineering | 88000 | Alice | 95000 |
| Bob | Sales | 70000 | Grace | 250000 |

Grace has no manager (`manager_id` is `NULL`), so `LEFT JOIN` keeps her
row with `NULL` manager columns instead of dropping her — this is the
same "keep the row, NULL the missing side" behavior as any other
`LEFT JOIN`, just applied to a table joined against itself.

```sql

-- Find employees who earn more than their manager
SELECT
    e.name         AS employee,
    e.salary       AS employee_salary,
    m.name         AS manager,
    m.salary       AS manager_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;

-- Three-level hierarchy: employee → manager → senior manager
SELECT
    e.name    AS employee,
    m.name    AS manager,
    sm.name   AS senior_manager
FROM employees e
JOIN employees m  ON e.manager_id = m.id
LEFT JOIN employees sm ON m.manager_id = sm.id;
```

---

## Joining Three or More Tables

```sql
-- Customer → Order → Order Items → Product
SELECT
    c.name          AS customer,
    o.id            AS order_id,
    o.order_date,
    p.name          AS product,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS line_total
FROM customers c
JOIN orders o      ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p    ON oi.product_id = p.id
WHERE o.status = 'delivered'
ORDER BY c.name, o.order_date, p.name;

-- With aggregation across 4 tables
SELECT
    c.name          AS customer,
    COUNT(DISTINCT o.id)   AS orders,
    COUNT(oi.id)           AS total_line_items,
    SUM(oi.quantity)       AS total_units,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS total_spent
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi  ON oi.order_id = o.id
JOIN products p     ON oi.product_id = p.id
WHERE o.status != 'cancelled'
GROUP BY c.id, c.name
ORDER BY total_spent DESC;
```

---

## JOIN Conditions Beyond Equality

```sql
-- Non-equi JOIN: employees who earn more than the department average
SELECT
    e1.name,
    e1.salary,
    e1.department
FROM employees e1
JOIN employees e2 ON e1.department = e2.department
                 AND e1.salary > e2.salary  -- non-equality condition
GROUP BY e1.id, e1.name, e1.salary, e1.department
HAVING COUNT(*) >= 1;
-- (This is a way to find above-median employees — see window functions for better approach)

-- Range JOIN: match events to time windows
SELECT
    e.event_name,
    p.promotion_name
FROM events e
JOIN promotions p ON e.event_date BETWEEN p.start_date AND p.end_date;

-- Multiple conditions in ON
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
            AND o.total > 100           -- additional condition in the join
            AND o.status = 'delivered';
-- NOTE: for LEFT/FULL JOINs, conditions in ON vs WHERE behave differently!
```

---

## ON vs WHERE for Outer Joins — Critical Difference

```sql
-- Setup: customer 3 (Carol) has no orders

-- LEFT JOIN with condition in WHERE: excludes Carol
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.total > 100;
-- Carol is excluded because her o.total is NULL (fails WHERE)

-- LEFT JOIN with condition in ON: keeps Carol (with NULL)
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
                  AND o.total > 100;
-- Carol appears with NULL (the condition restricted which orders to include,
-- but Carol still shows because it's a LEFT JOIN)

-- Rule:
-- ON clause: filters what the join considers
-- WHERE clause: filters the final result (LEFT JOIN rows that matched NULL)
-- For INNER JOINs, it doesn't matter — same result
```

---

## Row Multiplication — A Common Bug

```sql
-- If a customer has 3 orders, and you JOIN without aggregating:
SELECT c.name, c.email
FROM customers c
JOIN orders o ON o.customer_id = c.id;
-- Alice (2 orders) appears TWICE
-- Bob (1 order) appears ONCE
-- Carol (0 orders) doesn't appear

-- This "fan-out" causes wrong aggregates:
SELECT c.name, SUM(c.salary)  -- WRONG if customer has salary column
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.name;
-- Customer salary counted once per order — inflated!

-- Solutions:
-- 1. Aggregate order data first, then JOIN
SELECT c.name, c.email, t.order_count, t.total_spent
FROM customers c
JOIN (
    SELECT customer_id, COUNT(*) AS order_count, SUM(total) AS total_spent
    FROM orders
    GROUP BY customer_id
) t ON t.customer_id = c.id;

-- 2. Use DISTINCT
SELECT DISTINCT c.name, c.email
FROM customers c
JOIN orders o ON o.customer_id = c.id;
-- But DISTINCT only deduplicates identical rows, not aggregates
```

---

## Visual Summary

```
Table A:    1, 2, 3, 4       (4 rows)
Table B:    1, 2, 5, 6       (4 rows)

INNER JOIN: 1, 2             (matching rows only)
LEFT JOIN:  1, 2, 3, 4       (all A + matches from B, NULL where no B)
RIGHT JOIN: 1, 2, 5, 6       (all B + matches from A, NULL where no A)
FULL OUTER: 1, 2, 3, 4, 5, 6 (all rows, NULLs where no match)
CROSS JOIN: 4 × 4 = 16 rows  (every combination)
```

---

## Complete Real-World Examples

**Example 1 — Full order report:**
```sql
SELECT
    c.name                                  AS customer,
    c.country,
    o.id                                    AS order_id,
    o.order_date,
    o.status,
    STRING_AGG(p.name, ', ' ORDER BY p.name) AS products,
    SUM(oi.quantity)                        AS total_items,
    ROUND(SUM(oi.quantity * oi.unit_price), 2) AS computed_total
FROM orders o
JOIN customers c    ON c.id = o.customer_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id
GROUP BY c.name, c.country, o.id, o.order_date, o.status
ORDER BY o.order_date DESC;
```

**Example 2 — Customers who never ordered:**
```sql
SELECT c.name, c.email, c.city, c.country
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL
ORDER BY c.name;
```

**Example 3 — Products never ordered:**
```sql
SELECT p.name, p.category, p.price, p.stock
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
WHERE oi.id IS NULL
ORDER BY p.category, p.name;
```

**Example 4 — Employee org chart:**
```sql
SELECT
    e.name                          AS employee,
    e.department,
    e.salary,
    COALESCE(m.name, 'No Manager')  AS manager,
    CASE
        WHEN e.salary > m.salary THEN 'Earns more than manager'
        WHEN e.salary = m.salary THEN 'Same as manager'
        WHEN e.salary < m.salary THEN 'Earns less than manager'
        ELSE                          'No manager'
    END AS salary_vs_manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id
ORDER BY e.department, e.name;
```

---

## Exercises

**Exercise 1:** Write a query showing ALL customers and their most recent order date.
Customers with no orders should show NULL for order date. Order by customer name.

**Exercise 2:** Find all products that have been ordered at least once.
Show product name, category, and total quantity sold.

**Exercise 3:** Show the full order detail: customer name, order date, product name,
quantity, unit price, and line total (quantity × price). Only include delivered orders.

**Exercise 4:** Find the manager of each employee. Show employee name, their salary,
manager name, manager salary, and the salary difference.

**Exercise 5:** Which countries have customers who have placed an order? Use JOIN.
Then find which countries have customers who have NOT placed an order. Use LEFT JOIN.

---

## Key Takeaways

1. INNER JOIN: only matching rows on both sides
2. LEFT JOIN: all left rows + matching right (non-matches come back as NULL)
3. FULL OUTER JOIN: all rows from both tables
4. CROSS JOIN: every row × every row (Cartesian product)
5. SELF JOIN: join a table to itself (for hierarchical/comparative data)
6. In LEFT/FULL joins: conditions in ON filter what the join INCLUDES; conditions in WHERE filter the final result (can eliminate NULL rows you wanted to keep)
7. Watch for row multiplication when joining one-to-many relationships

---

## Next Lesson
[Lesson 07 — Subqueries & CTEs](07-subqueries-and-ctes.md)
