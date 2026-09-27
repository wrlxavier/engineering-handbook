### 09. Service Administration with systemd

As modern Linux environments evolved beyond synchronous shell-scripted initialization models, the demand for predictable state convergence, fine-grained process isolation, declarative dependency modeling, and unified telemetry led to the development of `systemd`. Today, `systemd` acts as the foundational system and service manager across major enterprise Linux distributions.

Beyond serving as the initial userspace process ($\text{PID } 1$), `systemd` provides an integrated platform architecture: it enforces process boundaries through the Linux kernel's Control Groups version 2 (`cgroups v2`) hierarchy, mediates inter-process state changes over the D-Bus system bus, supervises daemon lifecycles with deterministic recovery guarantees, schedules asynchronous and calendar events via kernel timers, and captures high-fidelity structured logs via binary journal streams.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Linux Userspace Boundary                        │
│                                                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                      systemd (PID 1)                           │   │
│   │  - Dependency Engine (Directed Acyclic Graph / Transaction)    │   │
│   │  - State Machine & Service Supervisor                          │   │
│   │  - D-Bus Interface: `org.freedesktop.systemd1`                │   │
│   │  - Socket Activation / Cgroups v2 Resource Accounting          │   │
│   └──────┬─────────────────────┬──────────────────────┬────────────┘   │
│          │                     │                      │                │
│          ▼                     ▼                      ▼                │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐         │
│   │ Target Units │      │ Service Units│      │ Timer Units  │         │
│   │ (.target)    │      │ (.service)   │      │ (.timer)     │         │
│   └──────┬───────┘      └──────┬───────┘      └──────┬───────┘         │
│          │                     │                     │                 │
│          └─────────────────┐   │   ┌─────────────────┘                 │
│                            ▼   ▼   ▼                                   │
│                 ┌─────────────────────────────┐                        │
│                 │   systemd-journald.service  │                        │
│                 │   - Structured binary logs  │                        │
│                 │   - `/dev/log`, `/dev/kmsg` │                        │
│                 └──────────────┬──────────────┘                        │
│                                │                                       │
└────────────────────────────────┼───────────────────────────────────────┘
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│                          Linux Kernel Space                            │
│  - cgroup2 filesystem (`/sys/fs/cgroup`)                               │
│  - Kernel Event Subsystem (epoll, signalfd, inotify, timerfd)          │
│  - Virtual Filesystem (`/proc`, `/sys`, `/dev`, `/run`)                │
└────────────────────────────────────────────────────────────────────────┘
```

---

#### 1. Initialization Architecture and Daemon Management via `systemctl`

##### `systemd` Architecture and PID 1 Core Mechanics
When the kernel finishes early userspace hardware initialization and invokes `/lib/systemd/systemd`, the process assumes Process ID 1 ($\text{PID } 1$). In contrast to legacy SysV-init—which treated daemons as unmonitored background scripts detached from their launcher—`systemd` maintains active supervision over all child, grandchild, and orphaned processes throughout their entire lifecycle.

###### 1. Cgroups v2 Unified Tree Management
`systemd` relies intrinsically on Linux Control Groups (`cgroups v2`) located at `/sys/fs/cgroup`. Every unit managed by `systemd` is assigned a distinct slice and cgroup path:
* **Slices (`.slice`)**: Resource partitions allocating fractions of CPU, memory, and I/O (e.g., `system.slice` for core system daemons, `user.slice` for interactive user sessions).
* **Scopes (`.scope`)**: Groups of externally created processes tracked by `systemd` but not launched directly by it (e.g., user login sessions, container runtimes).
* **Services (`.service`)**: Workloads executed, supervised, and isolated directly by PID 1.

Because the kernel cgroups v2 hierarchy is strictly hierarchical, a daemon cannot escape supervision by double-forking (the classic UNIX daemonizing trick). When a process forks, its children remain trapped within the unit's dedicated cgroup sub-tree until the cgroup is explicitly destroyed by PID 1 upon unit termination.

###### 2. D-Bus Messaging Bus Interface
PID 1 exposes an extensive IPC API on the system message bus (`/run/dbus/system_bus_socket`) and an internal private UNIX socket (`/run/systemd/private`) using the D-Bus protocol under the destination interface `org.freedesktop.systemd1`. CLI utilities like `systemctl` do not manipulate files directly during execution; they construct structured D-Bus method calls (e.g., `StartUnit()`, `StopUnit()`, `GetUnit()`), which PID 1 validates against Polkit authorization rules before scheduling the requested transaction.

###### 3. Transaction Engine and Dependency Resolution
When an administrative action is requested, `systemd` does not immediately execute commands. Instead, it generates a **Transaction Queue**. It constructs a directed acyclic graph (DAG) representing the requested unit alongside its dependencies, checks for circular ordering conflicts, verifies prerequisite conditions, and executes jobs concurrently using non-blocking asynchronous event loops driven by `epoll(7)`.

###### 4. Unit States and Transitions
At any point in time, a unit exists in well-defined internal states:
* **Load State**: Reflects whether the unit configuration file was located, parsed, and validated (`loaded`, `not-found`, `bad-setting`, `error`, `masked`).
* **Active State**: High-level execution status (`active`, `reloading`, `inactive`, `failed`, `activating`, `deactivating`).
* **Sub State**: Detailed, unit-type-specific low-level state machine phase (e.g., for a `.service`, this includes `dead`, `start-pre`, `running`, `exited`, `stop-sigterm`, `failed`).

```
                    ┌──────────────┐
                    │  not-found   │
                    └──────┬───────┘
                           │ (Unit configuration read)
                           ▼
                    ┌──────────────┐
       ┌───────────>│   inactive   │<────────────┐
       │            └──────┬───────┘             │
       │                   │ (Job queued: start) │
       │                   ▼                     │
       │            ┌──────────────┐             │
       │            │  activating  │             │
       │            └──────┬───────┘             │
(Clean │                   │ (Process running    │ (Execution
 exit) │                   │  or readiness ACK)  │  aborted)
       │                   ▼                     │
       │            ┌──────────────┐             │
       │            │    active    │             │
       │            └──────┬───────┘             │
       │                   │                     │
       │         ┌─────────┴─────────┐           │
       │         ▼                   ▼           │
       │  ┌────────────┐      ┌────────────┐     │
       │  │ reloading  │      │deactivating│     │
       │  └─────┬──────┘      └──────┬─────┘     │
       │        │ (Reload ACK)       │           │
       │        └──────────┐         │           │
       │                   ▼         ▼           │
       │            ┌──────────────────────┐     │
       └────────────┤   failed / dead      ├─────┘
                    └──────────────────────┘
```

---

##### The System Unit Hierarchy and Precedence Paths
`systemd` dynamically parses unit files across several primary filesystem directories. When duplicate unit names exist across these directories, `systemd` enforces a strict precedence hierarchy where higher-priority layers completely override or extend lower-priority layers.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Unit Precedence Layers                          │
├──────────────────────────┬─────────────────────────────────────────────┤
│ `/etc/systemd/system/`   │ Local administrative configurations and     │
│                          │ manual customizations (HIGHEST PRECEDENCE). │
├──────────────────────────┼─────────────────────────────────────────────┤
│ `/run/systemd/system/`   │ Runtime, volatile configurations generated  │
│                          │ dynamically at boot (e.g., by generators).  │
├──────────────────────────┼─────────────────────────────────────────────┤
│ `/usr/lib/systemd/system/`│ Vendor/Distribution package defaults        │
│ (or `/lib/systemd/system`)│ (LOWEST PRECEDENCE - Never edit directly!). │
└──────────────────────────┴─────────────────────────────────────────────┘
```

###### The Masking Mechanism
Administrative masking prevents a service from running under any circumstance—whether invoked directly, triggered by a socket, or requested as an upstream dependency of another running service.
Masking is implemented at the VFS layer by creating a symbolic link pointing to the null character device:

$$\text{Symlink: } \texttt{/etc/systemd/system/bad-service.service} \longrightarrow \texttt{/dev/null}$$

When PID 1 resolves unit lookups, encountering a symlink to `/dev/null` forces the Load State to `masked`, halting all transaction planning for that unit.

```bash
# Mask a unit to permanently prevent execution:
sudo systemctl mask redis-server.service

# Verify the symlink generated in the administrative directory:
ls -l /etc/systemd/system/redis-server.service
# lrwxrwxrwx 1 root root 9 Sep 27 10:00 /etc/systemd/system/redis-server.service -> /dev/null

# Attempting to start a masked unit fails immediately at the D-Bus interface:
sudo systemctl start redis-server.service
# Failed to start redis-server.service: Unit redis-server.service is masked.

# Return the unit to normal operational availability:
sudo systemctl unmask redis-server.service
```

---

##### Core Daemon Management Workflows with `systemctl`

The `systemctl` utility is the primary CLI interface for inspecting and controlling system states.

```bash
# 1. Immediate Execution Control:
sudo systemctl start nginx.service            # Instructs PID 1 to start the service
sudo systemctl stop nginx.service             # Sends SIGTERM (then SIGKILL if timeout expires)
sudo systemctl restart nginx.service          # Unconditionally terminates, then starts unit
sudo systemctl try-restart nginx.service      # Restarts ONLY IF unit is currently active
sudo systemctl reload nginx.service           # Triggers in-process config reload without termination
sudo systemctl reload-or-restart nginx.service# Reloads if supported; restarts if not

# 2. Lifecycle Interrogation:
systemctl status nginx.service                # High-level overview: load, active, cgroup, logs
systemctl is-active nginx.service             # Returns exit code 0 if active, non-zero otherwise
systemctl is-enabled nginx.service            # Checks if target symlinks exist in .wants/.requires
systemctl is-failed nginx.service             # Returns 0 if unit halted in a failed substate

# 3. Boot State Persistence:
sudo systemctl enable nginx.service           # Creates activation symlinks in default target
sudo systemctl disable nginx.service          # Removes activation symlinks; does not stop running unit
sudo systemctl enable --now nginx.service     # Atomically creates symlinks AND starts daemon
sudo systemctl disable --now nginx.service    # Atomically stops daemon AND removes symlinks
sudo systemctl reenable nginx.service         # Disables and enables, restoring package default symlinks
```

###### Status Output Breakdown
Executing `systemctl status` provides a multi-dimensional diagnostic snapshot:

```text
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
    Drop-In: /etc/systemd/system/nginx.service.d
             └─override.conf
     Active: active (running) since Sun 2026-09-27 10:14:22 UTC; 2h 10min ago
       Docs: man:nginx(8)
    Process: 4120 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 4122 (nginx)
      Tasks: 5 (limit: 9348)
     Memory: 28.4M (peak: 32.1M, swap: 0B)
        CPU: 1.842s
     CGroup: /system.slice/nginx.service
             ├─4122 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─4123 "nginx: worker process"
             ├─4124 "nginx: worker process"
             ├─4125 "nginx: worker process"
             └─4126 "nginx: worker process"
```

* **Loaded Line**: Shows load state (`loaded`), path to unit definition file, persistence configuration (`enabled`), and distribution preset policy (`preset: enabled`).
* **Drop-In Line**: Displays any modular configuration snippet directories actively modifying the base unit.
* **Active Line**: Active state (`active`), sub-state (`running`), and uptime duration.
* **Process Tracking**: Traces pre-execution verification commands (`ExecStartPre`) and exit codes.
* **Main PID**: Tracks the designated master process identifier.
* **Tasks & Resource Accounting**: Quantifies operating system threads, kernel cgroup memory footprint (resident, peak, swap), and cumulative CPU consumption time.
* **CGroup Hierarchy**: Lists every active process running under this unit's cgroup tree.

---

##### Daemon Configuration Introspection and Reloading

Whenever unit files on disk are modified, created, or unlinked, PID 1's in-memory dependency graph becomes out of sync with the filesystem.

###### Daemon-Reload vs. Daemon-Reexec
* **`systemctl daemon-reload`**: Scans all unit directories, regenerates dynamic generator scripts, reparses configuration files, reconciles dependency graphs, and rebuilds internal transaction trees. It does not disrupt running service processes.
* **`systemctl daemon-reexec`**: Re-executes the PID 1 binary itself. The current state is serialized into an in-memory file descriptor, the new `systemd` executable image replaces the old one via `execve(2)`, and state is deserialized. This is essential when updating `systemd` package versions during operating system upgrades.

###### Introspecting Unit Properties
```bash
# Print the merged unit configuration file with all drop-ins included:
systemctl cat nginx.service

# Interrogate low-level D-Bus properties directly from PID 1 memory:
systemctl show nginx.service

# Query specific property variables programmatically:
systemctl show nginx.service --property=MainPID,ActiveState,SubState,MemoryCurrent
# MainPID=4122
# ActiveState=active
# SubState=running
# MemoryCurrent=29777920
```

---

##### Dynamic Overrides via Drop-in Snippets
Directly editing package-managed unit files located in `/usr/lib/systemd/system/` introduces configuration drift: package managers will overwrite manual edits during software updates.

Instead, administrators implement the **Drop-in Directory Architecture**:

```
/etc/systemd/system/<unit-name>.service.d/
├── 10-resources.conf
└── 20-networking.conf
```

`systemd` parses every file ending in `.conf` inside this directory in strict lexicographical order, applying overrides on top of the vendor-supplied unit file.

###### Using the Interactive Editor
The standard method to configure drop-ins is `systemctl edit`:

```bash
# Opens an interactive editor, generating /etc/systemd/system/<unit>.service.d/override.conf:
sudo systemctl edit nginx.service

# Alternatively, copy the ENTIRE unit file to /etc/systemd/system/ for total replacement:
sudo systemctl edit --full nginx.service
```

###### Drop-In Construction Rules
When overriding array-like or list directives (such as `ExecStart=`, `ExecStartPre=`, or `Environment=`), appending a new assignment extends the list. To reset or replace the list, you must first pass an empty assignment:

```ini
# /etc/systemd/system/custom-service.service.d/override.conf
[Service]
# Clear existing ExecStart directives:
ExecStart=
# Assign the new execution instruction:
ExecStart=/usr/local/bin/custom-service --daemon --port 8080
```

---

#### 2. Creating Custom Unit Files (`.service`) and Configuring Dependency Management

A `systemd` service unit file is a declarative UTF-8 plain-text configuration file organized into INI-style sections:
1. `[Unit]`: Metadata, operational descriptions, documentation references, and dependency definitions.
2. `[Service]`: Execution logic, environment controls, process lifecycle typing, restart behavior, sandboxing policies, and cgroup resource limits.
3. `[Install]`: Relationship targets that dictate how and where the unit binds when registered via `systemctl enable`.

```ini
# /etc/systemd/system/production-api.service
[Unit]
Description=Enterprise Production API Gateway
Documentation=https://docs.internal.net/api/
After=network-online.target firewalld.service
Wants=network-online.target
Requires=redis.service
PartOf=redis.service
Conflicts=maintenance.target

[Service]
Type=notify
User=appuser
Group=appuser
WorkingDirectory=/opt/production-api
EnvironmentFile=/etc/default/production-api
ExecStartPre=/opt/production-api/bin/preflight-check.sh
ExecStart=/opt/production-api/bin/api-server --config=/etc/production-api/config.yaml
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
StartLimitIntervalSec=60s
StartLimitBurst=3

# Resource Sandboxing & Security Hardening
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectControlGroups=yes
NoNewPrivileges=yes
RestrictNamespaces=yes
MemoryMax=1G
CPUQuota=200%

[Install]
WantedBy=multi-user.target
```

---

##### Dependency Modeling and Directed Acyclic Graph (DAG) Execution

A critical distinction in `systemd` architecture is the strict decoupling of **Requirement Dependencies** from **Ordering Dependencies**. Declaring that Service B requires Service A does *not* automatically mean Service A starts before Service B.

```
       Requirement Plane                    Ordering Plane
       
      ┌─────────────────┐                 ┌─────────────────┐
      │  Service B      │                 │  Service A      │
      └────────┬────────┘                 └────────┬────────┘
               │                                   │
               │ Requires=                         │ After=
               ▼                                   ▼
      ┌─────────────────┐                 ┌─────────────────┐
      │  Service A      │                 │  Service B      │
      └─────────────────┘                 └─────────────────┘
  (Both start IN PARALLEL             (Service B waits until
   unless Ordering is set!)            Service A signals ready)
```

###### 1. Requirement Dependencies
* **`Requires=`**: Strong hard dependency. When this unit starts, all units listed in `Requires=` are activated concurrently. If any required unit fails to activate, this unit aborts. Conversely, if a required unit crashes or is stopped, this unit is stopped immediately.
* **`Wants=`**: Soft dependency (recommended default). `systemd` attempts to start the listed units in parallel. However, if a wanted unit fails to activate or crashes, this unit continues execution unaffected.
* **`BindsTo=`**: Extreme hard lifecycle coupling. Similar to `Requires=`, but if the bound unit transitions to an inactive state (e.g., a physical network interface unplugged, or an unmounted block device), this unit is pulled down instantly.
* **`PartOf=`**: Administrative lifecycle grouping. Stopping or restarting the target unit cascades to this unit, but starting the target unit does not automatically start this unit.
* **`Requisite=`**: Immediate state validation. If the unit listed in `Requisite=` is not already fully active and running when this unit is called, this unit fails instantly without attempting to start the prerequisite.

###### 2. Ordering Dependencies
* **`After=`**: Guarantees that if both units are scheduled in the same transaction, this unit delays its execution until the unit listed in `After=` has finished starting (or reached steady-state active readiness).
* **`Before=`**: Guarantees that this unit finishes its startup phase before the unit listed in `Before=` begins initialization.

$$\mathbf{Critical\ Rule:}\quad \text{Dependency} \neq \text{Ordering}$$

If an administrator configures `Requires=redis.service` without adding `After=redis.service`, PID 1 spawns both services simultaneously. If the primary service attempts to connect to Redis during initialization before Redis has bound to its listening socket, the dependent service will fail. Always pair `Requires=` and `Wants=` with corresponding `After=` directives when sequential startup is required.

###### 3. Conflicts and Conditions
* **`Conflicts=`**: Negative dependency. Declares mutual exclusivity. If Unit A has `Conflicts=Unit B`, starting Unit A forces Unit B to terminate. If both are queued simultaneously in a transaction, `systemd` rejects the transaction with an ordering/conflict cycle error.
* **`ConditionPathExists=`**: Evaluates whether a file or directory exists prior to activation. If the check returns false, unit activation is cleanly skipped without entering a `failed` state.
* **`AssertPathExists=`**: Similar to conditions, but if the condition fails, the unit aborts and marks the transaction as an operational error.

---

##### Service Process Lifecycles and `Type=` Directives

The `Type=` directive tells `systemd` how to interpret process execution phases, detect when a service has reached an operational active state, and wire process tracking logic.

```
┌──────────────┬───────────────────────────────┬───────────────────────────────┐
│ `Type=`      │ Active Readiness Indicator    │ Primary Production Use Case   │
├──────────────┼───────────────────────────────┼───────────────────────────────┤
│ `simple`     │ `fork()`/`execve()` of binary │ Standard single foreground    │
│              │ returns successfully.         │ processes (Node, Go, Python). │
├──────────────┼───────────────────────────────┼───────────────────────────────┤
│ `exec`       │ Binary is read, mapped, and   │ Foreground services where     │
│              │ begins execution in userspace.│ missing binaries must halt DAG│
├──────────────┼───────────────────────────────┼───────────────────────────────┤
│ `forking`    │ Parent exits; child process   │ Traditional SysV C daemons   │
│              │ remains (`PIDFile=` tracked). │ that fork to background.      │
├──────────────┼───────────────────────────────┼───────────────────────────────┤
│ `oneshot`    │ `ExecStart` process runs to   │ Setup scripts, migrations,    │
│              │ completion and exits ($0$).   │ partition prep, batch steps.  │
├──────────────┼───────────────────────────────┼───────────────────────────────┤
│ `dbus`       │ Daemon acquires reserved      │ Core services integrating with│
│              │ name on D-Bus system bus.     │ D-Bus (NetworkManager).       │
├──────────────┼───────────────────────────────┼───────────────────────────────┤
│ `notify`     │ Daemon sends explicit packet  │ Robust enterprise daemons     │
│              │ via `$NOTIFY_SOCKET`.         │ with internal bootup phases.  │
├──────────────┼───────────────────────────────┼───────────────────────────────┤
│ `idle`       │ Delays launch until all jobs  │ Console status scripts to     │
│              │ have finished dispatching.    │ avoid terminal log clobber.   │
└──────────────┴───────────────────────────────┴───────────────────────────────┘
```

###### Deep Dive: The `Type=notify` Protocol
`Type=notify` provides the most reliable startup orchestration in Linux. When using `Type=simple`, downstream units scheduled `After=` the service begin executing immediately when the binary starts, before internal components (database connections, listening sockets, cache warming) are initialized.

`Type=notify` resolves this race condition using a dedicated UNIX domain datagram socket:
1. PID 1 creates an ephemeral UNIX socket and injects its path into the child process via the environment variable `$NOTIFY_SOCKET`.
2. The service process performs internal initialization (e.g., loads large models, connects to database pools).
3. Once fully operational, the process issues the `sd_notify(3)` library call, transmitting the datagram payload `READY=1\n` across the socket.
4. PID 1 receives this notification, updates the unit's sub-state to `running`, marks it `active`, and triggers the execution of dependent downstream units.

```c
/* Implementation of systemd notification protocol in C */
#include <systemd/sd-daemon.h>

int main(int argc, char *argv[]) {
    // Perform complex daemon initialization routines...
    setup_database_connection();
    bind_listening_sockets();

    // Signal readiness to systemd:
    sd_notify(0, "READY=1\nSTATUS=Initialized cache and listening on 0.0.0.0:8080");

    // Enter primary event loop:
    run_server_event_loop();

    // Signal graceful termination:
    sd_notify(0, "STOPPING=1");
    return 0;
}
```

```bash
# In shell scripts, the sd_notify protocol can be sent via systemd-notify:
systemd-notify --ready --status="Processing worker queues"
```

---

##### Restart Policies and Failure Recovery
A production service must recover from unexpected crashes without entering infinite loop storms that consume available CPU cycles.

```ini
[Service]
# Restart triggers:
# 'no'         : Never restart.
# 'always'     : Restart regardless of exit status (clean exit 0, signals, core dumps).
# 'on-success' : Restart only when process exits cleanly with status 0.
# 'on-failure' : Restart if exit code is non-zero, killed by signal, or hit watchdog timeout.
# 'on-abort'   : Restart only if killed by an unhandled signal (SIGABRT, SIGSEGV).
Restart=on-failure

# Delay introduced before attempting process resurrection:
RestartSec=3s

# Burst Protection Mechanics:
# If the service restarts more than 'StartLimitBurst' times within 
# 'StartLimitIntervalSec', PID 1 halts restart attempts and locks the unit in 'failed'.
StartLimitIntervalSec=120s
StartLimitBurst=5
```

---

##### Security Sandboxing and Resource Isolation Directives
`systemd` provides an extensive suite of process sandboxing directives that leverage Linux namespaces, seccomp filters, and cgroups v2 directly from unit configurations without requiring external container runtimes.

###### 1. Filesystem Namespace Isolation
* **`ProtectSystem=strict`**: Mounts the entire VFS hierarchy (`/usr`, `/boot`, `/etc`, `/root`) strictly read-only for processes under this service. Only explicitly defined paths via `ReadWritePaths=` remain writable.
* **`ProtectHome=yes`**: Makes `/home`, `/root`, and `/run/user` completely inaccessible (empty directory mount). Setting this to `read-only` allows read-only visibility.
* **`PrivateTmp=yes`**: Unshares the mount namespace and provisions a fresh, isolated `tmpfs` file system for `/tmp` and `/var/tmp`. The service cannot see or alter temporary files generated by the host or other daemons, preventing symlink attacks.
* **`ReadOnlyPaths=/etc/app/`**: Selectively enforces read-only access on specific directory branches.
* **`ReadWritePaths=/var/log/app/ /var/lib/app/`**: Whitelists isolated directories where write access is permitted under `ProtectSystem=strict`.

###### 2. Privilege Dropping and Credentials
* **`User=` / `Group=`**: Drops root privileges immediately before executing the binary image, running the workload as the specified unprivileged UID/GID.
* **`DynamicUser=yes`**: Allocates an ephemeral, transient UID/GID from the dynamic allocation range ($61184\text{--}65519$) at startup. When the service stops, the dynamic user is destroyed, preventing persistent ownership vulnerabilities on disk.
* **`NoNewPrivileges=yes`**: Sets the kernel `PR_SET_NO_NEW_PRIVS` flag via `prctl(2)`. Ensures neither the service nor any child processes can gain privileges via `setuid`, `setgid`, or file system capabilities (rendering tools like `sudo` or SUID binaries ineffective).

###### 3. Kernel and Hardware Boundary Controls
* **`ProtectKernelTunables=yes`**: Denies modifications to `/proc/sys`, `/sys`, `/proc/sysrq-trigger`, and sysfs hardware control points.
* **`ProtectControlGroups=yes`**: Makes `/sys/fs/cgroup` read-only within the service's mount namespace, preventing malicious workloads from altering their own resource limits.
* **`PrivateDevices=yes`**: Provisions a virtual, isolated `/dev` directory containing only essential pseudo-devices (`/dev/null`, `/dev/zero`, `/dev/random`, `/dev/urandom`), hiding physical storage disks, raw block partitions, and serial interfaces.
* **`PrivateNetwork=yes`**: Unshares the network namespace (`CLONE_NEWNET`), isolating the daemon with only a loopback interface (`lo`) and blocking physical network access.

###### 4. System Call Filtering via Seccomp
* **`SystemCallFilter=@system-service`**: Applies a Berkeley Packet Filter (BPF) program via `seccomp(2)` allowing only system calls typical for network services.
* **`SystemCallFilter=~@privileged @resources @mount`**: Prepending a tilde (`~`) inverts matching logic to blocklisted system call groups. This example rejects calls related to hardware access, mount namespace adjustments, and clock manipulation.
* **`SystemCallArchitectures=native`**: Prohibits secondary sub-architectures (e.g., executing 32-bit x86 system calls on a 64-bit AMD64 architecture), closing ABI exploit paths.

###### 5. Cgroups v2 Resource Limits
* **`MemoryMax=2G`**: Hard physical memory allocation limit. If the cgroup exceeds $2\text{ GiB}$ and cannot reclaim memory, the Out-Of-Memory (OOM) killer selectively terminates processes within this cgroup.
* **`CPUQuota=150%`**: Enforces CFS (Completely Fair Scheduler) runtime band limits. $150\%$ guarantees the process can utilize a maximum of $1.5$ physical CPU cores over the scheduling period.
* **`TasksMax=512`**: Enforces maximum process/thread limit in the cgroup, preventing fork bombs.

---

##### Parameterized (Template) Units

When running multiple instances of identical workloads (e.g., microservice worker nodes, network tunnels, VPN clients), hardcoding distinct unit files causes configuration drift. `systemd` solves this via **Template Units**.

Template units include an `@` character in their filename:

$$\texttt{/etc/systemd/system/worker@.service}$$

When a template is instantiated, an instance identifier is embedded between the `@` sign and the unit suffix:

```bash
sudo systemctl enable --now worker@alpha.service
sudo systemctl enable --now worker@beta.service
```

Inside the template file, `systemd` expands runtime specifiers dynamically:

```ini
# /etc/systemd/system/worker@.service
[Unit]
Description=High-Throughput Processing Worker %I
After=network.target

[Service]
Type=simple
User=app-%i
WorkingDirectory=/var/workers/%i
ExecStart=/usr/local/bin/worker-engine --instance=%i --config=/etc/worker/%i.conf
Restart=on-failure
MemoryMax=512M

[Install]
WantedBy=multi-user.target
```

###### Core Unit Specifiers
* **`%i`**: The unescaped instance string (e.g., `alpha` for `worker@alpha.service`).
* **`%I`**: The unescaped instance string with path components un-mangled.
* **`%n`**: The full unit name (`worker@alpha.service`).
* **`%N`**: The unescaped unit name without the suffix (`worker@alpha`).
* **`%u`**: The configured runtime user name.
* **`%h`**: The home directory of the configured runtime user.

---

#### 3. Modern Periodic Task Scheduling with `systemd.timer`

Historically, periodic task automation in Linux relied on the cron daemon (`crond`). While functional, cron has significant limitations in modern environments:
1. **No Dependency Integration**: Cron runs jobs based solely on clock intervals, with no awareness of network connectivity, filesystem mounts, or dependent services.
2. **Process Escapes and Zombie Tracking**: Cron does not execute tasks within dedicated cgroups. If a cron script forks runaway background tasks, they survive when the main script completes.
3. **Log Fragmentation**: Output defaults to local mail spool delivery (`sendmail`) or unstructured syslog lines, complicating centralized log shipping.
4. **No Catch-Up Guarantees**: If a server is powered off during a scheduled run, standard cron jobs are skipped entirely.

`systemd` replaces cron using dedicated **Timer Units (`.timer`)**. A timer unit schedules and activates its companion service unit (`.service`), running periodic workloads with full cgroup isolation, dependency awareness, and integrated journal logging.

```
Cron Paradigm:
┌───────────┐      fork()      ┌──────────────────────────────┐
│   crond   ├─────────────────>│  Script execution (no cgroup)│
└───────────┘                  └──────────────┬───────────────┘
                                              │ Unstructured stdout
                                              ▼
                                       Local Mail Spool

systemd.timer Paradigm:
┌───────────────┐  Target Activation  ┌──────────────────────────────┐
│ backup.timer  ├────────────────────>│ backup.service               │
└───────────────┘                     │  - cgroup isolation          │
                                      │  - Security sandboxing       │
                                      │  - Dependency checks         │
                                      └──────────────┬───────────────┘
                                                     │ Structured stdout/stderr
                                                     ▼
                                              systemd-journald
```

---

##### Timer Unit Architecture and Companion Pairing

A timer unit binds to a `.service` file sharing the exact same base name by default:

$$\texttt{backup.timer} \quad \Longleftrightarrow \quad \texttt{backup.service}$$

If a custom target service name is required, administrators override this link via the `Unit=` directive inside the `[Timer]` section:

```ini
# /etc/systemd/system/db-backup.timer
[Unit]
Description=Automated Database Backup Schedule
Documentation=https://wiki.internal.net/ops/backup

[Timer]
Unit=database-dump-worker.service
OnCalendar=*-*-* 02:30:00
Persistent=true
RandomizedDelaySec=15m

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/database-dump-worker.service
[Unit]
Description=Database Dump Execution Script
After=network-online.target

[Service]
Type=oneshot
User=postgres
ExecStart=/usr/local/bin/pg_dump_cluster.sh
```

---

##### Monotonic vs. Realtime (Calendar) Timers

`systemd` categorizes scheduling mechanics into two functional paradigms:

###### 1. Monotonic Timers
Monotonic timers evaluate intervals relative to a dynamic system event or hardware tick counter. They do not depend on the system wall-clock (UTC), protecting them against clock drift, Daylight Saving Time (DST) shifts, or NTP slewing adjustments.
* **`OnBootSec=`**: Relative to the moment the kernel booted.
* **`OnStartupSec=`**: Relative to the moment PID 1 initialized userspace.
* **`OnUnitActiveSec=`**: Relative to when the companion service was last brought to an active state.
* **`OnUnitInactiveSec=`**: Relative to when the companion service last exited.

```ini
# Example: Run 15 minutes after system boot, then re-execute every 4 hours:
[Timer]
OnBootSec=15min
OnUnitActiveSec=4h
```

###### 2. Realtime (Calendar) Timers
Calendar timers trigger at specific real-world dates and times using the `OnCalendar=` configuration key.

```text
DayOfWeek Year-Month-Day Hour:Minute:Second
```

```ini
# Syntax Examples:
OnCalendar=Mon..Fri *-*-* 03:00:00           # Every weekday at 03:00 UTC
OnCalendar=*-*-01 00:00:00                   # Midnight on the 1st of every month
OnCalendar=*-*-* 00/2:00:00                  # Every two hours on the hour
OnCalendar=hourly                            # Built-in shortcut: *-*-* *:00:00
OnCalendar=daily                             # Built-in shortcut: *-*-* 00:00:00
OnCalendar=weekly                            # Built-in shortcut: Mon *-*-* 00:00:00
```

###### Validating Calendar Expressions
You can validate complex calendar schedules prior to deployment using `systemd-analyze calendar`:

```bash
$ systemd-analyze calendar "Mon..Fri *-*-* 02/4:30:00"
  Normalized form: Mon..Fri *-*-* 02/4:30:00
    Next elapse: Mon 2026-09-28 02:30:00 UTC
       From now: 14h left
```

---

##### Advanced Scheduling Controls

###### 1. Missed Executions and Catch-Up (`Persistent=`)
When `Persistent=true` is set, `systemd` writes a timestamp to disk whenever the companion service successfully triggers:

$$\text{Timestamp Path: } \texttt{/var/lib/systemd/timers/stamp-<timer\_name>}$$

If the system is offline, suspended, or undergoing maintenance when the timer event was scheduled to fire, `systemd` compares the stamp file's modification time against the current time when it next boots. If the execution window was missed, the companion service is executed immediately.

###### 2. Mitigating Thundering Herds (`RandomizedDelaySec=`)
In high-density environments (e.g., thousands of virtual machines backing up to a shared storage cluster), identical cron schedules can overwhelm storage backends and network switches:

```
Without RandomizedDelaySec:
Time: 02:00:00 ──> [1,000 nodes execute backup simultaneously] ──> Storage I/O Collapse

With RandomizedDelaySec=30m:
Time: 02:00:00 to 02:30:00 ──> [1,000 nodes spread executions across the window]
```

Setting `RandomizedDelaySec=30m` instructs `systemd` to add a pseudorandom delay between $0$ and $30$ minutes to the target execution window. This evenly distributes network and storage load across the fleet.

###### 3. Wakeups and Power Control
* **`AccuracySec=`**: Bundles timer wakeups with adjacent kernel timers within a specified range (defaults to `1min`) to keep the CPU in low-power C-states longer, reducing energy consumption and CPU overhead.
* **`WakeSystem=true`**: If the host platform is suspended (ACPI S3 state), this instructs the hardware Real-Time Clock (RTC) to wake the system to execute the job.

---

##### Managing and Inspecting Active Timers

```bash
# List all active timer units sorted by execution schedule:
systemctl list-timers

# Include dormant, inactive, and disabled timers in the inspection:
systemctl list-timers --all
```

###### Sample Output
```text
NEXT                        LEFT        LAST                        PASSED     UNIT             ACTIVATES
Sun 2026-09-27 12:00:00 UTC 45min left  Sun 2026-09-27 11:00:00 UTC 14min ago  logrotate.timer  logrotate.service
Mon 2026-09-28 00:00:00 UTC 12h left    Sun 2026-09-27 00:00:00 UTC 11h ago    fstrim.timer     fstrim.service
Mon 2026-09-28 02:30:00 UTC 14h left    n/a                         n/a        db-backup.timer  db-backup.service
```

---

#### 4. Reading, Filtering, and Maintaining Binary Logs with `journalctl`

Traditional logging daemons (`syslogd`, `rsyslog`) rely on unstructured ASCII text streams written directly to flat files under `/var/log/`. This approach has significant drawbacks:
* Text logs are prone to corruption during concurrent writes or sudden system crashes.
* Parsing dates, hostnames, and process IDs requires complex regex rules that are vulnerable to log injection attacks.
* High-volume log events can block disk I/O operations.

The `systemd` ecosystem resolves these challenges via `systemd-journald`. The journal ingests structured operational events, enriches them with verified kernel-space metadata, indexes the fields, and stores them in an indexed binary format (`.journal`).

```
                    ┌──────────────────────────────┐
                    │      Kernel Core (printk)    │
                    └──────────────┬───────────────┘
                                   │ /dev/kmsg
                                   ▼
┌──────────────────┐      ┌─────────────────────────┐      ┌──────────────────┐
│ stdout / stderr  ├─────>│                         │<─────┤ Posix syslog(3)  │
│ (from services)  │      │ systemd-journald        │      │ (via `/dev/log`) │
└──────────────────┘      │                         │      └──────────────────┘
                          └────────────┬────────────┘
                                       │ Enriches with metadata:
                                       │   _PID, _UID, _SYSTEMD_CGROUP,
                                       │   _BOOT_ID, _COMM, _HOSTNAME
                                       ▼
                          ┌─────────────────────────┐
                          │ Indexed Binary Storage  │
                          │ `/var/log/journal/`     │
                          └────────────┬────────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │ journalctl CLI Tool     │
                          └─────────────────────────┘
```

---

##### Storage Persistence and Resource Management
The journal daemon runs in one of two modes depending on `/etc/systemd/journald.conf`:
1. **Volatile**: Logs are written to the RAM-backed memory filesystem at `/run/log/journal/` and are cleared on reboot.
2. **Persistent**: Logs are written to non-volatile storage at `/var/log/journal/<machine-id>/` and persist across reboots.

```ini
# /etc/systemd/journald.conf
[Journal]
# Storage allocation mode: 'volatile', 'persistent', 'auto', 'none'
Storage=persistent

# Size capping mechanics:
# Enforces absolute boundary limit on journal disk footprint:
SystemMaxUse=4G

# Minimum space reserved for external filesystem files:
SystemKeepFree=2G

# Maximum size of an individual binary .journal chunk before rotation:
SystemMaxFileSize=256M

# Maximum duration to retain log entries:
MaxRetentionSec=1month

# Enable kernel audit logs and local syslog socket:
ForwardToSyslog=no
Audit=yes
```

---

##### Querying and Filtering with `journalctl`

Because journal logs are indexed on disk by structured fields, filtering does not require piping output to slow text parsers like `grep` or `awk`. Filtering operations are evaluated directly against the indexed binary journal files:

```bash
# 1. Scope Filtering by Unit:
journalctl -u nginx.service                   # All historical logs for a unit
journalctl -u nginx.service -u php-fpm.service# Interleave logs for multiple units concurrently

# 2. Filtering by Time Windows:
journalctl --since "2026-09-27 00:00:00" --until "2026-09-27 12:00:00"
journalctl --since "1 hour ago"
journalctl --since "yesterday" --until "20 min ago"

# 3. Filtering by Boot Sessions:
journalctl --list-boots                       # Enumerate all recorded boot cycles
journalctl -b 0                               # Logs from the CURRENT running boot session
journalctl -b -1                              # Logs from the PREVIOUS boot session
journalctl -b -2 -u nginx.service             # Logs for a specific service two boots ago

# 4. Severity / Priority Filtering (Syslog numeric 0-7 or textual names):
# 0: emerg, 1: alert, 2: crit, 3: err, 4: warning, 5: notice, 6: info, 7: debug
journalctl -p err..emerg                      # Isolate critical errors and failures

# 5. Real-Time Diagnostics:
journalctl -f                                 # Stream live entries (equivalent to tail -f)
journalctl -f -u production-api.service       # Stream live entries for a specific unit
journalctl -n 50 --no-pager                   # Print the last 50 lines without launching less

# 6. Kernel Buffer Inspection:
journalctl -k                                 # Inspect kernel ring buffer (dmesg equivalent)
journalctl -k -b 0                            # Kernel logs strictly from current boot
```

###### Filtering by Structured Metadata Attributes
Every entry captured by `journald` includes verified, tamper-proof metadata fields appended by PID 1 and the kernel. These fields are prefixed with an underscore (`_`):

```bash
# Match process identity:
journalctl _PID=4122

# Match by system user ID:
journalctl _UID=1000

# Match by binary executable path:
journalctl _COMM=nginx

# Match by full cgroups v2 scope:
journalctl _SYSTEMD_CGROUP=/system.slice/nginx.service

# List all unique values recorded for a metadata field across the database:
journalctl -F _COMM
```

---

##### Output Serialization Formats
`journalctl` can serialize structured log data into several output formats for consumption by external collectors, SIEMs, or processing scripts:

```bash
# Standard high-verbosity output showing all metadata fields:
journalctl -u nginx.service -n 1 -o verbose
```

```text
Sun 2026-09-27 10:14:22.102345 UTC [s=a1b2c3d4;i=1a2b;b=4e9...;m=1f2e;t=5d...]
    _BOOT_ID=4e9d7c6b5a4f3e2d1c0b9a8f7e6d5c4b
    _MACHINE_ID=8a7b6c5d4e3f2a1b0c9d8e7f6a5b4c3d
    _HOSTNAME=gateway01.internal
    _TRANSPORT=stdout
    PRIORITY=6
    _UID=0
    _GID=0
    _COMM=nginx
    _EXE=/usr/sbin/nginx
    _CMDLINE=nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
    _CAP_EFFECTIVE=1ffffffffff
    _SYSTEMD_CGROUP=/system.slice/nginx.service
    _SYSTEMD_UNIT=nginx.service
    _SYSTEMD_SLICE=system.slice
    MESSAGE=Configuration file /etc/nginx/nginx.conf test is successful
```

```bash
# Serialize logs as newline-delimited JSON for log forwarders (Fluentbit, Logstash):
journalctl -u nginx.service -n 5 -o json

# Format JSON with human-readable indentation:
journalctl -u nginx.service -n 1 -o json-pretty

# Strip metadata and output raw payloads (useful for parsing):
journalctl -u nginx.service -o cat
```

---

##### Log Maintenance, Vacuuming, and Forward Secure Sealing

###### 1. Disk Space Auditing and Vacuuming
Over time, binary journal logs can consume significant storage. Rather than deleting active `.journal` files directly from disk (which can corrupt index headers), administrators rotate and vacuum logs using `journalctl`:

```bash
# Interrogate the current disk space consumed by the journal database:
journalctl --disk-usage
# Archived and active journals take up 2.4G in the file system.

# Force immediate log file rotation (active chunks marked archived, new chunks opened):
sudo journalctl --rotate

# Retain only the most recent 1 GB of logs, removing the oldest archives:
sudo journalctl --vacuum-size=1G

# Retain only logs from the last 14 days:
sudo journalctl --vacuum-time=14d

# Retain a maximum number of individual archive files:
sudo journalctl --vacuum-files=10
```

###### 2. Cryptographic Tamper-Resistance via Forward Secure Sealing (FSS)
To prevent malicious actors from altering historical log records after compromising a host, `systemd-journald` implements **Forward Secure Sealing (FSS)** using the Bellare-Miner forward-secure signature algorithm.

```
Time Timeline (Epochs):
[Epoch 0: Key K_0] ──> Sealed Journal Entries ──> K_0 permanently destroyed
         │
         ▼
[Epoch 1: Key K_1] ──> Sealed Journal Entries ──> K_1 permanently destroyed
         │
         ▼
[Epoch 2: Key K_2] ──> Active Journal Entries (Current)

If an attacker gains root access at Epoch 2, they obtain ONLY Key K_2.
They CANNOT retroactively modify Epoch 0 or Epoch 1 entries,
as the corresponding mathematical verification keys no longer exist!
```

```bash
# 1. Initialize an FSS key pair:
# Generates a local Verification Key (save this OFFLINE) and an internal sealing key.
sudo journalctl --setup-keys

# Sample terminal output:
# The secret sealing key has been generated and stored in /var/log/journal/.../fss/
# The verification key is:
# 1a2b3c-4d5e6f-7a8b9c.../0-4/14s

# 2. Cryptographically verify the integrity of all journal archives:
sudo journalctl --verify --verify-key=1a2b3c-4d5e6f-7a8b9c.../0-4/14s
# PASS: /var/log/journal/8a7b.../system.journal: Sealing sequence valid.
```

---

#### 5. Practical Laboratories and Production Implementations

##### Lab 1: Deploying a Resilient, Sandboxed Background Microservice

###### Objective
Build a hardened production API service running an isolated Go/Node.js-style binary. The service must run as an unprivileged, dynamic user, drop all non-essential kernel capabilities, implement strict read-only filesystem sandboxing, enforce CPU and memory boundaries via cgroups v2, and communicate its operational readiness using `Type=notify`.

###### Implementation Script
```bash
#!/usr/bin/env bash
set -euo pipefail

APP_DIR="/opt/secure-api"
SERVICE_FILE="/etc/systemd/system/secure-api.service"

echo "[*] Step 1: Creating application directories and mock binary..."
sudo mkdir -p "${APP_DIR}/bin" "${APP_DIR}/data"

# Create a mock daemon using Python that communicates via sd_notify protocol:
sudo bash -c "cat << 'EOF' > ${APP_DIR}/bin/api-server.py
#!/usr/bin/env python3
import socket
import os
import time
import signal
import sys

def notify(status_str):
    sock_path = os.environ.get('NOTIFY_SOCKET')
    if not sock_path:
        return
    if sock_path.startswith('@'):
        sock_path = '\0' + sock_path[1:]
    with socket.socket(socket.AF_UNIX, socket.SOCK_DGRAM) as s:
        s.sendto(status_str.encode(), sock_path)

def sigterm_handler(signum, frame):
    notify('STOPPING=1')
    sys.exit(0)

signal.signal(signal.SIGTERM, sigterm_handler)

# Simulate initial application boot/loading latency:
time.sleep(2)

# Send readiness notification to systemd:
notify('READY=1\nSTATUS=API Server listening on 127.0.0.1:9090')

# Primary operational event loop:
while True:
    time.sleep(1)
EOF"

sudo chmod +x "${APP_DIR}/bin/api-server.py"

echo "[*] Step 2: Generating hardened systemd service unit..."
sudo bash -c "cat << 'EOF' > ${SERVICE_FILE}
[Unit]
Description=Production Hardened Secure API
After=network.target
Wants=network.target

[Service]
Type=notify
WorkingDirectory=${APP_DIR}
ExecStart=/usr/bin/python3 ${APP_DIR}/bin/api-server.py
Restart=on-failure
RestartSec=5s

# Security Sandboxing
DynamicUser=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
NoNewPrivileges=yes
RestrictRealtime=yes
RestrictSUIDSGID=yes
LockPersonality=yes
ReadWritePaths=${APP_DIR}/data

# Cgroups v2 Resource Allocations
MemoryMax=256M
CPUQuota=50%
TasksMax=32

[Install]
WantedBy=multi-user.target
EOF"

echo "[*] Step 3: Validating unit syntax and reloading PID 1..."
sudo systemd-analyze verify "${SERVICE_FILE}"
sudo systemctl daemon-reload

echo "[*] Step 4: Activating service and validating operational state..."
sudo systemctl enable --now secure-api.service

sleep 3
systemctl status secure-api.service --no-pager

echo -e "\n[*] Step 5: Validating cgroups v2 resource boundaries..."
systemctl show secure-api.service -p MemoryCurrent,CPUUsageNSec,TasksCurrent

echo -e "\n[*] Cleaning up lab assets..."
sudo systemctl stop secure-api.service
sudo systemctl disable secure-api.service
sudo rm -rf "${SERVICE_FILE}" "${APP_DIR}"
sudo systemctl daemon-reload
echo "[+] Lab complete. Environment restored."
```

---

##### Lab 2: Enterprise Database Maintenance Timer and Catch-Up Scheduling

###### Objective
Implement an automated database optimization and vacuum pipeline executed by a `systemd.timer`. The task must execute daily at 03:00, recover missed runs using `Persistent=true`, avoid cluster load spikes using `RandomizedDelaySec=`, and log directly to the system journal.

###### Implementation Script
```bash
#!/usr/bin/env bash
set -euo pipefail

TIMER_FILE="/etc/systemd/system/db-maintenance.timer"
SERVICE_FILE="/etc/systemd/system/db-maintenance.service"
WORK_SCRIPT="/usr/local/bin/run-db-maintenance.sh"

echo "[*] Step 1: Provisioning the maintenance script..."
sudo bash -c "cat << 'EOF' > ${WORK_SCRIPT}
#!/usr/bin/env bash
set -euo pipefail
echo \"[INFO] Database maintenance pipeline started at \$(date -u -Iseconds)\"
# Simulate workload:
sleep 2
echo \"[INFO] Vacuuming tables completed successfully.\"
echo \"[INFO] Re-indexing indices completed successfully.\"
echo \"[INFO] Maintenance completed successfully.\"
exit 0
EOF"

sudo chmod +x "${WORK_SCRIPT}"

echo "[*] Step 2: Creating companion service unit (Type=oneshot)..."
sudo bash -c "cat << 'EOF' > ${SERVICE_FILE}
[Unit]
Description=Database Maintenance Task
After=network.target

[Service]
Type=oneshot
ExecStart=${WORK_SCRIPT}
StandardOutput=journal
StandardError=journal
EOF"

echo "[*] Step 3: Constructing timer unit with calendar and persistence..."
sudo bash -c "cat << 'EOF' > ${TIMER_FILE}
[Unit]
Description=Trigger Database Maintenance Daily with Catch-Up
Documentation=https://docs.internal.net/runbooks/db/

[Timer]
Unit=db-maintenance.service
OnCalendar=*-*-* 03:00:00
RandomizedDelaySec=10m
Persistent=true
AccuracySec=1s

[Install]
WantedBy=timers.target
EOF"

echo "[*] Step 4: Reloading systemd and enabling timer..."
sudo systemctl daemon-reload
sudo systemctl enable --now db-maintenance.timer

echo "[*] Step 5: Inspecting timer registration and calendar schedule..."
systemctl list-timers --all db-maintenance.timer

echo -e "\n[*] Step 6: Testing manual one-off execution of companion service..."
sudo systemctl start db-maintenance.service

echo -e "\n[*] Step 7: Verifying execution logs in journal database..."
journalctl -u db-maintenance.service -n 10 --no-pager

echo -e "\n[*] Cleaning up lab assets..."
sudo systemctl stop db-maintenance.timer
sudo systemctl disable db-maintenance.timer
sudo rm -f "${TIMER_FILE}" "${SERVICE_FILE}" "${WORK_SCRIPT}"
sudo systemctl daemon-reload
echo "[+] Lab complete. Environment restored."
```

---

##### Lab 3: Troubleshooting a Failing Service with Drop-Ins and Journal Diagnostics

###### Objective
Diagnose a misconfigured service using `journalctl -xeu`, trace failing permissions, apply a non-destructive drop-in override to remediate the issue, and confirm successful recovery.

###### Implementation Script
```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE_FILE="/etc/systemd/system/broken-app.service"
OVERRIDE_DIR="/etc/systemd/system/broken-app.service.d"

echo "[*] Step 1: Creating deliberately broken service..."
sudo bash -c "cat << 'EOF' > ${SERVICE_FILE}
[Unit]
Description=Faulty Application Service
After=network.target

[Service]
Type=simple
# User does not exist, causing activation failure:
User=nonexistent_service_user
ExecStart=/usr/bin/sleep 30
Restart=on-failure
RestartSec=1s
StartLimitBurst=2
StartLimitIntervalSec=10s

[Install]
WantedBy=multi-user.target
EOF"

sudo systemctl daemon-reload

echo "[*] Step 2: Attempting to start the broken service..."
set +e
sudo systemctl start broken-app.service
STATUS=$?
set -e

if [ ${STATUS} -ne 0 ]; then
    echo "[!] Expected failure: Service failed to start."
fi

echo -e "\n[*] Step 3: Diagnosing failure via journalctl..."
# Query exact errors associated with this failure:
journalctl -u broken-app.service -n 5 --no-pager -p err..emerg

echo -e "\n[*] Step 4: Applying drop-in override remediation..."
sudo mkdir -p "${OVERRIDE_DIR}"
sudo bash -c "cat << 'EOF' > ${OVERRIDE_DIR}/fix-user.conf
[Service]
# Override the invalid user:
User=nobody
EOF"

echo "[*] Step 5: Reloading daemon and restarting service..."
sudo systemctl daemon-reload
sudo systemctl restart broken-app.service

echo -e "\n[*] Step 6: Verifying restored operational state..."
systemctl status broken-app.service --no-pager

echo -e "\n[*] Cleaning up lab assets..."
sudo systemctl stop broken-app.service
sudo rm -rf "${SERVICE_FILE}" "${OVERRIDE_DIR}"
sudo systemctl daemon-reload
echo "[+] Lab complete. Environment clean."
```

---

#### 6. Comprehensive Operational Reference Matrices

##### Unit Types and Execution Semantics
| Extension | Functional Classification | Operational Purpose | Default Active State Trigger |
| :--- | :--- | :--- | :--- |
| **`.service`** | Service Execution | Supervises daemons, processes, and scripts. | Process active, ready signal, or exit $0$. |
| **`.socket`** | Socket Interception | Listens on IPC/network sockets for on-demand activation. | Socket bound and listening in kernel. |
| **`.target`** | Synchronization Group | Groups units to achieve designated boot/runtime targets. | All required child units reach active state. |
| **`.timer`** | Task Scheduling | Manages monotonic or calendar-based periodic schedules. | Timer registered with kernel event loop. |
| **`.mount`** | Filesystem Attachment | Controls filesystem mounts (replaces `/etc/fstab`). | VFS mount completed successfully. |
| **`.automount`**| On-Demand Mounting | Mounts filesystems transparently upon directory access. | Virtual automount directory established. |
| **`.slice`** | Cgroups Allocation | Partitions memory, CPU, and I/O resource trees. | Cgroup slice directory created in sysfs. |
| **`.path`** | Inotify Monitoring | Triggers services upon file/directory modification. | Inotify watch established by kernel. |

---

##### `systemctl` Command Matrix
| Command | Operational Domain | Functional Purpose |
| :--- | :--- | :--- |
| `systemctl start <unit>` | Runtime Control | Enqueues and executes unit activation transaction. |
| `systemctl stop <unit>` | Runtime Control | Sends termination signals to cgroup processes. |
| `systemctl restart <unit>` | Runtime Control | Terminates running unit and starts it fresh. |
| `systemctl reload <unit>` | Runtime Control | Triggers non-disruptive configuration reload. |
| `systemctl enable <unit>` | Boot Persistence | Creates target symlinks in `/etc/systemd/system/`. |
| `systemctl disable <unit>` | Boot Persistence | Removes target activation symlinks. |
| `systemctl mask <unit>` | Access Control | Symlinks unit to `/dev/null`, blocking all execution. |
| `systemctl unmask <unit>` | Access Control | Removes `/dev/null` symlink, restoring accessibility. |
| `systemctl daemon-reload` | Metadata Re-sync | Reparses all unit configurations from disk. |
| `systemctl daemon-reexec` | Manager Execution | Re-executes PID 1 binary, preserving active state. |
| `systemctl list-units` | Diagnostics | Lists active units currently loaded in memory. |
| `systemctl list-unit-files`| Diagnostics | Lists all unit files installed on disk and enable states. |
| `systemctl edit <unit>` | Configuration | Opens editor to generate modular drop-in override. |

---

##### Calendar Expressions Reference (`OnCalendar=`)
| Expression | Equivalence / Normalization | Elapse Cadence |
| :--- | :--- | :--- |
| `minutely` | `*-*-* *:*:00` | Once per minute at the start of the minute. |
| `hourly` | `*-*-* *:00:00` | At the top of every hour. |
| `daily` | `*-*-* 00:00:00` | Once per day at midnight UTC. |
| `weekly` | `Mon *-*-* 00:00:00` | Every Monday morning at midnight UTC. |
| `monthly` | `*-*-01 00:00:00` | First day of every month at midnight UTC. |
| `yearly` | `*-01-01 00:00:00` | Annually on January 1st at midnight UTC. |
| `*-*-* 02..05:00:00` | Explicit Hour Range | Fires at 02:00, 03:00, 04:00, and 05:00. |
| `*-*-* 00:00/15:00` | Incremental Stepping | Every 15 minutes starting on the hour. |

---

##### Sandboxing and Hardening Reference
| Directive | Valid Values | Security Effect |
| :--- | :--- | :--- |
| **`ProtectSystem`** | `strict`, `full`, `yes`, `no` | Mounts system directories (`/usr`, `/etc`) read-only. |
| **`ProtectHome`** | `yes`, `read-only`, `no` | Blocks or restricts access to `/home` and `/root`. |
| **`PrivateTmp`** | `yes`, `no` | Mounts an isolated `tmpfs` for `/tmp`. |
| **`PrivateDevices`**| `yes`, `no` | Restricts `/dev` to loopback and pseudo-devices. |
| **`PrivateNetwork`**| `yes`, `no` | Disconnects process from all physical network interfaces. |
| **`DynamicUser`** | `yes`, `no` | Allocates transient UID/GID at runtime. |
| **`NoNewPrivileges`**| `yes`, `no` | Disables SUID execution and capability gains. |
| **`ProtectKernelTunables`**| `yes`, `no` | Prevents modifications to `/proc/sys` and `/sys`. |
| **`ProtectControlGroups`**| `yes`, `no` | Makes cgroups hierarchy read-only within the namespace. |
| **`SystemCallFilter`**| Filter sets / Call names | Seccomp BPF filter restricting allowed system calls. |

---

##### `journalctl` Filtering and Maintenance Cheat Sheet
| Syntax Flag | Target Evaluation | Operational Usage |
| :--- | :--- | :--- |
| **`-u, --unit=`** | `_SYSTEMD_UNIT` | Filter logs by matching systemd unit name. |
| **`-b, --boot=`** | `_BOOT_ID` | Filter by current (`0`), previous (`-1`), or explicit boot ID. |
| **`-p, --priority=`**| `PRIORITY` | Filter by severity level (e.g., `err`, `warning`, `info`). |
| **`--since / --until`**| Realtime Timestamp | Restricts output to absolute or relative time ranges. |
| **`-k, --dmesg`** | Kernel Facilities | Displays kernel messages from `/dev/kmsg`. |
| **`-f, --follow`** | Stream Tail | Continuously streams incoming log lines. |
| **`-o, --output=`** | Formatter Engine | Select output format: `short`, `verbose`, `json`, `cat`. |
| **`--vacuum-size=`** | Disk Allocation | Deletes oldest archived journals until below target size. |
| **`--vacuum-time=`** | Data Retention | Purges logs older than the specified duration. |
| **`--rotate`** | File Management | Rotates active log files into archived status immediately. |
| **`--verify`** | Security Audit | Cryptographically audits Forward Secure Sealing (FSS). |