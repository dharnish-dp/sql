# Lesson 11 — Database Design & Normalization

## Goal
Learn to design databases from scratch — from requirements to ERD to
normalized schema. This is the skill that separates strong engineers
from SQL query writers.

## Prerequisites
- [Lesson 10](10-transactions-and-acid.md) — transactions

## After This Lesson You Will Be Able To
- Identify entities, attributes, and relationships from requirements
- Draw an Entity-Relationship Diagram (ERD)
- Apply 1NF, 2NF, 3NF, and BCNF normalization rules
- Recognize when to denormalize for performance
- Design common patterns: many-to-many, self-referencing, soft delete

---

## What Is Database Design?

**Why you need this:** every earlier lesson assumed a schema already
existed — you queried `orders`, `customers`, `products` without ever
deciding *why* those tables look the way they do. This lesson is about
that decision itself: **before you can write a `JOIN` ([Lesson 06](06-joins-complete.md))
or a `CHECK` constraint ([Lesson 03](03-data-types-and-schema.md)), someone
has to decide what tables and columns exist in the first place** — get
that wrong, and every query downstream becomes harder than it should be.

Database design is the process of deciding:
1. **What entities** to store (tables)
2. **What attributes** each entity has (columns)
3. **How entities relate** to each other (foreign keys)
4. **What rules** the data must satisfy (constraints)

Bad design causes: redundancy, anomalies, difficult queries, poor performance.
Good design causes: one source of truth, easy queries, maintainability.

---

## Step 1: Identify Entities and Attributes

**Why you need this:** before any `CREATE TABLE`, you need to know
*what* tables to even create. This step is just reading a plain-English
requirement and pulling out the nouns — the same informal move used to
kick off the full walkthrough in [Lesson 22](22-real-world-schema-walkthrough.md).

From a requirements document, extract nouns (entities) and their properties (attributes).

**Requirements example: "We need to manage an online library where members can borrow books written by authors and tracked by librarians."**

**Entities (nouns):**
- Member
- Book
- Author
- Librarian
- Loan (a borrowing event)

**Attributes (properties of each entity):**
```
Member:    id, name, email, phone, membership_date, is_active
Book:      id, title, isbn, publication_year, genre, description, copies_available
Author:    id, name, bio, nationality
Librarian: id, name, employee_id, hire_date
Loan:      id, member_id, book_id, librarian_id, borrow_date, due_date, return_date, fine_amount
```

---

## Step 2: Identify Relationships

**Why you need this:** entities alone (Step 1) just give you a pile of
disconnected tables. Relationships are what decide *which foreign keys*
you'll need, and — critically — whether you'll need an extra junction
table at all (covered concretely later in this lesson, and already seen
in practice with `order_items` in [Lesson 03](03-data-types-and-schema.md)
and `enrollments`/self-joins in [Lesson 06](06-joins-complete.md)).

Relationships are the verbs connecting entities:

```
Member ─── borrows ──► Book        (Many-to-Many: through Loan)
Author ─── writes ───► Book        (Many-to-Many: one author writes many books, book has multiple authors)
Librarian ── manages ─► Loan       (One-to-Many: one librarian processes many loans)
Book ─── has ────────► Author      (through junction table book_authors)
```

**Relationship cardinality — three shapes, and why each needs a
different structure:**

- **One-to-One (1:1):** One person has one passport. In SQL, this
  usually just means putting a `UNIQUE` foreign key on one of the two
  tables (or storing it on the same row entirely) — there's no need for
  a junction table, since neither side is ever "many."
- **One-to-Many (1:N):** One customer has many orders. This is a plain
  foreign key on the "many" side (`orders.customer_id`) — you've been
  writing this relationship since [Lesson 03](03-data-types-and-schema.md)
  without naming it.
- **Many-to-Many (M:N):** Students enroll in courses; a course has many
  students, and a student takes many courses. **Neither table can hold
  a simple foreign key to the other** — a single column can't point at
  multiple rows. This is *why* a separate **junction table** is
  required (worked through in detail later in this lesson).

---

## Entity-Relationship Diagram (ERD)

**Why you need this:** Steps 1 and 2 above produced a list of entities
and a list of relationships in your head — an ERD is just drawing both
at once, so you (and anyone reviewing the design) can see the whole
shape before writing a single `CREATE TABLE`. Read the diagram below
the same way you'd read the `\d table_name` output from
[Lesson 01](01-how-databases-work.md): each box is a table, `(PK)` marks
its primary key, `(FK)` marks a foreign key, and arrows show which
column points at which.

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│  authors │     │ book_authors │     │   books  │
├──────────┤     ├──────────────┤     ├──────────┤
│ id  (PK) │──┐  │ book_id (FK) │  ┌──│ id  (PK) │
│ name     │  └─►│author_id(FK)│◄─┘  │ title    │
│ bio      │     └──────────────┘     │ isbn     │
│ nation.. │                          │ genre    │
└──────────┘                          │ copies   │
                                      └──────────┘
                                           │
                                      ┌────┴──────┐
                                      │   loans   │
                                      ├───────────┤
                                      │ id (PK)   │
              ┌──────────┐            │ book_id   │
              │ members  │            │ member_id │
              ├──────────┤◄───────────│ libr._id  │
              │ id (PK)  │            │ borrow_dt │
              │ name     │            │ due_date  │
              │ email    │            │ return_dt │
              └──────────┘            └───────────┘
```

---

## The Normal Forms

**Why you need this:** normalization is the actual *rulebook* behind a
question you've been answering by instinct since [Lesson 03](03-data-types-and-schema.md)
— "should this be its own table, or a column?" Each normal form is just
a specific, checkable test for one particular kind of redundancy — think
of them as a sequence of code-review questions you run against a
draft schema, each one catching a different mistake.

Normalization is the process of structuring a database to reduce redundancy
and improve data integrity. Each "normal form" fixes a specific class of problem.

---

## First Normal Form (1NF)

**Why you need this:** this is the most basic check — "can I actually
query this cleanly with SQL at all?" A table that fails 1NF isn't
subtly inefficient, it's actively broken for the relational model
([Lesson 01](01-how-databases-work.md)) — you literally cannot `JOIN`
or filter on the offending column properly.

**Rule:** Each column contains atomic (indivisible) values. No repeating groups.

### Violation — Non-atomic values:
```
books table (BAD):
┌────┬────────────────┬───────────────────────────┐
│ id │ title          │ authors                   │
├────┼────────────────┼───────────────────────────┤
│  1 │ Clean Code     │ Robert Martin             │
│  2 │ Design Patterns│ Gang of Four, Ralph,... │  ← comma-separated authors in one cell
└────┴────────────────┴───────────────────────────┘
```

**Why this specific example is broken:** the `authors` column crams
*multiple values* into one cell as a comma-separated string. SQL has no
built-in way to search, filter, or join on "one item inside a
comma-separated list" — `WHERE authors = 'Ralph'` would never match
`'Gang of Four, Ralph, ...'`, since as far as Postgres is concerned,
that whole string is one single, opaque value. You'd be stuck writing
fragile string-parsing (`LIKE '%Ralph%'`) instead of a real, indexable
comparison.

**Problem:** Can't search for "books by Ralph" without string parsing. Can't JOIN.

### Fix for 1NF — Apply the Junction Table Pattern

```sql
-- Move authors to a separate table and use a junction table
CREATE TABLE authors (id SERIAL PRIMARY KEY, name TEXT NOT NULL);
CREATE TABLE books   (id SERIAL PRIMARY KEY, title TEXT NOT NULL);
CREATE TABLE book_authors (
    book_id   INT REFERENCES books(id),
    author_id INT REFERENCES authors(id),
    PRIMARY KEY (book_id, author_id)
);
```

**Why this fixes it:** this is exactly the many-to-many shape from Step
2 above — one book can have several authors, one author can write
several books, so neither side can hold the other as a simple column.
`book_authors` gives each `(book, author)` pairing its own row. Now
"books by Ralph" is a real, indexable query:
```sql
SELECT b.title
FROM books b
JOIN book_authors ba ON ba.book_id = b.id
JOIN authors a ON a.id = ba.author_id
WHERE a.name = 'Ralph';
```

Sample output:
| title |
|---|
| Design Patterns |

### Another 1NF Violation — Repeating Columns:
```
orders table (BAD):
┌────┬─────────────┬──────────┬──────────┬──────────┐
│ id │ customer_id │ product1 │ product2 │ product3 │
├────┼─────────────┼──────────┼──────────┼──────────┤
│  1 │      1      │ Laptop   │ Mouse    │ NULL     │
└────┴─────────────┴──────────┴──────────┴──────────┘
```

**Why this is broken, concretely:** what happens on order #1's *fourth*
product? There's no `product4` column — you'd have to `ALTER TABLE`
just to sell someone a 4-item order. And `WHERE product2 = 'Mouse' OR
product1 = 'Mouse' OR product3 = 'Mouse'` is what searching "who ordered
a Mouse" turns into — every repeating column has to be checked
individually, and the query grows every time you add a `productN` column.

**Fix:** Use a separate `order_items` table (one row per item) — exactly
the structure you've been using since [Lesson 03](03-data-types-and-schema.md),
where an order with 5 products is simply 5 rows, and an order with 1
product is 1 row. No column limit, no repeated `OR` chains.

---

## Second Normal Form (2NF)

**Why you need this:** 1NF only checked "is each cell one atomic value."
2NF is the next, more specific check, and it **only applies when a
table has a composite primary key** (two or more columns together, like
`order_items`'s `(order_id, product_id)` from [Lesson 03](03-data-types-and-schema.md))
— if your table has a single-column primary key, 2NF is automatically
satisfied and you can skip straight to 3NF.

**Rule:** Must be in 1NF. Every non-key column must depend on the WHOLE primary key
(not just part of it).

### Violation:
```
order_items table (BAD):
┌──────────┬────────────┬──────────┬──────────────────┬──────────────┐
│ order_id │ product_id │ quantity │ product_name      │ product_price│
├──────────┼────────────┼──────────┼──────────────────┼──────────────┤
│     1    │     2      │    3     │ Wireless Mouse    │    29.99     │
│     2    │     2      │    1     │ Wireless Mouse    │    29.99     │
└──────────┴────────────┴──────────┴──────────────────┴──────────────┘
```

**Read "depends on" as "if I know X, do I already know Y?"** The
primary key here is `(order_id, product_id)` together. Ask, for each
other column: do I need *both* parts of the key to know its value, or
just one?

- `quantity` — genuinely needs both: "3 units" only means something for
  *this specific order's* copy of *this specific product*. This column
  is fine.
- `product_name` and `product_price` — these only need `product_id`.
  Notice both rows above have `product_id = 2`, and both show the exact
  same name and price — the `order_id` played no role in determining
  them at all. That's the violation: a column depending on only *part*
  of a composite key, not the whole thing.

**Problem:** `product_name` and `product_price` depend only on `product_id`,
not on the composite key `(order_id, product_id)`. Data is duplicated.
If the price changes, you must update it in every row.

**Concretely, what breaks if you leave it this way:** raise the Wireless
Mouse's price to $34.99, and you'd have to find and update it in *every
single order row* that ever included it — miss one, and you now have
two different "true" prices for the same product, with no way to tell
which one is correct.

**Fix for 2NF:**
```sql
-- Move product info to its own table
CREATE TABLE products (
    id    SERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    price NUMERIC(10,2) NOT NULL
);

CREATE TABLE order_items (
    order_id   INT REFERENCES orders(id),
    product_id INT REFERENCES products(id),
    quantity   INT NOT NULL,
    unit_price NUMERIC(10,2) NOT NULL,  -- snapshot price at time of order
    PRIMARY KEY (order_id, product_id)
);
-- product.name/price are in the products table (not duplicated)
-- unit_price captures the price at time of order (intentional snapshot)
```

**Why `unit_price` still exists here, and isn't itself a violation:**
this looks like it's repeating `products.price`, but it's capturing a
*different fact* — "what this product cost *at the time this specific
order was placed*," which is intentionally allowed to differ from the
product's *current* price. This is a deliberate, documented exception —
covered properly in the Denormalization section below — not a mistake.

---

## Third Normal Form (3NF)

**Why you need this:** 2NF only concerned composite keys. 3NF applies
to **every** table, and catches a different, more common mistake: one
non-key column silently depending on *another non-key column*, rather
than the table's key at all.

**Rule:** Must be in 2NF. No transitive dependencies — non-key columns must depend
ONLY on the primary key, not on other non-key columns.

### Violation:
```
employees table (BAD):
┌────┬──────────┬────────────┬──────────────────────┬──────────────┐
│ id │ name     │ dept_id    │ dept_name            │ dept_location│
├────┼──────────┼────────────┼──────────────────────┼──────────────┤
│  1 │ Alice    │    1       │ Engineering          │ Floor 3      │
│  2 │ Bob      │    1       │ Engineering          │ Floor 3      │
│  3 │ Carol    │    2       │ Sales                │ Floor 2      │
└────┴──────────┴────────────┴──────────────────────┴──────────────┘
```

**"Transitive dependency" unpacked:** the primary key here is just
`id`. Ask the same question as before — does `dept_name` need `id` to
be known, or does it actually only need `dept_id`? Alice and Bob have
*different* `id`s but the *same* `dept_name` — proof that `dept_name`
is really riding along on `dept_id`, not on `id` directly. That chain —
`id` → `dept_id` → `dept_name` — is what "transitive" means: `dept_name`
depends on the key only *indirectly*, through another non-key column.

**Problem:** `dept_name` and `dept_location` depend on `dept_id`, not on `id`.
If Engineering moves floors, you must update every Engineering employee row.

**Concretely, what breaks:** Engineering relocates from Floor 3 to Floor
5. Update Alice's row but forget Bob's, and now the same department has
two different floors on record, depending on which employee you look
at — an inconsistency that a well-designed schema should make
impossible, not just unlikely.

**Fix for 3NF:**
```sql
CREATE TABLE departments (
    id       SERIAL PRIMARY KEY,
    name     TEXT NOT NULL,
    location TEXT
);

CREATE TABLE employees (
    id      SERIAL PRIMARY KEY,
    name    TEXT NOT NULL,
    dept_id INT REFERENCES departments(id)
    -- dept_name and dept_location are in the departments table
);
```
Now "Engineering moved to Floor 5" is a **single-row update** to
`departments`, and every employee referencing `dept_id = 1` picks up the
new location automatically the next time you `JOIN` — one source of
truth, exactly the goal stated at the top of this lesson.

---

## Boyce-Codd Normal Form (BCNF)

**Why you need this:** 3NF handles the overwhelming majority of
real-world cases. BCNF exists for a narrower edge case — tables with
**more than one candidate key** (more than one column or column-set that
could independently serve as a unique identifier) where 3NF's rules
don't quite catch every redundancy. You will rarely need this in
practice, but it's worth recognizing the shape.

**Rule:** Must be in 3NF. Every determinant must be a candidate key.
More strict than 3NF — handles edge cases with multiple overlapping keys.

```
course_registration (BAD — assuming a student can take each course with one teacher,
                    and each teacher teaches one course):
┌────────────┬────────────┬────────────┐
│ student_id │ course_id  │ teacher_id │
│     1      │     A      │    T1      │
│     1      │     B      │    T2      │
│     2      │     A      │    T1      │

-- If teacher_id determines course_id (T1 only teaches course A),
-- then teacher_id is a determinant but not a candidate key.
-- Fix: separate teacher-course assignments into another table.
```

**In plain terms:** knowing `teacher_id` alone is enough to tell you the
`course_id` (since each teacher only teaches one course in this
scenario) — `teacher_id` is "determining" another column's value, the
same smell as the 3NF violation above, but here it's happening between
two columns that are *both* part of describing the relationship, not a
clean non-key-depends-on-key case. The fix is the same instinct as
always: pull the teacher-course pairing into its own table, so that
fact is stored exactly once.

In practice, 3NF is sufficient for most applications.

---

## Denormalization — When to Break the Rules

**Why you need this:** every normal form above pushed you toward
*fewer* copies of the same fact. Denormalization is the deliberate,
informed choice to go the other way — accepting some redundancy on
purpose, in exchange for faster reads. This isn't "doing normalization
wrong" — it's a second, separate design decision made *after* you
understand what normalization would cost you.

Normalization is great for write integrity. But sometimes you need
to denormalize for read performance.

### When to denormalize:
1. Frequently read, rarely written data
2. Reports that JOIN many tables (consider materialized views instead — see [Lesson 13](13-views-and-materialized-views.md))
3. Aggregated values that are expensive to recompute

```sql
-- Normalized: customer's total order count is computed each time
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;
```
This is always *correct* — it can never drift out of sync, because it's
recalculated fresh every time. The tradeoff: on a customer with 10,000
orders, or a page that runs this for every customer at once, that's
real, repeated aggregation work ([Lesson 09](09-indexes-and-performance.md)
territory) every single time the page loads.

```sql
-- Denormalized: store total_orders on the customer row
ALTER TABLE customers ADD COLUMN total_orders INT DEFAULT 0;
-- Update it with a trigger (Lesson 14) every time an order is inserted/deleted
```
**Pro:** reading `customers.total_orders` is an instant single-row
lookup — no aggregation, no scanning `orders` at all. **Con, and this is
the real cost of denormalizing:** this column can now silently become
*wrong* if something updates `orders` without also updating
`total_orders` — a bulk data import, a manual `DELETE`, a bug in the
trigger. Normalized data is self-consistent by construction; denormalized
data requires ongoing discipline (usually a trigger) to stay correct.

**The rule of thumb this leads to:** default to normalized. Denormalize
only a specific column, for a specific measured performance reason, and
document *why* right next to it — exactly like the example already
built into this course:

```sql
-- When denormalization is intentional and documented:
-- order_items.unit_price is a denormalized snapshot of products.price at time of order
-- This is CORRECT: we want historical price, not current price
```
This one isn't even really "redundant" in the bad sense — it's storing
a *fact that will diverge on purpose* (the historical price at time of
sale vs. today's price), which is a different, legitimate reason to
duplicate a value, distinct from "we didn't want to write a `JOIN`."

---

## Common Design Patterns

**Why you need this:** the normal forms above are *checks* you run
against a schema. This section is the flip side — recurring *shapes*
that come up so often across real applications that it's worth
recognizing them by name, so you reach for the right structure
immediately instead of re-deriving it from scratch each time.

### Many-to-Many Relationship (Junction Table)

**Why you need this:** this is the direct application of the M:N
cardinality flagged back in Step 2 — neither `students` nor `courses`
can hold a simple foreign key to the other (a student takes *many*
courses, a course has *many* students), so a third table is structurally
required, not optional.

```sql
-- Students and Courses (many-to-many)
CREATE TABLE students (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE courses (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    code TEXT UNIQUE NOT NULL
);

CREATE TABLE enrollments (
    student_id INT REFERENCES students(id) ON DELETE CASCADE,
    course_id  INT REFERENCES courses(id)  ON DELETE CASCADE,
    enrolled_at TIMESTAMPTZ DEFAULT NOW(),
    grade       CHAR(2),
    PRIMARY KEY (student_id, course_id)
);
-- The junction table can have its own columns (enrolled_at, grade)
```

**Two things worth noticing that are easy to miss:**
- **The composite primary key does double duty.** `(student_id,
  course_id)` both identifies each enrollment row *and* automatically
  prevents the same student from enrolling in the same course twice —
  a duplicate insert would violate the primary key, no application code
  needed to enforce that rule.
- **A junction table can carry its own data** (`enrolled_at`, `grade`)
  — it's not just a bare pairing of two IDs. This is exactly why it
  needs to be a real table, not just some Postgres-specific array
  trick — a `grade` genuinely belongs to *this specific* student-course
  pairing, nowhere else.

Sample output, joining back to real names:
```sql
SELECT s.name, c.name AS course, e.grade
FROM enrollments e
JOIN students s ON s.id = e.student_id
JOIN courses c ON c.id = e.course_id;
```
| name | course | grade |
|---|---|---|
| Alice | Databases 101 | A |
| Bob | Databases 101 | B |
| Alice | Web Dev | A- |

### Self-Referencing Table (Hierarchy)

```sql
-- Categories with subcategories
CREATE TABLE categories (
    id        SERIAL PRIMARY KEY,
    name      TEXT NOT NULL,
    parent_id INT REFERENCES categories(id)  -- NULL for top-level
);

INSERT INTO categories (name, parent_id) VALUES
    ('Electronics', NULL),           -- id=1, top-level
    ('Computers',   1),              -- id=2, under Electronics
    ('Laptops',     2),              -- id=3, under Computers
    ('Gaming',      1);              -- id=4, under Electronics

-- Query the hierarchy using recursive CTE (Lesson 07)
```

**Querying it, applying the exact `org_chart` recursive CTE pattern from
[Lesson 07](07-subqueries-and-ctes.md):**

```sql
WITH RECURSIVE category_tree AS (
    SELECT id, name, parent_id, 0 AS depth, name AS path
    FROM categories
    WHERE parent_id IS NULL          -- base case: top-level categories

    UNION ALL

    SELECT c.id, c.name, c.parent_id, ct.depth + 1, ct.path || ' → ' || c.name
    FROM categories c
    JOIN category_tree ct ON c.parent_id = ct.id   -- recursive case: children of the previous level
)
SELECT REPEAT('  ', depth) || name AS indented_name, depth, path
FROM category_tree
ORDER BY path;

-- Output:
-- Electronics                          | 0 | Electronics
--   Computers                          | 1 | Electronics → Computers
--     Laptops                          | 2 | Electronics → Computers → Laptops
--   Gaming                             | 1 | Electronics → Gaming
```

#### A Common Misconception — "Shouldn't the Primary Key and Foreign Key Be the Same?"

**No — they're always two separate columns, even in a self-reference.**
What has to match is the **data type** (`id` is effectively `INT`,
`parent_id` is `INT` — comparable), never the column itself.

```sql
id        SERIAL PRIMARY KEY,          -- this row's OWN identity
parent_id INT REFERENCES categories(id)  -- a pointer to a DIFFERENT row's id
```

Look at the "Computers" row from the data above:
```
id | name      | parent_id
2  | Computers | 1
```
This single row has **both** an `id` (`2` — its own identity) **and** a
`parent_id` (`1` — pointing to a *different* row, Electronics). If `id`
and `parent_id` had to be "the same," this row couldn't hold two
different values at once — it needs one number to say who it is, and a
separate number to say who its parent is.

**The only thing "self-referencing" actually means:** normally a foreign
key points to a *different* table (`order_items.order_id` → `orders.id`).
Here, it points back to its **own** table (`categories.parent_id` →
`categories.id`). That's the entire meaning of "self-referencing" — not
that any columns are the same, just that the FK's target table happens
to be the table it's defined in.

### Audit Trail (Immutable History)

**Why you need this:** normal tables only show you the *current* state
— `UPDATE orders SET status = 'shipped'` overwrites the old value
entirely (setting aside MVCC's internal dead-row versions from
[Lesson 09](09-indexes-and-performance.md), which aren't meant for
human-readable history). If you need to answer "who changed this, and
what did it used to say," you need a table specifically for that.

```sql
-- Keep full history of changes to orders
CREATE TABLE order_audit (
    id          SERIAL PRIMARY KEY,
    order_id    INT NOT NULL,
    changed_at  TIMESTAMPTZ DEFAULT NOW(),
    changed_by  TEXT,
    field_name  TEXT,
    old_value   TEXT,
    new_value   TEXT
);
-- Populated by a trigger (Lesson 14)
```

**Why one row per *field change*, not one row per update:** this shape
lets you log "status changed from 'pending' to 'shipped'" as one clean
record, separate from "total changed from 99.98 to 89.98" — even if both
happened in the same `UPDATE` statement. Querying "show me every change
ever made to order #5" becomes a simple filter:

```sql
SELECT changed_at, field_name, old_value, new_value
FROM order_audit
WHERE order_id = 5
ORDER BY changed_at;
```
| changed_at | field_name | old_value | new_value |
|---|---|---|---|
| 2024-03-01 10:00 | status | pending | shipped |
| 2024-03-02 14:30 | status | shipped | delivered |

### Soft Delete

**Why you need this:** a real `DELETE` is permanent — the row, and
anything it was needed for (an audit trail, an old order referencing a
now-gone customer), is simply gone. "Soft delete" is a convention for
marking something as deleted *without* physically removing it, when you
need to preserve history or allow undoing.

```sql
-- Instead of DELETE, set deleted_at
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    email      TEXT UNIQUE NOT NULL,
    deleted_at TIMESTAMPTZ  -- NULL = active, timestamp = deleted
);
```

**The convention itself:** "deleting" a user is just an `UPDATE`, not a
`DELETE`:
```sql
UPDATE users SET deleted_at = NOW() WHERE id = 3;
```
The row is still physically there — every foreign key pointing at
`users.id = 3` still resolves correctly, and nothing else in the
database needs to know this user is "gone."

```sql
-- "Active" query
SELECT * FROM users WHERE deleted_at IS NULL;
```
**Why this pattern is genuinely risky if you're not careful:** every
single query against `users` from now on has to remember to add `WHERE
deleted_at IS NULL`, or "deleted" users start silently showing up again
in results. Forgetting this filter even once in a report is a real,
common bug.

```sql
-- View to make it transparent
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;
```
**This view is the actual fix for that risk** — applying
[Lesson 13](13-views-and-materialized-views.md) directly: put the
`WHERE deleted_at IS NULL` filter in exactly *one* place (the view
definition), and every query that should only see active users queries
`active_users` instead of `users` directly — the filter can no longer be
forgotten, because it's baked into the thing being queried.

### Polymorphic Association (Tagging, Comments)

**Why you need this:** sometimes one table (`comments`) needs to attach
to *multiple different* parent tables (`posts` **or** `products`) —
and a single foreign key column can only ever point at one specific
table. This pattern is how to model "this comment belongs to *some*
other row, but which table that row lives in varies per comment."

```sql
-- Comments that can belong to posts OR products
CREATE TABLE comments (
    id           SERIAL PRIMARY KEY,
    entity_type  TEXT NOT NULL CHECK (entity_type IN ('post', 'product')),
    entity_id    INT NOT NULL,
    content      TEXT NOT NULL,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);
-- Can't use a FK here (polymorphic) — enforce in application layer
```

**Walk through why this actually works, and why it's risky:**
`entity_type` says *which table* this comment belongs to; `entity_id`
holds the id *within that table*. So `entity_type='post', entity_id=5`
means "this comment is on `posts.id = 5`." **The real cost, spelled
out:** `REFERENCES posts(id)` from [Lesson 03](03-data-types-and-schema.md)
is what normally *guarantees* the referenced row exists — Postgres
enforces it automatically. `entity_id` here has no such guarantee: it's
just a plain `INT`, so nothing stops you from inserting `entity_type =
'post', entity_id = 99999` where post 99999 doesn't exist. That
integrity check has to be written and maintained in application code
instead — a real tradeoff, not a free convenience.

```sql
-- Alternative: separate tables (cleaner but more tables)
CREATE TABLE post_comments    (id SERIAL PK, post_id INT FK, content TEXT, ...);
CREATE TABLE product_comments (id SERIAL PK, product_id INT FK, content TEXT, ...);
```
**The tradeoff in the other direction:** this version gets real foreign
keys back (Postgres enforces the reference again), but now "get all
comments regardless of what they're on" needs a `UNION` ([Lesson 12](12-advanced-queries.md))
across both tables instead of one simple query — more structural
safety, less querying convenience. Neither option is universally
"correct" — which one to pick depends on whether you'd rather enforce
integrity at the database level or keep querying simple.

---

## Full Design Example — Blog Platform

**Why you need this:** this is every pattern above, applied together in
one realistic schema — the same kind of end-to-end exercise carried
further in [Lesson 22](22-real-world-schema-walkthrough.md). As you read
it, try to name which pattern each table is applying *before* reading
the inline comments — that's the actual skill this lesson was building.

**Requirements:** Users write posts. Posts have tags. Users can comment on posts.
Users can follow other users.

```sql
-- Users
CREATE TABLE users (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username   TEXT NOT NULL UNIQUE CHECK (LENGTH(username) BETWEEN 3 AND 30),
    email      TEXT NOT NULL UNIQUE,
    bio        TEXT,
    avatar_url TEXT,
    is_active  BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

-- Posts
CREATE TABLE posts (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    author_id    BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title        TEXT NOT NULL CHECK (LENGTH(title) BETWEEN 1 AND 200),
    slug         TEXT NOT NULL UNIQUE,         -- URL-friendly title
    content      TEXT,
    status       TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'archived')),
    view_count   INT NOT NULL DEFAULT 0 CHECK (view_count >= 0),
    published_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Tags
CREATE TABLE tags (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    slug TEXT NOT NULL UNIQUE
);

-- Post-Tags (many-to-many)
CREATE TABLE post_tags (
    post_id INT REFERENCES posts(id) ON DELETE CASCADE,
    tag_id  INT REFERENCES tags(id)  ON DELETE CASCADE,
    PRIMARY KEY (post_id, tag_id)
);

-- Comments
CREATE TABLE comments (
    id         BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    post_id    BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
    author_id  BIGINT NOT NULL REFERENCES users(id) ON DELETE SET NULL,
    parent_id  BIGINT REFERENCES comments(id) ON DELETE CASCADE,  -- for nested comments
    content    TEXT NOT NULL,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- User Follows (self-referencing many-to-many)
CREATE TABLE follows (
    follower_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    followee_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    followed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    CHECK (follower_id != followee_id)  -- can't follow yourself
);

-- Indexes
CREATE INDEX idx_posts_author_id ON posts(author_id);
CREATE INDEX idx_posts_status ON posts(status) WHERE status = 'published';
CREATE INDEX idx_posts_published_at ON posts(published_at DESC)
    WHERE status = 'published';
CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_follows_followee_id ON follows(followee_id);
```

---

## Exercises

**Exercise 1:** Identify all normalization violations and fix them:
```
employee_projects (BAD):
emp_id | emp_name | project_id | project_name | project_manager_id | manager_name
```

**Exercise 2:** Design a schema for a music streaming service. Consider:
artists, albums, tracks, users, playlists, and play history.

**Exercise 3:** What's the difference between:
- Storing `country = 'US'` on every user row
- Storing `country_id` and having a `countries` table
When would you choose each approach?

**Exercise 4:** Design a messaging system where users send messages to each other.
Messages can have attachments. Design the tables, relationships, and constraints.

---

## Key Takeaways

1. 1NF: atomic values, no repeating columns — fix with separate tables
2. 2NF: no partial dependency on composite key — fix by moving columns to their own table
3. 3NF: no transitive dependency — fix by extracting non-key-dependent columns
4. Many-to-many always needs a junction table
5. Self-referencing FK enables hierarchies (categories, employees, comments)
6. Denormalize intentionally and document why — snapshot price in order_items is valid
7. Soft delete (`deleted_at` column) is safer than actual DELETE for audit trails

---

## Next Lesson
[Lesson 12 — Advanced Queries](12-advanced-queries.md)
