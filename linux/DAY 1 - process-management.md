# DAY 1 - Linux Process Management

> **Goal:** Understand what a Linux process is, how to find it, and how to troubleshoot a process that is using too much CPU or memory.

## 1. First: what is Linux?

Linux is an operating system. Just like Windows or macOS, it manages the computer's CPU, memory, disks, network and running programs.

As a DevOps engineer, you will often manage Linux servers such as AWS EC2 machines.

## 2. What is a process?

A **process is a running program**.

For example, when you run:

```bash
sleep 100
```

Linux creates a process for `sleep` and gives it a unique number called a **PID**.

Think of it like this:

```text
Program on disk
      ↓
You start it
      ↓
Linux creates a process
      ↓
CPU + memory are used
      ↓
Process finishes
```

## 3. PID and PPID

- **PID** = Process ID. Unique number for a running process.
- **PPID** = Parent Process ID. The PID of the process that started it.

Example:

```bash
ps -ef
```

You may see:

```text
UID   PID   PPID   CMD
root  1000  1      /usr/sbin/sshd
user  2450  1000   sshd: user
```

The second process was started by the first one.

## 4. Find running processes

### `ps`

Shows a snapshot of processes.

```bash
ps
ps aux
ps -ef
```

Useful meanings:

- `ps` = process status
- `a` = processes from all users with a terminal
- `u` = user-oriented output
- `x` = include processes without a terminal
- `-e` = every process
- `-f` = full-format information

### `pgrep`

Find a PID by process name:

```bash
pgrep nginx
```

### `pstree`

Shows parent/child relationships:

```bash
pstree -p
```

`-p` shows PIDs.

## 5. Process states

A process is not always actively using the CPU.

Common states:

| State | Simple meaning |
|---|---|
| `R` | Running or ready to run |
| `S` | Sleeping/waiting normally |
| `D` | Waiting for I/O, usually uninterruptible |
| `T` | Stopped |
| `Z` | Zombie; process finished but parent has not collected its status |

## 6. Watch processes in real time

```bash
top
```

`top` continuously shows CPU, memory and process activity.

Useful keys inside `top`:

- `P` = sort by CPU
- `M` = sort by memory
- `k` = send a signal to a process
- `q` = quit

If available, `htop` provides a more user-friendly interface:

```bash
htop
```

## 7. What is a signal?

A signal is a message sent to a process asking it to do something.

Common signals:

| Signal | Meaning | Typical use |
|---|---|---|
| `SIGTERM` / 15 | Please terminate gracefully | Normal shutdown |
| `SIGKILL` / 9 | Stop immediately | Last resort |
| `SIGINT` / 2 | Interrupt | Ctrl+C |
| `SIGHUP` / 1 | Hangup/reload depending on program | Reload configuration |
| `SIGSTOP` / 19 | Stop process | Pause |
| `SIGCONT` / 18 | Continue stopped process | Resume |

## 8. Stop a process correctly

First try a graceful stop:

```bash
kill <PID>
```

`kill` does not necessarily mean "kill immediately". By default it sends `SIGTERM`.

If the process refuses to stop and you understand the impact:

```bash
kill -9 <PID>
```

`-9` sends `SIGKILL`.

**Production rule:** prefer graceful termination. Use `kill -9` only when necessary because the process cannot clean up.

## 9. Real-world example: high CPU

Suppose an EC2 server is slow.

Start here:

```bash
top
```

Find the process using the CPU. Then:

```bash
ps -p <PID> -o pid,ppid,user,%cpu,%mem,stat,cmd
```

Check:

1. Which process is using CPU?
2. Who owns it?
3. What command started it?
4. Is high CPU expected?
5. Is it a service?
6. Did a recent deployment cause it?

Do not immediately kill a process just because CPU is high. First understand what it is doing.

## 10. Zombie processes

A zombie is a process that has finished, but its parent has not yet collected its exit status.

Find them:

```bash
ps -eo pid,ppid,stat,cmd | grep ' Z'
```

The important point is that killing the zombie itself usually does not solve the problem because it is already finished. Investigate the parent process.

## 11. Simple troubleshooting flow

```text
Server is slow
    ↓
top / htop
    ↓
Find suspicious PID
    ↓
ps -p PID -o ...
    ↓
Understand process + parent
    ↓
Check logs/service
    ↓
Decide: wait / restart / terminate / escalate
```

## 12. Commands to remember

```bash
ps aux
ps -ef
pgrep nginx
pstree -p
top
ps -p <PID> -o pid,ppid,user,%cpu,%mem,stat,cmd
kill <PID>
kill -9 <PID>
```

### Beginner takeaway

If you remember only three things today:

1. **Process = running program.**
2. **PID = unique number of that process.**
3. **Use `ps`/`top` to investigate before sending signals.**

> Labs and interview questions for this topic are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.