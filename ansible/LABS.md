# Ansible Labs

Hands-on exercises are centralized here. Keep day-wise learning notes in their respective DAY files.

## Lab 1 — Inventory discovery and host targeting

**Goal:** Understand groups, aliases, host variables and patterns.

1. Create an INI inventory with a `web` group containing two lab hosts.
2. Set `ansible_host` and `ansible_user` for each host.
3. Run:

   ```bash
   ansible-inventory -i inventory.ini --graph
   ansible web -i inventory.ini --list-hosts
   ansible 'web:!web02' -i inventory.ini --list-hosts
   ```

4. Explain why the inventory alias can differ from the actual remote hostname.

**Expected result:** The graph shows the web group and its members; the exclusion pattern selects only web01.

## Lab 2 — Ad-hoc command execution

**Goal:** Execute one-off diagnostic commands without writing a playbook.

1. Test connectivity with `ansible web -i inventory.ini -m ansible.builtin.ping`.
2. Run `uptime` and `df -h` using `ansible.builtin.command`.
3. Gather facts with `ansible.builtin.setup`.
4. Repeat a command against one host only.

**Expected result:** You can explain the target pattern, module, arguments and per-host result.

## Lab 3 — Facts inspection

**Goal:** Inspect facts and use them in a playbook.

1. Run the setup module and inspect distribution, architecture, memory and IPv4 facts.
2. Create a play targeting `web` with `gather_facts: true`.
3. Print distribution and default IPv4 address using `ansible.builtin.debug`.
4. Test on a host with multiple interfaces and inspect whether the default IPv4 fact matches the expected route.

**Expected result:** The play displays host-specific values. Explain why a fact may be missing or differ between hosts.

## Lab 4 — Idempotency: run the same playbook twice

**Goal:** Verify convergence instead of assuming a successful first run proves idempotency.

Use the `facts-and-idempotency.yml` example from DAY 1, or create a playbook that ensures a directory and a file with fixed content and permissions.

1. Run the playbook once and record the recap.
2. Run it a second time without changing the inputs.
3. Compare the `changed` count.
4. Change the file content intentionally, run again, and explain why one task changes.
5. Restore the desired content and repeat.

**Expected result:** First run creates missing resources; the next unchanged run reports `changed=0` per host. A deliberate content change causes the copy task to report changed once, then converge again.

## Lab 5 — Deliberate failure with verbose output

**Goal:** Distinguish connection failures from remote command failures.

Run:

```bash
ansible web -i inventory.ini -m ansible.builtin.command -a "ls /path/that/does/not/exist" -vvv
```

1. Confirm that the task fails because the path is missing.
2. Find the selected host, module invocation, return code and stderr.
3. Compare with a deliberate connection failure in a disposable lab by using an incorrect test port or unreachable test host.
4. Explain the difference between `FAILED` and `UNREACHABLE`.

**Expected result:** A missing path returns a remote command failure; an SSH connection problem is reported as unreachable. Do not expose secrets or unredacted verbose logs.

## Lab safety

- Use disposable hosts or an approved test environment.
- Replace documentation-only IP addresses with your own lab addresses.
- Never commit private keys, passwords or unencrypted secrets.
- Use `--list-hosts` before broad targeting and `--diff` only when file contents are safe to display.


## DAY 2 — Variables and precedence

### Lab 6 — Trace variable precedence

**Goal:** Identify which variable definition wins at runtime.

1. Reuse the DAY 2 local inventory with `group_vars/web.yml` and `host_vars/localhost.yml`.
2. Set `app_port` to 8080 in group vars, 8081 in host vars, and 8082 in play vars.
3. Add a debug task with a task-level `app_port: 8083`.
4. Run the playbook and record both debug outputs.
5. Run again with `-e "app_port=9090"` and explain why the extra var wins.
6. Inspect `ansible-inventory --host localhost` and explain why it does not display the final play/task/CLI value.

**Expected result:** The ordinary play task sees 8082, the task with its local override sees 8083, and both see 9090 when the extra var is supplied.

### Lab 7 — Verify idempotency with variable-driven configuration

**Goal:** Prove that a variable-driven configuration converges.

1. Use the DAY 2 playbook to create `/tmp/ansible-vars-lab/app.conf`.
2. Run it twice with no overrides; the second run should report `changed=0`.
3. Run twice with `-e "app_port=9090"`; the first run updates the file and the second reports `changed=0`.
4. Run twice without the override and observe it converge back to the play-level port.
5. Inspect the final file contents after each change.

**Expected result:** Stable inputs produce no further changes; changing the effective variable changes the desired file once.

### Lab 8 — Diagnose an undefined variable

**Goal:** Use verbose output to find a variable typo.

1. Add a temporary debug task that references `app_prt` instead of `app_port`.
2. Tag it `deliberate_failure`.
3. Run `ansible-playbook -i inventory.ini variables.yml --tags deliberate_failure -vvv`.
4. Identify the undefined name and failed task.
5. Correct the typo, rerun, then remove the deliberate-failure task.

**Expected result:** Ansible reports an undefined variable. Verbose output provides task/host context; correcting the variable name resolves the failure.


## DAY 3 — Playbooks, idempotency and handlers

### Lab 9 — Read and trace a playbook

**Goal:** Understand play, task, handler and notification flow.

1. Read the DAY 3 `web.yml` example.
2. Identify the inventory target, privilege escalation, ordered tasks and handler.
3. Run `ansible-playbook -i inventory.ini web.yml --syntax-check`.
4. Explain why the service is started before the reload handler can run.

**Expected result:** You can trace the execution order and explain which task can notify the handler.

### Lab 10 — Prove idempotency and handler behavior

**Goal:** Verify that a handler only runs when a managed resource changes.

1. Use a disposable Linux machine with Nginx and systemd support.
2. Run the DAY 3 playbook once and record changed tasks and handler execution.
3. Run it again without changing inputs; expect `changed=0` and no handler execution.
4. Change the HTML heading in the playbook and run again; expect the copy task to change and notify the handler.
5. Run once more; expect no change and no handler execution.
6. Verify the page with `curl -I http://localhost/ansible-demo.html`.

**Expected result:** The handler runs after the first deployment and after the deliberate content update, but not on an unchanged run.

### Lab 11 — Diagnose an undefined variable with verbosity

**Goal:** Identify a task failure from Ansible's diagnostic output.

1. Run `ansible-playbook -i inventory.ini web.yml --tags deliberate_failure -vvv`.
2. Identify the failed task and undefined variable `app_prt`.
3. Explain why the failure is not an SSH or inventory problem.
4. Correct the typo or define the intended value and rerun.
5. Remove the deliberate failure task from normal operational automation.

**Expected result:** Ansible reaches localhost, executes the debug task and fails because the referenced variable is undefined. The verbose output helps locate the exact task and context.

### Lab safety

- Use a disposable Linux host; the example installs Nginx and writes under `/var/www/html`.
- Run with privilege escalation only where authorized.
- Do not run package/service changes on a production host just to complete the exercise.
- Keep secrets out of playbook output and verbose logs.


## DAY 4 — Templates, loops, conditionals and registers

### Lab 12 — Render a dynamic configuration template

**Goal:** Generate a configuration file from variables and verify template idempotency.

1. Create the DAY 4 inventory, Jinja2 template and playbook.
2. Run ansible-playbook -i inventory.ini templates-loops-registers.yml --syntax-check.
3. Run the playbook once and inspect /tmp/ansible-template-lab/app.conf.
4. Run it again without changing inputs; expect changed=0.
5. Change app_port from 8080 to 9090 and run again; expect the template task to report changed.
6. Run once more; expect no further change.

**Expected result:** The rendered file contains current variables and loop-generated upstream entries. Stable inputs converge; changing an input updates the file once.

### Lab 13 — Trace loops, conditionals and registered results

**Goal:** Explain how task output controls later execution.

1. Inspect the loop that creates the logs and cache directories.
2. Add a third directory to app_directories and run the playbook.
3. Inspect the registered config_file result and identify stat.exists and stat.size.
4. Inspect port_check and identify rc and stdout.
5. Temporarily change the grep expression so it finds no matching line. Observe the non-zero return code and the conditional debug task being skipped.
6. Restore the correct expression and verify the task runs.

**Expected result:** Each list item is processed, registered values are available to later tasks, and when evaluates for the host.

### Lab 14 — Diagnose an undefined variable in a conditional

**Goal:** Use verbose output to identify a condition evaluation failure.

1. Run ansible-playbook -i inventory.ini templates-loops-registers.yml --tags deliberate_failure -vvv.
2. Find the deliberate task and identify the undefined variable app_prt.
3. Explain why this is not an SSH, inventory or template-source failure.
4. Correct the variable to app_port and rerun the tagged task.
5. Remove the deliberate-failure task after practice.

**Expected result:** The task fails while evaluating its when expression. Verbose output helps locate the task and undefined name.

### Lab safety

- The DAY 4 example writes only under /tmp/ansible-template-lab and uses a local connection.
- Do not use real credentials or secrets in template variables or debug output.
- Keep the deliberate-failure task out of normal operational playbooks.
- A successful template render does not prove the application accepts the configuration; production deployments should validate the file and perform a health check.


## DAY 5 — Roles and reusable structure

### Lab 15 — Build and invoke a reusable role

**Goal:** Move configuration tasks out of a playbook and into a role.

1. Create the DAY 5 layout with roles/app_config/defaults, tasks, handlers and templates.
2. Put configurable baseline values in defaults/main.yml.
3. Put directory and template tasks in tasks/main.yml.
4. Add the Jinja2 source under templates/ and the notification handler under handlers/main.yml.
5. Invoke the role from site.yml, overriding app_environment and app_port.
6. Run ansible-playbook -i inventory.ini site.yml --syntax-check, then run the playbook.

**Expected result:** The role renders /tmp/ansible-role-lab/app.conf using values supplied by the play. The changed template notifies the handler.

### Lab 16 — Prove role idempotency and reuse

1. Run ansible-playbook -i inventory.ini site.yml a second time without changing inputs.
2. Confirm the second run reports changed=0 and the handler does not run.
3. Change app_port from 9090 to 9191 in the role invocation and run again.
4. Confirm the template changes once and the handler runs.
5. Run again without changes and confirm changed=0.
6. Explain why changing role inputs is preferable to editing internal tasks for each environment.

**Expected result:** Same role, different input values; stable inputs converge without repeated changes.

### Lab 17 — Diagnose a role-play failure with verbosity

1. Run ansible-playbook -i inventory.ini site.yml --tags deliberate_failure -vvv.
2. Identify the task whose when expression references missing_app_version.
3. Explain why this is an undefined-variable failure rather than a missing role or inventory/connection issue.
4. Define the variable or correct the condition and rerun the tagged task.
5. Remove the deliberate-failure task after practice.

**Expected result:** The task fails during conditional evaluation. Verbose output shows the host, task and undefined variable.

### Lab safety

- The DAY 5 role uses a local connection and writes only under /tmp/ansible-role-lab.
- The handler is a debug task, not a real service reload. Use a service handler only after validating a real service configuration.
- Do not run package/service changes on production systems for this exercise.
- Keep intentional failures and secrets out of normal automation.
