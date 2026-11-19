# Ansible

Ansible is an agentless automation tool commonly used for configuration management, application deployment and operational tasks.

## Learning Structure

- [DAY 1 — Inventory, Ad-hoc Commands and Facts](DAY%201%20-%20Ansible%20Inventory%20Ad-hoc%20Commands%20and%20Facts.md)
- [Hands-on Labs](LABS.md)
- [Interview Questions](INTERVIEW%20QUESTIONS.md)

## DAY 1 — Learning Focus

- Inventory files, host groups, host variables and host patterns
- Ad-hoc commands and common modules
- Fact gathering with the setup module
- Minimal inventory + playbook example
- Run a playbook twice and verify idempotency through the changed count
- Diagnose an intentional remote command failure using `-vvv`

## Suggested practice sequence

1. Inspect inventory with `ansible-inventory --graph`.
2. Preview targets with `ansible ... --list-hosts`.
3. Test connectivity with `ansible ... -m ansible.builtin.ping`.
4. Gather facts with `ansible ... -m ansible.builtin.setup`.
5. Run the DAY 1 playbook twice and compare recap output.
6. Trigger the deliberate missing-path failure and inspect verbose output.

Use a disposable lab and replace documentation-only IP addresses in examples with reachable hosts. Never commit private keys or plaintext secrets.
