# Linux

Linux notes for DevOps interview preparation and real-world troubleshooting.

## 🎯 Central Study Resources

### 🧠 Interview Question Bank
All Linux interview questions from every DAY are maintained in one centralized, revision-friendly file. Each question has a clickable answer.

[Open Linux Interview Questions](./INTERVIEW%20QUESTIONS.md)

### 🧪 Practical Lab Workbook
All hands-on labs and practical exercises from every DAY are maintained in one centralized workbook so all practice is available in one place.

[Open Linux Practical Labs](./LABS.md)

---

## Study Progress

### DAY 1 — Process Management
- Process basics, PID / PPID, ps, pgrep, pstree
- Process states, top / htop, signals, kill
- Zombie processes and high-CPU troubleshooting

[Open DAY 1 notes](./DAY%201%20-%20process-management.md)

### DAY 2 — Filesystems & Disk Troubleshooting
- Filesystems, mounts and mount points
- df, du, lsblk, mount, findmnt
- Inode exhaustion, large-file investigation
- Deleted-but-open files and production disk troubleshooting

[Open DAY 2 notes](./DAY%202%20-%20filesystems-disks.md)

### DAY 3 — Permissions, Ownership, ACLs & sudo Troubleshooting
- chmod, chown, chgrp, namei
- ACLs with getfacl / setfacl
- setuid, setgid, sticky bit
- sudo, sudoers, visudo and Permission denied troubleshooting

[Open DAY 3 notes](./DAY%203%20-%20permissions-ownership-acl-sudo.md)

### DAY 4 — systemd Services, Dependencies & Restart Behavior
- systemctl lifecycle and active vs enabled
- Requires, Wants, After and Before
- Restart policies, daemon-reload
- Troubleshooting services that start and immediately exit

[Open DAY 4 notes](./DAY%204%20-%20systemd-services.md)

### DAY 5 — journalctl & Log-Driven Troubleshooting
- journalctl filters, service/boot/time/priority filtering
- Progressive log narrowing and root-cause troubleshooting
- Journal persistence/disk usage and AWS/EC2 connection

[Open DAY 5 notes](./DAY%205%20-%20journalctl-log-troubleshooting.md)

### DAY 6 — grep, sed, awk, sort & xargs
- Production log filtering and transformation
- Field extraction, sorting and argument construction
- AWS/EC2 and Kubernetes log-debugging connection

[Open DAY 6 notes](./DAY%206%20-%20grep-sed-awk-sort-xargs.md)

### DAY 7 — Bash Scripting
- Variables, arguments, conditions, loops and functions
- Exit codes, command chaining, set -euo pipefail
- AWS CLI/Kubernetes scripting and deployment failure troubleshooting

[Open DAY 7 notes](./DAY%207%20-%20bash-scripting.md)

### DAY 8 — Bash Error Handling, Traps & Safe Automation
- trap, cleanup, signal handling and explicit error handling
- mktemp, flock, safe command construction
- AWS CLI/Kubernetes automation safety

[Open DAY 8 notes](./DAY%208%20-%20bash-error-handling-traps.md)

### DAY 9 — Cron, systemd Timers & Scheduled Automation
- Cron/crontab and scheduler environment
- systemd `.service` + `.timer`
- OnCalendar, OnBootSec, OnUnitActiveSec and Persistent
- Timer vs service troubleshooting
- EC2 scheduled automation and AWS integration

[Open DAY 9 notes](./DAY%209%20-%20cron-systemd-timers-scheduled-automation.md)

### DAY 10 — SSH: Keys, Agent, Tunneling & Common Failures
- SSH public/private keys and permissions
- ssh-agent, ssh-add and agent forwarding
- ProxyJump / bastion architecture
- Local, remote and dynamic SSH tunneling
- sshd_config, client config and verbose debugging
- EC2 Security Groups, key pairs and IAM vs SSH
- Timeout, publickey, agent, host-key and tunnel failures
- Production SSH troubleshooting mental model

[Open DAY 10 notes](./DAY%2010%20-%20ssh-keys-agent-tunneling-failures.md)

---

## 🧪 Practical Practice

[Open the centralized Linux Lab Workbook](./LABS.md)

Use the workbook to repeat every lab from DAY 1 onward.

## 📌 Daily Update Rule

For every new Linux study DAY:

1. Create/update the DAY-wise topic file with complete notes, practical lab and interview questions.
2. Update this README with the new DAY and link.
3. Add the new interview questions to the centralized **[Linux Interview Questions](./INTERVIEW%20QUESTIONS.md)** file.
4. Add the new practical exercises to the centralized **[Linux Practical Labs](./LABS.md)** workbook.
5. Keep the DAY file's own interview and practical sections as well.
6. Do not create duplicate topic files when an existing canonical file has already been established.
