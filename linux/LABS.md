# 🧪 Linux Practical Labs

> **DevOps Job-Switch Hands-On Workbook**
>
> This is the centralized Linux practice area. Every Linux study DAY adds its practical exercises here so all hands-on work is available in one place.

## How to use this file

- Complete labs in order.
- Prefer a disposable VM, EC2 test instance, container or local Linux environment.
- Do not run destructive commands against production systems.
- Mark a lab complete only after you can explain **what happened, why it happened and how you would troubleshoot it in production**.

---

## 📅 DAY 1 — Process Management

### Lab 1 — Process investigation

```bash
ps aux
ps -ef
pgrep bash
pstree
```

**Tasks:**
- Identify your shell PID and PPID.
- Find a process belonging to your user.
- Inspect it with `ps -p <PID> -f`.
- Identify its state.

### Lab 2 — Signals

Start a test process:

```bash
sleep 300 &
echo $!
```

Practice:

```bash
kill -STOP <PID>
ps -p <PID> -o pid,ppid,stat,cmd
kill -CONT <PID>
kill -TERM <PID>
```

Then explain SIGTERM vs SIGKILL and why graceful termination is preferred.

### Lab 3 — High CPU investigation

Use `top` or `htop` to identify a high-CPU process, inspect its PID with `ps`, and document the troubleshooting sequence you would use in production.

---

## 📅 DAY 2 — Filesystems & Disk Troubleshooting

### Lab 1 — Disk usage investigation

```bash
df -h
df -i
lsblk -f
findmnt
```

Then investigate a safe test directory:

```bash
mkdir -p ~/linux-labs/disk-test
for i in {1..100}; do echo "test" > ~/linux-labs/disk-test/file-$i; done
du -sh ~/linux-labs/disk-test
```

**Tasks:**
- Compare `df` and `du`.
- Count the files.
- Explain block-space vs inode usage.

### Lab 2 — Find large files

Create test files and practice:

```bash
find ~/linux-labs -type f -printf '%s %p\n' | sort -nr | head
```

Explain each pipeline stage.

### Lab 3 — Deleted-but-open file

On a disposable Linux host, create a large file, open it from a test process, delete the directory entry and inspect open deleted files with:

```bash
lsof +L1
```

Explain why `df` and `du` can disagree.

---

## 📅 DAY 3 — Permissions, Ownership, ACLs & sudo

### Lab 1 — Permission troubleshooting

```bash
mkdir -p ~/linux-labs/permissions
printf 'secret\n' > ~/linux-labs/permissions/app.conf
chmod 600 ~/linux-labs/permissions/app.conf
ls -l ~/linux-labs/permissions/app.conf
```

**Tasks:**
- Explain `600`.
- Change it to `640`.
- Create a group-based access scenario.
- Use `namei -l` to inspect parent-directory traversal.

### Lab 2 — ACL

If ACL tools are installed:

```bash
setfacl -m u:$(whoami):r ~/linux-labs/permissions/app.conf
getfacl ~/linux-labs/permissions/app.conf
```

Practice:

```bash
setfacl -x u:$(whoami) ~/linux-labs/permissions/app.conf
setfacl -b ~/linux-labs/permissions/app.conf
```

Explain `-m`, `-x` and `-b`.

### Lab 3 — sudo troubleshooting

```bash
whoami
id
sudo -l
sudo visudo -c
```

Explain how you would investigate a `sudo: permission denied` or “not allowed to run” problem without making broad sudoers changes.

---

## 📅 DAY 4 — systemd Services

### Lab 1 — Service lifecycle

Choose a harmless installed service such as SSH on a test VM.

```bash
systemctl status ssh
systemctl is-active ssh
systemctl is-enabled ssh
```

Practice understanding:

```bash
sudo systemctl start <service>
sudo systemctl stop <service>
sudo systemctl restart <service>
sudo systemctl enable <service>
```

Do not stop critical access services on a remote production host.

### Lab 2 — Create a test service

Create `/etc/systemd/system/devops-lab.service`:

```ini
[Unit]
Description=DevOps Lab Service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo "DevOps lab ran"'
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl start devops-lab.service
systemctl status devops-lab.service
journalctl -u devops-lab.service --no-pager
```

### Lab 3 — Break it intentionally

Change `ExecStart` to a nonexistent path, reload and start it again.

Troubleshoot with:

```bash
systemctl status devops-lab.service
journalctl -u devops-lab.service -n 50 --no-pager
```

Identify the exact failure and restore the service.

---

## 📅 DAY 5 — journalctl & Log Troubleshooting

### Lab 1 — Filter service logs

```bash
journalctl -u ssh -n 50 --no-pager
journalctl -u ssh --since "1 hour ago" --no-pager
journalctl -p err -b --no-pager
```

**Tasks:**
- Filter by service.
- Filter by time.
- Filter by priority.
- Explain why timestamps matter during incidents.

### Lab 2 — Generate and find a test failure

Run a harmless failed command from a test systemd service, then use:

```bash
systemctl status devops-lab.service
journalctl -u devops-lab.service -n 100 --no-pager
```

Find the first meaningful error.

### Lab 3 — Journal storage

```bash
journalctl --disk-usage
```

Explain journal retention and why log cleanup should follow an operational retention policy.

---

## 📅 DAY 6 — grep, sed, awk, sort & xargs

### Lab 1 — Build a test access log

```bash
cat > access.log <<'EOF'
10.0.0.1 GET /api/orders 200 812
10.0.0.2 GET /api/orders 500 421
10.0.0.1 GET /api/users 200 520
10.0.0.2 GET /api/orders 500 419
10.0.0.3 GET /api/orders 404 120
10.0.0.2 GET /api/orders 500 422
EOF
```

Practice:

```bash
grep '500' access.log
awk '{print $5}' access.log | sort | uniq -c | sort -nr
awk '$5 == 500 {print $1}' access.log | sort | uniq -c | sort -nr
awk '{print $3}' access.log | sort | uniq -c | sort -nr
```

### Lab 2 — sed safely

```bash
sed 's#/api/orders#/api/v2/orders#g' access.log
```

Verify that the original file was not changed.

Then explain why `sed -i` deserves extra caution during production incidents.

### Lab 3 — xargs safety

```bash
printf '%s\n' file1 file2 file3 | xargs -n 1 printf 'TARGET=%s\n'
```

Then practice NUL-safe input:

```bash
find . -maxdepth 1 -type f -print0 | xargs -0 printf '%s\n'
```

Explain why `-0` matters for filenames containing spaces/newlines.

---

## 📅 DAY 7 — Bash Scripting

### Lab 1 — Health-check script

Create `health-check.sh` that:

- Accepts service names as arguments.
- Uses a function.
- Checks `systemctl is-active`.
- Returns non-zero if any service is unhealthy.
- Prints useful output.

Validate:

```bash
bash -n health-check.sh
bash -x ./health-check.sh <service>
echo $?
```

### Lab 2 — Exit-code experiment

Create:

```bash
#!/usr/bin/env bash
false
echo "status=$?"
```

Then compare:

```bash
false && echo success
false || echo failure
false; echo continues
```

Explain exactly why the three behaviors differ.

### Lab 3 — AWS CLI automation design

Write a test script that checks the exit status of an AWS CLI command without performing a destructive action.

Example:

```bash
if ! aws sts get-caller-identity; then
  echo "AWS authentication/authorization check failed" >&2
  exit 1
fi
```

---

## 📅 DAY 8 — Bash Error Handling, Traps & Safe Automation

### Lab 1 — Cleanup with trap

Create a script using:

```bash
TMP_DIR=$(mktemp -d)
trap 'rm -rf -- "$TMP_DIR"' EXIT
```

Create files inside the directory, intentionally fail the script and verify the temporary directory is removed.

### Lab 2 — ERR trap

Practice:

```bash
set -euo pipefail
trap 'rc=$?; echo "ERROR rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR

false
echo "should not run"
```

Explain the values of `$?`, `$LINENO` and `$BASH_COMMAND`.

### Lab 3 — Prevent overlapping jobs

Use:

```bash
exec 9>/tmp/devops-lab.lock
flock -n 9 || { echo "Already running"; exit 1; }
sleep 30
```

Run the script twice quickly. Explain why the second instance exits.

---

## 📅 DAY 9 — Cron & systemd Timers

### Lab 1 — Safe cron test

Create a simple script:

```bash
mkdir -p ~/linux-labs/cron
cat > ~/linux-labs/cron/heartbeat.sh <<'EOF'
#!/usr/bin/env bash
echo "cron ran at $(date -Is)" >> "$HOME/linux-labs/cron/heartbeat.log"
EOF
chmod +x ~/linux-labs/cron/heartbeat.sh
```

Add a temporary crontab entry:

```cron
*/2 * * * * /home/YOUR_USER/linux-labs/cron/heartbeat.sh
```

After a few minutes:

```bash
cat ~/linux-labs/cron/heartbeat.log
crontab -l
```

Remove the test entry when finished:

```bash
crontab -e
```

**Goal:** prove that the scheduler executes the job and observe the difference between your interactive environment and cron's environment.

### Lab 2 — Cron failure troubleshooting

Create a job that intentionally uses a command that is not available through the cron `PATH`.

Investigate:

```bash
crontab -l
command -v <command>
journalctl -u cron --since "10 minutes ago"
```

Fix it using an absolute command path.

### Lab 3 — Build a systemd timer

Create:

`/etc/systemd/system/devops-lab.service`

```ini
[Unit]
Description=DevOps Timer Lab Service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo "systemd timer ran at $(date -Is)"'
```

Create:

`/etc/systemd/system/devops-lab.timer`

```ini
[Unit]
Description=Run DevOps Timer Lab Every Minute

[Timer]
OnUnitActiveSec=1min

[Install]
WantedBy=timers.target
```

Activate:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now devops-lab.timer
```

Observe:

```bash
systemctl list-timers --all | grep devops-lab
systemctl status devops-lab.timer
systemctl status devops-lab.service
journalctl -u devops-lab.service -n 20 --no-pager
```

Clean up:

```bash
sudo systemctl disable --now devops-lab.timer
sudo rm /etc/systemd/system/devops-lab.timer /etc/systemd/system/devops-lab.service
sudo systemctl daemon-reload
```

### Lab 4 — Timer failure investigation

Break the `ExecStart` path intentionally, reload and trigger the service.

Troubleshoot:

```bash
systemctl status devops-lab.timer
systemctl status devops-lab.service
journalctl -u devops-lab.service -n 50 --no-pager
```

Answer:

1. Is the timer healthy?
2. Did the timer trigger the service?
3. Why did the service fail?
4. What exact evidence proves the root cause?
5. What would you change to prevent recurrence?

### Lab 5 — AWS scheduled automation design

Design a non-destructive EC2 backup job:

```text
systemd timer / cron
        ↓
backup.sh
        ↓
AWS CLI
        ↓
S3
```

Your script must:

- Use absolute paths.
- Use an EC2 instance role rather than hardcoded keys where appropriate.
- Return non-zero on AWS CLI failure.
- Log stdout/stderr.
- Prevent overlapping executions with `flock` when required.

Then compare the design with an AWS-managed scheduler approach.

---

## 📅 DAY 10 — SSH: Keys, Agent, Tunneling & Common Failures

### Lab 1 — Create and inspect an SSH key

```bash
mkdir -p ~/linux-labs/ssh
chmod 700 ~/linux-labs/ssh
ssh-keygen -t ed25519 -f ~/linux-labs/ssh/lab_key -C "ssh-lab"
ls -l ~/linux-labs/ssh
chmod 600 ~/linux-labs/ssh/lab_key
```

**Tasks:**
- Identify the private and public key.
- Explain why the private key must remain secret.
- Explain `-t`, `-f` and `-C`.
- Explain why `600` is used for the private key.

### Lab 2 — ssh-agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/linux-labs/ssh/lab_key
ssh-add -l
ssh-add -L
ssh-add -D
ssh-add -l
```

**Tasks:**
- Explain every command and flag.
- Explain why an agent is useful.
- Explain the security risk of agent forwarding.

### Lab 3 — SSH verbose troubleshooting

```bash
ssh -vvv user@<test-host>
```

Record:
- Destination
- Username
- Keys offered
- Authentication method
- Exact final error

Classify the failure as network, service, authentication or configuration.

### Lab 4 — ProxyJump / bastion

On a disposable environment with a bastion and private host:

```bash
ssh -J user@bastion user@private-host
```

Then create a client alias:

```sshconfig
Host private-host
    HostName <private-ip>
    User <user>
    ProxyJump <user>@<bastion>
```

Test:

```bash
ssh private-host
```

Explain the difference between ProxyJump and agent forwarding.

### Lab 5 — Local SSH tunnel

If the SSH server can reach an internal HTTP service:

```bash
ssh -L 8080:<internal-host>:80 user@<test-host>
```

From the client:

```bash
curl http://localhost:8080
```

Then verify reachability from the SSH server:

```bash
nc -vz <internal-host> 80
```

Explain why the destination must be reachable from the SSH server side.

### Lab 6 — SSH failure injection

Introduce one safe failure at a time:

1. Wrong username.
2. Wrong private key.
3. Private key permission `644`.
4. Unreachable TCP/22.
5. Destination service not listening for a tunnel.
6. Agent loaded with multiple unrelated keys.

Useful commands:

```bash
ssh -vvv user@host
ssh-add -l
ssh -o IdentitiesOnly=yes -i <key> user@host
nc -vz host 22
ss -lntp
```

For every failure document:

```text
Observed error:
Layer that failed:
Evidence:
Root cause:
Fix:
Prevention:
```

### Lab 7 — Production-style SSH incident

Scenario:

```text
Laptop → Bastion → Private EC2

Bastion SSH works.
Private EC2 SSH times out.
```

Investigate in this order:

```bash
nc -vz <private-ip> 22
```

Then, if you have an alternate access path:

```bash
sudo ss -lntp | grep ':22'
sudo systemctl status sshd
sudo journalctl -u sshd -n 100 --no-pager
```

Inspect AWS network controls conceptually:

```text
Bastion SG: TCP 22 from trusted admin source
Private SG: TCP 22 from Bastion SG
Route table: private subnet route is valid
NACL/firewall: traffic is permitted
```

**Success criteria:** prove whether the failure is network reachability, SSH listener, authentication, authorization or configuration before making a change.

---

# 🎯 Final Capstone — Production Scheduling Incident

### Scenario

A nightly EC2 backup is reported as “not running.”

Evidence:

```text
Timer: active
Service: failed
Manual execution: works
```

### Your task

Troubleshoot without guessing.

Start with:

```bash
systemctl list-timers --all
systemctl status backup.timer
systemctl status backup.service
journalctl -u backup.service -n 100 --no-pager
```

Then investigate:

```bash
systemctl cat backup.timer
systemctl cat backup.service
command -v aws
sudo -u <service-user> command -v aws
```

Determine whether the root cause is:

- Scheduler configuration
- Timer activation
- Service configuration
- Permission
- PATH/environment
- AWS credentials/IAM
- Network/DNS
- Script logic
- Application/AWS API failure

### Success criteria

You are finished only when you can explain:

```text
Schedule → Trigger → Service → Environment → Command → Exit code → Logs → Result
```

and identify exactly which layer failed.

---

## 🏁 Practical Progress

- [ ] DAY 1 — Processes & signals
- [ ] DAY 2 — Filesystems & disk
- [ ] DAY 3 — Permissions, ACLs & sudo
- [ ] DAY 4 — systemd services
- [ ] DAY 5 — journalctl troubleshooting
- [ ] DAY 6 — grep/sed/awk/sort/xargs
- [ ] DAY 7 — Bash scripting
- [ ] DAY 8 — Bash error handling/traps
- [ ] DAY 9 — Cron & systemd timers
- [ ] DAY 10 — SSH keys, agent, tunneling & failures
- [ ] Capstone — Production scheduling incident

> **Future rule:** Every new Linux DAY will add its practical labs to this centralized workbook. No separate per-day LABS file will be created.
