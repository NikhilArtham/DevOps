# DAY 3 — HTTP/HTTPS, TLS Handshake and `curl` Troubleshooting

## Why This Topic Matters

In DevOps, many incidents eventually become:

```text
The server is reachable, but the application request is failing.
```

At this point, knowing only `ping` or `ss` is not enough. You need to understand what happens **after TCP connectivity succeeds**.

A simplified HTTPS request looks like:

```text
DNS
 ↓
IP / routing
 ↓
TCP connection
 ↓
TLS handshake
 ↓
HTTP request
 ↓
HTTP response
 ↓
Application
```

This topic teaches you how to identify which layer is failing.

---

# 1. What Is HTTP?

HTTP stands for **Hypertext Transfer Protocol**.

It is an application-layer protocol used for communication between clients and servers.

For example:

```text
Client → GET /health → Server
Client ← 200 OK ← Server
```

A client could be:

- Browser
- `curl`
- Application
- Load balancer
- Monitoring system
- Kubernetes probe

A server could be:

- Nginx
- Apache
- Java application
- Node.js application
- Python application
- Kubernetes service
- AWS load balancer

---

# 2. HTTP Request and Response

A basic HTTP request contains things such as:

```text
Method
Path
Headers
Optional body
```

Example:

```http
GET /health HTTP/1.1
Host: example.com
```

The server returns a response containing:

```text
Status code
Headers
Optional body
```

Example:

```text
HTTP/1.1 200 OK
Content-Type: application/json
```

---

# 3. Common HTTP Methods

## GET

Retrieve data.

```bash
curl https://example.com/api/users
```

## POST

Send data to create/process something.

```bash
curl -X POST https://example.com/api/users
```

## PUT

Usually replaces or updates a resource.

## PATCH

Partially updates a resource.

## DELETE

Deletes a resource.

For DevOps troubleshooting, the important point is that the HTTP method can affect the application's response even when the network and TLS layers are completely healthy.

---

# 4. Important HTTP Status Codes

## 2xx — Success

```text
200 OK
201 Created
204 No Content
```

## 3xx — Redirection

```text
301 Moved Permanently
302 Found
304 Not Modified
```

## 4xx — Client-side/request problem

```text
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
405 Method Not Allowed
408 Request Timeout
429 Too Many Requests
```

## 5xx — Server-side problem

```text
500 Internal Server Error
502 Bad Gateway
503 Service Unavailable
504 Gateway Timeout
```

### Important troubleshooting distinction

A `404` proves that an HTTP server or HTTP-aware component responded. It is therefore very different from a TCP timeout.

Similarly:

```text
502 → usually a proxy/load balancer received a bad response from an upstream
503 → service is unavailable
504 → gateway/proxy waited for an upstream response and timed out
```

These are clues, not automatic proof of one root cause.

---

# 5. What Is HTTPS?

HTTPS is HTTP carried over a TLS-protected connection.

Simplified:

```text
HTTP
 ↓
TLS encryption
 ↓
TCP
```

A common HTTPS connection uses:

```text
TCP port 443
```

HTTP commonly uses:

```text
TCP port 80
```

HTTPS gives the client protection for:

- Confidentiality
- Integrity
- Server authentication

---

# 6. HTTP vs HTTPS

```text
HTTP:
Client → TCP → HTTP → Server

HTTPS:
Client → TCP → TLS → HTTP → Server
```

This distinction is extremely useful during troubleshooting.

If TCP cannot connect, TLS has not started yet.

If TCP succeeds but TLS fails, investigate certificates, protocol versions, SNI, trust, or other TLS configuration.

If TLS succeeds but HTTP returns `503`, the problem is above the TLS layer.

---

# 7. TLS Handshake — Beginner Mental Model

TLS establishes a secure session before normal HTTPS application data is exchanged.

A simplified TLS 1.3 flow is:

```text
Client
  ↓ ClientHello
Server
  ↓ ServerHello + certificate + handshake messages
Client
  ↓ certificate verification / key agreement messages
Both
  ↓ encrypted application traffic
```

The exact wire exchange is more detailed, but the mental model is enough for initial troubleshooting.

---

# 8. What Is `ClientHello`?

The client starts TLS negotiation with a `ClientHello`.

It can contain information such as:

- Supported TLS versions
- Supported cipher suites
- Server Name Indication (SNI)
- Key-share information
- Other TLS extensions

A key DevOps concept is **SNI**.

A single load balancer or server can host multiple HTTPS domains, and SNI helps the server determine which hostname the client is requesting during TLS negotiation.

---

# 9. What Is SNI?

SNI means **Server Name Indication**.

Suppose one load balancer serves:

```text
api.example.com
payments.example.com
admin.example.com
```

The client can tell the TLS endpoint which hostname it wants during the TLS handshake.

This allows the endpoint to select the appropriate certificate/configuration.

Troubleshooting example:

```bash
curl https://api.example.com
```

may behave differently from a connection made directly to an IP because the hostname influences TLS through SNI.

---

# 10. TLS Certificates

A server certificate helps the client verify the server identity.

A certificate commonly contains information such as:

- Subject
- Subject Alternative Names (SANs)
- Issuer
- Validity period
- Public key

For modern HTTPS troubleshooting, the **SAN** is especially important because the requested hostname must match a valid name in the certificate.

Example:

```text
Requested hostname:
api.example.com

Certificate SAN:
api.example.com
```

Good match.

But:

```text
Requested hostname:
api.example.com

Certificate SAN:
www.example.com
```

can cause certificate hostname verification failure.

---

# 11. Certificate Chain

A server certificate may be signed by an intermediate Certificate Authority (CA), which ultimately chains to a trusted root CA.

Conceptually:

```text
Root CA
   ↓
Intermediate CA
   ↓
Server certificate
```

The client uses its trusted CA store to validate the chain.

A missing or incorrect intermediate certificate can cause TLS failures even when the server certificate itself looks valid.

---

# 12. TLS Failure vs HTTP Failure

This distinction is extremely important.

### TLS failure

Example:

```text
curl: (60) SSL certificate problem
```

The HTTP request may never have been sent because TLS validation failed.

### HTTP failure

Example:

```text
HTTP/1.1 503 Service Unavailable
```

TLS successfully completed and the client received an HTTP response.

Therefore:

```text
TLS error ≠ HTTP error
```

---

# 13. `curl` — The Most Useful HTTP Troubleshooting Tool

`curl` is a command-line client for transferring data using many protocols, especially HTTP/HTTPS.

Basic request:

```bash
curl https://example.com
```

Show response headers:

```bash
curl -I https://example.com
```

Follow redirects:

```bash
curl -L https://example.com
```

Show verbose connection details:

```bash
curl -v https://example.com
```

Show even more TLS/connection debugging:

```bash
curl -vvv https://example.com
```

---

# 14. Understanding `curl -v`

Run:

```bash
curl -v https://example.com
```

You may see stages similar to:

```text
* Trying 93.184.216.34:443...
* Connected to example.com
* TLS handshake...
> GET / HTTP/1.1
> Host: example.com
< HTTP/1.1 200 OK
```

This is extremely useful because it shows where the request stops.

For example:

```text
* Trying ...
```

then nothing useful:

→ investigate routing/connectivity/firewall/TCP.

If you see:

```text
Connected to ...
```

but TLS fails:

→ investigate TLS.

If TLS succeeds and you receive:

```text
HTTP/1.1 503
```

→ investigate HTTP/application/upstream behavior.

---

# 15. Useful `curl` Options

## `-I` — Headers only

```bash
curl -I https://example.com
```

Useful for quickly inspecting HTTP status and headers.

## `-v` — Verbose

```bash
curl -v https://example.com
```

Useful for connection, TLS, request, and response debugging.

## `-k` — Skip certificate verification

```bash
curl -k https://example.com
```

This disables normal TLS certificate verification.

**Do not use `-k` as the permanent fix.** It is useful as a diagnostic comparison.

Example:

```text
curl https://server.example.com
→ certificate verification fails

curl -k https://server.example.com
→ HTTP 200
```

This strongly suggests the network/TLS connection works but certificate validation needs investigation.

## `-L` — Follow redirects

```bash
curl -L http://example.com
```

Useful when HTTP returns `301` or `302`.

## `-o` — Save response to a file

```bash
curl -o response.html https://example.com
```

## `-w` — Print selected metrics

Example:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://example.com
```

This prints only the HTTP status code.

Useful metrics include:

```text
http_code
time_namelookup
time_connect
time_appconnect
time_starttransfer
time_total
```

Example:

```bash
curl -s -o /dev/null \
  -w 'code=%{http_code} dns=%{time_namelookup}s connect=%{time_connect}s tls=%{time_appconnect}s total=%{time_total}s\n' \
  https://example.com
```

This is useful for identifying which stage is slow.

---

# 16. Test HTTP Without HTTPS

To test plain HTTP:

```bash
curl -v http://example.com
```

To test HTTPS:

```bash
curl -v https://example.com
```

If HTTP works but HTTPS fails, focus on:

```text
TCP/443
TLS
certificate
SNI
HTTPS listener
load balancer TLS configuration
```

Do not assume the application itself is down.

---

# 17. Inspect a TLS Certificate With OpenSSL

`openssl s_client` is extremely useful for low-level TLS troubleshooting.

Basic test:

```bash
openssl s_client -connect example.com:443
```

Include SNI explicitly:

```bash
openssl s_client -connect example.com:443 -servername example.com
```

The `-servername` option supplies the hostname through SNI.

This is important when multiple HTTPS sites share one IP/load balancer.

Useful certificate inspection:

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null
```

Look for:

- Certificate chain
- TLS version
- Cipher
- Verification result
- Certificate subject/SAN
- Expiration information

---

# 18. Common TLS Errors

## Certificate expired

The certificate validity period has ended.

## Certificate hostname mismatch

The requested hostname does not match the certificate's valid names.

## Unknown CA / unable to verify

The client does not trust the certificate chain or cannot build a trusted chain.

## TLS protocol version mismatch

Client and server cannot agree on a supported TLS version.

## Handshake failure

The TLS negotiation could not complete. Causes can include incompatible protocol/cipher configuration, certificate issues, SNI/configuration problems, or other endpoint configuration problems.

---

# 19. AWS Components That Commonly Sit in the HTTPS Path

A common AWS architecture might look like:

```text
User
 ↓
Route 53
 ↓
CloudFront / ALB
 ↓
Target Group
 ↓
EC2 / ECS / EKS
 ↓
Application
```

Possible TLS termination points include:

```text
Client
 ↓ HTTPS
ALB
 ↓ HTTP/HTTPS
Application
```

or:

```text
Client
 ↓ HTTPS
CloudFront
 ↓ HTTPS
ALB
 ↓ HTTP
Application
```

You must know **where TLS terminates** before troubleshooting certificates.

---

# 20. AWS Load Balancer HTTPS Troubleshooting

For an Application Load Balancer, check the listener:

```text
HTTPS : 443
```

Then check:

```text
Listener
 ↓
Certificate
 ↓
Listener rules
 ↓
Target group
 ↓
Target health
 ↓
Application
```

A client may successfully complete TLS with the ALB while the ALB cannot reach a backend target.

That can result in an HTTP error such as:

```text
502
503
504
```

Therefore:

```text
TLS to ALB successful
≠
Backend application healthy
```

---

# 21. AWS Security Groups and HTTPS

For a public HTTPS endpoint, the relevant Security Group commonly needs inbound TCP `443` from the appropriate client source.

For an ALB-to-EC2 architecture, the EC2 Security Group should normally allow the application port from the ALB Security Group rather than broadly exposing the application port to the internet.

Conceptually:

```text
Internet
   ↓ TCP/443
ALB Security Group
   ↓ application port
EC2 Security Group
   ↓
Application
```

This is a better security model than exposing every backend port publicly.

---

# 22. Kubernetes HTTP/HTTPS Troubleshooting Connection

A typical Kubernetes path may look like:

```text
Client
 ↓
Ingress / Gateway
 ↓
Service
 ↓
Pod
```

For HTTPS troubleshooting, investigate:

- Ingress/Gateway configuration
- TLS certificate/Secret
- Hostname/SNI
- Service port
- Target port
- Pod readiness
- NetworkPolicy if used
- Application listener

The same layered troubleshooting model still applies.

---

# 23. Realistic Failure Scenario — HTTPS Returns 503

### Incident

Users report:

```text
https://api.example.com is returning 503
```

### Step 1 — DNS

```bash
dig +short api.example.com
```

Suppose it returns the ALB address.

DNS is working sufficiently for the request to proceed.

### Step 2 — TCP

```bash
nc -vz api.example.com 443
```

Suppose:

```text
succeeded
```

TCP/443 is reachable.

### Step 3 — TLS

```bash
curl -v https://api.example.com/health
```

Suppose TLS succeeds and the certificate is accepted.

Therefore TLS is not the immediate problem.

### Step 4 — HTTP

The response is:

```text
HTTP/1.1 503 Service Unavailable
```

Now investigate the ALB/backend layer.

Check:

```text
ALB listener
 ↓
Listener rules
 ↓
Target group
 ↓
Target health
 ↓
EC2/application
```

### Root cause example

Suppose all targets are unhealthy because the application is listening on port `8080`, while the target group health check is configured for port `8081`.

The network and TLS layers were healthy. The failure was in the load-balancer-to-application configuration.

### Lesson

Do not stop troubleshooting at the first familiar command.

```text
DNS ✓
TCP ✓
TLS ✓
HTTP ✗
Backend configuration ✗
```

This is the level of reasoning expected in production DevOps troubleshooting.

---

# 24. Realistic Failure Scenario — Certificate Error

Suppose:

```bash
curl https://api.example.com
```

returns a certificate verification error.

Run:

```bash
curl -k -v https://api.example.com
```

If `-k` allows the request to reach an HTTP response, the TCP connection and TLS exchange are likely functioning enough to establish the connection, but certificate verification needs investigation.

Then inspect the certificate:

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

Do not permanently solve certificate problems by using `-k`.

---

# 25. Production Troubleshooting Decision Tree

When HTTPS fails:

```text
Does DNS resolve?
        ↓ yes
Can TCP/443 connect?
        ↓ yes
Does TLS handshake complete?
        ↓ yes
Does certificate validation pass?
        ↓ yes
Does HTTP return a response?
        ↓ yes
Is the status code expected?
        ↓ no
Investigate proxy/load balancer/application/backend
```

Useful commands:

```bash
dig +short <hostname>

ip route get <destination-ip>

nc -vz <hostname> 443

curl -v https://<hostname>

curl -I https://<hostname>

openssl s_client -connect <hostname>:443 -servername <hostname> </dev/null
```

---

# 26. Common Mistakes

### Mistake 1 — Using `ping` to troubleshoot HTTPS

`ping` tests ICMP, not HTTPS.

Use:

```bash
curl -v https://example.com
```

for application-layer testing.

### Mistake 2 — Using `curl -k` as the fix

`-k` disables certificate verification. It is a diagnostic tool, not a production certificate solution.

### Mistake 3 — Assuming TCP 443 being open means HTTPS works

TCP can succeed while TLS fails.

### Mistake 4 — Assuming successful TLS means the application is healthy

TLS can terminate at an ALB or CloudFront while the backend application is unhealthy.

### Mistake 5 — Looking only at the server

In AWS, the failure may be at:

```text
Route 53
CloudFront
ALB
Security Group
Target Group
EC2
Application
```

### Mistake 6 — Ignoring hostname/SNI

Testing an IP directly can produce a different TLS result than testing the real hostname because certificate selection and validation depend on the requested hostname.

---

# 27. Interview-Ready Mental Model

Remember this sequence:

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
HTTP
 ↓
Application
```

And remember what each tool proves:

```text
 dig
 → DNS

 nc
 → TCP port connectivity

 ss
 → local socket/listener

 curl -v
 → HTTP + TLS + connection flow

 openssl s_client
 → detailed TLS inspection
```

The key production skill is not memorizing commands. It is knowing **which command proves which layer**.

---

## Beginner Takeaway

If you remember only these commands:

```bash
curl -v https://example.com

curl -I https://example.com

curl -k -v https://example.com

curl -L https://example.com

openssl s_client -connect example.com:443 -servername example.com </dev/null

nc -vz example.com 443
```

you can investigate a large percentage of common HTTP/HTTPS connectivity and TLS incidents.

Your troubleshooting model should become:

```text
DNS
 ↓
TCP/443
 ↓
TLS handshake
 ↓
Certificate validation
 ↓
HTTP response
 ↓
Load balancer/proxy
 ↓
Backend
 ↓
Application
```

**Labs and interview questions for this topic are maintained in the centralized Networking files.**
