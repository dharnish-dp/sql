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

Database design is the process of deciding:
1. **What entities** to store (tables)
2. **What attributes** each entity has (columns)
3. **How entities relate** to each other (foreign keys)
4. **What rules** the data must satisfy (constraints)

Bad design causes: redundancy, anomalies, difficult queries, poor performance.
Good design causes: one source of truth, easy queries, maintainability.

---

## Step 1: Identify Entities and Attributes

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

Relationships are the verbs connecting entities:

```
Member ─── borrows ──► Book        (Many-to-Many: through Loan)
Author ─── writes ───► Book        (Many-to-Many: one author writes many books, book has multiple authors)
Librarian ── manages ─► Loan       (One-to-Many: one librarian processes many loans)
Book ─── has ────────► Author      (through junction table book_authors)
```

**Relationship cardinality:**
- **One-to-One (1:1):** One person has one passport
- **One-to-Many (1:N):** One customer has many orders
- **Many-to-Many (M:N):** Students enroll in courses, courses have many students → requires junction table

---

## Entity-Relationship Diagram (ERD)

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

Normalization is the process of structuring a database to reduce redundancy
and improve data integrity. Each "normal form" fixes a specific class of problem.

---

## First Normal Form (1NF)

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

**Problem:** Can't search for "books by Ralph" without string parsing. Can't JOIN.

### Fix for 1NF:
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

### Another 1NF Violation — Repeating Columns:
```
orders table (BAD):
┌────┬─────────────┬──────────┬──────────┬──────────┐
│ id │ customer_id │ product1 │ product2 │ product3 │
├────┼─────────────┼──────────┼──────────┼──────────┤
│  1 │      1      │ Laptop   │ Mouse    │ NULL     │
└────┴─────────────┴──────────┴──────────┴──────────┘
```

**Fix:** Use a separate `order_items` table (one row per item).

---

## Second Normal Form (2NF)

**Rule:** Must be in 1NF. Every non-key column must depend on the WHOLE primary key
(not just part of it). Only matters when there's a composite primary key.

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

**Problem:** `product_name` and `product_price` depend only on `product_id`,
not on the composite key `(order_id, product_id)`. Data is duplicated.
If the price changes, you must update it in every row.

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

---

## Third Normal Form (3NF)

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

**Problem:** `dept_name` and `dept_location` depend on `dept_id`, not on `id`.
If Engineering moves floors, you must update every Engineering employee row.

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

---

## Boyce-Codd Normal Form (BCNF)

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

In practice, 3NF is sufficient for most applications.

---

## Denormalization — When to Break the Rules

Normalization is great for write integrity. But sometimes you need
to denormalize for read performance.

### When to denormalize:
1. Frequently read, rarely written data
2. Reports that JOIN many tables (consider materialized views instead)
3. Aggregated values that are expensive to recompute

```sql
-- Normalized: customer's total order count is computed each time
SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id;

-- Denormalized: store total_orders on the customer row
ALTER TABLE customers ADD COLUMN total_orders INT DEFAULT 0;
-- Update it with a trigger (Lesson 14)
-- Pro: instant lookup
-- Con: must keep it in sync with every INSERT/DELETE to orders

-- When denormalization is intentional and documented:
-- order_items.unit_price is a denormalized snapshot of products.price at time of order
-- This is CORRECT: we want historical price, not current price
```

---

## Common Design Patterns

### Many-to-Many Relationship (Junction Table)

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

### Audit Trail (Immutable History)

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

### Soft Delete

```sql
-- Instead of DELETE, set deleted_at
CREATE TABLE users (
    id         SERIAL PRIMARY KEY,
    email      TEXT UNIQUE NOT NULL,
    deleted_at TIMESTAMPTZ  -- NULL = active, timestamp = deleted
);

-- "Active" query
SELECT * FROM users WHERE deleted_at IS NULL;

-- View to make it transparent
CREATE VIEW active_users AS
SELECT * FROM users WHERE deleted_at IS NULL;
```

### Polymorphic Association (Tagging, Comments)

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

-- Alternative: separate tables (cleaner but more tables)
CREATE TABLE post_comments    (id SERIAL PK, post_id INT FK, content TEXT, ...);
CREATE TABLE product_comments (id SERIAL PK, product_id INT FK, content TEXT, ...);
```

---

## Full Design Example — Blog Platform

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
