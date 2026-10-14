# Networking Labs

Centralized practical labs for Networking. Labs from every Networking DAY are added here.

---

## DAY 1 — TCP/IP, Ports, Sockets and `ss`/`netstat`

### Lab 1 — Inspect Your Linux Network

```bash
ip addr
ip route
ip route get 8.8.8.8
```

**Goal:** Identify your active interface, local IP, default gateway, and selected route.

### Lab 2 — Inspect Listening Services

```bash
sudo ss -lntup
ss -lnt
sudo ss -lntp
```

**Goal:** Understand listening sockets and owning processes.

### Lab 3 — Create a Simple TCP Service

```bash
python3 -m http.server 8080 --bind 0.0.0.0
sudo ss -lntp | grep ':8080'
curl http://127.0.0.1:8080
```

**Goal:** Connect process → socket → port → request.

### Lab 4 — Understand `127.0.0.1` vs `0.0.0.0`

```bash
python3 -m http.server 8080 --bind 127.0.0.1
sudo ss -lntp | grep ':8080'
python3 -m http.server 8080 --bind 0.0.0.0
```

**Goal:** Understand why local access can work while remote access fails.

### Lab 5 — DNS Troubleshooting

```bash
dig example.com
dig +short example.com
nslookup example.com
dig @8.8.8.8 example.com
```

**Goal:** Separate DNS from connectivity.

### Lab 6 — Test a TCP Port

```bash
nc -vz example.com 443
nc -vz example.com 81
```

**Goal:** Compare reachable and unavailable TCP ports.

### Lab 7 — Build a Full Troubleshooting Flow

```bash
dig +short <hostname>
ip route get <destination-ip>
ping <destination-ip>
nc -vz <destination-ip> 8080
sudo ss -lntp | grep ':8080'
curl http://<destination-ip>:8080
```

**Goal:** Prove each layer instead of guessing.

### Lab 8 — AWS EC2 Connectivity Troubleshooting

Verify EC2 IP, route table, Security Group, NACL if relevant, Linux listener, host firewall, bind address, and client-side `nc`/`curl`.

**Goal:** Separate AWS-layer failures from Linux-layer failures.

### DAY 1 Capstone — Diagnose One Unreachable Service

Document:

```text
DNS:
Route:
Reachability:
TCP port:
Server listener:
Bind address:
Host firewall:
AWS Security Group:
Network ACL:
Application:
Root cause:
Evidence:
```

---

## DAY 2 — DNS Resolution, `dig`, `nslookup` and Route 53

### Lab 1 — Basic DNS Lookup

```bash
dig example.com
dig +short example.com
nslookup example.com
```

### Lab 2 — Query Record Types

```bash
dig A example.com
dig AAAA example.com
dig MX example.com
dig NS example.com
dig TXT example.com
```

### Lab 3 — Compare DNS Resolvers

```bash
dig example.com
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
```

Compare answers and TTL values.

### Lab 4 — Trace DNS Delegation

```bash
dig +trace example.com
```

### Lab 5 — Separate DNS From Connectivity

```bash
dig +short <hostname>
ip route get <destination-ip>
ping <destination-ip>
nc -vz <destination-ip> <port>
```

### Lab 6 — Route 53 Public Hosted Zone

Using an AWS test account and a domain you control, create a test Route 53 record and verify it with:

```bash
dig <hostname>
dig +short <hostname>
```

### Lab 7 — Route 53 Private DNS

Using a test VPC, create/associate a private hosted zone and query the internal record from an associated instance:

```bash
dig app.internal.example.com
```

Then test connectivity to its returned private IP.

### Lab 8 — Diagnose a DNS-Based Service Failure

Investigate:

```text
DNS → resolver → IP → route → TCP/443 → listener → AWS controls → application
```

Useful commands:

```bash
dig api.example.com
dig @8.8.8.8 api.example.com
dig +trace api.example.com
ip route get <destination-ip>
nc -vz <destination-ip> 443
sudo ss -lntp | grep ':443'
```

### DAY 2 Capstone — DNS + AWS Troubleshooting Report

Document hostname, resolver, DNS status, record type, IP, TTL, route, TCP port, listener, Security Group, NACL, application, root cause, and evidence.

---

## DAY 3 — HTTP/HTTPS, TLS Handshake and `curl` Troubleshooting

### Lab 1 — Start a Local HTTP Server

Terminal 1:

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Terminal 2:

```bash
curl -v http://127.0.0.1:8080
curl -I http://127.0.0.1:8080
```

Observe:

```text
TCP connection
HTTP request
HTTP response
status code
headers
```

**Goal:** See the difference between network connectivity and an HTTP response.

---

### Lab 2 — Explore HTTP Status Codes

Use a test HTTP endpoint and run:

```bash
curl -I https://example.com
curl -L -I https://example.com
```

Identify whether the response is `2xx`, `3xx`, `4xx`, or `5xx`.

**Goal:** Learn to interpret HTTP status codes during incidents.

---

### Lab 3 — Compare HTTP and HTTPS

Run:

```bash
curl -v http://example.com
curl -v https://example.com
```

Observe that HTTPS adds TLS negotiation before HTTP application data.

**Goal:** Understand where TLS sits in the connection flow.

---

### Lab 4 — Inspect TLS With `curl -v`

Run:

```bash
curl -vvv https://example.com
```

Find these stages in the output:

```text
DNS resolution
TCP connection
TLS handshake
certificate information
HTTP request
HTTP response
```

**Goal:** Identify the exact stage where a request stops.

---

### Lab 5 — Inspect the Certificate With OpenSSL

Run:

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null
```

Inspect:

```text
TLS version
cipher
certificate subject
SAN
issuer
certificate chain
verification result
```

**Goal:** Build confidence diagnosing certificate and TLS problems independently of `curl`.

---

### Lab 6 — Demonstrate the Diagnostic Use of `-k`

First run:

```bash
curl -v https://<test-host>
```

If certificate verification fails, compare:

```bash
curl -k -v https://<test-host>
```

**Goal:** Understand that `-k` can help prove that certificate validation is the problem, but it should not be used as the production fix.

> Only perform this against a test endpoint you control or are authorized to troubleshoot.

---

### Lab 7 — Measure HTTP Request Stages

Run:

```bash
curl -s -o /dev/null \
  -w 'code=%{http_code} dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s total=%{time_total}s\n' \
  https://example.com
```

**Goal:** Understand where request time is being spent.

---

### Lab 8 — AWS HTTPS Path

For a test AWS architecture such as:

```text
Route 53 → ALB → Target Group → EC2 application
```

Test from a client:

```bash
dig +short api.example.com
nc -vz api.example.com 443
curl -v https://api.example.com/health
```

Then inspect:

```text
ALB listener :443
certificate
listener rules
target group
health status
EC2 Security Group
application listener
```

**Goal:** Understand that successful TLS to the ALB does not prove the backend application is healthy.

---

### Lab 9 — Realistic Failure Scenario: HTTPS 503

Assume:

```text
https://api.example.com/health → 503
```

Prove each layer:

```bash
dig +short api.example.com
nc -vz api.example.com 443
curl -v https://api.example.com/health
```

If DNS and TCP succeed, TLS succeeds, and the response is:

```text
HTTP/1.1 503 Service Unavailable
```

investigate:

```text
ALB listener
→ listener rules
→ target group
→ target health
→ backend port
→ application
```

**Example root cause:** Target group health check is configured for port `8081` while the application is listening on `8080`.

**Goal:** Identify the first failed application/load-balancer layer rather than incorrectly blaming DNS or TLS.

---

### Lab 10 — Realistic Failure Scenario: Certificate Error

Run:

```bash
curl -v https://api.example.com
```

If certificate validation fails, compare:

```bash
curl -k -v https://api.example.com
```

Then inspect:

```bash
openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null
```

Check:

```text
hostname/SAN
expiration
issuer
certificate chain
verification result
```

**Goal:** Distinguish TCP connectivity, TLS negotiation, and certificate validation failures.

---

### DAY 3 Capstone — Diagnose One HTTPS Failure

Given:

```text
https://api.example.com/health
```

produce an incident note:

```text
DNS:
Destination IP:
TCP/443:
TLS handshake:
Certificate:
SNI:
HTTP status:
ALB/Ingress:
Target health:
Backend port:
Application:
Root cause:
Evidence:
```

**Success criteria:** Identify whether the first failure is DNS, TCP, TLS, certificate validation, HTTP, load balancer/proxy, backend, or application.

---

## DAY 4 — Load Balancing: L4 vs L7 and Common Failure Paths

### Lab 1 — Identify the Load-Balancing Layer

For each example, decide whether the routing decision is primarily L4 or L7:

```text
1. TCP/443 → backend pool
2. api.example.com → API target group
3. /payments/* → payment service
4. UDP/53 → DNS backend pool
```

Expected reasoning:

```text
TCP/443 → L4
Host/path routing → L7
UDP/53 → L4
```

**Goal:** Learn to classify a load-balancing problem by what information the load balancer needs to inspect.

---

### Lab 2 — Inspect a Local Backend Listener

Start a test server:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

Inspect it:

```bash
sudo ss -lntp | grep ':8080'
curl -v http://127.0.0.1:8080
```

**Goal:** Connect the load-balancer backend concept to a real listening process and port.

---

### Lab 3 — Demonstrate a Backend Port Mismatch

Run an application on port `8080`, but imagine the target group is configured for `8081`.

Verify the actual listener:

```bash
sudo ss -lntp | grep ':8080'
nc -vz 127.0.0.1 8080
nc -vz 127.0.0.1 8081
```

Document:

```text
Configured target port:
Actual application port:
Health-check result:
Expected LB behavior:
```

**Goal:** Recognize a common reason for unhealthy targets and HTTP 503 responses.

---

### Lab 4 — Demonstrate Loopback Binding Failure

Start a server bound only to loopback:

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Check:

```bash
sudo ss -lntp | grep ':8080'
curl http://127.0.0.1:8080
```

Then reason about why another host cannot connect to the instance's private IP.

**Goal:** Understand the difference between “application is running” and “application is reachable by the load balancer.”

---

### Lab 5 — AWS ALB Request Path

For a test AWS environment:

```text
Client
 ↓
ALB :443
 ↓
Listener rule
 ↓
Target group :8080
 ↓
EC2 application
```

Inspect:

```text
ALB listener
Listener rules
Target group
Target health
Target port
EC2 Security Group
Application listener
```

From a client, test:

```bash
dig +short api.example.com
nc -vz api.example.com 443
curl -v https://api.example.com/health
```

**Goal:** Understand the complete client-to-backend path.

---

### Lab 6 — L7 Host and Path Routing

Using a test ALB, create two simple backend services and route using concepts such as:

```text
api.example.com/* → API target group
admin.example.com/* → Admin target group
```

or:

```text
/api/* → API target group
/admin/* → Admin target group
```

Verify with:

```bash
curl -v https://api.example.com/health
curl -v https://api.example.com/api/users
```

**Goal:** Understand why host/path routing is an L7 capability.

---

### Lab 7 — Target Health Investigation

When a target is unhealthy, check the configured:

```text
Health-check protocol
Health-check port
Health-check path
Expected success codes
Timeout
Interval
```

On the backend, verify:

```bash
sudo ss -lntp
curl -v http://127.0.0.1:<port>/<health-path>
nc -vz <private-ip> <port>
```

**Goal:** Prove whether the health-check failure is caused by the application, port, protocol, path, or network controls.

---

### Lab 8 — ALB Security Group vs Backend Security Group

For:

```text
Internet → ALB:443 → EC2:8080
```

Verify the intended security model:

```text
ALB SG:
  inbound 443 from approved clients

EC2 SG:
  inbound 8080 from ALB SG
```

**Goal:** Understand the difference between client-to-LB access and LB-to-backend access.

---

### Lab 9 — 503 Failure Scenario

Simulate or document:

```text
DNS ✓
TCP/443 ✓
TLS ✓
HTTP → 503
```

Investigate in this order:

```text
Listener
→ listener rule
→ target group
→ target health
→ backend port
→ Security Group
→ application
```

Example root cause:

```text
Application → 8080
Target group → 8081
```

**Goal:** Find the first failed LB/backend layer instead of blaming DNS or TLS.

---

### Lab 10 — Intermittent Failure From One Bad Target

Assume:

```text
Target A → healthy
Target B → healthy
Target C → unhealthy
```

Compare the healthy and unhealthy target configurations:

```text
Listening port
Bind address
Health endpoint
Application logs
CPU/memory
Security Group
```

**Goal:** Understand why an application can be intermittently successful when only one backend is unhealthy.

---

### DAY 4 Capstone — Diagnose a Load-Balancer Incident

Given:

```text
https://api.example.com/orders
```

produce an incident report:

```text
DNS:
Load balancer type:
L4/L7:
Listener:
TLS termination point:
Listener rule:
Target group:
Target health:
Backend port:
Backend bind address:
Security Group:
Application:
HTTP status:
Root cause:
Evidence:
```

**Success criteria:** Identify the first failed layer and explain why the failure is L4/network-level, L7/load-balancer-level, backend-level, or application-level.
