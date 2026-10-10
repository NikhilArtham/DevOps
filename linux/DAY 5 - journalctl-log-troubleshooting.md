# DAY 5 - Linux Logs & journalctl Troubleshooting

> **Goal:** Learn how Linux records service events and how to use logs to find the reason something failed.

## 1. What is a log?

A log is a record of something that happened.

For example:

```text
Application started
Database connection failed
Permission denied
Port already in use
```

Logs are one of the most important sources of evidence during a production incident.

## 2. What is journald?

On systemd-based Linux systems, **systemd-journald** collects many system and service messages.

You normally read those messages with:

```bash
journalctl
```

Think:

```text
Linux / services / applications
          ↓
       journald
          ↓
      journalctl
          ↓
      Engineer
```

## 3. See recent logs

```bash
journalctl
```

If the output is long, use:

```bash
journalctl --no-pager
```

`--no-pager` prints directly instead of opening a pager.

Recent lines:

```bash
journalctl -n 50
```

`-n 50` = show the last 50 lines.

## 4. Logs for one service

This is one of the most useful commands:

```bash
journalctl -u nginx
```

`-u` = unit/service.

For a recent view:

```bash
journalctl -u nginx -n 100 --no-pager
```

## 5. Follow logs live

```bash
journalctl -u nginx -f
```

`-f` = follow new log entries as they arrive.

This is useful when reproducing a problem while watching the logs.

Stop with `Ctrl+C`.

## 6. Logs from the current boot

```bash
journalctl -b
```

`-b` = current boot.

This is useful when a server restarted and you want to investigate what happened during the current boot.

## 7. Filter by time

```bash
journalctl --since "1 hour ago"
journalctl --since "2026-10-10 15:00" --until "2026-10-10 16:00"
```

This reduces noise and helps you focus on the incident window.

## 8. Filter by priority

```bash
journalctl -p err
```

This asks for error-priority messages and above.

Common priorities include:

```text
emerg
alert
crit
err
warning
notice
info
debug
```

## 9. Search inside logs

You can combine journalctl with normal Linux tools:

```bash
journalctl -u myapp --since "30 minutes ago" | grep -i error
```

`grep -i` means case-insensitive matching.

Some versions also support journalctl's own pattern search:

```bash
journalctl -g "timeout"
```

## 10. Kernel messages

```bash
journalctl -k
```

`-k` = kernel messages.

These can help investigate hardware, drivers, memory, filesystem and networking problems.

## 11. Find the first meaningful error

A common troubleshooting mistake is to look only at the final error.

Example:

```text
Application failed
   ↓
Database connection failed
   ↓
DNS lookup failed
   ↓
Network configuration changed
```

The application error is a symptom. The earlier network/DNS problem may be the root cause.

A useful method:

```text
Identify incident time
       ↓
Read logs around that time
       ↓
Find first abnormal event
       ↓
Follow the dependency chain
       ↓
Confirm root cause
```

## 12. Example: service keeps restarting

Run:

```bash
systemctl status myapp
journalctl -u myapp -n 100 --no-pager
```

You might discover:

```text
Permission denied
```

Then investigate the service user and file permissions rather than repeatedly restarting it.

## 13. Example: database connection failure

Suppose an API returns HTTP 500.

Do not immediately assume the API code is broken.

Check:

```text
API logs
 ↓
Database connection error
 ↓
DNS / network / credentials
 ↓
Database health
```

Use timestamps to correlate events across services.

## 14. Journal disk usage

Check how much disk the journal uses:

```bash
journalctl --disk-usage
```

`--disk-usage` = show current journal disk usage.

Retention can be managed using appropriate systemd-journald configuration or controlled cleanup commands such as:

```bash
sudo journalctl --vacuum-time=7d
```

`--vacuum-time=7d` removes journal files older than the specified retention period.

Do not delete logs blindly during an incident because they may be useful evidence.

## 15. Production troubleshooting example

Problem:

> Application stopped responding at 14:20.

Approach:

```bash
journalctl -u myapp --since "14:15" --until "14:25"
```

Then correlate:

```text
14:18 configuration changed
14:19 application restarted
14:19 permission denied
14:20 health check failed
```

The timeline is more useful than simply knowing that the application is down.

## 16. AWS / DevOps connection

On EC2, local Linux logs are one layer of troubleshooting.

You may also investigate:

```text
Linux journal
    +
Application logs
    +
CloudWatch Logs
    +
CloudWatch metrics
    +
AWS infrastructure events
```

The goal is to correlate evidence rather than blame the first component that reports an error.

## 17. Commands to remember

```bash
journalctl
journalctl -u nginx
journalctl -u nginx -f
journalctl -u nginx -n 100 --no-pager
journalctl -b
journalctl -k
journalctl -p err
journalctl --since "1 hour ago"
journalctl --disk-usage
journalctl --vacuum-time=7d
```

### Beginner takeaway

Think of `journalctl` as **a searchable history of what Linux and its services have been saying**.

When something breaks:

```text
What broke?
   ↓
When did it break?
   ↓
Which service?
   ↓
What does its log say?
   ↓
What happened immediately before the failure?
```

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.