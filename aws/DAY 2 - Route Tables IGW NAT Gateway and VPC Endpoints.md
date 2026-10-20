# DAY 2 — Route Tables, IGW, NAT Gateway and VPC Endpoints

## 1. Why This Topic Matters

When an AWS workload cannot reach another service, the first question is not always **"Is the Security Group open?"**.

You first need to understand the network path:

```text
Source
  ↓
Subnet
  ↓
Route Table
  ↓
Next Hop
  ↓
IGW / NAT Gateway / VPC Endpoint / Local
  ↓
Destination
```

A DevOps engineer should be able to look at a route table and explain exactly where traffic will go.

---

## 2. Route Tables

A **route table** is a collection of rules that tells AWS where network traffic should go.

Think of it like a road-sign system:

```text
Destination        Next Hop
10.0.0.0/16        local
0.0.0.0/0          igw-xxxx
```

Meaning:

- Traffic for `10.0.0.0/16` stays inside the VPC.
- Other IPv4 traffic goes to the Internet Gateway.

### Important route-table concepts

- A subnet is associated with a route table.
- A route has a destination and a target/next hop.
- AWS chooses the most specific matching route.
- Every VPC route table has a local route for the VPC CIDR.
- A subnet does not become public simply because someone names it `public`.
- Its routing determines whether it has a path to the internet.

Example:

```text
VPC: 10.0.0.0/16

Public route table:
10.0.0.0/16 → local
0.0.0.0/0   → Internet Gateway
```

```text
Private route table:
10.0.0.0/16 → local
0.0.0.0/0   → NAT Gateway
```

---

## 3. Longest-Prefix Matching

If more than one route matches a destination, AWS uses the most specific route.

Example:

```text
10.0.0.0/16  → local
10.0.5.0/24  → some-target
```

For destination `10.0.5.20`, the `/24` route is more specific than `/16`, so the `/24` route wins.

This matters when troubleshooting unexpected traffic paths.

---

## 4. Internet Gateway — IGW

An **Internet Gateway (IGW)** is a VPC component that provides internet connectivity for resources that are configured for public internet access.

Think:

```text
VPC ↔ Internet Gateway ↔ Internet
```

An IGW alone does **not** make an EC2 instance internet-accessible.

You also need the correct route, addressing, and security configuration.

### Typical public path

```text
EC2
 ↓
Public Subnet
 ↓
Route Table
 ↓
0.0.0.0/0 → IGW
 ↓
Internet
```

For an EC2 instance to communicate directly with the public internet, it normally needs an appropriate public IPv4 address or another supported public addressing mechanism, plus the required security controls.

### Common IGW mistake

```text
Route table:
10.0.0.0/16 → local
```

but no:

```text
0.0.0.0/0 → IGW
```

The instance can communicate within the VPC but does not have a default internet route.

---

## 5. NAT Gateway

A **NAT Gateway** allows resources in private subnets to initiate outbound connections to the internet without giving those resources direct inbound internet access through an IGW.

Typical architecture:

```text
Private EC2
   ↓
Private Route Table
   ↓
0.0.0.0/0 → NAT Gateway
   ↓
Public Subnet
   ↓
Public Route Table
   ↓
0.0.0.0/0 → IGW
   ↓
Internet
```

### Important distinction

```text
IGW  → provides VPC internet path
NAT  → provides outbound internet access for private resources
```

A NAT Gateway does not make a private EC2 instance directly reachable from the internet.

### NAT Gateway placement

A common design places the NAT Gateway in a **public subnet**.

The public subnet containing the NAT Gateway needs a route to the Internet Gateway.

The private subnet routes internet-bound traffic to the NAT Gateway.

### Production design

For multi-AZ workloads, a common resilient design is to deploy a NAT Gateway per AZ and route each private subnet to the NAT Gateway in the same AZ where practical.

This can improve resilience and avoid unnecessary cross-AZ traffic, although NAT Gateways have additional cost.

---

## 6. VPC Endpoints

A **VPC endpoint** provides private connectivity from a VPC to supported AWS services and, for interface endpoints, supported endpoint services without requiring traffic to travel through the public internet.

This is extremely useful for private workloads.

Instead of:

```text
Private EC2
 ↓
NAT Gateway
 ↓
IGW
 ↓
Internet
 ↓
AWS service endpoint
```

you can often use:

```text
Private EC2
 ↓
VPC Endpoint
 ↓
AWS service
```

### Why use VPC endpoints?

- Keep traffic on private AWS networking where supported.
- Reduce dependence on NAT for supported AWS services.
- Improve security posture.
- Potentially reduce NAT Gateway data-processing cost.
- Allow private subnets to access services such as Amazon S3 or DynamoDB using supported endpoint types.

---

## 7. Gateway Endpoints vs Interface Endpoints

There are two important endpoint categories to know.

### Gateway Endpoint

Gateway endpoints are used for supported AWS services such as:

- Amazon S3
- DynamoDB

They work through route-table entries.

Conceptually:

```text
Private EC2
 ↓
Route Table
 ↓
Prefix-list destination → Gateway Endpoint
 ↓
S3 / DynamoDB
```

There is no ENI created in each subnet for a gateway endpoint.

### Interface Endpoint

An interface endpoint uses **AWS PrivateLink** and creates endpoint network interfaces in selected subnets.

Conceptually:

```text
Private EC2
 ↓
DNS name
 ↓
Interface Endpoint ENI
 ↓
AWS service / endpoint service
```

Interface endpoints require appropriate Security Group rules on the endpoint ENIs.

They are commonly used for many AWS services and supported endpoint services.

---

## 8. Gateway vs Interface Endpoint — Quick Comparison

| Feature | Gateway Endpoint | Interface Endpoint |
|---|---|---|
| Common examples | S3, DynamoDB | Many AWS services / PrivateLink services |
| Main mechanism | Route table | ENI + PrivateLink |
| Route-table entry | Yes | Usually DNS/ENI-based path |
| Security Group on endpoint ENI | No endpoint ENI | Yes |
| Typical private access | S3/DynamoDB | APIs/services supported by PrivateLink |

The exact supported services and pricing can change, so verify current AWS service documentation before designing production architecture.

---

## 9. Route Table + VPC Endpoint Example

Suppose:

```text
VPC: 10.0.0.0/16
Private subnet: 10.0.11.0/24
```

The route table may contain:

```text
10.0.0.0/16 → local
pl-xxxxxxxx → vpce-xxxxxxxx
0.0.0.0/0   → nat-xxxxxxxx
```

Now traffic behaves differently depending on destination.

```text
Destination: internal VPC
→ local
```

```text
Destination: S3
→ gateway endpoint
```

```text
Destination: public internet
→ NAT Gateway
```

This is an important production mental model: **different destinations can take different paths from the same subnet.**

---

## 10. VPC Endpoint DNS

Interface endpoints commonly rely on private DNS names so applications can continue using normal AWS service hostnames while resolving them to private endpoint addresses.

For example, an application might call an AWS service using its normal regional endpoint name.

If private DNS is enabled for the interface endpoint, DNS resolution can direct the workload to the private endpoint ENI addresses.

Troubleshooting therefore includes both:

```text
DNS resolution
+
Endpoint ENI connectivity
+
Endpoint Security Group
```

---

## 11. AWS CLI Commands

### List route tables

```bash
aws ec2 describe-route-tables
```

Useful fields include:

```text
RouteTableId
VpcId
Routes
Associations
```

### Inspect routes for one route table

```bash
aws ec2 describe-route-tables \
  --route-table-ids rtb-xxxxxxxx
```

### List Internet Gateways

```bash
aws ec2 describe-internet-gateways
```

### List NAT Gateways

```bash
aws ec2 describe-nat-gateways
```

### List VPC endpoints

```bash
aws ec2 describe-vpc-endpoints
```

### Inspect subnets

```bash
aws ec2 describe-subnets
```

### Inspect route-table associations

```bash
aws ec2 describe-route-tables \
  --query 'RouteTables[].{RT:RouteTableId,VPC:VpcId,Routes:Routes,Associations:Associations}'
```

### Inspect endpoint details

```bash
aws ec2 describe-vpc-endpoints \
  --query 'VpcEndpoints[].{Id:VpcEndpointId,Type:VpcEndpointType,Service:ServiceName,VPC:VpcId,State:State,Subnets:SubnetIds,RouteTables:RouteTableIds,SGs:Groups}'
```

---

## 12. Useful AWS Objects to Recognize

```text
VPC
├── Subnet
│   └── Route Table Association
│
├── Route Table
│   ├── local
│   ├── IGW route
│   ├── NAT route
│   └── VPC Endpoint route
│
├── Internet Gateway
│
├── NAT Gateway
│   └── Elastic IP
│
└── VPC Endpoint
    ├── Gateway Endpoint
    └── Interface Endpoint
        └── Endpoint ENI
```

---

## 13. Linux Troubleshooting Connection

When troubleshooting from an EC2 instance, AWS routing is only one layer.

Useful commands:

```bash
ip addr
ip route
ip route get 8.8.8.8
ss -tulpn
curl -v https://example.com
curl -v https://s3.<region>.amazonaws.com
getent hosts <hostname>
```

Think in layers:

```text
Application
 ↓
DNS
 ↓
Linux route
 ↓
VPC route table
 ↓
IGW / NAT / Endpoint
 ↓
Security Group / NACL
 ↓
Destination
```

---

## 14. Kubernetes / EKS Connection

EKS workloads use AWS VPC networking, so route-table and endpoint problems can appear as Kubernetes problems.

Examples:

- Nodes cannot pull images because private subnets cannot reach the required registry endpoints.
- Pods cannot reach AWS APIs because required routes, endpoints, DNS, or Security Groups are incorrect.
- Private EKS clusters commonly depend on VPC endpoints and/or controlled NAT access for required AWS services.

When an EKS workload fails, do not immediately assume Kubernetes is the root cause.

Trace:

```text
Pod / Node
 ↓
ENI / subnet
 ↓
Route table
 ↓
NAT / VPC endpoint / local route
 ↓
DNS + Security Groups
 ↓
AWS service
```

---

## 15. Realistic Production Failure Scenario

### Incident

An EC2 instance in a private subnet suddenly cannot download packages or call an AWS API.

Architecture:

```text
Private EC2
10.0.11.25
     ↓
Private Route Table
     ↓
0.0.0.0/0 → NAT Gateway
     ↓
Public Route Table
     ↓
0.0.0.0/0 → IGW
```

The team recently changed route tables and added a VPC endpoint.

### Symptoms

```text
curl https://example.com → timeout
curl AWS service endpoint → timeout
```

### Investigation

First check the subnet's route-table association.

```bash
aws ec2 describe-route-tables \
  --query 'RouteTables[].{RT:RouteTableId,Routes:Routes,Associations:Associations}'
```

Then check the expected routes:

```text
Private subnet:
0.0.0.0/0 → NAT Gateway
```

```text
Public subnet containing NAT:
0.0.0.0/0 → IGW
```

Then inspect NAT status:

```bash
aws ec2 describe-nat-gateways
```

Then inspect VPC endpoints:

```bash
aws ec2 describe-vpc-endpoints
```

### Root cause

The private subnet was accidentally associated with a route table intended for an isolated endpoint-only subnet. It had no default NAT route.

The newly created VPC endpoint only provided private access to a specific AWS service. It did **not** provide general internet access.

Therefore:

```text
AWS service covered by endpoint
→ endpoint path may work

example.com
→ no NAT/default route
→ timeout
```

### Correct fix

Associate the workload subnet with the intended private route table containing:

```text
VPC CIDR     → local
0.0.0.0/0    → NAT Gateway
```

Keep the VPC endpoint route/configuration only for the AWS services that should use it.

### Key lesson

**A VPC endpoint is not a replacement for a NAT Gateway for arbitrary internet traffic.**

Always ask:

```text
Where is the destination?
↓
Which route matches it?
↓
What is the next hop?
```

---

## 16. Production Troubleshooting Decision Tree

When a workload cannot reach a destination:

```text
1. Identify source
   ↓
2. Identify source subnet
   ↓
3. Find associated route table
   ↓
4. Identify destination IP/service
   ↓
5. Find the most specific matching route
   ↓
6. Identify next hop
   ↓
7. Validate IGW / NAT / endpoint state
   ↓
8. Check endpoint DNS if applicable
   ↓
9. Check Security Group / NACL
   ↓
10. Test from the workload
```

For VPC endpoints, specifically ask:

```text
Is this service supported?
Is the endpoint available?
Is the route table associated correctly? (gateway endpoint)
Are endpoint subnets correct? (interface endpoint)
Is private DNS configured correctly? (interface endpoint)
Does the endpoint Security Group allow traffic? (interface endpoint)
```

---

## 17. Common Mistakes

### Mistake 1 — "IGW gives internet automatically"

Wrong.

You need the correct route and appropriate public addressing/security configuration.

### Mistake 2 — "Private subnet means no internet"

Not necessarily.

A private subnet can have outbound internet access through NAT.

### Mistake 3 — "NAT Gateway accepts inbound connections"

A NAT Gateway is for outbound connectivity initiated by private resources; it is not a general inbound reverse proxy.

### Mistake 4 — "VPC endpoint gives internet"

No.

It provides private connectivity to supported services/endpoints, not arbitrary public websites.

### Mistake 5 — "A VPC endpoint automatically fixes AWS API access"

Not necessarily. Endpoint type, route table, DNS, endpoint subnet placement, Security Groups, and service support all matter.

### Mistake 6 — "Changing Security Groups will fix every timeout"

If the route is missing, Security Group changes cannot create a network path.

---

## 18. Interview-Ready Mental Model

Remember this:

```text
Route Table = where should traffic go?

IGW = path between VPC and internet for public connectivity.

NAT Gateway = private subnet → outbound internet.

VPC Endpoint = private path to supported AWS services/endpoints.
```

And the troubleshooting model:

```text
SOURCE
 ↓
SUBNET
 ↓
ROUTE TABLE
 ↓
MOST SPECIFIC ROUTE
 ↓
NEXT HOP
 ↓
IGW / NAT / VPC ENDPOINT / LOCAL
 ↓
SECURITY + DNS
 ↓
DESTINATION
```

## Beginner Takeaway

If you remember only five things:

1. **Route tables decide the path.**
2. **IGW provides the public internet path for appropriately configured resources.**
3. **NAT Gateway lets private resources initiate outbound internet connections.**
4. **VPC endpoints provide private access to supported services without needing the public internet path.**
5. **A VPC endpoint for one service does not replace a NAT Gateway for general internet traffic.**
