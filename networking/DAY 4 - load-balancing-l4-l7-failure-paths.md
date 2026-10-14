# DAY 4 — Load Balancing: L4 vs L7 and Common Failure Paths

## Why This Topic Matters

A load balancer sits between clients and backend services and decides where traffic should go.

In production, a common incident looks like:

```text
Client can reach the load balancer
        ↓
But the application still fails
```

To troubleshoot this correctly, you must know:

- What the load balancer actually understands
- Whether it operates at L4 or L7
- Where TLS terminates
- How health checks work
- How traffic reaches the backend
- Which layer failed first

A useful mental model is:

```text
Client
  ↓
DNS
  ↓
Load Balancer
  ↓
Listener
  ↓
Routing decision
  ↓
Target Group / Backend
  ↓
Application
```

---

# 1. What Is Load Balancing?

Load balancing distributes incoming traffic across multiple backend servers or application instances.

Without a load balancer:

```text
Client
  ↓
Server 1
```

If Server 1 fails, the service may become unavailable.

With a load balancer:

```text
              ┌── Server 1
Client → LB ──┼── Server 2
              └── Server 3
```

The load balancer can distribute requests and stop sending traffic to unhealthy targets.

---

# 2. Why Do We Need Load Balancers?

Common reasons include:

- High availability
- Horizontal scaling
- Fault tolerance
- Traffic distribution
- Health-based routing
- TLS termination
- HTTP routing
- Connection management

In AWS, load balancing is commonly used with:

- EC2
- ECS
- EKS
- Auto Scaling Groups
- CloudFront architectures

---

# 3. L4 vs L7 — The Core Difference

The easiest way to remember the difference is:

```text
L4 → understands network/transport information
L7 → understands application information
```

### L4

Typically works with:

```text
IP
Port
TCP
UDP
```

It does not need to understand HTTP paths such as `/login` or `/api/users`.

### L7

Understands application protocols such as HTTP/HTTPS.

It can inspect:

```text
HTTP method
Host header
URL path
Headers
Cookies
HTTP status
```

That allows much more intelligent routing.

---

# 4. Simple L4 Example

Suppose a service is available on TCP port 443:

```text
Client
  ↓ TCP/443
L4 Load Balancer
  ↓
Backend Server
```

The load balancer mainly needs to establish/forward the network connection.

It does not need to understand whether the request is:

```text
GET /login
GET /payments
POST /users
```

Those are application-level details.

---

# 5. Simple L7 Example

An L7 load balancer can make decisions based on HTTP information.

For example:

```text
api.example.com/users
        ↓
Backend A

api.example.com/payments
        ↓
Backend B
```

Or:

```text
Host: api.example.com
Path: /v1/*
        ↓
Service A
```

```text
Host: admin.example.com
        ↓
Service B
```

This requires understanding the HTTP request, so it is an L7 function.

---

# 6. L4 vs L7 Comparison

| Feature | L4 | L7 |
|---|---|---|
| Main layer | Transport | Application |
| Common protocols | TCP/UDP | HTTP/HTTPS |
| Understands URL path | No | Yes |
| Understands HTTP headers | No | Yes |
| Host-based routing | No | Yes |
| Path-based routing | No | Yes |
| TLS termination | Possible depending on product/design | Common |
| Protocol-aware HTTP routing | No | Yes |

The key interview point is not memorizing a product name. Understand **what information the load balancer can inspect and use for routing**.

---

# 7. AWS Load Balancers — What to Know

AWS commonly provides these load balancing choices:

```text
Application Load Balancer (ALB) → L7
Network Load Balancer (NLB) → L4-oriented
Gateway Load Balancer (GWLB) → network/security appliance use cases
```

For this topic, focus mainly on ALB vs NLB.

---

# 8. Application Load Balancer (ALB)

ALB is designed for application-layer traffic, especially HTTP/HTTPS.

A simplified path:

```text
Client
  ↓
ALB
  ↓
Listener :443
  ↓
Listener Rule
  ↓
Target Group
  ↓
EC2 / ECS / IP target
```

ALB can route using application information such as:

- Host
- Path
- HTTP headers
- Query parameters in supported routing scenarios

Example:

```text
api.example.com
        ↓
ALB
        ↓
/api/*
        ↓
API target group
```

---

# 9. Network Load Balancer (NLB)

NLB is designed for very high-performance network traffic and operates primarily at the connection/transport layer.

Simplified:

```text
Client
  ↓ TCP/443
NLB
  ↓
Target
```

The important mental model is:

```text
ALB → HTTP-aware routing
NLB → connection/transport-oriented routing
```

Do not reduce this to “ALB is always HTTP and NLB is always TCP.” AWS supports different listener/protocol configurations, so always check the actual architecture.

---

# 10. Listener — The Entry Point

A load balancer listener waits for traffic on a specific protocol and port.

Example:

```text
ALB
 ↓
HTTPS listener :443
```

The listener can then use rules to decide where traffic goes.

For an ALB:

```text
Listener :443
   ↓
Rule
   ↓
Target Group
```

If the listener is missing or configured incorrectly, traffic may never reach the intended backend.

---

# 11. Target Groups

A target group is a collection of backend targets that receive traffic.

Targets can include supported types such as:

- Instances
- IP addresses
- Containers through appropriate integrations

Conceptually:

```text
ALB
 ↓
Target Group
 ├── Target 1
 ├── Target 2
 └── Target 3
```

The load balancer uses target health to determine whether traffic should be sent to a target.

---

# 12. Health Checks

Health checks answer a critical question:

> Can this backend successfully serve the expected health-check request?

For an HTTP target group, a health check might look like:

```text
GET /health
```

The load balancer checks the configured target and port.

Possible failure points include:

```text
Wrong port
Wrong path
Wrong protocol
Application not running
Security Group
Host firewall
Application returning unexpected status
Network routing
```

A target can therefore be:

```text
EC2 instance = RUNNING
```

but:

```text
Target health = UNHEALTHY
```

These are not contradictory.

---

# 13. TLS Termination and Load Balancers

TLS can terminate at the load balancer.

Example:

```text
Client
  ↓ HTTPS
ALB :443
  ↓ HTTP
Application :8080
```

The client has a secure HTTPS connection to the ALB, but the ALB communicates with the backend using HTTP.

Another design is:

```text
Client
  ↓ HTTPS
ALB :443
  ↓ HTTPS
Application :8443
```

In this case TLS exists on both sides, but they are separate connections.

This distinction is important when debugging certificates.

---

# 14. Common Failure Path #1 — DNS Works, Load Balancer Does Not

Example:

```text
curl https://api.example.com
```

DNS resolves successfully, but TCP/443 times out.

Troubleshooting:

```bash
dig +short api.example.com
nc -vz api.example.com 443
```

Investigate:

```text
DNS
 ↓
Returned IP
 ↓
Route
 ↓
Internet/VPC path
 ↓
Load balancer listener
 ↓
Security Group / NACL where applicable
```

Do not investigate the application first if the client cannot establish TCP to the load balancer.

---

# 15. Common Failure Path #2 — TCP Works, But No HTTP Response

Suppose:

```bash
nc -vz api.example.com 443
```

succeeds, but:

```bash
curl -v https://api.example.com
```

fails during TLS.

Now investigate:

```text
TCP ✓
TLS ✗
```

Possible causes:

- Wrong certificate
- Certificate expired
- SAN mismatch
- SNI-related configuration
- Unsupported TLS version/cipher
- Incorrect TLS listener configuration

This is no longer primarily a routing problem.

---

# 16. Common Failure Path #3 — TLS Works, HTTP Returns 503

Suppose:

```text
DNS ✓
TCP ✓
TLS ✓
HTTP → 503
```

This strongly moves the investigation toward the load balancer and backend path.

Check:

```text
Listener
 ↓
Listener rule
 ↓
Target group
 ↓
Target health
 ↓
Backend port
 ↓
Application
```

A common example:

```text
Application listening on 8080
Target group configured for 8081
```

Result:

```text
Targets unhealthy
        ↓
ALB has no usable healthy targets
        ↓
Client receives 503
```

---

# 17. Common Failure Path #4 — 502 Bad Gateway

A `502` commonly indicates that a proxy/load balancer received an invalid or unusable response from an upstream.

Possible causes include:

- Backend connection failure
- Protocol mismatch
- Application closed connection unexpectedly
- Incorrect backend configuration
- TLS mismatch on the LB-to-backend connection

Do not treat `502` as proof of one specific root cause.

Use evidence from the load balancer and backend.

---

# 18. Common Failure Path #5 — 504 Gateway Timeout

A `504` commonly means the proxy/load balancer waited for an upstream response and timed out.

Possible causes:

```text
ALB
 ↓
Backend
 ↓
Application
 ↓
Database/external dependency
```

The application may be reachable but too slow.

Investigate:

- Backend response time
- Application logs
- Dependency latency
- Load balancer timeout configuration
- Network connectivity
- Resource saturation

---

# 19. Common Failure Path #6 — One Target Is Bad

Suppose a target group contains:

```text
Target A → healthy
Target B → healthy
Target C → unhealthy
```

The service may still work for many requests.

This can create an intermittent incident:

```text
Request → Target A → 200
Request → Target B → 200
Request → Target C → 5xx/timeout
```

Do not assume intermittent errors are random.

Compare target health and backend logs.

---

# 20. Common Failure Path #7 — Listener Rule Sends Traffic to the Wrong Backend

ALB routing can depend on conditions such as host/path.

Example:

```text
/api/* → API target group
/admin/* → Admin target group
```

If a rule is changed incorrectly:

```text
/api/users
   ↓
Admin target group
```

The network can be completely healthy while the application behavior is wrong.

This is a classic L7 failure.

---

# 21. Common Failure Path #8 — Backend Port Mismatch

Example:

```text
ALB target group → port 8080
Application → listening on 8081
```

Check the backend:

```bash
sudo ss -lntp
```

Then compare the actual listener with the target group configuration.

This is one of the most useful real-world troubleshooting checks.

---

# 22. Common Failure Path #9 — Application Binds Only to Loopback

Suppose the application is running:

```text
127.0.0.1:8080
```

The local host may successfully access it:

```bash
curl http://127.0.0.1:8080/health
```

But the load balancer cannot connect through the instance's network interface.

Check:

```bash
sudo ss -lntp | grep ':8080'
```

If required, configure the application to listen on an appropriate interface such as:

```text
0.0.0.0:8080
```

while still controlling access with Security Groups/firewalls.

---

# 23. Common Failure Path #10 — Security Group Blocks LB-to-Backend Traffic

A common AWS architecture is:

```text
Internet
   ↓ 443
ALB
   ↓ 8080
EC2
```

The ALB Security Group may allow inbound 443 from clients.

The EC2 Security Group should allow the backend port from the ALB Security Group as appropriate.

If this rule is missing:

```text
Client → ALB ✓
ALB → EC2 ✗
```

The client can still reach the load balancer while the backend remains unreachable.

---

# 24. Kubernetes Load Balancing Connection

A typical Kubernetes application path is:

```text
Client
 ↓
Load Balancer / Ingress / Gateway
 ↓
Service
 ↓
Endpoint / Pod
 ↓
Application
```

Depending on the environment, an AWS load balancer may be provisioned through the AWS Load Balancer Controller.

For troubleshooting, check:

```text
Ingress/Gateway
 ↓
Listener
 ↓
TLS Secret
 ↓
Service
 ↓
Service port
 ↓
Target port
 ↓
Endpoints
 ↓
Pod readiness
 ↓
Container port
 ↓
Application
```

A healthy Pod does not automatically prove that the complete external load-balancing path is healthy.

---

# 25. Useful Linux Commands for Load-Balancer Troubleshooting

## Check listeners

```bash
sudo ss -lntp
```

## Test TCP connectivity

```bash
nc -vz <host> <port>
```

## Test HTTP/HTTPS

```bash
curl -v https://<host>
```

## Check DNS

```bash
dig +short <hostname>
```

## Check the route

```bash
ip route get <destination-ip>
```

## Inspect TLS

```bash
openssl s_client -connect <host>:443 -servername <host> </dev/null
```

The goal is not to run every command automatically. Use the command that proves the layer you are investigating.

---

# 26. Production Troubleshooting Decision Tree

When a service behind a load balancer fails:

```text
DNS resolves?
   │
   ├── NO → DNS investigation
   │
   └── YES
        ↓
TCP to LB works?
   │
   ├── NO → route / firewall / SG / NACL / LB listener
   │
   └── YES
        ↓
TLS works?
   │
   ├── NO → certificate / SNI / TLS configuration
   │
   └── YES
        ↓
HTTP response?
   │
   ├── 4xx → request/routing/auth/application logic
   │
   ├── 5xx → LB/backend/application investigation
   │
   └── 2xx → request path is working
        ↓
Target healthy?
   │
   ├── NO → health check / port / SG / application
   │
   └── YES
        ↓
Backend response healthy?
   │
   ├── NO → application/dependency investigation
   │
   └── YES → inspect higher-level behavior
```

---

# 27. Realistic Production Failure Scenario

### Incident

Users report:

```text
https://api.company.com/orders
```

is intermittently returning `503`.

### Step 1 — DNS

```bash
dig +short api.company.com
```

DNS returns the expected load balancer address.

```text
DNS ✓
```

### Step 2 — TCP

```bash
nc -vz api.company.com 443
```

Connection succeeds.

```text
TCP ✓
```

### Step 3 — TLS

```bash
curl -v https://api.company.com/orders
```

TLS succeeds and the certificate is valid.

```text
TLS ✓
```

### Step 4 — HTTP

The response is intermittently:

```text
503 Service Unavailable
```

Now inspect the target group.

Suppose:

```text
Target A → healthy
Target B → healthy
Target C → unhealthy
```

### Step 5 — Compare Target C

On Target C:

```bash
sudo ss -lntp | grep ':8080'
```

Output shows:

```text
127.0.0.1:8080
```

while the ALB expects to reach the target through its private IP.

### Root Cause

The application on Target C was bound only to loopback.

Therefore:

```text
Client → ALB ✓
ALB → Target A ✓
ALB → Target B ✓
ALB → Target C ✗
```

The incident was not a DNS or TLS problem.

### Lesson

Intermittent load-balancer errors can be caused by a single unhealthy backend.

Always inspect target health and compare healthy vs unhealthy targets.

---

# 28. Common Mistakes

### Mistake 1 — Thinking load balancer = application

A healthy load balancer does not mean the backend is healthy.

### Mistake 2 — Checking only the load balancer

You must investigate the LB-to-target path.

### Mistake 3 — Assuming 503 means the application is definitely down

A 503 is a symptom. Check target health, listener rules, backend connectivity, and application behavior.

### Mistake 4 — Confusing L4 and L7

Remember:

```text
L4 → transport/connection information
L7 → application information
```

### Mistake 5 — Ignoring health checks

Health checks are one of the fastest ways to understand why an LB is not using a backend.

### Mistake 6 — Forgetting the backend port

Always compare:

```text
Target group port
vs
Application listening port
```

### Mistake 7 — Assuming a running EC2 instance is a healthy target

EC2 `running` state and load-balancer target health are different signals.

---

# 29. Interview-Ready Mental Model

If asked:

> How do you troubleshoot an application behind a load balancer?

Answer with the path:

```text
DNS
 ↓
Load balancer IP
 ↓
TCP listener
 ↓
TLS if applicable
 ↓
HTTP response
 ↓
Listener rules
 ↓
Target group
 ↓
Target health
 ↓
Backend port
 ↓
Security Group / firewall
 ↓
Application
```

Then identify the **first failed layer**.

That is more useful than memorizing isolated commands.

---

# 30. Beginner Takeaway

Remember these five ideas:

```text
1. L4 → transport/connection level
2. L7 → application-aware routing
3. ALB → L7 HTTP/HTTPS use cases
4. NLB → L4-oriented network load balancing
5. Always troubleshoot Client → LB → Target → Application
```

And the most important production rule:

> **A successful connection to the load balancer does not prove that the backend application is healthy.**
