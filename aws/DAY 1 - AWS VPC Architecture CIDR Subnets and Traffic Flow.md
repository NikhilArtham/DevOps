# DAY 1 — AWS VPC Architecture: CIDR, Subnets and Traffic Flow

## 1. Why this topic matters

A VPC is the private networking boundary where many AWS resources run.

As a DevOps engineer, you should be able to look at an AWS connectivity problem and answer:

```text
Which network is this resource in?
Which subnet is it in?
What IP address does it have?
Is the destination inside or outside the VPC?
Which route will be selected?
Does the subnet have a path to the destination?
Is the traffic blocked by a security control?
```

The core mental model is:

```text
VPC
 ├── CIDR range
 ├── Subnets
 │    ├── Route table association
 │    └── Availability Zone
 ├── Route tables
 ├── Internet Gateway
 ├── NAT Gateway
 ├── Security Groups
 └── Network ACLs
```

---

## 2. What is a VPC?

Amazon VPC (Virtual Private Cloud) is a logically isolated network in AWS that you control.

A VPC provides the network space in which you can place resources such as:

- EC2 instances
- Load balancers
- RDS databases
- EKS worker nodes
- other AWS networking resources

A VPC itself is not automatically public or private.

Whether a resource can reach the internet depends on the subnet routing, gateways, public addressing, and security controls involved.

---

## 3. CIDR — the address range of the network

CIDR means **Classless Inter-Domain Routing**.

A CIDR block describes a range of IP addresses.

Example:

```text
10.0.0.0/16
```

Think of `/16` as defining how much of the address is the network portion.

For IPv4:

```text
10.0.0.0/16
```

contains 65,536 addresses mathematically.

AWS reserves five IPv4 addresses in each subnet, so the number of usable IPv4 addresses in an AWS subnet is smaller than the mathematical total.

---

## 4. CIDR prefix length

Common examples:

```text
10.0.0.0/16   → large network
10.0.1.0/24   → smaller network
10.0.1.0/28   → very small subnet
```

The larger the prefix number, the smaller the address range.

For example:

```text
/16 → 65,536 addresses
/24 → 256 addresses
/28 → 16 addresses
```

For subnet design, do not memorize only the numbers. Understand the relationship:

```text
larger CIDR prefix → smaller network
smaller CIDR prefix → larger network
```

---

## 5. VPC CIDR vs subnet CIDR

Suppose the VPC is:

```text
VPC: 10.0.0.0/16
```

You can divide it into smaller subnet ranges:

```text
Public subnet A:   10.0.1.0/24
Public subnet B:   10.0.2.0/24
Private subnet A:  10.0.11.0/24
Private subnet B:  10.0.12.0/24
Database subnet A: 10.0.21.0/24
Database subnet B: 10.0.22.0/24
```

The subnet CIDRs must fit inside the VPC CIDR and cannot overlap.

A good design commonly spreads subnets across multiple Availability Zones for resilience.

---

## 6. What is a subnet?

A subnet is a smaller IP address range inside a VPC.

Each subnet belongs to one Availability Zone.

Example:

```text
VPC 10.0.0.0/16

AZ-a
 ├── Public subnet   10.0.1.0/24
 └── Private subnet  10.0.11.0/24

AZ-b
 ├── Public subnet   10.0.2.0/24
 └── Private subnet  10.0.12.0/24
```

A subnet is therefore not simply a label such as "public" or "private".

Its behavior comes largely from the routing configuration associated with it.

---

## 7. Public subnet vs private subnet

A common beginner mistake is:

> Public subnet = subnet with public IPs.

More accurately, a subnet is considered public when its associated route table has a route that provides a path to an Internet Gateway.

Example public route table:

```text
Destination     Target
10.0.0.0/16     local
0.0.0.0/0       igw-xxxxxxxx
```

Private subnet example:

```text
Destination     Target
10.0.0.0/16     local
0.0.0.0/0       nat-xxxxxxxx
```

The NAT Gateway allows resources in the private subnet to initiate outbound internet connections without making those resources directly reachable from the internet through the NAT Gateway.

A private subnet can also have no internet route at all.

---

## 8. Route tables — the traffic decision-maker

A route table tells AWS where to send traffic based on the destination IP.

Example:

```text
Destination     Target
10.0.0.0/16     local
0.0.0.0/0       igw-xxxxxxxx
```

The first route means traffic destined for the VPC CIDR uses the VPC's local routing.

The default route:

```text
0.0.0.0/0
```

matches destinations that do not have a more specific matching route.

---

## 9. Longest-prefix matching

AWS routing uses the most specific matching route.

Suppose the table contains:

```text
10.0.0.0/16  → local
10.0.5.0/24  → specific-target
0.0.0.0/0    → internet-gateway
```

Traffic to:

```text
10.0.5.20
```

matches both `/16` and `/24`, but `/24` is more specific, so the `/24` route wins.

This concept is critical during troubleshooting.

---

## 10. Local VPC traffic

A VPC route table normally contains a local route for the VPC CIDR:

```text
10.0.0.0/16 → local
```

This enables communication between resources using addresses inside the VPC, subject to security controls and other network configuration.

Example:

```text
EC2 A: 10.0.1.10
EC2 B: 10.0.11.20
```

Both are inside:

```text
10.0.0.0/16
```

The route table's local route provides the routing path between the subnets.

Security Groups, NACLs, application listeners, and host-level controls can still prevent the connection.

---

## 11. Internet Gateway traffic flow

A typical public EC2 path is:

```text
Internet
   ↓
Internet Gateway
   ↓
Public subnet route table
   ↓
EC2 network interface
   ↓
EC2 instance
```

The subnet route table needs an appropriate route such as:

```text
0.0.0.0/0 → Internet Gateway
```

The instance also needs an appropriate public addressing setup, such as a public IPv4 address or Elastic IP for IPv4 internet communication.

Security Group and NACL rules must allow the intended traffic.

---

## 12. Private subnet and NAT Gateway

A common architecture is:

```text
Internet
   ↓
Internet Gateway
   ↓
NAT Gateway
   ↓
Private subnet route table
   ↓
EC2/EKS workload
```

The private route table commonly contains:

```text
10.0.0.0/16 → local
0.0.0.0/0   → NAT Gateway
```

The NAT Gateway itself is normally placed in a public subnet so it can reach the internet through an Internet Gateway.

Important:

> NAT Gateway provides outbound connectivity initiated from private resources. It is not a replacement for an Internet Gateway and does not make private resources directly internet-addressable.

---

## 13. Traffic flow inside AWS

For an EC2-to-EC2 connection:

```text
Application
   ↓
EC2 ENI
   ↓
Subnet
   ↓
Route table
   ↓
Local VPC route
   ↓
Destination subnet
   ↓
Destination ENI
   ↓
Destination application
```

At the security layer, also consider:

```text
Source Security Group
Destination Security Group
Network ACLs
Host firewall
Application listener
```

Routing answers **where the packet should go**.

Security controls answer **whether the traffic is allowed**.

---

## 14. Security Groups vs Network ACLs

### Security Group

Security Groups are stateful and attached to network interfaces/resources.

Example inbound rule:

```text
TCP 443
Source: 0.0.0.0/0
```

This allows HTTPS inbound according to the security group's configuration.

### Network ACL

A Network ACL is associated with a subnet and provides stateless traffic filtering.

Because NACLs are stateless, return traffic must be allowed appropriately.

Beginner mental model:

```text
Route table → where traffic goes
Security Group → resource-level traffic control
NACL → subnet-level traffic control
```

---

## 15. Network interfaces and private IPs

AWS resources communicate through Elastic Network Interfaces (ENIs).

An ENI can have private IP addresses and may have public addressing associated with it depending on the resource/configuration.

For troubleshooting, identify the actual private IP and subnet before assuming the problem is with the application.

Useful AWS CLI command:

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx \
  --query 'Reservations[].Instances[].{PrivateIP:PrivateIpAddress,Subnet:SubnetId,VPC:VpcId,AZ:Placement.AvailabilityZone}'
```

---

## 16. Useful AWS CLI commands

### List VPCs

```bash
aws ec2 describe-vpcs
```

### Show VPC CIDR blocks

```bash
aws ec2 describe-vpcs \
  --query 'Vpcs[].{VpcId:VpcId,CIDR:CidrBlock}'
```

### List subnets

```bash
aws ec2 describe-subnets
```

### Show subnet details

```bash
aws ec2 describe-subnets \
  --query 'Subnets[].{SubnetId:SubnetId,VpcId:VpcId,CIDR:CidrBlock,AZ:AvailabilityZone}'
```

### List route tables

```bash
aws ec2 describe-route-tables
```

### Inspect a specific subnet

```bash
aws ec2 describe-subnets \
  --subnet-ids subnet-xxxxxxxx
```

### Find an instance's networking details

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx \
  --query 'Reservations[].Instances[].{PrivateIP:PrivateIpAddress,Subnet:SubnetId,VPC:VpcId,SG:SecurityGroups[*].GroupId}'
```

### Check network interfaces

```bash
aws ec2 describe-network-interfaces
```

---

## 17. AWS Console objects you should recognize

When troubleshooting VPC networking, identify these objects:

```text
VPC
 ├── CIDR block
 ├── Subnet
 │    ├── CIDR
 │    ├── Availability Zone
 │    └── Route table association
 ├── Route table
 │    └── Routes
 ├── Internet Gateway
 ├── NAT Gateway
 ├── Security Group
 ├── Network ACL
 └── ENI
```

For an EC2 connectivity problem, do not start by randomly changing Security Groups.

First establish:

```text
VPC → Subnet → Route table → Destination → Security controls
```

---

## 18. Kubernetes connection

EKS networking makes VPC knowledge especially important.

Depending on the EKS networking model, Pods and nodes use VPC networking components such as subnets, ENIs, route tables, Security Groups, and the VPC CNI.

A simplified path can look like:

```text
Client
  ↓
Load Balancer
  ↓
VPC subnet
  ↓
Node/Pod network interface
  ↓
Pod
```

A broken subnet route, exhausted IP space, incorrect Security Group, or incorrect network placement can therefore appear as a Kubernetes problem even when the application itself is healthy.

---

## 19. Realistic failure scenario — Private EC2 cannot reach the internet

### Architecture

```text
VPC: 10.0.0.0/16

Public subnet:  10.0.1.0/24
Private subnet: 10.0.11.0/24
```

The private EC2 instance needs to download packages.

The engineer expects:

```text
Private EC2
   ↓
Private route table
   ↓
NAT Gateway
   ↓
Public route table
   ↓
Internet Gateway
   ↓
Internet
```

But the private route table contains only:

```text
10.0.0.0/16 → local
```

There is no:

```text
0.0.0.0/0 → NAT Gateway
```

### Symptoms

```bash
curl https://example.com
```

fails or times out.

### Troubleshooting

Identify the instance:

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxx \
  --query 'Reservations[].Instances[].{Subnet:SubnetId,VPC:VpcId,PrivateIP:PrivateIpAddress}'
```

Inspect the subnet's route table association and routes:

```bash
aws ec2 describe-route-tables
```

Check that the private subnet has:

```text
0.0.0.0/0 → nat-xxxxxxxx
```

Then verify the NAT Gateway's subnet is public and its route table has:

```text
0.0.0.0/0 → igw-xxxxxxxx
```

### Root cause

Missing default route from the private subnet to the NAT Gateway.

### Fix

Add the correct route to the private subnet's route table, then test again.

The key lesson:

> **A private subnet does not magically have internet access. It needs an appropriate egress path, commonly through a NAT Gateway.**

---

## 20. VPC troubleshooting decision tree

When a resource cannot reach another destination:

```text
1. Identify source
      ↓
2. Identify VPC and subnet
      ↓
3. Identify source private IP
      ↓
4. Identify destination IP/name
      ↓
5. Check route table
      ↓
6. Determine next hop
      ↓
7. Check Security Groups
      ↓
8. Check NACLs
      ↓
9. Check host firewall
      ↓
10. Check application/listener
```

Do not jump directly to Security Groups.

A packet cannot be allowed by a Security Group if the routing path does not exist in the first place.

---

## 21. Common mistakes

### Mistake 1 — Treating public/private as a subnet name

Public/private behavior depends on routing and addressing, not the subnet's name.

### Mistake 2 — Overlapping CIDRs

Overlapping VPC/subnet CIDRs create serious routing and connectivity problems, especially when connecting networks with VPC peering, Transit Gateway, VPN, or hybrid networking.

### Mistake 3 — Forgetting Availability Zones

A subnet belongs to one AZ. Highly available architectures normally use multiple subnets across multiple AZs.

### Mistake 4 — Confusing route tables with Security Groups

A route table determines the network path. A Security Group controls allowed traffic at the resource/network-interface level.

### Mistake 5 — Assuming NAT Gateway allows inbound traffic

NAT Gateway is primarily for outbound connections initiated by private resources.

### Mistake 6 — Ignoring IP capacity

A subnet can fail to support new resources when available IP addresses are exhausted. This becomes particularly important in EKS and highly scaled environments.

---

## 22. Interview-ready mental model

Remember this:

```text
VPC
 ↓
CIDR
 ↓
SUBNET
 ↓
ROUTE TABLE
 ↓
NEXT HOP
 ↓
SECURITY GROUP / NACL
 ↓
ENI
 ↓
APPLICATION
```

For internet access:

```text
PUBLIC:
Resource → Public subnet → IGW → Internet

PRIVATE:
Resource → Private subnet → NAT → Public subnet → IGW → Internet
```

---

## Beginner takeaway

If you remember only these points:

1. **VPC = your AWS network boundary.**
2. **CIDR = the IP address range.**
3. **Subnet = a smaller CIDR range inside the VPC and belongs to one AZ.**
4. **Route table = decides where traffic goes.**
5. **Public subnet normally has a route to an Internet Gateway.**
6. **Private subnet commonly uses a NAT Gateway for outbound internet access.**
7. **Security Groups/NACLs control traffic; they do not create routing paths.**
8. **Always troubleshoot VPC connectivity from source → subnet → route → next hop → security → application.**
