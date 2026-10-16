# DAY 2 — Git Branching, Merge vs Rebase and Conflict Recovery

## Why This Topic Matters

Branches are one of the most important Git concepts in DevOps because teams use them to develop features, fix production issues, review code, and safely integrate changes.

The three ideas you must understand are:

```text
branching
   ↓
parallel development
   ↓
merge OR rebase
   ↓
conflict resolution when Git cannot combine changes automatically
```

A useful mental model:

> **Branch = movable ref. Merge combines histories. Rebase replays commits on a new base. Conflict recovery means resolving the file correctly, then continuing or aborting the operation safely.**

---

# 1. What Is a Branch?

A branch is a movable reference pointing to a commit.

Imagine:

```text
A → B → C
        ↑
       main
```

Create a feature branch:

```bash
git switch -c feature/payment
```

Now:

```text
A → B → C
        ↑
       main
        
        ↑
   feature/payment
```

After a new feature commit:

```text
A → B → C → D
        ↑     ↑
       main  feature/payment
```

The branches are simply names pointing at different commits.

---

# 2. Creating and Switching Branches

Modern Git recommends `git switch` for branch switching.

## Create a branch

```bash
git switch -c feature/payment
```

`-c` means **create a new branch and switch to it**.

## Switch to an existing branch

```bash
git switch main
```

## List branches

```bash
git branch
```

Useful options:

```bash
git branch -a
```

- `-a` → show local and remote-tracking branches.

```bash
git branch -vv
```

- `-vv` → show upstream/tracking information and the relationship between local branches and their upstream branches.

---

# 3. Creating a Branch from a Specific Commit

A branch can start from any valid commit:

```bash
git switch -c recovery/payment <commit-id>
```

This is useful during recovery because it creates a new ref without immediately rewriting an existing branch.

---

# 4. Local Branch vs Remote-Tracking Branch

These are different:

```text
main
```

is your local branch.

```text
origin/main
```

is a remote-tracking ref representing the last fetched state of the remote branch.

Update remote-tracking refs with:

```bash
git fetch origin
```

`fetch` downloads remote information without automatically merging it into your current branch.

Inspect the difference with:

```bash
git log --oneline --decorate --graph --all
```

or:

```bash
git diff main origin/main
```

---

# 5. Fast-Forward Merge

Suppose:

```text
A → B → C
        ↑
       main
```

The feature branch contains new commits:

```text
A → B → C → D → E
        ↑         ↑
       main     feature
```

If `main` has not moved, Git can simply move the `main` ref forward:

```bash
git switch main
git merge feature
```

Result:

```text
A → B → C → D → E
                  ↑
                 main
```

This is called a **fast-forward merge** because Git does not need to create a merge commit.

---

# 6. Three-Way Merge

Now imagine both branches changed after splitting:

```text
        D → E   feature
       /
A → B → C
       \
        F → G   main
```

Git cannot simply move one branch forward because both sides have unique commits.

A normal merge can create a new merge commit:

```text
        D → E
       /     \
A → B → C       M
       \
        F → G /
```

Run:

```bash
git switch main
git merge feature
```

The merge commit records the integration of both histories.

A merge commit normally has two parents:

```text
parent 1 → previous main history
parent 2 → feature history
```

---

# 7. What Does `git merge` Actually Do?

Conceptually, Git finds a common ancestor and compares the changes made on both sides.

For example:

```text
          feature
             ↓
        D → E
       /
A → B → C
       \
        F → G
             ↑
            main
```

Git uses the common history around `B` to determine what changed on each side.

If the changes can be combined automatically, Git completes the merge.

If the same part of a file was changed differently, Git reports a conflict.

---

# 8. What Is Rebase?

Rebase changes the base of your commits.

Suppose:

```text
        D → E   feature
       /
A → B → C
       \
        F → G   main
```

If you rebase the feature branch onto `main`, Git conceptually takes the feature commits and replays them on top of the latest main history:

```text
A → B → C → F → G → D' → E'
                       ↑     ↑
                    replayed feature commits
```

The important point is:

> **Rebase creates new commit IDs for the replayed commits.**

Even if the file changes look identical, `D'` and `E'` are not the original `D` and `E` commits.

Command:

```bash
git switch feature
git rebase main
```

---

# 9. Merge vs Rebase

## Merge

```text
feature ── D ── E
          /     \
main ─ A ─ B ─ C ─ M
```

Advantages:

- Preserves the original branch topology.
- Does not rewrite existing commits.
- Useful when preserving the exact historical relationship matters.

Disadvantages:

- Can create additional merge commits.
- Long-lived branches can make history harder to read.

## Rebase

```text
main ─ A ─ B ─ C ─ F ─ G ─ D' ─ E'
```

Advantages:

- Produces a more linear history.
- Makes the feature branch appear to have started from the latest base.
- Can make `git log --graph` easier to read.

Disadvantages:

- Rewrites commit IDs.
- Dangerous when rewriting commits that other people already depend on.

---

# 10. Golden Rule of Rebase

The most important practical rule is:

> **Do not casually rebase commits that have already been shared and are being used by other people.**

Why?

Because rebase creates replacement commits.

Before:

```text
A → B → C
```

After rebasing:

```text
A → B → C'
```

`C'` is a different commit object.

If other people already based work on `C`, rewriting it can create confusing history and require force-pushing.

A common safe pattern is:

```text
rebase your private feature branch
        ↓
resolve/test
        ↓
push for review
```

Be especially careful with shared branches such as `main`.

---

# 11. `git pull --rebase`

A common DevOps workflow is:

```bash
git pull --rebase origin main
```

Conceptually this means:

```text
fetch remote changes
       ↓
replay your local commits on top of the updated remote branch
```

It can keep a local development history linear.

Always understand the repository's team policy before choosing merge or rebase automatically.

---

# 12. Conflict — What Does It Mean?

A conflict occurs when Git cannot safely decide how to combine changes.

A common example:

`main` changes:

```text
port=8080
```

a feature branch changes the same line to:

```text
port=9090
```

If both histories changed the same area differently, Git may stop and ask you to decide the correct final content.

Conflict markers can look like:

```text
<<<<<<< HEAD
port=8080
=======
port=9090
>>>>>>> feature/payment
```

Meaning:

```text
<<<<<<< HEAD
current branch version

=======
separator

>>>>>>> feature/payment
incoming branch version
```

Do not blindly choose the top or bottom section. Decide what the application actually needs.

---

# 13. The Conflict Recovery Workflow

The general workflow is:

```text
start merge/rebase
      ↓
conflict
      ↓
git status
      ↓
inspect conflicted files
      ↓
edit files
      ↓
remove conflict markers
      ↓
git add <resolved-file>
      ↓
continue operation
```

For a merge:

```bash
git add <file>
git commit
```

For a rebase:

```bash
git add <file>
git rebase --continue
```

---

# 14. Check Conflict State with `git status`

When Git reports a conflict, do not guess.

Run:

```bash
git status
```

It tells you whether you are in a merge or rebase and which files need attention.

Useful command:

```bash
git diff
```

It shows unresolved changes and conflict markers in the working tree.

You can also inspect the staged result with:

```bash
git diff --cached
```

---

# 15. Marking a Conflict as Resolved

After editing the file into the correct final state:

```bash
git add <file>
```

`git add` does not mean “Git automatically fixed the conflict.”

It means:

> **I have reviewed this file and staged the version I want to keep.**

Then continue the operation.

Merge:

```bash
git commit
```

Rebase:

```bash
git rebase --continue
```

---

# 16. Abort a Merge

If you started a merge and realize you need to return to the state before the merge:

```bash
git merge --abort
```

This is useful when the merge decision is unclear or you want to restart after investigating.

Before aborting, understand that you should not treat `--abort` as a substitute for understanding what local changes you had before the operation.

---

# 17. Abort a Rebase

If a rebase becomes confusing or you decide the operation should not continue:

```bash
git rebase --abort
```

This attempts to return the branch to the state it had before the rebase began.

Useful related commands:

```bash
git rebase --continue
git rebase --skip
git rebase --abort
```

Meanings:

- `--continue` → continue after resolving the current conflict.
- `--skip` → skip the current commit being replayed. Use only when you intentionally want to omit that commit.
- `--abort` → cancel the rebase and return to the pre-rebase state when possible.

---

# 18. `ours` and `theirs` — Be Careful

During conflict resolution you may encounter:

```bash
git checkout --ours <file>
git checkout --theirs <file>
```

or newer workflows using `git restore` with appropriate options.

These commands can replace the file with one side's version.

Do not use them blindly.

In a rebase, the meaning of “ours” and “theirs” can be surprising because Git is replaying a commit onto another base.

Always inspect the conflict and understand which side represents the desired application state before choosing one side wholesale.

---

# 19. Merge Conflict vs Rebase Conflict

A conflict can happen in both operations, but the mental model differs.

### Merge

You are combining two histories:

```text
main + feature
      ↓
   merge
      ↓
possible conflict
```

Resolve the combined result and commit the merge.

### Rebase

You are replaying commits one at a time:

```text
commit D
   ↓
apply onto new base
   ↓
conflict?
   ↓
resolve
   ↓
rebase --continue
```

A rebase may therefore stop multiple times if several commits conflict.

---

# 20. Useful Branch Visualization

When debugging branch history, use:

```bash
git log --oneline --decorate --graph --all
```

For a more complete view:

```bash
git log --oneline --decorate --graph --all --reflog
```

You can inspect branch divergence with:

```bash
git merge-base main feature/payment
```

This finds the common ancestor used as the merge base.

Check commits that are on one side but not the other:

```bash
git log --oneline main..feature/payment
```

This shows commits reachable from `feature/payment` but not from `main`.

Reverse it:

```bash
git log --oneline feature/payment..main
```

This shows commits reachable from `main` but not from `feature/payment`.

---

# 21. Practical Branching Workflow for DevOps

A simple feature workflow:

```bash
git switch main
git pull --ff-only origin main
git switch -c feature/payment
```

Make changes:

```bash
git add .
git commit -m "Add payment health check"
```

Before opening or updating a PR, bring the branch up to date according to team policy.

Rebase workflow:

```bash
git fetch origin
git rebase origin/main
```

Or merge workflow:

```bash
git fetch origin
git merge origin/main
```

Then run tests and push the branch.

The exact policy should follow the team's repository rules.

---

# 22. AWS / DevOps Connection

Git itself is not an AWS service, but Git branching and integration are directly connected to DevOps delivery workflows.

A typical path is:

```text
feature branch
      ↓
Pull Request
      ↓
CI checks
      ↓
merge/rebase policy
      ↓
main
      ↓
CI/CD pipeline
      ↓
AWS deployment
```

Examples of AWS resources that may be changed through Git-driven automation include:

```text
Terraform
CloudFormation
ECS task definitions
EKS manifests
IAM policies
ALB configuration
Route 53 records
Lambda configuration
```

A bad merge can therefore become a deployment incident even though the original problem is only a Git conflict.

---

# 23. Realistic Failure Scenario — Rebase Conflict Before Production Deployment

### Incident

You have a feature branch:

```text
feature/payment-health-check
```

`main` changed the deployment configuration while your branch changed the same configuration.

You run:

```bash
git fetch origin
git rebase origin/main
```

Git stops with a conflict.

### Step 1 — Stop and inspect

```bash
git status
git diff
```

Do not immediately run `git add .`.

### Step 2 — Identify the affected file

Suppose the conflict is in:

```text
k8s/deployment.yaml
```

You inspect the conflict markers and understand both changes.

For example, `main` changed the container port while your branch changed the health-check path.

The correct production configuration may need both changes.

### Step 3 — Edit the file

Produce one valid final configuration and remove:

```text
<<<<<<<
=======
>>>>>>>
```

Then validate the configuration before staging it.

For Kubernetes manifests, use the team's available validation/linting tools. For Terraform, run appropriate formatting/validation/plan checks.

### Step 4 — Stage the resolved file

```bash
git add k8s/deployment.yaml
```

Check:

```bash
git status
```

### Step 5 — Continue the rebase

```bash
git rebase --continue
```

If another commit conflicts, repeat the same process.

### Step 6 — Verify history and changes

```bash
git log --oneline --decorate --graph --all
git diff origin/main...HEAD
```

Run tests and deployment validation.

### Step 7 — Push carefully

Because rebase rewrites the feature branch's commit IDs, a previously pushed branch may require a force update.

Prefer:

```bash
git push --force-with-lease origin feature/payment-health-check
```

rather than:

```bash
git push --force
```

`--force-with-lease` provides a safety check that the remote branch has not moved unexpectedly.

### Key lesson

```text
Rebase conflict
      ↓
inspect
      ↓
understand application intent
      ↓
resolve
      ↓
validate
      ↓
git add
      ↓
git rebase --continue
      ↓
test
      ↓
force-with-lease only when required
```

---

# 24. Common Mistakes

### Mistake 1 — Thinking merge and rebase are identical

They both integrate work, but they produce different history.

### Mistake 2 — Rebasing shared branches casually

Rebase rewrites commit IDs.

### Mistake 3 — Running `git add .` without checking conflicts

You may accidentally stage an incorrect resolution.

### Mistake 4 — Choosing “ours” or “theirs” without understanding the operation

The meaning can be especially confusing during rebase.

### Mistake 5 — Using `git push --force` by default

Prefer:

```bash
git push --force-with-lease
```

when a rewritten remote branch genuinely needs a force update.

### Mistake 6 — Solving a Git conflict without validating the application

A conflict can be syntactically resolved but operationally wrong.

Always test configuration and application behavior after resolution.

### Mistake 7 — Treating conflict markers as harmless

Never commit unresolved markers such as:

```text
<<<<<<<
=======
>>>>>>>
```

---

# 25. Interview-Ready Mental Model

If asked **“Merge vs rebase?”**:

> Merge combines two lines of development and can create a merge commit. Rebase replays commits on top of a new base and creates new commit IDs. Merge preserves the existing topology; rebase produces a more linear history but rewrites commits.

If asked **“How do you recover from a conflict?”**:

```text
git status
   ↓
inspect conflict
   ↓
edit correct final state
   ↓
git add <file>
   ↓
merge: git commit
OR
rebase: git rebase --continue
```

If the operation should be abandoned:

```bash
git merge --abort
git rebase --abort
```

If a rebased branch must be pushed:

```bash
git push --force-with-lease
```

---

# Beginner Takeaway

Remember these four things:

```text
BRANCH
→ movable ref pointing to a commit

MERGE
→ combine histories, optionally creating a merge commit

REBASE
→ replay commits on a new base and rewrite commit IDs

CONFLICT RECOVERY
→ inspect → resolve → stage → continue OR abort
```

For production DevOps work, the most important rule is:

> **Never resolve a Git conflict only to make Git happy. Resolve it according to the intended application behavior, then validate before deployment.**
