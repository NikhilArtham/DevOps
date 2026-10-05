# DAY 5 - journalctl and Log-Driven Troubleshooting

## 1. What is journalctl?

`journalctl` is the command-line tool used to query logs collected by **systemd-journald**.

In DevOps troubleshooting, logs help answer:

- What failed?
- When did it fail?
- Which process/service failed?
- What happened immediately before the failure?
- Is the problem repeating?
- Did a configuration, permission, network or dependency change cause it?

Think:

```text
Incident
   ↓
Identify affected service/process
   ↓
Check status
   ↓
Query logs
   ↓
Find first useful error
   ↓
Correlate timestamp + process + request
   ↓
Form hypothesis
   ↓
Verify with commands/metrics/config
   ↓
Fix root cause
   ↓
Verify from logs
```

---

## 2. journald and journalctl

`systemd-journald` is the service that collects journal entries.

`journalctl` is the tool used to read/query them.

```bash
systemctl status systemd-journald
journalctl
```

The journal can contain messages from:

- system services
- applications writing to stdout/stderr
- kernel messages
- boot processes
- authentication-related services
- scheduled/systemd jobs

---

## 3. Command anatomy

Do not memorize flags blindly. Understand what they mean.

| Command/flag | Meaning |
|---|---|
| `journalctl` | Query the systemd journal |
| `-u` | **unit** — filter logs for a systemd unit/service |
| `-f` | **follow** — continuously show new log entries |
| `-n` | show the specified number of recent lines |
| `-b` | filter by **boot** |
| `-p` | filter by **priority** |
| `--since` | show entries after the specified time |
| `--until` | show entries before the specified time |
| `-k` | show kernel messages |
| `-r` | reverse order; newest entries first |
| `--no-pager` | print directly instead of opening a pager |
| `-o` | choose output format |
| `-g` | filter by a regular-expression pattern in the message |
| `-t` | filter by syslog identifier/tag |
| `--disk-usage` | show journal disk usage |
| `--vacuum-time` | remove old journal entries based on age |
| `--vacuum-size` | remove old entries until journal usage is below a size |

Example:

```bash
journalctl -u nginx -n 100 --no-pager
```

Read it as:

> Show the last 100 journal entries for the `nginx` systemd unit and print them directly.

---

## 4. Most useful journalctl commands

### View all available journal entries

```bash
journalctl
```

### View the newest entries first

```bash
journalctl -r
```

### View the latest 100 entries

```bash
journalctl -n 100
```

### Follow logs in real time

```bash
journalctl -f
```

This is useful while reproducing an issue.

### Follow a specific service

```bash
journalctl -u nginx -f
```

### Show logs for a service

```bash
journalctl -u nginx
```

### Show only recent service logs

```bash
journalctl -u nginx -n 100 --no-pager
```

### Show logs from the current boot

```bash
journalctl -b
```

### Show logs from the previous boot

```bash
journalctl -b -1
```

### Show kernel messages

```bash
journalctl -k
```

---

## 5. Time-based troubleshooting

When investigating an incident, narrow the time window instead of reading thousands of lines.

```bash
journalctl --since "30 minutes ago"
```

Specific range:

```bash
journalctl --since "2026-10-05 08:00:00" --until "2026-10-05 08:30:00"
```

For a service:

```bash
journalctl -u myapp --since "10 minutes ago" --no-pager
```

### Why time filtering matters

Suppose an application stopped at 08:15.

Instead of searching the entire journal:

```bash
journalctl -u myapp
```

focus on the incident window:

```bash
journalctl -u myapp --since "08:10" --until "08:20" --no-pager
```

This makes the sequence of events much easier to understand.

---

## 6. Log priorities

The journal supports priority levels from most severe to least severe:

```text
0  emerg      system unusable
1  alert      immediate action required
2  crit       critical condition
3  err        error
4  warning    warning
5  notice     normal but significant
6  info       informational
7  debug      debugging information
```

Show errors for a service:

```bash
journalctl -u myapp -p err
```

A useful production command:

```bash
journalctl -u myapp -p warning --since "1 hour ago" --no-pager
```

This helps reduce noise when an application produces many informational messages.

---

## 7. Output formats

Default output is human-readable.

```bash
journalctl -u myapp
```

Short output:

```bash
journalctl -u myapp -o short
```

JSON output can be useful for automation/integration:

```bash
journalctl -u myapp -o json
```

For troubleshooting, the default format is usually easiest to read.

---

## 8. Filtering logs

Search messages using a pattern:

```bash
journalctl -g "failed"
```

For a service:

```bash
journalctl -u myapp -g "timeout"
```

Filter by identifier/tag:

```bash
journalctl -t sshd
```

Combine filters:

```bash
journalctl -u myapp -p err --since "1 hour ago" --no-pager
```

The key troubleshooting idea is **progressive narrowing**:

```text
All logs
  ↓
Time window
  ↓
Service
  ↓
Priority
  ↓
Error pattern
```

---

## 9. Log-driven troubleshooting method

Do not just search for the word `error`.

Use this process:

### Step 1 — Define the symptom

Example:

> API service is returning HTTP 500 errors.

### Step 2 — Identify the affected component

```bash
systemctl status myapp
```

### Step 3 — Establish the incident time

Find when the problem started.

### Step 4 — Narrow the logs

```bash
journalctl -u myapp --since "15 minutes ago" --no-pager
```

### Step 5 — Find the first meaningful failure

Look for:

- connection refused
- timeout
- permission denied
- address already in use
- authentication failure
- configuration error
- out of memory
- dependency failure

### Step 6 — Correlate

Compare the application log with:

- system logs
- kernel logs
- network state
- disk usage
- memory usage
- deployment/change timeline
- AWS service/API errors

### Step 7 — Verify the hypothesis

If the log says:

```text
Connection refused: 10.0.2.15:5432
```

do not immediately restart the application.

Check whether the database is listening:

```bash
ss -lntp | grep 5432
```

Then check connectivity:

```bash
nc -vz 10.0.2.15 5432
```

The log gives the clue; another command should validate the cause.

---

## 10. Realistic failure scenario — application cannot connect to database

### Situation

A production Java application suddenly starts returning errors.

You check:

```bash
systemctl status myapp
```

The service is still active, so simply restarting it is not the correct first step.

### Step 1 — Check recent application logs

```bash
journalctl -u myapp --since "15 minutes ago" -n 200 --no-pager
```

You find:

```text
ERROR Database connection failed
Connection refused: 10.0.2.15:5432
```

### Step 2 — Identify whether the database is listening

```bash
ss -lntp | grep 5432
```

No listener is found.

### Step 3 — Check the database service

```bash
systemctl status postgresql
journalctl -u postgresql --since "20 minutes ago" --no-pager
```

Suppose the database log shows:

```text
FATAL: could not open file ... Permission denied
```

### Step 4 — Follow the evidence

Now the problem is not the Java application's systemd service.

The chain is:

```text
Application
   ↓
Database connection refused
   ↓
PostgreSQL not listening
   ↓
PostgreSQL failed to start
   ↓
Journal shows Permission denied
   ↓
Investigate database file ownership/permissions
```

### Step 5 — Verify the permission problem

```bash
ls -l /path/to/database/file
namei -l /path/to/database/file
```

Fix the actual ownership/permission issue according to the database installation's requirements.

### Step 6 — Restart the dependency and verify

```bash
sudo systemctl restart postgresql
systemctl status postgresql
journalctl -u postgresql -n 50 --no-pager
```

Then verify the application:

```bash
systemctl status myapp
journalctl -u myapp -n 50 --no-pager
```

### Production lesson

The application log identified the symptom. The database log identified the root cause.

**Always follow the dependency chain instead of restarting the first service that reports an error.**

---

## 11. Another common scenario — service starts but keeps failing

```bash
systemctl status myapp
```

shows repeated restarts.

Check:

```bash
journalctl -u myapp -n 200 --no-pager
```

If you see:

```text
Address already in use
```

verify the port:

```bash
ss -lntp | grep :8080
```

Then identify which process owns it.

The troubleshooting path is:

```text
systemd restart loop
       ↓
journal error
       ↓
Port already in use
       ↓
ss -lntp
       ↓
Identify conflicting process
       ↓
Determine why it owns the port
       ↓
Fix root cause
```

---

## 12. Journal persistence and disk usage

Check journal disk usage:

```bash
journalctl --disk-usage
```

On some systems, journal storage may be persistent under `/var/log/journal`; on others, it may be stored temporarily under `/run/log/journal` depending on configuration.

Inspect journald configuration:

```bash
cat /etc/systemd/journald.conf
```

After configuration changes, restart journald when appropriate:

```bash
sudo systemctl restart systemd-journald
```

### Important production point

Do not manually delete journal files as your first cleanup action. Understand retention and use journald's supported vacuum controls.

Example:

```bash
sudo journalctl --vacuum-time=7d
```

This removes archived journal entries older than seven days.

Size-based cleanup:

```bash
sudo journalctl --vacuum-size=1G
```

This asks journald to remove archived entries until usage is below the specified size.

---

## 13. Useful log locations besides journald

Not every application uses journald exclusively.

Common locations include:

```text
/var/log/
/var/log/syslog
/var/log/messages
/var/log/auth.log
/var/log/secure
```

Exact files depend on the Linux distribution and logging configuration.

Application logs may also be stored under paths such as:

```text
/var/log/myapp/
/opt/myapp/logs/
```

Always determine where the application actually writes logs instead of assuming every log is in the same place.

---

## 14. AWS connection

On EC2, use Linux logs together with AWS observability and control-plane information.

Example troubleshooting chain:

```text
EC2 application failure
       ↓
systemctl status
       ↓
journalctl -u myapp
       ↓
OS/network/config clue
       ↓
AWS dependency?
       ↓
CloudWatch Logs / Metrics
       ↓
CloudTrail / AWS API error if relevant
       ↓
IAM / security group / route / service issue
```

Examples:

- `AccessDenied` → investigate IAM identity/resource policies and CloudTrail when relevant.
- Connection timeout → investigate routes, security groups, NACLs, DNS and target health.
- Out of memory → investigate `journalctl -k`, memory metrics and the process.
- Disk full → correlate logs with `df`, `du` and journal disk usage.

**Do not confuse Linux logs with AWS CloudWatch Logs.** They can provide different parts of the same incident.

---

## 15. Practical lab

On a disposable Linux VM:

### Lab 1 — Follow a service

```bash
journalctl -u systemd-journald -f
```

In another terminal, perform a harmless service operation and observe new entries.

### Lab 2 — Time filtering

```bash
journalctl --since "10 minutes ago"
journalctl --since "10 minutes ago" --until "5 minutes ago"
```

### Lab 3 — Boot investigation

```bash
journalctl -b
journalctl -b -1
```

### Lab 4 — Create and investigate a failing service

Use your DAY 4 `systemd-lab.service` and intentionally point `ExecStart` at a nonexistent executable.

Then run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart systemd-lab
systemctl status systemd-lab
journalctl -u systemd-lab -n 50 --no-pager
```

Identify the exact failure from the journal, fix `ExecStart`, reload systemd and verify the service.

---

# Interview Questions

<details>
<summary>1. What is journalctl?</summary>

`journalctl` is the command-line tool used to query logs collected by systemd-journald.
</details>

<details>
<summary>2. What is systemd-journald?</summary>

It is the systemd logging service that collects and stores journal entries from the system, services and other sources.
</details>

<details>
<summary>3. How do you view logs for a specific service?</summary>

Use `journalctl -u service-name`.
</details>

<details>
<summary>4. How do you follow logs in real time?</summary>

Use `journalctl -f` or `journalctl -u service-name -f` for one service.
</details>

<details>
<summary>5. What does -u mean in journalctl?</summary>

`-u` means unit. It filters journal entries belonging to the specified systemd unit.
</details>

<details>
<summary>6. How do you view logs from the previous boot?</summary>

Use `journalctl -b -1`.
</details>

<details>
<summary>7. How do you filter logs by time?</summary>

Use `--since` and `--until`, for example `journalctl --since '30 minutes ago'`.
</details>

<details>
<summary>8. How do you show only error-level logs?</summary>

Use `journalctl -p err`. You can combine it with `-u` and a time range.
</details>

<details>
<summary>9. What does -f mean?</summary>

`-f` means follow. It keeps the command running and displays new log entries as they arrive.
</details>

<details>
<summary>10. How would you troubleshoot a service that is active but the application is failing?</summary>

Check the service's journal, narrow the incident time, identify the first meaningful error, then validate the suspected dependency/configuration/network/permission problem with independent commands.
</details>

<details>
<summary>11. Why should you not search only for the word error?</summary>

The root cause may be logged as a warning, timeout, refusal, permission problem, dependency failure or another message. Troubleshooting requires understanding the sequence of events, not just matching one word.
</details>

<details>
<summary>12. How do you check journal disk usage?</summary>

Use `journalctl --disk-usage`.
</details>

<details>
<summary>13. How do you safely reduce old journal storage?</summary>

Use supported vacuum controls such as `journalctl --vacuum-time=7d` or `journalctl --vacuum-size=1G`, according to the retention requirement.
</details>

<details>
<summary>14. What is log-driven troubleshooting?</summary>

It is using logs as evidence to reconstruct what happened, form a hypothesis and then verify that hypothesis with configuration, process, network, filesystem, metric or cloud checks.
</details>

<details>
<summary>15. An application says connection refused. What do you do?</summary>

Identify the target host/port, check whether the target process is listening with `ss`, verify connectivity, then inspect the target service's logs and infrastructure/network controls.
</details>

<details>
<summary>16. What is the difference between journalctl and CloudWatch Logs?</summary>

`journalctl` queries the local Linux systemd journal. CloudWatch Logs is an AWS-managed logging service. They may contain different evidence from the same incident.
</details>

<details>
<summary>17. How do you investigate a failure after reboot?</summary>

Compare `journalctl -b` with `journalctl -b -1`, identify the service that failed, inspect its logs and correlate the failure with startup ordering, dependencies, configuration and resource availability.
</details>

<details>
<summary>18. A service is restarting repeatedly. What should you inspect?</summary>

Check `systemctl status`, `journalctl -u`, exit codes, restart policy, configuration, permissions, ports, dependencies and the underlying application error.
</details>

<details>
<summary>19. Why is the first meaningful error important?</summary>

Later errors can be cascading symptoms. The first failure in the sequence is often closer to the root cause.
</details>

<details>
<summary>20. What is your production log-troubleshooting approach?</summary>

Define the symptom and incident time, identify the affected component, narrow logs by service/time/priority, find the first meaningful failure, correlate dependencies and infrastructure, verify the hypothesis, fix the root cause and confirm recovery through logs and metrics.
</details>

---

## Production mental model

```text
Symptom
  ↓
When did it start?
  ↓
Which component?
  ↓
Which logs?
  ↓
What happened first?
  ↓
What dependency failed?
  ↓
What evidence confirms the hypothesis?
  ↓
Fix root cause
  ↓
Verify recovery
```

**Logs are evidence, not the final answer.** Use the log to form a hypothesis, then verify it with the appropriate Linux, application or AWS command.