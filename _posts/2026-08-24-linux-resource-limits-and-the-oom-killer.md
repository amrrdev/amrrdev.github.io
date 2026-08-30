---
mermaid: true
title: "Linux Resource Limits and the OOM Killer"
date: 2026-08-24
categories: ["System Engineering"]
tags: [linux, cgroups, oom-killer, rlimit, resource-limits]
series: "System Engineering"
stage: "Stage 2 - Linux and Operating System Internals"
stage_order: 2
series_order: 8
---

> Stage 2 :  Linux and Operating System Internals  
> Subject area 2.2 :  Scheduling and Resource Control  
> Article 8

## The short version

Every resource on a Linux machine runs out at some point. A program can run out of file descriptors, address space, or threads even when the machine still has free disk space and free CPU. A group of programs can also use more memory than the machine has. When that happens, the kernel tries to free memory. If that fails, it ends a process as a last resort.

Limits exist to keep one program from taking everything. A limit only helps if you know what happens when you reach it and you can see that you are getting close. The out-of-memory killer, often called the OOM killer, is that last resort. When the kernel cannot free enough memory to satisfy an important request, it picks a process to end so the rest of the system keeps running. It is not a way to manage memory. It is a way to keep the machine alive after memory management has already failed.

This article answers one question. How does Linux keep usage under control, and what happens when a program or a group hits its limit?

## Where this article fits

The previous article explained how the kernel shares CPU time between threads. This article explains the boundaries that keep those threads and the services they form from using more than they should.

You will need this before virtual memory and containers, because those topics build on the same ideas of accounting and reclaim. An earlier article described why limits matter for correctness, not just for keeping things running. Here you see how the kernel enforces those limits.

## Why limits are needed

Without limits, one program could create so many processes that the table fills up. It could open files until no one else can open one. It could grab memory until the machine becomes unstable.

Limits do two things. First, they stop one program from spreading its usage and breaking the whole machine. Second, they create a point where the program must decide what to do when it cannot get more. That second point is easy to miss. A limit does not remove failure. It moves failure to a place where the program can reject work, wait with a timeout, or tell an operator.


What to do at the limit depends on the resource. Running out of a file descriptor is not handled like running out of memory. A full queue is not handled like a crashed process.

## Limits for one process

Linux keeps a set of limits for each process. A program can read them with `getrlimit` and change them with `setrlimit`. A shell shows them with `ulimit -a`. Each limit has two numbers. The soft limit is what is enforced now. The hard limit is the largest value an ordinary program can raise the soft limit to. Raising the hard limit needs special permission.

```text
hard limit is the ceiling
    soft limit is what is enforced now, and can be raised up to the ceiling
```

A limit set in one shell affects programs started from that shell. But a service started by `systemd` or a container runtime may get different limits from that manager.

### File descriptors

One limit controls how many file descriptors a process may have open. A file descriptor is a small number the kernel gives a program to refer to an open file, socket, pipe, or other kernel object. When the limit is reached, calls like `open` or `socket` fail. A program that prints `too many open files` is telling you exactly that, even though disk and memory may still look fine. The cause can be real concurrency, a limit that is too low, or a leak where the program opened descriptors and forgot to close them.

### Count of processes and threads

One limit controls how many processes or threads a user may have. When it is reached, creating a new thread or process fails with `EAGAIN`. This is a per-user limit, not a per-service limit. A user who runs several services can hit the total even when each service looks small.

### Address space

One limit controls how large the virtual address space of a process may grow. The virtual address space is the full range of memory addresses a program can use. This is not the amount of resident memory currently in RAM. A program can map a large address range while only a part of it is in RAM. Reaching this limit makes new mappings fail even when the machine still has free RAM.

### CPU time, stack, and locked memory

Other limits control how much CPU time a process may consume, how large its stack may grow, and how much memory an ordinary program may lock so it cannot be swapped out. Locked memory is used by some real-time and security-sensitive programs. But allowing it without bound could starve the rest of the machine. The details of each limit, including its unit and when it is checked, are in the manual. Names that look alike do not always mean the same behavior.

## A limit is not usage

A limit says how much a program may use. It does not say how much it uses now. To debug a problem, you compare the two.

For example, the descriptor limit is the ceiling, while `/proc/<pid>/fd` shows the current entries. The address space limit is the ceiling, while `/proc/<pid>/maps` shows the current mappings and other files show resident usage. The CPU time limit is the ceiling, while process accounting shows what has been used so far. For a group, the file `memory.max` is the ceiling and `memory.current` is what is used now. You need the current value, the ceiling, how fast the current value is changing, and which call failed.

## What happens when a per-process limit is reached

The kernel usually just rejects the call that would cross the limit. It returns an error that matches the resource.


The program must handle that error. The kernel cannot know whether it should try again, close an old resource, reject a request, or exit. If `accept` fails because descriptors are used up, trying again immediately will fail again. The program may need to close idle descriptors, reject new connections, or alert an operator.

## When a limit is for many programs

A single process limit is not enough when a service is many processes. Suppose a database allows 200 connections. Twenty service replicas that each open 20 connections reach a demand of 400, even though each process stays below its own setting.

```text
connections per process × number of processes = total demand on the shared resource
```

The same is true for memory, threads, and temporary files. A local limit does not protect a shared resource unless you add them up at the level of the shared resource.

## Groups of processes with control groups

A control group, usually called a cgroup, is the kernel's way to track a group of processes together. A cgroup can hold a single service and all its children, a container, or any other set of programs that you want to account for together.


Cgroups can account for and control CPU time, memory usage, number of processes, disk I/O, and which devices can be reached. They matter for containers because a service is rarely one process. If you only track the parent, you miss the workers it started.

## Memory controls for a group on modern Linux

Current systems usually use cgroup v2, which has one hierarchy and a set of memory files per group. The exact files depend on the kernel, but a few ideas matter most.

`memory.current` shows how much memory the group and its children use right now. It is useful, but it does not tell you how much reclaim work is happening.

`memory.peak` remembers the largest value `memory.current` has had since the group was created or reset. It matters because a measurement taken after a burst can look calm even though the group was close to its limit a moment ago.

`memory.high` is a point where the kernel starts to throttle the group. When the group goes above it, its programs are put under heavy reclaim and may slow down while the kernel tries to bring usage down. Crossing it does not by itself end a process. It is an early pressure signal that can give an external manager time to react.

`memory.max` is the hard ceiling. If the group reaches it and the kernel cannot reclaim enough, the group is out of memory and the OOM killer may end one of its tasks.

`memory.events` is a set of counters. It records how often the group was throttled, hit its high or max, or had an OOM kill. Watching these counters tells you whether the group was merely under pressure or actually lost a process.

`memory.oom.group` changes what happens on OOM. When it is enabled, the whole group is treated as one job and all its tasks may be ended together. That is right for a stateless set of workers that should be restarted together. It is wrong for a group that holds unrelated programs.

## What happens before the killer runs

The kernel does not immediately end a process when memory gets scarce. It first tries to reclaim memory. It can drop clean pages from the file cache, write dirty pages back to disk, swap anonymous pages if swapping is allowed, and shrink kernel caches. Reclaim takes CPU and can make programs slower.


A machine can be very slow before any process is ended. It spends its time reclaiming and paging while throughput falls.

## Two different out-of-memory situations

The kernel can run out of memory for the whole machine. Or it can run out for one group that hit its `memory.max` while the machine still has free memory. In the first case it may choose a task anywhere on the machine. In the second case it is limited to the group that hit its ceiling.

```text
Whole machine out of memory → may end a task anywhere
One group at memory.max → ends a task inside that group
```

This distinction matters. A container that is killed for exceeding its limit does not mean the host is out of memory. A host that is out of memory can affect programs that were individually below their limits.

## What the choice of victim looks like

When the OOM killer is needed, the kernel picks a task to sacrifice. It does not always pick the program that uses the most memory. A privileged operator can make a program more or less likely to be chosen by writing to `/proc/<pid>/oom_score_adj`. A value of `-1000` protects a task completely. But protecting everything leaves the kernel with no useful choice and the machine may stay under pressure.

The task that triggered the final allocation is not necessarily the program that slowly grew over time. The program that happened to ask for memory at the moment the system ran out is the one seen asking, while the program that grew earlier may be the real cause. A log line that says which task was chosen is evidence, not proof of the root cause.

## What you see after an OOM kill

The chosen task is usually sent `SIGKILL`, which it cannot handle, so it does not run any cleanup code. The kernel will reclaim its memory and descriptors. But any external effects it already caused, like a message it sent or a file it left half written, remain.

Typical signs are messages about OOM in the kernel log, a process that exited due to signal 9, a service manager that reports the process was killed, a container runtime that reports an OOM kill, rising reclaim and swapping before the kill, counters in `memory.events` increasing, and later a restart loop as the supervisor starts the program again. Because `SIGKILL` cannot be handled, the program itself may not have logged anything useful at the end, so kernel and supervisor logs become important.

## An OOM kill is not the same as an allocation failure

A program can get an allocation failure without any process being ended. A mapping can fail because of a per-process address space limit, a group limit, an overcommit policy, or because the kernel could not satisfy the specific request. Some kernel allocations do not trigger the killer at all and just return an error.

The program should still check every allocation and handle failure. It should not assume that the kernel will end something else and make room.

## Overcommit and address space

Linux can allow a program to reserve more virtual address space than can be backed by RAM right now. This is called overcommit. It is useful because programs often reserve large ranges and touch only a part of them. The risk is that many programs eventually touch their reservations at once, and the total demand exceeds what the machine can provide.

The kernel has a policy that decides when to allow a reservation and how much total reservation is considered commitable. That policy is a system-wide setting, not something a single program controls.

Trying to reserve address space and actually being able to use the memory are different things. A successful `malloc` does not guarantee that every byte can be touched later without pressure. A program still needs to watch real usage and handle failures.

## Why memory accounting can be confusing

Memory is used for anonymous heap and stack, file mappings, the page cache, shared pages, kernel structures, and socket buffers. The same physical page can be shared by several programs, so adding up each program's virtual or resident size can overstate what is physically used. A group view and a per-process view count sharing differently.

That is why you compare several views together. The per-process resident size, its mappings, its allocation profile, the group's current and peak usage, the page cache, swapping, pressure, and the kernel log together tell the story that one number alone cannot.

## An example where the limit hides the real problem

A service runs in a container with a 2 GiB limit. Its heap is 1.4 GiB and its in-memory cache grows during a traffic spike. At the same time it uses a few hundred megabytes of page cache and socket buffers. The container hits `memory.max`. The kernel tries to reclaim, but the active anonymous memory that the program actually needs cannot be reclaimed, so a task in the group is ended. Other containers on the same host stay fine because the host still has free memory.

The first reaction is to raise the limit to 4 GiB. The service survives the next spike, but the usage keeps growing. The real problem was a cache without a bound and without an eviction rule.

A lasting fix bounds the cache by size and age. It measures heap, cache, page cache, and group usage separately. It uses `memory.high` as an early warning where it fits, and it alerts on `memory.events`, peak usage, and restart counts. It also keeps headroom for bursts and for two copies of the service running during a deployment. The limit remains useful as a safety net, but it does not fix unbounded growth by itself.

## How to look at descriptor exhaustion

When a program reports `too many open files`, a practical sequence is to confirm which process failed, check its soft and hard descriptor limits, count entries under `/proc/<pid>/fd`, and sort them into files, sockets, and pipes to see what kind of descriptor is leaking. Then watch whether the count grows over time and compare that growth with the lifetime of requests or connections, look at error paths where cleanup might be missed, and only then decide whether the limit is too low for the real workload. The key is to tell the difference between a limit that is too low and a leak that the limit exposed.

## How to look at an OOM

When a process was ended for memory, a practical sequence is to find whether the host or just one group ran out, check kernel and manager logs, note which task was chosen and what its `oom_score_adj` was, and look at current and peak usage before the event. Then check allocation rate, cache growth, reclaim and swapping, compare the workload with its expected working set and its limit, and see whether another program caused the pressure to grow slowly. The chosen task is a clue, not automatically the cause, and the fix may be to repair a leak, bound a cache, limit concurrency, change placement, or add capacity.

## Limits should lead to a policy

A limit should be paired with a decision about what the program does when it is reached. For memory, that could mean rejecting new work before allocating more, dropping cache entries, limiting how many requests run at once, spilling work to disk, turning off an optional feature, or, for a disposable worker, allowing the whole group to be ended together.

What is appropriate depends on what the program owns. A cache entry can be dropped, a payment cannot. A background job can be delayed, a health check should stay available while optional work is shed. Limits are therefore part of program design, not just kernel configuration.

## How to choose a limit

A good limit starts from how the workload actually behaves and what failure mode is acceptable. It helps to ask what the normal and peak usage are, what burst must be handled, what other work shares the machine, what the working set that must stay resident is, what can be reclaimed or dropped, what work can be rejected or delayed, what should happen at the boundary, how quickly the service can scale or restart, what happens when two copies run during a deployment or when a machine fails, and which signal will show the limit is approaching.

A limit chosen from one successful test is rarely enough. Concurrency, data size, traffic shape, background work, and allocator behavior all affect what will be needed in production.

## What memory.max really means as a hard limit

`memory.max` is the hard ceiling for a cgroup, but reaching the number does not by itself end a process. The kernel first tries to reclaim, and only when reclaim cannot bring usage back under the ceiling does the OOM killer run inside that group. A group can sit at or slightly above `memory.max` for a short time while the kernel drops clean pages or swaps anonymous memory. During that window the workload simply slows rather than dies. The useful mental model is that `memory.max` is the point where the kernel is allowed to kill, not the point where it must kill. You can also write to `memory.reclaim` to ask the kernel to reclaim a given amount from the group on your own schedule. That is a way to shed pressure before the ceiling is reached. If you want earlier, non-lethal action, `memory.high` is where throttling and forced reclaim begin, leaving the hard ceiling as the final safety net.

## Steering the OOM killer with oom_score_adj

The OOM killer picks a victim using a score that starts from memory usage and is then shifted by each process's `oom_score_adj`. Writing `-1000` to that file fully protects a process, which is the right choice for a critical supervisor or a system daemon you never want sacrificed. Writing a positive value instead marks a process as more likely to be chosen, which can be a deliberate decision for a disposable worker or a batch job you would rather lose first. Protecting everything with `-1000` is a mistake, because then the kernel has no useful victim and may keep the machine under pressure or fall back to ending whatever it can. The adjustment is per process and does not propagate to children automatically. A manager that forks workers must set it on each one if the protection is meant to cover the whole tree.

## What the memory controller actually counts, including the page cache

The cgroup memory controller accounts for anonymous memory such as the heap and stack, file mappings, the page cache, shared pages, kernel data structures, and socket buffers. What surprises people is that the page cache counts toward the group's usage even though most of it is reclaimable. A service that reads or writes many files can look close to its limit while much of that memory could be dropped. Shared pages are charged under rules that depend on the kernel, and the same physical page can appear in several views, so summing per-process numbers overstates what is physically used. When you debug, expect `memory.current` to include cache you did not allocate directly, and read `memory.stat` to separate anonymous, file, and kernel memory rather than assuming the total is all heap.

## Process and thread limits inside a cgroup with pids.max

The per-process limits described earlier are not the only way to bound how many processes a service can create. cgroup v2 has `pids.max`, which caps the number of processes and threads across the whole group and its children. It is the limit that matters for a container or a systemd service that is many processes. When `pids.max` is hit, `fork` and `clone` fail with `EAGAIN`. A service that leaks threads or spawns helpers without bound stops being able to start anything new, while its existing work may keep running. This is different from the per-user ulimit on process count, which is applied per login session and can be exhausted by unrelated services that share the same user. A group limit is the one that actually contains a single service, because it counts the workers the service started rather than the identity that started them.

## Which limit actually applies: ulimit, systemd, and cgroup v2

The same resource can be constrained in three places, and the effective bound is the most restrictive of them. A per-process `RLIMIT` set by `ulimit` applies when a program raises its own soft limit or runs under a shell that set one. `systemd` can set the same kind of limit through unit directives such as `LimitNOFILE` or `MemoryMax` for the service it starts. A cgroup v2 limit applies to the whole group and its children regardless of what each process believes its own limit to be. Common cgroup v2 deployments do not add a separate file-descriptor limit, so the per-process `RLIMIT_NOFILE` is normally what a program hits when it reports too many open files. The practical rule is that the kernel enforces the tightest of the process limit and the cgroup limit. A service may be killed by `memory.max` even when its own ulimit looked generous, or it may hit `RLIMIT_NOFILE` long before any group-level bound. When a limit is reached, check all three layers before concluding which one fired.

## Interview definitions

### What is a resource limit?

> A resource limit is a cap on how much CPU, memory, processes, file descriptors, or another finite resource a program or a group may use.

### What is the difference between a soft and a hard limit?

> The soft limit is what is enforced now. The hard limit is the largest value an ordinary program may raise the soft limit to. Raising the hard limit needs special permission.

### What is `RLIMIT_NOFILE`?

> `RLIMIT_NOFILE` caps how many file descriptors a process may have open, including files, sockets, and pipes.

### What is a cgroup?

> A control group is how Linux tracks a group of programs together so their resource use can be counted and limited as one unit.

### What is the OOM killer?

> The OOM killer is the kernel's last resort when it cannot free enough memory to satisfy an important request. It ends a chosen task so the rest of the system can continue.

### What is `memory.high`?

> `memory.high` in cgroup v2 is the point where a group is throttled and put under heavy reclaim but is not yet at the hard ceiling. It is an early warning sign of pressure.

### What is `memory.max`?

> `memory.max` in cgroup v2 is the hard ceiling. If the group reaches it and reclaim cannot bring usage down, the group can trigger an OOM kill inside it.

## Interview follow-up questions

### Why is raising a limit not always the right fix?

> The limit may be showing a leak or unbounded work. Raising it can just hide the failure or push pressure to a shared resource downstream. I would first look at who uses the resource, how fast it grows, and what the real capacity needs to be.

### What is the difference between a per-process limit and a cgroup limit?

> A per-process limit applies to one program, while a cgroup limit applies to a group and its children. A service that is many processes needs a group limit to be contained.

### Does the OOM killer always end the program that uses the most memory?

> No. It uses a heuristic that considers usage and how the administrator adjusted `oom_score_adj`. The program that triggered the final allocation may not be the one that slowly grew.

### What is the difference between a host OOM and a container OOM?

> A container can hit its `memory.max` while the host still has free memory and only that container's tasks may be ended. A host-wide OOM happens when the whole machine cannot reclaim enough and may affect work outside the original container.

### What should a program do when an allocation fails?

> It should check the failure, free or drop what is optional if possible, reject or delay work according to its contract, and record enough context to debug. It should not assume the kernel will end something else or that trying again immediately will help.

### How would you look into an OOM kill?

> I would check kernel and manager logs, see whether it was host-wide or group-local, look at current and peak usage, check reclaim and swapping, note which task was chosen and its adjustment, and look for the workload that made usage grow.

## Common misconceptions

### “The OOM killer allocates memory.”

It does not. It is the recovery step that runs after normal reclaim has failed. Allocation, reclaim, and swapping happen first.

### “The killed program must have had a leak.”

The chosen task is often the one that happened to ask for memory when the system ran out, while another program grew slowly beforehand. Logs and history are needed to know the cause.

### “A container limit protects the whole host.”

It protects that group, but the host still needs memory for the kernel and other work. The host can run out even when each program stays below its own limit.

### “More swap always prevents OOM.”

Swap can hold some anonymous memory, but it is slower than RAM and it cannot fix unbounded growth or every kind of pressure. Heavy swapping can make the machine unusable before any kill happens.

### “`memory.high` and `memory.max` are the same.”

`memory.high` is where the group is throttled and reclaim is forced. `memory.max` is the hard ceiling where tasks inside the group may be ended.

### “Per-process limits are enough for a service.”

A service is often many programs and replicas that share a resource. Containing it usually needs group accounting as well as per-process limits.

## Summary

Limits keep a program from taking more than it should, and they create a clear boundary where the program must decide what to do. Per-process limits like `RLIMIT` cap one program, while control groups cap a set of programs together. That is how services and containers are contained. Under pressure the kernel reclaims first and may throttle a group at `memory.high`. That hard ceiling at `memory.max`, or a whole-machine shortage, can trigger the OOM killer. The killer is a last resort that keeps the machine alive but does not fix the workload. Reliable services use bounded caches and queues, limit concurrency, have a clear policy at the boundary, and watch usage, peaks, pressure, and kill counters.

## If you want to build this later

Make a small lab on Linux to try out limits. Write programs that open files until `RLIMIT_NOFILE` is hit, create threads until a limit is hit, and allocate memory inside a small cgroup. Note the errors you get, look at `/proc`, and compare a per-process limit with a group limit. For the memory experiment, use a disposable workload and a conservative limit, watch `memory.current`, `memory.peak`, and `memory.events`, and make the program drop its cache or reject work when pressure appears. The goal is to see how containment behaves and why the killer is not a normal control mechanism.
