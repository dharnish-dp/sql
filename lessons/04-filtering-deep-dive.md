# Lesson 04 — Filtering Deep Dive

## Goal
Master every filtering technique in SQL. After this lesson, no WHERE clause
will be beyond you.

## Prerequisites
- [Lesson 03](03-data-types-and-schema.md) — data types

## After This Lesson You Will Be Able To
- Use LIKE and ILIKE for pattern matching
- Use IN, NOT IN, BETWEEN for range and set filtering
- Write complex multi-condition WHERE clauses
- Use CASE expressions for conditional logic
- Understand the three-valued logic of SQL (TRUE / FALSE / NULL)

---

## WHERE — The Full Picture

The WHERE clause filters rows before any SELECT computation.
Every condition in WHERE must evaluate to TRUE for the row to be included.

### Comparison Operators

```sql
-- All standard comparisons
SELECT * FROM products WHERE price = 29.99;
SELECT * FROM products WHERE price != 29.99;    -- not equal
SELECT * FROM products WHERE price <> 29.99;    -- same as !=
SELECT * FROM products WHERE price > 100;
SELECT * FROM products WHERE price >= 100;
SELECT * FROM products WHERE price < 50;
SELECT * FROM products WHERE price <= 50;

-- Comparing text (alphabetical order)
SELECT * FROM customers WHERE name > 'M';  -- names after 'M' alphabetically
SELECT * FROM employees WHERE department = 'Engineering';

-- Comparing dates
SELECT * FROM orders WHERE order_date > '2024-02-01';
SELECT * FROM orders WHERE order_date = CURRENT_DATE;
```

**Output** for `SELECT * FROM products WHERE price > 100;`:

| id | name       | category    | price  | stock |
|----|------------|-------------|--------|-------|
| 2  | Laptop     | Electronics | 999.99 | 15    |
| 5  | Desk Chair | Furniture   | 149.99 | 8     |

Only rows where `price` is strictly greater than 100 survive — a
product at exactly `100.00` would NOT appear here (use `>=` for that).

---

## BETWEEN — Inclusive Range

```sql
-- BETWEEN is inclusive on BOTH ends
SELECT * FROM products WHERE price BETWEEN 50 AND 200;
-- Exactly equivalent to:
SELECT * FROM products WHERE price >= 50 AND price <= 200;

-- Date range
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31';

-- NOT BETWEEN
SELECT * FROM products WHERE price NOT BETWEEN 50 AND 200;

-- GOTCHA: BETWEEN with TIMESTAMP
-- '2024-01-31' as a timestamp is '2024-01-31 00:00:00'
-- Orders on January 31 at any time AFTER midnight are excluded!
SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';
-- The safe way for full days:
SELECT * FROM orders
WHERE order_date >= '2024-01-01'
  AND order_date <  '2024-02-01';   -- exclusive upper bound
```

**Output** for `SELECT * FROM products WHERE price BETWEEN 50 AND 200;`:

| id | name       | category    | price  |
|----|------------|-------------|--------|
| 3  | Headphones | Electronics | 79.99  |
| 5  | Desk Chair | Furniture   | 149.99 |

Both `50` and `200` themselves would be included if a row matched
exactly — that's what "inclusive on both ends" means in practice.

---

## IN — Match Against a List

```sql
-- IN matches any value in the list
SELECT * FROM customers WHERE country IN ('US', 'UK', 'CA');
-- Same as:
SELECT * FROM customers WHERE country = 'US'
   OR country = 'UK'
   OR country = 'CA';

-- NOT IN
SELECT * FROM customers WHERE country NOT IN ('US', 'UK', 'CA');

-- IN with numbers
SELECT * FROM orders WHERE customer_id IN (1, 3, 5, 7, 9);
```

**Output** for `SELECT * FROM customers WHERE country IN ('US', 'UK', 'CA');`:

| id | name  | country |
|----|-------|---------|
| 1  | Alice | US      |
| 2  | Bob   | UK      |
| 4  | Dan   | CA      |

Carol (country = `'FR'`) is excluded — her country isn't in the list.

```sql

-- IN with subquery (covered deeply in Lesson 07)
SELECT * FROM customers
WHERE id IN (SELECT DISTINCT customer_id FROM orders WHERE total > 500);

-- GOTCHA: NOT IN with NULL
-- If any value in the list is NULL, NOT IN returns no rows at all!
SELECT * FROM employees WHERE id NOT IN (1, 2, NULL);
-- Returns NOTHING because NULL makes the IN check indeterminate

-- Safe pattern: use NOT EXISTS instead (Lesson 07)
```

**Output** for `SELECT * FROM employees WHERE id NOT IN (1, 2, NULL);`:

| id | name |
|----|------|
| *(no rows returned)* | |

Even though employees with `id = 3, 4, 5...` clearly aren't `1` or `2`,
**zero rows come back** — the `NULL` in the list poisons the entire
comparison, because SQL can't prove any row is definitely NOT equal to
an unknown value. This is exactly the gotcha the comment above warns
about.

---

## LIKE and ILIKE — Pattern Matching

### Wildcards
- `%` — matches any sequence of zero or more characters
- `_` — matches exactly one character

```sql
-- Starts with 'A'
SELECT * FROM customers WHERE name LIKE 'A%';

-- Ends with '.com'
SELECT * FROM customers WHERE email LIKE '%.com';

-- Contains 'lee' anywhere
SELECT * FROM customers WHERE name LIKE '%lee%';  -- case-sensitive

-- ILIKE — case-insensitive (PostgreSQL extension)
SELECT * FROM customers WHERE name ILIKE '%alice%';  -- finds 'Alice', 'ALICE', etc.
```

**Output** for `SELECT * FROM customers WHERE name LIKE 'A%';`:

| id | name  |
|----|-------|
| 1  | Alice |
| 6  | Adam  |

`LIKE 'A%'` matches names starting with a capital `A` exactly — a name
like `'alice'` (lowercase) would NOT match, because `LIKE` is
case-sensitive. `ILIKE 'a%'` would match both.

```sql

-- Exactly 5 characters
SELECT * FROM products WHERE name LIKE '_____';  -- 5 underscores

-- Second character is 'o'
SELECT * FROM customers WHERE name LIKE '_o%';

-- Escape the wildcard character itself using ESCAPE
SELECT * FROM products WHERE description LIKE '%50\%%' ESCAPE '\';
-- Matches descriptions containing '50%' literally

-- NOT LIKE
SELECT * FROM customers WHERE email NOT LIKE '%@gmail.com';

-- Multiple LIKE conditions
SELECT * FROM customers
WHERE name ILIKE '%john%'
   OR name ILIKE '%jane%';
```

### SIMILAR TO — Regex-like patterns (rarely used)

```sql
-- SIMILAR TO uses SQL regex syntax (more limited than POSIX)
SELECT * FROM customers WHERE name SIMILAR TO '(Alice|Bob)%';

-- POSIX regex (more powerful, PostgreSQL-specific)
SELECT * FROM customers WHERE name ~ '^A';       -- starts with A (case-sensitive)
SELECT * FROM customers WHERE name ~* '^alice';  -- case-insensitive regex
SELECT * FROM customers WHERE name !~ '^A';      -- does NOT match
SELECT * FROM customers WHERE name !~* '^alice'; -- case-insensitive NOT match

-- Extract with regex
SELECT REGEXP_REPLACE(phone, '[^0-9]', '', 'g') AS digits_only FROM customers;
```

---

## IS NULL / IS NOT NULL

```sql
-- Find rows where a column has no value
SELECT * FROM employees WHERE manager_id IS NULL;     -- top-level managers
SELECT * FROM orders WHERE customer_id IS NULL;       -- orphaned orders
SELECT * FROM products WHERE description IS NULL;     -- no description

-- Find rows where a column HAS a value
SELECT * FROM employees WHERE manager_id IS NOT NULL;
SELECT * FROM customers WHERE phone IS NOT NULL;
```

**Output** for `SELECT * FROM employees WHERE manager_id IS NULL;`:

| id | name  | department | manager_id |
|----|-------|------------|------------|
| 1  | Grace | Executive  | NULL       |

Only the top of the org chart has no manager — this is the same
`manager_id IS NULL` pattern used as the base case of the recursive CTE
in [Lesson 07](07-subqueries-and-ctes.md).

```sql

-- Combine with other conditions
SELECT * FROM employees
WHERE manager_id IS NULL
  AND salary > 100000;

-- IS DISTINCT FROM: handles NULL-safe equality
SELECT * FROM orders WHERE status IS DISTINCT FROM 'cancelled';
-- This is NULL-safe: also returns rows where status IS NULL
-- (Unlike status != 'cancelled' which would exclude NULLs)

SELECT * FROM orders WHERE status IS NOT DISTINCT FROM NULL;
-- Returns rows where status IS NULL
```

**Output** for `SELECT * FROM orders WHERE status IS DISTINCT FROM 'cancelled';`:

| id | status    | total  |
|----|-----------|--------|
| 1  | delivered | 999.99 |
| 3  | NULL      | 129.99 |
| 4  | pending   | 299.99 |

Notice the `NULL` status row is **included** here — that's the whole
point of `IS DISTINCT FROM`. Compare with plain `status != 'cancelled'`:
that version would silently drop the `NULL` row too, since `NULL != anything`
evaluates to `NULL` (neither true nor false), and `WHERE` only keeps
rows where the condition is exactly `TRUE`.

---

## AND, OR, NOT — Logical Operators

### Operator Precedence (high to low)
1. NOT
2. AND
3. OR

```sql
-- This means: country = 'US' AND (category = 'Electronics' OR price < 30)
-- NOT what was intended!
SELECT * FROM products
WHERE country = 'US' AND category = 'Electronics' OR price < 30;

-- Use parentheses — ALWAYS:
SELECT * FROM products
WHERE country = 'US'
  AND (category = 'Electronics' OR price < 30);
```

**Why the first query is a bug, shown concretely.** Say `products` has
a $20 item with `country = 'FR'` (not US) and `category = 'Furniture'`.
Because `AND` binds tighter than `OR`, the unparenthesized version reads
as `(country='US' AND category='Electronics') OR price < 30` — so this
French $20 furniture item **still matches**, purely from the `price < 30`
branch, regardless of country:

| id | name        | country | category  | price |
|----|-------------|---------|-----------|-------|
| 9  | Cheap Vase  | FR      | Furniture | 20.00 |

That's almost certainly not what "US electronics OR cheap items" was
meant to express. The parenthesized version correctly excludes this row,
since it requires `country = 'US'` no matter which OR-branch fires.

```sql

-- Complex conditions
SELECT * FROM employees
WHERE department IN ('Engineering', 'Sales')
  AND salary > 70000
  AND hire_date >= '2020-01-01'
  AND (manager_id IS NOT NULL OR department = 'Engineering');

-- NOT with IN
SELECT * FROM customers WHERE NOT (country = 'US' OR country = 'UK');
-- Same as:
SELECT * FROM customers WHERE country NOT IN ('US', 'UK');
```

---

## CASE Expression — Conditional Logic in SQL

CASE is the SQL equivalent of if/else. It works inside SELECT, WHERE, ORDER BY.

### Simple CASE (equality checks)

```sql
SELECT
    name,
    status,
    CASE status
        WHEN 'pending'   THEN 'Waiting for processing'
        WHEN 'shipped'   THEN 'On the way'
        WHEN 'delivered' THEN 'Completed'
        WHEN 'cancelled' THEN 'Cancelled — no action needed'
        ELSE             'Unknown status'
    END AS status_description
FROM orders;
```

**Output:**

| name  | status    | status_description       |
|-------|-----------|---------------------------|
| Alice | delivered | Completed                 |
| Bob   | pending   | Waiting for processing    |
| Carol | shipped   | On the way                |

Each row's `status` picks exactly one `WHEN` branch — `CASE` here just
translates a raw status code into a human-readable label per row.

### Searched CASE (general conditions)

```sql
-- Categorize products by price
SELECT
    name,
    price,
    CASE
        WHEN price < 30               THEN 'Budget'
        WHEN price BETWEEN 30 AND 100 THEN 'Mid-range'
        WHEN price BETWEEN 100 AND 500 THEN 'Premium'
        ELSE                               'Luxury'
    END AS price_tier
FROM products
ORDER BY price;
```

**Output:**

| name       | price  | price_tier |
|------------|--------|------------|
| Notebook   | 4.99   | Budget     |
| Headphones | 79.99  | Mid-range  |
| Desk Chair | 149.99 | Premium    |
| Laptop     | 999.99 | Luxury     |

```sql

-- Compute a discount based on status and quantity
SELECT
    product_id,
    quantity,
    unit_price,
    CASE
        WHEN quantity >= 10  THEN unit_price * 0.80  -- 20% off
        WHEN quantity >= 5   THEN unit_price * 0.90  -- 10% off
        ELSE                      unit_price          -- no discount
    END AS discounted_price
FROM order_items;

-- CASE in WHERE clause
SELECT * FROM employees
WHERE CASE department
          WHEN 'Engineering' THEN salary > 90000
          WHEN 'Sales'       THEN salary > 65000
          ELSE                    salary > 50000
      END;
```

**Output** (assuming Alice is Engineering/95000, Bob is Sales/70000, Carol is Marketing/55000):

| name  | department | salary |
|-------|------------|--------|
| Alice | Engineering| 95000  |
| Bob   | Sales      | 70000  |
| Carol | Marketing  | 55000  |

Each row runs its **own** threshold depending on its department: Alice's
95000 clears Engineering's 90000 bar, Bob's 70000 clears Sales' 65000
bar, Carol's 55000 clears the `ELSE` (default) 50000 bar. A Marketing
employee earning 48000 would be filtered OUT — it fails the `ELSE`
branch's `salary > 50000` check.

```sql
-- CASE in ORDER BY
SELECT name, salary FROM employees
ORDER BY
    CASE department
        WHEN 'Engineering' THEN 1
        WHEN 'Sales'       THEN 2
        ELSE                    3
    END,
    salary DESC;

-- CASE for conditional aggregation (preview — Lesson 05)
SELECT
    COUNT(*)                                     AS total_orders,
    COUNT(CASE WHEN status = 'delivered' THEN 1 END) AS delivered,
    COUNT(CASE WHEN status = 'pending'   THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'cancelled' THEN 1 END) AS cancelled
FROM orders;
```

---

## EXISTS — Check for Existence

```sql
-- Find customers who have placed at least one order
SELECT c.name, c.email
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
-- EXISTS returns TRUE if the subquery returns any row at all
-- The SELECT 1 is a convention — the actual value doesn't matter
```

**Output** (assuming Carol has never placed an order):

| name  | email             |
|-------|-------------------|
| Alice | alice@example.com|
| Bob   | bob@example.com  |

Carol is excluded — the inner subquery finds zero matching `orders` rows
for her, so `EXISTS (...)` evaluates to `FALSE` for her row.

```sql

-- Find customers who have NOT placed any order
SELECT c.name, c.email
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);

-- EXISTS vs IN — they're often equivalent, but:
-- EXISTS: short-circuits (stops after first match) → usually faster
-- NOT EXISTS: handles NULL correctly (unlike NOT IN)
-- EXISTS: more readable when the correlation is complex
```

---

## ANY / ALL — Compare Against a Set

```sql
-- ANY: true if condition holds for at least one value
SELECT * FROM products
WHERE price > ANY (SELECT total FROM orders WHERE status = 'pending');
-- Products more expensive than the cheapest pending order
```

**Output** (assuming pending order totals are `50.00` and `299.99` — so
the cheapest is `50.00`):

| id | name       | price  |
|----|------------|--------|
| 2  | Laptop     | 999.99 |
| 3  | Headphones | 79.99  |
| 5  | Desk Chair | 149.99 |

Every product priced above `50.00` (the smallest value in the pending
orders subquery) qualifies for `> ANY` — it only needs to beat **one**
value in the set, not all of them.

```sql

-- ALL: true only if condition holds for ALL values
SELECT * FROM products
WHERE price > ALL (SELECT total FROM orders);
-- Products more expensive than ALL orders (probably nothing)

-- = ANY is same as IN
SELECT * FROM customers WHERE id = ANY (ARRAY[1, 2, 3]);
SELECT * FROM customers WHERE id = ANY (SELECT customer_id FROM orders);
```

---

## Filtering on Computed Values

```sql
-- You can't use SELECT aliases in WHERE (runs before SELECT)
-- WRONG:
SELECT price * 1.08 AS total_price FROM products WHERE total_price > 50;

-- CORRECT — repeat the expression:
SELECT price * 1.08 AS total_price FROM products WHERE price * 1.08 > 50;
```

**The WRONG version fails with:**
```
ERROR:  column "total_price" does not exist
```
**The CORRECT version's output** (for a $79.99 product):

| total_price |
|--------------|
| 86.39        |

Since `WHERE` runs before `SELECT` computes any aliases (see the
execution order in [Lesson 05](05-aggregations.md)), `total_price`
simply doesn't exist yet at the point `WHERE` is evaluated — the alias
is only created afterward, in the `SELECT` step.

```sql

-- Or use a subquery / CTE:
SELECT * FROM (
    SELECT name, price * 1.08 AS total_price FROM products
) sub
WHERE total_price > 50;

-- Or CTE (cleaner):
WITH priced AS (
    SELECT name, price * 1.08 AS total_price FROM products
)
SELECT * FROM priced WHERE total_price > 50;
```

---

## Practical Filtering Patterns

### Date Range Queries (Safe Pattern)

```sql
-- Inclusive month range (January 2024)
SELECT * FROM orders
WHERE order_date >= DATE_TRUNC('month', '2024-01-01'::DATE)
  AND order_date <  DATE_TRUNC('month', '2024-01-01'::DATE) + INTERVAL '1 month';

-- Last 30 days
SELECT * FROM orders
WHERE order_date >= NOW() - INTERVAL '30 days';

-- Current week
SELECT * FROM orders
WHERE order_date >= DATE_TRUNC('week', NOW())::DATE;

-- Year to date
SELECT * FROM orders
WHERE order_date >= DATE_TRUNC('year', NOW())::DATE;
```

### Text Search Patterns

```sql
-- Find email domains
SELECT * FROM customers WHERE email LIKE '%@gmail.com';

-- Find names starting with vowels
SELECT * FROM customers WHERE name ~* '^[AEIOU]';

-- Find products with numbers in their names
SELECT * FROM products WHERE name ~ '[0-9]';
```

### Numeric Range Patterns

```sql
-- Percentile approximation using offset/limit
-- Bottom 20% of products by price
SELECT * FROM products
ORDER BY price
LIMIT (SELECT CEIL(COUNT(*) * 0.20) FROM products);

-- Products within 10% of average price
SELECT name, price
FROM products
WHERE price BETWEEN
    (SELECT AVG(price) * 0.90 FROM products)
    AND
    (SELECT AVG(price) * 1.10 FROM products);
```

---

## Complete Query Examples

**Example 1 — Advanced employee filter:**
```sql
SELECT
    name,
    department,
    salary,
    hire_date,
    CASE
        WHEN salary >= 100000 THEN 'Senior'
        WHEN salary >= 75000  THEN 'Mid-level'
        ELSE                       'Junior'
    END AS seniority
FROM employees
WHERE department IN ('Engineering', 'Sales')
  AND hire_date >= '2019-01-01'
  AND salary BETWEEN 60000 AND 120000
  AND manager_id IS NOT NULL
ORDER BY department, salary DESC;
```

**Output** (sample rows matching all four conditions):

| name  | department  | salary | hire_date  | seniority |
|-------|-------------|--------|------------|-----------|
| Alice | Engineering | 95000  | 2019-03-15 | Senior    |
| Dan   | Engineering | 82000  | 2020-07-01 | Mid-level |
| Bob   | Sales       | 70000  | 2021-02-10 | Junior    |

Every row satisfies all four `WHERE` conditions at once (department,
hire date, salary range, and having a manager) — then `CASE` labels
each surviving row's seniority based purely on its own salary.

**Example 2 — Product availability with stock category:**
```sql
SELECT
    name,
    category,
    price,
    stock,
    CASE
        WHEN stock = 0     THEN 'Out of Stock'
        WHEN stock < 10    THEN 'Low Stock'
        WHEN stock < 50    THEN 'In Stock'
        ELSE                    'Well Stocked'
    END AS availability,
    CASE
        WHEN price < 50   THEN price * 0.05
        WHEN price < 200  THEN price * 0.10
        ELSE                   price * 0.15
    END AS discount_amount
FROM products
WHERE stock > 0
  AND category NOT IN ('Stationery')
ORDER BY stock ASC, price DESC;
```

**Example 3 — Orders from Q1 2024 that are large or small:**
```sql
SELECT
    id,
    customer_id,
    order_date,
    total,
    status,
    CASE WHEN total >= 500 THEN 'Large' ELSE 'Small' END AS order_size
FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-03-31'
  AND status != 'cancelled'
  AND total IS NOT NULL
ORDER BY order_date DESC, total DESC;
```

**Example 4 — Pattern matching for email domains:**
```sql
SELECT
    name,
    email,
    CASE
        WHEN email ILIKE '%@gmail.com'    THEN 'Gmail'
        WHEN email ILIKE '%@outlook.com'  THEN 'Outlook'
        WHEN email ILIKE '%@yahoo.com'    THEN 'Yahoo'
        ELSE                                   'Other'
    END AS email_provider
FROM customers
ORDER BY email_provider, name;
```

---

## Exercises

**Exercise 1:** Find all products with names containing the word "Pro" or "USB".
Show name, category, price. Use ILIKE for case-insensitive matching.

**Exercise 2:** List employees in Engineering or HR who earn between $70,000 and $100,000
and were hired before 2022. Show their name, department, salary, and a CASE-based label:
'High' for salary > 90000, 'Medium' for 75000–90000, 'Standard' for below.

**Exercise 3:** Find orders where:
- Status is 'shipped' or 'delivered'
- Total is above 100
- Order date is in 2024
Show order id, date, status, total, and a column showing whether total >= 500 as
'Big Order' or 'Normal Order'.

**Exercise 4:** Find customers from cities whose name starts with 'S' or ends with 'o'.

**Exercise 5:** List products with stock between 20 and 100, OR
products in 'Stationery' category with any stock level. Order by category then price.

---

## Key Takeaways

1. `BETWEEN` is inclusive on both ends — be careful with timestamp ranges
2. `NOT IN (list)` returns no results if list contains a NULL — use NOT EXISTS
3. `LIKE` is case-sensitive; `ILIKE` is PostgreSQL's case-insensitive version
4. `%` matches anything; `_` matches exactly one character in LIKE
5. CASE expressions work inside SELECT, WHERE, ORDER BY, HAVING
6. Always parenthesize mixed AND/OR conditions
7. `IS DISTINCT FROM` is NULL-safe equality checking

---

## Next Lesson
[Lesson 05 — Aggregations: GROUP BY, HAVING, COUNT, SUM, AVG](05-aggregations.md)
