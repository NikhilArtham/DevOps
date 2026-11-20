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
