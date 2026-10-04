# Linux

Linux notes for DevOps interview preparation and real-world troubleshooting.

## Study Progress

### DAY 1 — Process Management
- Process basics
- PID / PPID
- ps, pgrep, pstree
- Process states
- top / htop
- Signals and kill
- Zombie processes
- High CPU troubleshooting
- Interview questions + practical troubleshooting

[Open DAY 1 notes](./DAY%201%20-%20process-management.md)

### DAY 2 — Filesystems & Disk Troubleshooting
- Filesystems, mounts and mount points
- df, du, lsblk, mount, findmnt
- Inode exhaustion
- Large-file investigation
- Deleted-but-open files
- Production disk troubleshooting
- Interview questions + practical labs

[Open DAY 2 notes](./DAY%202%20-%20filesystems-disks.md)

### DAY 3 — Permissions, Ownership, ACLs & sudo Troubleshooting
- Linux permissions and numeric modes
- chmod, chown, chgrp
- Parent-directory traversal with namei
- ACLs with getfacl / setfacl
- ACL mask and default ACLs
- setuid, setgid and sticky bit
- sudo, sudoers, visudo, sudo -l
- Production Permission denied troubleshooting
- Linux vs AWS authorization layers
- Interview questions + practical labs

[Open DAY 3 notes](./DAY%203%20-%20permissions-ownership-acl-sudo.md)

### DAY 4 — systemd Services, Dependencies & Restart Behavior
- systemctl status/start/stop/restart/enable
- Active vs enabled
- systemd unit structure
- Requires, Wants, After and Before
- Restart policies and restart limits
- journalctl troubleshooting
- daemon-reload
- Troubleshooting services that start and immediately exit
- EC2/systemd production connection
- Interview questions + practical lab

[Open DAY 4 notes](./DAY%204%20-%20systemd-services.md)

## Upcoming Topics

- Users and groups
- Networking commands
- Memory troubleshooting
- Logs
- Shell basics
