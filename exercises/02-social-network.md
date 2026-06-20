# Exercise 02 — Social Network Queries

## Overview
Design and query a social network schema. Build the tables, load data,
then answer progressively harder questions.

## Skills Practiced
Self-joins, recursive CTEs, complex aggregations, EXISTS, window functions

---

## Step 1: Build the Schema

```sql
CREATE TABLE sn_users (
    id         SERIAL PRIMARY KEY,
    username   TEXT NOT NULL UNIQUE,
    email      TEXT NOT NULL UNIQUE,
    bio        TEXT,
    joined_at  TIMESTAMPTZ DEFAULT NOW(),
    is_active  BOOLEAN DEFAULT TRUE
);

CREATE TABLE sn_follows (
    follower_id INT REFERENCES sn_users(id) ON DELETE CASCADE,
    followee_id INT REFERENCES sn_users(id) ON DELETE CASCADE,
    followed_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id),
    CHECK (follower_id != followee_id)
);

CREATE TABLE sn_posts (
    id         SERIAL PRIMARY KEY,
    author_id  INT NOT NULL REFERENCES sn_users(id) ON DELETE CASCADE,
    content    TEXT NOT NULL,
    likes      INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sn_likes (
    user_id    INT REFERENCES sn_users(id) ON DELETE CASCADE,
    post_id    INT REFERENCES sn_posts(id) ON DELETE CASCADE,
    liked_at   TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, post_id)
);

-- Load sample data
INSERT INTO sn_users (username, email) VALUES
    ('alice',   'alice@sn.com'),
    ('bob',     'bob@sn.com'),
    ('carol',   'carol@sn.com'),
    ('dave',    'dave@sn.com'),
    ('eve',     'eve@sn.com'),
    ('frank',   'frank@sn.com'),
    ('grace',   'grace@sn.com');

-- Follow relationships
INSERT INTO sn_follows (follower_id, followee_id) VALUES
    (1, 2), (1, 3), (1, 4),   -- alice follows bob, carol, dave
    (2, 1), (2, 3),            -- bob follows alice, carol
    (3, 1),                    -- carol follows alice
    (4, 1), (4, 2), (4, 5),   -- dave follows alice, bob, eve
    (5, 4), (5, 6),            -- eve follows dave, frank
    (6, 5), (6, 7),            -- frank follows eve, grace
    (7, 1);                    -- grace follows alice

-- Posts
INSERT INTO sn_posts (author_id, content, created_at) VALUES
    (1, 'SQL is amazing!',            NOW() - INTERVAL '5 days'),
    (1, 'Learning window functions',   NOW() - INTERVAL '3 days'),
    (2, 'PostgreSQL tips',             NOW() - INTERVAL '4 days'),
    (2, 'Indexes matter',              NOW() - INTERVAL '2 days'),
    (3, 'Database design patterns',    NOW() - INTERVAL '6 days'),
    (3, 'Normalization guide',         NOW() - INTERVAL '1 day'),
    (4, 'CTEs are clean code',         NOW() - INTERVAL '3 days'),
    (5, 'Query optimization tricks',   NOW() - INTERVAL '2 days'),
    (6, 'JSONB is powerful',           NOW() - INTERVAL '1 day'),
    (7, 'Full text search in Postgres',NOW() - INTERVAL '4 days');

-- Likes
INSERT INTO sn_likes (user_id, post_id) VALUES
    (2,1),(3,1),(4,1),(5,1),(7,1),  -- 5 people liked alice's first post
    (2,2),(3,2),                     -- 2 liked second post
    (1,3),(3,3),(4,3),(5,3),         -- 4 liked bob's first post
    (1,4),(3,4),
    (1,5),(2,5),(4,5),(6,5),(7,5),  -- 5 liked carol's post
    (1,6),(2,6),(4,6),
    (1,7),(2,7),(3,7),
    (1,8),(3,8),(4,8),(6,8),
    (1,9),(2,9),(3,9),(4,9),(5,9),
    (1,10),(2,10),(3,10);

UPDATE sn_posts p SET likes = (SELECT COUNT(*) FROM sn_likes WHERE post_id = p.id);
```

---

## Queries

**Q1.** Who does Alice (id=1) follow? Show their username and when Alice followed them.

**Q2.** How many followers does each user have? Show username and follower count,
sorted by most followers first.

**Q3.** Find "mutual follows" — users who follow each other.
Show each pair once (user A id < user B id). Show both usernames.

**Q4.** Find Alice's second-degree connections — people followed by Alice's followees,
who Alice herself doesn't already follow (excluding Alice).

**Q5.** For each user, show:
- Username
- Post count
- Total likes received
- Average likes per post
Order by total likes descending.

**Q6.** Find the most liked post per user.
Show username, post content, and like count.

**Q7.** Which posts were liked by the MOST users who also follow the author?
Show content, like count, follower-likes count, and the ratio.

**Q8.** Find users who have NO followers (nobody follows them).

**Q9.** Find users who follow more people than follow them ("net outbound" followers).
Show username, following_count, follower_count, and the difference.

**Q10.** Build a "follow graph" reachability query:
Starting from user Alice (id=1), find all users reachable within 2 hops of follows.
(People Alice follows, and people they follow.)
Use a recursive CTE. Show the chain.

---

## Challenge

**C1.** Find "influencers": users with > 3 followers whose posts average > 3 likes.

**C2.** Find "echo chambers": groups of 3+ users who all follow each other.
(Every user in the group follows every other user in the group.)

---

## Answer Hints

<details>
<summary>Q3 Mutual Follows</summary>

```sql
SELECT u1.username, u2.username
FROM sn_follows f1
JOIN sn_follows f2 ON f2.follower_id = f1.followee_id AND f2.followee_id = f1.follower_id
JOIN sn_users u1 ON u1.id = f1.follower_id
JOIN sn_users u2 ON u2.id = f1.followee_id
WHERE f1.follower_id < f1.followee_id;
```
</details>

<details>
<summary>Q10 Reachability</summary>

```sql
WITH RECURSIVE reachable AS (
    SELECT followee_id AS user_id, 1 AS depth, ARRAY[1, followee_id] AS path
    FROM sn_follows WHERE follower_id = 1

    UNION ALL

    SELECT f.followee_id, r.depth + 1, r.path || f.followee_id
    FROM sn_follows f
    JOIN reachable r ON f.follower_id = r.user_id
    WHERE r.depth < 2
      AND NOT f.followee_id = ANY(r.path)
)
SELECT DISTINCT u.username, r.depth, r.path
FROM reachable r
JOIN sn_users u ON u.id = r.user_id
ORDER BY r.depth, u.username;
```
</details>
