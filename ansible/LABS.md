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


## DAY 6 — Vault and secret handling

### Lab 18 — Encrypt a secret variable file

**Goal:** Keep a variable file unreadable in Git while allowing Ansible to decrypt it at runtime.

1. Follow the DAY 6 local inventory and create `group_vars/all/vault.yml` with the fake value `not-a-real-password`.
2. Run `ansible-vault encrypt group_vars/all/vault.yml` and enter a temporary lab Vault password.
3. Inspect only the first line with `head -n 1 group_vars/all/vault.yml`; confirm it has the Ansible Vault header.
4. Run `ansible-vault view group_vars/all/vault.yml` only in a safe terminal, and do not copy the decrypted output into logs or screenshots.
5. Run `git status --short` and confirm no plaintext copy or password file is staged.

**Expected result:** The tracked variable file is encrypted. The Vault password is not stored in the repository.

### Lab 19 — Run twice and verify idempotency without leaking the secret

**Goal:** Use an encrypted variable in a managed file while suppressing sensitive task output.

1. Run `ansible-playbook -i inventory.ini site.yml --syntax-check`.
2. Run `ansible-playbook -i inventory.ini site.yml --ask-vault-pass` and enter the lab password.
3. Run the same command a second time without changing inputs.
4. Inspect the recap: the second run should show `changed=0`.
5. Check permissions using `stat -c '%a %U %n' /tmp/ansible-vault-lab-secret.txt`; expected mode is 600.
6. Do not cat the destination file. The task uses `no_log: true`, and the lab does not require printing the secret.

**Expected result:** First run creates the file; the second run converges with no changes. The secret-bearing task output is censored.

### Lab 20 — Diagnose a deliberate failure safely

**Goal:** Use verbose output to locate a missing required variable without printing secret values.

1. Add the DAY 6 tagged assert task that checks whether `vault_missing_demo_value` is defined.
2. Run `ansible-playbook -i inventory.ini site.yml --ask-vault-pass --tags deliberate_failure -vvv`.
3. Identify the selected host, failed task and undefined variable/assertion.
4. Explain why this is not a Vault decryption or connection failure.
5. Correct the variable name or define the required non-secret demo value, rerun, then remove the deliberate task.
6. Review the output before sharing it; redact hostnames, paths or any sensitive context.

**Expected result:** The tagged assert fails with a clear message. Verbose output provides execution context but is not treated as inherently safe.

### Lab safety

- Use only the fake demo value. Never use a real password, API token, private key or production credential.
- Use `--ask-vault-pass` for this lab; do not commit a Vault password file.
- Never commit plaintext vault YAML, shell history, password files or decrypted copies.
- `no_log: true` suppresses task output but does not encrypt the destination file.
- Remove the demo file after the exercise: `rm -f /tmp/ansible-vault-lab-secret.txt`.
- If a real secret is exposed, treat it as compromised and rotate it; deleting the visible copy is not sufficient.


## DAY 7 — Troubleshooting and verbose execution

### Lab 21 — Establish the baseline before debugging

**Goal:** Prove the inventory, target selection and playbook syntax are correct before investigating task failures.

1. Create the DAY 7 local inventory and playbook.
2. Run `ansible-playbook -i inventory.ini site.yml --syntax-check`.
3. Run `ansible-playbook -i inventory.ini site.yml --list-hosts`.
4. Run `ansible-inventory -i inventory.ini --graph`.
5. Explain why `ansible.builtin.ping` checks Ansible connectivity and is not an ICMP ping.

**Expected result:** Syntax passes, localhost is selected, and the inventory graph shows the local group and host.

### Lab 22 — Verify idempotency and diagnose the deliberate command failure

**Goal:** Run the same desired-state play twice, then inspect a controlled failure.

1. Run `ansible-playbook -i inventory.ini site.yml --skip-tags deliberate_failure`.
2. Run it again with the same command; confirm the second recap reports `changed=0`.
3. Inspect `/tmp/ansible-troubleshooting-lab/status.txt` and its permissions.
4. Run `ansible-playbook -i inventory.ini site.yml --tags deliberate_failure -vvv`.
5. Identify the failing command, selected host and non-zero return code (`rc=1` is expected for the missing file).
6. Explain why the error is not a connection or YAML syntax problem.

**Expected result:** Normal runs converge; the tagged command check fails for the expected reason and verbose output provides the task context.

### Lab 23 — Practise targeted troubleshooting

1. Change the inventory group name or play host pattern so no hosts match. Use `--list-hosts` to catch the scope issue before execution, then restore it.
2. Temporarily change the expected file path in the playbook. Use task output and the registered stat result to identify the mismatch.
3. Run with `-v`, then `-vvv`; note what additional information helps and what output is unnecessary.
4. Use `--start-at-task` to resume at a named task only after confirming its prerequisites already exist.
5. Restore the correct path and run twice; verify the second run has `changed=0`.

**Expected result:** You can distinguish inventory selection, path/precondition and task execution failures, and can return the playbook to a clean idempotent state.

### Lab safety

- DAY 7 uses localhost and writes only under `/tmp/ansible-troubleshooting-lab`.
- The tagged missing-file command is deliberately expected to fail; skip the tag for normal runs.
- Do not use `--limit`, `--start-at-task` or `--step` blindly in production; understand which prerequisite tasks are being bypassed.
- Verbose output may expose paths, usernames, hostnames and command arguments. Review and redact it before sharing.
- Remove the lab directory after practice: `rm -rf /tmp/ansible-troubleshooting-lab`.


## DAY 8 — Automating AWS configuration with Ansible + SSM

### Lab 24 — Prepare the SSM connection path

**Goal:** Confirm the EC2 instance and Ansible controller meet the SSM connection plugin prerequisites.

1. Use a disposable EC2 Linux instance with SSM Agent installed and an instance profile that includes the `AmazonSSMManagedInstanceCore` baseline.
2. Confirm the instance appears as an online managed node in Systems Manager.
3. Confirm outbound HTTPS connectivity from the instance to Systems Manager and S3. For private subnets, inspect NAT or the required VPC endpoints and DNS.
4. On the controller, install the `amazon.aws` collection, AWS SDK dependencies and Session Manager Plugin.
5. Run `aws sts get-caller-identity` and confirm the controller identity is the intended role/profile.
6. Create a dedicated S3 staging bucket with restrictive access and review versioning, encryption and lifecycle settings.

**Expected result:** The instance is online in Systems Manager and the controller has the required SSM and staging-bucket access. Do not proceed with real secrets in this lab.

### Lab 25 — Configure the EC2 instance over SSM and prove idempotency

**Goal:** Run normal Ansible modules over SSM without inbound SSH.

1. Create the DAY 8 static inventory with the EC2 instance ID, `ansible_connection=amazon.aws.aws_ssm`, Region and S3 bucket name.
2. Run `ansible-playbook -i inventory.ini site.yml --list-hosts`.
3. Run `ansible-playbook -i inventory.ini site.yml --syntax-check`.
4. Run `ansible-playbook -i inventory.ini site.yml --skip-tags deliberate_failure`.
5. Run the exact same command again.
6. Confirm the second recap reports `changed=0`.
7. Verify the file under `/tmp/ansible-ssm-lab/agent.conf` on the instance through an authorized path.

**Expected result:** Ansible connects through SSM, creates the directory and config file once, and reports no changes on the second run.

### Lab 26 — Separate SSM transport failures from task failures

**Goal:** Use verbose output to diagnose a controlled task failure and identify the layer where it occurs.

1. Run `ansible-playbook -i inventory.ini site.yml --tags deliberate_failure -vvv`.
2. If the SSM connection succeeds, identify the missing-file command and its non-zero return code (normally `rc=1`).
3. Explain why this is a task-level failure rather than an SSM transport failure.
4. For additional diagnosis, compare the result with a deliberately incorrect bucket name or a test identity without required SSM permission only in a disposable environment; restore the correct settings immediately.
5. Classify the failure as controller/plugin, IAM/SSM session, S3 transfer/network, target runtime or task logic.
6. Restore the working configuration and rerun the normal playbook twice to verify `changed=0`.

**Expected result:** You can distinguish an SSM connection failure from a command/module failure and can return the environment to a working idempotent state.

### Lab safety

- Use a disposable EC2 instance and a dedicated S3 staging bucket; this exercise may incur AWS charges.
- Do not open inbound SSH just to make the SSM exercise work.
- Never place AWS access keys, passwords or production secrets in inventory, playbooks, shell history or task output.
- The SSM connection plugin stages module files through S3. Understand object cleanup, bucket versioning and secret exposure before using it in production.
- Restrict S3 and SSM permissions to the required account, Region, instance and bucket prefix.
- Remove the lab configuration file and clean up test infrastructure and staging objects when finished.
