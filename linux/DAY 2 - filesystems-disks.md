# Linux Filesystems, Mounts, Inodes & Disk Troubleshooting

> Job-switch study notes — learn the commands, then practice the troubleshooting flow.

## 1. What you need to be able to do

By the end of this topic, you should be able to:

- Use `df`, `du`, `lsblk`, and `mount` to locate capacity or mount problems.
- Trace a full filesystem to the largest directories and files.
- Explain inode exhaustion.
- Diagnose disk-space and inode problems safely in production.
- Distinguish filesystem capacity problems from block-device, mount, and inode problems.

---

# 2. Linux Filesystem Mental Model

Start with this basic hierarchy:

```text
Physical / Virtual Disk
        |
        v
Block Device
        |
        v
Partition / LVM Volume
        |
        v
Filesystem
        |
        v
Mount Point
        |
        v
Directory Tree
        |
        v
Files
```

Example:

```text
/dev/nvme0n1
      |
      +-- /dev/nvme0n1p1
              |
              +-- ext4 filesystem
                      |
                      +-- mounted at /
```

A disk/device is not automatically a usable directory tree. A filesystem must be mounted at a mount point before its files are accessible through the normal directory hierarchy.

---

# 3. Filesystem vs Mount Point

A **filesystem** is the structure used to organize files and metadata on storage.

A **mount point** is a directory where that filesystem is attached to the Linux directory tree.

Example:

```text
Filesystem: /dev/sdb1
Mount point: /data
```

After mounting:

```text
/dev/sdb1
    |
    v
   /data
    |
    +-- app/
    +-- logs/
    +-- backups/
```

Useful command:

```bash
mount
```

Or:

```bash
findmnt
```

---

# 4. df — Filesystem Capacity

Use `df` to answer:

> How much space is available on each mounted filesystem?

Basic:

```bash
df
```

Human-readable:

```bash
df -h
```

Important columns:

```text
Filesystem
Size
Used
Avail
Use%
Mounted on
```

Example:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  100G   92G    8G  92% /
/dev/sdb1       500G  300G  200G  60% /data
```

Interpretation:

- `/` is 92% full.
- `/data` is 60% full.
- The filesystem mounted at `/` needs investigation if the usage continues increasing.

### Inode usage

Always remember:

```bash
df -i
```

This shows inode usage instead of block-space usage.

Human-readable inode information:

```bash
df -ih
```

---

# 5. du — Directory and File Usage

Use `du` to answer:

> Which directories/files are consuming space?

Basic:

```bash
du
```

Human-readable:

```bash
du -h
```

Summarize a directory:

```bash
du -sh /var/log
```

Example:

```text
12G    /var/log
```

This tells you that `/var/log` consumes approximately 12 GB.

---

# 6. Find the Largest Directories

A very useful troubleshooting command:

```bash
du -xh --max-depth=1 /var | sort -h
```

This shows the approximate usage of immediate subdirectories under `/var`, sorted by size.

For the root filesystem:

```bash
du -xh --max-depth=1 / | sort -h
```

To show the largest directories first:

```bash
du -xh --max-depth=1 /var | sort -hr
```

Typical investigation:

```text
/
└── /var
    ├── /var/log
    ├── /var/lib
    ├── /var/cache
    └── /var/tmp
```

If `/var/lib` is huge, investigate it next:

```bash
du -xh --max-depth=1 /var/lib | sort -hr
```

Continue drilling down until the source is identified.

---

# 7. Find the Largest Files

A common approach:

```bash
find /var -type f -printf '%s %p\n' 2>/dev/null | sort -nr | head
```

For human-readable output:

```bash
find /var -type f -exec du -h {} + 2>/dev/null | sort -hr | head
```

Be careful when running broad `find` commands on large production systems because they can scan many files.

---

# 8. lsblk — Block Devices

Use `lsblk` to understand disks, partitions, and block devices.

Basic:

```bash
lsblk
```

Useful form:

```bash
lsblk -f
```

This can show:

- Device
- Partition
- Filesystem type
- UUID
- Mount point

Example:

```text
NAME        FSTYPE FSVER LABEL UUID                                 MOUNTPOINT
nvme0n1
├─nvme0n1p1 ext4   1.0         abc...                               /
└─nvme0n1p2 xfs    5.0         def...                               /data
```

Mental model:

```text
lsblk
  ↓
What storage devices and partitions exist?

df
  ↓
How full are the mounted filesystems?

du
  ↓
What directories/files are consuming the space?

mount/findmnt
  ↓
Where are filesystems mounted?
```

---

# 9. mount — View Mounts

Show currently mounted filesystems:

```bash
mount
```

A more structured command:

```bash
findmnt
```

Find information about a specific mount:

```bash
findmnt /
```

or:

```bash
findmnt /data
```

---

# 10. Mount a Filesystem

General syntax:

```bash
mount <device> <mount-point>
```

Example:

```bash
mount /dev/sdb1 /data
```

After mounting:

```bash
df -h /data
```

Verify:

```bash
findmnt /data
```

> Do not experiment with mounting/unmounting production filesystems without understanding what services depend on them.

---

# 11. /etc/fstab

`/etc/fstab` contains persistent filesystem mount configuration.

Check it:

```bash
cat /etc/fstab
```

Typical entry:

```text
UUID=xxxx-xxxx  /data  ext4  defaults  0  2
```

Important idea:

```text
Manual mount
    ↓
mount command
    ↓
temporary until unmounted/reboot depending on configuration

Persistent mount
    ↓
/etc/fstab
    ↓
system boot / mount -a
```

Before making changes to `fstab`, validate carefully. A bad entry can cause boot or service problems.

---

# 12. Inodes

An **inode** stores filesystem metadata about a file.

An inode can contain information such as:

- File type
- Permissions
- Owner
- Group
- Size
- Timestamps
- References to the file's data blocks

The filename itself is associated with the inode through directory entries.

Important:

> Every file consumes an inode.

Therefore a filesystem can have free disk space but still be unable to create new files if it runs out of inodes.

---

# 13. Disk Space vs Inode Exhaustion

There are two different capacity problems.

### Block-space exhaustion

```text
Disk space → 100%
Inodes     → available
```

Typical check:

```bash
df -h
```

### Inode exhaustion

```text
Disk space → may still have free space
Inodes     → 100%
```

Check:

```bash
df -i
```

This distinction is extremely important in production.

---

# 14. Example of Inode Exhaustion

Imagine:

```text
Filesystem size: 100 GB
Used space:      40 GB
Free space:      60 GB
Inodes used:     100%
```

You may still see:

```text
Avail: 60G
```

but creating a new file can fail because there are no free inodes.

Common causes:

- Millions of tiny files
- Application-generated temporary files
- Session files
- Mail queues
- Cache directories
- Container/image layer files
- Log fragments
- Application bugs creating files continuously

---

# 15. Finding Inode-Heavy Directories

A useful starting point:

```bash
df -i
```

Then investigate directories.

One approach:

```bash
find /var -xdev -type f | cut -d/ -f1-4 | sort | uniq -c | sort -nr | head
```

A more direct directory-count approach:

```bash
find /var -xdev -type f -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -nr | head
```

The exact command can vary depending on the filesystem, scale, and available tools.

The important troubleshooting concept is:

```text
df -i
  ↓
Identify filesystem with high inode usage
  ↓
Find directories containing huge numbers of files
  ↓
Identify application / service creating them
  ↓
Clean up safely
  ↓
Fix retention / application behavior
```

---

# 16. The -xdev Option

When searching a filesystem, `-xdev` prevents `find` from crossing into other mounted filesystems.

Example:

```bash
find / -xdev -type f
```

This is useful when investigating a specific filesystem.

For example, if `/` is full but `/data` is a separate filesystem, you don't want a root-filesystem investigation accidentally scanning all of `/data`.

---

# 17. Production Disk Troubleshooting Flow

Suppose monitoring reports:

```text
/root filesystem > 90%
```

### Step 1 — Confirm the filesystem

```bash
df -h
```

Identify:

```text
/dev/nvme0n1p1  100G  92G  8G  92% /
```

### Step 2 — Check inodes

```bash
df -i
```

Determine whether the problem is:

- Space exhaustion
- Inode exhaustion
- Both

### Step 3 — Understand the storage layout

```bash
lsblk -f
```

### Step 4 — Check mounts

```bash
findmnt
```

### Step 5 — Find large top-level directories

```bash
du -xh --max-depth=1 / | sort -hr
```

### Step 6 — Drill down

If `/var` is large:

```bash
du -xh --max-depth=1 /var | sort -hr
```

If `/var/log` is large:

```bash
du -xh --max-depth=1 /var/log | sort -hr
```

Continue until the source is identified.

### Step 7 — Find large files

```bash
find /var/log -type f -exec du -h {} + 2>/dev/null | sort -hr | head
```

### Step 8 — Identify the owner/application

Ask:

- Which service generated these files?
- Is log rotation configured?
- Is there a retention policy?
- Is a temporary directory growing?
- Is a backup/cache process responsible?
- Is a container or application generating unexpected files?

### Step 9 — Clean up safely

Do **not** blindly delete files.

First determine:

- Whether the file is actively used
- Whether it is safe to remove
- Whether the application needs it
- Whether a retention policy exists

### Step 10 — Prevent recurrence

Examples:

- Configure log rotation
- Set retention policies
- Fix application cleanup
- Configure container log limits
- Increase filesystem capacity when justified
- Add monitoring/alerts

---

# 18. Important Production Scenario: Deleted File Still Uses Disk

A particularly important troubleshooting case:

```text
df -h
→ filesystem is 95% full

du
→ doesn't explain the usage
```

One possible reason is a **deleted file that is still open by a running process**.

The file's directory entry has been removed, but the process still holds the file open.

Check with:

```bash
lsof +L1
```

or:

```bash
lsof | grep deleted
```

Concept:

```text
Application
    |
    | opens large.log
    ↓
large.log
    |
    | file deleted
    ↓
directory entry disappears
    |
    ↓
process still holds file descriptor
    |
    ↓
disk space remains allocated
```

Restarting the relevant process may release the space, but determine the service impact before doing so.

---

# 19. Common Mistakes

### Mistake 1

Using only:

```bash
df -h
```

and assuming it tells you which file is responsible.

It does not. `df` reports filesystem usage; use `du` to investigate directory/file usage.

### Mistake 2

Checking only disk space.

Always consider:

```bash
df -h
df -i
```

### Mistake 3

Running `du /` blindly on a large production server.

It can be expensive and may cross into other mounted filesystems.

Prefer targeted investigation and `-xdev` where appropriate.

### Mistake 4

Immediately deleting large files.

First identify the application, file purpose, retention requirements, and whether the file is actively being written.

### Mistake 5

Using `rm` as the first response to a disk alert.

The correct approach is:

```text
Measure
  ↓
Identify
  ↓
Understand
  ↓
Clean safely
  ↓
Prevent recurrence
```

---

# 20. Command Cheat Sheet

| Goal | Command |
|---|---|
| Filesystem usage | `df -h` |
| Inode usage | `df -i` |
| Filesystem + inode usage | `df -h`, `df -ih` |
| Block devices | `lsblk` |
| Filesystem information | `lsblk -f` |
| Current mounts | `mount` |
| Structured mount information | `findmnt` |
| Directory size | `du -sh <dir>` |
| Top-level directory sizes | `du -xh --max-depth=1 <dir>` |
| Sort largest first | `du -xh --max-depth=1 <dir> \| sort -hr` |
| Find large files | `find ... -type f ... \| sort -hr` |
| Persistent mounts | `cat /etc/fstab` |
| Deleted-but-open files | `lsof +L1` |

---

# 21. Interview Questions

> Click the arrow next to each question to reveal the answer.

<details>
<summary>1. What is a filesystem?</summary>

A filesystem is the structure and set of rules Linux uses to store, organize, name, and retrieve files and directories on a storage device.

Examples include **ext4, XFS, and Btrfs**.

</details>

<details>
<summary>2. What is a mount point?</summary>

A mount point is a directory where a filesystem is attached to the Linux directory tree.

Example:

```text
/dev/sdb1 → /data
```

After mounting, the contents of the filesystem are accessible through `/data`.

</details>

<details>
<summary>3. What does df -h show?</summary>

`df -h` shows filesystem-level disk usage in human-readable units.

It shows values such as:

- Total size
- Used space
- Available space
- Usage percentage
- Mount point

Use it to identify **which filesystem is running out of space**.

</details>

<details>
<summary>4. What does du -sh show?</summary>

`du -sh <dir>` shows the total disk space consumed by a directory in human-readable form.

Example:

```bash
du -sh /var/log
```

This helps identify **which directory is consuming space**.

</details>

<details>
<summary>5. What is the difference between df and du?</summary>

`df` reports **filesystem-level usage**, while `du` reports **directory/file-level usage**.

Typical troubleshooting:

```text
df -h
  ↓
Which filesystem is full?
  ↓
du
  ↓
Which directory/file is consuming the space?
```

</details>

<details>
<summary>6. What does lsblk -f show?</summary>

`lsblk -f` shows block devices together with filesystem information.

It can show:

- Device/partition
- Filesystem type
- Filesystem label
- UUID
- Mount point

It is useful for understanding the relationship between **disks, partitions, filesystems, and mounts**.

</details>

<details>
<summary>7. How do you find which filesystem is full?</summary>

Run:

```bash
df -h
```

Look at the `Use%` column and identify the filesystem with high utilization.

Also check inode usage:

```bash
df -i
```

</details>

<details>
<summary>8. How do you find the largest directory?</summary>

Start with:

```bash
du -xh --max-depth=1 / | sort -hr
```

Then drill into the largest directory:

```bash
du -xh --max-depth=1 /var | sort -hr
```

Continue until you identify the directory responsible for the growth.

</details>

<details>
<summary>9. How do you find the largest files?</summary>

One approach is:

```bash
find /var -type f -printf '%s %p\n' 2>/dev/null | sort -nr | head
```

For human-readable output:

```bash
find /var -type f -exec du -h {} + 2>/dev/null | sort -hr | head
```

Use targeted paths on production systems because broad searches can be expensive.

</details>

<details>
<summary>10. What is an inode?</summary>

An inode is a filesystem data structure that stores metadata about a file.

It can contain information such as:

- File type
- Permissions
- Owner/group
- Size
- Timestamps
- References to the file's data blocks

The filename is associated with the inode through a directory entry.

</details>

<details>
<summary>11. Can a filesystem have free disk space but no inodes?</summary>

Yes.

A filesystem can have free block space but reach **100% inode usage**. In that situation, creating new files can fail even though `df -h` shows free space.

Check with:

```bash
df -i
```

A common cause is a very large number of small files.

</details>

<details>
<summary>12. How do you check inode usage?</summary>

Run:

```bash
df -i
```

For human-readable inode information:

```bash
df -ih
```

Check the `IUse%` column to see inode utilization.

</details>

<details>
<summary>13. What causes inode exhaustion?</summary>

Common causes include:

- Millions of small files
- Temporary files
- Session files
- Cache files
- Mail queues
- Container/image layer files
- Log fragments
- Application bugs that continuously create files

The key point is that **each file consumes an inode**.

</details>

<details>
<summary>14. How would you troubleshoot a filesystem at 100%?</summary>

Use a structured flow:

```text
df -h
  ↓
Identify the full filesystem
  ↓
df -i
  ↓
Check space vs inode exhaustion
  ↓
lsblk -f
  ↓
Understand the storage layout
  ↓
findmnt
  ↓
Check mounts
  ↓
du
  ↓
Find large directories
  ↓
find
  ↓
Find large files
  ↓
lsof +L1
  ↓
Check deleted-but-open files
  ↓
Identify the application
  ↓
Clean safely
  ↓
Prevent recurrence
```

Do not immediately delete files without understanding their purpose and service impact.

</details>

<details>
<summary>15. What does -xdev do?</summary>

`find -xdev` prevents `find` from crossing into directories on other mounted filesystems.

Example:

```bash
find / -xdev -type f
```

This is useful when investigating one filesystem without unintentionally scanning other mounted filesystems.

</details>

<details>
<summary>16. Why might df show 95% usage while du doesn't explain it?</summary>

A common reason is a **deleted-but-open file**.

A process may still have a file open after its directory entry has been deleted. The space remains allocated until the process closes the file descriptor.

Check with:

```bash
lsof +L1
```

The relevant process may need to be restarted, but first assess the service impact.

</details>

<details>
<summary>17. What is a deleted-but-open file?</summary>

It is a file whose directory entry has been deleted while a running process still holds the file open.

The filename disappears from the directory, but the file's disk blocks remain allocated until the process closes the file descriptor.

Check with:

```bash
lsof +L1
```

</details>

<details>
<summary>18. How would you safely clean a production filesystem?</summary>

First identify the cause before removing anything:

1. Confirm the affected filesystem with `df -h`.
2. Check inode usage with `df -i`.
3. Locate large directories/files with `du` and `find`.
4. Identify the owning application or service.
5. Check whether files are actively being written or used.
6. Verify retention and backup requirements.
7. Clean up using the application's supported process where possible.
8. Confirm that usage drops.
9. Fix the underlying retention, logging, cache, or application issue.

Avoid blindly using `rm` as the first response.

</details>

<details>
<summary>19. What is /etc/fstab?</summary>

`/etc/fstab` is the configuration file that defines filesystems and how they should be mounted, commonly during boot.

A typical entry contains:

```text
UUID=xxxx-xxxx  /data  ext4  defaults  0  2
```

Changes should be validated carefully because an incorrect entry can cause boot or service problems.

</details>

<details>
<summary>20. What is the difference between a block device, filesystem, and mount point?</summary>

**Block device:** represents storage that can provide blocks of data, such as `/dev/sdb`.

**Filesystem:** organizes data, files, directories, and metadata on that storage, such as ext4 or XFS.

**Mount point:** the directory where the filesystem is attached to the Linux directory tree, such as `/data`.

Mental model:

```text
Block device
    ↓
Filesystem
    ↓
Mount point
    ↓
Directories and files
```

</details>

---

# 22. Practical Lab

Practice this on a Linux VM/container rather than production.

### Lab 1 — Filesystem discovery

```bash
df -h
df -i
lsblk -f
findmnt
```

Write down:

- Root filesystem
- Filesystem type
- Total size
- Used size
- Available size
- Inode usage
- Mount point

### Lab 2 — Find large directories

```bash
sudo du -xh --max-depth=1 /var | sort -hr
```

Pick the largest directory and drill down.

### Lab 3 — Find large files

Use `find` to identify the largest files under a safe test directory.

### Lab 4 — Understand inodes

Create many small files in a temporary test directory and observe:

```bash
df -i
```

Then remove them and observe the inode count again.

### Lab 5 — Mount investigation

Inspect:

```bash
lsblk -f
findmnt
cat /etc/fstab
```

Do not modify `fstab` unless you understand the consequences.

---

# 23. Production Mental Model

When you receive:

> **Disk usage alert: 95%**

Think:

```text
ALERT
  ↓
df -h
  ↓
Which filesystem?
  ↓
df -i
  ↓
Space or inode problem?
  ↓
lsblk -f
  ↓
Understand device/filesystem
  ↓
findmnt
  ↓
Understand mount
  ↓
du
  ↓
Which directory?
  ↓
du again
  ↓
Which subdirectory?
  ↓
find
  ↓
Which files?
  ↓
lsof +L1
  ↓
Could deleted-open files explain the difference?
  ↓
Identify application
  ↓
Clean safely
  ↓
Prevent recurrence
```

## Next related topics

After this topic, continue with:

1. Linux permissions and ownership
2. Linux users and groups
3. systemd / systemctl
4. journalctl and log troubleshooting
5. CPU troubleshooting
6. Memory troubleshooting and OOM killer
7. Process priorities — nice / renice
8. Linux networking troubleshooting
