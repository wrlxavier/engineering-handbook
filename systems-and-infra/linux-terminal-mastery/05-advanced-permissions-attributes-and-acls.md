### 05. Advanced Permissions, Attributes, and Access Control Lists (ACLs)



Linux implements a layered access control model to protect data integrity, enforce process isolation, and prevent unauthorized privilege escalation. At the base lies the traditional POSIX Discretionary Access Control (DAC) model, which assigns read, write, and execute rights across user, group, and other classes. To address real-world administrative demands, the Linux kernel extends this foundation through special execution bits (SUID, SGID, Sticky Bit), low-level filesystem attributes (`chattr`/`lsattr`), and fine-grained Access Control Lists (POSIX.1e ACLs via `setfacl`/`getfacl`).

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Kernel VFS Access Check                         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ 1. Immutable / Append-Only Inode Flags (chattr: S_IMMUTABLE / S_APPEND)│
│    - Evaluated before standard DAC permissions                         │
│    - Blocks even UID 0 (root) operations if set                        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Flag check passes
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ 2. POSIX.1e Access Control Lists (ACLs)                                │
│    - Checked if an Extended ACL exists (indicated by '+' in `ls -l`)   │
│    - Evaluates: Owner -> Named User -> Owning/Named Group & Mask -> Oth│
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ If no Extended ACL present
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ 3. Traditional POSIX DAC Permissions                                   │
│    - Evaluates 16-bit inode mode (`i_mode`)                            │
│    - Matches process EUID/EGID against User, Group, or Other           │
└────────────────────────────────────────────────────────────────────────┘

```

---

#### 1. Traditional POSIX Permissions: Octal Modes, Symbolic Modes, and `umask` Calculation



##### Inode `i_mode` Bitfield Architecture

In Linux, file type and discretionary access permissions are stored inside a single 16-bit integer field (`i_mode`) within the filesystem inode structure (`struct inode` in the Virtual File System layer).

The 16 bits of `i_mode` are partitioned into distinct functional bitmasks:

```
 15  14  13  12   11  10   9    8   7   6    5   4   3    2   1   0
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│   File Type   │ S │ S │ T │ r │ w │ x │ r │ w │ x │ r │ w │ x │
│  (Bits 12-15) │UID│GID│Bit│   User    │   Group   │   Other   │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
 ◄── 4 Bits ───► ◄── 3 Bits ► ◄────────────── 9 Bits ────────────►

```

* **File Type (Bits 12–15)**: Identifies the inode type (`S_IFREG` for regular file `0100000`, `S_IFDIR` for directory `0040000`, `S_IFLNK` for symbolic link `0120000`, `S_IFIFO` for named pipe `0010000`, `S_IFSOCK` for socket `0140000`, `S_IFCHR` for character device `0020000`, `S_IFBLK` for block device `0060000`).


* **Special Execution Bits (Bits 9–11)**: Set-User-ID (`04000`), Set-Group-ID (`02000`), and Sticky Bit (`01000`).
* **Standard Permissions (Bits 0–8)**: Divided into three triplets of 3 bits each representing **User** (owner), **Group**, and **Other** permissions:
* Read (`r`): Weight $4$ (binary `100`).
* Write (`w`): Weight $2$ (binary `010`).
* Execute (`x`): Weight $1$ (binary `001`).



##### Permission Semantics: Regular Files vs. Directories

A common administrative error is assuming identical semantics for permission bits across regular files and directory structures. Because a directory is simply a specialized file containing filename-to-inode mappings, the kernel interprets these bits differently:

| Permission | Regular File Semantic | Directory Semantic |
| --- | --- | --- |
| **Read (`r` / 4)** | Open and read payload bytes (`read(2)`). | Read the list of filenames inside the directory (`opendir(3)`, `readdir(3)`). Without `+x`, file metadata cannot be read. |
| **Write (`w` / 2)** | Modify, overwrite, or truncate payload bytes (`write(2)`, `truncate(2)`). | Create, delete, or rename directory entries (`link(2)`, `unlink(2)`, `rename(2)`). Requires directory `+x`. |
| **Execute (`x` / 1)** | Execute the binary or script image via `execve(2)`. | Traverse into the directory (`chdir(2)`), access child inodes, and inspect file metadata (`stat(2)`, `openat(2)`). |

```bash
# Demonstration: Read without Execute on a Directory
mkdir -p /tmp/perm_demo/secret
touch /tmp/perm_demo/secret/payload.txt
chmod 0400 /tmp/perm_demo/secret   # Read-only, no traverse (+x)

$ ls /tmp/perm_demo/secret
ls: cannot access '/tmp/perm_demo/secret/payload.txt': Permission denied
payload.txt                        # Filename is listed, but inode cannot be resolved!

$ ls -l /tmp/perm_demo/secret
ls: cannot access '/tmp/perm_demo/secret/payload.txt': Permission denied
total 0
-????????? ? ? ? ? ? payload.txt   # Inode metadata retrieval failed

$ cat /tmp/perm_demo/secret/payload.txt
cat: /tmp/perm_demo/secret/payload.txt: Permission denied

```

##### Octal vs. Symbolic Modes via `chmod`

The `chmod` utility alters the 12 least significant bits of an inode's `i_mode` via the `fchmodat(2)` system call.

```bash
# Absolute Octal Assignment
chmod 0750 script.sh     # User=rwx (7), Group=r-x (5), Other=--- (0)
chmod 0644 document.txt  # User=rw- (6), Group=r-- (4), Other=r-- (4)

# Relative Symbolic Modification
chmod u+x,g-w,o=r file   # Add exec to user, remove write from group, set other to read
chmod a+r file           # Add read to all (User, Group, Other)
chmod -R g+rwX shared/   # Recursive: set 'x' on directories and files ALREADY executable

```

The symbolic conditional execute flag (`X`) applies execute permissions strictly to directories or to files that already have execute permissions set for at least one user class. This prevents recursive `chmod -R +x` operations from converting plain text documents into executable binaries.

---

##### File Creation Mask: Mathematical `umask` Calculation

When a process creates a new filesystem object via `openat(2)` (with `O_CREAT`), `creat(2)`, or `mkdir(2)`, it passes an intended mode argument (typically `0666` for files and `0777` for directories). The operating system does not grant these permissions directly; it applies the process's current file creation mask (`umask`).

```
                Intended Permission Mode (e.g., 0666)
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │   Bitwise AND NOT:    │
                     │  mode & (~umask)      │
                     └───────────────────────┘
                                 ▲
                                 │
                     Process Mask (e.g., 0022)
                                 │
                                 ▼
                 Resulting On-Disk Inode Mode (0644)

```

The mathematical transformation is defined strictly by bitwise logic:

$$\text{Final Mode} = \text{Base Mode} \ \& \ (\sim\text{umask})$$

###### Step-by-Step Bitwise Calculation (Base File `0666`, umask `0027`)

1. Represent Base Mode in binary:

$$0666_8 = (110\ 110\ 110)_2$$


2. Represent `umask` in binary:

$$0027_8 = (000\ 010\ 111)_2$$


3. Compute bitwise NOT ($\sim$) of `umask`:

$$\sim(000\ 010\ 111)_2 = (111\ 101\ 000)_2$$


4. Perform bitwise AND ($\&$) between Base Mode and Inverted Mask:
```
  110 110 110   (Base Mode: 0666)
& 111 101 000   (~umask:   ~0027)
─────────────
  110 100 000   (Result:    0640 / rw-r-----)

```



*Common Misconception*: Subtracting octal numbers arithmetic-style (e.g., $666 - 027$) is mathematically incorrect and fails when subtracting odd bits from even bases. Always apply bitwise negation and conjunction.

```bash
# Query current umask
$ umask
0022

# Display umask in symbolic notation
$ umask -S
u=rwx,g=rx,o=rx

# Set a restrictive umask for current shell session
$ umask 0077
$ touch private.key && ls -l private.key
-rw------- 1 user user 0 Sep 27 12:00 private.key

```

###### Administrative Architecture: User Private Groups (UPG)

Modern Linux distributions configure default masks in `/etc/login.defs`, `/etc/profile`, and PAM modules (`pam_umask.so`). Under the **User Private Group (UPG)** scheme:

* Every user is provisioned with an individual, dedicated group sharing their UID (e.g., UID `1001` has primary GID `1001`).
* Because the group is private to that user, the default `umask` can safely be set to `0002` (creating files with `0664` and directories with `0775`).
* If a system relies on a single shared group (e.g., legacy UNIX systems where all users share GID `100` / `users`), the default `umask` must be set to `0022` (`0644` / `0755`) to prevent users from modifying one another's files.

---

#### 2. Special Execution and Security Bits: SUID, SGID, and Sticky Bit



The three high-order bits of the permission segment in `i_mode` alter process execution credentials and directory entry deletion semantics.

```
┌───────────────────┬───────────────────┬───────────────────┐
│     Bit 11 (4)    │     Bit 10 (2)    │     Bit 9 (1)     │
│   Set-User-ID     │   Set-Group-ID    │    Sticky Bit     │
│     (SUID)        │     (SGID)        │  (Restricted Del) │
└───────────────────┴───────────────────┴───────────────────┘

```

##### 1. Set-User-ID (SUID - Octal `4000`, Symbolic `u+s`)

When an executable binary file with the SUID bit is invoked by an unprivileged user, the operating system does not execute the process under the caller's identity. Instead, the kernel sets the process's **Effective User ID (EUID)** to match the file's owner UID.

```
Process Execution with SUID:
Calling User: alice (UID: 1000, GID: 1000)
Target Binary: /usr/bin/passwd (Owned by root: UID 0, Mode: 4755 / -rwsr-xr-x)

               execve("/usr/bin/passwd", ...)
                              │
                              ▼
        ┌───────────────────────────────────────────┐
        │        Kernel Process Credentials         │
        ├───────────────────────────────────────────┤
        │ Real UID (RUID):       1000 (alice)       │
        │ Effective UID (EUID):  0    (root) <──────┼── Elevated via SUID!
        │ Saved UID (SUID):      0    (root)        │
        └───────────────────────────────────────────┘
                              │
                              ▼
Access to /etc/shadow permitted by VFS based on EUID == 0

```

###### Security Hazards and Vulnerability Vectors

* **Privilege Escalation**: Any vulnerability (buffer overflow, path traversal, unchecked environment variable) in an SUID binary gives an attacker root access.
* **Environment Sanitization**: The GNU C Library dynamic linker (`ld.so`) ignores environment variables such as `LD_PRELOAD`, `LD_LIBRARY_PATH`, and `LD_AUDIT` if the process's Real UID differs from its Effective UID, preventing arbitrary shared object injection into SUID processes.
* **Kernel Shebang Shell Script Restriction**: The Linux kernel intentionally **ignores** the SUID bit on interpreted text scripts (files beginning with `#!`). This prevents race conditions where an attacker symlinks or modifies the target script between the interpreter invocation and argument evaluation (Time-of-Check to Time-of-Use / TOCTOU).

---

##### 2. Set-Group-ID (SGID - Octal `2000`, Symbolic `g+s`)

The SGID bit behaves differently depending on whether it is applied to an executable file or a directory:

1. **On Executable Binaries**: The kernel sets the process's **Effective Group ID (EGID)** to the GID of the file's owning group upon execution. Used historically for utilities that require read/write access to spool directories or specific hardware (e.g., `/usr/bin/write` owned by group `tty`).
2. **On Directories (Group Collaboration Architecture)**:
* Any file or subdirectory created within an SGID directory **inherits the Group ID of the directory**, rather than inheriting the primary GID of the creating user.
* Any subdirectory created inside automatically inherits the parent directory's SGID bit.



```
Directory SGID Inheritance Behavior:
Directory: /srv/project (Owner: root, Group: developers, Mode: 2775 / drwxrwsr-x)

User bob (Primary Group: bob, Secondary Group: developers) creates a file:
$ touch /srv/project/notes.txt

Resulting Inode Metadata:
-rw-r--r-- 1 bob developers 0 Sep 27 12:00 /srv/project/notes.txt
                  ▲
                  └─ Group set to 'developers' instead of 'bob'!

```

---

##### 3. Sticky Bit (Restricted Deletion Flag - Octal `1000`, Symbolic `+t`)

* **Historical Context**: On early UNIX systems, setting the sticky bit (`S_ISVTX`) on executable binaries instructed the virtual memory manager to retain the program's text segment in the swap area after termination, accelerating future launches. This behavior is obsolete on modern Linux kernels.
* **Modern Directory Semantics**: When set on a world-writable directory (such as `/tmp` or `/var/tmp`, permissions `1777`), the kernel enforces a restricted deletion policy. A user cannot delete (`unlink(2)`), truncate, or rename files owned by other users inside that directory, even though the directory permissions (`rwxrwxrwx`) would otherwise allow it.

Under POSIX and Linux VFS rules, deletion inside a sticky directory is permitted **only** if at least one of the following conditions is true:

1. The process EUID matches the file owner's UID.
2. The process EUID matches the directory owner's UID.
3. The process possesses the `CAP_FOWNER` capability (e.g., UID `0` / root).

```bash
# Verify permissions on system temporary scratchpads
$ ls -ld /tmp /var/tmp
drwxrwxrwt 28 root root 4096 Sep 27 12:00 /tmp
drwxrwxrwt  8 root root 4096 Sep 27 12:00 /var/tmp

```

---

##### Symbolic Display Rules: Capital vs. Lowercase Bits

When inspecting files with `ls -l`, special bits are integrated into the execution slot of each permission triplet:

| Class | Lowercase Character | Meaning | Capital Character | Meaning |
| --- | --- | --- | --- | --- |
| **User (SUID)** | `s` | SUID set **and** User execute (`x`) enabled. | `S` | SUID set, but User execute (`x`) **disabled**. |
| **Group (SGID)** | `s` | SGID set **and** Group execute (`x`) enabled. | `S` | SGID set, but Group execute (`x`) **disabled**. |
| **Other (Sticky)** | `t` | Sticky bit set **and** Other execute (`x`) enabled. | `T` | Sticky bit set, but Other execute (`x`) **disabled**. |

```bash
# Creating a broken SUID configuration (SUID without execute)
$ touch /tmp/dummy && chmod 4644 /tmp/dummy
$ ls -l /tmp/dummy
-rwSr--r-- 1 user user 0 Sep 27 12:00 /tmp/dummy   # Capital 'S' indicates non-executable SUID

```

##### Auditing and Hardening SUID/SGID Files

SUID/SGID binaries represent entry points for local privilege escalation (LPE). Systems engineers must audit these binaries regularly:

```bash
# Locate all SUID binaries across the root filesystem:
find / -xdev -type f -perm -4000 -exec ls -ld {} + 2>/dev/null

# Locate all SGID binaries across the root filesystem:
find / -xdev -type f -perm -2000 -exec ls -ld {} + 2>/dev/null

# Locate world-writable directories lacking the Sticky Bit:
find / -xdev -type d \( -perm -0002 -a ! -perm -1000 \) -ls 2>/dev/null

```

###### Kernel Hardening: Mount Options and Sysctl Protections

1. **The `nosuid` Mount Option**: Ensures that the SUID and SGID bits are ignored on the mounted filesystem. This should always be applied to `/tmp`, `/var/tmp`, `/dev/shm`, and removable media.


2. **Restricted Deletion Sysctls**:
* `fs.protected_regular`: When set to `1` or `2`, prevents unprivileged users from opening files in world-writable sticky directories under conditions that allow file spoofing.
* `fs.protected_fifos`: Prevents attackers from using named pipes in `/tmp` to intercept writes by root daemons.



---

#### 3. Immutable Attributes and Low-Level Restrictions with `chattr` and `lsattr`

While POSIX DAC permissions are evaluated against user and group identities in the VFS layer, filesystem **extended attributes and flags** are enforced directly by the filesystem driver (e.g., Ext4, XFS, Btrfs) at the inode level.

```
                                  Process Operation
                                (e.g., rm /etc/shadow)
                                          │
                                          ▼
                         ┌─────────────────────────────────┐
                         │   VFS Permission Validation     │
                         │   (Is process UID == 0 / root?) │
                         └────────────────┬────────────────┘
                                          │ Yes (DAC checks pass)
                                          ▼
                         ┌─────────────────────────────────┐
                         │  Underlying Driver Inode Check  │
                         │   Does inode have FS_IMMUTABLE? │
                         └────────────────┬────────────────┘
                                          │
                         ┌────────────────┴────────────────┐
                         │                                 │
                   Flag IS SET                       Flag NOT SET
                         │                                 │
                         ▼                                 ▼
              Kernel returns -EPERM                Operation allowed
              "Operation not permitted"

```

These flags are manipulated using the `FS_IOC_SETFLAGS` and `FS_IOC_GETFLAGS` ioctl system calls, exposed to userspace via the `chattr` and `lsattr` utilities.

##### Inode Attribute Flags Breakdown

The most critical low-level inode attributes include:

| Flag | Name | Operational Behavior | Minimum Capability Required |
| --- | --- | --- | --- |
| `i` | **Immutable** | The file cannot be modified, deleted, renamed, linked to, or truncated. Its metadata cannot be modified, and no write handles can be opened. Even `root` cannot alter it without first clearing this bit. | `CAP_LINUX_IMMUTABLE` |
| `a` | **Append-Only** | The file can only be opened for writing in append mode (`O_APPEND`). It cannot be truncated, overwritten, renamed, or deleted. Critical for audit logs. | `CAP_LINUX_IMMUTABLE` |
| `d` | **No-Dump** | The file is skipped during filesystem backups created via the legacy `dump(8)` utility. | Standard User Ownership |
| `A` | **No Atime** | The kernel does not update the access timestamp (`atime`) on the inode when the file is read, reducing I/O overhead on high-frequency targets.

 | Standard User Ownership |
| `c` | **Compressed** | Instructs the kernel to transparently compress the file's data blocks on disk (driver dependent). | Standard User Ownership |
| `s` | **Secure Deletion** | When deleted, all blocks allocated to the file are zero-filled before being returned to the free block pool (limited support in modern journaled Ext4). | `CAP_SYS_RESOURCE` |
| `u` | **Undeletable** | Marks the file so that if it is unlinked, its data blocks are saved to allow userspace recovery (experimental/rarely supported). | `CAP_SYS_RESOURCE` |

---

##### Practical Application: Securing Critical System Files

System administrators apply the immutable attribute to protect critical configuration files from accidental deletion, automated configuration management drift, or unauthorized modification by rootkits:

```bash
# Securing DNS configuration from overwrite by NetworkManager or DHCP daemons:
chattr +i /etc/resolv.conf

# Verifying attributes:
lsattr /etc/resolv.conf
----i---------e------- /etc/resolv.conf
# Note: 'e' indicates the inode uses Ext4 extent mappings.

# Testing root immunity against immutable files:
$ rm -f /etc/resolv.conf
rm: cannot remove '/etc/resolv.conf': Operation not permitted

$ echo "nameserver 1.1.1.1" > /etc/resolv.conf
bash: /etc/resolv.conf: Permission denied

# To modify the file, root must explicitly clear the attribute:
chattr -i /etc/resolv.conf
echo "nameserver 1.1.1.1" >> /etc/resolv.conf
chattr +i /etc/resolv.conf

```

##### Securing Logging Infrastructure with Append-Only (`+a`)

To prevent attackers from clearing their traces after compromising a system, security-sensitive audit logs should be marked append-only:

```bash
# Configure append-only on the authentication log:
chattr +a /var/log/auth.log

# Verifying write operations:
# 1. Appending lines succeeds:
echo "Manual Security Audit Mark" >> /var/log/auth.log

# 2. Truncation or in-place overwrite fails:
$ : > /var/log/auth.log
bash: /var/log/auth.log: Operation not permitted

# 3. Text editors fail (editors use temporary files and rename(2)):
$ sed -i '/Failed password/d' /var/log/auth.log
sed: cannot rename /var/log/sedXXXXXX: Operation not permitted

```

*Operational Note on Log Rotation*: Setting `chattr +a` on system logs will cause standard `logrotate` jobs to fail if they use the default `rename`-and-compress strategy. Log rotation must be configured with `copytruncate` mode, or the rotation script must run pre- and post-commands (`prerotate` / `postrotate`) to clear and reapply the attribute via `chattr -a` and `chattr +a`.

##### Linux Capabilities: Decoupling Immutability from Superuser Identity

Under the POSIX capabilities model, traditional superuser privileges are partitioned into discrete units. Modifying the `+i` or `+a` attributes does not simply require UID `0`; it requires the **`CAP_LINUX_IMMUTABLE`** capability.

If a system administrator drops `CAP_LINUX_IMMUTABLE` from a container runtime or systemd service unit (e.g., using `CapabilityBoundingSet=~CAP_LINUX_IMMUTABLE`), processes within that environment cannot clear the immutable bit, even if they obtain root privileges.

---

#### 4. Granular Access Control Beyond the Owner/Group Model Using ACLs (`getfacl`, `setfacl`)



##### Architectural Limitations of Traditional POSIX DAC

The standard POSIX discretionary model limits access control by binding an inode to exactly **one** owner UID and **one** owning GID. Consider this common operational requirement:

* A shared project directory `/srv/billing` is owned by `alice:finance`.
* The external auditor `marcus` needs read-only access.
* The contractor `charlie` needs read-write access to specific spreadsheets.
* The intern group `interns` must be denied all access.

Under standard DAC, this model cannot be implemented without creating multiple synthetic groups, nesting group memberships, or compromising directory permissions.

To solve this problem, modern Linux filesystems support **POSIX.1e Draft Standard Access Control Lists (ACLs)**.

---

##### Extended Attribute Storage Mechanics

ACLs are not stored within the fixed-size 256-byte or 512-byte inode table. Instead, they are stored in the filesystem's **Extended Attributes (`xattr`)** storage space under the `system` namespace:

* Access ACLs: Stored under `system.posix_acl_access`.
* Default ACLs: Stored under `system.posix_acl_default` (directories only).

When an extended ACL is assigned to an inode, the permissions string displayed by `ls -l` appends a plus sign (`+`):

```bash
-rw-r--r--  1 alice alice 1024 Sep 27 12:00 standard.txt
-rw-rwxr--+ 1 alice alice 1024 Sep 27 12:00 acl_enabled.txt
          ▲
          └─ Plus sign indicates presence of Extended ACL in xattrs

```

---

##### ACL Entry Types and Structural Syntax

An ACL consists of multiple delimited entries defining permissions for specific subjects:

| ACL Entry Syntax | Description | DAC Equivalent |
| --- | --- | --- |
| `u::rwx` | Owning User (file owner permissions). | Standard User triplet (`chmod u=...`) |
| `u:username:rwx` | **Named User**: Grants specific rights to an individual user. | *No DAC equivalent* |
| `g::rwx` | Owning Group permissions. | Standard Group triplet (`chmod g=...`) |
| `g:groupname:rwx` | **Named Group**: Grants specific rights to an arbitrary group. | *No DAC equivalent* |
| `m::rwx` | **ACL Mask**: Upper limit on effective permissions for named users, groups, and owning group. | *No DAC equivalent* |
| `o::rwx` | Other (everyone else). | Standard Other triplet (`chmod o=...`) |

---

##### The ACL Mask and Effective Permission Calculation

The **Mask** (`m::`) is the most important concept in POSIX extended ACLs. It defines the **maximum permission limit** for all named users, named groups, and the owning group. It does **not** restrict the file owner or the "other" class.

```
Named User Entry:      u:marcus:rwx  (Binary: 111)
                                      ▲
                                      │ Bitwise AND
                                      ▼
ACL Mask Entry:        m::r-x        (Binary: 101)
                                      │
                                      ▼
Effective Permission:                 r-x (Binary: 101)
(Marcus is denied write access despite having 'w' in his entry!)

```

```bash
# Inspecting ACLs via getfacl
$ getfacl document.pdf
# file: document.pdf
# owner: alice
# group: finance
user::rw-
user:marcus:rwx                 #effective:r-x  <-- Truncated by mask!
group::r--
group:contractors:rw-           #effective:r--  <-- Truncated by mask!
mask::r-x
other::---

```

###### The Critical Interaction Between `chmod` and ACL Masks

When an extended ACL is added to an inode, the group triplet displayed by `ls -l` no longer reflects the owning group's permissions! **It displays the current ACL Mask.**

```
Without ACL:  -rw-r-----  1 alice finance ...  (Group triplet = 'finance' group: r--)
With ACL:     -rw-rwxr--+ 1 alice finance ...  (Group triplet = ACL MASK: rwx!)

```

If an administrator runs `chmod g-w file` on an ACL-enabled file:

* The kernel does **not** update the owning group's permissions.
* The kernel **modifies the ACL Mask**, revoking write access from **all** named users and named groups at once.

---

##### Default ACLs (Inheritance Mechanics on Directories)

Standard ACLs apply strictly to existing filesystem objects. **Default ACLs** can only be assigned to directories; they serve as an inheritance template for newly created children.

When a new file or directory is created inside a directory that has Default ACLs:

1. The child object automatically inherits the parent's Default ACLs as its **Access ACL**.
2. If the child is a directory, it also inherits the parent's Default ACLs as its own **Default ACL** (recursive inheritance).
3. The process's standard `umask` is **completely ignored** for that creation operation; permission boundaries are dictated entirely by the inherited Default ACL Mask.

```bash
# Set default read-write permissions for the 'devops' group on a directory:
setfacl -d -m g:devops:rw- /srv/shared
setfacl -d -m m::rwx /srv/shared

# Verify directory ACL structure:
$ getfacl /srv/shared
# file: srv/shared
# owner: root
# group: root
user::rwx
group::r-x
other::r-x
default:user::rwx
default:group::r-x
default:group:devops:rw-
default:mask::rwx
default:other::r-x

```

---

##### Practical Management with `setfacl` and `getfacl`

The `setfacl` and `getfacl` utilities manage access control lists from the command line:

```bash
# 1. Granting read-write access to a named user:
setfacl -m u:charlie:rw- /srv/data/report.xlsx

# 2. Granting read-only access to a named group:
setfacl -m g:auditors:r-- /srv/data/report.xlsx

# 3. Explicitly setting the ACL mask:
setfacl -m m::r-x /srv/data/report.xlsx

# 4. Removing a specific ACL entry:
setfacl -x u:charlie /srv/data/report.xlsx

# 5. Removing ALL extended ACL entries (reverting to standard POSIX DAC):
setfacl -b /srv/data/report.xlsx

# 6. Applying ACLs recursively to a directory hierarchy:
setfacl -R -m g:marketing:rx /srv/media/

# 7. Backing up and restoring entire ACL trees:
# Export all ACLs across a storage tree:
getfacl -R /srv/storage > /tmp/storage_acls.bak

# Restore ACLs from the backup manifest:
setfacl --restore=/tmp/storage_acls.bak

```

---

#### 5. Practical Laboratories & Diagnostic Walkthroughs

##### Lab 1: Building a Multi-User Collaborative Workspace

Create an isolated shared directory (`/srv/collab`) where members of the `engineering` and `analytics` teams can collaborate safely:

* Files created by anyone must be editable by both groups.
* Individual users must not be able to delete files created by other users (Sticky Bit protection).
* External auditors (`auditor_dave`) must have read-only access to everything created inside.

```bash
#!/usr/bin/env bash
set -euo pipefail

LAB_DIR="/tmp/collab_lab"
rm -rf "${LAB_DIR}"
mkdir -p "${LAB_DIR}/workspace"

echo "[*] Creating synthetic groups and testing users..."
sudo groupadd -f engineering
sudo groupadd -f analytics
sudo id -u alice &>/dev/null || sudo useradd -M -s /usr/sbin/nologin alice
sudo id -u bob &>/dev/null || sudo useradd -M -s /usr/sbin/nologin bob
sudo id -u auditor_dave &>/dev/null || sudo useradd -M -s /usr/sbin/nologin auditor_dave

sudo usermod -aG engineering alice
sudo usermod -aG analytics bob

echo "[*] Configuring base ownership and collaborative bits (SGID + Sticky Bit)..."
sudo chown root:engineering "${LAB_DIR}/workspace"

# Set base permissions: Read/Write/Execute for Owner and Group, traverse for Others
# Add SGID (2000) for group inheritance + Sticky Bit (1000) for deletion protection
sudo chmod 3770 "${LAB_DIR}/workspace"

echo "[*] Establishing Default ACLs for cross-team inheritance..."
# Ensure the secondary group 'analytics' has default read-write-execute:
sudo setfacl -d -m g:analytics:rwx "${LAB_DIR}/workspace"
# Ensure the primary group 'engineering' has default read-write-execute:
sudo setfacl -d -m g:engineering:rwx "${LAB_DIR}/workspace"
# Grant the auditor read-only access across all new objects:
sudo setfacl -d -m u:auditor_dave:r-x "${LAB_DIR}/workspace"
# Ensure the mask permits full group collaboration:
sudo setfacl -d -m m::rwx "${LAB_DIR}/workspace"

echo "[*] Testing file creation as 'alice' via sudo..."
sudo -u alice touch "${LAB_DIR}/workspace/alice_design.doc"

echo "[*] Inspecting generated file metadata and inherited ACLs:"
ls -l "${LAB_DIR}/workspace/alice_design.doc"
getfacl "${LAB_DIR}/workspace/alice_design.doc"

echo "[*] Verifying write access for 'bob' (analytics group)..."
sudo -u bob bash -c "echo 'Bob review notes' >> ${LAB_DIR}/workspace/alice_design.doc"

echo "[*] Verifying Sticky Bit deletion protection (bob attempts to delete alice's file)..."
set +e
sudo -u bob rm "${LAB_DIR}/workspace/alice_design.doc" 2>/tmp/err.log
EXIT_CODE=$?
set -e

if [ ${EXIT_CODE} -ne 0 ]; then
    echo "[SUCCESS] Sticky bit blocked deletion by non-owner: $(cat /tmp/err.log)"
fi

echo "[*] Cleaning up lab environment..."
rm -rf "${LAB_DIR}" /tmp/err.log

```

---

##### Lab 2: Tamper-Resistant Security Hardening with Attributes and Capabilities

Audit an application logging pipeline, simulate a root compromise, and use low-level filesystem attributes to protect audit integrity.

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR="/tmp/immutable_lab"
rm -rf "${WORKDIR}"
mkdir -p "${WORKDIR}"
cd "${WORKDIR}"

echo "[*] Initializing mock critical service files..."
echo "system_status=operational" > service.conf
echo "[$(date)] SYSTEM BOOT COMPLETED" > audit.log

echo "[*] Applying low-level protections via chattr..."
# Make configuration completely immutable:
sudo chattr +i service.conf
# Make audit log append-only:
sudo chattr +a audit.log

echo "[*] Validating applied attributes:"
lsattr service.conf audit.log

echo "[*] Simulating malicious activity by UID 0 (root attacker)..."

echo "Attempt 1: Overwriting immutable configuration..."
set +e
sudo bash -c "echo 'system_status=compromised' > service.conf" 2>/tmp/attack_err.log
set -e
echo "Result: $(head -n 1 /tmp/attack_err.log)"

echo "Attempt 2: Deleting audit log..."
set +e
sudo rm -f audit.log 2>/tmp/attack_err.log
set -e
echo "Result: $(head -n 1 /tmp/attack_err.log)"

echo "Attempt 3: Truncating append-only audit log..."
set +e
sudo bash -c ": > audit.log" 2>/tmp/attack_err.log
set -e
echo "Result: $(head -n 1 /tmp/attack_err.log)"

echo "Attempt 4: Legitimate log appending (authorized action)..."
sudo bash -c "echo '[$(date)] SECURITY WARNING DETECTED' >> audit.log"
echo "Log Contents Successfully Preserved:"
cat audit.log

echo "[*] Tearing down protections..."
sudo chattr -i service.conf
sudo chattr -a audit.log
rm -rf "${WORKDIR}" /tmp/attack_err.log

```

---

##### Lab 3: Auditing SUID Binaries and Preventing Injection Attacks

Trace how environment variables are handled during privilege transitions, and observe how `nosuid` mounts neutralize escalation attempts.

```bash
#!/usr/bin/env bash
set -euo pipefail

WORKDIR="/tmp/suid_lab"
rm -rf "${WORKDIR}"
mkdir -p "${WORKDIR}/mountpoint"
cd "${WORKDIR}"

echo "[*] 1. Compiling a C program that inspects its Effective UID..."
cat << 'EOF' > test_suid.c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main() {
    printf("[+] Real UID      : %d\n", getuid());
    printf("[+] Effective UID : %d\n", geteuid());
    if (geteuid() == 0) {
        printf("[!] ROOT PRIVILEGES CONFIRMED!\n");
    } else {
        printf("[-] Running with unprivileged credentials.\n");
    }
    return 0;
}
EOF

gcc -Wall test_suid.c -o test_suid
sudo chown root:root test_suid
sudo chmod 4755 test_suid  # Set SUID bit

echo "[*] 2. Executing SUID binary from a standard filesystem..."
ls -l test_suid
./test_suid

echo "[*] 3. Creating an isolated loopback device with 'nosuid' protection..."
dd if=/dev/zero of=backing.img bs=1M count=20 status=none
mkfs.ext4 -F backing.img >/dev/null

# Mount with nosuid flag:
sudo mount -o loop,nosuid backing.img "${WORKDIR}/mountpoint"

echo "[*] 4. Copying SUID binary onto the 'nosuid' filesystem..."
sudo cp "${WORKDIR}/test_suid" "${WORKDIR}/mountpoint/"
ls -l "${WORKDIR}/mountpoint/test_suid"

echo "[*] 5. Executing binary from 'nosuid' filesystem..."
# The binary still has mode 4755, but the kernel ignores the bit:
"${WORKDIR}/mountpoint/test_suid"

echo "[*] Cleaning up resources..."
sudo umount "${WORKDIR}/mountpoint"
rm -rf "${WORKDIR}"

```

---

#### 6. Comprehensive Access Control Reference Matrix

Use this matrix to understand where each mechanism operates, how it is stored, and which access checks take precedence during VFS path evaluation:

| Security Layer | Governing Entity | Underlying Kernel Storage | Command-Line Interface | Precedence / Evaluation Order | Bypass / Privilege Requirement |
| --- | --- | --- | --- | --- | --- |
| **Filesystem Attributes** | Filesystem Driver (Ext4, XFS, etc.)

 | Inode internal flags (`i_flags`, Ext4 `ext4_iloc`)

 | `chattr`, `lsattr`<br> | **1st**: Checked before DAC. Blocks all file modification if set. | Requires `CAP_LINUX_IMMUTABLE` capability to toggle `+i` or `+a`. UID 0 cannot bypass without the capability. |
| **POSIX.1e ACLs** | Virtual File System (VFS) POSIX ACL subsystem

 | Extended Attributes (`xattr`), namespace `system.posix_acl_*` | `setfacl`, `getfacl`<br> | **2nd**: Checked if an extended ACL exists (`+` flag). Evaluates Owner, Named Users, Groups via Mask, then Other. | Discretionary: File owner or `CAP_FOWNER` can alter or remove ACL entries. |
| **Traditional POSIX DAC** | Virtual File System (VFS) DAC core

 | Inode `i_mode` 16-bit integer (low 9 bits)

 | `chmod`, `umask`<br> | **3rd**: Evaluated when no extended ACL is attached to the inode. Matches EUID/EGID against User, Group, or Other. | Discretionary: File owner or processes with `CAP_DAC_OVERRIDE` (root) bypass read/write checks. |
| **Special Execution Bits (SUID/SGID)** | Kernel Process Management (`execve(2)`) | Inode `i_mode` bits 11 and 10 (`04000`, `02000`)

 | `chmod u+s`, `chmod g+s`<br> | **Execution time**: Modifies process credentials (EUID/EGID) when the binary binary image is executed. | Neutralized by the `nosuid` mount option, `PR_SET_NO_NEW_PRIVS` prctl flags, or user namespaces.

 |
| **Sticky Bit (Restricted Deletion)** | VFS Inode Operations (`unlinkat(2)`, `renameat(2)`)

 | Inode `i_mode` bit 9 (`01000`)

 | `chmod +t`<br> | **Directory mutation**: Checked when an unprivileged process attempts to delete or rename an entry in a directory. | Only file owner, directory owner, or processes with `CAP_FOWNER` can delete or rename entries. |
