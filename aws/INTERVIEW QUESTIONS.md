# AWS Interview Questions

Centralized interview questions for AWS. Questions from every AWS DAY are added here.

---

## DAY 1 — AWS VPC Architecture: CIDR, Subnets and Traffic Flow

### 1. What is an AWS VPC?

<details><summary>Answer</summary>
A VPC is a logically isolated virtual network in AWS where resources such as EC2 instances, load balancers, and other services can be placed and networked using controlled IP ranges and routing.
</details>

### 2. What is CIDR?

<details><summary>Answer</summary>
CIDR, or Classless Inter-Domain Routing, represents an IP address range using a network address and prefix length, such as `10.0.0.0/16`.
</details>

### 3. What does `/16` mean in `10.0.0.0/16`?

<details><summary>Answer</summary>
The `/16` is the prefix length. It means the first 16 bits identify the network portion. For IPv4, the range contains 65,536 mathematical addresses before AWS-specific reserved-address considerations for a subnet.
</details>

### 4. What is the difference between a VPC CIDR and a subnet CIDR?

<details><summary>Answer</summary>
The VPC CIDR defines the VPC's overall IP range. A subnet CIDR is a smaller range inside that VPC used for resources in one Availability Zone. Subnet CIDRs must fit within the VPC and should not overlap.
</details>

### 5. What is a subnet in AWS?

<details><summary>Answer</summary>
A subnet is a subdivision of a VPC IP range. Each subnet belongs to one Availability Zone and is associated with a route table that determines how traffic is routed.
</details>

### 6. What makes a subnet public?

<details><summary>Answer</summary>
A subnet is generally considered public when its associated route table has a route that provides a path to an Internet Gateway, typically through `0.0.0.0/0 → IGW`. Resources also need appropriate public addressing and security controls for internet communication.
</details>

### 7. What makes a subnet private?

<details><summary>Answer</summary>
A private subnet does not provide a direct route for resources to the internet through an Internet Gateway. It may use a NAT Gateway for outbound internet access, or it may have no internet route at all.
</details>

### 8. What is a route table?

<details><summary>Answer</summary>
A route table contains routing rules that determine where traffic destined for different IP ranges should be sent. Each subnet uses an associated route table, and AWS selects the most specific matching route.
</details>

### 9. What is the local route in a VPC route table?

<details><summary>Answer</summary>
The local route represents traffic for the VPC's CIDR and enables routing between addresses within the VPC, subject to security and other controls.
</details>

### 10. What is longest-prefix matching?

<details><summary>Answer</summary>
When multiple routes match a destination, the most specific route wins. For example, `10.0.5.0/24` is more specific than `10.0.0.0/16`, so traffic to `10.0.5.20` uses the `/24` route if both exist.
</details>

### 11. What is an Internet Gateway?

<details><summary>Answer</summary>
An Internet Gateway is a VPC component that provides a path between the VPC and the public internet when the appropriate routing, addressing, and security configuration exists.
</details>

### 12. What is a NAT Gateway?

<details><summary>Answer</summary>
A NAT Gateway provides outbound internet connectivity for resources in private subnets without requiring those resources to have direct inbound internet reachability through an Internet Gateway. It is commonly placed in a public subnet.
</details>

### 13. Can a private subnet access the internet?

<details><summary>Answer</summary>
Yes. A private subnet can use a NAT Gateway for outbound internet access. Its route table commonly contains `0.0.0.0/0 → NAT Gateway`. A private subnet can also intentionally have no internet access.
</details>

### 14. What is the difference between a Security Group and a Network ACL?

<details><summary>Answer</summary>
A Security Group is stateful and associated with network interfaces/resources. A Network ACL is associated with a subnet and is stateless, so return traffic must be allowed explicitly as appropriate. They provide traffic filtering, while route tables determine the network path.
</details>

### 15. An EC2 instance cannot reach the internet. What do you check first?

<details><summary>Answer</summary>
Identify the instance, VPC, subnet, private IP, route table association, and default route. Then determine whether the subnet has an IGW path or NAT path as intended. After routing, check Security Groups, NACLs, host firewall, DNS, and the application.
</details>

### 16. A private EC2 instance cannot download packages. What common VPC issue could cause this?

<details><summary>Answer</summary>
A common issue is a missing or incorrect `0.0.0.0/0 → NAT Gateway` route in the private subnet's route table, or a NAT Gateway whose public subnet lacks a route to an Internet Gateway.
</details>

### 17. Does adding a Security Group rule create a route?

<details><summary>Answer</summary>
No. Security Groups control allowed traffic but do not create a network path. If the route table does not provide a path to the destination, opening a Security Group will not fix the routing problem.
</details>

### 18. Why should VPC CIDRs not overlap?

<details><summary>Answer</summary>
Overlapping address ranges make routing ambiguous or impossible when connecting networks. This is especially important with VPC peering, Transit Gateway, VPN, Direct Connect, and hybrid environments.
</details>

### 19. Why is VPC knowledge important for EKS?

<details><summary>Answer</summary>
EKS networking relies heavily on VPC components such as subnets, route tables, ENIs, Security Groups, and IP address capacity. A VPC problem can therefore appear as a Kubernetes networking or Pod scheduling problem.
</details>

### 20. Explain AWS VPC traffic flow in one minute.

<details><summary>Answer</summary>
A VPC starts with an overall CIDR range, which is divided into non-overlapping subnets across Availability Zones. Each subnet is associated with a route table. The route table determines the next hop using the most specific matching route. Public subnets commonly route internet traffic through an Internet Gateway, while private subnets commonly use a NAT Gateway for outbound internet access. After routing, Security Groups, NACLs, host controls, and the application determine whether the connection succeeds. For troubleshooting, I trace source → VPC → subnet → route table → next hop → security controls → destination.
</details>
