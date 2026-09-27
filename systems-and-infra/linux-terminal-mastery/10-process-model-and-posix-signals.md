# Process Model, POSIX Signals, and Execution Control

In Unix-like operating systems, the **process** is the fundamental abstraction for resource allocation, memory isolation, and execution scheduling. The Linux kernel mediates CPU time, address spaces, credentials, and communication channels by maintaining rigorous hierarchical boundaries between processes. Mastering Linux systems administration and engineering requires a granular understanding of how processes are synthesized, executed, suspended, observed, and destroyed.

This module deconstructs the POSIX process model, kernel-space execution primitives, signal dispatch mechanics, interactive terminal job control, and system introspection utilities.

---

## 1. Process Creation and Lifecycle Architecture

### The Kernel Task Abstraction (`struct task_struct`)
In the Linux kernel, every executing entity—whether a single-threaded program or an individual thread inside a multi-threaded application—is represented by an instance of `struct task_struct` (defined in `<linux/sched.h>`). The scheduler treats these tasks uniformly as schedulable entities (*tasks*), differentiated primarily by the degree to which they share virtual address spaces, file descriptor tables, and signal handlers.

```
                            Linux Kernel Memory
 ┌────────────────────────────────────────────────────────────────────────┐
 │                      struct task_struct (PCB)                          │
 │  ├── pid_t pid;                   // Thread ID (TID in userspace)      │
 │  ├── pid_t tgid;                  // Thread Group ID (PID in userspace)│
 │  ├── volatile long state;         // Execution state (R, S, D, T, Z)   │
 │  ├── struct task_struct *parent;  // Pointer to parent task            │
 │  ├── struct list_head children;   // Sub-tree of child tasks           │
 │  ├── struct mm_struct *mm;        // Virtual memory descriptor         │
 │  ├── struct files_struct *files;  // Open file descriptor table        │
 │  ├── struct signal_struct *signal;// Shared signal handlers & pending  │
 │  └── struct sighand_struct *sighand;                                   │
 └────────────────────────────────────────────────────────────────────────┘
```

*   **Process ID ($PID$) vs. Thread Group ID ($TGID$)**:
    *   In kernel space, every scheduled task has a unique `pid`.
    *   In userspace (via POSIX `glibc`), the integer returned by `getpid(2)` is actually the task's `tgid`.
    *   For a single-threaded process: $\text{PID} = \text{TGID}$.
    *   For multi-threaded processes created via `pthread_create(3)`: Each thread has its own kernel `pid` (visible as a Thread ID, or $TID$, via `gettid(2)`), but all threads share the identical `tgid` (reported as the $PID$ to userspace).
*   **Virtual Memory Pointer (`mm_struct`)**: References the process's page directory, memory mappings, code/data/stack segments, and total memory allocations. For kernel threads (e.g., `kcompactd0`, `kworker`), this pointer is `NULL` because they execute entirely within kernel address space.

---

### Process Creation Primitives: `fork(2)`, `vfork(2)`, and `clone(2)`

The instantiation of a new userspace process never occurs spontaneously; it relies on kernel cloning mechanisms followed by optional memory image replacement.

```
[ Parent Process: PID 1040 ]
             │
             │ 1. fork() / clone()
             ▼
┌───────────────────────────────────────────────┐
│ Child Process: PID 1041                       │
│ - Exact copy of Parent's task_struct          │
│ - Page tables cloned with Read-Only + CoW bit │
│ - Inherits open file descriptors, UID/GID     │
└──────────────────────┬────────────────────────┘
                       │
                       │ 2. execve("/usr/bin/python3", argv, envp)
                       ▼
┌───────────────────────────────────────────────┐
│ Child Process: PID 1041 (Active Program)      │
│ - Previous address space wiped                │
│ - Executable text, heap, stack re-mapped      │
│ - Retains PID 1041, PPID 1040, open FDs*      │
│   (*unless FD_CLOEXEC is configured)          │
└───────────────────────────────────────────────┘
```

#### 1. The `fork(2)` System Call
When a program calls `fork()`, the kernel creates a child process that is an exact duplicate of the calling parent process:
*   The child receives a unique Process ID ($PID$) and records the caller's $PID$ as its Parent Process ID ($PPID$).
*   The system call returns **twice**: it returns the child's $PID$ to the parent process, and it returns $0$ to the child process. If the operation fails, it returns $-1$ in the parent.

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        exit(EXIT_FAILURE);
    } else if (pid == 0) {
        // Child context
        printf("Child: PID=%d, Parent PPID=%d\n", getpid(), getppid());
        _exit(EXIT_SUCCESS);
    } else {
        // Parent context
        printf("Parent: PID=%d, Created Child PID=%d\n", getpid(), pid);
    }
    return 0;
}
```

#### 2. Copy-on-Write (CoW) Mechanics
Historically, early Unix systems duplicated the entire physical memory space of the parent into a newly allocated space for the child during `fork()`, an operation of $\mathcal{O}(M)$ complexity (where $M$ is the memory footprint). 

Modern Linux avoids this overhead using **Copy-on-Write (CoW)**:
1.  **Page Table Duplication**: During `fork()`, the kernel duplicates only the **page table architecture** (the mapping of virtual addresses to physical pages), not the underlying physical pages.
2.  **Read-Only Flag Enforcement**: All writable pages in both the parent and child memory spaces are marked as **read-only** ($RO$), and their reference count in the page frame structure (`struct page`) is incremented.
3.  **Page Fault Interception**: If either process attempts to perform a write operation to a CoW page:
    *   The CPU memory management unit (MMU) traps the unauthorized write and triggers a hardware **page fault** (`do_wp_page` in the kernel).
    *   The kernel inspects the faulting address and verifies the page belongs to a valid writable mapping marked for CoW.
    *   The kernel allocates a **new physical page frame**, copies the $4\text{ KiB}$ contents of the original page into this new frame, updates the writing process's page table entry ($PTE$) to point to the new frame, marks the page as **read-write** ($RW$), and decrements the reference count of the original page.
    *   The CPU resumes execution transparently.

Because most `fork()` calls in Unix systems are followed immediately by an `execve()` call, Copy-on-Write prevents gigabytes of physical memory from being uselessly duplicated only to be instantly discarded.

#### 3. The `clone(2)` Foundation
In modern Linux, `fork()` and `vfork()` are trivial wrappers around the consolidated `clone(2)` system call:
```c
int clone(int (*fn)(void *), void *stack, int flags, void *arg, ... 
          /* pid_t *parent_tid, void *tls, pid_t *child_tid */ );
```
The behavior of `clone()` is dictated by granular bitwise flags passed in `flags`:
*   `CLONE_VM`: Parent and child share the same virtual memory space (threads).
*   `CLONE_FS`: Parent and child share filesystem context (root, current working directory, umask).
*   `CLONE_FILES`: Parent and child share the identical open file descriptor table.
*   `CLONE_SIGHAND`: Parent and child share signal disposition handlers.
*   `CLONE_NEWPID` / `CLONE_NEWNET` / `CLONE_NEWNS`: Instantiates the child into new Linux namespaces (the foundation of container virtualization).

---

### Program Execution: The `execve(2)` System Call

The `execve(2)` system call replaces the calling process's entire memory space, execution context, and thread execution with a new program image loaded from an executable binary or interpreter script.

```c
int execve(const char *pathname, char *const argv[], char *const envp[]);
```

#### The Kernel Execution Sequence
When `execve()` is invoked, the kernel executes the following sequence:
1.  **Path Lookup and Access Validation**: Resolves the path within the VFS, verifying read and execute permissions (`+x`) against the caller's credentials.
2.  **Binary Format Matching (`search_binary_handler`)**: The kernel scans registered binary format handlers (`struct linux_binfmt`).
    *   **ELF (`binfmt_elf.c`)**: If the file begins with the 4-byte magic sequence `\x7fELF`, the kernel validates ELF headers, reads segment tables, and constructs virtual memory mappings (`PT_LOAD` segments).
    *   **Script (`binfmt_script.c`)**: If the file starts with `#!` (shebang), the kernel parses the interpreter path and options (e.g., `#!/usr/bin/env python3`), shifts original arguments, and recursively executes the target interpreter binary.
3.  **Memory Space Deallocation**: Releases the old virtual memory space (`mm_struct`). All historical stack, heap, data, and code pages are unmapped.
4.  **Memory Layout Re-mapping**:
    *   Maps the executable's `.text` (machine instructions) as read-only and executable (`R-X`).
    *   Maps the `.data` (initialized global variables) and `.bss` (uninitialized variables) as read-write (`RW-`).
    *   Sets up the runtime **heap** (managed via `brk(2)` and `mmap(2)`).
    *   Allocates the userspace **stack**, populating it with the arrays passed via `argv` (argument strings), `envp` (environment variables), and the **Auxiliary Vector** (ELF auxiliary variables providing system parameters such as CPU page sizes and hardware capabilities to the dynamic linker).
5.  **Dynamic Linker Execution**: If the binary is dynamically linked, the kernel reads the `.interp` section of the ELF file (typically pointing to `/lib64/ld-linux-x86-64.so.2`), maps the dynamic linker into userspace memory, and sets the Instruction Pointer ($IP$/$PC$) to the dynamic linker's entry point. The linker resolves shared libraries (`.so`) before jumping to the program's `main()` entry point.
6.  **Descriptor Retention & `FD_CLOEXEC`**: Open file descriptors remain open across `execve()` unless explicitly marked with the `FD_CLOEXEC` (Close-on-Exec) flag via `fcntl(2)` or `open(2)` (`O_CLOEXEC`).

---

### The Process Lifecycle State Machine

A Linux process transitions through discrete execution states managed by the kernel scheduler:

```
                          ┌────────────────────────┐
                          │   Created via fork()   │
                          └───────────┬────────────┘
                                      │
                                      ▼
                        ┌───────────────────────────┐
                        │    TASK_RUNNING (R)       │ ◄──────────┐
                        │   (Executing or in Run-   │            │
                        │    queue awaiting CPU)    │            │
                        └─────┬───────────────▲─────┘            │
                              │               │                  │
            Scheduler blocks  │               │ Event Occurs /   │ Wakeup /
            for I/O or Lock   │               │ Signal Received  │ SIGCONT
                              ▼               │                  │
      ┌─────────────────────────────────┐     │                  │
      │      TASK_INTERRUPTIBLE (S)     │─────┤                  │
      │   (Sleeping; wakes on event or  │     │                  │
      │    incoming signals)            │     │                  │
      └─────────────────────────────────┘     │                  │
                                              │                  │
      ┌─────────────────────────────────┐     │                  │
      │     TASK_UNINTERRUPTIBLE (D)    │─────┘                  │
      │   (Deep sleep; blocks signals;  │                        │
      │    waiting on disk/driver I/O)  │                        │
      └─────────────────────────────────┘                        │
                              │                                  │
                              │ Signal: SIGSTOP / SIGTSTP        │
                              ▼                                  │
                        ┌───────────────────────────┐            │
                        │     TASK_STOPPED (T)      │────────────┘
                        │    (Execution suspended)  │
                        └─────────────┬─────────────┘
                                      │
                                      │ Terminated via exit()
                                      ▼
                        ┌───────────────────────────┐
                        │      EXIT_ZOMBIE (Z)      │
                        │ (Dead; resources freed;   │
                        │  retained in task list)   │
                        └─────────────┬─────────────┘
                                      │
                                      │ Parent calls waitpid()
                                      ▼
                        ┌───────────────────────────┐
                        │      EXIT_DEAD (X)        │
                        │ (task_struct freed from   │
                        │  kernel memory)           │
                        └───────────────────────────┘
```

#### Detailed State Classifications
*   **`TASK_RUNNING` (`R`)**: The process is actively executing on a CPU core or sits in the run queue ready to be scheduled by the Completely Fair Scheduler (CFS) or Real-Time scheduler.
*   **`TASK_INTERRUPTIBLE` (`S`)**: The process is sleeping, waiting for an external condition (such as hardware input, network data availability, a timer event, or an unlocked mutex). It will wake up immediately if it receives a signal.
*   **`TASK_UNINTERRUPTIBLE` (`D`)**: The process is sleeping, typically waiting for direct hardware I/O operations (such as physical block storage transfers or synchronous NFS system calls). 
    $$\mathbf{Critical\ Property:}\quad \text{The kernel does not deliver signals to a task in } D \text{ state.}$$
    A process in state `D` cannot be terminated, even with `SIGKILL` (`kill -9`). If a storage system hangs indefinitely, the process remains frozen in `D` state until the I/O request times out or completes, contributing permanently to the system **load average**.
*   **`TASK_STOPPED` (`T`)**: Execution has been explicitly halted via job control signals (`SIGSTOP`, `SIGTSTP`, `SIGTTIN`, `SIGTTOU`). The process retains its memory footprint but is barred from scheduling until it receives `SIGCONT`.
*   **`TASK_TRACED` (`t`)**: The task's execution has been paused by a debugging tracer (such as `gdb` or `strace`) using the `ptrace(2)` system call.
*   **`EXIT_ZOMBIE` (`Z`)**: The process has finished executing via `exit(2)` and has released its memory pages and file descriptors. It persists solely as a record in the kernel task table to hold its 16-bit exit status until the parent retrieves it.
*   **`EXIT_DEAD` (`X`)**: The terminal state when the parent retrieves the exit status via `wait(2)`. The kernel unlinks and frees the `struct task_struct` from memory.

---

### Process Hierarchy, Sub-Reapers, and PID 1

Every process within a Linux system traces its lineage back to a single ancestor: **Process ID 1** (`init` or `systemd`).

```
                              systemd (PID 1)
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
      sshd (PID 840)          dbus-daemon (PID 910)     crond (PID 950)
           │
     sshd [priv] (PID 2100)
           │
      sshd: user (PID 2105)
           │
      bash (PID 2106)
           │
    python3 (PID 2410)
```

#### The Special Status of PID 1
`PID 1` has unique responsibilities within the POSIX execution model:
1.  **Immunity to Default Signals**: `PID 1` is protected from unhandled signals. If an unprivileged or superuser process issues `kill -9 1` or `kill -15 1`, the kernel checks the target $PID$. If target $PID = 1$, the signal is discarded unless explicit handlers are implemented inside the `init` binary. The kernel forbids userspace from killing `PID 1` to prevent immediate kernel panics.
2.  **Orphan Adoption**: When any intermediate parent process terminates without waiting for its children, those child processes are orphaned. Traditionally, the kernel automatically reparents all orphaned processes directly to `PID 1`.
3.  **Continuous Reaping Duty**: `PID 1` must run an event loop that continuously calls `wait(2)` or `waitpid(2)` to reap terminated orphaned children. If `PID 1` fails to reap orphans, terminated processes accumulate as permanent zombies.

#### The Sub-Reaper Mechanism (`PR_SET_CHILD_SUBREAPER`)
In modern container runtimes (such as Docker, containerd, and systemd user instances), relying exclusively on `PID 1` to adopt orphaned grandchildren breaks service tracking. If a double-forking service orphans a process within a container or systemd slice, `PID 1` would adopt it, detaching it from its controlling service supervisor.

To resolve this, Linux (since kernel 3.4) provides the `prctl(2)` command `PR_SET_CHILD_SUBREAPER`:
```c
#include <sys/prctl.h>

// Mark the calling process as a sub-reaper:
prctl(PR_SET_CHILD_SUBREAPER, 1, 0, 0, 0);
```
When a process marked as a sub-reaper spawns children that subsequently fork and orphan their offspring, the kernel **walks up the process hierarchy tree** and reparents the orphan to the nearest ancestor marked as a sub-reaper, rather than defaulting all the way up to `PID 1`.

---

## 2. Zombie and Orphan Processes: Mechanics and Remediation

### The Process Termination Protocol

When a process completes its execution (via standard return from `main()`, explicit `exit(3)`, or termination via signals), the kernel tears down its resources through `do_exit()`:
1.  All open file descriptors are closed (triggering reference count decrements on the underlying open file descriptions).
2.  The virtual address space (`mm_struct`) is released; allocated memory pages are freed back to the buddy allocator.
3.  The task's state changes to `EXIT_ZOMBIE`.
4.  The kernel delivers a `SIGCHLD` signal to the parent process, alerting it that a state change has occurred.
5.  The dead process remains in the kernel process table holding its $PID$, termination timestamp, resource usage metrics, and exit status code.

```
 Parent Process                       Child Process
       │                                     │
       │                                     │ Calls exit(0)
       │                                     ▼
       │                              Kernel Tears Down:
       │                              - Frees Address Space
       │                              - Closes File Descriptors
       │                              - State: EXIT_ZOMBIE
       │ ◄──── Sends SIGCHLD ───────────────┤
       │                                     │
       ├─ Calls waitpid() ─────────────────► │
       │                                     ▼
       │                              Kernel Releases:
       │                              - struct task_struct
       │                              - PID returned to pool
       ▼                                     X (EXIT_DEAD)
```

The parent must harvest this termination metadata using one of the wait system calls:
```c
#include <sys/wait.h>

pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
```

---

### Zombie Processes (`<defunct>`)

#### Definition and Root Cause
A **zombie process** is a process that has completed execution, but whose parent process has not yet read its exit status via `waitpid()`. In process listings (`ps`, `top`), a zombie process is identified by the status code `Z` and the label `<defunct>`.

```c
/* Reproducing a Zombie Process in C */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();

    if (pid == 0) {
        // Child terminates immediately:
        printf("[Child PID %d] Exiting now...\n", getpid());
        exit(42);
    }

    // Parent sleeps without calling wait(), leaving child as a zombie:
    printf("[Parent PID %d] Sleeping for 60 seconds without wait()...\n", getpid());
    sleep(60);
    return 0;
}
```

#### The Architectural Risk: PID Table Exhaustion
A zombie process consumes **zero CPU** and **zero bytes of RAM** for its execution segments (stack, heap, and code have been deallocated). 

However, each zombie retains:
1.  An allocated `struct task_struct` inside the kernel SLAB/SLUB cache.
2.  An entry in the kernel's process table, holding a unique $PID$.

The kernel limits the total number of allocatable process IDs via the sysctl parameter:
```bash
$ cat /proc/sys/kernel/pid_max
4194304
```
On embedded systems or default enterprise deployments, `pid_max` may be set to $32\,768$. If an application leaks threads or child processes without reaping them, it will exhaust the available PID pool. Once exhausted, the kernel returns `EAGAIN` (`Resource temporarily unavailable`) on subsequent `fork()` and `clone()` calls, preventing new commands, cron jobs, or SSH logins from running across the entire system.

#### Diagnosing and Resolving Zombies
Because a zombie process is already dead, it **cannot be terminated using signals**:
```bash
$ kill -9 <ZOMBIE_PID>   # Has NO EFFECT. A dead process cannot handle SIGKILL!
```

To eliminate a zombie process, you must force the exit status to be harvested:

##### Method 1: Signal the Parent to Reap
Notify the negligent parent process via `SIGCHLD` to trigger its internal signal handler and harvest the zombie:
```bash
kill -CHLD <PARENT_PID>
```

##### Method 2: Eliminate the Negligent Parent
If the parent application is stuck, misconfigured, or has ignored `SIGCHLD`, terminate the parent process:
```bash
kill -15 <PARENT_PID>
# If unresponsive:
kill -9 <PARENT_PID>
```
When the parent terminates, the Linux kernel identifies all of its running and zombie children as orphans. The kernel reparents these orphans to their nearest sub-reaper or `PID 1` (`systemd`), which reaps the zombie processes immediately.

---

### Orphan Processes

#### Definition and Behavior
An **orphan process** is an actively executing process whose original parent has terminated before the child finished its execution.

```c
/* Reproducing an Orphan Process in C */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();

    if (pid > 0) {
        // Parent exits immediately:
        printf("[Parent PID %d] Terminating. Leaving child orphaned.\n", getpid());
        exit(EXIT_SUCCESS);
    }

    // Child continues execution:
    while (1) {
        printf("[Child PID %d] Current PPID=%d\n", getpid(), getppid());
        sleep(2);
    }
    return 0;
}
```

```
Parent (PID 5000) forks Child (PID 5001)
        │
        ├─ Parent (PID 5000) exits immediately
        ▼
Child (PID 5001) becomes an ORPHAN
        │
        ▼ (Kernel Reparenting Event)
Nearest Sub-Reaper or systemd (PID 1) adopts Child
        │
Child (PID 5001): PPID changes from 5000 to 1
```

Orphaning is not necessarily an error condition. In traditional Unix architectures, intentional double-forking was the standard method for converting a command into a background **daemon**:
1.  Process $A$ forks Process $B$.
2.  Process $B$ calls `setsid(2)` to create a new session and detach from the controlling terminal.
3.  Process $B$ forks Process $C$ and terminates immediately.
4.  Process $C$ (the actual daemon) is adopted by `PID 1`. It cannot reacquire a controlling terminal because it is not a session leader.

---

## 3. Kernel Signal Architecture and Dispatch Mechanics

### What is a POSIX Signal?

A **signal** is an asynchronous software interrupt generated by the kernel, a hardware fault, or an authorized userspace process (`kill(2)`). Signals interrupt a process's normal execution flow to communicate operating system events, hardware exceptions, or lifecycle commands.

```
       Signal Generation Sources:
       
       1. Hardware Exceptions
          - CPU trap: Divide by 0 ───────► SIGFPE  (8)
          - MMU trap: Bad memory read ───► SIGSEGV (11)
          - Bus fault: Unaligned read ───► SIGBUS  (7)
       
       2. Kernel Subsystems
          - Pipe closed / consumer gone ─► SIGPIPE (13)
          - Child status changed ────────► SIGCHLD (17)
          - Software timer expired ──────► SIGALRM (14)
       
       3. Terminal Driver (TTY line discipline)
          - User presses Ctrl+C ─────────► SIGINT  (2)
          - User presses Ctrl+\ ─────────► SIGQUIT (3)
          - User presses Ctrl+Z ─────────► SIGTSTP (20)
       
       4. Userspace Process System Calls
          - `kill(pid, sig)` ────────────► Dispatched to target task
```

---

### Internal Kernel Signal Delivery Lifecycle

The delivery of a signal does not immediately alter user CPU registers; it operates as a two-stage transaction: **Generation** and **Delivery**.

```
  1. Generation Phase:
     Kernel sets bit in task_struct:
     - task->pending.signal (Thread-private) OR
     - task->signal->shared_pending.signal (Thread-group shared)
                      │
                      ▼
  2. Latency Gap (Pending State):
     Signal remains pending until task is scheduled to run.
     Task checks pending signals on transition from 
     Kernel Space ──► Userspace.
                      │
                      ├─ Is Signal Blocked by task->blocked mask?
                      │  ├── YES: Remains pending; execution resumes.
                      │  └── NO:  Kernel evaluates Disposition.
                      │
                      ▼
  3. Delivery Phase (Disposition Actions):
     ├── A. Terminate / Core Dump ──► Task invokes do_exit()
     ├── B. Ignore (SIG_IGN)     ──► Bit cleared; execution resumes
     ├── C. Stop / Continue      ──► Task changes state (T <-> R)
     └── D. Custom User Handler:
            - Kernel constructs user stack frame (sigframe)
            - Sets EIP/RIP register to handler function address
            - Context switches to userspace handler
            - Handler calls sigreturn(2) ──► Restores original context
```

1.  **Pending State**: When generated, the kernel sets the corresponding bit in the task's pending bitmask (`sigset_t`). The signal is marked as **pending**.
2.  **Signal Masking (Blocked Signals)**: Each process has a signal mask (`sigset_t blocked`) configured via `sigprocmask(2)` or `pthread_sigmask(3)`. If a signal is blocked, its bit remains asserted in the pending set, but delivery is deferred until the process unblocks that signal.
3.  **The Delivery Interception**: When the CPU finishes an interrupt or system call and is about to transition execution **from kernel mode back to userspace**, it checks the current task's pending and unblocked signal masks.
4.  **Executing Custom Handlers (`sigaction`)**: If the process has registered a custom signal handler:
    *   The kernel constructs a signal frame on the process's **userspace stack**.
    *   It updates the instruction pointer to point to the address of the custom signal handler.
    *   Execution enters userspace to run the handler.
    *   Upon completion, the handler executes a trampoline sequence that invokes the `sigreturn(2)` system call, instructing the kernel to tear down the signal frame and restore the original CPU register states (instruction pointer, stack pointer, arithmetic flags).

---

### Comprehensive POSIX Standard Signal Matrix

The Linux kernel implements standard POSIX signals ($1\text{ to }31$), followed by real-time signals ($32\text{ to }64$).

| Value | Name | Default Action | Core Dump? | Catchable / Ignorable? | Functional Origin & Semantic Usage |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | `SIGHUP` | Terminate | No | **Yes** | Hangup detected on controlling terminal or death of controlling process. Modern daemons repurpose `SIGHUP` as a directive to reload configuration files dynamically without process downtime. |
| **2** | `SIGINT` | Terminate | No | **Yes** | Interrupt from keyboard. Emitted by the terminal line discipline when the user presses `Ctrl+C`. Designed to cancel the current foreground execution cleanly. |
| **3** | `SIGQUIT` | Terminate | **Yes** | **Yes** | Quit from keyboard. Emitted when the user presses `Ctrl+\`. Halts process execution and directs the kernel to write a process memory dump (`core`) to disk for post-mortem debugging. |
| **4** | `SIGILL` | Terminate | **Yes** | **Yes** | Illegal Instruction. Emitted by the CPU when an attempt is made to execute an invalid, corrupt, or privileged machine code instruction. |
| **5** | `SIGTRAP` | Terminate | **Yes** | **Yes** | Trace/breakpoint trap. Used internally by debuggers (`gdb`, `strace`) to pause execution at breakpoints. |
| **6** | `SIGABRT` | Terminate | **Yes** | **Yes** | Abort signal. Raised programmatically by the C library function `abort(3)` when assertions fail or dynamic heap memory corruption is detected. |
| **7** | `SIGBUS` | Terminate | **Yes** | **Yes** | Bus error. Emitted when accessing an invalid physical address (e.g., misaligned memory access or referencing an `mmap()` file extent that extends beyond physical file storage). |
| **8** | `SIGFPE` | Terminate | **Yes** | **Yes** | Floating-point exception. Triggered by arithmetic hardware faults (e.g., division by zero, floating-point overflow). |
| **9** | `SIGKILL` | Terminate | No | **NO** | Kill signal. Immediate, unconditional termination. Handled directly by the kernel; it cannot be caught, blocked, or ignored by userspace code. |
| **10** | `SIGUSR1` | Terminate | No | **Yes** | User-defined signal 1. Allocated for custom application signaling protocols (e.g., Nginx log rotation). |
| **11** | `SIGSEGV` | Terminate | **Yes** | **Yes** | Segmentation Violation. Emitted by the MMU on invalid memory references (e.g., dereferencing a null/dangling pointer, or writing to read-only memory pages). |
| **12** | `SIGUSR2` | Terminate | No | **Yes** | User-defined signal 2. Allocated for custom application signaling protocols (e.g., zero-downtime binary upgrades in Nginx). |
| **13** | `SIGPIPE` | Terminate | No | **Yes** | Broken pipe. Emitted when a process writes to a pipe or socket whose downstream reader has closed its file descriptor. Default behavior terminates the writer. |
| **14** | `SIGALRM` | Terminate | No | **Yes** | Timer signal. Emitted by the kernel when real-time software timers scheduled by `alarm(2)` or `setitimer(2)` expire. |
| **15** | `SIGTERM` | Terminate | No | **Yes** | Termination signal. The standard administrative request for graceful process termination. Allows the target to flush buffers, release file locks, close database connections, and remove PID/socket files. |
| **16** | `SIGSTKFLT`| Terminate | No | **Yes** | Stack fault on coprocessor (legacy; rarely used by modern Linux kernels). |
| **17** | `SIGCHLD` | **Ignore** | No | **Yes** | Child stopped or terminated. Delivered to the parent process when a child terminates, stops, or resumes. |
| **18** | `SIGCONT` | **Continue** | No | **Yes** | Continue execution. Resumes a process that was previously suspended by `SIGSTOP` or `SIGTSTP`. |
| **19** | `SIGSTOP` | **Stop** | No | **NO** | Stop process execution. The kernel immediately removes the process from the CFS scheduler queue and transitions it to `TASK_STOPPED`. Cannot be caught, blocked, or ignored. |
| **20** | `SIGTSTP` | **Stop** | No | **Yes** | Terminal stop signal. Emitted by the terminal driver when the user presses `Ctrl+Z`. Unlike `SIGSTOP`, `SIGTSTP` can be caught, handled, or ignored by userspace programs. |
| **21** | `SIGTTIN` | **Stop** | No | **Yes** | Background process read attempt. Emitted when a process in a background process group attempts to read from its controlling terminal (`/dev/tty`). |
| **22** | `SIGTTOU` | **Stop** | No | **Yes** | Background process write attempt. Emitted when a process in a background process group attempts to write to the terminal while the terminal's `TOSTOP` flag is active. |
| **23** | `SIGURG` | **Ignore** | No | **Yes** | Urgent condition on socket (e.g., Out-of-Band data arriving over TCP). |
| **24** | `SIGXCPU` | Terminate | **Yes** | **Yes** | CPU time limit exceeded. Emitted when a process crosses its soft resource limit configured via `setrlimit(RLIMIT_CPU)`. |
| **25** | `SIGXFSZ` | Terminate | **Yes** | **Yes** | File size limit exceeded. Emitted if an application attempts to write a file larger than `RLIMIT_FSIZE`. |
| **26** | `SIGVTALRM`| Terminate | No | **Yes** | Virtual timer expired. Emitted when CPU time consumed by the process itself expires. |
| **27** | `SIGPROF` | Terminate | No | **Yes** | Profiling timer expired. Tracks CPU time spent by both the process and the kernel on its behalf. |
| **28** | `SIGWINCH` | **Ignore** | No | **Yes** | Window size change. Emitted to foreground applications when terminal geometry (rows, columns) changes. Used by tools like `vim`, `tmux`, and `htop` to redraw. |
| **29** | `SIGIO` | Terminate | No | **Yes** | Asynchronous I/O event available on a file descriptor configured with `O_ASYNC`. |
| **30** | `SIGPWR` | Terminate | No | **Yes** | Power failure warning. Typically sent by UPS monitoring daemons (e.g., `nut`, `apcupsd`) to trigger a controlled system shutdown. |
| **31** | `SIGSYS` | Terminate | **Yes** | **Yes** | Bad system call. Emitted when a process attempts to invoke an unauthorized system call blocked by a Secure Computing (**seccomp**) BPF filter. |

---

### Deep Dive: Core Operational Signals

#### 1. `SIGTERM` (15) vs. `SIGKILL` (9)
The architectural division between `SIGTERM` and `SIGKILL` forms the foundation of Unix process administration:

```
                      Administrative Stop Request
                                  │
                                  ▼
                        Sends SIGTERM (Signal 15)
                                  │
                  ┌───────────────┴───────────────┐
                  ▼                               ▼
       Application Caught?              Application Ignores/Hangs?
                  │                               │
        Executes Cleanup:                         │ Timeout Expires
        - Flush data to disk                      │ (e.g., systemd 90s)
        - Close sockets & locks                   ▼
        - Remove state files             Sends SIGKILL (Signal 9)
                  │                               │
                  ▼                               ▼
        Clean exit(0)                      Kernel Forces Exit:
        Zero data loss                     - Drops memory pages
                                           - Closes FDs abruptly
                                           - Risk of dirty data loss
```

*   **`SIGTERM` (Signal 15)**: The default signal sent by administrative commands (`kill <pid>`, `systemctl stop`). It represents a polite, cooperative shutdown request. Because it is catchable, applications can bind custom termination routines to flush database transactions to disk, safely complete inflight HTTP requests, and remove `/run/*.pid` locks.
*   **`SIGKILL` (Signal 9)**: The non-cooperative administrative termination signal. It bypasses userspace execution entirely:
    *   The kernel intercepts the signal.
    *   It removes the task from the CFS scheduling queue.
    *   It deallocates the task's address space and closes its open file descriptors.
    *   The application code is **never scheduled again**; no custom cleanup logic or destructor runs. This introduces risks of half-written transaction records, corrupted local database files, or orphaned network locks. 
    $$\mathbf{Rule\ of\ Operation:}\quad \text{Never use } SIGKILL \text{ as a default; reserve it exclusively for uncooperative, frozen tasks.}$$

#### 2. `SIGHUP` (1)
Originally used to signify that the serial line connecting a physical terminal to the host had been disconnected ("hung up"). 

In modern server administration, long-running daemons (such as Nginx, Apache, and PostgreSQL) do not run attached to physical terminals. By convention, they repurpose `SIGHUP` as a **live configuration reload trigger**:
```bash
# Instruct Nginx to parse new configs and spawn new workers without dropping TCP connections:
sudo kill -HUP $(cat /run/nginx.pid)
```

#### 3. `SIGINT` (2) vs. `SIGQUIT` (3)
*   **`SIGINT` (Signal 2)**: Generated by pressing `Ctrl+C`. The TTY line discipline parses this key binding and transmits the signal to all processes within the **foreground process group**. It requests that the interactive command abort its execution.
*   **`SIGQUIT` (Signal 3)**: Generated by pressing `Ctrl+\`. In addition to terminating the foreground process group, the kernel captures the processor context and memory footprint, generating a `core` file in the current working directory (or systemd-coredump storage) for debugging.

---

### Signal Registration and Async-Signal-Safety

#### Setting Up Signal Handlers: `sigaction(2)`
The legacy `signal(2)` API has historical semantics that varied between BSD and System V implementations. Production software relies exclusively on `sigaction(2)`:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <string.h>

void safe_signal_handler(int signum) {
    // Write is async-signal-safe; printf is NOT!
    const char msg[] = "Received SIGINT. Cleanly shutting down...\n";
    write(STDOUT_FILENO, msg, sizeof(msg) - 1);
    _exit(0);
}

int main(void) {
    struct sigaction sa;
    memset(&sa, 0, sizeof(sa));

    sa.sa_handler = safe_signal_handler;
    sigemptyset(&sa.sa_mask); // Block no extra signals during handler execution
    sa.sa_flags = SA_RESTART; // Automatically restart interrupted system calls

    if (sigaction(SIGINT, &sa, NULL) == -1) {
        perror("sigaction");
        exit(EXIT_FAILURE);
    }

    while (1) {
        pause(); // Sleep until a signal arrives
    }
    return 0;
}
```

#### The Constraint of Async-Signal Safety
Because a signal can interrupt process execution at **any arbitrary CPU instruction**, a signal handler must not alter global state in a way that creates race conditions or deadlocks.
*   **Non-Reentrant Functions**: If a thread calls `malloc()` or `printf()`, it acquires an internal glibc allocator/stream lock. If interrupted by a signal, and the signal handler *also* calls `printf()` or `malloc()`, the thread attempts to acquire a lock it already holds, causing an **instant deadlock**.
*   **POSIX Async-Signal-Safe Functions**: POSIX explicitly enumerates a restricted whitelist of safe functions callable from inside signal handlers (`write(2)`, `read(2)`, `_exit(2)`, `sigprocmask(2)`).
*   **Volatile Flags**: If communicating between a signal handler and the main execution loop, developers must use the `volatile sig_atomic_t` integer type to guarantee atomic, non-cached CPU memory reads and writes:
```c
volatile sig_atomic_t keep_running = 1;

void handle_sigterm(int sig) {
    keep_running = 0;
}
```

---

## 4. Interactive Execution Control and Terminal Topologies

### Sessions, Process Groups, and Controlling Terminals

To understand background jobs, disowning, and terminal management, one must understand POSIX execution containerization: **Sessions** and **Process Groups**.

```
Session (SID: 2000) - Managed by Session Leader (Bash, PID 2000)
Controlling Terminal: /dev/pts/2
 │
 ├── Foreground Process Group (PGID: 3100)  <-- Receives Terminal Keyboard Inputs
 │    ├── grep "ERROR" /var/log/syslog (PID 3100)
 │    └── awk '{print $1}' (PID 3101)
 │
 └── Background Process Groups (No Terminal Input Access)
      ├── Background Group 1 (PGID: 3200)
      │    └── python3 worker.py & (PID 3200)
      │
      └── Background Group 2 (PGID: 3300) [STOPPED]
           └── vim config.txt (PID 3300)
```

#### 1. Process Group
A collection of one or more related processes identified by a **Process Group ID** ($PGID$).
*   When you execute a multi-stage shell pipeline:
    ```bash
    cat /var/log/nginx/access.log | grep "500" | sort | uniq -c
    ```
    The shell forks four distinct processes. To manage them collectively, the shell groups all four tasks into a single **Process Group**, setting the $PGID$ equal to the $PID$ of the first process (`cat`).
*   Delivering a signal to a process group delivers it to every process in the pipeline simultaneously:
    ```c
    kill(-PGID, SIGTERM); // A negative PID targets the entire Process Group!
    ```

#### 2. Session
A collection of one or more process groups sharing a single **controlling terminal** (`/dev/pts/X`).
*   A session is created when a user authenticates via SSH or opens a terminal window. The login shell invokes `setsid(2)` and becomes the **Session Leader**, where $\text{SID} = \text{PID} = \text{PGID}$.
*   A session has at most **one foreground process group** and zero or more **background process groups**.
*   **The Controlling Terminal (`/dev/tty`)**: Transmits keyboard input and signals exclusively to the foreground process group. If a background process attempts to read from the controlling terminal via `read(0, ...)`, the terminal line discipline intercepts the operation, pauses the process group, and delivers a `SIGTTIN` signal.

---

### Shell Job Control Mechanics

The shell manages process groups using **Job Control**.

#### Job Management Lifecycle Commands
*   **Execution in Background (`&`)**: Appending an ampersand tells the shell to place the new process group into a background job queue rather than transferring terminal focus to it:
    ```bash
    python3 data_ingest.py > /tmp/out.log 2>&1 &
    # [1] 84210  <-- [Job ID] Process ID (PID)
    ```
*   **Suspending Execution (`Ctrl+Z`)**: Transmits a `SIGTSTP` signal to the current foreground process group, pausing execution and shifting the task to the shell's job table in a `Stopped` state:
    ```text
    ^Z
    [1]+  Stopped                 sleep 100
    ```
*   **Inspecting Active Jobs (`jobs -l`)**: Queries the shell's internal job tracking table:
    ```bash
    $ jobs -l
    [1]+ 84210 Stopped                 sleep 100
    [2]- 84215 Running                 python3 data_ingest.py &
    ```
    *   `+`: Represents the *current* default job (referenced by `%+` or `%%`).
    *   `-`: Represents the *previous* job (referenced by `%-`).
*   **Resuming in Background (`bg`)**: Sends a `SIGCONT` signal to the targeted stopped job, instructing it to resume execution while remaining in the background:
    ```bash
    bg %1
    # [1]+ sleep 100 &
    ```
*   **Promoting to Foreground (`fg`)**: Transfers terminal focus and controlling TTY input to the background job:
    ```bash
    fg %1
    ```

---

### Terminal Disowning and Process Detachment

When an interactive shell terminates (such as closing a terminal window or an SSH connection dropping), the kernel line discipline sends a `SIGHUP` (Signal 1) to the **Session Leader**. In turn, the standard shell iterates through its job table and broadcasts `SIGHUP` to every active child process, causing all active background jobs to terminate abruptly.

To ensure long-running background tasks survive shell termination, you can use several detachment strategies:

```
                      Process Detachment Mechanics
┌────────────────────────────────────────────────────────────────────────┐
│ 1. nohup command &                                                     │
│    - Masks SIGHUP (SIG_IGN) before execve()                            │
│    - Redirects stdin from /dev/null                                    │
│    - Redirects stdout/stderr to nohup.out                              │
└────────────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────────┐
│ 2. command & disown -h %1                                              │
│    - Launches task under standard shell control                        │
│    - disown -h removes SIGHUP delivery mark from internal job table    │
│    - Job remains alive if shell exits cleanly                          │
└────────────────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────────────────┐
│ 3. Terminal Multiplexer (tmux / screen)                                │
│    - Spawns persistent detached daemon session                         │
│    - Owns virtual PTY independent of external SSH transport            │
└────────────────────────────────────────────────────────────────────────┘
```

#### 1. The `nohup` Utility
`nohup` (No Hangup) prepares the execution environment before the target application loads:
```bash
nohup /usr/local/bin/backup_database.sh > /var/log/backup.log 2>&1 &
```
*   Sets the signal disposition of `SIGHUP` to `SIG_IGN` (Ignore).
*   If standard output is connected to a terminal, `nohup` redirects it to append to `./nohup.out` (or `$HOME/nohup.out`).
*   Redirects standard input to `/dev/null` to prevent the process from hanging on `SIGTTIN`.
*   Even if the parent shell closes and attempts to broadcast `SIGHUP`, the running child ignores the signal.

#### 2. The `disown` Builtin
`disown` is a Bash/Zsh shell builtin that alters the shell's internal job tracking table:
```bash
# 1. Launch a job in the background:
python3 heavy_computation.py &
# [1] 94102

# 2. Disown the job:
disown %1

# 3. Disown while keeping in job table but suppressing SIGHUP (-h):
disown -h %1

# 4. Disown ALL active background jobs (-a):
disown -a
```
*   By default, `disown %N` completely removes Job $N$ from the shell's active job table. When the shell terminates, it only broadcasts `SIGHUP` to jobs registered in that table. Because the task has been unlinked, it receives no signal and is adopted by `PID 1`.
*   `disown -h` leaves the job in the shell's `jobs` list, but marks it with a flag instructing the shell not to send `SIGHUP` on session exit.

#### Comparison Matrix: Process Persistence Techniques

| Mechanism | SIGHUP Protection | Terminal Detachment | Re-attachable UI? | Primary Use Case |
| :--- | :--- | :--- | :--- | :--- |
| `nohup <cmd> &` | **Yes** (`SIG_IGN`) | Partial (stdin $\to$ `/dev/null`) | No | Unattended scripts and legacy batch jobs. |
| `disown -h %N` | **Yes** (Shell skips signal) | No (still references PTY) | No | Preserving a job started in the foreground and pushed to background. |
| `tmux` / `screen` | **Yes** (Independent PTY) | Complete (Virtual session) | **Yes** | Interactive remote engineering sessions over unstable connections. |
| `systemd-run` | **Yes** (Runs in `system.slice`)| Complete (System Cgroup) | No (Use `journalctl`)| Production service-like background workloads. |

---

### Process Inspection Utilities

#### 1. `ps` (Process Status): Syntax Standards and Diagnostic Fields
`ps` supports two distinct syntax paradigms: **Standard POSIX Syntax** (prefixed with hyphens, e.g., `ps -ef`) and **BSD Syntax** (un-prefixed, e.g., `ps aux`).

```bash
# High-Fidelity POSIX Diagnostics:
ps -eo pid,ppid,pgid,sid,stat,user,%cpu,%mem,wchan:20,comm
```

##### Deciphering the BSD `STAT` Field Codes
The `STAT` column encodes internal process state using a series of character codes:

```
                       STAT Code Composition
                    ┌─────────────────────────┐
                    │ Primary Execution State │  (First character)
                    └────────────┬────────────┘
                                 │
         ┌───────────┬───────────┼───────────┬───────────┐
         ▼           ▼           ▼           ▼           ▼
       R (Run)    S (Sleep)   D (Disk/IO) T (Stop)   Z (Zombie)
                                 │
                    ┌────────────┴────────────┐
                    │ Extended Attribute Flags│  (Subsequent characters)
                    └────────────┬────────────┘
                                 │
         ┌───────────┬───────────┼───────────┬───────────┐
         ▼           ▼           ▼           ▼           ▼
      < (High)     N (Low)     s (Leader)  + (Foregr)  l (Threads)
```

*   **Primary States**:
    *   `R`: `TASK_RUNNING` (Running or ready on the run queue).
    *   `S`: `TASK_INTERRUPTIBLE` (Interruptible sleep waiting for an event/signal).
    *   `D`: `TASK_UNINTERRUPTIBLE` (Uninterruptible sleep waiting for disk or hardware I/O).
    *   `T`: `TASK_STOPPED` (Stopped by job control signal).
    *   `t`: `TASK_TRACED` (Stopped by a debugger via `ptrace`).
    *   `Z`: `EXIT_ZOMBIE` (Defunct, terminated, pending parent harvest).
*   **Secondary Attribute Modifiers**:
    *   `<`: High-priority task (negative nice value, favoring scheduler allocation).
    *   `N`: Low-priority task (positive nice value, yielding CPU time).
    *   `s`: Session Leader (e.g., login shells, managing an execution session).
    *   `l`: Multi-threaded task (the process contains multiple cloned threads via `CLONE_VM`).
    *   `+`: Process is a member of the **foreground process group** attached to the controlling terminal.

##### The `wchan` Diagnostic Column
When a process is sleeping in `S` or `D` state, passing `wchan` instructs `ps` to display the specific **kernel function** in which the process is currently blocked (e.g., `nfs_wait_event`, `futex_wait_queue_me`, `do_select`):
```bash
$ ps -eo pid,stat,wchan:25,comm | grep -E "D|comm"
  PID STAT WCHAN                     COMMAND
 9841 D    nfs_wait_bit_uninterrupt  backup-sync
```

---

#### 2. `top`: Real-Time Scheduling and Memory Metrics
`top` provides a real-time, dynamic view of active kernel processes:

```text
top - 12:45:01 up 14 days,  3:22,  2 users,  load average: 0.15, 0.22, 0.18
Tasks: 214 total,   1 running, 212 sleeping,   1 stopped,   0 zombie
%Cpu(s):  2.3 us,  0.7 sy,  0.0 ni, 96.5 id,  0.3 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :  15820.4 total,   3102.1 free,   4210.8 used,   8507.5 buff/cache
MiB Swap:   4096.0 total,   4096.0 free,      0.0 used.  11215.4 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 2401 postgres  20   0  521408 112044  94100 S   1.7   0.7  12:41.20 postgres
 1042 root      20   0 1204210  45210  28100 S   0.7   0.3   4:10.80 containerd
```

##### Critical Diagnostic Metrics in Header
*   **Load Average**: The average number of processes in `TASK_RUNNING` ($R$) and `TASK_UNINTERRUPTIBLE` ($D$) states across 1, 5, and 15-minute windows.
*   **CPU Breakdown**:
    *   `us` (User): Percentage of time running un-niced userspace processes.
    *   `sy` (System): Percentage of time running kernel code and handling system calls.
    *   `wa` (I/O Wait): Percentage of CPU cycles idle while processes are blocked in `D` state waiting on storage devices. High `wa` indicates storage hardware or network filesystem saturation.
    *   `st` (Steal): In virtualized environments, the percentage of time physical CPU capacity was taken by the underlying hypervisor to service other virtual machines.

##### Essential Interactive Shortcuts
*   `k`: Prompts for a target $PID$ and numeric signal to send (defaults to `15` / `SIGTERM`).
*   `r`: Prompts for a target $PID$ to adjust scheduling priority (**renice** from $-20$ down to $+19$).
*   `M`: Sorts the process list by resident memory footprint (`RES`).
*   `P`: Sorts the process list by CPU utilization (`%CPU`, default).
*   `T`: Sorts the process list by cumulative CPU time consumed (`TIME+`).
*   `1`: Toggles the header between a single aggregated CPU state and individual breakdowns for every CPU core.
*   `c`: Toggles the `COMMAND` display between binary names and full executable arguments.

---

#### 3. Targeted Process Filtering and Signaling: `pgrep` and `pkill`

Piping `ps aux` into `grep` is a common operational anti-pattern:
```bash
# ANTI-PATTERN: Prone to matching the grep process itself:
ps aux | grep nginx | awk '{print $2}' | xargs kill -9
```

The `pgrep` and `pkill` utilities (from the `procps` package) query the kernel's `/proc` directory directly, bypassing string-parsing hazards:

```bash
# 1. Match processes by full command line string (-f):
pgrep -f "worker.py --env=prod"

# 2. Interrogate processes owned by a specific user (-u):
pgrep -u www-data -l

# 3. Exact binary name match (-x):
pgrep -x python3

# 4. Target the OLDEST (-o) or NEWEST (-n) matching process:
pgrep -n -f "celery worker"

# 5. List matching PIDs alongside process names (-l) or full arguments (-a):
pgrep -a -u root sshd

# 6. Deliver explicit signals directly to filtered sets using pkill:
sudo pkill -HUP -f "/usr/sbin/nginx"
sudo pkill -15 -u deploy_user
```

---

## 5. Practical Laboratories and Diagnostic Walkthroughs

### Lab 1: Tracing `fork`, `execve`, and `wait4` via `strace`

#### Objective
Trace how the shell interprets a command, creates a child process, sets up redirection, executes a binary, and reaps the child status using `strace`.

#### Execution Command
```bash
strace -f -e trace=clone,fork,vfork,execve,wait4,close,dup2 bash -c 'ls /dev/null > /dev/null'
```

#### Annotated Trace Walkthrough
```c
// 1. Parent Shell (PID 4000) clones itself to create child task:
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, ...) = 4001

// 2. Child Process (PID 4001) prepares redirections (dup2):
[pid 4001] dup2(3, 1)                   = 1
[pid 4001] close(3)                     = 0

// 3. Child Process (PID 4001) replaces memory space with /bin/ls:
[pid 4001] execve("/usr/bin/ls", ["ls", "/dev/null"], 0x7ffd1...) = 0

// 4. Parent Process (PID 4000) pauses execution waiting for child 4001:
wait4(4001, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 4001

// 5. Kernel delivers SIGCHLD to 4000; child 4001 transitions to EXIT_DEAD.
--- SIGCHLD {si_signo=SIGCHLD, si_code=CLD_EXITED, si_pid=4001, si_status=0} ---
```

---

### Lab 2: Investigating and Neutralizing a Leaked Zombie Process

#### Objective
Write a self-contained C program that leaks a zombie process, diagnose its presence through terminal introspection tools, trace its parentage, and eliminate the zombie without restarting the system.

#### Step 1: Compile and Run the Problematic Binary
Save the following code as `zombie_factory.c`:
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork");
        exit(1);
    }

    if (pid == 0) {
        // Child terminates instantly:
        printf("[+] Child (%d) terminating now.\n", getpid());
        exit(0);
    }

    // Parent deliberately fails to call wait() and loops indefinitely:
    printf("[*] Parent (%d) alive. Ignoring child status...\n", getpid());
    while (1) {
        sleep(1);
    }
    return 0;
}
```
Compile and execute in the background:
```bash
gcc -Wall zombie_factory.c -o zombie_factory
./zombie_factory &
# Output:
# [*] Parent (88210) alive. Ignoring child status...
# [+] Child (88211) terminating now.
```

#### Step 2: Diagnose the Zombie via `ps`
Query for processes in `Z` state:
```bash
$ ps -eo pid,ppid,stat,comm | grep -E "Z|COMM"
    PID    PPID STAT COMMAND
  88211   88210 Z+   zombie_factory <defunct>
```
The output confirms:
*   The zombie process is PID `88211`.
*   Its Parent Process ID ($PPID$) is `88210`.
*   The process is marked `Z+` (`EXIT_ZOMBIE`, foreground group of subshell).

#### Step 3: Attempt Direct Termination (Verification of Failure)
Attempt to force-kill the zombie:
```bash
kill -9 88211
ps -p 88211 -o pid,stat,comm
# Result: PID 88211 STILL EXISTS. SIGKILL has no effect on a dead task!
```

#### Step 4: Neutralize the Issue via Parent Remediation
Send `SIGCHLD` to the parent to check if it has a latent signal handler:
```bash
kill -CHLD 88210
ps -p 88211 -o pid,stat,comm
# PID 88211 remains, indicating parent does not handle SIGCHLD.
```
Terminate the unresponsive parent process:
```bash
kill -15 88210
# Verify both parent and child have been cleaned up:
ps -p 88210,88211 -o pid,ppid,stat,comm
# Output: PID 88210 and 88211 no longer exist in the system.
```
*Result*: When parent `88210` exited, the kernel reparented zombie `88211` to `systemd` (`PID 1`), which reaped it immediately.

---

### Lab 3: Crafting a Signal-Resilient Bash Worker

#### Objective
Author a robust Bash daemon script that catches terminal signals (`SIGINT`, `SIGTERM`), ignores hangup signals (`SIGHUP`), gracefully cleans up temporary runtime state files, and safely exits.

#### Script Implementation (`resilient_worker.sh`)
```bash
#!/usr/bin/env bash
set -u

LOCK_FILE="/tmp/worker_job.lock"
RUNNING=1

# Clean shutdown function:
cleanup() {
    echo -e "\n[*] Intercepted termination signal. Initiating cleanup..."
    if [[ -f "${LOCK_FILE}" ]]; then
        rm -f "${LOCK_FILE}"
        echo "[+] Lock file (${LOCK_FILE}) removed."
    fi
    echo "[+] Cleanup complete. Exiting gracefully."
    exit 0
}

# Trap registrations:
# Catch SIGINT (Ctrl+C) and SIGTERM (Kill command):
trap cleanup SIGINT SIGTERM

# Ignore SIGHUP:
trap '' SIGHUP

# Create temporary lock file:
echo "$$" > "${LOCK_FILE}"
echo "[*] Worker started with PID $$."
echo "[*] Registered traps: SIGINT/SIGTERM -> cleanup(), SIGHUP -> IGNORE."
echo "[*] Simulating background processing. Press Ctrl+C or kill $$ to stop..."

# Main execution loop:
COUNTER=0
while [[ ${RUNNING} -eq 1 ]]; do
    ((COUNTER++))
    echo "[$(date +%T)] Processing batch #${COUNTER}..."
    sleep 2
done
```

#### Verification Workflow
1.  Run the script in the foreground:
    ```bash
    chmod +x resilient_worker.sh
    ./resilient_worker.sh
    ```
2.  In a separate terminal, issue a reload/hangup test:
    ```bash
    kill -HUP $(pgrep -f resilient_worker.sh)
    # Observe: Worker ignores SIGHUP and continues processing without halting!
    ```
3.  Terminate the worker gracefully:
    ```bash
    kill -15 $(pgrep -f resilient_worker.sh)
    # Observe: Worker executes cleanup() routine, unlinks /tmp/worker_job.lock, and exits cleanly.
    ```

---

## 6. Comprehensive Reference Matrices

### Linux Process State Codes Matrix

| Code | State Classification | Interruptible? | Consumes CPU? | Causes Load Average Spike? | Description & Diagnostic Interpretation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`R`** | `TASK_RUNNING` | Yes | **Yes** | **Yes** | Process is executing on a CPU core or sitting in the active CFS run-queue awaiting a time slice. |
| **`S`** | `TASK_INTERRUPTIBLE` | **Yes** | No | No | Sleeping on an event, socket, lock, or timer. Wakes up immediately upon receiving a signal. |
| **`D`** | `TASK_UNINTERRUPTIBLE`| **NO** | No | **Yes** | Deep hardware sleep. Blocks all signal delivery. Typically waiting on physical disk I/O, synchronous locks, or unresponsive NFS mounts. |
| **`T`** | `TASK_STOPPED` | No | No | No | Process execution suspended via job control signals (`SIGSTOP`, `SIGTSTP`). Retains memory; barred from CPU scheduling until `SIGCONT`. |
| **`t`** | `TASK_TRACED` | No | No | No | Execution paused by an attached debugging tracer (e.g., `gdb`, `strace`) using `ptrace(2)`. |
| **`Z`** | `EXIT_ZOMBIE` | No | No | No | Terminated process whose parent has not yet called `waitpid()`. Consumes a slot in the kernel PID table, but no RAM or CPU. |
| **`X`** | `EXIT_DEAD` | N/A | N/A | N/A | Ephemeral kernel state during final cleanup of the `task_struct`. Never observable in userspace utilities. |

---

### Process Management Commands and Operators

| Command / Operator | Layer | Primary Function | Key Syntax & Flag Examples |
| :--- | :--- | :--- | :--- |
| **`&`** | Shell Syntax | Dispatches a command pipeline directly into a background process group. | `make -j8 &` |
| **`Ctrl+Z`** | Terminal / Line Disc. | Sends `SIGTSTP` to the foreground process group, suspending its execution. | Interactively typed in terminal. |
| **`jobs`** | Shell Builtin | Displays the shell's internal table of managed foreground and background jobs. | `jobs -l` (Displays PIDs). |
| **`bg`** | Shell Builtin | Sends `SIGCONT` to a stopped job, resuming its execution in the background. | `bg %2` |
| **`fg`** | Shell Builtin | Brings a background or stopped job into the foreground, restoring terminal input. | `fg %1` |
| **`disown`** | Shell Builtin | Removes jobs from the shell's active table to prevent `SIGHUP` delivery on exit. | `disown -h %1`<br>`disown -a` (All jobs). |
| **`nohup`** | Core Utility | Masks `SIGHUP` (`SIG_IGN`) and redirects terminal streams to disk before executing. | `nohup script.sh > out.log 2>&1 &` |
| **`kill`** | System / Builtin | Sends a specified signal to a process, process group, or thread group. | `kill -15 <pid>`<br>`kill -9 -<pgid>` (Process Group). |
| **`killall`** | Process Utility | Sends signals to all active processes matching a designated binary name. | `killall -HUP httpd` |
| **`pgrep`** | Process Query | Scans `/proc` and outputs the Process IDs matching defined filtering criteria. | `pgrep -u www-data -f "celery"` |
| **`pkill`** | Process Signaling | Finds and signals processes matching defined attribute criteria directly. | `pkill -9 -f "worker.py"` |
| **`ps`** | System Snapshot | Reports a static snapshot of all processes running across the operating system. | `ps aux`<br>`ps -eo pid,ppid,stat,comm` |
| **`top`** | Dynamic Monitor | Provides real-time dynamic process scheduling, memory footprints, and CPU load. | Interactive: `k` (kill), `r` (renice), `P` (sort CPU). |
| **`pstree`** | Tree Visualizer | Renders a visual hierarchical tree of running processes illustrating lineages. | `pstree -psa <pid>` |