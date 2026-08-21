---
title: "Consistency and Data Models in Real Life: The Art of Building Systems That Never Lie to You"
date: 2025-10-17
categories: ["Distributed Systems"]
tags: [consistency, data-models, distributed-systems, cap-theorem]
---
## Introduction

At some point you ship something that works perfectly in staging. The code is clean. The tests pass. You deploy it and feel good about it.

Six months later you're debugging a production issue at 2 AM. A customer transferred money to a friend. The debit processed. The credit didn't. Or both happened twice. The customer is confused because their world no longer makes sense.

The system didn't fail because you wrote bad code. It failed because you were building in a world where servers crash, networks drop packets, clocks drift, and two things can happen "at the same time" on different machines — and you hadn't thought through what that means for your data.

This article is about that world. Not the clean textbook version. The real one.

---

## Topics Covered

- What consistency actually means and why it fractures in distributed systems
- The CAP theorem — what it really says and what it doesn't
- Consistency models: strong, causal, read-your-own-writes, eventual
- Distributed transactions: Two-Phase Commit, how it breaks, and when to use it
- The Saga pattern: a different philosophy for distributed writes
- Data conflicts and vector clocks
- Idempotency and exactly-once semantics
- Network partitions: what happens and how to survive them

---

## What Consistency Actually Means

In a single-node database, consistency is simple. You commit a transaction, and the database moves from one valid state to another. You either see the new state or the old one — never something in between. You can reason about your data without fear.

This works because there is one authoritative copy. One source of truth. One place that decides what's current.

The moment you distribute data across multiple machines — which you must do for redundancy, for scale, for geographic latency — you lose this simplicity. Now you have multiple copies. Each copy can be written to. Each copy can fail independently. Each copy can temporarily disagree with the others.

And suddenly "is this data consistent?" stops having a simple answer.

The distributed systems literature has formalized this into several distinct consistency models. They sit on a spectrum from strongest to weakest, and each one makes a different tradeoff between correctness and performance.

---

## The CAP Theorem: What It Actually Says

CAP is the most misunderstood theorem in distributed systems. Let's be precise.

**CAP states**: a distributed system that stores data can guarantee at most two of these three properties simultaneously:

- **Consistency (C)**: every read returns the most recent write or an error
- **Availability (A)**: every request receives a response (not an error)
- **Partition tolerance (P)**: the system continues operating even when network messages between nodes are dropped or delayed

The critical insight people miss: **partition tolerance is not optional**. Networks fail. Packets get dropped. Data centers lose connectivity. If your system runs on more than one machine, you will experience partitions. You don't get to choose P — you're always in the CP or AP quadrant.

So the real choice is: **when a partition happens, do you sacrifice consistency or availability?**

```
                    Network Partition Occurs
                           │
            ┌──────────────┴──────────────┐
            ▼                             ▼
     Refuse requests                Accept requests
     until partition heals          despite possible stale data
            │                             │
     Consistent (CP)               Available (AP)
     (correct data or error)       (response, possibly stale)
```

**CP systems** (like HBase, Zookeeper, etcd): When a partition happens, they stop accepting writes rather than risk inconsistency. You get an error instead of stale data. Correct, but potentially unavailable.

**AP systems** (like Cassandra, CouchDB, DynamoDB in default config): When a partition happens, they keep accepting reads and writes on both sides. When the partition heals, they reconcile the diverged state. Available, but potentially inconsistent during the partition.

**What CAP doesn't say**: it doesn't say you have to be all-or-nothing. Real systems tune their behavior — Cassandra lets you choose consistency level per-query. DynamoDB has strongly consistent read options. The theorem is a ceiling, not a sentence.

**PACELC — the more useful model**: CAP only describes behavior during partitions. Eric Brewer later introduced PACELC, which adds: even when there's no partition (else), you still trade off **latency (L)** against **consistency (C)**. A system that requires all replicas to confirm before responding is more consistent but slower. This is the tradeoff you make _every day_, not just during failures.

```
PACELC:
  IF Partition:  choose between Availability vs Consistency
  ELSE:          choose between Latency vs Consistency
```

Most systems are PA/EL (Dynamo, Cassandra) or PC/EC (traditional RDBMS). Very few are PC/EL — high consistency with low latency requires significant engineering investment (Google Spanner achieves this with atomic clocks and a global transaction layer).

> **Takeaway**: Partition tolerance is mandatory — you always choose between CP and AP. During normal operation, the tradeoff is latency vs consistency. CAP is a ceiling, not a choice between three things. Real systems let you tune per-operation.

---

## Consistency Models: The Full Spectrum

"Consistent" means different things at different levels. These are the models you'll encounter, from strongest to weakest:

### Linearizability (Strong Consistency)

Linearizability is the strongest model. It guarantees that the system behaves as if there is a single copy of the data, and every operation appears to take effect atomically at a single point in time between its start and completion.

```
Timeline:

Client A:  [Write x=1]────────────────────────────────▶
Client B:          [Read x]──────────▶  must return 1
Client C:                    [Read x]──▶  must return 1
```

Once a write completes, every subsequent read on any node sees that write. No exceptions. No "maybe you'll see the old value."

This is what single-node ACID databases give you. It's also what ZooKeeper, etcd, and Google Spanner give you — at significant cost in latency and availability.

The cost: every write must be acknowledged by a quorum of replicas before returning success. If a replica is slow, your write is slow. If enough replicas are unreachable, your write fails entirely.

### Causal Consistency

Causal consistency is weaker than linearizability but preserves something important: **cause and effect**. If operation A causally precedes operation B (A happened before B, or B was triggered by the result of A), then every node sees A before B.

```
Client A posts a message:     [Write: "Hello"]
Client B replies:             [Write: "Hi there!"] ← causally depends on A's write

Causal consistency guarantees:
  Any node that serves "Hi there!" MUST also show "Hello"
  No one sees a reply before the original message
```

Events that are causally independent — two users posting unrelated messages — can appear in different orders on different nodes. That's fine. The causal chain is what matters.

This maps naturally to human communication. Replies should come after originals. Comments should come after posts. Causal consistency enforces this without requiring global synchronization.

MongoDB's causally consistent sessions and Cassandra's lightweight transactions approximate this model.

### Read-Your-Own-Writes

A weaker but practically critical guarantee: after you write something, you always see your own write in subsequent reads, even if other users might temporarily see the old value.

```
User changes their profile picture:

User's perspective (read-your-own-writes):
  Write new picture ──▶ Read profile ──▶ sees new picture ✓

Other user's perspective (eventual consistency):
  Read profile ──▶ might see old picture temporarily ✓ (acceptable)
```

This prevents the jarring experience of updating something and immediately seeing the old version. It's a subset of causal consistency — your own writes causally precede your own reads.

Implementation: route reads to the same replica you just wrote to, or include a "read-after" token with the write that followers must satisfy before serving the read.

### Monotonic Read Consistency

Guarantees that once you've seen a value, you'll never see an older value. If you read version 5 of a row, your next read returns version 5 or higher — never version 3.

Without this, a client reading from different replicas could observe time going backwards. You see a new post, refresh, and it's gone — because the second read hit a lagging replica.

Solution: pin each client session to a specific replica, or tag reads with a version number that replicas must satisfy.

### Eventual Consistency

The weakest model. If no new writes occur, all replicas will eventually converge to the same value. No guarantees about when. No guarantees about intermediate states.

During active writes, different replicas can hold different values. A read from replica A might return something different than a read from replica B.

```
Write x=1 to replica A:
  Replica A: x=1   ← immediately
  Replica B: x=0   ← for some time
  Replica C: x=0   ← for some time

  ...eventually...

  Replica A: x=1
  Replica B: x=1   ← converged
  Replica C: x=1   ← converged
```

Eventual consistency is appropriate for data where temporary disagreement is acceptable — social media timelines, product view counts, user presence indicators. It is **not** appropriate for account balances, inventory counts, or anything where temporary disagreement causes real-world harm.

> **Takeaway**: Linearizability is the gold standard — one logical copy, operations appear atomic. Causal consistency preserves cause-and-effect order without global coordination. Read-your-own-writes prevents users from seeing their own changes disappear. Eventual consistency is the weakest — replicas converge eventually, but can disagree in the meantime. Match the model to what your data actually needs.

---

## Distributed Transactions: Two-Phase Commit

When an operation spans multiple independent services or databases, you need distributed transactions. The classic protocol is **Two-Phase Commit (2PC)**.

### How 2PC Works

2PC involves a coordinator (the service orchestrating the transaction) and participants (the services doing the actual work).

```
Phase 1 — Prepare:

Coordinator ──▶ "Can you commit?" ──▶ Inventory Service
Coordinator ──▶ "Can you commit?" ──▶ Payment Service
Coordinator ──▶ "Can you commit?" ──▶ Order Service

Each participant:
  - Checks if it can fulfill its part
  - Writes changes to a temporary area (WAL, undo log)
  - Acquires necessary locks
  - Responds YES or NO

Phase 2 — Commit (if all said YES):

Coordinator ──▶ "COMMIT" ──▶ Inventory Service  (releases locks, makes permanent)
Coordinator ──▶ "COMMIT" ──▶ Payment Service    (releases locks, makes permanent)
Coordinator ──▶ "COMMIT" ──▶ Order Service      (releases locks, makes permanent)

Phase 2 — Abort (if any said NO):

Coordinator ──▶ "ROLLBACK" ──▶ all participants  (discard changes, release locks)
```

### Where 2PC Breaks

2PC has a fundamental problem: **the coordinator is a single point of failure during the commit phase**.

**Scenario 1 — Coordinator crashes after prepare, before commit:**

```
Coordinator sends PREPARE to all ──▶ all respond YES
Coordinator crashes ──▶ all participants are now stuck

Participants:
  - Holding locks
  - Cannot commit (haven't received COMMIT)
  - Cannot rollback (might miss a COMMIT when coordinator recovers)
  - Must wait indefinitely
```

This is called the **blocking problem**. Participants are stuck holding locks until the coordinator recovers. Every transaction that touches those rows queues behind the locks. The system degrades.

**Scenario 2 — Network partition during commit:**

```
Coordinator sends COMMIT ──▶ Inventory Service receives it, commits ✓
                         ──▶ Payment Service never receives it (partition)

After partition heals:
  Inventory: decremented ✓
  Payment: not charged ✗

  → Inconsistent state
```

**Scenario 3 — Participant crashes after YES, before COMMIT:**

```
All participants say YES
Coordinator sends COMMIT
Participant A: receives COMMIT, commits ✓
Participant B: crashes before receiving COMMIT

Participant B recovers:
  - It said YES, meaning it promised to commit
  - But it doesn't know if the coordinator sent COMMIT or ABORT
  - Must contact coordinator to find out
  - If coordinator is also down: blocked again
```

### When to Use 2PC

Despite these problems, 2PC is appropriate when:

- All participants are in the same data center (network partition probability is low)
- You need atomicity and the participants support it (PostgreSQL, MySQL both support XA transactions)
- The transaction window is short (locks held for milliseconds, not seconds)

```sql
-- PostgreSQL XA transaction (distributed transaction support)
-- Coordinator prepares the transaction
PREPARE TRANSACTION 'order-txn-12345';

-- Later, coordinator commits or rolls back
COMMIT PREPARED 'order-txn-12345';
-- or
ROLLBACK PREPARED 'order-txn-12345';

-- View all in-doubt prepared transactions
SELECT * FROM pg_prepared_xacts;
-- gid              | prepared            | owner
-- order-txn-12345  | 2024-01-15 14:23:01 | app_user
-- If rows stay here, the coordinator crashed — you have a stuck transaction
```

In-doubt prepared transactions in `pg_prepared_xacts` that are hours old are a sign the coordinator crashed during commit. They hold locks. You need to manually commit or rollback based on what you know about the coordinator's intent.

> **Takeaway**: 2PC guarantees atomicity across multiple participants. Its weakness is the blocking problem — if the coordinator crashes during commit, participants hold locks indefinitely. Use it for short, co-located transactions. Avoid it for long-running cross-service operations spanning unreliable networks.

---

## The Saga Pattern: Distributed Transactions Without Locks

Sagas take a fundamentally different philosophy: instead of holding locks across all participants while coordinating a commit, execute each step independently with its own local transaction. If something fails, run **compensating transactions** to undo what already succeeded.

```
Saga for "Place Order":

Step 1: Create order record          → compensate: delete order
Step 2: Deduct inventory             → compensate: return inventory
Step 3: Charge payment               → compensate: refund payment
Step 4: Send confirmation email      → compensate: send cancellation email

Happy path:
  Step 1 ✓ → Step 2 ✓ → Step 3 ✓ → Step 4 ✓

Failure at Step 3 (payment declined):
  Step 3 ✗ → compensate Step 2 (return inventory) → compensate Step 1 (delete order)
```

No distributed locks. No coordinator holding participants in a suspended state. Each step is atomic locally. The saga is eventually consistent — there are brief moments where inventory is decremented but payment hasn't been charged yet.

### Choreography vs Orchestration

There are two ways to implement sagas:

**Choreography**: each service publishes events and other services react.

```
Order Service publishes: OrderCreated
  → Inventory Service receives it, decrements stock, publishes: InventoryReserved
    → Payment Service receives it, charges card, publishes: PaymentProcessed
      → Notification Service receives it, sends email

On failure:
Payment Service publishes: PaymentFailed
  → Inventory Service receives it, returns stock, publishes: InventoryReleased
    → Order Service receives it, cancels order
```

Choreography is decoupled — services don't know about each other, only about events. But the flow is implicit and hard to visualize. Debugging requires tracing events across multiple services.

**Orchestration**: a central orchestrator explicitly tells each service what to do.

```
Order Orchestrator:
  1. Call Inventory Service: "Reserve item 42"  → success
  2. Call Payment Service: "Charge $99.99"      → failed
  3. Call Inventory Service: "Release item 42"  → success (compensation)
  4. Return failure to client
```

Orchestration is explicit — the entire flow is visible in one place. Easier to debug. But the orchestrator is a central dependency that all services couple to.

Neither is universally better. Choreography scales better and avoids a central bottleneck. Orchestration is easier to reason about and debug. Most teams start with orchestration.

### The Hard Parts of Sagas

**Compensating transactions are not always possible.** You can refund a payment. You cannot un-send an email. You cannot un-notify a user. For truly irreversible operations, you need to delay them until you're confident the saga will succeed — only send the confirmation email after payment clears, not before.

**Sagas are only eventually consistent.** During execution, the system is in an intermediate state. A user's inventory is reserved but payment hasn't been charged yet. Other parts of the system will see this intermediate state. Design your system to tolerate it.

**Compensations can fail too.** If you fail to compensate, you have a partial saga — some steps succeeded, some compensations failed. You need monitoring, alerting, and a dead-letter queue for failed compensations. Manual intervention is sometimes required.

**Idempotency is mandatory.** Because each step can be retried on failure, every step must be idempotent. Charging a payment twice is worse than not charging at all.

> **Takeaway**: Sagas replace distributed locks with compensating transactions. Each step is its own local transaction. On failure, explicitly undo completed steps. Use orchestration for visibility, choreography for decoupling. Compensations must be designed upfront — you can't add them after the fact.

---

## Data Conflicts and Vector Clocks

When two replicas accept writes independently (in an AP system during a partition, or in a multi-leader setup), they can diverge. When the partition heals, they must reconcile. The question is: which version wins?

### Last Write Wins (LWW)

The simplest strategy: each write is tagged with a timestamp. On conflict, the write with the higher timestamp wins.

```
Replica A: x = "alice"  at t=100
Replica B: x = "bob"    at t=101

LWW result: x = "bob"   ← higher timestamp wins
```

Simple. But dangerous. **Clocks in distributed systems are not reliable.** Network Time Protocol (NTP) synchronizes clocks to within ~1ms on a good day, but clock skew of tens or hundreds of milliseconds is common. Two writes that appear to have the same timestamp can be ambiguous. And a write with a slightly earlier timestamp gets silently dropped — data loss with no error.

LWW is used by Cassandra by default. It's acceptable for data where the latest value is always correct (user preferences, last known location) and unacceptable for data where every write matters (counters, financial ledgers).

### Vector Clocks

Vector clocks solve the problem LWW can't: they track causality, not wall-clock time. Each value carries a vector that records how many writes each node has seen.

```
System with 3 nodes: A, B, C
Vector clock format: [A_writes, B_writes, C_writes]

Initial state: x = "v0", clock = [0, 0, 0]

Node A writes x = "v1": clock = [1, 0, 0]  (A has seen 1 write from A)
  Replicated to B and C

Node B writes x = "v2": clock = [1, 1, 0]  (A: 1, B: 1, C: 0)
  (B had seen A's write, then wrote its own)

Node C writes x = "v3": clock = [1, 0, 1]  (A: 1, B: 0, C: 1)
  (C had seen A's write but NOT B's write, then wrote its own)
```

Now we have a conflict: `[1, 1, 0]` and `[1, 0, 1]` are **concurrent** — neither happened before the other. Node B and node C both wrote based on the same ancestor.

```
Conflict detection with vector clocks:

Clock X dominates Clock Y if: every component of X >= every component of Y
  [1, 1, 0] vs [1, 0, 1]: neither dominates → CONFLICT

Clock X dominates Clock Y:
  [2, 1, 0] vs [1, 1, 0]: X dominates → no conflict, X is newer
```

When a conflict is detected, the system has options:

**Option 1: Surface the conflict to the application.** Return both versions and let application logic merge them. This is what DynamoDB's original Dynamo paper described, and what Riak does with "siblings."

**Option 2: Merge automatically.** For certain data types, you can merge without ambiguity. Amazon's shopping cart: merge both carts (take the union of items). This sometimes adds items back that the user deleted, but it never loses items — and losing items from a shopping cart is worse than having extras.

**Option 3: Use a CRDT (Conflict-free Replicated Data Type).** Design your data structure so that concurrent writes can always be merged automatically and correctly. Counters, sets, and registers all have CRDT variants. Redis and Riak support CRDTs natively.

```
CRDT example: G-Counter (grow-only counter)
Each node maintains its own counter. The value is the sum of all nodes' counters.

Node A increments: A's counter = 3
Node B increments: B's counter = 2

Total = 3 + 2 = 5   ← no conflict possible, addition is commutative

Merge: take the max of each node's counter
  A has [3, 1], B has [2, 2]
  Merged: [max(3,2), max(1,2)] = [3, 2] → total = 5 ✓
```

> **Takeaway**: Last Write Wins is simple but causes silent data loss when clocks skew. Vector clocks track causality and detect true concurrent writes. Detected conflicts must be resolved — by the application, by domain-specific merge logic, or by using CRDTs that eliminate conflicts by design.

---

## Network Partitions: What Actually Happens

A network partition is when nodes in your system can't communicate with each other. Not slow communication — absent communication. The east data center can't reach the west data center. A service can receive connections but not respond. A database cluster splits into two halves that can't see each other.

### The Anatomy of a Partition

```
Normal operation:
  Node A ←──────────────────▶ Node B
  (writing, replicating, syncing)

Partition occurs (network link fails):
  Node A ✗━━━━━━━━━━━━━━━━━━━✗ Node B

  Node A: still accepting writes
  Node B: still accepting writes
  Neither knows the other is still running

  ...30 seconds pass...

  Node A state: x=5, y=10, z=3  (from its writes)
  Node B state: x=5, y=7,  z=8  (from its writes)

  Network comes back:
  Node A ←──────────────────▶ Node B

  Conflict: y is 10 on A, 7 on B. z is 3 on A, 8 on B.
  How do you reconcile?
```

### The Split-Brain Problem

In a primary-replica setup, if the primary and replica lose contact, the replica might promote itself to primary (assuming the primary died). Now you have two primaries — both accepting writes. Both think they're the authoritative source. When the partition heals, you have two diverged histories.

```
Before partition:
  Primary (Node A): accepts writes
  Replica (Node B): replicates from A

Partition:
  Node B: "A is dead, I'm promoting myself to primary"
  Node A: still running, still accepting writes as primary

Both accepting writes to the same data:
  Node A writes: user_balance = 500 (deduct 100)
  Node B writes: user_balance = 600 (add 50 to original 650)

After partition heals:
  Which is correct? Unknown.
```

Solutions:

**Fencing tokens**: every primary gets a monotonically increasing token. When a new primary is elected, it gets a higher token. Storage nodes reject writes from primaries with lower tokens. Old primary's writes get rejected — even if it thinks it's still valid.

**Quorum writes**: require writes to succeed on a majority of nodes. With 3 nodes, require 2 to confirm. If a node is partitioned and isolated (only 1 node), it can't get a majority, so it can't accept writes. Split-brain is impossible — a minority can never accept writes.

```
Quorum with 3 nodes (majority = 2):

Scenario: Node C is partitioned off.
  Nodes A and B can still form a quorum → continue accepting writes ✓
  Node C is isolated → cannot accept writes ✗

No split-brain: C can never have a majority, so it's read-only or offline.
```

### What to Do During a Partition

You have three options, each representing a different tradeoff:

**Option 1 (CP): Refuse writes until the partition heals.** Return errors. This preserves consistency — no divergence — but the system is partially unavailable. Correct for financial systems, inventory management, anything where inconsistency causes real harm.

**Option 2 (AP): Continue accepting writes on all sides.** The system stays available but diverges. Reconcile when the partition heals using one of the conflict resolution strategies above. Appropriate for social features, analytics, anything where temporary disagreement is tolerable.

**Option 3 (AP with bounded staleness):** Accept reads but refuse or delay writes above a certain staleness threshold. A middle ground — you stay available for reads, but protect against diverging too far.

> **Takeaway**: Partitions are inevitable. The split-brain problem — two nodes both thinking they're primary — is the most dangerous failure mode. Fencing tokens and quorum writes prevent it. During a partition, you explicitly choose consistency (refuse writes) or availability (accept writes and reconcile later). This choice must be made deliberately, not accidentally.

---

## Idempotency and Exactly-Once Semantics

In a distributed system, **every operation will be retried**. Network requests time out. The client doesn't know if the request succeeded. So it sends again. Your system must handle this correctly.

### The Three Delivery Semantics

**At-most-once**: deliver the message once, don't retry on failure. If it's lost, it's lost. Simple but loses data. Acceptable for metrics, logs, things where losing one data point is fine.

**At-least-once**: retry until you get an acknowledgment. The message will definitely be delivered, but possibly more than once. Your consumers must handle duplicates.

**Exactly-once**: the message is delivered and processed exactly once, even with retries and failures. The hardest to achieve. Requires cooperation between the producer, the broker, and the consumer.

Most systems target at-least-once and make their consumers idempotent. True exactly-once requires either distributed transactions or idempotency keys.

### Idempotency Keys

An operation is idempotent if executing it multiple times has the same effect as executing it once. `DELETE user WHERE id=42` is idempotent — running it twice still results in the user being deleted. `INSERT INTO orders VALUES (...)` is not idempotent — running it twice creates two orders.

The idempotency key pattern makes non-idempotent operations safe to retry:

```
Client generates a unique key for each logical operation:
  key = "order-create-user42-item99-2024011514230001"

Client sends request with key:
  POST /orders
  Idempotency-Key: order-create-user42-item99-2024011514230001
  Body: { user_id: 42, item_id: 99 }

Server behavior:
  1. Check if key exists in idempotency table
  2. If YES: return the stored response (no operation executed)
  3. If NO: execute operation + store (key, response) atomically

Client retries (network timeout):
  Same request, same key
  Server: key found → return stored response
  Client: receives same response as if first attempt succeeded
```

The critical word is **atomically**. The operation and the idempotency key record must be written in the same transaction. If you write the result first and then crash before saving the key, the retry will re-execute. If you save the key first and then crash before executing, the retry will return a "success" response for an operation that never happened.

```sql
-- PostgreSQL: atomic idempotency
CREATE TABLE idempotency_keys (
    key        TEXT PRIMARY KEY,
    response   JSONB        NOT NULL,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ  NOT NULL
);

-- In a single transaction:
BEGIN;

-- Check if we've seen this key
SELECT response FROM idempotency_keys WHERE key = $1;
-- If found: ROLLBACK and return stored response

-- Execute the actual operation
INSERT INTO orders (user_id, item_id, total) VALUES ($2, $3, $4)
RETURNING id INTO order_id;

-- Store the key and response atomically
INSERT INTO idempotency_keys (key, response, expires_at)
VALUES ($1, jsonb_build_object('order_id', order_id, 'status', 'created'),
        now() + interval '24 hours');

COMMIT;
-- If any part fails, the whole transaction rolls back.
-- The key is never stored without the order, and vice versa.
```

### The Confirm-Before-Ack Pattern

In message queues, the order of persist and acknowledge determines your delivery guarantee.

**Wrong order** (ack then persist):

```
Worker:
  1. Pull message from queue
  2. Send ACK to queue ← queue removes message
  3. Process and persist result
  4. [CRASH] ← result lost forever, message already removed
```

**Right order** (persist then ack):

```
Worker:
  1. Pull message from queue
  2. Process and persist result ← durable
  3. Send ACK to queue ← queue removes message
  4. [CRASH] ← message will be re-delivered, idempotency handles duplicate
```

If you crash between step 2 and 3, the message is redelivered. Your idempotency logic detects the duplicate and returns the stored result. No data loss. No double-processing.

This pattern — persist first, acknowledge after — is the foundation of reliable message processing.

> **Takeaway**: Every request will be retried. Make all state-changing operations idempotent using idempotency keys. Store the key and the result atomically — never one without the other. In message queues, always persist before acknowledging. At-least-once delivery + idempotency = effectively exactly-once semantics.

---

## Putting It Together: Designing for the Real World

Here's how these concepts combine in a real scenario. You're building a payment system. A user initiates a transfer.

**Step 1: Idempotency from the start.**
The client generates an idempotency key before sending. Every retry uses the same key. The server is safe to retry without double-charging.

**Step 2: Choose your consistency model deliberately.**
Account balances need linearizability — you cannot allow two concurrent debits to both read the same balance, subtract, and write back. This is a classic read-modify-write race condition.

```
Wrong (eventual consistency on balance):
  Transaction A reads balance: 100
  Transaction B reads balance: 100   ← reads before A writes
  Transaction A writes: 100 - 50 = 50
  Transaction B writes: 100 - 30 = 70  ← overwrites A's write!
  Final balance: 70 (should be 20)

Right (linearizable, with row-level locking):
  Transaction A acquires lock, reads: 100, writes: 50, releases lock
  Transaction B acquires lock, reads: 50, writes: 20, releases lock
  Final balance: 20 ✓
```

**Step 3: Use Sagas for cross-service coordination.**
Charging the source account, crediting the destination account, and notifying the user are three separate operations across potentially three separate services. Use a saga with explicit compensations.

```
Transfer Saga:
  Step 1: Debit source account         compensate: credit back
  Step 2: Credit destination account   compensate: debit back
  Step 3: Record ledger entry          compensate: delete entry
  Step 4: Send notification            (no compensation — accept that emails can't be unsent)
```

**Step 4: Handle the partition.**
Your database cluster spans two data centers. A partition occurs. You choose CP — refuse writes rather than allow divergence on account balances. Return a 503. The client retries when the partition heals. The idempotency key ensures no double charge.

**Step 5: Reconcile.**
Run a nightly reconciliation job that compares the debit ledger against the credit ledger. Any mismatch surfaces a failed saga compensation that needs manual review. This is your safety net — even if everything else works, reconciliation catches the gaps.

This is Stage 3 thinking. Not defensive code that catches errors. Proactive design that anticipates the ways reality will betray your assumptions.

---

## Key Takeaways

**CAP theorem** means you always choose between CP (consistent but potentially unavailable during partitions) and AP (available but potentially inconsistent). Partition tolerance is not optional. PACELC extends this: even without partitions, you trade latency for consistency on every operation.

**Consistency models** range from linearizability (single logical copy, operations appear atomic) to eventual consistency (replicas converge eventually, can disagree in the meantime). Match the model to what your data actually requires. Account balances need linearizability. Social feeds are fine with eventual consistency.

**2PC** guarantees atomic distributed commits but blocks indefinitely if the coordinator crashes during the commit phase. Use it for short, co-located transactions. Avoid it for long-running cross-service flows.

**Sagas** replace distributed locks with compensating transactions. No cross-service locks — each step commits locally. Failure triggers explicit compensations. Design compensations upfront — you cannot retrofit them. Accept that the system is eventually consistent during saga execution.

**Vector clocks** detect true concurrent writes that wall-clock timestamps miss. Concurrent writes are conflicts — resolve them with application logic, domain-specific merge, or CRDTs. Last Write Wins is simple but silently drops data when clocks skew.

**Network partitions** are inevitable. Split-brain — two nodes both accepting writes as primary — is the most dangerous outcome. Prevent it with quorum writes or fencing tokens. During a partition, explicitly choose to refuse writes (CP) or accept divergence (AP). This is a design decision, not an accident.

**Idempotency** is the foundation of reliable distributed systems. Every state-changing operation needs an idempotency key. Store the key and the result in the same transaction — atomically. In queues, persist before acknowledging. At-least-once delivery plus idempotency gives you effectively exactly-once semantics.

**Design for failure deliberately.** Think through failure scenarios during design, not during the 2 AM incident. Simulate partitions. Test compensations. Run reconciliation jobs. The systems that survive production are the ones where someone asked "what happens when this fails?" before shipping.
