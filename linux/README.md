# Linux

Linux notes for DevOps interview preparation and real-world troubleshooting.

## 🎯 Central Interview Question Bank

All Linux interview questions from every DAY are also maintained in one centralized, revision-friendly file. Each question has a clickable answer.

[🧠 Open Linux Interview Questions](./INTERVIEW%20QUESTIONS.md)

---

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
- Command/flag meanings
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

### DAY 5 — journalctl & Log-Driven Troubleshooting
- systemd-journald and journalctl
- journalctl command/flag meanings
- Service, boot, time and priority filtering
- Real-time log monitoring
- Progressive log narrowing
- Log-driven root-cause troubleshooting
- Realistic application/database failure scenario
- Journal persistence and disk usage
- AWS/EC2 log troubleshooting connection
- Practical failure lab
- Interview questions + production mental model

[Open DAY 5 notes](./DAY%205%20-%20journalctl-log-troubleshooting.md)

### DAY 6 — grep, sed, awk, sort & xargs for Production Debugging
- grep search and filtering
- sed text transformation
- awk field extraction and calculations
- sort numeric/human-readable ordering
- xargs command argument construction
- Command/flag meanings and command anatomy
- Production-safe pipelines
- Log analysis with combined shell commands
- Realistic API 500/latency troubleshooting scenario
- AWS/EC2 and Kubernetes log-debugging connection
- Practical debugging lab
- Interview questions + production mental model

[Open DAY 6 notes](./DAY%206%20-%20grep-sed-awk-sort-xargs.md)

### DAY 7 — Bash Scripting: Variables, Loops, Functions & Exit Codes
- Bash scripts and shebangs
- Variables, quoting and command substitution
- Environment variables and script arguments
- if conditions and file/string tests
- for/while loops, break and continue
- Functions, local variables, return vs exit
- Exit codes and CI/CD failure handling
- &&, || and ; command chaining
- set -euo pipefail
- Bash syntax checking and tracing
- Realistic deployment-script failure scenario
- AWS CLI and Kubernetes scripting connection
- Practical health-check lab
- Interview questions + production mental model

[Open DAY 7 notes](./DAY%207%20-%20bash-scripting.md)

### DAY 8 — Bash Error Handling, Traps & Safe Automation
- Explicit error handling and accurate exit codes
- set -euo pipefail limitations
- trap, EXIT, ERR and signal handling
- Cleanup and graceful shutdown
- Temporary files/directories with mktemp
- flock and duplicate-job prevention
- Input validation and safe command construction
- Realistic failed deployment + cleanup scenario
- AWS CLI and Kubernetes automation safety
- Practical trap/error-handling lab
- Interview questions + production mental model

[Open DAY 8 notes](./DAY%208%20-%20bash-error-handling-traps.md)

## Upcoming Topics

- Users and groups
- Networking commands
- Memory troubleshooting
- Shell basics

---

## 📌 Daily Update Rule

For every new Linux study DAY:

1. Create/update the DAY-wise topic file with complete notes and its interview questions.
2. Update this README with the new DAY and link.
3. Add the new interview questions to the centralized **[Linux Interview Questions](./INTERVIEW%20QUESTIONS.md)** file.
4. Keep the DAY file's own interview section as well, so the topic remains self-contained.
5. Do not create duplicate topic files when an existing canonical file has already been established.
