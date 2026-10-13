# Networking

Networking notes for Linux/AWS/DevOps troubleshooting.

## Learning Standard

These notes are written **beginner-first**. A topic should explain the basic terminology and mental model before moving into commands, production troubleshooting, or interview-level depth.

## Study Structure

- `DAY X - topic.md` → learning notes only
- `LABS.md` → all practical labs for Networking
- `INTERVIEW QUESTIONS.md` → all Networking interview questions
- `README.md` → Networking learning path and progress

**Do not create separate per-day LABS or INTERVIEW QUESTIONS files.**

DAY numbering starts at `DAY 1` independently for Networking.

## Progress

- [DAY 1 - TCP/IP, Ports, Sockets and ss/netstat Troubleshooting](DAY%201%20-%20tcp-ip-ports-sockets-ss-netstat-troubleshooting.md) — TCP/IP basics, ports, sockets, `ip`, `ss`, `netstat`, `ping`, `traceroute`, `dig`, `nslookup`, TCP testing, and Linux/AWS connectivity troubleshooting.
- [DAY 2 - DNS Resolution, dig, nslookup and Route 53](DAY%202%20-%20DNS-resolution-dig-nslookup-route53.md) — DNS resolution, record types, `dig`, `nslookup`, TTL, delegation, Route 53 hosted zones, routing policies, private DNS, and DNS troubleshooting.
- [DAY 3 - HTTP/HTTPS, TLS Handshake and curl Troubleshooting](DAY%203%20-%20HTTP-HTTPS-TLS-curl-troubleshooting.md) — HTTP fundamentals, HTTPS, TLS handshake, certificates, SNI, `curl`, `openssl s_client`, AWS load balancer HTTPS paths, and production troubleshooting.

## Centralized Resources

- [LABS](LABS.md)
- [INTERVIEW QUESTIONS](INTERVIEW%20QUESTIONS.md)

## Core Troubleshooting Mental Model

```text
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
TCP
 ↓
TLS
 ↓
HTTP
 ↓
LOAD BALANCER / PROXY
 ↓
BACKEND
 ↓
APPLICATION
```

The goal is to identify the **first failed layer** instead of guessing the cause.
