# DAY 7 - Bash Scripting: Variables, Loops, Functions & Exit Codes

## 1. Why Bash matters in DevOps

Bash is widely used for deployment scripts, health checks, backups, log collection, cleanup, CI/CD steps and AWS CLI automation.

Production mental model:

```text
Input → Variables → Logic → Functions/Loops → Exit code → Automation decision
```

A good script should be predictable, readable, safe, and able to report failures accurately.

## 2. Script basics

```bash
#!/usr/bin/env bash

echo "Hello DevOps"
```

The first line is the **shebang**. `#!/usr/bin/env bash` asks the environment to locate Bash.

```bash
chmod +x script.sh
./script.sh
bash script.sh
```

| Part | Meaning |
|---|---|
| `#!` | tells the OS to use an interpreter |
| `/usr/bin/env` | locates a program using `PATH` |
| `bash` | shell/interpreter |
| `chmod` | change permissions |
| `+x` | add execute permission |
| `./` | current directory |

## 3. Variables

```bash
name="Nikhil"
environment="production"
count=5

echo "$name"
echo "$environment"
echo "$count"
```

**Important:** no spaces around `=`.

```bash
name = "Nikhil"   # WRONG
name="Nikhil"     # CORRECT
```

### Quoting

Prefer quoting variable expansions:

```bash
echo "$name"
rm -- "$file"
```

Double quotes allow variable expansion. Single quotes prevent it:

```bash
name="Nikhil"
echo '$name'    # prints $name
echo "$name"    # prints Nikhil
```

## 4. Command substitution

`$(command)` means: **run the command and substitute its output**.

```bash
hostname=$(hostname)
today=$(date +%F)
current_user=$(whoami)
files=$(find /var/log -type f | wc -l)
```

## 5. Environment variables

```bash
echo "$PATH"
echo "$HOME"
echo "$USER"
echo "$PWD"
export APP_ENV=production
env | grep APP_ENV
```

`export` makes a variable available to child processes.

## 6. Script arguments

For:

```bash
./deploy.sh production v1.2.0
```

| Variable | Meaning |
|---|---|
| `$0` | script name/path |
| `$1` | first argument |
| `$2` | second argument |
| `$#` | number of arguments |
| `$@` | all arguments; quote as `"$@"` to preserve boundaries |
| `$?` | exit status of the previous command |
| `$$` | current shell/script PID |

Example:

```bash
echo "Script: $0"
echo "Environment: $1"
echo "Version: $2"
echo "Arguments: $#"

for arg in "$@"; do
  echo "$arg"
done
```

## 7. Exit codes

Every Linux command returns an exit status.

```text
0       = success
non-zero = failure/error
```

```bash
ls /tmp
echo "$?"
```

Use explicit failure handling:

```bash
if systemctl is-active --quiet nginx; then
  echo "nginx is running"
else
  echo "nginx is down" >&2
  exit 1
fi
```

`exit 0` means success. A non-zero value reports failure; `1` is a common generic failure code.

### Why this matters in DevOps

CI/CD systems, cron jobs and automation tools can use the script's exit code to determine whether a step succeeded.

```text
exit 0     → success
exit != 0  → failure
```

## 8. if conditions

```bash
if [[ "$ENV" == "production" ]]; then
  echo "Production"
elif [[ "$ENV" == "staging" ]]; then
  echo "Staging"
else
  echo "Unknown environment"
fi
```

Useful Bash tests:

```bash
[[ -f "$file" ]]       # regular file exists
[[ -d "$dir" ]]        # directory exists
[[ -r "$file" ]]       # readable
[[ -w "$file" ]]       # writable
[[ -x "$file" ]]       # executable
[[ -n "$value" ]]      # non-empty
[[ -z "$value" ]]      # empty
[[ "$a" == "$b" ]]    # strings equal
```

`[[ ... ]]` is Bash conditional syntax and is preferred for Bash-specific scripts over relying on older test syntax.

## 9. Loops

### for

```bash
for service in nginx docker sshd; do
  echo "Checking $service"
  systemctl is-active "$service"
done
```

### while

```bash
count=1
while [[ $count -le 5 ]]; do
  echo "Attempt $count"
  ((count++))
done
```

### C-style for

```bash
for ((i=1; i<=5; i++)); do
  echo "$i"
done
```

### break / continue

- `break` → immediately exit the loop
- `continue` → skip the current iteration and move to the next

```bash
for file in /var/log/*.log; do
  [[ -f "$file" ]] || continue
  echo "$file"
done
```

## 10. Functions

Functions package reusable logic.

```bash
check_service() {
  local service="$1"

  if systemctl is-active --quiet "$service"; then
    echo "$service is running"
    return 0
  else
    echo "$service is down"
    return 1
  fi
}

check_service nginx
```

| Part | Meaning |
|---|---|
| `check_service()` | defines the function |
| `local` | variable is scoped to the function |
| `$1` | first function argument |
| `return 0` | function reports success |
| `return 1` | function reports failure |

### `return` vs `exit`

`return` exits the current function. `exit` terminates the entire script/process.

Do not use `exit` inside reusable functions unless terminating the whole script is intentional.

## 11. local variables

```bash
check_disk() {
  local path="$1"
  local usage
  usage=$(df -P "$path" | awk 'NR==2 {gsub("%", "", $5); print $5}')
  echo "Usage: ${usage}%"
}
```

`local` prevents function variables from unnecessarily changing variables in the surrounding shell scope.

## 12. Combining commands

### `&&` — run next command only if previous succeeds

```bash
mkdir -p /opt/app && echo "Directory ready"
```

### `||` — run next command if previous fails

```bash
systemctl is-active --quiet nginx || echo "nginx is down"
```

### `;` — run next command regardless of previous result

```bash
command1; command2
```

This distinction is important in production scripts.

## 13. `set -euo pipefail`

A common Bash safety baseline:

```bash
set -euo pipefail
```

| Part | Meaning |
|---|---|
| `set` | changes Bash shell options |
| `-e` | exit when a command failure is treated as an error |
| `-u` | treat unset variables as errors |
| `pipefail` | pipeline status reflects failures from commands before the last command |

Example:

```bash
set -euo pipefail
result=$(some_command)
echo "Result: $result"
```

**Important:** `set -e` is not a universal exception handler. For critical operations, explicit checks are clearer:

```bash
if ! systemctl restart myapp; then
  echo "ERROR: restart failed" >&2
  exit 1
fi
```

## 14. Debugging Bash scripts

```bash
bash -n deploy.sh
bash -x deploy.sh
```

- `bash -n` → syntax check without executing
- `bash -x` → trace commands as Bash executes them

Inside a script:

```bash
set -x
set +x
```

Send errors to stderr:

```bash
echo "ERROR: deployment failed" >&2
```

## 15. Realistic production failure scenario

A deployment script restarts an application on an EC2 instance:

```bash
#!/usr/bin/env bash

APP_SERVICE="$1"

systemctl restart "$APP_SERVICE"
echo "Deployment successful"
```

The engineer runs:

```bash
./deploy.sh myapp
```

The service name is wrong, `systemctl` fails, but the script still prints `Deployment successful`.

### Root cause

The script never checked the exit status of the critical command.

Test it:

```bash
systemctl restart wrong-service
echo "$?"
```

The result is non-zero.

### Production-safe version

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_SERVICE="${1:-}"

if [[ -z "$APP_SERVICE" ]]; then
  echo "Usage: $0 <service-name>" >&2
  exit 2
fi

if ! systemctl restart "$APP_SERVICE"; then
  echo "ERROR: failed to restart $APP_SERVICE" >&2
  exit 1
fi

echo "Deployment successful"
```

Validate and debug:

```bash
bash -n deploy.sh
bash -x ./deploy.sh myapp
echo "$?"
```

### Production lesson

The dangerous problem was not only that the service failed. It was that **the automation hid the failure**. A DevOps script must return an accurate non-zero status when a critical operation fails.

## 16. AWS / DevOps connection

Bash commonly wraps AWS CLI commands:

```bash
aws ec2 describe-instances --output json
aws s3 sync ./build "s3://$BUCKET/build/"
```

Check important AWS operations:

```bash
if ! aws s3 sync ./build "s3://$BUCKET/build/"; then
  echo "ERROR: S3 upload failed" >&2
  exit 1
fi
```

Common uses:

- EC2 health checks
- deployment automation
- S3 uploads
- backups and cleanup
- log collection
- CI/CD steps
- AWS CLI orchestration

Kubernetes scripts may wrap:

```bash
kubectl get pods
kubectl logs "$POD"
kubectl rollout status deployment/myapp
```

Again, check exit codes before declaring success.

## 17. Practical lab

Create `health-check.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

check_service() {
  local service="$1"

  if systemctl is-active --quiet "$service"; then
    echo "OK: $service is running"
    return 0
  fi

  echo "ERROR: $service is not running" >&2
  return 1
}

services=(ssh nginx)

for service in "${services[@]}"; do
  if ! check_service "$service"; then
    echo "Health check failed for $service" >&2
    exit 1
  fi
done

echo "All service checks passed"
```

Practice:

```bash
bash -n health-check.sh
bash -x health-check.sh
echo "$?"
```

Then intentionally use a service name that does not exist and observe the function failure, loop behavior, script exit, exit code and error output.

# Interview Questions

<details><summary>1. What is a Bash variable?</summary>A variable stores a value that can be referenced later. Assignment uses `name=value` with no spaces around `=`.</details>

<details><summary>2. Why quote variables?</summary>Quoting such as `"$file"` prevents spaces and many special characters from being interpreted as separate shell words or syntax.</details>

<details><summary>3. What does `$?` mean?</summary>It contains the exit status of the most recently executed command or pipeline.</details>

<details><summary>4. What does exit code 0 mean?</summary>Conventionally, successful completion. Non-zero values indicate failure or another exceptional condition.</details>

<details><summary>5. Difference between return and exit?</summary>`return` exits a function and provides a status to its caller. `exit` terminates the entire script/process.</details>

<details><summary>6. What does `$1` represent?</summary>The first positional argument passed to the script or function.</details>

<details><summary>7. What is `$@`?</summary>All positional arguments. Quoted as `"$@"`, Bash preserves each argument as a separate word.</details>

<details><summary>8. What is command substitution?</summary>`$(command)` executes a command and substitutes its output into the surrounding command or assignment.</details>

<details><summary>9. What does export do?</summary>It makes a shell variable available to child processes.</details>

<details><summary>10. Difference between for and while?</summary>`for` commonly iterates over a collection/counter; `while` repeats while a condition remains true.</details>

<details><summary>11. What is a Bash function?</summary>A reusable block of shell commands that can accept arguments and return a status code.</details>

<details><summary>12. Why use local?</summary>It scopes a variable to the function and reduces accidental changes to variables outside it.</details>

<details><summary>13. What does set -euo pipefail do?</summary>`-e` handles many command failures by exiting, `-u` treats unset variables as errors, and `pipefail` propagates failures from earlier commands in a pipeline.</details>

<details><summary>14. Is set -e enough for production error handling?</summary>No. Bash has contexts where `set -e` does not behave like a universal exception handler. Critical operations should still have explicit checks.</details>

<details><summary>15. Difference between && and ;?</summary>`&&` runs the next command only after success. `;` runs the next command regardless of the previous command's status.</details>

<details><summary>16. How do you check Bash syntax?</summary>Use `bash -n script.sh`.</details>

<details><summary>17. How do you trace a Bash script?</summary>Use `bash -x script.sh` or `set -x` inside the script.</details>

<details><summary>18. A deployment script reports success after a failed deployment. What do you check?</summary>Check whether critical command exit codes are tested. Add explicit error handling and meaningful non-zero exits.</details>

<details><summary>19. Why are exit codes important in CI/CD?</summary>CI/CD systems use process exit status to determine whether a step succeeded or failed. A false zero can cause an incorrect successful deployment.</details>

<details><summary>20. What is a good production Bash approach?</summary>Use a clear shebang, quote variables, validate inputs, use functions, check critical commands, return meaningful exit codes, log failures to stderr, use safe error handling, and test with `bash -n` and `bash -x`.</details>

## Production mental model

```text
Validate inputs
      ↓
Set variables safely
      ↓
Run command
      ↓
Check exit code
      ↓
Handle failure
      ↓
Continue only if safe
      ↓
Return accurate final status
```

**Most important lesson:** a Bash script is part of your automation system. If it reports success when the underlying operation failed, it can turn a small operational failure into a production incident.
