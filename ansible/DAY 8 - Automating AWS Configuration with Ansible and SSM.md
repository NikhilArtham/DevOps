# DAY 8 — Automating AWS Configuration with Ansible + SSM

## 1. What problem does Ansible + SSM solve?

Traditional Ansible over SSH generally needs network reachability from the controller to the managed EC2 instance on TCP 22. Private instances often have no public IP and security policies may prohibit inbound SSH.

With AWS Systems Manager Session Manager, Ansible can use an SSM session as its connection transport. The controller asks Systems Manager to start a session, and the SSM Agent on the instance maintains outbound communication with Systems Manager.

**Mental model:**

- Ansible: configuration management and task orchestration.
- AWS Systems Manager: managed-node connectivity and session transport.
- SSM Agent: agent running on the EC2 instance.
- IAM instance profile: allows the instance to register with and communicate with Systems Manager.
- S3 staging bucket: transfers Ansible module files and data between controller and instance for the `amazon.aws.aws_ssm` connection plugin.

This is not the same as the Ansible `amazon.aws.ssm_parameter` lookup or the AWS SSM Run Command module. Today we are using the **connection plugin** to run normal Ansible tasks on an EC2 instance.

## 2. Architecture and prerequisites

```text
Ansible controller
  |-- AWS identity: start/manage SSM session
  |-- Session Manager Plugin installed
  |-- amazon.aws collection + boto3/botocore
  |-- S3 permissions for staging bucket
  |
  +---- AWS Systems Manager API ----> SSM service
  |                                    ^
  |                                    |
  +---- S3 staging bucket              | outbound HTTPS
        ^                              |
        |                              |
        +---- EC2 instance <---- SSM Agent
                 |
                 +-- IAM instance profile
                 +-- outbound HTTPS to SSM endpoints
                 +-- curl for S3 transfers
```

### EC2 managed-node prerequisites

1. EC2 instance is running a supported OS and has a working SSM Agent.
2. Instance profile grants Systems Manager managed-node permissions. The AWS-managed `AmazonSSMManagedInstanceCore` policy is a common baseline.
3. SSM Agent can reach the regional Systems Manager endpoints over outbound TCP 443.
4. For a private subnet, provide NAT egress or suitable Systems Manager interface VPC endpoints. Depending on Region/agent compatibility, endpoints include `ssm`, `ssmmessages`, and sometimes `ec2messages`.
5. The instance needs network access to the S3 staging bucket and `curl` installed for Linux file transfers.

### Ansible controller prerequisites

1. Install the collection: `ansible-galaxy collection install amazon.aws`.
2. Install the AWS Session Manager Plugin binary on the controller.
3. Install compatible AWS SDK dependencies such as boto3 and botocore as required by the collection.
4. Configure AWS credentials using an approved identity source, such as an AWS profile, environment credentials or an assumed role.
5. The controller needs permissions to start/manage SSM sessions and upload, download, list and delete objects in the configured staging bucket.

Check the installed collection and plugin:

```bash
ansible-galaxy collection list | grep amazon.aws
ansible-doc amazon.aws.aws_ssm
aws sts get-caller-identity
session-manager-plugin --version
```

The exact IAM permissions should be scoped to the intended instance, account, Region, S3 bucket and object prefix. Do not use administrator permissions as a shortcut.

## 3. Important S3 staging behavior and security

The `amazon.aws.aws_ssm` connection plugin uses S3 to transfer Ansible module files and data. This is required even for tasks such as `command` or `shell`, because Ansible must transfer module code to the target.

- The controller creates presigned S3 URLs; the instance uses `curl` to transfer files.
- The controller requires bucket/object permissions including `s3:GetObject`, `s3:PutObject`, `s3:ListBucket`, `s3:DeleteObject` and `s3:GetBucketLocation` as applicable.
- The bucket and instance should be in compatible Regions; use a dedicated bucket/prefix and least-privilege policies.
- The plugin attempts to delete transferred objects after a playbook run, but an interrupted run may leave objects behind.
- If bucket versioning is enabled, deleted objects can remain in versions.
- **Do not pass secrets through tasks using this transport unless you have assessed the S3 exposure path.** The plugin documentation warns that secret-bearing transferred files may remain in S3 version history in plaintext. Prefer avoiding secret transfer through this mechanism; use a dedicated protected bucket, encryption, tight access, lifecycle controls and no versioning where appropriate.

Ansible Vault and `no_log` do not automatically make the plugin's S3 staging path safe. Never use a real password or production token in this lab.

## 4. Minimal static inventory

Create this layout:

```text
ansible-ssm-lab/
├── inventory.ini
└── site.yml
```

### inventory.ini

Replace the example values with a disposable EC2 instance ID, Region and dedicated S3 bucket name:

```ini
[ssm_targets]
i-0123456789abcdef0 ansible_connection=amazon.aws.aws_ssm ansible_aws_ssm_region=ap-south-1 ansible_aws_ssm_bucket_name=my-dedicated-ansible-ssm-transfer-bucket
```

The inventory hostname is the EC2 instance ID. The connection plugin uses it to identify the instance. The example Region is Mumbai; change it to the instance's actual Region. Do not commit account credentials or access keys in this file.

You can use a named AWS profile through your controller environment:

```bash
export AWS_PROFILE=devops-lab
export AWS_REGION=ap-south-1
```

The profile must resolve to an identity with the required SSM and S3 permissions. Prefer short-lived assumed-role credentials over long-lived IAM user keys.

## 5. Smallest useful playbook

### site.yml

```yaml
---
- name: Configure a disposable EC2 instance over AWS SSM
  hosts: ssm_targets
  gather_facts: false

  tasks:
    - name: Create the lab configuration directory
      ansible.builtin.file:
        path: /tmp/ansible-ssm-lab
        state: directory
        mode: "0755"

    - name: Deploy the lab configuration file
      ansible.builtin.copy:
        content: |
          managed_by=ansible
          transport=aws_ssm
          environment=learning
        dest: /tmp/ansible-ssm-lab/agent.conf
        mode: "0644"

    - name: Verify the configuration file exists
      ansible.builtin.stat:
        path: /tmp/ansible-ssm-lab/agent.conf
      register: agent_config

    - name: Assert that configuration was deployed
      ansible.builtin.assert:
        that:
          - agent_config.stat.exists
        fail_msg: "SSM configuration file was not deployed"
        success_msg: "SSM configuration file is present"

    - name: Deliberately check for a missing file
      ansible.builtin.command:
        argv:
          - /usr/bin/test
          - -f
          - /tmp/ansible-ssm-lab/missing.conf
      register: deliberate_check
      changed_when: false
      tags:
        - deliberate_failure
```

No SSH key or inbound TCP 22 rule is used by this inventory. The instance still needs outbound access to Systems Manager and S3. The lab writes only under `/tmp` and does not install packages or change system services.

## 6. Run the playbook

From the controller, inside the lab directory:

```bash
# Confirm the target before connecting
ansible-playbook -i inventory.ini site.yml --list-hosts

# Validate playbook structure
ansible-playbook -i inventory.ini site.yml --syntax-check

# Run normal tasks; skip the intentionally failing tagged task
ansible-playbook -i inventory.ini site.yml --skip-tags deliberate_failure

# Run it again to verify idempotency
ansible-playbook -i inventory.ini site.yml --skip-tags deliberate_failure
```

Expected result:

- First run: directory and file are created; assert passes.
- Second run: file content and mode are already correct; recap should show `changed=0`.
- The SSM connection must be established for the tasks to execute.

Verify the file on the EC2 instance through a safe Ansible task or an authorized SSM session. Do not assume that a successful Ansible recap proves application health.

## 7. Diagnose one deliberate failure with -vvv

Run only the tagged missing-file check:

```bash
ansible-playbook -i inventory.ini site.yml --tags deliberate_failure -vvv
```

Expected diagnosis:

- If the SSM session opens successfully, the task runs on the selected EC2 instance.
- `/usr/bin/test -f /tmp/ansible-ssm-lab/missing.conf` returns non-zero (normally `rc=1`) because the file does not exist.
- This is an intentional command failure, not an SSM connectivity failure.
- If the task instead fails before execution with a connection error, troubleshoot SSM prerequisites before interpreting the command result.

Keep the deliberate failure out of normal automation. The purpose is to learn how to separate transport failures from task-level failures.

## 8. SSM troubleshooting decision tree

| Symptom | Likely cause | What to check |
|---|---|---|
| Instance absent from Systems Manager managed nodes | Agent, instance role or network | SSM Agent service/logs, instance profile, outbound 443, SSM endpoints |
| Session Manager works in console but Ansible fails | Controller setup or plugin | Collection, Session Manager Plugin, AWS identity, Python SDK dependencies |
| AccessDenied on StartSession | Controller IAM policy | Caller identity and scoped SSM session permissions |
| S3 AccessDenied | Controller S3 policy/bucket policy/KMS | Bucket and prefix permissions, encryption key policy, Region |
| S3 transfer timeout | Network or endpoint | Controller and instance S3 access, S3 gateway endpoint/NAT, DNS and route |
| SSM session opens but module fails | Target runtime | Python availability, module compatibility, permissions, disk space |
| Permission denied writing target path | Remote OS user | Actual SSM session user, file ownership and narrowly scoped become settings |
| Playbook works once but leaves S3 objects | Interrupted run or versioning | Inspect the dedicated bucket/prefix, object versions and lifecycle cleanup |

Useful instance-side checks (run through an existing authorized session):

```bash
sudo systemctl status amazon-ssm-agent
sudo journalctl -u amazon-ssm-agent --since "30 minutes ago"
sudo tail -n 100 /var/log/amazon/ssm/amazon-ssm-agent.log
command -v curl
python3 --version
```

Useful controller-side checks:

```bash
aws sts get-caller-identity
aws ssm describe-instance-information --region ap-south-1
ansible-doc amazon.aws.aws_ssm
ansible-playbook -i inventory.ini site.yml -vvv
```

Redact account IDs, instance IDs, usernames, bucket names and any sensitive values before sharing verbose logs.

## 9. Production design considerations

- Use EC2 instance profiles and short-lived controller credentials; do not distribute long-lived access keys.
- Prefer private subnets with VPC endpoints when the architecture requires no NAT path, and validate DNS, endpoint policies, security groups and routes.
- Use a dedicated S3 staging bucket with restrictive bucket policy, encryption, public-access blocking and lifecycle controls.
- Understand the plugin's file-transfer and object-versioning behavior before sending any sensitive data.
- Pin and test the `amazon.aws` collection version in the automation environment.
- Use dynamic EC2 inventory for fleets, filtering by account, Region and tags; use static inventory for a small controlled lab.
- Use `become` only when the task needs elevated OS privileges. The SSM plugin does not use `ansible_user`/`remote_user` in the same way as SSH; consult its documentation for run-as behavior and use `become_user` where appropriate.
- Add post-deployment health checks; SSM connectivity only proves the transport works.

## 10. Interview-ready mental model

**Controller identity starts an SSM session → SSM Agent on EC2 maintains the managed-node channel → Ansible's SSM connection plugin uses S3 to stage module files → Ansible modules converge the EC2 operating-system configuration.**

SSM removes the need for inbound SSH, but it does not remove the need for IAM, network connectivity, S3 transfer security or operating-system permissions.
