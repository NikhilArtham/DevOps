# DAY 6 — Ansible Vault and Secret Handling

## 1. Why secret handling matters

Automation commonly needs database passwords, API tokens, private keys, service credentials and certificates. A secret can leak through more places than the source YAML:

- A plaintext variable committed to Git.
- Shell history or process arguments when a password is passed with `-e`.
- Task output, debug statements, callback plugins or CI logs.
- Generated configuration files with broad permissions.
- Backups, artifacts, cached facts or temporary files.
- Verbose troubleshooting output copied into tickets.

**Rule:** encrypt secrets at rest, restrict who can decrypt them, minimize where they are exposed at runtime, and prevent task output from logging them.

## 2. Ansible Vault mental model

Ansible Vault encrypts files or individual strings so the secret is not readable in the repository. Ansible decrypts the data when a playbook needs it and an authorized vault password is supplied.

Vault does not:
- Automatically encrypt every variable in a playbook.
- Stop a task from printing a decrypted value.
- Secure a secret after it has been written to a target host.
- Replace access control for the repository, CI system or vault password.
- Prevent someone who has both the encrypted file and vault password from reading the secret.

Think of Vault as **encryption at rest**, not complete end-to-end secret management.

## 3. Smallest working example

Use a local inventory so the lab does not need an SSH host.

Project layout:

```text
ansible-vault-lab/
├── inventory.ini
├── site.yml
└── group_vars/
    └── all/
        └── vault.yml
```

Create the files:

```bash
mkdir -p ansible-vault-lab/group_vars/all
cd ansible-vault-lab

cat > inventory.ini <<'EOF'
[local]
localhost ansible_connection=local
EOF

cat > group_vars/all/vault.yml <<'EOF'
vault_demo_password: "not-a-real-password"
EOF
```

The value above is intentionally fake. Never use a real credential for a learning exercise.

Encrypt the variable file:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

Ansible prompts for a new Vault password and confirmation. Choose a temporary lab password that is not used anywhere else. After encryption, inspect the file:

```bash
head -n 1 group_vars/all/vault.yml
ansible-vault view group_vars/all/vault.yml
```

The first command should show an Ansible Vault header, not the YAML value. The second decrypts and displays the content, so use it only in a safe terminal and do not copy its output into logs or screenshots.

## 4. Playbook: use the encrypted variable safely

Create `site.yml`:

```yaml
---
- name: Demonstrate Ansible Vault
  hosts: local
  gather_facts: false

  tasks:
    - name: Create a private demo file from the vaulted value
      ansible.builtin.copy:
        dest: /tmp/ansible-vault-lab-secret.txt
        content: "{{ vault_demo_password }}\n"
        owner: "{{ lookup('env', 'USER') }}"
        mode: "0600"
      no_log: true
```

Run a syntax check and execute with an interactive Vault prompt:

```bash
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

The task writes a deliberately fake value to a local file with owner-only read/write permissions. `no_log: true` hides the task arguments and result because they contain the decrypted value. It does not encrypt the destination file; inspect only its permissions in this exercise, not its contents.

Check permissions without printing the secret:

```bash
stat -c '%a %U %n' /tmp/ansible-vault-lab-secret.txt
```

Expected mode: `600`.

## 5. Verify idempotency

Run the same playbook twice with the same inputs:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

Expected behavior:

- First run: the file is created; the task reports changed (the result is censored because of `no_log`).
- Second run: the file already has the desired content and permissions; the task should report no change.
- The recap should show `changed=0` on the second run.

The task is idempotent because the copy module compares the desired content and metadata with the existing file. Do not remove `no_log` just to see the secret-bearing task result.

## 6. Deliberate failure and verbose diagnosis

Add this second task after the copy task:

```yaml
    - name: Deliberately fail when a required vaulted variable is missing
      ansible.builtin.assert:
        that:
          - vault_missing_demo_value is defined
        fail_msg: "The required demo variable is not defined."
      tags:
        - deliberate_failure
```

Run only the deliberate task:

```bash
ansible-playbook -i inventory.ini site.yml --ask-vault-pass --tags deliberate_failure -vvv
```

What to inspect:

1. Confirm Ansible selected the local host and reached the assert task.
2. Read the task name and undefined-variable/failed assertion message.
3. Distinguish this from an inventory, SSH or Vault decryption failure.
4. Fix the variable name or define the required value, then rerun the tagged task.
5. Remove the deliberate failure task after practising.

**Important:** `-vvv` is useful for task, connection and execution context, but verbose logs are not automatically safe. Never paste unredacted output into tickets or chat. `no_log` is not a reason to assume every other task or plugin is safe.

## 7. Useful Vault commands

| Command | Purpose |
|---|---|
| `ansible-vault encrypt file.yml` | Encrypt an existing file |
| `ansible-vault decrypt file.yml` | Decrypt a file in place |
| `ansible-vault view file.yml` | Display decrypted content |
| `ansible-vault edit file.yml` | Edit encrypted content safely through Vault |
| `ansible-vault rekey file.yml` | Change the Vault password protecting a file |
| `ansible-vault encrypt_string 'value' --name 'variable_name'` | Encrypt one string for embedding in YAML |
| `ansible-playbook site.yml --ask-vault-pass` | Prompt for the Vault password |
| `ansible-playbook site.yml --vault-password-file .vault_pass` | Read password from a file (protect it carefully) |
| `ansible-playbook site.yml --vault-id dev@prompt` | Select a named Vault ID and prompt for its password |

Use `ansible-vault edit` to modify encrypted files; avoid decrypting them in place for routine edits. If you decrypt a file, re-encrypt it before staging or committing.

## 8. Vault IDs and multiple environments

A team may use different Vault passwords for development, QA and production. Vault IDs label which password should be used:

```bash
ansible-vault encrypt --vault-id dev@prompt group_vars/dev/vault.yml
ansible-vault encrypt --vault-id qa@prompt group_vars/qa/vault.yml
ansible-vault encrypt --vault-id prod@prompt group_vars/prod/vault.yml
ansible-playbook -i inventory.ini site.yml --vault-id dev@prompt
```

A Vault ID is a label, not a permission boundary. Authorization still depends on who can access the encrypted file and obtain the matching password. Production credentials should have stricter access controls and a separate rotation process.

## 9. Vault password files and CI/CD

Interactive prompts are useful for local learning. In CI/CD, do not hardcode the Vault password in the repository, playbook, Docker image or pipeline YAML.

Preferred production patterns:

- Retrieve the Vault password at runtime from an approved secret manager or protected CI secret store.
- Restrict which pipelines, branches and identities can access production secrets.
- Avoid printing secret variables or enabling shell tracing around secret-handling commands.
- Keep password files outside the repository with restrictive permissions (for example, `chmod 600`); ensure they are not included in artifacts or backups.
- Rotate credentials and Vault passwords through a documented process.
- Separate development, QA and production secret scopes.
- Audit access and keep the encrypted file's repository permissions appropriately restricted.

A local password file can be convenient, but it becomes another sensitive asset. Never commit it. Prefer a prompt for this lab.

## 10. `no_log` and safe output

Use `no_log: true` on tasks that handle sensitive arguments or return values, such as creating a user with a password, calling an API with a token, or rendering a secret-bearing configuration. Example:

```yaml
- name: Create application account
  ansible.builtin.user:
    name: appsvc
    password: "{{ app_password_hash }}"
  no_log: true
```

Important distinctions:

- Vault protects encrypted source data at rest.
- `no_log` suppresses task details in normal Ansible output.
- File permissions protect a secret-bearing file on the managed host.
- CI secret masking is an additional platform control, not a substitute for careful logging.
- Avoid `debug` on secret variables. Do not use `-vvv` as a way to reveal secret values.

## 11. Common mistakes and troubleshooting

- **Vault password prompt fails:** confirm the password, Vault ID and encrypted file are paired correctly. Do not repeatedly expose the password while debugging.
- **Decryption error:** verify the file is actually Vault-encrypted and that the selected password/Vault ID is correct.
- **Variable is undefined:** confirm the encrypted file is in the expected `group_vars`/ `host_vars` location or explicitly loaded with `vars_files`; check YAML indentation and variable spelling.
- **Secret appears in output:** identify the task that printed it, remove debug output, add `no_log: true` where appropriate, and review callback/CI logs already produced.
- **Secret file is world-readable:** set restrictive ownership and mode, and review the destination path and backups.
- **Secret changed unexpectedly:** check which inventory/group file is loaded and whether environment-specific variables are overriding the intended value.
- **A password file was committed:** treat the password as compromised; rotate it and review repository history and access. Deleting the latest file alone does not erase Git history.

## 12. Production checklist

- [ ] No plaintext credentials in Git.
- [ ] Vault-encrypted files are committed only where repository access is appropriate.
- [ ] Vault passwords are stored separately in an approved secret store.
- [ ] Dev, QA and production secrets are separated.
- [ ] Sensitive tasks use `no_log: true`.
- [ ] Secret-bearing target files use restrictive ownership and permissions.
- [ ] No debug statements print secret values.
- [ ] CI logs, artifacts and shell tracing are reviewed for exposure.
- [ ] Secret and Vault-password rotation is documented.
- [ ] A second unchanged run confirms idempotency.

## Key takeaway

**Vault encrypts the source; `no_log` reduces output leakage; file permissions protect the deployed copy; access controls and rotation protect the overall lifecycle.** No single one of these controls is sufficient by itself.
