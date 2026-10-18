# DAY 4 — Git Reset, Revert, Cherry-pick and Recovery Lab

## 1. Why this topic matters

In real DevOps work, Git mistakes can affect application code, Terraform, Kubernetes manifests, CI/CD pipelines, and deployment configuration.

You need to know the difference between:

- moving a branch pointer (`reset`)
- creating a new commit that undoes a change (`revert`)
- copying one commit onto another branch (`cherry-pick`)
- recovering work safely after a mistake (`reflog`, refs, recovery branches)

The most important rule is:

> **Before changing history, understand whether the commit is already shared.**

If history is already on a shared/protected branch, `git revert` is usually safer than rewriting it with `git reset`.

---

## 2. `git reset` — move the current branch

`git reset` changes where the current branch points.

Example history:

```text
A → B → C → D
        ↑
      main
```

After:

```bash
git reset --hard B
```

the branch becomes:

```text
A → B
    ↑
  main

C → D
```

The old commits may still be recoverable through the reflog for some time, but they are no longer part of the current branch history.

### Reset modes

#### `--soft`

```bash
git reset --soft HEAD~1
```

Moves the branch pointer but keeps the changes from the removed commit staged.

Useful when you want to redo or combine commits while keeping the changes ready to commit.

#### `--mixed`

```bash
git reset --mixed HEAD~1
```

Moves the branch pointer and unstages the changes, but keeps the working-tree files changed.

`--mixed` is the default mode.

#### `--hard`

```bash
git reset --hard HEAD~1
```

Moves the branch pointer and makes the working tree/index match the target commit.

**Danger:** local uncommitted changes can be discarded, and the branch history is rewritten.

---

## 3. Understand reset using three areas

A useful beginner mental model is:

```text
Working tree → files you are editing
Index/staging area → changes prepared for commit
HEAD/branch → current committed history
```

The reset modes affect these areas differently:

```text
                    HEAD/ref   Index   Working tree
--soft                move       keep      keep
--mixed               move       reset     keep
--hard                move       reset     reset
```

This is why `--soft`, `--mixed`, and `--hard` are not three different kinds of undo. They differ in what they do to the index and working tree after moving the branch.

---

## 4. `git revert` — undo with a new commit

`git revert` does not move the branch backward.

Suppose:

```text
A → B → C
        ↑
      main
```

If `C` introduced a bad configuration:

```bash
git revert C
```

Git creates a new commit:

```text
A → B → C → R
             ↑
           main
```

`R` contains the inverse of `C`.

### Why revert is safer for shared branches

The existing history remains intact. Other developers and CI systems do not suddenly see the branch pointer move backward.

For a production branch, this is often the safer rollback pattern:

```bash
git switch main
git pull --ff-only origin main
git revert <bad-commit>
git push origin main
```

In a protected repository, the revert may instead be made through a pull request.

---

## 5. `git revert` a merge commit

Reverting a merge commit requires Git to know which parent represents the mainline.

Example:

```bash
git revert -m 1 <merge-commit>
```

`-m 1` means parent 1 is treated as the mainline parent.

Do not blindly use `-m 1`. First inspect the merge:

```bash
git show --summary <merge-commit>
git log --graph --oneline --decorate --all
```

Then determine which parent represents the branch you want to preserve.

---

## 6. `git cherry-pick` — copy one commit

Cherry-pick applies the changes introduced by an existing commit onto your current branch.

Example:

```text
main:     A → B
               
hotfix:   A → B → H
```

If `H` fixes a production bug and you also need that fix on another release branch:

```bash
git switch release
 git cherry-pick H
```

Result:

```text
main:     A → B → H

release:  A → B → H'
```

`H'` has the same logical change but normally a different commit ID because its parent is different.

### Useful cherry-pick commands

```bash
git cherry-pick <commit>
git cherry-pick <commit1> <commit2>
git cherry-pick <old>..<new>
git cherry-pick --no-commit <commit>
```

`--no-commit` applies the changes without immediately creating the commit, allowing you to inspect or combine them first.

---

## 7. Cherry-pick conflict recovery

A cherry-pick can conflict just like merge or rebase.

Start with:

```bash
git status
git diff
```

Resolve the files, then:

```bash
git add <resolved-file>
git cherry-pick --continue
```

If you decide the cherry-pick should not happen:

```bash
git cherry-pick --abort
```

If the selected commit becomes empty because the change already exists:

```bash
git cherry-pick --skip
```

Do not use `--skip` automatically. Confirm that the intended change is already present.

---

## 8. Reset vs revert vs cherry-pick

| Command | Main purpose | Rewrites existing branch history? | Typical shared-branch use |
|---|---|---:|---|
| `reset` | Move branch to another commit | Yes | Usually avoid on shared branches |
| `revert` | Undo a commit with a new commit | No | Safe rollback pattern |
| `cherry-pick` | Copy a commit's change | No, but creates a new commit | Targeted hotfix/backport |

Simple interview answer:

> **Reset moves a ref, revert adds an inverse commit, and cherry-pick copies a commit's change onto the current branch.**

---

## 9. Recovery after a bad reset

Suppose production work was committed:

```text
A → B → C → D
```

Then someone runs:

```bash
git reset --hard HEAD~2
```

Now the visible branch shows:

```text
A → B
    ↑
  main
```

First, **do not immediately run more destructive commands**.

Check:

```bash
git status
git reflog
```

Find the old `HEAD` position:

```bash
git reflog
```

Inspect the candidate:

```bash
git show <recovered-commit>
```

Create a temporary recovery ref:

```bash
git switch -c recovery/bad-reset <recovered-commit>
```

Compare it with the current branch:

```bash
git log --oneline --decorate --graph --all
git diff main...recovery/bad-reset
```

Only after validation should you decide whether to move the original branch back.

---

## 10. Recovery after a bad cherry-pick

If a cherry-pick is still in progress:

```bash
git status
git cherry-pick --abort
```

If the cherry-pick already created a bad commit on a shared branch, prefer:

```bash
git revert <cherry-picked-commit>
```

If it is a private local branch and you deliberately want to rewrite it, reset may be appropriate—but understand the consequences first.

---

## 11. AWS and Kubernetes connection

These Git operations directly affect DevOps configuration stored in repositories.

### Terraform

A bad commit might change:

```text
IAM policy
Security Group
ALB listener
Route 53 record
EC2 settings
EKS configuration
Terraform module variables
```

A production rollback may be implemented by reverting the bad infrastructure commit and then running the normal CI/CD pipeline.

### Kubernetes

A bad commit might change:

```text
Deployment image
replicas
Service port
Ingress/Gateway rule
ConfigMap
Secret reference
readinessProbe
resource limits
```

A targeted production fix may be cherry-picked from a tested hotfix branch into the release branch.

The important principle is:

```text
Git operation → CI/CD pipeline → infrastructure/application change
```

Git history is therefore part of the operational safety system.

---

## 12. Realistic production failure scenario

### Incident: Bad Kubernetes deployment reaches production

A deployment commit changes:

```yaml
image: payments:v2.4.0
```

to a broken image:

```yaml
image: payments:v2.4.1
```

The production deployment succeeds technically, but the pods fail readiness checks.

### Incorrect response

Someone runs:

```bash
git reset --hard HEAD~1
git push --force
```

This rewrites the shared production branch and can create problems for other developers and CI/CD.

### Safer response

Inspect the bad commit:

```bash
git show <bad-commit>
```

Create an inverse commit:

```bash
git revert <bad-commit>
```

Validate the Kubernetes change:

```bash
kubectl diff -f deployment.yaml
kubectl apply --dry-run=server -f deployment.yaml
```

Then use the repository's normal PR/CI/CD process.

### Separate hotfix scenario

A security fix exists as commit `H` on a development branch, but production currently needs only that fix.

Instead of merging unrelated development work:

```bash
git switch production-release
git cherry-pick H
```

Resolve and validate any conflict, then run the normal CI/CD pipeline.

---

## 13. Production decision tree

When you need to undo or move a Git change, ask:

```text
Is the commit already shared?
        |
   +----+----+
   |         |
  NO        YES
   |         |
Need to    Need safe
rewrite?   shared rollback?
   |         |
 reset      revert

Need one specific existing commit on another branch?
        |
     cherry-pick
```

Before any destructive operation:

```bash
git status
git branch --show-current
git log --oneline --decorate --graph -n 10
git reflog -n 10
```

For risky work, create a backup ref first:

```bash
git branch backup-before-reset
```

---

## 14. Common mistakes

### Mistake 1 — Using `reset --hard` on shared `main`

Why it is risky: it rewrites the branch's visible history and can discard uncommitted work.

Prefer `git revert` for a shared rollback.

### Mistake 2 — Thinking revert deletes the bad commit

It does not. The bad commit remains in history; a new commit reverses its changes.

### Mistake 3 — Thinking cherry-pick moves a commit

It does not move the original commit. It applies the change and normally creates a new commit on the current branch.

### Mistake 4 — Cherry-picking without checking dependencies

A commit may depend on earlier commits. A clean cherry-pick does not guarantee that the application behavior is correct.

### Mistake 5 — Continuing a conflict without validating the final file

Always inspect the resolved configuration and run the appropriate tests, Terraform validation/plan, or Kubernetes validation.

### Mistake 6 — Panicking after a reset

First inspect reflog. Recovery often starts by finding the previous HEAD and creating a recovery branch.

---

## 15. Useful command cheat sheet

### Reset

```bash
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1
```

### Revert

```bash
git revert <commit>
git revert -m 1 <merge-commit>
```

### Cherry-pick

```bash
git cherry-pick <commit>
git cherry-pick --no-commit <commit>
git cherry-pick --continue
git cherry-pick --abort
git cherry-pick --skip
```

### Recovery

```bash
git reflog
git show <commit>
git branch recovery/<name> <commit>
git switch -c recovery/<name> <commit>
git fsck --no-reflogs
```

---

## 16. Interview-ready mental model

Remember these three verbs:

```text
RESET      → MOVE the branch
REVERT     → UNDO with a new commit
CHERRY-PICK → COPY a commit's change
```

And for recovery:

```text
STOP → INSPECT → FIND OLD HEAD → CREATE RECOVERY REF → VALIDATE → RESTORE
```

---

## Beginner takeaway

If you remember only this:

1. **`git reset` moves the branch pointer and can rewrite history.**
2. **`git revert` creates a new commit that undoes an earlier commit.**
3. **`git cherry-pick` applies one existing commit's change to your current branch.**
4. **`git reflog` is one of the first places to look after an accidental local history change.**
5. **For shared production branches, prefer safe, reviewable rollback through revert/PR rather than rewriting history.**
