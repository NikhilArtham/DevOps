# DAY 3 - Permissions, Ownership, ACLs and sudo Troubleshooting

## 1. Access-control mental model

Linux access troubleshooting should follow:

**Identity → Path traversal → Ownership → Mode bits → ACLs → sudo policy → Application restrictions**

This prevents blind `chmod 777` or recursive permission changes.

## 2. Permissions

Check permissions:

```bash
ls -l file.txt
stat file.txt
```

For regular files:

| Permission | Meaning |
|---|---|
| `r` | Read contents |
| `w` | Modify contents |
| `x` | Execute |

For directories:

| Permission | Meaning |
|---|---|
| `r` | List entries |
| `w` | Create/delete/rename entries |
| `x` | Traverse/access entries |

Numeric values:

```text
r = 4
w = 2
x = 1
```

Examples:

```bash
chmod 755 script.sh
chmod 644 config.txt
chmod 700 private-key.pem
```

## 3. chmod command anatomy

```bash
chmod u+x deploy.sh
chmod g-w config.txt
chmod o-r secret.txt
chmod ug+rw shared.txt
chmod a+r file.txt
chmod u=rwx,g=rx,o= file.txt
chmod -R 750 /opt/app
```

Meaning:

- `u` = user/owner
- `g` = group
- `o` = others
- `a` = all
- `+` = add
- `-` = remove
- `=` = set exactly
- `r` = read
- `w` = write
- `x` = execute/traverse
- `-R` = recursive

Prefer targeted changes over unnecessary recursive changes.

## 4. Ownership

```bash
ls -l file.txt
chown appuser file.txt
chown appuser:appgroup file.txt
chown :appgroup file.txt
chgrp appgroup file.txt
chown -R appuser:appgroup /opt/myapp
```

Command anatomy:

- `chown` = change owner
- `user` = new owner
- `user:group` = set owner and group
- `:group` = change group only
- `-R` = recursive

Avoid blindly running `chown -R` or `chmod -R 777` on system/application trees.

## 5. Path traversal and namei

A file can have correct permissions while access still fails because the user cannot traverse a parent directory.

For:

```text
/opt/app/config/prod.env
```

inspect every path component:

```bash
namei -l /opt/app/config/prod.env
```

Directory `x` permission is especially important for traversal.

## 6. ACLs

ACLs provide more granular permissions for specific users/groups beyond owner/group/other.

Inspect:

```bash
getfacl file.txt
```

Modify:

```bash
setfacl -m u:alice:r file.txt
setfacl -m g:developers:rw file.txt
```

Remove a specific ACL:

```bash
setfacl -x u:alice file.txt
```

Remove all extended ACL entries:

```bash
setfacl -b file.txt
```

Default ACL on a directory:

```bash
setfacl -m d:g:developers:rwx /shared
```

Important flags:

- `-m` = modify/add ACL entry
- `-x` = remove the specified ACL entry
- `-b` = remove all extended ACL entries
- `-d` = work with a directory default ACL inherited by new entries
- `-R` = recursive
- `u` = user ACL entry
- `g` = group ACL entry

Example anatomy:

```bash
setfacl -m u:alice:r file.txt
```

means: modify the ACL and give user `alice` read permission on `file.txt`.

## 7. ACL mask

The ACL mask limits effective permissions for named users, named groups and the owning group.

Example:

```text
user:alice:rwx
mask::r-x
```

Alice's effective permissions cannot exceed the mask. Check `effective:` values in `getfacl` output.

## 8. Special permissions

### setuid

Executable runs with the effective UID of the file owner.

```text
-rwsr-xr-x
```

### setgid

On directories, new files/directories inherit the directory's group:

```bash
chmod g+s /shared
```

### Sticky bit

Common on shared directories such as `/tmp`:

```bash
chmod +t /shared
```

It restricts deletion/renaming of files owned by other users in the directory.

## 9. sudo

`s‍udo` allows an authorized user to run commands with another user's privileges, commonly root.

```bash
sudo systemctl restart nginx
sudo -u appuser whoami
sudo -l
```

Important meanings:

- `sudo` = privilege escalation for an authorized command
- `-u user` = run as specified user
- `-l` = list current sudo privileges

## 10. sudoers

Main configuration:

```text
/etc/sudoers
/etc/sudoers.d/
```

Edit safely with:

```bash
sudo visudo
sudo visudo -f /etc/sudoers.d/devops
```

Validate without editing:

```bash
sudo visudo -c
```

Avoid unnecessarily broad rules such as unrestricted `ALL` access.

## 11. Permission denied troubleshooting

Start with identity:

```bash
whoami
id
groups
```

Check sudo authorization:

```bash
sudo -l
sudo visudo -c
```

Check command resolution:

```bash
command -v systemctl
```

Check the target and every parent directory:

```bash
ls -l /path/to/file
namei -l /path/to/file
getfacl /path/to/file
```

Check logs where applicable:

```bash
journalctl | grep sudo
grep sudo /var/log/auth.log
grep sudo /var/log/secure
```

Log locations vary by distribution.

## 12. Production failure example

An application runs as `appuser` and fails with:

```text
Permission denied: /opt/myapp/config/application.yml
```

Investigate:

```bash
ps -ef | grep myapp
id appuser
ls -l /opt/myapp/config/application.yml
namei -l /opt/myapp/config/application.yml
getfacl /opt/myapp/config/application.yml
```

A common root cause is missing traverse permission on `/opt/myapp/config` even though the file itself looks correct.

Fix the intended access model rather than using `777`, then verify as the application user:

```bash
sudo -u appuser test -r /opt/myapp/config/application.yml
```

## 13. DevOps/AWS relevance

Linux permissions affect EC2 services, deployment users, SSH keys, application directories, Jenkins agents and systemd services. AWS IAM controls AWS API authorization separately; a deployment can be blocked by either AWS authorization or Linux authorization, so identify the failing layer before changing access.