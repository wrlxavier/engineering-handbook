# Memory Management and the Swap Subsystem

The Linux memory management subsystem mediates between userspace execution requests and hardware storage limits. It abstracts physical random-access memory (RAM) through hardware-enforced virtual memory addressing, organizes cache hierarchies to minimize I/O latency, balances volatile memory against non-volatile swap storage, and enforces containment through resource limits and memory reclamation protocols.

Operating a production Linux system requires understanding these mechanics: how virtual addresses resolve through hardware page tables, how the kernel calculates true memory availability, why naive process memory accounting leads to faulty capacity planning, and how the Out-Of-Memory (OOM) Killer evaluates workloads under exhaustion.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Virtual Address Space                           │
│  [User Space: 0x0000000000000000 - 0x00007FFFFFFFFFFF] (128 TiB)       │
│  [Non-Canonical Addressing Gap / Safety Hole]                          │
│  [Kernel Space: 0xFFFF800000000000 - 0xFFFFFFFFFFFFFFFF] (128 TiB)     │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Hardware Paging (CR3 / 4-Level Paging)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   Memory Management Unit (MMU) & TLB                   │
│         Translates Virtual Addresses to Physical Page Frames           │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        Physical Memory (DRAM)                          │
│  ┌───────────────────────┐  ┌──────────────────────────────────────┐  │
│  │     Active Memory     │  │          Reclaimable Memory          │  │
│  │  - Anonymous Pages    │  │  - Inactive File Pages (Page Cache)  │  │
│  │  - Unreclaimable Slab │  │  - Inactive Dirty Buffers            │  │
│  │  - Kernel Stacks/Data │  │  - Reclaimable Slab (dentry/inode)   │  │
│  └───────────────────────┘  └──────────────────────────────────────┘  │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │                                │
    Memory Pressure │ kswapd Reclaim / Compaction    │ Writeback Engine
                    ▼                                ▼
┌───────────────────────────────────┐  ┌─────────────────────────────────┐
│           Swap Subsystem          │  │     Non-Volatile Storage        │
│  - Swap Partitions / Files        │  │     (NVMe / SSD / SATA HDD)     │
│  - ZSWAP (Compressed RAM Pool)    │  │  - Backing filesystems (ext4)   │
│  - Anonymously Swapped Pages      │  │  - Raw block storage blocks     │
└───────────────────────────────────┘  └─────────────────────────────────┘
```

---

## 1. Memory Topology and Physical Memory Management

Modern architectures decouple the physical location of memory cells from the address spaces accessed by processor instructions. Virtual memory provides hardware-enforced isolation, dynamic allocation, overcommit capabilities, and memory-mapped persistence.

### Virtual Memory, Paging, and Page Frames

The fundamental unit of memory managed by the Linux kernel is the **Page Frame** (in physical memory) and the **Page** (in virtual memory). On the x86-64 architecture:
* Standard base page size is $4\,096\text{ bytes}$ ($4\text{ KiB}$).
* Huge pages are allocated at $2\text{ MiB}$ (Level 2 page directory entries) and $1\text{ GiB}$ (Level 3 page directory entries).

```
                      Virtual Address (48-bit Canonical)
 47             39 38             30 29             21 20             12 11          0
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┬─────────────┐
│   PGD Index     │   P4D/PUD Index │    PMD Index    │    PTE Index    │ Page Offset │
│    (9 bits)     │    (9 bits)     │    (9 bits)     │    (9 bits)     │  (12 bits)  │
└────────┬────────┴────────┬────────┴────────┬────────┴────────┬────────┴──────┬──────┘
         │                 │                 │                 │               │
         ▼                 ▼                 ▼                 ▼               │
    Page Global       Page Upper        Page Middle       Page Table           │
     Directory         Directory         Directory          Entry              │
       (PGD)             (PUD)             (PMD)            (PTE)              │
                                                               │               │
                                                               ▼               ▼
                                                     Physical Page Frame + Offset
```

When an instruction dereferences a virtual address, the processor's **Memory Management Unit (MMU)** walks a 4-level (or 5-level with paging-57) page table tree anchored by the physical address stored in the `CR3` control register:
1. **PGD (Page Global Directory)**: Bits 47–39 index the top-level table.
2. **PUD (Page Upper Directory)**: Bits 38–30 index the second-level table.
3. **PMD (Page Middle Directory)**: Bits 29–21 index the third-level table.
4. **PTE (Page Table Entry)**: Bits 20–12 index the leaf page table containing the physical page frame base address.
5. **Physical Offset**: Bits 11–0 index the exact byte within the $4\text{ KiB}$ frame ($2^{12} = 4\,096$).

To prevent multi-cycle memory lookups on every single dereference, the CPU caches translated addresses inside the **Translation Lookaside Buffer (TLB)**. A TLB miss forces a hardware page-table walk across system buses; a TLB hit resolves the physical frame in sub-nanosecond clock cycles.

### Memory Zones and NUMA

Linux groups physical memory into **Zones** based on hardware addressability and device access constraints:

| Zone | Addressing Limit | Architectural Purpose |
| :--- | :--- | :--- |
| `ZONE_DMA` | First $16\text{ MiB}$ | Preserved for legacy ISA hardware devices requiring 24-bit direct memory access addressing. |
| `ZONE_DMA32` | $16\text{ MiB}$ to $4\text{ GiB}$ | Allocates memory for legacy PCI devices limited to 32-bit physical bus addresses on 64-bit systems. |
| `ZONE_NORMAL` | $4\text{ GiB}$ to Max Physical | Directly mapped address space of standard physical RAM accessible to all regular kernel and user operations. |
| `ZONE_HIGHMEM` | 32-bit platforms only | Physical memory beyond $896\text{ MiB}$ that cannot be continuously mapped into the kernel's virtual space. Absent on 64-bit kernels. |
| `ZONE_MOVABLE` | Dynamic / Hot-plug | Zone containing hot-pluggable memory pages that can be dynamically relocated or migrated to support hot-unplug. |
| `ZONE_DEVICE` | Persistent / NVDIMM | Maps Non-Volatile Memory (NVDIMM, DAX) and external PCI accelerator memory frames outside standard page accounting. |

On modern multi-socket hardware, systems employ **Non-Uniform Memory Access (NUMA)**. Physical memory is physically attached to distinct CPU sockets (NUMA Nodes). Accessing memory attached to a local socket is faster than routing requests across interconnect fabrics (Intel UPI or AMD Infinity Fabric) to access remote nodes:

```bash
# Display system NUMA node topology and memory splits:
numactl --hardware
```

```text
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7
node 0 size: 64182 MB
node 0 free: 1240 MB
node 1 cpus: 8 9 10 11 12 13 14 15
node 1 size: 64512 MB
node 1 free: 8432 MB
node distances:
node   0   1 
  0:  10  21 
  1:  21  10 
```

The memory distance matrix represents the latency multiplier: local node access has a baseline distance metric of `10`, while remote node access imposes a `21` metric ($2.1\times$ latency penalty).

---

### Dissecting `/proc/meminfo`

The primary diagnostic interface for kernel-wide memory allocation is `/proc/meminfo`. It reflects the internal state of the kernel's zone allocators, page-allocator tracking structures, and slab caches:

```bash
$ cat /proc/meminfo | head -n 25
```

```text
MemTotal:       32649188 kB
MemFree:         1204856 kB
MemAvailable:   24891240 kB
Buffers:          341028 kB
Cached:         23112456 kB
SwapCached:        12480 kB
Active:         14890212 kB
Inactive:        9812400 kB
Active(anon):    1248920 kB
Inactive(anon):   542100 kB
Active(file):   13641292 kB
Inactive(file):  9270300 kB
Unevictable:       18420 kB
Mlocked:           18420 kB
SwapTotal:       8388604 kB
SwapFree:        8104200 kB
Dirty:              1140 kB
Writeback:             0 kB
AnonPages:       1778540 kB
Mapped:           412092 kB
Shmem:             24896 kB
KReclaimable:     891240 kB
Slab:            1241080 kB
SReclaimable:     891240 kB
SUnreclaim:       349840 kB
```

#### Line-by-Line Subsystem Breakdown

* **`MemTotal`**: Total usable physical RAM recognized by the kernel, excluding the kernel's own static code image, firmware data reservations, and early architectural reserve pages.
* **`MemFree`**: The sum of completely unallocated, idle memory pages. This memory is not performing any productive work (neither program storage nor cache).
* **`MemAvailable`**: An estimate of how much memory is available for starting new applications without forcing the system into swap thrashing.
* **`Buffers`**: Temporary in-memory block storage for raw disk blocks, device sector caching, and filesystem metadata operations.
* **`Cached`**: The VFS Page Cache. Holds file payloads read from or written to persistent filesystems.
* **`SwapCached`**: Memory that once resided in swap, has been read back into physical RAM, but still exists concurrently on the swap storage device.
* **`Active` vs `Inactive`**:
  * `Active`: Pages referenced recently; protected from immediate eviction by the page replacement engine.
  * `Inactive`: Pages not accessed across recent scanning passes; prime candidates for eviction or writing to swap.
* **`Active(anon)` / `Inactive(anon)`**: Memory pages bound to process heap, stack, anonymous `mmap` regions, and runtime state. Eviction requires writing to swap space.
* **`Active(file)` / `Inactive(file)`**: Memory pages backed by persistent filesystem storage. Eviction requires discarding (if clean) or flushing to disk (if dirty).
* **`Unevictable` / `Mlocked`**: Pages locked into physical memory via `mlock()`, `mlockall()`, or kernel driver pins that cannot be paged out or evicted under any memory pressure.
* **`Dirty`**: Memory pages in the Page Cache that have been modified by userspace write operations but have not yet been synchronized to the underlying non-volatile block storage.
* **`Writeback`**: Pages actively in transit across storage controller buses, currently being serialized to physical media.
* **`AnonPages`**: Anonymous pages mapped directly into userspace process page tables.
* **`Mapped`**: Files mapped directly into process address spaces via `mmap()` (such as shared application libraries, `.so` objects, and direct file mappings).
* **`Shmem`**: Memory utilized by POSIX shared memory allocations (`shmget`, `shm_open`) and volatile `tmpfs` mounts.
* **`Slab` / `SReclaimable` / `SUnreclaim`**:
  * `Slab`: Memory consumed by the kernel's internal object caches (`dentry`, `inode`, `task_struct`, `buffer_head`).
  * `SReclaimable`: Portions of the slab cache (primarily dcache and icache) that can be reclaimed by the VFS shrinker under memory pressure.
  * `SUnreclaim`: Kernel object allocations that cannot be reclaimed while the kernel or holding drivers are running.

---

### The `MemFree` vs `MemAvailable` Paradigm

A legacy systems misconception is that low `MemFree` indicates a machine running out of memory:

$$\text{Incorrect Legacy Assertion: } \text{Available Capacity} = \text{MemFree}$$

Linux follows the design principle: **Unused RAM is wasted RAM.** If physical pages are not actively allocated to application stacks and heaps, the kernel uses them as **Page Cache** to eliminate storage I/O latency. If an application suddenly demands memory, the kernel evicts clean cached file pages to satisfy the request.

Prior to Linux kernel version `3.14`, administrators approximated usable memory via the formula:

$$\text{Legacy Usable Estimate} \approx \text{MemFree} + \text{Buffers} + \text{Cached}$$

This approximation was flawed. It assumed that *all* Page Cache and Buffers could be reclaimed. In practice:
1. `Shmem` (shared memory and `tmpfs`) is aggregated into the `Cached` metric, but cannot be dropped without swap.
2. A portion of the page cache is dirty and cannot be dropped instantly without writeback latency.
3. Every memory zone enforces a non-zero **Low Watermark** (`watermark[WMARK_LOW]`), a floor of reserved physical frames that the kernel will not surrender to userspace to avoid deadlocking interrupt allocations.

Starting in Linux 3.14, the kernel exports `MemAvailable`. The kernel source (`fs/proc/task_mmu.c` and `mm/page_alloc.c`) estimates `MemAvailable` via this operational algorithm:

$$\text{MemAvailable} = \text{MemFree} - \sum \text{LowWatermarks} + \left( \text{PageCache}_{\text{reclaimable}} - \min\left(\frac{\text{PageCache}_{\text{reclaimable}}}{2}, \sum \text{LowWatermarks}\right) \right) + \text{SReclaimable} - \min\left(\frac{\text{SReclaimable}}{2}, \sum \text{LowWatermarks}\right)$$

Where:
* The kernel calculates total free pages minus the architectural low watermarks across all system zones.
* It identifies reclaimable file-backed pages (`Active(file)` + `Inactive(file)`), subtracts dirty pages, and caps the reclaim potential against zone reservations.
* It evaluates `SReclaimable` (dentries and inodes), applying a $50\%$ safety margin for metadata that resists immediate purging.

If `MemAvailable` nears zero, the machine is approaching true memory exhaustion, regardless of how small or large `MemFree` appears.

---

### Buffers vs Page Cache

Though often conflated, Buffers and the Page Cache operate at distinct architectural abstraction levels:

```
┌────────────────────────────────────────────────────────────────────────┐
│                   Userspace File System Operations                     │
│                read(), write(), mmap(), pread(), pwrite()              │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        VFS Page Cache Layer                            │
│  - Organizes data by `struct address_space` & File Inode Offset        │
│  - Tracks file content in 4 KiB memory pages                           │
│  - Completely decoupled from physical disk sector layouts              │
└───────────────────┬────────────────────────────────┬───────────────────┘
                    │                                │
    Direct File I/O │ Metadata / Block Operations    │ Raw Block Access
                    ▼                                ▼
┌───────────────────────────────────┐  ┌─────────────────────────────────┐
│          Filesystem Drivers       │  │          Buffer Heads           │
│        (Ext4, XFS, Btrfs)         │  │  - Maps exact disk sectors to   │
│  - Resolves logical file extents  │  │    page structures (`sb`, inode)│
│  - Determines block addresses     │  │  - Manages raw block transfers  │
└───────────────────┬───────────────┘  └─────────────────┬───────────────┘
                    │                                    │
                    ▼                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                           Generic Block Layer                          │
│                     Request Queues & I/O Schedulers                    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                      Physical Storage Controller                       │
└────────────────────────────────────────────────────────────────────────┘
```

#### Page Cache
The Page Cache caches **file data**. It is indexed by the inode's address space mapping (`struct address_space`) and the byte offset within that file. When an application calls `read()` on a regular file, the kernel checks whether the requested $4\text{ KiB}$ file-offset page resides in the Page Cache. If present, data is transferred from kernel memory to userspace buffers via `copy_to_user()` without touching the block layer.

#### Buffers (Buffer Cache)
Buffers cache **raw block device abstractions**. A buffer represents a single physical disk block mapped by a descriptor called `struct buffer_head`. Buffers handle:
* Filesystem metadata writes (inode table updates, superblock commits, directory allocation maps).
* Raw disk operations that access `/dev/sda` or `/dev/nvme0n1` directly without a mounted filesystem.
* Bridging non-$4\text{ KiB}$ storage sector structures (e.g., $512\text{-byte}$ sectors) to $4\text{ KiB}$ memory pages.

---

### Writeback Mechanics and Dirty Page Flushing

When a process invokes `write(fd, buf, count)`, the kernel does not write the payload straight to physical media. Doing so would serialize program execution down to storage I/O speeds. 

Instead, the kernel writes the bytes into the Page Cache, marks the corresponding memory pages with the `PG_dirty` bit flag, and returns control to the calling process.

```
Userspace write() ──> [ Marks Page PG_dirty ] ──> Process Resumes Instantly
                             │
            ┌────────────────┴────────────────┐
            ▼                                 ▼
   Periodic Flusher Threads        Direct Synchronous Throttling
   (background writeback)          (foreground process stall)
   Triggered by:                   Triggered by:
   - `dirty_background_ratio`      - `dirty_ratio`
   - `dirty_writeback_centisecs`   - `dirty_bytes`
   - `dirty_expire_centisecs`      
            │                                 │
            └────────────────┬────────────────┘
                             │
                             ▼
   Flushes pages across Block Layer to Non-Volatile Storage Medium
```

Synchronizing dirty memory pages to non-volatile storage is managed by kernel flusher threads (`kworker/u:X` writeback workers). This behavior is governed by `sysctl` knobs:

```bash
$ sysctl -a | grep -E 'vm\.dirty_(background_)?(ratio|bytes)'
```

```text
vm.dirty_background_ratio = 10
vm.dirty_background_bytes = 0
vm.dirty_ratio = 20
vm.dirty_bytes = 0
vm.dirty_expire_centisecs = 3000
vm.dirty_writeback_centisecs = 500
```

#### The Dirty Flushing Parameters
* **`vm.dirty_background_ratio`**: The percentage of total system memory containing dirty pages that triggers background flusher threads to start writing data to storage asynchronously. Default is $10\%$. Userspace processes continue executing unblocked.
* **`vm.dirty_ratio`**: The hard ceiling. If dirty pages reach this percentage of system memory, **all write operations in userspace processes are blocked**. Calling processes are forced into direct writeback mode to flush pages to disk synchronously until dirty memory drops below the threshold.
* **`vm.dirty_background_bytes` / `vm.dirty_bytes`**: Absolute byte alternatives to ratios. Setting one automatically clears the other to `0`. On systems with large physical memory footprints (e.g., $512\text{ GiB}$ RAM), a $10\%$ dirty ratio equates to $51.2\text{ GiB}$ of dirty pages—large enough to stall a storage controller for minutes during a flush. Defining `vm.dirty_background_bytes = 268435456` ($256\text{ MiB}$) and `vm.dirty_bytes = 1073741824` ($1\text{ GiB}$) prevents this I/O spike.
* **`vm.dirty_expire_centisecs`**: Age threshold (in hundredths of a second) for a dirty page. A page dirty for longer than this duration ($3\,000\text{ cs} = 30\text{ seconds}$) is flagged for immediate disk synchronization.
* **`vm.dirty_writeback_centisecs`**: The wake-up interval for kernel flusher threads ($500\text{ cs} = 5\text{ seconds}$). Every period, flusher threads wake to write out all expired dirty pages.

---

### Page Replacement: Active vs. Inactive LRU Lists

Linux manages page eviction using a **Least Recently Used (LRU)** list architecture. The kernel divides physical pages across two dual linked lists:
1. **Anonymous Lists**: `Active(anon)` and `Inactive(anon)`
2. **File-backed Lists**: `Active(file)` and `Inactive(file)`

```
                       The Dual-List Page Lifecycle
                         
  Allocated Page (New)
          │
          ▼
   [ Inactive List ] <────────────────────────────┐
   (PG_referenced = 0)                            │
          │                                       │
          │ Accessed a 2nd time?                  │ Evicted under scan
          ├── YES ──> Promoted ──> [ Active List ]│ pressure
          │                        (PG_referenced = 1)
          │                                │
          └── NO (Cold page)               │ Not accessed recently?
                   │                       └─────── Demoted ──────────┘
                   ▼
       Evicted (Page Reclaim)
       - File Page: Purged / Written to disk
       - Anon Page: Swapped to disk
```

#### The Scan-and-Evict Algorithm
* When a page is first allocated, it is inserted into the head of an **`Inactive`** list with its reference bit set to zero.
* If the page is referenced while on the `Inactive` list, its reference bit is set (`PG_referenced`). If accessed again, the kernel promotes the page to the **`Active`** list.
* Under memory pressure, the kernel memory scanner (`kswapd`) sweeps the `Active` list. If pages on the `Active` list have not been referenced across scanning cycles, they are demoted back to the `Inactive` list.
* Pages that reach the tail of the `Inactive` list without active references are evicted:
  * **File pages**: If clean, the page frame is reclaimed immediately. If dirty, it is scheduled for writeback before being freed.
  * **Anonymous pages**: The page cannot be dropped because there is no backing file on disk. It must be written to **swap space**. If swap is absent or exhausted, anonymous pages cannot be reclaimed, forcing the kernel toward out-of-memory states.

---

## 2. Swap Subsystem and Kernel Tuning

Swap space is often misunderstood as emergency virtual memory used only when physical RAM is exhausted. In a modern Linux kernel, swap acts as a **tiering engine** that demotes cold anonymous memory pages to non-volatile storage, preserving high-value physical RAM for productive caching and active execution.

### The Role of Swap

A Linux system without swap space runs with degraded efficiency even when physical RAM appears abundant:
1. **Preserving Page Cache Capacity**: Real-world processes maintain anonymous allocations (initialization code, idle daemon memory, memory-mapped assets) that execute once and are never touched again. If swap is disabled, the kernel is forced to pin these cold anonymous pages permanently in RAM. 
2. **Mitigating Asymmetric Reclamation**: When memory pressure occurs on a swapless system, the kernel can reclaim only **file-backed pages**. This discards high-frequency page cache and hot filesystem libraries while leaving cold anonymous memory untouched, which can lead to disk thrashing.
3. **Absorbing Transient Allocation Bursts**: Swap provides a buffer when bursty workloads demand short-term memory spikes, preventing the kernel from invoking the OOM Killer during momentary allocation peaks.

---

### Swap Partitions vs. Swap Files

Historically, dedicated swap partitions were required to avoid filesystem overhead. In modern Linux (kernels $\ge 5.0$), **swap files achieve the exact same I/O performance as dedicated swap partitions**. 

The kernel accesses swap files by querying the underlying filesystem driver for the direct physical block allocation mapping (the file's contiguous extent map via `bmap` / `FIEMAP`). The kernel bypasses the filesystem layer entirely and writes directly to raw disk sectors using block-level drivers.

#### Provisioning an Optimized Swap File on Modern Filesystems

```bash
# 1. Allocate a sparse-free, contiguous 8 GiB block container:
# NEVER use fallocate on Btrfs or older XFS; dd guarantees zeroed physical block allocation:
sudo dd if=/dev/zero of=/swapfile bs=1M count=8192 status=progress

# 2. Enforce strict permissions (security requirement: read/write strictly by UID 0):
sudo chmod 0600 /swapfile

# 3. Format the block sequence with a Linux swap header:
sudo mkswap /swapfile
```

```text
Setting up swapspace version 1, size = 8 GiB (8589930496 bytes)
no label, UUID=a4e8d356-9d32-472e-8367-a2f07149a4bf
```

```bash
# 4. Activate the swap file within the running kernel:
sudo swapon /swapfile

# 5. Verify the kernel recognizes the block allocation:
swapon --show
```

```text
NAME      TYPE SIZE USED PRIO
/swapfile file   8G   0B   -2
```

```bash
# 6. Ensure persistent activation across reboots in /etc/fstab:
# Append using UUID or path:
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

#### Swap on Modern Copy-On-Write (CoW) Filesystems (Btrfs)
Btrfs historically prohibited swap files because its Copy-On-Write mechanics relocate block extents on modification, breaking the kernel's static sector map. On Btrfs (kernel $\ge 5.0$), you must create a dedicated uncompressed, non-CoW subvolume:

```bash
# Set up a non-CoW swap directory on Btrfs:
sudo btrfs subvolume create /swap
sudo chattr +C /swap # Disables Copy-on-Write (NOCOW attribute)
sudo dd if=/dev/zero of=/swap/swapfile bs=1M count=4096 status=progress
sudo chmod 0600 /swap/swapfile
sudo mkswap /swap/swapfile
sudo swapon /swap/swapfile
```

---

### Tuning `vm.swappiness`

The `vm.swappiness` sysctl knob is often mischaracterized as the memory capacity threshold at which swapping begins (e.g., claiming `swappiness=60` means "swap when RAM reaches $40\%$ free"). **This is incorrect.**

`vm.swappiness` governs the **proportional balance** between anonymous memory scanning and file-backed Page Cache scanning during a memory reclamation cycle.

In the kernel source code (`mm/vmscan.c`, inside the function `get_scan_count()`), the kernel calculates the scanning pressure for both memory types:

```c
/* Pseudocode representation of get_scan_count() in mm/vmscan.c */
unsigned long anon_prio = swappiness;
unsigned long file_prio = 200 - anon_prio;

/* The scan targets are proportionally scaled */
scan_anon = (recent_scanned[0] + 1) * anon_prio;
scan_file = (recent_scanned[1] + 1) * file_prio;
```

$$\text{Scanning Ratio} = \frac{\text{Anon Pages Scanned}}{\text{File Pages Scanned}} = \frac{\text{swappiness}}{200 - \text{swappiness}}$$

```
                       Swappiness Balance Spectrum
 [swappiness = 0]            [swappiness = 60]             [swappiness = 200]
 File Scanning Priority      Production Default Balanced   Aggressive Swap Priority
        ┌────────────────────────────┬────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
 Anon Scanning: 0             Anon Scanning: 60            Anon Scanning: 200
 File Scanning: 200           File Scanning: 140           File Scanning: 0
 Avoids swap completely       Proportionally favors        Scans anonymous pages
 unless memory watermarks     Page Cache eviction over     exclusively; preserves
 fail completely.             swapping anonymous pages.    Page Cache heavily.
```

#### Operational Effects of Swappiness Settings

| `vm.swappiness` Value | Anonymous Scan Weight | File Scan Weight | Operational Behavior |
| :--- | :--- | :--- | :--- |
| **`0`** | $0$ | $200$ | Completely disables swapping of anonymous pages until the free memory falls below the zone high and low watermarks. **Warning**: Can induce early OOM conditions if file pages cannot be freed. |
| **`1`** | $1$ | $199$ | Minimum possible swapping without disabling it entirely. The kernel keeps anonymous pages in physical RAM whenever possible, swapping only under severe physical memory pressure. |
| **`10`** | $10$ | $190$ | Recommended for low-latency databases (e.g., PostgreSQL, Redis, MySQL). Keeps query execution workspaces in physical RAM while permitting swapping to prevent hard OOM kills. |
| **`60`** | $60$ | $140$ | Linux default. Provides a balanced configuration that evicts clean file caches while gradually moving cold anonymous process memory to swap. |
| **`100`** | $100$ | $100$ | Equal weighting. Anonymous pages and file-backed pages are scanned and evaluated for eviction with equal priority. |
| **`150–200`** | $150\text{--}200$ | $50\text{--}0$ | Heavily favors swapping anonymous memory to maximize the Page Cache. Useful when using fast NVMe-backed swap or in-memory compression (ZRAM/ZSWAP). Supported on kernels $\ge 5.8$. |

To adjust `swappiness` at runtime and persist it across system boots:

```bash
# Check current system value:
cat /proc/sys/vm/swappiness

# Modify live kernel state immediately:
sudo sysctl -w vm.swappiness=10

# Persist configuration across system reboots:
echo "vm.swappiness = 10" | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
```

---

### Related Virtual Memory Kernel Parameters

Along with `swappiness`, several other `vm.*` sysctl parameters govern system memory reclamation, allocation limits, and cache retention:

#### 1. `vm.vfs_cache_pressure` (Default: `100`)
Controls the kernel's tendency to reclaim Directory Entries (dentries) and Inode objects from the Slab cache relative to standard Page Cache and anonymous memory:
* **`< 100`**: The kernel prefers to retain dentries and inodes in memory. Useful for file servers handling large directory trees, but risks growing the slab cache under memory pressure.
* **`100`**: Balanced reclamation of VFS metadata structures and page cache frames.
* **`> 100`**: The kernel aggressively evicts cached dentries and inodes. Reduces kernel slab memory, but increases filesystem metadata lookup latency on storage devices.

#### 2. `vm.watermark_scale_factor` (Default: `10`, representing $0.1\%$ of the zone)
Controls the distance between the zone allocation watermarks (`WMARK_MIN`, `WMARK_LOW`, `WMARK_HIGH`):

$$\text{Watermark Distance} = \frac{\text{Zone Size} \times \text{watermark\_scale\_factor}}{10000}$$

Under bursty allocation patterns, increasing this value (e.g., to `50` or `100`) forces background reclaim daemon `kswapd` to wake earlier, freeing pages before applications stall on direct reclamation.

#### 3. `vm.min_free_kbytes`
Calculates the minimum physical memory floor that must remain unallocated to service atomic kernel allocations (e.g., network packet interrupts from drivers within `GFP_ATOMIC` context):
* If set too low: High-speed network interfaces (40GbE/100GbE) may fail to allocate socket ring buffers, dropping network packets with out-of-memory errors even with ample RAM.
* If set too high: Artificially reduces usable system memory, forcing early reclaim cycles.

---

### Memory Overcommit Strategies

Linux allows processes to allocate more virtual memory than the machine physically possesses. This design accommodates modern application models (such as `fork()`, where child processes duplicate the parent's virtual address mappings via Copy-on-Write without writing to all pages).

Overcommit behavior is governed by `vm.overcommit_memory`:

```bash
$ cat /proc/sys/vm/overcommit_memory
0
```

```
                              Memory Overcommit Modes
               
  [Mode 0: Heuristic]            [Mode 1: Always]             [Mode 2: Strict]
  (vm.overcommit_memory=0)       (vm.overcommit_memory=1)     (vm.overcommit_memory=2)
          │                              │                            │
          ▼                              ▼                            ▼
  Heuristic check. Rejects       Unconditional overcommit.    Deterministic overcommit
  unrealistic allocations        Never denies virtual         limit enforced via:
  (e.g., malloc(128TiB)),        allocations. High risk       CommitLimit =
  permits reasonable ones.       of triggering OOM Killer.    (RAM * ratio) + Swap
```

#### The Overcommit Policies
1. **`vm.overcommit_memory = 0` (Heuristic Overcommit - Default)**:
   The kernel evaluates requested allocations against a heuristic algorithm. It allows reasonable overcommits, but rejects obvious memory requests that exceed system capacity (e.g., an application requesting multiple terabytes of virtual memory on a $16\text{ GiB}$ host).
2. **`vm.overcommit_memory = 1` (Always Overcommit)**:
   The kernel grants all virtual memory requests, disabling allocation verification checks entirely. This setting is often required for high-throughput scientific computing, virtualization, or engines like **Redis**, where processes allocate large virtual footprints but touch only a fraction of their mapped pages.
3. **`vm.overcommit_memory = 2` (Strict Non-Overcommit)**:
   Disables overcommit. The total virtual address allocation for the system cannot exceed a mathematically enforced limit called the **`CommitLimit`**. Any allocation that would cause the system's `Committed_AS` to exceed this limit fails at the system call level (e.g., `malloc()` returns `NULL` with `ENOMEM`).

When `vm.overcommit_memory = 2`, the ceiling is governed by:

$$\text{CommitLimit} = \left( \text{Physical RAM} \times \frac{\text{vm.overcommit\_ratio}}{100} \right) + \text{SwapTotal}$$

Or, using an absolute value via `vm.overcommit_kbytes`:

$$\text{CommitLimit} = \text{vm.overcommit\_kbytes} + \text{SwapTotal}$$

```bash
# Inspect the active CommitLimit and Committed Allocation Space:
grep -E 'Commit(Limit|_AS)' /proc/meminfo
```

```text
CommitLimit:    24713196 kB
Committed_AS:   18420912 kB
```

* **`CommitLimit`**: The maximum virtual memory address space the kernel will allocate under Mode 2.
* **`Committed_AS`**: The total virtual memory allocated across all processes on the system. If `Committed_AS > CommitLimit` under Mode 2, subsequent allocation requests are rejected.

---

### In-Memory Compression: ZRAM and ZSWAP

Modern Linux kernels provide memory compression technologies that create a high-speed compression cache layer in front of, or in place of, standard storage devices.

```
                                ZSWAP vs. ZRAM Path
                                
                  [ Anonymous Memory Page Marked for Eviction ]
                                        │
                    ┌───────────────────┴───────────────────┐
                    ▼                                       ▼
           [ ZRAM Subsystem ]                      [ ZSWAP Subsystem ]
    - Standalone Block Device in RAM        - Write-through cache in front of swap
    - No physical disk required             - Requires a real backing swap device
    - Replaces traditional swap             - Evicts compressed pages to disk
            │                                       │
            ▼                                       ▼
  [ Compressed RAM Pool ]                 [ Compressed RAM Pool ]
  (lz4 / zstd compression)                (lz4 / zstd compression)
                                                    │
                                                    ▼ (Pool fills up)
                                          [ Backing Swap Device ]
                                          (NVMe / SSD / HDD storage)
```

#### ZRAM: Compressed Block Device in RAM
ZRAM creates a virtual block device backed directly by physical memory. Data written to this device is dynamically compressed using algorithms like LZ4 or Zstandard (zstd). 
* ZRAM replaces traditional swap partitions or files.
* Typically achieves a $2:1$ to $3:1$ compression ratio, effectively doubling or tripling available memory capacity for compressible data.
* Commonly deployed on resource-constrained embedded platforms, IoT systems, and cloud virtualization hosts.

#### ZSWAP: Compressed Page-Cache Tier
ZSWAP is a compressed write-back cache that intercepts anonymous pages before they are written to a traditional swap device.
* ZSWAP sits **in front of** a real backing swap device (partition or file).
* When a page is flagged for swap, ZSWAP attempts to compress it and store it in an in-memory pool.
* If the ZSWAP pool reaches its capacity limit (configured by `zswap.max_pool_percent`), it selects the coldest compressed page, decompresses it, and writes it out to the physical swap device.
* This model minimizes physical disk I/O, extending the lifespan of flash media (SSDs/NVMe) and reducing latency spikes.

---

## 3. Diagnosing Process Memory Footprints: VSZ, RSS, PSS, and USS

Standard monitoring tools often report process memory metrics that lead to inaccurate conclusions. Adding the memory usage of all processes on a system often produces a total that significantly exceeds the machine's physical RAM:

```bash
# Naive aggregation of RSS often produces misleading metrics:
ps -eo rss | awk '{sum+=$1} END {printf "Summed RSS: %.2f GiB\n", sum/1024/1024}'
# Output on a 32 GiB host can report: Summed RSS: 58.42 GiB!
```

This discrepancy occurs because processes share memory pages through shared libraries, shared memory segments, and Copy-On-Write forks. Understanding process footprints requires distinguishing between **Virtual Size (VSZ)**, **Resident Set Size (RSS)**, **Proportional Set Size (PSS)**, and **Unique Set Size (USS)**.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Virtual Size (VSZ)                                │
│  Entire address space: mapped libraries, uncommitted allocations,      │
│  reserved threads, heap, stack, and mapped memory files.               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    Resident Set Size (RSS)                       │  │
│  │  Pages held in physical DRAM. Includes shared libraries!         │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │
│  │  │                 Proportional Set Size (PSS)                │  │  │
│  │  │  Private allocations + (Shared Library Pages / Num Procs)  │  │  │
│  │  │  ┌──────────────────────────────────────────────────────┐  │  │  │
│  │  │  │              Unique Set Size (USS)                   │  │  │  │
│  │  │  │  Private memory owned EXCLUSIVELY by this process.   │  │  │  │
│  │  │  │  Returned directly to the kernel if killed.          │  │  │  │
│  │  │  └──────────────────────────────────────────────────────┘  │  │  │
│  │  └────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────┘
```

---

### Process Memory Layout

Every userspace ELF application executed by `execve(2)` is mapped into an isolated virtual memory space. On 64-bit Linux systems, this layout is organized into distinct segments:

```
0x00007FFFFFFFFFFF ┌──────────────────────────────────────────┐ Top of Userspace Stack
                   │                  Stack                   │ (Grows downward)
                   │  - Local variables                       │
                   │  - Stack frames & function parameters    │
                   ├──────────────────────────────────────────┤
                   │                    │                     │
                   │                    ▼                     │
                   │                                          │
                   │                    ▲                     │
                   │                    │                     │
                   ├──────────────────────────────────────────┤
                   │         Memory Mapping Segment           │ (Grows upward/downward)
                   │  - Dynamic Libraries (libc.so)           │
                   │  - Shared Memory (shmget, mmap)          │
                   │  - Anonymous mmap allocations            │
                   ├──────────────────────────────────────────┤
                   │                    ▲                     │
                   │                    │                     │
                   │                  Heap                    │ (Grows upward via brk/sbrk)
                   │  - Dynamic allocations (malloc, free)    │
                   ├──────────────────────────────────────────┤
                   │                BSS Segment               │
                   │  - Uninitialized global & static vars    │
                   │  - Zero-initialized by kernel            │
                   ├──────────────────────────────────────────┤
                   │                Data Segment              │
                   │  - Initialized global & static variables │
                   ├──────────────────────────────────────────┤
                   │                Text Segment              │
                   │  - Compiled binary machine code (RX)     │
                   │  - Read-only, shared across instances    │
0x0000000000400000 └──────────────────────────────────────────┘ Base Executable Address
```

---

### Metric Definitions: VSZ, RSS, PSS, USS

#### 1. VSZ (Virtual Memory Size)
The total size of the virtual address space allocated by the process.
* Includes all allocated memory: executable text segments, allocated heap that has never been written to, threads reserved with default stack sizes (typically $8\text{ MiB}$ per thread), shared dynamic libraries (`libc.so`), and file mappings created via `mmap()`.
* **Diagnostic Flaw**: An application may show a VSZ of $20\text{ GiB}$ while consuming only $50\text{ MiB}$ of physical RAM. Allocating an array via `malloc(1024*1024*1024)` increments VSZ by $1\text{ GiB}$ instantly, but consumes zero bytes of physical memory until the program writes to those addresses.

#### 2. RSS (Resident Set Size)
The amount of physical memory (RAM) mapped into the process's page table.
* Includes all pages currently in RAM: the process's private code, allocated and initialized heap pages, the active runtime stack, and **shared dynamic libraries**.
* **Diagnostic Flaw**: RSS overcounts memory when processes share libraries or shared memory. If $100$ instances of an application link against a $10\text{ MiB}$ shared library (`libcuda.so`), each process's RSS includes that $10\text{ MiB}$. Summing their RSS overcounts physical RAM by $990\text{ MiB}$.

#### 3. PSS (Proportional Set Size)
The metric for capacity planning and process accounting. PSS tracks a process's private memory allocations and adds a proportional fraction of its shared memory pages based on how many processes share them:

$$\text{PSS} = \text{USS} + \sum_{i=1}^{M} \frac{\text{SharedPage}_i}{N_i}$$

Where:
* $M$ is the set of all shared memory pages mapped by the process.
* $N_i$ is the total number of processes currently mapping the shared page $i$.

If a $10\text{ MiB}$ shared library is mapped by $10$ processes, each process accounts for:

$$\frac{10\text{ MiB}}{10} = 1\text{ MiB PSS}$$

Unlike RSS, **summing the PSS across all running processes accurately reflects the total physical RAM consumed by userspace**.

#### 4. USS (Unique Set Size)
The memory owned exclusively by a single process.
* USS contains the private anonymous pages, private heap, and private stack allocated by the process.
* **Operational Significance**: USS represents the **true return on termination**. If you terminate a process, its USS is the exact amount of physical RAM that will be returned to the kernel's free page pool immediately.

---

### Deep Dive: `/proc/[pid]/smaps` and `smaps_rollup`

Process memory structures are exposed through the `/proc/[pid]/` virtual filesystem:
* `/proc/[pid]/maps`: Displays virtual address ranges, permissions, offsets, and mapped files.
* `/proc/[pid]/smaps`: Displays detailed memory metrics (RSS, PSS, Shared/Private, Dirty/Clean) for **every individual virtual address allocation**.
* `/proc/[pid]/smaps_rollup`: Available in modern kernels; aggregates all allocations from `smaps` into a single summary, reducing the parsing overhead of reading thousands of memory maps.

```bash
# Inspect smaps_rollup for a running system daemon (e.g., systemd-journald):
sudo cat /proc/$(pgrep -o systemd-journal)/smaps_rollup
```

```text
00400000-7ffffff00000 ---p 00000000 00:00 0                      [rollup]
Rss:               24920 kB
Pss:                8412 kB
Pss_Anon:           4108 kB
Pss_File:           4204 kB
Pss_Shmem:           100 kB
Shared_Clean:      18104 kB
Shared_Dirty:          0 kB
Private_Clean:      2708 kB
Private_Dirty:      4108 kB
Referenced:        24920 kB
Anonymous:          4108 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:         0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:               1024 kB
SwapPss:             128 kB
Locked:                0 kB
```

#### Key Metrics in `smaps_rollup`
* **`Private_Clean`**: Memory pages backed by files that have been read into memory, not modified, and mapped exclusively by this process.
* **`Private_Dirty`**: Memory pages modified by this process that belong exclusively to it. **This is the primary component of USS**.
* **`Shared_Clean`**: Read-only shared library pages mapped concurrently across multiple processes.
* **`Shared_Dirty`**: Shared memory pages (such as POSIX shared memory segments) that have been modified by this or another process.
* **`Swap`**: The total amount of this process's memory that has been moved to the swap subsystem.
* **`SwapPss`**: Proportional swap accounting. Calculates this process's proportional share of swapped memory across shared instances.

---

### Process Memory Inspection Utilities

```bash
# 1. Standard process memory listing with ps:
# VSZ and RSS are returned in KiB
ps -eo pid,user,vsz,rss,comm --sort=-rss | head -n 10

# 2. Detailed segment breakdown using pmap:
# The -x flag displays detailed resident and dirty segment accounting
sudo pmap -x $(pgrep -o nginx) | head -n 20
```

```text
34120:   nginx: worker process
Address           Kbytes     RSS   Dirty Mode  Mapping
000055d491c28000    1148     812       0 r-x-- nginx
000055d491f46000      20      20      20 r---- nginx
000055d491f4b000     116     116     108 rw--- nginx
000055d49242a000    2144    1840    1840 rw--- [ heap ]
00007fc28a400000    1980     420       0 r-x-- libc.so.6
00007fc28a5ef000      16      16      16 r---- libc.so.6
00007fc28a5f3000       8       8       8 rw--- libc.so.6
00007ffd431a0000     132      24      24 rw--- [ stack ]
---------------- ------- ------- ------- 
total kB           41240   14920    2016
```

#### Comprehensive PSS/USS Auditing with `smem`
The `smem` tool parses `/proc/[pid]/smaps` to calculate accurate PSS and USS figures across the system:

```bash
# Output system memory consumption sorted by PSS:
sudo smem -t -p -k --sort=pss | tail -n 15
```

```text
PID  User     Command                         Swap      USS      PSS      RSS 
2104 mysql    /usr/sbin/mysqld                0.0%   412.4M   424.1M   450.2M 
4120 root     /usr/bin/dockerd                0.0%    48.2M    54.6M    72.1M 
8912 www-data php-fpm: pool www               0.0%    18.4M    24.2M    52.1M 
8913 www-data php-fpm: pool www               0.0%    19.1M    25.1M    53.4M 
8914 www-data php-fpm: pool www               0.0%    18.9M    24.8M    52.8M 
--------------------------------------------------------------------------
TOTAL                                         0.0%     1.2G     1.6G     3.8G 
```

The output highlights the difference between RSS and PSS: the `php-fpm` workers display an aggregate RSS exceeding $150\text{ MiB}$, but their true combined physical impact (PSS) is roughly half that figure because they share core opcode caches and execution runtimes.

---

## 4. The Out-Of-Memory (OOM) Killer Subsystem

The **Out-Of-Memory (OOM) Killer** is the kernel's mechanism of last resort. It terminates one or more processes to free physical memory and prevent a total system lockup when:
1. Physical memory and swap space are completely exhausted.
2. The kernel's background (`kswapd`) and direct page reclamation mechanisms cannot free enough pages to satisfy a memory request.
3. System memory allocations fall below the minimum safe watermarks (`WMARK_MIN`).

```
                            The OOM Killer Invocation Path
                                          
               [ Userspace Process Calls malloc() / Page Fault ]
                                         │
                                         ▼
                     [ Kernel Page Allocator: alloc_pages() ]
                                         │
                                         ▼
               Memory Available? ──YES──> [ Allocate Page Frame ]
                       │
                       NO
                       ▼
        [ Direct Reclamation Engine: try_to_free_pages() ]
        - Flushes dirty page cache to disk
        - Evicts clean file-backed pages
        - Writes anonymous pages to swap
                       │
                       ▼
            Memory Freed? ──YES──> [ Allocate Page Frame ]
                       │
                       NO
                       ▼
            [ Memory Compaction: compact_zone() ]
            Migrates pages to resolve fragmentation
                       │
                       ▼
       Sufficient Pages Freed? ──YES──> [ Allocate Page Frame ]
                       │
                       NO
                       ▼
          [ CRITICAL SYSTEM STATE: OUT OF MEMORY ]
          - Page allocator exhausted
          - System cannot service basic kernel tasks
                       │
                       ▼
      [ Kernel Invokes out_of_memory() in mm/oom_kill.c ]
      - Scans task list via select_bad_process()
      - Calculates badness score (oom_score) for each task
      - Selects process with highest combined score
      - Delivers SIGKILL (or terminates cgroup if configured)
```

---

### The Scoring Algorithm: How the Kernel Picks the Victim

The kernel selects processes for termination using the `badness()` function defined in `mm/oom_kill.c`. The selection process balances two priorities:
1. **Minimize collateral damage**: Avoid killing critical system daemons or processes that will not free significant memory.
2. **Maximize recovered memory**: Terminate processes using the largest amounts of physical memory to resolve the memory deficit in as few terminations as possible.

The badness formula computes a baseline integer score between $0$ and $1000$:

$$\text{Points} = \frac{\text{Task PSS + Swap Usage}}{\text{Total Usable System RAM}} \times 1000$$

The base calculation uses the process's physical memory footprint (RSS/PSS) along with its active swap consumption. Processes that consume large fractions of physical RAM and swap score higher base values.

The kernel then applies modifiers to this base score:

#### 1. Root Privileges
Processes running under superuser privileges (`UID == 0`) or possessing explicit administrative capabilities (`CAP_SYS_ADMIN`, `CAP_SYS_RAWIO`) receive a minor score deduction (typically $-3\%$ or $-30$ points). This deduction provides a small buffer for core administrative services, though root-owned workloads consuming massive memory footprints can still be selected for termination.

#### 2. The `oom_score_adj` Adjustment
Userspace can modify a process's OOM priority using `/proc/[pid]/oom_score_adj`. This interface accepts values from **`-1000` to `+1000`**:

$$\text{Final oom\_score} = \text{Points}_{\text{base}} + \text{oom\_score\_adj}$$

* The resulting score is clamped between $0$ and $1000$.
* Setting `oom_score_adj = 1000` makes the process the top candidate for termination during an OOM event.
* Setting `oom_score_adj = -1000` assigns the special value `OOM_SCORE_ADJ_MIN`, which **exempts the process from the OOM Killer entirely**. The kernel will not target a process with a `-1000` score unless no other target exists on the system.

```bash
# Query the live, calculated score of a specific process:
cat /proc/$(pgrep -o postgres)/oom_score

# Query the manual score adjustment value:
cat /proc/$(pgrep -o postgres)/oom_score_adj

# Protect an application process from OOM termination:
echo -1000 | sudo tee /proc/$(pgrep -o postgres)/oom_score_adj
```

> **Legacy Interface Note**: Legacy Linux kernels used `/proc/[pid]/oom_adj`, which scaled from `-17` to `+15` (with `-17` indicating exemption). Modern kernels map `oom_adj` to `oom_score_adj` automatically. Production systems should use `oom_score_adj`.

---

### Cgroups v2: Scoped OOM and `memory.oom.group`

Under modern Linux deployments using Control Groups v2 (cgroups v2), the kernel can manage OOM events at the cgroup level rather than across the entire host.

```
                  Cgroup v2 Memory Enclosure
 /sys/fs/cgroup/system.slice/docker-container-xyz.scope/
  ├── memory.max = 4G
  ├── memory.current = 4G (Limit reached!)
  └── memory.oom.group = 1
```

If a cgroup's memory usage reaches the hard boundary defined in `memory.max` and memory cannot be reclaimed:
1. The kernel invokes an **isolated cgroup OOM event**, targeting processes within that specific cgroup hierarchy. System processes outside the cgroup are unaffected.
2. By default, the OOM Killer terminates only the single highest-scoring process in that cgroup. However, in applications with coupled parent and child processes (e.g., a web server master process and its workers), terminating a single child can leave the application in an inconsistent state.
3. Enabling **`memory.oom.group = 1`** changes this behavior: if any process in the cgroup triggers an OOM kill, the kernel **terminates every process within that cgroup simultaneously**. This ensures clean application restarts through its supervising process manager (such as `systemd` or a container runtime).

---

### Systemd Integration: OOM Management

`systemd` provides configuration directives to control OOM Killer behavior across services:

```ini
# /etc/systemd/system/production-database.service
[Unit]
Description=High-Performance Production Database
After=network.target

[Service]
ExecStart=/usr/bin/db-engine --config /etc/db.conf
Restart=always

# 1. Protect the daemon from OOM termination:
OOMScoreAdjust=-900

# 2. Enforce strict cgroups v2 resource boundaries:
MemoryHigh=28G
MemoryMax=30G

# 3. Terminate the entire unit if memory is exhausted:
OOMPolicy=stop
```

#### The `OOMPolicy` Directive
* **`OOMPolicy=continue`**: If a process inside the service unit is killed by the OOM killer, systemd logs the event and leaves any remaining processes running.
* **`OOMPolicy=stop`**: If any process in the service unit is killed, systemd cleanly stops the entire service, releasing its resources.
* **`OOMPolicy=kill`**: If an OOM event occurs, systemd terminates all remaining processes in the unit's cgroup immediately using `SIGKILL`.

#### Userspace OOM: `systemd-oomd`
Modern distributions (Fedora, Ubuntu, Debian) include `systemd-oomd`. Unlike the kernel's in-tree OOM Killer—which acts only after allocations fail—`systemd-oomd` operates in userspace. It monitors **Pressure Stall Information (PSI)** via `/proc/pressure/memory`:

```bash
$ cat /proc/pressure/memory
some avg10=0.00 avg60=0.00 avg300=0.00 total=0
full avg10=0.00 avg60=0.00 avg300=0.00 total=0
```

* **`some`**: The percentage of time that at least one task was stalled waiting for memory resources (e.g., paging, swap-in, direct reclaim).
* **`full`**: The percentage of time that **all non-idle tasks** were simultaneously stalled waiting for memory. This metric indicates severe memory starvation.

When memory pressure metrics exceed configured thresholds (defined in `ManagedOOMMemoryPressure=` or `ManagedOOMSwap=`), `systemd-oomd` terminates non-essential cgroups **before** the system locks up or drops into kernel-level direct reclamation.

---

### Inspecting and Analyzing OOM Logs

When the kernel-level OOM Killer is invoked, it writes an event dump to the kernel ring buffer. Reviewing this output provides insight into the system state at the moment of failure:

```bash
# 1. Query the kernel log for OOM kill events:
sudo dmesg -T | grep -i -E '(out of memory|killed process)'

# 2. Extract the complete OOM incident trace using journalctl:
sudo journalctl -k --grep="Out of memory" -B 0 -C 50
```

#### Annotated Breakdown of an OOM Kernel Incident

```text
[Sun Sep 27 12:15:02 2026] mysqld invoked oom-killer: gfp_mask=0x1100cca(GFP_HIGHUSER_MOVABLE), order=0, oom_score_adj=0
[Sun Sep 27 12:15:02 2026] CPU: 3 PID: 14201 Comm: mysqld Not tainted 6.8.0-31-generic #31-Ubuntu
[Sun Sep 27 12:15:02 2026] Hardware name: Dell Inc. PowerEdge R640/0W23HK, BIOS 2.12.1 07/12/2021
[Sun Sep 27 12:15:02 2026] Call Trace:
[Sun Sep 27 12:15:02 2026]  <TASK>
[Sun Sep 27 12:15:02 2026]  dump_stack_lvl+0x48/0x70
[Sun Sep 27 12:15:02 2026]  dump_header+0x52/0x290
[Sun Sep 27 12:15:02 2026]  oom_kill_process+0x118/0x240
[Sun Sep 27 12:15:02 2026]  out_of_memory+0x245/0x590
[Sun Sep 27 12:15:02 2026]  __alloc_pages_slowpath.constprop.0+0xa12/0xdf0
[Sun Sep 27 12:15:02 2026]  __alloc_pages+0x32d/0x350
[Sun Sep 27 12:15:02 2026]  handle_mm_fault+0xb20/0x1410
[Sun Sep 27 12:15:02 2026]  do_user_addr_fault+0x312/0x680
[Sun Sep 27 12:15:02 2026]  exc_page_fault+0x7e/0x180
[Sun Sep 27 12:15:02 2026]  asm_exc_page_fault+0x26/0x30
[Sun Sep 27 12:15:02 2026]  </TASK>
```

* **`gfp_mask=0x1100cca`**: The Get Free Page allocation flags passed to the page allocator. `GFP_HIGHUSER_MOVABLE` indicates a standard userspace page allocation request that can be satisfied from `ZONE_NORMAL` or `ZONE_MOVABLE`.
* **`order=0`**: The allocation request size, expressed as a power of two ($2^{\text{order}}$ pages). `order=0` indicates a single $4\text{ KiB}$ page frame ($2^0 = 1$). If the kernel fails to allocate even a single $4\text{ KiB}$ page, physical memory is completely exhausted.
* **`Call Trace`**: Shows the execution path that led to the event. Here, an address fault triggered `do_user_addr_fault`, which called `__alloc_pages_slowpath` after direct reclamation failed, ultimately invoking `out_of_memory()`.

#### The Active Memory Zone Dump

```text
[Sun Sep 27 12:15:02 2026] Mem-Info:
[Sun Sep 27 12:15:02 2026] active_anon:4112040 inactive_anon:1020412 isolated_anon:0
                            active_file:120 inactive_file:80 isolated_file:0
                            unevictable:4502 dirty:0 writeback:0
                            slab_reclaimable:14201 slab_unreclaimable:45012
                            mapped:4120 shmem:120 pagetables:18402 bounce:0
                            free:12402 free_pcp:412 local_pcp:12
[Sun Sep 27 12:15:02 2026] Node 0 DMA free:12104kB boost:0kB min:64kB low:80kB high:96kB reserved_highatomic:0KB active_anon:0kB inactive_anon:0kB
[Sun Sep 27 12:15:02 2026] Node 0 DMA32 free:45120kB boost:0kB min:12840kB low:16050kB high:19260kB reserved_highatomic:0KB active_anon:1240100kB
[Sun Sep 27 12:15:02 2026] Node 0 Normal free:32100kB boost:0kB min:65120kB low:81400kB high:97680kB reserved_highatomic:0KB active_anon:14207060kB
```

* **`active_file:120 inactive_file:80`**: The system has almost no file-backed pages left ($200 \times 4\text{ KiB} = 800\text{ KiB}$). The Page Cache has been completely evicted, leaving no further reclaimable file pages.
* **`Node 0 Normal ... free:32100kB ... min:65120kB`**: The free memory in `ZONE_NORMAL` ($32.1\text{ MiB}$) has fallen well below the minimum watermark threshold (`min:65120kB`).

#### The Process State Table and Target Selection

The kernel prints a state table of all active tasks, detailing their virtual footprints, physical allocations, and OOM settings:

```text
[Sun Sep 27 12:15:02 2026] Tasks state (memory values in pages):
[Sun Sep 27 12:15:02 2026] [  pid  ]   uid  tgid total_vm      rss pgtables_bytes swapents oom_score_adj name
[Sun Sep 27 12:15:02 2026] [    890]     0   890     6104      412    65536        0             0 systemd-journal
[Sun Sep 27 12:15:02 2026] [   1120]     0  1120    24102     1240   147456      120         -1000 sshd
[Sun Sep 27 12:15:02 2026] [   3412]   100  3412   124100    14102   348160     1024             0 systemd-resolved
[Sun Sep 27 12:15:02 2026] [  14201]  1001 14201  2412490  1840120 18452480   412000             0 python3
[Sun Sep 27 12:15:02 2026] [  16840]   106 16840   812400   241200  4120576        0          -900 postgres
[Sun Sep 27 12:15:02 2026] Out of memory: Killed process 14201 (python3) total-vm:9649960kB, anon-rss:7360480kB, file-rss:0kB, shmem-rss:0kB, UID:1001 pgtables:18020kB oom_score_adj:0
```

#### Interpreting the Selection Result
1. The kernel evaluated `sshd` (PID 1120), but skipped it due to `oom_score_adj: -1000`.
2. It evaluated `postgres` (PID 16840), but its high base score was significantly reduced by `oom_score_adj: -900`.
3. It identified `python3` (PID 14201), which had an `rss` of `1840120` pages ($\approx 7.18\text{ GiB}$) and `swapents` of `412000` pages ($\approx 1.6\text{ GiB}$). With an unadjusted `oom_score_adj: 0`, this process produced the highest badness score on the system.
4. The kernel sent `SIGKILL` to PID 14201, reclaiming approximately $8.78\text{ GiB}$ of combined memory space.

---

## 5. Practical Laboratories and Diagnostic Walkthroughs

These hands-on exercises demonstrate how to profile process memory metrics, observe page cache writeback behavior, and safely trace an OOM event in a controlled environment.

---

### Lab 1: Profiling Process Footprints: VSZ vs. RSS vs. PSS

#### Objective
Write a Python test application that allocates memory, writes to only a subset of those pages, maps dynamic shared libraries, and forks child processes. Then analyze how VSZ, RSS, PSS, and USS diverge across each phase.

#### Diagnostic Script (`mem_profile_test.py`)

```python
#!/usr/bin/env python3
import os
import time
import mmap

def main():
    pid = os.getpid()
    print(f"[*] Process initialized. PID: {pid}")
    print("[*] Phase 1: Baseline idle state. Check metrics now.")
    time.sleep(15)

    # Allocate 200 MiB of virtual memory without touching pages:
    print("[*] Phase 2: Allocating 200 MiB anonymous virtual memory via mmap...")
    size = 200 * 1024 * 1024  # 200 MiB
    anon_map = mmap.mmap(-1, size, mmap.MAP_PRIVATE | mmap.MAP_ANONYMOUS, mmap.PROT_READ | mmap.PROT_WRITE)
    print("    -> Virtual memory allocated. Data has not been written to pages.")
    time.sleep(15)

    # Touch only 50 MiB of the allocated pages (forces physical frame allocation):
    print("[*] Phase 3: Writing data to 50 MiB of pages (triggering page faults)...")
    touch_size = 50 * 1024 * 1024  # 50 MiB
    anon_map.write(b"\xAA" * touch_size)
    print("    -> 50 MiB populated. 150 MiB remains uncommitted virtual space.")
    time.sleep(15)

    # Fork a child process to demonstrate Copy-on-Write sharing:
    print("[*] Phase 4: Forking child process to observe CoW memory sharing...")
    child_pid = os.fork()
    if child_pid == 0:
        # Child process execution path
        child_self = os.getpid()
        print(f"    [Child] Spawned with PID: {child_self}. Idling in shared state...")
        time.sleep(20)
        os._exit(0)
    else:
        # Parent process execution path
        time.sleep(20)

    print("[*] Lab run complete. Terminating.")

if __name__ == "__main__":
    main()
```

#### Execution and Monitoring Workflow

In terminal 1, run the Python program:
```bash
python3 mem_profile_test.py
```

In terminal 2, monitor the process metrics across each phase:

```bash
# Monitor the process via ps, smaps_rollup, and smem:
TARGET_PID=$(pgrep -f "mem_profile_test.py" | head -n1)

# Phase 1: Baseline idle state
ps -o pid,vsz,rss,comm -p $TARGET_PID
sudo cat /proc/$TARGET_PID/smaps_rollup | grep -E '^(Rss|Pss|Private_Dirty|Shared_Clean)'

# Phase 2: After 200 MiB mmap allocation
# Observe: VSZ increases by ~200 MiB; RSS and PSS remain unchanged!
ps -o pid,vsz,rss,comm -p $TARGET_PID

# Phase 3: After writing to 50 MiB
# Observe: RSS and PSS both increase by ~50 MiB; Private_Dirty increases by 50 MiB.
ps -o pid,vsz,rss,comm -p $TARGET_PID
sudo cat /proc/$TARGET_PID/smaps_rollup | grep -E '^(Rss|Pss|Private_Dirty)'

# Phase 4: After fork()
# Observe: Combined RSS doubles, while PSS splits shared memory between parent and child!
smem -P mem_profile_test.py -k
```

---

### Lab 2: Investigating Page Cache Writeback and Dirty Page Flushing

#### Objective
Observe the behavior of the Page Cache writeback mechanism. Generate dirty pages, track flushing thresholds, and observe process stalls when hitting `vm.dirty_ratio`.

#### Test Script (`dirty_flusher_lab.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

LAB_DIR="/tmp/writeback_lab"
mkdir -p "${LAB_DIR}"
cd "${LAB_DIR}"

echo "=== STEP 1: CURRENT DIRTY FLUSHING THRESHOLDS ==="
sysctl vm.dirty_background_ratio vm.dirty_ratio vm.dirty_expire_centisecs vm.dirty_writeback_centisecs

echo -e "\n=== STEP 2: MONITORING DIRTY PAGES (BASELINE) ==="
grep -E '^(Dirty|Writeback):' /proc/meminfo

echo -e "\n=== STEP 3: GENERATING 500 MIB DIRTY CACHE VIA DD ==="
# Write 500 MiB of data without syncing:
dd if=/dev/zero of=testfile.bin bs=1M count=500 conv=notrunc status=none &
DD_PID=$!

echo "Sampling /proc/meminfo during write operation:"
for i in {1..5}; do
    grep -E '^(Dirty|Writeback):' /proc/meminfo
    sleep 0.5
done

wait $DD_PID
echo "[*] Direct userspace write completed. Data now resides in Dirty Page Cache."

echo -e "\n=== STEP 4: TRACKING ASYNCHRONOUS FLUSHING ==="
echo "Waiting for kernel writeback flusher threads to synchronize pages to storage..."
while true; do
    DIRTY_KB=$(awk '/^Dirty:/ {print $2}' /proc/meminfo)
    WRITEBACK_KB=$(awk '/^Writeback:/ {print $2}' /proc/meminfo)
    printf "Dirty: %8d kB | Active Writeback: %8d kB\r" "$DIRTY_KB" "$WRITEBACK_KB"
    if [ "$DIRTY_KB" -lt 4096 ]; then
        break
    fi
    sleep 0.5
done
echo -e "\n[SUCCESS] Dirty page cache synchronized to storage."

# Clean up
rm -rf "${LAB_DIR}"
```

---

### Lab 3: Inducing a Controlled OOM Event and Auditing Logs

#### Objective
Safely trigger an Out-Of-Memory condition inside an isolated cgroup v2 boundary. Protect your interactive shell session, observe process termination, and analyze the resulting kernel log dump.

#### Execution Script (`oom_simulation_lab.sh`)

```bash
#!/usr/bin/env bash
set -euo pipefail

# Ensure cgroups v2 hierarchy is accessible:
if [ ! -d "/sys/fs/cgroup" ]; then
    echo "[ERROR] cgroups v2 not detected at /sys/fs/cgroup" >&2
    exit 1
fi

CGROUP_DIR="/sys/fs/cgroup/oom_sandbox"
sudo mkdir -p "${CGROUP_DIR}"

echo "=== STEP 1: PROTECTING CURRENT INTERACTIVE SHELL ==="
# Set current shell oom_score_adj to -1000 so it cannot be killed
echo -1000 | sudo tee "/proc/$$/oom_score_adj" > /dev/null
echo "Current Shell (PID $$) OOM Adjustment: $(cat /proc/$$/oom_score_adj)"

echo -e "\n=== STEP 2: CONFIGURING ISOLATED CGROUP LIMITS ==="
# Constrain the cgroup to 100 MiB of RAM and disable swap for this group:
echo "100M" | sudo tee "${CGROUP_DIR}/memory.max" > /dev/null
echo "0"    | sudo tee "${CGROUP_DIR}/memory.swap.max" > /dev/null

echo "Memory Max Limit: $(cat "${CGROUP_DIR}/memory.max")"
echo "Swap Max Limit:   $(cat "${CGROUP_DIR}/memory.swap.max")"

echo -e "\n=== STEP 3: SPAWNING MEMORY ALLOCATOR IN CGROUP ==="
# Launch a background process inside the target cgroup that exceeds 100 MiB:
echo "[*] Spawning python workload into ${CGROUP_DIR}..."

# Launch child directly into the sandbox cgroup:
sudo bash -c "echo \$$ > ${CGROUP_DIR}/cgroup.procs && exec python3 -c '
import time
print(\"[Child] Allocating memory beyond 100 MiB...\")
data = []
try:
    while True:
        # Allocate 10 MiB chunks and touch pages
        data.append(b\"X\" * (10 * 1024 * 1024))
        time.sleep(0.1)
except Exception as e:
    print(f\"Allocation failed: {e}\")
'" || true

echo -e "\n=== STEP 4: AUDITING CGROUP OOM EVENT METRICS ==="
echo "Cgroup Events Record:"
cat "${CGROUP_DIR}/memory.events"

echo -e "\n=== STEP 5: EXTRACTING KERNEL OOM AUDIT LOG ==="
sudo dmesg -T | grep -A 15 -B 2 "invoked oom-killer" | tail -n 25

echo -e "\n=== STEP 6: CLEANING UP LAB ENVIRONMENT ==="
sudo rmdir "${CGROUP_DIR}"
echo "[*] Sandbox cgroup removed."
```

---

## 6. Comprehensive Operational Reference Matrices

### `/proc/meminfo` Diagnostic Reference Matrix

| Metric Name | Storage Source | Eviction Potential | System Diagnostic Meaning |
| :--- | :--- | :--- | :--- |
| **`MemTotal`** | Hardware RAM | None | Total physical address space mapped by the kernel image. |
| **`MemFree`** | Unallocated DRAM | None | Completely idle, unassigned physical pages. |
| **`MemAvailable`** | Algorithm Estimate | High | Estimated memory available for starting new applications without swapping. |
| **`Buffers`** | Block Layer Metadata | High (if clean) | Raw disk block caches, filesystem superblocks, dentry descriptors. |
| **`Cached`** | VFS Page Cache | High (if clean) | Data pages mapped from files on persistent filesystems. |
| **`SwapCached`** | RAM + Swap Copy | Immediate | Pages holding identical copies in both RAM and swap. |
| **`Active(anon)`** | Process Heap/Stack | Swap Required | Anonymous memory touched recently; protected from early scan. |
| **`Inactive(anon)`**| Process Heap/Stack | Swap Required | Anonymous memory unreferenced recently; top candidate for swap out. |
| **`Active(file)`** | Page Cache | Low | File-backed pages touched recently. Eviction requires demotion. |
| **`Inactive(file)`**| Page Cache | Immediate | File-backed pages unreferenced recently. Primary target for instant eviction. |
| **`Dirty`** | Page Cache | Writeback First | Modified file pages awaiting background or direct disk synchronization. |
| **`Writeback`** | Storage Bus Queue | Transient | Pages currently in flight to storage media. |
| **`AnonPages`** | Userspace Maps | Swap Required | Non-file memory mapped directly into process address tables. |
| **`Mapped`** | Userspace Maps | High (if clean) | Files mapped into processes via `mmap()` (shared objects, `.so`). |
| **`Shmem`** | RAM (tmpfs / IPC) | Swap Required | Shared memory allocations and volatile memory filesystems. |
| **`Slab`** | Kernel Allocations | Partial | Memory allocated to internal kernel caches via SLAB/SLUB allocator. |
| **`SReclaimable`** | Kernel Slab | High | Portions of the slab cache (dentry/inode) that can be safely freed. |
| **`SUnreclaim`** | Kernel Slab | None | Kernel internal objects that cannot be freed while the system is active. |

---

### Process Memory Metrics Reference Matrix

| Metric | Full Descriptor | Accounting Scope | Overcounts Shared Memory? | Primary Operational Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **VSZ** | Virtual Memory Size | All mapped virtual addresses | Yes (Massively) | Detects virtual address exhaustion, thread leaks, and address space growth. |
| **RSS** | Resident Set Size | All physical pages in RAM | Yes (Includes shared libs) | Quick check of process memory; inadequate for summing across processes. |
| **PSS** | Proportional Set Size | Private pages + Proportional shared | **No** (Proportionally divided) | **Standard for capacity planning**, memory billing, and host accounting. |
| **USS** | Unique Set Size | Private pages exclusive to PID | **No** (Strictly exclusive) | **Determines memory returned immediately** if the target process is terminated. |
| **Swap** | Swapped Memory Footprint | Anonymous pages in swap | No | Measures the memory demoted to swap due to inactivity or memory pressure. |

---

### Virtual Memory Kernel Tunables (`sysctl vm.*`)

| Parameter Name | Default Value | Tunable Range | Operational Effect | Recommended Tuning |
| :--- | :--- | :--- | :--- | :--- |
| **`vm.swappiness`** | `60` | `0` to `200` | Governs the ratio between anonymous memory scanning and file cache scanning. | `10` for databases; `60` for standard servers; `100–180` for ZRAM/ZSWAP setups. |
| **`vm.dirty_background_ratio`** | `10` | `0` to `100` | Percentage of RAM with dirty pages that triggers asynchronous background flushing. | Set to `3` to `5` on high-memory systems to prevent large I/O bursts. |
| **`vm.dirty_ratio`** | `20` | `0` to `100` | Percentage of RAM with dirty pages that **blocks userspace writes** to force synchronous flushes. | Set to `10` on production servers to prevent long system stalls. |
| **`vm.dirty_background_bytes`** | `0` | Any byte limit | Byte equivalent of `dirty_background_ratio`. (Setting this overrides the ratio). | Set to `268435456` ($256\text{ MiB}$) on servers with $> 64\text{ GiB}$ RAM. |
| **`vm.dirty_bytes`** | `0` | Any byte limit | Byte equivalent of `dirty_ratio`. (Setting this overrides the ratio). | Set to `1073741824` ($1\text{ GiB}$) on servers with $> 64\text{ GiB}$ RAM. |
| **`vm.vfs_cache_pressure`**| `100` | `0` to `1000` | Governs the kernel's tendency to reclaim dentry and inode caches relative to page cache. | `50` for file servers with large directory trees; `100` for standard environments. |
| **`vm.overcommit_memory`** | `0` | `0, 1, 2` | Defines the virtual memory overcommit allocation policy. | `0` for general workloads; `1` for Redis/scientific computing; `2` for hard memory safety. |
| **`vm.overcommit_ratio`** | `50` | `0` to `100` | Percentage of physical RAM factored into the `CommitLimit` when `overcommit_memory = 2`. | `80` on systems with minimal swap; adjust based on allocation headroom. |
| **`vm.min_free_kbytes`** | Scales with RAM | $1\text{ MiB}$ to $5\%\text{ RAM}$ | Reserves a free memory floor for atomic kernel and network interrupt allocations. | Increase on hosts with 40GbE/100GbE NICs to prevent packet drops under load. |

---

### Cgroups v2 Memory Controller Parameters

| Cgroup Controller Interface | Accepted Format | Operational Behavior Under Limit Breach |
| :--- | :--- | :--- |
| **`memory.min`** | Absolute Bytes | Hard memory protection. Pages below this threshold are **never reclaimed** by outside memory pressure. |
| **`memory.low`** | Absolute Bytes | Soft memory protection. Reclaimed only when memory cannot be reclaimed from unprotected cgroups. |
| **`memory.high`** | Absolute Bytes | Soft limit / Throttle threshold. If breached, the cgroup's processes are **throttled and forced into direct reclamation**, but are not terminated. |
| **`memory.max`** | Absolute Bytes | Hard ceiling. If breached and direct reclamation fails, **triggers an OOM kill event** scoped to this cgroup. |
| **`memory.swap.max`** | Absolute Bytes | Hard swap ceiling. Prevents processes in the cgroup from consuming more than this amount of swap space. |
| **`memory.oom.group`** | `0` or `1` | If set to `1`, an OOM event triggers a `SIGKILL` to **every process inside this cgroup**, terminating the unit cleanly. |
| **`memory.events`** | Read-Only Counters | Outputs runtime counters for cgroup events: `low`, `high`, `max`, `oom`, and `oom_kill`. |
| **`memory.pressure`** | Read-Only (PSI) | Exposes Memory Pressure Stall Information (`some`, `full`) for tasks inside this cgroup. |