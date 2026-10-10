# DAY 10 — SSH: Keys, Agent, Tunneling & Common Failures

> Linux / AWS — DevOps Job-Switch Notes

## 1. SSH basics

SSH provides encrypted remote access and command execution.

```bash
ssh user@server
ssh -i ~/.ssh/my-key.pem ec2-user@<public-ip>
```

- `ssh` = SSH client
- `-i` = identity/private-key file
- `user@server` = remote user and destination

Common DevOps uses: EC2 access, bastions, remote commands, file transfer and secure tunnels.

## 2. SSH keys

```text
Private key → stays secret on client
Public key  → installed/authorized on server
```

Typical files:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
~/.ssh/authorized_keys
```

Generate:

```bash
ssh-keygen -t ed25519 -C "devops-lab"
```

- `-t` = key type
- `-C` = comment

Protect keys:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 600 ~/.ssh/authorized_keys
```

`700` gives the owner `rwx`; `600` gives the owner `rw` and no group/other access.

Never commit or share a private key.

## 3. SSH client/server configuration

Client:

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

Important options:

- `Host` = alias/pattern
- `HostName` = actual destination
- `User` = remote username
- `IdentityFile` = private key
- `IdentitiesOnly yes` = restrict identity selection

Server configuration is commonly:

```text
/etc/ssh/sshd_config
```

Validate before changing/reloading:

```bash
sudo sshd -t
```

Do not blindly restart SSH after configuration changes on a remote production host; a mistake can lock you out.

## 4. SSH debugging

```bash
ssh -v user@host
ssh -vv user@host
ssh -vvv user@host
```

Look for connection establishment, keys offered, authentication methods and the final failure.

The goal is to identify the failing layer rather than change configuration blindly.

## 5. ssh-agent

`ssh-agent` holds private-key material in memory so SSH can use keys without repeatedly requesting their passphrases.

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-add -l
ssh-add -L
ssh-add -D
```

- `ssh-agent -s` = shell-compatible environment output
- `ssh-add -l` = list fingerprints
- `ssh-add -L` = list public keys
- `ssh-add -D` = remove all identities

Too many loaded keys can cause authentication-attempt failures. Use:

```bash
ssh -o IdentitiesOnly=yes -i ~/.ssh/my-key.pem ec2-user@host
```

## 6. Agent forwarding

Architecture:

```text
Laptop → Bastion → Private EC2
```

Enable forwarding:

```bash
ssh -A ec2-user@bastion
```

The private key remains on the laptop, but the bastion can use the forwarded agent during the session. A compromised intermediate host may be able to use the forwarded agent, so enable it only when required.

## 7. ProxyJump / bastion

```bash
ssh -J ec2-user@bastion ec2-user@private-server
```

`-J` = ProxyJump through an intermediate SSH host.

Configuration:

```sshconfig
Host private-server
    HostName 10.0.2.50
    User ec2-user
    ProxyJump ec2-user@bastion
```

**ProxyJump** provides the network path through the bastion. **Agent forwarding** forwards the authentication agent. They solve different problems.

## 8. SSH tunneling

```text
-L → local forwarding
-R → remote forwarding
-D → dynamic SOCKS forwarding
```

### Local forwarding

```bash
ssh -L 8080:10.0.2.50:80 ec2-user@bastion
curl http://localhost:8080
```

Traffic:

```text
Laptop localhost:8080 → SSH → Bastion → 10.0.2.50:80
```

Syntax:

```text
-L local_port:destination_host:destination_port
```

The SSH server side must be able to reach the destination.

### Remote forwarding

```bash
ssh -R 9000:localhost:3000 user@server
```

This exposes a client-side destination through a remote listening port. Use carefully because it can expose services.

### Dynamic forwarding

```bash
ssh -D 1080 ec2-user@bastion
```

Creates a SOCKS proxy at `localhost:1080`.

## 9. AWS SSH architecture

Typical private-EC2 design:

```text
Laptop
  ↓ TCP 22
Bastion / public EC2
  ↓ TCP 22
Private EC2
```

Security Group design:

```text
Bastion SG:
TCP 22 ← trusted admin source

Private EC2 SG:
TCP 22 ← Bastion SG
```

Avoid broad SSH exposure such as `0.0.0.0/0` when a narrower source is possible.

Also consider route tables, NACLs, host firewall, subnet routing and DNS.

### SSH vs IAM

```text
IAM → AWS API authorization
SSH → network + sshd + Linux authentication
```

IAM permission does not automatically provide OS-level SSH access. AWS Systems Manager Session Manager can be a managed alternative to direct inbound SSH in suitable architectures.

## 10. Connection timeout

For:

```text
ssh: connect to host <host> port 22: Connection timed out
```

Think:

```text
DNS/IP → route → Security Group → NACL → firewall → sshd listening
```

Test from an appropriate host:

```bash
nc -vz <host> 22
sudo ss -lntp | grep ':22'
sudo systemctl status sshd
```

Do not start by changing `authorized_keys` when the error is a timeout. Authentication may not have been reached.

## 11. Permission denied (publickey)

```text
Permission denied (publickey).
```

Client checks:

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

Common causes:

- Wrong private key
- Wrong username
- Public key missing from `authorized_keys`
- Incorrect ownership/permissions
- sshd authentication configuration
- Wrong key being offered by agent

## 12. Too many authentication failures

Often caused by an agent containing many keys:

```bash
ssh-add -l
ssh -o IdentitiesOnly=yes -i ~/.ssh/my-key.pem ec2-user@host
```

## 13. Host key changed

For:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

SSH stores known host keys in:

```text
~/.ssh/known_hosts
```

Do not blindly delete the warning. Verify whether the server was rebuilt, the IP/DNS changed or the key was rotated. A malicious interception is also possible.

After verifying the change is legitimate:

```bash
ssh-keygen -R <hostname-or-ip>
```

Then validate the new fingerprint through a trusted source.

## 14. Tunnel failure

For:

```bash
ssh -L 8080:10.0.2.50:80 ec2-user@bastion
```

Check:

1. SSH connection itself works.
2. Local port is bound.
3. Bastion can reach `10.0.2.50:80`.
4. Destination service is listening.
5. Security Groups/firewalls permit the traffic.

From the bastion:

```bash
nc -vz 10.0.2.50 80
```

A tunnel does not bypass destination-side networking.

## 15. Production failure scenario

A DevOps engineer can reach a bastion but cannot SSH to a private EC2 instance:

```bash
ssh -J ec2-user@bastion ec2-user@10.0.2.50
```

The error is:

```text
connect to host 10.0.2.50 port 22: Connection timed out
```

Changing `authorized_keys` is the wrong first move. Test reachability from the bastion:

```bash
nc -vz 10.0.2.50 22
```

Suppose the private instance Security Group allows TCP/22 only from an old bastion Security Group. The new bastion is not an allowed source.

Fix the Security Group source to the intended bastion Security Group, then retest:

```bash
nc -vz 10.0.2.50 22
ssh -J ec2-user@bastion ec2-user@10.0.2.50
```

## 16. SSH troubleshooting mental model

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

Confirm expected shell/command execution.

### APPLICATION

For tunnels, verify the destination application from the SSH server side.

## 17. Command cheat sheet

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
| `ssh-keygen -y -f key` | Derive public key |
| `ssh-add key` | Add key to agent |
| `ssh-add -l` | List agent keys |
| `ssh-add -D` | Remove all agent keys |
| `chmod 600 private-key` | Protect private key |
| `ss -lntp` | Listening TCP sockets |
| `systemctl status sshd` | SSH server status |
| `journalctl -u sshd` | SSH server logs |

## 18. AWS/DevOps relevance

SSH troubleshooting is a layered problem. Classify the error before changing configuration:

```text
Timeout / refused → network/service layer
Permission denied → authentication/configuration
Host-key warning → trust/identity
Tunnel failure → forwarding/destination reachability
```

The most important skill is identifying the failed layer before taking action.