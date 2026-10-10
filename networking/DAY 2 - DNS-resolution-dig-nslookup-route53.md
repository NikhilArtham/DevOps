# DAY 2 — DNS Resolution: `dig`, `nslookup` and Route 53 Concepts

## Why DNS Matters

DNS (Domain Name System) translates human-friendly names such as:

```text
api.example.com
```

into network addresses such as:

```text
203.0.113.10
```

When an application cannot connect to `api.example.com`, DNS is only **one layer** of the troubleshooting path. A successful DNS lookup does not prove that the destination is reachable or that the service port is open.

A useful mental model is:

```text
Application
   ↓
DNS name
   ↓
IP address
   ↓
Routing
   ↓
Network reachability
   ↓
TCP port
   ↓
Listening service
```

---

# 1. What Happens When You Access a Domain?

Suppose you run:

```bash
curl https://api.example.com
```

A simplified flow is:

```text
1. Application needs api.example.com
2. Operating system checks its local DNS sources/cache
3. DNS resolver is contacted if necessary
4. Resolver finds the DNS record
5. Resolver returns an IP address
6. Client uses the IP to establish a network connection
7. TCP connection is established to port 443
8. TLS/HTTP communication begins
```

This is important during troubleshooting because the failure could happen at any of these stages.

---

# 2. DNS Terminology

## Domain name

A human-readable name such as:

```text
www.example.com
```

## DNS resolver

A server that receives a DNS query and finds the answer for the client.

Examples include organizational DNS resolvers, VPC-provided DNS, and public resolvers.

## DNS server / authoritative name server

The authoritative server holds the official DNS records for a DNS zone.

## DNS zone

A logical portion of the DNS namespace managed together.

For example:

```text
example.com
```

can be a DNS zone containing records such as:

```text
www.example.com
api.example.com
mail.example.com
```

---

# 3. Common DNS Record Types

## A record

Maps a hostname to an IPv4 address.

```text
api.example.com → 203.0.113.10
```

Query it with:

```bash
dig A api.example.com
```

## AAAA record

Maps a hostname to an IPv6 address.

```bash
dig AAAA api.example.com
```

## CNAME record

Creates an alias from one DNS name to another name.

Example:

```text
www.example.com → web.example.net
```

Query:

```bash
dig CNAME www.example.com
```

Important: a CNAME points to another **name**, not directly to an IP address.

## MX record

Specifies mail servers for a domain.

```bash
dig MX example.com
```

## NS record

Identifies the authoritative name servers for a domain/zone.

```bash
dig NS example.com
```

## TXT record

Stores text data used by many systems, including domain verification and email security mechanisms such as SPF-related records.

```bash
dig TXT example.com
```

---

# 4. `dig` — The Main DNS Troubleshooting Tool

`dig` means **Domain Information Groper**.

It is one of the most useful Linux commands for investigating DNS.

Basic lookup:

```bash
dig example.com
```

Short answer:

```bash
dig +short example.com
```

Example:

```text
93.184.216.34
```

The `+short` option reduces the output to the useful answer section.

---

# 5. Understand `dig` Output

Run:

```bash
dig example.com
```

Important sections include:

```text
QUESTION SECTION
ANSWER SECTION
AUTHORITY SECTION
ADDITIONAL SECTION
```

The **ANSWER SECTION** contains the DNS answer returned for the query.

You may also see:

```text
status: NOERROR
```

This indicates that the DNS server successfully processed the query. It does not necessarily mean the requested name has an address record.

For example, `NXDOMAIN` indicates that the queried domain name does not exist according to the responding DNS server.

---

# 6. Query a Specific DNS Resolver

Normally `dig` uses the resolver configured for the system.

You can explicitly select a DNS server:

```bash
dig @8.8.8.8 example.com
```

Format:

```text
 dig @<dns-server> <domain>
```

Example:

```bash
dig @1.1.1.1 example.com
```

This is extremely useful when diagnosing:

- Local resolver problems
- Corporate DNS problems
- VPC DNS problems
- Different answers from different resolvers
- DNS propagation issues

---

# 7. Query Specific Record Types

```bash
dig A example.com
```

```bash
dig AAAA example.com
```

```bash
dig MX example.com
```

```bash
dig NS example.com
```

```bash
dig TXT example.com
```

```bash
dig CNAME www.example.com
```

This is better than assuming every DNS problem is simply an A-record problem.

---

# 8. Trace DNS Delegation

`dig` can trace the DNS resolution hierarchy:

```bash
dig +trace example.com
```

Conceptually, DNS resolution can move through:

```text
Root DNS servers
      ↓
.com DNS servers
      ↓
authoritative servers for example.com
      ↓
example.com answer
```

`+trace` is useful when you want to understand where delegation or authoritative resolution is failing.

---

# 9. `nslookup`

`nslookup` is another DNS query utility.

Basic lookup:

```bash
nslookup example.com
```

Specify a DNS server:

```bash
nslookup example.com 8.8.8.8
```

Interactive mode:

```bash
nslookup
```

Then:

```text
> server 8.8.8.8
> example.com
```

### `dig` vs `nslookup`

For modern Linux troubleshooting, prefer learning `dig` deeply because it exposes detailed DNS response information and is convenient for scripting and troubleshooting.

You should still recognize `nslookup` because it is common in existing environments, Windows troubleshooting, documentation, and interviews.

---

# 10. DNS Failure vs Network Failure

Suppose:

```bash
curl https://api.example.com
```

fails.

Do not immediately say:

```text
Firewall problem.
```

First check DNS:

```bash
dig +short api.example.com
```

If it returns an IP:

```text
10.20.30.40
```

DNS is at least returning an answer.

Now test routing:

```bash
ip route get 10.20.30.40
```

Then test network reachability where appropriate:

```bash
ping 10.20.30.40
```

Then test the actual service port:

```bash
nc -vz 10.20.30.40 443
```

Then inspect the destination listener:

```bash
sudo ss -lntp | grep ':443'
```

The important lesson is:

```text
DNS working ≠ network working
Network working ≠ port working
Port open ≠ application working
```

---

# 11. DNS Caching and TTL

DNS responses can be cached.

A DNS record commonly has a **TTL (Time To Live)**.

Example:

```text
TTL = 300
```

This means a resolver can generally cache the response for the specified period before it needs to obtain fresh information.

You can see TTL information with:

```bash
dig example.com
```

Caching explains why changing a DNS record does not necessarily make every client see the new answer immediately.

Important troubleshooting point:

```text
DNS record changed
        ↓
Some clients may still have cached data
        ↓
Different clients/resolvers may temporarily return different answers
```

---

# 12. Route 53 — AWS DNS Service

Amazon Route 53 is AWS's highly available DNS and domain-management service.

At a basic level, Route 53 can:

- Host DNS zones
- Store DNS records
- Register/manage domains
- Route DNS queries using routing policies
- Perform health checks

A common AWS architecture looks like:

```text
User
 ↓
DNS query
 ↓
Route 53
 ↓
AWS resource / endpoint
 ↓
Application
```

---

# 13. Route 53 Hosted Zones

A **hosted zone** is a container for DNS records for a domain.

For example:

```text
example.com
```

could have records:

```text
example.com
api.example.com
www.example.com
```

There are two important types.

## Public hosted zone

Used for DNS names that should be resolvable on the public internet.

Example:

```text
api.example.com
```

resolving to a public endpoint.

## Private hosted zone

Used for DNS names inside one or more associated VPCs.

Example:

```text
db.internal.example.com
```

A private hosted zone is useful for internal service discovery without exposing the name publicly.

---

# 14. Route 53 Alias Records

An Alias record is an AWS-specific DNS feature that can point a DNS name to supported AWS resources.

Common examples include:

```text
Route 53
   ↓
Application Load Balancer
```

or:

```text
Route 53
   ↓
CloudFront
```

Alias records are especially important in AWS because they allow DNS names to target supported AWS resources without treating the AWS resource as a traditional hostname-only CNAME target.

---

# 15. Route 53 Routing Policies

Route 53 supports different routing strategies.

Important ones to know:

### Simple routing

Basic DNS response for a resource.

### Weighted routing

Distributes traffic according to assigned weights.

Example:

```text
Version A → 90%
Version B → 10%
```

Useful for gradual releases and testing.

### Latency-based routing

Routes users toward the AWS region expected to provide the lowest network latency.

### Failover routing

Provides primary/secondary behavior.

Example:

```text
Primary → production
Secondary → disaster-recovery environment
```

### Geolocation routing

Routes based on the geographic location of the requester.

### Geoproximity routing

Routes based on geographic proximity and configurable bias.

### Multivalue answer routing

Can return multiple healthy resource values and is useful for simple DNS-level availability distribution.

---

# 16. Route 53 Health Checks

Route 53 health checks can monitor endpoints and influence certain routing decisions.

Conceptually:

```text
Route 53
   ↓
Health check
   ↓
Is endpoint healthy?
   ↓
Choose appropriate DNS response
```

A health check is not the same thing as proving that every part of an application is healthy. It checks the configured health-check target and conditions.

---

# 17. AWS Private DNS Mental Model

For internal AWS services, think about DNS and networking separately.

Example:

```text
client EC2
   ↓
private DNS name
   ↓
VPC DNS resolution
   ↓
private IP
   ↓
route
   ↓
Security Group / NACL
   ↓
service
```

If the private name resolves correctly but the connection fails, do not continue troubleshooting DNS forever. Move to routing, security controls, port listening, and the application.

---

# 18. Production DNS Troubleshooting Flow

When:

```text
https://api.example.com
```

is unreachable, use this sequence.

### Step 1 — Does the name resolve?

```bash
dig +short api.example.com
```

### Step 2 — Which DNS server answered?

```bash
dig api.example.com
```

### Step 3 — Compare another resolver

```bash
dig @8.8.8.8 api.example.com
```

### Step 4 — Check the route to the returned IP

```bash
ip route get <destination-ip>
```

### Step 5 — Test reachability where appropriate

```bash
ping <destination-ip>
```

Remember that failed ping does not automatically prove the host is unreachable.

### Step 6 — Test the actual application port

```bash
nc -vz <destination-ip> 443
```

### Step 7 — Check the destination listener

```bash
sudo ss -lntp | grep ':443'
```

### Step 8 — Check security controls

On AWS, investigate the relevant:

- Security Group
- Network ACL
- Route table
- Subnet/VPC path
- Linux firewall

### Step 9 — Check the application

If DNS, routing, TCP connectivity, and security controls are working, investigate the application and TLS/HTTP layer.

---

# 19. Common DNS Mistakes

### Mistake 1 — Assuming DNS is the same as connectivity

DNS only answers the question:

```text
What address does this name resolve to?
```

It does not prove the service is reachable.

### Mistake 2 — Testing only `ping hostname`

This combines name resolution and ICMP testing. If it fails, you do not immediately know which layer failed.

Better:

```bash
dig +short hostname
ip route get <ip>
nc -vz <ip> <port>
```

### Mistake 3 — Testing only one DNS resolver

Different resolvers can temporarily have different cached information or configuration.

### Mistake 4 — Assuming `NXDOMAIN` means the server is down

`NXDOMAIN` is a DNS response indicating that the queried name does not exist according to the responding DNS system. It is not a server-health check.

### Mistake 5 — Confusing Route 53 with AWS networking

Route 53 answers DNS questions. Security Groups, NACLs, route tables, and other networking components control different parts of the traffic path.

---

# 20. Interview-Ready Mental Model

Remember this:

```text
DNS answers:
"What IP/name should I use?"

Routing answers:
"Where should packets go?"

Security controls answer:
"Is this traffic allowed?"

TCP answers:
"Can I establish a connection to this port?"

The application answers:
"Can I actually serve the request?"
```

For production troubleshooting, always identify the **first failed layer** instead of guessing.

---

## Beginner Takeaway

If you remember only a few commands from this topic, remember:

```bash
dig +short example.com

dig example.com

dig @8.8.8.8 example.com

nslookup example.com

ip route get <destination-ip>

nc -vz <destination-ip> <port>

sudo ss -lntp
```

Your basic troubleshooting thought process should become:

```text
Name
 ↓
DNS
 ↓
IP
 ↓
Route
 ↓
Port
 ↓
Listener
 ↓
Security controls
 ↓
Application
```

**Labs and interview questions for this topic are maintained in the centralized Networking files.**
