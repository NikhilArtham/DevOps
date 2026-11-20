# Ansible

Ansible is an agentless automation tool commonly used for configuration management, application deployment and operational tasks.

## Learning Structure

- [DAY 1 — Inventory, Ad-hoc Commands and Facts](DAY%201%20-%20Ansible%20Inventory%20Ad-hoc%20Commands%20and%20Facts.md)
- [DAY 2 — Variables and Precedence](DAY%202%20-%20Ansible%20Variables%20and%20Precedence.md)
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

## Suggested practice sequence

1. Inspect inventory with `ansible-inventory --graph`.
2. Preview targets with `ansible ... --list-hosts`.
3. Test connectivity with `ansible ... -m ansible.builtin.ping`.
4. Gather facts with `ansible ... -m ansible.builtin.setup`.
5. Run the DAY 1 and DAY 2 playbooks twice and compare recap output.
6. Trace a variable override and trigger the DAY 2 deliberate undefined-variable failure.

Use a disposable lab. The DAY 2 example uses a local connection and writes only under `/tmp`. Never commit private keys or plaintext secrets.
