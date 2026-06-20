# Lesson 14 — Stored Procedures & Functions

## Goal
Write reusable database-side logic using PL/pgSQL. Learn functions, procedures,
and triggers — the tools that let your database enforce business logic.

## Prerequisites
- [Lesson 13](13-views-and-materialized-views.md) — views

## After This Lesson You Will Be Able To
- Write SQL and PL/pgSQL functions
- Create stored procedures
- Use triggers to automate database actions
- Return tables and record sets from functions
- Write functions with error handling

---

## Functions vs Procedures — The Difference

| | Function | Procedure |
|--|----------|-----------|
| Return value | Must return a value | No return value |
| Call syntax | `SELECT my_func()` | `CALL my_proc()` |
| Transactions | Can't manage transactions | Can COMMIT/ROLLBACK |
| Use in SQL | Yes (in SELECT, WHERE) | No |
| Added in | Early PostgreSQL | PostgreSQL 11+ |

---

## SQL Functions — Simplest Form

```sql
-- Function that returns a scalar value
CREATE OR REPLACE FUNCTION get_customer_count()
RETURNS INT
LANGUAGE SQL
AS $$
    SELECT COUNT(*)::INT FROM customers;
$$;

-- Call it
SELECT get_customer_count();

-- Function with parameters
CREATE OR REPLACE FUNCTION get_orders_by_status(p_status TEXT)
RETURNS INT
LANGUAGE SQL
AS $$
    SELECT COUNT(*)::INT FROM orders WHERE status = p_status;
$$;

SELECT get_orders_by_status('delivered');
SELECT get_orders_by_status('pending');

-- Function with default parameters
CREATE OR REPLACE FUNCTION get_orders_by_status(p_status TEXT DEFAULT 'pending')
RETURNS INT LANGUAGE SQL AS $$
    SELECT COUNT(*)::INT FROM orders WHERE status = p_status;
$$;

SELECT get_orders_by_status();           -- uses 'pending'
SELECT get_orders_by_status('shipped');  -- explicit

-- SQL function returning a table row
CREATE OR REPLACE FUNCTION get_customer(p_id INT)
RETURNS customers  -- returns full row type
LANGUAGE SQL AS $$
    SELECT * FROM customers WHERE id = p_id;
$$;

SELECT (get_customer(1)).*;  -- expand the row
SELECT (get_customer(1)).name, (get_customer(1)).email;
```

---

## PL/pgSQL Functions — Procedural Logic

PL/pgSQL is PostgreSQL's procedural language (like Python in the DB).

```sql
-- Basic PL/pgSQL function
CREATE OR REPLACE FUNCTION add_numbers(a INT, b INT)
RETURNS INT
LANGUAGE plpgsql
AS $$
DECLARE
    result INT;
BEGIN
    result := a + b;
    RETURN result;
END;
$$;

SELECT add_numbers(3, 4);  -- 7
```

### Variables and Assignment

```sql
CREATE OR REPLACE FUNCTION demo_variables()
RETURNS TEXT
LANGUAGE plpgsql AS $$
DECLARE
    customer_count  INT;
    avg_order       NUMERIC(10,2);
    greeting        TEXT;
    today           DATE := CURRENT_DATE;  -- initialize at declaration
BEGIN
    SELECT COUNT(*) INTO customer_count FROM customers;
    SELECT AVG(total) INTO avg_order FROM orders;

    greeting := 'There are ' || customer_count || ' customers.';
    greeting := greeting || ' Avg order: $' || avg_order;
    greeting := greeting || ' Today: ' || today::TEXT;

    RETURN greeting;
END;
$$;

SELECT demo_variables();
```

### IF / ELSE / ELSIF

```sql
CREATE OR REPLACE FUNCTION categorize_price(p_price NUMERIC)
RETURNS TEXT
LANGUAGE plpgsql AS $$
BEGIN
    IF p_price < 30 THEN
        RETURN 'Budget';
    ELSIF p_price < 100 THEN
        RETURN 'Mid-range';
    ELSIF p_price < 500 THEN
        RETURN 'Premium';
    ELSE
        RETURN 'Luxury';
    END IF;
END;
$$;

SELECT name, price, categorize_price(price) AS tier FROM products;
```

### Loops

```sql
-- FOR loop over a range
CREATE OR REPLACE FUNCTION sum_to_n(n INT)
RETURNS INT LANGUAGE plpgsql AS $$
DECLARE
    total INT := 0;
    i     INT;
BEGIN
    FOR i IN 1..n LOOP
        total := total + i;
    END LOOP;
    RETURN total;
END;
$$;

SELECT sum_to_n(100);  -- 5050

-- WHILE loop
CREATE OR REPLACE FUNCTION countdown(start INT)
RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE
    current INT := start;
    result  TEXT := '';
BEGIN
    WHILE current > 0 LOOP
        result := result || current::TEXT || ', ';
        current := current - 1;
    END LOOP;
    RETURN result || 'Go!';
END;
$$;

SELECT countdown(5);  -- '5, 4, 3, 2, 1, Go!'

-- FOR loop over query results
CREATE OR REPLACE FUNCTION log_high_earners(threshold NUMERIC)
RETURNS TEXT LANGUAGE plpgsql AS $$
DECLARE
    emp  RECORD;
    msg  TEXT := '';
BEGIN
    FOR emp IN
        SELECT name, salary FROM employees WHERE salary > threshold ORDER BY salary DESC
    LOOP
        msg := msg || emp.name || ': $' || emp.salary || E'\n';
    END LOOP;
    RETURN msg;
END;
$$;

SELECT log_high_earners(90000);
```

---

## Functions That Return Tables

```sql
-- RETURNS TABLE syntax
CREATE OR REPLACE FUNCTION get_customer_orders(p_customer_id INT)
RETURNS TABLE (
    order_id    INT,
    order_date  DATE,
    status      TEXT,
    total       NUMERIC(10,2)
)
LANGUAGE SQL AS $$
    SELECT id, order_date, status, total
    FROM orders
    WHERE customer_id = p_customer_id
    ORDER BY order_date DESC;
$$;

-- Call it like a table
SELECT * FROM get_customer_orders(1);
SELECT status, COUNT(*) FROM get_customer_orders(1) GROUP BY status;

-- RETURNS SETOF — returns multiple rows of a type
CREATE OR REPLACE FUNCTION get_expensive_products(min_price NUMERIC DEFAULT 100)
RETURNS SETOF products  -- returns full product rows
LANGUAGE SQL AS $$
    SELECT * FROM products WHERE price >= min_price ORDER BY price DESC;
$$;

SELECT * FROM get_expensive_products(200);
SELECT name, price FROM get_expensive_products();
```

---

## Error Handling

```sql
CREATE OR REPLACE FUNCTION safe_divide(a NUMERIC, b NUMERIC)
RETURNS NUMERIC LANGUAGE plpgsql AS $$
BEGIN
    IF b = 0 THEN
        RAISE EXCEPTION 'Division by zero: cannot divide % by 0', a;
    END IF;
    RETURN a / b;
EXCEPTION
    WHEN division_by_zero THEN
        RETURN NULL;  -- or raise a custom message
    WHEN others THEN
        RAISE WARNING 'Unexpected error in safe_divide: %', SQLERRM;
        RETURN NULL;
END;
$$;

-- RAISE levels: DEBUG, LOG, INFO, NOTICE, WARNING, EXCEPTION
RAISE NOTICE 'Processing customer %', customer_id;
RAISE WARNING 'Low stock: product % has only % units', p_id, p_stock;
RAISE EXCEPTION 'Customer % not found', p_id;  -- aborts the function

-- PostgreSQL error codes
WHEN unique_violation THEN ...        -- SQLSTATE 23505
WHEN foreign_key_violation THEN ...   -- SQLSTATE 23503
WHEN check_violation THEN ...         -- SQLSTATE 23514
WHEN not_null_violation THEN ...      -- SQLSTATE 23502
WHEN division_by_zero THEN ...        -- SQLSTATE 22012
WHEN others THEN ...                  -- catch all
```

---

## Stored Procedures

Procedures (PostgreSQL 11+) can manage transactions. You CALL them.

```sql
CREATE OR REPLACE PROCEDURE place_order(
    p_customer_id   INT,
    p_product_ids   INT[],
    p_quantities    INT[]
)
LANGUAGE plpgsql AS $$
DECLARE
    v_order_id  INT;
    v_total     NUMERIC(10,2) := 0;
    v_price     NUMERIC(10,2);
    v_stock     INT;
    i           INT;
BEGIN
    -- Verify customer exists
    IF NOT EXISTS (SELECT 1 FROM customers WHERE id = p_customer_id) THEN
        RAISE EXCEPTION 'Customer % does not exist', p_customer_id;
    END IF;

    -- Create the order
    INSERT INTO orders (customer_id, status, total)
    VALUES (p_customer_id, 'pending', 0)
    RETURNING id INTO v_order_id;

    -- Process each product
    FOR i IN 1..array_length(p_product_ids, 1) LOOP
        -- Lock and check stock
        SELECT price, stock INTO v_price, v_stock
        FROM products
        WHERE id = p_product_ids[i]
        FOR UPDATE;

        IF v_stock < p_quantities[i] THEN
            RAISE EXCEPTION 'Insufficient stock for product %: have %, need %',
                p_product_ids[i], v_stock, p_quantities[i];
        END IF;

        -- Deduct stock
        UPDATE products SET stock = stock - p_quantities[i]
        WHERE id = p_product_ids[i];

        -- Add order item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price)
        VALUES (v_order_id, p_product_ids[i], p_quantities[i], v_price);

        v_total := v_total + v_price * p_quantities[i];
    END LOOP;

    -- Update order total
    UPDATE orders SET total = v_total WHERE id = v_order_id;

    RAISE NOTICE 'Order % created for customer %, total: $%',
        v_order_id, p_customer_id, v_total;
END;
$$;

-- Call the procedure
CALL place_order(1, ARRAY[1, 2], ARRAY[1, 2]);
-- Creates order: 1 Laptop + 2 Wireless Mice
```

---

## Triggers

A trigger automatically calls a function when data changes.

```
Event: INSERT / UPDATE / DELETE / TRUNCATE
Timing: BEFORE / AFTER / INSTEAD OF
Level: FOR EACH ROW / FOR EACH STATEMENT
```

### Auto-Update updated_at Column

```sql
-- Step 1: Create the trigger function
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at := NOW();  -- NEW = the row being modified
    RETURN NEW;
END;
$$;

-- Step 2: Attach the trigger to tables
CREATE TRIGGER orders_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER products_updated_at
BEFORE UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Now every UPDATE automatically sets updated_at:
UPDATE orders SET status = 'shipped' WHERE id = 1;
-- updated_at is set automatically without your application doing anything
```

### Audit Log Trigger

```sql
CREATE TABLE order_audit_log (
    id         SERIAL PRIMARY KEY,
    order_id   INT NOT NULL,
    operation  TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
    old_status TEXT,
    new_status TEXT,
    old_total  NUMERIC(10,2),
    new_total  NUMERIC(10,2),
    changed_at TIMESTAMPTZ DEFAULT NOW(),
    changed_by TEXT DEFAULT current_user
);

CREATE OR REPLACE FUNCTION log_order_changes()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO order_audit_log (order_id, operation, new_status, new_total)
        VALUES (NEW.id, 'INSERT', NEW.status, NEW.total);

    ELSIF TG_OP = 'UPDATE' THEN
        IF OLD.status != NEW.status OR OLD.total != NEW.total THEN
            INSERT INTO order_audit_log (order_id, operation, old_status, new_status, old_total, new_total)
            VALUES (NEW.id, 'UPDATE', OLD.status, NEW.status, OLD.total, NEW.total);
        END IF;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO order_audit_log (order_id, operation, old_status, old_total)
        VALUES (OLD.id, 'DELETE', OLD.status, OLD.total);
    END IF;

    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER orders_audit
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH ROW EXECUTE FUNCTION log_order_changes();

-- Test it:
UPDATE orders SET status = 'shipped' WHERE id = 1;
SELECT * FROM order_audit_log;
```

### Validation Trigger

```sql
-- Prevent an order from going backwards in status
CREATE OR REPLACE FUNCTION validate_status_transition()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
DECLARE
    allowed_next TEXT[];
BEGIN
    -- Define valid transitions
    allowed_next := CASE OLD.status
        WHEN 'pending'   THEN ARRAY['shipped', 'cancelled']
        WHEN 'shipped'   THEN ARRAY['delivered', 'cancelled']
        WHEN 'delivered' THEN ARRAY[]::TEXT[]  -- terminal
        WHEN 'cancelled' THEN ARRAY[]::TEXT[]  -- terminal
        ELSE ARRAY[]::TEXT[]
    END;

    IF NOT (NEW.status = ANY(allowed_next)) THEN
        RAISE EXCEPTION 'Invalid status transition: % → % (allowed: %)',
            OLD.status, NEW.status, allowed_next;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER orders_status_guard
BEFORE UPDATE OF status ON orders
FOR EACH ROW EXECUTE FUNCTION validate_status_transition();

-- Test valid:
UPDATE orders SET status = 'shipped' WHERE id = 5 AND status = 'pending';
-- Test invalid:
UPDATE orders SET status = 'pending' WHERE id = 1 AND status = 'delivered';
-- ERROR: Invalid status transition: delivered → pending
```

### Trigger Variables Available in PL/pgSQL

```sql
-- In a row-level trigger:
NEW     -- the new row (INSERT, UPDATE) — modify it in BEFORE triggers
OLD     -- the old row (UPDATE, DELETE)
TG_OP   -- operation: 'INSERT', 'UPDATE', 'DELETE', 'TRUNCATE'
TG_TABLE_NAME  -- name of the table that fired the trigger
TG_WHEN        -- 'BEFORE', 'AFTER', 'INSTEAD OF'
TG_LEVEL       -- 'ROW', 'STATEMENT'

-- Returning values from row-level triggers:
-- BEFORE trigger: return NEW to proceed, return NULL to cancel the operation
-- AFTER trigger: return value is ignored, return NULL or NEW
-- INSTEAD OF trigger: return NEW to indicate success, NULL to skip
```

---

## Managing Functions

```sql
-- List all functions
\df               -- psql meta-command
\df+ function_name -- with detail

-- Drop a function
DROP FUNCTION get_customer_count();
DROP FUNCTION IF EXISTS get_orders_by_status(TEXT);
-- Must include parameter types when overloaded functions exist

-- List triggers on a table
SELECT trigger_name, event_manipulation, action_timing
FROM information_schema.triggers
WHERE event_object_table = 'orders';

-- Drop a trigger
DROP TRIGGER orders_audit ON orders;
DROP TRIGGER IF EXISTS orders_updated_at ON orders;
```

---

## Exercises

**Exercise 1:** Create a function `customer_tier(customer_id INT)` that returns
'VIP', 'Regular', or 'New' based on total spending from the orders table.

**Exercise 2:** Create a trigger that automatically inserts a welcome email
into a `notifications` table whenever a new customer is inserted.
(Create the notifications table too.)

**Exercise 3:** Write a PL/pgSQL function `transfer_stock(from_product INT, to_product INT, qty INT)`
that moves `qty` units from one product to another. Include validation that:
- Both products exist
- from_product has enough stock
- qty is positive

**Exercise 4:** Create an audit trigger on the `products` table that records
every price change (old price, new price, changed at).

---

## Key Takeaways

1. SQL functions are simpler (just a SELECT); PL/pgSQL functions support full procedural logic
2. `RETURNS TABLE (...)` lets functions return multiple rows with named columns
3. Procedures (PostgreSQL 11+) can manage transactions; functions cannot
4. Triggers fire automatically on data changes — great for audit logs, validation, defaults
5. In BEFORE row triggers: modify `NEW` to change what gets written; return NULL to abort
6. `RAISE EXCEPTION` with `EXCEPTION WHEN` blocks for structured error handling
7. Use triggers sparingly — they make debugging harder and can cause performance issues

---

## Next Lesson
[Lesson 15 — Query Optimization](15-query-optimization.md)
