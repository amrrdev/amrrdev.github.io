---
title: "Building an LSM-Tree Storage Engine in Go, Part 2: The Memtable"
date: 2026-05-06
categories: ["Database Internals"]
tags: [lsm-tree, memtable, storage-engine, golang]
---
## Where We Are

In Part 1 we built a Write-Ahead Log. Every mutation — write or delete — is recorded to disk and fsynced before we acknowledge it to the caller. We have durability.

What we do not have is a usable data structure. The WAL is append-only — you cannot query it efficiently. Reading a key from the WAL means scanning every record from the beginning. That is O(n) on the number of writes, which is unusable.

The **memtable** is the in-memory data structure that sits in front of the WAL and makes reads fast. Every write goes to the WAL first (durability), then into the memtable (queryability). Reads check the memtable first — O(log n), not O(n).

The memtable has one additional requirement the WAL does not: it must keep keys in **sorted order**. The reason is flushing. When the memtable gets too large, we write it to disk as an SSTable — a sorted, immutable file. If the memtable is unsorted, flushing it requires a sort pass before writing. If it is always sorted, flushing is a sequential scan from smallest to largest key — no sorting step needed, and sequential writes to disk are the fastest possible I/O pattern.

So the memtable needs to be: sorted, fast for point lookups, fast for inserts, and able to produce a sorted iterator for flushing. The data structure that satisfies all of this is a **skip list**.

---

## Why Not a Binary Search Tree or a Hash Map

Before explaining what a skip list is, it is worth being precise about why the obvious alternatives do not work here.

**Hash map** — O(1) insert and lookup, but no ordering. You cannot iterate keys in sorted order without sorting them first, which is O(n log n) at flush time. Ruled out.

**Binary Search Tree (BST)** — sorted, O(log n) insert and lookup, sorted iteration in O(n). Sounds perfect. The problem is that a naive BST can degenerate to O(n) operations if keys are inserted in sorted or nearly-sorted order — the tree becomes a linked list. The fix is a self-balancing BST (AVL tree or red-black tree), but self-balancing trees have complex rebalancing logic involving rotations that is difficult to implement correctly. RocksDB uses a skip list for its memtable. So does LevelDB. So does Redis for its sorted sets.

**Skip list** — O(log n) insert, lookup, and delete on average, sorted iteration in O(n), and the implementation is significantly simpler than a self-balancing BST. The tradeoff is that the O(log n) guarantee is probabilistic, not worst-case. In practice, the probability of a skip list operation taking more than O(log n) time is astronomically small — small enough that every major storage system that has evaluated this uses skip lists without concern.

---

## What a Skip List Is

A skip list is a linked list with multiple levels of forward pointers. Each level is a faster "express lane" over the level below it.

Start with a sorted linked list at level 0:

```
head → [a] → [c] → [e] → [g] → [i] → nil
```

Finding `i` requires traversing 5 nodes. Now add a level 1 that skips every other node:

```
level 1:  head ————————→ [e] ————————→ [i] → nil
level 0:  head → [a] → [c] → [e] → [g] → [i] → nil
```

To find `i`: start at level 1, jump to `e`, jump to `i`. 2 steps instead of 5. Add a level 2:

```
level 2:  head ————————————————————→ [i] → nil
level 1:  head ————————→ [e] ————————→ [i] → nil
level 0:  head → [a] → [c] → [e] → [g] → [i] → nil
```

Finding `i` from level 2 is now 1 step. With enough levels, search time is O(log n) — each level halves the search space, exactly like binary search.

The key insight: you do not need to maintain these levels manually. When inserting a new node, you flip a coin to decide how many levels it participates in. Heads = add another level, tails = stop. With a fair coin, 50% of nodes appear at level 0 only, 25% at levels 0–1, 12.5% at levels 0–2, and so on. This produces the same "every other node at level 1, every fourth node at level 2" distribution as a perfectly maintained structure — but without any rebalancing.

The probability that a node reaches level k is (1/2)^k. With n nodes and a max level of log₂(n), the expected number of nodes at each level is:

- Level 0: n nodes
- Level 1: n/2 nodes
- Level 2: n/4 nodes
- Level k: n/2^k nodes

Search touches at most O(log n) nodes on average because each level roughly halves the remaining candidates. This is the probabilistic guarantee — not a worst-case bound, but the probability of significantly worse performance shrinks exponentially.

---

## Implementation

### The Node and Skip List

```go
// memtable/skiplist.go

package memtable

import (
	"math/rand"
	"sync"
)

const (
	// maxLevel is the maximum number of levels in the skip list.
	// log₂(65536) = 16, so this handles up to ~65,000 entries efficiently.
	// RocksDB uses 12. Redis uses 32. 16 is reasonable for a memtable
	// that flushes at a few MB.
	maxLevel = 16

	// probability is the coin-flip probability for level promotion.
	// 0.5 gives the classic "halving at each level" distribution.
	// Some implementations use 0.25 (used by Redis) for a shallower,
	// wider structure that is faster in practice due to cache locality.
	probability = 0.5
)

// node is a single element in the skip list.
// forward[i] is the next node at level i.
// forward[0] is the standard linked list — every node appears here.
// forward[k] skips over all nodes that do not participate in level k.
type node struct {
	key     string
	value   []byte
	deleted bool    // tombstone flag — marks a deleted key
	forward []*node // len(forward) == this node's level count
}

// newNode allocates a node with the given level count.
// level is determined at insertion time by randomLevel().
func newNode(key string, value []byte, level int) *node {
	return &node{
		key:     key,
		value:   value,
		forward: make([]*node, level),
	}
}

// randomLevel determines how many levels a new node participates in.
// It flips a biased coin until it gets tails or hits maxLevel.
// This is the core of why skip lists work without explicit rebalancing.
func randomLevel() int {
	level := 1
	for level < maxLevel && rand.Float64() < probability {
		level++
	}
	return level
}

// SkipList is a sorted, concurrent-safe key-value store.
// It supports O(log n) insert, lookup, and delete on average,
// and O(n) sorted iteration.
//
// Concurrency model: single-writer, multiple-reader via sync.RWMutex.
// Writers hold the exclusive lock. Readers hold the shared lock.
// This is correct for a memtable: writes are serialized through the WAL
// anyway, and reads are concurrent.
type SkipList struct {
	mu     sync.RWMutex
	head   *node // sentinel head node — never holds real data
	level  int   // current highest level in use
	count  int   // total nodes including tombstones, used for Iter capacity
	length int   // number of live (non-deleted) keys
	size   int64 // approximate byte size of all live keys and values
}

// NewSkipList returns an empty SkipList.
func NewSkipList() *SkipList {
	// The head node is a sentinel with maxLevel forward pointers,
	// all initially nil. It never holds a real key or value.
	// Search always starts from head.forward[currentLevel-1].
	return &SkipList{
		head:  newNode("", nil, maxLevel),
		level: 1,
	}
}
```

### Insert

The insert algorithm has two phases:

1. **Find the insertion point** — traverse from the highest level down, recording at each level the last node whose key is less than the new key. These are the **update** nodes — after inserting the new node, their forward pointers need to be updated to point to it.

2. **Splice in the new node** — link the new node into each level it participates in by updating the relevant forward pointers.

```go
// Set inserts or updates a key. If the key already exists, its value
// is replaced. Deleted keys (tombstones) are overwritten with the new value.
func (sl *SkipList) Set(key string, value []byte) {
	sl.mu.Lock()
	defer sl.mu.Unlock()

	// update[i] is the rightmost node at level i whose key < key.
	// After inserting the new node, update[i].forward[i] must point to it.
	update := make([]*node, maxLevel)

	current := sl.head

	// Traverse from the highest active level down to level 0.
	// At each level, move right as far as possible without passing the target key.
	// When we can no longer move right, drop down one level.
	// This is the standard skip list search traversal.
	for i := sl.level - 1; i >= 0; i-- {
		for current.forward[i] != nil && current.forward[i].key < key {
			current = current.forward[i]
		}
		update[i] = current
	}

	// Check if the key already exists at level 0.
	existing := update[0].forward[0]
	if existing != nil && existing.key == key {
		if existing.deleted {
			// Resurrect a tombstone — the key is coming back to life.
			// The tombstone had value=nil and was excluded from sl.size,
			// so we only add the new value size, not subtract anything.
			existing.value = value
			existing.deleted = false
			sl.length++
			sl.size += int64(len(key) + len(value))
		} else {
			// Live key — replace value and adjust size by the delta.
			sl.size += int64(len(value)) - int64(len(existing.value))
			existing.value = value
		}
		return
	}

	// New key — determine its level by coin flipping.
	level := randomLevel()

	// If the new node's level exceeds the current max level, initialize
	// the extra levels in update[] to point to head. The head node acts
	// as the left boundary at every level.
	if level > sl.level {
		for i := sl.level; i < level; i++ {
			update[i] = sl.head
		}
		sl.level = level
	}

	n := newNode(key, value, level)

	// Splice the new node into each level it participates in.
	// This is a standard linked list insertion at each level:
	//   new.forward[i] = update[i].forward[i]
	//   update[i].forward[i] = new
	for i := 0; i < level; i++ {
		n.forward[i] = update[i].forward[i]
		update[i].forward[i] = n
	}

	sl.length++
	sl.count++
	sl.size += int64(len(key) + len(value))
}

// Delete marks a key as deleted by setting a tombstone flag.
// The node is not removed from the structure — it remains as a tombstone
// so that during SSTable compaction we know the key was explicitly deleted,
// not merely absent. Physical deletion happens during compaction.
//
// This is how LSM-tree deletes work universally — you cannot simply remove
// a key from the memtable because an older version of that key may exist
// in an SSTable on disk. The tombstone must propagate to disk and survive
// until compaction merges all SSTables that contain the old value.
func (sl *SkipList) Delete(key string) {
	sl.mu.Lock()
	defer sl.mu.Unlock()

	update := make([]*node, maxLevel)
	current := sl.head

	for i := sl.level - 1; i >= 0; i-- {
		for current.forward[i] != nil && current.forward[i].key < key {
			current = current.forward[i]
		}
		update[i] = current
	}

	target := update[0].forward[0]
	if target == nil || target.key != key || target.deleted {
		// Key does not exist or is already a tombstone — nothing to do.
		// We still write a tombstone WAL record at the db.go level,
		// but the memtable has nothing to change.
		return
	}

	target.deleted = true
	sl.length--
	sl.size -= int64(len(target.key) + len(target.value))
	target.value = nil // release memory, tombstone carries no value
}
```

### Lookup

```go
// Get retrieves a key. It returns the value and whether the key exists.
// A tombstoned key returns (nil, false) — from the caller's perspective,
// a deleted key does not exist.
func (sl *SkipList) Get(key string) ([]byte, bool) {
	sl.mu.RLock()
	defer sl.mu.RUnlock()

	current := sl.head

	// Same traversal as insert — move right at each level as far as possible,
	// then drop down. At level 0, check if the next node is the target key.
	for i := sl.level - 1; i >= 0; i-- {
		for current.forward[i] != nil && current.forward[i].key < key {
			current = current.forward[i]
		}
	}

	candidate := current.forward[0]
	if candidate == nil || candidate.key != key || candidate.deleted {
		return nil, false
	}

	return candidate.value, true
}
```

### Sorted Iteration

The sorted iterator is why we chose a skip list over a hash map. Level 0 is a standard sorted linked list — iterating it from head to tail visits every key in ascending order. This is the flush path: call `Iter()` and write each entry sequentially to an SSTable file.

```go
// Entry is a single key-value pair returned by the iterator.
// Deleted entries are included — the SSTable needs to persist tombstones.
type Entry struct {
	Key     string
	Value   []byte
	Deleted bool
}

// Iter returns all entries in sorted key order, including tombstones.
// The caller is responsible for handling tombstones appropriately —
// during flush, tombstones are written to the SSTable as deletion markers.
func (sl *SkipList) Iter() []Entry {
	sl.mu.RLock()
	defer sl.mu.RUnlock()

	// Use sl.count (total nodes including tombstones) not sl.length (live only)
	// as the initial capacity — Iter includes tombstones and the slice
	// must not grow mid-iteration.
	entries := make([]Entry, 0, sl.count)
	current := sl.head.forward[0] // start at the first real node

	for current != nil {
		entries = append(entries, Entry{
			Key:     current.key,
			Value:   current.value,
			Deleted: current.deleted,
		})
		current = current.forward[0]
	}

	return entries
}

// Size returns the approximate byte size of all live keys and values.
// Used by the memtable to decide when to flush to an SSTable.
func (sl *SkipList) Size() int64 {
	sl.mu.RLock()
	defer sl.mu.RUnlock()
	return sl.size
}
```

---

## The Memtable

The memtable wraps the skip list and adds the flush threshold logic. When the skip list exceeds `maxSize` bytes, the memtable signals that it is ready to be flushed to an SSTable.

```go
// memtable/memtable.go

package memtable

const (
	// defaultMaxSize is the memtable flush threshold in bytes.
	// 4MB matches LevelDB's default write_buffer_size.
	// RocksDB defaults to 64MB. Larger memtables mean fewer SSTables
	// (better read performance) but higher memory usage and longer
	// recovery time if the process crashes.
	defaultMaxSize = 4 * 1024 * 1024 // 4MB
)

// Memtable is the in-memory write buffer for the storage engine.
// Writes go here after the WAL. Reads check here before checking SSTables.
// When Size() exceeds MaxSize, the engine flushes this memtable to an SSTable
// and replaces it with a fresh one.
type Memtable struct {
	sl      *SkipList
	MaxSize int64
}

// New returns an empty Memtable with the default flush threshold.
func New() *Memtable {
	return &Memtable{
		sl:      NewSkipList(),
		MaxSize: defaultMaxSize,
	}
}

// Set inserts or updates a key-value pair.
func (m *Memtable) Set(key string, value []byte) {
	m.sl.Set(key, value)
}

// Delete marks a key as deleted.
func (m *Memtable) Delete(key string) {
	m.sl.Delete(key)
}

// Get retrieves a key. Returns (value, true) if found, (nil, false) if not
// found or deleted.
func (m *Memtable) Get(key string) ([]byte, bool) {
	return m.sl.Get(key)
}

// Iter returns all entries in sorted key order, including tombstones.
// Called by the flush path to write the memtable contents to an SSTable.
func (m *Memtable) Iter() []Entry {
	return m.sl.Iter()
}

// Size returns the approximate byte size of the memtable contents.
func (m *Memtable) Size() int64 {
	return m.sl.Size()
}

// ShouldFlush returns true when the memtable has exceeded its size threshold
// and should be flushed to disk as an SSTable.
func (m *Memtable) ShouldFlush() bool {
	return m.sl.Size() >= m.MaxSize
}
```

---

## Wiring the Memtable to the WAL

Now both components exist. Here is how they connect — this is the write path that will eventually live in `db.go`:

```go
// preview of db.go — not the final version, shows the write path clearly

func (db *DB) Set(key string, value []byte) error {
	// Step 1: Write to WAL and fsync.
	// If this fails, we return the error immediately — the memtable
	// is not updated, so nothing is inconsistent.
	if _, err := db.wal.Write(wal.TypeWrite, encodeKV(key, value)); err != nil {
		return err
	}

	// Step 2: Update the memtable.
	// This only runs after the WAL write succeeds. If the process crashes
	// between step 1 and step 2, the WAL record exists on disk and will
	// be replayed into the memtable on recovery.
	db.mem.Set(key, value)

	// Step 3: If the memtable is over its size limit, trigger a flush.
	// The flush writes the memtable to an SSTable file and replaces
	// db.mem with a new empty memtable.
	if db.mem.ShouldFlush() {
		return db.flush()
	}

	return nil
}
```

The ordering — WAL first, memtable second — is the invariant that makes recovery correct. On crash, the WAL replay reconstructs the memtable state exactly as it was before the crash. The memtable is ephemeral; the WAL is the source of truth.

---

## Recovery: Rebuilding the Memtable from the WAL

When the engine starts, it replays the WAL into a fresh memtable:

```go
// preview of db.go — recovery path

func (db *DB) recover() error {
	return db.wal.Recover(func(rec wal.Record) error {
		switch rec.Type {
		case wal.TypeWrite:
			key, value := decodeKV(rec.Payload)
			db.mem.Set(key, value)
		case wal.TypeDelete:
			key := decodeKey(rec.Payload)
			db.mem.Delete(key)
		}
		return nil
	})
}
```

Every WAL record that was fsynced before the crash gets replayed. The memtable is reconstructed to its exact pre-crash state. This is why we write to the WAL before the memtable — the WAL is durable, the memtable is not.

---

## Complexity Summary

| Operation | Average  | Worst Case | Notes                                  |
| --------- | -------- | ---------- | -------------------------------------- |
| Set       | O(log n) | O(n)       | Worst case probability: (1/2)^maxLevel |
| Get       | O(log n) | O(n)       | Same probabilistic bound               |
| Delete    | O(log n) | O(n)       | Tombstone, not physical removal        |
| Iter      | O(n)     | O(n)       | Sequential scan of level 0             |

The worst case for a skip list is O(n) — if every coin flip produces the maximum level, the structure degenerates. With maxLevel = 16 and probability = 0.5, the probability of this happening for any single node is (0.5)^16 = 0.0015%. Over 1 million insertions, the probability that even one node degenerates this severely is still negligible. In practice, skip lists are treated as O(log n) structures.

---

## Design Decisions and Tradeoffs

**Single-writer, multiple-reader mutex vs. lock-free skip list.**
We use `sync.RWMutex` — writers take an exclusive lock, readers take a shared lock. Concurrent reads do not block each other; a write blocks all readers until it completes. For a memtable where writes are serialized through the WAL anyway, this is sufficient. Lock-free skip lists (used in Java's `ConcurrentSkipListMap`) use CAS operations to allow concurrent writers without a mutex. The implementation complexity is significantly higher and the benefit only materializes under very high write concurrency that our WAL serialization already prevents.

**Tombstones instead of physical deletion.**
Deleting a key in an LSM-tree does not remove it from the memtable. It sets a tombstone. The reason: an older version of the key may exist in one or more SSTable files on disk. If we physically delete the key from the memtable, a subsequent read would fall through to the SSTable and find the old value — the delete would appear to have not happened. The tombstone must propagate to disk and survive until compaction merges all SSTables containing the old value and physically removes both the old value and the tombstone. This is one of the most commonly misunderstood aspects of LSM-tree correctness.

**Size-based flush threshold vs. entry-count threshold.**
We flush when `Size()` exceeds `maxSize` bytes, not when the entry count exceeds some limit. Size is the right metric because entry sizes are variable — a memtable with 1,000 entries of 4KB each should flush sooner than one with 1,000,000 entries of 4 bytes each. The size estimate is approximate (it does not account for skip list node overhead), which is fine — the threshold is a heuristic, not a hard limit.

**probability = 0.5 vs. 0.25.**
Redis uses 0.25 for its skip list. At p=0.25, nodes have fewer levels on average, producing a shallower, wider structure. Cache locality improves because you traverse more nodes at low levels (which are close together in memory) and fewer at high levels. At p=0.5, the structure is taller and slightly faster for pure lookup time but worse for cache behavior. For a memtable that holds millions of small keys, 0.25 is often faster in practice. We use 0.5 here for clarity — the probabilistic analysis is cleaner at 0.5.

---

## What's Next

We have a WAL for durability and a memtable for fast in-memory access. The remaining problem: the memtable is bounded. When it fills up, we need to write it to disk in a format that supports efficient lookups — without reading the entire file. That is the **SSTable**: a sorted, immutable file with an index block and a bloom filter.

Post 3 builds the SSTable: binary file format, index block for O(log n) key lookup, bloom filter to avoid reading files that cannot contain a key, and the flush path that drains the memtable into a new SSTable file.

---

The full code is on [GitHub](https://github.com/amrrdev/lsm-engine).
