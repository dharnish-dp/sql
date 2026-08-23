# Lesson 00 (Orientation) — Choosing a Database & How It Fits Into an Application

## Goal
Step back before installing anything and understand the landscape: what
database engines exist, why so many exist, how to actually choose one for
a real application, and where "the database" sits inside a real system
you'd build or work on at a job.

## Prerequisites
None — read this before [Lesson 00 — Installing PostgreSQL](00-installing-postgresql.md).

## After This Lesson You Will Be Able To
- Tell the difference between a database **engine**, a **client tool**, and a **driver**
- Compare the major engines (Postgres, MySQL, SQLite, MongoDB, Redis) and know when each fits
- Make a real choice-of-database decision for a project, not just default to whatever's familiar
- Draw where the database sits in a full application's architecture, end to end
- Understand hosting reality: self-managed vs. managed/cloud databases, and why almost nobody runs Postgres on their laptop in production

---

## Three Things People Conflate: Engine, Client, Driver

This confusion (`psql` vs `mysql` vs "the database") comes up constantly,
so let's nail the vocabulary once:

| Term | What it is | Example |
|---|---|---|
| **Database engine / server** | The actual program storing and managing your data, running as a process | PostgreSQL, MySQL, MongoDB |
| **CLI client** | A terminal tool a human uses to talk to the engine directly | `psql` (for Postgres), `mysql` (for MySQL), `mongosh` (for MongoDB) |
| **Driver / library** | Code your *application* uses to talk to the engine programmatically | `psycopg` (Python→Postgres), `mysql-connector` (Python→MySQL), `pg` (Node→Postgres) |

**One engine, many ways to reach it.** `psql` and your Python app both
ultimately do the same thing — open a connection and send SQL — they're
just two different doors into the same room.

---

## The Database Landscape

### Relational (SQL) engines — data in tables, connected by keys

| Engine | Best known for | Watch out for |
|---|---|---|
| **PostgreSQL** | Feature depth (JSON, full-text search, window functions), strict correctness, free & open-source, no licensing traps | Slightly heavier to self-host than SQLite; overkill for a tiny single-user tool |
| **MySQL / MariaDB** | Massive ecosystem (WordPress, countless legacy apps), simple replication setups, historically fast for simple read-heavy loads | Historically looser on data-type strictness by default; fewer advanced features than Postgres |
| **SQLite** | Zero server — it's a single file on disk, embedded directly in your app | No concurrent writers at scale; wrong choice for a multi-user web backend |
| **SQL Server / Oracle** | Enterprise environments already standardized on Microsoft/Oracle stacks | Commercial licensing costs, often locked into Windows/enterprise tooling |

### NoSQL engines — data that doesn't fit neatly into rows/columns

| Engine | Type | Best known for | Watch out for |
|---|---|---|---|
| **MongoDB** | Document store | Flexible schema — each record can have different fields, good for rapidly-changing data shapes | Weaker at enforcing relationships/consistency across documents; easy to end up duplicating data everywhere |
| **Redis** | Key-value, in-memory | Extremely fast reads/writes, great for caching, sessions, rate-limiting, leaderboards | Data lives in memory — not meant as your *only* copy of important data; usually paired *alongside* a real database, not instead of one |
| **Elasticsearch** | Search index | Full-text search at scale, log aggregation | Not a source of truth — it's a search layer built on top of your real data |

**Rule of thumb:** relational databases are the default choice unless
you have a *specific* reason to reach for something else (search,
caching, embedded/offline apps, wildly variable document shapes).

---

## How to Actually Choose — A Real Decision Process

Ask these in order:

**1. Does the data have clear relationships?**
Users have orders, orders have line items, line items reference products.
→ If yes: relational (Postgres/MySQL). This covers the overwhelming
majority of real business applications.

**2. Do I need strong correctness guarantees?**
Money, inventory counts, bookings — anything where "close enough" causes
real damage.
→ Relational, with real ACID transactions (see [Lesson 10](10-transactions-and-acid.md)).
Postgres is generally considered the strongest here among free engines.

**3. Is the app single-user, local, or embedded** (a desktop tool, a
mobile app's local storage, a CLI tool)?
→ SQLite. No server to install, manage, or secure — it's just a file.

**4. Does my data genuinely vary wildly in shape per record**, with no
practical way to model it as fixed columns (e.g., arbitrary user-defined
form fields, deeply nested unstructured logs)?
→ Consider MongoDB — but note Postgres's `JSONB` type (Lesson 16) covers
a lot of "flexible schema" needs *within* a relational database, which is
often the better middle ground.

**5. Do I need a fast, disposable cache** on top of data I already store
somewhere durable (session tokens, rate limits, computed leaderboards)?
→ Redis, *in addition to* your primary database, never as a replacement.

**In practice:** for the vast majority of applications — SaaS products,
internal tools, e-commerce, anything with users/accounts/transactions —
**PostgreSQL is the safe, modern, boring-in-a-good-way default.** It does
almost everything MySQL does, plus JSON, full-text search, arrays, and
stricter correctness. That's why this entire course teaches Postgres
specifically.

---

## Where the Database Sits in a Real Application

```
┌─────────────────────┐
│  User's Browser/App  │   ← never talks to the database directly
└──────────┬───────────┘
           │ HTTPS / API calls
           ▼
┌─────────────────────────────┐
│      Your App Server         │   ← business logic: auth, validation,
│  (Python / Node / Java etc.) │      "can this user do this?"
└──────────┬───────────────────┘
           │ driver (psycopg, pg, JDBC...)
           │ — see Lesson 18
           ▼
┌─────────────────────────────┐
│     PostgreSQL Server        │   ← the actual data, and only the data
│  (roles, databases, tables)  │      — see Lessons 00–19
└───────────────────────────────┘
```

**Key point:** the browser/mobile app never gets direct database
credentials. It talks to *your app server* over an API; your app server
holds the database credentials (via `.env`/secrets manager, [Lesson 18](18-connecting-from-python.md))
and is the only thing that ever opens a connection to Postgres. This is
also why access control matters at the app-server layer *and* the
database layer ([Lesson 17](17-roles-users-and-access-management.md)) —
they're two independent walls, not one.

**A slightly more complete real-world picture** adds a caching layer and
a connection pool:

```
Browser → App Server → [check Redis cache first] → PostgreSQL (via connection pool)
```

Many requests never even reach Postgres — the app checks Redis for a
cached answer first, and only queries Postgres on a cache miss. This is
why "the database" in a real system is rarely the *only* data-related
component, even though it's the one holding the true, durable copy.

---

## Where Does It Actually Run? (Hosting Reality)

Everything in this course so far runs Postgres on **your own laptop** —
great for learning, but real applications almost never do this. Options,
roughly in order of how common they are in industry:

| Option | What it means | When used |
|---|---|---|
| **Managed cloud database** | AWS RDS, Google Cloud SQL, Azure Database, Supabase, Neon — the provider handles backups, patching, scaling, failover | Default choice for almost every modern company/startup |
| **Self-hosted on a cloud VM** | You install Postgres yourself on an EC2/Droplet-style server | Cost-sensitive teams, or specific control requirements |
| **Containerized (Docker)** | Postgres runs in a container, often alongside your app in the same deployment | Local dev environments, some production setups (Kubernetes) |
| **Your own laptop** | What you're doing now | Learning only — never a real deployment target |

The reason managed hosting dominates: databases are hard to operate
correctly (backups, replication, security patches, scaling) and getting
it wrong means losing real data — most companies would rather pay a cloud
provider to handle that than do it themselves.

---

## Key Takeaways

1. "Engine" (Postgres/MySQL/Mongo), "client" (psql/mysql), and "driver" (psycopg) are three different things — don't conflate them
2. Relational databases are the default for anything with clear relationships and correctness needs — which is most real applications
3. NoSQL/cache tools (MongoDB, Redis, Elasticsearch) solve specific problems and usually sit *alongside* a relational database, not instead of one
4. The browser/app never talks to the database directly — it always goes through your app server, which holds the credentials and driver connection
5. Real applications run on managed cloud databases, not a personal laptop — this course's local setup is purely for learning

---

## Next Lesson
[Lesson 00 — Installing PostgreSQL](00-installing-postgresql.md)
