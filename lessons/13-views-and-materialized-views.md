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

**Why you need this:** by [Lesson 12](12-advanced-queries.md) your
queries have gotten long — multiple joins, `GROUP BY`, computed columns.
Retyping that same 10-line query every time you need it (and keeping every
copy in sync if the logic ever changes) doesn't scale. A view is
PostgreSQL's answer: give a query a permanent name, and query that name
like a table from then on.

**The analogy that makes this click:** think of a view as a **saved search**
folder in an email client. The folder doesn't store its own copies of
emails — it's just a saved filter that re-runs live every time you open
it. A view works the same way: it's a saved `SELECT`, not a saved copy
of the *results* of that `SELECT`.

```
view = named SELECT statement
     = virtual table
     = no storage (unless materialized — covered later in this lesson)
```

**The one fact that explains almost everything about views:** every
time you query a view, Postgres substitutes in the view's original
`SELECT` and runs it *fresh*, against the live tables, right then. A
view is never "out of date," because it never stored anything to become
outdated — read that back after the Materialized View section below,
where this stops being true on purpose.

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
```

**What just happened:** Postgres did **not** run this query yet — it
just saved the query text under the name `customer_summary`. Nothing
was computed, nothing was stored. The computation only happens the
moment someone actually selects from the view:

```sql
-- Query the view like a table
SELECT * FROM customer_summary ORDER BY lifetime_value DESC;
```

| id | name | email | country | total_orders | lifetime_value | last_order_date |
|---|---|---|---|---|---|---|
| 1 | Alice Johnson | alice@example.com | US | 3 | 1329.98 | 2024-03-10 |
| 2 | Bob Smith | bob@example.com | UK | 1 | 129.99 | 2024-02-01 |
| 3 | Carol White | carol@example.com | US | 0 | 0.00 | NULL |

```sql
SELECT name, total_orders FROM customer_summary WHERE country = 'US';
SELECT * FROM customer_summary WHERE total_orders = 0;  -- customers with no orders
```

**The genuinely useful part:** you can `WHERE`/`ORDER BY`/filter a view
exactly like a real table, even though `customer_summary` itself already
contains a `GROUP BY` and a `JOIN`. The view hides that complexity —
from the outside, it just looks like a table called `customer_summary`
with those columns.

---

## Why Use Views?

**Why you need this:** "save a query under a name" is the mechanism —
this section is the actual payoff, four distinct real reasons people
reach for views in practice.

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

**The idea:** instead of granting a role access to an entire table, you
grant it access to a *view* that already filters down to only the rows
that role should see — the role never even gets the option to query
beyond that filter.

```sql
-- Only show your own data
CREATE VIEW my_orders AS
SELECT * FROM orders WHERE customer_id = current_user_id();
-- Grant access to this view, not to the orders table directly
```

**Why this actually enforces security, not just convenience:** if a
role only has `GRANT SELECT ON my_orders` (see
[Lesson 17](17-roles-users-and-access-management.md)) and **no** grant
on the real `orders` table, that role has *no way* to query `orders`
directly, no matter what they write — every query they run must go
through the view's baked-in `WHERE customer_id = current_user_id()`
filter. This is meaningfully different from just remembering to add
`WHERE customer_id = ...` in application code, which a bug or a raw SQL
console could bypass.

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

**Same underlying trick as #2, applied to columns instead of rows:**
`public_customer_info` simply never selects `password_hash` or `ssn` in
its definition — so a role granted only on this view has **no way** to
retrieve those columns, even by asking for `SELECT *`, because those
columns were never part of the view's own `SELECT` list to begin with.

| id | name | email | city | country | created_at |
|---|---|---|---|---|---|
| 1 | Alice Johnson | alice@example.com | Boston | US | 2023-05-01 |
| 2 | Bob Smith | bob@example.com | London | UK | 2023-06-15 |

Notice `password_hash` and `ssn` simply don't exist as columns here at
all — not hidden, structurally absent from this view's output.

### 4. Abstraction Layer

```sql
-- If you rename a table, create a view with the old name
-- Old code still works without changes
CREATE VIEW orders_legacy AS SELECT * FROM orders;
```

**The scenario this solves:** say you rename `orders` to
`customer_orders` as part of a bigger refactor, but a dozen old reports
and scripts still say `SELECT * FROM orders`. Instead of hunting down
and rewriting every one immediately, you create a view named `orders`
(the old name) that just points at the new table — every old query
keeps working, unmodified, while you migrate callers to the new name at
your own pace.

---

## Modifying and Dropping Views

**Why you need this:** views are rarely "create once, never touch" —
requirements change, and you need to know how to safely update or
remove one without breaking whatever already depends on it.

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
```

**Why prefer `CREATE OR REPLACE` over `DROP` + `CREATE`:** if you
`DROP` a view, every `GRANT` you'd set up on it ([Lesson 17](17-roles-users-and-access-management.md))
is gone too — you'd have to re-grant permissions from scratch after
recreating it. `CREATE OR REPLACE` swaps the view's underlying query
while leaving its existing permissions untouched.

**Why the "can only add columns at the end" restriction exists:**
anything already querying this view by column *position* or expecting
the existing columns in their existing order would silently break if
you removed or reordered columns underneath it. Postgres blocks that
specific kind of change under `REPLACE` — appending new columns at the
end is safe (nothing existing shifts), removing/reordering isn't, so
you'd need to `DROP` and recreate for that instead.

```sql
-- Drop a view
DROP VIEW customer_summary;
DROP VIEW IF EXISTS customer_summary;
DROP VIEW customer_summary CASCADE;  -- also drops views/rules that depend on it
```

**Why `CASCADE` matters here specifically:** views can be built *on top
of* other views (a view's `SELECT` can itself query another view).
Plain `DROP VIEW` fails with an error if anything else depends on it —
`CASCADE` says "yes, take those dependents down too," the same
reasoning as `ON DELETE CASCADE` from [Lesson 03](03-data-types-and-schema.md),
just for schema objects instead of rows.

```sql
-- Rename a view
ALTER VIEW customer_summary RENAME TO customer_stats;

-- Describe a view
\d+ customer_summary
-- or
SELECT definition FROM pg_views WHERE viewname = 'customer_summary';
```
`\d+ view_name` works on a view the same way it works on a table
([Lesson 01](01-how-databases-work.md)) — showing its columns; the
second form additionally shows the actual `SELECT` text the view was
built from, useful when you've forgotten exactly what a view does.

---

## Updatable Views

**Why you need this:** so far every view has only been *read from*. This
section covers writing *through* a view — and the specific limits on
when Postgres will let you do that.

**Why writing through a view isn't always possible — think about it
concretely first:** `customer_summary` from earlier includes
`COUNT(o.id) AS total_orders` — a computed aggregate. If you tried to
`UPDATE customer_summary SET total_orders = 5`, what would that even
*mean* to write back to the real `customers`/`orders` tables? There's no
sensible translation — `total_orders` isn't a real column anywhere, it's
a calculation. This is exactly why Postgres can only auto-update *simple*
views:

PostgreSQL can automatically make a view updatable if it meets conditions:
- FROM has only ONE base table
- No GROUP BY, HAVING, DISTINCT, LIMIT, UNION, aggregates, window functions

**In plain terms: the view must be a "thin wrapper" over one real
table** — just a filtered/renamed subset of its columns, nothing
computed or combined — so every write has one obvious, unambiguous real
row to land on.

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
```

**What's actually happening under the hood:** Postgres translates each
of these into the equivalent statement against the real `customers`
table — `UPDATE active_customers SET city = 'Boston' WHERE id = 1`
becomes, for all practical purposes, `UPDATE customers SET city =
'Boston' WHERE id = 1 AND deleted_at IS NULL`. The view is a pass-through,
not a separate copy of the data.

### `WITH CHECK OPTION` — Closing a Loophole

**The loophole, made concrete:** `active_customers` only shows rows
where `deleted_at IS NULL`. What stops someone from running `UPDATE
active_customers SET deleted_at = NOW() WHERE id = 1` — an update that
*succeeds*, but immediately makes that row vanish from the view's own
filter, right after the update that just went through it?

```sql
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

**What `WITH CHECK OPTION` actually enforces:** after any write through
this view, Postgres re-checks whether the *resulting* row would still
satisfy the view's own `WHERE` clause. If not, the write is rejected
outright — the row is never allowed to "write itself out of visibility"
through the same view that let the write happen.

---

## INSTEAD OF Triggers on Views

**Why you need this:** `order_with_customer` below joins two tables —
so by the updatable-views rule above, it fails the "only one base table"
condition and cannot be auto-updated. `INSTEAD OF` triggers are how you
manually teach Postgres what a write through a complex view should
actually *do*, since Postgres can no longer infer it automatically.

```sql
-- Complex view (not auto-updatable — joins TWO tables)
CREATE VIEW order_with_customer AS
SELECT o.id, o.total, o.status, c.name AS customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

**The trick: intercept the write instead of letting Postgres attempt
it.** `INSTEAD OF` means exactly what it says — *instead of* trying (and
failing) to actually run the `UPDATE` against this view, run this
trigger function's logic in its place.

```sql
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

**Walking through what this means in practice:** `NEW` here refers to
the row *as the user tried to update it* through the view (the new
`total`/`status` values they attempted to set). The trigger function
takes those values and manually applies them to the *real* `orders`
table instead — you, the schema author, are writing the translation
from "a write attempt on the view" to "the actual write that should
happen on real tables," since Postgres has no way to guess it for a
two-table join on its own.

**After this trigger exists**, code like
`UPDATE order_with_customer SET total = 150 WHERE id = 3;` works exactly
as if the view were a real, writable table — even though moments ago it
would have failed outright.

---

## Materialized Views — Cached Query Results

**Why you need this:** every view so far re-runs its full query on every
single access — fine for simple lookups, genuinely slow if the
underlying query does heavy aggregation across millions of rows
([Lesson 09](09-indexes-and-performance.md)'s cost concepts apply here
directly). A materialized view breaks the "always re-run it live" rule
from the start of this lesson **on purpose** — trading perfect freshness
for real speed.

**The one sentence that separates the two:** a regular view stores a
*query*; a materialized view stores a *result*.

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
```

**What actually happens on `CREATE`:** unlike a regular view, this
*does* run the query immediately — the full join and aggregation across
`products`/`order_items` executes once, right now, and its output rows
get physically written to disk, exactly like a real table's data would.

```sql
-- Query it — very fast, uses stored data
SELECT * FROM product_sales_summary ORDER BY total_revenue DESC;
SELECT category, SUM(total_revenue) FROM product_sales_summary GROUP BY category;
```

| id | name | category | price | total_units_sold | total_revenue | order_count |
|---|---|---|---|---|---|---|
| 3 | Laptop Pro | Electronics | 1299.99 | 42 | 54599.58 | 30 |
| 7 | Office Chair | Furniture | 249.99 | 15 | 3749.85 | 12 |

**Why this is genuinely faster, not just "cached":** querying
`product_sales_summary` never touches `products` or `order_items` at
all — it reads pre-computed rows directly, the same speed as querying
any ordinary table. The expensive `JOIN` + `GROUP BY` work already
happened once, back when you ran `CREATE MATERIALIZED VIEW` (or last
refreshed it — next section).

**The tradeoff this section's title hints at — "cached" always implies
a catch:** the data you just queried is a **snapshot** from whenever it
was last computed. If new orders come in afterward, `product_sales_summary`
does **not** reflect them until someone explicitly tells it to
recompute — which is exactly what "refreshing" means, covered next.

### Refreshing Materialized Views

```sql
-- Manual refresh (blocks reads until done)
REFRESH MATERIALIZED VIEW product_sales_summary;
```

**What this does, concretely:** re-runs the *entire* original query
from scratch, and replaces every stored row with the new results. This
is why the comment says "blocks reads until done" — while Postgres is
in the middle of throwing away the old snapshot and computing the new
one, the materialized view is briefly unusable, similar to the table
locking discussed for plain `CREATE INDEX` in [Lesson 09](09-indexes-and-performance.md).

```sql
-- Concurrent refresh (allows reads during refresh — requires UNIQUE index)
CREATE UNIQUE INDEX ON product_sales_summary(id);
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;
-- Concurrent refresh: old data available during refresh, no downtime
```

**Why `CONCURRENTLY` specifically requires a `UNIQUE` index first:** to
refresh without blocking readers, Postgres needs a way to compare the
*old* stored rows against the *newly computed* rows one-by-one, so it
can update only what actually changed instead of wiping everything and
starting over. A unique index gives it exactly that — a reliable way to
match "this old row" to "its corresponding new row." Without one,
Postgres has no such matching key and can't do the comparison safely,
so it refuses to run a concurrent refresh at all.

```sql
-- Full vs partial refresh
-- Materialized views always do a full refresh — no incremental option
-- (Timescale or custom solutions handle incremental refresh)
```
**Even `CONCURRENTLY` still recomputes the *entire* query** — the
concurrency only affects whether readers get blocked *during* that
recomputation, not how much work gets redone. Plain PostgreSQL has no
built-in way to refresh just "the rows that changed" — that's a
specialized feature some extensions add on top.

### Automating Refresh

**Why you need this:** a materialized view that's never refreshed just
gets progressively staler forever — something needs to trigger the
refresh on some cadence. Two genuinely different strategies for that:

```sql
-- Option 1: pg_cron (extension)
SELECT cron.schedule('refresh-sales', '0 * * * *',  -- every hour
    'REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary');
```
**Scheduled refresh — the usual choice.** `pg_cron` runs the refresh on
a fixed clock schedule (here, hourly, via the same cron syntax from
`CronCreate`-style tools), regardless of how much or how little data
changed in between. Simple, predictable, and the right default for most
dashboards where "a few minutes/hours old" is perfectly acceptable.

#### Installing `pg_cron` — Why It's Not Just `CREATE EXTENSION`

**Why this is different from every other extension in this course:**
`pg_trgm`, `uuid-ossp`, and the others ship *inside* Postgres already —
`CREATE EXTENSION` just switches on a feature already sitting on disk.
`pg_cron` is genuinely third-party code that must be **compiled against
your exact Postgres version**, placed where Postgres can find it, and
loaded via a config setting that only takes effect on a full server
restart — you can't just run one command cold.

**Steps for a Homebrew Postgres setup (from [Lesson 00](00-installing-postgresql.md)):**

**1. Clone and build it against your specific Postgres install:**
```bash
git clone https://github.com/citusdata/pg_cron.git
cd pg_cron
make PG_CONFIG=$(brew --prefix postgresql@16)/bin/pg_config
make install PG_CONFIG=$(brew --prefix postgresql@16)/bin/pg_config
```
`PG_CONFIG` must point at *your* Postgres version's `pg_config` tool —
this is what makes the build match your server instead of some
unrelated Postgres install.

**2. Tell Postgres to load it at startup.** Find the config file:
```sql
SHOW config_file;
```
Edit that file (`postgresql.conf`) and add:
```
shared_preload_libraries = 'pg_cron'
```
**Why this needs a restart, not just a session command:** `pg_cron`
runs a background worker process inside Postgres itself — that only
works if it's loaded when Postgres *starts*, not turned on mid-session.

**3. Restart Postgres to apply it:**
```bash
brew services restart postgresql@16
```

**4. Now enable the extension inside your database:**
```sql
CREATE EXTENSION pg_cron;
```

**5. Verify it's working:**
```sql
SELECT * FROM cron.job;   -- lists scheduled jobs, empty until you schedule one
```

**If this feels heavy for a learning database** — that reaction is
fair. This is exactly why Option 2 below (a plain OS-level `cron` job
calling `psql -c "REFRESH MATERIALIZED VIEW ..."`) is worth knowing: it
achieves the identical result without touching Postgres's build or
config at all. `pg_cron`'s extra setup cost mainly pays off on a managed
production server, where you don't have OS-level cron access in the
first place.

```sql
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
**Trigger-based refresh — refresh immediately whenever the underlying
data changes**, instead of waiting for the next scheduled tick. The
warning in the comment is the real catch: `product_sales_summary`'s
underlying query does a full `JOIN` + `GROUP BY` scan — re-running that
after *every single* `order_items` write, on a busy production table,
means the refresh cost could dwarf the actual write itself. This
pattern only makes sense when writes are infrequent, or you batch
several writes before refreshing once — otherwise, prefer Option 1.

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

**Why you need this:** two realistic, complete examples, applying every
idea from this lesson together — the aggregation itself, the unique
index `CONCURRENTLY` requires, and a sensible refresh cadence.

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

**Why `day` is the column chosen for the unique index:** it's the one
column that's actually unique per row in this materialized view's output
(one row per calendar day) — matching the requirement from the
"Refreshing" section above that `CONCURRENTLY` needs a reliable way to
match old rows to new ones.

Sample output:
| day | orders | unique_customers | revenue | avg_order_value | largest_order |
|---|---|---|---|---|---|
| 2024-03-08 | 6 | 5 | 812.45 | 135.41 | 299.99 |
| 2024-03-09 | 4 | 4 | 540.00 | 135.00 | 250.00 |

Refreshing nightly at 2am (via `pg_cron`, as shown) means this dashboard
is always at most one day stale — an entirely reasonable tradeoff for a
report a human checks once a day, per the "Best for" row in the decision
table above.

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

**Notice the `CASE` expression building a `segment` column** — this is
the exact `CASE WHEN ... THEN ... ELSE ... END` pattern from
[Lesson 02](02-sql-fundamentals.md), applied to bucket customers by
spend. Because this runs as part of a materialized view rather than a
regular view, that segmentation logic gets computed *once* per refresh
rather than recalculated on every single dashboard load.

Sample output:
| customer_id | name | ... | lifetime_value | segment |
|---|---|---|---|---|
| 1 | Alice Johnson | ... | 1329.98 | VIP |
| 2 | Bob Smith | ... | 129.99 | New |
| 3 | Carol White | ... | 0.00 | Never Ordered |

---

## Schema Views (Inspecting the Database)

**Why you need this:** once a real project accumulates a dozen views
and materialized views, you need a way to answer "what views exist,"
"what does this one actually do," and "is this materialized view stale"
— without having to remember every `CREATE VIEW` you ever ran.

```sql
-- List all views
SELECT viewname, definition FROM pg_views WHERE schemaname = 'public';
\dv   -- in psql
```
`pg_views` is one of Postgres's built-in system catalogs — a real table
Postgres maintains automatically, listing every view that exists.
`\dv` is just the `psql` shortcut for the same information (parallel to
`\dt` for tables, from [Lesson 01](01-how-databases-work.md)).

```sql
-- List all materialized views
SELECT matviewname, definition FROM pg_matviews WHERE schemaname = 'public';
\dm   -- in psql
```
Same idea, `pg_matviews` instead — a separate catalog specifically for
materialized views, since they're a genuinely different kind of object
(they physically own stored data, unlike regular views).

```sql
-- Check if a mat view needs refresh
SELECT
    matviewname,
    last_refresh,
    NOW() - last_refresh AS age
FROM pg_stat_user_tables
WHERE relname IN (SELECT matviewname FROM pg_matviews);
```
Ties directly back to the "stale until refreshed" row in the decision
table earlier — this is how you actually *check* that staleness instead
of guessing: `NOW() - last_refresh` gives you exactly how old the
snapshot is, using the same timestamp-subtraction pattern from
[Lesson 02](02-sql-fundamentals.md)'s `INTERVAL` arithmetic.

```sql
-- Find views that depend on a table
SELECT viewname
FROM pg_views
WHERE definition ILIKE '%orders%';
```
**A practical use case for this:** before altering or dropping `orders`,
you'd want to know every view whose definition references it — this
text-search over `pg_views.definition` (using `ILIKE`, [Lesson 04](04-filtering-deep-dive.md))
is a quick way to find them all, so nothing breaks unexpectedly.

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
