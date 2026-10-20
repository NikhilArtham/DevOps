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

---

## DAY 2 — Route Tables, IGW, NAT Gateway and VPC Endpoints

### 1. What does a route table do in a VPC?

<details><summary>Answer</summary>
It contains rules that determine where traffic destined for different IP ranges should be sent. AWS selects the most specific matching route and forwards traffic to its target, such as local, an Internet Gateway, NAT Gateway, or supported endpoint path.
</details>

### 2. What is the difference between a route table and a Security Group?

<details><summary>Answer</summary>
A route table decides the network path. A Security Group controls whether traffic is allowed to or from network interfaces. A Security Group rule cannot create a missing route.
</details>

### 3. What is an Internet Gateway?

<details><summary>Answer</summary>
An Internet Gateway is a VPC component that provides the VPC-side path to the public internet when routing, addressing, and security are correctly configured. Merely attaching an IGW to a VPC does not make every subnet public.
</details>

### 4. Does an IGW automatically give an EC2 instance internet access?

<details><summary>Answer</summary>
No. The subnet needs an appropriate route, typically `0.0.0.0/0 → IGW`, and the resource needs suitable public addressing and security configuration for direct public internet connectivity.
</details>

### 5. What is a NAT Gateway?

<details><summary>Answer</summary>
A NAT Gateway allows resources in private subnets to initiate outbound connections to destinations outside the VPC without making those private resources directly reachable from the internet through an IGW.
</details>

### 6. Where is a NAT Gateway normally deployed?

<details><summary>Answer</summary>
A NAT Gateway is commonly deployed in a public subnet. That subnet needs a route to an Internet Gateway so the NAT Gateway itself can reach the internet.
</details>

### 7. Explain the traffic path from a private EC2 instance to the internet.

<details><summary>Answer</summary>
The private instance sends traffic to its subnet's route table. The default route typically points to a NAT Gateway. The NAT Gateway is in a public subnet whose route table sends internet traffic to an Internet Gateway, which provides the path to the internet.
</details>

### 8. Can a NAT Gateway receive unsolicited inbound internet connections to a private EC2 instance?

<details><summary>Answer</summary>
No. A NAT Gateway is designed for outbound connections initiated by private resources. It is not a general inbound reverse proxy or load balancer.
</details>

### 9. Why might a multi-AZ architecture use more than one NAT Gateway?

<details><summary>Answer</summary>
Using a NAT Gateway per Availability Zone can improve resilience and avoid unnecessary cross-AZ traffic. It costs more, so the design should balance availability, traffic patterns, and cost.
</details>

### 10. What is a VPC endpoint?

<details><summary>Answer</summary>
A VPC endpoint provides private connectivity from a VPC to supported AWS services or endpoint services without requiring the workload to use the public internet path. The exact mechanism depends on the endpoint type.
</details>

### 11. What is a gateway VPC endpoint?

<details><summary>Answer</summary>
A gateway endpoint is a VPC endpoint type used for supported services such as S3 and DynamoDB. It integrates with route tables, allowing traffic to the service through an endpoint-specific route.
</details>

### 12. What is an interface VPC endpoint?

<details><summary>Answer</summary>
An interface endpoint uses AWS PrivateLink and creates endpoint network interfaces in selected subnets. Applications can resolve supported service names to private endpoint addresses when private DNS is configured as intended.
</details>

### 13. What is the main difference between gateway and interface endpoints?

<details><summary>Answer</summary>
Gateway endpoints use route-table integration and are commonly used for S3 and DynamoDB. Interface endpoints use ENIs and PrivateLink and support many AWS services and endpoint services. Interface endpoints also involve endpoint Security Groups.
</details>

### 14. Does a VPC endpoint provide general internet access?

<details><summary>Answer</summary>
No. A VPC endpoint provides private connectivity to supported services or endpoint services. It does not replace a NAT Gateway when a private workload needs general outbound internet access.
</details>

### 15. A private EC2 can access S3 but cannot access example.com. Is that possible?

<details><summary>Answer</summary>
Yes. The workload may have a working S3 gateway endpoint while lacking a `0.0.0.0/0 → NAT Gateway` route for general internet traffic. Different destinations can therefore use different paths from the same subnet.
</details>

### 16. An interface endpoint exists but an application cannot connect. What do you check?

<details><summary>Answer</summary>
Check endpoint state, service support, endpoint subnet/AZ placement, private DNS configuration, DNS resolution, endpoint ENI Security Group rules, NACLs, and the application's destination hostname and port.
</details>

### 17. A gateway endpoint exists but S3 access fails. What should you inspect?

<details><summary>Answer</summary>
Check that the workload subnet is associated with the intended route table, the endpoint is associated with that route table, the endpoint route/prefix-list entry exists, the endpoint policy permits the request, and Security Groups/NACLs/DNS are not causing a separate failure.
</details>

### 18. A private subnet has `0.0.0.0/0 → NAT Gateway`, but internet access still fails. What next?

<details><summary>Answer</summary>
Check that the NAT Gateway is available, is in a public subnet, has the required public connectivity, and that the public subnet's route table has `0.0.0.0/0 → IGW`. Then check Security Groups, NACLs, DNS, and the host/application.
</details>

### 19. What is a common production mistake involving VPC endpoints and NAT?

<details><summary>Answer</summary>
Assuming that creating a VPC endpoint provides general internet connectivity. An endpoint only covers supported services. A private workload may still require NAT for package repositories, public APIs, or other internet destinations.
</details>

### 20. Explain route table, IGW, NAT Gateway and VPC endpoint in one minute.

<details><summary>Answer</summary>
A route table decides where traffic goes. An Internet Gateway provides the VPC path to the public internet for appropriately configured public resources. A NAT Gateway provides outbound internet access for private resources without direct inbound internet exposure. A VPC endpoint provides private connectivity to supported AWS services or endpoint services. In production troubleshooting I identify the source subnet, find its route table, find the most specific destination route, identify the next hop, and then validate the IGW, NAT, endpoint, DNS, and security controls involved.
</details>
