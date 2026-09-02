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

Imagine `users` has 10 million rows, with no index, and you run:
```sql
SELECT * FROM users WHERE email = 'alice@example.com';
```

Postgres has no way to know where Alice's row lives, so it does the only
thing it can: **check every single row, one by one**, from the first to
the last, testing each one's `email` against `'alice@example.com'`. This
is a **sequential scan** — literally reading the whole table cover to
cover. With 10 million rows, that's genuinely slow — seconds, not
milliseconds.

**An index changes this from "check every row" to "look it up directly."**
Think of the index on a book: instead of reading every page to find
"Postgres," you check the index at the back, see "page 214," and flip
straight there. A database index works the same way — it's a separate,
much smaller structure that tells Postgres exactly where a value lives,
so it never has to scan rows it doesn't need.

With a B-tree index on `email`:
- Postgres searches the index (fast — it's small and structured for searching)
- The index tells it exactly where Alice's row is stored
- Postgres fetches just that one row — microseconds, not seconds

**The core tradeoff to hold onto for this entire lesson:** an index is
extra bookkeeping the database maintains so *reads* get faster. That
bookkeeping isn't free — it costs disk space, and it costs extra work on
every *write*. Every section below is really just different angles on
that one tradeoff.

---

## How B-Tree Indexes Work

**Why you need this:** the previous section told you indexes make
lookups fast, but not *how*. This section is the mechanism — understand
this once, and every later section (why column order matters, why the
planner sometimes skips your index, why writes get slower) will make
sense as a consequence of this structure, instead of being separate
rules to memorize.

B-tree ("Balanced tree") is the default index type Postgres uses. Here's
the shape of one, built on a numeric column with values roughly ranging
10 to 95:

```
                    [50 | 75]                  ← Root node
                   /    |    \
          [20|30]     [60|70]     [80|90]      ← Internal nodes
         /   |   \    / | \      /   |   \
      [10][25][35] [55][65][72] [77][85][95]   ← Leaf nodes (contain actual values + row pointers)
```

Read this the same way you'd read a decision tree, or the "guess a
number" game — at each level, you're told which direction narrows the
search:

- **Root node** — the very first check. It holds a couple of boundary
  values (`50`, `75`) that split everything into three zones: less than
  50, between 50 and 75, or greater than 75.
- **Internal nodes** — the next level of narrowing, within whichever
  zone you landed in.
- **Leaf nodes** — the actual destination. Each leaf holds real values
  from the table, and critically, **a pointer to exactly where that row
  physically lives on disk** (the "heap"). This is the payoff — the leaf
  doesn't just confirm the value exists, it tells Postgres where to go
  fetch the full row.

**Walk through an actual lookup** — say the query is `WHERE price = 65`:

1. Start at the **root**: is 65 less than 50, between 50–75, or greater
   than 75? It's between 50 and 75 → go to the middle branch, `[60|70]`.
2. At `[60|70]`: is 65 less than 60, between 60–70, or greater than 70?
   Between 60 and 70 → go into that leaf group, `[55][65][72]`.
3. At the **leaf level**: scan this small group directly — `65` is
   right there, with a pointer to its row on disk.
4. Postgres follows that pointer, fetches the row, done.

That's **3 steps** to find one row out of potentially millions — instead
of checking every row one at a time. This is what "O(log n)" means in
practice: each additional million rows in the table only adds one or two
more steps to the tree, not a million more comparisons.

**Why "balanced" matters:** every leaf sits at the exact same depth, so
*every* lookup takes roughly the same small number of steps, no matter
which value you're searching for. Postgres automatically rebalances the
tree as rows are added — you never have to think about this yourself.

**Why "ordered" matters:** because leaves are stored in sorted order and
linked to their neighbors, a *range* query (`WHERE price > 60`) works
almost the same way — navigate to where 60 would be, then just walk
forward through the linked leaves collecting everything larger, instead
of restarting the search for every value. This is also exactly why a
B-tree index can satisfy `ORDER BY` for free — the data's already sorted.

---

## Index Types

**Why you need this:** just so the names don't surprise you later —
B-tree is what you'll actually use almost every time; this table exists
so you can recognize the others by name when you see them elsewhere,
not because you need to master all six right now.

B-tree is the one you'll use 95% of the time. The others exist for
specific data shapes where B-tree doesn't fit naturally:

| Index Type | Use Case | PostgreSQL Command |
|---|---|---|
| **B-tree** | Equality, ranges, ORDER BY, LIKE 'prefix%' | Default |
| **Hash** | Equality only (=), faster than B-tree for that | `USING HASH` |
| **GIN** | Arrays, JSONB, full-text search | `USING GIN` |
| **GiST** | Geometry, ranges, text search (tsvector) | `USING GIST` |
| **BRIN** | Very large tables with naturally ordered data | `USING BRIN` |
| **SP-GiST** | Non-balanced structures (quad-trees, k-d trees) | `USING SPGIST` |

**Rule of thumb:** start with a plain B-tree (the default — you don't
even need to specify it). Only reach for GIN when indexing JSONB/arrays
(see [Lesson 16](16-postgresql-power-features.md)), and don't worry
about the rest until you have a specific, unusual need.

---

## Creating Indexes

**Why you need this:** the previous sections explained the *concept* of
an index (B-tree structure, why it's fast). This section is the actual
*syntax* — how you tell Postgres "build one of those" for a real column,
plus a few specialized variants for specific situations you'll recognize
later (only querying active rows, querying a lowercased column, etc).

### What Is `index_name`?

```sql
CREATE INDEX index_name ON table_name (column1, column2, ...);
```

`index_name` is **not a keyword Postgres requires you to match** — it's
just a label *you* invent, exactly like naming a variable in code.
Postgres doesn't care what you call it; it exists purely so *you* (and
future-you, reading `EXPLAIN` output or a list of indexes months from
now) can tell what this index is for, and so you can `DROP` it by name
later.

```sql
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
--            ^^^^^^^^^^^^^^^^^^^^^   ^^^^^^ ^^^^^^^^^^^
--            the name YOU picked     table   column being indexed
```

You could technically name it `banana` and it would work identically —
```sql
CREATE INDEX banana ON orders(customer_id);  -- works, just an unhelpful name
```
— it's only convention pushing you toward `idx_<table>_<column>`,
because that name tells you what the index is *for* just by reading it,
which `banana` never will. If you omit a name entirely, Postgres
auto-generates one like `orders_customer_id_idx`.

```sql
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

**What each variant is actually for, in plain terms:**
- **Partial index** — you almost always query `WHERE status = 'pending'`,
  never the huge pile of `'delivered'` rows. Indexing only the small
  active subset keeps the index tiny and fast, instead of indexing
  millions of rows you'll never look up this way.
- **Expression index** — a plain index on `email` can't help
  `WHERE LOWER(email) = ...`, because the index stores the *raw* values,
  not the lowercased ones. Indexing `LOWER(email)` directly makes the
  index match what your query actually computes.
- **Covering index (`INCLUDE`)** — normally, after the index finds a
  match, Postgres still has to go fetch the full row from the table (the
  "heap") to get columns not in the index. If you stuff those extra
  columns *into* the index itself, that extra fetch is skipped entirely
  — see "Index Only Scan" below.
- **`CONCURRENTLY`** — a normal `CREATE INDEX` locks the table against
  writes while it builds. On a live production table, that's disruptive.
  `CONCURRENTLY` builds it more slowly but without blocking anything.

---

## Composite Indexes — Column Order Matters

**Why you need this:** every index so far covered one column. Real
queries often filter on *multiple* columns at once
(`WHERE customer_id = 1 AND status = 'delivered'`) — this section covers
indexing several columns together, and the one non-obvious rule
(column order) that determines whether such an index actually gets used.

### Forget SQL for a Second — the Phone Book

Imagine a paper phone book, sorted by **last name first, then first
name**:
```
Adams, John
Adams, Mary
Baker, Steve
Baker, Tom
Clark, Anna
```

- **"Find everyone named Baker"** — easy. The book is sorted by last
  name, so every Baker is grouped together — flip straight there.
- **"Find Baker, Tom" specifically"** — also easy, one more step: find
  "Baker," then within that small group, find "Tom."
- **"Find everyone whose first name is Tom"** — the phone book is
  useless here. Tom Baker, Tom Clark, Tom Adams could be scattered
  anywhere, because the book was never sorted by first name — you'd
  have to read the *entire* book page by page.

**A composite index is exactly this phone book.** Writing:
```sql
CREATE INDEX idx_orders_compound ON orders(customer_id, status, order_date);
```
tells Postgres: "sort the index by `customer_id` first, then within
each `customer_id`, sort by `status`, then within that, sort by
`order_date`" — the exact same structure as last-name-then-first-name.

| Query | Uses the index? | Why |
|---|---|---|
| `WHERE customer_id = 1` | ✅ Yes | Uses the leftmost column |
| `WHERE customer_id = 1 AND status = 'delivered'` | ✅ Yes | Leftmost, then the next column in order |
| `WHERE customer_id = 1 AND status = 'delivered' AND order_date > '2024-01-01'` | ✅ Yes | All three columns, left to right |
| `WHERE status = 'delivered'` | ❌ No | Skips `customer_id` — the "phone book" isn't sorted by `status` alone |
| `WHERE order_date > '2024-01-01'` | ❌ No | Skips both columns before it |
| `WHERE customer_id = 1 AND order_date > '2024-01-01'` | ⚠️ Partially | Uses `customer_id`, but since `status` is skipped, it can't also use `order_date` efficiently — works, but less optimally |

**The one rule to memorize:** you must use a composite index's columns
starting from the **first one**, in order. You can stop partway through
and still benefit (just `customer_id` alone is fine) — what you can't do
is skip the first one and jump to a later column, same as you can't
search a last-name-sorted phone book by first name alone.

**Why a range condition (`>`, `<`, `BETWEEN`) breaks the chain, but
equality (`=`) doesn't:** an equality condition narrows the index down to
one exact slice — the search can keep going deeper from there. A range
condition instead selects a *spread* of values — there's no single next
point to keep narrowing from, so any column listed after a range
condition can't be used efficiently.

**The practical rule this produces: order composite index columns as
equality columns first, range columns last.**
```sql
-- Good:  (customer_id, status, order_date)   → equality, equality, range
-- Bad:   (order_date, customer_id, status)   → range first blocks the rest
```

---

## EXPLAIN — Reading the Query Plan

Before creating indexes based on guesswork, you can ask Postgres exactly
*how* it intends to run a query — this is what `EXPLAIN` is for.

```sql
-- EXPLAIN shows the plan WITHOUT running the query
EXPLAIN SELECT * FROM orders WHERE customer_id = 1;

-- EXPLAIN ANALYZE actually RUNS the query and shows real timing alongside the plan
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 1;

-- EXPLAIN with full output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT) 
SELECT * FROM orders WHERE customer_id = 1;

-- EXPLAIN with JSON format (for programmatic parsing)
EXPLAIN (FORMAT JSON, ANALYZE) SELECT * FROM orders WHERE customer_id = 1;
```

**The distinction that matters most:** `EXPLAIN` alone is a *prediction*
— Postgres's planner estimates what it thinks will happen, without
actually doing it. `EXPLAIN ANALYZE` actually executes the query for
real and reports what *actually* happened. Use plain `EXPLAIN` to check
a plan without side effects (careful with `EXPLAIN ANALYZE` on an
`UPDATE`/`DELETE` — it really runs it); use `EXPLAIN ANALYZE` when you
want to compare the estimate against reality.

### Reading EXPLAIN Output — Line by Line

```
                               QUERY PLAN
----------------------------------------------------------------------
 Index Scan using idx_orders_customer_id on orders  (cost=0.14..8.17 rows=2 width=48)
                                                    (actual time=0.045..0.048 rows=2 loops=1)
   Index Cond: (customer_id = 1)
 Planning Time: 0.132 ms
 Execution Time: 0.068 ms
```

Translate this line by line into plain English:

> "I'm going to use the index `idx_orders_customer_id` to scan the
> `orders` table. I *predict* this will take between 0.14 and 8.17
> arbitrary cost-units, and return about 2 rows averaging 48 bytes each.
> Having actually run it: it took 0.045ms to get the first row and
> 0.048ms total, it really did return 2 rows, and this step ran once.
> The condition I used to narrow things down was `customer_id = 1`.
> Total planning took 0.132ms, total execution took 0.068ms."

**Node type** — *what operation this step performs*:
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

**Cost** — *the planner's prediction, in arbitrary units, not milliseconds*:
```
cost=0.14..8.17
      ↑    ↑
      |    Total cost to return ALL rows this step produces
      Startup cost — the cost to produce the FIRST row
```
Think of "startup cost" as "how long before results even start coming
back" and "total cost" as "how long until every row from this step is
done." These numbers only mean something *relative to each other* —
useful for comparing two candidate plans, meaningless as an actual time.

**Rows** — *the planner's row-count prediction*:
```
rows=2
```
Compare this against the *actual* rows once you use `EXPLAIN ANALYZE`
(`actual ... rows=2`). If these two numbers are wildly different — say,
predicted 2 but actually 200,000 — the planner's statistics about your
data are stale, and it may be choosing a bad plan based on outdated
assumptions (fixed by running `ANALYZE`, covered later in this lesson).

**Width** — *estimated average size of each row this step returns, in bytes*:
```
width=48
```

**Actual time** — *only appears with EXPLAIN ANALYZE, real measured milliseconds*:
```
actual time=0.045..0.048
            ↑       ↑
            Time until the FIRST row was ready   Time until the LAST row was ready (ms)
```

**Loops** — *how many times this exact step ran*:
```
loops=1
```
Most steps run once. But inside a `Nested Loop` join, the *inner* side
often runs once *per row* of the outer side — so `loops=500` there would
mean "this step repeated 500 times," and its cost/time numbers are
*per execution*, not total — worth multiplying out mentally when
diagnosing a slow nested loop.

---

## Scan Types — What They Mean, and When Each Appears

**Why you need this:** the EXPLAIN section above taught you to read the
"node type" line (`Seq Scan`, `Index Scan`, etc.) but not what each name
actually *means*, or whether seeing it is good or bad news. This section
is the decoder key for that one line — once you can judge "is this scan
type expected here, or a problem?", EXPLAIN output stops being just
data and starts being a diagnosis.

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

**How these four relate, as one spectrum from "return almost nothing" to
"return almost everything":**

| Scan type | Roughly how much of the table it returns | Why this is the right choice at that volume |
|---|---|---|
| **Index Only Scan** | Any amount, but every needed column lives in the index | Never touches the actual table at all — pure index lookup |
| **Index Scan** | A small slice (< ~1%) | Index finds exact rows quickly, one heap fetch each |
| **Bitmap Heap Scan** | A moderate slice (~1–10%) | Too many rows for one-at-a-time heap fetches to be efficient — instead, mark all matching page locations first, then read those *pages* in physical order, minimizing disk jumping around |
| **Sequential Scan** | A large slice (> ~10%) or no index exists | At that volume, reading everything start to finish beats the overhead of consulting an index at all |

This is *why* the planner sometimes ignores a perfectly good index — see
the next section.

---

## Why the Planner Might Ignore Your Index

An index existing doesn't guarantee Postgres will use it — the planner
always picks whichever plan it estimates will be *fastest*, and
sometimes that's a sequential scan even with an index available.

```sql
-- 1. Query returns too many rows (low selectivity)
-- If >10% of rows match, a sequential scan is faster
EXPLAIN SELECT * FROM orders WHERE status = 'delivered';
-- If most orders are 'delivered', planner may choose Seq Scan
```
**Why:** if 80% of rows match, using the index means: look up each
matching entry in the index, then jump to the heap for each one
individually — scattered, random disk access. Just reading the table
straight through, in physical order, ends up faster once you're
fetching *most* of it anyway.

```sql
-- 2. Statistics are stale
-- Run ANALYZE to update statistics
ANALYZE orders;
-- Or let autovacuum handle it
```
**Why:** the planner's row-count predictions come from cached statistics
about your data's distribution, not a live count. If a huge data load
just happened and stats haven't refreshed, the planner may be working
off numbers that no longer reflect reality.

```sql
-- 3. Table is too small
-- For tiny tables, sequential scan is always faster than index lookup
-- Don't worry about missing indexes on tables with < 1000 rows
```
**Why:** navigating a B-tree (even 3 steps) has *some* overhead. For a
100-row table, just reading all 100 rows directly is faster than that
overhead — there's nothing to save.

```sql
-- 4. Data type mismatch
-- Index on INT, but query compares with TEXT
EXPLAIN SELECT * FROM orders WHERE customer_id = '1';   -- '1' is text!
-- May not use index. Always match types.
```
**Why:** the index stores integers. Comparing against the text `'1'`
may force a type conversion that the index can't be searched with
directly, depending on the comparison — matching types avoids the
ambiguity entirely.

```sql
-- 5. Function on column
-- Index on salary, but query uses function:
EXPLAIN SELECT * FROM employees WHERE UPPER(name) = 'ALICE';
-- Won't use idx_employees_name — create expression index instead:
CREATE INDEX idx_employees_upper_name ON employees(UPPER(name));
```
**Why:** the index on `name` stores the raw values (`'Alice'`), but the
query is searching for the *result of a function* (`UPPER(name)` =
`'ALICE'`) — those are different values as far as the index is
concerned. An expression index solves this by indexing the function's
*output* directly, so the two finally match.

---

## When to Create an Index

**Why you need this:** everything above explained the mechanics. This
section is the judgment call — given a real table, which columns are
actually worth indexing, and which ones would just add cost with no
real benefit (tying back to the read-vs-write tradeoff from the very
first section of this lesson).

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

**"Cardinality" is just "how many distinct values a column has."** An
`email` column has near-unique values for every row — high cardinality
— so an index there narrows a search down to almost one exact row every
time. A `boolean` column has only 2 possible values — low cardinality —
so an index there can, at best, only narrow the search down to "half the
table," which is barely better than not searching at all.

**A one-line mental checklist:** *"Is this column something I `WHERE`,
`JOIN`, or `ORDER BY` on often, and does it actually narrow the result
down to a small slice when I do?"* If yes to both — index it. If either
answer is no, it's probably not worth it.

### When indexes HURT:

```sql
-- 1. High-write tables
-- Every INSERT/UPDATE/DELETE must update all indexes
-- A table with 10 indexes takes ~10x longer to write to
```
**Why:** every index is a second (or third, or tenth) structure that has
to stay in sync with the table. Every write updates *all* of them, not
just the table itself — this is the "cost" side of the tradeoff from the
very first section of this lesson.

```sql
-- 2. Small tables
-- Sequential scan is faster for < ~1000 rows
```
Same reasoning as before — nothing to save when the whole table already
fits in a quick read.

```sql
-- 3. Low-cardinality columns
-- Boolean column: only 2 values — index barely helps
-- Status column with 3 values: ~33% selectivity — probably not useful
```
The index exists, costs disk space and write overhead, but the planner
will likely ignore it anyway (see the selectivity discussion above) — so
it's pure cost with no real benefit.

```sql
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
`idx_scan = 0` means Postgres has never once used this index since
statistics were last reset — it's pure dead weight, safe to consider
dropping.

---

## Index Maintenance

**Why you need this:** you now know how to create and judge indexes.
This section is just the everyday housekeeping commands — checking how
big an index is, rebuilding one, deleting one you no longer need. Not
new concepts, just the toolkit for managing indexes you've already
decided to create.

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

**Why an index would ever need "rebuilding" (`REINDEX`):** as rows are
updated and deleted over time, an index can accumulate wasted, unused
space inside its structure ("bloat") — similar to fragmentation.
`REINDEX` rebuilds it from scratch, clean and compact. This is
maintenance you'd reach for occasionally on a heavily-updated table, not
something you need to think about for a small learning database.

---

## Table Statistics and VACUUM

**Why you need this:** this connects back to the EXPLAIN section — the
planner's row-count *predictions* come from statistics that can go
stale, which is one of the reasons `EXPLAIN` estimates can drift from
reality. This section explains where those statistics come from and how
they're kept fresh, plus a second, unrelated cleanup job (`VACUUM`) that
happens to be covered alongside it.

Postgres doesn't physically delete a row the instant you `DELETE` or
`UPDATE` it — for reasons tied to how transactions work (see
[Lesson 10](10-transactions-and-acid.md)), the old row version sticks
around on disk marked as "dead," until something cleans it up.

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

**Two different jobs, easy to conflate:**
- **`VACUUM`** — reclaims space from those leftover "dead" row versions,
  so the table doesn't grow forever from updates/deletes alone.
- **`ANALYZE`** — refreshes the planner's *statistics* about your data
  (roughly: "how many rows," "how are values distributed") — this is
  what feeds the row-count predictions you saw in `EXPLAIN` output
  earlier in this lesson.

**`autovacuum`** does both of these automatically in the background on
a normal running database — you generally don't have to run these
manually, *except* right after a large bulk data load, where it's worth
forcing an `ANALYZE` immediately so the planner isn't working off
stale, pre-load statistics in the meantime.

---

## Practical EXPLAIN ANALYZE Workflow

Putting everything in this lesson together into an actual diagnostic
process:

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

Each of Step 2's warning signs maps directly back to a section above:
`Seq Scan` on a big table means "why the planner might ignore an index"
territory (or you're missing one entirely); estimated-vs-actual
mismatches mean stale statistics (`ANALYZE`); disk-based sorts/hashes
mean the operation needed more working memory than Postgres was
configured to use in RAM.

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

1. An index trades write cost and disk space for faster reads — every section of this lesson is a variation on that one tradeoff
2. B-tree indexes are O(log n) lookups vs O(n) sequential scans — a few navigation steps instead of checking every row
3. Composite index columns must be in left-to-right order — equality first, range last
4. EXPLAIN shows a *prediction*; EXPLAIN ANALYZE shows *reality* — big gaps between estimated and actual rows mean stale statistics
5. `Seq Scan` on large tables = missing index, or the query matches too large a fraction of the table for an index to help
6. Create indexes on foreign key columns — PostgreSQL doesn't do this automatically
7. Indexes slow down writes — don't over-index write-heavy tables or low-cardinality columns
8. Use `CREATE INDEX CONCURRENTLY` in production to avoid table locks
9. `ANALYZE` refreshes planner statistics; `VACUUM` reclaims space from dead rows — different jobs, both usually automatic via autovacuum

---

## Next Lesson
[Lesson 10 — Transactions & ACID](10-transactions-and-acid.md)
