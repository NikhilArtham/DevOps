# DAY 6 — EC2 Boot, User-Data, Metadata and System Troubleshooting

## 1. Why This Topic Matters

When an EC2 instance is "not working," the failure may be at several different layers:

```text
AWS instance state
↓
EC2 boot / OS startup
↓
cloud-init / user-data
↓
Network configuration
↓
SSH / SSM access
↓
Application startup
```

A strong DevOps engineer does not immediately reboot or recreate the instance. The goal is to identify the first layer that failed.

---

## 2. EC2 Boot Mental Model

A simplified EC2 startup flow is:

```text
EC2 launched
    ↓
Instance reaches running state
    ↓
Virtual hardware becomes available
    ↓
Bootloader starts
    ↓
Linux kernel starts
    ↓
systemd starts
    ↓
cloud-init / OS initialization runs
    ↓
User-data commands run as configured
    ↓
Network/services start
    ↓
Application starts
```

Important point:

> `running` in the EC2 console does not necessarily mean the application is ready.

An instance can be `running` while:

- cloud-init is still executing
- user-data failed
- a systemd service failed
- the application is not listening
- SSH is unavailable
- SSM is unavailable
- the instance has a network problem

---

## 3. EC2 Instance State vs System Health

EC2 exposes instance and system status checks.

Common status-check concepts:

```text
System status check
Instance status check
```

Think of them as different layers:

```text
AWS infrastructure / underlying system
        ↓
EC2 instance / OS
```

Useful AWS CLI command:

```bash
aws ec2 describe-instance-status \
  --instance-ids i-xxxxxxxx
```

For a broader inspection:

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx
```

When an instance is inaccessible, first determine whether AWS itself reports a status-check problem before immediately investigating application code.

---

## 4. User-Data — What Is It?

EC2 user-data is initialization data supplied when an instance is launched.

A common Linux example is a shell script:

```bash
#!/bin/bash

yum update -y

yum install -y nginx
systemctl enable nginx
systemctl start nginx
```

The intention is:

```text
EC2 launches
↓
cloud-init processes user-data
↓
packages/configuration are applied
↓
service starts
```

User-data is commonly used for:

- package installation
- configuration bootstrap
- service startup
- pulling application artifacts
- installing agents
- registering software
- initial configuration

For production, keep bootstrap scripts observable, idempotent where possible, and small enough to troubleshoot.

---

## 5. User-Data Is Not the Same as a Permanent Configuration Manager

A common beginner mistake is treating user-data as if it were a continuously running configuration system.

A useful mental model is:

```text
User-data
= bootstrap / initialization
```

Whereas tools such as:

```text
Ansible
Terraform
SSM
configuration management systems
```

serve different operational purposes.

If an application configuration must be maintained throughout the instance lifecycle, relying only on a one-time bootstrap script can make troubleshooting and drift management harder.

---

## 6. User-Data Execution and cloud-init

On many Linux AMIs, cloud-init handles instance initialization.

Useful files/commands include:

```bash
cloud-init status
```

```bash
cloud-init status --long
```

Inspect cloud-init logs:

```bash
sudo less /var/log/cloud-init.log
```

```bash
sudo less /var/log/cloud-init-output.log
```

Depending on the AMI, useful directories can include:

```text
/var/lib/cloud/
```

The exact behavior depends on the operating system and AMI, so do not assume every Linux image has identical cloud-init configuration.

---

## 7. How to Troubleshoot Failed User-Data

Suppose an instance launches but nginx is not running.

Do not immediately conclude that nginx is broken.

Check in this order:

```text
1. Is the EC2 instance running?
2. Are EC2 status checks healthy?
3. Did cloud-init run?
4. Did user-data execute successfully?
5. Was the package installed?
6. Did systemd start the service?
7. Is the service listening?
8. Is the network path reachable?
```

Commands:

```bash
cloud-init status --long
```

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
```

```bash
sudo systemctl status nginx
```

```bash
sudo ss -lntp
```

This separates bootstrap failure from application failure.

---

## 8. EC2 Instance Metadata

EC2 instance metadata provides information about the running instance from inside the instance.

Modern EC2 environments should use **IMDSv2** where possible.

IMDSv2 uses a session token.

Request a token:

```bash
TOKEN=$(curl -X PUT \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" \
  http://169.254.169.254/latest/api/token)
```

Then query metadata:

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/
```

For the instance ID:

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

For the local IPv4 address:

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4
```

---

## 9. Why Metadata Is Useful During Troubleshooting

Metadata can answer questions from inside the instance without requiring you to remember or manually enter the instance details.

For example:

```text
Which instance am I on?
Which private IP does AWS report?
Which instance identity information is available?
```

This is useful in:

- automation scripts
- bootstrap scripts
- troubleshooting
- identifying instances from inside the OS
- retrieving instance-related information

The metadata service is available through the link-local address:

```text
169.254.169.254
```

---

## 10. Metadata Security — Why IMDSv2 Matters

A major security concern is unintended access to instance metadata from applications or processes running on the instance.

IMDSv2 adds a session-oriented token requirement.

The high-level flow is:

```text
Client
 ↓
PUT metadata token
 ↓
Receive token
 ↓
GET metadata with token
 ↓
Receive metadata
```

For security-sensitive workloads, understand and enforce the intended metadata-options configuration rather than treating metadata access as an invisible implementation detail.

AWS CLI inspection:

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx \
  --query 'Reservations[].Instances[].MetadataOptions'
```

Look for settings related to:

```text
HttpTokens
HttpEndpoint
HttpPutResponseHopLimit
```

---

## 11. Instance Identity Document

The metadata service can expose an instance identity document.

Example:

```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/dynamic/instance-identity/document
```

This can provide information such as:

```text
instanceId
region
availabilityZone
instanceType
accountId
```

It is useful when scripts or diagnostics need to identify the environment dynamically.

---

## 12. EC2 Access Path: SSH

A common Linux access path is:

```text
Laptop
 ↓
DNS / public IP / bastion / VPN
 ↓
Network path
 ↓
Security Group
 ↓
NACL
 ↓
EC2 network interface
 ↓
sshd
 ↓
Linux user
```

For SSH troubleshooting, separate:

```text
DNS failure
↓
route failure
↓
port 22 blocked
↓
sshd not listening
↓
authentication failure
```

Useful checks on the instance:

```bash
sudo systemctl status sshd
```

```bash
sudo ss -lntp | grep ':22'
```

From a client:

```bash
nc -vz <destination-ip> 22
```

Then:

```bash
ssh -v ec2-user@<destination>
```

The `-v` output can help separate TCP connection failure from SSH authentication or negotiation problems.

---

## 13. SSM as an Alternative Access Path

AWS Systems Manager Session Manager can provide shell access without requiring the normal inbound SSH path.

High-level path:

```text
Operator
 ↓
AWS Systems Manager
 ↓
SSM Agent
 ↓
EC2 instance
```

This can be valuable when:

```text
SSH is blocked
No public IP exists
The instance is private
```

But SSM is not magic. The instance needs the required Systems Manager configuration, including an operational SSM Agent and appropriate IAM/network connectivity for the environment.

AWS CLI example:

```bash
aws ssm describe-instance-information
```

A common troubleshooting question is:

> Is the EC2 instance broken, or is only my SSH path broken?

SSM can help answer that question when configured correctly.

---

## 14. System Logs and Boot Troubleshooting

If you can access the instance, inspect systemd and kernel-related information.

Useful commands:

```bash
systemctl --failed
```

```bash
systemctl status <service>
```

```bash
journalctl -b
```

```bash
journalctl -u <service>
```

For recent errors:

```bash
journalctl -p err -b
```

Kernel messages:

```bash
dmesg | tail -n 100
```

Use these to answer:

```text
Did the system boot?
Did systemd start the service?
Did the service crash?
Was there a filesystem/network/kernel error?
```

---

## 15. AWS Console System Log / Serial Console Concepts

When normal access is unavailable, AWS-side diagnostic options can be important.

Depending on the instance and configuration, investigate available EC2 diagnostic mechanisms such as:

- system status checks
- instance status checks
- console output
- EC2 Serial Console
- Systems Manager

The important troubleshooting principle is:

```text
No SSH
≠
Instance is definitely dead
```

The problem may exist in the network path, SSH service, authentication, or OS boot process.

---

## 16. Realistic Failure Scenario — User-Data Broke Application Startup

### Architecture

```text
Route 53
   ↓
ALB :443
   ↓
Target Group
   ↓
EC2
   ↓
Application :8080
```

### Incident

A new EC2 instance is launched as part of an Auto Scaling deployment.

The EC2 console shows:

```text
Instance state = running
```

But the ALB reports:

```text
Target = unhealthy
```

Users receive:

```text
503 Service Unavailable
```

### Investigation

First inspect target health:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

Then access the instance using SSM if available.

Check:

```bash
cloud-init status --long
```

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
```

Then:

```bash
sudo systemctl status myapp
```

```bash
sudo ss -lntp
```

### Root cause

The user-data script contains:

```bash
systemctl start myapp
```

but the application package installation failed earlier in the script. The script continued in a way that did not produce the expected running service.

Result:

```text
EC2 = running
↓
ALB = reachable
↓
Target health check = fails
↓
Application = not listening on 8080
↓
ALB = 503
```

### Correct troubleshooting conclusion

The failure is **not automatically a Route 53, ALB, or Security Group problem**.

The first broken layer is:

```text
EC2 bootstrap / application startup
```

Fix the bootstrap/application installation, validate the service, confirm the port is listening, and then verify ALB target health.

---

## 17. EC2 Troubleshooting Decision Tree

Use this sequence:

```text
EC2 failure
   ↓
Is instance running?
   ├── NO → investigate EC2 lifecycle/start failure
   └── YES
        ↓
Are status checks healthy?
   ├── NO → investigate AWS/system health
   └── YES
        ↓
Can you access through SSM?
   ├── YES → inspect OS/bootstrap/services
   └── NO
        ↓
Can SSH connect?
   ├── NO → DNS → route → SG/NACL → port 22 → sshd
   └── YES → inspect OS/application
        ↓
Did cloud-init/user-data succeed?
   ├── NO → inspect cloud-init logs
   └── YES
        ↓
Is the service running?
   ├── NO → systemd/journal/application logs
   └── YES
        ↓
Is the expected port listening?
   ├── NO → application configuration/startup
   └── YES
        ↓
Can the load balancer/client reach it?
   ├── NO → SG/NACL/routing
   └── YES → inspect TLS/HTTP/application behavior
```

---

## 18. Useful AWS CLI Commands

### EC2 state and status

```bash
aws ec2 describe-instances --instance-ids i-xxxxxxxx
```

```bash
aws ec2 describe-instance-status --instance-ids i-xxxxxxxx
```

### Metadata options

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx \
  --query 'Reservations[].Instances[].MetadataOptions'
```

### SSM

```bash
aws ssm describe-instance-information
```

### Load balancer target health

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

These commands help correlate AWS control-plane state with what you observe inside the instance.

---

## 19. Common Mistakes

### Mistake 1 — Assuming `running` means healthy

`running` is an EC2 lifecycle state, not proof that the OS and application are healthy.

### Mistake 2 — Blaming the Security Group first

If user-data failed and the application never started, changing the Security Group does nothing.

### Mistake 3 — Checking only `systemctl`

If a service never installed because user-data failed, systemd status alone does not explain the original failure. Check cloud-init logs.

### Mistake 4 — Treating metadata as an AWS API endpoint

The metadata service is accessed from inside the instance through its link-local address. It is not the same as calling the AWS API from your laptop.

### Mistake 5 — Assuming SSH is the only access path

SSM can provide a separate operational path when configured correctly.

### Mistake 6 — Making bootstrap scripts impossible to debug

Large scripts with no logging, error handling, or validation create difficult production incidents. Keep bootstrap steps observable and validate important dependencies.

---

## 20. Interview-Ready Mental Model

When an EC2 instance is failing:

```text
EC2 STATE
   ↓
STATUS CHECKS
   ↓
BOOT / OS
   ↓
CLOUD-INIT / USER-DATA
   ↓
SSM / SSH ACCESS
   ↓
SYSTEMD / LOGS
   ↓
LISTENING PORT
   ↓
NETWORK PATH
   ↓
LOAD BALANCER / CLIENT
   ↓
APPLICATION
```

For metadata:

```text
EC2
 ↓
169.254.169.254
 ↓
IMDSv2 token
 ↓
Metadata
```

For production troubleshooting, always identify the **first failed layer** instead of changing multiple AWS resources at once.

---

## Beginner Takeaway

Remember these five ideas:

1. **`running` does not mean the application is healthy.**
2. **User-data is commonly used for EC2 bootstrap and is often processed by cloud-init on Linux AMIs.**
3. **IMDS provides instance information from inside the EC2 instance; prefer IMDSv2.**
4. **SSM and SSH are different access paths.**
5. **When troubleshooting, find the first failed layer: AWS state → boot → user-data → access → service → port → network → application.**
