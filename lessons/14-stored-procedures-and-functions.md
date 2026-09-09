# Lesson 14 — Stored Procedures & Functions

## Goal
Write reusable database-side logic using PL/pgSQL. Learn functions, procedures,
and triggers — the tools that let your database enforce business logic.

## Prerequisites
- [Lesson 13](13-views-and-materialized-views.md) — views

## After This Lesson You Will Be Able To
- Write SQL and PL/pgSQL functions
- Create stored procedures
- Use triggers to automate database actions
- Return tables and record sets from functions
- Write functions with error handling

---

## Why This Whole Lesson Exists

**Why you need this:** every earlier lesson taught you to write SQL that
*you* run, one statement at a time. This lesson is different — it's
about **teaching the database itself a piece of logic once**, so it can
be reused (a function you call from many queries), run with full
procedural control (loops, if/else — like a small program instead of a
single query), or triggered automatically whenever data changes (a
trigger, with zero effort from your application).

This is also the first lesson where you write genuinely **procedural**
code — step-by-step instructions, variables, loops — instead of a
single declarative `SELECT` that describes *what* you want without
saying *how* to compute it. If you've never written procedural code
inside a database before, that shift is the main new mental model here;
everything else builds on it.

---

## Functions vs Procedures — The Difference

| | Function | Procedure |
|--|----------|-----------|
| Return value | Must return a value | No return value |
| Call syntax | `SELECT my_func()` | `CALL my_proc()` |
| Transactions | Can't manage transactions | Can COMMIT/ROLLBACK |
| Use in SQL | Yes (in SELECT, WHERE) | No |
| Added in | Early PostgreSQL | PostgreSQL 11+ |

**The one-line way to decide which you need:** *"Do I need this to
compute and hand back a value I'll use inside a query (`SELECT`,
`WHERE`)?"* → function. *"Do I need this to perform a multi-step
operation — like 'place an order,' which touches several tables and
should commit or roll back as a whole ([Lesson 10](10-transactions-and-acid.md))?"*
→ procedure.

---

## SQL Functions — Simplest Form

**Why you need this:** the plainest possible function — no procedural
logic yet, just "wrap a query in a name so I can call it like a value."
This is the smallest possible step up from a query you already know how
to write.

```sql
-- Function that returns a scalar value
CREATE OR REPLACE FUNCTION get_customer_count()
RETURNS INT
LANGUAGE SQL
AS $$
    SELECT COUNT(*)::INT FROM customers;
$$;

-- Call it
SELECT get_customer_count();
```

**Reading this line by line:**
- `RETURNS INT` — declares what type comes back; the body's query must produce exactly that
- `LANGUAGE SQL` — the function body is plain SQL, nothing procedural (no `IF`, no loops — that comes with PL/pgSQL below)
- `AS $$ ... $$` — everything between the two `$$` markers is the function body; `$$` is just a way to mark "start/end of a chunk of text" without fighting over quote characters inside it
- The body is literally the query from [Lesson 05](05-aggregations.md), just wrapped so you can call it by name

**Output:**
```sql
SELECT get_customer_count();
--  get_customer_count
-- ---------------------
--                   10
```

**Adding a parameter — same idea, now it takes an input:**

```sql
-- Function with parameters
CREATE OR REPLACE FUNCTION get_orders_by_status(p_status TEXT)
RETURNS INT
LANGUAGE SQL
AS $$
    SELECT COUNT(*)::INT FROM orders WHERE status = p_status;
$$;

SELECT get_orders_by_status('delivered');
SELECT get_orders_by_status('pending');
```

`p_status` is a parameter — same as a function argument in any
programming language. Whatever you pass in (`'delivered'`) becomes the
value of `p_status` inside the function body's `WHERE` clause.

| Call | Output |
|---|---|
| `get_orders_by_status('delivered')` | `5` |
| `get_orders_by_status('pending')` | `3` |

**The `p_` prefix is a naming convention, not a requirement** — it just
means "parameter," making it visually distinct from a table's own
column names when you're reading the function body later. You'll see
this convention throughout the rest of this lesson.

```sql
-- Function with default parameters
CREATE OR REPLACE FUNCTION get_orders_by_status(p_status TEXT DEFAULT 'pending')
RETURNS INT LANGUAGE SQL AS $$
    SELECT COUNT(*)::INT FROM orders WHERE status = p_status;
$$;

SELECT get_orders_by_status();           -- uses 'pending'
SELECT get_orders_by_status('shipped');  -- explicit
```
`DEFAULT 'pending'` means "if the caller doesn't pass an argument at
all, use `'pending'`" — same concept as a default parameter value in
any language. Calling with `()` and no argument is valid here
specifically *because* a default was given.

**A function that returns a whole row instead of one number:**

```sql
-- SQL function returning a table row
CREATE OR REPLACE FUNCTION get_customer(p_id INT)
RETURNS customers  -- returns full row type
LANGUAGE SQL AS $$
    SELECT * FROM customers WHERE id = p_id;
$$;

SELECT (get_customer(1)).*;  -- expand the row
SELECT (get_customer(1)).name, (get_customer(1)).email;
```

**Why the parentheses around `get_customer(1)` are necessary:**
`RETURNS customers` means the function's result *is* a `customers` row
as a single composite value — not automatically split into columns.
`(get_customer(1)).*` says "take that composite value, and expand it
back into its individual columns," the same way `.name` would access a
field on an object in most programming languages. Without the
parentheses, `get_customer(1).name` would be a parsing error.

**Output of `SELECT (get_customer(1)).*;`:**

| id | name | email | ... |
|---|---|---|---|
| 1 | Alice Johnson | alice@example.com | ... |

---

## PL/pgSQL Functions — Procedural Logic

PL/pgSQL is PostgreSQL's procedural language (like Python in the DB).

**Why you need this:** SQL functions above can only run *one query*.
Real logic often needs more: "first check something, then decide what
to do, then loop over results." PL/pgSQL is what unlocks that — variables,
`IF`, loops, and step-by-step control flow, inside the database.

```sql
-- Basic PL/pgSQL function
CREATE OR REPLACE FUNCTION add_numbers(a INT, b INT)
RETURNS INT
LANGUAGE plpgsql
AS $$
DECLARE
    result INT;
BEGIN
    result := a + b;
    RETURN result;
END;
$$;

SELECT add_numbers(3, 4);  -- 7
```

**The four building blocks every PL/pgSQL function has, in order:**
- **`DECLARE`** — lists every variable you'll use, with its type, *before* you use it. Think of this as "reserving named boxes to put values in" — `result INT` reserves a box named `result` that can hold an integer.
- **`BEGIN ... END;`** — the actual step-by-step instructions, executed top to bottom, one line at a time — this is the procedural part that plain SQL functions don't have.
- **`:=`** — the assignment operator: "put this value into that variable." Different from `=`, which in SQL means *comparison* (as in `WHERE status = 'pending'`) — mixing these up is a common early mistake.
- **`RETURN`** — hands back the final value and immediately exits the function, exactly like `return` in any programming language.

### Variables and Assignment

**Why you need this:** a single function often needs several
intermediate values before producing its final answer — this section
shows how to pull a value *out of a query* and into a variable you can
keep using.

```sql
CREATE OR REPLACE FUNCTION demo_variables()
RETURNS TEXT
LANGUAGE plpgsql AS $$
DECLARE
    customer_count  INT;
    avg_order       NUMERIC(10,2);
    greeting        TEXT;
    today           DATE := CURRENT_DATE;  -- initialize at declaration
BEGIN
    SELECT COUNT(*) INTO customer_count FROM customers;
    SELECT AVG(total) INTO avg_order FROM orders;

    greeting := 'There are ' || customer_count || ' customers.';
    greeting := greeting || ' Avg order: $' || avg_order;
    greeting := greeting || ' Today: ' || today::TEXT;

    RETURN greeting;
END;
$$;

SELECT demo_variables();
```

**The pattern to notice: `SELECT ... INTO variable_name`** — you saw
this exact syntax back in [Lesson 10](10-transactions-and-acid.md)'s
`RETURNING id INTO order_id` example. Here it's the same idea applied to
a plain `SELECT`: run the query, and instead of displaying the result,
store it into the named variable so later lines can use it.

**`today DATE := CURRENT_DATE`** — a variable can be given a starting
value right where it's declared, using the same `:=` assignment
operator, instead of assigning it later inside `BEGIN`.

**Building the string step by step with `||`** (string concatenation,
from [Lesson 02](02-sql-fundamentals.md)) — each line appends more text
onto `greeting`, reassigning it to itself plus something new.

**Output:**
```sql
SELECT demo_variables();
-- "There are 10 customers. Avg order: $452.99 Today: 2026-09-09"
```

### IF / ELSE / ELSIF

**Why you need this:** this is the first genuinely new control-flow
concept in the whole course — branching logic, choosing *which* code
runs based on a condition, rather than filtering *rows* (`WHERE`) or
choosing a *value* per row (`CASE`, [Lesson 03](03-data-types-and-schema.md)).
`IF` inside PL/pgSQL controls the function's own execution path.

```sql
CREATE OR REPLACE FUNCTION categorize_price(p_price NUMERIC)
RETURNS TEXT
LANGUAGE plpgsql AS $$
BEGIN
    IF p_price < 30 THEN
        RETURN 'Budget';
    ELSIF p_price < 100 THEN
        RETURN 'Mid-range';
    ELSIF p_price < 500 THEN
        RETURN 'Premium';
    ELSE
        RETURN 'Luxury';
    END IF;
END;
$$;

SELECT name, price, categorize_price(price) AS tier FROM products;
```

**How Postgres walks through this, for a specific input** —
say `p_price = 75`:
1. Check `p_price < 30` → `75 < 30` is false, skip to the next check
2. Check `p_price < 100` (this is the `ELSIF`) → `75 < 100` is true → run `RETURN 'Mid-range'` and stop — the remaining `ELSIF`/`ELSE` are never even evaluated

**This is conceptually the same idea as `CASE ... WHEN ... THEN`**
([Lesson 03](03-data-types-and-schema.md)) — check conditions in order,
take the first one that matches — just written as procedural
control-flow instead of an expression inside a `SELECT`.

**Sample output:**

| name | price | tier |
|---|---|---|
| USB Cable | 12.99 | Budget |
| Wireless Mouse | 45.00 | Mid-range |
| Office Chair | 249.99 | Premium |
| Laptop Pro | 1299.99 | Luxury |

### Loops

**Why you need this:** so far every function computed one value from
one query. Loops let a function do something **repeatedly** — a fixed
number of times, while some condition holds, or once per row of a
query result. This is the biggest departure from "SQL as a single
declarative statement" you've seen yet.

```sql
-- FOR loop over a range
CREATE OR REPLACE FUNCTION sum_to_n(n INT)
RETURNS INT LANGUAGE plpgsql AS $$
DECLARE
    total INT := 0;
    i     INT;
BEGIN
    FOR i IN 1..n LOOP
        total := total + i;
    END LOOP;
    RETURN total;
END;
$$;

SELECT sum_to_n(100);  -- 5050
```

**Trace it for a small `n`, say `sum_to_n(4)`, step by step:**

| Iteration | `i` | `total := total + i` | `total` after |
|---|---|---|---|
| start | — | — | 0 |
| 1 | 1 | 0 + 1 | 1 |
| 2 | 2 | 1 + 2 | 3 |
| 3 | 3 | 3 + 3 | 6 |
| 4 | 4 | 6 + 4 | 10 |

`FOR i IN 1..n LOOP ... END LOOP` runs the body once for every integer
from `1` to `n`, automatically setting `i` to the current number each
time — you never manually increment `i` yourself, unlike some other
languages' `for` loops.

```sql
-- WHILE loop
CREATE OR REPLACE FUNCTION countdown(start INT)
RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE
    current INT := start;
    result  TEXT := '';
BEGIN
    WHILE current > 0 LOOP
        result := result || current::TEXT || ', ';
        current := current - 1;
    END LOOP;
    RETURN result || 'Go!';
END;
$$;

SELECT countdown(5);  -- '5, 4, 3, 2, 1, Go!'
```

**`WHILE condition LOOP ... END LOOP`** — unlike the `FOR` loop above,
this doesn't know in advance how many times it'll run; it just keeps
repeating **as long as the condition stays true**, checked fresh before
every iteration. Here, that's "as long as `current` is still above 0" —
each pass through the loop appends a number to `result` and decrements
`current`, until `current` finally hits 0 and the condition fails,
ending the loop.

```sql
-- FOR loop over query results
CREATE OR REPLACE FUNCTION log_high_earners(threshold NUMERIC)
RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE
    emp  RECORD;
    msg  TEXT := '';
BEGIN
    FOR emp IN
        SELECT name, salary FROM employees WHERE salary > threshold ORDER BY salary DESC
    LOOP
        msg := msg || emp.name || ': $' || emp.salary || E'\n';
    END LOOP;
    RETURN msg;
END;
$$;

SELECT log_high_earners(90000);
```

**This is the loop shape you'll use most often in practice:** instead of
looping over a fixed range of numbers, it loops **once per row a query
returns**. `emp RECORD` declares a variable that can hold "a whole row,
whatever shape it turns out to be" — inside the loop, `emp.name` and
`emp.salary` access that current row's columns, exactly like `NEW.column`
in a trigger (you'll see that pattern again later in this lesson).

**Output** (assuming Grace and Alice earn over 90000):
```
Grace Chen: $120000
Alice Johnson: $95000
```

---

## Functions That Return Tables

**Why you need this:** every function so far returns one value (a
number, a string, one row). Sometimes you need a function that behaves
like an entire **table** — something you can `SELECT * FROM`, filter,
and join against, not just call for a single answer.

```sql
-- RETURNS TABLE syntax
CREATE OR REPLACE FUNCTION get_customer_orders(p_customer_id INT)
RETURNS TABLE (
    order_id    INT,
    order_date  DATE,
    status      TEXT,
    total       NUMERIC(10,2)
)
LANGUAGE SQL AS $$
    SELECT id, order_date, status, total
    FROM orders
    WHERE customer_id = p_customer_id
    ORDER BY order_date DESC;
$$;

-- Call it like a table
SELECT * FROM get_customer_orders(1);
SELECT status, COUNT(*) FROM get_customer_orders(1) GROUP BY status;
```

**The key difference from earlier functions is right there in the call
site:** you write `FROM get_customer_orders(1)`, not
`SELECT get_customer_orders(1)` — because the function's *result* is a
whole table, so it belongs in the `FROM` clause, exactly like a real
table or a CTE ([Lesson 07](07-subqueries-and-ctes.md)) would. This is
also why the second example can `GROUP BY` on it directly — it's just
another queryable row source.

**Output of `SELECT * FROM get_customer_orders(1);`:**

| order_id | order_date | status | total |
|---|---|---|---|
| 8 | 2024-03-10 | delivered | 150.00 |
| 3 | 2024-02-01 | delivered | 90.00 |
| 1 | 2024-01-15 | cancelled | 200.00 |

```sql
-- RETURNS SETOF — returns multiple rows of a type
CREATE OR REPLACE FUNCTION get_expensive_products(min_price NUMERIC DEFAULT 100)
RETURNS SETOF products  -- returns full product rows
LANGUAGE SQL AS $$
    SELECT * FROM products WHERE price >= min_price ORDER BY price DESC;
$$;

SELECT * FROM get_expensive_products(200);
SELECT name, price FROM get_expensive_products();
```

**`RETURNS SETOF products` vs. `RETURNS TABLE (...)`** — both return
multiple rows, but `SETOF products` says "many rows, each shaped exactly
like the real `products` table" (every column, automatically), whereas
`RETURNS TABLE (...)` from the example above lets you define a
*custom* shape (only the columns you listed, possibly renamed/computed).
Use `SETOF <table>` when you want the full existing row shape; use
`RETURNS TABLE (...)` when you want a specific, different shape.

---

## Error Handling

**Why you need this:** so far, nothing in this lesson's functions has
handled things going wrong — a bad input, a division by zero. This
section covers PL/pgSQL's way of catching and responding to errors
*inside* a function, instead of letting them propagate up as a raw
Postgres error.

```sql
CREATE OR REPLACE FUNCTION safe_divide(a NUMERIC, b NUMERIC)
RETURNS NUMERIC LANGUAGE plpgsql AS $$
BEGIN
    IF b = 0 THEN
        RAISE EXCEPTION 'Division by zero: cannot divide % by 0', a;
    END IF;
    RETURN a / b;
EXCEPTION
    WHEN division_by_zero THEN
        RETURN NULL;  -- or raise a custom message
    WHEN others THEN
        RAISE WARNING 'Unexpected error in safe_divide: %', SQLERRM;
        RETURN NULL;
END;
$$;
```

**Two different error-handling ideas layered on top of each other here
— worth separating:**

1. **`RAISE EXCEPTION`** — deliberately trigger an error yourself, with
   a custom message. `%` inside the message string is a placeholder,
   filled in by the value listed after the comma (`a`) — similar in
   spirit to a parameterized query's `%s` placeholder from
   [Lesson 18](18-connecting-from-python.md), just for building error
   text instead of SQL.
2. **`EXCEPTION WHEN ... THEN ...`** — this is a *catch block*: "if an
   error of this specific kind happens anywhere above, in the `BEGIN`
   section, run this instead of letting the function crash."
   `WHEN division_by_zero` catches specifically a divide-by-zero error;
   `WHEN others` is a catch-all for anything else unexpected.

**Why this function's `IF b = 0` check and its `EXCEPTION WHEN
division_by_zero` clause seem redundant — they actually are, on
purpose, for illustration:** the manual `RAISE EXCEPTION` inside `IF`
would already stop execution before Postgres's own division ever runs —
so in this exact function, the `WHEN division_by_zero` branch would
only ever fire if you removed the `IF` check and let a real `a / b`
divide-by-zero error occur naturally instead. Both approaches are shown
here so you can recognize either style in code you read later.

```sql
-- RAISE levels: DEBUG, LOG, INFO, NOTICE, WARNING, EXCEPTION
RAISE NOTICE 'Processing customer %', customer_id;
RAISE WARNING 'Low stock: product % has only % units', p_id, p_stock;
RAISE EXCEPTION 'Customer % not found', p_id;  -- aborts the function
```

**Why these levels matter — they're not just log formatting.** Only
`EXCEPTION` actually **stops** the function and rolls back whatever it
was doing (per [Lesson 10](10-transactions-and-acid.md)'s transaction
rules). `NOTICE` and `WARNING` are purely informational — they print a
message (visible in `psql` or your application's logs) but execution
continues normally afterward. Choosing the wrong level is a real bug:
using `WARNING` where you meant `EXCEPTION` would let bad data slip
through silently, with just a log line nobody may ever read.

```sql
-- PostgreSQL error codes
WHEN unique_violation THEN ...        -- SQLSTATE 23505
WHEN foreign_key_violation THEN ...   -- SQLSTATE 23503
WHEN check_violation THEN ...         -- SQLSTATE 23514
WHEN not_null_violation THEN ...      -- SQLSTATE 23502
WHEN division_by_zero THEN ...        -- SQLSTATE 22012
WHEN others THEN ...                  -- catch all
```

**Where these names come from:** every constraint type you learned in
[Lesson 03](03-data-types-and-schema.md) — `UNIQUE`, `FOREIGN KEY`,
`CHECK`, `NOT NULL` — has a corresponding named error you can catch
specifically. `unique_violation` is exactly what fires if you try to
insert a duplicate email into a column with a `UNIQUE` constraint, for
example — this is the same violation from that lesson, just now
something your PL/pgSQL code can catch and react to instead of letting
it bubble up as a raw error.

---

## Stored Procedures

Procedures (PostgreSQL 11+) can manage transactions. You CALL them.

**Why you need this:** this is the function-vs-procedure distinction
from the top of this lesson, now put to real use. "Place an order"
touches multiple tables (create the order, check stock, deduct stock,
insert order items) and must either fully succeed or fully roll back —
exactly the transactional guarantee from [Lesson 10](10-transactions-and-acid.md).
A function *cannot* manage transactions; a procedure can — which is
why this operation has to be a procedure, not a function.

```sql
CREATE OR REPLACE PROCEDURE place_order(
    p_customer_id   INT,
    p_product_ids   INT[],
    p_quantities    INT[]
)
LANGUAGE plpgsql AS $$
DECLARE
    v_order_id  INT;
    v_total     NUMERIC(10,2) := 0;
    v_price     NUMERIC(10,2);
    v_stock     INT;
    i           INT;
BEGIN
    -- Verify customer exists
    IF NOT EXISTS (SELECT 1 FROM customers WHERE id = p_customer_id) THEN
        RAISE EXCEPTION 'Customer % does not exist', p_customer_id;
    END IF;

    -- Create the order
    INSERT INTO orders (customer_id, status, total)
    VALUES (p_customer_id, 'pending', 0)
    RETURNING id INTO v_order_id;

    -- Process each product
    FOR i IN 1..array_length(p_product_ids, 1) LOOP
        -- Lock and check stock
        SELECT price, stock INTO v_price, v_stock
        FROM products
        WHERE id = p_product_ids[i]
        FOR UPDATE;

        IF v_stock < p_quantities[i] THEN
            RAISE EXCEPTION 'Insufficient stock for product %: have %, need %',
                p_product_ids[i], v_stock, p_quantities[i];
        END IF;

        -- Deduct stock
        UPDATE products SET stock = stock - p_quantities[i]
        WHERE id = p_product_ids[i];

        -- Add order item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (v_order_id, p_product_ids[i], p_quantities[i], v_price);

        v_total := v_total + v_price * p_quantities[i];
    END LOOP;

    -- Update order total
    UPDATE orders SET total = v_total WHERE id = v_order_id;

    RAISE NOTICE 'Order % created for customer %, total: $%',
        v_order_id, p_customer_id, v_total;
END;
$$;

-- Call the procedure
CALL place_order(1, ARRAY[1, 2], ARRAY[1, 2]);
-- Creates order: 1 Laptop + 2 Wireless Mice
```

**Walking through what this procedure actually does, tying every piece
back to something you already know:**

1. **Guard clause** — `IF NOT EXISTS (...)` checks the customer is real
   before doing anything else, using the `EXISTS` pattern from
   [Lesson 04](04-filtering-deep-dive.md)/[07](07-subqueries-and-ctes.md).
2. **Create the order, capture its new id** — the exact
   `RETURNING id INTO v_order_id` pattern explained earlier in this
   lesson and in [Lesson 10](10-transactions-and-acid.md).
3. **Loop over each product being ordered** — `p_product_ids` and
   `p_quantities` are **arrays** (parallel lists — the product at
   `p_product_ids[i]` corresponds to the quantity at `p_quantities[i]`,
   matched by position). `array_length(p_product_ids, 1)` gets how many
   items are in the array, so the loop runs exactly once per product.
4. **`FOR UPDATE`** — this is new: it locks the selected `products` row
   for the rest of this transaction, preventing another concurrent
   `place_order` call from reading and acting on the *same* stock count
   at the same time. This is the actual fix for the overbooking-style
   race condition described conceptually in
   [Lesson 22](22-real-world-schema-walkthrough.md) — `FOR UPDATE` is
   the concrete mechanism, applied here to inventory instead of class
   capacity.
5. **Stock check, then deduct, then record the line item, then
   accumulate the running total** — four steps per product, all inside
   the loop.
6. **Finally, update the order's total** once the loop finishes.

**Why this entire sequence needs to be one procedure instead of several
separate statements run by your application:** if step 5 fails halfway
through processing the *second* product (say, insufficient stock), you
do not want the *first* product's stock already deducted and its order
item already inserted, with no order total ever set. Wrapping the whole
thing in one procedure means a `RAISE EXCEPTION` partway through aborts
everything that happened since the procedure started — the same
all-or-nothing guarantee as a manual `BEGIN`/`COMMIT`/`ROLLBACK` block,
just packaged as reusable, named logic instead of ad-hoc application code.

---

## Triggers

A trigger automatically calls a function when data changes.

**Why you need this:** every function and procedure so far runs only
when something explicitly *calls* it. A trigger flips this around —
it runs **automatically**, whenever a specific kind of change happens
to a table, with zero effort from whatever application or user made
that change. This is how a database enforces rules that must *never* be
skippable, no matter what code is writing to it.

```
Event: INSERT / UPDATE / DELETE / TRUNCATE
Timing: BEFORE / AFTER / INSTEAD OF
Level: FOR EACH ROW / FOR EACH STATEMENT
```

**Breaking down these three independent choices, since a trigger is
defined by picking one of each:**
- **Event** — *what kind of change* triggers it: an `INSERT`, `UPDATE`, `DELETE`, or a bulk `TRUNCATE`
- **Timing** — *when*, relative to the actual change: `BEFORE` the row is written (so the trigger can still modify or reject it), `AFTER` it's already written (for logging/side-effects), or `INSTEAD OF` (used only on views, replacing the operation entirely)
- **Level** — *how often* it fires: `FOR EACH ROW` (once per affected row) or `FOR EACH STATEMENT` (once total, regardless of how many rows changed)

### Auto-Update updated_at Column

```sql
-- Step 1: Create the trigger function
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at := NOW();  -- NEW = the row being modified
    RETURN NEW;
END;
$$;

-- Step 2: Attach the trigger to tables
CREATE TRIGGER orders_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Now every UPDATE automatically sets updated_at:
UPDATE orders SET status = 'shipped' WHERE id = 1;
-- updated_at is set automatically without your application doing anything
```

**Why this always comes in two separate steps, and what each does:**
step 1 defines *what to do* (a special kind of function, `RETURNS
TRIGGER`, that can only be used as a trigger — never called directly
like the earlier functions in this lesson). Step 2 defines *when to do
it* — attaching that function to a specific table and event. The same
trigger function (`set_updated_at`) gets reused across both `orders`
and `products` — write the logic once, attach it to as many tables as
need it.

**`NEW`** — inside a row-level trigger, `NEW` represents *the row as it
will be written* (its updated values, still pending). Setting
`NEW.updated_at := NOW()` changes what actually gets saved, before it's
saved — only possible because this is a `BEFORE` trigger; an `AFTER`
trigger runs too late to still change the row.

**Trace it concretely:** before this trigger existed, `UPDATE orders
SET status = 'shipped' WHERE id = 1;` would only change `status`,
leaving `updated_at` stale. With the trigger attached, Postgres
silently also runs `NEW.updated_at := NOW()` as part of that same
`UPDATE`, so the row ends up with both `status = 'shipped'` **and**
`updated_at` set to right now — automatically, on *every* future
update, without your application ever having to remember to set it.

### Audit Log Trigger

**Why you need this:** a common real requirement — "keep a permanent
history of every change to `orders`, who changed what and when" — done
here without any application code, by having the database watch itself.

```sql
CREATE TABLE order_audit_log (
    id         SERIAL PRIMARY KEY,
    order_id   INT NOT NULL,
    operation  TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    old_status TEXT,
    new_status TEXT,
    old_total  NUMERIC(10,2),
    new_total  NUMERIC(10,2),
    changed_at TIMESTAMPTZ DEFAULT NOW(),
    changed_by TEXT DEFAULT current_user
);

CREATE OR REPLACE FUNCTION log_order_changes()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO order_audit_log (order_id, operation, new_status, new_total)
        VALUES (NEW.id, 'INSERT', NEW.status, NEW.total);

    ELSIF TG_OP = 'UPDATE' THEN
        IF OLD.status != NEW.status OR OLD.total != NEW.total THEN
            INSERT INTO order_audit_log (order_id, operation, old_status, new_status, old_total, new_total)
            VALUES (NEW.id, 'UPDATE', OLD.status, NEW.status, OLD.total, NEW.total);
        END IF;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO order_audit_log (order_id, operation, old_status, old_total)
        VALUES (OLD.id, 'DELETE', OLD.status, OLD.total);
    END IF;

    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION log_order_changes();

-- Test it:
UPDATE orders SET status = 'shipped' WHERE id = 1;
SELECT * FROM order_audit_log;
```

**`TG_OP`** — a special variable, only available inside a trigger
function, that tells you *which* event actually fired it (`'INSERT'`,
`'UPDATE'`, or `'DELETE'`) — necessary here because one function is
handling all three events at once (see `AFTER INSERT OR UPDATE OR
DELETE` below), so it needs to branch on which one actually happened.

**Why `OLD` appears here but didn't in the previous trigger:** `OLD`
represents the row's values *before* the change — only meaningful for
`UPDATE` (comparing before/after) and `DELETE` (the row being removed
had no "after" state). An `INSERT` has no `OLD` at all — there was no
row before it existed — which is exactly why the `INSERT` branch above
only references `NEW`, never `OLD`.

**Why the `UPDATE` branch has an extra `IF OLD.status != NEW.status OR
OLD.total != NEW.total` check inside it:** without this, *every* update
to an order — even one that changes an unrelated column — would log a
row, cluttering the audit trail with no-op noise. This check means
"only log it if something we actually care about changed."

**`RETURN COALESCE(NEW, OLD);`** — this is a defensive pattern:
`COALESCE` (from [Lesson 04](04-filtering-deep-dive.md)) returns the
first non-NULL argument. On `INSERT`/`UPDATE`, `NEW` exists, so it's
returned; on `DELETE`, `NEW` is `NULL` (nothing was inserted/updated),
so `OLD` is returned instead. Since this is an `AFTER` trigger, the
return value isn't used to change anything (per the rules explained
below) — but a trigger function is still required to return *something*
of the right type, so this pattern reliably provides one regardless of
which event fired.

**Output of `SELECT * FROM order_audit_log;` after the test UPDATE:**

| id | order_id | operation | old_status | new_status | changed_at |
|---|---|---|---|---|---|
| 1 | 1 | UPDATE | pending | shipped | 2026-09-09 10:15:00 |

### Validation Trigger

**Why you need this:** a trigger doesn't only have to *record* changes
— a `BEFORE` trigger can **reject** a change entirely, enforcing a
business rule no `CHECK` constraint could express (a `CHECK` can only
see the row being written, not compare it against its *previous*
state — which is exactly what "valid status transitions" needs).

```sql
-- Prevent an order from going backwards in status
CREATE OR REPLACE FUNCTION validate_status_transition()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
    allowed_next TEXT[];
BEGIN
    -- Define valid transitions
    allowed_next := CASE OLD.status
        WHEN 'pending'   THEN ARRAY['shipped', 'cancelled']
        WHEN 'shipped'   THEN ARRAY['delivered', 'cancelled']
        WHEN 'delivered' THEN ARRAY[]::TEXT[]  -- terminal
        WHEN 'cancelled' THEN ARRAY[]::TEXT[]  -- terminal
        ELSE ARRAY[]::TEXT[]
    END;

    IF NOT (NEW.status = ANY(allowed_next)) THEN
        RAISE EXCEPTION 'Invalid status transition: % → % (allowed: %)',
            OLD.status, NEW.status, allowed_next;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER orders_status_guard
BEFORE UPDATE OF status ON orders
FOR EACH ROW EXECUTE FUNCTION validate_status_transition();

-- Test valid:
UPDATE orders SET status = 'shipped' WHERE id = 5 AND status = 'pending';
-- Test invalid:
UPDATE orders SET status = 'pending' WHERE id = 1 AND status = 'delivered';
-- ERROR: Invalid status transition: delivered → pending
```

**Walking through the logic for the invalid test case, concretely:**
1. `OLD.status` is `'delivered'` (the order's current status before this update attempt)
2. `allowed_next := CASE OLD.status WHEN 'delivered' THEN ARRAY[]::TEXT[] ...` → `allowed_next` becomes an **empty array** — `'delivered'` is a dead end, nothing can follow it
3. `NEW.status = ANY(allowed_next)` — `= ANY(...)` (from [Lesson 12](12-advanced-queries.md)) checks whether `NEW.status` (`'pending'`) matches *any* value in the array — since the array is empty, this is always false
4. `IF NOT (false)` → `IF true` → the `RAISE EXCEPTION` fires, and the `UPDATE` is aborted entirely — the row is never actually changed

**Why `BEFORE UPDATE OF status`, specifically, instead of just `BEFORE
UPDATE`:** `OF status` means this trigger only fires when the `status`
column itself is part of the update — an update that only touches
`total`, say, never invokes this check at all, since there's no status
transition to validate.

### Trigger Variables Available in PL/pgSQL

```sql
-- In a row-level trigger:
NEW     -- the new row (INSERT, UPDATE) — modify it in BEFORE triggers
OLD     -- the old row (UPDATE, DELETE)
TG_OP   -- operation: 'INSERT', 'UPDATE', 'DELETE', 'TRUNCATE'
TG_TABLE_NAME  -- name of the table that fired the trigger
TG_WHEN        -- 'BEFORE', 'AFTER', 'INSTEAD OF'
TG_LEVEL       -- 'ROW', 'STATEMENT'

-- Returning values from row-level triggers:
-- BEFORE trigger: return NEW to proceed, return NULL to cancel the operation
-- AFTER trigger: return value is ignored, return NULL or NEW
-- INSTEAD OF trigger: return NEW to indicate success, NULL to skip
```

**Why the return value's meaning changes depending on trigger timing —
this is the single most important rule on this page to internalize:**
a `BEFORE` trigger runs *before* Postgres commits to writing the row, so
its return value still has power — return `NEW` (possibly modified, as
in the `updated_at` example) to let the write proceed with those values,
or return `NULL` to silently cancel the write entirely, as if it never
happened. An `AFTER` trigger runs *after* the row is already written —
by then it's too late to change or cancel anything, so its return value
is ignored; it exists purely to trigger side effects like the audit log
above.

---

## Managing Functions

```sql
-- List all functions
\df               -- psql meta-command
\df+ function_name -- with detail

-- Drop a function
DROP FUNCTION get_customer_count();
DROP FUNCTION IF EXISTS get_orders_by_status(TEXT);
-- Must include parameter types when overloaded functions exist

-- List triggers on a table
SELECT trigger_name, event_manipulation, action_timing
FROM information_schema.triggers
WHERE event_object_table = 'orders';

-- Drop a trigger
DROP TRIGGER orders_audit ON orders;
DROP TRIGGER IF EXISTS orders_updated_at ON orders;
```

**Why `DROP FUNCTION` sometimes needs parameter types specified:**
Postgres allows **overloading** — multiple functions sharing the same
name but different parameter types (e.g., two versions of
`get_orders_by_status`, one taking `TEXT`, another taking `INT`). If
more than one function shares a name, `DROP FUNCTION name()` alone is
ambiguous — Postgres needs the parameter list to know exactly which one
you mean, the same way it's ambiguous to say "delete the file named
report" if there are two files named that in different folders.

---

## Exercises

**Exercise 1:** Create a function `customer_tier(customer_id INT)` that returns
'VIP', 'Regular', or 'New' based on total spending from the orders table.

**Exercise 2:** Create a trigger that automatically inserts a welcome email
into a `notifications` table whenever a new customer is inserted.
(Create the notifications table too.)

**Exercise 3:** Write a PL/pgSQL function `transfer_stock(from_product INT, to_product INT, qty INT)`
that moves `qty` units from one product to another. Include validation that:
- Both products exist
- from_product has enough stock
- qty is positive

**Exercise 4:** Create an audit trigger on the `products` table that records
every price change (old price, new price, changed at).

---

## Key Takeaways

1. SQL functions are simpler (just a SELECT); PL/pgSQL functions support full procedural logic
2. `RETURNS TABLE (...)` lets functions return multiple rows with named columns
3. Procedures (PostgreSQL 11+) can manage transactions; functions cannot
4. Triggers fire automatically on data changes — great for audit logs, validation, defaults
5. In BEFORE row triggers: modify `NEW` to change what gets written; return NULL to abort
6. `RAISE EXCEPTION` with `EXCEPTION WHEN` blocks for structured error handling
7. Use triggers sparingly — they make debugging harder and can cause performance issues, precisely because the logic runs invisibly, with no line of application code showing it happened

---

## Next Lesson
[Lesson 15 — Query Optimization](15-query-optimization.md)
