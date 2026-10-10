# DAY 7 - Bash Scripting Basics

> **Goal:** Learn how to turn Linux commands into reusable automation scripts.

## 1. What is Bash?

Bash is a command-line shell commonly available on Linux.

You already use Bash when you type commands such as:

```bash
ls
cd /var/log
systemctl status nginx
```

A Bash script is simply a file containing commands that Bash can execute in sequence.

## 2. Your first script

Create:

```bash
nano hello.sh
```

Put this inside:

```bash
#!/usr/bin/env bash

echo "Hello DevOps"
```

The first line is called the **shebang**. It tells the operating system which interpreter should run the script.

Run it:

```bash
bash hello.sh
```

Or make it executable:

```bash
chmod +x hello.sh
./hello.sh
```

## 3. Variables

```bash
name="Nikhil"
echo "$name"
```

Important: do not put spaces around `=`.

Good:

```bash
name="Nikhil"
```

Bad:

```bash
name = "Nikhil"
```

Use quotes around variables when they may contain spaces:

```bash
echo "$name"
```

## 4. Command substitution

You can save command output in a variable:

```bash
hostname=$(hostname)
echo "Server: $hostname"
```

`$(...)` means: run the command and use its output.

## 5. Script arguments

If you run:

```bash
./deploy.sh production v1.2
```

Then:

- `$0` = script name
- `$1` = first argument (`production`)
- `$2` = second argument (`v1.2`)
- `$#` = number of arguments
- `$@` = all arguments

Example:

```bash
echo "Environment: $1"
echo "Version: $2"
```

## 6. Exit codes

Every command returns an exit status.

```text
0     = success
non-0 = failure
```

Check the previous command:

```bash
echo "$?"
```

This is extremely important in automation.

## 7. if statements

```bash
if systemctl is-active --quiet nginx; then
    echo "Nginx is running"
else
    echo "Nginx is not running"
fi
```

`if` checks the command's exit status.

## 8. Comparing values

```bash
if [[ "$ENV" == "production" ]]; then
    echo "Production deployment"
fi
```

Common comparisons:

```bash
[[ "$a" == "$b" ]]   # strings equal
[[ "$a" != "$b" ]]   # strings different
[[ -z "$a" ]]         # empty
[[ -n "$a" ]]         # not empty
[[ $n -gt 10 ]]        # number greater than 10
```

## 9. Loops

### for loop

```bash
for server in web1 web2 web3; do
    echo "Checking $server"
done
```

### while loop

```bash
count=1
while [[ $count -le 3 ]]; do
    echo "$count"
    ((count++))
done
```

`break` stops a loop. `continue` skips to the next iteration.

## 10. Functions

Functions allow you to reuse logic.

```bash
check_service() {
    systemctl is-active --quiet "$1"
}

check_service nginx
```

Use `local` for function variables when appropriate:

```bash
check_disk() {
    local usage
    usage=$(df -P / | awk 'NR==2 {print $5}' | tr -d '%')
    echo "Disk usage: $usage%"
}
```

## 11. `return` vs `exit`

This is important:

- `return` → leave a function
- `exit` → terminate the entire script

Example:

```bash
check() {
    return 1
}

if ! check; then
    echo "Check failed"
    exit 1
fi
```

## 12. `&&`, `||` and `;`

```bash
command1 && command2
```

Run command2 only if command1 succeeds.

```bash
command1 || command2
```

Run command2 if command1 fails.

```bash
command1 ; command2
```

Run command2 regardless of command1's result.

## 13. `set -euo pipefail`

A common safety baseline:

```bash
set -euo pipefail
```

- `-e` = stop on many unhandled command failures
- `-u` = treat unset variables as errors
- `pipefail` = a pipeline can fail if an earlier command fails

Do not assume this replaces explicit checks for critical operations.

## 14. Debugging a script

Syntax check:

```bash
bash -n script.sh
```

Trace commands:

```bash
bash -x script.sh
```

`-n` checks syntax without executing the script. `-x` prints commands as Bash executes them.

## 15. Real DevOps example

A deployment script might do:

```text
Download artifact
      ↓
Validate artifact
      ↓
Stop service
      ↓
Deploy files
      ↓
Start service
      ↓
Check health
```

A dangerous script may print `Deployment successful` even if the service restart failed.

A better approach checks every critical operation:

```bash
if ! systemctl restart myapp; then
    echo "ERROR: service restart failed" >&2
    exit 1
fi
```

## 16. AWS / Kubernetes connection

Bash is commonly used to automate:

```bash
aws s3 sync ...
aws ec2 describe-instances ...
kubectl rollout status deployment/myapp
```

Example:

```bash
if ! kubectl rollout status deployment/myapp --timeout=120s; then
    echo "Rollout failed" >&2
    exit 1
fi
```

## 17. Beginner script example

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE="${1:-}"

if [[ -z "$SERVICE" ]]; then
    echo "Usage: $0 <service>" >&2
    exit 2
fi

if systemctl is-active --quiet "$SERVICE"; then
    echo "$SERVICE is running"
else
    echo "$SERVICE is NOT running" >&2
    exit 1
fi
```

This script demonstrates arguments, validation, conditionals and exit codes.

### Beginner takeaway

A Bash script is just **Linux commands + logic + error handling**.

Learn this order:

```text
Variables
   ↓
Arguments
   ↓
Conditions
   ↓
Loops
   ↓
Functions
   ↓
Exit codes
   ↓
Error handling
   ↓
Automation
```

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.