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
→ public subnet route table
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
