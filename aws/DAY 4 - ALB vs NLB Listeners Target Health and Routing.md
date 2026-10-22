# DAY 4 — ALB vs NLB: Listeners, Target Health and Routing

## 1. Why This Topic Matters

A load balancer is often the entry point to a production application.

When users report:

```text
502
503
504
connection timeout
connection refused
intermittent failures
```

you need to determine whether the problem is at:

```text
DNS
 ↓
Load Balancer
 ↓
Listener
 ↓
Listener Rule
 ↓
Target Group
 ↓
Target Health
 ↓
Backend Port
 ↓
Application
```

The most important distinction is:

```text
ALB → Layer 7 HTTP/HTTPS-aware routing
NLB → Layer 4 TCP/UDP/TLS-aware load balancing
```

---

## 2. What Is a Load Balancer?

A load balancer receives client traffic and distributes it across backend targets.

Example:

```text
Client
  ↓
Load Balancer
  ↓
 ┌─────────┬─────────┬─────────┐
 ↓         ↓         ↓
Target 1  Target 2  Target 3
```

Instead of clients connecting directly to individual application servers, they connect to a stable load-balancer endpoint.

Benefits include:

- distributing traffic
- health checking targets
- high availability
- TLS termination where appropriate
- routing traffic based on application information for ALB
- supporting high-performance Layer 4 workloads with NLB

---

## 3. ALB — Application Load Balancer

An **Application Load Balancer (ALB)** operates primarily at Layer 7.

It understands application protocols such as HTTP and HTTPS.

This allows ALB to make routing decisions based on information such as:

- host name
- URL path
- HTTP headers
- query parameters
- HTTP methods in supported routing use cases

Example:

```text
api.example.com
      ↓
     ALB
      ↓
Target Group: API
```

```text
www.example.com
      ↓
     ALB
      ↓
Target Group: Web
```

Path-based routing:

```text
example.com/api/*
      ↓
API target group
```

```text
example.com/orders/*
      ↓
Orders target group
```

### Common ALB use cases

- web applications
- REST APIs
- microservices
- HTTP/HTTPS applications
- host/path-based routing
- TLS termination
- container and Kubernetes ingress architectures

---

## 4. NLB — Network Load Balancer

A **Network Load Balancer (NLB)** operates primarily at Layer 4.

It is designed for high-performance network traffic and can handle TCP, UDP and TLS use cases.

NLB does not use ALB-style HTTP path routing.

Think:

```text
Client
 ↓
TCP/TLS/UDP connection
 ↓
NLB
 ↓
Target
```

Typical use cases include:

- TCP applications
- UDP applications
- high-throughput services
- applications that need static IP-style endpoints
- TLS pass-through or TLS listener use cases
- protocols that are not HTTP

---

## 5. ALB vs NLB — Quick Comparison

| Feature | ALB | NLB |
|---|---|---|
| Main layer | Layer 7 | Layer 4 |
| HTTP/HTTPS aware | Yes | No HTTP routing logic like ALB |
| Path-based routing | Yes | No |
| Host-based routing | Yes | No |
| TCP | Via HTTP/HTTPS use cases, not raw TCP | Yes |
| UDP | No | Yes |
| TLS listener | Yes | Yes |
| Target groups | Yes | Yes |
| Health checks | Yes | Yes |
| Best mental model | Application-aware proxy/load balancer | Connection-aware network load balancer |

Do not memorize only the table. The key question is:

> **Does the load balancer need to understand HTTP application information?**

If yes, ALB is usually the natural choice.

If the workload needs Layer 4 TCP/UDP/TLS behavior, NLB is usually the natural choice.

---

## 6. Listeners

A **listener** checks for incoming client connections on a configured protocol and port.

Example ALB listeners:

```text
HTTP  :80
HTTPS :443
```

Example NLB listeners:

```text
TCP  :443
TLS  :443
UDP  :53
```

The exact listener protocols depend on the load balancer and architecture.

### Listener mental model

```text
Client
  ↓
Load Balancer IP/DNS
  ↓
Listener :443
  ↓
Listener rules / default action
  ↓
Target Group
```

If the client connects to port `443` but there is no listener on `443`, the request cannot reach the expected application path.

---

## 7. ALB Listener Rules

ALB listeners can have rules that inspect HTTP request information.

Example:

```text
HTTPS :443
    ↓
Host = api.example.com
    ↓
API target group
```

Another rule:

```text
HTTPS :443
    ↓
Path = /payments/*
    ↓
Payments target group
```

Rules are evaluated according to their configured priority, followed by the listener's default action when no earlier rule matches.

### Important troubleshooting question

Ask:

```text
Did the request reach the listener?
        ↓
Did a listener rule match?
        ↓
Which target group was selected?
```

A healthy target in the wrong target group does not help the request.

---

## 8. Target Groups

A **target group** is a logical group of backend targets.

Targets can include supported resources such as:

- EC2 instances
- IP addresses
- Lambda functions for ALB use cases
- containers/tasks through the relevant AWS integration

A target group also defines important backend behavior such as:

- target type
- protocol
- port
- health check configuration
- thresholds and timing settings

Example:

```text
ALB :443
   ↓
Target Group: web
   ↓
HTTP :8080
   ↓
EC2 instances
```

---

## 9. Target Health

The load balancer should not normally send new traffic to a target that is considered unhealthy for the relevant target group.

Health checks test whether the backend is reachable and responding as expected according to the configured health-check settings.

For an HTTP target, a health check might be:

```text
Protocol: HTTP
Port: traffic-port
Path: /health
```

The application might need to return an expected successful response.

### Important distinction

```text
Load balancer healthy
≠
Backend application healthy
```

The load balancer can be running perfectly while every backend target is unhealthy.

---

## 10. ALB Target Health Failure Example

Suppose:

```text
ALB HTTPS :443
      ↓
Target Group HTTP :8080
      ↓
EC2 application
```

Application actually listens on:

```text
127.0.0.1:8081
```

Health check expects:

```text
EC2:8080/health
```

The target becomes unhealthy.

The client may receive:

```text
503 Service Unavailable
```

The important point is that the failure is not necessarily the ALB listener itself.

The path is:

```text
Client
 ↓
ALB listener works
 ↓
Target group selected
 ↓
Target health check fails
 ↓
No healthy targets
 ↓
503
```

---

## 11. ALB Health Check Troubleshooting

Check:

```text
1. Target group
2. Target IP/instance
3. Target port
4. Health-check protocol
5. Health-check path
6. Expected response code
7. Security Group
8. NACL
9. Application listener
10. Application logs
```

Linux commands:

```bash
ss -lntp
```

Check the application locally:

```bash
curl -v http://127.0.0.1:8080/health
```

Check the instance IP/interface path where appropriate:

```bash
curl -v http://<private-ip>:8080/health
```

The second test is important because an application listening only on loopback can pass a local test but fail when accessed through the network.

---

## 12. Security Groups and Load Balancers

A common ALB architecture is:

```text
Internet
 ↓
ALB Security Group
 ↓
Target Security Group
 ↓
Application
```

A strong design usually allows the target Security Group to accept application traffic from the ALB Security Group rather than from the entire internet.

Example:

```text
ALB SG:
TCP 443 ← Internet
```

```text
Target SG:
TCP 8080 ← ALB SG
```

This creates a clearer trust boundary.

For NLB, the exact security model depends on whether the NLB and target configuration uses Security Groups and the specific NLB architecture. Do not blindly apply an ALB Security Group pattern to every NLB design.

---

## 13. ALB Routing Flow

Example:

```text
Client
 ↓
Route 53
 ↓
ALB
 ↓
HTTPS listener :443
 ↓
Listener rule
 ↓
Target group
 ↓
Healthy target
 ↓
Application
```

For example:

```text
api.example.com/orders
```

could become:

```text
Route 53
 ↓
ALB :443
 ↓
Host = api.example.com
 ↓
Path = /orders/*
 ↓
Orders target group
 ↓
EC2 :8080
```

---

## 14. NLB Routing Flow

NLB routing is primarily connection-oriented.

Example:

```text
Client
 ↓
NLB
 ↓
TCP listener :443
 ↓
Target group
 ↓
EC2 :8443
```

The NLB does not make an ALB-style decision such as:

```text
/path → target group A
```

Instead, it forwards network connections according to its Layer 4 configuration.

---

## 15. NLB Target Health

NLB target health still matters.

A target can be unhealthy because:

- wrong target port
- application not listening
- Security Group/NACL issue
- health-check protocol mismatch
- health-check port mismatch
- application is not responding as expected

Linux:

```bash
ss -lntup
```

TCP test:

```bash
nc -vz <target-ip> <port>
```

For TLS:

```bash
openssl s_client -connect <target-ip>:443 -servername example.com
```

---

## 16. Useful AWS CLI Commands

### Describe load balancers

```bash
aws elbv2 describe-load-balancers
```

### Describe listeners

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn <load-balancer-arn>
```

### Describe listener rules

```bash
aws elbv2 describe-rules \
  --listener-arn <listener-arn>
```

### Describe target groups

```bash
aws elbv2 describe-target-groups
```

### Check target health

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

### Inspect target health with a useful query

```bash
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --query 'TargetHealthDescriptions[].{Target:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason,Description:TargetHealth.Description}'
```

### Describe ALB/NLB attributes

```bash
aws elbv2 describe-load-balancer-attributes \
  --load-balancer-arn <load-balancer-arn>
```

---

## 17. AWS Objects to Recognize

```text
Load Balancer
├── ALB
│   └── Listener
│       ├── Listener Rules
│       └── Default Action
│
└── NLB
    └── Listener

Target Group
├── Target type
├── Protocol
├── Port
├── Health check
└── Targets
```

Important relationships:

```text
ALB/NLB
   ↓
Listener
   ↓
Target Group
   ↓
Target
```

For ALB:

```text
Listener
 ↓
Rule priority
 ↓
Host/path condition
 ↓
Target group
```

---

## 18. EKS / Kubernetes Connection

AWS load balancers are commonly used with Kubernetes through the AWS Load Balancer Controller.

Typical ALB-style path:

```text
Internet
 ↓
ALB
 ↓
Listener
 ↓
Ingress rules
 ↓
Kubernetes Service / target integration
 ↓
Pod
```

Typical NLB-style path:

```text
Internet
 ↓
NLB
 ↓
Listener
 ↓
Kubernetes Service type LoadBalancer
 ↓
Node/Pod target
```

When an EKS application is unavailable, inspect both layers:

```text
AWS load balancer
+
Kubernetes
```

Do not assume that a Kubernetes Service problem is always a Kubernetes-only problem.

---

## 19. Common Failure Paths

### Failure A — Listener missing

```text
Client → ALB:443
          ↓
       no :443 listener
```

Result: connection cannot follow the expected listener path.

### Failure B — Wrong listener rule

```text
Request
 ↓
ALB listener
 ↓
Wrong target group
 ↓
Unexpected application response
```

### Failure C — No healthy targets

```text
Client
 ↓
ALB listener works
 ↓
Target group
 ↓
0 healthy targets
 ↓
503
```

### Failure D — Wrong backend port

```text
ALB → target :8080

Application → :8081
```

Result: target health failure.

### Failure E — Application bound to loopback

```text
127.0.0.1:8080
```

The application works locally but not through the load balancer.

### Failure F — Security Group blocks ALB → target

```text
ALB
 ↓
Target SG blocks :8080
 ↓
Unhealthy target
```

### Failure G — NLB TCP connection works but application fails

A successful TCP connection does not prove that the application protocol is healthy.

Continue troubleshooting above Layer 4.

---

## 20. Realistic Production Failure Scenario

### Incident

A production API is deployed behind an ALB.

Architecture:

```text
Client
 ↓
Route 53
 ↓
ALB HTTPS :443
 ↓
Listener rule: /api/*
 ↓
API target group
 ↓
EC2 :8080
```

Suddenly users receive:

```text
503 Service Unavailable
```

### Step 1 — Check the load balancer

```bash
aws elbv2 describe-load-balancers
```

Confirm the ALB is available.

### Step 2 — Check listener

```bash
aws elbv2 describe-listeners \
  --load-balancer-arn <alb-arn>
```

Confirm HTTPS `443` exists.

### Step 3 — Check rules

```bash
aws elbv2 describe-rules \
  --listener-arn <listener-arn>
```

Confirm `/api/*` routes to the API target group.

### Step 4 — Check target health

```bash
aws elbv2 describe-target-health \
  --target-group-arn <api-target-group-arn>
```

Result:

```text
Target: 10.0.11.25:8080
State: unhealthy
Reason: Target.ResponseCodeMismatch
```

### Step 5 — Test application

On the target:

```bash
ss -lntp
curl -v http://127.0.0.1:8080/health
```

The application responds with:

```text
401 Unauthorized
```

But the target group's health check expects a successful health response.

### Root cause

The deployment changed `/health` so that it now requires authentication.

The ALB health checker cannot authenticate, so the target is marked unhealthy.

Therefore:

```text
ALB listener → healthy
Listener rule → correct
Target group → correct
Target → unhealthy
Health check → fails
No healthy backend targets
→ 503
```

### Correct fix

Restore an unauthenticated health endpoint intended for load-balancer health checking, or configure the health-check path/expected response appropriately for the application's design.

Do not solve this by blindly opening Security Groups.

### Validation

```bash
curl -v http://127.0.0.1:8080/health
```

Then:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <api-target-group-arn>
```

Wait for the target to become healthy and test through the ALB endpoint.

---

## 21. Production Troubleshooting Decision Tree

```text
Client cannot reach application
          ↓
       DNS works?
       /        \
     No          Yes
     ↓            ↓
  DNS issue    Listener reachable?
                    ↓
              Listener exists?
                    ↓
              Correct rule?
                    ↓
              Correct target group?
                    ↓
              Healthy targets?
                 /       \
               No         Yes
               ↓           ↓
        Health-check      Backend
          failure         response
             ↓               ↓
        Port/path/SG/    App/TLS/HTTP
        NACL/app
```

For NLB:

```text
Client
 ↓
NLB listener
 ↓
Target group
 ↓
Target health
 ↓
TCP/TLS/UDP backend
 ↓
Application protocol
```

---

## 22. Common Mistakes

### Mistake 1 — Choosing ALB for every workload

ALB is excellent for HTTP/HTTPS application-aware routing, but raw TCP/UDP requirements may call for NLB.

### Mistake 2 — Assuming a healthy load balancer means healthy application

The load balancer can be healthy while every target is unhealthy.

### Mistake 3 — Ignoring target health reason

Always inspect `describe-target-health` instead of guessing.

### Mistake 4 — Testing only localhost

```bash
curl http://127.0.0.1:8080/health
```

can succeed even when the application is not reachable through its network interface.

### Mistake 5 — Confusing listener port and target port

Example:

```text
ALB listener: 443
Target: 8080
```

This is perfectly valid. HTTPS termination can occur at the ALB while the backend uses HTTP.

### Mistake 6 — Fixing 503 by opening everything

A 503 often means no healthy targets. Find why the targets are unhealthy instead of blindly changing security rules.

---

## 23. Interview-Ready Mental Model

Remember:

```text
ALB = Layer 7 + HTTP/HTTPS + host/path routing
NLB = Layer 4 + TCP/UDP/TLS + high-performance network traffic
```

And:

```text
LOAD BALANCER
 ↓
LISTENER
 ↓
RULE / DEFAULT ACTION
 ↓
TARGET GROUP
 ↓
TARGET HEALTH
 ↓
BACKEND PORT
 ↓
APPLICATION
```

### Beginner takeaway

If you remember only five things:

1. **ALB understands HTTP/HTTPS and can route using host/path information.**
2. **NLB primarily handles Layer 4 traffic such as TCP, UDP and TLS.**
3. **A listener accepts traffic on a configured port/protocol.**
4. **A target group defines backend targets and health-check behavior.**
5. **For a 503, check target health before randomly changing security rules.**
