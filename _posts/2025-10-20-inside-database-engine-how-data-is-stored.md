---
title: "How Databases Actually Store Data: Pages, Tuples, and the Architecture That Matters"
date: 2025-10-20
categories: ["Database Internals"]
tags: [storage-engine, postgres, pages, database-internals]
---
## Introduction

Most developers understand SQL. Fewer understand what happens after the query planner finishes — how rows are physically laid out on disk, why certain writes cost more than others, why VACUUM exists, and why RocksDB and PostgreSQL make opposite tradeoffs and are both right.

This article covers the storage layer with no abstraction. Every concept here explains a real behavior you can observe in production.

---

## Topics Covered

- The page: the fundamental unit of all database I/O
- Slotted pages: how rows are physically organized and why
- Tuple headers and MVCC: how PostgreSQL versions rows
- TOAST: what happens when a row doesn't fit on a page
- Write amplification: the hidden cost of in-place updates
- LSM Trees: a completely different storage philosophy
- SSTables, MemTables, and the level structure
- Bloom filters: the probabilistic shortcut that makes LSM reads viable
- Compaction strategies and write amplification in LSM trees
- When each architecture wins — and why

---

## The Page: The Fundamental Unit of I/O

Everything in a relational database is organized around one concept: the **page**. A page is the smallest unit of data a database reads from or writes to disk. You never read a single row from disk. You never write a single byte. You always read and write entire pages.

PostgreSQL uses 8KB pages by default, set at compile time. MySQL InnoDB uses 16KB. SQLite uses 4KB. These numbers aren't arbitrary — they align with OS memory page sizes and filesystem block sizes to minimize I/O overhead.

The consequence of page-based I/O is fundamental: **any operation that touches a row reads the entire page containing that row into memory, modifies it, and writes the entire page back to disk.** A one-byte update is still an 8KB read followed by an 8KB write. This is called **read-modify-write**, and it is the central cost model of page-based storage.

The page size determines how many rows fit in a single I/O operation. For a table with 100-byte rows and 8KB pages, roughly 80 rows share a page. A sequential scan reads those 80 rows in a single disk operation. A random lookup by primary key reads one page and potentially wastes the other 79 rows loaded alongside the target row.

---

## Slotted Pages: The Internal Structure of a Heap Page

A naive page design would store rows sequentially from the beginning. This breaks immediately when rows are deleted or updated to different sizes — you either leave gaps or shift every subsequent row, invalidating every index pointer in the process.

The solution is the **slotted page** design, used by PostgreSQL, MySQL InnoDB, Oracle, and SQL Server.

```
┌────────────────────────────────────────────────────────────┐
│  Page Header (24 bytes in PostgreSQL)                      │
│  LSN | Checksum | Lower | Upper | Special | Flags          │
├────────────────────────────────────────────────────────────┤
│  Slot 1: (offset=7900, length=60)                          │
│  Slot 2: (offset=7820, length=80)  ← slot directory        │
│  Slot 3: (offset=0, length=0)      ← dead slot             │
│  Slot 4: (offset=7760, length=58)  grows downward          │
├────────────────────────────────────────────────────────────┤
│                                                            │
│                    Free Space                              │
│              (between Lower and Upper)                     │
│                                                            │
├────────────────────────────────────────────────────────────┤
│  Tuple 4 (at offset 7760)          ← tuples grow upward    │
│  Tuple 2 (at offset 7820)          from end of page        │
│  Tuple 1 (at offset 7900)                                  │
└────────────────────────────────────────────────────────────┘
```

The page header contains two critical pointers:

- **Lower**: points to the end of the slot directory (grows downward as slots are added)
- **Upper**: points to the start of the tuple area (grows upward from the end of the page as tuples are added)

Free space is everything between Lower and Upper. When PostgreSQL needs to insert a row, it checks if `Upper - Lower >= tuple_size + slot_entry_size`. If yes, it writes the tuple at Upper, decrements Upper by the tuple size, appends a slot entry at Lower, and increments Lower by the slot entry size.

The slot directory is the key innovation. Each slot entry is just 4 bytes — an offset and a length. Index entries and row pointers refer to `(page_id, slot_number)`, not `(page_id, byte_offset)`. This indirection is what makes the slotted page design resilient.

**What happens on DELETE**: PostgreSQL does not immediately remove the tuple. It sets the `xmax` field in the tuple header to the transaction ID that deleted it and marks the slot as pointing to a dead tuple. The slot entry remains. The space remains occupied. The tuple is invisible to transactions that started after the deletion committed, but physically it's still there. This is the foundation of MVCC — dead tuples are the price of non-locking reads.

**What happens on UPDATE**: PostgreSQL treats an UPDATE as a DELETE + INSERT. The old tuple is marked dead (xmax set). A new tuple is written elsewhere — possibly on the same page if there's space, possibly on a different page. The old tuple's `t_ctid` field is updated to point to the new version, forming a version chain. Index entries still point to the old tuple's slot, and the index access method follows the `t_ctid` chain to find the current version.

**What happens on DELETE then INSERT**: The dead slot from the DELETE can be reused. When PostgreSQL inserts a new tuple, it can overwrite a dead slot's entry to point to the new tuple's location. The slot number is recycled, but the tuple at the new offset has no relationship to the dead tuple that previously occupied that slot number.

**Free Space Map**: PostgreSQL maintains a **Free Space Map (FSM)** for every table — a separate file (`tablename_fsm`) that tracks approximately how much free space each page has. The FSM is a tree structure where leaf nodes store a fraction of free space for each page (in units of 1/256th of a page). When PostgreSQL needs to insert a row, it consults the FSM to find a page with sufficient free space rather than scanning sequentially.

**Visibility Map**: Alongside the FSM, PostgreSQL maintains a **Visibility Map (VM)** (`tablename_vm`). Each page gets two bits: one indicating all tuples on the page are visible to all current transactions (no dead tuples), one indicating all tuples are frozen (safe for transaction ID wraparound). Sequential scans use the VM to skip VACUUM work on already-clean pages. Index-only scans use the VM to avoid heap fetches when all tuples on a page are known visible.

> **Takeaway**: Slotted pages decouple physical location from logical identity. Index entries reference slot numbers, not byte offsets. Deletes mark tuples dead but leave them physically in place. Updates write new versions elsewhere. The Free Space Map and Visibility Map are the operational infrastructure that makes this efficient.

---

## Tuple Headers and MVCC

Every row in PostgreSQL is a **tuple** with a 23-byte header (plus padding to alignment) before the actual column data.

```
Tuple Header (23 bytes):
┌──────────┬──────────┬──────────┬──────────┬──────────┬────────────┐
│  t_xmin  │  t_xmax  │  t_ctid  │ t_infomask │ t_hoff │  t_bits   │
│ (4 bytes)│ (4 bytes)│ (6 bytes)│  (2+2 bytes)│(1 byte)│(variable) │
└──────────┴──────────┴──────────┴──────────┴──────────┴────────────┘
```

- **t_xmin**: transaction ID that inserted this tuple. A tuple is visible to a transaction if its xmin committed before that transaction started.
- **t_xmax**: transaction ID that deleted or updated this tuple. Zero means the tuple is not dead. A non-zero xmax means the tuple is either deleted (if that transaction committed) or in the process of being deleted (if that transaction is still in progress).
- **t_ctid**: a `(page_id, slot_number)` pair pointing to the current version of this tuple. For a live, non-updated tuple, `t_ctid` points to itself. For an updated tuple, `t_ctid` points to the newer version.
- **t_infomask**: bit flags encoding visibility hints, whether the tuple has nulls, whether xmin/xmax are known committed or aborted. These hints avoid expensive transaction status lookups on the pg_clog (commit log) for tuples whose visibility is already known.
- **t_hoff**: byte offset from the start of the header to the actual column data, accounting for the null bitmap.
- **t_bits**: the null bitmap — one bit per column, set if that column is NULL. Only present if the table has nullable columns (`HEAP_HASNULL` bit in t_infomask).

### MVCC Visibility Rules

When PostgreSQL evaluates whether a tuple is visible to a transaction with snapshot `S`:

```
Tuple is visible if:
  1. t_xmin committed before S was taken
     AND
  2. t_xmax is zero
     OR t_xmax has not yet committed
     OR t_xmax started after S was taken
```

This check happens for every tuple accessed during a query. The `t_infomask` hints (`HEAP_XMIN_COMMITTED`, `HEAP_XMAX_COMMITTED`) cache the commit status to avoid consulting `pg_clog` repeatedly. Once a tuple's xmin is known committed, the hint bit is set and subsequent visibility checks are cheap.

### Transaction ID Wraparound

Transaction IDs in PostgreSQL are 32-bit unsigned integers. With ~2 billion possible values, a busy database generating 1000 transactions per second exhausts the space in about 68 years. But PostgreSQL's visibility model treats transaction IDs as circular — a transaction ID is "in the past" if it's within 2^31 of the current ID. This means after 2^31 transactions (~2.1 billion), old xmin values wrap around and appear to be "in the future," making old tuples invisible.

This is why `VACUUM FREEZE` exists. It replaces xmin values with a special `FrozenTransactionId` (value 2) that is always considered "in the past" by all transactions. Once a tuple is frozen, its visibility is no longer contingent on transaction ID arithmetic. The `age(datfrozenxid)` metric in `pg_database` tells you how close a database is to wraparound — above ~200 million, autovacuum becomes aggressive; above 1.6 billion, PostgreSQL will refuse new connections to prevent data corruption.

> **Takeaway**: Every PostgreSQL tuple carries the transaction IDs that created and destroyed it. Visibility is computed per-transaction by comparing these IDs against the transaction's snapshot. Dead tuples accumulate until VACUUM removes them. Transaction ID wraparound is a real operational concern — monitor `age(datfrozenxid)` in production.

---

## TOAST: Storing Values Larger Than a Page

A single page is 8KB. A row must fit on a single page (with the exception of TOAST). What happens when a `TEXT` column contains a 1MB document?

**TOAST** (The Oversized-Attribute Storage Technique) is PostgreSQL's mechanism for storing large column values outside the main heap. Each table with potentially large column types automatically gets a companion TOAST table (named `pg_toast.pg_toast_<oid>`). The TOAST table has its own heap, its own indexes, and its own VACUUM lifecycle.

### TOAST Storage Strategies

PostgreSQL applies TOAST based on column storage strategy, which you can set per column:

- **PLAIN**: no compression, no out-of-line storage. Used for small fixed-size types (integer, boolean). If the row doesn't fit on a page, the INSERT fails.
- **EXTENDED** (default for text, jsonb, arrays): compress first, then store out-of-line if still too large. PostgreSQL uses LZ4 (since v14) or pglz by default.
- **EXTERNAL**: store out-of-line without compression. Useful when the value will be accessed as a substring — decompression requires reading the entire value, but with EXTERNAL, you can use `substring()` with seek-based I/O.
- **MAIN**: compress first, but prefer to keep in-line. Only move out-of-line as a last resort.

### TOAST Threshold and Chunking

The TOAST threshold is 2KB (actually `TOAST_TUPLE_THRESHOLD = 2048` bytes). If a row exceeds 2KB after accounting for header overhead, PostgreSQL triggers TOAST processing on the largest columns until the row fits.

Out-of-line values are split into **chunks** of 2000 bytes each (slightly less than 2KB to fit chunk metadata in a page). Each chunk is stored as a separate row in the TOAST table with a chunk ID, chunk sequence number, and the raw data. The main table stores a **TOAST pointer** — 18 bytes containing the OID of the TOAST table, the OID of the value, the decompressed size, and the compressed size.

```
Main table row:
┌──────────┬──────────┬──────────────────────┐
│  id: 42  │ name: X  │ content: [TOAST ptr] │  ← 18-byte pointer
└──────────┴──────────┴──────────────────────┘
                              │
                              ▼
TOAST table:
┌───────────┬──────┬──────────────────────────────┐
│ chunk_id  │ seq  │ chunk_data (2000 bytes)       │
│ chunk_id  │ seq  │ chunk_data (2000 bytes)       │
│ chunk_id  │ seq  │ chunk_data (remaining bytes)  │
└───────────┴──────┴──────────────────────────────┘
```

### Performance Implications of TOAST

TOAST is transparent — queries work without any changes. But it has significant performance implications:

**Column selectivity matters more with TOAST**: `SELECT id, name FROM posts` never touches the TOAST table even if `content` is TOASTed. `SELECT *` fetches every TOASTed column, triggering additional I/O for each one. This is a strong argument for not using `SELECT *` beyond just theoretical cleanliness.

**TOAST and sequential scans**: A full table scan on a table with TOASTed columns only reads the TOAST table if the query references those columns. Table bloat statistics from `pg_class.relpages` only count main heap pages, not TOAST pages — the actual disk footprint of a table with large columns is significantly higher than `relpages * 8KB`.

**TOAST and VACUUM**: The TOAST table has its own dead tuples. When you update a row with a TOASTed column, the old TOAST chunks become dead and need to be vacuumed independently. VACUUM on the main table triggers VACUUM on associated TOAST tables automatically, but TOAST table bloat can be significant and is often overlooked.

> **Takeaway**: TOAST automatically handles values larger than 2KB by compressing and storing them in a separate table. Only accessed columns incur I/O. TOAST tables have their own bloat and need their own VACUUM attention. `SELECT *` on tables with large columns silently triggers TOAST reads for every row.

---

## B-Tree Storage Internals

PostgreSQL's default index type is a B+Tree (the database literature says "B-Tree" but virtually all implementations are B+Trees). Understanding the internal structure explains index performance characteristics that are otherwise mysterious.

### B+Tree Structure

A B+Tree has three types of nodes:

**Internal nodes** store only keys and child pointers — no actual row data. Each internal node in PostgreSQL's implementation holds up to `(page_size - header) / (key_size + pointer_size)` entries. For an 8KB page with 8-byte integer keys, this is roughly 500 entries per internal node.

**Leaf nodes** store keys paired with heap pointers (`(page_id, slot_number)` — called `ItemPointer` or `tid` in PostgreSQL). Leaf nodes are linked in a doubly-linked list in key order.

**The root node** is either an internal node (for large indexes) or a leaf node (for small indexes that fit on one page).

```
B+Tree structure:
                    [Internal: 30 | 60]
                   /         |          \
        [Int: 15|20]    [Int: 40|50]    [Int: 70|80]
        /    |    \       ...              ...
[Leaf] [Leaf] [Leaf] ←──────────────────────────────▶ (linked list)
  ↓      ↓      ↓
heap  heap    heap
tids  tids    tids
```

### Page Splits and Tree Height

When a leaf page is full and a new entry must be inserted, PostgreSQL splits the page. The median key moves up to the parent internal node. Two half-full pages replace the one full page. If the parent is also full, the split propagates upward — potentially all the way to the root.

When the root splits, a new root is created one level up. This is the only way a B+Tree grows in height. A B+Tree with `N` entries and branching factor `B` has height `log_B(N)`. With 500 entries per internal node and 1 billion rows, the tree height is `log_500(1,000,000,000) ≈ 3.5` — meaning at most 4 page reads to locate any entry. This is why B-Tree indexes remain fast at any realistic data size.

### The Fill Factor

PostgreSQL's B-Tree implementation uses a **fill factor** (default 90%) that leaves 10% of each leaf page free during initial builds and during splits. This free space absorbs future insertions without triggering immediate page splits. For tables that receive mostly INSERT workloads with sequential keys (like auto-incrementing IDs), the default fill factor wastes space. For tables with frequent UPDATEs to indexed columns (which are DELETE + INSERT in the index), a lower fill factor (70-80%) reduces split frequency.

### HOT Updates and Index Maintenance

When you UPDATE a row in PostgreSQL and the update does not change any indexed column, PostgreSQL can use **Heap-Only Tuple (HOT)** optimization. Instead of inserting a new index entry, PostgreSQL:

1. Writes the new tuple version on the same heap page as the old version
2. Sets the old tuple's `t_ctid` to point to the new version
3. Does NOT insert anything into the index

Index scans follow the `t_ctid` chain from the index-referenced tuple to the current version. HOT chains are pruned during page access — when a page is accessed and the chain is found, dead tuples in the chain are reclaimed immediately without waiting for VACUUM.

HOT is only possible when the new tuple version fits on the same page as the old version (free space permitting) and no indexed column is modified. This is why a low fill factor on heavily-updated tables pays off: more free space on heap pages = more HOT updates = fewer index writes = lower write amplification.

> **Takeaway**: B+Tree indexes have height `O(log N)` with branching factors of hundreds, making 3-4 page reads sufficient for any realistic dataset. Page splits propagate upward and are managed with fill factors. HOT updates avoid index writes entirely when no indexed column changes — this is PostgreSQL's primary mechanism for reducing update overhead.

---

## Write Amplification in Page-Based Storage

The read-modify-write pattern of page-based storage creates **write amplification** — the ratio of bytes written to disk versus bytes logically changed by the application.

For a single-row UPDATE in PostgreSQL:

```
Application changes: 1 row (e.g., 100 bytes)

Actual disk writes:
  1. WAL record for the change: ~100-200 bytes (sequential)
  2. Heap page containing the new tuple version: 8KB
  3. Each index covering changed columns: 8KB per index page

For a table with 3 indexes on updated columns:
  Total write: 1 WAL record + 1 heap page + 3 index pages
             = ~200 bytes + 8KB + 24KB
             = ~33KB written to serve a 100-byte logical change
  Write amplification: ~330x
```

The WAL write is sequential and cheap. The heap and index writes are random — each page lives at a different location on disk. On mechanical disks, random I/O is catastrophically slower than sequential I/O. On SSDs, the gap is smaller (SSDs handle random reads well but random writes still cause wear amplification internally), but the principle holds.

PostgreSQL mitigates this with:

- **The shared buffer pool**: pages are modified in memory first and written to disk lazily by the background writer. Multiple modifications to the same page accumulate in memory and are written in a single disk operation. This is why a large `shared_buffers` setting (typically 25% of RAM) dramatically reduces physical write amplification for write-heavy workloads with temporal locality.
- **WAL batching**: `wal_buffers` accumulates WAL records in memory. A single `fsync` flushes many records. `synchronous_commit = off` delays WAL flushes further (with the tradeoff of potential data loss in a crash — the last few transactions may be lost, but the database remains consistent).
- **Checkpoint coalescing**: the checkpoint process writes dirty pages to disk periodically. Between checkpoints, the same page can be modified many times in memory and written only once at checkpoint. `checkpoint_completion_target` spreads checkpoint I/O over time to avoid spikes.

Despite these mitigations, random write amplification is the fundamental ceiling on write throughput for page-based storage. This is why LSM trees were invented.

> **Takeaway**: A single logical write generates many physical writes — WAL records, heap page, index pages. Write amplification of 100-300x is common. Large buffer pools and WAL batching reduce the impact by coalescing writes in memory. But the random I/O pattern is a fundamental constraint of in-place page updates.

---

## LSM Trees: Sequential Writes as a First Principle

The Log-Structured Merge Tree (LSM Tree) makes one radical decision: **never update data in place**. All writes are appends. Disk writes are always sequential. Random I/O is eliminated from the write path entirely.

This is a complete reversal of the page-based philosophy. Instead of maintaining sorted, indexed data in place and paying random I/O on every write, LSM trees accept writes sequentially, sort them in memory, periodically flush sorted batches to disk, and merge those batches in the background.

### The Write Path

```
Write arrives (key=X, value=V)
         │
         ▼
┌─────────────────┐
│    WAL (disk)   │  ← Sequential append, for crash recovery
└─────────────────┘
         │
         ▼
┌─────────────────┐
│   MemTable      │  ← In-memory sorted structure (red-black tree or skip list)
│   (in RAM)      │  ← All writes go here first
└─────────────────┘
         │
         │ (when MemTable reaches threshold, e.g., 64MB)
         ▼
┌─────────────────┐
│  Immutable      │  ← Previous MemTable frozen while flushing
│  MemTable       │  ← New writes go to a new active MemTable
└─────────────────┘
         │
         ▼
┌─────────────────┐
│  Level 0        │  ← SSTable written to disk sequentially
│  SSTable        │  ← Sorted by key, immutable once written
└─────────────────┘
```

The WAL ensures crash recovery — if the process dies with data in the MemTable, the WAL replays the missing writes on restart. The MemTable (typically a red-black tree or skip list) maintains sorted order for efficient in-order flushing to disk.

The critical property: the MemTable flush to disk is **one sequential write** of the entire MemTable contents, sorted by key. No random seeking. The disk write bandwidth is limited only by disk throughput, not by seek latency.

### SSTable Structure

A **Sorted String Table (SSTable)** is an immutable, sorted file on disk. It is written once and never modified. Updates and deletions are represented as new entries — a deletion is a **tombstone** (a special marker indicating the key is deleted).

```
SSTable layout:
┌─────────────────────────────────────────────────────┐
│  Data Block 1 (4KB compressed)                      │
│  [key1:val1][key2:val2]...[keyN:valN]               │
├─────────────────────────────────────────────────────┤
│  Data Block 2 (4KB compressed)                      │
│  [keyN+1:valN+1]...[keyM:valM]                      │
├─────────────────────────────────────────────────────┤
│  ...                                                │
├─────────────────────────────────────────────────────┤
│  Index Block                                        │
│  [first_key_of_block_1 → offset_1]                  │
│  [first_key_of_block_2 → offset_2]                  │
│  ...                                                │
├─────────────────────────────────────────────────────┤
│  Meta Block                                         │
│  [Bloom filter data]                                │
│  [Compression type, block size, entry count, ...]   │
├─────────────────────────────────────────────────────┤
│  Footer (fixed size)                                │
│  [Offset of Index Block] [Offset of Meta Block]     │
│  [Magic number for validation]                      │
└─────────────────────────────────────────────────────┘
```

**Data blocks** are independently compressed. To find a key, you don't decompress the entire SSTable — you binary search the index block to find which data block might contain the key, decompress only that block, and binary search within it.

**Compression per block** is significant. LZ4 achieves 2-4x compression on typical data with negligible decompression overhead. Snappy achieves similar ratios with lower CPU usage. zstd achieves higher compression ratios (3-6x) at higher CPU cost. Since SSTable reads always involve decompression, CPU cost matters — LZ4 is the most common production choice.

### The Level Structure

SSTables are organized into **levels**. Each level has a maximum size, and SSTables within a level (except Level 0) have non-overlapping key ranges.

```
Level 0: [SST-A: 1-100] [SST-B: 50-200] [SST-C: 150-300]
          ← Key ranges CAN overlap — just flushed from MemTable
          ← Limited to 4 SSTables before compaction triggered

Level 1: [1-250] [251-500] [501-750] [751-1000]
          ← Non-overlapping key ranges
          ← Max size: e.g., 256MB

Level 2: [1-100][101-200]...[901-1000]
          ← Non-overlapping key ranges
          ← Max size: e.g., 2.5GB (10x Level 1)

Level 3: ... (10x Level 2)
Level 4: ... (10x Level 3)
...
Level N: Most data lives here — all compacted, no overlap
```

**Level 0 is special**: SSTables here can have overlapping key ranges because they're flushed directly from the MemTable without sorting against existing SSTables. A read that requires checking Level 0 must check every Level 0 SSTable (there may be 4-8 before compaction is triggered). This is why Level 0 is kept small and compacted aggressively.

**Levels 1 and above**: Each level has non-overlapping key ranges. A key can appear in at most one SSTable per level (ignoring tombstones and old versions). A read that checks Level 1 reads at most one SSTable.

The size multiplier between levels (typically 10x) determines the total number of levels for a given dataset size. For 1TB of data with 256MB at Level 1: Level 2 = 2.5GB, Level 3 = 25GB, Level 4 = 250GB, Level 5 = 2.5TB. Most data sits at Level 4-5. The tree height is `log_10(data_size / L1_size)`.

---

## Bloom Filters: Making LSM Reads Viable

Without Bloom filters, a point lookup (find key X) in an LSM tree would require checking every SSTable at every level — potentially dozens of disk reads to confirm a key doesn't exist. Bloom filters reduce negative lookups to near-zero disk I/O.

### How Bloom Filters Work

A Bloom filter is a bit array of size `m` and `k` hash functions. To insert a key:

```
Insert key "alice":
  hash_1("alice") % m = 23  →  set bit 23
  hash_2("alice") % m = 91  →  set bit 91
  hash_3("alice") % m = 147 →  set bit 147

Query key "bob":
  hash_1("bob") % m = 23   →  bit 23 is set (was set by "alice")
  hash_2("bob") % m = 204  →  bit 204 is NOT set
  → "bob" is DEFINITELY NOT in this SSTable (one unset bit proves absence)

Query key "carol":
  hash_1("carol") % m = 23  →  bit 23 is set
  hash_2("carol") % m = 91  →  bit 91 is set
  hash_3("carol") % m = 147 →  bit 147 is set
  → "carol" MIGHT be in this SSTable (false positive — "carol" was never inserted)
```

Bloom filters have:
- **No false negatives**: if a key is in the SSTable, all its hash positions will be set. A "definitely not present" answer is always correct.
- **False positives**: a key might appear to be present when it isn't, if all its hash positions were set by other keys.

The false positive rate `p` is:
```
p ≈ (1 - e^(-kn/m))^k

where:
  k = number of hash functions
  n = number of inserted elements
  m = number of bits

Optimal k = (m/n) * ln(2) ≈ 0.693 * (m/n)

At 10 bits per key (m/n = 10):
  optimal k ≈ 7
  false positive rate ≈ 0.8%
```

RocksDB defaults to 10 bits per key, giving ~1% false positive rate. For an SSTable with 1 million entries, the Bloom filter is 1.25MB — small enough to keep in memory (RocksDB caches Bloom filters in its block cache).

### Bloom Filter Placement

Each SSTable has its own Bloom filter covering all keys in that SSTable. A point lookup proceeds:

```
Lookup key X:

1. Check MemTable: O(log n), in memory → found? return. not found? continue.
2. Check immutable MemTable (if flushing): O(log n), in memory → found? return.
3. For each Level 0 SSTable (check all, they may overlap):
   a. Check Bloom filter (in memory): "definitely not here"? skip. → continue.
   b. Binary search index block: find candidate data block → read + decompress → search.
4. For Level 1: identify the one SSTable whose key range includes X.
   a. Check Bloom filter: "definitely not here"? done, return not found.
   b. Binary search → read candidate block → search.
5. Repeat for Level 2, 3, ...
```

With 1% false positive rate and 7 levels, the expected number of false positive SSTable reads per lookup is `7 * 0.01 = 0.07`. Nearly every negative lookup terminates at the Bloom filter check without touching disk. The dominant cost for a positive lookup is one data block read per level traversed, typically stopping at the first level where the key exists.

> **Takeaway**: Bloom filters make LSM read performance viable by eliminating disk I/O for negative lookups. With 10 bits per key (~1% false positive rate), the Bloom filter for a 1M-entry SSTable is 1.25MB — cheap to keep in memory. Without Bloom filters, negative lookups would require reading from every level.

---

## Compaction: The Background Cost of Sequential Writes

The LSM tree's sequential write advantage comes at a cost: **data accumulates across levels and must be periodically merged** (compacted). Compaction is expensive but happens in the background and doesn't block writes.

### Leveled Compaction

RocksDB's default. When a level exceeds its size limit, pick one SSTable from that level and merge it with all overlapping SSTables in the next level. Output is a new set of non-overlapping SSTables at the next level.

```
Level 1 exceeds limit. SSTable covering keys [300-600] is chosen.

Level 2 SSTables overlapping [300-600]: [201-400] [401-550] [551-700]

Compaction:
  Read: [300-600] from L1 + [201-400][401-550][551-700] from L2
  Merge: sort all entries, discard tombstones for keys with no newer versions
  Write: new SSTables covering the merged key range at L2

Result: L1 loses one SSTable. L2 gets new SSTables (same total data, re-organized).
```

**Write amplification for leveled compaction**:

Each byte written to Level 0 eventually gets compacted through every subsequent level. At each level, it's read once and written once. With `L` levels and size ratio `R`:

```
Write amplification ≈ L * R / (R - 1) ≈ L * R (for large R)

With 7 levels, R=10:
  WA ≈ 7 * 10 / 9 ≈ 7.8  (theoretical per-level contribution)

In practice (accounting for Level 0 overlap and partial compactions):
  WA ≈ 10-30x for typical workloads
```

The benefit: **read performance is predictable**. At most one SSTable per level needs to be checked (plus all Level 0 SSTables). Total SSTables checked per lookup: Level 0 count + (number of levels - 1) ≈ 4 + 6 = 10 Bloom filter checks, 1-2 actual data block reads.

### Tiered (Size-Tiered) Compaction

Cassandra's default. Instead of one SSTable per key range per level, accumulate multiple similarly-sized SSTables per tier. When a tier has enough SSTables (typically 4), merge all of them into one larger SSTable in the next tier.

```
Tier 1 (small SSTables, ~64MB each):
  [SST-1] [SST-2] [SST-3] [SST-4]  ← 4 accumulated, trigger compaction

Merge all 4 into one 256MB SSTable → goes to Tier 2

Tier 2 (medium SSTables, ~256MB each):
  [SST-5] [SST-6] [SST-7] [SST-8]  ← 4 accumulated, trigger compaction
...
```

**Write amplification for tiered compaction**:

```
WA ≈ number of tiers (much lower than leveled)
  Typical: 3-10x vs 10-30x for leveled
```

Lower write amplification means better write throughput. But read performance degrades — multiple SSTables at each tier can have overlapping key ranges (they were flushed independently), so a read might check multiple SSTables per tier.

**Space amplification** is also higher with tiered compaction: before a compaction, you have multiple copies of the same key range sitting in different SSTables at the same tier. With 4 SSTables accumulating before compaction, space amplification is up to 4x.

### The Compaction Tradeoff

```
              Write Amplification    Read Performance    Space Amplification
Leveled:           High (10-30x)        Good (predictable)   Low
Tiered:            Low  (3-10x)         Worse (variable)     High
```

The choice is workload-dependent:
- Write-heavy, time-series, append-mostly → tiered compaction
- Mixed workload with frequent reads → leveled compaction
- RocksDB's default (leveled) is the right choice for most mixed workloads

### Compaction and Tombstone Handling

Tombstones (deletion markers) are only removed during compaction when there are no older versions of the key in any higher level. A tombstone at Level 0 cannot be discarded during L0→L1 compaction if the key might still exist at Level 2 or below — the tombstone must propagate downward until it reaches the level where the original key lives.

This creates a subtle problem: **tombstone accumulation**. If you frequently delete keys and your compaction rate is slower than your delete rate, tombstones accumulate across levels. Every read that passes through levels with tombstones for a key it's looking for must process each tombstone before concluding the key is deleted. In extreme cases (Cassandra workloads that delete heavily), tombstone accumulation causes read latency to spike.

The fix is tuning compaction aggressiveness — lower the size threshold that triggers compaction, or force periodic major compactions that ensure all levels are merged.

> **Takeaway**: Compaction is the background process that maintains LSM tree read performance by merging SSTables and removing obsolete versions and tombstones. Leveled compaction has higher write amplification but predictable reads. Tiered compaction has lower write amplification but higher space usage and more variable reads. Tombstone accumulation is a real operational hazard for delete-heavy workloads.

---

## VACUUM: PostgreSQL's Answer to Dead Tuple Accumulation

MVCC in PostgreSQL means every update and delete leaves a dead tuple in the heap. Without cleanup, tables grow indefinitely and queries slow down as they scan increasing numbers of dead tuples.

**VACUUM** is the process that reclaims dead tuple space. It does not shrink the table file — it marks dead tuple space as reusable and updates the Free Space Map. `VACUUM FULL` rewrites the entire table file to actually return space to the OS, but it requires an exclusive lock and is rarely run in production.

### What VACUUM Does

1. **Scans heap pages**: for each page, identifies tuples where `t_xmax` is committed and the transaction is old enough that no running transaction can see the pre-deletion version.

2. **Removes dead tuple references from indexes**: for each dead tuple, removes its entry from every index on the table. This requires scanning all indexes.

3. **Marks space free in the Free Space Map**: pages with freed space are updated in the FSM so future inserts can reuse that space.

4. **Updates the Visibility Map**: pages where all tuples are now visible to all transactions are marked in the VM, allowing future index-only scans to skip heap fetches.

5. **Advances the frozen transaction ID**: tuples old enough are frozen (xmin replaced with `FrozenTransactionId`), preventing transaction ID wraparound.

### Autovacuum Tuning

PostgreSQL's autovacuum daemon runs VACUUM automatically when tables accumulate enough dead tuples. The trigger is:

```
Autovacuum triggers when:
  dead_tuples > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * table_row_count

Defaults:
  autovacuum_vacuum_threshold = 50        (minimum dead tuples before considering vacuum)
  autovacuum_vacuum_scale_factor = 0.2    (20% of table must be dead)

For a table with 10 million rows:
  Trigger at: 50 + 0.2 * 10,000,000 = 2,000,050 dead tuples
```

For large, write-heavy tables, these defaults are too conservative. A 10-million-row table can accumulate 2 million dead tuples before autovacuum runs — that's 20% table bloat tolerated by default. Production tables with high write rates typically need:

```
autovacuum_vacuum_scale_factor = 0.01   (1% dead tuples triggers vacuum)
autovacuum_vacuum_cost_delay = 2ms      (less throttling, more aggressive)
autovacuum_vacuum_cost_limit = 400      (higher I/O budget per vacuum run)
```

These settings can be applied per-table with `ALTER TABLE ... SET (autovacuum_vacuum_scale_factor = 0.01)`, allowing fine-grained control without changing global settings.

### Table Bloat

Bloat is the percentage of a table's physical size occupied by dead tuples and empty space. Bloat grows when VACUUM can't keep up with dead tuple generation.

```
Bloat impacts:
  1. Table scan time: scanning 100 pages when 30 are dead = 43% wasted I/O
  2. Index size: dead tuples in indexes cause index bloat separately
  3. Cache efficiency: bloated tables evict more useful pages from shared_buffers
  4. VACUUM time: more dead tuples = longer vacuum runs = longer VACUUM cycles
```

Index bloat is independent of table bloat. An index can be 50% bloated even if the table is clean, because index VACUUM (removing dead entries from B-Tree pages) is a separate operation from heap VACUUM. `REINDEX CONCURRENTLY` rebuilds an index from scratch without blocking reads or writes (though it requires more temporary disk space).

### WAL and Durability

Every change to a heap page or index page is first written to the **Write-Ahead Log** before the page itself is modified. The WAL is a sequential append-only file that records the exact changes made to each page. On crash recovery, PostgreSQL replays the WAL forward from the last checkpoint to reconstruct any pages that were modified in memory but not yet written to disk.

WAL writes are sequential. Heap and index page writes are random. This is why WAL has lower latency — no seek overhead. The `fsync` call that makes a WAL write durable flushes the WAL buffer (typically 64MB, set by `wal_buffers`) to disk. A commit is durable once the WAL record for that transaction is fsynced.

`synchronous_commit` controls when the commit acknowledgment is sent to the client:
- `on` (default): wait for WAL fsync before acknowledging commit — full durability
- `remote_apply`: wait for WAL to be applied on standbys — for read-your-writes on replicas
- `off`: acknowledge commit before WAL fsync — up to ~1 second of transactions can be lost on crash, but never corruption

> **Takeaway**: VACUUM reclaims dead tuple space without shrinking the physical file. Autovacuum defaults are too conservative for high-write tables — tune `autovacuum_vacuum_scale_factor` per table. Index bloat is independent of table bloat and requires `REINDEX CONCURRENTLY`. WAL durability is controlled by `synchronous_commit` — `off` risks losing the last ~1 second of commits but never causes corruption.

---

## When Each Architecture Wins

### Page-Based B-Trees (PostgreSQL, MySQL, SQLite)

**Best for**: OLTP workloads with mixed reads and writes, complex queries, transactions with rollback, workloads where the hot dataset fits in the buffer pool.

The in-place update model works when writes are concentrated on a small hot set that stays in memory. Write amplification is high in bytes, but the buffer pool absorbs it — pages modified frequently are written once per checkpoint, not once per modification.

Range queries are efficient: the B+Tree leaf level is a sorted linked list. A range scan traverses the list sequentially after finding the start point. No merging across levels required.

MVCC overhead (dead tuples, VACUUM, bloat) is manageable with proper autovacuum tuning. The tuple header overhead (23 bytes per row) is significant for narrow rows — a table storing (id, value) with 8-byte id and 4-byte value has 23/(23+12) = 66% overhead. TOAST handles wide rows without penalizing narrow-row access patterns.

**Hard ceiling**: random write throughput. Beyond what the buffer pool can absorb, you're doing random disk I/O. On NVMe SSDs this is ~500K-1M IOPS. On network-attached storage (EBS, GCS persistent disk), it's 10x-50x lower. Heavy write workloads hitting this ceiling need either LSM-based storage or significant vertical scaling.

### LSM Trees (RocksDB, Cassandra, LevelDB)

**Best for**: write-heavy workloads, time-series data with monotonically increasing keys, event logs, analytical ingestion pipelines, any workload where write throughput matters more than read latency consistency.

Sequential writes saturate disk bandwidth rather than IOPS. On a 1GB/s NVMe SSD, an LSM tree can sustain ~1 million 1KB writes/second (after factoring write amplification, effectively ~100K-300K logical writes/second). This is 3-10x more than a page-based system hitting its random write ceiling.

Time-series is the ideal workload: keys are timestamps, always increasing. MemTable fills with the newest data, flushes cleanly to Level 0. Level 0 SSTables don't overlap with each other chronologically. Compaction is mostly sequential merge of adjacent time ranges. No hot-spot page contention. Deletes are rare. Tombstone accumulation is not an issue.

**Hard ceiling**: read tail latency during compaction, and point lookup performance for random key distributions. Compaction consumes I/O bandwidth and can cause p99 read latency spikes. On large datasets with high compaction frequency, p999 latency can be orders of magnitude higher than p50. This is why LSM trees are less suitable for latency-sensitive OLTP with strict SLA requirements on tail latency.

> **Takeaway**: Page-based B-Trees win on read predictability, range query efficiency, and transaction semantics. They lose on write throughput when random I/O saturates. LSM trees win on write throughput and time-series workloads. They lose on tail read latency during compaction and random key distribution lookups. The choice should be driven by your write/read ratio, key distribution, and latency requirements — not by which database is newer or more popular.

---

## Key Takeaways

**Pages** are the atomic unit of disk I/O. Modifying one byte means reading and writing the entire page. Page size (8KB, 16KB) is a fundamental tradeoff between sequential scan efficiency and memory overhead.

**Slotted pages** decouple logical row identity from physical location. Indexes reference slot numbers. Deletes mark tuples dead in place. Updates write new versions elsewhere. The Free Space Map tracks available space. The Visibility Map tracks which pages need no visibility checks.

**Tuple headers** (23 bytes in PostgreSQL) carry `xmin`, `xmax`, and `t_ctid` for MVCC. Dead tuples accumulate until VACUUM. Transaction ID wraparound requires periodic VACUUM FREEZE. Monitor `age(datfrozenxid)`.

**TOAST** handles values over 2KB by compressing and storing them out-of-line. Only accessed columns incur TOAST I/O. TOAST tables have independent bloat. `SELECT *` on TOASTed tables silently pays full TOAST I/O cost.

**B+Trees** have height `O(log N)` with high branching factors — 3-4 page reads for billion-row tables. Fill factors trade space for split frequency. HOT updates avoid index writes when no indexed column changes.

**Write amplification** in page-based storage is 100-300x for indexed tables. Buffer pools and WAL batching reduce physical I/O, but random writes are the fundamental ceiling.

**LSM Trees** eliminate random writes by appending everything sequentially. MemTable absorbs writes in memory, flushes to immutable SSTables, compaction merges levels in the background. Writes are always sequential. The cost is read amplification (multiple levels to check) and background compaction I/O.

**SSTables** are sorted, immutable, compressed files. Each block is independently compressed. Binary search on the index block finds the candidate data block. The Bloom filter gate-checks each SSTable before any block read.

**Bloom filters** give probabilistic negative lookups with no false negatives. At 10 bits per key (~1% FPR), they eliminate disk I/O for ~99% of negative lookups across all levels.

**Compaction** merges SSTables, removes tombstones, and maintains level invariants. Leveled compaction: lower write amplification, predictable reads. Tiered compaction: higher write amplification, better write throughput, higher space usage.

**VACUUM** reclaims dead tuple space without shrinking physical files. Autovacuum defaults are too conservative for high-write tables — tune per table. WAL ensures crash recovery; `synchronous_commit = off` trades durability for throughput.