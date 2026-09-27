# Resource Isolation: Namespaces and Cgroups

Containerization in Linux is not achieved through a monolithic kernel hypervisor or dedicated virtual machine abstraction layer. Instead, what modern systems refer to as a "container" is simply a standard Linux process governed by two orthogonal kernel mechanisms: **Namespaces** and **Control Groups (cgroups)**.

While **Namespaces** control *what a process can see* by virtualizing global operating system resources (such as process tables, network stacks, filesystem mount points, and user IDs), **Control Groups** govern *what a process can consume* by metering, prioritizing, and limiting resource consumption (such as CPU bandwidth, physical memory, block I/O throughput, and process counts).

This module deconstructs the low-level kernel foundations of containment, analyzes the architectural evolution to unified cgroups v2, and demonstrates how to assemble secure, isolated execution sandboxes using native command-line utilities.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Linux Userspace Boundary                        │
│   Container Runtimes (crun, runc, systemd-nspawn) & Native Utilities   │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │                                │
                    ▼                                ▼
┌───────────────────────────────────┐┌───────────────────────────────────┐
│     Virtualization of View:       ││     Regulation of Consumption:    │
│        Linux Namespaces           ││          cgroups v2               │
│                                   ││                                   │
│  - PID (Hierarchical trees)       ││  - Unified Hierarchy (`/sys/fs/cgroup`)
│  - Mount (VFS visibility & roots) ││  - Memory (`memory.max`, `high`)  │
│  - Network (Interfaces, routing)  ││  - CPU (`cpu.max`, `cpu.weight`)  │
│  - User (UID/GID translation)     ││  - I/O (`io.max`, `io.weight`)    │
│  - IPC (SysV / POSIX message queues││ - PIDs (`pids.max` fork-bomb def)│
│  - UTS (Hostname, domain)         ││  - PSI (Pressure Stall Info)      │
└───────────────────┬───────────────┘└──────────────────┬────────────────┘
                    │                                   │
                    ▼                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│                          Linux Kernel Core                             │
│       `task_struct`, `nsproxy`, `struct cred`, `struct cgroup`         │
│          System Calls: `clone(2)`, `unshare(2)`, `setns(2)`            │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 1. The Architectural Foundation of Containment: Namespaces

### Virtualization of View vs. Virtualization of Hardware
Traditional Type-1 and Type-2 hypervisors (such as KVM, Xen, or VMware ESXi) emulate physical hardware—virtual CPUs, guest MMUs, network interface cards, and SCSI controllers. The guest operating system runs its own kernel binary on top of this virtual hardware layer, incurring virtualization overhead, guest memory reservations, and multi-stage scheduling penalties.

Operating system-level containment requires zero hardware virtualization. The guest workloads execute directly on the host CPU in userspace (Ring 3) through standard system calls into the host kernel (Ring 0). Isolation is established by partitioning the kernel's internal object tables via namespaces. When a process queries the system, the kernel filters the returned data based on the namespaces bound to that process's descriptor.

```
Type-2 Hypervisor Architecture               Linux Container Architecture
┌───────────────────────────────┐           ┌───────────────────────────────┐
│  Guest App    │   Guest App   │           │ Container App │ Container App │
├───────────────┼───────────────┤           ├───────────────┼───────────────┤
│ Guest Kernel  │ Guest Kernel  │           │  Namespaces   │  cgroups v2   │
├───────────────┼───────────────┤           │  (View)       │  (Limits)     │
│ Virtual HW    │ Virtual HW    │           ├───────────────┴───────────────┤
├───────────────┴───────────────┤           │                               │
│ Hypervisor / Host OS Kernel   │           │      Host Linux Kernel        │
├───────────────────────────────┤           ├───────────────────────────────┤
│ Physical Hardware (Bare Metal)│           │ Physical Hardware (Bare Metal)│
└───────────────────────────────┘           └───────────────────────────────┘
```

### Kernel Infrastructure: `task_struct`, `nsproxy`, and `/proc/[pid]/ns/`
Every schedulable entity in the Linux kernel is represented by an instance of `struct task_struct` (defined in `<linux/sched.h>`). Within this structure, two pointers establish the process's isolation boundary:

1. **`struct nsproxy *nsproxy`**: Points to a structure (defined in `<linux/nsproxy.h>`) containing pointers to the specific namespace instances bound to the task:
   * `uts_ns`: UTS namespace pointer.
   * `ipc_ns`: System V / POSIX IPC namespace pointer.
   * `mnt_ns`: Mount namespace pointer.
   * `pid_ns_for_children`: PID namespace in which child processes will be spawned.
   * `net_ns`: Network stack namespace pointer.
   * `cgroup_ns`: Control group namespace pointer.
   * `time_ns`: Virtualized system boot/monotonic time offsets.
2. **`struct cred *cred`**: Tracks user and group credentials. Crucially, the pointer to the process's **User Namespace** (`struct user_namespace *user_ns`) is located inside `struct cred`, not in `nsproxy`. This design choice exists because all security authorizations, POSIX capability vectors, and UID/GID lookups depend directly on the user credential context.

```
                          Kernel Memory Space
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      struct task_struct (PCB)                          │
 │  ├── pid_t pid;                                                        │
 │  ├── struct cred *cred; ───► struct user_namespace *user_ns            │
 │  └── struct nsproxy *nsproxy;                                          │
 └────────────────────────────┬───────────────────────────────────────────┘
                              │
                              ▼
 ┌────────────────────────────────────────────────────────────────────────┐
 │                        struct nsproxy                                  │
 │  ├── struct uts_namespace *uts_ns;                                     │
 │  ├── struct ipc_namespace *ipc_ns;                                     │
 │  ├── struct mnt_namespace *mnt_ns;                                     │
 │  ├── struct pid_namespace *pid_ns_for_children;                        │
 │  ├── struct net           *net_ns;                                     │
 │  ├── struct cgroup_namespace *cgroup_ns;                               │
 │  └── struct time_namespace   *time_ns;                                 │
 └────────────────────────────────────────────────────────────────────────┘
```

#### Magic Namespace Symlinks in `/proc`
For every process, the kernel exposes its active namespaces via the virtual directory `/proc/[pid]/ns/`. Each entry is a magic symbolic link whose target format is `namespace_type:[inode_number]`:

```bash
$ ls -l /proc/$$/ns/
total 0
lrwx------ 1 root root 0 Sep 27 10:14 cgroup -> 'cgroup:[4026531835]'
lrwx------ 1 root root 0 Sep 27 10:14 ipc -> 'ipc:[4026531839]'
lrwx------ 1 root root 0 Sep 27 10:14 mnt -> 'mnt:[4026531840]'
lrwx------ 1 root root 0 Sep 27 10:14 net -> 'net:[4026531992]'
lrwx------ 1 root root 0 Sep 27 10:14 pid -> 'pid:[4026531836]'
lrwx------ 1 root root 0 Sep 27 10:14 pid_for_children -> 'pid:[4026531836]'
lrwx------ 1 root root 0 Sep 27 10:14 time -> 'time:[4026531834]'
lrwx------ 1 root root 0 Sep 27 10:14 user -> 'user:[4026531837]'
lrwx------ 1 root root 0 Sep 27 10:14 uts -> 'uts:[4026531838]'
```

The numeric value in brackets is the kernel's internal **inode number** identifying the active namespace object. 
* If two processes exhibit identical inode values for a given namespace type, they inhabit the exact same isolation domain.
* If a process opens a file descriptor to one of these pseudo-links (e.g., `open("/proc/1234/ns/net", O_RDONLY)`), the kernel keeps that namespace alive in memory even if every process residing inside it terminates.

---

### The System Call Triad: `clone(2)`, `unshare(2)`, and `setns(2)`

The Linux kernel exposes three primary system calls for manipulating namespaces:

```c
#define _GNU_SOURCE
#include <sched.h>

// 1. Create a child process in new namespaces:
int clone(int (*fn)(void *), void *stack, int flags, void *arg, ... 
          /* pid_t *parent_tid, void *tls, pid_t *child_tid */ );

// 2. Disassociate current process from shared namespaces into new ones:
int unshare(int flags);

// 3. Attach current process to an existing namespace via an open FD:
int setns(int fd, int nstype);
```

#### 1. `clone(2)`: Instantiation
Unlike `fork(2)`, which duplicates the parent process with identical namespace memberships, `clone(2)` accepts bitwise flags (`CLONE_NEW*`). If one or more `CLONE_NEW` flags are asserted, the kernel instantiates brand new namespace instances for the child task, leaving the parent unchanged.

#### 2. `unshare(2)`: Self-Detachment
`unshare(2)` allows a process to break away from its current shared environment without spawning a new task. The calling process drops its reference to the inherited namespace and attaches to a newly allocated namespace instance.

#### 3. `setns(2)`: Reassociation
`setns(2)` allows a process to attach to a pre-existing namespace. The process passes a file descriptor pointing to a `/proc/[pid]/ns/<type>` file. This system call is the foundation of debugging utilities like `docker exec` and `nsenter(1)`.

---

### Deep Dive: The Core Linux Namespaces

```
┌─────────────────┬───────────────────┬────────────────────────────────────────────────────────┐
│ Namespace Type  │ Clone Flag        │ Primary Isolated Kernel Resources                      │
├─────────────────┼───────────────────┼────────────────────────────────────────────────────────┤
│ **UTS**         │ `CLONE_NEWUTS`    │ Hostname and NIS domain name                           │
│ **IPC**         │ `CLONE_NEWIPC`    │ System V IPC objects, POSIX message queues, semaphores │
│ **PID**         │ `CLONE_NEWPID`    │ Process ID numbers, parent-child hierarchies, init     │
│ **Mount**       │ `CLONE_NEWNS`     │ Virtual File System (VFS) mount table, root directory  │
│ **Network**     │ `CLONE_NEWNET`    │ Network devices, IP addresses, routes, firewalls, ports│
│ **User**        │ `CLONE_NEWUSER`   │ UID/GID credential mappings, POSIX capability sets     │
│ **Cgroup**      │ `CLONE_NEWCGROUP` │ Virtualized `/proc/self/cgroup` view                   │
│ **Time**        │ `CLONE_NEWTIME`   │ Monotonic and boot system clocks (offsets)             │
└─────────────────┴───────────────────┴────────────────────────────────────────────────────────┘
```

#### 1. UTS Namespace (`CLONE_NEWUTS`)
The UTS (UNIX Timesharing System) namespace isolates two system identifiers set via `sethostname(2)` and `setdomainname(2)`:
* The system hostname.
* The NIS domain name.

Without UTS isolation, changing the hostname inside a container alters the host's actual network hostname. UTS isolation ensures that web services, mail servers, and shell prompts identify themselves independently of the underlying host.

#### 2. IPC Namespace (`CLONE_NEWIPC`)
The IPC namespace isolates inter-process communication resources:
* System V IPC objects: Shared memory segments (`shmget`), message queues (`msgget`), and semaphore sets (`semget`).
* POSIX message queues (`mq_open`).

Processes within distinct IPC namespaces cannot interact via shared memory segments or message queues, even if they execute under identical UIDs. Each IPC namespace maintains its own independent IPC identifier and file system projection in `/dev/mqueue`.

#### 3. PID Namespace (`CLONE_NEWPID`)
The PID namespace virtualizes the process identification numbers. Processes inside a newly instantiated PID namespace have an independent sequence of PIDs starting at $\text{PID } 1$.

```
               Host / Root PID Namespace
 ┌───────────────────────────────────────────────────────┐
 │ PID 1 (systemd)                                       │
 │  ├── PID 890 (dockerd)                                │
 │  │    └── PID 2401 (container-shim)                   │
 │  │         └── PID 2402 (web-server) ────────┐        │
 │  └── PID 1045 (sshd)                         │        │
 └──────────────────────────────────────────────┼────────┘
                                                │
                                                ▼ Maps to
               Child PID Namespace
 ┌───────────────────────────────────────────────────────┐
 │ PID 1 (web-server)                                    │
 │  └── PID 2 (worker-process)                           │
 └───────────────────────────────────────────────────────┘
```

##### Architectural Mechanics:
1. **PID Virtualization and Duality**: A process possesses multiple PIDs—one for each namespace in which it is nested. As illustrated above, the process known as `PID 2402` on the host root namespace is simultaneously `PID 1` inside its container namespace.
2. **The Role of PID 1 (Init)**:
   * The first process created inside a new PID namespace becomes `PID 1` for that namespace.
   * It assumes initialization duties: it must adopt orphaned child tasks (`PR_SET_CHILD_SUBREAPER` behavior is automatic for namespace init) and reap zombies via `waitpid(2)`.
   * **Signal Immunity**: In Linux, `PID 1` ignores standard default signal dispositions. If a process inside the namespace issues `kill -9 1` (SIGKILL) or `kill -15 1` (SIGTERM), the kernel discards the signal unless `PID 1` explicitly registered a custom signal handler for it.
   * **Parent Namespace Termination**: If `PID 1` inside a child namespace is killed by a process residing in an *ancestor* (parent) namespace, the kernel sends a non-maskable `SIGKILL` to *all* other processes inhabiting that child namespace, effectively terminating the entire container.
3. **The `CLONE_NEWPID` Trap with `unshare(2)`**:
   When a process invokes `unshare(CLONE_NEWPID)`, **the calling process does not move into the new PID namespace**. Instead, only *subsequent children* spawned via `fork()` or `clone()` will reside in the new namespace. The first child spawned becomes `PID 1`. This is why invoking `unshare -p` from the terminal without `--fork` causes system utilities to fail or crash.

#### 4. Mount Namespace (`CLONE_NEWNS`)
Historically the first namespace introduced to Linux (kernel 2.4.19), it retained the generic flag `CLONE_NEWNS`. It isolates the list of mounted filesystems (the VFS mount table) seen by a process. Processes in different mount namespaces can mount, remount, or unmount storage devices, loop images, and pseudo-filesystems without affecting the rest of the host.

##### Mount Propagation Mechanics
When filesystems are mounted inside modern systems, changes might need to propagate across namespaces (e.g., mounting an optical disc or USB drive in the host should appear inside a desktop session). The kernel handles this via four propagation modes:
* **`shared`**: Mount/unmount events propagate bidirectionally. A mount created inside this namespace appears in peer namespaces, and vice versa.
* **`slave`**: Propagation is unidirectional. Mount events created on the host propagate into the container, but container mounts do not propagate back to the host.
* **`private`**: Completely isolated. No mount events cross the boundary in either direction.
* **`unbindable`**: A private mount that cannot be cloned via `mount --bind`, preventing recursive bind loops.

```bash
# Remount the host root filesystem recursively as private to prevent leaks:
mount --make-rprivate /
```

#### 5. Network Namespace (`CLONE_NEWNET`)
The network namespace provides complete virtualization of the system networking subsystem. Each network namespace possesses its own:
* Physical and virtual network interfaces (e.g., `eth0`, `lo`, `veth*`).
* IPv4 and IPv6 protocol stacks, routing tables, and ARP/neighbor caches.
* Sockets and port allocation tables (`0.0.0.0:80` inside a netns does not conflict with `0.0.0.0:80` on the host).
* Firewall tables (Netfilter `iptables`, `nftables`, IPVS).
* Configuration interfaces in `/proc/net` and `/sys/class/net`.

##### The Virtual Ethernet Pair (`veth`) Conduit
Because physical network interfaces belong to at most one network namespace at a time, containers communicate with the external host via **Virtual Ethernet (`veth`) pairs**. A `veth` device acts as a bidirectional virtual wire: packets injected into one end instantly emerge from the peer end.

```
 Host Network Namespace                     Container Network Namespace
┌──────────────────────────────────────┐   ┌───────────────────────────┐
│ Host Physical Interface: `eth0`      │   │ Container Loopback: `lo`  │
│ Host Bridge: `br0` (172.20.0.1/16)   │   │                           │
│   └── Attached: `veth-host-a`        │   │ Container Interface:      │
└──────────────────┬───────────────────┘   │   `eth0` (172.20.0.2/16)  │
                   │                       └─────────────┬─────────────┘
                   │       Virtual Wire Link             │
                   └─────────────────────────────────────┘
```

#### 6. User Namespace (`CLONE_NEWUSER`)
The User namespace isolates security identities:
* User IDs (UIDs) and Group IDs (GIDs).
* Root keys and credentials.
* POSIX capabilities (`cap_sys_admin`, `cap_net_admin`, etc.).

##### Mechanics of Rootless Containers
The user namespace allows a process to have UID `0` (root) inside the container while mapping directly to an unprivileged UID (such as `1000`) on the host system.

```
       Host System Context                 Container User Namespace
┌───────────────────────────────┐         ┌───────────────────────────┐
│ Real UID: 1000 (alice)        │ ──────► │ Virtual UID: 0 (root)     │
│ Capabilities: NONE (Restricted│ Mapping │ Capabilities: ALL PERMITTED│
│ Cannot alter physical network │         │ Can mount tmpfs, run pkg  │
└───────────────────────────────┘         └───────────────────────────┘
```

Inside the user namespace, the process enjoys full administrative rights over resources *owned by that namespace*. However, if the process attempts an operation that touches host resources (such as writing to `/dev/sda` or creating a raw network socket on the host NIC), the kernel checks its credentials in the host's root user namespace, where it is restricted to UID `1000` without capabilities.

##### UID and GID Mapping Tables
Mapping is configured by writing to `/proc/[pid]/uid_map` and `/proc/[pid]/gid_map`. The format consists of lines containing three space-delimited fields:

$$\text{ID-inside-ns} \quad \text{ID-outside-ns} \quad \text{Length}$$

```text
0 100000 65536
```
This line instructs the kernel to map a range of $65\,536$ identifiers:
* Virtual UID `0` inside the namespace corresponds to host UID `100000`.
* Virtual UID `1` corresponds to host UID `100001`.
* Virtual UID `65535` corresponds to host UID `165535`.

An unprivileged user can only map their own host UID (or subordinate ranges assigned to them in `/etc/subuid` and `/etc/subgid` via the `newuidmap` and `newgidmap` setuid helpers).

---

## 2. Constraining Consumption: Control Groups (cgroups v2)

### Architectural Evolution: cgroups v1 vs. cgroups v2
Control Groups were integrated into the Linux kernel in version 2.6.24 (2008). The original design—now referred to as **cgroups v1**—suffered from deep architectural flaws:
1. **Multi-Hierarchy Complexity**: In v1, each controller (CPU, Memory, I/O, Network) operated as an independent, decoupled tree under `/sys/fs/cgroup/<controller>/`. A process could be at path `/system.slice/db` in the memory controller, but at `/user.slice/batch` in the CPU controller.
2. **Controller Disconnects**: High-speed memory allocations generate storage writeback I/O. In v1, because the memory and block I/O controllers did not share a single context, buffered I/O could not be charged back to the memory-allocating process, rendering I/O throttling ineffective for write workloads.
3. **Thread-Level Assignment Issues**: In v1, individual threads could be arbitrarily shifted between different cgroups, resulting in race conditions and inconsistent resource state.

To resolve these limitations, Linux kernel 4.5 introduced the **Unified Hierarchy (cgroups v2)**, which became the default in enterprise distributions (RHEL 9, Ubuntu 22.04+, Debian 11+, Fedora).

```
cgroups v1 (Legacy Fragmented Multi-Tree)
/sys/fs/cgroup/
├── cpu/       ───► Tree A: [Slice 1] ──► [Process 100]
├── memory/    ───► Tree B: [Group X] ──► [Process 100] (Disconnected!)
└── blkio/     ───► Tree C: [Group Y] ──► [Process 100]

cgroups v2 (Modern Unified Single-Tree)
/sys/fs/cgroup/
├── cgroup.controllers
├── cgroup.subtree_control
└── workload.slice/
    ├── cgroup.procs (Process 100 is strictly bounded across ALL controllers)
    ├── cpu.max
    ├── memory.max
    └── io.max
```

---

### The Unified Hierarchy Mechanics

Under cgroups v2, `/sys/fs/cgroup` is mounted as a single `cgroup2` filesystem:

```bash
$ mount | grep cgroup2
cgroup2 on /sys/fs/cgroup type cgroup2 (rw,nosuid,nodev,noexec,relatime,nsdelegate)
```

#### Core Operational Rules:
1. **Single Unified Hierarchy**: All controllers enabled for a process are governed along the exact same directory path.
2. **Top-Down Resource Distribution**: Controllers cannot be enabled in a child cgroup unless they have been explicitly enabled in the parent cgroup.
3. **The "No Internal Processes" Rule**: 
   * A cgroup directory cannot both **host processes** and **distribute resources to child cgroups**.
   * Except for the root cgroup, a parent cgroup containing subdirectories (children) cannot have processes listed in its `cgroup.procs` file if it has controllers enabled in `cgroup.subtree_control`.
   * All executing tasks must live in leaf nodes.

```
       Correct v2 Topology                     Invalid v2 Topology
         [Parent Cgroup]                         [Parent Cgroup]
      subtree_control: "+cpu"                 subtree_control: "+cpu"
      cgroup.procs: EMPTY                     cgroup.procs: [PID 400] ──► CONFLICT!
       ┌──────────┴──────────┐                            │
       ▼                     ▼                            ▼
 [Child Group 1]       [Child Group 2]             [Child Group 1]
 cgroup.procs: [400]   cgroup.procs: [500]         cgroup.procs: [500]
```

#### Control Files Present in Every Directory:
* **`cgroup.controllers`** (Read-Only): Lists the available controllers that *can* be activated for child groups.
* **`cgroup.subtree_control`** (Read/Write): Dictates which controllers are *actively distributed* into immediate subdirectories. Adding or removing controllers is performed using `+` or `-` flags:
  ```bash
  echo "+cpu +memory +io +pids" > /sys/fs/cgroup/custom_group/cgroup.subtree_control
  ```
* **`cgroup.procs`** (Read/Write): Lists the Thread Group IDs (PIDs) of processes assigned to this cgroup. Writing a PID into this file atomically moves the entire process and all of its constituent threads into the cgroup:
  ```bash
  echo 14201 > /sys/fs/cgroup/custom_group/leaf/cgroup.procs
  ```
* **`cgroup.events`** (Read-Only): Exposes population state (`populated 0` or `1`) and cgroup-level freeze state.

---

### Core Cgroups v2 Controllers

```
┌─────────────┬─────────────────────────────────┬────────────────────────────────────────────────────────┐
│ Controller  │ Core Interface Files            │ Operational Effect                                     │
├─────────────┼─────────────────────────────────┼────────────────────────────────────────────────────────┤
│ **cpu**     │ `cpu.max`, `cpu.weight`         │ Bandwidth throttling (CFS quota) and weighted shares   │
│ **memory**  │ `memory.max`, `memory.high`     │ Hard/soft allocations, swap control, and OOM scoping   │
│ **io**      │ `io.max`, `io.weight`           │ Read/write IOPS and byte-rate throttling per disk      │
│ **pids**    │ `pids.max`, `pids.current`      │ Strict limits on total processes/threads (anti-forkbomb)│
└─────────────┴─────────────────────────────────┴────────────────────────────────────────────────────────┘
```

#### 1. The CPU Controller (`cpu.*`)
The CPU controller regulates execution time using the Completely Fair Scheduler (CFS) bandwidth engine:

##### Bandwidth Ceiling: `cpu.max`
Defines the maximum computational ceiling. It accepts two integer values:
$$\text{cpu.max} = \text{Quota} \quad \text{Period}$$

* **$\text{Quota}$**: The cumulative microseconds ($\mu\text{s}$) of execution time processes within the group are permitted to run within each time slice. If set to `max`, no ceiling is enforced.
* **$\text{Period}$**: The duration of the scheduling window in microseconds (typically $100\,000\ \mu\text{s} = 100\text{ ms}$).

To cap a group to precisely $2.5$ CPU cores:
$$\text{Quota} = 2.5 \times 100\,000 = 250\,000$$

```bash
# Enforce a 2.5-core maximum ceiling:
echo "250000 100000" > /sys/fs/cgroup/mygroup/cpu.max
```

##### Proportional Sharing: `cpu.weight`
Accepts a value between $1$ and $10\,000$ (default: $100$). When the CPU is oversubscribed (contention occurs), CPU time is distributed strictly proportional to the relative weights of competing active groups:

$$\text{CPU Share} = \frac{\text{weight}_i}{\sum_{j} \text{weight}_j}$$

##### Telemetry: `cpu.stat`
Exposes high-precision tracking:
* `usage_usec`: Cumulative CPU time utilized.
* `nr_periods`: Number of elapsed CFS scheduling periods.
* `nr_throttled`: Count of periods where processes were paused because they exhausted their quota.
* `throttled_usec`: Total time spent in a throttled state.

#### 2. The Memory Controller (`memory.*`)
Unlike v1, which had a single abrupt hard limit, cgroups v2 implements a stepped memory reclamation ladder:

```
[ memory.min ]  ──► Protected base. Never reclaimed under host pressure.
      │
[ memory.low ]  ──► Soft protection. Reclaimed only after unprotected groups.
      │
[ memory.high ] ──► Throttling boundary. Kernel slows tasks & invokes reclaim.
      │
[ memory.max ]  ──► HARD CEILING. Immediate OOM killer invocation if reclaim fails.
```

* **`memory.current`**: Total physical memory (RAM) mapped to the cgroup.
* **`memory.min`**: Hard memory protection. If memory falls below this threshold, the kernel will not reclaim pages from this group under any circumstances.
* **`memory.low`**: Best-effort protection. The kernel avoids reclaiming pages from this group unless all unprotected memory has been exhausted.
* **`memory.high`**: Throttle boundary. If breached, the kernel slows down the process's system calls and synchronously forces dirty page reclamation. The process is **never terminated** at this boundary.
* **`memory.max`**: Hard allocation ceiling. If memory exceeds this limit and anonymous/file-backed pages cannot be reclaimed, the cgroup triggers the kernel's Out-Of-Memory engine.
* **`memory.swap.max`**: Upper limit for anonymous swap usage. Setting this to `0` completely blocks swapping for tasks in this group.
* **`memory.oom.group`**: Accepts `0` or `1`. If set to `1`, an OOM event will terminate **every process** in the cgroup simultaneously, preventing broken, half-dead daemon topologies.

#### 3. The I/O Controller (`io.*`)
The I/O controller manages read and write operations across underlying storage devices. Devices are specified by their Major and Minor device numbers (e.g., `8:0` for `/dev/sda`).

##### Throttling Ceilings: `io.max`
Accepts space-delimited key-value limits:
* `rbps`: Read bytes per second.
* `wbps`: Write bytes per second.
* `riops`: Read I/O operations per second.
* `wiops`: Write I/O operations per second.

```bash
# Limit cgroup I/O on disk 8:0 to 10 MB/s write throughput and 100 read IOPS:
echo "8:0 wbps=10485760 riops=100" > /sys/fs/cgroup/mygroup/io.max
```

#### 4. The PIDs Controller (`pids.*`)
Mitigates fork-bomb attacks and runaway thread creation:
* **`pids.max`**: Hard upper limit on the combined count of processes and threads (`struct task_struct` instances) inside the group.
* **`pids.current`**: Current number of allocated tasks.
* **`pids.events`**: Reports `max <counter>`, incremented every time `clone()` or `fork()` fails with `EAGAIN` due to limit exhaustion.

```bash
# Protect against local fork bombs:
echo 100 > /sys/fs/cgroup/mygroup/pids.max
```

---

### Resource Pressure Stall Information (PSI)

Integrated directly into cgroups v2 is the **Pressure Stall Information (PSI)** engine, which exposes real-time stalls caused by resource starvation before catastrophic failures occur:

```bash
$ cat /sys/fs/cgroup/mygroup/cpu.pressure
some avg10=0.00 avg60=0.00 avg300=0.00 total=0

$ cat /sys/fs/cgroup/mygroup/memory.pressure
some avg10=1.25 avg60=0.45 avg300=0.10 total=4512000
full avg10=0.15 avg60=0.02 avg300=0.00 total=89210
```

* **`some`**: The percentage of wall-clock time during which *at least one task* in the cgroup was stalled waiting for available CPU, memory, or block I/O.
* **`full`**: The percentage of wall-clock time during which *all non-idle tasks* were stalled simultaneously. A non-zero `full` memory or I/O pressure indicates severe thrashing.

---

### Systemd Integration: Managing Slices, Scopes, and Transient Units

Modern distributions manage the cgroups v2 unified tree via `systemd`. Modifying files in `/sys/fs/cgroup` directly can cause configuration conflicts with systemd's state machine. Production workloads should interface through systemd primitives:

```
                          `/sys/fs/cgroup` Root
 ┌──────────────────────────────────┼──────────────────────────────────┐
 │                                  │                                  │
 ▼                                  ▼                                  ▼
`system.slice`                `user.slice`                       `machine.slice`
(Core system services)       (Interactive user sessions)         (Containers & VMs)
 ├── nginx.service            └── user-1000.slice                 └── container-a.scope
 └── postgresql.service            └── session-2.scope
```

#### Creating Transient Isolated Workloads with `systemd-run`
`systemd-run` launches transient services or scopes subject to strict, on-the-fly cgroup parameters:

```bash
# Launch a memory- and CPU-bounded command dynamically:
sudo systemd-run \
  --unit=isolated-worker \
  --slice=workload.slice \
  --property=MemoryMax=512M \
  --property=CPUQuota=80% \
  --property=TasksMax=50 \
  --property=ProtectSystem=strict \
  /usr/bin/python3 -m http.server 8080
```

---

## 3. Command-Line Isolation Primitives: Native Utilities

### `chroot(1)` & `chroot(2)`: Mechanics and Breakout Vulnerabilities

Introduced in Version 7 AT&T UNIX (1979), `chroot(2)` changes the apparent filesystem root directory for the calling process and its descendants. The process's VFS root pointer (`current->fs->root`) is pointed to a new directory path.

#### Why `chroot` is Not a Security Boundary
`chroot` modifies only the VFS path lookup mechanism. It **does not isolate**:
* Process ID lists (`ps` still reveals all host processes).
* Network sockets or interfaces.
* Inter-process communication.
* System time or device nodes.

Furthermore, a process retaining `CAP_SYS_CHROOT` inside a `chroot` environment can easily break out using a classic file-descriptor escape:

```c
/* Classic chroot Breakout Vector (chroot_escape.c) */
#include <sys/stat.h>
#include <unistd.h>
#include <fcntl.h>

int main() {
    // 1. Open an existing file descriptor to the current root directory:
    int fd = open(".", O_RDONLY);

    // 2. Create and enter an arbitrary sub-jail:
    mkdir("subjail", 0755);
    chroot("subjail");

    // 3. Change directory back through the original outer file descriptor:
    fchdir(fd);

    // 4. Climb back up beyond the virtualized root into the real root:
    for (int i = 0; i < 100; i++) {
        chdir("..");
    }

    // 5. Reset root to the real host root:
    chroot(".");
    execl("/bin/sh", "sh", NULL);
    return 0;
}
```

Because `chroot` leaves open file descriptors intact and does not alter mount namespaces, the program steps backward out of the sub-jail.

---

### `pivot_root(2)`: The Container Foundation

Unlike `chroot(2)`, `pivot_root(2)` swaps the entire physical root mount of the process's mount namespace:

```c
int pivot_root(const char *new_root, const char *put_old);
```

* Moves the current root mount directory (`/`) to `put_old`.
* Promotes `new_root` as the new system-wide root mount (`/`) for the calling mount namespace.
* Once the swap occurs, the process unmounts `put_old` via `umount2(put_old, MNT_DETACH)`.

After `pivot_root` and `MNT_DETACH`, the old host root filesystem ceases to exist inside the process's VFS context. File-descriptor climbing and directory traversal attacks cannot escape because the original root inode is no longer mounted in the namespace.

```
Step 1: Mount Target Filesystem            Step 2: Execute `pivot_root`
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│ Host Root (`/`)                 │       │ New Root Mount (`/`)            │
│  └── /mnt/container_fs (Mounted)│       │  └── /old_root (Previous `/`)   │
└─────────────────────────────────┘       └─────────────────────────────────┘
                                                           │
                                                           ▼ Step 3: Detach
                                          ┌─────────────────────────────────┐
                                          │ Isolated Root Mount (`/`)       │
                                          │ (Old host VFS fully unlinked)   │
                                          └─────────────────────────────────┘
```

---

### Isolating Environments with `unshare(1)`

The `unshare(1)` command provides direct userspace access to `unshare(2)`. It decouples specified namespaces and executes a command:

```bash
# Complete single-command isolation sandbox:
sudo unshare \
  --mount \
  --uts \
  --ipc \
  --net \
  --pid \
  --fork \
  --mount-proc \
  /bin/bash
```

#### Detailed Flag Breakdown:
* **`-m, --mount`**: Unshares the VFS mount namespace.
* **`-u, --uts`**: Unshares the UTS namespace (permits changing hostname safely).
* **`-i, --ipc`**: Unshares the System V/POSIX IPC namespace.
* **`-n, --net`**: Unshares the networking subsystem (begins with an empty loopback).
* **`-p, --pid`**: Unshares the PID namespace.
* **`-f, --fork`**: Spawns the specified program as a child process. **Mandatory** when creating a new PID namespace so that the launched child assumes $\text{PID } 1$.
* **`--mount-proc`**: Mounts a fresh, isolated `/proc` filesystem inside the new mount namespace before executing the target binary. This ensures `ps aux` and `top` reflect only the processes running inside the new PID namespace.

---

### Inspecting and Entering Environments with `nsenter(1)`

`nsenter(1)` locates an existing process by PID and attaches the calling shell into its namespaces:

```bash
# Attach into an existing container's PID, Mount, and Network namespaces:
sudo nsenter --target 14201 --mount --uts --ipc --net --pid /bin/bash
```

This utility enables systems administrators to diagnose network issues or inspect locked file descriptors inside containers without having to install diagnostic binaries within the container image itself.

---

## 4. Practical Laboratories and Diagnostic Walkthroughs

### Lab 1: Assembling a Complete Rootless Container Sandbox from Scratch

#### Objective
Build a fully isolated container environment without Docker, runc, or root privileges. The sandbox must use user namespaces to map permissions, an isolated mount namespace, an independent PID namespace with working `/proc`, private UTS, and a secured filesystem root.

#### Execution Script (`build_rootless_sandbox.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOTFS="/tmp/sandbox_rootfs"
echo "[*] STEP 1: Preparing minimal rootfs at ${ROOTFS}..."
rm -rf "${ROOTFS}"
mkdir -p "${ROOTFS}"/{bin,lib,lib64,proc,sys,dev,etc}

# Copy essential utilities and their shared libraries:
copy_deps() {
    local bin="$1"
    cp -v "$bin" "${ROOTFS}/bin/"
    ldd "$bin" | grep -o '/lib[^ ]*' | while read -r lib; do
        if [ -f "$lib" ]; then
            mkdir -p "${ROOTFS}/$(dirname "$lib")"
            cp -n "$lib" "${ROOTFS}/${lib}"
        fi
    done
}

echo "[*] Populating binaries..."
copy_deps /bin/bash
copy_deps /bin/ls
copy_deps /bin/ps
copy_deps /bin/hostname
copy_deps /bin/mkdir

echo "[*] Writing mock /etc/passwd..."
cat << 'EOF' > "${ROOTFS}/etc/passwd"
root:x:0:0:root:/root:/bin/bash
guest:x:1000:1000:guest:/:/bin/bash
EOF

echo "[*] STEP 2: Launching rootless nested execution environment..."
# unshare flags:
# -U: New user namespace
# -m: New mount namespace
# -u: New UTS namespace
# -p: New PID namespace
# -f: Fork process to make child PID 1
# -r: Map current user to root in the new namespace (calls newuidmap/newgidmap)
unshare -U -m -u -p -f -r bash -c "
    echo '[+] Inside the User & Mount Namespace!'
    echo 'Virtual User ID: \$(id)'

    # Set independent hostname
    hostname isolated-sandbox
    echo 'Virtual Hostname: \$(hostname)'

    # Mount private tmpfs for rootfs manipulation
    mount -t tmpfs none ${ROOTFS}/dev
    mknod -m 666 ${ROOTFS}/dev/null c 1 3 || true
    mknod -m 666 ${ROOTFS}/dev/zero c 1 5 || true

    # Prepare pivot_root directories
    mkdir -p ${ROOTFS}/old_root
    cd ${ROOTFS}

    # Bind mount the target rootfs onto itself to convert it into an independent mountpoint
    mount --bind ${ROOTFS} ${ROOTFS}

    # Mount real isolated /proc for the new PID namespace
    mount -t proc proc ${ROOTFS}/proc

    # Change to rootfs and pivot mounts
    cd ${ROOTFS}
    pivot_root . old_root

    # Unmount the host's old root filesystem and clean up
    umount -l /old_root
    rmdir /old_root

    echo '[+] Filesystem and PID hierarchy completely isolated!'
    exec /bin/bash
"
```

#### Verification Inside the Sandbox:
```bash
# Inside the container:
hostname
# Output: isolated-sandbox

id
# Output: uid=0(root) gid=0(root) groups=0(root)

ps aux
# Output:
# USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# root         1  0.0  0.0  14120  4012 ?        S    12:00   0:00 /bin/bash
# root         2  0.0  0.0  12540  2100 ?        R+   12:00   0:00 ps aux

ls -la /
# Real host root is completely invisible; only sandbox files are accessible.
```

---

### Lab 2: Hard Resource Throttling and Stress Testing with cgroups v2

#### Objective
Configure an isolated cgroups v2 slice directly via the sysfs tree. Constrain CPU consumption to $0.5$ cores and memory to $150\text{ MiB}$, verify CFS throttling via `cpu.stat`, and observe memory management behavior under load.

#### Execution Script (`cgroups_v2_throttler.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

CGROUP_ROOT="/sys/fs/cgroup"
BENCH_GROUP="${CGROUP_ROOT}/bench_isolation"

# Verify cgroups v2 existence
if [ ! -f "${CGROUP_ROOT}/cgroup.controllers" ]; then
    echo "[-] System is not running native cgroups v2!" >&2
    exit 1
fi

echo "[*] STEP 1: Enabling subtree controllers in root..."
echo "+cpu +memory +pids" | sudo tee "${CGROUP_ROOT}/cgroup.subtree_control" > /dev/null

echo "[*] STEP 2: Creating cgroup node at ${BENCH_GROUP}..."
sudo mkdir -p "${BENCH_GROUP}"

echo "[*] STEP 3: Applying resource constraints..."
# 1. CPU Quota: 50,000 microseconds out of 100,000 = 0.5 CPU Core
echo "50000 100000" | sudo tee "${BENCH_GROUP}/cpu.max" > /dev/null

# 2. Memory Limits: High throttle at 100M, Hard max at 150M
echo "100M" | sudo tee "${BENCH_GROUP}/memory.high" > /dev/null
echo "150M" | sudo tee "${BENCH_GROUP}/memory.max" > /dev/null
echo "0"    | sudo tee "${BENCH_GROUP}/memory.swap.max" > /dev/null

# 3. Process Maximum: 10 processes/threads
echo "10" | sudo tee "${BENCH_GROUP}/pids.max" > /dev/null

echo "[*] STEP 4: Spawning CPU Burner into cgroup..."
# Spawns an infinite loop trying to consume 100% of available CPU
sudo bash -c "echo \$$ > ${BENCH_GROUP}/cgroup.procs && exec python3 -c '
import time
print(\"[Child] Burning CPU...\")
start = time.time()
while time.time() - start < 5:
    pass
print(\"[Child] Done.\")
'"

echo -e "\n[*] STEP 5: Collecting CPU Throttling Metrics..."
cat "${BENCH_GROUP}/cpu.stat"

echo -e "\n[*] STEP 6: Cleaning up..."
sudo rmdir "${BENCH_GROUP}"
echo "[+] Lab complete. Cgroup node deleted."
```

#### Diagnostic Output Analysis:
```text
[*] Collecting CPU Throttling Metrics...
usage_usec 2510200
user_usec 2490100
system_usec 20100
nr_periods 50
nr_throttled 49
throttled_usec 2489000
```
* **Analysis**: Over $50$ scheduling periods ($5\text{ seconds}$), the process attempted to consume full CPU power. The CFS bandwidth controller throttled execution in $49$ out of the $50$ periods (`nr_throttled`), enforcing the $50\%$ cap and pausing the process for a total of $2.48\text{ seconds}$ (`throttled_usec`).

---

### Lab 3: Dual Network Namespace Interconnect via `veth` Pairs and NAT

#### Objective
Create two independent network namespaces (`netns-client` and `netns-router`), interconnect them using a virtual ethernet pair, assign static IP addresses, enable packet routing, and verify connectivity across the namespace boundary.

```
 netns-client (10.200.1.2/24)               netns-router (10.200.1.1/24)
┌───────────────────────────┐              ┌───────────────────────────┐
│ Virtual Int: `veth-cli`   │ ───────────► │ Virtual Int: `veth-rtr`   │
│ Gateway: 10.200.1.1       │  veth pair   │ Loopback: `lo`            │
└───────────────────────────┘              └───────────────────────────┘
```

#### Execution Script (`network_namespace_lab.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "[*] STEP 1: Creating network namespaces..."
sudo ip netns add netns-client
sudo ip netns add netns-router

echo "[*] STEP 2: Creating virtual ethernet pair..."
sudo ip link add veth-cli type veth peer name veth-rtr

echo "[*] STEP 3: Assigning veth endpoints to respective namespaces..."
sudo ip link set veth-cli netns netns-client
sudo ip link set veth-rtr netns netns-router

echo "[*] STEP 4: Configuring IP addresses and bringing links up..."
# Configure client side
sudo ip netns exec netns-client ip addr add 10.200.1.2/24 dev veth-cli
sudo ip netns exec netns-client ip link set veth-cli up
sudo ip netns exec netns-client ip link set lo up

# Configure router side
sudo ip netns exec netns-router ip addr add 10.200.1.1/24 dev veth-rtr
sudo ip netns exec netns-router ip link set veth-rtr up
sudo ip netns exec netns-router ip link set lo up

echo "[*] STEP 5: Setting default route in client namespace..."
sudo ip netns exec netns-client ip route add default via 10.200.1.1

echo "[*] STEP 6: Testing cross-namespace bidirectional connectivity..."
sudo ip netns exec netns-client ping -c 3 10.200.1.1

echo -e "\n[*] STEP 7: Inspecting isolated socket bindings..."
# Start a background listener inside the router namespace:
sudo ip netns exec netns-router nc -l -p 8080 &
ROUTER_NC_PID=$!
sleep 0.5

echo "[+] Checking host sockets for port 8080 (should be empty):"
ss -tulpn | grep 8080 || echo "Port 8080 is NOT listening in the host namespace!"

echo "[+] Checking router namespace for port 8080 (should be present):"
sudo ip netns exec netns-router ss -tulpn | grep 8080

# Terminate netcat instance
kill $ROUTER_NC_PID 2>/dev/null || true

echo -e "\n[*] STEP 8: Cleaning up network namespaces..."
sudo ip netns del netns-client
sudo ip netns del netns-router
echo "[+] Network interfaces and namespaces cleanly dismantled."
```

---

## 5. Comprehensive Operational Reference Matrices

### Linux Namespaces Reference Matrix

| Namespace | C System Call Flag | Inode Name | Kernel Config Symbol | Key Primary Isolated System Resources |
| :--- | :--- | :--- | :--- | :--- |
| **Mount** | `CLONE_NEWNS` | `mnt` | `CONFIG_BLK_DEV_INITRD` | VFS Mount table, root directory (`/`), propagation states |
| **UTS** | `CLONE_NEWUTS` | `uts` | `CONFIG_UTS_NS` | System hostname, NIS domain name |
| **IPC** | `CLONE_NEWIPC` | `ipc` | `CONFIG_IPC_NS` | System V message queues, semaphores, shared memory, POSIX queues |
| **PID** | `CLONE_NEWPID` | `pid` | `CONFIG_PID_NS` | Process ID hierarchy, namespace init ($\text{PID } 1$), signal scoping |
| **Network** | `CLONE_NEWNET` | `net` | `CONFIG_NET_NS` | Network interfaces (`eth*`, `veth*`), IP routing, iptables, ports |
| **User** | `CLONE_NEWUSER` | `user` | `CONFIG_USER_NS` | UID/GID mappings, POSIX capability sets, security tokens |
| **Cgroup** | `CLONE_NEWCGROUP`| `cgroup`| `CONFIG_CGROUPS` | Virtualized view of `/proc/self/cgroup`, mitigates host leakage |
| **Time** | `CLONE_NEWTIME` | `time` | `CONFIG_TIME_NS` | Monotonic and boot clocks (virtualized per-container clock drift)|

---

### cgroups v2 Core Controller Interfaces Matrix

| Controller | Virtual File Path | I/O Mode | Accepted Input Syntax | Primary Operational Effect |
| :--- | :--- | :--- | :--- | :--- |
| **Core** | `cgroup.procs` | R/W | `<PID>` | Atomically migrates all threads of a process into the cgroup |
| **Core** | `cgroup.subtree_control`| R/W | `+<ctrl> -<ctrl>` | Enables or disables controllers for all immediate child nodes |
| **CPU** | `cpu.max` | R/W | `<quota> <period>` | CFS bandwidth ceiling (e.g., `100000 100000` = 1 physical core) |
| **CPU** | `cpu.weight` | R/W | `1` to `10000` | Proportional CPU shares under active contention (Default: 100) |
| **CPU** | `cpu.stat` | R/O | Key-Value Metrics | High-precision tracking of CPU usage, elapsed periods, and throttle events |
| **Memory** | `memory.current` | R/O | Integer (Bytes) | Exact physical RAM consumption of all processes in the cgroup |
| **Memory** | `memory.min` | R/W | Integer (Bytes) / `max`| Hard memory protection floor; completely exempt from host reclaim |
| **Memory** | `memory.low` | R/W | Integer (Bytes) / `max`| Soft memory protection; reclaimed only under severe host pressure |
| **Memory** | `memory.high` | R/W | Integer (Bytes) / `max`| Throttle limit; forces direct synchronous reclaim, avoiding OOM kill |
| **Memory** | `memory.max` | R/W | Integer (Bytes) / `max`| Hard upper ceiling; triggers OOM killer if pages cannot be freed |
| **Memory** | `memory.oom.group` | R/W | `0` or `1` | If `1`, triggers SIGKILL across **all** tasks in group during OOM |
| **I/O** | `io.max` | R/W | `<maj>:<min> <type>=<N>`| Limits rate of disk reads/writes (`rbps`, `wbps`, `riops`, `wiops`) |
| **I/O** | `io.stat` | R/O | Key-Value Metrics | Real-time byte counters and read/write operations per disk partition |
| **PIDs** | `pids.max` | R/W | Integer / `max` | Strict ceiling on maximum permitted processes and threads |
| **PIDs** | `pids.current` | R/O | Integer | Number of active `task_struct` instances running in the cgroup |

---

### Command-Line Native Isolation Primitives Matrix

| Utility | Underlying System Calls | Scope of Isolation | Security Boundary Strength | Primary Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **`chroot`** | `chroot(2)` | VFS Path Resolution | **Extremely Weak** (Escape via open FD or `fchdir` possible) | Legacy package builds, recovery chroots via LiveCD |
| **`pivot_root`**| `pivot_root(2)` | VFS Root Mount Table | **Very Strong** (Old root unmounted and completely unlinked) | True container runtime startup (`runc`, `crun`, `initramfs`) |
| **`unshare`** | `unshare(2)`, `clone(2)`| Arbitrary Namespaces | **Strong** (Full kernel table virtualization per flags) | One-off CLI sandboxes, rootless container initialization |
| **`nsenter`** | `setns(2)`, `open(2)` | Joins Existing Namespaces| Identical to target task context | Live container diagnostics, debugging crashed container nets |
| **`systemd-run`**| D-Bus `systemd1` calls | Cgroups v2 + Namespaces | **High** (Enforced by host PID 1 with recovery supervision) | Production service isolation, dynamic batch task sandboxing |