# DAY 1 — Git Internals: Objects, HEAD, Refs and Reflog

## Why This Topic Matters

Most DevOps engineers use Git commands such as `clone`, `pull`, `commit`, `push`, `merge`, and `rebase` without needing to know how Git stores the data underneath.

But when something goes wrong — a branch is deleted, a commit appears to disappear, a reset was done accidentally, or a rebase rewrites history — Git internals become extremely useful.

The most important mental model is:

```text
Git repository
│
├── Objects      → actual Git data
│   ├── blob     → file contents
│   ├── tree     → directory structure
│   └── commit   → snapshot + metadata + parent(s)
│
├── Refs        → names pointing to commits
│   ├── branches
│   └── tags
│
├── HEAD        → where your current checkout points
│
└── Reflog      → local history of ref movements
```

A very useful sentence to remember:

> **Objects store Git data, refs give commits names, HEAD tells Git where you are, and reflog helps recover where refs used to point.**

---

# 1. What Is Inside a Git Repository?

When you run:

```bash
git init
```

Git creates a hidden `.git` directory.

You can inspect it with:

```bash
ls -la
```

Typical important areas include:

```text
.git/
├── HEAD
├── config
├── objects/
├── refs/
│   ├── heads/
│   └── tags/
└── logs/
    └── refs/
```

Do not manually edit these files in a real repository unless you understand exactly what you are doing. Git commands should normally manage them.

---

# 2. Git Objects — The Foundation

Git stores repository content as objects.

The three object types you must know first are:

```text
blob
 tree
commit
```

There is also an annotated `tag` object, but branches and most everyday Git history work primarily through commits, trees, and blobs.

---

# 3. Blob Object

A blob stores file content.

For example:

```text
app.txt
```

might contain:

```text
hello DevOps
```

Git stores the content as a blob object.

Important:

> A blob stores file contents, but it does not store the filename.

The filename and its relationship to directories are represented by tree objects.

You can inspect objects with:

```bash
git cat-file -t <object-id>
git cat-file -p <object-id>
```

Flags:

- `-t` → show the object type
- `-p` → pretty-print the object contents

---

# 4. Tree Object

A tree represents a directory snapshot.

Conceptually:

```text
Tree
├── app.txt      → blob
├── config.txt   → blob
└── scripts/     → another tree
```

The tree connects filenames to the corresponding blobs or subtrees.

So:

```text
blob → file content
 tree → directory structure
```

---

# 5. Commit Object

A commit represents a snapshot of the repository plus metadata.

A simplified commit contains information such as:

```text
tree <tree-id>
parent <previous-commit-id>
author ...
committer ...

commit message
```

The commit points to a tree representing the project state.

A normal history therefore looks like:

```text
Commit C
  ↓ parent
Commit B
  ↓ parent
Commit A
```

A merge commit can have multiple parents.

---

# 6. Why Does Git Use Hashes?

Git identifies objects using object IDs derived from their content and metadata.

Modern Git repositories commonly use SHA-1 object IDs, with SHA-256 repositories also supported by Git.

You will commonly see an ID such as:

```text
3f8a7c...
```

You do not need to memorize the hash algorithm to troubleshoot Git. The important point is:

```text
Object content
      ↓
Object ID
      ↓
Git can find that object
```

---

# 7. Inspect Git Objects

After creating a commit, run:

```bash
git cat-file -p HEAD
```

You may see:

```text
tree abc123...
parent def456...
author ...
committer ...

My commit
```

Then inspect the tree:

```bash
git cat-file -p <tree-id>
```

You may see entries such as:

```text
100644 blob 123abc... README.md
040000 tree 456def... scripts
```

This shows the relationship:

```text
commit
  ↓
tree
  ├── blob → README.md
  └── tree → scripts/
```

---

# 8. `git rev-parse`

`git rev-parse` is extremely useful when learning Git internals and troubleshooting references.

Examples:

```bash
git rev-parse HEAD
git rev-parse --show-toplevel
git rev-parse --abbrev-ref HEAD
```

What they mean:

```text
git rev-parse HEAD
→ resolve HEAD to a commit ID

--show-toplevel
→ show repository root

--abbrev-ref HEAD
→ show the current branch name when HEAD is attached to a branch
```

---

# 9. What Is a Ref?

A ref is a human-friendly name that points to an object, usually a commit.

Instead of remembering:

```text
9c4d7a8b...
```

you can use:

```text
main
feature/login
v1.0.0
```

Conceptually:

```text
main ───────────────→ Commit C
feature/login ──────→ Commit B
```

This is why a branch is best understood as a **movable reference to a commit**, not as a separate copy of the repository.

---

# 10. Branches Are Refs

You can inspect local branches with:

```bash
git branch
```

Git stores local branch refs under a structure conceptually represented by:

```text
.git/refs/heads/
```

For example:

```text
.git/refs/heads/main
```

contains a commit ID for the `main` branch in the traditional loose-ref representation.

Inspect a branch with:

```bash
git rev-parse main
```

---

# 11. What Is HEAD?

`HEAD` identifies the commit or branch you are currently checked out at.

Most of the time, HEAD points to the current branch:

```text
HEAD
 ↓
main
 ↓
Commit C
```

You can inspect it with:

```bash
cat .git/HEAD
git symbolic-ref HEAD
git rev-parse HEAD
```

A normal `.git/HEAD` often contains something similar to:

```text
ref: refs/heads/main
```

That means:

```text
HEAD → main → Commit C
```

---

# 12. Detached HEAD

A detached HEAD means HEAD points directly to a commit instead of a branch ref.

Example:

```bash
git checkout <commit-id>
```

The structure becomes:

```text
HEAD
 ↓
Commit C
```

instead of:

```text
HEAD
 ↓
main
 ↓
Commit C
```

You can still create commits in detached HEAD state, but those commits are not automatically attached to a normal branch.

To preserve the work:

```bash
git switch -c recovery-branch
```

---

# 13. `git show-ref`

Use:

```bash
git show-ref
```

to inspect refs.

You may see:

```text
abc123 refs/heads/main
def456 refs/remotes/origin/main
789abc refs/tags/v1.0.0
```

This helps you understand that Git's branch and tag names are references to object IDs.

---

# 14. Local Branch vs Remote-Tracking Branch

These are different concepts.

```text
main
```

is your local branch.

```text
origin/main
```

is normally a remote-tracking ref representing the last known state of the remote `main` branch.

Conceptually:

```text
Local:
main ─────────────→ Commit C

Remote-tracking:
origin/main ──────→ Commit B
```

After fetching:

```bash
git fetch origin
```

Git updates the remote-tracking information without automatically changing your local branch.

---

# 15. What Is Reflog?

Reflog is a local record of movements of references such as HEAD and local branches.

View it with:

```bash
git reflog
```

Example:

```text
abc123 HEAD@{0}: reset: moving to HEAD~1
def456 HEAD@{1}: commit: Add payment API
789abc HEAD@{2}: commit: Add login API
```

The reflog answers a very useful question:

> **Where was this reference pointing recently?**

This is why reflog is one of the most useful Git recovery tools.

---

# 16. Reflog Is Not the Same as Git Log

`git log` shows reachable commit history.

```bash
git log --oneline
```

It answers:

> What commits are in the history I am currently following?

`git reflog` records local reference movements.

```bash
git reflog
```

It answers:

> Where did HEAD or another local ref point recently?

This distinction matters after destructive-looking operations such as reset or rebase.

---

# 17. The Most Important Recovery Pattern

Suppose you accidentally run:

```bash
git reset --hard HEAD~2
```

Your branch moved backward.

First inspect:

```bash
git reflog
```

You may find the previous position:

```text
abc123 HEAD@{0}: reset: moving to HEAD~2
def456 HEAD@{1}: commit: Important production fix
```

You can recover by moving the branch back to the desired commit:

```bash
git reset --hard def456
```

Before doing this in a real repository, verify exactly which commit you want to restore.

---

# 18. Reflog Is Local

This is a critical limitation.

```text
Your local reflog ≠ remote GitHub history
```

If another engineer deletes a branch on the remote, your local reflog may help only if your local repository previously had the relevant reference/commit.

Also, reflog entries are not permanent backups. They can expire according to Git's reflog expiration settings and repository maintenance.

Therefore:

> **Reflog is a powerful local recovery mechanism, not a replacement for backups or remote history.**

---

# 19. Useful Internal Inspection Commands

## Show current commit

```bash
git rev-parse HEAD
```

## Show current branch

```bash
git branch --show-current
```

## Show HEAD structure

```bash
cat .git/HEAD
```

## Show refs

```bash
git show-ref
```

## Show object type

```bash
git cat-file -t <object-id>
```

## Show object contents

```bash
git cat-file -p <object-id>
```

## Show recent reference movements

```bash
git reflog
```

## Show commit graph

```bash
git log --oneline --decorate --graph --all
```

---

# 20. `git fsck` — Finding Dangling Objects

`git fsck` can help identify objects that are no longer reachable through normal refs.

Run:

```bash
git fsck --no-reflogs
```

You may see objects described as:

```text
dangling commit ...
dangling blob ...
```

A dangling commit is not necessarily garbage or useless. It may contain work that became unreachable after a reset, rebase, or deleted branch.

Inspect a candidate with:

```bash
git show <commit-id>
```

Then, if it contains important work, create a branch pointing to it:

```bash
git switch -c recovered-work <commit-id>
```

---

# 21. Garbage Collection

Git eventually cleans up unreachable objects according to its maintenance and expiration rules.

Commands you may encounter include:

```bash
git gc
```

and:

```bash
git maintenance run
```

Do not think of dangling objects as permanent storage. Recovery becomes harder after unreachable objects are pruned.

---

# 22. Realistic Failure Scenario — Accidental Hard Reset

### Incident

You are working on a production-related branch:

```text
feature/payment-fix
```

The branch contains:

```text
A → B → C → D
```

You accidentally run:

```bash
git reset --hard HEAD~2
```

Now the visible branch history appears to be:

```text
A → B
```

You might think commits C and D are permanently lost.

### Step 1 — Do not immediately create more destructive changes

First inspect:

```bash
git status
git reflog
```

### Step 2 — Find the old HEAD

Look for the entry before the reset:

```text
HEAD@{1}
```

or another appropriate reflog entry.

### Step 3 — Inspect the candidate commit

```bash
git show <candidate-commit>
```

Confirm that it contains the missing work.

### Step 4 — Recover safely

Create a temporary recovery branch first:

```bash
git switch -c recovery/payment-fix <candidate-commit>
```

Now the previously hidden commits are reachable through a branch.

### Step 5 — Decide the final history

After verifying the recovery, move or merge the intended branch as appropriate.

The key lesson:

```text
reset moved a ref
      ↓
objects may still exist
      ↓
reflog remembers the ref movement
      ↓
find old commit
      ↓
create recovery ref
```

---

# 23. Common Mistakes

### Mistake 1 — Thinking a branch contains commits

Better mental model:

```text
branch = ref pointing to a commit
```

### Mistake 2 — Thinking HEAD is the same thing as a branch

Usually:

```text
HEAD → branch → commit
```

But in detached HEAD:

```text
HEAD → commit
```

### Mistake 3 — Using `git log` when you need reflog

After reset/rebase, the commit may no longer be reachable from the current branch history.

Try:

```bash
git reflog
```

### Mistake 4 — Treating reflog as a remote backup

Reflog is primarily local and can expire.

### Mistake 5 — Running `git gc` before investigating recovery

If you suspect important work became unreachable, investigate refs, reflog, and dangling objects before intentionally performing aggressive cleanup.

---

# 24. Interview-Ready Mental Model

If asked **“Explain Git internals simply”**, say:

> Git stores data as objects. Blobs store file contents, trees represent directory structure, and commits point to trees and parent commits. Branches and tags are refs that give commits human-friendly names. HEAD identifies the current checkout. Reflog records local movements of refs, which makes it extremely useful for recovering from accidental reset, rebase, or branch movement.

If asked **“How would you recover an accidentally reset branch?”**:

```text
git reflog
   ↓
find previous HEAD
   ↓
git show <commit>
   ↓
create recovery branch
   ↓
verify
   ↓
restore intended branch state
```

---

# Beginner Takeaway

You do not need to memorize every `.git` file.

Understand these four ideas first:

```text
OBJECTS
→ Git's stored data

REFS
→ names pointing to commits

HEAD
→ your current position

REFLOG
→ recent local movements of refs
```

Once these make sense, commands such as `reset`, `rebase`, `checkout`, `switch`, branch deletion, and Git recovery become much easier to understand.
