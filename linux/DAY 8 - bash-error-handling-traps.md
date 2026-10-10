# DAY 8 - Bash Error Handling, traps & Safe Automation

> **Goal:** Learn how to make Bash scripts fail safely instead of leaving production systems in a broken state.

## 1. Why error handling matters

Imagine a deployment script:

```text
Stop application
   ↓
Copy new files   ← fails
   ↓
Start application
```

If the script does not handle the failure, the application may remain stopped.

Production automation should:

```text
Detect failure
   ↓
Log useful information
   ↓
Clean up
   ↓
Recover / rollback when possible
   ↓
Return failure status
```

## 2. `set -euo pipefail`

Common starting point:

```bash
set -euo pipefail
```

- `-e` = stop when an unhandled command failure occurs
- `-u` = treat unset variables as errors
- `pipefail` = make a pipeline reflect failures from earlier commands

This is helpful, but it is not a complete error-handling system. Critical commands should still be checked explicitly.

## 3. Explicitly check important commands

```bash
if ! systemctl restart myapp; then
    echo "ERROR: restart failed" >&2
    exit 1
fi
```

`!` reverses the command's success/failure result for the `if` condition.

## 4. What is `trap`?

`trap` tells Bash to run something when a signal or shell event happens.

Basic form:

```bash
trap 'echo "cleanup"' EXIT
```

Useful events/signals:

| Event | Simple meaning |
|---|---|
| `EXIT` | script is exiting |
| `ERR` | eligible command failure |
| `INT` | Ctrl+C / interrupt |
| `TERM` | termination request |
| `HUP` | hangup |

## 5. Cleanup with EXIT

Temporary resources should be cleaned up even when a script fails.

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp_dir=$(mktemp -d)

cleanup() {
    rm -rf -- "$tmp_dir"
}

trap cleanup EXIT
```

`mktemp -d` creates a unique temporary directory.

`trap cleanup EXIT` means Bash calls `cleanup` when the script exits.

## 6. Why `mktemp` is safer

Avoid predictable names such as:

```bash
/tmp/deployment
```

Prefer:

```bash
tmp_dir=$(mktemp -d)
```

The generated name is difficult to predict and reduces collisions between processes.

## 7. `trap ERR`

You can record useful failure context:

```bash
trap 'rc=$?; echo "ERROR rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR
```

Useful Bash variables:

- `$?` = previous command's exit status
- `$LINENO` = current script line
- `$BASH_COMMAND` = command being executed

Remember: `ERR` has Bash-specific rules and does not behave like a universal exception handler.

## 8. Handle Ctrl+C and termination

For long-running automation:

```bash
cleanup() {
    echo "Cleaning up..."
}

trap cleanup INT TERM EXIT
```

Possible cleanup tasks:

- remove temporary files
- release locks
- stop child processes
- restore configuration
- write final logs

## 9. Prevent duplicate jobs with flock

Imagine two deployment jobs run at the same time.

They could overwrite each other's files.

Use `flock`:

```bash
exec 9>/var/lock/myapp-deploy.lock
flock -n 9 || {
    echo "Another deployment is already running" >&2
    exit 1
}
```

Meaning:

- `exec 9>` = open a file using file descriptor 9
- `flock -n 9` = try to acquire the lock without waiting
- `||` = handle lock acquisition failure

This is better than simply checking whether a lock file exists because a stale file can remain after a crash.

## 10. Safe command construction

Avoid building shell commands as strings when possible.

Risky:

```bash
cmd="aws s3 cp $file s3://$bucket/$key"
$cmd
```

Prefer arrays:

```bash
args=(s3 cp "$file" "s3://$bucket/$key")
aws "${args[@]}"
```

Arrays preserve argument boundaries.

## 11. Validate inputs

```bash
if [[ $# -ne 2 ]]; then
    echo "Usage: $0 <environment> <version>" >&2
    exit 2
fi
```

Common convention:

```text
0 = success
1 = operational failure
2 = invalid usage/arguments
```

## 12. Production deployment example

Unsafe:

```bash
systemctl stop myapp
cp release.tar.gz /opt/myapp/
systemctl start myapp
```

If the copy fails, the service may remain stopped.

Safer thinking:

```text
Validate release first
      ↓
Create temporary workspace
      ↓
Install safely
      ↓
Stop service only when necessary
      ↓
Start service
      ↓
Health check
      ↓
Rollback if required
      ↓
Cleanup
```

## 13. AWS and Kubernetes

The same principles apply to automation around:

```bash
aws s3 cp ...
aws s3 sync ...
kubectl rollout status deployment/myapp --timeout=120s
```

Always check critical command results.

## 14. Beginner example

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp_dir=$(mktemp -d)

cleanup() {
    echo "Removing $tmp_dir"
    rm -rf -- "$tmp_dir"
}

trap cleanup EXIT
trap 'rc=$?; echo "ERROR rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR

echo "test" > "$tmp_dir/file.txt"
cat "$tmp_dir/file.txt"

false
```

What happens?

1. `false` returns a failure status.
2. The error handler can log context.
3. `set -e` stops normal execution.
4. The EXIT trap cleans the temporary directory.
5. The script returns non-zero.

### Beginner takeaway

Safe Bash automation means:

```text
Validate
   ↓
Run command
   ↓
Check result
   ↓
Log failure
   ↓
Cleanup
   ↓
Recover if possible
   ↓
Return correct exit code
```

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.