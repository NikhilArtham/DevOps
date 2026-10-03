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

## 17. Interview Questions

> Click the arrow next to each question to reveal the answer.

<details>
<summary>1. What is a process?</summary>

A **process** is a running instance of a program managed by the operating system.

For example, when you run `nginx`, Linux creates a process and assigns it a PID.

</details>

<details>
<summary>2. What is a PID?</summary>

PID stands for **Process ID**. It is a unique identifier assigned to a running process.

You can inspect a process using:

```bash
ps -p <PID> -f
```

</details>

<details>
<summary>3. What is a PPID?</summary>

PPID stands for **Parent Process ID**. It identifies the process that created or launched the current process.

Example:

```text
PID   PPID
1200    1
1201 1200
```

Here, process 1201 is a child of process 1200.

</details>

<details>
<summary>4. What is the difference between ps and top?</summary>

`ps` provides a **point-in-time snapshot** of processes.

`top` continuously refreshes and provides **real-time process and system resource monitoring**.

Mental model:

```text
ps  → snapshot
top → continuous monitoring
```

</details>

<details>
<summary>5. What is the difference between top and htop?</summary>

Both provide real-time process monitoring.

`htop` generally provides a more interactive and visual interface for sorting, navigating, and managing processes.

</details>

<details>
<summary>6. What does ps aux show?</summary>

`ps aux` shows processes from all users in a user-oriented format, including processes that do not have a controlling terminal.

Common interpretation:

- `a` — processes for all users
- `u` — user-oriented output
- `x` — include processes without a terminal

</details>

<details>
<summary>7. What does ps -ef show?</summary>

`ps -ef` provides a full-format process listing and is especially useful for viewing **PID, PPID, user, and the full command**.

</details>

<details>
<summary>8. How do you find a specific process?</summary>

Common approaches include:

```bash
ps aux | grep nginx
ps -ef | grep nginx
pgrep nginx
```

`pgrep` is useful when you want the PID directly.

</details>

<details>
<summary>9. What are common Linux process states?</summary>

Common states include:

| State | Meaning |
|---|---|
| R | Running or runnable |
| S | Sleeping / waiting |
| D | Uninterruptible sleep, often waiting on I/O |
| T | Stopped |
| Z | Zombie |

</details>

<details>
<summary>10. What is a zombie process?</summary>

A zombie process has **finished execution**, but its parent has not yet collected its exit status.

It is not an actively running process consuming CPU; it remains as a process-table entry until the parent handles its exit status.

</details>

<details>
<summary>11. What is a signal in Linux?</summary>

A signal is a notification sent to a process to request or indicate an event or action.

Examples include:

```text
SIGTERM
SIGKILL
SIGINT
SIGSTOP
SIGCONT
SIGHUP
```

List available signals with:

```bash
kill -l
```

</details>

<details>
<summary>12. What does kill PID do?</summary>

By default:

```bash
kill <PID>
```

sends **SIGTERM (15)**.

It requests graceful termination and gives the application an opportunity to clean up.

The `kill` command actually means **send a signal**; it does not inherently mean force-kill.

</details>

<details>
<summary>13. What does kill -9 do?</summary>

```bash
kill -9 <PID>
```

sends **SIGKILL (9)**.

SIGKILL cannot be caught, blocked, or handled by the target process, so it forcefully terminates the process.

Prefer SIGTERM first when appropriate:

```bash
kill <PID>
```

Use SIGKILL when the process does not terminate cleanly and a forceful termination is justified.

</details>

<details>
<summary>14. What is the difference between SIGTERM and SIGKILL?</summary>

**SIGTERM (15):**

- Requests graceful termination.
- The application can handle it.
- The application can perform cleanup.

**SIGKILL (9):**

- Forcefully terminates the process.
- Cannot be caught or handled.
- Does not allow application-level graceful cleanup.

</details>

<details>
<summary>15. What is SIGINT?</summary>

SIGINT is the **interrupt** signal, commonly generated when you press:

```text
Ctrl+C
```

It is signal number 2.

You can also send it with:

```bash
kill -2 <PID>
```

</details>

<details>
<summary>16. What are SIGSTOP and SIGCONT?</summary>

SIGSTOP pauses/stops a process:

```bash
kill -STOP <PID>
```

SIGCONT resumes a stopped process:

```bash
kill -CONT <PID>
```

The process is paused rather than terminated.

</details>

<details>
<summary>17. What is SIGHUP used for?</summary>

SIGHUP historically means **hangup**.

Many daemonized applications use SIGHUP as a request to reload configuration without fully restarting, but the exact behavior depends on the application.

Example:

```bash
kill -HUP <PID>
```

Always verify the specific application's signal behavior before relying on it.

</details>

<details>
<summary>18. What is the difference between kill -15 and kill -9?</summary>

```bash
kill -15 <PID>
```

sends SIGTERM and requests graceful termination.

```bash
kill -9 <PID>
```

sends SIGKILL and forcefully terminates the process.

In production, normally attempt graceful termination before using SIGKILL when appropriate.

</details>

<details>
<summary>19. How would you troubleshoot a process consuming high CPU?</summary>

A basic investigation flow:

```text
top / htop
    ↓
Identify high-CPU process
    ↓
Get PID
    ↓
ps -p <PID> -f
    ↓
Understand the process
    ↓
Check logs / application behavior
    ↓
Determine whether the CPU usage is expected
    ↓
Take the appropriate remediation
```

Do not immediately kill the process just because CPU usage is high; first determine why it is consuming CPU and whether it is serving a critical workload.

</details>

<details>
<summary>20. Why is process management important for a DevOps engineer?</summary>

DevOps engineers frequently troubleshoot:

- High CPU
- High memory usage
- Hung processes
- Failed services
- Zombie processes
- Application crashes
- Processes that need graceful restarts
- Production incidents

Understanding processes and signals provides the foundation for troubleshooting services with tools such as `systemctl`, `journalctl`, `top`, and `ps`.

</details>

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
