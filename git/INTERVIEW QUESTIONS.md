# Git Interview Questions

Centralized interview questions for Git. Questions from every Git DAY are added here.

---

## DAY 1 — Git Internals: Objects, HEAD, Refs and Reflog

### 1. What are Git objects?

<details><summary>Answer</summary>
Git stores repository data as objects. The key types to know are blobs for file contents, trees for directory structure, and commits for snapshots plus metadata and parent relationships.
</details>

### 2. What is a blob in Git?

<details><summary>Answer</summary>
A blob stores file content. It does not itself store the filename; tree objects associate names with blobs.
</details>

### 3. What is a tree object?

<details><summary>Answer</summary>
A tree represents a directory snapshot and maps filenames to blobs or subtrees. A commit points to a tree representing the project state.
</details>

### 4. What is a commit object?

<details><summary>Answer</summary>
A commit records a repository snapshot through a tree reference and also stores metadata such as author, committer, message, and parent commit references.
</details>

### 5. How are a commit, tree and blob related?

<details><summary>Answer</summary>
A commit points to a tree, the tree represents the directory structure, and entries in the tree point to blobs containing file contents or other trees for subdirectories.
</details>

### 6. What is a Git ref?

<details><summary>Answer</summary>
A ref is a named reference to a Git object, commonly a commit. Branches and tags are examples of refs. Refs let users work with names such as `main` instead of raw commit IDs.
</details>

### 7. Is a branch a copy of the repository?

<details><summary>Answer</summary>
No. A branch is best understood as a movable reference pointing to a commit. Creating or moving a branch does not create a separate copy of all repository data.
</details>

### 8. What is HEAD?

<details><summary>Answer</summary>
HEAD identifies the current checkout position. Normally HEAD points to a branch, and that branch points to a commit. In detached HEAD state, HEAD points directly to a commit.
</details>

### 9. What is detached HEAD?

<details><summary>Answer</summary>
Detached HEAD means HEAD points directly to a commit rather than to a branch ref. Commits created there can become difficult to find unless you create a branch or otherwise retain a reference to them.
</details>

### 10. How do you check what commit HEAD points to?

<details><summary>Answer</summary>
Use `git rev-parse HEAD`. You can also inspect the commit with `git show HEAD`.
</details>

### 11. What does `git rev-parse --abbrev-ref HEAD` do?

<details><summary>Answer</summary>
It resolves HEAD to a short symbolic branch name when HEAD is attached to a branch. In detached HEAD state it will not give a normal current branch name.
</details>

### 12. What does `git show-ref` show?

<details><summary>Answer</summary>
It lists refs and the object IDs they currently point to, such as local branches, remote-tracking refs, and tags.
</details>

### 13. What is the difference between `main` and `origin/main`?

<details><summary>Answer</summary>
`main` is normally a local branch. `origin/main` is normally a remote-tracking ref representing the last known state of the remote `main` branch after fetching.
</details>

### 14. What is reflog?

<details><summary>Answer</summary>
Reflog is a local record of reference movements, including movements of HEAD and local branches. It is especially useful for recovering commits after reset, rebase, branch movement, or other history-changing operations.
</details>

### 15. What is the difference between `git log` and `git reflog`?

<details><summary>Answer</summary>
`git log` shows commit history reachable from the selected ref. `git reflog` records where local refs such as HEAD have pointed recently, including positions that may no longer be reachable from the current branch.
</details>

### 16. How would you recover from `git reset --hard HEAD~2`?

<details><summary>Answer</summary>
First stop making unnecessary changes. Run `git reflog`, identify the previous HEAD, inspect it with `git show`, and create a recovery branch pointing to the correct commit. After verification, restore the intended branch state.
</details>

### 17. Is reflog available on the remote repository?

<details><summary>Answer</summary>
Reflog is primarily local. Your local reflog does not represent a universal remote recovery log, and reflog entries can expire. Remote backups, protected branches, and other repository history should not be replaced by relying only on reflog.
</details>

### 18. What is `git fsck` used for in recovery?

<details><summary>Answer</summary>
`git fsck` can identify objects that are not reachable through normal refs. A dangling commit may contain work that became unreachable after reset, rebase, or branch deletion. Such a commit can be inspected and, if valid, referenced by a new branch.
</details>

### 19. Why can an accidentally reset commit still exist?

<details><summary>Answer</summary>
Reset normally moves a ref; it does not instantly erase every object belonging to the previous history. The old commit may remain reachable through reflog or as an unreachable object until Git's cleanup and expiration processes remove it.
</details>

### 20. Explain Git internals in an interview in one minute.

<details><summary>Answer</summary>
Git stores repository data as objects: blobs contain file contents, trees represent directory structure, and commits point to trees and parent commits. Branches and tags are refs that give commits names. HEAD identifies the current checkout position. Reflog records recent local movements of refs, making it a key recovery mechanism after accidental reset, rebase, or branch movement.
</details>

---

## DAY 2 — Git Branching, Merge vs Rebase and Conflict Recovery

### 1. What is a Git branch?

<details><summary>Answer</summary>
A branch is a movable reference pointing to a commit. It is not a separate copy of the repository.
</details>

### 2. What is the difference between `git switch -c` and `git switch`?

<details><summary>Answer</summary>
`git switch -c feature/name` creates a new branch and switches to it. `git switch feature/name` switches to an existing branch.
</details>

### 3. What is a fast-forward merge?

<details><summary>Answer</summary>
A fast-forward merge happens when the target branch has not diverged from the source branch. Git can simply move the target branch ref forward to the source commit without creating a merge commit.
</details>

### 4. What is a three-way merge?

<details><summary>Answer</summary>
A three-way merge combines two branch tips using their common ancestor. When both branches have unique commits, Git may create a merge commit with both branch tips as parents.
</details>

### 5. What is the difference between merge and rebase?

<details><summary>Answer</summary>
Merge combines histories and may create a merge commit while preserving the existing topology. Rebase replays commits on top of a new base, creating new commit IDs and usually producing a more linear history.
</details>

### 6. Why does rebase change commit IDs?

<details><summary>Answer</summary>
A commit's identity depends on its content and parent relationship. When a commit is replayed onto a different parent, its commit object changes, so it receives a new ID.
</details>

### 7. When should you avoid rebasing?

<details><summary>Answer</summary>
Avoid casually rebasing commits that are already shared and relied upon by other developers. Rebase rewrites history and can require force-pushing, which can disrupt collaborators.
</details>

### 8. What does `git pull --rebase` do?

<details><summary>Answer</summary>
It fetches remote changes and then rebases local commits on top of the updated upstream history instead of creating a normal merge commit for those local changes.
</details>

### 9. What is a merge conflict?

<details><summary>Answer</summary>
A merge conflict occurs when Git cannot automatically determine the correct combined result, commonly because both sides changed the same area differently.
</details>

### 10. How do you resolve a merge conflict?

<details><summary>Answer</summary>
Run `git status`, inspect the conflicted files, edit them into the correct final state, remove conflict markers, run `git add <file>`, and then complete the merge with `git commit`.
</details>

### 11. How do you resolve a rebase conflict?

<details><summary>Answer</summary>
Run `git status` and inspect the conflict. Resolve the file, stage it with `git add <file>`, then run `git rebase --continue`. Repeat if another commit conflicts.
</details>

### 12. How do you abort a merge?

<details><summary>Answer</summary>
Use `git merge --abort` to attempt to return to the state before the merge started.
</details>

### 13. How do you abort a rebase?

<details><summary>Answer</summary>
Use `git rebase --abort` to cancel the rebase and attempt to return to the pre-rebase state.
</details>

### 14. What does `git rebase --skip` do?

<details><summary>Answer</summary>
It skips the commit currently being replayed during a rebase. It should only be used when you intentionally decide that the commit should not be replayed.
</details>

### 15. What are conflict markers?

<details><summary>Answer</summary>
Git may place markers such as `<<<<<<<`, `=======`, and `>>>>>>>` in a conflicted file. They identify the competing versions. They must be removed before the resolved file is committed.
</details>

### 16. Why should you not blindly use `ours` or `theirs`?

<details><summary>Answer</summary>
Choosing one side wholesale can discard valid changes. Also, the meaning of ours and theirs can be less intuitive during rebase because Git is replaying a commit onto another base. Inspect the operation and application intent first.
</details>

### 17. How do you find where two branches diverged?

<details><summary>Answer</summary>
Use `git merge-base main feature` to find their common ancestor. `git log main..feature` and `git log feature..main` can then show commits unique to each side.
</details>

### 18. Why is `git push --force-with-lease` preferred over `git push --force` after a rebase?

<details><summary>Answer</summary>
Both can update a remote branch non-fast-forward, but `--force-with-lease` checks that the remote branch is still at the expected state. This reduces the chance of overwriting someone else's newer work unexpectedly.
</details>

### 19. How would you handle a rebase conflict on a production-related Kubernetes or Terraform change?

<details><summary>Answer</summary>
Inspect the conflict with `git status` and `git diff`, understand what both branches intended, resolve the configuration to the correct operational state, remove markers, stage it, continue the rebase, and run the appropriate Kubernetes/Terraform validation and tests before pushing or deploying.
</details>

### 20. Explain merge vs rebase in one minute.

<details><summary>Answer</summary>
A merge combines two existing lines of history and can create a merge commit, so it preserves the branch topology. A rebase takes commits from one branch and replays them onto another base, producing new commit IDs and usually a linear history. I use rebase carefully for private or controlled feature branches and avoid rewriting shared history. If either operation conflicts, I inspect the conflict, resolve the correct application behavior, stage it, then continue or abort as appropriate.
</details>

---

## DAY 3 — Pull Requests, Code Review and Clean Commit History

### 1. What is a pull request?

<details><summary>Answer</summary>
A pull request is a controlled proposal to merge changes from a source branch into a base branch. It provides a place for review, automated checks, discussion, and approval before the change is integrated.
</details>

### 2. What are source/head and base branches in a PR?

<details><summary>Answer</summary>
The source or head branch contains the proposed changes. The base branch is the branch that will receive those changes, such as `main` or a release branch.
</details>

### 3. What makes a good DevOps PR?

<details><summary>Answer</summary>
It should be focused, understandable, reasonably small, and include the operational impact. For Terraform/Kubernetes changes, reviewers should check permissions, networking, ports, health checks, replicas, resource settings, and deployment behavior in addition to syntax.
</details>

### 4. Why should PRs be small and focused?

<details><summary>Answer</summary>
Smaller PRs are easier to review, test, reason about, and roll back. They also make it less likely that an important operational change is hidden among unrelated modifications.
</details>

### 5. What is clean commit history?

<details><summary>Answer</summary>
A clean history groups work into meaningful logical commits with useful messages rather than a long sequence of temporary fixes such as typo fixes and review-noise commits.
</details>

### 6. What does `git rebase -i HEAD~N` do?

<details><summary>Answer</summary>
It opens an interactive rebase for the selected range of recent commits. You can reorder, reword, squash, fix up, edit, or drop commits before the rewritten history is recreated.
</details>

### 7. What is the difference between `squash` and `fixup` in interactive rebase?

<details><summary>Answer</summary>
Both combine a commit into an earlier commit. `squash` lets you edit the combined commit message, while `fixup` normally discards the later commit's message automatically.
</details>

### 8. What does `git commit --amend` do?

<details><summary>Answer</summary>
It replaces the latest commit with a new commit containing the amended content and/or message. Because the commit changes, its ID normally changes too.
</details>

### 9. Why can cleaning history require a force push?

<details><summary>Answer</summary>
Interactive rebase and amend rewrite commits. The rewritten branch may no longer be a fast-forward update from the remote, so pushing it can require `git push --force-with-lease`.
</details>

### 10. Why is `--force-with-lease` safer than `--force`?

<details><summary>Answer</summary>
`--force-with-lease` checks that the remote ref is still at the expected value before replacing it. This helps prevent accidentally overwriting newer remote work from another developer.
</details>

### 11. What should a reviewer check in Terraform code?

<details><summary>Answer</summary>
Check IAM scope, security groups, networking, resource changes, state implications, destructive actions, module inputs, environment targeting, and the Terraform plan. Syntax alone is not enough.
</details>

### 12. What should a reviewer check in Kubernetes manifests?

<details><summary>Answer</summary>
Review image versions, replicas, ports, selectors, probes, resource requests/limits, configuration references, security settings, and whether the Service/Ingress/Gateway paths still reach the intended Pods.
</details>

### 13. Why is the final diff important even if commit history is clean?

<details><summary>Answer</summary>
A clean history improves readability but does not prove that the resulting configuration is safe. The final diff is what will actually reach the target branch, so it must be reviewed for correctness and operational impact.
</details>

### 14. What is branch protection used for?

<details><summary>Answer</summary>
Branch protection can require pull requests, approvals, successful status checks, and other rules before changes reach protected branches. It turns the review process into an enforceable repository control.
</details>

### 15. How do you inspect what your branch changes relative to `main`?

<details><summary>Answer</summary>
Useful commands include `git log main..HEAD` for commits unique to the branch and `git diff main...HEAD` for the effective change compared with the common ancestor.
</details>

### 16. How would you review an IAM change in a PR?

<details><summary>Answer</summary>
Inspect the actual policy diff, identify principals/actions/resources, check whether permissions are broader than required, and run the repository's Terraform formatting, validation, tests, and plan checks. A clean commit message is not evidence that the IAM change is safe.
</details>

### 17. What happens when you rebase a PR branch?

<details><summary>Answer</summary>
The feature commits are replayed onto the new base, so commit IDs may change. If the branch is already pushed, the updated history commonly requires a safe force push such as `git push --force-with-lease`.
</details>

### 18. What is a good commit message?

<details><summary>Answer</summary>
It should describe the logical change clearly and briefly, for example `Add payment health check` rather than `fix stuff` or `changes`. The exact convention may depend on the team's commit standard.
</details>

### 19. Why are CI checks important in a PR?

<details><summary>Answer</summary>
They provide automated evidence that the proposed change meets repository requirements. For DevOps repositories, checks can include Terraform validation/plan, Kubernetes validation, unit tests, security scanning, and deployment checks.
</details>

### 20. Explain the PR and clean-history workflow in one minute.

<details><summary>Answer</summary>
I create a focused branch, make logical commits, push it, and open a PR against the correct base branch. I inspect the final diff and commit history, review operational impact, and rely on required CI checks and approvals. If I need to clean private branch history, I can use interactive rebase and then update the remote with `--force-with-lease`. I never assume a clean history means the infrastructure change is safe.
</details>

---

## DAY 4 — Git Reset, Revert, Cherry-pick and Recovery

### 1. What does `git reset` do?

<details><summary>Answer</summary>
`git reset` moves the current branch reference to another commit and, depending on the mode, can also change the index and working tree. It is commonly used for local history correction and should be used carefully on shared branches.
</details>

### 2. What is the difference between `git reset --soft`, `--mixed`, and `--hard`?

<details><summary>Answer</summary>
All move the current branch to the target commit. `--soft` keeps the changes staged, `--mixed` keeps them in the working tree but unstaged, and `--hard` makes the index and working tree match the target commit. `--hard` can discard uncommitted work.
</details>

### 3. Why is `git reset --hard` dangerous?

<details><summary>Answer</summary>
It changes the branch history and also replaces the index and working tree with the target commit. Any uncommitted changes that are overwritten may be lost, and rewriting a shared branch can disrupt other users and automation.
</details>

### 4. What does `git revert` do?

<details><summary>Answer</summary>
`git revert` creates a new commit that applies the inverse of an existing commit. It preserves the existing history, which makes it a safer rollback mechanism for shared branches.
</details>

### 5. What is the main difference between reset and revert?

<details><summary>Answer</summary>
Reset moves a branch reference and can rewrite history. Revert adds a new commit that undoes a previous change without moving the branch backward. For a shared production branch, revert is usually the safer choice.
</details>

### 6. How do you revert a merge commit?

<details><summary>Answer</summary>
A merge commit has multiple parents, so Git needs to know which parent is the mainline. A common command is `git revert -m 1 <merge-commit>`, but you should inspect the merge and confirm which parent represents the mainline before choosing the parent number.
</details>

### 7. What is `git cherry-pick`?

<details><summary>Answer</summary>
Cherry-pick applies the change introduced by an existing commit to the current branch and normally creates a new commit. It is useful for targeted fixes or backports when you do not want to merge an entire branch.
</details>

### 8. Does cherry-pick move the original commit?

<details><summary>Answer</summary>
No. The original commit remains where it was. Cherry-pick creates a new commit on the current branch containing the selected change, so the new commit normally has a different ID.
</details>

### 9. When would you use cherry-pick in a DevOps environment?

<details><summary>Answer</summary>
A common case is a tested production hotfix that exists on a development or hotfix branch but must be applied to a release branch without bringing unrelated work. The selected commit can be cherry-picked and then validated through the normal CI/CD process.
</details>

### 10. What does `git cherry-pick --no-commit` do?

<details><summary>Answer</summary>
It applies the selected commit's changes to the working tree/index without immediately creating the commit. This lets you inspect, modify, or combine the changes before creating the final commit.
</details>

### 11. How do you resolve a cherry-pick conflict?

<details><summary>Answer</summary>
Run `git status` and `git diff`, resolve the conflicting files, stage them with `git add`, and run `git cherry-pick --continue`. If the operation should be abandoned, use `git cherry-pick --abort`.
</details>

### 12. What does `git cherry-pick --skip` do?

<details><summary>Answer</summary>
It skips the commit currently being cherry-picked. Use it only when you have confirmed that the selected change should not be applied, for example because the change is already present in the target branch.
</details>

### 13. How do you recover after an accidental `git reset --hard`?

<details><summary>Answer</summary>
Stop making further destructive changes, inspect `git reflog`, identify the previous HEAD, inspect it with `git show`, and create a recovery branch or ref pointing to the correct commit. Validate the recovered state before restoring the original branch.
</details>

### 14. Can reflog help after deleting a branch?

<details><summary>Answer</summary>
Often yes, if the relevant reference movement is still present in the local reflog. You can find the old commit ID and create a new branch pointing to it. Reflog is local and entries can expire, so it is not a permanent backup system.
</details>

### 15. What is the safest first step after a suspected Git history mistake?

<details><summary>Answer</summary>
Stop and inspect before doing more writes. Useful commands include `git status`, `git branch --show-current`, `git log --oneline --decorate --graph`, and `git reflog`. For risky recovery work, create a backup/recovery ref before changing anything further.
</details>

### 16. How would you choose between reset, revert and cherry-pick?

<details><summary>Answer</summary>
Use reset mainly for private/local history correction, revert for safely undoing a shared change, and cherry-pick when you need one specific existing commit's change on another branch. The decision depends on whether the history is shared and what operational result is required.
</details>

### 17. What should you do if a bad commit reached a shared production branch?

<details><summary>Answer</summary>
Inspect the bad commit, confirm the operational impact, and normally create a revert through the team's review/CI process rather than resetting the shared branch. Validate the resulting Terraform/Kubernetes/application state before deployment.
</details>

### 18. Why can cherry-picking a commit still be dangerous even when there is no conflict?

<details><summary>Answer</summary>
A clean cherry-pick only means Git could apply the patch mechanically. The commit may depend on earlier commits, configuration, files, or environment assumptions that are missing from the target branch. Always validate behavior after the cherry-pick.
</details>

### 19. How would Git recovery connect to Kubernetes or Terraform operations?

<details><summary>Answer</summary>
A Git change may control the Terraform or Kubernetes configuration that a CI/CD pipeline deploys. After a rollback or recovery, the recovered/reverted configuration should be reviewed and validated, for example with `terraform fmt -check`, `terraform validate`, `terraform plan`, `kubectl diff`, or server-side dry-run as appropriate.
</details>

### 20. Explain reset, revert, cherry-pick and recovery in one minute.

<details><summary>Answer</summary>
Reset moves the branch and can rewrite history, so I use it carefully for private/local corrections. Revert creates a new commit that undoes an earlier commit, making it suitable for shared rollback. Cherry-pick copies one commit's change onto the current branch, which is useful for targeted hotfixes. If I make a history mistake, I stop, inspect reflog, identify the old HEAD, create a recovery ref, validate the recovered state, and only then restore the intended branch.
</details>
