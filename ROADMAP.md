# Systems Engineering Roadmap

This roadmap is a learning map for becoming a strong systems engineer who can understand operating systems, hardware, networks, storage, distributed systems, backend services, infrastructure, and production reliability.

It is intentionally organized by article-sized learning units.

```text
Stage
  └── Subject area
        └── Focused article
              └── Concepts explained inside that article
```

The lines under an article are not automatically separate blogs. They describe the content that belongs inside that article.

## How to use this roadmap

- A small, closely related group of concepts should be one article.
- A major concept with its own mental model, mechanism, diagrams, failure modes, and tradeoffs should be a separate deep article.
- System calls, TCP, virtual memory, concurrency, WAL, Raft, and similar concepts should not be reduced to short subsections of unrelated articles.
- Small details such as syscall arguments and return values belong inside the larger article that explains system calls.
- An overview article may introduce a subject area before the deeper articles begin.
- The final number of articles is allowed to change when writing reveals that a topic is too broad or too narrow.

## Recommended starting order

Stage 0 is useful but can be deferred. Start with the technical systems path:

```text
Stage 1 → Stage 2 → Stage 3 → Stage 4 → Stage 5 → Stage 6
        → Stage 7 → Stage 8 → Stage 9 → Stage 10
        → Stage 11 → Stage 12 → Stage 13 → Stage 14
        → Stage 15 → Stage 16
```

Return to Stage 0 when you want a change from low-level technical material or before focusing on large-company engineering practices and interviews.

# Stage 0 — Professional Engineering Foundations

This stage is deferred initially. It explains how engineering work is organized and maintained in a professional environment.

## Subject area 0.1 — How Professional Software Engineering Works

### Article: What Software Engineering Actually Involves

- Software engineering versus programming
- Solving real problems under constraints
- Requirements, design, implementation, testing, deployment, operation, and maintenance
- Ownership and long-term responsibility
- How experienced engineers make decisions with incomplete information

### Article: Requirements, Constraints, and Tradeoffs

- Functional and non-functional requirements
- Performance, availability, security, cost, and compatibility constraints
- Turning vague requests into measurable outcomes
- Making tradeoffs visible
- Choosing a simple solution when it is sufficient

### Article: Reading and Understanding an Unfamiliar Codebase

- Finding the entry point
- Following data and control flow
- Identifying boundaries and ownership
- Using tests, logs, and version history as documentation
- Building a useful mental model without reading every file

### Article: Code Ownership, Maintenance, and Technical Debt

- Ownership boundaries
- Maintenance work
- Deliberate versus hidden technical debt
- Deprecation
- Backward compatibility
- Keeping systems understandable as they grow

## Subject area 0.2 — Developer Tools and Build Systems

### Article: Linux Command-Line Workflow

- Files, processes, permissions, pipes, redirection, and environment variables
- `grep`, `sed`, `awk`, `find`, and `xargs`
- SSH and remote debugging basics
- Inspecting processes, files, sockets, and resource usage

### Article: Git and Collaborative Change

- Commits, branches, merges, rebases, and tags
- Reviewing history with `log`, `diff`, and `bisect`
- Safe collaboration and resolving conflicts
- How version history supports debugging and ownership

### Article: Compilation, Build Tools, and Reproducible Artifacts

- Make, CMake, Cargo, and Go tooling
- Dependencies and package managers
- Build configuration
- Artifacts and release inputs
- Reproducible builds
- Continuous integration basics

## Subject area 0.3 — Testing and Correctness

### Article: Testing Strategy for Systems

- Unit, integration, contract, and end-to-end tests
- What each level can and cannot prove
- Test doubles and test isolation
- Regression tests and flaky tests

### Article: Testing Performance, Concurrency, and Failure

- Benchmarks, load tests, stress tests, and soak tests
- Race testing and fuzzing
- Fault injection
- Deterministic reproduction
- Testing recovery instead of only success

# Stage 1 — Systems Programming Foundations

## Subject area 1.1 — What Systems Programming Means

### Article: What Systems Programming Means

- Systems programming versus application programming
- Systems software versus business software
- The operating system as a resource provider
- Why systems code must reason about ownership, limits, failure, and boundaries

### Article: CPU, Memory, Storage, and Network Resources

- What each resource provides
- Why each resource is limited
- Resource contention
- How a service can be CPU-bound, memory-bound, storage-bound, or network-bound
- Latency and throughput at the resource level

### Article: Resource Ownership and Limits

- Who owns CPU time, memory, files, sockets, and connections
- Resource quotas and exhaustion
- Cleanup and lifetime
- What happens when a process or service reaches a limit

### Article: Failure, Determinism, and Control

- Failure handling
- Partial failure
- Deterministic behavior
- Control versus convenience
- Why systems engineers design for recovery

### Article: Portability, Compatibility, and Abstraction Leaks

- Portability across Linux, macOS, and Windows
- CPU architecture differences
- APIs and ABIs
- Backward compatibility
- Why lower-level behavior can leak through a high-level abstraction

## Subject area 1.2 — Systems Programming Languages

### Article: C as a Systems Programming Language

- Pointers, memory, object lifetime, and undefined behavior
- C's relationship with the operating system
- Compilation and binary interfaces
- Why C remains important

### Article: Rust, Zig, Go, and C Tradeoffs

- Ownership and memory safety
- Manual memory and garbage collection
- Runtime behavior
- FFI and interoperability
- Build and deployment tradeoffs
- Choosing a language based on system requirements

# Stage 2 — Linux and Operating System Internals

## Subject area 2.1 — The Operating System Model

### Article: What the Operating System Provides

- Processes, memory, files, devices, networking, and security
- Kernel responsibilities
- User space and kernel space
- Privilege boundaries
- Why applications need the kernel

### Article: System Calls: How Programs Request Kernel Services

- What a system call is
- The user/kernel transition
- System-call arguments and validation
- Return values and error reporting
- Common calls such as `read`, `write`, `open`, `fork`, `exec`, and `mmap`
- System-call cost and tracing

### Article: Linux Processes, Signals, and Services

- Process identifiers and lifecycle
- Parent and child processes
- `fork`, `exec`, and `wait`
- Zombies and orphans
- Signals and signal safety
- Daemons, services, and graceful shutdown

### Article: Linux Filesystem and System Interfaces

- `/proc`, `/sys`, and `/dev`
- Kernel messages
- Service logs
- System clocks and timers
- Hostnames and environment configuration

## Subject area 2.2 — Scheduling and Resource Control

### Article: CPU Scheduling and Context Switching

- Preemptive and cooperative scheduling
- Scheduling queues and priorities
- Time slices
- Context switches
- Voluntary and involuntary switches
- Scheduler latency

### Article: Linux Resource Limits and the OOM Killer

- File-descriptor, process, thread, CPU, and memory limits
- `nice` and priorities
- cgroups and resource accounting
- Memory pressure
- The Linux out-of-memory killer
- Diagnosing resource exhaustion

# Stage 3 — Hardware and Computer Architecture

## Subject area 3.1 — CPU Execution

### Article: How a CPU Executes Instructions

- Instructions, registers, and instruction sets
- x86-64 and ARM64
- Pipelines
- Superscalar and out-of-order execution
- Branch prediction
- Speculative execution
- SIMD and vector instructions

### Article: CPU Performance and Hardware Counters

- CPU frequency and turbo behavior
- Thermal throttling
- Instructions per cycle
- Branch misses
- Cache misses
- Hardware performance counters
- Why benchmarks can mislead

## Subject area 3.2 — Memory Hardware and Ordering

### Article: CPU Caches and Memory Locality

- Cache levels and cache lines
- Spatial and temporal locality
- Cache coherence
- False sharing
- Prefetching
- Memory bandwidth versus memory latency

### Article: Memory Ordering and Atomic Hardware

- Compiler and CPU reordering
- Visibility between cores
- Acquire, release, and sequential consistency
- Fences
- Store buffers and load buffers
- Hardware atomic instructions

### Article: Interrupts, Traps, Exceptions, and Device I/O

- Hardware interrupts
- Traps and exceptions
- Interrupt handlers
- Deferred work
- DMA
- Memory-mapped I/O
- Device drivers
- Polling versus interrupts

## Subject area 3.3 — Privilege and Protection

### Article: CPU Privilege Levels and Protection

- User and supervisor mode
- Protection rings
- Memory protection
- Controlled transitions
- Secure boot
- Hardware security features

# Stage 4 — From Source Code to Execution

## Subject area 4.1 — Compilation

### Article: The Compilation Pipeline

- Preprocessing
- Parsing and semantic analysis
- Intermediate representations
- Optimization
- Code generation
- Assembly and object files
- Debug information
- Optimization levels and undefined behavior

### Article: Assembly, Calling Conventions, and Stack Frames

- Registers and instructions
- Function calls
- Arguments and return values
- Stack frames
- Prologues and epilogues
- ABI differences

## Subject area 4.2 — Linking and Loading

### Article: Object Files, Sections, and Symbols

- Code, data, read-only data, and BSS
- Symbol tables
- Local and global symbols
- Relocations
- Debug symbols and DWARF

### Article: Linking: Static Libraries and Shared Libraries

- Static linking
- Dynamic linking
- Symbol resolution
- Shared objects
- Symbol visibility
- Position-independent code
- Link order and binary size

### Article: Executable Formats and Program Startup

- ELF, PE, and Mach-O
- Headers, sections, and segments
- Entry points
- Dynamic loaders
- PLT, GOT, and lazy binding
- Program arguments and environment
- Runtime initialization
- ASLR, PIE, stack canaries, and non-executable memory

# Stage 5 — Processes, Threads, and Concurrency Models

## Subject area 5.1 — Processes and Threads

### Article: Process Isolation and Lifecycle

- Address spaces
- Process creation and replacement
- Process termination
- Parent-child relationships
- Process supervision
- Worker processes and process pools

### Article: Threads and Shared Execution State

- User and kernel threads
- Thread-local storage
- Shared memory
- Thread creation and shutdown
- Thread pools
- Thread exhaustion

### Article: Scheduling, Affinity, and NUMA Effects

- CPU affinity
- Pinning
- Load balancing
- NUMA locality
- Priority inversion
- Starvation
- Real-time scheduling

## Subject area 5.2 — Choosing a Concurrency Model

### Article: Threads, Processes, Async I/O, and Event Loops

- Multi-threading
- Multi-processing
- Single-threaded event loops
- Async runtimes
- Coroutines and green threads
- Actors
- Structured concurrency
- Choosing based on workload and failure behavior

### Article: Queues, Pipelines, Backpressure, and Cancellation

- Work queues
- Pipeline stages
- Bounded queues
- Backpressure
- Cancellation
- Timeouts
- Graceful shutdown

# Stage 6 — Memory Management

## Subject area 6.1 — Virtual Memory

### Article: Virtual Memory and Process Address Spaces

- Virtual and physical addresses
- Address-space isolation
- Memory permissions
- Shared pages
- Copy-on-write
- Guard pages
- Address randomization

### Article: Page Tables and Address Translation

- Page sizes
- Multi-level page tables
- Page-table entries
- Address translation
- TLBs and TLB misses
- Huge pages

### Article: Page Faults, Swapping, and Working Sets

- Minor and major page faults
- Demand paging
- Swapping
- Working sets
- Page replacement
- Thrashing
- Memory pressure and overcommit

### Article: Memory-Mapped Files and `mmap`

- File-backed mappings
- Anonymous mappings
- Shared mappings
- Copy-on-write mappings
- Mapping permissions
- Shared-memory use cases
- Mapping failure and lifetime

## Subject area 6.2 — Process Memory and Allocation

### Article: Stack, Heap, and Process Memory Layout

- Code, data, BSS, heap, and stack
- Shared libraries
- Stack growth
- Heap growth
- Guard pages
- Lifetime and ownership

### Article: Memory Allocators and Fragmentation

- `malloc` and `free`
- Bump allocators
- Pools and arenas
- Slab allocation
- Internal and external fragmentation
- Allocation contention
- Per-thread caches
- Allocator benchmarking

### Article: Memory Safety and Detection

- Use-after-free
- Double-free
- Buffer overflows
- Out-of-bounds access
- Memory leaks
- Integer overflow
- Uninitialized memory
- AddressSanitizer, MemorySanitizer, Valgrind, and fuzzing

### Article: Zero-Copy, DMA, and High-Performance Buffers

- Copying costs
- Zero-copy paths
- Scatter-gather I/O
- DMA buffers
- Pinned memory
- Buffer reuse
- Page-cache interaction

# Stage 7 — Filesystems, Devices, and Storage I/O

## Subject area 7.1 — File Descriptors and Filesystem Interfaces

### Article: File Descriptors and Open File Objects

- File descriptors
- Open-file descriptions
- Descriptor tables
- Inheritance
- Duplication
- Close-on-exec
- Descriptor leaks
- Resource limits

### Article: Filesystem Paths, Inodes, and the VFS

- Virtual filesystem layers
- Inodes
- Directory entries
- Path resolution
- Mounts
- Mount namespaces
- File metadata
- Sparse files and extended attributes

### Article: File Permissions, ACLs, and Capabilities

- Users and groups
- Unix permissions
- Setuid and setgid
- Sticky bit
- ACLs
- Capabilities
- Umask
- Access checks

## Subject area 7.2 — File I/O and Correctness

### Article: Buffered, Direct, Blocking, and Non-Blocking File I/O

- `open`, `read`, `write`, `pread`, and `pwrite`
- Short reads and short writes
- Vectors and scatter-gather operations
- Blocking and non-blocking behavior
- Buffered I/O
- Direct I/O and `O_DIRECT`

### Article: The Page Cache and Durability

- Read caching
- Write caching
- Dirty pages and writeback
- Cache eviction
- Double buffering
- `fsync` and `fdatasync`
- Flush behavior
- Durability guarantees

### Article: Filesystem Locks and Crash Consistency

- Advisory and mandatory locks
- Atomic rename
- Journaling
- Write ordering
- Metadata consistency
- Power-loss behavior
- Corruption and repair

## Subject area 7.3 — Storage Hardware

### Article: Disks, SSDs, NVMe, and RAID

- HDD and SSD behavior
- NVMe queues
- IOPS, throughput, and latency
- Queue depth
- RAID
- Write amplification
- Wear leveling
- Garbage collection
- TRIM and device health

# Stage 8 — Concurrency and Interprocess Communication

## Subject area 8.1 — Synchronization

### Article: Mutexes, Semaphores, and Condition Variables

- Mutual exclusion
- Waiting and notification
- Critical sections
- Futexes
- Spinlocks
- Semaphores
- Condition variables
- Lock contention

### Article: Atomics and Memory Ordering in Programs

- Atomic reads and writes
- Compare-and-swap
- Fetch-and-add
- Acquire and release
- Sequential consistency
- Relaxed ordering
- Fences

### Article: Advanced Locking and Lock-Free Reclamation

- Reader-writer locks
- Optimistic concurrency
- RCU
- Seqlocks
- Lock-free and wait-free structures
- ABA problem
- Hazard pointers
- Epoch-based reclamation

### Article: Deadlocks, Livelocks, and Priority Inversion

- Deadlock conditions
- Lock ordering
- Livelock
- Starvation
- Priority inversion
- Lost wakeups
- Shutdown races
- Recovery strategies

## Subject area 8.2 — IPC

### Article: Pipes, FIFOs, Signals, and Message Queues

- Anonymous pipes
- Named pipes
- Signals
- POSIX message queues
- Message boundaries
- Blocking behavior
- IPC permissions

### Article: Unix Sockets and Shared Memory IPC

- Unix domain sockets
- Shared memory
- Memory-mapped IPC
- Inter-process synchronization
- File-descriptor passing
- Choosing between copying and shared memory

# Stage 9 — Networking and Protocols

## Subject area 9.1 — Network Foundations

### Article: How Data Moves Through a Network

- Ethernet frames
- MAC addresses
- IP packets
- Routing
- ARP
- Subnets
- MTU and fragmentation
- NAT
- Firewalls
- Ports

### Article: DNS and Name Resolution

- Resolvers
- Recursive and authoritative servers
- DNS records
- TTLs
- Caching
- Negative caching
- Local hosts files
- Split-horizon DNS
- DNS failure modes

## Subject area 9.2 — Transport Protocols

### Article: TCP: Connection Establishment and Reliable Delivery

- TCP segment structure
- Ports, sequence numbers, and acknowledgment numbers
- Flags and options
- Three-way handshake
- Connection state machine
- Ordered byte streams
- Retransmission
- Checksums
- Packet captures

### Article: TCP Flow Control, Congestion Control, and Teardown

- Receive windows
- Send windows
- Slow start
- Congestion avoidance
- Fast retransmit
- Delayed acknowledgments
- Nagle's algorithm
- Connection teardown
- `TIME_WAIT`
- TCP failure behavior

### Article: UDP and Datagram Communication

- UDP header and message boundaries
- No connection handshake
- Loss, duplication, and reordering
- Broadcast and multicast
- Application-level reliability
- Datagram sizing
- When UDP is appropriate

### Article: QUIC and Modern Transport Design

- Why QUIC runs over UDP
- Encrypted transport
- Streams
- Connection migration
- Reduced handshake latency
- Relationship to HTTP/3

## Subject area 9.3 — Sockets and High-Concurrency I/O

### Article: The Socket API

- Socket lifecycle
- Bind, listen, accept, connect, send, and receive
- Half-close
- Socket buffers
- Keep-alive
- Reuse options
- Timeouts
- Common socket errors

### Article: Blocking, Non-Blocking, and Event-Driven I/O

- Blocking I/O
- Non-blocking I/O
- `select`, `poll`, `epoll`, and `kqueue`
- Readiness versus completion
- Event loops
- Connection management

### Article: `io_uring`, Kernel Bypass, and Specialized Networking

- Completion-based I/O
- `io_uring`
- DPDK
- RDMA
- Kernel bypass
- When specialized paths are justified

## Subject area 9.4 — Secure and Application Protocols

### Article: TLS: Handshakes, Certificates, and Session Protection

- Encryption and authentication
- TLS handshake
- Certificates and certificate authorities
- Key exchange
- Forward secrecy
- Session resumption
- Mutual TLS
- Certificate rotation
- TLS termination

### Article: SSH: Secure Remote Access and Channels

- Protocol negotiation
- Key exchange
- Host-key verification
- User authentication
- Session encryption
- Shells, file transfer, and port forwarding
- SSH channels
- Inspecting the handshake with verbose logs

### Article: HTTP, RPC, and Service Protocols

- HTTP/1.1, HTTP/2, and HTTP/3
- Request and response structure
- REST and RPC
- gRPC
- WebSockets
- Message framing
- JSON and Protobuf
- Versioning and schema evolution

# Stage 10 — Debugging, Performance, and Observability

## Subject area 10.1 — Debugging

### Article: A Systematic Debugging Workflow

- Reproducing failures
- Forming hypotheses
- Collecting evidence
- Reducing test cases
- Binary search through changes
- Debugging production safely
- Debugging nondeterministic behavior

### Article: GDB, LLDB, Core Dumps, and Post-Mortem Debugging

- Breakpoints
- Watchpoints
- Stack traces
- Registers
- Memory inspection
- Threads
- Core dumps
- Optimized binaries
- Remote debugging

### Article: Tracing System and Process Behavior

- `strace`
- `ltrace`
- Process tracing
- Syscall tracing
- Scheduling traces
- I/O traces
- Network traces
- DTrace and SystemTap

## Subject area 10.2 — Performance

### Article: Profiling CPU, Memory, Locks, and I/O

- Sampling and instrumentation
- `perf`
- Flame graphs
- CPU profiling
- Allocation profiling
- Lock profiling
- I/O profiling
- Hardware counters

### Article: Measuring Latency, Throughput, and Saturation

- Latency distributions
- Percentiles and tail latency
- Throughput
- Queueing
- Saturation
- Bottlenecks
- Benchmark design
- Before-and-after validation

## Subject area 10.3 — Observability

### Article: Logs, Metrics, and Distributed Traces

- Structured logs
- Counters, gauges, and histograms
- Cardinality
- Correlation IDs
- Context propagation
- Trace sampling
- Dashboards
- Alert design

### Article: eBPF and Dynamic Observability

- eBPF execution model
- Tracepoints
- Kprobes and uprobes
- BPF maps
- Network observability
- Security monitoring
- Production safety

# Stage 11 — Systems Security

## Subject area 11.1 — Security Models

### Article: Trust Boundaries, Threats, and Least Privilege

- Confidentiality, integrity, and availability
- Threat modeling
- Attack surfaces
- Trust boundaries
- Security assumptions
- Defense in depth
- Least privilege

### Article: Authentication, Authorization, and Identity

- Authentication versus authorization
- Sessions and tokens
- OAuth and OpenID Connect
- Service identities
- RBAC and ABAC
- Capabilities
- Privilege separation

## Subject area 11.2 — Isolation

### Article: Sandboxing and Process Isolation

- `chroot`
- Namespaces
- seccomp
- Capabilities
- Application sandboxing
- Isolation limitations

### Article: Mandatory Access Control

- SELinux
- AppArmor
- Policy decisions
- Labels and profiles
- Debugging access denials

## Subject area 11.3 — Secure Development and Operations

### Article: Memory Safety and Secure Coding

- Input validation
- Integer safety
- Race-condition security
- Serialization risks
- Command injection
- Path traversal
- Safe cryptographic-library use

### Article: Secrets, Certificates, and Software Supply Chains

- Secret storage
- Key rotation
- Certificate lifecycle
- Dependency security
- Reproducible builds
- Code signing
- Secure updates

### Article: Host and Service Hardening

- Network segmentation
- Host hardening
- Patch management
- Security logging
- Auditing
- Vulnerability response
- Incident response

# Stage 12 — Virtualization, Containers, and Infrastructure

## Subject area 12.1 — Virtualization and Containers

### Article: Virtual Machines and Hypervisors

- CPU virtualization
- Memory virtualization
- Virtual devices
- Virtual disks
- Snapshots
- Live migration
- Isolation and overhead

### Article: How Containers Work

- Namespaces
- cgroups
- Container filesystems
- Images and layers
- OCI runtimes
- Rootless containers
- Container networking and storage
- Container security

### Article: Kubernetes as a Control System

- Pods, deployments, and services
- Scheduling
- Health checks
- ConfigMaps and secrets
- Volumes
- Stateful workloads
- Autoscaling
- Operators
- Network policies
- Cluster failure modes

## Subject area 12.2 — Infrastructure

### Article: Infrastructure as Code and Configuration Management

- Declarative infrastructure
- Terraform and Pulumi concepts
- State
- Drift
- Modules
- Environment promotion
- Infrastructure testing
- Safe changes and rollback

### Article: Cloud Infrastructure Fundamentals

- Compute
- Virtual networks
- Subnets and routing
- Firewalls
- Load balancers
- Managed databases
- Queues
- Object and block storage
- Regions and availability zones
- Cost and vendor lock-in

### Article: Control Planes, Data Planes, and Service Discovery

- Control-plane responsibilities
- Data-plane traffic
- Service discovery
- Health checking
- Configuration distribution
- Leases
- Dependency graphs

# Stage 13 — Distributed Systems

## Subject area 13.1 — Distributed Failure and Time

### Article: Why Distributed Systems Fail Differently

- Partial failure
- Network failure
- Process failure
- Delays, loss, duplication, and reordering
- Split brain
- Failure domains
- Failure detectors

### Article: Time, Clocks, and Ordering

- Clock skew and drift
- NTP
- Logical clocks
- Lamport clocks
- Vector clocks
- Causality
- Happens-before relationships
- Hybrid logical clocks

## Subject area 13.2 — Replication and Consensus

### Article: Replication and Consistency Models

- Primary-backup replication
- Leader and multi-leader replication
- Leaderless replication
- Synchronous and asynchronous replication
- Strong and eventual consistency
- Read-your-writes
- Linearizability
- Quorums
- Anti-entropy

### Article: Consensus and Raft

- Why consensus is needed
- Leader election
- Terms and epochs
- Log replication
- Commit indexes
- Membership changes
- Quorums
- Split brain and fencing

### Article: Paxos and Consensus Tradeoffs

- The problem Paxos solves
- Roles and phases
- Safety and liveness
- Relationship to Raft
- When consensus is unnecessary

## Subject area 13.3 — Partitioning and Distributed Coordination

### Article: Sharding and Partitioning

- Hash and range partitioning
- Consistent hashing
- Shard keys
- Hot partitions
- Rebalancing
- Resharding
- Tenant isolation
- Cross-shard operations

### Article: Distributed Transactions and Sagas

- Atomicity across services
- Two-phase commit
- Sagas
- Compensating actions
- Outbox and inbox patterns
- Deduplication
- Idempotency
- Reconciliation

### Article: Service Discovery, Gossip, and Health Checking

- Client-side and server-side discovery
- Load balancing
- Heartbeats
- Leases
- Gossip
- Health checks
- Dependency failure

# Stage 14 — Databases and Storage Engines

## Subject area 14.1 — Database Foundations

### Article: Relational Data Modeling and SQL

- Tables, relations, keys, and constraints
- Normalization and denormalization
- Transactions
- Query structure
- Data ownership
- Access patterns

### Article: Query Execution and Indexes

- Query planning
- Cost-based optimization
- Table scans
- B-trees and indexes
- Selectivity
- Covering indexes
- Query performance investigation

## Subject area 14.2 — Storage Engine Internals

### Article: Pages, Records, and Buffer Pools

- Database pages
- Record layout
- Slotted pages
- Buffer pools
- Page cache
- Page replacement
- Storage layout

### Article: B-Trees and LSM-Trees

- B-tree structure
- LSM-tree structure
- Write amplification
- Read amplification
- Compaction
- Bloom filters
- Choosing between them

### Article: Write-Ahead Logging and Crash Recovery

- WAL records
- Durability
- Redo and undo
- Checkpoints
- Commit records
- Fsync
- Torn writes
- Crash recovery
- Recovery testing

## Subject area 14.3 — Transactions and Operations

### Article: Isolation Levels, MVCC, and Locking

- Read phenomena
- Read committed
- Repeatable read
- Serializable isolation
- MVCC
- Two-phase locking
- Optimistic and pessimistic locking
- Deadlocks

### Article: Replication, Failover, and Online Migration

- Primary and replica databases
- Replication lag
- Synchronous and asynchronous replication
- Failover
- Split brain
- Schema compatibility
- Online schema changes

### Article: Backups, Restore, and Disaster Recovery for Data

- Full and incremental backups
- Snapshots
- Point-in-time recovery
- Backup consistency
- Retention
- Encryption
- Restore testing
- RTO and RPO

### Article: Non-Relational Storage Systems

- Key-value stores
- Document databases
- Wide-column stores
- Graph databases
- Time-series databases
- Search engines
- Object storage
- Choosing storage from access patterns

# Stage 15 — Backend and Service Engineering

## Subject area 15.1 — APIs and Service Boundaries

### Article: API Design and Compatibility

- Resource modeling
- REST and RPC
- Request validation
- Errors
- Pagination
- Filtering
- Idempotency keys
- Authentication and authorization
- Versioning
- Deprecation
- Backward compatibility

### Article: Monoliths, Modular Monoliths, and Microservices

- Service boundaries
- Domain ownership
- Shared libraries
- Deployment boundaries
- Synchronous and asynchronous communication
- Decomposition and migration
- Operational cost of services

## Subject area 15.2 — Messaging and Resilience

### Article: Queues, Topics, and Event-Driven Systems

- Message brokers
- Queues and topics
- Consumer groups
- Delivery guarantees
- Ordering
- Replay
- Dead-letter queues
- Poison messages
- Event schemas
- Event versioning

### Article: Retries, Timeouts, and Failure Isolation

- Timeouts
- Retries
- Exponential backoff
- Jitter
- Retry storms
- Circuit breakers
- Bulkheads
- Load shedding
- Graceful degradation

### Article: Caching and Invalidation

- Local and distributed caches
- Cache-aside
- Read-through and write-through
- TTLs
- Invalidation
- Cache stampedes
- Cache warming
- Negative caching
- Hot keys

### Article: Idempotency and Exactly-Once Claims

- Duplicate requests
- Idempotency keys
- At-least-once delivery
- Deduplication
- Exactly-once processing within a defined boundary
- Outbox and inbox patterns

# Stage 16 — Production Engineering and Reliability

## Subject area 16.1 — Reliability Objectives

### Article: SLIs, SLOs, SLAs, and Error Budgets

- Service-level indicators
- Service-level objectives
- Service-level agreements
- Error budgets
- Availability
- Durability
- Latency objectives
- Burn-rate alerts

### Article: Capacity Planning and Load Testing

- Traffic estimation
- CPU, memory, storage, and network estimates
- Headroom
- Scaling limits
- Load tests
- Queueing behavior
- Cost modeling

## Subject area 16.2 — Safe Delivery

### Article: Continuous Delivery and Deployment Safety

- Build artifacts
- Artifact promotion
- Feature flags
- Rolling deployments
- Blue-green deployments
- Canary deployments
- Shadow traffic
- Rollbacks
- Database migration safety

### Article: Rate Limiting, Backpressure, and Overload Control

- Token buckets
- Leaky buckets
- Concurrency limits
- Queue bounds
- Backpressure
- Load shedding
- Protecting dependencies

## Subject area 16.3 — Incidents and Recovery

### Article: Incident Response and Postmortems

- Detection
- Triage
- Severity
- Incident command
- Communication
- Mitigation
- Rollback
- Recovery
- Postmortems
- Corrective actions

### Article: Chaos Engineering and Failure Injection

- Failure hypotheses
- Dependency failure
- Network latency and packet loss
- Disk-full conditions
- Memory pressure
- CPU saturation
- Certificate expiration
- Clock problems
- Recovery drills

### Article: Disaster Recovery and Multi-Region Systems

- Availability zones
- Regions
- Failover and failback
- RTO and RPO
- Recovery runbooks
- Dependency recovery
- Regional failure
- Business continuity

### Article: Cost, Efficiency, and Operational Tradeoffs

- Cost per request
- Compute and storage efficiency
- Network cost
- Overprovisioning
- Autoscaling
- Reliability versus cost
- Performance versus cost
- FinOps basics

# Stage 17 — System Design and Architecture

## Subject area 17.1 — Designing Systems

### Article: Requirements and Capacity Estimation for System Design

- Functional requirements
- Non-functional requirements
- Traffic assumptions
- Data size
- Latency and availability goals
- Security and compliance
- Budget and organizational constraints

### Article: Designing for Reads, Writes, and Hot Paths

- Read-heavy and write-heavy systems
- Vertical and horizontal scaling
- Stateless and stateful services
- Hot paths
- Bottlenecks
- Caching
- Queuing
- Asynchronous processing

### Article: Architecture Patterns and Boundaries

- Layered architecture
- Modular monoliths
- Microservices
- Event-driven architecture
- Pipelines
- CQRS
- Event sourcing
- Actors
- Control plane and data plane
- Strangler migrations

### Article: Data, API, and Multi-Tenant Architecture

- Data ownership
- Schema design
- Data lifecycle
- Retention
- Migration
- API contracts
- Schema evolution
- Auditing
- Privacy
- Multi-tenancy

### Article: Evaluating an Architecture

- Failure-mode analysis
- Threat modeling
- Capacity and cost estimation
- Operational complexity
- Testing strategy
- Migration plan
- Rollback plan
- Observability plan
- Alternatives considered

## Subject area 17.2 — Complete Design Exercises

### Article: Designing Common Backend Systems

- URL shortener
- Rate limiter
- Notification system
- File-storage service
- Search system
- Feature-flag service
- Job scheduler
- Social feed

### Article: Designing Infrastructure and Distributed Systems

- Distributed cache
- Message queue
- Metrics platform
- Log aggregation system
- Distributed lock service
- Database replication system
- Multi-region API
- Video-processing platform

# Stage 18 — Specializations

Take these after the core path or when a career direction requires them.

## Subject area 18.1 — Kernel and Operating-System Development

### Article: Kernel Modules, Drivers, and Kernel Debugging

- Kernel modules
- Device drivers
- Kernel debugging
- Kernel synchronization
- Kernel tracing

### Article: Kernel Memory, Scheduling, Filesystems, and Networking

- Kernel memory management
- Scheduler internals
- Filesystem development
- Network-stack internals

## Subject area 18.2 — High-Performance and Specialized Systems

### Article: High-Performance Computing

- SIMD and vectorization
- GPU computing
- MPI
- NUMA optimization
- Cache-aware algorithms
- RDMA
- Kernel bypass

### Article: Embedded and Real-Time Systems

- Microcontrollers
- Bare-metal programming
- Interrupt-driven design
- RTOS
- Hard and soft real time
- Firmware updates
- Deterministic behavior

### Article: Storage and Filesystem Development

- FUSE
- Block-layer programming
- RAID systems
- Distributed filesystems
- Object-storage internals
- Erasure coding
- Data integrity and recovery

### Article: Networking Infrastructure

- Routers and switches
- Load balancers
- Proxies and firewalls
- BGP
- Software-defined networking
- DPDK
- RDMA

### Article: Formal Methods and Verification

- Invariants
- Model checking
- Temporal logic
- TLA+
- Alloy
- Protocol verification
- Model-based testing

# Progressive Projects

Projects are optional and should be built during longer breaks. They are ordered so that each project reuses ideas from earlier articles.

## Project 1 — Systems Utility

Build a command-line utility that reads files, validates input, handles errors, uses system calls, supports useful logging, and has tests.

## Project 2 — A Small Shell

Implement command execution, pipelines, redirection, environment variables, background processes, signals, and process groups.

## Project 3 — A TCP and UDP Server in Go

Build a TCP echo server and a UDP message server. Compare connection setup, message boundaries, failure behavior, timeouts, and concurrent clients.

## Project 4 — A Memory Allocator

Implement allocation, deallocation, alignment, splitting, coalescing, fragmentation measurement, and optional thread safety.

## Project 5 — A Key-Value Store

Build an in-memory store, add a WAL, recovery, checksums, indexing, compaction, and benchmarks.

## Project 6 — A Replicated Service

Build multiple nodes with replication, leader election, failure detection, retries, idempotency, metrics, and fault injection.

## Project 7 — A Production Service

Deploy a service with containers, CI/CD, infrastructure as code, TLS, authentication, metrics, logs, traces, alerts, SLOs, load tests, backups, and recovery documentation.

## Project 8 — Incident Simulations

Simulate high CPU, memory leaks, disk exhaustion, network latency, packet loss, dependency outages, expired certificates, replication lag, bad deployments, and regional failure. Investigate each failure and document the recovery process.

# How each blog will be written

The blog format is adaptive rather than mandatory. Every article should explain its concept deeply enough to understand the mechanism, observe its behavior when useful, reason about failures, and discuss the engineering tradeoffs that matter.

Use only the tools that help that article:

- Simple mental models
- Diagrams
- Code examples
- Packet, file, or message layouts
- State machines
- Internal data structures
- Production scenarios
- Failure analysis
- Performance analysis
- Security analysis
- Interview definitions and follow-ups
- Common misconceptions
- Optional projects
- Links to related articles

Do not add a section merely to satisfy a template. A file-descriptors article and a TCP article should not have the same shape. The article should be as long as necessary to be genuinely useful and no longer.
