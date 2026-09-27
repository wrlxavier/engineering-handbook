# 02. File Descriptors and I/O Redirection

In Unix-like operating systems, the abstraction "everything is a file" is not merely a high-level design mantra; it is an architectural contract enforced by the Virtual File System (VFS) and the kernel process model. Every interaction a process maintains with storage media, inter-process communication (IPC) channels, terminal emulators, hardware devices, and network sockets is mediated through uniform byte streams referenced by **file descriptors**.

Mastering file descriptors and I/O redirection requires looking beneath high-level shell syntax into kernel-level pointer arrays, system call mechanics, and userspace buffering layers.

---

## 1. Standard Descriptor Theory: `stdin` (0), `stdout` (1), and `stderr` (2)

### Kernel Architecture of File Descriptors

In the Linux kernel, every running process is represented by a `struct task_struct`. Within this structure lies a pointer to `struct files_struct`, which manages the process's open file table.

```
Process Address Space (task_struct)
 └── struct files_struct *files
      └── struct fdtable *fdt
           └── struct file **fd  ──> [0] ───┐
                                     [1] ───┼─┐
                                     [2] ───┼─┼─┐
                                     [3]    │ │ │
                                     ...    │ │ │
                                            ▼ ▼ ▼
                           Kernel System-Wide Open File Table
                         ┌────────────────────────────────────┐
                         │ struct file                        │
                         │  ├── f_mode (FMODE_READ, WRITE)    │
                         │  ├── f_pos (64-bit byte offset)    │
                         │  ├── f_count (reference counter)   │
                         │  ├── f_flags (O_APPEND, O_NONBLOCK)│
                         │  └── struct path                   │
                         │       └── struct dentry            │
                         │            └── struct inode        │
                         └────────────────────────────────────┘
                                            │
                                            ▼
                                 Underlying Hardware / VFS
                              (Ext4 Inode, PTY, Pipe, Socket)
```

1. **The Descriptor Array (`fdtable`)**:
   A file descriptor ($FD$) is simply a non-negative integer ($0, 1, 2, \dots, N$) that acts as an index into the private pointer array (`fd`) of the process. The maximum number of file descriptors a process can allocate is governed by `ulimit -n` (POSIX `RLIMIT_NOFILE`) and system-wide limits in `/proc/sys/fs/file-max`.

2. **The Open File Description (`struct file`)**:
   The index in `fdtable` points to a system-wide open file description structure maintained in kernel memory. This structure tracks:
   * The current file offset (`f_pos`).
   * Access modes and status flags (`f_mode`, `f_flags`).
   * The reference count (`f_count`) indicating how many file descriptors across all processes point to this specific description.
   * Pointers to the VFS inode operations table (`f_op`).

3. **The Inode (`struct inode`)**:
   The `struct file` points to the VFS inode representing the physical resource—a regular block file on disk, a character device node (such as `/dev/pts/1`), an anonymous pipe buffer, or a network socket.

### The Standard Trio

When a POSIX shell initializes a new command execution environment, it ensures three standard streams are available by default:

| Integer Value | POSIX C Macro | Shell Name | Default Channel Target | Purpose |
|---|---|---|---|---|
| **`0`** | `STDIN_FILENO` | `stdin` | Keyboard / Terminal Input (`/dev/pts/X`) | Data consumption stream |
| **`1`** | `STDOUT_FILENO` | `stdout` | Terminal Display Output (`/dev/pts/X`) | Primary program payload |
| **`2`** | `STDERR_FILENO` | `stderr` | Terminal Display Output (`/dev/pts/X`) | Diagnostics, errors, prompts |

```
                       +------------------------+
                       |      Linux Kernel      |
                       +------------------------+
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
        FD 0 (stdin)         FD 1 (stdout)        FD 2 (stderr)
     [Read Operations]    [Write Operations]   [Write Operations]
              ▲                    │                    │
              │                    ▼                    ▼
     ┌─────────────────┐  ┌─────────────────────────────────────┐
     │ Terminal Driver │  │           Terminal Driver           │
     │  (PTY / Keyboard│  │       (PTY / Display Display)       │
     └─────────────────┘  └─────────────────────────────────────┘
```

#### Why `stderr` Exists
Doug McIlroy introduced `stderr` to Unix after discovering that composing programs via pipes (`|`) broke down when diagnostics and data shared the same channel. If a tool writing structured tabular output encountered an unreadable record and printed `Error: Line 4 corrupted` directly into `stdout`, downstream utilities (`sort`, `awk`, `cut`) would ingest that error message as valid data, corrupting the pipeline. 

By dedicating FD 2 specifically to out-of-band operational reporting, data pipelines remain clean while operators retain visibility into runtime faults.

### Buffering Modes: Kernel vs. Userspace (`glibc` stdio)

A frequent point of confusion in stream processing stems from the interplay between kernel I/O and userspace runtime buffering (`glibc` standard I/O library: `fprintf`, `fread`, `fwrite`).

* **Unbuffered (Kernel Level)**: When a process invokes the raw system call `write(1, buf, count)`, the bytes are transferred immediately into the kernel page cache or the pipe buffer. There is no internal kernel delay.
* **Buffered (Userspace Level)**: To minimize expensive system call overhead, the C standard library wraps raw file descriptors inside `FILE *` streams (`stdin`, `stdout`, `stderr`), applying one of three buffering disciplines:

```
[ Application Layer (printf/puts) ]
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│               glibc stdio Buffer Layer                 │
│                                                        │
│  1. Unbuffered (_IONBF):                               │
│     Used by stderr. Flushes immediately on every byte. │
│                                                        │
│  2. Line Buffered (_IOLBF):                            │
│     Default when FD points to a Terminal (isatty(3)==1)│
│     Flushes on '\n' or when buffer fills.              │
│                                                        │
│  3. Fully Buffered (_IOFBF):                           │
│     Default when FD points to a File or Pipe (4-8 KiB).│
│     Flushes ONLY when buffer is 100% full.             │
└────────────────────────┬───────────────────────────────┘
                         │
                         ▼  (system call: write(2))
┌────────────────────────────────────────────────────────┐
│           Kernel Buffer (Pipe Ring / Page Cache)       │
└────────────────────────────────────────────────────────┘
```

#### The "Hanging Pipeline" Trap
Consider this pipeline:
```bash
tail -f /var/log/nginx/access.log | grep "500" | awk '{print $1}'
```
Operators often observe that `awk` outputs nothing for minutes, even while matches appear in the log. 

**Root Cause**: When `grep` detects its standard output is connected to a **pipe** instead of a terminal (`isatty(1) == 0`), `glibc` switches `stdout` from line-buffered to fully buffered (typically $4\,096$ or $8\,192$ bytes). `grep` buffers matching lines in userspace memory, refusing to call `write(2)` until the $4\text{ KiB}$ threshold is crossed.

**Remediations**:
```bash
# 1. Force line buffering using GNU coreutils stdbuf:
stdbuf -oL -eL grep "500" /var/log/nginx/access.log | awk '{print $1}'

# 2. Use utility-specific flags:
grep --line-buffered "500" /var/log/nginx/access.log | awk '{print $1}'

# 3. Force awk to flush explicitly:
awk '{print $1; fflush()}'
```

### Inspecting Process File Descriptors

The Linux `/proc` pseudo-filesystem exposes the descriptor state of every running task:

```bash
# Inspect descriptors held open by the current shell:
$ ls -la /proc/$$/fd
total 0
dr-x------ 2 alice alice  0 Sep 27 03:00 .
dr-xr-xr-x 9 alice alice  0 Sep 27 03:00 ..
lrwx------ 1 alice alice 64 Sep 27 03:00 0 -> /dev/pts/1
lrwx------ 1 alice alice 64 Sep 27 03:00 1 -> /dev/pts/1
lrwx------ 1 alice alice 64 Sep 27 03:00 2 -> /dev/pts/1
lrwx------ 1 alice alice 64 Sep 27 03:00 255 -> /dev/pts/1

# Read detailed descriptor position, mount, and access flags:
$ cat /proc/$$/fdinfo/0
pos:    0
flags:  0102002        # Octal representation of O_RDWR | O_LARGEFILE
mnt_id: 28
ino:    4              # Target inode
```

---

## 2. Compound Redirection Constructs and Fine-Grained Output Control

Redirection is the process of altering the pointer entries in a process's file descriptor table before executing a command.

### Redirection Operators Syntax Overview

| Operator | Direction | Primary Meaning | Open Flags (Underlying `openat(2)`) |
|---|---|---|---|
| `< file` | Input | Directs `file` contents into stdin (FD 0) | `O_RDONLY` |
| `> file` | Output | Directs stdout (FD 1) to `file` (truncates) | `O_WRONLY \| O_CREAT \| O_TRUNC` |
| `>> file` | Output | Directs stdout (FD 1) to `file` (appends) | `O_WRONLY \| O_CREAT \| O_APPEND` |
| `[n]> file` | Output | Directs arbitrary FD `n` to `file` (truncates) | `O_WRONLY \| O_CREAT \| O_TRUNC` |
| `[n]>> file`| Output | Directs arbitrary FD `n` to `file` (appends) | `O_WRONLY \| O_CREAT \| O_APPEND` |
| `[n]< file` | Input | Directs `file` into arbitrary FD `n` | `O_RDONLY` |
| `>\| file` | Output | Overwrites `file` even if `noclobber` is set | `O_WRONLY \| O_CREAT \| O_TRUNC` |
| `<> file` | Both | Opens `file` for read-write on FD 0 (or `[n]<>`) | `O_RDWR \| O_CREAT` |

### Strict Left-to-Right Evaluation Order

Redirection operators are **not** parsed globally as an unordered set; they are processed sequentially from left to right. Understanding this evaluation order is vital when duplicating descriptors.

#### Case Study: `> file 2>&1` vs. `2>&1 > file`

Consider the redirection syntax for combining stdout and stderr into a single target file.

##### 1. Correct Construct: `command > output.log 2>&1`

```
Initial State:
FD 1 ──> /dev/pts/1 (Terminal)
FD 2 ──> /dev/pts/1 (Terminal)

Step 1: Parse '> output.log'
Shell opens output.log (O_WRONLY | O_CREAT | O_TRUNC).
dup2() points FD 1 to output.log.
FD 1 ──> output.log
FD 2 ──> /dev/pts/1 (Terminal)

Step 2: Parse '2>&1'
Shell duplicates FD 1 onto FD 2 via dup2(1, 2).
FD 2 now points to whatever FD 1 is currently pointing to (output.log).
FD 1 ──> output.log
FD 2 ──> output.log

Result: Both streams write cleanly into output.log.
```

##### 2. Faulty Construct: `command 2>&1 > output.log`

```
Initial State:
FD 1 ──> /dev/pts/1 (Terminal)
FD 2 ──> /dev/pts/1 (Terminal)

Step 1: Parse '2>&1'
Shell duplicates FD 1 onto FD 2.
FD 1 points to /dev/pts/1, so FD 2 points to /dev/pts/1.
FD 1 ──> /dev/pts/1 (Terminal)
FD 2 ──> /dev/pts/1 (Terminal)

Step 2: Parse '> output.log'
Shell opens output.log.
dup2() points FD 1 to output.log.
FD 1 ──> output.log
FD 2 ──> /dev/pts/1 (Terminal)

Result: FD 1 (stdout) goes to output.log, but FD 2 (stderr) remains 
attached to the terminal display.
```

### Modern Bash Syntactic Sugar vs. POSIX Portability

Bash (and Zsh) provide abbreviated notation for combining stdout and stderr:

```bash
# Truncating combined output:
command &> output.log       # Bash 3.0+ shorthand
command >& output.log       # Older alternative syntax (Bash-specific)

# Appending combined output:
command &>> output.log      # Bash 4.0+ shorthand

# POSIX-Compliant Portable Alternative (Use in #!/bin/sh scripts):
command > output.log 2>&1
command >> output.log 2>&1
```

> **Warning on Ambiguity**: Avoid `>&` in POSIX scripts. In standard POSIX syntax, `>&` is reserved strictly for descriptor duplication (`2>&1`). If the target word expands to an integer, `>&` acts as descriptor duplication rather than file redirection.

### The `noclobber` Safeguard (`set -C`)

Accidental file truncation via typing `>` instead of `<` or `>>` is a catastrophic shell hazard. The shell provides the `noclobber` option to prevent overwriting existing files:

```bash
$ set -o noclobber   # or: set -C
$ echo "Payload" > important_data.json
$ echo "New line" > important_data.json
bash: important_data.json: cannot overwrite existing file

# Forcing truncation through the safeguard using the clobber operator (>|):
$ echo "Override payload" >| important_data.json
```

### Swapping `stdout` and `stderr` (The 3-Way File Descriptor Swap)

Suppose a command emits raw binary payload on `stdout` and status logs on `stderr`. You must pass the status logs to a downstream pipe (`grep`) while preserving the binary payload to disk:

```bash
# Objective: Swap FD 1 and FD 2 for the target command
command 3>&1 1>&2 2>&3 3>&- | grep "ERROR"
```

Let's trace the descriptor table changes:

```
Step 0 (Initial):
FD 1 ──> Pipe to grep
FD 2 ──> Terminal (/dev/pts/1)

Step 1: '3>&1'
Duplicate FD 1 onto newly allocated FD 3.
FD 3 ──> Pipe to grep

Step 2: '1>&2'
Duplicate FD 2 onto FD 1.
FD 1 ──> Terminal (/dev/pts/1)

Step 3: '2>&3'
Duplicate FD 3 onto FD 2.
FD 2 ──> Pipe to grep

Step 4: '3>&-'
Close temporary FD 3.
FD 3 ──> [CLOSED]

Final Result:
FD 1 (stdout) ──> Terminal (/dev/pts/1)
FD 2 (stderr) ──> Pipe to grep
```

---

## 3. Creating Custom File Descriptors for Script I/O Control

By default, redirections applied to a command terminate when that command finishes executing. To retain file descriptor configurations across multiple commands, or to create dedicated parallel I/O channels, Bash provides the `exec` builtin.

### Persistent Stream Modification via `exec`

When `exec` is invoked without a command binary argument, it **does not** replace the process image with a new binary. Instead, it alters the shell's own file descriptor table in-place:

```bash
#!/usr/bin/env bash

# Redirect all future stdout of this script process to an audit file:
exec > /var/log/script_execution.log 2>&1

echo "This goes directly to the log file"
ls -la /root
cat /nonexistent   # Both stdout and stderr are redirected to the file
```

### Allocating Static Custom Descriptors (FDs 3–9)

POSIX guarantees that shells support descriptor integers from $0$ through $9$. FDs $3$ through $9$ are available for developer use.

#### 1. Custom Channel for Input (FD 3)
```bash
#!/usr/bin/env bash
exec 3< /etc/os-release

# Read first line from FD 3:
read -r os_info <&3
echo "OS Line: $os_info"

# Read second line from FD 3 (internal offset f_pos was maintained!):
read -r os_info <&3
echo "Next Line: $os_info"

# Close FD 3:
exec 3<&-
```

#### 2. Custom Channel for Output (FD 4)
```bash
#!/usr/bin/env bash
exec 4> /tmp/debug.log

emit_debug() {
    # Send message to FD 4 without disrupting caller's stdout/stderr
    echo "[DEBUG $(date +%T)] $*" >&4
}

emit_debug "System initialized"
exec 4>&- # Close FD 4
```

#### 3. Read/Write Bidirectional Channels (FD `<>`)
Opening a descriptor with `<>` invokes the `openat(2)` system call with `O_RDWR | O_CREAT`. The file is created if missing, but it is **not truncated**. The read/write byte offset starts at position $0$:

```bash
exec 5<> /tmp/state.dat
read -r current_state <&5
echo "NEW_STATE" >&5
exec 5>&-
```

### Dynamic File Descriptor Allocation (Bash 4.1+)

Hardcoding integers like $3, 4, 5$ across large scripts creates collision risks, especially when library scripts or sourced files define their own descriptors. 

Bash 4.1 introduced dynamic allocation syntax using variable references: `{varname}>` and `{varname}<`.

```bash
#!/usr/bin/env bash

# Bash assigns the lowest available descriptor integer >= 10 to MY_LOG_FD:
exec {MY_LOG_FD}> /var/log/dynamic_audit.log

echo "Dynamically allocated descriptor number: $MY_LOG_FD"
# Output on typical systems: 10

# Direct data to the allocated descriptor using variable expansion:
echo "Transaction 001 SUCCESS" >&"$MY_LOG_FD"
echo "Transaction 002 FAILURE" >&"$MY_LOG_FD"

# Close the dynamic descriptor:
exec {MY_LOG_FD}>&-
```

### Production Pattern: Preserving Interactive Standard Input

A pervasive bug occurs when an outer `while read` loop consumes `stdin`, depriving interactive inner commands (`ssh`, `gpg`, or `read -p`) of input:

```bash
# THE BUGGY CODE:
cat servers.txt | while read -r server; do
    ssh "$server" "uname -a"  # ssh eats the remaining lines of servers.txt!
done
```

**The Solution with Custom Descriptors**:
Isolate the file reader onto FD 3, leaving FD 0 (`stdin`) connected to the interactive terminal:

```bash
#!/usr/bin/env bash

exec 3< "servers.txt"

while read -u 3 -r server; do
    echo "Connecting to: $server"
    # ssh receives interactive keyboard input from FD 0 without touching FD 3:
    ssh "$server" "sudo systemctl restart nginx"
done

exec 3<&-
```

### Concurrency Protection: Inter-Process File Locking via `flock`

Linux provides advisory file locking via `flock(2)`. You can bind a lock to an arbitrary descriptor to orchestrate concurrency across parallel script instances:

```bash
#!/usr/bin/env bash

LOCK_FILE="/var/lock/cluster_sync.lock"

# Open lock file for writing on FD 200:
exec 200>"$LOCK_FILE"

# Acquire an exclusive, non-blocking lock:
if ! flock -n 200; then
    echo "CRITICAL: Another instance is running! Terminating." >&2
    exit 1
fi

# Critical section begins
echo "Lock acquired. Executing sensitive sync operations..."
sleep 10
# Critical section ends

# The lock is automatically released when FD 200 closes or process exits:
exec 200>&-
```

---

## 4. Complex Data Input: Here Documents, Here Strings, and Process Substitution

Standard commands expect input either via command-line arguments or via byte streams on FD 0. Bash provides complex constructs to inject data into descriptors inline.

### Here Documents (`<<EOF`)

A Here Document (`HereDoc`) supplies an inline block of multiline text directly to a command's standard input.

```
                  +-----------------------------------+
                  |   Shell Parsing Here Document     |
                  +-----------------------------------+
                                    │
                       Is the Delimiter Quoted?
                              /          \
                            YES           NO
                            /              \
           [ Raw String Ingestion ]   [ Parameter Expansion ($VAR)    ]
           [ No expansions occur  ]   [ Command Substitution ($(cmd)) ]
           [ Literal \ preserved  ]   [ Arithmetic Eval ($((1+1)))    ]
                            \              /
                             ▼            ▼
                  +-----------------------------------+
                  | Write bytes to temporary storage  |
                  | (Virtual pipe or unlinked tmpfile)|
                  +-----------------------------------+
                                    │
                                    ▼
                  +-----------------------------------+
                  | Dup temporary storage onto FD 0   |
                  +-----------------------------------+
                                    │
                                    ▼
                  +-----------------------------------+
                  |          Execute Command          |
                  +-----------------------------------+
```

#### 1. Interpolated vs. Non-Interpolated HereDocs

* **Unquoted Delimiter (`<<EOF`)**: Expands variables, command substitutions, and arithmetic operations before feeding the stream to the command:
```bash
TARGET="/var/data"
cat <<EOF
Current user: $USER
Target directory: $TARGET
System time: $(date +%T)
EOF
```

* **Quoted Delimiter (`<<'EOF'`, `<<"EOF"`, `<<\EOF`)**: Suppresses **all** expansions. The stream is delivered byte-for-byte as written:
```bash
# Perfect for generating downstream scripts, Awk blocks, or configuration files:
cat <<'EOF' > /tmp/bootstrap.sh
#!/bin/sh
echo "Current PID is: $$"  # Literal $$, will NOT expand during generation
awk '{print $1}' /etc/hosts
EOF
```

#### 2. The Indentation-Stripping Operator (`<<-`)

A common formatting issue with HereDocs inside nested control blocks is that indentation spaces are treated as literal content. The `<<-` operator strips **leading tab characters** (`ASCII 0x09`), allowing you to indent the block alongside surrounding code:

```bash
if true; then
	cat <<-EOF
	This line has leading tabs in the script source.
	The tabs are stripped automatically at runtime.
	Leading spaces, however, are NOT stripped.
	EOF
fi
```

> **Crucial Rule**: The closing delimiter must also be indented using **only tabs**, not spaces. A single leading space prevents Bash from recognizing the termination tag, throwing a syntax error: `unexpected end of file while looking for matching delimiter`.

### Here Strings (`<<< "$string"`)

Here Strings feed a single string, followed by an automatic newline (`\n`), into standard input:

```bash
# Traditional subshell pipeline:
echo "192.168.1.10" | cut -d. -f4

# High-performance Here String alternative:
cut -d. -f4 <<< "192.168.1.10"

# Complex multi-line variables via Here String:
CONFIG_BLOB=$(curl -s https://api.internal/config)
jq '.database.port' <<< "$CONFIG_BLOB"
```

#### Performance Advantage of Here Strings
Piping data (`echo "$var" | command`) forces the shell to call `pipe(2)`, `fork(2)` twice (for both `echo` and `command`), and manage subshell process boundaries. 

The Here String (`command <<< "$var"`) runs within the main shell process, bypassing the extra subshell instantiation, and writes the string directly to a temporary file descriptor that is handed to the child process.

### Process Substitution: `<(command)` and `>(command)`

Process Substitution connects the output or input of a command directly to a filename argument, allowing non-linear data flows without intermediate disk writes.

```
Input Process Substitution: <(cmd)
┌──────────────┐     stdout     ┌────────────────┐    /dev/fd/63    ┌─────────────┐
│  cmd child   │ ─────────────> │ Anonymous Pipe │ <─────────────── │ Main Command│
└──────────────┘                └────────────────┘                  └─────────────┘

Output Process Substitution: >(cmd)
┌──────────────┐    /dev/fd/63  ┌────────────────┐     stdin        ┌─────────────┐
│ Main Command │ ─────────────> │ Anonymous Pipe │ ───────────────> │  cmd child  │
└──────────────┘                └────────────────┘                  └─────────────┘
```

#### Under the Hood: The `/dev/fd/N` Abstraction
When the shell evaluates `<(command)`:
1. It calls `pipe(2)` to create an anonymous pipe.
2. It forks an asynchronous subshell to run `command`, setting its `stdout` to the write end of the pipe.
3. It substitutes `<(command)` on the command line with a virtual path: `/dev/fd/63` (a symlink pointing to the read end of the pipe).
4. The outer command treats `/dev/fd/63` as a regular readable file path.

#### 1. Input Process Substitution: `<(cmd)`
Essential for tools that require multiple file arguments and do not accept raw standard input:

```bash
# Compare runtime system states without generating temporary files on disk:
diff -u <(ssh serverA "dpkg -l | sort") <(ssh serverB "dpkg -l | sort")

# Feeding input to while loops while avoiding subshell variable isolation:
line_count=0
while read -r line; do
    ((line_count++))
done < <(grep "CRITICAL" /var/log/syslog)

echo "Total errors found: $line_count"
# $line_count retains its value because the while loop executed in the main process!
```

#### 2. Output Process Substitution: `>(cmd)`
Enables non-linear branching data processing pipelines:

```bash
# Generate a backup archive, send it to a remote target, 
# while simultaneously calculating its checksum and writing to a local backup:
tar -czf - /data | tee \
    >(sha256sum > /backups/archive.tar.gz.sha256) \
    >(ssh storage "cat > /vault/archive.tar.gz") \
    > /backups/archive.tar.gz
```

#### Hazards and Limitations of Process Substitution
* **Asynchronous Execution**: Subshells spawned via process substitution execute concurrently with the main command. The main command's exit code does **not** reflect failures inside `<(cmd)` or `>(cmd)`.
* **Non-Seekable Streams**: Because `/dev/fd/63` is a FIFO (pipe), programs that rely on `lseek(2)` system calls (such as random-access databases or video indexers) fail when passed a process substitution path.

---

## 5. Duplicating, Redirecting, and Closing Communication Channels

Every shell redirection operator translates directly to POSIX system calls that manipulate file descriptors.

```
       POSIX System Calls:
       
       dup(oldfd)
       └── Scans descriptor array for lowest available index;
           duplicates pointer.
       
       dup2(oldfd, newfd)
       └── Atomically closes newfd (if open), then points
           newfd to the open file description of oldfd.
       
       close(fd)
       └── Removes pointer from array, decrements refcount in
           struct file. If refcount == 0, resource is released.
```

### POSIX System Call Mechanics

Underneath the hood, the shell implements redirections using three core C system calls:

```c
#include <unistd.h>
#include <fcntl.h>

// Duplicate an open descriptor to a specific target number:
int dup2(int oldfd, int newfd);

// Atomically duplicate with flags (e.g., close-on-exec):
int dup3(int oldfd, int newfd, int flags);

// Close an open descriptor:
int close(int fd);
```

When you write `2>&1`, the shell executes:
```c
dup2(1, 2); // Closes FD 2, sets FD 2 to mirror the target of FD 1
```

When you write `exec 3<&-`, the shell executes:
```c
close(3);   // Releases descriptor 3 from the process fdtable
```

### Comprehensive Descriptor Manipulation Reference

| Bash Shell Syntax | Kernel Action (`C` System Call Equivalent) | Operational Purpose |
|---|---|---|
| `exec [n]<&[m]` | `dup2(m, n);` | Duplicate input descriptor `m` onto `n` |
| `exec [n]>&[m]` | `dup2(m, n);` | Duplicate output descriptor `m` onto `n` |
| `exec [n]<&-` | `close(n);` | Close input descriptor `n` |
| `exec [n]>&-` | `close(n);` | Close output descriptor `n` |
| `exec <&-` | `close(0);` | Close standard input (`stdin`) |
| `exec >&-` | `close(1);` | Close standard output (`stdout`) |

### Descriptor Save, Redirect, and Restore Workflow

When writing complex bash automation, you frequently need to divert all script output into a log file temporarily, and later restore output back to the interactive terminal:

```bash
#!/usr/bin/env bash

# STEP 1: Save copies of original stdout (1) and stderr (2) to FDs 6 and 7:
exec 6>&1
exec 7>&2

# STEP 2: Redirect all script output to an installation log:
exec > /var/log/install.log 2>&1

echo "Beginning background installation..."
make install

# STEP 3: Temporarily print an urgent message to the original terminal display:
echo "ATTENTION: Prompting for administrative input on terminal!" >&6

# STEP 4: Restore stdout and stderr to their original terminal streams:
exec 1>&6
exec 2>&7

# STEP 5: Close the backup descriptors:
exec 6>&-
exec 7>&-

echo "Terminal output successfully restored."
```

#### Visualizing the Descriptor Table Lifecycle

```
Phase 1: Initial State
[0] ──> PTY In
[1] ──> PTY Out
[2] ──> PTY Out

Phase 2: Save State (6>&1, 7>&2)
[0] ──> PTY In
[1] ──> PTY Out
[2] ──> PTY Out
[6] ──> PTY Out (Duplicate of 1)
[7] ──> PTY Out (Duplicate of 2)

Phase 3: Redirect (> /var/log/install.log 2>&1)
[0] ──> PTY In
[1] ──> /var/log/install.log
[2] ──> /var/log/install.log
[6] ──> PTY Out
[7] ──> PTY Out

Phase 4: Restore (1>&6, 2>&7) and Cleanup (6>&-, 7>&-)
[0] ──> PTY In
[1] ──> PTY Out
[2] ──> PTY Out
[6] ──> [CLOSED]
[7] ──> [CLOSED]
```

### File Descriptor Leaks and `FD_CLOEXEC`

When a shell process forks a child and runs an external command (`execve(2)`), the child process **inherits all open file descriptors by default**.

```
Parent Shell (PID 1000)
 ├── FD 3 ──> /var/run/cluster.lock (Lock Held)
 │
 └── fork() + execve("/usr/bin/python3 worker.py")
      └── Child Process (PID 1001)
           └── Inherits FD 3 pointing to the lock!
```

If the child process crashes, hangs, or spawns long-running background tasks, it keeps FD 3 open. As a result:
* Storage locks cannot be released.
* Filesystem unmount operations fail with `EBUSY` (device or resource busy).
* Network sockets remain bound, preventing daemons from restarting.

#### The Close-On-Exec Flag (`FD_CLOEXEC`)
To prevent descriptor leaks across child processes, the kernel provides the `FD_CLOEXEC` flag:

```c
// Setting FD_CLOEXEC via fcntl(2) in C:
int flags = fcntl(fd, F_GETFD);
fcntl(fd, F_SETFD, flags | FD_CLOEXEC);
```

* **Bash Behavior**: When Bash dynamically allocates file descriptors using `{var}>filename` syntax, it automatically sets the `FD_CLOEXEC` attribute on those descriptors, preventing them from leaking to external commands.
* **Manual Cleanup**: When using static descriptors ($3$ through $9$), you must explicitly close them before invoking long-running services:
```bash
# Launch a background service without leaking FD 3:
my_daemon 3>&- &
```

---

## 6. Practical Laboratory: Low-Level Tracing and Diagnostic Patterns

### Exercise 1: Tracing Descriptors in Real Time with `strace`

Observe how the Linux kernel sets up redirection using the `strace` utility:

```bash
$ strace -f -e trace=openat,dup2,close,write bash -c 'ls -la /nonexistent > /tmp/out.log 2>&1'
```

**Annotated System Call Trace**:
```c
// 1. Shell opens the redirection target file:
openat(AT_FDCWD, "/tmp/out.log", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3

// 2. Shell duplicates target FD (3) onto stdout (1):
dup2(3, 1)                              = 1

// 3. Shell closes the redundant file descriptor:
close(3)                                = 0

// 4. Shell duplicates stdout (1) onto stderr (2) for '2>&1':
dup2(1, 2)                              = 2

// 5. The command execution attempts to access directory, fails, and writes error:
write(2, "ls: cannot access '/nonexistent'"..., 48) = 48
```

### Exercise 2: Auditing File Descriptors of Running Daemons

Locate and inspect all open descriptors for a running service (e.g., `nginx`):

```bash
# Find master worker PID:
PID=$(pgrep -f "nginx: master" | head -n1)

# List all open descriptors, their targets, and types:
sudo lsof -a -p "$PID" -d ^txt,^mem,^cwd,^rtd

# Output example:
# COMMAND  PID USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
# nginx   1234 root    0u   CHR                1,3      0t0       4 /dev/null
# nginx   1234 root    1u   CHR                1,3      0t0       4 /dev/null
# nginx   1234 root    2w   REG              259,2      450  131102 /var/log/nginx/error.log
# nginx   1234 root    6u  IPv4              28911      0t0     TCP *:80 (LISTEN)
# nginx   1234 root    7u  IPv6              28912      0t0     TCP *:80 (LISTEN)
```

### Exercise 3: Creating a Tee Pipeline Without Using `tee`

Implement the functionality of `tee` (splitting a stream to disk and display) using only custom file descriptors and process substitution:

```bash
#!/usr/bin/env bash

# Open process substitution writing directly to the log file on dynamic FD:
exec {TEE_FD}> >(cat >> /var/log/app_stream.log)

# Pipe data simultaneously to terminal (FD 1) and our custom FD:
while read -r line; do
    echo "INSPECT: $line"            # Terminal display
    echo "$line" >&"$TEE_FD"         # Log file stream
done < <(dmesg -w)

exec {TEE_FD}>&-
```

### Exercise 4: Network Socket I/O via Virtual Descriptors in Bash

When compiled with `--enable-net-redirections`, Bash supports dynamic network sockets directly via `/dev/tcp` and `/dev/udp`:

```bash
#!/usr/bin/env bash

# Open bidirectional socket to HTTP server on FD 3:
exec 3<> /dev/tcp/example.com/80

# Send RFC-compliant HTTP GET request:
printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" >&3

# Read and print the raw HTTP response headers and body:
while read -r -u 3 response_line; do
    echo "$response_line"
done

# Close the socket descriptor:
exec 3<&-
```

---

## 7. Comprehensive Redirection Reference Matrix

Use this table as an operational cheat sheet for designing bulletproof shell redirections:

| Syntactic Form | Source Descriptor | Target Mechanism | Kernel Syscall Behavior | POSIX Standard |
|---|---|---|---|---|
| `cmd < file` | `0` | File Path | `open(file, O_RDONLY)` $\to$ `dup2(fd, 0)` | Yes |
| `cmd > file` | `1` | File Path | `open(..., O_WRONLY\|O_CREAT\|O_TRUNC)` $\to$ `dup2(fd, 1)` | Yes |
| `cmd >> file` | `1` | File Path | `open(..., O_WRONLY\|O_CREAT\|O_APPEND)` $\to$ `dup2(fd, 1)` | Yes |
| `cmd 2> file` | `2` | File Path | `open(...)` $\to$ `dup2(fd, 2)` | Yes |
| `cmd 2>&1` | `2` | Descriptor `1` | `dup2(1, 2)` | Yes |
| `cmd 1>&2` | `1` | Descriptor `2` | `dup2(2, 1)` | Yes |
| `cmd &> file` | `1` and `2` | File Path | Shorthand for `> file 2>&1` | No (Bash/Zsh) |
| `cmd &>> file`| `1` and `2` | File Path | Shorthand for `>> file 2>&1` | No (Bash 4.0+) |
| `cmd >\| file`| `1` | File Path | Overrides `set -C` (`noclobber`) | Yes |
| `cmd 3>&-` | `3` | N/A | `close(3)` | Yes |
| `cmd {v}> file`| Auto ($\ge 10$) | File Path | Allocates lowest integer $\ge 10$, sets `FD_CLOEXEC` | No (Bash 4.1+) |
| `cmd <<< "$s"`| `0` | Memory String | Opens anonymous pipe/tmpfile, writes string, sets to FD 0 | No (Bash/Ksh) |
| `cmd <(other)`| Command Path | Pipe / FIFO | Forks subshell to pipe, substitutes `/dev/fd/N` | No (Bash/Ksh/Zsh) |
| `cmd >(other)`| Command Path | Pipe / FIFO | Forks subshell to pipe, substitutes `/dev/fd/N` | No (Bash/Ksh/Zsh) |