# DAY 10 — SSH Interview Questions

<details><summary>1. What is SSH?</summary>SSH is an encrypted protocol for secure remote login and command execution.</details>
<details><summary>2. Public key vs private key?</summary>The public key is installed on the server; the private key remains secret on the client.</details>
<details><summary>3. Where is an authorized public key normally stored?</summary>In the remote user's `~/.ssh/authorized_keys`.</details>
<details><summary>4. Why use chmod 600 for a private key?</summary>It prevents group and other users from reading the private key.</details>
<details><summary>5. What does ssh -vvv do?</summary>It provides detailed client-side debugging for connection and authentication.</details>
<details><summary>6. What is ssh-agent?</summary>A process that holds SSH identities in memory and performs key authentication operations for clients.</details>
<details><summary>7. What does ssh-add -l do?</summary>It lists fingerprints of keys loaded in the SSH agent.</details>
<details><summary>8. What is agent forwarding?</summary>It makes the local SSH agent available through an SSH session so a remote host can authenticate onward without copying the private key.</details>
<details><summary>9. What is ProxyJump?</summary>It routes an SSH connection through an intermediate jump/bastion host.</details>
<details><summary>10. ProxyJump vs agent forwarding?</summary>ProxyJump changes the network path; agent forwarding changes where the authentication agent is available.</details>
<details><summary>11. What does ssh -L do?</summary>It creates local port forwarding through the SSH server to a destination reachable from the server side.</details>
<details><summary>12. What does ssh -R do?</summary>It creates remote port forwarding from a remote listening port to a client-side destination.</details>
<details><summary>13. What does ssh -D do?</summary>It creates a dynamic SOCKS proxy through the SSH connection.</details>
<details><summary>14. Timeout vs Permission denied (publickey)?</summary>A timeout normally indicates network/service reachability; publickey denial normally indicates authentication or SSH configuration.</details>
<details><summary>15. How do you fix Too many authentication failures?</summary>Inspect the agent and use `ssh -o IdentitiesOnly=yes -i <key> user@host` to limit identity selection.</details>
<details><summary>16. What is known_hosts?</summary>It stores host-key information used to detect unexpected server identity changes.</details>
<details><summary>17. How do you troubleshoot a broken SSH tunnel?</summary>Verify SSH, local port binding, destination reachability from the SSH server, destination listener and firewall/security-group rules.</details>
<details><summary>18. How do Security Groups affect SSH?</summary>They control network reachability to TCP/22; SSH keys and Linux/sshd configuration handle authentication and host access.</details>
<details><summary>19. Why can IAM access exist while SSH fails?</summary>IAM controls AWS APIs, while SSH requires network reachability, a listening SSH service, a valid user/key and valid host configuration.</details>
<details><summary>20. Give your production SSH troubleshooting sequence.</summary>Reachability → TCP/22 → security/network layers → sshd listening → username → key → permissions → authorized_keys → sshd configuration → agent → logs.</details>
