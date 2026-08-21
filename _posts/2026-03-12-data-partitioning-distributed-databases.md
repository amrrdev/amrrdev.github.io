---
title: "Data Partitioning in Distributed Databases: How to Split Your Data Without Breaking Everything"
date: 2026-03-12
categories: ["Database Internals"]
tags: [partitioning, sharding, distributed-databases, database]
---
## Introduction

Your single database server handled your first 100,000 users just fine. Then you hit a million. Queries slowed down. Disk filled up. You threw more RAM at it. It helped — for a while.

At some point, a single machine isn't enough. You need to split your data across multiple machines. That's partitioning.

But partitioning isn't just "put some rows here and some rows there." The moment you split data, you introduce a new class of problems: How do you know which machine holds which row? What happens when you add a new machine? What do you do when one partition gets all the traffic?

This article covers everything you need to know — the strategies, the tradeoffs, and the failure modes that bite you in production.

---

## Topics Covered

- What partitioning is and why you need it
- Vertical partitioning — splitting by column
- Horizontal partitioning (sharding) — splitting by row
- Partitioning strategies: Range, Hash, and List
- Rebalancing — what happens when you add or remove nodes
- Hotspots and how to avoid them
- When to partition and when not to

---

## What Is Partitioning?

Partitioning means splitting one large dataset into smaller pieces called **partitions**, each stored and managed independently. Instead of one server holding all your data, multiple servers each hold a slice.

Why does this help?

- **Storage**: No single machine needs to hold everything.
- **Throughput**: Reads and writes spread across machines. More machines, more capacity.
- **Query performance**: Queries only scan the partition(s) they need, not the entire dataset.

There are two fundamentally different ways to partition data. They solve different problems and are often used together.

---

## Vertical Partitioning — Splitting by Column

Vertical partitioning splits a table by **columns**. Instead of one wide table with everything, you split it into narrower tables that each hold a subset of the columns.

Imagine a `users` table:

```
users
-----
id | name | email | password_hash | profile_bio | avatar_url | last_login | created_at
```

You might split this into:

```
users_core               users_profile
----------               -------------
id | name | email        id | profile_bio | avatar_url

users_auth               users_activity
----------               ---------------
id | password_hash       id | last_login | created_at
```

**Why do this?**

The most common reason is access patterns. Your authentication service only needs `users_auth`. Your profile page only needs `users_profile`. When those queries run, they read smaller rows, fit more of them in memory, and scan less data.

There's also a security angle. Keeping sensitive data like `password_hash` in a physically separate table (or even a separate database) makes it easier to restrict access at the infrastructure level.

**The catch**: joins get expensive. If you need data from multiple vertical partitions in one query, you're joining across tables — possibly across machines. Vertical partitioning works best when your access patterns are clearly separated and cross-partition queries are rare.

> **Takeaway**: Vertical partitioning is about splitting wide tables into narrower ones by grouping columns that are accessed together. It reduces I/O per query but makes cross-partition queries harder.

**PostgreSQL example:**

PostgreSQL doesn't have a `PARTITION BY COLUMN` syntax — vertical partitioning is just a table design decision. You create the narrower tables manually and grant access per service.

```sql
-- The original wide table, split into four focused tables

CREATE TABLE users_core (
  id    BIGSERIAL PRIMARY KEY,
  name  TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE
);

CREATE TABLE users_auth (
  id            BIGINT PRIMARY KEY REFERENCES users_core(id),
  password_hash TEXT NOT NULL
);

CREATE TABLE users_profile (
  id          BIGINT PRIMARY KEY REFERENCES users_core(id),
  profile_bio TEXT,
  avatar_url  TEXT
);

CREATE TABLE users_activity (
  id         BIGINT PRIMARY KEY REFERENCES users_core(id),
  last_login TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Your auth service only touches this — no access to profile or activity
SELECT id, password_hash
FROM users_auth
WHERE id = 42;

-- Your profile service only touches this — no access to credentials
SELECT profile_bio, avatar_url
FROM users_profile
WHERE id = 42;

-- Cross-partition query — now you're joining
-- This is the cost you pay
SELECT c.name, c.email, p.profile_bio, a.last_login
FROM users_core     c
JOIN users_profile  p ON p.id = c.id
JOIN users_activity a ON a.id = c.id
WHERE c.id = 42;
```

You can enforce the separation at the database level with `GRANT` — give the auth service credentials that only have access to `users_auth`, and physically prevent it from ever touching `users_profile`. That's the security benefit vertical partitioning buys you.

---

## Horizontal Partitioning — Splitting by Row

Horizontal partitioning (commonly called **sharding**) splits a table by **rows**. Every partition holds the same columns, but each one holds a different subset of the rows.

```
Partition A          Partition B          Partition C
-----------          -----------          -----------
id | name | ...      id | name | ...      id | name | ...
1  | Alice           4  | Dave            7  | Grace
2  | Bob             5  | Eve             8  | Heidi
3  | Carol           6  | Frank           9  | Ivan
```

This is what people usually mean when they say "partitioning" or "sharding." It's the strategy databases like Cassandra, MongoDB, and CockroachDB use to scale horizontally — you add more machines and split the rows across them.

The core question is: **given a row, how do you decide which partition it belongs to?** That's where partitioning strategies come in.

> **Takeaway**: Horizontal partitioning splits rows across machines. Every partition has the same schema but a different subset of the data. The critical decision is the partitioning strategy — it determines query performance, load distribution, and how hard rebalancing is.

---

## Partitioning Strategies

### Range Partitioning

You divide rows based on a range of key values. Each partition owns a contiguous range.

```
Partition A: user_id 1       → 1,000,000
Partition B: user_id 1,000,001 → 2,000,000
Partition C: user_id 2,000,001 → 3,000,000
```

Or by date:

```
Partition A: created_at January 2024
Partition B: created_at February 2024
Partition C: created_at March 2024
```

**Why range partitioning works well:**

Range queries are fast. `WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'` hits exactly one partition and ignores the rest. That's called **partition pruning** — the query planner eliminates irrelevant partitions before scanning.

This is why time-series databases almost always use range partitioning on timestamps. Your monitoring data from last month lives in last month's partition. Your query never touches this month's.

**The hotspot problem:**

Range partitioning has a dangerous failure mode: **hotspots**. If you partition by date and most writes are for today, every write goes to today's partition. One partition gets hammered while the others sit idle. You've distributed storage but not load.

The same happens with auto-incrementing IDs. New rows always go to the highest partition. Your "current" partition absorbs all the write traffic.

```
Partition A: ids 1-1M       ← cold, rarely written
Partition B: ids 1M-2M      ← cold, rarely written
Partition C: ids 2M-3M      ← all writes go here
```

> **Takeaway**: Range partitioning is excellent for range queries and partition pruning. It's the natural choice for time-series data. But watch for hotspots — if your write pattern isn't spread across the key range, one partition will absorb all the load.

**PostgreSQL example:**

```sql
-- Create the parent table, partitioned by range on created_at
CREATE TABLE orders (
  id         BIGSERIAL,
  user_id    BIGINT NOT NULL,
  total      NUMERIC(10, 2),
  status     TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create one partition per month
CREATE TABLE orders_2024_01
  PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02
  PARTITION OF orders
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03
  PARTITION OF orders
  FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Inserts route automatically — no application logic needed
INSERT INTO orders (user_id, total, status, created_at)
VALUES (42, 99.99, 'paid', '2024-02-14');
-- This row lands in orders_2024_02, transparently

-- Partition pruning in action
-- PostgreSQL only scans orders_2024_01, ignores the other two
EXPLAIN SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';

--  Append
--    ->  Seq Scan on orders_2024_01
--          Filter: (created_at >= '2024-01-01' AND created_at < '2024-02-01')
-- orders_2024_02 and orders_2024_03 are not even mentioned

-- Drop an entire month of data instantly — no DELETE scan needed
-- Just drop the partition
DROP TABLE orders_2024_01;
```

That last point is one of the most practical benefits of range partitioning on time-series data. Deleting old data with `DELETE WHERE created_at < '2024-01-01'` on a 500 million row table takes hours and hammers I/O. Dropping a partition is instantaneous — it's a metadata operation.

---

### Hash Partitioning

You run the partition key through a hash function and use the result to decide which partition gets the row.

```
partition = hash(user_id) % number_of_partitions
```

For example, with 4 partitions:

```
hash(1001) % 4 = 2  → Partition C
hash(1002) % 4 = 0  → Partition A
hash(1003) % 4 = 3  → Partition D
hash(1004) % 4 = 1  → Partition B
```

**Why hash partitioning works well:**

It distributes rows evenly. A good hash function spreads values uniformly across partitions regardless of what the keys look like. No hotspots. Every partition gets roughly the same number of rows and the same write throughput.

This is what Cassandra and DynamoDB use by default. When you write a row, the partition key gets hashed and the hash determines which node owns it. Load is even by design.

**The range query problem:**

Hash partitioning destroys ordering. `WHERE user_id BETWEEN 1000 AND 2000` can't use partition pruning — those IDs could be scattered across every partition. You end up querying all partitions and merging results. That's called a **scatter-gather query** and it's expensive.

```
-- This is fast with range partitioning, slow with hash partitioning
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

> **Takeaway**: Hash partitioning gives you uniform load distribution and eliminates hotspots. The tradeoff is range queries — they hit every partition. Use hash partitioning when writes are the bottleneck and range queries aren't common.

**PostgreSQL example:**

```sql
-- Create the parent table, partitioned by hash on user_id
CREATE TABLE users (
  id         BIGSERIAL,
  name       TEXT NOT NULL,
  email      TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
) PARTITION BY HASH (id);

-- Create 4 partitions — MODULUS is total partitions, REMAINDER is this partition's slot
CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_p3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Inserts route automatically based on hash(id)
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');
INSERT INTO users (name, email) VALUES ('Bob',   'bob@example.com');

-- This is fast — hash(42) tells PostgreSQL exactly which partition to check
SELECT * FROM users WHERE id = 42;

-- This is a scatter-gather — PostgreSQL scans all 4 partitions
EXPLAIN SELECT * FROM users WHERE created_at > '2024-01-01';

--  Append
--    ->  Seq Scan on users_p0  (filter on created_at)
--    ->  Seq Scan on users_p1  (filter on created_at)
--    ->  Seq Scan on users_p2  (filter on created_at)
--    ->  Seq Scan on users_p3  (filter on created_at)
-- All 4 partitions scanned — no pruning possible
```

PostgreSQL's hash partitioning uses its own internal hash function, not a simple modulo. The `MODULUS` and `REMAINDER` syntax is how you tell each partition which slice of the hash space it owns. This also makes adding partitions explicit — you know exactly what you're doing when you change `MODULUS`.

---

### List Partitioning

You explicitly define which values map to which partition. Instead of a formula (range or hash), you enumerate the mapping.

```
Partition US:  country IN ('US', 'CA', 'MX')
Partition EU:  country IN ('DE', 'FR', 'GB', 'NL')
Partition APAC: country IN ('JP', 'KR', 'SG', 'AU')
```

**Why list partitioning works well:**

It's explicit and predictable. You know exactly where every row lives. It maps naturally to business boundaries — data residency laws often require that EU user data stays in EU data centers. List partitioning makes compliance straightforward to implement and audit.

It also pairs well with application-level routing. Your EU-region application server always talks to the EU partition. No need to figure out where data lives at query time.

**The maintenance overhead:**

List partitioning requires you to maintain the mapping. When you expand to a new region, you add a new partition and update the mapping. If a value doesn't match any partition definition, the insert fails. You have to think ahead.

> **Takeaway**: List partitioning is the right choice when your data has natural categorical boundaries — geography, tenant ID, business unit. It gives you explicit control over data placement, which matters for compliance and data residency requirements.

**PostgreSQL example:**

```sql
-- Create the parent table, partitioned by list on region
CREATE TABLE users (
  id         BIGSERIAL,
  name       TEXT NOT NULL,
  email      TEXT NOT NULL,
  region     TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
) PARTITION BY LIST (region);

-- Each partition explicitly declares which values it owns
CREATE TABLE users_us
  PARTITION OF users
  FOR VALUES IN ('US', 'CA', 'MX');

CREATE TABLE users_eu
  PARTITION OF users
  FOR VALUES IN ('DE', 'FR', 'GB', 'NL', 'SE');

CREATE TABLE users_apac
  PARTITION OF users
  FOR VALUES IN ('JP', 'KR', 'SG', 'AU');

-- Routes to users_eu automatically
INSERT INTO users (name, email, region)
VALUES ('Hans', 'hans@example.com', 'DE');

-- Routes to users_us automatically
INSERT INTO users (name, email, region)
VALUES ('Alice', 'alice@example.com', 'US');

-- This fails — 'BR' has no matching partition
INSERT INTO users (name, email, region)
VALUES ('Carlos', 'carlos@example.com', 'BR');
-- ERROR: no partition of relation "users" found for row
-- DETAIL: Partition key of the failing row contains (region) = (BR)

-- Fix: add a partition for the new region
CREATE TABLE users_latam
  PARTITION OF users
  FOR VALUES IN ('BR', 'AR', 'CL', 'CO');

-- Now it works
INSERT INTO users (name, email, region)
VALUES ('Carlos', 'carlos@example.com', 'BR');

-- PostgreSQL prunes to users_eu only — GDPR queries stay in their partition
SELECT id, name, email FROM users WHERE region IN ('DE', 'FR', 'GB', 'NL', 'SE');
```

The failed insert on an unmapped value is a feature, not a bug. It forces you to explicitly handle every new category rather than silently dumping it somewhere wrong. In a data residency context, that's exactly what you want — an unknown region should fail loudly, not route to the wrong jurisdiction.

---

## Rebalancing — The Hard Part

You start with 4 partitions. Six months later your data has tripled and you need 8. Or one of your nodes dies and you need to redistribute its data across the survivors. That process is **rebalancing**.

Rebalancing is where most partitioning schemes break down in practice.

### The Naive Approach: `hash(key) % N`

The formula `hash(key) % N` seems simple and elegant. But change `N` — add or remove a node — and almost every row maps to a different partition. You have to move nearly all your data.

```
Before (N=4):  hash(key) % 4
After  (N=5):  hash(key) % 5

Row with hash value 100:
  100 % 4 = 0  → was on Partition A
  100 % 5 = 0  → stays on Partition A  ✓

Row with hash value 101:
  101 % 4 = 1  → was on Partition B
  101 % 5 = 1  → stays on Partition B  ✓

Row with hash value 103:
  103 % 4 = 3  → was on Partition D
  103 % 5 = 3  → stays on Partition D  ✓

Row with hash value 104:
  104 % 4 = 0  → was on Partition A
  104 % 5 = 4  → moves to new Partition E  ✗
```

On average, `(N-1)/N` of your data moves when you add one node. With a large dataset, that's a massive data transfer — expensive, slow, and risky.

### Consistent Hashing

Consistent hashing solves this. Instead of mapping keys to partitions directly, you imagine a ring of hash values from 0 to 2^32. Both nodes and keys get hashed onto this ring. Each key belongs to the first node clockwise from it.

```
          0
         / \
    N3  /   \  N1
       |     |
    N2  \   /
         \ /
         MAX
```

When you add a new node, it takes over responsibility for the keys between itself and its predecessor on the ring. Only those keys move — the rest stay put. On average, only `1/N` of data moves when you add a node. When you remove a node, its keys move to its successor. Again, only `1/N` of data moves.

This is what Cassandra, DynamoDB, and Riak use. It makes adding and removing nodes cheap enough to do routinely.

**Virtual nodes:**

A naive consistent hashing implementation can create uneven load — one node might get a large arc of the ring and another a tiny arc. The fix is **virtual nodes** (vnodes): each physical node owns multiple points on the ring. The keys are spread more evenly, and when a node joins or leaves, its virtual node positions are distributed across the ring rather than all going to one neighbor.

### Fixed Partitions

Another approach used by Elasticsearch and Couchbase: create far more partitions than you have nodes upfront — say, 1000 partitions for 10 nodes. Each node owns roughly 100 partitions.

When you add an 11th node, you move some partitions to it. When a node dies, its partitions are redistributed. The number of partitions never changes — only which node owns each partition.

This separates two concerns: the partitioning scheme (stable, fixed) from the assignment of partitions to nodes (flexible). Partition movements are well-defined, predictable, and easy to monitor.

> **Takeaway**: `hash(key) % N` breaks when N changes — most data moves. Consistent hashing limits movement to `1/N` of data per node change. Fixed partitions offer predictable rebalancing by keeping the partition count stable and only moving ownership. Real systems (Cassandra, Elasticsearch) use one of these two approaches.

**PostgreSQL example:**

PostgreSQL doesn't do automatic rebalancing across machines — it's a single-node database at its core. But it does let you add and drop partitions manually, which is how you rebalance range-partitioned data.

```sql
-- You have orders partitioned by month.
-- January is old and taking up space. Detach it and archive it.

-- Step 1: detach the old partition (it becomes a standalone table)
ALTER TABLE orders DETACH PARTITION orders_2024_01;

-- Step 2: archive it to cold storage, or just drop it
DROP TABLE orders_2024_01;

-- Adding a new partition for an upcoming month is one line
CREATE TABLE orders_2024_04
  PARTITION OF orders
  FOR VALUES FROM ('2024-04-01') TO ('2024-05-01');
```

For hash-partitioned tables, rebalancing is harder because you can't just add one partition — the `MODULUS` changes, so every existing partition needs to be recreated with the new modulus. This is the `hash % N` problem in practice:

```sql
-- You have 4 hash partitions and want to go to 8.
-- There's no ALTER to do this. You have to rebuild.

-- Step 1: create new parent with new partition count
CREATE TABLE users_new (LIKE users INCLUDING ALL) PARTITION BY HASH (id);

CREATE TABLE users_new_p0 PARTITION OF users_new FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE users_new_p1 PARTITION OF users_new FOR VALUES WITH (MODULUS 8, REMAINDER 1);
-- ... create all 8 partitions

-- Step 2: copy data (expensive — this is the full data move)
INSERT INTO users_new SELECT * FROM users;

-- Step 3: swap
ALTER TABLE users RENAME TO users_old;
ALTER TABLE users_new RENAME TO users;
DROP TABLE users_old;
```

This is exactly why teams using PostgreSQL for heavy write workloads eventually move to Citus (a PostgreSQL extension for distributed sharding) or to systems like Cassandra or CockroachDB that handle rebalancing automatically.

---

## Hotspots and How to Deal With Them

Even with good partitioning strategies, hotspots happen. A celebrity user with millions of followers creates disproportionate read traffic on their partition. A viral product gets thousands of orders per second, all hitting the same partition.

No partitioning strategy fully prevents application-level hotspots because the skew isn't in the key distribution — it's in the access pattern.

**Common mitigations:**

**Add a random suffix to hot keys.** If user `9999` is hot, split their data across `9999_0`, `9999_1`, ..., `9999_9`. Writes go to a random suffix. Reads query all 10 and merge. This spreads load at the cost of read complexity.

**Caching.** Hot data that's mostly read benefits enormously from an in-memory cache in front of the database. The partition still exists, but most traffic never reaches it.

**Application-level routing.** Identify known hot keys and give them dedicated resources — a separate partition, a separate read replica pool, a separate cache tier.

There's no general solution. Hotspots require you to understand your access patterns and design around them explicitly.

> **Takeaway**: Hotspots are an application-level problem, not just a partitioning problem. Even perfect hash distribution doesn't help if 10% of your keys get 90% of your traffic. Identify hot keys early and handle them explicitly.

---

## Cross-Partition Queries — The Hidden Cost

Partitioning solves storage and write throughput. It creates a new problem: queries that need data from multiple partitions.

A query like `SELECT * FROM orders WHERE user_id = 42` is clean — user 42's orders are all on one partition. But `SELECT count(*) FROM orders WHERE status = 'pending'` has to hit every partition, collect results, and merge them. As partitions grow, so does the coordination overhead.

This is why **choosing the right partition key matters so much.** The partition key should match your most common query pattern. If 80% of your queries filter by `user_id`, partition by `user_id`. Those queries become single-partition — fast and cheap. The occasional cross-user query pays the scatter-gather cost, but it's acceptable.

Bad partition key choices lead to most queries becoming scatter-gather. At that point you've paid all the costs of partitioning (complexity, rebalancing, cross-partition coordination) without getting the benefit (query isolation).

> **Takeaway**: Your partition key is the most important design decision. It should match your dominant query pattern. Queries that filter on the partition key are fast. Everything else is a scatter-gather query that touches every partition.

**PostgreSQL example:**

```sql
-- orders partitioned by HASH on user_id
-- Dominant query: fetch all orders for a specific user

-- GOOD — filter on partition key
-- PostgreSQL hashes user_id=42 and goes to exactly one partition
EXPLAIN SELECT * FROM orders WHERE user_id = 42;

--  Seq Scan on orders_p2  (cost=0.00..18.50 rows=4 width=48)
--    Filter: (user_id = 42)
-- Only one partition scanned ✓

-- BAD — filter on non-partition key
-- PostgreSQL has no choice but to scan every partition
EXPLAIN SELECT * FROM orders WHERE status = 'pending';

--  Append
--    ->  Seq Scan on orders_p0  Filter: (status = 'pending')
--    ->  Seq Scan on orders_p1  Filter: (status = 'pending')
--    ->  Seq Scan on orders_p2  Filter: (status = 'pending')
--    ->  Seq Scan on orders_p3  Filter: (status = 'pending')
-- All 4 partitions scanned ✗

-- If you regularly query by status, you have two options:
-- 1. Add an index on status in each partition
CREATE INDEX ON orders_p0 (status);
CREATE INDEX ON orders_p1 (status);
CREATE INDEX ON orders_p2 (status);
CREATE INDEX ON orders_p3 (status);
-- PostgreSQL automatically uses these across all partitions

-- 2. Reconsider your partition key
-- If status queries are dominant, maybe partition by status instead
-- There's no free lunch — the partition key optimizes one access pattern
-- at the cost of others
```

`EXPLAIN` (and `EXPLAIN ANALYZE` to actually run the query) is your best tool for understanding partition behavior. If you see all partitions in the plan when you expected pruning, your query isn't filtering on the partition key — that's the diagnosis.

---

## When to Partition (and When Not To)

Partitioning adds significant complexity. Before reaching for it, make sure you've exhausted simpler options.

**Try these first:**

- Add indexes for slow queries
- Add read replicas to distribute read load
- Upgrade hardware — SSDs, more RAM
- Optimize your queries

**Partition when:**

- Your dataset no longer fits on one machine
- Write throughput exceeds what one machine can handle
- You have regulatory requirements for data locality (GDPR, data residency)
- Query performance is bounded by data volume, not query structure

**Don't partition when:**

- Your dataset is tens of GBs, not TBs — modern hardware handles this easily
- Your access patterns are unpredictable — you'll pick the wrong partition key
- You need complex cross-record transactions — distributed transactions are hard
- You're just worried about future scale — premature partitioning creates problems you don't have yet

> **Takeaway**: Partitioning is a last resort after simpler scaling strategies are exhausted. It solves real problems at real scale, but it adds operational complexity, makes transactions harder, and punishes you for wrong partition key choices. Don't do it until you need to.

---

## Key Takeaways

**Vertical partitioning** splits tables by column. Good for separating access patterns and reducing I/O per query. Makes cross-partition joins expensive.

**Horizontal partitioning (sharding)** splits tables by row across machines. Scales storage and write throughput. The partition key choice determines everything.

**Range partitioning** is fast for range queries and natural for time-series data. Prone to hotspots when writes concentrate at one end of the range.

**Hash partitioning** gives uniform distribution and no hotspots. Breaks range queries — they become expensive scatter-gather operations.

**List partitioning** maps specific values to specific partitions. Ideal for geographic or categorical data. Requires manual maintenance as categories grow.

**Rebalancing** is the hardest operational challenge. `hash % N` is naive — most data moves when N changes. Use consistent hashing or fixed partitions instead.

**Hotspots** are an application-level problem. No partitioning strategy eliminates them. Identify hot keys and handle them explicitly with caching, key splitting, or dedicated resources.

**Cross-partition queries** are expensive. Your partition key should match your dominant query pattern to minimize scatter-gather queries.

**Partition last.** Vertical scaling and read replicas solve most problems at most scales. Reach for partitioning when you've genuinely hit the limits of a single machine.
