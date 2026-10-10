# Networking Labs

Centralized practical labs for Networking. Labs from every Networking DAY are added here.

---

## DAY 1 — TCP/IP, Ports, Sockets and `ss`/`netstat`

### Lab 1 — Inspect Your Linux Network

Run:

```bash
ip addr
ip route
ip route get 8.8.8.8
```

**Goal:** Identify your active interface, local IP, default gateway, and the route Linux chooses for an external destination.

---

### Lab 2 — Inspect Listening Services

Run:

```bash
sudo ss -lntup
```

Pick three listening ports and identify the process behind each one.

Then compare:

```bash
ss -lnt
sudo ss -lntp
```

**Goal:** Understand the difference between seeing a listening socket and identifying the owning process.

---

### Lab 3 — Create a Simple TCP Service

On one terminal:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

In another terminal:

```bash
sudo ss -lntp | grep ':8080'
```

Then test locally:

```bash
curl http://127.0.0.1:8080
```

**Goal:** Connect the application process → listening socket → TCP port → client request.

---

### Lab 4 — Understand `127.0.0.1` vs `0.0.0.0`

Stop the previous server and start:

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Check:

```bash
sudo ss -lntp | grep ':8080'
```

Then start it again with:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

Compare the listening address.

**Goal:** Understand why an application that works locally may still be unreachable remotely.

---

### Lab 5 — DNS Troubleshooting

Run:

```bash
dig example.com
dig +short example.com
nslookup example.com
```

Then query a specific DNS resolver:

```bash
dig @8.8.8.8 example.com
```

**Goal:** Understand hostname resolution and how to separate DNS problems from network connectivity problems.

---

### Lab 6 — Test a TCP Port

Run against a known reachable service:

```bash
nc -vz example.com 443
```

Then try a port that is unlikely to be open:

```bash
nc -vz example.com 81
```

Compare the results.

**Goal:** Understand the difference between testing ICMP reachability and testing an actual TCP service port.

---

### Lab 7 — Build a Full Troubleshooting Flow

Use a local HTTP service:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

Troubleshoot it using only these stages:

```text
DNS → route → reachability → TCP port → listener → application
```

Commands to use:

```bash
dig +short <hostname>
ip route get <destination-ip>
ping <destination-ip>
nc -vz <destination-ip> 8080
sudo ss -lntp | grep ':8080'
curl http://<destination-ip>:8080
```

**Goal:** Practice proving each stage instead of guessing that the firewall is responsible.

---

### Lab 8 — AWS EC2 Connectivity Troubleshooting

Create/use an EC2 test instance with a simple application listening on TCP `8080`.

Verify in order:

1. EC2 private/public IP.
2. Route table.
3. Security Group inbound rule.
4. Network ACL if relevant.
5. Linux listener with `ss`.
6. Linux host firewall if enabled.
7. Application bind address.
8. Client-side `nc -vz` or `curl`.

**Goal:** Separate AWS-layer failures from Linux-layer failures.

> Use the smallest appropriate Security Group source range. Do not open production services to `0.0.0.0/0` just for testing.

---

### DAY 1 Capstone — Diagnose One Unreachable Service

Given a service such as:

```text
app.example.com:8080
```

Pretend the client reports a timeout.

Produce a short incident note containing:

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

**Success criteria:** You can identify the first failed layer and explain why later layers were or were not investigated.

---

## DAY 2 — DNS Resolution, `dig`, `nslookup` and Route 53

### Lab 1 — Basic DNS Lookup

Run:

```bash
dig example.com
dig +short example.com
nslookup example.com
```

Identify the returned IP address, query status, and answer section.

**Goal:** Become comfortable with the basic DNS lookup workflow.

---

### Lab 2 — Query Different Record Types

Run:

```bash
dig A example.com
dig AAAA example.com
dig MX example.com
dig NS example.com
dig TXT example.com
```

**Goal:** Understand what different DNS records are used for.

---

### Lab 3 — Compare DNS Resolvers

Run:

```bash
dig example.com
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com
```

Compare the answers and TTL values.

**Goal:** Understand how to determine whether a problem is specific to a DNS resolver.

---

### Lab 4 — Trace DNS Delegation

Run:

```bash
dig +trace example.com
```

Follow the path from root DNS servers to the authoritative servers for the domain.

**Goal:** Understand DNS delegation rather than treating DNS as a single server.

---

### Lab 5 — Separate DNS From Connectivity

Choose a hostname that resolves to an IP and run:

```bash
dig +short <hostname>
ip route get <destination-ip>
ping <destination-ip>
nc -vz <destination-ip> <port>
```

Record the result of each stage.

**Goal:** Prove whether a failure is DNS, routing, reachability, or TCP-port related.

---

### Lab 6 — Route 53 Public Hosted Zone Exercise

In an AWS test account, create a Route 53 public hosted zone for a domain you control or use an existing test zone.

Create a simple DNS record and verify it from Linux with:

```bash
dig <hostname>
dig +short <hostname>
```

**Goal:** Connect the Route 53 hosted-zone concept to real DNS queries.

---

### Lab 7 — Route 53 Private DNS Exercise

Using a test VPC, create/associate a private hosted zone and create an internal record such as:

```text
app.internal.example.com
```

From an instance in the associated VPC, test:

```bash
dig app.internal.example.com
```

Then test connectivity to the returned private IP.

**Goal:** Understand the difference between public DNS and VPC-internal DNS resolution.

---

### Lab 8 — Diagnose a DNS-Based Service Failure

Simulate or document a service failure using:

```text
api.example.com:443
```

Investigate in this order:

```text
DNS name
 ↓
DNS answer
 ↓
resolver used
 ↓
IP address
 ↓
route
 ↓
TCP/443
 ↓
listener
 ↓
AWS Security Group / NACL
 ↓
application
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

**Goal:** Produce evidence showing the first failed layer instead of simply reporting "DNS is down" or "firewall issue".

---

### DAY 2 Capstone — DNS + AWS Troubleshooting Report

Create a short incident report for an unreachable AWS service containing:

```text
Hostname:
Resolver:
DNS status:
DNS record type:
Resolved IP:
TTL:
Route:
TCP port:
Listener:
Security Group:
Network ACL:
Application:
Root cause:
Evidence:
```

**Success criteria:** You can clearly separate DNS resolution from the network and application path and explain how Route 53 fits into the architecture.
