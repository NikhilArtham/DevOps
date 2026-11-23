# Ansible

Ansible is an agentless automation tool commonly used for configuration management, application deployment and operational tasks.

## Learning Structure

- [DAY 1 — Inventory, Ad-hoc Commands and Facts](DAY%201%20-%20Ansible%20Inventory%20Ad-hoc%20Commands%20and%20Facts.md)
- [DAY 2 — Variables and Precedence](DAY%202%20-%20Ansible%20Variables%20and%20Precedence.md)
- [DAY 3 — Playbooks, Idempotency and Handlers](DAY%203%20-%20Ansible%20Playbooks%20Idempotency%20and%20Handlers.md)
- [DAY 4 — Templates, Loops, Conditionals and Registers](DAY%204%20-%20Ansible%20Templates%20Loops%20Conditionals%20and%20Registers.md)
- [DAY 5 — Roles and Reusable Structure](DAY%205%20-%20Ansible%20Roles%20and%20Reusable%20Structure.md)
- [Hands-on Labs](LABS.md)
- [Interview Questions](INTERVIEW%20QUESTIONS.md)

## DAY 1 — Inventory, Ad-hoc Commands and Facts

- Inventory files, host groups, host variables and host patterns
- Ad-hoc commands and common modules
- Fact gathering with the setup module
- Minimal inventory + playbook example
- Verify idempotency by running the same playbook twice
- Diagnose an intentional remote command failure using `-vvv`

## DAY 2 — Variables and Precedence

- Variable sources: group vars, host vars, play vars, task vars and extra vars
- Practical precedence model and effective runtime values
- Minimal local inventory + playbook example
- Verify idempotency with variable-driven configuration
- Override a variable with `-e`
- Diagnose an undefined variable with `-vvv`

## DAY 3 — Playbooks, Idempotency and Handlers

- Play/task structure and execution order
- Desired-state modules and idempotency
- Notify/handler behavior and handler execution timing
- Nginx example: install, start, deploy a page and reload only after change
- Run twice and verify the second run has `changed=0`
- Diagnose a deliberate undefined-variable failure using `-vvv`

## DAY 4 — Templates, loops, conditionals and registers

- Render dynamic configuration with Jinja2 and the template module
- Iterate over lists and dictionaries with loop
- Use when expressions to control task execution
- Capture task results with register and inspect rc/stdout/stderr
- Use changed_when for read-only command checks
- Run twice to verify template and file-task idempotency
- Diagnose an undefined variable in a conditional using -vvv

## DAY 5 — Roles and reusable structure

- Understand the standard role directory structure and role search paths
- Generate a role skeleton with ansible-galaxy role init
- Separate defaults, tasks, handlers and templates
- Invoke a role from a playbook and override its defaults
- Compare roles:, import_role and include_role
- Run twice to verify role idempotency
- Diagnose an undefined variable using -vvv

## Suggested practice sequence

1. Inspect inventory with `ansible-inventory --graph`.
2. Preview targets with `ansible ... --list-hosts`.
3. Test connectivity with `ansible ... -m ansible.builtin.ping`.
4. Gather facts with `ansible ... -m ansible.builtin.setup`.
5. Run the DAY 1, DAY 2, DAY 3, DAY 4 and DAY 5 playbooks twice and compare recap output.
6. Trace a variable override, verify handler notification behavior and diagnose the deliberate failures.

Use disposable lab hosts. DAY 2 uses a local connection and writes under `/tmp`; DAY 3 installs Nginx and writes under `/var/www/html`. Never commit private keys or plaintext secrets.
