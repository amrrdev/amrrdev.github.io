---
mermaid: true
title: "CPU Scheduling and Context Switching"
date: 2026-08-23
categories: ["System Engineering"]
tags: [linux, scheduling, context-switch, cpu]
series: "System Engineering"
stage: "Stage 2 - Linux and Operating System Internals"
stage_order: 2
series_order: 7
---

> Stage 2 :  Linux and Operating System Internals  
> Subject area 2.2 :  Scheduling and Resource Control  
> Article 7

## The short version

A machine may have a few CPUs and many threads that want to run. The kernel's scheduler decides which thread runs on which CPU and when to give the CPU to another thread.

The scheduler tries to be fair, keep the system responsive, and handle priorities, but those goals can conflict. Giving one thread more time can delay another. Running many more threads than there are CPUs creates waiting and extra work to switch between them.

A context switch is what happens when the CPU stops running one thread and starts another. The kernel saves the first thread's registers and restores the second thread's registers. The switch itself costs time, and the new thread may find that its data is no longer close in the caches.

The central question is how the system decides what runs next when there is more work than can run at once, and what that decision costs.

## Where this article fits

The previous articles showed how to observe live process state through `/proc` and how time is measured. This article explains the part of that state that says whether a thread is running, ready to run but waiting for a CPU, or waiting for something else.

Later articles will connect this to threads, locks, caches, and resource limits. Scheduling is where the resource called CPU time is shared, and where contention first appears as waiting.

## The smallest unit the scheduler works with

On Linux the scheduler works with threads, not whole processes. A process is a container for resources like an address space and file descriptors. Inside it there can be one or many threads that share that address space.

A process with one thread has one path of execution that the scheduler can choose. A process with three threads has three paths that can be chosen independently, even though they share the same memory and file descriptors.


This matters when you read CPU usage. A process can show high usage because one thread is busy, or because many threads are each a little busy. A process with many threads can still make slow progress if most of them are waiting for the same lock.

## States a thread moves through

A thread is not always running. It moves between a few states as it runs, waits, and finishes.


A running thread is currently on a CPU executing instructions. A thread that is runnable is ready to run but is not on a CPU. It may be waiting because all CPUs are busy or because another thread has higher priority in the scheduling policy. Runnable is different from sleeping. A runnable thread wants a CPU now. A sleeping thread is waiting for something else and should not be given a CPU until that thing happens. A stopped thread is not allowed to run until it is continued, for example after `SIGSTOP` or when a debugger stops it.

## Why the kernel has to decide

If one thread could run forever without being interrupted, no other thread would make progress. The kernel therefore shares the CPUs.


On a machine with several cores, several threads run at once, but there can still be more runnable threads than cores. Each CPU has its own queue, and the kernel may move threads between CPUs to balance the load.

## The kernel can interrupt a running thread

On Linux the scheduler is preemptive, which means the kernel can interrupt a running thread and give the CPU to another thread. The running thread does not have to call a function to give up the CPU for this to happen. This protects fairness. A program that does a lot of computation and never waits would otherwise keep the CPU forever.

The kernel may make a new decision after a timer interrupt, after a thread becomes runnable, after a running thread blocks, after a priority changes, after a CPU becomes idle, or after a system call returns.

Interrupting has a cost. The kernel must save the thread's registers, choose the next thread, and restore its registers. Doing this very often helps the system feel responsive, but it adds overhead and makes caches less effective.

## Programs should still wait in a good way

Even though the kernel can interrupt, programs should not waste CPU. A common waste is a loop that keeps checking whether work is available without sleeping.

```c
while (!work_available) {
    // keeps using the CPU while checking
}
```

If `work_available` changes rarely, this loop burns a CPU that could do useful work. A better way is to wait for an event, a condition variable, a file descriptor, or a timer, so the thread sleeps until there is something to do.

An interrupt guarantees that a busy thread does not starve others forever, but it does not make busy waiting free.

## Priorities and fairness

The scheduler uses a policy and priorities to decide which runnable thread should run. For ordinary work, the policy tries to share CPU time fairly and keep the system responsive. A nice value can give a hint. A program that is nicer lets other programs run more, while a less nice program asks for more CPU.

There are also policies for work with deadlines. They can give a thread stronger priority, but if they are used without care they can starve ordinary work. A priority is not the same as business importance. Giving a thread a high kernel priority means it will run even when that makes other work miss its deadline, so the choice should consider the whole machine.

Fairness does not mean every thread gets the same amount of time. A thread that is sleeping should not get CPU while it has nothing to do. A thread in a group that is limited by a control group may get less than a thread in another group. The exact algorithm changes across kernel versions, so it is better to think in terms of policy, priority, whether a thread is runnable, and measured waiting time, rather than memorizing one implementation.

## Time slices

You may hear about a time slice as the amount of time a thread can run before the scheduler considers another thread. Modern Linux is more complex than giving every thread a fixed slice, but the idea is still useful. A thread that runs briefly and then blocks may get many chances to run without ever using a full slice. A thread that runs continuously may be interrupted so others get a turn.

Very frequent switching makes the system feel quick, but it adds overhead. Very infrequent switching helps a long computation finish faster, but it can make interactive work feel slow. The scheduler balances the two.

## What a context switch does

A context switch changes which thread a CPU runs. The kernel saves the execution state of the old thread and restores the state of the new thread. That state includes general registers, the instruction pointer, the stack pointer, flags, and when needed floating point state and scheduling information. If the next thread belongs to a different process, the kernel may also change the address space it uses.


The switch does not copy all of the process's memory. It only saves and restores the registers and other execution state needed to resume. Changing the address space when switching processes can make it a bit more expensive, but modern hardware reduces some of that cost with tagged translation caches.

## Why a switch costs more than just saving registers

Saving and restoring registers is only part of the cost. When a new thread starts, the data it needs may not be in the CPU caches. The instructions it will run and the data it touches may have to be fetched again.

```text
Thread A had useful lines in the caches
    switch to Thread B
Thread B needs different lines, so some are fetched again
    some of A's lines are evicted
```

If many threads take turns on the same CPUs, they keep displacing each other's working sets. The cost also includes the scheduler's own work, any contention on kernel locks, and the work to wake and sleep threads.

## Two reasons a switch happens

A switch can happen because the running program asked to wait. For example, it tried to read from a socket with no data, wait for a lock, or sleep for a timer. In that case the kernel puts it to sleep and runs another thread. This kind of switch tells you the thread is waiting for something other than CPU.

A switch can also happen because the kernel decided to interrupt the running thread. Its time was up, or another thread with higher priority became runnable. This tells you that many threads want CPU at the same time or that the system is preempting often.

The distinction helps you debug. Many switches where threads give up the CPU suggest they wait for disk, network, or locks. Many switches where the kernel interrupts suggest many runnable threads competing for the same CPUs.

Neither kind is automatically bad. The question is whether the waiting and the switching match what the workload should do.

## Moving between CPUs and keeping data close

On a machine with many cores, a thread can run on different CPUs over time. Moving helps balance load, but it can hurt caches and memory locality. The new CPU may not have the thread's recent data, and on a NUMA machine some memory is cheaper to reach from one CPU than from another.

Pinning, which means restricting a thread to a set of CPUs, can make latency more predictable for a special workload or keep a thread near a device queue. It can also hurt when the chosen CPU becomes busy and the thread could have run elsewhere. The scheduler usually knows more about the overall balance than a single program, so pinning should be used only after measuring and for a clear reason like device affinity or real-time needs.

Tools like `taskset` and `numactl` can show or set affinity, but they should be used with an understanding of the whole workload.

## When waiting is better than looping

A thread that has nothing to do should usually wait for an event instead of looping.


Looping can be right when the wait is expected to be extremely short and every microsecond matters, or in specialized high-performance code. For ordinary work, waiting with an event, condition variable, or file descriptor uses far less CPU and leaves it for other threads.

## Why more threads can make things slower

Threads are not free. Each one needs a stack, kernel bookkeeping, and often application state. If many threads are runnable at once, they compete for the same CPUs and spend more time switching and waiting for locks.

If most threads are waiting for disk or network, having many can be reasonable. If most are doing computation, the number that can usefully run at once is close to the number of CPUs that can do the work, after accounting for memory and locks.

A thread pool makes the limit explicit, but its size should match the workload. Too small and work waits even though CPUs are free. Too large and memory grows, contention rises, and queueing hurts latency. The right question is not how many threads the machine can create, but how many concurrent operations the CPUs, memory, and dependencies can actually support.

## Real-time work

Some work has deadlines that are more important than average speed. A hard real-time system must meet its deadlines under its stated conditions, while a soft real-time system tries but may miss occasionally. Policies that give a thread stronger priority can help protect that work, but a misconfigured real-time thread can also keep ordinary system threads from running at all.

Deadlines also depend on more than the scheduler. Interrupt handling, memory allocation that faults, locks, and device latency all affect whether a deadline is met. Choosing a real-time policy alone does not guarantee it.

## How to look at scheduling

Linux lets you see scheduling through ordinary tools.

```bash
ps -eLo pid,tid,psr,stat,pri,ni,pcpu,comm
pidstat -w -p 2450 1
taskset -pc 2450
perf stat -e context-switches,cpu-migrations ./program
```

These show the process and thread identifiers, which CPU a thread last ran on, its state, priority and nice value, and how often it switched or moved. A large count of switches or migrations alone does not say why. Combined with CPU usage, lock wait time, and latency, it points to whether many threads are runnable and competing or many are waiting and waking often.

For example, a service that was changed from 16 workers to 256 workers used more CPU, but throughput hardly changed and tail latency got worse. Measuring switches, migrations, and lock waits showed the workers were mostly doing computation and fighting over the same cache lock. Fewer workers with less sharing improved latency, even though the number of threads went down.

## How experienced engineers think about it

They first ask what the thread is doing. Is it actually running, is it ready but waiting for a CPU, is it sleeping for disk or a lock, is it stopped, or is it waking and sleeping repeatedly. Then they compare that with what the user sees. Is CPU usage high, is the run queue long, are there many switches, are migrations high, how long do locks wait, how long does disk or network wait, and what happens to tail latency and throughput.

They do not pin CPUs or raise priorities just because those knobs exist. A change is made to fix a specific observation, and afterward they check that it did not starve another part of the system.

## How the Completely Fair Scheduler turns fairness into a number

The Linux scheduler that most ordinary work uses is called the Completely Fair Scheduler, or CFS. Its core idea is to give every runnable thread a share of CPU time that is proportional to its weight, so that over time each thread's accumulated runtime lines up with its fair portion.

CFS keeps a per-thread value called vruntime, which is the amount of time that thread has spent running, weighted by its priority. Lower vruntime means the thread has run less than its fair share, so the scheduler picks the runnable thread with the smallest vruntime next. This keeps the threads that have gotten the least of what they deserve moving forward, without giving any single thread a fixed time slice.

Fairness here is approximate rather than exact. vruntime advances more slowly for higher-priority threads, so a nice value of minus twenty gets more CPU per unit of vruntime than a nice value of nineteen. The scheduler rebalances on a timescale of milliseconds, not instantly, and it organizes runnable threads in a red-black tree so choosing the next one stays cheap. The thing to remember is that CFS is a heuristic for "who has gotten least of what they deserve," not a guarantee of equal time.

## Real-time policies and the danger of starving everything else

For work with strict timing, Linux offers real-time policies that outrank the normal CFS class entirely. SCHED_FIFO runs a thread at a fixed priority and never preempts it for a lower-priority thread until that thread blocks or yields. SCHED_RR is the same but adds a time slice so equal-priority threads take turns. SCHED_DEADLINE uses a reservation model, where a thread declares a period and a runtime budget, and the scheduler guarantees that budget within each period.

These policies are powerful and risky. A SCHED_FIFO thread that never blocks will keep its CPU forever and prevent every lower-priority thread, including init and your shell, from running at all. A misconfigured SCHED_DEADLINE reservation can also consume so much of the machine that ordinary work starves. The safe practice is to use the lowest real-time priority that meets the deadline, keep the critical section tiny, and never run unbounded loops in that class.

## Controlling CPU with the cgroup v2 cpu controller

On a modern system the scheduler is usually shaped through cgroup v2 rather than by tuning individual threads. The cpu controller lets you place a group of threads into a hierarchy and put limits on them that the scheduler enforces.

The file cpu.max sets a bandwidth limit as a pair of numbers, a quota and a period, such as "100000 100000" for one CPU's worth of time per period. The file cpu.weight sets a proportional share, where a group with weight two hundred gets twice the CPU of a default group at one hundred when both want the CPU. The file cpu.stat reports what actually happened, including how much time the group was throttled for exceeding its quota. This is how you stop one noisy job from eating the whole machine while still letting it burst when nothing else is contending.

## Isolating CPUs for latency-sensitive work

Sometimes the goal is not to share fairly but to keep a CPU quiet so one thread can run with the least interruption. The kernel boot parameter isolcpus removes chosen CPUs from the normal scheduling and load-balancing pool, so the scheduler will not place ordinary threads there. The nohz_full option goes further by stopping periodic timer ticks on those CPUs, which removes a steady source of forced context switches.

You then steer your important thread onto the isolated CPU with sched_setaffinity from the program or with taskset from outside. The benefit is predictable latency with almost no scheduler or timer noise. The cost is that the isolated CPU is wasted for everything else, and you must still handle interrupts, which by default can arrive on any CPU unless you also steer them with irqaffinity or the irqbalance daemon.

## The noisy-neighbor problem and the idle and batch policies

On a shared or virtualized host, your workload can slow down because another tenant on the same machine keeps the CPUs busy. This is the noisy-neighbor problem: you are not getting less CPU because your program changed, but because the physical hardware is shared and the scheduler is dividing it with someone else. Measuring only your own process will not reveal it; you have to look at host-level CPU saturation and at whether your run queue grows while your own threads stay busy.

Linux offers two policies at the opposite end from real-time. SCHED_BATCH tells the scheduler this thread is doing batch computation and can be woken less aggressively, which lets the kernel group its work to improve cache use. SCHED_IDLE is for work that should run only when no one else wants the CPU at all, such as background indexing or log compaction. Used well, these keep low-priority jobs from disturbing interactive or latency-sensitive work, which is the other half of fairness that strict priorities alone cannot give you.

## Interview definitions

### What is CPU scheduling?

> CPU scheduling is how the kernel chooses which runnable thread should run on each CPU and when to give the CPU to another thread.

### What is a runnable thread?

> A runnable thread is ready to run but is waiting for a CPU. It is different from a sleeping thread, which is waiting for a disk operation, a lock, or a timer and should not be given a CPU until that thing is ready.

### What is preemptive scheduling?

> Preemptive scheduling means the kernel can interrupt a running thread and run another thread, without waiting for the first thread to say it is willing to give up the CPU.

### What is a context switch?

> A context switch is when the kernel saves the execution state of one thread and restores the state of another so the CPU can continue different work.

### What is a switch because the thread waited versus a switch because it was interrupted?

> In one case the running thread asked to wait, for example for disk or a lock, so the kernel runs another thread while it waits. In the other case the running thread could continue, but the kernel interrupted it because another thread had been waiting or had higher priority.

### What is CPU affinity?

> CPU affinity restricts a program to a set of CPUs. It can help keep data close to the cache or near a device, but it can also make balance worse if the chosen CPU becomes busy.

## Interview follow-up questions

### What is the difference between a process and a thread for the scheduler?

> The kernel schedules threads. Threads in one process share the same address space, while threads in different processes have separate spaces.

### Why does a context switch have a cost?

> The kernel must save and restore registers, and the new thread may have lost the data that was warm in the caches. It may need to fetch instructions and data again, and the kernel also does scheduling work.

### Why can more threads make a program slower?

> Many runnable threads compete for the same CPUs and the same locks, and they cause more switches and more cache misses. More concurrency only helps while there is useful independent work and a resource to run it.

### What is the difference between runnable and sleeping?

> Runnable means ready and waiting for a CPU. Sleeping means waiting for something else, like disk or a lock, and not needing a CPU until that thing is ready.

### When can pinning help?

> Pinning can help when you want to keep a latency-sensitive thread near its data or near a device queue, or when you have a specific real-time need. It can hurt when it stops the scheduler from balancing load or keeps work on a CPU whose memory is far away on a NUMA machine.

### What is starvation?

> Starvation is when a runnable thread waits for a very long time because other threads with higher priority or better sharing always get chosen first.

### Is many context switches always bad?

> No. Switches are needed when threads wait for disk or when many threads share a CPU. It becomes a problem when switching and waking use a noticeable amount of CPU or when they contribute to higher latency.

## Common misconceptions

### “The scheduler runs processes, not threads.”

The scheduler runs individual threads. A process is the container with address space and resources, but the schedulable unit is the thread.

### “A context switch copies all memory.”

It saves and restores registers and execution state. It does not copy the whole address space, although changing address spaces can make the next accesses miss in the translation cache.

### “More cores always fix scheduling.”

More cores help when CPU is the bottleneck, but locks, memory bandwidth, disk, and serial parts of the workload can still be the limit.

### “A sleeping thread wastes CPU.”

A sleeping thread is waiting and does not use a CPU. A loop that keeps checking again without waiting is the pattern that wastes CPU.

### “Pinning always makes things faster.”

It can keep data close, but it can also keep work on a busy or distant CPU and prevent the scheduler from balancing a burst.

### “Real-time priority guarantees real-time behavior.”

Priority helps choose who runs, but deadlines also depend on interrupts, memory, locks, and devices, not just the scheduling class.

## Summary

The kernel schedules threads, not processes. A thread can be running, runnable and waiting for a CPU, or sleeping and waiting for something else. Preemptive scheduling lets the kernel interrupt a running thread so others get a turn. A context switch saves and restores state, and it often loses some cache warmth. Many runnable threads cause more switches and more competition for locks without adding useful throughput. The right way to diagnose is to look at thread states, CPU usage, queue length, switching, lock waits, and latency together, and to change affinity or priority only to fix a specific observation.

## If you want to build this later

Write a small program that can start a chosen number of threads that either do computation or mostly sleep. Measure how long it takes, how much CPU it uses, and how often it switches and migrates. Start with a number close to the number of CPUs and then increase it. Add a shared lock that all workers use, then change the program so each worker touches less shared data. Compare the default scheduler with a run where you pin threads to CPUs. The goal is to see the difference between running, runnable, and sleeping, and to see when adding concurrency turns into overhead.

