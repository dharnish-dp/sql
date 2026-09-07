# Lesson 10 — Transactions & ACID

## Goal
Understand how PostgreSQL guarantees data integrity in the face of concurrent
users and system failures. This is the foundation of reliable software.

## Prerequisites
- [Lesson 09](09-indexes-and-performance.md) — indexes

## After This Lesson You Will Be Able To
- Explain all four ACID properties with real examples
- Use BEGIN, COMMIT, ROLLBACK to control transactions
- Understand the four isolation levels and their trade-offs
- Recognize and avoid deadlocks
- Use SAVEPOINT for partial rollbacks
- Use SELECT FOR UPDATE / FOR SHARE for pessimistic locking

---

## What Is a Transaction?

A transaction is a group of SQL statements that are treated as a single unit.
Either ALL succeed, or NONE of them take effect.

**Classic example: Bank transfer**

```sql
-- Transfer $500 from account 1 to account 2

-- WITHOUT transactions (DANGEROUS):
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
-- ← CRASH HERE: $500 has left account 1 but never arrived in account 2
UPDATE accounts SET balance = balance + 500 WHERE id = 2;

-- WITH transactions (SAFE):
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;
-- If anything fails between BEGIN and COMMIT, both changes are undone
```

---

## ACID — The Four Guarantees

### A — Atomicity
**All or nothing.** Every statement in a transaction succeeds together, or
none of them take effect.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
UPDATE accounts SET balance = balance + 500 WHERE id = 2;
-- If the second UPDATE fails (e.g., account 2 doesn't exist),
-- the first UPDATE is also rolled back automatically
COMMIT;
```

### C — Consistency
**The database goes from one valid state to another.** Constraints (CHECK,
FOREIGN KEY, UNIQUE) are enforced at transaction boundaries.

```sql
BEGIN;
UPDATE accounts SET balance = -100 WHERE id = 1;  -- violates CHECK (balance >= 0)
COMMIT;
-- ERROR: new row for relation "accounts" violates check constraint
-- The transaction is aborted. Balance stays unchanged.
```

### I — Isolation
**Concurrent transactions don't interfere.** While a transaction is in progress,
changes are not visible to other transactions (at certain levels).

```sql
-- Session 1                     | Session 2
BEGIN;                           |
UPDATE orders SET status =       |
  'shipped' WHERE id = 1;       | SELECT status FROM orders WHERE id = 1;
                                 | -- Returns 'pending' (not yet seeing Session 1's change)
COMMIT;                          |
                                 | SELECT status FROM orders WHERE id = 1;
                                 | -- Now returns 'shipped'
```

### D — Durability
**Committed transactions survive failures.** Once COMMIT succeeds, the data
is written to the WAL (Write-Ahead Log) and will survive a power outage or crash.

```
COMMIT; → PostgreSQL writes to WAL → fsync to disk → returns "success"
→ Server crashes
→ Restart: PostgreSQL replays WAL → committed data is restored
```

---

## BEGIN, COMMIT, ROLLBACK

```sql
-- Start a transaction
BEGIN;
-- or: START TRANSACTION;

-- Make changes
INSERT INTO orders (customer_id, total) VALUES (1, 299.99);
UPDATE products SET stock = stock - 1 WHERE id = 3;

-- If everything is OK: commit (make permanent)
COMMIT;

-- If something went wrong: rollback (undo everything since BEGIN)
ROLLBACK;

-- PostgreSQL also has:
BEGIN;
UPDATE orders SET status = 'shipped' WHERE id = 1;
-- Oops, wrong order
ROLLBACK;
-- Now we're back to before the BEGIN
```

### Why ROLLBACK Has to Be a Separate, Explicit Command

**A common assumption:** "if a query fails between `BEGIN` and `COMMIT`,
surely everything just reverts automatically — why do I need to type
`ROLLBACK` myself?" Postgres's actual behavior is more subtle than that.

**What really happens when a statement fails mid-transaction:** Postgres
does **not** silently undo anything. Instead, it marks the *entire
transaction* as broken, but leaves it **open** — every further command
you type gets rejected, until you explicitly issue `ROLLBACK` yourself.

```sql
BEGIN;
INSERT INTO orders (customer_id, total) VALUES (1, 299.99);  -- succeeds
INSERT INTO orders (customer_id, total) VALUES (999, 'bad');  -- FAILS

-- Try anything else now:
SELECT * FROM orders;
```
```
ERROR:  current transaction is aborted, commands ignored until end of transaction block
```

The transaction is now stuck in this broken state — nothing works —
**until you run `ROLLBACK;`**, which finally clears it and undoes
everything since `BEGIN`.

**Why Postgres is designed this way instead of auto-rolling-back:** if
it silently reverted the instant any statement failed, your application
would never get a chance to *decide* what to do — log the error, alert
someone, inspect what went wrong. Leaving the transaction stuck-but-open
forces something (you, or your app's error handling) to make an
explicit decision.

**A second, unrelated reason `ROLLBACK` needs to exist as its own
command:** it's not only for error recovery. Every statement can succeed
perfectly fine, and you can *still* decide to abandon the whole
transaction on purpose:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- succeeds, no error
-- App logic checks: "this would make balance negative — abort this transaction"
ROLLBACK;
```
Nothing failed here — `ROLLBACK` is the deliberate "changed my mind, undo
everything since BEGIN" command, independent of whether an error ever
occurred.

### Autocommit Mode

By default in PostgreSQL, every single statement runs in its own implicit transaction:

```sql
-- These are AUTOMATICALLY committed:
INSERT INTO customers (name) VALUES ('Test');
-- ← automatically committed, even without BEGIN/COMMIT

-- To make them a single atomic unit, you must explicitly use BEGIN:
BEGIN;
INSERT INTO customers (name) VALUES ('Test 1');
INSERT INTO customers (name) VALUES ('Test 2');
COMMIT;
-- Both succeed or both fail
```

---

## SAVEPOINT — Partial Rollback

SAVEPOINT lets you mark a point inside a transaction to roll back to,
without undoing the entire transaction.

```sql
BEGIN;

INSERT INTO orders (customer_id, total) VALUES (1, 100);
-- Order created successfully

SAVEPOINT before_items;  -- mark a save point

INSERT INTO order_items (order_id, product_id, quantity) VALUES (99, 5, 2);
-- ERROR: order_id 99 doesn't exist!

ROLLBACK TO SAVEPOINT before_items;
-- Undo only the failed INSERT, keep the order we created

-- Try again with correct order_id
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 5, 2);

COMMIT;
-- Final result: order created AND items added (after retry)

-- Release a savepoint (frees resources — optional)
RELEASE SAVEPOINT before_items;
```

---

## Transaction Isolation Levels

The "I" in ACID doesn't mean total isolation — it's configurable.
More isolation = more safety, but less concurrency (more locking/blocking).

### The Four Anomalies

**Why you need this:** these four are ordered from mildest to most
severe — each is a specific way that running two transactions *at the
same time* can produce a result that wouldn't happen if they ran one
after another. The isolation levels table right after this exists
purely to say "this level blocks these anomalies, but not those" — so
understanding each anomaly concretely is what makes that table make
sense.

| Anomaly | Description |
|---------|-------------|
| **Dirty Read** | Read uncommitted changes from another transaction (that might roll back) |
| **Non-repeatable Read** | Same query returns different values within the same transaction |
| **Phantom Read** | Same WHERE returns different rows within the same transaction |
| **Serialization Anomaly** | Result inconsistent with any serial execution of transactions |

#### 1. Dirty Read — reading data that was never actually saved

You read a value another transaction changed but **hasn't committed
yet** — then that transaction rolls back, meaning the value you read
never really existed.

**IMPORTANT — this is a HYPOTHETICAL, not what Postgres actually does.**
The scenario below is what a *weaker* database engine (one that allows
dirty reads) would do. Real Postgres, even at its weakest isolation
level, blocks this — Transaction B always sees the OLD, committed value
until A actually commits. The example exists only to show you what the
anomaly *would* look like if it weren't prevented.

```sql
-- Transaction A:
BEGIN;
UPDATE accounts SET balance = 1000 WHERE id = 1;
-- not committed yet

-- Transaction B — HYPOTHETICAL ONLY, assuming a database that allows dirty reads:
SELECT balance FROM accounts WHERE id = 1;   -- would see 1000 — a value that isn't final!

-- Transaction A:
ROLLBACK;   -- balance goes back to whatever it was before — 1000 never really happened
```

**What Transaction B actually sees in real Postgres, right now, in this
exact scenario:**
```sql
-- Transaction B, for real, in Postgres:
SELECT balance FROM accounts WHERE id = 1;   -- sees the OLD value — NOT 1000
-- Postgres never lets you see another transaction's uncommitted changes,
-- at any isolation level. The dirty read above cannot actually happen here.
```
Transaction B acted on a number that turned out to be fiction — the
worst anomaly of the four. Postgres never allows this at all, even at
its weakest isolation level.

#### 2. Non-repeatable Read — an existing row you already read gets changed underneath you

```sql
-- Transaction A:
BEGIN;
SELECT total FROM orders WHERE id = 1;   -- reads 100.00

-- (Transaction B commits: UPDATE orders SET total = 150.00 WHERE id = 1;)

SELECT total FROM orders WHERE id = 1;   -- reads 150.00 — same row, changed value!
COMMIT;
```
Inside **one single transaction**, the *same exact query* gave two
different answers for the *same row* — because someone else's committed
change slipped in between the two reads.

#### 3. Phantom Read — a brand-new row appears (or disappears) that matches your filter

```sql
-- Transaction A:
BEGIN;
SELECT COUNT(*) FROM orders WHERE status = 'pending';   -- returns 5

-- (Transaction B commits: INSERT INTO orders (..., status) VALUES (..., 'pending');)

SELECT COUNT(*) FROM orders WHERE status = 'pending';   -- returns 6 — a new row showed up!
COMMIT;
```
Nothing already seen *changed* — a row that didn't exist yet during the
first read now satisfies the `WHERE` clause on the second read. The
distinction from #2: non-repeatable read = an existing row's *value*
changed; phantom read = the *set of matching rows itself* changed.

#### 4. Serialization Anomaly — the combined result, not any single read, is the problem

This one isn't about a single row or query going stale — it's about the
**final combined result** of two transactions being something that
**could never happen** if they'd run one at a time, in either order.

```sql
-- Rule: "total inventory across two warehouses must always be >= 0"
-- Warehouse A has 10 units, Warehouse B has 10 units. Total = 20.

-- Transaction A:                      Transaction B:
BEGIN;                                 BEGIN;
SELECT SUM(stock) FROM warehouses;     SELECT SUM(stock) FROM warehouses;
-- sees total = 20, safe to remove 15  -- sees total = 20, safe to remove 15
UPDATE warehouse_a SET stock = -5;     UPDATE warehouse_b SET stock = -5;
COMMIT;                                COMMIT;
```
**Each transaction individually looked completely fine** — both checked
the total, both saw 20, both concluded "removing 15 leaves 5, that's
safe." But run **together**, the *combined* effect removes 30 units
from a pool of 20 — impossible if either transaction had run to
completion before the other started. Neither transaction did anything
wrong individually; the *combination* is what's inconsistent with any
possible one-at-a-time ordering.

**Why this needs its own category:** anomalies #1–#3 are all about one
transaction seeing *stale or shifting data*. A serialization anomaly can
happen even when every individual read was perfectly accurate the whole
time — the problem is purely the *combined outcome* of both
transactions' writes, which is why only the strictest level,
`SERIALIZABLE`, prevents it.

### PostgreSQL Isolation Levels

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Notes |
|-------|-----------|---------------------|-------------|-------|
| `READ UNCOMMITTED` | Possible (but PostgreSQL prevents it) | Yes | Yes | |
| `READ COMMITTED` | No | Yes | Yes | **Default in PostgreSQL** |
| `REPEATABLE READ` | No | No | No in PG (better than standard) | |
| `SERIALIZABLE` | No | No | No | Full serialization guarantee |

```sql
-- Set isolation level for the current transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- default
-- or
BEGIN ISOLATION LEVEL READ COMMITTED;

BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Set default for the session
SET default_transaction_isolation = 'repeatable read';
```

### READ COMMITTED (Default) — Practical Behavior

```sql
-- Session 1                          | Session 2
BEGIN;                                |
SELECT total FROM orders WHERE id=1;  | -- returns 100
                                      | BEGIN;
                                      | UPDATE orders SET total=200 WHERE id=1;
                                      | COMMIT;
SELECT total FROM orders WHERE id=1;  | -- returns 200 (sees the committed change!)
COMMIT;                               |
-- Non-repeatable read: same query returned different values in same transaction
-- This is expected at READ COMMITTED level
```

### REPEATABLE READ — Snapshot Isolation

```sql
-- Session 1                          | Session 2
BEGIN ISOLATION LEVEL REPEATABLE READ;|
SELECT total FROM orders WHERE id=1;  | -- returns 100
                                      | BEGIN;
                                      | UPDATE orders SET total=200 WHERE id=1;
                                      | COMMIT;
SELECT total FROM orders WHERE id=1;  | -- still returns 100 (snapshot from BEGIN)
COMMIT;                               |
-- Session 1 always sees the snapshot taken at the start of its transaction
```

### SERIALIZABLE — Full Safety

```sql
-- Session 1                          | Session 2
BEGIN ISOLATION LEVEL SERIALIZABLE;  |
SELECT SUM(balance) FROM accounts;   | BEGIN ISOLATION LEVEL SERIALIZABLE;
-- Returns 1000                       |
                                      | INSERT INTO accounts(balance) VALUES (500);
                                      | COMMIT;
INSERT INTO accounts(balance)        |
  VALUES (SELECT SUM(balance)        |
          FROM accounts);            |
-- This would create an inconsistency |
COMMIT;                               |
-- One of the transactions gets:      |
-- ERROR: could not serialize access  |
-- You must retry the transaction     |
```

---

## Locking

### Row-Level Locks

```sql
-- SELECT FOR UPDATE: lock rows you intend to update
-- Other transactions trying to update/delete the same rows will WAIT
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE;
-- ← Now locked. Other transactions trying SELECT FOR UPDATE on id=1 will block.
UPDATE orders SET status = 'shipped' WHERE id = 1;
COMMIT;

-- SELECT FOR SHARE: allow other readers, block writers
BEGIN;
SELECT * FROM customers WHERE id = 1 FOR SHARE;
-- Other transactions can read (FOR SHARE), but not update/delete
COMMIT;

-- SKIP LOCKED: skip rows locked by other transactions (don't wait)
BEGIN;
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE SKIP LOCKED LIMIT 10;
-- Get up to 10 pending orders that aren't locked by another process
-- Perfect for job queues: multiple workers can process orders concurrently
UPDATE orders SET status = 'processing' WHERE id = ANY(ARRAY[...]);
COMMIT;

-- NOWAIT: fail immediately instead of waiting
BEGIN;
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;
-- If locked by another transaction: ERROR immediately (don't wait)
```

### Table-Level Locks

```sql
-- Most operations use appropriate locks automatically
-- Manual table locks when needed:
LOCK TABLE orders IN EXCLUSIVE MODE;
LOCK TABLE orders IN SHARE MODE;
LOCK TABLE orders IN ACCESS EXCLUSIVE MODE;  -- strongest, blocks everything

-- Access modes (weakest to strongest):
ACCESS SHARE       -- SELECT
ROW SHARE          -- SELECT FOR UPDATE/SHARE
ROW EXCLUSIVE      -- INSERT, UPDATE, DELETE
SHARE UPDATE EXCLUSIVE -- VACUUM, CREATE INDEX CONCURRENTLY
SHARE              -- CREATE INDEX
SHARE ROW EXCLUSIVE -- rare
EXCLUSIVE          -- blocks everything except ACCESS SHARE
ACCESS EXCLUSIVE   -- blocks everything (ALTER TABLE, DROP TABLE)
```

---

## Deadlocks

A deadlock occurs when two transactions each hold a lock the other needs.

```
Transaction 1: Locks A, waiting for B
Transaction 2: Locks B, waiting for A
← Neither can proceed — deadlock!
```

```sql
-- Session 1                              | Session 2
BEGIN;                                   | BEGIN;
UPDATE accounts SET balance=balance-100  |
  WHERE id=1;  -- Locks row 1            |
                                         | UPDATE accounts SET balance=balance+50
                                         |   WHERE id=2;  -- Locks row 2
UPDATE accounts SET balance=balance+100  |
  WHERE id=2;  -- Waiting for row 2!     |
                                         | UPDATE accounts SET balance=balance-50
                                         |   WHERE id=1;  -- Waiting for row 1!
-- DEADLOCK! PostgreSQL detects it and   |
-- aborts ONE of the transactions:       |
-- ERROR: deadlock detected              |
```

### Preventing Deadlocks

```sql
-- Rule: always lock resources in the SAME ORDER across all transactions

-- GOOD: both transactions lock id=1 first, then id=2
-- Session 1: Lock(1), Lock(2) — Session 2 waits for Lock(1)
-- Session 1: commits, Session 2 gets Lock(1), then Lock(2)

-- Bad: one transaction locks 1 then 2, another locks 2 then 1

-- For multi-row updates, sort the IDs first:
-- Python:
ids = sorted([id1, id2])  -- always [1, 2], never [2, 1]
-- SQL with ARRAY:
SELECT * FROM accounts WHERE id = ANY(ARRAY[2, 1]) ORDER BY id FOR UPDATE;
-- ORDER BY id ensures consistent lock order
```

---

## Real-World Transaction Patterns

### Pattern 1 — E-commerce Order Placement

```sql
BEGIN;

-- 1. Lock the product to check and update stock atomically
SELECT stock FROM products WHERE id = 3 FOR UPDATE;
-- Check in application: if stock < quantity, ROLLBACK

-- 2. Decrease stock
UPDATE products SET stock = stock - 2 WHERE id = 3;

-- 3. Create the order
INSERT INTO orders (customer_id, order_date, status, total)
VALUES (1, CURRENT_DATE, 'pending', 99.98)
RETURNING id INTO order_id;

-- 4. Create order items
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (order_id, 3, 2, 49.99);

COMMIT;
-- All or nothing: stock decreases, order created, items inserted
```

### Pattern 2 — Idempotent Operations (Upsert)

```sql
-- INSERT or UPDATE if row already exists
INSERT INTO customers (email, name, city)
VALUES ('alice@example.com', 'Alice Updated', 'Boston')
ON CONFLICT (email) DO UPDATE SET
    name = EXCLUDED.name,
    city = EXCLUDED.city;
-- EXCLUDED refers to the row that would have been inserted

-- Insert only if not exists (ignore duplicates)
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
VALUES (1, 2, 1, 29.99)
ON CONFLICT (order_id, product_id) DO NOTHING;

-- Upsert with condition
INSERT INTO products (id, name, price, updated_at)
VALUES (5, 'New Name', 399.99, NOW())
ON CONFLICT (id) DO UPDATE SET
    name = EXCLUDED.name,
    price = EXCLUDED.price,
    updated_at = EXCLUDED.updated_at
WHERE products.updated_at < EXCLUDED.updated_at;  -- only update if newer
```

### Pattern 3 — Job Queue (SKIP LOCKED)

```sql
-- Worker process claims a job atomically
BEGIN;

SELECT * FROM jobs
WHERE status = 'queued'
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- ← Gets one job that no other worker has locked
-- Multiple workers can run this simultaneously without conflict

UPDATE jobs SET status = 'processing', started_at = NOW()
WHERE id = (the id from above);

COMMIT;
```

---

## Monitoring Locks and Transactions

```sql
-- See all current locks
SELECT
    pid,
    query,
    wait_event_type,
    wait_event,
    state,
    query_start
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- See blocked queries
SELECT
    blocked.pid     AS blocked_pid,
    blocked.query   AS blocked_query,
    blocking.pid    AS blocking_pid,
    blocking.query  AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Kill a blocking query
SELECT pg_terminate_backend(pid);  -- or pg_cancel_backend(pid) for softer kill
```

---

## Exercises

**Exercise 1:** Write a transaction that:
- Decreases the stock of product_id=1 by 1
- Creates a new order for customer_id=1 with total=1299.99
- Creates an order_item for that order
- If any step fails, all changes must be rolled back

**Exercise 2:** Use SAVEPOINT to write a transaction that tries to insert
an order item with invalid data, catches the failure, rolls back to the
savepoint, and inserts with correct data.

**Exercise 3:** What isolation level prevents non-repeatable reads?
Write a scenario demonstrating the problem at READ COMMITTED and the
solution at REPEATABLE READ.

**Exercise 4:** What's the difference between:
```sql
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE SKIP LOCKED;
```
When would you use each?

---

## Key Takeaways

1. ACID: Atomicity (all or nothing), Consistency (constraints), Isolation (concurrency), Durability (crash-safe)
2. `BEGIN; ... COMMIT;` groups statements into one atomic unit
3. `ROLLBACK` undoes everything since `BEGIN`
4. `SAVEPOINT` lets you partially rollback within a transaction
5. Default isolation: READ COMMITTED — queries see committed changes from other transactions
6. `REPEATABLE READ`: snapshot from transaction start — no surprises mid-transaction
7. `SERIALIZABLE`: total isolation — safest but slowest
8. `FOR UPDATE`: lock rows you intend to modify — prevents race conditions
9. `FOR UPDATE SKIP LOCKED`: perfect for concurrent job queues
10. Deadlocks: prevent by always locking rows in the same order

---

## Next Lesson
[Lesson 11 — Database Design & Normalization](11-database-design-normalization.md)
