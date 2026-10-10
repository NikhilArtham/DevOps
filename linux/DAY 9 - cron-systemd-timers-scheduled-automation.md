# DAY 9 - Cron, systemd Timers & Scheduled Automation

> **Goal:** Learn how Linux runs commands automatically at a future time and how to troubleshoot scheduled jobs.

## 1. What is scheduled automation?

Sometimes we do not want to run a command manually.

Examples:

```text
Every night → backup
Every 10 minutes → health check
Every Sunday → cleanup
At boot → start an application
```

A scheduler answers:

> **When should this job run?**

The script/service answers:

> **What should happen?**

## 2. Cron

**cron** is a traditional Linux scheduler for recurring jobs.

Check the service:

```bash
systemctl status cron
```

Some distributions use:

```bash
systemctl status crond
```

## 3. User crontab

Edit your scheduled jobs:

```bash
crontab -e
```

List them:

```bash
crontab -l
```

Remove them:

```bash
crontab -r
```

Be careful: `crontab -r` removes the user's crontab.

## 4. Cron syntax

A normal user cron entry has five time fields:

```text
* * * * * command
│ │ │ │ │
│ │ │ │ └ day of week
│ │ │ └── month
│ │ └──── day of month
│ └────── hour
└──────── minute
```

Example:

```cron
30 9 * * 1-5 /opt/scripts/check.sh
```

Means:

> Run at 09:30 Monday to Friday.

## 5. Cron symbols

```text
*     every value
,     multiple values
-     range
/     step
```

Examples:

```cron
*/5 * * * * command       # every 5 minutes
0 9 * * 1-5 command      # weekdays at 09:00
0 9,18 * * * command     # 09:00 and 18:00
```

## 6. Common cron examples

Every day at 2 AM:

```cron
0 2 * * * /opt/scripts/backup.sh
```

Every Sunday at 3:30 AM:

```cron
30 3 * * 0 /opt/scripts/weekly.sh
```

## 7. Why cron jobs fail even when the script works manually

This is a very common production problem.

Your interactive shell may have:

```text
PATH
environment variables
working directory
credentials
```

Cron may not have the same environment.

Bad:

```cron
*/10 * * * * aws s3 sync ./backup s3://bucket/backup
```

Better:

```cron
*/10 * * * * /usr/local/bin/aws s3 sync /opt/backup s3://bucket/backup >> /var/log/backup.log 2>&1
```

Use absolute paths for important commands and files.

Find a command's path:

```bash
command -v aws
command -v bash
```

## 8. Logging cron jobs

A scheduled job should not fail silently.

```cron
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

Meaning:

- `>>` = append standard output
- `2>&1` = send errors to the same place as standard output

## 9. Cron troubleshooting

If a job did not run:

```text
Is cron running?
      ↓
crontab -l
      ↓
Is the schedule correct?
      ↓
Can the script execute?
      ↓
Does it work manually?
      ↓
Does it depend on PATH/env?
      ↓
Check logs
      ↓
Check user + permissions
```

Useful commands:

```bash
systemctl status cron
crontab -l
ls -l /opt/scripts/job.sh
command -v aws
journalctl -u cron --since "1 hour ago"
```

## 10. systemd timers

Modern Linux systems can also schedule systemd services using **timers**.

The architecture is:

```text
myjob.timer
     ↓ triggers
myjob.service
     ↓
actual script/work
```

The timer decides **when**. The service performs **what**.

## 11. Example systemd service

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Backup job

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
```

`Type=oneshot` means the service performs a job and exits.

## 12. Example systemd timer

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run backup every day

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Important settings:

- `OnCalendar` = when to run
- `Persistent=true` = catch up after a missed calendar event when appropriate
- `WantedBy=timers.target` = enable with the timers target

## 13. Activate a timer

After creating or changing unit files:

```bash
sudo systemctl daemon-reload
```

Start now:

```bash
sudo systemctl start backup.timer
```

Enable at boot:

```bash
sudo systemctl enable backup.timer
```

Both:

```bash
sudo systemctl enable --now backup.timer
```

## 14. See scheduled timers

```bash
systemctl list-timers
```

Including inactive timers:

```bash
systemctl list-timers --all
```

Check one:

```bash
systemctl status backup.timer
```

## 15. Troubleshoot the timer vs the service

This distinction is very important.

```text
Timer did not trigger
        ↓
Troubleshoot timer

Timer triggered
        ↓
Service failed
        ↓
Troubleshoot service
```

Check:

```bash
systemctl status backup.timer
systemctl status backup.service
journalctl -u backup.service -n 100 --no-pager
```

## 16. Different timer types

### `OnCalendar`

Run at a calendar time:

```ini
OnCalendar=Mon..Fri 09:00
```

### `OnBootSec`

Run a certain time after boot:

```ini
OnBootSec=10min
```

### `OnUnitActiveSec`

Run repeatedly based on the last activation:

```ini
OnUnitActiveSec=1h
```

## 17. Cron vs systemd timer

| Cron | systemd timer |
|---|---|
| Simple | More integrated with systemd |
| Very common | Strong service/log integration |
| Cron syntax | Calendar + monotonic timers |
| Basic environment | Native systemd lifecycle |

Do not think one is always better. Choose based on the environment and operational requirements.

## 18. AWS connection

On an EC2 server you might have:

```text
EC2
 ↓
cron / systemd timer
 ↓
Bash script
 ↓
AWS CLI
 ↓
S3 / EC2 / CloudWatch
```

For example, a scheduled backup:

```bash
aws s3 sync /opt/backups s3://my-backup-bucket/
```

Use an EC2 IAM role rather than hardcoding AWS access keys when appropriate.

For AWS-managed workloads, a service such as EventBridge Scheduler may be more appropriate than running a scheduler inside an EC2 host.

## 19. Production best practices

- Use absolute paths.
- Log output and failures.
- Use least-privilege service accounts.
- Make jobs idempotent where possible.
- Prevent overlapping jobs when required, for example with `flock`.
- Check exit codes.
- Monitor important failures.

### Beginner takeaway

Remember:

- **cron** = traditional scheduled jobs
- **crontab** = where user cron jobs are defined
- **systemd timer** = scheduler integrated with systemd
- **`.timer`** = when
- **`.service`** = what
- **`journalctl`** = investigate service execution

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.