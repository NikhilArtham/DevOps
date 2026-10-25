# AWS DAY 7 — Systems Manager: Session Manager, Run Command and Patch

## 1. What is AWS Systems Manager?

AWS Systems Manager (SSM) is an AWS service for operating and maintaining managed nodes such as EC2 instances.

For DevOps, remember:

```text
Session Manager → interactive shell/session
Run Command     → execute commands remotely
Patch Manager   → patch assessment and maintenance
```

A useful control-plane model is:

```text
Engineer
   ↓
Systems Manager
   ↓
Managed Node
   ↓
SSM Agent
   ↓
Operating System / Application
```

An EC2 instance being `running` does not automatically mean that it is available through Systems Manager.

---

## 2. Managed Node Prerequisites

For a typical EC2 managed node, verify:

1. SSM Agent is installed and running.
2. The instance has an appropriate IAM instance profile.
3. IAM permissions allow the SSM Agent to communicate with Systems Manager.
4. The instance can reach the required AWS Systems Manager endpoints.
5. DNS/network controls are working.

Common managed-node policy:

```text
AmazonSSMManagedInstanceCore
```

Do not memorize only the policy name. Understand the dependency:

```text
EC2
 ↓
Instance Profile
 ↓
IAM permissions
 ↓
SSM Agent
 ↓
Network connectivity
 ↓
Systems Manager
```

---

## 3. Session Manager

Session Manager provides interactive access to a managed node without requiring inbound SSH access.

Traditional path:

```text
Engineer → Bastion/VPN → TCP 22 → EC2
```

Session Manager path:

```text
Engineer → Systems Manager → SSM Agent → EC2
```

Start a session:

```bash
aws ssm start-session --target i-xxxxxxxxxxxxxxxxx
```

The `--target` value identifies the managed node.

### Why it matters

For private EC2 instances, Session Manager can remove the need for a public SSH path and can reduce reliance on bastion hosts.

It does **not** mean SSH is obsolete. It is an alternative operational access path.

---

## 4. Session Manager Port Forwarding

Session Manager can also provide controlled port forwarding.

Example:

```bash
aws ssm start-session \
  --target i-xxxxxxxxxxxxxxxxx \
  --document-name AWS-StartPortForwardingSession \
  --parameters 'portNumber=[8080],localPortNumber=[8080]'
```

Conceptually:

```text
Laptop :8080
    ↓
Session Manager
    ↓
EC2 :8080
```

This is useful for accessing an internal service for troubleshooting without making that service publicly reachable.

---

## 5. Run Command

Run Command executes a command on one or more managed nodes without requiring an interactive shell.

Example:

```bash
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets "Key=instanceids,Values=i-xxxxxxxxxxxxxxxxx" \
  --parameters 'commands=["uname -a","df -h","systemctl status nginx"]'
```

Flow:

```text
send-command
 ↓
SSM Document
 ↓
Targets
 ↓
Parameters/commands
 ↓
SSM Agent
 ↓
Command result
```

Check the result:

```bash
aws ssm list-command-invocations \
  --command-id <command-id> \
  --details
```

List command history:

```bash
aws ssm list-commands
```

### Session Manager vs Run Command

```text
Session Manager
→ interactive access

Run Command
→ non-interactive execution
```

---

## 6. SSM Documents

An SSM document defines instructions for a Systems Manager operation.

Examples:

```text
AWS-RunShellScript
AWS-RunPowerShellScript
AWS-StartPortForwardingSession
```

Do not confuse an SSM document with EC2 user-data.

```text
User-data
→ usually bootstrap/startup
→ commonly runs during instance initialization

Run Command
→ operational command after the node is managed
```

---

## 7. Patch Manager

Patch Manager helps assess and maintain operating-system patch compliance on managed nodes.

Simplified model:

```text
Managed node
   ↓
Patch baseline
   ↓
Assessment
   ↓
Compliance state
   ↓
Patch operation
   ↓
Validation / reboot if required
```

Important concepts:

- Managed nodes
- Patch baselines
- Patch compliance
- Patch groups/tags where applicable
- Maintenance Windows
- Reboots and application impact

### Maintenance Window mental model

```text
Maintenance Window
 ↓
Task
 ↓
Target nodes
 ↓
Patch operation
 ↓
Reboot if required
 ↓
Application validation
```

A production patch should be planned and controlled. It should not be treated as blindly running `yum update` or `apt upgrade` on every server.

---

## 8. Patch Compliance vs EC2 Health

These are different concepts.

```text
EC2 state = running
```

does not mean:

```text
Patch compliance = compliant
```

A node can be:

```text
EC2 = running
SSM = connected
Patch = non-compliant
```

This distinction is important for security and operational audits.

---

## 9. Important AWS CLI Commands

### Managed nodes

```bash
aws ssm describe-instance-information
```

Useful fields include:

```text
InstanceId
PingStatus
LastPingDateTime
AgentVersion
PlatformName
```

### Session

```bash
aws ssm start-session --target <instance-id>
```

### Run Command

```bash
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets "Key=instanceids,Values=<instance-id>" \
  --parameters 'commands=["hostname","uptime"]'
```

### Command result

```bash
aws ssm list-command-invocations \
  --command-id <command-id> \
  --details
```

### SSM documents

```bash
aws ssm list-documents
```

### Maintenance windows

```bash
aws ssm describe-maintenance-windows
```

### Patch baselines

```bash
aws ssm describe-patch-baselines
```

---

## 10. SSM Agent Troubleshooting

On Linux:

```bash
systemctl status amazon-ssm-agent
```

Restart when appropriate:

```bash
sudo systemctl restart amazon-ssm-agent
```

Inspect logs:

```bash
journalctl -u amazon-ssm-agent
```

Useful process/network checks:

```bash
ps aux | grep amazon-ssm-agent
ss -ntp
```

The objective is to prove whether the agent is running and communicating rather than repeatedly retrying the SSM operation.

---

## 11. Network Requirements

A private EC2 instance still needs a network path to the Systems Manager services it uses.

Possible architecture:

```text
Private EC2
   ↓
NAT Gateway
   ↓
AWS Systems Manager endpoints
```

or private connectivity through VPC interface endpoints for the required Systems Manager services.

Troubleshoot:

```text
Route table
 ↓
NAT or VPC endpoints
 ↓
Endpoint Security Group
 ↓
NACL
 ↓
DNS
 ↓
HTTPS connectivity
```

Useful checks include:

```bash
getent hosts <ssm-endpoint>
curl -I https://<ssm-endpoint>
```

Do not assume that because the EC2 instance has no public IP, Session Manager cannot work. Private connectivity is a normal design.

---

## 12. Realistic Failure Scenario — Session Manager Cannot Connect

### Architecture

```text
Engineer
   ↓
Systems Manager
   ↓
VPC interface endpoint
   ↓
Private EC2
   ↓
SSM Agent
```

Incident:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

fails because the node is not connected/available.

### Troubleshooting order

```text
1. EC2 state
2. describe-instance-information
3. SSM Agent status
4. IAM instance profile
5. IAM permissions
6. DNS resolution
7. Route to SSM endpoint
8. VPC endpoint state
9. Endpoint Security Group
10. NACL
11. SSM Agent logs
```

### Example root cause

Suppose:

```text
EC2 = running
SSM Agent = running
IAM = correct

VPC interface endpoint for SSM = exists

Endpoint Security Group:
TCP 443 from the EC2 security group = NOT allowed
```

Traffic fails here:

```text
SSM Agent
 ↓
HTTPS :443
 ↓
VPC endpoint ENI
 ↓
Endpoint SG blocks traffic
 ↓
SSM connectivity fails
```

Correct the endpoint Security Group according to the approved network design.

**Do not expose SSH to the internet just to bypass the SSM problem.**

Validate:

```bash
aws ssm describe-instance-information
```

Then:

```bash
aws ssm start-session --target i-0123456789abcdef0
```

Finally test Run Command:

```bash
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets "Key=instanceids,Values=i-0123456789abcdef0" \
  --parameters 'commands=["hostname","uptime"]'
```

---

## 13. Run Command Failure Scenario

A node can be connected while a Run Command still fails.

Trace:

```text
Command ID
 ↓
Invocation status
 ↓
SSM Document
 ↓
Parameters
 ↓
SSM Agent
 ↓
OS permissions
 ↓
Service/application
```

Example:

```bash
aws ssm send-command \
  --document-name AWS-RunShellScript \
  --targets "Key=instanceids,Values=i-xxxxxxxx" \
  --parameters 'commands=["systemctl restart myapp"]'
```

If this fails because of Linux permissions, that is not necessarily an AWS IAM problem.

Remember:

```text
AWS IAM permissions
≠
Linux sudo/service permissions
```

---

## 14. Patch Failure Scenario

Suppose a production node is marked non-compliant after a patch operation.

Separate these layers:

```text
SSM control-plane connectivity
        ↓
Patch Manager configuration
        ↓
Patch baseline
        ↓
Package repository/network
        ↓
OS package manager
        ↓
Reboot
        ↓
Application health
```

Check:

```text
Managed node status
Patch baseline
OS support
Repository connectivity
Patch operation result
Dependency errors
Reboot requirement
Application status
```

A patch failure is not automatically an SSM connectivity failure.

---

## 15. EKS Relevance

Systems Manager can be useful when troubleshooting the underlying EC2 nodes used by an EKS environment.

But do not confuse:

```text
SSM Session Manager
```

with:

```text
kubectl exec
```

`kubectl exec` enters a container through Kubernetes. Session Manager provides access to the managed operating-system node.

---

## 16. Production Troubleshooting Decision Tree

```text
EC2 running?
  |
  +-- NO → EC2/startup problem
  |
  +-- YES
       ↓
SSM recognizes node?
  |
  +-- NO
  |    ↓
  |  Agent → IAM → DNS → Route → NAT/Endpoint → SG/NACL
  |
  +-- YES
       ↓
Session fails?
       ↓
IAM + session configuration + agent + network

Run Command fails?
       ↓
Command ID → invocation → document → parameters → OS command

Patch fails?
       ↓
Node → baseline → repository → package manager → reboot → application
```

---

## 17. Common Mistakes

### Mistake 1 — "EC2 is running, so SSM must work."

False. Running is only the EC2 instance state.

### Mistake 2 — Opening SSH to fix Session Manager

This can hide the real problem and unnecessarily increase attack surface.

### Mistake 3 — Treating Session Manager and Run Command as identical

```text
Session Manager → interactive
Run Command → non-interactive
```

### Mistake 4 — Treating patch compliance as instance health

A running instance can be non-compliant.

### Mistake 5 — Blaming IAM for every SSM failure

IAM can be correct while the agent, DNS, route, endpoint, Security Group, or NACL is broken.

---

## 18. Interview-Ready Mental Model

```text
SYSTEMS MANAGER
      ↓
MANAGED NODE
      ↓
SSM AGENT
      ↓
IAM + NETWORK
      ↓
OPERATING MODE
      ├── Session Manager → interactive shell
      ├── Run Command → remote command/document
      └── Patch Manager → patch/compliance
```

One-minute answer:

> AWS Systems Manager provides centralized operational access and automation for managed nodes. Session Manager provides interactive access without requiring inbound SSH, Run Command executes commands or SSM documents on selected nodes, and Patch Manager handles patch assessment and maintenance. For troubleshooting, I verify the instance is running and appears in `describe-instance-information`, then check the SSM Agent, IAM instance profile, DNS and network connectivity to Systems Manager endpoints, and endpoint Security Groups/NACLs. Once the node is managed, I troubleshoot the specific session, command, or patch operation separately.

---

## Beginner Takeaway

You should now be able to explain:

- What Systems Manager is.
- What a managed node is.
- Why the SSM Agent matters.
- Session Manager vs SSH.
- Run Command vs interactive sessions.
- What an SSM document is.
- Patch Manager and patch compliance.
- Why IAM and network connectivity both matter.
- How to diagnose an EC2 instance that is running but unavailable through Session Manager.
