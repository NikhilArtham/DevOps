# DAY 3 — Pull Requests, Code Review and Clean Commit History

## Why This Topic Matters

In a real DevOps team, changes normally move through a Pull Request (PR), review, automated checks, approval, and finally merge/deployment.

```text
feature branch → clean commits → PR → CI + code review → approval → merge → CI/CD → AWS/Kubernetes
```

The goal is not merely to get a PR merged. The goal is to produce a change that is **understandable, reviewable, testable, and safe to deploy**.

---

# 1. What Is a Pull Request?

A Pull Request is a request to integrate changes from one branch into another branch.

```text
feature/payment
      ↓
   Pull Request
      ↓
main
```

A PR normally contains the source branch, target branch, changed files, commits, description, review comments, CI results, approvals, and merge status.

Git provides the branch and commit history. GitHub provides the Pull Request workflow around those Git objects.

---

# 2. Source Branch vs Base Branch

For:

```text
feature/payment-health-check → main
```

```text
feature/payment-health-check = source/head branch
main                         = base/target branch
```

The PR asks whether the changes from the feature branch can be integrated into `main`.

---

# 3. Basic PR Workflow

```bash
git switch main
git pull --ff-only origin main
git switch -c feature/payment-health-check
```

Make changes:

```bash
git add .
git commit -m "Add payment health check"
```

Push:

```bash
git push -u origin feature/payment-health-check
```

`-u` establishes the upstream relationship so later `git pull` and `git push` commands know the default remote branch.

Then open a PR from the feature branch into `main`.

---

# 4. What Makes a Good PR?

A good PR should make these questions easy to answer:

### What changed?

The title and description should explain the change.

### Why did it change?

Explain the problem or requirement.

### How was it tested?

Include relevant tests, validation commands, screenshots/logs when appropriate, or deployment validation.

### What could be risky?

Call out production configuration, database, IAM, networking, Terraform, or other high-impact changes.

### How can the reviewer verify it?

Keep the PR focused and make the important diff easy to understand.

---

# 5. Keep PRs Small and Focused

Avoid a PR like:

```text
Add monitoring
+ refactor Terraform
+ rename 50 files
+ update README
+ change IAM
+ fix unrelated bug
```

Prefer separate focused PRs. A smaller PR is usually easier to review, test, revert, and troubleshoot.

---

# 6. What Is Code Review?

Code review is a structured check of a change before integration.

A reviewer should ask more than “Does this code work?”

Consider:

- Is the design appropriate?
- Could it break existing behavior?
- Is error handling sufficient?
- Are security implications understood?
- Are permissions too broad?
- Are infrastructure changes safe?
- Are tests sufficient?
- Is observability adequate?
- Can another engineer maintain it?

For DevOps, review is especially important because a small configuration change can affect many systems.

---

# 7. Code Review for DevOps Changes

For Terraform:

```text
Does the plan create, modify, or destroy resources?
Are IAM permissions broader than required?
Could this affect production?
Are variables/defaults safe?
```

For Kubernetes:

```text
Did replicas change?
Did resource requests/limits change?
Did container ports change?
Did readiness/liveness probes change?
Did Service selectors change?
Did RBAC change?
```

For AWS:

```text
Security Group rules
IAM policies
ALB/NLB listeners
Route 53 records
CloudWatch alarms
ECS/EKS configuration
```

The reviewer should understand the operational effect, not just the syntax.

---

# 8. CI Checks in a Pull Request

A mature PR workflow can run:

```text
PR opened
   ↓
unit tests
   ↓
lint
   ↓
security scan
   ↓
Terraform fmt/validate/plan
   ↓
Kubernetes manifest validation
   ↓
build
   ↓
approval
```

A green PR means the configured checks passed. It does **not** automatically mean the change is perfect. Human review and automated validation solve different problems.

---

# 9. Clean Commit History

A clean commit history means commits are understandable and represent meaningful changes.

Bad:

```text
fix
fix2
final
final-final
changes after review
small correction
actual final
```

Better:

```text
Add payment health check
Add unit tests for payment health check
Handle payment timeout error
```

A reviewer should be able to understand the evolution of the change from the commits.

---

# 10. Good Commit Messages

A useful pattern is:

```text
<verb> <specific change>
```

Examples:

```text
Add ALB health check
Fix Terraform IAM policy
Update EKS deployment resources
Add CloudWatch alarm for worker failures
Remove obsolete security group rule
```

Avoid vague messages such as `update`, `changes`, `fix stuff`, or `work`.

Follow your team's commit convention if one exists.

---

# 11. Squashing Commits

Suppose your branch contains:

```text
Add feature
fix typo
fix review
fix test
final fix
```

You may want one clean commit before merging:

```text
Add payment health check
```

Interactive rebase can combine commits:

```bash
git rebase -i HEAD~5
```

You may see:

```text
pick abc123 Add feature
pick def456 fix typo
pick 123abc fix review
pick 456def fix test
pick 789abc final fix
```

Change later commits from `pick` to `squash` or `fixup` as appropriate.

Conceptually:

```text
5 messy commits → interactive rebase → 1 clean commit
```

---

# 12. `squash` vs `fixup`

Both combine commits during interactive rebase.

### `squash`

Combines the commit with the previous commit and allows you to edit the resulting commit message.

### `fixup`

Combines the commit with the previous commit while discarding the fixup commit's message.

Example:

```text
main change
fix review comment
fix typo
fix test
```

Use `fixup` when later changes are simply corrections to the previous logical commit.

---

# 13. Amending the Latest Commit

If you forgot something in the latest commit:

```bash
git add <file>
git commit --amend
```

To change only the message:

```bash
git commit --amend -m "Add payment health check"
```

Amending changes the commit ID, so be careful if that commit has already been shared.

---

# 14. Interactive Rebase for Clean History

```bash
git rebase -i HEAD~N
```

Common actions:

```text
pick   → keep commit
reword → change commit message
edit   → stop and modify commit
squash → combine with previous commit and edit message
fixup  → combine with previous commit and keep previous message
drop   → remove commit
```

Example:

```text
pick   A Add feature
fixup  B fix typo
fixup  C fix review comment
reword D Update docs
```

Result:

```text
A' Add feature
D' Update docs
```

Rewritten commits receive new IDs.

---

# 15. PR Review Changes and History

A common review cycle is:

```text
Developer opens PR
       ↓
Reviewer comments
       ↓
Developer changes code
       ↓
Developer commits fixes
       ↓
CI runs again
       ↓
Reviewer approves
```

You may temporarily have:

```text
Add feature
fix reviewer comment
fix test
fix lint
```

Before merging, the team may squash these into one logical commit or preserve the individual commits if they are meaningful. Follow the repository's merge policy.

---

# 16. Updating a PR After `main` Changes

If your PR is behind `main`:

```bash
git fetch origin
```

Then either merge:

```bash
git merge origin/main
```

or rebase:

```bash
git rebase origin/main
```

The correct choice depends on the team's workflow.

If you rebased a previously pushed branch, you may need:

```bash
git push --force-with-lease origin feature/payment-health-check
```

Never use a force push casually on a shared branch.

---

# 17. Useful PR Inspection Commands

Current branch:

```bash
git branch --show-current
```

Commits in your branch but not in `main`:

```bash
git log --oneline main..HEAD
```

Effective diff against the merge base:

```bash
git diff main...HEAD
```

Readable history:

```bash
git log --oneline --decorate --graph --all
```

Ahead/behind status:

```bash
git status -sb
```

Last few commits:

```bash
git log --oneline -n 10
```

---

# 18. GitHub Branch Protection and Required Reviews

GitHub repository policies can require controls such as:

```text
Pull Request required
Approval required
Status checks required
Conversation resolution required
No direct pushes to main
```

The exact rules depend on repository configuration.

The DevOps principle is:

> **Production branches should not depend on one engineer remembering every safety step manually.**

Repository rules can enforce part of the process.

---

# 19. AWS / Kubernetes Connection

PRs are often the gate before infrastructure reaches AWS or Kubernetes.

```text
Terraform/Kubernetes change
          ↓
feature branch
          ↓
PR
          ↓
code review
          ↓
CI validation
          ↓
merge
          ↓
CD pipeline
          ↓
AWS / EKS / ECS / Lambda
```

Changes that deserve careful review include:

```text
Terraform IAM policy
Terraform Security Group
ALB listener/rule
Route 53 record
EKS Deployment
Kubernetes Service
Ingress/Gateway
ConfigMap/Secret references
ECS task definition
CloudWatch alarm
```

A clean PR makes it easier to understand exactly what infrastructure will change before deployment.

---

# 20. Realistic Failure Scenario — Dangerous PR with a “Clean” History

## Incident

A developer has a PR for an AWS application deployment.

The branch contains:

```text
Add deployment change
fix review
fix typo
final changes
```

During review, someone asks for an IAM policy change. The developer edits the policy and squashes everything into one commit:

```text
Update deployment
```

The history looks clean, but the final diff accidentally contains an overly broad permission:

```text
Allow:
  s3:*
  on:
  *
```

instead of the intended narrow permission.

The PR gets approved because reviewers focused on the commit history and did not inspect the final IAM diff carefully.

## Why This Is Dangerous

A clean commit history does **not** guarantee a safe change.

Reviewers should review the final diff and understand the operational effect.

## Recovery

Before deployment:

```bash
git fetch origin
git diff origin/main...HEAD
```

Inspect the IAM policy carefully.

Fix it:

```bash
git add path/to/iam-policy.tf
git commit -m "Restrict application S3 permissions"
```

Run the appropriate Terraform checks:

```bash
terraform fmt -check
terraform validate
terraform plan
```

Then update the PR and request another review if the change materially affects the approved behavior.

## Key Lesson

```text
Clean history ≠ safe change

Good PR =
clear intent
+ focused diff
+ automated checks
+ meaningful review
+ correct infrastructure behavior
```

---

# 21. Common Mistakes

### Mistake 1 — Huge PRs

Large unrelated changes make review weaker.

### Mistake 2 — “LGTM” without reading the diff

Approval should mean the reviewer understands the change and its risk.

### Mistake 3 — Reviewing only commit messages

The final diff is what matters operationally.

### Mistake 4 — Squashing everything without checking the result

Squashing can make history cleaner but can also hide the evolution of a risky change.

### Mistake 5 — Rebasing shared branches casually

Rebase rewrites commit IDs.

### Mistake 6 — Force-pushing without coordination

Prefer `--force-with-lease` when a rewritten branch genuinely needs a force update.

### Mistake 7 — Merging a green PR without understanding deployment impact

CI passing means configured checks passed. It does not replace engineering judgment.

---

# 22. Interview-Ready Mental Model

If asked **“What is a Pull Request?”**:

> A Pull Request is a controlled request to integrate changes from a source branch into a target branch. It provides a place for code review, automated checks, discussion, approval, and controlled merging.

If asked **“What makes a good PR?”**:

```text
small
focused
clear description
meaningful commits
passing CI
appropriate tests
understandable diff
clear operational risk
```

If asked **“How do you maintain clean Git history?”**:

```text
small logical commits
meaningful messages
interactive rebase when appropriate
squash/fixup review noise
avoid rewriting shared history
```

If asked **“How do PRs fit into DevOps?”**:

```text
Git branch
  ↓
PR
  ↓
review + CI
  ↓
merge
  ↓
CD pipeline
  ↓
AWS/Kubernetes
```

---

# Beginner Takeaway

Remember these four ideas:

```text
PULL REQUEST
→ controlled request to merge branch changes

CODE REVIEW
→ humans verify correctness, design, security and operational impact

CLEAN HISTORY
→ small logical commits with useful messages

SAFE MERGE
→ review the final diff + CI + infrastructure impact before deployment
```

The most important DevOps lesson is:

> **A PR is not just a GitHub formality. It is a production safety gate.**
