# Linux Process Management

## 1. What is a process?

A **process** is a running instance of a program.

Example:

```bash
nginx
```

When Linux starts the program, it creates a process and assigns it a unique **PID (Process ID)**.

A process commonly has:

- PID — Process ID
- PPID — Parent Process ID
- User — process owner
- CPU usage
- Memory usage
- Process state
- Start time
- Command

Linux processes form parent-child relationships.

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

`ps` means **process status**. It shows a snapshot of processes.

### Basic

```bash
ps
```

### Most useful form

```bash
ps aux
```

Common interpretation of `aux`:

- `a` — processes for all users
- `u` — user-oriented output
- `x` — include processes without a terminal

### Another common form

```bash
ps -ef
```

Useful for seeing PID, PPID, and the full command.

### Inspect one process

```bash
ps -p <PID> -f
```

### Find a process

```bash
ps aux | grep nginx
ps -ef | grep nginx
pgrep nginx
```

### Show process hierarchy

```bash
pstree
```

## 3. PID and PPID

**PID** identifies the process.

**PPID** identifies its parent process.

Example:

```text
PID   PPID   CMD
1200     1   nginx
1201  1200   nginx worker
```

Here PID 1201 is a child of PID 1200.

## 4. Common process states

The `STAT` column in `ps` or `top` can show process state.

| State | Meaning |
|---|---|
| R | Running or runnable |
| S | Sleeping / waiting |
| D | Uninterruptible sleep, often waiting on I/O |
| T | Stopped |
| Z | Zombie |

### Zombie process

A zombie process has finished execution, but the parent has not yet collected its exit status.

## 5. top — real-time process monitoring

`top` continuously refreshes process and system resource information.

```bash
top
```

Useful information includes:

- CPU utilization
- Memory utilization
- Load average
- Running processes
- Process IDs
- Per-process CPU and memory usage

### Common interactive keys

Inside `top`:

- `P` — sort by CPU usage
- `M` — sort by memory usage
- `k` — send a signal to a process
- `q` — quit

Load average commonly displays 1-minute, 5-minute, and 15-minute values.

## 6. htop — interactive process monitoring

`htop` is an interactive alternative to `top`.

```bash
htop
```

It provides a more visual interface for:

- Finding processes
- Sorting processes
- Viewing CPU and memory usage
- Navigating process hierarchy
- Sending signals

## 7. ps vs top vs htop

| Tool | Main use |
|---|---|
| ps | Point-in-time process snapshot |
| top | Continuous process/system monitoring |
| htop | Interactive real-time monitoring |

Mental model:

```text
ps    → What processes exist right now?
top   → What is happening right now?
htop  → Give me an interactive view of it.
```

## 8. Signals

A **signal** is a notification sent to a process.

List signals:

```bash
kill -l
```

Important signals:

| Signal | Number | Typical purpose |
|---|---:|---|
| SIGHUP | 1 | Hangup; some daemons use it for reload |
| SIGINT | 2 | Interrupt; commonly Ctrl+C |
| SIGQUIT | 3 | Quit |
| SIGKILL | 9 | Forceful termination |
| SIGTERM | 15 | Graceful termination request |
| SIGSTOP | 19 | Stop/suspend process |
| SIGCONT | 18 | Continue/resume stopped process |

## 9. SIGTERM — graceful termination

```bash
kill <PID>
```

By default, `kill <PID>` sends **SIGTERM (15)**.

The application can handle SIGTERM and perform cleanup before exiting.

Typical sequence:

```text
SIGTERM
  ↓
application cleanup
  ↓
release resources
  ↓
exit
```

## 10. SIGKILL — forceful termination

```bash
kill -9 <PID>
```

This sends **SIGKILL**.

The process cannot catch or handle SIGKILL.

Use it when a process does not terminate cleanly and a forceful stop is appropriate.

Prefer:

```bash
kill <PID>
```

before:

```bash
kill -9 <PID>
```

## 11. SIGINT — Ctrl+C

Pressing:

```text
Ctrl+C
```

normally sends **SIGINT (2)** to the foreground process.

Equivalent signal form:

```bash
kill -2 <PID>
```

## 12. SIGSTOP and SIGCONT

Stop a process:

```bash
kill -STOP <PID>
```

Resume it:

```bash
kill -CONT <PID>
```

The process is paused, not terminated.

## 13. SIGHUP

Historically means **hangup**.

Many daemonized applications use SIGHUP as a request to reload configuration, but the exact behavior depends on the application.

Example:

```bash
kill -HUP <PID>
```

## 14. Important point about kill

The `kill` command means **send a signal**. It does not inherently mean force-kill.

Examples:

```bash
kill -TERM <PID>
kill -KILL <PID>
kill -STOP <PID>
kill -CONT <PID>
kill -HUP <PID>
```

## 15. Practical DevOps troubleshooting flow

Suppose a Java application is consuming high CPU.

### Step 1 — Observe

```bash
top
```

### Step 2 — Identify the PID

```bash
ps aux | grep java
```

### Step 3 — Inspect the process

```bash
ps -p <PID> -f
```

### Step 4 — Graceful stop when required

```bash
kill <PID>
```

### Step 5 — Verify

```bash
ps -p <PID>
```

### Step 6 — Forceful stop only when appropriate

```bash
kill -9 <PID>
```

### Step 7 — Verify again

```bash
ps -p <PID>
```

## 16. Commands to remember

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

## 17. Interview questions

### What is a process?
A running instance of a program managed by the operating system.

### What is a PID?
A unique identifier assigned to a process.

### ps vs top?
`ps` gives a snapshot; `top` provides continuously refreshed monitoring.

### top vs htop?
Both monitor processes in real time; `htop` is generally more interactive and visual.

### What does kill -9 do?
It sends SIGKILL and forcefully terminates the target process.

### What does kill PID do?
By default it sends SIGTERM (15), requesting graceful termination.

### SIGTERM vs SIGKILL?
SIGTERM allows the application an opportunity to handle termination and clean up. SIGKILL cannot be handled and is immediate/forceful.

### What is a zombie process?
A terminated process whose parent has not yet collected its exit status.

### What is process state?
The current state of a process, such as running, sleeping, stopped, or zombie.

## DevOps relevance

Process management is foundational for production troubleshooting. The next related areas to learn are:

- systemd and systemctl
- service management
- logs and journalctl
- CPU troubleshooting
- memory pressure and OOM killer
- zombie/orphan processes
- process priorities (nice/renice)
- background jobs and job control
