# AWS Labs

Centralized practical labs for AWS. Labs from every AWS DAY are added here.

---

## DAY 1 — AWS VPC Architecture: CIDR, Subnets and Traffic Flow

### Lab 1 — Inspect VPCs and CIDRs

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[].{VpcId:VpcId,CIDR:CidrBlock,State:State}'
```

Record:

```text
VPC ID:
CIDR:
State:
```

**Goal:** Identify the VPC network range.

---

### Lab 2 — Map Subnets to Availability Zones

```bash
aws ec2 describe-subnets \
  --query 'Subnets[].{SubnetId:SubnetId,VpcId:VpcId,CIDR:CidrBlock,AZ:AvailabilityZone}'
```

Build a simple map:

```text
VPC
 ├── AZ-a → subnet → CIDR
 ├── AZ-b → subnet → CIDR
 └── AZ-c → subnet → CIDR
```

**Goal:** Understand that each subnet belongs to one Availability Zone and uses a non-overlapping CIDR inside the VPC.

---

### Lab 3 — Inspect Route Tables

```bash
aws ec2 describe-route-tables
```

For each relevant subnet, identify:

```text
Subnet
Route table
Local route
Default route
Next hop
```

Determine which subnets are behaving as public and which as private.

**Goal:** Learn to derive subnet traffic behavior from route tables instead of subnet names.

---

### Lab 4 — Trace Public vs Private Traffic Flow

Draw these two paths:

```text
Public:
EC2 → public subnet → route table → IGW → Internet
```

```text
Private:
EC2 → private subnet → route table → NAT Gateway → public subnet → IGW → Internet
```

Then identify the actual AWS resource IDs in your environment.

**Goal:** Explain the complete path instead of saying only "the instance has internet access."

---

### Lab 5 — Inspect an EC2 Instance's Network Placement

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx \
  --query 'Reservations[].Instances[].{InstanceId:InstanceId,PrivateIP:PrivateIpAddress,Subnet:SubnetId,VPC:VpcId,AZ:Placement.AvailabilityZone,SG:SecurityGroups[*].GroupId}'
```

Record:

```text
Instance:
VPC:
Subnet:
Subnet CIDR:
AZ:
Private IP:
Security Groups:
```

**Goal:** Start troubleshooting from the actual source resource rather than guessing.

---

### Lab 6 — Diagnose a Missing NAT Route

Use a test environment where you have permission to inspect networking.

Find a private subnet route table and check whether it contains:

```text
0.0.0.0/0 → NAT Gateway
```

If it is missing, explain why a private instance cannot reach the internet through that subnet.

Then identify the expected path:

```text
Private instance
→ private route table
→ NAT Gateway
→ public route table
→ Internet Gateway
→ Internet
```

**Goal:** Diagnose routing failure before changing Security Groups.

---

### Lab 7 — AWS CLI VPC Connectivity Investigation

For a simulated connectivity incident, collect:

```bash
aws ec2 describe-instances --instance-ids i-xxxxxxxx
aws ec2 describe-subnets --subnet-ids subnet-xxxxxxxx
aws ec2 describe-route-tables
aws ec2 describe-network-interfaces
```

Create an incident report:

```text
Source instance:
Source VPC:
Source subnet:
Source subnet CIDR:
Source AZ:
Destination:
Relevant route:
Next hop:
Internet/NAT/IGW path:
Security Group:
NACL:
Likely root cause:
Fix:
Validation:
```

**Goal:** Practice a repeatable VPC troubleshooting workflow.

---

### DAY 1 Capstone — VPC Traffic Flow Failure

Create or use a test architecture containing:

```text
VPC: 10.0.0.0/16

AZ-a
 ├── Public subnet: 10.0.1.0/24
 └── Private subnet: 10.0.11.0/24

AZ-b
 ├── Public subnet: 10.0.2.0/24
 └── Private subnet: 10.0.12.0/24
```

Simulate or diagnose a private workload that cannot reach an external endpoint.

Prove the answer by checking:

```text
VPC CIDR
↓
Subnet CIDR
↓
Subnet/AZ
↓
Route table association
↓
Default route
↓
NAT Gateway
↓
Public route table
↓
Internet Gateway
↓
Security controls
```

**Success criteria:** Explain the traffic path, identify the missing/broken component, and validate the fix without randomly changing security rules.

---

## DAY 2 — Route Tables, IGW, NAT Gateway and VPC Endpoints

### Lab 1 — Map a Complete Route Path

Choose an EC2 instance and identify:

```text
Instance
↓
Subnet
↓
Associated Route Table
↓
Matching Route
↓
Next Hop
```

Commands:

```bash
aws ec2 describe-instances --instance-ids i-xxxxxxxx
aws ec2 describe-route-tables --route-table-ids rtb-xxxxxxxx
```

For these destinations, write the expected next hop:

```text
10.0.5.20
8.8.8.8
AWS service endpoint
```

**Goal:** Learn to reason from destination → matching route → next hop.

---

### Lab 2 — Public Subnet + IGW

Inspect a public subnet and prove whether it has an internet path.

Check for:

```text
0.0.0.0/0 → Internet Gateway
```

Commands:

```bash
aws ec2 describe-route-tables
aws ec2 describe-internet-gateways
aws ec2 describe-subnets
```

Draw:

```text
EC2
↓
Public subnet
↓
Route table
↓
IGW
↓
Internet
```

**Goal:** Understand why an IGW alone is not enough.

---

### Lab 3 — Private Subnet + NAT Gateway

Find a private subnet and trace its outbound internet path.

Verify:

```text
Private route table:
0.0.0.0/0 → NAT Gateway
```

Then verify the NAT Gateway's subnet route table has:

```text
0.0.0.0/0 → IGW
```

Commands:

```bash
aws ec2 describe-nat-gateways
aws ec2 describe-route-tables
```

**Goal:** Prove the two-hop design instead of assuming that a NAT Gateway automatically provides connectivity.

---

### Lab 4 — Gateway VPC Endpoint for S3/DynamoDB

In a test VPC, inspect gateway endpoints:

```bash
aws ec2 describe-vpc-endpoints
```

Identify:

```text
Endpoint ID
Service
Type
Route tables
State
```

For an S3 or DynamoDB gateway endpoint, inspect the associated route table and identify the prefix-list route targeting the endpoint.

Draw:

```text
Private EC2
↓
Route Table
↓
Prefix List → Gateway Endpoint
↓
S3 / DynamoDB
```

**Goal:** Understand that gateway endpoints integrate with route tables.

---

### Lab 5 — Interface VPC Endpoint

Inspect an interface endpoint:

```bash
aws ec2 describe-vpc-endpoints \
  --query 'VpcEndpoints[?VpcEndpointType==`Interface`].{Id:VpcEndpointId,Service:ServiceName,State:State,Subnets:SubnetIds,SGs:Groups}'
```

Identify:

```text
Endpoint service
Endpoint subnets
Endpoint Security Group
Endpoint ENIs
Private DNS setting
```

Then test DNS resolution from a workload in the VPC if you have access.

Useful commands:

```bash
getent hosts <service-endpoint-hostname>
curl -v https://<service-endpoint-hostname>
```

**Goal:** Understand the ENI + PrivateLink model.

---

### Lab 6 — Endpoint vs NAT Decision Exercise

For each destination, choose the best expected path and explain why:

```text
Public website
S3
DynamoDB
Supported AWS API through interface endpoint
Another VPC workload
```

Example reasoning:

```text
Public website → NAT/IGW path
S3 → gateway endpoint where appropriate
DynamoDB → gateway endpoint where appropriate
AWS service with interface endpoint → interface endpoint
VPC workload → local/VPC routing
```

**Goal:** Stop treating NAT, IGW, and endpoints as interchangeable components.

---

### Lab 7 — Diagnose an Endpoint Connectivity Failure

Use this checklist for an interface endpoint failure:

```text
1. Is the endpoint service supported?
2. Is endpoint state available?
3. Is the endpoint deployed in the required subnet/AZ?
4. Is Private DNS configured as intended?
5. Does DNS resolve to private endpoint addresses?
6. Does the endpoint Security Group allow the source traffic?
7. Are NACLs blocking traffic?
8. Is the application using the expected hostname/port?
```

For a gateway endpoint, additionally check:

```text
Route table association
Prefix-list route
Endpoint policy
```

**Goal:** Diagnose endpoint failures at the correct layer.

---

### DAY 2 Capstone — Private Workload with NAT + VPC Endpoint

Use or design:

```text
VPC: 10.0.0.0/16

Public subnet
 └── NAT Gateway

Private subnet
 └── EC2 / workload

Route tables
 ├── Public: 0.0.0.0/0 → IGW
 └── Private: 0.0.0.0/0 → NAT

VPC Endpoint
 └── Private access to a supported AWS service
```

Simulate this incident:

```text
AWS service access works
Public internet access fails
```

Prove whether the workload is using:

```text
VPC Endpoint
```

for the AWS service while lacking:

```text
0.0.0.0/0 → NAT Gateway
```

for general internet traffic.

**Success criteria:** Explain why one destination works and another fails, identify the matching routes, identify the next hops, and fix the correct networking component without randomly opening Security Groups.

---

## DAY 3 — Security Groups vs NACLs: Stateful vs Stateless Troubleshooting

### Lab 1 — Build a Minimal SG + NACL Terraform Configuration

Create a small Terraform configuration containing:

```text
VPC
Subnet
Security Group
Network ACL
NACL inbound rule
NACL outbound rule
```

Use explicit resource names such as:

```text
aws_security_group.web
aws_network_acl.private
aws_network_acl_rule.https_in
aws_network_acl_rule.https_out
```

Run:

```bash
terraform fmt
terraform validate
terraform plan
```

**Goal:** Understand the Terraform resource graph before applying network controls.

---

### Lab 2 — Inspect Exact Terraform Resource Addresses

After creating the configuration, run:

```bash
terraform state list
```

Then inspect each address:

```bash
terraform state show aws_security_group.web
terraform state show aws_network_acl.private
terraform state show aws_network_acl_rule.https_in
terraform state show aws_network_acl_rule.https_out
```

Record:

```text
Resource address:
AWS resource ID:
Subnet/VPC relationship:
Rule number:
Direction:
Protocol:
Port range:
CIDR/source:
```

**Goal:** Become comfortable moving from Terraform resource address → state → AWS resource.

---

### Lab 3 — Prove Security Group Stateful Behavior

In a controlled test environment, allow a client to connect to a test service through the Security Group.

Observe:

```text
Inbound connection allowed
        ↓
Response traffic succeeds
```

Inspect the SG configuration and explain why a separate response rule is not required in the same way as a stateless NACL rule.

**Goal:** Understand connection tracking instead of memorizing the word "stateful."

---

### Lab 4 — Prove NACL Stateless Behavior

Use a test subnet with an explicit NACL.

Allow the inbound test connection but intentionally make the outbound response rule incompatible with the connection.

Expected symptom:

```text
Client → server
may reach the subnet

Server → client
response blocked

Client
→ timeout
```

Restore the outbound rule and verify the connection.

**Goal:** See the practical difference between stateful SGs and stateless NACLs.

---

### Lab 5 — Compare SG and NACL Failure Modes

Create four controlled cases:

```text
Case A: Route missing
Case B: SG blocks
Case C: Inbound NACL blocks
Case D: Outbound NACL blocks response
```

For each case record:

```text
Source
Destination
Port
Route result
NACL result
SG result
Observed symptom
Root cause
Fix
```

Use:

```bash
nc -vz <destination-ip> <port>
curl -v http://<destination-ip>:<port>
```

**Goal:** Stop treating every timeout as a Security Group problem.

---

### Lab 6 — Terraform Plan and Drift Exercise

Create a known-good Terraform configuration.

Run:

```bash
terraform plan
```

Then inspect:

```bash
terraform state list
terraform state show aws_security_group.web
terraform state show aws_network_acl_rule.https_out
```

In a permitted test environment, make a controlled console-side change to a Terraform-managed rule.

Then run:

```bash
terraform plan
```

Record:

```text
Expected configuration:
Terraform state:
Actual AWS configuration:
Detected drift:
Planned correction:
```

**Goal:** Learn to distinguish configuration, state, and real infrastructure.

---

### Lab 7 — Safe Recovery from a Bad NACL Change

Simulate a production-style incident where an outbound NACL rule is changed incorrectly.

Workflow:

```text
1. Identify affected workload
2. Confirm route is correct
3. Inspect NACL
4. Identify Terraform resource address
5. Inspect state
6. Reproduce/confirm the bad rule
7. Revert the Terraform configuration
8. terraform fmt
9. terraform validate
10. terraform plan
11. Review exact changes
12. Apply through the approved workflow
13. Test connectivity
```

**Do not use `terraform destroy` as the recovery method.**

**Goal:** Practice controlled infrastructure recovery.

---

### DAY 3 Capstone — Stateful vs Stateless Production Failure

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

Simulate this incident:

```text
HTTPS requests time out
Route is correct
Security Group is correct
Application is listening
```

Investigate and prove whether the failure is:

```text
Inbound NACL
or
Outbound NACL
```

Then recover through Terraform.

Final incident report:

```text
Source:
Destination:
Protocol/port:
Route table:
NACL:
SG:
Terraform resource address:
Terraform state:
Root cause:
Recovery change:
terraform plan result:
Validation result:
```

**Success criteria:** Explain exactly why the connection failed, identify the stateful/stateless behavior involved, identify the exact Terraform resource address, recover safely, and validate the application path.

---

## DAY 4 — ALB vs NLB: Listeners, Target Health and Routing

### Lab 1 — Identify ALB vs NLB Architecture

Inspect load balancers:

```bash
aws elbv2 describe-load-balancers
```

For each load balancer record:

```text
Load balancer:
Type: ALB / NLB
Scheme:
VPC:
Availability Zones:
DNS name:
```

Explain whether the workload needs:

```text
Layer 7 HTTP/HTTPS routing
```

or:

```text
Layer 4 TCP/UDP/TLS behavior
```

**Goal:** Choose the load-balancer mental model before troubleshooting details.

---

### Lab 2 — Inspect Listeners

For an ALB/NLB, run:

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn <load-balancer-arn>
```

Record:

```text
Protocol:
Port:
Default action:
Certificate, if applicable:
Target group:
```

**Goal:** Prove whether the expected listener exists on the expected port.

---

### Lab 3 — Inspect ALB Listener Rules

For an HTTPS ALB listener:

```bash
aws elbv2 describe-rules \
  --listener-arn <listener-arn>
```

Build a routing map:

```text
Host/path condition
↓
Rule priority
↓
Target group
```

Test at least two requests that should route to different target groups.

**Goal:** Understand why a healthy target in the wrong target group does not fix an application request.

---

### Lab 4 — Inspect Target Groups and Health

```bash
aws elbv2 describe-target-groups
```

Then:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

Record:

```text
Target:
Port:
Health state:
Reason:
Description:
Health-check protocol:
Health-check path:
```

**Goal:** Connect listener → target group → target health.

---

### Lab 5 — Trace an Unhealthy ALB Target

Use a controlled target group where the application exposes a test health endpoint.

On the target:

```bash
ss -lntp
curl -v http://127.0.0.1:<port>/health
curl -v http://<private-ip>:<port>/health
```

Compare the results with the ALB target health.

Investigate:

```text
Wrong port
Wrong path
Wrong expected response
Application not listening
Loopback-only bind
Security Group
NACL
```

**Goal:** Diagnose target health at the backend instead of randomly changing the ALB.

---

### Lab 6 — ALB vs NLB Failure Comparison

Create or analyze these cases:

```text
Case A: ALB listener missing
Case B: ALB rule sends traffic to wrong target group
Case C: ALB has zero healthy targets
Case D: NLB TCP listener reaches an unhealthy application
Case E: Backend port mismatch
Case F: Target Security Group blocks load-balancer traffic
```

For each case record:

```text
Layer:
Expected path:
Observed symptom:
AWS object involved:
Root cause:
Validation command:
Fix:
```

**Goal:** Learn to map symptoms to the correct load-balancer layer.

---

### Lab 7 — AWS CLI Load Balancer Investigation

Run:

```bash
aws elbv2 describe-load-balancers
aws elbv2 describe-listeners --load-balancer-arn <load-balancer-arn>
aws elbv2 describe-rules --listener-arn <listener-arn>
aws elbv2 describe-target-groups
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

Build this incident map:

```text
Load balancer:
Listener:
Rule:
Target group:
Target:
Target port:
Health state:
Health reason:
```

**Goal:** Become comfortable investigating load balancers without relying only on the AWS console.

---

### DAY 4 Capstone — ALB 503 Production Failure

Architecture:

```text
Client
 ↓
Route 53
 ↓
ALB HTTPS :443
 ↓
/api/* rule
 ↓
API target group
 ↓
EC2 :8080
```

Incident:

```text
Users receive 503
```

Investigate in this order:

```text
1. ALB availability
2. HTTPS listener
3. Listener rule
4. Target group
5. Target health
6. Health-check path/response
7. Target port
8. Security Group
9. NACL
10. Application listener
```

Expected failure scenario:

```text
Target health = unhealthy
Reason = Target.ResponseCodeMismatch
/health = 401 Unauthorized
```

Identify why the ALB marks the target unhealthy and recover by correcting the health-check design rather than randomly opening network access.

**Success criteria:** Explain why the listener can be healthy while the application returns 503, identify the exact failing layer, validate target health, and prove the fix.

---

## DAY 5 — Route 53: Records, Health Checks and DNS Troubleshooting

### Lab 1 — Inspect Route 53 Hosted Zones and Records

Run:

```bash
aws route53 list-hosted-zones
```

Then inspect records:

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id <hosted-zone-id>
```

For selected records, record:

```text
Name:
Type:
TTL:
Routing policy:
Alias target / value:
Health-check association, if any:
```

Classify each record as:

```text
A
AAAA
CNAME
MX
TXT
NS
Alias
```

**Goal:** Understand what Route 53 is configured to return before troubleshooting the network path.

---

### Lab 2 — Compare `dig` and `nslookup`

Test the same hostname:

```bash
dig api.example.com
dig +short api.example.com
nslookup api.example.com
```

Record:

```text
Returned IP/hostname:
Record type:
TTL:
Resolver:
```

Then query a specific resolver:

```bash
dig @8.8.8.8 api.example.com
```

**Goal:** Separate what Route 53 is configured to serve from what a recursive resolver is returning.

---

### Lab 3 — Trace DNS Delegation

Run:

```bash
dig +trace api.example.com
```

Identify:

```text
Root
↓
TLD
↓
Authoritative name servers
↓
Final answer
```

If the trace fails, identify the stage where the delegation or authoritative response breaks.

**Goal:** Understand where DNS resolution can fail instead of treating DNS as one black box.

---

### Lab 4 — Test A vs AAAA Paths

Run:

```bash
dig +short api.example.com A
dig +short api.example.com AAAA
```

If both exist, test:

```bash
curl -4 -v https://api.example.com
curl -6 -v https://api.example.com
```

Record:

```text
IPv4 answer:
IPv6 answer:
IPv4 result:
IPv6 result:
```

**Goal:** Detect cases where DNS is correct but one IP family has a broken network/application path.

---

### Lab 5 — Separate DNS, Routing and Port Failures

Use one reachable service and intentionally analyze each layer:

```text
1. DNS
2. Route
3. TCP port
4. Firewall / SG / NACL
5. TLS
6. HTTP
```

Commands:

```bash
dig +short api.example.com
ip route get <destination-ip>
ping <destination-ip>
nc -vz <destination-ip> 443
curl -v https://api.example.com
```

For a TLS investigation:

```bash
openssl s_client \
  -connect <destination-ip>:443 \
  -servername api.example.com
```

Create a result table:

```text
Layer:
Command:
Result:
Pass/Fail:
Likely issue:
```

**Goal:** Stop treating every unreachable service as a DNS problem.

---

### Lab 6 — Route 53 Health Check Investigation

Inspect health checks:

```bash
aws route53 list-health-checks
```

Then:

```bash
aws route53 get-health-check \
  --health-check-id <health-check-id>
```

Check status:

```bash
aws route53 get-health-check-status \
  --health-check-id <health-check-id>
```

Record:

```text
Protocol:
Endpoint:
Port:
Path:
Failure threshold:
Health state:
```

Then test the health-check endpoint yourself with `curl`.

**Goal:** Verify whether the health check actually represents service health and whether its path/port/protocol can succeed.

---

### Lab 7 — Public vs Private Route 53 Troubleshooting

For a public hostname, test:

```bash
dig api.example.com
dig @8.8.8.8 api.example.com
```

For a private hostname from an associated VPC workload, test:

```bash
dig api.internal.example.com
getent hosts api.internal.example.com
```

Determine whether the name is expected to resolve publicly or privately.

Check the hosted-zone/VPC relationship where applicable.

**Goal:** Avoid troubleshooting a private DNS name through the wrong resolver path.

---

### Lab 8 — Route 53 + ALB Failure Investigation

Use this architecture:

```text
Client
 ↓
Route 53
 ↓
ALB :443
 ↓
Target Group
 ↓
EC2 :8080
```

Simulate or diagnose one of these:

```text
DNS resolves but TCP 443 times out
DNS resolves and TCP works but ALB returns 503
DNS returns an unexpected ALB
Health check is unhealthy
```

Use:

```bash
dig +short api.example.com
ip route get <returned-ip>
nc -vz <returned-ip> 443
curl -v https://api.example.com
aws elbv2 describe-target-health --target-group-arn <target-group-arn>
```

**Goal:** Identify whether the failure belongs to Route 53, routing, the load balancer, target health, or the application.

---

### DAY 5 Capstone — Intermittent DNS-Based Production Failure

Architecture:

```text
api.example.com
      ↓
Route 53 weighted records
 ├── 90% → ALB-A
 └── 10% → ALB-B
```

Incident:

```text
Users report intermittent failures.
```

Investigation:

```text
1. Run dig and identify returned destinations.
2. Test each returned endpoint independently.
3. Inspect Route 53 routing configuration.
4. Inspect health-check associations.
5. Inspect ALB-B target health.
6. Trace the failing target to its backend port.
7. Test the backend health endpoint.
8. Identify whether DNS, routing, port, firewall, TLS, or application health is the failing layer.
```

Expected scenario:

```text
ALB-A → healthy
ALB-B → targets unhealthy
ALB-B /health → 401 Unauthorized
```

**Success criteria:** Explain why DNS can be working while users still experience failures, identify the unhealthy destination, trace the complete path using `dig`, `ip`, `ss`, `nc`, `curl`, and AWS CLI, and fix the actual failing layer without randomly changing Security Groups or DNS records.

---

## DAY 6 — EC2 Boot, User-Data, Metadata and System Troubleshooting

### Lab 1 — EC2 State and Status Checks
```bash
aws ec2 describe-instances --instance-ids <instance-id>
aws ec2 describe-instance-status --instance-ids <instance-id> --include-all-instances
```
Record instance state, system status and instance status.

### Lab 2 — User-Data and cloud-init
```bash
cloud-init status --long
sudo tail -n 100 /var/log/cloud-init.log
sudo tail -n 100 /var/log/cloud-init-output.log
```
Find the first bootstrap failure and determine whether it prevented the application from starting.

### Lab 3 — IMDSv2
```bash
TOKEN=$(curl -sS -X PUT 'http://169.254.169.254/latest/api/token' -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')
curl -sS -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id
```
Explain the token flow and inspect metadata options.

### Lab 4 — SSH and Application Port Troubleshooting
```bash
sudo systemctl status sshd
ss -lntp
curl -v http://127.0.0.1:<port>/health
curl -v http://<private-ip>:<port>/health
```
Separate access, listener binding and application failures.

### Lab 5 — SSM Access
```bash
aws ssm describe-instance-information
aws ssm start-session --target <instance-id>
```
Identify IAM, SSM Agent and network prerequisites.

### DAY 6 Capstone — EC2 Boot Failure
EC2 is `running`, but an ALB target is unhealthy. Use status checks, SSM, cloud-init, systemd, `ss` and `curl` to prove whether the failure is AWS networking or instance bootstrap/application startup.

---

## DAY 7 — Systems Manager: Session Manager, Run Command and Patch

### Lab 1 — Managed Node Prerequisites
```bash
aws ssm describe-instance-information
sudo systemctl status amazon-ssm-agent
```

### Lab 2 — Session Manager
```bash
aws ssm start-session --target <instance-id>
```
Explain why inbound SSH is not required.

### Lab 3 — Run Command
```bash
aws ssm send-command --instance-ids <instance-id> --document-name 'AWS-RunShellScript' --parameters 'commands=["hostname","uptime","df -h"]'
```
Inspect the result and required IAM permission.

### Lab 4 — Diagnose SSM Failure
Investigate missing/offline managed nodes, Pending commands, agent failures, IAM failures and endpoint/network failures.

### Lab 5 — Patch Compliance
Inspect missing, installed and failed patches and explain compliance state.

### DAY 7 Capstone — SSM Access Incident
EC2 is running but Session Manager cannot connect. Prove which prerequisite fails: registration, agent, IAM, endpoint/network connectivity or target state.

---

## DAY 8 — IAM Policy Evaluation, Explicit Deny and Least Privilege

### Lab 1 — Identify Caller
```bash
aws sts get-caller-identity
```

### Lab 2 — Inspect Identity Policies
```bash
aws iam list-attached-role-policies --role-name <role-name>
aws iam list-role-policies --role-name <role-name>
aws iam get-policy-version --policy-arn <policy-arn> --version-id <version-id>
```

### Lab 3 — Inspect Resource Policy
```bash
aws s3api get-bucket-policy --bucket <bucket-name>
```

### Lab 4 — Policy Simulator
```bash
aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names s3:GetObject --resource-arns arn:aws:s3:::<bucket>/*
```

### Lab 5 — Explicit Deny
Create a controlled S3 case where identity policy allows an action but a resource-policy condition denies it. Prove that the explicit Deny wins.

### Lab 6 — Least-Privilege Cleanup
Reduce `s3:*` on `*` to the minimum required action/resource set and retest.

### DAY 8 Capstone — AccessDenied Decision Tree
Record caller, action, resource, identity policy, resource policy, boundary, session policy, SCP, conditions and explicit Deny. Do not use AdministratorAccess as the fix.

---

## DAY 9 — Cross-account IAM Roles and Trust Policies

### Lab 1 — Two-Gate AssumeRole Path
Document source principal → `sts:AssumeRole` permission → target trust policy → temporary session → target API permission.

### Lab 2 — Inspect Trust Policy
```bash
aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'
```

### Lab 3 — Test AssumeRole
```bash
aws sts assume-role --role-arn arn:aws:iam::<target-account>:role/<role-name> --role-session-name devops-lab
aws sts get-caller-identity
```

### Lab 4 — Separate Trust from Target Permissions
If AssumeRole succeeds but the target API fails, inspect target-role permissions, resource policies, KMS, SCPs, boundaries and conditions.

### Lab 5 — Least-Privilege Trust
Compare trusting an entire source account with trusting the exact CI role.

### DAY 9 Capstone — CI-to-Production AccessDenied
Trace Account A CI role → Account B production role → S3 API. Identify the failing gate and make only the required change.

---

## DAY 10 — S3 Security, Encryption, Bucket Policies and Access Troubleshooting

### Lab 1 — Identify Caller and Operation
```bash
aws sts get-caller-identity
aws s3api head-object --bucket <bucket> --key <key>
aws s3api get-object --bucket <bucket> --key <key> /tmp/object
```

### Lab 2 — Bucket Policy and Public Access
```bash
aws s3api get-bucket-policy --bucket <bucket>
aws s3api get-public-access-block --bucket <bucket>
aws s3api get-bucket-ownership-controls --bucket <bucket>
```

### Lab 3 — Encryption and KMS
```bash
aws s3api get-bucket-encryption --bucket <bucket>
```
If SSE-KMS is used, trace S3 authorization separately from KMS authorization.

### Lab 4 — Secure Transport and VPC Endpoint Conditions
Investigate `aws:SecureTransport` and `aws:SourceVpce` conditions and prove how network path can affect authorization.

### Lab 5 — PutObject Encryption Enforcement
Test an upload against a controlled policy requiring encryption and identify the exact Deny when the condition is missing.

### Lab 6 — S3 Access Matrix
Classify each AccessDenied case as IAM, bucket policy, Block Public Access, Object Ownership/ACL, S3 encryption, KMS, endpoint policy, SCP/boundary or condition.

### DAY 10 Capstone — SSE-KMS GetObject Failure
Trace caller → `s3:GetObject` → bucket policy → public-access controls → ownership/ACL → KMS → explicit denies/conditions. Apply the least-privilege fix and retest.

---

## DAY 11 — RDS HA, Backups, Multi-AZ and Connection Troubleshooting

### Lab 1 — Map the RDS Connection Path
```bash
aws rds describe-db-instances --db-instance-identifier <db-id> --query 'DBInstances[].{Endpoint:Endpoint.Address,Port:Endpoint.Port,Status:DBInstanceStatus,AZ:AvailabilityZone,MultiAZ:MultiAZ,Engine:Engine}'
```
Trace:
```text
Application → RDS DNS endpoint → DNS → route/network → RDS SG → DB port → RDS state → authentication
```
Use `dig`/`nslookup` and `nc -vz <endpoint> <port>`.

### Lab 2 — Inspect HA and Backup Settings
```bash
aws rds describe-db-instances --db-instance-identifier <db-id> --query 'DBInstances[].{MultiAZ:MultiAZ,BackupRetention:BackupRetentionPeriod,BackupWindow:PreferredBackupWindow,Status:DBInstanceStatus}'
```
Explain Multi-AZ/failover versus backups/PITR.

### Lab 3 — Diagnose RDS Security Group Failure
Verify the actual client source and RDS SG inbound rule for the database port. Test:
```bash
nc -vz <rds-endpoint> 5432
nc -vz <rds-endpoint> 3306
```
Use the actual engine port.

### Lab 4 — DNS vs TCP vs Database State Matrix
```text
DNS fails → name-resolution layer
DNS works + TCP timeout → route/SG/NACL/network layer
TCP connects + DB rejects → authentication/database/TLS layer
RDS unavailable → database/service state
```
Prove each result with a command or AWS observation.

### Lab 5 — Inspect DB Subnet Group
```bash
aws rds describe-db-subnet-groups --db-subnet-group-name <subnet-group>
```
Map the VPC and Availability Zones.

### Lab 6 — Failover and Endpoint Exercise
In non-production, inspect Multi-AZ and document expected failover. Verify applications use the RDS endpoint, not a hard-coded IP. Do not trigger production failover as a learning exercise.

### Lab 7 — Backup and Restore Investigation
Inspect automated-backup settings and snapshots. Explain automated backups/PITR versus manual snapshots and record the recovery workflow.

### DAY 11 Capstone — RDS Connection Incident
Scenario:
```text
Application → RDS endpoint :5432
Client reports connection timeout
RDS is running
```
Trace endpoint DNS → route/network → RDS SG → port → RDS state → HA configuration → authentication/application. Identify the exact failing layer and explain why changing backups or Multi-AZ would not fix a pure SG/port failure.
