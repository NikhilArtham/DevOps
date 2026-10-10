# DAY 10 - SSH: Keys, Agent, Tunneling & Failures

> **Goal:** Understand SSH from the beginning and troubleshoot common SSH problems systematically.

## 1. What is SSH?

**SSH (Secure Shell)** is a secure way to connect to another computer over a network.

Example:

```bash
ssh user@server
```

In DevOps, SSH is commonly used to access Linux servers such as EC2 instances.

Think:

```text
Your laptop
    ↓ encrypted SSH connection
Linux server
```

## 2. SSH key authentication

Instead of relying only on passwords, SSH commonly uses a key pair.

There are two keys:

```text
Private key → stays secret on your computer
Public key  → can be placed on the server
```

Typical files:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
~/.ssh/authorized_keys
```

The server checks whether your public key matches the private key you are using.

## 3. Create an SSH key

```bash
ssh-keygen -t ed25519 -C "devops-lab"
```

Meaning:

- `ssh-keygen` = create/manage SSH keys
- `-t` = key type
- `ed25519` = key algorithm
- `-C` = comment

Never commit a private key to GitHub.

## 4. SSH key permissions

Protect private keys:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
```

A private key that is readable by other users may be rejected by SSH.

You may see:

```text
WARNING: UNPROTECTED PRIVATE KEY FILE!
```

Fix the permissions first.

## 5. `authorized_keys`

On the server, a user's allowed public keys are commonly stored in:

```text
~/.ssh/authorized_keys
```

For example:

```bash
cat ~/.ssh/authorized_keys
```

The private key should remain on the client.

## 6. SSH client configuration

Instead of repeatedly typing a long command, use:

```text
~/.ssh/config
```

Example:

```sshconfig
Host my-ec2
    HostName 203.0.113.10
    User ec2-user
    IdentityFile ~/.ssh/my-key.pem
    IdentitiesOnly yes
```

Then simply:

```bash
ssh my-ec2
```

Important options:

- `Host` = shortcut name
- `HostName` = real server address
- `User` = remote username
- `IdentityFile` = private key
- `IdentitiesOnly yes` = restrict identity selection

## 7. Debug SSH with `-vvv`

When SSH fails, use:

```bash
ssh -vvv user@host
```

The extra `v`s increase debugging information.

You are looking for where the failure happens:

```text
Network connection
      ↓
SSH server
      ↓
Authentication
      ↓
Authorization
      ↓
Shell/session
```

## 8. ssh-agent

`ssh-agent` keeps private-key identities available in memory so you do not have to repeatedly provide the key/passphrase.

Start an agent:

```bash
eval "$(ssh-agent -s)"
```

Add a key:

```bash
ssh-add ~/.ssh/id_ed25519
```

List keys:

```bash
ssh-add -l
```

Remove all loaded keys:

```bash
ssh-add -D
```

## 9. Too many authentication failures

Suppose your agent contains many keys.

SSH may try several of them and the server can reject the connection before the correct key is attempted.

Use:

```bash
ssh -o IdentitiesOnly=yes -i ~/.ssh/my-key.pem ec2-user@host
```

`-o` sets an SSH option.

`IdentitiesOnly=yes` tells SSH to use the explicitly selected identities instead of trying a large collection of agent keys.

## 10. Bastion / jump host

A private EC2 instance may not be directly reachable from your laptop.

Architecture:

```text
Laptop
  ↓
Bastion / jump server
  ↓
Private EC2
```

Use `ProxyJump`:

```bash
ssh -J ec2-user@bastion ec2-user@private-server
```

`-J` = connect through a jump host.

This is primarily about the **network path**.

## 11. Agent forwarding

Agent forwarding is different from ProxyJump.

```bash
ssh -A ec2-user@bastion
```

It allows the remote session to use your forwarded SSH agent.

Important security point: a compromised intermediate server may be able to use the forwarded agent during the session. Enable it only when needed.

Remember:

```text
ProxyJump       → network path
Agent forwarding → authentication agent
```

## 12. SSH tunneling

SSH can securely forward network traffic.

Three common options:

```text
-L = local forwarding
-R = remote forwarding
-D = dynamic SOCKS forwarding
```

### Local forwarding

```bash
ssh -L 8080:10.0.2.50:80 user@bastion
```

Now:

```text
Laptop localhost:8080
       ↓
SSH connection
       ↓
Bastion
       ↓
10.0.2.50:80
```

`-L local_port:destination_host:destination_port`.

The important point: the SSH server side must be able to reach the destination.

### Remote forwarding

```bash
ssh -R 9000:localhost:3000 user@server
```

This exposes a client-side destination through a remote listening port. Use carefully because it can expose services.

### Dynamic forwarding

```bash
ssh -D 1080 user@bastion
```

Creates a SOCKS proxy at local port `1080`.

## 13. SSH server configuration

Server configuration is commonly:

```text
/etc/ssh/sshd_config
```

Before applying a configuration change:

```bash
sudo sshd -t
```

`-t` tests configuration syntax.

Do not blindly restart SSH after changing configuration on a remote production server. Keep a working session available where possible.

## 14. Failure: connection timeout

Example:

```text
ssh: connect to host server port 22: Connection timed out
```

This usually means the connection is failing before SSH authentication.

Think:

```text
DNS/IP
 ↓
Route
 ↓
Security Group
 ↓
NACL
 ↓
Firewall
 ↓
sshd listening
```

Test connectivity where appropriate:

```bash
nc -vz host 22
```

On the server, if another access path exists:

```bash
sudo ss -lntp | grep ':22'
sudo systemctl status sshd
```

Do **not** start by changing `authorized_keys` when the error is a timeout.

## 15. Failure: Permission denied (publickey)

Example:

```text
Permission denied (publickey).
```

Now authentication is the focus.

Check:

```bash
ssh -vvv -i ~/.ssh/my-key.pem ec2-user@host
ls -l ~/.ssh/my-key.pem
```

Potential causes:

- wrong private key
- wrong username
- public key missing from `authorized_keys`
- incorrect permissions/ownership
- wrong key selected by agent
- SSH server authentication configuration

## 16. Failure: host key changed

You may see:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

SSH stores known host keys in:

```text
~/.ssh/known_hosts
```

Do not blindly delete the warning. First determine whether the server was rebuilt or the key legitimately changed.

If the change is verified:

```bash
ssh-keygen -R <hostname>
```

Then verify the new fingerprint through a trusted source.

## 17. Failure: tunnel does not work

For:

```bash
ssh -L 8080:10.0.2.50:80 user@bastion
```

Check:

```text
1. Is SSH connected?
2. Is localhost:8080 listening?
3. Can the bastion reach 10.0.2.50:80?
4. Is the destination application listening?
5. Do firewalls/Security Groups allow it?
```

From the bastion:

```bash
nc -vz 10.0.2.50 80
```

A tunnel does not magically bypass destination networking.

## 18. AWS SSH architecture

Typical private EC2 design:

```text
Laptop
  ↓ TCP 22
Bastion
  ↓ TCP 22
Private EC2
```

Security Groups might allow:

```text
Bastion SG:
TCP 22 ← trusted admin IP

Private EC2 SG:
TCP 22 ← Bastion SG
```

Avoid opening SSH to `0.0.0.0/0` when a narrower source is possible.

Also remember:

- IAM controls AWS API permissions.
- SSH controls Linux access.
- Security Groups control network reachability.

Having IAM permission does not automatically give you Linux SSH access.

AWS Systems Manager Session Manager can also provide a managed alternative to direct inbound SSH in suitable environments.

## 19. Production troubleshooting model

Use this order:

```text
REACH
  ↓
LISTEN
  ↓
AUTHENTICATE
  ↓
AUTHORIZE
  ↓
SESSION
  ↓
APPLICATION
```

Examples:

```text
Timeout
→ investigate network/reachability

Connection refused
→ investigate service/listener/firewall

Permission denied (publickey)
→ investigate authentication

Host key changed
→ investigate trust/identity

Tunnel failure
→ investigate forwarding + destination reachability
```

## 20. Commands to remember

```bash
ssh user@host
ssh -i key user@host
ssh -vvv user@host
ssh -J user@bastion user@target
ssh -A user@bastion
ssh -L 8080:host:80 user@server
ssh -R 9000:localhost:3000 user@server
ssh -D 1080 user@server
ssh-keygen -t ed25519
ssh-keygen -R host
ssh-add key
ssh-add -l
ssh-add -D
ss -lntp
nc -vz host 22
sudo sshd -t
```

### Beginner takeaway

Do not memorize SSH troubleshooting as random commands.

First identify **which layer failed**:

```text
Can I reach it?
   ↓
Is SSH listening?
   ↓
Did authentication work?
   ↓
Is the user authorized?
   ↓
Does the session work?
   ↓
Can the target application be reached?
```

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.