# DAY 2 - Linux Filesystems, Mounts, Inodes & Disk Troubleshooting

> **Goal:** Understand where Linux stores files and how to troubleshoot a server when disk space is full.

## 1. Start with the basic picture

Think of Linux storage like this:

```text
Physical disk
   ↓
Partition / LVM
   ↓
Filesystem
   ↓
Mounted somewhere
   ↓
Directories and files
```

A **filesystem** is the structure Linux uses to store files. A **mount point** is the directory where that filesystem becomes accessible.

Example:

```text
/dev/xvdf1  →  ext4 filesystem  →  /data
```

## 2. What is `/`?

Linux has one main directory tree starting at `/`, called the **root filesystem**.

Common directories:

```text
/       root of the filesystem tree
/etc    configuration
/var    changing data and logs
/home   user home directories
/tmp    temporary files
/dev    devices
/proc   process/kernel information
```

## 3. Check disk space: `df`

`df` means **disk free**. It tells you how much space is available in each mounted filesystem.

```bash
df -h
```

`-h` = human-readable sizes such as GB and MB.

Example:

```text
Filesystem   Size  Used Avail Use% Mounted on
/dev/xvda1   20G   19G  1G   95% /
```

If `/` is 95% full, applications may start failing.

## 4. Check inode usage

Files need more than disk blocks. Linux also uses **inodes** to store file metadata.

Check inode usage:

```bash
df -i
```

Human-readable:

```bash
df -ih
```

A filesystem can have free GB but still be unable to create files if all inodes are used.

## 5. Find what is using space: `du`

`du` means **disk usage**.

```bash
du -sh /var
```

- `-s` = summary only
- `-h` = human-readable

Check directories:

```bash
du -xh --max-depth=1 /var | sort -hr
```

Meanings:

- `-x` = stay on the current filesystem
- `-h` = human-readable
- `--max-depth=1` = show one directory level
- `sort -hr` = sort human-readable sizes from largest to smallest

## 6. Find large files

```bash
find /var -xdev -type f -size +1G -ls
```

Meaning:

- `find` = search files/directories
- `/var` = search location
- `-xdev` = do not cross into another filesystem
- `-type f` = regular files
- `-size +1G` = larger than 1 GB
- `-ls` = show detailed information

## 7. Understand `lsblk`

`lsblk` shows block devices such as disks and partitions.

```bash
lsblk
lsblk -f
```

`-f` shows filesystem information.

Example mental model:

```text
xvda       disk
└─xvda1    partition
   └─/     mounted filesystem
```

## 8. What does mounting mean?

A filesystem on a disk is not automatically visible through a directory. Linux must **mount** it.

Example:

```bash
sudo mount /dev/xvdf1 /data
```

Now files stored on that filesystem appear under `/data`.

Check mounts:

```bash
findmnt
mount
```

## 9. `/etc/fstab`

`/etc/fstab` contains filesystem mount configuration that can be used during boot.

Example concept:

```text
UUID=xxxx  /data  ext4  defaults  0  2
```

Why UUID is useful: device names can change, while a filesystem UUID identifies the filesystem more reliably.

**Production warning:** a bad `fstab` entry can cause boot problems. Validate changes carefully.

## 10. Disk full vs inode full

### Situation A: disk blocks are full

```bash
df -h
```

shows very high `Use%`.

Investigate:

```bash
du -xh --max-depth=1 /
find / -xdev -type f -size +1G -ls
```

### Situation B: inodes are full

```bash
df -i
```

shows very high inode usage.

This often happens when an application creates millions of tiny files.

Find directories containing huge numbers of files:

```bash
find /var -xdev -type f | cut -d/ -f1-4 | sort | uniq -c | sort -nr | head
```

## 11. Deleted file still using disk

Sometimes an application deletes a large log file while keeping it open.

The filename disappears, but the process still holds the file descriptor.

Check:

```bash
sudo lsof +L1
```

If a huge deleted file is held open, restart the responsible application carefully or use the application's log-rotation mechanism.

## 12. Production disk troubleshooting

When an application says `No space left on device`:

```text
1. df -h
      ↓
2. df -i
      ↓
3. Find the full filesystem
      ↓
4. du to find large directories
      ↓
5. find to locate large files
      ↓
6. Check deleted-but-open files
      ↓
7. Check logs / application behavior
      ↓
8. Clean safely or expand storage
```

Never blindly run `rm -rf` on a production server. Identify the owner and purpose of the data first.

## 13. DevOps / AWS connection

On EC2, disk problems are common when logs or application data grow unexpectedly.

Typical investigation:

```bash
df -h
lsblk -f
du -xh --max-depth=1 /var
journalctl --disk-usage
```

If the underlying EBS volume needs more capacity, the Linux filesystem may also need to be extended after the AWS-side volume change.

### Beginner takeaway

Remember this difference:

- `df` → **How full is the filesystem?**
- `du` → **Which files/directories are using the space?**
- `lsblk` → **What disks/partitions exist?**
- `mount`/`findmnt` → **Where are filesystems attached?**
- `df -i` → **Are we out of inodes instead of disk blocks?**

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.