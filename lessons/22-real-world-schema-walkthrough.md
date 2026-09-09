# Lesson 22 — Real-World Schema Design Walkthrough

## Goal
Build one real application's database from scratch, end to end — every
concept from this entire course applied *in context*, in the order a
real project actually needs it, instead of as isolated syntax lessons.

## Prerequisites
Everything through [Lesson 19](19-network-access-and-lan-connections.md)
— this lesson is a synthesis, not new syntax.

## After This Lesson You Will Be Able To
- Design a schema from a plain-English product description, not a pre-made spec
- Recognize *when*, during real design work, each course concept actually gets reached for
- See how a design decision made early (schema shape) constrains or enables everything downstream (query performance, data integrity)
- Apply this same process to a database of your own

---

## Why This Lesson Exists

**Why you need this:** every other lesson taught one concept in
isolation — normalization, or indexes, or transactions. But real work
doesn't arrive one concept at a time; you get a vague product
requirement and have to figure out *which* concepts apply, *in what
order*, and *why*. This lesson simulates that — building a booking
system for a small gym/studio (classes, instructors, members,
bookings) from a one-paragraph brief, the way an actual project starts.

**The brief, as a stakeholder would actually give it to you:**

> "We need a system where members can book spots in fitness classes.
> Each class has an instructor, a time slot, and a max capacity. Members
> pay for a membership plan. We need to know who's coming to each class,
> prevent overbooking, and track revenue."

---

## Step 1 — Find the Entities (Nouns in the Brief)

Before any SQL, read the brief and underline the nouns that are clearly
*things the business tracks*: **members, classes, instructors, bookings,
membership plans.** These become candidate tables — this is the informal
version of the entity-relationship thinking from
[Lesson 11](11-database-design-normalization.md).

**Why this step matters and can't be skipped:** jumping straight to
`CREATE TABLE` without this step is how you end up with a `classes`
table that has an `instructor_name` text column instead of a proper
relationship — a normalization mistake baked in before you've written a
single constraint.

---

## Step 2 — Sketch Relationships Before Columns

```
instructors ──┐
              │ (one instructor teaches many classes)
           classes ──┐
              │       │ (one class has many bookings)
membership_plans      bookings
     │                    │
     │ (one plan has many members)  (many-to-many: members ↔ classes, via bookings)
  members ───────────────┘
```

**Two relationship types are visible immediately, both from
[Lesson 06](06-joins-complete.md) and [Lesson 11](11-database-design-normalization.md):**
- **One-to-many:** one instructor → many classes; one plan → many members
- **Many-to-many:** a member can book many classes, a class has many
  members — this needs a **junction table** (`bookings`), exactly the
  pattern from [Lesson 11](11-database-design-normalization.md)'s
  `enrollments` example and [Lesson 03](03-data-types-and-schema.md)'s
  dual-`CASCADE` explanation

---

## Step 3 — Write the Schema, Applying Normalization As You Go

```sql
CREATE TABLE instructors (
    id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name  TEXT NOT NULL,
    email TEXT NOT NULL UNIQUE
);

CREATE TABLE membership_plans (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name         TEXT NOT NULL,            -- 'Basic', 'Premium'
    monthly_price NUMERIC(8,2) NOT NULL CHECK (monthly_price >= 0)
);

CREATE TABLE members (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        TEXT NOT NULL,
    email       TEXT NOT NULL UNIQUE,
    plan_id     INT REFERENCES membership_plans(id) ON DELETE SET NULL,
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE classes (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name          TEXT NOT NULL,           -- 'Morning Yoga'
    instructor_id INT NOT NULL REFERENCES instructors(id) ON DELETE RESTRICT,
    starts_at     TIMESTAMPTZ NOT NULL,
    capacity      INT NOT NULL CHECK (capacity > 0)
);

CREATE TABLE bookings (
    member_id  INT NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    class_id   INT NOT NULL REFERENCES classes(id)  ON DELETE CASCADE,
    booked_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (member_id, class_id)
);
```

**Every design decision here traces back to a specific earlier lesson —
notice these as you read the schema, don't just accept them:**

- `email TEXT NOT NULL UNIQUE` on both `instructors` and `members` —
  [Lesson 03](03-data-types-and-schema.md)'s constraint syntax, applied
  because the brief implies each person is uniquely identifiable
- `plan_id ... ON DELETE SET NULL` — if a plan is discontinued, members
  shouldn't be deleted, just unlinked (the "customer keeps their orders,
  loses the link" pattern from [Lesson 03](03-data-types-and-schema.md))
- `instructor_id ... ON DELETE RESTRICT` — you should NOT be able to
  delete an instructor who still has classes scheduled; force cleanup
  first, same reasoning as `product_id` in the `order_items` example
- `bookings` with a **composite primary key** `(member_id, class_id)` —
  this single constraint does two jobs at once: it's the junction table
  *and* it automatically prevents the same member from booking the same
  class twice (a duplicate row would violate the PK) — a data integrity
  rule enforced by schema design alone, no application code needed
- `bookings` with **two independent `CASCADE`s** — exactly the
  `enrollments` pattern from [Lesson 03](03-data-types-and-schema.md)
  and [Lesson 11](11-database-design-normalization.md): delete a member
  or a class, their bookings disappear with them

---

## Step 4 — The Requirement That Needs More Than a Constraint: "Prevent Overbooking"

The brief says "prevent overbooking" — capacity is already a column, but
**nothing yet stops bookings from exceeding it.** This is where
declarative schema design (constraints) hits its limit, and you need
either application logic or a database-level check — a genuinely common
real-world fork in the road.

**Why a simple `CHECK` constraint can't solve this:** a `CHECK` on
`bookings` can only see the *current row* being inserted — it has no way
to count how many other bookings already exist for that class. This
needs a query, which means either:

**Option A — check before inserting, inside a transaction**
(applying [Lesson 10](10-transactions-and-acid.md) directly):
```sql
BEGIN;

SELECT capacity FROM classes WHERE id = 5;              -- e.g. capacity = 20
SELECT COUNT(*) FROM bookings WHERE class_id = 5;        -- e.g. 20 already booked

-- Application logic: if count >= capacity, ROLLBACK and reject the booking
-- Otherwise:
INSERT INTO bookings (member_id, class_id) VALUES (42, 5);

COMMIT;
```

**Why this needs a transaction, specifically:** without wrapping the
check-then-insert in one transaction, two members booking the last spot
*simultaneously* could both pass the capacity check before either
commits — a race condition. This is the exact "two transactions, each
individually correct, combined result wrong" shape from the
**serialization anomaly** in [Lesson 10](10-transactions-and-acid.md).
Using `SERIALIZABLE` isolation (or a `SELECT ... FOR UPDATE` row lock)
here isn't optional polish — it's the actual fix for the overbooking bug
the brief called out by name.

**Option B — a trigger, enforced at the database level** (a forward
pointer, not covered in depth in this course, but worth knowing exists):
a `BEFORE INSERT` trigger on `bookings` that counts existing bookings and
raises an error if capacity would be exceeded — enforced no matter what
application code does, closer in spirit to how `CHECK` constraints work,
just needing a query instead of a static condition.

---

## Step 5 — Indexes, Chosen By Actual Query Patterns (Not Guessing)

**Why you need this:** [Lesson 09](09-indexes-and-performance.md) said
"index foreign keys and frequently-filtered columns" — here's what that
means for *this specific schema*, reasoned from the app's real queries:

```sql
-- "Show a member their upcoming bookings" — filters bookings by member_id
CREATE INDEX idx_bookings_member_id ON bookings(member_id);

-- "Show a class's roster" — filters bookings by class_id
-- (already covered by the composite PRIMARY KEY (member_id, class_id)
--  ONLY if class_id were first — since it's (member_id, class_id), a
--  lookup by class_id alone does NOT use this index efficiently;
--  see Lesson 09's composite index column-order rule)
CREATE INDEX idx_bookings_class_id ON bookings(class_id);

-- "Show today's classes" — filters classes by time range
CREATE INDEX idx_classes_starts_at ON classes(starts_at);

-- Foreign keys not automatically indexed by Postgres (Lesson 09)
CREATE INDEX idx_classes_instructor_id ON classes(instructor_id);
CREATE INDEX idx_members_plan_id ON members(plan_id);
```

**Notice the composite-index catch, applying [Lesson 09](09-indexes-and-performance.md)
directly:** the `bookings` primary key is `(member_id, class_id)`, which
— per the composite index column-order rule — only helps lookups
*starting* with `member_id`. A query filtering by `class_id` alone (to
build a class roster) needs its *own* separate index, which is exactly
why `idx_bookings_class_id` is added explicitly above rather than
assumed to already exist.

---

## Step 6 — Answering "Track Revenue," With a View

The brief's last requirement, "track revenue," is a reporting need —
exactly what [Lesson 13](13-views-and-materialized-views.md) covers:

```sql
CREATE VIEW monthly_revenue AS
SELECT
    DATE_TRUNC('month', m.joined_at) AS month,
    SUM(p.monthly_price) AS revenue
FROM members m
JOIN membership_plans p ON p.id = m.plan_id
GROUP BY DATE_TRUNC('month', m.joined_at);
```

**Why a view here, not just a saved query:** this calculation will be
run repeatedly (a dashboard, a monthly report) — wrapping it as a view
means the business logic for "what counts as revenue" lives in one
place, not copy-pasted across every report that needs it.

---

## The Point of This Whole Exercise

Look back at what just happened: a two-sentence product brief turned
into entity identification ([Lesson 11](11-database-design-normalization.md)),
relationship modeling and junction tables ([Lesson 06](06-joins-complete.md)),
constraint design with deliberate `CASCADE`/`RESTRICT` choices
([Lesson 03](03-data-types-and-schema.md)), a genuine concurrency bug
requiring transactions and isolation levels ([Lesson 10](10-transactions-and-acid.md)),
targeted indexing based on actual query patterns
([Lesson 09](09-indexes-and-performance.md)), and a view for recurring
reporting ([Lesson 13](13-views-and-materialized-views.md)).

**None of these concepts were reached for because a lesson said to —
each one was reached for because the requirement demanded it.** That's
the skill this whole course was building toward: not memorizing syntax,
but recognizing *which tool a real requirement is asking for.*

---

## Try It Yourself

Take a different one-paragraph brief — a food delivery app, a library
system, a ticket-booking site — and run it through these same six steps
yourself before looking at any reference schema. Compare where your
instincts matched this walkthrough and where they didn't; the gaps are
exactly what's worth re-reading in earlier lessons.

---

## Key Takeaways

1. Real schema design starts from a plain-English requirement, not a spec — find the nouns, then the relationships, then the columns
2. Many-to-many relationships need a junction table with a composite primary key — which doubles as a duplicate-prevention constraint for free
3. Some requirements ("prevent overbooking") can't be solved by constraints alone — they need transactions, and sometimes stricter isolation levels, to close real race conditions
4. Index choices should trace back to actual query patterns, not a generic "index the foreign keys" rule applied blindly — check composite key column order specifically
5. Recurring reporting needs belong in a view, so the business logic for "what counts" lives in one place
6. The actual skill is recognizing which concept a requirement is asking for — not recalling syntax in isolation
