# DAY 9 - Cron, systemd Timers & Scheduled Automation

> Linux / AWS — scheduled automation for DevOps

## 1. Scheduled automation mental model

```text
Schedule → trigger → command/script/service → exit status → logs/monitoring/alert
```

Common examples: backups, log cleanup, reports, certificate checks, health checks, AWS resource scheduling, database maintenance and file synchronization.

A scheduler alone is not production automation. The job must be observable and able to report failure.

## 2. Cron

Cron is a traditional Unix/Linux mechanism for recurring jobs.

Check the daemon depending on distribution:

```bash
systemctl status cron
systemctl status crond
```

Manage a user's crontab:

```bash
crontab -e
crontab -l
crontab -r
```

- `-e` = edit
- `-l` = list
- `-r` = remove the user's entire crontab
- `-u USER` = operate on another user's crontab when permitted

## 3. Cron expression anatomy

```text
* * * * * command
│ │ │ │ │
│ │ │ │ └── day of week (0-7)
│ │ │ └──── month (1-12)
│ │ └────── day of month (1-31)
│ └──────── hour (0-23)
└────────── minute (0-59)
```

Example:

```cron
30 9 * * 1-5 /opt/scripts/health-check.sh
```

Runs at 09:30 Monday-Friday.

Operators:

- `*` = every value
- `,` = list
- `-` = range
- `/` = step

Examples:

```cron
* * * * * /opt/scripts/job.sh
*/5 * * * * /opt/scripts/job.sh
0 2 * * * /opt/scripts/backup.sh
30 3 * * 0 /opt/scripts/weekly.sh
0 9 * * 1-5 /opt/scripts/report.sh
0 1 1 * * /opt/scripts/monthly.sh
```

## 4. Cron environment — common production failure

A job may work manually but fail under cron because the environment differs:

- Different `PATH`
- Different working directory
- Missing startup files
- Missing environment variables
- Different user/permissions
- No interactive terminal

Prefer absolute paths:

```cron
*/10 * * * * /usr/local/bin/aws s3 sync /opt/backup s3://my-bucket/backup/ >> /var/log/backup.log 2>&1
```

Find command paths:

```bash
command -v aws
command -v bash
command -v python3
```

`>>` appends stdout. `2>&1` sends stderr to the same destination.

## 5. Cron troubleshooting

If a backup job did not run:

```text
Is cron running?
 ↓
crontab installed?
 ↓
schedule correct?
 ↓
script executable?
 ↓
manual execution works?
 ↓
PATH/environment correct?
 ↓
permissions/user correct?
 ↓
logs show what?
```

Commands:

```bash
systemctl status cron
crontab -l
ls -l /opt/scripts/backup.sh
/opt/scripts/backup.sh
command -v aws
journalctl -u cron --since '1 hour ago'
```

System cron locations commonly include:

```text
/etc/crontab
/etc/cron.d/
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

System crontab entries can include a user field.

## 6. Cron vs systemd timers

| Cron | systemd timer |
|---|---|
| Simple and portable | Integrated with systemd |
| Traditional scheduler | Richer lifecycle/dependency model |
| Basic scheduling | Strong journal integration |
| Cron expressions | Calendar and monotonic timers |

Do not claim timers universally replace cron. Choose based on portability, systemd integration, observability and workload requirements.

## 7. systemd timer architecture

A timer activates a `.service` unit:

```text
backup.timer
    ↓ triggers
backup.service
    ↓
backup script
```

The timer schedules the service; the service performs the work.

## 8. Service unit for a scheduled job

```ini
# /etc/systemd/system/devops-backup.service
[Unit]
Description=DevOps Backup Job

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

`Type=oneshot` is appropriate for a job that runs and exits.

## 9. Timer unit

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

- `OnCalendar=` = calendar schedule
- `Persistent=true` = run after a missed calendar event when the timer becomes active again
- `WantedBy=timers.target` = enable into timer target

## 10. Enable and inspect timers

After creating/changing units:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now devops-backup.timer
systemctl list-timers
systemctl list-timers --all
systemctl status devops-backup.timer
```

Inspect the triggered service:

```bash
systemctl status devops-backup.service
journalctl -u devops-backup.service -n 50 --no-pager
```

## 11. systemd timer scheduling types

### OnCalendar

Wall-clock/calendar scheduling:

```ini
OnCalendar=Mon..Fri 09:00
```

### OnBootSec

Relative to boot:

```ini
OnBootSec=10min
```

### OnStartupSec

Relative to activation of the systemd manager.

### OnUnitActiveSec

Relative to the last activation of the associated unit:

```ini
OnUnitActiveSec=1h
```

### OnUnitInactiveSec

Relative to when the unit became inactive.

## 12. Persistent timers

If a timer was scheduled for 02:00 while the server was powered off, `Persistent=true` allows systemd to trigger the missed job when the timer becomes active after boot.

## 13. Timer troubleshooting

Timer not triggering:

```bash
systemctl status myjob.timer
systemctl list-timers --all
systemctl cat myjob.timer
```

If the timer triggers but the job fails:

```bash
systemctl status myjob.service
journalctl -u myjob.service -n 100 --no-pager
```

Important distinction:

```text
Timer problem ≠ service/job problem
```

If the unit file changed:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myjob.timer
```

For a `Type=oneshot` service, an exit is expected; determine whether the exit status is successful.

## 14. AWS connection

Common EC2 pattern:

```text
EC2 → cron/systemd timer → Bash → AWS CLI → AWS API
```

Example:

```bash
#!/usr/bin/env bash
set -euo pipefail
aws s3 sync /opt/backups "s3://my-backup-bucket/"
```

Production requirements:

- Prefer EC2 instance roles over hardcoded access keys where appropriate.
- Use absolute paths.
- Log stdout/stderr.
- Check exit codes.
- Prevent overlapping executions when required.
- Monitor failures.

For AWS-managed workloads, an AWS-native scheduler such as **Amazon EventBridge Scheduler** may be a better fit than running the schedule inside an EC2 host.

## 15. Production best practices

Use absolute paths, make jobs idempotent where possible, prevent overlapping runs with `flock` when needed, log failures, use dedicated service accounts, avoid unnecessary root execution and monitor both scheduler and job success.

Example overlap prevention:

```bash
flock -n /var/lock/backup.lock /opt/scripts/backup.sh
```

## Command cheat sheet

```bash
crontab -e
crontab -l
systemctl status cron
journalctl -u cron
systemctl list-timers --all
systemctl status myjob.timer
systemctl status myjob.service
journalctl -u myjob.service
systemctl daemon-reload
systemctl enable --now myjob.timer
```

## DevOps mental model

Never troubleshoot only the schedule. A scheduled automation chain has multiple failure points:

```text
scheduler → trigger → environment → permissions → command → exit status → expected result
```