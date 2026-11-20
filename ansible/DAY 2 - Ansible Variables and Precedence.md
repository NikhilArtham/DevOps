# DAY 2 — Ansible Variables and Precedence

## Today's objective

Understand where Ansible variables can be defined, which value wins when names collide, and how to verify the effective value. Build a minimal local inventory and playbook, run it twice to prove idempotency, and diagnose one intentional variable error with verbose output.

## 1. Mental model

A variable is a named value used by a playbook, role, template or task. Variables let one automation definition work across environments without hardcoding every setting.

For example, the same application deployment may use:

- Development: `app_port: 8080`
- QA: `app_port: 8081`
- Production: `app_port: 8443`

When the same variable is defined in multiple places, Ansible applies variable precedence. The value that wins is the **effective value** used by the task.

Do not confuse variable precedence with play execution order. Precedence decides which value is used; it does not mean the variable source's task executes later.

## 2. Common places to define variables

| Location | Typical use |
|---|---|
| Role defaults (`defaults/main.yml`) | Safe defaults intended to be overridden |
| Inventory group vars (`group_vars/`) | Shared values for a group, such as web servers |
| Inventory host vars (`host_vars/`) | Host-specific overrides |
| Play `vars:` | Values scoped to a play |
| `vars_files:` | Load variables from external YAML files |
| Block/task `vars:` | Narrow-scope overrides |
| `include_vars` | Load a variable file during execution |
| `set_fact` | Create host variables during a play |
| Registered variables | Capture a task's result |
| Extra vars (`-e`) | Explicit runtime overrides |

The official precedence system contains more categories and detailed ordering rules than this introductory list. The next section gives a practical ordering for the common sources shown here.

## 3. Precedence: which value wins?

For the sources used in today's example, the useful simplified ordering from lower to higher is:

1. Inventory group variables
2. Inventory host variables
3. Play variables
4. Task variables
5. Extra variables passed with `-e`

Example definitions:

- Group says `app_port: 8080`
- Host says `app_port: 8081`
- Play says `app_port: 8082`
- One task says `app_port: 8083`
- CLI extra vars say `app_port=9090`

Expected effective values:

| Where it is read | Effective value |
|---|---:|
| Task without task-level override | 8082 |
| Task with its own `vars:` override | 8083 |
| Either task when CLI passes `-e app_port=9090` | 9090 |

**Important:** This is not the complete Ansible precedence chart. Role defaults, role vars, include parameters, `set_fact`, registered variables, connection variables, and other sources have their own documented precedence. Extra vars have the highest variable precedence. For production troubleshooting, consult the official precedence table for the exact source pair you are comparing.

### Group variable ordering

When multiple inventory groups apply to a host, child groups normally override parent groups. For sibling groups at the same level, Ansible's group merge ordering matters; `ansible_group_priority` can control ordering in inventory. Keep group layouts simple and avoid defining the same setting in many places.

## 4. Smallest reproducible example

This lab uses a local connection so it can be tested without a remote VM or SSH key.

Project structure:

```text
ansible-day2/
├── inventory.ini
├── group_vars/
│   └── web.yml
├── host_vars/
│   └── localhost.yml
└── variables.yml
```

### inventory.ini

```ini
[web]
localhost ansible_connection=local
```

### group_vars/web.yml

```yaml
---
app_port: 8080
app_environment: development
```

### host_vars/localhost.yml

```yaml
---
app_port: 8081
```

Ansible automatically loads `group_vars/` and `host_vars/` when they are placed in the expected locations relative to the inventory source or playbook. For this example, keep these directories beside `inventory.ini` and `variables.yml`.

### variables.yml

```yaml
---
- name: Demonstrate Ansible variable precedence
  hosts: web
  gather_facts: false

  vars:
    app_port: 8082

  tasks:
    - name: Display the play-level effective value
      ansible.builtin.debug:
        msg: "Port={{ app_port }}, environment={{ app_environment }}"

    - name: Display a task-level override
      ansible.builtin.debug:
        msg: "Task-level port={{ app_port }}"
      vars:
        app_port: 8083

    - name: Ensure application configuration directory exists
      ansible.builtin.file:
        path: /tmp/ansible-vars-lab
        state: directory
        mode: "0755"

    - name: Write configuration using the effective play-level value
      ansible.builtin.copy:
        dest: /tmp/ansible-vars-lab/app.conf
        content: |
          port={{ app_port }}
          environment={{ app_environment }}
        mode: "0644"
```

Without command-line overrides, expect:

- First debug task: port 8082 and environment development.
- Second debug task: task-level port 8083.
- The generated file: port 8082 and environment development.

The task-level override is scoped to that debug task. It does not change the play-level value used by the later copy task.

## 5. Run it twice and verify idempotency

Run from the project directory:

```bash
ansible-inventory -i inventory.ini --host localhost
ansible-playbook -i inventory.ini variables.yml
ansible-playbook -i inventory.ini variables.yml
```

Expected result:

- Inventory inspection shows inventory-derived values, including host-level `app_port: 8081`. It does not show the final play/task/CLI value because those scopes are applied during play execution.
- First playbook run creates the directory and file; normally `changed=2`.
- Second run should report `changed=0`, assuming the file content and permissions have not changed.

Verify the rendered configuration:

```bash
cat /tmp/ansible-vars-lab/app.conf
```

Expected:

```ini
port=8082
environment=development
```

### Verify extra-variable precedence

```bash
ansible-playbook -i inventory.ini variables.yml -e "app_port=9090"
```

Both debug tasks now see 9090, including the task that declares 8083 locally, because extra vars have higher precedence. The configuration file changes to port 9090 once. Run the exact command again; it should report `changed=0`.

To restore the original play-level value, run without `-e`. That run changes the file back to port 8082; the next identical run should report `changed=0`.

**Idempotency rule:** The same inputs and desired state should converge. Changing a variable intentionally changes the desired configuration, so one run reporting `changed` is expected.

## 6. Deliberate failure: misspelled variable

Add this temporary task to the play:

```yaml
    - name: Deliberately reference an undefined variable
      ansible.builtin.debug:
        msg: "Port={{ app_prt }}"
      tags:
        - deliberate_failure
```

Run only that task with verbose output:

```bash
ansible-playbook -i inventory.ini variables.yml --tags deliberate_failure -vvv
```

Expected behavior: the task fails because `app_prt` is undefined. The correct variable is `app_port`.

How to diagnose:

1. Confirm that Ansible selected the expected inventory host.
2. Locate the failed task name in the output.
3. Read the undefined-variable message; it identifies the unresolved name.
4. Compare the variable spelling with `group_vars`, `host_vars` and play vars.
5. Correct the typo and rerun the normal playbook.

Verbose output increases diagnostic detail, but it cannot make a missing variable exist. The fix is to correct the variable name or deliberately provide a valid default when that is appropriate.

Remove the deliberate-failure task after the exercise.

## 7. Troubleshooting variable values

| Symptom | Likely cause | How to investigate |
|---|---|---|
| Variable is undefined | Wrong name, wrong file location or file not loaded | Check spelling and `group_vars/` / `host_vars/` placement |
| Unexpected value | Higher-precedence source overrides the expected value | Search all definitions and inspect task/play/CLI inputs |
| Group value does not apply | Host is not in that group or inventory source differs | Run `ansible-inventory --graph` |
| CLI value is ignored | Extra var syntax is malformed or another variable name is used | Check `-e 'key=value'` and the exact key |
| Value differs between hosts | Host vars or group membership differs | Inspect each host and compare inventory |
| YAML parsing error | Indentation, quoting or invalid YAML structure | Run `ansible-playbook --syntax-check` |

Useful commands:

```bash
# Display inventory groups
ansible-inventory -i inventory.ini --graph

# Display inventory variables for one host
ansible-inventory -i inventory.ini --host localhost

# Validate playbook syntax
ansible-playbook -i inventory.ini variables.yml --syntax-check

# Show task execution and variable-related diagnostics
ansible-playbook -i inventory.ini variables.yml -vvv

# Show changes made to managed files (avoid when content contains secrets)
ansible-playbook -i inventory.ini variables.yml --diff
```

Remember: `ansible-inventory --host` shows inventory-derived variables, not every variable that may be introduced later by play scope, task scope, roles, facts or extra vars.

## 8. Production practices and common mistakes

- Keep environment-specific values in separate inventory/group variable files rather than duplicating entire playbooks.
- Use role defaults for overridable defaults; avoid placing every setting in high-precedence locations.
- Avoid defining the same variable in many places. It makes incidents harder to debug.
- Use meaningful names such as `app_port`, `app_environment` and `db_endpoint`; avoid generic names that collide with other roles.
- Use YAML variable files for structured values and quote strings where YAML type inference could surprise you.
- Store passwords, tokens and private data in Ansible Vault or an approved secrets manager. Marking a value `no_log: true` can reduce log exposure but does not encrypt the source file or protect every external copy.
- Do not use `-e` as a permanent substitute for a well-structured inventory. Extra vars are useful for deliberate runtime overrides and CI inputs, but can unexpectedly override other values.
- Validate required variables early with `assert` or role argument specifications.
- Avoid relying on undocumented assumptions about precedence. If a value matters in production, make its source explicit and test the resolved behavior.

## 9. Today's quick revision

- Variables make playbooks reusable across environments.
- Group vars apply to a group; host vars can specialize one host.
- Play vars apply within a play; task vars apply within their task scope.
- Extra vars passed with `-e` have the highest variable precedence.
- `ansible-inventory --host` helps inspect inventory values but does not show all runtime scopes.
- Idempotency must be tested by running the same playbook twice with the same inputs.
- `-vvv` helps locate the failed task and diagnostic context; an undefined variable still requires a real correction.

**Interview mental model:** Find every definition of the variable, identify its scope and precedence, inspect the effective runtime value, then make the source of truth unambiguous.
