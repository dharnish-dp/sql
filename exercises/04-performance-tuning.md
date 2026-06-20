# Exercise 04 — Performance Tuning Lab

## Overview
Diagnose and fix deliberately slow queries. Each problem gives you a slow query
and asks you to identify why it's slow and make it fast.

## Skills Practiced
EXPLAIN ANALYZE, indexes, query rewriting, statistics

---

## Setup — Create Large Tables

```sql
-- Generate a large dataset for meaningful performance testing
CREATE TABLE perf_users (
    id         SERIAL PRIMARY KEY,
    email      TEXT NOT NULL UNIQUE,
    first_name TEXT,
    last_name  TEXT,
    country    TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE perf_orders (
    id          SERIAL PRIMARY KEY,
    user_id     INT REFERENCES perf_users(id),
    amount      NUMERIC(10,2),
    status      TEXT DEFAULT 'pending',
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Insert 100,000 users
INSERT INTO perf_users (email, first_name, last_name, country)
SELECT
    'user' || n || '@example.com',
    CASE (n % 5) WHEN 0 THEN 'Alice' WHEN 1 THEN 'Bob' WHEN 2 THEN 'Carol' WHEN 3 THEN 'Dave' ELSE 'Eve' END,
    'User' || n,
    CASE (n % 4) WHEN 0 THEN 'US' WHEN 1 THEN 'UK' WHEN 2 THEN 'CA' ELSE 'AU' END
FROM GENERATE_SERIES(1, 100000) n;

-- Insert 500,000 orders
INSERT INTO perf_orders (user_id, amount, status, created_at)
SELECT
    (RANDOM() * 100000 + 1)::INT,
    ROUND((RANDOM() * 1000 + 1)::NUMERIC, 2),
    CASE (n % 4) WHEN 0 THEN 'delivered' WHEN 1 THEN 'shipped' WHEN 2 THEN 'pending' ELSE 'cancelled' END,
    NOW() - (RANDOM() * 365 || ' days')::INTERVAL
FROM GENERATE_SERIES(1, 500000) n;

ANALYZE perf_users;
ANALYZE perf_orders;
```

---

## Problem 1 — Missing Index on FK

```sql
-- Run this and measure the time
\timing
SELECT u.email, COUNT(o.id) AS order_count
FROM perf_users u
JOIN perf_orders o ON o.user_id = u.id
WHERE u.country = 'US'
GROUP BY u.email;
```

**Task:**
1. Run `EXPLAIN ANALYZE` on this query
2. Identify the problem (look at the scan types)
3. Create the appropriate index(es)
4. Re-run EXPLAIN ANALYZE and compare times
5. What was the bottleneck?

---

## Problem 2 — Function Defeating Index

```sql
-- This is slow even with an index on email
CREATE INDEX idx_perf_users_email ON perf_users(email);

SELECT * FROM perf_users WHERE LOWER(email) = 'user500@example.com';
```

**Task:**
1. Check that the index exists
2. Run EXPLAIN ANALYZE — is the index being used?
3. Fix the query AND fix the index so future queries like this are fast
4. Run EXPLAIN ANALYZE again

---

## Problem 3 — Correlated Subquery

```sql
-- Count orders per user using a correlated subquery
SELECT
    u.email,
    (SELECT COUNT(*) FROM perf_orders WHERE user_id = u.id) AS orders,
    (SELECT SUM(amount) FROM perf_orders WHERE user_id = u.id AND status = 'delivered') AS revenue
FROM perf_users u
WHERE country = 'US'
LIMIT 100;
```

**Task:**
1. Run EXPLAIN ANALYZE — how many total rows are processed?
2. Rewrite using JOIN + GROUP BY (or CTE) to eliminate the correlated subqueries
3. Measure the improvement

---

## Problem 4 — Inefficient Status Filter

```sql
-- Find pending orders with high value
SELECT * FROM perf_orders
WHERE status = 'pending'
  AND amount > 500
ORDER BY amount DESC;
```

**Task:**
1. Run EXPLAIN ANALYZE
2. Create a partial index specifically for high-value pending orders
3. Verify the partial index is used
4. How much smaller is the partial index vs a full index on (status, amount)?
   Check with `pg_size_pretty(pg_relation_size('index_name'))`

---

## Problem 5 — Sort Overflow

```sql
-- Large sort operation
SELECT user_id, SUM(amount) AS total
FROM perf_orders
WHERE created_at > NOW() - INTERVAL '365 days'
GROUP BY user_id
ORDER BY total DESC;
```

**Task:**
1. Run EXPLAIN ANALYZE — look for "Sort Method: external merge  Disk:"
2. If you see disk usage, increase work_mem: `SET work_mem = '256MB';`
3. Re-run — is the sort now in memory?
4. What does "Sort Method: quicksort" vs "external merge" mean?

---

## Problem 6 — COUNT on Large Table

```sql
-- These are both slow on large tables
SELECT COUNT(*) FROM perf_orders;
SELECT COUNT(*) FROM perf_orders WHERE status = 'pending';
```

**Task:**
1. Time both queries
2. For the first: find the approximate count from `pg_class` (instant)
3. For the second: what index would help?
4. What's the trade-off between exact and approximate counts?

---

## Tuning Checklist

After completing each problem, review:

```
□ Did EXPLAIN ANALYZE show a Seq Scan on a large table?
  → Add an index on the filter/join column

□ Did estimated rows differ greatly from actual rows?
  → Run ANALYZE to update statistics

□ Was there "Sort Method: external merge  Disk:"?
  → Increase work_mem or add index for ORDER BY

□ Were there "Hash Batches: N > 1"?
  → Increase work_mem

□ Did a function wrap an indexed column?
  → Create an expression index

□ Was there a correlated subquery running once per outer row?
  → Rewrite with JOIN or CTE

□ Was there a Nested Loop with many loops on a large inner table?
  → Check for missing index on inner table's join column
```
