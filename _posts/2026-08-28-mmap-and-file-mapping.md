---
mermaid: true
title: "mmap and File Mapping"
date: 2026-08-28
categories: ["System Engineering"]
tags: [mmap, file-mapping, shared-memory, page-cache, zero-copy, madvise]
series: "System Engineering"
stage: "Stage 6 - Memory Management"
stage_order: 6
series_order: 4
---

The previous chapter closed by pointing at `mmap` as the place where many page faults are born. This chapter opens that mechanism. It is the fourth article of Stage 6.

`mmap` is the system call that maps a region of memory to something else: a file, a device, or nothing at all. After it returns, the program reads and writes bytes through ordinary pointers, and the kernel turns those accesses into file I/O, copy-on-write, or shared memory behind the scenes. It is the bridge between the address space you have been reading about and the data that actually lives on disk or in another process.

## What mmap does at a high level

A normal file read copies bytes from the kernel's page cache into a buffer you allocated. `mmap` instead asks the kernel to project the file directly into your virtual address space. After that, reading the file is just dereferencing a pointer, and the page faults described in the last chapter are what pull the bytes in.

```mermaid
flowchart LR
    F[File] --> C[Page cache]
    C -->|copy into| UB[User buffer - read path]
    C -->|mapped, no copy| UP[User pointer - mmap path]
```

The difference is the copy. With `read` and a buffer, the data moves from the cache into your memory. With `mmap`, your pointer sits on top of the cache, so there is no second copy. That is the heart of why `mmap` is described as zero-copy for the read side, and it is also why the faults you saw earlier show up: the first touch of a mapped page pulls it from the file.

## Anonymous versus file-backed mappings

An anonymous mapping is not attached to any file. It behaves like a private region of memory that several processes can share if created for that purpose, or that one process uses as a large allocation. It is backed by RAM (and possibly swap) and starts zeroed.

A file-backed mapping attaches a file to the region. Reading a byte reads the file; writing, depending on flags, either changes the file or a private copy. The file is the source of truth, and the page cache is the middle layer that holds the bytes while they are mapped.

The two kinds share the same machinery. Both are ranges in the virtual address space described by a `vm_area_struct`, both translate through page tables, and both can fault. They differ in what happens when the kernel needs to fill or reclaim a page.

## MAP_SHARED versus MAP_PRIVATE

These two flags decide what a write means, and they are the most important choice you make with `mmap`.

A `MAP_SHARED` mapping means writes are visible to other processes that map the same file, and they are eventually written back to the file itself. This is how memory-mapped shared state works: two processes mapping the same file see each other's changes.

A `MAP_PRIVATE` mapping means writes are not shared and are not written to the file. The kernel implements this with copy-on-write, the same mechanism from the previous chapter. The first write to a private page forks a copy, and the writer sees its own version while others keep the original.

```mermaid
flowchart TD
    W[Write to mapped page] --> K{Shared or private?}
    K -->|shared| V[Visible to others and to file]
    K -->|private| P[Copy-on-write, private only]
```

The trap is assuming one when you got the other. Mapping a file privately and then writing to it changes nothing on disk, which surprises people who expected their edits to persist. Mapping it shared and writing changes the file under every other reader, which surprises people who expected privacy.

## The flags worth knowing

Beyond the sharing mode, a few flags and protections decide behavior.

| Flag or protection | Effect |
|---|---|
| PROT_READ | The mapping may be read |
| PROT_WRITE | The mapping may be written |
| PROT_EXEC | Instructions may be fetched from the mapping |
| MAP_SHARED | Writes are shared and written back to the file |
| MAP_PRIVATE | Writes are private via copy-on-write |
| MAP_ANONYMOUS | The mapping has no file backing |
| MAP_FIXED | Place the mapping at the exact address given |
| MAP_POPULATE | Fault all pages in at mmap time instead of on access |
| MAP_NORESERVE | Do not reserve swap space for the mapping |

`MAP_FIXED` is dangerous because it can overwrite an existing mapping at that address, which can clobber your program's own code or libraries. `MAP_POPULATE` trades a slow mmap call for faster later access, which is useful when you would rather pay the fault cost up front than suffer it during a request. `MAP_NORESERVE` is a memory-overcommitting choice that matters when swap reservation matters.

## How mmap meets the page cache

For a file-backed mapping, the kernel does not keep a separate copy of the file for your process. It uses the same page cache that `read` and `write` use. If another process reads the file with `read`, or maps it, they share those cached pages. This is why mapping a large file that is also read elsewhere does not double the memory: the bytes live once in the cache.

```mermaid
flowchart LR
    P1[Process A maps file] --> Cache[Page cache holds file pages]
    P2[Process B maps same file] --> Cache
    P3[Process C reads file] --> Cache
    Cache --> Disk[File on disk]
```

Writes to a shared mapping go through the cache too. They mark pages dirty, and the kernel writes them back to disk later, just as it would for a `write` syscall. The timing of that writeback is up to the kernel unless you call `msync` to force it, which is one reason a crash can lose recently written mapped data.

This also explains the major-fault story from the last chapter. A mapped page that is not in the cache faults in from disk; a mapped page already in the cache faults in as a minor fault. The page cache is the bridge that makes both possible, and it is shared across every way the file is accessed.

## mmap as zero-copy reading

The classic use is serving or scanning a large file without copying it through a user buffer. Instead of `open`, `read` into a buffer, process, repeat, you `mmap` the file and walk the pointer. There is no per-call copy from kernel to user, and the kernel only faults in the pages you actually touch.

This matters for large file handling. A tool that scans a multi-gigabyte file can map it and read sequentially, letting the kernel page it in a window at a time, rather than managing a buffer loop by hand. For sending a file over a socket, `mmap` plus writing the pointer, or better, `sendfile`, avoids copying the bytes through user space at all.

The catch is that `mmap` works in page-sized units. A small file still consumes at least one page of address space, and a file smaller than a page wastes the rest of that page. For many tiny files, the overhead of a mapping per file outweighs the copy you avoided, and ordinary `read` is simpler and faster.

## Prefaulting and pinning with madvise and mlock

Because the first touch faults, a workload that needs predictable latency can ask the kernel to prepare pages ahead of time. `madvise(MADV_WILLNEED)` tells the kernel to prefetch a range, turning later faults into hits. `madvise(MADV_DONTNEED)` tells it a range will not be used soon, releasing the pages. `madvise(MADV_HUGEPAGE)` asks for huge pages in that range, tying back to the translation chapter.

`mlock` goes further: it pins pages in RAM so they cannot be swapped out and will not fault. This removes both the swap risk and the fault latency, at the cost of tying up physical memory and, on most systems, requiring privilege. Pinning a large region under memory pressure makes the situation worse for everyone else, so it is used sparingly for small, critical areas such as crypto keys or a latency-critical hot loop.

## Use cases that earn their place

Shared memory between processes is the clearest win. Two processes that map the same file with `MAP_SHARED`, or that use an anonymous shared mapping, exchange data by writing memory, with no syscalls on the hot path. This is how many high-performance IPC designs avoid copying.

Read-only shared datasets are another. A service with several worker processes can map a large read-only index file once and let every worker share the cached pages. The file is loaded into the cache a single time, and each worker's resident memory stays small because they point at the same frames.

Large file scanning benefits from avoiding the copy and the buffer management. And hardware access, by mapping a device register range, lets a driver touch hardware through pointers, though that is specialized and outside the usual backend path.

## Risks and pitfalls

The most famous trap is `SIGBUS`. If you map a file and then the file is truncated or the mapped region falls past the end of the file, touching that page raises a bus error rather than a polite fault. This happens when a background job rewrites a file in place while you have it mapped.

Mapping a file larger than RAM is fine, because not all of it is resident at once. What breaks is assuming it is all available instantly: touching far regions faults from disk, and under pressure those pages can be reclaimed and faulted again. An anonymous private mapping of a huge region is different: it is backed by RAM and swap, so it can push the system toward swapping if the working set is large.

Address space limits matter on 32-bit builds, where a few gigabytes of mapping can exhaust the space. Threads and `fork` interact badly with large mappings because the address space is duplicated or shared, and a `MAP_PRIVATE` file mapping in a forked child behaves with copy-on-write just like the heap does.

`MAP_SHARED` writes are not flushed immediately. If you write through a shared mapping and the machine crashes before writeback, those writes can be lost, unlike a direct `write` that you might `fsync`. Use `msync` when durability matters.

## Observing mappings and residency

The mappings of a process are visible in `/proc/<pid>/maps`, and their memory accounting in `/proc/<pid>/smaps`. The latter shows, per mapping, its size, resident set, and the page size used, which is how you confirm a mapping is using huge pages or how much of it is actually resident.

```go
package main

import (
    "fmt"
    "os"
    "syscall"
)

func main() {
    data := []byte("mmap example content for a small file\n")
    tmp, err := os.CreateTemp("", "mmap-*.txt")
    if err != nil {
        panic(err)
    }
    defer os.Remove(tmp.Name())
    if _, err := tmp.Write(data); err != nil {
        panic(err)
    }
    tmp.Sync()
    tmp.Seek(0, 0)

    size := int64(len(data))
    mem, err := syscall.Mmap(int(tmp.Fd()), 0, int(size), syscall.PROT_READ, syscall.MAP_PRIVATE)
    if err != nil {
        panic(err)
    }
    fmt.Printf("mapped content: %q\n", string(mem))
    if err := syscall.Munmap(mem); err != nil {
        panic(err)
    }
    fmt.Println("unmapped cleanly; see /proc/self/maps while running to inspect")
    select {}
}
```

```bash
go build -o mmapdemo main.go
./mmapdemo &
pid=$!
sleep 0.3
grep -E "mmap-.*\.txt" /proc/$pid/maps
grep -A6 "mmap-.*\.txt" /proc/$pid/smaps
kill $pid
```

What it shows is that the mapped file appears as a region in `maps`, and `smaps` breaks down how much of it is resident. For a production service, scanning `smaps` for the large mappings tells you which files are mapped, how much is actually in RAM, and whether huge pages are in use, which connects directly back to the TLB and working-set chapters.

## A realistic production example

A service used a large read-only index file to answer queries. Each worker process opened the file and read it into its own buffer at startup, so the index was loaded once per worker and the total resident memory scaled with the number of workers. The team switched to mapping the file `MAP_PRIVATE` and sharing it across workers, which cut resident memory sharply because all workers pointed at the same cached pages.

Then a deploy pipeline started rewriting that index file in place: it opened the existing path, truncated it, and wrote the new contents, rather than writing to a temporary file and renaming. A worker that had the old file mapped, and that touched a page past where the new truncated file ended, received `SIGBUS` and crashed. The mapping still pointed at the old inode for workers that held it, but the rewrite raced with workers that re-mapped, and the in-place truncation removed pages from under them.

The fix was to stop rewriting in place. The pipeline wrote the new index to a temporary file, called `fsync`, then renamed it over the old name. Renaming changes which inode the path points to but leaves existing mappings valid, because they hold the old inode. Workers that already had the old file mapped kept serving from it until they reloaded, and new workers mapped the new inode. They also added a signal handler for `SIGBUS` on the mapping as a belt-and-suspenders guard, so a truncated mapping would log and recover rather than crash. The lesson was that `mmap` couples your address space to a file's lifetime, and the safe pattern is immutable, versioned files that are replaced by rename, never truncated under a live mapping.

## How engineers actually reason about mmap

They choose it for sharing and for avoiding copies, not for every file. Small files, or files read once in a streaming fashion, are often simpler and faster with `read`. The win from `mmap` grows with file size and with how many readers or writers share the data.

They respect the coupling to the file. A live mapping assumes the file stays at least as large as the region touched, so the deployment and rotation story around that file matters as much as the `mmap` call itself. Immutable files swapped by rename are the safe default.

They think about residency. A mapped file is not free memory, and touching far parts faults from disk. `madvise` hints and, for the truly hot path, `mlock`, turn unpredictable faults into predictable behavior, but both cost memory that is then unavailable to the rest of the system.

They remember durability. Shared mappings are written back lazily, so important writes need `msync` or an explicit `fsync` path, or a crash can lose them.

## Beyond the basic flags: fixed replacement, huge pages, locked and populated mappings, mremap, and mincore

The common flags cover most uses, but a set of related options and helpers matter in production. `MAP_FIXED` overwrites whatever mapping already sits at the address, which can silently clobber your own code or libraries. `MAP_FIXED_NOREPLACE` does the opposite: it places the mapping only if the address is free and fails otherwise, which is almost always what you wanted `MAP_FIXED` for. Use the safe variant unless you are deliberately taking over a region you control.

For special memory, `MAP_HUGETLB` requests the mapping from the huge-page pool, and `MAP_SYNC` (with a persistent-memory or DAX file) makes `msync` flush writes all the way to persistent media rather than just the page cache. `MAP_LOCKED` is the combination of `mmap` and `mlock` in one call so the region is pinned immediately, and `mlockall(MCL_FUTURE)` pins everything faulted in afterward. A related helper, `mremap`, grows or shrinks an existing mapping in place, which is how some allocators extend the heap and how a shared buffer is resized without copying. `mincore` answers a different question: given a range, which of its pages are currently resident, so a program can learn residency without faulting the pages in, which is useful for warmup checks and for predicting whether the next access will be a hit or a major fault.

## memfd, anonymous shared memory, and fork and core-dump safety

Not every shared mapping needs a file on a visible filesystem. `memfd_create` makes an anonymous file that lives in tmpfs, has no path, and can be shared with another process by passing its descriptor over a Unix socket or through `fork`. Because it is a real file descriptor, it can also be sealed with `F_SEAL_SEAL`, `F_SEAL_SHRINK`, `F_SEAL_GROW`, and `F_SEAL_WRITE` so that a receiver can trust the contents will not change underneath it. This is the modern, safer basis for passing shared memory and for sandboxing, and it ties directly to the sealing discussion from the file-descriptor article.

```mermaid
flowchart LR
    A[Process creates memfd] --> B[Anonymous tmpfs file, no path]
    B --> C[Share fd via fork or SCM_RIGHTS]
    C --> D[Both map it MAP_SHARED]
```

The older paths still exist: POSIX shared memory via `shm_open` plus `mmap`, and the legacy System V `shmget` and `shmat`. They work, but they leak named objects and lack sealing, which is why `memfd` is preferred for new code. Fork and core-dump safety matter when a mapping holds secrets. `MADV_DONTFORK` excludes a region from the child after `fork`, so a child that goes on to `exec` another program never sees a parent's keys. `MADV_DONTDUMP` keeps a region out of core dumps for the same reason, and `MADV_WIPEONFORK` zeroes the region in the child so the child can never read the parent's copy. On the other side, `MADV_MERGEABLE` lets the kernel deduplicate identical pages across processes, a feature called KSM that saves memory for many similar guests or containers at some CPU cost.

## Making mapped writes durable with msync

A `MAP_SHARED` write lands in the page cache and is written back lazily, so it is not durable until the kernel decides to flush. `msync` forces that flush for a range. `MS_SYNC` blocks until the pages are written back to the file on storage, `MS_ASYNC` schedules the writeback without waiting, and `MS_INVALIDATE` discards cached copies so later reads come from the file. For correctness across a crash, `msync(MS_SYNC)` (or an `fsync` on the underlying file descriptor) is what turns a shared mapped write into a committed one, and only after that can you assume the bytes will survive power loss.

This is the mapped analog of the durability rules from the storage stage. The trap is assuming a write through a pointer is saved just because it returned: like a buffered `write`, it is only a promise to the cache until explicitly synced. The difference is that a mapped write has no `write` call to attach the sync to, so the discipline is to call `msync` on the exact range you need durable, and to do it on a cadence rather than after every write.

## Definitions

### mmap

> A system call that maps a file, device, or anonymous region into a process's virtual address space so it can be accessed through ordinary pointers, with the kernel handling the underlying I/O and faults.

### A file-backed mapping

> A mapping whose pages are sourced from a file through the page cache, so reads pull file bytes and shared writes are eventually written back to the file.

### MAP_SHARED

> A mapping flag where writes are visible to other mappers of the same file and are written back to the file, making the mapping a shared view of the data.

### MAP_PRIVATE

> A mapping flag where writes are kept private through copy-on-write and are never written to the file, so the mapper sees its own version unseen by others.

### The page cache

> The kernel's in-memory store of file contents, shared by `read`, `write`, and file-backed `mmap`, so the same file mapped or read by many processes uses the bytes only once in memory.

### madvise and mlock

> `madvise` gives the kernel hints about future access so it can prefetch or release pages, while `mlock` pins pages in RAM so they cannot be swapped or faulted, trading memory for predictability.

## Beyond the definitions

### Why is mmap described as zero-copy

> Because it projects the kernel page cache directly into the address space, so reading a file is a pointer dereference rather than a copy from kernel buffer to user buffer. The data lives in one place and is shared with every other accessor of the file.

### What is the difference between reading a file and mapping it

> `read` copies bytes into a buffer you own and pays the copy per call; `mmap` lets you address the cached pages directly and pays through page faults on first touch. Mapping avoids the copy but adds fault and residency behavior to manage.

### Why does a truncated mapped file cause SIGBUS

> The mapping promises a range of addresses backed by the file. If the file shrinks so a mapped page no longer exists, there is no data to fault in, and the access raises a bus error instead of a normal page fault. Replacing a file by rename avoids this because existing mappings keep their original inode.

### When should you avoid mmap

> For many small files, where mapping overhead per file outweighs the copy saved; for one-shot streaming reads where a simple `read` loop is clearer; and when you cannot control the file's lifecycle, because a mapped file that gets truncated or replaced under you is a hazard.

### How do you make mapped access predictable

> Use `madvise(MADV_WILLNEED)` to prefetch the range you will touch, `madvise(MADV_DONTNEED)` to release what you are done with, and `mlock` only for small critical regions that must never fault or swap. The goal is to move the fault cost to a known time.

## Common misconceptions

**"mmap loads the whole file into memory."** It maps the address range. Pages are faulted in on access and may be reclaimed under pressure, so a file larger than RAM is fine and only touched parts become resident.

**"Writing to a mapped file saves it immediately."** A shared mapping writes back lazily through the page cache. Without `msync` or `fsync`, a crash can lose those writes, unlike an explicit durable write path.

**"Private mappings are always safe to write."** They are private to your process via copy-on-write, but they are never written to the file, so edits vanish when the mapping is gone unless you also write them elsewhere.

**"mmap is always faster than read."** For large shared or randomly accessed files it often wins by avoiding copies. For many small files or simple streaming, the mapping overhead and fault handling make `read` the better choice.

**"Sharing memory through mmap needs a real file."** An anonymous mapping created shared, or a POSIX shared memory object, gives the same interprocess sharing without a file on a filesystem, which is often cleaner for IPC.

## Summary

`mmap` projects files, devices, or anonymous regions into the address space so they are reached through pointers, with the kernel turning accesses into page-cache reads, copy-on-write, or shared writes. The sharing mode decides whether writes are visible and durable, the page cache means the bytes live once and are shared with every reader, and `madvise` or `mlock` trade memory for predictable faults. The coupling to the file's lifetime is the real hazard, and the safe pattern is immutable files replaced by rename. The next chapter steps down from these system mappings to the memory a program asks for day to day, through stack and heap layout and the allocators that hand it out.
