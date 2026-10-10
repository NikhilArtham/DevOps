# Linux

Linux notes for DevOps interview preparation and real-world troubleshooting.

## 🎯 How these notes are written

These notes are intentionally written for a **complete beginner**.

Every topic should follow this learning order:

```text
What is it?
   ↓
Why does Linux need it?
   ↓
Simple real-world example
   ↓
Basic command
   ↓
Command/flag meaning
   ↓
Practical DevOps example
   ↓
Troubleshooting
   ↓
Interview-level understanding
```

You should not need to already know Linux to start reading DAY 1.

The goal is not just to memorize commands. The goal is to understand **what Linux is doing and why the command is useful**.

## 🧠 Central Study Resources

### Interview Questions
All Linux interview questions are maintained in one central file with clickable answers.

[Open Linux Interview Questions](./INTERVIEW%20QUESTIONS.md)

### Practical Labs
All Linux practical exercises are maintained in one central workbook.

[Open Linux Practical Labs](./LABS.md)

**DAY files contain learning notes only.** Do not add separate Practical Lab or Interview Questions sections to DAY files.

## 📚 Study Progress

### DAY 1 — Process Management
What is a process, PID/PPID, `ps`, `top`, signals, `kill` and zombie processes.

[Open DAY 1](./DAY%201%20-%20process-management.md)

### DAY 2 — Filesystems & Disk Troubleshooting
Filesystem basics, mounts, `df`, `du`, `lsblk`, inodes and disk-full troubleshooting.

[Open DAY 2](./DAY%202%20-%20filesystems-disks.md)

### DAY 3 — Permissions, Ownership, ACLs & sudo
Users, groups, `chmod`, `chown`, ACLs, sudo and `Permission denied` troubleshooting.

[Open DAY 3](./DAY%203%20-%20permissions-ownership-acl-sudo.md)

### DAY 4 — systemd Services
What services are, `systemctl`, start/stop/restart/enable, dependencies and service failures.

[Open DAY 4](./DAY%204%20-%20systemd-services.md)

### DAY 5 — Logs & journalctl
What logs are, `journald`, `journalctl`, filtering, timelines and log-driven troubleshooting.

[Open DAY 5](./DAY%205%20-%20journalctl-log-troubleshooting.md)

### DAY 6 — Linux Text Processing
`grep`, `sed`, `awk`, `sort`, `xargs` and production log analysis.

[Open DAY 6](./DAY%206%20-%20grep-sed-awk-sort-xargs.md)

### DAY 7 — Bash Scripting Basics
Variables, arguments, conditions, loops, functions, exit codes and basic automation.

[Open DAY 7](./DAY%207%20-%20bash-scripting.md)

### DAY 8 — Bash Error Handling & Safe Automation
`set -euo pipefail`, `trap`, cleanup, `flock`, validation and safer deployment scripts.

[Open DAY 8](./DAY%208%20-%20bash-error-handling-traps.md)

### DAY 9 — Cron & systemd Timers
Scheduled automation, cron syntax, timer/service architecture and scheduler troubleshooting.

[Open DAY 9](./DAY%209%20-%20cron-systemd-timers-scheduled-automation.md)

### DAY 10 — SSH
SSH keys, `ssh-agent`, bastions, ProxyJump, tunneling and systematic SSH troubleshooting.

[Open DAY 10](./DAY%2010%20-%20ssh-keys-agent-tunneling-failures.md)

## 🔄 Rules for every future topic

### 1. Understand the topic before choosing the folder

Do **not** automatically put every new topic under Linux.

Before creating a file, determine which technology the topic belongs to.

Examples:

```text
Linux process management       → linux/
AWS IAM                        → aws/
Terraform modules              → terraform/
Docker images                  → docker/
Kubernetes deployments         → kubernetes/
Jenkins pipelines              → jenkins/
Git branching                  → git/
Prometheus/Grafana             → monitoring/
Ansible playbooks              → ansible/
```

If a topic overlaps multiple technologies, place the main concept in the folder where it primarily belongs and explain the cross-technology connection inside the note.

### 2. DAY numbering is per technology

DAY numbers restart for each technology.

For example:

```text
linux/DAY 11 - ...
aws/DAY 1 - ...
docker/DAY 1 - ...
kubernetes/DAY 1 - ...
```

Do not continue Linux numbering for an AWS topic just because Linux was studied immediately before it.

### 3. One new topic = one new DAY file

Create only the new topic file.

Do not create separate files such as:

```text
DAY 11 - LABS.md
DAY 11 - INTERVIEW QUESTIONS.md
```

Instead:

- topic notes → DAY file
- practical exercises → central `LABS.md`
- interview questions → central `INTERVIEW QUESTIONS.md`

### 4. Beginner-first writing standard

Every new DAY file should:

- assume the reader may know nothing about the technology
- explain terminology before using it
- explain why the technology/command exists
- show simple examples before advanced examples
- explain command flags individually
- explain what the output means
- include production/DevOps context
- include troubleshooting thinking
- clearly separate beginner concepts from advanced details
- end with a short **Beginner takeaway**

### 5. Preserve the learning path

Do not blindly add advanced commands just because they are useful.

Build the concept first, then introduce the command.

For example:

```text
What is a process?
      ↓
What is a PID?
      ↓
How do I find a PID?
      ↓
How do I inspect it?
      ↓
How do I troubleshoot it?
```

### 6. Central files must stay synchronized

Whenever a new topic is created:

1. Create the correct technology/DAY file.
2. Add its practical exercises to that technology's `LABS.md`.
3. Add its interview questions to that technology's `INTERVIEW QUESTIONS.md`.
4. Update that technology's README.
5. Never create duplicate topic/lab/interview files.

This structure applies to Linux and should be followed for other technology folders as they are built out.