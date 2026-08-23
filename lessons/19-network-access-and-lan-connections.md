# Lesson 19 — Network Access & LAN Connections

## Goal
Understand exactly what blocks another machine from connecting to your
local PostgreSQL server, and what to change (carefully) to allow it —
covering the three independent layers that all have to agree.

## Prerequisites
[Lesson 17 — Roles, Users & Access Management](17-roles-users-and-access-management.md)

## After This Lesson You Will Be Able To
- Explain why `localhost` blocks connections from other machines by default
- Identify the three separate layers that must all allow a connection
- Safely open LAN access for testing, without exposing the server to the internet
- Know what changes for real production hosting instead of a local machine

---

## Why a Random Laptop on Your WiFi Can't Connect Right Now

Even though your sister's laptop is on the *same* WiFi network as yours,
three independent things currently block it — and all three exist by
default specifically to keep a local dev database private:

| Layer | What it controls | Current default |
|---|---|---|
| `postgresql.conf` → `listen_addresses` | Which network interfaces the server *listens* on at all | `localhost` only (loopback, `127.0.0.1`) |
| `pg_hba.conf` | Which IPs/subnets are *allowed* to authenticate, and how | Only allows `127.0.0.1` / `::1` (your own machine) |
| macOS Firewall | Whether incoming connections on port 5432 reach the OS at all | Blocks unsolicited incoming connections |

All three have to say "yes" for a remote connection to succeed. Missing
any one of them blocks it entirely — which is why just changing
`listen_addresses` alone isn't enough.

---

## Step 1 — Let the Server Listen on the Network

Find the config file:
```bash
psql -U ddp -d postgres -c "SHOW config_file;"
```

Change:
```
listen_addresses = 'localhost'
```
to:
```
listen_addresses = '*'
```
(`*` means "all network interfaces," not just loopback.)

---

## Step 2 — Allow Specific IPs in `pg_hba.conf`

Same directory as `postgresql.conf`. Find your WiFi subnet first:
```bash
ipconfig getifaddr en0
```
A typical home router gives you something like `192.168.1.42` — assume a
`/24` subnet (`192.168.1.0/24`) unless you know otherwise.

Add a line:
```
host    mydb    app_user    192.168.1.0/24    scram-sha-256
```

**Never write `trust` on this line.** `trust` means "let anyone from this
subnet in with zero password" — fine for `127.0.0.1` (only you can reach
that), but on a LAN rule it means anyone on the WiFi (a neighbor on a
shared network, a compromised IoT device) gets in freely. Always require a
real password method (`scram-sha-256`) for anything beyond loopback.

This is also why you connect app roles specifically ([Lesson 17](17-roles-users-and-access-management.md))
rather than opening this rule for your own superuser role — scope the
network exposure to exactly the role and database that needs it.

---

## Step 3 — Allow the Port Through the Firewall

**System Settings → Network → Firewall** → allow incoming connections for
`postgres`, or add an explicit rule for port 5432.

If you skip this step, the first two changes won't matter — the OS drops
the connection attempt before Postgres ever sees it.

---

## Step 4 — Restart and Connect

```bash
brew services restart postgresql@16
```

Find your machine's LAN IP (what the other machine connects to):
```bash
ipconfig getifaddr en0
```

From the other machine:
```bash
psql -h <your-private-ip> -p 5432 -U app_user -d mydb
```
She'll be prompted for `app_user`'s password.

---

## What This Is — and Isn't — Good For

**Good for:** testing a local app from a second device, letting a teammate
hit your dev database temporarily, learning how network auth actually works.

**Not what real deployments do.** A production database:
- Lives on a managed host (AWS RDS, Cloud SQL, a dedicated server) — not a
  personal laptop that sleeps, changes IP, or sits behind a home router
- Restricts access to specific application server IPs, not "the whole
  office WiFi subnet"
- Almost always requires SSL/TLS on the connection (`sslmode=require` or
  stronger) — LAN testing usually skips this, production never should
- Sits behind a VPC/firewall with much narrower rules than "allow this
  whole subnet"

Treat LAN access as a temporary, deliberate, undo-able thing — flip
`listen_addresses` back to `localhost` and remove the `pg_hba.conf` line
when you're done testing.

---

## Key Takeaways

1. Three independent layers gate a remote connection: `listen_addresses`, `pg_hba.conf`, and the OS firewall — all three must allow it
2. `localhost`/loopback only accepts connections from the same machine, by design
3. Never use `trust` auth on any rule beyond `127.0.0.1`/`::1` — always require a real password on network rules
4. Scope LAN rules to a specific app role and database, not your superuser role
5. LAN access is a testing convenience, not how real applications get deployed — production uses managed hosting with SSL and narrow IP allowlists
