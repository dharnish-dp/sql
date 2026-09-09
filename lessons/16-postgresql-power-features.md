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

**Why this whole lesson exists:** [Lesson 20](20-standard-sql-vs-postgresql.md)
draws a line between "standard SQL" (works on any engine) and
"PostgreSQL-specific extensions" (Postgres only). Everything in *this*
lesson sits firmly on the PostgreSQL-only side of that line — none of
it is ANSI SQL, none of it is guaranteed to exist if you ever move to
MySQL or SQL Server. That's not a downside — these are genuinely
powerful capabilities most other engines simply don't have — just know
going in that this entire lesson is "bonus power," not universal SQL
knowledge.

---

## JSONB — Semi-Structured Data

**Why you need this:** every table so far had a fixed, known set of
columns ([Lesson 03](03-data-types-and-schema.md)). But sometimes data
genuinely varies row to row — a `metadata` field where one product has
`{"color": "red"}` and another has `{"size": "XL", "material": "cotton"}`
— with no sensible fixed set of columns to normalize it into ahead of
time. JSONB lets you store that flexible, semi-structured data directly
inside a single column, while still letting you query and index it.

JSONB stores JSON in a decomposed binary format:
- Validated on insert
- Supports operators and indexing
- Keys are sorted, duplicates removed
- Always prefer JSONB over JSON

**Why "always prefer JSONB over JSON":** the plain `JSON` type in
Postgres stores your text *exactly as typed* — same key order, same
whitespace, even duplicate keys preserved — and every single time you
query it, Postgres has to re-parse that text from scratch. `JSONB`
instead parses and converts it **once**, at insert time, into an
efficient binary layout — which is also the only format that indexing
(below) can work with at all. The tradeoff is `JSONB` inserts are
slightly slower (parsing happens up front); reads are much faster
(no re-parsing). For virtually every real use case, you want the read
speed and indexing — hence "always."

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
```

Sample output of `SELECT * FROM products_v2;`:

| id | name | price | metadata |
|---|---|---|---|
| 1 | Laptop Pro | 1299.99 | `{"color":"silver","weight":1.4,"tags":["sale","popular"],"specs":{"ram":16,"storage":512}}` |
| 2 | Mouse | 29.99 | `{"color":"black","weight":0.1,"tags":["accessory"],"wireless":true}` |
| 3 | Monitor | 399.99 | `{"color":"black","size":27,"resolution":"4K","tags":["display","popular"]}` |

### The Access Operators — `->` vs `->>` vs `#>` vs `#>>`

These four symbols are the part that trips people up most, because
nothing about `->` visually suggests "get me a JSON field." Think of it
this way instead: **the arrow points at the value you want; one `>`
keeps it as JSON, two `>`s "flatten" it out to plain text.**

| Operator | Goes how deep | Returns |
|---|---|---|
| `->` | One level | JSON (keeps type — could be a JSON string, number, object, array) |
| `->>` | One level | TEXT (always flattened to plain text) |
| `#>` | Any depth (via a path array) | JSON |
| `#>>` | Any depth (via a path array) | TEXT |

```sql
SELECT metadata -> 'color'                AS color_json,    -- "silver" (JSON string)
       metadata ->> 'color'               AS color_text,    -- silver (text)
       metadata -> 'specs' -> 'ram'       AS ram_json,      -- 16 (JSON integer)
       metadata -> 'specs' ->> 'ram'      AS ram_text,      -- "16" (text)
       (metadata -> 'specs' ->> 'ram')::INT AS ram_int      -- 16 (integer)
FROM products_v2
WHERE id = 1;
```

**Walk through why each line differs, even though they're all "getting
the ram value":**
- `metadata -> 'color'` returns `"silver"` — note the quotes; this is
  still a *JSON value*, specifically a JSON string, not a plain Postgres
  `TEXT` value. You can't directly compare this to `'silver'` with `=`
  and expect it to just work everywhere — quoting differences can bite.
- `metadata ->> 'color'` returns `silver` — no quotes, a real Postgres
  `TEXT` value, directly comparable to a normal string.
- `metadata -> 'specs' -> 'ram'` chains two single-level `->`s: first
  step reaches the `specs` object, second step reaches `ram` inside it
  — still JSON the whole way, so it prints as `16`.
- `metadata -> 'specs' ->> 'ram'` — same two-step path, but the *last*
  hop uses `->>` to flatten to text — so despite `ram` being a JSON
  number, you get back the text `"16"` (as a string, shown without
  quotes in output but is `TEXT` type, not `INT`).
- `(metadata -> 'specs' ->> 'ram')::INT` — explicitly casts that text
  back into a real integer, so you could use it in math or a numeric
  comparison.

Sample output for id=1:

| color_json | color_text | ram_json | ram_text | ram_int |
|---|---|---|---|---|
| "silver" | silver | 16 | 16 | 16 |

**Why bother with the path operators (`#>`, `#>>`) instead of chaining
`->`?** Chaining works, but gets unwieldy for deep nesting, and can't be
built dynamically from a variable path length. The path operators take
an array of keys instead:

```sql
-- Path operator (deeper nesting) — array of keys instead of chained ->
SELECT metadata #> '{specs,ram}'          AS ram_json,      -- same as -> 'specs' -> 'ram'
       metadata #>> '{specs,storage}'     AS storage_text   -- '512'
FROM products_v2
WHERE id = 1;
```

`'{specs,ram}'` is Postgres array syntax for `['specs', 'ram']` — "go
into `specs`, then into `ram`" — identical result to the chained
version above, just expressed as one path instead of repeated arrows.

Sample output:

| ram_json | storage_text |
|---|---|
| 16 | 512 |

### Filtering by a JSONB Field

```sql
-- Filter on JSONB field
SELECT * FROM products_v2 WHERE metadata ->> 'color' = 'silver';
SELECT * FROM products_v2 WHERE (metadata -> 'specs' ->> 'ram')::INT >= 16;
```
Both are ordinary `WHERE` clauses — the only new idea is that the
*value* being compared comes from a JSONB extraction instead of a plain
column. The second one needs the `::INT` cast because `->>` always
hands back text, and you can't do a numeric `>=` comparison on text
reliably.

Sample output (first query):

| id | name | price | metadata |
|---|---|---|---|
| 1 | Laptop Pro | 1299.99 | (full metadata, color=silver) |

### Containment `@>` — "Does This JSONB Contain That Shape?"

**Why this is the single most useful JSONB operator:** it lets you ask
"does this JSON document contain this smaller piece" without knowing or
caring about every other field in the document — and, importantly, it's
the one JSONB operator with full GIN index support (see indexing below).

```sql
-- Containment @> — does the left JSONB contain the right?
SELECT * FROM products_v2 WHERE metadata @> '{"color":"black"}';
SELECT * FROM products_v2 WHERE metadata @> '{"specs":{"ram":16}}';
SELECT * FROM products_v2 WHERE metadata @> '{"tags":["popular"]}';
```

Read `@>` as "contains." `metadata @> '{"color":"black"}'` means "does
this row's `metadata` document contain, somewhere within it, the key
`color` set to `black`" — it doesn't matter what else is in the
document, or in what order.

Sample output for `metadata @> '{"color":"black"}'` (matches Mouse and
Monitor, both `color: black`):

| id | name | metadata |
|---|---|---|
| 2 | Mouse | `{"color":"black",...}` |
| 3 | Monitor | `{"color":"black",...}` |

### Key Existence — `?`, `?|`, `?&`

```sql
-- Key existence ? — does the key exist?
SELECT * FROM products_v2 WHERE metadata ? 'wireless';     -- has 'wireless' key
SELECT * FROM products_v2 WHERE metadata ?| ARRAY['wireless','specs'];  -- has ANY
SELECT * FROM products_v2 WHERE metadata ?& ARRAY['color','tags'];      -- has ALL
```

Different question from `@>` — you're not checking a key's *value*
here, only whether the *key itself* exists at all, regardless of what
it's set to:
- `?` — does this one specific key exist? (`wireless` — only Mouse has it)
- `?|` — does **any** key from this list exist? (like a JSONB version of `OR`)
- `?&` — do **all** keys from this list exist? (like a JSONB version of `AND`)

Sample output for `metadata ? 'wireless'` (only Mouse has that key):

| id | name |
|---|---|
| 2 | Mouse |

### Modifying JSONB — `||` (merge), `-` (remove key), `JSONB_SET` (nested update)

```sql
-- Modify JSONB
UPDATE products_v2
SET metadata = metadata || '{"on_sale": true}'  -- merge (overwrite same keys)
WHERE id = 1;

UPDATE products_v2
SET metadata = metadata - 'on_sale'             -- remove a key
WHERE id = 1;
```

`||` here is JSONB's merge operator (not string concatenation, even
though it's the same symbol you saw for text in
[Lesson 02](02-sql-fundamentals.md) — same symbol, different meaning
depending on the types either side). `metadata || '{"on_sale": true}'`
takes the existing document and layers the new key on top — if
`on_sale` already existed, the new value wins; every other key is
untouched.

`metadata - 'on_sale'` removes just that one top-level key, leaving
everything else in the document as-is.

```sql
-- Set a specific path
UPDATE products_v2
SET metadata = JSONB_SET(metadata, '{specs,ram}', '32')   -- update nested
WHERE id = 1;

UPDATE products_v2
SET metadata = JSONB_SET(metadata, '{new_key}', '"new_value"', true)  -- add key
WHERE id = 1;
```

`||` and `-` only operate on **top-level** keys — they can't reach
*into* a nested object like `specs.ram`. `JSONB_SET(document, path,
new_value)` is the tool for that: give it the same `'{specs,ram}'` path
syntax from `#>` above, plus the new value, and it rewrites just that
nested field, leaving the rest of the structure untouched. The 4th
argument (`true` in the second example) tells it "create this key if it
doesn't already exist" — without it, `JSONB_SET` silently does nothing
if the path doesn't exist yet.

### JSONB Utility Functions

```sql
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
```

**The one worth calling out specially — `JSONB_ARRAY_ELEMENTS`:** this
is JSONB's version of `UNNEST` (which you'll see below for arrays) — it
takes a JSON array field and expands it into **one row per element**,
turning nested JSON data into ordinary rows you can `GROUP BY`, `JOIN`,
or filter with normal SQL.

Sample output of `SELECT title, JSONB_ARRAY_ELEMENTS(metadata->'tags') AS tag FROM products_v2 WHERE id = 1;`
(Laptop Pro's tags are `["sale","popular"]`):

| name | tag |
|---|---|
| Laptop Pro | "sale" |
| Laptop Pro | "popular" |

```sql
-- Aggregate rows into JSONB
SELECT JSONB_AGG(ROW_TO_JSON(p)) FROM products_v2 p;  -- array of all rows as JSON
SELECT JSONB_OBJECT_AGG(name, price) FROM products_v2;  -- {name: price, ...}
```
The reverse direction — `JSONB_AGG` collapses many *rows* back down into
one JSON array, the same "many rows → one aggregated value" idea as
`ARRAY_AGG`/`STRING_AGG` from [Lesson 05](05-aggregations.md), just
producing JSON instead of an array or string.

Sample output of `JSONB_OBJECT_AGG(name, price)`:
```json
{"Laptop Pro": 1299.99, "Mouse": 29.99, "Monitor": 399.99}
```

### JSONB Indexing

**Why you need this:** everything above works without any index — but
on a large table, Postgres would have to check every row's `metadata`
one at a time ([Lesson 09](09-indexes-and-performance.md)'s sequential
scan). A **GIN index** (mentioned in Lesson 09's index-types table, now
put to actual use) is what makes JSONB containment/existence queries
fast at scale.

```sql
-- GIN index for containment and key-existence operators (@>, ?, ?|, ?&)
CREATE INDEX idx_products_metadata_gin ON products_v2 USING GIN (metadata);
-- Supports: @>, ?, ?|, ?& operators

-- GIN with specific path (jsonb_path_ops) — smaller, only supports @>
CREATE INDEX idx_products_metadata_path ON products_v2 USING GIN (metadata jsonb_path_ops);

-- B-tree on specific path (for equality/range queries on one field)
CREATE INDEX idx_products_color ON products_v2 ((metadata->>'color'));
-- Supports: WHERE metadata->>'color' = 'silver' or ORDER BY metadata->>'color'
```

**Which one to actually pick:**
- Plain `GIN (metadata)` — the general-purpose choice, supports all four
  operators, larger index size.
- `GIN (metadata jsonb_path_ops)` — smaller and faster, but *only*
  speeds up `@>` — pick this if `@>` is genuinely the only operator you
  query with.
- B-tree on `(metadata->>'color')` — this is an **expression index**,
  the exact pattern from [Lesson 09](09-indexes-and-performance.md)'s
  `LOWER(email)` example, just applied to a JSONB extraction instead —
  use this when you're always filtering/sorting on one specific,
  known field, not doing general containment checks.

```sql
-- After indexing:
SELECT * FROM products_v2 WHERE metadata @> '{"color":"silver"}';
-- Uses the GIN index — much faster on large tables
```

---

## Arrays

**Why you need this:** sometimes a value is naturally a small,
unordered-or-ordered *list* — tags on a post, scores on a test — and
creating a whole separate child table (per [Lesson 11](11-database-design-normalization.md)'s
normalization rules) feels like overkill for something you'll never
query relationally. PostgreSQL arrays let a single column legitimately
hold a list, as a first-class type — any type can be an array.

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
```

Sample output of `SELECT title, tags, scores FROM posts;`:

| title | tags | scores |
|---|---|---|
| SQL Guide | {sql,tutorial,database} | {95,87,92} |
| Python Tips | {python,tips} | {78,90} |
| Web Dev | {html,css,javascript} | {88,76,95,83} |
| Test | {a,b,c} | NULL |

**Two equivalent ways to write an array literal**, both shown above:
`ARRAY['sql','tutorial','database']` (the explicit function-like form)
and `'{"a","b","c"}'` (a plain string that Postgres parses as an array)
— same result either way; `ARRAY[...]` reads more clearly in application
code, the `'{...}'` string form is what you'll see when Postgres
*displays* an array back to you.

### Accessing Array Elements — 1-Based, Not 0-Based

```sql
-- Access by index (1-based!)
SELECT title, tags[1] AS first_tag, tags[2] AS second_tag FROM posts;
```

**This is the single most common bug for anyone coming from
Python/JavaScript/most programming languages:** Postgres arrays are
**1-indexed** — `tags[1]` is the *first* element, not `tags[0]`.
`tags[0]` doesn't error, it just silently returns `NULL` (there's no
0th element), which is a much sneakier bug than a loud error would be.

Sample output:

| title | first_tag | second_tag |
|---|---|---|
| SQL Guide | sql | tutorial |
| Python Tips | python | tips |

```sql
-- Array slicing
SELECT title, tags[1:2] AS first_two FROM posts;
```
Same 1-based counting extended to a range — `tags[1:2]` means "elements
1 through 2 inclusive," i.e. the first two tags.

Sample output:

| title | first_two |
|---|---|
| SQL Guide | {sql,tutorial} |
| Python Tips | {python,tips} |

```sql
-- Array length
SELECT title, ARRAY_LENGTH(tags, 1) AS tag_count FROM posts;
-- Second arg: dimension (1 for 1D arrays)
```
The `1` here isn't a typo or an offset — it's the **dimension number**
(recall `matrix INTEGER[][]` above is a 2D array; `ARRAY_LENGTH(matrix, 1)`
would ask "how many rows," `ARRAY_LENGTH(matrix, 2)` "how many columns
in each row"). For an ordinary 1D array like `tags`, there's only
dimension `1` to ask about.

Sample output:

| title | tag_count |
|---|---|
| SQL Guide | 3 |
| Python Tips | 2 |
| Web Dev | 3 |

### Searching Inside Arrays — `@>`, `= ANY()`, `&&`

**Why the containment operator looks familiar:** `@>` means the same
"contains" idea here as it did for JSONB above — same symbol, same
concept, different data type underneath.

```sql
-- Array contains element: @> or = ANY()
SELECT * FROM posts WHERE tags @> ARRAY['sql'];          -- contains 'sql'
SELECT * FROM posts WHERE 'sql' = ANY(tags);             -- same
SELECT * FROM posts WHERE tags @> ARRAY['sql','tutorial']; -- contains both
```

`tags @> ARRAY['sql']` and `'sql' = ANY(tags)` answer the exact same
question ("is `'sql'` one of the tags") from two different directions —
`@>` reads as "does the array contain this," `= ANY()` reads as "does
this value equal any element in the array." Both exist because
different query shapes read more naturally with one or the other;
`@>` is the one that gets GIN index support (see below).

Sample output for `tags @> ARRAY['sql']` (only SQL Guide has it):

| title | tags |
|---|---|
| SQL Guide | {sql,tutorial,database} |

```sql
-- Array overlap &&
SELECT * FROM posts WHERE tags && ARRAY['sql','python']; -- has at least one of
```
`&&` is a *weaker* check than `@>` — "do these two arrays share **at
least one** element," not "does the row's array contain **all** of
these." `tags && ARRAY['sql','python']` matches any post tagged `sql`
*or* `python` (or both) — SQL Guide and Python Tips both qualify.

Sample output:

| title | tags |
|---|---|
| SQL Guide | {sql,tutorial,database} |
| Python Tips | {python,tips} |

### UNNEST — Expanding an Array Into Rows

```sql
-- Unnest — expand array to rows
SELECT title, UNNEST(tags) AS tag FROM posts ORDER BY title, tag;
```

**Why this matters:** every operator above (`@>`, `&&`, indexing into
`tags[1]`) treats the array as one unit. But sometimes you need to
treat each element as its *own row* — e.g., to `COUNT` how many posts
use each tag with a normal `GROUP BY`. `UNNEST` is the conversion: one
row per array element, with the rest of that row's columns repeated
alongside each one.

Sample output (partial, alphabetically ordered):

| title | tag |
|---|---|
| SQL Guide | database |
| SQL Guide | sql |
| SQL Guide | tutorial |
| Python Tips | python |
| Python Tips | tips |

Notice `SQL Guide` now appears **three times** — once per tag — exactly
the "row multiplication" effect from [Lesson 06](06-joins-complete.md),
just produced by `UNNEST` instead of a `JOIN`.

### Array Aggregation and Utility Functions

```sql
-- Array aggregation
SELECT ARRAY_AGG(name ORDER BY name) AS all_names FROM customers;
SELECT ARRAY_AGG(DISTINCT country) AS countries FROM customers;
```
The reverse of `UNNEST` — collapsing many rows back into one array, the
same function from [Lesson 05](05-aggregations.md), shown here in its
natural home (this *is* the array lesson).

Sample output of `ARRAY_AGG(DISTINCT country)`:

| countries |
|---|
| {UK,US,Germany} |

```sql
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
```
These are largely self-explanatory from their names and the inline
comments — worth calling out only: `ARRAY_TO_STRING`/`STRING_TO_ARRAY`
are the two-way bridge between an array and a delimited text value
(e.g. exporting tags as a comma-separated string for a CSV, or the
reverse when importing one), and `ARRAY_UPPER`/`ARRAY_LOWER` give the
*index bounds* of the array (remembering arrays are 1-based, `ARRAY_LOWER`
is almost always `1` for a normal array you built yourself).

### GIN Index on Arrays

```sql
-- GIN index on array columns
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);
-- Supports: @>, &&, = ANY() (with GIN)
```
Same reasoning as JSONB's GIN index above — without it, `tags @>
ARRAY['sql']` has to check every row's array one at a time on a large
table; with it, Postgres can look the value up directly.

---

## Full-Text Search

**Why you need this:** `WHERE title ILIKE '%sql%'`
([Lesson 04](04-filtering-deep-dive.md)) finds exact substring matches,
but it can't handle "find posts about running, even if the post says
'runner' or 'ran'," rank results by relevance, or search efficiently at
scale (a leading `%` in `LIKE` can't use a normal B-tree index at all).
Full-text search is Postgres's built-in solution to all three problems.

PostgreSQL has powerful built-in full-text search using `tsvector` and `tsquery`.

**The core idea, in plain English before any syntax:** a `tsvector` is
your *document*, preprocessed — words reduced to their root form
("running" → "run"), common filler words ("the", "a", "is") stripped
out. A `tsquery` is your *search terms*, preprocessed the same way. The
`@@` operator asks "does this preprocessed document match this
preprocessed query" — matching on meaning-bearing word roots, not exact
substrings.

```sql
-- tsvector: preprocessed document (stemmed, stopwords removed)
-- tsquery: search query

-- Basic full-text search
SELECT title
FROM posts
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'sql & tutorial');
```
`to_tsvector('english', title)` converts the `title` column into its
searchable form on the fly, every time this query runs. `to_tsquery('english',
'sql & tutorial')` builds the search — the `&` here means "AND" (full
list of operators below). `@@` checks whether the query matches the
document.

Sample output (matches "SQL Guide" — contains both root words):

| title |
|---|
| SQL Guide |

**Why re-computing `to_tsvector(title)` on every single query is
wasteful:** it repeats the same preprocessing work every time, on every
row, whether or not you're searching. The fix is the same pattern from
[Lesson 09](09-indexes-and-performance.md)'s expression-index discussion,
but taken one step further — store the preprocessed result permanently:

```sql
-- Create a tsvector column for efficiency
ALTER TABLE posts ADD COLUMN search_vector TSVECTOR;

UPDATE posts SET search_vector =
    TO_TSVECTOR('english', COALESCE(title,'') || ' ' || COALESCE(content,''));
```
`COALESCE(title,'') || ' ' || COALESCE(content,'')` combines two columns
into one searchable blob (title *and* content both become searchable
together), with `COALESCE` guarding against either being `NULL` — recall
from [Lesson 05](05-aggregations.md) that concatenating with a `NULL`
would otherwise make the whole result `NULL`.

```sql
-- Create a GIN index on the tsvector
CREATE INDEX idx_posts_fts ON posts USING GIN (search_vector);

-- Search
SELECT title
FROM posts
WHERE search_vector @@ TO_TSQUERY('english', 'sql & database');
```
Now the expensive preprocessing happened once, at write time, and every
search just reads the pre-built `search_vector` column — the same
"do the work once at insert, benefit at every read" principle behind
every expression index in this course.

### `tsquery` Operators

```sql
-- tsquery operators:
-- &   AND: 'sql & database'
-- |   OR:  'sql | python'
-- !   NOT: '!python'
-- <-> FOLLOWED BY: 'sql <-> tutorial' (adjacent words)
-- <2> WITHIN 2: 'sql <2> database' (within 2 words of each other)
```
Read these as a tiny boolean/proximity language for text: `&`/`|`/`!`
work exactly like `AND`/`OR`/`NOT` do in a `WHERE` clause, just spelled
differently because they operate on a query string, not full SQL syntax.
`<->` and `<N>` have no `WHERE`-clause equivalent — they're proximity
checks unique to text search: "these two words, right next to each
other" (`<->`) or "within N words of each other" (`<N>`).

```sql
-- Phrase search
SELECT * FROM posts WHERE search_vector @@ PHRASETO_TSQUERY('english', 'sql tutorial');
```
`PHRASETO_TSQUERY` automatically builds the `<->` (adjacent-word) form
for you from plain text — "sql tutorial" becomes "find 'sql' followed
immediately by 'tutorial'," without you writing `<->` by hand.

```sql
-- Plainto_tsquery — parses plain text (no operators needed)
SELECT * FROM posts WHERE search_vector @@ PLAINTO_TSQUERY('english', 'sql tutorial guide');
-- Same as to_tsquery('english', 'sql & tutorial & guide')
```
`PLAINTO_TSQUERY` is the "just search for all these words" helper —
takes a plain phrase, ANDs every word together automatically. This is
the function you'd actually wire up to a search box, since real users
type plain phrases, not `&`/`|` syntax.

### Highlighting and Ranking Results

```sql
-- Highlighting matches
SELECT
    title,
    TS_HEADLINE('english', title,
        TO_TSQUERY('english', 'sql'),
        'StartSel=<b>, StopSel=</b>'
    ) AS highlighted
FROM posts
WHERE search_vector @@ TO_TSQUERY('english', 'sql');
```
`TS_HEADLINE` returns the matched text with the search term wrapped in
markup you specify (`<b>...</b>` here) — exactly the "highlight the
matched word" behavior you see on real search results pages.

Sample output:

| title | highlighted |
|---|---|
| SQL Guide | <b>SQL</b> Guide |

```sql
-- Ranking results
SELECT
    title,
    TS_RANK(search_vector, TO_TSQUERY('english', 'sql & database')) AS rank
FROM posts
WHERE search_vector @@ TO_TSQUERY('english', 'sql & database')
ORDER BY rank DESC;
```
Not every match is equally relevant — a post mentioning "sql" once
should probably rank below one mentioning it five times. `TS_RANK`
computes a relevance score (roughly: how often and how prominently the
terms appear) so you can `ORDER BY rank DESC` and show the best matches
first, the same way a search engine would.

Sample output:

| title | rank |
|---|---|
| SQL Guide | 0.36 |

```sql
-- TS_RANK_CD also considers proximity
SELECT title, TS_RANK_CD(search_vector, query) AS rank
FROM posts, TO_TSQUERY('english', 'sql & tutorial') query
ORDER BY rank DESC;
```
`TS_RANK_CD` ("cover density") is a fancier ranking that *also* rewards
matched words appearing close together, not just present anywhere in
the document — generally a better relevance signal for multi-word
searches.

### Auto-Updating `search_vector` With a Trigger

**Why you need this:** the manual `UPDATE ... SET search_vector = ...`
shown earlier has to be re-run by hand every time a post's title or
content changes, or the search index silently goes stale. A **trigger**
(a PL/pgSQL function that runs automatically on an event —
[Lesson 14](14-stored-procedures-and-functions.md) covers these in
depth) fixes this permanently.

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
`NEW` here refers to the row *being inserted or updated* — the function
recomputes `search_vector` from whatever the new `title`/`content`
values are, and `BEFORE INSERT OR UPDATE ... FOR EACH ROW` wires it to
run automatically, every time, on every affected row — so
`search_vector` can never drift out of sync with the real content again.

### `pg_trgm` — Fuzzy Search (Typo-Tolerant)

**Why you need this:** full-text search above matches word *roots*, but
still requires a correctly-spelled word. `pg_trgm` handles the different
problem of typos and near-matches — "Aliice" should still find "Alice."

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Find similar strings (handles typos)
SELECT * FROM customers WHERE name % 'Aliice Johnson';  -- typo 'Aliice'
SELECT similarity('Alice', 'Aliice');  -- 0.5 (0=no match, 1=identical)
```
**How trigram matching actually works, briefly:** a "trigram" is every
3-character substring of a word (`"Alice"` → `Ali`, `lic`, `ice`, plus
padding). `%` checks whether two strings share enough trigrams in
common to be considered "similar" (above a configurable threshold) — a
typo changes only a few trigrams, so a misspelled word still shares most
of its trigrams with the correct one. `similarity()` returns that
overlap as a number from 0 (no shared trigrams) to 1 (identical).

Sample output of `similarity('Alice', 'Aliice')`:

| similarity |
|---|
| 0.5 |

```sql
-- GIN index for trigram search
CREATE INDEX idx_customers_name_trgm ON customers USING GIN (name gin_trgm_ops);

-- Fast LIKE and ILIKE with trigram index
SELECT * FROM products WHERE name ILIKE '%laptop%';
-- With GIN trgm index, this is much faster than without!
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);
```
**A genuinely useful side effect worth knowing:** a `gin_trgm_ops` index
doesn't just speed up `%` fuzzy matching — it *also* speeds up
`LIKE`/`ILIKE` patterns with a **leading** wildcard (`'%laptop%'`),
which a normal B-tree index cannot help with at all (B-tree indexes only
help patterns anchored at the start, like `'laptop%'` — see
[Lesson 09](09-indexes-and-performance.md)). This is the one real
exception to "leading wildcard LIKE can't use an index."

---

## Table Partitioning

**Why you need this:** [Lesson 09](09-indexes-and-performance.md)
taught you indexing to make lookups fast within one table. But once a
table reaches tens or hundreds of millions of rows, even indexed
operations (backups, bulk deletes of old data, vacuum) get slow just
from the table's sheer physical size. Partitioning is a *structural*
solution — splitting one giant table into smaller physical pieces,
while every query still sees it as one logical table.

Partitioning splits a large table into smaller physical tables (partitions)
while presenting them as one logical table.

**Build the intuition first:** imagine `events` has 500 million rows of
click/pageview data going back 3 years. If you partition it *by month*,
Postgres physically stores January's rows separately from February's,
separately from March's, and so on — but a plain `SELECT * FROM events`
still works exactly as if it were one table. The benefit shows up when
your `WHERE` clause mentions the partitioning column: Postgres can
figure out *which single monthly partition* holds the answer, and skip
reading the other 35 months entirely.

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
```
`PARTITION BY RANGE (created_at)` declares the *strategy* on the parent
table — "split by ranges of `created_at`" — but creates no actual
storage for rows yet. Each `CREATE TABLE ... PARTITION OF events FOR
VALUES FROM (...) TO (...)` then defines one real physical partition
and the exact range of `created_at` values it's responsible for (`FROM`
inclusive, `TO` exclusive — March 1st itself belongs to the March
partition, not February's). The `DEFAULT` partition is a safety net —
any row whose date doesn't fall in any explicitly-defined range (e.g.,
a bad future or past date) lands there instead of erroring out.

```sql
-- Insert works on the parent table, data goes to correct partition
INSERT INTO events (id, event_type, user_id) VALUES (1, 'click', 42);
```
You never insert into a partition directly in normal use — you insert
into the *parent* table name (`events`), and Postgres automatically
routes the row into whichever partition's date range matches.

```sql
-- Query the parent — PostgreSQL does partition pruning automatically
SELECT COUNT(*) FROM events WHERE created_at >= '2024-01-01';
-- Only scans events_2024_01, not all partitions
```
This is the actual payoff, called **partition pruning**: because the
`WHERE` clause filters on the exact column the table is partitioned by,
Postgres's planner can look at the query *before* running it and
realize "only `events_2024_01` (and later) could possibly match" —
skipping `events_default` and any other irrelevant partitions entirely,
without you doing anything special in the query itself.

```sql
-- List partitions
SELECT tablename FROM pg_tables WHERE tablename LIKE 'events_%';

-- Detach and drop an old partition (archiving pattern)
ALTER TABLE events DETACH PARTITION events_2024_01;
DROP TABLE events_2024_01;
```
**Why this "detach then drop" pattern is genuinely valuable in
practice:** deleting a year's worth of old data with `DELETE FROM events
WHERE created_at < '2023-01-01'` on a huge table means scanning and
deleting millions of individual rows — slow, and it bloats the table
with dead rows ([Lesson 09](09-indexes-and-performance.md)). Detaching
an entire partition and dropping it instead is close to instantaneous —
you're removing one whole physical table object, not deleting rows one
at a time.

```sql
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
```
**How list partitioning differs from range partitioning:** range
partitioning (`events` above) splits by a *continuous* value (dates,
numbers) using `FROM`/`TO` boundaries. List partitioning splits by
*discrete, named* categories — each partition claims an explicit list
of values (`FOR VALUES IN ('Electronics')`) rather than a range. Use
range for "time-series or sequential data," list for "a small fixed set
of categories."

```sql
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
**When you'd reach for hash instead of range/list:** when there's no
natural range or category to split by, but you still want rows spread
evenly across several partitions purely to shrink each individual
partition's size (for parallelism or storage management, not for
pruning specific queries). `MODULUS 4, REMAINDER 0/1/2/3` works like a
hash-based "deal into 4 piles" — each row's `id` gets hashed, and the
result's remainder when divided by 4 decides which of the 4 partitions
it lands in. Unlike range/list, you generally *can't* predict which
partition a given row goes to just by looking at its value — this
trades off query-time pruning for even storage distribution.

---

## COPY — Bulk Import/Export

**Why you need this:** you've been inserting rows with `INSERT` one
statement at a time. For loading (or exporting) thousands or millions of
rows at once — a CSV export from another system, a data migration —
running individual `INSERT` statements is dramatically slower than
necessary. `COPY` is Postgres's purpose-built bulk-transfer mechanism.

```sql
-- Import CSV into a table
COPY customers (name, email, city, country)
FROM '/tmp/customers.csv'
CSV HEADER;  -- first row is header
```
`(name, email, city, country)` tells Postgres which columns the CSV's
columns map to, in order. `CSV HEADER` tells it the file's first line is
a header row (column names), not actual data — skip it rather than
trying to insert it as a row.

```sql
-- Import with custom delimiter
COPY customers FROM '/tmp/data.tsv' DELIMITER E'\t' CSV;
```
Not every "CSV" is actually comma-separated — `DELIMITER E'\t'`
overrides the separator character to a tab, for a genuine TSV
(tab-separated) file.

```sql
-- Export to CSV
COPY customers TO '/tmp/customers_export.csv' CSV HEADER;

-- Export query result
COPY (SELECT id, name, email FROM customers WHERE country = 'US')
TO '/tmp/us_customers.csv' CSV HEADER;
```
`COPY` works in both directions — `FROM` loads a file into a table, `TO`
writes a table (or, in the second example, an arbitrary query's result)
out to a file.

```sql
-- From psql client (reads from client machine, not server)
\copy customers FROM '/local/path/file.csv' CSV HEADER
\copy (SELECT * FROM customers) TO '/local/path/export.csv' CSV HEADER
```
**The distinction that actually matters here:** plain `COPY` (no
backslash) runs on the **database server** — the file path must exist
on whatever machine Postgres itself is running on, which is often not
your laptop (think back to [Lesson 19](19-network-access-and-lan-connections.md)'s
client-vs-server distinction). `\copy` (with the backslash) is a `psql`
**client-side** command — it reads/writes the file on *your* machine,
the one running `psql`, and streams the data over the connection. If
you're connecting to a remote database, `\copy` is almost always what
you actually want.

```sql
-- Bulk insert with COPY for performance
-- COPY is 10-100x faster than INSERT for large datasets
-- Use for initial data loads or ETL processes
```
**Why COPY is so much faster:** each individual `INSERT` statement pays
overhead — parsing, planning, transaction bookkeeping — every single
time. `COPY` streams rows in bulk through a much more direct path,
paying that overhead once for the whole batch instead of once per row.

---

## Useful Extensions

**Why you need this:** everything so far in this lesson (JSONB, arrays,
full-text search) ships built into Postgres. **Extensions** are
*optional* add-on modules — additional functionality you explicitly
opt into per-database, when the built-in feature set isn't enough.

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
```
`CREATE EXTENSION IF NOT EXISTS name;` is the one command that installs
any of these — Postgres ships the extension's code already, this just
switches it on for the current database. `pg_trgm` here is the same
fuzzy-search extension explained in detail earlier in this lesson.

```sql
-- uuid-ossp
SELECT uuid_generate_v4();   -- random UUID
SELECT uuid_generate_v1();   -- time-based UUID
```
A UUID is a 128-bit identifier, astronomically unlikely to collide even
generated independently on different machines — useful as a primary key
when you need IDs generated *before* a row is inserted (e.g., client-side,
before talking to the database), which a `SERIAL`/`IDENTITY` column
(assigned only at insert time) can't do.

```sql
-- pgcrypto
SELECT gen_random_uuid();                        -- random UUID (no extension in PG13+)
SELECT crypt('mypassword', gen_salt('bf'));       -- bcrypt hash
SELECT crypt('mypassword', stored_hash) = stored_hash AS valid FROM users;
SELECT encode(digest('hello', 'sha256'), 'hex'); -- SHA-256 hash
```
`crypt()`/`gen_salt('bf')` is bcrypt password hashing done directly in
SQL — `gen_salt('bf')` generates a fresh random salt each time, so
hashing the same password twice produces two *different* hashes (this is
intentional and correct — it's what defeats precomputed "rainbow table"
attacks). The login-check pattern (`crypt('mypassword', stored_hash) =
stored_hash`) works because `crypt()`, given an *existing* hash as its
salt argument, re-derives a hash using that same embedded salt — so it
only matches if the password was correct.

```sql
-- hstore (key-value)
SELECT 'color=>red, size=>large'::hstore;
SELECT ('color=>red'::hstore)->'color';  -- 'red'
```
`hstore` is a simpler, flatter key-value store than JSONB — no nesting,
only text-to-text pairs. Largely superseded by JSONB today (JSONB does
everything `hstore` does, plus nesting and richer types) — you'll
mostly encounter `hstore` in older codebases rather than choose it for
something new.

```sql
-- pg_cron (if installed)
SELECT cron.schedule('nightly-stats', '0 2 * * *', 'ANALYZE;');
SELECT cron.schedule('hourly-refresh', '0 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales');
SELECT * FROM cron.job;  -- list scheduled jobs
SELECT cron.unschedule('nightly-stats');
```
`pg_cron` schedules SQL to run automatically on a recurring schedule,
using standard cron syntax (`'0 2 * * *'` = "2:00 AM every day," the
same 5-field format from the `CronCreate` scheduling you may have seen
elsewhere). This is genuinely how production systems automate the
`ANALYZE` ([Lesson 09](09-indexes-and-performance.md)) and materialized
view refresh ([Lesson 13](13-views-and-materialized-views.md)) tasks you
learned to run manually — scheduled instead of remembered.

---

## Row Security Policies (RLS)

**Why you need this:** [Lesson 17](17-roles-users-and-access-management.md)
taught you `GRANT`/`REVOKE` — access control at the **table** level
("`app_user` can `SELECT` from `orders`"). RLS goes one level deeper:
access control at the **row** level ("`app_user` can `SELECT` from
`orders`, but only rows belonging to them"). Table-level grants alone
can't express "this row, not that one" — RLS is the tool for that.

```sql
-- Enable row-level security on a table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create a policy: users can only see their own orders
CREATE POLICY orders_isolation_policy ON orders
    FOR ALL
    TO app_user
    USING (customer_id = current_setting('app.current_customer_id')::INT);
```
`ENABLE ROW LEVEL SECURITY` first turns the feature on for the table —
by itself, this actually **blocks all access** until at least one policy
explicitly allows something (fail-closed by design). The `CREATE POLICY`
then defines the actual rule: `FOR ALL` (applies to every operation —
`SELECT`/`INSERT`/`UPDATE`/`DELETE`), `TO app_user` (which role this
rule applies to), `USING (...)` (the actual per-row condition — this
row is visible/usable **only if** this expression is true for it).

```sql
-- Set the context (in application code)
SET app.current_customer_id = '42';
SELECT * FROM orders;
-- Only returns orders where customer_id = 42
```
`current_setting('app.current_customer_id')` reads a custom session
variable — your application sets this once per connection/request
(right after a user logs in, say "you are customer 42"), and from then
on, **every** query against `orders` in that session is automatically
filtered by the policy, without the query itself needing a `WHERE
customer_id = 42` at all. This is the real value of RLS: the isolation
rule is enforced by the database itself, so even a buggy or malicious
query from that connection still can't see another customer's rows.

```sql
-- Bypass RLS for superuser/owner
ALTER TABLE orders FORCE ROW LEVEL SECURITY;  -- even owner is subject to policy
-- Or:
SET row_security = OFF;  -- superuser bypasses all policies
```
By default, a table's **owner** (often a superuser/admin role) bypasses
RLS entirely — useful for admin tooling and migrations, but worth
knowing explicitly so you're not surprised when an admin query sees
"too much" compared to what `app_user` sees. `FORCE ROW LEVEL SECURITY`
removes even that exemption, applying the policy to literally everyone,
owner included.

```sql
-- Different policies for different operations
CREATE POLICY read_policy ON orders FOR SELECT TO app_user
    USING (customer_id = current_setting('app.current_customer_id')::INT);
    
CREATE POLICY write_policy ON orders FOR INSERT TO app_user
    WITH CHECK (customer_id = current_setting('app.current_customer_id')::INT);
```
You can split the single `FOR ALL` policy above into separate,
operation-specific policies instead — here, one rule for reads
(`USING`, which filters *existing* rows) and a different rule for
inserts (`WITH CHECK`, which validates *new* rows being written). The
distinction matters: `USING` answers "can you see/touch this row that
already exists," `WITH CHECK` answers "is this new/changed row allowed
to be written at all" — a customer trying to `INSERT` an order under
someone *else's* `customer_id` gets blocked by `WITH CHECK`, separately
from whether they could `SELECT` it afterward.

---

## LISTEN / NOTIFY — Real-Time Pub/Sub

**Why you need this:** every mechanism so far is *pull*-based — your
application asks the database a question and gets an answer. Sometimes
you want the opposite: the database telling your application "something
just happened," the moment it happens, without polling in a loop asking
"anything new yet? anything new yet?" `LISTEN`/`NOTIFY` is Postgres's
built-in, lightweight publish-subscribe mechanism for exactly that.

```sql
-- In one psql session (the listener):
LISTEN order_updates;
-- Waits for notifications...

-- In another psql session (the notifier):
NOTIFY order_updates, 'Order 42 status changed to shipped';
-- The first session receives the message
```
Think of `order_updates` as a named "channel" or radio frequency.
`LISTEN order_updates` tunes a connection into that channel and then
just waits — it does nothing until someone sends something. `NOTIFY
order_updates, 'message'` broadcasts a message on that channel — every
connection currently listening receives it, essentially instantly, with
no polling involved on either side.

```sql
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
**Tying this to two things you already know:** this is the exact
trigger mechanism from [Lesson 14](14-stored-procedures-and-functions.md)
(a function that runs automatically on `INSERT`/`UPDATE`), combined with
`NOTIFY` (here called via the function form `PG_NOTIFY`, usable inside
PL/pgSQL where the plain `NOTIFY` statement isn't). `TG_OP` is a
trigger-only variable holding which operation fired it (`'INSERT'` or
`'UPDATE'` here). Real-world use: an application server keeps a
`LISTEN order_updates` connection open, and the moment any order
changes anywhere, it's notified immediately — powering a live-updating
dashboard without that dashboard ever having to poll "any changes yet?"

---

## Useful System Queries

**Why you need this:** these are diagnostic queries against Postgres's
own internal statistics tables — some you've already seen individually
in earlier lessons ([Lesson 09](09-indexes-and-performance.md)'s
`pg_stat_user_tables`), collected here as a practical reference for
day-to-day health checks on a real running database.

```sql
-- Database size
SELECT pg_size_pretty(pg_database_size(current_database()));
```
Same `pg_size_pretty()` helper from [Lesson 09](09-indexes-and-performance.md),
now applied to the whole database rather than one table/index.

```sql
-- Table sizes (sorted by largest)
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_only,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS indexes
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```
`pg_relation_size` gives just the table's own data; `pg_total_relation_size`
adds its indexes and other associated storage on top; subtracting the
two isolates "how much space is *just* the indexes costing" — directly
useful when deciding whether an index from
[Lesson 09](09-indexes-and-performance.md) is worth its storage cost.

Sample output:

| tablename | total | table_only | indexes |
|---|---|---|---|
| orders | 45 MB | 32 MB | 13 MB |
| customers | 8 MB | 6 MB | 2 MB |

```sql
-- Long-running queries
SELECT
    pid, now() - pg_stat_activity.query_start AS duration,
    query, state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > INTERVAL '5 minutes'
  AND state != 'idle';
```
`pg_stat_activity` lists every current connection/query. This filters
down to only queries that have been running for **over 5 minutes** and
aren't idle — in real operations, this is exactly how you'd spot a
runaway query worth investigating or killing before it causes bigger
problems. `now() - query_start` is the same interval arithmetic from
earlier in this course, computing "how long has this been running."

```sql
-- Active connections
SELECT count(*) FROM pg_stat_activity WHERE state = 'active';
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
```
Ordinary `COUNT`/`GROUP BY` ([Lesson 05](05-aggregations.md)) applied to
the connections table — "how many connections are doing what right now"
— useful for noticing if you're approaching Postgres's max-connections
limit.

```sql
-- Cache hit rate (should be > 99% on production)
SELECT
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) AS cache_hit_ratio
FROM pg_statio_user_tables;
```
`heap_blks_hit` counts data pages found already in memory (fast);
`heap_blks_read` counts pages that had to be fetched from disk (slow).
This ratio tells you what fraction of reads are being served from
memory — a low ratio (well under 99%) on a production system usually
means the server doesn't have enough RAM for its working data set,
directly connecting to the `Buffers: shared hit=/read=` numbers you saw
in `EXPLAIN (ANALYZE, BUFFERS)` output back in
[Lesson 09](09-indexes-and-performance.md).

```sql
-- Vacuum / analyze status
SELECT relname, last_vacuum, last_autovacuum, last_analyze, n_dead_tup, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```
The exact same `VACUUM`/`ANALYZE` monitoring query from
[Lesson 09](09-indexes-and-performance.md)'s "Table Statistics and
VACUUM" section — repeated here as part of this lesson's "day-to-day
health check" reference collection, not a new concept.

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
3. Arrays are first-class — UNNEST expands them into rows; remember 1-based indexing
4. Full-text search: `to_tsvector` preprocesses text; `tsquery` is the search query; `@@` matches
5. pg_trgm enables fuzzy search and fast `LIKE`/`ILIKE` with GIN indexes
6. Partitioning splits large tables for faster queries and near-instant bulk archival via detach+drop
7. COPY is 10-100x faster than INSERT for bulk data loading — use `\copy` from a remote client
8. Row Security Policies enforce row-level access control at the database level, beneath application code
9. LISTEN/NOTIFY gives real-time push notifications from the database, avoiding polling

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
- [Lesson 19 — Network Access & LAN Connections](19-network-access-and-lan-connections.md) — understand what actually gates a remote connection
- [Lesson 20 — Standard SQL vs PostgreSQL](20-standard-sql-vs-postgresql.md) — know what transfers if you ever work on a different engine
- [Lesson 21 — Database Engine Internals](21-database-engine-internals.md) — the mechanisms behind engine choice
- [Lesson 22 — Real-World Schema Design Walkthrough](22-real-world-schema-walkthrough.md) — every concept applied together, from one product brief
- Learn about replication and high availability
- Explore Timescale for time-series data
- Try PostGIS for geospatial queries
- Learn about logical replication and streaming
