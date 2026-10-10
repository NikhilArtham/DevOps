# DAY 4 - Linux systemd Services

> **Goal:** Understand what a Linux service is, how `systemctl` manages it, and how to troubleshoot a service that fails.

## 1. What is a service?

A service is a program that normally runs in the background and provides something useful.

Examples:

- SSH server
- Web server such as Nginx
- Application server
- Monitoring agent
- Database

On many modern Linux systems, **systemd** is responsible for starting and managing these services.

## 2. What is systemd?

Think of systemd as a manager for Linux startup and services.

```text
Linux boots
   ↓
systemd starts
   ↓
systemd starts required services
   ↓
services keep running / perform work
```

`systemctl` is the command-line tool used to talk to systemd.

## 3. Check a service

```bash
systemctl status nginx
```

This tells you whether the service is running and often shows recent log messages.

Important states:

- `active (running)` = currently running
- `inactive` = not running
- `failed` = systemd tried and the service failed
- `activating` = startup is in progress

## 4. Start, stop and restart

```bash
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
```

Meanings:

- `start` = start now
- `stop` = stop now
- `restart` = stop and start again

Check after changing something:

```bash
systemctl status nginx
```

## 5. `enable` vs `start`

This is a common interview question.

```bash
systemctl start nginx
```

means **start it now**.

```bash
systemctl enable nginx
```

means **configure it to start automatically during boot**.

To do both:

```bash
sudo systemctl enable --now nginx
```

- `enable` = boot-time activation
- `--now` = also start immediately

## 6. What is a unit?

systemd manages different types of units.

Common ones:

```text
.service  → service
.timer    → scheduled trigger
.socket   → socket activation
.target   → group of units
.mount    → filesystem mount
```

For today's topic, focus on `.service` units.

## 7. Inspect a service definition

```bash
systemctl cat nginx.service
```

You can also inspect properties:

```bash
systemctl show nginx
```

A service file commonly contains sections such as:

```ini
[Unit]
Description=Example Application

[Service]
ExecStart=/opt/myapp/start.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## 8. Important service settings

### `ExecStart`

The command systemd starts.

### `Restart`

Controls whether systemd should restart a service after certain failures.

Common values include:

```text
no
on-failure
always
```

### `RestartSec`

How long systemd waits before restarting.

```ini
RestartSec=5
```

## 9. Dependencies

Services often depend on other resources.

Two useful settings:

```ini
Requires=database.service
Wants=network-online.target
```

- `Requires` = stronger dependency
- `Wants` = weaker dependency

Ordering is separate:

```ini
After=network-online.target
```

**Important:** `After=` controls order. It does not by itself start the other unit.

## 10. What is `daemon-reload`?

If you create or modify a systemd unit file:

```bash
sudo systemctl daemon-reload
```

This tells systemd to reread unit definitions.

Then start/restart the service as appropriate.

## 11. Troubleshoot a failed service

Start with:

```bash
systemctl status myapp.service
```

Then:

```bash
journalctl -u myapp.service -n 100 --no-pager
```

Look for the **first useful error**, not just the last message.

## 12. Service starts and immediately exits

First ask: is that actually wrong?

A `Type=oneshot` service is expected to run and exit.

For a normal long-running service, an immediate exit can indicate:

- wrong command
- missing file
- permission problem
- bad configuration
- missing environment variable
- port already in use
- dependency failure
- application crash

Commands:

```bash
systemctl status myapp
journalctl -u myapp -n 100 --no-pager
```

## 13. Permission problem example

Suppose systemd runs the application as:

```ini
User=appuser
```

but the application tries to read:

```text
/opt/myapp/config/app.conf
```

Your own SSH user may be able to read it while `appuser` cannot.

Check:

```bash
namei -l /opt/myapp/config/app.conf
getfacl /opt/myapp/config/app.conf
```

The key question is:

> **What user is the service actually running as?**

## 14. A production troubleshooting flow

```text
Service is down
     ↓
systemctl status
     ↓
Read journalctl logs
     ↓
Identify first meaningful error
     ↓
Check config / permissions / ports / dependencies
     ↓
Fix root cause
     ↓
daemon-reload if unit changed
     ↓
Restart service
     ↓
Verify status + application behavior
```

## 15. Useful commands

```bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl enable nginx
systemctl enable --now nginx
systemctl is-active nginx
systemctl is-enabled nginx
systemctl --failed
systemctl cat nginx
systemctl show nginx
sudo systemctl daemon-reload
journalctl -u nginx -n 100 --no-pager
```

### Beginner takeaway

Remember:

- **systemd** = service manager
- **systemctl** = command used to control systemd
- **start** = start now
- **enable** = start automatically at boot
- **status** = see current state
- **journalctl** = investigate service logs
- **daemon-reload** = reread changed unit files

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.