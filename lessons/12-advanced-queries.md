# Lesson 12 — Advanced Queries

## Goal
Master the advanced SQL constructs that turn good SQL into great SQL:
set operations, pivoting, lateral joins, and deep dives into string,
date, and mathematical functions.

## Prerequisites
- [Lesson 11](11-database-design-normalization.md) — normalization

## After This Lesson You Will Be Able To
- Combine result sets with UNION, INTERSECT, EXCEPT
- Pivot rows into columns (and unpivot columns into rows)
- Use LATERAL for powerful correlated subqueries in FROM
- Apply all important string, date, and math functions
- Use GENERATE_SERIES for data generation
- Write DISTINCT ON queries

---

## Set Operations

**Why you need this:** every query so far combined tables *sideways*
with `JOIN` (matching columns across tables — [Lesson 06](06-joins-complete.md)).
Set operations instead stack two queries' results *vertically* — same
shape of output, rows combined top-to-bottom, like literally gluing two
result tables together. Think of `JOIN` as "widen the row," and `UNION`
as "add more rows."

### UNION — Combine Results

```sql
-- All customer emails and employee emails combined
SELECT email, 'customer'  AS source FROM customers
UNION
SELECT email, 'employee'  AS source FROM employees
ORDER BY email;
```

**What happens:** Postgres runs both `SELECT`s independently, then
stacks their results into one list, removing any exact-duplicate rows
(the same email + same source appearing from both sides) — this is why
`UNION` behaves like `DISTINCT` applied to the combined result.

Sample output:
| email | source |
|---|---|
| alice@example.com | customer |
| bob@example.com | employee |
| carol@example.com | customer |

```sql
-- UNION ALL (keep duplicates, more performant)
SELECT name FROM customers
UNION ALL
SELECT name FROM employees;
-- Alice and an employee named Alice both appear
```

**`UNION ALL` skips the duplicate-removal step** — every row from both
queries appears, even if two rows are identical. **Why prefer it when
you can:** de-duplicating means Postgres has to sort/compare every row
against every other row, which costs real time on large result sets. If
you already know the two sources can't overlap (or don't care if they
do), `UNION ALL` is strictly faster.

Sample output (both `Alice`s kept):
| name |
|---|
| Alice |
| Bob |
| Alice |
| Dan |

**The rule that must hold for every set operation on this page:** the
number of columns, and their types, must match across every `SELECT`
involved. **Why:** Postgres is stacking rows into one shared table shape
— if one side returned 2 columns and the other returned 3, there's no
sensible way to combine them, so Postgres refuses with an error before
even running the query.

```sql
-- Column count and types must match across all SELECT statements
-- Aliases come from the FIRST SELECT statement
SELECT id, name, 'customer' AS type FROM customers
UNION ALL
SELECT id, name, 'employee'         FROM employees;
```
Both sides return exactly 3 columns (`id`, `name`, a literal text
value) — types line up (`INT`, `TEXT`, `TEXT`) even though the second
`SELECT` writes its literal without an alias; the *column name* shown in
the final result always comes from the first `SELECT`'s aliases.

```sql
-- UNION with different column sets — use NULL for missing columns
SELECT id, name, email,   NULL AS department FROM customers
UNION ALL
SELECT id, name, NULL,    department         FROM employees;
```
Customers don't have a `department`, employees don't have an `email` —
so each side fills in `NULL` as a placeholder exactly where the other
table's real column would be, keeping both sides at the same 4-column
shape.

Sample output:
| id | name | email | department |
|---|---|---|---|
| 1 | Alice | alice@example.com | NULL |
| 5 | Dan | NULL | Engineering |

### INTERSECT — Common Rows

**Why you need this:** `UNION` combines everything; `INTERSECT` does the
opposite — it keeps **only rows that appear in both** queries' results,
discarding everything that appears in just one.

```sql
-- Emails that appear in BOTH tables (people who are both customers and employees)
SELECT email FROM customers
INTERSECT
SELECT email FROM employees;
```
Only an email present in *both* the `customers` and `employees` results
survives. If `customers` has `alice@example.com, bob@example.com` and
`employees` has `alice@example.com, dan@example.com`, the result is just
`alice@example.com` — the one overlap.

```sql
-- Products that have been ordered AND are in stock
SELECT id FROM products WHERE stock > 0
INTERSECT
SELECT DISTINCT product_id FROM order_items;
```
This answers "which products satisfy both conditions at once" —
in-stock **and** actually ordered before — without needing a `JOIN`.

Sample output (product ids satisfying both):
| id |
|---|
| 3 |
| 7 |

### EXCEPT — Rows in First But Not Second

**Why you need this:** `EXCEPT` is the "subtract" operation — take the
first query's rows, then remove any row that also shows up in the
second query's results. Order matters here, unlike `UNION`/`INTERSECT`
— `A EXCEPT B` is not the same as `B EXCEPT A`.

```sql
-- Customers who have NEVER placed an order
SELECT id FROM customers
EXCEPT
SELECT DISTINCT customer_id FROM orders;
```
Start with every customer id, then remove every id that *does* appear
in `orders.customer_id` — whatever's left are customers with zero
orders. (This is the same result you'd get with the `LEFT JOIN ... WHERE
o.id IS NULL` pattern from [Lesson 06](06-joins-complete.md) — `EXCEPT`
is just a different, sometimes more readable, way to express the same
question.)

Sample output:
| id |
|---|
| 3 |
| 9 |

```sql
-- Products in the catalog but never ordered
SELECT id FROM products
EXCEPT
SELECT DISTINCT product_id FROM order_items;

-- Note: EXCEPT ALL keeps duplicates (like UNION ALL vs UNION)
```
Same "subtract" idea, applied to products instead of customers.
`EXCEPT ALL` is the rarely-needed variant that keeps duplicate rows
around instead of collapsing them — parallel to `UNION` vs `UNION ALL`.

### Combining All Three

**Why you need this:** real reporting questions often chain several of
these together — "get me this set, minus that set, where the first set
was itself built from two unions." Reading a nested one, piece by piece,
is the actual skill.

```sql
-- Multi-level: US and UK customers, minus those who cancelled all orders
(SELECT id FROM customers WHERE country = 'US'
 UNION
 SELECT id FROM customers WHERE country = 'UK')
EXCEPT
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING COUNT(*) = COUNT(CASE WHEN status = 'cancelled' THEN 1 END);
```

**Read it in two steps, inside-out:**
1. The parenthesized part first: `UNION` all US customer ids with all UK
   customer ids → one combined list of ids.
2. Then `EXCEPT` removes any id from that list whose *every single*
   order was cancelled (`COUNT(*) = COUNT(cancelled-only)` means "total
   orders equals cancelled orders" — i.e., nothing but cancellations).

**Result:** US/UK customers who have at least one non-cancelled order
(or no orders at all — worth noticing as an edge case: a customer with
zero orders trivially satisfies neither side of the `HAVING` count
comparison the same way, so check this against your own data if it
matters for a real report).

---

## DISTINCT ON — Keep First Row Per Group

**Why you need this:** you already know `DISTINCT` removes duplicate
*entire rows* ([Lesson 02](02-sql-fundamentals.md)). But a common,
different need is "for each customer, give me just their **most recent**
order" — one representative row per group, chosen by some ordering, not
just deduplicated rows. Plain `DISTINCT` can't do this; `DISTINCT ON` is
PostgreSQL's specific answer to it (not standard SQL — see
[Lesson 20](20-standard-sql-vs-postgresql.md)).

### Build the Intuition First, Without SQL

Imagine sorting every order by customer, then by date (newest first),
and physically laying them out in that order:

```
customer_id | order_date | total
1           | 2024-03-10 | 150   ← newest for customer 1
1           | 2024-02-01 | 90
1           | 2024-01-15 | 200
2           | 2024-03-05 | 75    ← newest for customer 2
2           | 2024-01-20 | 300
```

**`DISTINCT ON (customer_id)` walks down this sorted list and keeps only
the *first* row it sees each time `customer_id` changes** — then throws
away every other row in that group. Applied above, it keeps rows 1 and 4
(the ones marked "newest"), discarding the rest.

### Now the Actual Query

```sql
SELECT DISTINCT ON (customer_id)
    customer_id,
    id AS order_id,
    order_date,
    total,
    status
FROM orders
ORDER BY customer_id, order_date DESC;
```

**Line by line:**
- `DISTINCT ON (customer_id)` — group by `customer_id`, keep exactly one row per group
- `ORDER BY customer_id, order_date DESC` — this `ORDER BY` does double duty: `customer_id` first because `DISTINCT ON` needs rows grouped together to pick from; `order_date DESC` second, so within each customer's group, the newest order sorts first — and "first" is exactly what `DISTINCT ON` keeps

**Sample output**, using the sorted data above:

| customer_id | order_id | order_date | total | status |
|---|---|---|---|---|
| 1 | (whichever id matches 2024-03-10) | 2024-03-10 | 150 | ... |
| 2 | (whichever id matches 2024-03-05) | 2024-03-05 | 75 | ... |

One row per customer — their single most recent order — exactly the
"most recent order per customer" the comment promised, now traceable to
*why* it works instead of just trusting the syntax.

**The one rule you must never break:** the `ORDER BY` **must start with**
the same column(s) named in `DISTINCT ON`, in the same order. `DISTINCT
ON (customer_id) ORDER BY order_date DESC` (without `customer_id` first
in the `ORDER BY`) would pick an essentially arbitrary row per
customer — not the newest one — because the rows would no longer even
be grouped by customer before "first" gets decided.

### The Same Pattern, Two More Times — Now You Can Read Them Yourself

```sql
-- Most expensive product per category
SELECT DISTINCT ON (category)
    category,
    name,
    price
FROM products
ORDER BY category, price DESC;
```
Same logic: group by `category`, sort each group by `price DESC`, keep
the first (= highest-priced) row per group.

| category | name | price |
|---|---|---|
| Electronics | Laptop Pro | 1299.99 |
| Furniture | Office Chair | 249.99 |

```sql
-- Latest employee hired in each department
SELECT DISTINCT ON (department)
    department,
    name,
    hire_date
FROM employees
ORDER BY department, hire_date DESC;
```
Group by `department`, sort each group by `hire_date DESC`, keep the
first (= most recently hired) row per group.

| department | name | hire_date |
|---|---|---|
| Engineering | Dan Lee | 2024-06-01 |
| Sales | Mia Torres | 2024-05-20 |

---

## LATERAL — Power Joins

**Why you need this:** you already solved "most recent order per
customer" with `DISTINCT ON` above — but that only gets you **one**
row per group. What if you need "each customer's **top 2** most recent
orders"? `DISTINCT ON` can't do that; `LATERAL` can. It's also the
answer to a limitation you may not have noticed yet in ordinary
subqueries.

### The Limitation LATERAL Solves

A normal subquery in a `FROM` clause is evaluated **once**, completely
on its own — it cannot see any other table also listed in that `FROM`
clause. Try this (it fails):
```sql
-- This does NOT work — a plain subquery can't see `c` from the outer FROM
SELECT c.name, o.id
FROM customers c,
     (SELECT id FROM orders WHERE customer_id = c.id LIMIT 2) o;
-- ERROR: invalid reference to FROM-clause entry for table "c"
```
The subquery `(SELECT ... WHERE customer_id = c.id ...)` wants to use
`c.id`, but a plain subquery runs in isolation — it has no idea `c`
exists yet.

**`LATERAL` is the keyword that grants exactly this permission:** it
tells Postgres "run this subquery *once for each row* of what comes
before it in the `FROM` clause, and let it reference that row's
columns." Think of `LATERAL` as a `FOR EACH` loop over the outer rows,
re-running the inner subquery fresh for every single one.

### Walking Through the First Example

```sql
-- For each customer, get their most recent 2 orders
SELECT
    c.name,
    o.id,
    o.order_date,
    o.total
FROM customers c
CROSS JOIN LATERAL (
    SELECT id, order_date, total
    FROM orders
    WHERE customer_id = c.id    -- ← references c.id from outer FROM
    ORDER BY order_date DESC
    LIMIT 2
) o;
```

**Line by line:** for every row `c` in `customers`, Postgres now runs
the subquery `o` fresh, substituting that specific customer's `id` into
`WHERE customer_id = c.id`, ordering their orders newest-first, and
keeping just the top 2. `CROSS JOIN LATERAL` then combines each
customer with their (up to) 2 resulting order rows — this is the
"per-customer top-N" query that `DISTINCT ON` alone cannot express.

Sample output (Alice has 3 orders, only her 2 newest show; Carol has 0
orders and disappears entirely — see the next example for why):
| name | id | order_date | total |
|---|---|---|---|
| Alice | 12 | 2024-03-10 | 150.00 |
| Alice | 9  | 2024-02-01 | 90.00 |
| Bob | 15 | 2024-03-05 | 75.00 |

**Why Carol (0 orders) vanished:** `CROSS JOIN` (even the `LATERAL`
kind) only keeps a customer row if the subquery returns *at least one*
matching row — exactly the same "inner join drops non-matches" behavior
from [Lesson 06](06-joins-complete.md), just with a subquery standing in
for a second table.

### Fixing That With LEFT JOIN LATERAL

```sql
-- LATERAL with LEFT JOIN (include customers with no orders)
SELECT
    c.name,
    o.id,
    o.order_date
FROM customers c
LEFT JOIN LATERAL (
    SELECT id, order_date
    FROM orders
    WHERE customer_id = c.id
    ORDER BY order_date DESC
    LIMIT 1
) o ON TRUE;  -- ON TRUE because the correlation is inside the subquery
```

Same idea as [Lesson 06](06-joins-complete.md)'s `LEFT JOIN` — keep
**every** row from `customers`, even ones with no matching order, filling
`o.id`/`o.order_date` with `NULL` when there's nothing to join. **Why
`ON TRUE` and not a real condition:** the actual matching logic
(`customer_id = c.id`) already lives *inside* the subquery itself — by
the time Postgres reaches the `ON` clause, there's nothing left to
compare, so `ON TRUE` just means "always attach whatever this subquery
produced, join condition already handled."

Sample output (now Carol appears, with NULLs):
| name | id | order_date |
|---|---|---|
| Alice | 12 | 2024-03-10 |
| Bob | 15 | 2024-03-05 |
| Carol | NULL | NULL |

### Unnesting an Array, Row by Row

```sql
-- Unnest array with row context
SELECT
    p.name,
    tag
FROM products p
CROSS JOIN LATERAL UNNEST(p.tags) AS tag
WHERE tag IS NOT NULL;
-- Each tag becomes its own row, keeping the product context
```

If `products.tags` is an array column (e.g., `{'electronics','sale'}`
for one product — see [Lesson 16](16-postgresql-power-features.md)),
`UNNEST` explodes that array into one row per element. `LATERAL` here
lets each product's *own* `tags` array feed `UNNEST`, row by row, rather
than trying to unnest one array in isolation with no idea which
product it belongs to.

Sample output:
| name | tag |
|---|---|
| Laptop Pro | electronics |
| Laptop Pro | sale |
| Desk Lamp | home |

---

## Pivoting — Rows to Columns

**Why you need this:** so far, `GROUP BY` always turns "many rows" into
"fewer rows, one per group." Pivoting is a specific reporting shape
where you instead want each group's *sub-categories* to become
**separate columns** — e.g., one row per month, but a separate column
for each order status's revenue, instead of one row per month+status
combination. Spreadsheet users know this as a "pivot table"; PostgreSQL
has no dedicated `PIVOT` keyword, so you build the same effect yourself
with `CASE` + `GROUP BY`.

### The Trick, Explained Before the Full Query

Recall `CASE` from [Lesson 04](04-filtering-deep-dive.md) can return a
value per row. Combine that with `SUM`: **`SUM(CASE WHEN condition THEN
value ELSE 0 END)` only adds up the rows matching `condition`, and
contributes `0` for every other row.** Do this once per status you care
about, in the same `SELECT`, and each `CASE` expression becomes its own
column — one column per status, all computed in a single pass over the
data.

```sql
-- Sales by month per status (pivot)
SELECT
    TO_CHAR(order_date, 'YYYY-MM') AS month,
    SUM(CASE WHEN status = 'delivered' THEN total ELSE 0 END) AS delivered_revenue,
    SUM(CASE WHEN status = 'shipped'   THEN total ELSE 0 END) AS shipped_revenue,
    SUM(CASE WHEN status = 'pending'   THEN total ELSE 0 END) AS pending_revenue,
    SUM(CASE WHEN status = 'cancelled' THEN total ELSE 0 END) AS cancelled_revenue,
    SUM(total) AS total_revenue
FROM orders
GROUP BY TO_CHAR(order_date, 'YYYY-MM')
ORDER BY month;
```

**Without pivoting**, a plain `GROUP BY month, status` would give you
one row *per month per status* — many rows, hard to scan. Pivoting
collapses that into **one row per month**, with each status as its own
column instead of its own row:

| month | delivered_revenue | shipped_revenue | pending_revenue | cancelled_revenue | total_revenue |
|---|---|---|---|---|---|
| 2024-01 | 1200.00 | 300.00 | 150.00 | 0.00 | 1650.00 |
| 2024-02 | 950.00 | 0.00 | 400.00 | 90.00 | 1440.00 |

```sql
-- Category count by country (pivot)
SELECT
    country,
    COUNT(*) FILTER (WHERE city = 'New York')     AS new_york,
    COUNT(*) FILTER (WHERE city = 'London')       AS london,
    COUNT(*) FILTER (WHERE city = 'Toronto')      AS toronto,
    COUNT(*) FILTER (WHERE city NOT IN ('New York','London','Toronto')) AS other
FROM customers
GROUP BY country;
```
Same trick, using `FILTER` ([Lesson 05](05-aggregations.md)) instead of
`CASE` inside `SUM` — `COUNT(*) FILTER (WHERE city = 'New York')` only
counts rows matching that city, contributing to its own column.

| country | new_york | london | toronto | other |
|---|---|---|---|---|
| US | 4 | 0 | 0 | 2 |
| UK | 0 | 3 | 0 | 1 |
| Canada | 0 | 0 | 2 | 0 |

### When You Genuinely Don't Know the Columns Ahead of Time

The `CASE`/`FILTER` trick above requires you to **hand-write one column
per value** (`delivered`, `shipped`, ... ). That's fine when the set of
statuses is small and fixed. But "one column per year an employee was
hired" could mean an unknown, growing number of columns as years pass —
hand-writing them doesn't scale. This is what the `crosstab` extension
solves:

```sql
-- Using crosstab (tablefunc extension — more powerful pivot)
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM crosstab(
    'SELECT department, EXTRACT(YEAR FROM hire_date)::TEXT, COUNT(*)
     FROM employees
     GROUP BY 1, 2
     ORDER BY 1, 2',
    'SELECT DISTINCT EXTRACT(YEAR FROM hire_date)::TEXT FROM employees ORDER BY 1'
) AS ct (department TEXT, "2017" BIGINT, "2018" BIGINT, "2019" BIGINT, "2020" BIGINT, "2021" BIGINT, "2022" BIGINT);
```

**Breaking down `crosstab`'s two arguments:**
1. **The first string** is a query returning exactly 3 columns: *row
   label* (`department`), *column label* (`year`), *value* (`count`) —
   this is the raw, "long format" data crosstab will reshape.
2. **The second string** is a query listing every distinct column label
   that should appear, in order — this is what tells crosstab how many
   output columns to create and in what order.
3. **The final `AS ct (...)`** must then manually declare the resulting
   column names and types to match — this part still requires knowing
   the years in advance to write the column list, which is `crosstab`'s
   one real limitation (this is a known Postgres quirk, not something
   you're doing wrong).

Sample output:
| department | 2017 | 2018 | 2019 | 2020 | 2021 | 2022 |
|---|---|---|---|---|---|---|
| Engineering | 1 | 2 | 0 | 3 | 1 | 2 |
| Sales | 0 | 1 | 2 | 1 | 0 | 1 |

**When to reach for which:** use the `CASE`/`FILTER` trick when the
categories are small and known ahead of time (order statuses, a handful
of cities). Reach for `crosstab` only when the categories are numerous
or dynamic (every year, every product SKU) — it's more powerful, but
also more setup for a one-off report.

---

## GENERATE_SERIES — Generate Data

**Why you need this:** every query so far only shows data that *exists*
in a table. But a report asking "revenue per day for Q1" needs a row
for **every single day**, including days with zero orders — and there's
no table row to `SELECT` for a day that had nothing happen.
`generate_series` manufactures rows out of thin air, specifically to
plug this gap.

```sql
-- Generate a sequence of integers
SELECT * FROM generate_series(1, 10);
-- 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 — one row per number

SELECT * FROM generate_series(0, 100, 5);  -- 0, 5, 10, ..., 100
-- third argument is the step size — here, count by 5s
```

```sql
-- Generate dates
SELECT d::DATE
FROM generate_series(
    '2024-01-01'::TIMESTAMP,
    '2024-12-31'::TIMESTAMP,
    '1 day'::INTERVAL
) d;
```
Same idea as the integer version, but stepping through **dates**
instead of numbers — start, end, and step size (using `INTERVAL`, from
[Lesson 02](02-sql-fundamentals.md)) — producing one row per day of the
entire year.

```sql
-- Generate months
SELECT d::DATE AS month_start
FROM generate_series(
    '2024-01-01'::TIMESTAMP,
    '2024-12-01'::TIMESTAMP,
    '1 month'::INTERVAL
) d;
```
Identical mechanism, just a `'1 month'` step instead of `'1 day'` —
produces one row per month's start date for the whole year.

### The Actual Payoff — Filling Gaps in Real Data

```sql
-- Fill in gaps in time series data
WITH calendar AS (
    SELECT d::DATE AS day
    FROM generate_series(
        '2024-01-01'::TIMESTAMP,
        '2024-03-31'::TIMESTAMP,
        '1 day'
    ) d
),
daily_orders AS (
    SELECT order_date, COUNT(*) AS order_count, SUM(total) AS revenue
    FROM orders
    GROUP BY order_date
)
SELECT
    c.day,
    COALESCE(d.order_count, 0) AS orders,
    COALESCE(d.revenue, 0) AS revenue
FROM calendar c
LEFT JOIN daily_orders d ON d.order_date = c.day
ORDER BY c.day;
-- This shows EVERY day, even days with no orders (COALESCE fills 0)
```

**Walk through what's really happening, piece by piece:**
1. `calendar` (a CTE, from [Lesson 07](07-subqueries-and-ctes.md)) uses
   `generate_series` to manufacture one row **for every single day** in
   Q1 — 91 rows, regardless of whether any orders happened that day.
2. `daily_orders` is the *real* data — actual order counts and revenue,
   but **only for days that had at least one order**. A day with zero
   orders simply has no row here at all.
3. `LEFT JOIN calendar (all 91 days) with daily_orders (only days with
   orders)` — exactly the "keep everything from the left side, NULL
   where there's no match" behavior from [Lesson 06](06-joins-complete.md).
   Days with no orders get `NULL` for `order_count`/`revenue`.
4. `COALESCE(..., 0)` ([Lesson 04](04-filtering-deep-dive.md)) turns
   those `NULL`s into `0` — a day with genuinely zero orders should show
   `0`, not a blank.

**Sample output** — notice `2024-01-03` has real data, but
`2024-01-04` had no orders and still gets a row, correctly zeroed:
| day | orders | revenue |
|---|---|---|
| 2024-01-01 | 3 | 450.00 |
| 2024-01-02 | 1 | 90.00 |
| 2024-01-03 | 2 | 275.00 |
| 2024-01-04 | 0 | 0.00 |

**Why this matters for real reporting:** a chart or dashboard with
silently missing days (instead of explicit zeros) looks broken — a gap
in a line chart reads as "something went wrong," not "nothing happened
that day." This pattern is the standard fix.

---

## Deep Dive: String Functions

**Why you need this:** this section is a **reference**, not a
narrative — you're not meant to memorize every line. Each function
already shows its own output as an inline comment, so treat this like a
dictionary: skim once to know what exists, then come back and look up
the exact one you need when a real query calls for it. A few
easily-confused pairs are worth reading closely (`SUBSTRING` is
1-indexed, not 0-indexed like most programming languages; `LPAD` with no
padding character truncates rather than errors) — those are called out
inline below.

```sql
-- Length
SELECT LENGTH('Hello World');         -- 11
SELECT CHAR_LENGTH('Hello');           -- 5 (synonym)
SELECT BIT_LENGTH('Hello');            -- 40 (bits)
SELECT OCTET_LENGTH('Hello');          -- 5 (bytes)

-- Case
SELECT UPPER('hello');                 -- HELLO
SELECT LOWER('HELLO');                 -- hello
SELECT INITCAP('hello world');         -- Hello World

-- Trim
SELECT TRIM('  spaces  ');             -- 'spaces'
SELECT TRIM(BOTH 'x' FROM 'xxxhelloxxx'); -- 'hello'
SELECT LTRIM('  left  ');              -- 'left  '
SELECT RTRIM('  right  ');             -- '  right'

-- Pad
SELECT LPAD('42', 6, '0');             -- '000042'
SELECT RPAD('yes', 5, '.');            -- 'yes..'
SELECT LPAD('hello', 3);               -- 'hel' (truncates if longer!)

-- Substring
SELECT SUBSTRING('hello world', 1, 5);  -- 'hello' (1-indexed)
SELECT SUBSTRING('hello world' FROM 7); -- 'world'
SELECT SUBSTR('hello', 2, 3);           -- 'ell'

-- Position / Index
SELECT POSITION('world' IN 'hello world');  -- 7
SELECT STRPOS('hello world', 'world');      -- 7 (same)

-- Replace
SELECT REPLACE('hello world', 'world', 'SQL'); -- 'hello SQL'

-- Repeat
SELECT REPEAT('ab', 3);                -- 'ababab'

-- Reverse
SELECT REVERSE('hello');               -- 'olleh'

-- Split
SELECT SPLIT_PART('a,b,c,d', ',', 2);  -- 'b' (2nd element)
SELECT STRING_TO_ARRAY('a,b,c', ',');  -- ARRAY['a','b','c']
SELECT UNNEST(STRING_TO_ARRAY('a,b,c', ',')); -- 3 rows

-- Format
SELECT FORMAT('Hello %s, you have %s messages', 'Alice', 5);
-- 'Hello Alice, you have 5 messages'

-- Regular expressions
SELECT REGEXP_REPLACE('abc123def', '[0-9]+', 'NUM');  -- 'abcNUMdef'
SELECT REGEXP_REPLACE('abc123def456', '[0-9]+', 'N', 'g');  -- 'abcNdefN' (global)
SELECT REGEXP_MATCHES('a1b2c3', '[0-9]', 'g');  -- array of matches
SELECT (REGEXP_MATCH('user@example.com', '^(.+)@(.+)$'))[1]; -- 'user'

-- Encode / Decode
SELECT ENCODE('hello'::BYTEA, 'base64');  -- 'aGVsbG8='
SELECT DECODE('aGVsbG8=', 'base64')::TEXT; -- 'hello'

-- MD5 hash
SELECT MD5('password');  -- '5f4dcc3b5aa765d61d8327deb882cf99'
```

---

## Deep Dive: Date/Time Functions

**Why you need this:** same reference-style section as string
functions above — skim for what exists, look things up as needed rather
than memorizing. `INTERVAL` arithmetic itself was already covered in
depth back in [Lesson 02](02-sql-fundamentals.md); everything below is
just the fuller function library built on that same idea. The one
genuinely tricky bit worth reading closely is `EXTRACT(DOW ...)` vs
`EXTRACT(ISODOW ...)` — they number weekdays differently (`DOW` starts
at Sunday=0, `ISODOW` starts at Monday=1), a classic source of off-by-one
bugs in date reports.

```sql
-- Current values
SELECT NOW();                         -- current timestamp with timezone
SELECT CURRENT_DATE;                  -- current date
SELECT CURRENT_TIME;                  -- current time
SELECT CURRENT_TIMESTAMP;             -- same as NOW()
SELECT LOCALTIME;                     -- current time without timezone
SELECT LOCALTIMESTAMP;                -- current timestamp without timezone
SELECT CLOCK_TIMESTAMP();             -- actual current time (changes during queries)
SELECT TRANSACTION_TIMESTAMP();       -- time at start of transaction (same as NOW())
SELECT STATEMENT_TIMESTAMP();         -- time at start of statement

-- Extract parts
SELECT EXTRACT(YEAR   FROM NOW());    -- 2024
SELECT EXTRACT(MONTH  FROM NOW());    -- 3
SELECT EXTRACT(DAY    FROM NOW());    -- 15
SELECT EXTRACT(HOUR   FROM NOW());    -- 14
SELECT EXTRACT(MINUTE FROM NOW());    -- 30
SELECT EXTRACT(SECOND FROM NOW());    -- 22.123456
SELECT EXTRACT(DOW    FROM NOW());    -- 5 (0=Sun, 6=Sat)
SELECT EXTRACT(ISODOW FROM NOW());    -- 5 (1=Mon, 7=Sun — ISO standard)
SELECT EXTRACT(DOY    FROM NOW());    -- day of year (1-366)
SELECT EXTRACT(WEEK   FROM NOW());    -- ISO week number
SELECT EXTRACT(QUARTER FROM NOW());   -- 1-4
SELECT EXTRACT(EPOCH  FROM NOW());    -- Unix timestamp (seconds since 1970-01-01)

-- Date_part (older equivalent of EXTRACT)
SELECT DATE_PART('year', NOW());      -- same as EXTRACT

-- Truncate to period boundary
SELECT DATE_TRUNC('year',    NOW());  -- 2024-01-01 00:00:00
SELECT DATE_TRUNC('quarter', NOW());  -- 2024-01-01 (start of Q1)
SELECT DATE_TRUNC('month',   NOW());  -- 2024-03-01
SELECT DATE_TRUNC('week',    NOW());  -- 2024-03-11 (Monday)
SELECT DATE_TRUNC('day',     NOW());  -- 2024-03-15 00:00:00
SELECT DATE_TRUNC('hour',    NOW());  -- 2024-03-15 14:00:00

-- Arithmetic with intervals
SELECT NOW() + INTERVAL '1 year';
SELECT NOW() + INTERVAL '2 months 3 days 4 hours';
SELECT '2024-03-15'::DATE + 30;                    -- +30 days
SELECT '2024-03-15'::DATE - '2024-01-01'::DATE;   -- days difference = 74
SELECT AGE(NOW(), '2000-01-01'::DATE);             -- interval: 24 years 2 months...
SELECT AGE('2024-03-15'::DATE);                    -- age from today

-- Format / Parse
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS');
SELECT TO_CHAR(NOW(), 'Day, DD Month YYYY');       -- 'Friday , 15 March  2024'
SELECT TO_CHAR(NOW(), 'Mon DD, YYYY');             -- 'Mar 15, 2024'
SELECT TO_CHAR(12345.67, 'FM$999,999.00');         -- '$12,345.67'
SELECT TO_DATE('15/03/2024', 'DD/MM/YYYY');
SELECT TO_TIMESTAMP('2024-03-15 14:30', 'YYYY-MM-DD HH24:MI');

-- Timezone
SELECT NOW() AT TIME ZONE 'America/New_York';
SELECT NOW() AT TIME ZONE 'UTC';
SELECT NOW() AT TIME ZONE 'Asia/Seoul';

-- Make date from parts
SELECT MAKE_DATE(2024, 3, 15);            -- 2024-03-15
SELECT MAKE_TIMESTAMP(2024, 3, 15, 14, 30, 0);  -- 2024-03-15 14:30:00
SELECT MAKE_INTERVAL(years=>1, months=>2, days=>3);

-- End of month
SELECT DATE_TRUNC('month', NOW()) + INTERVAL '1 month' - INTERVAL '1 day';
-- First day of next month - 1 day = last day of current month
```

---

## Deep Dive: Mathematical Functions

**Why you need this:** same reference-style treatment as the two
sections above. Two things worth reading closely rather than skimming:
`17 / 5` returning `3` (integer division truncates — the exact same
gotcha explained in depth back when you asked about
`SUM(stock)/COUNT(*)`; cast to `NUMERIC` or use `17.0 / 5` to get a real
decimal), and `RANDOM()` combined with `SETSEED()` — seeding makes
"random" values reproducible, which matters if you ever need a test
dataset that generates the same "random" numbers every run.

```sql
-- Basic
SELECT ABS(-42);                  -- 42
SELECT SIGN(-42);                 -- -1 (or 0 or 1)
SELECT MOD(17, 5);                -- 2 (remainder)
SELECT 17 % 5;                    -- 2 (same)
SELECT DIV(17, 5);                -- 3 (integer division)
SELECT 17 / 5;                    -- 3 (integer division in SQL)
SELECT 17.0 / 5;                  -- 3.4

-- Power and roots
SELECT POWER(2, 10);              -- 1024
SELECT 2 ^ 10;                    -- 1024 (PostgreSQL shorthand)
SELECT SQRT(144);                 -- 12
SELECT CBRT(27);                  -- 3 (cube root)
SELECT EXP(1);                    -- 2.71828... (e^x)
SELECT LN(2.71828);               -- ~1 (natural log)
SELECT LOG(100);                  -- 2 (log base 10)
SELECT LOG(2, 1024);              -- 10 (log base 2)

-- Rounding
SELECT ROUND(3.456, 2);          -- 3.46
SELECT ROUND(3.454, 2);          -- 3.45
SELECT TRUNC(3.999, 1);          -- 3.9 (truncate, no rounding)
SELECT FLOOR(3.7);                -- 3 (round down)
SELECT CEIL(3.2);                 -- 4 (round up)
SELECT CEILING(3.2);              -- 4 (same as CEIL)

-- Trigonometry
SELECT SIN(PI() / 2);            -- 1
SELECT COS(0);                   -- 1
SELECT TAN(PI() / 4);            -- 1
SELECT DEGREES(PI());            -- 180
SELECT RADIANS(180);             -- 3.14159...

-- Statistics
SELECT RANDOM();                  -- random float 0.0 to 1.0
-- Seed for reproducible random
SELECT SETSEED(0.5);
SELECT RANDOM(), RANDOM();        -- reproducible after setseed

-- Statistical aggregates
SELECT
    STDDEV(salary)    AS std_deviation,
    VARIANCE(salary)  AS variance,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) AS p25,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) AS p75,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY salary) AS p90
FROM employees;

-- MODE — most common value
SELECT MODE() WITHIN GROUP (ORDER BY department) FROM employees;
```

**Why you need this:** every aggregate you've used so far —
`COUNT`/`SUM`/`AVG`/`MIN`/`MAX` ([Lesson 05](05-aggregations.md)) —
takes a plain column and produces one number, with no internal ordering
involved. This query introduces a different *kind* of aggregate — one
whose calculation genuinely depends on the data's **sorted order**,
which is why `WITHIN GROUP (ORDER BY ...)` looks unlike anything you've
written before.

**`STDDEV(salary)` and `VARIANCE(salary)`** — ordinary aggregates, same
shape as `AVG`. Both measure "how spread out are the salaries from the
average" — `VARIANCE` is the raw spread measure, `STDDEV` (standard
deviation) is its square root, which is why it's in the same units as
salary itself and easier to interpret directly (e.g. "salaries typically
vary by about $8,000 from the average").

**`PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)`** — this is the
unusual one. Break it into two pieces:
- `PERCENTILE_CONT(0.5)` — "find the value at the 50th percentile" (0.5
  = 50%). The 50th percentile is just another name for the **median**:
  the value that splits the data exactly in half, half below, half
  above.
- `WITHIN GROUP (ORDER BY salary)` — this tells Postgres *which column,
  sorted in which direction* to compute that percentile against. It
  exists as a separate clause (instead of just writing
  `PERCENTILE_CONT(0.5, salary)`) specifically because a percentile is
  meaningless without sorting the data first — this syntax makes that
  sorting step explicit and unmissable.

**Why not just use `AVG(salary)` instead of the median?** Average gets
pulled toward extreme outliers — one executive's very high salary drags
the average up for everyone. Median doesn't move nearly as much, since
it only cares about the middle position, not the actual size of extreme
values — which is why "median household income" is the number you see
quoted in real-world statistics, not the average.

**The four `PERCENTILE_CONT` calls together** are computing a full
spread of the salary distribution — 25th percentile, median (50th),
75th, and 90th — the same idea as a box plot's five-number summary,
letting you see the whole shape of the data, not just its center.

**Trace it against sample data** — say `employees.salary` sorted is
`[45000, 52000, 58000, 61000, 67000, 72000, 95000]` (7 employees):

| std_deviation | variance | median | p25 | p75 | p90 |
|---|---|---|---|---|---|
| ~15800 | ~2.5e8 | 61000 | 52000 | 72000 | ~88600 |

The median (`61000`) is the exact middle value of the 7 sorted salaries
— `PERCENTILE_CONT` interpolates between values when the requested
percentile doesn't land exactly on one row (hence "CONT" for
*continuous* — it's allowed to return a value that didn't literally
appear in the data, unlike its sibling `PERCENTILE_DISC`, which is
restricted to returning an actual existing row's value).

**`MODE() WITHIN GROUP (ORDER BY department)`** — same `WITHIN GROUP`
syntax, different question: "which single value appears most often?"
(the statistical *mode*). If `Engineering` has more employees than any
other department, `MODE()` returns `'Engineering'` — again, the
`ORDER BY` inside `WITHIN GROUP` names which column this is computed
over, not an actual sort applied to the final result.

---

## Advanced Aggregation Functions

**Why you need this:** unlike the reference sections above, these
functions are genuinely non-obvious — they're statistics, not just
string/date utilities — so this section gets the full walkthrough
treatment, same as the rest of this lesson.

### PERCENTILE_CONT vs PERCENTILE_DISC — Two Different Ideas of "Median"

**The problem plain `AVG` doesn't solve:** an average can be skewed by
one extreme outlier (one $10M salary drags the average way up). A
**median** — the middle value when everything is sorted — doesn't have
that problem. `PERCENTILE_CONT(0.5)` computes the median; `0.5` means
"the value at the 50% mark," and other fractions give other percentiles
(`0.25` = 25th percentile, `0.9` = 90th percentile, etc.).

```sql
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_interpolated,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY salary) AS median_actual
FROM employees;
```

**The difference between `CONT` and `DISC`, concretely.** Say sorted
salaries are `50000, 60000, 70000, 80000` (4 values, no exact middle
one):
- **`PERCENTILE_DISC`** (discrete) always returns an **actual value
  that exists in the data** — here, `60000` (the closest real row at or
  past the 50% mark).
- **`PERCENTILE_CONT`** (continuous) **interpolates** between the two
  middle values — here, halfway between `60000` and `70000` = `65000`,
  a number that may not correspond to any real employee's actual salary.

`WITHIN GROUP (ORDER BY salary)` is required syntax — it tells Postgres
*which column* to rank rows by before computing the percentile.

### CORR — Do Two Numbers Move Together?

```sql
-- CORR: correlation coefficient (-1 to 1)
-- Does salary correlate with years of service?
SELECT CORR(salary, EXTRACT(YEAR FROM hire_date)) AS salary_tenure_correlation
FROM employees;
```

`CORR(x, y)` returns a single number between **-1 and 1** describing how
strongly two columns move together:
- **Close to `+1`** — as one goes up, the other reliably goes up too
  (longer tenure → higher salary, a believable real-world relationship)
- **Close to `-1`** — as one goes up, the other reliably goes down
- **Close to `0`** — no real relationship between them

This single number is a quick sanity check before assuming "X affects
Y" — e.g., "do people who've been here longer really earn more, or is
that just an assumption?"

### COVAR_POP / COVAR_SAMP — Covariance (a Building Block, Rarely Read Directly)

```sql
-- COVAR_POP, COVAR_SAMP: covariance
SELECT COVAR_POP(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date))
FROM employees;
```
Covariance also measures "do these two move together," but unlike
`CORR`, its result isn't scaled to a fixed -1..1 range — the raw number
is hard to interpret on its own (it depends on the units of both
columns). In practice, `CORR` is what you'd actually read; covariance
mostly matters as the math `CORR` and the regression functions below
are built from.

### REGR_* — Fitting a Straight Line Through the Data

```sql
-- REGR_* functions for linear regression
SELECT
    REGR_SLOPE(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date))     AS slope,
    REGR_INTERCEPT(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date)) AS intercept,
    REGR_R2(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date))        AS r_squared
FROM employees;
```

These fit the classic "line of best fit" (`y = slope * x + intercept`)
through your data — here, `x` = years of tenure, `y` = salary:
- **`REGR_SLOPE`** — for each additional year of tenure, how much does
  salary tend to increase? (e.g., a slope of `2000` means "roughly
  $2,000 more per year of tenure")
- **`REGR_INTERCEPT`** — the predicted salary at zero years of tenure
  (where the fitted line crosses the y-axis)
- **`REGR_R2`** — how well the straight line actually fits the real
  data, from `0` (the line explains nothing) to `1` (the line predicts
  the data perfectly). A low `R2` means "tenure alone doesn't really
  explain salary" even if `SLOPE` looks meaningful.

Sample output for all four together:
| median_interpolated | median_actual | salary_tenure_correlation | slope | r_squared |
|---|---|---|---|---|
| 65000.00 | 60000.00 | 0.62 | 2100.00 | 0.38 |

**Reading this row:** a moderate positive correlation (`0.62`) and a
slope suggesting ~$2,100/year of tenure — but an `R2` of `0.38` means
tenure only explains about 38% of the variation in salary; other
factors (role, department, performance) clearly matter too.

---

## Complete Advanced Query Examples

**Why these are worth reading slowly:** both examples below combine
several techniques from this entire lesson into one query — exactly the
skill [Lesson 22](22-real-world-schema-walkthrough.md) is about:
recognizing which tool a real requirement calls for, not just knowing
each tool in isolation.

**Example 1 — Complete time series with zero-fill:**
```sql
WITH months AS (
    SELECT m::DATE AS month
    FROM generate_series('2024-01-01', '2024-03-01', '1 month'::INTERVAL) m
),
monthly_orders AS (
    SELECT DATE_TRUNC('month', order_date)::DATE AS month,
           COUNT(*) AS orders,
           SUM(total) AS revenue
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    m.month,
    COALESCE(mo.orders, 0) AS orders,
    COALESCE(mo.revenue, 0) AS revenue,
    ROUND(mo.revenue / NULLIF(mo.orders, 0), 2) AS avg_order_value
FROM months m
LEFT JOIN monthly_orders mo ON mo.month = m.month
ORDER BY m.month;
```

**What this combines, piece by piece:** `months` is the
`generate_series` gap-filling pattern from earlier in this lesson,
guaranteeing a row for every month even with zero orders. `monthly_orders`
is a normal aggregation. The `LEFT JOIN` + `COALESCE` pair is the same
"keep every row, zero-fill the missing side" pattern explained in full
under `GENERATE_SERIES` above. The one new piece: `NULLIF(mo.orders, 0)`
inside the division — this exists specifically to prevent a
**divide-by-zero error** for any month with 0 orders; `NULLIF` turns a
`0` count into `NULL` first, and dividing by `NULL` safely produces
`NULL` instead of crashing (revisit [Lesson 04](04-filtering-deep-dive.md)
if `NULLIF` itself is unfamiliar).

Sample output:
| month | orders | revenue | avg_order_value |
|---|---|---|---|
| 2024-01-01 | 12 | 3400.00 | 283.33 |
| 2024-02-01 | 0 | 0.00 | NULL |
| 2024-03-01 | 9 | 2100.00 | 233.33 |

**Example 2 — Pivoted report:**
```sql
SELECT
    department,
    COUNT(*) AS total_employees,
    SUM(salary) AS total_payroll,
    COUNT(*) FILTER (WHERE hire_date >= '2021-01-01') AS hired_2021_plus,
    COUNT(*) FILTER (WHERE manager_id IS NULL) AS managers,
    ROUND(AVG(salary), 0) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees
GROUP BY department
ORDER BY total_payroll DESC;
```

**What this combines:** a plain `GROUP BY` ([Lesson 05](05-aggregations.md))
as the backbone, with two `FILTER`-based conditional counts (the
pivoting technique from earlier in this lesson) mixed into the *same*
row as ordinary aggregates — proof that pivoted columns and normal
aggregates can coexist in one `SELECT`, they're not mutually exclusive
techniques.

Sample output:
| department | total_employees | total_payroll | hired_2021_plus | managers | avg_salary | min_salary | max_salary |
|---|---|---|---|---|---|---|---|
| Engineering | 8 | 720000 | 3 | 1 | 90000 | 65000 | 145000 |
| Sales | 5 | 310000 | 2 | 1 | 62000 | 48000 | 95000 |

---

## Exercises

**Exercise 1:** Use GENERATE_SERIES to create a daily calendar for January 2024.
LEFT JOIN to orders to show orders and revenue per day. Fill missing days with 0.

**Exercise 2:** Use DISTINCT ON to get the most recent order per customer.
Then join to the customers table to show customer name and their last order date.

**Exercise 3:** Using UNION ALL, create a combined list of "transactions":
- Orders with type='order', date=order_date, amount=total
- Products with type='inventory', date=created_at, amount=price * stock (inventory value)
Order by date.

**Exercise 4:** Pivot the employee table:
Show one row per department, with columns for each year of hire (2017-2022),
showing how many employees were hired in each department per year.

---

## Key Takeaways

1. UNION removes duplicates; UNION ALL doesn't — prefer UNION ALL for performance
2. INTERSECT finds common rows; EXCEPT finds rows in first but not second
3. DISTINCT ON (cols) keeps the first row per distinct value — ORDER BY must start with those cols
4. LATERAL lets a FROM subquery reference columns from earlier in the FROM clause
5. GENERATE_SERIES fills time gaps — essential for reporting
6. PERCENTILE_CONT(0.5) is the correct way to compute median in SQL
7. Pivoting uses CASE WHEN inside SUM/COUNT with GROUP BY

---

## Next Lesson
[Lesson 13 — Views & Materialized Views](13-views-and-materialized-views.md)
