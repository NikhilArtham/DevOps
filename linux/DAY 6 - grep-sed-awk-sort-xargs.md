# DAY 6 - grep, sed, awk, sort & xargs

> **Goal:** Learn five Linux commands that help you search, transform, analyze and process text. These are extremely useful for logs and production troubleshooting.

## 1. Why text-processing commands matter

Production systems generate lots of text:

```text
application logs
system logs
configuration files
CSV/text output
command output
```

Instead of reading thousands of lines manually, Linux gives us small tools that can be combined.

Think:

```text
Large output
   ↓
grep → find what matters
   ↓
sed → change/remove text
   ↓
awk → extract fields/calculate
   ↓
sort → organize results
   ↓
xargs → use results as command arguments
```

## 2. grep = find text

`grep` searches input for matching text.

```bash
grep "ERROR" app.log
```

Case-insensitive:

```bash
grep -i "error" app.log
```

Useful options:

- `-i` = ignore case
- `-n` = show line number
- `-v` = show lines that do NOT match
- `-r` = search directories recursively
- `-E` = extended regular expressions

Example:

```bash
grep -in "timeout" app.log
```

## 3. sed = change text

`sed` is a stream editor. It is commonly used to replace or delete text.

Replace first matching text on each line:

```bash
sed 's/old/new/' file.txt
```

Replace all matches on each line:

```bash
sed 's/old/new/g' file.txt
```

Useful idea:

```text
s = substitute
old = text to find
new = replacement
g = all matches on each line
```

Delete lines containing a pattern:

```bash
sed '/DEBUG/d' app.log
```

`d` = delete the matching line.

## 4. awk = work with columns

`awk` is especially useful when output has fields/columns.

Example:

```text
nginx 200 /login
nginx 500 /payment
```

Print the first field:

```bash
awk '{print $1}' file.txt
```

Print first and third fields:

```bash
awk '{print $1, $3}' file.txt
```

Important variables:

- `$1` = first field
- `$2` = second field
- `$NF` = last field
- `NF` = number of fields
- `NR` = current record/line number

## 5. awk with a delimiter

For CSV-like data:

```bash
awk -F',' '{print $1, $3}' users.csv
```

`-F','` tells awk that comma is the field separator.

## 6. awk calculations

Suppose a log contains:

```text
GET / 120
GET /login 250
GET /api 800
```

Find requests slower than 500 ms:

```bash
awk '$3 > 500 {print $0}' access.log
```

`$3 > 500` is the condition.

## 7. sort = arrange output

```bash
sort file.txt
```

Numerical sorting:

```bash
sort -n numbers.txt
```

Reverse order:

```bash
sort -r file.txt
```

Combine options:

```bash
sort -nr numbers.txt
```

For human-readable sizes:

```bash
sort -hr
```

## 8. xargs = turn input into arguments

Suppose:

```bash
printf '%s\n' file1 file2 file3 | xargs ls -l
```

`xargs` takes input and builds command arguments from it.

A common use:

```bash
find /tmp -name '*.log' -print0 | xargs -0 rm
```

Why `-print0` and `-0`?

They safely handle filenames containing spaces and unusual characters.

## 9. Pipelines

A pipe `|` sends the output of one command into another command.

Example:

```bash
ps aux | grep nginx
```

Meaning:

```text
ps aux
  ↓ output
 grep nginx
```

The power comes from combining simple tools.

## 10. Production example: find HTTP 500 errors

```bash
grep ' 500 ' access.log
```

Count status codes:

```bash
awk '{print $9}' access.log | sort | uniq -c | sort -nr
```

The exact field number depends on the log format, so always inspect a sample line first.

## 11. Production example: find slow requests

If response time is field 10:

```bash
awk '$10 > 1000 {print $0}' access.log
```

Then sort by that field:

```bash
awk '$10 > 1000 {print $0}' access.log | sort -k10 -nr
```

`-k10` tells `sort` to sort using field 10.

## 12. Production troubleshooting mindset

Do not blindly paste pipelines.

Build them one command at a time:

```bash
cat access.log | head
```

then:

```bash
grep ' 500 ' access.log
```

then:

```bash
grep ' 500 ' access.log | awk '{print $7}'
```

then sort/count the result.

This makes debugging much easier.

## 13. AWS and Kubernetes connection

These commands are useful with output from:

```bash
aws ...
kubectl logs ...
docker logs ...
journalctl ...
```

Example:

```bash
kubectl logs deployment/myapp | grep -i error
```

Or:

```bash
journalctl -u myapp --since "30 min ago" | grep -i timeout
```

## 14. Common mistakes

### Assuming awk field numbers

Different log formats have different fields.

### Forgetting case sensitivity

Use `grep -i` when appropriate.

### Unsafe `xargs`

Use null-delimited input (`-print0 | xargs -0`) when filenames may contain spaces.

### Editing production files blindly with sed

First preview the result without `-i`. Only edit in place after validating the change.

## 15. Commands to remember

```bash
grep -in "error" app.log
sed 's/old/new/g' file.txt
awk '{print $1}' file.txt
awk -F',' '{print $1}' file.csv
sort -nr numbers.txt
find /tmp -type f -print0 | xargs -0 ls -l
```

### Beginner takeaway

Remember the jobs:

- **grep** → find
- **sed** → change text
- **awk** → understand fields/data
- **sort** → arrange
- **xargs** → turn input into command arguments

> Labs and interview questions are maintained centrally in `linux/LABS.md` and `linux/INTERVIEW QUESTIONS.md`.