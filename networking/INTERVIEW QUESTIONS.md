# Networking Interview Questions

Centralized interview questions for Networking. Questions from every Networking DAY are added here.

---

## DAY 1 — TCP/IP, Ports, Sockets and `ss`/`netstat`

### 1. What is an IP address?

<details><summary>Answer</summary>
An IP address identifies a network interface on a network so traffic can be routed to the correct destination. In troubleshooting, an application connection commonly looks like `10.0.2.15:8080`, where the IP identifies the host and the port identifies the service endpoint.
</details>

### 2. What is the difference between an IP address and a port?

<details><summary>Answer</summary>
The IP address identifies the network destination, while the port identifies a particular service or socket on that destination. For example, `10.0.1.10:443` means TCP traffic to port 443 on host `10.0.1.10`.
</details>

### 3. What is a TCP three-way handshake?

<details><summary>Answer</summary>
TCP normally establishes a connection using SYN → SYN-ACK → ACK. This synchronizes the endpoints before application data is exchanged.
</details>

### 4. What is a socket?

<details><summary>Answer</summary>
A socket is a network communication endpoint. A TCP connection can be identified using protocol, source IP, source port, destination IP, and destination port.
</details>

### 5. What is the difference between LISTEN and ESTABLISHED in `ss`?

<details><summary>Answer</summary>
`LISTEN` means a process is waiting for incoming TCP connections. `ESTABLISHED` means a TCP connection has already been successfully established between two endpoints.
</details>

### 6. Why is `ss` preferred over `netstat` on modern Linux?

<details><summary>Answer</summary>
`ss` is the modern socket-statistics utility and is generally available on current Linux systems. `netstat` comes from the older `net-tools` package but is still common in legacy systems and interviews.
</details>

### 7. What does `ss -lntp` mean?

<details><summary>Answer</summary>
`-l` shows listening sockets, `-n` keeps addresses and ports numeric, `-t` selects TCP sockets, and `-p` shows the process using the socket.
</details>

### 8. What is the difference between `127.0.0.1` and `0.0.0.0`?

<details><summary>Answer</summary>
`127.0.0.1` is the loopback address, so a service bound only to it normally accepts local connections. `0.0.0.0` means the service is listening on all IPv4 interfaces, subject to firewall and network controls.
</details>

### 9. Does successful `ping` prove that TCP port 443 is reachable?

<details><summary>Answer</summary>
No. `ping` tests ICMP reachability, while HTTPS uses TCP port 443. ICMP can succeed while TCP/443 is blocked or unavailable.
</details>

### 10. Does failed `ping` prove that a server is unreachable?

<details><summary>Answer</summary>
No. ICMP may be blocked or rate-limited. Test the actual application port with `nc`, `curl`, or the application client.
</details>

### 11. What is the difference between connection refused and connection timed out?

<details><summary>Answer</summary>
A refusal commonly indicates that the destination actively rejected the connection, often because no service is listening. A timeout means the expected response was not received in time and can indicate filtering, routing, host availability, or other path problems.
</details>

### 12. How do you check which process is listening on port 8080?

<details><summary>Answer</summary>
Use `sudo ss -lntp | grep ':8080'`. The `-p` option exposes process information.
</details>

### 13. How do you check which route Linux will use to reach an IP?

<details><summary>Answer</summary>
Use `ip route get <destination-ip>`. It shows the route Linux would select for that specific destination.
</details>

### 14. How do you troubleshoot DNS separately from connectivity?

<details><summary>Answer</summary>
First use `dig +short example.com` to resolve the name. Then test the returned IP separately with routing and TCP tools such as `ip route get` and `nc -vz`.
</details>

### 15. What is `traceroute` used for?

<details><summary>Answer</summary>
`traceroute` shows intermediate network hops and can help identify where a path may become problematic. A `*` at a hop does not automatically mean that router is broken.
</details>

### 16. How would you troubleshoot an unreachable service from a client to an EC2 instance?

<details><summary>Answer</summary>
Use a layered approach: DNS → route → reachability → TCP port → server listener → bind address → Linux firewall → Security Group → Network ACL/AWS routing → application.
</details>

### 17. An application is listening on port 8080, but clients cannot connect. What do you check next?

<details><summary>Answer</summary>
First inspect the listening address. If it is `127.0.0.1:8080`, remote clients normally cannot connect. Then check firewall, Security Group, NACL, routing, and client-side TCP testing.
</details>

### 18. What is the difference between an AWS Security Group and a Linux firewall?

<details><summary>Answer</summary>
A Security Group is an AWS-level network control associated with resources/network interfaces. A Linux firewall operates inside the operating system. Both can affect whether traffic reaches an application.
</details>

### 19. What is the difference between a Security Group and a Network ACL?

<details><summary>Answer</summary>
A Security Group is associated with resources/network interfaces and is stateful. A Network ACL operates at subnet level and is stateless, so inbound and outbound rules must both be considered.
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

### 2. What is the difference between a DNS resolver and an authoritative DNS server?

<details><summary>Answer</summary>
A resolver receives a client's query and finds an answer, often using cache or querying other DNS servers. An authoritative server holds the official records for a DNS zone.
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
A CNAME creates an alias from one DNS name to another DNS name. It points to a name rather than directly to an IP address.
</details>

### 6. What is the difference between `dig` and `nslookup`?

<details><summary>Answer</summary>
Both query DNS. `dig` generally provides richer DNS response details and is widely used for Linux troubleshooting. `nslookup` remains common in existing environments and interviews.
</details>

### 7. What does `dig +short example.com` do?

<details><summary>Answer</summary>
It performs a DNS lookup and reduces the output to the short answer, useful when you mainly want the returned address or record value.
</details>

### 8. How do you query a specific DNS resolver using `dig`?

<details><summary>Answer</summary>
Use `dig @8.8.8.8 example.com`. The `@8.8.8.8` portion selects the DNS server.
</details>

### 9. What does `dig +trace` do?

<details><summary>Answer</summary>
It follows DNS delegation starting from the root and helps show how the query reaches authoritative DNS servers.
</details>

### 10. What is TTL in DNS?

<details><summary>Answer</summary>
TTL controls how long a DNS response can normally be cached by a resolver. Cached data can remain visible until the TTL expires.
</details>

### 11. What is `NXDOMAIN`?

<details><summary>Answer</summary>
`NXDOMAIN` indicates that the queried domain name does not exist according to the responding DNS system.
</details>

### 12. How do you separate a DNS problem from a connectivity problem?

<details><summary>Answer</summary>
Resolve the name independently with `dig +short`, then test the returned IP with routing and TCP tools. If DNS returns a valid IP but TCP fails, investigate networking and security controls.
</details>

### 13. What is a Route 53 hosted zone?

<details><summary>Answer</summary>
A Route 53 hosted zone is a container for DNS records for a domain or DNS namespace. It can be public or private.
</details>

### 14. What is the difference between a public and private Route 53 hosted zone?

<details><summary>Answer</summary>
A public hosted zone is intended for public DNS resolution. A private hosted zone provides DNS records for associated VPCs and internal service names.
</details>

### 15. What is a Route 53 Alias record?

<details><summary>Answer</summary>
An Alias record is an AWS-specific DNS feature that can point a DNS name to supported AWS resources such as an ALB or CloudFront distribution.
</details>

### 16. What Route 53 routing policies should a DevOps engineer know?

<details><summary>Answer</summary>
Know simple, weighted, latency-based, failover, geolocation, geoproximity, and multivalue answer routing, including why each is used.
</details>

### 17. What is weighted routing in Route 53?

<details><summary>Answer</summary>
Weighted routing distributes DNS responses according to configured weights, such as 90% to one version and 10% to another.
</details>

### 18. What is latency-based routing in Route 53?

<details><summary>Answer</summary>
Latency-based routing directs users toward the AWS region expected to provide the lowest network latency among configured resources.
</details>

### 19. What is failover routing in Route 53?

<details><summary>Answer</summary>
Failover routing supports a primary/secondary model and can use health-check results to influence which resource is returned.
</details>

### 20. A DNS name resolves successfully but the application is unreachable. What do you check?

<details><summary>Answer</summary>
Treat DNS as a completed layer and continue through IP → route → reachability → TCP port → listener → firewall/Security Group/NACL → application. Identify the first failed layer with evidence.
</details>

---

## DAY 3 — HTTP/HTTPS, TLS Handshake and `curl` Troubleshooting

### 1. What is HTTP?

<details><summary>Answer</summary>
HTTP is an application-layer protocol used for client-server communication. A request contains items such as method, path, headers, and optionally a body; the server returns a status code, headers, and optionally a body.
</details>

### 2. What is HTTPS?

<details><summary>Answer</summary>
HTTPS is HTTP carried over a TLS-protected connection. A common HTTPS endpoint uses TCP port 443.
</details>

### 3. What is the difference between HTTP and HTTPS?

<details><summary>Answer</summary>
HTTP sends application traffic without TLS protection. HTTPS adds TLS, providing confidentiality, integrity, and server authentication before normal HTTP application data is exchanged.
</details>

### 4. What happens before an HTTPS HTTP request is sent?

<details><summary>Answer</summary>
Typically the client resolves the name, establishes TCP, performs the TLS handshake and certificate validation, and only then sends HTTP application data.
</details>

### 5. What is a TLS handshake?

<details><summary>Answer</summary>
The TLS handshake negotiates security parameters, establishes shared key material, and authenticates the server using certificates so encrypted application traffic can begin.
</details>

### 6. What is `ClientHello`?

<details><summary>Answer</summary>
`ClientHello` starts TLS negotiation and can include supported TLS versions, cipher suites, SNI, key-share information, and other extensions.
</details>

### 7. What is SNI and why is it important?

<details><summary>Answer</summary>
SNI (Server Name Indication) lets the client indicate the hostname it wants during TLS negotiation. This allows a shared IP/load balancer to select the appropriate certificate and TLS configuration for multiple HTTPS domains.
</details>

### 8. What is a certificate SAN?

<details><summary>Answer</summary>
SAN means Subject Alternative Name. It lists hostnames/IP identities covered by a certificate. The requested hostname should match an appropriate SAN during certificate validation.
</details>

### 9. What is a certificate chain?

<details><summary>Answer</summary>
A certificate chain links the server certificate through intermediate CA certificates to a trusted root CA. Clients use their trust store to validate that chain.
</details>

### 10. What is the difference between a TLS failure and an HTTP failure?

<details><summary>Answer</summary>
A TLS failure occurs before normal HTTP application data can be exchanged. An HTTP failure such as `503` means TLS completed sufficiently for the client to receive an HTTP response.
</details>

### 11. What does `curl -v` help you troubleshoot?

<details><summary>Answer</summary>
`curl -v` shows useful connection, TLS, request, and response details. It helps identify whether a request fails during DNS/connection, TLS, HTTP request, or HTTP response processing.
</details>

### 12. What does `curl -I` do?

<details><summary>Answer</summary>
`curl -I` requests headers only, making it useful for quickly checking HTTP status codes and response headers.
</details>

### 13. What does `curl -L` do?

<details><summary>Answer</summary>
`-L` tells curl to follow HTTP redirects such as `301` and `302` responses.
</details>

### 14. What does `curl -k` do and why should it not be the permanent fix?

<details><summary>Answer</summary>
`-k` disables normal TLS certificate verification. It can be useful diagnostically to determine whether certificate verification is causing a failure, but using it permanently weakens HTTPS security.
</details>

### 15. How do you inspect a TLS connection with OpenSSL?

<details><summary>Answer</summary>
Use `openssl s_client -connect example.com:443 -servername example.com </dev/null`. This can expose the negotiated TLS version, cipher, certificate chain, verification result, and other handshake information.
</details>

### 16. How can you distinguish TCP failure from TLS failure using curl?

<details><summary>Answer</summary>
Run `curl -v https://host`. If it cannot establish the TCP connection, investigate routing/firewalls/port reachability. If it connects but fails during TLS, investigate certificates, SNI, protocol/cipher compatibility, or TLS endpoint configuration.
</details>

### 17. What does an HTTP 503 mean?

<details><summary>Answer</summary>
It means the HTTP endpoint is reporting that the service is unavailable. In an ALB/proxy architecture, it can indicate that the proxy cannot currently use a healthy backend. The exact cause must be investigated.
</details>

### 18. What is the difference between 502, 503 and 504?

<details><summary>Answer</summary>
`502` commonly indicates a bad upstream response, `503` indicates service unavailable, and `504` commonly indicates a gateway/proxy timeout while waiting for an upstream. These are troubleshooting clues rather than absolute root-cause labels.
</details>

### 19. HTTPS works to an ALB, but the application is down. How is that possible?

<details><summary>Answer</summary>
TLS may terminate at the ALB. The ALB can successfully complete TLS with the client while its target group has unhealthy targets or cannot connect to the backend application. Always troubleshoot the ALB-to-target path separately.
</details>

### 20. How would you troubleshoot an HTTPS service that is failing in production?

<details><summary>Answer</summary>
Use a layered flow: DNS → TCP/443 → TLS handshake → certificate/SAN/trust → HTTP status → load balancer/proxy → target health → backend port → application. Useful commands include `dig`, `nc`, `curl -v`, and `openssl s_client`. Identify the first failed layer and collect evidence rather than guessing.
</details>
