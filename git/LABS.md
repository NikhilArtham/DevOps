# Git Labs

Centralized practical labs for Git. Labs from every Git DAY are added here.

---

## DAY 1 — Git Internals: Objects, HEAD, Refs and Reflog

### Lab 1 — Create a Repository and Inspect `.git`

```bash
mkdir git-internals-lab
cd git-internals-lab
git init
ls -la
find .git -maxdepth 2 -type f | sort
```

**Goal:** Identify `HEAD`, `objects`, `refs`, and `logs`.

---

### Lab 2 — Create a Commit and Inspect Its Object

```bash
echo 'hello DevOps' > app.txt
git add app.txt
git commit -m 'Add app file'
git rev-parse HEAD
git cat-file -t HEAD
git cat-file -p HEAD
```

**Goal:** See that HEAD resolves to a commit object and inspect its tree and parent information.

---

### Lab 3 — Follow Commit → Tree → Blob

Get the tree from:

```bash
git cat-file -p HEAD
```

Then inspect the tree:

```bash
git cat-file -p <tree-id>
```

Find the blob ID for `app.txt` and inspect it:

```bash
git cat-file -t <blob-id>
git cat-file -p <blob-id>
```

**Goal:** Understand:

```text
commit → tree → blob
```

---

### Lab 4 — Inspect HEAD and Branch Refs

```bash
cat .git/HEAD
git branch --show-current
git rev-parse HEAD
git rev-parse --abbrev-ref HEAD
git show-ref
```

**Goal:** Understand:

```text
HEAD → branch → commit
```

---

### Lab 5 — Observe a Detached HEAD

Create another commit, then inspect an older commit:

```bash
git log --oneline
git checkout <older-commit>
cat .git/HEAD
git status
git rev-parse HEAD
```

Create a recovery branch immediately if you make work while detached:

```bash
git switch -c detached-work
```

**Goal:** Understand the difference between a branch-attached HEAD and detached HEAD.

---

### Lab 6 — Compare `git log` and `git reflog`

```bash
git log --oneline --decorate --graph --all
git reflog
```

Then make a new commit and move the branch backward:

```bash
git reset --hard HEAD~1
git log --oneline
git reflog
```

**Goal:** Observe that `git log` follows current reachable history while reflog records local ref movement.

---

### Lab 7 — Recover an Accidentally Reset Commit

Create two commits, then run:

```bash
git reset --hard HEAD~2
git reflog
```

Find the old HEAD and inspect it:

```bash
git show <old-commit>
```

Recover safely:

```bash
git switch -c recovery <old-commit>
```

**Goal:** Practice the production recovery pattern without guessing.

---

### Lab 8 — Find Dangling Objects

After making commits and moving refs, run:

```bash
git fsck --no-reflogs
```

Inspect any candidate dangling commit:

```bash
git show <commit-id>
```

If it is important:

```bash
git switch -c recovered-work <commit-id>
```

**Goal:** Understand that an object can become unreachable from normal refs while still existing temporarily.

---

### DAY 1 Capstone — Git Recovery Incident

Simulate:

```text
A → B → C → D
```

Then accidentally move the branch back to `B`.

Produce an incident note:

```text
Current branch:
Current HEAD:
Old HEAD from reflog:
Recovered commit:
Object type:
Branch/ref relationship:
Recovery command:
Why git log alone was insufficient:
Why reflog helped:
```

**Success criteria:** Explain and demonstrate how Git objects, refs, HEAD, and reflog work together to recover lost-looking work.
