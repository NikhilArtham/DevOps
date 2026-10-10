# DAY 3 - Linux Permissions, Ownership, ACLs & sudo

> **Goal:** Understand who can access a file, what they can do, and why Linux sometimes says `Permission denied`.

## 1. Why permissions exist

Linux is a multi-user operating system. Many users and services can exist on the same server.

Permissions prevent one user or application from changing files that belong to someone else.

Think:

```text
Who are you?
   ↓
Who owns the file?
   ↓
What permissions do you have?
   ↓
Are you allowed to perform the operation?
```

## 2. Read a file's permissions

```bash
ls -l file.txt
```

Example:

```text
-rw-r--r-- 1 nikhil devops 120 Oct 10 file.txt
```

The important part is:

```text
-rw-r--r--
 │ │  │  │
 │ │  │  └─ others
 │ │  └──── group
 │ └─────── owner
 └───────── file type
```

## 3. Read, write and execute

For a normal file:

- `r` = read contents
- `w` = change contents
- `x` = execute the file as a program

For a directory:

- `r` = list names
- `w` = create/delete/rename entries
- `x` = enter/traverse the directory

That directory `x` meaning is very important when troubleshooting access problems.

## 4. Users and groups

Every process runs as a user.

Check your identity:

```bash
whoami
id
```

Check groups:

```bash
groups
```

A file has an owner and group:

```bash
ls -l file.txt
```

## 5. Change permissions with chmod

Example:

```bash
chmod u+x script.sh
```

Breakdown:

- `chmod` = change mode/permissions
- `u` = user/owner
- `+` = add
- `x` = execute

Other symbols:

- `g` = group
- `o` = others
- `a` = all
- `-` = remove
- `=` = set exactly

Examples:

```bash
chmod u+x script.sh
chmod g+w shared.txt
chmod o-r secret.txt
chmod a+r file.txt
```

## 6. Numeric permissions

Permissions have numbers:

```text
r = 4
w = 2
x = 1
```

Therefore:

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
0 = ---
```

Example:

```bash
chmod 750 script.sh
```

means:

```text
owner  = rwx = 7
group  = r-x = 5
others = --- = 0
```

## 7. Change ownership

```bash
sudo chown alice file.txt
sudo chown alice:devops file.txt
```

- `chown` = change owner
- `alice` = new owner
- `:` separates owner and group
- `alice:devops` = owner `alice`, group `devops`

Recursive:

```bash
sudo chown -R alice:devops /opt/myapp
```

`-R` = recursively apply to the directory and its contents.

Use recursive ownership changes carefully in production.

## 8. ACLs: permissions for specific users

Traditional permissions give you owner/group/others. Sometimes you need something more specific.

Example requirement:

> Give Alice read/write access without changing the file's main group.

Use an ACL:

```bash
setfacl -m u:alice:rw file.txt
```

Breakdown:

- `setfacl` = manage ACLs
- `-m` = modify/add an ACL entry
- `u` = user entry
- `alice` = username
- `:` = separator
- `rw` = read + write

View ACLs:

```bash
getfacl file.txt
```

Remove Alice's ACL:

```bash
setfacl -x u:alice file.txt
```

- `-x` = remove the specified ACL entry

Remove all extended ACL entries:

```bash
setfacl -b file.txt
```

- `-b` = remove all extended ACL entries

Default ACLs for newly created files in a directory:

```bash
setfacl -d -m g:devops:rwx /shared
```

- `-d` = default ACL

## 9. sudo

`sudo` allows an authorized user to run a command with another user's privileges, commonly root.

```bash
sudo systemctl restart nginx
```

Check what you are allowed to run:

```bash
sudo -l
```

Check sudo configuration syntax:

```bash
sudo visudo -c
```

`visudo` is preferred for editing sudoers because it validates syntax before saving.

## 10. Troubleshooting `Permission denied`

Do not immediately run `chmod 777`.

Use this process:

```text
1. whoami
2. id
3. ls -l target
4. getfacl target
5. namei -l /full/path/to/target
6. Check parent-directory permissions
7. Check service user if an application is involved
8. Check SELinux/AppArmor if relevant
```

`namei -l` is especially useful because it shows permissions for every directory in the path.

Example:

```bash
namei -l /opt/myapp/config/app.conf
```

## 11. Real-world example

An application runs as user `appuser` and cannot read:

```text
/opt/myapp/config/app.conf
```

You check:

```bash
ls -l /opt/myapp/config/app.conf
namei -l /opt/myapp/config/app.conf
```

The file may be readable, but `/opt/myapp/config` may not allow `appuser` to traverse it.

The lesson:

> Access to a file depends on the whole path, not only the final file.

## 12. Common mistakes

### `chmod 777` as a quick fix

It gives everyone read/write/execute access and can create security problems.

### Forgetting the service user

Your SSH user may be able to read a file while the application user cannot.

### Changing ownership recursively without checking

A careless `chown -R` can break an entire application.

### Ignoring ACLs

`ls -l` may not tell the whole story when ACLs are present.

## 13. Commands to remember

```bash
whoami
id
groups
ls -l file
chmod 750 script.sh
chown user:group file
getfacl file
setfacl -m u:alice:rw file
setfacl -x u:alice file
setfacl -b file
namei -l /path/to/file
sudo -l
sudo visudo -c
```

### Beginner takeaway

When Linux says **Permission denied**, ask:

```text
Who am I?
Who owns it?
What group am I in?
What permissions exist?
Can I traverse every directory in the path?
Is there an ACL?
Is sudo/service-user behavior involved?
```

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.