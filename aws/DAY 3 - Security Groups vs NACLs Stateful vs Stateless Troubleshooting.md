# DAY 3 — Security Groups vs NACLs: Stateful vs Stateless Troubleshooting

## Why this topic matters

Security Groups and Network ACLs both filter network traffic, but they work differently.

A DevOps engineer must be able to answer:

> Is the traffic failing because of the route, the Security Group, the NACL, the host firewall, or the application?

This topic focuses on the security layer after routing is understood, and uses Terraform to implement and safely recover the configuration.

---

## 1. Beginner mental model

Think of the path as:

```text
Client
  ↓
Route table
  ↓
Network ACL
  ↓
ENI / Security Group
  ↓
EC2 / application
```

For return traffic, remember:

```text
Security Group → STATEFUL
NACL           → STATELESS
```

That difference is one of the most important AWS networking interview concepts.

---

# 2. Security Group

A Security Group (SG) is a virtual firewall associated with resources through network interfaces.

Typical examples:

```text
ALB Security Group
EC2 Security Group
RDS Security Group
EKS node Security Group
```

Security Groups are primarily used to control traffic at the resource/network-interface level.

### Example

Suppose an EC2 web server listens on TCP 8080.

```text
Inbound SG rule:
TCP 8080
Source: ALB Security Group
```

This means the EC2 instance accepts port 8080 traffic from resources associated with the allowed source Security Group.

---

# 3. Why Security Groups are stateful

Security Groups are **stateful**.

If an allowed inbound connection is established, the return traffic is automatically allowed by the Security Group even if there is no separate outbound rule specifically matching the response.

Conceptually:

```text
Client → EC2
TCP 443
ALLOW

EC2 → Client
Response traffic
ALLOW because the connection is tracked
```

This does **not** mean Security Groups ignore outbound rules in every situation. The complete SG configuration still matters, and the traffic must be allowed according to the connection state and applicable rules.

### Interview shortcut

> SG remembers the connection.

---

# 4. Network ACL (NACL)

A Network ACL is a subnet-level network filter.

It applies to traffic entering and leaving the subnet.

Important properties:

- associated with a subnet
- numbered rules
- evaluated in rule-number order
- first matching rule wins
- supports ALLOW and DENY
- stateless
- inbound and outbound rules are evaluated independently

---

# 5. Why NACLs are stateless

NACLs do not automatically remember an established connection.

If inbound traffic is allowed, the return traffic must also be allowed by the outbound NACL rules.

Example:

```text
Client
  ↓
TCP 443
  ↓
Inbound NACL → ALLOW
  ↓
EC2
  ↓
Response
  ↓
Outbound NACL → must ALLOW
  ↓
Client
```

If the outbound NACL blocks the response, the application can appear to receive a connection but the client may still experience a timeout.

### Interview shortcut

> NACL does not remember the connection; return traffic needs its own matching permission.

---

# 6. Security Group vs NACL

| Feature | Security Group | Network ACL |
|---|---|---|
| Scope | Resource / ENI | Subnet |
| Stateful | Yes | No |
| Rules | Allow rules | Allow and deny rules |
| Rule order | No numbered first-match model | Lowest rule number first |
| Return traffic | Automatically tracked | Must be explicitly allowed |
| Typical use | Resource-level access control | Subnet-level guardrail / deny control |
| Association | ENI/resource | Subnet |

A strong production engineer understands that SG and NACL are complementary, not replacements for each other.

---

# 7. Traffic troubleshooting order

Do not immediately change the Security Group.

Start with the path:

```text
1. DNS
2. Route
3. NACL
4. Security Group
5. Host firewall
6. Application listener
7. Application health
```

For a TCP connection, validate the destination IP and port first.

Useful commands:

```bash
curl -v http://10.0.2.10:8080
nc -vz 10.0.2.10 8080
ss -lntp
ip route get 10.0.2.10
```

---

# 8. Typical failure patterns

## Failure A — SG blocks inbound traffic

```text
Route works
↓
NACL allows
↓
SG blocks TCP 443
↓
Connection fails
```

Fix the intended SG rule, not the route.

---

## Failure B — NACL blocks inbound traffic

```text
Route works
↓
Inbound NACL DENY
↓
SG never gets useful traffic
↓
Connection fails
```

Fix the NACL only if that deny is actually unintended.

---

## Failure C — NACL allows inbound but blocks return traffic

This is a classic stateless-firewall failure.

```text
Client
 ↓
Inbound NACL ALLOW
 ↓
SG ALLOW
 ↓
EC2
 ↓
Response
 ↓
Outbound NACL DENY
 ↓
Client never receives response
```

The client may see a timeout rather than an immediate connection refusal.

---

# 9. Terraform: Security Group pattern

A small Terraform configuration can manage the SG explicitly.

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTPS from the ALB"
  vpc_id      = var.vpc_id

  ingress {
    description     = "HTTPS from ALB"
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Allow outbound HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Important Terraform resource address:

```text
aws_security_group.web
```

If a resource is indexed with `count` or `for_each`, the address changes, for example:

```text
aws_security_group.web[0]
aws_security_group.web["prod"]
```

The exact address matters when inspecting state or targeting a resource during troubleshooting.

---

# 10. Terraform: NACL pattern

For a controlled subnet-level policy, Terraform can manage an explicit NACL.

```hcl
resource "aws_network_acl" "private" {
  vpc_id = var.vpc_id

  tags = {
    Name = "private-nacl"
  }
}

resource "aws_network_acl_rule" "https_in" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 100
  egress         = false
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "10.0.0.0/16"
  from_port      = 443
  to_port        = 443
}

resource "aws_network_acl_rule" "https_out" {
  network_acl_id = aws_network_acl.private.id
  rule_number    = 100
  egress         = true
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "10.0.0.0/16"
  from_port      = 1024
  to_port        = 65535
}
```

The exact port range required for return traffic depends on the traffic pattern and operating-system/network behavior. Do not blindly copy a broad ephemeral-port rule into production without understanding the connection path.

Important Terraform resource addresses:

```text
aws_network_acl.private
aws_network_acl_rule.https_in
aws_network_acl_rule.https_out
```

---

# 11. Terraform plan

Before changing AWS:

```bash
terraform fmt
terraform validate
terraform plan
```

Read the plan carefully.

Look for:

```text
+ create
~ update in-place
- destroy
-/+ replace
```

For this topic, verify the exact resources and attributes being changed.

Example:

```text
# aws_security_group.web will be updated in-place
# aws_network_acl_rule.https_out will be updated in-place
```

Do not treat a successful `terraform plan` as proof that the application will work. The plan only describes the intended infrastructure change.

---

# 12. Inspect Terraform state

Use:

```bash
terraform state list
```

Then inspect an exact address:

```bash
terraform state show aws_security_group.web
terraform state show aws_network_acl.private
terraform state show aws_network_acl_rule.https_in
terraform state show aws_network_acl_rule.https_out
```

This teaches an important DevOps habit:

> Never troubleshoot Terraform-managed infrastructure only from the console. Know the Terraform resource address and compare configuration, state, and real infrastructure.

---

# 13. Configuration vs state vs real AWS

Think in three layers:

```text
Terraform configuration
        ↓
Terraform state
        ↓
AWS real infrastructure
```

They can temporarily disagree.

Example:

```text
main.tf
  says: NACL rule 100 = ALLOW

Terraform state
  says: rule 100 = ALLOW

AWS console
  shows: rule 100 = DENY
```

This is configuration drift.

---

# 14. Safe drift investigation

Start with:

```bash
terraform plan
```

If Terraform detects drift, do not immediately run `apply` in production.

First determine:

```text
What changed?
Who changed it?
Was it intentional?
Should Terraform become the source of truth?
Could applying Terraform break traffic?
```

Useful commands:

```bash
terraform state list
terraform state show aws_security_group.web
terraform plan
```

If the real AWS configuration is intentionally changed and Terraform should own the new configuration, update the Terraform code and review the resulting plan.

If the change was unauthorized or accidental, restore the intended configuration through the normal Terraform workflow.

---

# 15. Safe recovery from a bad SG/NACL change

Suppose a deployment accidentally introduces:

```text
NACL outbound DENY
```

and production traffic starts timing out.

Recommended process:

```text
1. Confirm the incident and affected path
2. Identify SG/NACL resource address
3. Inspect current Terraform state
4. Inspect the Terraform diff/plan
5. Identify the exact bad rule
6. Revert the Terraform change
7. Run terraform fmt
8. Run terraform validate
9. Run terraform plan
10. Review the plan
11. Apply through the approved workflow
12. Validate connectivity
```

Avoid:

```bash
terraform destroy
```

or random console edits as a first response.

---

# 16. Realistic failure scenario

## Incident: HTTPS works from one subnet but times out from another

Architecture:

```text
Client
  ↓
ALB
  ↓
EC2 private subnet
  ↓
Application :443
```

The route is correct and the EC2 Security Group allows HTTPS from the ALB Security Group.

After a Terraform change, users report intermittent timeouts.

Investigation:

```text
Route table → correct
Security Group → correct
Application → listening
Inbound NACL → ALLOW
Outbound NACL → DENY for return traffic
```

Because the NACL is stateless, the response traffic is blocked.

### Recovery

1. Identify the affected NACL and rule address.
2. Confirm the rule was introduced by the recent Terraform change.
3. Revert the configuration to the known-good rule.
4. Run `terraform plan`.
5. Confirm only the intended NACL change is present.
6. Apply through the approved deployment process.
7. Test with `curl -v` from the correct source path.
8. Confirm application metrics and load-balancer target health.

### Root cause

```text
Stateless NACL
      ↓
Return traffic not allowed
      ↓
TCP response dropped
      ↓
Client timeout
```

---

# 17. AWS CLI inspection commands

### Security Groups

```bash
aws ec2 describe-security-groups
```

Specific group:

```bash
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxx
```

### Network ACLs

```bash
aws ec2 describe-network-acls
```

Specific NACL:

```bash
aws ec2 describe-network-acls \
  --network-acl-ids acl-xxxxxxxx
```

### Find subnet association

```bash
aws ec2 describe-network-acls \
  --filters Name=association.subnet-id,Values=subnet-xxxxxxxx
```

### Inspect an instance

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx \
  --query 'Reservations[].Instances[].{Subnet:SubnetId,PrivateIP:PrivateIpAddress,SGs:SecurityGroups[*].GroupId}'
```

---

# 18. EKS connection

In EKS, SG and NACL failures can appear as Kubernetes failures.

Example:

```text
Pod
 ↓
Node ENI / Pod ENI
 ↓
Security Group
 ↓
Subnet NACL
 ↓
Route
 ↓
Destination
```

Possible symptoms:

- Pod cannot reach an AWS service
- Node cannot reach control-plane-related endpoints
- application health checks fail
- service-to-service communication times out
- external API calls fail

Do not assume the Kubernetes Service object is the root cause before checking the underlying AWS network path.

---

# 19. Production troubleshooting decision tree

```text
Connection fails
      ↓
Does DNS resolve?
      ↓ yes
Is the route correct?
      ↓ yes
Is NACL allowing traffic in BOTH directions?
      ↓ yes
Is Security Group allowing the connection?
      ↓ yes
Is the host firewall allowing it?
      ↓ yes
Is the application listening on the expected IP:port?
      ↓ yes
Check application / load balancer / service health
```

For an SG/NACL incident, always identify:

```text
Source IP
Source subnet
Destination IP
Destination subnet
Destination port
Protocol
Route table
NACL inbound rule
NACL outbound rule
Security Group inbound rule
Security Group outbound rule
Application listener
```

---

# 20. Common mistakes

### Mistake 1
Thinking SG and NACL do the same job.

**Correction:** SG is stateful/resource-level; NACL is stateless/subnet-level.

### Mistake 2
Changing the SG before checking routing.

**Correction:** Prove the route first.

### Mistake 3
Allowing inbound NACL traffic but forgetting the response path.

**Correction:** Remember NACL stateless behavior.

### Mistake 4
Using overly broad `0.0.0.0/0` rules without understanding the requirement.

**Correction:** Start with the narrowest required source/destination/port.

### Mistake 5
Running `terraform apply` immediately after detecting drift.

**Correction:** Investigate the drift and review the plan first.

### Mistake 6
Using `terraform state rm` as a generic recovery command.

**Correction:** Removing an object from state does not remove it from AWS. Use state operations only when you understand the ownership and lifecycle implications.

---

# Interview-ready mental model

> A Security Group is a stateful resource-level firewall, while a Network ACL is a stateless subnet-level firewall. SGs use allow rules and track connection state, so return traffic is automatically handled for an allowed connection. NACLs evaluate inbound and outbound traffic independently using ordered rules, so return traffic must be explicitly permitted. During troubleshooting I first verify DNS and routing, then NACLs and Security Groups, then the host firewall and application listener. With Terraform, I inspect the exact resource addresses using `terraform state list` and `terraform state show`, review `terraform plan`, and recover bad changes through a controlled configuration change rather than random console edits.

---

# Beginner takeaway

Remember only this at first:

```text
ROUTE TABLE → where does traffic go?
NACL         → is subnet traffic allowed?
SECURITY SG  → is resource traffic allowed?
```

And:

```text
SG   = STATEFUL
NACL = STATELESS
```

For Terraform:

```text
CONFIGURATION
     ↓
terraform plan
     ↓
STATE
     ↓
AWS
```

Always know the exact resource address before changing or recovering infrastructure.
