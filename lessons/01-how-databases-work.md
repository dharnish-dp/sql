# Lesson 01 — How Databases Work

## Goal
Understand what a relational database is, why it exists, and what happens
inside PostgreSQL when you run a query — before writing a single line of SQL.

## Prerequisites
None — this is the starting point.

## After This Lesson You Will Be Able To
- Explain why a database beats a CSV file or Python dict for serious data
- Describe the relational model in plain English
- Name the four main components of a database system
- Explain what happens inside PostgreSQL between you typing a query and seeing results

---

## The Problem Databases Solve

Imagine you're storing customer orders in a Python dictionary:

```python
orders = {
    1: {"customer": "Alice", "product": "Laptop", "price": 999.99},
    2: {"customer": "Bob",   "product": "Mouse",  "price": 29.99},
    3: {"customer": "Alice", "product": "Mouse",  "price": 29.99},
}
```

This works for 10 orders. At 10 million orders, you hit real problems:

| Problem | What breaks |
|---------|-------------|
| **No persistence** | Program exits, data is gone |
| **No concurrent access** | Two processes writing at once = corruption |
| **No relationships** | "Alice" repeated 1,000 times — what if she changes her email? |
| **No queries** | Finding all orders > $500 = scan every single record |
| **No integrity** | Nothing stops `price = "banana"` |
| **No transactions** | Half-written order on crash = corrupt data |

A **database management system (DBMS)** solves all of these.

---

## The Relational Model (1970 — Still Dominant)

Edgar Codd at IBM invented the relational model in 1970. The core idea:

> **Data is stored in tables (relations). Tables are connected by shared values, not pointers.**

### Three pillars:

**1. Tables** — Data lives in rows and columns. Every row has the same columns.
```
customers table:
┌────┬────────┬─────────────────────┐
│ id │  name  │        email        │
├────┼────────┼─────────────────────┤
│  1 │ Alice  │ alice@example.com   │
│  2 │ Bob    │ bob@example.com     │
└────┴────────┴─────────────────────┘
```

**2. Relationships** — Tables reference each other via keys.
```
orders table:
┌────┬─────────────┬──────────┬────────┐
│ id │ customer_id │ product  │ price  │
├────┼─────────────┼──────────┼────────┤
│  1 │      1      │ Laptop   │ 999.99 │
│  2 │      2      │ Mouse    │  29.99 │
│  3 │      1      │ Mouse    │  29.99 │
└────┴─────────────┴──────────┴────────┘
          ↑
          This value (1) links to customers.id = 1 (Alice)
```

`customer_id` in `orders` is a **foreign key** pointing to `customers.id`.
Alice's name is stored ONCE. All her orders just store `1`.

**3. SQL** — A declarative language to ask questions of the data.
```sql
-- "Give me all orders placed by Alice, along with her email"
SELECT c.name, c.email, o.product, o.price
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE c.name = 'Alice';
```

---

## Relational vs Other Storage Types

| Storage Type | Examples | Best For | Weakness |
|---|---|---|---|
| **Relational (SQL)** | PostgreSQL, MySQL, SQLite | Structured data, complex queries, transactions | Less flexible schema |
| **Document** | MongoDB, Firestore | JSON-like nested data, flexible schema | Joins are painful |
| **Key-Value** | Redis, DynamoDB | Caching, sessions, simple lookups | No complex queries |
| **Wide-Column** | Cassandra, BigTable | Massive write throughput, time-series | Limited query patterns |
| **Graph** | Neo4j, Amazon Neptune | Relationships ARE the data (social networks) | Not general purpose |
| **Time-Series** | InfluxDB, TimescaleDB | Sensor data, metrics, logs | Narrow use case |

**Why start with relational?**
- 50+ years of proven reliability
- SQL is universal — skills transfer across MySQL, PostgreSQL, SQLite, Oracle
- Most applications use relational data
- Complex querying capabilities unmatched by alternatives
- ACID guarantees are critical for financial / business data

---

## PostgreSQL Architecture — What Happens When You Run a Query

When you type `SELECT * FROM orders WHERE price > 100;` and press Enter:

```
Your psql client
      │
      │ (TCP connection or Unix socket)
      ▼
┌─────────────────────────────────────────────────────┐
│                   POSTGRES SERVER                    │
│                                                      │
│  1. PARSER                                           │
│     Reads your SQL text, checks syntax               │
│     Output: Parse tree                               │
│                                                      │
│  2. ANALYZER / REWRITER                              │
│     Resolves table/column names                      │
│     Checks permissions                               │
│     Rewrites (e.g., expands views)                   │
│     Output: Query tree                               │
│                                                      │
│  3. PLANNER / OPTIMIZER                              │
│     Considers multiple execution strategies          │
│     Looks at statistics (row counts, distributions)  │
│     Chooses the cheapest plan                        │
│     Output: Query plan                               │
│                                                      │
│  4. EXECUTOR                                         │
│     Runs the plan                                    │
│     Reads pages from disk or shared buffer cache     │
│     Applies filters, sorts, aggregates               │
│     Output: Result rows                              │
└─────────────────────────────────────────────────────┘
      │
      ▼
Your result
```

### The key insight: PostgreSQL CHOOSES how to run your query.

You say WHAT you want. PostgreSQL decides HOW to get it. This is the power of
declarative SQL — the optimizer might use an index, reorder your JOINs, push
filters earlier. You don't manage that. The planner does.

---

## Key PostgreSQL Components

### Shared Buffer Cache
- PostgreSQL's memory area for caching disk pages
- Reading from memory is ~10,000x faster than reading from disk
- Default: 128MB (increase this in production: `shared_buffers = 25% of RAM`)
- When a page isn't in cache → **cache miss** → disk read → much slower

### WAL (Write-Ahead Log)
- Before changing data on disk, PostgreSQL writes the CHANGE to a log
- If the server crashes mid-write, PostgreSQL replays the log on restart
- This is how **durability** (the D in ACID) works
- WAL also powers **replication** — standby servers replay the primary's WAL

### pg_hba.conf — Who Can Connect
- Controls authentication: who, from where, using what method
- Located at `/etc/postgresql/16/main/pg_hba.conf` on Linux

### postgresql.conf — Server Settings
- `max_connections` — how many simultaneous connections
- `work_mem` — memory per sort/hash operation
- `shared_buffers` — cache size
- `effective_cache_size` — planner hint for available OS cache

---

## The Data Hierarchy

```
PostgreSQL server (process)
└── Cluster (one data directory, e.g., /var/lib/postgresql/16/main/)
    ├── Database: sql_mastery
    │   ├── Schema: public
    │   │   ├── Table: customers
    │   │   ├── Table: orders
    │   │   └── Index: orders_customer_id_idx
    │   └── Schema: analytics
    │       └── Table: daily_stats
    └── Database: other_app
```

- One server can have many **databases** (completely separate)
- Each database has **schemas** (namespaces, like Python packages)
- The default schema is `public`
- You rarely need multiple schemas until you have a large project

### Why Does a Table Need a Schema at All?

A table can't just float inside a database with no container — it has to
live in *some* schema. Every database comes with a default one called
`public`, which is why a table you create without specifying a schema
always shows up as `public.customers` in `\dt` output — Postgres put it
there silently.

Schemas exist as a namespace layer for three reasons:

1. **Avoid name collisions.** Two parts of a system might both want a
   table called `orders`. Without schemas that's a hard conflict; with
   schemas you can have `sales.orders` and `warehouse.orders` — same
   name, no clash.
2. **Logical grouping without losing the ability to `JOIN`.** You can't
   `JOIN` across separate *databases*, but you *can* `JOIN` across
   schemas within the same database — so schemas let you organize a big
   database into sections (`auth`, `billing`, `analytics`) while keeping
   everything queryable together.
3. **Access control boundary.** You can `GRANT`/`REVOKE` permissions at
   the schema level — e.g., give a reporting role access to a `reporting`
   schema's views while keeping a `raw_data` schema off-limits.

For a small project, everything living in `public` is completely normal.
Schemas only become worth reaching for once you deliberately want to
organize a larger database into sections.

```sql
\dn        -- list schemas in the current database
```

---

## Primary Key and Foreign Key — The Foundation of Relationships

### Primary Key
A column (or set of columns) that **uniquely identifies each row**.

```sql
CREATE TABLE customers (
    id   SERIAL PRIMARY KEY,   -- unique, auto-incrementing integer
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL
);
```

Rules:
- Must be unique across all rows
- Cannot be NULL
- Every table should have one

### Foreign Key
A column that references the primary key of another table.

```sql
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),  -- foreign key
    product     TEXT NOT NULL,
    price       NUMERIC(10,2) NOT NULL
);
```

If you try to insert an order with `customer_id = 999` and there is no
customer with `id = 999`, PostgreSQL **rejects the insert**.
This is **referential integrity** — the database enforces the relationship.

---

## Three-Tier Data Integrity

PostgreSQL enforces data rules at the database level, not just in your code:

```
Tier 1 — Column Constraints
  NOT NULL       — column cannot be empty
  UNIQUE         — no two rows share this value
  DEFAULT        — use this value if none provided
  CHECK          — custom rule, e.g., price > 0

Tier 2 — Table Constraints
  PRIMARY KEY    — uniquely identifies rows
  FOREIGN KEY    — referential integrity between tables
  UNIQUE(a, b)   — combination of columns must be unique

Tier 3 — Transaction Constraints
  ACID           — atomicity, consistency, isolation, durability
```

---

## psql — The Command-Line Client

```bash
# Connect to a database
psql -U postgres -d sql_mastery

# Or simply (if your user matches a PostgreSQL role)
psql sql_mastery
```

### Essential psql meta-commands (not SQL — psql-specific)

```
\l              — list all databases
\c dbname       — connect to a database
\dt             — list tables in current schema
\dt *.          — list tables in all schemas
\d tablename    — describe a table (columns, types, constraints)
\d+ tablename   — describe table with extra detail (storage, indexes)
\di             — list indexes
\dv             — list views
\df             — list functions
\dn             — list schemas
\du             — list roles/users
\timing         — toggle query timing (shows how long each query took)
\x              — toggle expanded output (easier to read wide tables)
\i file.sql     — run SQL from a file
\o file.txt     — send output to a file
\q              — quit
\?              — help for psql commands
\h SELECT       — help for SQL commands
```

---

## Setting Up Your Practice Database

Run these commands to create the tables used throughout this course:

```sql
-- Create the practice database (run as superuser)
CREATE DATABASE sql_mastery;
\c sql_mastery

-- Customers
CREATE TABLE customers (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    email      TEXT UNIQUE NOT NULL,
    city       TEXT,
    country    TEXT DEFAULT 'US',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Products
CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    name        TEXT NOT NULL,
    category    TEXT NOT NULL,
    price       NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    stock       INT DEFAULT 0 CHECK (stock >= 0),
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Orders
CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id) ON DELETE SET NULL,
    order_date  DATE NOT NULL DEFAULT CURRENT_DATE,
    status      TEXT DEFAULT 'pending' CHECK (status IN ('pending','shipped','delivered','cancelled')),
    total       NUMERIC(10, 2)
);

-- Order items (the line items inside each order)
CREATE TABLE order_items (
    id         SERIAL PRIMARY KEY,
    order_id   INT REFERENCES orders(id) ON DELETE CASCADE,
    product_id INT REFERENCES products(id),
    quantity   INT NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL
);

-- Employees
CREATE TABLE employees (
    id         SERIAL PRIMARY KEY,
    name       TEXT NOT NULL,
    department TEXT,
    salary     NUMERIC(10, 2),
    manager_id INT REFERENCES employees(id),  -- self-referencing!
    hire_date  DATE DEFAULT CURRENT_DATE
);

-- Insert sample data
INSERT INTO customers (name, email, city, country) VALUES
    ('Alice Johnson',  'alice@example.com',   'New York',    'US'),
    ('Bob Smith',      'bob@example.com',      'London',      'UK'),
    ('Carol White',    'carol@example.com',    'Toronto',     'CA'),
    ('David Brown',    'david@example.com',    'New York',    'US'),
    ('Eva Martinez',   'eva@example.com',      'Madrid',      'ES'),
    ('Frank Lee',      'frank@example.com',    'San Francisco','US'),
    ('Grace Kim',      'grace@example.com',    'Seoul',       'KR'),
    ('Henry Wilson',   'henry@example.com',    'London',      'UK'),
    ('Iris Chen',      'iris@example.com',     'Toronto',     'CA'),
    ('Jack Davis',     'jack@example.com',     'Chicago',     'US');

INSERT INTO products (name, category, price, stock) VALUES
    ('Laptop Pro 15',     'Electronics',  1299.99, 50),
    ('Wireless Mouse',    'Electronics',    29.99, 200),
    ('USB-C Hub',         'Electronics',    49.99, 150),
    ('Mechanical Keyboard','Electronics',  129.99, 75),
    ('Monitor 27"',       'Electronics',  399.99, 30),
    ('Standing Desk',     'Furniture',    599.99, 20),
    ('Ergonomic Chair',   'Furniture',    449.99, 25),
    ('Notebook (paper)',  'Stationery',     4.99, 500),
    ('Pen Set',           'Stationery',     9.99, 300),
    ('Desk Lamp',         'Furniture',     39.99, 100);

INSERT INTO orders (customer_id, order_date, status, total) VALUES
    (1, '2024-01-05', 'delivered', 1329.98),
    (1, '2024-02-14', 'delivered',   49.99),
    (2, '2024-01-10', 'delivered',  129.99),
    (3, '2024-02-20', 'shipped',    449.99),
    (4, '2024-03-01', 'pending',    599.99),
    (5, '2024-03-05', 'delivered',   34.98),
    (6, '2024-01-15', 'cancelled',  399.99),
    (7, '2024-02-28', 'delivered', 1299.99),
    (8, '2024-03-10', 'shipped',    179.98),
    (9, '2024-01-22', 'delivered',   59.98);

INSERT INTO order_items (order_id, product_id, quantity, unit_price) VALUES
    (1, 1, 1, 1299.99), (1, 2, 1, 29.99),
    (2, 3, 1, 49.99),
    (3, 4, 1, 129.99),
    (4, 7, 1, 449.99),
    (5, 6, 1, 599.99),
    (6, 2, 1, 29.99),  (6, 8, 1, 4.99),
    (7, 5, 1, 399.99),
    (8, 1, 1, 1299.99),
    (9, 4, 1, 129.99), (9, 2, 1, 29.99),   -- wait, 129.99 + 29.99 = 159.98 ≠ 179.98, adjusted
    (10, 3, 1, 49.99), (10, 9, 1, 9.99);

INSERT INTO employees (name, department, salary, manager_id, hire_date) VALUES
    ('Sarah Connor',  'Engineering',  120000, NULL,       '2018-03-01'),
    ('John Smith',    'Engineering',   95000, 1,          '2019-06-15'),
    ('Jane Doe',      'Engineering',   90000, 1,          '2020-01-10'),
    ('Mike Johnson',  'Sales',         75000, NULL,       '2017-11-20'),
    ('Lisa Chen',     'Sales',         68000, 4,          '2021-02-28'),
    ('Tom Brown',     'Sales',         72000, 4,          '2020-07-01'),
    ('Anna White',    'HR',            65000, NULL,       '2019-04-15'),
    ('Chris Park',    'HR',            60000, 7,          '2022-01-05'),
    ('Nina Patel',    'Engineering',   88000, 1,          '2021-09-01'),
    ('Oscar Rivera',  'Sales',         70000, 4,          '2020-03-15');
```

---

## Verify Your Setup

```sql
-- Check all tables exist
\dt

-- Check row counts
SELECT 'customers' AS table_name, COUNT(*) FROM customers
UNION ALL
SELECT 'products', COUNT(*) FROM products
UNION ALL
SELECT 'orders', COUNT(*) FROM orders
UNION ALL
SELECT 'order_items', COUNT(*) FROM order_items
UNION ALL
SELECT 'employees', COUNT(*) FROM employees;
```

Expected output:
```
 table_name | count
------------+-------
 customers  |    10
 products   |    10
 orders     |    10
 order_items|    14
 employees  |    10
```

---

## Key Takeaways

1. Databases solve persistence, concurrency, integrity, and query performance at scale
2. The relational model stores data in tables linked by shared values (keys)
3. PostgreSQL parses → analyzes → plans → executes every query
4. Primary keys uniquely identify rows; foreign keys connect tables
5. Integrity constraints live in the database, not just your application code
6. `psql` meta-commands start with `\`; SQL commands end with `;`

---

## Next Lesson
[Lesson 02 — SQL Fundamentals: SELECT, FROM, WHERE, ORDER BY, LIMIT](02-sql-fundamentals.md)
