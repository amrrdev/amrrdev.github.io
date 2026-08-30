---
mermaid: true
title: "Linux Processes and Lifecycle"
date: 2026-08-23
categories: ["System Engineering"]
tags: [linux, processes, fork, exec, wait, pid, copy-on-write, zombie]
series: "System Engineering"
stage: "Stage 2 - Linux and Operating System Internals"
stage_order: 2
series_order: 3
---

> Stage 2 :  Linux and Operating System Internals  
> Subject area 2.1 :  The Operating System Model  
> Article 3

## The short version

A process is one running copy of a program. It owns its own memory, its threads, its identity, and the resources it has opened. Think of it as a box that holds a running program. The kernel creates the box, runs it, and cleans it up when done.

Linux makes a process with `fork` or `clone`, swaps in new code with `exec`, and cleans up after it with `wait`. One program file can turn into many processes, each with its own ID, arguments, and resources. A child that exits becomes a zombie (a leftover table entry) unless its parent collects the exit status. A child whose parent dies first becomes an orphan and is handed to another parent. In a backend service, if you forget to `wait`, zombies pile up and fill the process table. And if you fork without `close-on-exec`, open files can leak into the next program.

The central lifecycle is:

```text
Create (fork) → Run → Replace (exec) → Exit → Reap (wait)
```

## Where this article fits

The previous article showed how programs enter the kernel through system calls. This article explains the life of the thing that makes those calls: the process.

**Prerequisites:** System Calls :  how a program crosses into the kernel.  
**Next:** Linux Signals and Service Supervision :  how the kernel or another process notifies a process and how a service is supervised.

Later articles look at scheduling, memory, files, and concurrency inside this lifecycle.

## A program is not a process

A program is a file that holds instructions and data. A process is that program while it is running.

The same executable can produce many independent processes:


The executable file is not the process's complete state. The process also has:

- A process identifier
- An address space
- One or more threads
- A current working directory
- Environment variables
- Command-line arguments
- User and group identity
- Signal dispositions and masks
- Open file descriptors
- Resource limits
- Scheduling state
- Parent and child relationships

Two processes running the same program can act differently because their state differs.

## Process identifiers and relationships

Linux gives each process a number called a PID (process identifier). Other processes use that number to send signals, wait for exit, check status, or apply limits.

Processes relate to each other. The process that creates another is the parent. The new one is the child.


This matters because the parent must collect the child's exit status. The relationship also affects signals, process groups, job control, and what happens when the parent exits first.

A PID only means something inside a PID namespace. A container can use a namespace so a process sees different numbers than the host does. The process still has a real ID on the host, but the number it sees depends on its namespace.

## What Linux creates when it creates a process

Making a process takes more than picking a PID. Linux must set up the execution state, memory maps, credentials, open files, signal state, scheduling info, and links to other processes.

Some of these resources can be shared between related processes. Others are copied or made fresh. The details depend on which creation call and flags you use.

The traditional Unix model is often explained with two operations:

1. `fork` creates a child process based on the calling process.
2. `exec` replaces the current process image with another program.

Linux also has `clone` and related calls. They can make a process or a thread and let you pick which parts of the state are shared. Still, the simple `fork` then `exec` model is the best place to start.

## `fork`: creating a child

`fork` makes a child process that starts out like the parent. The child gets a new PID and starts running right after the `fork` call returns.

The call returns different values:

- In the child, it returns `0`.
- In the parent, it returns the child's PID.
- If creation fails, the parent receives `-1` and an error.

```c
pid_t child = fork();

if (child < 0) {
    perror("fork");
    return 1;
}

if (child == 0) {
    // Child process.
    printf("child: pid=%ld\n", (long)getpid());
} else {
    // Parent process.
    printf("parent: child pid=%ld\n", (long)child);
}
```

The return value tells the parent and child apart. The parent gets the child's PID; the child gets zero. That is how a shell knows which branch becomes the command. In theory `fork` copies the whole address space. In practice the kernel uses copy-on-write so this stays cheap until the child calls `exec`. If you run the program, you should see two lines: one from the parent with the child's PID, and one from the child with its own PID.

After `fork`, both processes keep running from the same spot in the code, but they are separate. Two variables that happen to hold the same value are not shared memory just because they matched before the fork.

### Copy-on-write

Copying the whole address space up front would be slow. Linux usually uses copy-on-write instead. The parent and child first point to the same physical memory. When one of them writes, the kernel makes a private copy for that process.


Copy-on-write makes process creation cheaper when the child soon calls `exec` and swaps out its memory. It still costs something: page-table work, memory pressure when pages change, and extra trouble for large or memory-heavy processes.

## `exec`: replacing the process image

An `exec` call loads a new program into the current process and replaces the old one. It does not make a new PID. The process keeps its identity, but its code, data, stack, and other image state are swapped out.


This split is useful. A shell can make a child and then turn that child into any command. The shell keeps running while the child runs the chosen program.

The `exec` family differs in how you pass the program path, arguments, and environment. When `exec` succeeds, it does not return to the old program, because that program is gone. If `exec` returns at all, it failed, and the child must handle the error.

## File descriptors across `fork` and `exec`

Open file descriptors are part of a process. After `fork`, the child normally inherits them. The child's descriptors point to the same open files as the parent's.

This is how a shell can create a pipeline:

```text
process A stdout → pipe write end
process B stdin  → pipe read end
```

The parent makes a pipe, forks the children, and wires each child's standard input or output with `dup2`. The children then call `exec` to become the commands.

Descriptors can also stay open across `exec` unless they have the close-on-exec flag. If they are inherited by accident, a pipe may never reach end-of-file, a file may stay open too long, or a new program may get a resource it should not touch.

So the close-on-exec rule is both a correctness and a security concern.

## `wait`: collecting a child

When a child exits, the kernel records its exit status and keeps a small table entry until the parent collects it. The parent uses `wait`, `waitpid`, or a similar call to collect that state.

```c
int status;
pid_t result = waitpid(child, &status, 0);

if (result == -1) {
    perror("waitpid");
    return 1;
}

if (WIFEXITED(status)) {
    printf("child exit code: %d\n", WEXITSTATUS(status));
} else if (WIFSIGNALED(status)) {
    printf("child was terminated by signal %d\n", WTERMSIG(status));
}
```

This call lets the parent collect the exit status so the child does not stay a zombie. `WIFEXITED` and `WIFSIGNALED` tell the parent whether the child exited normally or was killed by a signal. A supervisor uses that to decide whether to restart. When you run it, it prints the exit code or the signal number.

The parent can wait for one specific child, wait for any child, or use a non-blocking mode to check whether a child changed state.

## Zombies and orphans

A zombie is a child that has exited but whose parent has not collected its exit status yet. The child is no longer running and its memory is freed. Still, the kernel keeps a small record so the parent can learn how it ended.

```text
Child exits
    ↓
Kernel releases most child resources
    ↓
Exit status remains
    ↓
Parent calls waitpid
    ↓
Zombie record is removed
```

A few short-lived zombies are usually harmless. But a parent that keeps making children and never reaps them can fill the process table.

An orphan is a process whose parent has exited. Linux gives it a new parent, often PID 1 inside its namespace, or a configured subreaper. The new parent can then supervise or reap it.

Zombies and orphans are different:

- A zombie has exited but has not been reaped.
- An orphan is still running but has lost its original parent.

Mixing up these two terms can lead to wrong debugging and cleanup choices.

## Exit status

A process can exit normally with a number, or it can be killed by a signal. The parent can check which one happened.

Exit codes are not a full error system, but they help scripts, service managers, and parents. A program should pick meaningful codes and not treat every nonzero code as the same failure.

When a signal kills a process, the parent can see which signal. A service manager can use that to tell a planned stop from a crash or a resource problem.

## A realistic production example

A backend job runner forks one worker per job but forgets to `wait` in the parent when a job finishes fast. Under load, `ps` shows many `defunct` entries. New `fork` calls start failing with `EAGAIN`. The reason is not low memory. The process table is full of zombies.

The fix is not to raise `RLIMIT_NPROC`. The fix is to reap every child, including the success path. Use `fork`, then child `exec` worker, then parent `waitpid`, with `SIGCHLD` handling. Also watch the `ps stat=Z` count. The next article adds `SIGTERM` handling so workers stop cleanly instead of being killed.

## How experienced engineers investigate a process problem

When a process behaves unexpectedly, experienced engineers check:

- Is the process running, sleeping, or becoming a zombie (`ps -o stat`)?
- What is its PID, parent PID, and PID namespace (`/proc/<pid>/status`)?
- Which file descriptors are inherited across `exec` (`/proc/<pid>/fd`, `lsof`)?
- Is the parent reaping children (`strace -e waitpid`, `pstree`)?

Tools like `ps`, `/proc`, `pstree`, `lsof`, and `strace` answer different parts of the question. Their output only helps once you tie it to a lifecycle guess.

## Forking inside a multithreaded program and the limits of what is safe

You can call `fork` from a thread that is not the main one, but this is risky. Only the calling thread survives in the child. Every other thread disappears. If another thread held a lock when `fork` was called, that lock still looks taken in the child, but no thread exists to free it. Any function that needs that lock then hangs. The standard rule is strict. Between `fork` and a later `exec`, the child may call only async-signal-safe functions. `printf`, `malloc`, and most library calls are not on that list. So a child that logs or allocates memory before `exec` can hang without warning. This is why many programs fork only from one dedicated thread, or skip `fork` entirely in threaded services.

## posix_spawn as a safer and faster route than fork and exec

`posix_spawn` and `posix_spawnp` let a process make a child that runs a new program without your code ever running in the child. Instead of forking and then calling `exec` after your own setup, you describe the attributes, file actions, and environment in a spawn object. The C library or kernel then does the switch. Because the child never runs your code, the async-signal-safe hazard of a threaded `fork` goes away. On Linux the call can use `clone` or a `vfork`-like fast path, so it is often faster than the old `fork` plus `exec` pair. The trade-off is less control. You give setup as data, not as steps, which suits plain command launches better than complex pipelines.

## What a process sees inside a PID namespace, and why PID 1 matters

A PID namespace gives a group of processes its own numbering that starts at 1. A container uses such a namespace, so the process it sees as PID 1 may really be PID 4123 on the host. The host still tracks the real numbers, but programs inside the namespace see the in-namespace view. That makes in-namespace PID 1 special. It acts as the init for that tree and must reap orphaned children, because no higher parent exists. If that PID 1 exits, the kernel treats the namespace as dead and kills the rest. So a service manager running as PID 1 must start children and also collect their exit status, or zombies build up inside the namespace just like they would on a host.

## Detaching from the terminal through setsid when a service becomes a daemon

A long-running service often needs to detach from the terminal that started it. If it does not, closing that terminal can send it a hangup signal or end its life with the login session. The tool for this is `setsid`. It makes a new session and a new process group, with the calling process as leader and with no controlling terminal. A common daemon setup forks, lets the parent exit, has the child call `setsid`, then forks again so the daemon is not a session leader that could grab a terminal later. After that, standard input, output, and error are sent to logs or `/dev/null`, so the process no longer needs the original shell. The result is a process that the service manager supervises and restarts, not the user's session.

## How the kernel reads the shebang line and sets ARGV[0] during execve

When `execve` loads a file, it first reads the first two bytes. If they are `#!`, the kernel treats the rest of that line as an interpreter path and an optional argument, then runs the interpreter with the script path added to its arguments. That is why a script that starts with `#!/bin/sh` needs no extension or explicit interpreter on the command line. The kernel also sets `ARGV[0]`, the first argument a program sees. Normally this is the program name, but `execve` lets the caller set it to anything. That is how `ps` can show names like `(python)` or a custom label. If a program trusts `ARGV[0]` for identity or security, it should remember that the caller controls it, so it is not a safe credential.

## Interview definitions

### What is a process?

> A process is one running copy of a program. It has its own memory, execution state, identity, and resources managed by the kernel. One executable can become many processes, each with its own PID, arguments, environment, and open files.

### What does `fork` do?

> `fork` makes a child process that starts out like the parent. In the child it returns zero; in the parent it returns the child's PID.

### What does `exec` do?

> `exec` swaps the current process image for a new program. The PID stays the same, but the old code and data are gone.

### What does `waitpid` do?

> `waitpid` lets a parent collect a child's exit status so the child does not stay a zombie. A zombie has exited but not been reaped. An orphan is still running after its parent exited, and it has been given a new parent.

### Why can a restart policy hide a bug?

> If a service keeps failing for the same reason, restarting it at once just repeats the failure, fills the logs, and hides the real config error. A delay with `RestartSec`, plus different exit codes for config errors versus short-lived ones, stops the loop.

## Interview follow-up questions

### Why are `fork` and `exec` separate?

> Keeping them separate lets a parent set up the child's descriptors, environment, and process group with `dup2`/`setpgid` before the child becomes the wanted program. This is how shells build pipelines.

### Does `exec` create a new PID?

> No. It swaps the image in place. `fork` is what creates the new PID.

### Why do zombies exist?

> The kernel keeps a small exit record until `wait`, so the parent can learn the exit code or the signal that ended it.

## Common misconceptions

### “`fork` copies all memory immediately.”

Copy-on-write shares pages until a write happens. The logical spaces are separate, but the physical pages are shared for a while.

### “`exec` starts a child.”

It swaps the current image. It does not make a new PID.

### “A zombie holds all its memory.”

Most resources are freed. Only a small exit record remains. But many zombies can fill the process table.

## Summary

A process is the kernel's unit of life. `fork` creates it, `exec` swaps its code, `wait` cleans it up. Copy-on-write, descriptor inheritance with `close-on-exec`, and zombie and orphan handling decide whether a backend leaks resources or cleans them up. The next layer is notification. Signals and supervision decide how a process is asked to stop.

## If you want to build this later

Build a tiny shell that runs one `fork` then `exec` then `wait` pipeline. Start with a single command. Then check `close-on-exec` by listing `/proc/self/fd` before and after `exec`. Next, add a bug where the parent skips `wait` on success, and watch the zombies with `ps`. Fix it by adding `waitpid(-1, &status, WNOHANG)` in a loop to reap all children. This ties together lifecycle, descriptor inheritance, and reaping before you add signals in the next article.

