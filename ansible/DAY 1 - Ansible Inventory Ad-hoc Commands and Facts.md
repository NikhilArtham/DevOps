# DAY 1 — Ansible Inventory, Ad-hoc Commands and Facts

## Today's objective

Understand how Ansible selects managed hosts, runs one-off tasks, and discovers host information. Build the smallest inventory and playbook, run the playbook twice, and diagnose one intentional failure with verbose output.

## 1. The mental model

Ansible control flow:

1. The **control node** runs Ansible.
2. The **inventory** defines managed hosts and groups.
3. A **pattern** selects which inventory hosts to target.
4. A **module** performs the requested task on each selected host.
5. **Facts** provide information gathered from managed hosts.

Ansible is agentless for its common Linux/Unix SSH workflow. The control node connects over SSH and usually requires Python on the managed host. Windows commonly uses WinRM or PSRP rather than SSH-based Linux assumptions.

## 2. Inventory: which hosts should Ansible manage?

Create `inventory.ini`:

```ini
[web]
web01 ansible_host=192.0.2.10
web02 ansible_host=192.0.2.11

[web:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/ansible_lab
```

The IP addresses above are documentation-only examples. Replace them with reachable lab hosts before running commands.

- `[web]` creates a host group.
- `web01` and `web02` are inventory aliases; they do not need to match DNS names.
- `ansible_host` is the actual address Ansible connects to.
- `ansible_user` sets the SSH user.
- `ansible_ssh_private_key_file` selects the private key.

Keep private keys outside the repository. Do not commit credentials or private key material.

### Useful inventory commands

```bash
# Show all hosts
ansible-inventory -i inventory.ini --list

# Show the inventory as a tree
ansible-inventory -i inventory.ini --graph

# Show variables resolved for one host
ansible-inventory -i inventory.ini --host web01
```

### Host patterns

```bash
ansible all -i inventory.ini --list-hosts
ansible web -i inventory.ini --list-hosts
ansible web01 -i inventory.ini --list-hosts
ansible 'web:!web02' -i inventory.ini --list-hosts
```

Always use `--list-hosts` first when a pattern could select production hosts unexpectedly.

### Inventory variable precedence — practical view

Variables can be defined in inventory, group vars, host vars, play vars, task vars and other supported locations. More specific and higher-precedence definitions can override broader values. Avoid solving every conflict by adding more overrides: keep environment-specific values organized and inspect the resolved host data with `ansible-inventory --host`.

A common layout:

```text
ansible-project/
├── inventory/
│   ├── hosts.ini
│   ├── group_vars/
│   │   └── web.yml
│   └── host_vars/
│       └── web01.yml
└── site.yml
```

## 3. Ad-hoc commands: one task without a playbook

General syntax:

```bash
ansible <pattern> -i <inventory> -m <module> -a "<module arguments>"
```

Examples:

```bash
# Connectivity and Python/module execution check
ansible web -i inventory.ini -m ansible.builtin.ping

# Uptime (command module)
ansible web -i inventory.ini -m ansible.builtin.command -a "uptime"

# Disk usage
ansible web -i inventory.ini -m ansible.builtin.command -a "df -h"

# Gather facts
ansible web -i inventory.ini -m ansible.builtin.setup

# Gather only IPv4 address facts
ansible web -i inventory.ini -m ansible.builtin.setup -a "filter=ansible_all_ipv4_addresses"
```

Important distinction: Ansible's `ping` module is not ICMP ping. It verifies that Ansible can connect and execute its module successfully.

Use `command` for commands that do not require shell features such as pipes, redirects, variable expansion or globbing. Use `ansible.builtin.shell` only when shell behavior is genuinely required.

### Common ad-hoc options

- `-i inventory.ini`: choose the inventory source.
- `-m ping`: choose the module.
- `-a "uptime"`: pass module arguments.
- `-b`: become another user, usually root, when authorized.
- `-K`: prompt for the become password.
- `-f 10`: set the number of parallel forks.
- `-o`: display compact output.
- `-v`, `-vv`, `-vvv`, `-vvvv`: increase diagnostic verbosity.

Do not add `-b` to every command by habit. Use privilege escalation only for tasks that need it.

## 4. Facts: what Ansible knows about a host

Facts are gathered by the `setup` module. With `gather_facts: true`, a playbook normally gathers facts before running its tasks.

Useful examples:

```bash
# All gathered facts
ansible web -i inventory.ini -m ansible.builtin.setup

# Distribution-related facts
ansible web -i inventory.ini -m ansible.builtin.setup -a "filter=ansible_distribution*"

# IPv4 addresses
ansible web -i inventory.ini -m ansible.builtin.setup -a "filter=ansible_all_ipv4_addresses"

# Memory-related facts
ansible web -i inventory.ini -m ansible.builtin.setup -a "filter=ansible_mem*"
```

In a playbook, reference facts using `ansible_facts`, for example:

- `ansible_facts['distribution']`
- `ansible_facts['default_ipv4']['address']`
- `ansible_facts['architecture']`

Fact availability can differ by operating system, network configuration and Ansible version. A fact such as `default_ipv4` may be absent when Ansible cannot identify a default interface. Inspect actual gathered facts before assuming a key exists.

## 5. Minimal inventory + playbook example

Use this inventory, replacing the example addresses and SSH settings with your own lab hosts:

```ini
[web]
web01 ansible_host=192.0.2.10

[web:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/ansible_lab
```

Create `facts-and-idempotency.yml`:

```yaml
---
- name: Inspect hosts and ensure a marker file exists
  hosts: web
  gather_facts: true
  become: false

  tasks:
    - name: Display host operating system and IP
      ansible.builtin.debug:
        msg: "OS={{ ansible_facts['distribution'] }}, IP={{ ansible_facts['default_ipv4']['address'] }}"

    - name: Ensure the lab directory exists
      ansible.builtin.file:
        path: /tmp/ansible-inventory-lab
        state: directory
        mode: "0755"

    - name: Ensure the marker file has the expected content
      ansible.builtin.copy:
        dest: /tmp/ansible-inventory-lab/status.txt
        content: "Managed by Ansible\n"
        mode: "0644"
```

The playbook demonstrates inventory targeting, automatic fact gathering, fact references, and idempotent state management.

### Run it twice

```bash
# First run
ansible-playbook -i inventory.ini facts-and-idempotency.yml

# Second run
ansible-playbook -i inventory.ini facts-and-idempotency.yml
```

Expected result on a clean host:

- First run: the directory and file are created, so the recap should normally show `changed=2` per host.
- Second run: the desired state already exists, so the recap should show `changed=0` per host.
- Both runs should complete with `failed=0` and `unreachable=0`.

The exact first-run count can differ if the directory or file already exists in the desired state. Idempotency means repeated runs converge on the same desired state without additional changes; it does not mean every task always reports `ok` on its first run.

Verify the result:

```bash
ansible web -i inventory.ini -m ansible.builtin.command -a "cat /tmp/ansible-inventory-lab/status.txt"
```

## 6. Deliberate failure and verbose diagnosis

Run a command against a path that intentionally does not exist:

```bash
ansible web -i inventory.ini -m ansible.builtin.command -a "ls /path/that/does/not/exist" -vvv
```

Expected behavior: the task fails with a non-zero return code because the path does not exist. In verbose output, inspect:

1. Which inventory host was selected.
2. Whether SSH connection setup succeeded.
3. The module Ansible invoked.
4. The command's `rc`, `stdout` and `stderr`.
5. Whether the failure is a connection problem or a remote command failure.

Interpretation:

- `UNREACHABLE`: investigate DNS/IP, routing, port 22, SSH user, key permissions and SSH authentication.
- `FAILED` with `rc != 0`: the host was reached, but the module/command failed. Read `stderr` and the return code.
- Python interpreter/module errors: check Python availability and interpreter configuration on the managed host.

Do not paste unredacted `-vvvv` logs into tickets or public chats. Verbose output can expose usernames, paths, host details and command arguments.

## 7. Troubleshooting checklist

| Symptom | What to check |
|---|---|
| No hosts matched | Inventory path, group name, host pattern and `--list-hosts` |
| SSH connection refused | Host address, SSH service, firewall and port |
| Permission denied (publickey) | SSH username, correct key, key permissions and authorized_keys |
| Host is unreachable | DNS, routing, security groups/firewall and SSH connectivity |
| sudo/become fails | Whether the user has sudo rights; use `-b` and `-K` only when needed |
| A fact is undefined | Inspect `setup` output; verify the fact exists on that host |
| Second run reports changed | Identify which task changed and whether inputs/content/mode are stable |

Useful diagnostic sequence:

```bash
ansible-inventory -i inventory.ini --graph
ansible web -i inventory.ini --list-hosts
ansible web -i inventory.ini -m ansible.builtin.ping -vvv
ansible web -i inventory.ini -m ansible.builtin.setup -a "filter=ansible_distribution*"
ansible-playbook -i inventory.ini facts-and-idempotency.yml --diff
```

Use `--diff` carefully: it can display file content changes. Avoid using it with secrets.

## 8. Production considerations and common mistakes

- Keep separate inventories or inventory groups for development, QA and production.
- Confirm the target list before executing ad-hoc commands against a broad pattern such as `all`.
- Prefer playbooks for repeatable changes; ad-hoc commands are useful for diagnostics and one-off operations.
- Use fully qualified collection names such as `ansible.builtin.ping` to make module origin clear.
- Keep secrets in Ansible Vault or an approved secrets manager; never store passwords or private keys in plain-text inventory.
- Use least-privilege SSH users and privilege escalation only where required.
- Facts can be cached, but cached facts may become stale. Know whether a play uses freshly gathered or cached data.
- Do not assume an inventory alias is the machine's hostname; `ansible_host` may point to a different address.
- Idempotency is a property of the task's desired-state behavior. A command that blindly appends text or restarts services every run is not automatically idempotent.

## 9. Today's quick revision

- Inventory = hosts, groups and connection variables.
- Pattern = selection of inventory hosts.
- Ad-hoc command = one module execution without a playbook.
- `ping` = Ansible connectivity/module check, not ICMP.
- `setup` = fact-gathering module.
- Facts = host-specific data exposed to playbooks.
- `-vvv` = useful diagnostic detail for connection and task failures.
- Idempotency = repeated application of the same desired state produces no further changes.

**Interview mental model:** Inventory selects the target, a module performs the operation, facts provide context, and idempotent tasks safely converge hosts toward the declared state.
