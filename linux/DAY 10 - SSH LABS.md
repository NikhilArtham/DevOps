# DAY 10 — SSH Practical Labs

> Use a disposable Linux VM/EC2 environment. Never experiment with SSH configuration or keys on production access without an approved recovery path.

## Lab 1 — Create and inspect an SSH key

```bash
mkdir -p ~/linux-labs/ssh
chmod 700 ~/linux-labs/ssh
ssh-keygen -t ed25519 -f ~/linux-labs/ssh/lab_key -C "ssh-lab"
ls -l ~/linux-labs/ssh
chmod 600 ~/linux-labs/ssh/lab_key
```

Tasks:
- Identify private vs public key.
- Explain why the private key is secret.
- Explain `-t`, `-f` and `-C`.
- Explain why `600` is used for the private key.

## Lab 2 — ssh-agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/linux-labs/ssh/lab_key
ssh-add -l
ssh-add -L
ssh-add -D
ssh-add -l
```

Tasks:
- Explain every command and flag.
- Explain why an agent is useful.
- Explain the security risk of agent forwarding.

## Lab 3 — SSH verbose troubleshooting

```bash
ssh -vvv user@<test-host>
```

Record:
- destination
- username
- keys offered
- authentication method
- exact final error

Classify the failure as network, service, authentication or configuration.

## Lab 4 — ProxyJump / bastion

On a disposable environment with a bastion and private host:

```bash
ssh -J user@bastion user@private-host
```

Then create a client alias:

```sshconfig
Host private-host
    HostName <private-ip>
    User <user>
    ProxyJump <user>@<bastion>
```

Test:

```bash
ssh private-host
```

Explain the difference between ProxyJump and agent forwarding.

## Lab 5 — Local SSH tunnel

If the SSH server can reach an internal HTTP service:

```bash
ssh -L 8080:<internal-host>:80 user@<test-host>
```

From the client:

```bash
curl http://localhost:8080
```

Then verify reachability from the SSH server:

```bash
nc -vz <internal-host> 80
```

Explain why the destination must be reachable from the SSH server side.

## Lab 6 — Failure injection

Introduce one safe failure at a time:

1. Wrong username.
2. Wrong private key.
3. Private key permission `644`.
4. Unreachable TCP/22.
5. Destination service not listening for a tunnel.
6. Agent loaded with multiple unrelated keys.

Useful commands:

```bash
ssh -vvv user@host
ssh-add -l
ssh -o IdentitiesOnly=yes -i <key> user@host
nc -vz host 22
ss -lntp
```

For every failure document:

```text
Observed error:
Layer that failed:
Evidence:
Root cause:
Fix:
Prevention:
```

## Lab 7 — Production-style incident

Scenario:

```text
Laptop → Bastion → Private EC2

Bastion SSH works.
Private EC2 SSH times out.
```

Investigate in this order:

```bash
nc -vz <private-ip> 22
```

Then, if you have an alternate access path:

```bash
sudo ss -lntp | grep ':22'
sudo systemctl status sshd
sudo journalctl -u sshd -n 100 --no-pager
```

Inspect AWS network controls conceptually:

```text
Bastion SG: TCP 22 from trusted admin source
Private SG: TCP 22 from Bastion SG
Route table: private subnet route is valid
NACL/firewall: traffic is permitted
```

Success criteria: prove whether the failure is network reachability, SSH listener, authentication, authorization or configuration before making a change.

## Interview drill

Answer these without notes:

1. Why does `Permission denied (publickey)` point to a different layer than a timeout?
2. What is the difference between `-J` and `-A`?
3. What does `-L` do?
4. Why can an SSH tunnel still fail even when SSH itself works?
5. Why can IAM permission exist while SSH fails?
6. What does `IdentitiesOnly=yes` solve?
7. Why should you never blindly remove a `known_hosts` warning?
8. What is your production SSH troubleshooting sequence?
