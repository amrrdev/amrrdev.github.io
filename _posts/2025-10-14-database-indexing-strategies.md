---
title: "Database Indexing Strategies for High-Performance Systems"
date: 2025-10-14
categories: ["Database Internals"]
tags: [indexing, b-tree, query-optimization, database]
---
## Introduction

Your database has 50 million users. Someone searches by email. Without an index, the database reads every single row until it finds a match. That's a full table scan — and it gets slower with every row you add.

An index solves this. It's a separate data structure that maps values to their physical location on disk, so the database jumps straight to the row instead of scanning everything.

But indexes aren't free. Every index you create slows down writes, consumes storage, and needs maintenance. The "index everything" approach is a mistake that has taken down production systems. The skill is knowing which indexes to create, what type to use, and when to stop.

This article covers everything — from how indexes work internally, to the advanced types that most developers never touch, to the anti-patterns that silently kill performance.

---

## Topics Covered

- How indexes work under the hood
- Primary vs secondary indexes
- Clustered vs nonclustered indexes — and how MySQL and PostgreSQL differ
- B-Tree indexes and the left-prefix rule
- Hash indexes
- Covering indexes
- Partial indexes
- Expression indexes
- GIN and GiST indexes for complex data types
- Index maintenance
- Anti-patterns to avoid

---

## How Indexes Work

An index is a separate data structure that maps keys to row locations. Think of it like a book's index at the back — instead of reading every page to find "B-Tree," you look it up alphabetically, get a page number, and jump straight there.

The database does the same thing. The index stores the indexed column values in a structured format (usually a B-Tree), and each entry points to the physical location of the full row on disk.

```
Without index:                    With index:
Scan row 1... not it              Look up email in B-Tree
Scan row 2... not it              → points to row 4,521,893
Scan row 3... not it              → fetch that row directly
...
Scan row 50,000,000... found it
```

The cost: every write now updates not just the table, but every index on that table. Five indexes on a table means six writes per insert. This is the fundamental tradeoff — indexes trade write performance and storage for read performance.

---

## Primary vs Secondary Indexes

Not all indexes are equal. The distinction between primary and secondary indexes is fundamental, and it affects how data is stored and how lookups work.

### Primary Index

The primary index is built on the primary key. There is exactly one per table. In most databases, this index is **clustered** — meaning the actual row data is stored in the same order as the index. When you look something up by primary key, one seek finds the data directly.

```
PRIMARY INDEX (by id)
┌──────────────────┐     ┌──────────────────┐
│ Index:           │     │ Data:            │
│ id=1  ───────────┼────▶│ [1, Alice, 30]   │
│ id=2  ───────────┼────▶│ [2, Bob, 25]     │
│ id=3  ───────────┼────▶│ [3, Charlie, 35] │
└──────────────────┘     └──────────────────┘
                          Data stored in id order
```

### Secondary Index

Any index that isn't the primary index is a secondary index. You can have many. They're built on columns you query frequently — `email`, `status`, `created_at`. Because the data is already physically ordered by the primary key, secondary indexes can't physically reorder the data. They're always nonclustered — they store a pointer back to the row.

```
SECONDARY INDEX (by name)     DATA FILE (by id order)
┌──────────────────┐          ┌──────────────────┐
│ Alice ───────────┼─────────▶│ [1, Alice, 30]   │
│ Bob   ───────────┼─────────▶│ [2, Bob, 25]     │
│ Charlie ─────────┼─────────▶│ [3, Charlie, 35] │
└──────────────────┘          └──────────────────┘
  Sorted alphabetically         Stored by id — different order
```

Secondary indexes can have duplicate entries. Many users can have the same `status` or `country`. The primary index is always unique — no two rows share a primary key.

> **Takeaway**: Every table has one primary index, built on the primary key. Everything else is a secondary index. Secondary indexes always point back to the row — either directly or through the primary key. The difference between those two approaches matters a lot.

---

## Clustered vs Nonclustered — And How MySQL and PostgreSQL Differ

This is where most articles lose people. Clustered and nonclustered describe the relationship between the index order and the physical storage order of the data. And MySQL and PostgreSQL handle this completely differently.

### Clustered Index

A clustered index means the row data is physically stored in the order of the index. The index and the data are the same thing. There can only be one clustered index per table — data can only be sorted one way.

```
CLUSTERED INDEX (by id)
┌──────────────────────────────────┐
│  Index and data stored together  │
│                                  │
│  id=1 → [1, Alice, 30]           │
│  id=2 → [2, Bob, 25]             │
│  id=3 → [3, Charlie, 35]         │
│  id=4 → [4, Diana, 28]           │
└──────────────────────────────────┘
         ↑
  No separate "table" — this IS the table
```

Range scans on a clustered index are extremely fast. Rows with consecutive primary keys are physically next to each other on disk. One contiguous read, minimal I/O.

### Nonclustered Index

A nonclustered index is a separate structure that stores pointers to the actual rows. The index has its own order, and the data has its own order — they're independent.

```
NONCLUSTERED INDEX (by name)    DATA FILE (by id order)
┌──────────────────┐            ┌──────────────────┐
│ Alice  → ptr ────┼───────────▶│ [1, Alice, 30]   │
│ Bob    → ptr ────┼───────┐    │ [2, Bob, 25]     │
│ Charlie→ ptr ────┼─────┐ │    │ [3, Charlie, 35] │
│ Diana  → ptr ────┼───┐ │ │    │ [4, Diana, 28]   │
└──────────────────┘   │ │ │    └──────────────────┘
                       │ │ └──────────────▲
                       │ └───────────────▲│
                       └────────────────▲││
```

Range scans on nonclustered indexes involve random I/O — each pointer might point to a different physical location on disk.

### MySQL InnoDB: Clustered Primary Key, Secondary Indexes Through PK

In MySQL's InnoDB engine, the primary key is always a clustered index. The entire row is stored inside the B-Tree leaf node. This is called an **index-organized table (IOT)**.

Secondary indexes in InnoDB don't store pointers to disk locations. They store the **primary key value**. So a lookup by secondary index requires two steps: find the primary key in the secondary index, then look up the full row in the primary (clustered) index.

```
SECONDARY INDEX (name)    PRIMARY INDEX (id)    DATA
┌──────────────┐          ┌──────────────┐      ┌──────────────┐
│ Alice → id=1 │─────────▶│ id=1         │─────▶│[1,Alice,30]  │
│ Bob   → id=2 │─────────▶│ id=2         │─────▶│[2,Bob,25]    │
└──────────────┘          └──────────────┘      └──────────────┘
  Step 1: get id            Step 2: get row        Actual data
```

Two seeks for any secondary index lookup. This is why choosing a small primary key in MySQL matters — it's stored in every secondary index. A `BIGINT` primary key costs less than a `UUID` string when you have 10 secondary indexes.

```sql
-- MySQL InnoDB: secondary index stores the primary key value
CREATE TABLE users (
    id   INT PRIMARY KEY,    -- clustered index, data stored here
    name VARCHAR(50),
    age  INT
);

CREATE INDEX idx_name ON users(name);
-- internally stores (name, id) pairs
-- lookup by name: find id in this index, then find row by id in primary index
```

### PostgreSQL: Heap File, All Indexes Point Directly to Heap

PostgreSQL works differently. There is no clustered primary index by default. Data is stored in a **heap file** — rows are written in insertion order, not sorted by any key. Every index, including the primary key index, is a separate structure that stores a pointer directly to the row's physical location in the heap (called a `ctid` — a tuple identifier containing page number and slot).

```
PRIMARY INDEX (id)          HEAP FILE (insertion order)
┌──────────────┐            ┌──────────────────┐
│ id=1 → ctid  │───────────▶│ [1, Alice, 30]   │
│ id=2 → ctid  │───────────▶│ [2, Bob, 25]     │
└──────────────┘            └──────────────────┘

SECONDARY INDEX (name)      HEAP FILE (same file)
┌──────────────┐            ┌──────────────────┐
│ Alice → ctid │───────────▶│ [1, Alice, 30]   │
│ Bob   → ctid │───────────▶│ [2, Bob, 25]     │
└──────────────┘            └──────────────────┘
  Direct pointer — no double lookup needed
```

Secondary index lookups in PostgreSQL are one step — the index points directly to the row's location. But if the row moves (due to an update or VACUUM), every index pointing to that row must be updated. This is the tradeoff for avoiding the double-lookup.

PostgreSQL does support manual clustering with `CLUSTER`, but it's a one-time operation — the table doesn't stay clustered as new data comes in.

```sql
-- PostgreSQL: all indexes are nonclustered heap pointers
CREATE TABLE users (
    id   INT PRIMARY KEY,   -- separate index → heap pointer
    name VARCHAR(50),
    age  INT
);

CREATE INDEX idx_name ON users(name);
-- stores (name, ctid) pairs
-- lookup by name: one step, direct to heap location
```

> **Takeaway**: MySQL's InnoDB stores data in the primary index itself. Secondary indexes store the PK, requiring two lookups. PostgreSQL stores data in a heap file, and all indexes point directly to the heap — one lookup, but rows must update all indexes when they move.

---

## B-Tree Indexes: The Default

B-Tree indexes (technically B+Tree in most databases) are the default index type for good reason. They handle equality checks, range queries, and sorting. They maintain sorted order, provide `O(log n)` lookup, and leaf nodes are linked together for efficient range scans.

```sql
CREATE INDEX idx_users_created_at ON users(created_at);
```

This makes `WHERE created_at > '2024-01-01'` fast. It also helps `ORDER BY created_at` — the database retrieves rows in index order without a separate sort operation.

**PostgreSQL example:**

```sql
CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT      NOT NULL,
    status     TEXT        NOT NULL,
    total      NUMERIC(10,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- B-Tree index on created_at
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- This uses the index — range scan walks linked leaf nodes
EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at > '2024-01-01';

-- Index Scan using idx_orders_created_at on orders
--   Index Cond: (created_at > '2024-01-01')
--   Rows Removed by Filter: 0
--   Execution time: 2.1 ms   ← vs 800ms+ without the index
```

### The Left-Prefix Rule

Composite B-Tree indexes are sorted by the first column, then the second, then the third. Like a phone book — sorted by last name, then first name within that. An index on `(last_name, first_name, age)` helps queries that filter on:

- `last_name`
- `last_name, first_name`
- `last_name, first_name, age`

It cannot help queries filtering only on `first_name` or `age`. The leftmost column must be present.

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Uses the index — user_id is the leftmost column
EXPLAIN SELECT * FROM orders WHERE user_id = 42;

-- Also uses the index — full prefix match
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'pending';

-- Cannot use the index — user_id is missing
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- Seq Scan on orders ← full table scan
```

Column order in composite indexes matters more than most developers realize. Put the most selective column first, unless a specific query pattern requires otherwise.

> **Takeaway**: B-Tree is the default index type. It handles equality, ranges, and sorting. For composite indexes, the left-prefix rule applies — the leftmost columns must be present in the query for the index to be used. Column order is a real design decision.

---

## Hash Indexes: Equality Only

Hash indexes apply a hash function to the indexed column and store entries in a hash table. Lookups are `O(1)` — theoretically faster than B-Tree's `O(log n)`.

The catch: hash indexes are completely useless for range queries. `WHERE email > 'a'` can't use a hash index because hashing destroys ordering.

```sql
CREATE INDEX idx_users_email ON users USING HASH (email);

-- This is fast — O(1) hash lookup
SELECT * FROM users WHERE email = 'alice@example.com';

-- This cannot use the index — range query on a hash index
SELECT * FROM users WHERE email > 'a';
-- Seq Scan on users
```

In practice, reach for hash indexes rarely. A B-Tree index on an equality column is nearly as fast and also supports range queries. Hash indexes make sense only when you have very high-volume equality lookups and you've profiled to confirm B-Tree isn't sufficient.

> **Takeaway**: Hash indexes give O(1) equality lookups but are useless for range queries. B-Tree handles both. Only switch to hash if profiling shows a real bottleneck on pure equality lookups.

---

## Covering Indexes: Eliminating the Table Lookup

A covering index includes all the columns a query needs directly inside the index. The database never has to touch the actual table — everything it needs is in the index itself. This is called an **index-only scan**.

```sql
-- Regular index on email — covers the WHERE clause only
CREATE INDEX idx_users_email ON users(email);

-- Query still needs to fetch first_name, last_name, status from the heap
SELECT first_name, last_name, status FROM users WHERE email = 'alice@example.com';

-- Covering index — includes all columns the query needs
CREATE INDEX idx_users_covering ON users(email) INCLUDE (first_name, last_name, status);

-- Now the query never touches the table at all
SELECT first_name, last_name, status FROM users WHERE email = 'alice@example.com';
```

**PostgreSQL example:**

```sql
CREATE INDEX idx_users_covering ON users(email) INCLUDE (first_name, last_name, status);

EXPLAIN ANALYZE
SELECT first_name, last_name, status FROM users WHERE email = 'alice@example.com';

-- Index Only Scan using idx_users_covering on users
--   Index Cond: (email = 'alice@example.com')
--   Heap Fetches: 0    ← never touched the table
--   Execution time: 0.08 ms
```

`Heap Fetches: 0` is what you want to see. The entire query was served from the index.

Why is this fast? Index data is much smaller than full row data. Indexes fit more entries per page and are cached more effectively. Avoiding the heap fetch also eliminates the random I/O of jumping to a different location on disk.

The tradeoff: larger indexes, slower writes. Every insert now has to write more data into the index. For read-heavy workloads with specific, known query patterns, covering indexes are one of the highest-impact optimizations available.

> **Takeaway**: Covering indexes serve queries entirely from the index — no table access needed. `INCLUDE` adds non-search columns to the index leaf nodes. `Heap Fetches: 0` in `EXPLAIN ANALYZE` confirms an index-only scan. Best for read-heavy workloads with known query shapes.

---

## Partial Indexes: Indexing Only What You Query

A partial index only indexes rows that match a specific condition. If you only ever query active users, why index deleted ones?

```sql
-- Full index on email — indexes every user including deleted ones
CREATE INDEX idx_users_email ON users(email);

-- Partial index — only indexes active users
CREATE INDEX idx_active_users_email ON users(email) WHERE status = 'active';
```

If 95% of your queries filter on `status = 'active'`, the partial index is smaller, faster to maintain, and fits better in memory.

**PostgreSQL example:**

```sql
-- A common pattern: soft deletes
-- Most queries always include "WHERE deleted_at IS NULL"
CREATE INDEX idx_orders_pending ON orders(user_id)
WHERE status = 'pending' AND deleted_at IS NULL;

-- This query uses the partial index
EXPLAIN SELECT * FROM orders
WHERE user_id = 42 AND status = 'pending' AND deleted_at IS NULL;

-- Index Scan using idx_orders_pending on orders
--   Index Cond: (user_id = 42)

-- Check how much smaller it is
SELECT
    pg_size_pretty(pg_relation_size('idx_orders_pending'))    AS partial_index,
    pg_size_pretty(pg_relation_size('idx_orders_user_id'))    AS full_index;

--  partial_index | full_index
-- ---------------+-----------
--  12 MB         | 180 MB     ← 15x smaller
```

The query must include the same condition as the `WHERE` clause in the index definition for PostgreSQL to recognize it can use the partial index. `WHERE status = 'pending'` in the query matches `WHERE status = 'pending'` in the index — PostgreSQL prunes to only the matching rows.

> **Takeaway**: Partial indexes only index rows matching a condition. They're smaller, faster to maintain, and more cache-friendly than full indexes. The biggest wins come in tables with soft deletes or clear status-based access patterns.

---

## Expression Indexes: Index What You Actually Query

Sometimes you query on a transformation of your data, not the raw column. A regular index on `email` is useless for `WHERE LOWER(email) = 'alice@example.com'` — the database is searching for `LOWER(email)`, not `email`. Expression indexes solve this by indexing the result of a function.

```sql
-- Regular index — useless for case-insensitive searches
CREATE INDEX idx_users_email ON users(email);

-- Expression index — indexes the lowercased value
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

-- Now this query uses the index
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
```

**PostgreSQL example:**

```sql
-- Case-insensitive email lookup
CREATE INDEX idx_users_lower_email ON users(LOWER(email));

EXPLAIN SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- Index Scan using idx_users_lower_email on users
--   Index Cond: (lower(email) = 'alice@example.com')

-- Date truncation — queries grouped by day
CREATE INDEX idx_orders_day ON orders(DATE_TRUNC('day', created_at));

EXPLAIN SELECT COUNT(*) FROM orders
WHERE DATE_TRUNC('day', created_at) = '2024-01-15';
-- Index Scan using idx_orders_day on orders

-- JSON field extraction
CREATE INDEX idx_events_user ON events((payload->>'user_id'));

EXPLAIN SELECT * FROM events WHERE payload->>'user_id' = '42';
-- Index Scan using idx_events_user on events
```

The query must use the exact same expression as the index definition. `LOWER(email)` in the query matches `LOWER(email)` in the index. `lower(email)` also works — PostgreSQL normalizes function names. But `UPPER(email)` does not match.

> **Takeaway**: Expression indexes index the result of a function, not a raw column. The query must use the exact same expression for the optimizer to use it. Common use cases: case-insensitive searches, date truncation, JSON field extraction.

---

## GIN and GiST Indexes: For Complex Data Types

B-Tree indexes work on scalar values with a natural sort order. Some data types don't fit that model — arrays, full-text documents, geometric shapes. PostgreSQL has two specialized index types for these.

### GIN — Generalized Inverted Index

GIN indexes work by inverting the relationship. Instead of "row → values," they map "value → rows containing it." This makes them perfect for containment queries — "which rows contain this element?"

```sql
-- Full-text search
CREATE INDEX idx_posts_content ON posts USING GIN(to_tsvector('english', content));

-- This query uses the GIN index
SELECT * FROM posts WHERE to_tsvector('english', content) @@ to_tsquery('postgresql & index');

-- Array containment
CREATE INDEX idx_products_tags ON products USING GIN(tags);

-- Find all products tagged with 'sale'
SELECT * FROM products WHERE tags @> ARRAY['sale'];

-- JSONB queries
CREATE INDEX idx_events_payload ON events USING GIN(payload);

-- Find events where payload contains a specific key-value
SELECT * FROM events WHERE payload @> '{"type": "click"}';
```

GIN indexes are larger and slower to update than B-Trees. Updates require removing old entries and inserting new ones across the inverted structure. But for full-text search and array containment, there's no alternative.

### GiST — Generalized Search Tree

GiST is designed for geometric data, range types, and nearest-neighbor searches. It's a "lossy" index — it can produce false positives that require a recheck against the actual data. This makes it space-efficient for cases where exact matching isn't possible.

```sql
-- Geometric data — find all locations within a radius
CREATE INDEX idx_locations_point ON locations USING GiST(coordinates);

SELECT * FROM locations
WHERE coordinates <-> POINT(40.7128, -74.0060) < 10; -- within 10 units

-- Range type — find overlapping reservations
CREATE INDEX idx_reservations_period ON reservations USING GiST(period);

SELECT * FROM reservations
WHERE period && '[2024-01-01, 2024-01-07]'::daterange; -- overlapping dates
```

GiST vs GIN: use GIN for full-text search and array containment. Use GiST for geometric data and range types. When in doubt for full-text search, GIN is usually faster for reads; GiST updates faster.

> **Takeaway**: GIN maps values to rows — use it for full-text search, arrays, and JSONB containment. GiST is for geometric data and range types. Both are slower to update than B-Tree. Use them only when B-Tree can't model the data type.

---

## Index Maintenance: The Part Everyone Ignores

Indexes degrade over time. In PostgreSQL this is called **bloat** — as rows are inserted, updated, and deleted, the index develops dead entries and empty pages. A bloated index is larger, slower to scan, and wastes memory.

**Check index size and usage:**

```sql
-- Which indexes exist and how large they are
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    idx_scan    AS times_used,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

**Find unused indexes — these are pure overhead:**

```sql
-- Indexes that have never been used since the last stats reset
SELECT indexname, pg_size_pretty(pg_relation_size(indexrelid)) AS wasted_space
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND schemaname = 'public';
```

An index with `idx_scan = 0` is paying the write cost on every insert and update but never helping any query. Drop it.

**Rebuild bloated indexes:**

```sql
-- Rebuild without locking out writes — takes longer but safe for production
REINDEX INDEX CONCURRENTLY idx_users_email;

-- Rebuild all indexes on a table
REINDEX TABLE CONCURRENTLY users;
```

Never run `REINDEX` without `CONCURRENTLY` on a production table. Without it, the operation takes an exclusive lock that blocks all reads and writes for the duration.

**Check for bloat:**

```sql
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid))   AS current_size,
    pg_stat_get_dead_tuples(indexrelid)             AS dead_tuples
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_stat_get_dead_tuples(indexrelid) DESC;
```

A high dead tuple count means the index needs rebuilding. AUTOVACUUM handles this automatically for most workloads, but heavy write loads can outpace it. Monitor and rebuild manually when needed.

> **Takeaway**: Indexes bloat over time. Use `pg_stat_user_indexes` to find unused indexes and drop them — they slow down writes for free. Use `REINDEX CONCURRENTLY` to rebuild bloated indexes without locking. Set up monitoring. A 10x query slowdown from index bloat is real and common.

---

## Indexing Strategy in Practice

Here's how to approach indexing in a real system, in order:

**1. Start with the obvious.** Primary keys are indexed automatically. Add indexes on foreign keys (PostgreSQL doesn't do this automatically — MySQL does), and on columns that appear in frequent `WHERE` clauses.

**2. Profile before adding.** Use `EXPLAIN ANALYZE` on slow queries. Find the `Seq Scan` on large tables. Add an index. Verify with `EXPLAIN ANALYZE` again. Don't guess.

```sql
-- Find slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Then explain the worst ones
EXPLAIN ANALYZE <your slow query here>;
```

**3. Match indexes to access patterns.** If 80% of queries filter on `user_id`, that's your index. If queries always include `status = 'active'`, that's a partial index candidate. Let your actual query workload drive the decision.

**4. Consider the write/read ratio.** A table that absorbs millions of inserts per hour needs fewer indexes than a read-heavy analytics table. Every index costs write throughput.

**5. Test with real data volumes.** An index that helps on 10,000 rows may behave differently on 100 million. The query planner can choose different execution plans at different scales. Test at production scale.

**6. Monitor regularly.** Index effectiveness changes as data grows and query patterns shift. Check `pg_stat_user_indexes` monthly. Drop what's unused. Rebuild what's bloated.

---

## Anti-Patterns

**Over-indexing.** Every index slows down writes. Tables with 15+ indexes become write bottlenecks. Be ruthless — if `idx_scan = 0`, drop it.

**Indexing low-cardinality columns.** A B-Tree index on a boolean column is nearly useless. If a column has only a few distinct values, the database still scans a large fraction of the table after the index lookup. Use partial indexes instead.

**Wrong column order in composite indexes.** An index on `(status, user_id)` won't help a query filtering only on `user_id`. The leftmost column must be present. This trips up experienced developers regularly.

**Not using `CONCURRENTLY`.** Running `REINDEX` or `CREATE INDEX` without `CONCURRENTLY` takes an exclusive lock. On a large table in production, that lock can last minutes and takes down your application. Always use `CONCURRENTLY` in production.

**Indexing for one-off queries.** Don't add an index for a report that runs once a month at 3 AM. Run it on a read replica, or just let it be slow. Indexes that serve one-off queries slow down every write for nothing.

**Forgetting about index maintenance.** Indexes need care. Set up monitoring for bloat and unused indexes. Schedule regular checks.

---

## Key Takeaways

**Primary indexes** are built on the primary key. In MySQL InnoDB, the data lives inside the primary index (clustered). In PostgreSQL, the primary index points to a separate heap file.

**Secondary indexes** are everything else. In MySQL, they store the primary key value — two lookups to get a row. In PostgreSQL, they point directly to the heap — one lookup, but all indexes update when a row moves.

**Clustered indexes** store data in index order. MySQL's primary key is always clustered. PostgreSQL requires manual `CLUSTER` and doesn't stay clustered automatically.

**B-Tree** is the default. It handles equality, ranges, and sorting. Composite indexes follow the left-prefix rule — column order matters.

**Hash indexes** give O(1) equality lookups. Useless for range queries. Rarely the right choice over B-Tree.

**Covering indexes** serve queries entirely from the index — no table access. Use `INCLUDE` to add columns. Look for `Heap Fetches: 0` in `EXPLAIN ANALYZE`.

**Partial indexes** index only rows matching a condition. Smaller, faster, more cache-friendly. The biggest wins come from tables with soft deletes or filtered access patterns.

**Expression indexes** index the result of a function. The query must use the exact same expression. Essential for case-insensitive searches and computed lookups.

**GIN** for full-text search, arrays, and JSONB. **GiST** for geometric data and ranges. Both are slower to update than B-Tree.

**Maintenance matters.** Drop unused indexes. Rebuild bloated ones with `REINDEX CONCURRENTLY`. Monitor `pg_stat_user_indexes` regularly.

**Profile first.** `EXPLAIN ANALYZE` before and after every index change. Don't guess — measure.
