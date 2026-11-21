# DAY 3 — Ansible Playbooks, Idempotency and Handlers

## Today's objective

Build a small playbook that installs and starts Nginx, manages a web page, and reloads Nginx only when the page changes. Run the playbook twice and verify that the second run makes no changes and does not notify the handler. Then trigger one deliberate undefined-variable failure and diagnose it with verbose output.

## 1. Playbook mental model

A playbook is a YAML automation file containing one or more plays. Each play targets a set of hosts and defines the desired state through tasks.

A typical execution flow:

1. Ansible reads the inventory and selects hosts using the play's `hosts` pattern.
2. It gathers facts if enabled.
3. It runs tasks in order.
4. A task that reports `changed` can notify one or more handlers.
5. Notified handlers normally run once at the end of the play, even if several tasks notify the same handler.

Important playbook keys:

| Key | Purpose |
|---|---|
| `hosts` | Inventory pattern that selects managed hosts |
| `become` | Enables privilege escalation for tasks |
| `gather_facts` | Controls automatic fact gathering |
| `vars` | Defines play-scoped variables |
| `tasks` | Ordered list of work to perform |
| `handlers` | Event-triggered tasks, commonly service reloads/restarts |
| `notify` | Connects a changed task to a handler |

Use fully qualified module names such as `ansible.builtin.copy` and `ansible.builtin.service`.

## 2. Idempotency: describe the desired state

Idempotency means repeated application of the same desired state converges without additional changes after the first successful run.

For example:

```yaml
- name: Ensure application directory exists
  ansible.builtin.file:
    path: /opt/myapp
    state: directory
    mode: "0755"
```

- If the directory is missing, Ansible creates it and reports `changed`.
- If it already exists with the desired type and permissions, Ansible reports `ok`.
- If its permissions drift, Ansible corrects them and reports `changed`.

Contrast that with a shell command that appends a line every time:

```yaml
- name: Non-idempotent example
  ansible.builtin.shell: echo "APP_ENV=production" >> /etc/myapp.env
```

Each run may append another copy. Prefer state-aware modules such as `file`, `copy`, `template`, `package`, `service` and `lineinfile`.

A task reporting `changed` is not automatically a problem. It means Ansible changed the host to move it toward the desired state. The goal is for a stable second run to report no further changes.

## 3. Handlers: run follow-up work only after a change

A handler is a special task that runs when another task notifies it.

```yaml
tasks:
  - name: Deploy application configuration
    ansible.builtin.template:
      src: app.conf.j2
      dest: /etc/myapp/app.conf
    notify: Restart application

handlers:
  - name: Restart application
    ansible.builtin.service:
      name: myapp
      state: restarted
```

Key behavior:

- A task must report `changed` for its notification to trigger.
- If the task reports `ok`, its handler is not notified.
- If several tasks notify the same handler during one play, the handler normally runs once.
- Handlers run at the end of the play by default, after ordinary tasks.
- A handler can be run earlier using `ansible.builtin.meta: flush_handlers`.
- Handler names must match the notification name, unless you use a handler's `listen` topic.

Handlers are useful for service restarts/reloads, cache refreshes and other actions that should happen only after configuration changes. Do not use a handler for work that must run on every playbook execution.

## 4. Minimal inventory

This exercise uses the local machine, so no remote VM or SSH key is required. It installs Nginx and writes a page under `/var/www/html`; run it only on a disposable Linux lab machine where you have sudo/root access and a systemd-managed Nginx service.

Create `inventory.ini`:

```ini
[web]
localhost ansible_connection=local
```

## 5. Small playbook: install Nginx, deploy a page, notify a handler

Create `web.yml`:

```yaml
---
- name: Configure a simple Nginx web server
  hosts: web
  gather_facts: false
  become: true

  tasks:
    - name: Ensure Nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Ensure Nginx is enabled and running
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy the managed web page
      ansible.builtin.copy:
        dest: /var/www/html/ansible-demo.html
        content: |
          <html>
          <head><title>Ansible Demo</title></head>
          <body><h1>Managed by Ansible</h1></body>
          </html>
        owner: root
        group: root
        mode: "0644"
      notify: Reload Nginx

    - name: Deliberate failure for troubleshooting practice
      ansible.builtin.debug:
        msg: "Application port is {{ app_prt }}"
      tags:
        - deliberate_failure

  handlers:
    - name: Reload Nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

The failure task is intentionally tagged so the normal playbook run does not execute it unless you select that tag. The correct variable name would be `app_port`; `app_prt` is deliberately misspelled.

### Why the task order matters

1. Install the package.
2. Ensure the service is started.
3. Deploy the page.
4. If the page changed, notify the reload handler.
5. The handler reloads Nginx after the normal tasks complete.

The service is started before the handler can reload it. On a clean host, the first run installs and starts Nginx, creates the page and then reloads Nginx. On the second run, the package, service and page should already be in the desired state, so no handler notification is expected.

## 6. Run it twice and verify idempotency

Syntax check first:

```bash
ansible-playbook -i inventory.ini web.yml --syntax-check
```

Run one:

```bash
ansible-playbook -i inventory.ini web.yml
```

Expected on a clean supported Linux lab host:

- Nginx package may be installed.
- Nginx is started and enabled.
- The HTML page is created.
- The copy task reports `changed` and notifies the handler.
- The handler runs once and reloads Nginx.

Run two without changing the playbook:

```bash
ansible-playbook -i inventory.ini web.yml
```

Expected:

- Package task: `ok`
- Service task: `ok`
- Copy task: `ok`
- No handler notification and no handler execution
- Recap should normally show `changed=0`

The exact first-run count depends on the machine's initial state and package manager. The important verification is that the second unchanged run reports no further changes.

Check the page:

```bash
cat /var/www/html/ansible-demo.html
curl -I http://localhost/ansible-demo.html
```

The Nginx default site must be enabled and listening for the HTTP request. If your distribution uses a different document root or site configuration, adapt the destination to that environment.

### Prove that the handler runs only after a change

Change the page heading from `Managed by Ansible` to `Updated by Ansible` in the playbook and run it again.

Expected: the copy task reports `changed`, notifies the handler, and the handler reloads Nginx. Run the exact same playbook once more; the copy task should report `ok` and the handler should not run.

## 7. Deliberate failure with verbose output

Run only the intentionally failing task:

```bash
ansible-playbook -i inventory.ini web.yml --tags deliberate_failure -vvv
```

Expected behavior: the debug task fails because `app_prt` is undefined.

Troubleshooting sequence:

1. Confirm the inventory host selected is `localhost`.
2. Locate the task named `Deliberate failure for troubleshooting practice`.
3. Read the undefined-variable diagnostic and identify `app_prt`.
4. Compare it with the intended variable name, `app_port`.
5. Correct the spelling or define the variable intentionally, then rerun.

Verbose flags:

- `-v`: more task result detail.
- `-vv`: additional task and execution detail.
- `-vvv`: useful connection and module diagnostic detail.
- `-vvvv`: very detailed connection debugging; protect logs because they may expose sensitive operational information.

Verbose output helps locate the failing host, task and context. It does not fix the underlying issue.

After completing the exercise, remove the deliberate failure task or leave it tagged and excluded from normal runs.

## 8. Handler troubleshooting and common mistakes

| Symptom | Likely reason | What to check |
|---|---|---|
| Handler does not run | Notifying task reported `ok` | Check whether the managed content actually changed |
| Handler name not found | Notification name does not match | Match `notify` to handler `name`, or use `listen` |
| Handler runs more than expected across plays | Each play has its own handler execution context | Inspect play boundaries and notifications |
| Service reload fails | Service is not running or does not support reload | Check service status and whether restart is required |
| Every run reports changed | Unstable content, timestamps, permissions or a non-idempotent command | Inspect task diff and remove unnecessary dynamic values |
| Play fails before tasks | YAML syntax or inventory issue | Run `--syntax-check` and inspect inventory |
| Handler is skipped after a later task fails | By default, a host that fails later in a play may not run its pending handler | Consider `force_handlers: true` only when safe and appropriate |

### Important handler details

- A handler normally runs once at the end of the play even if multiple tasks notify it.
- `meta: flush_handlers` runs pending handlers at that point; later notifications can cause the handler to run again.
- `force_handlers: true` can run notified handlers even after a later task failure, subject to host reachability and other conditions. Do not enable it blindly: a handler may act on partially configured state.
- A service reload is not always interchangeable with restart. Reload asks the service to reread configuration without a full stop/start when supported; restart stops and starts it.

## 9. Production considerations

- Use templates for environment-specific configuration, with variables stored in inventory/group vars or a secrets manager.
- Validate application configuration before notifying a restart/reload. A bad configuration should not be deployed just because the file task succeeded.
- Use handlers for change-triggered operations, not for every routine action.
- Keep tasks small and named clearly so failures are easy to locate.
- Use `serial` and health checks for controlled rolling changes across production fleets.
- Consider `force_handlers` carefully; a restart after a failed deployment can make an incident worse.
- Keep secrets out of debug output and verbose logs.
- Confirm service health after reload, rather than treating a successful handler result as proof that the application is healthy.

## 10. Today's quick revision

- Playbook = declarative, ordered automation for selected hosts.
- Task = one operation using a module.
- Idempotency = repeated execution converges without additional changes.
- Handler = event-triggered task notified by a changed task.
- A task that reports `ok` does not notify its handler.
- Multiple notifications for the same handler in one play normally result in one handler execution.
- Handlers normally run at the end of the play; `meta: flush_handlers` can run them earlier.
- `-vvv` helps diagnose the deliberate undefined-variable failure.

**Interview mental model:** Tasks enforce desired state; changed tasks emit notifications; handlers perform necessary follow-up actions once, avoiding unnecessary service restarts.
