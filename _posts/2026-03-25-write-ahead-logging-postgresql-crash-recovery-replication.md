---
title: "Write-Ahead Logging: How PostgreSQL Survives Crashes, Powers Replication, and Never Loses Your Data"
date: 2026-03-25
categories: ["Database Internals"]
tags: [wal, postgresql, crash-recovery, replication]
---
## Introduction

Your application writes a row. PostgreSQL says "committed." The server loses power one millisecond later.

When it comes back, is your row there?

The answer is yes — and the reason is Write-Ahead Logging. WAL is the mechanism that makes PostgreSQL's durability guarantee real. It is also the foundation that replication, point-in-time recovery, and crash recovery are all built on top of.

Most developers know WAL exists. Fewer know how it actually works — what gets written, when, in what order, and why that order is the entire point. This article covers all of it.

---

## Topics Covered

- What WAL is and the problem it solves
- The write path — what happens step by step when you commit
- Crash recovery — how PostgreSQL replays WAL after a restart
- Checkpoints — what they are, why they exist, and what happens when they go wrong
- WAL and replication — how standbys consume the WAL stream
- WAL archiving and Point-in-Time Recovery (PITR)

---

## What WAL Is and Why It Exists

PostgreSQL stores table and index data in pages — 8KB chunks on disk. When you update a row, PostgreSQL modifies the relevant page in memory (in the shared buffer pool) and eventually writes it to disk. But "eventually" is the problem.

Writing pages to disk is expensive. Pages are scattered across the disk — different tables, different indexes, all at different physical locations. If PostgreSQL wrote every modified page to disk on every commit, your write throughput would be limited by how fast you can do random I/O. On even a fast NVMe SSD, that ceiling is low.

The naive approach fails in another way too. If the server crashes while modified pages are in memory but not yet on disk, that data is gone. PostgreSQL could fsync every page on every commit, but that makes writes painfully slow. There has to be a better way.

**Write-Ahead Logging is that better way.**

The idea is simple: before modifying anything on disk, write a description of the change to a sequential log. That log is the WAL. Sequential writes are dramatically faster than random writes — you are always appending to the end of a file, no seeking required.

The "write-ahead" in the name is the rule: **the WAL record must reach disk before the data page does.** Always. Without exception. This rule is what makes everything work.

```
The WAL guarantee:

Step 1: Client sends UPDATE
Step 2: PostgreSQL modifies the page in shared buffers (in memory)
Step 3: PostgreSQL writes a WAL record describing the change (sequential, fast)
Step 4: WAL record is fsynced to disk  ← the commit happens here
Step 5: PostgreSQL tells the client: COMMIT successful
Step 6: The modified page is written to disk later (background writer, checkpoint)

If the server crashes between step 4 and step 6:
  The WAL record is on disk. The page is not.
  On restart, PostgreSQL replays the WAL record and reconstructs the page.
  No data lost.
```

The client gets a durability guarantee the moment the WAL record hits disk — not when the data page hits disk. The data page is written lazily in the background. WAL is what makes this safe.

> **Takeaway**: WAL decouples the durability guarantee from the expensive random I/O of writing data pages. Sequential WAL writes are fast. Data page writes happen lazily in the background. The rule — WAL must reach disk before the data page — is what makes crash recovery possible.

---

## The Write Path: What Actually Happens on COMMIT

Let's walk through exactly what PostgreSQL does when you commit a transaction, from the moment the SQL arrives to the moment you get a response.

### Step 1: The Change Happens in Memory

When you run `UPDATE accounts SET balance = 500 WHERE id = 1`, PostgreSQL finds the relevant page in the shared buffer pool (reading it from disk if it is not already cached), modifies the row in memory, and marks the page as **dirty** — meaning it has been changed but not yet written to disk.

The page now exists in two states simultaneously: the old version on disk, the new version in memory. This gap is fine as long as the WAL record is written first.

### Step 2: The WAL Record is Constructed

For every change to a data page, PostgreSQL constructs a **WAL record** — a structured description of what changed. WAL records are not SQL statements. They are low-level descriptions of physical page changes.

A WAL record for an `UPDATE` contains:

```
WAL Record structure:

Header:
  LSN (Log Sequence Number) — unique position in the WAL stream
  Transaction ID (xid)
  Resource Manager ID — which part of PostgreSQL handles this record type
  Record length
  CRC checksum — detects corruption

Body:
  Relation (table OID, tablespace OID, database OID)
  Block number — which page was modified
  Offset within the page
  Old tuple data — for rollback and hot standby conflict detection
  New tuple data — what the page should look like after applying this record
```

The **LSN (Log Sequence Number)** is the key concept. It is a monotonically increasing 64-bit integer representing a byte offset within the WAL stream. Every WAL record has a unique LSN. Every data page stores the LSN of the most recent WAL record that modified it. This is how PostgreSQL knows during recovery which WAL records have already been applied and which still need to be replayed.

### Step 3: The WAL Record is Written to the WAL Buffer

PostgreSQL does not write each WAL record directly to disk — that would be one fsync per record, which is too slow. Instead, WAL records are written to a shared memory region called the **WAL buffer** (`wal_buffers` in PostgreSQL config, default 64MB).

Multiple transactions can be accumulating WAL records in the buffer simultaneously. The buffer absorbs the writes and flushes to disk in batches.

### Step 4: The WAL Buffer is Flushed to Disk (the Commit)

When a transaction commits, PostgreSQL flushes the WAL buffer to disk up to and including the LSN of that transaction's commit record. This flush — an `fsync` call — is what makes the commit durable. Until this flush completes, the commit has not happened as far as durability is concerned.

```
WAL files on disk live in $PGDATA/pg_wal/:

000000010000000000000001   (16MB WAL segment file)
000000010000000000000002
000000010000000000000003
...

Each file is 16MB (wal_segment_size, set at initdb time).
WAL records are appended sequentially within these files.
When one fills up, the next is used.
```

The fsync is the bottleneck in write-heavy workloads. Every synchronous commit waits for the OS to confirm the WAL bytes are physically on storage. This is why `synchronous_commit = off` exists — it skips the fsync wait and instead acknowledges the commit once the WAL is written to the OS buffer (not fsynced). Faster, but the last ~`wal_writer_delay` (200ms by default) of commits can be lost on a crash. The database never corrupts — you just might lose recent transactions.

### Step 5: The Client Gets a Response

Only after the WAL flush succeeds does PostgreSQL return `COMMIT` to the client. The data page may still be in memory, dirty, not yet on disk. That is fine. The WAL record is on disk and that is sufficient for durability.

**PostgreSQL example — observing the WAL write process:**

```sql
-- Current WAL write position
SELECT pg_current_wal_lsn();
-- Returns something like: 0/3A21F8B0

-- How much WAL has been generated since the last checkpoint
SELECT pg_size_pretty(pg_current_wal_lsn() - pg_last_checkpoint_lsn());

-- WAL activity stats
SELECT
    wal_records,
    wal_bytes,
    wal_write_time,
    wal_sync_time
FROM pg_stat_wal;

-- See WAL files currently on disk
SELECT name, size
FROM   pg_ls_waldir()
ORDER  BY modification DESC
LIMIT  10;
```

> **Takeaway**: On commit, PostgreSQL flushes the WAL buffer to disk and returns success. The data page is written lazily later. The LSN on every data page tracks which WAL records have been applied. WAL files are 16MB segments in `pg_wal/`, appended sequentially. `synchronous_commit = off` skips the fsync for speed, risking the last few hundred milliseconds of commits on a crash.

---

## Crash Recovery: Replaying the WAL

The server crashes. Power returns. PostgreSQL starts up. Here is exactly what happens.

### Phase 1: Find the Last Checkpoint

PostgreSQL cannot replay the entire WAL from the beginning — that could be months of history. Instead, it starts from the most recent **checkpoint**. A checkpoint is a point in time where PostgreSQL guarantees that all dirty pages have been written to disk. After a checkpoint, WAL records before that checkpoint's LSN are no longer needed for recovery.

PostgreSQL stores the location of the last completed checkpoint in the **control file** (`$PGDATA/global/pg_control`). This is the first thing read on startup.

```
pg_control contains:
  - Last checkpoint LSN
  - Last checkpoint redo LSN (where to start replaying from)
  - Database state (in production, in crash recovery, shutting down, etc.)
  - PostgreSQL version, page size, WAL segment size
  - Current transaction ID, OID counters
```

If the database state in pg_control is anything other than "shut down cleanly," PostgreSQL knows a crash occurred and enters recovery mode.

### Phase 2: Replay WAL Forward from the Checkpoint

Starting from the checkpoint's redo LSN, PostgreSQL reads WAL records forward and replays each one:

```
Recovery process:

For each WAL record from checkpoint LSN to end of WAL:

  Step 1: Read the WAL record
  Step 2: Check the target data page's current LSN
          → if page LSN >= WAL record LSN: skip (already applied)
          → if page LSN <  WAL record LSN: apply the record
  Step 3: Apply the change described by the WAL record to the page
  Step 4: Update the page's LSN to this WAL record's LSN
  Step 5: Continue to next WAL record
```

The LSN comparison in step 2 makes recovery **idempotent** — you can replay the same WAL record multiple times safely. If a page was written to disk after the WAL record but before the crash, its LSN is already up to date and the record is skipped. No duplicate changes.

This also means the order of operations matters: WAL is always replayed strictly forward, never backward, from the last checkpoint to the end of available WAL.

### Phase 3: Undo Uncommitted Transactions

After replaying all WAL records, the database is in the state it was in at the moment of the crash — including any in-progress transactions that were never committed. PostgreSQL identifies these from the WAL (commit records are present for committed transactions, absent for those that were in flight) and rolls them back using the undo information stored in the WAL records themselves.

After this, the database is clean and ready for connections.

```
Full recovery timeline:

  [Last Checkpoint LSN]────────────────────[Crash Point]
         │                                       │
         └───── Replay all WAL forward ──────────┘
                                                 │
                                    Undo uncommitted transactions
                                                 │
                                    Database ready for connections
```

**PostgreSQL example — inspecting crash recovery:**

```sql
-- After a restart, check when the last recovery happened
SELECT
    pg_postmaster_start_time()          AS server_started,
    pg_conf_load_time()                 AS config_loaded;

-- Check the current WAL position and last checkpoint
SELECT
    pg_current_wal_lsn()                AS current_lsn,
    pg_last_checkpoint_lsn()            AS last_checkpoint_lsn,
    pg_size_pretty(
        pg_current_wal_lsn() -
        pg_last_checkpoint_lsn()
    )                                   AS wal_since_checkpoint;
```

> **Takeaway**: Crash recovery starts from the last checkpoint, replays WAL records forward to the crash point, then rolls back uncommitted transactions. The LSN on each page makes replay idempotent — already-applied records are skipped. Recovery time depends entirely on how much WAL has accumulated since the last checkpoint.

---

## Checkpoints: Bounding Recovery Time

Without checkpoints, PostgreSQL would need to replay WAL from the very beginning on every restart. A database that has been running for months would take hours to recover. Checkpoints bound recovery time by periodically flushing all dirty pages to disk.

### What a Checkpoint Does

A checkpoint is a moment where PostgreSQL guarantees: **everything modified before this point is on disk.** After a successful checkpoint, any WAL records before the checkpoint's LSN can never be needed for recovery — the data pages they describe are already on disk.

The checkpoint process:

```
Step 1: Record the checkpoint LSN in pg_control
Step 2: Write all dirty pages in the shared buffer pool to disk
        (this is the expensive part — potentially thousands of pages)
Step 3: fsync all data files to ensure pages are physically on disk
Step 4: Write a checkpoint WAL record
Step 5: Update pg_control with the new checkpoint location
Step 6: WAL segments older than this checkpoint can be recycled
```

Step 2 is what makes checkpoints expensive. PostgreSQL has to write potentially gigabytes of dirty data pages to disk. If done all at once, this creates a massive I/O spike that degrades query performance while it runs.

### Checkpoint Spreading

PostgreSQL spreads checkpoint I/O over time using `checkpoint_completion_target` (default 0.9). This means PostgreSQL aims to complete the dirty page writes over 90% of the interval between checkpoints, rather than all at once.

```
checkpoint_timeout = 5min (default)
checkpoint_completion_target = 0.9

→ PostgreSQL spreads dirty page writes over 5min * 0.9 = 4.5min
→ Only the final fsync happens abruptly
→ I/O impact is smoothed out over the whole interval
```

### Two Checkpoint Triggers

Checkpoints happen in two situations:

**Time-based**: `checkpoint_timeout` (default 5 minutes). A checkpoint runs every 5 minutes regardless of WAL volume.

**WAL-volume-based**: `max_wal_size` (default 1GB). If WAL has grown by 1GB since the last checkpoint, a checkpoint is triggered immediately — even if only 30 seconds have passed since the last one.

```
Frequent checkpoints (small max_wal_size):
  + Short recovery time (less WAL to replay)
  + Less WAL storage needed
  - Higher I/O from frequent dirty page writes
  - More write amplification

Infrequent checkpoints (large max_wal_size):
  + Lower I/O overhead during normal operation
  + Less write amplification
  - Longer recovery time after a crash
  - More WAL storage needed
```

For most OLTP workloads, the defaults are reasonable. For write-heavy workloads where checkpoints are triggered too frequently (you can see this with `log_checkpoints = on`), increase `max_wal_size`.

**PostgreSQL example — monitoring checkpoints:**

```sql
-- Enable checkpoint logging in postgresql.conf:
-- log_checkpoints = on

-- Check checkpoint frequency and I/O impact
SELECT
    checkpoints_timed,
    checkpoints_req,          -- checkpoints forced by WAL volume (bad if high)
    checkpoint_write_time,
    checkpoint_sync_time,
    buffers_checkpoint,
    buffers_clean,
    buffers_backend           -- pages written directly by backends (very bad)
FROM pg_stat_bgwriter;
```

`checkpoints_req` being high relative to `checkpoints_timed` means WAL is growing faster than the checkpoint interval — increase `max_wal_size`. `buffers_backend` being non-zero means backends are writing dirty pages themselves because the buffer pool is full and the background writer is not keeping up — increase `shared_buffers` or tune `bgwriter` settings.

> **Takeaway**: Checkpoints flush all dirty pages to disk, bounding recovery time to the WAL accumulated since the last checkpoint. They are spread over `checkpoint_completion_target` of the checkpoint interval to avoid I/O spikes. Too-frequent checkpoints (high `checkpoints_req`) indicate `max_wal_size` is too small. Recovery time is directly proportional to how much WAL exists since the last checkpoint.

---

## WAL and Replication: How Standbys Stay in Sync

WAL is not just for crash recovery. It is also the mechanism that powers replication. Every change that happens on the primary is described in WAL. A standby just needs to receive and apply those WAL records to stay in sync.

### Streaming Replication

In streaming replication, the standby connects to the primary and receives WAL records in real time as they are generated — before they are even written to WAL segment files.

```
Primary:
  Transaction commits
       │
       ▼
  WAL record written to WAL buffer
       │
       ├──► flushed to pg_wal/ on primary disk
       │
       └──► streamed to standby via replication connection

Standby:
  WAL receiver process receives the record
       │
       ▼
  Written to pg_wal/ on standby disk
       │
       ▼
  WAL redo process replays the record against standby's data pages
       │
       ▼
  Standby's data pages now reflect the change
```

The standby is essentially doing the same thing as crash recovery — replaying WAL records forward — but continuously, against a live stream of new records.

### Replication Lag

The standby is always some amount behind the primary. How much depends on network latency, the standby's replay speed, and whether you are using synchronous or asynchronous replication.

```sql
-- On the primary: check replication lag for each standby
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
    )                   AS replication_lag_bytes,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;
```

Four LSN positions tell you exactly where the standby is in the pipeline:

```
sent_lsn    — how much WAL the primary has sent to the standby
write_lsn   — how much the standby has written to its pg_wal/
flush_lsn   — how much the standby has fsynced to disk
replay_lsn  — how much the standby has actually applied to its data pages
```

The gap between `pg_current_wal_lsn()` and `replay_lsn` is the total replication lag. A large gap between `sent_lsn` and `write_lsn` indicates network issues. A large gap between `flush_lsn` and `replay_lsn` indicates the standby's replay process is falling behind.

### Synchronous vs Asynchronous Replication

By default, replication is **asynchronous** — the primary commits as soon as its own WAL is flushed, without waiting for the standby to confirm receipt. If the primary crashes before the standby receives the WAL, those transactions are lost on the standby.

**Synchronous replication** (`synchronous_standby_names`) makes the primary wait for at least one standby to confirm it has received and flushed the WAL before returning `COMMIT` to the client. Zero data loss — but every commit now waits for a round trip to the standby.

```sql
-- Configure synchronous replication (in postgresql.conf)
-- synchronous_standby_names = 'standby1'

-- Check if synchronous replication is active
SELECT
    application_name,
    sync_state      -- 'sync', 'async', 'potential', or 'quorum'
FROM pg_stat_replication;
```

`sync_state = 'sync'` means that standby is currently the synchronous standby — the primary is waiting for it on every commit.

### Replication Slots

By default, if a standby falls too far behind, the primary recycles old WAL segments that the standby still needs. When the standby reconnects, it cannot continue — it is missing WAL records it never received.

**Replication slots** prevent this. A slot tracks how far each standby has consumed and prevents the primary from recycling WAL segments that have not yet been consumed.

```sql
-- Create a replication slot for a standby
SELECT pg_create_physical_replication_slot('standby1_slot');

-- Check slot status
SELECT
    slot_name,
    active,
    restart_lsn,
    pg_size_pretty(
        pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ) AS retained_wal
FROM pg_replication_slots;
```

The danger: if a standby goes offline for days and has a replication slot, the primary will retain all WAL generated since the standby disconnected. `retained_wal` keeps growing. The disk fills up. The primary crashes. Always monitor `retained_wal` and set `max_slot_wal_keep_size` to limit how much WAL a slot can retain before it is invalidated.

> **Takeaway**: Standbys are just doing continuous crash recovery — replaying WAL records from a live stream. `pg_stat_replication` shows exactly where each standby is in the pipeline. Async replication is fast but can lose recent transactions on primary failure. Sync replication is zero-loss but adds commit latency. Replication slots prevent WAL recycling but can fill the disk if a standby disconnects — always monitor `retained_wal`.

---

## WAL Archiving and Point-in-Time Recovery (PITR)

Streaming replication protects against hardware failure. But what if someone runs `DROP TABLE accounts` — and it replicates to every standby instantly? Or what if data corruption happened three days ago and you only noticed today?

WAL archiving and Point-in-Time Recovery (PITR) protect against these scenarios.

### WAL Archiving

WAL archiving copies completed WAL segment files to an external location — S3, GCS, a network share — as soon as they are filled. This creates a complete, continuous history of every change ever made to the database.

```sql
-- Enable WAL archiving in postgresql.conf:
-- wal_level = replica          (minimum for archiving)
-- archive_mode = on
-- archive_command = 'aws s3 cp %p s3://my-bucket/wal/%f'
--   %p = full path to the WAL segment file
--   %f = filename only

-- Check archiving status
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

`failed_count` being non-zero means WAL files are not making it to the archive. The primary will keep those files around until archiving succeeds — monitor disk space on `pg_wal/` when archiving is failing.

### Base Backups

WAL archiving alone is not enough for PITR. You also need a **base backup** — a consistent snapshot of the entire data directory at a point in time. Recovery starts from the base backup and replays WAL forward from that point.

```sql
-- Take a base backup (from the primary or a standby)
-- Modern approach using pg_basebackup:
-- pg_basebackup -h localhost -U replicator -D /backup/base -Ft -z -P

-- Or using SQL commands:
SELECT pg_backup_start('my_backup_label');
-- (copy the data directory)
SELECT pg_backup_stop();
```

### Point-in-Time Recovery

With a base backup and a continuous WAL archive, you can restore the database to any point in time — down to a specific second, or even to a specific transaction ID.

```
Recovery scenario: someone ran DROP TABLE at 14:23:15 today.
You want to recover to 14:23:00 — 15 seconds before the mistake.

Step 1: Restore the base backup from last night to a new server
Step 2: Configure recovery target in postgresql.conf:

  restore_command = 'aws s3 cp s3://my-bucket/wal/%f %p'
  recovery_target_time = '2024-01-15 14:23:00'
  recovery_target_action = 'promote'

Step 3: Start PostgreSQL
  → PostgreSQL reads WAL from the archive
  → Replays records forward from the base backup
  → Stops at 14:23:00, before the DROP TABLE
  → Promotes to a standalone database

Step 4: Verify the data is intact
Step 5: Redirect application traffic to the recovered database
```

The `restore_command` tells PostgreSQL how to fetch WAL segments from the archive. PostgreSQL calls this command for each WAL segment it needs, replays the records, and stops when it reaches the `recovery_target_time`.

You can also recover to a specific transaction ID (`recovery_target_xid`) or a named restore point (`recovery_target_name`) that you created with `pg_create_restore_point()`.

```sql
-- Create a named restore point before a risky operation
SELECT pg_create_restore_point('before_migration_v2');

-- If the migration goes wrong, recover to this exact point
-- recovery_target_name = 'before_migration_v2'
```

**Recovery time** depends on two things: how old the base backup is (older = more WAL to replay) and how fast the archive can serve WAL segments. A base backup from last night with 16 hours of WAL might take 30-60 minutes to restore. A base backup from this morning takes 5-10 minutes.

This is why **continuous base backups** matter. Tools like pgBackRest and Barman automate base backups, WAL archiving, and recovery. They also support incremental backups and parallel WAL restore, which dramatically reduces recovery time.

> **Takeaway**: WAL archiving copies every WAL segment to external storage as it is filled. Combined with a base backup, this enables PITR — restoring to any point in time. Recovery replays WAL forward from the base backup and stops at the target time. Named restore points (`pg_create_restore_point`) let you mark safe points before risky operations. Recovery time scales with base backup age and WAL volume.

---

## Key Takeaways

**WAL exists** to decouple the durability guarantee from expensive random I/O. Sequential WAL writes are fast. Data page writes happen lazily in the background. The rule — WAL must reach disk before the data page — is what makes crash recovery possible.

**On every commit**, PostgreSQL flushes the WAL buffer to disk and returns success to the client. The LSN (Log Sequence Number) on every data page tracks which WAL records have already been applied.

**Crash recovery** replays WAL forward from the last checkpoint. LSN comparisons make replay idempotent — already-applied records are skipped. Recovery time is proportional to how much WAL has accumulated since the last checkpoint.

**Checkpoints** flush all dirty pages to disk, bounding recovery time. They are spread over `checkpoint_completion_target` of the interval to avoid I/O spikes. High `checkpoints_req` means WAL is outpacing the checkpoint schedule — increase `max_wal_size`.

**Replication is continuous recovery** — standbys replay WAL records from a live stream. `pg_stat_replication` shows four LSN positions tracking exactly where each standby is in the pipeline. Async replication is fast but can lose recent transactions. Sync replication is zero-loss but adds commit latency.

**Replication slots** prevent WAL recycling for lagging standbys. Always monitor `retained_wal` — a disconnected standby with a slot will eventually fill your disk.

**WAL archiving + base backup = PITR**. You can restore to any point in time by replaying WAL forward from a base backup. Named restore points (`pg_create_restore_point`) let you mark safe recovery targets before risky operations. Recovery time depends on base backup age and WAL volume.

**`synchronous_commit = off`** skips the WAL fsync for speed. The database never corrupts, but the last ~200ms of commits can be lost on a crash. Only use this when you understand and accept that tradeoff.
