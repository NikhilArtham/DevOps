# Linux Filesystems, Mounts, Inodes & Disk Troubleshooting

> Job-switch study notes — learn the commands, then practice the troubleshooting flow in `linux/LABS.md`.

## 1. Linux filesystem mental model

```text
Disk → Block device → Partition/LVM → Filesystem → Mount point → Directory tree → Files
```

Example:

```text
/dev/nvme0n1p1 → ext4 → /
/dev/sdb1      → xfs  → /data
```

A filesystem must be mounted before its files are accessible through the normal directory tree.

## 2. Filesystem vs mount point

A **filesystem** organizes files and metadata. A **mount point** is the directory where that filesystem is attached.

```text
/dev/sdb1 → /data
```

Inspect mounts:

```bash
mount
findmnt
findmnt /data
```

## 3. df — filesystem capacity

Use `df` to answer: **How full is the filesystem?**

```bash
df
df -h
df -i
df -ih
```

`df -h` reports block-space usage. `df -i` reports inode usage.

## 4. du — directory/file usage

Use `du` to answer: **Which directories/files consume the space?**

```bash
du -h
du -sh /var/log
du -xh --max-depth=1 /var | sort -hr
du -xh --max-depth=1 / | sort -hr
```

Drill down from a large directory until the source is identified.

## 5. Find large files

```bash
find /var -type f -printf '%s %p\n' 2>/dev/null | sort -nr | head
find /var -type f -exec du -h {} + 2>/dev/null | sort -hr | head
```

Avoid unnecessarily broad scans on large production systems.

## 6. lsblk and findmnt

`lsblk` shows block devices and partitions:

```bash
lsblk
lsblk -f
```

`lsblk -f` is useful for filesystem type, UUID, labels and mount points.

`findmnt` gives structured mount information:

```bash
findmnt
findmnt /
findmnt /data
```

Mental model:

```text
lsblk     → what storage exists?
df        → how full is each mounted filesystem?
du        → what consumes the space?
findmnt   → where is it mounted?
```

## 7. Mounting and /etc/fstab

General syntax:

```bash
mount <device> <mount-point>
```

Example:

```bash
mount /dev/sdb1 /data
df -h /data
findmnt /data
```

`/etc/fstab` stores persistent mount configuration:

```text
UUID=xxxx-xxxx  /data  ext4  defaults  0  2
```

Validate `fstab` changes carefully. A bad entry can cause mount, boot or service problems.

## 8. Inodes

An **inode** stores filesystem metadata such as file type, permissions, owner/group, size, timestamps and references to data blocks.

Every file consumes an inode. Therefore a filesystem can have free disk space but still fail to create files if it runs out of inodes.

## 9. Disk-space vs inode exhaustion

```text
Block-space exhaustion:
Disk space → 100%
Inodes     → available

Inode exhaustion:
Disk space → may still have free space
Inodes     → 100%
```

Always check both:

```bash
df -h
df -i
```

Common inode causes include millions of tiny files, temporary/session files, caches, log fragments, container data and application cleanup bugs.

## 10. Finding inode-heavy directories

Start with:

```bash
df -i
```

Then inspect file counts. `-xdev` prevents `find` from crossing into other mounted filesystems:

```bash
find /var -xdev -type f -printf '%h\n' 2>/dev/null | sort | uniq -c | sort -nr | head
```

Troubleshooting model:

```text
df -i → locate filesystem → find inode-heavy directories → identify creator → clean safely → fix retention
```

## 11. Production disk troubleshooting

If `/` is above 90%:

```bash
df -h
df -i
lsblk -f
findmnt
du -xh --max-depth=1 / | sort -hr
```

Drill into large paths:

```bash
du -xh --max-depth=1 /var | sort -hr
du -xh --max-depth=1 /var/log | sort -hr
find /var/log -type f -exec du -h {} + 2>/dev/null | sort -hr | head
```

Identify the owning service, retention policy and whether files are actively used before cleaning.

## 12. Deleted-but-open files

Sometimes `df` shows a full filesystem while `du` does not explain the usage. A process may still have a deleted file open.

```bash
lsof +L1
lsof | grep deleted
```

The space remains allocated until the process closes the file descriptor. Restarting the relevant process can release it, but assess service impact first.

## 13. Common mistakes

- Using only `df -h` and expecting it to identify the offending file.
- Checking disk space but not inodes.
- Running an unrestricted `du /` on a large production host.
- Deleting files before identifying their owner and purpose.
- Using `rm` as the first response to a disk alert.

Safe mental model:

```text
Measure → Identify → Understand → Clean safely → Prevent recurrence
```

## 14. Command cheat sheet

| Goal | Command |
|---|---|
| Filesystem usage | `df -h` |
| Inode usage | `df -i` |
| Block devices | `lsblk` |
| Filesystem details | `lsblk -f` |
| Current mounts | `mount` |
| Structured mounts | `findmnt` |
| Directory size | `du -sh <dir>` |
| Top-level directory sizes | `du -xh --max-depth=1 <dir>` |
| Sort largest first | `du -xh --max-depth=1 <dir> \| sort -hr` |
| Persistent mounts | `cat /etc/fstab` |
| Deleted-but-open files | `lsof +L1` |

## DevOps relevance

Disk troubleshooting is common on Linux application servers and EC2. The key distinction is **filesystem capacity vs inode capacity**, followed by a safe drill-down to the owning workload and prevention through retention, rotation, cleanup and monitoring.