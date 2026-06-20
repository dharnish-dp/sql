# Lesson 09 — Indexes & Performance

## Goal
Understand how PostgreSQL indexes work internally, when to create them,
and how to read EXPLAIN ANALYZE output to diagnose slow queries.

## Prerequisites
- [Lesson 08](08-window-functions.md) — window functions

## After This Lesson You Will Be Able To
- Explain how a B-tree index works
- Create indexes for common query patterns
- Read EXPLAIN and EXPLAIN ANALYZE output
- Identify whether a query uses an index or a sequential scan
- Understand when indexes hurt performance

---

## The Problem Indexes Solve

Without an index on a table with 10 million rows:
```sql
SELECT * FROM users WHERE email = 'alice@example.com';
```
PostgreSQL reads EVERY row in the table to find Alice — this is a **sequential scan**.
With 10M rows, this might take seconds.

With a B-tree index on `email`:
- PostgreSQL looks up the index (like a book index)
- Finds the location of Alice's row directly
- Fetches it in microseconds

---

## How B-Tree Indexes Work

B-tree (Balanced Tree) is the default index type. Here's the structure:

```
                    [50 | 75]                  ← Root node
                   /    |    \
          [20|30]     [60|70]     [80|90]      ← Internal nodes
         /   |   \    / | \      /   |   \
      [10][25][35] [55][65][72] [77][85][95]   ← Leaf nodes (contain actual values + row pointers)
```

Properties:
- **Balanced**: every leaf is at the same depth — O(log n) lookups
- **Ordered**: values are sorted — supports range queries and ORDER BY
- **Pointers**: each leaf value points to the actual row location on disk (heap)

```sql
-- What happens with: WHERE price > 50
-- 1. Start at root
-- 2. Navigate: 50 is in the root, go right to [60|70]
-- 3. Navigate to leaf node starting at 55
-- 4. Scan forward (leaves are linked) until price > the range
-- Result: found in a few steps instead of scanning all rows
```

---

## Index Types

| Index Type | Use Case | PostgreSQL Command |
|---|---|---|
| **B-tree** | Equality, ranges, ORDER BY, LIKE 'prefix%' | Default |
| **Hash** | Equality only (=), faster than B-tree for that | `USING HASH` |
| **GIN** | Arrays, JSONB, full-text search | `USING GIN` |
| **GiST** | Geometry, ranges, text search (tsvector) | `USING GIST` |
| **BRIN** | Very large tables with naturally ordered data | `USING BRIN` |
| **SP-GiST** | Non-balanced structures (quad-trees, k-d trees) | `USING SPGIST` |

---

## Creating Indexes

```sql
-- Basic index syntax
CREATE INDEX index_name ON table_name (column1, column2, ...);

-- Index on a single column (most common)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_order_date ON orders(order_date);

-- Unique index (also enforces uniqueness constraint)
CREATE UNIQUE INDEX idx_customers_email ON customers(email);
-- UNIQUE constraint automatically creates a unique index

-- Index with a specific method
CREATE INDEX idx_products_category_hash ON products(category) USING HASH;

-- Partial index — only indexes rows matching a condition
-- Much smaller, much faster when querying only that subset
CREATE INDEX idx_active_orders ON orders(customer_id, order_date)
WHERE status IN ('pending', 'shipped');

-- Expression index — index on a computed value
CREATE INDEX idx_customers_lower_email ON customers(LOWER(email));
-- Enables: WHERE LOWER(email) = 'alice@example.com'

-- Covering index — include extra columns to avoid heap access
CREATE INDEX idx_orders_covering ON orders(customer_id)
INCLUDE (order_date, total, status);
-- Query can be satisfied entirely from the index — "index-only scan"

-- Concurrent index (doesn't lock the table during creation)
CREATE INDEX CONCURRENTLY idx_orders_date ON orders(order_date);
-- Takes longer but doesn't block reads/writes — use in production
```

---

## Composite Indexes — Column Order Matters

```sql
-- Composite index
CREATE INDEX idx_orders_compound ON orders(customer_id, status, order_date);

-- This index supports:
WHERE customer_id = 1                            -- ✅ leftmost column
WHERE customer_id = 1 AND status = 'delivered'   -- ✅ leftmost + next
WHERE customer_id = 1 AND status = 'delivered' AND order_date > '2024-01-01'  -- ✅ all three

-- This index does NOT support:
WHERE status = 'delivered'                       -- ✗ skipped customer_id
WHERE order_date > '2024-01-01'                  -- ✗ skipped customer_id + status

-- Rule: index is used from left to right, can't skip columns
-- (For equality conditions you can skip, but range conditions stop index use)

WHERE customer_id = 1 AND order_date > '2024-01-01'
-- customer_id used (equality), order_date used (range on last column)
-- BUT: status is skipped — still works, just less efficient

-- Best practice: put equality columns first, range columns last
-- Good:  (customer_id, status, order_date)   → equality, equality, range
-- Bad:   (order_date, customer_id, status)   → range first blocks the rest
```

---

## EXPLAIN — Reading the Query Plan

```sql
-- EXPLAIN shows the plan (no query execution)
EXPLAIN SELECT * FROM orders WHERE customer_id = 1;

-- EXPLAIN ANALYZE actually runs the query and shows real timing
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 1;

-- EXPLAIN with full output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT) 
SELECT * FROM orders WHERE customer_id = 1;

-- EXPLAIN with JSON format (for programmatic parsing)
EXPLAIN (FORMAT JSON, ANALYZE) SELECT * FROM orders WHERE customer_id = 1;
```

### Reading EXPLAIN Output

```
                               QUERY PLAN
----------------------------------------------------------------------
 Index Scan using idx_orders_customer_id on orders  (cost=0.14..8.17 rows=2 width=48)
                                                    (actual time=0.045..0.048 rows=2 loops=1)
   Index Cond: (customer_id = 1)
 Planning Time: 0.132 ms
 Execution Time: 0.068 ms
```

#### Decoding Each Part

**Node type:**
```
Sequential Scan        — reads all rows (no index)
Index Scan             — reads index then heap (for each row)
Index Only Scan        — reads only the index (covering index)
Bitmap Heap Scan       — reads many rows via index, then heap in bulk
Bitmap Index Scan      — builds a bitmap of matching rows
Hash Join              — joins via hash table
Nested Loop            — for each row in outer, probe inner
Merge Join             — joins two sorted inputs
Sort                   — sorts rows
Hash Aggregate         — aggregates via hash table
GroupAggregate         — aggregates pre-sorted rows
Gather / Gather Merge  — parallel query coordinator
Limit                  — applies LIMIT/OFFSET
```

**Cost:**
```
cost=0.14..8.17
      ↑    ↑
      |    Total cost to return all rows
      Startup cost (before first row is returned)
```
Cost is in arbitrary units (not ms). Used to compare plans, not to measure time.

**Rows:**
```
rows=2
```
Estimated number of rows. Compare with `actual...rows=2` — if very different, statistics are stale.

**Width:**
```
width=48
```
Estimated average row width in bytes.

**Actual time:**
```
actual time=0.045..0.048
            ↑       ↑
            Time to first row    Time to last row (in ms)
```

**Loops:**
```
loops=1
```
How many times this node executed. For nested loops, inner node may loop many times.

---

## Scan Types — What They Mean

```sql
-- Sequential Scan — no usable index, reads all rows
--   When: no index, or selectivity is low (query returns >10% of rows)
Seq Scan on products  (cost=0.00..1.10 rows=10 width=...)

-- Index Scan — uses index to find rows, then fetches from heap
--   When: selective query with a B-tree index
Index Scan using idx_orders_customer_id on orders

-- Index Only Scan — index has all needed columns (covering index)
--   Fastest for read queries
Index Only Scan using idx_orders_covering on orders

-- Bitmap Index + Bitmap Heap Scan
--   When: moderate number of rows to return (between 1% and ~10%)
--   Builds a bitmap of page locations, reads heap pages in order
Bitmap Index Scan on idx_orders_status
Bitmap Heap Scan on orders
```

---

## Why the Planner Might Ignore Your Index

```sql
-- 1. Query returns too many rows (low selectivity)
-- If >10% of rows match, a sequential scan is faster
EXPLAIN SELECT * FROM orders WHERE status = 'delivered';
-- If most orders are 'delivered', planner may choose Seq Scan

-- 2. Statistics are stale
-- Run ANALYZE to update statistics
ANALYZE orders;
-- Or let autovacuum handle it

-- 3. Table is too small
-- For tiny tables, sequential scan is always faster than index lookup
-- Don't worry about missing indexes on tables with < 1000 rows

-- 4. Data type mismatch
-- Index on INT, but query compares with TEXT
EXPLAIN SELECT * FROM orders WHERE customer_id = '1';   -- '1' is text!
-- May not use index. Always match types.

-- 5. Function on column
-- Index on salary, but query uses function:
EXPLAIN SELECT * FROM employees WHERE UPPER(name) = 'ALICE';
-- Won't use idx_employees_name — create expression index instead:
CREATE INDEX idx_employees_upper_name ON employees(UPPER(name));
```

---

## When to Create an Index

### Create indexes on:

```sql
-- Primary keys (created automatically)
-- Foreign keys (not automatic in PostgreSQL — create manually)
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Columns frequently used in WHERE
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_order_date ON orders(order_date);

-- Columns used in ORDER BY on large tables
CREATE INDEX idx_orders_total_desc ON orders(total DESC);

-- Columns used in JOIN ON clauses (foreign keys)

-- Columns used in GROUP BY on large tables

-- Columns with high cardinality (many unique values)
-- e.g., email, user_id, timestamp — good index candidates
-- e.g., boolean, status with 2-3 values — usually not worth it
```

### When indexes HURT:

```sql
-- 1. High-write tables
-- Every INSERT/UPDATE/DELETE must update all indexes
-- A table with 10 indexes takes ~10x longer to write to

-- 2. Small tables
-- Sequential scan is faster for < ~1000 rows

-- 3. Low-cardinality columns
-- Boolean column: only 2 values — index barely helps
-- Status column with 3 values: ~33% selectivity — probably not useful

-- 4. Columns never queried
-- Unused indexes waste space and slow writes
-- Find unused indexes:
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,    -- how many times used
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Index Maintenance

```sql
-- Check index sizes
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename = 'orders'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Check table and index sizes together
SELECT
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
    pg_size_pretty(pg_relation_size(relid)) AS table_size,
    pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) AS indexes_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- Rebuild a bloated index
REINDEX INDEX idx_orders_customer_id;
REINDEX TABLE orders;           -- rebuild all indexes on a table

-- Drop an index
DROP INDEX idx_orders_status;
DROP INDEX IF EXISTS idx_orders_status;
DROP INDEX CONCURRENTLY idx_orders_status;  -- non-blocking

-- List all indexes for a table
\d orders
-- Or:
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';
```

---

## Table Statistics and VACUUM

```sql
-- PostgreSQL maintains statistics about table data for the planner
-- These get stale as data changes

-- Update statistics manually
ANALYZE products;
ANALYZE;  -- all tables in database

-- VACUUM removes dead rows (from deleted/updated records)
VACUUM orders;
VACUUM ANALYZE orders;  -- vacuum + update statistics
VACUUM FULL orders;     -- full table rewrite (locks table — use carefully)

-- autovacuum runs automatically, but sometimes you need to force it
-- after a large data load

-- Check table statistics
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    last_analyze,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## Practical EXPLAIN ANALYZE Workflow

```sql
-- Step 1: Run EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS) 
SELECT c.name, SUM(o.total)
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.order_date > '2024-01-01'
GROUP BY c.name;

-- Step 2: Look for these warning signs:
-- • Seq Scan on large table (should be Index Scan)
-- • Estimated rows WAY off from actual rows
-- • High "cost" or "actual time" nodes
-- • Hash Join with large "Batches" (hash overflows to disk)
-- • Sort with "Disk: N MB" (sort overflowed to disk)

-- Step 3: Fix
-- Seq Scan → add index on the WHERE/JOIN column
-- Wrong row estimates → run ANALYZE
-- Sort on disk → increase work_mem: SET work_mem = '256MB';
-- Hash batches → increase work_mem

-- Step 4: Run EXPLAIN ANALYZE again to confirm improvement
```

---

## Exercises

**Exercise 1:** Create appropriate indexes for this query, then verify they're used
with EXPLAIN ANALYZE:
```sql
SELECT c.name, o.total, o.order_date
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'delivered'
  AND o.order_date > '2024-01-01'
ORDER BY o.total DESC;
```

**Exercise 2:** Find the EXPLAIN plan for a sequential scan vs an index scan.
Run the same query before and after creating an index. Compare timing.

**Exercise 3:** Create a partial index that only covers active (non-cancelled) orders.
What are the trade-offs compared to a full index?

**Exercise 4:** What does EXPLAIN show for this? Why might the index not be used?
```sql
SELECT * FROM employees WHERE salary::TEXT LIKE '9%';
```

---

## Key Takeaways

1. B-tree indexes are O(log n) lookups vs O(n) sequential scans
2. Composite index columns must be in left-to-right order — equality first, range last
3. EXPLAIN ANALYZE shows estimated vs actual rows — big differences = stale stats
4. `Seq Scan` on large tables = missing index (or very low selectivity)
5. Create indexes on foreign key columns — PostgreSQL doesn't do this automatically
6. Indexes slow down writes — don't over-index write-heavy tables
7. Use `CREATE INDEX CONCURRENTLY` in production to avoid table locks
8. Run `ANALYZE` after large data loads to update planner statistics

---

## Next Lesson
[Lesson 10 — Transactions & ACID](10-transactions-and-acid.md)
