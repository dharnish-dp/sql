# Lesson 02 — SQL Fundamentals

## Goal
Master the core SELECT statement — the foundation of every SQL query you'll
ever write. Every advanced topic builds directly on this.

## Prerequisites
- [Lesson 01](01-how-databases-work.md) — database setup and sample data loaded

## After This Lesson You Will Be Able To
- Write any basic SELECT query with confidence
- Filter, sort, and limit results
- Understand how PostgreSQL evaluates a SELECT statement
- Use column aliases and computed columns

---

## The SELECT Statement — Mental Model

SQL is **declarative**: you describe WHAT you want, not HOW to get it.

```sql
SELECT  what columns to return
FROM    which table
WHERE   which rows to include (filter)
ORDER BY how to sort the results
LIMIT   how many rows to return
OFFSET  how many rows to skip
```

**Critical rule:** PostgreSQL evaluates these clauses in this order —
NOT the order you write them:

```
1. FROM        — identify the source table(s)
2. WHERE       — filter rows
3. GROUP BY    — group rows (Lesson 05)
4. HAVING      — filter groups (Lesson 05)
5. SELECT      — compute/select the output columns
6. DISTINCT    — remove duplicates
7. ORDER BY    — sort
8. LIMIT/OFFSET — paginate
```

This matters. It explains why you CAN'T do:
```sql
-- WRONG: alias defined in SELECT, but WHERE runs before SELECT
SELECT price * 0.9 AS discounted
FROM products
WHERE discounted < 50;   -- ERROR: column "discounted" does not exist
```

```sql
-- CORRECT: use the expression in WHERE
SELECT price * 0.9 AS discounted
FROM products
WHERE price * 0.9 < 50;
```

---

## SELECT — Choosing Columns

```sql
-- All columns (avoid in production: fragile if schema changes)
SELECT * FROM customers;

-- Specific columns
SELECT name, email FROM customers;

-- With table prefix (good habit when joining multiple tables)
SELECT customers.name, customers.email FROM customers;

-- Column alias with AS
SELECT name AS customer_name, email AS contact_email FROM customers;

-- AS is optional (but explicit is better)
SELECT name customer_name FROM customers;

-- Computed columns — SQL can do arithmetic inline
SELECT
    name,
    price,
    price * 1.1           AS price_with_tax,
    price * 0.9           AS discounted_price,
    price - (price * 0.1) AS also_discounted   -- same result
FROM products;

-- String concatenation with ||
SELECT name || ' (' || category || ')' AS product_label FROM products;

-- Example output: "Laptop Pro 15 (Electronics)"

-- Constants in SELECT
SELECT
    name,
    price,
    'USD'   AS currency,
    NOW()   AS queried_at
FROM products;
```

---

## FROM — The Source Table

```sql
-- Basic FROM
SELECT * FROM customers;

-- Schema-qualified table name (if not using default 'public' schema)
SELECT * FROM public.customers;

-- Table alias — a short name for the table (essential in JOINs)
SELECT c.name, c.email FROM customers c;
SELECT c.name, c.email FROM customers AS c;  -- AS is optional here too
```

---

## WHERE — Filtering Rows

WHERE uses boolean conditions. Only rows where the condition is TRUE are returned.

```sql
-- Equality
SELECT * FROM customers WHERE country = 'US';

-- Inequality
SELECT * FROM customers WHERE country != 'US';
SELECT * FROM customers WHERE country <> 'US';  -- same, older syntax

-- Comparison operators
SELECT * FROM products WHERE price > 100;
SELECT * FROM products WHERE price >= 100;
SELECT * FROM products WHERE price < 50;
SELECT * FROM products WHERE price <= 50;

-- Combining conditions with AND
SELECT * FROM products WHERE price > 50 AND price < 500;

-- Combining with OR
SELECT * FROM products WHERE category = 'Electronics' OR category = 'Furniture';

-- NOT
SELECT * FROM products WHERE NOT category = 'Stationery';

-- Parentheses for clarity (always use them with AND + OR)
SELECT * FROM products
WHERE (category = 'Electronics' OR category = 'Furniture')
  AND price > 100;

-- Without parentheses, AND binds tighter than OR:
-- category = 'Electronics' OR (category = 'Furniture' AND price > 100)
-- That's different! Always use parens.
```

### NULL comparisons — a common gotcha

```sql
-- WRONG: NULL = NULL is not TRUE, it's NULL (unknown)
SELECT * FROM employees WHERE manager_id = NULL;   -- returns 0 rows!

-- CORRECT: use IS NULL
SELECT * FROM employees WHERE manager_id IS NULL;   -- the top-level managers

-- NOT NULL
SELECT * FROM employees WHERE manager_id IS NOT NULL;  -- everyone with a manager
```

**Rule**: NULL represents "unknown/missing". Comparing NULL with `=` or `!=` always
returns NULL (falsy), never TRUE. Use `IS NULL` and `IS NOT NULL`.

---

## ORDER BY — Sorting Results

```sql
-- Ascending (default — smallest to largest, A to Z)
SELECT * FROM products ORDER BY price;
SELECT * FROM products ORDER BY price ASC;   -- same

-- Descending (largest to smallest, Z to A)
SELECT * FROM products ORDER BY price DESC;

-- Sort by multiple columns
-- Primary sort: category ASC, then within each category sort by price DESC
SELECT name, category, price
FROM products
ORDER BY category ASC, price DESC;

-- Sort by column alias
SELECT name, price * 0.9 AS discounted
FROM products
ORDER BY discounted DESC;

-- Sort by column position (fragile — avoid in production code)
SELECT name, category, price FROM products ORDER BY 3 DESC;  -- 3 = price

-- NULLs sort last by default in ASC, first in DESC
-- Override with NULLS FIRST / NULLS LAST
SELECT name, manager_id FROM employees ORDER BY manager_id ASC NULLS LAST;
```

---

## LIMIT and OFFSET — Pagination

```sql
-- Return only 5 rows
SELECT * FROM products ORDER BY price DESC LIMIT 5;

-- Skip first 5, return next 5 (page 2 of 5-per-page)
SELECT * FROM products ORDER BY price DESC LIMIT 5 OFFSET 5;

-- Page 3
SELECT * FROM products ORDER BY price DESC LIMIT 5 OFFSET 10;

-- General formula: page N (1-indexed), page_size rows
-- OFFSET = (N - 1) * page_size
-- Page 4: OFFSET = 3 * 5 = 15
SELECT * FROM products ORDER BY price DESC LIMIT 5 OFFSET 15;
```

**Important**: Always use `ORDER BY` with `LIMIT`. Without ORDER BY, PostgreSQL
can return rows in any order — your "page 1" and "page 2" might contain overlaps
or gaps.

---

## DISTINCT — Removing Duplicates

```sql
-- All countries customers are from (unique values)
SELECT DISTINCT country FROM customers ORDER BY country;

-- All unique categories
SELECT DISTINCT category FROM products;

-- DISTINCT on multiple columns — unique combinations
SELECT DISTINCT country, city FROM customers ORDER BY country, city;

-- DISTINCT with COUNT — how many unique countries?
SELECT COUNT(DISTINCT country) AS unique_countries FROM customers;
```

---

## Working With Text

```sql
-- Case-insensitive comparison using LOWER() or ILIKE
SELECT * FROM customers WHERE LOWER(name) = 'alice johnson';
SELECT * FROM customers WHERE name ILIKE 'alice johnson';  -- PostgreSQL-specific

-- String length
SELECT name, LENGTH(name) AS name_length FROM customers ORDER BY name_length DESC;

-- Substring
SELECT SUBSTRING(email, 1, 5) AS email_start FROM customers;
-- Or: SUBSTR(email, 1, 5)

-- Left and right
SELECT LEFT(email, 5), RIGHT(email, 4) FROM customers;

-- Trim whitespace
SELECT TRIM('  hello world  ');       -- 'hello world'
SELECT LTRIM('  hello world  ');      -- 'hello world  '
SELECT RTRIM('  hello world  ');      -- '  hello world'

-- Upper and lower
SELECT UPPER(name), LOWER(email) FROM customers;

-- Replace
SELECT REPLACE(email, '@example.com', '@newdomain.com') FROM customers;

-- Concatenation
SELECT first_name || ' ' || last_name AS full_name FROM some_table;
-- Or using CONCAT function
SELECT CONCAT(name, ' from ', city) AS description FROM customers;
-- CONCAT ignores NULLs; || propagates NULL (NULL || 'x' = NULL)
SELECT CONCAT_WS(', ', name, city, country) AS address FROM customers;
-- CONCAT_WS: concatenate with separator, skips NULLs
```

---

## Working With Numbers

```sql
-- Arithmetic
SELECT
    price,
    price * 1.08        AS with_sales_tax,
    ROUND(price * 1.08, 2) AS rounded,
    FLOOR(price)        AS floor_price,
    CEIL(price)         AS ceil_price,
    ABS(-price)         AS absolute,
    price ^ 2           AS squared,
    SQRT(price)         AS sqrt_price,
    price % 10          AS remainder
FROM products;

-- Integer division
SELECT 10 / 3;       -- Result: 3 (integer division, not 3.333)
SELECT 10.0 / 3;     -- Result: 3.333... (float division)
SELECT 10 / 3.0;     -- Result: 3.333...
SELECT 10::float / 3; -- Cast to float first

-- Rounding
SELECT ROUND(3.456, 2);   -- 3.46
SELECT ROUND(3.454, 2);   -- 3.45
SELECT ROUND(3.5);        -- 4 (banker's rounding: rounds to even if exactly .5)
SELECT TRUNC(3.999);      -- 3 (truncates, doesn't round)
SELECT TRUNC(3.999, 1);   -- 3.9
```

---

## Working With Dates

```sql
-- Current date and time
SELECT NOW();              -- 2024-03-15 14:30:22.123456+00
SELECT CURRENT_DATE;       -- 2024-03-15
SELECT CURRENT_TIME;       -- 14:30:22.123456+00
SELECT CURRENT_TIMESTAMP;  -- same as NOW()

-- Extract parts from a date
SELECT
    order_date,
    EXTRACT(YEAR  FROM order_date) AS year,
    EXTRACT(MONTH FROM order_date) AS month,
    EXTRACT(DAY   FROM order_date) AS day,
    EXTRACT(DOW   FROM order_date) AS day_of_week,  -- 0=Sunday, 6=Saturday
    EXTRACT(QUARTER FROM order_date) AS quarter
FROM orders;

-- Date arithmetic
SELECT
    order_date,
    order_date + 30         AS plus_30_days,
    order_date - 7          AS minus_7_days,
    NOW() - order_date      AS age,           -- interval type
    NOW()::date - order_date AS days_since    -- integer
FROM orders;

-- Date truncation
SELECT
    DATE_TRUNC('month', order_date)  AS month_start,
    DATE_TRUNC('year',  order_date)  AS year_start,
    DATE_TRUNC('week',  order_date)  AS week_start
FROM orders;

-- Format dates as strings
SELECT TO_CHAR(order_date, 'Month DD, YYYY') FROM orders;
-- Output: "January  05, 2024"

SELECT TO_CHAR(order_date, 'YYYY-MM') AS year_month FROM orders;

-- Parse strings to dates
SELECT TO_DATE('15/03/2024', 'DD/MM/YYYY');

-- Interval literals
SELECT NOW() - INTERVAL '30 days';
SELECT NOW() + INTERVAL '1 year 2 months 3 days';
SELECT NOW() + INTERVAL '2 hours 30 minutes';
```

---

## Type Casting

```sql
-- Cast syntax: value::type  OR  CAST(value AS type)
SELECT '42'::INT;             -- text → integer
SELECT '3.14'::FLOAT;         -- text → float
SELECT '3.14'::NUMERIC(5,2);  -- text → fixed-point
SELECT 42::TEXT;              -- integer → text
SELECT '2024-03-15'::DATE;    -- text → date
SELECT NOW()::DATE;           -- timestamp → date (drops time)
SELECT 10::FLOAT / 3;         -- integer → float before division

-- CAST() is standard SQL, :: is PostgreSQL shorthand
SELECT CAST('42' AS INT);
SELECT CAST(NOW() AS DATE);
```

---

## Complete Query Examples

**Example 1 — Products under $100 sorted by price:**
```sql
SELECT name, category, price
FROM products
WHERE price < 100
ORDER BY price ASC;
```

**Example 2 — Top 3 most expensive products:**
```sql
SELECT name, price
FROM products
ORDER BY price DESC
LIMIT 3;
```

**Example 3 — US customers from New York, with computed field:**
```sql
SELECT
    name,
    email,
    city,
    'Member since: ' || TO_CHAR(created_at, 'YYYY') AS member_info
FROM customers
WHERE country = 'US'
  AND city = 'New York'
ORDER BY name;
```

**Example 4 — Orders from January 2024:**
```sql
SELECT id, customer_id, order_date, total
FROM orders
WHERE order_date >= '2024-01-01'
  AND order_date < '2024-02-01'
ORDER BY order_date;
```

**Example 5 — Delivered or shipped orders, total > 100:**
```sql
SELECT id, customer_id, status, total
FROM orders
WHERE (status = 'delivered' OR status = 'shipped')
  AND total > 100
ORDER BY total DESC;
```

**Example 6 — Products with price between 30 and 150, showing tax:**
```sql
SELECT
    name,
    price,
    ROUND(price * 0.08, 2)        AS tax,
    ROUND(price * 1.08, 2)        AS total_with_tax,
    stock,
    ROUND(price * stock, 2)       AS inventory_value
FROM products
WHERE price BETWEEN 30 AND 150   -- inclusive on both ends
ORDER BY inventory_value DESC;
```

**Example 7 — Employees hired in 2021 or later:**
```sql
SELECT
    name,
    department,
    salary,
    hire_date,
    NOW()::DATE - hire_date AS days_employed
FROM employees
WHERE hire_date >= '2021-01-01'
ORDER BY hire_date;
```

**Example 8 — Employees without a manager (top-level):**
```sql
SELECT name, department, salary
FROM employees
WHERE manager_id IS NULL
ORDER BY salary DESC;
```

---

## Exercises

**Exercise 1:** Find all products in the 'Electronics' category with a price over $50.
Show only the name and price, sorted by price ascending.

**Exercise 2:** Find all customers NOT from the US. Show name, city, country.

**Exercise 3:** Show the top 5 most expensive products. Include name, category, price,
and a column called `discounted` that shows 15% off the price.

**Exercise 4:** List all orders placed in February or March 2024 that were NOT cancelled.
Show order id, date, status, and total.

**Exercise 5:** Show unique departments from the employees table.
How many unique departments are there?

**Exercise 6:** Find employees with a salary between 70,000 and 100,000.
Show name, department, salary, and how many days they've been employed.

---

## Key Takeaways

1. PostgreSQL evaluates: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
2. You can't reference SELECT aliases in WHERE (alias doesn't exist yet)
3. NULL comparisons use `IS NULL` / `IS NOT NULL`, never `= NULL`
4. Always ORDER BY when using LIMIT — otherwise results are unpredictable
5. `||` for string concat propagates NULL; use `CONCAT()` to ignore NULLs
6. Type casting with `::` is PostgreSQL shorthand; `CAST()` is standard SQL

---

## Next Lesson
[Lesson 03 — Data Types & Schema Design](03-data-types-and-schema.md)
