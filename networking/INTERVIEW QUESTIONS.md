# Networking Interview Questions

Centralized interview questions for Networking. Questions from every Networking DAY will be added here.

---

## DAY 1 — TCP/IP, Ports, Sockets and `ss`/`netstat`

### 1. What is an IP address?

<details>
<summary>Answer</summary>

An IP address identifies a network interface on a network so traffic can be routed to the correct destination. In troubleshooting, an application connection commonly looks like `10.0.2.15:8080`, where the IP identifies the host and the port identifies the service endpoint.

</details>

### 2. What is the difference between an IP address and a port?

<details>
<summary>Answer</summary>

The IP address identifies the network destination, while the port identifies a particular service or socket on that destination. For example, `10.0.1.10:443` means TCP traffic to port 443 on host `10.0.1.10`.

</details>

### 3. What is a TCP three-way handshake?

<details>
<summary>Answer</summary>

TCP normally establishes a connection using SYN → SYN-ACK → ACK. This synchronizes the endpoints before application data is exchanged.

</details>

### 4. What is a socket?

<details>
<summary>Answer</summary>

A socket is a network communication endpoint. A TCP connection can be identified using protocol, source IP, source port, destination IP, and destination port.

</details>

### 5. What is the difference between LISTEN and ESTABLISHED in `ss`?

<details>
<summary>Answer</summary>

`LISTEN` means a process is waiting for incoming TCP connections. `ESTABLISHED` means a TCP connection has already been successfully established between two endpoints.

</details>

### 6. Why is `ss` preferred over `netstat` on modern Linux?

<details>
<summary>Answer</summary>

`ss` is the modern socket-statistics utility and is generally available on current Linux systems. `netstat` comes from the older `net-tools` package but is still commonly seen in legacy systems and interviews.

</details>

### 7. What does `ss -lntp` mean?

<details>
<summary>Answer</summary>

`-l` shows listening sockets, `-n` keeps addresses and ports numeric, `-t` selects TCP sockets, and `-p` shows the process using the socket. It is a useful command for finding which process is listening on a TCP port.

</details>

### 8. What is the difference between `127.0.0.1` and `0.0.0.0` when an application listens on a port?

<details>
<summary>Answer</summary>

`127.0.0.1` is the loopback address, so a service bound only to it normally accepts local connections. `0.0.0.0` means the service is listening on all IPv4 interfaces, subject to firewall and network controls. This is a common reason an application works locally but not remotely.

</details>

### 9. Does successful `ping` prove that TCP port 443 is reachable?

<details>
<summary>Answer</summary>

No. `ping` tests ICMP reachability, while HTTPS uses TCP port 443. ICMP can succeed while TCP/443 is blocked, no process is listening, or an intermediate network control prevents the connection.

</details>

### 10. Does failed `ping` prove that a server is unreachable?

<details>
<summary>Answer</summary>

No. ICMP may be blocked or rate-limited. Test the actual application port with tools such as `nc`, `curl`, or the application client and inspect routing and security controls.

</details>

### 11. What is the difference between connection refused and connection timed out?

<details>
<summary>Answer</summary>

A refusal commonly indicates that the destination actively rejected the connection, often because no service is listening. A timeout means the expected connection response was not received in time and can indicate filtering, routing, host availability, or other network-path problems. These are clues, not absolute proof of a single root cause.

</details>

### 12. How do you check which process is listening on port 8080?

<details>
<summary>Answer</summary>

Use:

```bash
sudo ss -lntp | grep ':8080'
```

The `-p` option exposes process information, allowing you to connect the socket to the owning process.

</details>

### 13. How do you check which route Linux will use to reach an IP?

<details>
<summary>Answer</summary>

Use:

```bash
ip route get <destination-ip>
```

This is more useful than only viewing the routing table because it shows the route Linux would select for that specific destination.

</details>

### 14. How do you troubleshoot DNS separately from connectivity?

<details>
<summary>Answer</summary>

First resolve the name:

```bash
dig +short example.com
```

Then test the returned IP separately. You can also query a specific DNS resolver:

```bash
dig @8.8.8.8 example.com
```

This prevents a DNS problem from being confused with a TCP, routing, or firewall problem.

</details>

### 15. What is `traceroute` used for?

<details>
<summary>Answer</summary>

`traceroute` shows the intermediate network hops toward a destination and can help identify where a path may become problematic. A `*` at a hop does not automatically mean that router is broken because devices may suppress or rate-limit traceroute responses.

</details>

### 16. How would you troubleshoot an unreachable service from a client to an EC2 instance?

<details>
<summary>Answer</summary>

Use a layered approach:

```text
DNS
→ route
→ destination reachability
→ TCP port
→ server listener
→ bind address
→ Linux firewall
→ Security Group
→ Network ACL / AWS routing
→ application
```

Useful commands include `dig`, `ip route get`, `ping`, `nc -vz`, and `ss -lntp`. On AWS, inspect the Security Group, Network ACL, route table, subnet, and EC2 network interface as applicable.

</details>

### 17. An application is listening on port 8080, but clients cannot connect. What do you check next?

<details>
<summary>Answer</summary>

First inspect the listening address. If it is `127.0.0.1:8080`, remote clients normally cannot connect. If it is bound to an appropriate interface such as `0.0.0.0:8080`, continue with Linux firewall, AWS Security Group, Network ACL, routing, and client-side TCP testing.

</details>

### 18. What is the difference between an AWS Security Group and a Linux firewall?

<details>
<summary>Answer</summary>

A Security Group is an AWS-level network control associated with resources such as network interfaces. A Linux firewall operates inside the operating system. Both can affect whether traffic reaches an application, so allowing a port in one does not automatically mean the other allows it.

</details>

### 19. What is the difference between a Security Group and a Network ACL?

<details>
<summary>Answer</summary>

A Security Group is associated with resources/network interfaces and is stateful. A Network ACL operates at the subnet level and is stateless, so inbound and outbound traffic rules must be considered separately. Both can affect connectivity in AWS.

</details>

### 20. What is your general methodology when a production service is unreachable?

<details>
<summary>Answer</summary>

Do not start by guessing the firewall. Prove each layer in order:

```text
Name → DNS → IP → route → host → port → listener → firewall/security controls → application
```

The goal is to identify the **first failed layer** and collect evidence for the root cause.

</details>
