---
mermaid: true
title: "Linux Signals and Service Supervision"
date: 2026-08-23
categories: ["System Engineering"]
tags: [linux, signals, sigterm, supervision, systemd, graceful-shutdown]
series: "System Engineering"
stage: "Stage 2 - Linux and Operating System Internals"
stage_order: 2
series_order: 4
---

> Stage 2 :  Linux and Operating System Internals  
> Subject area 2.1 :  The Operating System Model  
> Article 4

## The short version

A process does not just exit when its work is done. It can also be told to stop from outside at any time. That outside message is called a signal. A signal is small and carries almost no data. It can arrive at any point, even between two steps of the program.

A signal can mean many things. A user may have pressed Ctrl-C. A child process may have finished. A program may have read memory the wrong way. Or a supervisor may want the service to stop. The signal that matters most for a backend is `SIGTERM`. It asks the service to stop, but it does not force it. The service should finish its current work, stop taking new work, and exit before a deadline. If it is too slow, the supervisor sends `SIGKILL`. That signal cannot be caught, and it ends the process right away.

The supervisor is the program that keeps a service running. On Linux this is often `systemd`. It starts the service, restarts it if it fails, sets its limits, and sends the signal to stop it. If a service handles `SIGTERM` wrong, it can look alive while it is actually dropping requests. If the supervisor restarts too fast, a simple config error can turn into a crash loop.

Here is a simple way to think about shutdown. `SIGTERM` should start a drain, not stop the service at once. Close the door to new connections. Let the requests that are already running finish or cancel within a timeout. Save anything that must be saved. Then exit. The supervisor waits until `TimeoutStopSec` passes, and only then does it send `SIGKILL`.

## Where this article fits

The last article followed a process from birth to death: `fork`, `exec`, `wait`, and the cleanup of finished children. This article is about the events that break into that lifecycle.

You need this to read process state in `/proc`, to find out why a container will not stop, and to build a backend that you can deploy without losing traffic. Scheduling, filesystem views, and resource limits all depend on one idea: a service is a supervised group of processes, not just one PID.

## Signals are notices that arrive at any time

Many things can send a signal. The kernel can send it. Another process can send it with `kill`. A terminal can send it when you press Ctrl-C. A process can even send one to itself. A signal is not like a message in a queue. It carries no payload. It is only a number that means something.

Some signals show up so often that you should know them. `SIGINT` is what the terminal sends when you press Ctrl-C. `SIGTERM` is the polite request to stop. `SIGKILL` and `SIGSTOP` are the two signals you cannot catch or ignore. `SIGHUP` once meant a terminal was closed; many services now use it to reload their config. `SIGCHLD` tells a parent that a child changed state. `SIGSEGV` usually means the program touched memory it should not. `SIGPIPE` often means you wrote to a pipe whose reader went away. `SIGUSR1` and `SIGUSR2` are free for any program to define.

A signal is not a reliable message queue. If the same signal is sent twice in a row, the kernel may merge them and deliver just one. What happens by default depends on the signal. It may end the process, stop it, resume it, ignore it, or dump core. Because the signal can arrive at any time, it may show up while the program holds a lock or is partway through updating a structure.

## What a process does with a signal

For each signal, a process has a disposition. That is the plan for what to do when the signal arrives. The plan can be the default action, it can be to ignore the signal, or it can be a handler function that the kernel runs. A process can also block a signal. A blocked signal does not vanish. It waits as pending and is delivered once the process unblocks it. This helps when a short piece of code must not be interrupted, such as when you update a global list of children.


The diagram below shows the path, but the words after it matter more. Blocking a signal does not lose it. It only delays it. Threads make this trickier. A signal can target one thread, or it can target the whole process. In the second case, the kernel picks any thread that is allowed to take it. So do not assume that `SIGTERM` sent to a multi-threaded HTTP server lands on the thread that runs your handler.

## Signal handlers must do almost nothing

A signal can arrive between any two instructions. The handler then runs in a context where most library functions are not safe. If you call `printf`, allocate memory, or take a mutex inside the handler, you can corrupt the very state the interrupted code was using.

The safe pattern is to do almost nothing in the handler and let the main loop do the work. Usually that means setting a flag of type `volatile sig_atomic_t`, or writing a single byte to a `self-pipe` that the main loop watches.

The program below shows the flag pattern.

```c
static volatile sig_atomic_t stop_requested = 0;

static void handle_term(int signal_number) {
    (void)signal_number;
    stop_requested = 1;
}

int main(void) {
    struct sigaction action = {0};
    action.sa_handler = handle_term;
    sigemptyset(&action.sa_mask);
    sigaction(SIGTERM, &action, NULL);

    while (!stop_requested) {
        do_one_unit_of_work();
    }

    release_resources_and_exit();
}
```

The handler only records that a stop was asked for. The loop sees the flag on its next pass, stops taking new work, and cleans up in normal code where `printf` and locks are safe. In a real program you must also plan for blocking calls like `read` or `accept`. A signal may make them return with `errno == EINTR`. Or you may use `pselect` or `signalfd` so the wait can be broken cleanly. The rule stays the same. Keep the async part tiny and move decisions into the normal loop.

## `SIGTERM` and graceful shutdown

When a supervisor wants a service to stop, it sends `SIGTERM`. The word asks matters. `SIGTERM` does not mean the process must vanish at once. It means the process should begin to shut down.

A graceful shutdown is a small protocol between the manager and the service. The manager sends `SIGTERM` and then waits.


For a backend, this means a few clear steps. Close the listening socket so the load balancer sends new requests somewhere else. That is why a readiness probe should start failing the moment `SIGTERM` arrives. Give in-flight requests a deadline shorter than the supervisor's `TimeoutStopSec`, so they can finish or cancel in time. In HTTP handlers, that is where `context.WithTimeout` belongs. Tell background workers to stop taking new jobs, flush what must be saved, and close file descriptors. If the service needs more time than allowed, the manager sends `SIGKILL` and the rest of the state is lost. So each stage needs its own timeout.

## `SIGKILL` is not graceful

`SIGKILL` ends a process right away. The kernel still frees memory and file descriptors, but the program's own cleanup never runs. Buffered logs may not be flushed. Temporary files can be left behind. A message that was sent but not yet acknowledged stays uncertain.

It is the right tool when a process is stuck and ignores `SIGTERM`. But it should not be the normal way to stop a service. A deploy that uses `kill -9` all the time is hiding a shutdown bug.

## Signals are not a message bus

Signals work well for small, rare notifications. Asking a process to stop soon, to reload config, to reap children, or to dump diagnostics all fit this. They are not good for sending work details, payloads, or ordered commands. A program that needs those should use a pipe, a socket, or a queue, where bytes are buffered, ordered, and acknowledged. If you need backpressure, signals cannot give it to you.

## Process groups and sessions

A process group is a set of processes that the shell manages as one unit. A session is a set of process groups, usually tied to a terminal. When you run a pipeline in a shell and press Ctrl-C, the terminal sends `SIGINT` to the whole foreground process group, not to one PID.


The same idea matters for a supervisor. Suppose a service started several workers and you send `SIGTERM` only to the parent. The workers can keep running and hold the listening port. The supervisor should send the signal to the process group, or better, track the whole control group so it can stop everything that belongs to the service.

## Daemons and long-running services

A daemon is just a long-running service with no terminal, like a web server or a scheduler. Old guides tell you to double-fork, make a new session, change directories, and close the standard streams. On modern Linux, the service manager already does all that. You get more value by making the service itself behave well. Start with known config and check it at boot. Report a clear error if startup fails. Handle `SIGTERM` and `SIGHUP`. Expose logs and a readiness endpoint. Drain on shutdown and exit with a status that tells the manager whether to restart.

## Service supervision

On most systems today, `systemd` is that manager. It sets up the environment, decides the restart policy, sets resource limits, orders dependencies, handles logging, and sets a deadline for shutdown.

A small unit file shows the important fields.

```ini
[Unit]
Description=Example worker
After=network-online.target

[Service]
ExecStart=/usr/local/bin/example-worker
Restart=on-failure
RestartSec=2
TimeoutStopSec=30

[Install]
WantedBy=multi-user.target
```

This file starts the binary, restarts it a couple seconds after a failure, and waits up to thirty seconds for it to stop after `SIGTERM`. The unit does not make the program graceful by itself. The program must still handle the signal, stop new work, and exit before the deadline.

## Crash loops

A restart policy helps when a failure is temporary, such as a downstream service that is briefly down. It hurts when the failure is permanent, such as a bad config file. Then the service starts, fails for the same reason, exits, and is restarted at once. It wastes CPU, fills the logs, and hides the original error.

A good supervisor adds a delay and a rate limit. A good service uses different exit codes for different cases. A code that means a config error should tell the manager not to restart. A code that means a brief error can allow a restart.

The same check applies to health probes. A liveness probe that restarts a container while it is still warming up can create the very loop it was meant to cure.

## When a parent dies and children are left behind

When a parent exits, its children do not always exit too. Linux gives them a new parent: PID 1, or a subreaper set up in that PID namespace. An orphaned worker can keep serving traffic, hold a port, or keep files open after the service that made it is gone. This is why checking only one PID is not enough. The supervisor should know which control group or process group belongs to the service, and it should be able to stop the whole group. Health checks should test whether the service can actually handle a request, not just whether a process is alive.

## A realistic production example

A service got `SIGTERM` during a rolling deploy and exited at once because it had no handler. It took the default action, so the HTTP handlers that were running were cut off. Messages that lived only in memory were lost. The clients retried. Those retries created duplicate work while the new replica was still starting up, and a downstream connection pool spiked.

The team made shutdown explicit. On `SIGTERM`, the service marked itself not ready so the load balancer would drain it. It stopped the accept loop. It gave in-flight requests a deadline a few seconds shorter than `TimeoutStopSec`. It told background workers to stop taking new jobs, flushed important state to durable storage, closed listeners and idle connections, and then exited. When the next deploy sent `SIGTERM`, traffic drained instead of dropping. The change was not about adding a new library. It was about treating termination as a protocol with a deadline.

## How experienced engineers investigate

When a process misbehaves, they rarely look at signals first. They start with the lifecycle they already know. Is the process running, sleeping, stopped, or a zombie? What are its PID, parent PID, and process group? Which user and groups does it run as? Which file descriptors and sockets does it hold? How much CPU and memory does it use? Are any signals blocked or pending in `/proc/<pid>/status`? Does it have children that outlived it, as seen in `pstree`? Is a service manager restarting it, as seen in `systemctl status` and `journalctl -u example-worker`?

The tools only help when you have a guess to test. If you know `SIGTERM` was delivered but the handler blocked on `accept` without handling `EINTR`, you can see why the process did not stop. Just seeing that the PID exists tells you nothing.

## Standard signals and real-time signals

Linux splits signals into two classes. The standard signals run from SIGHUP (1) to SIGSYS (usually 31). The kernel treats them as a bitmap of pending notifications. If the same standard signal arrives while it is already pending, the kernel does not queue a second copy. It keeps the one pending bit set, so the handler runs just once. That is why rapid repeated SIGTERMs can collapse into a single delivery.

The real-time signals start at SIGRTMIN and run to SIGRTMAX, which is usually SIGRTMIN plus 31. These signals are queued. Each send adds a separate entry to the pending queue, so if you send SIGRTMIN three times you get three deliveries, up to the limit RLIMIT_SIGPENDING. Real-time signals also carry an extra integer value through sigqueue. That makes them the closest thing Unix has to a small out-of-band message. They are still not a real transport, because the queue is bounded and the value is only an int. But they matter when you need to know that an event happened N times, not just at least once.

## The sigaction fields that make a handler safe

The earlier example calls sigaction with a zeroed struct, which is the smallest correct form. In practice you will want the other fields. sa_mask lists the signals that should be blocked while your handler runs. Without it, the same signal can interrupt its own handler, or a related signal can arrive in the middle of cleanup.

sa_flags changes behavior in two ways that matter. SA_RESTART asks the kernel to quietly restart certain interrupted syscalls such as read or accept, so a slow call does not fail with EINTR. That is handy for simple programs, but it can defeat a shutdown that needs the syscall to return. SA_SIGINFO switches the handler to a three-argument form that gets a siginfo_t describing the sending PID, the user ID, and the extra value for real-time signals. If you need to know who sent the signal or read a queued value, this flag is required, and the handler type changes from sa_handler to sa_sigaction.

## Receiving signals in an event loop via a file descriptor

A flag and a self-pipe are not the only ways to bring a signal into normal code. signalfd creates a file descriptor that turns readable when a signal is pending. You can give that descriptor to epoll, poll, or select alongside your sockets and timers. The program then blocks the signal in the normal mask so no handler runs, and the event loop reads a struct signalfd_siginfo that says what arrived.

This removes a whole class of problems. No async handler touches shared state. No EINTR needs handling. Receiving a signal becomes just another readable event with a clear order next to your other events. For a server already built around an epoll loop, signalfd is usually cleaner than a handler plus a pipe, because the single loop already owns the wait and the wakeups. The cost is one extra file descriptor, and the discipline to keep the signal blocked in every thread that shares the loop.

## Which thread actually receives a signal

In a single-threaded process this is simple, but a multi-threaded program has several choices and the kernel follows a rule. A signal sent to the whole process, for example by kill, goes to exactly one thread that does not currently block that signal. You cannot predict which thread, so if your handler touches shared state you must coordinate. Do not assume it runs on your main or event-loop thread.

A signal sent with a thread-targeting primitive goes to a specific place. pthread_kill sends a signal to a named thread inside the same process. That is how a worker can be told to break out of its own blocked call. The system call underneath is tgkill. It targets a thread group and a specific thread ID, so the kernel cannot deliver to the wrong process even if the thread ID is reused later. When you want every thread to notice, you broadcast to the process group or to each thread by name. When you want just one, you must name it.

## Why init must clean up zombies inside a container

The section on orphaned children said that Linux gives them a new parent at PID 1 when their own parent exits. This detail matters a lot inside containers. A container often runs the service as PID 1, with no separate init. If that service forks workers and then exits or crashes without reaping them, the workers become zombies parented to PID 1. In a minimal container, PID 1 may never call wait at all.

A zombie still takes a slot in the kernel process table. An init that never reaps will pile them up until the table is full and no new process can start. The fix is to run a proper init such as tini or dumb-init as PID 1. Or write your own supervisor that installs a SIGCHLD handler and calls waitpid in a loop until it collects every child. This is also why a good container supervisor does more than forward signals. It must own the process group, reap all children, and only exit once the group is truly empty.

## Interview definitions

### What is a signal?

> A signal is a small notification that arrives at any time. The kernel or another process sends it to a process, for example to tell it to stop, to say that a child changed state, or to report an invalid memory access. It carries almost no data, it can be merged into one, and it can arrive between any two instructions.

### What is the difference between `SIGTERM` and `SIGKILL`?

> `SIGTERM` asks a process to stop and can be handled, so the process can drain and clean up. `SIGKILL` cannot be caught or ignored and ends the process at once, so no application cleanup runs.

### What is graceful shutdown?

> Graceful shutdown is a procedure where a process stops taking new work, finishes or cancels the work already in flight within a deadline, saves important state, frees resources, and exits. The supervisor only needs to use `SIGKILL` if the deadline passes.

### What is a zombie process? An orphan?

> A zombie is a process that has exited but whose parent has not yet collected its status with `wait`. An orphan is a process that is still running but whose parent has exited, so it has been given a new parent.

## Interview follow-up questions

### Why should a signal handler do very little?

> The handler can run between any two instructions, while the code it interrupted might hold a lock or be partway through updating a structure. Many library functions are not safe in that moment, so the handler should just set a flag and let the main loop do the work.

### How should a backend handle `SIGTERM`?

> It should mark itself not ready, stop accepting work, give the in-flight handlers a timeout shorter than `TimeoutStopSec`, stop workers from taking new jobs, flush state, and exit before the deadline.

### Why can a restart policy create an outage?

> If the service exits because of a permanent configuration error, restarting it at once just repeats the failure, wastes resources, and hides the original error. A delay and a rate limit, plus different exit codes for config versus brief errors, prevent the loop.

## Common misconceptions

### “A service is healthy if its PID exists.”

A PID can exist while the process is stuck, blocked on a resource, or returning errors to every request. Check readiness and metrics, not just whether the process is alive.

### “Signals are reliable messages.”

They are not. They carry almost no information, they can be merged into one, and they send no acknowledgment. Use a pipe or a queue when you need payloads and order.

### “`SIGKILL` is the safe way to stop a service.”

It skips cleanup. Buffered data can be lost, and external leases can be left in an unsure state. Prefer `SIGTERM` with a deadline.

## Summary

A process gives you a box to run code. Signals give you a way to tell it something. `SIGTERM` asks for a graceful drain. `SIGKILL` forces an instant end. A handler should do almost nothing except tell the main loop what happened. Long-running services need a supervisor that gives config, a restart policy, resource limits, logging, and a deadline for shutdown. A healthy service is not just a live PID. It is a process that can make progress, report whether it is ready, and stop without losing correctness.

## If you want to build this later

Extend the small shell from the previous article. Make it handle `SIGTERM` with the flag pattern and make a supervisor that respects a timeout. The supervisor should send `SIGTERM`, wait for `TimeoutStopSec`, and then send `SIGKILL` if the process has not exited. Have it kill the whole process group, not just the parent. Test it with a handler that sleeps through `SIGTERM` and with a pipeline where one stage ignores the signal. Check that the group is stopped and that zombies are still reaped. Then run the same logic as a `systemd` unit and watch how it throttles a crash loop.
