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
