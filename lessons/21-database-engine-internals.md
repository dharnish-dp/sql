# Lesson 21 — Database Engine Internals: How They Actually Differ

## Goal
Go past "Postgres vs MySQL vs SQLite, pick one" surface-level advice
([Lesson 00](00-choosing-a-database-and-how-it-fits.md)) into *how* these
engines actually differ under the hood — concurrency control, replication,
and licensing — so you can reason about engine choice from mechanism,
not just reputation.

## Prerequisites
[Lesson 10 — Transactions & ACID](10-transactions-and-acid.md),
[Lesson 20 — Standard SQL vs PostgreSQL](20-standard-sql-vs-postgresql.md)

## After This Lesson You Will Be Able To
- Explain MVCC and why Postgres uses it instead of plain locking
- Compare how Postgres, MySQL (InnoDB), and SQLite actually handle concurrent writes
- Understand what "replication" means and why it matters for real deployments
- Know the licensing differences between major engines and why it occasionally matters

---

## Why You Need This

**Why you need this:** [Lesson 00](00-choosing-a-database-and-how-it-fits.md)
gave you a decision *checklist* for picking an engine. This lesson
explains the actual *mechanisms* behind that checklist — specifically,
why Postgres handles concurrent reads/writes the way it does, and what
"managed hosting" and "replication" actually mean in practice, since
those terms were used but not unpacked yet.

---

## MVCC — How Postgres Handles Many People Reading/Writing at Once

### The Problem This Solves

Imagine two things happening on `orders` at the exact same moment:
- Transaction A is running `SELECT * FROM orders` (a long-running report)
- Transaction B is running `UPDATE orders SET status = 'shipped' WHERE id = 5`

**The naive solution** would be: lock the whole table so B has to wait
until A's `SELECT` finishes. This works, but it's slow — a long-running
read would block every write behind it, and vice versa.

### What MVCC Actually Does

**MVCC = Multi-Version Concurrency Control.** Instead of locking rows to
prevent conflicts, Postgres keeps **multiple versions of a row
simultaneously** — this is the exact mechanism behind "dead rows" from
[Lesson 09](09-indexes-and-performance.md): every `UPDATE` doesn't
overwrite the row in place, it creates a *new version* and marks the old
one as superseded.

**Why this solves the problem above:** Transaction A's `SELECT`, which
started before B's `UPDATE` committed, keeps seeing the **old version**
of row 5 for its entire duration — it gets a consistent snapshot of the
data as it existed when A started, completely unaffected by B's change.
Transaction B doesn't have to wait for A, and A doesn't see a
half-finished update. Both proceed at the same time, no blocking.

```
Time →
Row 5 version 1: status='pending'   [created at t=0]
                                      ↑ Transaction A's SELECT (started t=1) keeps seeing THIS version
Row 5 version 2: status='shipped'   [created at t=2, by Transaction B's UPDATE]
                                      ↑ any transaction starting AFTER t=2 sees THIS version instead
```

This is *why* Postgres needs `VACUUM` ([Lesson 09](09-indexes-and-performance.md))
— old row versions ("dead rows") pile up as a direct consequence of how
MVCC works, and eventually need cleaning up once nothing needs them
anymore.

### How This Compares to Other Engines

| Engine | Concurrency approach |
|---|---|
| **PostgreSQL** | MVCC — readers never block writers, writers never block readers (mostly) |
| **MySQL (InnoDB)** | Also MVCC, very similar mechanism to Postgres — this is why MySQL and Postgres feel similar under concurrent load |
| **SQLite** | Traditionally whole-database locking — only one writer at a time, ever, full stop (modern WAL mode improves this, but it's still fundamentally more restrictive) |
| **Older MySQL (MyISAM engine)** | Table-level locking — an entire table locks for a write, blocking all other access to it. Rarely used today; InnoDB is the modern MySQL default |

**Why this matters for engine choice:** this is the *actual* mechanical
reason SQLite is wrong for a multi-user web backend ([Lesson 00](00-choosing-a-database-and-how-it-fits.md)
said this, but here's why) — its single-writer model means concurrent
users would serialize on every write, one at a time, no matter how fast
the hardware is.

---

## Replication — Why "Just One Database Server" Isn't the Full Picture

**Why you need this:** [Lesson 19](19-network-access-and-lan-connections.md)
mentioned production databases don't run on one personal machine.
Replication is the actual mechanism that makes a *real* production setup
resilient — copying your data to additional servers automatically.

### The Basic Idea

**Replication = keeping a live, continuously-updated copy of your
database on one or more additional servers.**

```
Primary server (accepts writes)
      │  streams every change
      ▼
Replica server #1 (read-only copy, always catching up)
Replica server #2 (read-only copy, always catching up)
```

**Why this exists — two separate reasons:**

1. **Failover / disaster recovery.** If the primary server dies (hardware
   failure, data center outage), a replica can be promoted to become the
   new primary — your application keeps running with, at most, a brief
   interruption, instead of total data loss.
2. **Read scaling.** If your app does far more reads than writes (true
   for most apps), you can point read-only queries (reports, dashboards)
   at a replica, freeing up the primary to handle writes without
   contention.

### Streaming Replication in Postgres, Concretely

Postgres replicates by shipping its **WAL** ([Lesson 10](10-transactions-and-acid.md)
— the same Write-Ahead Log responsible for durability) to replica
servers, which replay it continuously:

```
Primary: COMMIT → written to WAL → WAL streamed to replicas → replicas replay WAL
```

This is the same mechanism that makes a single server crash-safe,
extended across the network to a second machine — the replica is
always just "a few WAL entries behind" the primary, not a totally
separate synchronization process.

**Why this connects back to your earlier LAN question**
([Lesson 19](19-network-access-and-lan-connections.md)): a replica is
just another Postgres server that needs network access to the primary
— the same `pg_hba.conf`/firewall concepts apply, just for
server-to-server replication traffic instead of an app connecting.

---

## Licensing — Why It Occasionally Actually Matters

| Engine | License | Practical implication |
|---|---|---|
| **PostgreSQL** | PostgreSQL License (very permissive, similar to MIT/BSD) | Use it anywhere, commercially, without restriction or fees — this is a big reason it's the modern default |
| **MySQL** | Dual-licensed: GPL (free) + a paid commercial license from Oracle | Free to use, but embedding MySQL inside a product you *sell* can trigger the need for a commercial license, depending on how it's distributed |
| **MariaDB** | Fully GPL (a MySQL fork created specifically to stay fully open, after Oracle acquired MySQL) | No dual-licensing ambiguity — created as a direct response to concerns about MySQL's Oracle ownership |
| **SQL Server** | Proprietary, paid | Real licensing costs, tied to Microsoft's ecosystem |
| **Oracle Database** | Proprietary, famously expensive at scale | Enterprise-only in practice, given cost |
| **SQLite** | Public domain | No licensing question at all — arguably the least restrictive of any engine here |

**Why this occasionally actually matters, concretely:** if you're
building a product you plan to sell or distribute (not just run as a
service you host yourself), MySQL's dual-license model is worth a
lawyer's five minutes; Postgres's permissive license means this
question basically never comes up. This is a real, if secondary, reason
many companies default to Postgres over MySQL today.

---

## Putting It Together — Revisiting the Lesson 00 Decision, With Mechanism

| Lesson 00 said... | The mechanism behind it |
|---|---|
| "Postgres is the safe modern default" | MVCC concurrency + permissive licensing + no vendor lock-in risk |
| "SQLite for embedded/single-user" | Single-writer locking model — genuinely can't handle concurrent multi-user writes |
| "Managed hosting handles backups/replication for you" | What "replication" concretely means: WAL streaming to standby servers, so a managed provider can auto-failover without you building that yourself |
| "MySQL has a huge ecosystem" | Also MVCC-based (InnoDB) — mechanically similar to Postgres for concurrency, main differences are feature depth and licensing |

---

## Key Takeaways

1. MVCC lets Postgres handle concurrent readers and writers without blocking each other, by keeping multiple row versions instead of locking — this is also *why* dead rows and `VACUUM` exist
2. SQLite's single-writer locking model is the concrete, mechanical reason it's wrong for multi-user backends, not just a rule of thumb
3. Replication streams the WAL to standby servers — the same mechanism behind crash durability, extended across machines, for failover and read scaling
4. PostgreSQL's permissive license removes a real business consideration that MySQL's dual-licensing can occasionally raise
5. Engine choice reputation ("Postgres is popular," "MySQL is fast") has actual mechanical reasons behind it — you can now reason from the mechanism, not just the reputation

---

## Next Lesson
[Lesson 22 — Real-World Schema Design Walkthrough](22-real-world-schema-walkthrough.md)
