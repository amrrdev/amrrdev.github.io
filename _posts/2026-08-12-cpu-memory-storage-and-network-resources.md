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

This is the second chapter in the System Engineering series. The first chapter said that systems programming is mostly about the resources behind every operation. This chapter names those resources. Almost every system you build, debug, or run uses four things to do its work: CPU time, memory, storage, and the network. Learn to spot them and learn how they limit each other. That gives you the words you need for the rest of this series.

This chapter covers a lot of ground but stays at the surface. Later chapters dig into caches, schedulers, filesystems, and protocols one topic at a time. The goal here is simpler. Learn what each resource is, what happens when it runs out, and why a slow system is often slow because of a resource you cannot see on a usage graph.

Every system uses resources to do work. Four resources show up in almost every system: CPU, memory, storage, and the network.

The CPU runs instructions. Memory holds the data and program state the CPU needs right away. Storage keeps data for a long time, even after a program stops. The network moves data between programs, machines, and services.

These resources connect to each other, but you cannot swap one for another. More CPU does not fix a slow disk. More memory does not fix a slow network call. A system can have plenty of total capacity and still be slow. The work may be stuck in a queue, a lock, a connection pool, or a service on another machine. This trips up many engineers. Every resource on a machine can look fine on a dashboard while users still see a slow, broken system. The real limit is something a simple usage graph cannot show.

The central skill in this topic is learning to ask:

> Which resource is limiting the work? How is the work waiting for that resource? What happens when the resource hits its limit?

Ask that question and answer it with evidence, not guesses. That is worth more than memorizing any number in this chapter.

## What counts as a resource

A resource is anything a system uses to do work that has a limited amount or cost. CPU time, memory space, disk bandwidth, and network bandwidth are resources. So are file descriptors, database connections, threads, queue slots, and locks. This definition is broader than the four main resources this chapter covers. Almost anything a program can run out of counts. CPU, memory, storage, and network are the four that sit under almost everything else you will learn about later. A database connection is really a network socket plus some memory at both ends. A thread is really CPU scheduling plus a block of memory for its stack.

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

Not every request uses all four the same way. A request that does heavy math may spend almost all its time on the CPU. A file download may spend more time waiting for storage or the network. A request to another service may spend most of its time waiting for that service to answer, while the local CPU does almost nothing but wait on a socket.

The diagram is not a fixed order, and you should not read it as one. A real system may read from memory before storage, use the network many times, or run many requests at once, each at a different stage. The diagram is a starting model for asking where time and capacity go. It is not a description of the exact order every request follows.

## The CPU: where instructions actually run

The CPU runs instructions. An instruction does a small job like math, a comparison, a branch, a memory read, a function call, or a sync step. Every line of code you have ever written, no matter the language, ends up as a string of these small steps.

When people say a program "uses CPU," they mean the program is using processor time instead of waiting for something else. High CPU is not always bad. Many engineers treat a high CPU number on a dashboard as a problem, but that is not right. A CPU running near full may mean the system is using its hardware well and getting value from it. The problem starts when work cannot finish in time, tasks wait too long for CPU time, or there is no room left for important work like a sudden spike in traffic or a background job.

### What a core can really do

CPU capacity depends on more than the number of cores. It also depends on clock speed, instruction cost, cache behavior, branch prediction, memory latency, and the work being done. Two machines with the same core count and the same clock speed can run the same job very differently. The difference may come only from cache sizes or how predictable the job's branches are.

A core can only run a limited amount of work at once. If more work is ready than the cores can run, the operating system hands out CPU time over time. Some tasks run while others wait. They take turns in slices of a millisecond or less.

```text
Available CPU capacity
    = number of usable cores
    × effective work per core
    × time available
```

This is a simplified model. Treat it as a way to think, not as a formula you would actually compute. The useful work per core changes with the mix of instructions, cache misses, branch behavior, CPU frequency, and other activity on the machine. A core running tight, cache-friendly math code can do several times the work of the same core running code that jumps around memory unpredictably. Both may show as "using 100% CPU" on a monitoring tool.

### When the CPU is the limit

A workload is CPU-bound when the CPU is the main thing holding it back. Examples include compression, encryption, image processing, parsing, compilation, and math. In each case the limit is "how many instructions can run," not "how long until something else answers."

If you give a CPU-bound program more CPU and the work spreads well across cores, it may go faster. If the program uses only one thread, adding cores may not help at all. A single thread can only run on one core, no matter how many sit idle. If threads share a lock or fight for memory bandwidth, adding more threads can make the program slower. The cost of coordination grows past the gain from running in parallel.

This is why "use more threads" is not a fix for everything. Treating it as one is a common beginner mistake in systems work. Threads still need CPU time. The cost of coordinating them only shows up when you run the program under real concurrency.

### Busy is not the same as slow

A service can have low CPU use and still be slow. It may be waiting for storage, a network call, a lock, or a queue, while the CPU sits idle the whole time a user watches a spinner. On the other hand, a service can have high CPU use and still respond quickly if the work is predictable and there is enough capacity to handle it without queueing.

CPU use answers "how busy is the processor?" It does not answer "how long is a request waiting?" Both questions matter. Mixing them up is a fast way to misdiagnose a performance problem. An engineer who sees low CPU and says "the CPU isn't the problem, so let's look elsewhere" is usually right. An engineer who sees low CPU and says "nothing is wrong" stopped one step too early.

### Who decides what runs next

The operating system scheduler decides which ready thread gets CPU time. It tries to share the CPU by priority and policy. But no task runs straight from start to finish. Tasks are interleaved, sometimes so finely that only a tracing tool can see it.

A context switch happens when the CPU stops one thread and starts another. Context switches are needed to share the CPU. But they require saving and restoring state, and they may hurt cache locality. The new thread's data is unlikely to already be in the cache the old thread used. Too many threads, too many ready tasks, or heavy synchronization can raise scheduling overhead. The machine may spend a real part of its time switching between tasks instead of doing them.

Later articles cover scheduling and context switches in more detail. The key point here is that CPU is a shared resource, and waiting for CPU is not the same as waiting for I/O. A thread stuck behind a busy CPU and a thread waiting on a slow disk look very different to the operating system. Both just appear as "not running" to someone looking at a process list.

## Memory: what the machine keeps close

Memory, usually called RAM, holds the instructions and data that running programs need. The CPU can reach memory much faster than storage. But memory is limited and loses its contents when the power goes off. That tradeoff is what separates memory from storage.

Memory holds more than your application objects. A process also needs memory for its code, stacks, heaps, shared libraries, runtime structures, buffers, caches, and operating-system bookkeeping. A program that "only" allocates a few objects on the heap still uses memory for all those other structures, even if you do not see them.

### Why memory stalls look like CPU problems

When a program needs data, that data must be reachable through a memory path before the CPU can use it. If the data is not in a nearby cache, the CPU waits longer. On a modern processor this can waste hundreds of cycles. If the data is not mapped into the program's usable memory, the operating system must handle a page fault, which is an even more costly interruption. If the system is short on memory, it may reclaim pages or move data to storage. That is slower by several orders of magnitude.

These events cost different amounts, but they share one idea. Memory behavior affects how fast the CPU gets work done. From the outside, both a cache miss and a page fault just look like "the program is running slowly."

### Capacity, bandwidth, and latency are separate limits

Memory capacity is how much data can be held. Memory bandwidth is how fast data can move. Memory latency is how long it takes to start getting data after you ask for it. It is easy to mix these up. But a system can be limited by any one of them on its own.

A workload may have enough memory capacity but still be limited by memory bandwidth. For example, a program that scans a very large array again and again may spend more time moving data than doing math. Adding more RAM does nothing to speed it up, because capacity was never the limit.

Another workload may fit in memory but run slowly because it reads data in random spots and misses the CPU caches often. This is why the amount of memory alone does not describe memory performance. Two programs using the same number of megabytes can run at very different speeds based only on how they access the data.

### When memory runs out

Memory pressure happens when the memory in use gets close to or passes what the system has. The operating system may reclaim unused cache pages, compress memory, swap pages to storage, or kill a process when it cannot meet the demand. The exact response depends on the operating system and its settings. But the situation is always the same: more is being asked for than exists.

Swapping means moving memory pages between RAM and storage to free RAM for other work. Storage is much slower than RAM. Heavy swapping can make a system spend most of its time moving pages instead of doing useful work. This is called thrashing. It is one of the more dramatic failures in systems work. A machine under heavy thrashing can look almost completely dead even though it is technically still running.

Memory pressure can also cause indirect failures that are hard to trace back to their cause. A process may spend more time waiting for page faults. A garbage collector in a managed runtime may run more often and steal CPU time from useful work. The operating system may kill a process outright, often the one using the most memory right then, whether or not it caused the pressure. A cache may drop useful data and force repeated reads from a slower layer. That quietly turns a memory problem into a storage problem downstream.

### Who is responsible for returning memory

At the application level, memory may be owned by a data structure, a request, a thread, a process, or a cache. At the operating-system level, memory is tracked through address spaces and pages. This is a lower-level bookkeeping system that application code rarely sees directly.

Ownership matters because memory that is no longer needed must become reusable. That does not happen just because the code stopped using a value. A memory leak happens when a program keeps references to memory it no longer needs, so the memory cannot be reused. This can happen even in languages with garbage collection, where a leak just means "still reachable, but no longer needed." A cache that grows without a limit is a form of resource failure even if every cached object is valid and correctly referenced. Valid is not the same as needed.

A later article on virtual memory will explain how each process gets an address space and how the operating system maps that space to physical memory. It fills in the mechanics this section leaves out on purpose.

## Storage: keeping data past a restart

Storage keeps data past the life of a process and usually past a machine restart. Examples include hard drives, SSDs, NVMe devices, distributed filesystems, and object-storage services. These range from a single spinning disk in a laptop to a storage service copied across the world.

Storage has several properties that you must look at separately. Mixing them up is a common source of confusion. Engineers often talk about "storage performance" as if it were one number:

- Capacity: how much data can be stored
- Throughput: how much data can be read or written per second
- Latency: how long one operation takes
- IOPS: how many individual input/output operations can be completed per second
- Durability: whether acknowledged data survives failures
- Availability: whether the data can be accessed when needed

A storage device can have high throughput and still have poor latency for small random reads. This is exactly what happens with spinning disks doing random access instead of reading in sequence. A device can have low latency for cached data while a cache miss needs a much slower operation. So the same device can feel instant or sluggish based only on what was read moments before. It can have enough capacity today and grow past its limit next month, quietly, with no alert until the day it actually fills.

### Why storage is not just slow memory

Storage and memory both hold data, but they play different roles. Memory is built for fast access by running programs. Storage is built to keep data and to hold more of it. Its access latency is usually higher, often by several orders of magnitude compared to RAM.

The operating system often makes storage look more like memory through the page cache. It may keep recently used file data in memory, so reading the same file again is much faster. This can make a program look fast during a test but behave differently after a restart or under memory pressure. Then the cache is empty and every read must go back to the physical device.

The page cache is useful, but it can hide the real storage behavior right when you most need to see it. A benchmark that reads the same small file over and over may measure memory, not the storage device. That gives a falsely hopeful picture of how the system will perform once real users read files never touched before.

### What it means for a write to be durable

Durability answers whether data stays available after a failure such as a process crash, operating-system crash, power loss, or device failure. It is a narrower and more precise question than "did the write succeed." Many data-loss incidents happen in the gap between those two questions.

An application may write data into a process buffer or the operating-system page cache and get a success back before the data is truly durable on the device. The exact guarantees depend on the filesystem, storage device, operating system, and flush operations. A program that never checks these guarantees is trusting an assumption it never verified.

This is why databases use techniques like write-ahead logging and explicit synchronization. They need a clear boundary between "the program asked for the write" and "the system can recover the write after a crash." Building that boundary correctly is one of the harder and more important jobs in the whole storage stack. Later articles on storage engines cover it in depth.

### All the ways storage can fail without filling up

Storage pressure is not just "the disk is full." It can also mean the device is too slow, I/O queues are full, writeback is delayed, or a filesystem has run out of inodes or other metadata space. This last failure is subtle. There may be plenty of free bytes, but no room to create a single new file because every inode slot is already used.

When storage fills, normal operations may fail in surprising places that have nothing obvious to do with disk space. A service may fail to write logs, create temporary files, update a database, or create a socket state file. Treat a full disk as a failure mode you plan for, not an impossible event. In a long-running production system, it is closer to a certainty than an edge case.

## The network: when one machine talks to another

The network moves data between processes, machines, services, regions, and sometimes continents. It adds a boundary that is different from a function call inside one process. This is not just a difference in size. It is a difference in kind, and it shows up again and again throughout this series.

Network communication can be delayed, lost, duplicated, reordered, rejected, or cut off. Even when you use a reliable transport like TCP, the application must still handle connection setup, timeouts, remote process failure, and the chance that a request finished even though its response never arrived. A local function call can never do this. A function call either runs or it does not. It cannot half-finish on the other side of a wire that was cut.

### What you spend to send data

Network capacity includes bandwidth, packet-processing capacity, connection capacity, and the ability of the receiving service to process incoming work. Any one of these four can be a limit on its own. A system can be held back by one while the others are far from their limit.

Bandwidth is the amount of data that can move over a period. Latency is the time for data to travel and be processed. A high-bandwidth link can still have high latency. A satellite link can move huge amounts of data per second yet still take hundreds of milliseconds for the first byte to arrive. A low-latency link can still get congested when too much data is sent. A near-instant local network can still choke under enough simultaneous traffic.

```text
Network request time
    = connection setup
    + request transmission
    + server queueing
    + server processing
    + response transmission
```

This is a simplified model, but it shows why network latency is not just the physical distance between two machines. Queueing, encryption, routing, server load, and application processing all add time. Any one of them can dominate the total depending on the situation. A request across the world to an idle server can sometimes be faster than a request to an overloaded server in the next rack.

### Connections cost something

A connection uses state in the client, the server, the operating system, and sometimes load balancers or firewalls along the path. A service that opens a new connection for every request may spend a lot of time on setup and may hit connection limits. This matters most during the kind of traffic spike that makes the problem worse.

A connection pool keeps a fixed number of open connections ready to reuse. Each request borrows a connection, does its work, and returns it. This avoids repeated setup. But it adds a new limit. Requests may wait for a pool slot when every connection is busy. You trade the cost of setup for the cost of occasional queueing.

The pool does not remove the resource limit. It makes the limit clear and controllable. It turns an invisible, unbounded cost into a visible, tunable one. That is almost always a better trade, even though at first glance it can feel like you added a new problem where none existed before.

### Choosing a timeout that is not fatal

A network call cannot wait forever. A timeout sets an upper bound on how long the caller waits before it treats the operation as failed. Choose the timeout with care. A timeout that is too short rejects slow but valid work. That turns a merely sluggish dependency into a source of real failures. A timeout that is too long ties up threads, memory, connections, and request slots. A handful of stuck requests can slowly starve the rest of the system of the resources it needs for everyone else.

A timeout also does not necessarily cancel work on the remote server. The client may stop waiting while the server keeps processing the request, unaware that the caller already gave up. If the client retries right away, the original operation and the retry may both run at once. That doubles the load on the very service that was already struggling.

This is why network reliability needs more than retries. "Just retry it" is dangerous folk wisdom in distributed systems when applied without thought. The system must consider idempotency, cancellation, backpressure, connection limits, and the effect of extra requests on a struggling dependency. A naive retry policy during an outage can turn a partial slowdown into a full collapse.

## Why these resources never act alone

You must study the four resources separately, but in production they act together. Almost every truly difficult incident involves at least two of these resources affecting each other in a way nobody expected.

### When enough CPU is still not enough

A program may have enough CPU but still run slowly because it waits for memory accesses. It spends most of its cycles stalled rather than computing. It may also use more CPU because memory pressure causes repeated work, garbage collection, decompression, or page faults. So a memory problem first shows up as a CPU problem to anyone watching only one metric.

Adding memory can improve performance when it stops swapping or keeps useful data cached. It may not help when the real problem is costly computation or a lock that serializes all work. In that case, an engineer who throws more RAM at the machine will be disappointed and confused when nothing changes.

### The cache that hides your real disk

The page cache uses memory to speed up storage. Adding memory can cut storage reads. But a large cache also uses memory that other processes may need. A database buffer pool makes a similar trade on purpose. It sizes itself to balance how much it caches against how much memory it leaves for everything else running next to it.

When memory pressure drops cached pages, storage traffic can rise, sometimes sharply. Reads that used to come instantly from RAM now have to go to disk. When storage slows, requests may stay active longer and use more memory while waiting, holding buffers open longer than they should. A storage problem can become a memory problem, and a memory problem can become a storage problem. Each can trigger the other in a loop that is hard to untangle during an incident.

### The cost of compressing or not compressing

Encryption, compression, serialization, packet processing, and TLS can use CPU. Under high enough throughput, the amount can be surprising. Sending less data may cut network time but cost more CPU for compression. Sending compressed data may improve throughput while raising latency for small responses. The overhead of compressing a tiny payload can exceed the bytes it saves on the wire.

The best choice depends on whether the system is limited by CPU, bandwidth, latency, or the cost of handling the data. There is no one right answer. There is only a right answer for a specific workload under specific constraints, found by measurement rather than guessed from a rule of thumb.

### The slowest stage decides the total time

A service may read data from storage and send it across the network. The faster component does not decide the total time when another component is slower. A fast disk cannot make a distant client receive data faster than the network allows. A fast network cannot help if the storage engine takes too long to build the response. The total time is set by the slowest stage at that moment, not the fastest. This is simple, but easy to forget when you celebrate an upgrade to just one part of a pipeline.

### Where waiting piles up

When a resource is busy, work waits in a queue. Queues may be explicit, like a message queue. Or they may be hidden, like threads waiting for a lock, requests waiting for a connection, or packets waiting in a network buffer. These are invisible to anyone not specifically looking for them.

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

Queueing raises latency, and it does so nonlinearly as a system nears its limit. A small rise in load near saturation can cause a large rise in wait time. If arrivals outpace service long enough, the queue grows until it hits a limit. At that point the system must reject work, drop work, delay work, or fail. Which of those four it picks is a design decision, not something that happens automatically for the better.

This is why you cannot talk about throughput and latency separately. A system may process work at a high average rate yet still have unacceptable latency when queues grow near saturation. A dashboard that only shows the average rate will miss this until users complain.

## Busy, full, and saturated are not the same

Capacity is the amount of work a resource can handle under defined conditions. Utilization is how much of that capacity is in use right now. Saturation is the point where extra work mostly creates waiting instead of useful progress. These three terms sound related and almost interchangeable in casual talk. But treating them as the same thing is a reliable way to misread a system's health.

Imagine a service with four workers processing requests. If all workers are busy but requests finish fast and no queue forms, utilization is high without immediate saturation. The system is working hard but effectively. If requests arrive faster than the workers can finish them, the queue grows and the service becomes saturated. The gap between these two states can be the difference between a snappy system and a broken one, even at similar utilization numbers.

Utilization is also not always measured correctly. This matters because it undermines many naive monitoring setups. CPU utilization may look low if threads are blocked on storage, since a blocked thread uses no CPU while it waits. Storage utilization may look low while requests wait for a lock before reaching storage. The storage device itself is idle even though users are waiting. Network bandwidth may be available while a remote service is overloaded. So a graph of "network usage" tells you nothing about whether the thing on the other end can keep up.

Good diagnosis looks at both the resource and the waiting around it. Never just one or the other.

## Finding the bottleneck

A bottleneck is the part of a system that limits the overall progress of the workload. The bottleneck may move as the system changes. That is exactly why finding bottlenecks is an ongoing process, not a one-time fix.

Suppose an application reads a large file and sends it to a client. At first, storage may be the bottleneck. Adding a faster disk may show that the network is now the bottleneck. Compressing the data may cut network usage but make CPU the bottleneck. Increasing CPU may show that the client connection is slow. At that point no amount of further server-side optimization will help.

```mermaid
flowchart LR
    Before[Storage-limited] --> FasterStorage[Faster storage]
    FasterStorage --> Middle[Network-limited]
    Middle --> Compress[Compress data]
    Compress --> After[CPU-limited]
```

This is why you must measure optimization every single time, rather than assume it still applies from the last time someone looked. Improving a component that is not the limit may have little effect. You waste engineering time on a change users will never notice. It may also move the bottleneck somewhere else without improving the end-to-end result. That feels deeply unsatisfying after real effort went into the fix.

A useful investigation asks:

1. What user-visible behavior is too slow or failing?
2. Where does the request spend its time?
3. Which resources are busy, and which are waiting?
4. Are queues growing?
5. Is the resource shared with other work?
6. What happens as load increases?
7. What evidence would distinguish the possible causes?

Work through this list in order instead of jumping to a guess. That is usually what separates a fix that actually holds from one that only seems to help until the next traffic spike.

## A slowdown traced end to end

Imagine an API that returns a list of products. Users say it gets slow during a sale. The on-call engineer opens a dashboard to find out why.

The first guess might be that the database is too slow. Metrics show database CPU is only 35 percent. But the application has many requests waiting for database connections. The connection pool is too small, so requests wait before their queries even begin. That wait never shows up in any query-latency metric, because from the database's view those queries have not started yet.

The team increases the pool size. Latency improves briefly. But the database now gets more concurrent queries than it can process. Database CPU hits 100 percent, lock waits rise, and all requests get slower. The bottleneck just moved one layer over instead of going away.

The team investigates the query and finds that the endpoint reads more columns and rows than it needs. It adds a suitable index, reduces the result, and caches product data that changes rarely. The final solution is not "increase the pool." It is a mix of reducing work, controlling concurrency, and reusing stable results. None of that would have been obvious from the very first metric anyone looked at.

The lesson is that a resource limit is often a symptom of a deeper mismatch between workload, capacity, and system design. Raising a limit can help. But it can also move the overload to the next component. Knowing when you have actually fixed the problem versus merely relocated it is a skill built through exactly this kind of investigation, repeated many times over a career.

## Where to look when something breaks

The exact tools depend on the operating system. But the questions stay the same no matter which platform you are on.

For CPU, look at utilization, run queues, context switches, and profiling data. For memory, look at resident usage, allocation behavior, page faults, cache pressure, and swapping. For storage, look at latency, throughput, queue depth, I/O errors, and filesystem capacity. For networks, look at connection counts, latency, retransmissions, packet loss, bandwidth, and remote-service response time.

On Linux, tools like `top`, `vmstat`, `iostat`, `pidstat`, `ss`, `lsof`, `df`, and `strace` can give useful evidence. These tools do not identify the cause on their own. Be clear about that limit before you reach for them mid-incident. They show different parts of the system. The engineer must connect the observations to a hypothesis. A doctor connects symptoms to a diagnosis rather than treating each test result as the answer.

For example, high CPU does not prove that inefficient computation is the root cause. It may come from a retry loop, excessive serialization, busy polling, or a cache-miss pattern. Each points to a different fix. A full disk does not prove that application data grew unexpectedly. Logs, temporary files, or deleted files still held open may be the cause. This subtlety has caught out more than one engineer staring at `df` output in confusion.

The tool is only useful with a question attached. Running a monitoring command without a hypothesis to test usually just produces more numbers to be confused by.

## Definitions

### What are the main resources a system manages?

> The main resources are CPU for running work, memory for active data and program state, storage for data that lasts, and the network for communication between components.

### What is a CPU-bound workload?

> A CPU-bound workload is mainly limited by how much processor time it has to run its instructions.

### What is a memory-bound workload?

> A memory-bound workload is mainly limited by memory capacity, memory bandwidth, or the time needed to reach data, rather than by computation alone.

### What is the difference between storage capacity and storage performance?

> Capacity is how much data storage can hold. Performance is how fast it can read or write, how long each operation takes, and how many operations it can do at once.

### What is network latency?

> Network latency is the time for a message to travel through the network and for the remote system to process the request and produce a response.

### What is a bottleneck?

> A bottleneck is the part of a system that limits the overall progress of the workload.

### What is resource contention?

> Resource contention happens when several operations compete for the same limited resource, so some of them must wait.

### What is saturation?

> Saturation happens when a resource is busy enough that extra work mostly adds waiting instead of useful work.

## Beyond the definitions

### How do you find whether a service is CPU-bound or I/O-bound?

> I measure where requests spend their time and compare CPU activity with the relevant I/O and wait signals. High CPU with little blocking suggests CPU-bound work. Low CPU with requests waiting on storage, network calls, locks, or queues suggests an I/O or coordination bottleneck. I confirm the hypothesis with profiling and tracing rather than relying on one utilization number.

### Why can a system be slow when CPU usage is low?

> The system may be waiting for storage, a network dependency, a lock, a connection pool, or a queue. CPU utilization measures processor activity, not the total time a request spends waiting.

### Why does adding capacity sometimes make a system worse?

> Adding capacity at one layer can send more work to a downstream layer that is already saturated. For example, increasing an application's database connection pool may raise database contention and make every query slower.

### Why are bounded resources useful?

> Bounds stop one workload from using all the memory, connections, threads, or queue space. They force the system to pick an overload behavior, such as rejection, waiting, queueing, or graceful degradation.

### What is the difference between latency and throughput?

> Latency is how long one operation takes. Throughput is how much work the system completes per unit of time. A system can have high throughput but poor latency if requests wait in large queues or batches.

## Common misconceptions

### "High CPU usage always means the system is unhealthy."

High CPU may mean the system is using its capacity well. It becomes a problem when important work waits too long, latency rises, or the system has no room left for bursts and failures.

### "More memory always makes a system faster."

More memory can cut swapping and keep more useful data cached. But it does not fix CPU work, lock contention, slow networks, or inefficient algorithms. Memory can also hide storage behavior during tests.

### "A fast network means network calls are cheap."

Network calls still carry costs: connection setup, serialization, encryption, routing, queueing, remote processing, timeouts, and failures.

### "A full disk is the only storage problem."

Storage can also be a problem because of high latency, queueing, I/O errors, exhausted metadata, delayed writeback, or weak durability guarantees.

### "The bottleneck is always the busiest resource."

The busiest resource is a useful clue, but not proof. A low-utilization component can still be on the critical path. A busy component may be doing useful work without limiting user-visible progress.

## Summary

CPU, memory, storage, and network are the main resources behind most software systems. The CPU runs instructions, memory holds active state, storage keeps data, and the network connects components across boundaries.

Each resource has capacity, latency, throughput, limits, and failure modes. They also interact. Memory pressure can raise storage traffic. Storage delays can raise memory usage. Network calls can use CPU and connection capacity. Adding capacity in one layer can overload another.

The systems-engineering approach is to connect user-visible behavior to resource usage and waiting. Measure the bottleneck, understand the limit, choose an overload policy, and make the result observable. Do not optimize a resource merely because its number looks large. Understand whether it is actually limiting the work.
