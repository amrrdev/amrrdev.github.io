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

The last article showed how to push a write down to the device. This article shows how to keep the filesystem correct while it changes. Two risks can break it. Other programs can write at the same time. The machine can lose power in the middle of a change. This is the sixth article in Stage 7. It closes the topic of file I/O and correctness.

Two problems appear here. The first is concurrency. Concurrency means several programs act on the same file at the same time. Without coordination their writes can mix into garbage. The second is crash consistency. Crash consistency means the files stay valid after a sudden power loss. Suppose the machine dies between two writes that belong together. The file on disk must not stay as a broken half update. Both problems have simple and old fixes. Both are easy to get wrong. This article is a reference. It covers lock tools, atomic update patterns that modern kernels provide, how journaling and copy on write filesystems stay valid, how the device flush contract works, and how app tools like write ahead logs keep your data correct.

## Advisory versus mandatory locking

The common file locks on Linux are `flock` and `fcntl` record locks. Both are advisory. Advisory means voluntary. The lock works only if every program that shares the file agrees to use it. Suppose one program ignores the lock and writes anyway. The kernel does not stop it. That program simply races with the others. This sounds weak. In practice most systems are built from parts that all respect the same lock. For those systems advisory locking is enough.

Mandatory locking is different. Mandatory means the kernel enforces the lock. The kernel blocks even programs that do not cooperate. Linux does provide it. But it is fragile and mostly deprecated. You can turn it on per filesystem. But it has odd side effects. So engineers use advisory locks by agreement instead. The rule lives in your code, not in the kernel.

```mermaid
flowchart LR
    A[Write new content to temp file] --> B[fsync the temp file]
    B --> C[rename temp over target]
    C --> D[Readers see an atomic swap]
```

The diagram shows the atomic replace pattern. This is the most useful trick in this article. It needs no lock at all.

## flock and fcntl record locks

`flock` locks a whole open file. One process holds it at a time. Other processes block or get an error when they try. It is simple. It fits jobs that need one writer at a time. Examples are rotating a log or running a single cron task. `fcntl` locks are finer. Finer means they can lock a byte range in a file. Different processes can then edit different parts at the same time. `fcntl` ties the lock to the open file description. An open file description is the kernel object that tracks an open file. That tie has subtle effects across `fork` and `dup`. `fork` creates a child process. `dup` copies a file descriptor. `flock` avoids these subtleties.

Suppose your service appends to a shared log. Your service takes an `flock` on the log before each write. Then two workers cannot mix their records. Suppose a file has separate sections. A range lock lets several processes edit different sections at the same time. The choice is about granularity. Granularity means how much of the file you lock. You can pick whole file simplicity. Or you can pick more concurrency with range locks.

POSIX `fcntl` locks have a historic trap. They attach to the process and the inode. An inode is the filesystem object that stores a file's metadata and block list. The lock does not attach to a single open file description. Suppose a process opens the same file twice. Both opens point to the same inode. So their locks conflict. Worse, closing any one handle drops all POSIX locks that the process holds on that inode. This surprises programs that open a file twice and expect separate locks. It also surprises programs that close one handle and lose a lock held through another handle. Linux added open file description locks to fix this. You use them with `fcntl` commands `F_OFD_SETLK`, `F_OFD_SETLKW`, and `F_OFD_GETLK`. They act in a sane way. The lock follows the open file description. The kernel releases it only when that description closes. `dup` and `fork` share it as you expect. Closing one file descriptor does not drop another descriptor's lock. New code that needs range locking should use OFD locks.

| Property | `flock` | POSIX `fcntl` lock | OFD `fcntl` lock |
|---|---|---|---|
| Granularity | whole file | byte range | byte range |
| Owned by | open file description | process + inode | open file description |
| Close of another fd to same file | keeps lock | drops all locks on that inode | keeps lock on its own description |
| `fork` | child shares | child inherits then independent | child shares the description |
| Deadlock detection | no | yes, returns `EDEADLK` | yes, returns `EDEADLK` |

## Lock types, deadlock detection, and leases

Both `fcntl` and OFD locks support two modes. Shared mode is `F_RDLCK`. It allows many readers at once. Exclusive mode is `F_WRLCK`. It allows one writer and blocks others. Both also support a query form `F_GETLK`. It reports a conflicting lock if one exists. The kernel also detects deadlock. Deadlock means two processes each hold a lock that the other wants. Without help they would block forever. The kernel returns `EDEADLK` to one of them instead of hanging. `flock` does not detect deadlock. It just blocks. This is one more reason to prefer range locks when your locking graph is complex.

A file lease is a separate tool. You set it with `fcntl` `F_SETLEASE`. A lease tells a process when another process tries to open or truncate a file that the first process is caching. The holder can then flush or drop its cached state. Samba uses leases. Local caches use them too. They use leases so they do not keep stale data when a remote writer appears. A lease is not mutual exclusion in the usual sense. Mutual exclusion means only one holder can proceed. A lease is a coherence signal. Coherence means caches agree on current data. The kernel revokes the lease if the holder does not answer in time.

## The atomic rename pattern and its modern forms

The safest way to update a file that readers may read is this. Write the new content to a temporary file. Call `fsync` on it. `fsync` asks the kernel to flush the file's data to the device. Then `rename` the temp over the target. `rename` is atomic at the directory level. Atomic means it appears as one indivisible step. A reader sees the old file or the new file. The reader never sees a half written file. Package managers, editors, and config systems all replace files this way. The temp file must sit on the same filesystem as the target. `rename` is atomic only within one filesystem. Across filesystems it becomes a copy. A copy is not atomic.

The `fsync` before the `rename` is the step people skip. It matters for crash consistency. Suppose you `rename` a temp file whose data is still in the page cache. The page cache is the kernel's memory buffer for file data. A crash before writeback can leave the renamed file pointing at blocks that never reached the disk. The result is a zero length file or garbage. Flushing the temp first puts the bytes on the device before the name switches. This pattern also avoids in place corruption. You never overwrite the live file. You just replace its name.

Modern kernels add more options. `renameat2` supports `RENAME_NOREPLACE`. It renames only if the target does not exist. It does so atomically. This acts like link if absent. `renameat2` also supports `RENAME_EXCHANGE`. It swaps two existing paths atomically. Neither path ever disappears. `RENAME_EXCHANGE` closes the brief window where the temp exists and the target is being replaced. The cost is that the old version stays under the temp's name. You must clean it up. `O_TMPFILE` opens an unnamed temp file inside the directory. It has no visible name to leak. The final step is a `linkat` that gives it a name. This is the cleanest form of the temp plus rename pattern.

## Journaling and write ordering

A filesystem metadata update touches several on disk structures. Metadata means data about data, like inodes and directories. Creating a file is an example. Appending to a file is another. The update changes the inode, the directory entry, the block bitmap, and the data blocks. Suppose the machine crashes between two of those writes. The structures can then disagree. You can get a file whose size says it is large but whose blocks are unallocated. You can get a directory entry that points at a freed inode. That is metadata corruption.

Journaling fixes this. A journal is a reserved log region on disk. The filesystem writes the intended changes into the journal first. Then it applies them to the real structures. It marks the journal entry complete only after the real writes finish. If a crash happens, the filesystem replays the journal. It finishes incomplete updates or discards them. So the on disk state stays consistent. ext3 and ext4 have three classic modes. `data=ordered` journals metadata and writes data before its metadata commit. It is the default. It gives a good balance. `data=writeback` journals metadata but does not enforce data order. It is faster. But it can show old data after a crash. `data=journal` journals both data and metadata. It is the slowest. It is also the strongest. A backend engineer can read the mount's `data=` option to learn the real crash time guarantees.

```mermaid
flowchart LR
    A[Record change in journal] --> B[Apply change to real structures]
    B --> C[Mark journal entry committed]
```

Copy on write filesystems take a different path. btrfs and ZFS are examples. They do not update structures in place. They do not journal the intent. Instead they write new versions of changed blocks to free space. Once the write is complete, they switch the pointer to the new tree atomically. A crash leaves the old tree intact. Consistency is a property of the structure, not of a replayed log. ZFS and btrfs also checksum every block. A checksum is a small value that detects bit errors. They can detect silent corruption. On redundant storage they can repair it. Journaling alone does not do this. The tradeoff is more write amplification and more complex code. Write amplification means more bytes written than the user asked for. The consistency story is stronger though. And it does not need a separate journal.

## Write barriers and device reordering

Even with journaling, the kernel must stop the device from reordering writes in a way that breaks the log. A disk or its controller may reorder writes for speed. A volatile device cache may delay writes. A volatile cache loses its contents on power loss. Write barriers fix this. A write barrier is an ordering rule. The flush commands behind it force the device to finish certain writes before later ones. Then the journal commit is truly on disk before the metadata it protects is treated as durable. Durable means it will survive a crash. Modern kernels express this with block layer requests. `REQ_PREFLUSH` flushes the device cache first. `REQ_FUA` means force unit access. It writes through the cache. It replies only after the data is stable. `REQ_OP_FLUSH` is a standalone flush.

This is the same flush contract from the durability article. Crash consistency depends on it. Without barriers the journal guarantee can be undone. A device could write the metadata before the journal record. The lesson from both articles is the same. The kernel's promise rests on the device honoring flushes.

## Power-loss behavior, fsck, and self-healing

Without journaling a crash can leave the filesystem structurally inconsistent. A check tool like `fsck` must scan and repair it. `fsck` means filesystem check. That scan is slow. It sometimes loses data. With journaling the normal case needs no `fsck`. The kernel replays the log at mount. A journal aware `e2fsck` still runs. But mostly it just replays. Modern filesystems still provide `fsck` for rare corruption from hardware errors. Day to day crash recovery is automatic.

Copy on write filesystems recover differently. The old tree stays valid. A mount after a crash simply uses the last consistent tree. A scrub pass checks checksums against redundant copies and repairs them. The filesystem does not solve your app's own consistency. Journaling protects the filesystem's own structures. It does not protect the meaning of your data. Suppose your app must update two files together. The filesystem will not keep them in sync. You must do that yourself. You can use atomic rename, a transaction, or an intent log. Crash consistency at the filesystem level prevents corruption. Application consistency is a separate and higher layer concern.

## Application-level consistency: WAL, intent logs, and double-write

Sometimes correctness needs several changes to happen together. Or it needs a half applied update to stay invisible. Apps build on the tools above to get this. A write ahead log records the intended change in a separate log. Write ahead means you log before you apply. The log is appended sequentially and `fsync`'d before the change reaches the main data. After a crash you can replay the log to finish. Or you can undo it to roll back. Databases and key value stores are built on this. An intent log is a lighter form. It records the note 'I am about to do X'. The app clears it once X completes. A crash during X is then easy to detect and recover.

```mermaid
flowchart LR
    A[Write intent to WAL and fsync] --> B[Apply change to data files]
    B --> C[Mark intent done and fsync]
    C --> D[Crash before C is replayed from WAL]
```

The diagram shows the shape of a write ahead log. The intent is durable before the change. A crash during the change can be completed from the log. A crash before the intent leaves the data untouched. A double write buffer is a related technique. Storage engines like InnoDB use it. Each page is written twice. First it goes to a staging region. Then it goes to its real location. A torn page from a crash mid write can then be found and rebuilt. A torn page is a page that was only partly written. For updates that span two files the practical answer is usually this. Write the new combined state to a temp file. Call `fsync`. Then `rename` or `RENAME_EXCHANGE` the relevant names. Then `fsync` the directory. `fsync` on the directory makes the name change durable. This turns a multi object update into a sequence of steps that are each atomic.

Preallocating space with `fallocate` before writing is a related aid. It guarantees the blocks exist. A later write cannot fail with `ENOSPC` partway through. `ENOSPC` means no space left on device. Without preallocation you could get a half written file that you cannot recover. Combined with the atomic rename pattern it removes two common failure modes at once. The modes are crash and capacity.

## Observing locks and journaling

Held locks are visible in `/proc/locks` and via `lslocks`. The journal and mount options are visible through filesystem tools and `/proc/mounts`. When two writers seem to stomp on each other, the first step is to see what is locked and by which process. `fuser` and `lsof` answer the question 'which process has this file open'. That often explains a lock that will not release.

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

The code shows the safe update pattern. The program writes to a temp file. It syncs the temp. Then it renames the temp over the target. Any reader of `config.txt` sees the old file or the new file. The reader never sees a partial file. A crash before the rename leaves only the temp. The temp is harmless to clean up. The `lslocks` and `flock` commands show real locks held by processes. That is how you find a concurrency bug. `/proc/mounts` shows the journal mode that decides behavior at crash time.

## A realistic production example

A team kept a config file. A control loop rewrote it in place several times per minute. The loop opened the file, moved to the start, and wrote the new contents. Most of the time this worked. Now and then a deploy or a crash happened during the write. The file was left half old and half new. Or it was truncated to a smaller size. On the next start the service failed to parse its own config. It refused to boot. This is the worst kind of failure. It is not a simple crash. It is a self made state that will not start.

The fix used the atomic pattern. The service wrote the new config to a temp file in the same directory. It called `fsync`. Then it renamed the temp over the live path. The directory must be the same. Then the rename stays on one filesystem and stays atomic. The team also added an `flock` around the update. Then two control loop iterations could not race each other. The rename alone already stops readers from seeing partial data. For a later two file update they used `RENAME_EXCHANGE`. The config and its hash file swapped together. There was no window where one was new and the other was old. They preallocated the new file with `fallocate`. Then a full disk could not abort the write partway. After the change a crash during an update at worst left a stray temp file. The live config was always a complete and previously valid version. The lesson is this. In place writes are corruption waiting for a crash. Atomic rename plus `fsync` turns an unsafe update into a safe swap.

## How engineers actually reason about consistency

Engineers never update a live file in place. The temp plus rename pattern is the default for any file that another process may read. With a `fsync` before the rename, the swap is atomic and crash safe. When two names must change together, they use `RENAME_EXCHANGE` or a write ahead log.

They treat locks as coordination, not enforcement. Advisory locks work because every writer respects them. Your job is to make sure every writer does. Pick the right granularity with `flock` or OFD range locks. Remember that closing one descriptor can drop all POSIX locks that the process holds on that file.

They separate filesystem consistency from app consistency. Journaling keeps the filesystem valid across crashes. It will not keep two of your files coherent. That part is your design. Use atomic rename, transactions, intent logs, or a WAL to keep them coherent.

They remember the flush contract. Atomic rename is crash safe only if the temp's data reached the device before the rename. That is the `fsync` step. That step depends on the device honoring flushes. The durability article described that contract.

## Network and distributed filesystems, and application crash patterns

The locking and consistency rules change when the filesystem is not local. NFS and other network filesystems must pass locks and opens across the network. The server holds the true lock state. Clients use leases or delegations that the server can recall. This is why the server can revoke a long held advisory lock on NFS if the client goes silent. FUSE filesystems run operations in a user space daemon. Their consistency is whatever the daemon enforces. They also add a context switch and copy cost on each operation. For these filesystems the atomic rename pattern is still the safe update method. Durability of the `fsync` before `rename` depends on the server's own flush contract, not just the local kernel.

At the app layer the write ahead log pattern is the workhorse for crash consistency across many files. Here is a minimal sketch. Before you change the real data, append a record that describes the change to a sequential and `fsync`'d log. Then apply the change to the data files. Then mark the log record committed and `fsync` again. After a crash replay the log from the last committed point. Redo any change whose commit was not yet recorded. This turns a multi step update into an ordered and recoverable sequence. The filesystem does not need to understand the meaning of your data. Databases, key value stores, and some queue systems are built on exactly this. They often use a separate write ahead device. A fast NVMe is common. Then the `fsync`'d commits do not contend with bulk writes.

## Definitions

### Advisory locking

> A voluntary locking scheme. Examples are `flock`, POSIX `fcntl`, and OFD `fcntl` locks. The kernel does not block a process that ignores the lock. Correctness depends on every cooperating writer choosing to take the lock. OFD locks fix the odd process and inode surprises of POSIX locks.

### Atomic rename and its variants

> A pattern for safe updates. You write new content to a temp file. You flush it with `fsync`. Then you rename it over the target. Readers see either the old file or the new file. The switch is atomic. `renameat2` adds `RENAME_NOREPLACE`, which is an atomic link if absent, and `RENAME_EXCHANGE`, which is an atomic swap of two paths. `O_TMPFILE` gives you an unnamed temp file.

### Journaling and copy-on-write

> Journaling writes intended changes to a journal first. It then applies them to the real structures and commits only at the end. A crash replays the journal or discards incomplete updates. Copy on write filesystems write new versions of blocks to free space. They switch pointers atomically. The old tree stays consistent. They add checksums to detect and repair errors.

### A write barrier

> An ordering rule for the device. The device must finish earlier writes before later ones. The kernel enforces it with flush commands such as `REQ_PREFLUSH` and `REQ_FUA`. Then journal commits are truly durable. The device cannot reorder them away.

### Crash and application consistency

> Crash consistency means the filesystem survives a power loss without corrupt structures. Journaling or copy on write gives you that property. Application consistency is a higher layer property. It means your multi file or multi step updates stay coherent. You must build it yourself with atomic updates, intent logs, or a write ahead log.

## Beyond the definitions

### Why is advisory locking enough if the kernel does not enforce it

> The systems that share a file are usually cooperating parts that all use the same lock. Enforcement lives in the code that agrees to coordinate. That is simpler and faster than kernel mandated exclusion. Kernel mandated exclusion has its own sharp edges. OFD locks make that coordination sane.

### Why fsync the temp before renaming it

> `rename` only swaps names. If the temp's data is still in the page cache, a crash can leave the new file pointing at blocks that never reached the disk. Flushing first puts the bytes on the device before the name points at them.

### How is journaling different from application transactions

> Journaling protects the filesystem's own structures from corruption after a crash. It does not understand the meaning of your data. So if your design needs two files updated together, you must provide that guarantee yourself. You can use a transaction, an intent log, or a WAL.

### What do write barriers protect against

> A device can reorder or delay writes. It can write metadata before the journal record that should come first. That breaks the log. Barriers force ordering. Then the journal guarantee holds even with a reordering disk or a volatile cache.

### When would a rename still leave a problem

> If the temp and target are on different filesystems, `rename` is not atomic and may copy. If you skip the `fsync` before `rename`, a crash can leave the new file empty. If two names must change together, one `rename` leaves a window. You need `RENAME_EXCHANGE` or a WAL to close it.

### Why did closing a file drop my lock

> POSIX `fcntl` locks attach to the process and the inode. Closing any descriptor for that inode releases all POSIX locks that the process holds on it. OFD locks attach to the open file description, not the inode. They do not have this surprise. Prefer them for new code.

## Common misconceptions

**"flock stops other processes from writing."** It stops only processes that also take the lock. A writer that ignores the lock and opens the file directly still races. The lock is advisory. Coordination is a rule in your code, not an enforcement by the kernel.

**"Atomic rename means the data is durable."** Rename is atomic in visibility, not in durability. Visibility means what readers see. Durability means what survives a crash. Without an `fsync` of the temp first, a crash can leave the new file's blocks unwritten. Atomic and durable are two different properties.

**"The filesystem journals, so my files are safe."** Journaling keeps the filesystem consistent. It does not keep your multi file logic consistent. It will not keep a database and its index in sync. That is your job. You need transactions, a WAL, or atomic patterns.

**"In place rewrite is fine if I rewrite fast."** A crash or kill at any moment during the rewrite can leave a partial or truncated file. The next reader may be your own service failing to start. Atomic rename removes the window entirely. It does not just shrink it.

**"Mandatory locking is stronger, so use it."** Mandatory locking on Linux is fragile and mostly deprecated. It has odd interactions. Advisory locking by convention and OFD locks for sane range locking are the practical and reliable choices.

**"Closing one fd only affects that fd's lock."** For POSIX `fcntl` locks it affects all locks that the process holds on that inode. The kernel keys those locks by process and inode. This is the classic trap. OFD locks use the open file description as the key and avoid it.

## Summary

File consistency has two faces. Concurrency means handling parallel writers. You handle it by cooperation with `flock`, POSIX `fcntl` locks, or the newer OFD locks. All are advisory. They work only if every writer respects them. OFD locks fix the odd process and inode surprises of POSIX locks. Crash consistency for apps comes from the atomic rename pattern. `renameat2` with `RENAME_EXCHANGE` and `O_TMPFILE` extend that pattern. Crash consistency for the filesystem itself comes from journaling or copy on write. The rename pattern is the workhorse. You write a temp file, flush it, then swap or exchange. This keeps a live file from ever showing partial content. The flush before the swap is what lets it survive a crash. The filesystem journal keeps its own structures valid. It does not make your multi file updates coherent. That part is your design. You build it from atomic updates, intent logs, and write ahead logs. The next article in this stage leaves software. It looks at the hardware underneath: disks, SSDs, NVMe, and RAID.
