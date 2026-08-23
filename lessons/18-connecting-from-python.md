# Lesson 18 — Connecting to PostgreSQL from Python

## Goal
Go from "I can run SQL in psql" to "my Python application can talk to the
database" — connection strings, drivers, and the pattern real apps use
instead of hardcoding credentials.

## Prerequisites
[Lesson 17 — Roles, Users & Access Management](17-roles-users-and-access-management.md)

## After This Lesson You Will Be Able To
- Install and use a PostgreSQL driver in Python
- Build a connection string safely (no hardcoded passwords)
- Run parameterized queries without SQL injection risk
- Understand when to use a connection pool instead of one-off connections

---

## The Driver: psycopg

Python doesn't talk to Postgres natively — you need a driver library.
The standard one is **psycopg** (currently version 3; you'll also see the
older `psycopg2` in a lot of existing code, API is very similar).

```bash
pip install "psycopg[binary]"
```

`[binary]` pulls a precompiled version so you don't need Postgres's C
headers installed to build it from source.

---

## Your First Connection

Using the `app_user` role and `mydb` database from
[Lesson 17](17-roles-users-and-access-management.md):

```python
import psycopg

conn = psycopg.connect(
    host="localhost",
    port=5432,
    dbname="mydb",
    user="app_user",
    password="change-me-in-real-life",
)

cur = conn.cursor()
cur.execute("SELECT id, name FROM customers LIMIT 5;")
for row in cur.fetchall():
    print(row)

cur.close()
conn.close()
```

This is the same thing `psql mydb` does — connect to `localhost:5432`,
authenticate as a role, run SQL — just from Python instead of a terminal.

---

## Never Hardcode Credentials

The snippet above has a password sitting in plain text in your source code
— exactly what would leak if this file ever hit a public GitHub repo.
Use environment variables instead:

```bash
pip install python-dotenv
```

```
# .env  (add this file to .gitignore — never commit it)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mydb
DB_USER=app_user
DB_PASSWORD=change-me-in-real-life
```

```python
import os
from dotenv import load_dotenv
import psycopg

load_dotenv()

conn = psycopg.connect(
    host=os.environ["DB_HOST"],
    port=os.environ["DB_PORT"],
    dbname=os.environ["DB_NAME"],
    user=os.environ["DB_USER"],
    password=os.environ["DB_PASSWORD"],
)
```

**Production environments** (Docker, cloud hosting, CI) set these same
environment variables through the platform's secrets manager instead of a
`.env` file — the code doesn't change either way.

---

## Never Format SQL Strings by Hand

```python
# NEVER do this — SQL injection risk
name = "Alice"
cur.execute(f"SELECT * FROM customers WHERE name = '{name}'")
```

If `name` ever comes from user input (a web form, an API request), an
attacker can inject `'; DROP TABLE customers; --` and it runs as real SQL.

```python
# ALWAYS do this — the driver escapes the value safely
cur.execute("SELECT * FROM customers WHERE name = %s", (name,))
```

The `%s` is a placeholder the driver fills in *after* escaping — it's never
concatenated into the SQL text itself.

---

## Transactions from Python

By default, psycopg opens a transaction on the first query and does **not**
auto-commit — you must call `conn.commit()` explicitly, or your changes
never persist. See [Lesson 10](10-transactions-and-acid.md) for what a
transaction actually is.

```python
try:
    cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1;")
    cur.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2;")
    conn.commit()
except Exception:
    conn.rollback()
    raise
```

Or use psycopg's context manager, which commits automatically on success
and rolls back automatically on an exception:
```python
with psycopg.connect(...) as conn:
    with conn.cursor() as cur:
        cur.execute("UPDATE accounts SET balance = balance - 100 WHERE id = 1;")
        cur.execute("UPDATE accounts SET balance = balance + 100 WHERE id = 2;")
    # commits here if no exception was raised
```

---

## Connection Pooling — Why You'll Need It Soon

Opening a new TCP connection to Postgres for every request is slow and
doesn't scale — a web app handling concurrent requests would exhaust
Postgres's connection limit fast. Instead, open a **pool** of connections
once, and hand them out/return them as requests come in.

```bash
pip install "psycopg[pool]"
```

```python
from psycopg_pool import ConnectionPool

pool = ConnectionPool(
    "host=localhost port=5432 dbname=mydb user=app_user password=change-me-in-real-life",
    min_size=2,
    max_size=10,
)

with pool.connection() as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT 1;")
        print(cur.fetchone())
```

Most Python web frameworks (FastAPI, Flask, Django) either handle pooling
for you via an ORM, or expect you to set up a pool like this once at app
startup and reuse it for the app's lifetime.

---

## Where ORMs Fit In

Once an app grows, writing raw SQL strings everywhere gets repetitive and
error-prone. Tools like **SQLAlchemy** or **Django's ORM** sit on top of a
driver like psycopg and let you write Python objects instead of SQL text:

```python
# SQLAlchemy example — conceptually the same connection underneath
customer = session.query(Customer).filter_by(name="Alice").first()
```

Everything in this lesson (roles, connection strings, pooling, transactions)
is still happening underneath — the ORM just generates the SQL for you.
Understanding raw SQL and psycopg first (what you're doing now) makes ORM
behavior much less mysterious later.

---

## Key Takeaways

1. `psycopg` is the standard PostgreSQL driver for Python — connects the same way `psql` does, just from code
2. Load credentials from environment variables (`.env` locally, secrets manager in production) — never hardcode them
3. Always use parameterized queries (`%s` placeholders) — never f-string/format raw SQL
4. Transactions don't auto-commit by default — call `commit()` or use a context manager
5. Use a connection pool for any real application — one-off connections per request don't scale
6. ORMs (SQLAlchemy, Django) are a layer on top of this, not a replacement for understanding it

---

## Next Lesson
[Lesson 19 — Network Access & LAN Connections](19-network-access-and-lan-connections.md)
