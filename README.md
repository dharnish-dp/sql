# SQL & Database Mastery — Complete Learning Guide

A complete, lesson-by-lesson SQL course designed to take you from zero to top 1%.
Every lesson includes deep concepts, mental models, dozens of runnable examples, and exercises.

---

## Lessons

| Lesson | Topic | What You Master |
|--------|-------|-----------------|
| [Lesson 01](lessons/01-how-databases-work.md) | How Databases Work | Why DBs exist, RDBMS internals, architecture |
| [Lesson 02](lessons/02-sql-fundamentals.md) | SQL Fundamentals | SELECT, FROM, WHERE, ORDER BY, LIMIT, DISTINCT |
| [Lesson 03](lessons/03-data-types-and-schema.md) | Data Types & Schema Design | All types, CREATE TABLE, constraints, ALTER |
| [Lesson 04](lessons/04-filtering-deep-dive.md) | Filtering Deep Dive | WHERE, AND/OR/NOT, LIKE, IN, BETWEEN, IS NULL, CASE |
| [Lesson 05](lessons/05-aggregations.md) | Aggregations | GROUP BY, HAVING, COUNT, SUM, AVG, MIN, MAX |
| [Lesson 06](lessons/06-joins-complete.md) | JOINs — Complete Guide | INNER, LEFT, RIGHT, FULL, CROSS, SELF joins |
| [Lesson 07](lessons/07-subqueries-and-ctes.md) | Subqueries & CTEs | Correlated, EXISTS, WITH, Recursive CTEs |
| [Lesson 08](lessons/08-window-functions.md) | Window Functions | OVER, PARTITION, ROW_NUMBER, RANK, LAG, LEAD |
| [Lesson 09](lessons/09-indexes-and-performance.md) | Indexes & Performance | B-tree, Hash, EXPLAIN, composite indexes |
| [Lesson 10](lessons/10-transactions-and-acid.md) | Transactions & ACID | ACID, isolation levels, deadlocks, locking |
| [Lesson 11](lessons/11-database-design-normalization.md) | Database Design | ERD, 1NF–3NF, BCNF, when to denormalize |
| [Lesson 12](lessons/12-advanced-queries.md) | Advanced Queries | UNION, INTERSECT, EXCEPT, PIVOT, string/date functions |
| [Lesson 13](lessons/13-views-and-materialized-views.md) | Views & Materialized Views | Views, materialized views, refresh strategies |
| [Lesson 14](lessons/14-stored-procedures-and-functions.md) | Stored Procedures & Functions | PL/pgSQL, functions, triggers, events |
| [Lesson 15](lessons/15-query-optimization.md) | Query Optimization | EXPLAIN ANALYZE, planner, rewriting slow queries |
| [Lesson 16](lessons/16-postgresql-power-features.md) | PostgreSQL Power Features | JSONB, arrays, full-text search, extensions |

---

## Real-World Exercises

| Exercise | Scenario | Skills Practiced |
|----------|----------|-----------------|
| [01 — E-Commerce Analytics](exercises/01-ecommerce-analytics.md) | Sales, orders, products | JOINs, aggregations, window functions |
| [02 — Social Network Queries](exercises/02-social-network.md) | Users, follows, posts | Self-joins, CTEs, recursive queries |
| [03 — Financial Ledger](exercises/03-financial-ledger.md) | Accounts, transactions, balances | Transactions, running totals, constraints |
| [04 — Performance Tuning Lab](exercises/04-performance-tuning.md) | Slow queries, explain plans | Indexes, EXPLAIN ANALYZE, rewrites |
| [05 — Schema Design Challenge](exercises/05-schema-design.md) | Design from scratch | Normalization, relationships, constraints |

---

## Quick Reference

- [SQL Cheat Sheet](reference/cheatsheet.md) — every clause with examples
- [Glossary](reference/glossary.md) — every term defined precisely
- [Common Patterns](reference/patterns.md) — copy-paste solutions for recurring problems

---

## How This Course Works

- **Do lessons in order** — each one builds on the last
- **Run every query** — reading SQL without running it doesn't stick
- **PostgreSQL is the dialect** — the most powerful open-source RDBMS, and the one top engineers use
- **MySQL/SQLite notes** are added where behavior differs significantly
- **Concepts first, syntax second** — understanding WHY makes the syntax obvious

---

## Setup (You Install, No Automation Here)

```bash
# PostgreSQL — the database used throughout
brew install postgresql@16      # macOS
sudo apt install postgresql     # Ubuntu/Debian

# Start PostgreSQL
brew services start postgresql@16

# Connect
psql postgres

# Create a practice database
CREATE DATABASE sql_mastery;
\c sql_mastery
```

---

## Recommended Learning Path

**Week 1 — Foundation:** Lessons 01–05
**Week 2 — Intermediate:** Lessons 06–08
**Week 3 — Advanced SQL:** Lessons 09–12
**Week 4 — Engineering Depth:** Lessons 13–16
**Week 5 — Exercises:** All 5 real-world projects

---

*Every query in this course was written to be runnable on PostgreSQL 14+.*
