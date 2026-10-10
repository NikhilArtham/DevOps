# DAY 1 - TCP/IP, Ports, Sockets and `ss`/`netstat` Troubleshooting

> **Scope:** Linux + AWS networking
>
> **Goal:** Learn how to trace a connection from a client to a destination and separate DNS, routing, port-listening, firewall, and AWS Security Group problems.

---

## 1. First: What Is Networking?

Networking is simply **how one computer communicates with another computer**.

For example:

```text
Your laptop → AWS EC2 server → application running on port 8080
```

If the application cannot be reached, the problem can exist at several different layers:

```text
Name resolution
      ↓
DNS
      ↓
Routing
      ↓
Destination IP reachable?
      ↓
Port reachable?
      ↓
Is an application listening?
      ↓
Firewall / Security Group
      ↓
Application response
```

The most important troubleshooting skill is **not guessing**. Check each stage separately.

---

# 2. IP Address

An **IP address** identifies a network interface on a network.

Example:

```text
192.168.1.10
10.0.1.25
```

For an AWS EC2 instance you may have:

```text
Private IP: 10.0.2.15
Public IP: 3.x.x.x
```

A client normally connects to an IP address and a port.

```text
10.0.2.15:8080
```

Here:

- `10.0.2.15` = destination IP
- `8080` = destination port

---

# 3. What Is TCP/IP?

**TCP/IP** is the collection of networking protocols used for communication between systems.

For DevOps troubleshooting, understand these first:

- **IP** → identifies where packets should go.
- **TCP** → provides reliable connection-oriented communication.
- **UDP** → connectionless communication; commonly used by DNS and other services.
- **DNS** → converts names such as `example.com` into IP addresses.
- **ICMP** → used by tools such as `ping` for reachability testing.

A useful simplified model is:

```text
Application
   ↓
TCP / UDP
   ↓
IP
   ↓
Network interface
   ↓
Network
```

---

# 4. TCP Connection: The Basic Idea

TCP normally establishes a connection using a **three-way handshake**:

```text
Client                         Server
  |                              |
  | -------- SYN --------------> |
  | <------- SYN-ACK ----------- |
  | -------- ACK --------------> |
  |                              |
  | ===== TCP connection ======= |
```

If the TCP connection cannot be established, the application may show errors such as:

```text
Connection timed out
Connection refused
No route to host
```

These errors are useful clues, but they do **not** always identify the exact root cause by themselves.

---

# 5. What Is a Port?

A **port** identifies a network service on a machine.

Think of an IP address as a building address and a port as a specific door in that building.

Examples:

```text
22    → SSH
80    → HTTP
443   → HTTPS
53    → DNS
3306  → MySQL
5432  → PostgreSQL
8080  → Common application port
```

A server can have many services running on different ports:

```text
10.0.1.10:22
10.0.1.10:80
10.0.1.10:443
10.0.1.10:8080
```

---

# 6. What Is a Socket?

A **socket** represents an endpoint of network communication.

A useful simplified view is:

```text
Protocol + Source IP + Source Port + Destination IP + Destination Port
```

For example:

```text
TCP
Client:      10.0.1.20:53142
Server:      10.0.2.15:443
```

The client usually uses a temporary **ephemeral port**, while the server listens on a known service port.

---

# 7. Listening vs Established

This distinction is extremely important in troubleshooting.

### LISTEN

A process is waiting for incoming connections.

```text
0.0.0.0:8080    LISTEN
```

This means something on the machine is listening on port `8080`.

### ESTABLISHED

A TCP connection has been successfully established between two endpoints.

```text
10.0.1.20:53142 → 10.0.2.15:8080    ESTABLISHED
```

### TIME-WAIT

A connection has been closed, but the operating system keeps the connection information temporarily to prevent delayed packets from being confused with a new connection.

Large numbers of `TIME-WAIT` connections can be normal in busy systems, but an unexpected increase may deserve investigation.

---

# 8. `ip` — Inspect the Network Configuration

The `ip` command is the modern Linux networking command.

## Show IP addresses

```bash
ip addr
```

Short form:

```bash
ip a
```

Useful information includes:

- interface name
- IPv4 address
- IPv6 address
- interface state

## Show routing table

```bash
ip route
```

Example:

```text
10.0.2.0/24 dev eth0
10.0.0.0/16 via 10.0.2.1 dev eth0
default via 10.0.2.1 dev eth0
```

The `default` route is where traffic goes when no more specific route exists.

## Show a specific route

```bash
ip route get 8.8.8.8
```

This is extremely useful because it tells you which route/interface Linux would use to reach a destination.

---

# 9. `ss` — Inspect Ports and Connections

`ss` means **socket statistics**.

It is the preferred modern tool for inspecting sockets on Linux.

## Show listening TCP ports

```bash
ss -lnt
```

Meaning:

- `-l` = listening sockets
- `-n` = show numeric addresses/ports instead of resolving names
- `-t` = TCP

Example:

```text
LISTEN 0 128 0.0.0.0:22    0.0.0.0:*
LISTEN 0 128 0.0.0.0:8080  0.0.0.0:*
```

This tells us that something is listening on TCP ports `22` and `8080`.

## Show listening TCP/UDP ports with process information

```bash
sudo ss -lntup
```

Additional flags:

- `-u` = UDP
- `-p` = show process information

Example:

```text
LISTEN 0 128 0.0.0.0:8080 0.0.0.0:* users:(("java",pid=1234,fd=87))
```

Now you know the process behind the port.

## Show established TCP connections

```bash
ss -nt state established
```

## Check one specific port

```bash
sudo ss -lntp | grep ':8080'
```

If nothing appears, there may be no TCP service listening on port `8080`.

---

# 10. `netstat`

`netstat` is an older tool from the `net-tools` package.

You may still see it in older production environments and interviews.

Examples:

```bash
netstat -lnt
```

```bash
sudo netstat -lntp
```

```bash
netstat -ant
```

Know the equivalent modern command:

```text
netstat -lntp
        ↓
ss -lntp
```

Prefer `ss` on modern Linux systems.

---

# 11. `ping` — Test Basic IP Reachability

```bash
ping 8.8.8.8
```

`ping` uses ICMP Echo Request/Echo Reply.

It can help answer:

> Can I get an ICMP response from this IP?

But an important DevOps lesson is:

> **Ping success does NOT prove that an application port is reachable.**

For example:

```text
ping server       → SUCCESS
server:443        → FAIL
```

The host can be reachable while TCP/443 is blocked or the application is not listening.

Also, some systems intentionally block ICMP, so ping failure does not automatically mean the server is unreachable.

---

# 12. `traceroute` — Trace the Network Path

```bash
traceroute example.com
```

On some systems you may need:

```bash
sudo traceroute example.com
```

The command shows intermediate hops between the source and destination.

Conceptually:

```text
Client
  ↓
Router 1
  ↓
Router 2
  ↓
Router 3
  ↓
Destination
```

Useful for investigating where traffic may stop or become slow.

However, `* * *` does not automatically mean the route is broken. Routers may simply refuse or rate-limit traceroute responses.

---

# 13. `dig` — Troubleshoot DNS

DNS converts a hostname into an IP address.

Example:

```bash
dig example.com
```

For a shorter answer:

```bash
dig +short example.com
```

Check a specific DNS server:

```bash
dig @8.8.8.8 example.com
```

This is useful when you suspect that your normal DNS resolver is the problem.

---

# 14. `nslookup`

`nslookup` is another DNS troubleshooting tool.

```bash
nslookup example.com
```

You may also see:

```bash
nslookup example.com 8.8.8.8
```

Modern Linux troubleshooting generally favors `dig`, but you should recognize both.

---

# 15. The Most Important Troubleshooting Separation

When a service cannot be reached, do **not** immediately say:

> "The firewall is blocking it."

Separate the problem into stages.

```text
1. DNS
   ↓
2. Routing
   ↓
3. Destination host
   ↓
4. Port
   ↓
5. Listening process
   ↓
6. Host firewall
   ↓
7. AWS Security Group / Network ACL
   ↓
8. Application
```

---

# 16. Example: Application Is Unreachable

Suppose:

```text
Client: 10.0.1.20
Server: app.example.com
Port:   8080
```

The client runs:

```bash
curl http://app.example.com:8080
```

and gets a timeout.

Do not guess. Walk the path.

---

## Step 1 — DNS

```bash
dig +short app.example.com
```

Expected:

```text
10.0.2.15
```

If DNS returns nothing or the wrong IP, investigate DNS first.

---

## Step 2 — Routing

```bash
ip route get 10.0.2.15
```

Confirm Linux has a valid route and is using the expected interface/gateway.

---

## Step 3 — Basic Reachability

```bash
ping 10.0.2.15
```

Remember: ping failure is not conclusive because ICMP may be blocked.

---

## Step 4 — Is the Port Listening on the Server?

On the destination server:

```bash
sudo ss -lntp | grep ':8080'
```

If there is no listener:

```text
Client cannot connect
        ↓
No process listening on 8080
        ↓
Investigate application/service
```

If it is listening:

```text
LISTEN ... 0.0.0.0:8080
```

continue troubleshooting.

---

# 17. `127.0.0.1` vs `0.0.0.0`

This is a very common DevOps issue.

Suppose the application listens on:

```text
127.0.0.1:8080
```

This generally means it accepts connections only through the local loopback interface.

A remote machine trying:

```text
server-ip:8080
```

may fail.

If the application listens on:

```text
0.0.0.0:8080
```

it is listening on all IPv4 interfaces, subject to firewall/security controls.

Check with:

```bash
sudo ss -lntp | grep ':8080'
```

This is one of the first things to check when an application works locally but not remotely.

---

# 18. Test a TCP Port Directly

`nc` (netcat) is useful for testing TCP connectivity.

```bash
nc -vz 10.0.2.15 8080
```

Meaning:

- `-v` = verbose output
- `-z` = scan/check without sending application data

Possible result:

```text
Connection to 10.0.2.15 8080 port [tcp/*] succeeded!
```

This tells you TCP connectivity to that port succeeded.

If it fails, the next step is to determine why.

---

# 19. Timeout vs Connection Refused

These two errors are important clues.

## Connection refused

Usually means the destination was reachable but the TCP connection was actively rejected.

A common cause is:

```text
No application listening on that port
```

But do not treat the error as absolute proof; network devices and firewalls can also generate rejection behavior.

## Connection timed out

Usually means the connection attempt did not receive the expected response within the timeout period.

Possible causes include:

- security group blocking traffic
- network ACL issue
- host firewall
- routing problem
- destination unavailable
- network path problem

The correct response is to continue testing each layer.

---

# 20. Host Firewall

Linux may have a firewall such as:

- `ufw`
- `firewalld`
- `nftables`
- legacy `iptables` rules

Examples:

```bash
sudo ufw status
```

```bash
sudo firewall-cmd --list-all
```

For nftables:

```bash
sudo nft list ruleset
```

Do not modify firewall rules blindly in production. First understand what traffic should be allowed and why.

---

# 21. AWS Security Groups

In AWS, a common architecture is:

```text
Client
  ↓
AWS networking
  ↓
Security Group
  ↓
EC2 network interface
  ↓
Linux firewall
  ↓
Application port
```

A Security Group is a virtual firewall associated with resources such as EC2 network interfaces.

Example requirement:

```text
Application listens on TCP 8080
        ↓
Security Group must allow inbound TCP 8080
from the appropriate source
```

Do not simply open `0.0.0.0/0` for convenience. Allow the smallest appropriate source range.

---

# 22. Security Group vs Linux Firewall

These are different controls.

```text
AWS Security Group
        ↓
AWS network boundary
        ↓
Linux host
        ↓
Linux firewall
        ↓
Application
```

If the Security Group allows port `8080` but Linux blocks it, the connection can still fail.

If Linux allows `8080` but the Security Group blocks it, the connection can still fail.

You need to check both where applicable.

---

# 23. AWS Network ACLs

Network ACLs operate at the subnet level and are different from Security Groups.

For troubleshooting, keep this mental model:

```text
Security Group → resource/network interface level
Network ACL    → subnet level
Linux firewall → host level
Application    → process level
```

When traffic is unexpectedly blocked in AWS, determine which layer owns the rule instead of treating all firewall controls as the same thing.

---

# 24. A Practical End-to-End Troubleshooting Flow

Use this sequence when a service is unreachable:

```text
START
  ↓
Does DNS resolve?
  ↓ yes
Is the route correct?
  ↓ yes
Can the destination be reached?
  ↓
Can TCP connect to the port?
  ↓
Is the service listening?
  ↓
Is it listening on the correct interface?
  ↓
Does the host firewall allow it?
  ↓
Does AWS Security Group allow it?
  ↓
Do Network ACLs/routing allow it?
  ↓
Does the application itself work?
```

Useful commands:

```bash
dig +short app.example.com
ip route get <destination-ip>
ping <destination-ip>
traceroute <destination-ip>
nc -vz <destination-ip> <port>
sudo ss -lntp
sudo nft list ruleset
```

AWS side:

```text
Security Group
Network ACL
Route Table
Subnet
EC2 network interface
```

---

# 25. Example Production Incident

### Symptom

An application on EC2 is supposed to be available at:

```text
https://app.example.com:8443
```

Users report timeout errors.

### Investigation

#### 1. DNS

```bash
dig +short app.example.com
```

Returns the expected public IP.

**DNS looks good.**

#### 2. Route

From the client:

```bash
ip route get <destination-ip>
```

Route exists.

**Routing looks good.**

#### 3. TCP connectivity

```bash
nc -vz <destination-ip> 8443
```

Timeout.

#### 4. Server listener

On EC2:

```bash
sudo ss -lntp | grep ':8443'
```

Output:

```text
LISTEN ... 127.0.0.1:8443
```

Root cause candidate:

```text
Application is listening only on localhost.
```

Even if the AWS Security Group allows `8443`, remote clients cannot reach an application that is bound only to loopback.

### Lesson

Always check the **actual listening address**, not just the port.

---

# 26. Another Production Scenario: Everything Looks Right Except AWS

Suppose:

```text
DNS        → correct
Route      → correct
Application → LISTEN 0.0.0.0:8080
Linux FW   → allows 8080
```

But:

```bash
nc -vz <server-ip> 8080
```

still times out.

Now inspect AWS:

```text
Security Group inbound rules
        ↓
Network ACL
        ↓
Subnet route table
        ↓
Internet Gateway / NAT / Load Balancer path
```

This is why DevOps networking troubleshooting must combine **Linux knowledge and cloud networking knowledge**.

---

# 27. Common Mistakes

### Mistake 1: `ping` works, therefore port works

Wrong.

```text
ICMP reachability ≠ TCP port reachability
```

### Mistake 2: Port is open, therefore application works

Wrong.

A process may listen but the application can still be unhealthy.

### Mistake 3: Security Group allows the port, therefore it must work

Wrong.

Check routing, NACLs, host firewall, listener, and application.

### Mistake 4: `ss` shows `8080`, therefore remote clients can connect

Not necessarily.

Check whether it is:

```text
127.0.0.1:8080
```

or:

```text
0.0.0.0:8080
```

### Mistake 5: `traceroute` shows `*`, therefore that router is broken

Not necessarily. Intermediate devices can intentionally suppress traceroute responses.

---

# 28. Linux vs AWS Troubleshooting Mental Model

### Linux side

```text
Interface
  ↓
IP address
  ↓
Route
  ↓
Socket
  ↓
Listening process
  ↓
Host firewall
  ↓
Application
```

### AWS side

```text
DNS
  ↓
Route table
  ↓
Subnet
  ↓
Network ACL
  ↓
Security Group
  ↓
ENI / EC2
  ↓
Linux networking
  ↓
Application
```

You need to understand where these two worlds meet.

---

# 29. Quick Command Cheat Sheet

| Goal | Command |
|---|---|
| Show IP addresses | `ip addr` |
| Show routes | `ip route` |
| Check route to destination | `ip route get <IP>` |
| Test ICMP reachability | `ping <IP>` |
| Trace path | `traceroute <host>` |
| Resolve DNS | `dig <host>` |
| Short DNS result | `dig +short <host>` |
| Alternative DNS tool | `nslookup <host>` |
| Show listening TCP ports | `ss -lnt` |
| Show listening TCP/UDP + process | `sudo ss -lntup` |
| Show established TCP connections | `ss -nt state established` |
| Check a specific listener | `sudo ss -lntp \| grep ':8080'` |
| Test TCP port | `nc -vz <IP> <PORT>` |
| Legacy socket tool | `netstat -lntp` |
| Check UFW | `sudo ufw status` |
| Check nftables | `sudo nft list ruleset` |

---

# 30. Beginner Takeaway

If you remember only one thing from this topic, remember this troubleshooting chain:

```text
NAME
 ↓
DNS
 ↓
IP
 ↓
ROUTE
 ↓
HOST
 ↓
PORT
 ↓
LISTENER
 ↓
FIREWALL / SECURITY GROUP
 ↓
APPLICATION
```

When a service is unreachable, move through the chain one step at a time.

Do not jump directly to:

> "Firewall issue."

Instead ask:

> **Which exact stage of the connection path is failing?**

That mindset is much more valuable in production than memorizing individual commands.
