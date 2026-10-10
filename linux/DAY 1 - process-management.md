# Linux Process Management

## 1. What is a process?

A **process** is a running instance of a program managed by the operating system. Linux assigns each process a **PID (Process ID)** and tracks its parent with a **PPID (Parent Process ID)**.

A process commonly has:

- PID
- PPID
- Owner/user
- CPU and memory usage
- Process state
- Start time
- Command

Example process hierarchy:

```text
systemd (PID 1)
├── sshd
│   └── bash
│       └── command
└── nginx
    ├── worker
    └── worker
```

## 2. ps — process snapshot

`ps` means **process status** and provides a point-in-time snapshot.

```bash
ps
ps aux
ps -ef
ps -p <PID> -f
```

Common `ps aux` interpretation:

- `a` — processes for all users
- `u` — user-oriented output
- `x` — include processes without a controlling terminal

Find a process:

```bash
ps aux | grep nginx
ps -ef | grep nginx
pgrep nginx
```

Show hierarchy:

```bash
pstree
```

## 3. PID and PPID

**PID** identifies the current process. **PPID** identifies its parent.

```text
PID   PPID   CMD
1200     1   nginx
1201  1200   nginx worker
```

## 4. Common process states

| State | Meaning |
|---|---|
| R | Running or runnable |
| S | Sleeping / waiting |
| D | Uninterruptible sleep, often waiting on I/O |
| T | Stopped |
| Z | Zombie |

A **zombie** has finished execution but remains in the process table until its parent collects its exit status.

## 5. top and htop

`top` continuously monitors processes and system resources.

```bash
top
htop
```

Useful `top` keys:

- `P` — sort by CPU
- `M` — sort by memory
- `k` — send a signal
- `q` — quit

Load average normally shows 1-, 5-, and 15-minute values.

Mental model:

```text
ps    → snapshot
top   → continuous monitoring
htop  → interactive monitoring
```

## 6. Linux signals

A signal is a notification sent to a process.

```bash
kill -l
```

Important signals:

| Signal | Number | Purpose |
|---|---:|---|
| SIGHUP | 1 | Hangup; some daemons use it for reload |
| SIGINT | 2 | Interrupt; commonly Ctrl+C |
| SIGQUIT | 3 | Quit |
| SIGKILL | 9 | Forceful termination |
| SIGTERM | 15 | Graceful termination request |
| SIGSTOP | 19 | Stop/suspend |
| SIGCONT | 18 | Resume |

## 7. Graceful vs forceful termination

Default `kill` sends SIGTERM:

```bash
kill <PID>
kill -TERM <PID>
```

The application can handle SIGTERM and clean up resources.

SIGKILL cannot be handled:

```bash
kill -9 <PID>
kill -KILL <PID>
```

Use SIGKILL only when graceful termination is unsuccessful and forceful termination is justified.

## 8. SIGINT, SIGSTOP, SIGCONT and SIGHUP

`Ctrl+C` normally sends SIGINT:

```bash
kill -2 <PID>
```

Stop and resume a process:

```bash
kill -STOP <PID>
kill -CONT <PID>
```

SIGHUP historically means hangup. Many daemons use it for configuration reload, but behavior depends on the application:

```bash
kill -HUP <PID>
```

## 9. Production troubleshooting flow

For a Java process consuming high CPU:

```text
top / htop
   ↓
identify PID
   ↓
ps -p <PID> -f
   ↓
check logs and application behavior
   ↓
determine whether usage is expected
   ↓
remediate safely
```

If termination is required:

```bash
kill <PID>
ps -p <PID>
# only when appropriate
kill -9 <PID>
```

Do not immediately kill a high-CPU production process without understanding the workload and impact.

## 10. Command cheat sheet

```bash
ps aux
ps -ef
ps -p <PID> -f
pgrep <process>
pstree
top
htop
kill <PID>
kill -15 <PID>
kill -9 <PID>
kill -STOP <PID>
kill -CONT <PID>
kill -l
```

## DevOps relevance

Process management is foundational for troubleshooting high CPU, hung applications, failed services, crashes, zombie processes and production incidents. It connects directly to `systemctl`, `journalctl`, resource troubleshooting and application recovery.