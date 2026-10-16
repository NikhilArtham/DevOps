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
