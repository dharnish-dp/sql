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
```
Output (first 3 of 10 rows):
| id | name | email | city | country |
|---|---|---|---|---|
| 1 | Alice Johnson | alice@example.com | New York | US |
| 2 | Bob Smith | bob@example.com | London | UK |
| 3 | Carol White | carol@example.com | Toronto | CA |

```sql
-- Specific columns
SELECT name, email FROM customers;
```
Output (first 3 rows) — same rows, only the two chosen columns:
| name | email |
|---|---|
| Alice Johnson | alice@example.com |
| Bob Smith | bob@example.com |
| Carol White | carol@example.com |

```sql
-- With table prefix (good habit when joining multiple tables)
SELECT customers.name, customers.email FROM customers;

-- Column alias with AS
SELECT name AS customer_name, email AS contact_email FROM customers;
```
Output — identical data, only the column headers change:
| customer_name | contact_email |
|---|---|
| Alice Johnson | alice@example.com |

```sql
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
```
Output (first 2 of 10 rows):
| name | price | price_with_tax | discounted_price | also_discounted |
|---|---|---|---|---|
| Laptop Pro 15 | 1299.99 | 1429.989 | 1169.991 | 1169.991 |
| Wireless Mouse | 29.99 | 32.989 | 26.991 | 26.991 |

```sql
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
Output (first 2 rows) — `currency` and `queried_at` are the same literal
value repeated on every row, not looked up from any table:
| name | price | currency | queried_at |
|---|---|---|---|
| Laptop Pro 15 | 1299.99 | USD | 2026-09-08 10:15:00 |
| Wireless Mouse | 29.99 | USD | 2026-09-08 10:15:00 |

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
```
Output (using the sample data — Alice, David, Frank, Jack are all `US`):
| id | name | email | city | country |
|---|---|---|---|---|
| 1 | Alice Johnson | alice@example.com | New York | US |
| 4 | David Brown | david@example.com | New York | US |
| 6 | Frank Lee | frank@example.com | San Francisco | US |
| 10 | Jack Davis | jack@example.com | Chicago | US |

```sql
-- Inequality
SELECT * FROM customers WHERE country != 'US';
SELECT * FROM customers WHERE country <> 'US';  -- same, older syntax

-- Comparison operators
SELECT * FROM products WHERE price > 100;
```
Output (`price > 100`, from the products sample data):
| name | category | price | stock |
|---|---|---|---|
| Laptop Pro 15 | Electronics | 1299.99 | 50 |
| Mechanical Keyboard | Electronics | 129.99 | 75 |
| Monitor 27" | Electronics | 399.99 | 30 |
| Standing Desk | Furniture | 599.99 | 20 |
| Ergonomic Chair | Furniture | 449.99 | 25 |

```sql
SELECT * FROM products WHERE price >= 100;
SELECT * FROM products WHERE price < 50;
SELECT * FROM products WHERE price <= 50;

-- Combining conditions with AND
SELECT * FROM products WHERE price > 50 AND price < 500;
```
Output — both conditions must hold, so this excludes anything ≤ $50 or ≥ $500:
| name | category | price |
|---|---|---|
| USB-C Hub | Electronics | 49.99 |
| Mechanical Keyboard | Electronics | 129.99 |
| Monitor 27" | Electronics | 399.99 |
| Ergonomic Chair | Furniture | 449.99 |

```sql
-- Combining with OR
SELECT * FROM products WHERE category = 'Electronics' OR category = 'Furniture';

-- NOT
SELECT * FROM products WHERE NOT category = 'Stationery';

-- Parentheses for clarity (always use them with AND + OR)
SELECT * FROM products
WHERE (category = 'Electronics' OR category = 'Furniture')
  AND price > 100;
```
Output — must be Electronics OR Furniture, AND over $100 (Notebook and
Pen Set are excluded even though cheap Electronics/Furniture items exist,
because they fail the price condition):
| name | category | price |
|---|---|---|
| Laptop Pro 15 | Electronics | 1299.99 |
| Mechanical Keyboard | Electronics | 129.99 |
| Monitor 27" | Electronics | 399.99 |
| Standing Desk | Furniture | 599.99 |
| Ergonomic Chair | Furniture | 449.99 |

```sql
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
```
Output (first 3 of 10 rows — cheapest first):
| name | price |
|---|---|
| Notebook (paper) | 4.99 |
| Pen Set | 9.99 |
| Wireless Mouse | 29.99 |

```sql
-- Descending (largest to smallest, Z to A)
SELECT * FROM products ORDER BY price DESC;
```
Output (first 3 rows — most expensive first, exact reverse of above):
| name | price |
|---|---|
| Laptop Pro 15 | 1299.99 |
| Standing Desk | 599.99 |
| Ergonomic Chair | 449.99 |

```sql
-- Sort by multiple columns
-- Primary sort: category ASC, then within each category sort by price DESC
SELECT name, category, price
FROM products
ORDER BY category ASC, price DESC;
```
Output (first 4 rows) — categories alphabetically (Electronics first),
and *within* Electronics, most expensive first:
| name | category | price |
|---|---|---|
| Laptop Pro 15 | Electronics | 1299.99 |
| Monitor 27" | Electronics | 399.99 |
| Mechanical Keyboard | Electronics | 129.99 |
| USB-C Hub | Electronics | 49.99 |

```sql
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
Output (first 4 of 10 rows) — the three employees with `manager_id IS NULL`
(top-level managers) are pushed to the bottom instead of the top, because
of `NULLS LAST`:
| name | manager_id |
|---|---|
| John Smith | 1 |
| Jane Doe | 1 |
| Nina Patel | 1 |
| Lisa Chen | 4 |

---

## LIMIT and OFFSET — Pagination

```sql
-- Return only 5 rows
SELECT * FROM products ORDER BY price DESC LIMIT 5;
```
Output — the 5 most expensive products, full stop:
| name | price |
|---|---|
| Laptop Pro 15 | 1299.99 |
| Standing Desk | 599.99 |
| Ergonomic Chair | 449.99 |
| Monitor 27" | 399.99 |
| Mechanical Keyboard | 129.99 |

```sql
-- Skip first 5, return next 5 (page 2 of 5-per-page)
SELECT * FROM products ORDER BY price DESC LIMIT 5 OFFSET 5;
```
Output — the *next* 5 products after skipping the 5 above (page 2):
| name | price |
|---|---|
| USB-C Hub | 49.99 |
| Desk Lamp | 39.99 |
| Wireless Mouse | 29.99 |
| Pen Set | 9.99 |
| Notebook (paper) | 4.99 |

```sql
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
```
Output — 10 customer rows collapse down to just the distinct countries:
| country |
|---|
| CA |
| ES |
| KR |
| UK |
| US |

```sql
-- All unique categories
SELECT DISTINCT category FROM products;

-- DISTINCT on multiple columns — unique combinations
SELECT DISTINCT country, city FROM customers ORDER BY country, city;
```
Output — unique *combinations*, not unique per column (both Toronto rows
are Canadian, so they collapse to one; but UK has two distinct cities):
| country | city |
|---|---|
| CA | Toronto |
| ES | Madrid |
| KR | Seoul |
| UK | London |
| US | Chicago |
| US | New York |
| US | San Francisco |

```sql
-- DISTINCT with COUNT — how many unique countries?
SELECT COUNT(DISTINCT country) AS unique_countries FROM customers;
```
Output:
| unique_countries |
|---|
| 5 |

---

## Working With Text

```sql
-- Case-insensitive comparison using LOWER() or ILIKE
SELECT * FROM customers WHERE LOWER(name) = 'alice johnson';
SELECT * FROM customers WHERE name ILIKE 'alice johnson';  -- PostgreSQL-specific
```
Output — both match regardless of how the stored name is capitalized:
| name | email |
|---|---|
| Alice Johnson | alice@example.com |

```sql
-- String length
SELECT name, LENGTH(name) AS name_length FROM customers ORDER BY name_length DESC;
```
Output (first 3 of 10 rows, longest name first):
| name | name_length |
|---|---|
| Eva Martinez | 12 |
| Alice Johnson | 13 |
| Henry Wilson | 12 |

```sql
-- Substring
SELECT SUBSTRING(email, 1, 5) AS email_start FROM customers;
-- Or: SUBSTR(email, 1, 5)

-- Left and right
SELECT LEFT(email, 5), RIGHT(email, 4) FROM customers;
```
Output (first 2 rows) — first 5 / last 4 characters of the email string:
| left | right |
|---|---|
| alice | .com |
| bob@e | .com |

```sql
-- Trim whitespace
SELECT TRIM('  hello world  ');       -- 'hello world'
SELECT LTRIM('  hello world  ');      -- 'hello world  '
SELECT RTRIM('  hello world  ');      -- '  hello world'

-- Upper and lower
SELECT UPPER(name), LOWER(email) FROM customers;
```
Output (first 2 rows):
| upper | lower |
|---|---|
| ALICE JOHNSON | alice@example.com |
| BOB SMITH | bob@example.com |

```sql
-- Replace
SELECT REPLACE(email, '@example.com', '@newdomain.com') FROM customers;
```
Output (first 2 rows):
| replace |
|---|
| alice@newdomain.com |
| bob@newdomain.com |

```sql
-- Concatenation
SELECT first_name || ' ' || last_name AS full_name FROM some_table;
-- Or using CONCAT function
SELECT CONCAT(name, ' from ', city) AS description FROM customers;
-- CONCAT ignores NULLs; || propagates NULL (NULL || 'x' = NULL)
SELECT CONCAT_WS(', ', name, city, country) AS address FROM customers;
-- CONCAT_WS: concatenate with separator, skips NULLs
```
Output of the `CONCAT_WS` line (first 2 rows):
| address |
|---|
| Alice Johnson, New York, US |
| Bob Smith, London, UK |

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
```
Output for one row (`price = 29.99`, Wireless Mouse) — each column is an
independent transformation of the same starting value:
| price | with_sales_tax | rounded | floor_price | ceil_price | absolute | squared | sqrt_price | remainder |
|---|---|---|---|---|---|---|---|---|
| 29.99 | 32.3892 | 32.39 | 29 | 30 | 29.99 | 899.4001 | 5.4763... | 9.99 |

```sql
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
```
Output for order id=1 (`order_date = 2024-01-05`, a Friday):
| order_date | year | month | day | day_of_week | quarter |
|---|---|---|---|---|---|
| 2024-01-05 | 2024 | 1 | 5 | 5 | 1 |

```sql
-- Date arithmetic
SELECT
    order_date,
    order_date + 30         AS plus_30_days,
    order_date - 7          AS minus_7_days,
    NOW() - order_date      AS age,           -- interval type
    NOW()::date - order_date AS days_since    -- integer
FROM orders;
```
Output for order id=1, run "today" (2026-09-09):
| order_date | plus_30_days | minus_7_days | age | days_since |
|---|---|---|---|---|
| 2024-01-05 | 2024-02-04 | 2023-12-29 | 979 days | 979 |

```sql
-- Date truncation
SELECT
    DATE_TRUNC('month', order_date)  AS month_start,
    DATE_TRUNC('year',  order_date)  AS year_start,
    DATE_TRUNC('week',  order_date)  AS week_start
FROM orders;
```
Output for order id=1 (`2024-01-05`) — each rounds *down* to the start
of its respective period:
| month_start | year_start | week_start |
|---|---|---|
| 2024-01-01 | 2024-01-01 | 2024-01-01 |

```sql
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
Output:
| name | category | price |
|---|---|---|
| Notebook (paper) | Stationery | 4.99 |
| Pen Set | Stationery | 9.99 |
| Wireless Mouse | Electronics | 29.99 |
| Desk Lamp | Furniture | 39.99 |
| USB-C Hub | Electronics | 49.99 |

**Example 2 — Top 3 most expensive products:**
```sql
SELECT name, price
FROM products
ORDER BY price DESC
LIMIT 3;
```
Output:
| name | price |
|---|---|
| Laptop Pro 15 | 1299.99 |
| Standing Desk | 599.99 |
| Ergonomic Chair | 449.99 |

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
Output — two customers match both `country = 'US'` and `city = 'New York'`:
| name | email | city | member_info |
|---|---|---|---|
| Alice Johnson | alice@example.com | New York | Member since: 2024 |
| David Brown | david@example.com | New York | Member since: 2024 |

**Example 4 — Orders from January 2024:**
```sql
SELECT id, customer_id, order_date, total
FROM orders
WHERE order_date >= '2024-01-01'
  AND order_date < '2024-02-01'
ORDER BY order_date;
```
Output — 4 of the 10 sample orders fall inside January:
| id | customer_id | order_date | total |
|---|---|---|---|
| 1 | 1 | 2024-01-05 | 1329.98 |
| 3 | 2 | 2024-01-10 | 129.99 |
| 7 | 6 | 2024-01-15 | 399.99 |
| 10 | 9 | 2024-01-22 | 59.98 |

**Example 5 — Delivered or shipped orders, total > 100:**
```sql
SELECT id, customer_id, status, total
FROM orders
WHERE (status = 'delivered' OR status = 'shipped')
  AND total > 100
ORDER BY total DESC;
```
Output — `cancelled`/`pending` orders are excluded regardless of total,
and delivered/shipped orders ≤ $100 are excluded too:
| id | customer_id | status | total |
|---|---|---|---|
| 1 | 1 | delivered | 1329.98 |
| 8 | 7 | delivered | 1299.99 |
| 4 | 3 | shipped | 449.99 |
| 9 | 8 | shipped | 179.98 |
| 3 | 2 | delivered | 129.99 |

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
Output — only 3 products fall in the $30–$150 range; sorted by total
inventory value (`price × stock`), not price itself:
| name | price | tax | total_with_tax | stock | inventory_value |
|---|---|---|---|---|---|
| Mechanical Keyboard | 129.99 | 10.40 | 140.39 | 75 | 9749.25 |
| USB-C Hub | 49.99 | 4.00 | 53.99 | 150 | 7498.50 |
| Desk Lamp | 39.99 | 3.20 | 43.19 | 100 | 3999.00 |

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
Output — only 3 of the 10 employees were hired in 2021 or later
(`days_employed` shown here as run "today"; it grows every day you run this):
| name | department | salary | hire_date | days_employed |
|---|---|---|---|---|
| Lisa Chen | Sales | 68000 | 2021-02-28 | ~2020 |
| Nina Patel | Engineering | 88000 | 2021-09-01 | ~1835 |
| Chris Park | HR | 60000 | 2022-01-05 | ~1708 |

**Example 8 — Employees without a manager (top-level):**
```sql
SELECT name, department, salary
FROM employees
WHERE manager_id IS NULL
ORDER BY salary DESC;
```
Output — the 3 employees with no manager (the top of each department's
hierarchy), highest salary first:
| name | department | salary |
|---|---|---|
| Sarah Connor | Engineering | 120000 |
| Mike Johnson | Sales | 75000 |
| Anna White | HR | 65000 |

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
