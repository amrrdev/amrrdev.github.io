---
mermaid: true
title: "Stack and Heap Layout"
date: 2026-08-28
categories: ["System Engineering"]
tags: [stack, heap, address-space, call-frames, stack-overflow, thread-stacks]
series: "System Engineering"
stage: "Stage 6 - Memory Management"
stage_order: 6
series_order: 5
---

The first four chapters of this stage described the address space as a whole. They covered the tables that translate it, the faults that fill it, and the mappings that project files into it. This chapter looks at the two regions a running program uses every moment: the stack and the heap. It is the fifth article of Stage 6. It is also the base for the allocator and memory-safety chapters that come next.

Every function call puts something on the stack. Almost every dynamic data structure lives on the heap. A backend engineer must know where each one lives, how it grows, and what bounds it. You must also know what breaks when those bounds fail. A wrong stack size can crash a thread. A misunderstood heap can become a slow memory leak in production. This difference is also where many security bugs start. The later memory-safety chapter covers those bugs.

## The shape of a process address space

A Linux process on x86-64 does not get one flat block of memory. The kernel places several distinct regions in the virtual address space. Each region has a job. From low to high addresses, the usual order is: the program text (the machine code), the initialized data, the zero-initialized data, the heap, the memory-mapped region, and the stack at the top.

```mermaid
flowchart LR
    T[Text: program code] --> D[Data and BSS: globals]
    D --> H[Heap grows upward]
    H --> M[mmap region]
    M --> S[Stack grows downward]
```

The heap and the stack grow toward each other. The heap starts just above the data and bss. It expands upward as the program asks for more memory. The stack starts near the top of the address space. It expands downward as functions are called and local variables are created. Between them sits the mmap region. This is where the previous chapter's file mappings and shared memory live. Most shared libraries live there too. As long as the two moving boundaries do not meet, the process has room.

This layout explains the first chapter's observation of `/proc/<pid>/maps`. The command showed several named regions. `[heap]` and `[stack]` appear as labeled segments. The mmap region is the cluster of file and anonymous mappings in the middle. ASLR randomizes the addresses, as covered earlier. The relative order stays stable.

## The stack and what it holds

The stack is the region that backs function calls. Each time a function runs, the compiler emits code. That code reserves a slice of the stack for that function's frame. The frame holds the return address (where to go back when the function ends), saved registers, and the function's local variables. When the function returns, its frame is discarded. The stack pointer moves back.

```mermaid
flowchart LR
    Top[Top: newer frames] --> F2[Caller's caller frame]
    F2 --> F1[Caller frame]
    F1 --> F0[Current frame: ret addr, regs, locals]
    F0 --> Bot[SP moves down as frames added]
```

The key property is automatic lifetime. A local variable exists exactly as long as its function is on the call chain. It vanishes the instant the function returns. The program does not ask for it or free it. Moving the stack pointer is the whole allocation and deallocation step. This makes the stack very fast. Reserving space is one register adjustment. Freeing is another. There is no bookkeeping, no search for free space, and no fragmentation.

The stack is a single contiguous block per thread. This gives it excellent cache behavior. Variables in the current frame and in recently called frames sit close together in memory. They are likely already in cache. This locality is one reason that deeply nested code with small local variables runs fast.

## Stack growth and the guard page

The stack grows downward. Each new frame uses a slightly lower address. The kernel allocates the stack lazily. It does not wire up every page at the start. Instead it places a guard page just past the current limit. When a function writes into the guard page, a page fault fires. The kernel extends the stack by mapping the next page. Execution continues. This is the same demand-paging idea from the page-fault chapter, applied to one specific region.

The guard page is also the safety rail. Suppose the stack grows past what is available. There is no room left before it would collide with another region. The next access hits an unmapped area with no valid mapping. The process receives a segmentation fault. This is the classic stack overflow. It is not a graceful error. It is a signal, and by default it kills the process.

The maximum stack size is bounded. On Linux the default is often eight megabytes per thread. You can see it through `ulimit -s`. That limit is deliberate. A stack that grew without bound would let a bug consume the address space. A thread that needs more must ask for it explicitly. It usually does so when the thread is created.

## Thread stacks and the heap they share

In a multi-threaded program, each thread gets its own stack. The main thread uses the process stack described above. Every worker thread gets a separate stack. It is typically allocated in the mmap region and sized to the requested or default limit. This is why a crash in one thread's deep call chain does not corrupt another thread's stack. They are separate regions with separate guard pages.

The heap is shared. All threads allocate from the same heap and the same allocator. This means heap allocation must be thread-safe. The allocator uses locks or per-thread caches (the next chapter covers how). Two threads can call `malloc` at once without corrupting the heap. The stack needs no such coordination, because each thread owns its own.

This split explains a recurring design tension. Stack memory is private, fast, and automatically reclaimed. But it is small and fixed in size. Heap memory is shared, large, and long-lived. But it is slower to obtain, and the engineer must manage its lifetime. Choosing where data lives is one of the most important decisions in systems code.

## The heap and dynamic lifetime

The heap exists for data whose size or lifetime is not known at compile time. Suppose a function allocates a buffer whose length comes from a request. Or it builds a structure that must outlive the function that created it. That memory comes from the heap. The allocator carves it out of a larger region the kernel granted. It returns a pointer.

Unlike the stack, heap memory does not vanish when the allocating function returns. Its lifetime is controlled by the programmer. In C and C++ this happens through an explicit free. In Go, Java, and others a runtime system does it through a garbage collector. That freedom is powerful. It lets programs build trees, lists, caches, and request-scoped buffers of any size. It is also the source of two lasting problems. The first is leaks, when memory is never freed. The second is fragmentation, when free memory exists but not in a usable contiguous shape.

The heap grows upward through `brk` and `mmap`. Small or early allocations often extend the heap boundary with `brk`. Larger allocations and many runtime allocators use `mmap` to obtain whole pages directly. Either way, the kernel hands out pages. The user-space allocator subdivides them into the small blocks a program requests. The next chapter examines that subdivision closely.

## Internal and external fragmentation

Fragmentation is why the heap is not simply a big array you append to. Two kinds matter.

Internal fragmentation happens when the allocator gives you a block larger than you asked for. Suppose the allocator works in fixed sizes. A request for 24 bytes may round up to 32. The extra 8 bytes are wasted inside the block. Across millions of allocations this waste adds up to noticeable memory overhead.

External fragmentation happens when free memory exists in total but not in a single contiguous piece. Suppose the heap has two free 1 KB blocks separated by an allocated 1 KB block. A request for 2 KB contiguous bytes cannot be satisfied from those free pieces, even though 2 KB is free overall. The allocator either fails, falls back to asking the kernel for more, or spends effort joining adjacent free blocks. Long-running services that allocate and free many different sizes are exactly where external fragmentation bites.

The stack does not have these problems. Every frame at its level is the same shape, and the stack reclaims it whole. The price the stack pays is its rigid size limit.

## Stack versus heap, in practice

The tradeoffs are worth stating plainly, because they decide where data should live.

| Property | Stack | Heap |
|---|---|---|
| Allocation speed | One pointer move, very fast | Allocator work, slower, may fault |
| Lifetime | Until function returns | Until explicitly freed or collected |
| Size limit | Small, fixed per thread | Large, bounded by RAM and address space |
| Thread safety | Private per thread | Shared, needs coordination |
| Fragmentation | None | Internal and external possible |
| Failure mode | Overflow kills the thread or process | Leak grows RSS, fragmentation wastes it |

Here is a useful rule. If the data's lifetime matches a function call and it fits within the stack limit, the stack is the right home. If the size is large, comes from input, or must survive the call, it belongs on the heap. The bugs in this area are predictable. Putting a huge buffer on the stack invites overflow. Putting short-lived small objects on the heap invites allocator pressure and leaks.

## Stack overflow in the real world

Stack overflow is not only a student's infinite recursion. It appears in production as deep call chains on hostile or unexpected input. It also appears as a too-large local variable. Suppose a parser recurses on nested structure and gets pathologically deep input. It descends until it exhausts the eight-megabyte stack. Suppose a function declares a large local array, say a buffer of several megabytes. It consumes a large slice of the stack in a single frame. That leaves little room for anything else.

When the guard page is breached, the process takes a `SIGSEGV`. The crash dump shows a stack trace many thousands deep. Or it shows a single frame whose local variable is enormous. The fix is rarely to raise the limit endlessly. The fix is to move large buffers to the heap. Or rewrite the recursion as an explicit loop with a heap-allocated work stack. That trades a bounded stack for an effectively unbounded heap.

## Observing the regions

You can see the stack and heap directly. `/proc/<pid>/maps` labels them, and `/proc/<pid>/status` reports their sizes. A program can also print the addresses of its own variables to show where each region sits.

```go
package main

import (
    "fmt"
    "os"
    "runtime"
)

var global int

func show() {
    local := 1
    heap := new(int)
    fmt.Printf("code   (show fn): %p\n", show)
    fmt.Printf("global var    : %p\n", &global)
    fmt.Printf("local  var    : %p\n", &local)
    fmt.Printf("heap   alloc  : %p\n", heap)
}

func main() {
    show()
    fmt.Println("goroutines:", runtime.NumGoroutine())
    _ = os.Getpid()
    select {}
}
```

```bash
go build -o layout main.go
./layout &
pid=$!
sleep 0.3
grep -E "\[heap\]|\[stack\]" /proc/$pid/maps
grep -E "VmStack|VmData|VmExe|VmRSS" /proc/$pid/status
echo "default stack limit (bytes):"; ulimit -s
kill $pid
```

What it shows depends on the runtime. In a C program the local's address is high (near the stack top). The heap allocation's address is lower (in the heap region). The global is in the data/bss area. The function code is in the text region. This exactly matches the layout diagram. In Go the picture is more subtle. Goroutine stacks are small and can be moved and grown by the runtime. What `new` returns still lives in the heap. The printed addresses still show the same relative ordering at a high level. The point is that these regions are real, addressable, and inspectable. They are not abstract.

## A realistic production example

A team ran a service that parsed a configuration language. The parser was written recursively. It descended into nested blocks with one function call per level. For normal configurations with a few dozen levels of nesting, it worked well. Then a customer submitted a configuration with thousands of nested blocks. This was a shape the authors had never tested.

Each nesting level consumed a stack frame. The frames accumulated down the call chain. At roughly eight megabytes of stack, the next frame crossed the guard page with no room left. The worker thread took a `SIGSEGV`. The process died. Because the parser ran on the request thread, the whole worker process crashed. Not just the one request. The logs showed a stack trace thousands of frames deep ending in the parser. That was the tell.

The first fix some engineers tried was to raise the stack size with `ulimit -s`. Or they used a larger thread stack. That only delayed the crash. A deeper input would still overflow a larger stack. A bigger stack per thread also raised the memory cost of every worker. The real fix was to change the parser to use an explicit loop. It used a heap-allocated stack of work items. The depth of nesting became a heap allocation bounded by available memory. It was no longer bounded by the fixed thread stack. After that, the same pathological configuration parsed correctly. The only cost was a larger but manageable heap allocation. The service could reject it if it grew truly unreasonable.

The lesson is that the stack is a fixed, per-thread resource. It is sized for ordinary call depth, not for adversarial or unexpected input. Anything whose depth or size can be driven by external data belongs on the heap. There, growth is bounded by policy and by RAM. It is not bounded by a hard eight-megabyte wall that fails closed.

## How engineers actually reason about stack and heap

They decide placement by lifetime and size. Data that lives exactly as long as a function call and is small belongs on the stack. There it is free to allocate and free, and it stays cache-hot. Data whose size comes from input, or that must outlive the call, belongs on the heap. Someone must manage it.

They treat deep recursion as a stack-risk. Any parser, serializer, or walker that descends on input should be assumed to meet a pathologically deep input eventually. The design should move that depth to the heap. The crash from a stack overflow is a process death, not a recoverable error. It must be designed out, not caught.

They watch the heap for the slow failures. A stack overflow is loud and immediate. A heap leak is quiet and cumulative. It grows resident memory until the machine swaps or the OOM killer arrives. The address-space regions are observable. A steady climb in `VmData` or heap `RSS` in `smaps` is an early warning. The next chapter's tools can act on it.

They remember that the heap is shared and the stack is not. Concurrent code can allocate on the heap freely only because the allocator is built to be thread-safe. That safety has a cost that the stack never pays. Hot paths that can stay on the stack are faster for this reason.

## Thread-local storage and the per-thread memory region

The stack is not the only per-thread region. A variable declared with `thread_local` in C++ or `__thread` in C lives in a thread-local storage region. Any language's per-thread state does the same. The runtime allocates this region for each thread. It is separate from both the stack and the shared heap. Each thread sees its own copy under the same name. So the variable behaves like a global but is private to the thread. This is how libraries keep state without locks.

```mermaid
flowchart LR
    A[Thread 1] --> S1[Stack 1]
    A --> T1[TLS 1]
    B[Thread 2] --> S2[Stack 2]
    B --> T2[TLS 2]
    A --> H[Shared heap]
    B --> H
```

TLS sits between the stack and the heap in the layout. Like the stack, it is private and fast. But its lifetime matches the thread, not the function call. A thread that exits loses its TLS. This matters when reasoning about memory. A thread-local cache looks like a small per-thread allocation. Under many short-lived threads it multiplies. A leak held in TLS survives the function that created it for as long as the thread runs.

## Stack hardening: canaries, the stack protector, shadow stacks, and stack-clash protection

Because the stack holds return addresses and local buffers adjacent in memory, it has always been a prime target for memory corruption. The stack protector, enabled by `-fstack-protector`, places a random canary value on the stack frame. It checks the value before the function returns. If a buffer overflow wrote past the local variable into the canary or the return address, the check fails. The program aborts instead of jumping to attacker-controlled code.

```mermaid
flowchart LR
    F[Function frame] --> C[Canary value before return address]
    C --> R[Return address]
    Ret[On return] --> Check{Canary unchanged?}
    Check -->|no| Abort[Abort: stack smashed]
    Check -->|yes| OK[Return normally]
```

Modern CPUs add a hardware shadow stack. It is part of Intel CET and ARM Pointer Authentication. It keeps a separate, protected copy of return addresses. A buffer overflow cannot overwrite that copy. Even a corrupted stack return address is ignored in favor of the shadow copy. Stack-clash protection, `-fstack-clash-protection`, closes a different hole. An attacker who allocates a huge stack object can skip over the guard page and write into unrelated memory. The protection touches the stack in small steps so the guard page is always hit. Together these turn the stack from a fragile region into one that fails safely. They are standard in hardened production builds.

## Signal stacks, alloca, and special stack uses

A few stack uses deserve their own caution. `alloca` and variable-length arrays allocate within the current frame. They disappear when the function returns. But they consume stack space that the compiler may not have reserved against the limit. A large or attacker-influenced size is a direct path to overflow. They are best avoided in network-facing code. There the size can be driven by input.

Signal delivery is the other edge. When a signal such as `SIGSEGV` or `SIGBUS` arrives, the handler runs on the current stack by default. If the signal was caused by a stack overflow, the handler itself has no room and faults again. `sigaltstack` lets a process install a small dedicated alternate stack for signal handlers. A handler can run even when the normal stack is exhausted. This is how a service can log a crash or clean up instead of dying silently. For a backend engineer building robustness, an alternate signal stack plus a `SIGSEGV` handler that records the fault and exits is the difference between a mysterious disappearance and a useful crash report.

## Definitions

### The stack

> The stack is the per-thread region that backs function calls. A frame holds the return address, saved registers, and local variables. Each frame is given out automatically when a function is called and freed when it returns. The stack grows downward from near the top of the address space.

### A stack frame

> A stack frame is the slice of the stack that one function call uses. It holds the function's local variables, its saved state, and the return address. The return address tells the CPU where to go back to in the caller when the function ends.

### The heap

> The heap is the shared region for memory whose size or lifetime is not known at compile time. The kernel grants it in pages. A user-space allocator splits those pages into smaller blocks. The memory stays alive past the function that asked for it.

### Stack overflow

> Stack overflow is the failure that happens when the stack grows past its guard page and its available space. It usually comes from unbounded recursion or an over-large local variable. The system delivers it as a segmentation fault, which by default kills the process or thread.

### Fragmentation

> Fragmentation is the gap between the total free memory in the heap and the free memory you can actually use. It comes in two kinds. Internal fragmentation means the allocator gives you blocks larger than you asked for. External fragmentation means free pieces are scattered and cannot meet one contiguous request.

## Beyond the definitions

### Why is allocating on the stack so much faster than the heap

> The stack only moves a pointer to reserve or release a frame. It does no search, no bookkeeping, and no thread coordination. The heap must find a suitable free block, split it, and track it. It must also do this safely while many threads access it at once. That is real work on every allocation.

### Why does each thread need its own stack

> The stack holds the call chain and local state of one thread's execution. A single shared stack would let one thread's function calls overwrite another's. So each thread gets a private region with its own guard page. The heap stays shared because data often needs to be visible across threads.

### What is the guard page for

> The guard page is an unmapped page placed just past the stack's current limit. An access into it triggers a fault. The kernel handles the fault by mapping the next page and extending the stack. If there is no room to extend, the access becomes a fatal fault. That fatal fault is the stack-overflow signal.

### Why is deep recursion dangerous even when it looks correct

> Deep recursion's stack use grows with the depth of the input, not by a fixed amount. A parser may work on test inputs and still consume the entire fixed stack on a deeply nested hostile input. A correctness bug then becomes a process crash. It takes down the worker, not just the one request.

### How do Go and other managed runtimes differ here

> Managed runtimes still have a stack and a heap. But the runtime often grows stacks on its own and may move them. The heap is reclaimed by a garbage collector instead of explicit frees. The regions and their tradeoffs stay the same. The engineer no longer does manual frees and no longer sets a fixed stack size for ordinary depth.

## Common misconceptions

**"The heap is slow, so everything should be on the stack."** The stack is fast precisely because it cannot do what the heap does. Large or long-lived data cannot live there. Forcing it there causes overflow. The right move is to match lifetime to region, not to avoid the heap.

**"A stack overflow is a recoverable error."** It is a signal that by default kills the process or thread. You can install a handler. But the stack is already corrupted at that point, so the only safe response is often to exit. It must be designed out, not caught.

**"Raising the stack size fixes deep recursion."** It only raises the threshold. A larger input still overflows. A bigger stack per thread costs memory for every thread. Moving depth to the heap is the real fix.

**"The heap has no limit."** It is bounded by RAM, by swap, and by the address space. Allocations fail or trigger the OOM path when those are exhausted. The limit is softer than the stack's hard wall, but it is very real. Leaks find it slowly.

**"Local variables are always on the stack."** In most native code they are. But compilers may keep them in registers. Some runtimes and some optimizations place certain objects on the heap even though their scope looks local. The mental model holds for reasoning. But the details vary by language and optimization level.

## Summary

The stack and the heap are the two working regions of every process, and they have opposite strengths. The stack is fast, private, automatically reclaimed, and strictly bounded. It backs function calls and holds local state until the function returns. The heap is shared, large, long-lived, and managed by an allocator. That allocator pays for flexibility with bookkeeping and fragmentation. The boundary between them is where a program's correctness and its resource use are decided. Overflow the stack and the process dies loudly. Mismanage the heap and it dies slowly. The next chapter goes inside the heap to the allocator itself. That allocator is the machinery that turns kernel pages into the small blocks a program actually requests.
