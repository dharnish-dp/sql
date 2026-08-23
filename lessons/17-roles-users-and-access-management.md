# Lesson 17 — Roles, Users & Access Management

## Goal
Learn how PostgreSQL controls *who* can connect, and *what* they're allowed
to do once connected — the operational layer every real application needs
but SQL tutorials usually skip.

## Prerequisites
[Lesson 00 — Installing PostgreSQL](00-installing-postgresql.md),
[Lesson 01 — How Databases Work](01-how-databases-work.md)

## After This Lesson You Will Be Able To
- Explain the difference between a role, a user, and a group in Postgres
- Create a dedicated app user instead of connecting as your own OS user
- Grant exactly the permissions an application needs — no more, no less
- Restrict access to specific databases, schemas, tables, and columns

---

## There's No Such Thing as "Users" in Postgres — Only Roles

Postgres has one concept: the **role**. A role can:
- Log in (then people casually call it a "user")
- Or just group permissions together (then people call it a "group")

`CREATE USER foo;` is literally shorthand for `CREATE ROLE foo WITH LOGIN;`
— same thing under the hood.

**Why this matters:** when you ran `psql mydb` earlier with no username
specified, you connected *as your own macOS username* (`ddp`), which
Homebrew's install made into a superuser role automatically. That's fine
for learning, but it is exactly what you should **never** do for a real
application.

---

## Why Not Just Use the Superuser for Everything?

Because a superuser (or your own admin role) can drop tables, alter other
users' passwords, and read every row in every database. If your app's
connection string leaks (logs, a GitHub commit, a stack trace), the blast
radius is "the entire cluster" instead of "one database, limited rights."

**The real-world pattern:** every application gets its own role, scoped
tightly to only what that application needs.

---

## Creating an Application Role

```sql
-- Connect as your admin user first
psql postgres

-- Create a role for your app, with a password, that can log in
CREATE ROLE app_user WITH LOGIN PASSWORD 'change-me-in-real-life';

-- Let it connect to the specific database it needs
GRANT CONNECT ON DATABASE mydb TO app_user;
```

At this point `app_user` can *connect* to `mydb`, but can't read or write
anything yet — permissions are opt-in, not opt-out.

---

## Granting Table-Level Access

```sql
\c mydb

-- Read + write on specific tables
GRANT SELECT, INSERT, UPDATE, DELETE ON orders TO app_user;

-- Read-only on a reference table
GRANT SELECT ON products TO app_user;

-- Grant on ALL current tables in a schema (doesn't cover future tables)
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;

-- Make it apply to tables created *later* too
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;
```

If a table uses a `SERIAL`/`IDENTITY` primary key, the app also needs
sequence access to `INSERT`:
```sql
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_user;
```

---

## Common Access Patterns

| Scenario | What to grant |
|---|---|
| App backend (reads + writes) | `SELECT, INSERT, UPDATE, DELETE` on its tables |
| Read-only dashboard / BI tool | `SELECT` only, maybe only on specific views |
| Migration/admin tool | Broader rights, but still not full superuser |
| Analyst who shouldn't see PII | `SELECT` on a *view* that excludes sensitive columns (see [Lesson 13](13-views-and-materialized-views.md)) |

```sql
-- Read-only role example
CREATE ROLE dashboard_reader WITH LOGIN PASSWORD 'x';
GRANT CONNECT ON DATABASE mydb TO dashboard_reader;
GRANT SELECT ON orders, customers TO dashboard_reader;
```

---

## Revoking and Checking Access

```sql
-- Take away a permission
REVOKE INSERT ON orders FROM app_user;

-- See who can access what
\dp orders          -- table-level privileges (inside psql)

-- List all roles
\du
```

---

## Grouping Roles (Role Inheritance)

Instead of granting the same permissions to five app roles one by one,
create a "group role" and have others inherit from it:

```sql
CREATE ROLE readonly_group NOLOGIN;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_group;

CREATE ROLE analyst WITH LOGIN PASSWORD 'x' IN ROLE readonly_group;
-- 'analyst' now has everything readonly_group has, automatically
```

---

## Key Takeaways

1. Postgres only has "roles" — `CREATE USER` is just `CREATE ROLE ... LOGIN`
2. Never connect your application as a superuser/admin role
3. Permissions are opt-in — a new role can do nothing until you `GRANT` it
4. Grant per-table, per-action (`SELECT`/`INSERT`/`UPDATE`/`DELETE`) — the minimum the app actually needs
5. Use `ALTER DEFAULT PRIVILEGES` so future tables inherit the same grants automatically
6. Group roles let you manage permissions for a category of user in one place, not one-by-one

---

## Next Lesson
[Lesson 18 — Connecting to PostgreSQL from Python](18-connecting-from-python.md)
