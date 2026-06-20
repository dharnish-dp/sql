# Lesson 15 — Query Optimization

## Goal
Learn to systematically find slow queries, read EXPLAIN ANALYZE output deeply,
and rewrite queries to be dramatically faster.

## Prerequisites
- [Lesson 14](14-stored-procedures-and-functions.md) — functions and triggers

## After This Lesson You Will Be Able To
- Find slow queries using pg_stat_statements
- Read EXPLAIN ANALYZE output at expert level
- Identify the common causes of slow queries
- Rewrite common slow patterns into fast ones
- Tune PostgreSQL configuration settings

---

## The Optimization Workflow

```
1. Identify slow queries   →  pg_stat_statements, slow query log
2. Reproduce the problem   →  run the query yourself with EXPLAIN ANALYZE
3. Understand the plan     →  read the plan tree
4. Hypothesize the fix     →  missing index? bad join order? stale stats?
5. Apply the fix           →  create index, rewrite query, update config
6. Verify improvement      →  run EXPLAIN ANALYZE again, compare
7. Monitor                 →  confirm it stays fast in production
```

---

## Finding Slow Queries

### pg_stat_statements (Extension)

```sql
-- Enable the extension (add to postgresql.conf: shared_preload_libraries = 'pg_stat_statements')
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 slowest queries by total time
SELECT
    query,
    calls,
    ROUND(total_exec_time::NUMERIC, 2) AS total_ms,
    ROUND(mean_exec_time::NUMERIC, 2)  AS avg_ms,
    ROUND(max_exec_time::NUMERIC, 2)   AS max_ms,
    ROUND(stddev_exec_time::NUMERIC, 2) AS stddev_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Most frequently called queries
SELECT query, calls, ROUND(mean_exec_time, 2) AS avg_ms
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Queries with high average time (potential outliers)
SELECT query, calls, ROUND(mean_exec_time, 2) AS avg_ms
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### Slow Query Log

```sql
-- In postgresql.conf:
-- log_min_duration_statement = 1000  -- log queries taking > 1000ms
-- log_min_duration_statement = 0     -- log ALL queries (very verbose)

-- Or set dynamically for the current session:
SET log_min_duration_statement = 500;

-- Check the PostgreSQL log file:
-- /var/log/postgresql/postgresql-16-main.log (Linux)
-- /usr/local/var/log/postgresql@16.log (macOS)
```

---

## EXPLAIN ANALYZE — Deep Reading

```sql
-- Full EXPLAIN with all options
EXPLAIN (
    ANALYZE,      -- actually run the query
    BUFFERS,      -- show buffer hit/miss counts
    VERBOSE,      -- show column names and function calls
    COSTS,        -- show cost estimates (default ON)
    TIMING,       -- show timing per node (default ON with ANALYZE)
    FORMAT TEXT   -- text, json, xml, yaml
)
SELECT c.name, SUM(o.total) AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'delivered'
GROUP BY c.name
ORDER BY revenue DESC;
```

### A Full EXPLAIN ANALYZE Output

```
                                QUERY PLAN
---------------------------------------------------------------------------------
 Sort  (cost=37.68..37.70 rows=10 width=40)
       (actual time=0.312..0.313 rows=5 loops=1)
   Sort Key: (sum(o.total)) DESC
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=37.30..37.48 rows=10 width=40)
                      (actual time=0.294..0.300 rows=5 loops=1)
         Group Key: c.name
         Batches: 1  Memory Usage: 24kB
         ->  Hash Join  (cost=1.23..36.85 rows=30 width=24)
                        (actual time=0.048..0.271 rows=9 loops=1)
               Hash Cond: (o.customer_id = c.id)
               ->  Seq Scan on orders o  (cost=0.00..1.12 rows=3 width=12)
                                         (actual time=0.012..0.018 rows=3 loops=1)
                     Filter: ((status)::text = 'delivered'::text)
                     Rows Removed by Filter: 7
               ->  Hash  (cost=1.10..1.10 rows=10 width=20)
                         (actual time=0.024..0.024 rows=10 loops=1)
                     Buckets: 1024  Batches: 1  Memory Usage: 9kB
                     ->  Seq Scan on customers c  (cost=0.00..1.10 rows=10 width=20)
                                                  (actual time=0.009..0.012 rows=10 loops=1)
 Planning Time: 0.342 ms
 Execution Time: 0.381 ms
```

### Reading the Tree (Bottom-Up)

PostgreSQL plans are trees, evaluated from leaves (innermost) to root.

```
Step 1: Read customers (Seq Scan → 10 rows)
Step 2: Build Hash table from customers
Step 3: Read orders (Seq Scan, filtered to 'delivered' → 3 rows)
Step 4: Hash Join: for each order, probe the hash table
Step 5: HashAggregate: GROUP BY c.name, compute SUM(o.total)
Step 6: Sort: ORDER BY revenue DESC
```

### Key Metrics to Check

```
1. Estimated rows vs Actual rows
   (cost=37.30..37.48 rows=10)  ← estimate: 10 rows
   (actual time=0.294..0.300 rows=5)  ← actual: 5 rows
   
   If off by 10x+: statistics are stale → run ANALYZE
   
2. Total time
   Execution Time: 0.381 ms  ← great
   Execution Time: 45000 ms  ← problem!
   
3. Buffers (with BUFFERS option):
   Buffers: shared hit=500, read=5000
   ↑ hit = from RAM (fast)  ↑ read = from disk (slow)
   High read count → table not in cache or sequential scan on large table
   
4. Sort Method:
   Sort Method: quicksort  Memory: 25kB   ← all in memory, fast
   Sort Method: external merge  Disk: 10MB  ← overflow to disk, slow!
   Fix: SET work_mem = '100MB'; or add index for ORDER BY
   
5. Batches:
   Hash Join Batches: 1   ← all in memory, fast
   Hash Join Batches: 8   ← overflow to disk, slow!
   Fix: SET work_mem = '256MB';
```

---

## Common Slow Query Patterns and Fixes

### Pattern 1: Sequential Scan on Large Table

```sql
-- SLOW: Sequential scan on 10M row table
EXPLAIN SELECT * FROM events WHERE user_id = 12345;
-- -> Seq Scan on events (cost=0.00..250000.00 rows=50 width=100)

-- FIX: Create an index
CREATE INDEX idx_events_user_id ON events(user_id);

-- FAST: Index Scan
EXPLAIN SELECT * FROM events WHERE user_id = 12345;
-- -> Index Scan using idx_events_user_id on events
--    Index Cond: (user_id = 12345)
```

### Pattern 2: Missing Foreign Key Index

```sql
-- SLOW: Order details query
EXPLAIN SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;
-- Nested Loop → Seq Scan on customers for each order row!

-- CHECK: Is there an index on orders.customer_id?
\d orders
-- If no index: every order does a full scan of customers!

-- FIX:
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

### Pattern 3: Function on Indexed Column

```sql
-- SLOW: wrapping a column in a function defeats the index
EXPLAIN SELECT * FROM customers WHERE LOWER(email) = 'alice@example.com';
-- Seq Scan — can't use index on email because LOWER() is applied

-- FIX A: Use ILIKE (PostgreSQL only, case-insensitive)
SELECT * FROM customers WHERE email ILIKE 'alice@example.com';

-- FIX B: Expression index
CREATE INDEX idx_customers_lower_email ON customers(LOWER(email));
SELECT * FROM customers WHERE LOWER(email) = 'alice@example.com';
-- Now uses Index Scan
```

### Pattern 4: Type Mismatch

```sql
-- orders.customer_id is INT, but query uses TEXT
EXPLAIN SELECT * FROM orders WHERE customer_id = '1';  -- '1' is text
-- May cause Seq Scan: PostgreSQL can't use index due to implicit cast

-- FIX: match data types exactly
SELECT * FROM orders WHERE customer_id = 1;  -- integer literal
```

### Pattern 5: Stale Statistics

```sql
-- SLOW after a large data load: planner makes wrong estimates
-- Estimate: rows=10, Actual: rows=1,000,000

-- FIX: update statistics
ANALYZE orders;        -- updates statistics for orders table
ANALYZE;               -- updates all tables in the database

-- After a massive INSERT:
INSERT INTO events SELECT ... FROM large_source;  -- 10M rows
ANALYZE events;                                    -- update stats immediately
```

### Pattern 6: COUNT(*) on Large Table

```sql
-- SLOW: exact count on a 100M row table
SELECT COUNT(*) FROM events;  -- Seq Scan, reads every row

-- FAST APPROXIMATION: use statistics table
SELECT reltuples::BIGINT AS approx_count
FROM pg_class
WHERE relname = 'events';
-- Not exact but instant — good enough for display purposes

-- FIX for exact count: maintain a counter table
CREATE TABLE table_counts (
    tablename TEXT PRIMARY KEY,
    count     BIGINT NOT NULL DEFAULT 0
);
-- Update via trigger
```

### Pattern 7: N+1 Query Pattern

```sql
-- SLOW pattern (in application code):
-- For each customer, run a query to get their orders
-- 10 customers = 11 queries (1 for customers + 10 for orders)

-- FIX: Use JOIN to get everything in one query
SELECT
    c.id, c.name,
    COUNT(o.id) AS orders,
    SUM(o.total) AS revenue
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;
-- 1 query instead of 11
```

### Pattern 8: OR on Different Columns

```sql
-- SLOW: OR prevents efficient index use
SELECT * FROM products
WHERE category = 'Electronics' OR price > 500;
-- Can't use a single index for this efficiently

-- FIX: UNION
SELECT * FROM products WHERE category = 'Electronics'
UNION
SELECT * FROM products WHERE price > 500;
-- Each branch can use its own index
```

### Pattern 9: SELECT *

```sql
-- BAD: fetches all columns including large TEXT, JSONB fields
SELECT * FROM products WHERE price > 100;

-- GOOD: fetch only what you need
SELECT id, name, price FROM products WHERE price > 100;

-- Significant for:
-- - Wide tables with many columns
-- - Tables with large TEXT or JSONB columns (large network transfer)
-- - Can enable "Index Only Scan" if covering index exists
```

### Pattern 10: Implicit Cartesian Product

```sql
-- SLOW: missing JOIN condition
SELECT * FROM orders, customers;
-- 10 orders × 10 customers = 100 rows (Cartesian product)
-- Probably a mistake, but can cause OOM on large tables

-- FIX: always specify your JOIN conditions
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;
```

---

## Rewriting Slow Queries

### Slow subquery → Fast JOIN

```sql
-- SLOW: correlated subquery runs once per outer row
SELECT name, salary
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department = e.department  -- correlated
);

-- FAST: compute department averages once with CTE
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary
FROM employees e
JOIN dept_avg d ON d.department = e.department
WHERE e.salary > d.avg_salary;
```

### Slow NOT IN → Fast NOT EXISTS

```sql
-- SLOW (with NULL issue):
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- Also slow because of the NULL problem — may be wrong!

-- FAST and NULL-safe:
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);

-- Also fast: LEFT JOIN + NULL check
SELECT c.* FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

### Slow aggregation → Pre-aggregate

```sql
-- SLOW: aggregate on the fly in every query
SELECT
    c.name,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) AS order_count,
    (SELECT SUM(total) FROM orders WHERE customer_id = c.id AND status='delivered') AS revenue
FROM customers c;
-- Runs 2 subqueries per customer row

-- FAST: single aggregation JOIN
SELECT
    c.name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total) FILTER (WHERE o.status='delivered'), 0) AS revenue
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;
```

---

## PostgreSQL Configuration for Performance

```sql
-- View current settings
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;

-- Key settings to tune (edit postgresql.conf):
-- shared_buffers = 25% of RAM         -- PostgreSQL's main cache
-- effective_cache_size = 75% of RAM   -- hint to planner about OS cache
-- work_mem = RAM / max_connections / 2 -- per-sort / per-hash operation
-- maintenance_work_mem = RAM * 0.1    -- for VACUUM, CREATE INDEX, etc.
-- max_connections = 100               -- reduce to allow larger work_mem

-- Set session-level (for testing):
SET work_mem = '256MB';           -- more memory for sorts/hashes
SET enable_seqscan = OFF;         -- force index scan (for testing)
SET enable_hashjoin = OFF;        -- force merge or nested loop join
SET enable_mergejoin = OFF;       -- force hash or nested loop join

-- Reset:
RESET work_mem;
RESET enable_seqscan;
```

---

## Query Hints (PostgreSQL Style)

PostgreSQL doesn't have HINT syntax like Oracle/MySQL, but you can guide the planner:

```sql
-- Force index use: disable seq scan (test only)
SET enable_seqscan = OFF;
SELECT * FROM orders WHERE customer_id = 1;
SET enable_seqscan = ON;

-- Use pg_hint_plan extension (if installed)
/*+ IndexScan(orders idx_orders_customer_id) */
SELECT * FROM orders WHERE customer_id = 1;

-- Set parallel query workers
SET max_parallel_workers_per_gather = 4;
SELECT COUNT(*) FROM large_table;
-- Look for "Parallel Seq Scan" or "Parallel Hash Join" in plan

-- Disable parallel query (for testing)
SET max_parallel_workers_per_gather = 0;
```

---

## Exercises

**Exercise 1:** Find the slowest 5 queries in your practice database using
pg_stat_statements. Then run EXPLAIN ANALYZE on the slowest one.

**Exercise 2:** Take this query and make it faster using appropriate indexes:
```sql
SELECT c.name, o.order_date, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'delivered'
  AND o.order_date > '2024-01-01'
ORDER BY o.total DESC;
```

**Exercise 3:** Rewrite this query to avoid the correlated subquery:
```sql
SELECT name, price,
    (SELECT AVG(price) FROM products WHERE category = p.category) AS cat_avg
FROM products p;
```

**Exercise 4:** EXPLAIN ANALYZE this query and identify what's happening:
```sql
SELECT * FROM products WHERE ROUND(price, 0) = 30;
```
Why won't an index on `price` help? How would you fix it?

---

## Key Takeaways

1. Use `pg_stat_statements` to find the worst queries by total_exec_time
2. EXPLAIN ANALYZE: compare estimated rows vs actual rows — big gap = stale stats
3. `Sort Method: external merge  Disk:` = sort overflowed to disk → increase work_mem
4. Functions on indexed columns defeat the index — use expression indexes instead
5. `NOT IN (subquery)` is NULL-unsafe AND slow — use NOT EXISTS instead
6. Always index foreign key columns (PostgreSQL doesn't do it automatically)
7. Pre-aggregate data in CTEs instead of running correlated subqueries per row
8. `shared_buffers = 25% of RAM` is the single most impactful config change

---

## Next Lesson
[Lesson 16 — PostgreSQL Power Features](16-postgresql-power-features.md)
