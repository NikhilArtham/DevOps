# DAY 9 - Cron, systemd Timers & Scheduled Automation

> Linux / AWS — scheduled automation for DevOps

## 🎯 Focus for this topic

By the end of this day, you should be able to:

- Explain cron and crontab scheduling.
- Read and write cron expressions confidently.
- Understand user crontabs and system cron directories.
- Troubleshoot cron jobs that do not run.
- Understand why cron environments differ from interactive shells.
- Use `systemctl` and `journalctl` to troubleshoot scheduled systemd units.
- Create a `.service` + `.timer` pair.
- Compare cron and systemd timers and choose the appropriate mechanism.
- Connect Linux scheduling to AWS automation.

---

# 1. What is scheduled automation?

Scheduled automation means running a command or job automatically at a defined time or interval.

Common DevOps examples:

- Backups
- Log cleanup
- Report generation
- Certificate checks
- Health checks
- Temporary-file cleanup
- AWS resource start/stop automation
- Database maintenance
- Synchronizing files
- Monitoring scripts

Mental model:

```text
Schedule
   ↓
Trigger
   ↓
Command / script / service
   ↓
Exit status
   ↓
Logs / monitoring / alert
```

A scheduler is not enough by itself. Production automation must also be observable and able to report failure.

---

# 2. Cron

**cron** is a traditional Unix/Linux scheduling mechanism for recurring jobs.

The cron daemon commonly runs in the background and evaluates scheduled entries.

Check the daemon on a systemd-based Linux host:

```bash
systemctl status cron
```

Some distributions use `crond` instead:

```bash
systemctl status crond
```

The exact service name depends on the distribution.

---

# 3. crontab

A user's scheduled jobs are normally managed with:

```bash
crontab -e
```

List the current user's cron jobs:

```bash
crontab -l
```

Remove the current user's crontab:

```bash
crontab -r
```

> Be careful with `crontab -r`: it removes the user's entire crontab.

### Command anatomy

| Command | Meaning |
|---|---|
| `crontab` | manage a user's cron table |
| `-e` | edit the current user's crontab |
| `-l` | list the current user's crontab |
| `-r` | remove the current user's crontab |
| `-u USER` | operate on another user's crontab when permitted |

---

# 4. Cron expression anatomy

A standard user crontab entry has five scheduling fields followed by the command:

```text
* * * * * command
│ │ │ │ │
│ │ │ │ └── Day of week (0-7, commonly Sunday = 0 or 7)
│ │ │ └──── Month (1-12)
│ │ └────── Day of month (1-31)
│ └──────── Hour (0-23)
└────────── Minute (0-59)
```

Example:

```cron
30 9 * * 1-5 /opt/scripts/health-check.sh
```

Meaning:

> Run `health-check.sh` at 09:30 Monday through Friday.

---

# 5. Cron operators

### `*` — every value

```cron
* * * * * command
```

Runs every minute.

### `,` — list

```cron
0 9,18 * * * command
```

Runs at 09:00 and 18:00.

### `-` — range

```cron
0 9 * * 1-5 command
```

Runs Monday through Friday at 09:00.

### `/` — step

```cron
*/15 * * * * command
```

Runs every 15 minutes.

---

# 6. Common cron examples

### Every minute

```cron
* * * * * /opt/scripts/job.sh
```

### Every 5 minutes

```cron
*/5 * * * * /opt/scripts/job.sh
```

### Every day at 02:00

```cron
0 2 * * * /opt/scripts/backup.sh
```

### Every Sunday at 03:30

```cron
30 3 * * 0 /opt/scripts/weekly.sh
```

### Weekdays at 09:00

```cron
0 9 * * 1-5 /opt/scripts/report.sh
```

### First day of every month at 01:00

```cron
0 1 1 * * /opt/scripts/monthly.sh
```

---

# 7. Cron environment — important production issue

One of the most common cron problems is:

> The script works manually but fails under cron.

Why?

The cron environment may differ from your interactive shell.

Important differences can include:

- Different `PATH`
- Different working directory
- Missing shell startup files
- Missing environment variables
- Different permissions/user
- No interactive terminal

Bad assumption:

```cron
*/10 * * * * aws s3 sync ./backup s3://my-bucket/backup
```

Better:

```cron
*/10 * * * * /usr/local/bin/aws s3 sync /opt/backup s3://my-bucket/backup >> /var/log/backup.log 2>&1
```

Use absolute paths for important commands and files.

Find command paths:

```bash
command -v aws
command -v bash
command -v python3
```

---

# 8. Cron output and logging

Cron jobs should not silently fail.

Redirect output:

```cron
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

Breakdown:

- `>>` → append stdout to the log
- `2>&1` → send stderr to the same destination as stdout

For systemd-based systems, cron activity may also appear in the system journal depending on the distribution/configuration:

```bash
journalctl -u cron
```

or:

```bash
journalctl -u crond
```

---

# 9. Cron troubleshooting flow

Suppose a backup cron job did not run.

### Step 1 — Is cron running?

```bash
systemctl status cron
```

or:

```bash
systemctl status crond
```

### Step 2 — Is the job installed?

```bash
crontab -l
```

### Step 3 — Is the schedule correct?

Read every field carefully.

### Step 4 — Can the script execute?

```bash
ls -l /opt/scripts/backup.sh
```

### Step 5 — Does the script work manually?

```bash
/opt/scripts/backup.sh
```

### Step 6 — Does it depend on PATH/environment variables?

```bash
command -v aws
```

### Step 7 — Check logs

```bash
journalctl -u cron --since "1 hour ago"
```

and/or inspect the redirected job log.

### Step 8 — Check permissions and user

The cron job runs as a specific user. Verify that user can access the script, files, directories and AWS credentials/configuration it needs.

---

# 10. Cron system directories

Many Linux distributions provide system-level cron locations such as:

```text
/etc/crontab
/etc/cron.d/
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

`/etc/crontab` and files under `/etc/cron.d/` can have an additional **user field** compared with a normal user crontab.

Example:

```cron
0 2 * * * root /opt/scripts/backup.sh
```

The exact cron implementation and available directories depend on the Linux distribution.

---

# 11. Cron vs systemd timers

Modern Linux systems also provide **systemd timers**.

High-level comparison:

| Cron | systemd timer |
|---|---|
| Simple recurring schedules | Richer scheduling model |
| Very common/portable | Integrated with systemd |
| Minimal configuration | Native service lifecycle |
| Basic logging | Excellent journal integration |
| Traditional choice | Strong choice for systemd-based servers |
| Cron expression | Calendar/monotonic timer expressions |

A useful interview answer is not “systemd timers replace cron everywhere.”

Instead:

> Cron is simple and widely supported; systemd timers provide stronger integration with systemd services, dependencies, logging and lifecycle management.

---

# 12. systemd timer architecture

A timer normally activates a `.service` unit.

Example:

```text
backup.timer
     |
     | triggers
     v
backup.service
     |
     v
backup script
```

Important concept:

> The timer schedules the service. The service performs the work.

Do not put the actual backup command in the timer unit.

---

# 13. Create a systemd service

Example:

```ini
# /etc/systemd/system/devops-backup.service
[Unit]
Description=DevOps Backup Job

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

`Type=oneshot` is appropriate for a job that runs and exits rather than remaining as a long-running daemon.

---

# 14. Create a systemd timer

```ini
# /etc/systemd/system/devops-backup.timer
[Unit]
Description=Run DevOps Backup Daily

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Meaning:

- `OnCalendar=` → calendar-based schedule
- `Persistent=true` → if the scheduled time was missed while the machine was powered off, systemd can trigger the service when the timer becomes active again
- `WantedBy=timers.target` → allows the timer to be enabled into the timers target

---

# 15. Activate the systemd timer

After creating or changing unit files:

```bash
sudo systemctl daemon-reload
```

Start the timer now:

```bash
sudo systemctl start devops-backup.timer
```

Enable it for future boots:

```bash
sudo systemctl enable devops-backup.timer
```

Often you will use both:

```bash
sudo systemctl enable --now devops-backup.timer
```

Meaning:

- `enable` → configure automatic activation
- `--now` → start it immediately as well

---

# 16. Inspect timers

List timers:

```bash
systemctl list-timers
```

Show all timers, including inactive ones:

```bash
systemctl list-timers --all
```

Inspect one timer:

```bash
systemctl status devops-backup.timer
```

Useful information includes:

- Last trigger
- Next trigger
- Timer state
- Related service

---

# 17. Inspect the service triggered by the timer

```bash
systemctl status devops-backup.service
```

Then inspect logs:

```bash
journalctl -u devops-backup.service
```

Recent logs:

```bash
journalctl -u devops-backup.service -n 50 --no-pager
```

This connects today's topic directly to the earlier systemd/journalctl troubleshooting topics.

---

# 18. systemd timer scheduling types

### `OnCalendar=`

Calendar/time based scheduling.

Example:

```ini
OnCalendar=Mon..Fri 09:00
```

### `OnBootSec=`

Run relative to system boot.

```ini
OnBootSec=10min
```

### `OnStartupSec=`

Run relative to activation of the systemd manager.

### `OnUnitActiveSec=`

Run relative to when the associated unit was last activated.

Example:

```ini
OnUnitActiveSec=1h
```

### `OnUnitInactiveSec=`

Run relative to when the associated unit became inactive.

These monotonic timers are useful when you want intervals rather than a specific wall-clock time.

---

# 19. Persistent timers

Consider a timer scheduled for 02:00.

The server is powered off at 02:00 and boots at 08:00.

With:

```ini
Persistent=true
```

systemd can trigger the associated service after the missed calendar event when the timer becomes active.

This is a major operational difference from a simple “run only while the scheduler is alive” model.

---

# 20. Troubleshooting a systemd timer

### Problem: timer is not triggering

Check:

```bash
systemctl status myjob.timer
systemctl list-timers --all
```

Check the timer definition:

```bash
systemctl cat myjob.timer
```

Check the service:

```bash
systemctl status myjob.service
```

Check logs:

```bash
journalctl -u myjob.service
```

If you changed the unit file:

```bash
sudo systemctl daemon-reload
```

Then restart the timer:

```bash
sudo systemctl restart myjob.timer
```

### Problem: timer triggers but job fails

The timer may be healthy while the service is failing.

Use:

```bash
systemctl status myjob.service
journalctl -u myjob.service -n 100 --no-pager
```

This distinction is critical:

```text
Timer problem
    ≠
Service/job problem
```

---

# 21. Service starts and immediately exits — today's troubleshooting connection

For a scheduled systemd service, an immediate exit is not automatically an error.

A `Type=oneshot` job is expected to exit after completing its work.

The real question is:

> Did it exit successfully or fail?

Check:

```bash
systemctl status myjob.service
journalctl -u myjob.service -n 100 --no-pager
```

Look for:

- Non-zero exit status
- Permission denied
- Missing executable
- Wrong `ExecStart`
- Missing environment variables
- Wrong working directory
- AWS credentials/configuration problems
- Network/DNS failures
- Application errors

This is why the earlier focus on `systemctl` + `journalctl` matters for scheduled automation.

---

# 22. AWS connection

Linux scheduling is commonly used on EC2 instances.

Examples:

```text
EC2
 ↓
cron / systemd timer
 ↓
Bash script
 ↓
AWS CLI
 ↓
S3 / EC2 / CloudWatch / other AWS APIs
```

Example scheduled S3 sync:

```bash
#!/usr/bin/env bash
set -euo pipefail

aws s3 sync /opt/backups "s3://my-backup-bucket/"
```

Production requirements:

- Use an EC2 instance role where appropriate instead of hardcoding access keys.
- Use absolute paths.
- Log stdout/stderr.
- Check exit codes.
- Prevent overlapping executions when required.
- Monitor failures.

### AWS-native alternative

For AWS workloads, not every scheduled task needs to run inside an EC2 host.

An AWS-native architecture can use an AWS scheduling service such as **Amazon EventBridge Scheduler** to invoke a supported target, depending on the workload.

Mental model:

```text
Linux-local scheduling
cron / systemd timer
        ↓
EC2 host

AWS-managed scheduling
EventBridge Scheduler
        ↓
AWS target
```

Choose based on where the workload should run and who should own the scheduling/lifecycle responsibility.

---

# 23. Production best practices

### 1. Use absolute paths

Prefer:

```cron
0 2 * * * /opt/scripts/backup.sh
```

over relying on the interactive shell's current directory.

### 2. Make jobs idempotent where possible

Running the same job twice should not corrupt data or produce an unexpected state.

### 3. Prevent overlapping jobs

For scripts that must not run concurrently:

```bash
flock -n /var/lock/backup.lock /opt/scripts/backup.sh
```

### 4. Log failures

```cron
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

For systemd timers, prefer the journal:

```bash
journalctl -u devops-backup.service
```

### 5. Use a dedicated service account

Avoid running scheduled application jobs as root unless root privileges are genuinely required.

### 6. Monitor the schedule itself

A successful scheduler trigger does not necessarily mean the job succeeded.

Monitor:

```text
Schedule
   ↓
Trigger
   ↓
Job execution
   ↓
Exit status
   ↓
Expected result
```

---

# 24. Practical troubleshooting scenario

### Situation

A nightly EC2 backup stopped running.

You discover:

```text
Timer → active
Service → failed
```

Investigate:

```bash
systemctl status backup.timer
systemctl status backup.service
journalctl -u backup.service -n 100 --no-pager
```

Suppose the journal shows:

```text
aws: command not found
```

The script works manually because your interactive shell has AWS CLI in `PATH`.

Cron/systemd execution does not have the same environment.

Fix the script to use the absolute AWS CLI path:

```bash
command -v aws
```

Then update the script accordingly and rerun the service manually:

```bash
sudo systemctl start backup.service
```

Verify:

```bash
systemctl status backup.service
journalctl -u backup.service -n 50 --no-pager
```

### Root cause

The scheduler was healthy. The scheduled **job environment** was wrong.

This distinction is an important production troubleshooting skill.

---

# 25. Command cheat sheet

```bash
# Cron
crontab -l
crontab -e
crontab -r
systemctl status cron
systemctl status crond
journalctl -u cron

# systemd timers
systemctl list-timers
systemctl list-timers --all
systemctl status myjob.timer
systemctl status myjob.service
systemctl cat myjob.timer
systemctl cat myjob.service
systemctl daemon-reload
systemctl enable --now myjob.timer
systemctl restart myjob.timer
journalctl -u myjob.service -n 50 --no-pager
```

---

# Interview Questions

<details><summary>1. What is cron?</summary>
Cron is a traditional Linux scheduling mechanism used to run recurring commands or scripts at specified times or intervals.
</details>

<details><summary>2. What is crontab?</summary>
A crontab is a user's table of scheduled cron jobs. `crontab -e` edits it and `crontab -l` lists it.
</details>

<details><summary>3. What are the five fields in a normal user cron expression?</summary>
Minute, hour, day of month, month and day of week, followed by the command.
</details>

<details><summary>4. What does */5 * * * * mean?</summary>
It runs the command every five minutes.
</details>

<details><summary>5. What is the difference between /etc/crontab and a user crontab?</summary>
System cron formats such as `/etc/crontab` can include an explicit user field, while a normal user crontab does not need that field because the user is already known.
</details>

<details><summary>6. Why does a cron job work manually but fail from cron?</summary>
Common causes are different PATH/environment variables, working directory, permissions, user identity, shell assumptions and missing configuration.
</details>

<details><summary>7. How do you troubleshoot a cron job that did not run?</summary>
Check the cron daemon, `crontab -l`, schedule syntax, script permissions, absolute paths, redirected logs/journal logs, environment variables and the executing user's access.
</details>

<details><summary>8. How do you redirect both stdout and stderr from cron?</summary>
Use `>> /path/job.log 2>&1`. `>>` appends stdout and `2>&1` sends stderr to the same destination.
</details>

<details><summary>9. What is a systemd timer?</summary>
A systemd timer is a unit that schedules or triggers another systemd unit, normally a `.service` unit.
</details>

<details><summary>10. Why does a systemd timer normally use a separate service unit?</summary>
The timer handles scheduling while the service defines the actual work, giving systemd a clean lifecycle, status and logging model.
</details>

<details><summary>11. What does OnCalendar= do?</summary>
It schedules a timer using calendar/time expressions, such as a specific time, day or recurring calendar schedule.
</details>

<details><summary>12. What does Persistent=true do?</summary>
For calendar timers, it allows systemd to account for a missed scheduled event when the timer was inactive, such as while the machine was powered off.
</details>

<details><summary>13. What is OnBootSec=?</summary>
It schedules activation relative to system boot, such as `OnBootSec=10min`.
</details>

<details><summary>14. What is OnUnitActiveSec=?</summary>
It schedules a timer relative to when the associated unit was last activated, making it useful for interval-style scheduling.
</details>

<details><summary>15. How do you list systemd timers?</summary>
Use `systemctl list-timers` or `systemctl list-timers --all`.
</details>

<details><summary>16. A timer is active but the job fails. Where do you look?</summary>
Inspect the triggered service with `systemctl status job.service` and `journalctl -u job.service`. The timer can be healthy while the service fails.
</details>

<details><summary>17. What should you do after modifying a systemd unit file?</summary
>
Run `systemctl daemon-reload`, then restart/start the affected unit as appropriate.
</details>

<details><summary>18. Why can a Type=oneshot service appear to exit immediately?</summary>
A oneshot service is designed to perform a task and exit. Check the exit status and journal to determine whether it completed successfully or failed.
</details>

<details><summary>19. Cron vs systemd timers — which is better?</summary>
Neither is universally better. Cron is simple and widely supported; systemd timers integrate tightly with systemd services, dependencies, status and journal logging.
</details>

<details><summary>20. How would you design scheduled AWS automation on EC2?</summary>
Use cron or a systemd timer to invoke a well-tested script, use an instance role rather than hardcoded credentials where appropriate, use absolute paths, log output, check exit codes, prevent unsafe overlap and monitor failures. For AWS-native workloads, consider a managed scheduler such as EventBridge Scheduler instead of putting the schedule on an EC2 host.
</details>

## Production mental model

```text
Schedule
   ↓
Trigger
   ↓
Service / script
   ↓
Environment + permissions
   ↓
Command execution
   ↓
Exit status
   ↓
Logs / monitoring
   ↓
Alert / remediation
```

**Key lesson:** never troubleshoot only the schedule. A scheduled automation chain has multiple failure points: scheduler → trigger → execution environment → permissions → command → exit status → expected result.
