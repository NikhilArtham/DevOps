# DAY 5 - journalctl and Log-Driven Troubleshooting

## 1. journald vs journalctl

`systemd-journald` collects journal entries. `journalctl` is the CLI used to query them.

Logs help answer:

- What failed?
- When did it fail?
- Which service/process failed?
- What happened immediately before it?
- Is the problem repeating?
- Which dependency/configuration/permission/network change is relevant?

Mental model:

```text
Incident → identify component → check status → query logs → find first useful error → correlate → verify → fix → verify again
```

## 2. Command anatomy

| Flag | Meaning |
|---|---|
| `-u` | unit/service filter |
| `-f` | follow new entries |
| `-n` | number of recent entries |
| `-b` | boot filter |
| `-p` | priority filter |
| `--since` | start time |
| `--until` | end time |
| `-k` | kernel messages |
| `-r` | reverse order/newest first |
| `--no-pager` | print directly |
| `-o` | output format |
| `-g` | message pattern filter |
| `-t` | syslog identifier/tag |
| `--disk-usage` | journal disk usage |
| `--vacuum-time` | remove old archived entries by age |
| `--vacuum-size` | remove old archived entries by size |

## 3. Essential commands

```bash
journalctl
journalctl -r
journalctl -n 100
journalctl -f
journalctl -u nginx
journalctl -u nginx -f
journalctl -u nginx -n 100 --no-pager
journalctl -b
journalctl -b -1
journalctl -k
journalctl -u myapp -p err
journalctl -u myapp --since '30 minutes ago'
journalctl -u myapp --since '08:10' --until '08:20' --no-pager
journalctl -u myapp -g 'timeout'
journalctl -t sshd
```

## 4. Priorities

```text
0 emerg
1 alert
2 crit
3 err
4 warning
5 notice
6 info
7 debug
```

Example:

```bash
journalctl -u myapp -p warning --since '1 hour ago' --no-pager
```

## 5. Progressive log narrowing

Do not read thousands of lines blindly.

```text
All logs
 ↓
Incident time window
 ↓
Affected service
 ↓
Priority
 ↓
Specific error/pattern
 ↓
Correlate with process/network/disk/memory/dependencies
```

The **first meaningful failure** is often more useful than the final cascade of errors.

## 6. Log-driven troubleshooting

For an API returning HTTP 500:

```bash
systemctl status myapp
journalctl -u myapp --since '15 minutes ago' --no-pager
```

Look for connection refused, timeout, permission denied, address-in-use, authentication failure, configuration errors, OOM or dependency failures.

If the log says:

```text
Connection refused: 10.0.2.15:5432
```

verify instead of blindly restarting:

```bash
ss -lntp | grep 5432
nc -vz 10.0.2.15 5432
```

Logs provide evidence; commands and metrics validate the hypothesis.

## 7. Realistic dependency failure

Application is active but cannot connect to PostgreSQL.

```text
myapp
 ↓
connection refused
 ↓
PostgreSQL not listening
 ↓
PostgreSQL failed
 ↓
journal shows Permission denied
 ↓
inspect DB file ownership/permissions
```

Commands:

```bash
systemctl status postgresql
journalctl -u postgresql --since '20 minutes ago' --no-pager
ls -l /path/to/database/file
namei -l /path/to/database/file
```

Fix the actual root cause, then:

```bash
sudo systemctl restart postgresql
systemctl status postgresql
journalctl -u postgresql -n 50 --no-pager
systemctl status myapp
```

## 8. Repeated restart / port conflict

If a service keeps restarting:

```bash
systemctl status myapp
journalctl -u myapp -n 200 --no-pager
```

If the log says `Address already in use`:

```bash
ss -lntp | grep :8080
```

Identify the process owning the port before deciding what to stop.

## 9. Journal disk usage and retention

```bash
journalctl --disk-usage
cat /etc/systemd/journald.conf
```

Supported cleanup examples:

```bash
sudo journalctl --vacuum-time=7d
sudo journalctl --vacuum-size=1G
```

Do not manually delete journal files as the first response to a disk alert.

## 10. Logs outside journald

Common locations vary by distribution:

```text
/var/log/
/var/log/syslog
/var/log/messages
/var/log/auth.log
/var/log/secure
```

Applications may also write to `/var/log/<app>/` or `/opt/<app>/logs/`.

## 11. AWS / DevOps connection

On EC2, correlate Linux logs with AWS observability/control-plane information:

```text
EC2 failure
 ↓
systemctl / journalctl
 ↓
OS/network/config clue
 ↓
CloudWatch Logs/Metrics when applicable
 ↓
CloudTrail/AWS API error when applicable
 ↓
IAM / security group / route / service investigation
```

Examples:

- `AccessDenied` → IAM policies and CloudTrail when relevant.
- Connection timeout → routes, security groups, NACLs, DNS and target health.
- OOM → kernel journal + memory metrics.
- Disk full → `df`, `du`, journal disk usage and application logs.

## Command cheat sheet

```bash
journalctl -u <service>
journalctl -u <service> -f
journalctl -u <service> -n 100 --no-pager
journalctl -u <service> -p err
journalctl -u <service> --since '1 hour ago'
journalctl -b
journalctl -b -1
journalctl -k
journalctl -g 'pattern'
journalctl --disk-usage
journalctl --vacuum-time=7d
```