# SQL Cheat Sheet — Quick Reference

---

## Core Query Structure

```sql
SELECT   columns / expressions / aggregates
FROM     table
JOIN     other_table ON condition
WHERE    row_filter
GROUP BY grouping_columns
HAVING   group_filter
ORDER BY sort_columns [ASC|DESC]
LIMIT    n
OFFSET   n;
```

**Evaluation order:** FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT

---

## SELECT Operators

```sql
*                          -- all columns
col AS alias               -- column alias
expr AS alias              -- computed column
DISTINCT col               -- remove duplicates
```

---

## Comparison Operators

```sql
=  !=  <>  <  <=  >  >=           -- standard comparisons
IS NULL / IS NOT NULL              -- NULL checks (never use = NULL)
IS DISTINCT FROM / IS NOT DISTINCT FROM  -- NULL-safe equality
BETWEEN x AND y                    -- inclusive range
IN (a, b, c)                      -- match any in list
NOT IN (a, b, c)                  -- not in list (careful with NULLs!)
LIKE 'pat%'                       -- case-sensitive pattern (% = any, _ = one char)
ILIKE 'pat%'                      -- case-insensitive pattern (PostgreSQL)
~  ~*  !~  !~*                    -- POSIX regex (case-sensitive/insensitive)
```

---

## JOINs

```sql
INNER JOIN  t ON condition    -- only matching rows
LEFT JOIN   t ON condition    -- all left + matching right (NULLs for no match)
RIGHT JOIN  t ON condition    -- all right + matching left
FULL OUTER JOIN t ON condition -- all rows from both (NULLs where no match)
CROSS JOIN  t                 -- every row × every row
```

---

## Aggregate Functions

```sql
COUNT(*)              -- count all rows
COUNT(col)            -- count non-NULL values
COUNT(DISTINCT col)   -- count unique non-NULL values
SUM(col)              -- sum (ignores NULLs)
AVG(col)              -- average (ignores NULLs)
MIN(col)              -- minimum
MAX(col)              -- maximum
STRING_AGG(col, sep)  -- concatenate strings
ARRAY_AGG(col)        -- collect into array

-- Conditional aggregation
COUNT(*) FILTER (WHERE condition)
SUM(col) FILTER (WHERE condition)
-- Or with CASE:
COUNT(CASE WHEN cond THEN 1 END)
```

---

## Window Functions

```sql
function() OVER (
    [PARTITION BY cols]
    [ORDER BY cols]
    [ROWS BETWEEN start AND end]
)

-- Numbering
ROW_NUMBER()          -- unique, no gaps, no ties
RANK()                -- gaps after ties
DENSE_RANK()          -- no gaps after ties

-- Lag/Lead
LAG(col, n, default)  -- n rows before current
LEAD(col, n, default) -- n rows after current

-- First/Last
FIRST_VALUE(col)      -- first row in window
LAST_VALUE(col)       -- last row (needs explicit frame)

-- Running aggregates (add ORDER BY to any aggregate)
SUM(col)   OVER (ORDER BY date)  -- cumulative sum
AVG(col)   OVER (ORDER BY date)  -- running average
COUNT(*)   OVER (ORDER BY date)  -- running count

-- Percentile
NTILE(n)              -- divide into n buckets
PERCENT_RANK()        -- 0.0 to 1.0 relative rank
CUME_DIST()           -- cumulative distribution
```

---

## CTEs

```sql
WITH name AS (
    SELECT ...
),
name2 AS (
    SELECT ... FROM name
)
SELECT * FROM name2;

-- Recursive
WITH RECURSIVE cte AS (
    SELECT ...          -- base case
    UNION ALL
    SELECT ... FROM cte  -- recursive case
)
SELECT * FROM cte;
```

---

## Set Operations

```sql
SELECT ... UNION     SELECT ...  -- combine, remove duplicates
SELECT ... UNION ALL SELECT ...  -- combine, keep duplicates (faster)
SELECT ... INTERSECT SELECT ...  -- common rows
SELECT ... EXCEPT    SELECT ...  -- rows in first, not in second
```

---

## DDL

```sql
-- Create
CREATE TABLE t (col type [constraint], ..., [table_constraint]);
CREATE INDEX idx ON t(col);
CREATE INDEX CONCURRENTLY idx ON t(col);  -- non-blocking
CREATE UNIQUE INDEX idx ON t(col);
CREATE INDEX idx ON t(col) WHERE condition;  -- partial
CREATE VIEW v AS SELECT ...;
CREATE MATERIALIZED VIEW mv AS SELECT ...;

-- Modify
ALTER TABLE t ADD COLUMN col type;
ALTER TABLE t DROP COLUMN col;
ALTER TABLE t RENAME COLUMN old TO new;
ALTER TABLE t ALTER COLUMN col TYPE new_type;
ALTER TABLE t ALTER COLUMN col SET DEFAULT value;
ALTER TABLE t ALTER COLUMN col SET NOT NULL;
ALTER TABLE t ADD CONSTRAINT name type ...;
ALTER TABLE t DROP CONSTRAINT name;

-- Drop
DROP TABLE t;
DROP TABLE IF EXISTS t CASCADE;
DROP INDEX idx;
DROP VIEW v;
TRUNCATE t;           -- fast delete all rows (unlogged)
TRUNCATE t CASCADE;   -- also truncate referencing tables
```

---

## DML

```sql
-- Insert
INSERT INTO t (cols) VALUES (vals);
INSERT INTO t (cols) VALUES (v1), (v2), (v3);  -- multi-row
INSERT INTO t SELECT ... FROM ...;              -- insert from query

-- Upsert (PostgreSQL)
INSERT INTO t (cols) VALUES (...)
ON CONFLICT (key_col) DO UPDATE SET col = EXCLUDED.col;
ON CONFLICT DO NOTHING;

-- Update
UPDATE t SET col = val WHERE condition;
UPDATE t SET col1 = v1, col2 = v2 WHERE id = 1;

-- Delete
DELETE FROM t WHERE condition;
DELETE FROM t;  -- delete all rows (returns row count)

-- Returning
INSERT INTO t (col) VALUES ('x') RETURNING id;
UPDATE t SET col = 'x' WHERE id=1 RETURNING *;
DELETE FROM t WHERE id=1 RETURNING id;
```

---

## Transactions

```sql
BEGIN;
SAVEPOINT name;
ROLLBACK TO SAVEPOINT name;
RELEASE SAVEPOINT name;
COMMIT;
ROLLBACK;

-- Isolation levels
BEGIN ISOLATION LEVEL READ COMMITTED;    -- default
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;

-- Locking
SELECT ... FOR UPDATE;              -- exclusive row lock
SELECT ... FOR UPDATE SKIP LOCKED;  -- skip locked rows (job queues)
SELECT ... FOR SHARE;               -- shared row lock
SELECT ... FOR UPDATE NOWAIT;       -- fail immediately if locked
```

---

## Useful Functions

```sql
-- String
LENGTH(s)  UPPER(s)  LOWER(s)  INITCAP(s)
TRIM(s)  LTRIM(s)  RTRIM(s)
SUBSTRING(s, start, len)  LEFT(s,n)  RIGHT(s,n)
REPLACE(s, from, to)  REGEXP_REPLACE(s, pat, rep, flags)
SPLIT_PART(s, delim, n)  STRING_TO_ARRAY(s, delim)
LPAD(s, n, fill)  RPAD(s, n, fill)
POSITION(sub IN s)  STRPOS(s, sub)
CONCAT(a, b, ...)  CONCAT_WS(sep, a, b, ...)
FORMAT('Hello %s', name)

-- Number
ROUND(n, d)  CEIL(n)  FLOOR(n)  TRUNC(n, d)
ABS(n)  MOD(a, b)  SQRT(n)  POWER(n, exp)
RANDOM()  (0.0 – 1.0)

-- Date
NOW()  CURRENT_DATE  CURRENT_TIMESTAMP
EXTRACT(part FROM d)
DATE_TRUNC('month', d)  DATE_TRUNC('year', d)
TO_CHAR(d, fmt)  TO_DATE(s, fmt)  TO_TIMESTAMP(s, fmt)
AGE(d1, d2)  date + n  date + INTERVAL 'x days'
MAKE_DATE(y, m, d)

-- NULL handling
COALESCE(a, b, c)   -- first non-NULL
NULLIF(a, b)        -- NULL if a=b, else a
GREATEST(a, b, c)   -- max ignoring NULLs
LEAST(a, b, c)      -- min ignoring NULLs

-- Type casting
val::INT  val::TEXT  val::DATE  val::NUMERIC(10,2)
CAST(val AS INT)
```

---

## EXPLAIN

```sql
EXPLAIN SELECT ...;
EXPLAIN ANALYZE SELECT ...;
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;

-- Scan types (speed order, approximate)
Index Only Scan     -- fastest, all data from index
Index Scan          -- index + heap fetch
Bitmap Heap Scan    -- batch heap reads via bitmap
Seq Scan            -- reads entire table

-- Join types
Nested Loop         -- for small tables or indexed inner
Hash Join           -- for large unsorted tables
Merge Join          -- for pre-sorted inputs
```

---

## Indexing

```sql
CREATE INDEX idx ON t(col);                        -- B-tree (default)
CREATE INDEX idx ON t(col) USING HASH;             -- hash, equality only
CREATE INDEX idx ON t(col) USING GIN;              -- JSONB, arrays, full-text
CREATE INDEX idx ON t USING GIN (col gin_trgm_ops); -- trigram
CREATE INDEX idx ON t(col1, col2);                 -- composite
CREATE INDEX idx ON t(col1) INCLUDE (col2, col3);  -- covering
CREATE INDEX idx ON t(LOWER(col));                 -- expression
CREATE INDEX idx ON t(col) WHERE condition;        -- partial
```

---

## psql Quick Commands

```
\l          list databases
\c db       connect to database
\dt         list tables
\d table    describe table
\di         list indexes
\dv         list views
\dm         list materialized views
\df         list functions
\du         list roles
\timing     toggle query timing
\x          toggle expanded output
\e          open editor
\i file.sql run SQL file
\q          quit
\?          psql command help
\h SELECT   SQL syntax help
```
