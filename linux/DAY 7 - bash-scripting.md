# DAY 7 - Bash Scripting: Variables, Loops, Functions & Exit Codes

## 1. Why Bash matters in DevOps

Bash is used for deployments, health checks, backups, cleanup, log collection, CI/CD steps and AWS CLI automation.

```text
Input → variables → logic → functions/loops → exit code → automation decision
```

A production script should be predictable, readable, safe and accurate about failures.

## 2. Script basics

```bash
#!/usr/bin/env bash

echo "Hello DevOps"
```

The shebang selects Bash through `env`.

```bash
chmod +x script.sh
./script.sh
bash script.sh
```

- `#!` = interpreter directive
- `/usr/bin/env` = locate program through `PATH`
- `bash` = interpreter
- `chmod +x` = add execute permission
- `./` = current directory

## 3. Variables and quoting

```bash
name="Nikhil"
environment="production"
count=5

echo "$name"
```

No spaces around `=`.

```bash
name="Nikhil"   # correct
name = "Nikhil" # wrong
```

Prefer quoted expansions:

```bash
echo "$name"
rm -- "$file"
```

Double quotes expand variables; single quotes prevent expansion.

## 4. Command substitution

`$(command)` runs the command and substitutes its output:

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
| `$@` | all arguments; use `"$@"` to preserve boundaries |
| `$?` | previous command's exit status |
| `$$` | current shell/script PID |

## 7. Exit codes

```text
0 = success
non-zero = failure/error
```

```bash
ls /tmp
echo "$?"
```

Critical operations should be checked:

```bash
if systemctl is-active --quiet nginx; then
  echo "nginx is running"
else
  echo "nginx is down" >&2
  exit 1
fi
```

Exit codes are how cron, CI/CD and other automation determine success/failure.

## 8. Conditions

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
[[ -f "$file" ]]  # file
[[ -d "$dir" ]]   # directory
[[ -r "$file" ]]  # readable
[[ -w "$file" ]]  # writable
[[ -x "$file" ]]  # executable
[[ -n "$value" ]] # non-empty
[[ -z "$value" ]] # empty
[[ "$a" == "$b" ]]
```

## 9. Loops

```bash
for service in nginx docker sshd; do
  echo "Checking $service"
  systemctl is-active "$service"
done
```

```bash
count=1
while [[ $count -le 5 ]]; do
  echo "Attempt $count"
  ((count++))
done
```

```bash
for ((i=1; i<=5; i++)); do
  echo "$i"
done
```

- `break` = exit loop
- `continue` = skip current iteration

## 10. Functions

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

`local` scopes the variable to the function. `return` exits the function; `exit` terminates the entire script.

## 11. Combining commands

```bash
mkdir -p /opt/app && echo "ready"
systemctl is-active --quiet nginx || echo "nginx is down"
command1; command2
```

- `&&` = next command only after success
- `||` = next command after failure
- `;` = run next command regardless of status

## 12. set -euo pipefail

```bash
set -euo pipefail
```

- `set` = change Bash options
- `-e` = exit on many unhandled command failures
- `-u` = unset variables are errors
- `pipefail` = pipeline status reflects failures from earlier commands

For critical operations, explicit checks are still clearer:

```bash
if ! systemctl restart myapp; then
  echo "ERROR: restart failed" >&2
  exit 1
fi
```

`set -e` is not a universal exception handler; understand its Bash semantics.

## 13. Debugging Bash

```bash
bash -n deploy.sh
bash -x deploy.sh
```

- `bash -n` = syntax check without executing
- `bash -x` = execution trace

Send errors to stderr:

```bash
echo "ERROR: deployment failed" >&2
```

## 14. Production failure example

Bad deployment script:

```bash
systemctl restart "$APP_SERVICE"
echo "Deployment successful"
```

If `systemctl` fails, the script can still print success.

Safer:

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

Validate:

```bash
bash -n deploy.sh
bash -x ./deploy.sh myapp
echo "$?"
```

The key lesson is that automation must not hide critical failures.

## 15. AWS and Kubernetes connection

Bash commonly wraps AWS CLI:

```bash
aws ec2 describe-instances --output json
aws s3 sync ./build "s3://$BUCKET/build/"
```

Check critical operations:

```bash
if ! aws s3 sync ./build "s3://$BUCKET/build/"; then
  echo "ERROR: S3 upload failed" >&2
  exit 1
fi
```

Kubernetes scripts may check:

```bash
kubectl get pods
kubectl logs "$POD"
kubectl rollout status deployment/myapp
```

Always check exit codes before declaring success.