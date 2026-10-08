# DAY 8 - Bash Error Handling, Traps & Safe Automation

## 1. Why error handling matters

A production script must fail safely, clean up temporary resources, preserve useful diagnostics, and return an accurate exit code.

```text
Command fails
    ↓
Detect failure
    ↓
Log useful context
    ↓
Clean up safely
    ↓
Exit non-zero
    ↓
CI/CD / scheduler detects failure
```

## 2. `set -euo pipefail`

A common baseline is:

```bash
set -euo pipefail
```

| Part | Meaning |
|---|---|
| `set` | changes Bash shell behavior |
| `-e` | exit when a command failure is treated as an unhandled error |
| `-u` | treat unset variables as errors |
| `pipefail` | make a pipeline fail when an earlier command fails instead of only using the last command's status |

Do not treat `set -e` as a complete exception-handling system. Critical operations should still be checked explicitly.

## 3. Explicit error handling

For important commands:

```bash
if ! aws s3 sync ./build "s3://$BUCKET/build/"; then
  echo "ERROR: S3 upload failed" >&2
  exit 1
fi
```

Useful pattern:

```bash
if ! command; then
  echo "ERROR: command failed" >&2
  exit 1
fi
```

`!` reverses the command's status for the condition: success becomes false and failure becomes true.

## 4. `trap`

`trap` tells Bash to execute a command or function when a signal or shell event occurs.

Basic syntax:

```bash
trap 'commands' SIGNAL
```

Examples:

```bash
trap 'echo "Script interrupted" >&2' INT
trap 'echo "Script received TERM" >&2' TERM
```

Common events/signals:

| Event | Meaning |
|---|---|
| `EXIT` | run when the shell/script exits |
| `ERR` | run when a command failure is eligible for the ERR trap |
| `INT` | interrupt, commonly Ctrl+C |
| `TERM` | termination request |
| `HUP` | hangup/session termination |
| `RETURN` | function or sourced-script return event |

## 5. Cleanup with `trap EXIT`

Temporary files should not be left behind if a script fails.

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp_file=$(mktemp)

cleanup() {
  rm -f -- "$tmp_file"
}

trap cleanup EXIT

# automation work here
```

Breakdown:

- `mktemp` → creates a temporary file safely
- `cleanup` → reusable cleanup function
- `rm -f` → remove without prompting; do not blindly use recursive deletion
- `trap cleanup EXIT` → run cleanup when the script exits

The cleanup happens on normal completion and on many failure paths that terminate the shell.

## 6. `trap ERR`

You can log additional context when an eligible command fails:

```bash
trap 'echo "ERROR: command failed at line $LINENO" >&2' ERR
```

Useful Bash variables:

| Variable | Meaning |
|---|---|
| `$LINENO` | current script line number |
| `$BASH_COMMAND` | command Bash was executing |
| `$FUNCNAME` | function call context |
| `$?` | most recent exit status |

A stronger logging example:

```bash
trap 'rc=$?; echo "ERROR rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR
```

Be careful: `ERR` traps have Bash rules and exceptions. Do not assume every possible failure will trigger them.

## 7. Signals and graceful shutdown

Long-running scripts or automation may receive `SIGTERM` or `SIGINT`.

```bash
cleanup() {
  echo "Cleaning up..."
}

trap cleanup INT TERM EXIT
```

The goal is to leave the system in a safe state before termination.

Typical cleanup tasks:

- Remove temporary files
- Release a lock
- Stop a child process
- Remove a temporary working directory
- Restore a changed configuration
- Write final diagnostics

## 8. Lock files and duplicate automation

Two copies of the same production script running simultaneously can cause corruption or conflicting deployments.

A simple lock using `flock`:

```bash
exec 9>/var/lock/myapp-deploy.lock
flock -n 9 || {
  echo "Another deployment is already running" >&2
  exit 1
}
```

Meaning:

- `exec 9>` → opens the lock file on file descriptor 9
- `flock -n` → try to acquire the lock without waiting
- `||` → handle the case where the lock cannot be acquired

This is often safer than checking whether a lock file merely exists, because a stale file can remain after a crash.

## 9. Temporary directories

Use `mktemp` instead of predictable temporary filenames.

```bash
tmp_dir=$(mktemp -d)
trap 'rm -rf -- "$tmp_dir"' EXIT
```

| Command/part | Meaning |
|---|---|
| `mktemp -d` | create a unique temporary directory |
| `rm -rf` | recursively remove files/directories without prompting |
| `--` | marks the end of options, protecting against names beginning with `-` |
| `"$tmp_dir"` | quoted target path |

**Safety rule:** only use `rm -rf` when the variable has been validated and is known to point to the intended temporary directory.

## 10. Input validation

Never assume script arguments exist or contain safe values.

```bash
if [[ $# -ne 2 ]]; then
  echo "Usage: $0 <environment> <version>" >&2
  exit 2
fi

environment="$1"
version="$2"
```

Common exit-code convention:

- `0` → success
- `1` → generic operational failure
- `2` → incorrect usage/invalid arguments

## 11. Safe command construction

Prefer arrays when passing multiple arguments to commands.

Unsafe pattern:

```bash
cmd="aws s3 cp $file s3://$bucket/$key"
$cmd
```

Safer pattern:

```bash
args=(s3 cp "$file" "s3://$bucket/$key")
aws "${args[@]}"
```

Arrays preserve argument boundaries and reduce word-splitting problems.

## 12. Realistic production failure scenario

### Situation

A deployment script:

1. Downloads a release into `/tmp`.
2. Stops an application.
3. Copies the release.
4. Starts the application.
5. Removes temporary files.

The copy operation fails because the artifact is corrupt. The script exits without cleanup and leaves the application stopped.

### Unsafe script

```bash
systemctl stop myapp
cp ./release.tar.gz /opt/myapp/
tar -xzf /opt/myapp/release.tar.gz -C /opt/myapp
systemctl start myapp
rm -f /opt/myapp/release.tar.gz
```

Problems:

- No input validation
- No explicit error handling
- No cleanup trap
- Application can remain stopped
- Temporary artifact may remain
- The automation does not clearly report the failure

### Safer approach

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_SERVICE="${1:-}"
RELEASE="${2:-}"
TMP_DIR=""

if [[ -z "$APP_SERVICE" || -z "$RELEASE" ]]; then
  echo "Usage: $0 <service> <release-file>" >&2
  exit 2
fi

cleanup() {
  if [[ -n "$TMP_DIR" && -d "$TMP_DIR" ]]; then
    rm -rf -- "$TMP_DIR"
  fi
}

trap cleanup EXIT
trap 'rc=$?; echo "ERROR rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR

TMP_DIR=$(mktemp -d)
cp -- "$RELEASE" "$TMP_DIR/release.tar.gz"

tar -xzf "$TMP_DIR/release.tar.gz" -C "$TMP_DIR"

if ! systemctl stop "$APP_SERVICE"; then
  echo "ERROR: failed to stop $APP_SERVICE" >&2
  exit 1
fi

# deployment operation here

if ! systemctl start "$APP_SERVICE"; then
  echo "ERROR: failed to start $APP_SERVICE" >&2
  exit 1
fi

echo "Deployment completed successfully"
```

### Production lesson

Error handling is not just about making the script exit. It is about making failure **observable, diagnosable and safe**.

For deployments, consider an even stronger design: validate the release before stopping the live service whenever possible, and have a rollback strategy if the new version cannot start.

## 13. AWS / DevOps connection

Safe Bash automation commonly wraps:

```bash
aws s3 cp
aws s3 sync
aws ec2 describe-instances
aws cloudwatch get-metric-data
```

Always validate critical AWS CLI operations:

```bash
if ! aws s3 cp "$artifact" "s3://$bucket/$key"; then
  echo "ERROR: artifact upload failed" >&2
  exit 1
fi
```

For EC2 deployment scripts, combine:

```text
Bash
 ↓
systemctl
 ↓
application
 ↓
AWS CLI / AWS APIs
```

A non-zero AWS CLI status should not be silently ignored.

Kubernetes automation follows the same principle:

```bash
if ! kubectl rollout status deployment/myapp --timeout=120s; then
  echo "ERROR: rollout failed" >&2
  exit 1
fi
```

## 14. Practical lab

Create a script that creates a temporary directory, writes a file, intentionally fails, and cleans up.

```bash
#!/usr/bin/env bash
set -euo pipefail

TMP_DIR=$(mktemp -d)

cleanup() {
  echo "Cleaning $TMP_DIR"
  rm -rf -- "$TMP_DIR"
}

trap cleanup EXIT
trap 'echo "ERROR at line $LINENO: $BASH_COMMAND" >&2' ERR

echo "hello" > "$TMP_DIR/test.txt"
cat "$TMP_DIR/test.txt"

false

echo "This should not run"
```

Run:

```bash
bash -x ./trap-lab.sh
echo "$?"
```

Observe:

1. `false` returns non-zero.
2. The ERR trap logs context.
3. `set -e` stops normal execution.
4. The EXIT trap removes the temporary directory.
5. The script returns a non-zero status.

Then remove `set -e` and observe how the behavior changes.

# Interview Questions

<details><summary>1. What is trap in Bash?</summary>`trap` registers commands or functions to run when specified signals or shell events occur.</details>

<details><summary>2. What is trap EXIT used for?</summary>It is commonly used for cleanup that should happen when the script exits, such as removing temporary files or releasing resources.</details>

<details><summary>3. What is trap ERR?</summary>It can run a handler when an eligible command fails. It has Bash-specific exception rules, so it is not a universal exception handler.</details>

<details><summary>4. What does SIGTERM mean?</summary>It is a termination request that allows a process or script an opportunity to perform graceful cleanup.</details>

<details><summary>5. SIGTERM vs SIGKILL?</summary>SIGTERM requests graceful termination and can be handled. SIGKILL forcibly terminates the process and cannot be handled or trapped.</details>

<details><summary>6. Why use mktemp?</summary>It creates unpredictable temporary filenames/directories and is safer than manually constructing predictable names under `/tmp`.</details>

<details><summary>7. Why use a cleanup trap?</summary>To ensure temporary resources are cleaned up when the script exits, including many failure paths.</details>

<details><summary>8. What does set -u protect against?</summary>It treats unset variables as errors, helping catch typos and missing inputs.</details>

<details><summary>9. Why is set -e not enough?</summary>Bash has contexts where failures do not behave as a simple global exception. Critical operations should be checked explicitly.</details>

<details><summary>10. What is pipefail?</summary>It causes a pipeline's status to reflect a failing command earlier in the pipeline rather than only the final command.</details>

<details><summary>11. Why validate script arguments?</summary>To prevent missing or invalid inputs from causing destructive, confusing or incorrect automation behavior.</details>

<details><summary>12. What does exit 2 commonly represent?</summary>It is commonly used for incorrect command usage or invalid arguments, although exact exit-code conventions should be documented by the script.</details>

<details><summary>13. Why use flock?</summary>It prevents multiple instances of an automation job from operating on the same resources simultaneously.</details>

<details><summary>14. Why is checking for a lock file alone unreliable?</summary>A stale lock file can remain after a crash. File locking with `flock` represents an actual held lock rather than merely file existence.</details>

<details><summary>15. Why prefer arrays for complex commands?</summary>Arrays preserve argument boundaries and reduce word-splitting and quoting problems.</details>

<details><summary>16. How would you log the failing line?</summary>Use an ERR trap with `$LINENO` and `$BASH_COMMAND`, for example `trap 'rc=$?; echo "ERROR rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR`.</details>

<details><summary>17. A deployment script fails after stopping the service. What should you consider?</summary>Cleanup, service recovery, rollback and whether validation could happen before stopping the live service.</details>

<details><summary>18. How should a script handle an AWS CLI failure?</summary>Check the AWS CLI command's exit status, log the failure, stop unsafe follow-up actions and return a non-zero status.</details>

<details><summary>19. How do traps help long-running automation?</summary>They can handle termination signals and perform cleanup or graceful shutdown before the script exits.</details>

<details><summary>20. What makes Bash automation production-safe?</summary>Input validation, quoting, controlled command execution, explicit error handling, accurate exit codes, cleanup traps, locking where needed, useful logging, rollback/recovery planning and testing.</details>

## Production mental model

```text
Validate
   ↓
Prepare safely
   ↓
Trap cleanup / termination
   ↓
Run critical operation
   ↓
Check status
   ↓
Log context
   ↓
Rollback / recover if required
   ↓
Cleanup
   ↓
Return accurate exit code
```

**Key lesson:** safe automation should assume that commands can fail, connections can disappear, users can interrupt scripts, and partial changes can happen. Design the script so failure leaves the system in the safest recoverable state.