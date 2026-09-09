# Lesson 20 — Standard SQL vs. PostgreSQL-Specific Features

## Goal
Know which of everything you've learned so far is **portable** — it'll
work the same on MySQL, SQL Server, Oracle, SQLite — and which is a
PostgreSQL extension that won't exist, or works differently, elsewhere.

## Prerequisites
[Lesson 16 — PostgreSQL Power Features](16-postgresql-power-features.md)

## After This Lesson You Will Be Able To
- Tell, for any feature you've learned, whether it's ANSI SQL standard or Postgres-specific
- Predict what breaks (or silently behaves differently) if a query moves from Postgres to MySQL/SQL Server
- Know which Postgres features have no real equivalent elsewhere, and which have a different name/syntax elsewhere
- Write more portable SQL when portability actually matters

---

## Why This Matters

**Why you need this:** every lesson so far taught you PostgreSQL. But
"SQL" isn't one single thing — it's a *standard* (maintained by
ISO/ANSI) that every database vendor implements *most* of, then extends
with their own additions. If you ever join a team using MySQL, or a
job posting says "SQL Server experience," you need to know which of
your Postgres knowledge transfers directly, and which doesn't.

**The core mental model:** think of "SQL" like "JavaScript" — there's a
standard core everyone agrees on, and then each vendor (Postgres, MySQL,
SQL Server, Oracle) adds its own extensions, the same way browsers add
non-standard APIs on top of core JS. Code using only the standard core
runs everywhere; code using an extension is tied to that one engine.

---

## Tier 1 — Fully Standard, Works Everywhere

Everything in this list behaves identically (or near-identically) on
Postgres, MySQL, SQL Server, and Oracle. This is the bulk of what you
learned in [Lessons 01–08](01-how-databases-work.md):

```sql
SELECT, FROM, WHERE, ORDER BY, LIMIT (mostly — see Tier 2), GROUP BY, HAVING
INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN, CROSS JOIN
Basic aggregates: COUNT, SUM, AVG, MIN, MAX
Subqueries, basic CTEs (WITH ...)
CASE WHEN ... THEN ... ELSE ... END
Basic data types: INT, VARCHAR/CHAR, DATE, BOOLEAN (naming varies slightly)
PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, CHECK constraints
Transactions: BEGIN/COMMIT/ROLLBACK, the ACID guarantees themselves
Basic window functions: ROW_NUMBER(), RANK(), OVER (PARTITION BY ... ORDER BY ...)
```

**If you only ever write queries using this list, your SQL is portable
across almost any relational database with minimal changes.** This is
genuinely most of what a typical application query needs.

---

## Tier 2 — Standard Concept, Different Syntax

These features exist in some form on every major engine, but the exact
keyword or syntax differs. Moving between engines means a find-and-replace,
not relearning the concept.

| Concept | PostgreSQL | MySQL | SQL Server |
|---|---|---|---|
| Limit rows returned | `LIMIT 10` | `LIMIT 10` (same) | `TOP 10` (goes in the `SELECT`, not at the end) |
| Auto-incrementing PK | `GENERATED ALWAYS AS IDENTITY` or `SERIAL` | `AUTO_INCREMENT` | `IDENTITY(1,1)` |
| String concatenation | `\|\|` (e.g. `first_name \|\| ' ' \|\| last_name`) | `CONCAT(first_name, ' ', last_name)` | `+` (e.g. `first_name + ' ' + last_name`) |
| Current timestamp | `NOW()` | `NOW()` (same) | `GETDATE()` |
| Case-insensitive match | `ILIKE` | `LIKE` (case-insensitive by default on most collations) | `LIKE` (case-insensitive by default on most collations) |
| Upsert | `ON CONFLICT ... DO UPDATE` | `ON DUPLICATE KEY UPDATE` | `MERGE` (more verbose statement) |
| Boolean type | `BOOLEAN` (`TRUE`/`FALSE`) | No real boolean — `TINYINT(1)`, `0`/`1` | `BIT` (`0`/`1`) |
| Limit + offset | `LIMIT 10 OFFSET 20` | `LIMIT 20, 10` (offset first, reversed order!) | `OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY` |

**The practical implication:** the *concepts* from [Lessons 02–10](02-sql-fundamentals.md)
transfer completely — you already understand what pagination, upsert,
and auto-increment mean. Moving engines just means looking up the local
spelling, not relearning the idea.

---

## Tier 3 — Genuinely PostgreSQL-Specific

These have **no direct equivalent** on other engines, or require a
completely different approach to achieve the same result. Everything
here came from [Lesson 16](16-postgresql-power-features.md) and parts
of [Lesson 09](09-indexes-and-performance.md):

| Feature | What it does | What you'd do instead elsewhere |
|---|---|---|
| `JSONB` type + operators (`->`, `->>`, `@>`) | Native indexed binary JSON storage | MySQL has `JSON` (text-based, less optimized); SQL Server has no native JSON type — stored as `NVARCHAR`, parsed with functions |
| Array types (`INTEGER[]`, `TEXT[]`) | Native array column type | Not available in MySQL/SQL Server — you'd normalize into a separate child table instead |
| `RETURNING` clause | Get back inserted/updated/deleted rows in one statement | MySQL/SQL Server require a separate `SELECT` after, or `OUTPUT` (SQL Server's own different mechanism) |
| `WITH RECURSIVE` | Recursive CTEs | MySQL 8.0+ and SQL Server both support recursive CTEs too, but with syntax quirks — Oracle uses a totally different `CONNECT BY` syntax instead |
| `GIN`/`GiST`/`BRIN` index types | Specialized indexes for arrays, JSONB, full-text, geometry | Other engines have their own specialized index types, but none match Postgers's exactly — full-text search especially differs a lot engine to engine |
| `ROLLUP`/`CUBE`/`GROUPING SETS` | Multi-level subtotal aggregation | MySQL supports `WITH ROLLUP` (less flexible); SQL Server supports all three; Oracle supports all three |
| `INHERITS`, table partitioning syntax | Postgres's own partitioning mechanics | Every engine partitions differently — concepts transfer, syntax doesn't |
| `EXPLAIN (ANALYZE, BUFFERS, ...)` output format | Postgres's specific plan format and node names | Every engine has *an* EXPLAIN, but the terminology (`Seq Scan` vs MySQL's `ALL`, etc.) is engine-specific |

**Why this tier matters most for your "top 1%" goal:** knowing this list
means when you interview or join a MySQL/SQL Server shop, you
immediately know "my JSONB/array/upsert knowledge needs translating,
but my joins/subqueries/transactions/normalization knowledge transfers
directly." That's a much stronger position than not knowing which is
which.

---

## Concepts vs. Syntax — the Distinction That Actually Matters

**Here's the reassuring part:** almost everything conceptual in this
entire course is universal, even where syntax isn't:

- **Normalization** ([Lesson 11](11-database-design-normalization.md)) — the same theory, on every relational engine, no exceptions
- **ACID and transaction isolation levels** ([Lesson 10](10-transactions-and-acid.md)) — the same four properties and anomalies are defined by the SQL standard itself; every engine implements them, sometimes with different default isolation levels (MySQL's InnoDB defaults to `REPEATABLE READ`; Postgres defaults to `READ COMMITTED`)
- **Indexing theory** (B-trees, selectivity, why composite index order matters) ([Lesson 09](09-indexes-and-performance.md)) — universal database theory; every engine uses B-tree-family indexes as its default, even though the specific EXPLAIN output differs
- **Query planning fundamentals** (predicates, joins, aggregation order) — universal; only the specific optimizer implementation differs

**What's engine-specific is almost always syntax and extensions, not
the underlying theory.** This is why this entire course is worth taking
even if you end up working on a different engine day-to-day — you're
learning relational database theory *through* Postgres, and the theory
is what actually transfers.

---

## A Practical Test: Is This Query Portable?

Ask these two questions about any query you write:

1. **Does it use anything from the Tier 3 table** (JSONB, arrays,
   `RETURNING`, Postgres-specific index types)? → Not portable as-is.
2. **Does it use Tier 2 syntax** (`LIMIT`, `SERIAL`, `\|\|`)? → Portable
   *in concept*, but needs a syntax swap to run elsewhere.

If neither applies, you're writing Tier 1 SQL — it'll run essentially
unchanged on any relational database you're likely to encounter.

---

## Key Takeaways

1. SQL has a standard core (ANSI/ISO SQL) that every major engine implements, plus vendor-specific extensions on top
2. Joins, subqueries, basic aggregates, transactions, and constraints are portable almost as-is across engines
3. Pagination, auto-increment, string concatenation, and upsert exist everywhere but with different syntax — same concept, different spelling
4. JSONB, native arrays, `RETURNING`, and Postgres's specific index types have no direct equivalent elsewhere
5. Database *theory* (normalization, ACID, B-tree indexing, query planning) is universal — only *syntax* and certain *features* are engine-specific
6. Before assuming a query will "just work" on a different engine, check whether it touches anything from the Tier 2/Tier 3 tables above

---

## Next Lesson
[Lesson 21 — Database Engine Internals](21-database-engine-internals.md)
