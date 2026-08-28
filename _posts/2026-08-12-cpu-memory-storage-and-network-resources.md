---
mermaid: true
title: "CPU, Memory, Storage, and Network Resources"
date: 2026-08-12
categories: ["System Engineering"]
tags: [cpu, memory, storage, networking, resources]
series: "System Engineering"
stage: "Stage 1 - Systems Programming Foundations"
stage_order: 1
series_order: 2
---

## The basic picture

This is the second chapter in the Systems Programming Foundations arc. The first chapter argued that systems programming is really about seeing the resources underneath every operation. This chapter names those resources concretely. Almost every system you will ever build, debug, or operate spends four things to do its work: CPU time, memory, storage, and the network. Learn to recognize them and learn how they limit each other, and you have the vocabulary for the rest of the roadmap.

The chapter is deliberately broad rather than deep. Later stages dig into caches, schedulers, filesystems, and protocols one at a time. Here the goal is simpler: understand what each resource is, what happens when it runs out, and why a slow system is often slow because of a resource you cannot see on a utilization graph.

Every system spends resources to do work. The four that show up in almost every system are CPU, memory, storage, and the network.

The CPU runs instructions. Memory holds the data and program state the CPU needs quickly. Storage keeps data for longer periods, even after a process stops. The network moves data between processes, machines, and services.

These resources are connected, but they are not interchangeable. More CPU does not fix a slow disk. More memory does not automatically fix a slow network dependency. A system can have enough total capacity and still be slow because work is stuck in a queue, a lock, a connection pool, or a remote service. This last point trips up a lot of engineers early on: it is entirely possible for every single resource on a machine to look fine on a dashboard while users still experience a slow, broken system, because the thing actually limiting progress is invisible to a simple utilization graph.

The central skill in this topic is learning to ask:

> Which resource is limiting the work, how is the work waiting for it, and what happens when the resource reaches its limit?

That question, asked correctly and answered with evidence rather than assumption, is worth more than memorizing any specific number in this chapter.

## What counts as a resource

A resource is something a system consumes to perform work and that has a limited capacity or cost. CPU time, memory space, disk bandwidth, and network bandwidth are resources, but so are file descriptors, database connections, threads, queue slots, and locks. Notice that this definition is broader than the four headline resources this chapter focuses on. Almost anything a program can run out of qualifies. CPU, memory, storage, and network are simply the four that sit underneath nearly everything else you will meet later. A database connection is ultimately a network socket plus some memory on both ends. A thread is ultimately CPU scheduling plus a chunk of memory for its stack.

The four resources in this article are useful because they form the basic path of many operations:

```mermaid
flowchart LR
    Request[Incoming request] --> CPU1[CPU work]
    CPU1 --> Memory[Read and update memory]
    Memory --> Storage[Read or write storage]
    Storage --> Network[Call another service]
    Network --> CPU2[Process the response]
    CPU2 --> Response[Return response]
```

Not every request uses all four in the same way. A computation-heavy request may spend almost all its time on the CPU. A file download may spend more time waiting for storage or the network. A request to another service may spend most of its time waiting for the remote service to respond, with the local CPU doing almost nothing but idling on a socket read.

The diagram is not a fixed sequence, and it would be a mistake to read it as one. A real system may read from memory before storage, use the network several times, or process many operations concurrently across dozens of requests at once, each at a different stage of this pipeline simultaneously. It is a starting model for asking where time and capacity go, not a description of the literal order every request follows.

## The CPU: where instructions actually run

The CPU executes instructions. Instructions perform operations such as arithmetic, comparisons, branches, memory accesses, function calls, and synchronization. Every line of code you have ever written, no matter how high-level the language, eventually becomes a sequence of these small, mechanical steps.

When people say a program "uses CPU," they usually mean the program is actively consuming processor time instead of waiting for another resource. High CPU is not automatically bad, and it is worth being explicit about that, because a lot of engineers instinctively treat a high CPU number on a dashboard as an alarm. A CPU running at high utilization may mean the system is using its hardware efficiently, getting real value from money already spent on it. It becomes a problem when work cannot finish within its required latency, tasks wait too long for CPU time, or there is no capacity left for important work such as a sudden traffic spike or a background job that needs to run.

### What a core can really do

CPU capacity depends on more than core count. It also depends on clock speed, instruction cost, cache behavior, branch prediction, memory latency, and the work being performed. Two machines with the same core count and the same advertised clock speed can perform very differently on the same workload, purely because of cache sizes or how predictable the workload's branches are.

A core can execute only a limited amount of work at a time. If more runnable work exists than the available cores can execute, the operating system schedules that work over time. Some tasks run while others wait, taking turns in slices measured in milliseconds or less.

```text
Available CPU capacity
    = number of usable cores
    × effective work per core
    × time available
```

This is a simplified model, and it is better treated as a mental scaffold than a formula you would ever actually compute. Effective work per core changes with the instruction mix, cache misses, branch behavior, CPU frequency, and other activity on the machine. A core running tight, cache-friendly numerical code might sustain several times the effective throughput of the same core running code that jumps around memory unpredictably, even though both are "using 100% CPU" by the numbers a monitoring tool reports.

### When the CPU is the limit

A workload is CPU-bound when the CPU is the main limit on how quickly it can progress. Examples include compression, encryption, image processing, parsing, compilation, and numerical calculations, anything where the bottleneck is genuinely "how many instructions can be executed" rather than "how long until some external thing responds."

If a CPU-bound program receives more CPU capacity and the work scales well across cores, its throughput may improve. If the program is single-threaded, adding more cores may not help at all, because a single thread can only ever occupy one core regardless of how many sit idle next to it. If threads share a lock or compete for memory bandwidth, adding more threads may make the program slower, not faster, as the overhead of coordinating between them starts to exceed the benefit of parallel execution.

This is why "use more threads" is not a universal performance solution, and treating it as one is one of the more common beginner mistakes in systems work. Threads still need CPU time, and the cost of coordinating between them does not show up until you actually run the program under realistic concurrency.

### Busy is not the same as slow

A service can have low CPU utilization and still be slow. It may be waiting for storage, a network dependency, a lock, or a queue, with the CPU sitting mostly idle the entire time a user stares at a spinner. Conversely, a service can have high CPU utilization and still have good latency if the work is predictable and there is enough capacity to absorb it without queueing.

CPU utilization answers "how busy is the processor?" It does not answer "how long is a request waiting?" Both questions matter, and conflating them is one of the fastest ways to misdiagnose a performance problem. An engineer who sees low CPU and concludes "the CPU isn't the problem, so let's look elsewhere" is usually right. An engineer who sees low CPU and concludes "nothing is wrong" has stopped one step too early.

### Who decides what runs next

The operating system scheduler decides which runnable thread receives CPU time. It tries to share the CPU according to priorities and scheduling policies, but the result is not that every task runs continuously from start to finish. Tasks are interleaved, sometimes at a granularity so fine that it is invisible to anything but a specialized tracing tool.

A context switch occurs when the CPU stops running one thread and starts running another. Context switches are necessary for sharing the CPU, but they require saving and restoring execution state and may reduce cache locality, since the new thread's data is unlikely to already be sitting in the cache the old thread was just using. Excessive thread creation, too many runnable tasks, or heavy synchronization can increase scheduling overhead to the point where the machine spends a meaningful fraction of its time switching between work rather than doing it.

The detailed behavior of scheduling and context switches belongs in later articles, where the mechanics get considerably more specific. The important point here is that CPU is a shared resource, and waiting for CPU is different from waiting for I/O. A thread stuck behind a busy CPU and a thread stuck waiting on a slow disk look completely different to the operating system, even though both simply appear "not running" to someone glancing at a process list.

## Memory: what the machine keeps close

Memory, usually meaning RAM in this context, holds the instructions and data that active programs need. The CPU can reach memory much faster than storage, but memory is limited and loses its contents when power is removed, which is the fundamental tradeoff that separates it from storage entirely.

Memory is used for more than application objects. A process also needs memory for its code, stacks, heaps, shared libraries, runtime structures, buffers, caches, and operating-system bookkeeping. A program that "only" allocates a few objects on the heap is still, invisibly, consuming memory for all of those other structures underneath it.

### Why memory stalls look like CPU problems

When a program needs data, the data must be available through a memory path before the CPU can work with it. If the data is not in a nearby cache, the CPU waits longer, sometimes for what amounts to hundreds of wasted cycles on a modern processor. If it is not mapped in the process's usable memory, the operating system may need to handle a page fault, an even more expensive interruption. If the system is under memory pressure, it may reclaim pages or move data to storage, which is slower again by several more orders of magnitude.

These events have different costs, but they all show the same principle: memory behavior affects CPU progress, even though from the outside, both a cache miss and a page fault might just look like "the program is running slowly."

### Capacity, bandwidth, and latency are separate limits

Memory capacity is how much data can be held. Memory bandwidth is how quickly data can be transferred. Memory latency is how long it takes to begin receiving data after requesting it. It is easy to conflate these three, but a system can be constrained by any one of them independently of the other two.

A workload may have enough memory capacity but still be limited by memory bandwidth. For example, a program that scans a very large array repeatedly may spend more time moving data than performing calculations, so adding more RAM capacity does nothing to speed it up, because capacity was never the constraint.

Another workload may fit comfortably in total memory but perform poorly because it accesses data randomly and misses the CPU caches frequently. This is why the amount of memory alone does not describe memory performance. Two programs using the exact same number of megabytes can have wildly different runtimes purely based on the access pattern they use.

### When memory runs out

Memory pressure occurs when active demand approaches or exceeds the memory available to the system. The operating system may reclaim unused cache pages, compress memory, swap pages to storage, or terminate a process when it cannot satisfy the demand. The specific response depends on the operating system and its configuration, but the underlying situation is the same: more is being asked for than is available.

Swapping means moving memory pages between RAM and storage to free RAM for other work. Storage is much slower than RAM, so heavy swapping can cause a system to spend most of its time moving pages instead of doing useful work. This condition is called thrashing, and it is one of the more dramatic failure modes in systems programming, because a machine under heavy thrashing can appear almost completely unresponsive despite technically still being "up."

Memory pressure can also create indirect failures that are harder to trace back to their root cause. A process may spend more time waiting for page faults. The garbage collector of a managed runtime may run more often, stealing CPU time that would otherwise go to useful work. The operating system may kill a process outright, often the one using the most memory at the moment, regardless of whether that process caused the pressure. A cache may evict useful data and force repeated reads from a slower layer, quietly turning a memory problem into a storage problem downstream.

### Who is responsible for returning memory

At the application level, memory may be owned by a data structure, request, thread, process, or cache. At the operating-system level, memory is associated with address spaces and pages, a lower-level bookkeeping system the application code rarely sees directly.

Ownership matters because memory that is no longer needed must become reclaimable, and this reclamation does not happen automatically just because the code stopped using a value. A memory leak occurs when a program keeps references to memory it no longer needs, preventing that memory from being reused, even in languages with garbage collection, where a leak simply means "still reachable, but functionally dead." A cache that grows without a bound is a form of resource-management failure even if every cached object is technically valid and correctly referenced. Validity is not the same thing as necessity.

The virtual-memory article later will explain how each process receives an address space and how the operating system maps that address space to physical memory, filling in the mechanics this section deliberately leaves out.

## Storage: keeping data past a restart

Storage keeps data beyond the lifetime of a process and usually beyond a machine restart. Examples include hard drives, SSDs, NVMe devices, distributed filesystems, and object-storage services, ranging from a single spinning disk in a laptop to a globally replicated storage service spanning continents.

Storage has several properties that must be considered separately, and conflating them is a common source of confusion when engineers discuss "storage performance" as though it were a single number:

- Capacity: how much data can be stored
- Throughput: how much data can be read or written per second
- Latency: how long one operation takes
- IOPS: how many individual input/output operations can be completed per second
- Durability: whether acknowledged data survives failures
- Availability: whether the data can be accessed when needed

A storage device can have high throughput and still have poor latency for small random reads, which is exactly the situation with traditional spinning disks doing random access rather than sequential streaming. It can have low latency for cached data while a cache miss requires a much slower device operation, so the same device can feel instant or sluggish depending purely on what was accessed moments before. It can have enough capacity today but grow beyond its limit next month, quietly, without a single alert firing until the day it actually fills.

### Why storage is not just slow memory

Storage and memory both hold data, but they serve different roles. Memory is designed for fast access by active programs. Storage is designed to retain data and provide more capacity, usually with higher access latency, often by several orders of magnitude compared to RAM.

The operating system often makes storage appear more memory-like through the page cache. Recently used file data may be kept in memory, so reading the same file again can be much faster. This can make a program appear fast during a test while behaving differently after a restart or under memory pressure, when the cache has been emptied and every read has to go all the way to the physical device again.

The page cache is useful, but it can also hide the real storage behavior in exactly the situations where you most need to see it. A benchmark that reads the same small file repeatedly may measure memory rather than the storage device, giving a falsely optimistic picture of how the system will perform once real users start reading files that were never touched before.

### What it means for a write to be durable

Durability answers whether data remains available after a failure such as a process crash, operating-system crash, power loss, or device failure. It is a narrower and more precise question than "did the write succeed," and the gap between those two questions is where a surprising number of data-loss incidents live.

An application may write data into a process buffer or the operating-system page cache and receive a successful return before the data is physically durable on the device. The exact guarantees depend on the filesystem, storage device, operating system, and flush operations, and a program that never investigates these guarantees is implicitly trusting an assumption it never actually verified.

This is why databases use techniques such as write-ahead logging and explicit synchronization. They need a reliable boundary that separates "the program requested the write" from "the system can recover the write after a crash," and building that boundary correctly is one of the harder and more consequential pieces of engineering in the entire storage stack, covered in far more depth in the storage-engine articles later in this roadmap.

### All the ways storage can fail without filling up

Storage pressure is not only "the disk is full." It can also mean the device is too slow, I/O queues are full, writeback is delayed, or a filesystem has run out of inodes or other metadata capacity, a subtler failure where there might be plenty of raw bytes free but no room left to create a single new file, because every inode slot is already used.

When storage fills, normal operations may fail in surprising places that have nothing obviously to do with disk space. A service may be unable to write logs, create temporary files, update a database, or create a socket-related state file. Disk-full conditions should be treated as a planned failure mode, not an impossible event, because in a long-running production system, they are closer to a certainty than an edge case.

## The network: when one machine talks to another

The network moves data between processes, machines, services, regions, and sometimes continents. It introduces a boundary that is different from a function call inside one process, and that difference is not just a matter of degree. It is a difference in kind, one that shows up again and again throughout the rest of this roadmap.

Network communication can be delayed, lost, duplicated, reordered, rejected, or interrupted. Even when a reliable transport such as TCP is used, the application still has to handle connection establishment, timeouts, remote process failure, and the possibility that a request completed even though its response did not arrive, a scenario a local function call can simply never produce, because a function call either runs or it doesn't. It cannot half-complete on the other side of a wire that got cut.

### What you spend to send data

Network capacity includes bandwidth, packet-processing capacity, connection capacity, and the ability of the receiving service to process incoming work. All four can be limits independently, and a system can be constrained by any one of them while the others sit nowhere near their ceiling.

Bandwidth is the amount of data that can be transferred over a period. Latency is the time taken for data to travel and be processed. A high-bandwidth link can still have high latency, the way a satellite connection can move enormous amounts of data per second while still taking hundreds of milliseconds for the first byte to arrive. A low-latency link can still become congested when too much data is sent, the way a nearly instant local network can still choke under enough simultaneous traffic.

```text
Network request time
    = connection setup
    + request transmission
    + server queueing
    + server processing
    + response transmission
```

This is a simplified model, but it shows why network latency is not only the physical distance between two machines. Queueing, encryption, routing, server load, and application processing all contribute, and any one of them can dominate the total time depending on the situation. A request across the world to an idle server can sometimes be faster than a request to an overloaded server sitting in the next rack.

### Connections cost something

A connection consumes state in the client, the server, the operating system, and sometimes load balancers or firewalls along the path between them. A service that creates a new connection for every request may spend significant time on setup and may exhaust connection limits, particularly under the kind of traffic spike that makes this problem matter most.

A connection pool keeps a bounded number of established connections available for reuse. Each request borrows a connection, performs its work, and returns it. This avoids repeated setup, but it also introduces a new limit: requests may wait for a pool slot when every connection is busy, trading the cost of connection setup for the cost of occasional queueing.

The pool does not remove the resource constraint. It makes the constraint explicit and controllable, turning an invisible, unbounded cost into a visible, tunable one, which is nearly always a better trade, even though it can feel, at first glance, like you have merely introduced a new problem where none existed before.

### Choosing a timeout that is not fatal

A network call cannot be allowed to wait forever. A timeout places an upper bound on how long the caller waits before treating the operation as failed. The timeout must be chosen with care. A timeout that is too short rejects slow but valid work, turning a merely sluggish dependency into a source of outright failures. A timeout that is too long ties up threads, memory, connections, and request slots, letting a handful of stuck requests slowly starve the rest of the system of the resources it needs to serve everyone else.

Timeouts also do not necessarily cancel work on the remote server. The client may stop waiting while the server continues processing the request, completely unaware that the caller has already given up. If the client retries immediately, both the original operation and the retry may be running at the same time, doubling the load on the very service that was already struggling.

This is why network reliability requires more than adding retries, and "just retry it" is one of the more dangerous pieces of folk wisdom in distributed systems when applied without thought. The system must consider idempotency, cancellation, backpressure, connection limits, and the effect of extra requests on a struggling dependency, because a naive retry policy applied during an outage can turn a partial degradation into a complete collapse.

## Why these resources never act alone

The four resources must be studied separately, but production behavior usually comes from their interaction. Almost every genuinely difficult production incident involves at least two of these resources influencing each other in a way nobody anticipated.

### When enough CPU is still not enough

A program may have enough CPU capacity but run slowly because it waits for memory accesses, spending most of its cycles stalled rather than computing. It may also use more CPU because memory pressure causes repeated work, garbage collection, decompression, or page faults, so a memory problem shows up first as an apparent CPU problem to anyone only looking at one metric.

Adding memory can improve performance when it prevents swapping or allows useful data to remain cached. It may not help when the real problem is expensive computation or a lock that serializes all work, in which case an engineer who throws more RAM at the machine will be disappointed and confused when nothing changes.

### The cache that hides your real disk

The page cache uses memory to accelerate storage. Increasing memory can reduce storage reads, but a large cache also consumes memory that other processes may need. A database buffer pool makes a similar tradeoff deliberately, sizing itself to balance how much it caches against how much memory it leaves for everything else running alongside it.

When memory pressure removes cached pages, storage traffic can increase, sometimes sharply, as reads that used to be served instantly from RAM suddenly have to go all the way to disk. When storage becomes slow, requests may remain active longer and consume more memory while waiting, holding buffers open for longer than they otherwise would. A storage problem can therefore become a memory problem, and a memory problem can become a storage problem, each one capable of triggering the other in a feedback loop that can be genuinely difficult to untangle during an incident.

### The cost of compressing or not compressing

Encryption, compression, serialization, packet processing, and TLS can consume CPU, sometimes a surprising amount of it under high enough throughput. Sending less data may reduce network time but require more CPU for compression. Sending compressed data may improve throughput while increasing latency for small responses, since the overhead of compressing a tiny payload can exceed whatever bytes it actually saves on the wire.

The best choice depends on whether the system is limited by CPU, bandwidth, latency, or the cost of handling the data, and there is no universally correct answer here. Only a correct answer for a specific workload under specific constraints, discovered by measurement rather than assumed from a rule of thumb.

### The slowest stage decides the total time

A service may read data from storage and send it across the network. The faster component does not determine the total time if another component is slower. A fast disk cannot make a distant client receive data faster than the network allows. A fast network cannot help if the storage engine takes too long to produce the response. The overall time is set by whichever stage is slowest at that moment, not by whichever stage is fastest, which is a simple point but one that is easy to forget when celebrating an upgrade to just one part of a pipeline.

### Where waiting piles up

When a resource is busy, work waits in a queue. Queues may be explicit, such as a message queue, or hidden, such as threads waiting for a lock, requests waiting for a connection, or packets waiting in a network buffer, invisible to anyone who isn't specifically looking for them.

```mermaid
flowchart LR
    Work[Incoming work] --> Q1[Request queue]
    Q1 --> CPU[CPU execution]
    CPU --> Q2[Memory or lock wait]
    Q2 --> Memory[Memory access]
    Memory --> Q3[Storage or network wait]
    Q3 --> IO[Storage or network]
    IO --> Done[Completed work]
```

Queueing increases latency, and it does so nonlinearly as a system approaches its limit. A small increase in load near saturation can produce a disproportionately large increase in wait time. If arrivals are faster than service for long enough, the queue grows until it reaches a limit. At that point, the system must reject work, drop work, delay work, or fail, and which of those four it chooses is a design decision, not something that happens automatically for the better.

This is why throughput and latency cannot be discussed independently. A system may process work at a high average rate but still have unacceptable latency when queues grow near saturation, and a dashboard that only shows the average rate will completely miss this until users start complaining.

## Busy, full, and saturated are not the same

Capacity is the amount of work a resource can handle under defined conditions. Utilization is how much of that capacity is currently being used. Saturation is the point where additional work mostly creates waiting rather than useful progress. These three terms sound related, almost interchangeable in casual conversation, but treating them as the same thing is a reliable way to misread a system's health.

Imagine a service with four workers processing requests. If all workers are busy but requests finish quickly and no queue forms, utilization is high without immediate saturation. The system is working hard, but effectively. If requests arrive faster than the workers can finish them, the queue grows and the service is becoming saturated, and the gap between these two states can be the difference between a system that feels snappy and one that feels broken, even at similar utilization numbers.

Utilization is also not always measured correctly, which is worth calling out because it undermines a lot of naive monitoring setups. CPU utilization may look low if threads are blocked on storage, since a blocked thread consumes no CPU while it waits. Storage utilization may look low while requests wait for a lock before reaching storage, meaning the storage device itself is idle even though users are waiting. Network bandwidth may be available while a remote service is overloaded, so a graph of "network usage" tells you nothing about whether the thing on the other end of that network can keep up.

Good diagnosis examines both the resource and the waiting around it, never just one or the other.

## Finding the bottleneck

A bottleneck is the part of a system that limits the overall progress of the workload. The bottleneck may move as the system changes, which is exactly what makes bottleneck-hunting an ongoing process rather than a one-time fix.

Suppose an application reads a large file and sends it to a client. At first, storage may be the bottleneck. Adding a faster disk may reveal that the network is now the bottleneck. Compressing the data may reduce network usage but make CPU the bottleneck. Increasing CPU may reveal that the client connection is slow, at which point no amount of further server-side optimization will move the needle at all.

```mermaid
flowchart LR
    Before[Storage-limited] --> FasterStorage[Faster storage]
    FasterStorage --> Middle[Network-limited]
    Middle --> Compress[Compress data]
    Compress --> After[CPU-limited]
```

This is why optimization must be measured, every single time, rather than assumed to still apply from the last time someone looked. Improving a component that is not limiting the workload may have little effect, wasting engineering time on a change that users will never notice. It may also move the bottleneck somewhere else without improving the end-to-end result, which can feel deeply unsatisfying after real effort went into the fix.

A useful investigation asks:

1. What user-visible behavior is too slow or failing?
2. Where does the request spend its time?
3. Which resources are busy, and which are waiting?
4. Are queues growing?
5. Is the resource shared with other work?
6. What happens as load increases?
7. What evidence would distinguish the possible causes?

Working through this list in order, rather than jumping straight to a guess, is usually what separates a fix that actually holds from one that only appears to help until the next traffic spike.

## A slowdown traced end to end

Imagine an API that returns a list of products. Users report that it becomes slow during a sale, and the on-call engineer opens a dashboard to figure out why.

The first assumption might be that the database is too slow. Metrics show that database CPU is only 35 percent, but the application has many requests waiting for database connections. The connection pool is too small, so requests wait before their queries even begin, a wait that never shows up in any query-latency metric, because from the database's point of view, those queries haven't started yet.

The team increases the pool size. Latency improves briefly, but the database now receives more concurrent queries than it can process. Database CPU reaches 100 percent, lock waits increase, and all requests become slower, because the bottleneck simply moved one layer over rather than disappearing.

The team investigates the query and finds that the endpoint reads more columns and rows than it needs. It adds a suitable index, reduces the result, and caches product data that changes infrequently. The final solution is not "increase the pool." It is a combination of reducing work, controlling concurrency, and reusing stable results, none of which would have been obvious from the very first metric anyone looked at.

The lesson is that a resource limit is often a symptom of a deeper mismatch between workload, capacity, and system design. Increasing a limit can help, but it can also move the overload to the next component, and knowing when you have actually fixed the problem versus merely relocated it is a skill built through exactly this kind of investigation, repeated many times over a career.

## Where to look when something breaks

The exact tools depend on the operating system, but the questions are consistent no matter which platform you are standing on.

For CPU, inspect utilization, run queues, context switches, and profiling data. For memory, inspect resident usage, allocation behavior, page faults, cache pressure, and swapping. For storage, inspect latency, throughput, queue depth, I/O errors, and filesystem capacity. For networks, inspect connection counts, latency, retransmissions, packet loss, bandwidth, and remote-service response time.

On Linux, tools such as `top`, `vmstat`, `iostat`, `pidstat`, `ss`, `lsof`, `df`, and `strace` can provide useful evidence. These tools do not automatically identify the cause, and it is worth being clear-eyed about that limitation before reaching for them mid-incident. They show different parts of the system, and the engineer must connect the observations to a hypothesis, the way a doctor connects symptoms to a diagnosis rather than treating each test result as an answer in itself.

For example, high CPU does not prove that inefficient computation is the root cause. It may be caused by a retry loop, excessive serialization, busy polling, or a cache-miss pattern, each of which points toward a completely different fix. A full disk does not prove that application data grew unexpectedly. Logs, temporary files, or deleted files still held open may be responsible, a subtlety that has caught out more than one engineer staring at `df` output in confusion.

The tool is only useful when paired with a question. Running a monitoring command without first having a hypothesis to test against it usually just produces more numbers to be confused by.

## Definitions

### What are the main resources a system manages?

> The main resources are CPU for executing work, memory for active data and program state, storage for persistent data, and the network for communication between components.

### What is a CPU-bound workload?

> A CPU-bound workload is mainly limited by the amount of processor time available to execute its instructions.

### What is a memory-bound workload?

> A memory-bound workload is mainly limited by memory capacity, memory bandwidth, or the time required to access data rather than by computation alone.

### What is the difference between storage capacity and storage performance?

> Capacity is how much data storage can hold. Performance includes how quickly it can read or write data, how long individual operations take, and how many operations it can handle concurrently.

### What is network latency?

> Network latency is the time required for communication to travel through the network and for the remote system to process the request and produce a response.

### What is a bottleneck?

> A bottleneck is the part of a system that limits the overall progress of the workload.

### What is resource contention?

> Resource contention occurs when multiple operations compete for the same limited resource, causing some of them to wait.

### What is saturation?

> Saturation occurs when a resource is busy enough that additional work mostly increases waiting instead of increasing useful throughput.

## Beyond the definitions

### How do you find whether a service is CPU-bound or I/O-bound?

> I measure where requests spend their time and compare CPU activity with the relevant I/O and wait signals. High CPU with little blocking suggests CPU-bound work, while low CPU with requests waiting on storage, network calls, locks, or queues suggests an I/O or coordination bottleneck. I would confirm the hypothesis with profiling and tracing rather than relying on one utilization number.

### Why can a system be slow when CPU usage is low?

> The system may be waiting for storage, a network dependency, a lock, a connection pool, or a queue. CPU utilization measures processor activity, not the total time a request spends waiting.

### Why does adding capacity sometimes make a system worse?

> Increasing capacity at one layer can send more work to a downstream layer that is already saturated. For example, increasing an application's database connection pool may increase database contention and make every query slower.

### Why are bounded resources useful?

> Bounds prevent one workload from consuming all available memory, connections, threads, or queue space. They force the system to choose an overload behavior such as rejection, waiting, queuing, or graceful degradation.

### What is the difference between latency and throughput?

> Latency is how long one operation takes. Throughput is how much work the system completes per unit of time. A system can have high throughput but poor latency if requests wait in large queues or batches.

## Common misconceptions

### "High CPU usage always means the system is unhealthy."

High CPU may mean the system is using its available capacity efficiently. It becomes a problem when important work waits too long, latency increases, or the system has no headroom for bursts and failures.

### "More memory always makes a system faster."

More memory can reduce swapping and keep more useful data cached, but it does not fix CPU work, lock contention, slow networks, or inefficient algorithms. Memory can also hide storage behavior during tests.

### "A fast network means network calls are cheap."

Network calls still have connection setup, serialization, encryption, routing, queueing, remote processing, timeout, and failure costs.

### "A full disk is the only storage problem."

Storage can also be a problem because of high latency, queueing, I/O errors, exhausted metadata, delayed writeback, or insufficient durability guarantees.

### "The bottleneck is always the busiest resource."

The busiest resource is a useful clue, but not proof. A low-utilization component can still be on the critical path, and a busy component may be doing useful work without limiting user-visible progress.

## Summary

CPU, memory, storage, and network are the main resources behind most software systems. CPU executes instructions, memory holds active state, storage preserves data, and the network connects components across boundaries.

Each resource has capacity, latency, throughput, limits, and failure modes. They also interact: memory pressure can increase storage traffic, storage delays can increase memory usage, network calls can consume CPU and connection capacity, and adding capacity in one layer can overload another.

The systems-engineering approach is to connect user-visible behavior to resource usage and waiting. Measure the bottleneck, understand the limit, choose an overload policy, and make the result observable. Do not optimize a resource merely because its number looks large. Understand whether it is actually limiting the work.
