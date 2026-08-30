---
mermaid: true
title: "System Calls: How Programs Request Kernel Services"
date: 2026-08-22
categories: ["System Engineering"]
tags: [linux, system-calls, syscall, file-descriptors, strace]
series: "System Engineering"
stage: "Stage 2 - Linux and Operating System Internals"
stage_order: 2
series_order: 2
---


> Stage 2 :  Linux and Operating System Internals  
> Subject area 2.1 :  The Operating System Model  
> Article 2

## The short version

A system call is a set way for a program to ask the kernel to do something it cannot do on its own. Think of it as a gate between the program and the kernel. The program passes in a number and some arguments. The CPU switches to a higher level of access. The kernel checks the program is allowed, checks the pointers it was given, does the work or lines it up for later, then hands back a result or an error.

Opening a file, starting a process, mapping memory, or reading from a socket all go through this gate. The gate matters because it links ordinary code to private state inside the kernel. The kernel cannot trust the pointers, lengths, or file descriptors the program hands it. For a backend service, the result of a `read` decides what happens next. A return of 0 means the end of the data. A short count means some bytes came back. A return of `-1` with `EAGAIN` means try again later or slow down.



## Where this article fits

The previous article showed the big picture of how an OS works. This article explains the *mechanism* behind it: the system call.

**Prerequisites:** What the Operating System Provides: the services that need a gate.  
**Next:** Linux Processes and Lifecycle: the resource that uses those gates to be created.

Later articles cover the things system calls work on: processes, memory, files, sockets, devices, and scheduling. This article shows the shared path they all use.

> Platform note: The register details below use **Linux x86-64** as the example (number in `rax`, arguments in `rdi/rsi/rdx/r10/r8/r9`). ARM64 puts the number in `x8` and arguments in `x0-x5`, and uses the `svc` instruction. macOS and Windows use different numbers. The main idea, a gate that checks every request, is the same on all of them.

## Why a program cannot simply call the kernel's functions

The kernel has powers that ordinary programs do not. It can change page tables, reach device hardware, decide which process runs next, read protected memory, and change files on disk.

If any program could call kernel functions directly or write into kernel memory, a single buggy or hostile program could take over the machine or break every other program. So the operating system offers a small, documented set of entry points instead of letting programs call kernel internals.


The system-call interface is a contract. It sets the operation number, what each argument means, how the result is returned, how errors are reported, and sometimes whether the call blocks or must happen in a certain order.

## A system call is not the same as a library function

Many programs call library functions instead of writing system-call instructions by hand. A library function may do any of these:

- Call one system call
- Combine several system calls
- Add buffering
- Convert data types
- Retry an interrupted operation
- Validate arguments at the library level
- Implement behavior entirely in user space

For example, `printf` is a library function. It turns values into text and may hold that text in a buffer before it finally uses the `write` system call. `fopen` is a C library function that reads the path and mode before it uses lower-level file operations. `memcpy` usually needs no system call at all, because it copies data inside the program's own memory.

```text
Application calls printf
        ↓
C library formats and buffers text
        ↓
Library eventually calls write
        ↓
Kernel validates the file descriptor and buffer
        ↓
Kernel writes to a file, pipe, terminal, or socket
```

This difference matters when you count system calls or measure speed. A program may run many library calls while making few system calls. Or one high-level operation may cause many system calls.

## The general system-call path

The exact steps depend on the operating system and the CPU, but the path usually looks like this:


The operation may finish at once. Or the process may have to wait for a device, file, socket, lock, timer, or some other event. If it waits, the scheduler can run another thread or process in the meantime.

## The user/kernel transition

A processor runs in different privilege modes. Code in user mode is limited. Code in kernel mode can do privileged work and reach protected kernel state.

On Linux x86-64, a program usually enters the kernel with the `syscall` instruction. That instruction sends control to an address the kernel set up, and it changes the CPU's mode. The kernel entry code saves enough of the program's state to return later, then switches to a kernel stack that is safe to use.

This change is not the same as a normal function call. A normal call stays inside the same program and the same privilege level. A system call crosses a protection line. It needs state handling and checks, and it may involve the scheduler or the hardware.

On Linux x86-64, the conventional raw syscall register arrangement is:

| Meaning | Register |
| --- | --- |
| System-call number | `rax` |
| First argument | `rdi` |
| Second argument | `rsi` |
| Third argument | `rdx` |
| Fourth argument | `r10` |
| Fifth argument | `r8` |
| Sixth argument | `r9` |
| Return value | `rax` |

The `syscall` instruction also changes registers such as `rcx` and `r11` in ways that depend on the CPU. These details belong to the Linux x86-64 ABI. Do not copy them straight into ARM64 code, another operating system, or a normal language function call.

The key point is not to memorize every register. It is to understand that the system-call ABI is a fixed contract between user-space code and the kernel.

## System-call numbers

The kernel needs to know which operation the program wants. A system-call number names that operation in the ABI.

For example, Linux gives numbers to operations such as `read`, `write`, `openat`, `mmap`, and `exit`. The numbers depend on the CPU architecture. The number for a call on x86-64 may differ on another architecture.

Libraries and language runtimes usually hide these numbers behind names. Code that puts a number straight into a register is tied closely to one operating system and one CPU family.

```text
System-call number + arguments
        ↓
Kernel dispatch table
        ↓
Implementation for that operation
```

The dispatch table is an internal kernel mechanism. It maps the requested number to the right handler. A program should normally use the documented library or syscall interface instead of relying on kernel-internal addresses.

## Arguments cross a trust boundary

System-call arguments come from the program, so the kernel must treat them as input it cannot trust. These arguments include numbers, file descriptors, flags, pointers, lengths, paths, structures, and arrays.

Here is a simple case. The program asks the kernel to read some data into a buffer the program owns. Before that read happens, the kernel has to confirm a few things:

- The buffer sits inside the program's own memory
- The length is a sensible number
- That memory can actually be written to
- The buffer is large enough for the operation
- The file descriptor points to something real and readable
- The program is allowed to do this at all

The kernel does not just follow the pointer the way an ordinary function would. It copies the data in carefully, or it checks the memory first. That way, a bad pointer cannot crash the machine or let one program read another's private data.


A hostile program may pass a bad pointer on purpose. A normal program may pass one by accident because of a bug. The kernel must handle both cases. It must not crash, and it must not leak protected data.

## Pointers can change while the kernel works

A pointer is not a permanent promise that the memory will stay the same. A program with many threads may change memory while another thread is mid-call. A signal handler or another operation may alter the state. A memory page may become unavailable or have its permissions changed.

The kernel must handle its access with care. It may copy the data into kernel-owned memory before using it. It may check again right before it touches the memory. Or it may use locking that stops unsafe changes.

This is one reason system-call interfaces use exact sizes and clearly defined structures. The kernel needs to know how much data it may read or write, and how to handle changes safely.

## Return values and errors

A system call returns a value that tells you whether it worked. What that value means depends on the call.

For `read`:

- A positive value means that many bytes were read.
- `0` means the end of the data for a file, or an orderly close for many sockets.
- `-1` from the C library wrapper indicates an error, with details available through `errno`.

For `write`:

- A positive value means that many bytes were accepted. It may be fewer than you asked for.
- `-1` indicates an error through the library interface.

For process creation, the return value may tell you which child was made or whether you are the parent or the child. For `mmap`, the return value is an address on success, or a sign of failure. The details are specific to each call.

The raw kernel rule and the C library rule are close but not the same. On Linux, a raw syscall often returns a negative number from the error range. The C library wrapper turns that into `-1` and puts the positive error number into `errno`.

Programs should use the documented wrapper or language API unless they have a clear reason to call the raw syscall interface.

## `read` and `write` are not guaranteed to process everything

A common beginner mistake is to assume that one `read` fills the whole buffer you asked for, or that one `write` sends every byte you requested.

The kernel may return a short result. Only some data may be ready. A pipe or socket may have little free space. A signal may have interrupted the call. Or the object may have a built-in boundary or limit.

```c
ssize_t total = 0;

while (total < wanted) {
    ssize_t n = read(fd, buffer + total, wanted - total);

    if (n > 0) {
        total += n;
        continue;
    }

    if (n == 0) {
        break; // End-of-file or orderly stream close.
    }

    if (errno == EINTR) {
        continue; // Retry if the operation was interrupted.
    }

    return -1; // A real error occurred.
}
```

This loop is only an example. Non-blocking descriptors, cancellation, deadlines, and message framing at the application level all need more decisions. The key lesson is that the return value tells you what actually happened, not what the caller hoped would happen.

## Blocking system calls

A blocking operation waits until it can move forward or until it fails. A blocking `read` may wait for data. A blocking `write` may wait for free buffer space. A blocking `accept` may wait for a new connection. A blocking lock may wait for ownership.

When a thread blocks, the operating system marks it as waiting and runs other work instead. Blocking is not automatically wasteful. It can be a simple and effective model when the program handles a manageable number of operations at once.

The trouble starts when a system holds too many resources while it waits. A request thread that waits on a slow dependency may hold onto memory, a connection, a transaction, and a queue slot. Enough blocked requests can exhaust the service even when the CPU is mostly idle.

Later networking articles will compare blocking, non-blocking, and event-driven I/O.

## Interrupted system calls

A signal can interrupt a system call while it waits. Depending on the call and how signals are set up, it may return an error such as `EINTR`, or it may be restarted automatically by the library or the kernel.

Your code must follow the contract for the specific call. Retrying every interrupted operation without thought can be wrong if the call had side effects or if your deadline already passed.

If a read has produced no data yet, retrying may be fine. But if an operation may have partly finished, the program must check the result and avoid doing the work twice.

This is another reason errors are part of the interface. A return value does not only say success or failure. It may tell you how far the operation got.

## A concrete example: `write`

Suppose a program wants to write some text to standard output.

At the application level, it may call:

```c
const char message[] = "hello\n";
write(STDOUT_FILENO, message, sizeof message - 1);
```

The program supplies:

- A file descriptor identifying standard output
- A pointer to the bytes
- The number of bytes it wants to write

The library or compiler exposes the call using the platform's ABI. The kernel checks the descriptor, confirms that the program's buffer can be read, and sends the data to the object behind the descriptor. That object could be a terminal, a pipe, a regular file, a socket, or redirected output.

The system call does not need to know that the program thinks of the destination as the screen. It works on the object the descriptor points to, which the kernel manages.

## File descriptors are capabilities within a process

A file descriptor is a small number that belongs to one process. It refers to an open object that the kernel manages. That object may be a file, socket, pipe, device, or another resource that behaves like a file.

The descriptor only means something inside the process that owns it. Another process cannot use the same number and expect it to point to the same object. Descriptors can pass to child processes when they are created, be copied, or be sent to another process through a special IPC mechanism.


This model explains two things. Closing a descriptor changes what later operations can do. And if a program leaks descriptors, it will eventually run out and new system calls will fail.

## Common system-call families

System calls are usually grouped by the resource they manage.

### Process and thread operations

These cover creating, replacing, waiting for, and ending execution. On Linux, calls such as `clone`, `fork`, `execve`, `wait4`, and `exit` handle the process lifecycle.

### File and filesystem operations

These cover opening paths, reading, writing, seeking, syncing, changing metadata, making directories, and removing names. Examples include `openat`, `read`, `write`, `lseek`, `fsync`, `stat`, and `rename`.

### Memory operations

These cover creating mappings, changing permissions, removing mappings, and telling the kernel how you plan to use memory. Examples include `mmap`, `munmap`, `mprotect`, and `madvise`.

### Networking operations

These cover creating sockets, binding addresses, listening, accepting connections, connecting, sending, receiving, and changing socket options. Some systems use separate calls. Others combine them through interfaces such as `socketcall` or related APIs.

### Information and synchronization

These cover reading clocks, waiting for events, changing scheduling settings, locking, and asking about process or resource state.

The names and exact interfaces differ across operating systems. The shared pattern is always a request to the kernel for a protected service.

## Observing system calls with `strace`

`strace` records the system calls a process makes on Linux. It can show the call name, its arguments, the return value, and timing information.

For a simple program:

```bash
strace ./program
```

A shortened trace might look like:

```text
openat(AT_FDCWD, "message.txt", O_RDONLY) = 3
read(3, "hello\n", 4096)                  = 6
close(3)                                  = 0
write(1, "hello\n", 6)                   = 6
exit_group(0)                             = ?
```

The real output depends on the program and system. The trace still shows several useful facts:

- The library opened a path and received descriptor 3.
- The process read 6 bytes from that descriptor.
- It closed the descriptor.
- It wrote 6 bytes to descriptor 1, standard output.
- The process exited.

Tracing helps because it shows what a program really asks the kernel to do. It can reveal unexpected file access, repeated operations, blocking calls, permission errors, and descriptor leaks.

It also adds overhead and may show sensitive arguments. Use it with care in production.

## System-call cost

A system call costs more than a normal in-process function call. It crosses a privilege boundary, saves and restores state, checks arguments, and may touch kernel data structures or devices.

The cost depends on the operation. A call that returns from a cache may be far cheaper than one that waits for storage. A call that blocks can involve scheduling and wake-up work. A call that moves a large buffer may spend most of its time copying data rather than entering the kernel.

Reducing system-call count can help when calls are small and frequent. Common techniques include:

- Buffering small writes
- Batching operations
- Reading or writing larger chunks
- Using vector operations
- Reusing connections and descriptors
- Using memory mappings where appropriate
- Using event-driven interfaces for many sockets

Fewer system calls are not always better. A large batch may raise latency or memory use. A memory mapping may make access easy but bring page faults and tricky lifetime rules. The optimization must fit the workload.

## A system call does not always mean a context switch

People often say that every system call causes a context switch. That is not quite right.

A system call does move the CPU from user mode to kernel mode and back. A context switch usually means changing which thread or process is running. That means saving one execution context and restoring another.

If a system call finishes at once, the same thread may enter the kernel and return without another thread running. If the call blocks, the scheduler may switch to another thread or process that can run. The two ideas are related but different.

This difference matters when you measure performance. The cost of crossing the privilege boundary and the cost of scheduling are separate parts of the path.

## Security at the system-call boundary

The system-call interface is a security boundary because it lets user-space code reach operations that change shared or privileged state.

The kernel must check:

- The process identity
- The requested operation
- File and device permissions
- Pointer validity
- Buffer lengths
- Flags and modes
- Resource limits
- Namespace or sandbox restrictions
- Whether the operation is allowed in the current context

A bug in argument checking can be worse than an app crash. It may let a user program read protected data, corrupt kernel state, or gain more privileges than it should have.

System-call filtering tools such as seccomp can limit which calls a process may make. Sandboxes and containers use several mechanisms together to limit what a process can see and do. Later security and container articles cover these topics.

## A realistic production example

Imagine a service with low CPU usage that still cannot keep up with incoming requests. Tracing shows that each request opens several files, does many small reads, and asks the kernel for status information again and again. The storage device is not full, but the process spends its time making system calls and waiting on small operations.

The team considers several changes:

1. Buffer small reads into larger operations.
2. Reuse open descriptors where the lifetime allows it.
3. Cache metadata that is stable enough to reuse.
4. Batch status checks.
5. Measure whether the workload is actually storage-bound or syscall-overhead-bound.

After batching, CPU usage may rise a little because each operation does more work. But request latency drops and throughput improves. The team still needs limits, so buffering does not grow memory without bound.

The key lesson is not to always reduce syscalls. It is that system-call traces reveal how application code really talks to the kernel, and the right fix depends on how the operation uses resources.

## How experienced engineers reason about system calls

When an operation behaves unexpectedly, experienced engineers ask:

- Which system call or library operation is involved?
- Is the library making more calls than expected?
- What arguments and buffers cross the boundary?
- What does the return value actually mean?
- Can the call return a partial result?
- Can it block or be interrupted?
- Which resource or permission can make it fail?
- Does the call create or consume a handle?
- Does it affect persistent or external state?
- Is the behavior portable or platform-specific?

They start with the simplest explanation that still fits the facts. Then they look at lower layers when the high-level view no longer explains the result. `strace`, a debugger, metrics, logs, and source code each answer a different part of the question.

## The virtual dynamic shared object lets some calls skip the trap

The kernel maps a small read-only page of code into every process. This page is called the vDSO, short for virtual dynamic shared object. For operations such as `clock_gettime` and `gettimeofday`, the kernel keeps the current time in a memory spot that user space can read directly. The wrapper can then work out the answer without running a syscall instruction or entering privileged mode.

This matters because time calls happen very often. A busy server may ask for the time on every request to label logs or enforce deadlines. If each of those calls trapped to the kernel, the cost would outweigh the small work being done. The vDSO turns a possible trap into a plain memory read plus a little math. That is often hundreds of times faster.

Not every call can use this path. Only operations whose answer the kernel can safely share with user space, with no extra checks and no side effects, belong in the vDSO. Anything that changes state, touches a device, or depends on who you are still needs the real gate.

## io_uring offers an asynchronous alternative to the synchronous gate

Most of this article describes the synchronous model: you make one call, and it returns when the kernel has an answer, possibly after blocking. io_uring is a Linux interface that changes how the interaction works. Instead of trapping for each operation, the program and kernel share two ring buffers in memory: a submission queue and a completion queue. The program puts requests into the submission ring, then later collects the finished ones from the completion ring. They talk through ordinary memory instead of a syscall for every step.

The benefit shows up when a workload makes many I/O operations and does not want to pay a transition for each one, or run many threads, or run an event loop with epoll. A single `io_uring_enter` call can submit dozens of operations and collect their completions. That collapses the cost of crossing the boundary per operation. It also supports true async operations, such as buffered reads that would otherwise block.

io_uring is not free and is not always the right tool. The shared rings add setup, memory, and kernel-version dependencies, and early versions had a larger attack surface. Use it when syscall frequency or blocking is the real bottleneck, not as a default for every program.

## The cost of crossing the boundary is more than the instruction itself

The instruction to enter the kernel is cheap on its own, but the work around it is what costs you. The CPU must switch privilege mode, save the user registers it will overwrite, and set up a kernel stack. On return it restores state and switches back. None of that is free next to a plain function call that stays in the same context.

Beyond the register and mode work, the switch disturbs the hardware caches and the translation lookaside buffer. Kernel code touches different memory than your function did. So entering the kernel can push out cache lines your program wanted, and the TLB entries that map your pages may be cold when you return. For very small, frequent calls, the data movement and cache effects can cost more than the instruction itself.

This is why batching helps. When one larger read or write replaces many small ones, you pay the transition a few times instead of thousands, and you keep more of your working set in cache between calls.

## seccomp filters the gate before the kernel validates arguments

seccomp is a kernel feature that filters which system calls a process may make. The detail that matters for a systems engineer is where it sits in the path. A seccomp filter runs once the syscall number is known, but before the kernel's normal argument checks and handler. It can reject the call based on the number and even on specific argument values.

The value is containment. A web server that only needs to read files, accept connections, and write logs can get a policy that blocks everything else, including calls that might appear in an exploit. If an attacker finds a memory bug and tries a dangerous operation, the filter denies it at the gate instead of letting the kernel's checks decide.

seccomp is one layer, not a full sandbox. It limits the interface but does not by itself restrict filesystem paths, network destinations, or resource use. Production sandboxes combine it with namespaces, capabilities, and resource limits. Later security articles cover these in more detail.

## Tracing with strace has a cost that perf trace can reduce

The strace section earlier showed how useful a trace is for seeing what a program asks of the kernel. What it did not stress is the cost. strace works by asking the kernel to stop the process on every syscall entry and exit so the tracer can look at it. That forced stop and the context interaction can slow a program by ten times or more. It also changes timing enough to hide or create races.

For lighter observation, perf trace uses the kernel's tracing system to record syscalls with far less interference. It samples and reports without stopping the target on every call. So it fits better in production-adjacent measurement, where you need totals and latency rather than a precise per-call argument dump. The lesson is the same as with all profiling: use the heaviest tool only when you need its detail, and reach for the lighter one when you only need the overall shape.

## Interview definitions

### What is a system call?

> A system call is a controlled entry point that lets a program ask the privileged kernel to do a service for it.

### Why are system calls needed?

> Programs need system calls to do things that need kernel-managed resources or privileges, such as reading files, creating processes, using sockets, and managing memory mappings.

### What happens during a system call?

> The program sets up a syscall number and arguments, enters the kernel through a protected CPU transition, and the kernel checks the request, does the work or lines it up for later, then returns a result or an error.

### What is the difference between a system call and a library call?

> A library call stays in user space unless it decides to enter the kernel. A system call crosses into the privileged kernel. A library function may wrap one system call, combine several, add buffering, or do its work entirely in user space.

### Why does the kernel validate user pointers?

> User pointers may be invalid, point outside the process, have the wrong permissions, or be set up on purpose to do harm. Checking them prevents crashes, memory corruption, and unauthorized access to kernel state.

### What is a short read or short write?

> A short read or write is a successful operation that moves fewer bytes than you asked for. The caller must use the returned count and continue or finish following the operation's contract.

## Interview follow-up questions

### Does every system call cause a context switch?

> Every system call crosses from user mode to kernel mode, but it does not necessarily switch to another thread or process. A context switch may happen if the call blocks or the scheduler picks something else.

### Why can a system call block?

> It can block when the resource it needs is not ready. Data may not have arrived on a socket yet. Buffer space may not be free. A lock may be held by someone else. Or storage I/O may still be running.

### How does `errno` work?

> In the common C library interface, a failing wrapper returns `-1` and stores an error number in thread-local `errno`. The caller must check it following the specific call's contract and should not assume every error can be retried.

### Why can `read` return fewer bytes than requested?

> The available data may be smaller than you asked for, the source may reach end-of-file, a stream may deliver data in pieces, or an interruption may occur. The return value tells the caller how much actually moved.

### How would you reduce system-call overhead?

> I would first measure the call pattern. Depending on the workload, I might buffer or batch small operations, reuse descriptors, use vector I/O, cut unnecessary metadata calls, or pick a fitting event-driven interface. I would check that the change does not raise memory use, latency, or failure complexity.

### Why is the system-call boundary important for security?

> It is where untrusted user-space input turns into a request to change protected or shared state. The kernel must check arguments, identity, permissions, and resource limits before it does the operation.

## Common misconceptions

### “Every library function is a system call.”

Many library functions run entirely in user space. Others call the kernel only when they must, or they group several application operations into fewer system calls.

### “A successful system call completed all requested work.”

Some calls return partial progress. A successful return may mean only part of a buffer was read or written, or that an operation was accepted to be finished later.

### “A system call is just a slow function call.”

A system call crosses a privilege boundary, requires checks, and may touch kernel state, devices, scheduling, and blocking behavior. Its cost and meaning are different from an ordinary function call.

### “The kernel can trust pointers from a process.”

Pointers and lengths come from user space and must be treated as input the kernel cannot trust. They may be invalid or built on purpose to do harm.

### “If a call timed out, the operation did not happen.”

A timeout tells the caller that no result arrived before the deadline. The operation may still have finished on the remote side, or it may keep running after the caller stops waiting.

## Summary

A system call is the protected path from user-space code to kernel-managed services. The program supplies a syscall number and arguments, the processor enters privileged kernel code, and the kernel checks, performs, or schedules the operation before returning a result.

System calls have precise contracts about arguments, pointers, lengths, return values, partial progress, blocking, interruption, and errors. The C library or language runtime may wrap those calls with buffering, conversion, retries, or higher-level behavior.

The system-call boundary is both a performance boundary and a security boundary. It can add transition and checking costs, and it must stop untrusted programs from corrupting protected state. Understanding the boundary makes tools such as `strace` far more useful, and it prepares us to study processes, files, memory, networking, and devices in detail.

## If you want to build this later

Build a small Linux system-call observability tool.

Start with a program that opens a file, reads it in chunks, writes the data to standard output, and closes the file. Trace it with `strace` and compare the operations you wrote with the actual calls. Then add buffering, change the chunk size, add an error path, and watch how the trace changes.

The goal is to see the difference between library code and kernel requests, understand return values and cleanup, and connect system-call count with performance and resource behavior.

