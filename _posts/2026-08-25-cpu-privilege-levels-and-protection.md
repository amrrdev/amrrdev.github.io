---
mermaid: true
title: "CPU Privilege Levels and Protection"
date: 2026-08-25
categories: ["System Engineering"]
tags: [privilege, protection-rings, memory-protection, syscalls, secure-boot]
series: "System Engineering"
stage: "Stage 3 - Hardware and Computer Architecture"
stage_order: 3
series_order: 6
---

The previous chapter described how control moves to the kernel through interrupts and traps. This chapter explains why that move is safe. It is the sixth and final chapter of Stage 3.

A modern CPU runs in at least two modes. The ordinary mode is for user programs, the privileged mode is for the kernel. In user mode you can do arithmetic and touch your own memory, but you cannot change page tables, disable interrupts, or program a device. Only the kernel can. The hardware enforces this, and it only allows transitions through gates it controls.

A useful way to think about it is that user mode can ask and kernel mode can do. Protection rings on x86-64 and exception levels on ARM64 are just names for these modes, and bits in each page table entry decide whether a page can be read, written, or executed by user code. Every system call, interrupt, and fault goes through a gate where the kernel validates the request. Hardware features like Secure Boot, the IOMMU, and extra memory protections build on this same boundary.

For a backend, this boundary is the reason `mmap` with the wrong protection fails with `SIGSEGV`, why a container with `seccomp` can block `mprotect`, and why your service cannot bypass the kernel to talk directly to a disk.

## Why privilege is needed

If every program could run any instruction, there would be no isolation. A program could read another process's memory by mapping any physical page, it could overwrite scheduling tables and starve others, or it could tell a network card to send any packet.

The operating system would be just a library that a buggy program could ignore. Hardware privilege turns it into an enforcer. Even when user code is buggy or malicious, it cannot bypass the checks, because the CPU itself refuses the operation.

```mermaid
flowchart TB
    User[User thread can only touch own pages]
    User -->|syscall| Gate[Gate checks number and pointers]
    Gate --> Kernel[Kernel can touch page tables and devices]
    Kernel -->|return| User
    User -.->|tries direct device access| Fault[Fault to kernel]
    Fault --> Kernel
```

The diagram shows the two paths. The intended path goes through a gate that validates. Any attempt to go around the gate faults, and the fault itself is delivered to the kernel so it can decide what to do.

## User mode and supervisor mode

User mode is where your program runs most of the time. It can use ordinary instructions and touch pages that are marked as accessible to user code. It cannot use instructions like `cli` to disable interrupts on x86, or write to the register that holds the page table base.

Supervisor or kernel mode can do those things. It can access all memory, including pages marked as kernel only, and it can program devices.

A thread spends almost all of its time in user mode. It only enters kernel mode when a defined event happens, which is a system call, an interrupt from a device, or an exception from its own instruction. When it returns, it uses a special instruction, `sysret` on x86-64 or `eret` on ARM64, that restores the previous mode.

It helps not to confuse kernel mode with kernel space as a software idea. In the earlier overview article, user space and kernel space were about which software owns the memory and which interfaces are used. Privilege levels are the hardware that makes that ownership stick. User space is restricted because the CPU enforces it, not just because files are organized a certain way.

## Protection rings and exception levels

Different architectures give these modes different names, but the idea is the same.

On x86-64 there are four rings in the specification, but modern operating systems use only two. Ring 3 is user, ring 0 is kernel. The middle rings are mostly unused. Virtualization adds another layer below the kernel, often called ring -1, where the hypervisor runs.

On ARM64 there are exception levels, EL0 for user, EL1 for the kernel, EL2 for the hypervisor, and EL3 for the secure monitor that starts the machine. A system call from user code is `svc`, which moves from EL0 to EL1, while other transitions use `hvc` or `eret`.

The exact number of levels matters less than the rule they enforce. More privileged code can look at less privileged memory if it chooses, but less privileged code cannot reach more privileged state except through a gate.

```mermaid
flowchart TB
    EL3[EL3 Secure Monitor] --> EL2[EL2 Hypervisor]
    EL2 --> EL1[EL1 Kernel]
    EL1 --> EL0[EL0 User]
    EL0 -.->|svc / syscall| EL1
```

## Memory protection on every page

Permission is checked on every memory access, and the check is stored in the page table entry. On x86-64, a page can be marked readable, writable, executable with the `U` bit for user access and the `NX` bit for no-execute. On ARM64, fields like `AP`, `XN`, and `PXN` do the same job with more detail.

For example, a page that holds your program's code might be marked as user-readable and executable, but not writable. A page that holds the heap might be readable and writable, but not executable. A page that holds kernel data is marked as not accessible to user code at all.

When user code tries to write a read-only page, or execute a page marked `NX`, the access faults. The CPU delivers the fault to the kernel, which sends `SIGSEGV` to the process. When kernel code tries to access user memory by mistake, mitigations like SMAP on x86 or PAN on ARM can also fault. This is intentional. It stops an attacker who controls a user pointer from tricking the kernel into using it.

Guard pages use the same mechanism. The kernel leaves an unmapped page at the end of a stack. If recursion goes too far and touches it, the access faults immediately instead of silently corrupting the next mapping.

For a backend, you meet this when you call `mmap` with `PROT_READ` and then try to write, or when you `mprotect` a page to make it executable. The call can fail, but an access can also fault later. Both are the hardware saying the permission was not allowed.

## Controlled transitions

User code cannot jump to an arbitrary address in the kernel. The CPU only allows entry through addresses stored in tables that the kernel set up. On x86-64 this is the interrupt descriptor table, on ARM64 it is the vector table. Each entry says what privilege is needed to use it.

A system call is one of those entries. User code puts a number in `rax` on x86-64 or `x8` on ARM64, puts arguments in the other registers, and runs `syscall` or `svc`. The CPU switches to the kernel stack, saves registers, and jumps to the single kernel entry point. From there the kernel can check the number, validate every user pointer and length, check the process's capabilities, and then decide what to do. If the number is wrong or a pointer is invalid, it returns `EPERM` or `EFAULT` instead of crashing.

```mermaid
sequenceDiagram
    participant App as User app
    participant CPU
    participant Entry as Kernel entry
    participant Check as Validation

    App->>CPU: syscall
    CPU->>Entry: save state, switch to privileged stack
    Entry->>Check: check number, pointers, credentials
    Check->>Entry: invalid? return error
    Entry->>CPU: return result
    CPU-->>App: resume in user mode
```

The kernel copies user memory with helpers like `copy_from_user` for a reason. Between the time it checks a pointer and the time it uses it, the user thread could change the memory. The helper handles that safely, which is the same point made in the system call article about trusting user pointers.

## Secure Boot and other hardware protections

Privilege protects the machine while it is running. Secure Boot protects which code is allowed to get privilege at all. Firmware checks the bootloader's signature, the bootloader checks the kernel's signature, and the kernel checks module signatures. The chain starts from a key burned into hardware or stored in a TPM. If a step fails, the machine stops or falls back instead of running a tampered kernel with full privilege.

Two other protections matter for a backend. An IOMMU gives devices their own page tables. Without it, a device that does DMA could write to any physical page. With it, DMA is limited to pages the kernel mapped for that device, so a network card cannot overwrite kernel memory even if its firmware is buggy. A TPM or secure enclave stores keys and can attest what software booted, which is how a cloud VM can prove to a peer that it is really the payment service and not an impostor.

In production, your service often relies on all three together. Secure Boot makes sure the right kernel gained ring 0, the IOMMU bounds DMA, and a certificate name proves which service you talked to. A hostname string alone does not prove that.

## Seeing protection without a kernel module

You do not need to write kernel code to observe this. The kernel already exposes the protections.

```bash
cat /proc/self/maps | head
```

The first columns show the address range and permissions. `r--p` or `rw-p` are ordinary user pages, and later lines will show where the heap and stack sit.

You can provoke a fault safely from a scripting language.

```bash
python3 -c "import mmap; m=mmap.mmap(-1, 4096, prot=mmap.PROT_READ); m[0]=1"
```

The write to a read-only mapping raises `SIGBUS` or `SIGSEGV` instead of corrupting another mapping.

System calls still cross the gate, and you can watch them.

```bash
strace -e trace=mmap,mprotect ./program
```

Protection details appear in kernel logs and in process status.

```bash
dmesg | grep -i "NX\|SMEP\|SMAP"
cat /proc/self/status | grep -E "CapEff|Seccomp"
```

If you run the same binary under a tighter `seccomp` filter, the same `mprotect` can return `EPERM` even though the page exists. The error comes from the gate, not from the arithmetic in your program.

A deeper experiment is to map a page as readable, fill it, then change it to readable and executable and call it through a function pointer. First try mapping it as writable and executable at the same time and see that some configurations reject it, because write and execute together is treated as a risk. Then map it writable, write, and `mprotect` to executable after, which separate steps are usually allowed.

## A realistic production example

A team added a native Python extension that used `mmap` and `mprotect` with `PROT_WRITE|PROT_EXEC` to create a small JIT. It worked on their laptops, but in production the service crashed on startup with `SIGSEGV` and `EPERM` in `strace`. The container runtime there enabled a `seccomp` profile that blocks `mprotect` with execute permission, and the kernel enforced `W^X`, which means a page should not be both writable and executable at once.

The first reaction was to disable the filter. That would have worked, but it would have removed a mitigation whose job is to stop an attacker from writing code and then running it. The better fix was to keep the mitigation and change the allocation. The code allocated a page writable, wrote the generated instructions, and then changed the mapping to readable and executable before calling it. It never asked for write and execute together, and it used an approved JIT path that the platform allowed. Latency stayed the same, but the service no longer needed to weaken the boundary.

The lesson was not that protection is slow. The extra `mprotect` is cheap. The lesson was that when `mmap` succeeds but `mprotect` or an access fails, the layer that rejected you is the privilege boundary, and the correct fix is to follow its rules instead of disabling them.

## How engineers actually think about privilege

When a call fails with `EPERM`, `EFAULT`, or `SIGSEGV`, they check future possibilities before blaming logic. They look at `errno`, at the mapping in `/proc/<pid>/maps`, and at `CapEff` and `Seccomp` in `/proc/<pid>/status`. They run `strace` to see whether a `syscall` was rejected at the gate or whether the error happened after. They check `dmesg` for IOMMU messages when DMA is involved, and for a production service they check whether Secure Boot or TPM attestation is part of how the peer proves its identity.

The habit is to ask which level said no. A page permission fault, a `seccomp` filter, and a normal permission check all look similar at first, but they are enforced by different layers and have different fixes.

## SMEP, SMAP, PXN, and PAN: keeping the kernel away from user memory

Privilege does not only protect user code from the kernel. Recent CPUs add protections that stop the kernel from carelessly touching user memory. SMEP, supervisor-mode execution prevention, marks user pages as non-executable even when the CPU is in kernel mode, so a kernel bug or exploit cannot jump into shellcode placed in user memory. SMAP, supervisor-mode access prevention, similarly blocks the kernel from reading or writing user pages unless it explicitly opens a window with a special instruction. On ARM these are PXN, privileged execute never, and PAN, privileged access never.

These matter because many kernel exploits work by getting the kernel to dereference an attacker-controlled user pointer as code or data. With these bits on, that path faults by hardware, not by convention, which is why `dmesg` shows `SMEP` and `SMAP` enabled as a line item in a security audit. They are the per-access complement to the page-table permission bits: the same page that is writable by the user is now also off-limits to the kernel unless the kernel asks for it on purpose.

## Linux capabilities: fine-grained privilege without root

On Linux, `root` is not one switch. Privilege is split into about forty capabilities, each granting one class of operation. `CAP_NET_BIND_SERVICE` lets a process bind a port below 1024, `CAP_SYS_ADMIN` covers a broad set of administrative actions, `CAP_SYS_PTRACE` allows debugging other processes, and so on. A program can drop every capability it does not need, keeping only the few it must have.

This is why the article's distinction between `root` and the privilege boundary matters. A container running as `root` may still have an empty capability set, so it cannot, for example, load a kernel module or change the system clock, while a non-root process granted `CAP_NET_BIND_SERVICE` can bind port 80 without being root at all. `CapEff` in `/proc/<pid>/status` shows the effective set. The engineering practice is least privilege: grant exactly the capabilities a service needs and drop the rest at startup, which shrinks what a compromise could do.

## seccomp-bpf: shrinking the syscall surface

A process under Linux can install a seccomp filter, a tiny program written in BPF, that inspects each syscall number and arguments and decides allow, deny, or trap. The common case is the default-deny profile used by containers and runtimes: only a small allowlist of syscalls is permitted, and anything else returns `EPERM`. This is exactly the mechanism that blocked the `mprotect` with execute permission in the production example: the filter did not know about the service's JIT, so it rejected the call at the gate.

The value is attack-surface reduction. Even if an attacker gains code execution inside a process, they can only invoke the syscalls the filter permits, which often removes the ones needed for further escape. `seccomp` is the deepest layer shown here, below capabilities and below page permissions, because it is checked at the syscall gate before the kernel validates arguments. Combined with dropping capabilities and running unprivileged, it is the core of container sandboxing.

## The kernel lives in your address space: KPTI and PCID

A convenient detail of Linux is that the kernel is mapped into the top of every process's virtual address space. A `syscall` therefore changes privilege level but usually does not switch page tables, because the kernel's code and data are already present, just marked inaccessible to user mode. This keeps system calls cheap: no full page-table reload on every call.

Two refinements modify this. The first is KPTI, kernel page-table isolation, added to defend against Meltdown-class attacks that let user code read kernel memory through speculative execution. KPTI keeps a separate kernel page table and switches to it on entry, so the kernel mapping is not present in user mode at all, at the cost of an extra TLB flush on every crossing. The second is PCID on x86 or ASID on ARM, a tag that lets the TLB keep entries for both address spaces so switching does not invalidate everything. Both are reminders that the privilege gate and the page tables are linked: the same transition that the syscall article described as a mode switch is also, depending on the mitigation, a possible page-table switch.

## Definitions

### Privilege levels

> Hardware modes that separate user code, which is restricted, from kernel code, which is privileged. On x86 user code runs in ring 3 and the kernel in ring 0, on ARM in EL0 and EL1, and transitions are only allowed through gates like `syscall`.

### User mode versus kernel mode

> User mode runs ordinary code with access only to its own virtual pages. Kernel mode can manage page tables, program devices, and touch other processes' state. A user thread must trap to do privileged work. If it tries directly, the access faults.

### Memory protection

> Each page has permission bits that are checked on every access, such as readable, writable, executable, and user accessible. If user code tries to break them, the access faults and the kernel delivers `SIGSEGV`. This is how the system enforces `W^X` and keeps one process from reading another's memory.

### Controlled transitions

> The only legal way to enter privileged mode. User code runs `syscall` or `svc`, the CPU looks up the handler in the IDT or vector table, switches to the kernel stack, and the kernel validates the number, pointers, and credentials before dispatching.

### Secure Boot

> A hardware-rooted chain that checks each boot stage's signature before that stage is given privilege. Firmware checks the bootloader, the bootloader checks the kernel, and the chain starts from a key in hardware.

## Beyond the definitions

### Why user code cannot change page tables

> The page table decides which physical page a virtual address maps to. If user code could write the table, it could map any physical page, including kernel memory, and break isolation. Only privileged mode may write the table base register.

### What NX and W^X enforce

> `NX` marks a page as not executable, so trying to run code there faults. `W^X` is the rule that a page should not be both writable and executable at once, which stops an attacker from writing shellcode and then running it.

### How the kernel reads user pointers safely

> It checks that the range is inside user space, verifies the user-accessible bit, and then copies with a helper like `copy_from_user` that handles the case where the user thread changes the memory during the check.

### What the IOMMU does

> It gives I/O devices their own page tables, so DMA is limited to the pages the kernel mapped for that device. Without it, a device could overwrite any RAM, including kernel memory.

## Common misconceptions

**"Kernel space is just a folder."** It is a privilege mode enforced by hardware. Directories like `/boot` have nothing to do with it. One is about files, the other is about which instructions the CPU will allow.

**"More privilege is always faster."** Entering the kernel is slower because the CPU must switch mode, save state, and validate. Privilege is for protection, not speed. If you cross it often, batch system calls or use `io_uring` to reduce trips.

**"If I am root, privilege does not matter."** `root` is a software identity. Even as `root`, user code still runs in user mode until it traps. Container `root` can still be blocked by `seccomp`, capabilities, and page protections.

**"Secure Boot is just for laptops."** Cloud VMs and containers use measured boot to seal keys and attest which kernel booted. That attestation is how you know the kernel that gained privilege is the one you expected.

## Summary

Privilege levels are the hardware reason the operating system can actually enforce isolation. User code asks through traps, the CPU only allows entry through gates, each page carries its own permission bits, and features like IOMMU and Secure Boot extend the same idea beyond the basic rings. For a backend, the difference between `EPERM`, `EFAULT`, and `SIGSEGV`, or between `mmap` succeeding and `mprotect` failing, is the sound of this boundary doing its job.

With this chapter, Stage 3 ends. We moved from how a single instruction is executed, through the counters that measure it, the caches and memory ordering that sit underneath it, and the device interrupts and privilege gates that let it talk to the rest of the machine safely. The next stage looks at how that source code becomes an executable in the first place.
