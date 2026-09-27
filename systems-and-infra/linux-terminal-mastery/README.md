# Linux Systems & Terminal Mastery Index

---

## Block 1: Unix Philosophy and Shell Mechanics

### [`01-unix-philosophy-and-shell-mechanics.md`](./01-unix-philosophy-and-shell-mechanics.md)

* Unix design principles: small, modular programs focused on text streams
* Command-line lifecycle: strict shell parsing order
* Quoting rules: expansions in double quotes, protection with single quotes, and escape characters
* Command evaluation: priority resolution among aliases, functions, builtins, and external binaries
* Managing environment variables, process scope, and shell initialization files

### [`02-file-descriptors-and-redirection.md`](./02-file-descriptors-and-redirection.md)

* Standard descriptor theory: stdin (0), stdout (1), and stderr (2)
* Compound redirection constructs and fine-grained output control
* Creating custom file descriptors for script I/O control
* Complex data input: *Here Documents*, *Here Strings*, and *Process Substitution*
* Duplicating, redirecting, and closing communication channels

### [`03-text-processing-and-transformation.md`](./03-text-processing-and-transformation.md)

* Filtering with basic and extended regular expressions via `grep`
* Non-interactive stream editing with `sed`
* Column-structured processing, arithmetic operations, and formatting with `awk`
* High-performance pipeline combinations: `cut`, `sort`, `uniq`, `tr`, and `xargs`

---

## Block 2: File Structure, Storage, and Permissions

### [`04-vfs-hierarchy-inodes-and-links.md`](./04-vfs-hierarchy-inodes-and-links.md)

* Filesystem Hierarchy Standard (FHS) and the VFS (Virtual File System) layer
* File metadata and inode structures
* Operational and architectural differences between symbolic links and hard links
* Mount lifecycle: parsing `/etc/fstab`, mount options, and the `findmnt` command
* Disk usage analysis: disparities between apparent size and allocated blocks (`du`, `df`)

### [`05-advanced-permissions-attributes-and-acls.md`](./05-advanced-permissions-attributes-and-acls.md)

* Traditional POSIX permissions: octal modes, symbolic modes, and `umask` calculation
* Special execution and security bits: SUID, SGID, and Sticky Bit
* Immutable attributes and low-level restrictions with `chattr` and `lsattr`
* Granular access control beyond the owner/group model using ACLs (`getfacl`, `setfacl`)

### [`06-archiving-compression-and-synchronization.md`](./06-archiving-compression-and-synchronization.md)

* Archiving with `tar` and compression algorithms (`gzip`, `bzip2`, `xz`, `zstd`)
* Efficient differential file transfer and synchronization via `rsync`
* Data integrity verification and auditing using cryptographic hashes (`sha256sum`, `md5sum`)

---

## Block 3: Hardware, Boot, and Service Management

### [`07-hardware-recognition-and-kernel-modules.md`](./07-hardware-recognition-and-kernel-modules.md)

* Terminal-based hardware topology inspection: PCI buses, USB devices, and block storage devices (`lspci`, `lsusb`, `lsblk`)
* Kernel event monitoring and device discovery via `dmesg` and `udev`
* Kernel module management: manual loading, dependencies, and unloading (`lsmod`, `modprobe`, `rmmod`)

### [`08-boot-sequence-and-system-initialization.md`](./08-boot-sequence-and-system-initialization.md)

* Boot process stages: UEFI/BIOS, GRUB, initramfs, and handover to PID 1
* Inspecting and modifying kernel boot parameters
* Recovering non-booting systems using target states (*rescue* and *emergency targets*)

### [`09-service-administration-with-systemd.md`](./09-service-administration-with-systemd.md)

* Initialization architecture and daemon management via `systemctl`
* Creating custom unit files (`.service`) and configuring dependency management
* Modern periodic task scheduling with `systemd.timer` as a replacement for cron
* Reading, filtering, and maintaining binary logs with `journalctl`

---

## Block 4: Processes, Memory, and Kernel Resources

### [`10-process-model-and-posix-signals.md`](./10-process-model-and-posix-signals.md)

* Process creation and lifecycle: `fork` and `exec` system calls, process hierarchy, and PID 1
* Zombie and orphan processes: causes, diagnosis, and resolution
* Kernel signal table and operational handling behavior (SIGTERM, SIGKILL, SIGHUP, SIGINT)
* Interactive execution control: background jobs, terminal disowning (`nohup`, `disown`), and inspection utilities (`ps`, `top`, `pgrep`, `pkill`)

### [`11-memory-management-and-swap-subsystem.md`](./11-memory-management-and-swap-subsystem.md)

* Memory topology: free memory, available memory, buffers, and page cache
* Swap space: balancing physical memory and tuning `swappiness`
* Diagnosing actual process memory footprints: RSS, VSZ, and PSS
* The role of the Out-Of-Memory (OOM) Killer: selection criteria, scoring, and log inspection

### [`12-virtual-kernel-interfaces-proc-and-sys.md`](./12-virtual-kernel-interfaces-proc-and-sys.md)

* The `/proc` filesystem as a real-time reflection of the kernel and running processes
* The `/sys` directory and the unified hardware device tree
* Reading and persistently modifying kernel runtime parameters via `sysctl`

### [`13-resource-isolation-namespaces-and-cgroups.md`](./13-resource-isolation-namespaces-and-cgroups.md)

* The foundation of Linux containment: namespaces (PID, Mount, Network, User, IPC, UTS)
* Constraining CPU, memory, and I/O consumption with Control Groups (cgroups v2)
* Command-line environment isolation using native utilities (`chroot`, `unshare`)

---

## Block 5: Networking, Connectivity, and Remote Access

### [`14-posix-networking-stack-and-routing.md`](./14-posix-networking-stack-and-routing.md)

* Network interface state and configuration via the `iproute2` suite (`ip link`, `ip addr`, `ip route`)
* Reading and modifying routing tables and default gateways
* Mapping network sockets, open ports, and active connections with `ss`
* Userspace network isolation using network namespaces (`ip netns`)

### [`15-name-resolution-and-traffic-analysis.md`](./15-name-resolution-and-traffic-analysis.md)

* DNS resolution workflow in Linux: `/etc/hosts`, `/etc/resolv.conf`, and `nsswitch.conf`
* Diagnosing and validating DNS queries with `dig`
* Route, hop, and latency analysis with `traceroute` and `mtr`
* Packet capture and inspection directly in the terminal with `tcpdump`

### [`16-remote-access-and-network-security.md`](./16-remote-access-and-network-security.md)

* Advanced SSH client configuration via `~/.ssh/config`
* Port forwarding and tunneling: local (`-L`), remote (`-R`), and dynamic (`-D`)
* Packet filtering fundamentals and firewall rules with `nftables` and `iptables`
* Auditing inbound connections and system authentication history

---

## Block 6: Debugging, Automation, and Productivity

### [`17-low-level-tracing-and-troubleshooting.md`](./17-low-level-tracing-and-troubleshooting.md)

* Intercepting system calls in real time using `strace`
* Troubleshooting shared library dependencies and dynamic loading (`ldd`)
* Identifying open files, process locks, and associated sockets with `lsof`
* Methods for detecting processes blocked in disk wait states (I/O Wait)

### [`18-defensive-shell-scripting-and-advanced-environment.md`](./18-defensive-shell-scripting-and-advanced-environment.md)

* Strict execution mode in scripts: `set -euo pipefail`
* Handling unexpected errors and cleanup routines with `trap`
* Static code analysis with `shellcheck`
* Terminal multiplexing with `tmux` and customizing shortcuts in Readline mode
