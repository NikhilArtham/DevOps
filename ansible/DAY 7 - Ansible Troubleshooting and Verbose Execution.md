# DAY 7 — Ansible Troubleshooting and Verbose Execution

## 1. Troubleshooting mindset

Ansible troubleshooting is a process of locating the failing layer rather than immediately changing the playbook. Start with the simplest checks and move toward task execution:

1. **Inventory and scope:** Are the expected hosts selected?
2. **Connectivity:** Can Ansible reach the hosts using the configured connection?
3. **Privilege escalation:** Does the remote user have the required become permissions?
4. **Syntax and inputs:** Is the YAML valid, and are variables defined with the expected types?
5. **Task behavior:** Which module failed, with what arguments and return code?
6. **Target state:** Did the task make the intended change, and is the application healthy afterward?

A playbook can be syntactically valid and still fail at runtime. A playbook can also complete successfully while the application remains unhealthy. Validate both automation results and the service's actual state.

## 2. Verbosity levels

Use the lowest level that answers the diagnostic question.

| Level | Command option | Typical use |
|---|---|---|
| Default | none | Normal task status and recap |
| 1 | `-v` | More task/result details |
| 2 | `-vv` | Additional execution information |
| 3 | `-vvv` | Detailed connection and module troubleshooting |
| 4 | `-vvvv` | Very detailed connection/plugin debugging |

Examples:

```bash
ansible-playbook -i inventory.ini site.yml -v
ansible-playbook -i inventory.ini site.yml -vvv
ansible-playbook -i inventory.ini site.yml -vvvv
```

Do not assume that more verbosity is always safer. Output can contain host details, paths, command arguments and other sensitive operational context. Review and redact logs before sharing them.

## 3. First checks before deep debugging

```bash
# Validate playbook syntax without running tasks
ansible-playbook -i inventory.ini site.yml --syntax-check

# Show which hosts a playbook pattern selects
ansible-playbook -i inventory.ini site.yml --list-hosts

# Display inventory group structure
ansible-inventory -i inventory.ini --graph

# Test Ansible connectivity (uses the ping module, not ICMP)
ansible -i inventory.ini all -m ansible.builtin.ping

# Gather facts from selected hosts
ansible -i inventory.ini all -m ansible.builtin.setup
```

Important distinction: `ansible.builtin.ping` checks that Ansible can log in and execute its module on the target. It is not the same as the operating-system `ping` command or an ICMP network test.

## 4. Minimal local troubleshooting lab

This lab uses localhost so you can practise task results without changing a remote server.

Project layout:

```text
ansible-troubleshooting-lab/
├── inventory.ini
└── site.yml
```

### inventory.ini

```ini
[local]
localhost ansible_connection=local
```

### site.yml

```yaml
---
- name: Troubleshooting and verbose execution lab
  hosts: local
  gather_facts: false

  vars:
    lab_dir: /tmp/ansible-troubleshooting-lab
    expected_message: Ansible troubleshooting lab is ready

  tasks:
    - name: Ensure lab directory exists
      ansible.builtin.file:
        path: "{{ lab_dir }}"
        state: directory
        mode: "0755"

    - name: Write expected lab state
      ansible.builtin.copy:
        content: "{{ expected_message }}\n"
        dest: "{{ lab_dir }}/status.txt"
        mode: "0644"

    - name: Verify expected file exists
      ansible.builtin.stat:
        path: "{{ lab_dir }}/status.txt"
      register: status_file

    - name: Assert expected file exists
      ansible.builtin.assert:
        that:
          - status_file.stat.exists
        fail_msg: "Expected status file was not created"
        success_msg: "Expected status file exists"

    - name: Deliberately check for a file that does not exist
      ansible.builtin.command:
        argv:
          - /usr/bin/test
          - -f
          - "{{ lab_dir }}/missing.txt"
      register: deliberate_check
      changed_when: false
      tags:
        - deliberate_failure
```

The last task is intentionally expected to fail because the file is absent. It is tagged so the normal idempotency run can skip it.

## 5. Execute and verify idempotency

From the lab directory:

```bash
# Validate YAML and task structure
ansible-playbook -i inventory.ini site.yml --syntax-check

# Run normal tasks; deliberately failing task is skipped
ansible-playbook -i inventory.ini site.yml --skip-tags deliberate_failure

# Run the same command again
ansible-playbook -i inventory.ini site.yml --skip-tags deliberate_failure
```

Expected behavior:

- First run: directory and file are created; the assert passes.
- Second run: no resource changes; recap should show `changed=0`.
- The stat and assert tasks are checks, not changes.

Verify the resulting file without modifying it:

```bash
cat /tmp/ansible-troubleshooting-lab/status.txt
stat -c '%a %U %n' /tmp/ansible-troubleshooting-lab/status.txt
```

Expected content: `Ansible troubleshooting lab is ready`.

## 6. Diagnose one deliberate failure with -vvv

Run only the tagged failure:

```bash
ansible-playbook -i inventory.ini site.yml --tags deliberate_failure -vvv
```

Expected diagnosis:

- The selected host is localhost.
- The failing task is **Deliberately check for a file that does not exist**.
- The command is `/usr/bin/test -f .../missing.txt`.
- The command returns a non-zero return code (normally `rc=1`) because the file does not exist.
- This is an expected command/test failure, not an inventory, SSH, YAML syntax or Vault decryption problem.

The registered result contains fields such as `rc`, `stdout` and `stderr`. A command can fail with an empty stderr; use the return code and task intent rather than assuming stderr must contain an explanation.

After diagnosing it, either create the expected file or correct the path if you want the check to pass. Remove the deliberate task/tag before adapting the playbook for normal operations.

## 7. Common failure patterns

| Symptom | Likely area | First checks |
|---|---|---|
| No hosts matched | Inventory/pattern | `--list-hosts`, `ansible-inventory --graph` |
| UNREACHABLE | Connection/auth/network | Host, port, user, SSH key, DNS, route, firewall |
| Permission denied | Become/OS permissions | `become`, sudo policy, file ownership |
| Undefined variable | Variable loading/name | Spelling, vars_files, group_vars/host_vars, scope |
| Module reports non-zero rc | Command/application | Exact command, arguments, rc, stdout/stderr |
| YAML syntax error | YAML structure | Indentation, quotes, list/dictionary shape |
| Task changes every run | Idempotency | Prefer state-based modules; inspect content/mode/owner |
| Play succeeds but app is down | Application health | Service state, config validation, logs, health endpoint |

## 8. Troubleshooting techniques that improve signal

- **Use --start-at-task** to resume from a named task during local debugging, while remembering that skipped earlier tasks may have prepared required state.
- **Use --step** when you need interactive confirmation before each task; avoid it for unattended CI.
- **Use --limit** to reduce the host scope during a safe investigation.
- **Use --check --diff** to preview supported changes. Check mode is not supported fully by every module and is not a substitute for a real test.
- **Use register** to capture a result and inspect non-sensitive fields such as `rc`, `stdout`, `stderr` or module-specific status.
- **Use assert** to fail early with a clear message when required inputs or preconditions are missing.
- **Use changed_when and failed_when** only when you understand the command's return semantics. Incorrect overrides can hide real failures or report false changes.

## 9. Production troubleshooting discipline

1. Reproduce the issue on one safe host or a non-production environment.
2. Record the exact command, inventory, limit, tags and relevant Ansible version.
3. Identify the first failing task; later failures may be consequences.
4. Check whether the task is safe to rerun before retrying.
5. Make the smallest correction and rerun the narrowest meaningful test.
6. Verify the managed resource and application health independently.
7. Remove temporary debug tasks and avoid committing credentials or sensitive logs.

Do not paste unrestricted `-vvvv` output into tickets or chat. Verbose output can expose operational metadata and command details. Use `no_log: true` on tasks that handle secrets, but remember it does not sanitize unrelated tasks, external tools or application logs.

## 10. Interview-ready mental model

**Inventory tells Ansible where → connection tells it how to reach the host → privilege escalation tells it what permissions it has → modules perform the desired state → task results explain what happened → application checks prove the outcome.**

A strong troubleshooting answer names the failing layer, cites the evidence (task, module, rc or error), explains why other layers are unlikely, and describes a safe correction plus verification.
