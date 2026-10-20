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
