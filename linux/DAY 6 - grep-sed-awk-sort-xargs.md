# DAY 6 - grep, sed, awk, sort & xargs for Production Debugging

## 1. Mental model

Production debugging often turns noisy text into useful evidence:

```text
logs/output → grep → sed → awk → sort → xargs
```

The commands become powerful when combined with pipes, but understand the input format before building a pipeline.

## 2. grep — find matching text

```bash
grep "ERROR" app.log
grep -i "error" app.log
grep -n "ERROR" app.log
grep -v "INFO" app.log
grep -c "ERROR" app.log
grep -w "failed" app.log
grep -C 3 "OutOfMemory" app.log
grep -rni "connection refused" /var/log/myapp/
grep -Ei "error|failed|timeout" app.log
```

Important flags:

- `-i` = ignore case
- `-v` = invert match
- `-n` = line numbers
- `-r`/`-R` = recursive search (`-R` follows symlinks)
- `-l` = filenames with matches
- `-L` = filenames without matches
- `-c` = count matches
- `-E` = extended regular expressions
- `-w` = whole word
- `-A N` = N lines after
- `-B N` = N lines before
- `-C N` = N lines around

## 3. sed — stream editor

`sed` processes text line by line for substitution, selection and deletion.

```bash
echo "ERROR old-server" | sed 's/old-server/new-server/'
sed 's/old-server/new-server/g' app.log
sed -n '10,20p' app.log
sed '5d' app.log
```

An expression such as `s/old/new/g` means:

- `s` = substitute
- `old` = search text
- `new` = replacement
- `g` = all matches on each line

Important flags:

- `-n` = suppress automatic printing
- `-e` = expression
- `-i` = edit file in place

During production incidents, prefer non-destructive output first. `sed -i` changes the actual file.

## 4. awk — fields, filters and calculations

`awk` is excellent for structured text.

```bash
awk '{print $1}' app.log
awk '{print $1, $2, $8}' data.txt
awk -F: '{print $1, $7}' /etc/passwd
awk '$3 > 80 {print $0}' metrics.txt
awk '{sum += $5} END {print sum}' access.log
```

Important concepts:

- `$0` = entire current line
- `$1`, `$2`, ... = fields
- `-F` = field separator
- `BEGIN` = before input processing
- `END` = after input processing
- `~` = matches regex
- `!~` = does not match regex
- `$NF` = last field

## 5. sort

```bash
sort numbers.txt
sort -nr numbers.txt
du -h /var/log/* 2>/dev/null | sort -hr
sort -k2 data.txt
```

Flags:

- `-r` = reverse
- `-n` = numeric
- `-h` = human-readable numeric
- `-k` = sort by field/key
- `-t` = delimiter
- `-u` = unique output
- `-o` = output file

## 6. xargs

`xargs` converts standard input into command arguments.

```bash
printf '%s\n' file1 file2 file3 | xargs ls -l
printf '%s\n' app1 app2 app3 | xargs -n 1 systemctl status
```

Important flags:

- `-n N` = max N input items per command
- `-I {}` = replace placeholder with each input item
- `-0` = read NUL-separated input
- `-r` = do not run command when there is no input on GNU xargs
- `-P N` = parallel execution; use carefully

Safer filename handling:

```bash
find /var/log -type f -print0 | xargs -0 ls -lh
```

Never blindly pipe arbitrary output into destructive commands such as `rm`.

## 7. High-value pipelines

Count HTTP status codes:

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

Find top client IPs:

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
```

Find context around errors:

```bash
grep -n -C 5 "ERROR" app.log
```

Find largest logs:

```bash
du -h /var/log/* 2>/dev/null | sort -hr | head -20
```

Find unique failed users:

```bash
grep "authentication failed" auth.log | awk '{print $NF}' | sort -u
```

## 8. Realistic API incident

For an API with rising HTTP 500s:

```text
user complaint
 ↓
count status codes
 ↓
find failing endpoint
 ↓
find affected source/IP
 ↓
correlate application logs
 ↓
identify dependency error
 ↓
verify dependency/network state
```

Example:

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
grep '" 500 ' access.log | awk '{print $7}' | sort | uniq -c | sort -nr
grep '" 500 ' access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head
journalctl -u myapp --since '09:05' --until '09:15' --no-pager | grep -Ei 'error|timeout|database|exception'
```

If logs show database timeouts, verify the database/network rather than blindly restarting the application.

## 9. Production-safe habits

- Inspect output before destructive actions.
- Prefer non-destructive `sed` output before `sed -i`.
- Quote shell variables.
- Understand delimiters before using `awk`/`sort`.
- Preserve logs as evidence.
- Use `find -print0 | xargs -0` for arbitrary filenames.
- Be cautious with `xargs -P` in production.

## 10. AWS and Kubernetes connection

On EC2, use these tools with application logs, deployment logs and CloudWatch/CloudTrail evidence.

Kubernetes examples:

```bash
kubectl logs deployment/myapp | grep -i error
kubectl logs pod/myapp-abc123 --since=30m | grep -Ei 'error|timeout|failed'
kubectl logs pod/myapp-abc123 | grep -i timeout | awk '{print $1}' | sort | uniq -c | sort -nr
```

Remember that `kubectl logs` retrieves container logs; the underlying collection/storage architecture depends on the cluster.

## Command cheat sheet

```bash
grep -in "pattern" file
grep -Ei "error|failed|timeout" file
grep -n -C 3 "ERROR" file
sed -n '10,20p' file
sed 's/old/new/g' file
awk '{print $1}' file
awk -F: '{print $1,$7}' /etc/passwd
sort -nr file
du -h /var/log/* 2>/dev/null | sort -hr
find . -print0 | xargs -0 ls -lh
```