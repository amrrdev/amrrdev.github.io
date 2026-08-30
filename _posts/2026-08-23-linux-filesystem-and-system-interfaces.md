---
mermaid: true
title: "Linux Filesystem and System Interfaces"
date: 2026-08-23
categories: ["System Engineering"]
tags: [linux, proc, sys, dev, dmesg]
series: "System Engineering"
stage: "Stage 2 - Linux and Operating System Internals"
stage_order: 2
series_order: 5
---


> Stage 2 :  Linux and Operating System Internals  
> Subject area 2.1 :  The Operating System Model  
> Article 5

## The short version

Linux shows a lot of its live state through interfaces that look like files. Programs and engineers use these interfaces to look at processes, memory, devices, hardware, kernel settings, logs, clocks, and environment state.

The most important interfaces are:

- `/proc` shows process and kernel runtime information
- `/sys` shows devices, drivers, hardware links, and kernel objects
- `/dev` holds device nodes and special kernel-managed resources
- Kernel logs hold messages from the kernel and drivers
- Service logs hold events reported by user-space services

Not all of these are ordinary files on a disk. Many are virtual entries, also called pseudo-filesystem entries. The kernel or a user-space service makes them when you read them. They give you one common way to inspect and control a live system. A pseudo-filesystem is like a file view of live kernel state. Reading it formats the current state as text. Writing to it changes the state.

The central idea is:

> When a Linux system acts in a way you did not expect, look at the state the kernel and services show. Do not only guess from the application itself.

For a backend service, this means a slow request can often be fixed faster by looking at these views. The cause might be open file descriptors, memory mappings, or a blocked device. These views show the cause faster than application logs alone.

## Where this article fits

The previous article explained how Linux makes and supervises processes. This article explains where to look at those processes and the rest of the operating system. You look through virtual filesystems.

**Prerequisites:** read Linux Processes, Signals, and Services first. You need to know what a PID and a process state are before you look at them.

**Next:** Linux Clocks, Hostnames, and Environment. That article shows where time, identity, and configuration are exposed, and how to use them for timeouts and service discovery.

Later articles use these interfaces to talk about memory, devices, filesystems, scheduling, resource limits, containers, security, and debugging. These interfaces are some of the first tools a systems engineer reaches for when checking a machine.

## Not every file is stored on disk

A regular file is data that stays on disk and is stored through a filesystem. A pseudo-file is an interface that shows information through file-like operations. It does not have to be data stored on disk.

The file-like design is useful because programs already know how to open, read, write, and close files. The kernel can show live state through that same simple mechanism.


Reading a pseudo-file can make the kernel turn the current state into text. Writing to one can change a kernel setting or send a request to a device driver. What it means depends on the path and the rules of that interface.

This means the usual rules about files do not always apply. A pseudo-file can change between reads. It may have no real disk size. It may reject normal file operations. It may need special permissions.

## `/proc`: process and kernel runtime information

`/proc` is a pseudo-filesystem. It shows process information and some kernel state. Linux usually mounts it at `/proc`.

```text
/proc
├── 1/
├── 2450/
├── self/
├── thread-self/
├── cpuinfo
├── meminfo
├── mounts
├── net/
├── sys/
└── uptime
```

A directory whose name is a number usually stands for a process ID. Inside that directory you find information about that process. This includes its command line, memory mappings, open files, status, and resource counts.

Some entries describe the whole system, not just one process. `cpuinfo` shows processor details. `meminfo` shows memory counts. `uptime` shows how long the system has been running. `mounts` and related files show the filesystems that are mounted.

## Inspecting a process through `/proc`

Suppose a process has PID 2450. These paths are useful:

```text
/proc/2450/cmdline       Command-line arguments
/proc/2450/environ       Environment variables
/proc/2450/status        Human-readable process status
/proc/2450/stat          Process statistics in a compact format
/proc/2450/fd/           Symbolic links for open file descriptors
/proc/2450/maps          Current memory mappings
/proc/2450/smaps         Detailed mapping statistics
/proc/2450/limits        Resource limits
/proc/2450/cgroup        Control-group membership
/proc/2450/task/         Threads belonging to the process
```

Who can read these paths depends on the user, system settings, security rules, and kernel version. One process may be blocked from looking at another process's environment or memory details.

### `/proc/<pid>/status`

The `status` file lists readable fields such as:

- Process name
- State
- Process ID and parent process ID
- Number of threads
- Virtual memory size
- Resident memory size
- User and group identifiers
- Capability information
- Signal masks

Resident memory means the pages that sit in physical memory for the process right now. Virtual memory size also counts address-space mappings that may not be in physical memory at this moment. Mixing up these two numbers can lead to wrong ideas about memory pressure.

### `/proc/<pid>/fd`

The `fd` directory holds one symbolic link for each open file descriptor the process can see.

```text
/proc/2450/fd/0 → /dev/null
/proc/2450/fd/1 → /var/log/example.log
/proc/2450/fd/2 → /var/log/example.log
/proc/2450/fd/7 → socket:[123456]
```

Descriptors 0, 1, and 2 are the usual standard input, standard output, and standard error. Other descriptors can point to files, pipes, sockets, devices, event objects, or anonymous kernel objects.

This interface helps when you debug 'too many open files', files that stay open by mistake, or a service that keeps a socket or pipe open longer than it should.

### `/proc/<pid>/maps`

The `maps` file shows the process's virtual-memory regions. Each mapping can be one of these:

- The executable
- A shared library
- The heap
- A thread stack
- An anonymous allocation
- A memory-mapped file
- A special kernel-provided region

Each entry usually shows an address range, permissions, file offset, device and inode information, and an optional pathname.

```text
address range        permissions  offset  object
55a00000-55a12000    r-xp         ...     /usr/bin/example
7f000000-7f020000    rw-p         ...     [heap]
7ffc0000-7ffc21000   rw-p         ...     [stack]
```

The permissions usually show as `r` for read, `w` for write, `x` for execute, and `p` or `s` for a private or shared mapping. Memory maps help you see the address-space layout, the shared libraries, which parts can run as code, and how memory is growing.

## `/proc/self` and `/proc/thread-self`

`/proc/self` is a handy link to the `/proc` directory of whatever process reads it. A program can read `/proc/self/status` without knowing its own PID.

`/proc/thread-self` points to the current thread's view, for cases where thread-specific information matters.

These paths are useful in tools and diagnostics. They avoid a race where a program finds its PID and then, by bad luck, looks at a different process after the old PID was reused.

## `/proc` is a live view

The state shown through `/proc` can change while a program reads it. A process can exit, make a thread, open a descriptor, close a descriptor, or change its memory mappings between two reads.

This creates an important rule:

> A `/proc` reading is a snapshot of state at one point in time. It is not always a fully consistent picture of the whole system.

A monitoring tool that reads several files may see values from slightly different moments. That is usually fine for diagnosis. It matters when a program makes a safety-critical decision from that data.

## `/sys`: the kernel's device and object model

`/sys` is usually mounted as `sysfs`. It shows information about devices, drivers, buses, kernel objects, and how they relate. It differs from `/proc`, though the two may overlap on some facts.

`/proc` focuses on processes and general kernel state. `/sys` shows a structured view of devices and the kernel object tree.

```text
/sys
├── block/
├── bus/
├── class/
├── devices/
├── firmware/
├── fs/
├── kernel/
├── module/
└── power/
```

The exact entries depend on the hardware, the drivers, the kernel settings, and the system state.

### `/sys/devices`

This tree shows devices as they exist in the system's device tree. It can show how a device relates to its bus, its parent controller, and its driver.

These links matter when you investigate a device problem. A network interface can depend on a PCI device, a driver, firmware, power settings, and a physical link. `/sys` helps you connect those pieces.

### `/sys/class`

The class tree groups devices by what they do, not only by where they sit. Examples include:

```text
/sys/class/net/      Network interfaces
/sys/class/block/    Block devices
/sys/class/tty/      Terminal devices
/sys/class/power_supply/
```

This is a handy way to find devices by function. A network tool can look at `/sys/class/net` without knowing where each interface sits in the hardware tree.

### `/sys/block`

Block devices store data in fixed-size blocks you can address. `/sys/block` can show device names, queue details, sizes, partitions, and links to the devices underneath.

Storage speed and behavior can depend on queue depth, scheduler settings, whether the disk spins, device state, and layered devices such as RAID or device-mapper targets.

### Reading and writing `/sys`

Some sysfs files are read-only views. Others are writable settings. Writing a value does not save data to a disk file. It may change kernel behavior right away.

A setting may last only until reboot, need a specific unit, or carry safety limits. Writing to a sysfs entry without knowing its rules can change device behavior or system speed.

This is why you should treat `/sys` as a typed kernel interface shown through files. Do not treat it as an ordinary directory you can edit without care.

## `/dev`: device nodes and special resources

`/dev` holds device nodes and other special entries. Programs use them to reach devices or kernel-managed resources.

Common examples include:

```text
/dev/null       Discards writes and returns end-of-file on reads
/dev/zero       Produces zero bytes
/dev/random     Kernel-provided random-data interface
/dev/tty        Controlling terminal
/dev/sda        A block-device node, when present
/dev/console    System console interface
```

These entries are not regular files that hold the device's full data. They are names tied to device drivers or kernel subsystems. Opening and using one runs the behavior that driver defines.

### Character and block devices

A character device usually represents a stream of bytes or operations. It is not addressed as fixed storage blocks. Terminals and many sensors are examples.

A block device provides storage operations built around blocks. Disks and virtual block devices are examples.

This split is useful but it does not tell you everything. A device's driver decides details such as blocking, buffering, supported operations, errors, and synchronization.

### Device permissions

Device nodes have owners and permission rules. Access to `/dev` can expose hardware, private random data, input devices, storage, or kernel features. Letting a process reach many devices can weaken isolation, even if that process cannot touch ordinary files.

Containers and service managers often use device policies to decide which devices a process may see.

## Kernel messages

The kernel and device drivers need a way to report events. These include hardware detection, driver startup, memory pressure, device errors, and security decisions. Linux keeps kernel logging facilities that user-space tools can read.

`dmesg` commonly displays messages from the kernel ring buffer:

```bash
dmesg --level=err,warn
```

The ring buffer has a fixed size. Newer messages can overwrite older ones. Access may also be limited because kernel messages can hold private information.

Kernel messages help during boot, device discovery, driver failures, filesystem errors, and hardware problems. They are not a full application log, and they should not replace service-level logging.

## Service logs and the system journal

User-space services report events through standard output, standard error, log libraries, files, or a logging service. On systems that use `systemd`, the journal can gather a service's output and metadata. This includes the unit name, PID, user, boot ID, and timestamp.

Useful commands include:

```bash
journalctl -u example-worker
journalctl -u example-worker --since "10 minutes ago"
journalctl -b
journalctl -k
```

The first command filters logs for one service unit. The second limits how far back it looks. `-b` picks the current boot, and `-k` focuses on kernel messages kept in the journal.

The journal is a user-space logging system. It has its own storage, filtering, rotation, and access rules. It is not the same as the kernel ring buffer, even when it holds copies of kernel messages.

Good service logs explain events with useful context:

- What operation was attempted
- Which resource or request was involved
- What failed
- What decision the service made
- Whether a retry or recovery occurred
- A request, job, or correlation identifier

Logging every small internal detail is not always useful. Logs use storage, may show private data, and can get noisy during an incident.

## The same interface can expose and change state

Linux system interfaces often support both looking and changing.

Reading `/proc/<pid>/status` shows process state. Writing a value to a control file under `/proc/sys` can change a kernel setting. Reading `/sys` can show a device property. Writing to a writable sysfs attribute can change a device or driver setting. Sending a signal changes process state through a different interface.

Tools and docs should make the difference between looking and changing clear. A diagnostic command that changes a live setting by mistake is dangerous.

## A small code example: inspect a process from user space

The Go function below reads a process's status file. It uses a normal file API, but the path points to a live view the kernel generates, not to data saved on disk.

```go
func processStatus(pid int) ([]byte, error) {
 path := fmt.Sprintf("/proc/%d/status", pid)
 data, err := os.ReadFile(path)
 if err != nil {
  return nil, fmt.Errorf("read %s: %w", path, err)
 }
 return data, nil
}
```

The function looks like ordinary file reading, but the path is not a regular file. `fmt.Sprintf` builds a name the kernel understands, and `os.ReadFile` makes the kernel turn the current process state into text. If it works, you get lines like `Name: example-worker` and `VmRSS: 12345 kB`. If the process exited after you listed it but before you read it, you get `no such file or directory`. If you lack permission, you get `permission denied`. The example is kept small on purpose. A fuller version would also read `/proc/<pid>/cmdline` to check that the PID was not reused, and it would handle the case where the text changes while you read it.

Even though the interface looks like a file, the program must still think about permissions, races, and changing state.

## Inspection tools are views built on these interfaces

Many familiar Linux tools read from these interfaces, or they use system calls that do the same job.

| Question | Useful tools or interfaces |
| --- | --- |
| Which processes are running? | `ps`, `pstree`, `/proc` |
| What is a process doing? | `/proc/<pid>/status`, `strace`, `top` |
| What files and sockets are open? | `lsof`, `/proc/<pid>/fd`, `ss` |
| What memory is mapped? | `/proc/<pid>/maps`, `pmap`, debugger |
| What devices exist? | `/sys`, `udevadm`, `/dev` |
| What did the kernel report? | `dmesg`, `journalctl -k` |
| Why did a service fail? | `systemctl`, `journalctl -u` |
| What time behavior is available? | `clock_gettime`, `timedatectl`, `/proc/uptime` |

A tool is a way to reach evidence, not an explanation by itself. `ps` may show that a process is sleeping, but you may need a trace or a stack inspection to learn what it is waiting for.

## Race conditions while inspecting state

Inspection can race with the system as it changes. A process may exit after a tool lists its PID but before the tool reads `/proc/<pid>/status`. A file descriptor may close between listing `/proc/<pid>/fd` and reading one of its links. A device may vanish during a hardware event.

Tools should treat these races as normal. A missing entry does not always mean the first observation was wrong. It may mean the system changed between steps.

This is one reason a single snapshot is not always enough to diagnose a problem. Repeated observations, timestamps, tracing, and links to service logs can give a more reliable explanation.

## A realistic production example

Imagine a service that the system reports as 'running', but requests are failing. The supervisor shows the process has a live PID. The first guess is that the application is healthy.

The engineer checks `/proc/<pid>/status` and sees the process has many threads but little CPU activity. The file-descriptor directory shows a large number of sockets. `ss` shows many connections waiting in a state tied to slow clients. Service logs show request deadlines being missed, while kernel and network statistics show no hardware failure.

The process is alive but it cannot make useful progress, because slow connections tie up its resources. The fix may use connection timeouts, bounded concurrency, backpressure, or a change in how responses are streamed. Restarting the process may bring the service back for a while, but the interfaces reveal the resource behavior that caused the problem.

## How experienced engineers use these interfaces

They start with a question instead of opening every file in `/proc`.

For a process problem, they might ask:

- Is it alive, blocked, or repeatedly restarting?
- What resources does it hold?
- Which files, sockets, and memory mappings are open?
- What identity and limits apply?
- Which threads are waiting?

For a device problem, they might ask:

- Does the kernel see the device?
- Which driver is attached?
- What does the device state say in `/sys`?
- Did the kernel log an error?
- Are permissions or device policies blocking access?

For a service problem, they might ask:

- Did the service start with the expected environment?
- Which unit owns the process?
- What did the journal record before the failure?
- Is the manager restarting it?
- Did the clock, hostname, or configuration change?

The goal is to turn a vague symptom into a system-level idea you can check.

## The virtual filesystem layer hides the differences between disk formats

A program that opens a file usually does not know or care whether the bytes sit on ext4, xfs, btrfs, or an overlay. This sameness exists because the kernel keeps a virtual filesystem (VFS) layer between applications and the real on-disk formats. The VFS defines one common set of operations: open, read, write, lookup, stat, and a few others. Each real filesystem registers its own version of those operations and turns them into its own on-disk layout.

The practical result is that the same system call works on every mounted filesystem. ext4, xfs, and btrfs each store metadata and data blocks in very different ways, but they all show the same inode-and-dentry model to the rest of the kernel. overlayfs is a good example: it stacks a lower read-only layer and an upper writable layer, and through the VFS it makes the pair look like one ordinary directory tree. A container image layered on top of a base image works the same way. When you debug a path problem, remember that the path you see is a VFS view. The layers underneath may spread across several filesystems with different traits.

## Watching files change with inotify and fanotify

Sometimes an engineer does not want to keep asking a directory whether it changed. Instead, the engineer wants the kernel to report the change. inotify watches a path for events such as create, modify, delete, move, and attribute changes. A watcher gets a queue of events and can act as soon as a file changes. fanotify is a coarser interface aimed at access and permission decisions. Antivirus scanners and some backup systems use it to watch or block file access.

These tools have limits you should know before you rely on them. The number of inotify watches a user can register is bounded by `/proc/sys/fs/inotify/max_user_watches`. The event queue can overflow under heavy load. After that, you get an overflow notice instead of every event. A less obvious limit is that inotify does not cross mount points. If you watch a directory and a different filesystem is later mounted on top of it, events inside that mount are not sent to your watch. fanotify, by contrast, can be scoped to a whole mount or filesystem, and it fits better when you must see activity across a large tree.

## A file name is not the same thing as an inode

It is easy to speak of 'a file' as if the name and the data were one object. In the filesystem model they are separate. A directory entry (often called a dentry) maps a name to an inode. The inode holds the metadata and the pointers to the data blocks. Several names in one or more directories can point at the same inode. Those are hard links. The inode keeps a link count so the system knows how many names point to it.

This split explains a behavior that surprises many engineers. When you delete a file with `rm`, you remove a name, which lowers the link count. If a running process still holds the file open through a file descriptor, the link count may reach zero, but the inode and its data blocks stay on disk until that last descriptor is closed. The space is not freed, and the file can keep growing. You can see this with `lsof` (look for a `DEL` or deleted marker) or by inspecting `/proc/PID/fd`. There, the link path ends in `(deleted)` while the descriptor is still valid. This is why a service whose log file was deleted can still fill the disk. The usual fix is to restart or signal the process so it closes and reopens the file.

## Bind mounts let a process see a different slice of the tree

A bind mount takes an existing directory or file and makes it appear at another spot in the same mount tree. After `mount --bind /srv/data /mnt/view`, the path `/mnt/view` shows the contents of `/srv/data`, and changes made through one path show up through the other. Bind mounts are a building block for the trimmed, overlaid views that containers present.

Inside a container, the kernel uses a mount namespace. The process then sees only the mounts placed in its namespace, not the host's full tree. Combined with bind mounts and overlayfs, this lets a container have a read-only `/usr` taken from an image layer, a fresh `tmpfs` at `/tmp`, and an overlaid root that mixes base and application layers. The container's `/proc/mounts` shows only what is visible in its namespace. That is why a path that exists on the host may be missing or different inside. When you debug a container, check whether the path you expect is actually mounted into its namespace, rather than assuming the host tree is visible.

## Other pseudo-filesystems a systems engineer reads, and the open-descriptor limit

Beyond `/proc` and `/sys`, Linux mounts several other pseudo-filesystems that expose kernel state. cgroupfs (usually at `/sys/fs/cgroup` for cgroup v2) shows the resource-control tree. You can read a process's memory limit and current usage, its CPU weight, and its I/O throttle settings. tracefs and debugfs (often under `/sys/kernel/debug`) expose tracing and driver debug state. An engineer chasing latency may read tracefs events, while a driver problem may show up in debugfs. These entries follow the same rule as other pseudo-filesystems. They are made when you read them, and they may be writable controls rather than persistent files.

A related limit that shows up in production is the maximum number of open file descriptors. Each process is bounded by `RLIMIT_NOFILE`. It has a soft limit (what the process sees by default, often 1024) and a hard limit (the ceiling it can raise itself to). You can check a process's current usage by counting entries in `/proc/PID/fd` and comparing it to its limit shown in `/proc/PID/limits`. When a service hits this limit, further `open`, `accept`, or `socket` calls fail with 'too many open files', even though disk space and memory look healthy. The usual causes are a descriptor leak, an unbounded connection pool, or a limit left at the small default. Raising it with `ulimit -n` or a service manager's `LimitNOFILE` only helps after the leak itself is fixed.

## Interview definitions

### What is `/proc`?

> `/proc` is a Linux pseudo-filesystem. It shows live process information and selected kernel state through file-like interfaces. Reading `/proc/<pid>/status` formats the current state of that process, and reading `/proc/<pid>/fd` lists its open descriptors. The data is a live view, not a file saved on disk.

### What is `/sys`?

> `/sys` is a pseudo-filesystem. It shows the kernel's devices, drivers, and hardware relationships. It shows the device tree under `/sys/devices` and a class view like `/sys/class/net`. Some entries are writable controls. Writing to them changes kernel behavior right away.

### What is `/dev`?

> `/dev` holds device nodes. They let programs talk to drivers through file-like operations. `/dev/null` discards what you write and returns end of file when you read, while `/dev/sda` is an interface to a block device driver. Operations there behave like device actions, not ordinary file actions.

### What is the difference between `/proc` and `/sys`?

> `/proc` is mostly about processes and general runtime state. `/sys` is about devices and the kernel object model. Both are virtual, but `/proc/<pid>` centers on processes and `/sys/class/net` centers on devices.

### What is a pseudo-filesystem?

> A pseudo-filesystem makes file-like entries from live system state, instead of storing them as persistent files. Programs can use `open`, `read`, and `write` to inspect kernel state. The contents can change between reads, and some operations may be rejected.

## Interview follow-up questions

### Why does Linux expose kernel state through files?

> File operations give a familiar interface. Tools and programs can use them to inspect or control state. The file-like form does not mean the data is stored on disk. It is often made on the fly by the kernel.

### Can `/proc` data be treated as a consistent snapshot?

> Not always. Processes and resources can change while the files are being read. So readings from several entries may come from slightly different moments.

### What is the difference between `/dev/null` and a regular file?

> `/dev/null` is a device interface made by the kernel. Writes are thrown away and reads return end-of-file. It does not hold persistent data on disk.

### Why might a process be alive but unhealthy?

> It may be blocked on a resource, stuck in a retry loop, unable to accept work, holding exhausted connections, or failing every request. A live PID only proves the process has not exited.

### Why might a `/proc` entry disappear during inspection?

> The process may have exited between the directory listing and the read. `/proc` shows live state, so programs and tools must handle changes and races.

## Common misconceptions

### “Everything under `/proc` is a normal file.”

The entries use file-like operations, but many are made on the fly and behave differently from persistent files.

### “Writing to `/sys` edits a configuration file.”

Writing to a sysfs attribute usually sends a control request to the kernel or driver. The change may be immediate, temporary, restricted, or it may affect hardware.

### “`/dev/sda` contains the entire disk as a normal file.”

It is a device node. It gives access to a block-device driver. Operations on it behave like device and kernel actions, not just ordinary file actions.

### “A process list is a reliable snapshot of the machine.”

Processes can start, exit, and change state while the list is being collected. It is an observation taken over time, not necessarily one single view.

## Summary

Linux shows live system state through file-like interfaces. `/proc` shows process and kernel runtime information. `/sys` shows devices and kernel-object relationships. `/dev` gives access to devices and special resources. Kernel logs and service journals show events from the kernel and user services.

These interfaces are powerful because they let ordinary tools inspect a complex machine. The tools do not need to know the kernel's internal data structures. They also have limits. State can change while it is being read. Permissions can hide information. And file-like behavior does not mean ordinary persistent-file rules.

The systems-engineering habit is to start with a question. Then inspect the interface that can give evidence. Understand its consistency and permission limits. Then connect the observation to a process, resource, device, or service idea.

Clocks, hostnames, and environment are the configuration side of these interfaces. The next article, *Linux Clocks, Hostnames, and Environment*, covers them.

## If you want to build this later

Build a small Linux system-inspection command. It should report one target process in a readable format.

Read `/proc/<pid>/status`, `/proc/<pid>/limits`, `/proc/<pid>/maps`, `/proc/<pid>/fd`, and `/proc/<pid>/cmdline`. Add options to show memory, open descriptors, threads, and resource limits. Handle processes that exit during inspection. In the output, explain that the values are observations, not one atomic snapshot.

Then add a device mode that lists network interfaces through `/sys/class/net` and reports their state. The project should teach you to treat `/proc`, `/sys`, and `/dev` as system interfaces. They have contracts, permissions, races, and changing state.
