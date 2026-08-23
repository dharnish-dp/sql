# Lesson 00 — Installing PostgreSQL & psql (macOS)

## Goal
Get a local PostgreSQL server running and connect to it with `psql` so you
have something real to practice queries against in the rest of the lessons.

## Prerequisites
[Lesson 00 (Orientation) — Choosing a Database & How It Fits Into an Application](00-choosing-a-database-and-how-it-fits.md)

## After This Lesson You Will Be Able To
- Explain the difference between the Postgres **server** and the `psql` **client**
- Install PostgreSQL on macOS via Homebrew
- Start and stop the server as a background service
- Create a database and connect to it with `psql`

---

## Client vs. Server — Two Different Things

**Server (`postgresql`)** — the actual database program. It stores your data
and runs in the background, listening on a port (default `5432`) for
connections. If it's not running, there is no database to talk to.

**Client (`psql`, from `libpq`)** — a command-line tool for *talking to* a
database. It stores nothing itself; it just connects to wherever the
database lives (your own machine, a cloud provider, a coworker's server,
Docker, etc.) and lets you type SQL.

| You want to... | Install |
|---|---|
| Connect to a database that already exists elsewhere (cloud, work server, Docker) | Client only — `brew install libpq` |
| Have your own local database to create and practice on | Full server — `brew install postgresql@16` (includes psql) |

For working through these lessons, install the **full server** — you need
somewhere to actually run the SQL.

---

## Install (Homebrew, macOS)

**1. Check if psql is already installed somewhere:**
```bash
which psql
psql --version
```

**2. Install PostgreSQL:**
```bash
brew install postgresql@16
```
This installs both the server and the `psql` client, and links `psql` onto
your `PATH` automatically.

**3. Start the server as a background service:**
```bash
brew services start postgresql@16
```
This launches the `postgres` server process in the background and registers
it to auto-start on login/reboot. It binds to `localhost:5432` — local-only
by default, not exposed to your network or the internet.

**4. Verify it's running:**
```bash
brew services list
lsof -i :5432
```

**5. Create a database and connect:**
```bash
createdb learning
psql learning
```
This drops you into an interactive `psql` prompt connected to the `learning`
database, ready for Lesson 01.

---

## Stopping the Server

```bash
brew services stop postgresql@16
```
This stops the server immediately and disables the auto-start-on-boot
registration. Run `brew services start postgresql@16` again whenever you
want to practice — Homebrew doesn't separate "pause" from "disable
autostart," `stop` does both.

---

## Troubleshooting

- **`lsof -i :5432` shows something already listening** — you likely have
  another Postgres install (e.g. Postgres.app) already using the port.
  Either use that existing install, or stop it before starting this one.
- **`psql: command not found`** — if you installed client-only via
  `brew install libpq`, it doesn't link automatically. Add it to your PATH:
  ```bash
  echo 'export PATH="/opt/homebrew/opt/libpq/bin:$PATH"' >> ~/.zshrc
  source ~/.zshrc
  ```
  (Use `/usr/local/opt/libpq/bin` on Intel Macs.)
- **Version mismatch with a remote server** — match your local `psql`
  major version to the server you're connecting to when possible, to avoid
  protocol/feature mismatches.

---

## Key Takeaways

1. The server stores data and runs in the background; `psql` is just a client that connects to it
2. `brew install postgresql@16` gives you both in one step
3. `brew services start/stop postgresql@16` controls whether the server is running
4. The server listens on `localhost:5432` by default — local-only, not network-exposed
5. `createdb` + `psql <dbname>` gets you into an interactive SQL prompt

---

## Next Lesson
[Lesson 01 — How Databases Work](01-how-databases-work.md)
