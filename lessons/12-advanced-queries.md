# Lesson 12 — Advanced Queries

## Goal
Master the advanced SQL constructs that turn good SQL into great SQL:
set operations, pivoting, lateral joins, and deep dives into string,
date, and mathematical functions.

## Prerequisites
- [Lesson 11](11-database-design-normalization.md) — normalization

## After This Lesson You Will Be Able To
- Combine result sets with UNION, INTERSECT, EXCEPT
- Pivot rows into columns (and unpivot columns into rows)
- Use LATERAL for powerful correlated subqueries in FROM
- Apply all important string, date, and math functions
- Use GENERATE_SERIES for data generation
- Write DISTINCT ON queries

---

## Set Operations

### UNION — Combine Results

```sql
-- UNION: removes duplicates (like DISTINCT)
-- UNION ALL: keeps duplicates (faster, use when no duplicates expected)

-- All customer emails and employee emails combined
SELECT email, 'customer'  AS source FROM customers
UNION
SELECT email, 'employee'  AS source FROM employees
ORDER BY email;

-- UNION ALL (keep duplicates, more performant)
SELECT name FROM customers
UNION ALL
SELECT name FROM employees;
-- Alice and an employee named Alice both appear

-- Column count and types must match across all SELECT statements
-- Aliases come from the FIRST SELECT statement
SELECT id, name, 'customer' AS type FROM customers
UNION ALL
SELECT id, name, 'employee'         FROM employees;

-- UNION with different column sets — use NULL for missing columns
SELECT id, name, email,   NULL AS department FROM customers
UNION ALL
SELECT id, name, NULL,    department         FROM employees;
```

### INTERSECT — Common Rows

```sql
-- Emails that appear in BOTH tables (people who are both customers and employees)
SELECT email FROM customers
INTERSECT
SELECT email FROM employees;

-- Products that have been ordered AND are in stock
SELECT id FROM products WHERE stock > 0
INTERSECT
SELECT DISTINCT product_id FROM order_items;
```

### EXCEPT — Rows in First But Not Second

```sql
-- Customers who have NEVER placed an order
SELECT id FROM customers
EXCEPT
SELECT DISTINCT customer_id FROM orders;

-- Products in the catalog but never ordered
SELECT id FROM products
EXCEPT
SELECT DISTINCT product_id FROM order_items;

-- Note: EXCEPT ALL keeps duplicates (like UNION ALL vs UNION)
```

### Combining All Three

```sql
-- Multi-level: US and UK customers, minus those who cancelled all orders
(SELECT id FROM customers WHERE country = 'US'
 UNION
 SELECT id FROM customers WHERE country = 'UK')
EXCEPT
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING COUNT(*) = COUNT(CASE WHEN status = 'cancelled' THEN 1 END);
```

---

## DISTINCT ON — Keep First Row Per Group

PostgreSQL-specific. Takes the first row for each distinct value of specified columns.

```sql
-- Most recent order per customer
SELECT DISTINCT ON (customer_id)
    customer_id,
    id AS order_id,
    order_date,
    total,
    status
FROM orders
ORDER BY customer_id, order_date DESC;
-- DISTINCT ON keeps the FIRST row in each group after ORDER BY
-- ORDER BY must start with the DISTINCT ON columns

-- Most expensive product per category
SELECT DISTINCT ON (category)
    category,
    name,
    price
FROM products
ORDER BY category, price DESC;

-- Latest employee in each department
SELECT DISTINCT ON (department)
    department,
    name,
    hire_date
FROM employees
ORDER BY department, hire_date DESC;
```

---

## LATERAL — Power Joins

LATERAL is like a correlated subquery in the FROM clause.
The subquery can reference columns from preceding FROM items.

```sql
-- For each customer, get their most recent 2 orders
SELECT
    c.name,
    o.id,
    o.order_date,
    o.total
FROM customers c
CROSS JOIN LATERAL (
    SELECT id, order_date, total
    FROM orders
    WHERE customer_id = c.id    -- ← references c.id from outer FROM
    ORDER BY order_date DESC
    LIMIT 2
) o;

-- Without LATERAL, this subquery couldn't reference c.id

-- LATERAL with LEFT JOIN (include customers with no orders)
SELECT
    c.name,
    o.id,
    o.order_date
FROM customers c
LEFT JOIN LATERAL (
    SELECT id, order_date
    FROM orders
    WHERE customer_id = c.id
    ORDER BY order_date DESC
    LIMIT 1
) o ON TRUE;  -- ON TRUE because the correlation is inside the subquery

-- Unnest array with row context
SELECT
    p.name,
    tag
FROM products p
CROSS JOIN LATERAL UNNEST(p.tags) AS tag
WHERE tag IS NOT NULL;
-- Each tag becomes its own row, keeping the product context
```

---

## Pivoting — Rows to Columns

PostgreSQL doesn't have a built-in PIVOT, but you can do it with CASE + GROUP BY.

```sql
-- Sales by month per status (pivot)
SELECT
    TO_CHAR(order_date, 'YYYY-MM') AS month,
    SUM(CASE WHEN status = 'delivered' THEN total ELSE 0 END) AS delivered_revenue,
    SUM(CASE WHEN status = 'shipped'   THEN total ELSE 0 END) AS shipped_revenue,
    SUM(CASE WHEN status = 'pending'   THEN total ELSE 0 END) AS pending_revenue,
    SUM(CASE WHEN status = 'cancelled' THEN total ELSE 0 END) AS cancelled_revenue,
    SUM(total) AS total_revenue
FROM orders
GROUP BY TO_CHAR(order_date, 'YYYY-MM')
ORDER BY month;

-- Category count by country (pivot)
SELECT
    country,
    COUNT(*) FILTER (WHERE city = 'New York')     AS new_york,
    COUNT(*) FILTER (WHERE city = 'London')       AS london,
    COUNT(*) FILTER (WHERE city = 'Toronto')      AS toronto,
    COUNT(*) FILTER (WHERE city NOT IN ('New York','London','Toronto')) AS other
FROM customers
GROUP BY country;

-- Using crosstab (tablefunc extension — more powerful pivot)
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM crosstab(
    'SELECT department, EXTRACT(YEAR FROM hire_date)::TEXT, COUNT(*)
     FROM employees
     GROUP BY 1, 2
     ORDER BY 1, 2',
    'SELECT DISTINCT EXTRACT(YEAR FROM hire_date)::TEXT FROM employees ORDER BY 1'
) AS ct (department TEXT, "2017" BIGINT, "2018" BIGINT, "2019" BIGINT, "2020" BIGINT, "2021" BIGINT, "2022" BIGINT);
```

---

## GENERATE_SERIES — Generate Data

```sql
-- Generate a sequence of integers
SELECT * FROM generate_series(1, 10);
SELECT * FROM generate_series(0, 100, 5);  -- 0, 5, 10, ..., 100

-- Generate dates
SELECT d::DATE
FROM generate_series(
    '2024-01-01'::TIMESTAMP,
    '2024-12-31'::TIMESTAMP,
    '1 day'::INTERVAL
) d;

-- Generate months
SELECT d::DATE AS month_start
FROM generate_series(
    '2024-01-01'::TIMESTAMP,
    '2024-12-01'::TIMESTAMP,
    '1 month'::INTERVAL
) d;

-- Fill in gaps in time series data
WITH calendar AS (
    SELECT d::DATE AS day
    FROM generate_series(
        '2024-01-01'::TIMESTAMP,
        '2024-03-31'::TIMESTAMP,
        '1 day'
    ) d
),
daily_orders AS (
    SELECT order_date, COUNT(*) AS order_count, SUM(total) AS revenue
    FROM orders
    GROUP BY order_date
)
SELECT
    c.day,
    COALESCE(d.order_count, 0) AS orders,
    COALESCE(d.revenue, 0) AS revenue
FROM calendar c
LEFT JOIN daily_orders d ON d.order_date = c.day
ORDER BY c.day;
-- This shows EVERY day, even days with no orders (COALESCE fills 0)
```

---

## Deep Dive: String Functions

```sql
-- Length
SELECT LENGTH('Hello World');         -- 11
SELECT CHAR_LENGTH('Hello');           -- 5 (synonym)
SELECT BIT_LENGTH('Hello');            -- 40 (bits)
SELECT OCTET_LENGTH('Hello');          -- 5 (bytes)

-- Case
SELECT UPPER('hello');                 -- HELLO
SELECT LOWER('HELLO');                 -- hello
SELECT INITCAP('hello world');         -- Hello World

-- Trim
SELECT TRIM('  spaces  ');             -- 'spaces'
SELECT TRIM(BOTH 'x' FROM 'xxxhelloxxx'); -- 'hello'
SELECT LTRIM('  left  ');              -- 'left  '
SELECT RTRIM('  right  ');             -- '  right'

-- Pad
SELECT LPAD('42', 6, '0');             -- '000042'
SELECT RPAD('yes', 5, '.');            -- 'yes..'
SELECT LPAD('hello', 3);               -- 'hel' (truncates if longer!)

-- Substring
SELECT SUBSTRING('hello world', 1, 5);  -- 'hello' (1-indexed)
SELECT SUBSTRING('hello world' FROM 7); -- 'world'
SELECT SUBSTR('hello', 2, 3);           -- 'ell'

-- Position / Index
SELECT POSITION('world' IN 'hello world');  -- 7
SELECT STRPOS('hello world', 'world');      -- 7 (same)

-- Replace
SELECT REPLACE('hello world', 'world', 'SQL'); -- 'hello SQL'

-- Repeat
SELECT REPEAT('ab', 3);                -- 'ababab'

-- Reverse
SELECT REVERSE('hello');               -- 'olleh'

-- Split
SELECT SPLIT_PART('a,b,c,d', ',', 2);  -- 'b' (2nd element)
SELECT STRING_TO_ARRAY('a,b,c', ',');  -- ARRAY['a','b','c']
SELECT UNNEST(STRING_TO_ARRAY('a,b,c', ',')); -- 3 rows

-- Format
SELECT FORMAT('Hello %s, you have %s messages', 'Alice', 5);
-- 'Hello Alice, you have 5 messages'

-- Regular expressions
SELECT REGEXP_REPLACE('abc123def', '[0-9]+', 'NUM');  -- 'abcNUMdef'
SELECT REGEXP_REPLACE('abc123def456', '[0-9]+', 'N', 'g');  -- 'abcNdefN' (global)
SELECT REGEXP_MATCHES('a1b2c3', '[0-9]', 'g');  -- array of matches
SELECT (REGEXP_MATCH('user@example.com', '^(.+)@(.+)$'))[1]; -- 'user'

-- Encode / Decode
SELECT ENCODE('hello'::BYTEA, 'base64');  -- 'aGVsbG8='
SELECT DECODE('aGVsbG8=', 'base64')::TEXT; -- 'hello'

-- MD5 hash
SELECT MD5('password');  -- '5f4dcc3b5aa765d61d8327deb882cf99'
```

---

## Deep Dive: Date/Time Functions

```sql
-- Current values
SELECT NOW();                         -- current timestamp with timezone
SELECT CURRENT_DATE;                  -- current date
SELECT CURRENT_TIME;                  -- current time
SELECT CURRENT_TIMESTAMP;             -- same as NOW()
SELECT LOCALTIME;                     -- current time without timezone
SELECT LOCALTIMESTAMP;                -- current timestamp without timezone
SELECT CLOCK_TIMESTAMP();             -- actual current time (changes during queries)
SELECT TRANSACTION_TIMESTAMP();       -- time at start of transaction (same as NOW())
SELECT STATEMENT_TIMESTAMP();         -- time at start of statement

-- Extract parts
SELECT EXTRACT(YEAR   FROM NOW());    -- 2024
SELECT EXTRACT(MONTH  FROM NOW());    -- 3
SELECT EXTRACT(DAY    FROM NOW());    -- 15
SELECT EXTRACT(HOUR   FROM NOW());    -- 14
SELECT EXTRACT(MINUTE FROM NOW());    -- 30
SELECT EXTRACT(SECOND FROM NOW());    -- 22.123456
SELECT EXTRACT(DOW    FROM NOW());    -- 5 (0=Sun, 6=Sat)
SELECT EXTRACT(ISODOW FROM NOW());    -- 5 (1=Mon, 7=Sun — ISO standard)
SELECT EXTRACT(DOY    FROM NOW());    -- day of year (1-366)
SELECT EXTRACT(WEEK   FROM NOW());    -- ISO week number
SELECT EXTRACT(QUARTER FROM NOW());   -- 1-4
SELECT EXTRACT(EPOCH  FROM NOW());    -- Unix timestamp (seconds since 1970-01-01)

-- Date_part (older equivalent of EXTRACT)
SELECT DATE_PART('year', NOW());      -- same as EXTRACT

-- Truncate to period boundary
SELECT DATE_TRUNC('year',    NOW());  -- 2024-01-01 00:00:00
SELECT DATE_TRUNC('quarter', NOW());  -- 2024-01-01 (start of Q1)
SELECT DATE_TRUNC('month',   NOW());  -- 2024-03-01
SELECT DATE_TRUNC('week',    NOW());  -- 2024-03-11 (Monday)
SELECT DATE_TRUNC('day',     NOW());  -- 2024-03-15 00:00:00
SELECT DATE_TRUNC('hour',    NOW());  -- 2024-03-15 14:00:00

-- Arithmetic with intervals
SELECT NOW() + INTERVAL '1 year';
SELECT NOW() + INTERVAL '2 months 3 days 4 hours';
SELECT '2024-03-15'::DATE + 30;                    -- +30 days
SELECT '2024-03-15'::DATE - '2024-01-01'::DATE;   -- days difference = 74
SELECT AGE(NOW(), '2000-01-01'::DATE);             -- interval: 24 years 2 months...
SELECT AGE('2024-03-15'::DATE);                    -- age from today

-- Format / Parse
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS');
SELECT TO_CHAR(NOW(), 'Day, DD Month YYYY');       -- 'Friday , 15 March  2024'
SELECT TO_CHAR(NOW(), 'Mon DD, YYYY');             -- 'Mar 15, 2024'
SELECT TO_CHAR(12345.67, 'FM$999,999.00');         -- '$12,345.67'
SELECT TO_DATE('15/03/2024', 'DD/MM/YYYY');
SELECT TO_TIMESTAMP('2024-03-15 14:30', 'YYYY-MM-DD HH24:MI');

-- Timezone
SELECT NOW() AT TIME ZONE 'America/New_York';
SELECT NOW() AT TIME ZONE 'UTC';
SELECT NOW() AT TIME ZONE 'Asia/Seoul';

-- Make date from parts
SELECT MAKE_DATE(2024, 3, 15);            -- 2024-03-15
SELECT MAKE_TIMESTAMP(2024, 3, 15, 14, 30, 0);  -- 2024-03-15 14:30:00
SELECT MAKE_INTERVAL(years=>1, months=>2, days=>3);

-- End of month
SELECT DATE_TRUNC('month', NOW()) + INTERVAL '1 month' - INTERVAL '1 day';
-- First day of next month - 1 day = last day of current month
```

---

## Deep Dive: Mathematical Functions

```sql
-- Basic
SELECT ABS(-42);                  -- 42
SELECT SIGN(-42);                 -- -1 (or 0 or 1)
SELECT MOD(17, 5);                -- 2 (remainder)
SELECT 17 % 5;                    -- 2 (same)
SELECT DIV(17, 5);                -- 3 (integer division)
SELECT 17 / 5;                    -- 3 (integer division in SQL)
SELECT 17.0 / 5;                  -- 3.4

-- Power and roots
SELECT POWER(2, 10);              -- 1024
SELECT 2 ^ 10;                    -- 1024 (PostgreSQL shorthand)
SELECT SQRT(144);                 -- 12
SELECT CBRT(27);                  -- 3 (cube root)
SELECT EXP(1);                    -- 2.71828... (e^x)
SELECT LN(2.71828);               -- ~1 (natural log)
SELECT LOG(100);                  -- 2 (log base 10)
SELECT LOG(2, 1024);              -- 10 (log base 2)

-- Rounding
SELECT ROUND(3.456, 2);          -- 3.46
SELECT ROUND(3.454, 2);          -- 3.45
SELECT TRUNC(3.999, 1);          -- 3.9 (truncate, no rounding)
SELECT FLOOR(3.7);                -- 3 (round down)
SELECT CEIL(3.2);                 -- 4 (round up)
SELECT CEILING(3.2);              -- 4 (same as CEIL)

-- Trigonometry
SELECT SIN(PI() / 2);            -- 1
SELECT COS(0);                   -- 1
SELECT TAN(PI() / 4);            -- 1
SELECT DEGREES(PI());            -- 180
SELECT RADIANS(180);             -- 3.14159...

-- Statistics
SELECT RANDOM();                  -- random float 0.0 to 1.0
-- Seed for reproducible random
SELECT SETSEED(0.5);
SELECT RANDOM(), RANDOM();        -- reproducible after setseed

-- Statistical aggregates
SELECT
    STDDEV(salary)    AS std_deviation,
    VARIANCE(salary)  AS variance,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median,
    PERCENTILE_CONT(0.25) WITHIN GROUP (ORDER BY salary) AS p25,
    PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY salary) AS p75,
    PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY salary) AS p90
FROM employees;

-- MODE — most common value
SELECT MODE() WITHIN GROUP (ORDER BY department) FROM employees;
```

---

## Advanced Aggregation Functions

```sql
-- PERCENTILE_CONT vs PERCENTILE_DISC
-- CONT: interpolates between values (can return non-existent value)
-- DISC: returns actual value from the dataset

SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_interpolated,
    PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY salary) AS median_actual
FROM employees;

-- CORR: correlation coefficient (-1 to 1)
-- Does salary correlate with years of service?
SELECT CORR(salary, EXTRACT(YEAR FROM hire_date)) AS salary_tenure_correlation
FROM employees;

-- COVAR_POP, COVAR_SAMP: covariance
SELECT COVAR_POP(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date))
FROM employees;

-- REGR_* functions for linear regression
SELECT
    REGR_SLOPE(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date))     AS slope,
    REGR_INTERCEPT(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date)) AS intercept,
    REGR_R2(salary, EXTRACT(YEAR FROM NOW()) - EXTRACT(YEAR FROM hire_date))        AS r_squared
FROM employees;
```

---

## Complete Advanced Query Examples

**Example 1 — Complete time series with zero-fill:**
```sql
WITH months AS (
    SELECT m::DATE AS month
    FROM generate_series('2024-01-01', '2024-03-01', '1 month'::INTERVAL) m
),
monthly_orders AS (
    SELECT DATE_TRUNC('month', order_date)::DATE AS month,
           COUNT(*) AS orders,
           SUM(total) AS revenue
    FROM orders
    WHERE status != 'cancelled'
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    m.month,
    COALESCE(mo.orders, 0) AS orders,
    COALESCE(mo.revenue, 0) AS revenue,
    ROUND(mo.revenue / NULLIF(mo.orders, 0), 2) AS avg_order_value
FROM months m
LEFT JOIN monthly_orders mo ON mo.month = m.month
ORDER BY m.month;
```

**Example 2 — Pivoted report:**
```sql
SELECT
    department,
    COUNT(*) AS total_employees,
    SUM(salary) AS total_payroll,
    COUNT(*) FILTER (WHERE hire_date >= '2021-01-01') AS hired_2021_plus,
    COUNT(*) FILTER (WHERE manager_id IS NULL) AS managers,
    ROUND(AVG(salary), 0) AS avg_salary,
    MIN(salary) AS min_salary,
    MAX(salary) AS max_salary
FROM employees
GROUP BY department
ORDER BY total_payroll DESC;
```

---

## Exercises

**Exercise 1:** Use GENERATE_SERIES to create a daily calendar for January 2024.
LEFT JOIN to orders to show orders and revenue per day. Fill missing days with 0.

**Exercise 2:** Use DISTINCT ON to get the most recent order per customer.
Then join to the customers table to show customer name and their last order date.

**Exercise 3:** Using UNION ALL, create a combined list of "transactions":
- Orders with type='order', date=order_date, amount=total
- Products with type='inventory', date=created_at, amount=price * stock (inventory value)
Order by date.

**Exercise 4:** Pivot the employee table:
Show one row per department, with columns for each year of hire (2017-2022),
showing how many employees were hired in each department per year.

---

## Key Takeaways

1. UNION removes duplicates; UNION ALL doesn't — prefer UNION ALL for performance
2. INTERSECT finds common rows; EXCEPT finds rows in first but not second
3. DISTINCT ON (cols) keeps the first row per distinct value — ORDER BY must start with those cols
4. LATERAL lets a FROM subquery reference columns from earlier in the FROM clause
5. GENERATE_SERIES fills time gaps — essential for reporting
6. PERCENTILE_CONT(0.5) is the correct way to compute median in SQL
7. Pivoting uses CASE WHEN inside SUM/COUNT with GROUP BY

---

## Next Lesson
[Lesson 13 — Views & Materialized Views](13-views-and-materialized-views.md)
