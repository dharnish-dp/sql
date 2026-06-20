# Exercise 05 — Schema Design Challenge

## Overview
Design database schemas from scratch given real-world requirements.
This tests everything: normalization, relationships, constraints, and indexes.

## Skills Practiced
ERD design, normalization, constraints, relationships, indexes

---

## Challenge 1 — Hotel Booking System

**Requirements:**
- Hotels have rooms. Each room has a type (single, double, suite) and a price per night.
- Guests can make reservations for specific rooms and date ranges.
- A reservation can include multiple rooms (group bookings).
- Each reservation has a payment. Payments can be split across multiple methods (credit card, cash, etc.).
- Hotels have amenities (pool, gym, wifi). Guests can rate amenities.

**Tasks:**
1. Identify all entities and their attributes
2. Draw the relationships (describe the ERD in text)
3. Write the full SQL `CREATE TABLE` statements with appropriate:
   - Primary keys
   - Foreign keys with correct ON DELETE actions
   - CHECK constraints
   - UNIQUE constraints
   - NOT NULL constraints
   - Defaults
4. Add indexes for the most common query patterns:
   - Find available rooms for a date range
   - Find all reservations for a guest
   - Find revenue per hotel per month
5. Write 3 sample queries against your schema

---

## Challenge 2 — Multi-Tenant SaaS Application

**Requirements:**
- The application has "organizations" (tenants) and "users".
- Users belong to one organization. Users have roles within their organization: admin, editor, viewer.
- Organizations have "projects". Each project belongs to one organization.
- Projects have "tasks". Tasks can be assigned to users.
- Tasks can have comments. Comments can have replies (nested).
- Tasks have labels (many-to-many). Labels are scoped to an organization.
- Everything must be isolated per tenant — organization A can never see organization B's data.

**Tasks:**
1. Design the full schema
2. Implement Row-Level Security policies that enforce tenant isolation
3. Write queries for:
   - All tasks assigned to a user across all their projects
   - All overdue tasks (past due date, not completed) per project
   - Activity feed: last 20 actions (tasks created/completed, comments added) for a user's organization

---

## Challenge 3 — Normalize This

**Requirements:** You've inherited a badly designed table. Normalize it to 3NF.

```sql
-- The existing (terrible) table
CREATE TABLE employee_projects_bad (
    emp_id         INT,
    emp_name       TEXT,
    emp_email      TEXT,
    dept_id        INT,
    dept_name      TEXT,
    dept_floor     INT,
    dept_manager   TEXT,          -- manager's name (not id!)
    project_id     INT,
    project_name   TEXT,
    project_status TEXT,
    project_budget NUMERIC,
    project_mgr_email TEXT,       -- project manager's email
    role_on_project TEXT,
    hours_worked    NUMERIC,
    hourly_rate     NUMERIC,
    total_cost      NUMERIC,      -- hours_worked * hourly_rate (computed!)
    skill_ids       TEXT          -- "1,3,7" comma-separated skills
);
```

**Tasks:**
1. Identify every normalization violation (list them all)
2. Write the normalized schema (as many tables as needed) in 3NF
3. Write a migration script that populates the new tables from the old one
4. Verify: can you reconstruct the original table as a view from the new tables?

---

## Challenge 4 — Inventory Management System

**Requirements:**
- Products have variants (color, size). Each variant is a separate SKU.
- Warehouses hold inventory. Each warehouse stores specific SKUs.
- Inventory levels change via "inventory movements" (IN = stock received, OUT = sold/used).
- The current stock level of a SKU at a warehouse = SUM of all movements.
- Products can belong to categories (hierarchical categories: Electronics > Phones > Smartphones).
- Suppliers supply products. One product can have multiple suppliers with different prices and lead times.

**Tasks:**
1. Design the full schema
2. Write a function `get_stock_level(sku_id, warehouse_id)` that returns current stock
3. Write a materialized view `inventory_snapshot` showing current stock of all SKUs at all warehouses
4. Write a trigger that prevents inventory movements that would result in negative stock
5. Write a recursive query to display the full category tree

---

## Review Questions

After completing the challenges, answer these:

1. When should a column have `ON DELETE CASCADE` vs `ON DELETE SET NULL` vs `ON DELETE RESTRICT`?

2. When should you use a UUID primary key instead of a serial integer?

3. In Challenge 1: a room is reserved from Jan 5 to Jan 8. Is Jan 8 check-in or check-out?
   How does this affect your date range query for availability?

4. In Challenge 2: should labels be in a separate table with a unique constraint on `(org_id, name)`,
   or should they be stored as an array on the project? What are the trade-offs?

5. In Challenge 3: the `total_cost` column is a computed value. Should you store it or compute it?
   What are the trade-offs? When is storing a computed value justified?
