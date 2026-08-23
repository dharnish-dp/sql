# Lesson 16 — PostgreSQL Power Features

## Goal
Master the PostgreSQL-specific features that set it apart from every other
database: JSONB, arrays, full-text search, partitioning, and extensions.

## Prerequisites
- [Lesson 15](15-query-optimization.md) — query optimization

## After This Lesson You Will Be Able To
- Store, query, and index JSONB data
- Use PostgreSQL arrays as first-class types
- Implement full-text search
- Partition large tables for performance
- Use useful extensions (uuid-ossp, pg_trgm, pg_cron, PostGIS basics)
- Use COPY for bulk data import/export

---

## JSONB — Semi-Structured Data

JSONB stores JSON in a decomposed binary format:
- Validated on insert
- Supports operators and indexing
- Keys are sorted, duplicates removed
- Always prefer JSONB over JSON

```sql
-- Create a table with JSONB
CREATE TABLE products_v2 (
    id       SERIAL PRIMARY KEY,
    name     TEXT NOT NULL,
    price    NUMERIC(10,2) NOT NULL,
    metadata JSONB DEFAULT '{}'
);

-- Insert JSONB
INSERT INTO products_v2 (name, price, metadata) VALUES
    ('Laptop Pro', 1299.99, '{"color":"silver","weight":1.4,"tags":["sale","popular"],"specs":{"ram":16,"storage":512}}'),
    ('Mouse',       29.99,  '{"color":"black","weight":0.1,"tags":["accessory"],"wireless":true}'),
    ('Monitor',    399.99,  '{"color":"black","size":27,"resolution":"4K","tags":["display","popular"]}');

-- Access operators
-- -> returns JSON value (keeps type)
-- ->> returns TEXT value

SELECT metadata -> 'color'                AS color_json,    -- "silver" (JSON string)
       metadata ->> 'color'               AS color_text,    -- silver (text)
       metadata -> 'specs' -> 'ram'       AS ram_json,      -- 16 (JSON integer)
       metadata -> 'specs' ->> 'ram'      AS ram_text,      -- "16" (text)
       (metadata -> 'specs' ->> 'ram')::INT AS ram_int      -- 16 (integer)
FROM products_v2;

-- Path operator (deeper nesting)
SELECT metadata #> '{specs,ram}'          AS ram_json       -- same as ->->'ram'
       metadata #>> '{specs,storage}'     AS storage_text   -- '512'
FROM products_v2;

-- Filter on JSONB field
SELECT * FROM products_v2 WHERE metadata ->> 'color' = 'silver';
SELECT * FROM products_v2 WHERE (metadata -> 'specs' ->> 'ram')::INT >= 16;

-- Containment @> — does the left JSONB contain the right?
SELECT * FROM products_v2 WHERE metadata @> '{"color":"black"}';
SELECT * FROM products_v2 WHERE metadata @> '{"specs":{"ram":16}}';
SELECT * FROM products_v2 WHERE metadata @> '{"tags":["popular"]}';

-- Key existence ? — does the key exist?
SELECT * FROM products_v2 WHERE metadata ? 'wireless';     -- has 'wireless' key
SELECT * FROM products_v2 WHERE metadata ?| ARRAY['wireless','specs'];  -- has ANY
SELECT * FROM products_v2 WHERE metadata ?& ARRAY['color','tags'];      -- has ALL

-- Modify JSONB
UPDATE products_v2
SET metadata = metadata || '{"on_sale": true}'  -- merge (overwrite same keys)
WHERE id = 1;

UPDATE products_v2
SET metadata = metadata - 'on_sale'             -- remove a key
WHERE id = 1;

-- Set a specific path
UPDATE products_v2
SET metadata = JSONB_SET(metadata, '{specs,ram}', '32')   -- update nested
WHERE id = 1;

UPDATE products_v2
SET metadata = JSONB_SET(metadata, '{new_key}', '"new_value"', true)  -- add key
WHERE id = 1;

-- JSONB functions
SELECT JSONB_KEYS(metadata)                FROM products_v2 WHERE id=1;  -- get all keys
SELECT JSONB_EACH(metadata)               FROM products_v2 WHERE id=1;  -- key-value pairs as rows
SELECT JSONB_EACH_TEXT(metadata)          FROM products_v2 WHERE id=1;  -- text values
SELECT JSONB_OBJECT_KEYS(metadata)        FROM products_v2 WHERE id=1;  -- set of keys
SELECT JSONB_BUILD_OBJECT('a', 1, 'b', 2);  -- {"a":1,"b":2}
SELECT JSONB_BUILD_ARRAY(1, 2, 'three');    -- [1,2,"three"]
SELECT JSONB_ARRAY_ELEMENTS(metadata->'tags') FROM products_v2;  -- expand array to rows
SELECT JSONB_ARRAY_LENGTH(metadata->'tags') FROM products_v2;    -- array size
SELECT JSONB_STRIP_NULLS('{"a":1,"b":null}');  -- {"a":1}
SELECT JSONB_PRETTY('{"a":1,"b":2}');          -- formatted output

-- Aggregate rows into JSONB
SELECT JSONB_AGG(ROW_TO_JSON(p)) FROM products_v2 p;  -- array of all rows as JSON
SELECT JSONB_OBJECT_AGG(name, price) FROM products_v2;  -- {name: price, ...}
```

### JSONB Indexing

```sql
-- GIN index for containment and key-existence operators (@>, ?, ?|, ?&)
CREATE INDEX idx_products_metadata_gin ON products_v2 USING GIN (metadata);
-- Supports: @>, ?, ?|, ?& operators

-- GIN with specific path (jsonb_path_ops) — smaller, only supports @>
CREATE INDEX idx_products_metadata_path ON products_v2 USING GIN (metadata jsonb_path_ops);

-- B-tree on specific path (for equality/range queries on one field)
CREATE INDEX idx_products_color ON products_v2 ((metadata->>'color'));
-- Supports: WHERE metadata->>'color' = 'silver' or ORDER BY metadata->>'color'

-- After indexing:
SELECT * FROM products_v2 WHERE metadata @> '{"color":"silver"}';
-- Uses the GIN index — much faster on large tables
```

---

## Arrays

PostgreSQL arrays are first-class types — any type can be an array.

```sql
CREATE TABLE posts (
    id       SERIAL PRIMARY KEY,
    title    TEXT NOT NULL,
    tags     TEXT[],
    scores   INTEGER[],
    matrix   INTEGER[][]  -- 2D array
);

-- Insert arrays
INSERT INTO posts (title, tags, scores) VALUES
    ('SQL Guide',   ARRAY['sql','tutorial','database'],   ARRAY[95, 87, 92]),
    ('Python Tips', ARRAY['python','tips'],               ARRAY[78, 90]),
    ('Web Dev',     ARRAY['html','css','javascript'],     ARRAY[88, 76, 95, 83]);

-- Array literal syntax
INSERT INTO posts (title, tags) VALUES ('Test', '{"a","b","c"}');

-- Access by index (1-based!)
SELECT title, tags[1] AS first_tag, tags[2] AS second_tag FROM posts;

-- Array slicing
SELECT title, tags[1:2] AS first_two FROM posts;

-- Array length
SELECT title, ARRAY_LENGTH(tags, 1) AS tag_count FROM posts;
-- Second arg: dimension (1 for 1D arrays)

-- Array contains element: @> or = ANY()
SELECT * FROM posts WHERE tags @> ARRAY['sql'];          -- contains 'sql'
SELECT * FROM posts WHERE 'sql' = ANY(tags);             -- same
SELECT * FROM posts WHERE tags @> ARRAY['sql','tutorial']; -- contains both

-- Array overlap &&
SELECT * FROM posts WHERE tags && ARRAY['sql','python']; -- has at least one of

-- Unnest — expand array to rows
SELECT title, UNNEST(tags) AS tag FROM posts ORDER BY title, tag;

-- Array aggregation
SELECT ARRAY_AGG(name ORDER BY name) AS all_names FROM customers;
SELECT ARRAY_AGG(DISTINCT country) AS countries FROM customers;

-- Array functions
SELECT ARRAY_APPEND(ARRAY[1,2,3], 4);           -- {1,2,3,4}
SELECT ARRAY_PREPEND(0, ARRAY[1,2,3]);           -- {0,1,2,3}
SELECT ARRAY_REMOVE(ARRAY[1,2,3,2], 2);          -- {1,3}
SELECT ARRAY_REPLACE(ARRAY[1,2,3], 2, 99);       -- {1,99,3}
SELECT ARRAY_CAT(ARRAY[1,2], ARRAY[3,4]);        -- {1,2,3,4}
SELECT ARRAY_UPPER(ARRAY[1,2,3], 1);             -- 3 (last index)
SELECT ARRAY_LOWER(ARRAY[1,2,3], 1);             -- 1 (first index)
SELECT ARRAY_NDIMS(ARRAY[[1,2],[3,4]]);           -- 2 (number of dimensions)
SELECT ARRAY_TO_STRING(ARRAY[1,2,3], ',');       -- '1,2,3'
SELECT STRING_TO_ARRAY('a,b,c', ',');            -- {a,b,c}
SELECT ARRAY_POSITION(ARRAY['a','b','c'], 'b');  -- 2 (index of 'b')
SELECT ARRAY_POSITIONS(ARRAY[1,2,1,3,1], 1);    -- {1,3,5}

-- GIN index on array columns
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);
-- Supports: @>, &&, = ANY() (with GIN)
```

---

## Full-Text Search

PostgreSQL has powerful built-in full-text search using `tsvector` and `tsquery`.

```sql
-- tsvector: preprocessed document (stemmed, stopwords removed)
-- tsquery: search query

-- Basic full-text search
SELECT title
FROM posts
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'sql & tutorial');

-- Create a tsvector column for efficiency
ALTER TABLE posts ADD COLUMN search_vector TSVECTOR;

UPDATE posts SET search_vector =
    TO_TSVECTOR('english', COALESCE(title,'') || ' ' || COALESCE(content,''));

-- Create a GIN index on the tsvector
CREATE INDEX idx_posts_fts ON posts USING GIN (search_vector);

-- Search
SELECT title
FROM posts
WHERE search_vector @@ TO_TSQUERY('english', 'sql & database');

-- tsquery operators:
-- &   AND: 'sql & database'
-- |   OR:  'sql | python'
-- !   NOT: '!python'
-- <-> FOLLOWED BY: 'sql <-> tutorial' (adjacent words)
-- <2> WITHIN 2: 'sql <2> database' (within 2 words of each other)

-- Phrase search
SELECT * FROM posts WHERE search_vector @@ PHRASETO_TSQUERY('english', 'sql tutorial');

-- Plainto_tsquery — parses plain text (no operators needed)
SELECT * FROM posts WHERE search_vector @@ PLAINTO_TSQUERY('english', 'sql tutorial guide');
-- Same as to_tsquery('english', 'sql & tutorial & guide')

-- Highlighting matches
SELECT
    title,
    TS_HEADLINE('english', title,
        TO_TSQUERY('english', 'sql'),
        'StartSel=<b>, StopSel=</b>'
    ) AS highlighted
FROM posts
WHERE search_vector @@ TO_TSQUERY('english', 'sql');

-- Ranking results
SELECT
    title,
    TS_RANK(search_vector, TO_TSQUERY('english', 'sql & database')) AS rank
FROM posts
WHERE search_vector @@ TO_TSQUERY('english', 'sql & database')
ORDER BY rank DESC;

-- TS_RANK_CD also considers proximity
SELECT title, TS_RANK_CD(search_vector, query) AS rank
FROM posts, TO_TSQUERY('english', 'sql & tutorial') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### Auto-Update tsvector with Trigger

```sql
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    NEW.search_vector :=
        TO_TSVECTOR('english', COALESCE(NEW.title,'') || ' ' || COALESCE(NEW.content,''));
    RETURN NEW;
END;
$$;

CREATE TRIGGER posts_search_vector
BEFORE INSERT OR UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION update_search_vector();
```

### pg_trgm — Fuzzy Search (Trigram)

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Find similar strings (handles typos)
SELECT * FROM customers WHERE name % 'Aliice Johnson';  -- typo 'Aliice'
SELECT similarity('Alice', 'Aliice');  -- 0.5 (0=no match, 1=identical)

-- GIN index for trigram search
CREATE INDEX idx_customers_name_trgm ON customers USING GIN (name gin_trgm_ops);

-- Fast LIKE and ILIKE with trigram index
SELECT * FROM products WHERE name ILIKE '%laptop%';
-- With GIN trgm index, this is much faster than without!
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);
```

---

## Table Partitioning

Partitioning splits a large table into smaller physical tables (partitions)
while presenting them as one logical table.

```sql
-- Range partitioning (most common for time-series data)
CREATE TABLE events (
    id          BIGINT NOT NULL,
    event_type  TEXT NOT NULL,
    user_id     INT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    data        JSONB
) PARTITION BY RANGE (created_at);

-- Create partitions (each month)
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
CREATE TABLE events_2024_03 PARTITION OF events
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Default partition (catches rows that don't match any range)
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Insert works on the parent table, data goes to correct partition
INSERT INTO events (id, event_type, user_id) VALUES (1, 'click', 42);

-- Query the parent — PostgreSQL does partition pruning automatically
SELECT COUNT(*) FROM events WHERE created_at >= '2024-01-01';
-- Only scans events_2024_01, not all partitions

-- List partitions
SELECT tablename FROM pg_tables WHERE tablename LIKE 'events_%';

-- Detach and drop an old partition (archiving pattern)
ALTER TABLE events DETACH PARTITION events_2024_01;
DROP TABLE events_2024_01;

-- List partitioning (by category)
CREATE TABLE products_partitioned (
    id        SERIAL,
    name      TEXT,
    category  TEXT NOT NULL,
    price     NUMERIC(10,2)
) PARTITION BY LIST (category);

CREATE TABLE products_electronics PARTITION OF products_partitioned
    FOR VALUES IN ('Electronics');
CREATE TABLE products_furniture PARTITION OF products_partitioned
    FOR VALUES IN ('Furniture');
CREATE TABLE products_other PARTITION OF products_partitioned DEFAULT;

-- Hash partitioning (distribute rows evenly)
CREATE TABLE large_table (
    id   BIGINT NOT NULL,
    data TEXT
) PARTITION BY HASH (id);

CREATE TABLE large_table_0 PARTITION OF large_table FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE large_table_1 PARTITION OF large_table FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE large_table_2 PARTITION OF large_table FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE large_table_3 PARTITION OF large_table FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

## COPY — Bulk Import/Export

```sql
-- Import CSV into a table
COPY customers (name, email, city, country)
FROM '/tmp/customers.csv'
CSV HEADER;  -- first row is header

-- Import with custom delimiter
COPY customers FROM '/tmp/data.tsv' DELIMITER E'\t' CSV;

-- Export to CSV
COPY customers TO '/tmp/customers_export.csv' CSV HEADER;

-- Export query result
COPY (SELECT id, name, email FROM customers WHERE country = 'US')
TO '/tmp/us_customers.csv' CSV HEADER;

-- From psql client (reads from client machine, not server)
\copy customers FROM '/local/path/file.csv' CSV HEADER
\copy (SELECT * FROM customers) TO '/local/path/export.csv' CSV HEADER

-- Bulk insert with COPY for performance
-- COPY is 10-100x faster than INSERT for large datasets
-- Use for initial data loads or ETL processes
```

---

## Useful Extensions

```sql
-- List installed extensions
SELECT * FROM pg_extension;
SELECT * FROM pg_available_extensions ORDER BY name;

-- Enable extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";      -- UUID generation
CREATE EXTENSION IF NOT EXISTS pgcrypto;         -- Cryptographic functions
CREATE EXTENSION IF NOT EXISTS pg_trgm;          -- Trigram fuzzy search
CREATE EXTENSION IF NOT EXISTS hstore;           -- Key-value in a column
CREATE EXTENSION IF NOT EXISTS tablefunc;        -- CROSSTAB (pivot)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements; -- Query statistics
CREATE EXTENSION IF NOT EXISTS btree_gist;       -- GiST index for B-tree types
CREATE EXTENSION IF NOT EXISTS unaccent;         -- Remove accents for search

-- uuid-ossp
SELECT uuid_generate_v4();   -- random UUID
SELECT uuid_generate_v1();   -- time-based UUID

-- pgcrypto
SELECT gen_random_uuid();                        -- random UUID (no extension in PG13+)
SELECT crypt('mypassword', gen_salt('bf'));       -- bcrypt hash
SELECT crypt('mypassword', stored_hash) = stored_hash AS valid FROM users;
SELECT encode(digest('hello', 'sha256'), 'hex'); -- SHA-256 hash

-- hstore (key-value)
SELECT 'color=>red, size=>large'::hstore;
SELECT ('color=>red'::hstore)->'color';  -- 'red'

-- pg_cron (if installed)
SELECT cron.schedule('nightly-stats', '0 2 * * *', 'ANALYZE;');
SELECT cron.schedule('hourly-refresh', '0 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales');
SELECT * FROM cron.job;  -- list scheduled jobs
SELECT cron.unschedule('nightly-stats');
```

---

## Row Security Policies (RLS)

```sql
-- Enable row-level security on a table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create a policy: users can only see their own orders
CREATE POLICY orders_isolation_policy ON orders
    FOR ALL
    TO app_user
    USING (customer_id = current_setting('app.current_customer_id')::INT);

-- Set the context (in application code)
SET app.current_customer_id = '42';
SELECT * FROM orders;
-- Only returns orders where customer_id = 42

-- Bypass RLS for superuser/owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;  -- even owner is subject to policy
-- Or:
SET row_security = OFF;  -- superuser bypasses all policies

-- Different policies for different operations
CREATE POLICY read_policy ON orders FOR SELECT TO app_user
    USING (customer_id = current_setting('app.current_customer_id')::INT);
    
CREATE POLICY write_policy ON orders FOR INSERT TO app_user
    WITH CHECK (customer_id = current_setting('app.current_customer_id')::INT);
```

---

## LISTEN / NOTIFY — Real-Time Pub/Sub

```sql
-- In one psql session (the listener):
LISTEN order_updates;
-- Waits for notifications...

-- In another psql session (the notifier):
NOTIFY order_updates, 'Order 42 status changed to shipped';
-- The first session receives the message

-- Use in trigger to notify on changes:
CREATE OR REPLACE FUNCTION notify_order_change()
RETURNS TRIGGER LANGUAGE plpgsql AS $$
BEGIN
    PERFORM PG_NOTIFY('order_updates',
        JSON_BUILD_OBJECT(
            'order_id', NEW.id,
            'status', NEW.status,
            'operation', TG_OP
        )::TEXT
    );
    RETURN NEW;
END;
$$;

CREATE TRIGGER order_change_notify
AFTER INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION notify_order_change();
-- Application can LISTEN for these notifications for real-time updates
```

---

## Useful System Queries

```sql
-- Database size
SELECT pg_size_pretty(pg_database_size(current_database()));

-- Table sizes (sorted by largest)
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_only,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS indexes
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Long-running queries
SELECT
    pid, now() - pg_stat_activity.query_start AS duration,
    query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > INTERVAL '5 minutes'
  AND state != 'idle';

-- Active connections
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
SELECT count(*), state FROM pg_stat_activity GROUP BY state;

-- Cache hit rate (should be > 99% on production)
SELECT
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;

-- Vacuum / analyze status
SELECT relname, last_vacuum, last_autovacuum, last_analyze, n_dead_tup, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## Exercises

**Exercise 1:** Add a `metadata` JSONB column to the products table.
Insert metadata with color, weight, and tags. Query products by color.
Create a GIN index and verify it's used.

**Exercise 2:** Create a tags array column on a new `articles` table.
Insert 5 articles with tags. Find articles tagged with both 'sql' and 'tutorial'.

**Exercise 3:** Set up full-text search on the products table (name + description).
Create a tsvector column, populate it, create a GIN index, and run a search.

**Exercise 4:** Create a partitioned `logs` table partitioned by month.
Create two monthly partitions. Insert data and verify partition pruning with EXPLAIN.

---

## Key Takeaways

1. JSONB is binary JSON — always prefer it over JSON; supports operators and GIN indexing
2. `@>` (containment) is the most powerful JSONB/array operator — always backed by GIN index
3. Arrays are first-class — UNNEST expands them into rows
4. Full-text search: `to_tsvector` preprocesses text; `tsquery` is the search query; `@@` matches
5. pg_trgm enables fuzzy search and fast `LIKE`/`ILIKE` with GIN indexes
6. Partitioning splits large tables for faster queries and easier data management
7. COPY is 10-100x faster than INSERT for bulk data loading
8. Row Security Policies enforce access control at the database level

---

## Congratulations!

You've completed the SQL Mastery course. You now have the knowledge to:

- Design normalized database schemas
- Write efficient queries at any complexity level
- Use window functions for analytics
- Optimize slow queries with indexes and EXPLAIN ANALYZE
- Use PostgreSQL-specific power features

**What's Next:**
- Complete all 5 exercises in the [exercises/](../exercises/) folder
- [Lesson 17 — Roles, Users & Access Management](17-roles-users-and-access-management.md) — connect as an app, not as a superuser
- [Lesson 18 — Connecting to PostgreSQL from Python](18-connecting-from-python.md) — wire this all up to real application code
- Learn about replication and high availability
- Explore Timescale for time-series data
- Try PostGIS for geospatial queries
- Learn about logical replication and streaming
