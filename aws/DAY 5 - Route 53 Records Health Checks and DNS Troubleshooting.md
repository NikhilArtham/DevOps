# DAY 5 — Route 53: Records, Health Checks and DNS Troubleshooting

## 1. Why This Topic Matters

Route 53 is often the first AWS component involved when a user says:

```text
The application is down.
The URL does not open.
DNS is not resolving.
The service works by IP but not by hostname.
```

DNS problems are easy to confuse with networking problems.

A production DevOps engineer should separate:

```text
DNS resolution
↓
IP routing
↓
Port reachability
↓
Firewall / Security Group / NACL
↓
TLS
↓
HTTP
↓
Application
```

The key question is:

> Did the hostname resolve, and if it did, did the client reach the returned IP and port?

---

## 2. What Is Route 53?

Amazon Route 53 is AWS's highly available DNS service.

It can provide:

- DNS hosting
- domain registration
- health checks
- DNS-based routing policies
- public hosted zones
- private hosted zones

A simplified request looks like:

```text
User
 ↓
DNS resolver
 ↓
Route 53
 ↓
DNS record
 ↓
IP / AWS endpoint
 ↓
Application path
```

Route 53 does not replace a load balancer, route table, Security Group, or application. It primarily answers the question:

```text
What destination should this hostname resolve to?
```

---

## 3. DNS Record Fundamentals

A DNS record maps a name to information about a service or domain.

Common records:

```text
A       → IPv4 address
AAAA    → IPv6 address
CNAME   → another hostname
MX      → mail servers
TXT     → text-based information
NS      → authoritative name servers
SOA     → zone authority information
```

Example:

```text
api.example.com
        ↓
A
        ↓
203.0.113.10
```

For IPv6:

```text
api.example.com
        ↓
AAAA
        ↓
2001:db8::10
```

For an alias-style AWS target:

```text
example.com
      ↓
Alias
      ↓
ALB DNS name
```

---

## 4. A Record

An A record maps a hostname to an IPv4 address.

Example:

```text
api.example.com → 10.0.10.25
```

For a public DNS record, the returned IP must be reachable from the client according to the network architecture.

Important:

```text
DNS resolution success
≠
Application connectivity success
```

If `dig` returns an IP but `curl` times out, DNS may be working perfectly.

---

## 5. AAAA Record

An AAAA record maps a hostname to an IPv6 address.

Example:

```text
api.example.com → 2001:db8::25
```

A common troubleshooting trap is that IPv6 resolution works while the application's IPv6 path is broken.

Test both families when necessary:

```bash
curl -4 -v https://api.example.com
curl -6 -v https://api.example.com
```

If `-4` works and `-6` fails, do not immediately blame Route 53. The DNS record may be correct while the IPv6 routing or application path is not.

---

## 6. CNAME Record

A CNAME maps one hostname to another hostname.

Example:

```text
www.example.com
        ↓
CNAME
        ↓
app.example.net
```

The resolver then follows the target name to obtain the final address information.

Use CNAME when the destination is another hostname.

For AWS resources, Route 53 **alias records** are often preferred where supported because they can point directly to supported AWS resources such as load balancers.

---

## 7. Route 53 Alias Records

An alias record can point a Route 53 name to supported AWS resources.

Common examples include:

```text
example.com
   ↓
Alias
   ↓
ALB
```

or:

```text
app.example.com
   ↓
Alias
   ↓
CloudFront distribution
```

Mental model:

```text
DNS name
 ↓
Route 53 alias
 ↓
AWS resource endpoint
```

An alias is not the same thing as a normal CNAME. It is an AWS Route 53 feature with special integration for supported AWS resources.

---

## 8. Hosted Zones

A hosted zone contains DNS records for a domain or namespace.

Two important types are:

```text
Public hosted zone
Private hosted zone
```

### Public hosted zone

Used for names that should resolve through public DNS.

Example:

```text
api.example.com
```

### Private hosted zone

Used for DNS names inside associated VPCs.

Example:

```text
api.internal.example.com
```

A private hosted zone can allow internal workloads to resolve private service names without exposing those names through public DNS.

---

## 9. DNS Resolution Mental Model

When a client runs:

```bash
curl https://api.example.com
```

DNS resolution happens before the HTTP connection.

Simplified flow:

```text
Application
 ↓
OS resolver
 ↓
Configured DNS resolver
 ↓
DNS hierarchy / Route 53 authoritative DNS
 ↓
DNS answer
 ↓
Returned IP
 ↓
TCP connection
 ↓
TLS
 ↓
HTTP
```

This is why DNS troubleshooting should happen before spending time on application logs.

---

## 10. `dig` — Primary DNS Troubleshooting Tool

`dig` gives detailed DNS information.

Basic lookup:

```bash
dig api.example.com
```

Short answer:

```bash
dig +short api.example.com
```

Query a specific record type:

```bash
dig api.example.com A
dig api.example.com AAAA
dig api.example.com CNAME
dig example.com MX
```

Query a specific DNS server:

```bash
dig @8.8.8.8 api.example.com
```

Query Route 53 resolver from an AWS environment where appropriate:

```bash
dig @<resolver-ip> api.internal.example.com
```

### Important flags

```text
@server       → choose the DNS server to query
+short        → return concise answer
+trace        → trace DNS delegation from root
```

Example:

```bash
dig +trace api.example.com
```

This helps determine where delegation or authoritative resolution is failing.

---

## 11. `nslookup`

`nslookup` is another DNS troubleshooting utility.

Basic lookup:

```bash
nslookup api.example.com
```

Specify a resolver:

```bash
nslookup api.example.com 8.8.8.8
```

It is useful for quick checks, while `dig` generally provides more detailed and script-friendly DNS information.

---

## 12. DNS Answer vs DNS Failure

### DNS succeeds

```bash
dig +short api.example.com
```

Output:

```text
203.0.113.20
```

This proves that the queried resolver returned an address.

It does **not** prove:

- the route is correct
- port 443 is reachable
- Security Groups allow traffic
- the application is listening
- TLS is valid
- HTTP is healthy

### DNS fails

Possible outputs include:

```text
NXDOMAIN
SERVFAIL
no answer
```

These have different meanings and should not be treated as the same failure.

---

## 13. NXDOMAIN

`NXDOMAIN` means the queried DNS name does not exist according to the DNS response.

Example:

```text
api2.example.com
        ↓
NXDOMAIN
```

Possible causes:

- typo in hostname
- missing record
- incorrect DNS delegation
- querying the wrong DNS namespace
- private name queried through public DNS

Do not start debugging Security Groups when the hostname does not exist in DNS.

---

## 14. SERVFAIL

`SERVFAIL` means the resolver could not successfully complete the DNS query.

Potential causes include:

- authoritative DNS problems
- DNSSEC-related issues
- delegation problems
- temporary resolver failures
- broken upstream resolution

The correct next step is to identify where the failure occurs rather than assuming the application is down.

Useful commands:

```bash
dig api.example.com
dig +trace api.example.com
```

Compare multiple resolvers if appropriate:

```bash
dig @8.8.8.8 api.example.com
dig @1.1.1.1 api.example.com
```

---

## 15. TTL and DNS Caching

TTL means **Time To Live**.

It tells DNS resolvers and caches how long an answer can be cached according to DNS behavior.

Example:

```text
api.example.com
TTL = 300
```

A record change may therefore not appear everywhere immediately because recursive resolvers can continue using cached data until the relevant TTL expires.

Production lesson:

> A DNS change being correct in Route 53 does not guarantee every client immediately sees the new answer.

When troubleshooting a recent DNS change, compare:

```text
Route 53 record
↓
Authoritative answer
↓
Recursive resolver answer
↓
Client resolver answer
```

---

## 16. Route 53 Routing Policies

Route 53 can choose answers using different routing policies.

Important policies include:

```text
Simple
Weighted
Latency-based
Failover
Geolocation
Geoproximity
Multivalue answer
```

### Simple

Returns the configured record answer without advanced routing logic.

### Weighted

Splits traffic according to configured weights.

Example:

```text
api-v1 → weight 90
api-v2 → weight 10
```

Useful for controlled traffic distribution.

### Latency-based

Returns the resource associated with the AWS Region that Route 53 determines provides the lowest latency for the requester.

### Failover

Uses primary/secondary behavior based on health evaluation.

Example:

```text
Primary → healthy → return primary
Primary → unhealthy → return secondary
```

### Geolocation

Routes based on the location from which the DNS query originates.

### Geoproximity

Routes based on geographic location and configured bias.

### Multivalue answer

Can return multiple healthy values for a DNS name and is useful for simple availability-oriented DNS responses.

Do not confuse Route 53 DNS routing with application-layer load balancing. Route 53 chooses DNS answers; an ALB/NLB distributes network traffic after the client connects to the returned endpoint.

---

## 17. Route 53 Health Checks

A Route 53 health check can monitor an endpoint and can be used with supported DNS routing configurations.

Conceptually:

```text
Route 53 health check
        ↓
Endpoint health
        ↓
DNS routing decision
```

A health check can evaluate an endpoint using supported protocols and health-check configuration.

The important production distinction is:

```text
Health check says endpoint is healthy
≠
Every user request will succeed
```

The health check tests according to its configured path, protocol, port, and expectations. It may not reproduce the exact user request.

---

## 18. Health Check Failure Scenario

Suppose:

```text
api.example.com
      ↓
Route 53 failover routing
      ↓
Primary ALB
```

The application is healthy for users, but the Route 53 health check calls:

```text
https://api.example.com/health
```

and the application now requires authentication for `/health`.

The health check fails.

Route 53 may therefore consider the endpoint unhealthy and change DNS behavior according to the configured routing policy.

The lesson is the same as load-balancer health checks:

```text
Health-check path
must represent real service health
```

but it should also be designed so the health checker can actually access it.

---

## 19. DNS vs Routing vs Port vs Firewall

Use this decision tree:

```text
Hostname fails to resolve?
        ↓ YES
      DNS problem

        ↓ NO

Resolved IP reachable?
        ↓ NO
      Routing / network path problem

        ↓ YES

TCP port reachable?
        ↓ NO
      Port / listener / firewall problem

        ↓ YES

TLS succeeds?
        ↓ NO
      Certificate / SNI / TLS problem

        ↓ YES

HTTP response correct?
        ↓ NO
      Load balancer / application problem
```

The goal is to identify the failing layer instead of changing everything at once.

---

## 20. Linux Commands for End-to-End DNS Troubleshooting

### Check local IP

```bash
ip addr
```

### Check routing

```bash
ip route
ip route get <destination-ip>
```

`ip route get` is especially useful because it shows how the Linux kernel would route traffic to a specific destination.

### Check listening ports

```bash
ss -lntp
```

Useful flags:

```text
-l → listening sockets
-n → numeric addresses/ports
-t → TCP
-p → owning process
```

### Test ICMP reachability

```bash
ping <destination-ip>
```

Remember:

```text
ping failure
≠
TCP failure
```

ICMP may be blocked even when TCP/HTTPS works.

### Trace network path

```bash
traceroute <destination-ip>
```

### Test TCP port

```bash
nc -vz <destination-ip> 443
```

### Test HTTP/HTTPS

```bash
curl -v https://api.example.com
```

### Test DNS

```bash
dig +short api.example.com
dig api.example.com
```

---

## 21. End-to-End Troubleshooting Example

Incident:

```text
Users report:
https://api.example.com is unavailable
```

### Step 1 — DNS

```bash
dig +short api.example.com
```

If no address is returned, investigate Route 53/DNS first.

### Step 2 — Route

If DNS returns:

```text
203.0.113.20
```

check the client route:

```bash
ip route get 203.0.113.20
```

### Step 3 — TCP port

```bash
nc -vz 203.0.113.20 443
```

If this times out, investigate network path, Security Groups, NACLs, firewalls, or listener availability.

### Step 4 — TLS

```bash
openssl s_client \
  -connect 203.0.113.20:443 \
  -servername api.example.com
```

Check certificate and handshake behavior.

### Step 5 — HTTP

```bash
curl -v https://api.example.com
```

Now inspect HTTP status and headers.

### Step 6 — AWS load balancer

If the endpoint is an ALB/NLB, inspect:

```bash
aws elbv2 describe-load-balancers
aws elbv2 describe-listeners --load-balancer-arn <arn>
aws elbv2 describe-target-groups
aws elbv2 describe-target-health --target-group-arn <arn>
```

### Step 7 — Application

On the target:

```bash
ss -lntp
curl -v http://127.0.0.1:<port>/health
```

This isolates the failing layer instead of guessing.

---

## 22. AWS CLI Commands for Route 53

List hosted zones:

```bash
aws route53 list-hosted-zones
```

List records:

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id <hosted-zone-id>
```

List health checks:

```bash
aws route53 list-health-checks
```

Get health-check details:

```bash
aws route53 get-health-check \
  --health-check-id <health-check-id>
```

Get health-check status:

```bash
aws route53 get-health-check-status \
  --health-check-id <health-check-id>
```

Inspect DNS configuration with AWS CLI, then validate actual DNS behavior separately using `dig`.

Important distinction:

```text
AWS CLI → what is configured

dig     → what DNS resolution returns
```

Both are useful during an incident.

---

## 23. Route 53 + ALB Architecture

A very common production architecture is:

```text
User
 ↓
Route 53
 ↓
ALB
 ↓
Target Group
 ↓
EC2 / ECS / EKS
 ↓
Application
```

Example:

```text
api.example.com
       ↓
Route 53 alias
       ↓
ALB :443
       ↓
Listener rule
       ↓
API target group
       ↓
Application :8080
```

Each layer has a different responsibility:

```text
Route 53 → DNS answer
ALB      → HTTP/HTTPS routing
Target   → backend availability
SG/NACL  → traffic filtering
Application → actual service behavior
```

This separation is extremely useful during production troubleshooting.

---

## 24. EKS / Kubernetes Connection

A common EKS path can look like:

```text
Client
 ↓
Route 53
 ↓
ALB / NLB
 ↓
Kubernetes Ingress / Service
 ↓
Pod
 ↓
Application
```

DNS can therefore be healthy while Kubernetes is broken.

For example:

```text
api.example.com resolves correctly
        ↓
ALB receives traffic
        ↓
Target/Pod is unhealthy
        ↓
503
```

When troubleshooting EKS, separate:

```text
DNS
AWS load balancer
Kubernetes routing
Pod networking
Application
```

---

## 25. Common DNS Troubleshooting Mistakes

### Mistake 1 — DNS resolves, so the application must be healthy

Wrong.

DNS only proves that a DNS answer was returned.

### Mistake 2 — `ping` fails, so the server is down

Wrong.

ICMP can be blocked while TCP/HTTPS works.

### Mistake 3 — Opening a Security Group to fix DNS

Wrong.

If the hostname does not resolve, investigate DNS first.

### Mistake 4 — Changing DNS TTL during an incident without understanding caching

This does not instantly clear every recursive cache.

### Mistake 5 — Testing only the hostname

Test the exact path:

```text
DNS
→ IP
→ route
→ TCP port
→ TLS
→ HTTP
```

### Mistake 6 — Assuming Route 53 health checks reproduce user traffic

They only test the configured health-check behavior.

### Mistake 7 — Forgetting public vs private DNS

A private hosted zone requires the correct VPC association and resolver path.

---

## 26. Realistic Production Failure Scenario

### Incident

Users report:

```text
https://api.example.com
sometimes works and sometimes fails.
```

Architecture:

```text
Client
 ↓
Route 53 weighted records
 ├── 90% → ALB-A
 └── 10% → ALB-B
```

ALB-B was recently deployed.

### Step 1 — DNS

Run:

```bash
dig +short api.example.com
```

Different answers are observed over time.

This is expected with weighted DNS and caching; the important question is whether each returned destination is healthy.

### Step 2 — Test each returned destination

For each ALB endpoint, test:

```bash
curl -v https://<endpoint>
```

ALB-A works.

ALB-B returns:

```text
503 Service Unavailable
```

### Step 3 — Inspect ALB-B

```bash
aws elbv2 describe-target-health \
  --target-group-arn <alb-b-target-group-arn>
```

Result:

```text
State: unhealthy
Reason: Target.ResponseCodeMismatch
```

### Step 4 — Test the backend

On the target:

```bash
ss -lntp
curl -v http://127.0.0.1:8080/health
```

The application returns:

```text
401 Unauthorized
```

### Root cause

ALB-B's health-check path requires authentication, so its targets are marked unhealthy.

Because Route 53 is still distributing some DNS responses to ALB-B, users experience intermittent failures.

### Correct recovery

Do not randomly modify DNS, Security Groups, or NACLs.

Instead:

```text
1. Confirm Route 53 routing policy
2. Identify each returned destination
3. Test each destination independently
4. Inspect ALB-B target health
5. Fix the health-check endpoint/configuration
6. Validate target health
7. Confirm Route 53 behavior
8. Monitor application responses
```

The key lesson is:

> Intermittent DNS-based failures often mean different DNS answers lead to different backend health states.

---

## 27. Production Troubleshooting Decision Tree

Use this sequence:

```text
1. What hostname is failing?
        ↓
2. Does DNS resolve?
        ↓
3. What IP/endpoint was returned?
        ↓
4. Is the route to that destination correct?
        ↓
5. Is the required TCP port reachable?
        ↓
6. Is the load-balancer listener available?
        ↓
7. Are targets healthy?
        ↓
8. Does TLS succeed?
        ↓
9. Does HTTP return the expected status?
        ↓
10. Is the application healthy?
```

For each failed step, stop and investigate that layer before changing another layer.

---

## 28. Interview-Ready Mental Model

Remember:

```text
Route 53
   ↓
DNS record / routing policy
   ↓
DNS answer
   ↓
IP / AWS endpoint
   ↓
Routing
   ↓
TCP port
   ↓
Firewall / Security Group / NACL
   ↓
TLS
   ↓
Load balancer
   ↓
Target health
   ↓
Application
```

Short interview explanation:

> Route 53 provides DNS resolution and DNS-based routing. During troubleshooting I first verify the record and actual DNS answer using `dig` or `nslookup`. If DNS resolves, I trace the returned IP through routing with `ip route get`, test the TCP port with `nc`, validate TLS with `openssl`, and then test HTTP with `curl`. For AWS endpoints I inspect Route 53 records and health checks, then the load balancer listener, target health, Security Groups, NACLs, and finally the application. This separates DNS failures from network, firewall, load-balancer, and application failures.

---

## 29. Beginner Takeaway

If you remember only one thing:

```text
DNS tells you WHERE to go.
Routing tells you HOW traffic gets there.
TCP tells you WHETHER the port is reachable.
Firewall rules tell you WHETHER traffic is allowed.
TLS establishes secure HTTPS communication.
HTTP tells you WHETHER the application responded correctly.
```

And the most useful troubleshooting sequence is:

```text
DNS
 ↓
IP
 ↓
ROUTE
 ↓
PORT
 ↓
FIREWALL
 ↓
TLS
 ↓
HTTP
 ↓
APPLICATION
```
