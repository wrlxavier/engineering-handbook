### 04. VFS Hierarchy, Inodes, and Links
Storage management in Linux extends far beyond hardware storage media and physical partition tables. The kernel abstracts physical storage, networked volumes, memory-backed scratchpads, and synthetic process states into a single, cohesive, hierarchical directory tree. This unified abstraction is driven by the Virtual File System (VFS) and organized according to the Filesystem Hierarchy Standard (FHS).

Mastering Linux storage requires understanding this abstraction from both architectural and operational standpoints: how directory paths resolve through kernel caches to on-disk inodes, how links manipulate low-level reference counters, how mounts attach filesystems to the global namespace, and why standard userland utilities like `du` and `df` frequently disagree on storage consumption.

---

#### 1. Filesystem Hierarchy Standard (FHS) and the VFS Layer
Unix-like operating systems present storage as a unified tree anchored at a single root (`/`). This design contrasts with systems that assign drive letters (such as `C:` or `D:`) to individual storage volumes. Under Linux, independent partitions, dynamic memory allocations, and network shares are mounted onto specific directories within this unified tree.

##### The Filesystem Hierarchy Standard (FHS 3.0)
The Filesystem Hierarchy Standard (maintained by the Linux Foundation) formalizes directory structure conventions across Linux distributions. It classifies files along two distinct operational axes:
1. **Shareable vs. Unshareable**: Can the data be shared across distinct host systems (e.g., via NFS), or is it bound strictly to the local host?
2. **Static vs. Variable**: Does the data remain unchanged without manual administrative action (binaries, static libraries, documentation), or is it altered dynamically by running system daemons (spools, logs, temporary runtime states)?

```
┌──────────────────┬───────────────────────────────┬───────────────────────────────┐
│                  │ Shareable                     │ Unshareable                   │
├──────────────────┼───────────────────────────────┼───────────────────────────────┤
│ **Static**       │ `/usr` (read-only binaries)   │ `/etc` (host configurations)  │
│                  │ `/opt` (third-party packages) │ `/boot` (kernel images/GRUB)  │
├──────────────────┼───────────────────────────────┼───────────────────────────────┤
│ **Variable**     │ `/var/mail` (user inboxes)    │ `/var/run` -> `/run` (PIDs)   │
│                  │ `/var/spool/news`             │ `/var/log` (host event logs)  │
│                  │ `/srv` (site data)            │ `/tmp` (ephemeral local data) │
└──────────────────┴───────────────────────────────┴───────────────────────────────┘
```

###### Core Directory Roles
* **`/` (Root)**: The root node of the hierarchy. It must contain the critical components needed to boot the system, mount secondary filesystems, and enter recovery or emergency maintenance modes.
* **`/boot`**: Static files required by the bootloader (GRUB/systemd-boot), including the Linux kernel binary (`vmlinuz`), the initial RAM disk image (`initramfs` or `initrd`), and secondary boot configurations.
* **`/dev`**: Device nodes managed dynamically by `udev` over the in-memory `devtmpfs` filesystem. Character and block devices (e.g., `/dev/nvme0n1`, `/dev/urandom`, `/dev/pts/1`) expose kernel hardware interfaces as standard filesystem paths.
* **`/etc`**: Host-specific, machine-local, static configuration files. Under the FHS, `/etc` must not contain executable binary code or dynamic operational state files.
* **`/bin`, `/sbin`, `/lib`, `/lib64` (The `/usr`-Merge)**: Historically, `/bin` (essential user binaries), `/sbin` (essential administrative binaries), and `/lib` (essential shared objects) lived on the root filesystem partition so the system could boot into single-user mode before secondary `/usr` partitions were mounted. Modern Linux distributions implement the **`/usr`-Merge**:
  ```bash
  $ ls -ld /bin /sbin /lib /lib64
  lrwxrwxrwx 1 root root 7 Jan  1  2024 /bin -> usr/bin
  lrwxrwxrwx 1 root root 8 Jan  1  2024 /lib -> usr/lib
  lrwxrwxrwx 1 root root 9 Jan  1  2024 /lib64 -> usr/lib64
  lrwxrwxrwx 1 root root 8 Jan  1  2024 /sbin -> usr/sbin
  ```
  All operational system binaries now reside in `/usr/bin`, consolidating the core system packages into `/usr` while preserving legacy script paths via symlinks.
* **`/home` and `/root`**: User home directories. Regular users are mapped to `/home/<username>`, while the administrative superuser uses `/root` to ensure access even when `/home` is unmounted or resides on an inaccessible remote partition.
* **`/proc`**: A synthetic, pseudo-filesystem generated dynamically by the kernel. It exposes kernel data structures, tunables, and per-process operational states (via `/proc/<PID>`). It consumes zero persistent disk storage.
* **`/sys`**: The `sysfs` pseudo-filesystem. It exports a unified, hierarchical view of kernel device drivers, buses, subsystems, and power-management controls.
* **`/run`**: A volatile, memory-backed (`tmpfs`) storage area housing operational runtime data generated since the machine booted. It contains PID files, UNIX domain control sockets, lock files, and systemd runtime state.
* **`/tmp` vs. `/var/tmp`**:
  * `/tmp`: Ephemeral scratchpad storage, typically backed by `tmpfs` (in RAM) or cleared automatically on system reboot.
  * `/var/tmp`: Persistent temporary storage meant to survive reboots, reserved for long-running batch jobs, dump files, and installation packages.
* **`/usr`**: The secondary hierarchy containing shareable, read-only system software: user binaries (`/usr/bin`), administrative binaries (`/usr/sbin`), header files (`/usr/include`), libraries (`/usr/lib`), and architecture-independent data (`/usr/share`).
* **`/var`**: Variable data written by running daemons: log files (`/var/log`), transient spool queues (`/var/spool`), cached application assets (`/var/cache`), and dynamic database directories (`/var/lib`).

---

##### The Virtual File System (VFS) Layer Architecture
Linux can interact seamlessly with dozens of fundamentally different filesystem formats: native journaling systems (Ext4, XFS), modern Copy-on-Write systems (Btrfs, ZFS), flash media filesystems (F2FS), network storage protocols (NFS, CIFS), and memory-backed pseudo-filesystems (`procfs`, `sysfs`, `tmpfs`).

To prevent userland programs and standard library system calls from needing dedicated logic for every distinct storage driver, the kernel inserts an intermediate abstraction layer: the **Virtual File System (VFS)**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                   User Applications (libc / POSIX)                     │
│               open(), read(), write(), stat(), unlink()                │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ System Call Interface
┌───────────────────────────────────▼────────────────────────────────────┐
│                    Virtual File System (VFS)                           │
│                                                                        │
│   ┌────────────────┐ ┌────────────────┐ ┌──────────────────────────┐  │
│   │ struct file    │ │ struct dentry  │ │ struct inode             │  │
│   │ (Open instance)│ │ (Path cache)   │ │ (Metadata & Operations)  │  │
│   └────────────────┘ └────────────────┘ └──────────────────────────┘  │
│                               │                                        │
│                     struct super_block                                 │
│                     (Filesystem Mount State)                           │
└───────┬───────────────────────┼───────────────────────┬────────────────┘
        │                       │                       │
┌───────▼───────┐       ┌───────▼───────┐       ┌───────▼────────────────┐
│  Ext4 Driver  │       │  XFS Driver   │       │ Network File System    │
│ (ext4_inode)  │       │  (xfs_inode)  │       │ (NFS Client / RPC)     │
└───────┬───────┘       └───────┬───────┘       └───────┬────────────────┘
        │                       │                       │
┌───────▼───────────────────────▼───────────────────────▼────────────────┐
│           Block Layer (Generic Block Layer, I/O Schedulers)            │
└───────────────────────────────┬────────────────────────────────────────┘
                                │
┌───────────────────────────────▼────────────────────────────────────────┐
│      Physical Storage (NVMe SSD, SATA HDD, SAN, Loopback Device)       │
└────────────────────────────────────────────────────────────────────────┘
```

The VFS defines an object-oriented C contract. Any filesystem driver registered with the kernel (`register_filesystem()`) must instantiate concrete functions that fulfill function pointer tables defined by four primary VFS objects:

###### 1. `struct super_block`
Represents an entire mounted filesystem instance.
* **Kernel Definition**: Tracks global filesystem characteristics, including block size, allocation flags, dirty flags, maximum file size limits, the root dentry pointer, and a pointer to the storage device.
* **Operations Table (`super_operations`)**: Defines system-level hooks including `alloc_inode()`, `destroy_inode()`, `write_inode()`, `sync_fs()`, and `statfs()`.

###### 2. `struct inode`
Represents a unique, addressable storage object (a regular file, directory, symlink, pipe, socket, or device node).
* **Kernel Definition**: Contains pure metadata describing the object: numeric inode identifier (`i_ino`), owner UID/GID (`i_uid`, `i_gid`), permissions bitmask (`i_mode`), size in bytes (`i_size`), link count (`i_nlink`), and data block/extent addressing pointers.
* **Operations Table (`inode_operations`)**: Defines hooks that manipulate file entries rather than content: `create()`, `lookup()`, `link()`, `unlink()`, `symlink()`, `mkdir()`, `rmdir()`, `rename()`, and `setattr()`.

###### 3. `struct dentry` (Directory Entry)
Represents a single path component in the filesystem tree, linking a textual name string to an inode.
* **Kernel Definition**: Dentries form an in-memory cache—the **dentry cache (dcache)**—that bridges human-readable path strings with VFS inodes. A dentry contains a pointer to the target inode (`d_inode`), the textual name of the path element (`d_name`), and pointers to parent/child dentries (`d_parent`, `d_subdirs`).
* **Dentry States**:
  * *Used*: Currently referenced by active processes; bound to an active inode.
  * *Unused*: Not actively held open by any process; retained in memory on a Least-Recently-Used (LRU) list to accelerate future lookups until memory pressure triggers slab reclamation.
  * *Negative*: Inode is null. The kernel records that a specific file path **does not exist**, allowing subsequent lookups for missing files (e.g., continuous 404 queries) to fail immediately without accessing underlying disk blocks.
* **Operations Table (`dentry_operations`)**: Contains hooks such as `d_revalidate()` (vital for network filesystems like NFS where remote files might change out-of-band), `d_hash()`, and `d_compare()`.

###### 4. `struct file`
Represents an open file instance created in memory when a process invokes `open()` or `openat()`.
* **Kernel Definition**: Does not exist persistently on disk. It tracks per-process operational state: the current file read/write byte offset (`f_pos`), access flags (`f_flags`, e.g., `O_RDWR`, `O_APPEND`), the reference count (`f_count`), a pointer to the matching dentry (`f_path.dentry`), and the active mount point (`f_path.mnt`).
* **Operations Table (`file_operations`)**: Handles payload input/output operations: `read()`, `write()`, `mmap()`, `fsync()`, `poll()`, `splice()`, and `unlocked_ioctl()`.

##### Path Resolution: The VFS Lookup Trace
When an application calls `open("/var/log/syslog", O_RDONLY)`, the VFS resolves the textual path string step by step:
1. **Root Pinning**: The kernel identifies the lookup root (either the current working directory from `current->fs->pwd` or the system root from `current->fs->root`).
2. **Dcache Traversal**: The path string is tokenized by `/`. The kernel hashes the token `"var"` alongside the parent dentry pointer and queries the dcache hash table (`d_hash`).
   * *Cache Hit*: The kernel grabs the `var` dentry and retrieves its bound inode.
   * *Cache Miss*: The kernel invokes `inode_operations->lookup()` on the root directory inode, causing the concrete filesystem driver (e.g., Ext4) to scan on-disk directory blocks, read the matching inode into memory, allocate a new dentry, and insert it into the dcache.
3. **Permission Checks**: At each token step, the VFS validates whether the calling process possesses execute (`+x`) permissions on the containing directory's inode.
4. **Subdirectory Resolution**: The lookup moves sequentially to `"log"`, then to `"syslog"`.
5. **Open File Construction**: Once the target inode for `"syslog"` is located, the kernel allocates a `struct file` in system memory, sets `f_pos = 0`, attaches the corresponding `file_operations` table from the underlying filesystem driver, and inserts a pointer into the process's `fdtable`, returning the lowest free integer file descriptor to userland.

---

#### 2. File Metadata and Inode Structures
In Linux filesystems, file data is cleanly separated from file metadata. The data blocks of a file store its raw payload (text, compiled binaries, media), while its structural properties, access parameters, and physical disk locations are stored in an **Index Node (Inode)**.

##### What is an Inode?
An inode is an on-disk, fixed-size data structure (typically $256\text{ bytes}$ in Ext4, or $512\text{ bytes}$ in XFS) allocated in dedicated inode tables during filesystem creation (`mkfs`). 

Every inode within a filesystem is identified by an unsigned integer: the **inode number** (`ino_t`). Crucially, an inode number is unique **only within its specific filesystem superblock**. Two files on different physical partitions can share the exact same inode number.

```
On-Disk Ext4 Inode Structure (Simplified, 256 bytes)
┌────────────────────────────────────────────────────────┐
│ File Mode (16 bits: File Type + Access Permissions)    │
├────────────────────────────────────────────────────────┤
│ Owner UID (32 bits)                                    │
├────────────────────────────────────────────────────────┤
│ Group GID (32 bits)                                    │
├────────────────────────────────────────────────────────┤
│ File Size in Bytes (64 bits: i_size_lo + i_size_high)  │
├────────────────────────────────────────────────────────┤
│ Timestamps:                                            │
│   ├── Access Time (atime: 32/64-bit nanosecond)        │
│   ├── Modification Time (mtime: 32/64-bit nanosecond)  │
│   ├── Change Time (ctime: 32/64-bit nanosecond)        │
│   └── Deletion / Creation Time (crtime / btime)        │
├────────────────────────────────────────────────────────┤
│ Hard Link Reference Counter (i_nlink, 16 bits)         │
├────────────────────────────────────────────────────────┤
│ Allocated 512-byte Sector Count (i_blocks, 32 bits)    │
├────────────────────────────────────────────────────────┤
│ Extended Attributes / Flags (chattr, immutability)     │
├────────────────────────────────────────────────────────┤
│ Storage Pointers:                                      │
│   ├── Legacy Ext2/3: 12 Direct, 1 Ind, 1 DInd, 1 TInd  │
│   └── Modern Ext4: Extent Tree Root Header             │
│       [ext4_extent_header] + [ext4_extent entries]     │
└────────────────────────────────────────────────────────┘
```

##### Inode Fields and Internal Metadata
1. **File Type and Permissions (`i_mode`)**:
   * Encodes both the POSIX access permissions (e.g., `0755` / `rwxr-xr-x`) and the file type classification bitmask:
     * `S_IFREG`: Regular data file
     * `S_IFDIR`: Directory
     * `S_IFLNK`: Symbolic link
     * `S_IFBLK`: Block device node
     * `S_IFCHR`: Character device node
     * `S_IFIFO`: Named pipe (FIFO)
     * `S_IFSOCK`: UNIX domain socket
2. **Ownership Identifiers (`i_uid`, `i_gid`)**:
   * Numeric User ID and Group ID governing standard POSIX permission checks.
3. **Payload Dimensions (`i_size`)**:
   * A 64-bit integer recording the apparent file size in bytes. This dictates the End-Of-File (EOF) boundary for reading applications.
4. **Allocated Block Count (`i_blocks`)**:
   * The actual number of $512\text{-byte}$ sectors allocated to the file on disk storage, including indirect blocks or extent overhead.
5. **The POSIX Timestamp Trio (+ Creation Time)**:
   * **`atime` (Access Time)**: Updated when file contents are read by a system call (`read(2)`, `execve(2)`, `mmap(2)`).
   * **`mtime` (Modification Time)**: Updated when file contents are modified (`write(2)`, `truncate(2)`).
   * **`ctime` (Change / Status Time)**: Updated whenever the file's **metadata** or inode attributes change (ownership, permissions via `chmod`, renaming, altering link count, or modifying data content). *An application cannot forge or directly set `ctime` via standard POSIX system calls; it is strictly updated by the kernel.*
   * **`crtime` / `btime` (Birth / Creation Time)**: The timestamp indicating when the inode was first populated. Modern Linux filesystems (Ext4, XFS, Btrfs) track creation time, which can be retrieved using the modern `statx(2)` system call.
6. **Reference Counter (`i_nlink`)**:
   * The hard link counter. It tracks how many directory entries across the filesystem point directly to this inode. When `i_nlink` reaches zero ($0$) and all running processes holding the inode open close their file descriptors, the filesystem marks the inode and its underlying disk blocks as unallocated.

##### What is NOT Stored in an Inode?
The inode contains nearly all metadata associated with a file, with two critical exceptions:
1. **The file's name**.
2. **The file's directory path**.

In Unix architecture, **a directory is simply a specialized file whose payload is a list of filename-to-inode mappings**. 

```
Directory Data Payload on Disk (e.g., /home/alice)
┌──────────────────────┬──────────────┬──────────────┬───────────────────┐
│ Inode Number (uint)  │ Entry Length │ Name Length  │ Text Filename     │
├──────────────────────┼──────────────┼──────────────┼───────────────────┤
│ 131074               │ 12           │ 1            │ .                 │
│ 131072               │ 12           │ 2            │ ..                │
│ 131075               │ 24           │ 11           │ .bashrc           │
│ 262145               │ 20           │ 8            │ data.txt          │
│ 262146               │ 24           │ 10           │ report.pdf        │
└──────────────────────┴──────────────┴──────────────┴───────────────────┘
```
Filenames exist exclusively as textual label mappings inside parent directory payload blocks. This design makes it possible for multiple different filenames in different directories to reference the exact same inode.

---

##### Block Pointer Addressing: Legacy Pointers vs. Modern Extents
How does an inode point to the physical storage blocks that contain a file's actual byte payload?

###### Legacy Ext2/Ext3 Indirect Block Addressing
Early Unix filesystems and Ext2/Ext3 used an indirect block pointer array consisting of 15 discrete pointers:
* **Direct Pointers (Pointers 0–11)**: Point directly to individual data blocks (e.g., $4\text{ KiB}$ each). Can address up to $12 \times 4\text{ KiB} = 48\text{ KiB}$.
* **Single Indirect Pointer (Pointer 12)**: Points to an indirect block full of pointers (a $4\text{ KiB}$ block contains $1\,024$ pointers of 4 bytes each), addressing up to $1\,024 \times 4\text{ KiB} = 4\text{ MiB}$.
* **Double Indirect Pointer (Pointer 13)**: Points to a block containing pointers to indirect blocks, addressing up to $1\,024 \times 1\,024 \times 4\text{ KiB} = 4\text{ GiB}$.
* **Triple Indirect Pointer (Pointer 14)**: Adds a third layer of indirection, addressing up to $1\,024 \times 1\,024 \times 1\,024 \times 4\text{ KiB} = 4\text{ TiB}$.

```
Legacy Inode Block Pointer Architecture
┌─────────────┐
│ Inode       │
│  [Direct 0] ───────────────────────> [ Data Block 0 (4 KiB) ]
│  [Direct 1] ───────────────────────> [ Data Block 1 (4 KiB) ]
│  ...        │
│  [Direct 11]───────────────────────> [ Data Block 11 (4 KiB) ]
│             │
│  [Indirect] ──────> [ Pointer Block ]
│             │        ├──> [ Data Block 12 ]
│             │        └──> [ Data Block 1035 ]
│             │
│  [Double]   ──────> [ Double Ind. Block ]
│             │        └──> [ Pointer Block ] ──> [ Data Block ... ]
│             │
│  [Triple]   ──────> [ Triple Ind. Block ] ──> ...
└─────────────┘
```
*Disadvantage*: For large multi-gigabyte files, indirect block addressing incurs significant I/O overhead and fragments filesystem space, requiring multiple metadata block reads just to locate payload data blocks.

###### Modern Ext4 Extent Trees
Ext4 replaces indirect block pointer arrays with **extents**. An extent represents a single descriptor mapping a contiguous range of logical file blocks to a contiguous range of physical disk blocks.
A single `struct ext4_extent` covers up to $32\text{ MiB}$ of contiguous storage (using $4\text{ KiB}$ blocks):
```c
struct ext4_extent {
    __le32  ee_block;    // First logical block extent covers
    __le16  ee_len;      // Number of blocks covered by extent (up to 32,768)
    __le16  ee_start_hi; // High 16 bits of physical block number
    __le32  ee_start_lo; // Low 32 bits of physical block number
};
```
If a file's allocations are contiguous, a multi-gigabyte file can be described by just a few extent records stored directly within the inode body (`struct ext4_extent_header`), bypassing indirect trees altogether. If a file fragments across many extents, the inode builds a balanced B-tree of extent index nodes (`struct ext4_extent_idx`).

---

##### Inspecting Inode Structures in Userspace
Linux provides several utilities to inspect inode metadata:

###### 1. Standard POSIX Inspection (`ls -li`, `stat`)
```bash
$ ls -li /etc/passwd
1441804 -rw-r--r-- 1 root root 3042 Sep 15 10:45 /etc/passwd
# ^Ino  ^Perms     ^Lnk ^Owner    ^Bytes             ^Path

$ stat /etc/passwd
  File: /etc/passwd
  Size: 3042      	Blocks: 8          IO Block: 4096   regular file
Device: 259,2	Inode: 1441804     Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2026-09-27 01:15:02.124589000 +0000
Modify: 2026-09-15 10:45:00.000000000 +0000
Change: 2026-09-15 10:45:00.005412891 +0000
 Birth: 2026-01-10 08:30:12.451000000 +0000
```

###### 2. Low-Level Ext4 Inspection via `debugfs`
To inspect on-disk extent trees, block mappings, and raw inode fields directly:
```bash
# Dump the raw inode details of inode 1441804 from device /dev/nvme0n1p2:
sudo debugfs -R 'stat <1441804>' /dev/nvme0n1p2
```
Output reveals the internal extent hierarchy:
```
Inode: 1441804   Type: regular    Mode:  0644   Flags: 0x80000
Generation: 10459812   Version: 0x00000000:00000001
User:     0   Group:     0   Size: 3042
File ACL: 0
Links: 1   Blockcount: 8
Fragment:  Address: 0    Number: 0    Size: 0
ctime: 0x68d8108c:00203a1b -- Sun Sep 15 10:45:00 2026
atime: 0x68e77a16:1db410c8 -- Sun Sep 27 01:15:02 2026
mtime: 0x68d8108c:00000000 -- Sun Sep 15 10:45:00 2026
crtime: 0x659e51fc:6b8a2100 -- Sat Jan 10 08:30:12 2026
EXTENTS:
(0): 5849216
```

###### 3. Inode Exhaustion Hazards
Because inodes are allocated in fixed quantities when a filesystem is created, **a filesystem can run out of space even if it has gigabytes of free disk capacity**, simply by exhausting its available inodes.
```bash
$ df -i /data
Filesystem      Inodes   IUsed   IFree IUse% Mounted on
/dev/sdb1      1310720 1310720       0  100% /data

$ touch /data/newfile.txt
touch: cannot touch '/data/newfile.txt': No space left on device
```
*Root Cause*: Storing millions of tiny files (such as unpurged PHP session files, micro-cache fragments, or deep email spool queues) can consume every available inode in the allocation table. The device returns `ENOSPC` (No space left on device) even when significant block storage capacity remains.

---

#### 3. Operational and Architectural Differences: Symbolic Links vs. Hard Links
Linux filesystems provide two distinct mechanisms for referencing files under multiple paths: **Hard Links** and **Symbolic (Soft) Links**. While they may appear to serve similar purposes in userland, their internal data structures and kernel operational behaviors are fundamentally different.

```
                      Hard Link vs Symbolic Link Architecture

          Hard Link Relationship:                     Symbolic Link Relationship:

   Directory Entry        Directory Entry        Directory Entry        Directory Entry
   "original.txt"          "hardlink.txt"        "original.txt"          "symlink.txt"
  ┌──────────────┐       ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
  │ Inode: 55101 │       │ Inode: 55101 │       │ Inode: 55101 │       │ Inode: 88492 │
  └──────┬───────┘       └──────┬───────┘       └──────┬───────┘       └──────┬───────┘
         │                      │                      │                      │
         └──────────┬───────────┘                      │                      ▼
                    ▼                                  │              ┌───────────────┐
          ┌───────────────────┐                        │              │ Inode 88492   │
          │ Inode 55101       │                        │              │ (Type: S_IFLNK│
          │ Links count: 2    │                        │              │ Payload:      │
          │ Blocks: [Block A] │                        │              │ "orig.txt")   │
          └─────────┬─────────┘                        │              └───────┬───────┘
                    │                                  │                      │
                    ▼                                  ▼                      │
             [ Block A Data ]                 ┌───────────────────┐           │
             ("Payload Bytes")                │ Inode 55101       │           │
                                              │ Links count: 1    │           │
                                              │ Blocks: [Block A] │ <─────────┘
                                              └─────────┬─────────┘
                                                        │
                                                        ▼
                                                 [ Block A Data ]
```

##### Hard Links (`link(2)`)
A hard link creates a new directory entry (a filename string mapped to an inode number) that points to an **existing inode**.

* **Mechanism**:
  ```bash
  ln source_file.txt hardlink_file.txt
  ```
  1. The kernel looks up the inode corresponding to `source_file.txt` (e.g., inode `55101`).
  2. The kernel verifies that the target path resides on the same filesystem.
  3. A new directory entry named `hardlink_file.txt` is written to the destination directory's data block, bound directly to inode `55101`.
  4. The kernel increments the reference counter in inode `55101`: `i_nlink++`.
* **Equal Status**: Neither directory entry is the "master" or "real" file. Both entries are identical, co-equal peers pointing to the exact same inode metadata and physical data blocks.
* **Deletion Semantics**: Invoking `rm hardlink_file.txt` executes the `unlink(2)` system call. The kernel removes the directory entry named `hardlink_file.txt` from the parent directory block and decrements the inode's link count (`i_nlink--`). As long as `i_nlink > 0`, the inode and its underlying data blocks remain completely untouched on disk.

###### Architectural Constraints on Hard Links
1. **Cannot Cross Filesystem Boundaries**:
   Because an inode number is meaningful only within the context of a single filesystem superblock, a hard link cannot point to an inode on another filesystem. Attempting to hard-link across partitions fails with `EXDEV` (Invalid cross-device link):
   ```bash
   $ ln /mnt/diskA/file.txt /mnt/diskB/link.txt
   ln: failed to create hard link '/mnt/diskB/link.txt' => '/mnt/diskA/file.txt': Invalid cross-device link
   ```
2. **Cannot Hard-Link Directories (POSIX Restriction)**:
   Linux forbids ordinary users and even the root superuser from creating hard links to directories.
   * *The Problem*: Permitting arbitrary hard links to directories would transform the filesystem's Directed Acyclic Graph (DAG) into a cyclic graph with loops. 
   * *Consequences*: Recursive traversal utilities (`find`, `du`, `tar`, backup agents) would become trapped in infinite loops. Cyclic links can also desynchronize parent-pointer tracking (`..`), leading to filesystem corruption that standard `fsck` passes cannot resolve safely.
   *(Note: The kernel automatically manages structural directory links: every directory has a link for its name, a link for `.` inside itself, and links for `..` inside each of its subdirectories).*

---

##### Symbolic Links (`symlink(2)`)
A symbolic link (or soft link) creates an entirely new, distinct inode with its file type set to `S_IFLNK`.

* **Mechanism**:
  ```bash
  ln -s /etc/hosts /tmp/hosts_link
  ```
  1. The kernel allocates a brand-new inode (e.g., inode `88492`) with type `S_IFLNK`.
  2. The payload of this new inode stores the **target path string**: `"/etc/hosts"`.
  3. A new directory entry named `hosts_link` is created, pointing to inode `88492`.
* **Fast Symlinks vs. Slow Symlinks**:
  * *Slow Symlink*: If the target path string is long, the filesystem allocates an external data block to store the path string.
  * *Fast Symlink*: If the target path string is shorter than $60\text{ bytes}$ (in Ext4), the filesystem stores the path string directly within the inode body—specifically inside the space normally reserved for extent/block pointers—requiring zero external data block allocations.
* **Resolution and Dereferencing**:
  When a program opens a symlink, the VFS inspects `i_mode`. Recognizing `S_IFLNK`, it reads the stored target string and restarts path resolution using the referent path. This dereferencing behavior can be overridden using specific system call flags (e.g., `O_NOFOLLOW` with `openat(2)`, or using `lstat(2)` instead of `stat(2)`).
* **Dangling (Broken) Symlinks**:
  Because a symlink stores only a textual path string rather than an inode pointer, deleting or renaming the target file does not alter the symlink. The symlink remains on disk pointing to a non-existent path. Any attempt to open a dangling symlink returns `ENOENT` (No such file or directory):
  ```bash
  $ rm /etc/hosts
  $ cat /tmp/hosts_link
  cat: /tmp/hosts_link: No such file or directory
  ```

---

##### Architectural Comparison Matrix
| Architectural Feature | Hard Link | Symbolic Link (Soft Link) |
| :--- | :--- | :--- |
| **Inode Allocation** | Reuses existing target inode | Allocates a brand-new inode (`S_IFLNK`) |
| **Link Count (`i_nlink`)** | Increments target inode's `i_nlink` | Does not alter target inode's `i_nlink` |
| **Cross-Filesystem Scope** | Strictly prohibited (`EXDEV`) | Fully supported (stores plain text path) |
| **Directory Targets** | Strictly prohibited by kernel | Fully supported |
| **Target Deletion Impact** | File data preserved while `i_nlink > 0` | Becomes a broken/dangling symlink |
| **Storage Consumption** | Only consumes parent directory entry space | Consumes an inode + optional data block |
| **Apparent File Size** | Exactly matches target file size | Length of target path string in characters |
| **Permissions Handling** | Mirrors target inode permissions | Inode permissions ignored; target rules apply |
| **System Calls** | `link(2)`, `linkat(2)` | `symlink(2)`, `symlinkat(2)`, `readlink(2)` |

---

#### 4. Mount Lifecycle: Parsing `/etc/fstab`, Mount Options, and `findmnt`
Mounting is the process of attaching a filesystem instance from a physical storage partition, network share, or virtual subsystem to a specific location in the global directory tree (the **mount point**).

##### The Mount Process and the Kernel Namespace
When a filesystem is mounted via `mount(2)` onto an existing directory (e.g., `/mnt/storage`), the VFS updates its internal mount table. The target directory's original dentry is masked by the root dentry of the incoming filesystem. Any subsequent operations targeting that path are redirected to the root inode of the newly mounted storage device.

```
Mount Masking Visualization

Before Mount:
/mnt/storage (dentry) ──────────> Inode 4012 (Local Ext4 root disk)
                                   ├── Payload: Empty directory on root

Execution: mount /dev/sdb1 /mnt/storage

After Mount:
/mnt/storage (dentry) ──[MASKED]
         │
         └─(VFS Redirect)───────> Inode 2 (Superblock Root of /dev/sdb1)
                                   ├── Payload: Actual contents of sdb1
```
*(Note: If the directory `/mnt/storage` contained files prior to the mount, those files are not overwritten or deleted. They remain on the underlying parent filesystem, invisible and unreachable until the mount is detached via `umount`).*

---

##### Mount Namespaces and Propagation Types
Linux separates mount hierarchies using **mount namespaces** (`CLONE_NEWNS`). A process in an isolated mount namespace can mount or unmount filesystems without affecting other processes on the host.

To govern how mount events propagate across namespaces, the kernel supports four explicit mount propagation flags:
1. **`shared`**: Mount or unmount events within this mount point propagate bidirectionally. If a filesystem is mounted under a shared mount in one namespace, it automatically appears across all matching namespaces.
2. **`slave`**: Propagation is unidirectional. Mount events from a shared parent propagate down to this slave mount, but mounts created inside this slave namespace do not propagate back up to the parent.
3. **`private`**: Completely isolated. No mount or unmount events propagate in or out.
4. **`unbindable`**: A private mount that cannot be duplicated or cloned via bind mounts, preventing recursive bind-mount explosions.

##### Bind Mounts (`mount --bind`)
A bind mount aliases an existing directory tree onto another directory location without creating a separate block filesystem:
```bash
# Alias /var/log into /srv/jail/var/log:
mount --bind /var/log /srv/jail/var/log

# Make the bind mount strictly read-only:
mount -o remount,ro,bind /srv/jail/var/log
```

---

##### Parsing `/etc/fstab`
The `/etc/fstab` configuration file dictates how system storage partitions, network shares, and memory filesystems are verified and mounted during system boot.

Each un-commented entry consists of **six white-space-delimited fields**:
```
# <fs_spec>               <mount_point>   <fs_type>   <options>          <dump>  <pass>
UUID=3a1b2c-4d5e...       /               ext4        defaults,noatime   0       1
UUID=9f8e7d-6c5b...       /home           xfs         defaults,nodev     0       2
192.168.1.50:/exports     /mnt/nfs        nfs4        _netdev,ro,soft    0       0
tmpfs                     /tmp            tmpfs       defaults,size=2G   0       0
```

###### Field Breakdown:
1. **Device Specification (`<fs_spec>`)**:
   Specifies the underlying block storage device. While direct kernel paths can be used (e.g., `/dev/sda1`), modern systems use persistent storage identifiers to prevent boot failures caused by non-deterministic device enumeration:
   * **`UUID=`**: Universally Unique Identifier assigned to the filesystem metadata during `mkfs`. Stable across controller and cabling changes.
   * **`PARTUUID=`**: Partition UUID assigned to the partition table entry (GPT standard). Remains valid even if the partition has not yet been formatted.
   * **`LABEL=`**: Human-readable volume label assigned to the filesystem (e.g., `LABEL=DataVault`).
2. **Mount Point (`<mount_point>`)**:
   The absolute target directory where the filesystem will be attached. For swap space, this field is set to `none` or `swap`.
3. **Filesystem Type (`<fs_type>`)**:
   Specifies the driver used to mount the filesystem: `ext4`, `xfs`, `btrfs`, `vfat`, `nfs`, `cifs`, `iso9660`, `tmpfs`. Setting this to `auto` causes the kernel to probe available drivers against the partition's magic byte headers using `libblkid`.
4. **Mount Options (`<options>`)**:
   A comma-delimited string of filesystem parameters and security flags.
5. **Dump Flag (`<dump>`)**:
   A legacy binary flag (`0` or `1`) indicating whether the filesystem should be backed up by the archaic Unix `dump` utility. Almost universally set to `0` on modern systems.
6. **Pass Number (`<pass>`)**:
   Determines the execution order for filesystem checks performed by `fsck` at boot time:
   * `1`: Reserved strictly for the root filesystem (`/`), ensuring it is verified and repaired first.
   * `2`: Secondary filesystems (e.g., `/home`, `/var`) are checked after the root filesystem passes. Filesystems on independent physical drives are verified in parallel.
   * `0`: Disables boot-time `fsck` checks entirely. Used for memory-backed mounts (`tmpfs`), network storage (`nfs`), and copy-on-write filesystems that handle their own consistency checks (e.g., Btrfs, ZFS).

---

##### Critical Mount Options
Mount options balance performance, system stability, and access control:

###### Standard POSIX and Security Options
* **`defaults`**: Applies standard default settings: `rw`, `suid`, `dev`, `exec`, `auto`, `nouser`, `async`.
* **`ro` / `rw`**: Mounts the volume as read-only or read-write.
* **`noexec`**: Prohibits the direct execution of binary executables located on the volume. Any attempt to invoke a binary returns `EACCES` (Permission denied). *Critical for untrusted directories such as `/tmp` or `/var/tmp`.*
* **`nosuid`**: Blocks the operation of Set-User-ID (`SUID`) and Set-Group-ID (`SGID`) execution bits. Prevents unprivileged users from executing local root-escalation binaries stored on removable media or network shares.
* **`nodev`**: Prohibits the kernel from interpreting character or block device nodes on the filesystem. Protects against attacks where an unprivileged user creates a raw disk device node inside `/tmp` via `mknod` to bypass standard file permissions.

###### Timestamp Optimization Options
Updating `atime` on every single read operation turns read-only workloads into write operations, introducing significant I/O overhead. Linux provides several options to control this behavior:
* **`strictatime`**: Forces the kernel to strictly update `atime` on every single file read, fully complying with legacy POSIX standards at the expense of storage performance.
* **`noatime`**: Disables `atime` updates across the filesystem entirely. Provides maximum I/O performance gains, but can break legacy mail user agents (e.g., `mutt`) that rely on `atime` to detect unread emails.
* **`nodiratime`**: Disables `atime` updates strictly for directory inodes while preserving them for regular files.
* **`relatime` (Relative Access Time)**: **The Linux default since kernel 2.6.30.** The kernel updates `atime` only if the previous `atime` is less than or equal to the current `mtime` or `ctime`, or if the existing `atime` is more than 24 hours old. This satisfies applications that need to know whether a file was read since it was last modified, while eliminating up to $95\%$ of write operations caused by `atime` tracking.

###### Journaling and Durability Options (Ext4)
* **`data=journal`**: Full metadata and payload journaling. All file payload data is written to the journal before being committed to persistent storage. Offers the highest data durability across power outages, but incurs significant write performance penalties.
* **`data=ordered` (Default)**: Metadata changes are journaled, but file payload data is written directly to the filesystem storage blocks before the corresponding metadata transaction is committed to the journal.
* **`data=writeback`**: Metadata is journaled, but payload write ordering is relaxed. Data blocks may be committed after metadata transactions complete, maximizing throughput at the risk of exposing stale data blocks if the system crashes during an uncompleted write.
* **`barrier=1` / `barrier=0`**: Enables or disables write barriers. Barriers instruct disk controllers to flush hardware write caches to non-volatile platters/cells before proceeding with dependent journal writes. Disabling barriers (`barrier=0`) improves write throughput on systems equipped with battery-backed write caches (BBWC), but risks catastrophic filesystem corruption on consumer drives if power fails abruptly.

###### Network and Systemd Integration Options
* **`_netdev`**: Prevents the operating system from attempting to mount the filesystem until network interfaces and routing services are fully initialized, avoiding boot hangs on NFS, iSCSI, or CIFS shares.
* **`x-systemd.automount`**: Configures systemd to create an on-demand automount point. The physical storage or network share is mounted transparently only when an application first attempts to access the directory path.
* **`nofail`**: Instructs the boot manager to continue system startup even if the target storage volume is missing or fails to mount.

---

##### Inspecting Mounts via `findmnt`
While legacy administrators often inspect `/etc/mtab` or `/proc/mounts`, the utility of choice for modern Linux system administration is `findmnt` (part of `util-linux`). It parses the kernel's real-time mount tree directly from `/proc/self/mountinfo`.

```bash
# 1. Display the hierarchical mount tree:
$ findmnt
TARGET                       SOURCE       FSTYPE      OPTIONS
/                            /dev/nvme0n1p2 ext4      rw,relatime,errors=remount-ro
├─/sys                       sysfs        sysfs       rw,nosuid,nodev,noexec,relatime
│ ├─/sys/kernel/security     securityfs   securityfs  rw,nosuid,nodev,noexec,relatime
│ └─/sys/fs/cgroup           cgroup2      cgroup2     rw,nosuid,nodev,noexec,relatime
├─/proc                      proc         proc        rw,nosuid,nodev,noexec,relatime
├─/dev                       udev         devtmpfs    rw,nosuid,relatime,size=16334864k
│ └─/dev/pts                 devpts       devpts      rw,nosuid,noexec,relatime,mode=620
├─/run                       tmpfs        tmpfs       rw,nosuid,nodev,noexec,relatime
└─/boot                      /dev/nvme0n1p1 vfat      rw,relatime,fmask=0077,dmask=0077

# 2. Query the exact mount point responsible for a specific path:
$ findmnt -T /var/log/journal
TARGET SOURCE         FSTYPE OPTIONS
/      /dev/nvme0n1p2 ext4   rw,relatime,errors=remount-ro

# 3. Filter mounts by explicit filesystem type:
$ findmnt -t ext4,xfs

# 4. Search for mounts violating specific security postures:
$ findmnt --real -O noexec

# 5. Emit script-friendly JSON output:
$ findmnt -J
```

---

#### 5. Disk Usage Analysis: Apparent Size vs. Allocated Blocks (`du`, `df`)
A frequent challenge for systems engineers is diagnosing discrepancies between file sizes and available storage capacity. Two standard utilities—`du` (Disk Usage) and `df` (Disk Free)—rely on fundamentally different measurement mechanisms, leading to situations where their reported values diverge dramatically.

##### Apparent Size vs. Allocated Disk Blocks
Every file is defined by two distinct size metrics:
1. **Apparent Size (`stat.st_size`)**:
   The exact number of bytes contained in the file, from byte offset $0$ up to the final byte index. This determines the End-Of-File (EOF) marker for read operations.
2. **Allocated Disk Blocks (`stat.st_blocks`)**:
   The actual physical storage capacity consumed on disk by the file's data blocks and metadata structures, calculated in $512\text{-byte}$ sectors.

```
Discrepancy 1: Allocation Slack Space
┌──────────────────────────────────────┐
│ Physical File System Block (4,096 B) │
├─────────────────────────┬────────────┤
│ Actual Payload: 120 B   │ Unused     │
│ (Apparent Size)         │ Slack      │
└─────────────────────────┴────────────┘
Result: Apparent Size = 120 Bytes; Allocated Size = 4,096 Bytes

Discrepancy 2: Sparse File (Hole Allocation)
File Offset: 0               1 GiB                                 1 GiB + 4 KiB
Logical:    [ Extent 0: 4 KiB ] [ Unallocated Zero Hole: ~1 GiB ]    [ Extent 1: 4 KiB ]
Physical:   [ Disk Block A    ] <No Blocks Allocated on Medium>    [ Disk Block B    ]
Result: Apparent Size = 1.000008 GiB; Allocated Size = 8 KiB
```

###### Cause 1: Allocation Block Size and Slack Space
Standard filesystems do not allocate raw bytes on physical media; they allocate space in fixed chunks called **blocks** (typically $4\,096\text{ bytes} = 4\text{ KiB}$). If a file contains only $10\text{ bytes}$ of data, the filesystem must allocate an entire $4\text{ KiB}$ block to house it. The remaining $4\,086\text{ bytes}$ is unallocated **slack space**.
* A directory containing $100\,000$ files of $10\text{ bytes}$ each has an aggregate apparent size of only $\approx 1\text{ MB}$, but consumes $100\,000 \times 4\text{ KiB} = 400\text{ MB}$ of physical storage.

###### Cause 2: Sparse Files
A sparse file contains long stretches of zero bytes that are not written to physical disk blocks. 
* *Mechanism*: If an application opens a new file and calls `lseek(fd, 1073741824, SEEK_SET)` (jumping $1\text{ GiB}$ forward) before writing $4\text{ KiB}$ of data, the filesystem updates the inode's `i_size` to $1\text{ GiB} + 4\text{ KiB}$. However, because no data was written to the intervening space, the filesystem allocates physical blocks **only** for the final $4\text{ KiB}$ extent. The unwritten gap is tracked as an unallocated hole.
* When reading from a hole, the VFS automatically returns streams of zero bytes ($0\text{x}00$) directly from memory without reading from disk.
* *Inspection*:
  ```bash
  $ ls -lhs sparse.img
  8.0K -rw-r--r-- 1 root root 1.1G Sep 27 02:00 sparse.img
  # ^Allocated Size           ^Apparent Size
  ```

---

##### Operational Mechanics: `du` vs. `df`
The differences between `du` and `df` stem from their underlying system calls and traversal algorithms:

```
┌──────────────────────────────────────────────┐  ┌──────────────────────────────────────────────┐
│           du (Disk Usage Engine)             │  │            df (Disk Free Engine)             │
└──────────────────────┬───────────────────────┘  └──────────────────────┬───────────────────────┘
                       │                                                 │
                       ▼                                                 ▼
        Walks directory tree (fts / fstatat)                Direct statvfs(2) System Call
                       │                                                 │
                       ▼                                                 ▼
        Parses dentries sequentially                      Queries superblock directly:
        - Resolves filenames to inodes                    - Total data blocks on device
        - Tracks visited inodes (no double-count)         - Free block bitmap counters
        - Sums allocated st_blocks                        - Available blocks to non-root users
                       │                                                 │
                       ▼                                                 ▼
        Reports: Storage consumed by                      Reports: Total physical capacity and
        ACCESSIBLE DIRECTORY ENTRIES                      global usage at the BLOCK DEVICE LEVEL
```

###### `du` Mechanics
`du` traverses the directory hierarchy recursively starting from a target path. It calls `lstat(2)` or `fstatat(2)` on every directory entry, tracks visited inode numbers in a hash table (to ensure files with multiple hard links are counted only once), and sums the allocated blocks (`stat.st_blocks` converted to bytes).
* **Limitation**: `du` can account only for files associated with **reachable, unmasked directory entries** that the calling user possesses permissions to read and traverse.

###### `df` Mechanics
`df` ignores directory trees, path names, and permissions entirely. It calls `statvfs(2)` or `statfs(2)` on the mount point, reading filesystem counters directly from the in-memory superblock:
* `f_blocks`: Total data blocks in the filesystem.
* `f_bfree`: Total free blocks remaining.
* `f_bavail`: Free blocks available to unprivileged users.
* **The Root Reservation Reserve**: On Ext4 filesystems, by default $5\%$ of total block capacity is reserved strictly for administrative processes running under UID 0 (`root`). This prevents unprivileged user processes from completely filling the filesystem, which could prevent system daemons (like `sshd` or `syslogd`) from writing state files or logging in administrators. As a result, `df` often shows a filesystem as $100\%$ full for standard users even when $5\%$ of disk capacity remains in reserve.

---

##### The "Ghost File" Disparity: `df` Shows 100% Full, `du` Finds Nothing
A classic Linux troubleshooting scenario occurs when `df` alerts that a partition is completely full ($100\%$ capacity consumed), yet running `du -sh /*` reveals only a fraction of that storage space in use.

```
The Open-but-Unlinked Inode Dilemma

Step 1: Long-running process (Nginx/Java, PID 4120) writes to /var/log/app.log
   Directory Entry "/var/log/app.log" ──> Inode 99120 ──> Data Blocks (50 GiB)
   Process File Descriptor (FD 3)   ────┘
   Inode 99120: i_nlink = 1, f_count = 1

Step 2: Administrator deletes file to clear space: rm /var/log/app.log
   Directory Entry is REMOVED from disk directory block.
   Inode 99120: i_nlink = 0, BUT f_count = 1 (Process holds open FD)

Step 3: Space Accounting Disparity
   ├── du walks /var/log: Cannot find the entry "app.log". Counts 0 Bytes.
   └── df queries superblock: Inode 99120 has NOT been freed.
       Data Blocks (50 GiB) REMAIN ALLOCATED. df shows 100% Full.
```

###### Root Cause Analysis
In POSIX filesystems, freeing a file's storage blocks requires two independent criteria to be met:
1. The inode's hard link reference counter must reach zero (`i_nlink == 0`), meaning all directory entry references to the file have been removed via `unlink(2)`.
2. The kernel's open-file reference counter must reach zero (`f_count == 0`), meaning every process holding an open file descriptor pointing to that inode has closed its descriptor or terminated.

When an administrator runs `rm` on a massive log file while a process (such as a database or web server) is actively writing to it, the `rm` command removes the directory entry and decrements `i_nlink` from $1$ to $0$. 

However, because the running process still holds an open file descriptor (`f_count >= 1`), **the kernel does not return the allocated data blocks to the filesystem's free block bitmap**. The file becomes a "ghost file":
* `du` cannot find the file because it traverses directory entries, none of which point to the unlinked inode.
* `df` queries the superblock counters, which still reflect the data blocks allocated to the unlinked inode.

---

###### Diagnostic and Remediation Workflow
Never reboot a production server or kill critical processes blindly to clear ghost files. You can identify and recover the leaked space online:

###### Step 1: Identify the Unlinked Open Inodes via `lsof`
Use `lsof` to locate deleted files that remain held open by running processes:
```bash
# Locate all open files with link count = 0:
sudo lsof +L1

# Alternatively, search for '(deleted)' descriptors:
sudo lsof | grep deleted
```
*Sample Output*:
```
COMMAND   PID USER   FD   TYPE DEVICE   SIZE/OFF  NODE NAME
nginx    4120  www    3w   REG  259,2 53687091200 99120 /var/log/nginx/access.log (deleted)
```
The output confirms PID `4120` holds file descriptor `3` open to an unlinked inode consuming $53\text{ GB}$ of space.

###### Step 2: Clear the Disk Blocks in Flight (Zero-Truncation)
If you cannot restart the application immediately, you can truncate the file payload directly through the kernel's `/proc` virtual interface:
```bash
# Truncate the file payload to 0 bytes using its open file descriptor:
: > /proc/4120/fd/3
```
*Mechanism*: The shell opens the target file descriptor via the `/proc/<PID>/fd/` virtual path with the `O_TRUNC` flag. The kernel immediately truncates the underlying inode's payload to $0\text{ bytes}$ and returns the allocated data blocks to the filesystem's free block pool, resolving the storage exhaustion alert instantly without disrupting the running daemon.

###### Step 3: Permanently Release the File Handle
Instruct the daemon to close its stale file handle and open a new log file using standard POSIX signal signaling (e.g., `SIGHUP` or log rotation signals):
```bash
# Signal Nginx to reopen log files cleanly:
kill -USR1 4120
```

---

#### 6. Practical Laboratory: Low-Level Storage Inspections
These hands-on labs explore inode allocation limits, sparse file creation, hole-punching mechanics, and the mount lifecycle.

##### Lab 1: Simulating Inode Exhaustion
Create an isolated loopback filesystem to observe what happens when a filesystem runs out of inodes before running out of block storage.

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR="/tmp/inode_lab"
mkdir -p "${WORKDIR}"
cd "${WORKDIR}"

echo "[*] Creating a 50 MiB backing storage image..."
dd if=/dev/zero of=storage.img bs=1M count=50 status=none

echo "[*] Formatting filesystem with a restricted inode count (1000 inodes)..."
# -N overrides default inode ratios, forcing exactly 1000 inodes:
mkfs.ext4 -N 1000 -F storage.img

mkdir -p mnt
sudo mount -o loop storage.img mnt

echo "[*] Checking filesystem block vs inode capacity:"
df -h mnt
df -i mnt

echo "[*] Attempting to create 1,200 tiny files to exhaust the inode table..."
set +e
for i in $(seq 1 1200); do
    if ! touch "mnt/file_${i}.txt" 2>/dev/null; then
        echo "[!] Touch failed at file index: ${i}"
        break
    fi
done
set -e

echo "[*] Diagnostic validation of exhaustion state:"
df -h mnt  # Shows plenty of free block storage remaining!
df -i mnt  # Shows 100% Inodes consumed (IFree = 0)

echo "[*] Cleaning up lab environment..."
sudo umount mnt
cd /tmp
rm -rf "${WORKDIR}"
```

---

##### Lab 2: Sparse File Creation, Hole Punching, and Allocation Analysis
Examine how sparse files are structured, compare their apparent sizes with their actual disk allocations, and punch holes in them dynamically using `fallocate`.

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR="/tmp/sparse_lab"
mkdir -p "${WORKDIR}"
cd "${WORKDIR}"

echo "[*] 1. Creating a sparse file using seek/truncate..."
# Seek 1 GiB ahead and write a single 4-byte string:
dd if=/dev/zero of=sparse.bin bs=1 count=0 seek=1G status=none
echo "END" >> sparse.bin

echo "[*] 2. Comparing apparent size vs actual allocated block storage:"
ls -lh sparse.bin
du -h sparse.bin
du -h --apparent-size sparse.bin

echo "[*] 3. Creating a completely dense, preallocated 100 MiB file..."
fallocate -l 100M dense.bin
du -h dense.bin
du -h --apparent-size dense.bin

echo "[*] 4. Punching an unallocated hole in the dense file using fallocate..."
# Deallocate a 50 MiB range from offset 10 MiB to 60 MiB:
fallocate -p -o 10M -l 50M dense.bin

echo "[*] 5. Inspecting the file post hole-punching:"
ls -lh dense.bin        # Apparent size remains completely unchanged (100 MiB)
du -h dense.bin         # Allocated size drops to 50 MiB!

echo "[*] Cleaning up lab environment..."
cd /tmp
rm -rf "${WORKDIR}"
```

---

##### Lab 3: Isolating Mount Namespaces and Restricting Mount Flags
Demonstrate how security mount flags (`noexec`, `nosuid`, `nodev`) neutralize attack vectors on untrusted filesystems.

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR="/tmp/mount_lab"
mkdir -p "${WORKDIR}"
cd "${WORKDIR}"

echo "[*] Creating small filesystem image..."
dd if=/dev/zero of=mount_test.img bs=1M count=30 status=none
mkfs.ext4 -F mount_test.img >/dev/null

mkdir -p mnt
# Mount with strict security flags:
sudo mount -o loop,noexec,nosuid,nodev mount_test.img mnt

echo "[*] Compiling a simple C test binary directly inside the secured mount..."
cat << 'EOF' > mnt/test_exec.c
#include <stdio.h>
int main() {
    printf("[+] Binary executed successfully!\n");
    return 0;
}
EOF
sudo gcc mnt/test_exec.c -o mnt/test_exec
sudo chmod +x mnt/test_exec

echo "[*] Verifying noexec flag enforcement by attempting execution:"
set +e
mnt/test_exec
EXIT_STATUS=$?
set -e

if [ ${EXIT_STATUS} -ne 0 ]; then
    echo "[SUCCESS] The kernel denied execution (Status: ${EXIT_STATUS}) due to noexec!"
fi

echo "[*] Inspecting mount options with findmnt:"
findmnt -T mnt/test_exec

echo "[*] Cleaning up mount lab..."
sudo umount mnt
cd /tmp
rm -rf "${WORKDIR}"
```

---

#### 7. Comprehensive Storage Reference Matrix
Use this matrix as an operational reference for filesystem design, storage diagnostics, and system auditing:

| Operation / Concept | Underlying System Call | Target Data Structure | Key Command-Line Tools | Operational Hazards / Failure Modes |
| :--- | :--- | :--- | :--- | :--- |
| **Path Traversal** | `openat(2)`, `statx(2)` | `dentry` $\to$ `inode` | `ls`, `stat`, `file` | Cache-miss performance drops, invalid path component permissions (`-x`). |
| **Hard Link Creation** | `link(2)`, `linkat(2)` | Increments `inode.i_nlink` | `ln file link` | Fails across filesystems (`EXDEV`); disallowed on directories to prevent graph cycles. |
| **Symlink Creation** | `symlink(2)` | Allocates `S_IFLNK` inode | `ln -s target link` | Dangling symlinks (`ENOENT`) if target is moved or deleted. |
| **File Deletion** | `unlink(2)`, `rmdir(2)` | Removes `dentry`, drops `i_nlink` | `rm`, `unlink` | Blocks remain allocated if an active process holds the descriptor open. |
| **Filesystem Mount** | `mount(2)` | `super_block`, `dentry` | `mount`, `findmnt` | Target directory contents are masked while mounted; risks lockups on inaccessible `_netdev`. |
| **Filesystem Unmount** | `umount2(2)` | Detaches VFS mount node | `umount`, `umount -l` | Returns `EBUSY` if file descriptors or processes are active in directory tree. |
| **Sparse Hole Punch** | `fallocate(FALLOC_FL_PUNCH_HOLE)` | Frees blocks/extents | `fallocate -p` | Unsupported on legacy filesystems; requires underlying driver support. |
| **Per-File Space Audit** | `lstat(2)` | Sums `inode.i_blocks` | `du`, `ncdu` | Inaccurate if files are unlinked while held open; slow on massive directory trees. |
| **Global Capacity Audit** | `statvfs(2)` | Reads `super_block` | `df -h`, `df -i` | Diverges from `du` due to root reserves, open-deleted files, and unmounted paths. |