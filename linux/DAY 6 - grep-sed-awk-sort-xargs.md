# DAY 6 - grep, sed, awk, sort & xargs for Production Debugging

## 1. Why these commands matter

Production debugging often means turning large amounts of text into useful evidence.

Think of these commands as a toolbox:

```text
Logs / command output
        ↓
      grep     → find relevant lines
        ↓
      sed      → inspect/transform text
        ↓
      awk      → extract fields / calculate / filter
        ↓
      sort     → order results
        ↓
      xargs    → pass results to another command
```

The real power comes from combining them with pipes:

```bash
command | grep ... | awk ... | sort ...
```

**Production rule:** first understand the data format, then filter and transform it. Do not run a complicated pipeline blindly.

---

## 2. `grep` — find matching text

`grep` searches input for lines matching a pattern.

Basic:

```bash
grep "ERROR" app.log
grep "timeout" app.log
grep -i "error" app.log
```

### Command anatomy

| Part | Meaning |
|---|---|
| `grep` | search text for a matching pattern |
| `"ERROR"` | search pattern |
| `-i` | ignore case |
| `-v` | invert match; show lines that do **not** match |
| `-n` | show line numbers |
| `-r` / `-R` | recursively search directories; `-R` also follows symlinks |
| `-l` | show only filenames containing a match |
| `-L` | show filenames with no match |
| `-c` | count matching lines |
| `-E` | use extended regular expressions |
| `-w` | match whole words |
| `-A N` | show N lines after a match |
| `-B N` | show N lines before a match |
| `-C N` | show N lines before and after a match |

Examples:

```bash
grep -n "ERROR" app.log
grep -in "timeout" app.log
grep -v "INFO" app.log
grep -c "ERROR" app.log
grep -w "failed" app.log
grep -C 3 "OutOfMemory" app.log
```

Read:

```bash
grep -in "timeout" app.log
```

as:

> Search `app.log` for `timeout`, ignore case, and show line numbers.

### Search multiple files

```bash
grep -n "ERROR" *.log
grep -rni "connection refused" /var/log/myapp/
```

### Production use

```bash
journalctl -u myapp --since "1 hour ago" --no-pager | grep -i "error\|failed\|timeout"
```

When using regular expressions, `\|` means OR in basic grep regular-expression syntax. `grep -E` can make OR expressions easier to read:

```bash
grep -Ei "error|failed|timeout" app.log
```

---

## 3. `sed` — stream editor

`sed` processes text line by line. It is useful for searching, replacing, deleting or printing selected lines.

### Command anatomy

| Part | Meaning |
|---|---|
| `sed` | stream editor |
| `-n` | suppress automatic printing |
| `-e` | specify a sed expression |
| `-i` | edit the file in place; use carefully in production |
| `s` | substitute |
| `g` | replace all matches on each line |
| `p` | print |
| `d` | delete from sed output |

### Replace text in output

```bash
echo "ERROR old-server" | sed 's/old-server/new-server/'
```

`sed 's/old-server/new-server/'` means:

- `s` → substitute
- `old-server` → search text
- `new-server` → replacement

Replace every occurrence on each line:

```bash
sed 's/old-server/new-server/g' app.log
```

### Print selected lines

```bash
sed -n '10,20p' app.log
```

Meaning:

- `-n` → do not print every line automatically
- `10,20` → line range
- `p` → print those lines

### Delete lines from output

```bash
sed '5d' app.log
```

This does not modify the original file unless `-i` is used.

### Production warning

Be careful with:

```bash
sed -i 's/old/new/g' config.conf
```

`-i` modifies the actual file. During an incident, prefer producing transformed output first and validate it before changing production configuration.

---

## 4. `awk` — field extraction and processing

`awk` is especially useful when logs or command output have columns/fields.

Example:

```text
2026-10-06 09:10:21 ERROR API timeout
```

Fields are commonly referenced as:

- `$1` → first field
- `$2` → second field
- `$3` → third field
- `$0` → entire current line

Basic example:

```bash
awk '{print $1}' app.log
```

### Command anatomy

| Part | Meaning |
|---|---|
| `awk` | text-processing language/tool |
| `{print $1}` | action: print field 1 |
| `$0` | complete current line |
| `$1`, `$2`, ... | first, second, ... field |
| `-F` | set field separator |
| `BEGIN` | run before processing input |
| `END` | run after processing input |
| `~` | matches a regular expression |
| `!~` | does not match a regular expression |
| `>` | greater than |
| `<` | less than |

### Extract a field

```bash
ps -ef | awk '{print $1, $2, $8}'
```

### Custom delimiter

For `/etc/passwd`, fields are separated by `:`:

```bash
awk -F: '{print $1, $7}' /etc/passwd
```

`-F:` means:

> Use `:` as the field separator.

### Filter rows

```bash
awk '$3 > 80 {print $0}' metrics.txt
```

Meaning:

> If field 3 is greater than 80, print the entire line.

### Calculate values

```bash
awk '{sum += $5} END {print sum}' access.log
```

This accumulates field 5 and prints the final total.

---

## 5. `sort` — order results

`sort` orders lines, often after `awk` or `grep` has extracted useful data.

### Command anatomy

| Part | Meaning |
|---|---|
| `sort` | sort lines |
| `-r` | reverse order |
| `-n` | numeric sort |
| `-h` | human-readable numeric sort (`K`, `M`, `G`) |
| `-k` | sort by a specific field/key |
| `-t` | specify field delimiter |
| `-u` | unique output; remove duplicate adjacent sorted lines |
| `-o` | write sorted output to a file |

Example:

```bash
sort -nr numbers.txt
```

Meaning:

- `-n` → numeric comparison
- `-r` → largest/highest first

For human-readable sizes:

```bash
du -h /var/log/* 2>/dev/null | sort -hr
```

This is extremely useful during disk investigations.

---

## 6. `xargs` — turn input into command arguments

`xargs` reads items from standard input and builds command arguments from them.

Simple example:

```bash
printf '%s\n' file1 file2 file3 | xargs ls -l
```

Conceptually it runs something similar to:

```bash
ls -l file1 file2 file3
```

### Command anatomy

| Part | Meaning |
|---|---|
| `xargs` | build and execute commands from standard input |
| `-n N` | use at most N input items per command |
| `-I {}` | replace `{}` with each input item |
| `-0` | read NUL-separated input; safer for filenames with spaces/newlines |
| `-r` | do not run the command when there is no input on GNU xargs |
| `-P N` | run up to N commands in parallel; use carefully in production |

Example:

```bash
printf '%s\n' app1 app2 app3 | xargs -n 1 systemctl status
```

`-n 1` means one input item per command invocation.

### Safer filename handling

For filenames containing spaces or special characters, prefer:

```bash
find /var/log -type f -print0 | xargs -0 ls -lh
```

`-print0` produces NUL-separated names and `xargs -0` reads that format safely.

### Production warning

Never blindly pipe arbitrary command output into a destructive command.

Be especially careful with patterns such as:

```bash
... | xargs rm
```

First inspect the input. During an incident, use `echo`, `printf`, `ls` or another harmless command to verify exactly what will be passed to the next command.

---

## 7. Combining the tools

The real DevOps skill is combining them.

### Find the most common HTTP status codes

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

Typical output:

```text
1200 200
 180 404
  75 500
```

Meaning:

- `awk '{print $9}'` → extract status-code field
- `sort` → group equal values together
- `uniq -c` → count consecutive duplicates
- `sort -nr` → highest count first

### Find top client IPs

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head
```

### Find errors around a specific event

```bash
grep -n -C 5 "ERROR" app.log
```

### Find the largest log files

```bash
du -h /var/log/* 2>/dev/null | sort -hr | head -20
```

### Extract unique failed users

```bash
grep "authentication failed" auth.log | awk '{print $NF}' | sort -u
```

`$NF` means the **last field** in the current `awk` record.

---

## 8. Realistic production failure scenario — API latency and 500 errors

### Situation

Users report that an API is slow and some requests return HTTP 500.

You have an Nginx access log:

```text
10.0.1.20 - - [06/Oct/2026:09:10:01] "GET /api/orders HTTP/1.1" 500 421
10.0.1.21 - - [06/Oct/2026:09:10:02] "GET /api/orders HTTP/1.1" 200 812
10.0.1.20 - - [06/Oct/2026:09:10:03] "GET /api/orders HTTP/1.1" 500 419
```

### Step 1 — Count status codes

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

Suppose you discover a sudden increase in `500` responses.

### Step 2 — Find the affected endpoint

```bash
grep '" 500 ' access.log | awk '{print $7}' | sort | uniq -c | sort -nr
```

Suppose `/api/orders` dominates the failures.

### Step 3 — Find the top client IPs generating failures

```bash
grep '" 500 ' access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head
```

Now you can determine whether failures are widespread or concentrated on a client/source.

### Step 4 — Correlate with application logs

```bash
journalctl -u myapp --since "09:05" --until "09:15" --no-pager | grep -Ei "error|timeout|database|exception"
```

Suppose the application log shows repeated database timeouts.

### Step 5 — Verify the dependency

```bash
ss -lntp | grep 5432
```

Then test connectivity to the database endpoint as appropriate.

### Step 6 — Use `awk` to quantify the evidence

If application logs contain a numeric latency field, extract it and sort it to identify high-latency requests.

The goal is not to run every command. The goal is to move from:

```text
User complaint
   ↓
500 increase
   ↓
Affected endpoint
   ↓
Affected source
   ↓
Application error
   ↓
Database timeout
   ↓
Verify database/network dependency
```

---

## 9. Production-safe habits

### 1. Inspect before modifying

Prefer:

```bash
sed 's/old/new/g' config.conf
```

over immediately using `sed -i` until you understand the output.

### 2. Quote variables

When using shell variables:

```bash
grep -F -- "$pattern" "$file"
```

This reduces surprises from spaces and special characters.

### 3. Be careful with xargs

Before a destructive pipeline, inspect:

```bash
find ... -print0 | xargs -0 printf '%s\n'
```

### 4. Preserve evidence

Do not modify or delete logs while investigating unless there is an approved operational reason.

### 5. Understand delimiters

`awk` and `sort` depend heavily on field separators and keys. Always confirm the input format first.

---

## 10. AWS / Kubernetes connection

### AWS

These commands are useful on EC2 hosts when investigating:

- Nginx/application logs
- deployment failures
- disk usage
- authentication failures
- CloudWatch agent output
- service errors

For AWS API errors, combine OS evidence with AWS-side evidence such as CloudWatch and CloudTrail.

### Kubernetes

The same text-processing skills are useful with Kubernetes logs:

```bash
kubectl logs deployment/myapp | grep -i error
kubectl logs pod/myapp-abc123 --since=30m | grep -Ei 'error|timeout|failed'
```

You can combine output with `awk`, `sort` and other shell tools:

```bash
kubectl logs pod/myapp-abc123 | grep -i timeout | awk '{print $1}' | sort | uniq -c | sort -nr
```

For Kubernetes, remember that `kubectl logs` retrieves container logs; the actual storage/collection path depends on the cluster runtime and logging architecture.

---

## 11. Practical lab

Create a sample access log:

```bash
cat > access.log <<'EOF'
10.0.0.1 - - [06/Oct/2026:09:10:01] "GET /api/orders HTTP/1.1" 200 812
10.0.0.2 - - [06/Oct/2026:09:10:02] "GET /api/orders HTTP/1.1" 500 421
10.0.0.1 - - [06/Oct/2026:09:10:03] "GET /api/users HTTP/1.1" 200 520
10.0.0.2 - - [06/Oct/2026:09:10:04] "GET /api/orders HTTP/1.1" 500 419
10.0.0.3 - - [06/Oct/2026:09:10:05] "GET /api/orders HTTP/1.1" 404 120
10.0.0.2 - - [06/Oct/2026:09:10:06] "GET /api/orders HTTP/1.1" 500 422
EOF
```

Practice:

1. Find all 500 responses.
2. Count status codes.
3. Find the top IP generating 500s.
4. Find the most frequently requested endpoint.
5. Extract unique endpoints.
6. Use `sed` to change `/api/orders` to `/api/v2/orders` in output without modifying the file.
7. Use `xargs` safely with a list of test files.
8. Build one pipeline combining `grep`, `awk`, `sort` and `uniq`.

---

# Interview Questions

<details><summary>1. What is grep used for?</summary>
`grep` searches input for lines matching a pattern. It is commonly used to narrow logs during troubleshooting.
</details>

<details><summary>2. What does grep -i mean?</summary>
`-i` makes the match case-insensitive.
</details>

<details><summary>3. What does grep -v do?</summary>
It inverts the match and displays lines that do not match the pattern.
</details>

<details><summary>4. What is sed?</summary>
`sed` is a stream editor used to search, replace, print and transform text streams.
</details>

<details><summary>5. What does sed 's/old/new/g' mean?</summary>
`s` means substitute, `old` is the search pattern, `new` is the replacement and `g` replaces all matches on each line.
</details>

<details><summary>6. What does sed -n '10,20p' do?</summary>
It suppresses normal output and prints lines 10 through 20.
</details>

<details><summary>7. What is awk mainly used for?</summary>
`awk` is useful for field extraction, filtering, calculations and structured text processing, especially with column-based output.
</details>

<details><summary>8. What are $0 and $1 in awk?</summary>
`$0` is the complete current line and `$1` is the first field. `$2`, `$3`, etc. represent subsequent fields.
</details>

<details><summary>9. What does awk -F: do?</summary>
It sets `:` as the field separator.
</details>

<details><summary>10. What is sort -nr?</summary>
`-n` performs numeric sorting and `-r` reverses the order, producing highest values first.
</details>

<details><summary>11. What is sort -h?</summary>
It performs human-readable numeric sorting, which is useful for values such as `10K`, `2M` and `1G`.
</details>

<details><summary>12. What is xargs?</summary>
`xargs` converts input from standard input into arguments for another command.
</details>

<details><summary>13. Why use xargs -0?</summary>
`-0` tells xargs to expect NUL-separated input, which safely handles filenames containing spaces and many special characters when paired with `find -print0`.
</details>

<details><summary>14. Why is xargs dangerous with rm?</summary>
If the input is wrong, xargs can pass unintended files to a destructive command. Always inspect and validate the input before using destructive operations.
</details>

<details><summary>15. How would you find the top client IPs in an access log?</summary>
A common approach is `awk '{print $1}' access.log | sort | uniq -c | sort -nr | head`.
</details>

<details><summary>16. How would you find the most common HTTP status codes?</summary>
Extract the status field with `awk`, sort it, count with `uniq -c`, then sort the counts numerically in reverse order.
</details>

<details><summary>17. How do you troubleshoot a sudden increase in HTTP 500 errors?</summary>
First quantify the increase, identify affected endpoints/sources, correlate timestamps with application logs, identify the first meaningful application error, and verify the suspected dependency or configuration issue.
</details>

<details><summary>18. What is the difference between filtering and transforming?</summary>
`grep` primarily filters lines. `sed` transforms or selects text. `awk` can both filter and transform structured fields. `sort` orders the result and `xargs` passes results to another command.
</details>

<details><summary>19. How are these commands useful in Kubernetes?</summary>
They can process `kubectl logs` output to find errors, extract fields, count occurrences and identify patterns during pod/application troubleshooting.
</details>

<details><summary>20. What is the most important production lesson when using these commands?</summary>
Understand the input before building the pipeline, validate intermediate results, avoid destructive operations during an incident, and use the output as evidence for a troubleshooting hypothesis rather than blindly executing commands.
</details>

---

## Production mental model

```text
Raw logs / command output
        ↓
      grep       → FIND
        ↓
      sed        → TRANSFORM
        ↓
      awk        → EXTRACT / CALCULATE
        ↓
      sort       → ORDER
        ↓
      xargs      → EXECUTE NEXT COMMAND
        ↓
Useful evidence
        ↓
Root-cause hypothesis
        ↓
Verification
```

**Goal:** do not become someone who only knows Linux commands. Become someone who can take noisy production output and turn it into evidence for a correct diagnosis.