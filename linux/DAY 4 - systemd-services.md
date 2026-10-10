# DAY 4 - systemd Services, Dependencies and Restart Behavior

## 1. systemd

`systemd` is the system and service manager used by many modern Linux distributions. It manages services, dependencies, startup ordering, timers, mounts and journal integration.

Mental model:

```text
systemd → unit → service → process
```

## 2. Core systemctl commands

```bash
systemctl status nginx
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx
sudo systemctl enable nginx
sudo systemctl enable --now nginx
sudo systemctl disable nginx
systemctl is-active nginx
systemctl is-enabled nginx
```

**Active** = current runtime state. **Enabled** = configured to start automatically. `start` does not automatically mean `enable`.

## 3. Inspect units

```bash
systemctl status nginx
systemctl cat nginx
systemctl show nginx
systemctl show -p FragmentPath nginx
systemctl --failed
systemctl list-units --type=service
systemctl list-unit-files --type=service
```

Common unit types: `.service`, `.socket`, `.target`, `.timer`, `.mount`, `.path`.

## 4. Service unit structure

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

- `[Unit]` — metadata, dependencies and ordering
- `[Service]` — how the process runs
- `[Install]` — enablement/target relationships

## 5. Dependencies and ordering

```ini
Requires=postgresql.service
Wants=redis.service
After=network-online.target
Before=myapp.service
```

`Requires=` is a stronger dependency relationship. `Wants=` is weaker.

**Important:** `After=` only controls ordering. It does not itself start the referenced unit.

Inspect dependencies:

```bash
systemctl list-dependencies nginx
systemctl list-dependencies --reverse nginx
systemctl show nginx -p After -p Before -p Requires -p Wants
```

## 6. Restart behavior

Common `Restart=` values:

- `no`
- `on-success`
- `on-failure`
- `on-abnormal`
- `on-abort`
- `always`

Example:

```ini
Restart=on-failure
RestartSec=5
```

`RestartSec` delays restart attempts. systemd can also rate-limit repeated starts with `StartLimitIntervalSec` and `StartLimitBurst`.

## 7. journalctl for service troubleshooting

```bash
journalctl -u nginx
journalctl -u nginx -n 100 --no-pager
journalctl -u nginx -f
journalctl -u nginx -p err
journalctl -u nginx --since '30 minutes ago'
journalctl -b
journalctl -b -1
```

Start with:

```bash
systemctl status myapp
journalctl -u myapp -n 100 --no-pager
```

## 8. daemon-reload

After changing a unit file:

```bash
sudo systemctl daemon-reload
sudo systemctl restart myapp
```

Without `daemon-reload`, systemd may still use the previously loaded unit definition.

## 9. Service starts and immediately exits

Investigate in this order:

```text
systemctl status myapp
        ↓
journalctl -u myapp
        ↓
exit code / failure reason
        ↓
check User/Group
        ↓
check ExecStart and environment
        ↓
check permissions / dependencies / ports
        ↓
test as service user
        ↓
fix root cause
        ↓
daemon-reload if unit changed
        ↓
restart and verify
```

Example permission investigation:

```bash
systemctl show myapp -p User -p Group
ls -l /opt/myapp/config/application.yml
namei -l /opt/myapp/config/application.yml
getfacl /opt/myapp/config/application.yml
sudo -u appuser test -r /opt/myapp/config/application.yml
echo $?
```

Do not use `chmod 777` as a generic fix. Correct ownership, group, mode bits or ACLs.

Other immediate-exit causes include invalid configuration, missing environment variables, missing executables, bad `ExecStart`, ports already in use, missing dependencies and application crashes.

## 10. AWS / DevOps connection

On EC2, systemd commonly manages Nginx, application processes, Docker, monitoring agents and deployment agents.

```text
EC2 → systemd → service → application → AWS APIs
```

Keep systemd failures, Linux permission failures, network failures and AWS IAM `AccessDenied` failures as separate troubleshooting layers.

## Command cheat sheet

```bash
systemctl status <service>
systemctl start <service>
systemctl stop <service>
systemctl restart <service>
systemctl reload <service>
systemctl enable <service>
systemctl enable --now <service>
systemctl is-active <service>
systemctl is-enabled <service>
systemctl cat <service>
systemctl show <service>
systemctl --failed
systemctl daemon-reload
journalctl -u <service>
journalctl -u <service> -f
```