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

**Why you need this:** [Lesson 09](09-indexes-and-performance.md) taught
you to read a *single* query's `EXPLAIN` output. But in a real
application with hundreds of different queries running, the first
problem isn't "how do I read this plan" — it's "**which** query, out of
hundreds, is actually the one hurting performance?" This lesson starts
there, then builds on Lesson 09's reading skills once you've found the
culprit.

```
1. Identify slow queries   →  pg_stat_statements, slow query log
2. Reproduce the problem   →  run the query yourself with EXPLAIN ANALYZE
3. Understand the plan     →  read the plan tree (Lesson 09's method)
4. Hypothesize the fix     →  missing index? bad join order? stale stats?
5. Apply the fix           →  create index, rewrite query, update config
6. Verify improvement      →  run EXPLAIN ANALYZE again, compare
7. Monitor                 →  confirm it stays fast in production
```

---

## Finding Slow Queries

### pg_stat_statements (Extension)

**Why you need this:** you can't run `EXPLAIN ANALYZE` on hundreds of
queries by hand to find the slow one. `pg_stat_statements` is an
extension that automatically tracks *every* query Postgres runs, along
with how often and how long each one takes — turning "which query is
the problem?" from a guess into a ranked list.

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
```

**What each column tells you:** `calls` = how many times this exact
query shape has run since tracking started; `total_ms` = the sum of
every one of those runs added together; `avg_ms` = `total_ms / calls`,
the typical cost of one run; `max_ms` = the single worst run; `stddev_ms`
= how much run times vary (high stddev means the same query is
sometimes fast, sometimes slow — often a sign of caching effects or lock
contention, worth investigating separately).

Sample output:
| query | calls | total_ms | avg_ms | max_ms | stddev_ms | rows |
|---|---|---|---|---|---|---|
| `SELECT * FROM orders WHERE customer_id = $1` | 48210 | 96420.50 | 2.00 | 350.10 | 8.40 | 48210 |
| `SELECT c.name, SUM(o.total) FROM ... GROUP BY ...` | 120 | 18000.00 | 150.00 | 210.30 | 12.10 | 120 |

**Why `ORDER BY total_exec_time`, not `avg_exec_time`, for this first
query:** a query that runs 50,000 times at 2ms each (`total = 100,000ms`)
is a bigger overall burden on the database than one that runs 10 times
at 50ms each (`total = 500ms`) — even though the second looks "slower"
per-call. Sorting by total time finds what's actually costing the most
cumulative time, which is usually the better place to start optimizing.

```sql
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
```
These two answer different questions than the first query: "what runs
constantly, even if each call is cheap" (worth optimizing because volume
adds up) vs. "what's slow on a per-call basis, ignoring one-off
queries that only ran once or twice" (the `calls > 10` filter excludes
rare queries whose one slow run might just be noise, not a real pattern).

```sql
-- Reset statistics
SELECT pg_stat_statements_reset();
```
Clears all tracked stats back to zero — useful right after you've
applied a fix, so the next round of numbers reflects only post-fix
behavior, not blended with the old slow numbers.

### Slow Query Log

**Why you need this:** `pg_stat_statements` tracks aggregated stats
per *query shape*. Sometimes you need to see an actual, specific,
individual slow query as it happened — the log is for that.

```sql
-- In postgresql.conf:
-- log_min_duration_statement = 1000  -- log queries taking > 1000ms
-- log_min_duration_statement = 0     -- log ALL queries (very verbose)

-- Or set dynamically for the current session:
SET log_min_duration_statement = 500;
```
**What this does:** any query that takes longer than the given number
of milliseconds gets written, in full, to the Postgres log file. Setting
it to `0` logs literally everything — useful briefly while debugging,
but generates enormous log volume if left on in a busy production
database, so it's normally set to a threshold like `500` or `1000`.

```sql
-- Check the PostgreSQL log file:
-- /var/log/postgresql/postgresql-16-main.log (Linux)
-- /usr/local/var/log/postgresql@16.log (macOS)
```

---

## EXPLAIN ANALYZE — Deep Reading

**Why you need this:** you now have a specific slow query (found via
`pg_stat_statements` above). This is where [Lesson 09](09-indexes-and-performance.md)'s
"How to Read Any EXPLAIN Plan Yourself" method actually gets applied —
this section walks through one full, real, multi-node plan using that
exact same method, on a query complex enough to need every step of it.

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

**Why you need this:** most real slow queries fall into a small number
of *repeating* shapes. Once you recognize a pattern, you already know
the fix — you don't have to reason from scratch every time. Each pattern
below traces back to a specific rule from [Lesson 09](09-indexes-and-performance.md).

### Pattern 1: Sequential Scan on Large Table

This is the "why the planner might ignore your index" / "missing index
entirely" case from Lesson 09, applied concretely:

```sql
-- SLOW: Sequential scan on 10M row table
EXPLAIN SELECT * FROM events WHERE user_id = 12345;
-- -> Seq Scan on events (cost=0.00..250000.00 rows=50 width=100)
```
**Why this is slow:** no index exists on `user_id`, so Postgres has no
choice but to check all 10 million rows one by one to find the ~50 that
match — exactly the "book with no index, read every page" problem from
Lesson 09's opening section.

```sql
-- FIX: Create an index
CREATE INDEX idx_events_user_id ON events(user_id);

-- FAST: Index Scan
EXPLAIN SELECT * FROM events WHERE user_id = 12345;
-- -> Index Scan using idx_events_user_id on events
--    Index Cond: (user_id = 12345)
```
Same query, same data — the only difference is Postgres can now jump
almost straight to the matching rows instead of scanning everything.

### Pattern 2: Missing Foreign Key Index

```sql
-- SLOW: Order details query
EXPLAIN SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;
-- Nested Loop → Seq Scan on customers for each order row!
```
**Why this is slow:** recall from [Lesson 09](09-indexes-and-performance.md)
that Postgres never auto-creates an index on a foreign key column. A
`Nested Loop` join here means: for *every single order row*, Postgres
re-scans the entire `customers` table looking for a match — the cost
multiplies with every order added.

```sql
-- CHECK: Is there an index on orders.customer_id?
\d orders
-- If no index: every order does a full scan of customers!

-- FIX:
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```
With the index in place, each order's lookup becomes a fast, direct
index hit instead of a full re-scan — the join goes from "re-read the
whole table N times" to "look up N times, quickly."

### Pattern 3: Function on Indexed Column

```sql
-- SLOW: wrapping a column in a function defeats the index
EXPLAIN SELECT * FROM customers WHERE LOWER(email) = 'alice@example.com';
-- Seq Scan — can't use index on email because LOWER() is applied
```
**Why this is slow:** this is the exact "function on column" reason
from Lesson 09's "Why the Planner Might Ignore Your Index" — a plain
index on `email` stores the *raw* values, but the query is searching for
the *result of a function call*, which the raw index can't match against.

```sql
-- FIX A: Use ILIKE (PostgreSQL only, case-insensitive)
SELECT * FROM customers WHERE email ILIKE 'alice@example.com';

-- FIX B: Expression index
CREATE INDEX idx_customers_lower_email ON customers(LOWER(email));
SELECT * FROM customers WHERE LOWER(email) = 'alice@example.com';
-- Now uses Index Scan
```
Fix A sidesteps the function entirely (`ILIKE` does case-insensitive
matching natively, no `LOWER()` needed). Fix B — the expression index —
indexes the function's *output* directly, so `LOWER(email)` in the query
now has a matching indexed value to compare against.

### Pattern 4: Type Mismatch

```sql
-- orders.customer_id is INT, but query uses TEXT
EXPLAIN SELECT * FROM orders WHERE customer_id = '1';  -- '1' is text
-- May cause Seq Scan: PostgreSQL can't use index due to implicit cast
```
**Why this is slow:** the index on `customer_id` stores integers. Comparing
against the text `'1'` forces Postgres to convert types before comparing
— depending on the exact types involved, this can prevent the index
from being used at all, the same "data type mismatch" reason from
Lesson 09.

```sql
-- FIX: match data types exactly
SELECT * FROM orders WHERE customer_id = 1;  -- integer literal
```
No conversion needed — the literal already matches the column's real type.

### Pattern 5: Stale Statistics

```sql
-- SLOW after a large data load: planner makes wrong estimates
-- Estimate: rows=10, Actual: rows=1,000,000
```
**Why this is slow:** this is Lesson 09's "statistics are stale" case —
the planner's row-count predictions come from cached statistics, not a
live count. After a big data load, the planner may still be reasoning
about the *old*, much smaller table size, and pick a bad plan (like a
nested loop that was fine for 10 rows but disastrous for 1,000,000).

```sql
-- FIX: update statistics
ANALYZE orders;        -- updates statistics for orders table
ANALYZE;               -- updates all tables in the database

-- After a massive INSERT:
INSERT INTO events SELECT ... FROM large_source;  -- 10M rows
ANALYZE events;                                    -- update stats immediately
```
Refreshing statistics gives the planner accurate numbers to reason
from, so it picks a plan suited to the table's *actual* current size.

### Pattern 6: COUNT(*) on Large Table

```sql
-- SLOW: exact count on a 100M row table
SELECT COUNT(*) FROM events;  -- Seq Scan, reads every row
```
**Why this is slow:** because of MVCC ([Lesson 21](21-database-engine-internals.md)),
Postgres has no single stored "row count" it can just look up — row
visibility depends on which transaction is asking. An exact `COUNT(*)`
genuinely has to check every row to know which ones are currently visible.

```sql
-- FAST APPROXIMATION: use statistics table
SELECT reltuples::BIGINT AS approx_count
FROM pg_class
WHERE relname = 'events';
-- Not exact but instant — good enough for display purposes
```
`reltuples` is a cached estimate from the last `ANALYZE`, not a live
count — instant to read, but can drift from the true number over time.
Good enough for "about 100 million events" on a dashboard; wrong for
anything requiring exactness.

```sql
-- FIX for exact count: maintain a counter table
CREATE TABLE table_counts (
    tablename TEXT PRIMARY KEY,
    count     BIGINT NOT NULL DEFAULT 0
);
-- Update via trigger
```
Instead of counting on every read, maintain a running total that's
incremented/decremented by a trigger on every insert/delete — trading a
small amount of extra work on writes for an instant, exact read.

### Pattern 7: N+1 Query Pattern

```sql
-- SLOW pattern (in application code):
-- For each customer, run a query to get their orders
-- 10 customers = 11 queries (1 for customers + 10 for orders)
```
**Why this is slow:** this isn't a bad index or stale stats — it's an
application-code mistake: looping over customers *in your app*, and
firing a separate database round-trip for each one's orders. Every
round-trip has network latency on top of the query itself; 10 customers
means paying that latency 11 times instead of once.

```sql
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
The `LEFT JOIN` + `GROUP BY` pattern (from [Lesson 06](06-joins-complete.md)
and [Lesson 05](05-aggregations.md)) gets every customer's order stats
in a single round-trip — one network hop, one query, done.

### Pattern 8: OR on Different Columns

```sql
-- SLOW: OR prevents efficient index use
SELECT * FROM products
WHERE category = 'Electronics' OR price > 500;
```
**Why this is slow:** even with separate indexes on `category` and
`price`, a single index scan can only efficiently satisfy *one*
condition at a time. An `OR` across two unrelated columns often forces
Postgres to either scan the whole table, or awkwardly combine two
partial index scans — neither is as clean as matching one condition.

```sql
-- FIX: UNION
SELECT * FROM products WHERE category = 'Electronics'
UNION
SELECT * FROM products WHERE price > 500;
```
Splitting into two separate queries lets **each half use its own index
cleanly** — the `category = 'Electronics'` half uses the category
index, the `price > 500` half uses the price index — then `UNION`
combines the results (deduplicating any product that satisfies both,
same behavior as [Lesson 12](12-advanced-queries.md)'s set operations).

### Pattern 9: SELECT *

```sql
-- BAD: fetches all columns including large TEXT, JSONB fields
SELECT * FROM products WHERE price > 100;

-- GOOD: fetch only what you need
SELECT id, name, price FROM products WHERE price > 100;
```
**Why this matters:** `SELECT *` fetches every column, including any
large `TEXT`/`JSONB` fields you don't actually need for this query —
wasted network transfer and memory for data you'll throw away. It also
blocks a specific optimization:

```
-- Significant for:
-- - Wide tables with many columns
-- - Tables with large TEXT or JSONB columns (large network transfer)
-- - Can enable "Index Only Scan" if covering index exists
```
**The "Index Only Scan" point specifically** ties back to
[Lesson 09](09-indexes-and-performance.md)'s covering index (`INCLUDE`)
feature — if every column your query needs already lives inside the
index, Postgres can skip touching the actual table entirely. `SELECT *`
guarantees you need columns outside the index, so this optimization
becomes impossible; naming only the columns you need keeps the door open.

### Pattern 10: Implicit Cartesian Product

```sql
-- SLOW: missing JOIN condition
SELECT * FROM orders, customers;
-- 10 orders × 10 customers = 100 rows (Cartesian product)
```
**Why this is slow (and wrong):** the comma between two tables with no
`ON` condition is exactly [Lesson 06](06-joins-complete.md)'s implicit
`CROSS JOIN` syntax — every row from `orders` paired with every row from
`customers`. This is almost always an accidental missing `JOIN`
condition, not a deliberate choice, and on real-sized tables (millions
of rows each) the row explosion (`rows_A × rows_B`) can exhaust memory
outright, not just run slowly.

```sql
-- FIX: always specify your JOIN conditions
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;
```
The `ON` condition tells Postgres exactly which rows actually relate to
each other, collapsing the result back down to one row per real match
instead of every possible pairing.

---

## Rewriting Slow Queries

**Why you need this:** the 10 patterns above were mostly fixed by adding
an index or updating stats — the query itself stayed the same. These
three are different: the query's *shape* itself is the problem, and no
index can fix it. You have to rewrite the SQL.

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
```
**Why this is slow:** a **correlated subquery** references the outer
row (`e.department`) inside the inner query, which means the inner
`SELECT AVG(salary) ... WHERE department = e.department` has to
re-run **once per outer row**. 100 employees means running that average
calculation 100 separate times, even though most employees share the
same department and could reuse the same average.

```sql
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
**Why this is fast:** the CTE ([Lesson 07](07-subqueries-and-ctes.md))
computes every department's average **exactly once**, up front. The
main query then just joins each employee to their department's
already-computed average — one calculation total, not one per row.

Sample output:
| name | salary |
|---|---|
| Alice | 95000 |
| Dan | 88000 |

### Slow NOT IN → Fast NOT EXISTS

```sql
-- SLOW (with NULL issue):
SELECT * FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- Also slow because of the NULL problem — may be wrong!
```
**Why this is slow AND potentially wrong:** if even *one* row in
`orders.customer_id` is `NULL`, `NOT IN` returns **no rows at all** —
not an error, just silently wrong, because comparing anything to `NULL`
with `NOT IN`'s logic evaluates to unknown, not true. This is a real,
well-known SQL trap, not a Postgres quirk specifically.

```sql
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
**Why both of these are safe:** neither one does a blanket "is this
value in that list" comparison — `NOT EXISTS` asks "does at least one
matching row exist," and the `LEFT JOIN` version checks "did this row
fail to find a match" ([Lesson 06](06-joins-complete.md)'s "customers
who never ordered" pattern). Neither logic breaks in the presence of
`NULL` values, and both typically run faster too, since Postgres can
use an index-backed lookup instead of building the full `NOT IN` list.

Sample output — customers with zero orders:
| id | name | email |
|---|---|---|
| 3 | Carol | carol@example.com |

### Slow aggregation → Pre-aggregate

```sql
-- SLOW: aggregate on the fly in every query
SELECT
    c.name,
    (SELECT COUNT(*) FROM orders WHERE customer_id = c.id) AS order_count,
    (SELECT SUM(total) FROM orders WHERE customer_id = c.id AND status='delivered') AS revenue
FROM customers c;
-- Runs 2 subqueries per customer row
```
**Why this is slow:** this has the same "runs once per outer row"
problem as the correlated subquery pattern above — except here it's
**two** separate subqueries per customer, each independently scanning
`orders`.

```sql
-- FAST: single aggregation JOIN
SELECT
    c.name,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total) FILTER (WHERE o.status='delivered'), 0) AS revenue
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;
```
**Why this is fast:** one `LEFT JOIN` brings every customer's orders
together in a single pass, and `FILTER` ([Lesson 05](05-aggregations.md))
computes the conditional sum without needing a second subquery — the
entire result comes from reading `orders` once, not twice per customer.

Sample output:
| name | order_count | revenue |
|---|---|---|
| Alice | 3 | 1329.98 |
| Carol | 0 | 0 |

---

## PostgreSQL Configuration for Performance

**Why you need this:** everything so far optimized individual queries.
These settings instead change how *all* queries run, by changing how
much memory Postgres is allowed to use — this is the practical, dial-in
version of the `work_mem`/`Batches`/disk-spill concepts already
introduced in [Lesson 09](09-indexes-and-performance.md)'s EXPLAIN
section.

```sql
-- View current settings
SHOW shared_buffers;
SHOW work_mem;
SHOW effective_cache_size;
```
**What each one actually controls:**
- **`shared_buffers`** — Postgres's own cache of table/index pages in
  memory. More of this means more data can be read from RAM instead of
  disk — directly related to the `Buffers: shared hit=X read=Y` line
  from EXPLAIN output (more `shared_buffers` → more `hit`, less `read`).
- **`effective_cache_size`** — not an actual memory allocation at all;
  just a *hint* telling the planner "assume roughly this much data can
  be served from OS-level disk cache," which affects which plans the
  planner considers cheap versus expensive.
- **`work_mem`** — the memory limit for a *single* sort or hash
  operation. This is the exact setting that determines whether a `Sort`
  or `Hash` node from EXPLAIN stays in memory or spills to disk
  (`Sort Method: external merge Disk:` from Lesson 09) — too small a
  `work_mem`, and otherwise-fast queries start hitting disk for sorting.

```sql
-- Key settings to tune (edit postgresql.conf):
-- shared_buffers = 25% of RAM         -- PostgreSQL's main cache
-- effective_cache_size = 75% of RAM   -- hint to planner about OS cache
-- work_mem = RAM / max_connections / 2 -- per-sort / per-hash operation
-- maintenance_work_mem = RAM * 0.1    -- for VACUUM, CREATE INDEX, etc.
-- max_connections = 100               -- reduce to allow larger work_mem
```
**Why `work_mem`'s formula divides by `max_connections`:** `work_mem` is
a *per-operation* limit, and multiple connections can each be sorting or
hashing at the same time. If you set it too generously without
accounting for concurrent connections, many simultaneous sorts could
together consume far more RAM than the server actually has — this is
why lowering `max_connections` (fewer simultaneous queries) is a valid
way to *safely* afford a larger `work_mem` per query.

```sql
-- Set session-level (for testing):
SET work_mem = '256MB';           -- more memory for sorts/hashes
SET enable_seqscan = OFF;         -- force index scan (for testing)
SET enable_hashjoin = OFF;        -- force merge or nested loop join
SET enable_mergejoin = OFF;       -- force hash or nested loop join

-- Reset:
RESET work_mem;
RESET enable_seqscan;
```
**Why set these at the session level instead of editing the config
file:** changing `postgresql.conf` affects *every* connection,
permanently, until you edit it back. `SET` (without `GLOBAL`) only
affects your current session — useful for testing "would a bigger
`work_mem` fix this specific slow query" without touching production
defaults for everyone else.

---

## Query Hints (PostgreSQL Style)

**Why you need this:** other engines (Oracle, MySQL) let you write
inline comments that *force* the planner's hand (`/*+ USE_INDEX(...) */`).
Postgres deliberately has no such built-in mechanism — this section
explains the actual philosophy, and the closest tools Postgres gives
you instead.

PostgreSQL doesn't have HINT syntax like Oracle/MySQL, but you can guide the planner:

```sql
-- Force index use: disable seq scan (test only)
SET enable_seqscan = OFF;
SELECT * FROM orders WHERE customer_id = 1;
SET enable_seqscan = ON;
```
**Why this is a testing tool, not a real fix:** turning `enable_seqscan`
off doesn't create a *better* plan — it just removes one option,
forcing Postgres to use whatever's left even if it's worse for this
particular query. Use this to *diagnose* ("is the planner's cost
estimate for the index scan actually wrong, or is Seq Scan genuinely
cheaper here?"), never as a permanent setting — leaving it off
everywhere would make every query without a usable index catastrophically
slow.

```sql
-- Use pg_hint_plan extension (if installed)
/*+ IndexScan(orders idx_orders_customer_id) */
SELECT * FROM orders WHERE customer_id = 1;
```
This is a third-party extension (not built into core Postgres) that
adds Oracle-style inline hints for teams that specifically want that
capability — worth knowing it exists, not something you'll need by default.

```sql
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
