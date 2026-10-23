# AWS Interview Questions

Centralized interview questions for AWS. Questions from every AWS DAY are added here.

---

## DAY 1 — AWS VPC Architecture: CIDR, Subnets and Traffic Flow

### 1. What is an AWS VPC?
<details><summary>Answer</summary>A VPC is a logically isolated virtual network in AWS where resources such as EC2 instances, load balancers, and other services can be placed and networked using controlled IP ranges and routing.</details>

### 2. What is CIDR?
<details><summary>Answer</summary>CIDR, or Classless Inter-Domain Routing, represents an IP address range using a network address and prefix length, such as `10.0.0.0/16`.</details>

### 3. What does `/16` mean in `10.0.0.0/16`?
<details><summary>Answer</summary>The `/16` is the prefix length. It means the first 16 bits identify the network portion. For IPv4, the range contains 65,536 mathematical addresses before AWS-specific reserved-address considerations for a subnet.</details>

### 4. What is the difference between a VPC CIDR and a subnet CIDR?
<details><summary>Answer</summary>The VPC CIDR defines the VPC's overall IP range. A subnet CIDR is a smaller range inside that VPC used for resources in one Availability Zone. Subnet CIDRs must fit within the VPC and should not overlap.</details>

### 5. What is a subnet in AWS?
<details><summary>Answer</summary>A subnet is a subdivision of a VPC IP range. Each subnet belongs to one Availability Zone and is associated with a route table that determines how traffic is routed.</details>

### 6. What makes a subnet public?
<details><summary>Answer</summary>A subnet is generally considered public when its associated route table has a route that provides a path to an Internet Gateway, typically through `0.0.0.0/0 → IGW`. Resources also need appropriate public addressing and security controls for internet communication.</details>

### 7. What makes a subnet private?
<details><summary>Answer</summary>A private subnet does not provide a direct route for resources to the internet through an Internet Gateway. It may use a NAT Gateway for outbound internet access, or it may have no internet route at all.</details>

### 8. What is a route table?
<details><summary>Answer</summary>A route table contains routing rules that determine where traffic destined for different IP ranges should be sent. Each subnet uses an associated route table, and AWS selects the most specific matching route.</details>

### 9. What is the local route in a VPC route table?
<details><summary>Answer</summary>The local route represents traffic for the VPC's CIDR and enables routing between addresses within the VPC, subject to security and other controls.</details>

### 10. What is longest-prefix matching?
<details><summary>Answer</summary>When multiple routes match a destination, the most specific route wins. For example, `10.0.5.0/24` is more specific than `10.0.0.0/16`, so traffic to `10.0.5.20` uses the `/24` route if both exist.</details>

### 11. What is an Internet Gateway?
<details><summary>Answer</summary>An Internet Gateway is a VPC component that provides a path between the VPC and the public internet when the appropriate routing, addressing, and security configuration exists.</details>

### 12. What is a NAT Gateway?
<details><summary>Answer</summary>A NAT Gateway provides outbound internet connectivity for resources in private subnets without requiring those resources to have direct inbound internet reachability through an Internet Gateway. It is commonly placed in a public subnet.</details>

### 13. Can a private subnet access the internet?
<details><summary>Answer</summary>Yes. A private subnet can use a NAT Gateway for outbound internet access. Its route table commonly contains `0.0.0.0/0 → NAT Gateway`. A private subnet can also intentionally have no internet access.</details>

### 14. What is the difference between a Security Group and a Network ACL?
<details><summary>Answer</summary>A Security Group is stateful and associated with network interfaces/resources. A Network ACL is associated with a subnet and is stateless, so return traffic must be allowed explicitly as appropriate. They provide traffic filtering, while route tables determine the network path.</details>

### 15. An EC2 instance cannot reach the internet. What do you check first?
<details><summary>Answer</summary>Identify the instance, VPC, subnet, private IP, route table association, and default route. Then determine whether the subnet has an IGW path or NAT path as intended. After routing, check Security Groups, NACLs, host firewall, DNS, and the application.</details>

### 16. A private EC2 instance cannot download packages. What common VPC issue could cause this?
<details><summary>Answer</summary>A common issue is a missing or incorrect `0.0.0.0/0 → NAT Gateway` route in the private subnet's route table, or a NAT Gateway whose public subnet lacks a route to an Internet Gateway.</details>

### 17. Does adding a Security Group rule create a route?
<details><summary>Answer</summary>No. Security Groups control allowed traffic but do not create a network path. If the route table does not provide a path to the destination, opening a Security Group will not fix the routing problem.</details>

### 18. Why should VPC CIDRs not overlap?
<details><summary>Answer</summary>Overlapping address ranges make routing ambiguous or impossible when connecting networks. This is especially important with VPC peering, Transit Gateway, VPN, Direct Connect, and hybrid environments.</details>

### 19. Why is VPC knowledge important for EKS?
<details><summary>Answer</summary>EKS networking relies heavily on VPC components such as subnets, route tables, ENIs, Security Groups, and IP address capacity. A VPC problem can therefore appear as a Kubernetes networking or Pod scheduling problem.</details>

### 20. Explain AWS VPC traffic flow in one minute.
<details><summary>Answer</summary>A VPC starts with an overall CIDR range, which is divided into non-overlapping subnets across Availability Zones. Each subnet is associated with a route table. The route table determines the next hop using the most specific matching route. Public subnets commonly route internet traffic through an Internet Gateway, while private subnets commonly use a NAT Gateway for outbound internet access. After routing, Security Groups, NACLs, host controls, and the application determine whether the connection succeeds. For troubleshooting, I trace source → VPC → subnet → route table → next hop → security controls → destination.</details>

---

## DAY 2 — Route Tables, IGW, NAT Gateway and VPC Endpoints

### 1. What does a route table do in a VPC?
<details><summary>Answer</summary>It contains rules that determine where traffic destined for different IP ranges should be sent. AWS selects the most specific matching route and forwards traffic to its target, such as local, an Internet Gateway, NAT Gateway, or supported endpoint path.</details>

### 2. What is the difference between a route table and a Security Group?
<details><summary>Answer</summary>A route table decides the network path. A Security Group controls whether traffic is allowed to or from network interfaces. A Security Group rule cannot create a missing route.</details>

### 3. What is an Internet Gateway?
<details><summary>Answer</summary>An Internet Gateway is a VPC component that provides the VPC-side path to the public internet when routing, addressing, and security are correctly configured. Merely attaching an IGW to a VPC does not make every subnet public.</details>

### 4. Does an IGW automatically give an EC2 instance internet access?
<details><summary>Answer</summary>No. The subnet needs an appropriate route, typically `0.0.0.0/0 → IGW`, and the resource needs suitable public addressing and security configuration for direct public internet connectivity.</details>

### 5. What is a NAT Gateway?
<details><summary>Answer</summary>A NAT Gateway allows resources in private subnets to initiate outbound connections to destinations outside the VPC without making those private resources directly reachable from the internet through an IGW.</details>

### 6. Where is a NAT Gateway normally deployed?
<details><summary>Answer</summary>A NAT Gateway is commonly deployed in a public subnet. That subnet needs a route to an Internet Gateway so the NAT Gateway itself can reach the internet.</details>

### 7. Explain the traffic path from a private EC2 instance to the internet.
<details><summary>Answer</summary>The private instance sends traffic to its subnet's route table. The default route typically points to a NAT Gateway. The NAT Gateway is in a public subnet whose route table sends internet traffic to an Internet Gateway, which provides the path to the internet.</details>

### 8. Can a NAT Gateway receive unsolicited inbound internet connections to a private EC2 instance?
<details><summary>Answer</summary>No. A NAT Gateway is designed for outbound connections initiated by private resources. It is not a general inbound reverse proxy or load balancer.</details>

### 9. Why might a multi-AZ architecture use more than one NAT Gateway?
<details><summary>Answer</summary>Using a NAT Gateway per Availability Zone can improve resilience and avoid unnecessary cross-AZ traffic. It costs more, so the design should balance availability, traffic patterns, and cost.</details>

### 10. What is a VPC endpoint?
<details><summary>Answer</summary>A VPC endpoint provides private connectivity from a VPC to supported AWS services or endpoint services without requiring the workload to use the public internet path. The exact mechanism depends on the endpoint type.</details>

### 11. What is a gateway VPC endpoint?
<details><summary>Answer</summary>A gateway endpoint is a VPC endpoint type used for supported services such as S3 and DynamoDB. It integrates with route tables, allowing traffic to the service through an endpoint-specific route.</details>

### 12. What is an interface VPC endpoint?
<details><summary>Answer</summary>An interface endpoint uses AWS PrivateLink and creates endpoint network interfaces in selected subnets. Applications can resolve supported service names to private endpoint addresses when private DNS is configured as intended.</details>

### 13. What is the main difference between gateway and interface endpoints?
<details><summary>Answer</summary>Gateway endpoints use route-table integration and are commonly used for S3 and DynamoDB. Interface endpoints use ENIs and PrivateLink and support many AWS services and endpoint services. Interface endpoints also involve endpoint Security Groups.</details>

### 14. Does a VPC endpoint provide general internet access?
<details><summary>Answer</summary>No. A VPC endpoint provides private connectivity to supported services or endpoint services. It does not replace a NAT Gateway when a private workload needs general outbound internet access.</details>

### 15. A private EC2 can access S3 but cannot access example.com. Is that possible?
<details><summary>Answer</summary>Yes. The workload may have a working S3 gateway endpoint while lacking a `0.0.0.0/0 → NAT Gateway` route for general internet traffic. Different destinations can therefore use different paths from the same subnet.</details>

### 16. An interface endpoint exists but an application cannot connect. What do you check?
<details><summary>Answer</summary>Check endpoint state, service support, endpoint subnet/AZ placement, private DNS configuration, DNS resolution, endpoint ENI Security Group rules, NACLs, and the application's destination hostname and port.</details>

### 17. A gateway endpoint exists but S3 access fails. What should you inspect?
<details><summary>Answer</summary>Check that the workload subnet is associated with the intended route table, the endpoint is associated with that route table, the endpoint route/prefix-list entry exists, the endpoint policy permits the request, and Security Groups/NACLs/DNS are not causing a separate failure.</details>

### 18. A private subnet has `0.0.0.0/0 → NAT Gateway`, but internet access still fails. What next?
<details><summary>Answer</summary>Check that the NAT Gateway is available, is in a public subnet, has the required public connectivity, and that the public subnet's route table has `0.0.0.0/0 → IGW`. Then check Security Groups, NACLs, DNS, and the host/application.</details>

### 19. What is a common production mistake involving VPC endpoints and NAT?
<details><summary>Answer</summary>Assuming that creating a VPC endpoint provides general internet connectivity. An endpoint only covers supported services. A private workload may still require NAT for package repositories, public APIs, or other internet destinations.</details>

### 20. Explain route table, IGW, NAT Gateway and VPC endpoint in one minute.
<details><summary>Answer</summary>A route table decides where traffic goes. An Internet Gateway provides the VPC path to the public internet for appropriately configured public resources. A NAT Gateway provides outbound internet access for private resources without direct inbound internet exposure. A VPC endpoint provides private connectivity to supported AWS services or endpoint services. In production troubleshooting I identify the source subnet, find its route table, find the most specific destination route, identify the next hop, and then validate the IGW, NAT, endpoint, DNS, and security controls involved.</details>

---

## DAY 3 — Security Groups vs NACLs: Stateful vs Stateless Troubleshooting

### 1. What is the main difference between a Security Group and a Network ACL?
<details><summary>Answer</summary>A Security Group is a stateful firewall associated with network interfaces/resources, while a Network ACL is a stateless firewall associated with a subnet. SGs primarily use allow rules; NACLs support both allow and deny rules.</details>

### 2. What does stateful mean for a Security Group?
<details><summary>Answer</summary>Stateful means the Security Group tracks connection state. When an allowed connection is established, return traffic for that connection is automatically handled without requiring a separate reverse rule in the same way a stateless firewall would.</details>

### 3. What does stateless mean for a NACL?
<details><summary>Answer</summary>Stateless means the NACL evaluates inbound and outbound traffic independently. If inbound traffic is allowed, the return traffic still needs an appropriate outbound rule.</details>

### 4. At what level does a Security Group operate?
<details><summary>Answer</summary>A Security Group is associated with network interfaces and therefore controls traffic at the resource/network-interface level.</details>

### 5. At what level does a NACL operate?
<details><summary>Answer</summary>A Network ACL is associated with a subnet, so its rules apply to traffic entering and leaving that subnet.</details>

### 6. Can a Security Group explicitly deny traffic?
<details><summary>Answer</summary>No. Security Groups are allow-based. If traffic is not allowed by the applicable SG rules, it is denied by the implicit behavior of the Security Group.</details>

### 7. Can a NACL explicitly deny traffic?
<details><summary>Answer</summary>Yes. NACLs support explicit ALLOW and DENY rules and evaluate them in rule-number order, with the first matching rule taking effect.</details>

### 8. Why can an inbound NACL rule be correct while the connection still times out?
<details><summary>Answer</summary>Because NACLs are stateless. The response traffic must also be allowed by the outbound NACL rules. A blocked response can produce a timeout even when the inbound request was allowed.</details>

### 9. Does a Security Group create a network route?
<details><summary>Answer</summary>No. A route table determines the network path. Security Groups only control whether applicable traffic is allowed at the resource/network-interface level.</details>

### 10. What is the typical troubleshooting order for an AWS connection failure?
<details><summary>Answer</summary>Start with DNS and routing, then inspect NACLs and Security Groups, followed by the host firewall, application listener, and application health. This prevents changing security rules when the actual problem is routing or the application itself.</details>

### 11. What is a common symptom of a blocked Security Group rule?
<details><summary>Answer</summary>A TCP connection can fail because the traffic never becomes allowed at the destination network interface. Depending on the path, the client commonly experiences a timeout rather than an application-level response.</details>

### 12. Why are NACL rule numbers important?
<details><summary>Answer</summary>NACL rules are evaluated from the lowest rule number upward, and the first matching rule is applied. A broad deny with a lower number can therefore prevent a later allow rule from being reached.</details>

### 13. What should you inspect when an NACL is suspected?
<details><summary>Answer</summary>Identify the source and destination, protocol and port, source/destination subnets, inbound NACL rules, outbound NACL rules, and rule ordering. Because NACLs are stateless, inspect both directions.</details>

### 14. How do you inspect Security Groups and NACLs using AWS CLI?
<details><summary>Answer</summary>Use `aws ec2 describe-security-groups` for Security Groups and `aws ec2 describe-network-acls` for NACLs. You can filter NACLs by subnet association when investigating a specific workload.</details>

### 15. What Terraform resource addresses are useful for this topic?
<details><summary>Answer</summary>For example: `aws_security_group.web`, `aws_network_acl.private`, `aws_network_acl_rule.https_in`, and `aws_network_acl_rule.https_out`. Exact addresses matter when using `terraform state show` and when tracing a change back to code.</details>

### 16. How do you inspect the exact Terraform state for a Security Group?
<details><summary>Answer</summary>Run `terraform state list` to find the address, then use `terraform state show aws_security_group.web` or the exact address returned by `state list`.</details>

### 17. What is configuration drift in Terraform?
<details><summary>Answer</summary>Drift occurs when the real infrastructure changes outside the Terraform configuration/state expectation. Running `terraform plan` can reveal differences between the desired configuration and the infrastructure Terraform observes.</details>

### 18. What should you do when Terraform detects a security-rule drift in production?
<details><summary>Answer</summary>Do not immediately apply. Determine what changed, whether it was intentional, whether Terraform should own the new state, and what impact restoring the declared configuration could have. Then update code if appropriate, review `terraform plan`, and use the approved deployment workflow.</details>

### 19. A Terraform change causes HTTPS timeouts. Route and SG are correct. What should you investigate?
<details><summary>Answer</summary>Inspect the subnet NACL in both directions, including rule numbers and port ranges. A common stateless-firewall failure is allowing the inbound connection but blocking the return traffic with the outbound NACL.</details>

### 20. Explain SG vs NACL in one minute.
<details><summary>Answer</summary>A Security Group is a stateful resource-level firewall. It primarily allows traffic and tracks connection state, so return traffic is automatically handled for an allowed connection. A NACL is a stateless subnet-level firewall with ordered allow and deny rules, so inbound and outbound traffic must be evaluated independently. During troubleshooting I first verify DNS and routing, then NACLs and Security Groups, and finally the host and application. With Terraform I identify the exact resource address, inspect state, review the plan, and recover changes through controlled code rather than random console edits.</details>

---

## DAY 4 — ALB vs NLB: Listeners, Target Health and Routing

### 1. What is the main difference between an ALB and an NLB?
<details><summary>Answer</summary>An ALB is primarily Layer 7 and understands HTTP/HTTPS application information such as host and path. An NLB is primarily Layer 4 and is designed for TCP, UDP, and TLS connection-level load balancing.</details>

### 2. When would you normally choose an ALB?
<details><summary>Answer</summary>Use an ALB when the application needs HTTP/HTTPS-aware routing such as host-based or path-based routing, common for web applications, APIs, microservices, and ingress-style architectures.</details>

### 3. When would you normally choose an NLB?
<details><summary>Answer</summary>Use an NLB for Layer 4 workloads such as TCP or UDP services, high-throughput network workloads, or architectures that need NLB-specific connection-level behavior.</details>

### 4. What is a listener?
<details><summary>Answer</summary>A listener checks for incoming connections on a configured protocol and port and then applies the configured action or routing behavior.</details>

### 5. What is an ALB listener rule?
<details><summary>Answer</summary>An ALB listener rule evaluates request conditions such as host or path and sends matching traffic to a target group or performs another configured action. Rules have priorities and a default action is used when no earlier rule matches.</details>

### 6. What is a target group?
<details><summary>Answer</summary>A target group is a logical group of backend targets with configuration such as protocol, port, target type, and health-check settings. The load balancer sends traffic to healthy targets in the selected target group.</details>

### 7. What does target health mean?
<details><summary>Answer</summary>Target health indicates whether a backend target is responding successfully according to the configured health-check protocol, port, path, and expected behavior.</details>

### 8. Can an ALB be healthy while its targets are unhealthy?
<details><summary>Answer</summary>Yes. The load balancer can be available and its listener can work while all targets in a target group fail health checks. Clients may then receive errors such as 503 because there are no healthy targets.</details>

### 9. A target group has no healthy targets. What should you inspect?
<details><summary>Answer</summary>Check target IP/instance, target port, health-check protocol, path, expected response, application listener, Security Groups, NACLs, and whether the application is bound to the correct network interface rather than only loopback.</details>

### 10. Why can an application pass a localhost health check but fail an ALB health check?
<details><summary>Answer</summary>The application may be listening only on `127.0.0.1`, so local requests work but network requests from the load balancer cannot reach it. Test the private IP/interface path as well as localhost.</details>

### 11. What does HTTP 503 commonly mean with an ALB?
<details><summary>Answer</summary>A common ALB 503 scenario is that the selected target group has no healthy targets. Always verify target health rather than assuming the ALB listener itself is broken.</details>

### 12. What can cause a target health check to fail with a response-code mismatch?
<details><summary>Answer</summary>The health-check endpoint may return a status code outside the configured success range. For example, a `/health` endpoint that now requires authentication could return 401 while the load balancer expects a successful response.</details>

### 13. How do you inspect target health with AWS CLI?
<details><summary>Answer</summary>Use `aws elbv2 describe-target-health --target-group-arn <target-group-arn>`. Inspect target state, reason, description, target ID, and port.</details>

### 14. How do you inspect an ALB's listener rules?
<details><summary>Answer</summary>First obtain the listener ARN, then run `aws elbv2 describe-rules --listener-arn <listener-arn>`. Check priorities, conditions, actions, and target groups.</details>

### 15. What is the difference between a listener problem and a target-health problem?
<details><summary>Answer</summary>A listener problem means the expected frontend protocol/port or listener configuration is missing or incorrect. A target-health problem means the request can reach the load balancer/listener but the selected backend targets are not considered healthy.</details>

### 16. An ALB listener is correct but requests go to the wrong application. What should you inspect?
<details><summary>Answer</summary>Inspect listener-rule priorities, host/path conditions, default action, and the target group selected by the matching rule. A healthy backend in the wrong target group can still produce an incorrect response.</details>

### 17. What is an important difference between ALB and NLB troubleshooting?
<details><summary>Answer</summary>ALB troubleshooting often includes HTTP routing, host/path rules, HTTP status codes, and application-aware health checks. NLB troubleshooting starts with Layer 4 connection behavior and then continues upward if the TCP/UDP connection succeeds but the application protocol fails.</details>

### 18. What AWS CLI commands are most useful for an ALB/NLB incident?
<details><summary>Answer</summary>`describe-load-balancers`, `describe-listeners`, `describe-rules`, `describe-target-groups`, and `describe-target-health` provide the main configuration and health path.</details>

### 19. How is this relevant to EKS?
<details><summary>Answer</summary>EKS workloads can be exposed through AWS load balancers using the AWS Load Balancer Controller and Kubernetes Services/Ingress resources. A Kubernetes outage can therefore originate in AWS load-balancer listeners, target health, security controls, or backend networking.</details>

### 20. Explain the ALB/NLB troubleshooting path in one minute.
<details><summary>Answer</summary>I identify the load balancer type first: ALB for Layer 7 HTTP/HTTPS routing and NLB for Layer 4 traffic. Then I inspect the listener, listener rules where applicable, target group, target health, backend port, Security Groups/NACLs, and application listener. For an ALB 503 I specifically check whether the selected target group has healthy targets before changing network rules.</details>

---

## DAY 5 — Route 53: Records, Health Checks and DNS Troubleshooting

### 1. What is Route 53?
<details><summary>Answer</summary>Route 53 is AWS's managed DNS service. It provides DNS hosting and supports features such as health checks and DNS-based routing policies. It can also provide domain registration.</details>

### 2. What is an A record?
<details><summary>Answer</summary>An A record maps a DNS name to an IPv4 address.</details>

### 3. What is an AAAA record?
<details><summary>Answer</summary>An AAAA record maps a DNS name to an IPv6 address.</details>

### 4. What is a CNAME record?
<details><summary>Answer</summary>A CNAME maps one DNS name to another hostname. The resolver then follows that name to obtain the destination information.</details>

### 5. What is a Route 53 alias record?
<details><summary>Answer</summary>An alias is a Route 53 feature that can point a DNS name to supported AWS resources such as an ALB or CloudFront distribution. It is different from a normal CNAME and is useful for AWS-integrated DNS targets.</details>

### 6. What is the difference between a public and private hosted zone?
<details><summary>Answer</summary>A public hosted zone serves DNS information through public DNS. A private hosted zone is associated with VPCs and is intended for internal name resolution through the VPC DNS path.</details>

### 7. What does `dig` do?
<details><summary>Answer</summary>`dig` queries DNS and shows detailed information about the response, including records, TTLs, authoritative information, and resolver behavior. `dig +short` provides a concise answer.</details>

### 8. What does `dig @8.8.8.8 api.example.com` do?
<details><summary>Answer</summary>It explicitly queries the DNS resolver at `8.8.8.8` for the name. This is useful for comparing resolver behavior instead of relying only on the host's configured resolver.</details>

### 9. What does `dig +trace` help diagnose?
<details><summary>Answer</summary>It traces DNS delegation from the root through the TLD and authoritative servers. It helps identify where delegation or authoritative resolution is failing.</details>

### 10. What is the difference between NXDOMAIN and SERVFAIL?
<details><summary>Answer</summary>NXDOMAIN indicates that the queried DNS name does not exist according to the response. SERVFAIL indicates that the resolver could not successfully complete the DNS query. They require different investigations.</details>

### 11. DNS resolves correctly but `curl` times out. Is that a DNS problem?
<details><summary>Answer</summary>Not necessarily. DNS may be completely healthy. A timeout after a valid DNS answer points toward routing, firewall/Security Group/NACL, listener, or another network-path issue.</details>

### 12. What does TTL mean in DNS?
<details><summary>Answer</summary>TTL is the time-to-live value that controls how long DNS responses can be cached according to DNS caching behavior. A recent Route 53 change may therefore not be visible immediately to every client.</details>

### 13. What are common Route 53 routing policies?
<details><summary>Answer</summary>Common policies include simple, weighted, latency-based, failover, geolocation, geoproximity, and multivalue answer routing.</details>

### 14. How does weighted routing work?
<details><summary>Answer</summary>Weighted routing assigns configured weights to records and uses them to distribute DNS responses among the records. It can be useful for controlled traffic distribution such as a 90/10 rollout.</details>

### 15. What does a Route 53 health check do?
<details><summary>Answer</summary>A health check monitors an endpoint according to its configured protocol, port, path, and health-check settings. When used with supported DNS routing configurations, health information can influence which DNS answer Route 53 returns.</details>

### 16. Can a Route 53 health check prove that every user request will succeed?
<details><summary>Answer</summary>No. A health check only tests the configured endpoint behavior. It may not reproduce the exact hostname, path, authentication, application workflow, client location, or request pattern used by real users.</details>

### 17. What is a good troubleshooting order for an unreachable DNS-based service?
<details><summary>Answer</summary>Start with DNS resolution using `dig` or `nslookup`, then identify the returned IP/endpoint. Check routing with `ip route get`, test the TCP port with `nc`, validate TLS with `openssl`, and test HTTP with `curl`. Then inspect load-balancer target health and the application.</details>

### 18. What does `ip route get <destination-ip>` tell you?
<details><summary>Answer</summary>It shows how the Linux kernel would route traffic to the specified destination, helping determine the selected route/interface/next hop instead of guessing from the routing table alone.</details>

### 19. Why can `ping` fail while HTTPS works?
<details><summary>Answer</summary>Ping uses ICMP, while HTTPS uses TCP and TLS. ICMP can be blocked by network controls even when TCP port 443 is reachable and the application works.</details>

### 20. Explain DNS troubleshooting in one minute.
<details><summary>Answer</summary>I first verify the hostname and DNS answer with `dig` or `nslookup`, including the record type, returned IP, TTL, and resolver. If DNS resolves, I do not assume the application is healthy. I trace the returned destination with `ip route get`, test the port using `nc`, validate TLS with `openssl`, and test HTTP with `curl`. In AWS I then inspect Route 53 records and health checks, the ALB/NLB listener and target health, Security Groups/NACLs, and finally the application. This separates DNS, routing, port, firewall, TLS, load-balancer, and application failures.
</details>
