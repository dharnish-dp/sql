# Lesson 13 — Views & Materialized Views

## Goal
Learn to create reusable query abstractions with views, and use materialized
views to cache expensive query results for fast reporting.

## Prerequisites
- [Lesson 12](12-advanced-queries.md) — advanced queries

## After This Lesson You Will Be Able To
- Create, update, and drop views
- Use views for security and abstraction
- Create materialized views and refresh them
- Know when to use a view vs a materialized view
- Use updatable views

---

## What Is a View?

A view is a **saved SELECT query** that you can treat like a table.
It does NOT store data — every time you query a view, PostgreSQL runs
the underlying query.

```
view = named SELECT statement
     = virtual table
     = no storage (unless materialized)
```

### Creating a View

```sql
CREATE VIEW view_name AS
SELECT ...;

-- Example: Active customers with order stats
CREATE VIEW customer_summary AS
SELECT
    c.id,
    c.name,
    c.email,
    c.country,
    COUNT(o.id)         AS total_orders,
    COALESCE(SUM(o.total), 0)  AS lifetime_value,
    MAX(o.order_date)   AS last_order_date
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status != 'cancelled'
GROUP BY c.id, c.name, c.email, c.country;

-- Query the view like a table
SELECT * FROM customer_summary ORDER BY lifetime_value DESC;
SELECT name, total_orders FROM customer_summary WHERE country = 'US';
SELECT * FROM customer_summary WHERE total_orders = 0;  -- customers with no orders
```

---

## Why Use Views?

### 1. Simplify Complex Queries

```sql
-- Without view: complex JOIN every time
SELECT
    c.name, o.id, o.order_date, p.name AS product, oi.quantity, oi.unit_price
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'delivered';

-- Create a view once
CREATE VIEW delivered_order_details AS
SELECT
    c.name AS customer_name,
    c.email,
    o.id AS order_id,
    o.order_date,
    p.name AS product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.quantity * oi.unit_price AS line_total
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON oi.product_id = p.id
WHERE o.status = 'delivered';

-- Now it's simple:
SELECT * FROM delivered_order_details WHERE customer_name = 'Alice Johnson';
SELECT product_name, SUM(quantity) AS sold FROM delivered_order_details GROUP BY product_name;
```

### 2. Row-Level Security

```sql
-- Only show your own data
CREATE VIEW my_orders AS
SELECT * FROM orders WHERE customer_id = current_user_id();
-- Grant access to this view, not to the orders table directly
```

### 3. Hide Sensitive Columns

```sql
-- Users table has password_hash, ssn, etc.
CREATE VIEW public_customer_info AS
SELECT id, name, email, city, country, created_at
FROM customers;
-- No password, no PII

-- Grant view access, not table access
GRANT SELECT ON public_customer_info TO app_user;
```

### 4. Abstraction Layer

```sql
-- If you rename a table, create a view with the old name
-- Old code still works without changes
CREATE VIEW orders_legacy AS SELECT * FROM orders;
```

---

## Modifying and Dropping Views

```sql
-- Replace a view (preserves permissions granted on it)
CREATE OR REPLACE VIEW customer_summary AS
SELECT
    c.id, c.name, c.email,
    COUNT(o.id) AS total_orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name, c.email;
-- Note: can only ADD columns at the end, not remove or reorder with REPLACE

-- Drop a view
DROP VIEW customer_summary;
DROP VIEW IF EXISTS customer_summary;
DROP VIEW customer_summary CASCADE;  -- also drops views/rules that depend on it

-- Rename a view
ALTER VIEW customer_summary RENAME TO customer_stats;

-- Describe a view
\d+ customer_summary
-- or
SELECT definition FROM pg_views WHERE viewname = 'customer_summary';
```

---

## Updatable Views

PostgreSQL can automatically make a view updatable if it meets conditions:
- FROM has only ONE base table
- No GROUP BY, HAVING, DISTINCT, LIMIT, UNION, aggregates, window functions

```sql
-- This view is automatically updatable
CREATE VIEW active_customers AS
SELECT id, name, email, city, country
FROM customers
WHERE deleted_at IS NULL;

-- You can INSERT, UPDATE, DELETE through it
UPDATE active_customers SET city = 'Boston' WHERE id = 1;
INSERT INTO active_customers (name, email) VALUES ('New User', 'new@ex.com');
DELETE FROM active_customers WHERE id = 1;  -- actually deletes the row

-- WITH CHECK OPTION: prevents modifications that would make the row
-- invisible in the view (i.e., violate the WHERE condition)
CREATE VIEW active_customers AS
SELECT id, name, email, deleted_at
FROM customers
WHERE deleted_at IS NULL
WITH CHECK OPTION;

UPDATE active_customers SET deleted_at = NOW() WHERE id = 1;
-- ERROR: new row violates check option for view "active_customers"
-- The update would make the row disappear from the view — blocked
```

---

## INSTEAD OF Triggers on Views

For complex views that can't be auto-updated, create an INSTEAD OF trigger:

```sql
-- Complex view (not auto-updatable)
CREATE VIEW order_with_customer AS
SELECT o.id, o.total, o.status, c.name AS customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id;

-- Create a trigger function to handle updates
CREATE OR REPLACE FUNCTION update_order_with_customer()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    UPDATE orders SET total = NEW.total, status = NEW.status
    WHERE id = NEW.id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER order_with_customer_update
INSTEAD OF UPDATE ON order_with_customer
FOR EACH ROW EXECUTE FUNCTION update_order_with_customer();
```

---

## Materialized Views — Cached Query Results

A materialized view stores the RESULT of a query on disk.
Unlike regular views, it doesn't re-run the query on each access.

```
Regular View:      query → executes every time → returns results
Materialized View: query → executed once → stores results → returns stored data
```

```sql
-- Create a materialized view
CREATE MATERIALIZED VIEW product_sales_summary AS
SELECT
    p.id,
    p.name,
    p.category,
    p.price,
    COALESCE(SUM(oi.quantity), 0) AS total_units_sold,
    COALESCE(SUM(oi.quantity * oi.unit_price), 0) AS total_revenue,
    COUNT(DISTINCT oi.order_id) AS order_count
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
GROUP BY p.id, p.name, p.category, p.price;

-- Query it — very fast, uses stored data
SELECT * FROM product_sales_summary ORDER BY total_revenue DESC;
SELECT category, SUM(total_revenue) FROM product_sales_summary GROUP BY category;
```

### Refreshing Materialized Views

```sql
-- Manual refresh (blocks reads until done)
REFRESH MATERIALIZED VIEW product_sales_summary;

-- Concurrent refresh (allows reads during refresh — requires UNIQUE index)
CREATE UNIQUE INDEX ON product_sales_summary(id);
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;
-- Concurrent refresh: old data available during refresh, no downtime

-- Full vs partial refresh
-- Materialized views always do a full refresh — no incremental option
-- (Timescale or custom solutions handle incremental refresh)
```

### Automating Refresh

```sql
-- Option 1: pg_cron (extension)
SELECT cron.schedule('refresh-sales', '0 * * * *',  -- every hour
    'REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary');

-- Option 2: Trigger-based refresh (after each order)
CREATE OR REPLACE FUNCTION refresh_product_sales()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;
    RETURN NULL;
END;
$$;

CREATE TRIGGER refresh_after_order_item
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH STATEMENT EXECUTE FUNCTION refresh_product_sales();
-- Warning: this runs on every single statement — expensive for high-traffic tables
-- Better: refresh on a schedule or after batch operations
```

---

## View vs Materialized View — Decision Guide

| Factor | Regular View | Materialized View |
|--------|-------------|-------------------|
| Storage | No (re-runs query) | Yes (stores results) |
| Query speed | Depends on underlying complexity | Fast (pre-computed) |
| Data freshness | Always current | Stale until refreshed |
| Write-through | Can be updatable | Never updatable |
| Best for | Simple abstraction, security, real-time data | Expensive reports, dashboards, aggregations |
| Index support | No (indexes on base tables) | Yes (add indexes to mat view) |

```sql
-- Use regular view for:
-- - Data that must be always up-to-date
-- - Simple JOINs with good indexes
-- - Row-level security policies
-- - Frequently updated data

-- Use materialized view for:
-- - Complex aggregations accessed frequently
-- - Dashboards/reports (can be minutes old)
-- - Slow queries with many JOINs
-- - Read-heavy, infrequently changed data
```

---

## Practical Materialized View Examples

**Daily Sales Dashboard:**
```sql
CREATE MATERIALIZED VIEW daily_sales AS
SELECT
    order_date                                      AS day,
    COUNT(*)                                        AS orders,
    COUNT(DISTINCT customer_id)                     AS unique_customers,
    ROUND(SUM(total), 2)                            AS revenue,
    ROUND(AVG(total), 2)                            AS avg_order_value,
    MAX(total)                                      AS largest_order
FROM orders
WHERE status = 'delivered'
GROUP BY order_date
ORDER BY order_date;

CREATE UNIQUE INDEX ON daily_sales(day);  -- for concurrent refresh

-- Refresh each night
-- SELECT cron.schedule('daily', '0 2 * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales');
```

**Customer Lifetime Value:**
```sql
CREATE MATERIALIZED VIEW customer_ltv AS
SELECT
    c.id             AS customer_id,
    c.name,
    c.email,
    c.country,
    c.created_at     AS joined_at,
    COUNT(o.id)      AS order_count,
    ROUND(SUM(o.total), 2)  AS lifetime_value,
    ROUND(AVG(o.total), 2)  AS avg_order_value,
    MIN(o.order_date)        AS first_order,
    MAX(o.order_date)        AS last_order,
    MAX(o.order_date) - MIN(o.order_date) AS customer_lifespan_days,
    CASE
        WHEN SUM(o.total) >= 1000 THEN 'VIP'
        WHEN SUM(o.total) >= 300  THEN 'Regular'
        WHEN COUNT(o.id)  >  0    THEN 'New'
        ELSE                           'Never Ordered'
    END AS segment
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status != 'cancelled'
GROUP BY c.id, c.name, c.email, c.country, c.created_at;

CREATE UNIQUE INDEX ON customer_ltv(customer_id);
```

---

## Schema Views (Inspecting the Database)

```sql
-- List all views
SELECT viewname, definition FROM pg_views WHERE schemaname = 'public';
\dv   -- in psql

-- List all materialized views
SELECT matviewname, definition FROM pg_matviews WHERE schemaname = 'public';
\dm   -- in psql

-- Check if a mat view needs refresh
SELECT
    matviewname,
    last_refresh,
    NOW() - last_refresh AS age
FROM pg_stat_user_tables
WHERE relname IN (SELECT matviewname FROM pg_matviews);

-- Find views that depend on a table
SELECT viewname
FROM pg_views
WHERE definition ILIKE '%orders%';
```

---

## Exercises

**Exercise 1:** Create a view called `order_summary` that shows for each order:
customer name, order date, status, number of items, and total.

**Exercise 2:** Create a materialized view for monthly revenue by category.
Add an index on it. Write the REFRESH command.

**Exercise 3:** Create a view that shows only active (not cancelled) orders
with WITH CHECK OPTION. Verify that trying to update an order to 'cancelled'
through the view fails.

**Exercise 4:** Create a view `employee_hierarchy` that shows each employee's
name, department, salary, and their manager's name.

---

## Key Takeaways

1. Views are saved SELECT queries — no data storage, always current
2. Use `CREATE OR REPLACE VIEW` to modify (preserves permissions)
3. Simple views are auto-updatable; complex views need INSTEAD OF triggers
4. `WITH CHECK OPTION` prevents modifications that violate the view's WHERE clause
5. Materialized views store results — fast reads, stale data until refreshed
6. `REFRESH MATERIALIZED VIEW CONCURRENTLY` doesn't block reads (requires UNIQUE index)
7. Add indexes to materialized views to speed up queries on them

---

## Next Lesson
[Lesson 14 — Stored Procedures & Functions](14-stored-procedures-and-functions.md)
