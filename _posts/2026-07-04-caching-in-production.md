---
title: "Caching in Production: Strategies, Pitfalls, and What Companies Actually Do"
date: 2026-07-04
categories: ["System Design"]
tags: [caching, redis, memcached, performance, system-design]
---
## Why Caching Exists

A single database query takes 5ms. A cache lookup takes 1ms. The difference seems small — but at 10,000 requests per second, that 4ms gap is 40 seconds of cumulative latency per second of real time.

More importantly: databases are disk-bound. Caches are memory-bound. Memory is 1000x faster than disk. A Redis instance handling 100,000 ops/second is normal. A Postgres instance handling 10,000 writes/second is straining.

This is why caching exists. You store frequently-accessed data in a faster system so you don't have to go to the slower one.

But caching is not "put stuff in Redis and forget about it." The moment you introduce a cache, you introduce a new set of problems: stale data, memory pressure, thundering herds, cache invalidation (which is famously one of the two hard things in computer science), and the risk that your cache makes things _slower_ instead of faster.

This article covers everything you need to know about caching in production — the strategies real companies use, the failure modes, and the monitoring that tells you whether your cache is helping or hurting.

---

## The Cost of a Cache Miss

Before choosing a caching strategy, understand what you're optimizing for. The hierarchy of data access latency:

```
L1 cache reference:                   0.5 ns
L2 cache reference:                   7 ns
Mutex lock/unlock:                   25 ns
Main memory reference:              100 ns
SSD random read:                  16,000 ns
Network round trip (same DC):     500,000 ns
Database query (simple index):  1,000,000 ns  (1ms)
Database query (complex join): 10,000,000 ns  (10ms)
```

When you add caching, your hit path goes from 1-10ms (database) to <1ms (in-memory cache) to ~0.5ms (Redis in same DC). Your miss path goes from 1-10ms to (cache check + database query + cache store) — slightly _slower_ than before.

This is the first rule of caching: **a cache that misses too often makes your system slower, not faster.** The cache hit rate must be high enough that the saved time on hits outweighs the added time on misses.

Typical target: 95%+ hit rate for read-heavy workloads. Below 80%, your cache is likely making things worse.

---

## The Cache Hierarchy

Companies don't use one cache. They use a hierarchy.

```
L1: In-process memory (Go map, Python dict, LRU cache)
    ~10ns, bounded by process memory, lost on restart

L2: Local embedded (Redis on same machine, SQLite, BoltDB)
    ~100ns, survives restarts, bounded by machine memory

L3: Distributed (Redis cluster, Memcached Elasticache)
    ~0.5ms, shared across all application servers
    survives server restarts, bounded by cluster memory
```

Each layer is faster but smaller and less durable. The pattern: check L1 first (fastest, smallest). On miss, check L2 if available. On miss, check L3 (distributed cache). On miss, query the database — and populate all caches on write-back.

Most applications only have L1 (optional) and L3. L2 is useful when you have significant traffic and want to reduce network calls to the distributed cache. The 50,000x speed difference between L1 (~10ns) and L3 (~500μs) makes the local cache compelling for hot keys, but it introduces consistency problems — every server has its own copy, and updates on one server leave the others stale.

---

## Caching Strategies

There are six fundamental caching strategies. Each makes a different tradeoff between read speed, write speed, consistency, and complexity.

### Strategy 1: Cache-Aside (Lazy Loading)

The most common pattern. The application is responsible for both the cache and the database. On a read, check the cache first. On a miss, load from the database and populate the cache. On a write, write to the database and **invalidate** the cache entry (delete it, don't update it).

```
READ path:
  Check cache → MISS → fetch from database → store in cache with TTL → return

WRITE path:
  Write to database → invalidate cache entry (delete, don't update)
```

Why invalidate instead of update on write? Two reasons. First, there's a race condition: two concurrent writes can produce wrong cached values — Writer A reads old value, Writer B reads old value, both compute new values and write to cache; the last writer's cache wins, which may be stale relative to the database. Second, write amplification: if a field is updated but rarely read, you're paying to update the cache for no benefit. Invalidating is free — the next read will repopulate.

The invalidation-on-write rule is simple: always delete from cache on write, never update in place. This single rule avoids an entire class of consistency bugs.

**Used by**: Most companies. Twitter, GitHub, Stripe use this as their primary caching pattern.

---

### Strategy 2: Read-Through

The cache itself is responsible for loading data from the database. The application only talks to the cache — which acts as a proxy to the database.

```
READ path:
  Request goes to cache layer → cache MISS → cache loads from database
  → cache stores the result → cache returns to client
```

In Redis, this isn't natively supported. But systems like Amazon ElastiCache with DAX implement this. At the application level, read-through means the cache abstraction handles the database call internally.

The advantage over cache-aside: the application code is simpler. It doesn't need to know about the database. The disadvantage: the cache layer needs a connection to the database, which couples concerns that cache-aside keeps separate.

**Used by**: Amazon DAX (DynamoDB Accelerator), RedisJSON, some ORM-level caches.

---

### Strategy 3: Write-Through

Every write to the database goes through the cache first. The cache writes the data, then synchronously writes to the database. The write is complete only when both succeed.

```
WRITE path:
  Write to cache → cache writes to database → return success
```

Write-through ensures the cache is always consistent with the database — on every write, both are updated. The tradeoff: writes are slower (two writes instead of one), and if the database write fails, you must clean up the cache entry. Every write path now involves a cache write, which adds latency and a dependency on the cache being available.

**Pros**: Strong consistency between cache and database.
**Cons**: Higher write latency, more complex error handling, cache becomes a write dependency.

**Used by**: Systems where write consistency matters more than write speed. Financial systems, inventory management.

---

### Strategy 4: Write-Behind (Write-Back)

The cache acknowledges the write immediately, then asynchronously writes to the database. The application gets fast writes; the database gets batched, delayed writes.

```
WRITE path:
  Write to cache → return success immediately
  (Async) Cache queues the write → background worker writes to database
```

This is the highest-performance write pattern — the application sees sub-millisecond writes regardless of database speed. The tradeoffs are significant:

- **Data loss on cache crash**: if Redis dies before flushing to the database, writes are lost.
- **Stale reads**: if the application reads from the database directly (e.g., another service), it won't see the write until the batch finishes.
- **Duplicate writes**: if the cache crashes and restarts, the write queue is empty. The application might retry, writing the same data twice.

The batch window is typically 100ms - 1 second. Within that window, the cache is the source of truth. Outside it, the database catches up. Amazon uses this pattern for leaderboards and ranking systems where losing a second of data is acceptable.

**Used by**: High-traffic systems where write throughput is the bottleneck and some data loss is acceptable. Analytics systems, click counters, session stores.

---

### Strategy 5: Write-Around

Writes go directly to the database, bypassing the cache entirely. Only reads that miss populate the cache.

```
WRITE path:
  Write to database only — leave cache as-is
```

This sounds strange — you're not updating the cache at all. But consider: a write to a key that nobody reads. Why pay the cache write cost? The next read of that key will hit a cache miss and repopulate from the database.

Write-around is essentially "cache-aside with invalidation on write" — the standard pattern. You write to the database and invalidate the cache entry. If nobody reads it, you never paid the cache write cost. When someone reads it, they pay one miss to repopulate.

---

### Strategy 6: Refresh-Ahead

The cache proactively refreshes entries before they expire. If a key is likely to be accessed again and its TTL is near expiry, the cache fires an async reload.

```
Refresh-Ahead:
  TTL = 10 minutes
  At 9 minutes remaining: cache predicts the entry will be needed
    → Async load the fresh data from database
    → Extend the TTL
    → Next read finds the entry still present and fresh
```

This prevents cache misses on popular keys. The cost: you're always loading data even if nobody reads it after the refresh. This trades cache-miss latency for a constant background load on the database. Best suited for workloads with predictable access patterns where the cost of a miss (thundering herd) outweighs the cost of unnecessary refreshes.

**Used by**: Netflix's EVCache. Prevalent in systems with predictable access patterns.

---

## Cache Consistency

The hardest problem with caching: **the cache and the database will disagree**.

A write hits the database but the cache still has the old value. A read misses the cache and gets data that's older than the cache's expired entry. Two concurrent writes produce conflicting cache values.

### The Consistency Spectrum

```
Strong Consistency        Eventual Consistency
  │                           │
  └── Write-Through          └── Write-Behind
     Write-Around                Cache-Aside (with TTL)
```

No caching strategy gives you strong consistency for free. Write-through gives the strongest guarantees but adds latency. Write-behind gives the weakest but highest throughput.

### The Primary Reason Caches Are Stale

Cache invalidation is hard because of a fundamental race condition. Imagine two writers operating on the same key:

```
Writer A reads user (name = "Alice") from DB
Writer A writes name = "Bob" to DB
Writer A INVALIDATES cache

Writer B reads user (name = "Alice") from DB
Writer B writes name = "Carol" to DB
Writer B INVALIDATES cache

Cache is now deleted. Next read fetches "Carol" from DB. Correct ✓
```

The dangerous scenario:

```
Writer A reads user (name = "Alice") from DB
Writer A writes name = "Bob" to DB
Writer A SETS cache: user = Bob   (update in place, not invalidate)

Writer B reads user (name = "Alice") from DB
Writer B writes name = "Carol" to DB
Writer B SETS cache: user = Carol

Cache now shows "Carol" which matches DB. But this is by luck.
The SET-after-write pattern is inherently racy.
```

The fix is always: **delete from cache, don't update it in place on write.** This ensures the next read gets a fresh value from the database. The SET-after-write pattern is inherently racy because the window between the database write and cache write allows a concurrent write to pollute the cache.

### When Strong Consistency Is Required

Some data cannot be stale. Account balances, inventory counts, seat availability.

Options for strong consistency:

1. **Skip the cache entirely** for this data. Use database reads directly.
2. **Use write-through** with a single writer (queue writes through one process). This serializes access.
3. **Use database-level locking** (SELECT FOR UPDATE) even for cache reads. This serializes reads too.
4. **Use a versioned cache** — each cache entry carries a version number. Writers increment the version atomically. Reads verify the version matches the database. On mismatch, discard the cached entry and reload.

Every approach to strong consistency adds latency. There is no free lunch — you're trading cache performance for data correctness.

---

## Eviction Policies

Caches have finite memory. When the cache is full, something must be removed. The eviction policy determines what.

### TTL-Based (Time-To-Live)

The most important eviction mechanism — and the one you should always configure. Every cache entry should have a TTL. This is your safety net against stale data. If your invalidation logic fails, TTL ensures the entry is eventually removed.

```
Best practice: Set TTL on every cache write.

Common TTLs by data type:
  Session data:    30 minutes
  User profiles:   5-10 minutes
  Product catalog: 1 hour (can be longer for stable data)
  Aggregations:    1-5 minutes
  Counters:        seconds to minutes
  Configuration:   hours or days
```

### LRU (Least Recently Used)

Remove the entry that was accessed least recently. This is the default for most in-memory caches (Redis, Memcached, local caches). LRU works well for most workloads because it keeps "hot" data — entries accessed frequently stay in cache; entries accessed once and never again get evicted first.

Redis implements a variant called **approximate LRU** that samples a few keys and evicts the oldest among the sample, rather than tracking exact access order for every key. This gives near-LRU performance with O(1) overhead.

### LFU (Least Frequently Used)

Remove the entry that was accessed the fewest times. LFU is better than LRU for workloads where "once popular, always likely to be requested again" holds true. The problem with LFU: a burst of traffic to an entry makes it "hot" in LFU terms, and it stays in cache forever even after traffic drops. LRU would evict it when it stops being accessed. Practical LFU implementations use a hybrid: track frequency with a counter, use LRU for tiebreaking.

### FIFO and Random

FIFO (First In, First Out) removes the oldest entry regardless of access frequency. This is the simplest policy but performs worst in practice — popular entries get evicted after a fixed time regardless of how frequently they're accessed. Random eviction is surprisingly effective for some workloads — when the workload has no clear hot set, random eviction performs similarly to LRU.

### Redis Eviction Policies

Redis offers six policies plus `noeviction`:

```
noeviction:        return error on write when memory limit is hit
allkeys-lru:       evict any key using LRU (most common)
allkeys-lfu:       evict any key using LFU
allkeys-random:    evict a random key
volatile-lru:      evict keys with TTL set using LRU
volatile-lfu:      evict keys with TTL set using LFU
volatile-random:   evict a random key with TTL
volatile-ttl:      evict the key with shortest remaining TTL
```

The `allkeys-lru` policy is the default for production Redis clusters.

---

## Production Problems

### The Thundering Herd

Scenario: 10,000 requests arrive simultaneously for the same key. The cache just expired that key. All 10,000 requests hit a cache miss. All 10,000 requests query the database simultaneously.

```
Without mitigation:
  Cache expires for user:42
  10,000 requests → all see MISS → all query database
  Database: 10,000 identical queries in <1 second
  Results: latency spike, connection pool exhaustion, 50x load
```

This is the thundering herd — the most common cache-related production incident.

**Solution 1: Mutex-based single-flight.** Only one request goes to the database. Others wait for that request to populate the cache. In Go, the `singleflight` package from `golang.org/x/sync/singleflight` implements this exactly. A call to `group.Do(key, fn)` ensures that for any given key, `fn` executes at most once. Concurrent callers for the same key block until `fn` returns, then all receive the same result.

```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

data, err, _ := group.Do("user:"+id, func() (interface{}, error) {
    return s.queryUser(ctx, id)
})
```

With singleflight, 10,000 concurrent requests for the same key result in exactly 1 database query, not 10,000. The rest wait on the in-memory channel and get the cached result.

**Solution 2: Early expiration + background refresh.** Instead of letting the cache expire and dealing with the miss storm, refresh the entry proactively before it expires. When a read finds a key close to expiring, fire an async refresh. Combine with singleflight to prevent N goroutines from all trying to refresh the same key.

### Cache Penetration

Cache penetration is when requests query for data that _doesn't exist_. Each request misses the cache, queries the database (which also returns nothing), and the database is hit for every request even though no useful data exists.

```
Scenario: DELETE API deletes a user. Users keep requesting that user ID.
Cache: MISS (entry never existed or was deleted)
Database: no rows returned
Next request: MISS again → Database: no rows again
Repeat forever.
```

Result: The database handles every request because the cache is never populated for nonexistent keys.

**Solution 1: Cache the negative result.** Store a sentinel value indicating the key doesn't exist. Use a short TTL (seconds, not minutes) so if the data is created later, the cache eventually reflects it.

**Solution 2: Bloom filter.** Before checking the cache, check a Bloom filter that contains all known valid keys. If the Bloom filter says "definitely not present", skip both cache and database entirely. This is how Cassandra protects against cache penetration for deleted tombstones, and how RocksDB avoids expensive lookups for nonexistent keys.

### The Dogpile Effect (Cache Stampede)

Similar to thundering herd, but more specific: many concurrent writes and reads for the same key cause a cascading failure. A cache entry expires, all readers detect the miss simultaneously, all query the database in parallel, the database slows down, response times increase, connections pile up, some readers time out, retries add more load — cascading failure.

The fix: singleflight (same as thundering herd) combined with early-expiration refresh and circuit breakers that stop hitting the database when it's struggling.

### Hot Keys

A hot key is a key that receives disproportionately high traffic. A celebrity tweet. A viral product. A user with millions of followers.

```
Without mitigation:
  redis:user:celebrity_42 → cached on ONE Redis node
  ALL reads for this user go to that ONE node
  That node handles 100K requests/second
  Other nodes handle 1K requests/second each
  Node is hot → latency spikes → CPU maxes out
```

**Solution: Replicate the hot key across multiple cache nodes.** Write the hot key's value to multiple keys (or shards) so reads distribute across nodes. A read picks a random replica, spreading the load. This is exactly what Facebook does with its memcached-based cache for hot objects — they call it "replication within the cache pool."

For extreme cases, promote the hot key to L1 (local in-process cache) on every application server, eliminating the network round trip entirely.

---

## Real-World Caching Patterns from Companies

### Twitter: Fanout Cache for Timeline

When a celebrity tweets, millions of followers need to see it. Twitter's approach: pre-compute and cache each user's timeline. When a user tweets, insert the tweet into all followers' cached timelines (fanout on write). For celebrities with too many followers, switch to fanout on read — fetch the celebrity's tweets at read time and merge with the cached timeline.

The key insight: certain users (celebrities) deserve different caching treatment than normal users. This is a pattern you'll see in many systems — not all data is equal, and your caching strategy should reflect that.

### Stripe: Idempotency Cache

Stripe's idempotency system is a write-through cache with TTL. When a request comes in with an `Idempotency-Key`, Stripe checks the cache for a stored response. If found, return it. If not, process the request and store the response. The cache TTL varies: successful responses are cached for 24 hours, failed responses for a shorter period.

This is a write-through cache used for **correctness**, not performance. The cache is the source of truth for whether a request has been processed.

### Facebook: Lookaside Cache with Leases

Facebook runs one of the largest memcached deployments in the world. Their key innovation: **leases** to prevent thundering herds. When a cache miss occurs, the cache server issues a lease token to exactly one request. Only the lease holder is allowed to regenerate the value. Other requests for the same key are told to wait and retry.

The lease token is invalidated if a write occurs before the lease holder returns — this ensures stale values are never served. This solves the thundering herd without requiring application-level coordination (like singleflight).

### Amazon: Leaderboards with Write-Behind

Amazon's product ranking and leaderboard systems use write-behind caching. Reads come from Redis (fast). Writes go to Redis immediately and are asynchronously flushed to DynamoDB in batches. The ranking algorithm runs against the Redis data, reducing latency for real-time leaderboards.

The tradeoff: if Redis crashes, up to 1 second of ranking data may be lost. For leaderboards, this is acceptable. For payments, it isn't.

### Netflix: EVCache with Refresh-Ahead

Netflix's EVCache (Ephemeral Volatile Cache) is a distributed cache built on top of memcached. It uses refresh-ahead with a "stale while revalidate" pattern: when a cached entry is near its TTL, serve the stale version while asynchronously refreshing from the origin. This eliminates both the thundering herd and cache miss latency.

---

## Local In-Process Caching (L1 Cache)

Sometimes a remote cache (Redis) is too slow. The network round trip alone is 0.5-1ms even in the same data center. For hot data accessed millions of times per second, this adds up.

A local in-process cache (L1) sits in the application's memory — a simple map, an LRU cache, or a concurrent cache like `ristretto`. The speed difference is dramatic:

```
L1 Read:      ~10ns   (in-process memory access)
L3 Read:      ~500μs  (Redis, network round trip)
Database:     ~1-10ms (varies)

Ratio: L1 is ~50,000x faster than Redis
```

A two-level cache checks L1 first, then L3, then the database. L1 entries get a shorter TTL (seconds, not minutes) to limit staleness.

**The problem with L1 caches**: consistency. Each application server has its own copy of the data. If one server updates the data, all other servers still have stale L1 copies. This is why L1 caches must have short TTLs and are only appropriate for:

- Read-only reference data (config, country codes)
- Data that changes slowly (product catalogs)
- Data where seconds of staleness is acceptable
- Hot keys that would otherwise overload a single Redis node

---

## Distributed Caching Architecture

### Single Redis Instance

Simplest setup. One Redis server. Works for small to medium traffic (< 10K ops/sec). Single point of failure — if Redis goes down, all cache misses hit the database. Plan for this: your database must be able to handle the full load during a cache outage.

### Redis Sentinel

Adds failover: a sentinel cluster monitors the Redis master. If the master dies, a replica is promoted to master. Clients discover the current master via sentinel. Failover time is typically 3-10 seconds. During that window, every cache request is a miss. Make sure your database can handle the full load during a Redis failover.

### Redis Cluster

Shards data across multiple nodes. Each node holds a subset of keys (hash slots). Clients route requests to the correct node. Supports automatic failover per shard. Pros: linear scaling — 3 nodes handle 3x the throughput. No single point of failure. Cons: multi-key operations are limited (keys must be in the same hash slot for atomic operations).

Choose Redis Cluster when you need > 10 GB of cache data or > 10K ops/sec throughput.

### Memcached

Simpler than Redis. No persistence, no replication, no clustering (client-side sharding). Faster for pure key-value workloads because of simpler internal architecture. For most production systems, choose Redis for its feature set (persistence, replication, data structures, Lua scripts). Consider Memcached only when you need maximum throughput for simple key-value operations and don't need Redis's extras.

---

## Monitoring Caching

You can't know if your cache is working unless you measure it.

### The Three Metrics That Matter

**Hit Rate**

```
hit_rate = cache_hits / (cache_hits + cache_misses)

Target:
  > 95%:    excellent
  85-95%:  good — check periodically
  < 80%:   poor — your cache may be making things slower
           (each miss costs more than the no-cache path)
```

Track hit rate per key pattern, not just globally. A strong global hit rate can hide individual key patterns with terrible hit rates.

**Memory Usage**

```
used_memory / maxmemory

Target:
  < 70%:   comfortable
  70-90%:  evictions likely increasing; monitor hit rate
  > 90%:   frequent evictions; hit rate dropping; add more memory or keys
```

Track `evicted_keys` in Redis. A high eviction rate means your working set doesn't fit in cache — you need more cache memory or more selective caching.

**Latency**

```
Cache read latency (p99):
  In-process:     < 1μs
  Redis local:    < 1ms
  Redis cross-DC: < 5ms

If cache latency exceeds database latency for your workload,
the cache is not helping — it's an additional hop for no benefit.
```

### What to Alert On

- Hit rate drops below 85% for 5 minutes
- Eviction rate spikes above 1K/sec
- Redis latency p99 exceeds 10ms
- Redis memory usage exceeds 90%

---

## Cache Invalidation

Invalidation is the hardest problem because of the fundamental race between writes and reads. Here are the strategies in order of preference:

### 1. TTL + Invalidation on Write (Cache-Aside)

The standard pattern. Every entry has a TTL. On write, delete the cache entry. On read, repopulate. TTL is the safety net. This is what most production caches use — it's simple, works, and handles most failure modes.

### 2. Write-Through

Every write updates cache and database atomically. No stale reads. Higher write latency. Use when consistency is critical and writes are relatively rare.

### 3. Version-Based (Optimistic Locking)

Each entry has a version. Reads verify the version. Stale reads detect they're stale and reload. Use when cache cannot tolerate stale reads and the database is too slow for write-through.

### 4. Pub/Sub Invalidation

When a write occurs, publish an invalidation event to a message bus. All cache nodes subscribe and invalidate the key. This allows cross-node invalidation without coupling. Good for multi-DC setups where direct cache sharing is expensive — each DC has its own cache, and writes in one DC invalidate cache in another DC via Kafka.

---

## When NOT to Cache

Caching is not always the answer.

### Data That Changes Constantly

A counter that increments 1000 times/second. Every write updates the cache. Every read finds the cache valid for microseconds. The cache is a write-through bottleneck with no read benefit. Better approach: write to the database directly, or use Redis as the primary store (no database behind it).

### Data That's Rarely Read

You cache a value. Nobody reads it. The TTL expires. The cycle repeats. This is wasted memory and wasted writes. Cache only data that has a high read-to-write ratio. A good rule: cache when read frequency > write frequency by at least 10x.

### Small Datasets

If your entire dataset fits in memory and your database handles the query load fine, adding a cache is unnecessary complexity. Profile first. If queries are fast enough with indexes, don't add a cache.

### When Staleness Is Unacceptable

Account balances, seat availability, inventory counts. If a user sees a cached balance and makes a decision based on it, you could lose money. For critical data, skip the cache. Or use strict write-through with version checking. Or accept that caching this data adds risk and complexity that may not justify the performance gain.

---

## What to Actually Do

1. **Start with cache-aside.** Read through cache. On miss, load from database and populate. On write, write to database and invalidate cache. Always set a TTL. This handles 90% of caching needs.

2. **Set TTLs on everything.** TTL is your safety net. Choose TTLs based on how stale the data can be. User profiles: 5-10 minutes. Product catalog: 1 hour. Aggregations: 1 minute.

3. **Use singleflight.** Every Go service that caches should use `golang.org/x/sync/singleflight`. It prevents thundering herds at no ongoing cost. The pattern is a one-liner: wrap your database load call in `group.Do(key, fn)`.

4. **Cache the null response.** If data doesn't exist, cache that fact for a short time. This prevents cache penetration where every request for a nonexistent key hits the database.

5. **Monitor hit rate.** If hit rate is below 90%, your cache might not be helping. Check if you're caching the right keys. Check if TTLs are too short. Track hit rate per key pattern, not just globally.

6. **Don't cache everything.** Cache data with high read-to-write ratios. For data that changes every second or is rarely read, caching is wasted effort.

7. **Plan for cache failure.** When your cache goes down (and it will), your database must handle the full load. Run with the cache disabled in staging. Load test with the cache off.

8. **Use local cache wisely.** L1 cache (in-process) is 50,000x faster than Redis. Use it for hot keys that are read-only or change slowly. Accept seconds of staleness.

9. **Handle hot keys explicitly.** A key that gets 100x the traffic of other keys needs special treatment. Replicate it across cache nodes. Use local cache for it. Consider pre-computing and writing it to all cache nodes.

10. **Invalidate by deleting, not updating.** On write, always delete the cache entry. Let the next read repopulate it. This avoids the race condition where concurrent updates write conflicting values to the cache.

---

## Key Takeaways

**Caching exists** because memory is 1000x faster than disk. The tradeoff: cache hit rate must be high enough to offset the cost of misses.

**Cache-aside** (lazy loading) is the most common pattern. Reads populate the cache on miss. Writes invalidate the cache entry. TTL is the safety net against stale data.

**Write-through** ensures cache and database are always consistent at the cost of higher write latency. Use it when consistency matters and writes are relatively rare.

**Write-behind** gives the fastest writes but risks data loss. Use it when write throughput is the bottleneck and some data loss is acceptable.

**The thundering herd** is the most common cache failure. Fix it with singleflight — all concurrent requests for the same key trigger exactly one database query.

**Cache penetration** happens when requests query for data that doesn't exist. Fix it by caching the negative result (null marker with short TTL) or using a Bloom filter.

**Hot keys** require special handling. Replicate across cache nodes, use local cache, or pre-compute and distribute.

**Cache invalidation** is hard because of race conditions. Delete cache entries on write (don't update them). Use TTL as a safety net. Version entries when strong consistency is required.

**Monitor hit rate, evictions, and latency**. If any of these are unhealthy, your cache is making the system slower, not faster.

**Not everything should be cached.** High write frequency, low read frequency, small datasets, and critical data with strict consistency requirements are all poor caching candidates.
