# SQL & PostgreSQL Glossary

---

**ACID** — The four guarantees of a reliable transaction:
Atomicity (all or nothing), Consistency (valid state to valid state),
Isolation (transactions don't see each other's uncommitted changes),
Durability (committed data survives crashes).

**Aggregate Function** — A function that computes one value across a group of rows:
`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`.

**Alias** — A temporary name for a column or table. `SELECT name AS customer_name`,
`FROM customers c`.

**Autocommit** — PostgreSQL mode where each statement is its own transaction,
automatically committed. Use `BEGIN` to group statements explicitly.

**B-tree** — Balanced Tree. The default index type in PostgreSQL.
Supports equality, range, ORDER BY, LIKE 'prefix%'.

**BCNF** (Boyce-Codd Normal Form) — A stricter form of 3NF. Every determinant
must be a candidate key.

**Candidate Key** — Any column or set of columns that could serve as a primary key
(unique and non-null).

**Cardinality** — The number of unique values in a column, or the ratio of distinct
values to total rows. High cardinality → good index candidate.

**CASCADE** — A foreign key action that propagates changes.
`ON DELETE CASCADE`: deletes child rows when parent is deleted.
`ON UPDATE CASCADE`: updates FK columns when the referenced key changes.

**Checkpoint** — PostgreSQL periodically writes dirty pages from memory to disk.
A checkpoint ensures durability without writing after every transaction.

**CTE (Common Table Expression)** — A named temporary result set defined with the
`WITH` clause. Improves query readability and allows recursive queries.

**Composite Key** — A primary key made of two or more columns.

**Correlated Subquery** — A subquery that references columns from the outer query.
Runs once per outer row.

**CROSS JOIN** — Returns every combination of rows from two tables (Cartesian product).

**Cursor** — A database object that allows iterating through a result set row by row.
Available in PL/pgSQL with `DECLARE cur CURSOR FOR SELECT ...`.

**Data Definition Language (DDL)** — SQL commands that define structure:
`CREATE`, `ALTER`, `DROP`, `TRUNCATE`.

**Data Manipulation Language (DML)** — SQL commands that manipulate data:
`SELECT`, `INSERT`, `UPDATE`, `DELETE`.

**Deadlock** — When two transactions each hold a lock the other needs, neither
can proceed. PostgreSQL detects and aborts one automatically.

**Derived Table** — A subquery in the FROM clause, treated as a temporary table.
Must have an alias.

**Dirty Read** — Reading uncommitted data from another transaction.
PostgreSQL never allows this (prevents it at READ UNCOMMITTED level).

**DISTINCT** — Removes duplicate rows from a result set.

**DISTINCT ON** — PostgreSQL extension: returns the first row per distinct value
of specified columns (requires ORDER BY starting with those columns).

**EXISTS** — Returns TRUE if a subquery returns any row.
Short-circuits on first match. NULL-safe (unlike IN).

**Expression Index** — An index on a computed expression rather than a raw column:
`CREATE INDEX ON customers(LOWER(email))`.

**EXPLAIN** — Shows the query execution plan without running the query.
`EXPLAIN ANALYZE` runs the query and shows actual timing.

**Foreign Key (FK)** — A column that references the primary key of another table.
Enforces referential integrity.

**Full-Text Search (FTS)** — Searching within text content using linguistic analysis
(stemming, stopword removal). Uses `tsvector` and `tsquery` in PostgreSQL.

**GIN (Generalized Inverted Index)** — An index type for multi-value data:
arrays, JSONB, full-text vectors. Better for search; slower to build than B-tree.

**GiST (Generalized Search Tree)** — An index type for complex types:
geometry, ranges, full-text search. Can index types that B-tree can't.

**GROUP BY** — Splits rows into groups for aggregate computation.

**HAVING** — Filters groups after GROUP BY aggregation.
WHERE filters rows; HAVING filters groups.

**Heap** — PostgreSQL's term for the main table storage file (the data pages,
as opposed to index pages).

**Index** — A data structure that enables fast row lookup without scanning
the entire table. Trade-off: faster reads, slower writes.

**Index Only Scan** — When all columns needed by a query are in the index,
PostgreSQL doesn't need to touch the heap at all.

**Index Scan** — PostgreSQL uses the index to find row locations, then fetches
from the heap for each match.

**Isolation Level** — Controls how visible uncommitted changes from other
transactions are: READ COMMITTED (default), REPEATABLE READ, SERIALIZABLE.

**JOIN** — Combines rows from two or more tables based on a related column.

**JSONB** — PostgreSQL's binary JSON storage type. Supports operators, indexing,
and fast querying. Prefer over `JSON`.

**Junction Table** — A table that implements a many-to-many relationship
by holding foreign keys to both related tables.

**LATERAL** — Allows a FROM subquery to reference columns from preceding
FROM items (like a correlated subquery in the FROM clause).

**LIMIT** — Restricts the number of rows returned.

**Lock** — A mechanism to prevent conflicts between concurrent transactions.
Row-level locks, table-level locks, and advisory locks.

**Materialized View** — A view that stores its result on disk. Must be refreshed
manually. Faster than regular views for expensive queries.

**NULL** — Represents an unknown or missing value. Not equal to anything,
including itself. Use `IS NULL` / `IS NOT NULL`.

**Normalization** — The process of organizing a database to reduce data redundancy
and improve integrity. Normal forms: 1NF, 2NF, 3NF, BCNF.

**OFFSET** — Skips the first N rows before returning results. Used with LIMIT for pagination.

**ON CONFLICT** — PostgreSQL UPSERT syntax: specify what to do when a unique
constraint is violated (DO UPDATE or DO NOTHING).

**ORDER BY** — Sorts the result set.

**Partial Index** — An index on a subset of rows (with a WHERE clause).
Smaller and faster for targeted queries.

**Partition** — A physical subtable of a larger partitioned table.
Data is automatically routed to the correct partition based on partition key.

**Phantom Read** — A transaction re-runs a query and sees new rows that didn't
exist before (inserted by another transaction). Prevented at REPEATABLE READ+.

**PL/pgSQL** — PostgreSQL's procedural language. Used in functions and triggers.
Supports variables, loops, conditions, exception handling.

**Primary Key (PK)** — One or more columns that uniquely identify each row.
Automatically creates a unique index.

**Query Planner** — The part of PostgreSQL that decides HOW to execute a query
(which indexes to use, join order, etc.) based on statistics and cost estimates.

**RECURSIVE CTE** — A CTE that references itself to traverse hierarchical data
(trees, graphs). Uses UNION ALL with a base case and recursive case.

**Referential Integrity** — The guarantee that foreign key values always point
to existing rows in the referenced table.

**Relation** — The formal term for a table in relational theory.

**Row-Level Security (RLS)** — Policies that filter which rows a user can
see or modify. Enforced at the database level.

**SAVEPOINT** — A named point within a transaction. You can ROLLBACK TO SAVEPOINT
to undo changes since that point without undoing the entire transaction.

**Schema** — A namespace for database objects (tables, views, indexes, functions).
Default is `public`. Like a Python package for your database objects.

**SELECT FOR UPDATE** — Locks the selected rows for the duration of the transaction.
Other transactions trying to lock the same rows must wait.

**Sequence** — A database object that generates sequential numbers.
Used by SERIAL and IDENTITY columns.

**Sequential Scan (Seq Scan)** — Reading all rows in a table from start to finish.
Fast for small tables or low-selectivity queries; slow for large tables.

**SERIAL** — Legacy PostgreSQL shorthand for an auto-incrementing integer column.
Prefer `GENERATED ALWAYS AS IDENTITY` in new code.

**Subquery** — A query nested inside another query.

**TABLE** — The fundamental storage unit in a relational database.
Organized into rows (records) and columns (fields).

**Transaction** — A group of SQL operations that succeed or fail together.
Controlled with BEGIN, COMMIT, ROLLBACK.

**Trigger** — A function that runs automatically when a table event occurs
(INSERT, UPDATE, DELETE, TRUNCATE). BEFORE or AFTER the event.

**tsvector** — A preprocessed text document used for full-text search.
Words are lexemized (normalized), stopwords removed.

**tsquery** — A full-text search query. Used with the `@@` operator against a tsvector.

**UNIQUE** — A constraint that ensures all values in a column (or combination)
are distinct. Automatically creates an index.

**UPSERT** — Insert or update. If the row exists, update it; otherwise insert.
PostgreSQL syntax: `INSERT ... ON CONFLICT ... DO UPDATE`.

**Vacuum** — PostgreSQL process that removes dead rows (from UPDATEs and DELETEs)
and reclaims space. Autovacuum runs automatically.

**VIEW** — A saved SELECT query. Querying a view runs the underlying SELECT.
No data storage (unlike materialized views).

**WAL (Write-Ahead Log)** — PostgreSQL writes changes to a log BEFORE applying
them to data files. Enables crash recovery and replication.

**Window Function** — A function that computes values across a set of rows
(a "window") related to the current row, without collapsing rows.
`ROW_NUMBER()`, `SUM() OVER (...)`, `LAG()`, etc.

**Work_mem** — PostgreSQL configuration: memory per sort or hash operation.
If a sort exceeds work_mem, it spills to disk (much slower).
