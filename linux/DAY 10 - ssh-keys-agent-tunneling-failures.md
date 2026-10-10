# DAY 10 — SSH: Keys, Agent, Tunneling & Common Failures

> **Linux / AWS — DevOps Job-Switch Notes**

## 🎯 Focus

- SSH key-based authentication
- `ssh-agent` and agent forwarding
- Jump hosts / `ProxyJump`
- Local, remote and dynamic SSH tunneling
- Linux SSH configuration and AWS networking objects
- Production-style SSH failure troubleshooting

---

## 1. SSH Basics

SSH securely connects to a remote system and provides encrypted remote command execution.

```bash
ssh user@server
ssh -i ~/.ssh/my-key.pem ec2-user@<public-ip>
```

**Command anatomy:**
- `ssh` = SSH client
- `-i` = identity/private-key file
- `user@server` = remote username and destination

Common DevOps uses: EC2 access, bastions, remote commands, file transfer, and secure tunnels.

---

## 2. SSH Keys

A key pair contains:

```text
Private key → stays secret on the client
Public key  → installed on the server
```

Typical files:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
~/.ssh/authorized_keys
```

Generate a key:

```bash
ssh-keygen -t ed25519 -C "devops-lab"
```

- `-t` = key type
- `ed25519` = modern OpenSSH key algorithm
- `-C` = comment

Never commit or share a private key.

### Key permissions

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/authorized_keys
```

`700` means owner has `rwx`; `600` means owner has `rw` and group/others have no access.

A common client error is:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
```

Fix the private-key permissions before troubleshooting anything more complicated.

---

## 3. SSH Configuration

Client configuration:

```text
~/.ssh/config
/etc/ssh/ssh_config
```

Example:

```sshconfig
Host my-ec2
    HostName 203.0.113.10
    User ec2-user
    IdentityFile ~/.ssh/my-key.pem
    IdentitiesOnly yes
```

Then:

```bash
ssh my-ec2
```

Important options:
- `Host` = local alias/pattern
- `HostName` = actual destination
- `User` = remote username
- `IdentityFile` = private key
- `IdentitiesOnly yes` = use explicitly selected identities instead of trying many agent keys

Server configuration is commonly:

```text
/etc/ssh/sshd_config
```

Validate before restarting/reloading:

```bash
sudo sshd -t
```

`-t` tests configuration syntax. On a remote production host, never blindly restart SSH after editing its configuration—you can lock yourself out.

---

## 4. Debugging SSH

```bash
ssh -v user@host
ssh -vv user@host
ssh -vvv user@host
```

`-v`, `-vv`, `-vvv` progressively increase client-side debugging.

Look for evidence such as:

```text
Connecting to ...
Offering public key ...
Server accepts key ...
Authentications that can continue: publickey
Permission denied (publickey).
```

The goal is to identify **which layer failed**.

---

## 5. ssh-agent

`ssh-agent` holds private-key material in memory and lets SSH use the key without repeatedly asking for its passphrase.

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l
ssh-add -L
ssh-add -D
```

Flags:
- `ssh-agent -s` = print shell commands for setting the agent environment
- `ssh-add -l` = list loaded key fingerprints
- `ssh-add -L` = list loaded public keys
- `ssh-add -D` = remove all identities

A common problem is an agent containing many keys. SSH may offer several and hit a server's authentication-attempt limit.

Use:

```bash
ssh -o IdentitiesOnly=yes -i ~/.ssh/my-key.pem ec2-user@host
```

`-o` sets an SSH client option; `IdentitiesOnly=yes` restricts identity selection.

---

## 6. Agent Forwarding

Architecture:

```text
Laptop → Bastion → Private EC2
```

With agent forwarding:

```bash
ssh -A ec2-user@bastion
```

The private key stays on the laptop; the bastion gets access to the forwarded agent during the session.

**Security warning:** a compromised intermediate host may be able to use the forwarded agent while the session is active. Enable forwarding only when needed.

---

## 7. ProxyJump / Bastion

Modern bastion access often uses:

```bash
ssh -J ec2-user@bastion ec2-user@private-server
```

`-J` = ProxyJump through an intermediate SSH host.

Equivalent configuration:

```sshconfig
Host private-server
    HostName 10.0.2.50
    User ec2-user
    ProxyJump ec2-user@bastion
```

### ProxyJump vs agent forwarding

```text
ProxyJump       → network path through a bastion
Agent forwarding → authentication agent is forwarded
```

They solve different problems.

---

## 8. SSH Tunneling

Three important forwarding modes:

```text
-L  Local forwarding
-R  Remote forwarding
-D  Dynamic SOCKS forwarding
```

### Local forwarding

```bash
ssh -L 8080:10.0.2.50:80 ec2-user@bastion
```

Traffic:

```text
Laptop localhost:8080
        ↓
SSH connection
        ↓
Bastion → 10.0.2.50:80
```

`-L local_port:destination_host:destination_port`.

Then:

```bash
curl http://localhost:8080
```

The important point: the **SSH server side must be able to reach the destination**.

### Remote forwarding

```bash
ssh -R 9000:localhost:3000 user@server
```

`-R` exposes a client-side destination through a remote listening port. Use carefully in production because it can expose services.

### Dynamic forwarding

```bash
ssh -D 1080 ec2-user@bastion
```

`-D` creates a SOCKS proxy. Applications configured for SOCKS5 `localhost:1080` can send traffic through the SSH connection.

---

## 9. AWS SSH Architecture

Typical private-EC2 access:

```text
Laptop
  ↓ TCP 22
Bastion / Public EC2
  ↓ TCP 22
Private EC2
```

Important AWS objects/layers:

### EC2 key pair
Provides the initial public/private key material used for instance SSH access according to the AMI/OS configuration.

### Security Groups
Control network reachability. A good design is:

```text
Bastion SG:
  TCP 22 ← trusted admin IP

Private EC2 SG:
  TCP 22 ← Bastion SG
```

Avoid broad SSH exposure such as `0.0.0.0/0` when the architecture permits a narrower source.

### Other network layers

Also consider:
- route tables
- NACLs
- host firewall
- subnet routing
- DNS

### SSH vs IAM

AWS IAM authorizes AWS API actions. SSH authenticates to the Linux operating system.

```text
IAM → AWS API authorization
SSH → network + sshd + Linux authentication
```

Having IAM permission does not automatically make OS-level SSH work.

AWS Systems Manager Session Manager can provide a managed alternative to direct inbound SSH in suitable architectures.

---

## 10. Common Failure: Connection Timeout

```text
ssh: connect to host <host> port 22: Connection timed out
```

Think:

```text
DNS/IP
→ route
→ Security Group
→ NACL
→ firewall
→ sshd listening
```

Test from an appropriate host:

```bash
nc -vz <host> 22
```

If you have another access path to the server:

```bash
sudo ss -lntp | grep ':22'
sudo systemctl status sshd
```

**Do not start by changing `authorized_keys` when the error is a timeout.** The connection may not have reached SSH authentication.

---

## 11. Common Failure: Permission Denied (publickey)

```text
Permission denied (publickey).
```

Debug:

```bash
ssh -vvv -i ~/.ssh/my-key.pem ec2-user@host
ls -l ~/.ssh/my-key.pem
ssh-keygen -y -f ~/.ssh/my-key.pem
```

Server-side checks:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Likely causes:
- wrong private key
- wrong username
- public key missing from `authorized_keys`
- bad key/file ownership or permissions
- sshd authentication configuration
- wrong key being offered by the agent

---

## 12. Common Failure: Too Many Authentication Failures

Often caused by an agent containing many keys.

```bash
ssh-add -l
ssh -o IdentitiesOnly=yes -i ~/.ssh/my-key.pem ec2-user@host
```

The second command tells SSH to use the specified identity rather than trying a large collection of agent keys.

---

## 13. Common Failure: Host Key Changed

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

SSH stores known host keys in:

```text
~/.ssh/known_hosts
```

Do **not** blindly delete the warning. First determine whether the server was rebuilt, the IP/DNS changed, or the key was legitimately rotated. A malicious interception is also a possibility.

If the change is verified as legitimate:

```bash
ssh-keygen -R <hostname-or-ip>
```

Then validate the new fingerprint through a trusted source.

---

## 14. Common Failure: Tunnel Does Not Work

For:

```bash
ssh -L 8080:10.0.2.50:80 ec2-user@bastion
```

check in this order:

1. Is SSH itself connected?
2. Is local port `8080` bound?
3. Can the bastion reach `10.0.2.50:80`?
4. Is the destination service listening?
5. Do Security Groups/firewalls permit the traffic?

From the bastion:

```bash
nc -vz 10.0.2.50 80
```

A tunnel does not bypass destination-side networking.

---

# 15. Realistic Production Failure Scenario

### Incident

A DevOps engineer can reach a bastion but cannot SSH to a private EC2 instance:

```bash
ssh -J ec2-user@bastion ec2-user@10.0.2.50
```

Error:

```text
connect to host 10.0.2.50 port 22: Connection timed out
```

The engineer starts changing `authorized_keys`.

### Diagnosis

That is the wrong layer. A timeout normally means the connection is failing before public-key authentication.

From the bastion:

```bash
nc -vz 10.0.2.50 22
```

Suppose the private instance Security Group allows TCP/22 only from the **old bastion Security Group**.

### Root cause

The new bastion is not an allowed source for TCP/22.

### Fix

Allow TCP/22 on the private EC2 Security Group from the intended bastion Security Group, then retest:

```bash
nc -vz 10.0.2.50 22
ssh -J ec2-user@bastion ec2-user@10.0.2.50
```

### Lesson

Classify the error first:

```text
Timeout / refused      → network/service layer
Permission denied      → authentication/configuration
Host-key warning       → trust/identity
Tunnel failure         → forwarding/destination reachability
```

---

# 16. Production Troubleshooting Mental Model

Use:

```text
REACH → LISTEN → AUTHENTICATE → AUTHORIZE → SESSION → APPLICATION
```

### REACH

```bash
nc -vz host 22
```

### LISTEN

```bash
sudo ss -lntp | grep ':22'
```

### AUTHENTICATE

```bash
ssh -vvv user@host
```

### AUTHORIZE

Check `authorized_keys`, account restrictions and `sshd_config`.

### SESSION

Confirm the user can obtain the expected shell/command execution.

### APPLICATION

For tunnels, verify the destination application from the SSH server side.

---

# 17. Command Cheat Sheet

| Command | Meaning |
|---|---|
| `ssh user@host` | SSH connection |
| `ssh -i key user@host` | Select private key |
| `ssh -vvv user@host` | Detailed client debugging |
| `ssh -J user@bastion user@target` | ProxyJump |
| `ssh -A user@bastion` | Agent forwarding |
| `ssh -L 8080:host:80 user@server` | Local tunnel |
| `ssh -R 9000:localhost:3000 user@server` | Remote tunnel |
| `ssh -D 1080 user@server` | SOCKS proxy |
| `ssh-keygen -t ed25519` | Generate key |
| `ssh-keygen -R host` | Remove verified old known-host entry |
| `ssh-keygen -y -f key` | Derive public key from private key |
| `ssh-add key` | Add key to agent |
| `ssh-add -l` | List agent keys |
| `ssh-add -D` | Remove all agent keys |
| `chmod 600 private-key` | Protect private key |
| `ss -lntp` | Show listening TCP sockets |
| `systemctl status sshd` | Check SSH server |
| `journalctl -u sshd` | SSH server logs |

---

# 18. Practical Lab

Use a disposable Linux VM/EC2 environment. Do not experiment on production SSH access.

### Lab 1 — Key pair

```bash
mkdir -p ~/linux-labs/ssh
chmod 700 ~/linux-labs/ssh
ssh-keygen -t ed25519 -f ~/linux-labs/ssh/lab_key -C "ssh-lab"
ls -l ~/linux-labs/ssh
chmod 600 ~/linux-labs/ssh/lab_key
```

Tasks: identify private/public keys, explain permissions, and explain why the private key must remain secret.

### Lab 2 — ssh-agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/linux-labs/ssh/lab_key
ssh-add -l
ssh-add -D
ssh-add -l
```

Explain every command and the security implications of agent forwarding.

### Lab 3 — SSH debugging

```bash
ssh -vvv user@<test-host>
```

Record the destination, username, keys offered, authentication result and final error.

### Lab 4 — Local tunnel

If the test VM can reach an internal HTTP service:

```bash
ssh -L 8080:<internal-host>:80 user@<test-vm>
curl http://localhost:8080
```

Verify the internal destination is reachable **from the SSH server**.

### Lab 5 — Break and troubleshoot

One at a time, safely introduce:

1. Wrong username
2. Wrong private key
3. Private key mode `644`
4. Unreachable port
5. Tunnel destination with no listener

For every failure, classify it as network, SSH service, authentication, authorization/configuration, or tunnel/destination and record the evidence.

---

# 19. Interview Questions

<details><summary>1. What is SSH?</summary>SSH is an encrypted protocol for secure remote login and command execution.</details>
<details><summary>2. Public key vs private key?</summary>The public key is installed on the server; the private key remains secret on the client.</details>
<details><summary>3. Where is an SSH public key normally authorized?</summary>In the remote user's `~/.ssh/authorized_keys`.</details>
<details><summary>4. Why use chmod 600 for a private key?</summary>It prevents group and other users from reading the private key.</details>
<details><summary>5. What does ssh -vvv do?</summary>It provides detailed client-side debugging for connection and authentication.</details>
<details><summary>6. What is ssh-agent?</summary>A process that holds identities in memory and performs key authentication operations for SSH clients.</details>
<details><summary>7. What does ssh-add -l do?</summary>Lists fingerprints of identities loaded in the agent.</details>
<details><summary>8. What is agent forwarding?</summary>It exposes the local agent through an SSH session so the remote host can authenticate onward without copying the private key.</details>
<details><summary>9. What is ProxyJump?</summary>It routes an SSH connection through an intermediate jump/bastion host.</details>
<details><summary>10. ProxyJump vs agent forwarding?</summary>ProxyJump changes the network path; agent forwarding changes where the SSH authentication agent is available.</details>
<details><summary>11. What does -L do?</summary>Local port forwarding from the client through the SSH server to a destination reachable by the server.</details>
<details><summary>12. What does -R do?</summary>Remote port forwarding from a remote listening port to a destination on the client side.</details>
<details><summary>13. What does -D do?</summary>Creates a dynamic SOCKS proxy.</details>
<details><summary>14. Timeout vs Permission denied (publickey)?</summary>Timeout usually indicates network/service reachability; publickey denial usually indicates authentication or SSH configuration.</details>
<details><summary>15. How do you fix Too many authentication failures?</summary>Inspect the agent and use `IdentitiesOnly=yes` with the intended private key.</details>
<details><summary>16. What is known_hosts?</summary>It stores host-key information used to detect unexpected server identity changes.</details>
<details><summary>17. How do you troubleshoot a broken SSH tunnel?</summary>Verify SSH, local port binding, destination reachability from the SSH server, destination listener and firewall/security-group rules.</details>
<details><summary>18. How do AWS Security Groups affect SSH?</summary>They control network reachability to TCP/22; they do not replace Linux SSH authentication.</details>
<details><summary>19. Why can IAM access exist while SSH fails?</summary>IAM controls AWS APIs, while SSH requires network reachability, sshd, a valid Linux user/key and host configuration.</details>
<details><summary>20. Give your production SSH troubleshooting sequence.</summary>Reachability → port 22 → security/network layers → sshd listening → username → key → permissions → authorized_keys → sshd configuration → agent → logs.</details>

---

# 🏁 Production Takeaways

```text
Private key = secret
Public key  = server-side authentication material

Timeout              → network/reachability
Connection refused   → listener/service
Permission denied     → authentication/configuration
Host-key changed      → trust/identity
Tunnel failure        → forwarding/destination

Reach → Listen → Authenticate → Authorize → Session → Application
```

The most important DevOps skill is not memorizing SSH flags—it is identifying **which layer failed before changing configuration**.
