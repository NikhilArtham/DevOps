# DAY 5 — Ansible Roles and Reusable Structure

## 1. Why roles?

A role packages related automation into a predictable directory structure. It makes playbooks smaller, encourages reuse and separates responsibilities. Instead of maintaining one large playbook for installation, configuration, users and handlers, build focused roles and compose them from a site playbook.

## 2. Standard role directory structure

~~~
roles/
└── app_config/
    ├── defaults/main.yml
    ├── files/
    ├── handlers/main.yml
    ├── meta/main.yml
    ├── tasks/main.yml
    ├── templates/
    ├── tests/
    └── vars/main.yml
~~~

| Directory | Purpose |
|---|---|
| defaults/main.yml | Low-precedence defaults designed to be overridden |
| tasks/main.yml | Main task entry point |
| handlers/main.yml | Event-driven tasks notified by changed tasks |
| templates/ | Jinja2 templates rendered by the role |
| files/ | Static files copied by the role |
| vars/main.yml | Higher-precedence role variables |
| meta/main.yml | Metadata and dependencies |
| tests/ | Optional test inventory and playbook |

Not every role needs every directory. Start with only what the role uses.

## 3. Generate a role skeleton

From the project root:

~~~
mkdir -p roles
cd roles
ansible-galaxy role init app_config
cd ..
~~~

This generates a standard role layout. For a small role, create only the required directories manually:

~~~
mkdir -p roles/app_config/{defaults,tasks,handlers,templates}
~~~

Do not run both methods if the role directory already exists.

## 4. Minimal reusable role example

Project layout:

~~~
ansible-role-lab/
├── inventory.ini
├── site.yml
└── roles/
    └── app_config/
        ├── defaults/main.yml
        ├── tasks/main.yml
        ├── handlers/main.yml
        └── templates/app.conf.j2
~~~

### roles/app_config/defaults/main.yml

~~~
---
app_name: demo-app
app_environment: lab
app_port: 8080
app_config_dir: /tmp/ansible-role-lab
~~~

### roles/app_config/templates/app.conf.j2

~~~
application={{ app_name }}
environment={{ app_environment }}
port={{ app_port }}
~~~

### roles/app_config/tasks/main.yml

~~~
---
- name: Ensure application configuration directory exists
  ansible.builtin.file:
    path: "{{ app_config_dir }}"
    state: directory
    mode: "0750"

- name: Render application configuration
  ansible.builtin.template:
    src: app.conf.j2
    dest: "{{ app_config_dir }}/app.conf"
    mode: "0640"
  notify: Report application configuration updated
~~~

Ansible resolves a role's template from its templates directory; no project-relative path is needed.

### roles/app_config/handlers/main.yml

~~~
---
- name: Report application configuration updated
  ansible.builtin.debug:
    msg: "Configuration changed; a production role could reload its service here."
~~~

This debug handler is intentionally safe for a local exercise. In a real service role, use a service reload/restart handler after validating the configuration.

### site.yml

~~~
---
- name: Configure application using a reusable role
  hosts: local
  gather_facts: false
  become: false

  roles:
    - role: app_config
      vars:
        app_environment: development
        app_port: 9090

  tasks:
    - name: Deliberate undefined-variable failure
      ansible.builtin.debug:
        msg: "This task is intentionally broken"
      when: missing_app_version == "1.0"
      tags:
        - deliberate_failure
~~~

### inventory.ini

~~~
[local]
localhost ansible_connection=local
~~~

Role defaults define baseline values. The playbook overrides environment and port for this invocation. The same role can be reused with different inputs without editing its tasks or template.

## 5. Run and verify idempotency

Run from the project root, where inventory.ini and site.yml are located:

~~~
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --list-hosts
ansible-playbook -i inventory.ini site.yml
ansible-playbook -i inventory.ini site.yml
~~~

Expected behavior:
- First run creates the directory and renders app.conf.
- The template task reports changed and notifies the handler.
- The handler runs after normal tasks.
- Second run with unchanged inputs should report changed=0; the handler should not run.

Inspect the file:

~~~
cat /tmp/ansible-role-lab/app.conf
stat /tmp/ansible-role-lab/app.conf
~~~

Expected content:

~~~
application=demo-app
environment=development
port=9090
~~~

Change app_port in the role invocation to 9191. The template should change once. Run again without further changes; it should report changed=0.

## 6. Diagnose a deliberate failure with -vvv

Run only the tagged failure task:

~~~
ansible-playbook -i inventory.ini site.yml --tags deliberate_failure -vvv
~~~

The task fails while Ansible evaluates the when expression because missing_app_version is undefined. This is a variable/condition failure, not an inventory or SSH failure. Verbose output helps identify the host, task and evaluation context.

For practice, define the variable or correct the expression, rerun the tagged task, then remove the deliberate-failure task. Do not leave intentional failures in production playbooks.

## 7. Ways to invoke roles

Static role inclusion under roles:

~~~
roles:
  - app_config
~~~

Static task inclusion with import_role:

~~~
tasks:
  - name: Apply application configuration role
    ansible.builtin.import_role:
      name: app_config
~~~

import_role is static: Ansible processes the role during playbook parsing.

Dynamic task inclusion with include_role:

~~~
tasks:
  - name: Apply role when deployment is requested
    ansible.builtin.include_role:
      name: app_config
    when: deploy_application | bool
~~~

include_role is dynamic: Ansible loads the role during execution. It is useful for runtime conditions and loops. Tag and condition inheritance can differ between static imports and dynamic includes, so test the task-selection behavior you intend.

## 8. Role design and production practices

1. Keep each role focused on one responsibility.
2. Put user-overridable values in defaults/main.yml.
3. Use vars/main.yml only for values that should be harder to override.
4. Keep templates and static files inside the role.
5. Use fully qualified module names such as ansible.builtin.template.
6. Validate inputs with ansible.builtin.assert when invalid values could cause unsafe changes.
7. Validate generated service configuration before notifying a reload handler.
8. Keep secrets in Vault or an approved secrets manager; do not print them.
9. Document role inputs, defaults, supported platforms and example usage.
10. Avoid hidden assumptions about package manager, OS, service name, paths and privilege escalation.

| Common mistake | Better approach |
|---|---|
| One role contains unrelated application and infrastructure tasks | Split by responsibility |
| Environment values are hardcoded in tasks | Put configurable values in defaults and override them |
| Static files are placed in templates | Use files/ and copy |
| Template path depends on current directory | Keep it in the role templates/ directory |
| Handler never runs | Check changed status and matching notify name |
| Role works only from one working directory | Use conventional project layout and role search paths |
| Service restarts every run | Notify only from state-aware tasks that report changed |

## 9. Quick revision

- Role = reusable structure of related tasks, handlers, templates, files and variables.
- defaults/main.yml = overridable baseline values.
- tasks/main.yml = role task entry point.
- handlers/main.yml = event-driven actions.
- templates/ = Jinja2 source; files/ = static files.
- roles: and import_role are static patterns; include_role is dynamic.
- Run twice with unchanged inputs; verify the second run reports changed=0.
