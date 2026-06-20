# Common SQL Patterns — Copy-Paste Solutions

---

## Pagination

```sql
-- Offset-based (simple but slow on large offsets)
SELECT * FROM products ORDER BY id LIMIT 20 OFFSET 40;  -- page 3

-- Keyset / cursor-based (fast, no drift)
-- After page 1 ends at id=20:
SELECT * FROM products WHERE id > 20 ORDER BY id LIMIT 20;
-- This scales infinitely, unlike OFFSET
```

---

## Top N Per Group

```sql
-- Top 3 products per category by price
WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY price DESC) AS rn
    FROM products
)
SELECT * FROM ranked WHERE rn <= 3;

-- PostgreSQL-specific: DISTINCT ON
SELECT DISTINCT ON (category)
    category, name, price
FROM products
ORDER BY category, price DESC;  -- gets top 1 per category
```

---

## Running Total

```sql
SELECT
    order_date,
    total,
    SUM(total) OVER (ORDER BY order_date) AS cumulative_revenue
FROM orders
WHERE status = 'delivered'
ORDER BY order_date;
```

---

## Month-Over-Month Growth

```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(total) AS revenue
    FROM orders GROUP BY 1
)
SELECT
    month,
    revenue,
    LAG(revenue) OVER (ORDER BY month) AS prev_month,
    ROUND((revenue - LAG(revenue) OVER (ORDER BY month))
          / NULLIF(LAG(revenue) OVER (ORDER BY month), 0) * 100, 1) AS pct_change
FROM monthly ORDER BY month;
```

---

## Gaps in Sequence / Missing IDs

```sql
-- Find missing IDs in a sequence
SELECT s.id
FROM GENERATE_SERIES(1, (SELECT MAX(id) FROM orders)) s(id)
LEFT JOIN orders o ON o.id = s.id
WHERE o.id IS NULL;
```

---

## Upsert (Insert or Update)

```sql
INSERT INTO customers (email, name, city)
VALUES ('alice@example.com', 'Alice Updated', 'Boston')
ON CONFLICT (email) DO UPDATE SET
    name = EXCLUDED.name,
    city = EXCLUDED.city;
```

---

## Soft Delete

```sql
-- Delete
UPDATE records SET deleted_at = NOW() WHERE id = 42;

-- Active records
SELECT * FROM records WHERE deleted_at IS NULL;

-- Restore
UPDATE records SET deleted_at = NULL WHERE id = 42;
```

---

## Hierarchical Data (Org Chart)

```sql
WITH RECURSIVE tree AS (
    SELECT id, name, manager_id, 0 AS depth, name AS path
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, t.depth+1, t.path||' > '||e.name
    FROM employees e JOIN tree t ON e.manager_id = t.id
)
SELECT REPEAT('  ', depth) || name AS org_chart, path FROM tree ORDER BY path;
```

---

## Pivot (Rows to Columns)

```sql
SELECT
    department,
    SUM(salary) FILTER (WHERE EXTRACT(YEAR FROM hire_date) = 2021) AS "2021",
    SUM(salary) FILTER (WHERE EXTRACT(YEAR FROM hire_date) = 2022) AS "2022",
    SUM(salary) FILTER (WHERE EXTRACT(YEAR FROM hire_date) = 2023) AS "2023"
FROM employees
GROUP BY department;
```

---

## Fill Time Series Gaps

```sql
WITH dates AS (
    SELECT d::DATE AS day
    FROM GENERATE_SERIES('2024-01-01', '2024-01-31', '1 day'::INTERVAL) d
)
SELECT d.day, COALESCE(SUM(o.total), 0) AS revenue
FROM dates d
LEFT JOIN orders o ON o.order_date = d.day
GROUP BY d.day ORDER BY d.day;
```

---

## Deduplicate (Keep One Row Per Group)

```sql
-- Keep the most recent order per customer
DELETE FROM orders
WHERE id NOT IN (
    SELECT DISTINCT ON (customer_id) id
    FROM orders
    ORDER BY customer_id, order_date DESC
);

-- Using ROW_NUMBER
WITH dups AS (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
)
DELETE FROM orders WHERE id IN (SELECT id FROM dups WHERE rn > 1);
```

---

## Find Duplicates

```sql
-- Rows with duplicate email
SELECT email, COUNT(*) AS count
FROM customers
GROUP BY email
HAVING COUNT(*) > 1;

-- All duplicate rows (keeping the lowest id)
SELECT * FROM customers
WHERE id NOT IN (
    SELECT MIN(id) FROM customers GROUP BY email
);
```

---

## Most Recent Record Per Group

```sql
-- Latest order per customer (3 equivalent ways)
-- 1. DISTINCT ON
SELECT DISTINCT ON (customer_id) *
FROM orders ORDER BY customer_id, order_date DESC;

-- 2. Window function
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS rn
    FROM orders
) t WHERE rn = 1;

-- 3. Correlated subquery
SELECT * FROM orders o
WHERE order_date = (SELECT MAX(order_date) FROM orders WHERE customer_id = o.customer_id);
```

---

## Percentage of Total

```sql
SELECT
    category,
    SUM(price * stock) AS inventory_value,
    ROUND(SUM(price * stock) * 100.0 / SUM(SUM(price * stock)) OVER (), 1) AS pct_of_total
FROM products
GROUP BY category;
```

---

## Safe Division (Avoid Divide-by-Zero)

```sql
SELECT 100.0 / NULLIF(total_orders, 0) AS conversion_rate;
-- Returns NULL instead of error when total_orders = 0
```

---

## Batch Update / Conditional Update

```sql
-- Update multiple rows with different values
UPDATE products SET price = c.new_price
FROM (VALUES
    (1, 1399.99),
    (2,   32.99),
    (3,   54.99)
) AS c(id, new_price)
WHERE products.id = c.id;
```

---

## EXISTS vs IN — Correct Pattern

```sql
-- Customers who have ordered (both correct, EXISTS faster for large datasets)
SELECT * FROM customers WHERE EXISTS (
    SELECT 1 FROM orders WHERE customer_id = customers.id
);

-- Customers who have NEVER ordered (NULL-safe, unlike NOT IN)
SELECT * FROM customers WHERE NOT EXISTS (
    SELECT 1 FROM orders WHERE customer_id = customers.id
);
```

---

## Multi-Column IN

```sql
-- Rows matching any of these (country, city) combinations
SELECT * FROM customers
WHERE (country, city) IN (
    ('US', 'New York'),
    ('UK', 'London')
);
```

---

## Conditional Insert (if not exists)

```sql
INSERT INTO tags (name)
SELECT 'sql'
WHERE NOT EXISTS (SELECT 1 FROM tags WHERE name = 'sql');
```

---

## Copy Table Structure

```sql
-- New table with same structure (no data)
CREATE TABLE orders_archive (LIKE orders INCLUDING ALL);
-- INCLUDING ALL copies constraints, indexes, defaults, etc.

-- New table with structure + data
CREATE TABLE orders_backup AS SELECT * FROM orders;
-- No constraints/indexes copied
```

---

## Analyze Table Size

```sql
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(quote_ident(tablename))) AS total_size,
    pg_size_pretty(pg_relation_size(quote_ident(tablename))) AS table_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(quote_ident(tablename)) DESC;
```
