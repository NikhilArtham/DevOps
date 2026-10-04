# DAY 3 - Permissions, Ownership, ACLs and sudo Troubleshooting

## 1. What this topic covers

Linux access control is built around four closely related areas:

1. **Permissions** — read, write, execute bits for owner/group/others.
2. **Ownership** — which user and group own a file or directory.
3. **ACLs (Access Control Lists)** — extra per-user/per-group permissions beyond the basic mode bits.
4. **sudo** — controlled privilege escalation and the configuration that determines who may run what as another user.

A useful troubleshooting model is:

**Identity → Path traversal → Ownership → Mode bits → ACLs → sudo policy → Application-specific restrictions**

---

## 2. Linux permission model

Check permissions with:

```bash
ls -l file.txt
```

Example:

```text
-rwxr-x--- 1 nikhil devops 1200 Oct  3 10:00 deploy.sh
```

Breakdown:

- `-` = regular file
- `rwx` = owner permissions
- `r-x` = group permissions
- `---` = others permissions
- `nikhil` = owner
- `devops` = owning group

### Permission meanings

For a regular file:

| Permission | Meaning |
|---|---|
| `r` | Read file contents |
| `w` | Modify file contents |
| `x` | Execute the file |

For directories:

| Permission | Meaning |
|---|---|
| `r` | List directory entries |
| `w` | Create/delete/rename entries |
| `x` | Traverse/access entries |

**Important:** Directory `x` is especially important. A user may have read permission on a directory but still be unable to access files inside it without execute/traverse permission.

---

## 3. Numeric permissions

The standard values are:

| Permission | Value |
|---|---:|
| `r` | 4 |
| `w` | 2 |
| `x` | 1 |

Examples:

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
0 = ---
```

Therefore:

```bash
chmod 755 script.sh
chmod 644 config.txt
chmod 700 private-key.pem
```

Typical meanings:

- `755` → owner can modify/execute; everyone else can read/execute.
- `644` → owner can read/write; others can read.
- `700` → only owner has access.

---

## 4. chmod

Change permissions:

```bash
chmod 755 script.sh
chmod u+x script.sh
chmod g-w config.txt
chmod o-r secret.txt
chmod -R 750 /opt/app
```

Prefer targeted changes over unnecessary recursive permission changes.

### Symbolic notation

- `u` = user/owner
- `g` = group
- `o` = others
- `a` = all

Examples:

```bash
chmod u+x deploy.sh
chmod g+r config.txt
chmod o-r secret.txt
chmod ug+rw shared.txt
```

---

## 5. Ownership

View ownership:

```bash
ls -l file.txt
stat file.txt
```

Change owner:

```bash
sudo chown appuser file.txt
```

Change owner and group:

```bash
sudo chown appuser:appgroup file.txt
```

Change group:

```bash
sudo chgrp appgroup file.txt
```

Recursive ownership change:

```bash
sudo chown -R appuser:appgroup /opt/myapp
```

### Production warning

Do not blindly run:

```bash
chown -R ...
chmod -R 777 ...
```

against system directories or application trees. Recursive changes can break services and create security problems.

---

## 6. Effective access is also about the path

Suppose:

```text
/opt/app/config/prod.env
```

Even if `prod.env` has correct permissions, access can fail because the user cannot traverse:

```text
/opt
/opt/app
/opt/app/config
```

Use:

```bash
namei -l /opt/app/config/prod.env
```

This is one of the most useful commands when debugging:

```text
Permission denied
```

---

## 7. ACLs

ACLs allow permissions for specific users or groups in addition to the traditional owner/group/other model.

Check ACLs:

```bash
getfacl file.txt
```

Add a user ACL:

```bash
setfacl -m u:alice:r file.txt
```

Give a group read/write:

```bash
setfacl -m g:developers:rw file.txt
```

Remove a specific ACL:

```bash
setfacl -x u:alice file.txt
```

Remove extended ACL entries:

```bash
setfacl -b file.txt
```

### Default ACLs

Default ACLs on directories can control permissions inherited by newly created files/directories.

```bash
setfacl -m d:g:developers:rwx /shared
```

Inspect:

```bash
getfacl /shared
```

---

## 8. ACL mask — common interview trap

Example:

```text
user::rwx
user:alice:rwx
group::r-x
mask::r-x
other::---
```

The **ACL mask** limits the effective permissions of named users, named groups, and the owning group.

So even though:

```text
user:alice:rwx
```

is present, the effective access can be restricted by:

```text
mask::r-x
```

Check the `effective:` value in `getfacl` output.

---

## 9. Special permissions

### setuid

A setuid executable runs with the effective UID of the file owner.

Check:

```bash
ls -l /path/to/file
```

Example permission representation:

```text
-rwsr-xr-x
```

### setgid

On executables, setgid can provide the effective group of the file.

On directories, setgid causes newly created files/directories to inherit the directory's group.

```bash
chmod g+s /shared
```

### Sticky bit

Commonly used on shared directories such as `/tmp`.

```bash
chmod +t /shared
```

It prevents users from deleting/renaming files they do not own in a sticky directory, subject to privileged-user rules.

---

## 10. sudo basics

`sudo` allows an authorized user to execute a command with elevated privileges.

Examples:

```bash
sudo systemctl restart nginx
sudo -u appuser whoami
sudo -l
```

Check the current user's sudo privileges:

```bash
sudo -l
```

This is a key troubleshooting command.

---

## 11. sudoers configuration

Main configuration:

```text
/etc/sudoers
```

Additional configuration is commonly stored under:

```text
/etc/sudoers.d/
```

Always validate sudoers changes with:

```bash
sudo visudo
```

For a specific file:

```bash
sudo visudo -f /etc/sudoers.d/devops
```

Do not casually edit `/etc/sudoers` with a normal text editor. A syntax error can break sudo access.

Example policy:

```text
%devops ALL=(ALL) /usr/bin/systemctl restart nginx
```

This grants members of the `devops` group permission to run the specified command through sudo.

Avoid broad rules such as:

```text
%devops ALL=(ALL) NOPASSWD: ALL
```

unless there is a deliberate, controlled requirement.

---

## 12. sudo troubleshooting

When:

```bash
sudo command
```

fails, check:

### Step 1 — Identity

```bash
whoami
id
groups
```

### Step 2 — Sudo policy

```bash
sudo -l
```

### Step 3 — Sudoers syntax

```bash
sudo visudo -c
```

### Step 4 — Command path

```bash
which systemctl
command -v systemctl
```

A sudoers rule may allow one exact path while the user is attempting another command/path.

### Step 5 — File permissions

```bash
ls -l /path/to/file
namei -l /path/to/file
getfacl /path/to/file
```

### Step 6 — Logs

Depending on the distribution:

```bash
journalctl -u sudo
journalctl | grep sudo
grep sudo /var/log/auth.log
grep sudo /var/log/secure
```

Log location differs by Linux distribution and configuration.

---

## 13. Realistic production failure scenario

### Scenario

A deployment service runs as:

```text
appuser
```

The deployment suddenly fails:

```text
Permission denied: /opt/myapp/config/application.yml
```

### Investigation

First identify the service user:

```bash
ps -ef | grep myapp
id appuser
```

Check the file:

```bash
ls -l /opt/myapp/config/application.yml
```

Then inspect every directory in the path:

```bash
namei -l /opt/myapp/config/application.yml
```

Check ACLs:

```bash
getfacl /opt/myapp/config/application.yml
```

### Possible root cause

A deployment engineer changed:

```text
/opt/myapp/config
```

from:

```text
drwxr-x---
```

to:

```text
drwx------
```

The file itself still looks correct, but `appuser` can no longer traverse the parent directory.

### Safe fix

Restore the required group/traverse permission instead of using `777`:

```bash
sudo chmod 750 /opt/myapp/config
sudo chgrp appgroup /opt/myapp/config
```

Then verify as the application user:

```bash
sudo -u appuser test -r /opt/myapp/config/application.yml && echo "readable"
```

Finally restart/retry the deployment and verify logs.

### Production lesson

Do not start with:

```bash
chmod 777
```

Start by identifying **which identity needs which access to which path**.

---

## 14. Linux permissions vs AWS permissions

These are different layers.

### Linux permissions

Control access inside the operating system:

```text
user → file/directory → Linux permissions/ACLs
```

Typical commands:

```bash
chmod
chown
chgrp
getfacl
setfacl
sudo
```

### AWS IAM

Controls access to AWS APIs/resources:

```text
IAM principal → IAM policy → AWS API/resource
```

For example, an EC2 instance may have Linux permission to read a local file but still receive:

```text
AccessDenied
```

when the application attempts to read an S3 object.

That is an **AWS IAM/resource-policy issue**, not a Linux file-permission issue.

Conversely, an EC2 instance may have an IAM role allowing S3 access but the application can still fail before making the AWS API call because the local credential/configuration file is not readable.

---

## 15. AWS troubleshooting connection

For an EC2 application that cannot read S3, separate the problem into layers:

1. **Linux identity**
   ```bash
   id
   ps -ef
   ```

2. **Local file permissions**
   ```bash
   ls -l ~/.aws/
   getfacl ~/.aws/config
   ```

3. **Credential source / IAM role**
   Check the EC2 instance profile and application credential configuration.

4. **IAM policy**
   Verify the required action such as `s3:GetObject`.

5. **S3 bucket/object policy**
   Check resource-based restrictions.

6. **Network path**
   If the application cannot reach the required AWS endpoint, investigate DNS, routing, security groups, NACLs, proxy/VPC endpoint configuration, and related connectivity.

This layered approach prevents mixing Linux `Permission denied` with AWS `AccessDenied`.

---

## 16. Command cheat sheet

| Goal | Command |
|---|---|
| View permissions | `ls -l file` |
| Detailed metadata | `stat file` |
| Change permissions | `chmod 640 file` |
| Change owner | `chown user file` |
| Change owner/group | `chown user:group file` |
| Change group | `chgrp group file` |
| Show identity | `id` |
| Show path permissions | `namei -l /path/to/file` |
| View ACL | `getfacl file` |
| Add ACL | `setfacl -m u:user:rwx file` |
| Remove ACL | `setfacl -x u:user file` |
| Remove extended ACLs | `setfacl -b file` |
| Check sudo privileges | `sudo -l` |
| Validate sudoers | `sudo visudo -c` |
| Safely edit sudoers | `sudo visudo` |
| Run as another user | `sudo -u user command` |
| Find command path | `command -v command` |
| Check sudo logs | `journalctl | grep sudo` |

---

## 17. Practical lab

### Lab 1 — Basic permissions

```bash
mkdir -p ~/permissions-lab
cd ~/permissions-lab
echo "secret" > secret.txt
chmod 600 secret.txt
ls -l secret.txt
```

Create a test user if your lab environment permits it, then verify that the user cannot read the file.

### Lab 2 — Ownership

```bash
sudo chown root:root secret.txt
ls -l secret.txt
```

Test access with different users.

### Lab 3 — ACL

```bash
sudo setfacl -m u:$(whoami):rw secret.txt
getfacl secret.txt
```

Observe the ACL entry and mask.

### Lab 4 — Path traversal

```bash
mkdir -p ~/permissions-lab/private/data
chmod 700 ~/permissions-lab/private
namei -l ~/permissions-lab/private/data
```

Understand why permissions on parent directories matter.

### Lab 5 — sudo

```bash
sudo -l
sudo -u root whoami
sudo visudo -c
```

Do not modify production sudoers rules during practice.

---

# Interview Questions

<details>
<summary>1. What are Linux file permissions?</summary>

Linux file permissions control read, write, and execute access for the file owner, owning group, and others.
</details>

<details>
<summary>2. What does 755 mean?</summary>

Owner has `rwx`; group has `r-x`; others have `r-x`.
</details>

<details>
<summary>3. What is the difference between file and directory execute permission?</summary>

For a file, execute allows execution. For a directory, execute allows traversal/access to entries inside it.
</details>

<details>
<summary>4. What is the difference between chmod and chown?</summary>

`chmod` changes permissions; `chown` changes ownership.
</details>

<details>
<summary>5. How do you troubleshoot Permission denied?</summary>

Check the user with `id`, inspect file and parent-directory permissions with `ls -l` and `namei -l`, check ACLs with `getfacl`, then investigate sudo or application-specific restrictions.
</details>

<details>
<summary>6. Why can a user read a file but still get Permission denied?</summary>

A parent directory may not grant the user execute/traverse permission.
</details>

<details>
<summary>7. What is an ACL?</summary>

An Access Control List provides additional per-user or per-group permissions beyond the traditional owner/group/other model.
</details>

<details>
<summary>8. How do you check ACLs?</summary>

Use `getfacl file`.
</details>

<details>
<summary>9. How do you grant a specific user access using ACL?</summary>

Use a command such as `setfacl -m u:alice:rw file`.
</details>

<details>
<summary>10. What is the ACL mask?</summary>

The ACL mask limits the effective permissions of named users, named groups, and the owning group in an extended ACL.
</details>

<details>
<summary>11. What is setgid on a directory?</summary>

It causes newly created files and directories under that directory to inherit the directory's group.
</details>

<details>
<summary>12. What is the sticky bit?</summary>

On a shared directory, it restricts file deletion/renaming so users generally can modify only entries they own, even when the directory is writable by multiple users.
</details>

<details>
<summary>13. What is sudo?</summary>

`sudo` provides controlled privilege escalation for authorized users.
</details>

<details>
<summary>14. How do you check what a user can run with sudo?</summary>

Use `sudo -l`.
</details>

<details>
<summary>15. Why should visudo be used?</summary>

`visudo` validates sudoers syntax and helps prevent locking administrators out because of a configuration syntax error.
</details>

<details>
<summary>16. Where are sudo rules commonly configured?</summary>

The main file is `/etc/sudoers`, with additional rules commonly placed in `/etc/sudoers.d/`.
</details>

<details>
<summary>17. What is the danger of chmod 777?</summary>

It grants read/write/execute permissions broadly and can create security and operational risks. It should not be used as a generic fix for Permission denied.
</details>

<details>
<summary>18. How would you troubleshoot an application that cannot read a configuration file?</summary>

Identify the process user, inspect the file and all parent directories, check ACLs, verify ownership, test access as the service user, and inspect application/system logs.
</details>

<details>
<summary>19. What is the difference between Linux Permission denied and AWS AccessDenied?</summary>

Linux Permission denied generally indicates an OS-level access-control failure. AWS AccessDenied generally indicates an AWS authorization failure involving IAM or resource policies. Both layers can affect the same application.
</details>

<details>
<summary>20. An EC2 application has s3:GetObject permission but still cannot read its local credentials/config file. What is wrong?</summary>

The AWS IAM permission may be correct, but Linux ownership, mode bits, ACLs, or path traversal permissions can prevent the application from reading its local configuration. Troubleshoot the OS layer before assuming IAM is the problem.
</details>

---

## Production mental model

When you see:

```text
Permission denied
```

think in this order:

```text
Who am I?
   ↓
Which process/user is actually accessing it?
   ↓
Can I traverse every parent directory?
   ↓
Who owns the file?
   ↓
What do mode bits allow?
   ↓
Are ACLs changing effective permissions?
   ↓
Do I need sudo?
   ↓
Does sudo actually authorize this exact command?
   ↓
Are there additional controls such as SELinux/AppArmor?
```

For AWS-backed applications:

```text
Linux authorization
        ↓
Application credentials
        ↓
IAM identity policy
        ↓
Resource policy
        ↓
Network/connectivity
        ↓
AWS service
```

The key DevOps skill is to identify **which authorization layer is actually failing** instead of changing permissions blindly.
