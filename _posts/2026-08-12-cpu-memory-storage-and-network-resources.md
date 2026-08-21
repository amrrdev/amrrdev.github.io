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

> Stage 1 - Systems Programming Foundations  
> Subject area 1.1 - What Systems Programming Means  
> Article 2

## The short version

Every system spends resources to do work. The four resources that appear in almost every system are CPU, memory, storage, and the network.

The CPU performs instructions. Memory holds the data and program state that the CPU needs quickly. Storage keeps data for longer periods, even after a process stops. The network moves data between processes, machines, and services.

These resources are connected, but they are not interchangeable. More CPU does not fix a slow disk. More memory does not automatically fix a slow network dependency. A system can also have enough total capacity and still be slow because work is waiting in a queue, a lock, a connection pool, or a remote service.

The central skill in this topic is learning to ask:

> Which resource is limiting the work, how is the work waiting for it, and what happens when the resource reaches its limit?

## Where this article fits

The previous article introduced systems programming as the work of managing resources and operating across boundaries. This article names the main resources and gives us a practical way to reason about them.

Later articles will explain each resource in much more detail. CPU scheduling and caches will explain how execution time is shared. Virtual memory will explain how memory is mapped and protected. Filesystems will explain storage interfaces. Networking articles will explain how data moves between machines.

For now, we are building the map before studying every road.

## What is a resource?

A resource is something a system consumes to perform work and that has a limited capacity or cost. CPU time, memory space, disk bandwidth, and network bandwidth are resources, but so are file descriptors, database connections, threads, queue slots, and locks.

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

Not every request uses all four resources in the same way. A computation-heavy request may spend almost all its time on the CPU. A file download may spend more time waiting for storage or the network. A request to another service may spend most of its time waiting for the remote service to respond.

The diagram is not a fixed sequence. A real system may read from memory before storage, use the network several times, or process many operations concurrently. It is a starting model for asking where time and capacity go.

## CPU: the resource that executes work

The CPU executes instructions. Instructions perform operations such as arithmetic, comparisons, branches, memory accesses, function calls, and synchronization.

When people say that a program “uses CPU,” they usually mean that the program is actively consuming processor time instead of waiting for another resource. CPU usage is not automatically bad. A CPU running at high utilization may mean the system is using its hardware efficiently. It becomes a problem when work cannot finish within its required latency, tasks wait too long for CPU time, or there is no capacity left for important work.

### CPU capacity

CPU capacity depends on more than the number of cores. It also depends on clock speed, instruction cost, cache behavior, branch prediction, memory latency, and the work being performed.

A core can execute only a limited amount of work at a time. If more runnable work exists than the available cores can execute, the operating system schedules that work over time. Some tasks run while others wait.

```text
Available CPU capacity
    = number of usable cores
    × effective work per core
    × time available
```

This is a simplified model. Effective work per core changes with the instruction mix, cache misses, branch behavior, CPU frequency, and other activity on the machine.

### CPU-bound work

A workload is CPU-bound when the CPU is the main limit on how quickly it can progress. Examples include compression, encryption, image processing, parsing, compilation, and numerical calculations.

If a CPU-bound program receives more CPU capacity and the work scales well across cores, its throughput may improve. If the program is single-threaded, adding more cores may not help. If threads share a lock or compete for memory bandwidth, adding more threads may make the program slower.

This is why “use more threads” is not a universal performance solution. Threads still need CPU time, and coordination between threads has a cost.

### CPU waiting is different from CPU usage

A service can have low CPU utilization and still be slow. It may be waiting for storage, a network dependency, a lock, or a queue. Conversely, a service can have high CPU utilization and still have good latency if the work is predictable and there is enough capacity.

CPU utilization answers “how busy is the processor?” It does not answer “how long is a request waiting?” Both questions matter.

### CPU scheduling and fairness

The operating system scheduler decides which runnable thread receives CPU time. It tries to share the CPU according to priorities and scheduling policies, but the result is not that every task runs continuously.

A context switch occurs when the CPU stops running one thread and starts running another. Context switches are necessary for sharing the CPU, but they require saving and restoring execution state and may reduce cache locality. Excessive thread creation, too many runnable tasks, or heavy synchronization can increase scheduling overhead.

The detailed behavior of scheduling and context switches belongs in later articles. The important point here is that CPU is a shared resource, and waiting for CPU is different from waiting for I/O.

## Memory: the resource that holds active state

Memory, usually referring to RAM in this context, holds the instructions and data that active programs need. The CPU can access memory much faster than storage, but memory is limited and loses its contents when power is removed.

Memory is used for more than application objects. A process also needs memory for its code, stacks, heaps, shared libraries, runtime structures, buffers, caches, and operating-system bookkeeping.

### Why memory matters

When a program needs data, the data must be available through a memory path before the CPU can work with it. If the data is not in a nearby cache, the CPU waits longer. If it is not mapped in the process's usable memory, the operating system may need to handle a page fault. If the system is under memory pressure, it may reclaim pages or move data to storage.

These events have different costs, but they all show the same principle: memory behavior affects CPU progress.

### Memory capacity and memory bandwidth

Memory capacity is how much data can be held. Memory bandwidth is how quickly data can be transferred. Memory latency is how long it takes to begin receiving data after requesting it.

A workload may have enough memory capacity but still be limited by memory bandwidth. For example, a program that scans a very large array repeatedly may spend more time moving data than performing calculations.

Another workload may fit comfortably in total memory but perform poorly because it accesses data randomly and misses the CPU caches frequently. This is why the amount of memory alone does not describe memory performance.

### Memory pressure

Memory pressure occurs when active demand approaches or exceeds the memory available to the system. The operating system may reclaim unused cache pages, compress memory, swap pages to storage, or terminate a process when it cannot satisfy the demand.

Swapping means moving memory pages between RAM and storage to free RAM for other work. Storage is much slower than RAM, so heavy swapping can cause a system to spend most of its time moving pages instead of doing useful work. This condition is called thrashing.

Memory pressure can also create indirect failures. A process may spend more time waiting for page faults. The garbage collector of a managed runtime may run more often. The operating system may kill a process. A cache may evict useful data and force repeated reads from a slower layer.

### Memory ownership

At the application level, memory may be owned by a data structure, request, thread, process, or cache. At the operating-system level, memory is associated with address spaces and pages.

Ownership matters because memory that is no longer needed must become reclaimable. A memory leak occurs when a program keeps references to memory that it no longer needs, preventing that memory from being reused. A cache that grows without a bound is a form of resource-management failure even if every cached object is technically valid.

The virtual-memory article later will explain how each process receives an address space and how the operating system maps that address space to physical memory.

## Storage: the resource that keeps data

Storage keeps data beyond the lifetime of a process and usually beyond a machine restart. Examples include hard drives, SSDs, NVMe devices, distributed filesystems, and object-storage services.

Storage has several properties that must be considered separately:

- Capacity: how much data can be stored
- Throughput: how much data can be read or written per second
- Latency: how long one operation takes
- IOPS: how many individual input/output operations can be completed per second
- Durability: whether acknowledged data survives failures
- Availability: whether the data can be accessed when needed

A storage device can have high throughput and still have poor latency for small random reads. It can have low latency for cached data while a cache miss requires a much slower device operation. It can have enough capacity today but grow beyond its limit next month.

### Storage is not just a large memory

Storage and memory both hold data, but they serve different roles. Memory is designed for fast access by active programs. Storage is designed to retain data and provide more capacity, usually with higher access latency.

The operating system often makes storage appear more memory-like through the page cache. Recently used file data may be kept in memory, so reading the same file again can be much faster. This can make a program appear fast during a test while behaving differently after a restart or under memory pressure.

The page cache is useful, but it can also hide the real storage behavior. A benchmark that reads the same small file repeatedly may measure memory rather than the storage device.

### Storage durability

Durability answers whether data remains available after a failure such as a process crash, operating-system crash, power loss, or device failure.

An application may write data into a process buffer or the operating-system page cache and receive a successful return before the data is physically durable on the device. The exact guarantees depend on the filesystem, storage device, operating system, and flush operations.

This is why databases use techniques such as write-ahead logging and explicit synchronization. They need a reliable boundary that separates “the program requested the write” from “the system can recover the write after a crash.”

### Storage pressure

Storage pressure is not only “the disk is full.” It can also mean that the device is too slow, I/O queues are full, writeback is delayed, or a filesystem has run out of inodes or other metadata capacity.

When storage fills, normal operations may fail in surprising places. A service may be unable to write logs, create temporary files, update a database, or create a socket-related state file. Disk-full conditions should be treated as a planned failure mode, not an impossible event.

## Network: the resource that connects systems

The network moves data between processes, machines, services, regions, and sometimes continents. It introduces a boundary that is different from a function call inside one process.

Network communication can be delayed, lost, duplicated, reordered, rejected, or interrupted. Even when a reliable transport such as TCP is used, the application still has to handle connection establishment, timeouts, remote process failure, and the possibility that a request completed even though its response did not arrive.

### Network capacity

Network capacity includes bandwidth, packet-processing capacity, connection capacity, and the ability of the receiving service to process incoming work.

Bandwidth is the amount of data that can be transferred over a period. Latency is the time taken for data to travel and be processed. A high-bandwidth link can still have high latency. A low-latency link can still become congested when too much data is sent.

```text
Network request time
    = connection setup
    + request transmission
    + server queueing
    + server processing
    + response transmission
```

This is a simplified model, but it shows why network latency is not only the physical distance between two machines. Queueing, encryption, routing, server load, and application processing all contribute.

### Network connections are resources

A connection consumes state in the client, the server, the operating system, and sometimes load balancers or firewalls. A service that creates a new connection for every request may spend significant time on setup and may exhaust connection limits.

A connection pool keeps a bounded number of established connections available for reuse. Each request borrows a connection, performs its work, and returns it. This avoids repeated setup, but it also introduces a new limit: requests may wait for a pool slot when every connection is busy.

The pool does not remove the resource constraint. It makes the constraint explicit and controllable.

### Network failure and timeouts

A network call cannot be allowed to wait forever. A timeout places an upper bound on how long the caller waits before treating the operation as failed. The timeout must be chosen with care. A timeout that is too short rejects slow but valid work. A timeout that is too long ties up threads, memory, connections, and request slots.

Timeouts also do not necessarily cancel work on the remote server. The client may stop waiting while the server continues processing the request. If the client retries immediately, both the original operation and the retry may be running.

This is why network reliability requires more than adding retries. The system must consider idempotency, cancellation, backpressure, connection limits, and the effect of extra requests on a struggling dependency.

## The resources interact

The four resources must be studied separately, but production behavior usually comes from their interaction.

### CPU and memory

A program may have enough CPU capacity but run slowly because it waits for memory accesses. It may also use more CPU because memory pressure causes repeated work, garbage collection, decompression, or page faults.

Adding memory can improve performance when it prevents swapping or allows useful data to remain cached. It may not help when the real problem is expensive computation or a lock that serializes all work.

### Memory and storage

The page cache uses memory to accelerate storage. Increasing memory can reduce storage reads, but a large cache also consumes memory that other processes may need. A database buffer pool makes a similar tradeoff deliberately.

When memory pressure removes cached pages, storage traffic can increase. When storage becomes slow, requests may remain active longer and consume more memory while waiting. A storage problem can therefore become a memory problem.

### CPU and network

Encryption, compression, serialization, packet processing, and TLS can consume CPU. Sending less data may reduce network time but require more CPU for compression. Sending compressed data may improve throughput while increasing latency for small responses.

The best choice depends on whether the system is limited by CPU, bandwidth, latency, or the cost of handling the data.

### Network and storage

A service may read data from storage and send it across the network. The faster component does not determine the total time if another component is slower. A fast disk cannot make a distant client receive data faster than the network allows. A fast network cannot help if the storage engine takes too long to produce the response.

### Queues connect resource limits

When a resource is busy, work waits in a queue. Queues may be explicit, such as a message queue, or hidden, such as threads waiting for a lock, requests waiting for a connection, or packets waiting in a network buffer.

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

Queueing increases latency. If arrivals are faster than service for long enough, the queue grows until it reaches a limit. At that point, the system must reject work, drop work, delay work, or fail.

This is why throughput and latency cannot be discussed independently. A system may process work at a high average rate but still have unacceptable latency when queues grow near saturation.

## Capacity, utilization, and saturation

Capacity is the amount of work a resource can handle under defined conditions. Utilization is how much of that capacity is currently being used. Saturation is the point where additional work mostly creates waiting rather than useful progress.

These terms are related but not identical.

Imagine a service with four workers processing requests. If all workers are busy but requests finish quickly and no queue forms, utilization is high without immediate saturation. If requests arrive faster than the workers can finish them, the queue grows and the service is becoming saturated.

Utilization is also not always measured correctly. CPU utilization may look low if threads are blocked on storage. Storage utilization may look low while requests wait for a lock before reaching storage. Network bandwidth may be available while a remote service is overloaded.

Good diagnosis examines both the resource and the waiting around it.

## Finding the bottleneck

A bottleneck is the part of a system that limits the overall progress of the workload. The bottleneck may move as the system changes.

Suppose an application reads a large file and sends it to a client. At first, storage may be the bottleneck. Adding a faster disk may reveal that the network is now the bottleneck. Compressing the data may reduce network usage but make CPU the bottleneck. Increasing CPU may reveal that the client connection is slow.

```mermaid
flowchart LR
    Before[Storage-limited] --> FasterStorage[Faster storage]
    FasterStorage --> Middle[Network-limited]
    Middle --> Compress[Compress data]
    Compress --> After[CPU-limited]
```

This is why optimization must be measured. Improving a component that is not limiting the workload may have little effect. It may also move the bottleneck somewhere else without improving the end-to-end result.

A useful investigation asks:

1. What user-visible behavior is too slow or failing?
2. Where does the request spend its time?
3. Which resources are busy, and which are waiting?
4. Are queues growing?
5. Is the resource shared with other work?
6. What happens as load increases?
7. What evidence would distinguish the possible causes?

## A realistic production example

Imagine an API that returns a list of products. Users report that it becomes slow during a sale.

The first assumption might be that the database is too slow. Metrics show that database CPU is only 35 percent, but the application has many requests waiting for database connections. The connection pool is too small, so requests wait before their queries even begin.

The team increases the pool size. Latency improves briefly, but the database now receives more concurrent queries than it can process. Database CPU reaches 100 percent, lock waits increase, and all requests become slower.

The team investigates the query and finds that the endpoint reads more columns and rows than it needs. It adds a suitable index, reduces the result, and caches product data that changes infrequently. The final solution is not “increase the pool.” It is a combination of reducing work, controlling concurrency, and reusing stable results.

The lesson is that a resource limit is often a symptom of a deeper mismatch between workload, capacity, and system design. Increasing a limit can help, but it can also move the overload to the next component.

## How to observe the resources

The exact tools depend on the operating system, but the questions are consistent.

For CPU, inspect utilization, run queues, context switches, and profiling data. For memory, inspect resident usage, allocation behavior, page faults, cache pressure, and swapping. For storage, inspect latency, throughput, queue depth, I/O errors, and filesystem capacity. For networks, inspect connection counts, latency, retransmissions, packet loss, bandwidth, and remote-service response time.

On Linux, tools such as `top`, `vmstat`, `iostat`, `pidstat`, `ss`, `lsof`, `df`, and `strace` can provide useful evidence. These tools do not automatically identify the cause. They show different parts of the system, and the engineer must connect the observations to a hypothesis.

For example, high CPU does not prove that inefficient computation is the root cause. It may be caused by a retry loop, excessive serialization, busy polling, or a cache-miss pattern. A full disk does not prove that application data grew unexpectedly; logs, temporary files, or deleted files still held open may be responsible.

The tool is only useful when paired with a question.

## Interview definitions

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

## Interview follow-up questions

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

### “High CPU usage always means the system is unhealthy.”

High CPU may mean the system is using its available capacity efficiently. It becomes a problem when important work waits too long, latency increases, or the system has no headroom for bursts and failures.

### “More memory always makes a system faster.”

More memory can reduce swapping and keep more useful data cached, but it does not fix CPU work, lock contention, slow networks, or inefficient algorithms. Memory can also hide storage behavior during tests.

### “A fast network means network calls are cheap.”

Network calls still have connection setup, serialization, encryption, routing, queueing, remote processing, timeout, and failure costs.

### “A full disk is the only storage problem.”

Storage can also be a problem because of high latency, queueing, I/O errors, exhausted metadata, delayed writeback, or insufficient durability guarantees.

### “The bottleneck is always the busiest resource.”

The busiest resource is a useful clue, but not proof. A low-utilization component can still be on the critical path, and a busy component may be doing useful work without limiting user-visible progress.

## Summary

CPU, memory, storage, and network are the main resources behind most software systems. CPU executes instructions, memory holds active state, storage preserves data, and the network connects components across boundaries.

Each resource has capacity, latency, throughput, limits, and failure modes. They also interact: memory pressure can increase storage traffic, storage delays can increase memory usage, network calls can consume CPU and connection capacity, and adding capacity in one layer can overload another.

The systems-engineering approach is to connect user-visible behavior to resource usage and waiting. Measure the bottleneck, understand the limit, choose an overload policy, and make the result observable. Do not optimize a resource merely because its number looks large; understand whether it is actually limiting the work.

## If you want to build this later

Build a small command-line resource observer that runs another program and records how it uses the machine.

Start by recording elapsed time and exit status. Then add CPU time, maximum memory, output size, and the number of opened files. Later, run a program that reads a large file, performs CPU work, and makes a network request. Compare which resource changes when you modify the input size, file location, or number of concurrent requests.

The goal is not to recreate `top` or build a perfect monitoring system. The goal is to connect an operation to the resources it consumes and learn to distinguish active work from waiting.
