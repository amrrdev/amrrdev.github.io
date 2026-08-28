---
mermaid: true
title: "Filesystem Locks and Crash Consistency"
date: 2026-08-28
categories: ["System Engineering"]
tags: [file-locks, advisory-lock, flock, fcntl, ofd-locks, atomic-rename, journaling, crash-consistency, fsck, wal, renameat2]
series: "System Engineering"
stage: "Stage 7 - Filesystems, Devices, and Storage I/O"
stage_order: 7
series_order: 6
---

The previous article covered how to make a write reach the device. This chapter covers how to keep a filesystem coherent while it is being changed, both by concurrent writers and by a power loss halfway through. It is the sixth article of Stage 7, completing the subject on file I/O and correctness.

Two separate problems live here. The first is concurrency: when several processes write the same file, their writes can interleave into garbage unless coordinated. The second is crash consistency: if the machine dies between two related writes, the on-disk state must not be left as a corrupt half-update. Both are solved by long-standing, simple patterns, and both are easy to get wrong. This article is a reference covering the lock primitives in full, the atomic-update patterns modern kernels provide, how journaling and copy-on-write filesystems keep structures consistent, the device-level flush contract, and the application-level techniques such as write-ahead logs that keep your own data coherent.

## Advisory versus mandatory locking

The common file locks on Linux, `flock` and `fcntl` record locks, are advisory. They work only when cooperating processes choose to take them. A process that ignores the lock and writes anyway is not stopped by the kernel; it simply races with everyone else. This sounds weak, but in practice most systems are built from cooperating components that all respect the same lock, so advisory locking is enough.

Mandatory locking, where the kernel enforces exclusion even on processes that do not cooperate, exists but is fragile and largely deprecated on Linux. It can be enabled per filesystem and has surprising interactions, so engineers rely on advisory locks by convention rather than mandatory enforcement. The discipline is in the code, not the kernel.

```mermaid
flowchart LR
    A[Write new content to temp file] --> B[fsync the temp file]
    B --> C[rename temp over target]
    C --> D[Readers see an atomic swap]
```

The diagram shows the atomic-replace pattern, which is the single most useful trick in this chapter and does not need any lock at all.

## flock and fcntl record locks

`flock` locks an entire open file: one process holds the lock, others block or fail when they try. It is simple and well suited to "only one writer at a time" jobs such as rotating a log or running a single cron task. `fcntl` locks are finer: they can lock a byte range of a file, so different processes can work on different regions concurrently. They also associate the lock with the open file description, which has subtle effects across `fork` and `dup` that `flock` avoids.

For a service that appends to a shared log, an `flock` on the log file before writing prevents two workers from interleaving records. For a file with independent sections, a range lock lets several processes edit different parts at once. The choice is about granularity: whole-file simplicity versus concurrency.

There is an important historical wrinkle in `fcntl` (POSIX) locks. They are associated with the process and the file inode, not with a single open file description. Two separate `open` calls in the same process point at the same inode, so their locks conflict with each other, and worse, closing any file descriptor to that inode releases all of the process's POSIX locks on it. This surprises programs that open a file twice expecting independent locks, or that close one handle and unintentionally drop a lock held through another. Linux added open file description locks via `fcntl` commands `F_OFD_SETLK`, `F_OFD_SETLKW`, and `F_OFD_GETLK`, which behave like a sane version: the lock follows the open file description and is released only when that description is closed, so `dup` and `fork` share it as you would expect and closing one fd does not drop another's lock. New code that needs range locking should prefer OFD locks.

| Property | `flock` | POSIX `fcntl` lock | OFD `fcntl` lock |
|---|---|---|---|
| Granularity | whole file | byte range | byte range |
| Owned by | open file description | process + inode | open file description |
| Close of another fd to same file | keeps lock | drops all locks on that inode | keeps lock on its own description |
| `fork` | child shares | child inherits then independent | child shares the description |
| Deadlock detection | no | yes, returns `EDEADLK` | yes, returns `EDEADLK` |

## Lock types, deadlock detection, and leases

Both `fcntl` and OFD locks support shared (`F_RDLCK`) and exclusive (`F_WRLCK`) modes, and a query form (`F_GETLK`) that reports whether a conflicting lock exists. The kernel detects deadlock when two processes each hold a lock the other wants and would block forever, returning `EDEADLK` to one of them rather than hanging. `flock` does not detect deadlock; it simply blocks, which is another reason range locks are preferred for complex locking graphs.

A distinct primitive is the file lease, set with `fcntl` `F_SETLEASE`, which lets a process be notified when another process tries to open or truncate a file it is caching, so it can flush or invalidate its state first. Leases are used by Samba and by local caches that must not hold stale data when a remote writer arrives. They are not mutual exclusion in the usual sense but a coherence signal, and they are revoked by the kernel if the leasing process does not respond in time.

## The atomic rename pattern and its modern forms

The most reliable way to update a file that readers may be reading is to write the new content to a temporary file, `fsync` it, then `rename` it over the target. `rename` is atomic at the directory level: a reader either sees the old file or the new one, never a half-written one. This is how package managers, editors, and config systems replace files safely. The temp file must be on the same filesystem as the target, because `rename` is atomic only within one filesystem and across filesystems it becomes a copy that is not atomic.

The `fsync` before the `rename` is the part people skip, and it matters for crash consistency. If you `rename` a temp file whose data is still in the page cache, a crash before writeback can leave the renamed file pointing at blocks that were never written, producing a zero-length or garbage file. Flushing the temp first ensures the bytes are on the device before the name switches to them. This pattern also avoids in-place corruption: you never overwrite the live file, you replace its name.

Modern kernels extend this. `renameat2` supports `RENAME_NOREPLACE`, which renames only if the target does not already exist, atomically (like a link-if-absent), and `RENAME_EXCHANGE`, which atomically swaps two existing paths so that neither ever disappears. `RENAME_EXCHANGE` removes the brief window where the temp exists and the target is being replaced, at the cost of leaving the old version under the temp's name to clean up. `O_TMPFILE` opens an unnamed temporary file in the directory, so there is no visible name to leak and the final step is a `linkat` to give it a name, which is the cleanest form of the temp-plus-rename pattern.

## Journaling and write ordering

A filesystem metadata update, such as creating a file or appending to one, touches several on-disk structures: the inode, the directory entry, the block bitmap, and the data blocks. If the machine crashes between two of those writes, the structures can disagree, leaving a file whose size says it is large but whose blocks are unallocated, or a directory entry pointing at a freed inode. That is metadata corruption.

Journaling fixes this by writing the intended changes into a reserved journal region first, then applying them to the real structures, and marking the journal entry complete only after the real writes are done. If a crash happens, the filesystem replays the journal to finish incomplete updates or discards them, so the on-disk state is always consistent. The classic ext3/ext4 modes are `data=ordered` (metadata journaled, data written before its metadata commit, the default and a good balance), `data=writeback` (metadata journaled, data ordering not enforced, faster but can expose old data after a crash), and `data=journal` (both data and metadata journaled, slowest but strongest). A backend engineer reads a mount's `data=` option to know which crash-time guarantees it actually has.

```mermaid
flowchart LR
    A[Record change in journal] --> B[Apply change to real structures]
    B --> C[Mark journal entry committed]
```

Copy-on-write filesystems such as btrfs and ZFS take a different approach. Instead of updating structures in place and journaling the intent, they write new versions of changed blocks to free space and atomically switch the pointer to the new tree once the write is complete. A crash leaves the old tree intact, so consistency is a property of the structure rather than of a replayed log. ZFS and btrfs also checksum every block and can detect, and on redundant storage repair, silent corruption, which journaling alone does not do. The tradeoff is more write amplification and more complex code, but the consistency story is stronger and does not depend on a separate journal.

## Write barriers and device reordering

Even with journaling, the kernel must ensure the device does not reorder writes in a way that defeats the log. A disk or its controller may reorder writes for performance, and a volatile device cache may delay them. Write barriers, and the flush commands behind them, force the device to complete certain writes before others, so the journal commit is truly on disk before the metadata it protects is considered durable. The mechanism in modern kernels is expressed as block-layer requests `REQ_PREFLUSH` (flush the device cache first), `REQ_FUA` (force unit access, write through the cache and acknowledge only after it is stable), and `REQ_OP_FLUSH` for a standalone flush.

This is the same flush contract discussed in the durability article. Crash consistency depends on it: without barriers, the journal's guarantee can be undone by a device that writes the metadata before the journal record. The lesson across both articles is that the kernel's promise rests on the device honoring flushes.

## Power-loss behavior, fsck, and self-healing

Without journaling, a crash can leave the filesystem structurally inconsistent, and a check utility such as `fsck` must scan and repair it, which is slow and sometimes lossy. With journaling, the normal case needs no `fsck` at all; the log is replayed at mount (a journal-aware `e2fsck` still runs but mostly just replays). Modern filesystems still offer `fsck` for rare corruption from hardware errors, but the day-to-day crash recovery is automatic.

On copy-on-write filesystems the recovery is different: because the old tree remains valid, a mount after a crash simply uses the last consistent tree, and a scrub pass verifies and repairs checksums against redundant copies. The application's own consistency, however, is not solved by the filesystem. Journaling protects the filesystem's structures, not your data's meaning. If your design required two files to be updated together, the filesystem will not keep them in sync; you must use atomic rename, a transaction, or an intent log yourself. Crash consistency at the filesystem level prevents corruption; application consistency is a separate, higher-layer concern.

## Application-level consistency: WAL, intent logs, and double-write

When correctness requires that several changes happen together, or that a half-applied update is never visible, applications build on the primitives above. A write-ahead log records the intended change to a separate, sequentially appended, `fsync`'d log before applying it to the main data, so that after a crash the log can be replayed to finish or undone to roll back. Databases and key-value stores are built on this. An intent log is a lighter form: a record of "I am about to do X" that is cleared once X completes, so a crash mid-X is detectable and recoverable.

```mermaid
flowchart LR
    A[Write intent to WAL and fsync] --> B[Apply change to data files]
    B --> C[Mark intent done and fsync]
    C --> D[Crash before C is replayed from WAL]
```

The diagram is the shape of a write-ahead log. The intent is durable before the change, so a crash during the change can be completed from the log, and a crash before the intent leaves the data untouched. A double-write buffer is a related technique, used by storage engines such as InnoDB, where each page is written twice (to a staging region and then to its real location) so a torn page from a crash mid-write can be detected and rebuilt. For updates spanning two files, the practical answer is usually: write the new combined state to a temp file, `fsync`, `rename` (or `RENAME_EXCHANGE`) the relevant names, and `fsync` the directory, turning a multi-object update into a sequence of individually atomic steps.

Preallocating space with `fallocate` before writing is a related correctness aid: it guarantees the blocks exist so a later write cannot fail with `ENOSPC` partway through, which would otherwise leave a half-written, unrecoverable file. Combined with the atomic-rename pattern, it removes two common crash and capacity failure modes at once.

## Observing locks and journaling

Held locks are visible in `/proc/locks` and via `lslocks`. The journal and mount options are visible through filesystem-specific tools and `/proc/mounts`. Seeing what is locked, and by which process, is the first step when two writers appear to stomp on each other. `fuser` and `lsof` answer "which process has this file open," which often explains a lock that will not release.

```bash
# who holds which locks
lslocks | head
cat /proc/locks | head
# mount options including journal mode
mount | grep -E "ext4|xfs|btrfs"
grep -E "data=" /proc/mounts
# does a writer hold a flock while it works
flock -n /var/log/app.log -c 'echo acquired; sleep 5' &
sleep 0.5
lslocks | grep app.log
# which process has a file open
fuser -v /var/log/app.log
lsof /var/log/app.log
# extent layout and fragmentation
filefrag -v somefile.dat
```

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    f, err := os.CreateTemp("", "safe-*.txt")
    if err != nil {
        panic(err)
    }
    tmp := f.Name()
    f.WriteString("new safe content\n")
    f.Sync() // flush before the name switches
    f.Close()

    // atomically replace the target
    if err := os.Rename(tmp, "config.txt"); err != nil {
        panic(err)
    }
    fmt.Println("config.txt replaced atomically via temp + fsync + rename")
    select {}
}
```

What it shows is the safe-update pattern in code: write to a temp, sync it, then rename over the target. Any reader of `config.txt` sees either the old or the new file, never a partial one, and a crash before the rename leaves only the temp, which is harmless to clean up. The `lslocks` and `flock` commands show real locks held by processes, which is how you diagnose a concurrency bug, and `/proc/mounts` reveals the journal mode that decides crash-time behavior.

## A realistic production example

A team maintained a configuration file that a control loop rewrote in place several times per minute by opening it, seeking to the start, and writing the new contents. Most of the time this was fine, but occasionally a deploy or a crash happened during the write, leaving the file half old and half new, or truncated to a smaller size. On the next start, the service failed to parse its own config and refused to boot, which is the worst kind of failure: not a crash but a self-inflicted unstartable state.

The fix was the atomic pattern: write the new config to a temp file in the same directory, `fsync` it, then `rename` it over the live path. The directory must be the same so the rename is on one filesystem and therefore atomic. They also added an `flock` around the update so two control-loop iterations could not race each other, even though the rename alone prevents readers from seeing partial data. For a later two-file update, they adopted `RENAME_EXCHANGE` so that the config and its hash file swapped together with no window where one was new and the other old, and they preallocated the new file with `fallocate` so a full disk could not abort the write partway. After the change, a crash during an update at worst left a stray temp file, and the live config was always a complete, previously valid version. The lesson was that in-place writes are a corruption waiting for a crash, and atomic rename plus fsync turns an unsafe update into a safe swap.

## How engineers actually reason about consistency

They never update a live file in place. The temp-plus-rename pattern is the default for any file another process may read, because it makes the swap atomic and crash-safe when combined with a pre-rename `fsync`. When two names must change together, they use `RENAME_EXCHANGE` or a write-ahead log.

They treat locks as coordination, not enforcement. Advisory locks work because every writer respects them; the job is to make sure all writers actually do, to pick the right granularity with `flock` or OFD range locks, and to remember that closing one descriptor can drop a process's POSIX locks on a file.

They separate filesystem consistency from application consistency. Journaling keeps the filesystem valid across crashes, but it will not keep two of your files coherent. That part is your design, using atomic rename, transactions, intent logs, or a WAL.

They remember the flush contract. Atomic rename is only crash-safe if the temp's data reached the device before the rename, which is the `fsync` step, and that step depends on the device honoring flushes as the durability article described.

## Network and distributed filesystems, and application crash patterns

The locking and consistency rules change when the filesystem is not local. NFS and other network filesystems must represent locks and opens across the wire. The server holds the authoritative lock state, and clients use leases or delegations that the server can recall, which is why a long-held advisory lock on NFS can be revoked if the client goes silent. FUSE filesystems implement the operations in a user-space daemon, so their consistency is whatever the daemon enforces, and they add a context-switch and copy cost on every operation. For these, the atomic-rename pattern is still the safe update method, but the fsync-before-rename durability depends on the server's own flush contract, not just the local kernel.

At the application layer, the write-ahead log pattern from this article is the workhorse for crash consistency across multiple files. A minimal sketch: before mutating the real data, append a record describing the change to a sequentially written, fsync'd log. Apply the change to the data files, then mark the log record committed, also fsync'd. After a crash, replay the log from the last committed point, redoing any change whose commit was not recorded. This turns a multi-step update into an ordered, recoverable sequence without needing the filesystem to understand the data's meaning. Databases, key-value stores, and some queue systems are built on exactly this, often with a separate write-ahead device, a fast NVMe, so the fsync'd commits do not contend with bulk writes.

## Definitions

### Advisory locking

> A locking scheme, such as `flock`, POSIX `fcntl`, and OFD `fcntl` locks, where the kernel does not block processes that ignore the lock. Correctness depends on all cooperating writers choosing to take the lock, with OFD locks fixing the process-associated surprises of POSIX locks.

### Atomic rename and its variants

> The pattern of writing new content to a temporary file, flushing it, and renaming it over the target, so readers see either the old or new file atomically. `renameat2` adds `RENAME_NOREPLACE` (atomic link-if-absent) and `RENAME_EXCHANGE` (atomic swap of two existing paths), and `O_TMPFILE` provides an unnamed temp.

### Journaling and copy-on-write

> Journaling records intended changes in a journal, applies them, and commits only after, so a crash replays or discards incomplete updates. Copy-on-write filesystems write new versions of blocks to free space and atomically switch pointers, leaving the old tree consistent and adding checksums for detection and repair.

### A write barrier

> A request to the device to complete prior writes before later ones, enforced through flush commands such as `REQ_PREFLUSH` and `REQ_FUA`, so that journal commits are truly durable and not reordered away by the device.

### Crash and application consistency

> Crash consistency is the property that the filesystem survives a power loss without corrupt structures, achieved by journaling or copy-on-write. Application consistency is the higher-layer property that your multi-file or multi-step updates are coherent, which you must build with atomic updates, intent logs, or a write-ahead log.

## Beyond the definitions

### Why is advisory locking enough if the kernel does not enforce it

> Because the systems that share a file are usually cooperating components that all use the same lock. Enforcement is a property of the code agreeing to coordinate, which is simpler and faster than kernel-mandated exclusion that has its own sharp edges, and OFD locks make the coordination sane.

### Why fsync the temp before renaming it

> Because rename only swaps names; if the temp's data is still in the page cache, a crash can leave the renamed file referencing unwritten blocks. Flushing first ensures the bytes are on the device before the name points at them.

### How is journaling different from application transactions

> Journaling protects the filesystem's own structures from corruption across a crash. It does not understand your data's meaning, so if your design needs two files updated together, you must provide that transaction, intent log, or WAL yourself.

### What do write barriers protect against

> A device that reorders or delays writes can otherwise write metadata before the journal record that should precede it, defeating the log. Barriers force ordering so the journal's guarantee holds even with a reordering disk or a volatile cache.

### When would a rename still leave a problem

> If the temp and target are on different filesystems, rename is not atomic and may copy. If you forget the pre-rename `fsync`, a crash can leave the renamed file empty. And if two names must change together, a single rename leaves a window that `RENAME_EXCHANGE` or a WAL must close.

### Why did closing a file drop my lock

> Because POSIX `fcntl` locks are associated with the process and the inode, so closing any descriptor to that inode releases all the process's POSIX locks on it. OFD locks, tied to the open file description rather than the inode, do not have this surprise; prefer them for new code.

## Common misconceptions

**"flock stops other processes from writing."** It stops only processes that also take the lock. A writer that ignores it and opens the file directly races freely, because the lock is advisory. Coordination is a code convention, not kernel enforcement.

**"Atomic rename means the data is durable."** Rename is atomic in visibility, not in durability. Without a `fsync` of the temp first, a crash can leave the new file's blocks unwritten. Atomic and durable are two different properties.

**"The filesystem journals, so my files are safe."** Journaling keeps the filesystem consistent, not your multi-file logic. It will not keep a database and its index in step; that is your responsibility through transactions, a WAL, or atomic patterns.

**"In-place rewrite is fine if I rewrite fast."** A crash or kill at any moment during the rewrite leaves a partial or truncated file, and the next reader may be the service itself failing to start. Atomic rename removes the window entirely rather than shrinking it.

**"Mandatory locking is stronger, so use it."** Mandatory locking on Linux is fragile and largely deprecated, with surprising interactions. Advisory locking by convention, and OFD locks for sane range locking, are the practical, reliable choices.

**"Closing one fd only affects that fd's lock."** For POSIX `fcntl` locks it affects the whole process's locks on that inode, because they are keyed by process and inode. This is the classic trap; OFD locks key by open file description and avoid it.

## Summary

File consistency has two faces. Concurrency is handled by cooperation through `flock`, POSIX `fcntl` locks, or the newer OFD locks, which are advisory and rely on every writer respecting them, with OFD locks fixing the process-and-inode association surprises. Crash consistency is handled by the atomic rename pattern for applications, extended by `renameat2` `RENAME_EXCHANGE` and `O_TMPFILE`, and by journaling or copy-on-write for the filesystem itself. The rename pattern, write temp, flush, then swap (or exchange), is the workhorse that keeps a live file from ever showing partial content, and the flush before the swap is what makes it survive a crash. The filesystem's journal keeps its own structures valid but does not make your multi-file updates coherent; that remains your design, built from atomic updates, intent logs, and write-ahead logs. The next subject of this stage leaves software and looks at the hardware underneath: disks, SSDs, NVMe, and RAID.
