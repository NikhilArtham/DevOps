# DAY 8 - Bash Error Handling, Traps & Safe Automation

## 1. Production error-handling model

```text
Command fails → detect → log context → cleanup → exit non-zero → automation detects failure
```

A production script must fail safely, clean up temporary resources, preserve diagnostics and return an accurate status.

## 2. set -euo pipefail

```bash
set -euo pipefail
```

- `-e` = exit when a command failure is treated as an unhandled error
- `-u` = unset variables are errors
- `pipefail` = pipeline status reflects earlier failures

`set -e` is not a universal exception handler. Critical operations should still be checked explicitly.

## 3. Explicit error handling

```bash
if ! aws s3 sync ./build "s3://$BUCKET/build/"; then
  echo "ERROR: S3 upload failed" >&2
  exit 1
fi
```

`!` reverses the status for the condition, so a failed command enters the `then` block.

## 4. trap

`trap` registers a command/function for signals or shell events.

```bash
trap 'echo "Interrupted" >&2' INT
trap 'echo "TERM received" >&2' TERM
```

Common events:

- `EXIT` = script/shell exit
- `ERR` = eligible command failure
- `INT` = interrupt/Ctrl+C
- `TERM` = termination request
- `HUP` = hangup
- `RETURN` = function/sourced-script return event

## 5. Cleanup with EXIT

```bash
#!/usr/bin/env bash
set -euo pipefail

tmp_file=$(mktemp)

cleanup() {
  rm -f -- "$tmp_file"
}

trap cleanup EXIT
```

`mktemp` creates a safe temporary file. The `EXIT` trap provides cleanup on normal completion and many failure paths.

## 6. ERR trap and diagnostics

```bash
trap 'rc=$?; echo "ERROR rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR
```

Useful Bash variables:

- `$LINENO` = script line number
- `$BASH_COMMAND` = command being executed
- `$FUNCNAME` = function context
- `$?` = most recent exit status

ERR traps have Bash-specific exceptions, so do not assume every failure triggers them.

## 7. Graceful signal handling

```bash
cleanup() {
  echo "Cleaning up..."
}

trap cleanup INT TERM EXIT
```

Cleanup may include temporary files, locks, child processes, temporary configuration changes and final diagnostics.

## 8. Prevent duplicate automation with flock

```bash
exec 9>/var/lock/myapp-deploy.lock
flock -n 9 || {
  echo "Another deployment is already running" >&2
  exit 1
}
```

- `exec 9>` = open lock file on file descriptor 9
- `flock -n` = acquire without waiting
- `||` = handle lock contention

`flock` is safer than checking whether a lock filename merely exists because stale files can remain after crashes.

## 9. Temporary directories

```bash
tmp_dir=$(mktemp -d)
trap 'rm -rf -- "$tmp_dir"' EXIT
```

- `mktemp -d` = unique temporary directory
- `rm -rf` = recursively remove
- `--` = end of options
- quoted variable = preserve path as one argument

Only use `rm -rf` when the variable has been validated and is known to point to the intended temporary directory.

## 10. Input validation

```bash
if [[ $# -ne 2 ]]; then
  echo "Usage: $0 <environment> <version>" >&2
  exit 2
fi

environment="$1"
version="$2"
```

Common convention:

- `0` = success
- `1` = generic operational failure
- `2` = incorrect usage/invalid arguments

## 11. Safe command construction

Avoid building shell commands as strings:

```bash
cmd="aws s3 cp $file s3://$bucket/$key"
$cmd
```

Prefer arrays:

```bash
args=(s3 cp "$file" "s3://$bucket/$key")
aws "${args[@]}"
```

Arrays preserve argument boundaries and reduce word-splitting problems.

## 12. Production deployment failure

An unsafe deployment can:

1. Stop the service.
2. Copy a corrupt release.
3. Fail during extraction.
4. Exit without cleanup.
5. Leave the service stopped.

Safer structure:

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

# deployment operation

if ! systemctl start "$APP_SERVICE"; then
  echo "ERROR: failed to start $APP_SERVICE" >&2
  exit 1
fi

echo "Deployment completed successfully"
```

For real production deployments, validate artifacts before stopping the live service where possible and have an explicit rollback/recovery strategy.

## 13. AWS and Kubernetes connection

Safe Bash automation commonly wraps:

```bash
aws s3 cp
aws s3 sync
aws ec2 describe-instances
aws cloudwatch get-metric-data
```

Check critical AWS commands:

```bash
if ! aws s3 cp "$artifact" "s3://$bucket/$key"; then
  echo "ERROR: artifact upload failed" >&2
  exit 1
fi
```

Kubernetes:

```bash
if ! kubectl rollout status deployment/myapp --timeout=120s; then
  echo "ERROR: rollout failed" >&2
  exit 1
fi
```

The principle is the same: do not silently ignore a non-zero command status.

## Command cheat sheet

```bash
set -euo pipefail
if ! command; then ... fi
trap cleanup EXIT
trap 'rc=$?; echo "rc=$rc line=$LINENO command=$BASH_COMMAND" >&2' ERR
tmp_dir=$(mktemp -d)
exec 9>/var/lock/job.lock
flock -n 9
rm -rf -- "$tmp_dir"
```