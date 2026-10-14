# Networking Interview Questions

Centralized interview questions for Networking. Questions from every Networking DAY are added here.

---

## DAY 1 — TCP/IP, Ports, Sockets and `ss`/`netstat`

### 1. What is an IP address?
<details><summary>Answer</summary>
An IP address identifies a network interface on a network so traffic can be routed to the correct destination. An application endpoint commonly looks like `10.0.2.15:8080`, where the IP identifies the host and the port identifies the service endpoint.
</details>

### 2. What is the difference between an IP address and a port?
<details><summary>Answer</summary>
The IP address identifies the network destination, while the port identifies a particular service or socket on that destination.
</details>

### 3. What is a TCP three-way handshake?
<details><summary>Answer</summary>
TCP normally establishes a connection using SYN → SYN-ACK → ACK before application data is exchanged.
</details>

### 4. What is a socket?
<details><summary>Answer</summary>
A socket is a network communication endpoint. A TCP connection can be identified using protocol, source IP, source port, destination IP, and destination port.
</details>

### 5. What is the difference between LISTEN and ESTABLISHED in `ss`?
<details><summary>Answer</summary>
LISTEN means a process is waiting for incoming TCP connections. ESTABLISHED means a TCP connection has already been established.
</details>

### 6. Why is `ss` preferred over `netstat` on modern Linux?
<details><summary>Answer</summary>
`ss` is the modern socket-statistics utility and is generally available on current Linux systems. `netstat` comes from the older `net-tools` package.
</details>

### 7. What does `ss -lntp` mean?
<details><summary>Answer</summary>
`-l` listening, `-n` numeric addresses/ports, `-t` TCP, and `-p` process information.
</details>

### 8. What is the difference between `127.0.0.1` and `0.0.0.0`?
<details><summary>Answer</summary>
`127.0.0.1` is loopback and normally accepts local connections only. `0.0.0.0` means listening on all IPv4 interfaces, subject to firewall and network controls.
</details>

### 9. Does successful `ping` prove TCP port 443 is reachable?
<details><summary>Answer</summary>
No. `ping` tests ICMP while HTTPS normally uses TCP/443.
</details>

### 10. Does failed `ping` prove a server is unreachable?
<details><summary>Answer</summary>
No. ICMP may be blocked or rate-limited. Test the actual application port with `nc`, `curl`, or the application client.
</details>

### 11. What is the difference between connection refused and timeout?
<details><summary>Answer</summary>
Refused commonly means the destination actively rejected the connection, often because nothing is listening. Timeout means an expected response was not received and can indicate filtering, routing, host, or path problems.
</details>

### 12. How do you check which process is listening on port 8080?
<details><summary>Answer</summary>
Use `sudo ss -lntp | grep ':8080'`.
</details>

### 13. How do you check which route Linux will use to reach an IP?
<details><summary>Answer</summary>
Use `ip route get <destination-ip>`.
</details>

### 14. How do you troubleshoot DNS separately from connectivity?
<details><summary>Answer</summary>
Use `dig +short` to resolve the name, then test the returned IP separately with routing and TCP tools such as `ip route get` and `nc -vz`.
</details>

### 15. What is `traceroute` used for?
<details><summary>Answer</summary>
It shows intermediate network hops and can help identify where a path may become problematic. A `*` does not automatically mean a router is broken.
</details>

### 16. How would you troubleshoot an unreachable service from a client to EC2?
<details><summary>Answer</summary>
Use DNS → route → reachability → TCP port → server listener → bind address → Linux firewall → Security Group → NACL/AWS routing → application.
</details>

### 17. An application listens on 8080 but clients cannot connect. What do you check?
<details><summary>Answer</summary>
Check the listening address first. If it is `127.0.0.1:8080`, remote clients normally cannot connect. Then check firewall, Security Group, NACL, routing, and TCP testing.
</details>

### 18. What is the difference between an AWS Security Group and a Linux firewall?
<details><summary>Answer</summary>
A Security Group is an AWS network control associated with resources/network interfaces. A Linux firewall operates inside the OS. Both can affect traffic.
</details>

### 19. What is the difference between a Security Group and Network ACL?
<details><summary>Answer</summary>
Security Groups are associated with resources/network interfaces and are stateful. Network ACLs operate at subnet level and are stateless.
</details>

### 20. What is your general methodology when a production service is unreachable?
<details><summary>Answer</summary>
Prove each layer in order: Name → DNS → IP → route → host → port → listener → firewall/security controls → application. Identify the first failed layer and collect evidence.
</details>

---

## DAY 2 — DNS Resolution, `dig`, `nslookup` and Route 53

### 1. What is DNS?
<details><summary>Answer</summary>
DNS translates names such as `api.example.com` into information such as IP addresses so applications can use human-readable names.
</details>

### 2. Resolver vs authoritative DNS server?
<details><summary>Answer</summary>
A resolver finds answers for clients, often using cache or querying other DNS servers. An authoritative server holds the official records for a DNS zone.
</details>

### 3. What is an A record?
<details><summary>Answer</summary>
An A record maps a hostname to an IPv4 address.
</details>

### 4. What is an AAAA record?
<details><summary>Answer</summary>
An AAAA record maps a hostname to an IPv6 address.
</details>

### 5. What is a CNAME record?
<details><summary>Answer</summary>
A CNAME aliases one DNS name to another DNS name.
</details>

### 6. What is the difference between `dig` and `nslookup`?
<details><summary>Answer</summary>
Both query DNS. `dig` generally provides richer DNS response details and is widely used for Linux troubleshooting; `nslookup` remains common in existing environments and interviews.
</details>

### 7. What does `dig +short example.com` do?
<details><summary>Answer</summary>
It performs a DNS lookup and reduces the output to the short answer.
</details>

### 8. How do you query a specific DNS resolver using `dig`?
<details><summary>Answer</summary>
Use `dig @8.8.8.8 example.com`; the `@8.8.8.8` selects the DNS server.
</details>

### 9. What does `dig +trace` do?
<details><summary>Answer</summary>
It follows DNS delegation starting from the root and helps show how the query reaches authoritative DNS servers.
</details>

### 10. What is TTL in DNS?
<details><summary>Answer</summary>
TTL controls how long a DNS response can normally be cached by a resolver.
</details>

### 11. What is NXDOMAIN?
<details><summary>Answer</summary>
NXDOMAIN indicates that the queried domain name does not exist according to the responding DNS system.
</details>

### 12. How do you separate a DNS problem from connectivity?
<details><summary>Answer</summary>
Resolve the name independently with `dig +short`, then test the returned IP with routing and TCP tools.
</details>

### 13. What is a Route 53 hosted zone?
<details><summary>Answer</summary>
A Route 53 hosted zone is a container for DNS records for a domain or DNS namespace. It can be public or private.
</details>

### 14. Public vs private Route 53 hosted zone?
<details><summary>Answer</summary>
A public zone is intended for public DNS resolution. A private hosted zone provides DNS records for associated VPCs and internal names.
</details>

### 15. What is a Route 53 Alias record?
<details><summary>Answer</summary>
An Alias is an AWS-specific DNS feature that can point a name to supported AWS resources such as an ALB or CloudFront distribution.
</details>

### 16. Which Route 53 routing policies should a DevOps engineer know?
<details><summary>Answer</summary>
Know simple, weighted, latency-based, failover, geolocation, geoproximity, and multivalue answer routing.
</details>

### 17. What is weighted routing?
<details><summary>Answer</summary>
Weighted routing distributes DNS responses according to configured weights, useful for controlled traffic distribution such as 90/10.
</details>

### 18. What is latency-based routing?
<details><summary>Answer</summary>
It directs users toward the configured AWS region expected to provide the lowest network latency.
</details>

### 19. What is failover routing?
<details><summary>Answer</summary>
It supports a primary/secondary model and can use health-check results to influence which resource is returned.
</details>

### 20. DNS resolves successfully but the application is unreachable. What do you check?
<details><summary>Answer</summary>
Treat DNS as completed and continue through IP → route → reachability → TCP port → listener → firewall/Security Group/NACL → application.
</details>

---

## DAY 3 — HTTP/HTTPS, TLS Handshake and `curl` Troubleshooting

### 1. What is HTTP?
<details><summary>Answer</summary>
HTTP is an application-layer protocol used for client-server communication. A request contains method, path, headers, and optionally a body; the server returns a status code, headers, and optionally a body.
</details>

### 2. What is HTTPS?
<details><summary>Answer</summary>
HTTPS is HTTP carried over a TLS-protected connection. A common HTTPS endpoint uses TCP port 443.
</details>

### 3. HTTP vs HTTPS?
<details><summary>Answer</summary>
HTTPS adds TLS to HTTP, providing confidentiality, integrity, and server authentication.
</details>

### 4. What happens before an HTTPS request is sent?
<details><summary>Answer</summary>
Typically DNS resolution, TCP connection, TLS handshake/certificate validation, then HTTP application data.
</details>

### 5. What is a TLS handshake?
<details><summary>Answer</summary>
It negotiates security parameters, establishes shared key material, and authenticates the server so encrypted application traffic can begin.
</details>

### 6. What is `ClientHello`?
<details><summary>Answer</summary>
It starts TLS negotiation and can include supported TLS versions, cipher suites, SNI, key-share information, and extensions.
</details>

### 7. What is SNI and why is it important?
<details><summary>Answer</summary>
SNI lets the client indicate the hostname during TLS negotiation, allowing a shared endpoint to select the appropriate certificate/configuration.
</details>

### 8. What is a certificate SAN?
<details><summary>Answer</summary>
SAN lists hostnames/IP identities covered by a certificate. The requested hostname should match an appropriate SAN.
</details>

### 9. What is a certificate chain?
<details><summary>Answer</summary>
It links the server certificate through intermediate CA certificates to a trusted root CA.
</details>

### 10. TLS failure vs HTTP failure?
<details><summary>Answer</summary>
A TLS failure occurs before normal HTTP application data can be exchanged. An HTTP 503 means the client received an HTTP response after TLS was sufficiently established.
</details>

### 11. What does `curl -v` help troubleshoot?
<details><summary>Answer</summary>
It shows useful connection, TLS, request, and response details and helps identify where a request stops.
</details>

### 12. What does `curl -I` do?
<details><summary>Answer</summary>
It requests headers only, useful for quickly checking HTTP status and headers.
</details>

### 13. What does `curl -L` do?
<details><summary>Answer</summary>
It tells curl to follow HTTP redirects such as 301 and 302.
</details>

### 14. What does `curl -k` do?
<details><summary>Answer</summary>
It disables normal TLS certificate verification. It can be useful diagnostically but should not be the permanent production fix.
</details>

### 15. How do you inspect a TLS connection with OpenSSL?
<details><summary>Answer</summary>
Use `openssl s_client -connect example.com:443 -servername example.com </dev/null` to inspect TLS version, cipher, certificates, chain, and verification.
</details>

### 16. How do you distinguish TCP failure from TLS failure using curl?
<details><summary>Answer</summary>
If curl cannot establish TCP, investigate routing/firewalls/port reachability. If TCP connects but TLS fails, investigate certificates, SNI, protocol/cipher compatibility, or TLS configuration.
</details>

### 17. What does HTTP 503 mean?
<details><summary>Answer</summary>
It means the HTTP endpoint reports that the service is unavailable. In an ALB/proxy architecture it can indicate that the proxy cannot currently use a healthy backend.
</details>

### 18. Difference between 502, 503 and 504?
<details><summary>Answer</summary>
502 commonly indicates an unusable upstream response, 503 service unavailable, and 504 a gateway/proxy timeout while waiting for upstream. They are clues, not absolute root-cause labels.
</details>

### 19. HTTPS works to an ALB but the application is down. How?
<details><summary>Answer</summary>
TLS can terminate at the ALB. The ALB can complete TLS with the client while its targets are unhealthy or unreachable.
</details>

### 20. How would you troubleshoot HTTPS in production?
<details><summary>Answer</summary>
Use DNS → TCP/443 → TLS → certificate/SAN/trust → HTTP status → load balancer/proxy → target health → backend port → application. Use `dig`, `nc`, `curl -v`, and `openssl s_client` and identify the first failed layer.
</details>

---

## DAY 4 — Load Balancing: L4 vs L7 and Common Failure Paths

### 1. What is load balancing?
<details><summary>Answer</summary>
Load balancing distributes incoming traffic across multiple backend targets to improve availability, scalability, and fault tolerance.
</details>

### 2. What is the main difference between L4 and L7 load balancing?
<details><summary>Answer</summary>
L4 primarily uses transport information such as IP, TCP/UDP, and ports. L7 understands application information such as HTTP host, path, headers, and methods.
</details>

### 3. Give an example of L4 routing.
<details><summary>Answer</summary>
TCP/443 traffic can be distributed across a backend pool without inspecting HTTP paths such as `/api` or `/payments`.
</details>

### 4. Give an example of L7 routing.
<details><summary>Answer</summary>
An HTTP load balancer can route `/api/*` to one target group and `/admin/*` to another, or route different Host headers to different services.
</details>

### 5. Which AWS load balancer is commonly associated with L7 HTTP/HTTPS routing?
<details><summary>Answer</summary>
Application Load Balancer (ALB) is designed for application-layer HTTP/HTTPS routing.
</details>

### 6. Which AWS load balancer is primarily network/transport oriented?
<details><summary>Answer</summary>
Network Load Balancer (NLB) is primarily designed for high-performance network traffic and connection/transport-oriented load balancing.
</details>

### 7. What is a load balancer listener?
<details><summary>Answer</summary>
A listener waits for traffic on a configured protocol and port, such as an HTTPS listener on port 443, and then applies the configured routing behavior.
</details>

### 8. What is a target group?
<details><summary>Answer</summary>
A target group is a collection of backend targets that can receive traffic from the load balancer. Health status helps determine which targets are usable.
</details>

### 9. What is a load balancer health check?
<details><summary>Answer</summary>
It is a periodic test used to determine whether a backend target can serve the expected traffic. Protocol, port, path, timeout, interval, and expected response are important.
</details>

### 10. Can an EC2 instance be RUNNING while its load-balancer target is UNHEALTHY?
<details><summary>Answer</summary>
Yes. EC2 running state only says the instance is running. The target can still fail health checks because of application, port, path, bind address, security, or network problems.
</details>

### 11. Can an HTTPS connection to an ALB succeed while the backend is down?
<details><summary>Answer</summary>
Yes. TLS can terminate at the ALB. The client-to-ALB connection can be healthy even when the ALB-to-target path is unhealthy.
</details>

### 12. What is a common cause of ALB 503?
<details><summary>Answer</summary>
One common cause is that the target group has no healthy usable targets, for example because the health-check port is wrong or the backend application is unavailable.
</details>

### 13. What is a common cause of 502 from a load balancer?
<details><summary>Answer</summary>
A 502 commonly indicates that a proxy/load balancer received an invalid or unusable upstream response or could not complete the expected backend exchange.
</details>

### 14. What is a common cause of 504?
<details><summary>Answer</summary>
A 504 commonly means a proxy/load balancer waited for an upstream response and timed out. Backend slowness, dependency latency, or connectivity can contribute.
</details>

### 15. How would you troubleshoot DNS works but TCP/443 to the load balancer times out?
<details><summary>Answer</summary>
Check the resolved destination, route, client/network path, listener, Security Groups, NACLs where relevant, and other network controls. Do not start with backend application debugging because TCP to the LB has not been proven.
</details>

### 16. How would you troubleshoot TCP works but TLS fails?
<details><summary>Answer</summary>
Check certificate validity/SAN/chain, SNI, TLS versions/ciphers, and the load balancer TLS listener configuration. Use `curl -v` and `openssl s_client`.
</details>

### 17. How would you troubleshoot TLS works but the ALB returns 503?
<details><summary>Answer</summary>
Check listener rules, target group, target health, health-check path/port/protocol, backend listener, Security Groups, and application logs.
</details>

### 18. What is a backend port mismatch?
<details><summary>Answer</summary>
It occurs when the load balancer/target group expects a port such as 8081 while the application actually listens on another port such as 8080. This can make targets unhealthy or unreachable.
</details>

### 19. Why can one bad target cause intermittent failures?
<details><summary>Answer</summary>
If some targets are healthy and another is unhealthy or behaves differently, requests distributed to the bad target can fail while requests sent to healthy targets succeed.
</details>

### 20. How would you troubleshoot a production application behind a load balancer?
<details><summary>Answer</summary>
Use the complete path: DNS → LB IP → TCP listener → TLS if applicable → HTTP response → listener rules → target group → target health → backend port → bind address → Security Group/firewall → application. Identify the first failed layer and compare healthy vs unhealthy targets.
</details>
