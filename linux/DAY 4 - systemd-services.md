# DAY 4 - systemd Services, Dependencies and Restart Behavior

## 1. What is systemd?

`systemd` is the system and service manager used by many modern Linux distributions. It manages services, dependencies, startup ordering, targets, timers, mounts and journal integration.

Think:
`systemd → unit → service → process`

## 2. Core systemctl commands

| Goal | Command |
|---|---|
| Check status | `systemctl status nginx` |
| Start | `sudo systemctl start nginx` |
| Stop | `sudo systemctl stop nginx` |
| Restart | `sudo systemctl restart nginx` |
| Reload config | `sudo systemctl reload nginx` |
| Enable at boot | `sudo systemctl enable nginx` |
| Enable + start | `sudo systemctl enable --now nginx` |
| Disable at boot | `sudo systemctl disable nginx` |
| Check active state | `systemctl is-active nginx` |
| Check boot state | `systemctl is-enabled nginx` |

### Active vs enabled

`active` means the service is currently running. `enabled` means systemd is configured to activate it during the appropriate boot process.

`start` does not automatically mean `enable`.

## 3. Inspecting services

```bash
systemctl status nginx
systemctl cat nginx
systemctl show nginx
systemctl show -p FragmentPath nginx
systemctl --failed
systemctl list-units --type=service
systemctl list-unit-files --type=service
```

Common unit types include `.service`, `.socket`, `.target`, `.timer`, `.mount` and `.path`.

## 4. Service unit structure

Example:

```ini
[Unit]
Description=My Application
After=network-online.target
Wants=network-online.target

[Service]
User=appuser
Group=appgroup
ExecStart=/opt/myapp/bin/app
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`[Unit]` contains metadata, dependencies and ordering. `[Service]` defines how the process is run. `[Install]` controls how the unit is connected to targets when enabled.

## 5. Dependencies and ordering

Requirement relationships:

```ini
Requires=postgresql.service
Wants=redis.service
```

Ordering:

```ini
After=network-online.target
Before=myapp.service
```

**Interview point:** `After=` does not itself start the other unit. It only controls ordering when both units are involved.

Useful commands:

```bash
systemctl list-dependencies nginx
systemctl list-dependencies --reverse nginx
systemctl show nginx -p After -p Before -p Requires -p Wants
```

`Requires=` is a stronger dependency relationship. `Wants=` is weaker and generally allows the requesting service to continue if the wanted unit fails.

## 6. Restart behavior

Common values for `Restart=` include:

- `no`
- `on-success`
- `on-failure`
- `on-abnormal`
- `on-abort`
- `always`

Example:

```ini
[Service]
Restart=on-failure
RestartSec=5
```

`RestartSec` prevents an immediate crash/restart loop. systemd can also rate-limit repeated starts using settings such as `StartLimitIntervalSec` and `StartLimitBurst`.

## 7. journalctl

Service logs are often the fastest way to identify why a service failed.

```bash
journalctl -u nginx
journalctl -u nginx -n 100 --no-pager
journalctl -u nginx -f
journalctl -u nginx -p err
journalctl -u nginx --since '30 minutes ago'
journalctl -b
journalctl -b -1
```

Most useful first check:

```bash
systemctl status myapp
journalctl -u myapp -n 100 --no-pager
```

## 8. daemon-reload

If a unit file is modified, reload systemd's unit definitions:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

Without `daemon-reload`, systemd may continue using the previously loaded unit definition.

## 9. Realistic failure scenario — service starts and immediately exits

Suppose:

```bash
sudo systemctl start myapp
systemctl status myapp
```

shows `Active: failed` and an exit code.

### Step 1 — Read the journal

```bash
journalctl -u myapp -n 100 --no-pager
```

Suppose it reports:

```text
/opt/myapp/config/application.yml: Permission denied
```

### Step 2 — Identify the service user

```bash
systemctl show myapp -p User -p Group
```

### Step 3 — Check access

```bash
ls -l /opt/myapp/config/application.yml
namei -l /opt/myapp/config/application.yml
getfacl /opt/myapp/config/application.yml
```

### Step 4 — Test as the actual service user

```bash
sudo -u appuser test -r /opt/myapp/config/application.yml
echo $?
```

### Step 5 — Fix the root cause

Do not use `chmod 777` as a generic fix. Correct ownership, group, mode bits or ACLs according to the application's required access.

Example:

```bash
sudo chown root:appgroup /opt/myapp/config/application.yml
sudo chmod 640 /opt/myapp/config/application.yml
```

Verify:

```bash
sudo -u appuser test -r /opt/myapp/config/application.yml && echo 'OK'
sudo systemctl restart myapp
systemctl status myapp
journalctl -u myapp -n 50 --no-pager
```

### Other causes of immediate exit

- Invalid application configuration
- Missing environment variable
- Missing executable or script
- Wrong `ExecStart` path
- Permission problem
- Port already in use
- Missing dependency
- Application crash
- Wrong working directory

## 10. Production troubleshooting flow

```text
systemctl status service
        ↓
journalctl -u service
        ↓
Exit code / failure reason
        ↓
systemctl cat service
        ↓
Check User/Group
        ↓
Check ExecStart / environment
        ↓
Check permissions / dependencies / ports
        ↓
Test as service user
        ↓
Fix root cause
        ↓
daemon-reload if unit changed
        ↓
restart and verify
```

## 11. AWS / DevOps connection

On EC2, systemd commonly manages Nginx, Docker, Java/Python/Node applications, monitoring agents and deployment agents.

```text
EC2 → systemd → service → application → AWS APIs
```

Keep the layers separate: a systemd failure, Linux permission failure, network failure and AWS IAM `AccessDenied` are different problems even when they affect the same application.

## 12. Practical lab

Create a disposable service in a Linux VM:

```bash
sudo mkdir -p /opt/systemd-lab
sudo tee /opt/systemd-lab/app.sh >/dev/null <<'EOF'
#!/bin/bash
while true; do
  echo 'application running'
  sleep 10
done
EOF
sudo chmod +x /opt/systemd-lab/app.sh
```

Create `/etc/systemd/system/systemd-lab.service`:

```ini
[Unit]
Description=Systemd Learning Service
After=network.target

[Service]
ExecStart=/opt/systemd-lab/app.sh
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl start systemd-lab
systemctl status systemd-lab
sudo systemctl enable systemd-lab
journalctl -u systemd-lab -f
```

Experiment with `Restart=no` versus `Restart=on-failure` and observe the journal.

# Interview Questions

<details><summary>1. What is systemd?</summary>
systemd is the Linux system and service manager responsible for services, dependencies, startup ordering and other unit types.
</details>

<details><summary>2. What is systemctl?</summary>
`systemctl` is the command-line interface used to manage systemd units and services.
</details>

<details><summary>3. Difference between start and enable?</summary>
`start` runs the service now. `enable` configures it to start automatically during boot.
</details>

<details><summary>4. What does enable --now do?</summary>
It enables the service for boot and starts it immediately.
</details>

<details><summary>5. Active vs enabled?</summary>
Active is the current runtime state; enabled is the configured boot/startup state.
</details>

<details><summary>6. How do you troubleshoot a failed service?</summary>
Use `systemctl status` first, then `journalctl -u`. Check exit status, unit configuration, service user, paths, permissions, dependencies and application configuration.
</details>

<details><summary>7. What is journalctl?</summary>
It queries logs stored in the systemd journal.
</details>

<details><summary>8. How do you follow service logs?</summary>
Use `journalctl -u service -f`.
</details>

<details><summary>9. What is daemon-reload?</summary>
It makes systemd reload unit definitions after a unit file has changed.
</details>

<details><summary>10. Requires vs Wants?</summary>
`Requires=` expresses a stronger dependency; `Wants=` expresses a weaker desired dependency.
</details>

<details><summary>11. Does After= create a dependency?</summary>
No. `After=` controls ordering when both units are involved; it does not itself start the other unit.
</details>

<details><summary>12. What does Restart=on-failure do?</summary>
It tells systemd to restart a service when it terminates due to a failure.
</details>

<details><summary>13. Why use RestartSec?</summary>
It delays restarts and helps prevent rapid crash/restart loops.
</details>

<details><summary>14. A service starts and immediately exits. What do you check?</summary>
Check status, journal logs, exit code, ExecStart, service user, permissions, environment, dependencies, ports and the application itself.
</details>

<details><summary>15. How do you find which user runs a service?</summary>
Use `systemctl show service -p User -p Group` or inspect `User=` and `Group=` in the unit.
</details>

<details><summary>16. How do you inspect a unit definition?</summary>
Use `systemctl cat service` and `systemctl show service`.
</details>

<details><summary>17. What happens if ExecStart points to a missing executable?</summary>
systemd cannot execute the service and reports the failure through status and the journal.
</details>

<details><summary>18. Why might a service repeatedly restart?</summary>
The application may be crashing, misconfigured, missing dependencies, unable to access files, or unable to bind its port. Check the journal before changing restart policy.
</details>

<details><summary>19. When is daemon-reload required?</summary>
After changing a systemd unit file so systemd reloads the new definition.
</details>

<details><summary>20. Is every service failure a systemd failure?</summary>
No. systemd may correctly report that the application exited because of a configuration, permission, dependency, port or application-level error.
</details>

## Production mental model

**Do not restart blindly.** First determine why the process exited.

`systemctl status → journalctl → exit code → unit configuration → user/permissions → dependencies → application → network/AWS`