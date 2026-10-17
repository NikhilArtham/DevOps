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

**Goal:** Observe that `git log` follows current reachable history while reflog records ref movement.

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

---

## DAY 2 — Git Branching, Merge vs Rebase and Conflict Recovery

### Lab 1 — Create and Visualize Branches

```bash
mkdir git-branching-lab
cd git-branching-lab
git init
echo "base" > app.txt
git add app.txt
git commit -m "Initial commit"
git switch -c feature/payment
echo "payment" >> app.txt
git add app.txt
git commit -m "Add payment feature"
git log --oneline --decorate --graph --all
```

**Goal:** See that a branch is a movable ref pointing to a commit.

---

### Lab 2 — Fast-Forward Merge

```bash
git switch main
git merge feature/payment
git log --oneline --decorate --graph --all
```

**Goal:** Observe a fast-forward merge where Git only moves the `main` ref forward.

---

### Lab 3 — Three-Way Merge

Create two branches from the same base and make independent changes:

```bash
git switch main
git switch -c feature-a
echo "line-a" >> app.txt
git add app.txt
git commit -m "Feature A"

git switch main
git switch -c feature-b
echo "line-b" >> app.txt
git add app.txt
git commit -m "Feature B"
```

Merge one feature into `main`, then the other:

```bash
git switch main
git merge feature-a
git merge feature-b
git log --oneline --decorate --graph --all
```

**Goal:** Understand when Git creates a merge commit because both histories moved independently.

---

### Lab 4 — Rebase a Feature Branch

Create divergent history:

```bash
git switch main
git switch -c feature/rebase
echo "feature" >> app.txt
git add app.txt
git commit -m "Feature change"

git switch main
echo "main" >> app.txt
git add app.txt
git commit -m "Main change"
```

Rebase:

```bash
git switch feature/rebase
git rebase main
git log --oneline --decorate --graph --all
```

**Goal:** Observe that the feature commit is replayed on the new base and receives a new commit ID.

---

### Lab 5 — Trigger and Resolve a Merge Conflict

Create conflicting changes to the same line:

```bash
git switch main
git switch -c feature/conflict
echo "port=9090" > config.txt
git add config.txt
git commit -m "Use port 9090"

git switch main
echo "port=8080" > config.txt
git add config.txt
git commit -m "Use port 8080"
```

Merge:

```bash
git merge feature/conflict
git status
git diff
```

Edit `config.txt` into the correct final state, remove all conflict markers, then:

```bash
git add config.txt
git commit
```

**Goal:** Practice the complete merge conflict lifecycle.

---

### Lab 6 — Resolve a Rebase Conflict

Create conflicting commits, then run:

```bash
git rebase main
```

If Git stops:

```bash
git status
git diff
git add <resolved-file>
git rebase --continue
```

If the operation should be abandoned:

```bash
git rebase --abort
```

**Goal:** Understand that rebase may stop repeatedly because commits are replayed one at a time.

---

### Lab 7 — Inspect Branch Divergence

```bash
git log --oneline --decorate --graph --all
git merge-base main feature/payment
git log --oneline main..feature/payment
git log --oneline feature/payment..main
git diff main...feature/payment
```

**Goal:** Identify the common ancestor, commits unique to each side, and the net change introduced by the feature branch.

---

### Lab 8 — Safe Recovery After a Rebase Mistake

Before a destructive rebase, record the current state:

```bash
git branch backup-before-rebase
git reflog
```

If the result is wrong, inspect:

```bash
git reflog
git log --oneline --all --decorate
```

Create a recovery ref if needed:

```bash
git switch -c recovery/rebase <known-good-commit>
```

**Goal:** Combine DAY 1 reflog knowledge with DAY 2 history rewriting.

---

### DAY 2 Capstone — Production Configuration Conflict

Simulate a feature branch and `main` changing the same deployment configuration. Rebase the feature onto `main`, intentionally create a conflict, and recover it.

Incident note:

```text
Current branch:
Operation: merge or rebase:
Conflict file:
What both sides changed:
Correct application behavior:
Resolution chosen:
Validation performed:
Continue command:
Abort command:
Final commit graph:
Push command if history was rewritten:
Why --force-with-lease is safer than --force:
```

**Success criteria:** Resolve the conflict correctly, validate the resulting configuration, explain merge vs rebase, and recover safely if the operation is wrong.

---

## DAY 3 — Pull Requests, Code Review and Clean Commit History

### Lab 1 — Create a PR-Ready Branch

```bash
git switch main
git pull --ff-only origin main
git switch -c feature/payment-health-check
```

Make a small change and create a meaningful commit:

```bash
git add .
git commit -m "Add payment health check"
git push -u origin feature/payment-health-check
```

**Goal:** Practice the basic branch → commit → push → PR flow.

---

### Lab 2 — Inspect What a PR Will Change

Before opening a PR, inspect the branch locally:

```bash
git status -sb
git log --oneline main..HEAD
git diff main...HEAD
git log --oneline --decorate --graph --all
```

**Goal:** Know exactly which commits and final changes you are asking reviewers to approve.

---

### Lab 3 — Practice Review-Quality Commits

Create several commits such as:

```text
Add payment health check
fix typo
fix review comment
fix test
```

Then inspect:

```bash
git log --oneline -n 10
```

Discuss whether each commit represents a meaningful logical change.

**Goal:** Learn to distinguish useful commits from review noise.

---

### Lab 4 — Squash Review Noise with Interactive Rebase

```bash
git rebase -i HEAD~4
```

Example:

```text
pick   A Add payment health check
fixup  B fix typo
fixup  C fix review comment
fixup  D fix test
```

Save and inspect:

```bash
git log --oneline --decorate --graph
```

**Goal:** Turn review-fix commits into one logical commit.

---

### Lab 5 — Reword a Commit

```bash
git rebase -i HEAD~3
```

Change one `pick` to:

```text
reword
```

Enter a better message and verify:

```bash
git log --oneline -n 5
```

**Goal:** Practice improving commit messages without changing the actual file content.

---

### Lab 6 — Amend the Latest Commit

```bash
echo "extra validation" >> app.txt
git add app.txt
git commit --amend -m "Add payment health check"
```

Inspect:

```bash
git log --oneline -n 3
```

**Goal:** Understand how `--amend` changes the latest commit and therefore its commit ID.

---

### Lab 7 — Review an Infrastructure Diff

Create a sample Terraform or Kubernetes change and inspect only the final diff:

```bash
git fetch origin
git diff origin/main...HEAD
```

For Terraform, perform the repository's normal validation/plan workflow. For Kubernetes, validate the manifest using the tools available in your environment.

Review specifically for:

```text
IAM permissions
Security Groups
replicas
resource limits
ports
health checks
selectors
Route 53 records
ALB rules
```

**Goal:** Practice reviewing operational impact rather than only syntax.

---

### Lab 8 — Branch Protection / PR Safety Exercise

Using a test repository or an existing non-production repository, configure or simulate rules such as:

```text
PR required
approval required
status checks required
no direct push to main
```

Then attempt a direct change to `main` and observe the repository policy.

**Goal:** Understand how repository controls turn the PR process into an enforceable safety gate.

---

### DAY 3 Capstone — Dangerous IAM Change Hidden by a Clean History

Simulate a PR with several messy commits, then squash them into one clean commit. Introduce an overly broad IAM permission during the exercise.

Before merge, use:

```bash
git fetch origin
git diff origin/main...HEAD
```

Identify the dangerous permission, fix it, run:

```bash
terraform fmt -check
terraform validate
terraform plan
```

Then document:

```text
PR purpose:
Source branch:
Base branch:
Original commits:
Cleaned commits:
Risky change found:
Why commit history alone was insufficient:
Final diff reviewed:
Validation performed:
Reviewer checks required:
```

**Success criteria:** Demonstrate that a clean history improves reviewability but the final diff, CI checks, and operational impact still determine whether a PR is safe.
